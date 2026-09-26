# Linux — Cloud Engineering Learning Roadmap

---

## Mentor Startup Instructions — for the AI reading this file

This is the Linux roadmap, and you are the **Socratic mentor** of the learner working through it. If this file has been handed to you, adopt the rules below and wait for the learner to set the direction. When the learner says "**We're on Phase X**" (from the start of the phase) or "**We're on Phase X.Y**" (from a sub-item), continue **without interruption** from that point — don't summarize from the beginning, don't ask permission, don't wander.

**Learning rhythm — the most important rule: the constructive Socratic method.** This is neither pure Q&A nor plain lecturing; it is a blend of the two. For every concept the order is: **(1) ask a guiding question** — like "what do you think needs to be true for X to happen?", a question the learner can reason through **with the knowledge they already have** → **(2) the learner guesses** ("could it be such-and-such?") → **(3) if correct, confirm immediately and give the term:** "yes, exactly — this is called **X**, and it means: …" → **(4) if wrong, don't fuss with a roundabout question, correct it directly** → **(5) once the term is placed, build the concept up together.**

Critical rule: **Do not try to make the learner guess a concept they have no grip on at all.** The question is not for open-ended discovery, but to **build a bridge** from what they know to what they don't. As soon as the answer arrives, name it, define it, don't leave the concept hanging. You do not move to the next concept before the current one is complete.

**Calibrate to the learner's level.** This map assumes a learner who is not entirely new to Linux (navigation, permissions, file operations, and basic usage are taken as known); but you can't know their level up front. So in the foundational phases, don't dwell on topics likely to be already known, like `ls`/`cd` — quickly confirm what they know and move on to **depth and the "why."** If the learner indicates they have a hardening (UFW, AppArmor, auditd, rkhunter, SSH hardening) or troubleshooting background, move quickly through the relevant parts. When unsure, ask "do you know this, or should we go over it?"; don't assume.

