---
layout: default
title: Does Rust Have Undefined Behavior?
date: 2026-05-10 00:00:00 +0000
categories: [software-security, undefined-behavior]
tags: [software-security, undefined-behavior, rust, formal-verification]
---

"Just rewrite it in Rust."

You've heard it. The implicit promise: Rust eliminates undefined behavior. Rust is memory-safe. Rust is the answer to the decades-long nightmare of C and C++ bugs.

And to be fair, Rust is genuinely impressive. The borrow checker is one of the most important innovations in systems programming in the last twenty years. It eliminates entire categories of bugs at compile time, for free, with no runtime overhead.

But "eliminates entire categories" is not the same as "eliminates all undefined behavior."

Let's talk about what Rust actually guarantees, and where the cracks are.

---

## What Safe Rust Actually Promises

The Rust compiler enforces a strict ownership model. In safe Rust, the following are impossible by construction:

- **Use-after-free**: the borrow checker ensures a value cannot be used after it has been moved or dropped.
- **Data races**: the `Send` and `Sync` traits make concurrent mutation of shared data a compile-time error.
- **Null pointer dereferences**: there are no null pointers in safe Rust. `Option<T>` forces explicit handling.
- **Buffer overflows**: slice indexing is bounds-checked at runtime.

This is real. These guarantees hold. If your entire codebase is safe Rust with no external dependencies, you are in genuinely better shape than equivalent C or C++.

But here is the thing: almost no real-world Rust codebase is pure safe Rust.

---

## Enter *unsafe*

Rust has an escape hatch: the `unsafe` keyword. Inside an `unsafe` block, the compiler's safety guarantees are suspended. The programmer is telling the compiler: *"Trust me. I know what I'm doing."*

`unsafe` is not a bug. It is a deliberate design decision. Some things genuinely require it:

- Calling C libraries via FFI (Foreign Function Interface).
- Writing device drivers or OS kernels with direct hardware access.
- Implementing performance-critical data structures that the borrow checker cannot reason about.
- Interfacing with raw memory or hardware-mapped registers in embedded systems.

The Rust standard library itself is full of `unsafe`. The `Vec<T>` implementation uses unsafe. `String`, `Box`, `Arc`, all use unsafe internally. The entire premise is that these `unsafe` internals are carefully encapsulated behind safe public APIs.

And most of the time, they are. But not always.

---

## Real Bugs: When *unsafe* Goes Wrong

**The `Vec::from_iter` double-free bug (2021)**

Issue [#83618](https://github.com/rust-lang/rust/issues/83618) in the Rust standard library: a double-free could be triggered by a `Drop` implementation that panics, using only `#![forbid(unsafe_code)]`. No `unsafe` in user code. The bug lived entirely inside the standard library's `unsafe` internals, behind a safe public API.

It was labeled `I-unsound` and `P-critical`. The guarantee had silently broken from the outside.

**`std::mem::transmute` misuse**

`transmute` is one of Rust's most dangerous functions. It reinterprets the raw bytes of a value as a different type: no checks, no conversion, pure bit-casting. It is the Rust equivalent of a C-style cast with all safety removed.

Misuse of `transmute` is a direct path to UB: reinterpreting a `u32` as a reference, transmuting between types of different sizes, or casting a value into an invalid enum variant. All of these are undefined behavior in Rust, and the compiler will not save you.

**The `MaybeUninit` footgun**

Reading from uninitialized memory is UB in Rust, just as in C. `MaybeUninit<T>` exists precisely to handle cases where a value is not yet initialized, but using it incorrectly (calling `.assume_init()` before the value is actually initialized) is silent UB, with no compiler warning.

---

## The Deeper Problem: Soundness

Here is a concept that even experienced Rust developers sometimes miss: **soundness**.

A Rust library is *sound* if it is impossible for safe code using that library to trigger undefined behavior. A library is *unsound* if a safe API can be used to produce UB.

Soundness holes are surprisingly common in the Rust ecosystem. The [RustSec Advisory Database](https://rustsec.org/) contains hundreds of advisories, many of which are soundness bugs: safe-looking APIs that, under specific conditions, produce undefined behavior.

Some examples from real crates:

- A safe `impl Send` on a type that was not actually safe to send across threads, causing data race UB.
- A safe indexing API that did not check bounds correctly, allowing out-of-bounds memory access.
- A safe abstraction over raw pointers that allowed aliasing violations, causing undefined behavior per Rust's memory model.

The pattern is always the same: `unsafe` code inside, safe API outside, incorrect invariant maintenance in between.

---

## Rust and Formal Verification

So where does this leave us?

Rust is a massive improvement over C and C++ for the average codebase. The borrow checker catches a class of bugs that would require significant manual effort, or formal verification, to eliminate in C. For most software, this is more than enough.

But for *critical* software, aerospace firmware, medical devices, cryptographic implementations, security-sensitive kernels, "most of the time safe" is not sufficient.

This is where formal verification enters, even for Rust:

- **TrustInSoft Analyzer 2026.04** brings Rust analysis fully into its GUI, project setup, root-cause investigation, and reporting, alongside AI-powered test driver generation and MC/DC coverage via formal methods.
- **Kani** (developed by Amazon Web Services) is a model checker for Rust that can prove the absence of UB, panics, and arithmetic overflows in Rust code, including across `unsafe` boundaries.
- **Creusot** uses Rust annotations to generate formal proofs of correctness.
- **Prusti** verifies Rust programs against user-provided specifications.
- **MIRI** is an interpreter that detects UB in Rust code at runtime, extremely useful for testing `unsafe` code, though it is not a static guarantee.

The gap between C and Rust tooling for formal verification is closing quickly.

---

## The Honest Summary

Rust gives you:

- **No UB in safe code**, if the standard library and your dependencies are sound.
- **Dramatically reduced UB surface** compared to C and C++.
- **Explicit, auditable `unsafe` blocks**, UB is at least localized and visible.

Rust does not give you:

- **Freedom from UB in `unsafe` blocks**.
- **Guarantees about third-party crates with unsound implementations**.
- **Protection against logic errors**, the borrow checker cannot verify that your algorithm is correct, only that it is memory-safe.

The rewrite-it-in-Rust narrative is compelling, and for most applications it is the right call. But for the highest-stakes software, the kind where a bug kills people or breaks cryptography, Rust is a better starting point, not a finishing line.

Formal verification is still needed. The tools are just getting better at speaking Rust.
