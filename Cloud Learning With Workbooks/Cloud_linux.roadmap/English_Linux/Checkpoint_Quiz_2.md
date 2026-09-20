# Checkpoint Quiz 2 — Phases 3–4: Processes, Memory and Performance

> **Navigation:** [◀ Phase 4 — Memory, I/O and Performance](Phase_4_Memory_IO_Performance.md) · **Checkpoint Quiz 2** · [Phase 5 — Boot, Init and systemd ▶](Phase_5_Boot_Init_systemd.md)

---

## What does this quiz measure?

The "Test yourself" section at the end of each phase probes a single phase. A checkpoint quiz measures
something **different**: *"Can you connect these two phases to each other?"*

Phase 3 gave you the **atom of a running system**: the process — how it is born, what state it is in,
how it dies, how its resources are capped. Phase 4 looked at the same system through the lens of
**resources**: memory, I/O, bottlenecks. In a real incident these two never stand apart. When someone
says "the server is slow," you use both phases at once: what **state** a process is in (Phase 3 —
R/S/D/Z) and which **resource** bottleneck that state points to (Phase 4 — CPU/memory/I/O). D-state is
a Phase 3 concept; but "high load + low CPU" is a Phase 4 diagnosis — and the two are two faces of the
**same** event.

**How to work through it:**

- Solve on paper; don't move to the next question before you've written your answer.
- The answer key tells you which **phase intersection** each question sits at — a missed question points
  not to a phase but to the **bridge** between two phases.
- 21 questions: Section A (connection reasoning, 1–9), Section B (scenario: a "the app is slow"
  incident, 10–15), Section C (reading commands and output, 16–21).
- Target time: ~1 hour. But time doesn't matter; what matters is being able to justify *why* each
  answer is what it is, in one sentence.

> **🤔 Before you start:** Complete this sentence in your own words: *"Load average is 12, but `top`
> shows the CPU 90% idle — how can a machine be both 'overloaded' and 'idle'?"* This single question
> contains both phases. Jot down your answer; we'll get to it in Question 3.

---

# Section A — Connection reasoning (1–9)

Each question here asks you to combine at least two phases (mostly Phase 3 × Phase 4). Answer briefly
but with reasoning.

**1.** A process is stuck in `D` state (Phase 3). This single fact affects which two Phase 4 metrics at
once — load average and `iowait`? Why does `kill -9` fail to rescue this process (Phase 3), and why is
the disk the right place to look (Phase 4)?

**2.** In Phase 3 we said "exceeding a cgroup memory limit causes an OOM." In Phase 4 we said "the OOM
Killer kills the process with the highest `oom_score`." When a container's **own** limit fills up, how
does it differ from a system-wide OOM, and what phrase in `dmesg` distinguishes them?

**3.** Load average is 12, but `top` shows the CPU 90% idle (`id`). How can the machine be both
"overloaded" and "idle"? Answer via the definition of load (Phase 3: R + D) and the `wa` metric
(Phase 4).

**4.** In `ps aux`, a process's `%MEM` column (based on RSS, Phase 3) looks high, but there is no real
memory pressure on the system. Which two Phase 4 concepts (RSS vs VSZ / shared memory / page cache) can
explain this contradiction? Give at least two possible reasons.

**5.** In Phase 3 we saw signals (SIGTERM/SIGKILL). When a process is killed by the OOM Killer (Phase
4), which signal does the kernel send it, and why can't this signal be caught? Can the process do a
"graceful shutdown"?

**6.** In `top` there is a JVM process with `VIRT` (VSZ) 20 GB, `RES` (RSS) 500 MB (Phase 3 columns).
This machine has 8 GB of RAM total. How does a process that has "reserved" 20 GB run on an 8 GB
machine? Which Phase 4 mechanism (virtual memory / lazy allocation) makes this possible?

**7.** In Phase 3 we saw the process/thread distinction and vCPU count. An 8-thread application runs on
a 4-vCPU machine and load is 8. Is this load "healthy full capacity" or "overload" — which single
number (Phase 4) do you divide by to settle it?

**8.** On a system thrashing in swap (Phase 4), in which state (Phase 3: R/S/D) do processes pile up,
and why does this inflate load but leave the CPU idle? Connect both phases' concepts in one sentence.

**9.** In Phase 3 we saw CPU priority via `nice`/`renice` and CPU quota via cgroups. If a process is
CPU-bound (Phase 4), does "slowing it down" with `renice` help; if it is I/O-bound, why does `renice`
not help? Establish the relationship between bottleneck type (Phase 4) and priority tool (Phase 3).

---

# Section B — Scenario: a "the app is slow" incident (10–15)

