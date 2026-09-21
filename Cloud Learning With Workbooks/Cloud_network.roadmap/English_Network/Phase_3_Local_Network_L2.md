# Phase 3 — The Local Network (L2): Delivery to the Neighbour

> **Navigation:** [◀ Checkpoint Quiz 1](Checkpoint_Quiz_1.md) · **Phase 3** · [Phase 4 — Routing (L3) ▶](Phase_4_Routing_L3.md)

---

## Where we come from

At the end of Phase 2 we left the machine at the exact moment of a decision: it decided the destination is
**on its own network**, so it will deliver directly. But all it holds is the destination's **IP** — not the
**MAC** it needs in order to build the L2 frame (Phase 1.1).

This phase begins by filling that gap. You brought three things with you:

- **MAC is the address of L2 and it only has meaning on the local network** (1.1.2). In this phase you will
  see what "local network" means physically.
- **`ff:ff:ff:ff:ff:ff` = broadcast** (1.1.1). The first mechanism of Phase 3 uses exactly this.
- **The prefix draws the "same network" boundary** (2.1.2). Phase 3 introduces the hardware counterpart of
  that logical boundary — the **broadcast domain**.

## The question of this phase

> *"My machine knows the destination's IP but not its MAC. How does it learn an address it does not know,
> when it does not even know whom to ask? And once that frame is on the wire, how does the switch send it
> to the right port?"*

In this phase you descend into the **physical reality** of the network. Phases 1 and 2 were about addresses
and arithmetic; here there are cables, switches and broadcast domains.

And what you learn pays off directly in the field: why an ARP table goes "dirty", why a switch sometimes
sprays a frame out of every port, why a VLAN can split one cable into several networks. In the cloud this
layer is largely **hidden** from you — but hidden does not mean gone, and you can only diagnose a failure in
a hidden layer if you know how it works.

---

## By the end of this phase

- You will be able to explain step by step what ARP does, and why the request is a **broadcast** while the
  reply is a **unicast**
- You will be able to read the ARP cache (`ip neigh`) and know what `REACHABLE`/`STALE`/`FAILED` mean
- You will be able to explain **how a switch learns** its MAC address table (source learning) and what it
  does with a destination it does not know (flooding)
- You will be able to state the difference between a **broadcast domain** and a **collision domain**, and
  explain the difference between a hub and a switch with those two concepts
- You will be able to say why a router does not forward broadcasts, and connect that to subnet design
- You will be able to explain what a VLAN is, why it splits one physical switch into logical networks, and
  what a trunk port is
- You will recognise the signatures of ARP failures (IP conflict, stale/wrong ARP entry, ARP spoofing)
- **Cloud:** You will be able to explain why L2 is invisible in the cloud, why broadcast is not supported in
  a VPC, and what the "subnet + security group instead of VLAN" model means

---

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 3.1 | ARP — from IP to MAC | `[mechanism]` | **The heart of the phase** — the gap Phase 2 left |
| 3.2 | The switch and the MAC address table | `[mechanism]` | Getting the frame to the right port |
| 3.3 | Broadcast domain vs collision domain | `[concept]` | The physical reason for splitting subnets |
| 3.4 | VLAN | `[concept]` | One box of hardware, several logical networks |
| 3.5 | When this phase breaks | — | The signatures of L2 failures |

> **How to work through this phase:** Most of the commands are 🟢 (`ip neigh`, `ip link`, `bridge fdb`) but
> one is 🟡: clearing the ARP cache (`ip neigh flush`). That is temporary — the table refills by itself —
> but still, do it on your own test machine, not on a production one. The most instructive experiment:
> look at the table with `ip neigh`, ping a neighbour, look again — watch the new line appear **with your
> own eyes**. This is the phase where theory is easiest to verify in a terminal; do not settle for reading.

---
---

# 3.1 ARP — The Bridge from IP to MAC

## 3.1.1 The problem: I have an IP, I need a MAC `[concept]`

Let us state the situation cleanly. Your machine is `10.0.1.50`, the destination is `10.0.1.80`. By the
arithmetic of Phase 2 they are on the same network (`/24`), so this will be a direct delivery.

To build the frame you need:

```
[ Dest MAC: ??? | Source MAC: mine | Dest IP: 10.0.1.80 | Source IP: 10.0.1.50 | data ]
   └─ L2 header ──────────────────┘ └─ L3 header ─────────────────────────────┘
```

You can fill in the L3 header. The **destination MAC** in the L2 header is empty — and if that field stays
empty, the frame cannot be sent.

The protocol that fills that gap is called **ARP**: Address Resolution Protocol. It does exactly one job:
**"Who has this IP? Tell me your MAC."**

## 3.1.2 How ARP works: request broadcast, reply unicast `[mechanism]`

Four steps:

**1. ARP Request (broadcast).** Your machine builds a frame and writes **`ff:ff:ff:ff:ff:ff`** into the
destination MAC field — meaning "to everybody on this network". The contents are:

> *"Who has 10.0.1.80? I am 10.0.1.50, my MAC is `aa:bb:cc:11:22:33`. Send the answer to me."*

Being a broadcast is **mandatory**: your machine does not know the destination's MAC, so it does not know
whom to send the question to. Its only option is to ask everybody.

**2. Everybody hears it, one answers.** Every device on the network receives the frame and looks at the IP
inside it. The ones that say "that is not me" **discard** it. The device that owns `10.0.1.80` prepares a
reply.

**3. ARP Reply (unicast).** The target device sends the answer **directly** to the asker — no broadcast is
needed, because the question already carried the asker's IP *and* MAC:

> *"10.0.1.80 is mine. My MAC is `dd:ee:ff:44:55:66`."*

**4. It is cached.** Your machine writes this pairing into its **ARP table** (ARP cache) and keeps it for a
while. On the next packet it does not ask again — it reads straight from the table. When the timer expires
the entry goes stale and is refreshed.

And there is a critical side effect: **everybody who hears the ARP Request has learned the asker's IP↔MAC
pairing.** Because the request carries the asker's own details too. This is why ARP traffic on a network
makes the tables fill themselves in.

![Figure 3.1 — ARP resolution: machine A broadcasts an ARP Request onto the network to learn B's MAC address (every device receives it, only the target answers), B announces its MAC in a unicast ARP Reply, and A writes the pairing into its ARP cache.](../diagrams/png/nw-3-01-arp-resolution.png)

In the figure, A on the left is the asker and B on the right is the target; the switch in the middle and the
other two devices represent the rest of the network. The dashed arrows show that the broadcast reaches
**everybody**, while the solid arrow shows that the reply goes to a **single** destination. Notice that the
devices that do not answer silently discard the packet — this is what the cost of a broadcast looks like:
everybody on the network has to process every ARP request.

> **🔧 See it on your machine** 🟢 — read the ARP table and watch it fill
>
> ```
> $ ip neigh
> 172.31.16.1 dev ens5 lladdr 06:8f:3c:2a:11:04 REACHABLE
> 172.31.20.9 dev ens5 lladdr 06:12:ab:55:9e:7d STALE
>
> $ ping -c1 172.31.20.30 > /dev/null; ip neigh | grep 172.31.20.30
> 172.31.20.30 dev ens5 lladdr 06:aa:7e:31:c0:15 REACHABLE
> ```
>
> Each line is an **IP ↔ MAC** pairing. Learn the states: **REACHABLE** = verified recently, trustworthy;
> **STALE** = the entry exists but has aged, it will be verified on next use; **DELAY/PROBE** = verification
> in progress; **FAILED** = no answer came back to the ARP request (the target does not exist, is down, or
> is on a different network — 3.5). The second command is the experiment itself: **before** the ping that
> line was not there, afterwards it appeared. You have just watched ARP work.

> **⚠️ Common misconception: "ARP is used to find the MAC of a remote server."**
>
> No — ARP works **only on the local network**. A broadcast does not cross a router (3.3.2), so an ARP
> request **never reaches** a remote device. For a remote destination your machine does not ARP for the
> destination's IP, it ARPs for the **default gateway's** IP (the answer to Think 1.1 in Phase 1). So when
> you send a packet to `93.184.216.34`, that address is **not** in your ARP table; what is there is
> `172.31.16.1` (the gateway). This is something you will notice immediately when reading `ip neigh`
> output: the table contains only your **neighbours**.

> **❓ A question that comes to mind: "What if everybody answered the broadcast?"**
>
> It would be chaos — and in fact that is exactly what an **ARP spoofing** attack does. ARP has **no**
> verification mechanism whatsoever: when a device says "10.0.1.1 is mine", nobody checks. A malicious
> device can broadcast fake ARP replies for the gateway's IP and pull the whole network's traffic through
> itself (man in the middle). The defences live at the switch level (dynamic ARP inspection, port security)
> and are outside the scope of this book. But it matters for understanding why L2 is fully abstracted away
> in the cloud: in AWS an instance **cannot** answer an ARP request on behalf of another instance's IP —
> the hypervisor does not allow it. The cloud hiding L2 is not a convenience, it is a **security decision**.

> **🤔 Think 3.1** — Your machine is `10.0.1.50/24`. You run two commands in order: `ping 10.0.1.80` and
> `ping 8.8.8.8`. (a) Is ARP performed in both cases? (b) If so, **for which IP**? (c) Afterwards, which
> new lines do you expect to see in the `ip neigh` output?
>
> *(Answer: at the end of the phase)*

---
---

# 3.2 The Switch and the MAC Address Table

## 3.2.1 What a switch does `[concept]`

The frame is ready: the destination MAC is filled in and it has left on the wire. Now it has arrived at a
**switch**. The switch has exactly one job: send this frame to the **right port**.

It does that by keeping a table: the **MAC address table** (also called the CAM table or forwarding table).
The table is simple:

| MAC address | Port |
|---|---|
| `aa:bb:cc:11:22:33` | 1 |
| `dd:ee:ff:44:55:66` | 3 |
| `11:22:33:aa:bb:cc` | 7 |

When a frame arrives the switch looks at the destination MAC, searches the table, and sends it out the port
it found. It **does not send it to the other ports** — this is the fundamental thing that separates a switch
from a hub (3.3.1).

## 3.2.2 How it learns the table: source learning `[mechanism]`

The switch is not given this table by anybody, it **learns it itself**. The mechanism is elegant:

**For every incoming frame it looks at the SOURCE MAC and notes "this MAC came in on this port".**

So the switch learns **sources**, not destinations. If a frame with source MAC `aa:bb:...` came in on port
1, then the switch now knows that frames destined for `aa:bb:...` should go out of port 1.

And what if the destination MAC is **not** in the table? The switch does not guess — it **sends it out of
every port** (except the one it came in on). This is called **flooding**. When the target device answers,
the switch learns its source MAC too and will not flood again.

This is why the first moments of a network are "noisy": the tables are empty and everything is flooded.
Within a few seconds the tables fill and the traffic becomes targeted.

The entries are **not permanent**: every entry has an aging time (typically 5 minutes). If a device stays
quiet for long enough its entry is deleted. This is what lets the table fix itself when a device is moved.

> **🔧 See it on your machine** 🟢 — the MAC table of a Linux bridge
>
> ```
> $ bridge fdb show
> 33:33:00:00:00:01 dev ens5 self permanent
> 06:8f:3c:2a:11:04 dev br0 master br0
> 06:12:ab:55:9e:7d dev veth2 master br0
> ```
>
> If you create a bridge on Linux, the kernel behaves exactly like a switch and keeps the **same** MAC
> table — `bridge fdb show` (forwarding database) shows it. The `dev` column is the interface that MAC was
> learned on; it is the counterpart of the "port" column on a physical switch. Entries marked `permanent`
> were added by hand, the rest were **learned**. If you use Docker or a virtual machine you will see several
> entries in this table — the virtual interfaces of containers are attached to a bridge.

> **🤔 Think 3.2** — A switch has A on port 1 and B on port 3, and its table is completely empty. A sends a
> frame to B. (a) What does the switch learn from that frame? (b) Which port or ports does it send the
> frame to? (c) When B answers, how does the table change, and will subsequent frames still be flooded?
>
> *(Answer: at the end of the phase)*

---
---

# 3.3 Broadcast Domain and Collision Domain

## 3.3.1 Two different "domains" `[concept]`

These two terms are often confused, but they describe **completely different** things.

**Collision domain:** the physical region where, if two devices transmit at the same time, the **signals
will collide**. On old **hubs** every port was in one single collision domain — while one device was
talking, the others had to wait. The **switch** solved this: every port is a **separate collision domain**.
On modern switched networks collisions are practically non-existent.

**Broadcast domain:** the set of all devices a broadcast frame (`ff:ff:ff:ff:ff:ff`) can reach. **A switch
does not stop a broadcast** — it spreads it to all of its ports. So everything attached to a switch is in
**one single broadcast domain**.

The summary is worth memorising:

| Device | Collision domain | Broadcast domain |
|---|---|---|
| **Hub** | All one (bad) | All one |
| **Switch** | Each port separate (good) | All one |
| **Router** | Each port separate | **Each port separate** |

The last row contains the single most important sentence of this phase: **the only device that splits a
broadcast domain is the router.**

## 3.3.2 Why a router does not forward a broadcast `[mechanism]`

A router is an L3 device: it opens the packet, looks at the **IP header** and routes it. The L2 destination
address of a broadcast frame is `ff:ff:ff:ff:ff:ff`, and that address means "to everybody on this segment" —
forwarding it onto another segment would be **meaningless**.

So the router stops broadcasts **deliberately**. And that has two big consequences:

**1. ARP cannot cross a router.** This is why you can never learn a remote destination's MAC (the
misconception box in 3.1.2) — and it is why the concept of a gateway exists (Phase 4.1).

