# Phase 0 — Mental Model: Why Do Layers Exist?

> **Navigation:** **Phase 0** · [Phase 1 — Addressing ▶](Phase_1_Addressing.md)

---

## Where we come from

Nowhere — this is the beginning. But you already hold more than you think: you have typed an address into a
browser countless times, watched a video, downloaded a file. All of it worked and you never once thought about
it. The job of this phase is to turn that "it works" feeling into a **model**.

Let us start with a warning, because it sets the tone of the entire book: **learning networking is not
memorising protocols.** There are plenty of people who can recite OSI's seven layers in order and still have no
idea where to look when someone says "I cannot reach this service". The difference is not in the memorising, it
is in the **mental model**. This book builds the network not as a list of protocols but as a **failure map**:
next to every topic sits the question "when this piece breaks, how does it show up in the system?"

If you have worked on the Linux side before, remember the lesson that a resource **existing** and a resource
being **reachable** are two different things. Networking is the purest form of that lesson: a server can be up,
the service can be running, and the packet can still never reach you. Finding where it died is the single
purpose of this book.

## The question of this phase

> *"When I type `ping`, how does that data leave my machine and get to the other side — and who adds what,
> and when, along the way?"*

The question looks innocent but its answer is the skeleton of the whole book. Because data does **not fly as a
single piece**: every layer adds its own information, and on the other side every layer strips off what it
added. Without internalising this wrap-unwrap cycle no troubleshooting instinct forms — because a fault always
happens **in a specific layer**, and the symptom carries that layer's signature.

Think of it this way: "the internet is not working" is not a diagnosis, it is a **complaint**. An engineer
translates it into: "is it L2 (I cannot find my neighbour), L3 (there is no path to the destination), L4 (the
connection will not establish), or L7 (the connection is there but the application errors out)?" To be able to
make that translation you first have to understand **why** the layers exist. Phase 0 is exactly that.

By the end of this phase you will be able to draw a packet's journey on paper in your own words — and when you
hear a failure symptom the question "which layer's job is this?" will run through your head automatically.

---

## By the end of this phase

- You will be able to explain with an example **why** layered architecture exists (to split the problem, to be
  able to change layers independently)
- You will be able to map OSI's 7 layers onto the 4 layers of TCP/IP used in practice, and you will know which
  one is the **model** and which one is the **reality**
- You will be able to explain what "every layer talks only to its peer" (the peer-to-peer illusion) means, and
  why it is an **illusion**
- You will be able to draw encapsulation and decapsulation step by step, saying which layer adds which header
- You will be able to match the PDU names (frame → packet → segment → data) to the right layer, and you will
  know the difference between someone saying "a packet was dropped" and "a frame was dropped"
- You will be able to say **which layer** you would start searching in when you hear a failure symptom
- **Cloud:** you will be able to explain that a packet inside a VPC passes through the same layers; that the
  cloud does not remove the layers, it only **abstracts** them

---

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 0.1 | The layer idea and the models (OSI / TCP-IP) | `[concept]` | Why we slice — the roof of the whole map |
| 0.2 | Encapsulation and decapsulation | `[mechanism]` | **The heart of the phase** — the engine of the data's journey |
| 0.3 | PDUs — the name of data at each layer | `[concept]` | Right term = right layer = right diagnosis |
| 0.4 | When this phase breaks | — | The failure signatures of mixing up layers |

> **How to work through this phase:** This phase is almost entirely conceptual, and that is a good thing — what
> you learn here is not a command but a **point of view**. Command boxes (🔧) are few here and all of them are
> 🟢 read-only (`ip link`, `ping`, watching a single packet with `tcpdump`). You can finish the phase without
> running any of them; if you do run them, the model becomes concrete in your hands. **Your real working tool is
> pen and paper:** while reading 0.2, draw the wrapping of a packet with your own hand. It is possible to "read
> this phase and move on", but it is wasted effort — every phase from 1 to 11 comes back to this phase's
> language. Twenty minutes of drawing here buys you hours of clarity later.

---
---

# 0.1 Layers: Why We Slice the Problem

## 0.1.1 The layer idea: breaking a problem into pieces `[concept]`

Getting two computers to talk is genuinely a hard problem. Think about it: how will you put an electrical
signal on the wire, how will you tell which machine is which, what do you do if the cable is faulty, how do you
find a path to a machine on the other side of the world, how will you notice if data is lost on the way, and
**which application** on the far side do you deliver it to... If you tried to write all of that as one giant
program, what you would get is a monster nobody could change and no part of which you could test on its own.

In engineering, the standard solution to this kind of problem is **layering**: split the big problem into
slices, each responsible for **a single job**, and put a clear interface between the slices. Every layer:

- **takes a service from the layer below it** ("put this on the wire for me"),
- **does its own job** ("let me decide the destination"),
- **offers a service to the layer above it** ("I will carry your data to the destination").

The real gain is this: **you can change one layer without touching the others.** Wi-Fi and an ethernet cable are
completely different physical technologies — but the IP layer above them does not care about either. Your
browser never asks "am I on Wi-Fi or on a cable?" In the same way, moving from IPv4 to IPv6 did not require
rewriting browsers. Layering is the only reason technology can evolve piece by piece.

And the real gain for you: **layers bound the fault.** A problem almost always starts in a single layer. When
you split the symptom "no internet" into layers, you search in one slice instead of in the whole universe. This
entire book rests on that single sentence.

> **⚠️ Common misconception: "Layers are real things, they physically exist inside the packet."**
>
> Layers are a **design idea**, not a physical reality. What flows in the cable is only electrical (or light,
> or radio) signals; there is no place in there called "layer 3". Layers are an agreement about **how the bits**
> inside that signal **are to be interpreted**: the first so many bits are for addressing, the next so many for
> another job... A layer is not a label on top of the data; it is a **contract about who reads which bit**.
> That is why "look at the packet's layer 3" actually means "read the fields at this offset of the packet".
> Knowing this is what makes you understand what you are looking at when you later stare at `tcpdump` output.

## 0.1.2 Two models: OSI's 7 and TCP/IP's 4 `[concept]`

Two different models make the layering idea concrete — and their **roles differ**, so let us make that
distinction clear from the start, because it gets confused a lot.

**The OSI model (7 layers)** is a **reference model**: academic, complete, designed for teaching and talking —
a shared language. Nobody runs "OSI protocols"; but everybody uses OSI's numbering when they say "this is an L2
problem". That is OSI's real job: **a common numbering language.**

| # | Layer | Its job in one sentence | Typical example |
|---|---|---|---|
| 7 | Application | The work the user actually wants | HTTP, DNS, SSH |
| 6 | Presentation | Data format, encryption, encoding | TLS (placed here in practice) |
| 5 | Session | Start / maintain / end the conversation | (not handled separately in practice) |
| 4 | Transport | End-to-end reliability + which application | TCP, UDP |
| 3 | Network | Addressing and path-finding between networks | IP, ICMP |
| 2 | Data link | Delivery to a neighbour on the **same** local network | Ethernet, Wi-Fi, ARP |
| 1 | Physical | Turn bits into a signal and carry them | Cable, fibre, radio |

**The TCP/IP model (4 layers)** is the one that **actually runs** — the internet runs on it. It collapses OSI's
top three layers into a single "Application" layer and merges the bottom two into "Network Access":

| TCP/IP layer | Equivalent (OSI) | What it does |
|---|---|---|
| Application | 7 + 6 + 5 | The application's work (HTTP, DNS, SSH, TLS) |
| Transport | 4 | End to end: TCP / UDP |
| Internet | 3 | The path between networks, with IP |
| Network Access (Link) | 2 + 1 | Local network + physical carriage |

In practice engineers speak a **hybrid**: they use OSI's numbers (L2, L3, L4, L7) with TCP/IP's reality. "L7
load balancer", "L4 firewall", "L2 switch" — all of these are an OSI number plus a real-world device. You will
almost never hear L5 and L6; that is why this book also focuses on the four numbers used in practice (L2, L3,
L4, L7).

> **💡 Cloud connection — AWS service names speak in OSI numbers:** The names you meet in the cloud console use
> these numbers directly. An **ALB** (Application Load Balancer) is an **L7** device: it decides by looking at
> the HTTP path and headers. An **NLB** (Network Load Balancer) is an **L4** device: it only sees IP + port, it
> does not look at content — which makes it faster but more "blind". A **security group** filters at L3/L4 (IP +
> port) and does not read content. So being able to read AWS documentation requires knowing this table. In Phase
> 11 we will map every one of them; for now, one note: the cloud does **not** remove the layers — it only
> manages some of them for you.

## 0.1.3 The peer-to-peer illusion: every layer talks to its peer `[concept]`

This is the most elegant idea in the layered model, and once it settles the network suddenly gets simpler.

Your browser (L7) behaves as if it is talking to the L7 of the web server on the other side. It says "give me
this page", and the page arrives. The browser knows no MAC address, has no idea routers exist, and does not
care how many hops the packet took. In the same way, the TCP on your machine (L4) talks to the TCP on the other
machine: "did you receive the 1461st byte I sent you?" The routers in between never open up TCP to look.