> On a production Ubuntu server (4 vCPU, 16 GB RAM) a web application is running. At 03:00 the on-call
> texts you: "the site is slow, sometimes returns 502." You SSH in. The six questions below are the
> diagnosis you run on this server, in order.

**10.** Your first command is `uptime`: `load average: 11.5, 10.2, 7.8`. What does this single line tell
you, and what does it **not**? What should your next command be and why — you can't yet say "CPU is
insufficient, let's scale the instance"; which metric do you need to see?

**11.** You open `top`: `%Cpu: 5 us, 3 sy, 0 ni, 8 id, 84 wa`. Load is 11.5, but 84% of the CPU is
`wa`. Which of the four bottleneck types (CPU/memory/I/O/network) is it, and how do you know? Would a
"bigger instance" (more vCPU) solve this?

**12.** You saw that `wa` is high. With which two commands (one for process state — Phase 3, one for
disk — Phase 4) do you narrow the question "which process is waiting on the disk and is the disk really
saturated"? Which exact column do you look at in each command?

**13.** `ps -eo stat,comm | grep '^D'` gave you two `D` processes: the application itself and
`kworker`. In `iostat -xz 1` the `nvme0n1` disk shows `%util 99, await 80ms`. Write the diagnosis in
one sentence. How could the 502s (recall from Phase 3: graceful shutdown / load balancer) be related
to this picture?

**14.** Meanwhile you also ran `free -h`: `used 3.2Gi, buff/cache 12Gi, available 12Gi`. Someone says
"look, RAM is nearly full, that's why it's slow." Is this a correct diagnosis? Refute it in one
sentence (Phase 4), and which column proves that memory is **not** the bottleneck on this server?

**15.** Root cause: a backup job running at 03:00 has saturated the disk. For a permanent fix, consider
three options and justify each with a bottleneck-type/tool logic: (a) lowering the backup's priority
with `ionice`/`nice`, (b) a higher-IOPS disk (gp3 → io2), (c) scaling the instance up in vCPU. Which
one does **not** solve this **particular** bottleneck and why?

---

# Section C — Reading commands and output (16–21)

For each output below, read the relevant lines and answer.

**16.** `vmstat 1` output:

```
 r  b   swpd   free   buff  cache   si   so   bi    bo   us sy id wa
 9  0      0 480000  40000 3800000   0    0   16    48   94  4  2  0
 8  1      0 470000  40000 3810000   0    0    8    40   91  6  3  0
```

Reading the `r`, `b`, `si/so`, `wa`, `us` columns, name the bottleneck type. Is this **CPU-bound** or
I/O-bound, and what does `si/so` being zero rule out?

**17.** `vmstat 1` from a second server:

```
 r  b   swpd   free   buff  cache   si   so   bi     bo    us sy id wa
 1  6  980000  90000  12000  140000  520  610  8200   240   4  5 10 81
```

How is this output different from Question 16's? Read together, which **two** bottlenecks do the
`si/so`, `b`, `wa`, `free`, `buff/cache` columns show at the same time (hint: both memory and I/O)?
What is the likely root cause?

**18.** `dmesg -T | tail` output:

```
[Tue 03:14:22] Out of memory: Killed process 2481 (python3) total-vm:8123400kB,
  anon-rss:7981200kB, file-rss:0kB, oom_score_adj:0
```

Which Phase 4 mechanism does this line show? Is `python3` really "guilty," or just the largest — how do
you decide? Why does it matter that the ~8 GB `anon-rss` is "anonymous" (vs page cache)?

**19.** `ps -eo pid,comm,vsz,rss --sort=-rss | head -3`:

```
   PID COMMAND        VSZ      RSS
  1102 java       20918000  512000
   890 postgres    2400000  410000
```

`java`'s VSZ is ~20 GB, RSS ~512 MB. What do you say, in one sentence, to someone who claims "Java is
eating 20 GB of RAM"? Which column gives the real RAM usage, and what is the large VSZ a result of?

**20.** `top` header block:

```
top - 03:20:01 up 5 days,  load average: 0.80, 0.75, 0.70
Tasks: 180 total, 1 running, 176 sleeping, 0 stopped, 3 zombie
%Cpu(s): 12.0 us, 3.0 sy, 0.0 ni, 84.0 id, 1.0 wa
```

Is this machine healthy? What do load (0.80 on 4 vCPU), `wa` (1.0) and `3 zombie` each tell you? Is `3
zombie` a reason to alarm — when would it be (Phase 3)?

**21.** What does the following command chain do, and what does its output reveal in a "slowness"
incident?

