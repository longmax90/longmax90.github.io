---
layout: default
title: Why Safety-Critical Software Has Coding Standards?
date: 2026-05-03 00:00:00 +0000
categories: [software-security, safety-standards]
tags: [software-security, undefined-behavior, c, cpp, formal-verification, safety-standards]
---

If you work in embedded systems long enough, you will eventually encounter a sentence like: *"This codebase must be compliant with DO-178C DAL A"* or *"We need ISO 26262 ASIL D certification."*

And if you are like most developers encountering these standards for the first time, your reaction is somewhere between confusion and dread.

These standards are not bureaucratic box-ticking. They exist because software failures in safety-critical systems kill people. They encode decades of hard-won lessons from aerospace disasters, automotive recalls, and medical device malfunctions. Understanding what they require, and why, makes you a better engineer, regardless of whether your project formally requires them.

---

## Why Safety Standards Exist

In 1978, a software bug in the Therac-25 radiation therapy machine killed three patients and seriously injured three more. In 1996, a software error destroyed the Ariane 5 rocket 37 seconds after launch. In 2018 and 2019, software failures in the Boeing 737 MAX's MCAS system contributed to two crashes that killed 346 people.

Every major safety standard today was shaped by incidents like these, codified answers to the question: *"What would we have had to do differently to prevent this?"*

---

## IEC 61508: The Root Standard

Before diving into domain-specific standards, it helps to understand **IEC 61508**, the foundational standard for functional safety of electrical, electronic, and programmable electronic safety-related systems.

IEC 61508 defines a framework for managing safety throughout the lifecycle of a system. Its core concept is the **Safety Integrity Level (SIL)**, ranging from SIL 1 (lowest) to SIL 4 (highest). Most domain-specific standards, DO-178C, ISO 26262, IEC 62304, are derived from or influenced by it.

---

## DO-178C: Software in the Sky

**DO-178C** (*Software Considerations in Airborne Systems and Equipment Certification*) governs software development for civil aviation, recognized by the FAA and EASA worldwide.

DO-178C defines five **Design Assurance Levels (DAL)**, from DAL E (no safety effect) to **DAL A** (catastrophic, failure could cause loss of the aircraft). The higher the DAL, the more stringent the requirements.

For DAL A software, DO-178C requires:

- **Modified Condition/Decision Coverage (MC/DC)**: every condition in every decision must independently affect the outcome. This is a demanding coverage criterion that rules out most off-the-shelf testing approaches.
- **Structural coverage analysis**: demonstrating that no untested code paths exist.
- **Independence**: the development team and the verification team must be separate.
- **Traceability**: every line of code must be traceable to a requirement, and every requirement must be traceable to a test.

DO-178C does not mandate specific tools or languages, but it requires that any tool used for development or verification be qualified, meaning the tool itself must be shown to perform its intended function correctly.

The standard explicitly recognizes formal methods as a means of satisfying certain verification objectives. Abstract interpretation tools can be used to satisfy structural coverage requirements and to provide mathematical evidence of the absence of runtime errors.

---

## ISO 26262: Software in the Car

**ISO 26262** (*Road vehicles — Functional safety*) is the automotive equivalent of DO-178C. It defines **Automotive Safety Integrity Levels (ASIL)**, from ASIL A (lowest) to **ASIL D** (highest). Modern vehicles contain hundreds of ECUs running millions of lines of C and C++ code controlling brakes, steering, throttle, and airbags. A software failure in any of these can be fatal.

ISO 26262 requirements at ASIL D include:

- **Formal verification** is explicitly listed as a recommended method for software unit verification.
- **Static analysis** is required to demonstrate the absence of certain runtime errors.
- **Defensive programming**: code must be written to detect and handle erroneous inputs rather than assuming correct inputs.
- **Freedom from interference**: software components must be isolated so that a failure in one cannot corrupt another.

The standard also addresses the challenge of **ASIL decomposition**: splitting a safety requirement between two independent components, each at a lower ASIL level, to achieve the same safety integrity with more architectural flexibility.

---

## IEC 62304: Software in Medical Devices

**IEC 62304** governs the software lifecycle for medical devices, defining three Software Safety Classes (A, B, C). Class C, where failure can cause serious injury or death, requires detailed documentation, defined coverage criteria, and full requirements traceability. The Therac-25 failures were a direct catalyst for this standard.

---

## MISRA C: Coding Rules for Safety-Critical C

**MISRA C** (Motor Industry Software Reliability Association) is not a certification standard. It is a set of coding guidelines for C and C++ in safety-critical systems. Originally developed for the automotive industry, it is now used across aerospace, medical, and industrial domains.

MISRA C does not eliminate undefined behavior. It constrains the language to a safer subset, reducing the surface area where UB can occur and making code more amenable to static analysis and formal verification.

Key categories of MISRA C rules:

- **Mandatory rules**: violations are never acceptable. Examples: no implicit type conversions that could lose information, no use of dynamic memory allocation (`malloc`, `free`) in safety-critical code.
- **Required rules**: violations require documented justification. Examples: no recursion, limited use of pointers to functions.
- **Advisory rules**: best-practice recommendations.

MISRA C is often misunderstood as a replacement for testing or verification. It is not. A MISRA-compliant codebase can still contain undefined behavior. MISRA reduces risk, but does not eliminate it. Formal verification is still necessary to provide guarantees beyond what coding rules can offer.

---

## What These Standards Have in Common

Despite their different domains and terminologies, all of these standards share a common philosophy.

**Risk-based thinking.** Safety requirements are proportional to the severity of potential failure. A DAL A flight control system faces stricter requirements than a DAL D cabin lighting system.

**Traceability.** Every requirement must be traceable to code, and every piece of code must be traceable to a requirement. Nothing is written without a reason; nothing is required without being verified.

**Independence.** Development and verification must be separated. The person who writes the code cannot be the only person who checks it.

**Formal methods as a recognized technique.** DO-178C, ISO 26262, and IEC 62304 all explicitly recognize formal verification, abstract interpretation, model checking, theorem proving, as listed tools for satisfying verification objectives.

---

## The Gap Between Compliance and Safety

One important caveat: compliance with a safety standard is not the same as safety. A system can be fully DO-178C compliant and still fail. Standards define a process; they do not guarantee outcomes.

What they provide is a structured framework for reducing risk, documenting decisions, and demonstrating due diligence. The engineer who understands *why* these standards exist, not just what they require, is the one who can apply them intelligently rather than mechanically.
