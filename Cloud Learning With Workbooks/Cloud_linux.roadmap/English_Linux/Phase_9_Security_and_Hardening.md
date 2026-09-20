# Phase 9 — Security and Hardening

> **Navigation:** [◀ Phase 8 — Packages and Service-ization](Phase_8_Packages_and_Service_ization.md) · **Phase 9** · [Checkpoint Quiz 4 ▶](Checkpoint_Quiz_4.md)

---

## Where we are coming from

At the end of Phase 8 we pointed out a trap you had already set for yourself: you wrote `User=appuser` in the
unit file — to run with a limited user instead of root. That was really a **security decision**, and you made
it back then. Phase 9 turns that instinct into a system. Until now we always did **more**: start a process
(Phase 3), mount a disk (Phase 6), open the service to the network (Phase 7), install a package (Phase 8).
Phase 9 asks the opposite for the first time: **what should I restrict?**

Knowledge from four phases turns into a defense here:

- **Phase 2 — users and permissions.** You saw the DAC (Discretionary Access Control — owner/group/other,
  `rwx`) permission model. Phase 9 puts a second lock (MAC) **on top of** it and systematizes the "least
  privilege" principle.
- **Phase 7 — the network surface.** SSH hardening, `0.0.0.0` vs `127.0.0.1`, host firewall + Security Group.
  Phase 9 gathers these into a "surface-reduction" discipline.
- **Phase 8 — who runs the service.** The `User=` line determines the identity a service runs as. Phase 9
  deepens this with the question "how much damage can it do if it is compromised?"

## The question of this phase

Phase 8 said "how do I install and run software." Phase 9 asks exactly the opposite:

> *"Every package I install, every port I open, every privilege I grant is an attack surface. How do I
> deliberately shrink that surface — and if a service is compromised, how do I limit the damage?"*

Security is not a product, it is a **discipline**, and there is no single magic setting. Instead there is a
**layered defense** (defense in depth): the network layer (firewall + SG), the access layer (SSH key-only,
root off), the privilege layer (least privilege, non-root, capabilities), mandatory access control
(AppArmor/SELinux), and secrets management (not on disk, but in IAM/Secrets Manager). Even if one layer is
breached, the others still stand. Two ideas sit at the center of this phase: **least privilege** (give
everything only as much as it needs) and **shrinking the blast radius** (if something is compromised, how far
can the damage spread).

By the end of this phase you will be able to deliberately harden a system; connect a confusing failure like
"the permissions are correct but it is still blocked" to the MAC layer (AppArmor/SELinux); and explain why
managing secrets with cloud identity mechanisms (IAM role), never embedding them on disk/in code, is
fundamental.

---

## By the end of this phase

- You will be able to explain the **principle of least privilege** and make the "only the privilege needed"
  decision for each service/user
- You will be able to systematically harden the SSH and network surface: key-only, root off, unnecessary
  ports closed, firewall (a continuation of Phase 7)
- You will be able to explain MAC (Mandatory Access Control) — AppArmor (Ubuntu) and SELinux (RHEL) — as a
  **second lock on top of** DAC permissions; and connect the "permissions correct but still blocked" failure
  to it
- You will know audit and integrity tools (auditd, rkhunter, file integrity) at a conceptual level; and be
  able to explain the **real** scenario where an auditd log explosion can choke journald and freeze the
  system, and the need for rate-limiting/rotation
- You will recognize **capabilities** as fine-grained privileges that break the root/non-root duality (e.g.
  port-bind only)
- You will be able to explain why embedding secrets on disk/in code is wrong; and how **Cloud:** IAM instance
  role (authority without keeping a key on disk), SSM Parameter Store / Secrets Manager and a CIS-benchmark
  hardened AMI solve it

---

## The phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 9.1 | Principle of least privilege | `[concept]` | The backbone of the whole phase: only the privilege needed |
| 9.2 | SSH and network surface hardening | `[application]` | Phase 7 turning into a defense discipline |
| 9.3 | MAC — AppArmor / SELinux | `[mechanism]` | A second lock over DAC; "permissions correct but blocked" |
| 9.4 | Audit and integrity | `[concept]` | auditd/rkhunter; the log-explosion trap |
| 9.5 | Capabilities | `[concept]` | Fine-grained privileges that break the root/non-root duality |
| 9.6 | Secrets management | `[application]` | **Embedding keys on disk** — solve it with an IAM role |
| 9.7 | When this phase breaks | — | Security/hardening failure signatures |

