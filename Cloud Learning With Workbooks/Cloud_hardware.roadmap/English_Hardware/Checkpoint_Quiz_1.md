# Checkpoint Quiz 1 — Phases 0–3: From the Transistor to the Disk

> **Navigation:** [◀ Phase 3 — Storage](Phase_3_Storage.md) · **Checkpoint Quiz 1** · [Phase 4 — System Buses and I/O ▶](Phase_4_Buses_and_IO.md)

---

## What does this quiz measure?

The "Test yourself" tests at the end of a phase probe a single phase. A checkpoint quiz measures something
**different**: *"Can you connect these four phases to each other?"*

Phase 0 gave you the **physical rules**: powers of 2, the critical path, heat, "what is fast is small." Phase 1
put those rules **inside the CPU**: pipeline, IPC, cores, vCPU. Phase 2 showed the CPU's **one great weakness** —
memory is slow — and the hierarchy built to hide it. Phase 3 repeated the same hierarchy on **disk**, a thousand
times larger in scale. These are not four topics; they are **one story told at four scales**: a small number of
principles (hierarchy, locality, block transfer, the queue curve) repeat again and again. This quiz measures
whether you can see the story, not the chapters.

**How to study:**

- Solve with pen and paper; do not move on without writing the answer.
- The answer key tells you **at which intersection of phases** each question stands — a missed question points
  not to a phase but to the **bridge** between phases.
