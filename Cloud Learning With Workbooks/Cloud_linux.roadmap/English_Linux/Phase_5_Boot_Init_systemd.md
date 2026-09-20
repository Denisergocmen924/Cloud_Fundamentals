# Phase 5 — Boot, Init and systemd: The Machine's Lifecycle

> **Navigation:** [◀ Checkpoint Quiz 2](Checkpoint_Quiz_2.md) · **Phase 5** · [Phase 6 — Storage and Filesystems ▶](Phase_6_Storage_and_Filesystems.md)

---

## Where we are coming from

Phases 3 and 4 taught you to read a **running** system: what state processes are in, what eats its
resources, where the bottleneck is. You now read every line of `top` from a model. But throughout those
two phases we took one thing for granted: **the machine was already running.** At the root of the
process tree sat PID 1 = systemd (Phase 3.1), and we accepted it as a box labeled "the ancestor of
everything."

This phase opens that box. Because the systemd at the root of the process tree is no accident: it is
what **brings** the machine from **boot** to that running state. In Phase 4 we protected a process from
OOM with `OOMScoreAdjust` — but you never saw **where** that setting is written (a systemd unit file).
`dmesg` showed you kernel events; now with `journalctl` you'll read systemd's central logbook.

Three things from Phases 3 and 4 will serve you directly here:

- **PID 1 = init = the root of the tree** (Phase 3.1). Now we'll open up *what* that PID 1 is (systemd)
  and what it does to keep the machine standing.
- **Resource limiting with cgroups** (Phase 3.6). You'll see that those cgroup limits are set in
  production not by hand but via `MemoryMax=`/`CPUQuota=` lines in systemd unit files.
- **`OOMScoreAdjust` and protecting a process** (Phase 4.3). The permanent form of that setting is a
  line in a `.service` file — in this phase you'll recognize that line.

## The question of this phase

Phases 3–4 answered "what is the system doing right now." This phase steps back and asks about the
**beginning**:

> *"How does a machine get from boot to a running service — and where, if this chain breaks, does the
> instance fail to come up?"*

In the cloud, the entire transformation of an instance from an AMI (a disk image) into a running server
rests on this chain. "The instance started but won't accept SSH," "the service is enabled but didn't
come up on boot," "the user-data script failed silently" — the most maddening cloud failures are all
the subject of this phase. An engineer who doesn't know this chain is helpless in front of an instance
that won't boot; one who does finds where it stalled in three commands.

By the end of this phase you'll know **why** `enable`ing a service differs from `start`ing it (the
difference surfaces after a reboot), be able to read `systemctl status` and `journalctl` output to
diagnose why a service didn't come up, and understand how an EC2 instance configures itself at boot
time (cloud-init + user-data).

---

## By the end of this phase

- You'll be able to describe the boot chain in order: firmware → bootloader (GRUB) → kernel → initramfs
  → init; and know **why** initramfs exists (a temporary root before the real root disk is mounted)
- You'll be able to explain what init is — the first process, PID 1, the ancestor of all processes —
  and why systemd became the modern standard init + service manager
- You'll be able to tell apart the systemd unit types (service, socket, timer, target, mount) and
  correctly use `systemctl start/stop/enable/status/restart`, especially the **`enable` ≠ `start`**
  distinction
- You'll be able to read dependency and ordering between units (`After=`, `Requires=`, `Wants=`), and
  know why "service is enabled but didn't start" is usually a dependency problem
- You'll be able to look at systemd's central log with `journalctl` and read a service's logs for this
  boot with the `-u service`, `-p err`, `-b` filters
- You'll be able to explain the difference between cron and a systemd timer
- **Cloud:** you'll be able to explain how cloud-init + user-data configures an instance out of an AMI
  at boot time (installing packages, adding users, starting services) and where to look
  (`cloud-init-output.log`) when that chain breaks

---

## The phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 5.1 | The boot chain: from firmware to init | `[mechanism]` | The map of the "instance won't boot" failure |
| 5.2 | init and systemd: the identity of PID 1 | `[concept]` | Opening Phase 3's PID 1 box |
| 5.3 | systemd units and `systemctl` | `[mechanism]` + `[application]` | **The heart of the phase** — the language of service management |
| 5.4 | journald and `journalctl` | `[application]` | Where the answer to "why didn't the service come up" lives |
| 5.5 | Scheduled jobs: cron vs timer | `[concept]` | Who runs recurring jobs |
| 5.6 | Cloud-init: the cloud's boot-time magic | `[application]` | **The cloud output** — from AMI to running server |
| 5.7 | When this phase breaks | — | The signatures of boot and service failures |

> **How to work through this phase:** This phase's observation commands are 🟢 (`systemctl status`,
> `systemctl list-units`, `journalctl`, `systemd-analyze` — read only). But this phase differs from
> Phase 4: here there really are **state-changing** commands. `systemctl restart/start/stop` stops and
> starts a service — 🟡 (temporary; but if you stop a production service, users are affected).
> `systemctl enable/disable` changes boot behavior **permanently** — 🔴 (it matters after a reboot;
> every 🔴 box ends with an **Undo** step). So try this phase's commands on **your own test machine**,
> not live production. The most illuminating moment: `enable` a service **without** starting it, reboot,
> and find it running afterwards — that one experiment settles what `enable` means.

---
---

# 5.1 The Boot Chain: From Firmware to init

## 5.1.1 The whole chain: from the power button to PID 1 `[mechanism]`

In Phase 3 we saw PID 1 at the root of the process tree and called it "the ancestor of everything." But
PID 1 doesn't descend from the sky — **a chain** puts it there. The time from pressing the power button
to the `login:` prompt is really a relay of handoffs. Each link finds the next one, starts it, and
**hands over** control:

```
Power → Firmware (BIOS/UEFI) → Bootloader (GRUB) → Kernel → initramfs → init (PID 1) → target
```

![Figure 5.1 — The boot chain: from power to PID 1; each link finds and hands off to the next, initramfs mounts the real root disk, and services start only after PID 1](../diagrams/png/lx-5-01-boot-chain.png)

Link by link:

1. **Firmware (BIOS/UEFI).** The first code, embedded on the motherboard. It runs a hardware check
   (POST), then looks for a "bootable" disk and reads the **bootloader** from it into memory and jumps
   to it. Firmware doesn't know the operating system; it just says "run the code at the start of the
   disk."
2. **Bootloader (GRUB).** GRUB (from *GRand Unified Bootloader* — it "grabs" and loads the kernel). Its
   job: decide which kernel loads, with which parameters, and load that kernel (and initramfs) from
   disk into memory and run it. That boot menu that lets you choose between kernel versions is GRUB.
