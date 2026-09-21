# Phase 11 — The Bridge to the Cloud: From an Empty VPC to Production

> **Navigation:** [◀ Phase 10 — Troubleshooting](Phase_10_Troubleshooting.md) · **Phase 11** · [Appendix A — Command Glossary ▶](Appendix_A_Command_Glossary.md)

---

## Where we come from

At the end of Phase 10 I asked: when you create a VPC, **why is the first question you meet "IPv4 CIDR
block"?**

The answer is the first section of this phase, but the essence is this: because a network is, before
everything else, an **address space.** Subnets are divided out of it, routes are written according to it, and
whether or not you will clash with your neighbours is decided there. A VPC's CIDR block **does not change
easily afterwards** — which is why it is the first question.

What you bring with you: **everything.** This phase has not a single new networking concept. You will only
learn the **AWS names** of the things you have been learning for eleven phases.

## The question of this phase

> *"Is cloud networking really something new, or is it the things I already know presented under different
> names?"*

The answer is the second — and that is this book's most important claim.

Cloud providers do not make you **build** a network, they make you **describe** one. You do not pull cable,
you do not configure a switch, you do not type commands into a router. Instead you fill in boxes in a console
and, behind the scenes, software builds that network on your behalf.

But what it builds is **the same network.** The same addresses, the same subnet logic, the same routing
tables, the same TCP, the same DNS. The only thing that changes is how you **express** them.

So there is no such thing as "learning" cloud networking; there is **mapping** it. And once you have built the
mapping, even when you look at a cloud service you have never seen you can say *"this is really that."* The
goal of this phase is to give you that reflex.

---

## By the end of this phase

- You will be able to explain why a VPC is an address space and which three decisions the CIDR choice affects
  permanently
- You will be able to state clearly that the difference between a "public subnet" and a "private subnet" comes
  from the **route table**
- You will be able to map an IGW, a NAT Gateway, an egress-only IGW and a VPC Endpoint to the right scenario
- You will be able to explain the difference between a Security Group and a NACL **on the basis of** the TCP
  state knowledge from Phase 5
- You will be able to justify the choice between an ALB and an NLB with the L7/L4 distinction
- You will know the options for connecting VPCs (peering, Transit Gateway, VPN, PrivateLink) and each one's
  CIDR constraint
- **You will be able to describe an empty VPC from scratch and explain which fundamental each of its parts
  corresponds to** — the real goal of this book

---

## Phase map

| Section | Topic | Depth | Its counterpart |
|---|---|---|---|
| 11.1 | VPC = an address space | `[concept]` | Phase 2 (CIDR) |
| 11.2 | Subnets and route tables | `[application]` | Phases 2 + 4 |
| 11.3 | The doors to the internet | `[application]` | Phase 7 (NAT) |
| 11.4 | Route 53 | `[concept]` | Phase 6 (DNS) |
| 11.5 | SG vs NACL | `[application]` | Phases 9 + 5 |
| 11.6 | ALB, NLB, CloudFront | `[concept]` | Phases 5 + 8 |
| 11.7 | Connecting VPCs | `[concept]` | Phases 2 + 4 + 7 |
| 11.8 | When this bridge breaks | — | **The complete mapping table** |

> **How to work through this phase:** There is no section marked `[skip]` in this phase, but there is one
> warning: this phase **is not a command phase.** What you will learn here is not a tool but a **mapping.** As
> you read each section ask yourself a single question: *"This is another name for what, learned in which
> phase?"* If you cannot answer immediately, go back to the relevant phase — because memorising a cloud name
> is of no use at all without knowing the mechanism underneath it. Console screens change every year;
> **fundamentals do not.** And be sure to do the finishing exercise at the end of the phase: drawing a VPC on
> a blank sheet of paper is the one-page exam for everything this book has given you.

---
---

# 11.1 A VPC Is Your Address Space

## 11.1.1 Why the first question is CIDR `[concept]`

The first thing asked of you when you create a VPC is an **IPv4 CIDR block** — for example `10.0.0.0/16`.

This is precisely what you learned in Phase 2 (2.2): a network definition. `10.0.0.0/16` says that your
network covers the 65,536 addresses between `10.0.0.0` and `10.0.255.255`.

Why is it the **first** question? Because this block draws the **limits** of three decisions you will make
later:

1. **How many subnets you can divide it into** (2.4) — a `/16` leaves you comfortable; if you start with a
   `/24` you run out of room after a few subnets.
2. **How many machines fit** — AWS **reserves 5 addresses** in every subnet (2.3.2), which is why you cannot
   create a subnet smaller than a `/28`.
3. **Whether you will clash with your neighbours** — and this is the most critical one. When you want to
   connect two VPCs with peering, **if the CIDR blocks overlap you cannot connect them** (11.7.1). The same
   applies when you set up a VPN with your corporate network.

The third item is the mistake that costs the most in the field. Everybody picks `10.0.0.0/16`; when two
companies merge or two teams want to connect their networks a clash appears, and the solution is **to rebuild
the VPC**. That is why in serious organisations CIDR blocks are handed out centrally.

