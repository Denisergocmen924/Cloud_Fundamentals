# Phase 4 — Memory, I/O and Performance Intuition

> **Navigation:** [◀ Phase 3 — Processes and Resources](Phase_3_Processes_and_Resources.md) · **Phase 4** · [Checkpoint Quiz 2 ▶](Checkpoint_Quiz_2.md)

---

## Where we are coming from

In Phase 3 we opened up the atom of a running system — the process: how it is born, what state it is
in, how it dies, how its resources are capped. When you open `top` on a server now, you read the table
you see from the model, not from memory. But at the end of Phase 3 we left one table half-finished.
Remember — you opened `top` on a server and saw this:

```
%Cpu(s):  4.0 us,  2.0 sy,  0.0 ni, 30.0 id, 63.5 wa, ...
```

Load was 6.2 but most of the CPU was `wa` (I/O wait) and `id` (idle). We said "the bottleneck is not
the CPU but the disk" — but we didn't open up **why**. This phase opens exactly what is under that
`wa` line.

Three things from Phase 3 will pay off directly here:

- **Load average = the count of R + D processes** (Phase 3.2.1). Now we'll see metric by metric why
  this doesn't mean "the CPU is busy", and why the D-state inflates load.
- **When a cgroup memory limit is exceeded, OOM happens** (Phase 3.6.2). Now we'll open up how the OOM
  Killer decides **whom** to kill (`oom_score`), and how swap fits into this picture.
- **`ps` has `%MEM`, `VSZ`, `RSS` columns** (Phase 3.7). Here we'll learn what those columns mean, and
  why `VSZ` is not "used memory".

## The question of this phase

This phase carries Phase 3's "what is the system doing" question onto the **resource** axis:

> *"What is eating the resources — and when someone says 'it's slow', where is the bottleneck?"*

The sentence a cloud engineer hears most is "the app is slow". But "slow" can have four different
causes: is the CPU saturated, is memory exhausted, is the disk choked, is the network congested? These
four are diagnosed with **different** tools and demand **different** fixes. A wrong diagnosis leads to
a wrong "get a bigger instance" decision and wasted money.

By the end of this phase you will **correct** the panic sentence "RAM is 95% full" when you look at
`free -h` — because most of that number is page cache and the real pressure is read in the `available`
column. And when someone says "my app is slow", you will start with a single command and narrow, in 60
seconds, which of the four bottleneck types it is. This is the OS side of EC2 right-sizing and of
reading CloudWatch metrics later.

---

## By the end of this phase

- You will be able to explain the difference between virtual and physical memory; between `RSS` (the
  RAM actually used) and `VSZ` (the virtual space reserved); and why summing processes' `RSS` is
  misleading (shared memory)
- You will be able to explain why Linux uses free RAM as disk cache (page cache); that the
  `buff/cache` in `free` is actually **available**; and that real pressure is read in the `available`
  column
- You will be able to explain what swap is and why it's slow; what `swappiness` tunes; and how, when
  RAM runs out, the OOM Killer decides **whom** to kill and why (`oom_score`)
- You will be able to interpret load average correctly against the core count; and know that "high
  load + low CPU" is almost always I/O wait
- You will be able to explain what `iowait` tells you; the difference between blocking and
  non-blocking I/O; and why disk saturation makes the whole system look "slow"
- You will be able to describe, as a methodology, the CPU-bound / memory-bound / I/O-bound /
  network-bound distinction when someone says "my app is slow", and which tool (`free`, `vmstat`,
  `iostat`, `top`, `/proc/meminfo`) reveals which

---

## The phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 4.1 | Virtual memory and process memory: RSS vs VSZ | `[mechanism]` | The correct answer to "how much RAM does the app eat" |
| 4.2 | Page cache: the "free RAM" fallacy | `[mechanism]` + `[application]` | **The most misread metric in cloud** |
| 4.3 | Swap and the OOM Killer | `[mechanism]` | What happens when memory runs out; who dies, why |
| 4.4 | Reading load average correctly | `[mechanism]` + `[application]` | What is under the `wa` line left from Phase 3 |
| 4.5 | I/O intuition: iowait, blocking I/O | `[concept]` + `[application]` | How "disk choked, CPU idle" happens |
| 4.6 | Bottleneck triage (from the OS side) | `[application]` | **The phase output** — narrow the bottleneck in 60s |
| 4.7 | When this phase breaks | — | The failure signatures of memory and I/O |

> **How to work through this phase:** Almost **all** of this phase's commands are 🟢 (read-only —
> reading `free`, `vmstat`, `iostat`, `top`, `/proc/meminfo`). This is an "observation" phase; you
> don't change anything permanent. Only a couple of boxes are 🟡: testing a cgroup limit live with
> `systemd-run`, or applying artificial load with `stress` — these are temporary and vanish when the
> job ends. Changing `swappiness` or OOM settings would be 🔴, but in this phase we only **read**,
> we don't change. The most illuminating moment: leaving `free -h` open and reading a file with
> `cat big_file > /dev/null` to watch `buff/cache` grow live.

---
---

# 4.1 Virtual Memory and Process Memory

## 4.1.1 Virtual memory: every process thinks it's alone `[mechanism]`

In Phase 3 we saw that a process is a "task" in the kernel's eyes. Now we look at that task's
**memory**. The first model we need to build: **no process touches physical RAM directly.**

Every process lives in a **virtual address space** the kernel presents to it. This space looks, to the
process, like a private, contiguous, huge memory starting at 0 — as if it were the only process on the
machine and all the memory were its own. But this is an illusion: the **virtual addresses** the
process sees are translated behind the scenes into **physical addresses** by the kernel and hardware
(the MMU — Memory Management Unit). Two different processes can use the same virtual address
(`0x400000`), but these land in completely different regions of physical RAM.

Why does this exist? Three reasons:

1. **Isolation.** A process cannot address another process's memory — because its virtual addresses
   translate to different physical places. This is the memory side of Phase 2's identity isolation.
2. **Simplicity.** Every programmer can assume "my memory starts at 0"; they don't have to think about
   where physical RAM happens to be free.
3. **Flexibility.** The kernel can back a virtual page with physical RAM, with disk (swap), or with
   nothing (not yet touched). This flexibility is the foundation of half of this phase.

