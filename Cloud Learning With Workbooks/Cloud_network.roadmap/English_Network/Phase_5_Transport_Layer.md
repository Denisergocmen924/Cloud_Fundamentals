# Phase 5 — The Transport Layer: Reliability and Flow

> **Navigation:** [◀ Checkpoint Quiz 2](Checkpoint_Quiz_2.md) · **Phase 5** · [Phase 6 — From Name to Address: DNS ▶](Phase_6_DNS.md)

---

## Where we come from

At the end of Phase 4 we asked you the most uncomfortable question: a router along the path dropped your
packet and **told nobody**. How will the sender find out?

Between Phases 0 and 4 you learned how to **get** a packet to its destination. This phase teaches what is
done about the ones that do not arrive. What you brought with you:

- **A port is the answer to the question "which application"** (1.4.1) and it is carried in the L4 header.
  This phase opens up the other fields inside that header.
- **A 4-tuple makes a connection unique** (1.4.2). Here you will learn what the word "connection" means
  exactly.
- **IP is best-effort** — a packet can be dropped, the order can change (the closing of Phase 4).
  Reliability is not IP's job; it is **this layer's** job.

## The question of this phase

> *"When the lower layers give no guarantees at all — packets can be dropped, reordered, duplicated — how
> does a file get downloaded **uncorrupted**?"*

The answer comes down to a single idea: **number it and get an acknowledgement.** Every byte sent is
numbered; the receiver acknowledges what it received; if no acknowledgement comes the sender sends it
again. From an idea this simple, the whole reliability layer of the internet is born.

But the real value of this phase is elsewhere: **most real-world packet loss problems originate here.**
"SSH connects but freezes", "small requests work but big files hang", "the site will not open over the
VPN" — all of these are explained in the last section of this phase (**MTU**). That is what the mentor
roadmap calls "the heart of packet loss", and as you read this phase, know that you are heading there.

---

## By the end of this phase

- You will be able to explain the difference between TCP and UDP, where each one is used and **why**
- You will be able to walk through the three-way handshake (SYN → SYN-ACK → ACK) step by step, and diagnose
  what it means when it gets stuck at each step
- You will be able to explain the sequence number and ACK mechanism, and how a lost packet is detected and
  retransmitted
- You will be able to tell flow control (the window) and congestion control apart — **who each one
  protects**
- You will be able to explain why high latency + packet loss **collapses** throughput
- You will know the difference between a graceful close with FIN/ACK and an abrupt cut with RST
- You will be able to explain **MTU, MSS and fragmentation**, and know why an **MTU black hole** is the
  most insidious failure and how it is diagnosed
- **Cloud:** You will be able to explain how load balancers terminate TCP, the MTU problem behind a
  VPN/tunnel, and when a jumbo frame helps

---

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 5.1 | TCP vs UDP | `[concept]` | Two different contracts |
| 5.2 | The three-way handshake | `[mechanism]` | How a connection is established — the basis of diagnosis |
| 5.3 | Sequence, ACK, retransmission | `[mechanism]` | **The heart of reliability** |
| 5.4 | Flow control (the window) | `[mechanism]` | Protecting the receiver |
| 5.5 | Congestion control | `[mechanism]` | Protecting the network |
| 5.6 | Closing a connection | `[concept]` | FIN vs RST |
| 5.7 | MTU, MSS, fragmentation | `[mechanism]` | **The heart of the phase** — the source of packet loss |
| 5.8 | When this phase breaks | — | The signatures of transport failures |

> **How to work through this phase:** Most of this phase's commands are 🟢 but one needs special care:
> `tcpdump` is 🟢 (it only reads) but it requires **root** and can create CPU load when you run it on a
> production machine — use a filter (like `port 443`), do not run a bare `tcpdump`. The most instructive
> experiment is the handshake capture in 5.2: start `tcpdump`, run `curl` in another terminal, and see the
> SYN/SYN-ACK/ACK trio **with your own eyes**. This is the phase best suited to verifying theory in a
> terminal. Do not rush 5.7 — that is the section that will help you most in the field.

---
---

# 5.1 TCP vs UDP — Two Different Contracts

## 5.1.1 Two approaches `[concept]`

There are two main protocols at the transport layer and their difference is philosophical.

**TCP** (Transmission Control Protocol): *"I promise the data will arrive complete and in order — I will
slow down if I have to."*

**UDP** (User Datagram Protocol): *"I will send the packet. Whether it arrives, whether the order holds, I
do not know. But I am fast."*

| Property | TCP | UDP |
|---|---|---|
| Connection | Yes (a handshake is needed) | No (sent straight away) |
| Reliability | Yes (losses are retransmitted) | No |
| Order guarantee | Yes | No |
| Flow/congestion control | Yes | No |
| Header size | 20+ bytes | 8 bytes |
| Latency | Higher | Lower |

## 5.1.2 Which one where — and why `[concept]`

The choice depends on what the application cares about: **correctness or timeliness?**

**Users of TCP** — the data must be complete:
- **HTTP/HTTPS** (the web) — half a page cannot arrive
- **SSH** — losing one character corrupts the command
- **File transfer, database connections, email**

**Users of UDP** — delay is worse than loss:
- **DNS** (Phase 6) — a single-packet question and answer; if it is lost, asking again is cheaper than
  establishing a connection
- **Live video/audio** — there is no point re-requesting a lost frame, that moment has passed
- **Games** — position data that arrives 200 ms late is already useless
- **VPN tunnels** (Phase 7.4) — there is already TCP inside, reliability again on the outside is redundant

The critical intuition: **UDP is not "bad TCP".** It is a different trade-off. If you used TCP in a live
video call, the whole stream would pause for one lost packet — and the viewer would see a frozen picture.
With UDP that frame is skipped, the picture glitches for a moment, and the stream continues. The second is
**better**.