> **How to work through this phase:** The observation commands in this phase are 🟢 (`aa-status`, `ss -tulpn`,
> `systemctl status auditd`, `getcap`). But the hardening steps are 🔴 by their nature — a wrong SSH rule, a
> wrong AppArmor profile, or a wrong firewall rule can **lock you out** or stop a service. Phase 7's golden
> rule applies here too: test critical changes while a **second open session** is running, and make every
> step have an undo path. The most instructive experiment: put an AppArmor profile into `complain` mode and
> watch what it blocks in `journalctl` — see the "permissions correct but blocked" failure live. Work on your
> own test machine.

---
---

# 9.1 The Principle of Least Privilege

## 9.1.1 "Give only what is needed": the backbone of defense `[concept]`

The one-sentence essence of the whole security discipline: **every user, every service, every process should
have only the least privilege needed to do its job — not one more.** This is called the **principle of least
privilege**, and everything in this phase is really an application of it.

Why so central? Because security is not "never being compromised" — sooner or later a vulnerability is found.
The real question is: **if something is compromised, how far does the damage spread?** This is called the
**blast radius**. If a service runs as root and is compromised, the attacker takes the whole machine. If the
same service runs as a limited user (the `User=appuser` from Phase 8!), the attacker sees only as much as
that user can — the blast radius is small. Least privilege does not prevent compromise; it **limits the
damage** when compromise happens.

> **⚠️ Common misconception: "Running everything as root is more practical, I won't deal with permission
> headaches."**
>
> On the contrary — that maximizes the blast radius. A single vulnerability in a service running as root =
> the whole machine falling. Writing `User=appuser` in Phase 8 was not "compromising on convenience," it was
> a deliberate security investment: even if that service is compromised, the attacker cannot become root,
> cannot read other users' files, cannot modify system files. "Dealing with permissions" is not a headache of
> security, it *is* security. The rule: before giving a service root, always ask — "does it really need root,
> or just one single privilege?" (the answer to that question is usually capabilities in 9.5).

## 9.1.2 Defense in depth: not one lock, many locks `[concept]`

Least privilege alone is not enough; you place it inside a **layered defense** (defense in depth). The idea:
for an attacker to reach the target (your app and its data) they must defeat **multiple independent layers**,
and even if one layer is breached, the others still stand. This phase's sections are in fact those layers —
from outside in:

![Figure 9.1 — Defense in depth: for an attacker to reach the application and its secrets they must defeat independent layers in turn. From outside in: ① Network (Security Group + host firewall, Phase 7) → ② Access (SSH key-only, root off, 9.2) → ③ Least privilege (non-root user, capabilities, 9.1/9.5) → ④ MAC (AppArmor/SELinux, a second lock over permissions, 9.3) → ⑤ the application + secrets at the core (not embedded on disk, accessed via IAM role, 9.6). Each layer is independent; breaching one is not a full compromise — least privilege shrinks the blast radius.](../diagrams/png/lx-9-01-defense-in-depth.png)

This model is the map of the whole phase. In Phase 7 you built the first two layers (network + access); Phase
9 adds the inner layers (privilege, MAC, secrets). The critical point: the layers are **independent** — even
if one is misconfigured (an instance lands in the wrong SG, one privilege is over-granted) the others can
still protect. Saying "I have one security wall" means a single-locked door; defense in depth is five locks in
a row.

> **🤔 Think 9.1** — You have two servers. On A the web application runs with `User=root`, its only defense a
> host firewall. On B the same application runs with `User=appuser`, and it also has SG + host firewall +
> AppArmor profile. The application code has a vulnerability allowing remote code execution (the same
> vulnerability on both). (a) Can the attacker exploit this vulnerability on both servers? (b) **After**
> compromise, why is the blast radius different on the two servers — which layers limit the attacker on B?
>
> *(Answer: at the end of the phase)*

---
---

# 9.2 SSH and Network Surface Hardening

## 9.2.1 Surface reduction: Phase 7 turning into a discipline `[application]`

In Phase 7 you learned **how** to run SSH and the network; Phase 9 turns that same knowledge into a
**reduction discipline**. The core question: "how much surface on the machine faces outward, and how much of
it is really needed?" Every open port, every active service, every allowed login method is an attack surface.
Hardening is deliberately shrinking that surface. On the network side, four basic steps (most familiar from
Phase 7):

1. **SSH key-only + root off** (Phase 7.3.4): `PasswordAuthentication no`, `PermitRootLogin no`. Makes
   brute-force meaningless, cuts direct root login.
