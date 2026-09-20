# Checkpoint Quiz 4 — Phase 7–9: Networking, Service-ization and Security

> **Navigation:** [◀ Phase 9 — Security and Hardening](Phase_9_Security_and_Hardening.md) · **Checkpoint Quiz 4** · [Phase 10 — Automation and Scripting ▶](Phase_10_Automation_and_Scripting.md)

---

## What does this quiz measure?

The previous checkpoint quizzes combined two phases. This quiz combines **three** — because Phase 7, 8 and 9
are really three faces of a single story: **the server's relationship with the outside world and its defense.**

Phase 7 **opened** the service to the network (reachability: bind address, port, security group, firewall).
Phase 8 **brought the service onto the machine and made it manageable** (installation, systemd unit, `User=`,
auto-restart). Phase 9 deliberately **limited** all of this (least privilege, MAC, secrets, blast radius). The
three intersect in a single event: "I can't connect to a service" (Phase 7) is often the result of a security
decision (Phase 9: SG/AppArmor); writing a unit instead of `nohup` (Phase 8) is both an operational and a
security (`User=`, Phase 9) decision; and the same `ss -tulpn` command looks at three phases through three
different lenses. This quiz measures whether you can see these three phases as a **single reflex**.

**How to study:**

- Solve with pen and paper; do not move on without writing the answer.
- The answer key tells you **at which intersection of phases** each question stands — a missed question points
  not to a phase but to the **bridge** between phases.
