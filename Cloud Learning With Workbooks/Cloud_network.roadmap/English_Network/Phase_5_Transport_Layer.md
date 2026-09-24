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

# Phase 5 — Answers to the Think questions

## Answer 5.1 — Correctness or timeliness

(a) **File transfer → TCP** (if a single byte is missing the file is corrupt). **Live screen sharing → UDP**
(delay matters more than quality).

(b) In TCP a single lost packet holds up the delivery of **all** the data behind it (head-of-line blocking,
the box in 5.1.2). What the user feels: the picture **freezes**, then the accumulated frames suddenly rush
past, then it freezes again. The delay builds up and the "live" quality is lost. With UDP that frame is
**skipped**: the picture glitches for a moment but the stream stays real-time — for the viewer the second is
far better.

(c) You would have to **write the reliability yourself**: numbering the packets, detecting the missing ones,
asking for them again, fixing the order, and adding congestion control. That is, reinventing TCP. This is
sometimes genuinely done (QUIC is exactly this, the box in 5.1.2) but it needs a justification — for an
ordinary file transfer TCP is already the right tool.
**Related section:** 5.1.1–5.1.2 · **Next:** Phase 8.3 (HTTP/3 and QUIC).

## Answer 5.2 — Silence and RST say different things

(a) The SYN goes out and there is no reply at all → three possibilities: (i) **a firewall is dropping the
packet silently** (a DROP policy — the most common cause, in the cloud a security group or NACL); (ii) **a
routing problem** — the packet does not reach the destination or there is no return path (asymmetric
routing, Phase 4.7); (iii) **the host is missing or down** — there is no machine running at that IP.

(b) **If an RST had come back** the diagnosis would narrow a great deal: an RST proves that **the target
machine exists, is reachable and received the packet** (5.2.2). The problem is not in the network but at the
target: no process is listening on that port (Phase 1.4.1 — "connection refused"). That is, the suspicion
moves **from the network to the application** — you would be talking to a completely different team.

(c) Because the two point to **completely different teams**. *Silence* = a network/security problem → look
at the firewall rules, the route table, the security group. *RST* = an application problem → is the service
up, is it listening on the right port, is it bound to the right interface (`0.0.0.0` vs `127.0.0.1`,
Phase 1.4.2). This single distinction saves hours of searching in the wrong place.
**Related section:** 5.2.2 · **Next:** Phase 9.2.3 (DROP and REJECT), Phase 10.1 (layer-by-layer diagnosis).

## Answer 5.3 — A cumulative ACK cannot jump a gap

(a) The ACKs the receiver sends:
- 1000 received → `ack=2000` ("I am waiting for 2000")
- 2000 was **lost** → nothing
- 3000 arrived (out of order) → **`ack=2000`** (still the same!)
- 4000 arrived (out of order) → **`ack=2000`** (the same again)

(b) The sender counts the **duplicate ACKs**. When it sees three identical ACKs (`ack=2000`, `ack=2000`,
`ack=2000`) it does not wait for the timeout: it understands that 2000 was lost and retransmits immediately
— **fast retransmit** (5.3.2). If the duplicate ACKs do not come either (for example because the following
packets were lost as well), then the **RTO timeout** kicks in, but that is slower.

(c) Because the ACK is **cumulative**: `ack=5000` would mean "I have received **everything** before 5000"
(5.3.1). But the receiver has not received 2000–2999 — that would be a lie, and the sender would never
retransmit the lost packet. Even if the receiver has received data beyond the gap, it must point at **the
start of the gap**. (Note: TCP's **SACK** — Selective ACK — extension makes it possible to say "2000 is
missing but I have 3000–4999" and makes retransmission far more efficient. It is on by default in modern
systems.)
**Related section:** 5.3.1–5.3.2 · **Next:** 5.5.1 (loss = a congestion signal).

## Answer 5.4 — Loss combined with distance is deadly

(a) **No.** The link is 1 Gbps and idle — bandwidth is not the problem. The problem is that TCP **cannot
use** that bandwidth.

