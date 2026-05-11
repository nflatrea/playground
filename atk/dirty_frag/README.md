# Dirty Frag — Universal Linux LPE (CVE-2026-43284 / CVE-2026-43500)

/////// Disclaimer /////////////////////////////////////////////////////////////

This project is provided solely for educational purposes.
By using any part of this repository, you acknowledge that you will not
utilize the code or techniques contained herein to gain unauthorized access
to systems that you do not own or have explicit permission to test.

The author (nflatrea) assumes no responsibility or liability for any misuse,
damage, or consequences resulting from the use of this proof-of-concept or
related materials, and you agree to use this code at your own risk.

////////////////////////////////////////////////////////////////////////////////

Dirty Frag is a universal Linux local privilege escalation that chains two page-cache
write vulnerabilities — `xfrm-ESP Page-Cache Write (CVE-2026-43284)` and
`RxRPC Page-Cache Write (CVE-2026-43500)` — to obtain root on every major distribution.

It extends the bug class to which [Dirty Pipe](https://dirtypipe.cm4all.com/) and
[Copy Fail](https://copy.fail/) belong. Because it is a deterministic logic bug that
does not depend on a timing window, no race condition is required, the kernel does not
panic when the exploit fails, and the success rate is very high.

Original research and PoC by [Hyunwoo Kim (@v4bel)](https://x.com/v4bel):
https://github.com/V4bel/dirtyfrag

### Vulnerability Overview

The exploit "dirties" the `frag` member of `struct sk_buff` to gain an arbitrary
4-byte STORE primitive into the page cache, then overwrites `/etc/passwd` (or any
read-only file) to escalate privileges.

**CVE-2026-43284** (xfrm-ESP) provides the page-cache write primitive on most
distributions but requires namespace privileges. Ubuntu sometimes blocks unprivileged
user namespace creation through AppArmor policy, which prevents this path.

**CVE-2026-43500** (RxRPC) does not require namespace privileges but depends on the
`rxrpc.ko` module, which is loaded by default on Ubuntu but absent on most other
distributions.

Chaining the two variants makes the blind spots cover each other, allowing root
privileges to be obtained on every major distribution.

### Affected Versions

**CVE-2026-43284** — from `cac2661c53f3` (2017-01-17) up to `f4c50a4034e6` (2026-05-05)
**CVE-2026-43500** — from `2dc334f1a63a` (2023-06-08) up to `aa54b1d27fe0` (2026-05-10)

The effective lifetime of the vulnerabilities is approximately 9 years.

Tested on:

* Ubuntu 24.04.4: 6.17.0-23-generic
* RHEL 10.1: 6.12.0-124.49.1.el10_1.x86_64
* openSUSE Tumbleweed: 7.0.2-1-default
* CentOS Stream 10: 6.12.0-224.el10.x86_64
* AlmaLinux 10: 6.12.0-124.52.3.el10_1.x86_64
* Fedora 44: 6.19.14-300.fc44.x86_64

### Requirements

* A Linux system running a vulnerable kernel (see above)
* `gcc` and basic build tools installed

### Running

```sh
git clone https://github.com/V4bel/dirtyfrag.git && cd dirtyfrag && gcc -O0 -Wall -o exp exp.c -lutil && ./exp
```

### Cleanup

After running the exploit, the page cache is contaminated. Clear it by running:

```sh
echo 3 > /proc/sys/vm/drop_caches
```

or reboot the system.

### Mitigation

Remove the affected modules and flush the page cache:

```sh
sh -c "printf 'install esp4 /bin/false\ninstall esp6 /bin/false\ninstall rxrpc /bin/false\n' > /etc/modprobe.d/dirtyfrag.conf; rmmod esp4 esp6 rxrpc 2>/dev/null; echo 3 > /proc/sys/vm/drop_caches; true"
```

Once each distribution backports a patch, update accordingly.

### Patches

* CVE-2026-43284 patched in mainline: [`f4c50a4034e6`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f4c50a4034e62ab75f1d5cdd191dd5f9c77fdff4)
* CVE-2026-43500 patched in mainline: [`aa54b1d27fe0`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=aa54b1d27fe0c2b78e664a34fd0fdf7cd1960d71)

### Relationship to Copy Fail

Copy Fail was the motivation for this research. The xfrm-ESP path in Dirty Frag shares
the same sink as Copy Fail, but triggers regardless of whether the `algif_aead` module is
available. Even on systems where the known Copy Fail mitigation (algif_aead blacklist) is
applied, the system remains vulnerable to Dirty Frag.

### Credit

Dirty Frag was discovered and reported by [Hyunwoo Kim (@v4bel)](https://x.com/v4bel).

Full write-up: https://github.com/V4bel/dirtyfrag/blob/master/assets/write-up.md
oss-security disclosure: https://www.openwall.com/lists/oss-security/2026/05/07/8
