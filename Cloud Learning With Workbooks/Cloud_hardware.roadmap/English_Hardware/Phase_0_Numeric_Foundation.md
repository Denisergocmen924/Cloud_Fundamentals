# Phase 0 — Numeric Foundation

> **Navigation:** [◀ Contents](README_en.md) · **Phase 0** · [Phase 1 — CPU Architecture ▶](Phase_1_CPU_Architecture.md)

---

## Why does this phase exist?

A cloud engineer's day is full of sentences like these:

- "`t3.micro` gives 1 GiB of RAM, but monitoring shows 976 MB."
- "I mounted a 100 GB EBS volume and `df` says 93 GB. Where did 7 GB go?"
- "The application crashes at 3 GB, and the server has 64 GB of RAM."
- "Cache line 64 bytes, page 4096 bytes, PCIe x16 — why always these numbers?"
- "The stack trace says `0x7ffd4a2b1c30`. What does that mean?"

**All** of these questions share one root: the computer counts in base two, humans count in
base ten, and the translation between those two worlds constantly leaks.

This phase goes down to that root. But its real purpose is bigger: **to build the vocabulary
for the rest of this map.**

In Phase 2 we will ask "why is SRAM expensive and DRAM cheap?" — the answer lies in the
transistor count from this phase. In Phase 1 we will say "clock speed has a physical
ceiling" — the answer lies in the delay of the adder circuit from this phase. In Phase 6 we
will ask "what is a vCPU, really?" — the foundation of that answer is the register from this
phase.

> **Honest warning:** This is the **most abstract** phase of the map and the one that looks
> furthest from the cloud. Feeling "I wanted to learn AWS — why am I drawing logic gates?"
> is normal. Think of it this way: Phase 0 is an investment. It pays off in Phase 2 and
> Phase 6, at the moment you say "ahh, **so that's why** it works like this." Someone who
> skips this phase **memorizes** the cloud; someone who works through it **derives** it.

---

## By the end of this phase