3. **Kernel.** The kernel loaded into memory starts running: it takes proper control of the hardware
   (CPU, memory, drivers), but it hasn't yet mounted the real root filesystem — because the drivers
   needed to read that disk may be **on the disk itself**. That's the chicken-and-egg problem.
4. **initramfs.** A **temporary, tiny root filesystem** that solves this chicken-and-egg problem (the
   subject of the next subsection). It lives in memory and contains the drivers needed to mount the
   real root disk. After mounting the real root disk it hands over control.
5. **init (PID 1).** Once the real root disk is mounted, the kernel starts the **first user-space
   process** there — `/sbin/init` (on modern systems, systemd) — as PID 1. The root of the tree you
   saw in Phase 3 is born exactly here.
6. **target.** systemd, according to the target of "how far exactly should the machine come up" (e.g.
   `multi-user.target` = network + services but no GUI), brings services up in order.

> **🔧 See it on your machine** 🟢 — read the kernel-to-init handoff of this boot
>
> ```
> $ journalctl -b -o short-monotonic | head -20
> [    0.000000] kernel: Linux version 6.8.0-45-generic ...
> [    1.842301] kernel: Freeing initrd memory: 62800K
> [    2.104882] systemd[1]: systemd 255 running in system mode.
> [    2.110044] systemd[1]: Detected virtualization amazon.
> ```
>
> `-b` = this boot. The seconds stamps on the left start from zero: 0.0 is the kernel's first moment,
> at ~2.1 `systemd[1]` (PID 1) takes over. The `Detected virtualization amazon` line even tells you
> this machine is an EC2 instance — this is where cloud-init (5.6) will kick in.

The practical value of keeping this chain as a "map" is this: when someone says **the instance won't
boot**, the failure is at one link of this chain, and each link's failure gives a different symptom. If
GRUB is broken, no boot menu appears at all; if initramfs can't find the root disk, it "drops to a
rescue shell"; if a unit hangs during init, the boot freezes there. Knowing the chain is tying the
symptom to the right link.

> **⚠️ Common misconception: "When the computer powers on, the operating system starts directly."**
>
> There are at least three independent software layers in between (firmware → GRUB → kernel), and each
> is a completely separate program from the previous one. The "operating system" (kernel + init) enters
> in the middle of the chain. That's why the symptom "the OS won't boot" often comes from a link
> **before** the OS (GRUB, initramfs) — and you can't reach those with the OS's tools (systemctl,
> journalctl); they aren't loaded yet.

---

## 5.1.2 initramfs: the temporary root before mounting `[concept]`

The most confusing link in the chain above is initramfs, and it exists precisely because it solves the
chicken-and-egg problem. Let's state the problem clearly:

- The kernel wants to mount the real root disk (`/`).
- But to read that disk it needs a **driver** (depending on the disk type: an NVMe driver, LVM, an
  encrypted-disk unlocker, a network-disk client...).
- Where does that driver live? Often **on the root disk itself**, as a file.
- So: to mount the disk you need the driver, to read the driver you need to mount the disk. Deadlock.

initramfs (*initial RAM filesystem*) breaks this deadlock. GRUB loads, **together** with the kernel, a
small compressed archive into memory: a temporary root filesystem living entirely in **RAM**,
containing the minimum drivers and scripts needed to mount the real root disk. The flow:

1. The kernel uses initramfs as the temporary root (`/`) — touching no real disk, entirely in memory.
2. Scripts inside initramfs load the needed drivers (e.g. the NVMe module), find the real root disk,
   and mount it.
3. Then it does a "root switch" (`switch_root`/`pivot_root`): it drops the temporary RAM root, makes
   the real disk `/`, and runs the real `init` there.
4. The initramfs memory is freed (the `Freeing initrd memory` line above is exactly this).

> **❓ Question that comes to mind: "If the root disk is always the same, why isn't initramfs fixed
> rather than different on every machine?"**
>
> Because the set of drivers needed to reach a given machine's root disk differs: one boots from a plain
> SATA disk, one from encrypted LVM, one over the network (iSCSI). initramfs is that machine's specific
> "recipe for reaching the disk," and it is regenerated (`update-initramfs`) when the kernel is updated.
> In the cloud, AMIs are therefore packaged with an initramfs suited to the target hardware
> (Nitro/NVMe) — an image with the wrong initramfs drops to a rescue shell on boot because it can't
> find the root disk.

> **🤔 Think 5.1** — You rebooted an instance and in the console output you saw: `Gave up waiting for
> root file system device. ... ALERT! UUID=abc-123 does not exist. Dropping to a shell!` Then an
> `(initramfs)` prompt appeared. This one screen tells you **which link** of the boot chain was reached
> and where it stalled? Can you SSH into this machine, and why?
>
> *(Answer: at the end of the phase)*

---
---

# 5.2 init and systemd: The Identity of PID 1

## 5.2.1 init = the first process, PID 1, the ancestor of everything `[concept]`

In Phase 3 we saw PID 1 at the root of the process tree. Now let's define it exactly: **init is the
first process the kernel starts in user space**, and its PID is always 1. At the end of boot the kernel
starts a single process — init — and **every** other process descends from it (via `fork`). That's what
we meant in Phase 3.1 by "PID 1 is the root of the tree."

init has two fundamental jobs, and these two make it the backbone of the system:

1. **Starting everything.** At the last stage of boot, init manages which services come up, in what
   order. The concrete meaning of the command "bring the machine to a running state" is the work init
   does.
2. **Adopting orphan processes.** In Phase 3.2.2 we saw zombie and orphan processes. If a parent dies
   before its child, the child is orphaned — and **PID 1 adopts it** (`reparenting`). When these
   adopted children die, PID 1 reaps them with `wait` and prevents them from lingering as zombies.
   That's why PID 1's healthy operation is critical for the whole system: if PID 1 dies, the kernel
   **panics**, because the root of the tree is gone.

> **⚠️ Common misconception: "PID 1 can be killed/restarted like any other process."**
>
> PID 1 is special. The kernel handles signals sent to it differently: PID 1 **ignores** signals for
> which it hasn't explicitly defined a handler — even `SIGTERM`, which kills a normal process, has no
> default effect on PID 1. And if PID 1 does die for any reason, the kernel says "kernel panic — not
> syncing: Attempted to kill init!" and the machine halts. That's why systemd, as PID 1, is kept
> extremely robust and minimal.

## 5.2.2 systemd: why it became the modern standard `[concept]`

