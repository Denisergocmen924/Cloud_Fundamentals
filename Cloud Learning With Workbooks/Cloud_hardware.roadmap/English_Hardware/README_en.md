# The Physical Layer of the Computer — Offline Workbook

> **Cloud Engineer Fundamental Roadmap — Hardware**
> This is the **standalone, AI-mentor-free, internet-free** workable version of the
> [`Cloud_hardware_roadmap_fundamental_en.md`](../../../Cloud%20Learning%20With%20Mentor/English/Cloud_hardware_roadmap_fundamental_en.md)
> roadmap.

> 🇹🇷 Türkçe sürüm: **[README.md](../Turkish_Hardware/README.md)**

---

## What is this book, and how does it differ from the main map?

The roadmap file in the mentor folder is a **skeleton**: you give it to an AI mentor and
the mentor opens the topics in order. Read on its own, the skeleton tells you "what you need
to learn" but it does not "teach it to you".

This book is the **flesh** on that skeleton. The same phases, the same sub-items, the same
depth labels — but here every item is:

- **explained** (intuition → mechanism → numbers),
- **asked** (questions left for you to think about),
- **answered** (later in the same file),
- **connected** (to the previous item and to its counterpart in AWS).

Nowhere does it say "ask your mentor about this". You can work through the file from start
to finish on a plane, with no internet, entirely on your own.

---

## Files

| File | Phase | Topic | Approximate time |
|---|---|---|---|
| [Phase_0_Numeric_Foundation.md](Phase_0_Numeric_Foundation.md) | 0 | Binary, transistor, logic gates, flip-flop | 3–5 hours |
| [Phase_1_CPU_Architecture.md](Phase_1_CPU_Architecture.md) | 1 | ALU, fetch-decode-execute, clock, pipeline, core/vCPU, ISA | 5–7 hours |
| [Phase_2_Memory_Hierarchy.md](Phase_2_Memory_Hierarchy.md) | 2 | Cache, RAM, virtual memory, swap, NUMA | 7–10 hours |
| [Phase_3_Storage.md](Phase_3_Storage.md) | 3 | HDD, SSD, NVMe, IOPS/throughput/latency, RAID | 5–7 hours |
| [Phase_4_Buses_and_IO.md](Phase_4_Buses_and_IO.md) | 4 | Bus, PCIe, DMA, interrupt, chipset | 4–5 hours |
| [Phase_5_Network_Hardware.md](Phase_5_Network_Hardware.md) | 5 | NIC, switch, bandwidth/latency, RDMA | 3–4 hours |
| [Phase_6_Virtualization_Hardware.md](Phase_6_Virtualization_Hardware.md) | 6 | Hypervisor, VT-x, EPT, vCPU scheduling, SR-IOV, container | 6–8 hours |
| [Phase_7_Cloud_Connection.md](Phase_7_Cloud_Connection.md) | 7 | Instance families, Nitro, choosing EBS, noisy neighbors, bottleneck hunting | 5–7 hours |

**Appendices:**

| File | What it is for |
|---|---|
| [Appendix_A_Reference_Tables.md](Appendix_A_Reference_Tables.md) | The latency ladder, unit tables, PCIe/DDR/EBS numbers — keep it open on your desktop |
| [Appendix_B_Glossary.md](Appendix_B_Glossary.md) | Every term, a short definition + the section it appears in + easily confused pairs |

---

## How to study

### 1. Don't break the order

Every phase speaks in the **words** of the one before it. When Phase 2 explains "cache
miss", it uses the SRAM cell from Phase 0 and the pipeline stall from Phase 1. If you read
Phase 4 without Phase 2 you will understand the sentences but not the **why**.

The single exception: the `[concept]`-labelled parts of Phase 0 (logic gates, flip-flop) can
be gone through quickly — but they must **not be skipped**, because the whole of Phase 2
rests on them.

### 2. Take the boxes seriously

There are four kinds of box in the text. Each has a different job:

