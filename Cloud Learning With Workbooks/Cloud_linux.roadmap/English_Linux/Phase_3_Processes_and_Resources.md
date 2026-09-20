# Phase 3 — Processes and Resource Management: The Running System

> **Navigation:** [◀ Checkpoint Quiz 1](Checkpoint_Quiz_1.md) · **Phase 3** · [Phase 4 — Memory, I/O and Performance ▶](Phase_4_Memory_IO_Performance.md)

---

## Where we are coming from

In Phase 0 we set up the kernel's single door (the syscall). In Phase 1 we learned to reach that
door through the shell, and in Phase 2 we learned that behind every request there is **an identity**:
before the kernel carries out a syscall, it asks "who is asking for this?"

But so far we have only looked at a **static** picture: who owns a file, what its permissions are,
which user can run which command. There is one thing we never got to ask: **"what is this machine
doing right now?"**

Remember the last question of Phase 2 — why does nginx run processes as both `root` and `www-data`
at the same time? Answering it, we used one word but never dug into it: **process.** We know a
process has an identity (Phase 2), but we have not yet seen the process itself — how it is born, who
started it, what state it is in, how much resource it eats.

Three things from Phases 0, 1 and 2 pay off directly here:

- **"Every process has an identity"** (Phase 2). Now we look at the container that carries that
  identity — the process itself. Next to the `Uid` line you saw in `/proc/<PID>/status`, we will now
  also read the `State`, `PPid` and `Threads` lines.
- **"Everything is a file"** (Phase 0). This phase's most powerful tool, `/proc`, is a virtual file
  system that presents running processes **as files**. `cat /proc/<PID>/status` is reading a process.
- **`sudo` starts a process under a different identity** (Phase 2). Now we open up the mechanism
  underneath that "start" (`fork` + `exec`).

## The question of this phase

A cloud engineer spends most of the day on a single question:

> *"What is the system doing right now, and what is eating its resources?"*

CPU pinned at 100% — which process? Memory is full, something was killed by OOM — who ate it? You
told a service `systemctl stop` but it won't stop. A container is using far more CPU than expected.
`kill` does nothing, the process just won't die.

By the end of this phase, when you log into a server and open `top`, you will read the table you see
not from memory but from a **model**: why this process is in D state, why that one is a zombie, why
this one's parent has become PID 1. And most important of all: you will see that **containers are
not lightweight magic — they are ordinary processes fenced in by cgroups + namespaces** — the basis
of future EC2/EKS decisions.

---

## By the end of this phase

- You'll be able to read a process's PID, PPID and state; you'll be able to explain why PID 1
  (init/systemd) is special
- You'll be able to walk through, step by step, how one process gives birth to another (`fork` +
  `exec`)
- You'll recognize the process states (R, S, D, T, Z) in `ps` output; you'll know why **zombie** and
  **D-state** are different problems and why `kill -9` cannot kill a D-state process
- You'll be able to explain the difference between a thread and a process, and how it relates to the
  vCPU count
- You'll know the difference between SIGTERM, SIGKILL and SIGHUP; you'll be able to explain why a
  container/instance receives SIGTERM first when it shuts down, and why "graceful shutdown" matters
- You'll be able to use job control (`&`, `nohup`, `jobs`, `fg`/`bg`) and set CPU priority with
  `nice`/`renice`
- You'll be able to explain what `ulimit`, cgroup and namespace do, and the equation **cgroup +
  namespace = container**
- You'll be able to narrow down "who is eating the CPU" methodically with `ps aux`, `top`/`htop`,
  `pgrep`/`pkill` and `/proc/<PID>/`

---

## The phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 3.1 | Process anatomy: PID, PPID, init; `fork` + `exec` | `[mechanism]` | The atom of the running system; "where a process comes from" |
| 3.2 | Process states: R, S, D, T, Z | `[mechanism]` | **The diagnostic heart of the phase** — the key to reading `top`/`ps` |
| 3.3 | Thread vs process | `[concept]` | Under vCPU decisions and "why more cores didn't speed it up" |
| 3.4 | Signals: SIGTERM, SIGKILL, SIGHUP | `[mechanism]` + `[application]` | Stopping a process **correctly**; graceful shutdown |
| 3.5 | Job control and priority: `&`, `nohup`, `nice` | `[concept]` | Managing long jobs; CPU priority |
| 3.6 | Resource limits: `ulimit`, cgroup, namespace | `[mechanism]` + `[application]` | **The truth of the container is here** — the basis of EC2/EKS |
| 3.7 | Observation tools: `ps`, `top`, `pgrep`, `/proc` | `[application]` | The "who is eating the CPU" reflex |
| 3.8 | When this phase breaks | — | The signatures of process and resource failures |

> **How to work through this phase:** Most of the commands in this phase are 🟢 (read-only — `ps`,
> `top`, reading `/proc`) or 🟡 (temporary: pushing a job to the background with `sleep &`, starting
> a job with `nice`). A few boxes are marked 🔴: killing a process with `kill -9` or changing its
> priority with `renice` — **if you kill the wrong PID** you can bring down a running service. Try
> these boxes first on a harmless `sleep` process you started yourself. Most of the cgroup/namespace
> boxes can be read even without a machine; but if you have an Ubuntu machine, seeing a cgroup limit
> **live** with `systemd-run` is the most illuminating moment of this phase.

---
---

# 3.1 Process Anatomy

## 3.1.1 PID, PPID and init: every process has an identity and a parent `[mechanism]`

In Phase 2 we said "every process has a UID/GID" — that was **on whose behalf** the process runs.
Now we look at the process's **own** identity: every running process has a **PID** (Process ID). The
PID is the unique number the kernel gives to that process; a single number identifies every process
on the machine.

But processes are not born out of thin air. Every process is **started by another process** — and
the starter's PID is the **PPID** (Parent PID) of the newborn. This makes the system a **tree**:
every process has a parent, and at the root of the tree stands a single process.

> **🔧 See it on your machine** 🟢 — the identity lines of a process
>
> ```
> $ echo $$              # the current shell's own PID
> 4123
> $ ps -o pid,ppid,user,comm -p $$
>     PID    PPID USER     COMMAND
>    4123    4118 ubuntu   bash
> ```
>
> Your shell's PID is 4123, its parent (PPID) is 4118. And who is 4118? Probably the `sshd` process
> that let you in. Its parent is another `sshd` above it, and its parent... if you keep going up, you
> always arrive at **PID 1**.

**The root of the tree: PID 1.** At the end of boot the kernel starts **one** user-space process and
gives it PID 1. On modern systems this process is **systemd** (on older systems, `init`). Every other
process, directly or indirectly, is a descendant of PID 1.

> **🔧 See it on your machine** 🟢 — who is PID 1?
>
> ```
> $ ps -p 1 -o pid,comm
>     PID COMMAND
>       1 systemd
> $ pstree -p | head -5      # the tree, visually
> systemd(1)─┬─systemd-journal(412)
>            ├─sshd(890)───sshd(4118)───bash(4123)───pstree(4200)
>            └─...
> ```
>
> In the `pstree` output, follow the chain from your own `bash` up through `sshd → sshd →
> systemd(1)`. This is exactly the path you would walk starting from `echo $$` and following the
> PPIDs.

PID 1 has two special jobs, and both come up again later in this phase:

1. **Adopting orphan processes.** If a process's parent dies before it does, the child is left an
   **orphan**. The kernel immediately attaches the orphan to PID 1 — that is, its PPID becomes 1.
   So when you see a process whose PPID is 1, you read it as "its parent has died, systemd has
   adopted it" (this connects to `nohup` in 3.5.1 and to zombies in 3.2).
2. **"Clearing away the bodies" of dead children** (reaping). When a process dies, the kernel keeps
   its exit code until the parent reads it; if the parent doesn't read it, a **zombie** forms. PID 1
   regularly `wait`s on the processes it has adopted — which is why, without a real init process (for
   example in a badly built container), zombies can pile up.

> **⚠️ Common misconception: "A PID is a fixed thing; a program always has the same PID."**
> No. A PID is specific to each **run**; if you run the same program twice you get two different
> PIDs. PIDs are recycled once a process ends (after a while the kernel may give the same number to
> another process). This is why saving a PID as a "permanent identity" (for example in a script) is
> dangerous — after that PID dies the same number may belong to a completely different process, and
> you could `kill` the wrong one (we return to this trap in 3.4 and 3.7).

**PID 1 also has a special immunity:** the kernel **ignores** signals sent to PID 1 whose default
behavior is "terminate" (including SIGTERM and SIGKILL), unless PID 1 has installed a handler for
that signal. The reason is simple: if PID 1 dies the system crashes (kernel panic). This is why
`kill -9 1` does nothing — systemd responds only to the signals it defines itself (for example,
reload).

