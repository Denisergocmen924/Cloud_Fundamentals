# Phase 0 — Mental Model: Why and How Linux?

> **Navigation:** [◀ Contents](README_en.md) · **Phase 0** · [Phase 1 — Shell and File System ▶](Phase_1_Shell_and_File_System.md)

---

## Why does this phase exist?

A cloud engineer's day is full of sentences like these:

- "I sent `kill -9` and the process is still there. How?"
- "Load average is 38 but CPU is at 3%. What is the machine doing?"
- "Where does `free` get these numbers from?"
- "My user-data script blew up on Amazon Linux. It worked on Ubuntu."
- "SSH says `Permission denied (publickey)`, but the key is right."

They look like five separate problems. But all five answers rest on the same three ideas:

1. A machine has **two separate worlds**: the kernel and user space.
2. There is **a single door** between those worlds: the syscall.
3. The Linux kernel shows you the world through a **file interface**.

This phase builds those three ideas. It does not teach commands, it teaches a **model**. All
twelve phases after this one stand on that model. When you work out why a process is stuck in
"D" state in Phase 3, why `free` looks misleading in Phase 4, or why root cannot do everything
in Phase 9, you will keep coming back here.

> **Honest warning:** You will run few commands in this phase, and feeling "I wanted to learn
> to use Linux, why are we talking about the kernel?" is normal. Think of it this way: someone
> who memorizes commands goes back to Google when a new error shows up. Someone who built the
> model **guesses where the error could be.** The cloud needs the second person.

---

## By the end of this phase

- You will be able to name the kernel's three basic jobs (abstraction, sharing, protection)
- You will be able to explain why an application cannot write to disk **directly**
- You will be able to answer "What happens when an application wants to write to disk?" in
  syscall steps
- You will be able to say, with the mechanism, why `kill -9` sometimes **does not work**
- You will know what `/proc` and `/sys` are and where tools like `top` and `free` get their data
- You will know **where** the difference between Ubuntu and Amazon Linux lies (and where it
  does not)

---

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 0.1 | The job of an operating system | `[concept]` | Why the kernel exists |
| 0.2 | Kernel space vs user space, syscall | `[mechanism]` | **The heart of the phase** — the foundation of the whole roadmap |
| 0.3 | "Everything is a file", `/proc`, `/sys` | `[concept]` | The universal interface for reading Linux |
| 0.4 | The distro landscape and AMIs | `[concept]` | The first decision made every day in the cloud |
| 0.5 | When this phase breaks | — | First seeds of a failure instinct |

---
---

# 0.1 The Job of an Operating System

## 0.1.1 Kernel — the layer that manages and shares the hardware `[concept]`

Hundreds of programs run on a computer at the same time: the SSH server, a web service, a log
collector, a monitoring agent, your shell. But the machine usually has a few CPU cores, a
single pool of RAM, one or two disks and one network card.

Someone has to answer questions like these:

- Which program uses the CPU right now? For how long?
- Which part of RAM belongs to whom?
- What happens if two programs write to the same file at the same time?
- A packet arrived at the network card. Which program does it go to?

The program that answers them is called the **kernel**. The kernel is the first program loaded
when the machine starts, and it runs until the machine shuts down. Every other program runs
with its permission and through it.

The kernel's job fits into three words:

| Job | What it means | Example |
|---|---|---|
| **Abstraction** | Hiding hardware complexity behind a simple interface | A program says "write to a file"; it does not know whether it is NVMe, SATA or EBS |
| **Sharing** | Dividing a limited resource among many programs | 4 CPU cores shared among 300 processes thousands of times per second |
| **Protection** | Protecting programs from each other and from the hardware | A bug in one program cannot corrupt another's memory |

> **Summary:** The kernel is a **referee**. It owns the resources; programs only request to use
> them. The rest of this phase is about **how** that refereeing is done.

**❓ A question that comes to mind: Is "Linux" an operating system, or only the kernel?**

Technically, **Linux is only the kernel.** What Linus Torvalds wrote in 1991 was a kernel.
The rest of what you call "Linux" — `bash`, `ls`, `cp`, `grep`, the C library (glibc), the
package manager, systemd — comes from other projects. The organization that puts them all
together, tests, packages and ships them is a **distribution (distro)**: Ubuntu, Debian,
Amazon Linux, RHEL. We return to this split in 0.4.

Saying "Linux server" in everyday language is perfectly correct. But when you hunt a bug, being
able to ask "is this a kernel problem or a problem with some part of the distro?" pays off.

> **🔧 See it on your machine** 🟢 — Which kernel are you running?
>
> ```
> $ uname -r
> 6.8.0-1015-aws
> ```
>
> - `6.8.0` → the kernel's main version (Linux 6.8)
> - `1015` → Ubuntu's package revision of this kernel
> - `aws` → the kernel flavor **tuned for AWS**. On desktop Ubuntu you see `generic` here.
>
> ```
> $ uname -a
> Linux ip-172-31-20-14 6.8.0-1015-aws #16-Ubuntu SMP Mon Aug 19 19:38:17 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
> ```
>
> The `x86_64` at the end of the line is the architecture. On a Graviton (ARM) instance you see
> `aarch64`. The version and date in your output will differ; what matters is being able to read
> the fields.

---

## 0.1.2 Why can't programs access the hardware directly? `[concept]`

Let's do a thought experiment. What if there were no kernel and every program could touch the
hardware directly?

**Scenario 1 — Two programs, one disk.** The database writes data to sector 5,000,000 of the
disk. At the same moment the log collector writes to the same sector, believing it is "free".
Neither knows. Result: a corrupted database. No error message.

**Scenario 2 — The curious program.** A web application has a vulnerability. An attacker runs
code and reads RAM from start to end, finding the SSH server's private key in memory.

**Scenario 3 — The infinite loop.** A program has a bug and enters a `while(true)` loop. It never
lets go of the CPU. Everything else on the machine — SSH included — freezes. You cannot even
connect to the server to kill the program.

The common solution to all three: **programs cannot touch the hardware; they can only ask the
kernel.**

- The kernel manages disk sectors. Programs do not see sectors, they see **files**.
- The kernel hands out RAM. Each program only sees the memory given to it.
- The kernel shares the CPU. No matter how stubborn a program is, the kernel **forcibly takes
  the CPU back** from it at intervals (preemption).

> **⚠️ Common misconception: "The kernel is a program and applications are programs. So the
> kernel is just a program that started earlier."**
>
> No. What makes the kernel special is not the start order but **the privilege the CPU gives
> it.** The kernel runs in the processor's privileged mode; applications run in restricted
> mode. This split is enforced by **hardware**, not software. That is exactly what 0.2 is about.

![Figure 0.1 — The layers of a Linux system](../diagrams/png/lx-0-01-os-layers.png)
*Figure 0.1 — Three layers: hardware at the bottom, the kernel in the middle as the only thing
that can touch the hardware, and programs on top asking the kernel for work. The kernel is the
only path between programs and hardware.*

> **🤔 Think 0.1**
> An EC2 instance is really a virtual machine running on a physical server. Other customers'
> instances run on the same physical server. On your instance, `uname -r` shows a kernel version.
>
> **Whose** kernel is that? Yours, or the physical server's? And what would you see if you ran
> `uname -r` inside a Docker container?
> *(Answer: at the end of the phase)*

---
---

# 0.2 Kernel Space vs User Space

## 0.2.1 Two worlds of privilege `[mechanism]`

A modern processor can run in at least two modes:

| Mode | Other names | Who runs | What it can do |
|---|---|---|---|
| **Privileged mode** | kernel mode, ring 0, supervisor mode | Only the kernel | Everything: send commands to hardware, change the memory map, enable and disable interrupts |
| **Restricted mode** | user mode, ring 3 | All applications (root's included) | Only computation and access to its own memory |

The two memory regions matching these modes are called **kernel space** and **user space**.

The processor itself enforces the split. If a program in restricted mode tries one of these,
the processor **does not execute** the instruction and immediately hands control to the kernel:

1. **Executing a privileged instruction** — for example sending a command directly to the disk
   controller or disabling interrupts.
2. **Touching a memory address that does not belong to it** — another program's memory or the
   kernel's memory.

The kernel usually kills the program in that case. We have all seen it:

> **🔧 See it on your machine** 🟢 — Touching a forbidden address
>
> The command below tells Python to read memory **address 0**. That address is never given to
> any program.
>
> ```
> $ python3 -c "import ctypes; ctypes.string_at(0)"
> Segmentation fault (core dumped)
> ```
>
> - The Python interpreter was running in restricted mode.
> - It tried to access address 0. The processor stopped the access and notified the kernel.
> - The kernel sent the process a **SIGSEGV** (segmentation violation) signal and the process
>   died.
> - The rest of the system felt nothing. **This is exactly what protection looks like.**
>
> This command only crashes the Python process it starts; it does not harm your system.

Each process sees its own **virtual address space**. Process A's address `0x7ffd1000` and the same
address in process B are **different pieces of physical RAM**. The kernel keeps that mapping (the
page tables) separately for each process, and processes cannot change it. That is why a process
cannot read someone else's memory — even if it knows the address. (We come back to this virtual
memory model in Phase 4.1.)

> **⚠️ Common misconception: "The root user runs in kernel mode."**
>
> **Wrong — and one of the most important corrections in this roadmap.** Every program you run as
> root also runs in **user space**, in restricted mode. Root's difference is not at the processor
> level but at the level of **the kernel's permission checks**: when the kernel receives a
> request it checks "is the requester UID 0?" and says "yes" to root more often.
>
> So even root does not touch the hardware directly; it can only ask the kernel for more things.
> If you run the Python command above with `sudo`, you get **the same segmentation fault**.
>
> This distinction matters a lot in Phase 2 (permissions) and Phase 9 (capabilities, AppArmor):
> the cases where even root can be blocked are possible for exactly this reason.

---

## 0.2.2 Syscall — the only door between the two worlds `[mechanism]`

If programs cannot touch the hardware, how do they write to a file? How do they send packets
over the network? How do they print text on the screen?

By **asking** the kernel. That request is called a **syscall** (system call = a user program
asking the kernel to do work).

Linux has a few hundred syscalls, each with a number and a name. The ones you will see most:

| Syscall | What it asks for |
|---|---|
| `openat` | Open a file and give me a number (file descriptor) |
| `read` / `write` | Read from / write to the file with this number |
| `close` | I am done with this file |
| `mmap` | Give me memory (or map a file into memory) |
| `clone` / `execve` | Create a new process / load another program into this process |
| `socket` / `connect` | Open a network connection |
| `exit_group` | The process is finished |

Let's follow a `write` call step by step. A program wants to print `hello` on the screen:

```
USER SPACE (restricted mode)
  1. The program calls the write() function in the C library:
     write(1, "hello\n", 6)
  2. The library places the syscall number (write = 1 on x86-64) and the
     three arguments in CPU registers.
  3. The library executes the "syscall" machine instruction.
─────────────────────── mode switch ───────────────────────
KERNEL SPACE (privileged mode)
  4. The CPU switches to privileged mode and jumps to the kernel's fixed
     entry point. (The program cannot choose where to jump — the door is
     single and fixed.)
  5. The kernel checks: is file descriptor 1 open in this process? Is
     writing allowed? Does the buffer address really belong to this process?
  6. The kernel does the work: it hands the 6 bytes to the terminal driver.
  7. It puts the result (bytes written: 6) in a register.
─────────────────────── mode switch ───────────────────────
USER SPACE
  8. The CPU returns to restricted mode and the program continues where it
     left off. write() returns 6.
```

![Figure 0.2 — The journey of a write() syscall](../diagrams/png/lx-0-02-syscall-path.png)
*Figure 0.2 — A write() call: the program calls the library, the syscall instruction switches the
CPU into kernel mode, the kernel checks the request and does the work, and the result returns to
user space.*

The flow has two critical properties:

**1. The door is single and fixed.** A program cannot say "jump to this kernel function". It can
only say "I want service number such-and-such". There is no other way into the kernel. This is the
foundation of security.

**2. The kernel checks every request.** The checks in step 5 are **where the permission model of
Phase 2 is enforced.** A "Permission denied" error is a syscall rejected by the kernel — the error
code comes back as `EACCES` or `EPERM`, and the program turns it into a message.

> **🔧 See it on your machine** 🟢 — Tracing a program's syscalls
>
> The `strace` tool prints every syscall a program makes. It is usually installed on Ubuntu
> servers. If not: `sudo apt install strace` (🔴 permanent: installs a package; undo with
> `sudo apt remove strace`).
>
> ```
> $ strace -e trace=write echo hello
> write(1, "hello\n", 6hello
> )                  = 6
> +++ exited with 0 +++
> ```
>
> The output looks a bit messy because two things are writing to the same screen:
> - `strace` starts writing its own line: `write(1, "hello\n", 6`
> - Right then `echo`'s output, `hello` and a newline, lands on the screen
> - `strace` finishes the line: `) = 6` → the kernel reports 6 bytes written
> - `+++ exited with 0 +++` → the process finished with exit code 0 (success)
>
> `-e trace=write` shows only `write` calls. Remove the filter and you see **dozens** of syscalls
> even for this one-word `echo`: loading the program, opening libraries, allocating memory...

> **🔧 See it on your machine** 🟢 — Syscall statistics
>
> ```
> $ strace -c ls / > /dev/null
> % time     seconds  usecs/call     calls    errors syscall
> ------ ----------- ----------- --------- --------- ----------------
>  28.57    0.000086           8        10           mmap
>  15.95    0.000048           6         8           openat
>  11.63    0.000035           3        10           close
>  ...
>   5.32    0.000016           8         2         2 statfs
> ------ ----------- ----------- --------- --------- ----------------
> 100.00    0.000301           4        68         4 total
> ```
>
> - `-c` counts how many times each syscall was called.
> - Even a simple `ls /` makes ~70 syscalls.
> - The `errors` column counts **failed** syscalls. They are not always a problem: programs often
>   try "does this file exist?" and treat "no" as normal.
> - Your numbers will differ. What matters is being able to read the table.
>
> In Phase 11 we will call `strace` **"the final arbiter"**: logs can lie about what a program
> does, documentation can be stale — but the syscall list does not lie.

**❓ A question that comes to mind: Is every `print()` or `printf()` a syscall?**

No, and the difference matters. A mode switch has a cost. A single simple syscall takes roughly
**hundreds of nanoseconds** on a modern processor; security mitigations (for Spectre/Meltdown) can
increase that. A normal function call takes a few nanoseconds.

So the C library and languages like Python first collect the text you write **in their own
buffers**, and make **one** `write` syscall when the buffer fills (or the line ends, or the program
exits). A thousand lines of output is not a thousand syscalls; it may be a few.

