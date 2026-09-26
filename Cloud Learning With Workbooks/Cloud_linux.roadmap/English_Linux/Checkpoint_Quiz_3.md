# Checkpoint Quiz 3 — Phases 5–6: Boot, systemd and Storage

> **Navigation:** [◀ Phase 6 — Storage and Filesystems](Phase_6_Storage_and_Filesystems.md) · **Checkpoint Quiz 3** · [Phase 7 — Networking (OS Layer) ▶](Phase_7_Networking_and_Connectivity.md)

---

## What does this quiz measure?

The end-of-phase "Test yourself" checks a single phase. A checkpoint quiz measures something **different**:
*"Can you connect these two phases?"*

Phase 5 gave you how a machine **comes to exist**: the boot chain, PID 1, systemd, units, dependency order.
Phase 6 gave you the same machine's **persistent layer**: block device, filesystem, mount, `fstab`. These
two intersect exactly **at boot**: every line in `/etc/fstab` is really a `.mount` unit to systemd (Phase
5), and that unit runs at a stage of the boot sequence (Phase 5). A wrong `fstab` line (Phase 6) is
therefore a **boot** failure (Phase 5) — and why you must look at the console instead of `journalctl` lies
exactly at this intersection. The most common cause of the "instance won't boot" event lives at this point
where the two phases meet.

**How to study:**

- Solve on paper; don't move to the next before writing an answer.
- The answer key tells you which **intersection of phases** each question sits at — a missed question points
  not to one phase but to the **bridge** between two.
- 21 questions: Section A (connection reasoning, 1–9), Section B (scenario: the "instance won't boot after a
  reboot" event, 10–15), Section C (reading commands and output, 16–21).
- Target time: ~1 hour. But time doesn't matter; what matters is being able to justify in one sentence *why*
  each answer is what it is.

> **🤔 Before you start:** Complete this sentence in your own words: *"If an `fstab` line is wrong the machine
> won't boot, but I also can't see the logs with `journalctl -xb` — because..."* This one question contains
> both phases (boot stage + storage). Set your answer aside; we'll see it in Question 4.

---

# Section A — Connection reasoning (1–9)

Each question here asks you to combine at least two phases (mostly Phase 5 × Phase 6). Give a short but
reasoned answer.

**1.** In Phase 6 we said "writing a line in `fstab` makes the mount permanent." In Phase 5 we said "systemd
manages everything as a unit." **What kind of unit** does an `fstab` line become on the systemd side, and
why does that make the "`fstab` = the `enable` of the mount" analogy (Phase 6, 6.2.2) technically correct?

**2.** In Phase 5 we saw the `systemctl enable`≠`start` distinction. In Phase 6 we saw the `fstab`≠`mount`
distinction. Explain in one sentence that these two distinctions are two faces of the **same** concept:
"acts now, gone on reboot" vs "re-established on every boot."

**3.** In Phase 6 we saw why a UUID is safer than the device name (nvme1n1). In Phase 5 we implied that the
boot sequence must be deterministic. How does device names being able to shift between boots explain why an
`fstab` mount is a "fragile node" in systemd's dependency graph (Phase 5: After/Requires)?

**4.** In Phase 5 we said "failures before PID 1 are invisible to `systemctl`/`journalctl` — look at the
console." An `fstab` error (Phase 6) drops the machine into rescue mode. Is this failure **visible** with
`journalctl -xb` or not — and why does your answer depend on which stage of the boot chain (Phase 5) it hung
at?

**5.** In Phase 6 we saw that the `nofail` option rescues boot for a data disk. In Phase 5's language (unit,
dependency, target), explain what `nofail` does: why, when a `.mount` unit with `nofail` fails, does systemd
not halt `local-fs.target` and therefore boot?

**6.** In Phase 5 we saw where journald writes logs (default: memory/`/run`, persistent:
`/var/log/journal`). In Phase 6 we implied `/var` can be mounted on a separate disk. If `/var/log/journal`
is mounted on a separate disk and that disk doesn't come up at boot, what chicken-and-egg problem arises for
persistent logs?