> **🤔 Think 3.1**
> In a terminal you typed `sleep 1000 &` to push it to the background, then closed that terminal
> (the shell). But `ps aux | grep sleep` still shows the process, and its PPID is now 1. What
> happened — why didn't the process die, and why did the PPID become 1?
> *(Answer: at the end of the phase)*

---
---

# 3.2 Process States

## 3.2.1 R, S, D, T, Z: the life states of a process `[mechanism]`

A process is not "running" every moment. Most processes spend the greater part of their life
**waiting**: for data from disk, for a key to be pressed, for a packet from the network. The kernel
keeps a **state** for each process, and this single letter is the key to reading `top`/`ps` output.

The five basic states you see in the `STAT` column of `ps`:

| Code | State | What it means |
|---|---|---|
| **R** | Running / Runnable | Running on the CPU **or** ready to run, waiting in the queue |
| **S** | Sleeping (interruptible) | Waiting for an event (I/O, timer, key); can be woken by a signal |
| **D** | Uninterruptible sleep | Usually locked in **disk/NFS** I/O; cannot be woken by a signal |
| **T** | Stopped | Stopped (Ctrl+Z or SIGSTOP); resumes with `fg`/`bg` |
| **Z** | Zombie | Dead, but the parent has not yet read its exit code |

> **🔧 See it on your machine** 🟢 — count the states live
>
> ```
> $ ps -eo stat,comm | sort | uniq -c | sort -rn | head
>     180 S    ...        # the overwhelming majority are sleeping — this is NORMAL
>      12 Ss   ...
>       3 R    ...        # a few processes actually running right now
>       2 Ssl  ...
>       1 R+   ...        # this is your ps command
> ```
>
> **The most important observation:** on a healthy system the overwhelming majority of processes are
> in state `S` (sleeping). "There are 100 processes but CPU is 5%" is not a contradiction — 97 of
> those 100 are waiting for an event, not consuming CPU. This is the key to not panicking the first
> time you open `top`.

The marks added next to the single letter in the `STAT` column also carry information: `s` (session
leader), `l` (multi-threaded), `+` (in the foreground process group), `<` (high priority, negative
nice), `N` (low priority). For example `Ssl` = a sleeping, session-leader, multi-threaded service;
`R+` = a running command in the foreground (often your own `ps`).

> **❓ Question that comes to mind: "If R means both 'running' and 'ready to run', which one is
> actually on the CPU?"** From the kernel's point of view both are in the same queue (the runqueue).
> As many processes as there are physical cores run **truly** at the same time; the rest are
> "runnable" — ready but waiting their turn. The number we call "load average" in `top` is exactly a
> moving average of the count of processes in this R state (running + waiting) (we open this up in
> Phase 4). So high load = "many processes want the CPU at the same time".

## 3.2.2 Zombie: dead but not buried `[mechanism]`

The name zombie sounds dangerous, but a zombie actually **consumes no resources** — no CPU, no
memory. A zombie is just a small **record** kept in the kernel's process table: only the PID and the
exit code.

Why does it exist? We saw it in 3.1: when a process dies the kernel keeps its exit code **until the
parent reads it**. The parent reads this code with the `wait()` (or `waitpid`) syscall — this is
called "reaping". If the parent does this, the zombie record is deleted immediately. If the parent
**doesn't** (buggy code, a program that forgets to call `wait`), the child hangs around as a zombie.

> **🔧 See it on your machine** 🟡 — create and see a zombie
>
> ```
> $ ( sleep 0.1 & exec sleep 5 ) &     # a subshell: the child dies fast, the parent doesn't wait
> [1] 5230
> $ sleep 1; ps -o pid,ppid,stat,comm --ppid 5230
>     PID    PPID STAT COMMAND
>    5231    5230 Z    sleep <defunct>       # <defunct> = zombie
> ```
>
> `<defunct>` and `Z` — this is a zombie. Because the parent (5230, `sleep 5`) does not `wait` for
> five seconds, the child's body hangs there. When the parent dies (after 5 s) the zombie is handed
> to PID 1, and systemd `wait`s on it and cleans it up right away. This is why zombies usually
> disappear **on their own**.

**When is a zombie a problem?** A single zombie is harmless. But if a parent process keeps giving
birth to children and never `wait`s, zombies **accumulate** and eventually the process table (PID
space) fills up — at which point the system cannot start new processes (errors like `fork: Cannot
allocate memory`, even though memory is free). The culprit is not the zombies but the **parent that
doesn't `wait`.**

> **⚠️ Common misconception: "I'll kill zombies with `kill -9`."**
> You cannot kill a zombie — it is **already dead.** `kill` sends a signal, but there is no running
> process to receive it; all that's left is a record. The only way to clean up a zombie is for its
> **parent to `wait` on it.** The fix: repair the parent (its code), or kill/restart the parent — then
> the zombie is handed to PID 1 and systemd cleans it up. So when fighting zombies the target is
> **the parent, not the child** (find the parent with `ps -o ppid= -p <zombie_pid>`).

## 3.2.3 D-state: a process that can't be woken even by a signal `[mechanism]`

State `D` (uninterruptible sleep) is the most misunderstood state of this phase — and the most
maddening in production. If a process is in state `D`, it is waiting inside the kernel for an
operation (usually disk or network I/O) to **complete**, and during this wait **no signal** can wake
it — not even `SIGKILL` (`kill -9`).

Why? Because the process is in the middle of the kernel, waiting for the result of a hardware
operation. If the kernel cut it off and killed it, the I/O being performed at that moment (for
example, a disk write) could be left in an **inconsistent** state. So the kernel says "no one can
touch this process until this operation finishes".

