---
layout: default
title: "What the Compiler Does When You Break the Rules - Part 2"
date: 2026-06-07 00:00:00 +0000
categories: [software-security, undefined-behavior]
tags: [software-security, undefined-behavior, c, cpp, compiler-optimizations]
---

Part 1 covered the five most widely exploited forms of UB. This part covers five less famous, but no less dangerous, categories: the ones developers are less likely to recognize, flag in review, or test for. The compiler is just as ruthless with them.

---

## 6. Strict Aliasing Violation

```c
int x = 42;
float *fp = (float *)&x;
float val = *fp;  // strict aliasing violation - UB
```

Strict aliasing is one of the most misunderstood forms of UB in C and C++. The rule: a pointer of one type may not alias an object of a different type (with exceptions like `char *`). The reason: if `float *` never aliases `int *`, the compiler can freely reorder loads and stores between them, a key optimization.

This shows up constantly in embedded and protocol parsing code:

```c
// Extracting a 32-bit integer from a byte buffer
uint32_t parse_uint32(uint8_t *buf) {
    return *(uint32_t *)buf;  // strict aliasing violation - UB
}
```

This pattern is everywhere. It is also undefined behavior. GCC with `-O2` and `-fstrict-aliasing` (enabled by default) may reorder or eliminate loads and stores around this access, producing results that have nothing to do with the bytes in the buffer.

The correct approach: `memcpy`, which the compiler will optimize to a single load on most architectures.

```c
uint32_t parse_uint32(uint8_t *buf) {
    uint32_t val;
    memcpy(&val, buf, sizeof(val));  // defined behavior, same performance
    return val;
}
```

**Compiler behavior summary:** With strict aliasing enabled, the compiler treats pointers of different types as non-overlapping. Writes through one pointer may not be visible through another, producing silent data corruption with no warning.

---

## 7. Shift by Negative or Excessive Amount

```c
int x = 1;
int y = x << 32;  // UB on 32-bit int: shift amount >= width
int z = x << -1;  // UB: negative shift amount
```

Bit shifts with out-of-range amounts are undefined behavior. On x86, the hardware masks the shift count to 5 bits, so `x << 32` produces `x` on x86. On ARM, it produces 0. On RISC-V, different again.

This is a portability trap in embedded code. Code that works correctly on one architecture silently produces wrong results on another, with no warning, no crash, no diagnostic.

The compiler exploits this: since a shift by an out-of-range amount is UB, the compiler may assume the shift amount is always in range and eliminate bounds checks on it that follow:

```c
if (shift_amount >= 32) return 0;  // range check
result = x << shift_amount;        // compiler may have removed the check above
```

MISRA C Rule 12.2 prohibits shift amounts that are negative or greater than or equal to the type width for exactly this reason.

**Compiler behavior summary:** Architecture-dependent hardware behavior. Shift amount range checks may be optimized away. Silent wrong results across platforms.

---

## 8. Reaching End of a Non-Void Function

```c
int compute(int x) {
    if (x > 0) {
        return x * 2;
    }
    // no return for x <= 0 - UB
}
```

If control reaches the end of a non-void function without a `return` statement, behavior is undefined. GCC warns about this with `-Wall`, but it is not a compile error, and in complex control flow, the missing return is not always obvious.

The compiler treats the missing path as unreachable. This means it may produce a function that returns whatever happens to be in the return register from a prior operation.

In security-sensitive code, this produces a subtle and dangerous failure mode: a function that is supposed to return an error code may silently return a success code, because the return register happens to contain zero from a previous operation. No crash. No log entry. The caller proceeds as if everything succeeded.

This is particularly common in error handling code, the paths least likely to be tested and most likely to contain missing returns.

**Compiler behavior summary:** The missing return path is treated as unreachable. The function may return a prior register value or trigger removal of preceding code.

---

## 9. Modifying a String Literal

```c
char *s = "hello";
s[0] = 'H';  // UB: string literals are read-only
```

String literals are stored in read-only memory. The compiler may deduplicate identical literals, so modifying one may corrupt others. The correct declaration is `const char *s = "hello"`, any modification becomes a compile-time error. MISRA C Rule 7.4 mandates `const`-qualified pointers for string literals.

**Compiler behavior summary:** Identical string literals may be merged into a single read-only location. Modifying one corrupts all references to the same literal. On modern platforms, typically a segfault.

---

## 10. Returning a Pointer to a Local Variable

```c
int *get_value() {
    int local = 42;
    return &local;  // UB: local is destroyed when function returns
}
```

The local variable `local` is allocated on the stack. When `get_value` returns, the stack frame is reclaimed. The returned pointer is now dangling, pointing to memory that has been logically freed, even though the bits may still be there.

The value may appear correct immediately after the call, the stack frame has not been overwritten yet, then silently change when the next function reuses the stack space. This often passes testing and only fails in production under different call patterns or optimization levels. GCC warns with `-Wreturn-local-addr`. Abstract interpretation detects it statically by tracking object lifetimes.

**Compiler behavior summary:** The stack frame is reclaimed on return. Higher optimization levels may reuse the stack slot earlier, making the bug manifest sooner and more reliably.

---

## The Common Thread

These ten categories, across both parts, share a common structure: the developer wrote code that assumed a certain behavior, the compiler assumed UB cannot occur and optimized accordingly, and the result is binary code that does something neither the developer nor the attacker fully predicted.

UB does not just help attackers. It makes the entire program harder to reason about for everyone. What it consistently does is remove safety checks and expose memory to corruption, silently, without warning, often only under specific conditions that testing never reaches.

Abstract interpretation handles all ten categories directly. It tracks value ranges (catching overflow and uninitialized reads), models pointer validity (catching null and dangling dereferences), enforces bounds on array accesses, flags strict aliasing violations, detects missing returns, and reasons about object lifetimes. Not as a test. As a proof.

The compiler assumes your code is correct. Formal verification checks whether it actually is.