That is why you sometimes cannot see a program's output right away: the output is still waiting
in the buffer. In Python, `print(..., flush=True)` or `python3 -u` turns buffering off. (In Phase 8,
this is exactly why a service's logs appear late in `journalctl`.)

> **🤔 Think 0.2**
> A team wrote its own logging library. The library writes **every character** of a log line to the
> file with a separate `write()` call. The average log line is 200 characters, and the application
> produces 5,000 lines of logs per second.
>
> What does this design cost? If a syscall takes ~0.5 microseconds (500 ns), how much CPU time per
> second does the application spend just writing logs? And in which column of `top` would you see
> it?
> *(Answer: at the end of the phase)*

---

## 0.2.3 What happens when an application wants to write to disk? `[mechanism]`

This is the phase's output question. Let's build it end to end with what we have learned so far.
A web application appends a line to `orders.log`:

**Step 1 — Open the file (`openat`).**
The application says "open `/var/log/app/orders.log` for writing". The kernel:
- Resolves the path piece by piece: `/` → `var` → `log` → `app` → `orders.log`
- Checks at each directory whether this process may pass through
- Checks whether the file may be written
- If everything is fine, gives the process a small integer: a **file descriptor** (an open-file
  number), for example `3`

**Step 2 — Write (`write`).**
The application says "write these 120 bytes to file 3". The kernel:
- **Copies** the data from the user-space buffer into kernel space
- Puts the data into the **page cache** (the in-RAM copy of disk data) and marks that page "dirty"
  (not yet written to disk)
- **Returns immediately:** "120 bytes written"

**Note: at this point the data is not on disk yet.** It is in RAM. The application got a "written"
answer, but if the power went out now, this line would be lost.

**Step 3 — The kernel writes to disk in the background (writeback).**
The kernel's own threads write dirty pages to disk at intervals (on Linux, by default a dirty page
waits at most ~30 seconds) or when the amount of dirty pages crosses a threshold. For that, the
file system (ext4, xfs) decides which disk block the data goes to, the block layer passes the
request to the driver, and the driver talks to the hardware.

**Step 4 (optional) — "Write it to disk now" (`fsync`).**
Programs that cannot accept data loss, like databases, make an `fsync` syscall. It **does not
return until the data is really on disk.** Safe but slow.

**Step 5 — Close (`close`).**
The file descriptor is released.

```
Application      Kernel                            Disk
   │  openat ───▶ resolve path, check permissions
   │  ◀─── fd=3
   │  write ────▶ user buffer → page cache (dirty)
   │  ◀─── 120    (returned — data is in RAM)
   │                   ...  within ~30 s  ...
   │              writeback: file system → block ───▶ written
   │  close ────▶ fd released
```

> **Why is it designed like this?** Because RAM is thousands of times faster than disk. If every
> `write` really went to disk, every log line would take milliseconds. Thanks to the page cache,
> applications "write" at RAM speed and the kernel carries the writes to disk in batches. The price:
> in a crash, the last few seconds of data can be lost. In Phase 4.2 we meet the page cache again,
> from the side that "makes RAM look full".

**❓ A question that comes to mind: Why does this matter in the cloud?**

Two concrete reasons:

1. **If an instance stops suddenly** (hardware failure, a spot instance being reclaimed), unwritten
   data in the page cache is lost. That is why databases use `fsync`, and why "disk write speed" is
   the heart of database performance.
2. **When you take a snapshot of an EBS volume**, data waiting in the page cache is not in the
   snapshot. For a consistent snapshot you first need to stop the application or freeze the file
   system (`fsfreeze`). We return to this in Phase 6.

---

## 0.2.4 When it breaks: D state — the process even `kill -9` cannot kill `[mechanism]`

We have reached the roadmap's first "When it breaks" point. A real scenario:

> On an EC2 instance, the application writes to an NFS directory (a disk shared over the network).
> The NFS server stops responding. The application freezes. You type `kill -9 <pid>`. Nothing
> happens. The process is still there.

How can that be? Wasn't `kill -9` supposed to mean "kill unconditionally"?

**Mechanism:** Signals (covered in detail in Phase 3.4) are not "injected" into a process instantly.
The kernel puts a **mark** on the process saying "you got a SIGKILL". The process notices the mark
at certain safe points inside the kernel — typically when returning from a syscall to user space,
or while in an **interruptible** wait — and dies.

But if the process is in the **middle** of a syscall, waiting inside the kernel for an I/O
operation to finish, and that wait is marked **uninterruptible**, the kernel does not wake it. Why?
Because cutting that wait short could leave the kernel's own data structures (for example, a
half-done disk operation) inconsistent.

The state letter of a process in this condition is **`D`** (uninterruptible sleep).

```
Normal case:               D state:
  kill -9 ──▶ mark         kill -9 ──▶ mark
  process wakes up         process waits for I/O in the kernel
  sees the mark            (uninterruptible) — CANNOT see the mark
  dies ✓                   stays there until the I/O finishes ✗
```

**Result:** A process in D state does not die until the I/O it waits for **completes** or **times
out**. If the NFS server comes back, the operation finishes, the process sees the signal and dies.
If it does not, the process can stay there — sometimes until the machine is rebooted.

> **🔧 See it on your machine** 🟢 — Finding processes in D state
>
> ```
> $ ps -eo pid,stat,wchan:32,comm | awk 'NR==1 || $2 ~ /^D/'
>     PID STAT WCHAN                            COMMAND
> ```
>
> - `stat` → process state. If the first letter is `D`, it is in uninterruptible sleep.
> - `wchan` → **which kernel function** the process waits in (wait channel)
> - `awk` shows the header line and lines whose state starts with D
>
> **On a healthy machine you should see nothing but the header** — or a few processes that write
> to disk appear briefly and vanish.
>
> On a troubled machine you see something like this:
>
> ```
>     PID STAT WCHAN                            COMMAND
>    4312 D    rpc_wait_bit_killable            python3
>    4318 D    rpc_wait_bit_killable            python3
>    4401 D    nfs_wait_on_request              rsync
> ```
>
> wchan values starting with `rpc_*` and `nfs_*` **tell you directly** that the problem is NFS. For a
> local disk problem you see names like `io_schedule` or `blk_*`, `ext4_*`, `xfs_*`.

The kernel logs long D states. On a troubled machine you see this line in `dmesg`:

```
INFO: task python3:4312 blocked for more than 120 seconds.
```

When you see that line, the thought should be: **"The problem is not the process, it is the storage
under it."** Trying to kill the process is wasted effort; you need to fix what it waits for (the NFS
server, the EBS volume, a failing disk).

> **⚠️ Common misconception: "D state = the process is burning a lot of CPU."**
>
> The opposite. A process in D state **uses no CPU**, it only waits. But Linux counts these processes
> in the load average too (Phase 4.4). So you see this odd picture: **load average 40, CPU 98% idle.**
> That combination almost always means "something is stuck on I/O".

> **🤔 Think 0.3**
> On a 4-vCPU instance, the NFS server crashed. 40 processes trying to access the NFS directory are
> in D state. `uptime` shows a load average of 40, `top` shows the CPU 97% idle.
>
> (a) Why is load 40, and why is the CPU idle?
> (b) You try to restart the machine with `sudo reboot`, and the reboot **hangs for minutes.** Why
> might that be?
> *(Answer: at the end of the phase)*

---
---

# 0.3 The "Everything Is a File" Philosophy

## 0.3.1 One interface, many things `[concept]`

Look at the syscall table in 0.2.2 again: `openat`, `read`, `write`, `close`. The most powerful
design idea of Unix (and Linux) is this: **these four syscalls work for almost everything.**

