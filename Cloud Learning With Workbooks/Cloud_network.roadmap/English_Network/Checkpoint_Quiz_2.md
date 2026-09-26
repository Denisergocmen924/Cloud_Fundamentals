# Checkpoint Quiz 2 — Phases 3–4: The Local Network and Routing

> **Navigation:** [◀ Phase 4 — Routing (L3)](Phase_4_Routing_L3.md) · **Checkpoint Quiz 2** · [Phase 5 — The Transport Layer ▶](Phase_5_Transport_Layer.md)

---

## What does this quiz measure?

A checkpoint quiz does not measure a single phase, it measures **the bridge between two phases**.

Phase 3 gave you **local delivery**: ARP, the MAC table, the broadcast domain, VLANs — that is, "how do I
reach my neighbour". Phase 4 gave you **remote delivery**: the default gateway, the routing table, longest
prefix match, TTL, traceroute — that is, "how do I reach something that is not my neighbour".

In a real packet these two are **interleaved.** When a packet goes to a remote destination an L3 decision is
made ("I will send it to the gateway"), but to carry out that decision **L2 is needed again** ("what is the
gateway's MAC?"). So at the end of every routing decision there is an ARP. Connecting these two phases is
what it means to grasp the most fundamental mechanics of the network — and a great deal of confusion starts
exactly at this joint: *"If the packet is going far away, why am I still doing ARP?"*

**How to work through it:**

- Solve it with pen and paper; do not move to the next question without writing the answer.
- The answer key states **which phases' intersection** each question stands at — a question you miss does
  not point at a phase, it points at the **bridge** between two phases.
- There are 21 questions: Part A (connection reasoning, 1–9), Part B (a scenario: the "a subnet is
  unreachable" incident, 10–15), Part C (reading commands and output, 16–21).
- Target time: ~1 hour. What matters is not the time but being able to justify in one sentence *why* each
  answer is what it is.

> **🤔 Before you start:** Complete this sentence in your own words: *"I am sending my packet to a server in
> Japan. So why is the first thing I do looking for the MAC address of the router **in the next room**?"*
> That one question contains both phases. Write your answer down; we will see it in Question 1.

---

# Part A — Connection reasoning (1–9)

**1.** The destination is `8.8.8.8` (a very remote address). Before sending the packet, **whose** MAC
address does your machine look for, and why does it not look for the destination's MAC? Combine Phase 3's
ARP mechanism with Phase 4's gateway concept in one sentence.

**2.** In Phase 3 we saw that ARP works with **broadcast**. In Phase 4 we said that routers do not forward
broadcasts. Why do these two together give the conclusion "every router is a broadcast domain boundary" —
and why is that a matter of **design** when you grow a network?

**3.** A packet reaches its destination by passing through three routers. Along the path, **how many times
is ARP performed** (roughly) and **how many times does the IP header change**? Which principle of Phase 0.1
does your answer confirm?

**4.** In Phase 4.3 we saw longest prefix match (LPM). A route table has the rows `0.0.0.0/0`, `10.0.0.0/8`
and `10.0.5.0/24`. The destination is `10.0.5.7`. Which one is chosen, and **why** does specificity win
rather than order?

**5.** In Phase 3.2 we saw that a switch fills its MAC table by **learning**, and in Phase 4.2 that a
router's route table is filled **by configuration or by a protocol**. How does that difference change the
two devices' failure behaviour — when it meets a destination it does not know, what does a switch do and
what does a router do?

**6.** In Phase 4.4 we saw TTL. When a routing loop forms, why do packets not circulate **forever**, and at
which layer does that protection work? Does the same protection exist at L2 (between switches) — and if
not, what happens?

**7.** Establish the relationship between a VLAN (Phase 3.4) and a subnet (Phase 2): is a VLAN a subnet? Why
does traffic between two VLANs require a **router**, and which Phase 4 concept is that the same thing as?

**8.** In Phase 4.5 we saw that traceroute works by increasing TTL step by step. In a traceroute output a
hop in the middle shows `* * *` but the later hops look normal. Does this mean the packet was **dropped**
there — justify your answer with the nature of ICMP (Phase 4.4).

**9.** **When** are the ARP table of Phase 3.1 and the route table of Phase 4.2 used together? Write the
**order** in which the kernel consults these two tables during a `ping 10.0.5.7`.

---

# Part B — Scenario: the "a subnet is unreachable" incident (10–15)

> A company network has two subnets: `10.10.1.0/24` (the office) and `10.10.2.0/24` (the servers). There is
> a router between them: `10.10.1.1` on the office side, `10.10.2.1` on the server side. The office machine
> `10.10.1.50` cannot reach `10.10.2.20` on the server network. The six questions below are the diagnosis
> you carry out on that machine, in order.

**10.** The first observation: `ping 10.10.1.1` (its own gateway) **works**, `ping 10.10.2.20` **times
out**. Which layers do these two together definitively rule out, and which possibilities remain?

**11.** In the `ip route` output the row `default via 10.10.1.1` **is present**. In the `ip neigh` output
you see `10.10.1.1 lladdr 00:1a:2b:3c:4d:5e REACHABLE`. What do these two lines together prove — and how do
they show that the problem is **not on this machine**?

**12.** You logged into the router. In the `ip route` output there **is** a row for `10.10.2.0/24` and the
interface is correct. But the ping to the machine on the server network still does not get through. Think
about which direction the problem may be in: outbound or return? Which single observation do you look for in
order to tell them apart?

**13.** You logged into the server machine (`10.10.2.20`) and in the `ip route` output there **is no
`default` row**; there is only `10.10.2.0/24 dev eth0`. (a) Does the incoming ping reach the server? (b) Can
the reply get back to the office? (c) Why does the symptom look like "nothing works at all"?

**14.** The administrator says "but I can ping between two machines on the server network, the network
works". Why does that observation not refute the problem — which kind of delivery has it actually tested
(Phase 3 or Phase 4)?

**15.** The root cause has been found: no default gateway was defined on the server machine. Three
solutions are proposed: (a) adding `default via 10.10.2.1` to the server, (b) adding only the specific route
`10.10.1.0/24 via 10.10.2.1` to the server, (c) adding something to the router. Evaluate each one: which
works, which is narrower in scope, which is of no use at all?

---

# Part C — Reading commands and output (16–21)

**16.** `ip neigh` output:

```
10.10.1.1   dev enp3s0 lladdr 00:1a:2b:3c:4d:5e REACHABLE
10.10.1.30  dev enp3s0 lladdr 3c:52:82:1a:4f:07 STALE
10.10.1.77  dev enp3s0  FAILED
```

Interpret all three lines. What do `REACHABLE`, `STALE` and `FAILED` tell you? Which failure does the third
line give away, and does it prove that that machine is down?

**17.** `ip route` output:

```
default via 192.168.1.1 dev enp3s0 proto dhcp metric 100
10.8.0.0/24 dev tun0 proto kernel scope link src 10.8.0.6
192.168.1.0/24 dev enp3s0 proto kernel scope link src 192.168.1.42
```

(a) Which row does a packet destined for `10.8.0.15` match, and which interface does it leave from? (b) A
packet destined for `8.8.8.8`? (c) What does `scope link` mean — why is there no `via` in those rows?

**18.** `ip route get` outputs:

```
$ ip route get 10.8.0.15
10.8.0.15 dev tun0 src 10.8.0.6 uid 1000

$ ip route get 172.20.5.5
172.20.5.5 via 192.168.1.1 dev enp3s0 src 192.168.1.42 uid 1000
```

Why is this command a more definitive diagnostic tool than `ip route`? What does the `via` difference
between the two outputs tell you?

**19.** `traceroute` output:

```
 1  192.168.1.1        1.2 ms   0.9 ms   1.1 ms
 2  10.20.0.1         12.4 ms  11.8 ms  12.0 ms
 3  * * *
 4  * * *
 5  * * *
```

How does this output differ from the situation in question 8? What does it mean that **no** answer comes
after hop 3, and does that by itself mean "the packet gets that far but no further"?

**20.** A summary from two machines' `ip addr` and `ip route` outputs:

```
Machine A: 10.10.1.50/24   default via 10.10.1.1
Machine B: 10.10.1.51/25   default via 10.10.1.1
```

B's mask is `/25`. (a) Can A reach B directly? (b) Can B reach A directly? (c) Why does this asymmetry
arise, and what would its symptom be?

**21.** An ARP exchange captured with `tcpdump`:

```
12:04:11 ARP, Request who-has 10.10.1.1 tell 10.10.1.50, length 28
12:04:11 ARP, Reply 10.10.1.1 is-at 00:1a:2b:3c:4d:5e, length 46
12:04:11 IP 10.10.1.50 > 8.8.8.8: ICMP echo request, id 1, seq 1, length 64
```

Explain the three lines in order. In the third line the destination IP is `8.8.8.8` — so what is the
destination MAC of that packet's **frame**? Which concepts from two phases does this show at the same time?

---

## Answer key

**1.** It looks for **the default gateway's (the router's) MAC address.** Because the L3 decision says "the
destination is not on my network, I will send it to the gateway" (4.1.1); but to put that packet on the wire
an L2 frame is needed, and the frame's destination MAC is **the next physical neighbour** (3.1.1). The
destination's MAC is not looked for because ARP only works inside **the same broadcast domain** — `8.8.8.8`
cannot answer there. In one sentence: *L3 decides where to go, L2 decides whom to hand it to.* · *Phase 3.1
× Phase 4.1*

**2.** ARP is a **broadcast** (3.1.1) and routers **do not forward** broadcasts (3.3.2, 4.1). Therefore
every interface of a router is a separate broadcast domain. The reason it is a design matter: the bigger the
broadcast domain, the more every ARP request reaches **all** machines, creating needless traffic and
processing load (with the risk of a broadcast storm). This is why large networks are split into VLANs (3.4)
and subnets (Phase 2) — splitting is not only about address management, it is about **bounding the
broadcast**. · *Phase 3.1/3.3 × Phase 4.1*

**3.** **ARP four times** (once per segment: source→R1, R1→R2, R2→R3, R3→destination — each one finds its
own neighbour's MAC) and **the IP header never changes** (the TTL field is decremented and the checksum
recomputed, but the source/destination IPs stay fixed, 4.4.1). This confirms Phase 0.1's principle of
**layer independence**: L2 information is regenerated at every hop, L3 information is carried end to end. ·
*Phase 0.1 × Phase 3.1 × Phase 4.4*

**4.** **`10.0.5.0/24` is chosen.** The LPM rule picks **the longest prefix** (the most specific) among the
matching rows (4.3.1). Specificity wins rather than order because a longer prefix means more **precise**
information about the destination: `/24` says "this is exactly this 256-address network" while `/8` says
"this is somewhere in this block of 16 million". Routing uses the most precise information it has — which is
why `0.0.0.0/0` (prefix length 0) is always the **last resort**. · *Phase 4.3 × Phase 2.2*

**5.** When a switch meets a destination MAC it does not know, it **floods** — it sends the frame to every
port except the ingress port and learns from the reply (3.2.2). That is, it solves its ignorance "by asking
everybody". A router, on meeting a destination it does not know, **drops** the packet and returns
`Destination Unreachable` (4.4.2) — it does not guess and it does not flood. The difference is critical in
diagnosis: L2 failures usually look like "it works but it is slow/noisy", while L3 failures look like "it
does not work at all". · *Phase 3.2 × Phase 4.2*

**6.** Thanks to **TTL** protection (4.4.1): every router decrements TTL by one as it forwards the packet,
and when it hits zero it drops the packet and sends `Time Exceeded`. The protection works **at L3** (in the
IP header). **No such field exists at L2** — there is no TTL in an Ethernet frame. This is why, if a loop
forms between switches, frames circulate forever and lock up the network (a **broadcast storm**, 3.3.2); a
separate protocol (STP) is used to prevent it. · *Phase 3.3 × Phase 4.4*

**7.** In practice **a VLAN corresponds to a subnet** — a VLAN splits the broadcast domain at L2, a subnet
splits the address space at L3, and the two are designed together (3.4.1). Traffic between two VLANs
requires a router because they are different broadcast domains: ARP cannot cross, so direct delivery is
impossible. This is what Phase 4 calls **inter-VLAN routing**, and mechanically it is **exactly the same** as
routing between two subnets. · *Phase 3.4 × Phase 4.1/4.2*

**8.** **No, it does not mean it was dropped.** A `* * *` says that that hop **does not generate ICMP Time
Exceeded** — either it has ICMP disabled by configuration, or it treats generating ICMP as low priority and
skips it (4.5.2). But it has carried on **forwarding** the packet; the proof of that is that the later hops
answer. In traceroute, only **no answer at all up to the final hop** is a sign of a real problem. · *Phase
4.4 × Phase 4.5*

**9.** The order: **the route table first, then the ARP table.** For `ping 10.0.5.7` the kernel (i) looks at
the route table and decides whether the destination is reached directly or via a gateway (4.2.1); (ii) looks
for the MAC of the resulting **next hop address** in the ARP table, and if it is not there broadcasts an ARP
request (3.1.2). So the L3 decision determines the L2 query — it cannot work in the reverse order, because
only after the route decision do you know whose MAC to look for. · *Phase 3.1 × Phase 4.2*

**10.** `ping 10.10.1.1` working rules out **L1, L2 and local L3**: the cable, the interface, the IP
configuration, ARP and the local switch are sound — you can reach the gateway (3.1, 4.1). A timeout means a
"silent drop" (you will see this in Phase 9). The remaining possibilities: the router has no `10.10.2.0/24`
route, there is no **return route**, the destination machine is not up, or there is filtering in between. ·
*Phase 3.1 × Phase 4.1/4.2*

**11.** Together they prove this: the machine **knows its gateway** (there is a default in the route table,
4.1.1) **and can reach it at the L2 level** (ARP is resolved, the state is `REACHABLE`, 3.1.2). So
everything this machine has to do is done: the packet leaves for the right place, with the right MAC. The
problem is not on this machine but **further along** — at the router, at the destination, or on the return
path. This is the evidence you need to move the diagnosis to the next hop. · *Phase 3.1 × Phase 4.1/4.2*

**12.** The problem is most likely **in the return direction** (4.1.1). The single observation to look for
in order to tell: **is the packet reaching the destination machine?** You run `tcpdump -ni any icmp` on the
destination and see whether the echo request appears. If the request appears but no reply comes back to the
office, **the outbound path is sound and the return is broken**; if the request never appears, the problem
is on the outbound side (the router or an obstacle in between). Testing one direction is always half a test.
· *Phase 4.2 × Phase 4.4*

**13.** (a) **Yes, the ping reaches the server** — the router has the `10.10.2.0/24` route and the server is
reachable on its own segment. (b) **No, the reply cannot get back:** the server's route table has only
`10.10.2.0/24`; the destination `10.10.1.50` matches no row, and since there is no `default` either the
packet is dropped (4.4.2 — "no route to host"). (c) The symptom looks like "nothing works at all" because
the user in the office only sees that **no reply came**; they cannot see that their packet reached the
destination. This is exactly why asymmetric failures are so confusing. · *Phase 4.1 × Phase 4.2*

**14.** Because the two servers are **on the same subnet**: that test only verifies **direct delivery**
(Phase 3 — ARP and the switch), it never uses routing at all. A missing default gateway has no effect on
local traffic; the problem only appears when you have to go to **a different network** (4.1.1). So the
observation is correct but it tests something irrelevant. This is a reasoning error seen very often in the
field: *local working does not mean routing is working.* · *Phase 3.1/3.2 × Phase 4.1*

**15.** (a) **It works** — a default route sends every destination the server does not know to the router;
it is the most general and most common solution (4.1.1). (b) **It works and is narrower in scope** — it only
opens the return to the office network; if it is a deliberate security choice that the server can reach no
other network, it is the right pick (under LPM this specific row would be chosen before the default anyway,
4.3.1). (c) **Adding something to the router is of no use** — the router already has routes to both
networks; the missing information is in **the server's** table. Intervening at the right layer is half the
diagnosis. · *Phase 4.1 × Phase 4.2/4.3*

**16.** `REACHABLE` = the ARP entry is verified and fresh, there is communication (3.1.2). `STALE` = the
entry exists but has not been verified for a while; it will be used and re-verified if necessary — it is
**not a fault**. `FAILED` = an ARP request was made and **no answer came**. That shows there is no machine
answering on the segment that IP belongs to (3.1.2). It does **not prove** the machine is down: the machine
may be powered off, may not answer ARP, may be in a different VLAN (3.4), or the address may not be in use at
all. · *Phase 3.1 × Phase 3.4*

**17.** (a) It matches the `10.8.0.0/24` row and leaves from **`tun0`** (a VPN tunnel). (b) `8.8.8.8`
matches only `default` → it leaves from `enp3s0` via `192.168.1.1` (4.3.1). (c) `scope link` says that that
network is **directly attached** — no intermediate router is needed to reach the destination, which is why
there is no `via` field. `via` is present only where the packet will be handed to a next hop (4.2.1). ·
*Phase 4.2 × Phase 4.3*

**18.** Because `ip route` **lists** the table, while `ip route get` shows the decision the kernel will
**actually make** for that destination — it applies LPM for you (4.3.1). In tables with many overlapping
rows, finding by eye which row wins is error-prone; this command ends the argument. The `via` difference: the
first output has **no** `via` → the destination is on a directly attached network (on the tunnel interface).
The second **has** `via 192.168.1.1` → the destination is reached through a **next hop** (4.2.1). · *Phase
4.2 × Phase 4.3*

**19.** In the situation in question 8 there was a gap in the middle but **the later hops answered** —
forwarding was continuing there. Here there is **no** answer at all after hop 3, so the path may really be
cut there. But that is **not conclusive by itself**: a region along the path where ICMP is blocked produces
exactly the same picture (4.5.2, Phase 9.3.1). The way to verify: try to reach the destination with the real
protocol and port (`nc -zv host port`). If access works, the `* * *`s are just ICMP policy; if not, the path
really is broken. · *Phase 4.4 × Phase 4.5*

**20.** (a) **Yes** — A's mask is `/24`, its network is `10.10.1.0–255`; B (`.51`) is in that range, so A
ARPs directly (2.1.2, 3.1). (b) **Yes** — B's mask is `/25`, its network is `10.10.1.0–127`; since A (`.50`)
is in that range, B reaches it directly too. (c) In this particular example **no asymmetry arises**, because
both addresses are in the first block of the `/25`. The asymmetry would arise if one of the addresses were in
the `.128–.255` range: A would count it as "my neighbour" and ARP, while B would say "another network" and
send it to the gateway — the symptom would be "it works in one direction, not in the other", or the router
being pulled in needlessly. The lesson: **a mask mismatch produces insidious failures** and the mask must be
the same on all machines. · *Phase 2.1 × Phase 3.1 × Phase 4.1*

**21.** Line 1: `10.10.1.50` is making an **ARP broadcast** — "who has the MAC of `10.10.1.1`?" (3.1.2).
Line 2: the gateway answers — "it is mine, `00:1a:2b:3c:4d:5e`" (3.1.2). Line 3: now that the MAC is known,
the ICMP packet is sent. The frame's **destination MAC is `00:1a:2b:3c:4d:5e`**, that is, **the gateway's
MAC** — even though the destination IP is `8.8.8.8`. These three lines are the proof of this quiz's main
idea: **the L3 destination is far away, the L2 destination is always a neighbour** (Question 1). · *Phase 3.1
× Phase 4.1 × Phase 0.2*

---

## Scoring

| Correct | Assessment |
|---|---|
| 19–21 | You have merged L2 and L3 into a single flow. Move to Phase 5 with confidence. |
| 15–18 | Solid. Take one pass back over the **bridge** the questions you missed point at. |
| 10–14 | You know the two phases separately but the joint is weak. Use the table below. |
| 0–9 | Redo Phase 3.1 (ARP) and Phases 4.1–4.3 (gateway, route, LPM); without this bridge Phase 5 hangs in the air. |

**Which question you missed → where to go back:**

| Question missed | Go back — this bridge is weak |
|---|---|
| 1, 3, 9, 21 | Phase 3.1 × Phase 4.1 — where ARP and the gateway join |
| 2, 7 | Phase 3.3/3.4 × Phase 4.1 — broadcast domain, VLAN, inter-VLAN routing |
| 4, 17, 18 | Phase 4.2/4.3 — the route table and LPM |
| 5, 16 | Phase 3.1/3.2 × Phase 4.2 — the learning switch vs the configured router |
| 6 | Phase 3.3 × Phase 4.4 — TTL, loops, broadcast storms |
| 8, 19 | Phase 4.4/4.5 — the nature of ICMP and reading traceroute |
| 10, 12, 13, 14, 15 | Phase 4.1/4.2 — the return route and asymmetric failures |
| 20 | Phase 2.1 × Phase 3.1 — the L2 consequence of a mask mismatch |

---

## Closing — from here to Phase 5

Phases 3 and 4 together taught you a packet's **journey**: how it is delivered to a neighbour, how it is
routed far away, and why there is again an ARP at the end of every routing decision. You can now look at an
`ip route` output and say where the packet will leave from, and at an `ip neigh` output and say whether the
L2 side is sound.

But you never asked one thing: **what happens if the packet is lost on the way?**

Every mechanism we have described so far works on a "I will do my best" principle. IP does not guarantee
that the packet arrives. If a router is full it drops it, if a cable is noisy it gets corrupted, if TTL runs
out it is destroyed — and **nobody is obliged to tell the sender.**

Phase 5 connects exactly here: how do you build **reliable** transmission on top of this unreliable ground?
Sequence numbers, acknowledgements, retransmission, flow and congestion control — the whole of TCP exists to
fill the gap Phase 4 left. And there is one more thing you will meet there:
MTU (5.7.1) will be the source of one of the most insidious classes of failure (5.7.3).

> **Before you go on:** If you can answer questions 1 and 21 above without hesitation — why the frame of a
> packet going to a remote destination carries the gateway's MAC — you are ready for Phase 5. If you cannot,
> take one pass back over Phase 3.1 and Phase 4.1; Phase 5 will carry on assuming that the packet **reaches**
> its destination.

---

> **Navigation:** [◀ Phase 4 — Routing (L3)](Phase_4_Routing_L3.md) · **Checkpoint Quiz 2** · [Phase 5 — The Transport Layer ▶](Phase_5_Transport_Layer.md)