This is called the **peer-to-peer communication illusion**: **every layer behaves as if it were talking directly
to the layer with the same number on the other side.** Why an illusion? Because in reality no data travels
horizontally. Data goes **down** on your machine (L7→L4→L3→L2→L1), crosses the wire, and goes **up** on the
other machine (L1→L2→L3→L4→L7). The horizontal arrow is a logical fiction; the vertical arrow is the physical
truth.

But the illusion is not pointless — it is **the foundation of diagnosis**. Because you become able to ask: "can
my L4 reach their L4?" If the TCP handshake does not complete (Phase 5), the problem is at the L4 peering or in
the layers **below** it — there is no point poking at L7. A layer can only work if the layers below it work.
That single rule is the core of the whole troubleshooting methodology in Phase 10: **verify from the bottom up.**

> **🤔 Think 0.1** — A colleague says "the site will not open" and you see with `ping` that you can reach the
> server's IP (replies come back). (a) What does `ping` working prove is **healthy**? (b) Which layers does this
> observation **leave** on the suspect list? (c) Why is this a practical application of the "peer-to-peer
> illusion" idea?
>
> *(Answer: at the end of the phase)*

---
---

# 0.2 Encapsulation and Decapsulation

## 0.2.1 Wrapping: each layer adds its own envelope `[mechanism]`

Now we reach the heart of the phase. Above we said "every layer does its own job" — but what does that job look
like **physically**? The answer: every layer puts the data it received from above into an **envelope** and
writes its own information on that envelope. That information is called a **header**, and the operation is
called **encapsulation**.

The letter analogy really works here. You write a letter (data). You put it in an envelope and write the
recipient's address on it (L3 — where the destination is). You put that envelope into a courier bag and stick a
label on the bag saying "this parcel goes to that branch" (L2 — the next stop). The courier branch opens the
bag, throws the label away, sticks a new label on and sends it to the next branch. **The inner envelope is never
opened.** Only the outermost label changes at every stop.

The same thing happens in the network. Your browser produces an HTTP request; as that request goes down, every
layer **prepends** its own header:

1. **L7 (Application)** — The data is produced: `GET /index.html HTTP/1.1` and its headers. This is called
   **data**.
2. **L4 (Transport)** — TCP prepends a **TCP header** to this data. What is in it? Source port, destination
   port (80), sequence number, ACK number, flags. The unit is now called a **segment**. The destination port
   here says **which application** it will be delivered to on the far machine.
3. **L3 (Network)** — IP prepends an **IP header** to the segment: source IP, destination IP, TTL, protocol
   number. It is now a **packet**. The destination IP here says **which machine** it is going to.
4. **L2 (Data link)** — Ethernet prepends an **ethernet header** to the packet: source MAC, destination MAC,
   and appends an error-checking field (FCS) at the end. It is now a **frame**. The destination MAC here says
   **the next physical stop** — not the destination machine.
5. **L1 (Physical)** — The frame's bits are turned into a signal and put on the wire / into the air.

Notice: **the data never changed, it only got thicker around the edges.** What flows in the cable is a series of
nested envelopes. This is why a packet on the network is always bigger than its real data — every layer brings
"overhead". When we talk about MTU in Phase 5 you will see why that overhead is vital.

![Figure 0.1 — Encapsulation: as an HTTP request goes down, each layer adds its own header (data → segment → packet → frame), and on the far side decapsulation strips the same headers in reverse order. The data never changes; only the envelopes around it are added and removed.](../diagrams/png/nw-0-01-encapsulation.png)

In the figure, the vertical arrow on the left shows the descent on your machine (encapsulation) and the vertical
arrow on the right shows the ascent on the far machine (decapsulation). The horizontal arrow in the middle is
the peer-to-peer connection that does **not** actually exist but that every layer assumes exists (0.1.3). Each
row's box carries the new name the data takes at that layer.

> **🔧 See it on your machine** 🟢 — see the layers inside one frame at the same time
>
> ```
> $ sudo tcpdump -n -c 1 -v icmp
> listening on ens5 ...
> 14:02:11.377 IP (tos 0x0, ttl 64, id 4711, proto ICMP (1), length 84)
>     172.31.20.15 > 8.8.8.8: ICMP echo request, id 3, seq 1, length 64
> ```
>
> Three layers appear in a single line: `IP (... ttl 64 ... proto ICMP)` is the **L3** header;
> `172.31.20.15 > 8.8.8.8` are L3's source/destination addresses; `ICMP echo request` is the payload
> **inside** L3. `length 84` = IP header (20) + ICMP payload (64). That is exactly what `tcpdump` does: it
> opens the envelopes from the outside in and shows them to you. You do not have to understand this in this
> phase — just see that **the envelopes are really there**. In Phase 10, `tcpdump` will be your final referee.

## 0.2.2 Unwrapping: stripped in the opposite order on the far side `[mechanism]`