| What | How it appears as a "file" | What reading means |
|---|---|---|
| Regular file | `/home/ubuntu/notes.txt` | Reading data on disk |
| Disk | `/dev/nvme0n1` | Reading the disk's raw bytes |
| Terminal | `/dev/pts/0` | Reading keys from the keyboard |
| Random number generator | `/dev/urandom` | Reading random bytes |
| Trash can | `/dev/null` | Swallows everything written; returns empty when read |
| Process information | `/proc/1234/status` | Reading the process's state |
| Pipe | the channel in `ls \| grep` | Reading the other program's output |
| Network socket | a file descriptor | Reading data from the network |

When a program says `read(fd, buffer, 100)`, it **does not need to know** whether a file, a terminal
or a network connection is behind that `fd`. That is why `grep` works the same on a file, on a pipe
and on data coming from a network connection. It is the foundation of Phase 1's "connect small
tools together" philosophy.

> **🔧 See it on your machine** 🟢 — File types
>
> ```
> $ ls -l /etc/hostname /dev/nvme0n1 /dev/null /bin
> lrwxrwxrwx 1 root root         7 Apr 22  2024 /bin -> usr/bin
> crw-rw-rw- 1 root root      1, 3 Sep 15 08:12 /dev/null
> brw-rw---- 1 root disk    259, 0 Sep 15 08:12 /dev/nvme0n1
> -rw-r--r-- 1 root root        16 Sep 15 08:12 /etc/hostname
> ```
>
> The **first character** of each line is the file type:
>
> | Character | Type | Example |
> |---|---|---|
> | `-` | Regular file | `/etc/hostname` |
> | `d` | Directory | `/etc` |
> | `l` | Symbolic link (shortcut) | `/bin → usr/bin` |
> | `c` | Character device (flows byte by byte) | `/dev/null`, terminals |
> | `b` | Block device (in blocks, random access) | `/dev/nvme0n1`, `/dev/sda` |
> | `s` | Socket | `/run/systemd/journal/socket` |
> | `p` | Named pipe (FIFO) | rare |
>
> For device files you see two numbers instead of a size (`1, 3` and `259, 0`). These are the
> **major, minor** numbers that tell which driver the kernel attached this device to.
>
> On your machine the disk may be `/dev/sda` or `/dev/xvda` instead of `/dev/nvme0n1`; run the command
> with your own disk's name (`lsblk` shows the disk name — Phase 6.1).

**Seeing file descriptors.** Every process keeps a number for everything it opened. The first three
numbers are always the same: `0` = standard input, `1` = standard output, `2` = standard error.
(We cover these three in detail in Phase 1.4.)

> **🔧 See it on your machine** 🟢 — Your own shell's open files
>
> ```
> $ ls -l /proc/$$/fd
> total 0
> lrwx------ 1 ubuntu ubuntu 64 Sep 15 09:40 0 -> /dev/pts/0
> lrwx------ 1 ubuntu ubuntu 64 Sep 15 09:40 1 -> /dev/pts/0
> lrwx------ 1 ubuntu ubuntu 64 Sep 15 09:40 2 -> /dev/pts/0
> lrwx------ 1 ubuntu ubuntu 64 Sep 15 09:40 255 -> /dev/pts/0
> ```
>
> - `$$` → the shell's own PID
> - `0`, `1`, `2` → all three are attached to your terminal (`/dev/pts/0`): reads the keyboard,
>   writes to the screen
> - `255` → a copy bash keeps for its own internal use
>
> Run the same command on a web server and you see hundreds of lines: log files, listening sockets,
> one connection per client. The `lsof` tool in Phase 11 shows exactly this list in readable form.

> **⚠️ Common misconception: "Everything is a file = everything is stored on disk."**
>
> No. "File" here is an **interface**, not a storage place. `/dev/null`, `/proc` or a socket take no
> space on disk. Read "everything is a file" as "everything is spoken to with the same four syscalls".
>
> And honestly, it is a **philosophy, not an absolute rule.** For example, network interfaces (`eth0`)
> have no file under `/dev`; setting up a network connection needs special syscalls like `socket` and
> `connect`. But once the connection is up, it is spoken to with `read`/`write` again.

---

## 0.3.2 `/proc` — reading the kernel's live state like a file `[concept]`

`/proc` is a **virtual** file system that is not on disk. The "files" in it are **generated** by the
kernel at the moment you read them. What you read is not a record but a live snapshot of the kernel's
current state.

> **🔧 See it on your machine** 🟢 — The odd size of files in `/proc`
>
> ```
> $ ls -l /proc/meminfo /proc/uptime
> -r--r--r-- 1 root root 0 Sep 15 09:41 /proc/meminfo
> -r--r--r-- 1 root root 0 Sep 15 09:41 /proc/uptime
> ```
>
> Size **0**. Because the file's content does not exist yet — it is created when read.
>
> ```
> $ cat /proc/uptime
> 4211.73 16538.20
> ```
>
> - First number: the machine has been up **4,211 seconds** (~70 minutes)
> - Second number: the total time all CPU cores spent idle
>
> Run it again two seconds later; the numbers will have changed. The file is "live".

The `/proc` files you will use most:

| File | What it tells | Which tool reads it |
|---|---|---|
| `/proc/cpuinfo` | CPU model, core count, features | `lscpu`, `nproc` |
| `/proc/meminfo` | Memory state, in detail | `free`, `top` |
| `/proc/loadavg` | Load average and process counts | `uptime`, `top` |
| `/proc/uptime` | Uptime | `uptime` |
| `/proc/mounts` | Mounted file systems | `mount`, `df` |
| `/proc/<PID>/` | Everything about one process | `ps`, `top`, `lsof` |
| `/proc/sys/` | Changeable kernel settings | `sysctl` |

Look at the last column: **`free`, `top`, `ps`, `uptime` know nothing on their own.** They are all
programs that read `/proc` and present it nicely. Let's prove it:

> **🔧 See it on your machine** 🟢 — Where does `free` get its data?
>
> ```
> $ strace -e trace=openat free 2>&1 | grep proc
> openat(AT_FDCWD, "/proc/meminfo", O_RDONLY) = 3
> ```
>
> - `strace` listed the files `free` opened (`2>&1` redirects strace's output into the pipe —
>   Phase 1.4)
> - `free` opened a single file: `/proc/meminfo`. **That is all it knows.**
>
> ```
> $ head -4 /proc/meminfo
> MemTotal:        3964584 kB
> MemFree:          283440 kB
> MemAvailable:    2987112 kB
> Buffers:           52876 kB
> ```
>
> Did you see the big gap between `MemFree` and `MemAvailable`? 280 MB "free", but 2.9 GB "available".
> That gap is all of Phase 4.2 — for now, just notice it.

The practical value of this is large: **even if no tools are installed** (a minimal container image,
rescue mode, a broken system), `cat /proc/...` always works.

> **🤔 Think 0.4**
> A monitoring agent reads `/proc/meminfo`, `/proc/loadavg` and `/proc/stat` every second. A teammate
> says "this agent reads from disk constantly and is eating our EBS IOPS quota".
>
> Are they right? Why?
> *(Answer: at the end of the phase)*

---

## 0.3.3 `/sys` — the device and driver tree `[concept]`

`/sys` is also a virtual file system, but it answers a different question. `/proc` mainly shows
**processes** and general kernel state; `/sys` shows **devices, drivers and their settings** as an
orderly tree.