**2. Broadcast noise is bounded.** The more devices there are on a network, the more ARP, DHCP and other
broadcast traffic there is. And every broadcast occupies the CPU of **every** device on the network —
because every device has to receive the frame and check "is this me?". A single broadcast domain with
thousands of devices can render a network unusable through a "broadcast storm".

This is the physical counterpart of the "shrink the broadcast domain" justification we gave in Phase 2.4.1:
**every subnet is a separate broadcast domain.** Splitting the network is splitting the noise.

> **💡 Cloud connection — there is no L2 in the cloud (as far as you are concerned):** Broadcast and
> multicast are **not supported** in an AWS VPC. You cannot send a broadcast from an instance; even the
> gateway MAC you see in your ARP table is synthesised by the hypervisor. The reason is scale: while tens of
> thousands of customers' instances share the same physical hardware, a real broadcast domain would be a
> disaster for both security and performance. Instead, AWS **solves everything at L3**: traffic between
> instances flows over a software-defined routing layer, not as if there were a physical switch in between.
> The practical consequences: (1) old protocols that rely on L2 (some cluster heartbeats, L2 discovery
> protocols) **do not work** in the cloud; (2) you cannot see another instance's traffic with `tcpdump` —
> promiscuous mode is meaningless; (3) ARP spoofing is impossible. L2 did not disappear, it was **hidden
> from you** — and its place was taken by the trio of subnet + route table + security group.