**7.** In Phase 5 we saw cloud-init runs on first boot. Assume you added a new EBS volume to `fstab` (Phase
6). How do cloud-init and `fstab` split the job of "preparing a disk at boot" — which runs **once** (first
boot), which runs **every boot**?

**8.** In Phase 5 we saw boot time with `systemd-analyze blame`. In Phase 6 we implied a `.mount` unit can
wait on its dependencies. How does a slow or hung disk mount (Phase 6) lengthen boot time (Phase 5), and what
kind of line reveals it in `blame` output?

**9.** In Phase 5 we saw `.service` units ordered with `After=`/`Requires=`. A database service keeps its
data on `/data` (a separate EBS volume). What dependency (`After=`/`Requires=` + the mount unit name) must
this service's unit declare so it doesn't start before the disk is mounted — and what failure occurs if this
declaration is missing?

---

# Section B — Scenario: "Instance won't boot after a reboot" (10–15)

> **Event:** An engineer attached a new 100 GB EBS data volume to a `t3.large` Ubuntu instance. They
> formatted it with `mkfs.ext4 /dev/nvme1n1`, mounted it to `/data`, moved the app over, everything worked.
> To make it permanent they added this line to `/etc/fstab`:
> `/dev/nvme1n1  /data  ext4  defaults  0  2`
> Then they rebooted "so the new settings take." The instance **never accepted SSH again**. The "System
> log" in the EC2 console shows:
> `[[ TIME ]] Timed out waiting for device /dev/nvme1n1.`
> `[[ TIME ]] Dependency failed for /data.`
> `You are in emergency mode. Give root password for maintenance (or press Ctrl-D to continue):`

**10.** First diagnosis: Is this a Phase 5 (boot) problem, a Phase 6 (disk) problem, or the intersection of
both? Match each of the lines "Timed out waiting for device," "Dependency failed for /data," and "emergency
mode" to its phase.

**11.** Root-cause reasoning: The disk is physically present (the attach is still active). So why "Timed out
waiting for device /dev/nvme1n1"? Using the device-name-shift possibility of 6.2.3, explain why that name
might not have been found on reboot. (Hint: was another volume attached/detached, or did NVMe numbering
change?)

**12.** Why did boot **halt** (emergency mode) — instead of just skipping that mount? Explain via the
`fstab` line's `pass` field (`0  2`) and the missing **option**. Which single-word option, had it been
added, would let boot continue and SSH come up even if the disk were missing?

**13.** Recovery: The engineer entered the root password in emergency mode (or used the EC2 rescue flow). To
fix `/etc/fstab`, which two changes must they make (6.2.3), and after fixing, **before rebooting**, which
single command (6.6.1) verifies the line will no longer lock up boot?

**14.** Why wasn't `journalctl` enough? Because they were left without SSH, the engineer had to look at the
EC2 console's "System log." Combining Phase 5.1 (before/after PID 1) and 5.4 (journald): was this particular
failure **visible** in `journalctl -xb` output (did systemd reach the emergency target), and why was the
console still a more reliable source?

**15.** Prevention + generalization: What is the **single habit** the engineer must adopt so this never
happens again (6.6.1)? Also summarize this scenario in Phase 5's language: "a wrong `.mount` unit, via the
`local-fs.target` dependency, blocked reaching the `___` target."

---

# Section C — Reading commands and output (16–21)

Read each output fragment below and answer what's asked. This section measures "what you actually see in a
real terminal."

**16.** `lsblk` output:
```
NAME        SIZE TYPE MOUNTPOINTS
nvme0n1      20G disk
└─nvme0n1p1  20G part /
nvme1n1     100G disk
```
The `nvme1n1` disk doesn't show in `df -h`. Is this (a) a broken disk, (b) an unmounted disk, or (c) an
unformatted disk? Which two clues in the output (6.1.2) decide it, and which step(s) remain to put it in
`df`?

**17.** Two `df` outputs:
```
$ df -h /data                              $ df -i /data
Filesystem     Size Used Avail Use% ...     Filesystem      Inodes  IUsed IFree IUse% ...
/dev/nvme1n1    98G  40G   53G  44% ...     /dev/nvme1n1  6553600 6553600     0  100% ...
```
The app errors "No space left on device." Which of these two outputs explains the failure, what's the name of
the failure (6.4.2), and why does growing the disk **not** fix it?