> **🔧 See it on your machine** 🟢 — Network cards and disks
>
> ```
> $ ls /sys/class/net
> ens5  lo
>
> $ cat /sys/class/net/ens5/address
> 0a:1b:2c:3d:4e:5f
>
> $ cat /sys/class/net/ens5/mtu
> 9001
> ```
>
> - `ens5` → the name of the network interface on EC2 (on your machine it may have a different name,
>   like `eth0` or `enp0s3`). `lo` → loopback, the interface the machine uses to talk to itself.
> - `address` → the interface's MAC address
> - `mtu` → 9001: the "jumbo frame" size AWS supports inside a VPC. On a home network you usually see
>   1500.
>
> ```
> $ ls /sys/block
> loop0  loop1  nvme0n1
> ```
>
> The block devices in the system. (`loop` devices are virtual disks used by snap packages on Ubuntu;
> seeing them is normal.)

Some files under `/sys` and `/proc/sys` are **writable**, and writing to such a file changes the
kernel's behavior **immediately**. For example:

```
/proc/sys/vm/swappiness                      → how eager the kernel is to swap (Phase 4.3)
/proc/sys/net/ipv4/ip_forward                → whether the machine forwards packets
/sys/kernel/mm/transparent_hugepage/enabled  → huge page behavior
```

The standard tool to read and change these settings is `sysctl`:

> **🔧 See it on your machine** 🟢 — Reading a kernel setting
>
> ```
> $ sysctl vm.swappiness
> vm.swappiness = 60
>
> $ cat /proc/sys/vm/swappiness
> 60
> ```
>
> Both commands read the same thing: `sysctl` turns the dots in the name (`vm.swappiness`) into path
> separators (`vm/swappiness`) and looks under `/proc/sys`.
>
> To change it: `sudo sysctl vm.swappiness=10` 🟡 — **temporary**, reverts on reboot. To make it
> permanent the setting goes into a file under `/etc/sysctl.d/` (🔴). We do that with a reason in
> Phase 4; for now, only read.

![Figure 0.3 — "Everything is a file": one interface, many things](../diagrams/png/lx-0-03-everything-is-a-file.png)
*Figure 0.3 — The same open/read/write/close interface opens onto different things: regular files in
/home, devices in /dev, process and kernel state in /proc, device settings in /sys, pipes and sockets.*

---
---

# 0.4 The Distro Landscape

## 0.4.1 Kernel + tools + decisions = distro `[concept]`

In 0.1.1 we said "Linux is only the kernel". A working server needs these next to the kernel:

- **Basic tools:** a shell (`bash`), `ls`, `cp`, `grep` — most from the GNU project
- **C library:** the layer that makes syscalls easy for programs (glibc)
- **init system:** the first process that starts services when the machine boots (today systemd
  almost everywhere — Phase 5)
- **Package manager:** installing, updating and removing software (Phase 8)
- **Defaults:** which firewall, which security module, which log layout, which network configuration
  tool
- **Life cycle:** how many years a release gets security patches

The product of the organization that picks these parts, fits them together, tests and ships them is
called a **distribution** (distro for short).

> **Summary:** The difference between two distros is **not in the kernel but in the decisions on top
> of it.** Processes, permissions, syscalls, `/proc`, systemd — 90% of this book is the same on every
> distro.

---

## 0.4.2 Two big families `[concept]`

The distros you meet in the cloud fall into two families in practice:

| Property | Debian family | Red Hat family |
|---|---|---|
| Examples | Debian, **Ubuntu** | RHEL, Fedora, Rocky, **Amazon Linux** |
| Package format | `.deb` | `.rpm` |
| Package manager | `apt` (underlying `dpkg`) | `dnf` / older `yum` (underlying `rpm`) |
| Default EC2 user | `ubuntu` (Ubuntu), `admin` (Debian) | `ec2-user` |
| Default MAC security module | AppArmor | SELinux |
| Default firewall tool | `ufw` | `firewalld` |
| General system log | `/var/log/syslog` | `/var/log/messages` (not present by default on Amazon Linux 2023; only `journalctl`) |
| Login/auth log | `/var/log/auth.log` | `/var/log/secure` (also not present by default on Amazon Linux 2023) |

The last two rows look minor, but in real life they cost a lot of time: someone looking for
`/var/log/syslog` on Amazon Linux out of Ubuntu habit is looking at an empty spot.

> **🔧 See it on your machine** 🟢 — Which distro are you on?
>
> ```
> $ cat /etc/os-release
> PRETTY_NAME="Ubuntu 24.04.1 LTS"
> NAME="Ubuntu"
> VERSION_ID="24.04"
> VERSION="24.04.1 LTS (Noble Numbat)"
> VERSION_CODENAME=noble
> ID=ubuntu
> ID_LIKE=debian
> HOME_URL="https://www.ubuntu.com/"
> ...
> ```
>
> - `ID=ubuntu` → the distro itself
> - `ID_LIKE=debian` → **which family** it comes from. Scripts usually decide "which package manager do
>   I use?" by looking at this line.
> - `VERSION_ID` → the version. `LTS` = Long Term Support: a release that gets security patches for a
>   long time.
>
> On Amazon Linux 2023 the same file shows `ID="amzn"`, `ID_LIKE="fedora"`.

> **⚠️ Common misconception: "Ubuntu and Amazon Linux are completely different systems; someone who
> knows one struggles on the other."**
>
> No. Same kernel, same systemd, same `/proc`, same permission model, same `ps`, `top`, `ss`,
> `journalctl`. The differences **fit in one table** (above). Someone who knows one family deeply
> adapts to the other in a few hours. This book teaches on Ubuntu and adds a note wherever Amazon Linux
> differs.

---

## 0.4.3 Cloud connection: AMIs and choosing a distro `[concept]`

When you launch an instance on EC2, the first thing you pick is an **AMI** (Amazon Machine Image = a
frozen Linux image).

An AMI is roughly:

- A **copy of the root disk** (snapshot) of an installed and configured operating system
- Plus metadata: architecture (x86_64 / arm64), boot mode, which disk device it attaches as

When you launch the instance, AWS creates a new EBS volume from that copy, attaches it to the instance
and boots the machine from that disk. So **an AMI = all the layers of Phase 0, frozen**: the kernel,
tools, packages, default settings.

**Amazon Linux 2023 or Ubuntu?** What does the choice change?

| Topic | What changes |
|---|---|
| **SSH user name** | `ec2-user` vs `ubuntu`. Connecting with the wrong user name is **the most common** cause of `Permission denied (publickey)`. |
| **Package manager** | `dnf` vs `apt` — your user-data scripts and documentation are written accordingly |
| **Security defaults** | SELinux vs AppArmor (Phase 9) |
| **Logs** | `journalctl` on both; file-based logs differ (0.4.2) |
| **AWS integration** | Amazon Linux is AWS's own product; it ships with AWS tools and a kernel tuned for AWS. Ubuntu's AWS images also use an AWS-tuned (`-aws`) kernel. |
| **Community and third-party software** | Many projects write install instructions for Ubuntu first |
| **Life cycle** | Both have long-supported releases; the team should pick a release and plan its upgrade schedule |

**Practical decision rule:** Technically both are good choices. The decision is usually made by **the
team's habits**, the distro **officially supported** by the software you run, and the company's
**standard image**. What matters is picking one standard and using it everywhere — in an environment
where different teams use different distros, every failure diagnosis takes twice as long.

