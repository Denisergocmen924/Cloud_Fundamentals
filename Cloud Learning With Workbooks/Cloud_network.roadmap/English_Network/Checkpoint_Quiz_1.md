# Checkpoint Quiz 1 — Phases 0–2: Model, Address and Division

> **Navigation:** [◀ Phase 2 — Subnetting and CIDR](Phase_2_Subnetting_and_CIDR.md) · **Checkpoint Quiz 1** · [Phase 3 — Local Network (L2) ▶](Phase_3_Local_Network_L2.md)

---

## What does this quiz measure?

The "Test yourself" tests at the end of a phase probe a single phase. A checkpoint quiz measures something
**different**: *"Can you connect these three phases to each other?"*

Phase 0 gave you the **map**: layers, encapsulation, PDUs. Phase 1 put the **addresses** onto the map: MAC,
IP, port, DHCP. And Phase 2 taught you how those addresses are **divided**: mask, CIDR, usable range.

In a real incident these three never stand apart. When someone says "the machine cannot get onto the network"
you use the knowledge of three phases at once: which layer the packet is at (Phase 0), which addresses it
carries (Phase 1), and whether those addresses are on the same network (Phase 2). The mask is a Phase 2
concept; but "doing ARP because of a wrong mask and never going to the gateway" is the intersection of three
phases — and it is the most common configuration mistake in networking.

**How to work through it:**

- Solve it with pen and paper; do not move to the next one without writing your answer down. Especially in
  the calculation questions, calculate **by hand**.
- The answer key says which **intersection of phases** each question stands at — a question you miss points
  not at a phase but at the **bridge** between two phases.
- There are 21 questions: Part A (connection reasoning, 1–9), Part B (a scenario: the "the new machine cannot
  get onto the network" incident, 10–15), Part C (reading commands and output, 16–21).
- Target time: ~1 hour. But the time does not matter; what matters is being able to justify in one sentence
  *why* each answer is what it is.

> **🤔 Before you start:** Complete this sentence in your own words: *"A machine's IP is right, its cable is
> plugged in, its gateway is right — but its mask is wrong. Why does the packet never leave?"* That single
> question contains all three phases. Write your answer down somewhere; we will see it in Question 5.

---

# Part A — Connection reasoning (1–9)

Every question in this part asks you to combine the knowledge of at least two phases. Give a short but
justified answer.

**1.** In Phase 0 we said "every layer uses the service of the layer below it". In Phase 1 we saw MAC and IP
separately. As a packet travels between networks, **which address changes and which stays constant** — and
this distinction is the direct consequence of which principle of Phase 0?

**2.** In encapsulation (Phase 0.2) every layer adds its own header. In Phase 1 we saw the port number. Which
layer's header carries the port number, and why does a router (a device working at L3) **normally not read**
this information?

**3.** In Phase 1 we said "an IP address states not an identity but a location". In Phase 2 we learned the
mask. Combine these two sentences: how does the mask determine which part of an IP address is "location" and
which part is "identity"?

**4.** In Phase 1.6 we saw DHCP's DORA flow. If a machine has taken a `169.254.x.x` (APIPA) address, which
step of DORA has not happened, and to which **layer** (Phase 0) does this information narrow the fault?

**5.** A machine's IP is `10.0.1.50`, its mask `/16` (wrong; it should have been `/24`), its gateway
`10.0.1.1`. The destination is `10.0.5.20`. What does the machine do with this packet and **why** does it
never send it to the gateway? Give your answer combining Phase 2's mask comparison with Phase 0's layer logic.

**6.** In Phase 2.3.2 we saw that two addresses in a subnet (network and broadcast) cannot be used; in AWS
**five** addresses are reserved. What is the reason for this difference, and which concepts of which phases do
the reserved `.1` and `.2` addresses correspond to?

**7.** In Phase 1.4 we saw that a socket is defined by the 4-tuple. Two separate browser tabs connect to the
same port of the same server. How are these two connections told apart without mixing — **which** element of
the 4-tuple differs and where does that element come from?

**8.** In Phase 2.4 we saw VLSM: subnets of different sizes. Why is variable sizing used instead of giving
everyone equally sized subnets — and which scarcity from Phase 1.3 does this choice relate to?

**9.** In Phase 0 we saw the PDU names (segment, packet, frame). In a fault report, do the sentences "frames
are being dropped" and "packets are being dropped" say **the same thing**? How does the difference change
which layer you look at in your diagnosis?

---

# Part B — Scenario: the "the new machine cannot get onto the network" incident (10–15)

> You have set up a new Ubuntu machine on an office network. The network is `192.168.10.0/24`, the gateway
> `192.168.10.1`. The machine cannot get onto the network. The six questions below are the diagnosis you do on
> this machine, in order.

**10.** Your first command is `ip addr`. In the output the interface is `UP` but there is **no** `inet` line.
What does this single observation tell you and what does it **not** tell you? Which mechanism of Phase 1 has
not worked, and what is your next step?

**11.** They fixed the DHCP server and the machine got an IP again: `169.254.8.12/16`. What does this address
mean? Up to which step of DORA (Phase 1.6) did it get, and at which one did it get stuck? Whom can the machine
talk to with this address?

**12.** DHCP finally worked: the machine got `192.168.10.57/24` and the gateway `192.168.10.1`. But one of the
administrators said "let us give it a static IP" and set this by hand: `192.168.10.57/16`. Can the machine now
ping `192.168.10.30` on its own network? What about `192.168.20.5`? Justify both answers with the mask
arithmetic.

**13.** Another machine was mistakenly given the same IP (`192.168.10.57`). The symptom is "the connection
works sometimes and drops sometimes". Why is this an **intermittent** fault — which table (Phase 1; we will
look at it in Phase 3, but its basis is here) is getting two different answers?