> **❓ A question that comes to mind: "If UDP is not reliable, can reliability not be added on top of it?"**
>
> It can — and that is exactly what is being done. The **QUIC** protocol (which runs underneath HTTP/3,
> Phase 8.3) builds its own layer of reliability, ordering and congestion control on top of UDP. Why does it
> not use TCP? Because TCP is embedded in the operating system kernel and developing it takes years; a
> protocol written on top of UDP runs in **user space** and can be updated quickly. QUIC also solves one of
> TCP's weaknesses: **head-of-line blocking** — in TCP one lost packet holds up the delivery of **all** the
> data behind it; in QUIC the streams are independent, and a loss in one stream does not stop the others.
> So "reliability on top of UDP" is not an oddity, it is the direction the modern internet is going.

> **🤔 Think 5.1** — You are writing a file-sharing application and you are about to add a live
> screen-sharing feature. (a) Which protocol would you choose for each? (b) If you used TCP for screen
> sharing, what would the user **feel** — describe it concretely. (c) If you used UDP for the file
> transfer, what would you be forced to do?
>
> *(Answer: at the end of the phase)*

---
---

# 5.2 The Three-Way Handshake

## 5.2.1 SYN, SYN-ACK, ACK `[mechanism]`

We said TCP "establishes a connection". But since no physical cable is being run, what does "connection"
mean?

**A connection is both sides having confirmed that they hear each other and having agreed on their starting
numbers.** The trio that establishes this is called the **three-way handshake**:

**1. SYN** (client → server)
The client: *"I want to connect. My starting sequence number is 1000."*
(SYN = synchronize. This packet carries no data.)

**2. SYN-ACK** (server → client)
The server: *"Heard you, I received 1000 (ACK=1001). I want to connect too, my starting number is 5000."*
(One packet carries both the acknowledgement and its own request — which is why there are three steps, not
four.)

**3. ACK** (client → server)
The client: *"I received your 5000 (ACK=5001). The connection is established."*

After this, data can flow.

After each step the connection has a **state**: `SYN-SENT` when the client sends the SYN, `SYN-RECV` when
the server sends the SYN-ACK, and `ESTABLISHED` on both sides after the ACK arrives.

**Why three steps?** Because both sides' ability **to send and to receive** has to be confirmed. With two
steps the server could not know that the client received its own reply. After three steps both sides know
this: *"I hear them, and they hear me."*

And the starting numbers are **random** (they do not start at 0). The reason is security: predictable
sequence numbers would make it easier for an attacker to inject forged packets into the stream.

![Figure 5.1 — The TCP three-way handshake: the client announces its starting sequence number with a SYN, the server both acknowledges it and sends its own number with a SYN-ACK, the client completes the acknowledgement with an ACK, and the connection moves to the ESTABLISHED state.](../diagrams/png/nw-5-01-tcp-handshake.png)

In the figure the vertical line on the left is the client and the one on the right is the server; time flows
from top to bottom. Above each arrow are the flags (SYN, ACK) and the numbers. On the right the connection
**state** after each step is marked — those are exactly the states you will see in `ss` output.

## 5.2.2 What you learn when the handshake gets stuck `[application]`

This is the most practical diagnostic tool of this phase. When you watch the handshake with `tcpdump`,
**where it gets stuck** tells you almost exactly what the problem is:

| Observation | What it means |
|---|---|
| SYN goes out, **nothing comes back** (timeout) | A firewall is dropping the packet **silently**, or there is no routing, or there is no host |
| SYN goes out, **an RST comes back** | The host exists and is reachable, but **nothing is listening** on that port (connection refused) |
| SYN goes out, **ICMP unreachable** comes back | A routing problem, or a firewall rejecting explicitly |
| SYN-ACK comes back, **no data flows after the ACK** | The connection is established but data is not getting through → **suspect MTU** (5.7) |
| The handshake completes, **an immediate RST** | The application refused the connection (auth, a limit, the wrong protocol) |

Keep the fourth row in mind especially. Behind the sentence "it connects but it hangs" there is almost
always MTU, and in 5.7 you will see exactly that.

> **🔧 See it on your machine** 🟢 — capture a real handshake
>
> ```
> # Terminal 1:
> $ sudo tcpdump -n -i any 'tcp port 443 and host example.com'
>
> # Terminal 2:
> $ curl -s https://example.com > /dev/null
>
> # Terminal 1 output:
> 10.0.1.50.51234 > 93.184.216.34.443: Flags [S],  seq 2847361029, win 64240
> 93.184.216.34.443 > 10.0.1.50.51234: Flags [S.], seq 1093847261, ack 2847361030
> 10.0.1.50.51234 > 93.184.216.34.443: Flags [.],  ack 1093847262, win 502
> ```
>
> Three lines, three steps. Read the flags: **`[S]`** = SYN, **`[S.]`** = SYN-ACK (the dot means ACK),
> **`[.]`** = ACK only. Note the `seq` and `ack` values: the server's `ack` is **one more than** the
> client's `seq` (2847361029 → 2847361030). That means "I have your number, I am waiting for the next
> byte" (5.3.1). `51234` is the client's **ephemeral source port** (1.4.1). Seeing these three lines once
> with your own eyes fixes the handshake in place permanently.

> **⚠️ Common misconception: "If the connection was established, everything is fine."**
>
> No — the handshake only proves that **small packets** get through. The SYN, SYN-ACK and ACK packets carry
> no data; they are a few dozen bytes. If there is an MTU problem along the path dropping 1500-byte packets,
> the handshake **completes without trouble** and then the connection freezes the moment data starts to flow
> (5.7.3). This is the classic cause of the complaints "SSH connects, the banner comes, then it freezes" and
> "the site opens but the images do not load". The right reflex: a connection **being established** and
> data **flowing** are two separate tests.

> **🤔 Think 5.2** — Your `curl https://api.example.com` command waits 30 seconds and then times out. You
> look with `tcpdump`: SYN packets are going out, there is no reply at all. (a) Write three possible
> causes. (b) If an **RST** had come back instead, how would your diagnosis change? (c) Why is telling
> these two situations apart so valuable?
>
> *(Answer: at the end of the phase)*

---
---

# 5.3 Sequence, ACK and Retransmission

## 5.3.1 Every byte is numbered `[mechanism]`

The answer to Phase 4's last question is here: the sender learns that its packet was lost **because no
acknowledgement came**.

TCP's mechanism is this: **every byte sent is numbered.**

