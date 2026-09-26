# Appendix B — Glossary

> **Navigation:** [◀ Appendix A — Reference Tables](Appendix_A_Reference_Tables.md) · **Appendix B** · [README ▶](README_en.md)

---

> **How to use it:** next to every term is **the section where it is covered.** The
> definition is there to remind you; don't read a term you don't understand here and move
> on — **go back to the section.**
>
> The terms are in English alphabetical order.

---

## A

**Address bus** *(4.1.1)* — The group of lines that carries which memory address the CPU
wants to reach. Its width determines how much memory can be addressed.

**Associativity** *(2.3.7)* — How many different places a cache line can be placed in.
There are fully associative, direct-mapped and n-way set-associative variants; it affects
conflict misses.

**Asynchronous** — Starting an operation and continuing without waiting for its result. The
basic working mode of DMA *(4.3)* and of NVMe queues *(3.3.2)*.

**await** *(3.4.4)* — In `iostat -x` output, the total **queue wait + service** time of an
I/O request. If it is high, the disk is saturated or there is a latency problem.

**AZ (Availability Zone)** *(5.3.4)* — A physically separate data center inside an AWS
region. Inter-AZ RTT is ~1–2 ms.

## B

**Balloon driver** *(6.4.4)* — A driver placed inside the guest operating system that
"inflates" by allocating memory, thereby forcing the guest to release pages. The
hypervisor's indirect way of reclaiming memory from a guest.

**Bandwidth** *(5.3.1)* — The amount of data carried per unit of time. **It is independent
of latency and it can be increased.**

**Bare metal** *(6.6.4, 7.2.3)* — The instance type in which the entire physical server is
given to the customer with no hypervisor (`.metal`).

**BDP (Bandwidth-Delay Product)** *(5.3.3)* — `Bandwidth × RTT`. The amount of data "in
flight" on the link at any moment. If the TCP window is smaller than this, the link never
fills.

**Bit** *(0.1.2)* — The smallest unit of the binary system: 0 or 1.

**Block (NAND)** *(3.2.2)* — The smallest **erasable** unit in an SSD (~1–4 MB). Writing is
done in pages (4–16 KB) — this asymmetry is the source of write amplification.

**Branch prediction** *(1.4.2)* — Predicting the outcome of conditional branches in advance
so the pipeline stays full. A wrong prediction flushes the pipeline — the basis of Spectre.

**Burst** *(5.2.3, 6.5.3)* — The ability to run above baseline performance for a short
time. It is credit-based and it runs out. This is what "up to X" phrasing describes.

**Bus** *(4.1.1)* — The shared group of lines that carries data between components. It
divides into three: address, data and control bus.

## C

**Cache** *(2.3)* — The small, fast memory between the CPU and RAM. It is built on the bet
that programs will exhibit **locality.**

**Cache coherence** *(2.3.8)* — The protocol (MESI) that keeps the same data consistent
across the caches of multiple cores.

**Cache line** *(2.3.5)* — The cache's unit of transfer, **64 bytes.** Even if you ask for a
single byte, 64 bytes are read. The fundamental unit of access-pattern optimization.

**Cache pollution** *(2.3.4, 7.4.3)* — One workload evicting another workload's hot data
from the cache. **The most insidious mechanism of noisy neighbors** — it appears in no
standard metric.

**CAS latency** *(2.4.2)* — The number of cycles in DRAM between the column address being
given and the data arriving.

**cgroup** *(6.7.2)* — The Linux kernel's resource limiting mechanism: CPU, memory, I/O,
process count. The "how much can you use" half of a container.

**Channel (memory channel)** *(2.4.3)* — An independent data path between the CPU and RAM.
`Bandwidth = number of channels × channel speed`.

**Checkpoint** *(7.6.4)* — A long-running job saving its intermediate state. A
precondition for using Spot instances.

**Chipset** *(4.5)* — The set of chips on the motherboard that connect the non-CPU
components to each other. Integrating the northbridge into the CPU **gave birth to NUMA**
*(4.5.2)*.

**Clock (clock frequency)** *(1.3.1)* — The number of cycles the CPU performs per second
(GHz). On its own it is not an indicator of performance — **it must be multiplied by IPC.**

**Container** *(6.7)* — A process isolated with namespaces and cgroups. **It is not a
lightweight VM** — it shares the host kernel.

**CPU credit** *(6.5.3)* — The unit of performance that accumulates in the t family and is
spent during a burst. When it runs out, you drop to baseline performance.

