# Phase 1 — CPU Architecture

> **Navigation:** [◀ Phase 0 — Numeric Foundation](Phase_0_Numeric_Foundation.md) · **Phase 1** · [Phase 2 — Memory Hierarchy ▶](Phase_2_Memory_Hierarchy.md)

---

## Where we are coming from

In Phase 0 we built a chain: **transistor → logic gate → adder + feedback → flip-flop →
register.** At the end of the chain we had a container that can hold 64 bits and a circuit
that can add numbers.

Now we combine those parts and build the **CPU**. And more importantly: once we've built the
CPU in this phase, you'll be able to read "8 vCPUs, up to 3.2 GHz" in the AWS console not as a
memorized label but **knowing what lies underneath it**.

## The question of this phase

The decision a cloud engineer makes most often — and most often gets wrong — is this:

> *"My application is slow. Should I get more vCPUs, a faster CPU, or is the problem not the
> CPU at all?"*

Which of these three options is right depends on the sum of this phase's six sections. By the
end you'll be able to answer this question **by reasoning, not by guessing**.

## By the end of this phase

- You will be able to explain, step by step, the full journey of an instruction inside the CPU
- You will be able to explain the difference between "3 GHz" and "fast" using the concept of IPC
- You will know why the pipeline exists and when it clogs
- You will be able to answer **"`c7i.4xlarge` and `c7g.4xlarge` both have 16 vCPUs — but how
  many physical cores?"** (and the answer will likely surprise you)
- You will be able to say what moving to Graviton gains you, what it costs, and when it
  doesn't make sense

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 1.1 | Components of a CPU | `[concept]` / `[mechanism]` | Know the parts |
| 1.2 | Fetch-Decode-Execute | `[mechanism]` | How the parts work together |
| 1.3 | Clock and IPC | `[mechanism]` / `[application]` | What "fast" means |
| 1.4 | Pipeline | `[concept]` | Why GHz alone is meaningless |
| 1.5 | Core, thread, vCPU | `[mechanism]` / `[application]` | **What your bill pays for** |
| 1.6 | ISA and CPU families | `[concept]` / `[application]` | The Graviton decision |

---
---

# 1.1 The Core Components of a CPU

Let's start by splitting a CPU into five parts. These haven't changed since the 1970s — modern
CPUs became more complex by **multiplying and deepening** these five, but the skeleton is the
same.

![Figure 1.1 — Basic block diagram of a CPU](diagrams/hw-1-01-cpu-blocks.png)
*Figure 1.1 — Data and control flow between the Control Unit, ALU, register file, PC and IR.
Thick arrows are data paths; thin arrows are control signals.*

## 1.1.1 ALU — Arithmetic Logic Unit `[concept]`

The **ALU** (Arithmetic Logic Unit) is the part of the CPU that **does the calculating**.

In Phase 0.3.3 we chained 64 full adders to build an addition circuit. The ALU is that circuit
with a few siblings placed beside it:

| Operation class | Examples | Counterpart in Phase 0 |
|---|---|---|
| Arithmetic | addition, subtraction, multiplication, division | 0.3.3 adder |
| Logic | AND, OR, XOR, NOT | 0.3.1, 0.3.2 gates |
| Shift | shift left/right (`<<`, `>>`) | shifting bits = multiplying/dividing by 2 |
| Comparison | equal?, greater? | XOR + zero check |

**How is subtraction done?** There is no separate subtractor circuit. With the two's complement
trick, subtraction is turned into addition:

```
A - B  =  A + (-B)  =  A + (NOT B + 1)
```

So invert all the bits of B (NOT gates), add 1, then feed it to the normal adder. One circuit,
two operations. **This is the soul of hardware design:** instead of adding a separate circuit,
choose a representation that reuses the existing one.

**Flags.** After every operation the ALU also produces a few bits about the result and keeps
them in a special register (`RFLAGS` on x86):

| Flag | Meaning | Where it is used |
|---|---|---|
| **ZF** (Zero) | Is the result zero? | `if (a == b)` — really `a-b` is computed and ZF is checked |
| **CF** (Carry) | Did a carry occur (unsigned)? | Big-number arithmetic |
| **SF** (Sign) | Is the result negative? | `if (a < 0)` |
| **OF** (Overflow) | Signed overflow | Integer overflow bugs |

This is the hardware counterpart of the `if` statement in programming: **a comparison is really
a subtraction, and a decision is looking at a flag bit.**

## 1.1.2 Registers — the CPU's hands `[mechanism]`

We learned it in Phase 0.4.3: 64 D flip-flops side by side = a 64-bit register.

**Registers are the CPU's hands.** The ALU can work **only** with data in registers. If you want
to add a number that's in memory, you first have to **load** it into a register.

x86-64 has 16 general-purpose registers:

```
RAX  RBX  RCX  RDX  RSI  RDI  RBP  RSP  R8  R9  R10  R11  R12  R13  R14  R15
```

ARM64 (AArch64) has 31: `X0`–`X30`.

Plus special registers:

- **RIP** / **PC** (Program Counter) — the address of the next instruction
- **RSP** (Stack Pointer) — the top of the stack
- **RFLAGS** — the flags above

**❓ A question that comes to mind: Isn't 16 registers very few? Why not 1000?**

Three reasons, all three instructive:

1. **Physical cost.** Each register is 64 flip-flops, each flip-flop ~20+ transistors. But the
   real problem isn't the count, it's **access speed**: the bigger the register file, the deeper
   the multiplexer tree needed to select a register from it, and the more delay. The whole point
   of a register is that it is "accessible within 1 cycle" — make it bigger and you lose that
   property.

2. **Instruction encoding.** Every instruction has to specify inside itself which registers it
   uses. 16 registers = 4 bits. 1024 registers = 10 bits. In a three-operand instruction that
   means 30 bits instead of 12. Instructions bloat, reading them from memory slows down, fewer
   fit in the instruction cache. **The trade-off logic from Phase 0 is in front of us again.**

3. **There is already a hidden pool.** In modern CPUs, the *architectural* register count (16)
   and the *physical* register count differ. A modern core has 180–500+ physical registers, and
   through **register renaming** they are dynamically mapped to the 16 architectural names. The
   goal: break false dependencies and run things in parallel. So the software sees 16; the
   hardware uses hundreds.

> **This last point is a recurring theme of this phase:** *The model shown to software and what
> the hardware actually does are not the same.* An ISA is a **contract**, not a recipe. As long
> as the hardware doesn't break the contract, it can do whatever it wants inside. Hold on to this
> distinction and speculative execution in Phase 1.4 and virtualization in Phase 6 will fall into
> place much more easily.

## 1.1.3 Program Counter and Instruction Register `[concept]`

**PC** (Program Counter, `RIP` on x86): the register that holds the memory address of the
**next** instruction to be executed.

After every instruction runs, the PC advances by the length of the instruction. That's how a
program moves forward.

```
PC = 0x4005a0   →  fetch instruction  →  PC = 0x4005a4  (if it's a 4-byte instruction)
```

**A branch is precisely an intervention on the PC.** `if`, `for`, a function call — all of them
mean "set the PC not to the next address but to this address." In Phase 1.4 we'll see what this
intervention costs the pipeline.

**IR** (Instruction Register): holds **the instruction itself** fetched from memory. The Control
Unit reads and decodes it from here.

> **Addresses like `0x7ffd4a2b1c30` that you see when reading a stack trace are exactly this:**
> the value of the PC (or of return addresses on the call stack) at the moment the program
> crashed. This was one of the payoffs of learning hex in Phase 0.

## 1.1.4 Control Unit — the conductor `[concept]`

The **Control Unit** **decodes** the instruction and sends "here's what you do now" signals to
all the other parts.

When the instruction `ADD RAX, RBX` arrives, the Control Unit does the following:

1. This is an addition instruction — send the ALU a "work in addition mode" signal
2. The operands are RAX and RBX — tell the register file "feed these two into the ALU inputs"
3. The result will be written to RAX — prepare the "write the ALU output into RAX" signal
4. Make all of it happen at once on the clock edge

So the Control Unit **doesn't compute**; it only sets the switches to the right positions.

**Microcode (awareness at the `[skip]` level).** In complex architectures like x86, some
instructions don't correspond to a single hardware operation; inside the CPU they are translated
into simpler **micro-operations** (µops). This translation table can be updated — that is a
**microcode update**. Some of the Spectre/Meltdown patches were distributed as microcode updates.
In the cloud you'll see this in `dmesg` or in the cloud provider's maintenance notices; don't dig
into the details, just know it exists.

