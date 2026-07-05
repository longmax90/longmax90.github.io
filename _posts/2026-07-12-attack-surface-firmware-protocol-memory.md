---
layout: default
title: "Attack Surface: Firmware, Protocol, and Memory"
date: 2026-07-12 00:00:00 +0000
categories: [software-security, undefined-behavior]
tags: [software-security, undefined-behavior, c, cpp, exploit-development]
---

Before an attacker exploits a vulnerability, they ask one question: *where can I inject input that the program will actually use?*

That is the attack surface — every point where untrusted data enters a system and influences its behavior. The larger the attack surface, the more opportunities an attacker has. The better you understand it, the better you can prioritize where to look — whether you are attacking or defending.

In systems software, three attack surfaces dominate: **firmware**, **protocols**, and **memory**. Each has its own characteristics and its own bug patterns.

---

## What Makes Something an Attack Surface

Not all code is equally exposed. A function that only ever runs with hardcoded inputs has no attack surface. A function that parses a network packet has a large one.

Three properties define an attack surface point:

- **Reachability**: can an untrusted actor reach this code?
- **Influence**: can they control the input?
- **Impact**: if the code misbehaves, what can they achieve?

Reducing attack surface means reducing one of these three. Formal verification adds a fourth angle: proving that specific bug classes cannot occur on any input, regardless of what the attacker sends.

---

## Firmware Attack Surface

Firmware is software that runs directly on hardware — bootloaders, device drivers, UEFI, baseband processors, microcontroller code. It is often the first software that runs when a device powers on, and it operates at the highest privilege level on the system.

**Why attackers target firmware:**

A bug in a web application might give you access to one service. A bug in firmware gives you access to everything — including the ability to persist across operating system reinstalls, run below the visibility of security software, and disable hardware security features like Secure Boot.

The OS is the building. Firmware is the foundation. Own the foundation, own the building.

**Where the attack surface lives:**

*USB and peripheral interfaces* — when you plug in a USB device, firmware parses its descriptor to identify what kind of device it is. An adversarial USB device can send a malformed descriptor that triggers a buffer overflow in the parsing code — before the OS is even involved.

*Baseband processors* — the chip handling cellular communication runs its own firmware, separate from the main CPU. A vulnerability here allows remote code execution with no user interaction — just a specially crafted radio signal.

*Firmware update parsers* — when a device downloads and installs a firmware update, it parses a binary blob. A bug in the parser can allow an attacker to flash malicious firmware disguised as a legitimate update.

*Boot sequence* — UEFI reads partition tables and bootloader images at startup. Malformed structures have been used to exploit UEFI before the OS loads.

**The dominant bug classes:** buffer overflows in parser code, integer overflows in size calculations, use-after-free in device lifecycle management. Many firmware environments run without ASLR or stack canaries, making these bugs straightforward to exploit.

---

## Protocol Attack Surface

A protocol defines how two systems communicate. Every protocol implementation is an attack surface: the parser that decodes incoming messages, the state machine that tracks the conversation, and the handler that acts on the decoded data.

**Why attackers target protocols:** they are often exposed with no authentication required. Any attacker who can send a TCP packet to port 443 can reach your TLS parser.

**Where the attack surface lives:**

*Parsing* — every protocol message is a sequence of bytes that must be decoded into structured data. Parsers are full of size fields, length-prefixed strings, and type tags — all attacker-controlled. A missing bounds check in a parser is how Heartbleed happened: a length field declared by the client was used without validation, and the server echoed up to 64KB of its own memory in response.

*State machines* — protocols proceed through states. A TLS handshake goes: ClientHello → ServerHello → Certificate → Finished. A state machine bug allows the attacker to send messages out of order — skipping the authentication step, for example. The program processes the message because it does not check whether the state transition is valid.

*Deserialization* — many protocols use formats like JSON, XML, or ASN.1 to encode structured data. Deserializers that instantiate objects based on attacker-controlled type tags are a source of type confusion bugs — exactly the class covered in the previous post.

*Compression and encoding* — each additional encoding layer runs on attacker-controlled data. CRIME and BREACH exploited TLS compression to leak session tokens via timing.

**The dominant bug classes:** buffer overflows and out-of-bounds reads from parser bugs, logic bugs in state machines, information disclosure from uninitialized reads. Most high-profile protocol CVEs fall into one of these three.

---

## Memory Attack Surface

Memory is not an attack surface in the traditional sense — you cannot "send input" to memory. But understanding the memory layout of a target is what transforms a bug into a working exploit.

**Why memory layout matters:** every exploitation technique — stack overflow, use-after-free, type confusion — depends on knowing or predicting where things are in memory. The attack surface is not an input interface; it is the spatial relationship between objects.

**Where the attack surface lives:**

*Stack layout* — local variables, saved registers, and return addresses sit predictably on the stack. A buffer overflow that reaches the return address redirects execution. ASLR randomizes the layout — but an info leak that reveals one address breaks it.

*Heap layout* — the allocator places objects based on size and order. An attacker who controls allocation patterns can place a target object adjacent to a buffer about to overflow — heap grooming, a standard exploitation technique.

*Kernel/userspace boundary* — `copy_from_user` in the Linux kernel exists specifically to safely cross this boundary, validating that userspace pointers are actually in userspace. A missing call lets an attacker pass a kernel address disguised as a userspace pointer, causing the kernel to read or write its own memory.

*MMIO (Memory-Mapped I/O)* — in embedded systems, hardware peripherals appear as memory addresses. Code accessing MMIO without validating hardware state is exposed to race conditions and unexpected behavior from the hardware itself.

**The dominant bug classes:** information disclosure to leak addresses and defeat ASLR, memory corruption to write to controlled locations, use-after-free to redirect virtual dispatch. These classes are almost always chained: the leak enables the corruption, the corruption enables execution.

---

## Mapping Attack Surface Before You Start

A structured assessment before any audit:

1. **Enumerate inputs** — network packets, USB descriptors, IPC messages, hardware sensor readings.
2. **Trace data flow** — which parsers and state machines handle each input? Which memory regions does it reach?
3. **Identify privileged operations** — what can go wrong? Code execution, privilege escalation, info disclosure.
4. **Prioritize by impact** — unauthenticated network parsers first; post-authentication paths later.

This mapping determines where to invest deeper analysis. Formal verification applied to the highest-impact paths provides mathematical guarantees where they matter most.
