# The Physical Layer of Computing — Cloud Engineer Learning Roadmap

---

## Mentor Startup Instructions — for the AI reading this file

This is a roadmap for the physical layer of computing (hardware), and you are the **Socratic mentor** of the learner working through it. If this file has been handed to you, adopt the rules below and wait for the learner to set the direction. When the learner says "**We're on Phase X**" (from the start of the phase) or "**We're on Phase X.Y**" (from a sub-item), continue **without interruption** from that point — don't summarize from the beginning, don't ask permission, don't wander.

**Learning rhythm — the most important rule: the constructive Socratic method.** This is neither pure Q&A nor plain lecturing; it is a blend of the two. For every concept the order is: **(1) ask a guiding question** — like "what do you think needs to be true for X to happen?", a question the learner can reason through **with the knowledge they already have** → **(2) the learner guesses** ("could it be such-and-such?") → **(3) if correct, confirm immediately and give the term:** "yes, exactly — this is called **X**, and it means: …" → **(4) if wrong, don't fuss with a roundabout question, correct it directly** → **(5) once the term is placed, build the concept up together.**

Critical rule: **Do not try to make the learner guess a concept they have no grip on at all.** The question is not for open-ended discovery, but to **build a bridge** from what they know to what they don't. As soon as the answer arrives, name it, define it, don't leave the concept hanging. You do not move to the next concept before the current one is complete.

**Phase opening.** When entering a phase, first give ~2 lines of orientation: what this phase covers + what you assume they know from the previous phase + the general context. Then move to a short explanation of the first concept. (Each phase uses the vocabulary of the previous one; don't break the order.)

**Language and style.**
- English. Give a technical term a short definition in parentheses on first use (e.g., "vCPU (virtual processor)").
- Follow the depth labels by **this map's own definitions**: `[concept]` enough to explain intuitively, `[mechanism]` explain step by step what happens, `[application]` **relate it to a cloud/architecture decision**, `[skip]` just know the name (not needed at the Solutions Architect level).
- Use the **"Cloud connection"** angle of every topic — this is the heart of this map: tying hardware to instance selection / bottleneck detection / an architecture decision. The goal is not to become a transistor designer, but to see the system deeply enough to justify decisions.

**Correction — keep it direct.** When the learner answers wrong, don't hunt for a second question to lead them to the right answer; **state the correct answer clearly**, then give a brief rationale. Indirect steering slows things down and tires people out. When the learner says "just tell me directly," explain directly, no argument.

**`[application]` topics and practice.** In this map, `[application]` does not mean "run a command on a machine"; it means being able to **tie the topic to a concrete cloud/architecture decision** (which EC2 family, which EBS type, which bottleneck — and why). On a topic labeled `[application]`, don't advance until the learner makes this connection. If a hands-on demonstration (on the learner's Ubuntu/Linux machine) fits, you **may suggest it** — but this is a suggestion, not a demand; the decision is the learner's, and you follow their decision.

**Phase closing.** Per this map's "Suggested way to progress," each phase closes with a test or an open-ended question set — to measure understanding, not memorization. When you reach the end of a phase, **suggest the closing questions automatically.** If the learner wants to defer, **defer** — don't impose.

**Phase 7 difference.** Phase 7 is an "apply" phase, not a "learn" phase: not new concepts, but building decisions from real scenarios. There, shift the rhythm from explanation-heavy to scenario/decision-heavy.

**Communication.** No trigger/command system — talk naturally. **The moment you don't understand the learner, don't assume you do; say plainly "I didn't quite understand you"** and ask a clarifying question.

**Boundaries.** One concept at a time; no topic skipping; no unrequested roadmap drift. **Don't track** in-phase progress yourself — the learner manages it and pins the position by saying "We're on Phase X.Y." Feedback is honest and direct.

---

> **Goal:** As a Cloud Solutions Architect, to be able to understand hardware decisions,
> justify instance selections, and detect bottlenecks.
> Not to become a transistor designer — but to see the system deeply enough.

---

## How to read this

You don't move to the next phase before the current one is finished.
Next to each heading is a `[Depth]` label:

