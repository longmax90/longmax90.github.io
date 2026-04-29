---
layout: default
title: Choosing a Formal Verification Tool
date: 2026-05-17 00:00:00 +0000
categories: [software-security, undefined-behavior]
tags: [software-security, undefined-behavior, c, cpp, rust, formal-verification]
---

Formal verification is no longer an academic curiosity. Safety standards like DO-178C and ISO 26262 explicitly require it. The tools have matured significantly, from research prototypes to production-grade analyzers used on real safety-critical codebases.

But the landscape is fragmented. This post maps the tools available today for C/C++ and Rust, what they do, how they differ, and when to use each one.

---

## The Techniques Behind the Tools

Before the tools, a brief taxonomy of formal verification techniques, because the technique determines what a tool can and cannot prove.

**Abstract interpretation** computes an over-approximation of all possible program behaviors. It is sound (never misses a real bug) and fully automated. It trades some precision for scalability, it may produce false positives, but it terminates on real codebases. Best for: proving the absence of runtime errors (null dereferences, overflows, out-of-bounds accesses).

**Model checking** exhaustively explores all reachable states of a program. It is complete for bounded programs, it can prove the absence of bugs up to a certain depth. Best for: concurrency bugs, state machine verification, bounded safety properties.

**Deductive verification** requires the developer to write formal specifications (pre/post-conditions, invariants), then proves that the code satisfies them using an SMT solver or interactive proof assistant. Most precise, most effort. Best for: functional correctness proofs.

**MIRI / symbolic execution** instruments the program at runtime or explores it symbolically to detect UB. Not a static guarantee, but extremely effective at catching UB in test scenarios.

---

## C / C++ Tools

### TrustInSoft Analyzer

TrustInSoft Analyzer is a commercial tool based on abstract interpretation, built on top of the Frama-C platform. It is designed for safety-critical and security-critical C/C++ codebases, with explicit support for DO-178C and ISO 26262 certification workflows.

The April 2026 release (2026.04) adds AI-powered test driver and stub generation, MC/DC coverage via formal methods, and full Rust support through a GUI, making it one of the few tools that spans both C/C++ and Rust in a single workflow.

**Strengths:** Exhaustive UB detection, certification-ready reports, mixed C/C++/Rust analysis, GUI-driven workflow.
**Trade-offs:** Commercial license, setup effort proportional to codebase complexity.
**Best for:** Safety-critical firmware, DO-178C / ISO 26262 certification, mixed-language codebases.

---

### Frama-C

Frama-C is an open-source platform for C code analysis developed by CEA and Inria. It is not a single tool but a framework: a collection of plugins, each implementing a different analysis technique.

The most important plugins:
- **Eva** (Evolved Value Analysis), abstract interpretation for runtime error detection, the open-source cousin of TrustInSoft Analyzer's core engine.
- **WP** (Weakest Precondition), deductive verification using ACSL annotations and SMT solvers.
- **PathCrawler**, test case generation.

Frama-C is the standard entry point for formal verification research on C code, and it is widely used in academia and industrial R&D.

**Strengths:** Open-source, modular, research-grade precision, strong community.
**Trade-offs:** Steeper learning curve than commercial tools, no GUI by default, requires ACSL annotations for deductive verification.
**Best for:** Research, prototyping verification workflows, cost-sensitive projects.

---

### Astrée

Astrée is a commercial abstract interpreter specialized for safety-critical embedded C, particularly synchronous programs typical of avionics and automotive ECUs. It has been used to verify Airbus flight control software.

**Strengths:** Extremely low false positive rate on its target domain, certified tool qualification support.
**Trade-offs:** Narrow domain focus (not general-purpose), commercial license.
**Best for:** Avionics DO-178C DAL A, automotive ASIL D, synchronous embedded C.

---

### Polyspace (MathWorks)

