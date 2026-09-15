# Phase 5 — Network Hardware

> **Navigation:** [◀ Phase 4 — System Buses and I/O](Phase_4_Buses_and_IO.md) · **Phase 5** · [Phase 6 — Virtualization Hardware ▶](Phase_6_Virtualization_Hardware.md)

---

> ### This phase is where three maps intersect.
>
> Here we cover the **hardware** of the network: how a packet is physically carried, how
> the NIC works with the CPU, what the components of latency are.
>
> The network's **protocols** (IP, TCP, DNS, TLS, routing) belong to the network map. This
> phase lays its physical ground: **to understand why a packet is delayed, you first need to
> know its physical journey.**

---

## Where we are coming from

The concepts of Phase 4 continue directly here:

| What you learned in Phase 4 | Where it continues here |
|---|---|
| 4.3 DMA | 5.1.3 — the NIC writes the packet directly to RAM |
| 4.4.2 interrupts/polling, NAPI | 5.1.4 — its full application in the network stack |
| 4.4.3 MSI-X distribution | 5.1.4 — distribution across cores with RSS |
| 4.1.2 shared line → point-to-point | 5.2.1 — the evolution from hub to switch, **exactly the same story** |
| 4.2 PCIe lane budget | 5.2.3 — the PCIe requirements of a 100 Gbps NIC |

And a concept from Phase 3 will show up here for the third time: **the queue curve.** On
disk it was the `await` explosion; here it will appear as **packet loss and jitter**.

---

## By the end of this phase

- You will be able to describe the path of a packet from arriving at the NIC to reaching the application
- You will be able to explain why bandwidth and latency are two independent quantities
- You will be able to tell apart the four components of latency and know which ones you can act on
- You will be able to diagnose "the link is 10 Gbps but my application gets 2 Gbps"
- You will be able to find where the bottleneck is in high-packet-rate workloads
- You will be able to read AWS's network performance figures (and EFA) with the reasoning behind them

---

## Phase map

| Section | Topic | Depth | Why it matters |
|---|---|---|---|
| 5.1 | NIC | `[mechanism]` | The packet's entry gate into the system; most bottlenecks live here |
| 5.2 | Switching and the physical network | `[mechanism]` / `[concept]` | The foundation of data center topology |
| 5.3 | **Bandwidth and latency** | `[mechanism]` | **The heart of the phase — the basis for architectural decisions** |
| 5.4 | RDMA and high-performance networking | `[concept]` | The reason EFA and ML clusters exist |

---

# 5.1 NIC — Network Interface Card

## 5.1.1 The NIC's job `[mechanism]`

The NIC (Network Interface Card) translates between the **bits** in the computer's memory and
the **electrical/optical signals** on the wire.

**When sending:**
```
1. The operating system prepares the packet in RAM
2. Tells the NIC "there's a packet at this address"
3. The NIC reads the packet from RAM via DMA     (Phase 4.3)
4. Wraps it in an Ethernet frame (header + CRC)
5. Converts the bits into a physical signal (encoding)
6. Drives it onto the wire
```

**When receiving:**
```
1. A signal arrives from the wire → converted into bits
2. The CRC is checked — if corrupt, the packet is dropped
3. The MAC address is checked — is it for me?
4. The NIC writes the packet to RAM via DMA      (Phase 4.3)
5. Sends the CPU an interrupt                    (Phase 4.4)
6. The kernel processes the packet, hands it to the application
```

> **Note step 4:** the packet is written to RAM **without passing through** the CPU. In Phase
> 4.3 we calculated that in a world without DMA, 10 Gbps would eat up cores — that calculation
> was describing exactly this spot.

## 5.1.2 The Ethernet frame and the MAC address `[mechanism]`

```
┌──────────┬──────────┬──────────┬──────┬─────────────────┬─────┐
│ Preamble │ Dest MAC │ Src MAC  │ Type │     Payload     │ CRC │
│  8 bytes │  6 bytes │  6 bytes │2 byte│  46–1500 bytes  │4 by.│
└──────────┴──────────┴──────────┴──────┴─────────────────┴─────┘
```

| Field | Function |
|---|---|
| **Preamble** | Clock synchronization for the receiver |
| **MAC addresses** | Who talks to whom on the local network — 48 bits, burned into the hardware |
| **Type** | The protocol above (0x0800 = IPv4, 0x86DD = IPv6, 0x0806 = ARP) |
| **Payload** | The data carried — **maximum 1500 bytes: the MTU** |
| **CRC** | Error detection — a corrupt frame is silently dropped |

**MTU (Maximum Transmission Unit) = 1500 bytes** is the standard, and it has an important
consequence:

```
Fixed overhead of every packet:
  Ethernet header+CRC : 18 bytes
  IP header           : 20 bytes
  TCP header          : 20 bytes
  ──────────────────────────────
  Total overhead      : 58 bytes

With a 1500-byte MTU:  1500 / 1558 = 96.3% efficiency
```

**Jumbo frames (MTU 9000):**
```
9000 / 9058 = 99.4% efficiency
And more importantly: the PACKET COUNT for the same data DROPS 6×
→ 6× fewer interrupts, 6× less header processing (the math from Phase 4.4.2)
```

> **Cloud connection:** Jumbo frames (9001 MTU) are supported inside an AWS VPC and make a
> noticeable difference for large data transfers. **But they can't be used for traffic going
> out to the internet** — if a device along the path is limited to 1500 MTU, the packet gets
> fragmented or dropped.
>
> A classic failure: after enabling jumbo frames inside a VPC, **some** connections break
> (the ones going through a gateway) while others work. The symptom is confusing because
> small packets (the SSH handshake) get through while large packets (a file transfer) stall —
> it looks like "the connection is established but no data flows".

## 5.1.3 The ring buffer `[mechanism]`

Data exchange between the NIC and the kernel happens through a **ring buffer**: a circular
array of descriptors allocated in RAM.

```
        ┌───┬───┬───┬───┬───┬───┬───┬───┐
   RX:  │ ▓ │ ▓ │ ▓ │   │   │   │   │   │
        └───┴───┴───┴───┴───┴───┴───┴───┘
          ↑           ↑
      CPU reads   NIC writes
      (consumer)  (producer)

  ▓ = packet waiting to be processed
```