What arrives at the far machine is a stream of bits. There the exact opposite happens — this is called
**decapsulation**:

1. **L1** reads the bits back from the signal and hands them to L2.
2. **L2** looks at the ethernet header: "is the destination MAC mine?" If not, it **drops** the frame
   (switches do this differently — Phase 3). If it is, it strips the header, checks integrity with the FCS,
   and hands the contents to L3. A field in the header says "there is IP inside"; that is how L2 knows whom to
   hand it to.
3. **L3** looks at the IP header: "is the destination IP mine?" If not, it either drops it or (if it is a
   router) forwards it — Phase 4. If it is, it strips the header and looks at the `proto` field: TCP, UDP or
   ICMP? It hands the contents to the right L4.
4. **L4** looks at the TCP header: what is the **destination port**? If it is 80, it delivers to the process
   listening on 80 (Phase 1.4). It checks sequence numbers and produces an ACK if needed (Phase 5).
5. **L7** finally receives the pure data: `GET /index.html HTTP/1.1`. The web server can now do its job.

The single idea here is very powerful: **every layer's header says who the next layer is.** The L2 header says
"there is IP inside", the L3 header says "there is TCP inside", the L4 header says "destination port 80". That
is how the chain unravels. It is like a funnel where the "to whom" information on the envelope gets one notch
sharper at every level: **which machine → which protocol → which application.**

> **⚠️ Common misconception: "Every device along the way opens all the layers."**
>
> No — and this is the answer to how the network can be fast. Devices in between open **only as far as the
> layer they need**. A **switch** (an L2 device) looks only at the ethernet header; it never sees the IP
> inside and does not care. A **router** (an L3 device) strips L2, looks at the IP header, makes its decision,
> then attaches a **new** L2 header and sends it on — but it never opens TCP. Only the **two ends** read the
> TCP header. That is why TCP is called an end-to-end protocol: the ones in between carry it but do not read
> it. And that is why looking for a TCP problem in router logs is pointless. (The **stateful firewalls** you
> will see in Phase 9 are the deliberate exception to this rule — which is exactly why they count as special
> devices.)

> **❓ A question that comes to mind: "If the router strips the L2 header and attaches a new one, does the
> destination MAC address change along the way?"**
>
> Yes — and this is the key to the whole of Phase 3 and Phase 4. The **source and destination IP** (L3)
> generally stay **constant** through the journey: they are the end-to-end identity. But the **source and
> destination MAC** (L2) **change at every hop**: because MAC means "the next physical stop", not "the final
> destination". In the courier analogy: the home address on the envelope never changes, but at every branch the
> "next branch" label on the bag is renewed. Settle this distinction now — in Phase 1 we will build why MAC is
> only meaningful on the local network, in Phase 3 why ARP exists, and in Phase 4 exactly what a router does,
> all on top of this one sentence.

> **🤔 Think 0.2** — The data of an HTTP request is 100 bytes. As this data leaves the wire its total size will
> be **bigger** than 100 bytes. (a) Because of which three headers? (b) In which order are these headers
> stripped on the far side? (c) How many of these three headers does a switch in between read, and how many
> does a router read?
>
> *(Answer: at the end of the phase)*

---
---

# 0.3 PDUs — The Name of Data at Each Layer

## 0.3.1 Frame, packet, segment, data: four names, one piece of data `[concept]`

You may have noticed above: the same data took a **different name** at every layer. These names are called
**PDUs** (Protocol Data Units). The table is short but deserves more than memorising:

| Layer | PDU name | What is inside | Its address |
|---|---|---|---|
| L7 Application | **Data** | Pure application data | — |
| L4 Transport | **Segment** (TCP) / **Datagram** (UDP) | TCP/UDP header + data | Port (which application) |
| L3 Network | **Packet** | IP header + segment | IP (which machine) |
| L2 Data link | **Frame** | Ethernet header + packet + FCS | MAC (which neighbour) |
| L1 Physical | **Bit** | 1s and 0s turned into a signal | — |

Why four names for the same data? Because **the name tells you which envelope is outermost at that moment.**
When you say "frame" you are talking about something that has an ethernet header; when you say "packet",
something that has an IP header. So the PDU name is a **layer label** — and that sets the precision of
conversation between engineers.

In everyday speech everybody calls everything a "packet" ("we are dropping packets", "capture the packets").
That is usually harmless. But at diagnosis time the difference becomes critical:

- **"Frames are dropping"** → an L2 problem: cable, NIC, switch port, duplex mismatch, CRC errors. Local.
- **"Packets are dropping"** → an L3 problem: routing, TTL, MTU, firewall, a remote hop. The end-to-end path.
- **"Segments are being retransmitted"** → an L4 problem: loss + resend, congestion. The reliability layer.