> **🔧 See it on your machine** 🟢 — a process's virtual memory map
>
> ```
> $ cat /proc/self/maps | head -5
> 55a3c1e00000-55a3c1e21000 r--p 00000000 08:01 1183     /usr/bin/cat
> 55a3c1e21000-55a3c1ea6000 r-xp 00021000 08:01 1183     /usr/bin/cat
> 55a3c1ea6000-55a3c1ecb000 r--p 000a6000 08:01 1183     /usr/bin/cat
> ...
> 7fff...       rw-p 00000000 00:00 0        [stack]
> ```
>
> `/proc/self/maps` is the virtual address map of the process reading it (this `cat`). Each line is a
> **region**: the address range on the left is virtual, `r-xp` are the permissions (like Phase 2's
> mode bits but for memory), and on the right is what the region is **backed by** — a file
> (`/usr/bin/cat`), the stack, the heap. "Everything is a file" (Phase 0) holds here too: even a
> process's memory is read like a file.

**Page.** The kernel manages memory not byte by byte but in fixed-size blocks: these are called
**pages**, typically 4 KB. Virtual-to-physical translation happens at page granularity. This detail
may look small now, but page cache, swap and OOM all work in units of "pages" — so get to know the
word already.

---

## 4.1.2 RSS vs VSZ: "reserved" versus "actually used" `[mechanism]`

Now we can separate the two memory columns you see in `ps` and `top` — this distinction is the
foundation of the correct answer to "how much RAM does this app eat".

- **VSZ (Virtual Size).** The **total** size of the process's virtual address space. That is, all the
  virtual space the process has "reserved" from the kernel: code, data, stack, loaded libraries,
  regions requested via `malloc` but not yet **touched** — all of it. This number is often very large
  and has **no direct relation to physical RAM**.
- **RSS (Resident Set Size).** The total of the process's virtual pages that are currently **actually
  resident in physical RAM**. "Resident" = living in RAM. How much the app actually occupies in RAM at
  that moment is shown by this number — not VSZ.

The source of the difference is the nature of virtual memory: a process can `malloc(1 GB)` and VSZ
grows by 1 GB — but until it **touches** that 1 GB, the kernel allocates not a single physical page.
This is called **lazy allocation**. Only when the process actually writes to a page does the kernel
bind a physical page at that moment (via a page fault), and RSS grows by that much. So:

> **VSZ = the space the process promised itself (reserved); RSS = the RAM the process actually uses.**

> **🔧 See it on your machine** 🟢 — catch the gap between VSZ and RSS
>
> ```
> $ ps -eo pid,comm,vsz,rss --sort=-rss | head -6
>    PID COMMAND            VSZ    RSS
>   1421 mysqld         2148372 512400
>    987 java           4821960 398210
>    623 systemd-journ   58210  18400
> ```
>
> `mysqld`'s VSZ is ~2.1 GB but its RSS is ~512 MB. So MySQL has reserved 2.1 GB of **virtual** space
> but actually holds ~512 MB in physical RAM. "MySQL eats 2 GB of RAM" would be wrong — that would be
> reading VSZ. The correct sentence: "MySQL is using ~512 MB RSS." **Rule: always read memory usage
> from RSS, not VSZ.** (In `top` too, `RES` = RSS, `VIRT` = VSZ.)

> **⚠️ Common misconception: "If VSZ is large, the process eats a lot of RAM."**
> No. A large VSZ only shows that the process has reserved a lot of **virtual** space — which is nearly
> free and costs no physical RAM until touched. JVM- and Go-style runtimes in particular reserve
> enormous virtual space at startup (VSZ can be tens of GB) while RSS stays small. Your panic metric is
> always RSS.

---

## 4.1.3 Shared memory: why you can't sum RSS `[concept]`

One step further: say you have 10 Apache/PHP worker processes, each showing an RSS of 200 MB. Is it
correct to say "then it eats 10 × 200 MB = 2 GB of RAM"? **No** — and the reason is this phase's
cleverest detail.

A large part of a process's RSS is often **shared** pages. The most typical example: **shared
libraries** (`libc`, `libssl`, etc.). All 10 workers use the same `libc`, and the kernel keeps this
library as **a single copy in physical RAM** and maps those same physical pages into all 10 processes'
virtual spaces. But each process's RSS counts these shared pages **toward its own share**. So the same
50 MB `libc` looks like 500 MB in the sum of 10 processes' RSS — while there is a single 50 MB in
physical RAM.

The same holds for `fork` (recall Phase 1.1.2): a child born from `fork` initially **shares** the
parent's memory pages (Copy-on-Write). A single copy is used until the parent and child write to a
page. So the sum of the parent's + child's RSS looks larger than the real physical usage.

> **🤔 Think 4.1** — On a server you see 8 `php-fpm` workers with `ps`, each with an RSS of 180 MB. A
> colleague says "these eat 8 × 180 = 1.44 GB of RAM, the instance is undersized." Why might this
> arithmetic be wrong, and with which concept do you explain the real physical usage?
>
> *(Answer: at the end of the phase)*

The practical consequence: **don't try to find "total RAM usage" by summing processes' RSS.** This
always overestimates. The total real usage must be read from the system level — from `free` or
`/proc/meminfo`. And this brings us to the heart of this phase, the "free RAM" fallacy.

> **❓ Question that comes to mind: "Then how do I see the RAM a process really uses 'privately'
> (unshared)?"**
> Instead of `RSS`, you look at `PSS` (Proportional Set Size) or, more precisely, the `Private` lines
> in `/proc/<PID>/smaps_rollup` — shared pages are divided by the number of processes. In practice the
> `smem` tool gives this readably. We don't go deep in this phase, but note the reflex "for real usage,
> look at PSS".

---
---

# 4.2 Page Cache — The "Free RAM" Fallacy

## 4.2.1 Linux doesn't see free RAM as waste: page cache `[mechanism]`

Now we reach the most misread topic of this phase — perhaps of the whole map. You log into a fresh
server, run `free -h`, see that 90% of RAM appears "in use" and might panic. Almost always this is a
fallacy. To understand why, we need to establish one of Linux's philosophies:

> **Free RAM is wasted RAM.**

Linux considers it wasteful to leave unused RAM simply empty. Instead, it keeps everything read from
disk — file contents — in RAM as a **copy**. This is called **page cache**. The logic: disk is
thousands of times slower than RAM (Phase 0 / hardware intuition). If you've read a file from disk
once, the kernel keeps it in page cache; the second time you read it, it serves it from RAM without
going to disk at all — that is, it **flies**.