**18.** `systemctl status data.mount` output:
```
● data.mount - /data
     Loaded: loaded (/etc/fstab; generated)
     Active: failed (Result: timeout)
      Where: /data
       What: /dev/nvme1n1
```
What does the `(/etc/fstab; generated)` in the `Loaded:` line tell you (Phase 5 unit × Phase 6 fstab
connection)? Which Section B line (Questions 10-11) does `Active: failed (Result: timeout)` match?

**19.** `blkid` output:
```
/dev/nvme1n1: UUID="a1b2c3d4-...-e5f6" TYPE="xfs"
```
But the `/etc/fstab` line reads: `UUID=a1b2c3d4-...-e5f6  /data  ext4  defaults,nofail  0  2`. What happens
when this line is attempted (6.3.1), does boot halt (`nofail` is present), and which single field on the line
do you change to fix it?

**20.** `journalctl -u data.mount -b` output:
```
systemd[1]: Mounting /data...
systemd[1]: data.mount: Mount process exited, code=exited, status=32/n/a
systemd[1]: data.mount: Failed with result 'exit-code'.
systemd[1]: Failed to mount /data.
```
That this log starts with `systemd[1]` (Phase 5.2.1) tells us boot **reached** which stage? That is, is this
failure before or after PID 1 — and why did `journalctl` work this time (compare with Question 14)?

**21.** `growpart` + `df` sequence:
```
$ lsblk
nvme0n1  60G disk
└─nvme0n1p1 20G part /
$ df -h /
/dev/nvme0n1p1  20G  ...  /
```
The EBS volume was grown from 20G→60G in the console. `lsblk` shows the disk as 60G, the partition as 20G;
`df` still shows 20G. Which two commands do you run and in which order (6.6.2), and which layer does each
command grow?

---

## Answer Key

**1.** An `fstab` line becomes a **`.mount` unit** on the systemd side (systemd-fstab-generator reads `fstab`
at boot and produces one `.mount` unit per line; its name is derived from the mount point, e.g. `/data` →
`data.mount`). This makes `fstab` technically "the enable of the mount": just as `enable` starts a service on
every boot, an `fstab` line automatically activates the corresponding `.mount` unit on every boot. · *Phase
5.3 × Phase 6.2.2* — **2.** Both are the distinction between "temporary running state" and "permanent config
re-established at boot": `start`/`mount` act on the kernel's current state and are lost on reboot;
`enable`/`fstab` do nothing immediately but re-establish automatically on every boot. · *Phase 5.3.3 × Phase
6.2.2* — **3.** Because a device name (`nvme1n1`) can shift between boots, if `fstab` is written by device
name the generated `.mount` unit depends on a device that may not exist; systemd waits for that device (a
`.device` unit), times out, and the unit fails — that's the fragile node in the graph. Using a UUID pins the
dependency to a disk-unique, non-shifting identity. · *Phase 5.3.4 × Phase 6.2.3* — **4.** It **is** visible —
but conditionally. An `fstab` error hangs boot at the **init stage** (PID 1 running, systemd dropping to the
emergency target), not **before** PID 1; so since systemd and journald are already up, `journalctl -xb` shows
the mount failure. For failures before PID 1 (GRUB, initramfs) journalctl is useless; but this failure is
after that. Still, since SSH doesn't come up, in practice you reach the log via the EC2 console. · *Phase 5.1
× Phase 6.2.3* — **5.** `nofail` makes the generated `.mount` unit bind to `local-fs.target` as **optional**
(not boot-critical) rather than **required**: even if the unit fails, the target is considered "met," the
dependency chain isn't broken, and boot continues to `multi-user.target`. Without `nofail`, a failed mount
collapses `local-fs.target` and boot halts. · *Phase 5.3.4 × Phase 6.2.3* — **6.** If the persistent journal
`/var/log/journal` is mounted on a separate disk and that disk doesn't come up at boot, systemd can't find the
persistent place to write early boot logs — and the very logs explaining "why the disk didn't come up" can't
be written to that missing disk either. This chicken-and-egg loses the trail of the failure at early boot;
that's why the journal is usually placed on root or a `nofail`-protected location. · *Phase 5.4 × Phase 6.2.2*
— **7.** cloud-init runs **once**, on first boot (disk prep, first-time format/mount, user data); `fstab`
re-establishes the mount on **every boot**. The split: cloud-init "one-time setup/bootstrap," `fstab`
"permanent, repeated attaching." If you don't write a permanent disk into `fstab`, the mount is lost on the
first reboot after cloud-init. · *Phase 5.6 × Phase 6.2.2* — **8.** A hung mount holds up `local-fs.target`
until its `.device`/`.mount` unit times out; that wait adds directly to boot time. In `systemd-analyze blame`
output a high-duration `*.mount` (or `*.device`) line reveals it. · *Phase 5.3 × Phase 6.2.1* — **9.** The
service unit must declare `After=data.mount` **and** `Requires=data.mount` (or `RequiresMountsFor=/data`) on
the mount unit; then the service won't start before the disk is mounted. If this is missing, the service
starts against an empty `/data` directory (before the mount), writes data to the root disk or errors "no
data" — and when the mount arrives later the data appears "lost." · *Phase 5.3.4 × Phase 6.2.2*