- **Sequence number (seq):** "the number of the first byte in this packet is this."
- **Acknowledgment number (ack):** "I have received everything up to this number, I am waiting for the
  **next** byte."

An example flow:

```
Sender   → seq=1000, 500 bytes of data     (bytes 1000–1499)
Receiver → ack=1500                        ("I got up to 1499, next is 1500")
Sender   → seq=1500, 500 bytes of data     (1500–1999)
Receiver → ack=2000                        ("I got up to 1999")
```

An ACK is **cumulative**: `ack=2000` means "I have received **everything** before 2000". This means lost
ACKs are tolerated automatically — even if one ACK is lost, the next one carries the same information and
more.

## 5.3.2 How loss is detected and repaired `[mechanism]`

When a packet is lost, two different mechanisms come into play:

**Path 1 — Timeout (RTO).** The sender starts a timer for every packet. If no ACK arrives within a certain
time, it **retransmits** the packet. The time is computed dynamically from the measured round-trip time
(RTT). This is a slow mechanism — it requires waiting.

**Path 2 — Duplicate ACK (fast).** More elegant. If the receiver gets data out of order, it sends **the
same ACK over and over**:

```
Sender:   seq=1000 ✓, seq=1500 ✗ (lost), seq=2000 ✓, seq=2500 ✓
Receiver: ack=1500,    —,                ack=1500,   ack=1500
                                          └── "still waiting for 1500!" ──┘
```

When the sender sees **three duplicate ACKs** it does not wait: it concludes "1500 was lost" and
retransmits immediately. This is called **fast retransmit** and it is far quicker than waiting for a
timeout.

The beauty here: the receiver never says "there is a loss". It just repeats **what it is waiting for**. The
sender **infers** the loss from that. No extra message type is needed.

> **🔧 See it on your machine** 🟢 — read the internal state of an open connection
>
> ```
> $ ss -ti
> ESTAB 0  0  10.0.1.50:51234  93.184.216.34:443
>     cubic wscale:8,7 rto:204 rtt:2.157/0.184 mss:1448 cwnd:10
>     bytes_sent:1842 bytes_acked:1842 segs_out:12 retrans:0/0
> ```
>
> `-t` is TCP, `-i` is the internal detail. How to read it: **`rtt:2.157`** = round-trip time in ms (your
> network distance); **`rto:204`** = the retransmission timeout, how many ms an ACK is waited for;
> **`mss:1448`** = the maximum data carryable in one segment (5.7.2 — keep an eye on this number);
> **`cwnd:10`** = the congestion window (5.5.1); **`retrans:0/0`** = the retransmission count. That last
> value is worth gold in diagnosis: **if `retrans` is greater than zero and rising, there is real packet
> loss along the path.** And `cubic` is the name of the congestion control algorithm in use.

> **🤔 Think 5.3** — The sender sends seq=1000, 2000, 3000, 4000 in order (1000 bytes each). The packet
> numbered 2000 is lost. (a) Which ACKs does the receiver send? (b) How and when does the sender learn of
> the loss? (c) Even though the receiver did get 3000 and 4000, why can it not say `ack=5000`?
>
> *(Answer: at the end of the phase)*

---
---

# 5.4 Flow Control — Protecting the Receiver

## 5.4.1 The window: "you may send this much" `[mechanism]`

There is one more problem: the sender may be fast and the receiver slow. If the receiver's buffer fills up
the incoming data is **discarded** — and retransmitted, which is pure waste.

The solution: with every ACK the receiver says **how much room is left**. That value is called the
**receive window** and it is carried in the TCP header.

```
Receiver → ack=2000, win=64240     "I am waiting for 2000, I have 64 KB of room"
Receiver → ack=5000, win=8192      "the buffer is filling, 8 KB left — slow down"
Receiver → ack=7000, win=0         "STOP. I have no room at all."
```

The sender may send at most **one window's worth** of data without an acknowledgement. If it sees `win=0`
it stops completely and waits until the receiver frees space and updates the window.

This is called a **sliding window**: the window slides forward as the ACKs come in. Every ACK carries both
"I got this" and "you may send this much more".

The critical distinction — the two most confused concepts of this phase:

> **Flow control protects the RECEIVER. Congestion control protects the NETWORK.**

Flow control is a limit the receiver states explicitly. Congestion control (5.5) is a limit nobody states
and the sender **estimates**.

---
---

# 5.5 Congestion Control — Protecting the Network

## 5.5.1 Nobody says it, the sender estimates `[mechanism]`

The receiver may be fast but the **path** may be congested. And the routers along the way do not tell you
to slow down — they simply **drop** your packets (the closing of Phase 4).

TCP interprets this as: **packet loss = a congestion signal.**

In addition to the receiver's window, the sender keeps its own internal limit: the **congestion window
(cwnd)**. The real sending rate is the **smaller** of the two:

```
data that may be sent = min(receive window, congestion window)
```

And cwnd is managed like this:

**Slow start:** At the beginning of a connection cwnd is small (a few segments). With every successful ACK
it grows **exponentially**: 1 → 2 → 4 → 8 → 16... This is probing the network: "how much can you take?"

**Congestion avoidance:** Past a certain threshold the growth becomes **linear** (+1 per RTT). From then on
it proceeds carefully.

**On a loss:** cwnd is cut sharply (typically in half) and linear growth restarts. This sawtooth pattern is
called **AIMD** (Additive Increase, Multiplicative Decrease): grow slowly, shrink fast. It is fair and
stable — because every connection applies the same rule, the network's capacity is shared automatically.

## 5.5.2 Why high latency + loss collapses throughput `[mechanism]`

This is the performance fact you will meet most often in the field, and it is counter-intuitive.

Get a feel for this formula:

```
maximum throughput ≈ window size / RTT
```

That is, **with the same window, if RTT doubles, throughput halves.** Even if the connection's bandwidth
does not change at all.

Now add loss: at every loss cwnd halves and recovers slowly, **at the speed of RTT**. If RTT is large,
recovery is slow too. High RTT + regular loss = cwnd can never grow = throughput stays on the floor.