Practical advice: pick a **generous block that nobody else is using** from the RFC 1918 range (1.3.1). The
`10.x` space is large; `172.16–31` is used less and carries a lower risk of collision.

> **⚠️ Common misconception: "A VPC is a data centre."**
>
> No — a VPC is an **address space and a routing domain.** It has no physical place; underneath it there is a
> software layer running on shared physical hardware. A VPC belongs to a **region** and can spread over
> several **availability zones** — but subnets cannot: **every subnet is in exactly one AZ** (11.2.1). Think
> of a VPC not as "a building you rent" but as "an address block allocated to you plus the routing rules
> written for that block". The distinction matters, because what a VPC gives you is not isolation but
> **addressing and routing control**; what actually provides the isolation is the route tables and the
> security rules (11.5).

---
---

# 11.2 Subnets and Route Tables

## 11.2.1 A subnet: dividing the block `[application]`

You divide the VPC's `10.0.0.0/16` block into subnets — exactly the job you did by hand in Phase 2.4:

```
VPC:            10.0.0.0/16          (65,536 addresses)
├── subnet-a:   10.0.1.0/24          (256 addresses, 251 usable — 5 reserved, 2.3.2)
├── subnet-b:   10.0.2.0/24
└── subnet-c:   10.0.3.0/24
```

Two rules:

- **Every subnet is in one AZ.** If you want high availability you build at least two subnets doing the same
  job **in different AZs**.
- **Subnets cannot overlap.** The logic of Phase 2 applies unchanged.

## 11.2.2 There is no public subnet setting `[application]`

This is the most misunderstood point in cloud networking and I said it once in Phase 7.3.1 — I am repeating it
here because everything depends on it:

> **A public subnet is not a subnet with a "public" box ticked. It is a subnet whose route table has a
> `0.0.0.0/0 → IGW` row.**

So the difference is not in the subnet itself but in the **route table attached to it** (precisely the routing
table of Phase 4.2):

| Subnet type | In its route table | What it means |
|---|---|---|
| **Public** | `0.0.0.0/0 → igw-xxx` | Goes out to the internet directly, reachable from outside with a public IP |
| **Private** | `0.0.0.0/0 → nat-xxx` | Goes out over NAT, **you cannot get in** (7.2.3) |
| **Isolated** | **no** `0.0.0.0/0` row | Cannot get out to the internet at all (only inside the VPC) |

Every route table also has an automatic **local** row: `10.0.0.0/16 → local`. This is how traffic inside the
VPC is routed and it **cannot be deleted** — which means all the subnets in the same VPC can reach each other
even if you do nothing. When I said in Phase 9.5.2 that "a private subnet does not protect you", this row was
exactly what I meant.

And the routing decision is made with the **longest prefix match** you learned in Phase 4.3: a packet destined
for `10.0.5.20` matches both the `10.0.0.0/16 → local` and the `0.0.0.0/0 → igw` rows, but because the `/16` is
more specific, **local** wins. The routing engine running in the cloud does exactly the job you did by hand in
Phase 4.

![Figure 11.1 — The anatomy of a VPC: the CIDR block, public and private subnets spread over AZs, the route table attached to each subnet, the internet gateway and the NAT gateway; under each box is written the fundamental concept it corresponds to.](../diagrams/png/nw-11-01-vpc-anatomy.png)

In the figure, the small label under each cloud component tells you which concept — learned in **which phase**
— that piece corresponds to. By the time you finish this phase you should be able to write those labels
yourself.

> **🤔 Think 11.1** — You created a subnet, put an instance in it and gave it a public IP. But the instance
> cannot get out to the internet. The route table has only the `10.0.0.0/16 → local` row. (a) What is the
> problem? (b) Why was having a public IP not enough? (c) Which type does this subnet count as?
>
> *(Answer: at the end of the phase)*

---
---

# 11.3 The Doors to the Internet: IGW, NAT GW, Endpoint

The whole of Phase 7 connects here. There are four doors and each answers a different question.

## 11.3.1 A comparison of the four `[application]`

| Door | What it does | Direction | Its counterpart |
|---|---|---|---|
| **Internet Gateway (IGW)** | Connects the VPC to the internet; carries traffic with public IPs both ways | **Both ways** | Phase 7.3.2 |
| **NAT Gateway** | Lets a private subnet **get out**, does not allow anything in | **Outbound only** | Phase 7.2 (PAT) |
| **Egress-only IGW** | The NAT GW equivalent for IPv6 — outbound only | **Outbound only** | Phase 7.5.3 |
| **VPC Endpoint** | Access to AWS services **without going out to the internet** | Inside the VPC | Phase 7.3.4 |

Keep three points in mind:

1. **A NAT Gateway sits in a public subnet** and itself goes out over the IGW. The private subnet's route
   table points at the NAT GW; the route table of the public subnet the NAT GW lives in points at the IGW. It
   is a two-step chain and if either step is missing it does not work.
2. **There is no NAT in IPv6** (7.5.1) — because there is no address shortage. If you want "let it go out but
   do not let anything in", you use an **egress-only IGW**; it is not NAT, only a direction filter.