2. **Turn off unnecessary services.** Audit the listening ports with `ss -tulpn`; stop and `disable` every
   service you do not need (Phase 5/8). A port that is not listening is a port that cannot be attacked.
3. **Make the firewall default-deny.** On the host firewall (ufw) and the Security Group the default policy
   should be "close everything, open only what is needed" — a whitelist, not a blacklist.
4. **Open ports to the narrowest source.** Open SSH not to `0.0.0.0/0` (the whole internet) but only to your
   own office/VPN IP; or do not open it at all (SSM — Phase 7.3.4).

> **🔧 See it on your machine** 🟢 — audit the outward-facing surface
>
> ```
> $ sudo ss -tulpn                 # which ports listen outward (0.0.0.0)?
> $ systemctl list-units --type=service --state=running   # how many services run
> $ sudo ufw status verbose        # host firewall policy (is it default deny?)
> ```
>
> Every `0.0.0.0:<port>` line in the first output is a question: "should this really be reachable from
> outside?" If the answer is "no," either bind the service to `127.0.0.1` (Phase 7.4.2) or close it in the
> firewall. This is surface reduction in its most concrete form.

## 9.2.2 Automatic security updates `[concept]`

One dimension of the surface is **time**: a package that is safe today can become vulnerable tomorrow because
of a flaw (a CVE). Mechanisms like `unattended-upgrades` apply security patches automatically. But this brings
back Phase 8's tension: an automatic upgrade can conflict with the reproducibility of version pinning (8.1.2).
The mature practice: enable automatic updates for **security patches**, but keep **application versions**
fixed with an immutable image (8.4). They are different layers.

---
---

# 9.3 MAC — Mandatory Access Control

## 9.3.1 A second lock on top of DAC `[mechanism]`

The permission model you learned in Phase 2 (`rwx`, owner/group/other) is **DAC** — Discretionary Access
Control. "Discretionary" because the owner of a file sets and can change the permissions **themselves**. This
is powerful but has a weakness: if a process runs as root (or a user is tricked), DAC cannot stop it — root
can do anything.

**MAC** (Mandatory Access Control) puts a **second, independent lock** on top of this. "Mandatory" because
the rules are set by a central policy and the process itself (even root) cannot loosen them. Two big
implementations: **AppArmor** (Ubuntu — path-based profiles) and **SELinux** (RHEL/Amazon Linux — label-based,
more granular). Both share the same idea: you assign each program a **profile** that says "you can access only
these files, these network operations, these capabilities." What the profile does not allow, the program
**cannot do**, even if the DAC permissions are wide open.

## 9.3.2 "Permissions are correct but it is still blocked" `[mechanism]`

This is MAC's most classic and most confusing failure — because your Phase 2 reflex ("check the permissions")
misleads you. Symptom: a service tries to access a file/port, the `ls -l` permissions are **flawless**
(`chmod`/`chown` correct), but access is still denied with "Permission denied." There is no problem at the DAC
layer — because the obstacle is not in DAC, it is in the **MAC layer above it**. The AppArmor/SELinux profile
does not allow that process to access that file/operation.

> **🔧 See it on your machine** 🟢 — audit the MAC layer
>
> ```
> $ sudo aa-status                 # AppArmor: which profiles are loaded, which enforcing
> $ sudo journalctl -k | grep -i apparmor | tail    # blocked operations (DENIED)
> # SELinux side (RHEL):
> $ getenforce                     # Enforcing / Permissive / Disabled
> $ sudo ausearch -m avc -ts recent   # accesses SELinux blocked (AVC denials)
> ```
>
> The diagnostic reflex: if the DAC permissions look correct but access is blocked, **look one layer up**.
> `journalctl -k | grep apparmor` (or `ausearch -m avc` on SELinux) tells you exactly what was blocked. The
> fix is to correct the profile (add the file to the allowed path) — not to turn the profile off entirely
> (`disable`), because then you lose the second lock.

> **💡 Cloud connection — hardened AMI and CIS benchmark:** In the cloud you do not write security profiles
> from scratch; you use **hardened AMIs** pre-hardened against standards like the **CIS benchmark**. These
> images come with sensible AppArmor/SELinux profiles, unnecessary services turned off, hardened SSH settings,
> and audit configuration. Combined with Phase 8's immutable philosophy: instead of setting up security by
> hand on every machine, you put the hardened image in the recipe once and every machine is born secure.
> "Security is not an installation step, it is a property of the image."