**The critical point:** this buffer is **fixed in size.** If the NIC writes faster than the
CPU can consume, the buffer fills up and **newly arriving packets are dropped.**

And that drop is silent — it produces no application error.

```bash
ethtool -g eth0        # ring buffer size
ethtool -S eth0 | grep -i -E "drop|miss|overrun|no_buf"
```
**Expected output:**
```
Ring parameters for eth0:
Pre-set maximums:
RX:  4096          ← the largest the hardware supports
TX:  4096
Current hardware settings:
RX:  512           ← currently in use — can be enlarged
TX:  512
```
```
     rx_dropped: 0          ← should be 0
     rx_missed_errors: 0    ← if rising, the buffer isn't enough
```

> **If `rx_dropped` is rising, there is one of two causes:** either the buffer is small
> (enlarge it with `ethtool -G`), or the CPU can't keep up (see 5.1.4). **It's important not
> to confuse the two** — if the CPU can't keep up, enlarging the buffer only increases latency;
> it doesn't prevent loss.
>
> The same logic as the queue curve in Phase 3.4.4: **enlarging the queue doesn't solve the
> problem when the service rate is insufficient — it just makes things wait in a deeper
> queue.** This is also the essence of the phenomenon called bufferbloat.

## 5.1.4 Interrupts, NAPI and RSS `[mechanism]`

We met NAPI in Phase 4.4.2. In the network context, here's how it fully works:

```
Low traffic:
  Packet arrives → NIC raises an interrupt → CPU processes it → idles
  ✓ Low latency, the CPU can stay idle

High traffic:
  Packet arrives → NIC raises an interrupt
  → The kernel DISABLES INTERRUPTS, switches to polling mode
  → Processes multiple packets per round (budget: usually 64–300)
  → When the queue empties, re-enables interrupts
  ✓ The interrupt storm is avoided
```

**RSS (Receive Side Scaling) — the network version of Phase 4.4.3.**

Modern NICs have **multiple RX queues**. The NIC computes a hash from the packet's headers
(source/destination IP and port) and puts the packet into a queue based on that hash. Each
queue is bound to a different core.

```
Incoming packets
      ↓
   NIC hash(src_ip, dst_ip, src_port, dst_port)
      ↓
  ┌───┴───┬───────┬───────┐
 RX0     RX1     RX2     RX3
  ↓       ↓       ↓       ↓
CPU0    CPU1    CPU2    CPU3     ← parallel processing
```

**Why a hash, and not random?** Because **all packets of the same connection must go to the
same core.** Otherwise:
- Packets are processed out of order → TCP reordering cost
- Connection state is shared across cores → the **false sharing** and cache-line ping-pong
  from Phase 2.3.8

> **This is a direct consequence, at the network layer, of the cache consistency lesson from
> Phase 2.** Keeping the same connection on the same core isn't just for ordering, it's also
> needed for **cache locality**: that connection's socket structure, TCP state and buffers are
> already in that core's cache.

```bash
ls /sys/class/net/eth0/queues/     # number of queues
ethtool -l eth0                    # channel (queue) configuration
cat /proc/interrupts | grep eth0   # distribution (Phase 4.4.3)
```

**Other offload mechanisms:**

| Feature | What it does | The gain |
|---|---|---|
| **Checksum offload** | The NIC computes the checksum | CPU savings |
| **TSO/GSO** | The NIC splits a large block into segments | Lower CPU cost per packet |
| **LRO/GRO** | The NIC/kernel merges small incoming packets | Less work for upper layers |
| **RSS** | Spreads packets across cores | Parallel processing |

```bash
ethtool -k eth0 | head -20
```

> **The common idea: move work from the CPU to hardware.** A small-scale version of the Nitro
> logic from Phase 4.3.3. And the same principle will repeat for virtualization in Phase 6.

> **🤔 Think 5.1**
> On a server, `rx_dropped` in the `ethtool -S eth0` output is rising by thousands per second.
> You raised the ring buffer from 512 to 4096. Loss went down but **latency went up**, and the
> loss didn't stop completely.
>
> a) What happened? Why wasn't enlarging the buffer enough?
> b) What would you do, in order?
> *(Answer: at the end of the phase)*

---

# 5.2 Switching and the physical network

## 5.2.1 From hub to switch — a familiar story `[mechanism]`

**Hub (old):** copies the incoming signal to **all** ports.
```
  A ──┐
  B ──┼── HUB ── While A talks, B, C, D have to listen
  C ──┤          If two devices talk at once: COLLISION
  D ──┘          Total bandwidth is SHARED
```

**Switch (modern):** learns MAC addresses and sends the packet **only to the destination port.**
```
  A ──┐
  B ──┼── SWITCH ── A→B and C→D AT THE SAME TIME at full speed
  C ──┤             No collisions
  D ──┘             Every port has its own bandwidth
```

> **Exactly the same as the shared bus → PCIe transition in Phase 4.1.2.** Shared medium →
> contention → bottleneck; the fix is point-to-point switching.
>
> **This is the most frequently repeated pattern of this map**, and you've now seen it at four
> separate layers: channels in memory (2.4.3), queues in storage (3.3.2), PCIe lanes in the
> system (4.1.2), switch ports in the network. Every time: *many dedicated paths instead of one
> shared path.*

**How a switch learns its MAC table:**
```
1. It starts with an empty table
2. A frame arrives on a port → it records the SOURCE MAC against that port (learning)
3. If the destination MAC is in the table → sends only to that port
4. If the destination MAC isn't in the table → sends to ALL ports (flooding)
5. When the reply comes, that MAC is learned too
```

> This "if you don't know, ask everyone and learn from whoever answers" mechanism repeats in
> many parts of the network (ARP, DNS caching). The network map will open it up at the
> protocol level.

## 5.2.2 Full duplex and half duplex `[concept]`

| | Half duplex | Full duplex |
|---|---|---|
| Communication | Taking turns (either send or receive) | **Send and receive at the same time** |
| Collisions | Yes (requires CSMA/CD) | **None** |
| Effective capacity | 1 Gbps link = 1 Gbps total | 1 Gbps link = **1 Gbps in each direction** |
| Status today | Only old hub-based networks | **Standard** |

On modern switched networks every link is full duplex. It's the same design choice as the
bidirectionality of a PCIe lane (Phase 4.2.1) and of NVMe queues.