That's why RAM "fills up" over time on a running system: the kernel accumulates read files in the
cache. But this seemingly-used RAM is **reclaimable at any moment**: if a new process asks for RAM,
the kernel instantly releases part of the page cache (because it's already on disk, nothing is lost)
and gives it to the process. So page cache is "in use" and "available" at the same time.

> **🔧 See it on your machine** 🟢 — grow the page cache live
>
> ```
> $ free -h
>                total        used        free      buff/cache   available
> Mem:            7.7Gi       1.2Gi       5.1Gi        1.4Gi        6.1Gi
> $ cat /var/log/*.log > /dev/null      # read big files (into cache, not to disk)
> $ free -h
>                total        used        free      buff/cache   available
> Mem:            7.7Gi       1.2Gi       3.0Gi        3.5Gi        6.1Gi
> ```
>
> Notice: `used` barely changed (1.2 GB), `free` dropped, `buff/cache` swelled (1.4 → 3.5 GB) —
> because the files you read went into page cache. But the most important column: **`available`
> stayed the same (6.1 GB).** That is, "really available" memory didn't change; only free RAM turned
> into cache. This is live proof that page cache is "reclaimable".

Let's make the columns of `free` explicit:

- **`used`** — the (anonymous) memory processes actually hold + the kernel. This is the real "spent".
- **`buff/cache`** — page cache + buffers. Looks full but is **reclaimable**.
- **`free`** — completely untouched, fully empty RAM. On a running system it is **normal** for this to
  be small.
- **`available`** — the kernel's estimate: how much RAM it can give a new app without swapping to
  disk. **This is the column that measures real pressure.**

![Figure 4.1 — What `free -h` really shows: used (real pressure) + buff/cache (reclaimable) → available](../diagrams/png/lx-4-01-free-memory-anatomy.png)

*Figure 4.1 — Left: the wrong model a naive look at `free -h` builds ("RAM is 98% full, panic"). Right:
the real model — `used` is small, `buff/cache` is reclaimable, and the real decision column is
`available`. The same RAM, read two ways.*

## 4.2.2 Reading `free` correctly: the `available` column `[application]`

Let's boil the rule down to one sentence:

> **The answer to "how full is RAM?" is not `used`/`total`, it's `available`/`total`.**

If `available` is high (say, over 40% of total), it doesn't matter how "full" RAM looks — the system
is relaxed. If `available` approaches zero, that's when there is real memory pressure and the swap /
OOM risk begins.

> **⚠️ Common misconception: "In `free`, `used` is high / `free` is low, RAM is running out, we need a
> bigger instance."**
> This is one of the most expensive misreadings in cloud. If `used` is low and the only thing dropping
> is the `free` column, what you see is page cache — the system is healthy and a bigger instance is a
> **waste of money**. Always look at `available` before deciding. CloudWatch's `MemoryUtilization`
> metric is the remote version of this fallacy: the default calculation often counts cache as "in
> use"; for the right value, a metric like `mem_available_percent` is used.

> **🤔 Think 4.2** — A monitoring panel raises a red alarm "RAM usage 88%". You log into the server and
> run `free -h`: `used` 2 GB, `buff/cache` 12 GB, `available` 11 GB, total 16 GB. Is the alarm right?
> What do you say in one sentence, and what would the situation that **should** raise an alarm actually
> be?
>
> *(Answer: at the end of the phase)*

---
---

# 4.3 Swap and the OOM Killer

## 4.3.1 Swap: the disk area RAM overflows into `[mechanism]`

What happens when `available` really approaches zero? The kernel has two cards. The first is **swap**.

**Swap** is a special area reserved on disk (a partition or a file). When RAM is full and the kernel
needs more physical pages, it takes the **least-used** memory pages from RAM at that moment and writes
them to this disk area (this is called "swap out") and uses the freed RAM for the urgent need. When
that page is touched again, it is read back from disk ("swap in").

Swap's critical property: **memory written to disk runs at disk speed.** That is, thousands of times
slower than RAM. If a process's actively-running pages start going in and out of swap (this is called
**thrashing**), the system "works" on the surface but, because every memory access has turned into a
disk access, it slows to a crawl. The CPU looks idle, but no work progresses.

> **🔧 See it on your machine** 🟢 — read swap usage and activity
>
> ```
> $ free -h
>                total        used        free      buff/cache   available
> Mem:            3.8Gi       3.4Gi       120Mi        280Mi        180Mi
> Swap:           2.0Gi       1.6Gi       0.4Gi
> $ vmstat 1 3
> procs -----------memory---------- ---swap-- ...  ----cpu----
>  r  b   swpd   free   buff  cache   si   so  ...  us sy id wa
>  2  1 1600000 120000  ...          420  380  ...   5  3 12 80
> ```
>
> The danger sign here: `available` is only 180 MB (RAM under pressure) **and** `vmstat`'s `si`/`so`
> (swap-in / swap-out) columns are not zero — the system is actively going in and out of swap. `wa`
> (I/O wait) is 80: most of the CPU is spent waiting on the disk (swap). This is the classic
> "thrashing" signature.

> **💡 Cloud connection — why swap is dangerous in production:** In most production environments
> (Kubernetes especially) swap is **turned off**. The reason: swap hides the reality that an app is
> "out of memory" — the app slows down instead of dying, and this slowdown turns into a
> very-hard-to-diagnose performance problem. On a swap-less system, if memory runs out the app dies
> **quickly** via OOM (a clear signal); on a swap-ful system it crawls for hours and misleads everyone.
> "Fast and clear failure" is often preferred to "slow and ambiguous".

**`swappiness`** — a 0–100 value that tunes how "eager" the kernel is to swap
(`/proc/sys/vm/swappiness`). A high value: the kernel swaps process memory out earlier to protect page
cache. A low value (e.g. 10): keep it in RAM as much as possible, swap as a last resort. On servers it
is usually kept low. In this phase we only **read** it (`cat /proc/sys/vm/swappiness`); changing it is
a 🔴 operation and affects production behavior.

## 4.3.2 OOM Killer: who dies when RAM runs out `[mechanism]`

The kernel's second — and last — card: if swap is also exhausted, or there is no swap and RAM is
completely spent, the kernel is in an impossible situation. A process wants more memory but there are
no pages to give. In this deadlock the kernel makes a hard decision: **it kills a process and reclaims
its memory.** This mechanism is called the **OOM Killer** (Out Of Memory Killer).

In Phase 3.6.2 we saw that when a cgroup limit is exceeded, an "in-group OOM" happens. The
**system-wide** OOM here is the same logic at machine scale. But the critical question: **whom** does
the kernel kill?

The kernel gives every process an **`oom_score`**. This score is roughly based on "how much RAM would
be reclaimed if I killed this process" — so the process holding the most memory gets the highest score
and is the first candidate to be killed. (An administrator can pull a process's score down or up with
`oom_score_adj`; for example, putting a critical database on the "kill these last" list.)