For many years init meant the classic "SysV init" (a system that started services with scripts,
sequential and slow). Today nearly all major distributions (Ubuntu, Debian, RHEL, Amazon Linux) use
**systemd**. Its name comes from "system daemon" (the trailing `d` is short for *daemon*, the name Unix
gives background services). systemd is not just an init; it combines two jobs:

- **init:** as PID 1, the thing that brings the machine from boot to a running state and manages its
  lifecycle.
- **service manager:** the central manager that starts, stops, monitors services, restarts them if they
  crash, caps their resources (with cgroups!), and collects their logs (with journald).

Three practical reasons systemd became the standard over classic init:

1. **Parallel startup.** Classic init started services one by one, in sequence — the next didn't start
   until the previous finished; boot was slow. systemd models dependencies as a graph and starts
   services that don't depend on each other **at the same time**. Boot time drops significantly.
2. **Dependency model.** You explicitly define rules like "this service should start only after the
   network is up" (`After=`, `Requires=`); systemd works out the right order itself.
3. **Integrated management.** A service's cgroup (Phase 3.6 — resource limits), its log (journald), its
   restart on crash (`Restart=`), its automatic start on boot (`enable`) — all managed from one
   consistent interface (`systemctl`). The **permanent home** of the cgroup and OOM settings you saw by
   hand in Phases 3–4 is systemd unit files.