> **Practical note:** if an interface has unexpectedly fallen back to half duplex (an
> auto-negotiation failure), the symptom is **low throughput and a rising `collisions`
> counter.** It's a failure still seen on physical servers; you won't see it in the cloud.

## 5.2.3 Ethernet speeds `[concept]`

| Standard | Speed | Bytes/s | Typical place |
|---|---|---|---|
| 1 GbE | 1 Gbps | 125 MB/s | Old servers, office |
| 10 GbE | 10 Gbps | 1.25 GB/s | Standard server |
| 25 GbE | 25 Gbps | 3.1 GB/s | Modern data center |
| 100 GbE | 100 Gbps | 12.5 GB/s | Backbone, HPC/ML |
| 400 GbE | 400 Gbps | 50 GB/s | Large-scale backbone |

**Conversion rule:** `Gbps ÷ 8 = GB/s` — but in practice ~95% is usable because of protocol
overhead.

> **Connecting to Phase 4:** 100 Gbps = 12.5 GB/s. That **exceeds** PCIe Gen3 x8 (8 GB/s).
> So a 100 Gbps NIC needs Gen4 x8 or Gen3 x16. **The capacity of the slot it's plugged into is
> just as decisive as the NIC's own speed** (the lesson of Phase 4.2.2).

**Cloud connection:** the network performance of AWS instances scales with size — and that
isn't an arbitrary commercial decision, it's a continuation of the logic from Phase 2.4.3:
**a larger instance gets a larger slice of the physical server, and therefore a larger share
of the NIC's capacity too.**

| Instance class | Typical network performance |
|---|---|
| `.large` | "Up to 10 Gigabit" — **burst, not guaranteed** |
| `.4xlarge` | "Up to 25 Gigabit" |
| `.12xlarge` | 25 Gbps (guaranteed) |
| `.24xlarge` / `.metal` | 50–100 Gbps (guaranteed) |

> **The phrase "up to" is a warning.** On small instances, network performance works on a
> **credit-based burst** model (the same idea as the t-family CPU credits in Phase 1.5.4). Put
> a workload that needs sustained high traffic on an "up to 10 Gigabit" instance and it runs
> perfectly for the first few minutes, then at the baseline level.
>
> This is a very common mistake in production: the load test runs for 5 minutes and gives
> great results; the real load runs for 3 hours and collapses at minute 20.

![Figure 5.1 — The shared medium of a hub versus the point-to-point switching of a switch](../diagrams/png/hw-5-01-hub-vs-switch.png)
*Figure 5.1 — On the left, devices sharing a collision domain; on the right, simultaneous
full-speed communication.*

---

# 5.3 Bandwidth and latency

**This section is the heart of the phase.** Confusing bandwidth with latency is one of the
most expensive mistakes made in cloud architecture.

## 5.3.1 Two independent quantities `[mechanism]`

| | Bandwidth | Latency |
|---|---|---|
| What it measures | Data carried per unit of time | How long a single packet takes to arrive |
| Unit | Gbps, MB/s | ms, μs |
| What limits it | The link's capacity | **Distance and the number of devices in between** |
| Increasing it | Possible (thicker link, parallel connections) | **Physical limit — the speed of light** |

**The classic analogy:** a truckload of disks has higher **bandwidth** than the
intercontinental internet (it carries petabytes), but its **latency** is measured in days.

```
Bandwidth CAN BE INCREASED.
Latency is BOUNDED by the speed of light.
```

**In numbers:**
```
Speed of light (in fiber) ≈ 200,000 km/s
Istanbul → Virginia ≈ 8,000 km (the fiber path is longer, ~10,000 km)

One-way minimum  : 10,000 / 200,000 = 50 ms
Round trip (RTT) : ~100 ms   ← PHYSICAL LOWER BOUND
```

> **No amount of money and no technology can lower this.** That's why CDNs and edge locations
> exist: you can't make data faster, but you **can bring it closer.**
>
> And that's why synchronous call chains are a disaster in multi-region architectures: every
> call adds a 100 ms floor. A five-call chain = 500 ms, before any computation happens.

## 5.3.2 The four components of latency `[mechanism]`

**This breakdown is the key to diagnosing network latency.** Each component is addressed
differently — and some can't be addressed at all.

```
Total latency = Propagation + Transmission + Processing + Queuing
```

| Component | What it is | What it depends on | Intervention |
|---|---|---|---|
| **Propagation** | The signal covering the distance | **Distance** | ❌ Only get closer (CDN, edge) |
| **Transmission** | Time to drive the packet onto the link | Packet size ÷ link speed | ✅ Faster link |
| **Processing** | Devices reading headers | Number and power of devices | ⚠️ Fewer hops, better hardware |
| **Queuing** | Waiting in line | **Load and congestion** | ✅ **The most actionable** |

**Calculate transmission delay:**
```
1500-byte packet, 1 Gbps link:
  (1500 × 8) bits ÷ 1,000,000,000 bits/s = 12 μs

Same packet, 10 Gbps link:
  1.2 μs
```

> **Careful:** making the link 10× faster cut transmission delay by 10× — but within total
> latency this component was already in the microsecond range. On an intercontinental link
> where propagation is 50,000 μs, cutting transmission from 12 μs to 1.2 μs **changes nothing.**
>
> **This is the direct answer to the complaint "we increased bandwidth but the application
> didn't get faster."**

**Queuing delay — the third repetition of Phase 3.4.4:**

As a link approaches saturation, queuing delay rises exponentially:
```
Link utilization 50% →  queuing delay negligible
Link utilization 80% →  noticeable
Link utilization 95% →  the dominant component
Link utilization 99% →  packet loss and jitter
```

> **On disk it was the `await` explosion (3.4.4); here it's jitter and packet loss.** The same
> queuing theory, a third layer. And the same conclusion: **don't aim to fill a link up to
> 100%.**

## 5.3.3 The bandwidth–delay product `[application]`

**This concept is the most common cause of "the link is fast but my transfer is slow".**

```
BDP (Bandwidth-Delay Product) = Bandwidth × RTT
```

The BDP is **the amount of data "in flight" on the link at any moment.** TCP needs to be able
to send that much data without waiting for acknowledgment.

