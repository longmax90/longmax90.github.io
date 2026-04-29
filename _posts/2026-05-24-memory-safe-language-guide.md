---
layout: default
title: Choosing a Memory-Safe Language
date: 2026-05-24 00:00:00 +0000
categories: [software-security, undefined-behavior]
tags: [software-security, undefined-behavior, c, cpp, rust, formal-verification, memory-safety]
---

In February 2024, the White House Office of the National Cyber Director issued a report with an unusually direct message for a government document: stop writing critical software in C and C++.

The report did not mince words. Memory safety issues were the second leading cause of exploited vulnerabilities in 2023. According to a Horizon3.ai analysis, 75% of memory-safety vulnerabilities were exploited as zero-days by threat actors. The solution, according to the ONCD, NSA, CISA, and a growing coalition of security agencies: migrate to memory-safe languages.

But what does "memory-safe" actually mean? And is switching languages really the answer?

---

## What Memory Safety Actually Means

A language is memory-safe if it prevents a class of bugs that arise from incorrect memory access:

- **Use-after-free**: accessing memory after it has been freed.
- **Buffer overflow**: writing beyond the bounds of an allocated buffer.
- **Null pointer dereference**: dereferencing a pointer to address zero.
- **Data races**: concurrent access to shared memory without synchronization.
- **Uninitialized reads**: reading memory before a value has been written to it.

In C and C++, all of these are possible, and all of them are undefined behavior. The compiler does not protect you. The runtime does not protect you. The programmer is solely responsible for correctness.

Memory-safe languages remove this responsibility by enforcing safety at the language or runtime level. But they do not all do it the same way.

---

## The C Approach: Discipline and Tooling

C does not prevent memory errors. It relies on developer discipline, coding standards (MISRA C, CERT C), static analysis tools, and runtime sanitizers to reduce the probability of errors occurring.

This approach has a 50-year track record. It works, imperfectly, at scale, with enormous engineering effort. The Linux kernel, OpenSSL, most embedded firmware, and virtually all safety-critical avionics software are written in C. These codebases are not safe because C is safe; they are safe because armies of engineers spend enormous resources finding and fixing bugs.

The problem is that this does not scale. As codebases grow, as development accelerates, as AI-generated code enters the picture, the discipline-based approach breaks down. Memory safety issues account for a large proportion of known exploited vulnerabilities, not because C developers are careless, but because the language makes certain bugs structurally inevitable at scale.

Formal verification, abstract interpretation and model checking, can provide mathematical guarantees about the absence of specific bug classes. But it requires significant investment, and it does not eliminate the underlying risk; it proves the absence of errors given a specific analysis scope.

---

## The Rust Approach: Safety by Construction

Rust takes a fundamentally different approach: it enforces memory safety at compile time, through the borrow checker and ownership system.

In safe Rust:

- Every value has exactly one owner.
- References cannot outlive the data they point to.
- Mutable references are exclusive, no data races by construction.
- There are no null pointers, `Option<T>` forces explicit handling.
- Array accesses are bounds-checked at runtime.

These guarantees are real and they are strong. A program written entirely in safe Rust cannot have use-after-free, data races, or null dereferences, by construction, enforced at compile time with no runtime overhead for most of these checks.

The White House ONCD report specifically mentioned Rust, and an earlier NSA report listed C#, Go, Java, Ruby, Swift, and others as memory-safe languages.

The practical catch: as discussed in a previous post, virtually no real-world Rust codebase is pure safe Rust. `unsafe` blocks exist for good reasons, FFI with C, hardware access, performance-critical data structures. And inside `unsafe`, the safety guarantees are suspended.

For most application-level code, Rust is an enormous improvement over C. For firmware and embedded systems with significant C interop, the picture is more nuanced.

---

## The Managed Runtime Approach: Java, C#, Go, Python

Languages like Java, C#, Go, Python, and Swift achieve memory safety through a different mechanism: garbage collection and runtime bounds checking.

- Memory is managed automatically, no manual `malloc` or `free`, no use-after-free.
- Array accesses throw exceptions rather than producing UB.
- Null pointer dereferences produce runtime exceptions, not undefined behavior.

This is effective for its target domain, web services, application software, tooling. But it comes with trade-offs that make it unsuitable for many embedded and firmware contexts:

- **Garbage collection pauses** are non-deterministic, unacceptable for real-time systems.
- **Runtime overhead** is significant, problematic for bare-metal firmware.
- **No direct hardware access**, managed runtimes abstract away the hardware.
- **Foreign function interface** with C still requires unsafe code at the boundary.

For a cloud service or a desktop application, Python or Go is often the right choice. For an RTOS running on a Cortex-M4 with 256KB of RAM, it is not.

---

## Ada/SPARK: The Forgotten Option

Ada and its formal subset SPARK are rarely mentioned in mainstream discussions of memory safety, but they deserve attention.

Ada was designed from the ground up for safety-critical systems. It has built-in support for strong typing, range-constrained types, tasking with formal synchronization semantics, and extensive runtime checks. SPARK, a formally verifiable subset of Ada, can be used with proof tools to provide mathematical guarantees of the absence of runtime errors.

Ada/SPARK is used in avionics (Airbus A380 flight management), military systems, and high-integrity embedded applications. It is not a popular language by any measure, but in domains where formal correctness matters more than ecosystem size, it remains one of the strongest options available.

---

## The Honest Map

No language eliminates all bugs. The question is where the risk sits and what tools are available to manage it.

| Language | Memory safety | Real-time suitable | Formal verification | Ecosystem |
|:---|:---|:---|:---|:---|
| C | No | Yes | Yes, mature tools | Enormous |
| C++ | No | Yes | Partial | Enormous |
| Rust | Yes, safe subset | Yes | Growing | Growing |
| Ada/SPARK | Yes | Yes | Mature | Niche |
| Go/Java/C# | Yes | No, GC pauses | Limited | Large |

---

## Where Does This Leave C?

CISA and the NSA's joint 2025 guide on memory-safe languages acknowledges that achieving better memory safety demands language-level protections, library support, robust tooling, and developer training. It does not mandate that all C code be rewritten immediately, that would be both impractical and counterproductive.

The realistic path forward is a combination: new projects in Rust or Ada/SPARK where feasible, formal verification for existing C codebases where rewriting is not an option, and a gradual migration informed by risk assessment rather than panic.

C is not going away. The firmware running most of the world's critical infrastructure will be C for decades. The question is not whether to use C, it is how to use it safely enough, with the right tools and verification processes in place.

That has always been the question. The difference now is that the tools to answer it are better than ever.