> **🤔 Think 0.5**
> A team gave a user-data script (the setup script an instance runs on first boot) that works fine on
> Ubuntu to an instance launched from an Amazon Linux 2023 AMI. The script starts like this:
>
> ```bash
> #!/bin/bash
> apt-get update
> apt-get install -y nginx
> systemctl enable --now nginx
> echo "setup done" > /home/ubuntu/status.txt
> ```
>
> The instance went to "running", but the website does not load. (a) Which lines fail, and why?
> (b) Why might the script have run to the end instead of failing and stopping? (c) Where inside the
> instance do you see these errors?
> *(Answer: at the end of the phase)*

---
---

# 0.5 When this phase breaks — failure signatures

The roadmap's "When it breaks" axis. Each row is a symptom you will see in production and the
mechanism behind it. You have not learned to **fix** these yet; the goal is to think of the right
layer when you see the symptom.

| Symptom | Likely mechanism | Where it was covered | First place to look |
|---|---|---|---|
| `kill -9` does not kill the process | The process is in an uninterruptible I/O wait in the kernel (D state) | 0.2.4 | `ps -eo pid,stat,wchan:32,comm`, `dmesg` |
| Very high load average, idle CPU | Processes piled up in D state (I/O stuck) | 0.2.4 | Processes in D state and their wchan values |
| "blocked for more than 120 seconds" in `dmesg` | Storage (NFS, EBS, disk) is not responding | 0.2.4 | Not the process — the storage under it |
| Program crashed with "Segmentation fault" | The program touched an address it does not own; the kernel sent SIGSEGV | 0.2.1 | The program's own bug; the system is healthy |
| Ran it with `sudo`, still "Permission denied" / "Operation not permitted" | Root is in user space; the kernel or a security layer still rejected the request | 0.2.1 | The decision tree of Phase 2 and Phase 9 |
| Application said "written", instance crashed, data gone | The data was in the page cache, not written to disk | 0.2.3 | Does the application use `fsync`? |
| A program's output reaches the logs late or in bursts | User-space buffering | 0.2.2 | `python3 -u`, `flush=True`, line buffering |
| `/var/log/syslog` does not exist | Distro difference (Amazon Linux 2023) | 0.4.2 | `journalctl` |
| `apt: command not found` / `dnf: command not found` | Command written for the wrong distro family | 0.4.2 | `cat /etc/os-release` |
| SSH `Permission denied (publickey)`, key is right | Wrong default user name (`ubuntu` vs `ec2-user`) | 0.4.3 | The AMI's distro and user name |
| No `top` / `free` / `ps` (minimal image) | The tools are user-space programs and may not be installed | 0.3.2 | `cat /proc/...` directly |

> **The lesson from this table:** Think of every symptom **with its layer**. "Is the process broken, is
> the kernel waiting for something, or is a decision on top (distro, user name) wrong?" This question is
> the first form of the layer-by-layer diagnosis method in Phase 11.

---

# Phase 0 — Answers to the think questions

## Answer 0.1 — Whose kernel does `uname -r` show on EC2 and in a container?

**Question:** Which kernel does `uname -r` show on an EC2 instance? What do you see in a Docker
container?

**On an EC2 instance: your own kernel.** A virtual machine (VM) runs its own operating system **together
with its own kernel**. Under the physical server there is a hypervisor (virtualization layer), and it
gives each VM virtual hardware. Your kernel boots on that virtual hardware. Another customer on the same
physical machine can run a completely different kernel (even a different distro).

```
Physical server
└── Hypervisor (AWS Nitro)
    ├── VM A: your kernel (6.8.0-aws) + your programs
    └── VM B: another customer's kernel (e.g. 6.1 amzn) + their programs
```

**In a container: the host's kernel (the machine running the container).** A container has no kernel of
its own. Processes inside a container make syscalls to **the same kernel** as the other processes on the
host. The kernel just gives them an isolated view (a different file system, a different process list).

```
EC2 instance (kernel: 6.8.0-aws)
├── normal processes              ──┐
├── container 1 (Alpine image)    ──┼── all make syscalls to the SAME kernel
└── container 2 (Debian image)    ──┘
```

So inside an Alpine-based container, `cat /etc/os-release` says "Alpine", but `uname -r` shows the host's
Ubuntu kernel. That is no contradiction: `/etc/os-release` is a **file** in the container image (user
space), while `uname -r` is a syscall asked of **the kernel**.

**Why it matters:** The misconception "a container is a lightweight VM" breaks right here. Containers
share the kernel; that is why they start fast, but a kernel vulnerability affects every container. In
Phase 3.6 we build the mechanism that provides this sharing (cgroups + namespaces).

**Related section:** 0.1.1, 0.1.2 · **Continues in:** Phase 3.6

---

## Answer 0.2 — A logging library that calls `write()` per character

**Question:** 200-character lines, 5,000 lines per second, a separate syscall per character, 500 ns per
syscall. The cost?

**Calculation:**

```
Syscalls per second   = 5,000 lines × 200 characters = 1,000,000 syscalls/s
Time per syscall      = 500 ns = 0.0000005 s
Total time            = 1,000,000 × 0.0000005 = 0.5 s / s
```

**The application spends half of every second in the kernel just writing logs.** That is 50% of one CPU
core — doing no real work.

**The right design:** Build the line in a user-space buffer and make **one** `write` **per line**:

```
5,000 syscalls/s × 500 ns = 0.0025 s/s → 0.25% of a CPU
```

**200 times** less. Even better is collecting lines and making a single `write` at intervals (but then the
last lines may be lost in a crash — always the same trade-off).

**Where you see it in `top`:** The CPU line in the `top` header has two values:
- `us` (user) → CPU time spent in user space
- `sy` (system) → CPU time spent in the kernel, that is, handling syscalls

In this scenario **`sy` is abnormally high**. On a healthy application server, `sy` is usually a small
fraction of `us`. When `sy` is high, the question to ask is: "is this program asking the kernel for
something **too often**?" `strace -c -p <PID>` answers it (Phase 11).

**Related section:** 0.2.2 · **Continues in:** Phase 4.6, Phase 11.3

---

## Answer 0.3 — NFS crashed: load 40, CPU idle, reboot hangs

**(a) Why is load 40, and why is the CPU idle?**

On Linux the load average counts: processes **running or waiting to run** **plus** processes **in D
state**. With 40 processes in D state, load is ~40.

But a process in D state is **sleeping** — it uses no CPU. So the CPU is 97% idle.

This combination (high load + idle CPU) on Linux almost always means "**something is stuck on I/O**".
Someone who reads load as "CPU load" cannot make sense of this picture. In Phase 4.4 we build the full
definition of load average.

**(b) Why does the reboot hang?**

During a restart the system stops services and tries to **detach** (unmount) file systems. Two things
hang:

1. To stop services, signals are sent to processes. Processes in D state **cannot see** those signals
   (0.2.4). The system waits a set time (in systemd typically 90 seconds per service) and then moves on.
2. To unmount the NFS directory, the processes using it must finish first and pending writes must be sent
   to the NFS server. With no server, this waits too.

Result: the reboot takes minutes. In the cloud you may need to **force stop** the instance from the
console — the equivalent of pulling the plug, and accepting that unwritten data in the page cache is lost
(0.2.3).

**The lesson:** Storage that lives "far away", like NFS and network disks, changes the **failure modes**
of a local disk. A local disk either works or returns an error; a network disk can **keep you waiting
without answering.** That is why NFS mount options (timeouts, `soft` vs `hard`) are a critical decision
in production.

**Related section:** 0.2.4 · **Continues in:** Phase 3.2, Phase 4.4, Phase 6.2

---

## Answer 0.4 — Does reading `/proc` eat disk IOPS?

**No, your teammate is wrong.**

