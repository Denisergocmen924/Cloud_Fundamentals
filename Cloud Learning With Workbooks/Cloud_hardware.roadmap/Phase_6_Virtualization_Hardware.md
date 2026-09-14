# Phase 6 — Virtualization Hardware

> **Navigation:** [◀ Phase 5 — Network Hardware](Phase_5_Network_Hardware.md) · **Phase 6** · [Phase 7 — Cloud Connection ▶](Phase_7_Cloud_Connection.md)

---

> ### In this phase an assumption collapses.
>
> From Phase 0 through Phase 5 we examined hardware **as if a single operating system owned it
> all by itself.** The CPU was yours, the RAM was yours, the disk and the NIC were yours.
>
> **In the cloud that isn't true.** The `m7i.2xlarge` you rent is an 8-vCPU slice of a
> physical server with 128 vCPUs. On the same silicon you share the same L3 cache, the same
> memory channels, the same PCIe lanes and the same NIC with dozens of tenants you've never met.
>
> **This phase describes the hardware that makes that sharing possible.** This is the place the
> source map calls "the most connective phase": the pieces of the previous six phases lock
> together here.

---

## Where we are coming from

This phase takes something from every previous phase and revisits it in a virtualization
context:

| Previous phase | What happens here |
|---|---|
| 1.5 core / thread / vCPU | 6.5 — **how** a vCPU is scheduled onto a physical core |
| 2.5 virtual memory, page tables | 6.4 — address translation **twice over** (EPT/NPT) |
| 2.3.4 shared L3 | 6.1.2 — the root of noisy neighbors |
| 3.3 NVMe queues | 6.6 — dividing the queues among VMs |
| Phase 4 Q4 — IOMMU | 6.6.3 — **the precondition for SR-IOV** (we promised this in Phase 4) |
| 5.1.4 multi-queue NIC | 6.6.3 — each VM gets its own queue |
| 4.3.3 Nitro | 6.6.4 — finally, the full mechanism |

---

## By the end of this phase

- You will be able to describe where an EC2 instance sits on a physical server
- You will be able to explain **where** the hypervisor's cost comes from and why Nitro removes it
- When you see `steal time`, you will know what it means and what to do about it
- You will be able to name and tell apart the three separate mechanisms of noisy neighbors
- You will be able to choose between a container and a VM **on security and performance grounds**
- You will be able to answer "t3 or c7i?" not with pricing, but with **physics**

---

## Phase map

| Section | Topic | Depth | Why it matters |
|---|---|---|---|
| 6.1 | Why virtualization is necessary | `[concept]` | The economic foundation of the cloud |
| 6.2 | Hypervisor types | `[mechanism]` | What's underneath EC2 |
| 6.3 | **Hardware-assisted virtualization** | `[mechanism]` | **The heart of the phase — the cost of a VM exit** |
| 6.4 | Memory virtualization | `[mechanism]` | How RAM is really divided up |
| 6.5 | **CPU virtualization and steal time** | `[application]` | **A daily diagnostic tool** |
| 6.6 | I/O virtualization | `[mechanism]` | The full explanation of Nitro |
| 6.7 | Container vs VM | `[application]` | A decision made every day |

---

# 6.1 Why virtualization is necessary

## 6.1.1 The economic necessity `[concept]`

Picture a physical server: 128 cores, 512 GB RAM, a 100 Gbps NIC.

**If you give it to a single customer:**
```
Typical web application usage: 5–15% CPU
→ 85% of the server sits idle
→ Power, cooling, floor space, depreciation: paid in full
→ Customer: "I don't want 128 cores, 4 is enough"
```

**If you virtualize it:**
```
32 customers × 4 vCPU = 128 vCPU
→ Each customer pays for what they need
→ The server reaches 60–80% utilization
→ The hardware cost is divided 32 ways
```

> **The entire cloud is built on this single idea.** Virtualization is the technical
> counterpart of the "pay for what you use" model. The reason AWS can sell a `t3.micro` for a
> few cents an hour is that it can host hundreds of `t3.micro`s on that physical server.

**But it has a price**, and the rest of this phase describes that price:

| The price | Where it's covered |
|---|---|
| The virtualization tax (lost CPU/latency) | 6.3.4 |
| Noisy neighbors | 6.1.2, 7.4 |
| The fragility of isolation boundaries | 6.1.2, 6.7.3 |
| Unpredictability (steal time) | 6.5.2 |

## 6.1.2 The tension between isolation and sharing `[mechanism]`

Virtualization has two contradictory goals to satisfy:

```
ISOLATION: Tenants must not see or affect each other
SHARING  : Resources must be divided efficiently (idle resources = loss)
```

These two **directly conflict.** Perfect isolation requires giving every tenant separate
physical hardware — and then there's no sharing. Perfect sharing requires using everything in
common — and then there's no isolation.

**The hypervisor's job is to stand somewhere between those two extremes.** And where it stands
determines what is isolated and what isn't:

| Resource | Isolation status | Consequence |
|---|---|---|
| CPU time | ✅ Divided by the scheduler | Relatively fair |
| RAM capacity | ✅ Separated by page tables | A hard boundary |
| **L3 cache** | ❌ **Shared** | **Noisy neighbors** |
| **Memory bandwidth** | ❌ **Shared** | **Noisy neighbors** |
| **PCIe lanes** | ⚠️ Partially | Contention under heavy I/O |
| Disk/network bandwidth | ⚠️ Can be quota'd | Improved by Nitro |

> **This is the root of noisy neighbors.** The hypervisor can guarantee you 8 vCPUs and 32 GB
> of RAM — those are countable, divisible resources. **But it cannot guarantee your share of
> the L3 cache or of memory bandwidth.**
>
> In Phase 2.3.4 we said "shared L3 is the mechanism of noisy neighbors." Now we complete that
> statement: the hypervisor **cannot put a quota on it**, because cache allocation happens
> inside the hardware — there is no notion of a "tenant" attached to whoever filled a cache
> line.
>
> (Technologies like Intel's CAT — Cache Allocation Technology — solve this partially, but
> they aren't widely used.)

> **🤔 Think 6.1**
> Two `c7i.2xlarge` instances sit on the same physical server. The first instance only does CPU
> computation (a small data set that fits in cache). The second instance scans a 100 GB data
> set from end to end.
>
> a) Which one is affected by the other, and why?
> b) What does the affected instance's CPU utilization show? Does steal time go up?
> c) How would you notice this situation through monitoring?
> *(Answer: at the end of the phase)*

---

# 6.2 Hypervisor types

## 6.2.1 Type 1 — bare metal `[mechanism]`

```
┌─────────┬─────────┬─────────┐
│   VM1   │   VM2   │   VM3   │   ← guest operating systems
├─────────┴─────────┴─────────┤
│        HYPERVISOR           │   ← directly on the hardware
├─────────────────────────────┤
│         HARDWARE            │
└─────────────────────────────┘
```

The hypervisor runs **directly on the hardware.** There is no operating system beneath it —
it is itself something like a minimal operating system.

| Property | Value |
|---|---|
| Performance | **High** — no layer in between |
| Attack surface | **Small** — minimal code base |
| Used in | **All production clouds** |
| Examples | KVM, Xen, VMware ESXi, Hyper-V |

## 6.2.2 Type 2 — hosted `[mechanism]`

```
┌─────────┬─────────┐
│   VM1   │   VM2   │
├─────────┴─────────┤
│    HYPERVISOR     │   ← runs like an ordinary application
├───────────────────┤
│     HOST OS       │   ← Windows, macOS, Linux
├───────────────────┤
│     HARDWARE      │
└───────────────────┘
```

The hypervisor runs as **an ordinary application** on top of a host operating system.

| Property | Value |
|---|---|
| Performance | **Low** — every access goes through the host OS |
| Setup | Easy |
| Used in | Development, testing, desktops |
| Examples | VirtualBox, VMware Workstation, Parallels |

> **Why is Type 2 slow?** Every privileged operation of the guest OS goes first to the
> hypervisor, from there to the host OS, and from there to the hardware. **Two layers of
> abstraction.**
>
> This is the exact opposite of the "remove the layers" idea you saw in Phase 5.4 — and it pays
> the same price.

