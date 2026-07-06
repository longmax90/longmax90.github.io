---
layout: default
title: "How to Attack the Kernel: Why copy_from_user Exists"
date: 2026-07-26 00:00:00 +0000
categories: [software-security, undefined-behavior]
tags: [software-security, undefined-behavior, c, formal-verification, exploit-development]
---

Every syscall is a trust boundary crossing.

When your program calls `read()` or `ioctl()`, it hands control to the Linux kernel — ring 0, full hardware privileges. The arguments come from userspace. Userspace is untrusted. If the kernel dereferences a userspace pointer without validation, the attacker controls what the kernel reads — and sometimes what it writes.

This is why `copy_from_user` exists.

---

## The Kernel/Userspace Boundary

Userspace runs at ring 3 — restricted. The kernel runs at ring 0 — unrestricted. When a syscall fires, the CPU switches to ring 0. The arguments — buffer addresses, lengths, flags — are userspace addresses. The kernel must read data at those addresses. Here is the naive approach:

```c
// BAD: naive kernel read from userspace
ssize_t bad_read_handler(int fd, void *user_buf, size_t count) {
    char kernel_buf[4096];
    memcpy(kernel_buf, user_buf, count);  // directly copy from userspace address
    // ... process kernel_buf ...
}
```

This looks reasonable. But it has at least three serious problems.

**Problem 1: The address might be a kernel address.** Nothing stops the attacker from passing `user_buf = 0xffffffff80000000` — a kernel virtual address. The `memcpy` reads kernel memory and returns it to the attacker. Private keys, session tokens, other processes' credentials — whatever happens to be adjacent in kernel memory.

**Problem 2: The address might not be mapped.** If `user_buf` points to unmapped memory, the kernel takes a page fault. In kernel context, an unhandled page fault is a kernel panic. An unprivileged attacker can crash the entire system by passing a bad pointer.

**Problem 3: TOCTOU with userspace memory.** The attacker controls userspace. Between the time the kernel validates the address and the time it reads the data, a second thread can modify the content. The kernel sees valid data during the check and attacker-controlled data during the use — a race condition that has been exploited in real vulnerabilities including Dirty COW (CVE-2016-5195).

---

## What copy_from_user Actually Does

`copy_from_user` is the correct way to read from a userspace address in the Linux kernel:

```c
// CORRECT: validated copy from userspace
ssize_t good_read_handler(int fd, void __user *user_buf, size_t count) {
    char kernel_buf[4096];
    if (count > sizeof(kernel_buf))
        return -EINVAL;
    if (copy_from_user(kernel_buf, user_buf, count))
        return -EFAULT;
    // ... process kernel_buf safely ...
}
```

`copy_from_user` does several things a raw `memcpy` does not:

**Address validation via `access_ok`** — checks the address range lies entirely within userspace. A kernel address gets `-EFAULT`, not memory disclosure.

**Fault-safe copy** — if the address is unmapped, handles the page fault gracefully instead of crashing the kernel.

**The `__user` annotation** — a sparse checker tag signaling the pointer is a userspace address and must not be dereferenced directly. Not compiler-enforced, but documents the contract.

---

## What Happens When copy_from_user Is Missing

CVE-2009-1897 — discussed in an earlier post — is one example where a kernel check was compiled away. But a missing `copy_from_user` is a more direct class of vulnerability: the kernel dereferences an attacker-controlled pointer directly, with no compiler involvement required.

The classic pattern:

```c
// Vulnerable: attacker-controlled pointer directly dereferenced
long vulnerable_ioctl(struct file *f, unsigned int cmd, unsigned long arg) {
    struct config *cfg = (struct config *)arg;  // arg is a userspace address
    int value = cfg->field;                      // direct dereference — no copy_from_user
    // ...
}
```

The attacker passes `arg` pointing to a kernel address. `cfg->field` reads from kernel memory and the value is used — potentially returned to userspace or used to influence kernel behavior. The attacker has an arbitrary kernel read primitive. Combined with a write primitive, this is enough for full privilege escalation.

This class of bug has appeared in device drivers, file system code, and network subsystems. Drivers are particularly common: kernel drivers are written by hardware vendors with varying levels of security expertise, and ioctl handlers are a frequent source of missing `copy_from_user` calls.

---

## Copy Fail: A Recent Real-World Example

In April 2026, Theori disclosed CVE-2026-31431 — "Copy Fail" — a logic bug in the kernel's AEAD crypto implementation (`algif_aead`). Improper scatter-gather list handling causes the AEAD implementation to write four bytes past its buffer into another file's page cache — the in-memory copy of files on disk.

An attacker triggers this write to corrupt the in-memory copy of a setuid-root binary like `/usr/bin/su`. The on-disk file is untouched. When a privileged process runs the corrupted in-memory version, it executes attacker code with root privileges.

The exploit: a 732-byte Python script. No heap overflow, no UAF — just a logic flaw in copy semantics writing four bytes to the wrong location. Affected: every Linux kernel from 4.14 (2017) to the 2026 patch.

---

## The Standard Kernel Exploitation Primitives

Beyond `copy_from_user` bugs, kernel exploitation relies on a handful of standard primitives that appear repeatedly across CVEs:

**Kernel address leak** — KASLR randomizes the kernel base address. Leaks come from uninitialized stack memory exposed to userspace, timing side channels, or `/proc` entries disclosing addresses.

**Arbitrary kernel write** — buffer overflow, UAF, or missing bounds check in a kernel buffer. The target: `cred`, the structure holding a process's UID, GID, and capabilities.

**Credential overwrite** — zeroing `cred.uid` and `cred.gid` gives the process root UID. Root shell.

**Exploit chain:** leak KASLR → arbitrary write → overwrite `cred` → root.

---

## Why Kernel Bugs Are High Value

A userspace bug gives the attacker code execution in one process, sandboxed by the OS. A kernel bug gives the attacker code execution in ring 0 — with full hardware privileges, the ability to modify any process's credentials, and access to all memory on the system.

This is why kernel exploitation is the final step in browser exploit chains: a userspace bug breaks out of the renderer, and a kernel bug elevates the sandboxed process to root. Two bugs, chained: one for escape, one for elevation.

It is also why kernel security research matters for firmware security teams. UEFI, hypervisors, and OS kernels form the trust foundation of the entire software stack. A bug at any of these layers undermines every security assumption above it. And with the Linux kernel receiving 3,529 CVE disclosures in 2024 alone — a tenfold increase after the kernel team became a CVE Numbering Authority — the attack surface is larger and better documented than ever.

---

## The Formal Verification Gap

The Linux kernel is 30+ million lines of C — full formal verification is not realistic. But targeted verification of critical subsystems is feasible. seL4 is the canonical example: a formally verified microkernel with machine-checked proofs of correctness and security properties.

For Linux, kernel-specific tools like Smatch and Coccinelle perform targeted static analysis — Smatch flags missing `__user` annotations and locking violations, Coccinelle finds recurring bug patterns. The sparse checker enforces `__user` contracts at compile time.

Abstract interpretation tools like TrustInSoft Analyzer go further: applied to specific drivers or subsystems, they prove the absence of runtime errors on any input. Not "we tested it." A proof.

CVE-2026-31431 was a logic bug in copy semantics. Formal specification of the expected behavior would have caught it before disclosure.