- `[concept]` → Being able to explain intuitively how it works is enough
- `[mechanism]` → You need to be able to explain step by step what happens
- `[application]` → You need to be able to relate it to a cloud/architecture decision
- `[skip]` → Not needed at the Solutions Architect level; knowing the name is enough

---

## Phase 0 — Numeric Foundation (The substrate of everything)

The goal of this phase: to understand why a computer works with 0s and 1s,
and how you get from electricity to a logic gate.
If this is skipped, every later concept hangs in the air.

### Topics

**0.1 Number systems**
- What the binary number system is, and why computers use it `[mechanism]`
- Decimal ↔ Binary ↔ Hexadecimal conversion `[mechanism]`
- What a bit and a byte are, and why 8 bits is 1 byte `[concept]`
- What a 32-bit vs 64-bit system means `[application]`

**0.2 The transistor (idea level only)**
- A transistor works like a switch: on/off = 0/1 `[concept]`
- How billions of transistors fit onto a single CPU chip (the idea of scale) `[concept]`
- Why you don't need to know transistor physics `[skip]`

**0.3 Logic gates**
- What AND, OR, NOT gates do `[mechanism]`
- NAND, NOR, XOR `[concept]`
- How a simple adder circuit is built from logic gates (the half-adder idea) `[concept]`
- Why you need this: this is the part where the CPU "does arithmetic"

**0.4 Flip-flops and latches (the basis of memory)**
- How a single bit is stored `[concept]`
- The SR latch, D flip-flop idea `[concept]`
- Why you need this: this is the physical basis of registers and cache

---

## Phase 1 — CPU Architecture (The thinking part)

The goal of this phase: to understand what's inside a CPU, how an instruction is processed,
and what "more cores" or "faster clock" actually means.

### Topics

**1.1 The core components of a CPU**
- ALU (Arithmetic Logic Unit) — the part that does arithmetic `[concept]`
- Register — the temporary memory inside the CPU `[mechanism]`
- Program Counter (PC) — tracks the next instruction `[concept]`
- Instruction Register — holds the current instruction `[concept]`
- Control Unit — coordinates all the parts `[concept]`

**1.2 The Fetch-Decode-Execute cycle**
- How a CPU fetches, interprets, and runs an instruction `[mechanism]`
- How the speed of this cycle relates to clock speed `[mechanism]`
- Cloud connection: why "a higher clock speed is not always better"

**1.3 Clock speed and IPC**
- What GHz means — how many cycles per second `[mechanism]`
- The IPC (Instructions Per Clock) concept `[concept]`
- The Clock × IPC = real performance idea `[application]`
- Cloud connection: which one matters in a single-threaded vs a multi-threaded application

**1.4 Pipeline**
- The assembly-line analogy `[concept]`
- What a pipeline hazard is (data dependency, branch prediction) `[concept]`
- Why some workloads benefit more from the pipeline `[concept]`

**1.5 Core, Thread, Hyper-Threading, vCPU**
- What a physical core is `[mechanism]`
- How a logical core / Hyper-Threading works `[mechanism]`
- What a vCPU is — what a vCPU physically means on EC2 `[application]`
- What an EC2 instance with 8 vCPUs actually uses `[application]`
- Cloud connection: the effect of core count on CPU-bound vs I/O-bound workloads

**1.6 Recognizing CPU families (architecture level)**
- Why the difference between Intel vs AMD vs ARM (Graviton) architectures matters `[application]`
- The Instruction Set Architecture (ISA) concept `[concept]`
- The power-consumption difference between x86-64 and ARM and its relation to cloud cost `[application]`

---

## Phase 2 — Memory Hierarchy (The most critical phase)

The goal of this phase: to understand the speed-capacity-cost triangle.
Almost every architecture decision in the cloud is a reflection of this triangle.

### Topics

**2.1 Why a hierarchy exists**
- How fast the CPU is, how slow RAM is — with numbers `[mechanism]`
- The "memory wall" problem `[concept]`
- The logic of the hierarchy: fast-small-expensive / slow-large-cheap `[mechanism]`

**2.2 Register**
- Inside the CPU, sub-nanosecond access `[concept]`
- Capacity: a few tens of bytes `[concept]`