3. **A VPC Endpoint saves both money and security**: if traffic going to S3 passes through the NAT Gateway you
   pay data processing charges; with an endpoint the traffic never leaves the VPC.

## 11.3.2 The three conditions for getting out to the internet `[application]`

The list I wrote in Phase 7.3.3 is the complete checklist for the "I cannot get out" problem in the cloud. For
an instance to be able to reach the internet **all three** are required:

1. **Is there a door** — is an IGW attached to the VPC (or has a NAT GW been set up)?
2. **Is there a path** — does the `0.0.0.0/0` row in the subnet's route table point at that door (11.2.2)?
3. **Is there an address** — if it is in a public subnet, does the instance have a **public IP**? (Not needed
   in a private subnet; the NAT GW's address is used.)

On top of these comes the security layer (the SG outbound rule, the NACL in **both directions**, 11.5). The
complete five-item list is in Phase 9.4.2, and getting through that list before opening an "I cannot connect"
ticket in the cloud saves time for you and for whoever is on the other end.

---
---

# 11.4 Route 53 = DNS

The cloud counterpart of Phase 6 is simple: **Route 53 is a DNS.** The same hierarchy, the same record types,
the same TTL (6.4.1).

There are two AWS-specific things:

**1. Alias records.** In Phase 6.3.3 you saw that a CNAME cannot be used at the apex (`example.com`). Route
53's **alias** record gets round that restriction: even at the apex it can point at an ALB or a CloudFront
distribution. Technically it is answered like an A record, but when the target's IP changes it is updated
automatically. And no query charge is made for it.

**2. Routing policies.** You define more than one answer for the same name and decide which one is returned:

| Policy | What it does | Its counterpart |
|---|---|---|
| Simple | A single answer | A classic A record (6.3.1) |
| Weighted | Splits by percentage | Gradual rollout (canary) |
| Latency | Returns the fastest region | An effect similar to anycast (8.5.3) |
| Failover | Moves to the standby according to a health check | High availability |
| Geolocation | According to the user's location | Regional content |

Watch out: all of these work **at the DNS level**, which means they are **subject to TTL** (6.4.1). Even a
failover policy does not take effect instantly, because of the clients' caches. Remember the rule from Phase
6.4.3: **lower the TTL before the change.** This is the single cause of the complaint "we turned failover on
but users are still going to the old server" in the cloud.

The VPC also has its own internal DNS: the address `10.0.0.2` (one of the addresses reserved in 2.3.2) is the
VPC's resolver, and it is what resolves private hosted zones.

---
---

# 11.5 Security Group vs NACL

This section is a summary of Phase 9.4 — but here once more, together with the **why**.

## 11.5.1 The difference in one sentence `[application]`

> **A Security Group is stateful: you do not write a rule for the return traffic. A NACL is stateless: you
> do.**

And the source of that difference is **Phase 5**: an SG tracks the state of the TCP connection (SYN → SYN-ACK
→ ACK, 5.2.1) and knows that a packet is "part of a connection I have already allowed" (9.1.3). A NACL, on the
other hand, evaluates every packet without memory, so the return packet has to match a rule **on its own** —
and that packet's destination port is in the **ephemeral range** (1.4.1):

```
NACL Inbound  100: ALLOW TCP 443        from 0.0.0.0/0
NACL Outbound 100: ALLOW TCP 1024-65535 to   0.0.0.0/0    ← the most frequently forgotten row
```

## 11.5.2 Practical design `[application]`

The field practice is clear: **do the real access control in the SG** and leave the NACL as a coarse extra
layer (9.4.2). The reason: because an SG is stateful the risk of making a mistake is low, and **you can write
another SG as the source** (9.4.1) — when IPs change the rule does not break.

The correct SG chain for a layered application looks like this:

```
Internet ──► alb-sg      ALLOW 443 from 0.0.0.0/0
              │
              ▼
            app-sg      ALLOW 8080 from alb-sg      ← an SG reference, not an IP
              │
              ▼
            db-sg       ALLOW 5432 from app-sg
```

Each layer knows only the one above it. This is the cloud equivalent of the principle of least privilege
(9.5.1) and it stops lateral movement even if one layer is compromised.

One last reminder: even when these rules are right, the instance's **own operating system firewall** and the
service's **listening address** can still block it (1.4.2, 9.4.2). Cloud rules are not the end of the network
layer, only one layer of it.

> **🤔 Think 11.2** — A team has written `ALLOW 5432 from 10.0.0.0/16` (the whole VPC) in the database SG. You
> are proposing `ALLOW 5432 from app-sg`. (a) Both work — what is the difference? (b) When instances die and
> are born with auto-scaling, which one breaks? (c) What is the difference from a security point of view?
>
> *(Answer: at the end of the phase)*

---
---

# 11.6 ALB, NLB and CloudFront

## 11.6.1 ALB vs NLB: L7 or L4 `[concept]`

The distinction you saw in Phase 8.5.1 turns into a product choice here:

| | **ALB** (Application LB) | **NLB** (Network LB) |
|---|---|---|
| Layer | **L7** — understands HTTP | **L4** — only TCP/UDP |
| Looks at, when deciding | Path, host header, cookie | IP and port |
| TLS | Terminates it (8.4.4) | Can pass it through or terminate it |
| The client IP | In the `X-Forwarded-For` header | **Preserved** (at packet level) |
| A fixed IP | No (a DNS name) | **Yes** (one per AZ) |
| Speed | Slightly slower (it reads the content) | Very fast |
| Related phase | Phase 8 | Phase 5 |

The selection rule is simple: **if you want to decide according to HTTP, ALB; if you do not, NLB.** If you
want to send `/api/*` requests to one target and `/static/*` requests to another, you need L7 — and only a
device that can read the content can do that. But if you are carrying a non-HTTP protocol (a database, a game
server, your own protocol) or you need a fixed IP, it is the NLB.

And remember the warning from Phase 8.4.4: because the ALB terminates TLS, **the server behind it cannot see
the client's IP**; it reads it from the `X-Forwarded-For` header. Session handling and log analysis depend on
that header. With an NLB there is no such problem because it does not open the packets.

## 11.6.2 CloudFront = CDN `[concept]`

CloudFront is the CDN you learned in Phase 8.5.2: it keeps content at points geographically close to the user
(edge locations) and lowers the latency. The routing is done with **anycast** (8.5.3) — the same IP address is
announced from many points around the world and the user goes to the nearest one.

Two practical notes:

- **Cache invalidation is expensive and slow.** The solution from Phase 8.5.4 applies here too: put a version
  in the file name (`app.a3f9c2.js`) and never need invalidation at all.
- **Protect the origin.** If the server behind CloudFront stays directly reachable, the CDN can be bypassed.
  Opening the origin's SG only to traffic coming from CloudFront is the standard practice (the logic of
  11.5.2).