All three give the symptom "data is being lost", but **the place to look differs** for all three. The
difference between an engineer saying "we have packet loss" and "the CRC error counter is climbing" narrows the
search by kilometres.

> **💡 Cloud connection — which PDU do you see in the cloud:** In the cloud you generally **never see** L1 and
> L2 — no cable, no switch, a virtual NIC. That is good news: the cable/duplex class of faults leaves your life.
> But it has a price: **because you cannot see L2, your diagnostic tools start at L3.** VPC Flow Logs (Phase
> 10.4) give you records at the **packet** level — source IP, destination IP, port, ACCEPT/REJECT. You cannot
> see frames, because the cloud provider manages that layer on your behalf. So saying "packet" in the cloud is
> usually genuinely correct; it is the lowest unit you can see.

> **🤔 Think 0.3** — You see the "CRC error" counter climbing in a server's NIC statistics. At the same time the
> application team says "TCP retransmits are very high". (a) Which PDU/layer does the CRC error belong to?
> (b) Which PDU/layer does the TCP retransmit belong to? (c) Can a **cause-and-effect** relationship be drawn
> between the two — which one leads to the other?
>
> *(Answer: at the end of the phase)*

---
---

# 0.4 When This Phase Breaks — The Signatures of Mixing Up Layers

Phase 0 is not a command phase but a **model** phase — so its "breaking" looks different too. What breaks here
is not the system itself but **your map**: searching in the wrong layer. The table below shows how time is lost
in the field when this phase's concepts have not settled.

| Symptom / behaviour | The underlying model error | The right reflex | Related section |
|---|---|---|---|
| Saying "no internet" and resetting the modem | No layer distinction; a single "internet" is assumed | First: which layer — L2, L3, L4 or L7 | 0.1.1 |
| Saying "the network is fine" because `ping` works | `ping` tests L3, not L4/L7 | L3 healthy ≠ the service is reachable | 0.1.3 |
| Looking for an application error in router logs | The router is assumed to read L4/L7 | The router stops at L3; L4 is end to end | 0.2.2 |
| The "the packet got bigger" surprise (MTU exceeded) | Forgetting that headers bring overhead | Every layer adds bytes; compute the total size | 0.2.1 |
| Looking up the MAC address of a remote server | Assuming MAC is end to end | MAC is local and changes at every hop; IP is end to end | 0.2.2 |
| Saying "packet loss" and replacing the cable | PDU names not matched to layers | Frame, packet or segment — which one is dropping | 0.3.1 |
| Starting the search at L7 (the application) | Going up before the lower layers are verified | Verify bottom up: link → IP → path → port → app | 0.1.3 |

> **The lesson from this table:** The single output of this phase is a reflex: **"which layer's job is this?"**
> When you hear a symptom, the first thing you do is place it in a layer — because the layer determines **where**
> to search. Two rules carry this: (1) **A layer can only work if the layers below it work** — which is why
> diagnosis is always done from the bottom up (that is the whole methodology of Phase 10). (2) **Terms are layer
> labels** — saying "frame", "packet" or "segment" sends the search to different places; do not use them
> loosely. Take these two rules now; from Phase 1 on we will put flesh on them in every phase.

---
---

# Phase 0 — Answers to the Think questions

## Answer 0.1 — `ping` proves L3 and leaves L4 and L7 suspect

(a) `ping` is an **ICMP** message and works at **L3**. A reply arriving proves: there is a physical connection
(**L1**), you can reach your neighbour/gateway on the local network (**L2**), and the **path** to the
destination IP works in both directions (**L3** — outbound *and* return, because the reply came back). So the
bottom three layers are healthy.

(b) The layers that remain suspect are **L4 and L7**: the destination machine is on the network, but (i) the
service may not be listening on that port, (ii) a firewall may be closing that **port** (a rule that allows
ICMP but blocks TCP 443 is very common), (iii) the TCP handshake may not be completing, (iv) the connection may
be established but the application returns an error. The success of `ping` excludes **none** of these.

(c) This is the peer-to-peer illusion in its directly practical form: with `ping` you tested that your **L3**
can talk to their **L3** — only that pairing. Every layer "talks" to its own peer separately, so every layer is
**tested** separately. A layer working never proves that the layers above it work; it only proves that the ones
below it work.
**Related section:** 0.1.3 · **Next:** Phase 10.1 (layer-by-layer methodology).

## Answer 0.2 — Three headers are added, stripped in reverse, and the devices in between look at different depths

