# Phase 12 — Bridge to the Cloud: From an Empty AMI to Production

> **Navigation:** [◀ Phase 11 — Observability and Troubleshooting](Phase_11_Observability_and_Troubleshooting.md) · **Phase 12** · [Appendix A — Command Glossary ▶](Appendix_A_Command_Glossary.md)

---

## Where we are coming from

From Phase 0 to Phase 11, each phase taught **one Linux fundamental**: the mental model, the shell, permissions,
processes, memory, boot, storage, network, packages, security, automation, diagnosis. Each is a stone on its own.
Phase 12 weaves all these stones into **a single arch**. Because your job as a cloud engineer is to seat each of
these fundamentals onto its counterpart in AWS, and to be able to explain each step of an empty machine's journey
to production based on **why it must be there**, grounded in Linux fundamentals.

This phase teaches no new topic — it **unifies everything you learned into one story**. So this phase's "where we
are coming from" is not a single phase but **the whole journey**:

- **Phase 0 + 8** → AMI (a frozen Linux)
- **Phase 5 + 10** → cloud-init / user-data (boot-time configuration)
- **Phase 7** → SSH keypair / SSM (access)
- **Phase 9** → IAM instance role (authority without keeping a key on disk)
- **Phase 6** → EBS + fstab/UUID (persistent storage)
- **Phase 5 + 8** → systemd service (the application's life cycle)
- **Phase 4 + 11** → CloudWatch agent / Logs (remote observability)
- **Phase 3.6** → cgroup + namespace → ECS/EKS (kernel-sharing Linux)
- **Phase 7 + 9** → Security Group vs host firewall / AppArmor (two defense layers)

## The question of this phase

Throughout Phase 0-11 we always said "**how** does this Linux fundamental work." Phase 12 asks the final, most
unifying question:

> *"When I take an empty Ubuntu AMI and turn it into a running production service — cloud-init → user/SSH → EBS
> mount → package → systemd service → log/monitoring → hardening — which Linux fundamental does each step rest
> on, and **why** must it be exactly there? When I see an AWS concept (AMI, user-data, IAM role, SG), can I say
> which Linux truth lies beneath it?"*

One idea sits at the center of this phase: **AWS abstractions are built on top of Linux.** An AMI is a frozen
Linux, a Lambda is a Linux you don't see, an ECS container is a kernel-sharing Linux. What separates a cloud
engineer from a junior is being able to **see beneath** the abstraction: when an instance says "healthy but no
app," they don't panic, because they can read the systemd service (Phase 8), the bind address (Phase 7), the IAM
role (Phase 9) beneath it.

By the end of this phase you will be able to connect each step of an empty AMI's journey from boot to production
to a Linux fundamental; state the Linux truth beneath each major AWS concept; and know which layer to descend to
(which link of this journey) when a problem arises.

---

## By the end of this phase

- You will be able to narrate the seven steps of an empty Ubuntu AMI's **journey from boot to production**
  (cloud-init → user/SSH → EBS mount → package → systemd → log → hardening) in order
- You will be able to state **which Linux fundamental** each step rests on and **why** it is there
- You will be able to open the Linux truth beneath the major **AWS concepts**: AMI, cloud-init, IAM role, EBS,
  systemd, CloudWatch, ECS/EKS, Lambda, SG
- You will be able to distinguish the **two defense layers** (Security Group vs host firewall/AppArmor)
- When a problem arises you will be able to find **which link of this journey snapped** with your diagnostic
  reflex (Phase 11)

---

## The phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 12.1 | AMI = a frozen Linux | `[application]` | Phase 0 + 8: package layer + mental model seat onto AWS |
| 12.2 | cloud-init / user-data and IAM role | `[application]` | Phase 5 + 10 + 9: boot-time config + no key on disk |
| 12.3 | EBS + fstab and the systemd service | `[application]` | Phase 6 + 5 + 8: persistent storage + app life cycle |
| 12.4 | Two defense layers and observability | `[concept]` | Phase 7 + 9 + 11: SG vs host, CloudWatch |
| 12.5 | Container and Lambda — still Linux | `[concept]` | Phase 3.6: cgroup/namespace → ECS/EKS; Lambda |
| 12.6 | When this bridge breaks | — | Which link of the journey snaps → which symptom |

> **How to work through this phase:** This is a **synthesis** phase — you work through it not to memorize new
> commands but to connect everything you learned. The most valuable habit of this phase: whenever you see an AWS
> concept, ask yourself "**what is the Linux truth beneath this?**" An AMI? → a frozen root filesystem (Phase 6) +
> installed packages (Phase 8). If you have access to an AWS account (your own test account), you work through
> this phase best by **actually launching a t2.micro**: open an empty Ubuntu instance, SSH in and do this phase's
> seven steps by hand in order — asking yourself at each step "which phase was this." This gathers the whole
> workbook into a single practice. Even without access, you make the same synthesis by narrating the journey step
> by step on paper.

This whole phase in a single picture: the journey of an empty AMI turning into production in seven steps, and
which phase each step rests on.

![Figure 12.1 — The journey from an empty Ubuntu AMI to production. A seven-stage pipeline, each stage an AWS step and the Linux phase beneath it: (1) cloud-init / user-data (Phase 5+10), (2) user/SSH access (Phase 7), (3) EBS mount + fstab (Phase 6), (4) package install (Phase 8), (5) systemd service (Phase 5+8), (6) log/monitoring — CloudWatch (Phase 4+11), (7) hardening — SG + host firewall + IAM role + AppArmor (Phase 7+9) → production service. The lesson at the bottom: every stage is a Linux fundamental wearing an AWS name; when a stage breaks you don't debug "AWS," you drop to its Linux layer and find the evidence there (Phase 11).](../diagrams/png/lx-12-01-ami-to-production.png)

---
---

# 12.1 AMI = A Frozen Linux

## 12.1.1 What is beneath an AMI `[application]`

An **AMI** (Amazon Machine Image) is the template you use to launch an instance in AWS. But there is nothing new
beneath it: an AMI is a **frozen Linux root filesystem.** What is in it?

- A **root filesystem** (Phase 6) — the `/`, `/etc`, `/usr`, `/var` directory tree (Phase 1's FHS)
- **Installed packages** (Phase 8) — the kernel, systemd, the apt/dpkg database, pre-installed services
- A **user/permission structure** (Phase 2) — the `ubuntu` user, sudoers, SSH configuration
- A **boot configuration** (Phase 5) — GRUB, initramfs, systemd targets

When you "launch" an AMI, AWS actually copies this frozen filesystem onto an EBS disk (Phase 6) and creates a
virtual machine that boots from that disk. So an AMI is the exact AWS counterpart of Phase 8's "immutable / baked
image" idea (a frozen, reproducible system): instead of patching the server by hand, you take an image with
everything "baked" into it and produce new machines from it.

> **💡 Cloud connection — baking your own AMI:** You start with an empty Ubuntu AMI; but in production you usually
> produce **your own AMI** (with tools like Packer): launch the empty AMI, install your packages on it, do the
> hardening (Phase 9), then "freeze" that state as an AMI. The next instance comes up from this frozen state in
> **seconds** — this is exactly Phase 10's "mutable patching vs immutable rebuild" philosophy. The AMI is the
> concrete tool of the "set it up right once, then reproduce" idea.


> **🤔 Think 12.1** — Team X launches an empty Ubuntu AMI and installs its packages by hand over SSH every time. Team Y installs and hardens once, then bakes its own AMI. Autoscaling must add 10 instances within a minute, and a critical security patch has just been released. (a) Which team scales faster, and why? (b) How does each team apply the patch? (c) Which idea from the earlier phases is this?
>
> *(Answer: at the end of the phase)*
---
---

# 12.2 cloud-init / user-data and IAM Role

## 12.2.1 cloud-init: the machine that configures itself at boot `[application]`

A frozen AMI is the same on every instance — but each instance should be a little different (different hostname,
different app config, different SSH key). **cloud-init** closes this gap. As you saw in Phase 5, cloud-init is a
systemd component that runs on the system's first boot; it runs the script you give as **user-data**:

```bash
#!/bin/bash
# EC2 user-data — runs on first boot (Phase 10 — a Bash script!)
apt-get update && apt-get install -y nginx    # install a package (Phase 8)
systemctl enable --now nginx                   # add to persistent definition + start (Phase 5)
```

This is the meeting of two phases: **Phase 5** (cloud-init is part of the boot chain) + **Phase 10** (user-data
is a Bash script, so it must be idempotent and robust with `set -euo pipefail`). The "frozen image + boot-time
config" pair is the AWS way to produce reproducible yet customizable machines.

## 12.2.2 IAM instance role: authority without keeping a key on disk `[application]`

In Phase 9 you saw the danger of keeping secrets on disk: if a machine is compromised, the AWS key on disk is
compromised too (blast radius). AWS's solution is the **IAM instance role**: you don't embed a permanent key into
the machine; instead you assign a **role** to the instance, and the machine gets temporary, automatically-rotating
credentials from the metadata service.

This directly applies Phase 9's two ideas: **least privilege** (the role carries only the needed permissions —
write to S3, nothing else) and **narrowing the blast radius** (even if the machine is compromised there is no
permanent key on disk; the credentials are temporary and rotating). Phase 9's "Secrets Manager / IAM role" cloud
box was exactly this: the secret does not live on disk, the identity comes from the machine's identity.

> **🤔 Think 12.2** — An EC2 instance needs to write a file to S3. There are two ways: (a) write an AWS access key
> to disk via user-data, (b) assign an IAM role to the instance. (a) In both ways the machine can write to S3 —
> so what is the security difference, in which one is the "blast radius" (Phase 9) smaller and why? (b) When you
> use an IAM role, **where** do the credentials live, what is lost if the disk is compromised? (c) How does this
> connect to Phase 9's "keep the secret off disk" lesson?
>
> *(Answer: at the end of the phase)*

---
---

# 12.3 EBS + fstab and the systemd Service

## 12.3.1 EBS: persistent storage and the fstab flow `[application]`

An EC2 instance's root disk is ephemeral; when the instance is terminated (in some cases) the data is gone. For
persistent data you attach **EBS** (Elastic Block Store) volumes — these are the AWS counterpart of Phase 6's
"block device" concept. When you attach an EBS volume, the journey is exactly like Phase 6:

1. The EBS volume appears to the instance as a block device (`/dev/nvme1n1` — Phase 6)
2. You create a filesystem on it (`mkfs.ext4` — Phase 6)
3. You mount it to a mount point (`mount /dev/nvme1n1 /data` — Phase 6)
4. You add a persistent definition to **`/etc/fstab`** by UUID (Phase 6!) — otherwise the mount is lost on reboot

Phase 6's "to exist ≠ to be reachable" and "running mount vs persistent definition (fstab)" lessons gain
production criticality here: if you don't add it to fstab, when the instance reboots `/data` comes up empty and
the app can't find its data. The reason to use UUID was also in Phase 6: device names (`/dev/nvme1n1`) can change,
the UUID does not.

## 12.3.2 The systemd service: the application's life cycle `[application]`

Starting your app once by hand is not enough (Phase 8). In production your app must be a **systemd service** — so
it starts automatically at boot, restarts when it crashes, and its log lands in journald. This is the meeting of
Phase 5 + Phase 8:

```ini
# /etc/systemd/system/myapp.service
[Service]
ExecStart=/opt/myapp/venv/bin/python /opt/myapp/app.py
Restart=on-failure          # restart on crash (Phase 8)
User=myapp                  # not as root! (Phase 9 — least privilege)

[Install]
WantedBy=multi-user.target  # start at boot (Phase 5 — persistent definition)
```

Phase 8's "installed ≠ running as a service" and Phase 5's "running state vs persistent definition" lessons are
the heart of production: with `systemctl enable --now myapp` you both start it now (running state) and write it
into boot (persistent definition). And with `User=myapp` you apply Phase 9's "don't run as root" lesson.

> **🤔 Think 12.3** — You mounted an EBS volume to `/data` and ran your app, all is well. But you forgot to add it
> to `/etc/fstab`. The next week the instance rebooted. (a) What is the state of `/data` after the reboot, what
> does your app see? (b) This is the exact production counterpart of which of Phase 6's lessons (two ideas)? (c)
> How does running your app as a systemd service make this worse or more visible — what happens if the service
> starts at boot and sees an empty `/data`, and how would you diagnose it with your Phase 11 reflex?
>
> *(Answer: at the end of the phase)*

---
---

# 12.4 Two Defense Layers and Observability

## 12.4.1 Security Group vs host firewall / AppArmor: two separate layers `[concept]`

One of the most confused things in the cloud is that defense sits in **two separate layers**, and both rest on
Phase 7 + Phase 9:

- **Security Group (SG)** — a firewall at the AWS level, at the network layer. It filters traffic **before it
  ever reaches** the instance (Phase 7 — network access). Rules like "allow port 443 from 0.0.0.0/0." This is the
  wall **outside** the machine.
- **Host firewall (ufw/iptables) + AppArmor/SELinux** — **inside** the machine, at the OS level. The host
  firewall is Phase 7's OS-level firewall; AppArmor/SELinux is Phase 9's MAC layer ("permissions are right but
  blocked").

These two layers are the cloud form of Phase 9's **defense in depth** idea: if the SG is the outer gate, the host
firewall + MAC are the inner gates. One does not replace the other — even if an attacker gets past the SG
(misconfiguration), the inner layers still stand. Phase 9's two-lock model (DAC + MAC) becomes three locks here:
SG (network) + host firewall (OS network) + AppArmor (OS mandatory).

## 12.4.2 CloudWatch: remote observability `[concept]`

In Phase 11 we said "move the evidence outside the machine." **CloudWatch agent / Logs** is the AWS tool for this:
the machine's logs (journald, /var/log, app log — Phase 5, 11) and metrics (CPU, RAM, disk — Phase 4) flow to
AWS. So you read the evidence without entering the instance, even if the instance has crashed. This is the meeting
of Phase 4 (resource metrics) + Phase 11 (log and diagnosis). And Phase 11's lesson is critical here: you put the
CloudWatch agent into the AMI or user-data **from the start** — observability is something designed from the
beginning, not added later.

> **💡 Cloud connection — three layers, one reflex:** When someone says an instance is "unreachable," you can now
> read all three layers: (1) is the SG blocking (AWS console / `aws ec2 describe-security-groups`), (2) is the
> host firewall (`ufw status` — Phase 7), (3) is the app bound to the right address (`ss -tulpn` — Phase 7).
> Phase 7's "three lenses" become "SG + host + bind" in the cloud. A junior engineer looks only at the SG; the
> master eliminates all three. This was the whole point of this workbook: seeing beneath the abstraction.


> **🤔 Think 12.4** — Users cannot reach your web app on port 443. The security group allows 443 from 0.0.0.0/0, and a junior colleague concludes "the SG is right, so the problem is not the network." (a) Which two other layers can still block the traffic? (b) Which command inspects each layer? (c) Why can the SG not see these two?
>
> *(Answer: at the end of the phase)*
---
---

# 12.5 Container and Lambda — Still Linux

## 12.5.1 ECS/EKS container: kernel-sharing Linux `[concept]`

In Phase 3.6 you saw the anatomy of a container: a container is not a separate machine but **a process group that
shares the kernel with the host but is separated by namespaces (isolation) and cgroups (resource limits).** AWS's
**ECS/EKS** (container orchestration) is built on top of this Linux truth:

- **namespace** (Phase 3.6) → lets the container see its own filesystem, network, process tree
- **cgroup** (Phase 3.6) → enforces the container's CPU/RAM limit (Phase 4)
- Container image → is really Phase 8's "layered filesystem" idea (a small, portable form of an AMI)

So a container is not a "lightweight VM"; it is an isolated Linux process that shares the kernel. When a container
is "OOM killed" (Phase 4), when a container's permissions are wrong (Phase 2), these are all still Linux truths —
just packaged with namespaces/cgroups.

## 12.5.2 Lambda: a runtime you don't see but is still Linux `[concept]`

**Lambda** (serverless) shows you no machine at all — you just write a function, AWS runs it. But there is still a
Linux beneath it: your function runs as a Linux process, inside a micro-VM (Firecracker), with a filesystem
(`/tmp`), a memory limit (cgroup — Phase 4), a runtime (Python/Node). "Serverless" does not mean "no Linux" — it
means "**you don't manage** the Linux." Even when the abstraction rises to its highest, beneath it all the
fundamentals of this workbook (process, memory, filesystem, permission) are still there.

> **🤔 Think 12.5** — A friend of yours says "I use Lambda, I don't need to learn Linux anymore." (a) Which Linux
> truths are still running beneath Lambda (name at least three)? (b) If your Lambda function gives an "out of
> memory" error or says `/tmp` is full, which phases of this workbook's knowledge help you? (c) How do you answer
> the claim "as abstraction rises, Linux knowledge becomes unnecessary" with this phase's central idea ("AWS
> abstractions are built on top of Linux")?
>
> *(Answer: at the end of the phase)*

---
---

# 12.6 When This Bridge Breaks — Which Link of the Journey Snaps

This phase "breaking" is the snapping of a link in the boot-to-production journey. Each symptom points to a Linux
fundamental (and therefore a phase):

| Symptom (in AWS) | Snapped link | The Linux truth beneath | Phase |
|---|---|---|---|
| Instance "healthy" but app not responding | systemd service failed | Installed ≠ running; `systemctl status` | 8, 11 |
| user-data didn't run, machine half-configured | cloud-init script errored | Silent failure; `set -euo pipefail`, cloud-init log | 10 |
| `/data` empty after reboot | Not added to fstab | Running mount ≠ persistent definition | 6 |
| App can't write to S3 but SG is open | IAM role missing/wrong | Authority is in the identity layer, not the network | 9 |
| Unreachable externally but SG is 0.0.0.0/0 | App bound to 127.0.0.1 | Listening ≠ bound to the right address (three lenses) | 7 |
| Instance crashed, no log at all | CloudWatch agent not set up from the start | Move the evidence outside ahead of time | 11 |
| Container constantly restarting / OOM killed | cgroup memory limit exceeded | Container = resource-limited Linux process | 3.6, 4 |
| "Permissions are right but the app is blocked" | AppArmor/SELinux (MAC) | DAC is right, MAC is blocking | 9 |

> **The lesson from this table — and the summary of the whole workbook:** This phase's one big idea lies beneath
> every row: **beneath every AWS symptom there is a Linux truth.** "Instance healthy but no app" is not an AWS
> mystery but a systemd service being failed (Phase 8). "Unreachable but SG open" is not an AWS bug but the app
> being bound to 127.0.0.1 (Phase 7). This is exactly what separates a cloud engineer from a junior: instead of
> getting lost in AWS's abstraction, descending to the Linux layer beneath it and finding the evidence (Phase 11
> reflex). Every stone we've laid since this workbook's first phase converges in this final phase into a single
> ability: **seeing beneath the abstraction.**

---
---

# Phase 12 — Answers to the think questions

## Answer 12.1 — Baked image: the new machine starts frozen and ready, patches go through the recipe

(a) Team Y: its instances come up from the frozen state in seconds, while X must wait for installs on every machine and risks a different result each time (drift, human error). (b) X patches machines one by one over SSH (mutable — the fleet drifts apart). Y updates the recipe, bakes a new AMI and replaces the old instances (immutable — every machine is identical). (c) Phase 8/10's "mutable patching vs immutable rebuild": set it up right once, then reproduce.
**Related section:** 12.1.1 · **Continues in:** 12.2.1 (cloud-init user-data)

## Answer 12.2 — IAM role vs a key on disk; blast radius; keep the secret off disk

(a) In both ways the machine can write to S3, but the security difference is large: an AWS key written to disk is
**permanent** and **static** — if the machine is compromised (Phase 9), the attacker reads this key and with it
accesses S3 from anywhere, as much as they want; if the key leaks you have to revoke and rotate it by hand. With
an IAM role the "blast radius" is much smaller because the credentials are **temporary** (short-lived) and
**automatically rotating**. (b) When you use an IAM role the credentials **never live on disk** — the machine gets
them on demand from the instance metadata service and holds them in memory; even if the disk is compromised there
is no permanent key, at most a temporary token that expires in minutes. (c) This is the direct AWS application of
Phase 9's "keep the secret off disk" lesson: the secret (a long-lived key) is never written to disk, the identity
is derived from the machine's **own identity** (the IAM role) — so least privilege + small blast radius are
achieved together.
**Related section:** 12.2.2 · **Connects to:** Phase 9 (secrets, blast radius).

## Answer 12.3 — A mount not added to fstab; running vs persistent; more visible with systemd

(a) After the reboot `/data` is **empty** — because the mount was only "running state," it wasn't written to the
persistent definition (fstab); the EBS volume is still there but not mounted to `/data`. When your app looks at
`/data` it sees either an empty directory or (the old root-disk content under the mount point) — it can't find its
data. (b) This is the exact production counterpart of Phase 6's **two lessons**: "to exist ≠ to be reachable" (the
volume exists but isn't reachable) and "running mount ≠ persistent definition (fstab)" (the mount command was
temporary, fstab would have been permanent). (c) A systemd service makes this both worse and more visible: the
service starts automatically at boot (Phase 5 — persistent definition) and on seeing an empty `/data` either
crashes (no data) or starts writing to the empty directory (worse — writes the data to the wrong place). But
visibility also increases: with the Phase 11 reflex `journalctl -u myapp` shows the app's "file not found" error,
and `df -h`/`mount` proves that `/data` isn't mounted — the evidence chain leads you to the missing fstab entry.
**Related section:** 12.3.1-12.3.2 · **Connects to:** Phase 6, Phase 5, Phase 11.

## Answer 12.4 — SG + host firewall + bind address: eliminate all three layers

(a) The host firewall (ufw/iptables inside the machine) and the application's bind address (a service listening only on `127.0.0.1` is unreachable from outside no matter what the SG says); AppArmor/SELinux is a further inner lock for what the process may access. (b) SG: `aws ec2 describe-security-groups` (or the console); host firewall: `sudo ufw status`; bind: `ss -tulpn`. (c) The SG filters traffic **before it ever reaches** the instance; what happens inside the OS — the host firewall's rules and which address the process listens on — is outside its view. The junior looks only at the SG; the master eliminates all three.
**Related section:** 12.4.1 · **Continues in:** 11.4.2 ("Why is it unreachable?")

## Answer 12.5 — The Linux beneath Lambda; OOM/tmp; as abstraction rises

(a) The Linux truths still running beneath Lambda (at least three): the function runs as a **Linux process**
(Phase 3); it has a **memory limit** enforced by a cgroup (Phase 4 + 3.6); it has a **filesystem** (`/tmp`, a root
filesystem — Phase 6); it has a runtime and its process model (Phase 3), a permission structure (Phase 2). (b) An
"out of memory" error → Phase 4 (memory, the OOM killer) and Phase 3.6 (cgroup limit) knowledge; `/tmp` full →
Phase 6 (filesystem, disk fullness) knowledge helps — because these Lambda limits are the Linux mechanisms
themselves. (c) The answer to "as abstraction rises Linux becomes unnecessary": the abstraction **hides Linux but
does not destroy it.** In Lambda you don't manage the Linux, but when a problem (OOM, timeout, /tmp full,
permission) arises you can only diagnose it by knowing the Linux truth beneath. This phase's central idea is
exactly this: AWS abstractions are built on top of Linux — no matter how high the abstraction rises, the floor you
can descend to when something breaks is still Linux. That is why Linux knowledge **does not become unnecessary
with abstraction; on the contrary, it becomes the distinguishing skill.**
**Related section:** 12.5.1-12.5.2 · **Connects to:** Phase 3.6, Phase 4, the whole workbook.