(b) Two steps:
1. **Throughput ≈ window / RTT** (5.5.2). Because the RTT is 160 ms, the rate achievable with the same
   window is already low — the window would have to grow very large.
2. **At every loss cwnd is halved** and it grows back **at the speed of RTT** (AIMD, 5.5.1). With a 160 ms
   RTT recovery is very slow; with 0.8% loss the window can never grow before a new loss arrives. The
   result: cwnd stays permanently low and throughput sits on the floor. The link is empty but TCP cannot use
   it because it cannot trust it.

(c) **It would hardly matter.** With an RTT of 0.5 ms, cwnd recovers within milliseconds of a loss — 0.8%
loss would not even be noticed. The same loss rate turns into a disaster with distance. That is why "loss
percentage" on its own is meaningless; it has to be judged **together with RTT**.
**Related section:** 5.5.1–5.5.2 · **Next:** Phase 8.5 (CDN — shortening the distance), Phase 10.2.3 (finding
the loss along the path with `mtr`).

## Answer 5.5 — Something forgets the idle connection

(a) Two causes: (i) **a NAT or stateful firewall session timeout** (5.6.1) — the device in between deletes
the entry for a connection it has not seen a packet on for a long time; the next packet is treated as an
"unknown connection" and rejected with an RST (Phase 7.2, Phase 9.1). (ii) **A server-side idle timeout** —
sshd or a load balancer closing an idle session.

(b) With **keepalive**: you make small packets be sent over the connection at regular intervals, so the
device in between sees the connection as "alive" and does not delete its entry. In SSH that is
`ServerAliveInterval 60` on the client side and `ClientAliveInterval 60` on the server side. There is also
`SO_KEEPALIVE` at the TCP level, but its default interval is very long (2 hours) — application-level
keepalive is more reliable.

(c) Because the problem is not in the connection **itself** but in the **memory** of the device in between.
That device remembers a connection only as long as it sees traffic (Phase 9.1 — stateful tracking). Regular
small packets refresh that memory. You fix the problem without touching the network configuration, only by
changing behaviour at the endpoints — which is usually the only option you have, because the NAT/firewall in
between is under someone else's control.
**Related section:** 5.6.1 · **Next:** Phase 7.2 (the NAT session table), Phase 9.1 (the stateful firewall).

## Answer 5.6 — The classic MTU black hole

(a) An **MTU black hole** (5.7.3). The signature fits perfectly: small packets get through (ping ✓, the
handshake ✓ — `telnet` connects), large packets do not (the query result = large data → it freezes). That
the VPN was only just set up makes the suspicion certain: the IPsec headers have pushed the MTU below 1500.

(b) **`ping -M do -s 1472 <target>`** (the box in 5.7.3). If this does not get through, the path MTU is below
1500. Reduce the size (1400, 1372, 1300...) to find the largest value that gets through; add 28 to learn the
real path MTU.

(c) Two solutions:
- **MSS clamping** (lowering the MSS value of SYN packets on the VPN device/gateway) — the two sides use
  small segments from the start.
- **Allowing ICMP type 3 code 4 (Fragmentation Needed)** — it lets path MTU discovery work on its own.

**The second is the more correct one**, because it fixes the root cause: the mechanism already exists, it
is just blocked. Once ICMP is open the system finds the right MTU itself **for every path** — not only for
this VPN but for all future paths. MSS clamping works but it is a patch: it has to be configured separately
for every new tunnel and it protects only TCP (it does not help with UDP tunnels). In practice most
organisations apply **both**.
**Related section:** 5.7.3 · **Next:** Phase 7.4 (tunnelling), Phase 9.3 (ICMP policy), Phase 10.1 (the
diagnostic methodology).

---
---

# Phase 5 — Frequently asked questions

**Q1 — Why is UDP still used; does TCP not do everything better?** Because TCP's guarantees have a
**price**: handshake latency, stalling while waiting for a lost packet (head-of-line blocking), and
congestion control throttling the speed. In real-time applications that price is more harmful than the loss
(5.1.2). And for single-packet question-and-answer exchanges like DNS, setting up a handshake is pure waste.