**CRC** *(5.1.2)* — The error-detection code in an Ethernet frame. A corrupted frame is
silently dropped.

**Cycle** *(1.3.1)* — One tick of the CPU clock. At 3 GHz, 0.33 nanoseconds.

## D

**Data bus** *(4.1.1)* — The group of lines that carries the actual data.

**Dedicated Host** *(7.4.6)* — A specific physical server being reserved for the customer.
It provides placement visibility; it is required for licenses tied to physical cores.

**DIMM** *(2.4.2)* — The physical form of a RAM module. It contains a Rank → Bank → Row →
Column hierarchy.

**DMA (Direct Memory Access)** *(4.3)* — A device reading from and writing to RAM directly,
without going through the CPU. A precondition for high-speed I/O.

**DRAM** *(0.4.4, 2.4.1)* — Memory that holds data as charge in a capacitor and therefore
**requires periodic refresh.** Cheap and dense but slower than SRAM.

**Duplex** *(5.2.2)* — Half duplex: send and receive in turn. Full duplex: **at the same
time** — the modern standard.

## E

**EBS** *(3.3.4, 7.3)* — AWS's persistent block storage service. **It is reached over the
network** — which explains both its latency and its IOPS quota.

**ECC** *(2.4.4)* — Encoding that corrects single-bit errors and detects double-bit errors
in memory. Protection against silent data corruption.

**EFA (Elastic Fabric Adapter)** *(5.4.3)* — AWS's RDMA-like low-latency network interface.
Decisive for multi-node ML training and HPC.

**ENA (Elastic Network Adapter)** *(6.6.4)* — AWS's SR-IOV-based high-performance network
interface ("enhanced networking").

**EPT / NPT** *(6.4.3)* — Intel's/AMD's technology that moves the second address
translation into hardware. It removes the VM exit cost of shadow page tables; it lengthens
the page table walk.

**Ethernet frame** *(5.1.2)* — The data unit of the physical network layer. It contains MAC
addresses, a type field, a payload (max 1500 bytes) and a CRC.

## F

**False sharing** *(2.3.8)* — Different cores writing to **different** variables that sit on
the same cache line. There is no logical sharing, but at the hardware level the line
ping-pongs. Solved with padding.

**Flip-flop** *(0.4.2)* — The basic sequential circuit that stores one bit. The building
block of registers.

**FTL (Flash Translation Layer)** *(3.2.3)* — The SSD layer that maps logical addresses to
physical NAND locations. **The in-device counterpart of virtual memory.**

## G

**Garbage collection (SSD)** *(3.2.3)* — Collecting invalid pages in an SSD and making
blocks reusable again. The source of write amplification.

**gp3** *(7.3.2)* — An AWS EBS SSD type. IOPS and throughput are configured **independently
of disk size** — that is its fundamental advantage over gp2.

**Graviton** *(1.6.4, 7.1.6)* — AWS's ARM64-based processor. **1 vCPU = 1 physical core**
(on x86 it is half a core).

**GRO / LRO** *(5.1.4)* — An offload feature that merges incoming small packets so the
upper layer has less work to do.

## H

**HBM (High Bandwidth Memory)** *(4.2.4)* — The very high bandwidth memory used in GPUs
(~2000–3000 GB/s). ~70× faster than PCIe — the source of the GPU bottleneck.

**HDD** *(3.1)* — A mechanical disk. Its random IOPS (80–120) **has not increased in 30
years** because it is limited by physical head movement.

**Hit rate** *(2.3.6)* — The percentage of requests found in the cache. Going from 90% to
99% makes average access about 4× faster.

**Huge page** *(2.5.4)* — A memory page of 2 MiB or 1 GiB. It widens TLB reach by 512×; **it
is even more critical under virtualization** *(6.4.3)*.

**Hypervisor** *(6.2)* — The software layer that manages virtual machines. Type 1 runs
directly on the hardware, Type 2 runs on top of a host OS.

## I

**Inference** *(7.1.5)* — Running a trained ML model. It wants a different hardware profile
than training does (the `g`, `inf` families).

**Instance store** *(7.1.4)* — Local NVMe attached directly to the physical server. Very
fast but **lost when the instance stops.**

**Interrupt** *(4.4.1)* — The signal a device sends to get the CPU's attention. Its cost
includes a pipeline flush and cache pollution.

**IOMMU (VT-d / AMD-Vi)** *(Phase 4 Q4, 6.6.3)* — An MMU for devices: it translates and
constrains DMA accesses. **It is the precondition for SR-IOV** — without it isolation
disappears entirely.