---
---

# 11.7 Connecting VPCs to Each Other

## 11.7.1 Four options `[concept]`

| Method | What it connects | CIDR constraint | Its counterpart |
|---|---|---|---|
| **VPC Peering** | Two VPCs | **Cannot overlap** | Phases 2 + 4 (routes) |
| **Transit Gateway** | Many VPCs + on-prem | **Cannot overlap** | A central router (Phase 4) |
| **Site-to-Site VPN** | VPC ↔ corporate network | Cannot overlap | Phase 7.4 (IPsec) + 4.6 (BGP) |
| **PrivateLink** | Access to a single **service** | **No constraint** | Phase 7.3.4 (endpoint) |

The constraint the first three share is a direct consequence of Phase 2: **CIDR blocks cannot overlap.** The
reason is routing (4.3) — if two different networks use the same address range, a router cannot know which one
to send a packet destined for `10.0.1.5` to. That is why the advice in 11.1.1 is critical: choose your CIDR
**thinking about who you will want to talk to later**.

The reason PrivateLink knows no such constraint is the same logic: it does not merge two networks, it only
presents **a single service** as an interface inside your VPC. Because the address spaces never meet there is
no clash either — and this is usually the only practical way for two sides with clashing CIDRs to communicate.

Two extra subtleties:

- **Peering is not transitive.** If there is A↔B and B↔C peering, A and C **cannot talk**. A separate
  connection is needed for each pair and a separate route row on both sides. As the number of VPCs grows this
  explodes combinatorially — that is the reason Transit Gateway exists: a central router in a star topology.
- **Setting up peering is not enough, you have to write routes.** The other VPC's CIDR block must be added to
  both sides' route tables. If it is written on one side only you get the classic asymmetry (10.3.3): one
  direction works, the other times out.

> **💡 Cloud connection — the two phases of a Site-to-Site VPN:** A Site-to-Site VPN connecting a corporate
> network to a VPC uses two separate phases of this book at the same time. The tunnel itself is **IPsec**
> (7.4.1): the traffic is encrypted and carried over the internet. Learning the routes is usually done with
> **BGP** (4.6.1): the two sides announce to each other which CIDR blocks they own and the route tables fill
> automatically. And the trap from Phase 7.4.2 is fully in play here: because the tunnel headers make the
> packet bigger the effective MTU drops, and if ICMP Fragmentation Needed is blocked an **MTU black hole**
> forms (9.3.1) with the symptom "SSH connects but freezes". In VPN setups **MSS clamping** (5.7.4) is almost
> always needed. The intersection of these three phases is the class of fault that wastes the most time in the
> field.

> **🤔 Think 11.3** — Two companies are merging. Both of their VPCs use `10.0.0.0/16` and the networks need to
> talk. (a) Why will peering not work? (b) Which options are left? (c) What should have been done to prevent
> this situation from the start?
>
> *(Answer: at the end of the phase)*

---
---

# 11.8 When This Bridge Breaks — The Complete Mapping and Fault Map

## 11.8.1 Fundamental → cloud mapping

This table is the summary of this book. If you know the left column, the right column is only a name.

