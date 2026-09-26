# Checkpoint Quiz 2 — Phases 4–7: From the Bus to the Cloud Decision

> **Navigation:** [◀ Phase 7 — Cloud Connection](Phase_7_Cloud_Connection.md) · **Checkpoint Quiz 2** · [Back to the start of the map ▶](README_en.md)

---

## What does this quiz measure?

Phases 0–3 built the machine's **inside**: transistor, CPU, memory, disk. Phases 4–7 connect that machine to the
**outside** and then **rent it out**: the buses and DMA that carry data (Phase 4), the network card and the wire
(Phase 5), the hypervisor that slices one server into many (Phase 6), and finally the cloud decisions where all of
it turns into an instance type and a bill (Phase 7). The "Test yourself" tests of each phase probe one phase. This
quiz measures something **different**: *"When an application is slow in the cloud, can you walk the whole path —
CPU, memory, disk, bus, network, hypervisor — and name the layer that is actually to blame?"*

Notice how often the same three ideas returned in these phases: **the most expensive component is only as fast as
the path that feeds it** (PCIe, the GPU, the EBS ceiling); **the queue curve** (ring buffer, link, disk, capacity
planning); **the virtualization tax is the number of exits × their cost** (VM exit, SR-IOV, Nitro). This quiz
checks that you see those three ideas, not just the chapters.

**How to study:**

- Solve with pen and paper; do not read the answer key before writing your own answer.
- The answer key tells you **at which intersection of phases** each question stands — a missed question points
  not to a phase but to the **bridge** between phases.