**IOPS** *(3.4.1)* — The number of I/O operations per second.
`Throughput = IOPS × I/O size`.

**IPC (Instructions Per Cycle)** *(1.3.2)* — The number of instructions completed per cycle.
`Performance = Clock × IPC`. **This is what drops** under cache pollution, and it does not
show up in standard metrics.

**ISA (Instruction Set Architecture)** *(1.6)* — The instruction set the CPU understands
(x86-64, ARM64). A different ISA = a different build is required.

## J

**Jitter** *(5.3.2)* — The fluctuation in latency. It is the signature of queuing delay; it
appears as the link approaches saturation.

**Jumbo frame** *(5.1.2)* — An Ethernet frame with an MTU of 9000 bytes. It improves
efficiency and **packet count**; **every** device along the path must support it.

## K

**KSM (Kernel Same-page Merging)** *(6.4.4)* — Merging memory pages with identical content
onto a single physical page. It saves memory and carries a side-channel risk.

**KVM** *(6.2.3)* — The module that turns the Linux kernel into a Type 1 hypervisor. The
basis of the AWS Nitro hypervisor.

## L

**L1 / L2 / L3** *(2.3.2–2.3.4)* — The cache levels. L1 and L2 are private to the core,
**L3 is shared by all cores** — this is where noisy neighbors are born.

**Latency** *(5.3.1)* — The time it takes a single request to complete. **It is independent
of bandwidth and it is limited by the speed of light.**

**Little's Law** *(Answer 3.3, Answer 5.1)* — `Wait time = Queue length ÷ Service rate`. It
is used as an `await ≈ aqu-sz ÷ IOPS` consistency check on `iostat` output.

**Locality** *(2.1.3)* — The pattern in which programs access memory. **Temporal** (what was
accessed recently will be accessed again) and **spatial** (nearby addresses are accessed
together). This is the bet that cache is built on.

**Logic gate** *(0.3)* — A circuit that performs a basic boolean operation such as AND, OR
or NOT. The building block of all computation.

## M

**MAC address** *(5.1.2)* — The 48-bit hardware address of a network interface. Used for
addressing on the local network.

**Memory wall** *(2.1.2)* — The performance gap that arises because CPU speed grew much
faster than memory speed. The reason the cache hierarchy exists.

**MESI** *(2.3.8)* — The cache coherence protocol: Modified, Exclusive, Shared, Invalid.

**MMU (Memory Management Unit)** *(2.5.2)* — The hardware unit that translates virtual
addresses into physical addresses.

**MSI-X** *(4.4.3)* — A message-based interrupt mechanism. It allows interrupts to be
distributed across different cores — the hardware basis of RSS.

**MTU (Maximum Transmission Unit)** *(5.1.2)* — The maximum payload a frame can carry. The
standard is 1500 bytes. A mismatch gives the symptom "some connections work, some hang".

## N

**Namespace** *(6.7.2)* — The Linux kernel's visibility isolation mechanism (PID, network,
mount, hostname). The "what can you see" half of a container.

**NAND** *(3.2.1)* — The flash memory technology used in SSDs. It stores data by trapping
electrons in a floating gate.

**NAPI** *(4.4.2, 5.1.4)* — The hybrid network processing mechanism that uses interrupts at
low traffic and polling at high traffic. It prevents interrupt storms.

**NIC** *(5.1)* — The network interface card. It translates between bits in memory and
signals on the wire.

**Nitro** *(4.3.3, 6.6.4, 7.2)* — AWS's architecture that moves hypervisor functions onto
separate physical cards. It brought the virtualization tax below 1% and moved the security
boundary **from software to hardware.**

**Noisy neighbor** *(6.1.2, 7.4)* — Another tenant sharing the same physical hardware
affecting your performance. It has three mechanisms: CPU time (measurable), **L3 cache**
and **memory bandwidth** (not measurable).

**NUMA (Non-Uniform Memory Access)** *(2.7)* — The architecture in which each CPU socket has
its own local memory. Remote node access is 1.5–2× slower. **It is the consequence of
integrating the memory controller into the CPU** *(4.5.2)*.

**NVMe** *(3.3.2)* — The protocol designed for SSDs. **65,535 queues** (SATA has 1), each
65,536 deep.

## O

**OOM Killer** *(2.6.3)* — The Linux kernel's mechanism for terminating a process when
memory runs out. It also kicks in in containers when the cgroup limit is exceeded.

**Overcommit** *(6.4.4)* — Committing more than the physical resource. **AWS EC2 does not
overcommit memory** — for the sake of predictability.

