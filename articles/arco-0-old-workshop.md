---
layout: post
title: "I Built a Smart Lock with AI. Then the Scope Proved the AI Wrong."
date: 2026-10-09
---

# I Built a Smart Lock with AI. Then the Scope Proved the AI Wrong.

## Part 0 — The Old Workshop and the New Hammer

> This is the first article in a five-part series documenting the ES242F Smart Lock project — a firmware retrofit of an OEM electronic lock using dual LLM agents, oscilloscope validation, and a method that emerged from failure, not theory.

---

### The Question

This project started with a question I didn't know I was asking: **Can one engineer, with the help of AI, build a system that used to require a team?**

The stack says it all: bare-metal C on an ESP32-S3, a TypeScript backend with tRPC and Drizzle, SQL schema design, MQTT broker configuration, Docker containerization, and a Flutter mobile app for virtual credentials. Just a few years ago, that would have needed a firmware engineer, a backend engineer, a DBA, a DevOps specialist, and a mobile developer. Five contexts. Five standups. Five opportunities for misalignment.

Today, a single LLM has cross-domain knowledge that no individual human possesses. It knows about `volatile` qualifiers and memory barriers in C. It knows about `async/await` and type narrowing in TypeScript. It knows about index optimization in SQL. It knows about multi-stage Docker builds. It can translate between these domains in a single conversation.

But — and this is the thesis the rest of this series will prove and qualify — **the AI knows all the languages, but it has no access to the physical reality they control.** It knows an ISR shouldn't block, but it cannot measure whether it actually does until the code runs on silicon. It knows a SQL transaction should be atomic, but it cannot detect a deadlock that only manifests under concurrent load until the test suite exercises the race. It knows a Docker container should have health checks, but it cannot observe whether the restart policy actually recovers the service until the orchestrator runs it under failure.

The AI is the virtual team. The human remains the architect — the only one who connects the probes when the scope shows a pulse that shouldn't exist, and the only one who interprets what that pulse means for the system as a whole.

This article is about how that division of labor emerged, why it was necessary, and what happens when you let a tool go faster than your understanding can follow.

---

### The Old Workshop

I need to start with some context: I am not a young developer discovering AI. I have been programming for more than 30 years. I started with raw binary on a CPD1802 — entering instructions bit by bit before I ever wrote assembly. Over the decades I have moved through HC11, AVR, PIC, x86, HC32, ARM Cortex-M4, and now ESP32-S3, C3, and P4. I have used ASM, C, C++, VB, Java, JavaScript, and everything in between. I have touched every generation of embedded development except one: I never used punched cards.

I am not telling you this so you will trust me. I am telling you this so you will understand why my reaction to AI was disciplined rather than enthusiastic or fearful. When you have spent decades watching a watchdog timer restart a system because of a race condition you introduced six months ago, you do not ask an AI to "make the firmware work." You ask it to respect the rules that silicon does not negotiate.

For the last ten years, I have followed the advances in machine learning closely. I was genuinely excited when backpropagation made deep learning practical — it ended the period known as the "AI Winter." When DeepMind's AlphaGo competed at the highest level of Go against a human champion, it confirmed that the path we were on was unstoppable. I even experimented with TinyML — running neural networks on microcontrollers to improve sensor filtering in embedded systems. I was never a skeptic. I was someone who was surprised by the speed of the jump from small networks to large ones, and quietly grateful that I would live to use them.

So when I started the ES242F project — retrofitting a smart lock with an ESP32-S3 — using AI assistance was not an experiment in hype. It was a tool choice. Like choosing a new oscilloscope. Like switching from a PIC to an ARM. You evaluate the tool, you understand its limitations, and you build a workflow around both its capabilities and its failure modes.

---

### The New Hammer

The project began conventionally. I had a Tuya-based electronic lock, no documentation, no API, no manufacturer support. The first phase was reconnaissance: understand the PCB, identify the MCU, trace the buses, figure out how the fingerprint reader talked to the main processor, how the motor driver was controlled, how the capacitive keypad scanned.

