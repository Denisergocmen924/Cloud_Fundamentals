# Linux — Offline Workbook

> **Cloud Engineer Fundamental Roadmap — Linux / Operating System**
> This is the **standalone, AI-mentor-free, internet-free** workable version of the
> [`Cloud_linux_roadmap_fundamental_en.md`](../../../Cloud%20Learning%20With%20Mentor/English/Cloud_linux_roadmap_fundamental_en.md)
> roadmap.

> 🇹🇷 Türkçe sürüm: **[README.md](../Turkish_Linux/README.md)**

---

## What is this book, and how does it differ from the main map?

The roadmap file in the mentor folder is a **skeleton**: you give it to an AI mentor and the
mentor opens the topics in order. Read on its own, the skeleton tells you "what you need to
learn" but it does not "teach it to you".

This book is the **flesh** on that skeleton. The same phases, the same sub-items, the same
depth labels — but here every item is:

- **explained** (intuition → mechanism → command output),
- **asked** (questions left for you to think about),
- **answered** (later in the same file),
- **connected** (to the previous item and to its counterpart in AWS).

Nowhere does it say "ask your mentor about this". You can work through the file from start
to finish on a plane, with no internet, entirely on your own.

---

## How this book differs from the hardware book — important

In the hardware book, `[application]` meant "tie it to an architectural decision".
**Here, `[application]` really means "run it on your own machine".**

There is no other way to learn Linux. You cannot learn to read `ps aux` output by reading
about it — you have to look at the output on your own machine. That is why **the expected
output of every command is written down** in this book: if you have no machine, you read
the output and follow its line-by-line interpretation; if you have one, you run the command
and compare it with your own output.

> **What if you have no machine?** Nothing. This book can be worked through from start to
> finish without a machine — every output is written down and interpreted. But if you can
> find an Ubuntu machine (physical, a VM, WSL2 or an EC2 t3.micro), your learning speed
> **doubles**.

---

## Files

| File | Phase | Topic | Approximate time |
|---|---|---|---|
| [Phase_0_Mental_Model.md](Phase_0_Mental_Model.md) | 0 | Kernel/user space, syscall, "everything is a file", the distro landscape | 3–4 hours |
| [Phase_1_Shell_and_File_System.md](Phase_1_Shell_and_File_System.md) | 1 | Shell, FHS, streams, pipes, the text-processing arsenal | 5–7 hours |
| [Phase_2_Users_and_Permissions.md](Phase_2_Users_and_Permissions.md) | 2 | UID/GID, rwx, special bits, sudo, OS identity vs IAM | 4–6 hours |
| [**Checkpoint_Quiz_1.md**](Checkpoint_Quiz_1.md) | 0–2 | Combined quiz: model + shell + permissions | 1 hour |
| [Phase_3_Processes_and_Resources.md](Phase_3_Processes_and_Resources.md) | 3 | PID, fork/exec, states, signals, cgroup, namespace | 6–8 hours |
| [Phase_4_Memory_IO_Performance.md](Phase_4_Memory_IO_Performance.md) | 4 | RSS/VSZ, page cache, swap/OOM, load average, bottleneck triage | 6–8 hours |
| [**Checkpoint_Quiz_2.md**](Checkpoint_Quiz_2.md) | 3–4 | Combined quiz: "what is eating the resources" | 1 hour |
| [Phase_5_Boot_Init_systemd.md](Phase_5_Boot_Init_systemd.md) | 5 | The boot chain, systemd units, journald, cloud-init | 6–8 hours |
| [Phase_6_Storage_and_Filesystems.md](Phase_6_Storage_and_Filesystems.md) | 6 | Block devices, mount, fstab, ext4/xfs, inodes, LVM, the EBS flow | 5–7 hours |
| [**Checkpoint_Quiz_3.md**](Checkpoint_Quiz_3.md) | 5–6 | Combined quiz: "from boot to service" + "where the data is" | 1 hour |
| [Phase_7_Networking_and_Connectivity.md](Phase_7_Networking_and_Connectivity.md) | 7 | `ip`, the DNS client, SSH in depth, `ss`, the host firewall | 5–7 hours |
| [Phase_8_Packages_and_Service_ization.md](Phase_8_Packages_and_Service_ization.md) | 8 | apt/dnf, turning your own app into a systemd service, immutable AMIs | 4–6 hours |
| [Phase_9_Security_and_Hardening.md](Phase_9_Security_and_Hardening.md) | 9 | Least privilege, SSH hardening, AppArmor/SELinux, auditd, capabilities, secrets | 5–7 hours |
| [**Checkpoint_Quiz_4.md**](Checkpoint_Quiz_4.md) | 7–9 | Combined quiz: the outside world + defense | 1 hour |
| [Phase_10_Automation_and_Scripting.md](Phase_10_Automation_and_Scripting.md) | 10 | Bash basics, `set -euo pipefail`, quoting, idempotency, the IaC bridge | 5–7 hours |
| [Phase_11_Observability_and_Troubleshooting.md](Phase_11_Observability_and_Troubleshooting.md) | 11 | Logs, the layer-by-layer method, tool mastery, the forensic reflex | 6–8 hours |
| [Phase_12_Bridge_to_the_Cloud.md](Phase_12_Bridge_to_the_Cloud.md) | 12 | AMI → cloud-init → SSH → EBS → systemd → logs → hardening | 4–5 hours |

