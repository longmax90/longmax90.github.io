---
layout: default
title: "Undefined Behavior: The Silent Killer in Critical Software"
date: 2026-04-05 00:00:00 +0000
categories: [software-security, undefined-behavior]
tags: [software-security, undefined-behavior, c, cpp, formal-verification]
---

You have probably heard the phrase "undefined behavior" thrown around in C and C++ circles. Most developers treat it as a theoretical concern, something that only matters in edge cases, something that "works fine in practice."

Then a rocket explodes. Or a hospital patient gets a lethal radiation dose. Or an entire fleet of cars becomes remotely hackable.

Undefined behavior (UB) is not theoretical. It has killed people. It has cost billions of dollars. And it is still lurking in critical software running your car, your pacemaker, and your phone's firmware today.

Let me show you why.

---

## What Is Undefined Behavior?

In C and C++, certain operations have no defined semantics in the language standard. When your program triggers one of these operations, anything can happen, and the compiler is not obligated to warn you.

The compiler can:

- Produce the result you expected.
- Produce a completely different result.
- Delete entire code paths it deems unreachable.
- Silently corrupt memory.

Common sources of UB:

- Signed integer overflow.
- Out-of-bounds array access.
- Use of uninitialized variables.
- Dereferencing a null or dangling pointer.
- Data races in multithreaded code.

The insidious part is that UB often appears to work during development and testing, only to manifest catastrophically under specific conditions: a different compiler version, a higher optimization level, a different CPU architecture, or an unexpected input at runtime.

---

## Ariane 5: When a Type Conversion Destroyed EUR370 Million

On June 4, 1996, the Ariane 5 rocket exploded 37 seconds after launch. No lives were lost, but the payload, four scientific satellites representing a decade of work, was destroyed. Total cost: approximately EUR370 million.

The cause was a single unhandled numeric conversion error in the inertial reference system software, originally written for Ariane 4.

A 64-bit floating-point number representing horizontal velocity was converted to a 16-bit signed integer. On Ariane 4, this value never exceeded the range of a 16-bit integer. On Ariane 5, with its higher acceleration profile, the value was too large. The conversion overflowed, threw an operand error, and the backup system (running identical software) failed in exactly the same way.

The flight computer interpreted the error diagnostic as flight data. It commanded the rocket to perform a violent course correction that tore it apart.

This was not a hardware failure. It was not a logic error in the traditional sense. It was the assumption that a value would never exceed a numeric range, an assumption that was valid for one rocket and fatal for another.

---

## Therac-25: When UB Gave Patients Lethal Radiation Doses

Between 1985 and 1987, the Therac-25 radiation therapy machine delivered massive radiation overdoses to at least six patients. Three died directly as a result.

The Therac-25 was a computerized system that removed the hardware safety interlocks of its predecessor, relying entirely on software. A race condition, a classic form of UB in concurrent code, allowed the machine to enter a state where the high-power electron beam fired without the beam spreader in place.

The tragic irony: the bug only triggered when operators typed quickly. During testing, operators typed slowly. The race condition never manifested.

The software had been reused from earlier models with minimal review, the development team had no formal safety process, and there was no mechanism to detect that the machine had entered an impossible state.

The Therac-25 case is why DO-178C, IEC 61508, and ISO 26262 exist today, safety standards that now govern avionics, industrial, and automotive software. All of them, in some form, require demonstrating the absence of certain classes of runtime errors.

---

## The Linux Kernel Null Pointer Optimization Bug

This one is more subtle, and more modern.

In 2009, a class of Linux kernel vulnerabilities was discovered where null pointer dereferences, normally expected to crash the program, were being silently optimized away by GCC.

The sequence worked like this: a pointer `p` was dereferenced early in a function (UB if null). Later, the code checked `if (p != NULL)`. Because the compiler had already seen a dereference of `p`, it concluded that `p` must be non-null (otherwise the program would already be in UB territory), and optimized out the null check entirely.

An attacker could map memory at address zero on Linux (then possible via `mmap`), place shellcode there, trigger the null dereference, and the kernel would execute the attacker's code, with no null check to stop it.

The compiler was technically correct. The programmer had written UB. The result was a privilege escalation vulnerability in the Linux kernel.

This is what makes UB so dangerous in security contexts: the compiler becomes your adversary. Optimizations that are mathematically valid given the standard's semantics can destroy security-critical logic that was never formally verified.

---

## Why This Matters More Than Ever

Modern codebases are not getting safer. They are getting larger, more interconnected, and deployed in higher-stakes environments.

- Automotive firmware: millions of lines of C in ECUs controlling brakes, steering, and throttle. MISRA C reduces UB surface but does not eliminate it.
- Mobile firmware: baseband processors, secure enclaves, and bootloaders written in C and C++ with direct hardware access and no OS-level protection.
- Cryptographic libraries: OpenSSL, BoringSSL, mbedTLS, all C codebases where a single UB instance can leak private keys or bypass authentication.

The traditional defense has been testing and fuzzing. These are valuable but fundamentally incomplete, they can only find bugs that manifest on the inputs you test.

Formal verification, and specifically abstract interpretation, offers something different: a mathematical proof that certain classes of UB cannot occur on any input. Not "we did not find it." Not "it did not crash in ten million test cases." A proof.

That is the gap between software that has been tested and software that has been verified.

---

## Where Do We Go From Here?

The good news is that the industry is starting to take this seriously:

- Rust was designed from the ground up to eliminate entire categories of UB at the type-system level.
- CompCert produces formally verified compiled C code.
- TrustInSoft Analyzer, Frama-C, and similar tools based on abstract interpretation can verify the absence of runtime errors in existing C/C++ codebases.
- CHERI hardware capabilities offer hardware-enforced memory safety.

The bad news is that the vast majority of critical embedded and firmware code is still plain C, compiled with aggressive optimizations, and tested rather than verified.

Ariane 5 exploded in 1996. The Therac-25 killed patients in 1987. We still write C. We still have UB. We still assume that "it works on my machine" is good enough.

It is not.
