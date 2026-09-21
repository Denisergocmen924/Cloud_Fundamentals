# Phase 4 — Between Networks: Routing (L3)

> **Navigation:** [◀ Phase 3 — The Local Network (L2)](Phase_3_Local_Network_L2.md) · **Phase 4** · [Checkpoint Quiz 2 ▶](Checkpoint_Quiz_2.md)

---

## Where we come from

All through Phase 3 we kept hitting the same wall: **a broadcast cannot cross a router, ARP is local, the
MAC of a remote destination cannot be learned.** And at the end of the phase we asked you: what does the
gateway do when it receives the packet, and how does a router **know** the way?

This phase is the whole of those two questions. What you brought with you:

- The **"is the destination on my network?"** decision (2.1.2). Phase 4 tells you what happens after the
  answer is "no".
- **ARP is always performed for the next hop** (3.1.2). In this phase you will see how that "next hop" is
  chosen.
- **The longer the prefix, the more specific the network** (2.5.2). This phase turns that specificity into
  a **decision rule**: longest prefix match.

## The question of this phase

> *"How does my packet reach a destination through dozens of networks that have no direct connection to
> each other — and how is that possible when no device along the way knows the whole path?"*

The core of the answer, and we will say it right now: **no router knows the whole path.** Every router only
answers the question "where is the next step". The path is the **chain** of those decisions — a chain nobody
sees in full, but in which every single step is correct.

This is the place the mentor roadmap calls "the layer a cloud engineer debugs most". When something does not
work in the cloud, in the vast majority of cases the answer is one of three things: **a missing row in the
route table**, **a wrong next hop**, or **an asymmetric return path**. By the end of this phase you will
recognise all three.

---

## By the end of this phase

- You will be able to explain what a default gateway is and how the rule "everything not on my local
  network goes here" works
- You will know that **every device** has a routing table, and you will be able to read every column of
  `ip route` output
- You will be able to apply the longest prefix match rule and say which row wins when several rows match
- You will be able to explain what TTL is for, why it exists and what happens when it reaches 0
- You will be able to explain ICMP's role (error reporting + diagnosis) and know how `ping` and
  `traceroute` use it
- You will be able to read `traceroute`/`mtr` output and explain why `* * *` lines appear and why they **do
  not mean a fault**
- You will be able to explain, at awareness level, the difference between static and dynamic routing and
  BGP's role on the internet
- **Cloud:** You will be able to explain that an AWS route table is the cloud counterpart of this table,
  what the `0.0.0.0/0 → igw-...` row means, and why Direct Connect/VPN use BGP

---

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 4.1 | The default gateway | `[mechanism]` | The door in the wall Phase 3 left |
| 4.2 | The routing table | `[mechanism]` | Where the decision is stored |
| 4.3 | Longest prefix match | `[mechanism]` | **The heart of the phase** — the rule of the decision |
| 4.4 | TTL and ICMP | `[mechanism]` | Loop protection + error reporting |
| 4.5 | How traceroute works | `[mechanism]` | Making the path visible |
| 4.6 | Dynamic routing and BGP | `[concept]` | Where the tables get filled from |
| 4.7 | When this phase breaks | — | The signatures of routing failures |

> **How to work through this phase:** The observation commands are 🟢 (`ip route`, `ip route get`,
> `traceroute`, `mtr`, `ping`). **Adding/removing** a route is 🟡 (`ip route add/del` — it persists until
> reboot; making it permanent would be 🔴); only try those on your own test machine. The single most
> valuable command of this phase is **`ip route get <destination>`**: it asks the machine "how would you get
> to this destination" and tells you exactly which row it picked. After reading 4.3, run this command
> against five different destinations — it is the fastest way to take longest prefix match out of theory and
> see it with your own eyes.

---
---

# 4.1 The Default Gateway — The Door to the Unknown

## 4.1.1 The rule: if it is not local, send it to the gateway `[mechanism]`

Recall the decision we learned in Phase 2: the machine compares the destination's network address with its
own network address (2.1.2).

- **If they are the same** → direct delivery: find the destination's MAC with ARP, send the frame (Phase 3).
- **If they differ** → send it to the **default gateway**.

The **default gateway** is the IP address of a router on the machine's own network. The rule is one
sentence: **"I send everything that is not on my local network here."**

The most important detail here is missed by most people: when the machine sends the packet to the gateway,
it **does not change the IP destination**. The packet's destination IP is still the remote server. The only
thing that changes is the **L2 envelope**: the gateway's MAC is written into the destination MAC field
(Phase 0.2.2, Think 1.1). So:

```
Dest IP:  93.184.216.34      ← unchanged (the final destination)
Dest MAC: <the gateway's MAC> ← changes at every hop (the next step)
```

And this is why the machine **performs ARP** for the gateway (3.1.2) — the gateway is on its own network, so
its MAC can be learned.

A critical constraint: **the gateway must be inside the machine's own subnet.** Otherwise the machine cannot
reach it either — because reaching it would itself require a gateway, and that would be an infinite loop.
This is the classic mistake of a misconfigured gateway: you write
`ip route add default via 10.0.2.1` but your machine is on `10.0.1.50/24` → "Error: Nexthop has invalid
gateway."