**14.** The administrator wants to grow the subnet and proposes `192.168.10.0/23` instead of
`192.168.10.0/24`. (a) What does the new range become? (b) How many usable addresses does it give? (c) Do the
settings of the existing machines have to be changed — why?

**15.** The root cause was found: the DHCP pool was the range `192.168.10.100–192.168.10.150` and it was full.
Three solutions are proposed: (a) widening the pool, (b) shortening the lease time, (c) growing the subnet to
`/23`. Evaluate each one with the logic of Phase 1 and Phase 2 — which is the fastest, which is the most
correct, which creates the most work?

---

# Part C — Reading commands and output (16–21)

In each of the outputs below, read the relevant lines and answer the question.

**16.** `ip addr show` output:

```
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    link/ether 3c:52:82:1a:4f:07 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.57/24 brd 192.168.10.255 scope global dynamic enp3s0
       valid_lft 42318sec preferred_lft 42318sec
```

What do `link/ether`, `inet`, `/24`, `brd` and `valid_lft` say, in order? Which mechanism of which phase does
the word `dynamic` give away? Is this machine's IP fixed?

**17.** `ip addr` from a second machine:

```
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    inet 169.254.112.9/16 scope link enp3s0
```

How is this output different from the one in 16? What does `scope link` mean and why is the `brd` line
unimportant? Can this machine ping its gateway — why?

**18.** A subnet calculation. You are given the block `172.16.40.0/22`.

```
Network address:  ?
Broadcast:        ?
Usable:           ? – ?
Total hosts:      ?
```

Calculate all four values by hand and write in one line how you found them. Then: is `172.16.43.200`
**inside** this block?

**19.** An engineer thinks these two addresses are "on the same network":

```
A: 10.10.10.100/25
B: 10.10.10.200/25
```

Are they on the same network? Calculate. If they are not, which path does a packet from A to B follow and at
which layer of Phase 0 is that decided?

**20.** `ss -tulpn` output:

```
Netid State   Local Address:Port   Peer Address:Port  Process
tcp   LISTEN  0.0.0.0:22           0.0.0.0:*          sshd
tcp   LISTEN  127.0.0.1:5432       0.0.0.0:*          postgres
udp   UNCONN  0.0.0.0:68           0.0.0.0:*          dhclient
```

Interpret all three lines. (a) Which service can be connected to from outside? (b) Which one cannot, and why?
(c) Which mechanism of which phase does port `68` on the third line belong to?

**21.** You are making a network plan. You have the block `10.20.0.0/16` and these needs:

```
web tier:  500 machines
app tier:  200 machines
db tier:    20 machines
```

Choose a suitable prefix length for each tier and propose three non-overlapping blocks. Why did you not give
them all the same size — what is the name of that technique?

---

## Answer key

At the end of every answer, the **intersection of phases** that question stands at is stated.

**1.** **IP addresses stay constant, MAC addresses change at every hop.** This is the direct consequence of
Phase 0's principle of layer independence: L3 does end-to-end addressing (the source and the final
destination), while L2 only delivers **to the next physical neighbour**. Every router opens the frame, wraps
it in a new L2 header and sends it on its way — without touching the IPs inside the packet. · *Phase 0.1 ×
Phase 1.1/1.2*