**Q2 — Why is the handshake three steps; are two not enough?** No. With two steps (SYN, SYN-ACK) the server
cannot know that the client **received** its own reply. After the third ACK both sides are in the position
"I hear them, and they hear me" — two-way communication has been proven (5.2.1).

**Q3 — What exactly is the difference between flow control and congestion control?** **Flow control protects
the receiver** and is a limit the receiver **states explicitly** (the receive window, carried in every ACK).
**Congestion control protects the network** and is a limit nobody states, which the sender **estimates from
packet loss** (cwnd). The sender uses the **smaller** of the two (5.4.1, 5.5.1).

**Q4 — I see thousands of `TIME_WAIT`s in `ss` output; is that a problem?** Usually **no** — it is normal
behaviour (5.6.1). Thousands of TIME_WAITs are expected on a busy server. The only case where it is a
problem is when it leads to ephemeral port exhaustion (Phase 1.4.2, Think 1.4) — then you see "cannot assign
requested address" errors. The fix is usually **connection reuse** (keep-alive, connection pooling), not
fiddling with the TIME_WAIT duration.

**Q5 — Does lowering the MTU hurt performance?** To some extent, yes: the header ratio rises in every
packet and more packets are needed for the same data. But it is **far better than a connection that does
not work**. The right approach is to make path MTU discovery work (allow ICMP) rather than lowering the MTU
by hand — then every path uses its own correct value (5.7.3).

**Q6 — When do jumbo frames (9000 MTU) make sense?** Only in a network you **control end to end**: inside a
data centre, storage networks, traffic inside a VPC. If there is a single 1500-byte interface on the path,
jumbo frames are either fragmented or dropped. In AWS traffic inside a VPC supports 9001 MTU but traffic
leaving over an IGW/VPN drops to 1500 (5.7.1) — that transition point is where problems are born.

**Q7 — At what value of `retrans` should I worry?** Not the absolute number but the **ratio and the trend**
matter. Below 0.1% of the total segment count is usually noise. Above 1%, especially on a high-RTT
connection, is a serious performance problem (5.5.2). What to really look at is **whether it is rising**:
run `ss -ti` a few seconds apart; if `retrans` keeps climbing there is active loss on the path, and the next
tool is `mtr`.

---
---

# Phase 5 — Test yourself

Write your answers on paper, then compare them with the key. Target: 14+ out of 18.

## Part A — Definitions and mechanisms

1. Write the four differences between TCP and UDP.
2. Why does DNS use UDP? Why does video streaming use UDP? Are the two reasons different?
3. Write the three steps of the three-way handshake and what is said at each step.
4. Why is the handshake three steps — why are two not enough?
5. What do the sequence number and the ACK number mean? What does it mean that an ACK is "cumulative"?
6. What is fast retransmit and which signal triggers it?
7. What is the difference between flow control and congestion control? Whom does each protect?
8. Write the relationship between MTU and MSS as a formula.

## Part B — Apply and diagnose

9. In `tcpdump` the SYN goes out but there is no reply at all. Write two possible causes.
10. An RST comes back to the SYN. What is your diagnosis, and which team does the suspicion shift to?
11. In `ss -ti` output you see `retrans:47/1200`. What does it mean, and what is your next step?
12. SSH connects, the banner comes, and the moment you type `ls -la` it freezes. What is your diagnosis?
13. With which command do you test the path MTU by hand? Where does the `-s 1472` value come from?
14. RTT is 200 ms, loss is 1%, the link is 10 Gbps. Why is the transfer slow?

## Part C — Reasoning and connection

15. When you see the symptom "the handshake is fine but no data flows", what should your first suspicion be
    and **why**?
16. When a router drops a packet it does not tell the sender. How does TCP notice the loss? Write two ways.
17. What is ICMP's role in an MTU black hole? What would happen if ICMP were not blocked?
18. On a backend behind an ALB you see an RTT of 1 ms in `ss -ti`, but users complain about slowness. It is
    not a contradiction — explain.

