# Networking — Offline Workbook

> **Cloud Engineer Fundamental Roadmap — Networking**
> This is the **standalone, AI-mentor-free, internet-free** workable version of the
> [`Cloud_network_roadmap_fundamentals_en.md`](../../../Cloud%20Learning%20With%20Mentor/English/Cloud_network_roadmap_fundamentals_en.md)
> roadmap.

> 🇹🇷 Türkçe sürüm: **[README.md](../Turkish_Network/README.md)**

---

## What is this book, and how does it differ from the main map?

The roadmap file in the mentor folder is a **skeleton**: you give it to an AI mentor and the
mentor opens the topics in order. Read on its own, the skeleton tells you "what you need to
learn" but it does not "teach it to you".

This book is the **flesh** on that skeleton. The same phases, the same sub-items, the same
depth labels — but here every item is:

- **explained** (intuition → mechanism → command output),
- **asked** (questions left for you to think about),
- **answered** (later in the same file),
- **connected** (to the previous item and to its counterpart in AWS).

Nowhere does it say "ask your mentor about this". You can work through the file from start
to finish on a plane, with no internet, entirely on your own.

---

## How this book differs from the other two — important

In the hardware book `[application]` meant "relate it to an architectural decision"; in the
Linux book it meant "run it on your own machine".

**Here `[application]` mostly means "run it on your own machine", and sometimes "work it out
on paper".**

Part of networking is invisible: you cannot watch on your own machine how a router decides
about a packet. But you **can** do subnet arithmetic by hand, you **can** look at the ARP
table, you **can** capture a TCP handshake with `tcpdump`. That is why this book has two
kinds of practice: **`🔧 See it on your machine` boxes** (command + expected output +
line-by-line commentary) and **paper calculations** (especially Phase 2).

> **What if you do not have a machine?** Nothing. This book can be worked through from start
> to finish without one — every output is written out and commented. But if you can find a
> Linux machine (physical, a VM, WSL2 or an EC2 t3.micro) your learning speed **doubles**. For
> Phase 10's finishing lab a machine is all but required.

---

## The files

| File | Phase | Topic | Approximate time |
|---|---|---|---|
| [Phase_0_Mental_Model.md](Phase_0_Mental_Model.md) | 0 | Layers, encapsulation, PDUs, OSI as a diagnostic order | 3–4 hours |
| [Phase_1_Addressing.md](Phase_1_Addressing.md) | 1 | MAC, IPv4, ports, bind addresses, private blocks, DHCP/DORA, APIPA | 5–7 hours |
| [Phase_2_Subnetting_and_CIDR.md](Phase_2_Subnetting_and_CIDR.md) | 2 | Masks, CIDR, usable addresses, VLSM, supernetting, overlaps | 6–8 hours |
| [**Checkpoint_Quiz_1.md**](Checkpoint_Quiz_1.md) | 0–2 | Combined quiz: the model + addresses + subnets | 1 hour |
| [Phase_3_Local_Network_L2.md](Phase_3_Local_Network_L2.md) | 3 | ARP, the switch MAC table, the broadcast domain, VLANs | 4–6 hours |
| [Phase_4_Routing_L3.md](Phase_4_Routing_L3.md) | 4 | The default gateway, the routing table, LPM, TTL, ICMP, BGP | 6–8 hours |
| [**Checkpoint_Quiz_2.md**](Checkpoint_Quiz_2.md) | 3–4 | Combined quiz: "where is the packet going" | 1 hour |
| [Phase_5_Transport_Layer.md](Phase_5_Transport_Layer.md) | 5 | TCP vs UDP, the handshake, the window, states, MTU/MSS | 6–8 hours |
| [Phase_6_DNS.md](Phase_6_DNS.md) | 6 | The hierarchy, the resolution chain, record types, TTL, caching | 5–6 hours |
| [Phase_7_NAT.md](Phase_7_NAT.md) | 7 | NAT/PAT, the public-private distinction, IPsec tunnels, IPv6 | 5–7 hours |
| [Phase_8_HTTP_and_TLS.md](Phase_8_HTTP_and_TLS.md) | 8 | HTTP anatomy, status codes, the TLS handshake, proxies, CDNs | 5–7 hours |
| [Phase_9_Firewalls.md](Phase_9_Firewalls.md) | 9 | Stateful vs stateless, the 5-tuple, DROP vs REJECT, SG vs NACL | 5–7 hours |
| [**Checkpoint_Quiz_3.md**](Checkpoint_Quiz_3.md) | 5–9 | Combined quiz: "why was the packet dropped?" | 1–1.5 hours |
| [Phase_10_Troubleshooting.md](Phase_10_Troubleshooting.md) | 10 | The layer-by-layer methodology, tool mastery, `tcpdump`, Flow Logs | 6–8 hours |
| [Phase_11_Bridge_to_the_Cloud.md](Phase_11_Bridge_to_the_Cloud.md) | 11 | VPC, route tables, IGW/NAT GW, Route 53, SG/NACL, ALB/NLB | 4–6 hours |