`/proc` is a **virtual file system.** Its files are not on disk. When the agent reads `/proc/meminfo`:

1. The agent makes `openat` and `read` syscalls
2. The kernel runs the function registered for that file
3. The function reads the kernel's counters **in RAM** and turns them into text
4. The text is copied into the agent's buffer

At no step does a request go to a block device (EBS). So it does not show in EBS IOPS metrics.

**But it has a cost:** each read is a few syscalls and a bit of kernel CPU time. One read per second is
negligible. But a badly written agent that reads files under `/proc/<pid>/` for thousands of processes
several times a second can create a noticeable load on the **CPU** (in the `sy` column) — not on disk.

**The right diagnostic path:** Verify your suspicion by measuring. Do `iostat` (Phase 4) or the EBS metrics
in CloudWatch change before and after the agent starts? `strace -e trace=openat -p <agent_pid>` shows which
files the agent really opens.

**Related section:** 0.3.2 · **Continues in:** Phase 4.5, Phase 11.3

---

## Answer 0.5 — An Ubuntu user-data script on Amazon Linux

**(a) Which lines fail, and why?**

| Line | Result | Reason |
|---|---|---|
| `apt-get update` | ❌ `command not found` | Amazon Linux is in the Red Hat family; its package manager is `dnf` |
| `apt-get install -y nginx` | ❌ `command not found` | Same reason — nginx is **not installed** |
| `systemctl enable --now nginx` | ❌ `Unit nginx.service not found` | systemd exists, but nginx was not installed so there is no service file |
| `echo ... > /home/ubuntu/status.txt` | ❌ `No such file or directory` | Amazon Linux has no `ubuntu` user and no `/home/ubuntu` directory; the default user is `ec2-user` |

**(b) Why did the script run to the end?**

By default, bash **does not stop when a command fails**; it moves to the next line. Each line gives its own
error and the script ends as "finished". This is the most important lesson of Phase 10: had the script
started with `set -euo pipefail`, it would have stopped at the first failing command and failed
**visibly**. A setup that silently stops halfway is far more dangerous than one that fails visibly.

**(c) Where do you see the errors?**

User-data scripts are run by cloud-init, and their output is written to this file:

```
/var/log/cloud-init-output.log
```

In it you see the four `command not found` / `not found` / `No such file` lines. Phase 5.6 covers this file
and the whole cloud-init flow in detail.

**The lesson:** **Which distro family** a script or document was written for is the first thing to check. A
portable script either looks at `ID_LIKE` in `/etc/os-release` and behaves accordingly, or fails visibly
right at the start on an unsupported distro.

**Related section:** 0.4.2, 0.4.3 · **Continues in:** Phase 5.6, Phase 8.1, Phase 10.2

---

# Phase 0 — Frequently asked questions

This section collects questions that come up naturally while working through the phase but would break the
flow of the text.

### Q1. Do I really need to know the kernel/user space split? I just want to manage servers.

Yes, because without it you cannot **explain** these three common situations — you can only memorize them:

1. `kill -9` sometimes not working (0.2.4)
2. Getting "Operation not permitted" even with `sudo` (0.2.1 → Phase 9)
3. What a high `sy` value in `top` means (Answer 0.2)

All three are situations you will really meet in the cloud. The model ties them to one idea: **"programs
ask, the kernel decides."**

### Q2. Microkernel, monolithic kernel — what are these, and which is Linux?

Linux is a **monolithic** kernel: file systems, the network stack, drivers — all run in the same privileged
space. In a microkernel design, most of these run as separate programs in user space.

Linux stretches this with **modules**: drivers can be loaded into and removed from the kernel at runtime (you
see loaded modules with `lsmod`). But a loaded module still runs in kernel space.

This debate is at `[skip]` level for a cloud engineer. The only practical consequence you need: **a bug in a
kernel module can crash the whole machine** (kernel panic), while a bug in an application crashes only that
application.

### Q3. What is a kernel panic? How is it different from an application crash?

When an application crashes (like a segmentation fault), the kernel cleans up that process and **the system
keeps running.** If the kernel itself hits an unrecoverable error (corrupt memory, a driver bug), it cannot
continue safely and **stops.** That is a kernel panic.

In the cloud, an instance that hits a kernel panic usually fails its **status checks** and cannot be reached
over SSH. To see the evidence you use "Get system log" (serial console output) in the EC2 console — because
the machine itself is in no state to write logs.

### Q4. If `/proc` and `/sys` are virtual, why doesn't writing to a file there persist?

Because what you write does not go to disk; it changes a variable **in the kernel's RAM**. When the machine
restarts, the kernel starts from scratch and uses its default values.

Persistence needs a mechanism that **reapplies the setting at boot**: `/etc/sysctl.d/*.conf` files (for
sysctl settings), systemd services or udev rules. This is the first example of a general Linux pattern:
**"runtime state" and "persistent configuration" live in different places.** You will see the same split in
Phase 5 between `systemctl start` (now) and `systemctl enable` (at boot).

### Q5. Windows Server exists in the cloud too. Do these models apply there?

Conceptually, largely yes: Windows also has a kernel mode / user mode split, a syscall-like mechanism and
virtual memory. But the interfaces, tools and file layout are completely different; the "everything is a
file" philosophy does not exist in the same form on Windows.

This book covers only Linux. Most servers in the cloud, and **nearly all** containers and serverless services
like Lambda, run on Linux.

### Q6. Do I need to memorize syscalls?

No. Knowing **what the roughly 10 syscalls** in the 0.2.2 table ask for is enough: `openat`, `read`,
`write`, `close`, `mmap`, `clone`, `execve`, `socket`, `connect`, `exit_group`. You can learn the others you
see in `strace` output with `man 2 <syscall_name>` (manual section 2 is dedicated to syscalls).

### Q7. I ran very little in this phase. Am I ready for Phase 1?

If you can say these three sentences in your own words, you are ready:

1. "Programs cannot touch the hardware; they ask the kernel with syscalls and the kernel checks every
   request."
2. "Root is in user space too; it just passes the kernel's checks more easily."
3. "Tools like `free`, `top` and `ps` read their information from `/proc`; `/proc` is the kernel's live
   state and is not on disk."

---

# Phase 0 — Test yourself

Do not go back to the sections before writing your answers. The numbers of the questions you struggle with
point to the sections to review.

**Part A — Basics (1–6)**

1. Name the kernel's three basic jobs and give an example for each.
2. What does the word "Linux" technically refer to? What is the relationship between Ubuntu and Linux?
3. Explain the difference between kernel space and user space in two sentences.
4. What is a syscall? Why can't programs call the kernel's functions directly?
5. In `ls -l` output, what does it mean when the first character of a line is `b`, `c`, `l`, `d` and `-`?
6. Why do files under `/proc` show a size of 0?

**Part B — Mechanism (7–11)**

7. Describe step by step a `write()` call going from user space into the kernel and back.
8. An application got a "success" answer from `write()`. Where is the data right now? What happens if the
   power goes out? How can the application prevent that?
9. Why might a process that was sent `kill -9` not die? Which state letter does it show?
10. Why can't even a program running as root read another process's memory directly?
11. Where does the `free` command get memory information? How do you prove it?

**Part C — Application and reasoning (12–16)**

12. A server has a load average of 25 and CPU usage of 4%. What is your first hypothesis, and which command
    confirms it?
13. In `top`, `sy` is 45% and `us` is 10%. What does that tell you? What is your next step?
14. You try to connect to an Amazon Linux 2023 instance with `ssh -i key.pem ubuntu@<ip>` and get
    `Permission denied (publickey)`. The key is right. What is the most likely cause?