- There are 21 questions: Part A (connection reasoning, 1–9), Part B (scenario: "the database that got slow
  after the move to the cloud", 10–15), Part C (reading output and calculating, 16–21).
- The numbers you need are in Appendix A. Target time: ~1 hour. But time is not important; what matters is
  being able to justify in one sentence *why* each answer is what it is.

> **🤔 Before you start:** Complete this sentence in your own words: *"A CPU at 100% is not a diagnosis,
> because ___ also shows up as 100%."* Then add a second sentence: *"What is fast is ___; what is large is
> ___."* These two sentences are the thread that runs through all 21 questions.

---

# Part A — Connection reasoning (1–9)

Each question asks you to combine the knowledge of at least two phases. Give a short but justified answer.

**1.** In Phase 0 you saw that limits are powers of 2; in Phase 2 you met the 64-byte cache line and the
4,096-byte page. How many cache lines fit in one page? Why is it convenient for both sizes to be powers of 2 —
what does the hardware do with an address that it could not do so cheaply otherwise?

**2.** In Phase 0 the clock speed was limited by the **critical path**; in Phase 1 the pipeline appeared. Explain
how the pipeline raises the clock ceiling by attacking the critical path — and what it does **not** improve
(hint: one instruction's own latency).

**3.** In Phase 0 the heat budget stalled the clock and the cores multiplied. In Phase 1 you learned "vCPU ≠
core." On an x86 instance and on a Graviton instance, both with 8 vCPUs, how much physical compute do you get on
each, and why is "the same vCPU count" a trap when comparing them?

**4.** In Phase 1 you learned "CPU 100% is not a diagnosis"; in Phase 2 you learned the cost of a DRAM access. A
service shows 100% CPU in `top`, yet doubling the CPU count changes nothing. Which mechanism from Phase 2 can
make a "busy" CPU mostly idle, and which command (and which two counters) would separate real work from waiting?

**5.** In Phase 2 you used the formula `average access = hit × L1 + miss × RAM`. With L1 = 4 cycles and RAM = 200
cycles, compute the average access time at a 90% and a 99% hit ratio. By what factor does the program speed up
when the hit ratio goes from 90% to 99%, and why is that the "leverage of cache optimization"?

**6.** In Phase 2 you saw that swapping is the worst use of a disk (a major page fault). Put numbers on it with
the latency ladder from Phase 3: how many times slower than a RAM access (~70 ns) is a read from a local NVMe SSD
(50–100 μs)? And from an EBS gp3 volume (~1 ms)?

**7.** In Phase 2 the cache moves data in 64-byte **lines**; in Phase 3 the disk moves it in 4 KiB **blocks**. State
the single reason both hierarchies transfer in blocks. Then compare a 7200 RPM HDD reading 4 KiB blocks at
random (about 120 IOPS) with the same HDD reading sequentially (150–250 MB/s): roughly how many times slower is
the random pattern?

**8.** The queue curve appeared in Phase 3 (`await`), and it will return in the network and in capacity planning.
If a disk answers in 1 ms at 50% utilization, what latency do you expect at 80%, 90%, 95% and 99%? Which two
`iostat -x` columns show that you are on the steep part of the curve?

**9.** In Phase 0 the SRAM/DRAM distinction taught "what is fast is small; what is large is slow." Take these
five: L3 cache, DRAM, local NVMe instance store, EBS volume, HDD. Which are **volatile** and which
**persistent**? Which of the persistent ones survive an instance **stop**, and why does Phase 3 call
"no persistence needed" the only case for instance store?

---

# Part B — Scenario: "The database that got slow after the move to the cloud" (10–15)

> **Incident:** A team moved a PostgreSQL database from an on-premises server (256 GB RAM, local NVMe) to AWS: an
> `r`-family instance with **4 vCPU / 32 GiB RAM**, and a **500 GiB gp2** volume. The hot working set of the
> database is about **60 GB**. After the move, queries that took milliseconds now take seconds. The engineer
> collected this:
>
> ```
> $ top
> %Cpu(s):  8.1 us,  2.0 sy,  0.0 ni, 19.5 id, 70.2 wa,  0.0 hi,  0.2 si,  0.0 st
> $ free -h
>                total   used   free  shared  buff/cache  available
> Mem:            31Gi   30Gi  180Mi    40Mi       900Mi      800Mi
> Swap:          4.0Gi  3.1Gi  0.9Gi
> $ vmstat 1          # si/so columns:  si 200   so 300
> $ iostat -x 1       # the data volume
> Device   r/s     w/s    rkB/s    wkB/s    await  aqu-sz
> nvme1n1  1420    80     22720    1280     45     68
> ```
>
> The team's first reaction: "the CPU is too weak — let's double the vCPUs."

**10.** Read the `top` line. Is the CPU the bottleneck? Which single number shows what the machine is actually
doing, and why would doubling the vCPUs change almost nothing?

**11.** A gp2 volume gives `size × 3` IOPS. Compute the IOPS ceiling of this 500 GiB volume and compare it with
the `iostat` reading. Then use `Throughput = IOPS × I/O size` to find the average I/O size and the throughput,
and say which ceiling (IOPS or throughput) the volume has hit.

**12.** Look at `free -h` and `vmstat`. Why is `available` more meaningful than `free`? What do `si`/`so` > 0 tell
you? Explain step by step how a **memory** shortage (Phase 2) produced the **disk** load (Phase 3): what happens
to the cache/page-cache hit ratio when a 60 GB working set is squeezed into 32 GiB?

**13.** Use Little's law from Phase 3 (`await ≈ aqu-sz ÷ IOPS`) to check whether the `iostat` numbers are
consistent with each other. What does the result tell you about where the time is being spent?

**14.** The team has two fixes on the table: (a) move to a gp3 volume with 16,000 IOPS; (b) move to a larger
`r`-family size with at least 64 GiB of RAM. Which do you do **first**, and why? What single new number does each
fix change, and what must you still verify about the instance itself when choosing an EBS type?

**15.** After the fix, the database runs fine on a normal day but the p99 latency spikes at peak hours when the
volume is around 95% busy. Which curve explains this? At what utilization would you plan to stay, and what does
that tell you about buying capacity "just enough"?

---

# Part C — Reading output and calculating (16–21)

**16.** A default gp3 volume (3,000 IOPS, 125 MB/s) shows:

```
Device   r/s     w/s     rkB/s    wkB/s    await  aqu-sz
nvme1n1  2900    100     92800    3200     12     36
```

(a) Compute the total IOPS, the throughput and the average I/O size. (b) Which ceiling is the volume at? (c)
If the same workload used 64 KiB I/Os, what would happen at 3,000 IOPS — which ceiling would bind first?

**17.** `lscpu` on a physical host prints: `Thread(s) per core: 2`, `Core(s) per socket: 8`, `Socket(s): 2`, `NUMA
node(s): 2`. (a) How many logical CPUs does the operating system see, and how many physical cores are there? (b)
A process running on node 0 keeps reading memory that lives on node 1: what happens to its memory latency, and
which command family (Phase 2.7) shows the distances and lets you fix it?

**18.** This `vmstat 1` line comes from a slow server:

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 3  9 4194304  81920   2048 153600  850  1100  5200  4800  900 1500  4  3  5 88  0
```

Name the **two independent alarms** in this line, the phase each belongs to, and the causal chain that connects
them. Which resource do you fix first?

**19.** A database scans a 24 GiB working set at random. The TLB holds 1,536 entries. Compute the TLB reach with
4 KiB pages and with 2 MiB huge pages, and the fraction of the working set each can cover. Why do huge pages
help this workload?

**20.** (a) Compute the random IOPS of a 7200 RPM HDD from its three latency components (seek ≈ 4 ms, transfer ≈
0.01 ms). (b) Do the same for a 15,000 RPM disk with the same seek and transfer times: by how much did the
faster spindle help, and what does that say about mechanical disks? (c) Six 7200 RPM disks (120 IOPS each) run
as RAID 5: what are the approximate read and write IOPS, given the 4× write penalty?

**21.** Put three numbers on the human scale of Appendix A.1 (1 cycle = 1 second, CPU at 3 GHz): a RAM access
(~70 ns), an EBS gp3 read (~1 ms) and an HDD random read (~10 ms). Then say in one sentence what the ladder
teaches that "RAM is fast, disk is slow" does not.

---

## Answer key

**1.** 4096 ÷ 64 = **64 cache lines** per page. Both are powers of 2 (2⁶ and 2¹²), so the hardware splits an
address just by **cutting bits**: the low 6 bits are the offset inside a line, the next 6 select the line in the
page, the upper bits select the page. No division or multiplication circuit is needed — the "every limit is a
power of 2" reflex is an address-decoding economy. · *Phase 0.1 × Phase 2.3.5*

**2.** The clock period cannot be shorter than the **longest signal path** (the critical path) between two
flip-flops. The pipeline cuts one instruction's work into stages, so each stage's critical path is short and the
clock can be raised; a new instruction enters every cycle. It does **not** shorten the time one instruction takes
from start to finish (its latency stays roughly the same or even grows slightly) — it improves **throughput**,
not latency. · *Phase 0.3.3 × Phase 1.4.1*

**3.** Heat stopped the clock from rising, so vendors added cores instead. On x86, **1 vCPU = 1 SMT thread**
(half a physical core), so 8 vCPUs ≈ 4 cores; on Graviton **1 vCPU = 1 physical core**, so 8 vCPUs = 8 cores. The
same vCPU count therefore hides up to a 2× difference in physical compute, which is why the vCPU number alone must
never be compared across architectures. · *Phase 0.2.3 × Phase 1.5.3*

**4.** A DRAM access costs ~200–250 cycles; when the data is not in the cache the CPU **waits**, and waiting is
still counted as "busy" (CPU 100% is not a diagnosis). More cores do not shorten the wait. `perf stat -e
cache-misses,instructions,cycles <command>` shows it: a low **instructions per cycle** (IPC) and a high cache-miss
count mean the CPU is stalled on memory rather than computing. · *Phase 1.3.2 × Phase 2.1*

**5.** At 90%: 0.9 × 4 + 0.1 × 200 = 3.6 + 20 = **23.6 cycles**. At 99%: 0.99 × 4 + 0.01 × 200 = 3.96 + 2 =
**5.96 cycles**. 23.6 ÷ 5.96 ≈ **4×** faster. A 9-point improvement in hit ratio multiplies the speed because
the rare **misses** dominate the average (they cost 50× a hit): removing 9 misses per 100 accesses removes most
of the time. That is the leverage of cache optimization. · *Phase 2.3.6*

**6.** NVMe: 50–100 μs ÷ 70 ns ≈ **700–1,400×** slower than RAM. gp3: 1 ms ÷ 70 ns ≈ **14,000×**. A major page
fault (or swap-in) pays this price on every miss — which is why swapping is the worst possible use of a disk. ·
*Phase 2.1.1 × Phase 2.6 × Phase 3.3.3*

**7.** The cost of an access is mostly **fixed latency** (finding the data), not the transfer of each extra
byte; so one pays the fixed cost once and brings a whole block along (64 B on the cache line, 4 KiB on the disk),
betting on **locality**. Random 4 KiB HDD reads: 120 × 4 KiB ≈ 480 KiB/s ≈ 0.5 MB/s, against 150–250 MB/s
sequentially — the random pattern is roughly **300–500× slower** (the workbook's fuller model gives 466×). · *Phase 2.3.5 × Phase 3.1.2–3.1.3*

**8.** Relative to 50%: 80% → 2.5 ms, 90% → 5 ms, 95% → **8 ms**, 99% → **40 ms**. The columns are **`await`** (the
average time per request) and **`aqu-sz`** (the average queue length); when they climb together, the disk is on
the steep part of the queue curve. · *Phase 3.4.4 (the curve recurs in Phase 5 and 7.6.2)*

**9.** Volatile: **L3 and DRAM** (SRAM/DRAM lose their contents without power) and the **instance store** (it
lives on the host; its data is lost when the instance is stopped or the host fails). Persistent: **EBS** and
**HDD/SSD disks in general**. EBS survives an instance stop. Instance store is right only when no persistence is
needed (scratch, cache, replicated data) — in exchange for the highest IOPS. Persistence costs speed:
what is fast is small and volatile; what is large is slow and persistent. · *Phase 0.4.4 × Phase 3.3*

**10.** No. **`wa` = 70.2%** — the CPU is idle, **waiting for I/O**; `us` is only 8%. Doubling the vCPUs adds more
processors that would also wait for the same disk: the machine is **I/O bound** (and memory-bound behind it). CPU
count was the wrong knob. · *Phase 1.5.4 × Phase 3.4.3*

**11.** 500 × 3 = **1,500 IOPS**. The measurement: r/s + w/s = 1,420 + 80 = **1,500** — exactly at the ceiling.
Throughput: 22,720 + 1,280 = 24,000 kB/s ≈ **24 MB/s**, so the average I/O size is 24,000 ÷ 1,500 = **16 KiB**.
The volume's throughput limit is nowhere near; the **IOPS ceiling** has been hit. · *Phase 3.4.1*

**12.** `available` (800 Mi) is what can actually be given to a process without swapping (it includes reclaimable
cache); `free` only counts totally unused pages and is almost always tiny on a healthy machine, so it misleads.
`si`/`so` > 0 means the system is **actively swapping** — an urgent signal. The chain: 60 GB of hot data cannot
live in 32 GiB → the page cache/buffer cache **hit ratio falls** → most reads now miss and go to the disk
(1 ms instead of 70 ns) → the memory shortage turns into **disk load** (r/s at the ceiling) → `wa` 70%. ·
*Phase 2.6 × Phase 3.4.4*

**13.** 68 ÷ 1,500 ≈ 0.0453 s ≈ **45 ms** — exactly the reported `await`. The numbers are consistent: requests are
spending their time **in the queue**, not in the service; the 68 outstanding requests are waiting behind a
volume that can only do 1,500 per second. · *Phase 3.4.4 (Little's law, Answer 3.3)*

**14.** Fix the **memory first**: (b) a larger `r` size (≥ 64 GiB) lets the working set live in RAM, so most of
those 1,500 IOPS simply **stop being needed** — it removes the cause; a faster volume (a) would only make the
symptom cheaper (gp3 gives 3,000–16,000 IOPS) while every miss still costs ~1 ms. (b) changes the **hit ratio**;
(a) changes the **IOPS ceiling**. Then still check **both ceilings**: the volume limit **and** the instance's own
EBS bandwidth limit (`aws ec2 describe-instance-types … EbsInfo`). Moving gp2 → gp3 afterwards is cheap and
sensible. · *Phase 2.6 × Phase 3.4.5 × Phase 7.3.5*

**15.** The **queue curve**: at 95% utilization latency is about **8×** the 50% value, at 99% about 40×. p99 spikes
are the first thing that shows it. Plan capacity so peak utilization stays around **60–80%** (2.5× at 80%, not
8×): "just enough" capacity means running on the steep part of the curve. · *Phase 3.4.4 × Phase 7.6.2*

**16.** (a) IOPS = 2,900 + 100 = **3,000**; throughput = 92,800 + 3,200 = 96,000 kB/s = **96 MB/s**; I/O size =
96,000 ÷ 3,000 = **32 KiB**. (b) **The IOPS ceiling** (3,000) — the throughput of 96 MB/s is still under 125
MB/s. (c) 3,000 × 64 KiB = **192 MB/s** > 125 MB/s: the **throughput ceiling** would bind first, at roughly
125,000 ÷ 64 ≈ 2,000 IOPS. (Little's check: 36 ÷ 3,000 = 12 ms = `await` ✓.) · *Phase 3.4.1 × Phase 3.4.5*

**17.** (a) 2 × 8 × 2 = **32 logical CPUs** (= 32 vCPUs if sold as x86 instances), but only **16 physical cores**
(2 sockets × 8). (b) Remote-node memory is **~1.5–2× slower** (~140 ns instead of ~70 ns). `numactl --hardware`
(distances) and `numastat -m` show it; `numactl` lets you pin the process and its memory to one node. · *Phase 1.5.1
× Phase 2.7*

**18.** Alarm 1: **`si` 850 / `so` 1100 > 0** — the machine is swapping (memory, Phase 2.6; `free` is only 80 MB).
Alarm 2: **`wa` 88** — the CPU is waiting for I/O, `us` only 4% (disk, Phase 3.4). The chain: memory is
exhausted → pages are swapped to and from disk → the disk is saturated → every process waits (`b` = 9 blocked).
Fix the **memory** first (more RAM or a smaller working set); a faster disk only hides the swap. · *Phase 2.6 ×
Phase 3.4.4*

**19.** 4 KiB pages: 1,536 × 4 KiB = **6 MiB** → 6 MiB ÷ 24 GiB ≈ **0.024%** of the working set. 2 MiB pages: 1,536
× 2 MiB = **3 GiB** → 3 ÷ 24 = **12.5%** (512× larger reach). With random access over 24 GiB, 4 KiB pages make
nearly every access a TLB miss (a page-table walk); huge pages make one in eight hit the TLB directly. · *Phase
2.5.3–2.5.4*

**20.** (a) Rotational = 60,000 ÷ 7,200 ÷ 2 = **4.17 ms**; total = 4 + 4.17 + 0.01 ≈ **8.2 ms** → 1 ÷ 0.0082 ≈
**120 IOPS**. (b) Rotational = 60,000 ÷ 15,000 ÷ 2 = **2 ms**; total = 4 + 2 + 0.01 ≈ 6 ms → **~165 IOPS**: a
doubled spindle speed gains only ~35% because the seek (moving the head) does not change — a mechanical limit;
HDD IOPS has not grown in 30 years while capacity grew 1000×. (c) Read ≈ 6 × 120 = **720 IOPS**; write ≈ 720 ÷ 4 =
**180 IOPS**. · *Phase 3.1.2–3.1.3 × Phase 3.5.2*

**21.** 3 GHz → 1 ns = 3 cycles. RAM: 70 ns ≈ 210 cycles ≈ **3.5 minutes**. gp3: 1 ms = 3,000,000 cycles ≈ 35 days ≈
**about a month**. HDD: 10 ms = 30,000,000 cycles ≈ 347 days ≈ **about a year**. The ladder shows that the
differences are not "a bit" but **orders of magnitude**: a disk read is to RAM what a month-long wait is to a
coffee break — which is why every design decision in the map tries to avoid the next rung down. · *Phase 2.1.1 ×
Phase 3.1.2*

---

## Scoring

| Number correct | Assessment |
|---|---|
| 18–21 | You see the four phases as one story. You are ready for Phase 4. |
| 14–17 | Good. Re-read the **bridge** sections (table below) that your missed questions point to. |
| 9–13 | You know the phases individually but struggle at the intersection. Redo the Part B scenario from scratch. |
| 0–8 | Walk through Phases 0, 1, 2 and 3 again; especially 2.3 (cache), 2.6 (swap) and 3.4 (IOPS and the queue curve). |

**Which question you missed → where to return:**

| Question you missed | Return to — this bridge is weak |
|---|---|
| 1 | Powers of 2 × cache line and page (0.1 × 2.3.5) |
| 2 | Critical path × pipeline (0.3.3 × 1.4) |
| 3 | Heat budget × cores and vCPU (0.2.3 × 1.5) |
| 4 | "CPU 100%" × DRAM stalls (1.3.2 × 2.1) |
| 5 | Hit ratio and average access time (2.3.6) |
| 6, 21 | The latency ladder in numbers (2.1.1 × 3.1.3) |
| 7 | Block transfer × sequential vs random (2.3.5 × 3.1) |
| 8, 15 | The queue curve (3.4.4) |
| 9 | Fast = small × volatile vs persistent (0.4.4 × 3.3) |
| 10, 13, 16 | Reading `iostat` / Little's law (3.3 × 3.4) |
| 11 | IOPS × I/O size = throughput (3.4.1) |
| 12, 14, 18 | Memory shortage → swap → disk load (2.6 × 3.4) |
| 17 | Cores, threads, NUMA (1.5 × 2.7) |
| 19 | TLB reach and huge pages (2.5.4) |
| 20 | HDD components and RAID (3.1.2 × 3.5) |

---

## Closing — from here to Phase 4

This quiz tested four phases as one story: the physical rules (Phase 0), the CPU (Phase 1), the memory that
starves it (Phase 2) and the disk that starves the memory (Phase 3). The scenario's lesson is single and
expensive: **the symptom appeared at the bottom of the ladder (a slow disk), but the cause was one rung higher
(too little memory).** "CPU too weak, let's add vCPUs" was the reflex Phase 1 warned you about; the numbers
from Phase 2 and Phase 3 turned the diagnosis into arithmetic. Carry three lessons forward: (1) waiting counts as
busy — find *what* is waited on; (2) the same curve (hierarchy → locality → block → queue) repeats at every
scale; (3) measure, then compute, then buy.

Phase 4 opens the gap we left on purpose: **how does data actually travel** from the disk to the CPU — PCIe
lanes, DMA and interrupts. Every "fast" device you met in these four phases hangs off that path.

---

> **Navigation:** [◀ Phase 3 — Storage](Phase_3_Storage.md) · **Checkpoint Quiz 1** · [Phase 4 — System Buses and I/O ▶](Phase_4_Buses_and_IO.md)