> **🔧 See it on your machine** 🟢 — find your default gateway
>
> ```
> $ ip route | grep default
> default via 172.31.16.1 dev ens5 proto dhcp src 172.31.20.15 metric 100
>
> $ ip neigh | grep 172.31.16.1
> 172.31.16.1 dev ens5 lladdr 06:8f:3c:2a:11:04 REACHABLE
> ```
>
> The first line: everything remote goes to `172.31.16.1`, out of the `ens5` interface. `proto dhcp` says
> this information arrived **via DHCP** (Phase 1.6.1 — one of the four things DHCP hands out). The second
> command is the proof of Phase 3: the gateway is in your ARP table, because it is **your neighbour**.
> Remote destinations are not there. Put the two outputs side by side — Phase 3 and Phase 4 join up exactly
> here.

> **⚠️ Common misconception: "The gateway carries my packet all the way to the destination."**
>
> No — the gateway moves the packet **one step** forward, that is all. The gateway receives the packet,
> looks at its own routing table, finds **its own** next hop and sends the packet there in a new L2
> envelope. That one does the same. So the path is a **chain of independent decisions**; no device knows the
> whole path and no device takes on "the responsibility of getting this packet to the destination". This has
> two practical consequences: (1) the outbound path and the return path **can differ** (asymmetric routing —
> you will see it in 4.7, a very common source of failures in the cloud); (2) if a router in the middle of
> the path breaks, your machine only finds out **when the reply does not come back**.

> **🤔 Think 4.1** — Your machine is `10.0.1.50/24` and its gateway is `10.0.1.1`. Somebody accidentally
> changes the gateway to `10.0.9.1`. (a) Can the machine reach that gateway — why? (b) Will
> `ping 10.0.1.80` (a neighbour on the same network) work? (c) What error does `ping 8.8.8.8` give, and
> what do you see in `ip neigh`?
>
> *(Answer: at the end of the phase)*

---
---

# 4.2 The Routing Table

## 4.2.1 Every device has a table `[mechanism]`

A common misconception: "routing tables live on routers". No — **every device that speaks IP** has a routing
table. Your phone has one, your laptop has one, every EC2 instance has one.

The table holds the machine's answers to the question "for which destination should I send where". Before
every packet is sent, this table is consulted.

On Linux you see it with `ip route`:

```
$ ip route
default via 172.31.16.1 dev ens5 proto dhcp src 172.31.20.15 metric 100
172.31.16.0/20 dev ens5 proto kernel scope link src 172.31.20.15
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
```

Let us decode it line by line — these three lines are the entire network world of a typical machine:

**Line 1 — `default via 172.31.16.1 dev ens5`**
`default` = `0.0.0.0/0`, that is, **every destination** (the `/0` of Phase 2.2.1). "If no other row matches,
send the packet to `172.31.16.1` out of `ens5`." This is the default gateway row.

**Line 2 — `172.31.16.0/20 dev ens5 scope link`**
This is the machine's **own network** (computed with the AND operation in 2.1.2). There is no `via` —
because no gateway is needed. `scope link` says exactly that: **"I reach this network directly, with ARP."**

**Line 3 — `172.17.0.0/16 dev docker0`**
Docker's virtual bridge. If Docker is installed this row is added automatically. `linkdown` = there is no
active container right now.

What the columns mean:

| Column | Meaning |
|---|---|
| **Destination** (`default`, `172.31.16.0/20`) | Which destinations it applies to — a prefix |
| **`via <IP>`** | The next hop (the gateway). If absent, the destination is directly reachable |
| **`dev <interface>`** | The interface the packet will leave from |
| **`src <IP>`** | The address that will be written into the packet's source IP field |
| **`metric <n>`** | Cost — if there are several paths to the same destination, the **smaller** one wins |
| **`scope link`** | This network is reached directly, without a gateway |

> **💡 Cloud connection — an AWS route table is exactly this:** In a VPC every subnet is attached to a
> **route table**, and that table is the cloud counterpart of this `ip route` output. A typical public
> subnet table:
>
> | Destination | Target |
> |---|---|
> | `10.0.0.0/16` | `local` |
> | `0.0.0.0/0` | `igw-0a1b2c3d` |
>
> The first row is the counterpart of `scope link`: traffic inside the VPC flows directly, nobody routes it.
> The second row is the counterpart of `default via`: everything remote goes to the Internet Gateway. In a
> **private** subnet the second row's target is `nat-...` (Phase 7.3), or it is absent entirely — and then
> the subnet cannot reach the internet. This is exactly the most common cause of the "my instance cannot
> reach the internet" failure in the cloud: **there is no `0.0.0.0/0` row in the route table.** That single
> row is what makes a subnet "public" or "private" — not its name, its **route**.

> **🤔 Think 4.2** — A machine with the three-line table above sends three packets: to `172.31.20.9`,
> `172.17.0.5` and `8.8.8.8`. (a) Which row is picked for each? (b) For which of them is ARP performed, and
> **for which IP**? (c) In which case does the packet never leave the machine — and why?
>
> *(Answer: at the end of the phase)*

---
---

# 4.3 Longest Prefix Match — The Rule of the Decision

## 4.3.1 When more than one row matches `[mechanism]`

In a table, several rows can fit the same destination. For example:

```
10.0.0.0/8      via 192.168.1.1
10.0.5.0/24     via 192.168.1.2
0.0.0.0/0       via 192.168.1.254
```

Let the destination be `10.0.5.77`. **All three rows match**:
- `10.0.0.0/8` covers it (first octet 10 ✓)
- `10.0.5.0/24` covers it (first three octets 10.0.5 ✓)
- `0.0.0.0/0` covers everything ✓

Which one is chosen? The rule is one sentence and it has no exceptions:

> **The longest prefix wins.** (Longest Prefix Match — LPM)