A concrete example: on an intercontinental connection (RTT 150 ms) **1% packet loss** can in practice bring
a 1 Gbps link down to a few Mbps. The link is empty, the bandwidth is there — but TCP cannot use it because
it cannot trust it.

Two practical lessons follow:
1. **When someone says "I have bandwidth but it is slow", look at packet loss first** (`mtr`, `ss -ti` →
   `retrans`). Loss is more decisive than bandwidth.
2. **On long-distance connections small losses have big effects.** The same 1% loss that goes unnoticed on
   a local network is a disaster between continents.

> **💡 Cloud connection — a load balancer splits the TCP:** When you run an application behind an ALB
> (Application Load Balancer), the TCP connection from the client **terminates** at the ALB and the ALB
> opens a **separate** TCP connection to its backend. Two connections, two separate handshakes, two separate
> congestion windows. The benefit is large: if the client is on the other side of the continent (RTT 150 ms)
> the high-latency connection ends at the ALB; the RTT between the ALB and the backend is 1 ms, so that side
> runs at full speed (5.5.2). The ALB also **reuses** the connections it opens to the backend — the cost of
> a new handshake is not paid for every request. What you need to know for diagnosis: in a performance
> problem, know **which connection** you are looking at. The RTT you see with `ss -ti` on the backend is
> **not** the RTT the client experiences — it is the ALB's.

> **🤔 Think 5.4** — A team says that transferring a file from a server in Frankfurt to a client in
> Singapore is very slow (RTT ~160 ms). The link is 1 Gbps and idle. `mtr` shows 0.8% loss. (a) Is the
> bandwidth insufficient? (b) Explain the mechanism of the slowness in two steps. (c) Would the same 0.8%
> loss matter inside the same data centre (RTT 0.5 ms)?
>
> *(Answer: at the end of the phase)*

---
---

# 5.6 Closing a Connection

## 5.6.1 Gracefully with FIN, abruptly with RST `[concept]`

A connection ends in two ways, and the difference matters in diagnosis.

**A graceful close — FIN.** A four-step, mutual goodbye:

```
A → FIN      "I am done, I will send no more data"
B → ACK      "understood"
B → FIN      "I am done too"
A → ACK      "understood — closed"
```

The reason there are four steps is that TCP is **bidirectional**: each direction closes separately. A may
have finished sending while B still has data to send (this is called a "half-close").

**An abrupt cut — RST.** One packet, no discussion: *"This connection is invalid, cut it."*

The situations RST comes from:
- An attempt to connect to a closed port → **connection refused** (1.4.1, 5.2.2)
- The application crashed suddenly or forcibly closed the connection
- A firewall is **actively** rejecting the connection
- The connection entry timed out in a NAT/firewall (Phase 7.2) — the classic "a long-idle SSH session
  dying"

The **TIME_WAIT** state is at `[skip]` level but know its name: a closing connection waits in this state
for a while (typically 60 s) so that late-arriving packets do not get mixed into a new connection. Seeing
thousands of `TIME_WAIT`s on a busy server is **normal**, not a fault.

> **🤔 Think 5.5** — A user says "if I do not type anything for a long time my SSH session drops". Looking
> with `tcpdump`, an **RST** is seen at the moment of the drop. (a) Write two possible causes. (b) If the
> cause is a NAT/firewall timeout, how would you solve it without touching the network configuration?
> (c) Why does that solution work?
>
> *(Answer: at the end of the phase)*

---
---

# 5.7 MTU, MSS and Fragmentation — The Heart of Packet Loss

This section is the most valuable part of this phase. The most insidious network failure you will meet in
the field comes from here, and engineers who do not know how to diagnose it spend days looking in the wrong
place.

## 5.7.1 MTU: the largest payload a frame can carry `[mechanism]`

**MTU** (Maximum Transmission Unit) is the largest IP packet a network interface can carry in a single
frame. On standard Ethernet it is **1500 bytes**.

This is not a protocol preference, it is a **hardware/path limit**: every interface along the path has its
own MTU, and a packet has to fit **the smallest MTU on the path**. That is called the **path MTU**.

The typical things that lower MTU — all of them add an "envelope" (Phase 0.2.1):
- **VPN/IPsec tunnels** (~1400 or less)
- **PPPoE** connections (1492)
- **Cloud tunnels** (VXLAN, GRE — Phase 7.4)

In the other direction: inside data centres **jumbo frames** (9000 bytes) can be used. In AWS, traffic
inside a VPC supports an MTU of 9001 — which is why you see `mtu 9001` in your `ip link` output (Phase
1.1.1). But traffic **going out to the internet** drops back to 1500.

## 5.7.2 MSS: TCP's own share `[concept]`

MTU is the size of the whole IP packet — headers included. The amount of **data** TCP can actually carry is
less:

```
MSS = MTU − IP header (20) − TCP header (20)
1460 = 1500 − 20 − 20
```

**MSS** (Maximum Segment Size) is the amount of data TCP can carry in a single segment. The two sides
announce it to each other **during the handshake** (as an option in the SYN packet) and the smaller one is
used.

The `mss:1448` value you saw in the `ss -ti` output is this (5.3.2). If it is 1448 rather than 1460, there
are additional headers in play (such as timestamps).

## 5.7.3 The MTU black hole — the most insidious failure `[mechanism]`

Now let us come to the failure scenario. Step by step:

1. Your machine **assumes** the path's MTU is 1500 and sends 1500-byte packets.
2. There is a VPN tunnel along the path; its MTU is **1400**.
3. The packet reaches that point. The router has two options:
   - **Fragment** the packet — but if the **DF (Don't Fragment)** bit is set in the IP header this is
     forbidden. And modern TCP **always** sets DF, for path MTU discovery.
   - **Drop** the packet and send the sender an **ICMP Fragmentation Needed** (Phase 4.4.2).
4. The router behaves correctly: it drops the packet and sends the ICMP.
5. **But a firewall along the path blocks ICMP.** The message **never reaches** the sender.
6. The sender learns nothing. It retransmits its packet. Same size. It is dropped again. It sends it again.

The result: a **black hole.** Packets are lost but nobody says why.

And the signature is unmistakable; once you learn it you never forget it:

> **Small packets get through, large packets do not.**

The concrete symptoms:
- **SSH connects, the banner comes, it freezes on the first large output** (something like `ls -la`)
- **The web page opens (the HTML is small), the images do not load** (large)
- **`ping` works** (56 bytes), **`ping -s 1472` does not**
- **The TCP handshake completes, no data flows** (5.2.2, row 4)

> **🔧 See it on your machine** 🟢 — find the path MTU by hand
>
> ```
> $ ping -M do -s 1472 -c2 8.8.8.8
> PING 8.8.8.8 (8.8.8.8) 1472(1500) bytes of data.
> 1480 bytes from 8.8.8.8: icmp_seq=1 ttl=115 time=12.1 ms       ← got through
>
> $ ping -M do -s 1473 -c2 8.8.8.8
> ping: local error: message too long, mtu=1500                  ← the limit is here
> ```
>
> `-M do` = set the DF bit (no fragmentation), `-s 1472` = 1472 bytes of data. Why 1472? Because
> **1472 + 8 (the ICMP header) + 20 (the IP header) = 1500**. So this is the largest ping that fills the
> MTU exactly. The method: raise and lower the size to find **the largest value that gets through**, then
> add 28 — that is the path MTU. If you run this test behind a VPN or a tunnel you will find a value below
> 1500, and that difference is exactly the size of the envelope the tunnel adds (Phase 0.2.1).

> **💡 Cloud connection — the classic "it does not work over the VPN":** In the cloud you will see this
> failure most often behind a Site-to-Site VPN, and the scenario is always the same: instances can ping each
> other, SSH connects, but a large data transfer or an API call freezes. The cause is that the headers IPsec
> adds bring the MTU below 1500, and that ICMP is blocked by a security group or a NACL. There are three
> solutions: (1) **allow ICMP type 3 code 4 (Fragmentation Needed)** — the most correct fix, it makes path
> MTU discovery work; (2) **MSS clamping** — a network device lowers the MSS value in SYN packets so the two
> sides use small segments from the start (it is a standard setting on VPN devices); (3) **lower the
> instance's MTU by hand** (`ip link set dev eth0 mtu 1400` 🟡 — it is temporary, making it permanent
> requires a configuration file). Note: in AWS, traffic inside a VPC uses an MTU of 9001 but traffic leaving
> over a VPN/IGW drops to 1500 — that transition point is where the problem is born.

> **🤔 Think 5.6** — A team cannot connect to a database over the VPN they have just set up. The test
> results: `ping` succeeds, `telnet db 5432` connects, but when they run a query the client freezes.
> (a) What is your diagnosis? (b) With which single command do you confirm it? (c) Propose two different
> solutions and say which one is more correct.
>
> *(Answer: at the end of the phase)*

---
---

# 5.8 When This Phase Breaks — Transport Failure Signatures

The golden rule in diagnosing transport failures is this: **a connection being established and data flowing
are two separate events**, and which of them failed determines the cause almost on its own.

| Symptom | Likely cause | Verify | Related section |
|---|---|---|---|
| SYN goes out, no reply (timeout) | A firewall dropping silently / no routing | Watch the SYN with `tcpdump` | 5.2.2 |
| An RST comes back to the SYN (refused) | The port is not being listened on — the service is down | `ss -tulpn` (on the target) | 5.2.2 |
| The handshake completes, no data flows | **MTU black hole** | `ping -M do -s 1472` | 5.7.3 |
| SSH connects, freezes on the first big output | An MTU black hole (the classic signature) | The same test | 5.7.3 |
| The page opens, the images do not come | An MTU black hole | The same test | 5.7.3 |
| There is bandwidth but the transfer is slow | Packet loss + high RTT | `mtr`, `ss -ti` → `retrans` | 5.5.2 |
| `retrans` keeps rising in `ss -ti` | Real packet loss along the path | `mtr -rwc 50 <destination>` | 5.3.2 |
| Idle connections drop after a while | A NAT/firewall session timeout | The source of the RST, keepalive | 5.6.1 |
| The connection is established, an immediate RST | The application refused it (auth, a limit, the protocol) | The application logs | 5.2.2 |
| Thousands of `TIME_WAIT`s on a server | **Normal** — the waiting period of closed connections | `ss -tan \| grep TIME` | 5.6.1 |
| A UDP service "seems to work" but no data arrives | There is no handshake in UDP — silent loss | Watch both directions with `tcpdump` | 5.1.1 |
| Large requests die behind a VPN in the cloud | IPsec MTU + an ICMP block | `ping -M do`, MSS clamping | 5.7.3 |

> **The lesson from this table:** In transport diagnosis, answer **one single question** first: *is the
> connection being established?* If it is not, the answer is one of three things and `tcpdump` tells you
> within seconds — **a timeout** (a silent drop, a firewall), an **RST** (nothing listening on the port), or
> an **ICMP unreachable** (routing). If it is established but data does not flow, **MTU should be first in
> your suspicions** (5.7.3): handshake packets are small and get through; data packets are large and get
> dropped. That single distinction is the reflex that saves the most time in the field. And in performance
> complaints, look at **loss**, not bandwidth: the `retrans` value in `ss -ti` output says far more than
> whether the link is "full" — because TCP will not fully use a link it cannot trust anyway (5.5.2). One
> last warning: in UDP **none** of these symptoms exist. No handshake, no RST, no retransmission — just
> silence. Diagnosing UDP always requires `tcpdump` **at both ends**.

---
---

# Answers to the Think questions

## Answer 5.1 — The file-sharing application

**(a)** File transfer → **TCP** (the file must arrive complete and in order; a single lost byte corrupts it).
Screen sharing → **UDP** (or a UDP-based protocol like QUIC/WebRTC).

**(b)** If you used TCP for screen sharing: at every lost packet TCP would stop and wait for the
retransmission, and **all the frames behind it** would wait too (head-of-line blocking, 5.1.2). What the
user would feel: the picture freezes for a moment and then **fast-forwards** — the catching-up frames arrive
all at once. Worse, the delay accumulates: the more loss there is, the further behind reality the picture
falls. With UDP that frame would simply be skipped, the picture would glitch for an instant and then carry
on **live**.