---
---

# Phase 12 — Frequently asked questions

**Q1 — What exactly is an AMI?** A frozen Linux root filesystem: installed packages (Phase 8), the directory tree
(Phase 1, 6), the user/permission structure (Phase 2), the boot configuration (Phase 5). When you launch an
instance it is copied onto EBS and boots (12.1).

**Q2 — Are user-data and cloud-init the same thing?** cloud-init is the systemd component that runs at boot
(Phase 5); user-data is the script you give it (Phase 10 — a Bash script). cloud-init runs the user-data on first
boot (12.2.1).

**Q3 — Why an IAM role instead of writing an AWS key to disk?** A key on disk is permanent and static; if the
machine is compromised it leaks (large blast radius). An IAM role gives temporary, rotating credentials; even if
the disk is compromised there is no permanent secret (Phase 9) (12.2.2).

**Q4 — Why must I add an EBS mount to fstab?** The `mount` command is only "running state"; it is lost on reboot.
fstab is the persistent definition — otherwise `/data` comes up empty after a reboot (Phase 6). Use UUID, the
device name can change (12.3.1).

**Q5 — The difference between a Security Group and a host firewall?** The SG is **outside** the machine, at the
AWS network layer (Phase 7); the host firewall + AppArmor are **inside** the machine, at the OS layer (Phase 7 +
9). They are two separate gates of defense in depth, one does not replace the other (12.4.1).