I used the AI as a research assistant. Not as an oracle. I would describe what I saw on the PCB — a chip with this marking, a bus with this pinout — and the AI would propose hypotheses. "That could be an SPI bus at 8 MHz." "That trace layout suggests a switching regulator." "That component is likely a flash memory chip with this capacity."

Every hypothesis required validation. The AI said "SPI"; the logic analyzer said "SPI at 8 MHz, CPOL=0, CPHA=0." The AI didn't lie. But it also didn't know until the sniffer confirmed it. This established a rule that would govern the entire project: **the AI proposes, the bench decides.**

![The Tuya PCB, annotated with identified chips: FR8018H, NZ3801, ZB25VQ64C, and others. Reverse engineering begins with technical empathy — understanding how the original designer thought.](pcba_tuya_annotated.png)

Once the PCB was mapped — chips identified, buses traced — the AI generated code faster than I could review it. That is where the trouble started.

---

### The Velocity Trap

In Stage 7.1 of the project, the AI generated 49,690 lines of C in 9 days. 27,355 lines of production code. 22,335 lines of tests and mocks. Manually, that would have taken months.

But that velocity created a problem that gets lost in the enthusiasm for productivity metrics: **the loss of functional context.**

When you write code manually, the process itself is a navigation system:

- It doesn't compile → you study the compiler output and learn the incorrect syntax.
- It doesn't link → you remember which library or module you forgot.
- It doesn't boot → the boot trace or an unlit LED tells you where the pointer is.
- It doesn't run → you debug step by step, and each step adds a coordinate to your mental map.

With AI, those intermediate steps disappear. It always compiles. It always links. It always boots. It accelerates productivity enormously, but at the cost of **educational friction**: the process destroys the mental map you used to build by traversing each stage.

This is not an abstract concern. It manifested as five concrete failures, all discovered in hardware testing, all invisible to the automated test suite:

1. **RF interference on the keypad:** The 13.56 MHz RFID antenna injected noise into the capacitive keypad's I2C bus, generating phantom keypresses. Host tests: green. Logic analyzer: problem found.
2. **Hidden button dead zone (1-3 seconds):** The reset button didn't respond in a specific interval after boot. Host test suite: 57/57 green. Human operator: failure.
3. **Audio truncation at 5 seconds:** The voice menu cut off mid-utterance because the timer counted from menu start, not from audio buffer drain. Invisible to host mocks.
4. **FreeRTOS queue race conditions:** Priority collisions between `app_task` (prio 20) and worker tasks (prio 12) caused silent command loss during credential bursts. Unit tests didn't reproduce the burst pattern.
5. **Stale binary incident:** The AI reported "all green" while compiling against an outdated binary on the host, hiding a real timer table overflow. The new code compiled against old symbols; the silicon crashed.

**The moral:** The tests said green; the bench said no. One hundred percent of functional failures were discovered on the hardware bench, not by automated suites. Host tests guard understood logic against regressions, but silicon and electromagnetic phenomena exhibit behaviors that simulators don't model.