**(c)** If you used UDP for the file transfer you would have to write everything TCP does yourself:
numbering the packets, detecting losses, requesting retransmission, correcting the order, flow control. That
is exactly what QUIC does (5.1.2) — and it took years of engineering. For an ordinary file transfer it is a
pointless cost.

**Related section:** 5.1 · **Next:** 5.2 — how a TCP connection is established

## Answer 5.2 — The SYN gets no reply

**(a)** Three possible causes: (1) **a firewall/security group is dropping the packet silently** (DROP, not
REJECT) — the most common; (2) **there is no routing to that IP**, or the return path is missing (Phase
4.2); (3) **the host is down** or its network interface is off.

**(b)** If an **RST** came back the diagnosis would change completely: the packet **reaches the host** and
the host **replies**. That means routing is fine, the firewall is letting it through, the machine is alive —
but **nothing is listening on that port**. The service has not started, has crashed, or is listening on a
different port. The place to look moves from the network to the **application**.

**(c)** Because it splits the problem space in two. A timeout says "look at the network" (routing,
firewall, security group); an RST says "look at the machine" (is the service running, is it listening on the
right port, `ss -tulpn`). These two lead in completely opposite directions, and telling them apart takes one
`tcpdump` command and a few seconds. Trying to solve a problem in the wrong place is the most expensive
mistake in diagnosis.

**Related section:** 5.2.2 · **Next:** 5.3 — how loss is detected

## Answer 5.3 — The lost middle packet

**(a)** The receiver's ACKs: after 1000 arrives it says `ack=2000`. 2000 is lost. 3000 arrives — but the
receiver is still waiting for 2000, so it says **`ack=2000` again**. 4000 arrives — **`ack=2000` again**.
So: `ack=2000`, `ack=2000`, `ack=2000` — duplicate ACKs.

**(b)** The sender sees **three duplicate ACKs** and does not wait for the timer: it concludes "2000 was
lost" and retransmits immediately — **fast retransmit** (5.3.2). If only one duplicate had arrived it would
have waited for the RTO timer, which is far slower.

**(c)** Because the ACK number is **cumulative**: `ack=5000` would mean "I have received everything before
5000" — and that would be a lie, 2000 is missing. TCP's ACK mechanism can only say "I received an
uninterrupted run up to here". (The receiver does **hold on to** the 3000 and 4000 data in its buffer; it is
just not allowed to acknowledge them. The **SACK** option exists exactly for this — "I got these too" — but
the basic mechanism is cumulative.)

**Related section:** 5.3 · **Next:** 5.4 — who limits the sending rate

## Answer 5.4 — Frankfurt → Singapore

**(a)** No, the bandwidth is not the issue at all — the link is 1 Gbps and idle. The problem is that TCP
**cannot use** that bandwidth.

**(b)** The mechanism, in two steps: **(1)** Even without loss, maximum throughput ≈ window / RTT (5.5.2).
With a 160 ms RTT, one window's worth of data can be sent only 6 times a second — to fill 1 Gbps you would
need an enormous window. **(2)** Now add the 0.8% loss: at every loss cwnd is **halved** and grows back
linearly, +1 per RTT. Each RTT is 160 ms, so recovery is glacial — and by the time it recovers the next loss
has arrived. cwnd never manages to grow. The result: a 1 Gbps link may deliver only a few Mbps.

**(c)** Inside the same data centre (RTT 0.5 ms) the same 0.8% loss would matter **far** less: cwnd would be
halved, but recovery would also be 320 times faster (0.5 ms per RTT instead of 160 ms). It would be back at
its old size within milliseconds. That is the rule: **at small RTT loss is tolerable, at large RTT it is
devastating.** The same percentage, entirely different consequences.

**Related section:** 5.5.2 · **Next:** 5.6 — how a connection ends

## Answer 5.5 — The SSH session that drops

**(a)** Two possible causes: (1) **a NAT/firewall session timeout** — the device between drops the entry for
an idle connection from its table and sends an RST at the next packet (Phase 7.2); (2) **a server-side idle
timeout** — SSH's `ClientAliveInterval` setting, or a load balancer's idle timeout.

**(b)** Turn on **keepalive**. On the SSH client side `ServerAliveInterval 60` (in `~/.ssh/config`): the
client sends a tiny packet every 60 seconds. The connection is never idle, so the NAT entry is refreshed.
On the server side the equivalent is `ClientAliveInterval`. At the TCP level the same job is done by
`SO_KEEPALIVE`, but the SSH-level setting is more practical.

**(c)** It works because a NAT/firewall timeout is measured by **how long no traffic has passed**. Keepalive
guarantees that traffic passes regularly — so the counter never fills. The device thinks the connection is
alive, because it is. This is the shortest path to solving the problem without touching the network
configuration.

**Related section:** 5.6.1 · **Next:** 5.7 — the heart of packet loss

## Answer 5.6 — The database connection over the VPN

**(a)** Diagnosis: an **MTU black hole** (5.7.3). The signature fits perfectly: `ping` works (small
packet), `telnet` connects (the handshake packets are small), but running a query freezes it (the response
is large). The IPsec headers the VPN adds have brought the path MTU below 1500 and something along the way
is blocking the ICMP Fragmentation Needed message.

**(b)** A single command confirms it:

```
$ ping -M do -s 1472 <db-ip>      # if this fails but -s 1372 succeeds → MTU problem
```

If the small one gets through and the large one does not, the diagnosis is certain.

**(c)** Two solutions: **(1) Allow ICMP type 3 code 4** on the security group / NACL / firewall. This is
**the more correct** solution, because it makes path MTU discovery work as designed: every connection
discovers the right size by itself, and if the tunnel's MTU changes later the system adapts automatically.
**(2) MSS clamping** or lowering the MTU by hand (`ip link set dev eth0 mtu 1400` 🟡). This works but it is
a workaround: it has to be configured on every machine, it must be kept up to date, and if the value is
wrong the problem resurfaces. The rule: first make the mechanism work, only clamp if you cannot.

