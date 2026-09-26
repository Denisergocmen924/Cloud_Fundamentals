# Phase 11 — Observability and Troubleshooting: When Something Breaks

> **Navigation:** [◀ Phase 10 — Automation and Scripting](Phase_10_Automation_and_Scripting.md) · **Phase 11** · [Phase 12 — Bridge to the Cloud ▶](Phase_12_Bridge_to_the_Cloud.md)

---

## Where we are coming from

From Phase 0 to Phase 10, the end of every phase had a **"When this phase breaks"** table: a permission error in
Phase 2, zombie/OOM in Phase 3, a mount shift in Phase 6, unreachability in Phase 7, installed-but-not-running in
Phase 8, "permissions are right but blocked" in Phase 9, a script that silently does the wrong thing in Phase 10.
Each was a separate symptom-cause mapping. Phase 11 gathers **all of them into a single methodology**. Because in
the real world, when something breaks, no one tells you "this is a Phase 6 problem" — there is only a symptom
("the app won't open") and you must find the evidence layer by layer.

This phase sits specifically on top of three phases:

- **Phase 4 — resources.** You learned to read CPU/RAM/IO with `top`/`free`/`iostat`. Phase 11 makes these part
  of a **diagnostic reflex**: when someone says "the system is slow," which resource do you look at first.
- **Phase 5 — logs.** You saw journald and `journalctl`. Phase 11 makes the log the **first source of
  evidence**: the answer to "why didn't a service start" is almost always in the log.
- **Phase 10 — silent failure.** You saw that a script can error and keep going. Phase 11 teaches how to find
  the evidence of that silent failure systematically.

## The question of this phase

Up to Phase 10 we said "how do I set up, run and automate a system." Phase 11 asks the exact opposite:

> *"When something breaks — the app won't open, a service won't start, the system is slow — how do I find the
> real cause without panic, not by guessing but with evidence, narrowing down layer by layer (application log →
> service status → resource → network → kernel); and which tool do I reach for, when?"*