## 6.2.3 The special case of KVM `[concept]`

KVM (Kernel-based Virtual Machine) is an interesting hybrid: **a module that turns the Linux
kernel itself into a Type 1 hypervisor.**

```
Linux kernel + KVM module = Type 1 hypervisor
But it keeps running as ordinary Linux at the same time
```

This gives the hypervisor Linux's entire driver ecosystem, its scheduler and its memory
management for free. **That's why KVM became dominant in the cloud.**

> **Cloud connection:** **AWS's Nitro hypervisor is KVM-based** and has been shrunk
> dramatically — it only does CPU and memory virtualization. Networking, storage and management
> have been moved to **separate physical cards** (6.6.4).
>
> Historical note: before 2017 AWS used **Xen**. The move to Nitro was one of the biggest jumps
> in cloud performance, and section 6.6.4 of this phase explains why.

---

# 6.3 Hardware-assisted virtualization

**This section is the heart of the phase.** The cost of virtualization is born here, and every
optimization effort is aimed at reducing that cost.

## 6.3.1 Ring levels `[mechanism]`

On the x86 architecture the CPU has four privilege levels (rings). In practice two are used:

```
Ring 0  →  Kernel — can execute ALL instructions
Ring 3  →  User   — privileged instructions FORBIDDEN
```

**What is a privileged instruction?** An instruction that directly affects hardware: changing
the page table (`MOV CR3`), masking interrupts (`CLI`/`STI`), writing to I/O ports.

If a user program tries to execute one, the CPU **raises a trap** and control passes to the
kernel. That is the basis of the virtual memory protection from Phase 2.5.

## 6.3.2 The original problem of virtualization `[mechanism]`

Now picture a guest operating system. It is **also a kernel** and expects to run in ring 0. But
the real hypervisor is sitting in ring 0.

```
Put the guest OS in ring 0 → it crushes the hypervisor, no isolation
Put the guest OS in ring 3 → its privileged instructions won't run
```

**Pre-VT-x/AMD-V solutions (both were bad):**

| Method | How | The problem |
|---|---|---|
| **Binary translation** | Rewrite privileged instructions at runtime | Complex, slow, high risk of bugs |
| **Paravirtualization** | Modify the guest OS so it calls the hypervisor | **The guest OS has to be modified** |

> Early versions of Xen used paravirtualization — which is why a special kernel was required.
> That's why it couldn't run Windows.

## 6.3.3 VT-x / AMD-V — the hardware solution `[mechanism]`

Intel VT-x (2005) and AMD-V added **a new dimension** to the CPU: a mode distinction
perpendicular to the ring levels.

```
              VMX ROOT MODE                 VMX NON-ROOT MODE
              (hypervisor)                  (guest)
           ┌──────────────┐             ┌──────────────┐
  Ring 0   │  HYPERVISOR  │             │   GUEST OS   │  ← ring 0, but isolated
           ├──────────────┤             ├──────────────┤
  Ring 3   │  management  │             │  GUEST APP   │
           └──────────────┘             └──────────────┘
```

**The solution is elegant:** the guest OS runs **genuinely in ring 0** — it believes it has
full authority and doesn't need to be modified. But because it is in non-root mode, the CPU
automatically falls back into root mode for the operations **the hypervisor considers
critical.**

That transition is called a **VM exit**.

## 6.3.4 VM exit — the real cost of virtualization `[mechanism]`

```
Guest is running (non-root)
   ↓ a privileged event (I/O, interrupt, CR3 write, CPUID...)
VM EXIT → the CPU switches to root mode
   ↓
The hypervisor handles the event
   ↓
VM ENTRY → the CPU returns to non-root mode, the guest continues
```

**What does one VM exit cost?**

| Cost item | Explanation |
|---|---|
| Saving/restoring state | The entire CPU state (the VMCS structure) is written — not to disk, but to memory |
| **Pipeline flush** | The same mechanism as the branch misprediction penalty in Phase 1.4 |
| **Cache/TLB pollution** | The hypervisor's code and data occupy the cache |
| Hypervisor processing time | Handling the event itself |

```
Typical VM exit cost: 1,000 – 5,000 cycles  (≈ 0.3 – 1.7 μs)
```

> **Put that number on the ladder from Phase 2.1.1:**
> ```
> L1 access       :          4 cycles
> RAM access      :        200 cycles
> VM EXIT         :  1,000–5,000 cycles   ← 5–25× more expensive than RAM
> SSD access      :   ~200,000 cycles
> ```
> A VM exit is far more expensive than a memory access but far cheaper than a disk access.
> **That's why a single VM exit is irrelevant — hundreds of thousands per second are a
> disaster.**

**The virtualization tax is exactly this:** number of VM exits × cost of a VM exit.

| Era | Tax |
|---|---|
| The binary translation era (2000s) | 30–50% |
| VT-x + EPT (2010s) | 5–15% |
| **Nitro / SR-IOV (today)** | **Under 1%** |

**The entire optimization strategy fits in one sentence: reduce the number of VM exits.** The
next three sections (6.4, 6.5, 6.6) are all different ways of doing that:

| Section | Where it reduces VM exits |
|---|---|
| 6.4 EPT/NPT | In memory access |
| 6.5 CPU pinning | In scheduling |
| 6.6 SR-IOV / virtio | In I/O (**the biggest source**) |

> **🤔 Think 6.2**
> A VM receives 200,000 network packets per second. Assume each packet requires one VM exit
> (emulated I/O). A VM exit costs 2,000 cycles, and the CPU runs at 3 GHz.
>
> a) How much of the CPU do the VM exits alone consume?
> b) Why is this the reason SR-IOV exists?
> *(Answer: at the end of the phase)*

![Figure 6.1 — VMX root / non-root modes and the VM exit cycle](diagrams/hw-6-01-vmx-modes.png)
*Figure 6.1 — The mode distinction perpendicular to the ring levels, and the transitions
between guest and hypervisor.*

---

# 6.4 Memory virtualization

## 6.4.1 The problem of two-level address translation `[mechanism]`

In Phase 2.5 we learned about translation from virtual memory to physical memory. **Under
virtualization that translation happens twice:**

```
Guest application's virtual address (GVA)
        ↓  the guest OS's page table
Guest physical address (GPA)           ← the guest thinks this is "real RAM"
        ↓  the hypervisor's translation  ← but it isn't!
Host physical address (HPA)
```

**The guest operating system believes the GPA is a real RAM address.** Maintaining that
illusion is the hypervisor's job.

## 6.4.2 Shadow page tables — the old, expensive method `[concept]`

The pre-VT-x solution: the hypervisor kept a **shadow page table** for every guest — a combined
table going straight from GVA to HPA.

**The problem:** every time the guest changes its own page table (a new process, an `mmap`, a
page fault — that is, **constantly**) the hypervisor has to find out and update the shadow.

```
Every page table change → VM EXIT → update the shadow → VM ENTRY
```

For memory-intensive workloads this was **the single largest item in the virtualization tax.**

## 6.4.3 EPT / NPT — the hardware solution `[concept]`

Intel EPT (Extended Page Tables) and AMD NPT (Nested Page Tables) moved **the second
translation into hardware.**

```
The MMU now walks TWO tables at once:
  Guest page table (managed by the guest)   →  GPA
  EPT table (managed by the hypervisor)     →  HPA

In hardware, without the hypervisor stepping in.
```

**The gain:** the guest can change its own page table as much as it likes — **no VM exit.** The
virtualization tax for memory-intensive workloads fell from 10–20% to 2–3%.

**The price:** the page table walk got longer.
```
A 4-level walk without virtualization :  4 memory accesses
A two-level walk with EPT             : 24 memory accesses (worst case)
```

> **This makes the TLB (Phase 2.5.3) even more critical under virtualization.** A TLB miss can
> now cost 24 memory accesses instead of 4.
>
> **And this is the answer to why huge pages (Phase 2.5.4) make a bigger difference in virtual
> machines:** they both reduce TLB pressure and shorten the levels of the two-level walk. That
> is why using huge pages on database VMs is the standard recommendation.

## 6.4.4 Memory overcommit `[mechanism]`

The hypervisor can commit **more virtual RAM than there is physical RAM.**

