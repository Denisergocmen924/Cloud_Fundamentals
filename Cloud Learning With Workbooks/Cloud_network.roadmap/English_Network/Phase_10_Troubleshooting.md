# Phase 10 — Troubleshooting: Layer-by-Layer Diagnosis

> **Navigation:** [◀ Checkpoint Quiz 3](Checkpoint_Quiz_3.md) · **Phase 10** · [Phase 11 — The Bridge to the Cloud ▶](Phase_11_Bridge_to_the_Cloud.md)

---

## Where we come from

At the end of Phase 9 I asked: a user says "the site will not open", you have seven tools in your hand and
running them all takes 10 minutes — **which one should you start with?**

The hint was: the right tool is not the one that **gives the most information** but the one that **eliminates
the most possibilities**. This whole phase is the unfolding of that sentence.

Through ten phases so far you have seen a **"When this phase breaks"** table at the end of every one. Dozens
of symptoms, dozens of causes. This phase will not make you memorise them — it will **put them in order.**

What you bring with you: everything, really. This phase has almost no new knowledge; the only new thing is
**discipline**.

## The question of this phase

> *"Something is not working. I have thirty possibilities. In which order do I look so that I reach the right
> place in the fewest steps?"*

What an inexperienced engineer does is **guess**: "it could be DNS", they say, and run `dig`; "maybe it is the
firewall", and look at the rules; "maybe it is the application", and dive into the logs. They jump around at
random, are sometimes lucky, and most of the time lose hours.

What an experienced engineer does is **eliminate**: they start at the bottom, at each step delete half the
possibilities with certainty, and in a few steps have **proved** which layer the fault is in.

The difference is not intelligence, it is **order.** This phase gives you that order.

---

## By the end of this phase

- You will be able to apply the bottom-up, layer-by-layer diagnostic methodology in order
- You will be able to explain which possibilities each step eliminates and **why it is in that place**
- You will know which layer each tool looks at and be able to pick the right tool for the right question
- You will know why `tcpdump` is "the final referee" and when to turn to it
- You will be able to follow the decision trees of the three instinctive questions ("How does data move?",
  "Why was the packet dropped?", "Why is the router misbehaving?")
- **Cloud:** You will be able to read VPC Flow Logs lines and **prove, instead of guessing**, where and why a
  packet was dropped
- You will be able to diagnose a deliberately broken system in minutes by following the methodology

---

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 10.1 | The layer-by-layer methodology | `[application]` | **The heart of the phase** — the order itself |
| 10.2 | Tool mastery | `[application]` | Which tool looks at which layer |
| 10.3 | The three instinct questions | `[application]` | Decision trees |
| 10.4 | Cloud: VPC Flow Logs | `[application]` | Reading instead of guessing |
| 10.5 | When this phase breaks | — | When the diagnosis itself breaks |

> **How to work through this phase:** This phase is not learned by reading — it is learned by **doing**. All
> the commands are 🟢 (they change nothing, they only read), so run them without fear. But the real gain comes
> from following the flows at the end of each section **on your own machine**. A suggestion: keep a terminal
> open while reading this phase and run every command as you pass it, even when nothing is broken. Because
> **if you do not know what healthy output looks like, you cannot recognise broken output.** And the finishing
> lab at the end of the phase may be the most valuable 30 minutes in this book: you will set up a deliberate
> fault on your own machine and find it with the methodology.

---
---

# 10.1 The Layer-by-Layer Methodology

## 10.1.1 Why bottom-up `[concept]`

When you learned OSI in Phase 0 I said this: the layers are **a diagnostic order**. Now we collect on that
promise.

The rule is:

> **Climb from the bottom up. If a lower layer is broken, every test at an upper layer lies to you.**

Why? Because the dependency goes one way. DNS needs IP in order to work; IP needs the link in order to work.
But the link does not need DNS at all.

The practical consequence: **a fault coming from a lower layer appears in a completely different disguise at
an upper layer.** If the cable is not plugged in, `dig` says "cannot reach the DNS server" — and you spend an
hour fiddling with DNS configuration. But the problem was not DNS; DNS was only the side **reporting the
lower layer's fault**.

Going top-down is therefore the most common cause of wasted time: the topmost symptom hides the bottommost
cause.

## 10.1.2 The six steps `[application]`

Here is the methodology. Each step asks a question, is answered with one command, and when it passes it
**eliminates a group of possibilities with certainty.**

| # | Layer | Question | Command | Eliminated if it passes |
|---|---|---|---|---|
| 1 | **Link** (L2) | Is the interface up, is there an IP? | `ip addr` | Cable, interface, DHCP (1.6) |
| 2 | **IP** (L3) | Can I reach my own network? | `ping <gateway>` | ARP, switch, VLAN (Phase 3) |
| 3 | **Route** (L3) | Do I have a path out? | `ip route` + `ping 1.1.1.1` | A missing route, the gateway (Phase 4) |
| 4 | **DNS** (L7) | Does the name resolve? | `dig <name>` | The resolution chain (Phase 6) |
| 5 | **Port** (L4) | Is the target port open? | `nc -zv <host> <port>` | Firewall, service (Phases 5, 9) |
| 6 | **Application** (L7) | Does the service answer correctly? | `curl -v` | The HTTP/TLS layer (Phase 8) |