**Example:**
```
10 Gbps link, 100 ms RTT (intercontinental):
BDP = 1.25 GB/s × 0.1 s = 125 MB

→ The TCP window size must be 125 MB for the link to fill up
```

**But default TCP windows are much smaller** (usually 64 KB – a few MB).

```
Maximum achievable with a 64 KB window at 100 ms RTT:
  64 KB ÷ 0.1 s = 640 KB/s = 5.1 Mbps

You're getting 5 Mbps out of a 10 Gbps link — 0.05% of the link.
```

> **This is the answer to the complaint "I have a 10 Gbps link but my file transfer runs at 5
> MB/s."** The problem isn't the link, it's **the TCP window.**
>
> **Fixes:**
> - TCP window scaling — enabled on modern systems
> - Enlarge the `net.ipv4.tcp_rmem` / `tcp_wmem` buffers
> - **Use parallel streams** — 10 parallel TCP connections means 10× the window
>   (that's why `aws s3 cp` and download accelerators open multiple connections)
> - A better congestion control algorithm for long distances (BBR)
>
> **And this explains why long-distance data transfer is its own engineering discipline.** The
> network map will go into the TCP side in detail.

## 5.3.4 Real-world latency figures `[application]`

The network rungs of the ladder from Phase 2.1.1:

| Path | RTT |
|---|---|
| Within the same server (loopback) | ~0.02 ms |
| Same rack, same switch | ~0.1 ms |
| Same AZ (data center) | **~0.3–0.5 ms** |
| Between AZs in the same region | **~1–2 ms** |
| Between regions on the same continent | ~20–40 ms |
| Intercontinental | **~100–200 ms** |

**Architectural consequences:**

| Decision | Latency impact |
|---|---|
| Keep microservices in the same AZ | ~0.4 ms per call |
| Multi-AZ deployment | ~1.5 ms per call — **the price paid for resilience** |
| Synchronous call chain (5 services, multi-AZ) | ~7.5 ms of network alone |
| Intercontinental synchronous call | ~100 ms — **almost always the wrong design** |

> **Multi-AZ isn't free.** ~1.5 ms between AZs looks trivial for a single call. But in a
> 20-call request-handling chain it adds up to 30 ms — and most APIs have a 100 ms budget.
>
> **This is where the rule "make everything multi-AZ" deserves to be questioned:** resilience
> is a real gain, but the latency budget is real too. The architectural decision is to weigh
> the two knowingly — not by reflex.

> **🤔 Think 5.2**
> A team connects from an application server in Frankfurt to a database in Virginia. Page load
> time is 4 seconds. The team says "let's increase bandwidth".
>
> a) Would increasing bandwidth help? Why?
> b) Estimate where the 4 seconds come from (do the math).
> c) Propose three different solutions.
> *(Answer: at the end of the phase)*

![Figure 5.2 — The four components of latency and the room for intervention in each](../diagrams/png/hw-5-02-latency-components.png)
*Figure 5.2 — The share of propagation, transmission, processing and queuing delay in a
packet's journey.*

---

# 5.4 RDMA and high-performance networking `[concept]`

## 5.4.1 The cost of the normal network stack

The path a packet takes to reach the application:

```
NIC → write to RAM via DMA       (Phase 4.3)
    → interrupt                   (Phase 4.4)
    → kernel network stack (IP, TCP processing)
    → COPY from kernel buffer to user buffer
    → application
```

Every step adds latency and CPU cost. Especially the last two:
- **Kernel/user transition** — a context switch (the costs from Phase 4.4.1)
- **Memory copy** — we know from Phase 2: it consumes memory bandwidth and pollutes the cache

**Total overhead: ~10–50 μs.** Trivial for most applications. **Decisive** in HPC and ML
training.

## 5.4.2 RDMA — bypassing the kernel

**RDMA (Remote Direct Memory Access):** one machine's NIC writes **directly into the memory**
of another machine. The remote machine's CPU and kernel are not involved.

```
Normal:  App → kernel → NIC → network → NIC → kernel → App
RDMA:    App → NIC → network → NIC → remote RAM (directly)
              ↑ kernel bypassed, no copies (zero-copy)
```

| | Normal TCP | RDMA |
|---|---|---|
| Latency | ~10–50 μs | **~1–3 μs** |
| CPU cost | High | **Nearly zero** |
| Number of copies | 2+ | **0** |

> **DMA (Phase 4.3) extended across the network.** DMA was a device bypassing the CPU; RDMA
> bypasses the remote machine's CPU.

**Implementations:**
- **InfiniBand** — dedicated network hardware, the standard for HPC clusters
- **RoCE** (RDMA over Converged Ethernet) — RDMA over regular Ethernet

## 5.4.3 Cloud connection — AWS EFA `[application]`

**EFA (Elastic Fabric Adapter)** is AWS's network interface offering RDMA-like capability.

**Why does it exist?** In distributed ML training the GPUs synchronize gradients at every step
(all-reduce). That means **a large number of small, latency-sensitive messages.**

```
Training on 128 GPUs, thousands of synchronization rounds per second

Normal network : ~50 μs per round × thousands of rounds → GPUs keep waiting
With EFA       : ~5 μs per round                        → GPUs compute
```

**When you need it:**

| Workload | EFA |
|---|---|
| Web application, API | ❌ Unnecessary — the latency budget is in milliseconds |
| Database | ❌ Unnecessary |
| Single-node ML training | ❌ Unnecessary — no network involved |
| **Multi-node ML training** | ✅ **Decisive** |
| **HPC simulation (MPI)** | ✅ **Decisive** |

> **EFA is the answer to the "scaling efficiency" problem.** When you go from one node with 8
> GPUs to 8 nodes with 64 GPUs, speed should rise 8×. With a normal network it might rise 4× —
> because the GPUs wait for synchronization. **You pay full price for GPUs you aren't using.**
>
> The lesson of Phase 4.2.4 repeated over the network: **the bottleneck isn't in the most
> expensive component, but in the path that feeds it.**

---

# 5.5 When This Phase Breaks — Failure Signatures