| Fundamental | Phase | The AWS counterpart |
|---|---|---|
| A private address block (RFC 1918) | 1.3 | **The VPC CIDR** |
| Subnetting, the mask | 2.2-2.4 | **Subnet** |
| Reserved addresses | 2.3.2 | The VPC's 5 reserved addresses (`.0 .1 .2 .3 .255`) |
| DHCP | 1.6 | Automatic private IP assignment |
| ARP, the broadcast domain | 3.1-3.3 | The VPC's software L2 layer |
| The routing table, LPM | 4.2-4.3 | **Route Table** |
| The default gateway | 4.1 | The `0.0.0.0/0` row |
| BGP | 4.6 | Direct Connect / VPN route announcement |
| TCP state | 5.2 | The SG's stateful behaviour |
| MTU / MSS | 5.7 | Jumbo frames, MSS clamping in a VPN |
| The DNS hierarchy, TTL | 6.1-6.4 | **Route 53**, the VPC resolver (`.2`) |
| NAT / PAT | 7.1-7.2 | **NAT Gateway** |
| The public/private distinction | 7.3 | The route table's target (IGW vs NAT GW) |
| An IPsec tunnel | 7.4 | **Site-to-Site VPN** |
| IPv6, egress filtering | 7.5 | A dual-stack VPC, **Egress-only IGW** |
| L7 routing | 8.1-8.2 | **ALB** |
| TLS termination | 8.4.4 | ALB/CloudFront + `X-Forwarded-For` |
| Reverse proxy, CDN, anycast | 8.5 | **CloudFront**, Route 53 latency |
| A stateful firewall | 9.1.3 | **Security Group** |
| A stateless filter | 9.1.2 | **Network ACL** |
| Packet capture | 10.2.2 | **VPC Flow Logs** (+ Traffic Mirroring) |

## 11.8.2 When the bridge breaks

| Symptom | Likely cause | Verify | Related section |
|---|---|---|---|
| The instance cannot get out to the internet | There is no `0.0.0.0/0` in the route table | The route table + IGW/NAT GW | 11.2.2, 11.3.2 |
| It has a public IP but is unreachable | The subnet has no IGW route | The route table | 11.2.2 |
| The private subnet cannot get out | There is no NAT GW, or it is **not in a public subnet** | The NAT GW's location + both route tables | 11.3.1 |
| I set up peering, it does not work | **The route rows were not written** (on both sides) | Both route tables | 11.7.1 |
| Peering cannot be set up | **The CIDRs overlap** | The VPC CIDR blocks | 11.1.1, 11.7.1 |
| A→B ✓, B→A ✗ | A one-sided route or a NACL | Flow Logs, both directions | 10.3.3, 11.7.1 |
| The connection is made, no answer comes | The NACL outbound **ephemeral** rule is missing | The NACL outbound | 11.5.1 |
| New instances cannot connect to the DB | There is an **IP-based** rule in the SG | Turn the SG source into an SG reference | 11.5.2 |
| I turned failover on, traffic goes to the old place | **The DNS TTL** | The TTL with `dig`, lower it first | 11.4, 6.4.3 |
| I cannot write a CNAME at the apex | The CNAME apex restriction | Use an **alias** record | 11.4, 6.3.3 |
| The application sees all clients as the same IP | The ALB is terminating TLS | `X-Forwarded-For` | 11.6.1, 8.4.4 |
| Large transfers freeze over the VPN | The tunnel MTU / **MSS clamping** | `ping -M do -s ...` | the box in 11.7.1, 5.7.3 |
| S3 traffic is getting expensive | It is going through the NAT GW | Add a **VPC Endpoint** | 11.3.1, 7.3.4 |
| IPv4 ✓ IPv6 ✗ | The `::/0` route or rule is missing | The route table + SG/NACL | 11.3.1, 7.5.2 |

> **The lesson from this table:** Notice — **not one of the faults in this table is a cloud fault.** Every one
> of them is a networking concept you have been learning for eleven phases, expressed wrongly in the cloud: a
> missing route row (Phase 4), a clashing CIDR (Phase 2), a forgotten return rule (Phase 9), a TTL that was
> not lowered (Phase 6), a tunnel MTU (Phases 5+7). The cloud does not bring you new faults; it brings you
> **old faults under new names.** That is why the only method that works in cloud network diagnosis is Phase
> 10's methodology: from the bottom up, eliminating a group of possibilities at each step. The only thing that
> changes is the tools — **Flow Logs** instead of `tcpdump`, **the route table screen** instead of `ip route`.
> The question is the same: *how far did the packet get and what happened to it there?*

---
---

# Answers to the Think questions

## Answer 11.1 — A public IP is not a path

**(a)** **The subnet's route table has no `0.0.0.0/0` row.** There is only `10.0.0.0/16 → local`, which means
the instance can speak only inside its own VPC. No packet going outside the VPC has anywhere to go — exactly
the situation of "a machine with no default gateway" from Phase 4.1.

**(b)** Because **a public IP is an address, not a path** (7.3.1). An address says where the packet came from
and where the answer is to return; but for the packet to be able to **leave** the VPC a route and a door are
needed. Without a door the address is useless. Remember the three conditions in 11.3.2: a door, a path, an
address — all three at once.