> **🔧 See it on your machine** 🟢 — read an OOM event from `dmesg`
>
> ```
> $ dmesg -T | grep -i "killed process"
> [Tue Sep 16 03:14:22] Out of memory: Killed process 2481 (python3)
>   total-vm:8123400kB, anon-rss:7981200kB, file-rss:0kB, oom_score_adj:0
> ```
>
> The kernel always logs the OOM event. What to read here: **which** process died (`python3`, PID
> 2481), how much it held (`anon-rss` ~8 GB — anonymous, i.e. real non-cache memory), and
> `oom_score_adj:0` (not adjusted). The answer to the complaint "my app died out of nowhere" is almost
> always here — first `dmesg | grep -i oom`.

> **🤔 Think 4.3** — On a server at 3 a.m. the database process (the one holding the most RAM) suddenly
> died and restarted. In `dmesg` you see "Out of memory: Killed process ... (postgres)". (1) Did the
> database break, or did something else exhaust the RAM? How can you tell? (2) How do you keep the
> database from being the first to die on this list?
>
> *(Answer: at the end of the phase)*

> **💡 Cloud connection — the memory leak scenario:** In production the most common cause of OOM is
> memory an app leaks over time: RSS grows slowly over hours, `available` drops, and eventually the OOM
> Killer kicks in and usually kills the process holding **the most RAM** — often the leaking app
> itself, sometimes an innocent but large neighbor. The correct diagnosis: watch the **slope** of RSS
> over time (CloudWatch/Prometheus). A single instantaneous reading misleads; a leak is a **trend**.

---
---

# 4.4 Reading Load Average Correctly

## 4.4.1 Load = run queue length, not "CPU busy" `[mechanism]`

In Phase 3 we said load average is "the time-average of the count of R + D processes". Now let's
solidify this definition, because it's the most misunderstood metric in the Linux world.

The three numbers above `uptime` or `top` — for example `load average: 2.10, 1.80, 1.20` — are the
averages over the last **1, 5 and 15 minutes** respectively. The average of what? Of the count of
processes that "want to work":

- **running** (R — on the CPU right now) +
- **ready to run, waiting in line** (R — would run immediately if a CPU frees up) +
- **uninterruptible sleep** (D — usually locked in disk/NFS I/O).

There are two critical points. **First:** load is not a percentage but a **number**, and it's read
**against the core count**. On a 4-core machine, load 4.0 = "completely full, at full capacity"
(healthy); load 8.0 = "twice as much work waiting in the queue as capacity" (crowded). On a single-core
machine, load 4.0 = "the system is drowning". Practical rule: **divide the load by `nproc`** — under
1 is comfortable, ~1 is completely full, well above 1 the queue is building.

**Second** — and this is the Linux-specific part: the **D-state is counted in load too.** That's why
load does **not** mean "CPU busy". Processes waiting on disk (D) can inflate load without using the CPU
at all.

## 4.4.2 High load + low CPU = I/O wait `[application]`

Now we can solve the table left over from Phase 3. When you see high load but idle CPU, the culprit is
not the CPU but **I/O**:

> **High load + low CPU usage → almost always I/O wait (D-state processes).**

The `wa` (I/O wait) percentage on `top`'s `%Cpu` line tells you this directly. If `wa` is high, the CPU
is "sitting idle waiting on the disk" — there is work, but it isn't progressing **because of the disk**.

> **🔧 See it on your machine** 🟢 — decompose the load
>
> ```
> $ uptime
>  03:20:11 up 5 days,  load average: 9.40, 8.10, 6.50
> $ nproc
> 4
> $ top -bn1 | grep '%Cpu'
> %Cpu(s):  6.0 us,  3.0 sy,  0.0 ni, 11.0 id, 79.0 wa, ...
> $ ps -eo stat,comm | grep '^D'          # find D-state processes
> D    postgres
> D    kworker/u8:2
> ```
>
> Load is 9.4, but the machine is 4-core — so ~2.3× capacity. Looking at the CPU, `us`+`sy` is only 9%,
> but `wa` is 79%. So the CPU is actually **idle**, and work is choked waiting on the disk. `ps ... grep
> '^D'` gives the processes locked on disk (here `postgres`). Decision: adding CPU (a bigger instance)
> will **not** fix this — a faster disk (IOPS) or reducing I/O is needed.

> **⚠️ Common misconception: "Load is 9, the CPU isn't enough, let's get more vCPUs."**
> A high load does **not** mean a CPU bottleneck. First look at `wa`: if it's high, the problem is on
> the disk, adding vCPUs is a waste of money and won't fix it. Always read load **together** with the
> `%Cpu` line: high `us`+`sy` → truly CPU-bound; high `wa` → I/O-bound.

---
---

# 4.5 I/O Intuition

## 4.5.1 What iowait tells you: the CPU's time "waiting on disk" `[concept]`

Let's make the `wa` (iowait) column explicit, because it's often misinterpreted. **iowait is the
percentage of time the CPU "has no other work to do **and** at least one process is waiting for disk
I/O".**

The subtlety to note: iowait is **not** a "CPU busy" measure — quite the opposite, the CPU is **idle**
at that time. It's just that the reason it's idle is not "there's no work" but "there's work but it
can't progress because of the disk". So iowait = an indicator of a **hidden disk bottleneck**.

Under this lies the concept of **blocking I/O**. When a process wants to read from a file, it makes a
`read()` syscall (Phase 0). Until the data comes from disk, the process **blocks** — it enters the D
state (Phase 3.2), releases the CPU and waits. If the disk is slow or saturated, this wait lengthens;
if many processes wait at once, load swells and the CPU sits in `wa`. (The alternative is **non-blocking
/ asynchronous I/O**: the process says "notify me when data is ready" and moves to other work without
waiting — this is the foundation of high-performance servers, e.g. nginx. In this phase, recognizing
the concept is enough.)

## 4.5.2 Disk saturation: the whole system is "slow" but the CPU is idle `[application]`

A disk, too, has a capacity: how many operations per second (IOPS) and how many MB (throughput) it can
carry. When this ceiling fills, the disk becomes **saturated** — every new I/O request queues up and
waits. The result: **every** process that touches the disk slows down, the whole system feels "slow",
but the CPU graph is idle. A classic and confusing table.

The tool that reveals this is `iostat`:

> **🔧 See it on your machine** 🟢 — measure disk saturation
>
> ```
> $ iostat -xz 1 3
> Device   r/s    w/s   rkB/s   wkB/s  await  %util
> nvme0n1  12.0  380.0   480.0  152000  45.20   99.3
> ```
>
> Two columns to read: **`%util`** — how much of the disk's time was spent busy; near 100% means the
> disk is saturated. **`await`** — the average completion time of an I/O request (ms); if this number
> climbs, the disk can't keep up with requests. Here `%util` is 99.3 and `await` is 45 ms — the disk is
> choked. Looking at `w/s` and `wkB/s`, there's heavy **writing** (maybe a log flood or a batch job).
> This table is the answer to the "CPU idle but the system is slow" riddle.