- There are 21 questions: Part A (connection reasoning, 1–9), Part B (scenario: "one API, one quarter of
  incidents", 10–15), Part C (reading output and calculating, 16–21).
- The numbers you need are in Appendix A and in the phases. Target time: ~1 hour. Time is not the point; being
  able to justify each answer in one sentence is.

> **🤔 Before you start:** Complete this sentence in your own words: *"The application is slow, but no single
> resource is full; the first thing I measure is ___, because ___."* Then write a second sentence: *"Buying a more
> expensive ___ does not help when the path that feeds it is ___."* These two sentences are the thread through all
> 21 questions.

---

# Part A — Connection reasoning (1–9)

Each question asks you to combine the knowledge of at least two phases. Give a short but justified answer.

**1.** An NVMe SSD is rated for 7,000 MB/s sequential reads (Phase 3.3.3) and is designed for a Gen4 x4 link. What
is the highest speed you can expect if it is plugged into (a) a Gen4 x4 slot, (b) a Gen3 x4 slot, (c) an M.2 slot
that only gives it a Gen4 x2 link? Which sentence from Phase 4.2.2 describes what happened to the disk in (b) and
(c)?

**2.** During a large file copy `top` shows `wa` high and `us` low, yet the machine is not "busy" in the CPU sense.
Explain, using DMA (Phase 4.3) and the interrupt (Phase 4.4), what the CPU is doing during the milliseconds the
disk takes — and what would have happened to `us`/`sy` if the system had no DMA (programmed I/O).

**3.** A server CPU provides 64 PCIe Gen4 lanes. You want 2 GPUs (x16 each), 4 NVMe drives (x4 each) and one
100 GbE NIC. (a) How many lanes remain if the NIC gets a Gen4 x8 link? (b) Could you add a **second** 100 GbE NIC
on x8? (c) Why is Gen4 x8 enough for a 100 GbE NIC but Gen3 x8 is not?

**4.** A 25 Gbps NIC receives 1500-byte packets at line rate. If every packet raised its own interrupt at ~2 μs
of CPU work, how many cores' worth of time would the interrupts alone need? Name the two mechanisms — one from
Phase 4.4 and one from Phase 5.1 — that make the system survive this, and say what each one does.

**5.** The Frankfurt application server calls a database in Virginia, and page load takes 4 seconds. Someone
proposes upgrading the link from 1 Gbps to 10 Gbps. Using the four latency components of Phase 5.3.2, show why that
changes almost nothing for a 1500-byte packet — and give the arithmetic that explains where the 4 seconds come
from.

**6.** The queue curve appeared on the disk (`await`, Phase 3.4.4), inside the NIC (the ring buffer, Phase 5.1.3)
and on the link (Phase 5.3.2). A team sees `rx_dropped` rising because the CPU cannot drain the ring fast enough,
and raises the ring from 512 to 4096. Loss falls but latency rises. Explain why, and state what kind of change
actually helps when the arrival rate exceeds the service rate.

**7.** A guest VM receives 500,000 network packets per second with emulated I/O; each packet costs one VM exit of
2,500 cycles. The CPU runs at 3 GHz. What fraction of **one core** is spent on VM exits alone? Which two terms make
up the "virtualization tax", and what did SR-IOV/Nitro change in that product?

**8.** Bare-metal address translation needs up to 4 memory accesses on a TLB miss; under EPT/NPT it can need 24.
Explain why this makes the TLB more critical in a VM, and use the TLB-reach arithmetic of Phase 2.5.4 (1,536
entries; 4 KiB vs 2 MiB pages) to explain why huge pages are the standard recommendation for database VMs.

**9.** Phase 2.6 taught that a memory shortage does not degrade performance gradually — it falls off a cliff
(thrashing). Phase 6.4.4 says AWS does **not** overcommit memory. Connect the two: why is that a design choice
rather than a missing feature, and what habit brought over from enterprise VMware environments becomes expensive
on AWS?

---

# Part B — Scenario: "One API, one quarter of incidents" (10–15)

> **Incident:** A team runs an order API on AWS. Over one quarter it hits six different problems, in the order
> you will meet them below. Each question comes with its own evidence. The API answers within a 100 ms budget, is
> written in Go, and started life on a `t3.large` (2 vCPU, 8 GiB).

**10.** The API passed a 5-minute load test with p99 = 40 ms. In production, after about 40 minutes of peak
traffic, p99 jumps to 400 ms. `top` on the instance at that moment:

```
%Cpu(s): 19.8 us,  4.1 sy,  0.0 ni,  3.0 id,  0.3 wa,  0.0 hi,  0.8 si, 72.0 st
```

Which number is the diagnosis, how do you tell "noisy neighbor" from "credits exhausted", what will restarting the
instance do in each case, and why did the 5-minute load test lie?

**11.** Use the credit model of Phase 6.5.3 (the `t3.medium` numbers: 2 vCPU, 20% baseline, 24 credits earned per
hour, 2 credits spent per minute at full speed). (a) Which average utilization can the instance sustain forever?
(b) The instance starts a peak with a bank of 60 credits and runs both vCPUs at 100%: after how many minutes does
the bank hit zero? (c) How many credits does a 5-minute load test consume, and what does that say about the test?

**12.** The team decides to leave the t family. Two candidates: `c6g.xlarge` and `c6i.xlarge`. Decode both names
(Phase 7.1.0). How many **physical cores** does each give, and why is "both are 4 vCPU" a trap? Name three items
from the migration checklist that must be verified before choosing the Graviton one.

**13.** After the move, the team deploys the API across multiple AZs for resilience. Each request now makes 12
**sequential** calls to a service in another AZ. Using the RTT table of Phase 5.3.4, how much of the 100 ms budget
does the network alone consume, compared with the same-AZ case? If the chain later grows to 20 calls, what
happens? What is the right way to weigh this decision?

**14.** A nightly job pulls a 10 GiB file from a partner's server 100 ms away. The link is 10 Gbps, but the
transfer runs at a few hundred KB/s. The host uses a single TCP connection with a 64 KiB window. (a) What is the
BDP of this path? (b) What throughput does the 64 KiB window allow, and how long does the 10 GiB transfer take?
(c) Give three fixes and say which one `aws s3 cp` already uses.

**15.** On the API gateway tier, `top` shows about 40% CPU on average, yet there is packet loss and high latency.
Evidence:

```
$ mpstat -P ALL 1                      (one sample)
CPU   %usr  %sys  %soft  %idle
all    9.7   5.8   23.7   60.8
  0    3.0   5.0   92.0    0.0
  1   12.0   6.0    1.0   81.0
  2   12.0   6.0    1.0   81.0
  3   12.0   6.0    1.0   81.0

$ cat /proc/interrupts | grep eth0
 24:   98234511          0          0          0   PCI-MSI  eth0-rx-0
 25:   97120344          0          0          0   PCI-MSI  eth0-rx-1
 26:   96877210          0          0          0   PCI-MSI  eth0-rx-2
 27:   97456002          0          0          0   PCI-MSI  eth0-rx-3

$ ethtool -g eth0      →  RX current 512 / max 4096
$ ethtool -S eth0 | grep drop  →  rx_dropped: 1,204,331 (rising)
```

What is happening, why does the average hide it, what is the fix and in what order would you act, and why is
raising the ring buffer the **last** thing to try?

---

# Part C — Reading output and calculating (16–21)

**16.** An ML training job shows GPU utilization around 33% with GPU memory nearly full. Per training step the GPU
computes for 0.5 s; the data pipeline (disk → CPU preprocessing → PCIe) needs 1.0 s to deliver the next batch and
nothing overlaps. (a) Verify the 33% figure. (b) With prefetching so that loading overlaps computing, what is the
step time and the utilization? (c) How many times faster must the pipeline become for the GPU to be the
bottleneck? (d) Would a more expensive GPU help? (e) Compare the GPU's own memory bandwidth (~2,500 GB/s) with
Gen4 x16 (Phase 4.2.2): how much narrower is the feed line?

**17.** A 100 GbE NIC (Phase 5.2.3) is fitted into a slot that trains as **PCIe Gen3 x8**. (a) What is the NIC's
line rate in GB/s and what is the slot's ceiling? (b) What throughput can the system actually reach, in Gbps? (c)
Which slot configurations would fix it (Gen4 x8, Gen3 x16, Gen4 x4)?

**18.** Three servers report steal time:

| Server | Instance | `st` | Extra evidence |
|---|---|---|---|
| A | `m6i.large` | 1.5% | — |
| B | `m6i.large` | 14% | Other tenants on the host; `CPUCreditBalance` not applicable |
| C | `t3.medium` | 61% | `CPUCreditBalance` = 0 |

Classify each using the interpretation table of Phase 6.5.2 and give the action. Why is C's remedy different from
B's, and why does restarting C do nothing?

**19.** An `io2` volume (64,000 IOPS) is attached to an `m7i.large` whose EBS limit is about 10,000 IOPS.
`iostat -x`:

```
Device    r/s     w/s    await  aqu-sz
nvme1n1   8400    1550    6.0     60
```

(a) Which ceiling is the system at? (b) Check the numbers with Little's law. (c) What is wasted, and what are the
two sensible fixes? Which command shows the instance's limit?

**20.** Peak traffic is 3,800 requests/s. One instance handles 1,000 requests/s at 100% CPU. Compare two capacity
plans: run each instance at ~95% at peak versus ~60% at peak. (a) How many instances does each plan need? (b) One
instance fails at peak: what utilization do the survivors face in each plan? (c) Use the queue-curve figures to
explain what a 95% target buys and what it risks, and why over-provisioning is "visible" but under-provisioning
is not.

**21.** Classify each machine into one of the five bottleneck classes of Phase 7.5 (CPU, memory, I/O, network,
waiting) and name the first command of the diagnostic procedure that would show it:

- (a) `top`: 92% `us`; `perf stat` shows IPC 1.8.
- (b) `top`: 6% `us`, 90% `id`, `wa` 0, `st` 0; `iostat` idle; `free` healthy; yet p99 is 2 s and the threads sit
  blocked on a database connection pool.
- (c) `vmstat 1`: `si` 300, `so` 400; `free -h`: `available` 90 Mi.
- (d) `iostat -x`: r/s + w/s at the volume's limit; `await` and `aqu-sz` both high.
- (e) `sar -n DEV 1`: 9.4 Gbps on a 10 Gbps link; `rx_dropped` rising.

---

## Answer key

**1.** (a) Gen4 x4 = 4 × 2.0 = **8 GB/s** ceiling → the drive reaches its rated **~7,000 MB/s**. (b) Gen3 x4 = 4 × 1.0
= **4 GB/s** → about **4,000 MB/s**. (c) Gen4 x2 = 2 × 2.0 = **4 GB/s** → also about **4,000 MB/s**. The workbook's
sentence: *"the disk doesn't get slower, the road gets narrower."* The cause of "I bought a fast SSD and don't get
the speed" is usually the generation or lane count of the slot, not the disk. · *Phase 3.3.3 × Phase 4.2.2*

**2.** With DMA the CPU's involvement is two brief moments: it gives the controller the instruction ("read 1 MB to
this address, notify me when done") and it handles the completion **interrupt**. In between, the disk moves data
into RAM by itself and the CPU runs other threads; the waiting process sleeps. If there is nothing else to run,
the time is accounted as **`wa`** (waiting for I/O), not `us`/`sy`. Without DMA (programmed I/O) the CPU would copy
every word itself — 262,144 transfers for 1 MB — showing up as busy time and locking up the server. ·
*Phase 3.4.3 × Phase 4.3.2*

**3.** (a) 2 × 16 + 4 × 4 + 8 = 32 + 16 + 8 = 56 lanes used → **8 lanes remain**. (b) Yes — a second NIC on x8
uses exactly the remaining 8 lanes, and then the budget is **fully spent** (nothing left for any other device). (c)
100 GbE = 12.5 GB/s (≈ 11.9 GB/s usable after ~95% protocol efficiency). Gen4 x8 = 8 × 2.0 = **16 GB/s** ≥ 12.5 →
fits; Gen3 x8 = 8 × 1.0 = **8 GB/s** < 12.5 → the slot, not the NIC, becomes the limit. Lane count is a
first-class feature that separates server CPUs from desktop CPUs. · *Phase 4.2.3 × Phase 5.2.3*

**4.** 25 Gbps ÷ (1500 × 8 bits) ≈ **2.08 million packets/s**; × 2 μs = **~4.2 seconds of CPU per second** — more than
four cores just to take interrupts. Survival mechanisms: **NAPI** (Phase 4.4.2 / 5.1.4) — at low traffic use
interrupts (low latency, CPU can idle), at high traffic **disable interrupts and poll** in batches (budget
~64–300 packets per round), re-enabling interrupts when the queue empties; and **RSS** (Phase 5.1.4) — the NIC hashes
the connection tuple and spreads packets over several RX queues, each bound to a different core, so the work is
parallel and one connection always stays on the same core (ordering and cache locality). · *Phase 4.4.2 × Phase 5.1.4*

**5.** The four components: propagation + transmission + processing + queuing. Transmission of 1500 bytes:
1,500 × 8 ÷ 1 Gbps = **12 μs**; on 10 Gbps = **1.2 μs**. Intercontinental propagation is ~50,000 μs one way, so the
total moves from ~50,012 μs to ~50,001 μs — about 0.02%: **nothing**. The 4 seconds come from **round trips**: with an
RTT of ~100 ms (Frankfurt–Virginia is in the intercontinental ~100–200 ms class), a page that does ~40 sequential
queries pays 40 × 100 ms = **4 s** of pure propagation. Bandwidth is the wrong knob; reduce the number of round
trips (batch queries, cache, move the data closer). · *Phase 5.3.2 × Phase 5.3.4*

**6.** The ring drains at the CPU's rate; arrivals exceed it, so the queue is always full. A deeper ring just makes
packets **wait longer in a longer queue** (bufferbloat) — fewer are dropped, each one is later. It is the same queue
curve as `await` on the disk. What helps is changing the **service rate** (more cores draining it via RSS/IRQ
spreading, less work per packet, a faster CPU/NIC) or reducing the **arrival rate**; enlarging the buffer helps only
against short bursts, not a persistently overloaded consumer. · *Phase 3.4.4 × Phase 5.1.3 × Phase 5.3.2*

**7.** 500,000 × 2,500 = 1.25 × 10⁹ cycles/s; ÷ 3 × 10⁹ = **≈ 42% of one core**, before the guest does any work. The
tax is **the number of VM exits × the cost of one exit** (1,000–5,000 cycles; a pipeline flush plus cache/TLB
pollution — the same mechanism as the branch-misprediction penalty). SR-IOV (a virtual function assigned straight
to the guest) and Nitro (I/O on dedicated cards) remove most exits from the I/O path, taking the tax from 30–50%
(binary translation) and 5–15% (VT-x + EPT) to **under 1%**. · *Phase 1.4.2 × Phase 6.3.4 × Phase 6.6.4*

**8.** The MMU walks two tables (guest → GPA, EPT → HPA), so a full walk can need **24** accesses instead of 4 —
a TLB miss is up to 6× more expensive. Everything that lowers the miss rate therefore pays more. TLB reach: 1,536 ×
4 KiB = **6 MiB**; 1,536 × 2 MiB = **3 GiB** (512× larger). Huge pages cut TLB misses and shorten the levels of both
walks — that is why huge pages are the standard recommendation for database VMs. · *Phase 2.5.3–2.5.4 × Phase 6.4.3*

**9.** Overcommit means unpredictable performance, and predictability is the basis of the cloud contract: if a guest
were squeezed at the hypervisor level (balloon, KSM or hypervisor swap), the guest would hit the thrashing cliff
without knowing why. So AWS gives 32 GB when it says 32 GB — really allocated and really billed. The expensive
habit: **"commit RAM generously, there is overcommit anyway"** from enterprise VMware — on AWS every GB is paid for,
so right-size memory instead of padding it. · *Phase 2.6.2 × Phase 6.4.4*

**10.** The diagnosis is **`st` = 72%** (above the 25%+ "unacceptable" band): the vCPUs are ready to run but waiting for a
physical core; `us` is only ~20%. Distinguish the two causes: CloudWatch **`CPUCreditBalance`** — at 0 on a t-family
instance, AWS is throttling you **deliberately** toward the baseline; a healthy balance with high `st` points to an
overcommitted host. Restarting **does nothing** for credit exhaustion (only leaving the t family, or paying for
`unlimited`, helps); for a noisy host a restart may land on another host. The 5-minute test lied because credits and
burst quotas do not run out in 5 minutes — production runs out around minute 20–40; load-test **at least 30
minutes** and never on a t instance. · *Phase 6.5.2–6.5.3 × Phase 7.6.1*

**11.** (a) The baseline: **20%** of the instance — 0.4 vCPU-minutes per minute = 24 credits/hour earned. (b) Full
speed spends 2 credits/min while 0.4/min is still earned: net drain 1.6/min → 60 ÷ 1.6 = **37.5 minutes**. (c) A
5-minute test consumes ~10 credits (net ~8) — it never comes close to the wall; the instance looks "fast" exactly
because the test is shorter than the bank. Decision rule: sustained load (production API) → m/c/r; t only for
intermittent, low-average work. · *Phase 6.5.3*

**12.** `c6g.xlarge`: **c** = compute optimized, **6** = 6th generation, **g** = Graviton (ARM), **xlarge** = 4 vCPU.
`c6i.xlarge`: same but **i** = Intel. On Graviton **1 vCPU = 1 physical core** → 4 cores; on x86 **1 vCPU = 1 SMT
thread (half a core)** → **2 physical cores**. "Same vCPU count" hides up to a 2× difference in physical compute
(one reason Graviton's price/performance is 20–40% better). Verify before migrating (any three): the language
supports ARM64 (Go ✅), native dependencies have ARM64 builds, Docker images are multi-arch, third-party agents
(APM/log/security) support ARM64 — *the most common blocker* — and CI/CD can produce ARM64 builds. ·
*Phase 1.5.3 × Phase 7.1.6*

**13.** Multi-AZ RTT ≈ 1–2 ms (use 1.5): 12 × 1.5 = **18 ms** of network alone, i.e. 18% of the budget; same AZ
(0.3–0.5 ms, use 0.4): 12 × 0.4 = **4.8 ms**. With 20 calls it is **30 ms** — a third of the budget, before any
compute. Resilience is a real gain and the latency budget is real too: weigh them **knowingly, not by reflex** —
reduce the number of sequential cross-AZ calls (batch or parallelize), keep chatty services in the same AZ, and
spread only what needs resilience across AZs. · *Phase 5.3.4 × Phase 7.6*

**14.** (a) BDP = bandwidth × RTT = 1.25 GB/s × 0.1 s = **125 MB** — the window TCP would need to fill the link.
(b) 64 KiB ÷ 0.1 s = 655,360 B/s ≈ **640 KiB/s ≈ 5 Mbps** — about **0.05%** of the link. 10 GiB ÷ 640 KiB/s =
10,485,760 KiB ÷ 640 = 16,384 s ≈ **4.5 hours**. (c) Fixes: TCP window scaling (on by default on modern systems) and
larger `net.ipv4.tcp_rmem`/`tcp_wmem`; **parallel streams** (10 streams = 10× the window; `aws s3 cp` and download
accelerators do exactly this); a congestion control better suited to long distances (**BBR**). The problem is the
window, not the link. · *Phase 5.3.3*

**15.** **Interrupt imbalance.** All four RX queue interrupts land on **CPU0**: core 0 is at 92% `%soft` with 0% idle,
while cores 1–3 sit at 81% idle. The average (`top` says ~39% busy) hides the drowning core, and the ring drops
because that one core cannot drain it. Order of action: (1) confirm per core with `mpstat -P ALL 1`; (2) look at
`/proc/interrupts` — all counts in a single column; (3) **spread the IRQs** across cores (MSI-X queues bound to
different CPUs via `smp_affinity`/`irqbalance`) and check RSS with `ethtool -l eth0`; (4) only then consider the
ring size. Raising the ring buffer only deepens the queue in front of an overloaded consumer (more latency, the same
loss under load) — the queue curve again. · *Phase 4.4.3 × Phase 5.1.3–5.1.4*

**16.** (a) Utilization = 0.5 ÷ (0.5 + 1.0) = **33%** ✓. (b) With overlap the step takes max(0.5, 1.0) = **1.0 s**;
utilization = 0.5 ÷ 1.0 = **50%**. (c) The pipeline must deliver a batch in ≤ 0.5 s — **2× faster** — before the GPU
becomes the limit. (d) **No** — a more expensive GPU would just wait more; more workers in the data loader,
prefetching, faster storage/format, preprocessing moved to the GPU, a bigger batch or data kept in GPU memory are the
fixes. (e) ~2,500 GB/s ÷ 32 GB/s ≈ **78×** narrower (the workbook quotes ~70×). Watch `nvidia-smi` utilization,
per-core `top`/`mpstat`, disk `iostat`. · *Phase 3.4.4 × Phase 4.2.4*

**17.** (a) 100 Gbps ÷ 8 = **12.5 GB/s** (≈ 11.9 GB/s usable); Gen3 x8 = 8 × 1.0 = **8 GB/s**. (b) The slot caps the
system at **8 GB/s ≈ 64 Gbps** — about a third of the NIC is wasted. (c) **Gen4 x8** (16 GB/s ✓), **Gen3 x16** (16
GB/s ✓); Gen4 x4 = 8 GB/s ✗. The capacity of the slot is as decisive as the NIC's own speed. ·
*Phase 4.2.2 × Phase 5.2.3*

**18.** **A: 1.5% → normal, do nothing** (0–2%). **B: 14% → serious** (10–25%): the host is overcommitted → restart
(the instance may land on another host), a larger or dedicated instance. **C: 61% → unacceptable** (25%+), and
`CPUCreditBalance` = 0 shows it is **AWS throttling on purpose** — no neighbor involved — so restarting **does
nothing**; the only fix is to **change the family** (t → m/c) or pay for `unlimited`. The cause decides the
remedy, which is why steal time must always be paired with the credit metric on t instances. ·
*Phase 6.5.2–6.5.3*

**19.** (a) r/s + w/s = 8,400 + 1,550 = **9,950 IOPS** — at the **instance's EBS ceiling** (~10,000), not the volume's
(64,000). (b) 60 ÷ 9,950 ≈ 0.00603 s ≈ **6.0 ms** = `await` ✓ — requests spend their time waiting in the queue.
(c) The expensive io2 capability is **wasted** (64,000 paid for, 10,000 usable). Fixes: a **larger instance** whose
EBS bandwidth/IOPS ceiling is higher, or a **cheaper volume** (e.g. gp3 provisioned to ~10,000 IOPS) that matches
what the instance can use. `aws ec2 describe-instance-types --instance-types … --query 'InstanceTypes[0].EbsInfo'`
shows the instance's limit; buy disk performance by checking **both ceilings**. · *Phase 3.4.5 × Phase 7.3.5*

**20.** (a) 95% plan: 3,800 ÷ 950 = **4 instances**. 60% plan: 3,800 ÷ 600 = 6.33 → **7 instances** (1.75× the
fleet). (b) 95% plan: 3 survivors must carry 3,800 → 1,267 req/s each = **127% — overload, cascade**. 60% plan: 6
survivors → 633 req/s each = **63%** — fine. (c) A 95% target buys a lower bill on paper but sits on the steep part
of the queue curve: **95% utilization ≈ 8× the latency of 50%**, and any failure or spike falls off the cliff;
targets of CPU average 40–60% and p99 < 80% leave the headroom. Over-provisioning is **visible on the bill** and
easy to correct; under-provisioning is **invisible** — it is paid in lost customers and incidents — so a reasonable
amount of over-provisioning is rational, "reasonable" defined by measurement, with auto scaling on top. ·
*Phase 3.4.4 × Phase 7.6.2–7.6.3*

**21.** (a) **CPU bound** (IPC high, `us` high) — Step 1 `top`/`mpstat`. (b) **Waiting bound** — everything is idle but the
application is slow; blocked on a lock/pool — Step 5 (application profiling), reached by elimination. (c) **Memory
bound** with **swap** — `si`/`so` > 0 is a disaster to handle immediately — Step 2 `free -h`/`vmstat 1`. (d) **I/O
bound** — `await` + `aqu-sz` high, at the IOPS ceiling — Step 3 `iostat -x 1`. (e) **Network bound** — bandwidth at the
ceiling, drops rising — Step 4 `ss -s`/`ethtool -S`/`sar -n DEV`. The order (CPU → memory → disk → network →
waiting) runs from the cheapest and most definite measurement to the most expensive. · *Phase 7.5.1–7.5.2*

---

## Scoring

| Number correct | Assessment |
|---|---|
| 18–21 | You can walk the whole path from the bus to the bill. The hardware map is yours. |
| 14–17 | Good. Re-read the **bridge** sections (table below) that your missed questions point to. |
| 9–13 | You know the phases individually but struggle at the intersection. Redo the Part B scenario from scratch, question by question. |
| 0–8 | Walk through Phases 4, 5, 6 and 7 again; especially 4.2 (PCIe), 5.3 (latency and BDP), 6.3–6.5 (VM exit, steal) and 7.5 (diagnostic procedure). |

**Which question you missed → where to return:**

| Question you missed | Return to — this bridge is weak |
|---|---|
| 1, 17 | PCIe generations and lanes × the device it feeds (3.3.3 × 4.2.2 × 5.2.3) |
| 2 | DMA, interrupt and `wa` (3.4.3 × 4.3 × 4.4) |
| 3 | The lane budget (4.2.3) |
| 4 | Interrupt storm, NAPI, RSS (4.4.2 × 5.1.4) |
| 5, 13 | The four components of latency, RTT table (5.3.2 × 5.3.4) |
| 6 | The queue curve in the ring buffer (3.4.4 × 5.1.3) |
| 7 | VM exit cost and the virtualization tax (6.3.4 × 6.6) |
| 8 | TLB × EPT × huge pages (2.5.3–2.5.4 × 6.4.3) |
| 9 | Memory overcommit vs thrashing (2.6.2 × 6.4.4) |
| 10, 11, 18 | Steal time and t-family credits (6.5.2–6.5.3 × 7.6.1) |
| 12 | vCPU ≠ core, Graviton (1.5.3 × 7.1.6) |
| 14 | BDP and the TCP window (5.3.3) |
| 15 | Interrupt distribution, per-core reading (4.4.3 × 5.1.4) |
| 16 | The GPU feed line (4.2.4) |
| 19 | The two EBS ceilings (3.4.5 × 7.3.5) |
| 20 | Capacity planning and the queue curve (7.6.2–7.6.3) |
| 21 | The 20-minute diagnostic procedure (7.5) |

---

## Closing — from here to the next map

This quiz walked one application through four layers: a **bus** that can narrow the road (Phase 4), a **network**
whose latency is mostly geography (Phase 5), a **hypervisor** whose tax is exits × cost (Phase 6), and the
**decisions** where all of it becomes an instance type and a bill (Phase 7). The scenario's lesson repeats the
first quiz's: **the symptom appears in one place, the cause sits one layer away** — the CPU that "looked busy" was
being throttled (`st`), the "slow network" was a window and a round-trip count, the "packet loss" was one core
drowning in interrupts. Carry three habits forward: (1) measure per layer in the order CPU → memory → disk →
network → waiting; (2) do the arithmetic (BDP, credits, lane budget, exit cost) before you buy; (3) never run a
resource on the steep part of its queue curve.

The hardware map ends here. Its ground is what the next layers stand on: the **Linux map** (reading a running
server) and the **Network map** (IP, TCP, DNS, TLS), and after them AWS services, Terraform, containers and
observability. Services change, prices change; the physics does not.

---

> **Navigation:** [◀ Phase 7 — Cloud Connection](Phase_7_Cloud_Connection.md) · **Checkpoint Quiz 2** · [Back to the start of the map ▶](README_en.md)