```
Physical: 256 GB
Committed: 32 VMs × 16 GB = 512 GB   ← 2× overcommit
```

**Why does it work?** Because most VMs don't use all the RAM they've been promised **at the
same time.** It's the hypervisor-level version of the "demand paging" logic from Phase 2.5.

**How it's made to work — three mechanisms:**

| Mechanism | What it does | Its risk |
|---|---|---|
| **Balloon driver** | A driver inside the guest "inflates" memory, forcing the guest OS to release pages | Memory pressure inside the guest |
| **KSM** (Kernel Same-page Merging) | Merges identical pages into a single physical page | CPU cost, **side-channel risk** |
| **Swap** | Swapping at the hypervisor level | **Disaster** — the guest slows down without knowing why |

> **The balloon driver is clever:** the hypervisor can't ask the guest for memory (it doesn't
> know the guest OS's internal structures). Instead it puts a driver inside the guest; that
> driver allocates memory and "inflates", the guest OS feels memory pressure and frees its own
> pages. **The hypervisor runs the guest's own memory manager on its own behalf.**

> **Cloud connection — critical:** **memory overcommit is NOT done on AWS EC2.** If your
> instance says 32 GB, 32 GB of physical RAM has been set aside.
>
> **Why?** Because overcommit means unpredictable performance, and predictability is the basis
> of the cloud contract. Remember the thrashing feedback loop in Phase 2.6: when memory runs
> short, the system doesn't degrade gradually — it **falls off a cliff.**
>
> This has a practical consequence: **the habit brought from enterprise VMware environments —
> "commit RAM generously, there's overcommit anyway" — burns money on AWS.** Every GB really is
> allocated, and really is billed.

> **🔧 See it on your machine** (optional)
> ```bash
> # Are you inside a virtual machine?
> systemd-detect-virt
> lscpu | grep -i hypervisor
> ```
> **Expected output (on EC2):**
> ```
> kvm
> ```
> ```
> Hypervisor vendor:      KVM
> Virtualization type:    full
> ```
> If it returns `none` you're on a physical machine (or a bare metal instance).

---

# 6.5 CPU virtualization and steal time

## 6.5.1 How a vCPU is scheduled `[mechanism]`

**The basic fact:** a vCPU **is not** a physical core. It is **a thread** scheduled onto a
physical core.

```
Physical server: 64 cores
Running on it  : 40 VMs × 4 vCPU = 160 vCPU

160 vCPU → 64 physical cores   (2.5× overcommit)
```

The hypervisor's scheduler decides which vCPU runs on which physical core and for how long.
**Exactly like the operating system scheduling processes — one level up.**

> **We now complete the definition of a vCPU from Phase 1.5.3.** There we said: on x86, 1 vCPU
> = 1 SMT thread; on Graviton, 1 vCPU = 1 physical core. Now we add: **even that vCPU isn't
> continuously yours** — it can wait its turn in the scheduler.

## 6.5.2 Steal time — the measure of waiting `[application]`

**Steal time:** the percentage of time a vCPU **was ready to run but could not find a physical
core.**

```
Guest OS:   "I have work to run!"
Hypervisor: "It isn't your turn, wait."
            ↑ this waiting is recorded as steal time
```

**The critical point:** the guest OS **can see** it (the hypervisor tells it) but **cannot
prevent** it.

```bash
top     # the "st" field on the CPU line
vmstat 1
mpstat -P ALL 1
```
**Expected output (healthy):**
```
%Cpu(s): 23.1 us,  4.2 sy,  0.0 ni, 72.5 id,  0.1 wa,  0.0 hi,  0.1 si,  0.0 st
                                                                          ↑ 0.0 = good
```
**Expected output (a problem):**
```
%Cpu(s): 41.2 us,  6.1 sy,  0.0 ni, 33.4 id,  0.2 wa,  0.0 hi,  0.2 si, 18.9 st
                                                                         ↑ 18.9% being stolen
```

**Interpretation table:**

| Steal time | What it means | What to do |
|---|---|---|
| 0–2% | Normal | Nothing |
| 2–10% | Mild contention | Watch it; if persistent, consider moving |
| **10–25%** | **Serious** | Restart the instance (it may land on another host) or change type |
| **25%+** | **Unacceptable** | Move it urgently; dedicated/larger instance |

> **The most important property of steal time: it proves a slowdown that isn't your fault.**
>
> Your application got slower, CPU utilization looks low, the code didn't change. If steal time
> is 20%, **the answer has been found** — you're competing with other tenants for a physical
> core. It's a measurement that saves you hours of profiling code.

**Steal time has two separate causes — don't confuse them:**

| Cause | Mechanism | Solution |
|---|---|---|
| **The host is overcommitted** | Real contention for physical cores | Restart (another host), a larger instance, a dedicated host |
| **Your CPU credits ran out** (t family) | AWS is throttling you **deliberately** | **Change the instance type** — t → m/c |

> **The second one is badly misunderstood.** When credits run out on a `t3.medium`, steal time
> spikes — but there is no neighbor involved; AWS is making your vCPU wait in order to bring it
> down to baseline performance. Restarting the instance **does nothing at all**; the only
> solution is to leave the t family (or accept the bill for `unlimited` mode).

## 6.5.3 Credit-based versus dedicated CPU `[application]`

| | t family (credit-based) | m/c/r family (dedicated) |
|---|---|---|
| Baseline performance | 10–40% of the vCPU | **100%** |
| Burst | To full speed, using credits | None — it's already at full speed |
| If credits run out | **Drops to baseline** | — |
| Predictability | **Low** | **High** |
| Cost | Low | High |

**Credit mechanics:**
```
t3.medium: 2 vCPU, 20% baseline
  While idle    : 24 credits accrue per hour
  At full speed : 2 credits spent per minute (1 per vCPU)
  Credits at 0  : you drop to 20% performance
  Max accrual   : 24 hours' worth of earnings
```

**The decision rule:**

| Workload pattern | Family |
|---|---|
| Intermittent, low average (dev/test, small web) | ✅ **t** |
| Sustained load (production API, database, worker) | ✅ **m/c/r** |
| Short intense jobs with long gaps in between | ✅ **t** |
| **If you're going to run a load test** | ⚠️ **Don't test on t** — the result will mislead you |

> **The most common mistake:** running a production workload on t3 and concluding "AWS is
> slow". It is **exactly the same model** as the "Up to 10 Gigabit" trap in Phase 5.2.3 — and
> it produces the same symptom: perfect for the first few minutes, then collapse.

> **🤔 Think 6.3**
> An API running on a `t3.large` responds perfectly every day between 09:00 and 09:40, after
> which response times go up 5×, and it recovers after 18:00 in the evening. In CloudWatch, CPU
> utilization looks **steady at around 40%** all day long.
>
> a) What is happening?
> b) Why does CPU utilization look constant — is that a contradiction?
> c) Propose two different solutions and compare them on cost.
> *(Answer: at the end of the phase)*

---

# 6.6 I/O virtualization

**I/O is the largest source of VM exits.** This section describes three generations of
solutions built to reduce that number — and each generation reduces the previous one's VM
exits.

## 6.6.1 Emulation — the first generation `[concept]`

The hypervisor presents the guest with a virtual device that **imitates a real piece of
hardware** (an Intel e1000 network card, for example).

```
The guest driver writes to a register
   → VM EXIT
   → the hypervisor interprets it as "a register was written"
   → forwards it to the real hardware
   → VM ENTRY
```

**Every register access is a VM exit.** Sending a single packet can require dozens of exits.

| Pro | Con |
|---|---|
| No change needed in the guest OS (the standard driver works) | **Very slow** |

## 6.6.2 Paravirtualization / virtio — the second generation `[concept]`

**The idea:** stop imitating real hardware. Put a driver in the guest that **knows it is
virtual** and let it talk to the hypervisor over an efficient protocol.

**virtio** is the standard for this. Its basic mechanism is a **shared ring buffer** — the same
thing as the NIC ring buffer in Phase 5.1.3, but between the guest and the hypervisor.