> **🤔 Think 1.1**
> ARM64 has 31 general-purpose registers; x86-64 has 16. In terms of instruction encoding, ARM64
> has to spend more bits. Yet ARM64 instructions are a **fixed 4 bytes**, while x86-64
> instructions are **variable, between 1 and 15 bytes**. How can the architecture with more
> registers use shorter instructions?
> *(Answer: at the end of the phase)*

---

# 1.2 The Fetch – Decode – Execute Cycle `[mechanism]`

This cycle is **the only thing a CPU does.** From the moment it's powered on to the moment it's
powered off, billions of times per second, it repeats just this.

## 1.2.1 Five steps

The classic explanation says three steps, but separating memory access and write-back is
essential for understanding the pipeline in Phase 1.4. We'll work with five steps (the classic
five RISC stages):

| # | Stage | Abbreviation | What happens |
|---|---|---|---|
| 1 | **Fetch** | IF | Fetch the instruction from the address the PC points to, put it in the IR, advance the PC |
| 2 | **Decode** | ID | Decode the instruction, determine which operands are needed, read the registers |
| 3 | **Execute** | EX | Perform the ALU operation (or compute an address) |
| 4 | **Memory** | MEM | Read from / write to memory if needed |
| 5 | **Write-back** | WB | Write the result to the destination register |

![Figure 1.2 — The fetch-decode-execute cycle and data flow](diagrams/hw-1-02-fetch-decode-execute.png)
*Figure 1.2 — Which unit is active in each of the five stages and which path the data follows.*

## 1.2.2 A concrete trace

Let's follow this three-line job through the CPU's eyes. In C:

```c
int c = a + b;
```

The (simplified) machine instructions the compiler produces:

```asm
mov  rax, [rbp-8]     ; load a from memory into RAX
add  rax, [rbp-16]    ; read b from memory and add it to RAX
mov  [rbp-24], rax    ; write the result to memory (into c)
```

**Instruction 1: `mov rax, [rbp-8]`**

```
IF   : PC = 0x401136 → fetch 4-byte instruction from memory → put in IR → PC = 0x40113a
ID   : "This is a load instruction. Source: RBP register - 8. Destination: RAX."
       Read RBP's value from the register file (e.g. 0x7ffd4a2b1c30)
EX   : ALU computes the address: 0x7ffd4a2b1c30 - 8 = 0x7ffd4a2b1c28
MEM  : Read 8 bytes from address 0x7ffd4a2b1c28  ← THIS IS CRITICAL, see below
WB   : Write the value read into RAX
```

**Instruction 2: `add rax, [rbp-16]`**

```
IF   : PC = 0x40113a → fetch instruction → PC = 0x40113e
ID   : "Addition + memory read. RAX and [RBP-16]."
EX   : Compute address: RBP - 16
MEM  : Read from that address
EX'  : ALU: RAX + value read     (on x86 this instruction is split into multiple µops)
WB   : Write the result into RAX, update the flags
```

**Instruction 3: `mov [rbp-24], rax`**

```
IF   : fetch instruction
ID   : "Store instruction. Source: RAX. Destination: RBP-24."
EX   : Compute address: RBP - 24
MEM  : WRITE RAX's value to that address
WB   : (none — nothing is written to a register)
```

## 1.2.3 Why the MEM stage changes everything

In the trace above, four stages take place inside the CPU: all of them can complete **within a
clock cycle**. But the `MEM` stage goes **outside** the CPU.

Let's put in the numbers (we'll go into detail in Phase 2; for now a sense of scale is enough):

| Where it is read from | Time | Cycles at 3 GHz |
|---|---|---|
| Register | ~0 (same cycle) | 0 |
| L1 cache | ~1 ns | ~4 cycles |
| L2 cache | ~4 ns | ~12 cycles |
| L3 cache | ~15–40 ns | ~45–120 cycles |
| **Main memory (DRAM)** | **~80–100 ns** | **~240–300 cycles** |

> **This table is one of the most important tables in this map.** Notice this:
>
> If the `MEM` stage has to go to DRAM, the CPU **can do nothing for that instruction for ~250
> cycles.** In 250 cycles, under ideal conditions, it could have run 500–1000 other instructions.
>
> That's why **the real enemy of modern CPUs is not computation but waiting.** All of Phase 2 is
> a struggle to reduce this waiting. The pipeline and out-of-order execution in Phase 1.4 are also
> largely an attempt to hide this waiting.

**Cloud connection — "why a higher clock isn't always better" `[application]`:**

Imagine raising the clock from 3 GHz to 4 GHz. The work inside the CPU got 33% faster. But DRAM
access **didn't get faster at all** — it's still ~80 ns. It's just that those 80 ns now amount to
320 cycles instead of 240.

So:

- **Compute-bound** workload: data is in cache, the CPU is working continuously → a clock increase
  translates directly into performance
- **Memory-bound** workload: the CPU is already waiting → a clock increase changes **almost
  nothing**

Workloads that do random lookups in a large hash table, parse JSON, or traverse pointer-based
data structures (linked lists, trees) fall into the second group. Buying a high-clock `c` family
instance for such an application is **wasting money**; the right move is to look at memory
bandwidth and cache size (Phase 2, Phase 7).

> **🤔 Think 1.2**
> In an application's `perf` output you see IPC = 0.4 (i.e. on average 0.4 instructions complete
> per clock cycle). CPU usage in `top` shows 100%.
> Is this application really using the CPU? If you move the instance to a type with a higher
> clock, what do you expect?
> *(Answer: at the end of the phase)*

---
---

# 1.3 Clock Speed and IPC

## 1.3.1 What GHz really measures `[mechanism]`

**1 Hz** = 1 cycle per second. **1 GHz** = 1 billion cycles per second.

The duration of a cycle is the inverse of the frequency:

| Frequency | Cycle time |
|---|---|
| 1 GHz | 1 ns |
| 2.5 GHz | 0.4 ns |
| 3 GHz | 0.333 ns |
| 4 GHz | 0.25 ns |

**How short a cycle is, for a sense of scale:** light travels ~30 cm in 1 nanosecond in a vacuum.
At 3 GHz a cycle is 0.333 ns, i.e. the time it takes light to travel ~10 cm. In a copper wire, a
signal travels at roughly one-half to two-thirds of the speed of light → **in one cycle a signal
can travel ~5–7 cm.**

This is the answer to "why does the chip have to be small?" Sending a signal from one end of the
chip to the other may not fit in a single cycle. It is also exactly one of the reasons, as we'll
see in Phase 2, that L3 cache is slower than L1: **physical distance.**

## 1.3.2 IPC — the real measure `[concept]`

**IPC** (Instructions Per Cycle) = how many instructions complete, on average, in one clock cycle.

And now the real performance formula:

```
Performance  ≈  Clock frequency  ×  IPC                   ×  Core count
                (hardware)          (hardware + SOFTWARE)     (hardware)
```

The most important part of this formula is this: **IPC is not a property of the hardware; it is a
property of the hardware–software pair.** Different programs give very different IPCs on the
same CPU.

**Can IPC be greater than 1?** Yes, and on modern CPUs that's the norm. Because the core is
**superscalar**: it contains multiple ALUs, multiple load units, a separate multiplier unit, and
it runs independent instructions **in parallel within the same cycle**.

Typical IPC ranges (rough, depending on workload):

| Workload type | Typical IPC | Why |
|---|---|---|
| Tight numeric loop (matrix, SIMD) | 3–6 | Data in cache, few dependencies, few branches |
| Typical application code | 1–2 | Mixed |
| Memory-heavy (hashing, pointer chasing) | 0.2–0.7 | Constantly waiting on DRAM |
| Branch-heavy (interpreter, parser) | 0.5–1.2 | Constant branch mispredictions |

> **⚠️ Common misconception:** "CPU is at 100% usage, so the CPU is the bottleneck." **No.** The
> 100% in `top` means "the core is assigned to a thread" — not whether that thread is actually
> computing. If IPC is 0.3, the core spends most of its time **waiting for data from memory**.
> `top` counts that as "busy."
>
> Knowing this distinction saves you from running in the wrong direction for hours during a
> performance problem. The right measure is the `insn per cycle` value in `perf stat` output.

> **🔧 See it on your machine** (optional, requires the `linux-tools` package)
>
> ```bash
> perf stat -e cycles,instructions sleep 0        # test whether perf works
> perf stat -- openssl speed -elapsed sha256 2>/dev/null | tail -20
> ```
> **Expected output:** a number on the `insn per cycle` line. For tight compute workloads like
> cryptography this is usually above 2.0. If you compare it with `grep` on a large file (I/O- and
> memory-heavy), you'll see a much lower IPC.
>
> *Note: on virtual machines and most EC2 instances, hardware counters are restricted; `perf` may
> say "not supported". This is normal — we'll see why in Phase 6.3.*