- There are 21 questions: Section A (connection reasoning, 1–9), Section B (scenario: "service unreachable
  from outside, then compromised", 10–15), Section C (command and output reading, 16–21).
- Target time: ~1 hour. But time is not important; what matters is being able to justify in one sentence *why*
  each answer is what it is.

> **🤔 Before you start:** Complete this sentence in your own words: *"`systemctl status` is green but I still
> can't reach the service from outside — because `systemctl` only checks the ___ gate, not the rest of the
> gates."* This one sentence combines Phase 7 (gates) with Phase 8 (service is up). Write your answer aside; we
> will see it in Question 2 and Section B.

---

# Section A — Connection reasoning (1–9)

Each question asks you to combine the knowledge of at least two phases. Give a short but justified answer.

**1.** In Phase 8 you learned to write a service as a systemd unit instead of `nohup python app.py &`. In Phase
9 you saw that the `User=appuser` line is a security decision. Explain why running with `nohup` also violates
Phase 9's least privilege principle: what identity/privilege does the `nohup` process run with, and how does
that grow the blast radius compared to a unit?

**2.** In Phase 7 we said "all five gates must be open for a service" (DNS, SG, host firewall, bind address,
process). In Phase 8 we saw that `systemctl status` being green shows the service is up. Which **one** of these
five gates does `systemctl status` verify, and why is "green but unreachable" exactly the intersection of Phase
7 and Phase 8?

**3.** In Phase 7 we saw that binding a service to `127.0.0.1` instead of `0.0.0.0` closes it to the outside.
In Phase 9 we saw least privilege / surface reduction. On a server where only the local application uses a
database, binding the DB to `127.0.0.1` is a concrete application of which Phase 9 principle, and which extra
defense layer (defense in depth) does it provide, unlike closing the port in the SG?

**4.** In Phase 7 we saw that the Security Group is "invisible to the operating system" (the SG drops the
packet before it reaches the OS). In Phase 9 we saw defense in depth. Even if an attacker somehow gets past the
host firewall (ufw), why can the SG still protect — how does these two firewalls being **independent layers**
exemplify defense in depth?

**5.** In Phase 8 we saw that `daemon-reload` makes systemd read the change in the unit file (running state vs
persistent definition). Suppose in Phase 9 you change a service's `User=` or `AmbientCapabilities=` line. Which
two steps (Phase 8) must you do in order for this security change to **take effect**, and what happens if you
just edit the file and leave it?

**6.** In Phase 7 we made SSH key-only + root off. In Phase 9 we positioned this as an "access layer." The IAM
instance role idea in Phase 9 (access by identity without writing a secret to disk) is the data-access version
of which SSH idea in Phase 7.3 (SSM/identity instead of distributing a key) — write the shared principle in one
sentence.

**7.** In Phase 8 we saw immutable/baked AMI (bake the software into the image). In Phase 9 we saw the
CIS-hardened AMI. How do these two ideas combine in the form of "put X into the image once" instead of "do X by
hand on every machine" — and explain the sentence "security is not an installation step, it is a property of
the image" with this combination.

**8.** In Phase 7 we used `ss -tulpn` to check "is the service listening on the right address." In Phase 9 we
used the same command to check "how many unnecessary ports are open outward." Write the three questions a cloud
engineer looking at a single `ss -tulpn` output should ask **at the same time** (Phase 7 reachability, Phase 8
operation, Phase 9 attack surface).

**9.** In Phase 9 we saw that the "permissions correct but Permission denied" failure is at the MAC
(AppArmor/SELinux) layer. A service cannot bind to a port: `User=appuser` (not root) and port 80. Is this a
**MAC** problem or a **capability** (Phase 9.5) problem — how do you distinguish the two, and what is the
correct fix?

---

# Section B — Scenario: "Service unreachable from outside, then compromised" (10–15)

> **Incident:** An engineer ran a FastAPI application on port 8000 on a `t3.medium` Ubuntu instance.
> `curl localhost:8000` works from **inside** the machine, but it is **unreachable** from a browser (the
> instance's public IP). To "fix it fast," the engineer did the following: (1) restarted the application as
> **root** with `sudo nohup python app.py &`; (2) added a rule to the Security Group opening all ports for
> `0.0.0.0/0`; (3) left the application's bind address unchecked. Then access worked and the engineer closed
> the matter. **Three weeks later:** the server was compromised via a known vulnerability in the application;
> the attacker became root on the machine, read other services' data, and added the machine to a botnet. The
> logs (later) show that `app.py` first bound to `127.0.0.1:8000` instead of `0.0.0.0:8000`, but the engineer
> had added `--host 0.0.0.0`.

**10.** First diagnosis (unreachability): `curl localhost:8000` works but it is unreachable from outside. This
first failure, **before** compromise, can be explained by which **two** of Phase 7's five gates (bind address
and SG)? Connect each in one sentence.

**11.** The engineer's "fix" (1) (`sudo nohup python app.py &`, root) violated the lesson of which **two**
phases at once? (Phase 8: why `nohup` is not production; Phase 9: why root is a disaster.) Write each
separately.

**12.** The engineer's "fix" (2) (all ports for `0.0.0.0/0` in the SG) — why did it grow the attack surface to
a disastrous size while solving reachability? Combining Phase 7 (the SG's function) and Phase 9 (surface
reduction) knowledge, write what the "correct" fix should have been (which single port, to which source?).

**13.** Root-cause reasoning (what was the real failure?): According to the log the app actually bound to
`127.0.0.1`. The engineer adding `--host 0.0.0.0` was the correct fix — but they opened the SG wide **without
knowing that**. Once the bind address is fixed, why should the SG still be narrowed to only 8000/the needed
source — why is "access worked" a misleading success signal?

**14.** Blast radius: If the application had run with `User=appuser` (not root) and had an AppArmor profile, how
would the post-compromise damage have been **different**? Map Phase 9's two layers (least privilege + MAC)
one by one to what each would block the attacker from.

**15.** Prevention + generalization: What are the **three habits** the engineer should adopt so this incident
never happens again (each tied to a phase: Phase 7 direction of diagnosis, Phase 8 service-ization, Phase 9
least privilege)? Also summarize the incident in one sentence: "solving reachability at the wrong layer (___)
sacrificed security (___)."

---

# Section C — Command and output reading (16–21)

Read each output fragment below and answer what is asked.

**16.** `ss -tulpn` output:
```
Netid State  Local Address:Port   Process
tcp   LISTEN 127.0.0.1:8000       python
tcp   LISTEN 0.0.0.0:22           sshd
```
The application is unreachable from outside but SSH works. Which single difference in the output (Phase 7.4.2)
explains the failure, and what should you change the bind address to in order to make the app reachable?

**17.** `systemctl status myapp` output:
```
● myapp.service - My FastAPI app
     Loaded: loaded (/etc/systemd/system/myapp.service; enabled)
     Active: active (running) since ...
       Main PID: 1234 (python)
```
`Active: active (running)` is **green**, but the user says "I can't reach the site." Which gate (Phase 7) does
`systemctl status` being green verify, which ones does it **not** verify — and what is the next command to
check?

**18.** `sudo ufw status verbose` output:
```
Status: active
Default: deny (incoming), allow (outgoing)
To          Action  From
22/tcp      ALLOW   Anywhere
```
The application is on port 8000 but it is not in the list. What is happening from the host firewall's
perspective (Phase 7.4.4), and because this is a separate layer from the SG (Phase 9), why must you check
**both**?

**19.** `sudo journalctl -k | grep apparmor` output:
```
apparmor="DENIED" operation="open" profile="myapp" name="/var/www/data/x.txt" ...
```
The service's file permissions (`ls -l`) are flawless but it cannot read the file. What layer does this output
(Phase 9.3.2) prove the failure is at, and why is the correct fix not to **turn off** the profile?

**20.** `getcap` + `ss` sequence:
```
$ getcap /opt/myapp/server
(empty output)
$ sudo -u appuser /opt/myapp/server --port 80
Error: Permission denied (bind :80)
```
The service is not root (`appuser`) and cannot bind to port 80. This is not a permission/MAC problem; what
missing thing (Phase 9.5) causes it, and with which single command do you fix it without making the whole
service root?

**21.** Comparison of two situations:
```
A:  User=root       (in the unit)      →  when compromised: attacker is root on the whole machine
B:  User=appuser    (in the unit)      →  when compromised: attacker is only appuser
    + AppArmor profile "myapp"                             + the paths the profile allows
```
The same application vulnerability is exploited on both. In Phase 9's language: which concept names the
difference between A and B, and why is "limiting the damage" rather than "preventing the vulnerability" the real
job of security?

---

## Answer Key

**1.** `nohup python app.py &` runs the process with the identity of the user who started it; usually, "to be
quick," it is run with `sudo`, i.e. as **root**. If a process running as root is compromised, the attacker
instantly takes the whole machine — the blast radius is maximal. A systemd unit, by contrast, confines the
process to a limited identity with `User=appuser`; `nohup` offers this control at all, so it structurally
violates least privilege. · *Phase 8.2 × Phase 9.1* — **2.** `systemctl status` verifies only the **fifth gate**
(is the process up, is it running correctly); it **never sees** DNS, SG, host firewall, or the bind address.
That is why "green but unreachable" is exactly the intersection: the service (Phase 8) is fine but one of the
four network gates in front of it (Phase 7) is closed. · *Phase 7.4 × Phase 8.2* — **3.** Binding the DB to
`127.0.0.1` is a concrete application of **least privilege / surface reduction** — the service is reachable only
as much as needed. Unlike closing the port in the SG: this is a separate, independent layer (defense in depth).
Even if the SG is opened by mistake, the DB stays unreachable because it never listens on the external
interface — two independent locks. · *Phase 7.4.2 × Phase 9.1/9.2* — **4.** The SG and host firewall are
independent layers: the SG works at the cloud network level (before reaching the OS), ufw at the OS level. Even
if the attacker gets past ufw (e.g. via a misconfiguration), the SG can drop the packet before it even reaches
the OS; and vice versa. Because when one is breached the other still protects, this is a full example of
defense in depth. · *Phase 7.4 × Phase 9.1.2* — **5.** Two steps: (1) `sudo systemctl daemon-reload` (systemd
reading the changed unit definition), (2) `sudo systemctl restart myapp` (restarting the process with the new
definition). If you just edit the file and leave it, the running process keeps running with the **old**
`User=`/capability — the security change exists in the "persistent definition" but not in the "running state."
· *Phase 8.2.2 × Phase 9* — **6.** It is the data-access version of the Phase 7.3 idea "don't distribute the SSH
key to machines; access via SSM/identity." The shared principle: **authenticate an identity instead of
distributing a persistent secret (a key)** — when there is no persistent secret distributed/stored, there is no
secret to leak. · *Phase 7.3.4 × Phase 9.6.2* — **7.** Both are "bake X into the image once" instead of "do X by
hand on every machine": the baked AMI puts software, the hardened AMI puts security (profiles, closed services,
tight SSH) into the image. Combined, every new instance is **born** secure, no one hardens by hand → "security
is not an installation step, it is a property of the image"; the immutable philosophy makes security
reproducible. · *Phase 8.4 × Phase 9.3.2* — **8.** (i) *Reachability (Phase 7):* is the service listening on the
right address (`0.0.0.0` vs `127.0.0.1`)? (ii) *Operation (Phase 8):* is/are the service(s) I expect really up
and on the right port? (iii) *Attack surface (Phase 9):* how many of these outward-listening ports are
**unnecessary** and should be closed? · *Phase 7.4 × Phase 8.2 × Phase 9.2* — **9.** This is a **capability**
problem, not MAC. Distinguishing: a non-root process being unable to bind to a port below 1024 like 80 is the
classic privileged-port problem; if it were MAC you would see a DENIED line in `journalctl -k | grep apparmor`,
and there is none. Fix: without making the whole service root, grant `CAP_NET_BIND_SERVICE` (`setcap` or
`AmbientCapabilities=` in the unit). · *Phase 9.5 × Phase 9.3*

