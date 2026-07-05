---
layout: default
title: Undefined Behavior in Embedded Firmware
date: 2026-07-19 00:00:00 +0000
categories: [software-security, undefined-behavior]
tags: [software-security, undefined-behavior, c, cpp, formal-verification]
---

Everything covered so far — signed integer overflow, buffer overflows, use-after-free, compiler-assisted check removal — applies to all C and C++ code.

But embedded firmware is different. Not because the bugs are different. Because the environment makes them harder to find, harder to fix, and significantly easier to exploit.

---

## No OS, No Safety Net

When an application on Linux crashes due to a null pointer dereference, the OS catches it. The process terminates. The system keeps running.

In bare-metal firmware — no OS, no MMU, no process isolation — there is no safety net. A null dereference does not segfault. It reads or writes address zero, which on many microcontrollers maps to valid memory — the interrupt vector table or the start of flash. The program does not crash. It silently corrupts critical data and continues in an undefined state.

---

## No ASLR, No Stack Canaries, No NX

Modern operating systems deploy multiple layers of exploitation mitigations:

- **ASLR** randomizes memory layout, making it hard to predict addresses
- **Stack canaries** detect stack overflows before the return address is used
- **NX/DEP** prevents code execution in data regions

Most embedded environments have none of these. Memory layouts are fixed and documented in datasheets. Stack addresses are predictable. Flash and RAM are often both readable and executable.

A buffer overflow in embedded firmware is not just a bug — it is an immediately exploitable vulnerability. No ASLR to defeat. No ROP chain needed. Shellcode goes in the overflow buffer and the attacker jumps directly to it.

---

## Hardware Is Part of the Attack Surface

In application code, the program interacts with the hardware through OS abstractions — system calls, device drivers, file descriptors. The OS validates inputs and mediates hardware access.

In embedded firmware, the program talks directly to hardware through memory-mapped registers. A UART peripheral is accessed by writing to address `0x40004800`. A DMA controller is configured by writing to address `0x40020000`. There is no OS layer checking whether these writes make sense.

This creates several UB-adjacent problems:

**TOCTOU with hardware registers.** A firmware function might check a status register, then use the hardware based on that status — but hardware state can change between the check and the use. The hardware is a concurrent actor that firmware does not always account for.

**Unvalidated hardware input.** Data arriving from hardware peripherals — UART receive buffers, SPI data registers, CAN bus frames — is attacker-controlled if the hardware interface is accessible. Firmware that assumes hardware data is well-formed and skips validation is directly exploitable via the hardware interface.

**DMA and memory safety.** DMA controllers move data between memory and peripherals without CPU involvement. Incorrectly configured DMA can write to arbitrary memory addresses — including code regions in RAM — without any pointer dereference in the C code. This is a class of memory corruption that static analysis tools designed for application code often miss entirely.

---

## Real-Time Constraints Discourage Defensive Programming

Firmware often runs under hard real-time constraints — microsecond deadlines in automotive ECUs, cycle-level timing in motor controllers. These constraints create pressure against defensive programming: bounds checks, input validation, and error handling all cost cycles.

The result: firmware correct under normal operation, catastrophically vulnerable under adversarial input. That distinction did not matter on a closed network. It matters a great deal when the device has a USB port or a Bluetooth radio.

---

## Updates Are Hard — Bugs Persist for Years

When a vulnerability is discovered in a web application, the fix is deployed in minutes. When a vulnerability is discovered in embedded firmware, the situation is often very different.

Some devices cannot be updated at all — OTP memory, no remote access. Many others require physical access or authorized technicians. The patch may exist; deploying it to millions of field devices takes months or years. Update mechanisms are themselves an attack surface — a vulnerable parser can allow malicious firmware disguised as a legitimate update.

Low exploitation barriers plus slow patch cycles means firmware vulnerabilities have long effective lifespans. A buffer overflow discovered today may remain exploitable in vehicles on the road for a decade.

---

## The Formal Verification Argument Is Strongest Here

In application software, the argument for formal verification is: "testing and fuzzing miss bugs, and formal methods provide stronger guarantees."

In embedded firmware, the argument is stronger: "testing and fuzzing miss bugs, the standard runtime mitigations do not exist, exploitation barriers are low, and patches cannot be deployed quickly."

Every factor that makes UB dangerous in application code is amplified in embedded firmware. And several factors — direct hardware access, hard real-time constraints, long deployment lifetimes — are unique to embedded environments and have no analog in application security.

Abstract interpretation applied to embedded C firmware can prove the absence of runtime errors — null pointer dereferences, out-of-bounds accesses, integer overflows — before the firmware is ever flashed to a device. For safety-critical applications (automotive ASIL D, avionics DO-178C DAL A, medical IEC 62304 Class C), this is not optional — it is required by the certification standards.

For everything else, it is the difference between firmware that has been tested and firmware that has been verified. In an environment where there is no OS to catch errors and no easy way to push a fix, verified is the only acceptable standard.

---

## What This Looks Like in Practice

A typical embedded C codebase interacting with a UART peripheral:

```c
void uart_receive_handler(void) {
    uint8_t byte = UART->DR;           // read from hardware register
    rx_buffer[rx_index] = byte;        // write to buffer
    rx_index++;                        // increment index

    if (rx_index >= BUFFER_SIZE) {     // bounds check
        rx_index = 0;                  // reset — but data was already written
    }
}
```

Three issues in eight lines:

1. `UART->DR` is a hardware register — the read may have side effects (clearing interrupt flags). Reading it twice would be wrong, but the compiler might optimize multiple reads of the same address into one — unless `UART` is declared `volatile`.

2. `rx_buffer[rx_index] = byte` before the bounds check — the write happens before detecting overflow. If `rx_index == BUFFER_SIZE`, this is an out-of-bounds write.

3. `rx_index = 0` resets the index but does not clear the corrupted data. The next parse of `rx_buffer` will see a mix of old and new data — undefined behavior in the protocol state machine.

None of these are exotic edge cases. They are routine mistakes in interrupt handler code under time pressure. And in an embedded environment without OS protection, each one is directly exploitable.