| Symptom | Likely mechanism | Where | First look |
|---|---|---|---|
| Link is 10 Gbps but the transfer runs at 5 MB/s | **TCP window / BDP** | 5.3.3 | `ss -i`, tcp_rmem, parallel streams |
| `rx_dropped` rising | Ring buffer, or the CPU can't keep up | 5.1.3 | `ethtool -S`, `-g`, `mpstat` |
| One core at 100% at high pps | RSS not spreading the load | 5.1.4 | `/proc/interrupts`, `ethtool -l` |
| Fast for the first few minutes, then slow | **Network burst credits exhausted** | 5.2.3 | Instance network class, the "up to" wording |
| High latency, bandwidth idle | Propagation delay (distance) | 5.3.2 | Region/AZ placement, CDN |
| Fluctuating latency (jitter) | **Queuing delay — the link is saturated** | 5.3.2 | Link utilization, congestion |
| Some connections break, others work | **MTU mismatch** | 5.1.2 | `ping -M do -s 1472`, path MTU |
| p99 went up after moving to multi-AZ | Inter-AZ RTT × number of calls | 5.3.4 | Call chain depth |
| Multi-node training doesn't scale | Synchronization latency | 5.4.3 | EFA, node placement (placement group) |
| Low throughput, `collisions` rising | Fell back to half duplex (physical network) | 5.2.2 | The duplex line of `ethtool eth0` |

> **🔧 Diagnosing an MTU problem** (optional)
> ```bash
> # 1472 = 1500 - 28 (IP+ICMP headers). Fragmentation forbidden (-M do)
> ping -M do -s 1472 <target>
> ```
> **Expected output (healthy):**
> ```
> 1480 bytes from 10.0.1.5: icmp_seq=1 ttl=64 time=0.412 ms
> ```
> **If there is a problem:**
> ```
> ping: local error: message too long, mtu=1500
> ```
> or no reply at all. Reduce the size (1400, 1300...) to find the largest value that gets
> through — that is the path MTU.

---

# Phase 5 — Answers to the Think Questions

## Answer 5.1

**a) What happened?**

`rx_dropped` means that when a packet arrived at the ring buffer, **there was no free slot in
the buffer.** This has two separate causes, and their treatments differ:

| Cause | Mechanism | The right treatment |
|---|---|---|
| Buffer too small | Short bursts overflow the buffer | Enlarge with `ethtool -G` ✅ |
| CPU can't keep up | Consumption rate < arrival rate | **Enlarging the buffer doesn't fix it** ❌ |

You enlarged the buffer, and loss **went down but didn't stop.** That tells you exactly this:
**both causes were present.** Bursts are now absorbed (that part got fixed), but the average
arrival rate still exceeds the average consumption rate — that part remains.

**Why did latency go up?** Because the queue got deeper. A packet now waits in a longer line
before being processed.

```
Little's Law:  Wait time = Queue length ÷ Service rate

Service rate unchanged, queue 8× longer → wait time rises up to 8×
```

> **This is the same as the queue lesson in Phase 3.4.4, and on networks it's called
> "bufferbloat":** enlarging the buffer when the service rate is insufficient **doesn't solve
> the problem, it only hides it and turns it into latency.** Packets arrive late instead of
> being dropped — and for some workloads that's worse (TCP uses packet loss as a congestion
> signal; if it sees delay instead of loss, it behaves incorrectly).

**b) What would you do, in order?**

**1. Confirm the CPU really is the bottleneck:**
```bash
mpstat -P ALL 1
```
If one core is near 100% in the `%soft` (softirq) column, network processing is jammed on that
core.

**2. Check whether RSS is spreading the load:**
```bash
cat /proc/interrupts | grep eth0
ethtool -l eth0
```
If there is only one queue, or all interrupts go to a single core, **this is the real
problem.** Increase the queue count and spread the interrupts (Phase 4.4.3).

**3. Enable offloads:**
```bash
ethtool -k eth0 | grep -E "gro|gso|tso|rx-checksum"
```
If GRO is off, the kernel is processing every small packet individually. Turning it on
seriously lowers the per-packet cost.

**4. Tune interrupt coalescing:**
```bash
ethtool -c eth0
```
Delaying interrupts slightly and processing them in batches relieves the CPU at high pps — at
the cost of latency. **This is a trade-off, not a fix.**

**5. If none of these is enough:** the workload has exceeded that instance's network capacity.
A larger instance or horizontal scaling is needed.

> **The essence of the lesson:** `rx_dropped` is a symptom. **The reflex to enlarge the buffer
> is an intervention made without diagnosing the bottleneck** — the same mistake as the "let's
> add RAM" reflex in Phase 2 and the "let's buy a faster disk" reflex in Phase 3.

---

## Answer 5.2

**a) Would increasing bandwidth help?**

**No** — and the reason is in 5.3.1.

Frankfurt → Virginia ≈ 6,500 km in a straight line, ~8,000 km along the fiber path:
```
One way : 8,000 / 200,000 = 40 ms
RTT     : ~80–90 ms (in practice ~90 ms is measured)
```

This is **propagation delay**, and it's completely independent of bandwidth. Even if you raise
the link from 1 Gbps to 100 Gbps, the 90 ms stays the same. You can't buy the speed of light.

**b) Where do the 4 seconds come from?**

Think about how many times a typical page load talks to the database. If the queries are
sequential (synchronous):

```
Establishing the connection:
  TCP handshake (3-way)       : 1 RTT  =  90 ms
  TLS handshake (TLS 1.2)     : 2 RTT  = 180 ms
  Database auth               : 1 RTT  =  90 ms
                                        ────────
                                          360 ms

Then 40 sequential queries × 90 ms = 3600 ms

TOTAL ≈ 3.96 s   ← 4 seconds
```

**The number of queries is small, but each one waits one RTT.** This is the classic **N+1
query problem** — a pattern that looks harmless on its own turns into a disaster when the RTT
goes from 0.3 ms to 90 ms.

> **The key insight:** if the same code ran in the same AZ (0.4 ms RTT):
> `40 × 0.4 = 16 ms`. The code didn't change, **the physics did.**
>
> **That's why an application that is fast in a local development environment can collapse in
> production** — on localhost the RTT is 0.02 ms, 40 queries add up to 0.8 ms, and nobody
> notices.

**c) Three solutions:**

| # | Solution | What it does | Gain |
|---|---|---|---|
| **1** | **Move the database into the same region as the application** | RTT 90 ms → 0.4 ms | **~4 s → ~30 ms** — the biggest gain |
| **2** | **Reduce the number of queries** (JOIN, batching, remove N+1) | 40 queries → 2 queries | 3600 ms → 180 ms |
| **3** | **Add a read replica** (in Frankfurt) | Reads served locally | Reads get faster, writes stay slow |