> **🤔 Think 3.3** — An office has 500 devices on a single `/23` network (510 addresses), attached to a
> single stack of switches. (a) How many broadcast domains are there? (b) When one device sends an ARP
> request, how many devices have to process that frame? (c) If you split the network into four `/25`
> pieces, what changes — and which device do you need in order to do it?
>
> *(Answer: at the end of the phase)*

---
---

# 3.4 VLAN — One Box of Hardware, Several Networks

## 3.4.1 The idea: logical separation `[concept]`

In 3.3 we said "the only device that splits a broadcast domain is the router". Physically that would mean:
buying a **separate switch** for every network and putting a router between them. Expensive and inflexible.

**VLAN** (Virtual LAN) solves this problem: it forces a single physical switch to behave like **several
logical switches**.

How? Every port of the switch is assigned a **VLAN number**:

| Port | VLAN | Use |
|---|---|---|
| 1–8 | 10 | Accounting |
| 9–16 | 20 | Engineering |
| 17–24 | 30 | Guest Wi-Fi |

The rule is strict: **ports in different VLANs never see each other.** A broadcast from a device in VLAN 10
**does not go** to the ports in VLAN 20. The switch behaves as if it were three separate switches.

And the result: **every VLAN is a separate broadcast domain**, which means it corresponds to a separate
subnet. VLAN 10 = `10.0.10.0/24`, VLAN 20 = `10.0.20.0/24`, and so on. Traffic between VLANs still has to
pass through a **router** (or an L3 switch) — exactly by the rule in 3.3.2.

## 3.4.2 Trunk ports and the 802.1Q tag `[concept]`

There is a problem: when connecting two switches to each other, do we have to run a separate cable for every
VLAN?

No. A **trunk port** is used: the traffic of **several VLANs** is carried over a single cable. What makes
this possible is a small **tag** added to the frames (the 802.1Q tag, 4 bytes): it says which VLAN the frame
belongs to.

- **Access port:** belongs to a single VLAN, there is **no** tag (the devices know nothing about VLANs).
- **Trunk port:** carries several VLANs, the frames are **tagged**.

This detail is at the `[skip]` level — unless you administer an enterprise network there is no need to go
deep. But knowing the concept is necessary in order to see its counterpart in the cloud: cloud providers use
a far larger-scale version of the same idea (VXLAN) to run thousands of isolated virtual networks on top of
one physical network (Phase 7.4).

> **💡 Cloud connection — the cloud equivalent of a VLAN: subnet + security group:** In the cloud you do not
> configure VLANs — because L2 is not yours (3.3.2). But the problem a VLAN solves is still there:
> **isolating different workloads from each other.** The cloud solves it with two tools: (1) **Subnet** — it
> splits the network at the address level, and the route table determines where each subnet may go (a public
> subnet reaches the internet, a private one cannot); (2) **Security group** — at instance level, it
> determines which source may reach which port (Phase 9.4). In a VLAN, isolation was tied to the **port**;
> in the cloud it is tied to **identity** — you can name one security group as the source in another one
> ("open 5432 to traffic coming from the web SG"), which is more flexible than anything a VLAN can do. The
> mental mapping: *VLAN ≈ subnet + SG*, but the cloud model is more fine-grained.

> **🤔 Think 3.4** — A switch has VLAN 10 (`10.0.10.0/24`) and VLAN 20 (`10.0.20.0/24`). A machine in
> VLAN 10 pings a machine in VLAN 20. (a) Does the ARP request reach the target? (b) Will the ping work,
> and what is needed for it to work? (c) Explain why we count them as being "on different networks" even
> though they are attached to the same physical switch.
>
> *(Answer: at the end of the phase)*

---
---

# 3.5 When This Phase Breaks — The Signatures of L2 Failures

L2 failures tend to be **local and sharp**: either you reach your neighbour or you do not, there is nothing
in between. And the fastest diagnostic tool is `ip neigh` — the ARP table is the health report of L2.