```
Guest              Shared memory              Hypervisor
  │                ┌───┬───┬───┬───┐               │
  ├── descriptor →│   │   │   │   │→ reads ───────┤
  │                └───┴───┴───┴───┘               │
  │                                                 │
  └── ONE notification (VM exit) can carry 100 requests
```

**The gain:** **batched notification** instead of a VM exit per request. A 10–20× speedup is
typical.

> **The idea should feel familiar:** we did the same thing with NAPI in Phase 5.1.4 — many
> packets per round instead of an interrupt per packet. **Batching solves the same problem at
> every layer.**

## 6.6.3 SR-IOV — the third generation `[concept]`

**The idea:** take the hypervisor **completely out** of the I/O path.

An SR-IOV capable physical device (a NIC, an NVMe drive) presents itself as multiple **virtual
functions (VFs)**. Each VF is assigned **directly** to a VM.

```
        ┌──────── PHYSICAL NIC ────────┐
        │  PF  │ VF1 │ VF2 │ VF3 │ VF4 │
        └──────┴──┬──┴──┬──┴──┬──┴──┬──┘
                  │     │     │     │
                 VM1   VM2   VM3   VM4    ← direct, no hypervisor
```

**The result: NO VM exits for normal I/O.** The guest's driver talks to the hardware directly.

| | Emulation | virtio | **SR-IOV** |
|---|---|---|---|
| VM exit frequency | Very high | Medium | **Almost zero** |
| Performance | 20–40% | 70–90% | **95–99%** |
| Latency | High | Medium | **Low** |
| Live migration | ✅ | ✅ | ❌ **Hard** |

**Precondition: the IOMMU.** We promised this in Q4 of Phase 4; now we complete it:

> If the VM's driver talks to the hardware directly, it also means it is **supplying DMA
> addresses directly.** A malicious (or buggy) guest could point a DMA target at **another VM's
> memory** — and because DMA bypasses the CPU, no page table check would stop it.
>
> **The IOMMU (Intel VT-d / AMD-Vi) is exactly this: an MMU for devices.** It translates and
> bounds every DMA access — a VF can only write into its own VM's memory.
>
> **Without an IOMMU, SR-IOV removes isolation entirely.** That's why it is a condition for
> SR-IOV's existence, not an option.

**The price — live migration:** because the VM is bound directly to a piece of physical
hardware, it can't be hot-migrated to another server. This is a serious operational constraint
for a cloud provider, and one of the reasons behind AWS's maintenance windows and "instance
retirement" notices.

## 6.6.4 AWS Nitro — the architectural leap `[mechanism]`

Nitro takes the ideas above one step further: **it moves most of the hypervisor's work onto
separate physical hardware.**

```
TRADITIONAL:
┌──────────────────────────────────┐
│  VM1  │  VM2  │  VM3  │  VM4     │
├──────────────────────────────────┤
│  HYPERVISOR                      │  ← networking, storage, management, security
│  (eats 20–30% of the main CPU)   │     ALL of it here
├──────────────────────────────────┤
│  HARDWARE                        │
└──────────────────────────────────┘

NITRO:
┌──────────────────────────────────┐   ┌─────────────────┐
│  VM1  │  VM2  │  VM3  │  VM4     │   │  NITRO CARDS    │
├──────────────────────────────────┤   │  • Network (ENA)│
│  Nitro Hypervisor (minimal KVM)  │←→ │  • Storage (EBS)│
│  CPU + memory only               │   │  • Security     │
├──────────────────────────────────┤   │  • Management   │
│  HARDWARE (nearly all of it      │   │  (separate      │
│   goes to the VMs)               │   │   silicon)      │
└──────────────────────────────────┘   └─────────────────┘
```

**Three separate gains:**

| Gain | Mechanism |
|---|---|
| **Performance** | 100% of the main CPU goes to the customer — virtualization tax under 1% |
| **Predictability** | Network/storage processing sits on separate silicon — **unaffected by noisy neighbors** |
| **Security** | The management plane is **physically** separate — a hardware boundary, not a software one |

> **The third is the most important and the least understood.** In traditional virtualization
> the hypervisor can read the guest's memory — isolation is **a software boundary**, and
> software has bugs. In Nitro the management functions live on a separate card whose access to
> the main CPU is restricted by hardware.
>
> **This is also the answer to "why do bare metal instances exist?":** because the Nitro cards
> can work without virtualization too, AWS can give the customer **the entire physical server**
> and still provide networking, storage and management. `.metal` instances have no hypervisor,
> yet ENA and EBS still work.

> **The strongest connection in this phase:** in Phase 4.3.3 we introduced Nitro in the context
> of DMA. In Phase 5 we saw the idea of offloading. Now it's complete: **Nitro is the "move the
> work from the CPU to hardware" idea scaled up from a NIC feature to a data center
> architecture.**

![Figure 6.2 — A comparison of the traditional hypervisor and the Nitro architecture](diagrams/hw-6-02-nitro-architecture.png)
*Figure 6.2 — Moving the hypervisor's functions off the main CPU onto separate Nitro cards.*

---

# 6.7 Container vs VM — through the eyes of the hardware

## 6.7.1 The fundamental difference `[mechanism]`

```
VMs                                 CONTAINERS
┌────────┬────────┐                ┌────────┬────────┐
│  App   │  App   │                │  App   │  App   │
├────────┼────────┤                ├────────┴────────┤
│ Guest  │ Guest  │                │   (libraries)   │
│   OS   │   OS   │  ← a separate  ├─────────────────┤
├────────┴────────┤     OS in each │ Container engine│
│   HYPERVISOR    │        VM      ├─────────────────┤
├─────────────────┤                │  ONE KERNEL     │ ← SHARED
│    HARDWARE     │                ├─────────────────┤
└─────────────────┘                │    HARDWARE     │
                                   └─────────────────┘
```

**A container is not a "lightweight VM".** A container is **an isolated process.** It can be
seen with `ps` on the host; it has no kernel of its own.

## 6.7.2 cgroups and namespaces `[mechanism]`

Two Linux kernel features make containers possible:

**Namespaces — "what you see"**

| Namespace | What it isolates |
|---|---|
| `pid` | Process IDs — the container has its own PID 1 |
| `net` | Network interfaces, IPs, ports |
| `mnt` | The filesystem view |
| `uts` | Hostname |
| `ipc` | Shared memory, semaphores |
| `user` | UID/GID mapping |

**cgroups — "how much you can use"**

| cgroup controller | What it limits |
|---|---|
| `cpu` | CPU time (quota and shares) |
| `memory` | RAM usage — **exceed it and you get OOM killed** (Phase 2.6.3) |
| `blkio` | Disk I/O bandwidth and IOPS |
| `pids` | Number of processes |

```
Namespace → visibility isolation
cgroup    → resource isolation
The two together → "a container"
```

> **There is no kernel object called a container.** What Docker does is set up the right
> namespaces and cgroups and start a process. "Container" is the name of a packaging and
> tooling ecosystem, not of a kernel feature.

## 6.7.3 A comparison in numbers `[application]`

| Measure | VM | Container |
|---|---|---|
| Startup time | **30–60 s** (a full OS boot) | **50–500 ms** |
| Memory overhead | **512 MB – 2 GB** (guest OS) | **1–10 MB** |
| Disk overhead | Gigabytes (full OS image) | Megabytes (layered) |
| Density (64 GB host) | ~30 VMs | ~1000+ containers |
| CPU overhead | 1–5% (under 1% with Nitro) | **~0%** |
| **Kernel isolation** | ✅ **Separate kernel** | ❌ **Shared kernel** |
| Running a different OS | ✅ (Windows + Linux) | ❌ (same kernel family) |

## 6.7.4 The security boundary — the real distinction `[application]`

**This is the real axis of the container/VM decision.** The performance difference is a matter
of optimization; the isolation difference is a matter of **risk.**

```
VM escape        → requires a hypervisor vulnerability  → RARE, very valuable
Container escape → requires a kernel vulnerability      → MORE COMMON
```

**Attack surface comparison:**

| | VM | Container |
|---|---|---|
| What the attacker must get past | The hypervisor (~100K lines, minimal) | **The Linux kernel (~30M lines)** |
| System call surface | None (the guest is on its own kernel) | **~350 system calls** |