And the most critical point is the **order** of the steps:

- **Why is step 3's `ping 1.1.1.1` an IP and not a name?** Because if you use a name you drag DNS into the
  test as well and cannot tell which of the two is broken. Pinging an IP tests L3 **in isolation from DNS**.
  This is the methodology's most elegant move.
- **Step 3 must pass before step 4.** DNS is a network service; if L3 is not working a DNS test is
  meaningless.
- **If step 5 passes and 6 fails**, the problem is definitely in the application — the network side is proved.

There is also a two-step shortcut, and in practice it is the one used most:

```
ping 1.1.1.1      →  does it work?   (is L3 sound)
ping google.com   →  does it work?   (is DNS sound)
```

If the first works and the second does not, it is **definitely DNS**. If neither works, do not look at DNS at
all — the problem is further down. Those two commands split thirty possibilities in two in three seconds.

![Figure 10.1 — The bottom-up diagnostic ladder: each rung asks a question, is answered with one command, and when it is passed it eliminates a group of possibilities; when a rung fails the fault is in that layer and there is no point testing the rungs above it.](../diagrams/png/nw-10-01-layered-debugging.png)

On the right-hand side of the ladder in the figure, the groups of possibilities eliminated by passing each
rung are written out. When you get stuck on a rung there is only one thing to do: **stop looking upwards.**

> **🔧 See it on your machine** 🟢 — run the six steps in order
>
> ```
> $ ip addr show | grep -E "^[0-9]|inet "
> 2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
>     inet 192.168.1.42/24 brd 192.168.1.255 scope global enp3s0
>         └── UP + an IP exists → step 1 ✓
>
> $ ip route | head -2
> default via 192.168.1.1 dev enp3s0 proto dhcp metric 100
> 192.168.1.0/24 dev enp3s0 proto kernel scope link src 192.168.1.42
>         └── a default route exists → half of step 3 ✓
>
> $ ping -c1 192.168.1.1 >/dev/null && echo "L2/gateway OK"
> L2/gateway OK
>
> $ ping -c1 1.1.1.1 >/dev/null && echo "L3 OK (without DNS)"
> L3 OK (without DNS)
>
> $ dig +short google.com | head -1
> 142.250.185.78
>         └── DNS ✓
>
> $ nc -zv google.com 443
> Connection to google.com 443 port [tcp/https] succeeded!
> ```
>
> These six commands tell you in **30 seconds** which layer an "it does not work" complaint is in. All of them
> are 🟢 — they change nothing. See the healthy output now so that you can recognise the broken one.

> **⚠️ Common misconception: "Let me look at the most likely cause first, I will save time."**
>
> The opposite happens. "The most likely cause" is a **guess**, and when it turns out to be wrong you have
> learned nothing — you do not know what you eliminated and you check the same possibility a second time. The
> methodology, on the other hand, produces **evidence** at every step: if step 3 passed, you never think about
> the whole of L1–L3 again. A disciplined three-step order is almost always faster than a lucky ten-step
> guess. The only legitimate case for a shortcut is having **definite evidence from the symptom** (for
> example you got "connection refused" — that already proves L1–L4 were passed, 9.2.3).

> **🤔 Think 10.1** — On a server `ping 1.1.1.1` works but `ping google.com` gives "Temporary failure in name
> resolution". (a) Which layers are definitely sound? (b) How many possibilities have you eliminated? (c) What
> is your next command?
>
> *(Answer: at the end of the phase)*

---
---

# 10.2 Tool Mastery

Every tool looks at **one layer**. Mastery is not in knowing the tool but in knowing **which tool answers
which question**.

## 10.2.1 The tool–layer map `[application]`

| Tool | Layer | The question it answers | Related phase |
|---|---|---|---|
| `ip addr` | L2/L3 | Is my interface up, do I have an address? | 1.2, 1.6 |
| `ip route` | L3 | By which path do I get to this destination? | 4.2, 4.3 |
| `ip neigh` | L2 | Do I know my neighbour's MAC? | 3.1 |
| `ping` | L3 | Does a packet go to this address and come back? | 4.4 |
| `mtr` / `traceroute` | L3 | **Where** on the path is it getting lost? | 4.5 |
| `dig` | L7 (DNS) | Which IP does this name resolve to, and who says so? | 6.2 |
| `ss` | L4 | Who is listening, which connections are open? | 1.4, 5.3 |
| `nc -zv` | L4 | Is this port reachable? | 9.2.3 |
| `curl -v` | L7 | Where does the request/response chain break? | 8.1 |
| `tcpdump` | **all of them** | **What actually happened?** | — |

The last row is special and is the subject of the next section.

## 10.2.2 `tcpdump`: the final referee `[application]`

Every other tool gives you an **interpretation**: `ping` says "unreachable", `curl` says "timeout", `dig` says
"SERVFAIL". Those are **the operating system's or the tool's own decision**.

`tcpdump` does not interpret. It shows you **what actually passed over the wire**.