> **💡 Cloud connection:** On AWS, EBS disks' IOPS and throughput ceilings are set by the **type you
> buy** (gp3, io2, etc.). When a gp3 disk hits its IOPS ceiling, exactly this table appears: `%util`
> ~100%, `await` rises, the app is "slow" but the CPU metric is innocent. The fix is not CPU but
> increasing the disk's IOPS/throughput or fixing the I/O pattern. CloudWatch's `VolumeQueueLength` and
> `VolumeReadOps`/`WriteOps` metrics are the remote version of `iostat`.

---
---

# 4.6 Bottleneck Triage (From the OS Side)

## 4.6.1 The four bottleneck types and which tool reveals each `[application]`

This phase climbed here for a single practical reflex: when someone says "my app is slow", to narrow,
in **60 seconds**, which of the four bottleneck types it is. The four types are:

| Bottleneck type | Signature | First tool to reach for |
|---|---|---|
| **CPU-bound** | High load + high `us`+`sy`, low `wa` | `top` (%Cpu line), `mpstat` |
| **Memory-bound** | Low `available`, active swap `si/so`, OOM risk | `free -h`, `vmstat` (si/so) |
| **I/O-bound** | High load + low CPU + high `wa`, D-state | `iostat -xz` (%util, await) |
| **Network-bound** | CPU/disk idle but transfer slow, retransmits | `ss`, `iftop`, `sar -n` (Phase 7) |

The logic here: **each bottleneck has a different signature and a different tool.** If you use the
wrong tool, you make the wrong diagnosis — and in cloud, a wrong diagnosis means a wrong "get a bigger
instance" decision.

## 4.6.2 The 60-second triage order `[application]`

In practice the order followed is this — each step eliminates the next:

1. **`uptime`** — look at the load, divide by `nproc`. Comfortable or crowded? (Is there a problem, how
   much?)
2. **`top` (or `vmstat 1`)** — read the `%Cpu` line: is `us`+`sy` high (CPU-bound), or is `wa` high
   (I/O-bound)?
3. **`free -h`** — is `available` low? Is there `si/so` in `vmstat`? (Memory-bound / swap.)
4. **`iostat -xz 1`** — if `wa` is high: which disk, what are `%util` and `await`? (Confirm the I/O
   bottleneck.)
5. **`ss` / network tools** — if everything is idle but the app is slow, the bottleneck may be in the
   network or in the app's own logic (Phase 7).

> **🔧 See it on your machine** 🟢 — triage on one screen: `vmstat`
>
> ```
> $ vmstat 1 5
> procs -----------memory---------- ---swap-- -----io---- --system-- ------cpu-----
>  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs  us sy id wa st
>  1  0      0 512000  40000 3800000   0    0    12    40  520 1100  15  4 80  1  0
>  8  0      0 480000  40000 3810000   0    0     8    32 1200 3400  92  6  2  0  0
> ```
>
> `vmstat` summarizes the four bottlenecks in one line: `r` (run queue — CPU pressure), `si/so` (swap —
> memory pressure), `bi/bo` (block I/O), `wa` (I/O wait), `us/sy` (CPU usage). In the second sample row
> `r=8` and `us=92` → clearly **CPU-bound**; `si/so=0` and `wa=0` → memory and disk are innocent.
> Diagnosis: the CPU isn't enough, adding vCPUs (or optimizing the code) makes sense — because this time
> the signature is **truly** CPU.

> **🤔 Think 4.4** — Two `top` header lines from two different servers:
> **A:** `load average: 8.0` · `%Cpu: 95 us, 3 sy, 0 wa`
> **B:** `load average: 8.0` · `%Cpu: 4 us, 2 sy, 82 wa`
> The load is the same on both (8.0). Which is CPU-bound, which is I/O-bound? What would the right
> "fix" be for each, and for which would getting a bigger/faster disk be pointless?
>
> *(Answer: at the end of the phase)*

---
---

# 4.7 When This Phase Breaks — Failure Signatures

This phase opened up what's under the "slow" and "out of memory" complaints. The table below ties the
symptom you see in the field to the right mechanism — for diagnosis instead of panic.

| Symptom | Likely mechanism | Where explained | Look first at |
|---|---|---|---|
| `free` shows RAM 90% full, panic | Page cache (reclaimable) | 4.2.1 | The `available` column — is it low? |
| App died out of nowhere | OOM Killer | 4.3.2 | `dmesg -T \| grep -i oom` |
| System very slow, CPU idle | Swap thrashing or I/O saturation | 4.3.1, 4.5.2 | `vmstat` si/so and `iostat` %util |
| High load but CPU 10% | D-state / I/O wait | 4.4.2 | `top` %Cpu line `wa`; `ps grep '^D'` |
| "MySQL eats 2 GB of RAM" (wrong) | Confusing VSZ with RSS | 4.1.2 | The `RES`/`RSS` column, not VSZ |
| 8 workers' RSS sum > reality | Shared memory / CoW | 4.1.3 | `smem` / PSS |
| We upsized the instance but it's still slow | Wrong bottleneck diagnosis | 4.6 | Run the triage order from the top |
| gp3 disk is "slow", CPU idle | EBS IOPS ceiling saturated | 4.5.2 | `iostat await`/`%util`; EBS metrics |

> **The lesson from this table:** All of this phase's failures come from **the same root fallacy** —
> reading a metric on the wrong axis. "RAM is full" read without looking at `available` is panic; "load
> is high" read without looking at `wa` is a wrong CPU decision; "lots of memory" read from VSZ is an
> overestimate. The reflex is one: **before reacting to a metric, look at the second metric that
> confirms it.** `used` → `available`, load → `%Cpu`, VSZ → RSS, "slow" → `vmstat`/`iostat`.

---
---

# Phase 4 — Answers to the think questions

## Answer 4.1 — Why 8 workers × 180 MB isn't 1.44 GB

**Question:** `ps` shows 8 `php-fpm` workers, each with an RSS of 180 MB. Why might "8 × 180 = 1.44 GB"
be wrong?

Because RSS **counts shared pages toward each process's own share** (4.1.3). Most of the 8 workers
share the same code: the same PHP interpreter, the same shared libraries (`libc`, PHP extensions), and,
as is common in `php-fpm`, the initial pages shared via Copy-on-Write because they were `fork`ed from a
master process. These shared pages sit as **a single copy** in physical RAM, but are counted in full in
each worker's RSS.

