# Phase 9 — Firewalls, Filtering and Security

> **Navigation:** [◀ Phase 8 — The Application Layer: HTTP + TLS](Phase_8_HTTP_and_TLS.md) · **Phase 9** · [Checkpoint Quiz 3 ▶](Checkpoint_Quiz_3.md)

---

## Where we come from

At the end of Phase 8 we asked: if a firewall dropped your packet, **why did it not tell you "I refused
it"?** What advantage does staying silent have?

You will see the answer in full in 9.2.3, but the essence is this: **answering is giving information.** A
firewall that stays silent does not even tell the other side "there is a machine here but this port is
closed" — the person scanning cannot even work out whether anything exists at that address. This is not a
convenience but a deliberate choice. And it is exactly what makes your diagnostic life harder.

What you brought with you:

- **The 4-tuple and the socket** (1.4.2) — the ancestor of the 5-tuple here.
- **TCP state information** (5.2, 5.6) — this is exactly what a stateful firewall remembers.
- **The "the SYN goes out, no answer" signature** (5.2.2) — the most frequent cause of it is in this phase.
- **NAT's translation table** (7.1.2) — the same stateful tracking idea, for a different purpose.
- **ICMP and Fragmentation Needed** (4.4.2, 5.7.3) — the reason you cannot turn ICMP off completely.

## The question of this phase

> *"Until now packets were dropped because something was **broken**. But when nothing is broken, why does a
> packet get dropped?"*

The answer: because somebody wanted it that way. A rule was written and that rule did not accept your
packet.

This phase is about the network's **deliberate refusal**. And it has two faces: on the security side it
answers "what should I let in", and on the diagnostic side "who dropped my packet, and where". The second
is the class of failure you will meet most often in cloud engineering — and it is almost always **silent**.

The cloud part of this phase is especially important: **the difference between a Security Group and a
NACL** rests directly on the TCP state information you learned in Phase 5. Once you make that connection,
you will also have understood the cause of the most common configuration mistake in the cloud.

---

## By the end of this phase

- You will be able to explain the difference between stateful and stateless filtering and **why** stateful
  does not need a rule for return traffic
- You will know what the 5-tuple is and which fields a firewall decision looks at
- You will be able to explain why the allow/deny order and the default policy (default deny) are critical
- You will be able to tell DROP and REJECT apart and know **what they mean in diagnosis**
- You will be able to diagnose the "ping works but the application cannot connect" symptom in seconds
- You will be able to explain — through the MTU black hole connection — why turning ICMP off completely is a
  mistake
- **Cloud:** You will be able to explain clearly the difference between a Security Group (stateful) and a
  NACL (stateless), which of them needs a rule for return traffic and **why**
- You will be able to apply the layered-defence and least-privilege reflexes to your own rules

---

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 9.1 | Stateful vs stateless | `[mechanism]` | **The heart of the phase** — to remember or not to remember |
| 9.2 | Packet filtering and the 5-tuple | `[concept]` | How the decision is made |
| 9.3 | ICMP policy | `[concept]` | The price of turning it off |
| 9.4 | Cloud: SG vs NACL | `[concept]` | The most common mistake in the cloud |
| 9.5 | Security reflexes | `[concept]` | What to think about while writing rules |
| 9.6 | When this phase breaks | — | The signatures of a silent drop |

> **How to work through this phase:** Most of this phase's commands are 🟢 (`sudo ufw status`, `sudo iptables
> -L -n` — they only read), but **adding** a rule is 🟡 and you must be careful: writing the wrong rule on a
> remote machine can **cut off your own SSH connection** and you cannot get back in. The golden rule: before
> changing a firewall rule on a remote machine, open a second session and do not close it — if something
> goes wrong you fix it from that session. In this phase the theory matters more than memorising commands:
> in particular, do not move on to 9.4 before the stateful/stateless distinction in 9.1 has fully settled,
> because the cloud part is built directly on top of it.

---
---

# 9.1 Stateful vs Stateless

## 9.1.1 The source of the problem: traffic goes both ways `[mechanism]`

Think about this: your server runs `curl https://example.com` out to the internet. Allowing the outgoing
packet is easy — you say "allow outbound".

But **the answer is going to come back.** And from your side that answer is **a packet arriving from
outside**. If your firewall says "block everything coming from outside", it blocks the answer to your own
request too.

There are two solutions, and this whole phase is built on the difference between them.

## 9.1.2 Stateless: every packet is alone `[mechanism]`

A **stateless** firewall evaluates every packet **on its own**. It does not remember the past, it does not
know which connection the packet belongs to. The only thing it has is that packet's headers at that moment.

The consequence: **you have to write a separate rule for the return traffic.**

```
Rule 1 (outbound):  10.0.1.50 → any:443   ALLOW
Rule 2 (inbound):   any:443 → 10.0.1.50   ALLOW    ← without this the answer cannot get in
```

And there is a subtlety in the second rule: the **destination port** of the returning packet is not 443, it
is your **ephemeral source port** (1.4.1). So the real rule has to be:

```
Rule 2 (inbound): any:443 → 10.0.1.50:32768-60999   ALLOW
```

You have to open the whole ephemeral port range, because you cannot know in advance which port will be
used. This is the natural weakness of stateless filtering: **you inevitably write rules that are wider than
they need to be.**

## 9.1.3 Stateful: remember the connection `[mechanism]`

A **stateful** firewall keeps a **connection tracking table** — the same idea as the NAT table in Phase 7
(7.1.2), for a different purpose. When a connection is started from the inside it creates a row:

| Source | Destination | Protocol | State |
|---|---|---|---|
| 10.0.1.50:51234 | 93.184.216.34:443 | TCP | ESTABLISHED |

When the answer arrives the firewall looks at the table: *"This packet is part of a connection I allowed."*
And it lets it through **without needing any extra rule**.

That is:

> **In a stateful firewall you only write the direction in which the connection is started. The return
> traffic passes automatically.**

The reason it can do this is the TCP state information you learned in Phase 5: a SYN is the **beginning** of
a connection, the SYN-ACK is the answer, the ACK is its establishment (5.2.1). The firewall reads those
flags and knows which stage the connection is at. Because there is no state information in UDP (5.1.1),
firewalls keep an artificial timeout: if they see no traffic for a while they delete the row.

And two practical consequences follow from this:

1. **The table has a capacity** — a large number of simultaneous connections can fill it (the same limit as
   in 7.2.1).
2. **The rows have a timeout** — the record of a connection that sits idle for a long time is deleted and
   the next packet is refused. This is the second cause of the "an idle SSH session dies" problem in 5.6.1.

| | **Stateless** | **Stateful** |
|---|---|---|
| Does it remember | ❌ | ✅ (a connection table) |
| A rule for return traffic | **Required** | Not required |
| Number of rules | More, and wider | Fewer, and narrower |
| Resource usage | Low | Memory is needed for the table |
| The cloud equivalent | **NACL** | **Security Group** |

Keep that last row in mind — the whole of 9.4 is built on the difference between those two cells.

![Figure 9.1 — A comparison of stateful and stateless filtering: the stateful firewall records the outgoing connection in its table and lets the return traffic through without a rule; the stateless firewall evaluates every packet one by one, so a separate rule is needed for the return direction.](../diagrams/png/nw-9-01-stateful-vs-stateless.png)

In the figure the upper flow shows stateful and the lower one stateless behaviour. On the stateful side
watch how the table box in the middle matches the returning packet **without it entering the rule check at
all**; on the stateless side, how the returning packet has to pass through a second list of rules.

> **🤔 Think 9.1** — A server has a stateless firewall and the rule is: "allow outbound to 443, block
> everything inbound." The server runs `curl https://example.com`. (a) What happens? (b) Which rule do you
> have to add for it to work — watch the port number. (c) What would change if the same scenario happened on
> a stateful firewall?
>
> *(Answer: at the end of the phase)*

---
---

# 9.2 Packet Filtering and the 5-Tuple

## 9.2.1 Which fields the decision looks at `[concept]`

A firewall rule decides by looking at five fields in the packet's headers. This is called the **5-tuple**:

| Field | From which layer | Example |
|---|---|---|
| Source IP | L3 (Phases 1, 4) | 10.0.1.50 |
| Destination IP | L3 | 10.0.2.20 |
| Source port | L4 (1.4, Phase 5) | 51234 |
| Destination port | L4 | 5432 |
| Protocol | L3/L4 | TCP |

This is the **4-tuple** you learned in 1.4.2 with the protocol added — so you already knew it.

A typical rule reads like this:

```
ALLOW  TCP  source 10.0.1.0/24  →  destination 10.0.2.20  port 5432
       └── "allow a Postgres connection from the app subnet to the database"
```

Notice that a **CIDR block** can be written in the source field (Phase 2) — rules are written with ranges,
not with addresses one by one. In the cloud this goes further still: as a source you can write another
**security group** (9.4.1).

## 9.2.2 Order and the default policy `[concept]`

Two rules determine a firewall's behaviour completely:

**1. Rules are evaluated in order and usually the first match wins.** So if you put a broad DENY rule
**above** a narrow ALLOW rule, the ALLOW never runs. Rule order is not a detail, it is the logic itself.

**2. If no rule matches, the default policy applies.** And the right default is **always DENY**:

> **Default deny:** *"Everything that is not explicitly allowed is forbidden."*

The alternative (default allow) means: *"I blocked every danger I could think of, one by one."* A single
thing you did not think of invalidates the whole policy. That is why no serious system is built with default
allow. In AWS security groups already come with this model: for inbound traffic **nothing is allowed**,
until you add it.

## 9.2.3 DROP and REJECT: silence is a choice `[application]`

When a firewall does not want a packet it can do two things — and the answer to Phase 8's closing question
is here:

**DROP** — it throws the packet away **silently**. No answer at all.
The symptom on the sending side: **timeout.** The client waits for the answer, waits, and the time runs out
(5.2.2).

**REJECT** — it refuses the packet **and says so**: an RST for TCP, or an ICMP "port unreachable".
The symptom on the sending side: **an immediate error** — "connection refused" (5.2.2).

Why is DROP preferred? Because **answering is giving information.** If an attacker is scanning thousands of
addresses, every address that gives a REJECT tells them *"there is a machine here"*. The addresses that give
a DROP are silent — you cannot tell whether there is a machine there or whether the address is empty. The
scan slows down and becomes uncertain.

The price is **your diagnostic life**: seen from outside, "the firewall blocked it", "there is no machine"
and "there is no route" all look the same. They all give a timeout.

And from this comes the most practical rule of this phase:

> **Timeout = most likely a firewall (DROP). Connection refused = not a firewall, the service is not
> listening.**

Combine this with the table in 5.2.2: if **no answer comes at all** you look at the network/security side;
if **an RST comes** you look at the application side. A single observation separates two different teams.