> **The decision rule:**
>
> | Situation | Choice |
> |---|---|
> | Your own code, your own team | ✅ **Container** — the isolation is enough |
> | Microservices, CI/CD, fast scaling | ✅ **Container** |
> | **You're running untrusted code** (customer code, plugins) | ✅ **VM** |
> | **A hard multi-tenant boundary** | ✅ **VM** |
> | You need a different operating system | ✅ **VM** |
> | Compliance requires a hard boundary | ✅ **VM** |

**Hybrid solutions:** AWS Fargate and Firecracker present a container interface while running
**a micro-VM underneath**. A VM that starts in 125 ms with 5 MB of overhead — container
convenience plus VM isolation.

> **Firecracker exists precisely because of this phase:** Lambda and Fargate run code written
> by customers they don't know, on the same physical server. Container isolation is inadequate
> for that risk; a traditional VM, meanwhile, takes 30 seconds to start and 512 MB of space.
> Firecracker strips everything out of a VM (no BIOS, no PCI, no USB) and leaves only what's
> needed.

> **🤔 Think 6.4**
> A SaaS product runs JavaScript plugins written by customers on the server side. The team plans
> to use Docker containers: "a separate container per customer, that'll be isolated."
>
> a) What is the risk in this plan?
> b) What concrete attack scenarios become possible?
> c) Propose three alternatives and state their trade-offs.
> *(Answer: at the end of the phase)*

---

# 6.8 When This Phase Breaks — Failure Signatures

| Symptom | Likely mechanism | Where | First look |
|---|---|---|---|
| `st` high in `top`, the code didn't change | **Steal time** — host contention | 6.5.2 | `vmstat 1`, instance type |
| Fast for the first 40 min, then 5× slower | **t-family credits exhausted** | 6.5.3 | CloudWatch `CPUCreditBalance` |
| Low I/O throughput, high CPU `sy` | Emulated I/O (no virtio/SR-IOV) | 6.6 | `lspci`, driver name (`ena`?) |
| A memory-intensive application slower than expected | EPT two-level walk + TLB misses | 6.4.3 | Use huge pages |
| A container OOM killed "for no reason" | **cgroup memory limit** | 6.7.2 | `memory.max`, `dmesg` |
| `free` inside a container shows the host's RAM | `free` doesn't know about cgroups | 6.7.2 | `/sys/fs/cgroup/memory.*` |
| `nproc` in a VM shows the host's cores | cgroup CPU quota ≠ visible cores | 6.7.2 | Set an explicit limit for the JVM/Go |
| The same code is 15% faster on bare metal | Virtualization tax (older generation) | 6.3.4 | Is it a Nitro generation? |
| A brief freeze after a live migration | VM migration (a non-SR-IOV instance) | 6.6.3 | AWS maintenance notices |
| You slow down when a neighboring instance gets busy | **L3 / memory bandwidth contention** | 6.1.2 | Phase 7.4 — dedicated host |

> **The last two rows call for a distinction:** steal time shows contention for **CPU time** and
> is measurable. Contention for L3 and memory bandwidth **shows up in no counter at all** — it
> appears only as "my application got slower and no metric changed."
>
> **This is the hardest diagnosis in the cloud**, and Phase 7.4 covers exactly it.

---

# Phase 6 — Answers to the Think Questions

## Answer 6.1

**a) Which one is affected?**

**The first instance (the one doing CPU computation) is affected.**

The mechanism: the second instance is scanning a 100 GB data set. That data doesn't fit in L3 —
every access comes into the cache, **evicts** a line, and goes out to RAM.

```
The first instance's data used to fit in L3   → L3 hit, ~40 cycles
The second instance keeps filling L3          → the first one's lines get evicted
The first instance now misses L3              → RAM, ~200 cycles

Same code, memory access 5× slower.
```

This is called **cache pollution.** The second instance also saturates **memory bandwidth** —
the first one's RAM accesses have to queue.

> **The second instance isn't affected by the first**, because it couldn't use the cache anyway;
> its data didn't fit. **The harm is asymmetric: a cache-friendly workload is damaged by a
> cache-hostile neighbor; the reverse doesn't happen.**

**b) What does CPU utilization show? Does steal time go up?**

**Steal time does NOT go up.** That is the critical point of this question.

```
Steal time = the vCPU is ready to run but can't get a physical core
Here       = the vCPU HAS the physical core, but is waiting on memory
```

The first instance's vCPU got its core and is executing instructions — each instruction just
takes longer, because the data comes from RAM instead of cache.

**What you see:**
| Metric | Change |
|---|---|
| CPU utilization (%) | **The same or higher** (more cycles for the same work) |
| Steal time | **0% — doesn't change at all** |
| IPC (instructions per cycle) | **Falls** — but CloudWatch doesn't show it |
| Application response time | **Rises** |

> **This is the most insidious failure in the cloud:** every standard metric is normal, and the
> application is slow. The IPC concept from Phase 1.3.2 comes in here — what falls is IPC, and
> measuring it requires hardware performance counters (`perf`), which are restricted on most
> cloud instances.

**c) How would you notice?**

You can't measure it directly, but it has **indirect signatures:**

1. **Response time rose, and the CPU/memory/disk/network metrics didn't change** → this is the
   first suspect
2. **Same AMI, same code, different performance on different instances** → compare them
3. **Fluctuation over time** — performance changes as neighbors change
4. **If you have `perf stat` access:**
   ```bash
   perf stat -e cache-misses,cache-references,instructions,cycles ./myapp
   ```
   If the `cache-misses` ratio has gone up and IPC has fallen, it's confirmed.

**Solutions:** a Dedicated Host / Dedicated Instance, a larger instance (`.metal` = the whole
server = no neighbors), or making the workload cache-friendly (Phase 2.3.6).

---

## Answer 6.2

**a) How much of the CPU do the VM exits consume?**

```
VM exits per second : 200,000
Cost of a VM exit   :   2,000 cycles
Total cycles        : 200,000 × 2,000 = 400,000,000 cycles/s = 400 M cycles/s

CPU capacity        : 3 GHz = 3,000,000,000 cycles/s

Ratio: 400,000,000 / 3,000,000,000 = 13.3%
```

**13.3% of a single core goes to the VM exits themselves.**

And that is **excluding the actual work** — processing the packet, taking it through the TCP
stack and delivering it to the application hasn't happened yet. That costs CPU on top.

**Worse:** 200,000 pps is a modest number. We calculated it in Phase 4.4.2: a 10 Gbps link with
1500-byte packets produces **833,000 pps.**

```
833,000 × 2,000 = 1,666,000,000 cycles/s = 55.5% of a 3 GHz CPU
```

**A single 10 Gbps NIC makes a core spend more than half of itself on VM exits alone.**

**b) Why SR-IOV exists**

The calculation above shows that **emulated I/O is mathematically impossible at high speed.** At
100 Gbps the number becomes absurd.

```
Emulation : every access → VM exit           → 13–55%+ CPU, doesn't scale
virtio    : batched notification → few exits → 2–5% CPU
SR-IOV    : NO VM exits                      → ~0%
```

SR-IOV brings that cost **to zero** by letting the VM's driver talk to the hardware directly.

> **And that is the precondition for the cloud being able to offer 25/100 Gbps networking.**
> Without SR-IOV (called "enhanced networking" / ENA on AWS) those speeds could be offered — but
> only by eating half the customer's CPU, which means taking back half of what was sold.
>
> **The pattern from Phase 4.2.4 and 5.4 once more:** the bottleneck isn't in the data being
> moved, it's in **the overhead of moving it.**

---

## Answer 6.3

**a) What is happening?**

**The `t3.large` is running out of CPU credits.**

```
t3.large: 2 vCPU, 30% baseline performance
Credits accrued overnight (low traffic)
09:00 — traffic starts, running at full speed, spending credits
09:40 — credits exhausted → drop to 30% baseline performance
18:00 — traffic drops → credits start accruing again
```

The 40 "perfect" minutes in the morning aren't a coincidence — that is **exactly how long the
accrued credits last.**

**b) Why does CPU utilization look steady at 40%?**

**It isn't a contradiction — CloudWatch's `CPUUtilization` metric is relative to the vCPU's
allocated share, not to the whole physical core.**