| Symptom | Likely cause | Verify | Related section |
|---|---|---|---|
| Target shows `FAILED` in `ip neigh` | Target is down, absent, or on a different network | `ip neigh`, the target's prefix | 3.1.2 |
| A neighbour on the same network is never reachable | No ARP reply / no L2 link | `ip link` (LOWER_UP), `ip neigh` | 3.1.2 |
| Connection is intermittent, two devices keep dropping | **IP conflict** — two devices with the same IP | `ip neigh` (does the MAC change?), arp-scan | 3.1.2 |
| Right IP, traffic goes to the wrong machine | Old/stale ARP entry | `ip neigh` → `ip neigh flush` 🟡 | 3.1.2 |
| Everything is slow, switch lights blink constantly | Broadcast storm / loop | Broadcast domain size, STP | 3.3.2 |
| One device sees the network, another does not | The ports are in different VLANs | Switch port VLAN assignments | 3.4.1 |
| Same switch but ping does not work | Different VLAN or different subnet | VLAN + `ip addr` prefixes | 3.4.1 (2.1.2) |
| Traffic is being redirected to another device | ARP spoofing (gateway impersonation) | Is the gateway MAC the expected one | 3.1.2 |
| A broadcast-based application fails in the cloud | A VPC does not support broadcast/multicast | The application's L2 requirement | 3.3.2 |
| MAC table is full, everything is flooded | MAC flooding attack / too many devices | Switch MAC table occupancy | 3.2.2 |