**Appendices:**

| File | What it is for |
|---|---|
| [Appendix_A_Command_Glossary.md](Appendix_A_Command_Glossary.md) | Every command, with its purpose, its risk mark (🟢🟡🔴) and the phase it appears in — keep it open on your desktop |
| [Appendix_B_Concept_File_Map.md](Appendix_B_Concept_File_Map.md) | Layers, special address blocks, the prefix table, ports, DNS records, HTTP/ICMP codes, the AWS mapping |
| [Appendix_C_When_It_Breaks_Quick_Reference.md](Appendix_C_When_It_Breaks_Quick_Reference.md) | Symptom → likely cause → verification command → phase; all the "When it breaks" tables merged onto one page |

**Figures:** There is a diagram for each phase's key mechanism
([`../diagrams/png/`](../diagrams/png)). The diagrams are **language-neutral** (the labels are
in English) — the Turkish and English versions use the same images.

---

## How to work through it

### 1. Do not break the order

Each phase speaks in the **words** of the one before it. When Phase 4 explains "longest
prefix match" it uses Phase 2's mask logic; when Phase 9 explains "a stateful firewall" it
uses Phase 5's TCP state. The whole of Phase 10 is built on the "When it breaks" notes of the
ten phases before it — read on its own it turns into a list of commands.

**The one exception:** Phase 8.3 (HTTP versions) is labelled `[skip]`; knowing its name is
enough.

### 2. Take the boxes seriously

There are five kinds of box in the text. Each has a different job:

> **🤔 Think 4.2** — **Do not answer the question here as soon as you read it.** Take a pen,
> think for 2 minutes, write your guess down. The answer is in the *"Answers to the Think
> questions"* section at the end of the phase. If your guess turns out to be wrong, that is a
> good thing — a wrong guess makes the right answer stick.

**❓ A question that comes to mind:** A question that arises naturally as you read. The answer
is **immediately below it**. These do not wait, because leaving them unanswered breaks the
flow of reading.

**⚠️ Common misconception:** Something most people believe wrongly. If, while reading one of
these, you say "that is what I thought too", mark that sentence.

**💡 Cloud connection:** The AWS counterpart of the mechanism you have just learned. Phase 11
is the sum of these boxes — so take a note of every one.

**🔧 See it on your machine:** A command + the **expected output** + a line-by-line reading of
that output. At the head of the box is one of three marks:

| Mark | What it means |
|---|---|
| 🟢 | **Read-only.** It does not change your system; run it with an easy mind. |
| 🟡 | **A temporary change.** It goes back to its old state when the machine restarts. |
| 🔴 | **A permanent change.** Do not do it on a production machine. Do not run it without understanding what it does. |

In this book a 🔴 command **always** comes with its undo step. In networking there is one
extra danger: **you can cut off your own access.** Read Phase 9.5.3 before changing a
firewall or a route on a remote machine.

### 3. Respect the depth labels

| Label | What is expected | How you test it |
|---|---|---|
| `[concept]` | Being able to explain it intuitively | "Could I explain this to a friend in 30 seconds?" |
| `[mechanism]` | Being able to explain it step by step | "Could I draw the flow with pen and paper?" |
| `[application]` | **Being able to run it and read the output / work it out by hand** | "Would I spot what is abnormal in this output?" |
| `[skip]` | Just knowing the name | Being able to say "that was in that area" when you hear it is enough |

### 4. Do not skip the end of a phase

Every phase ends with:

1. **When this phase breaks — the fault signatures** — a symptom → mechanism → first place to
   look table
2. **Answers to the Think questions** — compare them with your own guesses
3. **Frequently asked questions** — the "but what about…" questions around the phase
4. **Test yourself** — an 18-question test + a full answer key + scoring
5. **Closing and the bridge** — what remains from this phase, where the next one connects to it

If you score below 70% on the test, **instead of repeating the phase**, go back to the section
the question you got wrong points at. At the end of every phase there is a "which section to
go back to for a question you missed" table.