**Appendices:**

| File | What it is for |
|---|---|
| [Appendix_A_Command_Glossary.md](Appendix_A_Command_Glossary.md) | Every command, with its purpose and risk mark (🟢🟡🔴) and the phase it came from — keep it open on your desktop |
| [Appendix_B_File_Directory_Map.md](Appendix_B_File_Directory_Map.md) | Where each file lives and what it does — FHS, `/etc`, `/var/log`, `/proc`, systemd units, SSH files |
| [Appendix_C_When_It_Breaks_Quick_Reference.md](Appendix_C_When_It_Breaks_Quick_Reference.md) | Symptom → likely cause → confirming command → phase; every "When it breaks" table merged into one page |

---

## How to study

### 1. Don't break the order

Every phase speaks in the **words** of the one before it. When Phase 4 explains the "OOM
Killer", it uses the process states from Phase 3 and the kernel/user space split from
Phase 0. The whole of Phase 11 is built on top of the "When it breaks" notes of the eleven
phases before it — read on its own, it turns into a list of commands.

**The single exception:** the navigation part of Phase 1 (`cd`, `ls`, `cp`, `mv`). If you
know these, go through them quickly — but `1.4` (streams and redirection) and `1.5` (text
processing) are not "known" topics and must not be skipped.

### 2. Take the boxes seriously

There are four kinds of box in the text. Each has a different job:

> **🤔 Think 3.2** — **Do not answer** the question here the moment you read it. Take a pen,
> think for 2 minutes, write down your guess. The answer is in the *"Answers to the think
> questions"* section at the end of the phase. If your guess turns out wrong, that is a good
> thing — a wrong guess makes the right answer permanent.

**❓ A question that comes to mind:** A question that arises naturally while reading. The
answer is **right below it**. These do not wait, because leaving them unanswered breaks the
flow of reading.

**⚠️ Common misconception:** Something most people get wrong. If you find yourself saying
"I thought that too" while reading these, mark that sentence.

**🔧 See it on your machine:** A command + the **expected output** + a line-by-line reading
of that output. Every box starts with one of three markers:

| Marker | Meaning |
|---|---|
| 🟢 | **Read-only.** It does not change your system; run it with peace of mind. |
| 🟡 | **Temporary change.** It goes back to the old state when the machine restarts. |
| 🔴 | **Permanent change.** Don't do it on a production machine. Don't run it without understanding what it does. |

In this book, commands marked 🔴 **always come with an undo step**.

### 3. Obey the depth labels

| Label | What is expected | How you test it |
|---|---|---|
| `[concept]` | Being able to explain it intuitively | "Could I explain this to a friend in 30 seconds?" |
| `[mechanism]` | Being able to explain it step by step | "Could I draw the flow with pen and paper?" |
| `[application]` | **Being able to run it on your machine and read the output** | "Would I spot what is abnormal in this output?" |
| `[skip]` | Just knowing the name | Hearing the name and being able to say "that was in that area" is enough |

### 4. Don't skip the end of a phase

Every phase ends with:

1. **When this phase breaks — failure signatures** — a symptom → mechanism → first place to look table
2. **Answers to the think questions** — compare them with your own guesses
3. **Frequently asked questions** — the "but what about…" questions around the phase
4. **Test yourself** — a 15–21 question test + a full answer key
5. **Closing and bridge** — what is left of this phase, and where the next phase connects to it