> **The lesson from this table:** For L2 failures the first place you look is **`ip neigh`**, and when you
> look, ask two questions: *is there an entry* and *is the entry correct*. **If there is no entry, or it is
> `FAILED`**, the target is either down or in fact not on your network — check the second possibility
> immediately with the prefix (Phase 2), because this is the most common symptom of a wrong mask: the
> machine thinks a remote host is a neighbour and ARPs for it, nobody answers, `FAILED`. **If there is an
> entry but the MAC keeps changing**, your suspicion should go to two things: an IP conflict (two devices
> defending the same address) or ARP spoofing (someone impersonating another's address). One last reflex:
> before you investigate L2, make sure **the link exists** — if `ip link` does not show `LOWER_UP` there is
> a problem at the cable/port level and discussing ARP is pointless. Always bottom-up:
> **link → ARP → IP → routing.**

---
---

# Phase 3 — Answers to the Think questions

## Answer 3.1 — ARP is always performed for the next hop

(a) **Yes, ARP happens in both cases** — because in both cases a frame is going to be built, and that
frame's destination MAC field has to be filled in.

(b) But **for different IPs**:
- `ping 10.0.1.80`: the destination is on the same network (with `/24` the network is `10.0.1.0`, and both
  are inside it). ARP is performed for **`10.0.1.80`** — that is, for the destination itself.
- `ping 8.8.8.8`: the destination is on a different network. Your machine will send the packet to the
  **default gateway** (Phase 4.1), so ARP is performed for the **gateway's IP** (for example `10.0.1.1`).
  **No ARP at all is performed for `8.8.8.8`** — a broadcast cannot cross a router (3.3.2).

(c) In the `ip neigh` output:
- After the first ping: `10.0.1.80 dev ... lladdr <B's MAC> REACHABLE`
- After the second ping: `10.0.1.1 dev ... lladdr <the gateway's MAC> REACHABLE`
- **No line at all for `8.8.8.8`** — and its absence is the correct behaviour.

This is the explanation for why your ARP table contains only your neighbours: **ARP is always performed for
the next hop, never for the final destination.**
**Related section:** 3.1.2 · **Next:** Phase 4.1 (default gateway), Phase 4.2 (the routing table).

## Answer 3.2 — A switch learns sources and floods destinations

(a) The switch looks at the frame's **source MAC** and writes "A's MAC is on port 1" into the table (3.2.2).
It learns nothing about the destination MAC — learning always happens from the **source**.

(b) The destination MAC (B's MAC) is **not** in the table. The switch does not guess: it sends the frame
out of **every port except the one it arrived on** — **flooding** (3.2.2). So it does reach port 3 (and all
the other ports, if there are any).

(c) When B answers, the reply frame's source MAC is B's MAC and it arrives on port 3. The switch learns
this: "B's MAC is on port 3". Now there are **two entries** in the table. Subsequent A→B frames are **not
flooded**, they are sent straight to port 3.

The beauty of this is that nobody configured the switch and no protocol ran — it learned the topology just
by watching the traffic that passed through it. And when the aging time expires the entries are deleted and
relearned, so the table corrects itself when a device is moved.
**Related section:** 3.2.1-3.2.2 · **Next:** 3.3.1 (collision domain), Phase 10.2 (L2 diagnosis).

## Answer 3.3 — One switch = one broadcast domain

(a) **One.** A switch does not stop broadcasts (3.3.1); however many switches are connected to each other,
if there is no router (or VLAN) in between they are all **one single broadcast domain**.

(b) **All 500 devices.** Every device receives the frame, looks at the IP inside, says "that is not me" and
discards it. So one single ARP request creates pointless work on the CPUs of 499 devices. If you consider
that a 500-device network can see dozens of ARP requests per second, that is a serious load — and it gets
**quadratically** worse as the network grows.

(c) Four `/25` pieces = **four separate broadcast domains**, roughly 126 devices in each. An ARP request now
occupies ~126 devices instead of 500 — the noise drops to a quarter. But for that you need a **router** (or
an L3 switch), because that is the only device that splits a broadcast domain (3.3.1). And since traffic
between the four subnets now passes through the router, it becomes **inspectable** — which is how the
"security boundary" justification of Phase 2.4.1 is earned as well.
**Related section:** 3.3.1-3.3.2 · **Next:** Phase 2.4.1 (why we split), 3.4.1 (splitting with VLANs).

## Answer 3.4 — Same switch, different VLAN = different network

(a) **No.** An ARP request is a broadcast, and a broadcast from VLAN 10 **does not go** to the ports in
VLAN 20 (3.4.1). The switch keeps the two VLANs as if they were two separate switches.

(b) The ping **can** work — but only if there is a **router** (or L3 switch) doing routing between the
VLANs. In that case: the machine sees the destination on a different network (a different subnet), sends
the packet to its own gateway, the router routes the packet into VLAN 20, a new ARP there finds the
destination's MAC and it is delivered. Without a router the ping **fails**, and in `ip neigh` the gateway
shows as `FAILED` or no entry appears at all.

(c) Because "the same network" is **not a physical proximity, it is a logical definition** — and two
separate mechanisms determine it together: (i) at L3 the prefix (Phase 2.1.2) puts them in different
subnets, (ii) at L2 the VLAN puts them in different broadcast domains. Both say "different network". Being
on the same cable, in the same box, changes nothing: a network is something determined by the
**configuration, not the cable**. In the cloud this idea reaches its extreme — physical proximity is
completely meaningless (the cloud box in 3.3.2).
**Related section:** 3.4.1 · **Next:** Phase 4.1 (routing between VLANs), Phase 11.2 (subnet isolation).

---
---

# Phase 3 — Frequently asked questions

**Q1 — How long is an ARP entry kept?** On Linux typically a few minutes, but it is not fixed: as long as an
entry is used it stays `REACHABLE`, if it is not used it drops to `STALE` and is verified on next use. The
timers are tuned with the parameters under `/proc/sys/net/ipv4/neigh/default/`. In practice you do not need
to fiddle with them; if an entry is wrong you clear it with `ip neigh flush` 🟡 (no undo needed — the table
refills by itself).

**Q2 — What is the difference between ARP and ICMP (ping)?** Different layers, different jobs. **ARP** is at
L2 and finds the IP↔MAC pairing — it does not even carry an IP packet. **ICMP** is at L3 and is carried
inside IP packets; it is used for reachability and error reporting (Phase 4.5). The ordering matters: when
you send a ping, **ARP happens first** (the MAC is found), **then** the ICMP packet is sent. If ARP fails
the ping cannot be sent at all.

**Q3 — What is the fundamental difference between a switch and a router?** A switch is an **L2** device: it
looks at MAC addresses, forwards broadcasts and works within one network. A router is an **L3** device: it
looks at IP addresses, stops broadcasts and connects **different networks** to each other. In one sentence:
a switch manages the inside of a network, a router manages between networks (3.3.1).

**Q4 — What is an "L3 switch"?** A device that can behave both as a switch and as a router: its ports work
like a switch's, but it can also route between VLANs at hardware speed. It is common in enterprise networks
because it is both faster and cheaper than a separate router. Conceptually it is nothing new — it is two
functions combined in one box.

**Q5 — Why do I always see the same MAC in my ARP table in the cloud?** Because most of the MACs you see are
**not real**: the hypervisor synthesises a MAC for the VPC router and answers the instance's ARP requests
itself. There is no real L2 segment (3.3.2). This is why `ip neigh` output in the cloud is usually very
short — typically just the gateway (`.1`) and perhaps a couple of neighbouring instances.

**Q6 — How do I confirm an IP conflict for certain?** The signature: in the `ip neigh` output the **MAC for
the same IP keeps changing**, and the connection works intermittently. The reason: both devices answer ARP
for that IP, and whichever reply arrives last updates the table, so traffic bounces between the two devices.
To confirm you can use `arp-scan -l` or `arping <ip>` (if two different MACs answer for the same IP, the
conflict is certain).

**Q7 — Do I need to learn this phase if I work in the cloud?** Yes — for two reasons. (1) **For diagnosis:**
seeing `FAILED` in `ip neigh` tells you instantly that the problem is at the L2/addressing level and narrows
the search. (2) **For the model:** the cloud's subnet, route table and security group model is exactly this
phase's concepts (broadcast domain, isolation, adjacency) repackaged. If you do not know what replaced what,
cloud abstractions look like "magic" — and magic cannot be diagnosed.

---
---

# Phase 3 — Test yourself

Write your answers on paper, then compare them with the answer key. Target: 14+ out of 18.

## Part A — Definition and mechanism

1. What is ARP for? What is its input and its output?
2. Why is an ARP Request a broadcast and an ARP Reply a unicast?
3. What do the `REACHABLE`, `STALE` and `FAILED` states in `ip neigh` output mean?
4. How does a switch learn its MAC address table? Which field does it look at?
5. What does a switch do if the destination MAC is not in the table? What is that behaviour called?
6. What is the difference between a collision domain and a broadcast domain?
7. Which device splits a broadcast domain, and why?
8. What is a VLAN? What is the difference between an access port and a trunk port?

## Part B — Apply and diagnose

9. You ping `10.0.1.99` from the machine `10.0.1.50/24`. Which IP is ARP performed for?
10. From the same machine you ping `1.1.1.1`. Which IP is ARP performed for, and does `1.1.1.1` appear in
    the ARP table?
11. In `ip neigh` output the target's state is `FAILED`. Write two possible causes.
12. A user's connection is intermittent and in `ip neigh` the MAC for that IP keeps changing. What is your
    diagnosis?
13. A switch with an empty table has A on port 1 and B on port 2. When an A→B frame arrives, what does the
    switch learn and where does the frame go?
14. Your application running inside a VPC does broadcast-based discovery and finds no peers at all. Why?

## Part C — Reasoning and connection

15. ARP's inability to cross a router is the reason for the existence of which concept (which you will see
    in Phase 4)? Explain.
16. A network has 1000 devices in one broadcast domain. Write two concrete problems and state the solution
    in one sentence.
17. Two machines are attached to the same physical switch but cannot ping each other. Write three different
    causes (one at L2, one at L3, one VLAN-related).
18. What three things does hiding L2 in the cloud make impossible? One sentence for each.

---

## Answer key

1. To find a **MAC address from an IP address**. Input: the destination's IP. Output: the MAC of the device
   that owns that IP (3.1.1). — 2. The **request is a broadcast**, because the asker does not know the
   target's MAC — not knowing whom to ask, it asks everybody. The **reply is a unicast**, because the
   request contained both the asker's IP and its MAC; the answer can be sent straight to it (3.1.2). —
   3. **REACHABLE** = verified recently, trustworthy; **STALE** = the entry exists but has aged, it will be
   verified on next use; **FAILED** = no answer came back to the ARP request (3.1.2). — 4. It looks at the
   **source MAC** field of every incoming frame and notes "this MAC is on this port" — **source learning**
   (3.2.2). — 5. It sends the frame out of **every port except the one it came in on**; this is called
   **flooding** (3.2.2). — 6. **Collision domain**: the region where signals can physically collide; on a
   switch every port is separate. **Broadcast domain**: all the devices a broadcast frame reaches; on a
   switch they are all one (3.3.1). — 7. The **router** (or an L3 switch / VLAN separation). Because the
   router is an L3 device and the L2 broadcast address (`ff:ff:...`) is not forwarded onto another segment —
   and this is deliberate, to bound the noise (3.3.2). — 8. A configuration that splits one physical switch
   into several logical switches; every VLAN is a separate broadcast domain. An **access port** belongs to a
   single VLAN and its frames are untagged; a **trunk port** carries several VLANs and its frames are tagged
   with 802.1Q (3.4.1-3.4.2).

9. For **`10.0.1.99`** — since the destination is on the same network, ARP is performed for the destination
   itself (3.1.2). — 10. For the **default gateway's IP** (e.g. `10.0.1.1`). `1.1.1.1` **does not appear**
   in the ARP table, because ARP cannot cross a router and is never performed for remote destinations
   (3.1.2). — 11. (i) The target device is down/absent/not connected to the network; (ii) the target is
   actually **on a different network** but because of a wrong mask the machine thinks it is a neighbour
   (3.5, Phase 2.6). — 12. An **IP conflict** — two devices are using the same IP and both answer ARP; the
   table is updated by whichever reply arrived last (3.5, Q6). — 13. The switch learns that A's MAC is on
   port 1 (from the source MAC). Since the destination MAC is not in the table the frame is **flooded** — to
   every port except port 1 (3.2.2). — 14. Because **a VPC does not support broadcast or multicast**; there
   is no real L2 segment in the cloud, everything is solved at L3 (3.3.2).

15. The concept of the **default gateway**. Since ARP cannot cross a router, a remote destination's MAC can
    never be learned; therefore the machine needs an address **on its own network** that it can say "send
    everything remote here" to. That address is the gateway, and it is what the machine ARPs for (3.1.2,
    3.3.2). — 16. (i) Every ARP/DHCP broadcast occupies the CPU of 1000 devices — pointless load; (ii) there
    is no isolation, every device can reach every other device directly — no security boundary can be drawn.
    The solution: **split the network into subnets (or VLANs)** and put router/L3 control in between
    (3.3.2, Phase 2.4.1). — 17. (i) **L2:** a cable/port fault or the interface is down (no `LOWER_UP` in
    `ip link`); (ii) **L3:** they are on different subnets or their masks are wrong — the machine thinks the
    other side is remote and sends to the gateway (Phase 2.6); (iii) **VLAN:** the ports are assigned to
    different VLANs and the switch separates them like two distinct switches (3.4.1). — 18. (i)
    **Broadcast/multicast-based protocols do not work** — a VPC does not carry them; (ii) **neighbouring
    traffic cannot be sniffed in promiscuous mode** — `tcpdump` only sees its own instance's traffic;
    (iii) **ARP spoofing cannot be done** — the hypervisor does not let an instance answer on behalf of
    another's IP (3.3.2).

