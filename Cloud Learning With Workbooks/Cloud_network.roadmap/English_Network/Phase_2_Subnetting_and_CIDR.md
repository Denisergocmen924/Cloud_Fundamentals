# Phase 2 — Subnetting and CIDR: Dividing the Network

> **Navigation:** [◀ Phase 1 — Addressing](Phase_1_Addressing.md) · **Phase 2** · [Checkpoint Quiz 1 ▶](Checkpoint_Quiz_1.md)

---

## Where we come from

At the end of Phase 1 we left you this question: what happens if two machines are given the same address range
but **different prefixes**? In this phase you get both the answer to that question and all the arithmetic
behind it.

The three things you carry from Phase 1 are the three legs of this phase:

- **An IP address is 32 bits** and the dots are only for reading (1.2.1). This phase works entirely on those
  32 bits — and the dots being meaningless means subnet boundaries **do not have to line up with octet
  boundaries**.
- **The binary ↔ decimal conversion** (1.2.2). In Phase 1 we said "do it by hand, you will need it in Phase
  2". This is the phase where you need it.
- **The network part / host part split** (1.2.3). Phase 1 said that split *exists*; Phase 2 teaches you
  **where** that boundary is and how to shift it.

## The question of this phase

> *"When I write `/24`, what exactly am I saying — and when I change that number by one, how many machines
> stop being 'on the same network'?"*

This is the question in the book that needs the most practice. Because the answer is not a definition but a
**skill**: being able to look at a prefix and **work out on paper** the boundaries of the network, the number
of usable addresses and the neighbourhood relationships.

And this skill carries the whole of cloud network design. When you create a VPC you write `10.0.0.0/16`; when
you split it into subnets you say `10.0.1.0/24` and `10.0.2.0/24`; in a security group rule you allow the
source `10.0.0.0/16`; when you peer two VPCs the operation is rejected if their CIDRs **overlap**. All of it
reduces to a single question: **which addresses does this prefix cover?**

## How you should read this phase — a special warning

This is the phase you must go through **most slowly** in the book. In the other phases it is enough to read a
section and feel that you have understood it; not here. Here, if you move on without **taking pen and paper
and doing the arithmetic** at the end of every section, Phase 4 (routing) and Phase 11 (AWS VPC) will stay in
you as memorisation, not as understanding.

The concrete goal: when you finish this phase, for any `a.b.c.d/nn` given to you at random, you should be able
to say the network address, the broadcast address, the usable range and the host count **in 10 seconds**. This
phase exists for that goal.

---

## By the end of this phase

- You will be able to explain what a subnet mask is, why it is "ones on the left, zeros on the right", and
  that `255.255.255.0` and `/24` are the same thing
- You will be able to read CIDR notation and know what a prefix from `/8` to `/32` means
- You will be able to calculate by hand, from an address + prefix pair, the **network address**, the
  **broadcast address** and the **usable host range**
- You will be able to explain the `2^host_bits - 2` formula and why the **-2** is there (network + broadcast)
- You will be able to **split** a network into smaller subnets (subnetting) and explain why it is split
  (broadcast domain, security, management)
- You will be able to describe the idea of supernetting/route aggregation and why it shrinks router tables
- You will be able to say quickly whether a prefix **covers** a given address — firewall rules and routing
  stand on this
- **Cloud:** You will be able to explain the choice of a VPC CIDR, splitting subnets, **why AWS reserves 5
  addresses** from every subnet, and why a CIDR overlap makes peering impossible

---

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 2.1 | The subnet mask idea | `[mechanism]` | Where the boundary is drawn |
| 2.2 | CIDR notation | `[concept]` | The short form of the same thing — this is what is used in the field |
| 2.3 | Network, broadcast, usable range | `[application]` | **The heart of the phase** — arithmetic by hand |
| 2.4 | Subnetting: dividing a network | `[application]` | Exactly the same as cloud subnet design |
| 2.5 | Supernetting and route aggregation | `[concept]` | The basis of the table shrinking in Phase 4 |
| 2.6 | When this phase breaks | — | The signatures of mask faults |

> **How to work through this phase:** The commands are again 🟢 (`ip addr`, `ipcalc`, `ip route`) — none of
> them changes anything. But the real tool of this phase is **paper**. At the end of every section make up an
> address/prefix for yourself and calculate it; then check it with `ipcalc 10.0.4.130/26` (if it is
> installed). Use it as a checking tool, **not as a calculator** — if a tool is doing something you cannot do
> by hand, you do not have that skill. Memorise the four-step method in 2.3: *write the mask in binary → AND
> → find the network → fill the host bits.* Those four steps are the whole of this phase.

---
---

# 2.1 The Subnet Mask — What Draws the Boundary

## 2.1.1 The idea of the mask: ones are network, zeros are host `[mechanism]`

In Phase 1.2.3 we said an IP address is split into network and host parts. But you **cannot tell** where that
boundary is by looking at the address itself — `10.0.4.130` on its own says nothing.

There is a second value that tells you the boundary: the **subnet mask**. The mask is 32 bits just like the
IP, and the rule is simple:

- **Bits that are 1** → that position belongs to the **network** part
- **Bits that are 0** → that position belongs to the **host** part

And the critical constraint: **the ones are always on the left, in one unbroken block.**
`11111111.11111111.11111111.00000000` is valid; `11111111.00000000.11111111.00000000` is **invalid**. There is
one boundary and it cannot have holes.

The three most common masks:

| Mask (decimal) | Mask (binary) | Network bits |
|---|---|---|
| `255.0.0.0` | `11111111.00000000.00000000.00000000` | 8 |
| `255.255.0.0` | `11111111.11111111.00000000.00000000` | 16 |
| `255.255.255.0` | `11111111.11111111.11111111.00000000` | 24 |