**Q6 — Is a container a real machine?** No — it is a Linux process group that shares the kernel with the host,
separated by namespaces (isolation) and cgroups (resource limits) (Phase 3.6). Not a "lightweight VM" but an
isolated process (12.5.1).

**Q7 — Do I need to learn Linux if I use Lambda?** Yes. Lambda hides Linux but beneath it there is still a Linux
process, a memory limit (cgroup), a filesystem (/tmp). When a problem arises you can only diagnose it with Linux
fundamentals (12.5.2).

---
---

# Phase 12 — Test yourself

This final exam is a synthesis exam: most questions test the "which AWS concept rests on which Linux fundamental"
connection. Write your answers on paper. Target: 14+ out of 18.

## Section A — Concept and counterpart

1. Which four Linux pieces are beneath an AMI (with their phases)?
2. The relationship between cloud-init and user-data and which two phases it rests on?
3. Why is an IAM instance role safer than writing a key to disk (Phase 9, two ideas)?
4. The four steps of the EBS mount journey and the role of fstab (Phase 6)?
5. In a systemd service unit, which phase does each of `Restart=on-failure`, `User=myapp`,
   `WantedBy=multi-user.target` connect to?
6. The layer difference between a Security Group and a host firewall/AppArmor?
7. The two Linux mechanisms that separate a container from a "lightweight VM" (Phase 3.6)?
8. Three Linux truths still running beneath Lambda?