**(c)** This subnet is **isolated** — neither public nor private. If it were public it would have a
`0.0.0.0/0 → IGW` row, if private a `0.0.0.0/0 → NAT GW` row (11.2.2). The fix: attach an IGW to the VPC and
add `0.0.0.0/0 → igw-xxx` to the route table; that single row makes the subnet public.

**Related section:** 11.2.2, 11.3.2 · **Next:** Phase 7.3.1, Phase 4.1.

## Answer 11.2 — An SG reference vs a CIDR rule

**(a)** Both allow the connection, but their **scopes** differ. `10.0.0.0/16` allows **everything in the VPC**
— the application servers, the bastion, test machines, any instance opened by accident. `app-sg` allows only
instances that carry that security group; the permission is tied not to an address but to a **role**.

**(b)** **Neither breaks** — but for different reasons, and the real difference shows up with narrower rules.
The CIDR rule keeps working because new instances are born inside that block; the SG reference is unaffected
anyway because it is independent of IPs. What does break is when someone says *"let me allow just these three
IPs"* and writes an IP list — when the IPs change with auto-scaling that rule breaks **silently** and the new
instances cannot connect. An SG reference removes that class of error completely (9.4.1).

**(c)** **The blast radius.** If any machine inside the VPC is compromised — a test server, a CI runner, a
forgotten instance — with the CIDR rule it can connect to the database **directly**. With an SG reference only
the application layer can reach it; the attacker has to compromise the application server first. This is the
concrete equivalent of the principle of least privilege (9.5.1).

**Related section:** 11.5.2 · **Next:** Phase 9.5.1-9.5.2, Phase 9.4.1.

## Answer 11.3 — Overlapping CIDRs

**(a)** Because peering works by adding the other side's **CIDR block** to the two VPCs' route tables
(11.7.1). If both sides are `10.0.0.0/16`, when you ask a router *"where do I send a packet destined for
`10.0.1.5`?"* the answer is ambiguous: both its own local network and the other side match. What is more,
**the local route always wins** (11.2.2), so the packet never crosses over. The routing logic of Phase 4.3
makes this impossible — and AWS does not allow peering with overlapping CIDRs to be set up in the first place.

**(b)** Three options: (i) **PrivateLink** — it does not merge the networks, it only presents specific
services to the other side as an interface; it has no CIDR constraint (11.7.1) and it is the standard solution
for this situation. (ii) One of the sides **rebuilding its VPC** and moving to a different CIDR — the cleanest
but the most expensive path (the instances are recreated). (iii) Putting **NAT** in between and translating
the addresses (7.1.1) — it works but it is complicated, it makes diagnosis harder and it is usually considered
a last resort.

**(c)** **Planning the CIDR blocks centrally** (11.1.1). Everybody picking the default `10.0.0.0/16` is the
single cause of this problem. In serious organisations CIDR blocks are handed out in advance for each
account/team/region and kept on record. The RFC 1918 space (1.3.1) is more than enough for this — as long as a
few minutes are spent thinking about it on day one. This is the decision in network design that is **most
expensive to reverse.**

**Related section:** 11.1.1, 11.7.1 · **Next:** Phase 2.5 (supernetting), Phase 4.3 (LPM).

---
---

# Frequently asked questions

**Q1 — What exactly is the difference between a VPC and a subnet?** A VPC is an **address space** (a CIDR
block) and belongs to a region. A subnet is a piece of that block and is in **exactly one AZ** (11.2.1).
Whether it is public or private is determined not by the subnet itself but by **the route table attached to
it** (11.2.2).

**Q2 — A NAT Gateway is expensive, what can I use instead?** Three ways: (i) if you are only reaching AWS
services, a **VPC Endpoint** (11.3.1) — both cheap and secure. (ii) In low-traffic environments a NAT instance
on an EC2 (you manage it). (iii) If it really does not need to reach the internet, leave the subnet isolated.
Most cost surprises come from S3/ECR traffic passing through the NAT.

**Q3 — ALB or NLB?** If you need to decide according to HTTP (path/host routing, cookies), **ALB**; if you
need a non-HTTP protocol, a fixed IP or very high performance, **NLB** (11.6.1). If the client IP is critical:
an NLB preserves it, an ALB puts it in the `X-Forwarded-For` header.

**Q4 — Why is there no DENY in a Security Group?** Because it works with **default deny** (9.2.2): everything
you do not write is already forbidden. If you need to block a specific source explicitly you use a **NACL** —
it is the only layer that can write a DENY (11.5.1).

**Q5 — When should I turn VPC Flow Logs on?** Before a problem appears. It is the only source of evidence that
reaches into the past (10.4.2), and turning it on at the moment of a fault does not bring back the traffic up
to that point. To lower the cost, collecting only REJECT records is a common choice.

**Q6 — How do I connect to a machine in a private subnet?** Three ways: a bastion host (in a public subnet, an
SSH jump point), **SSM Session Manager** (without ever opening the SSH port — the preferred one), or from the
corporate network over VPN/Direct Connect. Opening port 22 to `0.0.0.0/0` is a substitute for none of them
(9.5.1).

**Q7 — Which fundamental is the most critical for learning cloud networking?** **Phase 2 (CIDR) and Phase 4
(route tables).** Almost the whole of cloud networking is the question "which address block, in which table,
points where". Phase 9 (SG/NACL) comes right after. Someone who knows these three can read a cloud network
service they have never seen before.