Two ideas sit at the center of this phase: **systematic narrowing** (eliminate layer by layer instead of trying
things at random) and the **three instinct questions** ("What is the system doing right now?", "Why is it
unreachable?", "Where between boot and service did it break?"). A cloud engineer's most valuable skill is not
knowing a tool by heart — it is **searching for the evidence in the right order**.

By the end of this phase you will be able to narrow a problem down layer by layer; know which tool to reach for
when (top, ps, ss, lsof, strace, dmesg, journalctl, iostat/vmstat); and demonstrate the "instance healthy but
app not" distinction in the cloud with evidence.

---

## By the end of this phase

- You will be able to apply the **layer-by-layer narrowing** methodology (application log → service status →
  resource → network → kernel)
- You will gain **log mastery**: `journalctl` (unit/time/boot filters), `/var/log`, `logrotate`; what
  auth.log/syslog/dmesg tell you
- You will know when to reach for each **diagnostic tool**: `top`/`htop`, `ps`, `ss`, `lsof`, `strace` (the
  final arbiter), `dmesg`, `iostat`/`vmstat`/`free`, `/proc/<pid>/`
- You will be able to turn the **three instinct questions** into a decision tree ("why unreachable" →
  permission/ownership/mount/MAC/capability)
- **Cloud:** you will be able to make the "instance healthy but app not" distinction; and collect evidence
  remotely with CloudWatch Logs + SSM

---

## The phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 11.1 | Logs — the first source of evidence | `[application]` | journalctl, /var/log, logrotate; what auth/syslog/dmesg say |
| 11.2 | Layer-by-layer debugging methodology | `[concept]` | Not panic but systematic narrowing: log→service→resource→network→kernel |
| 11.3 | Tool mastery | `[application]` | Which tool when: top/ps/ss/lsof/strace/dmesg/iostat |
| 11.4 | The three instinct questions | `[concept]` | "What is it doing", "why unreachable", "boot to service" |
| 11.5 | Collecting evidence in the cloud | `[application]` | CloudWatch Logs + SSM; "instance healthy but app not" |
| 11.6 | When this phase breaks | — | Signatures of a broken diagnostic reflex |

> **How to work through this phase:** Almost every command in this phase is 🟢 **read-only** — `top`, `ps`,
> `ss`, `journalctl`, `dmesg`, `strace` observe the system, they do not change it. This makes it one of the
> safest phases: try freely, you break nothing. The most instructive habit of this phase: while solving a
> problem, **write down the evidence in hand before moving to the next step** — "what did the log say," "what is
> the service status," "is the port listening." Instead of trying random commands, run each command to test a
> hypothesis. The most instructive experiment: deliberately break a service (bind to the wrong port, corrupt a
> config) and find the evidence layer by layer — starting from the log until you reach the real cause.

---
---

# 11.1 Logs — The First Source of Evidence

## 11.1.1 `journalctl`: the modern system's memory `[application]`

In Phase 5 you saw journald (systemd's central log collector). In troubleshooting it is your **first stop**: if
a service didn't start, if an app crashed, the evidence is almost always in the journal. Critical filters:

```bash
journalctl -u nginx.service        # logs of a single unit (Phase 8!)
journalctl -u nginx -f             # live follow (like tail -f)
journalctl -u nginx --since "10 min ago"   # time filter
journalctl -b                      # logs of this boot
journalctl -b -1                   # the previous boot (critical after a crash)
journalctl -p err                  # only error level and above
journalctl -k                      # kernel messages (dmesg equivalent)
```

Why the first stop? Because when a service says it "didn't start," `systemctl status` shows you only the last
few lines; the actual cause (config parse error, port conflict, permission denied) is usually visible with
`journalctl -u <service> -e`. The reflex: if something didn't start, **read the log first, then guess**.

## 11.1.2 `/var/log` and logrotate: traditional logs and size control `[application]`

Even though journald collects everything, many traditional services still write under `/var/log`:

- **`/var/log/syslog`** (Debian/Ubuntu) or **`/var/log/messages`** (RHEL) — general system logs
- **`/var/log/auth.log`** — authentication: SSH logins, `sudo` usage, failed logins (Phase 9 security
  forensics starts here)
- **`dmesg`** / **`/var/log/kern.log`** — the kernel ring buffer: hardware, drivers, the OOM killer (Phase 4!),
  disk errors

If logs grow forever they fill the disk (the "disk 100% → system freezes" scenario you saw in Phase 6).
**`logrotate`** exists for exactly this: it rotates logs periodically, compresses them, deletes old ones. The
configs under `/etc/logrotate.d/` define when each service's log file is rotated.

> **🔧 See it on your machine** 🟢 — read failed SSH logins in auth.log
>
> ```
> $ sudo journalctl -u ssh --since today | grep -i "failed\|invalid"
> $ sudo grep -i "failed password" /var/log/auth.log | tail -20    # Ubuntu
> ```
>
> Who tried to log into a server, who ran `sudo` — it is all here. Phase 9's "evidence" idea becomes concrete
> here: after a security incident, the first place you look is auth.log. On your own machine, try `ssh
> localhost` a few times with the wrong password and see how the attempts land in the log.

> **⚠️ Common misconception: "No log means no problem."**
>
> **Absence** of a log is evidence — but usually not of the absence of a problem, but of the problem being
> **deeper**. If an app wrote no log at all, there are three possibilities: (1) it never started (broken service
> unit → journalctl -u), (2) it started but writes its log elsewhere (to a file instead of stdout → check the
> config), (3) logrotate deleted the log or the disk is full so it can't write (`df -h` → Phase 6). "No log" is
> not an answer, it is a new question: why is there none?


> **🤔 Think 11.1** — A service fails at start and `systemctl status myapp` shows only "failed" plus two unhelpful lines. (a) What do you run next, and why? (b) The service crashed and the machine rebooted overnight — which filter reaches the log from before the reboot? (c) You find no log at all: what are the three possibilities?
>
> *(Answer: at the end of the phase)*
---
---

# 11.2 Layer-by-Layer Debugging Methodology

## 11.2.1 Not panic, but systematic narrowing `[concept]`

Beginners' biggest mistake is **trying random commands** when something breaks: "maybe I'll restart it," "maybe
I'll fix the permissions." This can mask the symptom without finding the real cause, or make things worse. The
master engineer instead **narrows down layer by layer** — at each layer asking "is the problem here?" and
eliminating with evidence:

1. **Application log** — what does the app say? (journalctl -u, /var/log, the app's own log) — often the answer
   is here.
2. **Service status** — is the service running, in what state? (`systemctl status`, `is-active`, `is-failed` —
   Phase 8) Enabled but not started, or constantly restarting?
3. **Resource** — is CPU/RAM/IO exhausted? (`top`, `free`, `iostat` — Phase 4) Did the OOM killer trigger
   (`dmesg`)?
4. **Network** — is the port listening, does DNS resolve, is it reachable? (`ss -tulpn`, `curl`, `dig` — Phase
   7) Phase 7's three lenses: listening / firewall / bind address.
5. **Kernel** — is there a hardware/driver/disk error? (`dmesg`, `journalctl -k`) The lowest layer.

This order is not random: it goes from **the most likely and cheapest check to the deepest and rarest**. Most
problems are solved in the first two layers; going down to the kernel is rare. The essence of the methodology:
gather evidence at each layer, move to the next only when you have eliminated the current one.

![Figure 11.1 — Layer-by-layer debugging methodology. A decision flow starting from the symptom "the application won't open," passing through five layers: (1) Application log — journalctl -u / /var/log; (2) Service status — systemctl status; (3) Resource — top / free / iostat / dmesg (OOM); (4) Network — ss -tulpn / curl / dig; (5) Kernel — dmesg / journalctl -k. At each layer the question "is the evidence here?" is asked; if found the cause is identified, if not you descend one layer. On the side an arrow indicating the direction "from the cheapest and most likely check to the deepest and rarest." At the bottom, the lesson: not random trials but narrowing down with evidence, layer by layer.](../diagrams/png/lx-11-01-debugging-layers.png)

> **🤔 Think 11.2** — A web app "won't open" (timeout in the browser). The only thing you have is this. (a) If
> you apply the five layers above in order, which single command would you run at each layer and what would you
> look for? (b) If `systemctl status` says "active (running)" but the browser still won't open, which layer do
> you move to and why? (c) Why is this order (log→service→resource→network→kernel) better than the "restart it
> first" reflex?
>
> *(Answer: at the end of the phase)*

---
---

# 11.3 Tool Mastery — Which Tool When

## 11.3.1 Observation tools and their right moments `[application]`

Every tool has a **question**. Mastery is not memorizing the tool, but knowing which one to reach for at which
question:

| Tool | The question it answers | When |
|---|---|---|
| `top` / `htop` | What is the system doing right now? Which process eats CPU/RAM? | "The system is slow" (Phase 4) |
| `ps aux` | The current process list, state (Z/D), parent | When looking for a specific process (Phase 3) |
| `free -h` | What is the state of RAM/swap? | "Did memory fill up" (Phase 4) |
| `iostat` / `vmstat` | Disk IO / general system bottleneck | "Is the slowness disk or CPU" (Phase 4) |
| `ss -tulpn` | Who is listening on which port? | "Unreachable" — the listening lens (Phase 7) |
| `lsof` | Which process holds which file/socket open? | "File deleted but space not freed," "who has the port" |
| `dmesg` / `journalctl -k` | What does the kernel say? OOM, disk error, driver | Hardware/OOM suspicion (Phase 4) |
| `journalctl -u` | What did this service say? | Service didn't start/crashed (Phase 5, 8) |
| `strace` | Which **system call** is this process making, where is it stuck? | **Final arbiter** — when everything else is silent |
| `/proc/<pid>/` | Everything about this process (fd, limits, status, maps) | Deep inspection |

## 11.3.2 `strace`: the final arbiter `[mechanism]`

When all other tools stay silent — no log, status "running," resources plentiful but the app hangs — `strace`
comes in. `strace` shows live **every system call** a process makes (the user space → kernel space transitions
you saw in Phase 1):

```bash
strace -p 1234                     # attach to a running process
strace -f -e trace=openat,connect ./program   # trace file opens and network connections
```

Why the "final arbiter"? Because it shows, without lying, **what the process is actually trying to do**: which
file it tries to open and gets `EACCES` (permission denied — Phase 2!), which address it `connect`s to and gets
`ECONNREFUSED` (Phase 7!), which call it hangs on with `EAGAIN`. A vague problem like "the app can't find its
config file" turns, with `strace`, into the certainty "there — it tries to open `/etc/app/config.yml` and gets
`ENOENT` — the file is in the wrong place." It is heavy and slows the process, so it is a **last resort**; but
it speaks wherever everything else is silent.

> **🔧 See it on your machine** 🟢 — trace what a command touches with strace
>
> ```
> $ strace -e trace=openat cat /etc/hostname 2>&1 | grep -i hostname
> openat(AT_FDCWD, "/etc/hostname", O_RDONLY) = 3
> $ strace -e trace=openat cat /no/such/file 2>&1 | tail -3
> openat(AT_FDCWD, "/no/such/file", O_RDONLY) = -1 ENOENT (No such file or directory)
> ```
>
> Did you see the `ENOENT` in the second example? That is the power of `strace`: the abstract "file not found"
> error turns into **exactly which call got which error**. Phase 2's `EACCES`, Phase 7's `ECONNREFUSED` become
> concrete evidence here.

> **💡 Cloud connection — no SSH, no strace:** `strace` is powerful, but you need to be able to **get into** the
> machine. In the cloud, if your SSH access to an instance is broken (Phase 7 — SG, key), you can't strace. That
> is exactly why in the cloud you set up two-layer observation: (1) stream the app logs **out** (CloudWatch Logs
> — 11.5), read them without entering the machine; (2) use SSM Session Manager to get into the machine (even if
> the SSH port is closed). If locally `strace` is the final arbiter, in the cloud "moving the evidence outside
> the machine" is the final strategy.

> **🤔 Think 11.3** — A service shows "active (running)" via `systemctl status`, writes no log, CPU/RAM are
> normal in `top`, but it doesn't respond to requests (hung). (a) Why did these observations (status, log,
> resource) fall short — what can each not see? (b) If you attach to the process with `strace -p <pid>` and see
> it "hung" on a `read()` or `futex()` call, what does that tell you? (c) Why is `strace` the "final arbiter"
> here — what does it see that the other tools cannot?
>
> *(Answer: at the end of the phase)*

---
---

# 11.4 The Three Instinct Questions

A cloud engineer's reflex can be reduced to three questions. Each is a decision tree and connects to earlier
phases.

## 11.4.1 "What is the system doing right now?" `[concept]`

This is the **resource** question (Phase 4). Its answer is `top`/`htop` (CPU/RAM), `iostat`/`vmstat` (IO), `ss`
(network connections). "The system is slow" is a vague complaint; this question makes it concrete: is CPU at
100% (a process in a loop?), did RAM fill up (swapping, OOM near?), is IO the bottleneck (slow disk?), or is it
the network (a connection waiting?). The reflex: "slow" → which resource → which process.

## 11.4.2 "Why is it unreachable?" — a decision tree `[concept]`

This is the most frequent and most phase-unifying question. When someone says "I can't reach it" (a file, a
service, a port), you eliminate layer by layer:

- **Permission?** (Phase 2) — `ls -l`, user/group, rwx. Are you getting `EACCES`?
- **Ownership?** (Phase 2) — does the file belong to the right user?
- **Mount?** (Phase 6) — is the filesystem mounted, what do `df`/`mount` say? "To exist ≠ to be reachable."
- **MAC?** (Phase 9) — permissions are right but is AppArmor/SELinux blocking? `dmesg | grep -i denied`,
  `ausearch`. "Permissions are right but blocked."
- **Capability?** (Phase 9) — does the process have the needed capability (`CAP_NET_BIND_SERVICE` for port
  <1024)?
- **Network?** (Phase 7) — is the port listening, firewall, bind address (three lenses)?

This decision tree unifies Phase 2 + 6 + 7 + 9 into a single reflex: "unreachable" is not one cause, it can be
any of six different layers — and you find the right layer with evidence.

## 11.4.3 "Where between boot and service did it break?" `[concept]`

This is the **time/sequence** question (Phase 5). A machine came up but the app isn't running — where in the
chain did it break? Boot → init (systemd) → target → service unit → app. `systemd-analyze` (boot time),
`systemctl list-units --failed` (failed units), `journalctl -b` (this boot's full log). Phase 5's "running
state vs persistent definition" idea is critical here: is the service **enabled** (should start at boot) but
**failed**? Or never enabled at all? You find which link of the chain snapped.

> **🤔 Think 11.4** — An EC2 instance came up after a reboot, `ping` responds, SSH works, but the web app that
> should be running on it isn't. (a) With which commands do you check the "boot to service" chain (init →
> target → unit → app) step by step? (b) If `systemctl is-enabled myapp` returns "disabled," what is the
> problem, and how does this connect to Phase 5's "running state vs persistent definition" idea? (c) If it
> returns "enabled" but "failed," with which single command do you find the cause?
>
> *(Answer: at the end of the phase)*

---
---

# 11.5 Collecting Evidence in the Cloud

## 11.5.1 "Instance healthy but app not" `[application]`

This is the cloud's most confusing situation and the cloud counterpart of this whole phase's methodology. The
AWS console says **"healthy"** for an EC2 instance — but your app isn't working. Why? Because AWS's health
checks (status checks) look at two things: (1) **System status** — is the underlying physical hardware/hypervisor
fine; (2) **Instance status** — is the OS reachable over the network, did it boot. **Neither looks at your app.**
The OS may have booted perfectly, but the systemd service inside it (Phase 8) may be in a failed state — AWS
doesn't know this, it says "healthy."

This is the sharpest cloud form of Phase 8's "installed ≠ running as a service" and Phase 5's "running state vs
persistent definition": **instance being healthy ≠ app running.** To find the evidence you descend not to AWS's
health check but to your own layer-by-layer methodology (11.2) — but inside the machine.

## 11.5.2 CloudWatch Logs and SSM: reaching inside the machine without SSH `[application]`

Locally you look at the log with `journalctl`; in the cloud two AWS tools take its place:

- **CloudWatch Logs** — if you install the CloudWatch agent (Phase 10 — via user-data), the machine's logs
  (journald, /var/log, app log) **flow to AWS**. So you can read the logs without ever entering the instance,
  even if it has crashed/been deleted. This is the cloud application of 11.3's "move the evidence outside the
  machine" strategy.
- **SSM Session Manager** — lets you open a shell into the machine over IAM, with no SSH port (22) open at all,
  no key (Phase 7 + 9). Even if your SSH access is broken (wrong SG, lost key), with SSM you get inside and can
  run `journalctl`, `systemctl status`, `strace`.

The reflex: in the cloud, when an instance says "healthy but no app" — (1) first look at CloudWatch Logs
(without SSH), (2) if that's not enough get inside with SSM, (3) inside, apply this phase's layer-by-layer
methodology (log → service → resource → network). AWS's "healthy" label is the start of the diagnosis, not the
end.

> **💡 Cloud connection — set up observation ahead of time:** This phase's biggest cloud lesson is: you must set
> up the evidence-collection infrastructure **before** a problem occurs. You can't install the CloudWatch agent
> after an instance has crashed; the logs are already lost inside. So you put the CloudWatch agent and SSM into
> the AMI (Phase 8) or user-data (Phase 10) from the start. "Observability" is not a fix, it is a **design
> decision** — when you build the system, you also design how the evidence flows out.


> **🤔 Think 11.5** — The AWS console shows "2/2 checks passed" for an instance, yet the application returns errors. A colleague says "AWS says it is healthy, so the problem is not in the machine." (a) Why is he wrong? (b) In what order do you collect evidence without opening SSH? (c) What should have been done before the incident?
>
> *(Answer: at the end of the phase)*
---
---

# 11.6 When This Phase Breaks — Signatures of a Broken Diagnostic Reflex

This phase "breaking" is the breaking not of a system but of the **diagnostic reflex**. A wrong diagnosis hides
the real problem:

| Symptom | Likely cause (wrong reflex) | The right way | Related section |
|---|---|---|---|
| "I restarted it, fixed" but it breaks again | You masked the symptom, didn't find the cause | Start from the log, find the root cause | 11.2.1 |
| You tried random commands for hours | No methodology, panic | Layer by layer: log→service→resource→network→kernel | 11.2.1 |
| "No log, so no problem" | You misread the absence of a log | "Why is there no log" is a new question (didn't start / elsewhere / disk full) | 11.1.2 |
| You looked at CPU but the problem was IO | You looked at the wrong resource | `iostat`/`vmstat` — which resource (Phase 4) | 11.3.1 |
| "Unreachable" → you only looked at the firewall | You checked one layer | Permission/mount/MAC/capability/network — six layers | 11.4.2 |
| Thought the app was running because AWS said "healthy" | You misunderstood the health check | Status check ≠ app; get inside (SSM) | 11.5.1 |
| Instance crashed, no log | You tried to set up observation afterward | Install the CloudWatch agent from the start | 11.5.2 |
| Everything silent, app hung, no way out | You didn't reach for strace | `strace -p <pid>` — the final arbiter | 11.3.2 |

> **The lesson from this table:** The two big ideas of this phase lie beneath every row. The first is **evidence,
> not guesses**: a system's most dangerous "fix" is the one that masks the symptom without finding the cause ("I
> restarted it, fixed") — because the problem comes back, next time at a worse moment. The layer-by-layer
> methodology exists for exactly this: gather evidence at each step, move to the next only after eliminating the
> current layer. The second is **move the evidence outside ahead of time**: in the cloud, when the machine
> disappears the evidence inside it disappears too; you design observability while building the system, not when
> the crisis hits. And remember: this phase's tools are almost entirely read-only — don't be afraid to observe;
> what to fear is trying to fix without observing.

---
---

# Phase 11 — Answers to the think questions

## Answer 11.1 — Read the journal first: `journalctl -u`, and `-b -1` after a reboot

(a) `journalctl -u myapp -e` — `systemctl status` shows only the last few lines, while the real cause (config parse error, port conflict, permission denied) is usually visible in the journal. The reflex: read the log first, then guess. (b) `journalctl -b -1 -u myapp` — the previous boot's logs. (c) "No log" is a new question, not an answer: (1) the service never started (a broken unit — `journalctl -u`), (2) it writes its log elsewhere (to a file instead of stdout — check the config), (3) logrotate deleted the log or the disk is full so it cannot write (`df -h`, Phase 6).
**Related section:** 11.1.1-11.1.2 · **Continues in:** 11.2.1 (systematic narrowing)

## Answer 11.2 — Applying the five layers in order, why better than "restart first"

(a) Each layer's single command and what to look for: **(1) Log** — `journalctl -u myapp -e` → look for the
error line, exception, "connection refused," "permission denied." **(2) Service** — `systemctl status myapp` →
active or failed, constantly restarting; "active (running)" or "activating." **(3) Resource** — `top` + `free
-h` → is CPU 100%, did RAM fill, `dmesg | grep -i oom` → did the OOM killer trigger. **(4) Network** — `ss
-tulpn | grep :8000` → is the port listening, which address is it bound to (127.0.0.1 or 0.0.0.0 — Phase 7).
**(5) Kernel** — `dmesg -T | tail` → disk error, driver, hardware. (b) If `systemctl status` says "active
(running)" but the browser won't open, you move to the **network layer** (4) — because a service running as a
process does not mean "the port and address it listens on are right"; most likely the app bound to `127.0.0.1`
(Phase 7's classic trap) or the firewall is blocking. (c) This order is better than "restart first" because
restarting **masks the symptom without eliminating the cause**: the problem comes back, and moreover restarting
can clear the logs too, destroying the evidence. Layer-by-layer instead accumulates evidence at each step, finds
the root cause and fixes it for good.
**Related section:** 11.2.1 · **Continues in:** 11.4 the three instinct questions.

## Answer 11.3 — Why status/log/resource fell short; why strace is the final arbiter

(a) The three observations cannot see: **`systemctl status`** only tells you the process **exists**, not what it
is doing — a "running" process may well be in a lock (deadlock) or waiting forever on an I/O. **The log** only
shows what the app **chose to write** — if the app hung on a call and never reached the log line, the log is
silent. **`top`** only shows **resource consumption** — a process can hang without eating CPU/RAM (e.g. waiting
on `read()` from a socket, CPU is 0% but no work is being done). (b) Seeing the process hung on a `read()` or
`futex()` call with `strace -p <pid>` tells you: the process is **waiting for something** — if `read()`, data
that never arrives from a file/socket (maybe the other side isn't responding — Phase 7), if `futex()` a lock
(a lock another thread never released → deadlock). (c) `strace` is the final arbiter because it shows without
lying **which system call, with which argument, with which return value** the process is stuck on — it opens the
"what the process is asking the kernel and what answer it gets" layer that the other tools cannot see. Wherever
they stay silent, it speaks.
**Related section:** 11.3.2 · **Continues in:** 11.5 cloud evidence without SSH.

## Answer 11.4 — The boot-to-service chain; enabled vs failed; running state vs persistent definition

(a) The chain step by step: `systemctl get-default` (which target it booted to) → `systemctl list-units
--failed` (any failed unit) → `systemctl status myapp` (the unit's state) → `journalctl -u myapp -b` (what it
said this boot) → if needed the app's own log. (b) If `systemctl is-enabled myapp` returns "disabled," the
problem is: the service is **not defined to start automatically at boot** — someone started it by hand with
`systemctl start` (Phase 5's "running state") but did not add it to the persistent definition (Phase 5's
"persistent definition") with `systemctl enable`; on reboot the "running state" vanished, there was never a
"persistent definition," and the service didn't come back. This is exactly Phase 5's "running state ≠ persistent
definition" lesson. Fix: `systemctl enable myapp`. (c) If it returns "enabled" but "failed," the single command
is: `journalctl -u myapp -b` — the unit tried to start at boot but the log tells you why it couldn't (config
error, missing dependency, permission).
**Related section:** 11.4.3 · **Continues in:** 11.5 "instance healthy but app not."

## Answer 11.5 — Healthy instance ≠ running application: CloudWatch Logs first, then SSM

(a) The status checks look at two things only: system status (the underlying hardware/hypervisor) and instance status (the OS booted and is reachable). Neither looks at your application; the systemd service inside may be in a failed state and AWS still says "healthy" — the cloud form of "installed ≠ running as a service". (b) First CloudWatch Logs (no SSH needed), then SSM Session Manager to get inside over IAM, and inside apply the layer-by-layer methodology: log → service → resource → network. (c) The CloudWatch agent and SSM had to be in the AMI or user-data **beforehand** — you cannot install the agent after the machine has crashed; observability is a design decision.
**Related section:** 11.5.1-11.5.2 · **Continues in:** 12.4.2 (CloudWatch)

---
---

# Phase 11 — Frequently asked questions

**Q1 — What should I do first when something breaks?** Don't panic and restart. First read the log (`journalctl
-u <service> -e`), then narrow layer by layer: log → service → resource → network → kernel (11.2.1).

**Q2 — What's the difference between `journalctl` and `/var/log`?** journalctl reads systemd's central journal
(with unit/time/boot filters); `/var/log` is the traditional file logs (syslog, auth.log, kern.log). Modern
services write to the journal, many traditional ones still to /var/log (11.1).

**Q3 — When do I use `strace`?** When everything else is silent: no log, status "running," resources normal but
the app hung. `strace -p <pid>` shows which system call the process is stuck on — the final arbiter (11.3.2).

**Q4 — When someone says "unreachable," where do I look?** Not one place, six layers: permission (Phase 2),
ownership (Phase 2), mount (Phase 6), MAC (Phase 9), capability (Phase 9), network (Phase 7). Eliminate with the
decision tree (11.4.2).

**Q5 — If AWS says "healthy," does that mean the app is running?** No. Status checks look at hardware and OS
reachability, not your app. "Instance healthy ≠ app running" — get inside and check (11.5.1).

**Q6 — If the instance crashed, how do I look at the logs?** If you set up CloudWatch Logs ahead of time, the
logs flowed to AWS and you read them even without the machine. If SSH is broken you get inside with SSM Session
Manager. Set up observation from the start (11.5.2).

**Q7 — "The system is slow" — where do I start?** With the "which resource" question: `top` (CPU), `free`
(RAM), `iostat` (IO), `ss` (network). Slowness is not one thing; first find which resource is the bottleneck
(11.4.1, Phase 4).

---
---

# Phase 11 — Test yourself

Write your answers on paper, then compare with the answer key. Target: 14+ out of 18.

## Section A — Definition and mechanism

1. Name the five layers of the layer-by-layer debugging methodology in order. Why this order?
2. What do the `journalctl -u`, `-b`, `-p err`, `-k` filters do?
3. What does `strace` show and why is it the "final arbiter"?
4. Name the six layers of the "unreachable" decision tree (which phase).
5. Which tools answer "what is the system doing right now" and which resource does each look at?
6. What does auth.log contain and which phase's forensics does it connect to?
7. Why does logrotate exist, which Phase 6 scenario does it prevent?
8. The difference between `top` and `iostat` — which answers which question?

## Section B — Apply and diagnose

9. A web app "won't open." Which commands do you run in the first three layers and what do you look for?
10. `systemctl status` says "active (running)" but the browser won't open. Which layer do you move to, which
    command, what do you look for?
11. A service didn't come back after a reboot; `is-enabled` says "disabled." What is the problem, what is the
    fix (Phase 5 connection)?
12. The app is hung, no log, status "running," resources normal. What is your next step and why?
13. The AWS console says "healthy" but the app isn't working. With which two AWS tools do you collect the
    evidence?
14. "Your permissions to a file are right but it's unreachable." Which two layers (Phase 9) do you check and
    with which command?

## Section C — Reasoning and connection

15. Why is "I restarted it, fixed" a dangerous diagnostic habit?
16. Why is "no log" not an answer but a new question? Name three possible causes.
17. Why must you set up observability in the cloud "before a problem occurs"? Connect the "instance crashed, no
    log" situation.
18. How does the fact that almost all of this phase's tools are read-only shape the diagnostic philosophy?

---

## Answer key

1. Application log → service status → resource → network → kernel; from the cheapest/most-likely check to the
   deepest/rarest (11.2.1). — 2. `-u` unit log, `-b` this boot, `-p err` error level and above, `-k` kernel
   messages (11.1.1). — 3. Every system call the process makes (with the return value); it is the final arbiter
   because it opens, without lying, what the process is actually doing when everything else is silent (11.3.2).
   — 4. Permission (Phase 2), ownership (Phase 2), mount (Phase 6), MAC (Phase 9), capability (Phase 9), network
   (Phase 7) (11.4.2). — 5. `top`/`htop` (CPU/RAM), `iostat`/`vmstat` (IO), `ss` (network); "which resource is
   the bottleneck" (11.4.1). — 6. SSH logins, sudo usage, failed logins; Phase 9 security forensics (11.1.2). —
   7. Prevents logs from filling the disk (rotates, compresses, deletes); the "disk 100% → system freezes" of
   Phase 6 (11.1.2). — 8. `top` CPU/RAM/process ("what is it doing"), `iostat` disk IO ("is the slowness disk")
   (11.3.1).

9. (1) `journalctl -u myapp -e` → the error line; (2) `systemctl status myapp` → failed/running; (3) `top`+`free`
   → CPU/RAM/OOM (11.2.1). — 10. The network layer; `ss -tulpn | grep <port>` → is the port listening, bound to
   127.0.0.1 (Phase 7 trap) (Answer 11.2b). — 11. The service was started by hand as a "running state" but not
   added to the "persistent definition" (enable); on reboot it vanished; fix `systemctl enable myapp` (Phase 5)
   (Answer 11.4b). — 12. `strace -p <pid>` — shows which system call (read/futex) it is stuck on; the other
   tools can't see this (11.3.2). — 13. CloudWatch Logs (read logs without SSH) + SSM Session Manager (get
   inside); status check ≠ app (11.5). — 14. MAC (`dmesg | grep -i denied`, AppArmor/SELinux) and capability
   (`CAP_NET_BIND_SERVICE`); Phase 9 (11.4.2). — 15. It masks the symptom without finding the cause; the problem
   comes back, and restarting can clear the evidence (the log) (11.2.1, 11.6). — 16. Absence of a log can be
   evidence of the problem's depth, not its absence: (1) it never started, (2) it writes elsewhere, (3) disk
   full / logrotate deleted it (11.1.2). — 17. When the machine crashes/is deleted the evidence inside is lost;
   you can't install the agent afterward; observability is a design decision, you put the CloudWatch agent in
   the AMI/user-data from the start (11.5.2). — 18. Read-only-ness feeds the "don't be afraid to observe,
   understand before you change the system" philosophy; gather evidence, then make one single, evidence-based,
   permanent fix (11.6).

## Scoring

| Correct | What it means |
|---|---|
| 16-18 | Your systematic diagnostic reflex is set. You are ready for Phase 12 (Bridge to the Cloud). |
| 13-15 | Good. Re-read the sections of the questions you missed (especially 11.2 and 11.4). |
| 9-12 | The basics are there but the tools are scattered. Study the 11.3 table and the three instinct questions. |
| 0-8 | Walk through the phase again; deliberately break a service on your own test machine and find the evidence layer by layer. |

Missed question → section to return to:

| Question | Section |
|---|---|
| 1, 9, 15 | 11.2.1 Methodology |
| 2, 6, 7, 16 | 11.1 Logs |
| 3, 12 | 11.3.2 strace |
| 4, 14 | 11.4.2 "Why unreachable" |
| 5, 8 | 11.3.1 / 11.4.1 Tools and resource |
| 10 | 11.2 + Phase 7 network |
| 11 | 11.4.3 Boot to service |
| 13, 17 | 11.5 Cloud evidence |
| 18 | 11.6 Diagnostic philosophy |

---
---

# Phase 11 — Closing and Bridge to Phase 12

## What you carry from this phase

Phase 11 turned you from "someone who panics when something breaks" into "someone who finds the evidence layer
by layer." You gained two lasting ideas: **evidence, not guesses** (the systematic narrowing log → service →
resource → network → kernel; when to reach for each tool; strace as the final arbiter) and **move the evidence
outside ahead of time** (in the cloud reaching inside the machine without SSH via CloudWatch Logs + SSM,
"instance healthy ≠ app running"). This phase also gathered all the previous phases' "when this phase breaks"
notes into a single methodology: unreachability is one of six layers, a boot problem is a link in the chain,
slowness may be a resource bottleneck — and you find the right layer with evidence.

## Where Phase 12 connects

Phase 11 gave you "finding what broke"; **Phase 12 unifies the whole journey into a single story**: the journey
of an empty Ubuntu AMI from boot to production. Every piece you learned in Phases 0-11 (AMI, cloud-init,
systemd, mount, SG, IAM, capability, log) settles onto an AWS concept in Phase 12 and they all meet in a single
chain. Phase 11's diagnostic reflex serves you there too: when any link of that chain snaps, you find where it
broke with this phase's layer-by-layer methodology.

> **🤔 Phase output — ask yourself:** How does this phase's "move the evidence outside ahead of time" idea
> connect to Phase 12's "from empty AMI to production" story? Think about which **step** of the boot journey you
> should place an instance's observability (CloudWatch agent, SSM) at — the AMI (Phase 8), or user-data (Phase
> 10)? And why should this be a part of the boot chain from the start, not a "feature to add later"?
>
> **🧪 Lab 11 idea (on your own test machine):** Set up a deliberate fault and solve it layer by layer: (1)
> Write a simple web server service (`python -m http.server 8000` as a systemd unit), enable+start it. (2) Now
> **break** it: make it bind to `127.0.0.1:8000` in the config. (3) Try to reach it from another
> machine/terminal — timeout. (4) Apply the methodology: `journalctl -u` (what does the log say), `systemctl
> status` (is it running), `ss -tulpn` (which address is it bound to — there's the evidence: 127.0.0.1!). (5)
> Fix it (`0.0.0.0`), verify. This lab combines Phase 7's bind trap with Phase 11's diagnostic reflex — you
> find the evidence in the network layer.

---

> **Navigation:** [◀ Phase 10 — Automation and Scripting](Phase_10_Automation_and_Scripting.md) · **Phase 11** · [Phase 12 — Bridge to the Cloud ▶](Phase_12_Bridge_to_the_Cloud.md)