## Section B — Apply and diagnose

9. Instance "healthy" but the app doesn't respond. Which link snapped, with which command do you confirm?
10. user-data didn't run, the machine is half-configured. The likely cause and which Phase 10 habit would have
    prevented it?
11. `/data` is empty after reboot. What is the snapped link, the fix, which Phase 6 lesson?
12. The app can't write to S3 but the SG is open. In which layer is the problem, what is the fix?
13. Unreachable externally but the SG is 0.0.0.0/0. Which Linux truth, with which command do you prove it?
14. The instance crashed and there is no log. What should you have done, which Phase 11 lesson is this?

## Section C — Reasoning and synthesis

15. Explain the idea "AWS abstractions are built on top of Linux" with three examples (AMI, IAM role, container).
16. What does "seeing beneath the abstraction" — what separates a cloud engineer from a junior — mean? Explain
    with the "unreachable but SG open" example.
17. Name the seven steps of the journey from an empty AMI to production in order and connect each to a phase.
18. Write a one-sentence main idea from this workbook's first phase (mental model) to this final phase (cloud
    bridge): why is learning Linux fundamental for the cloud?

---

## Answer key

1. Root filesystem (Phase 6), installed packages (Phase 8), user/permission structure (Phase 2), boot
   configuration (Phase 5) (12.1.1). — 2. cloud-init is the systemd component at boot (Phase 5), user-data is the
   Bash script given to it (Phase 10); cloud-init runs user-data on first boot (12.2.1). — 3. Least privilege (the
   role carries only the needed permission) + small blast radius (no permanent key on disk, credentials
   temporary/rotating) (12.2.2). — 4. Block device appears → mkfs → mount → persistent definition in fstab by
   UUID; without fstab the mount is lost on reboot (12.3.1). — 5. `Restart=on-failure` → Phase 8 (restart on
   crash), `User=myapp` → Phase 9 (not root, least privilege), `WantedBy=multi-user.target` → Phase 5 (start at
   boot, persistent definition) (12.3.2). — 6. The SG is outside the machine at the AWS network layer (Phase 7),
   the host firewall/AppArmor inside the machine at the OS layer (Phase 7+9); two gates of defense in depth
   (12.4.1). — 7. namespace (isolation) + cgroup (resource limit); a container is a kernel-sharing isolated
   process (12.5.1). — 8. Linux process (Phase 3), memory limit/cgroup (Phase 4+3.6), filesystem /tmp (Phase 6) —
   and runtime/permission (Phase 2, 3) (12.5.2).