**10.** Two gates: (i) **bind address** — if the app binds to `127.0.0.1` it is reachable only from inside the
machine (`curl localhost` works), never from outside; (ii) **Security Group** — if port 8000 is not open
outward in the SG, the packet never reaches the OS. If even one of these is closed, it is unreachable from
outside. · *Phase 7.4.2 × Phase 7.4.3* — **11.** (Phase 8) `nohup` is not production: it does not restart on
crash, is gone on reboot, its logs are scattered, it can die when the session logs out — it is an unmanaged
stray process, not a managed service. (Phase 9) running as root maximizes the blast radius: a single
vulnerability = the whole machine. In one "quick" step the engineer sacrificed both operation and security. ·
*Phase 8.2.1 × Phase 9.1.1* — **12.** Opening **all ports** for `0.0.0.0/0` reverses the SG's sole function
(shrinking the surface): now every listening port on the machine (SSH, DB, internal services) is open to the
whole internet — a huge attack surface. The correct fix was to open only **port 8000**, and if possible only to
the needed source (e.g. a load balancer/CloudFront or office IP) — a whitelist, not a blacklist. · *Phase 7.4.3
× Phase 9.2.1* — **13.** Because the real failure was the bind address (`127.0.0.1`); once `--host 0.0.0.0` is
added the app already starts listening outward and access would have worked with a **narrow** SG (only 8000).
Opening the SG wide was unnecessary and dangerous. "Access worked" is misleading because they made **multiple
changes at once** and did not test which was really needed — the surface-reduction discipline (Phase 9) is
exactly "find the narrowest configuration that works." · *Phase 7.4.2 × Phase 9.2.1* — **14.** (least privilege)
With `User=appuser` the attacker would be only `appuser`, not root: cannot modify system files, cannot read
other users'/services' data, would find it hard to establish persistence. (MAC) Because the AppArmor profile
already limits the set of files/operations/network that process can reach, the attacker cannot step outside the
profile (e.g. reading `/etc/shadow` or arbitrary network connections are blocked). The two layers together
reduce the damage from "the whole machine" to "the narrow box of a single process." · *Phase 9.1.1 × Phase 9.3*
— **15.** Three habits: (i) **Phase 7 — diagnose outside → in:** on unreachability, first check bind address +
SG, do not open the SG blindly. (ii) **Phase 8 — write a unit, not `nohup`:** make the service a managed systemd
unit with `User=appuser`. (iii) **Phase 9 — least privilege:** do not run as root, open the SG to the narrowest
source, close unnecessary ports. One sentence: "solving reachability at the wrong layer (**by opening the SG
wide**) sacrificed security (**by maximizing the blast radius / root + open surface**)." · *Phase 7.4 × Phase
8.2 × Phase 9.1*