**2.3 Cache (L1 / L2 / L3)**
- What SRAM is, how it differs from DRAM `[concept]`
- L1: which core it belongs to, size, speed `[mechanism]`
- L2: core-private or shared, size, speed `[mechanism]`
- L3: shared across all cores, size, speed `[mechanism]`
- What a cache hit and a cache miss mean `[mechanism]`
- The concepts of spatial locality and temporal locality `[concept]`
- Cache coherence: same-data problems across multiple cores `[concept]`
- Cloud connection: the effect of a noisy neighbor on L3, the advantage of instances with a large L3

**2.4 RAM (Main Memory)**
- How DRAM works — the capacitor idea, why it's slow `[concept]`
- Row, column, DRAM access time `[concept]`
- Why the DDR4 vs DDR5 difference matters `[concept]`
- Volatile: why it's erased when power is cut `[mechanism]`
- The memory channel and bandwidth concept `[application]`
- Cloud connection: the reason memory-optimized instances exist

**2.5 Virtual memory**
- Physical RAM is not always enough — why the virtual address space was invented `[mechanism]`
- The page and page table concepts `[mechanism]`
- TLB (Translation Lookaside Buffer) — a cache that speeds up the page table `[concept]`
- What a page fault is, how it's resolved `[mechanism]`

**2.6 Swap**
- What happens when RAM fills up `[mechanism]`
- Why swap space is on disk and why it's slow `[mechanism]`
- The OOM Killer — what Linux does when RAM is completely full `[concept]`
- Cloud connection: why swap is dangerous in production, the memory-leak scenario

**2.7 NUMA (Non-Uniform Memory Access)**
- Why memory access is not equal on multi-CPU-socket servers `[mechanism]`
- The local vs remote NUMA node access difference `[mechanism]`
- Cloud connection: NUMA's effect on performance in large EC2 instances

---

## Phase 3 — Storage (Persistent memory)

The goal of this phase: to internalize the trio of IOPS, throughput, and latency.
EBS type selection, the instance-store choice, and database storage design rest on this.

### Topics

**3.1 HDD**
- Mechanical structure: platter, read head, spindle `[concept]`
- What seek time is and why random reads are slow `[mechanism]`
- The sequential vs random read difference, with numbers `[mechanism]`
- Why the HDD is nearly dying out in the cloud `[application]`

**3.2 SSD**
- How NAND Flash works — the floating-gate transistor idea `[concept]`
- Cell types: SLC, MLC, TLC, QLC — the balance of speed, endurance, and cost `[mechanism]`
- Why the concept of seek time carries a different meaning on an SSD `[concept]`
- The write amplification and erase block concept `[concept]`
- What wear leveling is `[concept]`

**3.3 Interfaces: SATA vs NVMe**
- SATA: old interface, limited bandwidth, bottlenecks the SSD `[mechanism]`
- NVMe: runs over PCIe, the advantage of queues and queue depth `[mechanism]`
- Practical latency and throughput numbers: SATA SSD vs NVMe SSD `[application]`
- Cloud connection: instance store = NVMe, EBS = block storage over the network

**3.4 IOPS, Throughput, and Latency**
- Latency: how long a single operation takes (ms, μs, ns) `[mechanism]`
- IOPS: the number of operations per second `[mechanism]`
- Throughput: the amount of data per second (MB/s, GB/s) `[mechanism]`
- When each of IOPS and throughput becomes the bottleneck `[application]`
- Small random I/O = IOPS-limited, large sequential I/O = throughput-limited `[application]`
- Cloud connection: the choice of EBS gp3, io2, st1, sc1 rests on this knowledge

**3.5 RAID (general idea)**
- The conceptual differences of RAID 0, 1, 5, 10 `[concept]`
- The role of RAID in the cloud vs managed storage services `[application]`

---

## Phase 4 — System Buses and I/O (Where the parts connect to each other)

The goal of this phase: to understand how the CPU, RAM, and disk talk to each other.
This phase explains the bottleneck in GPU workloads and PCIe bandwidth.

### Topics

**4.1 The bus concept**
- What a bus is — the data-path idea `[concept]`
- Address bus, data bus, control bus `[concept]`
- The link between bandwidth and latency `[concept]`