So adopt this rule:

> **If two sides are blaming each other, the referee is `tcpdump`.**

The most classic use: one team saying "the request never reached us" and another saying "we sent it". You run
`tcpdump` on the receiving side and the argument is over.

It answers three questions:

1. **Did the packet arrive?** If there are no lines at all the packet did not reach it — the problem is on the
   way (route, firewall, NAT).
2. **Did the answer go out?** If there is an inbound one but no outbound one, the problem is on this machine
   (the service or a local firewall).
3. **What kind of answer went out?** An RST in response to the SYN, or nothing at all? This **proves** the
   distinction in 9.2.3.

> **🔧 See it on your machine** 🟢 — follow a handshake with `tcpdump`
>
> ```
> $ sudo tcpdump -ni any 'tcp port 443 and host example.com' -c 6
> 10:14:22.104 IP 192.168.1.42.51234 > 93.184.216.34.443: Flags [S], seq 1829...
> 10:14:22.141 IP 93.184.216.34.443 > 192.168.1.42.51234: Flags [S.], seq 4471..., ack 1830
> 10:14:22.141 IP 192.168.1.42.51234 > 93.184.216.34.443: Flags [.], ack 4472
>                                                          └── S, S., . = SYN, SYN-ACK, ACK (5.2.1)
> ```
>
> Reading the flags: `[S]` = SYN, `[S.]` = SYN-ACK, `[.]` = ACK, `[P.]` = data (PSH+ACK), `[F.]` = FIN,
> **`[R]` = RST**. Here you see live the three-way handshake you drew in 5.2.1.
>
> The diagnostic reading is very simple: if **only `[S]` lines** appear over and over and no `[S.]` ever comes
> → the packet goes out and no answer comes back → **a firewall DROP** or a routing problem (9.2.3). If an
> **`[R]`** comes back in response to the `[S]` → the machine is reachable but **the service is not
> listening** (5.2.2). If there are no lines at all → the packet is not leaving this interface (the wrong
> interface, a local route, or your filter is wrong).
>
> Useful flags: `-n` (no name resolution, does not slow things down), `-i any` (all interfaces), `-c N` (stop
> after N packets), `-w file.pcap` (save it, then open it with Wireshark).

## 10.2.3 `mtr`: where on the path `[application]`

`ping` says "it does not go and come back" but does not say **where** it is getting lost. `mtr` is the
continuously running version of `traceroute` (4.5.1) and gives a **loss percentage** for each hop.

```
$ mtr -rwc 20 example.com
HOST                        Loss%  Snt   Avg  Best  Wrst
 1. 192.168.1.1              0.0%   20   1.2   0.9   2.1
 2. 10.20.0.1                0.0%   20  12.4  11.8  18.2
 3. isp-core.example.net     0.0%   20  14.1  13.2  22.0
 4. peer-x.example.net      35.0%   20  88.3  14.9 340.1   ← here
 5. 93.184.216.34            0.0%   20  15.2  14.7  19.8
```

The reading rule — and this is very often misunderstood:

> **If loss shows on a middle hop while the last hop is clean, the loss is not real.**

The reason: intermediate routers treat producing an ICMP answer as a **low-priority** job and skip it when
they are busy (4.5.2). But they carry on **forwarding** the traffic perfectly. So the only meaningful line is
the **last one**: if there is loss there, there is a real problem. Loss in the middle is meaningful if it
continues on the hops after it as well.

> **🤔 Think 10.2** — A client says "I am sending requests to the server and no answer comes". The server team
> says "nothing is reaching us". (a) Where do you set up the referee, and why? (b) If you see only `[S]` lines
> on the server, what is the diagnosis? (c) If you see no lines at all, what is the diagnosis?
>
> *(Answer: at the end of the phase)*

---
---

# 10.3 The Answer Map for the Three Instinct Questions

We have been carrying three questions since the beginning of this book. Now we draw each one's decision tree.

## 10.3.1 "How does data move?" `[application]`

This is not a failure question — it is a **model verification** question. The answer is the chain the packet
follows:

```
Name (Phase 6: DNS)  →  IP address (Phase 1)
   ↓
Is the destination on my network? (Phase 2: compare with the mask)
   ├── Yes → find the MAC with ARP (3.1) → switch → destination
   └── No  → find the gateway's MAC (4.1) → router
          ↓
       Every router: the next hop by LPM (4.3), TTL−1 (4.4)
          ↓
Target machine → to the application by port (1.4) → the TCP handshake (5.2)
          ↓
TLS (8.4) → the HTTP request (8.1) → the answer
```

When you meet a failure you run this chain through your mind and see **which link has not been tested**. That
is really what the methodology (10.1.2) does: verify the chain from start to finish, link by link.

## 10.3.2 "Why was the packet dropped?" `[application]`

This is the question you will ask most often in the cloud. The decision tree:

```
What is the symptom?
├── "Connection refused" (RST)
│      → The packet REACHED the target. The network is sound. The service is not listening. (9.2.3)
│        Check: ss -tulpn (on the target), is the listening address 127.0.0.1? (1.4.2)
│
├── "No route to host" / "Network unreachable"
│      → There is no path in the local route table. (4.2)
│        Check: ip route get <destination>
│
├── "Name or service not known"
│      → DNS. You never got to L3. (6.2)
│        Check: dig <name>, resolv.conf
│
├── TIMEOUT (no answer at all)
│      → The packet was dropped silently. Three possibilities:
│        ├── A firewall DROP (SG / NACL / OS)     → 9.2.3, the list in 9.4.2
│        ├── A missing route (outbound OR RETURN) → 4.2 — do not forget the return path
│        └── The host is not up                   → 4.4
│        Referee: tcpdump (on the target) → is the packet arriving? (10.2.2)
│
└── The connection is made but it HANGS / half the data
       → An MTU black hole. (5.7.3, 9.3.1)
         Check: ping -M do -s 1472 <destination>
```

The soundest reflex is the distinction at the top: **refused or timeout?** (9.2.3). Refused **proves** the
network works; a timeout proves nothing and takes you to the list.

## 10.3.3 "Why is the router misbehaving?" `[application]`

Routing failures are the most confusing ones, because they are usually **asymmetric**: the outbound direction
works, the return does not — and the symptom looks like "nothing works at all".

```
Does only one direction work?
├── Yes → The RETURN ROUTE is missing. (4.2)
│          On the target machine: "how is it going to come back to me?" — ip route
│          In the cloud: the far VPC/subnet's route table and the NACL outbound (9.4.1)
│
├── It goes to the wrong place → LPM. (4.3)
│          There is more than one matching route; the most SPECIFIC one wins.
│          Check: ip route get <destination>  — it tells you which row was chosen
│
├── It loops (TTL exceeded) → A routing loop. (4.4)
│          Two routers point at each other. Repeating hops in traceroute.
│
└── Some destinations work, some do not
           → A CIDR overlap or a missing specific route. (2.5, 4.3)
             The classic in the cloud: overlapping CIDR blocks in VPC peering (11.7)
```

A one-sentence summary: **testing the outbound direction is half a test; the one really forgotten is the
return path.**

> **💡 Cloud connection — asymmetric routing:** In the cloud "there is an outbound but no return" is very
> common and it has two typical causes. First: **the private subnet's route table has no `0.0.0.0/0 → NAT GW`
> row** (7.3.1) — the request looks as though it went out but it never did. Second: **the NACL's outbound rule
> is missing** (9.4.1) — the request gets in, the answer cannot get out, and the symptom is "the connection is
> made but no answer comes". In both cases the only thing visible on the client side is a **timeout**, so you
> cannot tell from the symptom which direction is broken. The only thing that solves this is **Flow Logs**, in
> the next section: there you see the traffic as **separate lines in each direction** and read directly which
> direction got the REJECT.

> **🤔 Think 10.3** — Two VPCs are connected with peering. You `ping` from a machine in A to a machine in B:
> timeout. You ping from B to A: **it works**. (a) What does this asymmetry tell you? (b) Where do you look?
> (c) Why does the symptom "ping from A does not work" not mean the problem is only A's?
>
> *(Answer: at the end of the phase)*

---
---

# 10.4 Cloud: Hunting Packets with VPC Flow Logs

## 10.4.1 Why it is needed `[concept]`

There is a problem in the cloud: **there is nowhere to run `tcpdump`.**

You cannot get into the router, you cannot get into the NAT Gateway, you cannot get into the NACL at the
subnet boundary. You can only capture packets on your own instance — but if the fault is **between** two
instances, you have no eyes there.

**VPC Flow Logs** fills that gap: it records the traffic passing through the network interfaces in the VPC and
for each flow writes whether it was **accepted or rejected**.

So it solves the "silent DROP" problem from Phase 9 (9.2.3). The firewall does not tell you, but **the log
does.**

## 10.4.2 Reading a line `[application]`

In the default format a line looks like this:

```
2 111122223333 eni-0abc123 10.0.1.50 10.0.2.20 51234 5432 6 12 2048 1699... 1699... ACCEPT OK
│ │            │           │         │         │     │    │ │  │    │       │      │
│ │            │           source IP dest IP   src   dst  │ pkt byte start   end    └── ACCEPT/REJECT
│ │            └── interface                   port  port └── protocol (6=TCP, 17=UDP, 1=ICMP)
│ └── account
└── version
```

Does it look familiar? The five fields in the middle are exactly the **5-tuple** (9.2.1). In Phase 9 you
learned which fields rules look at; Flow Logs uses the same fields **to tell you what happened**.

While reading it you look at three things:

1. **Is there a REJECT line?** If there is, the packet hit a rule — the 5-tuple tells you which rule is
   missing.
2. **Are there no lines at all?** The packet **never reached that interface** — the problem is earlier: the
   route table, the wrong destination, or an obstacle on the source side.
3. **Are there lines in only one direction?** The classic asymmetry (10.3.3): the outbound is ACCEPT, the
   return is missing or REJECT.