```
With credits : 40% utilization = 40% of a genuine 2 vCPUs
Without      : 40% utilization = 40% of a THROTTLED vCPU
               (throttled = 30% of physical capacity)

Real work output: 40% × 30% = 12% of physical capacity
```

In other words, the application **looks like it's running at the same percentage while doing a
third of the work.**

> **This metric illusion explains why t-family failures get diagnosed so late.** The metric to
> watch isn't `CPUUtilization`, it's **`CPUCreditBalance`**. If it's approaching zero, an alarm
> should be set.

**c) Two solutions and a cost comparison**

| | Solution 1: `t3.unlimited` | Solution 2: move to `m7i.large` |
|---|---|---|
| What it does | Continue at full speed for an extra fee when credits run out | 100% performance continuously |
| Performance | ✅ Full speed | ✅ Full speed |
| Cost | Base fee + **unpredictable** overage fee | **Fixed and predictable**, higher base |
| Risk | Under sustained heavy load the bill balloons — **can exceed the t3 price** | None |
| When it's right | If the load really is **intermittent** | If the load is **sustained** |

**The right answer in this scenario: Solution 2.**

The reason: the load runs from 09:00 to 18:00, that is, **9 continuous hours a day.** That isn't
an intermittent pattern; it violates the t family's assumption (idle most of the time). In
`unlimited` mode you'd pay 8+ hours of overage every day and would very likely end up more
expensive than `m7i.large` — with an unpredictable bill on top.

> **The general rule:** the t family is right **if your average CPU utilization is below the
> baseline performance.** `t3.large`'s baseline is 30%; if you're using 40% all day, **the t
> family was the wrong choice from the start.**
>
> The same rule applies to "Up to 10 Gigabit" in Phase 5.2.3 — and that parallel isn't a
> coincidence, it's the same commercial model applied to two different resources.

---

## Answer 6.4

**a) The risk in the plan**

**Containers share the host kernel** (6.7.1). Even running inside a container, customer code
**makes system calls into the host's Linux kernel.**

```
The boundary the attacker must get past: the Linux kernel (~30M lines, ~350 system calls)
Not:                                     the hypervisor (~100K lines, minimal surface)
```

The team's assumption — "container = isolation" — **is wrong for untrusted code.** Container
isolation was designed to prevent *accidental* interference, not to stand against a *deliberate*
attacker.

**b) Concrete attack scenarios**

| Scenario | Mechanism |
|---|---|
| **Escape via a kernel vulnerability** | A vulnerability in the syscall layer → code execution on the host → **every customer's data** |
| **Misconfiguration** | `--privileged`, mounting `/var/run/docker.sock`, `CAP_SYS_ADMIN` → **instant escape** |
| **Resource exhaustion (DoS)** | With no cgroup limits, consume CPU/memory/PIDs and take the neighbors down |
| **Side channel** | Inference about a neighboring container's data through the shared L3 cache (6.1.2) |
| **Exhausting kernel resources** | `inotify` watches, file descriptors, the conntrack table — kernel resources cgroups don't count |

> **The second row is the one that happens most often.** Most container escapes come not from a
> sophisticated kernel vulnerability but from **misconfiguration.** Mounting the Docker socket
> into a container gives the attacker the right to start any container they like (including
> `--privileged`) on the host — that isn't an escape, it's **opening the door.**

**c) Three alternatives and their trade-offs**

| Alternative | How | Pro | Con |
|---|---|---|---|
| **1. Micro-VM** (Firecracker/Fargate) | Each customer's code in its own micro-VM | ✅ **VM isolation, ~125 ms startup** | Slightly heavier than a container, infrastructure complexity |
| **2. WASM sandbox** | Run the code in a WebAssembly runtime | ✅ **Very fast, language-level isolation**, microsecond startup | Limited API, existing JS code must be adapted |
| **3. Hardened container** | gVisor/Kata + seccomp + AppArmor + non-root + read-only fs | ✅ Closest to the existing setup | ⚠️ **Still a shared kernel** (gVisor solves it partly), performance loss |

**Additional layers (whichever you choose):**
- Cut off network access entirely or allowlist it (prevent data exfiltration)
- Hard cgroup limits: CPU, memory, PIDs, I/O
- Execution timeouts
- Read-only filesystem, a separate `/tmp` with a size limit

> **Recommended: 1 or 2.**
>
> Firecracker (the technology under Lambda) exists **precisely because of this problem** — AWS
> asked the same question and decided that container isolation was inadequate for untrusted
> code. That is the most convincing thing to say to the team: **"Lambda doesn't do this with
> containers — why would you?"**

---

# Phase 6 — Frequently Asked Questions

> **Q1: "Is 1 vCPU = 1 physical core?"**
>
> **No, and for two separate reasons:**
>
> 1. **SMT/Hyper-Threading:** on x86 instances 1 vCPU = 1 SMT thread, i.e. half a physical core
>    *(Phase 1.5.2)*. On Graviton, 1 vCPU = 1 physical core.
> 2. **Scheduling:** a vCPU is a thread scheduled onto a physical core. If the host is
>    overcommitted it waits its turn → steal time *(6.5.1)*.
>
> **Practical consequence:** `c7g.4xlarge` (16 Graviton vCPUs = 16 physical cores) and
> `c7i.4xlarge` (16 x86 vCPUs = 8 physical cores) **are not the same thing.** Depending on how
> much the workload benefits from SMT, the difference can be 10% or 80%.

> **Q2: "Why do bare metal instances exist? If virtualization is this cheap they seem unnecessary."**
>
> Five reasons:
>
> | Reason | Explanation |
> |---|---|
> | **Running your own hypervisor** | VMware, nested virtualization |
> | **Licensing requirements** | Some software is licensed per physical core |
> | **Hardware performance counters** | Deep profiling with `perf` — restricted in a VM *(Answer 6.1c)* |
> | **Compliance** | Some regulations forbid shared hardware |
> | **The last 1%** | The virtualization tax is low but not zero |
>
> **Note:** thanks to Nitro, `.metal` instances can use ENA and EBS too — there is no hypervisor,
> but the cloud networking/storage services still work *(6.6.4)*.

> **Q3: "Why does my container see the host's entire RAM?"**
>
> Because `free` reads `/proc/meminfo`, and **`/proc` doesn't know about cgroups.** A container
> isn't a VM; it sees the host kernel's `/proc`.
>
> **The right way to read it:**
> ```bash
> cat /sys/fs/cgroup/memory.max      # cgroup v2 limit
> cat /sys/fs/cgroup/memory.current  # actual usage
> ```
>
> **This is a serious production problem:** the JVM or the Go runtime thinks "the host has 256 GB
> of RAM", sizes the heap accordingly, and then gets OOM killed at the 2 GB cgroup limit.
>
> **The fix:** `-XX:+UseContainerSupport` on modern JVMs (on by default), `GOMEMLIMIT` in Go, or
> stating the limit explicitly. The same problem exists for CPU: `nproc` shows the host's cores
> and `GOMAXPROCS` gets set wrong.

> **Q4: "Steal time is 5%. Should I be worried?"**
>
> **It depends on the context — and first, work out which kind of steal time it is:**
>
> | Situation | Assessment |
> |---|---|
> | m/c/r family, 5% sustained | ⚠️ The host is crowded. Watch it; if it's rising, move. |
> | m/c/r family, 5% momentary spike | ✅ Normal |
> | **t family, 5%+** | ⚠️ **Probably credit exhaustion** — check `CPUCreditBalance` |
> | Latency-sensitive application, 5% | ⚠️ **A visible effect on p99** — consider dedicated |
> | Batch job, 5% | ✅ Irrelevant |
>
> **The rule:** what matters is not steal time's absolute value but **its trend and the
> workload's latency sensitivity.** *(6.5.2)*

> **Q5: "What's the difference between a Dedicated Instance and a Dedicated Host?"**
>
> | | Dedicated Instance | Dedicated Host |
> |---|---|---|
> | Hardware | Reserved for you, but **you don't know which physical server** | **A specific physical server** |
> | Placement control | ❌ | ✅ You know which socket/core |
> | Licensing | ❌ Physical core count isn't visible | ✅ **Required for BYOL** |
> | Restart | Can land on different hardware | **Stays on the same hardware** |
> | Cost | High | **Higher** |
>
> **Both solve noisy neighbors.** What sets a Dedicated Host apart is **visibility and
> licensing** — it's the only option for software licensed per physical core, like Windows Server
> or Oracle.