Because `255 = 11111111` (all ones) and `0 = 00000000` (all zeros) (1.2.2), this table reads straight off.

## 2.1.2 The AND operation: finding the network address `[mechanism]`

The machine answers the question "is the destination on my network?" (1.2.3) in exactly this way: it **ANDs**
the IP with the mask bit by bit.

The AND operation works with one rule: **if both bits are 1 the result is 1, otherwise 0.**

```
IP:       00001010 . 00000000 . 00000100 . 10000010    (10.0.4.130)
Mask:     11111111 . 11111111 . 11111111 . 00000000    (255.255.255.0)
          ────────────────────────────────────────  AND
Network:  00001010 . 00000000 . 00000100 . 00000000    (10.0.4.0)
```

The result: `10.0.4.0`. This is called the **network address** — it is the identity of this network.

Notice: where the mask is 1 the IP's bit **passed through unchanged**; where it is 0 it was **zeroed**. That
is exactly the mask's job: to erase the host part and leave behind the network's identity.

And the machine's decision is this: **if its own network address and the destination's network address are the
same → same network.** It ANDs the destination with the same mask and compares the two. If they are equal,
direct delivery (Phase 3); if not, to the gateway (Phase 4).

> **🔧 See it on your machine** 🟢 — see your own mask and network address
>
> ```
> $ ip -4 addr show ens5 | grep inet
>     inet 172.31.20.15/20 brd 172.31.31.255 scope global dynamic ens5
>
> $ ip route | head -2
> default via 172.31.16.1 dev ens5 proto dhcp src 172.31.20.15
> 172.31.16.0/20 dev ens5 proto kernel scope link src 172.31.20.15
> ```
>
> The second `ip route` line shows **the network address of the machine's own network**: `172.31.16.0/20`.
> The machine calculated this itself — by ANDing the address `172.31.20.15` with the `/20` mask. The phrase
> `scope link` means "I reach this network directly, without a gateway". The `default via` on the first line
> means "everything else goes to this address" (Phase 4.1). These two lines are the concrete form, on the
> machine, of the decision in 1.2.3.

> **⚠️ Common misconception: "The subnet mask is part of the address."**
>
> No — the mask is **a separate 32-bit value** and it is not carried in the packet. An IP packet's header has a
> source IP and a destination IP, **no mask**. The mask is only for the machine's own local decision: "is this
> destination on my network?" Routers use the prefixes in their own tables too, they do not read them from the
> packet. The practical consequence is this: if you give a machine the wrong mask, **the packet still goes**
> but the machine sends it **to the wrong place** — what should go to the gateway goes to a neighbour, what
> should go to a neighbour goes to the gateway. The mask is not an address, it is a **decision rule**.

> **🤔 Think 2.1** — You have the address `192.168.5.77` and the mask `255.255.255.0`. (a) Find the network
> address by doing the AND in binary. (b) What would the network address be if the same address were given
> with the mask `255.255.0.0`? (c) In the two cases, would the address `192.168.9.4` count as being "on the
> same network" — answer separately for each.
>
> *(Answer: at the end of the phase)*

---
---

# 2.2 CIDR Notation

## 2.2.1 /nn: the short form of the mask `[concept]`

Writing `255.255.255.0` is long and tiring to read. Since the ones in the mask are always on the left and
unbroken (2.1.1), the only thing that needs saying is **how many ones there are**.

That is exactly what **CIDR** notation is (Classless Inter-Domain Routing): a slash and the number of network
bits at the end of the address.

```
192.168.1.10 / 255.255.255.0     ≡     192.168.1.10/24
```

The two say **exactly the same** thing. In the field, in documentation and in cloud consoles, CIDR notation is
used almost always.

The conversion table — understand it once, you do not need to memorise it:

| CIDR | Mask (decimal) | Host bits | Usable hosts |
|---|---|---|---|
| `/8` | `255.0.0.0` | 24 | 16,777,214 |
| `/16` | `255.255.0.0` | 16 | 65,534 |
| `/24` | `255.255.255.0` | 8 | 254 |
| `/25` | `255.255.255.128` | 7 | 126 |
| `/26` | `255.255.255.192` | 6 | 62 |
| `/27` | `255.255.255.224` | 5 | 30 |
| `/28` | `255.255.255.240` | 4 | 14 |
| `/29` | `255.255.255.248` | 3 | 6 |
| `/30` | `255.255.255.252` | 2 | 2 |
| `/32` | `255.255.255.255` | 0 | 1 (a single address) |