And one critical subtlety: **because a Security Group is stateful (9.4.1), the return traffic of a connection
the SG allowed appears in Flow Logs as a separate ACCEPT line** — because Flow Logs records flows, not rules.
But when a NACL refuses a packet you see a **REJECT** line. So in practice: *if you saw a REJECT there is most
likely an incoming connection that a NACL or an SG did not allow.*

## 10.4.3 The diagnostic flow `[application]`

The full flow for "I cannot connect" in the cloud — it combines the list in 9.4.2 with Flow Logs:

| What you see in Flow Logs | What it means | Where to look |
|---|---|---|
| ACCEPT at the source, **no line** at the target | The packet was lost on the way | **The route table** (4.2, 7.3.1), peering |
| **REJECT** at the target (inbound) | The target's SG/NACL blocked it | The SG inbound rule, the NACL inbound (9.4.1) |
| ACCEPT at the target, **REJECT** on the return | The return was blocked | **The NACL outbound ephemeral** (9.4.1) |
| ACCEPT in both directions but the application ✗ | The network is sound | **The OS firewall**, is the service listening (9.4.2) |
| No lines anywhere | The traffic was never produced | DNS may have returned the wrong IP (6.2) |

The last row is especially sneaky: if the application is not going to the right address at all, there is
nothing to see at the network layer. So before looking at Flow Logs, verify **which IP it is going to**
(`dig`, `curl -v`).

> **💡 Cloud connection — the limits of Flow Logs:** Flow Logs records **headers**, not **content**. So it
> answers the question "did the packet get through" and not "what was inside it" — whether the TLS handshake
> failed, whether a 502 came back, you cannot see those. It is also not real time: the aggregation interval is
> typically **1 or 10 minutes**, so you may not see the packet from a moment ago straight away. When you need
> content and really have to look at the packet level, the tool is **VPC Traffic Mirroring** (it sends a copy
> of the traffic to an analysis instance) — but it is expensive and heavy, not a day-to-day diagnostic tool.
> In practice the order is: Flow Logs first (did the packet get through), then `tcpdump` inside the instance
> (what was said), and mirroring last (if nothing else worked).

---
---

# 10.5 When This Phase Breaks — When the Diagnosis Itself Breaks

This phase teaches a habit, not a mechanism. So "breaking" is different here too: the fault is not in the
system but in **your approach.** The table below lists the classic mistakes made during diagnosis and how each
one is caught.

| Symptom | Likely cause | Verify | Related section |
|---|---|---|---|
| You have been wrestling with DNS for hours and it is not clearing | **You started at the top** — a lower layer is broken | `ping 1.1.1.1` | 10.1.1 |
| You are checking the same thing a second time | No methodology, only guesses | Write the steps down and tick them off | 10.1.2 |
| Two teams are blaming each other | There is no evidence, only interpretation | **`tcpdump`** (on the receiving side) | 10.2.2 |
| 30% loss in the middle of `mtr`, panic | ICMP rate-limiting — **the loss is fake** | Look at the last hop | 10.2.3 |
| You said "ping works, the network is sound", and it was not | Ping is not a permission test | `nc -zv host port` | 9.2.3 |
| You tested the outbound direction and found no problem | You forgot **the return path** | `ip route` on the target / Flow Logs | 10.3.3 |
| No lines at all in Flow Logs | The packet never reached that interface | The route table, the target IP with `dig` | 10.4.3 |
| Flow Logs is clean but the application does not work | The network is sound — the problem is above | The OS firewall, `ss -tulpn` | 10.4.3 |
| You cannot see the request from a moment ago in Flow Logs | The aggregation interval (1–10 min) | Wait a few minutes | the box in 10.4.3 |
| `curl` works, the browser does not | A different resolver / proxy / cache | `dig` + the browser's network tab | 6.4.2 |
| It works on one machine, not on another | Find the difference: SG, route, OS firewall | Run the same 6 steps on both | 10.1.2 |
| You broke the system while diagnosing it | You made the changes all at once | **One** change at a time | 10.5 |

> **The lesson from this table:** Diagnosis has three golden rules. First: **bottom-up** — no test at an upper
> layer is trustworthy until the lower layer is proved (10.1.1), and starting at the top is the most common
> cause of wasted time. Second: **one change at a time.** If you change three things at once and the problem
> clears, you cannot know which one cleared it — and worse, you may have created a new fault; if you do not
> know what each change did, you are not fixing, you are trying things at random. Third: **do not settle for
> an interpretation, look for evidence.** `ping` says "unreachable" but does not say **why**; `tcpdump` and
> Flow Logs show what actually happened (10.2.2, 10.4.2). And one last reflex: while diagnosing, **write down
> what you have eliminated.** Noting "L1–L3 ✓, DNS ✓, port ✗" both stops you repeating a step and saves half
> an hour for whoever you hand the problem over to.

---
---

# Phase 10 — Answers to the Think questions

## Answer 10.1 — The eliminating power of two commands

**(a)** **Everything from L1 to L3 is sound:** the interface is up, there is an IP address, ARP works, the
gateway is reachable, there is a default route, and going out to the internet works. Because for `ping
1.1.1.1` to work **all** of those have to be right — a single command verifies an entire stack (10.1.2).