9. systemd service failed; `systemctl status myapp` + `journalctl -u myapp` (status check ≠ app) (12.6, Phase
   8+11). — 10. The cloud-init script silently errored and continued; `set -euo pipefail` (Phase 10) would have
   prevented it; read the cloud-init log (`/var/log/cloud-init-output.log`) (12.6). — 11. The mount wasn't added
   to fstab; add it to `/etc/fstab` by UUID; "running mount ≠ persistent definition" (Phase 6) (12.3.1, Answer
   12.2). — 12. The authority is in the **identity** layer, not the network; the IAM role is missing/wrong —
   assign a role with the right permissions (Phase 9) (12.6). — 13. The app is bound to 127.0.0.1; see the bind
   address with `ss -tulpn | grep <port>` (listening ≠ the right address, Phase 7 three lenses) (12.6). — 14. You
   should have put the CloudWatch agent into the AMI/user-data from the start; "move the evidence outside ahead of
   time" (Phase 11) (12.4.2). — 15. AMI = a frozen Linux filesystem; IAM role = the AWS abstraction of Linux's
   secret/identity management; container = a Linux process packaged with namespaces/cgroups — all have a Linux
   mechanism beneath (12, whole phase). — 16. Seeing beneath the abstraction: reducing the AWS symptom to a Linux
   truth. "Unreachable but SG open" → a junior looks only at the SG; a master checks the app's bind address
   (127.0.0.1) with `ss -tulpn` (Phase 7) (12.4.2, 12.6). — 17. cloud-init/user-data (Phase 5+10) → user/SSH
   access (Phase 7) → EBS mount + fstab (Phase 6) → package install (Phase 8) → systemd service (Phase 5+8) →
   log/monitoring/CloudWatch (Phase 4+11) → hardening: SG + host firewall + IAM role + MAC (Phase 7+9) (12, "Final
   output"). — 18. Example: "The cloud is nothing but abstractions built on top of Linux; that is why a cloud
   engineer must master the one solid floor they can descend to when the abstraction breaks — Linux fundamentals"
   (the whole workbook).

## Scoring

| Correct | What it means |
|---|---|
| 16-18 | Congratulations — you have synthesized the whole workbook. You can see the Linux beneath every AWS concept. You are ready for the cloud journey. |
| 13-15 | Very good. Return to the phases of the connections you missed (especially 12.3 and the 12.6 table). |
| 9-12 | The fundamentals are there but the bridges are weak. Study the phase map and the 12.6 table again — connect each row to a phase. |
| 0-8 | This is a synthesis phase; first refresh the relevant fundamental phases (6, 7, 8, 9, 11), then return here. |

Missed question → section to return to:

| Question | Section |
|---|---|
| 1, 15 | 12.1 AMI |
| 2, 10 | 12.2.1 cloud-init |
| 3, 12 | 12.2.2 IAM role |
| 4, 11 | 12.3.1 EBS + fstab |
| 5, 9 | 12.3.2 systemd service |
| 6, 13 | 12.4.1 Two defense layers |
| 14 | 12.4.2 CloudWatch |
| 7, 8 | 12.5 Container and Lambda |
| 16, 17, 18 | 12.6 + Final output |

---
---

# Phase 12 — Closing: The End of the Workbook and the Start of the Cloud Journey

## What you carry from this phase — and from the whole workbook

Phase 12 united every stone you've laid since Phase 0 into a single arch: the journey of an empty Ubuntu AMI from
boot to production. You can now see the Linux truth beneath every AWS concept — an AMI is a frozen Linux, cloud-init
is a boot script, the IAM role is Phase 9's secret lesson, EBS+fstab is Phase 6's storage flow, the systemd
service is Phase 5+8's life cycle, a container is Phase 3.6's namespace/cgroup, Lambda is a Linux you don't see.
And most importantly: when something breaks (instance healthy but no app, unreachable but SG open, `/data` empty
after reboot) you don't panic — you descend to the Linux layer beneath that symptom and find the evidence (Phase
11 reflex).