(a) Three headers are added: the **TCP header** (L4 — source/destination port, sequence number), the **IP
header** (L3 — source/destination IP, TTL) and the **Ethernet header** (L2 — source/destination MAC; an FCS is
also appended at the end). So 100 bytes of data grows noticeably with each layer's overhead as it goes out on
the wire. That "growth" becomes critical in Phase 5.7 when we discuss MTU: the total size a frame can carry is
limited, and the headers take their share of that budget.

(b) In reverse order, **from the outside in**: first the Ethernet header (L2) is stripped, then the IP header
(L3), then the TCP header (L4), and at the end the pure data is delivered to L7. The order is not arbitrary but
mandatory — every header says what the next one is.

(c) A **switch** (L2) reads only **one** header: the ethernet header (destination MAC). It never sees the IP
inside. A **router** (L3) reads **two** headers: it strips the ethernet header, looks at the IP header and
makes its decision — then attaches a new ethernet header and sends it on. **Neither of them reads** the TCP
header; only the two end machines read it.
**Related section:** 0.2.1-0.2.2 · **Next:** Phase 5.7 (MTU), Phases 3-4 (switch and router).

## Answer 0.3 — CRC is L2, retransmit is L4, and one causes the other

(a) A **CRC error** is an **L2 / frame** event: the FCS field at the end of the ethernet frame checks whether
the frame was corrupted on the way. If it does not match, the frame is **dropped**. The causes are physical: a
bad cable, a loose connector, electrical noise, a faulty NIC/switch port, a duplex mismatch.

(b) A **TCP retransmit** is an **L4 / segment** event: when no ACK arrives for a sent segment, TCP sends it
again (Phase 5.3). TCP knows the data was lost but does not know **why** it was lost.

(c) Yes, there is a clear cause and effect, and the direction is **bottom up**: every frame dropped at L2 takes
with it the packet inside it and the segment inside that. TCP gets no ACK for that segment, times out and
**resends**. So **the CRC errors are the cause and the retransmits are the effect.** The right intervention is
not at L4 (TCP tuning) but at **L2**: fix the cable/port/NIC. This is the cleanest example of the "diagnose
from the bottom up" rule — the noisy symptom in the upper layer is the shadow of a silent fault in the lower
one.
**Related section:** 0.3.1, 0.1.3 · **Next:** Phase 5.3 (retransmission), Phase 10.1.

---
---

# Phase 0 — Frequently asked questions

**Q1 — Do I really have to memorise OSI?** Reciting the seven layers in order is an exam skill; what is useful
is **what the numbers mean**. In practice it is enough to internalise four numbers: **L2** local network/MAC,
**L3** IP and path, **L4** port and reliability, **L7** application. You will almost never hear L5 and L6 in
the field (TLS is usually waved off as "around L6") (0.1.2).

**Q2 — Why is OSI still used when the TCP/IP model exists?** Because OSI is a **language for talking**.
Devices run TCP/IP, people speak in OSI numbers: "L7 load balancer", "L2 switch", "L3 routing". The two are not
rivals — one is reality, the other is shared terminology (0.1.2).

**Q3 — Does encapsulation encrypt the data?** No, the two are completely different things. Encapsulation only
**wraps**: it prepends addressing and control information and does not touch the content — you can read the
content with `tcpdump`. Encryption is a separate job and is usually done **around L7** (with TLS) (Phase 8.4).
Even in encrypted HTTPS traffic the IP and TCP headers are **in the clear**; only the carried payload is
encrypted (0.2.1).

**Q4 — Why are there both IP and MAC addresses, is one not enough?** They answer different questions. **MAC**
says "which physical device on this local network" and **changes at every hop**; **IP** says "which machine on
the internet" and stays **constant** end to end. IP is hierarchical (it can be grouped and routed), MAC is flat
(it cannot be grouped). You cannot find a path at internet scale with MAC alone — that is the subject of Phase
1 and Phase 4 (0.2.2).

**Q5 — The difference between frame, packet and segment in one sentence?** They are the names of the same data
at different envelope layers: **segment** = with a TCP header (L4), **packet** = with an IP header (L3),
**frame** = with an ethernet header (L2). The name says which layer **the outermost header** belongs to at that
moment (0.3.1).

**Q6 — Do layers exist in the cloud too, or did AWS remove them?** They all exist — AWS does not **remove**
them, it **manages** some of them for you. A packet inside a VPC is encapsulated in exactly the same way and
carries the same headers. The difference: you do not see or manage L1/L2. That is why cloud diagnosis starts at
L3 (Flow Logs are at packet level) (0.3.1).

**Q7 — When can I say the sentence "this is an L3 problem"?** When the symptom is about **addressing or
path-finding**: the destination cannot be reached at all, `ping` gets no reply, TTL exceeded comes back, you
cannot get out to a different network while the local network works. If the symptom is "the connection is
established but no data flows" you look at L4; if it is "it connects but returns an error", at L7. Phase 10
will turn this mapping into a full decision tree (0.1.3, 0.4).