> **🤔 Think 9.2** — A web server you just set up tries to read a file from a directory under `/var/www/data/`
> but gets "Permission denied." `ls -ld /var/www/data` shows `drwxr-xr-x` and the file owner is correct — no
> problem from a DAC perspective. Even `sudo -u www-data cat /var/www/data/x.txt` works. (a) Which layer is
> most likely the obstacle? (b) With which single command would you confirm it, and (c) why is the correct
> fix not "turn off the profile"?
>
> *(Answer: at the end of the phase)*

---
---

# 9.4 Audit and Integrity

## 9.4.1 What happened, and did something change: auditd, rkhunter `[concept]`

The second half of security is **visibility**: being able to see what happened when (or after) something
happens. Three tool categories:

- **auditd** — a kernel-level **audit log**. It records events like "who, when, accessed which file / made
  which system call." Critical for post-incident forensics and compliance.
- **rkhunter / chkrootkit** — **rootkit scanners**. They look for traces of malware hidden on the system.
- **File integrity** (like AIDE) — takes a "fingerprint" (hash) of critical files and checks later whether
  they changed. The answer to "did someone modify `/etc/passwd`?"

## 9.4.2 The audit itself as a source of failure: log explosion `[application]`

Here is a real, lived trap: the audit tool **itself** can bring down the system. Scenario: auditd is
configured with a very broad rule set (like log every file access), the system runs heavily, and auditd starts
producing thousands of events per second. These logs flow to journald; journald chokes, the disk fills fast
(the `df` filling from Phase 6!), and the I/O bottleneck slows the whole system — eventually the machine
either freezes or the services die because the disk is full. The tool you set up for security turns into a
**denial-of-service** source.

The lesson is two-fold: (1) keep the audit rules **narrow** — log the events that truly matter, not
everything; (2) **rate-limiting and rotation** are a must — limit auditd's event rate, rotate the
journald/log files (Phase 5/6). This shows that even security tools are subject to the "least privilege /
least load" principle.

> **🔧 See it on your machine** 🟢 — check the audit service and log load
>
> ```
> $ sudo systemctl status auditd            # is the audit service up
> $ sudo journalctl --disk-usage            # how much disk journald eats
> $ df -h /var/log                          # is the log disk filling up (Phase 6!)
> ```
>
> `journalctl --disk-usage` and `df -h /var/log` together answer "are the logs filling the disk" — this is the
> early-warning indicator of the log-explosion trap in 9.4.2.

---
---

# 9.5 Capabilities

## 9.5.1 Breaking the root/non-root duality `[concept]`

Until now the world seemed split in two: either you are root (you can do anything) or you are not (you are
restricted). But real needs are usually somewhere in between. The classic example: a web server wants to
listen on **port 80**; opening ports below 1024 traditionally requires root privilege. So people run the whole
server as root — for a single privilege! This violates the principle of least privilege (9.1).

**Linux capabilities** break this duality: they split root's entire power into ~40 separate **fine-grained
privileges** you can grant one by one. In the web-server example, instead of full root you grant only the
`CAP_NET_BIND_SERVICE` privilege — "you can bind only to privileged ports, no other root power." So the
process opens port 80 but does **not** have root's remaining dangerous powers (reading any file, loading
kernel modules, killing other processes). The blast radius shrinks dramatically.

> **🔧 See it on your machine** 🟢 — see a binary's capabilities
>
> ```
> $ getcap /usr/bin/ping
> /usr/bin/ping cap_net_raw=ep       # ping is not root but can open a raw socket
> $ sudo setcap 'cap_net_bind_service=+ep' /opt/myapp/server   # 🔴 grant a fine-grained privilege
> ```
>
> `ping` itself is a nice example: it used to be setuid-root (dangerous); on modern systems it runs with only
> the `cap_net_raw` capability — the one privilege it needs. In a systemd unit you can also grant this with
> `AmbientCapabilities=CAP_NET_BIND_SERVICE` (a Phase 8 connection).

---
---

# 9.6 Secrets Management

## 9.6.1 What NOT to do: secrets embedded in code/on disk `[application]`

An application almost always needs **secrets**: a database password, an API key, a TLS private key. The most
dangerous mistake, which every beginner makes, is to **embed these secrets in code or on disk**:

```python
# NEVER DO THIS:
DB_PASSWORD = "s3cr3t-prod-password"      # inside the code
API_KEY = "AKIA...."                       # in a config file, plaintext
```