15. Inside an Alpine container, `cat /etc/os-release` says "Alpine" while `uname -r` says `6.8.0-1015-aws`.
    How is that possible?
16. `free` and `top` are not installed in a minimal container image. How do you find the memory state and
    load average?

---

## Answer key

**1.** **Abstraction** (a program says "write to a file" without knowing the disk type), **sharing** (a few
CPU cores are split among hundreds of processes), **protection** (a bug in one program cannot corrupt
another's memory). · *0.1.1*

**2.** Technically **only the kernel.** Ubuntu is a **distro**: the Linux kernel + GNU tools + C library +
systemd + package manager + default settings + a support schedule. · *0.1.1, 0.4.1*

**3.** Kernel space is the kernel's area, running in the processor's privileged mode with access to hardware.
User space is where all applications (root's included) run in restricted mode and can reach hardware only
through the kernel. · *0.2.1*

**4.** A syscall is a user program asking the kernel for work. Programs cannot jump straight into kernel
functions because the processor allows the switch from restricted to privileged mode only through **a fixed
entry point**. This guarantees the kernel checks every request. · *0.2.2*

**5.** `b` = block device (disk), `c` = character device (terminal, `/dev/null`), `l` = symbolic link,
`d` = directory, `-` = regular file. · *0.3.1*

**6.** Because their content is not stored on disk; the kernel generates the file **at the moment it is
read.** The kernel does not know how long the content will be before it is read. · *0.3.2*

**7.** (1) The program calls `write()` in the library, (2) the library puts the syscall number and arguments
in registers, (3) executes the `syscall` instruction, (4) the CPU switches to privileged mode and jumps to the
kernel's fixed entry point, (5) the kernel checks the fd, permission and buffer address, (6) does the work,
(7) puts the result in a register, (8) the CPU returns to restricted mode and the program continues.
· *0.2.2*

**8.** The data is in the **page cache** in RAM, marked "dirty"; it has not been written to disk yet. If the
power goes out it is **lost.** The application can call `fsync()` to wait until the data is really on disk
(slow but safe). · *0.2.3*

**9.** If the process is in an **uninterruptible** I/O wait inside the kernel, the kernel does not wake it and
the process cannot see the SIGKILL mark. It does not die until the I/O finishes or times out. State letter
**`D`**. · *0.2.4*

**10.** Because each process sees its own **virtual address space**, and the kernel manages how those
addresses map to physical RAM. Processes cannot see or change another process's mapping; this protection is
enforced at the processor level. Root's difference lies in the kernel's permission checks, not in processor
mode. · *0.2.1*

**11.** From the `/proc/meminfo` file. Proof: the output of `strace -e trace=openat free 2>&1 | grep proc`
shows the line `openat(..., "/proc/meminfo", ...)`. · *0.3.2*

**12.** Hypothesis: **processes are stuck in D state waiting on I/O** (disk, NFS, EBS). Confirmation: find the
processes in D state and their wchan with `ps -eo pid,stat,wchan:32,comm | awk '$2 ~ /^D/'`; look for "blocked
for more than 120 seconds" lines in `dmesg`. · *0.2.4, Answer 0.3*

**13.** Most CPU time is spent **in the kernel** — one or a few programs are asking the kernel for something
very often (an excessive number of syscalls). Next step: find the process using the most CPU in `top`, then
measure which syscall it makes how often with `strace -c -p <PID>`. · *0.2.2, Answer 0.2*

**14.** **Wrong user name.** Amazon Linux's default user is `ec2-user`; there is no `ubuntu` user. Correct
command: `ssh -i key.pem ec2-user@<ip>`. · *0.4.3*

**15.** A container has no kernel of its own; it shares the host's kernel. `/etc/os-release` is a file in the
container image (Alpine), while `uname -r` is a syscall asked of the kernel (the host's Ubuntu AWS kernel).
· *Answer 0.1*

**16.** The tools are only programs that read `/proc`. Read it directly: `cat /proc/meminfo` (memory),
`cat /proc/loadavg` (load average). · *0.3.2*

---

## Scoring

| Correct answers | What to do |
|---|---|
| 14–16 | Move on to Phase 1. The model is in place. |
| 11–13 | Move on to Phase 1, but reread the sections your wrong answers point to. |
| 7–10 | Review all of 0.2 — this section is the foundation for the rest of the roadmap. |
| 0–6 | Work through the phase again from the start. Don't rush; anything missing here comes back multiplied in Phases 3, 4 and 9. |

**Which section to go back to for each missed question:**

| Question | Section |
|---|---|
| 1, 2 | 0.1 — The kernel's job |
| 3, 4, 7, 10 | 0.2.1, 0.2.2 — Two worlds and the syscall |
| 8 | 0.2.3 — The journey of a disk write |
| 9, 12 | 0.2.4 — D state |
| 13 | 0.2.2 + Answer 0.2 — Syscall cost |
| 5, 6, 11, 16 | 0.3 — Everything is a file, `/proc` |
| 14 | 0.4 — Distros and AMIs |
| 15 | 0.1.2 + Answer 0.1 — VMs and containers |

---

# Phase 0 — Closing and bridge to Phase 1

## What you carry from this phase

When you finish this phase you hold **four tools**. Every phase after this uses them:

**1. The "programs ask, the kernel decides" model.**
When something is rejected ("permission denied"), slow (high `sy`) or stuck (D state), your first question
will now be: "**why** did the kernel answer this request this way?"

**2. The "root is in user space" correction.**
In Phase 2 you see root; in Phase 9 you see the layers that can block even root (capabilities, AppArmor).
All of it is built on this single correction.

**3. The "`/proc` is the source of everything" reflex.**
In Phase 3 you read processes, in Phase 4 memory, in Phase 11 everything through `/proc`. If a tool is
missing or seems to lie, you can go straight to the source.

**4. The "find the layer" way of thinking.**
Every row in the failure table pointed to a layer: application, kernel, storage, distro decision. The
layer-by-layer diagnosis method in Phase 11 is the systematic form of this thinking.

## Where Phase 1 connects to this

In Phase 1 we enter the shell and the file system. You will see these connections:

| What you learned in Phase 0 | What happens in Phase 1 |
|---|---|
| Programs run in user space (0.2.1) | The shell is just a program too; every command you type starts a new process (1.1) |
| The `execve` syscall (0.2.2) | The shell looks a command up in PATH and **executes** it (1.1) |
| File descriptors 0, 1, 2 (0.3.1) | stdin, stdout, stderr and redirection (1.4) |
| "Everything is a file" (0.3.1) | Connecting programs with pipes — the Unix philosophy (1.4) |
| `/proc`, `/sys`, `/dev` (0.3) | Finding their place in the file system hierarchy (FHS) (1.2) |
| Distro awareness (0.4) | Where log files and configs live (1.2) |

> **Phase 0 output — ask yourself before moving on:**
> Can I explain "What happens when an application wants to write to disk?" without skipping a single link?
>
> The chain: **the application calls `write()` → the library executes the syscall instruction → the CPU
> switches to privileged mode → the kernel checks the permission and the fd → copies the data into the page
> cache → returns "written" → the kernel writes to disk in the background through the file system and the
> driver (or the application waits for that with `fsync`).**
>
> If you can build this chain, you are ready for Phase 1.

---

> **Navigation:** [◀ Contents](README_en.md) · **Phase 0** · [Phase 1 — Shell and File System ▶](Phase_1_Shell_and_File_System.md)