> **💡 Cloud connection — why every cloud engineer knows systemd:** When you run an application on an
> EC2 instance, you package it not as a process that "you start by hand and dies when you close SSH,"
> but as a **systemd service** (a `.service` file). This way: it comes up automatically after a reboot
> (`enable`), systemd restarts it if it crashes (`Restart=always`), its resources are capped
> (`MemoryMax=` — Phase 4's OOM protection), and its log is collected in one place (`journalctl -u
> yourapp`). This is the OS-side answer to "how do I run my application in production."

> **🤔 Think 5.2** — A colleague says "systemd is just an init, i.e. the thing that boots the machine;
> once the application is running, systemd has nothing left to do." Using Phase 3 (cgroups), Phase 4
> (OOM/Restart) and the cloud box above, explain with an example **why** this sentence is incomplete:
> while the machine runs for hours, what job is systemd still doing?
>
> *(Answer: at the end of the phase)*

---
---

# 5.3 systemd Units and `systemctl`

## 5.3.1 What a unit is; the five core types `[mechanism]`

systemd manages everything through an abstraction called a **unit**. A unit is the general name for
"something systemd can start and manage" — a service, a timer, a mount point are each a unit. Each unit
is defined by a text file (`.service`, `.timer`, `.mount`...) and managed with `systemctl`. The five
types you need to know:

| Unit type | Extension | What it manages | Example |
|---|---|---|---|
| **service** | `.service` | A background process (daemon) | `ssh.service`, `nginx.service` |
| **socket** | `.socket` | A socket/port that triggers a service | `ssh.socket` — brings the service up on first connection |
| **timer** | `.timer` | Scheduled triggering (cron-like) | `logrotate.timer` |
| **target** | `.target` | A group of units / a boot stage | `multi-user.target`, `graphical.target` |
| **mount** | `.mount` | A filesystem mount point | `-` (bound via fstab in Phase 6) |

You'll work with `.service` most. `.target` is systemd's equivalent of the classic "runlevel" idea: a
target groups a set of units to say "bring the machine up to this point." `multi-user.target` = network
+ all services but no graphical desktop (a server's normal target); `graphical.target` adds the desktop
on top.

> **🔧 See it on your machine** 🟢 — see the running services and the boot target
>
> ```
> $ systemctl list-units --type=service --state=running | head -6
>   ssh.service          loaded active running OpenBSD Secure Shell server
>   cron.service         loaded active running Regular background program processing daemon
>   systemd-journald.service loaded active running Journal Service
>
> $ systemctl get-default
> multi-user.target
> ```
>
> `list-units` gives the units **running** at that moment; `get-default` gives the target the machine
> tries to reach on boot. Seeing this as `multi-user.target` on a server is normal — the graphical
> interface is never started at boot and consumes no resources for nothing.

## 5.3.2 `systemctl`: a service's lifecycle `[application]`

`systemctl` is systemd's command-line interface. Almost everything you do with a service is a
`systemctl <verb> <unit>` pattern:

| Command | What it does | Risk |
|---|---|---|
| `systemctl status ssh` | Shows the service's state + last log lines | 🟢 read only |
| `systemctl start ssh` | Starts the service **now** | 🟡 |
| `systemctl stop ssh` | Stops the service **now** | 🟡 (live: users affected) |
| `systemctl restart ssh` | Stops and restarts | 🟡 |
| `systemctl reload ssh` | Re-reads config without killing the process | 🟡 |
| `systemctl enable ssh` | **Permanently** turns on auto-start at boot | 🔴 |
| `systemctl disable ssh` | Turns off auto-start at boot | 🔴 |

> **🔧 See it on your machine** 🟢 — read `systemctl status` line by line
>
> ```
> $ systemctl status ssh
> ● ssh.service - OpenBSD Secure Shell server
>      Loaded: loaded (/lib/systemd/system/ssh.service; enabled; preset: enabled)
>      Active: active (running) since Wed 2025-09-17 08:12:03 UTC; 2h 5min ago
>    Main PID: 812 (sshd)
>      Memory: 6.1M
>         CPU: 240ms
>      CGroup: /system.slice/ssh.service
>              └─812 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
> ```
>
> Each line is a phase connection: `Loaded: ... enabled` → will it come up on boot (5.3.3); `Active:
> active (running)` → its current state; `Main PID: 812` → Phase 3's PID, you can look it up in the
> tree from here; `Memory / CPU` → Phase 4's resource accounting; **`CGroup:
> /system.slice/ssh.service`** → Phase 3.6's cgroup! Each service lives in its own cgroup; systemd
> measures and caps resources from here.

## 5.3.3 `enable` ≠ `start`: the phase's most critical distinction `[application]`

This single distinction sits at the root of the "the service was running but was gone after a reboot"
failure. The two commands manage **different times**:

- **`start`** = "start the service **now**." Its effect is immediate and **lost on reboot**. If you
  restart the machine, the service does not come back.
- **`enable`** = "from now on, start the service automatically **at every boot**." This is a permanent
  setting (it adds a symbolic link to the boot target). But it **does not start it now** — it only sets
  a marker for future boots.

So four states are possible:

| | `start`ed | not `start`ed |
|---|---|---|
| **`enable`d** | Running now **and** comes back on reboot ✅ | Off now, but comes up on reboot |
| **not `enable`d** | Running now, but **gone** on reboot ⚠️ | Fully off |

The top-right and bottom-left cells are the source of all the confusion. "I tested it, the service was
running" (`start`ed) but "it was gone after a reboot" (not `enable`d) — one of the most common mistakes
in production. The correct pattern is usually to do both: `systemctl enable --now nginx` (`--now` = both
enable and start).

> **🔧 See it on your machine** 🔴 — try the enable/start difference live (on a test machine)
>
> ```
> $ systemctl is-enabled cron        # does it come up on boot?
> enabled
> $ systemctl is-active cron         # is it running now?
> active
> ```
>
> **Undo:** These two commands are 🟢 (they only query). But if you want the real experiment — `enable`
> a service **without** starting it and reboot — put the original state back afterwards: if the service
> was `disabled` to begin with, restore it with `sudo systemctl disable <service>`. To undo a service
> you enabled: `sudo systemctl disable --now <service>` (removes the boot marker and stops it now).

## 5.3.4 Dependency and ordering: `After`, `Requires`, `Wants` `[concept]`

systemd's power is that it models "first this, then that" rules explicitly. In the `[Unit]` section of a
`.service` file, three keys establish this relationship — and confusing two of them is the classic cause
of the "service is enabled but didn't start" failure:

- **`After=network.target`** — **ordering.** "Start this service *after* `network.target`." It only
  sets the order; it doesn't care whether that target succeeded. A service that starts without the
  network may crash; `After` fixes the order to prevent that.
- **`Requires=postgresql.service`** — **hard dependency.** "For this service to run, PostgreSQL must
  also run; if PostgreSQL can't be started, this service **won't be started/is stopped** either." A
  hard bond.
- **`Wants=redis.service`** — **optional dependency.** "I'd like Redis to start too, but I'll start
  even if it doesn't." A soft bond; the flexible sibling of `Requires` and the most recommended in
  practice.

The critical subtlety: **`Requires` gives no ordering.** If you write `Requires=postgresql.service`
without `After=`, systemd starts both but **at the same time** — your service may start before
PostgreSQL is ready and crash. The correct pattern is to write both together:
`Requires=postgresql.service` **and** `After=postgresql.service`.

> **❓ Question that comes to mind: "I ran `systemctl enable`, `status` says 'enabled', but after a
> reboot the service is 'inactive (dead)' — why didn't it start?"**
>
> The two most common reasons: (1) A `Requires=` dependency of the service couldn't be started on boot
> (e.g. a mount or database it depends on didn't come up), so systemd didn't start your service either
> — `systemctl status yourservice` and `journalctl -u yourservice -b` will say so. (2) `After=` was
> missing, so the service started before something it needs (network, disk) and crashed. The diagnosis
> always goes through the same two commands (5.4). "Enabled but not running" is almost never about
> `enable` itself; it's a dependency story.

> **🤔 Think 5.3** — A `.service` file has only `Requires=mnt-data.mount`, with no `After=` line. The
> service is `enable`d. On some reboots it starts fine, on others it errors out with "couldn't run" —
> same machine, same file. Using the `Requires` vs `After` distinction from 5.3.4, explain why this
> **instability** (sometimes working, sometimes not) appears.
>
> *(Answer: at the end of the phase)*

---
---

# 5.4 journald and `journalctl`: The System's Central Logbook

## 5.4.1 Why journald exists; how to read `journalctl` `[application]`

In Phase 4 we read kernel messages with `dmesg`. But why a service (nginx, ssh, your app) crashed isn't
in `dmesg` — that's the service's own log. In the classic world each service wrote to its own file
(`/var/log/nginx/...`, `/var/log/auth.log`...) and to investigate an event you had to know which file
to look at. systemd **centralizes** this: `journald` (systemd's log service) collects the output of all
services into one structured logbook, and you look at that logbook through a single interface with
`journalctl`.

Three filters you need to know cover 90% of a failure investigation:

- **`journalctl -u ssh`** — show only a **specific service's** logs (`-u` = unit). The answer to "what
  is this service saying."
- **`journalctl -b`** — show only **this boot's** logs. `-b -1` is the previous boot. The question "what
  happened since the reboot" or "why did the last boot stall."
- **`journalctl -p err`** — show only messages of a **given priority** and above. `err`, `warning`,
  `crit`... To filter out noise and see what's serious.

These combine: `journalctl -u nginx -b -p err` = "nginx's, this-boot, error-level logs." In a cloud
failure this is the first reflex.

> **🔧 See it on your machine** 🟢 — read a service's logs for this boot
>
> ```
> $ journalctl -u ssh -b --no-pager | tail -4
> Sep 17 08:12:03 web-01 sshd[812]: Server listening on 0.0.0.0 port 22.
> Sep 17 09:44:10 web-01 sshd[3120]: Accepted publickey for deploy from 10.0.1.5
> Sep 17 10:01:55 web-01 sshd[3450]: Failed password for invalid user admin from 203.0.113.9
> ```
>
> In one command you see what the SSH service did across this boot: when it started listening, who
> logged in successfully (`deploy`), who failed (`admin` — most likely a bot). Phase 2's
> identity/authorization world and Phase 5's log world meet here.

> **💡 Cloud connection — the full reflex for the "service didn't come up" diagnosis:** If a service you
> deployed on an EC2 didn't come up, the order is always the same: (1) `systemctl status yourservice` →
> state + last few lines; (2) `journalctl -u yourservice -b --no-pager` → the service's full log for
> this boot, the crash reason is usually a single line here (port in use, config error, dependency
> didn't come, permission denied — Phase 2); (3) if needed, `systemctl cat yourservice` to read the
> unit file and check `After=`/`Requires=`. These three commands diagnose most cloud service failures in
> minutes.

> **🤔 Think 5.4** — An instance boots slowly: `systemd-analyze` shows total boot time as 95 seconds.
> At the top of `systemd-analyze blame` is `88.0s cloud-final.service`. Two hypotheses form: (a) the
> machine hardware is slow, (b) a service hangs waiting for something on boot. Which single command from
> 5.4 tells you which of the two is right and **what** was being waited on? (Hint: `cloud-final` is
> cloud-init's last step — 5.6.)
>
> *(Answer: at the end of the phase)*

---
---

# 5.5 Scheduled Jobs: cron vs systemd timer

## 5.5.1 Who runs recurring jobs `[concept]`

"Back up every night at 03:00", "send a metric every 5 minutes", "rotate logs every Sunday" — something
has to run these recurring jobs **on time**. There are two ways:

- **cron** — the classic, decades-old scheduler ("cron" — Greek *chronos*, time). Each user has a
  `crontab` (cron table); each line says "run this command at this time." Simple, everywhere, easy to
  learn:
  ```
  # min hr  dom mon dow    command
  0   3   *   *   *        /usr/local/bin/backup.sh    # every day at 03:00
  */5 *   *   *   *        /usr/local/bin/metrics.sh   # every 5 minutes
  ```
- **systemd timer** — systemd's scheduler. A `.timer` unit says "when", and an accompanying `.service`
  unit says "what runs." More detailed, but integrated into the systemd world.

## 5.5.2 Why the difference matters `[concept]`

Both do the same job; but the systemd timer's practical advantages over cron make it the modern choice:

| Feature | cron | systemd timer |
|---|---|---|
| Logging | Doesn't collect on its own; you redirect the output | Automatically in `journalctl -u ...` |
| Resource limit | None | As a service, a cgroup (Phase 3.6) can apply |
| Dependency | None | `After=`/`Requires=` (run once the network is ready) |
| Missed run | Lost if the machine was off | Can catch up on boot with `Persistent=true` |
| Status tracking | Hard | Clear via `systemctl status`, `list-timers` |

The most practical difference is **logging and diagnosis**: if a cron job crashes silently and you
didn't redirect its output anywhere, no trace remains — "the backup didn't run last night but it's not
clear why." Had the same job been a systemd timer + service, `journalctl -u backup.service` would give
you the crash line. This is a direct consequence of the centralized-log advantage we built in 5.4.

> **🔧 See it on your machine** 🟢 — see the active timers and the next runs on the machine
>
> ```
> $ systemctl list-timers --all --no-pager | head -4
> NEXT                        LEFT       LAST                        UNIT
> Wed 2025-09-17 00:00:00 UTC 13h left   Tue 2025-09-16 00:00:00 UTC logrotate.timer
> Wed 2025-09-17 06:00:00 UTC 19h left   Tue 2025-09-16 06:00:00 UTC apt-daily.timer
> ```
>
> Each line: when the next run is (`NEXT`), how long is left (`LEFT`), when it last ran (`LAST`). It
> answers "is the backup actually running" at a glance — cron doesn't have this clarity.

> **⚠️ Common misconception: "cron uses the system's clock, so there's no UTC/local-time confusion."**
>
> The opposite: cron and timer run according to the machine's **time zone**, and cloud servers are
> almost always in UTC. When you say "back up every day at 03:00" that's **UTC 03:00** — which can be
> hours off from your local time. Always think about the time zone explicitly for scheduled jobs; a job
> meant to "run at night" may run in the middle of the day and affect traffic.

---
---

# 5.6 Cloud-init: The Cloud's Boot-Time Magic

## 5.6.1 From AMI to running server: user-data `[application]`

Now all the pieces of this phase come together in a single cloud event. When you launch an EC2 instance,
under it is an **AMI** (Amazon Machine Image — a frozen disk image). But a thousand instances launch
from the same AMI and each is configured differently: one a web server, one a database, one with
different users. How does the same frozen image configure itself **differently** at boot time? The
answer: **cloud-init**.

cloud-init is a set of systemd services baked into cloud images (recall the `Detected virtualization
amazon` line from 5.1). During boot it does this:

1. As the instance starts, it reads a **metadata** and **user-data** from the cloud (on EC2, from the
   `http://169.254.169.254/...` metadata service).
2. **user-data** is a script or configuration you provide when launching the instance (a shell script
   or a `#cloud-config` YAML). You say "this instance should do the following at boot": install
   packages, add users, write files, start services.
3. cloud-init applies these instructions **once** at boot time — so the instance out of the same AMI
   comes up having transformed itself into the role you want.

```
#cloud-config
packages:
  - nginx
users:
  - name: deploy
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... deploy@laptop
runcmd:
  - systemctl enable --now nginx
```

This little user-data uses every piece of this phase: it installs a package (Phase 1's package
manager), adds a user + SSH key (Phase 2 — identity/authorization), and **both enables and starts**
nginx (`--now` — 5.3.3). So the transformation "from AMI to running web server" is your commands from
this phase, run automatically at boot time.

## 5.6.2 When it breaks: silently exploding user-data `[application]`

cloud-init's most insidious trait is its **silent** failure. If a line of your user-data script errors
(wrong package name, a permission problem, network not up yet), the instance still looks "started" —
SSH is open, the machine is up — but the configuration you expected **was not done**. nginx wasn't
installed, the user wasn't added. This is the "instance came up but there's no application" failure, and
the trace is in one place:

- **`/var/log/cloud-init-output.log`** — the **entire output and errors** of the user-data script are
  here. The answer to "why didn't nginx get installed" is a line in this file (e.g. `E: Unable to
  locate package nginx` — a typo).
- **`journalctl -u cloud-init`** — the systemd-side log of the cloud-init service (the same reflex as
  5.4).

> **🔧 See it on your machine** 🟢 — read what cloud-init did on an EC2
>
> ```
> $ cloud-init status
> status: done
> $ sudo tail -5 /var/log/cloud-init-output.log
> Get:3 http://.../nginx ...
> Setting up nginx (1.24.0) ...
> Created symlink /etc/systemd/system/.../nginx.service → /lib/systemd/system/nginx.service.
> Cloud-init v. 24.1 finished at Wed, 17 Sep 2025 08:12:40 +0000. Datasource DataSourceEc2.
> ```
>
> `status: done` shows cloud-init finished; the log's last lines show exactly what user-data did. The
> `Created symlink ... nginx.service` line is precisely the work `systemctl enable` does at boot
> (5.3.3) — you're seeing "enable adds a symbolic link to the boot target" live here.

> **💡 Cloud connection — the "instance came up but there's no application" diagnosis:** You launched an
> instance, you SSH in, but the service you expected isn't there. The order: (1) `cloud-init status` →
> did cloud-init finish / did it error; (2) `sudo cat /var/log/cloud-init-output.log` → which line of
> your user-data script exploded; (3) most often you'll see a typo, a permission problem, or a "trying
> to download a package while the network isn't ready yet" error. This is the moment you use the whole
> Phase 5 chain (boot → systemd → service → log) in a single cloud event.

> **🤔 Think 5.5** — A teammate wrote `runcmd: - systemctl start myapp` in their user-data (note:
> `start`, not `enable`). On the first boot myapp runs, everyone's happy. Two weeks later the instance is
> rebooted for routine maintenance and myapp never comes back — nobody changed the user-data. Using the
> `enable` ≠ `start` distinction from 5.3.3, explain **why** this happens and what the one-word fix is.
>
> *(Answer: at the end of the phase)*

---
---

# 5.7 When This Phase Breaks — Failure Signatures

As in Phases 3–4, the boot and service world has recognizable failure signatures. This table ties the
symptom you see to the right link of the chain:

| Symptom | Likely cause | Where to look first | Which section |
|---|---|---|---|
| No boot menu at all, black screen | GRUB / bootloader broken | Console (no SSH, no OS yet) | 5.1.1 |
| `Dropping to a shell! (initramfs)` | Root disk not found (wrong UUID/driver) | Console; `blkid`, kernel parameters | 5.1.2 |
| Boot freezes somewhere, never completes | A unit hangs waiting for something | Console; `systemd-analyze`, `journalctl -b` | 5.4, 5.6 |
| Service `enable`d but `inactive` on reboot | `Requires=` dependency didn't come / `After=` missing | `systemctl status`, `journalctl -u X -b` | 5.3.4 |
| "Worked in the test, gone after reboot" | `start`ed but not `enable`d | `systemctl is-enabled X` | 5.3.3 |
| Instance up, SSH works, no application | user-data script exploded silently | `/var/log/cloud-init-output.log` | 5.6.2 |
| Nightly job didn't run, no trace | cron output not redirected / time zone (UTC) | Move to a timer; `journalctl -u X`, `list-timers` | 5.5.2 |
| Service in a constant restart loop | `Restart=always` + crashes on every start | `journalctl -u X -b`, the crash line | 5.3.2 |

> **The lesson from this table:** In boot and service failures the first question is always "**which
> link of the chain am I on?**" For failures **before** the OS (GRUB, initramfs) systemctl/journalctl
> are useless — they aren't loaded yet; you read the console. For failures **after** the OS is up
> (service, cloud-init) the reflex is always the same two commands: `systemctl status X` and `journalctl
> -u X -b`. Tying the symptom to the right link is going to the right tool.

---
---

# Phase 5 — Answers to the think questions

## Answer 5.1 — the initramfs rescue shell

That screen tells you the boot chain reached the **initramfs** link but stalled there. Firmware ran,
GRUB loaded the kernel and initramfs, the kernel ran — the chain is intact up to here. But initramfs's
job was to find and mount the real root disk (5.1.2), and it says `UUID=abc-123 does not exist`: it
**couldn't find** the root disk it was looking for. The cause is usually a wrong/changed UUID or a
missing disk driver. The `(initramfs)` prompt shows you're still in the temporary RAM root, never
crossed over to the real disk.

You can't SSH in — because SSH is a **service**, and services are started by init (systemd) **after** the
real root disk is mounted (5.2, 5.3). You haven't even reached init yet; the network stack and sshd
aren't loaded. This kind of failure is handled only from the **console** (in the cloud, EC2 Serial
Console / attaching the disk to a rescue instance). This is exactly the live form of the lesson in
5.1.1: a failure in a link before the OS can't be reached with the OS's tools.