Recognise the two extremes: **`/32`** expresses a single address (used in firewall rules to say "only this
machine"), and **`/0`** covers **every** address (in Phase 4 it will come at you as `0.0.0.0/0` = "the default
route").

And the golden rule that comes out of this table: **the bigger the prefix, the smaller the network.** A `/24`
is a smaller network than a `/16`. It feels counter-intuitive but it is logical: the prefix is the number of
network bits — the more bits go to the network, the fewer are left for hosts.

![Figure 2.1 — Subnet mask and CIDR: the same 32-bit address split into network and host parts with the /16, /24 and /26 prefixes. As the prefix grows the network part extends to the right, the host part narrows, and the number of usable addresses in the network halves.](../diagrams/png/nw-2-01-cidr-prefix.png)

In the figure the three rows show the same address; the only thing that changes is **where** the prefix line
stands. The blue area to the left of the line is the network part, the grey area to its right is the host
part. The number on the right of each row shows how many machines count as being "on the same network" with
that prefix — notice that this number halves when the line moves one bit to the right.

> **💡 Cloud connection — the VPC CIDR choice is an irreversible decision:** When you create a VPC in AWS you
> give it a CIDR block — for example `10.0.0.0/16` (65,536 addresses). This block **cannot be narrowed** for
> the lifetime of the VPC (an extra block can be added to widen it, but the original block does not change).
> In practice two mistakes are expensive: (1) **choosing too narrow** — a `/24` VPC with 254 addresses fills
> up in a few months and there is no room left to add subnets; (2) **choosing a block that overlaps other
> networks** — if your office network is `192.168.1.0/24` and you give your VPC the same block, when you set
> up a VPN both sides say "this address is on my network" (2.1.2) and traffic never passes. For the same
> reason two VPCs with overlapping CIDRs **cannot be peered** — AWS rejects the request outright. That is why
> organisations allocate disjoint blocks to VPCs from the start: `10.0.0.0/16` prod, `10.1.0.0/16` staging,
> `10.2.0.0/16` dev.

> **🤔 Think 2.2** — A firewall rule has `10.0.0.0/8` written as its source. (a) Does this rule cover the
> address `10.5.200.13`? (b) Does it cover `172.16.0.5`? (c) If you made the same rule `10.0.0.0/16`, would
> the answer to (a) change — and why?
>
> *(Answer: at the end of the phase)*

---
---

# 2.3 Network Address, Broadcast and the Usable Range

This is the heart of the phase. You will learn a four-step method, and that method will work for **every**
`a.b.c.d/nn` you are given.

## 2.3.1 The three boundary values of a network `[concept]`

In every network there are two special addresses that **cannot be given** to hosts:

- **The network address** — host bits **all 0**. It is the network's identity and cannot be assigned to a
  device. (in the `10.0.4.0/24` example: `10.0.4.0`)
- **The broadcast address** — host bits **all 1**. A packet sent to this address goes to **everyone** on that
  network. (`10.0.4.255`)

Every address in between is a **usable host address**. (`10.0.4.1` – `10.0.4.254`)

This is where the famous **-2** comes from:

```
number of usable hosts = 2^(number of host bits) − 2
```

For `/24`: host bits 32−24 = 8 → 2⁸ = 256 → 256−2 = **254**.

## 2.3.2 The calculation in four steps `[application]`

Let us work through an example: **`10.0.4.130/26`**

**Step 1 — Write the mask in binary.** `/26` → 26 ones, then 6 zeros:

```
11111111 . 11111111 . 11111111 . 11000000     →  255.255.255.192
```

Only the last octet is interesting: `11000000` = 128+64 = **192**.

**Step 2 — Write the address in binary and AND.**

```
IP:       00001010 . 00000000 . 00000100 . 10000010    (10.0.4.130)
Mask:     11111111 . 11111111 . 11111111 . 11000000
          ────────────────────────────────────────  AND
Network:  00001010 . 00000000 . 00000100 . 10000000    (10.0.4.128)
```

**Network address: `10.0.4.128`**

**Step 3 — Find the broadcast by making the host bits 1.**

```
Network:    ... . 10000000    (10.0.4.128)
Broadcast:  ... . 10111111    (10.0.4.191)   ← the last 6 bits all 1
```

`10111111` = 128 + 32+16+8+4+2+1 = **191**. **Broadcast: `10.0.4.191`**

**Step 4 — Write the range and the count.**

- Usable range: **`10.0.4.129` – `10.0.4.190`**
- Host count: 2⁶ − 2 = 64 − 2 = **62**

Summary table:

| Value | Result |
|---|---|
| Given | `10.0.4.130/26` |
| Mask | `255.255.255.192` |
| Network | `10.0.4.128` |
| First host | `10.0.4.129` |
| Last host | `10.0.4.190` |
| Broadcast | `10.0.4.191` |
| Host count | 62 |

## 2.3.3 The fast method: block size `[application]`

Writing binary is safe but slow. Experienced engineers use the **block size** method:

```
block size = 256 − (the mask value in the relevant octet)
```

For `/26` the mask's last octet is 192 → block = 256 − 192 = **64**.

So this prefix forms blocks of 64 in the last octet: **0, 64, 128, 192**. If the address's last octet is 130,
the block containing 130 is **128** (because 128 ≤ 130 < 192). Network = `10.0.4.128`, broadcast = one less
than the next block = `10.0.4.191`. The same result, in three seconds.

Frequently used block sizes:

| Prefix | Mask's last octet | Block size | Blocks (last octet) |
|---|---|---|---|
| `/24` | 0 | 256 | 0 |
| `/25` | 128 | 128 | 0, 128 |
| `/26` | 192 | 64 | 0, 64, 128, 192 |
| `/27` | 224 | 32 | 0, 32, 64, 96, ... |
| `/28` | 240 | 16 | 0, 16, 32, 48, ... |
| `/30` | 252 | 4 | 0, 4, 8, 12, ... |

**Advice:** first be able to do the binary method (2.3.2) with confidence, then move to the block method. The
block method is a shortcut; if you do not know what is underneath it, you will get an unusual prefix (e.g.
`/19`) wrong.

> **🔧 See it on your machine** 🟢 — check your arithmetic
>
> ```
> $ ipcalc 10.0.4.130/26
> Address:   10.0.4.130           00001010.00000000.00000100.10 000010
> Netmask:   255.255.255.192 = 26 11111111.11111111.11111111.11 000000
> =>
> Network:   10.0.4.128/26        00001010.00000000.00000100.10 000000
> HostMin:   10.0.4.129           00001010.00000000.00000100.10 000001
> HostMax:   10.0.4.190           00001010.00000000.00000100.10 111110
> Broadcast: 10.0.4.191           00001010.00000000.00000100.10 111111
> Hosts/Net: 62
> ```
>
> If `ipcalc` is not installed: `sudo apt install ipcalc`. The space in the output shows exactly the **prefix
> boundary** — network to its left, host to its right. Use this tool for **checking**, not for calculating. Do
> it on paper first, then confirm here. When you get the same result on three or four examples, the method has
> settled in you.

> **❓ A question that comes to mind: "Why do `/31` and `/32` look like they break the '-2' rule?"**
>
> Because both are special cases. **`/32`**: there are no host bits, it expresses a single address — not a
> network but a **pointer**. It is used in firewall rules ("allow only `203.0.113.7/32`") and in routing
> tables ("this single address goes that way"); the -2 rule is meaningless here. **`/31`**: it gives 2
> addresses, and by the rule 0 hosts would be left — but RFC 3021 makes a special exception: because
> broadcast is not needed on **point-to-point** links between two routers, both addresses can be used.
> Formerly `/30` was used for this job (4 addresses, 2 usable, 2 wasted); `/31` removes that waste. In the
> cloud you will see `/32` every day, `/31` rarely.

> **🤔 Think 2.3** — Apply the four steps to `192.168.10.200/27`: (a) the mask in decimal, (b) the network
> address, (c) the broadcast address, (d) the usable range and the host count. Do it first with the binary
> method, then with the block size method, and confirm that the two results are the same.
>
> *(Answer: at the end of the phase)*

---
---

# 2.4 Subnetting — Dividing a Network

## 2.4.1 Why we divide `[concept]`

You have `10.0.0.0/16` — 65,534 usable addresses. Why not leave it all as a single network instead of
splitting it into small pieces?

Four reasons, all practical:

**1. Shrinking the broadcast domain.** A broadcast in a network (2.3.1) goes to **everyone** on that network.
In a single network of 65,000 devices, every ARP request (Phase 3.1) disturbs 65,000 devices. That chokes the
network. Dividing limits the noise.

**2. Drawing a security boundary.** Traffic between different subnets has to pass through a router (or, in the
cloud, through a route table + security group) — that is, it is **inspectable**. Traffic within the same
subnet, on the other hand, flows directly, with no control point in between. That is why you put the database
in a separate subnet.

**3. Management and readability.** An arrangement of the form `10.0.1.x = web`, `10.0.2.x = app`,
`10.0.3.x = db` tells you a machine's role by looking at the address. This is valuable both for humans and for
automation.

**4. Geographic/physical separation.** In the cloud every subnet belongs to **a single availability zone**. If
you want high availability you must have subnets in at least two AZs — so you have to divide.

## 2.4.2 How to divide: borrowing host bits `[application]`

Subnetting has a single move: **borrow bits from the host part and add them to the network part.**

Example: let us divide `10.0.1.0/24` into four equal pieces.

Four pieces need 2 bits (2² = 4). We increase the prefix by 2: `/24` → **`/26`**.

Block size (2.3.3): 256 − 192 = 64. So the blocks advance 64 at a time:

| # | Subnet | Network | Usable range | Broadcast | Hosts |
|---|---|---|---|---|---|
| 1 | `10.0.1.0/26` | `10.0.1.0` | `.1` – `.62` | `10.0.1.63` | 62 |
| 2 | `10.0.1.64/26` | `10.0.1.64` | `.65` – `.126` | `10.0.1.127` | 62 |
| 3 | `10.0.1.128/26` | `10.0.1.128` | `.129` – `.190` | `10.0.1.191` | 62 |
| 4 | `10.0.1.192/26` | `10.0.1.192` | `.193` – `.254` | `10.0.1.255` | 62 |

Two things stand out. **The total host count went down:** a single `/24` gives 254 hosts, while four `/26`s
give 248 in total. The loss comes from each subnet "eating" its own network + broadcast address — 4 subnets ×
2 addresses = 8, but the original already lost 2, so the net loss is 6. **Dividing has a price.**

The second: **the division does not always have to be equal.** In real designs subnets of different sizes are
used (this is called VLSM — Variable Length Subnet Masking). You can give the web tier a `/24` and the
database a `/28`. The only rule: **subnets must not overlap with each other.**

The formulas — both are needed and they must not be confused:

```
number of subnets  = 2^(bits borrowed)
hosts per subnet   = 2^(remaining host bits) − 2
```

> **💡 Cloud connection — AWS takes 5 addresses from every subnet:** When you create a subnet in your own VPC,
> you **do not get** the host count you expect. A `/24` subnet gives not 254 but **251** usable addresses. The
> reason: AWS reserves 5 addresses in every subnet — `.0` the network address (standard), `.1` the VPC router
> (that is, your default gateway, Phase 4.1), `.2` the DNS server (Phase 6), `.3` reserved for future use, and
> `.255` broadcast (standard). The practical consequence: in small subnets this loss grows out of proportion —
> a `/28` subnet gives you only **11** of its 16 addresses. That is why in the cloud you cannot create a
> subnet smaller than `/28` (AWS does not allow it) and in practice it rarely makes sense to go below `/24`.

> **🤔 Think 2.4** — You need to divide the block `172.16.0.0/16` into subnets that each take at least 500
> hosts. (a) How many host bits are needed at minimum per subnet? (b) What does the prefix become? (c) How
> many subnets do you get? (d) How many usable addresses are there in each subnet — and why is this more than
> 500?
>
> *(Answer: at the end of the phase)*

---
---

# 2.5 Supernetting and Route Aggregation

## 2.5.1 The other direction: combining `[concept]`

Subnetting divides a network. **Supernetting** (or route aggregation / route summarization) does exactly the
opposite: it expresses several adjacent networks with **a single prefix**.

Example: you have four networks —

```
10.0.0.0/24
10.0.1.0/24
10.0.2.0/24
10.0.3.0/24
```

All of them can be expressed with `10.0.0.0/22`. Why? Look at the third octet: 0, 1, 2, 3 → `00000000`,
`00000001`, `00000010`, `00000011`. The first **6 bits** are the same in all four. 8+8+6 = 22 → `/22`.

The rule: **the smaller you make the prefix, the more addresses you cover.**

## 2.5.2 Why it matters: shrinking router tables `[concept]`

The practical value of this will settle fully in Phase 4, but let us plant the seed now.

A router keeps a row in its table for every network it knows about. The routers on the internet backbone have
hundreds of thousands of rows in their tables, and that table is consulted for every packet. The bigger the
table, the higher the memory and lookup cost.

Aggregation solves this: instead of the four rows above, **a single row** (`10.0.0.0/22`) is kept. At internet
scale this means millions of rows coming down to tens of thousands. That was the main
reason CIDR was invented in the 1990s — the old "class" (class A/B/C) system did not allow aggregation and
router tables had come to bursting point.

The **longest prefix match** rule you will see in Phase 4 is born here too: if the table has both
`10.0.0.0/22` and `10.0.2.0/24`, a packet destined for `10.0.2.5` uses the **longer** (`/24`) row — because it
is more specific.

> **🤔 Think 2.5** — You have `192.168.8.0/24`, `192.168.9.0/24`, `192.168.10.0/24`, `192.168.11.0/24`.
> (a) Express them with a single prefix. (b) Does that prefix also cover `192.168.12.0/24`? (c) If you wrote
> `192.168.8.0/21`, which extra networks would you have covered, and why might that be a risk?
>
> *(Answer: at the end of the phase)*

---
---

# 2.6 When This Phase Breaks — The Signatures of Mask Faults

The most insidious thing about mask faults is this: **the network does not die completely.** Some destinations
work, some do not. That partialness makes diagnosis harder — but it is also the signature: when you hear the
sentence "I can reach some places but not others", the first thing that should come to mind is the **prefix**.

| Symptom | Probable cause | Confirm | Related section |
|---|---|---|---|
| Some machines on the same network are unreachable | The mask is too narrow (`/26` where it should be `/24`) | The prefix in `ip addr`, AND the destination | 2.1.2 |
| The gateway is unreachable, "network unreachable" | The gateway falls outside the network according to the mask | `ip route`, is the gateway in the same subnet | 2.1.2 |
| One-way traffic: it goes, it does not come back | **Different** masks on the two ends (asymmetric neighbourhood) | `ip addr` on both machines | 2.1.2 |
| A new subnet cannot be created | The parent block is used up / overlapping CIDR | List the existing subnet CIDRs | 2.4.2 |
| VPC peering is rejected | The two VPCs' CIDRs overlap | Compare the VPC CIDR blocks | 2.2.1 |
| Fewer IPs usable than expected | In the cloud, 5 reserved addresses + network/broadcast | The subnet's CIDR and the reservation rule | 2.4.2 |
| A firewall rule allows unexpected traffic | The prefix is too wide (it should have been `/24` not `/16`) | Calculate the coverage of the CIDR in the rule | 2.2.1 |
| The address from DHCP is on the wrong network | The mask of the DHCP scope is wrong | The DHCP server configuration | Phase 1.6 |
| The destination does not match in the route table | The prefixes overlap, the wrong row is picked | `ip route get <destination>` | 2.5.2 (Phase 4) |
| The same IPs collide in two environments | No CIDR planning was done before VPN/peering | A CIDR inventory of all environments | 2.2.1 |

> **The lesson from this table:** A mask **does not produce packets — it produces decisions.** A wrong mask
> runs the machine with a "lying neighbourhood map": it thinks a remote machine is a neighbour (it looks for
> it with ARP, nobody answers, timeout), and it thinks its neighbour is remote (it sends the packet to the
> gateway, which sometimes redirects it back and sometimes drops it). That is why the classic signature of a
> mask fault is **partial reachability**, not a total outage. And its most insidious form is **asymmetric**:
> if two machines' masks differ, A says "B is my neighbour" while B says "A is far away"; the outbound packet
> arrives, the return packet goes another way or never comes back. That is why, in a network fault, check the
> mask **on both ends** — looking at one side misleads you. The practical reflex: when you hear "some things
> work, some do not", let your first calculation be an AND with the prefix from `ip addr`.

---
---

# Phase 2 — Answers to the Think questions

## Answer 2.1 — Same address, different mask, different neighbourhood

(a) AND with `255.255.255.0` = `/24`:

```
IP:      11000000.10101000.00000101.01001101   (192.168.5.77)
Mask:    11111111.11111111.11111111.00000000
Network: 11000000.10101000.00000101.00000000   (192.168.5.0)
```
**Network address: `192.168.5.0`**

(b) AND with `255.255.0.0` = `/16`: the first two octets pass through unchanged, the last two are zeroed →
**Network address: `192.168.0.0`**

(c) For `192.168.9.4`:
- **In the `/24` case:** the destination's network address is `192.168.9.0`, ours is `192.168.5.0` →
  **different network**. The machine sends the packet to the gateway.
- **In the `/16` case:** the destination's network address is `192.168.0.0` and so is ours → **same network**.
  The machine tries to deliver the packet directly (looking for the MAC with ARP, Phase 3.1).

Here is the answer to Phase 1's last question: **the same address, a different prefix, a completely different
decision.** And if two machines have been given different prefixes, one may say "we are neighbours" while the
other says "we are far apart" — that is exactly the asymmetric fault in 2.6.
**Related section:** 2.1.1-2.1.2 · **Next:** 2.6, Phase 3.1 (ARP), Phase 4.1 (the gateway decision).

## Answer 2.2 — The prefix determines the coverage

(a) **Yes.** `10.0.0.0/8` → only the first octet (10) is fixed, the remaining 24 bits are free. Since the
first octet of `10.5.200.13` is 10, it is within the coverage. `/8` covers all ~16.7 million addresses between
`10.0.0.0` and `10.255.255.255`.

(b) **No.** The first octet of `172.16.0.5` is 172, not 10. Outside the coverage. (Both are RFC 1918 private
addresses but from different blocks — 1.3.1.)

(c) **Yes, it changes.** `10.0.0.0/16` → the first **two** octets are fixed: `10.0.x.x`. The second octet of
`10.5.200.13` is 5, not 0 → it **falls outside the coverage**. The reason: the bigger the prefix the smaller
the network (2.2.1), that is, the rule becomes narrower and more specific. This is the source of the most
common mistake when writing firewall rules: a prefix that is too wide also allows sources you did not want
(the 2.6 table).
**Related section:** 2.2.1 · **Next:** Phase 9.2 (security rules), Phase 11.5 (SG/NACL).

## Answer 2.3 — `192.168.10.200/27`

(a) **Mask:** `/27` → 27 bits of 1, 5 bits of 0. The last octet `11100000` = 128+64+32 = **224**. Mask =
**`255.255.255.224`**

(b) **Network:** the binary method — 200 = `11001000`. AND the last octet with the mask's last octet:
`11001000 AND 11100000 = 11000000` = 192. **Network = `192.168.10.192`**

(c) **Broadcast:** make the host bits (the last 5) 1: `11011111` = 128+64+16+8+4+2+1 = 223.
**Broadcast = `192.168.10.223`**

(d) **Range:** `192.168.10.193` – `192.168.10.222`. **Host count:** 2⁵ − 2 = 32 − 2 = **30**

**Check with the block method:** block = 256 − 224 = 32. Blocks: 0, 32, 64, 96, 128, 160, **192**, 224. Which
block does 200 fall into? 192 ≤ 200 < 224 → block **192**. Network `.192`, broadcast one less than the next
block, `.223`. **The two methods gave the same result** — verification complete.
**Related section:** 2.3.2-2.3.3 · **Next:** 2.4.2 (subnetting), Phase 11.2 (VPC subnet design).

## Answer 2.4 — `/23` for 500 hosts

(a) For 500 hosts: 2⁸ − 2 = 254 → **not enough**. 2⁹ − 2 = 510 → **enough**. So a minimum of **9 host bits**
is needed. (Do not forget the -2 in the formula — it is not 2⁹ = 512, the usable count is 510.)

(b) 32 − 9 = **`/23`**

(c) We went from `/16` to `/23` → 23 − 16 = **we borrowed 7 bits**. Number of subnets = 2⁷ = **128 subnets**.

(d) Each subnet has 2⁹ − 2 = **510 usable addresses**. More than 500, because prefixes advance in steps that
are **powers of two** — you cannot create a subnet of exactly 500. Because `/24` (254) is not enough you have
to go up one step (`/23`, 510) and 10 addresses are "wasted". This is an unavoidable property of subnet
design: **you always round up to the next power of two above what you need.**

Note: in the cloud 5 reserved addresses are added to this calculation too (the cloud box in 2.4.2) → a `/23`
in an AWS subnet gives 507 usable addresses; it still covers 500.
**Related section:** 2.4.2 · **Next:** Phase 11.2 (VPC subnet planning).

## Answer 2.5 — Aggregating with `/22`, over-covering with `/21`

(a) The third octets: 8, 9, 10, 11 → `00001000`, `00001001`, `00001010`, `00001011`. The first **6 bits** are
the same in all four (`000010`). Total: 8 + 8 + 6 = **22**. Answer: **`192.168.8.0/22`**

(b) **No.** The coverage of `/22` is `192.168.8.0` – `192.168.11.255`. `192.168.12.0` is outside that range
(the third octet 12 = `00001100`, its first 6 bits `000011` — different).

(c) The coverage of `/21` would be `192.168.8.0` – `192.168.15.255`; that is, it would also cover the networks
**`192.168.12.0/24` – `192.168.15.0/24`**. The risk goes two ways: (1) if those networks belong to **someone
else**, you have pulled the traffic destined for them to yourself (a "black hole" in routing — Phase 4.6);
(2) if you used this prefix in a **firewall rule**, you have also allowed four networks you did not want to
allow. Aggregation is useful but **over-aggregating is dangerous** — always calculate the range you cover.
**Related section:** 2.5.1-2.5.2 · **Next:** Phase 4.3 (longest prefix match), Phase 9.2 (rule coverage).

---
---

# Phase 2 — Frequently asked questions

**Q1 — Why is the mask not carried in the packet?** Because the mask is for **the sender's local decision**,
it is not something the receiver needs to know. The sender answers the question "is this destination on my
network" with its mask and sends the packet either directly or to the gateway. Routers use the prefixes in
their own tables too. Carrying a mask inside the packet would take up space for nothing and would be of no
use to any receiver (2.1.2).

**Q2 — Is the "class" (class A/B/C) system still valid?** No. It used to be that the mask was determined
**automatically** by looking at the first octet (A: `/8`, B: `/16`, C: `/24`). That system was abandoned in
1993 with CIDR because it was inflexible and led to terrible waste (an organisation was given 16 million
addresses). Today everything is **classless**: the prefix is stated explicitly. You may see class terms in old
documents but do not use them in planning (2.2.1).

**Q3 — Why do we always do `-2`?** Because in every network two addresses cannot be given to hosts: the
**network address**, whose host bits are all 0 (the network's identity), and the **broadcast address**, whose
host bits are all 1 (a broadcast to everyone). The exceptions: `/31` (RFC 3021, point-to-point links) and
`/32` (a single-address pointer) (2.3.1, the box in 2.3.3).

**Q4 — Why can I not make a subnet smaller than `/28` in the cloud?** Because AWS reserves 5 addresses from
every subnet (network, VPC router, DNS, future use, broadcast). A `/28` has 16 addresses; take away 5 and 11
are left. In a smaller subnet (e.g. `/29`, 8 addresses) the usable count would drop to 3 — not practical. That
is why `/28` is the lower limit (2.4.2).

**Q5 — What is VLSM, when is it used?** Variable Length Subnet Masking: dividing a block into **unequal**
pieces. For example you can split `10.0.0.0/16` as `/22` for web, `/24` for app and `/26` for db. Real designs
are VLSM almost always, because the tiers have different needs. The only rule: the pieces **must not overlap**
(2.4.2).

**Q6 — If two VPCs' CIDRs overlap, can really nothing be done?** Peering and a direct VPN become impossible —
because both sides count those addresses as "my own network" (2.1.2) and traffic never leaves. There are
partial solutions (translating addresses with NAT, publishing a single service with PrivateLink — Phase 11.7.1)
but all of them are extra complexity. **The right solution is to plan from the start**: allocate disjoint CIDR
blocks to environments (2.2.1).

**Q7 — Can I change the prefix later?** Changing a machine's prefix is easy (one configuration line). But
changing a **network's** prefix is hard: the configuration of every machine in that network, the route tables,
the firewall rules and the DNS records are all affected. In the cloud a VPC CIDR **cannot be narrowed**; an
extra block can be added to widen it, but that complicates the subnet plan too. That is why the CIDR choice is
one of those decisions that **must be got right at the start** (2.2.1).

---
---

# Phase 2 — Test yourself

Write your answers on paper, then compare them with the answer key. Target: 14+ out of 18.
**This phase has calculation questions — write the intermediate steps too, not just the result.**

## Part A — Definition and mechanism

1. How many bits is a subnet mask, and what do the 1/0 bits express?
2. Why must the ones in the mask be unbroken and on the left?
3. What is the rule of the AND operation, and what is it applied to in order to find the network address?
4. What is the decimal mask equivalent of the `/26` prefix? How did you find it?
5. Justify the sentence "the bigger the prefix, the smaller the network" in one sentence.
6. Which two addresses in every network cannot be given to hosts, and why not?
7. Where does the `-2` in the `2^host_bits − 2` formula come from? Write both exceptions too.
8. What is supernetting and why is it valuable for routers?

## Part B — Calculate

9. For `10.20.30.40/24`: network, broadcast, usable range, host count.
10. For `192.168.1.100/26`: mask (decimal), network, broadcast, host count.
11. For `172.16.5.200/28`: the network address and the broadcast address. Use the block size method.
12. Divide the block `10.0.1.0/24` into **8 equal** subnets. What is the new prefix, how many hosts does each
    subnet take, and what are the network addresses of the first three subnets?
13. An application needs 100 hosts. Which prefix do you choose with the least waste, and how many addresses
    are wasted?
14. Express the networks `10.0.16.0/24`, `10.0.17.0/24`, `10.0.18.0/24`, `10.0.19.0/24` with a single prefix.

## Part C — Reasoning and diagnosis

15. One machine is configured as `10.0.1.50/24`, the other as `10.0.1.80/26`. Which machine counts the other
    as a "neighbour" and which does not? What is the result?
16. A firewall rule allows the source `10.0.0.0/16`. The security team says "only `10.0.5.x` should have
    access". What should the new prefix be?
17. Why does an AWS `/24` subnet have 251 usable addresses instead of 254? List the five reserved addresses.
18. For a user who says "I can reach some machines but not others", which value do you check first and why?

---

## Answer key

1. **32 bits.** The 1 bits mark the network part, the 0 bits the host part (2.1.1). — 2. Because the
   network/host boundary is a **single** point; a boundary with holes is meaningless and the AND operation
   would not produce a consistent network address (2.1.1). — 3. **If both bits are 1 then 1, otherwise 0.** It
   is applied bit by bit to the IP address and the subnet mask; the result is the network address (2.1.2). —
   4. `/26` → 26 bits of 1: the first three octets 255, the last octet `11000000` = 128+64 = 192 →
   **`255.255.255.192`** (2.3.2). — 5. Because the prefix is the number of network bits; the more bits are
   given to the network, the fewer are left for hosts, so the network covers fewer addresses (2.2.1). —
   6. The **network address** (host bits all 0, the network's identity) and the **broadcast address** (host
   bits all 1, a broadcast to everyone). Both have a special meaning and cannot be assigned to a device
   (2.3.1). — 7. From those two addresses (network + broadcast). The exceptions: **`/31`** (RFC 3021, both
   addresses are used on point-to-point links) and **`/32`** (a single-address pointer, not a network)
   (2.3.1, 2.3.3). — 8. Expressing several adjacent networks with a single, shorter prefix. Its value: it
   reduces the number of rows in router tables — lowering memory and lookup cost (2.5.1-2.5.2).

9. Network **`10.20.30.0`**, broadcast **`10.20.30.255`**, range **`.1` – `.254`**, hosts **254**
   (2.3.1-2.3.2). — 10. Mask **`255.255.255.192`**; block = 64, 100 → block 64; network **`192.168.1.64`**,
   broadcast **`192.168.1.127`**, hosts **62** (2.3.2-2.3.3). — 11. `/28` → the mask's last octet 240, block =
   256−240 = **16**. Blocks: 0,16,...,192, **208**, 224... 200 → 192 ≤ 200 < 208, block **192**. Network
   **`172.16.5.192`**, broadcast **`172.16.5.207`** (2.3.3). — 12. For 8 subnets borrow 3 bits: `/24` + 3 =
   **`/27`**. Hosts = 2⁵ − 2 = **30**. Block = 32. The first three networks: **`10.0.1.0`, `10.0.1.32`,
   `10.0.1.64`** (2.4.2). — 13. For 100 hosts 2⁶−2 = 62 is not enough, 2⁷−2 = 126 is → 7 host bits →
   **`/25`**. Wasted: 126 − 100 = **26 addresses** (2.4.2). — 14. The third octets 16,17,18,19 →
   `00010000`...`00010011`, the first 6 bits in common → 8+8+6 = **`10.0.16.0/22`** (2.5.1).

15. **The `/24` machine** (`10.0.1.50`) counts the other as a neighbour: its network is `10.0.1.0` and the
    destination `10.0.1.80` is in that range. **The `/26` machine** (`10.0.1.80`) does not: its own network is
    `10.0.1.64` (block 64, range 64–127) and `10.0.1.50` is outside that range → it counts it as remote and
    sends the packet to the gateway. The result: **asymmetric traffic** — one tries to reach the other
    directly with ARP, the other sends to the gateway; the connection is either never established or works
    one way only (2.1.2, 2.6). — 16. **`10.0.5.0/24`** — it covers only the range `10.0.5.0` – `10.0.5.255`
    (2.2.1). — 17. AWS reserves 5 addresses in every subnet: **`.0`** the network address, **`.1`** the VPC
    router (default gateway), **`.2`** the DNS server, **`.3`** reserved for future use, **`.255`**
    broadcast. 256 − 5 = **251** (2.4.2). — 18. **The prefix (the subnet mask)** — the `/nn` value in the
    `ip addr` output, and **on both ends**. The reason: partial reachability is the classic signature of a
    mask fault; a wrong mask pushes the machine into counting some destinations as neighbours and others as
    remote, and the decision comes out wrong (2.6).

## Scoring

| Correct | What it means |
|---|---|
| 16-18 | The prefix arithmetic has settled in you. You are ready for Phase 3 and later for Phase 11 (VPC). |
| 13-15 | Good. **Recalculate** what you missed in Part B, do not just read it. |
| 9-12 | The concept is there, the skill is missing. Apply the four steps in 2.3.2 again with 5 different addresses. |
| 0-8 | **A stopping point.** Do not move to Phase 3; work through 2.1–2.3 from the start and calculate 10 examples. |

Question you missed → section to go back to:

| Question | Section |
|---|---|
| 1, 2, 3, 15 | 2.1 The subnet mask and AND |
| 4, 5, 16 | 2.2 CIDR notation |
| 6, 7, 9, 10, 11 | 2.3 Network/broadcast/range |
| 12, 13, 17 | 2.4 Subnetting |
| 8, 14 | 2.5 Supernetting |
| 18 | 2.6 The fault table |

---
---

# Phase 2 — Closing and Bridge to Phase 3

## What you carry from this phase

Phase 2 gave you a **skill**, not a piece of knowledge: being able to work out the boundaries of a network on
paper when you see an `a.b.c.d/nn`. You learned that the subnet mask marks the network part with ones and the
host part with zeros; that the AND operation produces the network address; why the network and broadcast
addresses cannot be given to hosts; and that the network shrinks as the prefix grows. You acquired the
four-step method and the block-size shortcut. You saw how to divide a network (subnetting) and how to combine
networks (aggregation).

The most lasting sentence is this: **a mask does not produce packets, it produces decisions** — and that
decision is the answer to a single question: *"is the destination on my network?"*

## Where Phase 3 connects to this

So far we have always arrived at the same place and always stopped there. The machine has made the decision
"the destination is on my network" (Phase 2). So **what happens next?**

The answer is this: the machine knows the destination's IP but **does not know its MAC** (Phase 1.1). And MAC
is required for L2 delivery. The mechanism that fills this gap is called **ARP** and it is the first topic of
Phase 3.

In Phase 3 you will also see how switches learn their MAC table, the difference between a broadcast domain and
a collision domain, and how a VLAN divides a single physical switch into logical networks — that is, the
**physical-world counterpart** of the boundaries you drew mathematically in Phase 2.

Phase 2 drew the boundary; Phase 3 shows what is inside it.

> **🤔 Phase output — ask yourself:** The machine said "the destination is on my network" and decided to
> deliver directly. But all it has is the destination's **IP**, not its MAC. To build the frame it has to fill
> in the destination MAC field. How can it learn a MAC it does not know — and whom can it ask, when it does
> not know the address to ask either? (Hint: in Phase 1.1.1 we mentioned the address `ff:ff:ff:ff:ff:ff`.)
>
> **🧪 Lab 2 idea (all 🟢, paper + terminal):** (1) Get your own IP and prefix with `ip -4 addr`; apply the
> four steps and calculate the network, the broadcast and the range **on paper**. (2) Compare it with the
> `scope link` line in the `ip route` output — is the kernel's calculation the same as yours? (3) Confirm with
> `ipcalc <your IP>/<prefix>`. (4) Make up 5 random address/prefix pairs for yourself (in different sizes such
> as `/22`, `/26`, `/28`, `/30`, `/19`) and calculate all of them first with the binary method, then with the
> block method; time yourself — the target is 10 seconds per address. (5) Run `ip route get 8.8.8.8` and
> `ip route get <an IP on your own network>`; explain, based on 2.1.2, why the outputs differ (one contains
> `via`, the other does not). That fifth step is the seed of Phase 4.

---

> **Navigation:** [◀ Phase 1 — Addressing](Phase_1_Addressing.md) · **Phase 2** · [Checkpoint Quiz 1 ▶](Checkpoint_Quiz_1.md)
