---
layout: default
title: "What the Compiler Does When You Break the Rules - Part 1"
date: 2026-05-31 00:00:00 +0000
categories: [software-security, undefined-behavior]
tags: [software-security, undefined-behavior, c, cpp, compiler-optimizations]
---

UB is not one thing, it is a family of distinct bug classes, each with its own mechanics, its own compiler interactions, and its own exploitation patterns.

Here are the first five categories, with concrete examples of what the compiler actually does with them.

---

## 1. Signed Integer Overflow

```c
int result = INT_MAX + 1;
```

In C, signed integer overflow is undefined behavior. The compiler is allowed to assume it never happens, and it does.

GCC and Clang use this assumption aggressively. Consider:

```c
int increment_and_check(int x) {
    if (x + 1 > x) {   // always true for non-overflowing ints
        return x + 1;
    }
    return -1;
}
```

The intent: detect overflow by checking if `x + 1 > x`. This is a common pattern in legacy code.

The reality: the compiler observes that `x + 1 > x` is always true for signed integers, because if it were not, overflow would have occurred, which is UB, which cannot happen in a correct program. The check is compiled away entirely. The function always returns `x + 1`, even when `x = INT_MAX`.

Compile with `-O2` and inspect the assembly. The branch is gone.

The correct fix: use unsigned arithmetic or `__builtin_add_overflow`. MISRA C Rule 10.1 prohibits overflowing signed integer operations for exactly this reason.

**Compiler behavior summary:** The compiler treats signed non-overflow as an invariant. Any code that only makes sense if overflow occurs may be silently removed.

---

## 2. Out-of-Bounds Array Access

```c
int arr[10];
int val = arr[10];  // off-by-one: valid indices are 0-9
```

Out-of-bounds access is UB. Unlike some languages that throw exceptions, C simply reads or writes whatever is in memory at that address.

The consequences depend on what is adjacent:

```c
void check_admin(char *password) {
    char buf[8];
    int is_admin = 0;

    strcpy(buf, password);   // no bounds check

    if (is_admin) {
        grant_access();
    }
}
```

`buf` and `is_admin` are both on the stack. A password longer than 8 bytes overflows `buf` and writes into adjacent stack memory, potentially overwriting `is_admin`. A password of exactly the right length sets `is_admin` to a non-zero value. Access granted.

This is a stack buffer overflow: out-of-bounds write is UB, and UB gives the attacker control over adjacent memory. GCC may also remove bounds checks it infers are redundant because a prior access at the same index implies (under the assumption of no UB) that the access was valid.

**Compiler behavior summary:** Out-of-bounds is assumed to never happen. Redundant checks may be removed. Adjacent memory is silently corrupted.

---

## 3. Use of Uninitialized Variables

```c
int x;
if (x > 0) {  // x is uninitialized - UB
    do_something();
}
```

Reading an uninitialized variable is UB. In practice, `x` contains whatever bytes happened to be in that stack location, from a previous function call, from the OS loader, from whatever ran before.

This is the basis of several exploitation techniques. On systems without stack randomization, stack contents after a context switch or system call are predictable. An attacker who can influence what ran before the vulnerable function can influence the value of the uninitialized variable.

More subtly: the compiler may optimize based on the assumption that the variable has a specific value. Consider:

```c
int result;
if (condition) {
    result = compute();
}
use(result);  // UB if condition was false
```

A compiler performing value range analysis may conclude that `result` is always the return value of `compute()`, because reading an uninitialized value is UB. It may then optimize `use(result)` based on the range of `compute()`'s return values, removing range checks that would otherwise catch out-of-bounds behavior downstream.

Tools like MemorySanitizer (MSan) detect uninitialized reads at runtime. Abstract interpretation detects them statically, without requiring specific inputs.

**Compiler behavior summary:** The compiler may assume the variable has been initialized. Downstream range checks based on the "impossible" uninitialized case may be removed.

---

## 4. Dereferencing a Null or Dangling Pointer

```c
int *ptr = NULL;
int val = *ptr;  // null dereference - UB
```

Null pointer dereference is UB. On most platforms, address zero is unmapped and the access produces a segfault, a crash. But this is a platform convention, not a language guarantee.

The compiler exploits null dereference UB in a well-documented pattern:

```c
void process(int *ptr) {
    *ptr = 42;          // dereference - implies ptr != NULL (or UB)
    if (ptr != NULL) {  // compiler: ptr is non-null (we already dereferenced it)
        do_secure_thing();
    }
}
```

The compiler observes: `ptr` was dereferenced unconditionally. If `ptr` were null, UB would have occurred, which cannot happen. Therefore `ptr != NULL` is always true, and the null check is redundant. The check is removed.

This is the exact mechanism behind CVE-2009-1897 in the Linux kernel. The null check that was supposed to prevent a security-sensitive operation was compiled away because a prior dereference implied the pointer was non-null.

Dangling pointers, pointers to freed memory, exhibit similar behavior. The compiler may cache values loaded through a pointer before `free()` and reuse them afterward, on the assumption that no one would read freed memory (UB), so the cached value is still valid.

**Compiler behavior summary:** A dereference implies non-null. Null checks after dereferences may be compiled away. Dangling pointer reads may return stale cached values.

---

## 5. Data Races in Multithreaded Code

```c
// Thread 1        // Thread 2
x = 1;             y = x;  // data race if unsynchronized
```

A data race, concurrent read and write of the same memory location without synchronization, is undefined behavior in C11 and C++11 and later. It is not just "non-deterministic behavior." It is UB, with all the compiler optimization implications that entails.

The compiler is allowed to assume no data races occur. This means:

```c
while (!done) {    // done is written by another thread
    do_work();
}
```

If `done` is not declared `volatile` or accessed through an atomic, the compiler may hoist the load of `done` out of the loop, reading it once before the loop starts and caching the value in a register. The loop becomes infinite, regardless of what the other thread does.

This is documented behavior of GCC and Clang on x86 and ARM. The fix is `std::atomic<bool> done` (C++) or `_Atomic bool done` (C11). Data races also produce torn reads on types wider than the platform word size: a 64-bit value on a 32-bit platform may be written as two separate operations, with a concurrent reader seeing a half-updated value.

**Compiler behavior summary:** The compiler caches values in registers across iterations. Synchronization-free concurrent accesses may produce stale reads or infinite loops.

---

## Five Down, Five to Go

These five categories are the most widely exploited forms of UB in production C and C++ code, behind Heartbleed, Linux kernel CVEs, and countless browser exploits.

Part 2 covers five less well-known categories that are in some ways more dangerous, because developers are less likely to recognize them: strict aliasing violations, shift overflow, missing return statements, string literal modification, and dangling stack pointers.