**Related section:** 5.1.2 (initramfs) · **Continues in:** 5.7 failure table (the initramfs row)

## Answer 5.2 — systemd is not just "the thing that boots"

The sentence is incomplete because systemd is as much a **service manager** as an init, and this second
role continues while the machine is up for hours (5.2.2). Concrete example: while a web service runs, (a)
systemd keeps that service in its own **cgroup** (Phase 3.6) and applies its `MemoryMax=` limit — i.e.
Phase 4's OOM protection is enforced **continuously** by systemd; (b) if the service crashes,
`Restart=always` makes systemd **restart** it — this is runtime behavior, not boot; (c) the service's
entire log keeps flowing through journald (5.4). So systemd isn't "started it and withdrew"; behind
every running service there's a live manager watching, limiting, reviving it. The "just an init" view
misses why systemd replaced classic init (5.2.2).

**Related section:** 5.2.2 (the service-manager role) · **Continues in:** 5.3.2 (the cgroup line), Phase
4.3 (OOM)

## Answer 5.3 — `Requires` present, `After` absent: instability

The instability's cause is that `Requires` establishes a **dependency** but not an **ordering** (5.3.4).
`Requires=mnt-data.mount` only says "these two should run together"; but because there's no
`After=mnt-data.mount`, systemd tries to start both **at the same time**. This creates a race: on some
boots the mount is ready a few milliseconds before the service → the service works; on other boots the
service starts before the mount is bound → it can't find the `/mnt/data` path → it crashes. Same file,
same machine, different timing = unstable result. The fix: add the `After=mnt-data.mount` line to the
unit as well — so systemd doesn't start the service before the mount completes, and the race is
eliminated. This is the classic root cause of "service is enabled but sometimes doesn't start."