**16.** The single difference is the **bind address**: the app listens on `127.0.0.1:8000` (local only), while
sshd listens on `0.0.0.0:22` (all interfaces). That is why SSH works from outside and the app does not. Fix:
make the app's bind address `0.0.0.0:8000` (the application's `--host 0.0.0.0` or config). · *Phase 7.4.2* —
**17.** `systemctl status` being green verifies only the **process gate** (Phase 7's 5th gate: the service is up
and running); it does not verify DNS, SG, host firewall, or the bind address. The next command: `sudo ss
-tulpn` — to see on which address:port the service is listening (the bind-address gate), then check the SG and
ufw. · *Phase 8.2 × Phase 7.4* — **18.** The host firewall's default is `deny (incoming)` and only 22 is in the
list; 8000 is **not open**, so ufw drops incoming 8000 traffic. Because the SG is a separate, cloud-level layer
(Phase 9 defense in depth), 8000 must be open in **both** the SG and ufw — if even one is closed, it is
unreachable. So on "unreachable" check **both** firewalls. · *Phase 7.4.4 × Phase 9.1.2* — **19.** The
`apparmor="DENIED"` line proves the failure is at the **MAC layer** (AppArmor) — even though the DAC permissions
(`ls -l`) are correct, the profile denies access to this path. The correct fix is not to turn off the profile
(`aa-disable`), because then you lose that service's second lock entirely; the right fix is to add the
`/var/www/data/` path to the profile as allowed (least privilege: open only the needed path). · *Phase 9.3.2* —
**20.** The missing thing is a **capability**: binding to a privileged port like 80 requires
`CAP_NET_BIND_SERVICE`; without it the non-root `appuser` cannot bind. The fix without making the whole service
root: `sudo setcap 'cap_net_bind_service=+ep' /opt/myapp/server` (or `AmbientCapabilities=CAP_NET_BIND_SERVICE`
in the unit). · *Phase 9.5.1* — **21.** The concept that names the difference is the **blast radius**: the same
vulnerability spreads to the whole machine (root) on A, while it is confined to the narrow box of a single
process (`appuser` + AppArmor) on B. Security's real job is "limiting the damage" because no system stays
vulnerability-free forever — sooner or later a vulnerability is exploited; defense in depth + least privilege
determine how far the loss spreads when that moment comes. · *Phase 9.1.1 × Phase 9.3*