**4.2 PCIe (Peripheral Component Interconnect Express)**
- The lane concept — x1, x4, x8, x16 `[mechanism]`
- PCIe generations: Gen 3, Gen 4, Gen 5 — bandwidth differences `[mechanism]`
- How NVMe, GPU, and network cards connect to PCIe `[mechanism]`
- Cloud connection: the CPU-GPU bandwidth bottleneck in GPU instances

**4.3 DMA (Direct Memory Access)**
- How device data reaches RAM without the CPU `[mechanism]`
- Why, without DMA, every I/O operation keeps the CPU busy `[concept]`
- Cloud connection: the Nitro System's use of DMA

**4.4 The interrupt mechanism**
- What a hardware interrupt is `[mechanism]`
- Polling vs interrupt comparison `[concept]`
- The IRQ and interrupt handler concepts `[concept]`
- Cloud connection: interrupt overhead in high-packet-rate network operations

**4.5 Chipset and motherboard architecture (overview)**
- The Northbridge, Southbridge concept (historical) `[concept]`
- Modern single-chip architecture `[concept]`
- Integration of the memory controller into the CPU `[concept]`

---

## Phase 5 — Network Hardware (The cloud's blood vessels)

The goal of this phase: to understand how a packet physically travels,
how the network card works with the CPU, and the hardware basis of bandwidth and latency.

### Topics

**5.1 NIC (Network Interface Card)**
- The NIC's basic job `[mechanism]`
- Ethernet frame and MAC address `[mechanism]`
- Transmit and receive buffers `[mechanism]`
- Interrupt vs polling modes (NAPI) `[concept]`

**5.2 Switching and the physical network**
- How a switch works — the MAC table `[mechanism]`
- Full duplex and half duplex `[concept]`
- The 1G, 10G, 25G, 100G Ethernet difference `[concept]`

**5.3 Bandwidth and latency (from a hardware standpoint)**
- Bandwidth: the capacity of the physical line `[mechanism]`
- Latency: the end-to-end delay of one packet/request (propagation + queuing + processing), independent of bandwidth `[mechanism]`
- Why latency can be high even when bandwidth is high `[mechanism]`
- Propagation delay, transmission delay, queuing delay `[concept]`

**5.4 RDMA and high-performance networking (awareness)**
- What RDMA is, why it's used in HPC and ML clusters `[concept]`
- InfiniBand and RoCE `[concept]`
- Cloud connection: the reason AWS EFA (Elastic Fabric Adapter) exists

---

## Phase 6 — Virtualization Hardware (The technical foundation of the cloud)

The goal of this phase: to understand what actually lies beneath EC2.
Without this phase, cloud decisions are made blindly.

### Topics

**6.1 Why virtualization is necessary**
- Giving a single physical server to multiple customers `[concept]`
- The tension between isolation and resource sharing `[mechanism]`

**6.2 Hypervisor types**
- Type 1 (bare metal): directly on the hardware `[mechanism]`
- Type 2 (hosted): on top of a host OS `[mechanism]`
- Examples: KVM, Xen, VMware ESXi, Hyper-V, VirtualBox `[concept]`
- Cloud connection: AWS Nitro = KVM-based Type 1

**6.3 Hardware-assisted virtualization**
- What Intel VT-x and AMD-V are, and why they matter `[mechanism]`
- The ring level concept: ring 0 (kernel), ring 3 (user) `[mechanism]`
- The VMX root / non-root mode concept `[concept]`
- What a VM exit is and why it's costly `[mechanism]`

**6.4 Memory virtualization**
- The shadow page table concept `[concept]`
- Extended Page Tables (EPT) / Nested Page Tables (NPT) `[concept]`
- The memory overcommit mechanism `[mechanism]`
- Balloon driver, KSM (Kernel Same-page Merging) `[concept]`
- Cloud connection: where an EC2 instance sits in physical RAM

**6.5 CPU virtualization**
- Scheduling a vCPU onto a physical core `[mechanism]`
- What CPU steal time is, how it's measured `[application]`
- Credit-based CPU (t series) vs dedicated CPU (c, m, r series) `[application]`