**10.** The **intersection** of both. "Timed out waiting for device /dev/nvme1n1" → Phase 6 (device name/block
device). "Dependency failed for /data" → Phase 5 (unit dependency graph, `.mount` failed). "emergency mode" →
Phase 5 (systemd halted boot, target unreachable). · *Phase 5.3.4 × Phase 6.2.3* — **11.** `fstab` was written
by **device name** (`/dev/nvme1n1`); NVMe device names can shift between boots and by attach order. On reboot
(or when another volume comes in between), the kernel gave that disk a different name (e.g. `nvme2n1`);
`/dev/nvme1n1` is now either absent or something else. systemd waited for that device unit and timed out. With
a UUID the name shift would be harmless. · *Phase 6.2.3* — **12.** The line has **no** `nofail` and the mount
is required by the boot-critical `local-fs.target`, so when the `.mount` failed systemd deemed the target
unmet and dropped into emergency mode — instead of just skipping that mount. Had it been added, **`nofail`**
would let boot continue and SSH come up even if the disk were missing. (`pass 2` also signals an fsck attempt
at boot, but the real disaster-maker is the missing `nofail`.) · *Phase 6.2.3* — **13.** Two changes: (1)
replace the device name with the **UUID** (get it with `blkid`); (2) add **`nofail`** to the options. After
fixing, before rebooting, run `sudo mount -a` — if it doesn't error, the line no longer locks up boot
(`findmnt --verify` as extra assurance). · *Phase 6.2.3 × Phase 6.6.1* — **14.** Since the failure is **after**
PID 1 (systemd dropped to the emergency target), `journalctl -xb` **would** show the mount failure; but since
the machine didn't accept SSH, the engineer couldn't get into those logs. The EC2 "System log" (serial
console) is independent of SSH and shows every stage including emergency mode — so on an unreachable machine
the console is the more reliable source. · *Phase 5.1 × Phase 5.4* — **15.** The single habit: verify with
**`sudo mount -a` after every `fstab` edit, before rebooting**. In Phase 5's language: "a wrong `.mount` unit,
via the `local-fs.target` dependency, blocked reaching the **`multi-user.target`**" (the target with
SSH/services). · *Phase 5.3.4 × Phase 6.6.1*