![The hardware test bench: Rigol oscilloscope, logic analyzer, and the modified Tuya lock PCB. This is where the AI's claims are validated or refuted.](bench_overview_logic_analyzer.png)

A parallel story illustrates the same problem from software. In September 2026, a senior engineer named Ethan deleted 3,084 lines of AI-generated code that worked, passed tests, and had been approved by review. His reason: *"I understand all of this when I'm looking at it. I don't have the system in my head."* His manager turned off the monitor and asked what would happen if the payment provider failed after processing. Ethan couldn't answer without reading the code. Recognition is not recall.

Ethan solved the problem by deleting the code and rewriting it manually, step by step, so the mental map would form during construction. The second version was 800 lines shorter, with fewer generic abstractions, and he could explain it without opening the repository. That is a valid solution for backend software, where a branch can be deleted and rewritten in days.

Firmware is different. Each iteration costs: compile (5 minutes), flash (2 minutes), connect probes (5 minutes), run the runbook (30 minutes), discover the RF noise is still there (hours of debug). The cost of "destroy and rebuild" is measured in weeks, not days.

The ES242F method offers a third path. It doesn't try to avoid velocity, and it doesn't require destroying the code. It adds guardrails that rebuild the map without destroying the code: Phase 0 reconstructs context when a bug appears; containment audits prevent blind iteration when the task gets complicated; and the physical harness validates what the tests cannot. The code stays. The map gets rebuilt around it.

---

### The Method Emerges

The method that emerged across the duration of the project rests on four pillars. None came from theory. All emerged as responses to real problems:

**1. Separation of Powers**
A single AI agent hallucinated, forgot contracts, and approved its own buggy code. The solution was architectural, not prompt-engineering: split cognition into two roles. The **Control Authority (AC)** guards contracts, governs architecture, issues rulings. It never touches code. The **Coder Agent (KC)** executes tactically, contained in a sandbox. It never decides what to do; only how to do it.

**2. Phase 0 — Plan Before Touching**
Before writing a single line of code, KC must: cite the exact file and line of the problem; reproduce the root cause with evidence; propose the minimal solution; and **STOP** for AC approval. No exceptions. Not even for a "one-line mini-fix." Phase 0 is the frontier between "the AI proposes" and "the AI executes."

This exists because when the AI generates 50,000 lines in 9 days, the engineer loses the mental map. When a bug appears, you no longer "intuit" which module it's in. Phase 0 forces the AI to rebuild that map before touching anything: cite file, cite line, explain why.

**3. Containment Audits**
When a task gets complicated — cascading failures, unexpected behaviors, misaligned contracts — KC doesn't iterate blindly. It stops work, emits an audit report, and escalates to AC. AC reviews the plan, issues a ruling (a typed decision from the project's decision log, recorded as D-numbers like D61 or D78), and only then does KC receive "proceed" to continue.

This also responds to velocity: when the AI generates code faster than the human can absorb, the human cannot evaluate whether iteration #5 is on the right track. The audit forces a pause where the human (via AC) recalibrates before the agent keeps digging.

**4. Context Architecture**
LLMs operate with finite context windows (typically 128K-256K tokens). When a project reaches 50K lines, the model begins to "drown" in its own context. Documented phenomena include "lost in the middle" (attention decays for information in the center of long context) and context dilution (precision degrades as context grows).

The solution was not a longer prompt. It was a **context architecture** that fragments the project into manageable units:

- **Stages:** Each stage is a closed context. When it ends, its output (contracts, runbooks, architecture) becomes "frozen context" that is not reopened.
- **Isolated architecture docs (`app_xxx_architecture.md`):** Each app has its own architecture document, born from a dedicated conversation thread. It is not "contaminated" by conversations from other apps.
- **Separate conversation threads:** Each app/stage has its own thread. The PROJECT_NOTES.md and previous harnesses act as "project context" imported selectively, not as an infinite dump of all history.
- **PROJECT_NOTES.md as institutional memory:** Not a chat log. A living contract containing only typed decisions (D-numbers), active contracts, and runbooks. The model reads it at the start of each session; it does not drag it through every prompt.

This context architecture is what allows the method to scale. Without it, a 50K-line project would collapse into a single monolithic prompt where the model forgot the decisions from the beginning before reaching the end.

---

The same method was later applied to the distributed backend (tRPC, Drizzle, Hono), with a different oracle: 293 deterministic tests instead of an oscilloscope. The institutional architecture stayed identical. Only the source of truth changed. But that is a story for a later article.

---

### What This Series Is Not

This is not a tutorial on "how to use AI for embedded development." It is not a benchmark comparing LLMs. It is not a claim that AI replaces engineers.

It is a case study. A single project, documented with evidence: scope traces, bus logs, runbook results, CI screenshots, typed decision records. Every claim is backed by something reproducible on a bench. There are no "it probably works" or "it should work" statements. There is only "the scope said" or "the test refuted it."

The next article will cover how we studied unknown hardware — a Tuya lock with no documentation — and why "reverse engineering" is better understood as "technical empathy": learning not just how the device works, but how the person who built it thought.

---

*The ES242F project is documented in real time. All evidence, runbooks, and decision records are available in the project repository. This series is written as the project advances; each article is published when the evidence that supports it has been validated on the bench.*