---
---

# Test yourself

Write your answers on paper, then compare them with the key. Target: 14+ out of 18.

## Part A — Counterparts

1. Which fundamental concept is a VPC the counterpart of?
2. A route table is the same as which structure from Phase 4? With which rule is the decision made?
3. Which mechanism does a NAT Gateway implement?
4. An SG being stateful rests on the knowledge of which phase?
5. Which restriction does a Route 53 alias record solve?
6. VPC Flow Logs is the cloud counterpart of which local tool — and in what respect is it weaker than it?

## Part B — Application

7. How is a "public subnet" defined? In one sentence.
8. Write the three conditions needed for an instance to get out to the internet.
9. Which subnet must a NAT Gateway sit in, and why?
10. Which port range do you open in a NACL for the return traffic?
11. Write the rule chain of three SGs in a layered application.
12. You set up peering but no traffic flows. What is the first thing you look at?

## Part C — Reasoning

13. Why is the first question when creating a VPC the CIDR block?
14. If two VPCs' CIDRs overlap, why is peering impossible?
15. Explain the security difference between an SG reference and a CIDR rule.
16. You turned on a failover policy but the traffic goes to the old server. Cause and solution?
17. The server behind an ALB sees all requests coming from the same IP. Why, and what is the solution?
18. Defend the claim "cloud networking is not something new" with three examples.

---

## Answer key

1. An **address space** — a network defined by an RFC 1918 private block (1.3, 2.2, 11.1.1). — 2. **The
   routing table** (4.2); the decision is made with **longest prefix match** (LPM) and, because the `local`
   row is more specific, it always wins for traffic inside the VPC (4.3, 11.2.2). — 3. **PAT** — it takes many
   private addresses out from behind a single public address, distinguishing them by port number (7.2.1,
   11.3.1). — 4. **Phase 5** — TCP connection state (SYN/SYN-ACK/ACK, 5.2.1). That is how an SG knows a packet
   is "part of a connection I have already allowed" (9.1.3). — 5. The restriction that **a CNAME cannot be
   used at the apex** (6.3.3); an alias can point at an ALB/CloudFront even at the apex (11.4). — 6.
   **`tcpdump`** (10.2.2). Its weakness: it records only the **headers**, not the content — and it is not real
   time, the aggregation interval is 1–10 minutes (10.4.2).

7. A subnet whose route table has a **`0.0.0.0/0 → Internet Gateway`** row (11.2.2). — 8. (1) A door (an IGW
   or a NAT GW), (2) a `0.0.0.0/0` row in the route table pointing at that door, (3) an address (a public IP
   in a public subnet; the NAT GW's in a private one) — plus the SG/NACL permissions (11.3.2). — 9. **In a
   public subnet** — because the NAT GW itself has to reach the internet and it can only do that in a subnet
   that has an IGW route. The private subnet's route table points at the NAT GW, and the NAT GW's subnet
   points at the IGW (11.3.1). — 10. **The ephemeral range** — AWS's recommendation is `1024-65535` (1.4.1,
   11.5.1). — 11. `alb-sg: ALLOW 443 from 0.0.0.0/0` → `app-sg: ALLOW 8080 from alb-sg` → `db-sg: ALLOW 5432
   from app-sg` (11.5.2). — 12. **The route rows** — has the other side's CIDR block been added to both VPCs'
   route tables? Setting up peering does not make traffic flow on its own; if it is written on one side only
   you get asymmetry (11.7.1, 10.3.3).

13. Because a network is before everything else an **address space**, and this block permanently determines
    three things: how many subnets it can be divided into (2.4), how many machines will fit, and who you will
    be able to connect to later (the overlap constraint). Changing it afterwards means, in practice, rebuilding
    the VPC (11.1.1). — 14. Because the routing becomes ambiguous: a packet destined for `10.0.1.5` matches
    both the local network and the other VPC, and **the local route always wins** (11.2.2), so the packet never
    crosses over. LPM requires a single definite answer (4.3, 11.7.1). — 15. A CIDR rule allows **everything in
    the VPC**; any compromised machine can connect to the database directly. An SG reference ties access to a
    **role**: only the application layer gets through and the blast radius shrinks (11.5.2, 9.5.1). — 16. The
    cause: **the DNS TTL** — clients and intermediate resolvers are caching the old answer (6.4.1). The
    solution: lower the TTL **before** the change, wait as long as the old TTL, then change it (6.4.3, 11.4). —
    17. Because the ALB **terminates** TLS and opens its own connection to the back (8.4.4); the source IP the
    server sees is the ALB's. The solution: read the **`X-Forwarded-For`** header (or use an NLB if the client
    IP is critical) (11.6.1). — 18. Three examples are enough: (i) **a VPC = a CIDR block** (Phase 2), (ii) **a
    route table = a routing table + LPM** (Phase 4), (iii) **a Security Group = a stateful firewall** (Phase 9)
    — an SG not needing a rule for the return traffic comes directly from TCP state (Phase 5). In addition: a
    NAT GW = PAT (Phase 7), Route 53 = DNS (Phase 6), CloudFront = a CDN + anycast (Phase 8) (11.8.1).