---

## Scoring

| Correct | What it means |
|---|---|
| 18-21 | You combined networking, service-ization and security into a single reflex. You are ready for Phase 10. |
| 14-17 | Good. Re-read the **bridge** sections (table below) that your missed questions point to. |
| 9-13 | You know the phases individually but struggle at the intersection. Redo the Section B scenario from scratch. |
| 0-8 | Walk through Phase 7, 8 and 9 individually again; especially 7.4 (gates), 8.2 (unit) and 9.1 (least privilege). |

Missed question → bridge to return to:

| Question | Bridge (section × section) |
|---|---|
| 1, 11 | `nohup` vs unit + root blast radius (8.2 × 9.1) |
| 2, 17 | Five gates × `systemctl status` green (7.4 × 8.2) |
| 3, 16 | Bind address × surface reduction (7.4.2 × 9.2) |
| 4, 18 | SG vs ufw independent layers (7.4.4 × 9.1.2) |
| 5 | daemon-reload × security change (8.2.2 × 9) |
| 6 | IAM role × SSH/SSM identity (7.3.4 × 9.6.2) |
| 7 | baked AMI × hardened AMI (8.4 × 9.3.2) |
| 8 | `ss -tulpn` three lenses (7.4 × 8.2 × 9.2) |
| 9, 20 | Capability vs MAC distinction (9.5 × 9.3) |
| 10, 12, 13, 15 | Scenario: reachability × surface (7.4 × 9.2) |
| 14, 21 | Blast radius × least privilege + MAC (9.1 × 9.3) |
| 19 | "Permissions correct but blocked" = MAC (9.3.2) |

---

## Closing

This quiz tested how three phases intersect in a single engineering event: opening a service to the network
(Phase 7), making it manageable (Phase 8), and deliberately limiting it (Phase 9) — three faces of the same
coin. The scenario's lesson is single and expensive: **solving reachability at the wrong layer sacrifices
security.** The engineer opened the SG wide to "get the site up"; the real failure was a single bind address.
Carry the three lessons together: (1) always diagnose outside → in, do not open gates blindly; (2) starting a
process is not managing it — write a unit with `User=appuser`, not `nohup`; (3) security is not about
preventing the vulnerability but about shrinking the blast radius when the vulnerability is exploited — least
privilege and defense in depth are exactly for that.

Phase 10 moves you toward **automating** the manual steps of these three phases: turning everything you set up,
opened, and hardened by hand into repeatable scripts and configuration. "Every manual task you do once should
be a script the second time."

---

> **Navigation:** [◀ Phase 9 — Security and Hardening](Phase_9_Security_and_Hardening.md) · **Checkpoint Quiz 4** · [Phase 10 — Automation and Scripting ▶](Phase_10_Automation_and_Scripting.md)