## P

**Packet loss** *(5.3.2)* — Packets being dropped when a queue fills. TCP uses this as a
congestion signal.

**Page** *(2.5.2)* — The unit of transfer and mapping in virtual memory. The standard is
4 KiB.

**Page fault** *(2.5.5)* — The accessed page not being in physical memory. Minor (in memory
but not mapped), major (must be read from disk), invalid (not valid — segfault).

**Paravirtualization** *(6.3.2, 6.6.2)* — The guest OS knowing that it is virtual and using
an efficient protocol with the hypervisor. virtio is the standard for this.

**PCIe** *(4.2)* — The modern point-to-point expansion bus. It scales with lanes (x1–x16)
and generations (Gen3–Gen6).

**Pipeline** *(1.4)* — Parallelizing instruction processing by splitting it into stages.
Hazards (data, control, structural) reduce its efficiency.

**Polling** *(4.4.2)* — Querying a device regularly instead of waiting for an interrupt.
More efficient than interrupts at high traffic.

**pps (packets per second)** *(4.4.2)* — The number of packets processed per second. On
small-packet workloads this limit fills before bandwidth does.

**Prefetching** *(2.3.9)* — The CPU pulling data that will be needed into the cache ahead of
time. It works on regular access patterns; it does not work on pointer chasing.

**Propagation delay** *(5.3.2)* — The time it takes a signal to cross a distance. **Limited
by the speed of light — no technology can reduce it.**

## Q

**Queue curve** *(3.4.4)* — Latency rising **exponentially** as utilization rises. 95%
utilization means about 8× the latency of 50%. **It recurs in three separate layers in this
map.**

**Queue depth** *(3.2.5, 3.3.2)* — The number of requests sent to a device at the same time.
On NVMe, the difference between QD=1 and QD=32 is a matter of multiples.

## R

**RAID** *(3.5)* — Using several disks as a single logical unit. It is usually unnecessary
on top of EBS (EBS is already replicated).

**RDMA** *(5.4.2)* — Accessing a remote machine's memory directly, without its CPU getting
involved. DMA extended over the network.

**Refresh** *(2.4.1)* — Periodically renewing the charge of DRAM cells. The fundamental
property that separates DRAM from SRAM.

**Register** *(2.2)* — The fastest storage unit inside the CPU. Zero-cycle latency.

**Right-sizing** *(7.6.2)* — Adjusting instance size to actual usage. Target: CPU average
40–60%.

**Ring buffer** *(5.1.3)* — The circular array of descriptors between the NIC and the
kernel. If it fills, packets are **silently dropped** (`rx_dropped`).

**Ring level** *(6.3.1)* — The privilege layer on x86. Ring 0 is the kernel, ring 3 is
userspace.

**RSS (Receive Side Scaling)** *(5.1.4)* — The NIC distributing incoming packets across
different queues and cores according to a header hash. **A hash is used because the same
connection must stay on the same core** (ordering + cache locality).

**RTT (Round-Trip Time)** *(5.3.4)* — The round-trip time. Same AZ ~0.4 ms, intercontinental
~100–200 ms.

## S

**Shadow page table** *(6.4.2)* — Before EPT, the hypervisor maintaining a combined table
from guest virtual address to real physical address. It requires a VM exit on every page
table change.

**SMT / Hyper-Threading** *(1.5.2)* — One physical core appearing as two threads. **On x86
instances, 1 vCPU = 1 SMT thread = half a core.**

**Spot instance** *(7.6.4)* — Capacity at a 70–90% discount that can be interrupted with 2
minutes' notice. Suitable for stateless and checkpointed workloads.

**SRAM** *(0.4.4)* — Fast memory that holds data with transistors and needs no refresh. Used
in caches; expensive and not dense.

**SR-IOV** *(6.6.3)* — A physical device presenting itself as multiple virtual functions,
each assigned directly to a VM. **It eliminates VM exits**; it requires an IOMMU.

**Steal time** *(6.5.2)* — The percentage of time a vCPU was ready to run but could not find
a physical core. **It has two causes:** host contention and t-family credit exhaustion.

**Swap** *(2.6)* — Moving memory pages to disk. Usually off in the cloud — the principle of
**fail fast, don't die slowly.**

**Switching** *(5.2.1)* — Directing data only to its destination instead of broadcasting it
onto a shared medium. The shared idea behind the hub→switch and shared bus→PCIe
transitions.

## T