**2.** The port is carried in the **L4 (transport) header** — in the TCP or UDP header (Phase 0.2, 1.4). A
router works at L3: it makes its decision based on the **destination IP** and has no need to open the L4
header. By the logic of encapsulation every layer reads only its own header. (The exception: devices doing
NAT/PAT or acting as a firewall look at L4 too — you will see this in Phases 7 and 9; that "layer violation"
is deliberate.) · *Phase 0.2 × Phase 1.4*

**3.** The mask says **how many bits of the address are the network part** (2.1). The network part is the
"location" — it determines which network the packet goes to, and routers look at this part. The remaining
bits are the "identity" — they pick the particular machine within that network. The same IP address belongs
to a different network with a different mask; that is why **an IP address without a mask is incomplete
information.** · *Phase 1.2 × Phase 2.1*

**4.** **The Offer (or the Acknowledge) has not arrived** — the machine broadcast a Discover but got no reply
(1.6.2). APIPA means "I could not reach the DHCP server, so I made up an address for myself". This
information narrows the fault to **L2 and below** (Phase 0.1): a cable/interface problem, the wrong VLAN, or
the DHCP server itself. Do not think about L3 and above at all — the machine does not have a valid L3 address
yet. · *Phase 0.1 × Phase 1.6*

**5.** The machine thinks the destination is on its own network and **does ARP** — it never sends to the
gateway. The arithmetic: with the `/16` mask its own network is `10.0.0.0/16` and `10.0.5.20` is **inside**
that range. The machine says "my neighbour", broadcasts an ARP for that address, nobody answers (because in
reality it is on another network) and the packet is dropped (2.1.2, 3.1). The Phase 0 side is this: the
decision is made **at L3** and a wrong L3 decision causes the packet to be lost **at L2** — the symptom shows
at the lower layer, the cause is higher up. · *Phase 0.1 × Phase 1.2 × Phase 2.1*

**6.** In a classic network only the **network** (all host bits 0) and **broadcast** (all host bits 1)
addresses cannot be used (2.3.2). AWS reserves three more addresses on top, because a VPC is a software
network and it sets addresses aside for infrastructure services: **`.1` = the router/default gateway** (the
gateway concept in Phase 4.1), **`.2` = the VPC DNS resolver** (the DNS you will see in Phase 6), and `.3` is
set aside for future use. · *Phase 2.3 × Phase 1.2 (+ later Phases 4, 6)*

**7.** The element that differs is **the client's source port** (the ephemeral port, 1.4.1). For every new
connection the operating system picks an unused port (on Linux the typical range is 32768–60999). So the two
tabs' 4-tuples separate as `(source IP, **different source port**, destination IP, 443)` and the kernel
directs the incoming packets to the right socket. The other three elements are the same. · *Phase 1.4 ×
Phase 0.2*

**8.** Because **addresses get wasted** (2.4.2). If you give everyone a `/24`, a tier of 20 machines occupies
254 addresses and 234 are wasted. VLSM sizes according to need. This choice is directly related to the
**IPv4 address scarcity** in Phase 1.3: because public addresses ran out, private blocks are used carefully —
and in the cloud the same discipline applies, because a VPC CIDR is limited too. · *Phase 1.3 × Phase 2.4*

**9.** **They do not say the same thing.** A frame is an **L2** PDU, a packet is an **L3** PDU (0.3). When
someone says "frames are being dropped" you look at the switch, the cable, the MTU or the interface level;
when someone says "packets are being dropped" you look at routing, the route table or filtering. The right
term determines **which layer** the diagnosis starts from — that is why PDU names are not a formality but a
diagnostic shortcut. · *Phase 0.3 × Phase 0.1*

**10.** It tells you: **the physical layer and the interface are healthy** — the cable is plugged in, the
driver works (`UP` and `LOWER_UP`). It does not tell you: why no address could be obtained. The mechanism
that has not worked is **DHCP** (1.6): the machine either cannot send a Discover or cannot get a reply. The
next step: look at `journalctl -u systemd-networkd` or the `dhclient` log; in parallel, confirm whether the
DHCP server is up. · *Phase 0.1 × Phase 1.6*