**Related section:** 5.7.3 · **Next:** 5.8 — the failure table

---
---

# Frequently asked questions

**Q1. If TCP is this reliable, why is UDP used at all? Would it not be better to make everything
reliable?**
Because reliability has a **price**: waiting. TCP waits for a lost packet, retransmits it and holds
everything behind it (5.1.2). In a live video call that wait is worse than the loss — a 200 ms-late frame
is already useless. UDP is not "bad TCP"; it is a different trade-off: *speed over correctness*. And when
reliability is wanted **over** UDP, QUIC is written (5.1.2).

**Q2. Why three steps in the handshake and four in the close?**
In the handshake the server can put both its acknowledgement and its own request into a single packet
(SYN-ACK) — so three are enough. In the close each direction has to be closed **separately**: A may have
finished sending while B still has data. So FIN, ACK, FIN, ACK — four steps (5.6.1).

**Q3. What is the difference between the sequence number and the ACK number?**
`seq` = "the number of the first byte **in this packet** I am sending". `ack` = "the number of the byte I
am **waiting for next**". That is, ack is always **one more** than the last byte received. In the handshake
capture in 5.2.2, the server's `ack` being one more than the client's `seq` is exactly this (5.3.1).

**Q4. What is the difference between flow control and congestion control — they both look like slowing
down?**
The limits are set by different parties. **Flow control:** the receiver says it explicitly — "my buffer is
this full, send this much" (the `win` value). **Congestion control:** nobody says it; the sender
**estimates** it from packet loss (cwnd). Flow control protects the **receiver**, congestion control
protects the **network** (5.4.1).

**Q5. Why is the "it connects but data does not flow" case always MTU?**
Not always, but it is always the first suspect — because it is the **only** failure that produces exactly
this pattern. Handshake packets carry no data, a few dozen bytes; they fit through any MTU. Data packets
are large (up to the MSS). If the path's MTU is small and ICMP is blocked, only the large ones are dropped.
The result: it connects, it freezes (5.7.3).

**Q6. What does it mean when there are thousands of TIME_WAITs? Is it a fault?**
It is **normal**. A closing connection waits in `TIME_WAIT` for about 60 seconds so that late packets do not
get mixed into a new connection (5.6.1). On a busy web server thousands of TIME_WAITs mean thousands of
requests were served successfully. It becomes a problem only if the ephemeral port pool is exhausted
(1.4.1) — and that shows up as "cannot assign requested address" errors.

**Q7. Why does packet loss hurt more on distant connections?**
Because recovery is measured in RTT (5.5.2). At every loss cwnd is halved and grows back **+1 per RTT**. If
RTT is 0.5 ms, recovery takes milliseconds; if RTT is 160 ms, hundreds of times longer — and in the
meantime the next loss has arrived. The same 1% loss that goes unnoticed locally can drop an
intercontinental connection to a few Mbps.

---
---

# Test yourself

Answer without looking. Write your answers down; check them against the key afterwards.

## Part A — Reasoning about the mechanisms

**1.** Explain in one sentence each what the SYN, SYN-ACK and ACK packets say.

**2.** Why are three steps needed in the handshake rather than two?

**3.** What is the difference between the `seq` and `ack` numbers? Answer using the words "I am waiting for".

**4.** What does it mean when the receiver sends the **same** ACK three times, and what does the sender do?

**5.** Which side does flow control protect, and which side does congestion control protect?

**6.** Write the formula `MSS = ...` and say what each number is.

**7.** What is the difference between a FIN and an RST?

**8.** What is a "path MTU" and why is it not the same as the interface's MTU?

## Part B — Scenario reasoning

**9.** `curl` times out; `tcpdump` shows the SYN going out with no reply at all. Write two possible causes
and say how you would tell them apart.

**10.** SSH connects, the banner comes, and the moment you type `ls -la` it freezes. What is your
diagnosis and with which command do you confirm it?

**11.** The link is 1 Gbps and idle, RTT is 180 ms, there is 1% loss and the transfer runs at 3 Mbps. Is
the bandwidth insufficient? Explain the mechanism.

**12.** There are thousands of `TIME_WAIT`s on a server. Is this a fault? In what case would it become
one?

**13.** A UDP-based service "seems to be working" but the client gets no data. Why is this harder to
diagnose than TCP, and how would you approach it?

**14.** A long-idle SSH session drops with an RST. Write the cause and a solution that does not touch the
network configuration.

## Part C — Reading commands and output

**15.** In `ss -ti` output you see `rtt:148.3/2.1 retrans:14/230 cwnd:4 mss:1448`. Interpret these four
values and say what the overall picture tells you.

**16.** In `tcpdump` output you see `Flags [S]`, then `Flags [R.]`. What happened, and where does the
problem lie — the network or the application?

**17.** `ping -M do -s 1472 10.0.5.20` fails, `ping -M do -s 1372 10.0.5.20` succeeds. What is the path
MTU roughly, and what would you look at?

**18.** What is different about `ss -tan | grep TIME_WAIT | wc -l` returning 8400 on a web server versus on
a database server?

---

## Answer key