## Scoring

| Correct | What it means |
|---|---|
| 16-18 | L2 has settled in. You can move on to Phase 4 (routing). |
| 13-15 | Good. Read 3.1.2 and 3.3.1 once more; repeat the `ip neigh` experiment. |
| 9-12 | ARP and switch learning may have got mixed up. Work through 3.1–3.2 verifying in a terminal. |
| 0-8 | Walk the phase again. Target: being able to answer "which IP is ARP performed for" in any scenario. |

Question you missed → section to return to:

| Question | Section |
|---|---|
| 1, 2, 3, 9, 10, 11 | 3.1 ARP |
| 4, 5, 13 | 3.2 The switch and the MAC table |
| 6, 7, 16 | 3.3 Broadcast/collision domain |
| 8, 17 | 3.4 VLAN |
| 12 | 3.5 The failure table |
| 14, 15, 18 | 3.3.2 + the cloud box |

---
---

# Phase 3 — Closing and the Bridge to Phase 4

## What you carry out of this phase

Phase 3 gave you the **local reality** of the network. You learned how ARP builds the bridge between IP and
MAC — why the request is a broadcast and the reply a unicast. You saw that a switch learns its table from
nobody, just by looking at the **source** addresses of the traffic passing through it, and that it floods
when it does not know. You separated the broadcast domain from the collision domain and learned that **the
only device that splits a broadcast domain is the router** — this is the physical counterpart of the
boundary you drew mathematically in Phase 2. And you saw how a VLAN does the same job logically inside a
single box.