---
---

# Phase 0 — Test yourself

Write your answers on paper, then compare them with the answer key. Target: 14+ out of 18.

## Part A — Definition and mechanism

1. Write two fundamental benefits of layered architecture (one about design, one about diagnosis).
2. List OSI's 7 layers in order and write each one's job in **a single word**.
3. Which OSI layers do TCP/IP's 4 layers correspond to? Map them in a table.
4. What does the "peer-to-peer illusion" mean, and why is it an **illusion**?
5. During encapsulation, which address/identity information is in the headers added by L4, L3 and L2,
   **respectively**?
6. In decapsulation, how does a layer know which layer to hand its contents to?
7. Match the four PDU names (data, segment, packet, frame) to their layers and write each one's address type.
8. How many headers deep does a router open an incoming frame, and how many does a switch? Explain the
   difference.

## Part B — Apply and diagnose

9. `ping 8.8.8.8` works but `curl https://8.8.8.8` cannot connect. Which layers are healthy and which are
   suspect?
10. The CRC error counter is climbing on a NIC. Which layer's fault is this, and what symptom does it show as
    in the upper layer?
11. You send 200 bytes of data but the size measured on the wire is much bigger. Write why, and which headers
    were added.
12. A packet travels from Istanbul to Frankfurt through 12 hops. **How many times** did the destination IP
    change during the journey, and **how many times** did the destination MAC change?
13. A colleague says "the application is slow, let us look at the router's TCP settings". Which model error is
    in that sentence?
14. Write the single question you would ask someone who says "we are dropping packets", to narrow the search
    (hint: the PDU name).

## Part C — Reasoning and connection

15. Does a layer working prove that the layers above it work? What does it say about the ones below it? How
    does this rule determine the order of diagnosis?
16. When HTTPS is used, are the IP and TCP headers encrypted too? How does your answer explain how a firewall
    (Phase 9) can work at all?
17. In the cloud you cannot see L1/L2. Which class of faults does this remove from your life, and which
    diagnostic tool does it make your starting point?
18. Without layering, why would moving from IPv4 to IPv6 have been much harder? Explain in one sentence.

---

## Answer key