Since `/24` > `/8` > `/0`, the winner is **`10.0.5.0/24`**. The packet goes to `192.168.1.2`.

The logic is this: **a long prefix = more specific information.** "Everything starting with 10 goes this
way" is a general rule; "the ones starting with 10.0.5 go this way" is more precise information. The network
always prefers the **more precise** one.

This is the seed we planted in Phase 2.5.2 coming into flower: with aggregation you can announce a broad
prefix (`10.0.0.0/8`), then add a more specific row for an exception (`10.0.5.0/24`) — and the exception
wins automatically. Being able to define an exception without breaking the general rule is routing's most
powerful property.

And it is now clear why `0.0.0.0/0` is "the default": its prefix length is **0**, the **shortest** possible
match. It is always the last choice — that is where the meaning "if nothing else fits" comes from.

![Figure 4.1 — Longest prefix match: all three rows in the routing table match the same destination IP, but the row with the longest prefix (the most specific one) is chosen; the default route 0.0.0.0/0 is always the last choice because its prefix length is zero.](../diagrams/png/nw-4-01-longest-prefix-match.png)

In the figure the destination address is on the left, the table rows are in the middle and the prefix length
of each row is on the right. Matching rows are marked; the winning row is highlighted. The lower part shows
the chosen row's next hop and the interface the packet will leave from — those two are the concrete result
of the decision.

> **🔧 See it on your machine** 🟢 — make the machine tell you its decision
>
> ```
> $ ip route get 8.8.8.8
> 8.8.8.8 via 172.31.16.1 dev ens5 src 172.31.20.15 uid 1000
>
> $ ip route get 172.31.20.9
> 172.31.20.9 dev ens5 src 172.31.20.15 uid 1000
> ```
>
> This command is the most valuable tool of the phase: it reads the table for you and tells you **which
> decision it will make**. The first output has `via 172.31.16.1` → via the gateway (a remote destination).
> The second has **no** `via` → directly, with ARP (a neighbour). That single difference is the output of
> the decision we have been chasing since Phase 2. When you have a routing suspicion, use this command
> instead of scanning the table by eye — it applies LPM for you and ends the argument.

> **❓ A question that comes to mind: "What if two rows have the same prefix length?"**
>
> Then **metric** comes into play: the smaller metric wins (4.2.1). If they also have the same metric, the
> system usually applies **ECMP** (Equal-Cost Multi-Path) — it spreads the traffic over both paths. The
> spreading is typically per connection (the same 4-tuple uses the same path, Phase 1.4.2), so the packets
> of one TCP connection do not get mixed up. You do not configure this directly in the cloud, but you see
> its effects: NAT Gateways, load balancers and Transit Gateway attachments scale behind the scenes with
> ECMP. What you need to know for diagnosis: when ECMP is in play, `traceroute` output **can change between
> runs** — because different packets may have taken different paths (4.5.2).

> **🤔 Think 4.3** — A table contains these rows: `0.0.0.0/0 via 10.0.0.1`, `192.168.0.0/16 via 10.0.0.2`,
> `192.168.5.0/24 via 10.0.0.3`, `192.168.5.10/32 via 10.0.0.4`. (a) Which row does the destination
> `192.168.5.10` use? (b) `192.168.5.99`? (c) `192.168.9.1`? (d) `8.8.8.8`? Write your reasoning for each.
>
> *(Answer: at the end of the phase)*

---
---

# 4.4 TTL and ICMP

## 4.4.1 TTL: the packet's lifetime `[mechanism]`

Routing has a danger. Since every router decides independently (4.1.1), a misconfiguration can push two
routers into throwing a packet at each other: A says "this destination is via B"; B says "this destination
is via A". The packet bounces forever and chokes the network. This is called a **routing loop**.

The solution is a small counter in every IP packet's header: **TTL** (Time To Live).

The mechanism is just three rules:

1. The sending machine writes an initial value into TTL (usually **64** on Linux, 128 on Windows).
2. **Every router** decrements TTL by **1** when it forwards the packet.
3. If TTL **reaches 0** the router **drops** the packet and sends an error message back to the sender:
   **ICMP Time Exceeded**.

So TTL does not mean "how many seconds", it means **"how many hops"** (the name is a historical leftover).
The initial value of 64 is plenty — a typical path on the internet is 10–20 hops.

And there is a side benefit: TTL **reveals** how long the path was. If you see `ttl=52` in a ping reply and
the other side started at 64, the packet passed through 12 hops.

## 4.4.2 ICMP: the network's language for errors and diagnosis `[concept]`

**ICMP** (Internet Control Message Protocol) is **not** a data transport protocol — it is the language the
network uses to report its own status. It is not a "carrier" like TCP or UDP, it is a helper standing beside
L3.

The message types you need to recognise:

| Message | When it is sent | What you learn |
|---|---|---|
| **Echo Request / Reply** | `ping` uses these | The destination is reachable and answering |
| **Time Exceeded** | TTL reached 0 | The path is too long or there is a routing loop |
| **Destination Unreachable** | No route to the destination / port closed | Routing is missing or the service is not there |
| **Fragmentation Needed** | The packet is big but DF is set | An MTU problem (Phase 5.7) — **very important** |

Note the last row already: if ICMP is **blocked**, MTU discovery does not work and an "MTU black hole"
forms — the connection is established but data does not flow (Phase 5.7). This is the most expensive side
effect of the security configurations that say "let us just turn ICMP off entirely, it will be safer".