**(b)** You eliminated **almost all of the "when it breaks" tables in this book that belong to Phases 1, 2, 3
and 4**: the cable, the interface, DHCP, a wrong mask, ARP, the switch, VLANs, the gateway, a missing route.
One layer is left: **DNS** (Phase 6) and everything above it.

**(c)** The next command is **`dig google.com`** — and in its output you look at two things: the **`SERVER:`**
line (which resolver you asked) and the **status** (`NOERROR` / `SERVFAIL` / `NXDOMAIN`). If `dig @1.1.1.1
google.com` works and `dig google.com` does not, the diagnosis is complete: **your configured resolver is
broken or unreachable** — not DNS itself, but which server you are asking (6.2.3).

**Related section:** 10.1.2 · **Next:** Phase 6.2.3 (the resolution order), 10.3.2 (the decision tree)

## Answer 10.2 — Setting up the referee in the right place

**(a)** You set the referee up on the **receiving side** (the server): `sudo tcpdump -ni any 'tcp port 443 and
host <client-ip>'`. The reason: the subject of the argument is the question "did the packet arrive", and
**only the receiver** can prove that. Capturing packets on the sender only confirms the "I sent it" claim —
which is already the unresolved half of the argument (10.2.2).

**(b)** If you see only `[S]` lines: **the packet IS REACHING the server** but the server is not answering. So
the claim "nothing is reaching us" is wrong and the problem is on this machine. Two possibilities: **a local
firewall is dropping it** (9.2.3) or **the service is not listening on that port** — you tell the second apart
instantly with `ss -tulpn`. (If the service were not listening the kernel would usually return an RST, so you
would see an `[R]`; no answer at all points at the firewall.)

**(c)** If you see no lines at all: the packet **is not reaching the server at all.** The problem is in
between — a missing route (4.2), an SG/NACL on the cloud side (9.4.1), a wrong destination IP (DNS, 6.2), or
the client is not sending anything. The next step: run `tcpdump` on the client side too — is the packet really
going out? In the cloud, look at **Flow Logs** (10.4.3): if there is a line you check the REJECT, if there is
no line you check the route.

**Related section:** 10.2.2 · **Next:** 10.4.3 (the cloud equivalent), 9.2.3 (DROP vs REJECT)

## Answer 10.3 — Asymmetry is a clue

**(a)** The asymmetry tells you **something very valuable**: **the foundation of L1–L3 is sound.** If a ping
from B to A works, there is a physical/logical path between the two VPCs, the peering is up and both sides'
addresses are right. The problem is **direction-related** — there is a rule or a route missing in one
direction (10.3.3).

**(b)** In three places, in this order: (i) **A's route table** — is there a row going to B's CIDR block, and
is the target the peering connection (4.2)? (ii) **B's NACL and SG** — do they allow the ICMP coming from A;
remember, ICMP requires a separate rule (the box in 9.3.1). (iii) **A's NACL outbound rule** (9.4.1). Also:
remember that ping is not a permission test — test the real application port too (the box in 9.2.3).

**(c)** Because for a ping to succeed the packet has to **both go and come back**. The symptom "it does not
work from A" can come from A's outgoing packet not reaching B, or from B's answer not being able to get back
to A — the symptom does **not distinguish** the two. A working ping from B proves the B→A direction but says
nothing about the A→B direction (ICMP echo and reply can hit different rules in different directions). The
tool that makes the distinction is Flow Logs: you see both directions as **separate lines** and read which one
got the REJECT (10.4.3).

**Related section:** 10.3.3 · **Next:** 10.4.3 (Flow Logs), Phase 11.7 (peering)

---
---

# Phase 10 — Frequently asked questions

**Q1 — Do I have to apply the methodology from the beginning every time?** No — if the symptom gives you
evidence you can start from there. If you got "connection refused", L1–L4 is already proved (9.2.3) and you go
straight to the service. But **if you have no evidence at all** (just "it does not work"), start from the
beginning; trying a shortcut is another name for going back to guessing (the box in 10.1.2).

**Q2 — `traceroute` or `mtr`?** `traceroute` is a one-off photograph; `mtr` runs continuously and gives a
**loss percentage**. For catching intermittent problems `mtr` is far better (`mtr -rwc 50`). But both carry
the same trap: loss in the middle is usually fake (10.2.3).

**Q3 — Is `tcpdump` safe on a production server?** It is a read operation and does not change the traffic —
but on a busy server it can create CPU load. Always use **a narrow filter** and `-c N`: `tcpdump -ni eth0
'host X and port Y' -c 100`. An unfiltered `tcpdump` on a busy machine means serious load.

**Q4 — The problem is intermittent and I cannot catch it. What do I do?** Three ways: (i) run `mtr` for a long
time (`-c 500`), (ii) record with `tcpdump -w file.pcap` and look at the timestamp when the problem happens,
(iii) in the cloud, query Flow Logs — it is the only source that reaches into the past (10.4.2). With
intermittent problems, **accumulating evidence** is more effective than looking at the moment.

