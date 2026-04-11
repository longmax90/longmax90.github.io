---
layout: default
title: Does AI-Generated C/C++ Code Have Undefined Behavior?
date: 2026-04-12 00:00:00 +0000
categories: [software-security, undefined-behavior]
tags: [software-security, undefined-behavior, c, cpp, formal-verification, aicoding]
---

AI writes code faster than any human. Ask ChatGPT or GitHub Copilot for a C function to copy a buffer, parse a string, or handle a network packet — and you will get something that *looks* correct in seconds.

But looks correct and *is* correct are very different things in C and C++.

The question nobody is asking loudly enough: does AI-generated code have undefined behavior? And if it does, does it matter?

The answer to both questions is yes.

---

## Why LLMs Do Not Understand Undefined Behavior

Large language models like GPT-4 or GitHub Copilot are trained on billions of lines of code scraped from GitHub, Stack Overflow, and open-source repositories. They learn statistical patterns: what typically follows `malloc`, what a `for` loop usually looks like, how a `memcpy` is commonly written.

They do not reason about semantics. They do not model the C abstract machine. They do not know what the compiler is allowed to do when a signed integer overflows, or what happens to a pointer after `free`. They predict the next token based on what humans have written before.

The problem is that humans write a lot of buggy C code. And that buggy code is in the training data.

UB is particularly dangerous here because it is *invisible*. A signed integer overflow does not crash your program. An out-of-bounds read does not always produce garbage. A use-after-free does not always corrupt memory immediately. The code looks like it works — in testing, on your machine, at your optimization level. So it gets committed. So it ends up on GitHub. So it gets fed to the next LLM.

---

## What the Research Shows

The data from 2025 is unambiguous. Veracode's *2025 GenAI Code Security Report* — testing more than 100 LLMs across 80 curated coding tasks — found that **AI-generated code introduced security vulnerabilities in 45% of all test cases**. The failure rate for Java exceeded 70%. And critically: larger models performed no better than smaller ones, suggesting this is a systemic problem, not a scaling problem.

Apiiro's independent research across Fortune 50 enterprises reinforced the picture: CVSS 7.0+ vulnerabilities appeared 2.5× more often in AI-generated code, with privilege escalation paths jumping 322% and architectural design flaws spiking 153%.

The vulnerability classes are not exotic. They are the classics: buffer overflows, integer overflows, use-after-free, improper input validation. Exactly the categories that map directly to undefined behavior in C and C++.

---

## A Concrete Example — And Why Context Is Everything

Consider this prompt, given to a modern AI assistant:

> *"Write a C function that computes the total size of an array of elements, given the element count and element size."*

ChatGPT returned a correct function using `size_t` — and even provided an overflow-safe variant with a `SIZE_MAX` check. Impressive.

Now consider what happens in the real world. A developer is deep in a codebase, under deadline, and writes:

```c
void *buf = malloc(count * element_size);
```

No wrapper. Just inline arithmetic — `count` and `element_size` typed as `int` because that is what the surrounding code uses. The multiplication overflows signed `int` before `malloc` ever sees the result. **Undefined behavior.** The allocation succeeds with a truncated size. Every subsequent write beyond the actual buffer is out-of-bounds.

The lesson is not that AI cannot generate safe code — it can, when the prompt is explicit about safety. The lesson is that **most prompts are not**. Developers ask for functionality. They do not ask for security. And AI optimizes for what is asked.

The undefined behavior that matters in production is not the kind that appears in a carefully constructed security evaluation. It is the kind that appears in a routine allocation, a simple arithmetic expression, a helper function written at 11pm before a release. That code does not get a security-focused prompt. It gets "make this work."

In production, the prompt determines the output. And most prompts do not mention safety.

---

## Why This Is More Dangerous Than Ordinary UB

Human-written UB has always existed. The difference with AI-generated UB is **scale and trust**.

**Scale:** A senior engineer writes a few hundred lines of C per day. GitHub Copilot can assist thousands of developers simultaneously, each generating hundreds of lines. The volume of AI-assisted code being merged into production codebases is growing exponentially. If even a small percentage of that code contains UB, the absolute number of vulnerabilities introduced per day is unprecedented.

**Trust:** When a developer writes code themselves, there is at least implicit ownership and review. When an AI generates code, there is a documented tendency to *under-review* it — the assumption being that AI output has already been vetted in some way. It has not. The model has no concept of correctness. It has a concept of plausibility.

**Invisibility:** AI-generated code tends to look clean. Proper formatting, meaningful variable names, idiomatic structure. UB hiding inside clean-looking code is harder to catch than UB hiding inside obviously messy code. The aesthetics of correctness are not the same as correctness.

---

## The Only Real Answer: Formal Verification

Testing catches bugs that manifest on the inputs you test. Sanitizers (AddressSanitizer, UBSan) catch UB at runtime on the executions you run. Static analyzers catch patterns that match known bug signatures.

None of these scale to the problem of AI-generated code. They are all, in some sense, bounded by the inputs and scenarios a human thinks to test.

Formal verification is different. Abstract interpretation — the technology behind tools like TrustInSoft Analyzer and Frama-C — analyzes all possible executions simultaneously, providing a mathematical proof that certain classes of UB cannot occur on *any* input. Not "we did not find it." A proof.

As AI-generated code becomes a larger fraction of production C/C++ codebases, the demand for this kind of exhaustive verification is only going to grow. The irony is that AI is creating the problem and formal verification — a decades-old discipline — is one of the few tools capable of addressing it at scale.

---

## The Bigger Picture

We are at an inflection point. AI is changing how code is written, reviewed, and shipped. For high-level languages with strong runtime guarantees, this is mostly fine. For C and C++ — where the compiler assumes your code is correct and optimizes accordingly — it is a different story.

The undefined behavior that killed Ariane 5 and the Therac-25 was written by careful, experienced engineers. Imagine what happens when the code is written by a model that has never heard of the C abstract machine.

The answer is not to stop using AI tools. They are genuinely useful. The answer is to stop trusting them blindly — and to apply the same rigorous verification to AI-generated code that we should have been applying to human-generated code all along.