### 5. Do not skip the checkpoint quizzes — they are different

The three checkpoint quizzes (Phases 0–2, 3–4, 5–9) measure **something different** from the
phase tests. A phase test asks "did you understand this phase?"; a checkpoint quiz asks "**can
you connect these phases to each other?**" Most of the questions cannot be answered from a
single phase.

Real work is like that too: no fault stays inside a single phase.

---

## Progress tracking

```
[ ] Phase 0  — The Mental Model: Layers and Envelopes
[ ] Phase 1  — Addressing: Who Is Who
[ ] Phase 2  — Subnetting and CIDR                 ← the pen-and-paper phase, not to be skipped
[ ] ✅ Checkpoint Quiz 1 (Phases 0–2)
[ ] Phase 3  — The Local Network (L2): ARP and the Switch
[ ] Phase 4  — Routing (L3): Where Is the Packet Going
[ ] ✅ Checkpoint Quiz 2 (Phases 3–4)
[ ] Phase 5  — The Transport Layer: TCP and UDP
[ ] Phase 6  — DNS: From a Name to an Address
[ ] Phase 7  — NAT and the Real World
[ ] Phase 8  — The Application Layer: HTTP + TLS
[ ] Phase 9  — Firewalls and Filtering             ← "timeout or refused" is here
[ ] ✅ Checkpoint Quiz 3 (Phases 5–9)
[ ] Phase 10 — Troubleshooting                     ← the summit of the map
[ ] Phase 11 — The Bridge to the Cloud
```

---

## The north star — three instinct questions

This whole book exists so that you can answer these three questions **without thinking**:

| # | Question | Where it is answered |
|---|---|---|
| 1 | **How does data move?** | Phases 0, 1, 3, 4, 5 |
| 2 | **Why was the packet dropped?** | Phases 4, 9, 10 |
| 3 | **Why is the router misbehaving?** | Phases 4, 7, 10 |

These three questions are almost the whole of a cloud engineer's work on the network side.
Phase 10.3 collects the decision trees of all three in one place.

**A design principle:** troubleshooting was not left to the end. Every phase has a *"what does
it look like when this knowledge breaks?"* angle. That is the only way instinct forms — Phase
10 turns those scattered notes into a single reflex.

---

## This book's goal and its limits

**The goal:** when an "I cannot connect" complaint arrives, to be able to narrow down what
happened at which layer with a **methodology** instead of a guess; to be able to see which
fundamental concept stands under every box in a VPC design.

**The limit:** this book does not produce network engineers (CCNA/CCNP). Switch/router
configuration syntax, the internals of OSPF/EIGRP, MPLS and the carrier side are out of scope.
The aim is for a cloud engineer to see deeply enough to **justify decisions and hunt faults.**

**Environment assumptions:** the command examples are on **Linux** (Ubuntu 22.04/24.04); the
cloud examples use **AWS** terminology. The Azure/GCP counterparts are other names for the
same concepts — the mapping table in Phase 11.8 lets you make that translation.

---

## Its relationship with the main map

| This book | The main map |
|---|---|
| Offline, worked through alone | Given to an AI mentor |
| Explanation + question + answer | A topic list + mentor instructions |
| Fixed content | The mentor calibrates to the student's level |
| Test-yourself quizzes | The mentor examines you orally |

The two are **not rivals**. Using them in sequence is recommended: work through a phase with
this book first, then give the main map to an AI mentor and repeat the same phase **orally**.

---

## Its relationship with the other books

This series consists of three books and the three of them **bound each other sensibly**:

| Book | What it takes | Where it meets this book |
|---|---|---|
| [Hardware](../../Cloud_hardware.roadmap/English_Hardware/README_en.md) | The physical layer: CPU, memory, disk, NIC, hypervisor | What is **underneath** the NIC, bandwidth and latency: Phase 5's "why is it slow" question |
| [Linux](../../Cloud_linux.roadmap/English_Linux/README_en.md) | The operating system: processes, memory, boot, permissions, tools | Linux Phase 7 takes the **OS side** of networking (`ip`, `ss`, SSH); the protocol itself is here |
| **Networking** (this book) | Protocol theory: OSI, addressing, routing, TCP, DNS, TLS, firewalls | — |

This book **stands independently** of the other two. If you have not read the hardware or the
Linux book you will not get stuck anywhere — where it is needed the relevant concept is
briefly explained here, and if you want to go deeper the relevant book is pointed at.

---

*This offline version is based on Denis Ergöçmen's Cloud Engineer fundamental roadmap series.*