> **🔧 See it on your machine** 🟢 — look for D-state (you usually won't find it, which is good)
>
> ```
> $ ps -eo stat,pid,comm | awk '$1 ~ /D/'
> (no output — a momentary D is rare on a healthy system)
> ```
>
> On a healthy system it's hard to catch a `D` state because modern disks finish I/O in
> milliseconds. If you see **long-lasting** `D` (especially on NFS mounts or a failing disk) that's
> an alarm: the storage underneath is stuck.

**Why can't `kill -9` kill a D-state process?** Because `kill -9` is a signal, and signals are
delivered to a process only when it **returns from the kernel to user space**. A process in state D
is locked inside the kernel; it isn't returning to user space, so the signal is queued but never
delivered. The process wakes only when the I/O it was waiting for **completes** (or times out), and
at that moment it receives the queued signal.

The practical consequence in the cloud is clear: if an EBS volume or NFS mount stops responding, the
processes accessing it drop into state `D` and can **in no way** be killed — not by `kill -9`, not by
`systemctl stop`. The only fix is to bring the storage back, or (as a last resort) reboot the
machine. When you hear "the process just won't die, even `kill -9` does nothing", the first place to
look is the `D` letter in `ps` output and the disk/NFS underneath it.

> **🤔 Think 3.2**
> You opened `top`; a process is in state `D` and the machine's **load average** has jumped to 8.0,
> but CPU usage is only 3%. How can load be this high while almost no CPU is used? (Hint: which
> process states does load average count?)
> *(Answer: at the end of the phase)*

---
---

# 3.3 Thread and Process

## 3.3.1 Sharing the same address space: what a thread is `[concept]`

So far we have said "process" and thought of each process as an isolated unit: its own memory, its
own identity, its own file descriptors. That is right. But **inside** a process there can be more
than one **thread**, and these share the same memory.

Draw the difference in one sentence:

- **Process** = a unit of execution with its own **address space** (memory). Two processes cannot
  see each other's memory directly — the isolation from Phase 0.
- **Thread** = a flow of execution inside the same process, **sharing the same memory.** If a
  process has 10 threads, those 10 flows access the same variables, the same open files.

An analogy: a process is a **house** (its own walls, its own kitchen). Threads are the **people**
living in that house — they share the same kitchen, the same refrigerator. Different houses
(processes) can't get into each other's fridge; but people in the same house (threads) use the same
fridge — this is fast, but if one drinks another's milk (corrupts the same memory at the same time)
you get trouble (a race condition).

> **🔧 See it on your machine** 🟢 — count a process's threads
>
> ```
> $ ps -o pid,nlwp,comm -p 1          # nlwp = number of light-weight processes = thread count
>     PID NLWP COMMAND
>       1    1 systemd
> $ ps -eo pid,nlwp,comm --sort=-nlwp | head -5   # the most-threaded processes
>     PID NLWP COMMAND
>    1450   48 mysqld               # database: dozens of threads
>    1120   16 containerd
>     980    9 systemd-journald
> ```
>
> The `NLWP` column tells you how many threads a process has. On Linux, threads are really
> "light-weight processes" (LWP); each thread has its own **TID** (thread ID) but they are all
> grouped under the same PID. With `ps -eLf` you can see the threads one by one (the `LWP` column =
> TID).

> **⚠️ Common misconception: "On Linux, threads and processes are completely different things."**
> The surprising truth from the kernel's point of view: **both a thread and a process are represented
> by the same structure** (task_struct). Both are created with the `clone()` syscall; the difference
> is which things you tell `clone` to **share**. Say "share memory, share files" and you get a
> thread; say "share nothing, copy everything" and you get a process (`fork` is really the name given
> to `clone` in this form). So the line between thread and process is not rigid — it's a setting of
> **what is shared**.

## 3.3.2 Cloud connection: threads and the vCPU count `[concept]`

The thread concept ties directly to a cloud decision: **how many vCPUs should the instance I pick
have?**

If an application is single-threaded, it can use only **one** CPU core at a time. Even if you give it
a 16-vCPU instance, that application's computation runs on a single core — the other 15 cores sit
idle. This is one of the most common reasons behind "I made the server bigger but the application
didn't speed up".

By contrast a multi-threaded application (for example most databases, web servers, or a service that
does parallel computation) splits the workload across several threads, and these can run on different
cores **at the same time**. For such an application, increasing the vCPU count really does speed it
up — up to a point (the cost of coordinating between threads eventually eats the gain; Amdahl's law).

Practical takeaway: when picking an instance size, ask "how many threads can my application split the
work into?" Compare the core count the machine sees (`nproc`) with the application's thread count
(`ps -o nlwp`). If the application uses 4 threads, buying 16 vCPUs is wasting money.

> **❓ Question that comes to mind: "Is a vCPU a real core?"** Usually no — on most cloud providers a
> vCPU is a **hyperthread** of a physical core (one physical core appears as two vCPUs). Because two
> hyperthreads share some units of the same physical core, 8 vCPUs don't always mean "8× the speed".
> This is under the CPU-architecture topic in the hardware book; what you need to know here: the vCPU
> count is an upper bound, the real gain depends on the application's parallelism.

---
---

# 3.4 Signals

## 3.4.1 SIGTERM, SIGKILL, SIGHUP: how to "talk" to a process `[mechanism]`

The most basic way to communicate with a process is the **signal**: a short message, just a single
number, that the kernel sends to a process. The `kill` command — despite its name — is really the
"send a signal" command; killing is just the **default** effect of one of those signals.

When a process receives a signal, three things can happen: (1) it applies the signal's **default**
behavior (for most, "terminate"), (2) if it has installed its own **handler** for the signal, it
runs that, or (3) it **ignores** (blocks) the signal. The critical point: some signals can be caught
and handled, some **absolutely** cannot.

The three signals you'll meet in daily life:

| Signal | No | Default | Catchable | What for |
|---|---|---|---|---|
| **SIGTERM** | 15 | Terminate | ✅ Yes | "Shut down gracefully" — clean up and exit. `kill`'s default. |
| **SIGKILL** | 9 | Terminate | ❌ **No** | "Force kill" — the process dies instantly with no say. |
| **SIGHUP** | 1 | Terminate | ✅ Yes | Historical: "terminal closed". Modern: for most services, "reload config". |

`kill <PID>` → sends **SIGTERM** (15) by default. `kill -9 <PID>` → **SIGKILL**. `kill -1` or `kill
-HUP` → **SIGHUP**. You can list all signals with `kill -l`.

> **🔧 See it on your machine** 🟡 — gentle stop vs force kill
>
> ```
> $ sleep 300 &
> [1] 6010
> $ kill 6010            # SIGTERM — sleep doesn't catch it, ends gracefully
> [1]+  Terminated              sleep 300
>
> $ sleep 300 &
> [1] 6042
> $ kill -9 6042         # SIGKILL — instant, no say
> [1]+  Killed                  sleep 300
> ```
>
> `sleep` is a simple program; since it doesn't catch SIGTERM either, both end it. The difference
> shows up in a **complex application**: a database that catches SIGTERM writes its open transactions
> to disk before dying; with SIGKILL it gets no such chance.

**Why can't SIGKILL be caught?** By design. If every signal could be caught, a badly written (or
malicious) program could ignore all signals and become **unkillable**. SIGKILL (and its stopping
counterpart SIGSTOP) is the "last word" the kernel keeps in hand: the process is terminated directly
by the kernel, with no notice at all. This is why, when `kill -9` "does nothing" (3.2.3), the culprit
is **not** the process catching the signal — the process is either a zombie (already dead) or in
state D (locked in the kernel, the signal can't be delivered).

> **⚠️ Common misconception: "To stop something I always use `kill -9`, it's the surest."**
> `kill -9` is the last resort, not the first move. Because SIGKILL gives the process **no chance to
> clean up**: half-finished file writes can be corrupted, open database transactions can be left
> without rollback, temp files are not deleted, locks are not released. The right reflex is to try
> `kill` (SIGTERM) first, giving the process a chance to shut down; **only** if it doesn't die within
> a few seconds do you move to `kill -9`. `systemctl stop` does exactly this: first SIGTERM, wait a
> while (`TimeoutStopSec`), then SIGKILL.

## 3.4.2 Cloud connection: graceful shutdown and SIGTERM `[application]`

Signals are not an abstract topic in the cloud — this mechanism runs every time a container or
instance shuts down, and this is where it shows whether you wrote your application correctly.

**Shutdown always starts with SIGTERM.** In all three of the following scenarios the same thing
happens:

- `docker stop <container>` → sends **SIGTERM** to the container's process number 1, waits 10 seconds
  by default, then **SIGKILL**.
- Kubernetes terminating a pod → **SIGTERM**, waits `terminationGracePeriodSeconds` (default 30 s),
  then **SIGKILL**.
- `systemctl stop myapp` → **SIGTERM** to the service, waits `TimeoutStopSec`, then **SIGKILL**.
- An EC2 instance shutting down (`shutdown`) → systemd sends **SIGTERM** to all services.

So your application always gets the message "you have a few seconds to gather yourself before
shutting down" (SIGTERM) before it closes. **Graceful shutdown** is the application catching this
message and doing the following: stop accepting new requests, finish the requests in progress, close
the database connection properly, tell the load balancer "take me out of the list", then exit.

What happens if your application does **not** catch SIGTERM? SIGTERM's default is "terminate
immediately" — so your application cuts off the requests in progress, users get errors, data is left
half-done. This is why every application going to production must have a SIGTERM handler.

> **🔧 See it on your machine** 🟡 — write a process that catches SIGTERM
>
> ```
> $ cat > /tmp/graceful.sh <<'EOF'
> #!/bin/bash
> trap 'echo "got SIGTERM, cleaning up..."; sleep 2; echo "clean exit"; exit 0' TERM
> echo "running (PID $$)"
> while true; do sleep 1; done
> EOF
> $ chmod +x /tmp/graceful.sh
> $ /tmp/graceful.sh &
> [1] 6210
> running (PID 6210)
> $ kill 6210            # SIGTERM
> got SIGTERM, cleaning up...
> clean exit
> [1]+  Done                    /tmp/graceful.sh
> ```
>
> The `trap '...' TERM` line says "when SIGTERM arrives, run this command". Instead of dying
> instantly, the process cleans up and exits. In a real application this `trap` block closes open
> connections and removes its registration from the load balancer. **Undo:** the process exited on
> its own; if it didn't, `kill -9 6210`.

> **❓ Question that comes to mind: "Can an application that catches SIGTERM and never exits keep
> `docker stop` waiting forever?"** No — because the wait is bounded. `docker stop` waits 10 seconds
> (Kubernetes 30 s, systemd `TimeoutStopSec`); when that time is up it sends **SIGKILL**, and SIGKILL
> can't be caught. So in the worst case the application is force-killed — but that's a "dirty"
> shutdown. The goal is for the application to exit cleanly on its own **before** that time runs out.

> **🤔 Think 3.3**
> When a web application is stopped with `docker stop`, some users get a "502 Bad Gateway" error. The
> application does not catch SIGTERM. Reasoning through the signal chain (SIGTERM → 10 s → SIGKILL),
> at exactly **which moment** and **why** do these 502s occur? How would graceful shutdown have
> prevented it?
> *(Answer: at the end of the phase)*

---
---

# 3.5 Job Control and Priority

## 3.5.1 Foreground, background, `&`, `nohup`, `jobs` `[concept]`

When you run a command in a terminal, that command runs in the **foreground**: it occupies your
terminal, and you can't type another command until it finishes. For most commands this is fine (`ls`
finishes instantly). But for a long-running job (a backup, a build) you don't want it to block your
terminal.

This is where **job control** comes in — the shell managing several jobs from the same terminal:

- **`command &`** → start the command in the **background**, get the terminal back immediately.
- **`Ctrl+Z`** → **stop** the foreground command (SIGSTOP, state `T`) and push it to the background.
- **`jobs`** → list the jobs this shell manages.
- **`fg %1`** → bring job number 1 to the foreground. **`bg %1`** → resume a stopped job, running in
  the background.

> **🔧 See it on your machine** 🟡 — push a job to the background, call it back
>
> ```
> $ sleep 300              # foreground — the terminal is locked
> ^Z                       # Ctrl+Z: stop
> [1]+  Stopped                 sleep 300
> $ jobs
> [1]+  Stopped                 sleep 300
> $ bg %1                  # resume in the background
> [1]+ sleep 300 &
> $ jobs
> [1]+  Running                 sleep 300 &
> $ fg %1                  # bring it back to the foreground
> sleep 300
> ^C                       # end it with Ctrl+C
> ```

**An important trap: `&` is not enough.** Even if you push a command to the background with `&`, that
command is still **a child of this shell** and is tied to your session (the terminal). When your SSH
session closes (or you close the terminal), the kernel sends **SIGHUP** to the processes in that
session — and SIGHUP's default is "terminate". So the job you started with `command &` **can die**
when SSH disconnects.

Here is what `nohup` (no hangup) solves exactly: it starts the command so it will **ignore** SIGHUP,
redirecting its output to the `nohup.out` file. That way the job keeps running even if the session
closes.

> **🔧 See it on your machine** 🟡 — a session-independent job
>
> ```
> $ nohup ./long_job.sh &
> nohup: ignoring input and appending output to 'nohup.out'
> [1] 6350
> $ exit                   # even if you close the SSH session, 6350 lives on
> ```
>
> Log in again over SSH and run `ps aux | grep long_job`; you'll see the process is still there and
> its PPID is **1** — when the session died, systemd adopted it (remember 3.1.1). **Undo:** `kill
> <PID>`.

> **⚠️ Common misconception: "I'll run a permanent service with `nohup ... &`, done."**
> `nohup &` is fine for one-off long jobs (a migration, a bulk download). But for a **permanent
> service** it's the wrong tool: if the machine reboots the job doesn't come back, if it crashes it
> doesn't restart, there's no log management, and someone else won't know how to stop it. The right
> home for permanent services is **systemd** (Phases 5 and 8): it starts/stops with `systemctl`,
> comes up automatically at boot, restarts on crash with `Restart=on-failure`, and its logs are
> collected in `journalctl`. `nohup` is for "let it run for now"; systemd is for "let it always run".

## 3.5.2 `nice` and `renice`: CPU priority `[concept]`

CPU is a limited resource; if several processes want the CPU at the same time, the kernel's
**scheduler** decides who gets how much of a turn. The setting that influences this decision is the
**nice** value.

The nice value ranges from **-20** to **+19** and works counter to intuition: **high nice = low
priority** ("nicer to others, I'll give up my turn to them"). The default is 0.

- **nice +10, +19** → "I'm in no hurry, let others go first". Ideal for background batch jobs.
- **nice -10, -20** → "move me to the front". Only root can set this (an ordinary user can **lower**
  its priority but not raise it).

> **🔧 See it on your machine** 🟡 — start a low-priority job
>
> ```
> $ nice -n 15 ./big_computation.sh &      # start at low priority
> $ ps -o pid,ni,comm -p $!                 # $! = PID of the last background job
>     PID  NI COMMAND
>    6410  15 big_computation.sh
> $ sudo renice -n 5 -p 6410                # change the priority of a running process
> 6410 (process ID) old priority 15, new priority 5
> ```
>
> The `NI` column is the nice value. You set it **at start** with `nice`, and **while running** with
> `renice`. **Undo:** it disappears when the process ends; `kill 6410`.

**When does nice help and when doesn't it?** nice arranges the queue only for the **CPU**, and only
when there is CPU **scarcity**. If the machine is idle, even a nice +19 job uses the whole CPU (no
one else wants the turn). But if two jobs compete for the CPU, the one with the lower nice takes the
lion's share. By contrast, nice does not manage **disk I/O** or **memory** — to prioritize a job by
disk I/O you need `ionice`, and for a real resource **quota** you need a cgroup (3.6). So nice is a
"polite request"; a cgroup is a "hard quota".

> **❓ Question that comes to mind: "If a process is eating 100% CPU, does lowering it with `renice`
> fix the problem?"** Usually no, it only **defers** it. If that process really has to do that work,
> lowering its priority slows it down but the work still has to be done. `renice` is good for pushing
> a background monster back temporarily while *another* important thing waits for the CPU. But it's
> not the **answer** to "why is the CPU at 100%" — you find that answer with the observation tools in
> 3.7 (which process, doing what).

---
---

# 3.6 Resource Limits: ulimit, cgroup and namespace

## 3.6.1 `ulimit`: per-process limits `[concept]`

So far we've seen how processes are born, die and get prioritized. But how hard can a single process
push the system? Is it unlimited? No — the kernel keeps a set of **upper bounds** (resource limits)
for each process, and you see and set them with `ulimit`.

The two limits you'll meet most:

- **Number of open files** (`ulimit -n`): how many files/sockets a process can have open at once.
  Remember from Phase 0 — every network connection is also a file descriptor. On a high-traffic
  server, when this limit fills up you get the `Too many open files` error; one of the most common
  production errors in the cloud.
- **Number of processes/threads** (`ulimit -u`): how many processes a user can start. This limit
  stops a "fork bomb" (a process that copies itself endlessly) from locking up the whole system.

> **🔧 See it on your machine** 🟢 — read your own limits
>
> ```
> $ ulimit -n              # open-file limit (soft)
> 1024
> $ ulimit -Hn             # the hard limit of the same
> 1048576
> $ ulimit -a | head       # all limits
> ```
>
> The **soft limit** is the currently effective limit; the **hard limit** is the ceiling the soft can
> be raised to. An ordinary user can raise the soft up to the hard but cannot exceed the hard (root
> or `/etc/security/limits.conf` sets that). If you see `Too many open files`, the first place to
> look is `ulimit -n` and the `LimitNOFILE` setting in the service's systemd unit (Phase 5/8).

`ulimit` sets a limit for **a single process** (and its children). But what if you want to say "I
don't want this **group** of processes to use more than 512MB of RAM in total"? Here `ulimit` falls
short and **cgroup** takes over.

## 3.6.2 cgroup: a resource quota for a group of processes `[mechanism]`

A **cgroup** (control group) is a kernel feature: it gathers one or more processes into a group and
puts a **collective resource quota** on that group — like "this group can use at most 50% CPU and
512MB RAM". Where `ulimit` is a per-process limit, a cgroup is a **per-group** quota, and it's far
more powerful.

A cgroup does three things:
1. **Limiting:** it puts an upper bound on the group's CPU, memory, disk I/O, network.
2. **Accounting:** it measures how much resource the group uses.
3. **Isolation:** it keeps one group's over-consumption from affecting the others.

cgroups too are managed by the "everything is a file" principle: they sit under `/sys/fs/cgroup/` as
a virtual file system. Creating a cgroup is creating a directory, and setting a limit is writing a
number to a file.

> **🔧 See it on your machine** 🟡 — run a memory-limited process (with systemd)
>
> ```
> $ systemd-run --user --scope -p MemoryMax=100M stress --vm 1 --vm-bytes 200M
> # (if stress is not installed: sudo apt install stress)
> # The process tries to fit into 100MB; asking for 200MB, the cgroup kills it with OOM:
> $ journalctl --user -n 5 | grep -i memory
> ... Memory cgroup out of memory: Killed process ... (stress)
> ```
>
> `MemoryMax=100M` puts a 100MB ceiling on this temporary cgroup; when `stress` asks for 200MB the
> kernel triggers an OOM **inside the group** and kills the process — without touching the rest of the
> machine. This is the **exact** answer to "why is a container killed when it exceeds its memory
> limit". **Undo:** with `--scope`, the cgroup is deleted on its own when the process ends.

This experiment shows a fact that is critical for the cloud: when a container is `OOMKilled`, the
machine's memory **may not** be full — only that container's **cgroup limit** is full. The answer to
"the instance has 8GB of RAM, why did the container die?" is very often "the container had a 512MB
cgroup limit set on it".

## 3.6.3 namespace: isolating the world a process sees `[concept]`

A cgroup limits **how much** resource a group will use. A **namespace** limits **what** a process can
see — it shows the process only a slice of the system and hides the rest.

Different namespace types isolate different things:

- **PID namespace:** the process sees its own isolated PID space. Inside, its own process is PID 1;
  it can't see the processes outside (on the host) at all.
- **Mount namespace:** its own file-system view — its own `/`, its own mounts.
- **Network namespace:** its own network interfaces, its own IP, its own routing table.
- **UTS namespace:** its own hostname. **User namespace:** its own UID mapping (root inside, possibly
  unprivileged outside).

> **🔧 See it on your machine** 🟢 — see a process's namespaces
>
> ```
> $ ls -l /proc/self/ns/
> lrwxrwxrwx ... pid -> 'pid:[4026531836]'
> lrwxrwxrwx ... mnt -> 'mnt:[4026531840]'
> lrwxrwxrwx ... net -> 'net:[4026531992]'
> ...
> ```
>
> Under `/proc/<PID>/ns/` it says which namespaces each process belongs to. If two processes share
> the same bracketed number they are in the **same** namespace. If you look at a process inside a
> container, you'll see its `net` and `pid` numbers are **different** from the host's — that is
> isolation.

## 3.6.4 The equation: container = process + cgroup + namespace `[application]`

Now we can state the most important sentence of this phase. A **container** is **not** a magical
lightweight virtual machine. A container is an **ordinary process** (or group of processes) running
on the host kernel — just wrapped in two things:

- **isolated** by a **namespace** ("sees only its own world": its own PID 1, its own network, its
  own file system),
- **limited** by a **cgroup** ("can use this much CPU/RAM").

That's all. The `nginx` process inside the container is **visible** when you run `ps aux` on the host
— as an ordinary process, with its own PID in the host's PID space. The container has only put it
inside a namespace bubble and fenced it with a cgroup quota.

![Figure 3.1 — Container = process + namespace (isolation) + cgroup (quota)](../diagrams/png/lx-3-01-container-anatomy.png)

*Figure 3.1 — Two containers on the same host kernel. Each is made of ordinary processes; the
namespace shows them a separate "world" (their own PID 1, their own network), and the cgroup quotas
each one's resources. The difference from a VM: there is a single kernel in the middle, no separate
operating systems.*

Why does this explain "a container is not a lightweight VM"? Because in a **VM** a separate operating
system kernel runs (a full kernel + userland on top of a hypervisor). In a **container** there is no
separate kernel — all containers share the **host's single kernel**. This is why containers start in
**milliseconds** rather than seconds (no new kernel to boot), take far less memory, but their
isolation is **not as strong** as a VM's (because they share the same kernel, a single kernel
vulnerability can affect all containers).