Why is it a disaster? Because that secret now: enters git history (even if you delete it, it stays in the
past), exists in every copy of the code, can leak into logs, and is read by anyone who reaches the machine.
A once-leaked secret is a leaked secret — there is no cure but to rotate it. Phase 2's permission lesson does
not suffice here: `chmod 600` protects a config file but the secret is still **plaintext on disk** and root
(or whoever takes over) reads it.

## 9.6.2 The right way: secrets not on disk, but in identity `[application]`

The correct model is to **remove** the secret from the machine entirely. Two approaches:

- **A secrets management service.** Keep secrets in a central, encrypted vault (AWS Secrets Manager, SSM
  Parameter Store, HashiCorp Vault). The application pulls the secret at runtime; the disk never holds a
  plaintext secret. When the secret is rotated, it rotates in one place.
- **Identity-based authorization (IAM role).** The strongest way — it eliminates the secret entirely. Instead
  of giving the application a password, you give the machine it runs on an **identity** (IAM instance role).
  The application says "I am this machine" and the cloud gives it temporary, auto-rotating credentials. No key
  is written to disk.

> **💡 Cloud connection — IAM instance role: zero secrets on disk:** When you attach an **IAM role** to an EC2
> instance, the application on top no longer has to store any key to access S3/RDS/Secrets Manager. The
> instance metadata service gives it temporary, short-lived, auto-rotating credentials. This is the
> data-access version of the Phase 7.3 idea "never distribute the SSH key, use SSM": **authenticate an
> identity instead of distributing a key.** The result: no persistent secret on disk → no secret to steal →
> the whole failure class of secret leakage largely disappears. "A secret that does not exist cannot leak" —
> this is the single strongest lesson of the phase.

> **🤔 Think 9.3** — Your application will read a file from S3. Two designs: (A) write an IAM user's access key
> into the application's config file; (B) attach an IAM role to the instance and let the application access
> keyless. (a) In (A), if the access key leaks (e.g. the config lands in git), what happens, and what must you
> do to limit the damage? (b) Why does (B) largely eliminate this risk class — what gets written to disk?
>
> *(Answer: at the end of the phase)*

---
---

# 9.7 When This Phase Breaks — Security/Hardening Failure Signatures

This phase's failures are special: most are of the form "something is **too** open" (a security hole) or
"something is **too** closed" (hardening broke a function). The table takes the symptom to the right layer:

| Symptom | Likely cause | Where to look | Related section |
|---|---|---|---|
| Permissions correct but "Permission denied" | MAC (AppArmor/SELinux) is blocking | `aa-status`, `journalctl -k \| grep apparmor`, `ausearch -m avc` | 9.3.2 |
| Service cannot bind to 80, don't want to give root | Missing privileged-port privilege | `setcap cap_net_bind_service` / `AmbientCapabilities` | 9.5.1 |
| System slowed/froze, disk filling fast | auditd log explosion → journald/disk | `journalctl --disk-usage`, `df -h /var/log`, audit rules | 9.4.2 |
| Service compromised, whole machine fell | Service ran as root (blast radius max) | `User=` in the unit, capabilities | 9.1.1 |
| A secret leaked (landed in git / in a log) | Secret embedded on disk/in code | Rotate the secret, move to Secrets Manager/IAM role | 9.6.1 |
| Locked out after SSH hardening | Wrong `sshd_config` / firewall | Get in via serial/cloud console, undo (Phase 7) | 9.2.1 |
| Automatic upgrade broke an application | A security patch changed the version | Pinning (8.1.2), immutable image (8.4) | 9.2.2 |

> **The lesson from this table:** The two big ideas of this phase lie beneath every row. The first is **layer
> separation**: the "permissions correct but blocked" failure comes from confusing DAC (Phase 2) with MAC
> (9.3) — when you hit a block, always ask "which layer is this: permission, MAC, firewall, or capability?"
> The second is **blast radius**: security is not about preventing compromise entirely, it is about limiting
> the damage when compromise happens — and the single strongest tool for that is least privilege (a non-root
> service, fine-grained capabilities, zero secrets on disk). And remember: hardening itself is 🔴 — every
> tightening step can break something or lock you out, so Phase 7's golden rule (a second session, an undo
> path) applies here too.

---
---

# Phase 9 — Answers to the think questions

## Answer 9.1 — The vulnerability is exploitable on both; the difference is the blast radius