> **⚠️ Common misconception: "If ping does not work, the server is down."**
>
> No. Ping uses **ICMP**; many servers and firewalls block ICMP **deliberately** (to make scanning harder).
> So a server that does not answer a ping may well be serving happily on port 443. The reverse is also true:
> ping may work while the application does not (L3 is fine, L4 is broken — Phase 1.4.1). The right reflex:
> use ping as a **hint about reachability**, not as **proof**. The correct way to find out whether a service
> is up is to try to connect to that service's **own port**: `curl -v https://host` or `nc -zv host 443`.

> **🤔 Think 4.4** — In the output of `ping google.com` you see `ttl=115`. (a) Which operating system family
> is the other side probably from, and what was its initial TTL? (b) How many hops did the packet pass
> through? (c) If you see different TTL values for the same destination at different times, what does that
> mean?
>
> *(Answer: at the end of the phase)*

---
---

# 4.5 How Traceroute Works

## 4.5.1 Using TTL as a weapon `[mechanism]`

`traceroute` reveals every hop along the path. But how? No router introduces itself by saying "I am here".

The trick is to use rule 3 of TTL (4.4.1): **when TTL reaches 0 the router sends an ICMP Time Exceeded —
and that message contains the router's own IP address.**

So traceroute does this:

1. It sends a packet to the destination with **TTL=1**. The first router decrements TTL to 0, discards the
   packet and sends **Time Exceeded**. → **hop 1's IP is learned.**
2. It sends with **TTL=2**. The first router decrements it to 1 and forwards it; the second router
   decrements it to 0 and sends the error. → **hop 2 is learned.**
3. It carries on with **TTL=3, 4, 5...**. Each time one more hop is exposed.
4. When the packet reaches the destination, the destination does **not** answer with Time Exceeded but with
   something else (Echo Reply or Port Unreachable). Traceroute sees this and stops.

It is an elegant trick: an error mechanism has been turned into a **discovery tool**.

## 4.5.2 Why `* * *` lines appear `[concept]`

Seeing starred lines in traceroute output is very common:

```
$ traceroute 8.8.8.8
 1  172.31.16.1   0.512 ms  0.489 ms  0.501 ms
 2  * * *
 3  100.66.12.33  1.221 ms  1.180 ms  1.203 ms
 4  * * *
 5  142.251.49.1  2.881 ms  2.902 ms  2.870 ms
```

`* * *` = "no answer came from this hop". The reasons — in order of importance:

- **The router does not generate ICMP, or rate-limits it.** The most common reason. Many backbone routers
  restrict ICMP error generation to reduce CPU load. Completely normal.
- **A firewall blocks ICMP.** Also normal, a security choice.
- **Cloud providers hide their internal hops.** The AWS/Azure/GCP backbones are usually invisible.

And the most important rule:

> **`* * *` lines in the middle are not a fault.** The traffic did pass through that hop — that router
> simply did not introduce itself. As long as the journey continues (the later hops answer) there is no
> problem.

The real sign of trouble is this: **everything is `*` from a certain hop onwards** — that is, past a
certain point no answer comes at all and the destination is not reached either. Then the break is somewhere
around that hop.

`mtr` (my traceroute) is a tool that runs traceroute **continuously** and shows the packet loss percentage
for every hop. It is far better than traceroute for finding intermittent problems — a single-shot traceroute
will miss an intermittent loss.

> **🔧 See it on your machine** 🟢 — watch the path live
>
> ```
> $ mtr -rwc 20 8.8.8.8
> HOST: myhost               Loss%   Snt   Last   Avg  Best  Wrst StDev
>   1. 172.31.16.1            0.0%    20    0.5   0.5   0.4   0.7   0.1
>   2. ???                   100.0%    20    0.0   0.0   0.0   0.0   0.0
>   3. 100.66.12.33           0.0%    20    1.2   1.3   1.1   2.4   0.3
>   4. 142.251.49.1           0.0%    20    2.9   3.1   2.8   4.2   0.4
> ```
>
> `-rwc 20` = report mode, wide output, 20 packets. The reading rule: **ignore the 100% loss in the middle
> (hop 2)** — that router does not generate ICMP, but the traffic passes through (the proof: hops 3 and 4
> answer). **The only meaningful thing is loss that continues all the way to the last hop.** The second
> thing to look at is a jump in the **Avg** column: if latency suddenly leaps at one hop and stays high
> afterwards, congestion starts there. Values that leap at a single hop and then return to normal are just
> that router's own CPU delay — they are misleading.

> **🤔 Think 4.5** — In a traceroute output hops 1–4 are normal, everything from hop 5 onwards is `* * *`
> and the destination is never reached. (a) Where is the break? (b) If you can connect to the same
> destination with `curl`, how does your conclusion change? (c) With which concept do you explain the
> difference between these two situations?
>
> *(Answer: at the end of the phase)*

---
---

# 4.6 Dynamic Routing and BGP — Awareness

## 4.6.1 Static vs dynamic `[concept]`

The rows in routing tables arrive in two ways:

**Static route** — written by hand. Something like `ip route add 10.0.5.0/24 via 192.168.1.2`. Its
advantage: predictable, simple. Its disadvantage: if the path breaks the table **does not fix itself**;
somebody has to go and change it by hand.

**Dynamic route** — **learned** by a protocol. Routers talk to each other, announce "I can reach these
networks", and when the topology changes the tables are updated **automatically**.

On small networks static is enough. But on the internet, with thousands of networks and constantly changing
paths, static is impossible — dynamic protocols are mandatory.