**16.** (b) an **unmounted** disk — not broken. Two clues: `nvme1n1` shows as `TYPE disk` (the kernel
recognizes it, it's healthy) and `MOUNTPOINTS` is empty (no mount). Whether it's formatted is unclear; to put
it in `df` you need `mkfs` if necessary, then definitely `mount` (and `fstab` for persistence). · *Phase
6.1.2* — **17.** The second output (`df -i`) explains it: `IUse% 100%` — **inode exhaustion**. Even though
`df -h` shows 44% free, no new file can be created. Growing the disk increases the **data-block** budget but
the inode budget is separate; the fix is to clean up the small files. · *Phase 6.4.2* — **18.** `(/etc/fstab;
generated)` tells you this `.mount` unit wasn't hand-written but **auto-generated from `fstab`** (the generator
from Question 1) — the exact connection between the Phase 5 unit and the Phase 6 `fstab`. `Active: failed
(Result: timeout)` matches the "Timed out waiting for device" of Questions 10-11: the device didn't come, the
unit timed out. · *Phase 5.3 × Phase 6.2.3* — **19.** The disk's real format is **xfs**, but the `fstab` type
says `ext4`; the mount fails with "wrong fs type / unknown filesystem." Since `nofail` is present, boot does
**not** halt, only `/data` isn't mounted. Fix: change the **type** field on the line from `ext4` → `xfs`. ·
*Phase 6.3.1* — **20.** That the log starts with `systemd[1]` shows **PID 1 (systemd) is running**, i.e. boot
**reached** the init stage (Phase 5.2.1). So the failure is **after** PID 1; that's why journald was up this
time and `journalctl` could record the mount error — the opposite of Question 14's "if it were before PID 1 it
would be invisible." · *Phase 5.2.1 × Phase 5.4* — **21.** First `sudo growpart /dev/nvme0n1 1` (extends the
partition 20G→60G — grows the `lsblk` partition line), then `sudo resize2fs /dev/nvme0n1p1` (grows the ext4
filesystem — makes `df` show 60G). Order: the lower layer (partition) first, the upper layer (filesystem)
next. · *Phase 6.6.2*

---

## Scoring

| Correct | What it means |
|---|---|
| 18-21 | You've built the bridge that joins boot and storage at boot time. Ready for Phase 7. |
| 14-17 | Good. Reread the **bridge** sections your missed questions point to (table below). |
| 9-13 | You know the phases individually but struggle at the intersection. Redo the Section B scenario. |
| 0-8 | Walk Phase 5 and Phase 6 separately again; especially 5.3 (unit/target) and 6.2.3 (fstab). |

Missed question → the bridge to return to:

| Question | Bridge (section × section) |
|---|---|
| 1, 2 | `fstab` → `.mount` unit; enable≠start / fstab≠mount (5.3 × 6.2.2) |
| 3, 11, 19 | UUID vs device name / filesystem type (5.3.4 × 6.2.3, 6.3.1) |
| 4, 14, 20 | Boot stage and journalctl visibility (5.1/5.2/5.4 × 6.2.3) |
| 5, 12, 15 | `nofail` and the `local-fs.target` dependency (5.3.4 × 6.2.3) |
| 6, 7, 8 | journald/cloud-init/boot time × mount (5.4/5.6 × 6.2) |
| 9 | Service → mount dependency (5.3.4 × 6.2.2) |
| 10, 13 | Scenario diagnosis and recovery (5.3.4 × 6.6.1) |
| 16, 17, 21 | Reading lsblk/df -i/growpart (6.1.2, 6.4.2, 6.6.2) |
| 18 | `.mount` unit × fstab generator (5.3 × 6.2.3) |

---

## Closing

This quiz tested the intersection at the heart of the most expensive Linux event you'll meet many times in
your career — "instance won't boot": a disk (Phase 6) defined wrong causes a boot (Phase 5) to halt. Carry
two lessons together: (1) persistence always means "some config somewhere is re-established on every boot" —
whether it's `systemctl enable` or `fstab`, if that config is wrong it locks up boot; (2) when a machine is
unreachable, the first place your eye goes is not `journalctl` but the **console** — because depending on
where in the boot chain the failure is, journalctl itself may never have come up.

Phase 7 takes you outside the machine, to the network. There too you'll see the same "existing ≠ reachable"
gap — this time via ports and security groups instead of disks.

---

> **Navigation:** [◀ Phase 6 — Storage and Filesystems](Phase_6_Storage_and_Filesystems.md) · **Checkpoint Quiz 3** · [Phase 7 — Networking (OS Layer) ▶](Phase_7_Networking_and_Connectivity.md)