**Related section:** 5.3.4 (`Requires` vs `After`) · **Continues in:** 5.4.1 (diagnosis via `journalctl
-u X -b`)

## Answer 5.4 — the command that diagnoses a slow boot

The right command is **`journalctl -b -u cloud-final.service`** (or, generally, that service's lines
within `journalctl -b`). `systemd-analyze blame` tells you only "who was slow" (cloud-final, 88s) but
not **why** — you can't tell whether the hardware is slow or a service is waiting on something. That
service's log for this boot, on the other hand, writes out plainly what it was waiting on: typically
cloud-final is waiting for a command in the user-data script (e.g. downloading a package from an
unreachable server) to time out. The log rules out the "hardware is slow" hypothesis and shows the
truth: "a service waited 88 seconds for something it couldn't reach and gave up." The lesson is the
essence of 5.4: `blame` gives *who*, `journalctl -u X -b` gives *why*; the diagnosis comes from
combining the two.

**Related section:** 5.4.1 (`-b`, `-u`) · **Continues in:** 5.6.2 (cloud-init/user-data failure)

## Answer 5.5 — user-data with `start`, lost on reboot

Because `systemctl start myapp` starts the service only **at that moment**; it leaves no permanent trace
about boot behavior (5.3.3). On the first boot, because cloud-init runs the user-data, `start` fires and
myapp comes up. But cloud-init applies user-data **only on the first boot** — it doesn't re-run on later
reboots. And because `enable` was never done, there's no "start this service at every boot" marker in
systemd. Result: reboot → user-data doesn't run (nobody to trigger start) + no enable marker → myapp
doesn't come up. The one-word fix: **`enable`** instead of `start` (ideally `enable --now`, which both
starts it now and sets the permanent marker). This is the cloud form of the "started / not enabled" cell
(gone on reboot ⚠️) of the table in 5.3.3.