(a) Yes — because the vulnerability is in the code, the attacker can use it to run remote code on **both**
servers. Layered defense does not always prevent a vulnerability from **being used**; its real job is to limit
the damage after it is used. (b) After compromise the blast radius is very different: on A, because the
application runs as `User=root`, the attacker becomes **root instantly** — reads/modifies all files, kills
other services, establishes persistence; the only defense, the host firewall, is useless against an attacker
who is already **inside**. On B, the attacker starts with only `appuser` privileges (9.1: cannot modify system
files, cannot read other users' data), the AppArmor profile (9.3) already restricts what files/operations that
process can reach, and the SG can also limit outbound connections (making data exfiltration harder). The same
vulnerability results in far smaller damage on B — that is the entire purpose of layered defense.
**Related section:** 9.1.1-9.1.2 · **Continues in:** the 9.7 failure table (row 4).

## Answer 9.2 — The obstacle is at the MAC layer; not `disable`, fix the profile

(a) Since DAC is fine (`ls -ld` permissions correct, even `sudo -u www-data cat` works), the obstacle is at a
**layer above** — most likely **MAC** (AppArmor on Ubuntu). The web server's AppArmor profile does not allow
it to read the `/var/www/data/` path. (b) Confirmation: `sudo aa-status` (is the profile enforcing) and
`sudo journalctl -k | grep -i apparmor | tail` — the blocked access shows up as a "DENIED" line pointing at
exactly this path (on SELinux, `sudo ausearch -m avc -ts recent`). (c) The correct fix is **not** to turn the
profile off entirely (`aa-disable`), because then you lose that service's second lock completely — if it is
compromised, no MAC limit remains. The correct fix is to **correct the profile**: add the `/var/www/data/`
path to it as allowed (if needed, first use `complain` mode to see what is required). Least privilege: open
only the needed path, not the whole profile.
**Related section:** 9.3.1-9.3.2 · **Continues in:** the 9.7 failure table (row 1).

## Answer 9.3 — A leaked key must be rotated; an IAM role writes no key to disk

(a) In (A), because the access key is written in plaintext in the config file, the moment the config lands in
git the key counts as **leaked** — even if you delete it from history it remains in git history and every
clone. A once-leaked secret is no longer trustworthy; the only correct response is to **rotate** the key
immediately (revoke the old one, generate a new one) and audit whether the resources that key accessed were
abused. Deleting is not enough, rotation is a must. (b) (B) largely eliminates this risk class because **no
persistent key is written to disk**: an IAM role is attached to the instance, the application gets its
credentials at runtime from the instance metadata service, and those credentials are **temporary and
auto-rotating**. Since there is no persistent secret to leak, the "config landed in git → key leaked" failure
class disappears. "A secret that does not exist cannot leak" (9.6.2).
**Related section:** 9.6.1-9.6.2 · **Continues in:** the 9.7 failure table (row 5).

---
---

# Phase 9 — Frequently asked questions

**Q1 — What is the principle of least privilege in one sentence?** Every user/service/process should have only
the least privilege needed to do its job — not one more. The goal is not to prevent compromise but to shrink
the blast radius when it happens (9.1.1).

**Q2 — What is the difference between DAC and MAC?** DAC (Phase 2: `rwx`, owner/group/other) permissions are
set by the file owner and root can override them. MAC (AppArmor/SELinux) is above, enforced by a central
policy, and even root cannot loosen it — a second lock on top of DAC (9.3.1).

**Q3 — "Permissions correct but Permission denied" — where do I look?** At the MAC layer:
`aa-status` + `journalctl -k | grep apparmor` (AppArmor), or `getenforce` + `ausearch -m avc` (SELinux). The
obstacle is not in DAC, it is in the profile (9.3.2).

**Q4 — Do I have to give root to run a service on port 80?** No. Grant only the `CAP_NET_BIND_SERVICE`
capability (`setcap` or systemd `AmbientCapabilities`) — it binds to the privileged port but does not have
root's remaining power (9.5.1).

**Q5 — Where should I put secrets?** NEVER plaintext in code/on disk. In a vault like Secrets Manager / SSM
Parameter Store, or best of all access them via an IAM role with identity, storing no secret at all (9.6).

**Q6 — An API key accidentally landed in git, I deleted it, is that enough?** No — it stays in git history and
in clones. The only correct response is to **rotate** the key (revoke + new). Deleting does not undo the leak
(9.6.1).

**Q7 — I enabled auditd, the system slowed down, why?** You are probably experiencing a log explosion from
too-broad rules: auditd produces thousands of events, journald/disk chokes. Narrow the rules, apply
rate-limiting + rotation (9.4.2).

---
---

# Phase 9 — Test yourself

Write your answers on paper, then compare with the answer key. Target: 14+ out of 18.

## Section A — Definition and mechanism

1. Define the principle of least privilege. What does "blast radius" mean and how does least privilege affect
   it?
2. What is defense in depth? Why is breaching one layer not a full compromise?
3. Explain the core difference between DAC and MAC: which one can root override, which one can it not?
4. What do AppArmor and SELinux do? What does the "profile" concept define?
5. How do Linux capabilities break the root/non-root duality? Give the `CAP_NET_BIND_SERVICE` example.
6. Why is embedding secrets in code/on disk a disaster? What is the only correct response to a once-leaked
   secret?
7. How does an IAM instance role eliminate writing secrets to disk?
8. Describe the auditd log-explosion scenario: how can an audit tool crash the system?

## Section B — Apply and diagnose

9. A service's file access gives "Permission denied" even though the `ls -l` permissions are flawless. Which
   layer do you look at, and with which command do you confirm it?
10. Your web server cannot bind to port 80 but you don't want to run the whole service as root. What is the
    solution?
11. An application was compromised and the attacker instantly controlled the whole machine. Which setting in
    the unit file could have prevented this?
12. `df -h /var/log` shows the disk full, `journalctl --disk-usage` shows a huge size. What is the likely
    security-tool cause and the two fix steps?
13. You realize an access key is in a config file and the file was committed to git. What are your first two
    steps?
14. Which three commands do you run to audit the outward-facing attack surface (ports, services, firewall)?

## Section C — Reasoning and connection

15. In Phase 8 you wrote `User=appuser`. Explain which principle in Phase 9 this is an application of and why
    it is a security decision.
16. How does the "permissions correct but blocked" failure connect Phase 2's permission model with Phase 9's
    MAC? Write the relationship of the two layers in one sentence.
17. Of which idea in Phase 7.3 (SSH/SSM) is the IAM role idea the data-access version? Write the shared
    principle.
18. How does a CIS-hardened AMI combine with Phase 8's immutable philosophy? What does "security is a property
    of the image" mean?

---

## Answer key

1. Every user/service should have only the least privilege needed; blast radius = the spread of damage when
   something is compromised; least privilege shrinks it (9.1.1). — 2. Reaching the target requires defeating
   multiple independent layers; because the layers are independent, even if one is breached the others still
   protect (9.1.2). — 3. DAC permissions are set by the owner and root can override; MAC is enforced by a
   central policy and even root cannot override — a second lock over DAC (9.3.1). — 4. They assign each
   program a profile saying "you can access only these files/operations/capabilities"; what the profile does
   not allow it cannot do even if DAC is open (9.3.1). — 5. They split root's power into ~40 fine-grained
   privileges granted one by one; `CAP_NET_BIND_SERVICE` = "bind only to privileged ports, no other root
   power" (9.5.1). — 6. The secret leaks into git history/copies/logs and cannot be undone; the only correct
   response is to rotate it (9.6.1). — 7. It gives the machine an identity (role) instead of a password to the
   application; credentials come from metadata, temporary/auto-rotating, no key written to disk (9.6.2). — 8.
   Too-broad rules → thousands of events per second → journald/disk chokes → disk fills → system
   slows/freezes; the security tool turns into a DoS (9.4.2).

9. At the MAC layer; `sudo aa-status` + `sudo journalctl -k | grep -i apparmor` (or `ausearch -m avc`)
   (9.3.2). — 10. Give the whole service not root but only the `CAP_NET_BIND_SERVICE` capability (`setcap` /
   `AmbientCapabilities`) (9.5.1). — 11. Running non-root with `User=` (and fine-grained capabilities if
   needed); a limited user instead of root would have shrunk the blast radius (9.1.1). — 12. auditd log
   explosion; (i) narrow the audit rules, (ii) apply rate-limiting + log rotation (9.4.2). — 13. (i)
   **Rotate** the key immediately (revoke + generate a new one); (ii) move the secret to Secrets Manager/SSM
   or an IAM role, take it out of code; (cleaning git history is secondary — treat the key as already leaked)
   (9.6.1). — 14. `sudo ss -tulpn` (ports), `systemctl list-units --type=service --state=running` (services),
   `sudo ufw status verbose` (firewall) (9.2.1).

15. It is an application of the principle of least privilege (9.1): running the service with a limited user
    instead of root shrinks the blast radius in case of compromise — so `User=appuser` is a deliberate
    hardening decision (9.1.1, Phase 8.2.2). — 16. DAC (Phase 2) and MAC (9.3) are two independent, stacked
    layers; an access requires **both** to allow it — even if DAC passes, MAC can deny, which is exactly
    "permissions correct but blocked" (9.3.2). — 17. It is the data-access version of the Phase 7.3 idea
    "don't distribute the SSH key, use SSM/identity"; the shared principle: **authenticate an identity instead
    of distributing a key** — no persistent secret (9.6.2). — 18. It "bakes" the hardening (profiles, closed
    services, tight SSH) into the image once; with the immutable philosophy every machine is born secure, no
    manual hardening needed → "security is not an installation step, it is a property of the image" (9.3.2,
    Phase 8.4.1).

## Scoring

| Correct | What it means |
|---|---|
| 16-18 | You grasped defense in depth and least privilege. You are ready for Checkpoint Quiz 4 (Phase 7-9). |
| 13-15 | Good. Re-read the sections of the questions you missed (especially 9.3 and 9.6). |
| 9-12 | The basics are there but fragile. Study DAC vs MAC and secrets management. |
| 0-8 | Walk through the phase again; on a test machine, read the `aa-status`, `getcap`, `ss -tulpn` outputs yourself. |

Missed question → section to return to:

| Question | Section |
|---|---|
| 1, 11, 15 | 9.1 Least privilege |
| 2 | 9.1.2 Defense in depth |
| 3, 4, 9, 16 | 9.3 MAC |
| 5, 10 | 9.5 Capabilities |
| 6, 7, 13, 17 | 9.6 Secrets management |
| 8, 12 | 9.4 Audit / log explosion |
| 14 | 9.2 Surface reduction |
| 18 | 9.3.2 + Phase 8 immutable |

---
---

# Phase 9 — Closing and Bridge to Checkpoint Quiz 4

## What you carry from this phase

Phase 9 gave you security as a **discipline**: not a single setting, but a layered defense. You gained two
lasting ideas: **least privilege** (give only what is needed — non-root service, fine-grained capabilities,
zero secrets on disk) and **blast radius** (security limits the damage, not the compromise). On top of Phase
2's DAC permissions you placed MAC (AppArmor/SELinux) and learned to connect the "permissions correct but
blocked" failure to the right layer. You saw the IAM role idea that removes secrets from disk entirely — the
data version of Phase 7.3's "don't distribute a key, authenticate an identity" principle. And you gathered all
these layers (network → access → privilege → MAC → secrets) into a single defense architecture in Figure 9.1.

## Where Checkpoint Quiz 4 (Phase 7-9) connects

Phase 7-8-9 together form a whole: the server's **relationship with the outside world and its defense**. Phase
7 opened the service to the network (reachability), Phase 8 brought the service onto the machine and made it
manageable (installation + service-ization), Phase 9 deliberately limited all of this (hardening). Checkpoint
Quiz 4 **intersects** these three phases: for example, a "I can't connect to a service" failure (Phase 7)
actually being a security group/AppArmor decision (Phase 9); or writing a unit instead of `nohup` (Phase 8)
being both an operational and a security (`User=`, Phase 9) decision. The quiz measures whether you can see
these three phases as different faces of a single event.

> **🤔 Phase output — ask yourself:** In these three phases (7-8-9) the same command, `ss -tulpn`, appeared
> through three different lenses: in Phase 7 "is the service listening on the right address" (reachability), in
> Phase 8 "is my service up" (operation), in Phase 9 "how many unnecessary ports are open outward" (attack
> surface). Before entering Checkpoint Quiz 4, think: how does a cloud engineer looking at a single `ss -tulpn`
> output ask these three questions **at the same time**? This is the three phases becoming a single reflex.
>
> **🧪 Lab 9 idea (on your own test instance):** (1) See the loaded AppArmor profiles with `sudo aa-status`.
> (2) Audit the outward-facing ports with `sudo ss -tulpn` — for every `0.0.0.0` line ask "is this really
> needed?" (3) Check the audit service with `sudo systemctl status auditd`; see the log load with
> `journalctl --disk-usage`. (4) Examine a capability example with `getcap /usr/bin/ping`. (5) Run a test
> service first with `User=root`, then with `User=nobody`, and see the difference in the files it can reach —
> shrink the blast radius yourself. These steps gather this phase's defense-in-depth reflex in your hands.

---

> **Navigation:** [◀ Phase 8 — Packages and Service-ization](Phase_8_Packages_and_Service_ization.md) · **Phase 9** · [Checkpoint Quiz 4 ▶](Checkpoint_Quiz_4.md)
