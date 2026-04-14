---
layout: default
title: How Abstract Interpretation Rules Out Undefined Behavior?
date: 2026-04-26 00:00:00 +0000
categories: [software-security, undefined-behavior]
tags: [software-security, undefined-behavior, c, cpp, formal-verification, abstract-interpretation]
---

Imagine you are trying to prove that a bridge is safe. You could test it by driving cars over it, thousands of cars, different weights, different speeds. After enough tests, you might feel confident. But you have not proven the bridge is safe. You have proven it survived the cars you tested.

Abstract interpretation takes a different approach. Instead of testing specific cars, it reasons about *all possible cars at once*, using mathematics rather than experiments.

This is not magic. It is a precise, well-understood technique. And once you understand how it works, the gap between testing and verification becomes very clear.

---

## The Problem With Testing

Every testing approach, unit tests, fuzzing, sanitizers, shares the same fundamental limitation: it can only check the executions that actually run.

Consider this function:

```c
int safe_divide(int a, int b) {
    return a / b;
}
```

Division by zero is undefined behavior in C. To catch it with testing, you need to call `safe_divide` with `b = 0`. If your test suite never generates that input, the bug is invisible, even after millions of test runs.

The input space of most programs is astronomically large. No test suite can cover it entirely. Abstract interpretation sidesteps this limitation.

---

## The Core Idea: Abstract Domains

Abstract interpretation works by replacing concrete values with *abstract values*, compact representations that describe entire sets of possible values at once.

Instead of tracking that `b = 5`, an abstract interpreter might track that `b` is in `[1, 100]`, meaning `b` is some integer between 1 and 100. Instead of computing `a / b = 20`, it computes what the result could be for *any* value of `a` and `b` in their respective abstract ranges.

The abstract domain is the choice of representation. Common examples:

- **Interval domain**: values are represented as ranges `[min, max]`. Simple, fast, sufficient for many analyses.
- **Octagon domain**: tracks relationships between pairs of variables, for example `x - y <= 5`. More precise, more expensive.
- **Polyhedra domain**: tracks arbitrary linear relationships between variables. Most precise, most expensive.

Each domain trades precision for performance. The choice depends on what you need to prove.

---

## A Concrete Walkthrough

Consider:

```c
void process(int x) {
    int y = x * 2;
    int z = y + 10;
    int result = 100 / z;  // potential division by zero?
}
```

**Step 1: Abstract the input.**
We do not know what `x` will be at runtime. So we represent it abstractly: `x` can be any integer.

**Step 2: Propagate through operations.**
- `y = x * 2` means `y` can also be any integer.
- `z = y + 10` means `z` can also be any integer.

**Step 3: Check the safety property.**
Is `z = 0` possible? Yes. The abstract interpreter reports a potential division by zero.

Now the analyzer raises an alarm. A developer can look at the code and decide: is `z = 0` actually reachable? In this case, yes. If `x = -5`, then `y = -10`, `z = 0`, and dividing by zero is real UB.

But what if the function had a precondition?

```c
void process(int x) {
    if (x <= 0) return;  // guard: x is always positive
    int y = x * 2;
    int z = y + 10;
    int result = 100 / z;
}
```

Now the abstract interpreter re-runs with the guard in place:
- After the check: `x` is at least 1.
- `y = x * 2` means `y` is at least 2.
- `z = y + 10` means `z` is at least 12.

Is `z = 0` possible? No. The division by zero cannot occur.

This is a proof, not a test result. It holds for *every possible value of `x`* that satisfies the precondition, not just the values we happened to test.

---

## Soundness: The Key Guarantee

Abstract interpretation is *sound*: it never misses a real bug. If a division by zero is possible, the analyzer will report it, or report something that covers it. It may sometimes report alarms that turn out to be false. This is called a false positive. But it will not silently ignore a real problem.

This soundness guarantee comes from a deliberate design choice: when the abstract domain cannot represent a value precisely, it *over-approximates*. It assumes the worst case. This is conservative, which means it may produce more alarms than strictly necessary, but it guarantees completeness.

Contrast this with testing: a test suite is neither sound nor complete. It can miss real bugs, and it cannot provide guarantees about untested inputs.

---

## Handling Loops and Branches

One of the hardest problems in program analysis is loops. Abstract interpretation handles this with a technique called *widening*. When a loop is detected, the analyzer accelerates convergence by deliberately over-approximating the loop's effect. The result is a conservative abstraction that is valid for any number of loop iterations, including zero and unbounded execution.

For a loop that increments a counter from 0 to `n`, instead of tracing every possible execution, widening produces a range such as `i` in `[0, +infinity)`. Conservative, but sound. The analysis terminates in finite time, even for unbounded loops.

---

## Why This Matters for Security

In security-critical software, firmware, cryptographic libraries, automotive ECUs, the question is not "did we find any bugs?" but "can we guarantee the absence of this class of bug?"

Tools like TrustInSoft Analyzer, Frama-C, and Astrée use abstract interpretation to provide mathematical proof that specific classes of undefined behavior, null pointer dereferences, out-of-bounds accesses, integer overflows, uninitialized reads, cannot occur on any input.

This is what sets formal verification apart from every other testing approach. It does not explore the input space. It reasons about it in its entirety, at once, with mathematical rigor.

Testing says: "We ran 10 million inputs and found no bug."
Abstract interpretation says: "On any input, this bug cannot occur."

One is confidence. The other is proof.

---

## The Trade-Offs

Abstract interpretation is not free. The more precise the abstract domain, the more expensive the analysis, in time and memory. Real-world tools make careful choices about which domains to use for which properties, and how to balance precision against scalability.

False positives are an inherent cost of soundness. Reducing them requires more precise domains or developer annotations. This is engineering work, not a fundamental limitation.

Abstract interpretation does not verify functional correctness. It tells you whether code crashes, reads uninitialized memory, or overflows an integer. For critical software, that is often exactly what you need.