**Related section:** 5.3.3 (`enable` ≠ `start`) · **Continues in:** 5.6.1 (user-data only on first boot)

---
---

# Phase 5 — Frequently asked questions

### Q1. How do I intervene in GRUB, is it dangerous?
While the boot menu is up you can stop the GRUB menu with a key (usually `Shift`/`Esc`) and temporarily
edit a kernel line with `e` (e.g. adding `single` for rescue). This is **temporary** — lost on reboot,
🟡. A permanent change is via `/etc/default/grub` + `update-grub` and is 🔴: get it wrong and the
machine won't boot. In the cloud you usually never touch GRUB; the AMI comes ready.

### Q2. When is `systemctl daemon-reload` needed?
When you edit a unit **file** (the `.service` content) by hand. systemd keeps unit definitions in memory;
if you change the file and don't say `daemon-reload`, systemd still uses the old definition. Rule: edited
a unit file → `systemctl daemon-reload` → then `restart`. Don't confuse it with `reload` (re-reading a
service's config); `daemon-reload` refreshes systemd's **own** unit definitions.

### Q3. Does `enable` start a service, does `start` turn it on at boot — I keep confusing them.
The mnemonic: **`start` = now, `enable` = every boot.** `start` is immediate and lost on reboot; `enable`
sets a marker for future boots but doesn't start it now. If you want both, `enable --now`. "Worked in the
test, gone after reboot" always means `start` was done, `enable` wasn't (5.3.3).

### Q4. Do journald logs disappear after a reboot?
By default journald may keep logs in RAM or a small area, and a previous boot can be lost on reboot. For
persistent logs the `/var/log/journal/` directory must exist (`sudo mkdir -p /var/log/journal` +
`systemctl restart systemd-journald`). If persistent, you can look at the previous boot with `journalctl
-b -1` — essential for saying "what happened on the last boot" after a crash.

### Q5. Should I use cron or a systemd timer?
For simple, single-line personal jobs that don't need logging, cron is fine. But in production, for jobs
that need to be logged/monitored/resource-limited, a systemd timer is superior (5.5.2) — because the job
becomes a service and enters the journald + cgroup + dependency world. Being able to see "did the backup
run last night" via `systemctl list-timers` and `journalctl -u backup` makes a big difference.

### Q6. `systemctl status` says "active (exited)" — is that an error?
No. `active (exited)` is usually for a **oneshot** service: services that run once and finish, not
staying up (e.g. a mount preparation or a setup step). "exited" = "it ran, finished successfully, there's
no process up anymore." Don't confuse this with `failed`; `failed` is a real error.

### Q7. An instance won't boot; there's no SSH either. Where do I start?
Think about which link of the chain you're on (5.7). No SSH means the OS most likely didn't fully come up
— so the failure may be at GRUB/initramfs/early boot and you can't reach OS tools (systemctl). In the
cloud, the path: look at the **console output** (EC2 System Log / Serial Console). There you'll see
whether it's an `initramfs` prompt, a unit wait, or an `fstab`-caused rescue mode (Phase 6). The symptom
determines the link; the link determines the tool.

---
---

# Phase 5 — Test yourself

Below are 18 questions. Section A probes the basic concepts, B the mechanism, C the application. Don't
look at the key before writing your answers.

## Section A — Basics

**1.** Write the five main links of the boot chain in order: from the power button to PID 1.

**2.** What is PID 1, why is its PID always 1, and what happens when it dies?

**3.** Why does initramfs exist — which "chicken-and-egg" problem does it solve?

**4.** What is the difference between `enable` and `start`? Which one is lost on reboot?

**5.** Name the five systemd unit types and write in one word what each manages.

**6.** What do the `-u`, `-b`, `-p` flags of `journalctl` do?

## Section B — Mechanism

**7.** Explain the three reasons systemd became the standard over the classic SysV init.

**8.** What is the difference between `Requires=` and `After=`? Why are the two usually written together?