```
$ ps -eo pid,ppid,stat,rss,comm --sort=-rss | awk '$3 ~ /^D/'
```

Why is filtering `stat` values that start with `D` the Phase 3 side of an "I/O bottleneck" diagnosis
(Phase 4)? Which phases' tools does this chain combine?

---

## Answer Key

Each answer ends with the **phase intersection** the question sits at.

**1.** A process in `D` state both inflates **load average** (Linux counts R + D toward load) and raises
`iowait` (CPU idle but the process is waiting on I/O). `kill -9` doesn't rescue it because the process
is waiting on I/O inside the kernel and a signal is delivered only once it returns to user space (Phase
3.2.3); the right place to look is the disk itself (`iostat`, Phase 4.5) — the process isn't locked,
the storage under it is slow/saturated. · *Phase 3.2 × Phase 4.4/4.5*

**2.** When a container's own cgroup memory limit fills, a **within-group** OOM happens: even if the
host has plenty of RAM, the kernel kills a process inside the group because the group exceeded its own
quota. A system-wide OOM happens when the machine's **entire** RAM (+swap) is exhausted. The `dmesg`
distinction: a group OOM contains the phrase "Memory cgroup out of memory"; a system OOM does not. ·
*Phase 3.6 × Phase 4.3*

**3.** Not a contradiction, because load does **not** mean "CPU busy." Load = the number of processes
wanting to run (R) + those in uninterruptible sleep waiting on I/O (D) (Phase 3.2.1). If the CPU is 90%
idle but load is 12, most processes are in D state — they are waiting on the **disk**, not the CPU
(`wa` is high, Phase 4.4.2). So the machine's "work queue is full" but "work isn't progressing because
of the disk." · *Phase 3.2 × Phase 4.4*

**4.** At least two reasons: (a) `%MEM`/RSS counts **shared** pages (shared libraries, CoW) in full for
every process; real physical usage may be less (Phase 4.1.3). (b) The apparent "full" RAM may actually
be page cache — reclaimable, no pressure (Phase 4.2). (c) Even if RSS is real, the system is relaxed if
`available` is high. In short: one process's `%MEM` doesn't show system pressure; look at `available`.
· *Phase 3.7 × Phase 4.1/4.2*

**5.** In the OOM Killer the kernel sends the process **SIGKILL** (9). This signal can't be caught
(Phase 3.4.1) — which is exactly why in an OOM a process **cannot** do a "graceful shutdown"; it dies
instantly without cleanup. OOM is an emergency memory-reclaim operation; the kernel isn't gentle
because the system is already in a memory dead-end. · *Phase 3.4 × Phase 4.3*

**6.** VSZ is the **virtual** space a process has reserved; it has no direct link to physical RAM (Phase
4.1.2). The JVM reserves a huge virtual space at startup, but thanks to **lazy allocation** not a single
physical page is spent until that space is touched — RSS stays 500 MB. The 20 GB is only "promised"
space; running on an 8 GB machine is normal because what it actually holds is 500 MB. · *Phase 3.7 ×
Phase 4.1*

**7.** "Healthy full capacity." Load is read by dividing by the **core count** (Phase 4.4.1): load 8 on
4 vCPU → roughly 2× capacity, so tight — but whether it's "overload" depends on the `%Cpu` line. If
`us` is high it's genuinely CPU-bound (the 8-thread app may have filled 4 cores and queued); if `wa` is
high the source of load is I/O. The settling number: load / `nproc` = 8/4 = 2. · *Phase 3.3 × Phase 4.4*

**8.** In thrashing, processes pile up in **D** state: as active memory pages swap in and out, processes
wait on disk (swap) I/O (Phase 4.3.1). Since D-state counts toward load, load inflates (Phase 3.2.1);
but because the work is on the disk not the CPU, the CPU stays idle (`wa` high). One sentence: *swap I/O
drops processes into D, and D both inflates load and leaves the CPU idle.* · *Phase 3.2 × Phase 4.3*

**9.** `renice` only adjusts queueing priority for the **CPU** (Phase 3.5.2). Pushing a CPU-bound
process to the background with `renice` works — it yields the CPU to others. But an I/O-bound process
isn't using the CPU anyway (it's waiting on disk); `renice` doesn't change its disk behavior, so it
doesn't help. Disk priority needs a separate tool (`ionice`). Lesson: pick the tool by **bottleneck
type** — CPU priority targets a CPU bottleneck, I/O priority an I/O bottleneck. · *Phase 3.5 × Phase
4.5/4.6*

