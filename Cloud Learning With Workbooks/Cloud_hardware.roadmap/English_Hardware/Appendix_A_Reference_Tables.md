# Appendix A — Reference Tables

> **Navigation:** [◀ README](README_en.md) · **Appendix A** · [Appendix B — Glossary ▶](Appendix_B_Glossary.md)

---

> **This appendix exists to remind, not to teach.** This is the page you keep open after
> you have finished the map. Next to every table is the section it comes from — when a
> number stops making sense, go back there.

**Contents**
- [A.1 The latency ladder](#a1-the-latency-ladder)
- [A.2 Units and conversions](#a2-units-and-conversions)
- [A.3 Memory reference](#a3-memory-reference)
- [A.4 Storage reference](#a4-storage-reference)
- [A.5 Bus and PCIe reference](#a5-bus-and-pcie-reference)
- [A.6 Network reference](#a6-network-reference)
- [A.7 Virtualization reference](#a7-virtualization-reference)
- [A.8 AWS instance families](#a8-aws-instance-families)
- [A.9 EBS types](#a9-ebs-types)
- [A.10 Command reference](#a10-command-reference)
- [A.11 Diagnostic flow chart](#a11-diagnostic-flow-chart)
- [A.12 Formula summary](#a12-formula-summary)

---

## A.1 The latency ladder

*(Phases 2.1.1, 3.1.3, 5.3.4)* — **The single most important table in this map.**

| Operation | Time | On a human scale (1 cycle = 1 second) |
|---|---|---|
| 1 CPU cycle (3 GHz) | 0.3 ns | 1 second |
| L1 cache access | ~1 ns (4 cycles) | 4 seconds |
| L2 cache access | ~4 ns (12 cycles) | 12 seconds |
| L3 cache access | ~13 ns (40 cycles) | 40 seconds |
| **RAM access** | **~70 ns (200 cycles)** | **3.5 minutes** |
| Remote NUMA node RAM | ~120 ns | 6 minutes |
| **VM exit** | **~0.3–1.7 μs** | **17 minutes – 1.5 hours** |
| NVMe SSD read | ~50–100 μs | **1.5–3 days** |
| SATA SSD read | ~100–200 μs | 3–6 days |
| **EBS gp3 read** | **~1 ms** | **~1 month** |
| Same-AZ network RTT | ~0.3–0.5 ms | ~2 weeks |
| Inter-AZ RTT | ~1–2 ms | ~1–2 months |
| **HDD random read** | **~10 ms** | **~1 year** |
| Intercontinental RTT | ~100–200 ms | **10–20 years** |

> **Don't memorize this table — internalize it.** The sentence "RAM access is slow" only
> acquires meaning on this ladder.

---

## A.2 Units and conversions

*(Phases 0.1, 5.2.3)*

**Data size**
| Unit | Value |
|---|---|
| 1 byte | 8 bits |
| 1 KiB | 1,024 bytes |
| 1 MiB | 1,024 KiB |
| 1 GiB | 1,024 MiB |
| 1 TiB | 1,024 GiB |

> **KiB vs KB:** storage vendors use 1 KB = 1,000 bytes; operating systems use
> 1 KiB = 1,024 bytes. **A 1 TB disk shows up as 931 GiB in the operating system** — nothing
> is missing.

**Speed conversions**
```
Gbps ÷ 8 = GB/s          (1 Gbps = 125 MB/s)
GB/s × 8 = Gbps

In practice, with protocol overhead, about 95% of it is usable.
```

| Link | Theoretical | Practical |
|---|---|---|
| 1 Gbps | 125 MB/s | ~118 MB/s |
| 10 Gbps | 1.25 GB/s | ~1.18 GB/s |
| 25 Gbps | 3.125 GB/s | ~2.96 GB/s |
| 100 Gbps | 12.5 GB/s | ~11.8 GB/s |

**Time**
| Unit | In seconds |
|---|---|
| 1 ms (millisecond) | 10⁻³ |
| 1 μs (microsecond) | 10⁻⁶ |
| 1 ns (nanosecond) | 10⁻⁹ |

---

## A.3 Memory reference

### Cache hierarchy *(Phase 2.3)*

| Level | Typical size | Latency | Sharing |
|---|---|---|---|
| Register | ~1 KB | 0 cycles | Private to the core |
| L1i / L1d | 32–64 KB (each) | ~4 cycles | **Private to the core** |
| L2 | 512 KB – 2 MB | ~12 cycles | Private to the core |
| **L3** | 8–100+ MB | ~40 cycles | **ALL cores — noisy neighbors live here** |

| Constant | Value |
|---|---|
| **Cache line** | **64 bytes** — this much is read on every access |
| Page size | 4 KiB (standard), 2 MiB (huge), 1 GiB (gigantic) |
| TLB entry count | ~1,536 (typical) |
| **TLB reach (4 KiB)** | 1,536 × 4 KiB = **6 MiB** |
| **TLB reach (2 MiB huge)** | 1,536 × 2 MiB = **3 GiB** (512×) |

### The effect of hit ratio *(Phase 2.3.6)*

```
Average access = (hit_ratio × L1) + (miss_ratio × RAM)
```

| Hit ratio | Average access (4 / 200 cycles) | Relative speed |
|---|---|---|
| 90% | 23.6 cycles | 1× |
| 95% | 13.8 cycles | 1.7× |
| 99% | 5.96 cycles | **4.0×** |
| 99.9% | 4.2 cycles | 5.6× |

> **Going from 90% to 99% makes it 4× faster.** That is the leverage of cache
> optimization.

### DRAM and bandwidth *(Phase 2.4)*

| Generation | Typical speed | Bandwidth per channel |
|---|---|---|
| DDR3-1600 | 1600 MT/s | 12.8 GB/s |
| DDR4-3200 | 3200 MT/s | 25.6 GB/s |
| DDR5-4800 | 4800 MT/s | 38.4 GB/s |

```
Total bandwidth = Number of channels × Speed per channel

8 channels DDR5-4800 = 307 GB/s
4 channels DDR5-4800 = 154 GB/s
```

> **Between generations bandwidth multiplies while latency stays almost constant**
> (~70 ns). That is the essence of the memory wall *(Phase 2.1.2)*.

### NUMA *(Phase 2.7)*

| Access | Relative distance | Relative latency |
|---|---|---|
| Local node | 10 | 1× |
| Neighboring node | 21 | **~1.5–2×** |

---

## A.4 Storage reference

### HDD vs SSD *(Phases 3.1, 3.2)*

| | HDD (7200 RPM) | SATA SSD | NVMe SSD |
|---|---|---|---|
| **Random IOPS** | **80–120** | 50,000–100,000 | **500,000–1,000,000+** |
| Sequential throughput | 150–250 MB/s | ~550 MB/s | 3,000–7,000 MB/s |
| Latency | ~10 ms | ~100 μs | **~50 μs** |
| Number of queues | 1 | 1 (depth 32) | **65,535 (65,536 each)** |

**HDD latency components (7200 RPM):**
```
Seek             : ~4 ms     ← head movement
Rotational       : ~4.17 ms  ← half a turn (60000/7200/2)
Transfer         : ~0.1 ms
─────────────────────────
Total            : ~8.3 ms   → ~120 IOPS
```

> **HDD random IOPS has not increased in 30 years** — because it is mechanical. Capacity
> grew 1000×, IOPS stayed the same.

### NAND types *(Phase 3.2.4)*

| Type | Bits/cell | Write endurance | Speed | Cost |
|---|---|---|---|---|
| SLC | 1 | ~100,000 cycles | Fastest | Most expensive |
| MLC | 2 | ~10,000 | Fast | Expensive |
| TLC | 3 | ~3,000 | Medium | Medium |
| QLC | 4 | ~1,000 | Slow | **Cheapest** |

### The IOPS / throughput relationship *(Phase 3.4.1)*

```
Throughput = IOPS × I/O size
```

| I/O size | Throughput at 3,000 IOPS |
|---|---|
| 4 KiB | 12 MB/s |
| 16 KiB | 48 MB/s |
| 64 KiB | 192 MB/s |
| 1 MiB | 3,000 MB/s (runs into the throughput limit) |

> **Same IOPS, different throughput.** The question "which one is the bottleneck" is
> answered by the I/O size.

### The queue curve *(Phase 3.4.4)* — **it recurs in three phases**

| Utilization | Relative latency |
|---|---|
| 50% | 1× |
| 80% | 2.5× |
| 90% | 5× |
| **95%** | **8×** |
| **99%** | **40×** |

### RAID levels *(Phase 3.5)*

| Level | Min. disks | Capacity | Read | Write | Durability |
|---|---|---|---|---|---|
| 0 | 2 | 100% | ✅ Fast | ✅ Fast | ❌ **None** |
| 1 | 2 | 50% | ✅ Fast | Normal | ✅ 1 disk |
| 5 | 3 | (n-1)/n | ✅ Fast | ❌ **4× penalty** | ✅ 1 disk |
| 10 | 4 | 50% | ✅ Fast | ✅ Fast | ✅ 1 from each mirror |

> **RAID on top of EBS is usually wrong** — EBS is already replicated. RAID 0 can only
> make sense for summing IOPS up to the instance ceiling *(Phase 3.5.3)*.

---

## A.5 Bus and PCIe reference

*(Phase 4.2)*

| Generation | Per lane | x4 | x8 | **x16** |
|---|---|---|---|---|
| Gen3 | 0.985 GB/s | 3.9 GB/s | 7.9 GB/s | **15.8 GB/s** |
| Gen4 | 1.969 GB/s | 7.9 GB/s | 15.8 GB/s | **31.5 GB/s** |
| Gen5 | 3.938 GB/s | 15.8 GB/s | 31.5 GB/s | **63 GB/s** |
| Gen6 | 7.563 GB/s | 30.3 GB/s | 60.5 GB/s | **121 GB/s** |

**Device requirements:**

| Device | Needs |
|---|---|
| NVMe SSD | Gen3/Gen4 x4 |
| 10 Gbps NIC | Gen3 x4 |
| **25 Gbps NIC** | **Gen3 x8** |
| **100 Gbps NIC** | **Gen4 x8 or Gen3 x16** |
| GPU | Gen4/Gen5 x16 |

```bash
lspci -vv | grep -E "LnkCap|LnkSta"   # the slot's capability and its actual state
```

> **GPU bandwidth comparison** *(Phase 4.2.4)*:
> ```
> GPU HBM memory : ~2000–3000 GB/s
> PCIe Gen4 x16  :        ~32 GB/s
> Difference     :          ~70×
> ```
> **The GPU's own memory is fast; getting data to it is slow.**

---

## A.6 Network reference

### Ethernet *(Phases 5.1.2, 5.2.3)*

| Standard | Gbps | GB/s |
|---|---|---|
| 1 GbE | 1 | 0.125 |
| 10 GbE | 10 | 1.25 |
| 25 GbE | 25 | 3.125 |
| 100 GbE | 100 | 12.5 |
| 400 GbE | 400 | 50 |

**Frame overhead:**
```
Preamble + SFD        :  8 bytes  (outside the MTU, on the wire)
Ethernet header + CRC : 18 bytes  (outside the MTU)
Inter-frame gap       : 12 bytes  (outside the MTU)
IP header             : 20 bytes  (inside the MTU)
TCP header            : 20 bytes  (inside the MTU)
────────────────────────────────
Total                 : 78 bytes

MTU 1500 → efficiency 94.9%  (1460 / 1538)
MTU 9000 → efficiency 99.1%  (8960 / 9038)  + 6× fewer packets
```

### The four components of latency *(Phase 5.3.2)*

| Component | Depends on | What you can do |
|---|---|---|
| **Propagation** | Distance (speed of light) | ❌ Only get closer (CDN) |
| **Transmission** | Packet ÷ link speed | ✅ A faster link |
| **Processing** | Number of devices | ⚠️ Fewer hops |
| **Queuing** | Load | ✅ **The most actionable one** |

### Real RTT values *(Phase 5.3.4)*

| Path | RTT |
|---|---|
| Loopback | ~0.02 ms |
| Same rack | ~0.1 ms |
| **Same AZ** | **~0.3–0.5 ms** |
| **Between AZs** | **~1–2 ms** |
| Regions on the same continent | ~20–40 ms |
| **Intercontinental** | **~100–200 ms** |

### BDP *(Phase 5.3.3)*

```
BDP = Bandwidth × RTT
Maximum throughput = Window size ÷ RTT
```

| Scenario | Calculation | Result |
|---|---|---|
| 10 Gbps, 100 ms RTT | 1.25 GB/s × 0.1 s | **BDP = 125 MB** |
| 64 KB window, 100 ms | 65,536 ÷ 0.1 | **655 KB/s = 5.2 Mbps** |
| 64 KB window, 0.4 ms | 65,536 ÷ 0.0004 | 164 MB/s |

> **Same window, same link — the RTT difference alone creates a 250× difference in
> throughput.**

### Packet rate *(Phase 4.4.2, Answer 6.2)*

```
pps = Bandwidth ÷ (Packet size × 8)

10 Gbps, 1500 bytes  →  833,000 pps
100 Gbps, 1500 bytes →  8,333,000 pps
```

---

## A.7 Virtualization reference

### The cost of a VM exit *(Phase 6.3.4)*

| Era | Virtualization tax |
|---|---|
| Binary translation (2000s) | 30–50% |
| VT-x + EPT (2010s) | 5–15% |
| **Nitro / SR-IOV** | **<1%** |

```
A single VM exit: 1,000–5,000 cycles (~0.3–1.7 μs)
```

### Generations of I/O virtualization *(Phase 6.6)*

| Method | VM exits | Performance | Live migration |
|---|---|---|---|
| Emulation | Very high | 20–40% | ✅ |
| virtio | Medium | 70–90% | ✅ |
| **SR-IOV** | **~Zero** | **95–99%** | ❌ |

### Interpreting steal time *(Phase 6.5.2)*

| Value | What it means | Action |
|---|---|---|
| 0–2% | Normal | — |
| 2–10% | Mild contention | Watch it |
| **10–25%** | **Serious** | Restart / change type |
| **25%+** | **Unacceptable** | Move immediately |

> **Distinguish the two causes:** host contention (m/c/r families) vs **credit exhaustion**
> (t family). In the second case restarting does not help.

### Container vs VM *(Phase 6.7.3)*

| | VM | Container |
|---|---|---|
| Startup | 30–60 s | **50–500 ms** |
| Memory overhead | 512 MB – 2 GB | **1–10 MB** |
| Density (64 GB host) | ~30 | **~1000+** |
| **Isolation boundary** | **Hypervisor (~100K lines)** | Kernel (~30M lines) |

---

## A.8 AWS instance families

*(Phase 7.1)*

### Decoding the name
```
m 7 g d . 2xlarge
│ │ │ │      └─ 8 vCPU
│ │ │ └──────── d = local NVMe
│ │ └────────── g = Graviton (i=Intel, a=AMD)
│ └──────────── 7th generation
└────────────── m = general purpose
```

### The families

| Family | vCPU:RAM | Physical highlight | For what |
|---|---|---|---|
| **m** | 1:4 | Balanced | **Start here if you don't know** |
| **c** | 1:2 | **High clock + lots of L3 per vCPU** | CPU-bound, single thread |
| **r** | 1:8 | Memory capacity + channel count | Caches, analytics |
| **x** | 1:16 | Very large RAM | In-memory databases |
| **i** | — | **NVMe instance store** | High-IOPS NoSQL |
| **d** | — | High-capacity local disk | Data warehousing |
| **p / g** | — | GPU | Training / inference |
| **t** | variable | **Credit based** | ⚠️ Intermittent load only |

### The vCPU equivalence *(Phases 1.5.3, 7.1.6)*

| Processor | 1 vCPU = |
|---|---|
| Intel / AMD (x86) | **1 SMT thread (half a core)** |
| **Graviton (ARM)** | **1 physical core** |

### t family baseline performance *(Phase 6.5.3)*

| Instance | vCPU | Baseline |
|---|---|---|
| t3.micro | 2 | 10% |
| t3.small | 2 | 20% |
| t3.medium | 2 | 20% |
| t3.large | 2 | 30% |
| t3.xlarge | 4 | 40% |
| t3.2xlarge | 8 | 40% |

> **Rule: the t family is right when your average CPU utilization is clearly below the
> baseline.** Otherwise the t family isn't cheap, it's broken.

### Network performance *(Phase 5.2.3)*

| Size | Network |
|---|---|
| `.large` | "Up to 10 Gigabit" — **burst, not guaranteed** |
| `.4xlarge` | "Up to 25 Gigabit" |
| `.12xlarge` | 25 Gbps **guaranteed** |
| `.24xlarge` / `.metal` | 50–100 Gbps guaranteed |

---

## A.9 EBS types

*(Phase 7.3)*

| Type | Physical | IOPS | Throughput | Latency | For what |
|---|---|---|---|---|---|
| **gp3** | SSD | 3,000–**16,000** | 125–1,000 MB/s | ~1 ms | **The default** |
| gp2 | SSD | size×3 (max 16,000) | bound to size | ~1 ms | ⚠️ **Move to gp3** |
| **io2 / BX** | SSD | **up to 256,000** | 4,000 MB/s | **<1 ms** | Critical DB |
| st1 | **HDD** | ~500 burst | 500 MB/s | High | **Sequential** |
| sc1 | **HDD** | ~250 | 250 MB/s | High | Archive |

### Decision tree
```
Sequential access?           → st1 / sc1
IOPS > 16,000?               → io2 Block Express
Latency < 1 ms required?     → io2
No persistence needed?       → instance store (i family)
Everything else              → gp3     (~80% of cases)
```

> **Check both ceilings:** the volume limit **and** the instance EBS bandwidth limit
> *(Phase 7.3.5)*.
> ```bash
> aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[0].EbsInfo'
> ```

---

## A.10 Command reference

### CPU
```bash
lscpu                      # CPU model, cores, cache, NUMA
top                        # us/sy/wa/st — st = steal time
top -H                     # per thread (for single-thread bottlenecks)
mpstat -P ALL 1            # per core, %soft = network processing
vmstat 1                   # r (run queue), st, si/so
uptime                     # load average
perf stat -e cache-misses,instructions,cycles <command>
```

### Memory
```bash
free -h                    # look at available, not at free
vmstat 1                   # si/so = swap activity → alarm
cat /proc/meminfo
numactl --hardware         # NUMA nodes and distances
numastat -m
dmesg | grep -i "out of memory"
```

### Storage
```bash
iostat -x 1                # await, aqu-sz, r/s, w/s
iotop -o                   # which process is doing I/O
lsblk                      # block device tree
df -h                      # filesystem utilization
nvme list                  # NVMe devices
smartctl -a /dev/sda       # disk health
```

### Network
```bash
ip a; ip r                 # addresses and routing
ss -s                      # socket summary
ss -i                      # cwnd, rtt (BDP analysis)
ethtool eth0               # speed, duplex
ethtool -S eth0            # drop/error counters
ethtool -g eth0            # ring buffer
ethtool -l eth0            # queue (channel) count
ethtool -k eth0            # offload status
sar -n DEV 1               # interface throughput
cat /proc/interrupts       # interrupt distribution
ping -M do -s 1472 <target> # MTU test
iperf3 -c <target> -P 10    # parallel stream test
```

### Virtualization / containers
```bash
systemd-detect-virt        # which hypervisor
lscpu | grep -i hypervisor
cat /sys/fs/cgroup/memory.max      # the container's real memory limit
cat /sys/fs/cgroup/memory.current
cat /sys/fs/cgroup/cpu.max
```

### Bus / devices
```bash
lspci                      # PCI devices
lspci -vv | grep -E "LnkCap|LnkSta"   # PCIe lanes and generation
lsusb
dmidecode -t memory        # RAM modules, channels, speed
```

---

## A.11 Diagnostic flow chart

*(Phase 7.5.2)* — **when you hear "my application is slow"**

```
STEP 0 — QUESTIONS (2 min)
  Since when? / p50 or p99? / How much? / Continuous?

STEP 1 — CPU (3 min)         top, mpstat -P ALL 1
  us high        → CPU BOUND
  sy high        → kernel: syscalls, interrupts, context switching
  wa high        → I/O BOUND
  st high        → NEIGHBOR or CREDIT
  all low        → ↓ continue

STEP 2 — MEMORY (3 min)      free -h, vmstat 1
  available low   → MEMORY BOUND (capacity)
  si/so > 0       → SWAP — urgent
  full, no swap   → could be bandwidth

STEP 3 — DISK (3 min)        iostat -x 1
  await + aqu-sz high       → I/O BOUND
  r/s+w/s ≈ limit           → IOPS ceiling
  kB/s ≈ limit              → throughput ceiling
  %util                     → MISLEADING on NVMe

STEP 4 — NETWORK (3 min)     ss -s, ethtool -S, sar -n DEV 1
  rx_dropped ↑    → NIC/CPU
  retransmit ↑    → packet loss
  bandwidth maxed → NETWORK BOUND
  link idle, slow → BDP / window

STEP 5 — NONE OF THEM (6 min) → WAITING BOUND
  Pool metrics / distributed tracing / GC / locks / downstream
  ⚠️ Buying hardware FIXES NOTHING IN THIS CASE
```

---

## A.12 Formula summary

| Formula | Where | For what |
|---|---|---|
| `Average access = hit×L1 + miss×RAM` | 2.3.6 | The gain from cache |
| `Bandwidth = channels × channel_speed` | 2.4.3 | Memory capability |
| `TLB reach = entries × page_size` | 2.5.4 | The huge page decision |
| `HDD latency = seek + rotational + transfer` | 3.1.2 | The HDD IOPS ceiling |
| `Rotational = 60000 / RPM / 2` (ms) | 3.1.2 | 7200 RPM → 4.17 ms |
| **`Throughput = IOPS × I/O size`** | **3.4.1** | **Which limit is the bottleneck** |
| `await ≈ aqu-sz ÷ IOPS` (Little) | Answer 3.3 | An `iostat` consistency check |
| `PCIe BW = lanes × generation_speed` | 4.2.2 | Is the slot enough |
| `pps = Gbps ÷ (packet_bytes × 8)` | 4.4.2 | Interrupt load |
| `Transmission delay = packet_bits ÷ link_bps` | 5.3.2 | A latency component |
| **`BDP = bandwidth × RTT`** | **5.3.3** | **The TCP window requirement** |
| `Max throughput = window ÷ RTT` | 5.3.3 | "Fast link, slow transfer" |
| `VM exit load = pps × exit_cost ÷ clock` | 6.3.4 | The case for SR-IOV |
| `Required vCPU = requests/s × vCPU-s/request ÷ target_utilization` | 7.6 | The capacity plan |

---

*End of Appendix A.* → **[Appendix B — Glossary](Appendix_B_Glossary.md)** · **[README](README_en.md)**