**Q5 — What do I do when "it works on one machine and not on another"?** The fastest way is to **find the
difference**: run the same six steps (10.1.2) on both machines and catch the first step that diverges. Most of
the time the difference is in the SG, the route table or the OS firewall — or it is the service's listening
address (1.4.2).

**Q6 — I see a REJECT in Flow Logs but I do not know which rule blocked it.** Flow Logs does not tell you the
rule, it only gives the 5-tuple and the result. You match the rule yourself: if an incoming connection is
REJECTed, look at the target's SG inbound rule and the NACL inbound rule; if the return is REJECTed, look at
the **NACL outbound ephemeral** rule (9.4.1). A practical hint: the **destination port** in the REJECT line
tells you directly which rule to look for.

**Q7 — When can I say "this is not a network problem"?** When these three hold: (i) `nc -zv host port`
succeeds (everything up to L4 works), (ii) `curl -v` makes a request and gets an answer (the L7 chain is
established), (iii) there are ACCEPTs in both directions in Flow Logs. From that point on the problem is in
the application and the right place is the application logs (10.4.3).

---
---

# Phase 10 — Test yourself

Write your answers on paper, then compare them with the key. Target: 14+ out of 18.

## Part A — Methodology

1. Why do you climb from the bottom up in diagnosis? Justify it in one sentence.
2. Write the six steps in order and add each one's command.
3. Why is an IP pinged in step 3 rather than a name?
4. What does `ping 1.1.1.1` ✓ and `ping google.com` ✗ mean?
5. If both are ✗, what is your next step?
6. Why is the "look at the most likely cause" approach slow?

## Part B — Tools and diagnosis

7. Write the layer each of these tools looks at: `ip neigh`, `ss`, `dig`, `mtr`, `curl -v`.
8. Why is `tcpdump` "the final referee"? Which three questions does it answer?
9. There are `[S]` lines in the `tcpdump` output, no `[S.]`. Diagnosis?
10. In the `tcpdump` output an `[R]` comes back in response to the `[S]`. Diagnosis?
11. In `mtr` there is 35% loss at hop 4 and 0% at the last hop. Is there a problem — why?
12. Two teams are blaming each other. What is your first move, and where?

## Part C — Cloud and reasoning

13. In a Flow Logs line, which five fields make up the 5-tuple?
14. In Flow Logs there is **no line at all** on the target interface. What does that mean?
15. The outbound is ACCEPT, the return is REJECT. Which rule is missing?
16. Write two limits of Flow Logs.
17. A→B ping does not work, B→A does. Which possibilities does this asymmetry eliminate?
18. What is the justification for the "one change at a time" rule?

---

## Answer key

1. Because the dependency goes one way: an upper layer needs the lower one, not the reverse — if a lower layer
   is broken, every test at an upper layer gives a **misleading** result (10.1.1). — 2. (1) Link `ip addr`, (2)
   IP/gateway `ping <gateway>`, (3) Route `ip route` + `ping 1.1.1.1`, (4) DNS `dig <name>`, (5) Port `nc -zv
   host port`, (6) Application `curl -v` (10.1.2). — 3. Because using a name **drags DNS into the test** as
   well and you cannot tell which of the two is broken; pinging an IP tests L3 **in isolation from** DNS
   (10.1.2). — 4. The whole of L1–L3 is sound, the problem is **definitely DNS** (or above) (10.1.2). — 5. Do
   not look at DNS at all; **go further down**: run the sequence `ip addr` → `ip route` → `ping <gateway>`
   (10.1.2). — 6. Because "the most likely cause" is a **guess**; when it turns out wrong you do not know what
   you eliminated and you check the same possibilities again. The methodology produces **evidence** at every
   step (the box in 10.1.2).