The most durable sentence is this: **ARP is always performed for the next hop, never for the final
destination.** That single sentence both lets you read `ip neigh` output and opens the door to Phase 4.

## Where Phase 4 connects to this

All through Phase 3 we kept hitting the same boundary: **a broadcast cannot cross a router, ARP stays local,
the MAC of a remote destination cannot be learned.**

So then how does a packet reach a remote destination at all?

The answer: the machine sends everything remote to one single address on its own network — the **default
gateway**. And that gateway sends it to the next one, and that one to the next... A new L2 delivery at every
step, but always the same L3 destination.

Phase 4 is that whole chain: how a routing table is read, which row the **longest prefix match** rule picks,
how TTL cuts infinite loops, how ICMP reports errors, and how `traceroute` maps the path.

Phase 3 taught delivery to the neighbour; Phase 4 teaches delivery to **the other side of the world**.

> **🤔 Phase output — ask yourself:** Your machine sends a packet to a remote destination and hands it to
> the gateway. The gateway receives it — but the gateway faces the same problem: the destination is not on
> its local network either. What does it do? And more importantly: how does a router **know** "which way to
> send this packet" — does it know every network on the internet by heart, or is there another mechanism?
> (Hint: we mentioned aggregation in Phase 2.5.)
>
> **🧪 Lab 3 idea (mostly 🟢, one 🟡):** (1) Record your current ARP table with `ip neigh` — how many lines,
> which IPs? (2) Ping your gateway and look at the table again; see that the gateway's line is `REACHABLE`.
> (3) Ping a remote address (`ping -c1 8.8.8.8`), then run `ip neigh | grep 8.8.8.8` — verify with your own
> eyes that **nothing comes out**. This is the cleanest proof of 3.1.2. (4) 🟡 Clear the table with
> `sudo ip neigh flush all`, see it empty with `ip neigh`, then send a ping and watch it fill again.
> *(No undo needed — the table is learned data and refills by itself.)* (5) Find the `LOWER_UP` flag in the
> `ip link` output — it tells you the physical link really exists; it is the first place to look in an L2
> diagnosis. These five steps take ARP out of theory and put it in your hands.

---

> **Navigation:** [◀ Checkpoint Quiz 1](Checkpoint_Quiz_1.md) · **Phase 3** · [Phase 4 — Routing (L3) ▶](Phase_4_Routing_L3.md)