1. (i) **Design:** one layer can be changed without touching the others (Wi-Fi ↔ ethernet, IPv4 ↔ IPv6);
   (ii) **Diagnosis:** the fault is bounded to a single layer and the search space narrows (0.1.1). — 2. 7
   Application (work), 6 Presentation (format), 5 Session (session), 4 Transport (reliability), 3 Network
   (path), 2 Data link (neighbour), 1 Physical (signal) (0.1.2). — 3. Application = 7+6+5; Transport = 4;
   Internet = 3; Network Access = 2+1 (0.1.2). — 4. Every layer behaves as if it talks directly to the layer
   with the same number on the other side; it is an illusion because data never actually moves horizontally —
   it goes down at one end and up at the other (0.1.3). — 5. L4: **port** (which application); L3: **IP**
   (which machine); L2: **MAC** (which neighbour / next stop) (0.2.1). — 6. Every header carries a field
   stating which protocol is carried inside it (L2 "there is IP inside", L3's `proto` field "there is TCP
   inside", L4's destination port "which application") (0.2.2). — 7. Data = L7 (no address); segment = L4
   (port); packet = L3 (IP); frame = L2 (MAC) (0.3.1). — 8. A switch opens **one** header (L2/ethernet); a
   router **two** (it strips L2, looks at L3, attaches a new L2). Neither reads L4 (0.2.2).

9. Healthy: L1, L2, L3 (ICMP goes and returns). Suspect: **L4** (port closed / firewall blocking the port /
   TCP handshake not completing) and **L7** (TLS/application error) (0.1.3). — 10. An **L2 / frame** fault
   (cable, connector, NIC, switch port, duplex). Above it shows as a rise in **TCP retransmits** — the cause is
   L2, the effect L4 (0.3.1). — 11. Every layer adds its own header: TCP header (L4) + IP header (L3) +
   ethernet header and FCS (L2). That overhead is the fixed cost of every packet (0.2.1). — 12. The destination
   **IP never changed** (0 times — constant end to end); the destination **MAC changed at every hop** (12 times
   — the next stop at each step) (0.2.2). — 13. The router stops at L3 and **does not read** the TCP header;
   TCP is an end-to-end protocol and only the two ends interpret it. The place to look is not the router but
   the end machines (0.2.2). — 14. "Is it a **frame**, a **packet** or a **segment** that is dropping?" — that
   is, in which counter do you see the loss: NIC CRC/drop (L2), loss at a hop (L3), TCP retransmit (L4)? The
   answer determines where to look (0.3.1).

15. No — a layer working says nothing about the ones **above** it; but it does prove that the ones **below**
    it work. That is why diagnosis is done **bottom up**: going up before verifying the lower layer is guessing
    without evidence (0.1.3, 0.4). — 16. No. TLS encrypts the **payload**; the IP and TCP headers stay **in the
    clear**. That is exactly why a firewall can work: without ever seeing the content it reads the
    source/destination IP and port and decides (the 5-tuple in Phase 9) (0.2.1, Q3). — 17. The
    cable/connector/duplex/CRC class of **L2 physical faults** leaves your life; your starting diagnostic tool
    becomes a record at L3 level (VPC Flow Logs — packet level) (0.3.1). — 18. Because changing the IP layer
    would have required rewriting TCP above it and all applications too; thanks to layering only L3 changed
    while L4 and L7 stayed the same (0.1.1).

## Scoring

| Correct | What it means |
|---|---|
| 16-18 | The model has settled. You can now place any symptom in a layer. You are ready for Phase 1. |
| 13-15 | Good. Re-read the sections of the questions you missed (especially 0.2 encapsulation). |
| 9-12 | The basics are there but fragile. Redo 0.2 by **drawing** it with pen and paper — reading is not enough. |
| 0-8 | Walk the phase again. The single goal: being able to draw a packet's down-and-up journey without memorising. |

Question you missed → section to go back to:

| Question | Section |
|---|---|
| 1, 18 | 0.1.1 The layer idea |
| 2, 3 | 0.1.2 OSI and TCP/IP |
| 4, 9, 15 | 0.1.3 The peer-to-peer illusion |
| 5, 11, 16 | 0.2.1 Encapsulation |
| 6, 8, 12, 13 | 0.2.2 Decapsulation and hops |
| 7, 10, 14 | 0.3.1 PDUs |
| 17 | 0.3.1 Cloud connection |

---
---

# Phase 0 — Closing and Bridge to Phase 1

## What you carry from this phase

Phase 0 gave you not a command but a **point of view**: the network is not a single "internet" but **layers**
standing on top of each other, each responsible for a single job. Data is wrapped as it goes down through these
layers (encapsulation), unwrapped as it comes up (decapsulation), and takes a different name at each stage
(data → segment → packet → frame). Every layer's header points to the next; every layer behaves as if it talks
to its peer on the other side, but in reality data always moves vertically.

The two most lasting rules are these: (1) **A layer only works if the ones below it work** — which is why
diagnosis is done bottom up. (2) **Addresses have different lifetimes** — IP is constant end to end, MAC
changes at every hop. Do not memorise these two sentences, **use** them: all eleven phases that follow put
flesh on them.

## Where Phase 1 connects to this

Phase 0 said "data passes through layers" and showed that every layer carries an **address**: MAC at L2, IP at
L3, port at L4. But we never opened up those addresses **themselves**: what exactly is a MAC, how is an IP
written, why is it split into "private" and "public", where does a machine get its IP from?

Phase 1 does exactly that: **how we point at a machine on the network.** There you will see why MAC is only
meaningful on the local network (the reason behind Phase 0's "MAC changes at every hop"), why an IP consists of
two parts (network + host) — which is the seed of the subnet in Phase 2 — and why the port is the answer to the
question "which application". Phase 0 showed the envelopes; Phase 1 teaches you to read the **addresses** on
them.

> **🤔 Phase output — ask yourself:** In Phase 0 we said "MAC changes at every hop, IP stays constant end to
> end". Before moving to Phase 1, think: if MAC changes at every hop, **which MAC address** does your machine
> write when it sends a packet to a remote server? How can it find a MAC it does not know — and which new
> question does that open? (Hint: the answer is the name of a protocol and we will see it in Phase 3; but its
> question is born in Phase 1.)
>
> **🧪 Lab 0 idea (on your own machine, all 🟢):** (1) See your interfaces and MAC addresses with `ip link` —
> these are your L2 identity. (2) See your IP addresses with `ip addr` — these are your L3 identity. Notice
> that both sit on the same interface, on two separate lines: one interface, two layers. (3) Run
> `ping -c 3 8.8.8.8` and at the same time capture the packets in another terminal with
> `sudo tcpdump -n -c 3 icmp` — see that the envelopes we described in Phase 0 are really there. (4) On paper,
> draw the downward chain from the moment you type `example.com` into your browser to the packet leaving the
> wire; next to every box write the header that layer adds and the PDU name. These four steps make this phase's
> model concrete in your hands.

---

> **Navigation:** **Phase 0** · [Phase 1 — Addressing ▶](Phase_1_Addressing.md)