> **⚠️ Common misconception: "The inside of a container is a separate machine, fully cut off from the
> host."**
> It isn't. The process inside the container is **visible and killable** on the host. `docker stop`
> is really sending SIGTERM to that process (3.4.2 — now it's clear why it's SIGTERM). When a
> container has "its own PID 1", that's a namespace illusion; the host sees and manages this process
> under a completely different number in its own PID space. This insight is a lifesaver later in
> EKS/ECS when you're asking "why did the container die, what shows up on the host".

> **🤔 Think 3.4**
> You gave a container `--memory=256m`. The application inside runs `free -m` and sees **8GB** (the
> whole host), but when it tries to use 300MB it dies with OOM. Why does `free` show the wrong (the
> host's) memory, but the limit is still enforced at 256MB? (Hint: which mechanism manages *what is
> visible*, and which manages *the quota*?)
> *(Answer: at the end of the phase)*

---
---

# 3.7 Observation Tools

## 3.7.1 `ps`, `top`/`htop`, `pgrep`/`pkill` `[application]`

All the concepts of this phase (PID, state, thread, priority, resource) meet in a single practical
question: **"what is this machine doing right now, and what is eating its resources?"** You find the
answer with three tools.

**`ps` — an instant, single-frame photo.** It gives an image of the system's processes at that
moment. The two most-used forms:

```
$ ps aux                 # all processes, with user-focused columns
USER  PID %CPU %MEM   VSZ   RSS STAT START   TIME COMMAND
root    1  0.0  0.1 16788  9012 Ss   09:00   0:02 /sbin/init
www-d 890  2.3  1.2 72340 51200 S    09:05   0:31 nginx: worker
$ ps -ef                 # the same info, with the parent (PPID) column, hierarchy-focused
```

Reading the `ps aux` columns is the summary of this phase: `PID` (3.1), `STAT` (3.2), `%CPU`/`%MEM`
(resource), `VSZ`/`RSS` (memory — opened up in Phase 4), `COMMAND` (what's running). For "who is
eating the most CPU?": `ps aux --sort=-%cpu | head`.

**`top`/`htop` — a live, flowing film.** Unlike `ps`, `top` refreshes once a second; you watch live
which process is eating CPU/memory **right now**. `htop` (installed separately) is the friendly
version of the same thing — colored, navigable with arrow keys, able to send a signal with `F9`.

> **🔧 See it on your machine** 🟢 — understand `top`
>
> ```
> $ top
> top - 14:22:01 up 5 days,  load average: 0.15, 0.20, 0.18
> Tasks: 142 total,   1 running, 141 sleeping,   0 stopped,   0 zombie
> %Cpu(s):  3.0 us,  1.0 sy, 0.0 ni, 95.5 id,  0.5 wa, ...
> MiB Mem : 7940.0 total, 4200.0 free, 1800.0 used, 1940.0 buff/cache
>   PID USER   PR  NI  %CPU  %MEM   TIME+ COMMAND
>   890 www-d  20   0   2.3   1.2  0:31.2 nginx
> ```
>
> Reading order (top to bottom): **load average** (3.2.1 — the workload in R+D state), the **Tasks**
> line (how many running/sleeping/**zombie** — 3.2), on the **%Cpu** line `id`=idle, `wa`=I/O wait
> (high `wa` = disk bottleneck, Phase 4), the **Mem** line (Phase 4), then the process list. Sort by
> CPU with `P`, by memory with `M`; send a signal to a PID with `k`; see the cores one by one with
> `1`; quit with `q`.

**`pgrep`/`pkill` — find by name, signal by name.** Instead of memorizing PIDs you work by name:

```
$ pgrep -a nginx         # list processes containing nginx with their PIDs
890 nginx: worker process
891 nginx: worker process
$ pkill -TERM nginx      # send SIGTERM to all of them (careful!)
```

> **⚠️ Common misconception: "`pkill java` kills only the application I want."**
> `pkill` hits **all** processes whose name matches. If there are three different Java services on the
> machine, `pkill java` kills all three. Likewise `pkill -f app` can target anything with "app" in
> its command line (maybe `myapp`, maybe something related to `apparmor`). Before running a `pkill`,
> **always** run `pgrep -a` with the same pattern first and **see who you'll hit.** In production this
> habit is the only safety net against accidentally bringing down another service.

## 3.7.2 `/proc/<PID>/`: looking inside a process `[application]`

`ps` and `top` give a summary; but if you want to learn **everything** about a process you go to the
source: `/proc/<PID>/`. The most powerful example of Phase 0's "everything is a file" principle — the
entire internal state of a running process sits here as files.

| Path | What it tells you |
|---|---|
| `/proc/<PID>/status` | State, PPID, UID/GID, thread count, memory summary — human-readable |
| `/proc/<PID>/cmdline` | The full command line that started the process (with its arguments) |
| `/proc/<PID>/cwd` | The process's working directory (a symlink) |
| `/proc/<PID>/exe` | The running binary itself (a symlink) |
| `/proc/<PID>/fd/` | The process's **open file descriptors** — which files/sockets are open |
| `/proc/<PID>/limits` | The ulimit values applied to this process |
| `/proc/<PID>/environ` | Environment variables (remember from 3.1: only the owner reads it) |

> **🔧 See it on your machine** 🟢 — "what is this process doing?" in depth
>
> ```
> $ pgrep -a nginx | head -1
> 890 nginx: worker process
> $ cat /proc/890/cmdline | tr '\0' ' '; echo    # the full command it was started with
> nginx: worker process
> $ sudo ls -l /proc/890/fd | head               # its open files and sockets
> lrwx------ ... 3 -> 'socket:[28451]'            # a network connection
> l-wx------ ... 5 -> /var/log/nginx/access.log   # the log file it writes to
> $ sudo cat /proc/890/limits | grep 'open files' # this process's file limit
> Max open files  1024  524288  files
> ```
>
> This is the solution to "a process is eating CPU/disk but I don't know **what it's doing**": with
> `/proc/<PID>/fd` you see which files/sockets it opened, with `cmdline` exactly how it was started.
> For a service getting "Too many open files", you count how many files it has open with `ls
> /proc/<PID>/fd | wc -l` and compare against `limits`.

## 3.7.3 Putting it together: the "who is eating the CPU" reflex `[application]`

This phase's tools combine into a single methodology. When a server slows down, in order:

1. **`top`** (or `uptime`) → is the load average high? On the `%Cpu` line, where is the time going —
   `us` (user code), `sy` (kernel), `wa` (I/O wait)?
2. **`P` inside `top`** → sort by CPU, take the top PID.
3. **`ps -p <PID> -o pid,ppid,user,stat,cmd`** → who is that process, whose child, in which state?
4. **`/proc/<PID>/`** → exactly what it's doing: `cmdline`, `fd/` (which files/sockets), if needed
   `sudo cat /proc/<PID>/status`.
5. Decide: is it really necessary work (then `renice` or add resources), or a runaway/faulty process
   (then `kill` SIGTERM first, then `kill -9` if needed)?

These five steps are "finding it by narrowing with data, without guessing" — the process leg of the
**forensic reflex** we will generalize to all systems in Phase 11.

---
---

# 3.8 When This Phase Breaks — Failure Signatures

Most failures that look "mysterious" in the process and resource world are actually symptoms of a few
of this phase's mechanisms. The table below ties the symptoms you'll see in the field to this phase's
mechanisms:

| Symptom | Likely mechanism | Where explained | Look first at |
|---|---|---|---|
| A process just won't die, even `kill -9` does nothing | State `D` — locked in I/O in the kernel | 3.2.3 | `ps -o stat`; disk/NFS health, `dmesg` |
| Memory is free but `fork: Cannot allocate memory` | Zombie buildup or `ulimit -u` full | 3.2.2, 3.6.1 | `ps aux | grep defunct`; `ulimit -u` |
| I stopped a service but a process is still there, PPID=1 | Parent died, systemd adopted it; the real process is elsewhere | 3.1.1 | `pstree -p`; find the right PID |
| `docker stop` takes 10 s then the container dies | The app doesn't catch SIGTERM, SIGKILL is awaited | 3.4.2 | Add a SIGTERM handler to the app |
| Users get 502/errors at shutdown | No graceful shutdown; requests cut off mid-flight | 3.4.2 | On SIGTERM "stop new requests + wait" |
| I made the server bigger but the app didn't speed up | Single-threaded app, extra vCPUs idle | 3.3.2 | `ps -o nlwp`; compare with `nproc` |
| load average 8 but CPU 3% | Load also counts `D`-state I/O-waiting processes | 3.2.1, 3.2.3 | `top` `wa`; which process is in `D` |
| `Too many open files` | `ulimit -n` (fd limit) full | 3.6.1 | `ls /proc/<PID>/fd | wc -l`; `LimitNOFILE` |
| Container `OOMKilled` but plenty of RAM on the host | cgroup **memory limit** full, not the host | 3.6.2 | Container memory limit; `MemoryMax` |
| A job I started with `nohup ... &` died when SSH dropped | SIGHUP; either no `nohup` or tied to the session | 3.5.1 | `nohup`/systemd; check PPID |
| `pkill` killed the wrong service too | The pattern matched too broadly | 3.7.1 | Run `pgrep -a <pattern>` first |
| A process at 100% CPU, the system is slow | Runaway/faulty process or real load | 3.7.3 | `top`→PID→`/proc/<PID>/`→decide |

> **The lesson from this table:** Most process failures reduce to two questions — *"what state is the
> process in?"* (the `STAT` in `ps`: if `Z` zombie, look for the parent; if `D`, look for storage)
> and *"who set the limit?"* (`ulimit` is per-process, cgroup is per-group). "I can't kill it" is a
> kernel state, "I can't start it" is a limit, "it didn't speed up" is a parallelism problem. The
> right reflex is to read the state first (the five steps in 3.7), then descend to the right
> mechanism — not to rain down random `kill -9`s.

---
---

# Phase 3 — Answers to the think questions

## Answer 3.1 — Why the background `sleep` didn't die, why PPID became 1

**Question:** You typed `sleep 1000 &`, pushed it to the background, closed the terminal. `ps` still
shows the process and its PPID is 1. What happened?

Two separate things come together here. **First, why the process didn't die:** when the terminal (the
shell) closes, the kernel sends **SIGHUP** to the processes in that session. SIGHUP's default is
"terminate" — so in most cases the `sleep` **should** have died too. If it didn't die in your shell,
`huponexit` is probably off for a job started with `&` (that's Bash's default: on a non-interactive
shell exit it doesn't send HUP to background jobs), or you detached the job from the session with
`disown` or `nohup`. So the "didn't die" scenario is the one where SIGHUP **never reached** that job.

**Second, why PPID became 1:** `sleep`'s parent was your shell. When the shell closed, `sleep` was
left an orphan. Remember from 3.1.1: the kernel immediately attaches an orphan process to **PID 1
(systemd)**. So in `ps` output the PPID is no longer the original shell's PID but `1`. So when you
see "PPID=1", what you should read is: *this process's original parent has died, systemd has adopted
it.* This is the typical signature of long-running background jobs and daemons.

**Related section:** 3.1.1, 3.5.1 · **Continues in:** 3.4 (SIGHUP), Phase 5 (systemd, daemons)

---

## Answer 3.2 — CPU 3% but load average 8: how?

**Question:** A process is in state `D`, load average is 8.0, but CPU usage is 3%. Why is load so
high while almost no CPU is used?

**Because on Linux, load average doesn't only count processes waiting for the CPU — it also counts
processes in state `D`** (uninterruptible sleep, usually locked in disk/NFS I/O). This is a
Linux-specific and much-misunderstood definition: load average = a time-average of the count of
processes in state "running (R) + ready-to-run waiting (R) + uninterruptible sleep (D)".

The scenario's table is exactly this: the CPU is idle (3%) because the processes are **not** asking
for the CPU — they are waiting for the **disk**. But every process in state `D` adds "1" to the load.
If 8 processes tried to access a slow/failing disk (or an unresponsive NFS mount) and dropped into
`D`, load jumps to 8 while the CPU is almost idle.

The diagnostic value of this is large: **when you see "high load but idle CPU", the culprit is not
the CPU but the I/O.** In `top` the `wa` (I/O wait) on the `%Cpu` line is high; with `ps -eo
stat,comm | grep '^D'` you find the processes in `D` and check the storage underneath (disk health,
NFS connection, EBS status). So reading load average as "CPU busy" is a classic mistake; it's really
a measure of "work queued for the CPU + disk".

**Related section:** 3.2.1, 3.2.3 · **Continues in:** Phase 4 (load average in depth, I/O bottleneck)

---

## Answer 3.3 — 502s on `docker stop`: at which moment, why

**Question:** The app doesn't catch SIGTERM; when stopped with `docker stop`, some users get 502. At
exactly which moment and why do these 502s occur?

Build the signal chain step by step: `docker stop` first sends **SIGTERM** to the container's process
number 1. Because the app doesn't catch it, SIGTERM's **default** runs: the process **terminates
immediately.** The 502s occur at exactly this moment — the instant SIGTERM arrives and the app dies
right away.

Why 502? Because when the app dies, the **requests being processed at that moment** are cut off, and
the load balancer / reverse proxy in front (nginx, ALB) is still sending requests to this container —
because the container died before it could say "take me out of the list". When the backend suddenly
closes the connection, the proxy says "couldn't reach upstream / invalid response" and returns **502
Bad Gateway** to the client. So the 502s occur in the window where "the proxy is still sending
requests to a dead backend".

**How graceful shutdown would have prevented it:** if the app caught SIGTERM, that 10-second window
(before SIGKILL arrives) would be used for exactly this: (1) set the health check to "unhealthy" so
the load balancer stops sending new requests, (2) **finish** the requests still being processed, (3)
close connections properly, (4) then exit. This way no request is cut off mid-flight, and no 502
occurs. Catching SIGTERM means "I'm about to die, let me first finish the work in my hands properly"
— and 10 seconds is more than enough for that.

**Related section:** 3.4.1, 3.4.2 · **Continues in:** Phase 7 (load balancer, health check), Phase 8
(systemd service)

---

## Answer 3.4 — Inside the container `free` shows 8GB but the limit is enforced at 256MB

**Question:** You gave the container `--memory=256m`. Inside, `free -m` shows **8GB** (the whole
host), but using 300MB kills it with OOM. Why does `free` show the wrong value while the limit is
still enforced?

The answer is that two separate mechanisms of this phase do **different jobs**: **the namespace
manages visibility, the cgroup manages the quota.**

Where does `free -m` read its info from? From the `/proc/meminfo` file. And the classic
`/proc/meminfo`, even inside the container's **mount namespace**, shows the **host's** memory info —
because `/proc` does not offer a container-specific view on this (this is a known "leak" of
containers; tools like `free`, `top`, `nproc` frequently show host values). So the namespace tries to
show the process an isolated world, but on memory **reporting** it is not fully isolated: `free` sees
the host's 8GB.

But the **limit** is enforced by another mechanism, the **cgroup**. `--memory=256m` puts the
container's processes into a 256MB memory cgroup. When the app asks for 300MB, the kernel evaluates it
not by looking at what `free` shows but by looking at the **cgroup's accounting**: the group exceeded
its own 256MB limit → the kernel triggers an OOM **inside the group** and kills the process. The fact
that `free` shows 8GB has no effect on this decision; because it's the cgroup that holds the quota,
not `free`.

Practical lesson: **don't trust** the output of `free`/`top`/`nproc` inside a container — they
frequently show the host. Read the real limits from the cgroup: in cgroup v2, `/sys/fs/cgroup/memory.max`
(limit) and `memory.current` (current usage). The answer to "why did my app get OOM, look, there's
plenty of RAM on the system" is always here — look not at the system's RAM but at the **cgroup
limit**.

**Related section:** 3.6.2, 3.6.3, 3.6.4 · **Continues in:** Phase 4 (memory metrics, OOM), Phase 6
(/proc, /sys)

---
---

# Phase 3 — Frequently asked questions

### Q1. When should I use `ps`, `top` or `htop`?

- **`ps`**: when you want a single **instant snapshot** — in a script, to record to a log, or to
  filter "exactly which processes exist right now" (`ps aux --sort=-%cpu | head`). It doesn't flow,
  it runs once and finishes.
- **`top`**: to **watch the system live** — how CPU/memory changes over time, which process stands
  out. It's installed everywhere, you can run it the moment you log into a server.
- **`htop`**: the friendly version of `top` — colored, navigable with arrow keys, sends a signal with
  `F9`, has a tree view (`F5`). It needs to be installed separately (`apt install htop`), so it may
  not always be present on a production server; that's why knowing `top` is a must.

### Q2. What is init, why is systemd PID 1, what does "init system" mean?

The **init system** is the **first** user-space process (PID 1) the kernel starts after finishing
boot, and it does two jobs: (1) start all the other services in the right order, (2) adopt orphan
processes and clean up zombies (3.1.1). This used to be done by `SysV init` (with scripts); on most
modern distributions **systemd** has taken its place. systemd is not just a "starter" but a whole that
manages services (units), logs (journald), scheduled jobs (timers) and more — Phase 5 is devoted
entirely to it. What you need to know here: PID 1 = init = systemd, and this is the root of the tree.

### Q3. Why does `kill -9` sometimes not work? What is `kill -0` for?

`kill -9` (SIGKILL) can't be caught, but in two cases it "does nothing" (3.2.3): (1) the process is a
**zombie** — already dead, nothing to kill, find its parent; (2) the process is in **state `D`** —
locked in I/O in the kernel, the signal can't be delivered, fix the storage underneath. So if `kill
-9` doesn't work, the problem is not the signal but the process's state. **`kill -0`**, on the other
hand, sends no signal at all; it just checks "can I send a signal to this PID (does the process exist
and do I have permission)?" — used in scripts to test whether a process is still alive.

### Q4. What is the PID 1 trap in a container, and why is `--init` needed?

In a container your application usually runs as **PID 1**. But ordinary applications are not designed
to be PID 1: (1) they don't reap zombies (3.2.2) — if child processes are born and die, zombies pile
up; (2) their default behavior for SIGTERM works differently at PID 1 — the kernel ignores signals
sent to PID 1 that have no handler (3.1.1), so if your app didn't install a SIGTERM handler `docker
stop` cannot **stop it gracefully**, it waits for SIGKILL. The fix: let a lightweight init process
(for example `tini`) be PID 1, run your application as its child, and reap the zombies — `docker run
--init` does exactly this. It's the classic answer to "the container doesn't shut down properly /
zombies pile up".

### Q5. What is the difference between `nice` and a cgroup CPU limit?

**`nice`** is relative and works only during **scarcity** (3.5.2): it decides who goes ahead among
processes competing for the CPU, but if the machine is idle even a low-priority job uses the whole
CPU. A **cgroup CPU limit** is an **absolute quota**: if you say "at most 0.5 cores to this group",
that group **cannot** take more than half a core even if the machine is completely idle. So `nice`
says "be polite in the fight for a turn"; a cgroup says "this is your ceiling, you can't exceed it
even when idle". Container CPU limits (`--cpus=0.5`) are enforced with a cgroup, not with `nice`.

### Q6. What does "daemon" mean, how does a process turn into a background service?

A **daemon** is a service process that runs continuously in the background, not attached to a terminal
(their names usually end with `d`: `sshd`, `systemd`, `dockerd`). Classically, to detach itself from
its session and turn into a daemon, a process did a dance called "double fork": `fork`, let the parent
exit (so the child is orphaned and attached to PID 1), open a new session (`setsid`), release the
terminal. In the modern world this handiwork isn't needed — **systemd** starts the service directly in
the right environment (Phase 5/8). So "making a daemon" now mostly means "writing a systemd unit
file".

### Q7. Is a load average of 1.0 good or bad?

**It depends on the core count.** Load average is the average count of processes wanting to work (R +
D) (3.2.1); you read it against the **core count**. On a single-core machine, load 1.0 = "completely
full, at full capacity" (good); load 4.0 = "4× as much work queued, the system is drowning" (bad). On
a 4-core machine, load 4.0 = "completely full", 1.0 = "at 25% usage" (comfortable). Practical rule:
divide the load by `nproc` — if the result is under 1 it's comfortable, around 1 it's completely
full, well above 1 the queue is building up. And remember: high load isn't always CPU, it can be I/O
wait in state `D` too (Answer 3.2).

---
---

# Phase 3 — Test yourself

Answer the 18 questions below. The answer key and scoring are right underneath. The goal is not
memorization but being able to answer "what is the system doing right now" from the model.

## Section A — Basics (1–6)

**1.** What is a process's PPID, and what does seeing a PPID of `1` tell you?

**2.** What does `Z` mean in the `STAT` column of `ps`, and how much CPU/memory does this process
consume?

**3.** Which signal does `kill <PID>` (without a flag) send, and what is that signal's basic
difference from SIGKILL?

**4.** Where does a process with nice value `+15` stand in CPU priority compared to one with nice
`0`?

**5.** What does `ulimit -n` limit, and which error message appears when it fills up?

**6.** What is the basic scope difference between a cgroup and `ulimit` (what do they limit)?

## Section B — Mechanism (7–12)

**7.** Why can't `kill -9` kill a process while it's in state `D` (uninterruptible sleep)?

**8.** You can't clean up a zombie process with `kill -9`. What is the **only** way to get rid of it,
and why?

**9.** From the Linux kernel's point of view, what is the difference between a thread and a process —
with which syscall are both created?

**10.** If load average is high but CPU usage is low, what state are the processes contributing to the
load most likely in, and what are they waiting for?

**11.** namespace and cgroup do two different jobs in a container. Which one manages "what it can
see", and which "how much it can use"?

**12.** Why are SIGKILL and SIGSTOP signals that cannot be caught (no handler can be installed)? What
is the design rationale?

## Section C — Application and reasoning (13–18)

**13.** When an application is stopped with `docker stop`, users get 502. The application doesn't
catch SIGTERM. Where is the problem, and what is the right fix?

**14.** You told a service `systemctl stop` but `ps` still shows a related process, with PPID 1.
Write two different possible explanations.

**15.** A container was `OOMKilled` but `top` shows 6GB free RAM on the host. Why did it die, and
where do you read the real limit?

**16.** A process is eating 100% CPU. Instead of killing it by guessing, which steps, in which order,
do you follow to understand "what it's doing"?

**17.** Which single command should you run before running `pkill -f worker`, and why?

**18.** You got an 8-vCPU instance but your single-threaded application didn't speed up. With which
command do you confirm this, and with which concept do you explain it?

---

## Answer key

**1.** PPID = the parent process's PID; every process is started by another process. If PPID is `1`,
the process's original parent has died and the process has been adopted by systemd (PID 1). · *3.1.1*

**2.** `Z` = zombie: dead but the parent has not yet `wait`ed on its exit code. It consumes **no**
CPU/memory; it's just a small record in the process table. · *3.2.2*

**3.** `kill <PID>` sends **SIGTERM** (15) — catchable, "shut down gracefully". SIGKILL (9) can't be
caught and kills the process instantly without a chance to clean up. · *3.4.1*

**4.** At a **lower** priority; high nice = low priority. When there is competition for the CPU the
nice-0 process runs first, the +15 one says "I'll give my turn to you". · *3.5.2*

**5.** The number of **file descriptors** (files + sockets) a process can have open at once. When it
fills up the `Too many open files` error appears. · *3.6.1*

**6.** `ulimit` sets a limit per **single process** (and its children); a cgroup sets a collective
quota on **a group of processes** (the group uses this much CPU/RAM in total). A cgroup also manages
resources like CPU/IO. · *3.6.1, 3.6.2*

**7.** Because a process in state `D` is waiting inside the kernel for an I/O to complete and is not
returning to user space; signals are delivered only when the process returns from the kernel to user
space. The signal is queued but can't be delivered until the I/O finishes. · *3.2.3*

**8.** Its parent `wait`ing on it (reaping). Because a zombie is already dead a signal does nothing;
the only fix is the parent reading the exit code — you either repair the parent or kill it (then the
zombie is handed to PID 1 and systemd cleans it up). · *3.2.2*

**9.** From the kernel's point of view both are represented by a **task** (task_struct) and both are
created with the `clone()` syscall; the difference is what you tell `clone` to share (memory, files).
A thread shares, a process copies. · *3.3.1*

**10.** Most likely in state **`D` (uninterruptible sleep)**, waiting for **disk/NFS I/O**. On Linux
load counts processes in R + D; high load + low CPU = an I/O bottleneck. · *3.2.1, 3.2.3*

**11.** **namespace** manages "what it can see" (an isolated PID/network/file-system view); **cgroup**
manages "how much it can use" (a CPU/RAM/IO quota). Container = the combination of the two. · *3.6.3,
3.6.4*

**12.** Because if every signal could be caught, a program could ignore all signals and become
**unkillable**. SIGKILL (kill) and SIGSTOP (stop) are the "last word" the kernel can apply without
giving the process a say. · *3.4.1*

**13.** Because the application doesn't catch SIGTERM, it **dies instantly** on the SIGTERM that
`docker stop` sends; the requests in progress are cut off and the proxy in front returns 502. The
fix: add a SIGTERM handler to the app and do a graceful shutdown (stop accepting new requests, finish
existing ones, then exit). · *3.4.2, Answer 3.3*

**14.** (a) The real process had spawned a child process; the parent died but the child lives and was
adopted by PID 1 (PPID=1). (b) `systemctl stop` is targeting the wrong process, or the service's real
main process (MainPID) is a different one; the process you see is related but not attached to the
service. In both cases you need `pstree -p` and to find the right PID. · *3.1.1, 3.8*

**15.** The container's **cgroup memory limit** was full (not the host's RAM). The kernel triggered an
OOM inside the group and killed the process. You read the real limit not from the host's `free` but
from the cgroup: in cgroup v2, `/sys/fs/cgroup/.../memory.max` and `memory.current`. · *3.6.2, 3.6.4,
Answer 3.4*

**16.** (1) `top`, sort by CPU with `P`, take the top PID; (2) `ps -p <PID> -o
pid,ppid,user,stat,cmd` for who/whose child/which state; (3) `/proc/<PID>/cmdline` and
`/proc/<PID>/fd/` for exactly what it's doing (which files/sockets); (4) decide: necessary work
(renice/add resources) or runaway (SIGTERM, then SIGKILL if needed). · *3.7.3*

**17.** `pgrep -a worker` — to see in advance exactly **whom** `pkill` will hit. If the pattern
matches more broadly than expected (if `worker` also appears in other services) you kill the wrong
processes; `pgrep` shows this first. · *3.7.1*

**18.** You look at the application's thread count with `ps -o nlwp -p <PID>` (or `ps -eLf`) and the
core count with `nproc`. If the application is single-threaded it uses a single core at a time; the
extra vCPUs sit idle. Concept: parallelism depends on the application, the vCPU count is only an
upper bound. · *3.3.2*

---

## Scoring

| Number correct | Assessment |
|---|---|
| 16–18 | You answer "what is the system doing" from the model. Move on to Phase 4. |
| 12–15 | Solid. Take a pass through the sections of the questions you missed. |
| 8–11 | The basics are in place but there are mechanism gaps. Use the table below. |
| 0–7 | Repeat the phase from the start, running the 🔧 boxes (especially 3.2, 3.4, 3.6) on your machine. |

**Which question you missed → which section to return to:**

| Question you missed | Return to |
|---|---|
| 1, 14 | 3.1.1 (PID, PPID, init, orphan/adoption) |
| 2, 8 | 3.2.2 (zombie and reaping) |
| 7, 10 | 3.2.1, 3.2.3 (states, D-state, load average) |
| 9, 18 | 3.3 (thread vs process, vCPU) |
| 3, 12, 13 | 3.4 (signals, SIGTERM/SIGKILL, graceful shutdown) |
| 4 | 3.5.2 (nice and priority) |
| 5, 6, 15 | 3.6 (ulimit, cgroup, container limits) |
| 11 | 3.6.3–3.6.4 (namespace, the container equation) |
| 16, 17 | 3.7 (observation tools, the "who is eating the CPU" reflex) |

---
---

# Phase 3 — Closing and bridge to Phase 4

## What you carry from this phase

In Phase 2 you saw that every process has an **identity**. Phase 3 opened up the process **itself**:
how it's born, what state it's in, how it dies, and how its resources are limited. Now when you log
into a server and open `top`, you read the table you see not from memory but from the model. You
carry four tools with you:

1. **The process tree.** Every process has a PID, a PPID and a parent; the root is PID 1 (systemd).
   It's born with `fork`/`exec`, and when orphaned PID 1 adopts it. "PPID=1" is now a signature.
2. **The state machine.** R/S/D/T/Z — a healthy system is mostly `S`; for `Z` (zombie) look for the
   parent, for `D` (locked) look for storage. Load average = R + D workload, not "CPU busy".
3. **Signals and stopping correctly.** SIGTERM (gentle) → wait → SIGKILL (force). Graceful shutdown
   and "when a container shuts down, SIGTERM comes first" won't be forgotten again.
4. **Resource limits.** `ulimit` is a per-process quota, cgroup is a per-group quota; namespace is
   isolation. **Container = process + cgroup + namespace** — not magic.

## Where Phase 4 connects to this

Phase 4 carries the same "what is the system doing" question onto the **resource** axis: memory, I/O,
and performance intuition. Phase 3's process model connects there directly:

| What you learned in Phase 3 | What Phase 4 adds on top |
|---|---|
| `ps` has `%MEM`, `VSZ`, `RSS` columns | RSS vs VSZ difference, page cache, the "used memory" fallacy |
| When a cgroup memory limit is exceeded, OOM happens | How the OOM Killer chooses, swap, reading OOM in `dmesg` |
| load average is the count of R + D processes | Reading load in depth, `%wa` (I/O wait), bottleneck triage |
| State `D` is a disk I/O wait | `iostat`, IOPS, tying a disk bottleneck to a process |
| The `%Cpu` line in `top` is us/sy/id/wa | Turning these metrics into a "which resource is the bottleneck" decision |

> **Phase 3 output — before moving on, ask yourself:**
> You opened `top` on a server and saw this:
>
> ```
> top - up 3 days,  load average: 6.20, 5.90, 4.10
> Tasks: 210 total,   1 running, 206 sleeping,   0 stopped,   3 zombie
> %Cpu(s):  4.0 us,  2.0 sy,  0.0 ni, 30.0 id, 63.5 wa, ...
> ```
>
> 1. Load is 6.2 but most of the CPU is `wa` (I/O wait) and `id` (idle). Is the bottleneck CPU or
>    disk? How can you tell?
> 2. Is `3 zombie` a problem? When does it become a problem, what do you look at first?
> 3. On this machine, with which two commands do you narrow down "which process is waiting on the
>    disk"?
>
> If you can answer these three without hesitation, the process and resource model has settled —
> Phase 4 will open up what's under this `wa` line.
>
> **Lab 3 idea:** On your own machine, start a few background jobs with `sleep 300 &`, manage them
> with `jobs`/`fg`/`bg`. See a service's threads with `ps -eLf`. Read a process's state and memory
> with `cat /proc/<PID>/status`. Then watch a cgroup limit trigger OOM **live** with `systemd-run
> --user --scope -p MemoryMax=100M stress --vm 1 --vm-bytes 200M` — the most illuminating five
> minutes of this phase. When done, clean up the background jobs with `kill`.

---

> **Navigation:** [◀ Checkpoint Quiz 1](Checkpoint_Quiz_1.md) · **Phase 3** · [Phase 4 — Memory, I/O and Performance ▶](Phase_4_Memory_IO_Performance.md)