---

## Answer key

1. **TCP:** connection-oriented (handshake), reliable (retransmission), ordered, flow/congestion
   controlled. **UDP:** connectionless, unreliable, unordered, uncontrolled — and a smaller header, lower
   latency (5.1.1). — 2. **DNS:** a single-packet question and answer; setting up a handshake would be
   waste, and asking again is cheaper if it is lost. **Video:** delay is more harmful than loss; a frame
   that arrives late is useless anyway. The reasons are **different**: one is efficiency, the other is
   timeliness (5.1.2). — 3. **SYN** ("I want to connect, my seq number is X"), **SYN-ACK** ("I received X,
   my number is Y"), **ACK** ("I received Y — established") (5.2.1). — 4. Because with two steps the server
   cannot know that the client **received its own reply**; the third step proves two-way communication
   (5.2.1). — 5. **Seq:** the number of the first byte in this packet. **ACK:** "I have received everything
   up to this number, I am waiting for the next". **Cumulative** = **everything** before the ack value has
   been received (5.3.1). — 6. Retransmitting the lost packet without waiting for the timeout; it is
   triggered by **three duplicate ACKs** (5.3.2). — 7. **Flow control protects the receiver** (the receive
   window the receiver announces); **congestion control protects the network** (the cwnd the sender
   estimates from loss). What is sent = min(the two) (5.4.1, 5.5.1). — 8. **MSS = MTU − IP header (20) − TCP
   header (20)**; on standard Ethernet 1460 = 1500 − 40 (5.7.2).

9. (i) A firewall is **dropping the packet silently** (DROP); (ii) a routing problem or no return path;
   (iii) the host is missing/down (5.2.2). — 10. The target machine **exists, is reachable and received the
   packet** — but **no process is listening** on that port. The suspicion shifts from the network to the
   **application** (5.2.2). — 11. 47 of 1200 segments were retransmitted (~4%) — **high**, there is real
   packet loss on the path. Next step: find at which hop the loss happens with **`mtr`** (5.3.2, 5.5.2). —
   12. An **MTU black hole** — small packets (the handshake, the banner) get through, large packets (the
   `ls -la` output) are dropped (5.7.3). — 13. **`ping -M do -s 1472 <target>`**. 1472 + 8 (ICMP header) + 20
   (IP header) = **1500**, that is, the largest ping that fills the MTU exactly (5.7.3). — 14. Throughput ≈
   window / RTT; because the RTT is large the window would have to grow very large, but 1% loss halves cwnd
   every time and, because recovery happens at the speed of RTT, the window can never grow. The bandwidth
   cannot be used (5.5.2).

15. **MTU.** Because handshake packets are **small** (a few dozen bytes) and always get through; data
    packets are **large** (up to the MSS) and are dropped if there is a low-MTU point on the path. "It
    connects but nothing flows" is the direct signature of this distinction (5.2.2, 5.7.3). — 16. (i)
    **Timeout (RTO):** if the ACK does not arrive within a certain time the packet is retransmitted. (ii)
    **Three duplicate ACKs:** when the receiver gets out-of-order data it repeats the same ACK; the sender
    treats this as a loss signal and does a **fast retransmit** (5.3.2). — 17. The ICMP **Fragmentation
    Needed** message carries "the packet is too big, the MTU is this" to the sender — it is the only
    information channel of path MTU discovery. If it is blocked the sender cannot learn, resends at the same
    size, and the packet is dropped again: a **black hole** (5.7.3, Phase 4.4.2). If ICMP were open the
    sender would learn the MTU and lower its segment size — the problem would resolve itself. — 18. Because
    the ALB **terminates** the TCP connection: client↔ALB and ALB↔backend are **two separate** connections.
    The 1 ms you see on the backend is the RTT between the ALB and the backend; the delay the client
    experiences (perhaps 150 ms + loss) is on the client↔ALB side and is **invisible** from the backend.
    In performance diagnosis it is essential to know which connection you are looking at (the cloud box in
    5.5.2).

## Scoring

| Correct answers | What it means |
|---|---|
| 16–18 | Transport has settled — especially the MTU reflex. You are ready for Phase 6. |
| 13–15 | Good. Read 5.7 (MTU) once more; that is what will help you most in the field. |
| 9–12 | The handshake and the loss mechanism may be getting mixed up. Verify 5.2 and 5.3 with `tcpdump`. |
| 0–8 | Walk the phase again. The goal: to say MTU the moment you hear "it connects but nothing flows". |

**Which section to go back to for a question you missed:**

| Question | Section |
|---|---|
| 1, 2 | 5.1 TCP vs UDP |
| 3, 4, 9, 10 | 5.2 The handshake |
| 5, 6, 11, 16 | 5.3 Seq/ACK/retransmission |
| 7 | 5.4 + 5.5 Flow vs congestion |
| 14, 18 | 5.5.2 Throughput and RTT |
| 8, 12, 13, 15, 17 | 5.7 MTU and the black hole |

---
---

# Phase 5 — Closing and the Bridge to Phase 6

## What you carry out of this phase

Phase 5 gave you **how reliability is built**. You learned that TCP and UDP are different contracts — one
chooses correctness, the other timeliness. You picked up the three-way handshake and its diagnostic value
(the timeout / RST / ICMP trio). You saw how numbering and acknowledgement detect loss without any extra
message. You told apart that flow control protects the **receiver** and congestion control protects the
**network**, and understood why high RTT + loss collapses throughput. And most valuably: you learned the
signature of the **MTU black hole** — *small packets get through, large packets do not.*

The two most durable sentences: **"establishing a connection and data flowing are two separate events"**
and **"if it connects but nothing flows, let your first suspicion be MTU."**

## Where Phase 6 connects to this

From Phase 0 up to here we did everything with **IP addresses**. `ping 8.8.8.8`, `curl
https://93.184.216.34`, route tables, prefixes — all of it in numbers.

But in real life you do not type `93.184.216.34`. You type `example.com`.

So **how** does that name turn into the 32-bit number the machine needs? And when that conversion breaks,
why does everything look like "no internet" — even though IP, routing and TCP are perfectly healthy?

Phase 6 is the answer: DNS. You will see the name hierarchy, the resolution chain (resolver → root → TLD →
authoritative), the record types, and why the cache/TTL is both the biggest accelerator and the most
insidious source of failures.

Phase 5 made data flow **reliably**; Phase 6 teaches finding **where** it will flow.

> **🤔 Phase output — ask yourself:** When you type `curl https://example.com`, for the machine to start the
> TCP handshake it first has to know the destination **IP** (5.2.1 — the destination IP field of the SYN
> packet has to be filled in). But all it has is a **name**. Whom will it ask for that name — and where will
> it get the address of that "whom" from? (Hint: it was one of the four things DHCP handed out in Phase
> 1.6.1.)
>
> **🧪 Lab 5 idea (all 🟢, tcpdump needs root):** (1) Start `sudo tcpdump -n 'tcp port 443 and host
> example.com'`, run `curl -s https://example.com > /dev/null` in another terminal; find the SYN / SYN-ACK /
> ACK trio together with its flags. (2) At the same time run `ss -ti` and read the `rtt`, `mss`, `cwnd` and
> `retrans` values. (3) Test your path MTU with `ping -M do -s 1472 8.8.8.8`; if it gets through you are at
> 1500, if not, reduce the size and find the limit. (4) Run `mtr -rwc 30 <a distant target>` and compare
> `retrans` with the per-hop loss — do the two agree? (5) Connect to a closed port (`curl -v
> http://localhost:9999`) and see **connection refused**; then connect to a filtered address and see the
> **timeout**. Living the two symptoms side by side makes 5.2.2 permanent.

---

> **Navigation:** [◀ Checkpoint Quiz 2](Checkpoint_Quiz_2.md) · **Phase 5** · [Phase 6 — From Name to Address: DNS ▶](Phase_6_DNS.md)