> **🤔 Think 2.3** — **Do not answer** the question here the moment you read it. Take a pen,
> think for 2 minutes, write down your guess. The answer is in the *"Answers to the think
> questions"* section at the end of the phase. If your guess turns out wrong, that is a good
> thing — a wrong guess makes the right answer permanent.

**❓ A question that comes to mind:** A question that arises naturally while reading. The
answer is **right below it**. These do not wait, because leaving them unanswered breaks the
flow of reading.

**⚠️ Common misconception:** Something most people get wrong. If you find yourself saying
"I thought that too" while reading these, mark that sentence.

**🔧 See it on your machine:** Optional. If you have a Linux machine, run it; if you don't,
the **expected output** is already written down — you can read it and move on. None of them
break your system (they are all read-only).

### 3. Obey the depth labels

| Label | What is expected | How you test it |
|---|---|---|
| `[concept]` | Being able to explain it intuitively | "Could I explain this to a friend in 30 seconds?" |
| `[mechanism]` | Being able to explain it step by step | "Could I draw the flow with pen and paper?" |
| `[application]` | Being able to tie it to an architectural decision | "Which instance / which disk — **why**?" |
| `[skip]` | Just knowing the name | Hearing the name and being able to say "that was in that area" is enough |

In this map, `[application]` **does not mean "run a command."** It means being able to
connect the topic to a concrete cloud decision. Don't move past an `[application]` item
until you can answer "which choice does this knowledge change?"

### 4. Don't skip the end of a phase

Every phase ends with:

1. **Answers to the think questions** — compare them with your own guesses
2. **Frequently asked questions** — the "but what about…" questions around the phase
3. **Test yourself** — an 18–22 question test + a full answer key
4. **Closing and bridge** — what is left of this phase, and where the next phase connects to it

If you score below 70% on the test, **instead of repeating the phase**, go back to the
section the question you got wrong points to. Every answer key tells you which section it
belongs to.

---

## Progress tracking

Track your own position here. Check the box when you finish a phase.

```
[ ] Phase 0 — Numeric Foundation
[ ] Phase 1 — CPU Architecture
[ ] Phase 2 — Memory Hierarchy            ← the most critical phase, don't rush
[ ] Phase 3 — Storage
[ ] Phase 4 — System Buses and I/O
[ ] Phase 5 — Network Hardware
[ ] Phase 6 — Virtualization Hardware     ← the phase of "aha" moments
[ ] Phase 7 — Cloud Connection            ← not a learning phase, a decision-making phase
```

---

## The goal and the limit of this book

**Goal:** When a cloud architecture decision lands in front of you — which instance family,
which EBS type, why is it slow, where is the bottleneck — being able to derive the answer
**from physics, not from memory.**

**Limit:** This book does not produce transistor designers, chip architects or kernel
developers. CMOS physics, VLSI and microcode design carry the `[skip]` label and have been
deliberately left out.

**Three instinct questions.** The whole of this book exists so that you can answer these
three questions without having to think:

1. **What is this workload limited by?** → CPU, memory, disk or network (Phases 1, 2, 3, 5, 7)
2. **What is physically underneath this instance?** → cores, cache, NUMA, hypervisor (Phases 1, 2, 6)
3. **Where is this slowness coming from?** → cache miss, swap, the IOPS ceiling, steal time, PCIe saturation (Phases 2, 3, 6, 7)

---

## The relationship to the main map

| This book | The main map |
|---|---|
| Offline, worked through alone | Given to an AI mentor |
| Explanation + question + answer | A topic list + mentor instructions |
| Fixed content | The mentor calibrates to the student's level |
| Test-yourself quizzes | The mentor examines you orally |

The two are **not rivals**. Sequential use is recommended: first work through the phase with
this book, then give the main map to an AI mentor and repeat the same phase **orally**. You
consolidate what you learned by reading when you explain it.

---

*This offline edition is based on Denis Ergöçmen's Cloud Engineer fundamental roadmap series.*