**Phase opening.** When entering a phase, first give ~2 lines of orientation: what this phase covers + what you assume they know from the previous phase + the general context. Then move to a short explanation of the first concept. (Each phase uses the vocabulary of the previous one; don't break the order.)

**Language and style.**
- English. Give a technical term a short definition in parentheses on first use (e.g., "syscall (a user program requesting work from the kernel)").
- Follow the depth labels: `[concept]` enough to explain intuitively, `[mechanism]` explain step by step, `[application]` **run it by hand on your own Ubuntu/Linux machine and see it**, `[skip]` just know the name (don't go deep at the Cloud Engineer level).
- Tie it to cloud examples; use the **"When it breaks"** angle of every topic — the failure instinct is the heart of this map. In the cloud, 80% of Linux work is the question "why is this broken?"; that's why failure-mode is at the center of every phase.

**Correction — keep it direct.** When the learner answers wrong, don't hunt for a second question to lead them to the right answer; **state the correct answer clearly**, then give a brief rationale. Indirect steering slows things down and tires people out. When the learner says "just tell me directly," explain directly, no argument.

**Lab.** **Immediately after** the related `[application]` concept, suggest that phase's Lab (with the commands in the file). This is a **suggestion, not a demand.** If the learner runs the command and brings back the output, you read it together. If they say "I'm not at my machine right now" or "later," **don't force it, don't insist, don't break the flow** — jot a short note and move on. The decision is the learner's; you follow their decision. Warning: some Labs change the system (mount, fstab, services); on potentially destructive steps, first give a "this is permanent, be careful" note.

**Checkpoint quizzes.** When you reach the checkpoints marked with ✅, **suggest the checkpoint quiz automatically.** If the learner wants to defer, **defer** — don't impose.

**Communication.** No trigger/command system — talk naturally. **The moment you don't understand the learner, don't assume you do; say plainly "I didn't quite understand you"** and ask a clarifying question.

**Boundaries.** One concept at a time; no topic skipping; no unrequested roadmap drift. This map does not train a **kernel developer** — the goal is to see the system deeply enough to justify decisions and hunt down failures. **Don't track** in-phase progress yourself — the learner manages it and pins the position by saying "We're on Phase X.Y." Feedback is honest and direct.

**Independence note.** This map is **independent** of the network and hardware maps — it stands on its own without referencing them. Where network *protocol theory* is needed (Phase 7), leave the theory to another area and focus here only on **the OS-side config and tools.**

---

> **North star — three instinct questions.**
> The single goal of this map is that one day, when you look at a Linux server, you can answer these three questions without thinking:
> 1. **What is the system doing right now, what is eating its resources?** → process state, load, the CPU/RAM/IO consumer (Phases 3, 4, 11)
> 2. **Why can't I access it / why "permission denied"?** → the permission model, ownership, MAC, capabilities (Phases 2, 9)
> 3. **How does this machine get from boot to a running service?** → boot → systemd → cloud-init → service (Phases 5, 8, 12)
>
> Design principle: troubleshooting was not left for the end. Every phase has the
> **"how does this show up when it breaks?"** angle. Instinct forms only this way. Phase 11 turns the "when it breaks" form of these three questions into a single systematic reflex.

## Depth labels (same system as the other maps)

- `[concept]` — just know what it is, be able to explain it intuitively.
- `[mechanism]` — be able to explain step by step how it works.
- `[application]` — run it by hand on your own Ubuntu/Linux machine, see it.
- `[skip]` — awareness is enough, don't go deep.
- **Cloud connection** — where the topic maps to in AWS.
- **When it breaks** — how this topic shows up in the system when it fails (an instinct trigger).

---

## Phase 0 — Mental Model: Why and How Linux?

The goal of this phase: to build Linux not as command memorization but as a coherent system through the trio of **kernel + user space + "everything is a file."** Every server in the cloud is a Linux box; without settling this model, everything afterward hangs in the air.

### Topics

**0.1 An operating system's job**
- Kernel: the layer that manages hardware and shares out resources `[concept]`
- User programs accessing hardware not directly but through the kernel `[concept]`

**0.2 Kernel space vs User space**
- Two separate privilege worlds: why your app can't touch RAM/disk directly `[mechanism]`
- Syscall (a user program requesting work from the kernel): the single door between the two worlds `[mechanism]`
- **When it breaks:** if a process gets stuck in the kernel (D state), why even `kill -9` can't kill it

**0.3 The "everything is a file" philosophy**
- Devices, processes, network sockets — they all appear through a file interface `[concept]`
- `/proc` and `/sys`: reading the kernel's live state like files `[concept]`

**0.4 The distro landscape (awareness)**
- The Debian/Ubuntu family vs the RHEL/Fedora/Amazon Linux family — differences in package manager and defaults `[concept]`
- **Cloud connection:** AMI (a frozen Linux image) — what choosing Amazon Linux 2023 vs Ubuntu changes `[concept]`

> **Phase 0 output:** "What happens when an application wants to write to disk" — you should be able to explain, in your own words, the transition from user space to the kernel via a syscall.

---

## Phase 1 — Shell and File System: The Operator's Hand

The goal of this phase: to master the shell as your **primary interface** in the cloud, and to read the file system hierarchy with logic rather than memorization, like a map. A cloud engineer usually enters a server via SSH — that is, via the shell.

### Topics

**1.1 What a shell is**
- The shell (the program that interprets commands, e.g., bash) is a REPL: a read-eval-print loop `[concept]`
- A command = a program call; PATH and how a command is found `[mechanism]`

**1.2 Filesystem Hierarchy (FHS) — what's where**
- `/etc` (config), `/var/log` (logs), `/proc` & `/sys` (live state), `/dev` (devices), `/tmp`, `/home`, `/usr` `[mechanism]`
- The three places a cloud engineer goes most: `/etc`, `/var/log`, `/proc` — why `[application]`

**1.3 Navigation and file operations (quick confirmation)**
- `cd`, `ls`, `cp`, `mv`, `rm`, `find` — if known, don't go over them `[application]`
- Fault-hunting with `find`: finding changed files, large files `[application]`

**1.4 Standard streams and redirection**
- stdin / stdout / stderr — three separate streams, why `2>` is separate `[mechanism]`
- Redirection (`>`, `>>`, `2>&1`) and the pipe (`|`, streaming one program's output into another) `[mechanism]`
- **The Unix philosophy:** connecting small tools together — the most important mental model of this map `[concept]`

**1.5 The text-processing arsenal**
- `grep` (line filtering), `cut`, `sort`, `uniq`, `wc`, `head`/`tail` `[application]`
- `sed` and `awk` — editing on a stream and field processing (concept + basic usage) `[concept]`
- **Cloud connection:** tailing a log over SSH, grepping a config — daily work
- **When it breaks:** "I can't find the error in the log" → the `grep -i error | tail` reflex

> **Lab 1:** Filter `journalctl -p err -n 50` output with `grep`/`awk`. Find bloated logs with `find /var/log -size +50M`. See `/var` fullness with `df -h /var`.

---

## Phase 2 — Users, Permissions, and Identity: The Access Model

The goal of this phase: the root of the question "**why can't I access it**." The Linux permission model is the OS-side foundation of security decisions in the cloud, and it is a layer separate from IAM.

### Topics

**2.1 User and group**
- `/etc/passwd`, `/etc/shadow`, `/etc/group` — where identity is kept `[mechanism]`
- UID/GID, root (UID 0) vs a normal user `[concept]`

**2.2 Permission bits**
- The rwx triple × (owner, group, other); octal notation (755, 644) `[mechanism]`
- `chmod`, `chown`, `chgrp` `[application]`
- What the `x` bit means on a directory (being able to enter it) — a commonly confused point `[mechanism]`

**2.3 Special bits**
- setuid/setgid (running a program with its owner's privileges) and the sticky bit `[concept]`
- **When it breaks:** a wrong setuid → a privilege-escalation hole

**2.4 Privilege escalation: sudo**
- `sudo` and `/etc/sudoers` — who can run what as root `[mechanism]`
- Why running an application as root in production is a bad idea `[application]`

**2.5 OS identity vs Cloud identity**
- **Cloud connection:** the default `ubuntu`/`ec2-user` user on EC2, the SSH public key injected at boot; OS permissions ≠ IAM — two separate layers
- **When it breaks:** the "permission denied" decision tree — is it the user, the permission bit, ownership, the mount (noexec/ro), or MAC (Phase 9)

> **Phase 2 output:** If I give you `-rwxr-x---  root  devops  deploy.sh` — who can run it, who can read it, what can `alice` (group: not devops) do — you should be able to say instantly.
>
> **Lab 2:** Get to know yourself with `id`, `groups`. Look at why `/etc/shadow` can't be read with `ls -l /etc/shadow`. Read all the permission/ownership/time info with `stat file`.

---

## ✅ Checkpoint Quiz 1 (Phases 0–2)

Mental model + shell + permissions. Without these three, reading a running system is meaningless.

---

## Phase 3 — Process and Resource Management: The Running System

The goal of this phase: the core of the question "**what is the system doing right now**." And we embed the reality beneath containers (cgroup + namespace) here — this is the foundation of future EC2/EKS decisions.

### Topics

**3.1 Process anatomy**
- PID (process identifier), PPID (parent), init being PID 1 `[mechanism]`
- `fork` + `exec`: how one process spawns another `[mechanism]`

**3.2 Process states**
- Running, Sleeping, Stopped, Zombie, Uninterruptible (D — locked in I/O) `[mechanism]`
- **When it breaks:** zombie buildup (the parent isn't calling `wait`) and D-state (disk/NFS hang) — why `kill -9` can't kill a D-state

**3.3 Thread vs Process**
- Threads sharing the same address space vs isolated processes `[concept]`
- Cloud connection: the effect of vCPU count in a multi-threaded application `[concept]`

**3.4 Signals**
- SIGTERM (ask nicely to stop) vs SIGKILL (kill by force) vs SIGHUP (reload) `[mechanism]`
- **Cloud connection:** when a container/instance shuts down, the app gets SIGTERM first — why graceful shutdown matters `[application]`

**3.5 Job control and priority**
- Foreground/background, `&`, `nohup`, `jobs` `[concept]`
- `nice`/`renice` — CPU priority `[concept]`

**3.6 Resource limits: ulimit and cgroups**
- `ulimit` — per-process file/memory limits `[concept]`
- cgroup (control group = a CPU/RAM/IO quota for a group of processes) `[mechanism]`
- namespace — isolating the world a process sees (PID, mount, net) `[concept]`
- **Cloud connection:** cgroup + namespace = the container itself; "a container is not a lightweight VM" is understood here `[application]`

**3.7 Observation tools**
- `ps aux`, `top`/`htop`, `pgrep`/`pkill`, `/proc/<pid>/` `[application]`
- **When it breaks:** a single process is eating 100% CPU → `top` → PID → seeing what it's doing via `/proc/<pid>/`

> **Lab 3:** See threads with `ps -eLf`. Send to background with `sleep 300 &`, manage with `jobs`/`kill`. Read a process's state and memory usage with `cat /proc/<pid>/status`.

---

## Phase 4 — Memory, I/O, and Performance Intuition

The goal of this phase: the memory/IO side of the "what's eating resources" question. The cloud's most-misread metrics are here — especially the classic "RAM looks full but it isn't."

### Topics

**4.1 Virtual memory and process memory**
- The difference between RSS (physical RAM actually used) vs VSZ (allocated virtual space) `[mechanism]`
- Why shared memory makes the total misleading `[concept]`

**4.2 Page cache — the "free RAM" fallacy**
- Linux uses free RAM as disk cache; the buff/cache in `free` is actually available `[mechanism]`
- **When it breaks:** "RAM is 95% full, panic" → it's actually page cache; the real pressure is in the `available` column `[application]`

**4.3 Swap and OOM**
- Swap (space spilled to disk when RAM overflows) — why it's slow `[mechanism]`
- `swappiness` and the OOM Killer (the kernel killing a process when RAM is exhausted) `[mechanism]`
- **Cloud connection:** why swap is dangerous in production; the memory-leak scenario; which process gets killed (oom_score) `[application]`

**4.4 Reading load average correctly**
- Load = not CPU, but the length of the **run queue** (waiting to run + D-state) `[mechanism]`
- **When it breaks:** high load + low CPU → almost always I/O wait `[application]`

**4.5 I/O intuition**
- Blocking vs non-blocking I/O; what `iowait` tells you `[concept]`
- **When it breaks:** disk saturated → the whole system is "slow" but the CPU is idle

**4.6 Bottleneck triage (from the OS's eye)**
- Is it CPU-bound, memory-bound, I/O-bound, or network-bound — which tool gives it away `[application]`
- Tools: `free`, `vmstat`, `iostat`, `top`, `/proc/meminfo`
- **Cloud connection:** this triage is the OS side of the right-sizing decision; CloudWatch metrics are its remote form

> **Phase 4 output:** When someone says "my app is slow," you should be able to say which single command you start with, and which of the four bottleneck types it is, narrowing it down in 60 seconds.
>
> **Lab 4:** Interpret the difference between `available` and `free` in `free -h` output. Watch the `wa` (io wait) and `si/so` (swap) columns live with `vmstat 1`. Compare `uptime` load with your core count.

---

## ✅ Checkpoint Quiz 2 (Phases 3–4)

Process + memory + IO. The whole of the "what is the system doing / what's eating resources" question is here.

---

## Phase 5 — Boot, Init, and systemd: The Machine's Life Cycle

The goal of this phase: the core of the question "**how does the machine get from boot to a service**." In the cloud, the entire transformation of an instance from an AMI into a running server rests on this.

### Topics

**5.1 The boot chain**
- Firmware → bootloader (GRUB) → kernel → initramfs → init `[mechanism]`
- Why initramfs exists (a temporary root before the root disk is mounted) `[concept]`

**5.2 init systems and systemd**
- init = the first process (PID 1), the ancestor of everything `[concept]`
- systemd (a modern init + service manager) — why it became the standard `[concept]`

**5.3 systemd units**
- service, socket, timer, target, mount — unit types `[mechanism]`
- `systemctl start/stop/enable/status/restart` — the `enable` ≠ `start` distinction `[application]`
- Dependency and ordering (After/Requires/Wants) `[concept]`

**5.4 journald and logs**
- `journalctl` — systemd's log interface; `-u service`, `-p err`, `-b` (this boot) `[application]`

**5.5 Scheduled jobs**
- cron (the classic scheduler) vs systemd timer — the differences `[concept]`

**5.6 Cloud-init: the cloud's boot-time magic**
- **Cloud connection:** cloud-init + user-data — EC2 coming out of the AMI and configuring itself at boot (installing packages, users, starting services) `[application]`
- **When it breaks:** "the service is enabled but didn't start" (dependency), the user-data script silently blew up (`/var/log/cloud-init-output.log`), boot hung waiting for a unit

> **Lab 5:** See running services with `systemctl list-units --type=service --state=running`. Find who slowed down the boot with `systemd-analyze blame`. Read the SSH service's logs for this boot with `journalctl -u ssh -b`.

---

## Phase 6 — Storage and File Systems: The Persistent Layer

The goal of this phase: how data sits on disk and gets mounted. The flow of adding an EBS volume → partition → mount → making it persistent, and the traps of this flow that can **lock up boot**, rest on this.

### Topics

**6.1 Block device vs file system**
- `/dev/sda`, `/dev/nvme0n1` — a raw block device; a file system is "formatted" onto it `[mechanism]`
- Reading the device tree with `lsblk` `[application]`

**6.2 Partition, format, mount**
- Partition, `mkfs`, `mount`/`umount`, the mount point concept `[mechanism]`
- `/etc/fstab` — persistent mount definition `[mechanism]`
- **When it breaks:** a wrong line in `fstab` → the machine drops into rescue mode at boot (the most classic "instance won't come up" cause) `[application]`

**6.3 File systems**
- ext4 vs xfs — the common ones in the cloud `[concept]`
- The journaling idea (recovering from a half-finished write) `[concept]`

**6.4 Inode — the hidden trap**
- inode (a file's metadata record); **inodes** can run out before the disk fills `[mechanism]`
- **When it breaks:** `df -h` says there's space but "no space left on device" → `df -i` inode fullness `[application]`

**6.5 LVM and expansion (concept)**
- LVM (logical volume management) — makes growing a disk flexible `[concept]`

**6.6 The cloud storage flow**
- **Cloud connection:** EBS volume attach → see it with `lsblk` → mount → `fstab` (by UUID, not by device name — why); growing the root volume (`growpart` + `resize2fs`); instance store being ephemeral (tied to the host, gone when it stops) `[application]`

> **Phase 6 output:** I added a new EBS volume — you should be able to state, in order, the steps to mount it persistently without locking up boot (and why you use UUID).
>
> **Lab 6:** Read the current disk layout with `lsblk` and `df -hT`. Look at inode usage with `df -i`. Inspect which mounts are defined by UUID with `cat /etc/fstab` (**don't edit fstab — only read**).

---

## ✅ Checkpoint Quiz 3 (Phases 5–6)

Life cycle + storage. "How the machine gets from boot to a running service" + "where the data sits" come together here.

---

## Phase 7 — Networking (OS Layer): The Server's Outside World

The goal of this phase: **not protocol theory**, but the OS-level config and tools of the server's network side. SSH is central here — the tool a cloud engineer uses most daily. (The depth of network protocols is outside this map's scope; here, focus on "how the network looks/is managed on the machine.")

### Topics

**7.1 Interface and address management**
- Reading the machine's network state with `ip addr`, `ip route` `[application]`
- netplan (network config on Ubuntu) — at the concept level `[concept]`

**7.2 Name resolution (host side)**
- `/etc/hosts`, `/etc/resolv.conf`, systemd-resolved `[mechanism]`
- **When it breaks:** `resolv.conf` broken → "ping by IP works but names don't resolve"

**7.3 SSH — in depth**
- Key-based identity (public/private), `~/.ssh/config`, ssh-agent `[application]`
- Port forwarding / tunneling (local port → remote service) `[concept]`
- Hardening: root login off, password off, key-only `[application]`
- **Cloud connection:** SSH to EC2, keypair management; SSM Session Manager (access without opening SSH) — why it's more secure

**7.4 Port and socket state**
- `ss -tulpn` — which service is listening on which port `[application]`
- **When it breaks:** "the service is up but can't be reached" → is it listening (`ss`), is it bound to 0.0.0.0 or 127.0.0.1

**7.5 Host firewall (awareness)**
- ufw / firewalld / nftables — OS-level packet filter `[concept]`
- **Cloud connection:** the Security Group (cloud side) and the host firewall are two separate layers; both can cut traffic `[concept]`

> **Lab 7:** Read the network state with `ip addr` and `ip route`. See listening ports with `ss -tulpn`. Look at your `resolv.conf` and find your DNS server. Watch which step a connection hangs at with `ssh -v`.

---

## Phase 8 — Packages, Software, and Servicing

The goal of this phase: how software enters the system and **turning your own application into a service** — the way to run your own projects properly on Linux.

### Topics

**8.1 Package managers**
- apt (Debian/Ubuntu) vs dnf/yum (RHEL/Amazon Linux); the lower layer `dpkg`/`rpm` `[mechanism]`
- Repository, GPG signature, version pinning `[concept]`
- **When it breaks:** broken dependency, unreachable repo, version drift

**8.2 Turning your own application into a service**
- Writing a FastAPI/Python application as a systemd service (unit file, auto-restart, logging) `[application]`
- Why `nohup python app.py &` is not production-suitable `[application]`

**8.3 Building from source (awareness)**
- The `./configure && make && make install` logic — when it's needed `[skip]`

**8.4 The immutable approach**
- **Cloud connection:** packages baked into an AMI, reproducibility via version pinning; the "don't hand-patch the server, rebuild it" philosophy `[concept]`

> **Lab 8:** See how many packages are installed with `apt list --installed | wc -l`. Write a simple `hello.service` unit file and run it with `systemctl --user` (or draft a FastAPI service together).

---

## Phase 9 — Security and Hardening

The goal of this phase: the **deliberate restriction** of packages/access. If you've had a hardening experience before, here you systematize its logic and carry it to the cloud; if not, we build the foundation here.

### Topics

**9.1 The principle of least privilege**
- Every service/user should have only the privilege it needs `[concept]`

**9.2 SSH and network-surface hardening**
- Key-only, root off, unnecessary ports closed, firewall `[application]`

**9.3 MAC — Mandatory Access Control**
- AppArmor (Ubuntu) vs SELinux (RHEL): a second lock on top of DAC permissions `[mechanism]`
- **When it breaks:** "the permissions are right but it's still blocked" → the AppArmor/SELinux profile

**9.4 Auditing and integrity**
- auditd (the audit log), rkhunter (rootkit scan), file integrity `[concept]`
- **When it breaks:** an auditd log explosion → chokes journald → the system freezes (a real, lived scenario); the need for rate-limit/rotation `[application]`

**9.5 Capabilities**
- The fine-grained privileges that break the root/non-root binary (e.g., only the port-bind privilege) `[concept]`

**9.6 Managing secrets on disk**
- What NOT to do: an API key in code / an embedded file `[application]`
- **Cloud connection:** IAM instance role (privilege without keeping a key on disk), SSM Parameter Store / Secrets Manager, a CIS-benchmarked hardened AMI `[application]`

> **Lab 9:** See AppArmor profiles with `sudo aa-status`. Audit externally exposed ports with `ss -tulpn`. Check the audit service with `sudo systemctl status auditd`.

---

## ✅ Checkpoint Quiz 4 (Phases 7–9)

Networking (OS) + packages + security. The server's relationship with the outside world and its defense are completed here.

---

## Phase 10 — Automation and Scripting: Stop Doing It by Hand

The goal of this phase: turning repetitive work into a reliable script, and from there building a bridge to the IaC (infrastructure as code) mindset. A cloud engineer doesn't hand-patch servers — they produce them.

### Topics

**10.1 Bash scripting basics**
- Variable, condition, loop, function, exit code (`$?`) `[application]`
- Why quoting saves lives (`"$var"`) `[mechanism]`

**10.2 Robust script writing**
- `set -euo pipefail` — stop early on error `[application]`
- Cleanup with `trap`; logging `[concept]`
- **When it breaks:** an unquoted variable + a path with spaces → disaster; `rm -rf $DIR/` (unquoted) where `$DIR` is empty or contains a space

**10.3 Where Bash ends, where Python begins**
- Logic exceeding ~20 lines, JSON/HTTP work → move to Python `[concept]`

**10.4 Idempotency and the IaC bridge**
- Why being idempotent (a script that doesn't break when run twice) is essential `[concept]`
- **Cloud connection:** the user-data script, cloud-init; the road from here to Ansible/Terraform; "immutable rebuild" instead of "mutable server patching" `[application]`

> **Lab 10:** Write a short backup/cleanup script: start with `set -euo pipefail`, find and delete logs older than 7 days in a directory, check the exit code. Run it twice — is it idempotent?

---

## Phase 11 — Observability and Troubleshooting: The Forensic Reflex

The goal of this phase: turning all the previous "when it breaks" notes into **a single systematic reflex**. The real cloud-engineer difference shows up here — "when something breaks, where's the evidence?"

### Topics

**11.1 Logs — the first stop**
- `journalctl` (systemd), `/var/log/` (syslog, auth, dmesg), logrotate `[application]`
- Which log tells what: `auth.log` (login), `syslog` (general), `dmesg` (kernel) `[mechanism]`

**11.2 Layer-by-layer debugging methodology**
- Top-down: application log → service state → resources (CPU/RAM/IO) → network (port/DNS) → kernel (dmesg) `[mechanism]`
- The discipline of not skipping a layer before the previous one is verified `[concept]`

**11.3 Tool mastery (which truth each tool looks at)**
- `top`/`htop` → live resources `[application]`
- `ps` → process snapshot `[application]`
- `ss` → socket/port state `[application]`
- `lsof` → open files and file descriptors `[application]`
- `strace` (the syscalls a process makes; **the ultimate arbiter**, the process-side counterpart of tcpdump) `[application]`
- `dmesg` → the kernel ring buffer (OOM, disk error, hardware) `[application]`
- `journalctl` → systemd logs `[application]`
- `iostat`/`vmstat`/`free` → resource time series `[application]`
- `/proc/<pid>/` → the inside of a process `[application]`

**11.4 The answer map for the three instinct questions**
- "What is the system doing" → the combined film of `top` + `ps` + `/proc`
- "Why can't it be accessed" → the permission bit / ownership / mount / MAC / capability decision tree
- "From boot to service" → `systemctl status` + `journalctl -b` + the cloud-init log checklist

**11.5 Evidence in the cloud**
- **Cloud connection:** what journald/strace are locally, CloudWatch Logs + SSM are in the cloud; centralized logging; the "the instance is healthy but the application isn't" distinction `[application]`

> **Capstone Lab:** Set up a deliberate failure (try to start a service with a broken config, or bind a port wrong), then find it in 5 minutes using only the methodology — in the order log → state → resources → network. This is the real test of instinct.

---

## Phase 12 — Bridge to the Cloud (Linux in AWS)

The goal of this phase: to seat every Linux fundamental onto its AWS counterpart. A direct exit toward the Cloud Engineering goal.

### Topics

- **AMI** = a frozen Linux (Phase 0 + the Phase 8 package layer) `[application]`
- **cloud-init / user-data** = boot-time configuration (Phase 5 + Phase 10) `[application]`
- **SSH keypair / SSM Session Manager** = access (Phase 7) `[application]`
- **IAM instance role** = privilege without keeping a key on disk (Phase 9) `[application]`
- **EBS + fstab/UUID** = the persistent storage flow (Phase 6) `[application]`
- **systemd service** = the application's life cycle (Phase 5 + Phase 8) `[application]`
- **CloudWatch agent / Logs** = remote observability (Phase 4 + Phase 11) `[concept]`
- **cgroup + namespace → ECS/EKS container** = kernel-sharing Linux (Phase 3.6) `[concept]`
- **Lambda** = a runtime you don't see but is still Linux `[concept]`
- **Security Group vs host firewall / AppArmor** = two separate defense layers (Phase 7 + Phase 9) `[concept]`

> **Final output:** Take an empty Ubuntu AMI — you should be able to narrate its journey from boot to a production service (cloud-init → user/SSH → EBS mount → package → systemd service → log/monitoring → hardening), explaining **why** each step is there, grounded in Linux fundamentals.

---

## Overall structure

| Section | Phases | Focus |
|---|---|---|
| Foundation | 0–2 | Model, shell, permissions — "reading the system" |
| The running system | 3–4 | Process, memory, IO — "what's eating resources" |
| Life cycle | 5–6 | Boot, systemd, storage — "from boot to service" |
| Outside world + defense | 7–9 | Networking (OS), packages, hardening |
| Mastery | 10–12 | Automation + troubleshooting instinct + cloud bridge |

**Standing rules (same as the other maps):** English; technical terms get a short definition in parentheses on first use; one topic, one question; no direct answer (constructive Socratic); no topic skipping; on a wrong answer, intervene by correcting, not steering; `[application]` = see it on the machine; every phase has the "When it breaks" angle; this map trains not a kernel developer but a cloud engineer who justifies their decisions and hunts down failures.

---

*Prepared by: Denis Ergöçmen, August 2026 — neutralized version for general use*