**10.** `uptime` tells you load is high (11.5, ~2.9× on 4 vCPU) — so there's pressure and it's rising
(1min > 5min > 15min). But what it does **not** tell you: whether that pressure is CPU, memory, or I/O.
Load alone doesn't give the bottleneck type (Phase 4.4). Your next command should be `top` (or `vmstat
1`): is `us`+`sy` high or is `wa` high on the `%Cpu` line — saying "let's scale the instance" before
seeing that risks a wrong diagnosis. · *Phase 4.4 × 4.6*

**11.** The bottleneck is **I/O-bound**. `wa` is 84%: most of the CPU sits idle waiting on disk I/O
(`us`+`sy` is only 8%). The source of load is D-state processes (Phase 4.4.2). A "bigger instance"
(more vCPU) does **not** solve this — the CPU is already idle; the problem is the disk. Right direction:
speed up the disk or reduce I/O. · *Phase 4.4 × 4.6*

**12.** (a) `ps -eo stat,comm | grep '^D'` — finds processes whose `STAT` starts with `D`
(uninterruptible sleep); these are the ones waiting on disk (Phase 3.2). (b) `iostat -xz 1` — on the
disk side you look at `%util` (saturation, near 100% means the disk is full) and `await` (wait per
request in ms) (Phase 4.5.2). The first answers "who is waiting," the second "is the disk really
saturated." · *Phase 3.2 × Phase 4.5*

**13.** Diagnosis: **the disk is saturated (I/O-bound bottleneck)** — `nvme0n1` at %util 99, await 80
ms, and two processes waiting on it in D state. Relation to the 502s: when the app blocks on disk I/O
it can't process requests in time, response times grow; the load balancer/proxy in front times out on
the unresponsive backend, says "couldn't reach it," and returns 502 (Phase 3.4.2 graceful/timeout
logic). So the 502s are a consequence of the disk stalling the app, not of CPU. · *Phase 3.2/3.4 ×
Phase 4.5*

**14.** No, a wrong diagnosis. `used` is only 3.2 GB; the 12 GB is **page cache** (reclaimable); most
importantly `available` is 12 GB — so there's no real memory pressure on the system (Phase 4.2.2). The
"RAM nearly full" look is the page-cache fallacy. The column that proves memory is **not** the
bottleneck: **`available`** (12/16 GB, relaxed). The bottleneck is not in memory, it's the disk. ·
*Phase 4.2*

**15.** (a) Lowering the backup to low I/O priority with `ionice`/`nice` — **works**, because the
bottleneck is I/O and `ionice` targets disk priority; the backup yields I/O to the app. (b) A
higher-IOPS disk — **works**, because it directly raises the I/O ceiling (Phase 4.5.2 cloud). (c)
Scaling up in vCPU — **doesn't solve it**, because the bottleneck is the disk not the CPU; the CPU is
already sitting idle in `wa`. (c) is the classic "scaling on the wrong axis" mistake and a waste of
money. · *Phase 3.5 × Phase 4.5/4.6*

**16.** **CPU-bound.** `r=9` (9 processes in the run queue on 4 vCPU, pressure on the CPU), `us≈91–94`
(the CPU is busy running user code), `wa=0` (no disk wait), `b=0/1` (no blocked processes), `si/so=0`
(no swap). `si/so` being zero rules out the **memory** bottleneck; `wa` being zero rules out **I/O**.
What's left is clean CPU: adding vCPU / optimizing the code makes sense. · *Phase 4.4/4.6*

**17.** This output is the opposite of 16: `us` is low (4), but `wa=81` (I/O wait) **and** `si/so` is
non-zero (520/610, active swap). So **two bottlenecks** at once: memory pressure (swap in/out active,
`free` very low at 90 MB) and the I/O saturation it triggers (`bi=8200`, `wa=81`, `b=6`). Likely root
cause: **memory exhausted → system went into swap → swap saturated disk I/O → thrashing.** The fix is
not CPU but RAM (or the process consuming memory). · *Phase 4.3 × 4.4/4.5*

**18.** The **OOM Killer** (Phase 4.3.2). `python3` may not really be guilty — the OOM Killer picks the
process with the highest `oom_score`, i.e. the one holding the most RAM; something else may have
consumed the RAM and made `python3` the biggest victim. To decide: look at the slope of `available` up
to the OOM moment and which process's RSS was growing at that time. It matters that the ~8 GB is
**anon-rss** (anonymous) because anonymous memory can't be reclaimed like page cache — it's real,
squeezing memory; if it were cache the kernel would drop it and no OOM would fire. · *Phase 4.3 ×
4.1/4.2*

**19.** "Java is eating 20 GB of RAM" is wrong — that's VSZ (reserved **virtual** space), not physical
RAM. The real RAM usage is given by the **RSS** column: ~512 MB. The large VSZ is a result of the JVM
reserving a huge virtual heap/arena at startup plus lazy allocation — untouched space spends no
physical RAM (Phase 4.1.2). Correct sentence: "Java is using ~512 MB RSS; the 20 GB is only reserved
virtual space." · *Phase 3.7 × Phase 4.1*

**20.** The machine is **healthy.** Load 0.80 (on 4 vCPU, ~20% usage — very relaxed); `wa` 1.0
(negligible disk wait); `us` 12 (light CPU work). `3 zombie` alone is **not** an alarm — zombies consume
no resources, they're just a small entry in the process table (Phase 3.2.2). It *would* be an alarm
**if** the number kept **rising** (meaning a parent isn't `wait`ing on its children) — then you'd hunt
the parent. A steady 3 zombies is harmless. · *Phase 3.2 × Phase 4.4*

**21.** The chain: `ps` lists all processes with PID/PPID/state/RSS/name columns sorted by RSS, and
`awk '$3 ~ /^D/'` filters those whose third column (`stat`) starts with `D` — i.e. it gives the
processes in **uninterruptible sleep waiting on disk**. This is the **Phase 3 side** of an "I/O
bottleneck" diagnosis (Phase 4.5): the bottleneck shows up as `wa` in the metric (Phase 4), but the
concrete processes **creating** it are in D state (Phase 3.2). The chain combines the tools of Phase 1
(pipe/awk text processing) + Phase 3 (process state) + Phase 4 (I/O diagnosis). · *Phase 1 × Phase 3 ×
Phase 4*

---

## Scoring

| Correct | Assessment |
|---|---|
| 19–21 | You've fused the process and resource models into a single diagnostic reflex. Move to Phase 5 with confidence. |
| 15–18 | Solid. Take one pass back to the **bridge** (not a single phase) that your missed questions point to. |
| 10–14 | You know the two phases separately, but the link between them is weak. Use the table below. |
| 0–9 | Review Phases 3 and 4, especially their "When this phase breaks" and "Closing" sections; a checkpoint isn't passed until these bridges settle. |

**Which question you missed → where to go back:**

| Missed question | Go back — this bridge is weak |
|---|---|
| 1, 3, 8, 21 | Phase 3.2 × Phase 4.4 — D-state, load average, iowait |
| 2, 5, 18 | Phase 3.4/3.6 × Phase 4.3 — signals, cgroup, OOM Killer |
| 4, 6, 19 | Phase 3.7 × Phase 4.1 — RSS/VSZ, virtual memory, shared memory |
| 10, 11, 16, 17 | Phase 4.4/4.6 — reading load with %Cpu, bottleneck triage |
| 12, 13, 15 | Phase 3.2 × Phase 4.5 — tying a D-state process to the disk, iostat |
| 7, 9 | Phase 3.3/3.5 × Phase 4.4/4.6 — vCPU, priority, tool by bottleneck type |
| 14, 20 | Phase 4.2 × Phase 3.2 — page cache/available, zombie fallacies |

---

## Closing — from here to Phase 5

Phases 3 and 4 together taught you to read the **live** picture of a server: what the system is doing
right now (process states) and what is eating its resources (memory, I/O, bottleneck type). You can now
split the single word "slow" into four separate diagnoses, and before reacting to one metric you look
at the second metric that confirms it. But there's one thing you still haven't asked: **how did this
machine come to be this running system in the first place?** You saw PID 1 = systemd at the root of the
process tree, but you haven't yet opened systemd itself — how it brings the machine from boot to a
running state, in what order it starts services.

Phase 5 connects right here: it takes Phase 3's "PID 1 = systemd, root of the tree" and puts "systemd
manages the machine's entire lifecycle" on top of it. In Phase 4 we protected a process from OOM with
`OOMScoreAdjust` — you'll see where that setting is written (a systemd unit file). `dmesg` showed you
kernel events; now with `journalctl` you'll read systemd's central logbook.

> **Before moving on:** If you can answer Question 11 without hesitation — why, with load 11.5 but `wa`
> 84%, the bottleneck is the disk and not the CPU, and why "scaling the instance" doesn't fix it —
> you're ready for Phase 5. If you can't, take a pass back to Phase 4's sections 4.4 (load) and 4.6
> (triage); Phase 5 will build *how the running system is assembled* on top of this diagnostic ground.

---

> **Navigation:** [◀ Phase 4 — Memory, I/O and Performance](Phase_4_Memory_IO_Performance.md) · **Checkpoint Quiz 2** · [Phase 5 — Boot, Init and systemd ▶](Phase_5_Boot_Init_systemd.md)