**1.** SYN: "I want to connect, my starting number is X." SYN-ACK: "I received X, I want to connect too, my
number is Y." ACK: "I received Y, the connection is established." (5.2.1) **2.** Because **both** sides'
ability to send and receive has to be confirmed; with two steps the server could not know that the client
received its reply (5.2.1). **3.** `seq` = the number of the first byte in the packet I am sending; `ack` =
"I received everything up to here, **I am waiting for** this number next" — always one more than the last
byte received (5.3.1). **4.** Duplicate ACKs: the receiver got out-of-order data and is repeating what it is
waiting for; after three the sender concludes there was a loss and retransmits immediately — fast
retransmit, without waiting for the RTO (5.3.2). **5.** Flow control protects the **receiver** (its buffer),
congestion control protects the **network** (the routers along the path) (5.4.1). **6.** `MSS = MTU − 20
(IP) − 20 (TCP)` → 1460 = 1500 − 20 − 20; MSS is the amount of **data** TCP can carry in one segment
(5.7.2). **7.** FIN is a graceful, mutual, four-step close; RST is an abrupt one-packet cut — a closed port,
a crash, a firewall rejection, a NAT timeout (5.6.1). **8.** The path MTU is the **smallest** MTU along the
whole path; your interface may be 1500 but a tunnel in the middle may be 1400 — the packet has to fit the
smallest one (5.7.1). **9.** (i) A firewall/security group dropping it silently, (ii) no routing / the host
is down. To tell them apart: check routing (`ip route get`), look for the packet on the target machine with
`tcpdump` — if it arrives there and no reply goes back, the filtering is local (5.2.2). **10.** An **MTU
black hole**: small packets get through, the large output does not. Confirm with `ping -M do -s 1472
<target>` — if it fails but a smaller size succeeds, the diagnosis is certain (5.7.3). **11.** No, the
bandwidth is not the problem: throughput ≈ window / RTT, and at every loss cwnd is halved and recovers +1
per RTT — with a 180 ms RTT recovery is glacial and the next loss arrives first; cwnd stays permanently
small (5.5.2). **12.** It is **normal** — closed connections wait about 60 s so that late packets are not
mixed into a new connection; it becomes a problem only when the ephemeral port pool is exhausted (5.6.1,
1.4.1). **13.** Because UDP has **no** handshake, no RST and no retransmission — there is only silence;
there is no symptom to read. The approach: run `tcpdump` **at both ends** simultaneously and see where the
packet stops (5.1.1). **14.** A NAT/firewall session timeout dropping the entry for an idle connection;
solution: `ServerAliveInterval 60` in `~/.ssh/config` — regular keepalive traffic keeps the entry alive
(5.6.1). **15.** rtt 148 ms = a very distant connection; `retrans:14/230` = 14 of 230 segments were
retransmitted, about 6% loss — very high; `cwnd:4` = the congestion window has been beaten down to almost
nothing; `mss:1448` = normal. The overall picture: a distant connection + serious loss → the throughput will
be on the floor, and the work is to find the loss (`mtr`), not to add bandwidth (5.3.2, 5.5.2). **16.** The
SYN went out and an **RST** came back: the host is alive and replied, but nothing is listening on that port.
The problem is on the **application** side — the service is down or on a different port; check with `ss
-tulpn` (5.2.2). **17.** The largest size that gets through is between 1372 and 1472, so the path MTU is
somewhere around 1400–1500 — probably 1400, meaning there is a **tunnel/VPN** in the path. What to look at:
is ICMP type 3 code 4 allowed, and is MSS clamping configured (5.7.3). **18.** On a web server it is
expected — thousands of short-lived HTTP connections open and close (each leaves a TIME_WAIT). On a database
server it is **suspicious**: database connections are normally long-lived and pooled; thousands of
TIME_WAITs mean the application is opening and closing a new connection for every query — a connection-pool
misconfiguration (5.6.1).

---

## Scoring

| Correct answers | What it means | What to do |
|---|---|---|
| 16–18 | The transport layer is solid | Go on to Phase 6 |
| 12–15 | The mechanisms are in place, the details are shaky | Reread 5.3 and 5.7, then continue |
| 8–11 | The outline is there, the reasoning is not | Rework 5.2, 5.5 and 5.7, redo Lab 5 |
| 0–7 | Not yet settled | Reread the phase from the beginning; do not skip 5.7 |

**Which section to go back to for a question you missed:**

| Question | Section |
|---|---|
| 1, 2, 9, 16 | 5.2 — the handshake and its diagnosis |
| 3, 4, 15 | 5.3 — seq/ACK and retransmission |
| 5 | 5.4, 5.5 — the window and congestion |
| 11, 15 | 5.5.2 — RTT, loss and throughput |
| 7, 12, 14, 18 | 5.6 — closing a connection |
| 6, 8, 10, 17 | 5.7 — MTU/MSS and the black hole |
| 13 | 5.1 — TCP vs UDP |

---
---

# Closing and the Bridge to Phase 6

You have finished the transport layer. You now know how a connection is established, how loss is detected
and repaired, who slows the sending rate down and why, and — most valuably — the signature of the MTU black
hole: **small packets get through, large packets do not**. That single sentence will save you days in the
field.

But notice something. Every example in this phase started with an **IP address**. `tcpdump` showed
`93.184.216.34.443`; the SYN packet went to an IP. Yet you wrote `curl https://example.com`.

> **🤔 Phase output — carry this question into the next phase:**
>
> The SYN packet is the **first** packet of the connection, and it needs a destination IP in its header. But
> you typed a **name**. That means before the connection is established, something turned that name into an
> address — and that step happened **before** everything in this phase. If that step fails, **the SYN is
> never sent at all**. In diagnosis this makes a big difference: "no reply to the SYN" and "there was no SYN
> at all" are two completely different failures. So who does that translation, how long does it take, and
> what happens when it goes wrong? That is Phase 6.

> **🧪 Lab 5 idea — watch a connection from birth to death**
>
> 1. Start `sudo tcpdump -n -i any 'tcp port 443 and host example.com' -w /tmp/cap.pcap` in one terminal,
>    run `curl -s https://example.com > /dev/null` in another, and stop the capture.
> 2. Open the capture with `tcpdump -r /tmp/cap.pcap` and find the three lines of the handshake; write down
>    the `seq` and `ack` numbers and check the "one more" rule for yourself (5.3.1).
> 3. Find the closing packets at the end of the capture: is it FIN or RST? (5.6.1)
> 4. Run `ss -ti` while a large download (`curl -o /dev/null https://speed.hetzner.de/100MB.bin`) is in
>    progress and watch `cwnd` grow and `retrans` change (5.3.2, 5.5.1).
> 5. Find your path MTU by hand: raise and lower the size with `ping -M do -s <size> 8.8.8.8` and find the
>    largest value that gets through; add 28 (5.7.3). If you have a VPN, repeat the same test with the VPN
>    on and off and compare the two numbers.

---

> **Navigation:** [◀ Checkpoint Quiz 2](Checkpoint_Quiz_2.md) · **Phase 5** · [Phase 6 — From Name to Address: DNS ▶](Phase_6_DNS.md)