**Additional solutions:**
- **Connection pooling** — so you don't pay the 360 ms setup cost on every request
- **Caching** (Redis/ElastiCache, in the same region as the application)
- **TLS 1.3** — cuts the handshake from 2 RTT to 1 RTT; 0 RTT with session resumption

> **The order of priority matters:** number 1 solves the problem at the root. Number 2 touches
> code but pays off at any distance. **Increasing bandwidth isn't on the list — because it
> doesn't solve the problem; it doesn't solve anything.**
>
> **The real lesson of this question:** when you hear a "slow" complaint, the first question to
> ask should be **"is it bandwidth or latency?"** They are different diseases, and the medicine
> for one does nothing for the other.

---

# Phase 5 — Frequently Asked Questions

> **Q1: "My network card is 10 Gbps but `iperf3` shows 9.4 Gbps. Is the card broken?"**
>
> No, **this is the expected and correct result.**
>
> The theoretical maximum of a 10 Gbps link is 1250 MB/s. But every packet carries protocol
> overhead (5.1.2):
> ```
> Efficiency at 1500-byte MTU: 1500 / 1538 ≈ 97.5%
> 10 Gbps × 0.975 ≈ 9.75 Gbps (theoretical ceiling)
> Measured in practice: 9.4 Gbps  ← 94%, healthy
> ```
> With jumbo frames (MTU 9000) you can see 9.8+ Gbps.
>
> **Alarm threshold:** if it falls below 85%, there is a problem — CPU, offload settings or
> queue distribution. **But 94% is proof the card is working properly.**

> **Q2: "Why is `ping` very low but my application slow?"**
>
> Because `ping` measures the round trip of **a single small packet**. An application's
> slowness usually comes from one of these:
>
> | Possible cause | How to tell |
> |---|---|
> | Many sequential calls (N+1) | Is number of calls × RTT ≈ total time? |
> | Insufficient TCP window (BDP, 5.3.3) | Large transfers slow, small requests fast |
> | Server-side processing time | Not in the network, in the application profile |
> | Queuing delay (link saturated) | Does `ping` fluctuate? Is there jitter? |
>
> **If `ping` is low, the *propagation* side of the network is healthy.** The other three
> components can still be the culprit (5.3.2).