> **Q6: "What exactly did Nitro gain? Are there numbers?"**
>
> | Measure | Before Nitro (Xen) | Nitro |
> |---|---|---|
> | Virtualization tax | 20–30% | **<1%** |
> | Network performance | ~10 Gbps ceiling | **Up to 100 Gbps** |
> | EBS latency | High, variable | **Low, predictable** |
> | Bare metal support | ❌ | ✅ |
> | Security boundary | Software | **Hardware** |
>
> **The biggest gain isn't measurable performance, it's predictability.** Because network and
> storage processing sit on separate silicon, they aren't affected by noisy neighbors — p99
> latencies improved markedly.

> **Q7: "Is nested virtualization possible?"**
>
> Technically yes (VT-x supports nested VMX), but:
>
> - **The performance loss is serious** — every layer adds its own VM exits *(6.3.4)*
> - **It isn't supported on EC2 at AWS** — rent a `.metal` instance, install your own hypervisor,
>   and nest there
> - GCP and Azure support it partially
>
> In practice, if you want nested virtualization in the cloud, the answer is **a bare metal
> instance**.

---

# Phase 6 — Test Yourself

## Part A — Fundamentals

**A1.** What is the difference between a Type 1 and a Type 2 hypervisor?

**A2.** What is a VM exit?

**A3.** What does steal time measure?

**A4.** What are namespaces and cgroups for? What's the difference between them?

**A5.** What is the basic idea of SR-IOV?

**A6.** What problem do EPT/NPT solve?

**A7.** What is the most important security difference between a container and a VM?

## Part B — Mechanism

**B1.** What was the fundamental problem of virtualization before VT-x? Name the two old
solutions and their drawbacks.

**B2.** Name the four items that make up the cost of a VM exit.

**B3.** Why is address translation two-level under virtualization? How does EPT improve it, and
what new cost does it bring?

**B4.** How does a balloon driver work, and why is the hypervisor forced to use this indirect
method?

**B5.** Rank emulation, virtio and SR-IOV by VM exit frequency and explain each one's mechanism
in a sentence.

**B6.** Why is the IOMMU **a precondition** for SR-IOV? What happens without it?

**B7.** The Nitro architecture provides three separate gains. Explain all three with their
mechanisms.

## Part C — Application and reasoning

**C1.** On an `m7i.xlarge`, `top` shows: `%Cpu(s): 35 us, 5 sy, 42 id, 0 wa, 18 st`. What's your
diagnosis? There could be two different causes — how do you tell them apart?

**C2.** A database VM is 18% slower than a bare metal installation on the same hardware. It's a
memory-intensive workload. Likely cause and improvement?

**C3.** A team wants to run 500 microservice instances as containers on 20 hosts instead of on
20 VMs. Calculate the resource saving (VM overhead ~1 GB, container ~10 MB). What risk are they
accepting?

**C4.** You're designing a Lambda-like service: customer code, must start within 100 ms, strong
isolation is essential. Which technology do you choose, and why?

**C5.** A Kafka consumer running on a `t3.xlarge` falls behind (lag rises) for a few hours three
times a day, then catches up. CPU utilization is steady at around 55%. Diagnosis and solution?

**C6.** You're choosing an instance for an application that needs 100 Gbps networking. Justify
with numbers why SR-IOV/ENA is mandatory (packet size 1500 B, VM exit 2000 cycles, CPU 3 GHz).

**C7.** A team moved from `c7i.8xlarge` to `c7i.metal` and saw the application get 12% faster,
but the price went up 4×. Where does that 12% come from, and does this move make sense? How
would you decide?

---

## Answer Key

### Part A

**A1.** Type 1 runs directly on the hardware (no OS beneath it) — high performance, small attack
surface, the standard for production clouds. Type 2 runs as an ordinary application on a host OS
— easy setup, low performance, development environments. *(6.2.1, 6.2.2)*

**A2.** When the guest operating system performs a privileged operation, the CPU switches from
VMX non-root mode to root mode and hands control to the hypervisor. **It is the real source of
virtualization's cost** (1,000–5,000 cycles). *(6.3.4)*

**A3.** The percentage of time a vCPU was ready to run but could not find a physical core.
*(6.5.2)*

**A4.** Namespaces isolate **visibility** (PIDs, network, filesystem, hostname). cgroups limit
**resource usage** (CPU, memory, I/O, process count). Together they make up a container.
*(6.7.2)*

**A5.** A physical device presents itself as multiple virtual functions (VFs) and each VF is
assigned directly to a VM — **the hypervisor leaves the I/O path entirely** and there are no VM
exits. *(6.6.3)*

**A6.** They move the second address translation (guest physical → host physical) into hardware.
That eliminates the constant VM exits shadow page tables required. *(6.4.3)*

**A7.** VMs run **a separate kernel**; escaping requires a hypervisor vulnerability (~100K lines,
minimal surface). Containers **share the host kernel**; a kernel vulnerability is enough to
escape (~30M lines, ~350 system calls). *(6.7.4)*

### Part B

**B1.** *(6.3.2)* The guest OS is also a kernel and expects ring 0, but the hypervisor is there.
Put it in ring 0 and isolation disappears; put it in ring 3 and its privileged instructions won't
run.

