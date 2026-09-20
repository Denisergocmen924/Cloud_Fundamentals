# Phase 7 — Networking (OS Layer): The Server's Outside World

> **Navigation:** [◀ Checkpoint Quiz 3](Checkpoint_Quiz_3.md) · **Phase 7** · [Phase 8 — Packages and Service-ization ▶](Phase_8_Packages_and_Service_ization.md)

---

## Where we are coming from

At the end of Phase 6 we left you a riddle: if a service is "listening on a port" but cannot be reached
from outside, which relative is this of the disk failure "the device exists but is not mounted → not in
`df`"? The answer is exactly this phase's subject. Until now we have always been **inside a single
machine**: processes (Phase 3), memory (Phase 4), boot (Phase 5), disk (Phase 6). Phase 7 connects the
machine to the **outside world** — and that lesson you learned in storage repeats here verbatim: a
resource **existing** and a resource being **reachable** are two different things.

The knowledge of three phases pays off directly here:

- **Phase 5 — services are units.** SSH, nginx, your app: all are `.service` units started by systemd
  (5.3). "Is the service listening" is really the question "is the unit `active (running)`."
- **Phase 3 — a listening process.** A port is opened and held by a **process** (3.1). The PID in the last
  column of `ss -tulpn` is Phase 3's process; "the port is closed" is most often "the process has died."
- **Phase 2 — SSH keys.** In Phase 2.3 you saw the journey of an SSH key (public/private,
  `authorized_keys`, permissions). This phase turns that foundation into daily operations: `~/.ssh/config`,
  ssh-agent, hardening.

## The question of this phase

Phase 6 said "how does data persist on disk." This phase opens the machine to the network:

> *"A service is running on the machine — which gates does an inbound connection have to pass through to
> reach it, and if one of those gates is closed, how do I find the fault?"*

Most of a cloud engineer's daily life revolves around a single sentence: **"I can't connect to the
service, why?"** This one question is 90% of network failures, and the answer is almost never "the
internet is broken." The answer is a **chain of gates**: the connection has to pass through the security
group (cloud), the host firewall (OS), and **which address the service binds to** (127.0.0.1 or 0.0.0.0).
SSH is at the center of this phase — because it is the tool a cloud engineer uses most, and says "why
can't I connect" about most.

By the end of this phase you will have gained a reflex that diagnoses the "I can't connect to the service"
event without panic, layer by layer (does the name resolve → is the port being listened on → is it bound
to the right address → does the firewall/SG allow it). And you will see SSH not merely as a "connect
command" but as a service that must be securely configured.

---

## By the end of this phase

- You will be able to read a machine's network state (interfaces, IPs, default gateway) with `ip addr` and
  `ip route`; and know at the concept level how netplan holds the network config on Ubuntu
- You will be able to explain the host side of name resolution (`/etc/hosts`, `/etc/resolv.conf`,
  systemd-resolved); and tie the failure "ping by IP works but by name does not" to `resolv.conf`
- You will be able to use SSH deeply: key-based identity, `~/.ssh/config`, ssh-agent, port
  forwarding/tunneling; and harden the server (root login off, password off, key only)
- You will be able to read which service is listening on which port with `ss -tulpn`; and in the failure
  "the service is up but I can't connect" tell apart the `127.0.0.1` vs `0.0.0.0` bind
- You will be able to explain what a host firewall (ufw/firewalld/nftables) is and why it is a **separate**
  layer from the cloud Security Group; and that both can cut traffic
- **Cloud:** you will be able to connect to EC2 over SSH and manage the keypair; explain why SSM Session
  Manager offers safer access without opening the SSH port; and that the Security Group and host firewall
  are two independent gates

---

## The phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 7.1 | Interface and address management | `[mechanism]` | The machine's network identity: `ip addr`, `ip route` |
| 7.2 | Name resolution (host side) | `[mechanism]` | The "IP works but name doesn't resolve" failure |
| 7.3 | SSH — in depth | `[application]` | **The heart of the phase** — the most-used daily tool |
| 7.4 | Port and socket state | `[application]` | **"The service is up but I can't connect"** |
| 7.5 | Host firewall and Security Group | `[concept]` | Two separate lines of defense |
| 7.6 | When this phase breaks | — | The signatures of network failures |

> **How to work in this phase:** This phase's observation commands are almost entirely 🟢 (`ip addr`,
> `ip route`, `ss -tulpn`, `cat /etc/resolv.conf`, `ssh -v` — all read/try, make no permanent change). Two
> places need care: (1) **SSH hardening** is 🔴 — if you change `sshd_config` wrong and drop the session,
> you **lock yourself out** (especially in the cloud, if you have no console access). Rule: test an
> `sshd_config` change while **a second session is still open**, and let every step have an undo path. (2)
> **Firewall rules** are 🔴 — a wrong rule (especially one closing the SSH port) locks you out. The most
> illuminating experiment: on your own test instance, bind a service first to `127.0.0.1` then to
> `0.0.0.0` and see the difference in the `ss -tulpn` output and how outside reachability changes.

---
---

# 7.1 Interface and Address Management

## 7.1.1 The machine's network identity: interface, IP, gateway `[mechanism]`

A machine's identity on the network has three parts, and the `ip` command shows all three:

1. **Interface.** The machine's "door" onto the network — a physical card, or a virtual NIC in the cloud.
   Names: `lo` (loopback — the internal interface the machine talks to itself over, always `127.0.0.1`),
   `eth0` / `ens5` / `enX0` (the actual network interface; in the cloud usually a single primary
   interface).
2. **IP address.** The address assigned to the interface — the machine's "house number" on the network. In
   the cloud this is usually a **private** IP (like `172.31.x.x`); the outward-facing **public** IP most
   often sits on a separate layer (NAT/Elastic IP) and may not be visible inside the machine.
3. **Default gateway and route.** The answer to "where do I send traffic that is going anywhere not on this
   network?" That is the `default via ...` line in `ip route`; if it is wrong the machine sees the local
   network but can't reach the internet.

> **🔧 See it on your machine** 🟢 — read the machine's network identity
>
> ```
> $ ip addr
> 1: lo: <LOOPBACK,UP> ... inet 127.0.0.1/8 scope host lo
> 2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
>     inet 172.31.20.15/20 ... scope global ens5
> $ ip route
> default via 172.31.16.1 dev ens5
> 172.31.16.0/20 dev ens5 proto kernel scope link src 172.31.20.15
> ```
>
> The `ens5` interface's private IP is `172.31.20.15`. `default via 172.31.16.1` = traffic going to the
> internet/other networks goes to this gateway. `lo`'s `127.0.0.1` is always there and has nothing to do
> with the outside world — in 7.4 this distinction will be critical.

> **⚠️ Common misconception: "I'll see my public IP in `ip addr`."**
>
> Usually you won't. Looking from **inside** the machine in the cloud, only the **private** IP is visible;
> the public IP is mapped to it (1:1) on a NAT layer outside the machine. That's why `ip addr` shows
> `172.31.x.x` while people connect from the internet via `3.120.x.x` — they are the same machine. The
> answer to "where is my public IP" is not inside the machine but in the cloud console (or via a metadata
> query). This distinction is vital when solving failures like "my machine reaches the internet but I
> can't connect to it."

## 7.1.2 netplan: the network config on Ubuntu `[concept]`

`ip addr` shows you the network's **current** state — but that state, just like `mount` in Phase 6, is not
persistent (it is rebuilt on reboot). The **persistent** definition of network settings on Ubuntu is in
**netplan**: the `/etc/netplan/*.yaml` files tell it, at boot, how each interface should be configured
(DHCP, or a static IP). In the cloud this file usually says "use DHCP" and the cloud's DHCP server handles
the rest — which is why an EC2 instance gets its IP automatically.

The conceptual link here is the same as Phase 6: the **running state** (`ip addr`, `mount`) and the
**persistent config** (netplan, `fstab`) are separate layers. An IP you add by hand with `ip addr add`
goes away on reboot; write it into netplan and it stays. You don't need to learn netplan deeply in this
phase — it's enough to be able to say "the persistent network setting lives here, and because it's usually
DHCP in the cloud you rarely touch it."

> **🤔 Think 7.1** — On a machine `ping 8.8.8.8` works (it reaches the internet by IP) but
> `ping google.com` gives "Name or service not known". Which of the three parts from 7.1 (interface, IP,
> gateway) is the problem **not** in, and which file of the next section (7.2) does this observation point
> you to? (Hint: if it can get out by IP, the interface/IP/gateway are working.)
>
> *(Answer: at the end of the phase)*

---
---

# 7.2 Name Resolution (Host Side)

## 7.2.1 From name to IP: the two stops of resolution `[mechanism]`

People type `google.com`, but the network wants an IP like `142.250.x.x`. This translation is called
**name resolution** (DNS). On a Linux machine, resolution looks in two places in order:

1. **`/etc/hosts`** — a hand-written, local "name → IP" ledger. If you write `10.0.0.5  db.internal` here,
   that machine turns `db.internal` directly into `10.0.0.5` without ever asking DNS. It is checked
   **before** DNS; used to test a name or to temporarily override a DNS entry.
2. **DNS server** — if not in `/etc/hosts`, the machine asks a configured DNS server. Which server it asks
   is written in the `nameserver` line of `/etc/resolv.conf`.