**11.** `169.254.x.x` = **APIPA / link-local** (1.6.3): it means "I could not get a reply from the DHCP
server". In DORA, **the Discover was sent but no Offer was received** (1.6.2). With this address the machine
can only talk to other link-local machines on the same physical segment — it cannot reach the gateway,
another subnet or the internet, because this address is not routable. · *Phase 1.6 × Phase 0.1*

**12.** **It can ping `192.168.10.30` on its own network** — with the `/16` mask its network becomes
`192.168.0.0/16` and `.10.30` is in that range, so it reaches it directly with ARP (which works because the
local switch is on the same physical segment). **But it cannot ping `192.168.20.5`**: the machine thinks that
one is "on my network" too (because `/16` covers it), does ARP, gets no reply and the packet is dropped — it
never goes to the gateway (2.1.2). This is the same mistake as in Question 5 and it is the most commonly seen
mask error in the field. · *Phase 2.1 × Phase 1.2*

**13.** Because when two machines announce the same IP, the **ARP table** (we will see the detail in Phase
3.1) gets two different MAC answers, and depending on which one it has cached, the traffic goes now to one
machine, now to the other. Whoever gave the last ARP reply pulls the traffic. The symptom is intermittent
because ARP entries have a **timeout** and the winner can change at every refresh. This is a violation of
Phase 1's rule "an address must be unique". · *Phase 1.2 × Phase 3.1 (preview)*

**14.** (a) `192.168.10.0/23` → `192.168.10.0` – `192.168.11.255`. (b) 512 in total, **510 usable** (the
network and broadcast are subtracted, 2.3.2). (c) **Yes, it is needed:** the existing machines still work
with the `/24` mask and will count the machines in `192.168.11.x` as "another network" and send them to the
gateway (2.1.2). The mask change must be made **on every machine** — if it is handed out by DHCP it arrives
automatically when the lease is renewed; the ones set by hand must be fixed one by one. · *Phase 2.2 ×
Phase 2.3*

**15.** (a) **Widening the pool** — the fastest and least risky; if there are free addresses inside the `/24`
(for example `.51–.99` and `.151–.250`) it is solved with a single setting. (b) **Shortening the lease time**
— it reclaims dead entries faster (1.6.2), which is the right move on networks with many guest devices, but
if the pool really is small it only buys time. (c) **Growing to `/23`** — it solves the address problem at
the root but **creates the most work**: every machine's mask must change (Question 14c). The right order:
(a) first, and (c) in a planned way if the growth is permanent. · *Phase 1.6 × Phase 2.2/2.4*

**16.** `link/ether` = the **MAC address** (the L2 identity, 1.1); `inet` = the **IPv4 address** (L3, 1.2);
`/24` = the **mask** — the network part is 24 bits (2.2); `brd 192.168.10.255` = this subnet's **broadcast**
address (2.3.2); `valid_lft 42318sec` = **the remaining time of the DHCP lease** (1.6.3). The word `dynamic`
gives away that the address was obtained **via DHCP**. No, this IP is **not fixed** — it is renewed when the
lease expires and (rarely, but still) it can change. · *Phase 1.1/1.2 × Phase 1.6 × Phase 2.3*

**17.** The difference: the address is `169.254.x.x` (APIPA) and instead of `scope global` there is **`scope
link`**. `scope link` says that this address is valid only on the **directly attached segment** and cannot be
routed. The reason the `brd` line is unimportant is that you cannot get off the network with this address
anyway. **It cannot ping its gateway:** the gateway `192.168.10.1` is on a different network and the machine
has neither a route to it nor a valid source address (1.6.3). · *Phase 1.6 × Phase 2.1*

**18.** `/22` → 10 host bits in the last two octets; the block size in the 3rd octet is **4**. The number
`40` is a multiple of 4, so: **Network = `172.16.40.0`**, **Broadcast = `172.16.43.255`**, **Usable =
`172.16.40.1` – `172.16.43.254`**, **Total hosts = 1022** (1024 − 2). How: `32 − 22 = 10` host bits →
`2^10 = 1024` addresses → the block is 4 units in the 3rd octet (`40–43`). **Yes**, `172.16.43.200` is inside
the block (it is in the `40–43` range and it is not the broadcast). · *Phase 2.2 × Phase 2.3*