If you score below 70% on the test, **instead of repeating the phase**, go back to the
section the question you got wrong points to. Every answer key tells you which section it
belongs to.

### 5. Don't skip the checkpoint quizzes — they are different

The four checkpoint quizzes (Phases 0–2, 3–4, 5–6, 7–9) measure **something different** from
the phase tests. A phase test asks "did you understand this phase?"; a checkpoint quiz asks
"**can you connect these phases to each other?**" Most of its questions cannot be answered
from a single phase.

Real work is like that too: no failure stays inside a single phase.

---

## Progress tracking

```
[ ] Phase 0  — Mental Model
[ ] Phase 1  — Shell and File System
[ ] Phase 2  — Users, Permissions, and Identity
[ ] ✅ Checkpoint Quiz 1 (Phases 0–2)
[ ] Phase 3  — Process and Resource Management
[ ] Phase 4  — Memory, I/O, and Performance Intuition   ← the most misread metrics
[ ] ✅ Checkpoint Quiz 2 (Phases 3–4)
[ ] Phase 5  — Boot, Init, and systemd
[ ] Phase 6  — Storage and File Systems                 ← the fstab trap is here
[ ] ✅ Checkpoint Quiz 3 (Phases 5–6)
[ ] Phase 7  — Networking (OS Layer)
[ ] Phase 8  — Packages and Servicing
[ ] Phase 9  — Security and Hardening
[ ] ✅ Checkpoint Quiz 4 (Phases 7–9)
[ ] Phase 10 — Automation and Scripting
[ ] Phase 11 — Observability and Troubleshooting        ← the summit of the map
[ ] Phase 12 — Bridge to the Cloud
```

---

## North star — three instinct questions

The whole of this book exists so that you can answer these three questions **without
having to think**:

| # | Question | Where it is answered |
|---|---|---|
| 1 | **What is the system doing right now, what is eating its resources?** | Phases 3, 4, 11 |
| 2 | **Why can't I access it / why "permission denied"?** | Phases 2, 6, 7, 9 |
| 3 | **How does this machine get from boot to a running service?** | Phases 5, 8, 12 |

These three questions are roughly 80% of a cloud engineer's job. The remaining 20% is
automating them (Phase 10).

**Design principle:** troubleshooting was not left for the end. Every phase has the *"how
does this knowledge look when it breaks?"* angle. Instinct only forms this way — Phase 11
turns these scattered notes into a single reflex.

---

## The goal and the limit of this book

**Goal:** When you SSH into a Linux server, being able to answer "what is happening on this
machine" with a **model of the system**, not with memorized commands; being able to narrow
down a failure with **method**, not with guesses.

**Limit:** This book does not produce kernel developers. Writing kernel modules, the inner
workings of the scheduler and device driver development are out of scope. The goal is to
see the system deeply enough to **justify decisions and hunt down failures**.

**Distribution assumption:** The examples run on **Ubuntu 22.04/24.04**. Amazon Linux /
RHEL differences are noted where they matter — you will meet that difference in the cloud
all the time.

---

## The relationship to the main map

| This book | The main map |
|---|---|
| Offline, worked through alone | Given to an AI mentor |
| Explanation + question + answer | A topic list + mentor instructions |
| Fixed content | The mentor calibrates to the learner's level |
| Test-yourself quizzes | The mentor examines you orally |

The two are **not rivals**. Sequential use is recommended: first work through the phase with
this book, then give the main map to an AI mentor and repeat the same phase **orally**.

---

## The relationship to the other books

This series has three books, and they **limit each other on purpose**:

| Book | What it takes | Overlap with this book |
|---|---|---|
| [Hardware](../../Cloud_hardware.roadmap/English_Hardware/README_en.md) | The physical layer: CPU, memory, disk, NIC, hypervisor | What lies **beneath** Phase 4's "why is it slow" question: cache, IOPS, steal time |
| **Linux** (this book) | The operating system: processes, memory management, boot, permissions, tools | — |
| [Network](../../Cloud_network.roadmap/English_Network/README_en.md) | Protocol theory: OSI, TCP/IP, subnets, DNS, TLS | Phase 7 here takes only the **OS side**; the protocol itself lives there |

This book **walks independently** of the other two. If you have not read the hardware or
network book, you will not get stuck anywhere — where a concept is needed, it is explained
briefly here, and the related book is pointed to if you want to go deeper.

---

*This offline edition is based on Denis Ergöçmen's Cloud Engineer fundamental roadmap series.*