**6.6 I/O virtualization**
- Emulated I/O vs paravirtualization `[concept]`
- What virtio is `[concept]`
- SR-IOV (Single Root I/O Virtualization) `[concept]`
- Cloud connection: AWS Nitro moves the NIC and storage onto separate hardware, and why this yields a performance gain

**6.7 Container vs VM (from the hardware's eye)**
- The cgroup and namespace Linux kernel features `[mechanism]`
- Why containers share the kernel and VMs don't `[mechanism]`
- The security isolation difference `[application]`
- The startup time and memory overhead difference, with numbers `[application]`

---

## Phase 7 — Cloud Connection (Where everything comes together)

The goal of this phase: to turn every learned topic into real cloud decisions.
This phase is not a "learn" phase but an "apply" phase.

### Topics

**7.1 Reading EC2 instance families through their physical basis**
- General purpose (m): why a balanced CPU/RAM ratio means what it does `[application]`
- Compute optimized (c): clock speed and L3 cache gain importance `[application]`
- Memory optimized (r, x): memory bandwidth and capacity priority `[application]`
- Storage optimized (i, d): NVMe instance store, high IOPS `[application]`
- Accelerated (p, g, trn): GPU and PCIe bandwidth `[application]`
- Graviton (arm): ARM architecture, power/performance balance `[application]`

**7.2 Understanding the AWS Nitro System**
- What a Nitro card is — moving the NIC and storage onto separate hardware `[mechanism]`
- Why this architecture reduces the noisy neighbor `[application]`
- The reason bare metal instances exist `[application]`

**7.3 EBS selection through its physical basis**
- gp3: IOPS and throughput tuned independently `[application]`
- io2 / io2 Block Express: sub-millisecond latency, high IOPS `[application]`
- st1: throughput-optimized HDD — sequential access `[application]`
- sc1: cold HDD — archive `[application]`
- Which workload wants which — a decision table `[application]`

**7.4 Noisy neighbor — full analysis**
- The L3 cache pollution mechanism `[application]`
- Memory bandwidth contention `[application]`
- PCIe bus saturation `[application]`
- AWS solutions: Dedicated Host, Placement Groups, Nitro `[application]`

**7.5 Bottleneck detection (integrated)**
- Distinguishing CPU-bound, memory-bound, I/O-bound, network-bound `[application]`
- The monitoring metrics of each bottleneck (from a CloudWatch view) `[application]`
- "My app is slow" → asking the right questions `[application]`

**7.6 Capacity planning**
- The process of choosing an instance type for a workload `[application]`
- The right-sizing concept `[application]`
- The costs of over-provisioning vs under-provisioning `[application]`

---

## Learning-depth summary

| Topic | Required depth | Why |
|---|---|---|
| Number systems | Mechanism | The language of everything that follows |
| Logic gates | Concept | Understanding what the CPU does |
| CPU pipeline / core | Mechanism | Instance selection, CPU-bound analysis |
| Cache hierarchy | Mechanism | The area that most often affects architecture decisions |
| RAM and virtual memory | Mechanism | Memory leak, swap, OOM scenarios |
| NUMA | Concept | Performance on large instances |
| HDD / SSD physics | Concept | Justifying EBS type selection |
| IOPS / Throughput | Application | Every storage decision rests here |
| PCIe | Concept | Understanding GPU instances |
| DMA / Interrupt | Concept | Understanding high-I/O workloads |
| NIC and network hardware | Concept | Bandwidth, latency decisions |
| Hypervisor | Mechanism | The foundation of EC2 |
| Virtualization hardware | Mechanism | vCPU, steal time, noisy neighbor |
| Container vs VM | Application | Decisions made every day |
| EC2 instance families | Application | Direct work output |
| CMOS, transistor physics | Skip | Not the Solutions Architect's job |

---

## Suggested way to progress

- Each phase uses the core vocabulary of the previous one — skipping the order breaks the accumulation
- Phases 0-2 are the densest and require patience
- Phases 3-5 move relatively fast — once intuition is gained
- Phase 6 is the most binding phase — the "so that's why it works this way" moments happen here
- Phase 7 is not technical learning but a context-building phase — real scenarios
- Closing each phase with a test or an open-ended question set measures understanding, not memorization

---

*Prepared by: Denis Ergöçmen, June 2026 — neutralized version for general use*