So of each worker's 180 MB, perhaps 120 MB is common (a single physical copy) and 60 MB is truly
private. The real physical usage is roughly: `120 MB (once) + 8 × 60 MB (private) ≈ 600 MB` — not 1.44
GB. For a correct measurement you read not the sum of RSS but **PSS** (Proportional Set Size, which
divides the shared portion by the number of processes), or `used` in `free` at the system level. Rule:
**don't find total RAM by summing RSS; that always overestimates.**

**Related section:** 4.1.3, 4.1.2 · **Continues in:** 4.2 (system-level measurement), Phase 6 (/proc)

---

## Answer 4.2 — Is the "88% RAM" alarm right?

**Question:** A panel raises a "RAM 88%" alarm. `free -h`: `used` 2 GB, `buff/cache` 12 GB, `available`
11 GB, total 16 GB. Is the alarm right?

**No, the alarm is misreading.** The 88% probably comes from the formula `(total − free) / total` —
that is, it counts page cache as "in use". But look at the real table: what processes actually hold
(`used`) is only 2 GB, the remaining 12 GB is **page cache** (reclaimable), and the kernel's own
estimate `available` = 11 GB (~69% of total). So the system is **not** under pressure; if you start a
new app, 11 GB can be given without swapping (4.2.1, 4.2.2).

The correct sentence: "RAM looks 88% full, but 12 GB of that is reclaimable page cache; real available
memory is 11 GB, the system is relaxed." **The situation that should actually raise an alarm:** when
`available` drops to a small percentage of total (say, below 10%) — because real memory pressure is
measured by `available`, not by `used`/`free`. The alarm's threshold should be built on `available`,
not `used`.

**Related section:** 4.2.1, 4.2.2 · **Continues in:** 4.3 (when available runs out), Phase 11 (metric
alarms)

---

## Answer 4.3 — The 3 a.m. OOM: is the database to blame?

**Question:** At 3 a.m. the `postgres` process holding the most RAM died via OOM. (1) Did the database
break, or did something else exhaust the RAM? (2) How do you keep it from being the first to die?

**(1) The process the OOM Killer kills is usually not the culprit — just the largest.** The OOM Killer
chooses by `oom_score`, and this score roughly points at the process that would "reclaim the most RAM"
(4.3.2). `postgres`, as a database, naturally holds the most RAM, so even when **something else**
exhausts the RAM, the victim is `postgres`. To find the real culprit: look at the **slope over time** of
`available` up to the OOM moment (a memory leak is a trend, 4.3.2 cloud box). For example, a cron/batch
job running at 3 a.m. may have slowly consumed RAM and driven `available` to zero, with the OOM Killer
landing the final blow on `postgres`. The process list at the OOM moment in `dmesg`, and whichever
process's RSS was growing then, reveal this.

**(2) Keeping `postgres` from being the first to die:** you pull its OOM score **down** with
`oom_score_adj` (e.g. `-900`), so the kernel kills other candidates first. If it runs as a systemd
service, you set `OOMScoreAdjust=-900` in the unit file. But this only shifts the **symptom** — the
real fix is finding and fixing the process that consumes the RAM, or sizing the memory correctly. "The
most critical process dies last" is a seatbelt, not the removal of the crash's cause.

**Related section:** 4.3.2, 4.3.1 · **Continues in:** Phase 5 (systemd unit, OOMScoreAdjust), Phase 11
(trend monitoring)

---

## Answer 4.4 — Same load, two different bottlenecks

**Question:** A: `load 8.0`, `%Cpu: 95 us`. B: `load 8.0`, `%Cpu: 82 wa`. Which is CPU-bound, which is
I/O-bound?

The load is the same on both (8.0), but **load alone doesn't tell you the bottleneck type** — you must
read it together with the `%Cpu` line (4.4.2).

**A = CPU-bound.** `us` 95%: nearly all of the CPU is spent running user-space code. Load 8 comes from
processes actually running/queued. Fix: more/faster CPU (add vCPUs) or optimize / parallelize the code.
Getting a faster disk here would be **completely pointless** — the disk isn't the bottleneck.

**B = I/O-bound.** `wa` 82%: the CPU is actually **idle**, the work is choked waiting on the disk
(4.5.1). Load 8 comes from D-state processes waiting on disk. Fix: a faster disk (raise
IOPS/throughput), fix the I/O pattern (batch, cache), or reduce load on the disk. Adding vCPUs here
would be **pointless** — the CPU is already sitting idle.

The essence of the lesson: two servers with the same load but **diametrically opposite fixes.** Never
read load apart from the `%Cpu` line — high `us` → CPU, high `wa` → disk.

**Related section:** 4.4.2, 4.6.1 · **Continues in:** 4.6 (triage), Phase 11 (troubleshooting)

---
---

# Phase 4 — Frequently asked questions

### Q1. Why is the `free` column in `free` always small? Is it a problem?

No, on the contrary it's a sign of health. Linux uses free RAM as page cache (4.2.1) — "free RAM is
wasted RAM". So on a system that has run for a while, it is **normal** for the `free` column to be
small and `buff/cache` to be large. The column to watch is not `free` but `available`. A large `free`
column shows that a system either just booted or has done no work — nothing to brag about.

### Q2. What's the difference between RSS and PSS, and when do I use which?

**RSS** is the total of a process's pages resident in RAM, but it counts shared pages in **full**
(4.1.3). **PSS** (Proportional Set Size) **divides** shared pages by the number of processes sharing
them — so each process gets "its fair share" of the shared part. For roughly how much a single process
holds, RSS is enough; but if you're trying to produce **the sum of many similar processes** (workers,
forks), RSS overestimates and PSS is closer to reality. `smem` or `/proc/<PID>/smaps_rollup` gives PSS.

### Q3. Should I turn off swap?

It depends. On production servers (especially Kubernetes nodes) swap is usually **turned off**: swap
hides the reality of "out of memory" by turning it into a slow performance problem (4.3.1 cloud box). A
swap-less system gives a fast, clear OOM when memory runs out — diagnosis is easy. But on a
desktop/dev machine, a small swap can smooth over rare memory spikes. Rule: if you want "fast and clear
failure" in production, keep swap small or off; never chronically rely on swap to under-provision RAM.

### Q4. Why is `VIRT` (VSZ) so large in `top`, should I worry?