**TCP window** *(5.3.3)* — The amount of data that can be sent without waiting for an
acknowledgment. If it is smaller than the BDP, the link never fills.

**Thrashing** *(2.6.2)* — The system spending more time swapping pages than doing work. It
is a positive feedback loop — the system does not slow down gradually, it falls **off a
cliff.**

**Throughput** *(3.4.1)* — The amount of data carried per unit of time. `IOPS × I/O size`.

**TLB (Translation Lookaside Buffer)** *(2.5.3)* — The cache of page table translations. Its
reach is limited by `number of entries × page size` — the rationale for huge pages.

**Training** *(7.1.5)* — Teaching an ML model from data. High GPU and inter-node network
requirements (the `p`, `trn` families).

**TRIM** *(3.2.3)* — The operating system telling the SSD "these pages are no longer valid".
It improves garbage collection efficiency.

**TSO / GSO** *(5.1.4)* — The NIC splitting a large block of data into packets. It lowers
the CPU cost per packet.

## V

**vCPU** *(1.5.3, 6.5.1)* — A virtual processor. On x86, 1 SMT thread; on Graviton, 1
physical core. **And in every case it is a thread scheduled onto a physical core** — it is
not permanently yours.

**virtio** *(6.6.2)* — The paravirtualized I/O standard. It lowers the number of VM exits by
batching notifications through a shared ring buffer.

**Virtual memory** *(2.5)* — The mechanism that gives every process its own contiguous
address space. It solves three problems: isolation, fragmentation, and an address space
larger than physical RAM.

**VM exit** *(6.3.4)* — Control passing to the hypervisor when the guest attempts a
privileged operation. **The fundamental cost unit of virtualization** (1,000–5,000 cycles).
All optimization aims at reducing it.

**VMX root / non-root** *(6.3.3)* — The mode distinction introduced by VT-x, orthogonal to
the ring levels. The guest runs in ring 0 but in non-root mode.

**VT-x / AMD-V** *(6.3.3)* — Hardware-assisted virtualization technology. It lets the guest
OS run unmodified.

## W

**Waiting bound** *(7.5.7)* — The application being slow while no hardware resource is
saturated. Locks, remote calls, pool exhaustion, GC pauses. **Buying hardware fixes nothing
in this situation.**

**Wear leveling** *(3.2.3)* — Extending an SSD's life by balancing writes across blocks.

**Write amplification** *(3.2.2)* — More data being written to NAND than the application
wrote. It arises from the page/block asymmetry.

**Write penalty** *(3.5.2)* — In RAID 5, a single logical write requiring 4 physical
operations (read-read-write-write).

---

## Extra: easily confused pairs

| Pair of terms | The difference | Section |
|---|---|---|
| **Bandwidth vs Latency** | One can be increased, the other is limited by the speed of light | 5.3.1 |
| **IOPS vs Throughput** | `Throughput = IOPS × I/O size` | 3.4.1 |
| **Core vs Thread vs vCPU** | Physical / SMT / the slice allocated to you | 1.5 |
| **Steal time vs Cache pollution** | One is measurable, the other is invisible | 7.4 |
| **Virtual memory vs Swap** | A mechanism vs one use of it | 2.5, 2.6 |
| **Container vs VM** | A shared kernel vs a separate kernel | 6.7 |
| **Instance store vs EBS** | PCIe direct vs over the network | 7.1.4 |
| **gp3 vs io2** | General purpose vs sub-millisecond latency | 7.3 |
| **RSS vs RPS** | Distribution in hardware vs in software | 5.1.4, Phase 5 Q6 |
| **DMA vs RDMA** | Skipping the local CPU vs skipping the remote CPU | 4.3, 5.4.2 |
| **Emulation vs virtio vs SR-IOV** | The three generations of I/O virtualization | 6.6 |
| **Overcommit vs Right-sizing** | The provider overselling vs you buying correctly | 6.4.4, 7.6.2 |
| **Type 1 vs Type 2 hypervisor** | On the hardware vs on a host OS | 6.2 |
| **Minor vs Major page fault** | In memory but not mapped vs must be read from disk | 2.5.5 |
| **SRAM vs DRAM** | Transistors, no refresh vs capacitors, refresh required | 0.4.4 |

---

*End of Appendix B.* → **[Appendix A — Reference Tables](Appendix_A_Reference_Tables.md)** · **[README](README_en.md)**

---

> **Navigation:** [◀ Appendix A — Reference Tables](Appendix_A_Reference_Tables.md) · **Appendix B** · [README ▶](README_en.md)