7. `ip neigh` → **L2** (the ARP table, 3.1); `ss` → **L4** (listeners and connections, 1.4); `dig` →
   **L7/DNS** (6.2); `mtr` → **L3** (the path and loss, 4.5); `curl -v` → **L7** (the HTTP/TLS chain, 8.1)
   (10.2.1). — 8. Because the other tools give an **interpretation** (the operating system's decision), while
   `tcpdump` shows the **truth** that passed over the wire. The three questions it answers: did the packet
   arrive, did the answer go out, **what kind** of answer went out (10.2.2). — 9. The packet **is reaching**
   the target but there is no answer → **a firewall is dropping it** or the service is not listening; no
   answer at all points at the firewall (9.2.3, 10.2.2). — 10. The machine is **reachable** and the kernel is
   answering, but **nothing is listening** on that port → the equivalent of "connection refused"; the network
   is sound, the problem is in the service (5.2.2, 9.2.3). — 11. **No.** Intermediate routers treat producing
   an ICMP answer as low priority and skip it, but carry on forwarding the traffic. What matters is the **last
   hop**; if it is clean the loss is fake (10.2.3, 4.5.2). — 12. **Set `tcpdump` up on the receiving side.**
   Only the receiver can prove the question "did the packet arrive"; capturing on the sender confirms the
   unresolved half of the argument (10.2.2).

13. Source IP, destination IP, source port, destination port, protocol (6=TCP, 17=UDP, 1=ICMP) — the same
    5-tuple as in 9.2.1 (10.4.2). — 14. The packet **never reached that interface** → the problem is earlier:
    the route table, peering, a wrong destination IP (DNS), or the source is not sending at all (10.4.3). —
    15. **The NACL's outbound ephemeral port rule** (`ALLOW TCP 1024-65535`) — because a NACL is stateless, the
    return traffic needs a separate rule (9.4.1, 10.4.3). — 16. (i) It records only **headers**, not content —
    you cannot see TLS/HTTP errors; (ii) it is **not real time**, the aggregation interval is 1–10 minutes (the
    box in 10.4.2). — 17. It eliminates the possibility that **there is no path** between the two VPCs and that
    the peering is down (the basic connectivity is sound). What remains are **direction-specific** causes: A's
    route table, B's inbound rules, A's NACL outbound rule (10.3.3). — 18. Because if you change several things
    at once and the problem clears, **you cannot know which one cleared it** — and you may have created a new
    fault. A problem that clears without your learning anything eats hours again when it repeats (10.5).

## Scoring

| Correct answers | What it means |
|---|---|
| 16–18 | The methodology is yours. You now trust a reflex, not a list. Move on to Phase 11. |
| 13–15 | Good. Read 10.3 once more; the decision trees are the part you will use most in practice. |
| 9–12 | Work on the six steps in 10.1.2 not until you can recite them but until you can **apply** them. |
| 0–8 | Do the finishing lab without fail. This phase is learned by breaking and fixing, not by reading. |

**Which section to go back to for a question you missed:**

| Question | Section |
|---|---|
| 1, 2, 3, 4, 5, 6 | 10.1 The layer-by-layer methodology |
| 7, 8, 9, 10, 11, 12 | 10.2 Tool mastery |
| 17 | 10.3 The three instinct questions |
| 13, 14, 15, 16 | 10.4 VPC Flow Logs |
| 18 | 10.5 When this phase breaks |

---
---

# Phase 10 — Closing and Bridge to Phase 11

## What you carry from this phase

Phase 10 taught you no new mechanism — it taught you **order**. The six bottom-up steps, what each one
eliminates, and how two commands (`ping 1.1.1.1`, `ping google.com`) split thirty possibilities in two. You saw
which layer each tool looks at and why `tcpdump` ends arguments. You drew the decision trees of the three
instinctive questions. And in the cloud you learned to **read from Flow Logs** why a packet was dropped instead
of guessing.

The three most durable sentences: **"bottom-up"**, **"do not settle for an interpretation, look for
evidence"** and **"one change at a time."**

## Where Phase 11 connects

Through eleven phases we have built the **fundamentals** of networking: addresses, subnets, ARP, routing, TCP,
DNS, NAT, HTTP, firewalls, diagnosis. In every phase you saw a "Cloud connection" box and I attached the
pieces to AWS one by one.

Phase 11 **brings those pieces together.**

Cloud networking is not something new — every concept you learned has a cloud name. A VPC is an address block
(Phase 2). A route table is a routing table (Phase 4). A Security Group is a stateful firewall (Phase 9). Route
53 is DNS (Phase 6). A NAT Gateway is PAT (Phase 7). The difference between an ALB and an NLB is the difference
between L7 and L4 (Phases 5, 8).

In Phase 11 we will draw that whole mapping — and at the end, **you will be able to draw an empty VPC from
scratch and explain every part of it on the basis of the fundamentals.** That was the real goal of this book.

> **🤔 Phase output — ask yourself:** You are creating a new VPC in a cloud console. The first thing you meet
> is an **"IPv4 CIDR block"** box. **Why is that the first question?** And which three decisions does the value
> you write there irreversibly affect later on? (Hint: remember 2.4 and 2.5 — one is about divisibility, one is
> about neighbours.)
>
> **🧪 Finishing lab — a deliberate fault (1 🟢, 2–4 🟡):** This is the most valuable exercise in the book. (1)
> 🟢 First record the **healthy** state: run the six steps (10.1.2) and write the outputs into a file. (2) 🟡
> Ask a friend (or yourself, a week later) to set up **one** of these three faults: (a) write a non-working
> nameserver into `/etc/resolv.conf`, (b) delete the default route with `sudo ip route del default`, (c) close
> a port with `sudo ufw deny 443/tcp`. (3) Find the fault **in 5 minutes** by applying the methodology — and,
> importantly, **write down what you eliminate** at each step. (4) Fix it and compare with the healthy output.
> *(Undo: for (a) take a backup of the file first: `sudo cp /etc/resolv.conf /etc/resolv.conf.bak`, then put it
> back. For (b) `sudo ip route add default via <gateway-ip> dev <interface>` — note the gateway IP with `ip
> route` BEFORE deleting it. For (c) `sudo ufw delete deny 443/tcp`.)* Do it on **your own local machine**, not
> on a remote one — (b) and (c) would cut off your connection on a remote server (9.5.3).

---

> **Navigation:** [◀ Checkpoint Quiz 3](Checkpoint_Quiz_3.md) · **Phase 10** · [Phase 11 — The Bridge to the Cloud ▶](Phase_11_Bridge_to_the_Cloud.md)