## Scoring

| Correct answers | What it means |
|---|---|
| 16-18 | You have built the mapping. You are no longer "learning" cloud networking, you recognise it. The book is done. |
| 13-15 | Good. Go through the 11.8.1 table once more, reading it from right to left. |
| 9-12 | Go back to 11.2 and 11.5; the whole of the cloud stands on those two sections. |
| 0-8 | Go back to the relevant fundamental phases (2, 4, 9). A cloud name is of no use without the mechanism. |

Which section to go back to for a question you missed:

| Question | Section |
|---|---|
| 1, 13 | 11.1 VPC = an address space |
| 2, 7, 12 | 11.2 Subnets and route tables |
| 3, 8, 9 | 11.3 The doors to the internet |
| 5, 16 | 11.4 Route 53 |
| 4, 10, 11, 15 | 11.5 SG vs NACL |
| 17 | 11.6 ALB, NLB, CloudFront |
| 14 | 11.7 Connecting VPCs |
| 6, 18 | 11.8 The complete mapping table |

---
---

# Closing: The End of the Workbook, the Beginning of the Network Journey

## What you carry from this phase — and from the whole workbook

In this phase you did not learn a single new networking concept. What you learned was **the mapping**: a VPC
is an address space, a route table is a routing table, a NAT Gateway is PAT, a Security Group is a stateful
firewall, Route 53 is a DNS, an ALB is an L7 proxy, Flow Logs is a packet record.

And that is the proof of this book's real claim: **cloud networking is not something new.** It is the way the
things you already know are described.

Look back. Over twelve phases you built these: that the layers are a diagnostic order (Phase 0), how a machine
gets its address (Phase 1), how a network is divided (Phase 2), how a MAC is found on the local network (Phase
3), how a packet travels between networks (Phase 4), how reliable delivery is constructed (Phase 5), how a
name is turned into an IP (Phase 6), how a single address is shared (Phase 7), how the web is built on top of
all of that (Phase 8), why a packet is deliberately dropped (Phase 9), and in which order a fault is hunted
(Phase 10).

The three most durable sentences are what remains of this whole book:

- **"From the bottom up."** Until the lower layer is proved, no test above it is trustworthy.
- **"Timeout is a firewall, refused is a service."** Two words, two different teams.
- **"Do not settle for an interpretation, look for evidence."** `tcpdump` and Flow Logs end arguments.

## Where to go from here

The end of this workbook is the beginning of the real cloud journey. Every service you meet from now on — ECS,
EKS, RDS, Lambda, API Gateway, App Mesh — will come with its own network model. And you will approach each of
them with the same two questions: *"Which network mechanism is underneath this?"* and *"When something breaks,
which layer do I go down to?"*

Those two questions are the most durable habit this book has given you.

> **🤔 Final output — ask yourself:** When you look at a cloud service you have never seen (say a "service
> mesh" or an "API Gateway"), what should your first questions be? Hint: *which layer does it work at, where
> does it get its address, how does it resolve names, who filters its traffic, and where do I look when it
> breaks?* If you can ask those five questions, you already know half of it before you read the documentation.
>
> **🧪 Finishing exercise — draw an empty VPC (paper 🟢, console 🟡):** The one-page exam of this book. (1) 🟢
> On a blank sheet of paper draw a VPC: choose the CIDR block and write **why you chose that block** (11.1.1).
> (2) Divide it into four subnets spread over two AZs — two public, two private — and **calculate each one's
> CIDR by hand** (2.4). (3) Write the route table for each subnet: which rows are there, what are their
> targets (11.2.2). (4) Place the IGW, the NAT GW and, if needed, a VPC Endpoint; explain **why** you put the
> NAT GW in a public subnet (11.3.1). (5) Write the SG chain for a three-tier application (11.5.2). (6) Add
> the Route 53 record and the ALB; write **why an alias** is needed for the apex (11.4). (7) Now the most
> important step: **under each box write which concept, learned in which phase, it is the counterpart of.** If
> you can write them all, you have finished this book. (8) 🟡 If you like, actually build the same design in
> your own test account and run `curl ifconfig.me` from an instance. *(Undo: when you are done, **delete** the
> NAT Gateway and the instances — a NAT GW is charged by the hour and is the resource that most often creates
> a cost surprise when forgotten.)*

You have finished this workbook. You now know where a packet goes, why it is dropped and in which layer a
fault is to be hunted. Your cloud journey starts here.

---

> **Appendices:** [Appendix A — Command Glossary](Appendix_A_Command_Glossary.md) · [Appendix B — Concept and File Map](Appendix_B_Concept_File_Map.md) · [Appendix C — "When It Breaks" Quick Reference](Appendix_C_When_It_Breaks_Quick_Reference.md)

> **Navigation:** [◀ Phase 10 — Troubleshooting](Phase_10_Troubleshooting.md) · **Phase 11** · [Appendix A — Command Glossary ▶](Appendix_A_Command_Glossary.md)
