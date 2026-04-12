---
layout: default
title: When Fuzzing Misses Undefined Behavior
date: 2026-04-19 00:00:00 +0000
categories: [software-security, undefined-behavior]
tags: [software-security, undefined-behavior, c, cpp, formal-verification, fuzzing]
---

Fuzzing is one of the best bug-finding tools ever invented. Google's OSS-Fuzz has found tens of thousands of vulnerabilities in critical open-source software. libFuzzer, AFL++, and Honggfuzz are standard fixtures in any serious security team's toolkit.

And yet: a clean fuzzing run does not mean your C or C++ code is free of undefined behavior.

Not even close.

---

## What Fuzzing Actually Does

A fuzzer generates large volumes of inputs, random, mutated, or coverage-guided, and feeds them to your program. When the program crashes, hangs, or triggers a sanitizer alarm, the fuzzer flags it.

This is powerful. Fuzzing finds real bugs, quickly, at scale. It is especially effective at finding memory corruption bugs that manifest on specific inputs: buffer overflows triggered by malformed packets, integer overflows triggered by specific numeric values, use-after-free bugs triggered by particular sequences of operations.

But the key phrase is *triggered by*. Fuzzing can only find bugs that manifest on the inputs it generates. And the space of all possible inputs is, in most cases, astronomically large.

---

## The Coverage Problem

Modern coverage-guided fuzzers like libFuzzer track which code paths have been exercised and bias input generation toward unexplored paths. This is a significant improvement over purely random fuzzing.

But coverage is not correctness. Reaching a line of code is not the same as testing it under all conditions that matter.

Consider a simple example:

```c
int compute(int a, int b) {
    if (a > 0 && b > 0) {
        return a * b;  // signed integer overflow if a * b > INT_MAX
    }
    return 0;
}
```

A fuzzer will almost certainly reach the multiplication path. It will exercise it thousands of times. But the overflow only triggers when `a * b` exceeds `INT_MAX` - for 32-bit integers, that means inputs like `a = 50000, b = 50000`. A fuzzer starting from small seeds may never generate values in that range. The line is covered. The bug is invisible.

This is not a hypothetical limitation. It is a fundamental one.

---

## The Sanitizer Gap

In practice, fuzzing is most effective when paired with sanitizers: AddressSanitizer (ASan), UndefinedBehaviorSanitizer (UBSan), and MemorySanitizer (MSan). These instrument the binary to detect UB at runtime: null dereferences, out-of-bounds accesses, signed integer overflows, uninitialized reads.

With sanitizers, fuzzing becomes significantly more powerful. Many classes of UB that would be silent in a release build become loud crashes in a sanitized build.

But sanitizers only fire on the executions that actually run. A signed integer overflow on input `(50000, 50000)` will trigger UBSan, if the fuzzer ever generates that input. If it does not, the sanitizer never fires.

More subtly: some UB has no observable effect under common conditions. A compiler that exploits UB for optimization may produce code where the UB branch is simply removed, meaning neither the crash nor the sanitizer alarm ever occurs, even if the input is tested. The code "works." The UB is real. And the fuzzer has no way to know.

---

## The Path Explosion Problem

Some bugs only manifest after a specific sequence of operations, not on a single input, but on a series of interactions. A state machine with $N$ states and $M$ transitions can have $M^N$ distinct paths. For any non-trivial codebase, exhaustive path coverage is computationally infeasible.

Fuzzing handles this with heuristics: corpus distillation, structure-aware generation, grammar-based fuzzing. These are genuinely useful. But they are still heuristics. They explore the most *likely* paths, not all paths.

Undefined behavior hiding in a rarely executed path, an error handler, a boundary condition, a combination of flags that only occurs in production, can survive years of continuous fuzzing.

The Heartbleed vulnerability in OpenSSL existed for two years. The codebase was heavily tested. The bug required a specific combination of conditions that testing never reached.

---

## What Formal Verification Offers Instead

Abstract interpretation, the technology behind tools like TrustInSoft Analyzer and Frama-C, takes a fundamentally different approach.

Instead of running the program on specific inputs, it computes an abstract representation of *all possible executions simultaneously*. It does not sample the input space. It reasons about the entire input space at once.

For the `compute` example above:

```c
return a * b;  // a > 0, b > 0, both are signed int
```

Abstract interpretation asks: given that `a` is any positive integer and `b` is any positive integer, can `a * b` overflow a signed 32-bit integer? The answer is yes, and the tool reports it, without ever needing to find the specific inputs that trigger the overflow.

This is the difference between *testing* and *verification*:

- Testing says: "We ran 10 million inputs and found no bug."
- Verification says: "On any input, this class of bug cannot occur."

One is a probability. The other is a proof.

---

## Fuzzing and Formal Verification Are Complementary

This is not an argument against fuzzing. Fuzzing is fast, cheap, and extraordinarily good at finding shallow bugs quickly. For most codebases, a well-configured fuzzing setup will find the majority of easily triggered vulnerabilities.

The argument is about what fuzzing cannot guarantee, and why that matters for critical software.

In safety-critical domains, avionics, automotive firmware, medical devices, cryptographic libraries, the question is not "did we find any bugs?" but "can we prove the absence of this class of bug?" These are different questions, and only one of them has a definitive answer.

The pragmatic approach is to use both:

- **Fuzzing** to find bugs quickly during development, explore the input space, and catch regressions.
- **Formal verification** to provide mathematical guarantees about the absence of specific bug classes before deployment.

Fuzzing covers the ground floor. Formal verification covers the ceiling.

---

## The Honest Takeaway

A clean fuzzing run is good news. It means the most easily triggered bugs are probably not there. It means your code has been exercised under a wide variety of conditions.

It does not mean your code is free of undefined behavior. It means your fuzzer did not find any, and those are very different statements.

The undefined behavior that matters most is often the kind that hides in the corners: the overflow that only triggers at the boundary of the integer range, the use-after-free that only manifests under a specific allocation pattern, the uninitialized read that only appears in an obscure error path. These are exactly the bugs that formal verification is designed to find, and that fuzzing, by its nature, may never reach.

"We fuzzed it for a week and found nothing" is a good start. It is not a guarantee.