## 1.3.3 Why GHz can't be compared across architectures `[application]`

Compare these two instances:

| | `c6i` (Intel Ice Lake) | `c7g` (Graviton3, ARM) |
|---|---|---|
| Base/turbo frequency | up to ~3.5 GHz | ~2.6 GHz |

Look only at frequency and Intel appears 35% ahead. But in real workloads Graviton3 delivers
**equal or better** results in many scenarios. How?

Because performance is `frequency × IPC`. Graviton's IPC can close that gap:

- Wider decode (decodes more instructions in the same cycle)
- Fixed-length instructions → simple, parallel decode (we'll see this in 1.6)
- More architectural registers → less memory traffic
- A different cache layout

> **Practical rule `[application]`:** **GHz comparisons across different architectures are
> meaningless.** Within the same architecture family (e.g. `c6i` vs `c7i`), comparing frequencies
> gives you an idea; when the architecture changes (Intel ↔ AMD ↔ Graviton), the only valid
> measure is **a benchmark run with your own workload**. Neither the provider's marketing number
> nor GHz is enough.

> **🤔 Think 1.3**
> You measured a workload on `c6i.2xlarge` (3.5 GHz, 8 vCPUs): 100 requests/second.
> You measured the same workload on `c7g.2xlarge` (2.6 GHz, 8 vCPUs): 118 requests/second.
> Roughly how much higher must Graviton's IPC be than Intel's for this workload?
> (I want a simple calculation; assume equal core counts.)
> *(Answer: at the end of the phase)*

---

# 1.4 Pipeline `[concept]`

## 1.4.1 The problem and the idea

Recall the five stages from Phase 1.2: IF → ID → EX → MEM → WB.

**Naive design:** pass each instruction through all five stages, then start the next one.

```
Instruction 1:  IF  ID  EX  MEM WB
Instruction 2:                      IF  ID  EX  MEM WB
Instruction 3:                                          IF  ID  EX  MEM WB
```

The problem is obvious: **while the IF unit is working, the other four units sit idle.** 80% of
the hardware is idle at any moment.

**The pipeline idea:** the moment instruction 1 moves to the ID stage, the IF unit is free — start
fetching instruction 2 immediately.

```
Cycle:          1    2    3    4    5    6    7    8    9
Instruction 1:  IF   ID   EX  MEM   WB
Instruction 2:       IF   ID   EX  MEM   WB
Instruction 3:            IF   ID   EX  MEM   WB
Instruction 4:                 IF   ID   EX  MEM   WB
Instruction 5:                      IF   ID   EX  MEM   WB
```

![Figure 1.3 — Pipeline timing diagram](diagrams/hw-1-03-pipeline.png)
*Figure 1.3 — Overlapping instructions in a five-stage pipeline. Once the pipe is full, one
instruction completes every cycle.*

**The laundry analogy.** You have three loads of laundry: wash (30 min), dry (30 min), fold
(30 min). Done sequentially, it's 3 × 90 = 270 minutes. But if you put the second load in the
washer while the first is in the dryer, the total drops to 150 minutes. The machines didn't
change; **their idle time went down.**

**Critical distinction — latency vs throughput:**

- **Latency:** the time for a single instruction from start to finish → **still 5 cycles.** The
  pipeline doesn't shorten it.
- **Throughput:** once the pipe is full, **1 instruction completes every cycle** → a 5×
  improvement.

> You'll run into this distinction everywhere in the cloud: latency and throughput are separate
> metrics, and improving one doesn't improve the other. You'll see the same distinction again as
> IOPS/throughput in Phase 3.4 and as bandwidth/latency in Phase 5.3.

**Modern depth:** real CPUs use not 5 but **14–20-stage** pipelines (some stages are split into
sub-stages). Deeper pipeline = less work per stage = each stage takes less time = **a higher clock
frequency becomes possible.** This is exactly the solution to the "critical path" problem from
Phase 0.3.3.

## 1.4.2 Pipeline hazards — why the pipe clogs `[concept]`

The pipeline seems to work beautifully, but three things break it. These are called **hazards**.

### 1. Data dependency (data hazard)

```asm
add  rax, rbx     ; RAX = RAX + RBX
mov  rcx, rax     ; RCX = RAX    ← needs RAX's new value
```

The second instruction wants to read, in its ID stage, the value the first will write in its WB
stage — but that value hasn't been written yet. The pipeline has to **stall**.

**Solution: forwarding / bypassing.** Extra paths are added that connect the ALU's output directly
to the next instruction's EX input without waiting for WB. In most cases this eliminates the
stall entirely.

But not completely: using data right after a load instruction (**load-use hazard**) creates at
least a one-cycle stall, because the data arrives in the MEM stage.

### 2. Branching (control hazard) — the most expensive

```asm
cmp  rax, 100
jg   label_a         ; jump to label_a if RAX > 100
mov  rbx, 1          ; ← will this run? WE DON'T KNOW
```

Until the `jg` instruction is resolved in the EX stage, the CPU **doesn't know which instruction
comes next.** But it has to keep feeding the pipeline, or 15+ cycles sit empty.

**Solution: branch prediction.** Based on past behavior, the CPU predicts "this branch will
probably be taken/not taken" and keeps fetching instructions along the predicted path.

Modern branch predictors are extraordinarily good: **95–99% accuracy**. The reason is that most
real code is predictable (if a loop runs 1000 times, "continue" is the right prediction 999 times).

**But if the prediction is wrong:** all the speculative instructions in the pipeline are
cancelled (**pipeline flush**) and it is refilled from the correct path. On modern CPUs the cost
is typically **~15–20 cycles.**

**Let's do the math — and the number is surprising:**

```
Branch prediction accuracy: 95%
Misprediction penalty: 18 cycles
20% of the code is branch instructions

100 instructions contain 20 branches
20 × 5% = 1 misprediction
1 × 18 = 18 cycles of penalty

100 instructions would ideally finish in 100 cycles → took 118 cycles
IPC = 100/118 = 0.85   (a 15% loss, from just 5% mispredictions)
```

If accuracy drops to 90%, the penalty doubles. **Branch predictability is one of the factors that
directly determines IPC.**

**Cloud/application connection `[application]`:** Which code produces unpredictable branches?

- **Interpreters** (Python, Ruby, PHP) — a different branch for each bytecode
- **JSON/XML parsers** — a different branch for each character
- **Branch-heavy business logic** — long `if/else` chains, `switch`
- **Branches that depend on random data** — `if (data[rand()] > x)`

That's why a Python application is slower than a C application doing the same job not only
"because it's interpreted" but **also because its IPC is low.** And that's why buying a high-clock
instance for a Python service doesn't bring as much gain as it would for a C service.

### 3. Structural hazard

If two instructions want the same hardware unit in the same cycle, one has to wait (e.g. fetching
an instruction and reading data when there's only one memory port). In modern designs this has
largely been solved with separate instruction/data caches (Harvard architecture) and duplicated
units. Know it at the awareness level; don't go deep.

## 1.4.3 The price of a deep pipeline — the Pentium 4 lesson `[concept]`

In the early 2000s, at the peak of "clock speed = performance" marketing, Intel released the
Pentium 4. The architecture was called NetBurst, and its idea was simple: **make the pipeline as
deep as possible so you can reach a very high clock.**

In the Prescott version the pipeline grew to **31 stages**. The clock reached 3.8 GHz —
incredible for that era.

**The result: a disaster.**

- The branch misprediction penalty rose to ~31+ cycles. Performance collapsed on branch-heavy code
- Heat and power consumption got out of control
- The contemporary Pentium M, with a lower clock and a shorter pipeline (and AMD's Athlon 64),
  **beat** the Pentium 4 in real workloads

Intel abandoned NetBurst and derived the Core series from the Pentium M's architecture. Today's
Intel CPUs trace their lineage back to it.

> **The lesson — and it applies in the cloud every day:** *Optimizing a single metric (GHz) can
> slow down the system as a whole.* Today's cloud version: a system that boasts "I've pushed CPU
> utilization to 90%, I'm running efficiently" may actually be experiencing cache thrashing or a
> context-switch storm. **Optimize the outcome, not the metric.**

## 1.4.4 Speculative execution and cloud security `[concept]`

Branch prediction means "guess and keep going." This is called **speculative execution**: the CPU
**executes** instructions on a path that isn't certain yet, and discards the results if it turns
out to be wrong.

Architecturally it looks safe — the discarded results aren't reflected in the state the program
sees.

**But there's a leak.** Even when speculatively executed instructions are discarded, **they leave
traces in the cache.** And the state of the cache can be read from outside by measuring timing.

The **Spectre** and **Meltdown** vulnerabilities published in 2018 used exactly this: an attacker
could force the CPU to speculatively read memory it wasn't allowed to access, and then extract the
result through cache timing.

**Why this matters especially in the cloud `[application]`:** because the cloud is
**multi-tenant**. Your instance and a stranger's instance may be running on the same physical CPU.
Speculative execution vulnerabilities theoretically offered a way to cross the tenant boundary.

The mitigations taken and their costs:

| Mitigation | Cost |
|---|---|
| Microcode updates | 5–30% performance loss on some workloads |
| Kernel page-table isolation (KPTI) | Syscall cost went up — syscall-heavy workloads were affected |
| Disabling SMT (Hyper-Threading) | vCPU count halved in some environments |
| Physical isolation (Dedicated Host, `.metal`) | Cost |

AWS's Nitro architecture narrows this threat model at the hardware level (Phase 6.6, Phase 7.2).

> **What to take away from this:** hardware optimizations made for performance (speculation,
> shared cache, SMT) **create a security surface in multi-tenant environments.** In the cloud,
> "speed" and "isolation" are in constant negotiation. Phases 6 and 7 go into the details of that
> negotiation.

> **🤔 Think 1.4**
> One of your services has 97% branch prediction accuracy, an 18-cycle misprediction penalty, and
> 25% of its instructions are branches. With code optimization you can raise accuracy to 99%.
> How many cycles do you save for a 1000-instruction job? What is that as a percentage?
> *(Answer: at the end of the phase)*

---

# 1.5 Core, Thread, Hyper-Threading and vCPU

This section is the part of this phase **most directly tied to money**. Every instance you pick
on AWS is labeled with a vCPU count and you are billed accordingly. But the word "vCPU" is an
abstraction that **hides** what lies underneath it.

## 1.5.1 The physical core `[mechanism]`

A **physical core** is a complete copy of everything we have described so far:

- Its own register file
- Its own ALUs (more than one — superscalar)
- Its own load/store units
- Its own pipeline
- Its own branch predictor
- Its own **L1 cache** (instructions + data, separate)
- Usually its own **L2 cache**

Things shared between cores: the **L3 cache**, the memory controller, the PCIe controller, and
of course the **power/heat budget.**

![Figure 1.4 — Multi-core CPU layout](diagrams/hw-1-04-multicore-layout.png)
*Figure 1.4 — Each core's private L1/L2, the L3 cache shared by all cores, and the memory
controller.*

Recall from Phase 0.2.3: the reason core counts went up is that making a single core faster
physically stopped. Multi-core is not a choice; it is **the consequence of a necessity.**

> **And the critical consequence:** adding cores increases performance **only if the work can
> be parallelized.** A single-threaded application uses 1 core on a 64-core machine; the
> remaining 63 watch. You pay the bill for 64 cores.
>
> This is one of the most common expensive mistakes in the cloud.

## 1.5.2 SMT / Hyper-Threading — splitting one core in two `[mechanism]`

**Observation:** in Phase 1.2.3 we saw that a core spends a significant part of its time
**waiting** — DRAM access, cache misses, branch mispredictions, data dependencies. In those
moments the core's expensive ALUs sit idle.

**Idea:** let's feed the core **two separate instruction streams** (threads). While one waits,
the other uses the idle units.

This is called **SMT** (Simultaneous Multi-Threading). Intel's trade name is
**Hyper-Threading**; AMD also calls it SMT.

**What is duplicated, what is shared:**

| Duplicated (each thread has its own copy) | Shared (threads compete) |
|---|---|
| Architectural registers | ALU and other execution units |
| Program counter | L1 cache |
| Instruction queue state | L2 cache |
| Some control structures | Branch predictor tables |
| | Memory bandwidth |

**So the second thread is not a "half core."** It **looks like a full core** to the operating
system — `/proc/cpuinfo` shows two separate CPUs, and the scheduler assigns work to both. But
underneath there is a single shared execution engine.

**How big is the real gain?**

| Workload | SMT gain |
|---|---|
| Memory-heavy, lots of waiting | **20–40%** (idle units really get filled) |
| Mixed/typical server workload | **10–30%** |
| Tight compute, units already busy (HPC, SIMD) | **0–5%, sometimes negative** |
| Cache-sensitive workload | **Can be negative** (the two threads pollute each other's L1/L2) |

> **⚠️ Common misconception: "Hyper-Threading doubles the CPU."** Absolutely not. The typical
> gain is **10–30%**, never 100%. The logic is simple: the second thread doesn't bring a new
> ALU; it only fills the **idle moments** of the existing ones. No idle moments, no gain.

**In HPC and some database worlds, SMT is often disabled.** The reasons:

1. No gain (units are already busy), but there is cache pollution → net loss
2. Performance **predictability** drops (the same job takes a different time each run)
3. If software licenses are priced per core, SMT inflates the license cost
4. Security: vulnerabilities like L1TF/MDS allowed leaks between threads on the same core

## 1.5.3 vCPU — what AWS is actually selling you `[application]`

**The single sentence of this section is the most practical piece of knowledge in this phase:**

> **On AWS, on most x86 instances a vCPU is not a physical core but a hardware thread (half of
> an SMT pair). On Graviton instances, one vCPU = one full physical core, because Graviton has
> no SMT.**

Let's put the consequences in a table:

| Instance | vCPU | Physical cores | Note |
|---|---|---|---|
| `c7i.4xlarge` (Intel) | 16 | **8** | 2 vCPUs = 1 core (Hyper-Threading) |
| `c7a.4xlarge` (AMD) | 16 | **8** | 2 vCPUs = 1 core (SMT) |
| `c7g.4xlarge` (Graviton3) | 16 | **16** | No SMT, 1 vCPU = 1 core |
| `c7i.metal-48xl` | 192 | 96 | Bare metal, still has SMT |

**The same "16 vCPU" label hides a 2× difference in physical cores.**

> **This is the least understood component of Graviton's price/performance claim.** A Graviton
> instance is both cheaper and gives you more real cores for the same vCPU count. On
> compute-heavy workloads that parallelize well, this difference translates directly into
> performance.
>
> But it is not an automatic win: on a single-threaded workload, Graviton's lower frequency can
> turn into a disadvantage. And on memory-wait-heavy workloads where SMT does deliver a gain, x86
> can close the gap. **Measuring is mandatory.**

**❓ A question that comes to mind: How do I verify this from inside an instance?**

> **🔧 See it on your machine** (on EC2 or on your own machine)
>
> ```bash
> lscpu | grep -E 'Model name|^CPU\(s\)|Thread|Core|Socket'
> ```
> **Expected output (x86 with SMT, 16 vCPUs):**
> ```
> CPU(s):                16
> Thread(s) per core:    2      ← SMT ON
> Core(s) per socket:    8      ← the real core count
> Socket(s):             1
> ```
> **Expected output (Graviton, 16 vCPUs):**
> ```
> CPU(s):                16
> Thread(s) per core:    1      ← NO SMT
> Core(s) per socket:    16     ← all real cores
> ```
>
> You can also see which vCPUs share the same physical core:
> ```bash
> cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list
> ```
> If the output is something like `0,8`: CPU 0 and CPU 8 are **two threads of the same physical
> core.** In performance tests, loading both of these heavily at the same time does **not** give
> the same result as loading two separate cores.

**Can you disable SMT on AWS?** Yes — when launching an instance you can set `ThreadsPerCore=1`
via `CpuOptions`. The vCPU count halves but **the price doesn't change** (you're billed by
instance type, not by vCPU). So:

- If license cost is per core → it pays off
- If performance predictability is critical → it may make sense
- Otherwise → usually unnecessary; you throw away part of the capacity

## 1.5.4 CPU-bound or I/O-bound — when does core count help? `[application]`

This distinction is the answer to "should I get more vCPUs?"

| | **CPU-bound** | **I/O-bound** |
|---|---|---|
| Bottleneck | Compute power | Waiting on disk/network/database |
| Symptom | CPU 100%, IPC high (>1.5) | CPU low, many waiting threads |
| Examples | Video encoding, encryption, ML training, compilation, compression | Web API, proxy, file server, most CRUD applications |
| **More vCPUs?** | **Yes, helps directly** | **Mostly no** |
| Right move | Core count and/or frequency | Concurrency model, connection pool, I/O speed |

**Why doesn't adding vCPUs to an I/O-bound application help?** Because the threads aren't
computing, they are **waiting.** If 90 of 100 threads are waiting for a database response,
giving them 32 vCPUs instead of 8 doesn't make them wait faster.

> **⚠️ Common misconception and expensive mistake:** "The application is slow → make the instance
> bigger." This is a step taken without measuring the bottleneck, and on an I/O-bound application
> it **only makes the bill bigger.** In Phase 7.5 we'll build the systematic way to make this
> diagnosis. For now, build the reflex: **"CPU 100%" is not enough data; don't decide without IPC
> and a wait profile.**

**An in-between case — and very common in the cloud:** the `t` family (burstable). These
instances guarantee **a fraction** of a physical core (e.g. `t3.micro` → 10% baseline per vCPU)
and let you use more than that with **CPU credits**. When the credits run out, performance drops
to the baseline — your application suddenly becomes 5–10× slower. We'll get into the mechanism in
Phase 6.5 and Phase 7.1. For now, hold on to this: **in the `t` family, "2 vCPUs" does not mean
continuous access to 2 cores.**

> **🤔 Think 1.5**
> You run a video transcoding service. The job is entirely CPU-bound, parallelizes perfectly,
> has little memory access (the data fits in cache), and makes heavy use of SIMD instructions.
>
> There are two options at roughly similar prices:
> - `c7i.8xlarge`: 32 vCPUs (16 physical cores + SMT), up to ~3.2 GHz
> - `c7g.8xlarge`: 32 vCPUs (32 physical cores), ~2.6 GHz
>
> Which one would you choose, and why? How much do you expect SMT to contribute on this workload?
> *(Answer: at the end of the phase)*

---

# 1.6 CPU Families and the ISA

## 1.6.1 What is an ISA — the contract between hardware and software `[concept]`

An **ISA** (Instruction Set Architecture) is **the promise a CPU makes to software**:

- Which instructions exist (`ADD`, `MOV`, `JMP`, ...)
- How instructions are encoded in memory (how many bytes, what each bit means)
- How many registers there are and what they are called
- How memory is addressed
- How interrupts and privilege levels work

**An ISA is an *interface*, not an *implementation*.** Intel and AMD have completely different
internal designs but implement **the same ISA** (x86-64). That's why the same binary runs on
both.

> **This distinction is directly useful in the cloud:** a Docker image is tagged as
> `linux/amd64` or `linux/arm64` — that is the ISA tag. When switching between Intel and AMD
> instances you don't need to change the image (both are amd64). When moving to Graviton you
> **do** (arm64).

## 1.6.2 x86-64 and ARM64 — two design philosophies `[application]`

### x86-64 (Intel, AMD)

**Lineage:** Intel 8086 in 1978 → 80286 → 80386 (32-bit) → AMD64 (64-bit, 2003). More than
forty years of **backward compatibility.** A modern Xeon can, in theory, run 8086 code from 1978.

Characteristics:

- **Variable-length instructions: 1–15 bytes.** To know where an instruction ends you have to
  decode it → **the decode stage is complex and energy-hungry**. Parallel decode is hard because
  you can't know where the next instruction starts.
- **16 general-purpose registers.** Few; the compiler frequently has to spill to memory
  (register spill).
- **Instructions with memory operands:** it can both read memory and compute in a single
  instruction such as `add rax, [rbp-16]` (a CISC trait).
- **A huge instruction set:** thousands of instructions + SIMD extensions such as SSE, AVX,
  AVX-512.
- Internally, instructions are translated into **µops** and executed on a RISC-like core.

### ARM64 / AArch64 (Graviton, Apple Silicon, Ampere)

**Lineage:** ARM1 in 1985, designed for low power. AArch64 (2011) is a clean slate — the
compatibility burden of 32-bit ARM was largely dropped.

Characteristics:

- **Fixed-length instructions: each exactly 4 bytes.** Where the next instruction starts is
  **always known** → decode is simple and **easily parallelized.** This is why Graviton can
  decode more instructions in the same cycle.
- **31 general-purpose registers.** Less memory traffic, fewer spills.
- **Load/store architecture:** memory is accessed only through dedicated `LDR`/`STR`
  instructions; arithmetic instructions work only with registers. Each instruction is simpler,
  the pipeline more regular.
- **A smaller, more regular instruction set.** The decode circuit is small → **fewer
  transistors, less energy.**

### And this is the answer to Think 1.1 from 1.1.4

ARM64 has **more** registers, yet its instructions are **shorter and fixed**. How? Because ARM
won its encoding budget by saving elsewhere: fewer addressing modes, no memory operands, a more
regular format. x86, because of backward compatibility, has to spend most of its encoding space
on old instructions.

**This is one more example of the trade-off we saw again and again in Phase 0:** there is no free
lunch, only a choice of where you put which cost.

## 1.6.3 CISC vs RISC — and why this distinction is misleading today `[concept]`

The classic explanation:

- **CISC** (Complex Instruction Set Computer): a small number of powerful, complex instructions.
  x86.
- **RISC** (Reduced Instruction Set Computer): many simple, fast instructions. ARM.

**This distinction was meaningful in the 1980s; today it has largely melted away:**

- x86 CPUs internally translate instructions into µops and run them on a **RISC-like** core
- Over time ARM added **complex instructions** such as SIMD (NEON, SVE), crypto and atomic
  operations
- Both are out-of-order, superscalar and speculative

**The real difference that still holds today comes down to one place: decode cost and power
efficiency.** x86's variable-length instructions require a complex decode circuit that runs on
every cycle, and that circuit burns energy constantly. ARM's fixed format removes that cost.

That's why ARM has a structural advantage in **performance per watt**. And in the cloud,
performance per watt directly means **cost**: less power = less cooling = more servers per rack =
cheaper per unit.

## 1.6.4 The Graviton decision — in practice `[application]`

**Graviton** is AWS's own ARM64 server processor (Graviton1 → 2 → 3 → 4).

**What it gains you:**

| Gain | Explanation |
|---|---|
| Price | List price typically **10–20% cheaper** than comparable x86 instances |
| Real cores | No SMT → **2× the physical cores** for the same vCPU count (1.5.3) |
| Energy | Lower consumption to the extent AWS states (also used for sustainability goals) |
| Price/performance | AWS claims improvements of **up to ~40%** on many workloads — *workload dependent* |

> **Treat these numbers with caution.** "40% better price/performance" is a marketing summary
> and comes from specific benchmarks. The real number for your workload could be much better, or
> much worse. **Don't migrate without measuring.**

**What it costs you — the real obstacles:**

1. **Recompilation is required.** Everything compiled (C/C++/Go/Rust binaries) must be rebuilt
   for arm64.
2. **Container images.** You need to set up multi-arch builds (`docker buildx`,
   `--platform linux/arm64`). Base images must have an arm64 variant.
3. **Dependencies.** Libraries with native extensions (Python `wheel`s, Node native modules,
   JVM JNI libraries) may not be available for arm64. Most popular packages support it now, but
   surprises show up in the long tail.
4. **Closed-source / vendor software.** If the vendor doesn't ship an arm64 build, the road is
   closed.
5. **Agents and tools.** Monitoring/APM/security agents must have arm64 builds.

**Decision tree:**

```
Interpreted/JIT language? (Python, Node.js, Java, Ruby, .NET Core)
├─ Yes → Do your native dependencies support arm64?
│         ├─ Yes → Graviton is a strong candidate. Test it; there's very likely a gain.
│         └─ No  → Can you replace the dependency? If not, stay on x86.
└─ No (compiled) → Can you add an arm64 build to CI/CD?
          ├─ Yes → Test it. Usually worth it.
          └─ No  → Stay on x86.

Closed-source vendor software? → If the vendor doesn't support arm64, the matter is closed.
Is single-thread performance critical? → Graviton's lower frequency may be a disadvantage, MEASURE.
```

**The easiest migration candidates:** managed services. RDS, ElastiCache, OpenSearch, Lambda,
Fargate — for these, moving to Graviton is often **a single setting change**, because AWS solves
the compilation and dependency problem. Starting your Graviton journey here is the fastest way to
get a gain without touching the application layer.

> **🤔 Think 1.6**
> Your team wants to move a Python/Django API to Graviton. The application uses `numpy`,
> `psycopg2` and `Pillow` — all three have C extensions. Deployment is with Docker; the base image
> is `python:3.11-slim`.
>
> What three things must you check before migrating, and which one could really stop you?
> *(Answer: at the end of the phase)*

---
---
---

# Phase 1 — Answers to the Think Questions

## Answer 1.1 — How does ARM64 use shorter instructions despite having more registers?

**Question:** ARM64 has 31 registers (needs 5 bits), x86-64 has 16 (4 bits). Yet ARM is a fixed
4 bytes and x86 is 1–15 bytes. How?

Because in instruction encoding, the register field is **not the largest cost item.** ARM wins
its encoding budget by saving in three places:

**1. Simple addressing modes.** On x86, an operand can take forms like `[rbx]`, `[rbx+8]`,
`[rbx+rcx*4+16]`, an absolute address, RIP-relative... Each requires extra bytes (ModR/M, SIB,
displacement) that lengthen the instruction. ARM has a limited number of regular modes for
load/store.

**2. No memory operands (load/store architecture).** On ARM, `ADD` works only with registers. On
x86, `add rax, [rbp-16]` is a single instruction that carries an address inside it — which
lengthens that instruction by 3–4 bytes.

**3. No backward-compatibility burden.** Much of x86's encoding space is occupied by instructions
and prefixes dating back to 1978. New instructions (AVX etc.) have to be encoded with long prefix
chains. ARM64 started with a clean slate.

> **The lesson:** in a system, "more resources" (registers) does not necessarily mean "more cost"
> (instruction size) — **if you simplified somewhere else.** Design is about where you spend the
> budget. This way of thinking will come up again in Phase 3 (choosing the SSD cell type) and
> Phase 7 (choosing the instance family).

**Related section:** 1.1.2, 1.6.2

---

## Answer 1.2 — IPC 0.4 and 100% CPU in `top`

**Question:** Is this application really using the CPU? What will a higher-clock instance do?

**No, it's not "using" the CPU — it's *making the CPU wait.***

IPC 0.4 means: on average, 4 instructions complete every 10 clock cycles. A modern superscalar
core has the capacity to complete 4–6 instructions in the same cycle. So it's using roughly
**10%** of the core's capacity.

What is it doing the rest of the time? Most likely **waiting**:

- Waiting for data from DRAM (cache miss) — the most likely cause
- The pipeline refilling after branch mispredictions
- Stuck in a chain of data dependencies

**Why does `top` show this as 100%?** Because what `top` measures is "the fraction of time the
core is assigned to a thread." While the thread waits for data from memory, the core is still
assigned to it — it isn't doing work, but it isn't given to anyone else either. From the
operating system's point of view, that is "busy."

**What happens if you move to a higher-clock instance?**

Very little. A clock increase speeds up the work inside the CPU, but the bottleneck is **outside
the CPU** (in DRAM). DRAM access is ~80 ns and stays 80 ns whatever the clock. If you raise the
clock by 30%, those 80 ns amount to more cycles — that's all. The typical gain stays below 5%.

**The right moves:**

| Move | Why |
|---|---|
| Make the data structure cache-friendly | The biggest gain. We'll see it in Phase 2.3 |
| An instance family with a larger L3 cache | Dramatic gain if the working set fits in cache |
| A family with high memory bandwidth (`r`, `x`) | Helps if bandwidth is saturated |
| Fix NUMA placement | Phase 2.7 |
| More vCPUs (if the work is parallel) | The waiting gets hidden, total throughput goes up |

> **Keep this as a reflex:** *"CPU 100%" is not a diagnosis, it is a symptom.* A diagnosis needs
> IPC, the cache miss rate and memory bandwidth.

**Related section:** 1.2.3, 1.3.2 · **Continued in:** Phase 2.3, Phase 7.5

---

## Answer 1.3 — Calculating Graviton's IPC advantage

**Question:** Intel at 3.5 GHz does 100 requests/s; Graviton at 2.6 GHz does 118 requests/s. How
large is the IPC difference?

Performance ≈ frequency × IPC × cores. We assumed equal core counts (as the question asked), so:

```
Intel:     100 = 3.5 × IPC_intel × k
Graviton:  118 = 2.6 × IPC_grav  × k

Divide:  118/100 = (2.6 × IPC_grav) / (3.5 × IPC_intel)
         1.18    = 0.743 × (IPC_grav / IPC_intel)

IPC_grav / IPC_intel = 1.18 / 0.743 = 1.588
```

**On this workload, Graviton's IPC is about 59% higher than Intel's.**

**But we need to be honest here — this calculation is a simplification:**

In reality, "IPC difference" doesn't explain it alone. Recall from Phase 1.5.3: for the same vCPU
count, **Graviton has twice the physical cores.** So `k` (core count) is not equal. The 118 you
measured mixes all of these together:

- The real IPC difference (architecture)
- The physical core difference (no SMT)
- Differences in cache size and memory bandwidth
- The application's degree of parallelism

**The takeaway — and it matters more than the calculation:** in the cloud, attributing a
performance difference to a single cause is almost always wrong. The right approach is to
**measure end-to-end with your own workload** and to ask "why" only after measuring. That's why
the sentence "Graviton is X% faster" is meaningless on its own; "X% faster on **my** workload" is
meaningful.

**Related section:** 1.3.2, 1.3.3, 1.5.3

---

## Answer 1.4 — The gain from better branch prediction

**Question:** 97% → 99% accuracy, 18-cycle penalty, 25% of instructions are branches, 1000
instructions.

```
1000 instructions × 25% = 250 branch instructions

At 97% accuracy:
  mispredictions = 250 × 0.03 = 7.5
  penalty = 7.5 × 18 = 135 cycles

At 99% accuracy:
  mispredictions = 250 × 0.01 = 2.5
  penalty = 2.5 × 18 = 45 cycles

Saving = 135 - 45 = 90 cycles
```

**What is that as a percentage?** It depends on the base cycle count. Assuming IPC = 1, 1000
instructions ideally take 1000 cycles:

```
Total at 97%: 1000 + 135 = 1135 cycles
Total at 99%: 1000 +  45 = 1045 cycles

Improvement: (1135 - 1045) / 1135 = 7.9%
```

**An 8% performance gain, just by raising branch prediction accuracy from 97% to 99%.**

**Why this matters:** a 2% accuracy increase sounds insignificant, but its effect is 8%. The
reason is that **the number of mispredictions drops threefold** (7.5 → 2.5). Small changes in a
ratio can cut rare but expensive events many times over.

This way of thinking also applies to cache miss rates, and you'll meet the same math in Phase 2.3:
**the difference between a 95% hit rate and a 99% hit rate is not 4% — it is 5× fewer misses.**

**Related section:** 1.4.2 · **Continued in:** Phase 2.3

---

## Answer 1.5 — Intel or Graviton for video transcoding?

**Question:** CPU-bound, perfectly parallel, data fits in cache, SIMD-heavy. `c7i.8xlarge`
(16 physical cores + SMT, ~3.2 GHz) vs `c7g.8xlarge` (32 physical cores, ~2.6 GHz).

**A rough calculation points to Graviton:**

```
c7i:  16 physical cores × 3.2 GHz = 51.2 "core-GHz"
      + SMT contribution: 0–5% on this workload  (the units are already busy!)
      ≈ 51–54

c7g:  32 physical cores × 2.6 GHz = 83.2 "core-GHz"
```

**On paper Graviton is clearly ahead** — and the reason is directly the rule from 1.5.2: on a
SIMD-heavy, compute-saturated workload **SMT gains almost nothing**, because there are no gaps in
the execution units to fill. The second half of `c7i`'s "32 vCPUs" is largely imaginary for this
workload.

**But before deciding you must check three things — and they can reverse the result:**

1. **SIMD instruction set fit.** If your transcoding library (e.g. x264/x265/libaom) contains
   hand-optimized assembly for AVX2 or AVX-512, its ARM counterpart (NEON/SVE) may not be equally
   mature. That alone can erase Graviton's advantage. **This is the most critical item.**
2. **Is there hardware acceleration?** The truly right answer for transcoding may be neither:
   specialized instances such as AWS `vt1` (Xilinx video transcoding) or GPU-based solutions can
   be many times more efficient than a general-purpose CPU. **The right question may not be
   "which CPU?" but "a CPU at all?"**
3. **Measurement.** Benchmark with your real media files and your real profile. The paper
   calculation gives direction; it doesn't make the decision.

> **The real lesson of this question:** all of Phase 1's concepts (physical core, the limits of
> SMT, frequency × IPC) lead you to **the right hypothesis**. But a hypothesis doesn't replace
> measurement. A good cloud engineer knows what to expect before measuring — and measures anyway.

**Related section:** 1.5.2, 1.5.3, 1.5.4, 1.6.4

---

## Answer 1.6 — Can Django + numpy/psycopg2/Pillow move to Graviton?

**Question:** Three things to check before migrating, and which one really stops you?

**Check 1: Does the base image have an arm64 variant?**
`python:3.11-slim` is an official image and **has** a `linux/arm64` variant. Not a problem.
Check: `docker manifest inspect python:3.11-slim | grep architecture`

**Check 2: Do the C-extension packages have arm64 wheels?**
- `numpy` → manylinux aarch64 wheels **exist**. Not a problem.
- `psycopg2-binary` → an aarch64 wheel **exists**. (The `psycopg2` source release can also be
  compiled; it needs `libpq-dev`.)
- `Pillow` → aarch64 wheels **exist**.

All three are large, well-maintained projects. But **if there is no wheel**, `pip install` falls
back to building from source: you need build tools (`gcc`, header files), the image bloats, and
build time goes from minutes to tens of minutes. Not a blocker, but it slows your pipeline.

**Check 3: Can CI/CD produce arm64 images?**
This is usually the item that **creates the most work**. What you need:
- Multi-arch builds with `docker buildx` (`--platform linux/amd64,linux/arm64`)
- Building under QEMU emulation is **very slow** — in practice you want native arm64 runners
  (CodeBuild's arm64 environment, GitHub Actions' arm64 runners, etc.)
- Multi-arch manifest management in the image registry

**Which one really stops you?**

All three above can be solved. **The real blockers come from places not on the list:**

| Hidden risk | Why it stops you |
|---|---|
| **Monitoring/APM agent** (Datadog, New Relic, Dynatrace) | If the agent has no arm64 build you lose observability — which is worth more than the performance gain |
| **Security agent** (EDR, compliance scanning) | In a corporate environment, running nodes without the agent is a policy violation |
| **An obscure dependency** | One of the 80 packages in `requirements.txt` may be an unmaintained package with no arm64 wheel |
| **Vendor library** | A closed-source SDK/driver may be amd64-only |

**The right method — don't guess, scan:**

```bash
# Try to resolve all dependencies for arm64 (binary wheels only)
pip download --platform manylinux2014_aarch64 --only-binary=:all: \
             --dest /tmp/arm64check -r requirements.txt
```
This command lists the packages without wheels **in one go.** You get the answer in five minutes.

**Recommended migration order:**
1. First move managed services to Graviton (RDS, ElastiCache) — without touching the application
2. Then run the application on arm64 in staging and validate the agents
3. Then roll out to prod gradually (mixed architectures are possible in an ASG — you can keep
   amd64 and arm64 nodes side by side and shift traffic slowly)

**Related section:** 1.6.2, 1.6.4

---

# Phase 1 — Frequently Asked Questions

### Q1. `nproc` tells me 8. Is that 8 cores or 8 threads?

It says **8 logical processors** (logical CPUs) — the number of units the operating system sees.
If SMT is on, that means 4 physical cores.

For a definitive answer:
```bash
lscpu | grep -E 'Thread|Core|Socket'
```
If `Thread(s) per core: 2`, the physical core count is `nproc / 2`.

This distinction is directly useful when sizing thread pools: sizing a CPU-bound pool to `nproc`
is usually right on a machine with SMT, but sizing it to **the physical core count** gives better
and more predictable results on some workloads. Don't assume without measuring.

### Q2. How do I see my instance's real frequency when turbo boost is on?

```bash
grep MHz /proc/cpuinfo | head -4          # current frequency
```

But **in virtualized environments this value is often misleading**: the hypervisor may hide the
real frequency or report a fixed value (Phase 6.3). On EC2, `/proc/cpuinfo` usually shows the base
frequency, not the real turbo behavior.

**The practical approach:** instead of trying to read the frequency, **measure work output.**
Running a fixed micro-benchmark (e.g. `openssl speed`) and comparing operations per second is a
far more reliable indicator than the reported MHz.

### Q3. My application opens lots of threads but doesn't get faster. Why?

Possible causes, in order of frequency:

1. **I/O-bound** — the threads are waiting, not computing (1.5.4)
2. **Lock contention** — the threads wait on the same lock and become serial
3. **Amdahl's Law** — the serial part of the job sets the ceiling. If 10% of the job can't be
   parallelized, you can speed up at most 10×, even with infinite cores
4. **Memory bandwidth saturated** — the cores are there but they are all waiting on the same DRAM
   (Phase 2.4)
5. **False sharing** — different threads write to the same cache line (we'll see this in
   Phase 2.3)
6. **Context-switch storm** — far more active threads than cores

Diagnosis order: first look at CPU usage and IPC, then the lock profile, then memory bandwidth.

### Q4. AWS says "up to 3.5 GHz". How far is "up to"?

It's vague, and deliberately so. The real frequency depends on:

- How many cores are active (the thermal budget — Phase 0.2.3)
- Which instructions are running (AVX-512-heavy code **lowers** the frequency — high power
  density)
- The physical server's current temperature and total load
- What neighboring tenants are doing (on shared instances — Phase 7.4)

That's why you should run performance tests **long, not short**. A 30-second test measures turbo
frequency; a 10-minute test measures real sustainable performance. The difference can reach 20%.

### Q5. You never explained out-of-order execution. Do I need to know it?

One sentence at the `[concept]` level is enough: **modern CPUs execute instructions not in the
order they were written but in the order their operands become ready**; they then "retire" the
results in an order consistent with the program. The goal: while one instruction waits for memory,
run independent instructions behind it and fill the gap.

In practice the consequence for you is: **the CPU hides some of the memory waiting on its own** —
but only to a limit (the reorder window is a few hundred instructions). If the cache miss rate is
high, this window fills up and the hiding collapses. So out-of-order doesn't reduce the importance
of Phase 2; it only hides small delays.

It's not a topic you need to go deep on.

### Q6. `perf` doesn't work on EC2; it says "not supported". What's wrong?

Hardware performance counters (PMU) are restricted by default in virtualized environments — the
hypervisor may not expose them to the guest, because counters can be used for side-channel attacks
(the threat model from Phase 1.4.4).

Options:
- **Use a `.metal` instance** — bare metal gives full access to the PMU
- **Make do with software counters**: `perf stat -e task-clock,context-switches,page-faults` needs
  no hardware counters
- **Use application-level profiling** (language-level profilers, APM)
- Some instance types have limited PMU access; check what's available with `perf list`

### Q7. How many cycles does an instruction really take?

It varies a lot by instruction, and **latency and throughput are measured separately** (the same
distinction as in Phase 1.4.1):

| Instruction | Latency (cycles) | Throughput (per cycle) |
|---|---|---|
| `ADD` (register) | 1 | 3–4 per cycle |
| `MUL` (integer) | 3–5 | 1 per cycle |
| `DIV` (integer) | 20–90 | very low |
| Load from L1 | 4–5 | 2 per cycle |
| Load from DRAM | ~200–300 | depends on the memory controller |

Note: **division is very expensive.** That's why compilers turn division by a constant into
multiplication + shift, and why division is avoided in performance-critical code. A small detail,
but it makes a measurable difference in a hot loop.

---

# Phase 1 — Test Yourself

**1.** Name the five basic components of a CPU and describe the job of each in one sentence.

**2.** How does the ALU perform subtraction? Why is there no separate subtractor circuit?

**3.** How is the expression `if (a == b)` carried out in hardware?

**4.** Why does x86-64 have only 16 general-purpose registers? Give three reasons.

**5.** List the five stages of the fetch-decode-execute cycle in order.

**6.** How many cycles does a read from DRAM correspond to on a 3 GHz CPU? Why does this matter?

**7.** Write the performance formula. Which term does software affect?

**8.** An application with IPC 0.3 shows 100% CPU in `top`. What is happening?

**9.** Does a pipeline improve latency or throughput? Explain.

**10.** Name the three types of pipeline hazard and give one solution for each.

**11.** Why is the cost of a branch misprediction so high?

**12.** Why did the Pentium 4 / NetBurst fail? What is the general lesson from it?

**13.** Why are Spectre/Meltdown-class vulnerabilities especially critical in the cloud?

**14.** In SMT, what is duplicated and what is shared? What is the typical gain percentage?

**15.** `c7i.4xlarge` has 16 vCPUs. How many physical cores? `c7g.4xlarge` has 16 vCPUs. How many
physical cores?

**16.** How can you tell from inside an instance whether SMT is on or off?

**17.** Why doesn't adding vCPUs to an I/O-bound application help?

**18.** What is an ISA? Why can Intel and AMD run the same binary?

**19.** Where does ARM64's structural energy advantage over x86-64 come from?

**20.** Name four obstacles you may run into when moving to Graviton.

---

## Answer Key

**1.** **ALU** (does the computation) · **Registers** (temporary data containers the ALU works on) ·
**Program Counter** (address of the next instruction) · **Instruction Register** (holds the current
instruction) · **Control Unit** (decodes the instruction and sends control signals to the other
units). · *section 1.1*

**2.** With two's complement: `A - B = A + (NOT B + 1)`. The bits of B are inverted, 1 is added, and
the normal adder is used. The reason: **reusing an existing circuit** is cheaper than adding a
separate one. · *section 1.1.1*

**3.** The CPU performs the subtraction `a - b`, discards the result but updates the **Zero Flag**
(ZF). A conditional branch instruction then looks at ZF to decide whether to jump. **A comparison
is a subtraction; a decision is a flag bit.** · *section 1.1.1*

**4.** (a) **Access speed** — as the register file grows, the selection multiplexer gets deeper and
1-cycle access is lost. (b) **Instruction encoding** — more registers = more bits per instruction =
bloated instructions. (c) **There is already a hidden pool** — with register renaming, 180–500+
physical registers are mapped to 16 architectural names. · *section 1.1.2*

**5.** **IF** (Fetch) → **ID** (Decode) → **EX** (Execute) → **MEM** (Memory) → **WB**
(Write-back). · *section 1.2.1*

**6.** ~80–100 ns ÷ 0.333 ns = **~240–300 cycles.** It matters because the CPU waits on that
instruction the whole time; in the same period it could have run hundreds of other instructions.
**The real bottleneck of modern CPUs is waiting, not computation.** · *section 1.2.3*

**7.** `Performance ≈ Clock × IPC × Core count`. **Software directly affects IPC** (data layout,
branch structure, memory access pattern). · *section 1.3.2*

**8.** About 10% of the core's capacity is being used; the rest is **waiting** (most likely cache
misses → DRAM). `top` counts "the core is assigned to a thread" as 100%, not whether work is really
being done. A higher clock changes very little. · *sections 1.2.3, 1.3.2*

**9.** It improves **throughput**. The start-to-finish time of a single instruction (latency)
doesn't change (it even increases in a deep pipeline); but once the pipe is full, **one instruction
completes every cycle.** · *section 1.4.1*

**10.** **Data hazard** → forwarding/bypassing. **Control hazard (branches)** → branch prediction +
speculative execution. **Structural hazard** → duplicated units, separate instruction/data caches.
· *section 1.4.2*

**11.** Because when a prediction is wrong, all speculative instructions in the pipeline are
cancelled (**pipeline flush**) and the pipe is refilled from the correct path. In modern deep
pipelines this is **~15–20 cycles**. · *section 1.4.2*

**12.** It pushed the pipeline to 31 stages to chase a high clock; the branch misprediction penalty
and power consumption got out of control, and it lost to lower-clock rivals on real workloads.
**Lesson: optimizing a single metric (GHz) can slow the system down as a whole.** · *section 1.4.3*

**13.** Because the cloud is **multi-tenant** — instances of different customers run on the same
physical CPU. The traces speculative execution leaves in the cache enabled information leaks across
the tenant boundary. Every mitigation (microcode, KPTI, disabling SMT) has a performance cost.
· *section 1.4.4*

**14.** **Duplicated:** architectural registers, program counter, instruction queue state.
**Shared:** the ALUs and all execution units, L1/L2 cache, branch predictor, memory bandwidth.
**Typical gain 10–30%**, never 100%; ~0 or negative on compute-saturated workloads. · *section 1.5.2*

**15.** `c7i.4xlarge` (Intel, has SMT) → **8 physical cores.** `c7g.4xlarge` (Graviton, no SMT) →
**16 physical cores.** The same vCPU label, a 2× difference. · *section 1.5.3*

**16.** `lscpu | grep 'Thread(s) per core'`. If the value is `2`, SMT is on; if `1`, it's off or
absent. In addition, `/sys/devices/system/cpu/cpu0/topology/thread_siblings_list` shows which
logical CPUs share the same physical core. · *section 1.5.3*

**17.** Because the threads aren't computing, they're **waiting** (disk, network, database). Giving
a waiting thread more cores doesn't make it wait faster. The right move: the concurrency model, the
connection pool, the speed of the I/O side. · *section 1.5.4*

**18.** An **ISA** is the contract hardware makes with software: which instructions exist, how they
are encoded, how many registers there are, how memory is addressed. Intel and AMD run the same
binary because, even though their **internal designs are completely different**, they implement
**the same ISA (x86-64)**. · *section 1.6.1*

**19.** **From decode cost.** x86's variable-length (1–15 byte) instructions require a complex,
energy-hungry decode circuit that runs on every cycle and make parallel decode difficult. ARM64's
fixed 4-byte instructions make decode simple, small and easily parallelizable → fewer transistors,
less energy. · *section 1.6.3*

**20.** (a) Recompiling compiled code for arm64 · (b) Multi-arch builds of container images and the
need for arm64 base images · (c) arm64 support in dependencies with native extensions (Python
wheels, Node native modules) · (d) arm64 builds of closed-source vendor software and of
monitoring/security **agents**. · *section 1.6.4*

---

### Scoring

| Correct answers | What to do |
|---|---|
| 17–20 | Move on to Phase 2. Your CPU model is solid. |
| 13–16 | Move on, but re-read 1.5 (core/thread/vCPU) — Phases 6 and 7 build on it. |
| 8–12 | Definitely repeat 1.2.3, 1.3.2 and 1.5. These three are prerequisites for Phase 2. |
| 0–7 | Work through the phase again from the start. |

**Which section to return to for each missed question:**

| Question | Section |
|---|---|
| 1, 2, 3, 4 | 1.1 — Components |
| 5, 6 | 1.2 — Fetch-decode-execute and the memory wall |
| 7, 8 | 1.3 — Clock and IPC |
| 9, 10, 11, 12, 13 | 1.4 — Pipeline and speculation |
| 14, 15, 16, 17 | 1.5 — Core, thread, vCPU |
| 18, 19, 20 | 1.6 — ISA and Graviton |

---

# Phase 1 — Closing and Bridge to Phase 2

## What you carry out of this phase

**1. The "vCPU ≠ core" reflex.**
When you look at an instance table, you'll now automatically ask "are these vCPUs SMT threads or
real cores?" This single question sits at the center of Graviton decisions and capacity planning.

**2. The "Performance = Clock × IPC × Cores" formula.**
And the most important piece of knowledge inside it: **software determines IPC.** Choosing hardware
alone doesn't buy performance.

**3. The "CPU 100% is not a diagnosis" reflex.**
Waiting also shows up as 100%. This is the cornerstone of the bottleneck analysis in Phase 7.

**4. The most critical number: DRAM access ≈ 250 cycles.**
This number is **the reason the next phase exists.**

## Where Phase 2 connects to this

In Phase 1 we hit the same wall again and again:

> *The CPU is fast, memory is slow. The CPU spends most of its time waiting.*

The pipeline tried to **hide** this waiting. Out-of-order execution tried to hide it. SMT tried to
hide it by running another thread while waiting.

**None of them solved the problem — they only covered it up.** The only thing that attacks the
problem itself is **the memory hierarchy**, and that is all of Phase 2.

| What you learned in Phase 1 | What happens in Phase 2 |
|---|---|
| DRAM access ~250 cycles (1.2.3) | **The reason cache exists** (2.1) |
| Register = fastest but smallest (1.1.2) | The top of the hierarchy (2.2) |
| SRAM/DRAM difference (Phase 0.4.4) | **The reason for the L1/L2/L3 levels** (2.3) |
| SMT threads share L1/L2 (1.5.2) | Cache pollution and false sharing (2.3) |
| Cores share L3 (1.5.1) | **The mechanism of the noisy neighbor** (2.3, Phase 7.4) |
| Memory access determines IPC (1.3.2) | Cache hit rate = the main driver of IPC (2.3) |
| Multi-socket servers | **NUMA** (2.7) |

> **Phase 1 output — ask yourself before continuing:**
> "Can I explain an instruction's journey inside the CPU step by step, including what can happen
> in the `MEM` stage? And when I hear 'this instance has 16 vCPUs', how many questions come to
> mind?"
>
> The answer to the second question should be at least three: *Is there SMT? Which architecture?
> Shared or dedicated?*

---

> **Navigation:** [◀ Phase 0 — Numeric Foundation](Phase_0_Numeric_Foundation.md) · **Phase 1** · [Phase 2 — Memory Hierarchy ▶](Phase_2_Memory_Hierarchy.md)