From this workbook's first phase to its last, a single idea has been carried: **learning Linux is not a
prerequisite for the cloud, it is the cloud itself.** AWS, GCP, Azure — all are abstractions built on top of Linux.
No matter how high the abstraction rises, the last solid floor you can descend to when something breaks is always
Linux. That is why what separates a cloud engineer from a junior is not knowing the newest AWS service, but being
able to **read the Linux truth beneath every abstraction.** You have now gained this ability.

## Where to go from here

The end of this workbook is the start of the real cloud journey. Two bridges carry you forward:

> **🤔 Final output — ask yourself:** How would you apply the "seeing beneath the abstraction" reflex you've
> learned throughout this workbook to an AWS service you've never seen (e.g. RDS, ECS, EKS)? When you see a new
> service, what should your first questions be: "which Linux mechanism is beneath this, and which layer do I
> descend to when something breaks?" This single question is the most lasting habit this workbook has given you.
>
> **🧪 Capstone Lab (on your own AWS test account) — the practice of the whole workbook:** Launch an empty Ubuntu
> t2.micro instance and do this phase's journey **from start to finish by hand**: (1) install nginx at boot via
> user-data (Phase 5+10). (2) SSH in, inspect the `ubuntu` user and permissions (Phase 2). (3) Attach an EBS
> volume, format it, mount it, **add it to fstab by UUID** (Phase 6). (4) Write a simple app as a systemd service
> with `User=` non-root and enable it (Phase 5+8+9). (5) Assign an IAM role, access S3 without writing any key to
> disk (Phase 9). (6) Check the bind address with `ss -tulpn`, the host firewall with `ufw`, the SG — three
> defense layers (Phase 7+9). (7) Install the CloudWatch agent, stream the logs out (Phase 4+11). Now **set up a
> deliberate fault** (bind the app to 127.0.0.1) and diagnose it with the Phase 11 reflex. This single lab brings
> the thirteen phases of this workbook alive on one real machine. (8) When done, **terminate** the instance so no
> cost accrues — this too is Phase 10's "immutable, disposable infrastructure" lesson.

You have finished this workbook. You now know **why** every command is there, **what** lies beneath every
abstraction, and **which layer** to descend to when something breaks. Your cloud journey starts here.

---

> **Appendices:** [Appendix A — Command Glossary](Appendix_A_Command_Glossary.md) · [Appendix B — File and Directory Map](Appendix_B_File_Directory_Map.md) · [Appendix C — "When It Breaks" Quick Reference](Appendix_C_When_It_Breaks_Quick_Reference.md)

> **Navigation:** [◀ Phase 11 — Observability and Troubleshooting](Phase_11_Observability_and_Troubleshooting.md) · **Phase 12** · [Appendix A — Command Glossary ▶](Appendix_A_Command_Glossary.md)