No. `VIRT`/`VSZ` is the **virtual** space the process reserved — it costs no physical RAM until touched
(4.1.2). JVM, Go, databases reserve tens of GB of virtual space at startup; this is nearly free. Your
worry metric is always `RES`/`RSS` (what's actually resident in RAM). Don't panic at "VIRT 20 GB!";
look at `RES`.

### Q5. The OOM Killer killed a process but the machine had free RAM, how?

Two common causes. **(1) cgroup limit:** if the process is inside a cgroup (container, systemd
service), an in-group OOM happens when **that group's own** memory limit fills, even if the host has
plenty of RAM (Phase 3.6.2). The phrase "Memory cgroup out of memory" in `dmesg` gives this away. **(2)
An instantaneous spike:** RAM was free at the moment you measured, but seconds earlier a process may
have asked for all the RAM in a sudden spike and triggered the OOM; after it died, RAM appears free. In
both cases `dmesg -T | grep -i oom` shows the real moment.

### Q6. `iowait` is high but the disk looks idle in `iostat`, a contradiction?

Not always. `iowait` is the CPU's "idle but at least one process is waiting on I/O" time (4.5.1). The
awaited I/O may not be a local disk: **NFS/network file system**, **a network-attached disk like EBS**,
or many small concurrent requests. Local `iostat` may show the local disk idle and miss that the wait
is on network storage. Also, on a many-core machine the `iowait` percentage can mislead because it's
averaged across cores. Rule: if `iowait` is high, find the D-state processes (`ps ... grep '^D'`) and
look at **what** they are waiting on (local disk or network storage).

### Q7. What exactly are the `r` and `b` columns in `vmstat 1`?

The two numbers under `vmstat`'s `procs` header are the essence of bottleneck triage: **`r`** = the
count of running + ready-to-run (run queue) processes — shows CPU pressure (if it exceeds the core
count, the CPU is crowded). **`b`** = the count of processes waiting in uninterruptible sleep (blocked,
D-state) — shows I/O pressure. So in one line: `r` high → look at the CPU; `b` high → look at the disk.
It's like an instantaneous, decomposed version of load average.

---
---

# Phase 4 — Test yourself

Answer the 18 questions below. The answer key and scoring are right underneath. The goal is not
memorization but being able to answer "what is eating the resources" from the model.

## Section A — Basics (1–6)

**1.** In `ps`/`top` output, what is the difference between `VSZ` (VIRT) and `RSS` (RES), and which is
"the RAM actually used"?

**2.** What does the `buff/cache` column in `free -h` show, and is this memory "in use" or "available"?

**3.** In `free -h`, which column do you look at to decide how full RAM is?

**4.** What is swap and why is it much slower than RAM?

**5.** When does the OOM Killer kick in, and roughly whom does it choose to kill?

**6.** Of two machines both with load average 4.0, one has 2 cores, the other 8. Which is under more
pressure?

## Section B — Mechanism (7–12)

**7.** A process does `malloc(1GB)` but while VSZ grows by 1 GB, RSS barely grows. Why? What is this
concept called?

**8.** Each of 10 workers has an RSS of 200 MB. Why might the real physical RAM usage be less than 2
GB?

**9.** On a healthy system that looks "90% RAM full", what is most of that used memory, and what happens
if a new app asks for RAM?

**10.** Load average is high (e.g. 9) but in `top` `us`+`sy` is low and `wa` is high. What state are the
processes in and what are they waiting for?

**11.** What exactly does `iowait` (`wa`) measure — is the CPU busy or idle?

**12.** What does the `swappiness` value tune; what do high and low values mean?

## Section C — Application and reasoning (13–18)

**13.** A monitoring panel raises a red "RAM 95%" alarm. On the server, `free -h`: used 2 GB, buff/cache
13 GB, available 12 GB. Is the alarm right, what do you say in one sentence?

**14.** Both of two servers have load 8.0. A: `%Cpu 95 us`. B: `%Cpu 82 wa`. Which is CPU-bound, which
is I/O-bound, and for which is adding vCPUs pointless?

**15.** At 3 a.m. the `postgres` process holding the most RAM died via OOM. What are the two things you
should ask first, and how do you look for the real culprit?

**16.** The complaint "my app is slow" comes in. Which commands, in which order, do you run to narrow
the bottleneck type in 60 seconds?

**17.** In `iostat -xz 1` output a disk's `%util` is 99 and `await` is 60 ms. What does this tell you,
and is the fix adding CPU?

**18.** A monitoring tool says "MemoryUtilization 98%" for a container but the app runs fine. Why might
this be misleading, and where do you read the real limit?

---

## Answer key

**1.** `VSZ` = the total **virtual** space the process reserved (no direct relation to physical RAM);
`RSS` = those virtual pages **actually resident in physical RAM**. "The RAM actually used" = **RSS**. ·
*4.1.2*

**2.** `buff/cache` = page cache + buffers, i.e. the RAM copy of what was read from disk. It looks "in
use" and is **reclaimable** — if a process asks for RAM, the kernel releases it instantly. · *4.2.1*

**3.** The **`available`** column — the kernel's estimate of "how much RAM I can give a new app without
swapping". Not `used`/`free`; this measures real pressure. · *4.2.2*

**4.** Swap is the **disk** area where least-used memory pages are offloaded when RAM overflows. Because
disk is thousands of times slower than RAM, if active memory starts going in and out of swap
(thrashing) the system is paralyzed. · *4.3.1*

**5.** When RAM (and swap) is completely exhausted and the kernel can't find a page to give. It kills
roughly the process with the highest **`oom_score`** — i.e. the one holding the most RAM — and reclaims
its memory. · *4.3.2*

**6.** The **2-core** one. Load is read against the core count: on 2 cores load 4.0 = 2× capacity
(crowded); on 8 cores load 4.0 = 50% usage (relaxed). Rule: load / `nproc`. · *4.4.1*

**7.** Because Linux does **lazy allocation**: `malloc` only reserves virtual space (VSZ grows), but the
kernel binds no physical RAM (RSS doesn't grow) until those pages are **touched**. A physical page is
bound only when written (page fault). · *4.1.2*

**8.** Because RSS counts **shared** pages (shared libraries, CoW pages from `fork`) in full toward each
process, but there is a single copy in physical RAM. For real usage don't sum RSS; look at PSS or `used`
in `free`. · *4.1.3*

**9.** Mostly **page cache** (a reclaimable copy of what was read from disk). If a new app asks for RAM,
the kernel instantly releases part of the page cache and gives it — because it's already on disk. ·
*4.2.1*

**10.** Most likely in **D (uninterruptible sleep)**, waiting for **disk/network I/O**. These D-state
processes inflate the high load; the CPU is actually idle (`wa` high). · *4.4.2*

**11.** `iowait` is the percentage of time the CPU is **idle** **but** at least one process is waiting
on I/O. The CPU is not busy — it's idle but work isn't progressing because of the disk. An indicator of
a "hidden disk bottleneck". · *4.5.1*

**12.** `swappiness` (0–100) tunes how eager the kernel is to swap. High = swaps process memory out
early to protect page cache; low = keep in RAM as much as possible, swap as a last resort. Usually low
on servers. · *4.3.1*

**13.** No, the alarm is misreading. Real `used` is only 2 GB; 13 GB is **page cache** (reclaimable);
`available` is 12 GB → the system is relaxed. Correct: "looks 95% full but most is reclaimable cache,
real available is 12 GB." The alarm should be built on `available`. · *4.2.2*

**14.** **A = CPU-bound** (`us` 95%, CPU full); **B = I/O-bound** (`wa` 82%, CPU idle, waiting on disk).
**Adding vCPUs is pointless for B** — the CPU is already idle; it needs a fast disk. For A, getting a
disk is pointless. · *4.4.2, 4.6.1*

**15.** (1) "Is the culprit really postgres, or did something else exhaust the RAM and make it the
largest victim?" (2) "With what slope did `available` drop up to the OOM moment?" For the real culprit,
find the process whose RSS was growing over time / the job running at that hour (`dmesg` + trend).
Protection: `OOMScoreAdjust=-900` on `postgres`. · *4.3.2*

**16.** (1) `uptime` — load / `nproc`; (2) `top`/`vmstat 1` — is `us+sy` or `wa` high; (3) `free -h` — is
`available` low, is there `si/so`; (4) `iostat -xz 1` — which disk `%util`/`await`; (5) network tools.
Each step eliminates one bottleneck type. · *4.6.2*

**17.** The disk is **saturated** (%util ~100) and every I/O waits 60 ms on average (await high) — the
disk can't keep up with requests. The fix is **not** CPU; it's a faster disk (IOPS/throughput), fixing
the I/O pattern, or reducing load. · *4.5.2*

**18.** Because memory metrics inside a container often report the host value or count cache as "in
use"; also `free`/`top` may show the host inside a container (Phase 3.6). Read the real limit from the
cgroup: in cgroup v2, `/sys/fs/cgroup/memory.max` and `memory.current`. · *4.2.2, Phase 3.6.4*

---

## Scoring

| Number correct | Assessment |
|---|---|
| 16–18 | You answer "what is eating the resources" from the model. Move on to Checkpoint Quiz 2. |
| 12–15 | Solid. Take a pass through the sections of the questions you missed. |
| 8–11 | The basics are in place but you're confusing the metrics. Use the table below. |
| 0–7 | Repeat the phase from the start, running the 🔧 boxes (especially 4.2, 4.4, 4.6) on your machine. |

**Which question you missed → which section to return to:**

| Question you missed | Return to |
|---|---|
| 1, 7 | 4.1.2 (RSS vs VSZ, lazy allocation) |
| 8, 18 | 4.1.3 (shared memory, summing RSS) |
| 2, 3, 9, 13 | 4.2 (page cache, the available column) |
| 4, 5, 12, 15 | 4.3 (swap, swappiness, OOM Killer) |
| 6, 10 | 4.4 (load average, core count, D-state) |
| 11, 17 | 4.5 (iowait, disk saturation, iostat) |
| 14, 16 | 4.6 (bottleneck triage, the four types) |

---
---

# Phase 4 — Closing and Bridge to Phase 5

## What you carry from this phase

Phase 3 gave you the "what is the system doing" question on the process axis. Phase 4 carried the same
question onto the **resource** axis: memory and I/O. Now you don't react to a metric on its own — you
look at the second number that confirms each one. You carry four reflexes with you:

1. **Memory reality is in RSS, in `available`.** `VSZ` is promised, `RSS` is used; don't sum RSS
   (shared memory). "RAM full" is measured by `available`, not `used`/`free` — page cache is
   reclaimable.
2. **When memory runs out → swap → OOM.** Swap is slow and hides reality; the OOM Killer kills the
   largest process (not the culprit). `dmesg | grep oom` is the first place to look.
3. **Read load together with `%Cpu`.** Load is divided by the core count; high `us` → CPU-bound, high
   `wa` → I/O-bound. Same load, opposite fixes.
4. **Bottleneck triage.** Four types (CPU/memory/I/O/network), each with a distinct signature and tool.
   The order `uptime → top → free → iostat` narrows the type in 60 seconds.

## Where Phase 5 connects to this

Phases 3 and 4 together fully covered "what is a running machine doing, what is eating its resources" —
these two are the subject of **Checkpoint Quiz 2**. The next big question is: **how did this machine
become a running system from the start?** Phase 5 (Boot, Init and systemd) opens the machine's life
cycle and gathers the threads this phase left:

| What you learned in Phase 4 | What Phase 5 adds on top |
|---|---|
| PID 1 = systemd, the root of the tree (from Phase 3) | systemd as more than a "starter": units, services, targets |
| A process is protected in OOM with `OOMScoreAdjust` | Resource and OOM settings in a systemd unit file |
| How a service is kept "running" | The systemd service life cycle, restart policies |
| `dmesg` shows kernel events | `journalctl` — systemd's central log book |
| Starting/stopping a process correctly | `systemctl start/stop/enable`, boot order |

> **Phase 4 output — before moving on, ask yourself:**
> A developer wrote "the prod server is very slow, RAM is 97% full, let's urgently get a bigger
> instance". You logged into the server:
>
> ```
> $ free -h
>                total   used   free   buff/cache   available
> Mem:            16Gi   2.4Gi  0.3Gi     13Gi         12Gi
> $ uptime
>  ... load average: 11.20, 10.80, 9.90     # (4-vCPU machine)
> $ top -bn1 | grep %Cpu
> %Cpu(s):  5.0 us, 2.0 sy, 0.0 ni, 8.0 id, 84.0 wa, ...
> ```
>
> 1. Is "RAM is 97% full" a valid justification? What do you write back to the developer in one
>    sentence?
> 2. What is the real bottleneck — CPU, memory, or disk? Which two numbers told you?
> 3. Will "a bigger instance" fix this problem? If not, what is the right step?
>
> If you can answer these three without hesitation, memory and I/O intuition have settled — you're
> ready for Checkpoint Quiz 2.
>
> **Lab 4 idea:** With `free -h` open, run `cat a_big_file > /dev/null` and watch how `buff/cache` and
> `available` change (`used` doesn't!). With `vmstat 1`, while copying a file, watch the `bi/bo` (block
> I/O) and `wa` columns live. If you have a machine, apply artificial memory pressure with `stress-ng
> --vm 2 --vm-bytes 1G --timeout 20s` and watch `si/so` (swap) wake up. Always compare your `uptime`
> load against `nproc`.

---

> **Navigation:** [◀ Phase 3 — Processes and Resources](Phase_3_Processes_and_Resources.md) · **Phase 4** · [Checkpoint Quiz 2 ▶](Checkpoint_Quiz_2.md)