On modern Ubuntu, **systemd-resolved** also sits in between: `/etc/resolv.conf` usually points to
`127.0.0.53` (the local resolved service), which forwards to the real DNS server (in the cloud usually the
VPC's resolver). The detail doesn't matter; what's critical is the chain: **name → `/etc/hosts` → DNS
server (`resolv.conf`) → IP.**

## 7.2.2 "IP works but the name doesn't resolve" — the resolv.conf failure `[mechanism]`

This is one of the most classic and most misleading network failures because it gives the feeling of
"the internet seems to work but doesn't." Symptom: `ping 8.8.8.8` **works** (packets go and come by IP —
so interface, IP, gateway, the internet path are **sound**), but `ping google.com` **gives** "Name or
service not known". The only difference: the second requires a **name resolution**. So the fault is
exactly at the resolution layer — most likely `/etc/resolv.conf` is empty, contains a wrong `nameserver`,
or the DNS server is unreachable.

> **🔧 See it on your machine** 🟢 — inspect the resolution chain
>
> ```
> $ ping -c1 8.8.8.8            # by IP: if it works, the network path is sound
> 64 bytes from 8.8.8.8: ...
> $ ping -c1 google.com        # by name: if it breaks here, a resolution problem
> ping: google.com: Name or service not known
> $ cat /etc/resolv.conf       # which DNS server is written?
> nameserver 127.0.0.53
> $ resolvectl status          # which server systemd-resolved actually uses
> ```
>
> Diagnostic logic: **if IP works and name doesn't, the problem is always in resolution** — not on the
> network path. The place to look is `/etc/resolv.conf` (and `resolvectl status`). Common cause:
> `resolv.conf` is empty or points to an unreachable DNS server.

> **💡 Cloud connection — DNS and the VPC resolver in the cloud:** Instances inside a VPC use a DNS
> resolver provided by AWS (the VPC's `.2` address, e.g. `172.31.0.2`). This resolver resolves both
> internet names and VPC-internal private names (e.g. the name of an RDS endpoint). Failures like "the
> instance reaches the internet but can't resolve an AWS service endpoint" are usually a break either in
> the path (route/SG) to that resolver or in the `resolv.conf`/resolved config. The rule is the same:
> first separate the network path with `ping IP`, then test name resolution separately — don't mix the
> two.

> **🤔 Think 7.2** — An application throws "could not connect to database `db.prod.internal`". You try
> `ping db.prod.internal`: "Name or service not known". But you know the database's IP (`10.0.5.20`) and
> `ping 10.0.5.20` **works**. (a) Which layer is the fault in (the network path, or resolution)? (b) Which
> single line do you add to `/etc/hosts` to make the app work **temporarily**, (c) why is this not a
> permanent fix?
>
> *(Answer: at the end of the phase)*

---
---

# 7.3 SSH — In Depth

## 7.3.1 Key-based identity: why not a password `[application]`

In Phase 2.3 you saw the journey of an SSH key as a concept; now we turn it into daily operations. SSH is
the standard way to open a secure shell on a remote machine and is the tool a cloud engineer uses most.
There are two ways to prove identity, but in the cloud practically only one is used:

- **Password** — memorable but weak: guessable, brute-forceable, leakable. In the cloud it is usually
  **turned off**.
- **Key pair** — two mathematically linked files: the **private key** (stays with you, never shared) and
  the **public key** (placed on the server, in `~/.ssh/authorized_keys`). When connecting, the server
  sends a challenge only the holder of the right private key can answer; the password never crosses the
  network. Phase 2.3's diagram was exactly this flow.

The critical permission rule (Phase 2's permissions come back here): SSH is **very strict** for security.
The private key must be `600` (only the owner reads it), the `~/.ssh` directory `700`; otherwise SSH
**silently refuses** the key (it says "permissions too open"). This is the most common cause of the
failure "my key is correct but it still asks for a password / rejects me."

> **🔧 See it on your machine** 🟢 — connect with a key and verify permissions
>
> ```
> $ ls -l ~/.ssh/id_ed25519           # private key must be 600
> -rw------- 1 you you 411 ... id_ed25519
> $ ssh -i ~/.ssh/id_ed25519 ubuntu@3.120.0.10
> # if you can't connect, see why:
> $ ssh -v -i ~/.ssh/id_ed25519 ubuntu@3.120.0.10   # -v = step-by-step handshake
> ```
>
> `ssh -v` (or `-vvv`) shows at **which step** the connection sticks: was TCP established, was the server
> key presented, which auth method was tried, why was it refused. It is the one tool that turns "I can't
> connect" into "I can't connect at this step, for this reason" — it stands at the center of 7.6's
> diagnostic reflex.

## 7.3.2 `~/.ssh/config` and ssh-agent: daily ergonomics `[application]`

Typing `ssh -i long/path/key.pem ubuntu@3.120.0.10 -p 22` every time is both tiring and error-prone.
`~/.ssh/config` turns it into named shortcuts:

```
Host prod-web
    HostName 3.120.0.10
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
```

Now just `ssh prod-web` is enough. This isn't only comfort; by pairing the right key with the right server
it also reduces "I sent the wrong key" errors.

**ssh-agent** is a helper that opens your private key once (if it's passphrase-protected) and holds it in
memory: you don't re-enter the key's passphrase on every connection, but the key is not written to disk
unencrypted. You add the key to the agent with `ssh-add`.

> **💡 Cloud connection — EC2 keypair management:** When launching an EC2 instance you choose a **keypair**;
> AWS automatically places the chosen public key into the instance's `~/.ssh/authorized_keys` (usually for
> the `ubuntu` or `ec2-user` user). The private key stays **with you** — AWS does not store it, and if you
> lose it you cannot re-download it. That's why "I can't connect to the instance, I lost the key" is a
> serious situation (the fix is usually: stop the instance, attach its root disk to another instance, fix
> `authorized_keys` — Phase 6 knowledge!). Rule: generate your private keys once, keep them safely, and
> distribute only the **public** side to servers.

## 7.3.3 Port forwarding (tunneling): safe access to a closed service `[concept]`

Sometimes you can't reach a service directly because it listens only **inside the machine** (`127.0.0.1`)
or the firewall/SG is closed to the outside — but you have SSH. SSH **port forwarding** (tunneling) solves
exactly this: from inside the SSH connection you open a tunnel and carry a port on the remote machine to
**your own local machine**.

Example: on the remote server a database listens on `127.0.0.1:5432` (closed to the outside, secure). This
command:

```
$ ssh -L 5432:127.0.0.1:5432 prod-web
```

binds `localhost:5432` on your own machine, through the SSH tunnel, to the remote server's
`127.0.0.1:5432`. Now you connect a local DB tool to `localhost:5432`, and the traffic flows through the
encrypted SSH tunnel. This is the safe way to reach a DB "without opening its port to the internet" and is
heavily used in the cloud (the bastion/jump host pattern).

## 7.3.4 SSH hardening: locking down the server `[application]`

SSH is an open door; every internet-facing server constantly receives SSH brute-force attempts. Hardening
has three basic steps (in `/etc/ssh/sshd_config`):

1. **Turn off password login** (`PasswordAuthentication no`) — key only. Makes brute-force pointless.
2. **Turn off direct root login** (`PermitRootLogin no`) — a normal user first, then `sudo`. Narrows the
   attack surface and improves the audit trail.
3. After the change, restart sshd with `sudo systemctl restart ssh` (Phase 5!).

> **🔧 See it on your machine** 🔴 — harden SSH (without locking yourself out)
>
> ```
> # in /etc/ssh/sshd_config:
> #   PasswordAuthentication no
> #   PermitRootLogin no
> $ sudo sshd -t                       # 🟢 first validate the config SYNTAX
> $ sudo systemctl reload ssh          # 🟡 apply (does not drop existing sessions)
> ```
>
> **Undo:** This is 🔴 because a wrong `sshd_config` can **lock you out**. Golden rule: before applying the
> change, validate the syntax with `sudo sshd -t`, and test it **while a second SSH session is still
> open** — don't close your current session until you've seen that you can open a new one. If you get
> locked out, the way back: get in via the cloud console / serial console and restore `sshd_config` (the
> network version of Phase 6's recovery logic).

> **💡 Cloud connection — SSM Session Manager: access without ever opening the SSH port:** AWS SSM Session
> Manager gives you shell access to an instance **without ever opening port 22 to the internet**: an agent
> on the instance makes an outbound connection to AWS, and you connect to that agent through the AWS
> console/CLI. Advantages: (1) the inbound SSH port stays closed → attack surface near zero; (2) access is
> authorized via IAM (Phase 9) → no need to distribute/rotate SSH keys; (3) every session is logged →
> auditability. Instead of "harden SSH," "never open SSH at all" is the preferred path in most modern
> setups. Still, knowing SSH is a must — both for legacy systems and for tunneling.

> **🤔 Think 7.3** — You are trying to connect to an EC2 instance with a newly generated key but keep
> getting "Permission denied (publickey)". The `ssh -v` output shows it offers the right key but the
> server refuses it. Name two possible causes: one to do with the **server's** `authorized_keys`/permissions
> (Phase 2), one to do with **your side's** private key permissions (7.3.1). Write the verification command
> for each.
>
> *(Answer: at the end of the phase)*

---
---

# 7.4 Port and Socket State

## 7.4.1 Which service listens on which port: `ss -tulpn` `[application]`

A service being "open to the network" means it is **listening on a port**. A port is a numbered connection
point on a machine (SSH 22, HTTP 80, HTTPS 443, PostgreSQL 5432...). A process (Phase 3!) opens a port and
listens for connections coming to it. The standard tool to see which ports are being listened on and the
process behind them is `ss`:

> **🔧 See it on your machine** 🟢 — read the listening ports and their processes
>
> ```
> $ sudo ss -tulpn
> Netid State  Local Address:Port   Process
> tcp   LISTEN 0.0.0.0:22           users:(("sshd",pid=712,...))
> tcp   LISTEN 127.0.0.1:5432       users:(("postgres",pid=980,...))
> tcp   LISTEN 0.0.0.0:80           users:(("nginx",pid=1120,...))
> ```
>
> Flag meanings: `-t` TCP, `-u` UDP, `-l` listening only, `-p` process name/PID, `-n` show numerically
> without resolving names. The last column ties you straight back to Phase 3: behind every port is a PID.
> When you see "the port is closed" the first question is: is the process up (Phase 5: `systemctl status`,
> Phase 3: `ps`)?

## 7.4.2 "The service is up but I can't connect": 127.0.0.1 vs 0.0.0.0 `[mechanism]`

Now we reach the single most important distinction of this phase — and this is the network version of the
"existing ≠ reachable" gap we promised back in Phase 6. In the output above two services are listening but
at **different addresses**:

- **`0.0.0.0:22` (sshd, nginx)** — `0.0.0.0` means "listen on **all** interfaces"; it accepts connections
  coming over both `lo` (127.0.0.1) and the external interface (`ens5`, 172.31.x.x). So it is **reachable
  from outside** (as long as the firewall/SG allows).
- **`127.0.0.1:5432` (postgres)** — `127.0.0.1` means "listen **only** on loopback, i.e. only from **inside
  the machine itself**." No connection from outside can reach this port — the process is up, the port is
  "open," but only locally.

Here is the number-one cause of the failure "the service is running (`systemctl` green), the port is being
listened on (`ss` shows it), but I can't connect from outside": the service has **bound** to `127.0.0.1`,
not `0.0.0.0`. The fix is to change the bind address in the service's own config (e.g. `listen_addresses`
in PostgreSQL, `--host 0.0.0.0` in many apps). This can also be a deliberate security choice: keeping a DB
only on local on purpose and reaching it via an SSH tunnel (7.3.3) is a common, safe pattern.

> **❓ Question that comes to mind: "`systemctl status` says green (`active running`), so the service must
> be reachable, right?"**
>
> No — these are three separate questions and `systemctl` answers only the first. (1) **Is the process
> running?** → `systemctl status` (Phase 5). (2) **Is it listening on the right address?** → `ss -tulpn`
> (`0.0.0.0` or `127.0.0.1`). (3) **Does the firewall/SG allow it?** → 7.5. "Green" is only (1).
> Reachability requires all three to be green. What separates an engineer from a novice is being able, in
> "green but not working," to eliminate these three layers in order.

## 7.4.3 The gates a connection must pass through `[mechanism]`

Let's line up those three questions into a single mental model. For an inbound connection to reach the
service it must pass through several gates **in order**; if any one is closed, the connection dies
silently:

![Figure 7.1 — The gates a connection must pass through: DNS resolution → Security Group (cloud) → host firewall (ufw) → the service's bind address (0.0.0.0 or 127.0.0.1) → process. If any gate is closed it's "I can't connect"; diagnosis is eliminating the gates in order.](../diagrams/png/lx-7-01-connection-gates.png)

This model is the diagnostic map of the "I can't connect to the service" event. Going from client to
service a connection passes these gates: (0) **does the name resolve** (7.2 — otherwise it never starts);
(1) does the **Security Group** open the port on the cloud side (7.5); (2) does the **host firewall** open
the port on the OS side (7.5); (3) is the service **bound to the right address** (7.4.2 — `0.0.0.0`); (4)
is the **process behind it up** (Phase 3/5). Diagnosis is eliminating these gates from outside in (or
inside out) in order — not panic, order.

---
---

# 7.5 Host Firewall and Security Group — Two Separate Layers

## 7.5.1 Two independent gates: OS firewall and cloud Security Group `[concept]`

Two of the "gates" a connection must pass are firewalls, and these are **two separate layers** — confusing
this distinction is the most common source of confusion in the cloud:

- **Host firewall (OS level)** — a packet filter running **inside** the machine: **ufw** (Ubuntu's simple
  frontend), **firewalld** (the RHEL family), or the underlying **nftables/iptables**. Its rules are
  applied by the machine itself; seen with `sudo ufw status`.
- **Security Group (cloud level)** — a virtual firewall running **outside** the machine, in the cloud
  provider's network layer. Without ever entering the instance, it filters traffic going to/from it on the
  cloud side. Managed from the AWS console, and **invisible** from inside the machine.

The critical point: **both can cut traffic, and both must be open.** For a connection to pass, both the
Security Group and the host firewall must keep that port open — if either closes it, the connection dies.
That's why the answer to "I opened port 80 in the SG but I still can't connect" is often the host firewall
(or vice versa).

> **🔧 See it on your machine** 🟢 — read the host firewall state
>
> ```
> $ sudo ufw status verbose
> Status: active
> To                         Action      From
> --                         ------      ----
> 22/tcp                     ALLOW IN    Anywhere
> 80/tcp                     ALLOW IN    Anywhere
> # 5432 is NOT here → the host firewall is cutting it (even if the SG is open)
> ```
>
> This output shows only the **host** layer. You **cannot see** the Security Group from here — you have to
> check it from the cloud console. In diagnosis handle the two separately: `ufw status` (host) + console/CLI
> (SG).

> **💡 Cloud connection — why two layers (defense in depth):** The two firewall layers are not a redundant
> repetition but a deliberate **defense in depth** design. The Security Group is a broad, network-level
> boundary (which instances can talk on which ports); the host firewall is a second, fine-tuned boundary
> specific to that instance. If one is misconfigured (or an instance ends up in the wrong SG) the other can
> still protect. In Phase 9 you'll see this as a full layered defense together with AppArmor/SELinux
> (application-level enforcement). For now the rule: in a connection failure, check **both firewall
> layers** — don't assume the other because one is "open."

---
---

# 7.6 When This Phase Breaks — Network Failure Signatures

Network failures are the kind that panic a novice engineer the most because the symptom is always the
same: "I can't connect." But almost all of them sit on a link of the **chain of gates** from 7.4.3. The
table below takes the symptom to the right gate:

| Symptom | Likely cause | Where to look | Related section |
|---|---|---|---|
| `ping IP` works, `ping name` doesn't | Name resolution broken (resolv.conf/DNS) | `cat /etc/resolv.conf`, `resolvectl status` | 7.2.2 |
| Service `active (running)` but not reachable from outside | Bound to `127.0.0.1`, not `0.0.0.0` | `ss -tulpn` (Local Address column) | 7.4.2 |
| Port doesn't appear in `ss` at all | Process died / service crashed | `systemctl status`, `ps`, `journalctl -u` | 7.4.1 (Phase 5) |
| Port listened, host firewall open, still nothing | Security Group closes the port (cloud layer) | Cloud console / CLI (SG rules) | 7.5.1 |
| SG open but still can't connect | Host firewall (ufw) cuts the port | `sudo ufw status` | 7.5.1 |
| SSH "Permission denied (publickey)" | Wrong key / `authorized_keys` / permissions | `ssh -v`, on server `ls -l ~/.ssh` | 7.3.1 (Phase 2) |
| SSH "Connection timed out" | Connection never reaches the gate (SG/firewall/wrong IP) | `ssh -v` (where it stuck), SG, IP | 7.5.1 |
| Locked out after SSH change | `sshd_config` wrong / firewall cut SSH | Get in via serial/cloud console, revert | 7.3.4 |

> **The lesson from this table:** "I can't connect" is not a diagnosis but a **starting point**. Every
> network failure is a link of the chain of gates in 7.4.3, and the right reflex is not to panic but to
> **eliminate in order**: (0) does the name resolve (`ping IP` vs `ping name`) → (1/2) are the firewall
> layers open (`ufw` + SG) → (3) is the service bound to the right address (`ss -tulpn`) → (4) is the
> process up (`systemctl status`). Never mix two distinctions: **IP vs name** (network path or resolution)
> and **127.0.0.1 vs 0.0.0.0** (local or open to outside). And remember: if a machine has become
> unreachable (especially from an SSH hardening/firewall mistake), your recovery path is from the same
> family as in Phase 6 — the **serial/cloud console**, because when the network layer is down `ssh` already
> won't help.

---
---

# Phase 7 — Answers to the think questions

## Answer 7.1 — The problem is not in interface/IP/gateway; it's in resolution

If `ping 8.8.8.8` works, it means a packet can leave your machine, reach an IP, and come back — so the
**interface** (a packet can get out), the **IP** (the machine has an address), and the **gateway** (routing
to the outside network works) are all sound. The problem is in **none** of these three. The only difference
in `ping google.com` giving "Name or service not known" is that the second requires a **name→IP
translation**. So the fault is exactly at the name resolution layer. This observation points you straight
to 7.2's file: **`/etc/resolv.conf`** (and `resolvectl status`) — the DNS server is empty, wrong, or
unreachable.
**Related section:** 7.1.1, 7.2.2 · **Continues in:** Answer 7.2.

## Answer 7.2 — A resolution failure; temporary recovery with /etc/hosts

(a) The fault is at the **resolution** layer, not the network path: `ping 10.0.5.20` (IP) works → the
network path to the database (interface/gateway/SG) is sound; but `ping db.prod.internal` (name) breaks →
the name can't be translated to an IP. The app uses the same name, so it can't connect. (b) This single
line in `/etc/hosts` makes the app work temporarily: `10.0.5.20  db.prod.internal` — it maps the name
directly to the IP without ever asking DNS (in 7.2.1, `/etc/hosts` is checked before DNS). (c) It's not a
permanent fix because: if the database's IP changes (it does on cloud failover/replace) `/etc/hosts` keeps
showing the old IP and the failure silently returns; also you'd have to put that line on every machine one
by one. The permanent fix is to repair the actual DNS/`resolv.conf` problem.
**Related section:** 7.2.1-7.2.2 · **Continues in:** the 7.6 failure table (row 1).

## Answer 7.3 — Two sides, two different permission/identity problems

**Server side (Phase 2):** The public key may never have been added to `~/.ssh/authorized_keys` on the
server, or it was added but the permissions are wrong — if `~/.ssh` is not `700` and `authorized_keys` not
`600` (the dir/file is "too open") sshd **silently refuses** the key. Verification: on the server
`ls -ld ~/.ssh` and `ls -l ~/.ssh/authorized_keys` (permissions + whether your public key line is really
there). **Client side (7.3.1):** Your private key may be in too-open permissions (not `600`); SSH finds
this dangerous and refuses to use the key. Verification: `ls -l ~/.ssh/id_ed25519` — if it's not
`-rw-------` (600), `chmod 600 ~/.ssh/id_ed25519`. In both cases `ssh -v` shows which key was offered and
why the server refused it.
**Related section:** 7.3.1 (Phase 2.3) · **Continues in:** the 7.6 failure table (row 6).

---
---

# Phase 7 — Frequently asked questions

**Q1 — Why doesn't `ip addr` show my public IP?** In the cloud, only the **private** IP is visible from
inside the machine; the public IP is mapped to it on a NAT layer outside. You learn the public IP from the
cloud console or the instance metadata, not from inside the machine (7.1.1).

**Q2 — `ping` works but I can't connect to the service — isn't that a contradiction?** No. `ping` (ICMP)
and a TCP connection to a **service port** are different things. `ping` shows the machine **exists** on the
network; but if the service isn't listening on that port, is bound to `127.0.0.1` instead of `0.0.0.0`, or
the firewall/SG closes the port, you can't connect despite the ping. `ping` tests only the outermost of
the chain of gates (7.4.3).

**Q3 — The difference between `127.0.0.1` and `0.0.0.0` in one sentence?** `127.0.0.1` = "listen reachable
only from inside the machine itself" (closed to outside); `0.0.0.0` = "listen on all interfaces" (reachable
from outside if the firewall/SG allows). A service meant to be reached from outside must bind to `0.0.0.0`
(7.4.2).

**Q4 — My SSH key is correct but it's still rejected / asks for a password. Why?** The most common cause is
**permissions**: if the private key isn't `600` and `~/.ssh` isn't `700`, SSH silently refuses for
security. On the server the `authorized_keys` permissions are equally strict. See the exact step with
`ssh -v`, verify permissions with `ls -l` (7.3.1).

**Q5 — How do I see the Security Group from inside the machine?** You can't. The Security Group lives
**outside** the machine, in the cloud network layer, and is invisible to the OS. `ufw`/`nftables` show you
only the **host** firewall. You must check the SG via the cloud console or CLI — and in a connection
failure handle **both** separately (7.5.1).

**Q6 — How do I connect to a DB port without opening it to the outside?** SSH port forwarding (tunneling):
`ssh -L 5432:127.0.0.1:5432 server` carries the remote DB's local port to your own machine; traffic flows
through the encrypted SSH tunnel and the DB port is never opened to the internet. This is the basis of the
bastion/jump host pattern (7.3.3).

**Q7 — Should I use SSH or SSM Session Manager?** If possible, SSM: it never opens port 22 to the internet
(attack surface ~zero), authorizes access via IAM (no key distribution), logs every session. Still know
SSH — you need it for legacy systems and tunneling. In modern setups "never open SSH" is increasingly
preferred over "harden SSH" (7.3.4).

---
---

# Phase 7 — Test yourself

Write your answers on paper, then compare against the answer key. Target: 14+ out of 18.

## Section A — Definition and mechanism

1. Name the three parts of a machine's network identity (interface, IP, gateway) and the command that
   shows each.
2. Why does `ip addr` usually not show the public IP in the cloud? Where does the public IP live?
3. Write the name resolution chain in order: when a name is translated to an IP, which two places are
   checked (in which order)?
4. Which single layer does the observation "ping by IP works, by name doesn't" narrow the fault to, and
   which file do you look at?
5. Why is key-based identity in SSH safer than a password? Where does each of the private and public key
   sit?
6. What does each of the `t`, `u`, `l`, `p`, `n` flags in `ss -tulpn` do?
7. What is the difference between the listen addresses `0.0.0.0:80` and `127.0.0.1:5432` — which is
   reachable from outside?
8. What do we mean by "the host firewall and Security Group are two separate layers"? Which one is
   invisible from inside the machine?

## Section B — Apply and diagnose

9. A service is green in `systemctl status` but not reachable from outside. Write the three separate
   questions (and their commands) you must check in order for reachability.
10. `ping 8.8.8.8` works, `ping github.com` gives "Name or service not known". Which single file do you
    look at and what do you look for there?
11. You can't connect to EC2 with a new key, "Permission denied (publickey)". What's the first command you
    run, and which two permissions (one on your side, one on the server) do you check?
12. In the `ss -tulpn` output your app is listening on `127.0.0.1:8080` but you want it reachable from
    outside. What is the root cause and where is it fixed?
13. You want to connect to a DB port (5432) with a local tool without ever opening it to the internet.
    Which command do you use?
14. You're about to harden `sshd_config`. To avoid locking yourself out, what two things must you do before
    and during applying it?

## Section C — Reasoning and connection

15. In the "I can't connect to the service" event, order the gates a connection must pass through (from
    outside in). Which section does each gate belong to?
16. In Phase 6 you saw the failure "the device exists but is not mounted → not in `df`". What is its exact
    counterpart in this phase? Write the shared lesson of the two in one sentence.
17. A machine became unreachable from an SSH hardening mistake. Why can't you recover it with `ssh`, and
    which recovery logic from Phase 6 do you use the network version of?
18. Name three security advantages of SSM Session Manager over SSH.

---

## Answer key

1. Interface (`ip addr` / `ip link`), IP (`ip addr`), default gateway (`ip route` → `default via`)
   (7.1.1). — 2. Because only the private IP is visible from inside; the public IP is mapped on a NAT layer
   outside — it lives in the cloud console/metadata (7.1.1, Q1). — 3. First `/etc/hosts` (local ledger),
   then the DNS server in `/etc/resolv.conf` (7.2.1). — 4. To the name **resolution** layer;
   `/etc/resolv.conf` (and `resolvectl status`) (7.2.2). — 5. The password never crosses the network and
   can't be brute-forced; the private key stays with you (never shared), the public key sits on the server
   in `authorized_keys` (7.3.1). — 6. `-t` TCP, `-u` UDP, `-l` listening only, `-p` process/PID, `-n`
   numeric (without resolving names) (7.4.1). — 7. `0.0.0.0` listens on all interfaces → reachable from
   outside; `127.0.0.1` only on loopback → only from inside the machine (7.4.2). — 8. The host firewall is
   inside the OS (`ufw`/`nftables`), the SG in the cloud network layer; **the SG is invisible from inside
   the machine**, and both can cut traffic (7.5.1).

9. (1) Is the process running → `systemctl status` (Phase 5); (2) is it listening on the right address →
   `ss -tulpn` (`0.0.0.0` vs `127.0.0.1`); (3) does the firewall/SG allow it → `ufw status` + cloud console
   (7.4.2). — 10. `/etc/resolv.conf`; you look for whether there is a valid/reachable `nameserver` line
   (7.2.2). — 11. First `ssh -v ...` (at which step it was refused); permissions: on your side is
   `~/.ssh/id_*` `600`, on the server is `~/.ssh` `700` / `authorized_keys` `600` and the public key there
   (7.3.1). — 12. The app has bound to `127.0.0.1`; to open it out, set the bind address to `0.0.0.0` in
   the app's config (7.4.2). — 13. `ssh -L 5432:127.0.0.1:5432 server` (SSH tunnel) (7.3.3). — 14. Validate
   the syntax with `sudo sshd -t`; test while **a second open session** is up, don't close the current
   session until you've seen you can open a new one (7.3.4).

15. (0) Does the name resolve (7.2) → (1) is the Security Group open (7.5) → (2) is the host firewall open
    (7.5) → (3) is the service bound to `0.0.0.0` (7.4.2) → (4) is the process up (Phase 3/5) (7.4.3). — 16.
    The exact counterpart: "the service is running / the port is listened on but it's bound to `127.0.0.1`
    or a firewall is closed → not reachable from outside." Shared lesson: a resource **existing** (mounted /
    process listening) and being **reachable** (in df / reachable from outside) are two different things
    (7.4.2, Phase 6.6). — 17. Because when the network/SSH layer is down `ssh` already won't work; you use
    the network version of Phase 6's "get into an unreachable machine via serial/cloud console" recovery
    logic (7.3.4, 7.6). — 18. (1) It never opens port 22 to the internet (attack surface ~zero); (2) it
    authorizes access via IAM (no key distribution/rotation); (3) it logs every session (auditability)
    (7.3.4).

## Scoring

| Correct | What it means |
|---|---|
| 16-18 | You've learned to eliminate "I can't connect" layer by layer. You're ready for Phase 8. |
| 13-15 | Good. Re-read the sections of the questions you missed (especially 7.4.2 and 7.5). |
| 9-12 | The basics are there but fragile. Work on 7.2 (resolution) and 7.4 (bind address). |
| 0-8 | Walk the phase again; run `ip addr`/`ss -tulpn`/`ssh -v` yourself on a test machine. |

Missed question → the section to return to:

| Question | Section |
|---|---|
| 1, 2 | 7.1 Interface and address |
| 3, 4, 10 | 7.2 Name resolution |
| 5, 11, 14 | 7.3 SSH |
| 6, 7, 9, 12 | 7.4 Port and bind address |
| 8, 15 | 7.5 Firewall / SG |
| 13 | 7.3.3 Tunneling |
| 16, 17 | 7.4.3 + the Phase 6 connection |
| 18 | 7.3.4 SSM |

---
---

# Phase 7 — Closing and Bridge to Phase 8

## What you carry from this phase

Phase 7 gave you the **reachability reflex**. Now when you say "I can't connect," instead of panicking you
eliminate the chain of gates in order: does the name resolve (`resolv.conf`) → are the firewall layers open
(host `ufw` + cloud SG) → is the service bound to the right address (`ss -tulpn`: `0.0.0.0` vs `127.0.0.1`)
→ is the process up (`systemctl`). You permanently acquired two distinctions: **IP vs name** (network path
or resolution) and **127.0.0.1 vs 0.0.0.0** (local or open to outside). And you now see SSH not as just a
command but as a service that must be securely configured — and know that in the cloud "never open SSH with
SSM" is often preferred over "harden SSH."

## Where Phase 8 connects to this

In Phase 5 you saw services are managed by systemd, in Phase 7 how those services open to the network. But
how do those services **get onto** the machine? Phase 8 is exactly this: package managers (`apt`),
installing software, and turning your own application into a systemd service (service-ization). The
outermost gate of an "I can't connect to the service" event is "is the service installed and running" —
and how that service got there (a package, your own binary, which version) is Phase 8's subject. The
network (Phase 7) connects the service to the **outside**; packages (Phase 8) bring the service **onto the
machine**. Put together, they complete the whole chain of bringing an application up from scratch in the
cloud and safely opening it to the outside world.

> **🤔 Phase output — ask yourself:** In Phase 7 you separated a service "running" from being "reachable."
> Before moving to Phase 8, think: you installed an app with `apt install` but `systemctl status` says "not
> found." Which distinction from Phase 7 is this one level up from — what step might be missing between
> "being installed" and "running as a service"? (Hint: not every package defines a systemd service
> automatically.)
>
> **🧪 Lab 7 idea (on your own test instance):** (1) Read your network identity with `ip addr` and
> `ip route`, find your private IP. (2) See the listened ports and the processes behind them with
> `ss -tulpn`. (3) Find your DNS server with `cat /etc/resolv.conf`; experience the difference between
> `ping 8.8.8.8` and `ping google.com`. (4) Bind a simple web server (e.g. `python3 -m http.server`) first
> to `127.0.0.1`, then to `0.0.0.0`, and see the difference in the `ss` output and in outside
> reachability. (5) Trace a connection step by step with `ssh -v` — see where the handshake completes.
> These five steps gather this phase's reachability reflex into your hands.

---

> **Navigation:** [◀ Checkpoint Quiz 3](Checkpoint_Quiz_3.md) · **Phase 7** · [Phase 8 — Packages and Service-ization ▶](Phase_8_Packages_and_Service_ization.md)