- You will be able to convert a number between binary, decimal and hexadecimal **by hand**
- You will be able to answer "why does a 1 TB disk show up as 931 GB?" with a calculation
- You will know **what** the difference between 32-bit and 64-bit changes (and what it doesn't)
- You will be able to explain the chain from transistor to logic gate, from gate to adder
  circuit, and from circuit to memory
- You will be able to explain **physically** why clock speed cannot keep increasing forever
- You will know where the cost difference between SRAM and DRAM comes from

---

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 0.1 | Number systems | `[mechanism]` | The language of every number |
| 0.2 | The transistor | `[concept]` | Why the clock stalled, why cores multiplied |
| 0.3 | Logic gates | `[mechanism]` / `[concept]` | The inside of the ALU + the clock ceiling |
| 0.4 | Flip-flops and latches | `[concept]` | The physical basis of registers and cache |

---
---

# 0.1 Number Systems

## 0.1.1 Why does a computer count in binary? `[mechanism]`

The common answer is: *"Because electricity is either on or off."* That answer is **wrong.**

Electricity is not "on/off." The voltage on a wire can be 0V, 0.3V, 1.1V, 2.7V, 3.3V — an
infinite number of values in between. Electricity is **analog**. So in principle we could
define 10 different voltage levels on a wire and build a computer that works in base ten.
It has even been tried (in the 1950s a Soviet computer called Setun, which worked in base
three, was actually built).

The real answer: **noise margin.**

As a signal travels along a wire, surrounding electromagnetic fields, heat, neighboring
wires and fluctuations in the power supply distort it. A signal you send at 3.3V arrives on
the other side as 3.1V or 3.5V.

Now compare two scenarios:

**Binary system (2 levels):** 0–1.0V = "0", 2.3–3.3V = "1". The 1.3-volt gap in between is a
**forbidden zone**. Even if the signal drifts by 0.4 volts, it is still read correctly. You
have an enormous error margin.

**Decimal system (10 levels):** If you divide 3.3 volts into 10, each level is 0.33V apart.
If the signal drifts by 0.2 volts, you read a 5 as a 6. On a chip where billions of
transistors switch billions of times per second, that error rate makes the system unusable.

> **Summary:** Binary was chosen not because of the nature of electricity, but because of
> **reliability engineering**. Two levels = maximum noise tolerance = billions of switches
> that work without errors.

![Figure 0.1 — Voltage levels and noise margin](../diagrams/png/hw-0-01-noise-margin.png)
*Figure 0.1 — Noise margin compared in binary and decimal encoding. In binary, the
"forbidden zone" is so wide that the signal is read correctly even when badly distorted.*

**❓ A question that comes to mind: Then why do SSDs use multi-level cells (TLC, QLC)?**

A very good question, and a preview of Phase 3. In storage the priority is different. The
voltage level in an SSD cell **does not change billions of times per second** — it is
written once and sits there for years. Without time pressure, you can squeeze in more levels
and gain capacity. The price is exactly what you would expect: less reliability, less write
endurance, slower reads. In Phase 3.2 we will see the full table. The lesson here: **same
physics, different priority, different decision.**

---

## 0.1.2 Bit, byte and the story of 8 `[concept]`

**Bit** (binary digit): a single binary digit. 0 or 1. The smallest unit of information.

**Byte:** 8 bits. So why 8? Because:

1. **History:** In 1964 the IBM System/360 encoded characters in 8-bit blocks (EBCDIC).
   Before that, 6-bit, 7-bit and 9-bit bytes existed. System/360 became so dominant that
   8 bits became the standard.
2. **ASCII compatibility:** ASCII is 7 bits (128 characters). The 8th bit could be used
   either for parity (error checking) or for extensions.
3. **Power of two:** 8 = 2³. Being able to split a byte in half (2 × 4 bits), with 4 bits
   mapping exactly to one hex digit, made life much easier.

> **⚠️ Common misconception:** "1 byte is 8 bits" is **not a law of nature; it is a tradition
> that won.** On some old systems a byte was 6 or 9 bits. Today it is universal, so we treat
> it as a standard — and in practice that is correct. But the answer to "why 8?" is history,
> not physics.

**Nibble:** 4 bits, i.e. half a byte. Exactly one hex digit. `0xA3` = `1010 0011` — left
nibble `A`, right nibble `3`.

---

## 0.1.3 Converting decimal ↔ binary ↔ hexadecimal `[mechanism]`

This section teaches a **method**, not a table to memorize. Understand the method once and
you will use it for life.

### Place-value table (binary)

Every binary digit is a power of 2. From right to left:

| Digit | 2⁷ | 2⁶ | 2⁵ | 2⁴ | 2³ | 2² | 2¹ | 2⁰ |
|---|---|---|---|---|---|---|---|---|
| **Value** | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |

**Memorize this table.** 128-64-32-16-8-4-2-1. A cloud engineer uses it several times a day:
calculating subnet masks, reading cache sizes, decoding permission bits.

### Binary → decimal (the easy direction)

Let's convert `1011 0010`. Write the place values underneath and add up the ones:

```
  1    0    1    1    0    0    1    0
128   64   32   16    8    4    2    1
 ✓         ✓    ✓              ✓

128 + 32 + 16 + 2 = 178
```

### Decimal → binary (method 1: subtraction — fastest for humans)

Let's convert `178`. Start from the left and ask "does it fit?":

```
178 ≥ 128?  Yes → bit = 1,  remainder: 178 - 128 = 50
 50 ≥  64?  No  → bit = 0
 50 ≥  32?  Yes → bit = 1,  remainder:  50 -  32 = 18
 18 ≥  16?  Yes → bit = 1,  remainder:  18 -  16 =  2
  2 ≥   8?  No  → bit = 0
  2 ≥   4?  No  → bit = 0
  2 ≥   2?  Yes → bit = 1,  remainder:   2 -   2 =  0
  0 ≥   1?  No  → bit = 0

Result: 1011 0010 ✓
```

### Decimal → binary (method 2: dividing by 2 — machine-like but slower)

Keep dividing `178` by 2, collect the remainders, and read them **from last to first**:

```
178 ÷ 2 = 89  remainder 0   ↑
 89 ÷ 2 = 44  remainder 1   |
 44 ÷ 2 = 22  remainder 0   |
 22 ÷ 2 = 11  remainder 0   |  read in
 11 ÷ 2 =  5  remainder 1   |  this direction
  5 ÷ 2 =  2  remainder 1   |
  2 ÷ 2 =  1  remainder 0   |
  1 ÷ 2 =  0  remainder 1   |

Result (bottom to top): 1011 0010 ✓
```

Both methods give the same result. The first is better for mental arithmetic; the second is
better for error-free work on paper.

### Hexadecimal (hex) — and why it exists

Hex is base 16: `0 1 2 3 4 5 6 7 8 9 A B C D E F`. `A`=10, `B`=11, `C`=12, `D`=13,
`E`=14, `F`=15.

Hex exists for one reason: **16 = 2⁴, so one hex digit is exactly 4 bits.** That means
converting between binary and hex requires no arithmetic — only grouping:

```
Binary: 1011 0010
        ↓    ↓
Hex:     B    2      →  0xB2
```

Same in the other direction:

```
Hex:    0x7F
         7    F
         ↓    ↓
Binary: 0111 1111
```

**Nibble → hex table** (memorize this too, 16 rows):

| Binary | Hex | Decimal | | Binary | Hex | Decimal |
|---|---|---|---|---|---|---|
| 0000 | 0 | 0 | | 1000 | 8 | 8 |
| 0001 | 1 | 1 | | 1001 | 9 | 9 |
| 0010 | 2 | 2 | | 1010 | A | 10 |
| 0011 | 3 | 3 | | 1011 | B | 11 |
| 0100 | 4 | 4 | | 1100 | C | 12 |
| 0101 | 5 | 5 | | 1101 | D | 13 |
| 0110 | 6 | 6 | | 1110 | E | 14 |
| 0111 | 7 | 7 | | 1111 | F | 15 |

**❓ A question that comes to mind: Why use hex instead of just writing binary?**

Because binary is **unreadable**. A 64-bit memory address is 64 characters in binary:

```
0111111111111101010010100010101100011100001100110000000000000000
```

The same address in hex is 16 characters:

```
0x7FFD4A2B1C330000
```

Same information, a quarter of the length, and **without losing the bit structure.** If you
converted it to decimal (`9223094...`) it would be readable, but you could not see the bit
pattern. Hex is the sweet spot between human readability and bit transparency.

> **🔧 See it on your machine** (optional)
>
> ```bash
> printf '%d\n' 0xB2        # hex → decimal
> printf '0x%X\n' 178       # decimal → hex
> echo 'obase=2; 178' | bc  # decimal → binary
> ```
> **Expected output:** `178`, `0xB2`, `10110010`

> **🤔 Think 0.1**
> The `755` in `chmod 755` is in octal (base 8). 8 = 2³.
> How many bits is one octal digit? And why, when you convert `755` to binary, do you see
> exactly the permissions `rwxr-xr-x`?
> *(Answer: in the "Answers to the Think Questions" section at the end of the phase)*

---

## 0.1.4 KB or KiB — the "missing disk space" puzzle `[application]`

This is the topic in this phase that causes **the most money lost and the most arguments**.
And almost everyone gets it wrong.

### Two different "kilos"

| Prefix | Base | Value | Who uses it |
|---|---|---|---|
| **KB** (kilobyte) | 10 | 1,000 bytes | Disk manufacturers, network equipment, AWS network metrics |
| **KiB** (kibibyte) | 2 | 1,024 bytes | Operating systems, RAM, `/proc`, `free`, `df` |
| **MB** / **MiB** | 10 / 2 | 1,000,000 / 1,048,576 | ↑ same |
| **GB** / **GiB** | 10 / 2 | 10⁹ / 2³⁰ (1,073,741,824) | ↑ same |
| **TB** / **TiB** | 10 / 2 | 10¹² / 2⁴⁰ | ↑ same |

The gap grows with each step:

| Unit | Binary / decimal ratio | Difference |
|---|---|---|
| KiB / KB | 1.024 | 2.4% |
| MiB / MB | 1.048576 | 4.9% |
| GiB / GB | 1.073741824 | 7.4% |
| TiB / TB | 1.099511627776 | 10.0% |

### "I bought a 1 TB disk and got 931 GB"

The calculation:

```
Manufacturer: 1 TB = 1,000,000,000,000 bytes  (decimal, 10¹²)
The operating system displays this in GiB:

1,000,000,000,000 ÷ 1,073,741,824 = 931.32 GiB
```

Nobody cheated you. The manufacturer speaks decimal, the operating system speaks binary. But
the operating system often **labels the unit "GB"** even though it is showing GiB. This
labeling laziness is the real source of the confusion.

> **⚠️ Common misconception:** "The missing space is due to file system metadata." Partly
> true, but a **very small** share. Of the 1 TB gap, 68 GB comes from the base difference
> above. ext4 metadata plus reserved blocks is typically 1–5% (including the default 5% root
> reserve). These are two separate effects — don't mix them up.

### Why does this matter in the cloud?

**1. EBS is billed in GiB; the network is measured in Gbps.**

- "100 GB EBS volume" → AWS actually gives you **100 GiB** and bills **100 GiB**.
- "10 Gbps network" → really **10 × 10⁹ bits/second**, i.e. decimal.

Two different bases in the same documentation. If you don't know which is which when doing
capacity planning, you'll be off by 7–10%.

**2. Bits or bytes?**

Networks always speak **bits**; storage always speaks **bytes**. Lowercase `b` = bit,
uppercase `B` = byte.

```
How many MB/s of file data does a 10 Gbps link actually carry?

10 Gbps = 10,000,000,000 bits/s
        ÷ 8 = 1,250,000,000 bytes/s
        = 1,250 MB/s  (decimal)
        ≈ 1,192 MiB/s (binary)

And that is the theoretical ceiling. After protocol overhead (Ethernet + IP + TCP
headers), in practice you'll see around ~1,100–1,180 MB/s.
```

Someone who can't do this calculation says "I have a 10 Gbps network, a 10 GB file will
transfer in 1 second." Reality: **about 8.5 seconds.**

> **🔧 See it on your machine** (optional)
>
> ```bash
> df -h /          # -h human-readable: shows 'G' but it is GiB
> df -H /          # -H decimal: real GB
> free -h          # RAM is always binary (GiB)
> ```
> **Expected output:** `df -h` and `df -H` show **different numbers** for the same disk.
> For example `50G` with `-h` and `54G` with `-H`. The 7.4% gap is exactly the GiB/GB
> difference.

> **🤔 Think 0.2**
> You see this on an AWS bill: "Data Transfer Out: 500 GB". You measured the total size of
> the files you downloaded from S3 that month with `du -sh`, and it came to `466G`.
> Is the bill wrong or the measurement? What happened?
> *(Answer: at the end of the phase)*

---

## 0.1.5 What 32-bit and 64-bit really mean `[application]`

This is the item in this section that connects **most directly** to the cloud.

The phrase "64-bit system" implies three different things at once, and separating them is
essential:

### 1. Register width

The temporary memory cells inside the CPU (registers — we'll go into detail in Phase 1.1)
are 64 bits wide. So the CPU can operate on a 64-bit number in a single step.

On a 32-bit CPU, a 64-bit addition requires **two separate operations** (the low 32 bits,
then the high 32 bits with the carry). On a 64-bit CPU it is one operation.

### 2. Address space — the part that really matters

The address a CPU uses to point to a location in memory is also a number. The number of bits
in the address determines how many distinct locations it can point to:

```
32-bit address:  2³² = 4,294,967,296 distinct addresses
                 Each address points to 1 byte
                 → 4,294,967,296 bytes = 4 GiB

64-bit address:  2⁶⁴ = 18,446,744,073,709,551,616 addresses
                 → 16 EiB (exbibytes) — practically infinite
```

**This is what killed 32-bit.** Even if the server has 64 GB of RAM, a 32-bit process
**cannot address** more than 4 GiB. Memory you cannot address is memory you cannot use.

In practice it's even worse: on Linux a 32-bit user process can generally use 3 GiB (the top
1 GiB is reserved for the kernel). On Windows the default is 2 GiB.

> **⚠️ Common misconception:** "A 64-bit system is twice as fast as 32-bit." **No.** The speed
> gain from being 64-bit is noticeable only in workloads that operate on 64-bit numbers
> (big-integer math, cryptography). Most applications work with 32-bit integers and **don't
> get any faster** after moving to 64-bit — in fact, because pointers double in size, holding
> the same data takes more memory and more cache lines, which **can slow things down**. The
> real gain of 64-bit is not speed; it is **address space.**

**Cloud connection `[application]`:**

Finding a 32-bit AMI on AWS is now practically impossible — they are all `x86_64` or
`arm64`. The reason is exactly the above: even when a modern instance has less than 4 GiB of
RAM (`t3.nano` has 0.5 GiB), running 32-bit has no advantages and many disadvantages.

But this scenario is still real and seen in the field: **you moved an old application to the
cloud with lift-and-shift.** The application is compiled as 32-bit. You put it on an
`r5.4xlarge` (128 GiB RAM). The application still throws `OutOfMemoryError` at 3 GB. Growing
the instance **does nothing** — the problem is not the instance but the binary's address
space. The fix: recompile.

This is exactly the kind of example that shows why Phase 0 exists. Without this knowledge,
you would spend hours resizing instances for this problem.

> **🤔 Think 0.3**
> A `c7g.large` Graviton (ARM) instance is 64-bit. If the 64-bit address space is 16 EiB,
> why doesn't any server support 16 EiB of RAM? Is it physically impossible, or is there
> another limit?
> *(Answer: at the end of the phase)*

---
---

# 0.2 The Transistor

This is the most "physics" section of the map, and it is deliberately kept **shallow**. The
goal is not to understand how a transistor is manufactured, but to understand **two
consequences**: why it behaves like a switch, and why its count and size shaped today's
cloud architecture.

## 0.2.1 A transistor is a switch `[concept]`

The type of transistor used in modern chips is the **MOSFET** (metal-oxide-semiconductor
field-effect transistor). It has three terminals:

- **Gate** — the control terminal
- **Source** — where current enters
- **Drain** — where current leaves

The operating principle in one sentence: **apply voltage to the gate and electricity flows
between source and drain; don't, and it doesn't.**

So a small control signal switches a large current on and off. That is a **switch**.

**Analogy:** Think of a valve on a garden hose. Turning the valve handle takes little force,
but it shuts off or opens all the water in the hose. Gate = handle, water = current. The
difference: you can turn this valve's handle **billions of times** per second and it never
wears out (there are no moving parts).

![Figure 0.2 — MOSFET switching behavior](../diagrams/png/hw-0-02-mosfet-switch.png)
*Figure 0.2 — When voltage is applied to the gate, the source–drain channel becomes
conductive. Left: off state (0); right: on state (1).*

**Why this matters:** From a switch you get a logic gate, from logic gates an adder circuit,
from adder circuits a CPU. This is the first link in the chain.

## 0.2.2 Scale — billions of switches `[concept]`

A sense of scale in numbers (rough, varies by generation):

| Chip | Year | Transistor count |
|---|---|---|
| Intel 4004 | 1971 | 2,300 |
| Intel Pentium | 1993 | ~3.1 million |
| Modern desktop CPU | ~2023 | ~10–25 billion |
| Modern server CPU (many cores) | ~2023 | ~50–100+ billion |
| Modern large GPU | ~2023 | ~80 billion+ |

**To grasp this:** 50 billion transistors means about six switches for every person on
Earth, packed into an area the size of a thumbnail, wired together, switching on and off
billions of times per second.

**What does "nm" mean?** Phrases like "3nm process" describe the size of a transistor feature
(**feature size** / process node). Today this number is more of a **marketing name** than a
real physical measurement — different manufacturers' "5nm" do not measure the same thing.
The only thing you need to know: **the smaller the number, the more transistors fit in the
same area.**

## 0.2.3 Moore's Law, its wall, and its effect on the cloud `[concept]`

**Moore's Law** (1965, Gordon Moore): the number of transistors that can be put on a chip
doubles roughly every two years.

This was not a law of nature but an **observation and an industry target**. It held for
about 50 years.

But alongside it was a second observation called **Dennard scaling**: as transistors shrink,
power consumption per unit area stays constant. This meant "while transistors shrink, we can
also raise clock speed for free." From the 1970s to the early 2000s, clock speed soared this
way: 1 MHz → 3 GHz.

**Then Dennard scaling collapsed around 2005.** Transistors became so small that leakage
current and heat density became unmanageable. Raising clock speed now pushed heat to
intolerable levels.

The industry's answer was: **"If we can't make a single core faster, let's add more cores."**

And this is the moment the whole architectural philosophy of the cloud was born:

```
Dennard collapse (~2005)
        ↓
Clock speed stuck at ~3-5 GHz  (still there today)
        ↓
Performance gains = more cores
        ↓
Software HAD TO become parallel
        ↓
"Vertical scaling" (a more powerful machine) hit the ceiling
        ↓
"Horizontal scaling" (more machines) became the only way out
        ↓
CLOUD
```

> **Read this chain carefully.** It is the deepest technical answer to "why does the cloud
> exist?" Before the cloud was a business-model innovation, it was the architectural answer
> to **a constraint born from physics**. Auto Scaling Groups, load balancers, stateless
> services, microservices — all are children of the fact that "we can't make a single
> machine faster."

**`[skip]` — what we are not going into:** the CMOS manufacturing process, dopant chemistry,
lithography, gate oxide thickness, quantum tunneling details. That is the chip designer's job.
There is no decision a cloud engineer would change with this knowledge.

**But carry one consequence as a `[concept]`:** A transistor **produces heat** when it
switches. That heat cannot exceed a limit the chip can withstand (TDP — thermal design
power). Therefore:

- When all cores are running at full load, a CPU runs at its **base clock**
- When only a few cores are active, the heat budget grows, so it rises to the **turbo/boost clock**
- So a "3.5 GHz CPU" is really a CPU that runs **anywhere between 2.4 and 4.2 GHz depending on the situation**

This will come up in Phase 1.3 and Phase 7 when we interpret instance performance.

> **🤔 Think 0.4**
> `c7i.2xlarge` (8 vCPUs) and `c7i.16xlarge` (64 vCPUs) use the same CPU family, and AWS lists
> the same "up to 3.2 GHz" for both. If you ran a single-threaded benchmark, which would you
> expect to score higher? Why?
> *(Answer: at the end of the phase)*

---

# 0.3 Logic Gates

A transistor is a switch. Combine switches in specific patterns and you get circuits that
**make decisions**. These are called logic gates.

## 0.3.1 The basic gates: AND, OR, NOT `[mechanism]`

Each gate looks at its inputs and produces a single output. Its behavior is fully defined by
a **truth table**.

### NOT — one input

Inverts the input.

| A | OUTPUT |
|---|---|
| 0 | 1 |
| 1 | 0 |

### AND — two inputs

1 if **all** inputs are 1.

| A | B | OUTPUT |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | **1** |

### OR — two inputs

1 if **at least one** input is 1.

| A | B | OUTPUT |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | **1** |
| 1 | 0 | **1** |
| 1 | 1 | **1** |

![Figure 0.3 — Basic logic gates and their symbols](../diagrams/png/hw-0-03-basic-gates.png)
*Figure 0.3 — Standard circuit symbols and truth tables of the AND, OR and NOT gates.*

**These tables should look familiar.** You use the same logic every day:

```bash
# In the shell
[ -f file ] && [ -r file ]      # AND: if both are true
[ -f a ] || [ -f b ]            # OR: if either is true
! [ -f file ]                   # NOT
```

And **Security Group rules**, **IAM policy evaluation**, **subnet mask calculation** — all
run on top of these three operations. The AND gate from Phase 0 is exactly the
`IP AND netmask = network address` operation from the network map.

## 0.3.2 Derived gates: NAND, NOR, XOR `[concept]`

### NAND = NOT + AND

The inverse of AND. 0 if all inputs are 1; 1 in every other case.

| A | B | OUTPUT |
|---|---|---|
| 0 | 0 | 1 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | **0** |

**NAND has a special power: it is a universal gate.** Using only NAND gates you can build
AND, OR, NOT and therefore **any logic circuit**.

```
NOT A        = A NAND A
A AND B      = NOT(A NAND B)  = (A NAND B) NAND (A NAND B)
A OR B       = (A NAND A) NAND (B NAND B)
```

**Why this matters:** In chip manufacturing, stamping out one gate type billions of times is
much cheaper and more reliable than mixing different types. Also, in CMOS technology NAND
requires **fewer transistors** than AND (NAND 4 transistors, AND 6 — because AND is really
NAND + NOT). That is why NAND and NOR are the dominant gates in real chips.

> **You can forget this detail for now, but keep one thing:** in hardware, "fewer
> transistors" always means "cheaper, less heat, faster." This principle becomes critical in
> Phase 2 when comparing SRAM and DRAM.

### NOR = NOT + OR

The inverse of OR. It is also a universal gate.

| A | B | OUTPUT |
|---|---|---|
| 0 | 0 | **1** |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 0 |

### XOR (exclusive or) — the most interesting gate

1 if the inputs **differ**, 0 if they are the same.

| A | B | OUTPUT |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | **1** |
| 1 | 0 | **1** |
| 1 | 1 | 0 |

Read XOR as the question "are A and B **different**?" That reading explains why XOR is
everywhere:

- **Addition:** 1+1 = 0 (carry 1). XOR does exactly this → next section
- **Comparison:** are two values equal? XOR them all; if the result is zero, they are equal
- **Parity / error checking:** the XOR of the bits tells you whether there is an odd number of 1s
- **RAID 5:** the parity disk is computed with exactly XOR → we'll see it in Phase 3.5
- **Encryption:** `data XOR key = ciphertext`, `ciphertext XOR key = data`. Reversible.

> **🤔 Think 0.5**
> In RAID 5, data can be recovered when one of 3 disks fails. The disks hold these bits:
> Disk A = `1`, Disk B = `0`, Parity = `A XOR B = 1`.
> Disk A failed. You only have B (`0`) and Parity (`1`). How do you recompute A?
> *(Answer: at the end of the phase)*

## 0.3.3 From gates to an adder circuit `[concept]`

Now the most satisfying link in the chain: **how does arithmetic come out of gates?**

### Half adder

Add two one-bit numbers. There are four possible cases:

```
0 + 0 = 0    (sum 0, carry 0)
0 + 1 = 1    (sum 1, carry 0)
1 + 0 = 1    (sum 1, carry 0)
1 + 1 = 10   (sum 0, carry 1)  ← "two" in binary = 10
```

Now look at the "sum" and "carry" columns separately:

| A | B | Sum | Carry |
|---|---|---|---|
| 0 | 0 | 0 | 0 |
| 0 | 1 | 1 | 0 |
| 1 | 0 | 1 | 0 |
| 1 | 1 | 0 | 1 |

**The sum column = the XOR table. The carry column = the AND table.**

So:

```
Sum   = A XOR B
Carry = A AND B
```

Two gates. That is addition. All of arithmetic starts here.

![Figure 0.4 — Half adder circuit](../diagrams/png/hw-0-04-half-adder.png)
*Figure 0.4 — Half adder: the same two inputs feed both an XOR and an AND gate; XOR produces
the sum, AND produces the carry.*

### Full adder and 64-bit addition

The half adder has one shortcoming: it cannot take into account **an incoming carry**. In
multi-digit addition, each digit receives a carry from the previous one.

A circuit that handles three inputs (A, B, carry-in) is called a **full adder**. It is built
from two half adders + one OR gate.

To add two 64-bit numbers, you **chain 64 full adders**: each one's carry-out goes to the
next one's carry-in. This is called a **ripple-carry adder**.

![Figure 0.5 — Ripple-carry adder: the carry chain](../diagrams/png/hw-0-05-ripple-carry.png)
*Figure 0.5 — The carry bit moves in sequence from the rightmost digit to the leftmost. The
last bit cannot get its correct value until all 63 digits before it have been computed.*

### And here is the physical ceiling on clock speed

The critical property of ripple-carry: **the leftmost bit cannot know its correct value until
the carry from the rightmost bit arrives.** The carry has to travel through 64 digits in
sequence.

Every gate has a **propagation delay** — after its input changes, its output takes time to
settle. Say one full-adder stage takes 30 picoseconds:

```
64 digits × 30 ps = 1,920 ps = 1.92 nanoseconds

If the CPU wants to finish this addition within one clock cycle,
a clock cycle must last at least 1.92 ns.

Maximum frequency = 1 / 1.92 ns ≈ 520 MHz
```

**520 MHz.** But modern CPUs run at 3–5 GHz. How?

Two ways:

1. **Smarter circuits.** Real CPUs don't use ripple-carry; they use circuits such as
   *carry-lookahead* and *carry-select* that predict the carry in advance. These spend far
   more transistors to cut the delay from 64× to ~6×. (The classic engineering trade-off:
   **spend area/power to buy time.**)
2. **Pipelining.** Instead of finishing the work in one cycle, split it into pieces and do
   each piece in a separate cycle. All of Phase 1.4 is about this.

> **This is the most important sentence of Phase 0:** The limit on clock speed is not
> arbitrary. A clock cycle cannot be shorter than the time it takes the **longest
> combinational path** in the circuit to settle. All of CPU design is a struggle to shorten
> that path. When we ask "why does the pipeline exist?" in Phase 1, the answer will be here.

> **🤔 Think 0.6**
> According to the calculation above, a 64-bit ripple-carry adder allowed ~520 MHz.
> What frequency would the same circuit allow at 32 bits? Does this mean "32-bit CPUs can
> reach higher clock speeds"? Is that what happened with real CPUs?
> *(Answer: at the end of the phase)*

---
# 0.4 Flip-Flops and Latches — The Basis of Memory

Every circuit we have built so far has one thing missing: **it has no memory.**

Give an AND gate inputs and it produces an output. Remove the inputs and the output
disappears too. The gate doesn't remember "what was there last time." Such circuits are
called **combinational logic**: the output depends **only** on the current input.

But a computer needs to remember. It needs to hold a variable, increment a counter, store
"which instruction am I on right now."

Circuits that can remember are called **sequential logic**: the output depends on the
current input **and on the past**.

So how do you get memory out of memoryless gates? With a single trick: **feedback.**

## 0.4.1 The SR latch — locking a bit `[concept]`

Take two NOR gates. Connect each one's output to the other's input. Cross-coupling.

![Figure 0.6 — NOR-based SR latch](../diagrams/png/hw-0-06-sr-latch.png)
*Figure 0.6 — Cross feedback between two NOR gates. Q and Q̄ (not-Q) are always the inverse
of each other, and they keep their last state even after the inputs are removed.*

There are two inputs:

- **S** (Set) → makes the output 1
- **R** (Reset) → makes the output 0

Behavior:

| S | R | Q (output) | What happened |
|---|---|---|---|
| 0 | 0 | *unchanged* | **Hold state** — keeps the last value |
| 1 | 0 | 1 | Set |
| 0 | 1 | 0 | Reset |
| 1 | 1 | *invalid* | Both at once is not allowed (undefined state) |

**The magic is not in the third row but in the first.** When you give S=0, R=0, the circuit
**keeps its last value.** The inputs are neutral, but the output stays.

Why does it stay? Because Q's output is connected to the other gate's input, and that gate's
output is connected back to Q's input. The two gates **feed each other.** If Q=1, that 1
travels around the circuit and holds Q at 1. A self-confirming loop.

**This is one bit of memory.** Two gates, thanks to feedback.

> **⚠️ Common misconception:** "Memory *writes* information somewhere." At the electronic
> level, memory doesn't write information anywhere — it **keeps information circulating in an
> electrical loop.** That's why it is lost when power is cut. When we say "volatile memory"
> in Phase 2, this is exactly what we mean: if the power feeding the circulation goes away,
> the circulation stops and the information vanishes.

## 0.4.2 The D flip-flop and the clock's real job `[concept]`

The SR latch has a practical problem: the inputs can change the output **at any moment**. On
a chip where billions of circuits are connected together, that means chaos — if one circuit
reads another's output before it has settled, it gets the wrong value.

The solution: **synchronize the moments at which changes are allowed.**

The **D flip-flop** (D = data) does this. It has two inputs:

- **D** — the value to store
- **CLK** (clock) — the "store now" signal

Its behavior in one sentence: **on the rising edge of the clock signal (the instant it goes
from 0 to 1), take the value on D and hold it until the next edge.** As long as no clock
edge arrives, no matter how much D changes, the output doesn't move.

![Figure 0.7 — D flip-flop and edge triggering](../diagrams/png/hw-0-07-d-flipflop.png)
*Figure 0.7 — Timing diagram: even though the D signal keeps changing, Q updates only on the
rising edges of the clock.*

> **This is the clock's real job.** It is not, as commonly believed, "an engine that makes the
> CPU faster." The clock is a **metronome**: by updating the state of billions of circuits at
> the same moment, at regular intervals, it keeps the system consistent.
>
> And now the calculation at the end of Phase 0.3 falls into place: **the time between two
> clock edges must be long enough for the combinational circuit in between to settle.** If
> it isn't, the flip-flop stores a value that is still unstable — that is corrupted data at
> the hardware level. This is exactly the physical reason systems crash when overclocked.

**❓ A question that comes to mind: What exactly is the difference between a latch and a flip-flop?**

- A **latch** is *level*-sensitive: it passes the input through **for as long as** the clock
  is 1 (transparent).
- A **flip-flop** is *edge*-sensitive: it samples **at the instant** the clock changes and is
  closed the rest of the time.

In practice, almost all modern synchronous designs use flip-flops, because "exactly when it
updates" is precise. In everyday speech the two are often used interchangeably; knowing the
distinction is enough — don't go deeper.

## 0.4.3 From flip-flop to register `[concept]`

One D flip-flop = 1 bit.

**Put 64 D flip-flops side by side and connect them all to the same clock → a 64-bit
register.** On the clock edge, all 64 bits update at the same moment.

This is the register we'll talk about in Phase 1. CPU registers like `RAX` and `RBX` are
physically exactly this: 64 flip-flops tied to the same metronome.

And now something becomes clear: **why are registers so fast and so few?**

- **Fast:** because they sit right next to the ALU, with wire distances measured in
  micrometers. A signal's round trip takes less than one clock cycle.
- **Few:** because each bit needs ~20+ transistors and takes up a huge amount of space. x86-64
  has only 16 general-purpose registers — about 128 bytes in total.

## 0.4.4 SRAM and DRAM — the root of the entire memory hierarchy `[concept]`

This is Phase 0's **doorway into Phase 2** and the item with the highest payoff in this phase.

There are two ways to store a bit, and the difference between them single-handedly explains
the entire memory hierarchy in a computer.

### SRAM (Static RAM) — the material of cache

Structure: the latch idea above, built with transistors. Typically **6 transistors** (a 6T
cell) hold one bit. Two cross-coupled inverters + two access transistors.

- **Static:** as long as power is applied, it keeps its state without doing anything. It
  doesn't need refreshing.
- **Fast:** to read, you just look at the state of the feedback loop. Under ~1 ns.
- **Expensive and large:** 6 transistors per bit. 1 MB of SRAM ≈ 50 million transistors.
- **Power-hungry:** 6 transistors are powered continuously, with constant leakage current.

### DRAM (Dynamic RAM) — the material of main memory

Structure: **1 transistor + 1 capacitor** (1T1C). Charge in the capacitor = 1, no charge = 0.

- **Dense and cheap:** 2 elements instead of 6. ~6–10× more bits fit in the same area.
- **Slow:** reading requires measuring the capacitor's charge — an analog operation that
  takes time.
- **Dynamic (and here's the problem):** the capacitor's charge **leaks.** It disappears within
  milliseconds. So DRAM has to be **refreshed** thousands of times per second: every row is
  read and written back. During refresh, that memory region cannot be accessed.
- **Destructive read:** reading DRAM drains the capacitor; the value read must be written
  back immediately.

### Comparison — and the consequence

| Property | SRAM | DRAM |
|---|---|---|
| Elements per bit | ~6 transistors | 1 transistor + 1 capacitor |
| Access time (roughly) | ~0.5–2 ns | ~50–90 ns |
| Density | Low | High (~6–10×) |
| Cost per bit | High (~10–100×) | Low |
| Needs refresh? | No | Yes (constantly) |
| Where it is used | **L1/L2/L3 cache, registers** | **Main memory (RAM sticks)** |

> **And here is the birth of the memory hierarchy:**
>
> SRAM is fast but expensive and bulky → we can only use a little → **cache is small.**
> DRAM is cheap and dense but slow → we can use plenty → **RAM is large.**
>
> This is not an engineering preference; it is an **economic necessity.** Building a 128 GB
> server entirely out of SRAM is technically possible — the price and size would be absurd,
> and it wouldn't work anyway because of heat.
>
> All of Phase 2 is built on managing this single tension: **what is fast is small; what is
> large is slow.** Every instance choice in the cloud is a reflection of this tension.

![Figure 0.8 — SRAM 6T cell and DRAM 1T1C cell](../diagrams/png/hw-0-08-sram-vs-dram.png)
*Figure 0.8 — Left: SRAM's cross-coupled 6-transistor cell; right: DRAM's cell made of a single
transistor and a capacitor. The area difference translates directly into cost and capacity
differences.*

> **🤔 Think 0.7**
> You learned that DRAM must be refreshed. A server has 256 GB of DRAM, and every cell must
> be refreshed once every 64 ms. During refresh, that rank cannot be accessed.
> How does this affect performance as RAM capacity grows? And what do you say to the sentence
> "more RAM is always better"?
> *(Answer: at the end of the phase)*

> **🔧 See it on your machine** (optional)
>
> ```bash
> lscpu | grep -i cache          # the CPU's amount of cache (SRAM)
> free -h | head -2              # the system's amount of RAM (DRAM)
> ```
> **Expected output:** total cache is typically in the MB range (e.g. L3: 32 MiB), while RAM
> is in the GB range (e.g. 16 Gi). The ~500–1000× difference is a direct consequence of the
> cost table above.

---
---

# Phase 0 — Answers to the Think Questions

## Answer 0.1 — Why is `chmod 755` `rwxr-xr-x`?

**Question:** How many bits is one octal digit, and why does `755` give exactly those permissions?

Because 8 = 2³, **one octal digit is exactly 3 bits.** (By the same logic, one hex digit was
4 bits — 16 = 2⁴.)

Unix permissions come in groups of exactly 3: `read, write, execute`. That is not a
coincidence; it is design:

```
7    5    5
↓    ↓    ↓
111  101  101

7 = 111 = rwx  → owner:   read, write, execute
5 = 101 = r-x  → group:   read and execute; no write
5 = 101 = r-x  → others:  same

Result: rwxr-xr-x
```

The weight of each bit: `r`=4 (2²), `w`=2 (2¹), `x`=1 (2⁰). So `6 = 4+2 = rw-`,
`5 = 4+1 = r-x`, `7 = 4+2+1 = rwx`.

**Why octal and not hex?** Because a permission group is 3 bits and octal holds exactly
3 bits. Hex would hold 4 bits and the groups would not line up. The right base is chosen
according to the structure of the problem.

**Related section:** 0.1.3

---

## Answer 0.2 — A 500 GB bill, a 466G measurement

**Question:** AWS bills "500 GB", `du -sh` says `466G`. Which one is wrong?

**Both are correct. They use different bases.**

- AWS reports Data Transfer metrics in **GB (decimal, 10⁹)**.
- `du -h` shows its output in **GiB (binary, 2³⁰)** but labels it `G`.

The calculation:

```
500 GB (decimal) = 500 × 10⁹ = 500,000,000,000 bytes
Convert to GiB:    500,000,000,000 ÷ 1,073,741,824 = 465.66 GiB
                                                     ≈ 466G  ✓
```

The entire 7.4% gap is the base difference. There is neither bill inflation nor measurement
error.

**Practical lesson:** When comparing a cloud bill with your own measurement, **align the
units first.** When you see an "inexplicable" 7% gap, the first thing to suspect is GiB/GB
confusion. If you use `du -h --si`, `du` also shows decimal and the numbers match.

**Related section:** 0.1.4

---

## Answer 0.3 — If 64-bit addresses 16 EiB, why doesn't anyone install 16 EiB of RAM?

**Question:** Is it physically impossible, or is there another limit?

**There are several limits, and none of them is "64 bits isn't enough":**

**1. The virtual address space doesn't use all 64 bits anyway.** Modern x86-64 processors do
not use all 64 bits of the address; they typically use **48 bits** (57 bits on some newer
generations — 5-level page tables).

```
2⁴⁸ = 256 TiB of virtual address space
2⁵⁷ = 128 PiB (newer generations, 5-level paging)
```

The reason: every additional address bit enlarges the page-table layers needed for address
translation (we'll see this in Phase 2.5). Nobody pays hardware costs for bits nobody needs.

**2. There are even fewer physical address bits.** A CPU's memory controller typically
supports 46–52 bits of physical address → 64–4096 TiB. Again, far below 16 EiB.

**3. The real limit is physical:** number of DIMM slots, number of channels, capacity per
DIMM, power and heat. Around 2024, the largest servers reach ~24 TiB of RAM — and even that is
extraordinarily expensive.

**4. Economics:** the cost of 16 EiB of RAM would be many times the world's total DRAM
production.

> **General lesson — and it will come up again and again in this map:** an *architectural
> limit* and a *practical limit* are different things. The gap between "theoretically
> supported" and "actually achievable" is where engineering lives.

**Related section:** 0.1.5

---

## Answer 0.4 — Is the small or the large instance faster single-threaded?

**Question:** `c7i.2xlarge` (8 vCPUs) vs `c7i.16xlarge` (64 vCPUs), both "up to 3.2 GHz".
Which does better on a single-threaded benchmark?

**Usually the smaller one — or at best they are equal.** It feels counterintuitive; the reason
is the heat budget from 0.2.3.

A CPU has a **total** power/heat budget (TDP). The turbo/boost frequency depends on how many
cores are active at that moment and how hot the chip is:

- If few cores are active → plenty of heat budget → a single core can reach a high frequency
- If many cores are active → the heat budget is shared → they all run at a lower frequency

`c7i.16xlarge` occupies a much larger portion (or all) of a physical socket. If the neighboring
vCPUs are under load, the chip runs hot and **your single thread also stays at a lower
frequency.**

But three caveats here:

1. **There is no rule that "smaller is always faster within the same family."** Very small
   instances (`.large`, `.xlarge`) share a socket with others → **noisy neighbor** (Phase 7.4).
   A neighbor's load lowers your frequency and pollutes your L3 cache.
2. **That's why mid-sized instances** (`.2xlarge`–`.4xlarge`) are generally the most
   consistent for single-thread performance: big enough to escape sharing, not big enough to
   exhaust the heat budget.
3. **Bare metal or `.metal`** instances have no neighbors — behavior is predictable, but if you
   load all the cores you still drop to the base clock.

**Cloud decision `[application]`:** If you have a single-thread-sensitive workload (the single
query path of a classic relational database, single-threaded parts of the JVM, some ETL
steps), **growing the instance may not improve performance and may even reduce it.** The
right move is to choose a family with a higher base/turbo frequency (e.g. the `c` or `z`
family), not a bigger instance.

**Related section:** 0.2.3

---

## Answer 0.5 — Recovering a lost disk in RAID 5 with XOR

**Question:** B = `0`, Parity = `1`. What was A?

```
Parity definition:  P = A XOR B
We have:            P = 1, B = 0

XOR's reversible property:  A = P XOR B

A = 1 XOR 0 = 1  ✓
```

Check: at the start we had A=1, B=0. `1 XOR 0 = 1 = P`. It holds.

**Why this works:** XOR is its own inverse. `X XOR Y XOR Y = X`. So if you XOR the parity with
the remaining disks, you get the lost disk back. This property comes from a single gate, and
all of RAID 5 rests on it.

**But see its limit too:** this trick works for **a single missing value**. If two disks fail
at once, you have one equation with two unknowns — it cannot be solved. That is the
mathematical reason for RAID 5's "single-disk tolerance." It is also why RAID 6 uses two
different parity calculations (one XOR, the other Reed-Solomon based).

**Related section:** 0.3.2 · **Continued in:** Phase 3.5

---

## Answer 0.6 — Does a 32-bit adder allow a higher clock?

**Question:** 64 digits × 30 ps = ~520 MHz. What does 32-bit give? Is that what happened in reality?

**The calculation:**

```
32 digits × 30 ps = 960 ps = 0.96 ns
Maximum frequency = 1 / 0.96 ns ≈ 1.04 GHz

So, theoretically, about twice as much.
```

**But that is not what happened in reality**, and the reason is instructive:

1. **Real CPUs don't use ripple-carry.** Circuits like carry-lookahead compute the carry in
   parallel; the delay grows **logarithmically, not linearly**, with the number of digits. The
   difference between 64-bit and 32-bit is much smaller than 2×.
2. **The adder is not the longest path.** In a modern CPU, the critical path is usually the
   multiplier, cache access or the bypass/forwarding network. Beyond a certain point, speeding
   up the adder doesn't change the overall frequency.
3. **Historical evidence:** the 32-bit Pentium 4 reached higher clock speeds than its 64-bit
   contemporaries — but that came not from address width but from its **very deep pipeline**
   (we'll see it in Phase 1.4; that design failed for other reasons).

> **The lesson to take away:** if you want to speed up a system, you have to find **the longest
> path** (the critical path / the bottleneck). Improving anything else changes nothing. This
> principle will show up as the critical path in hardware and as bottleneck analysis in
> workloads — all of Phase 7.5 is about it.

**Related section:** 0.3.3

---

## Answer 0.7 — DRAM refresh and "is more RAM always better?"

**Question:** How does refresh affect performance as capacity grows?

**First the mechanism:** DRAM refresh is done per **row**, and all rows must be scanned within a
refresh window (typically 32–64 ms). As capacity grows, the number of rows grows, so more
refreshes are squeezed into the same window. During refresh that rank is busy and normal
reads/writes wait.

In typical server configurations the effect is **small** (roughly on the order of 2–5% of
bandwidth), but it is not zero, and it grows with capacity/density. That's why techniques
like "fine-granularity refresh" are used on high-density DIMMs.

**The real answer to "is more RAM always better?" is clear: no.** But refresh is the weakest
reason. The real reasons:

1. **Cost.** Memory-optimized instances (`r`, `x` families) are much more expensive per vCPU.
   Unused RAM is pure waste.
2. **NUMA.** More RAM usually means more sockets; memory access is no longer uniform, and a
   badly placed workload **slows down** (Phase 2.7).
3. **Masking the problem.** "Fixing" a memory leak by growing RAM postpones the crash by weeks
   and makes diagnosis harder (Phase 2.6).
4. **The wrong bottleneck.** If your application is I/O-bound, adding RAM changes nothing — only
   the bill grows. (The one exception: extra RAM goes to the page cache and reduces disk reads.
   That is a real gain, but it is limited and depends on the workload.)

**Related section:** 0.4.4 · **Continued in:** Phase 2.4, 2.6, 2.7

---
# Phase 0 — Frequently Asked Questions

This section collects questions that naturally come up while working through the phase but
don't fit into the narrative without breaking its flow.

### Q1. Do I really need to know this low a level? I'm never going to draw a logic gate.

True, you won't. But the payoff of this phase is not drawing gates; it is **four lasting
intuitions**:

1. Every limit in a computer (4 GiB, 64 bytes, 4096 bytes) is a power of 2, and there is a reason for it
2. The trade-off between speed and capacity is **physical**, not a preference (SRAM/DRAM)
3. The ceiling on clock speed is **physical** → this is why the cloud moved to horizontal scaling
4. GiB ≠ GB, and that difference shows up on your bill

Without these four, Phase 2 and Phase 7 become memorization. With them, they become derivable.

### Q2. Why do some places write `2^10 = 1024` instead of saying `1000`? Who decided?

Nobody "decided"; it arose naturally. Because memory addressing is binary, memory capacities
are inevitably powers of 2 — 1024 is a round number (for hardware). 1000 is an odd number for
hardware.

Early on, engineers said "1024 ≈ 1000, let's call it kilo." Disk manufacturers, on the other
hand (since their capacities didn't have to be binary), used the real decimal kilo. The two
traditions collided.

In 1998 the IEC standardized the `KiB`/`MiB`/`GiB` prefixes. The standard is correct, but
**its adoption remained partial**: Linux tools mostly calculate in binary and print decimal
labels (`df -h` → `G`, but really GiB). That's why the confusion still continues 25 years
later.

**Practical rule:** RAM is always binary. Disk capacity is usually decimal. The network is
always decimal **and in bits**. When in doubt, check with `df -H` / `du --si`.

### Q3. When people say "64-bit," are they always talking about the same thing?

No — at least four different things go by the same name, and they are **independent of each
other**:

| What | Typical value | What it determines |
|---|---|---|
| Register width | 64 bits | Size of number processed in one operation |
| Virtual address width | 48 (or 57) bits | Address space a process can see |
| Physical address width | 46–52 bits | Maximum RAM that can be installed |
| Data bus width | 64 bits/lane | Data pulled from memory at once |

A CPU is "64-bit," but its virtual address can be 48 bits and its physical address 46 bits.
That is not a contradiction — each was chosen according to a different cost/benefit balance.

### Q4. Will quantum computers end the binary system? Is learning this a waste?

No. Quantum computers are not replacing general-purpose computers; they are **special-purpose
accelerators** that offer an advantage on certain classes of problems (factorization, some
simulations, some optimizations). A web server, a database or an operating system does not run
on quantum hardware, and it wouldn't make sense for it to.

Every instance you see in the cloud will be classical and binary for the foreseeable future.
(Services like AWS Braket offer *access* to quantum hardware — they don't replace
general-purpose compute.)

### Q5. Do ARM and x86 use different binary systems?

No, both use the same binary system. The difference is at the level of the **instruction set
architecture** (ISA): which instructions exist, how they are encoded, how many registers there
are. We'll see this in Phase 1.6.

Bits, bytes, hex, binary arithmetic — all are the same on both.

One small exception is **endianness** (byte order): x86 is little-endian; ARM supports both
but in practice (Linux, including Graviton) runs little-endian. That's why endianness causes
no problems in practice when moving to Graviton.

### Q6. Apart from the `0x` prefix, how do I recognize hex?

There are several notations depending on context:

| Notation | Where |
|---|---|
| `0xFF` | C, Python, Go, JavaScript, most languages and stack traces |
| `FFh` or `$FF` | Assembly (depending on dialect) |
| `\xFF` | Byte escape inside a string |
| `#FF0000` | CSS color code (3 bytes: R, G, B) |
| `ff:ff:ff:ff:ff:ff` | MAC address (6 bytes) |
| `2001:db8::1` | IPv6 address (16 bytes, hex groups) |

All the same thing: binary data read in 4-bit groups.

### Q7. I finished this phase but logic gates haven't fully clicked. Should I continue?

Yes, continue — **on one condition**: if you can say the following three sentences in your own
words, that is enough.

1. "Gates have no memory; memory appears when feedback is added."
2. "A clock cycle cannot be shorter than the time the circuit in between takes to settle."
3. "SRAM needs 6 transistors, DRAM 1 transistor + 1 capacitor — that's why cache is small and RAM is large."

These three are the only things carried into Phases 1, 2 and 6. Not having memorized XOR's
truth table is not a problem; you can look it up when you need it.

---

# Phase 0 — Test Yourself

The answers are right below. Try to answer all of them first, then look.

**1.** What are the decimal and hex equivalents of the binary number `1101 0110`?

**2.** What is `0x3F` in decimal? How many bits does it hold?

**3.** Explain in one sentence why computers count in binary. ("Electricity is on/off" is not
accepted.)

**4.** How many TiB does a 2 TB disk show up as in the operating system?

**5.** Theoretically, how many MB of data pass through a 25 Gbps network link in 1 second?

**6.** What is the most memory a 32-bit process can use on a server with 128 GiB of RAM? Why?

**7.** "A 64-bit CPU is twice as fast as a 32-bit one" — true? Explain.

**8.** How did the collapse of Dennard scaling shape cloud architecture? Build the chain.

**9.** Write the truth table of the AND gate. Where do you use this gate in network calculations?

**10.** Why is NAND called a "universal gate"?

**11.** In a half adder, which gates produce the sum and carry bits?

**12.** What determines the physical upper limit of a CPU's clock frequency?

**13.** What is the difference between combinational and sequential circuits? Which one gives
rise to memory, and how?

**14.** What is the real job of the clock signal? ("Making the CPU faster" is not accepted.)

**15.** Compare SRAM and DRAM in terms of elements per bit, speed, refresh requirement and where
they are used.

**16.** Answer "why is cache so small?" in one sentence, with a physical reason.

**17.** What permissions does `chmod 640` give? Show it by converting to binary.

**18.** Why does a turbo boost frequency drop when all cores are loaded?

---

## Answer Key

**1.** `1101 0110` → **214** (128+64+16+4+2), hex: `1101`=D, `0110`=6 → **`0xD6`**
· *section 0.1.3*

**2.** `0x3F` = 3×16 + 15 = **63**. Two hex digits = 8 bits = **1 byte**. (`0x3F` =
`0011 1111`) · *section 0.1.3*

**3.** **Noise margin.** Thanks to the wide "forbidden zone" between two voltage levels, a
signal is read correctly even when badly distorted; using more levels would make the error
rate unacceptable. · *section 0.1.1*

**4.** 2 TB = 2×10¹² bytes. `2,000,000,000,000 ÷ 1,099,511,627,776` = **~1.82 TiB**
· *section 0.1.4*

**5.** `25,000,000,000 bits/s ÷ 8` = 3,125,000,000 bytes/s = **3,125 MB/s** (decimal) ≈ 2,980
MiB/s. After protocol overhead, in practice ~2,900–3,050 MB/s. · *section 0.1.4*

**6.** **~3 GiB** (typical user space on Linux; the theoretical ceiling is 4 GiB, with the top
1 GiB reserved for the kernel). Reason: a 32-bit address can point to only 2³² = 4 GiB of
distinct locations. The amount of RAM in the server doesn't change that. · *section 0.1.5*

**7.** **False.** The gain of 64-bit is not speed but **address space**. Workloads that operate
on 64-bit numbers (crypto, big-integer math) do gain; for most applications there is no
difference. Because pointers double in size, cache pressure increases and in some cases it
can **slow things down**. · *section 0.1.5*

**8.** Dennard collapse (~2005) → raising clock speed became impossible because of heat →
clock stuck at ~3–5 GHz → performance gains could only come from **core count** → software had
to become parallel → vertical scaling hit the ceiling → **horizontal scaling** became the only
way → the elastic/distributed architecture of the cloud. · *section 0.2.3*

**9.** `0,0→0` · `0,1→0` · `1,0→0` · `1,1→1`. In networking: **`IP AND subnet mask = network
address`**. For example `192.168.1.50 AND 255.255.255.0 = 192.168.1.0`. · *section 0.3.1*

**10.** Because using **only NAND gates** you can build NOT, AND, OR and therefore any logic
circuit. Stamping out a single gate type is cheaper and more reliable in manufacturing; also,
in CMOS, NAND requires fewer transistors than AND. · *section 0.3.2*

**11.** **Sum = A XOR B**, **Carry = A AND B**. · *section 0.3.3*

**12.** The **settling time of the longest combinational path** in the circuit (the critical
path). If a clock cycle is shorter than that, the flip-flop samples a value that is still
unstable and the system breaks. This is the physical reason overclocking crashes happen.
· *section 0.3.3, 0.4.2*

**13.** **Combinational:** the output depends only on the current input; no memory (gates).
**Sequential:** the output depends on the input **and the past**. Memory is obtained by
**feeding** a gate's output **back** into its input (feedback) — the SR latch is the simplest
form of this. · *section 0.4.1*

**14.** **Synchronization.** The clock is a metronome that keeps the system consistent by
updating the state of billions of circuits at the same moment, at regular intervals. Speed is
a consequence of this, not its purpose. · *section 0.4.2*

**15.**

| | SRAM | DRAM |
|---|---|---|
| Elements/bit | ~6 transistors | 1 transistor + 1 capacitor |
| Speed | ~0.5–2 ns | ~50–90 ns |
| Refresh | Not needed | Constantly needed |
| Cost/bit | High | Low |
| Use | Registers, L1/L2/L3 cache | Main memory |

· *section 0.4.4*

**16.** Because SRAM needs ~6 transistors per bit, which makes it many times more expensive and
bulkier than DRAM — **making it large is economically impossible.** · *section 0.4.4*

**17.** `640` → `110 100 000` → **`rw-r-----`**: the owner reads+writes, the group only reads,
others can do nothing. · *section 0.1.3 / Answer 0.1*

**18.** The chip has a fixed **power/heat budget** (TDP). When all cores are active, that budget
is shared and each core has to run at a lower frequency. When few cores are active, the budget
is plentiful, so a single core can climb to the high turbo frequency.
· *section 0.2.3 / Answer 0.4*

---

### Scoring

| Correct answers | What to do |
|---|---|
| 15–18 | Move on to Phase 1. You've laid a solid foundation. |
| 11–14 | Move on to Phase 1, but reread the sections your wrong answers point to. |
| 7–10 | Be sure to review 0.1.4, 0.1.5 and 0.4.4 — they are prerequisites for Phase 2. |
| 0–6 | Work through the phase again from the start. Don't rush; a gap here grows exponentially in Phase 2. |

**Which section to return to for each missed question:**

| Question | Section |
|---|---|
| 1, 2, 17 | 0.1.3 — Base conversion |
| 3 | 0.1.1 — Why binary |
| 4, 5 | 0.1.4 — KB/KiB and bits/bytes |
| 6, 7 | 0.1.5 — 32/64-bit |
| 8, 18 | 0.2.3 — Moore, Dennard, heat budget |
| 9, 10, 11 | 0.3 — Logic gates |
| 12 | 0.3.3 + 0.4.2 — Critical path and clock |
| 13, 14 | 0.4.1, 0.4.2 — Latch and flip-flop |
| 15, 16 | 0.4.4 — SRAM vs DRAM |

---

# Phase 0 — Closing and Bridge to Phase 1

## What you carry out of this phase

When you finish this phase, you have **four tools**. Every phase after this one will use them:

**1. The "every limit is a power of 2" reflex.**
4 GiB, a 64-byte cache line, a 4096-byte page, PCIe x16, a `/24` subnet — when you see a
strange number somewhere, your first question will now be "which power of 2 is this, and why?"

**2. The law of "what is fast is small; what is large is slow."**
You learned it from the SRAM/DRAM distinction. Phase 2 is its cache-hierarchy form, Phase 3
its storage form, Phase 7 its instance-selection form.

**3. "Critical path" thinking.**
Speeding up a system = finding the longest path and shortening it. Improving anywhere else is
wasted effort. All of the bottleneck analysis in Phase 7.5 is an application of this idea.

**4. The "physics determines architecture" connection.**
Heat → the clock stalled → cores multiplied → horizontal scaling → cloud. The cloud is not a
fashion; it is the architectural answer to a physical constraint.

## Where Phase 1 connects to this

In Phase 1 we go inside the CPU. You will see these connections:

| What you learned in Phase 0 | What happens in Phase 1 |
|---|---|
| Flip-flop (0.4.2) | 64 of them side by side → **register** (1.1) |
| Adder circuit (0.3.3) | Together with other operations → **ALU** (1.1) |
| Clock = metronome (0.4.2) | Drives the fetch-decode-execute cycle (1.2) |
| Critical path → clock ceiling (0.3.3) | The reason the **pipeline** exists (1.4) |
| Heat budget → many cores (0.2.3) | The **core, thread, vCPU** concepts (1.5) |
| Transistor-budget trade-off (0.3.2) | The design-philosophy difference between x86 and ARM (1.6) |

> **Phase 0 output — ask yourself before continuing:**
> "Can I explain the chain from a transistor all the way to a CPU register without skipping a
> single link?"
>
> The chain: **transistor (switch) → logic gate (decision) → adder (arithmetic) +
> feedback (memory) → flip-flop (synchronous memory) → register (64 bits) → CPU.**
>
> If you can build this chain, you're ready for Phase 1.

---

> **Navigation:** [◀ Contents](README_en.md) · **Phase 0** · [Phase 1 — CPU Architecture ▶](Phase_1_CPU_Architecture.md)