**9.** Which earlier phase's concept does the `CGroup: /system.slice/ssh.service` line in `systemctl
status ssh` connect to, and what does it mean?

**10.** When does cloud-init apply user-data — on every boot, or only on the first boot? How does this
relate to the "my service was gone after a reboot" failure?

**11.** Explain two practical advantages of a systemd timer over cron (in terms of logging and
resources).

**12.** A service is defined with `Restart=always` and is in a constant restart loop. What is this a
symptom of and where do you look?

## Section C — Application

**13.** You `enable` a service **without** starting it and reboot. Does the service run after the reboot?
Why?

**14.** An EC2 instance came up, you SSH in, but the nginx you expected isn't there. Which commands do
you run, in order, to diagnose it?

**15.** A boot takes 95 seconds. Which two commands, in this order, do you use to find who slowed it down
and **why**?

**16.** You edited a `.service` file by hand but `systemctl restart` doesn't seem to see the change.
Which command did you skip?

**17.** In the console you see `Dropping to a shell! (initramfs)`. Which link of the boot chain are you
on, and why can't you SSH into this machine?

**18.** A cron job ("back up every day at 03:00") didn't run and there's no trace. Write two possible
reasons and suggest how to make this job more observable.

---

## Answer key

**1.** Firmware (BIOS/UEFI) → Bootloader (GRUB) → Kernel → initramfs → init (PID 1) [→ target]. (5.1.1)

**2.** PID 1 is the **first** process the kernel starts in user space; every other process descends from
it, so it's the root and ancestor of the tree. Its PID is 1 by definition. If it dies the kernel panics
("Attempted to kill init!") because the root of the tree is gone. (5.2.1)

**3.** Chicken-and-egg: to mount the root disk you need a driver, but the driver is often on the root
disk itself. initramfs is a temporary root filesystem living in RAM; it contains the needed drivers,
mounts the real root disk, then hands over control. (5.1.2)

**4.** `start` = start the service **now** (lost on reboot). `enable` = auto-start at every boot
(permanent marker, but doesn't start it now). Lost on reboot: `start`. (5.3.3)

**5.** service (a daemon), socket (a socket/port that triggers a service), timer (scheduled triggering),
target (a unit group / boot stage), mount (a filesystem mount point). (5.3.1)

**6.** `-u X` = only unit X's logs; `-b` = only this boot (`-b -1` previous boot); `-p X` = only priority
X and above (err, warning...). (5.4.1)

**7.** (a) Parallel startup — starts independent services at the same time, boot speeds up; (b)
dependency model — resolves order explicitly with `After`/`Requires`; (c) integrated management —
cgroup, log, restart, enable all from one interface (`systemctl`). (5.2.2)

**8.** `Requires=` establishes a **dependency** ("if X doesn't run, I don't either") but not **ordering**.
`After=` sets only the **order** ("start after X"). They're written together because `Requires` alone
starts a service at the same time as its dependency, creating a race/crash; `After` guarantees the
order. (5.3.4)

**9.** It connects to Phase 3.6's **cgroup**. Every systemd service lives in its own cgroup
(`system.slice/...`); systemd measures resources (memory, CPU) from here and limits them with
`MemoryMax`/`CPUQuota`. Phase 4's OOM protection works through this cgroup. (5.3.2)

**10.** cloud-init applies user-data **only on the first boot**. It doesn't run on later reboots. That's
why, if `start` (not enable) was used in user-data, the service comes up on the first boot but is gone
after a reboot — because neither does user-data re-run nor is there an enable marker. (5.6.1, 5.6.2)

**11.** (a) Logging: because a timer is a service, its output is collected automatically in `journalctl
-u X`; in cron, no trace remains unless you redirect the output. (b) Resources: a cgroup (Phase 3.6) can
apply to the service a timer triggers; a cron job's resources can't be limited. Also advantages like
`Persistent=`, `list-timers`. (5.5.2)

**12.** It means the service crashes every time it starts (`Restart=always` keeps bringing it back up,
and it keeps falling). You look at each attempt's crash line with `journalctl -u X -b` — usually a config
error, a busy port, a missing dependency, or a permission problem (Phase 2). (5.3.2, 5.7)

**13.** Yes, it runs. `enable` sets the "start at every boot" marker; even if you don't start it, the
next boot systemd sees that marker and starts the service. "Start now" via `start` is a separate thing.
(5.3.3)

**14.** (1) `cloud-init status` (did cloud-init finish / error); (2) `sudo cat
/var/log/cloud-init-output.log` (which line of the user-data exploded — e.g. wrong package name); (3) if
needed, `systemctl status nginx` / `journalctl -u nginx -b`. (5.6.2)

**15.** (1) `systemd-analyze blame` → who's slow (which service, how many seconds); (2) `journalctl -b -u
<that service>` → why it's slow (what it was waiting for). `blame` gives *who*, `journalctl` gives *why*.
(5.4.1)

**16.** `systemctl daemon-reload`. After changing a unit file, if you don't run this, systemd still uses
the old definition; then you must `restart`. (FAQ Q2)

**17.** You're on the **initramfs** link: the kernel loaded, initramfs ran but couldn't find and mount
the real root disk. You can't SSH in because SSH is a service and services start only after init
(systemd), which starts only after the real root disk is mounted — you haven't reached init yet. Handled
from the console. (5.1.2, Answer 5.1)

**18.** Possible reasons: (a) the script output wasn't redirected anywhere, so the error vanished
silently; (b) time zone — "03:00" is the machine's UTC time, which may be shifted from your local time.
Suggestion: move the job to a systemd timer + service; that way a crash is visible via `journalctl -u
backup`, and the last/next run is tracked via `list-timers`. (5.5.2)

## Scoring

| Correct | Assessment |
|---|---|
| 16–18 | You read the machine's lifecycle from end to end. Move to Phase 6. |
| 12–15 | Solid. Close the gaps with the table below. |
| 8–11 | The basics are settled but the dependency/cloud-init side is weak; go back to the relevant sections. |
| 0–7 | Review the phase, especially sections 5.3 (systemctl) and 5.6 (cloud-init). |

**Which question you missed → where to go back:**

| Question | Go back |
|---|---|
| 1, 3, 17 | 5.1 — boot chain, initramfs |
| 2, 7 | 5.2 — PID 1, systemd |
| 4, 5, 13 | 5.3.1–5.3.3 — unit types, enable/start |
| 8, 9 | 5.3.4, 5.3.2 — dependency, cgroup |
| 6, 12, 15, 16 | 5.4 — journalctl, the diagnosis reflex |
| 11, 18 | 5.5 — cron vs timer |
| 10, 14 | 5.6 — cloud-init, user-data |

---
---

# Phase 5 — Closing and Bridge to Phase 6

## What you carry from this phase

Phases 3–4 taught you to read a running system; Phase 5 gave you **how it comes to exist**. Now the
systemd at the root of the process tree is no longer a box: it's a live manager that brings the machine
from boot to a running state, brings services up in order, caps their resources (Phases 3–4's
cgroup/OOM), and collects their logs. You can now split the single sentence "the instance won't boot"
into the links of the chain, tell apart "enabled but didn't start" from "started but not enabled," and
explain how an EC2 transforms from an AMI into a running server (cloud-init + user-data).

## Where Phase 6 connects to this

We dodged one failure type in this phase without fully opening it: **a wrong line in `fstab` drops the
machine into rescue mode on boot.** Some of the "instance won't boot" rows in the 5.7 table are actually
a **disk/mount** problem — and mount is the world of systemd's `.mount` units. Phase 6 goes right here:
you'll see how a disk comes from a raw block device (`/dev/nvme0n1`) to a mounted, persistent
filesystem, how `/etc/fstab` can lock up the boot, and how to persistently mount an EBS volume **without
breaking the boot** (by UUID, not by device name). The disk side of Phase 5's "where does the boot chain
break" question is Phase 6.

> **🤔 Phase output — test yourself:** Of the two `systemctl status` outputs below, **which** causes
> trouble after a reboot, and why?
>
> ```
> (top)     Loaded: loaded (...; enabled;  ...)   Active: active (running)
> (bottom)  Loaded: loaded (...; disabled; ...)   Active: active (running)
> ```
>
> Both are running **right now** (`active (running)`). But the difference on the `Loaded:` line is
> critical: the **top** service is `enabled` — it comes back on reboot, no problem. The **bottom**
> service is `disabled` — it was started by hand now (`start`) but has no boot marker; if the machine is
> rebooted this service **won't** come up (5.3.3). "Running now" and "comes up on reboot" are different
> things; `Active` tells you the first, `Loaded: enabled/disabled` the second. If you can read this
> without hesitation, you're ready for Phase 6.
>
> **Lab 5 (from the mentor):** See the running services with `systemctl list-units --type=service
> --state=running`. Find who slowed the boot with `systemd-analyze blame`. Read the SSH service's logs
> for this boot with `journalctl -u ssh -b`. (All 🟢 — read only.)

---

> **Navigation:** [◀ Checkpoint Quiz 2](Checkpoint_Quiz_2.md) · **Phase 5** · [Phase 6 — Storage and Filesystems ▶](Phase_6_Storage_and_Filesystems.md)