> **Q3: "Why not just enable jumbo frames everywhere?"**
>
> **You can't, and enabling them might give you a very bad day.**
>
> For jumbo frames to work, **EVERY device along the path** must support the same MTU. If a
> single device is limited to 1500:
> - The packet gets fragmented (performance loss), or
> - If the DF (Don't Fragment) flag is set, it's **silently dropped**
>
> **Safe area of use:** inside the same VPC, in the same region, internal traffic only. **Don't
> enable them** for traffic going to the internet or over VPN/Direct Connect.
>
> Recognize the symptom: small packets get through (SSH connects), large packets stall (the
> file transfer freezes). That is almost always the MTU.

> **Q4: "How reliable is 'Up to 10 Gigabit' on AWS?"**
>
> **Not reliable for sustained load.** On small instances, network performance is
> credit-based:
> ```
> Idle          → credits accumulate
> Burst         → you can go up to 10 Gbps
> Credits run out → you drop to the baseline (typically 0.5–2 Gbps)
> ```
> It's the same model as the t-family CPU credits (Phase 1.5.4).
>
> **How to decide:**
> - Short-lived, intermittent high traffic → "Up to" is fine
> - **Sustained high traffic → move to a size with a guaranteed figure** (`.12xlarge` and up)
>
> **The trap:** a 5-minute load test finishes before the credits run out and gives great
> results. **Run load tests for at least 30 minutes.**

> **Q5: "What is a placement group for, and does it really make a difference?"**
>
> A **cluster placement group** places instances physically close to each other: in the same
> rack or neighboring racks.
>
> | | Normal placement | Cluster placement group |
> |---|---|---|
> | RTT within the AZ | ~0.3–0.5 ms | **~0.05–0.1 ms** |
> | Hop count | Several switches | Usually a single switch |
>
> **When it makes a difference:**
> - ✅ Multi-node ML training, HPC/MPI — **decisive**
> - ✅ High-frequency inter-node communication
> - ❌ Web applications — 0.3 ms is already within budget
>
> **The price:** because all the instances are packed into the same physical area, **the risk
> of correlated failure rises.** If you want resilience, a spread placement group (the exact
> opposite) is used. The classic performance–resilience trade-off.

> **Q6: "What's the difference between RSS and RPS?"**
>
> | | RSS | RPS |
> |---|---|---|
> | Where | **In hardware** (NIC) | **In software** (kernel) |
> | Requirement | Multi-queue NIC | Any NIC |
> | Cost | Zero (the NIC does it) | Uses CPU |
>
> **If you have RSS, use it.** RPS is a patch for old single-queue NICs: the packet has already
> landed on a single core, and the kernel redistributes it to another core in software. It
> works, but it isn't as efficient as RSS.
>
> All modern cloud instances (ENA) have RSS.

> **Q7: "SR-IOV, DPDK, kernel bypass — are these the same thing?"**
>
> No, but **they serve the same goal: removing the layers in between.**
>
> | Technology | What it bypasses | Where it's covered |
> |---|---|---|
> | **SR-IOV** | The hypervisor's virtual switch | **Phase 6** |
> | **DPDK** | The kernel network stack (driver in user space) | 5.4.1 in this phase |
> | **RDMA/EFA** | Both the kernel and the remote CPU | 5.4.2 |
>
> The common idea is the same as the Nitro logic in Phase 4.3.3: **every intermediate layer adds
> latency and CPU cost; if performance is critical, remove layers.** The price is always a loss
> of flexibility and abstraction.

---

# Phase 5 — Test Yourself

## Part A — Fundamentals

**A1.** What is the MTU in an Ethernet frame, and what is its standard value?

**A2.** What is the fundamental difference between a hub and a switch?

**A3.** Distinguish bandwidth from latency in one sentence.

**A4.** What does full duplex mean?

**A5.** What does DMA provide in the context of a NIC?

**A6.** What is the job of RSS?

**A7.** How many GB/s is 25 Gbps?

## Part B — Mechanism

**B1.** List the steps a packet goes through from the wire until it reaches the application.

**B2.** Name the four components of latency and state what each one depends on.

**B3.** Why does NAPI exist? What problem does it solve?

**B4.** Why does RSS use a hash instead of random distribution? Give two reasons.

**B5.** What happens when the ring buffer fills up? Is enlarging it always the fix?

**B6.** What is the BDP, and why is it related to the TCP window size?

**B7.** Jumbo frames provide two separate gains — explain both.

## Part C — Application and reasoning

**C1.** 10 Gbps link, 80 ms RTT. What is the BDP? What is the maximum throughput achievable
with a 256 KB window?

**C2.** On a server, `mpstat` shows CPU0 at 98% in the `%soft` column, while the other 15 cores
are idle. Network throughput is stuck at 3 Gbps on a 25 Gbps link. Your diagnosis and fix?

**C3.** A team moved to a microservice architecture. Every request makes 12 service calls, all
synchronous. The services are deployed multi-AZ. The p99 latency target is 100 ms. Can this
target be met? Calculate and interpret.

**C4.** You're going to buy a 100 Gbps NIC. The server has a free PCIe Gen3 x8 slot. What
happens?

**C5.** A file transfer runs at 900 MB/s within the same AZ and 4 MB/s intercontinentally. The
link is 10 Gbps in both cases. What is the cause, and how would you confirm it?

**C6.** A team says "the network is slow". You have this data: `ping` 0.4 ms and stable, link
utilization 30%, `rx_dropped` 0, application response time 800 ms. Is the network to blame?
How would you proceed?

**C7.** An ML team went from a single node with 8 GPUs to 4 nodes × 8 GPUs = 32 GPUs. Instead of
the expected 4× speedup they got 2.3×. Likely cause and fix?

---

## Answer Key

### Part A

**A1.** The MTU (Maximum Transmission Unit) is the maximum payload size a frame can carry. On
standard Ethernet it's **1500 bytes**. *(5.1.2)*

**A2.** A hub copies the incoming signal to all ports — bandwidth is shared and collisions
occur. A switch uses its MAC table to send only to the destination port — every port has its
own bandwidth and there are no collisions. *(5.2.1)*

**A3.** Bandwidth is the amount of data carried per unit of time (it can be increased); latency
is the time it takes a single packet to arrive (it is bounded by the speed of light). *(5.3.1)*

**A4.** Sending and receiving at the same time — each direction has its own full capacity.
*(5.2.2)*

**A5.** The NIC writes an incoming packet directly to RAM without involving the CPU, and reads
an outgoing packet directly from RAM. The CPU only kicks off the work and gets notified when
it's done. *(5.1.1, Phase 4.3)*

**A6.** Spreading incoming packets across different RX queues — and therefore different cores —
based on a header hash, parallelizing network processing. *(5.1.4)*

**A7.** 25 ÷ 8 = **3.125 GB/s**. *(5.2.3)*

### Part B

**B1.** *(5.1.1)*
```
1. Signal from the wire → bits
2. CRC check (drop if corrupt)
3. MAC check (is it for me?)
4. Write to the ring buffer in RAM via DMA
5. Interrupt (or NAPI poll)
6. Kernel network stack: IP, TCP processing
7. Copy from kernel buffer to user buffer
8. Application
```

**B2.** *(5.3.2)*

| Component | What it depends on |
|---|---|
| Propagation | Distance (speed of light) |
| Transmission | Packet size ÷ link speed |
| Processing | Number and power of devices |
| Queuing | Load and congestion |

**B3.** At high packet rates, a separate interrupt per packet exhausts the CPU (an interrupt
storm). When traffic rises, NAPI disables interrupts, switches to polling mode and processes
many packets per round. At low traffic it goes back to interrupts — providing both low latency
and high throughput. *(5.1.4, Phase 4.4.2)*

**B4.** *(5.1.4)*
1. **Ordering:** packets of the same connection must go to the same core; otherwise TCP
   reordering costs arise.
2. **Cache locality:** that connection's socket structure and TCP state are already in that
   core's cache; going to a different core creates false sharing and cache-line ping-pong
   *(Phase 2.3.8)*.

**B5.** Newly arriving packets are **silently dropped** (`rx_dropped` rises). Enlarging it is
**not always the fix**: if the CPU's consumption rate is lower than the arrival rate, a bigger
buffer only increases queuing delay — it delays loss rather than preventing it (bufferbloat).
*(5.1.3, Answer 5.1)*

**B6.** BDP = Bandwidth × RTT. It's the amount of data "in flight" on the link at any moment.
TCP can send at most a window's worth of data without waiting for acknowledgment; if the window
is smaller than the BDP, the link **never fills up** and throughput is capped by the window.
*(5.3.3)*

**B7.** *(5.1.2)*
1. **Higher efficiency:** the fixed 58 bytes of overhead is spread over a larger payload
   (96.3% → 99.4%).
2. **Fewer packets:** the same data fits into 6× fewer packets → 6× fewer interrupts and less
   header processing. **The second is usually more important.**

### Part C

**C1.** *(5.3.3)*
```
BDP = 1.25 GB/s × 0.08 s = 100 MB

Maximum with a 256 KB window:
  262,144 bytes ÷ 0.08 s = 3,276,800 B/s ≈ 3.3 MB/s ≈ 26 Mbps

0.26% of the link.
```
Fix: window scaling, larger buffers, parallel streams or BBR.

**C2.** *(5.1.4, Answer 5.1)*

**Diagnosis:** network processing is jammed on a single core — RSS is either absent or not
configured. 3 Gbps is the ceiling of a single core's softirq capacity; the link is idle but the
CPU is full.

**Confirmation:**
```bash
ethtool -l eth0                    # how many queues?
cat /proc/interrupts | grep eth0   # all on CPU0?
```

**Fix:**
1. Increase the queue count with `ethtool -L eth0 combined 16`
2. Spread interrupts across cores (`irqbalance` or manual `smp_affinity`)
3. If RSS isn't supported, enable RPS
4. Verify that the GRO/TSO offloads are on

**C3.** *(5.3.4)*
```
Inter-AZ RTT ≈ 1.5 ms
12 synchronous calls × 1.5 ms = 18 ms   ← network alone, at p50

At p99 each call's queuing delay also grows. Roughly 3–5×:
12 × 1.5 × 4 ≈ 72 ms   ← network alone
```
**Assessment:** **72% of the 100 ms target goes to the network**, leaving 28 ms for application
processing and the database. **Technically possible but extremely fragile** — a single slow
service blows the target.

**What needs to be done:**
- **Parallelize** the calls (call independent ones concurrently) → ~1.5 ms instead of 12 × 1.5
- Shorten the chain (merge services)
- Keep services on the critical path **in the same AZ** (provide resilience at another layer)

> **Note:** the real lesson here is that the problem isn't the number of microservices, but the
> **depth of the synchronous chain**.

**C4.** *(5.2.3, Phase 4.2.2)*
```
100 Gbps     = 12.5 GB/s needed
PCIe Gen3 x8 =  ~7.9 GB/s capacity
```
The NIC **gets stuck at ~63 Gbps** — 63% of 100 Gbps. The slot becomes the bottleneck.

**What's needed:** PCIe Gen4 x8 (~15.8 GB/s) or Gen3 x16 (~15.8 GB/s). The slot's `LnkCap`
value should have been checked with `lspci -vv` before buying.

> The lesson of Phase 4.2.4: **the most expensive component is only as fast as the path that
> feeds it.**

**C5.** *(5.3.3)*

**Cause:** the BDP. RTT is ~0.4 ms within the same AZ and ~150 ms intercontinentally.
```
Same AZ         : 256 KB / 0.0004 s = 640 MB/s  → the window isn't limiting
Intercontinental: 256 KB / 0.15 s   = 1.7 MB/s  → the window is limiting
```
The link is idle in both cases; what limits it is **the TCP window.**

**Confirmation:**
```bash
ss -i                    # cwnd, rtt, send buffer values
iperf3 -c <target> -P 10 # 10 parallel streams
```
If the total speed rises **~10×** with parallel streams, the diagnosis is confirmed — the single
stream's window was the bottleneck, not the link.

**C6.** *(5.3.2, Q2)*

**The network is not to blame.** The evidence:

| Data | What it says |
|---|---|
| `ping` 0.4 ms, stable | Propagation normal, no jitter |
| Link utilization 30% | No queuing delay |
| `rx_dropped` 0 | No buffer/CPU problem |

**How to proceed:**
1. **Count the calls** — 800 ms ÷ 0.4 ms = 2000 calls? (the N+1 problem)
2. If not, **profile the application** — where is the time going?
3. Database query times (slow query log)
4. Disk I/O (`iostat -x`, Phase 3.4.3)
5. CPU (`mpstat`, run queue)

> **Lesson:** when a team says "the network is slow", they often mean "we don't know". **Ruling
> out the network points the search in the right direction** — and these three measurements
> (ping, utilization, drops) are enough to rule it out.

**C7.** *(5.4.3)*

**Cause:** gradient synchronization (all-reduce). On a single node the GPUs talked over NVLink
(hundreds of GB/s, sub-microsecond latency). Across four nodes they now talk **over the
network** — both bandwidth and latency are many times worse.

```
Scaling efficiency = 2.3 / 4 = 57.5%
→ roughly half of the money spent on GPUs goes to waiting for synchronization
```

**Fix:**
1. **Use EFA** — cuts inter-node latency from ~50 μs to ~5 μs *(5.4.3)*
2. **Cluster placement group** — bring the nodes physically closer *(Q5)*
3. **Gradient accumulation** — reduce synchronization frequency
4. Use an instance size with guaranteed network performance *(5.2.3)*
5. Mixed precision — shrink the size of the gradients

> The same lesson as Phase 4.2.4, at a different scale: **the bottleneck isn't in the most
> expensive component, but in the path that feeds it.** Here the feeding path is the network.

---

## Scoring

| Correct answers | Assessment |
|---|---|
| 18–21 | You've finished the phase. Move on to Phase 6. |
| 14–17 | Good. Re-read the sections of the ones you got wrong, then move on to Phase 6. |
| 10–13 | Re-read 5.3 (bandwidth/latency) from the start — that's the heart of the phase. |
| 0–9 | Work through the phase again. Especially 5.1.4, 5.3.2 and 5.3.3. |

> **If you missed even one question in Part C, solve that question again.** Part C questions
> are real failure scenarios; in Phase 7 you'll run into combinations of them.

---

# Phase 5 — Closing and Bridge to Phase 6

## What you carry out of this phase

| Concept | The essence |
|---|---|
| **Bandwidth ≠ latency** | One can be increased, the other is bounded by physics |
| **The four components of latency** | Which one can be acted on, which one can't |
| **BDP** | The answer to the "fast link, slow transfer" mystery |
| **The queue curve (3rd time)** | `await` in storage, jitter and packet loss in the network |
| **From shared to switched (4th time)** | Hub → switch, the network version of the same evolution |
| **The offload idea** | Move work from the CPU to hardware — a small-scale Nitro |
| **The bottleneck is in the feeding path, not the fed component** | In PCIe, in the network, in a GPU cluster alike |

## Where Phase 6 connects to this

| What you learned in Phase 5 | How it will show up in Phase 6 |
|---|---|
| 5.1.1 the NIC writing to RAM via DMA | Which RAM, for a virtual machine? — **EPT/NPT** and address translation |
| 5.1.4 RSS and multi-queue NICs | **SR-IOV** — each VM gets its own virtual NIC queue |
| 5.4 the idea of removing layers | The same idea in virtualization: **paravirtualization → SR-IOV** |
| The "Up to 10 Gigabit" burst model | vCPU scheduling, **steal time** and credit models |
| The latency budget | The latency the hypervisor adds, and why Nitro brings it to nearly zero |
| 5.2.3 instance size ↔ capacity share | Carving up the physical server — **the root of noisy neighbors** |

> **In Phase 6 the question will be this:** so far we've examined hardware as if a single
> operating system owned it all by itself. **But that's not how it is in the cloud.** The same
> physical CPU, RAM, disk and NIC are carved up among dozens of customers.
>
> What hardware makes that carving-up possible, what does it cost, and where does it leak?

---

*Phase 5 complete.* → **[Phase 6 — Virtualization Hardware](Phase_6_Virtualization_Hardware.md)**