> **🔧 See it on your machine** 🟢 — read the local firewall rules
>
> ```
> $ sudo ufw status verbose
> Status: active
> Default: deny (incoming), allow (outgoing), disabled (routed)
>          └── default deny: the right default (9.2.2)
> To                         Action      From
> --                         ------      ----
> 22/tcp                     ALLOW IN    Anywhere
> 443/tcp                    ALLOW IN    Anywhere
>
> $ sudo iptables -L -n -v | head -12
> Chain INPUT (policy DROP 0 packets, 0 bytes)
>  pkts bytes target  prot opt source        destination
>  1847  142K ACCEPT  all  --  0.0.0.0/0     0.0.0.0/0    ctstate RELATED,ESTABLISHED
>     4   240 ACCEPT  tcp  --  0.0.0.0/0     0.0.0.0/0    tcp dpt:22
> ```
>
> `ufw` is an easy-to-read front end; `iptables -L -n -v` shows what is underneath. In the second output
> notice two things: **`policy DROP`** is the default policy (9.2.2), and the **`ctstate
> RELATED,ESTABLISHED`** on the first line is stateful tracking itself (9.1.3) — it means "let through
> packets that are part of an already established connection". Without that single line no return traffic
> could get in at all. The `pkts` column says how many times the rule has matched; it is the fastest way to
> work out whether a rule is working.

> **⚠️ Common misconception: "If ping works there is no connectivity problem."**
>
> No — and this is the misconception you will meet most often in this phase. `ping` uses **ICMP**; your
> application uses **TCP** and goes to a specific **port** (9.2.1). Firewall rules are based on protocol and
> port: allowing ICMP while blocking TCP 5432 is perfectly possible and very common. The sentence "ping
> works but the application cannot connect" almost always means **a missing port rule**. The right test is
> not ping but the real protocol and port: `curl -v telnet://host:5432`, or `nc -zv host 5432`, or looking
> with `ss`/`tcpdump`. Ping is a reachability test, **not a permission test** (you saw the same warning in
> 4.4.2).

> **🤔 Think 9.2** — The application server cannot connect to the database. `ping db.internal` works, `nc -zv
> db.internal 5432` waits 30 seconds and times out. (a) What is your diagnosis? (b) How would the diagnosis
> change if you got "connection refused" instead of a timeout? (c) Which team do you call in each of those
> two cases?
>
> *(Answer: at the end of the phase)*

---
---

# 9.3 ICMP Policy

## 9.3.1 "We turned ICMP off for security" `[concept]`

You will hear this sentence a lot in the field and most of the time it will have been applied wrongly.

The justification for blocking ICMP entirely is this: a machine that answers ping announces its existence to
scanners (the logic of 9.2.3). That justification is **reasonable** — but the solution is not "turn off all
ICMP".

Because ICMP is not only ping. Remember the message types you saw in 4.4.2: Destination Unreachable, Time
Exceeded, and the most critical one, **Fragmentation Needed (type 3, code 4)**.

What happens if you block that last message? The whole of 5.7.3: **an MTU black hole.** Path MTU discovery
cannot work, large packets disappear silently, and the symptom is *"small packets get through, large packets
do not"*. When you set up a VPN (7.4.2) this problem becomes almost inevitable.

The right policy is this:

| ICMP type | What to do | Why |
|---|---|---|
| Echo Request (ping, type 8) | **Can be blocked** from outside, keep it open on the internal network | Valuable for diagnosis, gives information outwards |
| Echo Reply (type 0) | The answer to your outgoing pings — must be open | Otherwise your own ping does not work |
| **Fragmentation Needed (3/4)** | **ALWAYS OPEN** | Path MTU discovery depends on it (5.7.3) |
| Time Exceeded (type 11) | Useful to keep open | `traceroute`/`mtr` rely on it (4.5.1) |
| Destination Unreachable (others) | Useful to keep open | Fast error reporting |

The one-sentence rule:

> **You may restrict ping, but never block Fragmentation Needed.**

> **💡 Cloud connection — the ICMP trap in AWS:** Allowing ICMP in a security group requires **a separate
> rule**; apart from "allow all traffic", no TCP/UDP rule covers ICMP. That is why "I cannot ping the
> instance but SSH works" is a very common situation and usually **not a fault** — there is simply no ICMP
> rule. The genuinely dangerous case is this: if ICMP type 3 code 4 is blocked on traffic passing over a VPN
> or a Transit Gateway, an MTU black hole forms (7.4.2) and the symptom points somewhere completely
> different ("the database queries are freezing"). That is why adding **ICMP Destination Unreachable** to
> the SG/NACL rules on inter-VPC and VPN traffic is standard practice. On the diagnostic side, know this: in
> AWS there is no message that tells you directly why a packet was dropped — you read that from **VPC Flow
> Logs** (10.4), as a REJECT line.

---
---

# 9.4 Cloud: Security Group vs NACL

This section is the source of the most common configuration mistake in cloud networking. If you understood
9.1, this section will fall into place by itself.

## 9.4.1 Two layers, two behaviours `[concept]`

In AWS **two** filters are applied to a packet and the two work in fundamentally different ways:

| | **Security Group (SG)** | **Network ACL (NACL)** |
|---|---|---|
| Where it is applied | The **ENI** (the instance's network interface) | The **subnet** boundary |
| Behaviour | **Stateful** | **Stateless** |
| A rule for return traffic | **Not required** | **Required** |
| Rule types | ALLOW only | ALLOW **and** DENY |
| Evaluation | All rules together (no order) | **By rule number**, first match wins |
| Default | Inbound: nothing; outbound: everything | The default NACL: allow everything |
| As a source | A CIDR **or another SG** | CIDR only |

The two most critical rows are the bold ones. And you already know why:

- **An SG is stateful** (9.1.3) — it keeps a connection table. If you allowed the request coming in to
  `443`, **you do not have to do anything extra** for the answer to go back out from the ephemeral port.
- **A NACL is stateless** (9.1.2) — it evaluates every packet one by one. Even if you allow the incoming
  request, you have to write **a separate outbound rule for the returning answer**.

And in a NACL the port range of that outbound rule is not the service's port but the **ephemeral range**
(1.4.1):

```
NACL Inbound  100: ALLOW TCP 443 from 0.0.0.0/0          ← the request comes in
NACL Outbound 100: ALLOW TCP 1024-65535 to 0.0.0.0/0     ← the answer goes back (ephemeral!)
                                └── forget this and the connection is made but the answer cannot leave
```

This is the most frequent NACL mistake in the cloud: the inbound rule is written, the outbound ephemeral
rule is forgotten, and the result is *"the connection seems to be made but nothing comes back"*.

The SG's second special ability is this: **as a source you can write another SG.** For example you tell the
database SG "source = app-sg"; from then on which IP it comes from does not matter, every instance carrying
that SG passes. When instances die and are reborn and their IPs change, the rule does not break — this is
the preferred approach in the cloud.

## 9.4.2 Which one, when `[concept]`

In practice the usage is this:

- **The SG = your main tool.** You do day-to-day access control here: which service accepts connections from
  whom. It is fine-grained, readable, and because it is stateful the risk of making a mistake is low.
- **The NACL = a coarse extra layer.** Broad rules for a whole subnet: blocking an IP block entirely, for
  instance. It is the only layer that can write DENY (there is no DENY in an SG) — so it is used when you
  need "I definitely do not want that address range".

To reach an instance a packet has to pass through **both of them**. If either one blocks, the packet is
dropped — and both of them drop silently (9.2.3), so the symptom is a **timeout**.

And that is why the checklist for an "I cannot connect" problem in the cloud is this:

1. **Route table** — is there a path (4.2, 7.3.1)
2. **NACL** — both the inbound **and the outbound** rule (stateless!)
3. **SG** — the target instance's SG and, if needed, the source's
4. **The operating system's firewall** — the instance's own `ufw`/`iptables` (it can block even when the
   cloud rules let it through)
5. **Is the service listening** — `ss -tulpn`, and is it on `0.0.0.0` or only `127.0.0.1` (1.4.2)

Do not skip the fifth item: after hours of picking through the cloud rules, discovering that the problem was
really the service listening only on localhost is a frequently lived story.

> **🤔 Think 9.3** — In a subnet the NACL rules are these: Inbound `ALLOW TCP 443 from 0.0.0.0/0`, Outbound
> `ALLOW TCP 443 to 0.0.0.0/0`. The SG allows 443. HTTPS requests come in from outside but none of them get
> an answer. (a) What is the problem? (b) Why is there no such problem in the SG? (c) What should the right
> NACL outbound rule be?
>
> *(Answer: at the end of the phase)*

---
---

# 9.5 Security Reflexes

This section teaches habits, not commands. What should go through your mind while writing rules.

## 9.5.1 Least privilege `[concept]`

As you write each rule, ask: **"Does it have to be this wide?"**

```
❌  ALLOW TCP 0-65535 from 0.0.0.0/0      "everything, from everyone"
🟡  ALLOW TCP 5432    from 0.0.0.0/0      "only Postgres, but from everyone"
✅  ALLOW TCP 5432    from app-sg         "only Postgres, only from the application tier"
```

All three "work". But the first opens a single door to the whole internet; the last limits an attacker's
lateral movement even if one instance is taken over.

Avoid two patterns in particular: opening **SSH (22) to `0.0.0.0/0`** — use a bastion, a VPN or SSM Session
Manager instead (Phase 7 Q5) — and opening **database ports to the internet**.

## 9.5.2 Layered defence `[concept]`

Do not trust a single layer. The layers applied to a packet in the cloud: route table → NACL → SG → the OS
firewall → the application's own authorisation. Each one is independent.

What this means in practice: **being behind NAT does not protect you** (the box in 7.1.2), **being in a
private subnet does not protect you** (another instance in the same VPC can still reach you), and **you
cannot skip the application's authentication just because the SG is set up right.** Each layer does its own
job.

## 9.5.3 Do not lock yourself out while making changes `[application]`

A very practical reflex: while changing a firewall rule on a remote machine you can **cut off your own
access**. And if you cannot get into that machine any other way, there is no way back.

Three ways to protect yourself:

1. **Keep a second session open.** Make the change from one session and do not close the other. If your
   connection drops you fix it from the other one.
2. **Write the SSH rule first, then turn on default deny.** Do it in the reverse order and you lock yourself
   out.
3. **Prepare a rollback plan.** In the cloud this is easy (undo the rule from the console); on a physical
   machine it can be painful.

> **🔧 See it on your machine** 🟡 — a safe rule experiment
>
> ```
> $ sudo ufw status numbered
> Status: active
>      To          Action      From
>      --          ------      ----
> [ 1] 22/tcp      ALLOW IN    Anywhere
>
> $ sudo ufw allow 8080/tcp          # 🟡 add a new rule
> Rule added
>
> $ sudo ufw status numbered | grep 8080
> [ 2] 8080/tcp    ALLOW IN    Anywhere
>
> $ sudo ufw delete 2                # undo it
> ```
>
> *(Undo: `sudo ufw delete <number>` — learn the number with `ufw status numbered`. List the numbers again
> before deleting a rule, because when one rule is deleted the numbers of the following ones shift.)*
> Do this experiment **on your own local machine**, not on a remote server. And never delete rule number 22
> — that is your way back in. Before trying it, read the three protections in 9.5.3 once more.

> **🤔 Think 9.4** — A team wrote `ALLOW TCP 0-65535 from 0.0.0.0/0` into every instance's SG and says
> "they are in a private subnet anyway, they cannot be reached from the internet". (a) Where is this
> reasoning wrong? (b) Describe a concrete attack scenario. (c) What would the right configuration be?
>
> *(Answer: at the end of the phase)*

---
---

# 9.6 When This Phase Breaks — The Signatures of a Silent Drop

All of this phase's failures share one property: **the firewall tells you nothing.** A dropped packet leaves
no trace, produces no message, writes nothing to a log (a log you can see). So diagnosis is done by working
backwards from the symptom.

| Symptom | Likely cause | Verify | Related section |
|---|---|---|---|
| The connection **times out** | A firewall is **dropping** it (SG/NACL/OS) | Follow the SYN with `tcpdump`, Flow Logs | 9.2.3 |
| **Connection refused** | Not a firewall — the service is not listening | `ss -tulpn` (on the target) | 9.2.3 |
| Ping ✓ but the application ✗ | **A missing port rule** (ICMP open, TCP closed) | `nc -zv host <port>` | the box in 9.2.3 |
| The connection is made but no answer comes | The **NACL outbound ephemeral** rule is missing | The NACL outbound rules | 9.4.1 |
| Outbound traffic does not work (stateless FW) | The return rule was not written | The inbound rule + the ephemeral range | 9.1.2 |
| The connection drops after a while | **The connection table's timeout** | Add keepalive | 9.1.3, 5.6.1 |
| Random errors under load | The connection table is full | The number of simultaneous connections | 9.1.3 |
| I wrote a rule but it does not work | **Order** — there is a broad DENY above it | `iptables -L -n -v`, the `pkts` column | 9.2.2 |
| Large packets do not get through | ICMP 3/4 is blocked → an **MTU black hole** | `ping -M do -s 1472` | 9.3.1, 5.7.3 |
| The instance cannot be pinged but SSH ✓ | **Normal** — there is no ICMP rule in the SG | Look at the SG rules | the box in 9.3.1 |
| The cloud rules are right but it still will not connect | **The OS firewall**, or the service is listening on `127.0.0.1` | `ufw status`, `ss -tulpn` | 9.4.2 |
| IPv4 ✓ IPv6 ✗ | The `::/0` rule is missing | The SG/NACL IPv6 rules | 7.5.2 |

> **The lesson from this table:** All firewall diagnosis begins with **a single distinction**: **is it a
> timeout or a refused?** A timeout says the packet was dropped silently (9.2.3) — your suspicion is a
> firewall, a route, or an unreachable host. A refused proves the packet **reached** the target and that the
> machine answered — that is, the firewall has been passed and the problem is in the service. Those two words
> point you at two different teams. The second reflex: **do not trust ping.** Allowing ICMP while the TCP
> port is closed is entirely normal; always test with the real protocol and port (the box in 9.2.3). In the
> cloud, follow an ordered list — route table, NACL (**both directions**), SG, the OS firewall, the service's
> listening address (9.4.2). The two items most often skipped in that list: the NACL's **outbound ephemeral**
> rule and the instance's **own** firewall. And finally, in the cloud you do not have to guess why a packet
> was dropped: **VPC Flow Logs** keeps an ACCEPT/REJECT record and in the next phase (10.4) you will learn to
> read it.

---
---

# Phase 9 — Answers to the Think questions

## Answer 9.1 — Forgetting the return

**(a)** **The connection is not established.** The outgoing SYN packet leaves, reaches the server, the server
sends a SYN-ACK — but because that answer is **a packet coming from outside** it hits the "block everything
inbound" rule and is dropped. The client never sees the answer, retries, and finally gives a **timeout**
(5.2.2, 9.2.3).

**(b)** A rule is needed in the inbound direction, but be careful: the returning packet's **destination port
is not 443** — it is your **ephemeral source port** (1.4.1). The right rule:

```
ALLOW TCP  source <any>:443  →  destination 10.0.1.50:32768-60999
```

So the source port is 443 and the destination port is the ephemeral range. Because you cannot know in advance
which port will be used, you have to open **the whole range** — that is the natural weakness of stateless
filtering (9.1.2).

**(c)** On a stateful firewall **you would add nothing.** When the outgoing connection is made the firewall
writes it into its connection table; when the returning packet arrives a match is found in the table and the
packet passes **without entering the rule check at all** (9.1.3). The one-line difference: in stateful you
only write the direction in which the connection is **started**.

**Related section:** 9.1.2-9.1.3 · **Next:** 9.4.1 (NACL vs SG), Phase 5.2.1 (the handshake)

## Answer 9.2 — Ping is not a permission test

**(a)** **A missing port rule** (the box in 9.2.3). `ping` uses ICMP and it has been allowed; the application
goes to **TCP 5432** and that has been blocked. Getting a timeout says the packet was **dropped silently**
(DROP, 9.2.3) — so there is a firewall in the way: a security group, a NACL, or the database machine's own
`iptables`.

**(b)** **If you had got "connection refused" the diagnosis would change completely.** Refused means an
**RST** (5.2.2), and for you to receive an RST the packet must have **reached** the target machine — that is,
the firewalls have been passed. The problem is not in the network: the database service is **not running**, is
listening on the wrong port, or is bound only to `127.0.0.1` (1.4.2). Check: `ss -tulpn | grep 5432` on the
target.

**(c)** **Timeout → the network/security team** (or, in the cloud, the owner of the SG/NACL): a rule is
missing. **Refused → the application / database team**: the service is not up or is misconfigured. That
one-word difference lets you call the right team on the first try — and it is the most practical gain of this
phase.

**Related section:** 9.2.3 · **Next:** Phase 10.3 (the diagnostic flow), Phase 10.4 (Flow Logs)

## Answer 9.3 — The forgotten ephemeral range

**(a)** **The NACL's outbound rule was written for the wrong port.** The **destination** port of an incoming
HTTPS request is 443 — that part is right. But the server's **answer** goes from source port 443 to the
client's **ephemeral port** (1.4.1). Because the outbound rule only allows "to port 443", the answer packets
whose destination port is 51234 **do not match and are dropped**. The result: the connection is attempted, no
answer comes back, the client times out.

**(b)** Because **an SG is stateful** (9.1.3, 9.4.1). When it allows the inbound 443 request it writes the
connection into its table; when the answer packet arrives it says "this is part of a connection I allowed"
and lets it through without looking at the outbound rules at all. A NACL is **stateless** — it evaluates
every packet one by one, with no memory (9.1.2), and the answer has to satisfy the rules on its own.

**(c)** The right thing is to open the **ephemeral range** the answer will go to:

```
NACL Outbound 100: ALLOW TCP 1024-65535 to 0.0.0.0/0
```

(The range AWS recommends is 1024–65535; Linux's own default is 32768–60999, but because clients come from
different systems the wide range is used, 1.4.1.) This is the price of using a NACL: because it is stateless
you inevitably write broad rules — and that is exactly why the real access control is done in the SG and the
NACL stays a coarse extra layer (9.4.2).

**Related section:** 9.4.1 · **Next:** Phase 10.4 (verifying with Flow Logs), Phase 11.5 (the SG/NACL mapping)

## Answer 9.4 — A "private subnet" is not a firewall

**(a)** The reasoning is **trusting a single layer** (9.5.2) and it has two errors. First: being in a private
subnet blocks connections coming **from the internet** (7.3.2) — but it does not block connections coming
**from inside the VPC**. Every instance in the same VPC can reach these instances **on all ports**. Second:
the machines in a private subnet **can go out** (over the NAT Gateway, 7.3.3), so a compromised machine can
connect to a command server and NAT does not block that at all (the box in 7.1.2).

**(b)** The scenario: a web server (or a bastion) in the public subnet is taken over through an application
vulnerability. The attacker is now **inside the VPC**. Because the SGs say "allow everything", from there they
can freely reach all ports of all the private instances: they connect directly to the database, scan the
internal services, move laterally. With the right SGs this movement would have been blocked **at every step**
— the web server's SG would not have been able to reach the database.

**(c)** With **least privilege** (9.5.1), layer by layer: the database SG only `ALLOW TCP 5432 from app-sg`;
the application SG only `ALLOW TCP 8080 from alb-sg`; the ALB SG only `ALLOW TCP 443 from 0.0.0.0/0`. Using an
**SG reference** as the source is doubly valuable here (9.4.1): the rule does not break when IPs change, and
access permission is tied to the machine's **role**. For management access, instead of opening port 22 to the
internet, SSM Session Manager or a bastion is used (9.5.1).

**Related section:** 9.5.1-9.5.2 · **Next:** Phase 11.5 (SG design), Phase 7.3 (the private subnet)

---
---

# Phase 9 — Frequently asked questions

**Q1 — Should I use DROP or REJECT?** On the outward-facing surface, **DROP** (it gives no information and
slows scanning down, 9.2.3). On the internal network **REJECT** is usually better: applications get an
immediate error instead of waiting 30 seconds for a timeout, and diagnosis becomes much easier. Cloud SGs and
NACLs always behave like DROP — there is no REJECT option.

**Q2 — Why is there no DENY rule in an SG?** Because an SG works with **default deny** (9.2.2): everything
you did not write is already forbidden, so there is no need for DENY and the rule logic is simpler (order
becomes irrelevant too — all the rules are evaluated together). If you need to block a specific source
explicitly you use a **NACL**; it is the only layer that can write DENY (9.4.2).

**Q3 — If both an SG and a NACL exist, which one wins?** Both are applied and the packet has to pass
**through both**. If either blocks, the packet is dropped. The order: for inbound traffic the NACL first (the
subnet boundary), then the SG (the ENI); for outbound traffic the reverse (9.4.1).

**Q4 — I wrote a rule but it does not work, where do I look?** Three things: (i) **Order** — there may be a
broad DENY rule above it (9.2.2); the **`pkts`** column in `iptables -L -n -v` output shows that the rule has
never matched. (ii) **Direction** — if you are on a stateless layer you may have forgotten the return rule
(9.1.2, 9.4.1). (iii) **The wrong layer** — the cloud rules may be right while the instance's own `ufw` is
blocking (9.4.2).

**Q5 — Why is allowing ping while blocking TCP so common?** Because they are **different protocols** (9.2.1)
and rules are written per protocol. Allowing ICMP is considered harmless (on an internal network it is useful
for diagnosis), but TCP ports are opened one by one. That is why "ping works" never means "the application
can connect".

**Q6 — Why is turning off all ICMP dangerous?** Because ICMP is not only ping. If you block the
**Fragmentation Needed (type 3 code 4)** message, path MTU discovery cannot work and an **MTU black hole**
forms (9.3.1, 5.7.3): small packets get through, large packets disappear silently. Behind a VPN/tunnel this
is almost inevitable (7.4.2).

**Q7 — Do firewall rules affect connection speed?** Unless the number of rules is very high it is
negligible. But on a stateful firewall the real limit is **the capacity of the connection table** (9.1.3):
when the table fills up new connections cannot be made and the symptom is "random errors under load" (the
same class of problem as in 7.2.1). This is a **concurrency** limit, not a speed limit.

---
---

# Phase 9 — Test yourself

Write your answers on paper, then compare them with the key. Target: 14+ out of 18.

## Part A — Definitions and mechanisms

1. What is the fundamental difference between a stateful and a stateless firewall?
2. How many rules does an outgoing HTTPS connection need on a stateless firewall? Write their ports.
3. What does a stateful firewall remember? From which phase do you know that information?
4. Which five fields make up the 5-tuple?
5. What does "default deny" mean and why is it the right default?
6. What is the difference between DROP and REJECT? What is each one's symptom on the client side?
7. Which ICMP type must never be blocked, and why?
8. Write three differences between an SG and a NACL.

## Part B — Apply and diagnose

9. The connection times out. What is your first suspicion?
10. You are getting "connection refused". Does suspecting a firewall make sense — why?
11. Ping works but the application cannot connect. Diagnosis and the verification command?
12. You allowed inbound 443 in the NACL, the connection is made but no answer comes. What is missing?
13. Write, in order, the five things you would check for an "I cannot connect" problem in the cloud.
14. How do you work out that an `iptables` rule you wrote has never run?

## Part C — Reasoning and connections

15. Why does a firewall prefer DROP over REJECT? What does that cost you?
16. Why do you have to open the ephemeral port range for return traffic in a NACL? Why not in an SG?
17. Explain the two errors in the sentence "we are in a private subnet, leaving the SG wide is fine".
18. What completely unrelated-looking failure does a firewall that blocks all ICMP lead to? Tell the chain.

---

## Answer key

1. **Stateful remembers** the connection state (a connection table) and lets return traffic through
   automatically; **stateless** evaluates every packet on its own and has no memory (9.1.2-9.1.3). — 2. **Two
   rules:** outbound `→ destination:443`, inbound `source:443 → your own ephemeral port (32768-60999)`. The
   returning packet's destination port is **not** 443 (9.1.2). — 3. Open connections: source/destination
   IP+port, the protocol and the **TCP state**. That information comes from Phase 5 — the SYN/SYN-ACK/ACK
   flags say which stage the connection is at (5.2.1, 9.1.3). — 4. Source IP, destination IP, source port,
   destination port, protocol (9.2.1). — 5. "Everything that is not explicitly allowed is forbidden." It is
   right because the alternative requires blocking **every danger you can think of** one by one, and a single
   thing you did not think of invalidates the policy (9.2.2). — 6. **DROP** throws it away silently → a
   **timeout** on the client. **REJECT** answers (RST/ICMP) → **connection refused** on the client (9.2.3). —
   7. **Fragmentation Needed (type 3, code 4)** — path MTU discovery relies on it; if it is blocked an **MTU
   black hole** forms (9.3.1, 5.7.3). — 8. (i) An SG is **stateful**, a NACL is **stateless**; (ii) an SG is
   applied at the ENI, a NACL at the **subnet**; (iii) an SG has ALLOW only, a NACL has ALLOW **and** DENY;
   (and in an SG another SG can be written as the source) (9.4.1).

9. **A firewall dropping the packet silently (DROP)** — or a missing route / an unreachable host. What they
   have in common: no answer comes back at all (9.2.3). — 10. **No, it does not make sense.** Refused means an
   **RST**; for you to receive an RST the packet must have **reached** the target — so the firewall has been
   passed. The problem is in the service: it is not listening, or it is bound to the wrong interface (9.2.3,
   1.4.2). — 11. **A missing port rule**; ping is ICMP, the application is TCP, and rules are per protocol.
   Verification: **`nc -zv host <port>`** (or `curl -v telnet://host:port`) (the box in 9.2.3). — 12. **The
   NACL outbound ephemeral rule** — the answer goes from source 443 to the client's ephemeral port; `ALLOW TCP
   1024-65535 to 0.0.0.0/0` is needed (9.4.1). — 13. (1) The route table, (2) the NACL in **both directions**,
   (3) the SG, (4) the OS firewall, (5) whether the service is listening and on which address (`ss -tulpn`)
   (9.4.2). — 14. If the **`pkts`** column in `iptables -L -n -v` output stays at **0** the rule is never
   matching — most likely there is a broader rule above it (9.2.2, the box in 9.2.3).

15. Because **answering is giving information**: a scanner that gets a REJECT gains the knowledge "there is a
    machine here"; one that gets a DROP cannot tell whether there is a machine or the address is empty — the
    scan slows down and becomes uncertain (9.2.3). **What it costs you:** diagnosis gets harder — seen from
    outside, "the firewall blocked it", "there is no host" and "there is no route" look exactly the same (all
    of them time out). — 16. Because **a NACL is stateless** (9.1.2): it evaluates the returning packet
    independently, and that packet's **destination port** is not the server's port but the client's
    **ephemeral port** (1.4.1) — since you cannot know which one it will be, you open the whole range. An SG
    is **stateful**: it holds the connection in its table and lets the returning packet through without
    looking at the rules at all (9.1.3, 9.4.1). — 17. (i) A private subnet blocks connections **from the
    internet**, not connections **from inside the VPC** — every instance in the same VPC can reach all ports
    (7.3.2). (ii) The machines in a private subnet **can go out** over NAT, so a compromised machine is not
    blocked (7.1.2). Trusting a single layer is a mistake (9.5.2). — 18. **An MTU black hole** (9.3.1): ICMP
    Fragmentation Needed is blocked → a device on the path drops the large packet but cannot tell the sender →
    the sender retries at the same size → the packets disappear silently. The symptom does not look like a
    firewall at all: *"SSH connects but freezes"*, *"the database queries hang"*, *"the site opens but the
    images do not load"* (5.7.3, 7.4.2).

## Scoring

| Correct answers | What it means |
|---|---|
| 16–18 | Filtering has settled — especially the stateful/stateless distinction. You are ready for Phase 10. |
| 13–15 | Good. Read 9.4 once more; that distinction causes the most mistakes in the cloud. |
| 9–12 | Go over the timeout/refused distinction and the NACL return rule again. |
| 0–8 | Walk the phase again. The goal: saying "firewall" when you hear timeout and "service" when you hear refused. |

**Which section to go back to for a question you missed:**

| Question | Section |
|---|---|
| 1, 2, 3, 16 | 9.1 Stateful vs stateless |
| 4, 5, 14 | 9.2.1-9.2.2 Filtering and order |
| 6, 9, 10, 11, 15 | 9.2.3 DROP vs REJECT |
| 7, 18 | 9.3 ICMP policy |
| 8, 12, 13 | 9.4 SG vs NACL |
| 17 | 9.5 Security reflexes |

---
---

# Phase 9 — Closing and Bridge to Phase 10

## What you carry from this phase

Phase 9 taught you **deliberate refusal**. You understood the difference between stateful and stateless
filtering — that one remembers and the other evaluates every packet alone — and why that difference requires
a return rule. You saw that decisions are made with the 5-tuple and why order and default deny are critical.
You picked up that the difference between DROP and REJECT is **the fastest distinction in diagnosis**. You
built the chain that runs from turning ICMP off entirely to an MTU black hole. And you saw that the
difference between an SG and a NACL in the cloud rests on the TCP state information from Phase 5.

The three most durable sentences: **"timeout is a firewall, refused is a service"**, **"ping is not a
permission test"** and **"do not forget the return rule in a NACL."**

## Where Phase 10 connects

Through ten phases you saw a **"When this phase breaks"** table at the end of each one. Address failures,
subnet mistakes, ARP problems, missing routes, MTU black holes, the DNS cache, the NAT table, 502/504, silent
DROPs.

Right now all of that is **scattered**. Ten separate tables, dozens of symptoms.

The difference a real engineer makes is not in memorising those tables — it is in knowing **in which order to
look.** When a user says "it does not work" you have thirty possibilities in your hand and no time to try them
all one by one.

Phase 10 is the answer to that: a layer-by-layer diagnostic methodology. A disciplined bottom-up order (link →
IP → route → DNS → port → application), which layer each tool looks at, decision trees for three instinctive
questions, and in the cloud, **reading** rather than guessing why a packet was dropped, with **VPC Flow
Logs**.

After that phase you will have not a list but a **reflex**.

> **🤔 Phase output — ask yourself:** A user says "the site will not open". You have `ping`, `dig`, `ss`,
> `curl`, `tcpdump`, `mtr` and `ip route`. Running them all takes 10 minutes and their outputs get tangled
> together. **Which one should you start with — and why that one?** (Hint: the right tool is not the one that
> gives the most information; it is the one that **eliminates the most possibilities**.)
>
> **🧪 Lab 9 idea (1–3 🟢, 4–5 🟡):** (1) Read the rules on your own machine with `sudo ufw status verbose`
> and `sudo iptables -L -n -v`; find the `ctstate RELATED,ESTABLISHED` line (the box in 9.2.3). (2) Connect to
> a closed port (`nc -zv localhost 9999`) and see the **refused**. (3) Connect to a filtered address and see
> the **timeout**; experience the two symptoms side by side (9.2.3). (4) 🟡 On your local machine add a rule
> with `sudo ufw allow 8080/tcp`, see it with `ufw status numbered`, then undo it with `sudo ufw delete
> <number>` *(Undo: list the rule numbers again before deleting — the numbers shift.)* (5) 🟡 Run `python3 -m
> http.server 8080` in one terminal and try to connect from another machine; see how the symptom changes with
> and without the rule *(Undo: stop the server with Ctrl+C and delete the rule.)*

---

> **Navigation:** [◀ Phase 8 — The Application Layer: HTTP + TLS](Phase_8_HTTP_and_TLS.md) · **Phase 9** · [Checkpoint Quiz 3 ▶](Checkpoint_Quiz_3.md)