| Old solution | Drawback |
|---|---|
| Binary translation | Complex, slow, risk of bugs |
| Paravirtualization | **The guest OS has to be modified** (Windows won't run) |

**B2.** *(6.3.4)* Saving/restoring state (VMCS), the pipeline flush, cache/TLB pollution, and the
hypervisor's time handling the event.

**B3.** *(6.4.1, 6.4.3)* The guest application's virtual address must first be translated to a
guest physical address by the guest OS's page table, then to a real physical address by the
hypervisor — because the address the guest thinks is "physical" isn't real.

EPT moves the second translation into hardware; the guest can change its own page table with no
VM exit. **The new cost:** a page table walk goes from 4 memory accesses to as many as 24 → a TLB
miss is far more expensive → **huge pages are more critical in a VM.**

**B4.** *(6.4.4)* The hypervisor doesn't know the internal structures of the guest's memory
manager and can't ask it for pages directly. Instead it puts a driver inside the guest; the
driver allocates memory and "inflates", the guest OS feels memory pressure and frees its own
pages. **The hypervisor runs the guest's own memory manager for its own purposes.**

**B5.** *(6.6)*
```
Emulation > virtio > SR-IOV   (VM exit frequency, most to least)
```
| Method | Mechanism |
|---|---|
| Emulation | Imitates real hardware; every register access is a VM exit |
| virtio | A shared ring buffer; many requests per notification (batching) |
| SR-IOV | A VF is assigned directly to the VM; the hypervisor leaves the path |

**B6.** *(6.6.3)* With SR-IOV the guest driver hands **DMA addresses directly** to the hardware.
Because DMA bypasses the CPU, page table protection doesn't apply — a malicious or buggy guest
could target another VM's memory. The IOMMU (VT-d/AMD-Vi) acts as an MMU for devices: it
translates and bounds every DMA access. **Without it, SR-IOV removes isolation entirely.**

**B7.** *(6.6.4)*
| Gain | Mechanism |
|---|---|
| Performance | Network/storage/management moved onto a separate card → 100% of the main CPU goes to the customer |
| Predictability | I/O processing on separate silicon → unaffected by noisy neighbors |
| Security | The management plane is physically separate → a **hardware** boundary, not a software one |

### Part C

**C1.** *(6.5.2)*

**Diagnosis:** 18% steal time — the vCPUs are waiting for a physical core. 42% of the CPU looks
idle while the application is slow.

**The two causes and how to tell them apart:**

| Cause | How to tell |
|---|---|
| The host is overcommitted | **m7i is a dedicated family** → credits can't be involved → this is the cause |
| CPU credits exhausted | Only happens in the t family |

Because the question says `m7i`, **the credit possibility is eliminated** — the host is crowded.

**Solution:** (1) Stop/start the instance — it will very likely land on a different host. (2) If
it recurs, a larger instance (a large instance = a large slice of the server = fewer neighbors).
(3) If it's critical, a Dedicated Instance/Host.

**C2.** *(6.4.3)*

**Cause:** the EPT two-level page table walk. On a memory-intensive workload TLB misses are
frequent, and each miss means up to **24** memory accesses instead of the 4 you'd have without
virtualization.

**Improvements:**
1. **Use huge pages (2 MiB)** — they widen TLB reach 512× *(Phase 2.5.4)* and shorten the levels
   of the two-level walk. **The single biggest gain** for databases.
2. Check NUMA placement — are the vCPUs and the memory on the same node *(Phase 2.7)*
3. Instance generation — newer generations have better EPT/TLB hardware
4. Prefer **explicit huge pages** over THP (see Phase 2.5.4 for THP defrag stalls)

**C3.**
```
VM approach        : 500 × 1 GB   = 500 GB overhead
Container approach : 500 × 10 MB  =   5 GB overhead
Saving             : 495 GB  (a 99× reduction)
```
Also: startup time 30–60 s → 50–500 ms, disk overhead from gigabytes to megabytes.

**The risk they're accepting:** *(6.7.4)* **a shared kernel.** A kernel vulnerability or a
misconfiguration affects every container on the same host. And one container crashing the kernel
(or exhausting a kernel resource cgroups don't count) takes down everything on that host.

**Assessment:** **because it's their own code, this is an acceptable trade-off.** The same
decision would be wrong for untrusted customer code. An extra precaution: group the containers
per host by trust/criticality level (limiting the blast radius).

**C4.** *(6.7.4)*

**Choice: a micro-VM — Firecracker (or AWS Fargate).**

**The justification:**
| Requirement | How it's met |
|---|---|
| 100 ms startup | ✅ Firecracker ~125 ms, less with a pre-warmed pool |
| Strong isolation | ✅ **A genuine VM boundary** — a separate kernel |
| Density | ✅ ~5 MB overhead — thousands of micro-VMs per host |

**Why not containers:** untrusted code, a shared kernel → unacceptable risk.

**Why not traditional VMs:** 30–60 s startup, 512 MB+ overhead → doesn't meet the requirement.

> **AWS Lambda made exactly this decision, and wrote Firecracker for it.**

**C5.** *(6.5.3)*

**Diagnosis:** the `t3.xlarge` is running out of CPU credits.

The chain of evidence:
- **The cyclical pattern** (fall behind → catch up → fall behind) is the signature of the
  credit accrual/exhaustion cycle
- **CPU looking steady at 55%** is misleading: once credits run out, the metric is computed
  against the throttled vCPU *(Answer 6.3b)*
- `t3.xlarge`'s baseline is 40%; **average utilization 55% > 40%** → **the t family was the wrong
  choice from the start**

**Confirmation:** CloudWatch `CPUCreditBalance` — if it keeps dropping to zero, it's settled.

**Solution:** move to `m7i.xlarge`. A Kafka consumer is a continuously running workload, not an
intermittent one. `t3.unlimited` would be wrong here — you'd pay overage fees for hours every
day.

**C6.** *(Answer 6.2, 6.6.3)*
```
100 Gbps ÷ (1500 × 8 bits) = 8,333,333 packets/s

Emulated I/O (assuming 1 VM exit per packet):
8,333,333 × 2,000 cycles = 16,666,666,000 cycles/s
                         = 16.7 billion cycles/s

A 3 GHz core = 3 billion cycles/s
→ 16.7 / 3 = 5.6 CORES, for VM exits alone
```

**Conclusion:** the VM exits alone consume 5.6 cores — **excluding** packet processing, the TCP
stack and the application. That means taking back most of the CPU that was sold.

**With SR-IOV the VM exit count goes to zero.** That is why high-speed networking **cannot be
offered economically** in the cloud without SR-IOV (ENA / enhanced networking).

> Also remember Phase 5.2.3: 100 Gbps = 12.5 GB/s, which requires PCIe Gen4 x8. **Two separate
> bottleneck checks: PCIe capacity and VM exit cost.**

**C7.** *(6.3.4, 6.1.2, Q2)*

**Where the 12% comes from — three sources:**

| Source | Share |
|---|---|
| Virtualization tax (under 1% with Nitro) | Small |
| **The absence of noisy neighbors** (all of L3 and memory bandwidth are yours) | **Large** |
| NUMA/placement control, full hardware counters | Medium |

Because Nitro's tax is under 1%, **most of the 12% comes from escaping noisy neighbors**
*(6.1.2)* — the shared L3 and memory bandwidth are no longer being divided up.

**Does it make sense — the decision framework:**

```
4× the price for 12% performance → on its own, a BAD trade
```

**The right comparison isn't `c7i.metal` vs `c7i.8xlarge`, it's this:**
```
c7i.metal (4× price, 12% faster)
    vs
2 × c7i.8xlarge (2× price, 100% more capacity through horizontal scaling)
```

**For a workload that can scale horizontally, `.metal` is almost always wrong.**

**The situations that justify `.metal`:**
- The workload **can't scale horizontally** (a single large database instance)
- **Licensing** is tied to physical cores
- **Compliance** forbids shared hardware
- You need to run your own hypervisor
- **p99 latency is critical** and noisy neighbors are unacceptable (exchanges, real-time bidding)

**How you decide:** first calculate **the monetary value** of that 12%. If this workload
generates revenue and 12% more requests = 12% more revenue, a 4× cost increase only makes sense
if revenue also goes up 4×. In most cases it doesn't.

---

## Scoring

| Correct answers | Assessment |
|---|---|
| 18–21 | You've finished the phase. Phase 7 is waiting for you. |
| 14–17 | Good. Re-read the sections of the ones you got wrong, then move on to Phase 7. |
| 10–13 | Re-read 6.3 (VM exit) and 6.5 (steal time) from the start. |
| 0–9 | Work through the phase again. **This phase is a prerequisite for Phase 7** — don't skip it. |

> **If you missed C1, C5 or C7 in particular, solve them again.** All of Phase 7 is built on
> decisions of that kind.

---

# Phase 6 — Closing and Bridge to Phase 7

## What you carry out of this phase

| Concept | The essence |
|---|---|
| **VM exit** | The single unit of virtualization's cost; all optimization is about reducing it |
| **Steal time** | Proof of a slowdown that isn't your fault — and its two separate causes |
| **EPT/NPT** | Two-level translation; why huge pages are more critical in a VM |
| **SR-IOV + IOMMU** | Taking the hypervisor out of the path, and the security precondition for doing so |
| **Nitro** | The "move the work from the CPU to hardware" idea at data center scale |
| **Resources that can't be quota'd** | L3 and memory bandwidth — the root of noisy neighbors |
| **A container ≠ a lightweight VM** | An isolated process; the security boundary is the kernel |
| **The t family ≠ a cheap m family** | A different performance model — decide by your average utilization |

## Where Phase 7 connects to this

Phase 7 **teaches no new material.** It turns everything you've accumulated from Phase 0 through
Phase 6 into real cloud decisions.

| Phase 7 section | Which phases feed it |
|---|---|
| 7.1 Instance families | Phase 1 (clock, L3), Phase 2 (memory bandwidth), Phase 3 (NVMe), Phase 4 (PCIe) |
| 7.2 Nitro | **Phase 6.6.4** + Phase 4.3.3 + Phase 5 (offloading) |
| 7.3 Choosing EBS | Phase 3 (all of it) |
| 7.4 **Noisy neighbors — the full analysis** | **Phase 6.1.2** + Phase 2.3.4 + Phase 4.2 |
| 7.5 Finding bottlenecks | **The "When This Phase Breaks" tables of every phase** |
| 7.6 Capacity planning | All of them |

> **Phase 7's question will be this:** a workload is described. Which instance? Which disk? Why?
> And when it slows down, **where do you look?**
>
> You'll now answer those questions not from a price list but **from physics.** That was the
> whole point of this map.

---

*Phase 6 complete.* → **[Phase 7 — Cloud Connection](Phase_7_Cloud_Connection.md)**