## 4.6.2 BGP: the internet's path announcement protocol `[concept]`

The internet consists of thousands of interconnected independent networks. Each one is called an **AS**
(Autonomous System): the block of network controlled by an ISP, a university, a cloud provider. Every AS has
a number (for example AWS's AS16509).

**BGP** (Border Gateway Protocol) is the protocol that announces **"which address blocks I can reach"**
between ASes. The backbone routing of the internet runs entirely on BGP.

Three things you need to know at awareness level:

1. **BGP is based on trust.** When an AS announces "this block is mine", there is no central authority that
   verifies it. A wrong or malicious announcement (a **route leak** or a **BGP hijack**) can pull the
   world's traffic to the wrong place — and this really has happened; it is the cause of a share of the
   large internet outages.
2. **Convergence takes time.** When a path breaks it can take minutes for the change to spread across the
   whole internet. This is why large outages do not recover "instantly" but **gradually**.
3. **Interior protocols are separate.** OSPF and RIP are protocols used **inside** an AS. In this book
   awareness of the names is enough — `[skip]`.

> **💡 Cloud connection — Direct Connect and VPN speak BGP:** When you connect your own data centre to AWS
> (Direct Connect or Site-to-Site VPN), a **BGP session** is established between the two sides. Your router
> announces "these on-prem CIDRs are mine"; the AWS side announces "these VPC CIDRs are mine". If **route
> propagation** is enabled, those announcements land as rows in the VPC route tables automatically — that
> is, you do not fill the table of Phase 4.2 by hand, BGP fills it. A practical diagnosis: if a VPN tunnel
> shows "UP" but traffic does not pass, in most cases the reason is that **the BGP session was not
> established** or that **route propagation is disabled** — the tunnel is up but nobody has announced any
> path to anybody. And Phase 2's warning comes back here: if the two sides' CIDRs overlap, the BGP
> announcement is meaningless and traffic never leaves at all.

---
---

# 4.7 When This Phase Breaks — Routing Failure Signatures

The signature of a routing failure is usually this: **everything local works, nothing remote works** — or,
more insidiously, **some remote destinations work and some do not**. The second almost always points to a
missing or wrong row in a table.

| Symptom | Likely cause | Verify | Related section |
|---|---|---|---|
| The local network works, the internet does not | The default gateway is missing or wrong | `ip route \| grep default` | 4.1.1 |
| "Network is unreachable" | No row for the destination, and no default either | `ip route get <destination>` | 4.2.1 |
| The gateway is unreachable | The gateway is outside the machine's subnet | `ip route`, `ip addr` prefix | 4.1.1 |
| Some destinations work, some do not | A missing/wrong specific route row | `ip route get` for each destination | 4.3.1 |
| The packet goes in the wrong direction | A too-specific row is winning under LPM | `ip route get <destination>` | 4.3.1 |
| A flood of "TTL exceeded" | A routing loop — two routers throwing to each other | `traceroute` (repeating hops) | 4.4.1 |
| The request goes out, the reply never comes | **Asymmetric routing** — the return path differs/is missing | `ip route` at both ends | 4.1.1 |
| `* * *` in the middle of a traceroute | ICMP restriction — **normal**, not a fault | Do the later hops answer | 4.5.2 |
| Everything is `*` after a certain hop + destination unreachable | The real break is around there | Repeated measurement with `mtr` | 4.5.2 |
| An instance in the cloud cannot reach the internet | No `0.0.0.0/0` row in the route table | The VPC route table | 4.2.1 |
| The VPN is "UP" but there is no traffic | No BGP session / route propagation disabled | VPN BGP status, route table | 4.6.2 |
| A peer VPC is unreachable in the cloud | Peering exists but no route table row was added | The route tables of both VPCs | 4.2.1 |

> **The lesson from this table:** In routing diagnosis one command is worth more than all the others:
> **`ip route get <destination>`**. Instead of scanning the table by eye it makes the machine state its own
> decision and applies longest prefix match for you — it ends the argument. The second reflex: routing is
> **bidirectional**. Your table may be correct, but if the return path is missing the connection still will
> not be established, and the symptom is "the request goes out, the reply never comes" — this is the class
> of failure that wastes the most time in the cloud (adding a route on only one side is the classic mistake
> in peering and VPN setups). So in a routing problem look at the table **at both ends**. The third: do not
> mistake `* * *` lines in a `traceroute` for a fault (4.5.2) — that is the most frequent misreading; only
> loss that **continues to the last hop** is meaningful. And always remember: no router knows the whole
> path, only the next step — which is why a fault is not looked for "somewhere along the way" but **in a
> specific router's table**.

---
---

# Phase 4 — Answers to the Think questions

## Answer 4.1 — The gateway must be in its own subnet

(a) **No, it cannot reach it.** The address `10.0.9.1` is **outside** the machine's network
(`10.0.1.0/24`) (the AND computation of 2.1.2: the network of `10.0.9.1` is `10.0.9.0`, the machine's is
`10.0.1.0` — different). To reach that address the machine would need a gateway, but that address *is* the
gateway — an infinite loop. Linux usually refuses to add this route in the first place ("Nexthop has
invalid gateway"); and if it is forced in, ARP fails.

(b) **Yes, it works perfectly.** `10.0.1.80` is on the same network, so the gateway is never involved: the
MAC is found with ARP and it is delivered directly (Phase 3.1). A gateway fault **does not affect** local
traffic — and this is exactly the source of the "the local network works, the internet does not" signature
(4.7).

(c) `ping 8.8.8.8` gives "Destination Host Unreachable" or "Network is unreachable". In `ip neigh` there is
either no entry at all for `10.0.9.1` or it shows **`FAILED`** — because the machine tries to ARP for that
IP (it may believe it is on its own network) and nobody answers (Phase 3.5).
**Related section:** 4.1.1 · **Next:** 4.7, Phase 10.3 (layer-by-layer diagnosis).

## Answer 4.2 — Three destinations, three different rows

(a) The row selections (by longest prefix match):
- `172.31.20.9` → the **`172.31.16.0/20`** row. (With `/20` the network is `172.31.16.0`; `.20.9` is in
  that range.)
- `172.17.0.5` → the **`172.17.0.0/16`** row (docker0).
- `8.8.8.8` → no specific row matches → the **`default`** row.

(b) The ARP situation:
- `172.31.20.9`: the row has **no** `via`, it is `scope link` → direct delivery → ARP is performed for
  **`172.31.20.9`** (the destination itself).
- `172.17.0.5`: again `scope link` → ARP is performed for the destination itself (via the docker0 bridge).
- `8.8.8.8`: `via 172.31.16.1` → ARP is performed for the **gateway's IP** (`172.31.16.1`). ARP is
  **never** performed for `8.8.8.8` (Phase 3.1.2).

(c) **`172.17.0.5`** — because that row has `linkdown`: there is no active container attached to the
docker0 bridge. The packet is handed to the interface but, with no physical/logical link, it does not go
out; the ARP goes unanswered and the connection fails. (The more general lesson: the **existence** of a
route row does not mean that path **works**.)
**Related section:** 4.2.1, 4.3.1 · **Next:** Phase 3.1.2 (ARP), 4.7.

## Answer 4.3 — The longest prefix always wins

(a) **`192.168.5.10`** → all four rows match (`/0`, `/16`, `/24`, `/32`). The longest: **`/32`** →
**`via 10.0.0.4`**. A `/32` denotes a single address (2.2.1) and is the most specific row possible.

(b) **`192.168.5.99`** → the `/32` row does not match (that row is only for `.10`). The longest of the
remaining three is **`/24`** → **`via 10.0.0.3`**.

(c) **`192.168.9.1`** → the `/24` does not match (the third octet is 9, not 5). The `/16` matches
(`192.168.x.x`) → **`via 10.0.0.2`**.

(d) **`8.8.8.8`** → no specific row matches. What remains is **`0.0.0.0/0`** → **`via 10.0.0.1`**.

The pattern in all four answers: the rows are evaluated not in **order of elimination** but in **order of
specificity**. The order of the rows in the table means nothing — the rule is single and always the same
(4.3.1).
**Related section:** 4.3.1 · **Next:** Phase 2.5.2 (aggregation), 4.7.

## Answer 4.4 — TTL reveals the length of the path

(a) The common initial values are 64 (Linux/Unix/macOS), 128 (Windows), 255 (some network devices). The
value `ttl=115` is closest to 128 and below it → the other side most likely **started at 128** (the Windows
family, or a load balancer/network device that uses 128).

(b) 128 − 115 = **13 hops**. The packet passed through 13 routers.

(c) It means the path is **changing**. The reasons: (i) ECMP — traffic is being spread over several
equal-cost paths (the box in 4.3.1); (ii) a path broke and BGP announced a new one (4.6.2); (iii) the
destination is an anycast address and you are landing on different servers at different times (Phase 8.6 —
`8.8.8.8` works exactly like that). This is direct proof that the internet's paths are **not fixed**.
**Related section:** 4.4.1 · **Next:** 4.5.1 (traceroute), Phase 8.6 (anycast).

## Answer 4.5 — A `*` line is not a fault, it is a silence

(a) At first glance the break is **around hop 5**: there are answers up to 4, after that silence and the
destination is not reached either. The "loss that continues to the last hop" rule (4.5.2) supports this
conclusion.

(b) **It changes completely.** If `curl` works, the traffic **is reaching** the destination — so the path is
sound. In that case the reason for the `* * *` lines is not a break but that from hop 5 onwards **ICMP is
blocked or is hitting a rate limit**. Traceroute depends on ICMP (4.5.1); with no ICMP the path is
invisible but **the data flows**.

(c) The difference is **the independence of ICMP and data traffic.** Traceroute measures ICMP error
messages; your application carries TCP/UDP. A path can block ICMP and pass TCP — and this is extremely
common. The practical lesson: use traceroute as a **map of the path**, not as a **health test**. Always do
the real test with the application's own protocol (`curl -v`, `nc -zv`) — the very same thing as the ping
misconception in 4.4.2.
**Related section:** 4.5.2 · **Next:** 4.4.2, Phase 10.2 (choosing a tool).

---
---

# Phase 4 — Frequently asked questions

**Q1 — Why does my laptop have a routing table, I am not a router?** Because a routing table is not needed
"to do routing", it is needed for the **"where should I send this packet"** decision — and every device that
speaks IP has to make that decision. The difference with being a router is forwarding **other people's**
packets too (IP forwarding is enabled). Your machine only routes its own packets, but it still needs a table
for that (4.2.1).

**Q2 — Are `default` and `0.0.0.0/0` the same thing?** Yes, entirely. `default` is just a readable alias.
Because its prefix length is 0, it is the **last** preference under LPM — that is where the meaning "if
nothing else fits" comes from (4.3.1).

**Q3 — Can there be more than one default gateway?** Yes, but the two are ordered by their `metric` value
and the smaller metric wins (4.2.1). This is used for redundancy: if the primary link breaks the second row
takes over. If two defaults have the same metric, ECMP is applied and the traffic is spread — which can
produce unwanted results (connections use different egress IPs).

**Q4 — Would increasing TTL help?** No, and there is no need. The value 64 is more than enough for any path
on the internet (a typical path is 10–20 hops). In a situation where TTL runs out the problem is not
distance but, almost always, a **routing loop** (4.4.1) — and increasing TTL does not solve the loop, it
just keeps the packet circulating for longer.

**Q5 — Is turning ICMP off entirely safe?** No, it is **dangerous**. ICMP is not just ping: the
**Fragmentation Needed** message is vital for MTU discovery (4.4.2). If you block it an "MTU black hole"
forms — the TCP connection is established, small packets pass, large packets vanish silently, and it is very
hard to diagnose (Phase 5.7). The right approach: instead of turning ICMP off entirely, filter it **by
type** — you may restrict Echo Request, but Time Exceeded and Fragmentation Needed must be allowed through.

**Q6 — Is asymmetric routing really a problem?** Not by itself — it is extremely common on the internet and
usually works fine. The problem starts if there is a **stateful** device along the path: a stateful firewall
or NAT (Phase 7, Phase 9). That device sees the outbound packet and records the connection; if the return
packet arrives by another path, it treats it as an **unknown connection** and drops it. In the cloud this is
the most common cause of the "the request goes out, the reply never comes" failure (4.7).

**Q7 — Will I write route tables by hand in the cloud?** Mostly yes — and that is a good thing, because it
is explicitly visible. When a VPC is created the `local` row comes automatically; for internet access **you**
add the `0.0.0.0/0 → igw-...` row. Rows also have to be added for peering, Transit Gateway and VPC
Endpoints. The one exception: on connections established with BGP (Direct Connect, S2S VPN) the rows land
automatically if **route propagation** is enabled (4.6.2).

---
---

# Phase 4 — Test yourself

Write your answers on paper, then compare them with the answer key. Target: 14+ out of 18.

## Part A — Definition and mechanism

1. What is a default gateway and under which rule is it used?
2. What does the machine write into the destination IP and destination MAC fields when it sends a packet to
   the gateway?
3. Why must the gateway be inside the machine's own subnet?
4. What do `via`, `dev`, `scope link` and `metric` mean in `ip route` output?
5. Write the longest prefix match rule in one sentence.
6. Why is `0.0.0.0/0` always the last preference?
7. What is TTL for, what happens at every hop, and what is sent when it reaches 0?
8. Write ICMP's four message types and when each one is sent.

## Part B — Apply and diagnose

9. A table contains `0.0.0.0/0 via 10.0.0.1`, `172.16.0.0/12 via 10.0.0.2`, `172.16.5.0/24 via 10.0.0.3`.
   Which row does `172.16.5.88` use?
10. In the same table, which row does `172.20.1.1` use?
11. On a machine there is no `default` row at all in `ip route` output. What symptom do you expect?
12. With which command do you ask the machine "how would you get to this destination"? What do you look at
    in its output?
13. In a traceroute output hop 3 is `* * *` but hops 4, 5 and the destination all answer. Is there a
    problem?
14. An instance in the cloud cannot reach the internet, its IP and gateway are correct. Where do you look?

## Part C — Reasoning and connection

15. Explain the sentence "no router knows the whole path" and write one practical consequence.
16. How does traceroute learn the hops when no router introduces itself?
17. What insidious side effect does blocking ICMP entirely have, and why?
18. In which situation does asymmetric routing turn into a real problem? Why?

---

## Answer key

1. The IP of a router on the machine's own network; the rule: **every destination not on the local network
   is sent here** (4.1.1). — 2. Into the destination IP field, **the final destination's IP** (unchanged);
   into the destination MAC field, **the gateway's MAC** (changes at every hop) (4.1.1). — 3. Because
   reaching it would itself require a gateway — that would be an infinite loop. The gateway has to be
   directly reachable with ARP (4.1.1). — 4. **`via`** = the next hop; **`dev`** = the interface the packet
   leaves from; **`scope link`** = this network is reached directly, without a gateway; **`metric`** = cost,
   and if there are several paths to the same destination the smaller one wins (4.2.1). — 5. If several rows
   match, the one with **the longest prefix (the most specific)** wins (4.3.1). — 6. Its prefix length is
   **0** — the shortest possible match, therefore always last under LPM (4.3.1). — 7. It cuts infinite loops
   (routing loops). Every router **decrements** TTL by 1; when it reaches 0 the packet is dropped and an
   **ICMP Time Exceeded** is sent to the sender (4.4.1). — 8. **Echo Request/Reply** (ping), **Time
   Exceeded** (TTL 0), **Destination Unreachable** (no route/port), **Fragmentation Needed** (packet too big
   + DF set, MTU discovery) (4.4.2).

9. **`172.16.5.0/24 via 10.0.0.3`** — all three rows match but `/24` is the longest prefix (4.3.1). —
   10. **`172.16.0.0/12 via 10.0.0.2`** — the `/12` range is `172.16.0.0`–`172.31.255.255` and `172.20.1.1`
   is in it; the `/24` row does not match (4.3.1). — 11. The local network works, **no remote destination
   is reachable**; the error is typically "Network is unreachable" (4.1.1, 4.7). — 12. **`ip route get
   <destination>`**. In the output you look at **whether there is a `via`**: if there is, it goes via the
   gateway (remote); if not, directly/with ARP (a neighbour) (4.3.1). — 13. **No, there is no problem.** A
   `* * *` in the middle only shows that that router does not generate ICMP; since the later hops answer,
   the traffic is passing (4.5.2). — 14. At the **VPC route table** — is there a `0.0.0.0/0` row, and is its
   target `igw-...` (a public subnet) or is it missing (4.2.1, 4.7).

15. Every router only answers the question **"where is the next step"**; the path is a chain of independent
    decisions. The practical consequences: (i) the outbound and return paths **can differ** (asymmetric
    routing); (ii) the source machine only notices a fault in the middle of the path when the reply does not
    come back; (iii) diagnosis is not done "somewhere along the way" but **in a specific router's table**
    (4.1.1, 4.7). — 16. By increasing TTL step by step: with TTL=1 the first router drops the packet and
    sends an **ICMP Time Exceeded**, and that message contains **the router's own IP**; with TTL=2, 3... each
    hop is exposed in turn (4.5.1). — 17. An **MTU black hole**: if the "Fragmentation Needed" message is
    blocked, the sender cannot learn that its packet is too big. The connection is established, small
    packets pass, large packets are dropped **silently** — a failure that is very hard to diagnose (4.4.2,
    Phase 5.7). — 18. When there is a **stateful** device along the path (a stateful firewall or NAT). That
    device sees the outbound packet and records the connection; if the return packet arrives by another path
    it is treated as an unknown connection and dropped (Q6, Phase 7 and Phase 9).

## Scoring

| Correct | What it means |
|---|---|
| 16-18 | Routing has settled in. You can now answer "why is the router being stupid". |
| 13-15 | Good. Read 4.3 (LPM) again and try five destinations with `ip route get`. |
| 9-12 | Table reading is weak. Find the three rows of 4.2.1 on your own machine and explain each one. |
| 0-8 | Walk the phase again. Target: translating an `ip route` output line by line, without memorising. |

Question you missed → section to return to:

| Question | Section |
|---|---|
| 1, 2, 3, 11 | 4.1 The default gateway |
| 4, 12, 14 | 4.2 The routing table |
| 5, 6, 9, 10 | 4.3 Longest prefix match |
| 7, 8, 17 | 4.4 TTL and ICMP |
| 13, 16 | 4.5 Traceroute |
| 15, 18 | 4.1.1 + the 4.7 failure table |

---
---

# Phase 4 — Closing and the Bridge to Phase 5

## What you carry out of this phase

Phase 4 gave you the whole of **remote delivery**. You learned the default gateway's "everything not local
goes here" rule, that every device has a routing table, and what each column of that table says. You can
apply longest prefix match — the single, exception-free rule of the decision. You saw how TTL cuts infinite
loops, that ICMP is the network's language for errors, and how traceroute turns an error mechanism into a
discovery tool. You know at awareness level how BGP announces the internet's paths.

The most durable sentence is this: **no router knows the whole path.** Each one knows only the next step —
and that is why a fault is not looked for "somewhere along the way" but **in a specific row of a specific
table**.

## Where Phase 5 connects to this

From Phase 0 up to here you have learned one thing: **the packet can reach its destination.** But there is
one thing we never discussed — **there is no guarantee that it will.**

IP is a "best effort" protocol: it sends the packet, but it does not promise delivery. The packet can be
dropped on the way, arrive out of order, or be duplicated. A router's buffer can fill up, a cable can be
noisy, a hop can get congested.

So then how does downloading a file work? Why does a web page not arrive corrupted?

The answer is the **transport layer** — the subject of Phase 5. You will see how TCP notices losses and
retransmits, why a three-way handshake is needed, how flow and congestion control adjust the speed, and the
**MTU** problem, the most insidious cause of packet loss.

Phase 4 **got** the packet to its destination; Phase 5 makes it **reliable**.

> **🤔 Phase output — ask yourself:** A router's buffer filled up along the way and it dropped your packet.
> That router tells the source machine **nothing** (ICMP is not mandatory and is often not sent). So **how**
> will the sender find out that its packet was lost? And once it has found out, how will it know which
> packet to send again? (Hint: the answer lies in a system of **numbering** and **acknowledgement**.)
>
> **🧪 Lab 4 idea (mostly 🟢, one 🟡):** (1) Translate your `ip route` output line by line — for each row
> write the destination, the next hop, the interface and the scope. (2) Try five different destinations with
> `ip route get`: an IP from your own network, your gateway itself, `8.8.8.8`, `127.0.0.1` and, if Docker is
> installed, `172.17.0.2`. Look at whether each output has a `via` and explain why. (3) From the `ttl=`
> value in `ping -c1 8.8.8.8` output, compute how many hops it passed through. (4) Run
> `mtr -rwc 20 8.8.8.8`; interpret the `* * *` lines and the jumps in the Avg column according to 4.5.2 —
> which are meaningful, which are noise? (5) 🟡 On your test machine add a fake route:
> `sudo ip route add 192.0.2.0/24 via <your gateway>`, then use `ip route get 192.0.2.5` to see the new row
> being chosen. *(Undo: `sudo ip route del 192.0.2.0/24` — and it also disappears by itself on reboot.)*
> The fifth step lets you change LPM with your own hands.

---

> **Navigation:** [◀ Phase 3 — The Local Network (L2)](Phase_3_Local_Network_L2.md) · **Phase 4** · [Checkpoint Quiz 2 ▶](Checkpoint_Quiz_2.md)