**19.** **They are not on the same network.** The `/25` block size is 128: the first block is
`10.10.10.0–127`, the second block `10.10.10.128–255`. A (`.100`) is in the first block, B (`.200`) is in the
second. So A counts B as "another network" and sends the packet to the **default gateway**; from there it is
routed (the detail in Phase 4). The decision is made **at L3**: the machine compares the destination IP with
its own mask (2.1.2). Even if they are plugged into the same physical switch the result does not change —
this distinction is logical, not physical. · *Phase 2.1/2.3 × Phase 0.1*

**20.** (a) **`sshd`** — `0.0.0.0:22` listens on all interfaces, it can be connected to from outside (1.4.2).
(b) **`postgres`** — `127.0.0.1:5432` listens only on **loopback**; even if connections from outside reach
the machine, the kernel does not direct them to this socket. It is not a firewall rule but a **bind address**
problem, and in the field it is often mistaken for a firewall. (c) UDP **68** is the DHCP **client** port
(the server is 67) — the DORA flow in Phase 1.6 runs over this port pair; the `UNCONN` state shows that UDP
is connectionless. · *Phase 1.4 × Phase 1.6*

**21.** The smallest sufficient block for the need (VLSM, 2.4.2): **web 500 → `/23`** (510 usable), **app
200 → `/24`** (254), **db 20 → `/27`** (30). A non-overlapping proposal: `web 10.20.0.0/23`
(`10.20.0.0–10.20.1.255`), `app 10.20.2.0/24`, `db 10.20.3.0/27`. I did not give them all the same size
because giving everyone a `/23` would waste more than 1000 addresses on 20 machines; the name of the
technique is **VLSM (Variable Length Subnet Masking)**. Note: a `/24` is not enough for 500 machines — that
is why the web tier goes up one block. · *Phase 2.4 × Phase 2.3*

---

## Scoring

| Correct | Assessment |
|---|---|
| 19–21 | The model, the addresses and the division have merged into a single mental map. Move on to Phase 3 with confidence. |
| 15–18 | Solid. Go back around the **bridge** (not the single phase) that the questions you missed point at. |
| 10–14 | You know the phases separately but the link between them is weak. Use the table below. |
| 0–9 | Redo Phases 1 and 2, especially the mask arithmetic and the "When this phase breaks" sections; until these bridges settle, Phase 3 will feel meaningless. |

**Which question you miss → where to go back:**

| Question you missed | Go back — this bridge is weak |
|---|---|
| 1, 2, 9 | Phase 0.1/0.2/0.3 — layers, encapsulation, PDU names |
| 3, 5, 12, 19 | Phase 1.2 × Phase 2.1 — the mask and the "same network?" decision |
| 4, 10, 11 | Phase 1.6 × Phase 0.1 — DHCP/DORA and the layer meaning of APIPA |
| 6, 14, 18 | Phase 2.2/2.3 — CIDR arithmetic, network/broadcast, reserved addresses |
| 7, 20 | Phase 1.4 — port, socket, 4-tuple, bind address |
| 8, 21 | Phase 1.3 × Phase 2.4 — address scarcity and VLSM |
| 13, 16, 17 | Phase 1.1/1.2 × Phase 1.6 — address uniqueness, lease, link-local |

---

## Closing — from here to Phase 3

Phases 0, 1 and 2 together gave you the **static** picture of the network: layers, addresses, and how those
addresses are divided. You can now look at a machine and answer the question "which network is this address
in, whom can it talk to directly, and whom does it need help to talk to".

But there is one thing you never asked: **how does the machine actually get the packet to the address it
calls "my neighbour"?** In Phase 2 we said over and over "if it is on the same network it sends directly" —
but knowing the IP address is not enough. The thing that walks on the wire is a frame, and a frame needs a
**MAC address** (Phase 0.3, 1.1). Where does that MAC come from?

Phase 3 connects exactly here: **ARP.** It will open up the mechanism underneath the sentence "if it is on
the same network it sends directly", you will see how a switch learns its MAC table, and you will understand
exactly why the duplicate-IP problem we touched on in Question 13 is intermittent.

> **Before you continue:** If you can answer Questions 5 and 12 above without hesitation — why a machine with
> a wrong mask never goes to the gateway — you are ready for Phase 3. If you cannot, go back around Phase 2.1
> and 2.3; Phase 3 will continue assuming that the "is it on the same network?" decision is made **correctly**.

---

> **Navigation:** [◀ Phase 2 — Subnetting and CIDR](Phase_2_Subnetting_and_CIDR.md) · **Checkpoint Quiz 1** · [Phase 3 — Local Network (L2) ▶](Phase_3_Local_Network_L2.md)