Polyspace is MathWorks' static analysis and formal verification suite, combining abstract interpretation (Bug Finder) with formal proof (Code Prover), tightly integrated into MATLAB/Simulink workflows.

**Strengths:** Deep MATLAB/Simulink integration, MISRA C/C++ checking, widely adopted in automotive.
**Trade-offs:** Best value in MathWorks ecosystems, less suited to general embedded Linux stacks.
**Best for:** Model-based development, ISO 26262 workflows, MISRA compliance.

---

## Rust Tools

### Kani (Amazon Web Services)

Kani is an open-source model checker for Rust developed by AWS, built on CBMC. It can prove the absence of UB, panics, arithmetic overflows, and assertion failures in Rust code, including across `unsafe` boundaries. It is actively used by AWS to verify Rust components in production.

**Strengths:** Handles `unsafe`, no annotations required for basic properties, open-source, actively maintained.
**Trade-offs:** Bounded verification (completeness up to a depth), can time out on complex code.
**Best for:** Verifying `unsafe` Rust, AWS-style service code, standard library verification.

---

### Creusot

Creusot is a deductive verifier for Rust. It translates Rust code into a logical model and discharges proof obligations using the Why3 platform and SMT solvers. It supports rich functional correctness specifications via `#[requires]` and `#[ensures]` annotations.

Creusot is actively developed, with a January 2026 talk presenting new features and case studies. It is one of the most expressive Rust verification tools available.

**Strengths:** Full functional correctness proofs, handles complex data structures, expressive specification language.
**Trade-offs:** Requires significant annotation effort, steep learning curve, research-stage maturity.
**Best for:** High-assurance Rust libraries, cryptographic implementations, research.

---

### Aeneas

Aeneas translates Rust programs to proof assistants including Lean. It is used in Microsoft's effort to port SymCrypt, the cryptographic library used in Windows, Azure, and Xbox, from C to verified Rust.

**Strengths:** Eliminates memory reasoning for safe Rust, leverages Lean's proof ecosystem, industrial adoption.
**Trade-offs:** Primarily targets safe Rust, requires proof assistant expertise.
**Best for:** Cryptographic libraries, high-assurance safe Rust.

---

### Prusti

Prusti is a verifier for Rust built at ETH Zurich. It verifies programs against user-provided specifications and can prove absence of panics and functional correctness properties.

**Strengths:** Handles ownership and borrowing naturally, well-documented.
**Trade-offs:** Research-stage, limited `unsafe` support.
**Best for:** Safe Rust verification, teaching formal methods.

---

### MIRI

MIRI is Rust's official mid-level IR interpreter, built into the Rust toolchain. It detects UB in Rust code at runtime, including stacked borrows violations, uninitialized memory reads, and invalid pointer arithmetic inside `unsafe` blocks.

MIRI is not a static guarantee, it only catches UB on the executions that run. But it is indispensable for testing `unsafe` code and is zero-friction to use.

**Strengths:** No setup, zero false positives, detects subtle `unsafe` UB that sanitizers miss.
**Trade-offs:** Runtime only, not a proof.
**Best for:** Testing `unsafe` code during development, CI integration.

---

## Choosing the Right Tool

| Situation | Recommended tool(s) |
|:---|:---|
| Safety certification (DO-178C, ISO 26262) | TrustInSoft Analyzer, Astrée, Polyspace |
| Open-source C verification | Frama-C (Eva + WP) |
| Verifying `unsafe` Rust | Kani, TrustInSoft Analyzer |
| Functional correctness proofs in Rust | Creusot, Aeneas, Prusti |
| Daily `unsafe` code testing | MIRI |
| Mixed C/C++/Rust codebase | TrustInSoft Analyzer |

---

## The Honest Summary

No single tool covers everything. The right choice depends on your language, your domain, your certification requirements, and your team's capacity to write formal specifications.

The tooling landscape in 2026 is vastly better than five years ago, particularly for Rust. The gap between "tested" and "verified" is narrowing.
