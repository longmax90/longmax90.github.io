---
layout: default
title: "Bug Classes: A Taxonomy"
date: 2026-06-21 00:00:00 +0000
categories: [software-security, undefined-behavior]
tags: [software-security, undefined-behavior, c, cpp]
---

Security researchers do not just talk about "bugs." They talk about *classes* of bugs, categories that share the same root cause, the same exploitation patterns, and the same defensive techniques.

Understanding bug classes is the mental model that lets a researcher look at unfamiliar code and immediately identify where the risk lives. This post maps the major bug classes in systems software.

---

## Memory Corruption

Memory corruption is the broadest and most exploited category in systems security. It occurs when a program reads or writes memory in a way that violates the intended memory layout.

**Buffer overflow** — a write goes beyond the bounds of an allocated buffer, overwriting adjacent memory. Stack-based overflows can corrupt return addresses and saved registers. Heap-based overflows can corrupt heap metadata or adjacent objects. This is the oldest and most well-documented exploitation primitive in software security.

**Heap corruption** — includes buffer overflows on the heap, use-after-free, double-free, and heap metadata corruption. Modern allocators like glibc's ptmalloc and Linux's SLUB maintain internal metadata; corrupting these can redirect allocator behavior in attacker-controlled ways.

**Use-after-free (UAF)** — memory is freed then accessed through a stale pointer. If the attacker allocates controlled data at the freed address before the dereference, they control what the program reads or executes. UAF has dominated browser exploitation for the past decade.

**Double-free** — the same memory is freed twice, corrupting allocator metadata and enabling arbitrary write primitives in subsequent allocations.

**Type confusion** — an object is accessed through a pointer of the wrong type. The program reads fields at the wrong offsets, producing values that make no sense in the context of the actual object. In C++, type confusion in virtual dispatch can redirect vtable lookups to attacker-controlled data.

---

## Integer Errors

Integer errors arise when arithmetic operations produce values outside the expected range, and those values are used in security-sensitive contexts, particularly as sizes for memory allocations or as indices into arrays.

**Signed integer overflow** — as discussed in previous posts, signed overflow is undefined behavior in C. The compiler may optimize away checks that depend on overflow semantics, and the actual runtime behavior is architecture-dependent.

**Unsigned integer overflow (wraparound)** — unsigned overflow is defined in C (it wraps modulo $2^N$), but wrapping behavior in a size calculation can lead to allocation of a much smaller buffer than intended. The classic pattern: `size_t total = count * size;` wraps if `count * size > SIZE_MAX`, producing a small allocation that is subsequently overflowed.

**Integer truncation** — a wider integer is assigned to a narrower type, silently discarding the high bits. A value that was 65536 in a 32-bit variable becomes 0 when truncated to 16 bits. If this truncated value is used as an allocation size or array index, the result is a dangerously small buffer.

**Signedness confusion** — a signed integer used where unsigned was expected. A length stored as `int` compared against `unsigned` allows negative values to bypass range checks.

---

## Logic Bugs

Logic bugs are correctness errors that do not involve memory corruption. They arise when the program's control flow or state machine does not match the developer's intent, typically due to missing checks, incorrect assumptions, or incomplete validation.

**Authentication bypass** — a missing or incorrect authentication check allows an unprivileged user to access a privileged operation. These bugs often arise from complex condition logic where one branch of a decision tree fails to check credentials.

**TOCTOU (Time-Of-Check to Time-Of-Use)** — a resource is checked for a property (e.g., file permissions, pointer validity) and then used, but the resource changes between the check and the use. CVE-2016-5195 (Dirty COW) is a kernel TOCTOU race condition that allowed unprivileged processes to write to read-only memory-mapped files, achieving privilege escalation.

**Missing input validation** — user-controlled input used without validation. An unvalidated length becomes a buffer overflow; an unvalidated format string becomes a format string vulnerability.

**State machine confusion** — a protocol allows impossible state transitions. TLS state machine bugs have allowed attackers to skip authentication steps or inject data into established connections.

---

## Concurrency Bugs

Concurrency bugs arise from incorrect synchronization between threads or processes accessing shared resources.

**Data races** — as covered in previous posts, concurrent read and write of the same memory location without synchronization is undefined behavior in C11/C++11. The compiler may optimize based on the assumption that data races do not occur, producing code that behaves incorrectly under concurrent access.

**Deadlocks** — threads wait on each other in a cycle that never resolves. In kernel code, deadlocks can be triggered by carefully ordered operations across interrupt handlers and process context.

**Lock-free programming errors** — incorrect memory ordering (e.g., `memory_order_relaxed` where `memory_order_acquire` is needed) produces data structures that appear correct in testing and fail catastrophically under specific scheduling conditions.

---

## Injection and Parsing Bugs

Injection bugs arise when untrusted input is interpreted as code or commands rather than data. Parsing bugs arise when a parser fails to correctly handle malformed or adversarial input.

**Format string vulnerabilities** — `printf(user_input)` instead of `printf("%s", user_input)`. Attacker-controlled format strings use `%n` to write to arbitrary addresses or `%x` to leak stack contents.

**Command injection** — user input is passed to a shell command without sanitization. `system("ping " + user_input)` with `user_input = "8.8.8.8; rm -rf /"` executes both commands.

**Parser differentials** — two components parse the same input differently. HTTP request smuggling exploits differential parsing between frontend proxies and backend servers to bypass security decisions.

---

## Information Disclosure

Information disclosure bugs leak data that should not be accessible, cryptographic material, memory addresses (defeating ASLR), or user data.

**Uninitialized memory reads** — reading memory before it is initialized leaks whatever was previously stored at that address. The Heartbleed vulnerability was fundamentally an information disclosure bug: the server echoed up to 64KB of its own memory in response to a malformed heartbeat request.

**Out-of-bounds reads** — reading beyond the end of a buffer exposes adjacent memory. Like Heartbleed, this can be used to leak cryptographic keys, session tokens, or other sensitive data.

**Side-channel leaks** — information leaked through timing or cache behavior rather than direct memory access. Spectre and Meltdown exploited CPU speculation to leak data across privilege boundaries.

---

## The Taxonomy in Practice

These bug classes are not independent. Real exploits frequently chain multiple classes:

1. An **information disclosure** bug leaks a heap address, defeating ASLR.
2. A **use-after-free** provides a write primitive at a controlled address.
3. A **type confusion** redirects a virtual function call to attacker-controlled data.
4. **Logic bugs** in the privilege check allow the exploit to complete.

Understanding which class provides which primitive, and how they chain, is the foundation of vulnerability research. Formal verification addresses each class at the root: abstract interpretation proves absence of memory corruption and integer errors; model checking verifies concurrency; deductive verification proves absence of logic bugs.

The bug is always in one of these classes. The question is whether you find it before the attacker does.
