# Phase 1 — Addressing: How Do We Find a Machine?

> **Navigation:** [◀ Phase 0 — Mental Model](Phase_0_Mental_Model.md) · **Phase 1** · [Phase 2 — Subnetting and CIDR ▶](Phase_2_Subnetting_and_CIDR.md)

---

## Where we come from

At the end of Phase 0 we left you a question: if MAC changes at every hop, **which MAC** does your machine
write when it sends a packet to a remote server? We start answering that question in this phase — but the
full answer comes in Phase 3, because first you need to know the addresses **themselves**.

You carry three things from Phase 0, and all three are directly useful here:

- **Every layer has its own address.** MAC at L2, IP at L3, port at L4. This phase opens up all three of them
  one by one — what they are, what they look like, which question each one answers.
- **MAC is local, IP is end to end** (0.2.2). Here you will learn the *reason* behind that sentence: why MAC,
  by its very structure, cannot be routed.
- **A header points to the next one** (0.2.2). The **port** in the L4 header is the answer to the question
  "which application should I deliver this to" — and in this phase we will get to know the port with exactly
  that eye.

## The question of this phase

> *"There are billions of devices in the world. How does my packet reach **exactly one** of them — and even
> exactly one **application** on that device?"*

The answer is not a single address but a **three-step funnel**: MAC says "which device on this local
network", IP says "which machine on the internet", port says "which application on that machine". Delivery
is not complete without all three, and because all three live in **different layers**, all three **break in
different ways**.

This distinction will keep coming at you in the field: "I can ping the machine but I cannot connect to the
service" means exactly "the IP is right, the port is wrong/closed". "I can reach the machines on my own
network but I cannot get out to the internet" means exactly "L2 works, there is no L3 path". Being able to
tell the address steps apart is being able to **translate** these sentences.

By the end of this phase, when you look at an `ip addr` output you will read the lines not from memory but
**with their meanings**; and you will know where a machine gets its IP from (DHCP) and what it looks like
when that mechanism breaks.

---

## By the end of this phase

- You will be able to explain what a MAC address is, how many bits it has and **why it is only meaningful on
  the local network**
- You will be able to take apart the anatomy of an IPv4 address (32 bits, 4 octets, dotted-decimal) and do
  the binary ↔ decimal conversion by hand
- You will be able to describe the distinction between the **network** part and the **host** part of an
  address — this is the whole foundation of Phase 2
- You will recognise the RFC 1918 private ranges (10/8, 172.16/12, 192.168/16) and be able to explain why a
  private IP **cannot** be routed on the internet
- You will be able to tell port and socket apart, recognise the well-known ports, and know exactly what the
  "address already in use" error means
- You will be able to say at an awareness level why IPv6 exists
- You will be able to walk through DHCP's DORA cycle step by step; you will know that DHCP hands out not only
  an IP but also **mask + gateway + DNS**; and you will recognise the signature a broken DHCP leaves
  (169.254.x.x / no IP at all)
- **Cloud:** You will be able to explain where the private IP inside a VPC comes from, **why an EC2's public
  IP is not visible inside the machine**, and what DHCP option sets are for

---

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 1.1 | MAC address (L2 identity) | `[concept]` | The address of local delivery — the prerequisite for Phase 3 |
| 1.2 | IPv4 anatomy (L3 identity) | `[mechanism]` | **The heart of the phase** — the network/host split gives birth to Phase 2 |
| 1.3 | Public vs private IP | `[concept]` | The whole architecture of cloud networking stands on this split |
| 1.4 | Port and socket | `[mechanism]` | The "which application" question — L4 identity |
| 1.5 | IPv6 (awareness) | `[skip]` | Know why it exists, do not go deep |
| 1.6 | DHCP — getting an IP automatically | `[mechanism]` | How a machine gets its address; the most common first fault |
| 1.7 | When this phase breaks | — | The signatures of addressing faults |

> **How to work through this phase:** Every observation command in this phase is 🟢 (`ip addr`, `ip link`,
> `ss -tulpn`, `ip -4 addr show`) — none of them changes your system, run all of them with an easy mind. The
> one thing to be careful about is the **binary conversion** in 1.2: do not just read it, **do it by hand**.
> Do not move on to Phase 2 (subnetting) without converting three or four addresses to binary on paper —
> there that skill is needed in every line, and if it is missing Phase 2 turns into torture. The most
> illuminating experiment: take your own IP from the `ip addr` output, convert it to binary, then with the
> prefix next to it (something like `/24`) mark which part is "network" and which part is "host". Come to
> Phase 2 with that sheet of paper.

---
---

# 1.1 The MAC Address — L2 Identity

## 1.1.1 A 48-bit hardware address `[concept]`

In Phase 0 we defined L2's job as "deliver to the neighbour on the same local network". The address of that
delivery is the **MAC address** (Media Access Control). Recognise it by three properties:

**It is 48 bits** and is usually written as six hexadecimal (hex) pairs: `3c:52:82:1a:0b:77`. 48 bits means
roughly 281 trillion different addresses — wide enough for every network interface in the world to get a
unique one.

**It is burned into the hardware.** The MAC is written by the manufacturer of the network interface card
(NIC) at production time. That is why it is also called the "hardware address" or "physical address". It can
be changed in software (MAC spoofing), but by default it comes with the card and travels with the machine.

**It has two parts.** The first 24 bits are the **OUI** (Organizationally Unique Identifier) — they identify
the manufacturer; the last 24 bits are the serial number that manufacturer gave to that card. From the first
three octets of a MAC you can find out whose card it is. This detail is at `[skip]` depth: knowing it is
enough, memorising it is unnecessary.

There is also one special address, recognise it right now: **`ff:ff:ff:ff:ff:ff`** = the **broadcast**
address. A frame sent to this destination goes to **everyone** on the local network. In Phase 3 you will see
that ARP uses exactly this.

> **🔧 See it on your machine** 🟢 — read your interfaces and MAC addresses
>
> ```
> $ ip link
> 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
>     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
> 2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 ...
>     link/ether 06:3c:52:82:1a:0b brd ff:ff:ff:ff:ff:ff
> ```
>
> The first address on the `link/ether` line is this interface's **MAC address** — your L2 identity. The
> `brd ff:ff:ff:ff:ff:ff` next to it is this interface's broadcast address (everyone on the local network).
> The MAC of the `lo` (loopback) interface is zero because it never goes out onto a physical network. Ignore
> the `mtu 9001` field for now — in Phase 5.7 you will see how important it is.

## 1.1.2 Why a MAC is only meaningful on the local network `[concept]`

Now let us give the answer we promised in Phase 0: why can we not communicate over the internet using MAC?

The answer is one word: **no hierarchy.** MAC addresses are **flat**. From the address `3c:52:82:...` you
**cannot extract any information** about which country, which city, which network that device is in. The
address only says "who", not "where".

The consequence is catastrophic: if the internet ran on MAC, every router in the world would have to have
**every device's** MAC in its table — billions of rows, and none of them groupable. Like the postal system
trying to work with identity numbers alone, without countries and cities.

The IP address solves this problem with **hierarchy**: the address `10.0.5.20` has a "network" part, and that
part represents a whole group of addresses in a single row. A router can say "everything starting with
10.0.5.0 goes that way" — one row, 254 machines. With MAC such grouping is **impossible**.

That is exactly why there are two addresses (Phase 0, Q4): **MAC is for local delivery, IP is for global path
finding.** And that is why MAC changes at every hop — every hop is a new "local delivery".

> **⚠️ Common misconception: "A MAC address belongs to the machine."**
>
> No, it belongs to the **interface**. If a machine has three network cards (ethernet, Wi-Fi, a virtual
> bridge) there are **three separate MACs**. The proof is that every entry in your `ip link` output has its
> own `link/ether` value. In the cloud this distinction becomes even more visible: if you attach a second ENI
> (Elastic Network Interface) to an EC2 instance, the machine gains a second MAC **and** a second IP. Identity
> is a property of the **interface**, not of the machine — and this is the basis of a machine's ability to
> stand on more than one network at once.

> **🤔 Think 1.1** — Machines A and B are on the same local network with a switch between them. A sends a
> packet to a remote internet server (`93.184.216.34`). (a) What is the **destination IP** A writes? (b) Which
> device does the **destination MAC** A writes belong to — the remote server, or something else? (c) Connect
> your answer to the sentence "MAC changes at every hop" from Phase 0.2.2.
>
> *(Answer: at the end of the phase)*

---
---

# 1.2 IPv4 Anatomy — L3 Identity

## 1.2.1 32 bits, 4 octets, dotted-decimal `[mechanism]`

An IPv4 address is really **a single 32-bit number**. But because it is impossible for humans to read a
32-bit-long binary string, we split it into four and write each piece in decimal:

```
11000000 10101000 00000001 00001010     ← what it really is (32 bits)
   192  .   168  .    1   .   10        ← how we write it
```

Each 8-bit piece is called an **octet**. Because an octet is 8 bits, the range of values it can take is
`00000000` to `11111111` — that is, **0 to 255**. This is why there can be no address like `192.168.1.300`:
300 does not fit in an octet. This way of writing is called **dotted-decimal**.

The one thing you need to understand here is this: **the dots are only for reading.** For network hardware
the address is a single 32-bit number; the dots have no mathematical meaning. This will be critical in Phase
2 — because subnet boundaries **do not have to line up** with the dots.

## 1.2.2 Binary ↔ decimal: converting by hand `[application]`

Do not just read this — the whole of Phase 2 rests on this skill. Fortunately you only need to memorise eight
numbers: the **weight** of each bit in an octet.

| Bit position | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|---|---|---|---|---|---|---|---|
| **Value** | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |

**Binary → decimal:** add up the values of the bits that are 1.

```
11000000  →  128 + 64                      = 192
10101000  →  128 +      32 +        8      = 168
00001010  →                 8 +        2   = 10
11111111  →  128+64+32+16+8+4+2+1          = 255
```

**Decimal → binary:** start from the left; if the number is greater than or equal to that bit's value write 1
and subtract, otherwise write 0.

```
172 →  128? yes (1), remainder 44
       64?  no  (0)
       32?  yes (1), remainder 12
       16?  no  (0)
       8?   yes (1), remainder 4
       4?   yes (1), remainder 0
       2?   no  (0)
       1?   no  (0)
       = 10101100
```

After doing it a few times it becomes a reflex. Three values worth memorising: `255 = 11111111` (all ones),
`0 = 00000000` (all zeros), `128 = 10000000` (only the first bit).

## 1.2.3 The network part vs the host part `[concept]`

Now we have arrived at the most important idea of this phase — and it is the door straight into Phase 2.

An IP address is not a single piece; it is split into **two logical parts**:

- **The network part** (the bits on the left) — "which network". It is **the same** on every machine on the
  same network.
- **The host part** (the bits on the right) — "which machine on that network". It is **different** on every
  machine.

The phone-number analogy works well: `0212` is the area code (network — the same across all of Istanbul), the
rest is the subscriber number (host — different on every line).

Example: the address `192.168.1.10` with a `/24` boundary (we will open this up fully in Phase 2):

```
 192  .  168  .   1  .   10
 └──────── network ────────┘ └host┘
      first 24 bits            last 8 bits
```

So `192.168.1.10` and `192.168.1.77` are **on the same network** (the first 24 bits are identical), but
`192.168.2.10` is **on a different network** (the third octet differs).

Let us say now why this distinction matters so much, because you will carry it all the way to Phase 4: **before
sending a packet, the first decision a machine makes is — "is the destination on my network or not?"** The
answer:

- **If it is on the same network** → deliver directly (with L2, to your neighbour — Phase 3).
- **If it is on a different network** → send it to the default gateway (the router) (Phase 4).

That single decision is the core of how networking works, and the machine makes it by **comparing the network
part**. And the answer to the question "where does the network part start and end" is the **subnet mask** —
the subject of Phase 2.

> **🔧 See it on your machine** 🟢 — read your own IP and prefix
>
> ```
> $ ip -4 addr show
> 1: lo: <LOOPBACK,UP,LOWER_UP> ...
>     inet 127.0.0.1/8 scope host lo
> 2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
>     inet 172.31.20.15/20 brd 172.31.31.255 scope global dynamic ens5
> ```
>
> Take the line `inet 172.31.20.15/20` apart: `172.31.20.15` is the machine's IP, and `/20` is **how many
> bits** the network part has — that is, the first 20 bits are network and the remaining 12 are host.
> `brd 172.31.31.255` is this network's broadcast address (Phase 2.3). `scope global` means "this address
> faces the outside world"; the `scope host` on `lo` means "valid only inside this machine". The word
> `dynamic` matters too: this address was not set by hand but obtained **via DHCP** (1.6). This single line
> contains three of this phase's topics at once.

> **❓ A question that comes to mind: "What do `/20` and `/24` change — the address seems to stay the same?"**
>
> The address does stay the same, but **the definition of "the same network"** changes, and that changes
> everything. If it is `/24` the network part is the first 24 bits: `172.31.20.15` and `172.31.21.15` are **on
> different networks** (the third octet differs). If it is `/20` the network part is the first 20 bits and
> covers only part of the third octet: those two are **on the same network**. Same address, different prefix,
> completely different neighbourhood. And when the answer to "is it on the same network" changes (1.2.3), so
> does whether the machine sends the packet directly or to the gateway. That is exactly why Phase 2 ("the most
> critical phase, the one to go through most slowly") is devoted entirely to this prefix arithmetic.

> **🤔 Think 1.2** — Convert the address `10.0.4.130` to binary. Then: if the network part is the first **24**
> bits, is this machine on the same network as `10.0.4.200`? What about `10.0.5.130`? (a) Justify both with a
> binary comparison. (b) Write down which decision of the machine this comparison determines.
>
> *(Answer: at the end of the phase)*

---
---

# 1.3 Public vs Private IP

## 1.3.1 RFC 1918: addresses that are not routed on the internet `[concept]`

The IPv4 address space is 32 bits — roughly **4.3 billion** addresses. Given the number of devices in the
world that is terrifyingly few. One of the solutions that eased this scarcity is to set aside part of the
address space as **"anyone may use it however they like, but only within their own network"**.

These reserved ranges are called **private** addresses and they are defined by the **RFC 1918** document:

| Range | CIDR notation | How many addresses | Typical use |
|---|---|---|---|
| `10.0.0.0` – `10.255.255.255` | `10.0.0.0/8` | ~16.7 million | Large enterprise networks, cloud VPCs |
| `172.16.0.0` – `172.31.255.255` | `172.16.0.0/12` | ~1 million | Mid-sized networks, the AWS default VPC |
| `192.168.0.0` – `192.168.255.255` | `192.168.0.0/16` | ~65 thousand | Home and small office networks |

Every address outside these (and not reserved for special purposes) is **public**: unique on the internet and
allocated by an authority.

One more familiar range: `127.0.0.0/8` — **loopback**. `127.0.0.1` is every machine's "itself"; traffic to
this address never goes out onto the wire. And `169.254.0.0/16` — **link-local**; it will come back in 1.6 as
the signature of a DHCP failure.

## 1.3.2 Why a private IP cannot be routed on the internet `[mechanism]`

The mechanism here is not technical but **based on agreement** — and that is exactly why it is solid.

The routers on the internet backbone **deliberately do not recognise** the RFC 1918 ranges. If a packet
destined for `192.168.1.10` reaches an ISP router, it is not forwarded — it is **dropped**. The reason is
simple: that address is not unique. Right now there is a device with the address `192.168.1.10` in millions
of home networks around the world. The router **cannot know** which one to send it to, because the address
itself is ambiguous.

So private addresses are not "blocked", they are **meaningless**. This is not a limitation but a design
decision: thanks to it every home, every company, every VPC can reuse the same address blocks without fear of
collision.

Then how does the phone on your home network get to the internet? The answer is **NAT** (Network Address
Translation): on the way out, the private source address is **replaced** with the router's public address.
That is the whole of Phase 7; for now one sentence is enough: **private addresses hide behind a public address
when they go out to the internet.**

> **💡 Cloud connection — the private IP in a VPC and an EC2's public IP:** When you create a VPC (Virtual
> Private Cloud) you give it a private CIDR block (e.g. `10.0.0.0/16`). Every EC2 inside it takes a **private
> IP** from that block, and that is the instance's real, persistent network identity — it is the address you
> see in the `ip addr` output. If a **public IP** has been assigned to the instance, that address is **not
> visible inside** the instance: AWS maps it 1:1 to the private IP outside, at a NAT layer. That is why you
> see `172.31.x.x` with `ip addr` while you connect from the internet with `3.120.x.x` — the two are the same
> machine. The answer to "where is my public IP" is not inside the machine but in the **cloud console** or the
> instance metadata service. This single fact is one of the most confusing things in the cloud, and it will
> settle fully once you learn NAT in Phase 7.

> **🤔 Think 1.3** — An engineer set up a server with the address `192.168.1.50` on the office network and
> told a friend "connect to this IP from home". The friend cannot connect. (a) Why? (b) What is the chance
> that one of the devices behind the friend's home router is also `192.168.1.50`, and why does that make the
> problem even clearer? (c) Which concept does the correct solution involve?
>
> *(Answer: at the end of the phase)*

---
---

# 1.4 Port and Socket

## 1.4.1 The port: separating different services on the same machine `[mechanism]`

The IP address brings the packet as far as the **machine**. But if a web server, an SSH server and a database
are running on that machine at the same time — **which one** will the packet be delivered to?

The answer is the **port**. A port is a 16-bit number (that is, **0–65535**) and it is carried in the L4
header (TCP/UDP). Remember the funnel metaphor from Phase 0: IP is "which machine", port is "which
application on that machine".

The building analogy: IP = the street address of the building, port = the flat number. The postman finds the
building by the address and the flat by the number.

Ports fall into three groups:

| Range | Name | Who uses it |
|---|---|---|
| 0 – 1023 | **Well-known** | Standard services; on Linux root privileges are needed to open one |
| 1024 – 49151 | **Registered** | Allocated to applications (e.g. 3306 MySQL, 5432 PostgreSQL) |
| 49152 – 65535 | **Ephemeral** | The source port the client picks at random for every connection |

Well-known ports worth memorising — these keep coming at you in the field:

| Port | Service | Where you see it |
|---|---|---|
| 22 | SSH | Connecting to a server |
| 53 | DNS | Name resolution (Phase 6) |
| 80 | HTTP | Web, unencrypted (Phase 8) |
| 443 | HTTPS | Web, with TLS (Phase 8) |
| 3306 | MySQL | Database |
| 5432 | PostgreSQL | Database |

One last detail, a very important one: a connection has **two** ports. The destination port is fixed (like
443), but the client picks the **source port** at random from the ephemeral range for each connection. That
is why you can open ten tabs to the same site at once: ten separate source ports, ten separate connections.

## 1.4.2 Socket = (IP : Port) `[concept]`

When you combine an IP address with a port, the pair you get is called a **socket**: `172.31.20.15:443`. A
socket is a **communication endpoint** on the network.

But the really important thing is this: a **connection** becomes unique not with a single socket but with
**four values**:

```
(source IP, source port, destination IP, destination port)
```

This is called the connection's **4-tuple**. (In Phase 9 the protocol will be added to it and it will become
the **5-tuple** — that is the basis of firewall rules.) This is exactly why a thousand different users can
connect to the **same** port 443 of the same web server: the destination (IP:port) is the same for all of
them, but the source (IP:port) is different for each, so every connection is unique.

> **🔧 See it on your machine** 🟢 — read the listening ports and sockets
>
> ```
> $ sudo ss -tulpn
> Netid State  Local Address:Port   Peer Address:Port  Process
> tcp   LISTEN 0.0.0.0:22           0.0.0.0:*          users:(("sshd",pid=712,...))
> tcp   LISTEN 127.0.0.1:5432       0.0.0.0:*          users:(("postgres",pid=980,...))
> udp   UNCONN 127.0.0.53:53        0.0.0.0:*          users:(("systemd-resolve",...))
> ```
>
> Every line is a **socket**. In the `Local Address:Port` column you see the socket's two parts.
> `0.0.0.0:22` = "listen on 22 on all interfaces"; `127.0.0.1:5432` = "reachable only from inside the
> machine". This distinction is vital but not the subject of this phase — it will come back in Phase 9 and
> Phase 10. The `LISTEN` state means "this socket is waiting for a connection". The last column shows the
> process behind the socket: behind every open port there is **a running program**.

> **⚠️ Common misconception: "Opening a port" means writing a rule in the firewall.**
>
> That phrase confuses two **completely separate** things and constantly creates confusion in the field.
> (1) A port **being listened on**: a process has opened that port and is waiting for connections (visible in
> `ss -tulpn`). (2) A port **being allowed through the firewall**: a rule permits traffic to that port. These
> are independent: you can "open" 8080 in the firewall, but if no process is listening the connection is still
> rejected (**connection refused**). Or a process may be listening but the firewall is cutting it off
> (**timeout**). It is no coincidence that the two symptoms differ: *refused* means "I reached the machine,
> nobody answered", *timeout* means "I never reached the machine at all". Settle this distinction now — in
> Phase 9 and Phase 10 it is half of the diagnosis.

> **❓ A question that comes to mind: "What exactly does the 'Address already in use' error mean?"**
>
> When a process starts listening on a port, it takes that (IP, port) pair **exclusively** — a second process
> cannot listen on the same address and port pair at the same time. If you get this error while starting your
> application, the meaning is clear: **somebody already holds that port.** It usually happens for two reasons:
> (1) an old copy of your application is still running (find its PID with `sudo ss -tulpn | grep :8080`, the
> Phase 0 rule "behind every socket there is a process"), or (2) a previous connection is in the `TIME_WAIT`
> state (Phase 5.6 — at `[skip]` depth, but hear the name). Note: the same port on **different IPs** is not a
> problem — `127.0.0.1:8080` and `10.0.1.5:8080` are two separate sockets, they do not collide.

> **🤔 Think 1.4** — A web server is listening on `0.0.0.0:443` and 5,000 users are connected at the same
> time. (a) How many different **destination** sockets are there on the server side? (b) How are the 5,000
> connections told apart — which values differ? (c) How does this explain why the "port exhaustion" problem
> arises on the **client** side?
>
> *(Answer: at the end of the phase)*

---
---

# 1.5 IPv6 — Awareness

## 1.5.1 Why it exists: IPv4 address exhaustion `[concept]`

IPv4's 32 bits give ~4.3 billion addresses. In the 1980s that looked infinite; in the 2010s it officially ran
out — the regional authorities that allocate addresses had no new blocks left. NAT (Phase 7) postponed this
collapse for years but did not solve it at the root.

**IPv6** solves it at the root: the address length is **128 bits**. That means roughly 3.4×10³⁸ addresses —
enough for every grain of sand in the world to get billions. Addresses are written in hex and separated by
colons:

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
2001:db8:85a3::8a2e:370:7334        ← the shortened form (zeros are skipped)
```

This is all you need to know for this phase — the depth tag is `[skip]`. Three practical notes are enough:
(1) because addresses are abundant in IPv6, **NAT is mostly unnecessary**, every device can have a public
address (Phase 7.5). (2) The transition-period model is **dual-stack**: a machine speaks both IPv4 and IPv6 at
the same time. (3) In dual-stack the most common fault is the application picking **the wrong family** — "it
works on one protocol, it hangs on the other".

---
---

# 1.6 DHCP — Getting an IP Automatically

## 1.6.1 Static vs dynamic: where the address comes from `[concept]`

So far we have talked about **what** addresses are. But where does a machine **get** its address from?

There are two ways. **Static**: you write it by hand (into the netplan file on Ubuntu). It is used for
servers — when the address must not change. **Dynamic**: you get it automatically from a server on the
network. The protocol that does this is called **DHCP** (Dynamic Host Configuration Protocol).

DHCP makes it unnecessary to configure every device plugged into the network by hand: that is why, when you
walk into a café and connect to the Wi-Fi, your phone can be on the internet within seconds.

And there is a critical point here that most people do not know: **DHCP does not hand out only an IP.** It
gives **all four** pieces of information a machine needs to work on the network:

1. **IP address** — the machine's identity (1.2)
2. **Subnet mask** — where the "same network" boundary is (Phase 2)
3. **Default gateway** — the address traffic not on the local network goes to (Phase 4)
4. **DNS server** — the address that will turn names into IPs (Phase 6)

These four correspond to four separate phases of this book. DHCP delivers the inputs of those four phases in
one move — which is why, when DHCP breaks, the symptom looks like "there is no network at all".

## 1.6.2 The DORA cycle: four steps to an address `[mechanism]`

DHCP's address allocation consists of four messages and is known by their initials as **DORA**:

1. **Discover** — The client does not have an IP yet. It sends a **broadcast** to the local network: "is
   there a DHCP server here?" Source IP `0.0.0.0` (it has no address yet), destination `255.255.255.255`
   (everyone). This step **must** be a broadcast — the client does not know the server's address.
2. **Offer** — The DHCP server answers: "I can offer you the address `10.0.1.47`; here is the mask, the
   gateway, the DNS, and the duration (lease) is 24 hours."
3. **Request** — The client accepts the offer and announces it, again by broadcast: "I want `10.0.1.47`."
   The reason it is a broadcast is so that, if there is more than one DHCP server on the network, the others
   can withdraw their offers too.
4. **Ack** — The server confirms: "fine, it is yours. The lease has started." The client now configures its
   address and joins the network.

The address is **lent**, not given forever: this is called a **lease**. Before the time runs out the client
sends a Request again to renew it. That is why, if a machine stays off for a long time, it may get a different
IP when it comes back.

![Figure 1.1 — The DHCP DORA cycle: the four messages between the client and the DHCP server (Discover → Offer → Request → Ack) and whether each step is broadcast or unicast. With the Ack the client has received not only an IP but also the subnet mask, the default gateway and the DNS server.](../diagrams/png/nw-1-01-dhcp-dora.png)

In the figure the vertical line on the left is the client and the one on the right is the DHCP server; time
flows from top to bottom. Note that the source address of the first message is `0.0.0.0` — the client has no
identity yet. The box at the bottom right lists the four pieces of information delivered with the Ack: each
one of them is the subject of a different phase of this book.

> **💡 Cloud connection — DHCP option sets in a VPC:** When you launch an EC2 instance you do not type its IP
> by hand; the DHCP service inside the VPC gives it its private IP, its mask, its gateway and its DNS server.
> The word `dynamic` in the `ip addr` output is exactly the trace of this. You can configure the content of
> this distribution with a **DHCP option set** — its most common use is to hand out your own DNS servers
> (e.g. an internal Active Directory DNS) instead of the VPC's default resolver. The practical warning here:
> misconfiguring a DHCP option set **breaks DNS resolution for all your instances at once** — and the symptom
> looks like "there is no internet", while the IP and the routing are perfectly fine (we will settle this
> distinction fully in Phase 6).

## 1.6.3 When DHCP breaks: 169.254.x.x and "no IP at all" `[mechanism]`

If DHCP fails — there is no server, it is unreachable, or there are no addresses left in its pool — the client
**has no address**. In that case you see two different signatures depending on the operating system:

- **Windows:** It gives itself an address from the `169.254.x.x` range. This is called **APIPA** (Automatic
  Private IP Addressing) or **link-local**. With this address you can only reach other link-local devices on
  the same segment; there is no gateway, no DNS, no internet.
- **Ubuntu (NetworkManager/systemd-networkd):** It usually gets no IPv4 address at all. In the `ip addr`
  output there is **no `inet` line** on that interface.

Both say the same thing: **"DHCP did not answer."** And from a diagnostic point of view this is a very
valuable signature, because an engineer who sees `169.254.x.x` narrows the search instantly — the problem is
not in the application, in DNS or in routing, it is in **address allocation**.

> **🔧 See it on your machine** 🟢 — find out whether your address came from DHCP
>
> ```
> $ ip -4 addr show ens5
> 2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
>     inet 172.31.20.15/20 brd 172.31.31.255 scope global dynamic ens5
>        valid_lft 2591998sec preferred_lft 2591998sec
> ```
>
> Two clues: the word `dynamic` tells you the address was obtained via DHCP (if it were static that word would
> not be there). And `valid_lft` (valid lifetime) is the **lease time** — in this example ~30 days are left.
> The time counts down second by second and is renewed. If you see no `inet` on this line at all, or the
> address is `169.254.x.x`, your diagnosis is ready: DHCP did not answer.

> **🤔 Think 1.5** — A user says "the internet is not working". In the `ip addr` output the interface's IP is
> `169.254.8.31`. (a) What does this address tell you? (b) What would you expect if you tried `ping 8.8.8.8`
> from this machine, and why? (c) If another machine on the same network works fine, how does the direction of
> your suspicion change?
>
> *(Answer: at the end of the phase)*

---
---

# 1.7 When This Phase Breaks — The Signatures of Addressing Faults

Addressing faults have one thing in common: the symptom almost always looks like "there is no internet", but
**the root cause lies in one of three different steps** — L2 identity (MAC), L3 identity (IP), or L4 identity
(port). The table below takes the symptom to the right step.

| Symptom | Probable cause | Where to look | Related section |
|---|---|---|---|
| No `inet` line on the interface | DHCP did not answer / interface down | `ip addr`, `ip link` (is it UP) | 1.6.3 |
| IP is `169.254.x.x` (APIPA) | The DHCP server cannot be reached | The DHCP server, the L2 link | 1.6.3 |
| There is an IP but no internet | DHCP gave the wrong mask/gateway | `ip route` (is there a `default via`) | 1.6.1 (Phase 4) |
| IP works, internet works, names do not resolve | DHCP gave the wrong DNS server | `/etc/resolv.conf` | 1.6.1 (Phase 6) |
| Two devices on the same network keep dropping | IP conflict (the same address on two machines) | `ip neigh`, DHCP lease records | 1.2.3 |
| A remote private IP cannot be reached | Private addresses are not routed on the internet | Is the address in an RFC 1918 range | 1.3.2 |
| "Address already in use" | Another process holds the port | `ss -tulpn \| grep :<port>` | 1.4.2 |
| "Connection refused" | No process is listening on that port | `ss -tulpn` (is there a LISTEN) | 1.4.1 |
| In the cloud `ip addr` does not show the public IP | The public IP is mapped outside with NAT | Cloud console / metadata | 1.3.1 |
| The machine gets a different IP on every reboot | The DHCP lease was not renewed / no reservation | DHCP lease time, static/reserved IP | 1.6.2 |

> **The lesson from this table:** "I have no address" and "I have an address but it is wrong" are **two
> completely different** faults, and telling them apart is the first thing you must do. If there is **no
> `inet` line at all** in the `ip addr` output, the fault is in address allocation (DHCP/L2) — there is no
> point looking higher up. If `inet` **is** there, address allocation worked and suspicion automatically moves
> one step up: is the mask wrong (Phase 2), is the gateway missing (Phase 4), is DNS broken (Phase 6)? Always
> take the three-step funnel (MAC → IP → port) in order: **first is there an identity, then is it correct,
> and last does it reach the intended application.** And do not forget: `169.254.x.x` is not an address, it is
> a **fault report**.

---
---

# Phase 1 — Answers to the Think questions

## Answer 1.1 — The destination IP is the remote server, the destination MAC is its own router

(a) The destination IP is **`93.184.216.34`** — the remote server itself. The L3 address is constant end to
end and shows the final destination (Phase 0.2.2).

(b) The destination MAC is **not** the remote server's MAC — it cannot be, because A can never learn that MAC
(the remote server is on another local network and MACs are not routed, 1.1.2). The destination MAC A writes
is **the MAC of the default gateway (the router) on its own local network**. So the frame physically goes to
the router; the router opens the envelope, sees the real destination in the IP header, and sends the packet on
to the next hop in a new L2 envelope.

(c) This is exactly the concrete form of the sentence "MAC changes at every hop": one MAC pair is used in the
A→router step, a completely different MAC pair at the next hop. The IP, meanwhile, never changes throughout
the journey. The next question naturally becomes: *how did A know the router's MAC?* The answer is **ARP** and
it is the subject of Phase 3.1.
**Related section:** 1.1.2 (Phase 0.2.2) · **Next:** Phase 3.1 (ARP), Phase 4.1 (gateway).

## Answer 1.2 — The binary comparison makes the "same network?" decision

`10.0.4.130` in binary: `00001010.00000000.00000100.10000010`
(10 = `00001010`, 0 = `00000000`, 4 = `00000100`, 130 = `10000010`).

(a) **With `10.0.4.200`:** the first 24 bits are `00001010.00000000.00000100` — **the same** in both (the
first three octets are `10.0.4`). So they are **on the same network**. Only the host part (the last octet: 130
vs 200) differs.
**With `10.0.5.130`:** the last octet of the first 24 bits differs — `00000100` (4) vs `00000101` (5). Because
the third octet is inside the network part, these two are **on different networks**.

(b) This comparison determines the machine's most fundamental decision: **"is the destination on my network?"**
If it is on the same network the packet is delivered directly to the neighbour (L2, Phase 3); if it is on a
different network it is sent to the **default gateway** (Phase 4). So the machine `10.0.4.130` tries to reach
`10.0.4.200` directly, and `10.0.5.130` through the router. If the prefix is misconfigured this decision is
made wrongly too — that is the most expensive class of mistake in Phase 2.
**Related section:** 1.2.2-1.2.3 · **Next:** Phase 2.1 (subnet mask), Phase 4.1 (the gateway decision).

## Answer 1.3 — A private address is meaningless on the internet

(a) `192.168.1.50` is an **RFC 1918 private** address (1.3.1). Internet routers deliberately do not forward
this range — they drop the packet. The friend's packet **never reaches** the office network; it dies on the
internet backbone.

(b) The chance is very high, because `192.168.1.0/24` is the most common default block of home routers. And
this makes the problem even clearer: if the friend tries to connect to that address, their machine decides
"this address is on **my** local network" (1.2.3) and never sends the packet out at all — it goes to the
`192.168.1.50` on their own network (a completely different device if there is one, nowhere if there is not).
So the address is not even "unreachable", it may **reach the wrong place**. That is exactly the practical
consequence of private addresses not being unique.

(c) The correct solution involves **NAT + port forwarding** (Phase 7.2): you connect to the office router's
**public** IP, and the router forwards the incoming traffic to `192.168.1.50` inside. Alternatively a **VPN**
(Phase 7.4) is set up and the friend is logically taken inside the office network.
**Related section:** 1.3.1-1.3.2 · **Next:** Phase 7.1-7.2 (NAT and port forwarding).

## Answer 1.4 — One destination socket, 5,000 different 4-tuples

(a) There is **a single** destination socket on the server side: `<server IP>:443`. 5,000 separate listening
sockets are not opened for 5,000 connections — one listens and accepts the incoming connections.

(b) Connections are told apart by the **4-tuple**: (source IP, source port, destination IP, destination port).
The destination IP and destination port are **the same** in all 5,000 connections (server:443). What differs
is the **source** side: either different source IPs (different users), or different **ephemeral source ports**
of the same user. Together these four make every connection unique (1.4.2).

(c) Because the bottleneck is on the source side: the ephemeral port range is limited (~49152–65535, that is
~16,000 ports). **A single client IP** can open at most that many simultaneous connections towards a single
destination socket — after that the 4-tuples run out. There is no such limit on the server side, because the
server uses one port and the variety comes from the source side. That is why port exhaustion is typically seen
at an **exit point behind NAT** (Phase 7.2) or on a busy proxy/client machine.
**Related section:** 1.4.1-1.4.2 · **Next:** Phase 7.2 (PAT), Phase 9.2 (the 5-tuple).

## Answer 1.5 — 169.254 is not an address, it is a fault report

(a) `169.254.8.31` is a **link-local / APIPA** address (1.3.1, 1.6.3). Its meaning is clear: the machine
**could not get** an address from DHCP and made up a temporary one for itself. So the fault is at the
address-allocation step — not in DNS, routing or the application layer.

(b) `ping 8.8.8.8` **fails** ("Network is unreachable" or similar). The reason: **no default gateway comes**
with a link-local address (1.6.1 — DHCP hands out the gateway too). The machine does not know where to send a
packet for a destination that is not on its local network; there is **no `default via ...` line** in the
`ip route` output (Phase 4.1). The packet never leaves the machine.

(c) Suspicion moves to **the machine itself**. If the other machines on the same network get addresses from
DHCP without trouble, the DHCP server is up and reachable. Then the problem is specific to this machine: a
cable/port fault (L2 — is `LOWER_UP` there in `ip link`?), a faulty NIC, a switch port that has fallen into
the wrong VLAN, or the client's DHCP service not running. The order of diagnosis is again bottom up (Phase
0.1.3): first **is there a link**, then can an address be obtained.
**Related section:** 1.6.3 · **Next:** Phase 4.1 (default gateway), Phase 10.1 (layer-by-layer diagnosis).

---
---

# Phase 1 — Frequently asked questions

**Q1 — Can a MAC address really not be changed?** It is burned into the hardware but it can be **overridden**
in software (MAC spoofing; on Linux `ip link set dev ens5 address ...`). In practice this is used to get past
some network access controls or to put a spare device into service with the same identity. In the cloud it is
usually **forbidden**: the hypervisor does not allow a source MAC other than the one assigned to the ENI
(1.1.1).

**Q2 — Why is `192.168.1.300` an invalid address?** Because every octet is **8 bits** and the largest value 8
bits can hold is 255. 300 does not fit in an octet. This is the fastest way to spot a "made up" address — if
you see a number greater than 255 in an octet the address is invalid (1.2.1).

**Q3 — Does using a private IP provide security?** Partly, and in a **misleading** way. Private addresses
cannot be routed directly from the internet, which is a natural obstacle. But this is **not a firewall**: that
service can be opened to the outside with NAT/port forwarding, the network can be entered with a VPN, and for
an attacker who has once got inside the network a private address provides no protection at all. Security is
the job of Phase 9; a private address is not a security property but an **addressing** property (1.3.2).

**Q4 — Why is the source port random?** Because the uniqueness of a connection rests on the 4-tuple (1.4.2).
If the same client is going to open more than one connection to the same port of the same server, the only
distinguishing value is the **source port**. Random (ephemeral) selection provides that variety. Also, because
predictable source ports make some attacks easier, randomness contributes to security too.

**Q5 — What is the difference between "connection refused" and "timeout"?** A critical distinction.
**Refused**: the packet **reached** the machine, but no process is listening on that port — the machine
actively answered "nobody here" (a TCP RST). **Timeout**: the packet never reached the destination or no
answer came back — usually a firewall dropped it silently, or the routing/address is wrong. Roughly: *refused*
= an L4 problem (no service), *timeout* = an L3/firewall problem (no path) (1.4.1).

**Q6 — Does the network work without DHCP?** Yes — if you write all the settings by hand (statically). But you
have to get four things right at once: IP, mask, gateway, DNS (1.6.1). Static configuration is common on
servers and network devices (the address must not change); on clients DHCP is used almost always. In the cloud,
writing an IP into an instance by hand is usually **wrong** — address allocation is managed by the cloud.

**Q7 — Do I need to learn IPv6 now?** No, at this phase it is at `[skip]` depth. What you need to know: why it
exists (IPv4 exhaustion), that it is 128 bits, and that in dual-stack there is a fault of the form "it works
on one protocol, it hangs on the other". You will see the cloud-side equivalents (dual-stack VPC, egress-only
IGW) in Phase 7.5 and Phase 11 (1.5.1).

---
---

# Phase 1 — Test yourself

Write your answers on paper, then compare them with the answer key. Target: 14+ out of 18.

## Part A — Definition and mechanism

1. How many bits is a MAC address, how is it written, and which layer's identity is it?
2. Why can MAC not be routed at internet scale? Write the one-word reason and a one-sentence explanation.
3. How many bits is an IPv4 address, how many octets does it consist of, and what is the range of values an
   octet can take?
4. Write the address `11000000.10101000.00001010.00000001` in dotted-decimal.
5. What do the "network part" and the "host part" of an IP address mean? Which one is the same on all
   machines on the same network?
6. Write RFC 1918's three private ranges in CIDR notation.
7. How many bits is a port, what is its range, and write the name + purpose of the three port groups.
8. What is a socket? Which four values make a **connection** unique?

## Part B — Apply and diagnose

9. Convert the number `172` to binary by hand, showing your steps.
10. In the `ip addr` output you see the line `inet 10.0.1.25/24 ... dynamic`. Write **three** separate pieces
    of information you can extract from this line.
11. A machine's IP is `169.254.3.44`. What is your diagnosis, and what are the first two things you would look
    at to confirm it?
12. You get an "address already in use" error while starting your application. With which command do you find
    the root cause, and what do you look at in the output?
13. `ping <ip>` works but `curl http://<ip>` says "connection refused". Which address step is healthy and
    which is the problem?
14. List the four pieces of information DHCP hands out and write which phase of this book each one
    corresponds to.

## Part C — Reasoning and connection

15. What does a machine write into the destination IP and destination MAC fields when it sends a packet to a
    remote server? Connect the difference between the two to Phase 0's "it changes at every hop" rule.
16. Why must the first message of the DORA cycle be a **broadcast**?
17. A user can get an IP, `ping 8.8.8.8` works, but `ping google.com` does not. Which piece of information
    handed out by DHCP might be wrong, and which pieces are working correctly?
18. Explain the difference between "connection refused" and "timeout", stating also which layers of the
    network they point to.

---

## Answer key

1. **48 bits**, as six hex pairs (`3c:52:82:1a:0b:77`), and it is the **L2** (data link) identity (1.1.1). —
   2. The reason: **no hierarchy**. MAC is flat; from the address you cannot tell where the device is, so it
   cannot be grouped and does not fit in a router table (1.1.2). — 3. **32 bits**, **4 octets**, each octet
   **0–255** (1.2.1). — 4. `192.168.10.1` (1.2.2). — 5. The network part is "which network" and it is the same
   on **all** machines on the same network; the host part is "which machine on that network" and it is
   different on every machine (1.2.3). — 6. `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` (1.3.1). —
   7. **16 bits**, 0–65535. Well-known (0–1023, standard services, root required), registered (1024–49151,
   allocated applications), ephemeral (49152–65535, the client's source port) (1.4.1). — 8. A socket = the
   **(IP : port)** pair. What makes a connection unique: **source IP, source port, destination IP, destination
   port** (the 4-tuple) (1.4.2).

9. 128? yes (1), remainder 44 → 64? no (0) → 32? yes (1), remainder 12 → 16? no (0) → 8? yes (1), remainder 4
   → 4? yes (1), remainder 0 → 2? no (0) → 1? no (0). Result: **`10101100`** (1.2.2). — 10. (i) The machine's
   IP is `10.0.1.25`; (ii) the network part is the first **24 bits** (`/24`), so `10.0.1.x` is the same
   network; (iii) the address was obtained via **DHCP** (`dynamic`) (1.2.3, 1.6.3). — 11. Diagnosis: **DHCP
   did not answer** (an APIPA/link-local address). What to look at: (a) with `ip link`, is the interface really
   UP/LOWER_UP (is there an L2 link), (b) can another machine on the same network get an address from DHCP (is
   the server broken, or the machine) (1.6.3). — 12. `sudo ss -tulpn | grep :<port>` — in the output you look
   at the name and the **PID** of the process holding that port in the `LISTEN` state (1.4.2). — 13. **L3 is
   healthy** (the IP is right, the machine is reachable — ping answers); the problem is **at L4**: no process
   is listening on that port, the machine refuses with an RST (1.4.1). — 14. IP address (Phase 1), subnet mask
   (Phase 2), default gateway (Phase 4), DNS server (Phase 6) (1.6.1).

15. Into the destination IP field goes **the final destination's IP**, and it does not change throughout the
    journey. Into the destination MAC field goes the MAC of **the next hop** (for a remote destination: of the
    default gateway), and it is **renewed at every hop**. IP = end-to-end identity, MAC = local delivery label
    (1.1.2, Phase 0.2.2). — 16. Because at that moment the client has neither its own IP nor the DHCP server's
    address; not knowing whom to ask, it has to ask **everyone** (source `0.0.0.0`, destination
    `255.255.255.255`) (1.6.2). — 17. What is wrong is the **DNS server** information. What is working: the IP
    address (the machine has an identity), the subnet mask and the default gateway (it can get out to the
    internet by IP). So address allocation and routing are healthy, **name resolution** is broken (1.6.1,
    Phase 6). — 18. **Refused:** the packet reached the machine but no process is listening on that port → the
    machine returned an RST; this is an **L4** problem (no/dead service). **Timeout:** no answer came back at
    all → the packet was dropped on the way or never reached the destination; this is an **L3/firewall**
    problem (no path, or silently blocked) (1.4.1, Q5).

## Scoring

| Correct | What it means |
|---|---|
| 16-18 | You can tell the three address steps (MAC/IP/port) apart. You are ready for Phase 2. |
| 13-15 | Good. Re-read the sections of the questions you missed — especially 1.2.3 (the network/host split). |
| 9-12 | The basics are there but fragile. **Practise the binary conversion in 1.2.2 by hand** — Phase 2 does not move without it. |
| 0-8 | Walk the phase again. The goal: being able to explain every field of the `ip addr` output without memorising. |

Question you missed → section to go back to:

| Question | Section |
|---|---|
| 1, 2, 15 | 1.1 The MAC address |
| 3, 4, 9 | 1.2.1-1.2.2 IPv4 and binary |
| 5, 10 | 1.2.3 The network/host split |
| 6, 13 | 1.3 Public vs private |
| 7, 8, 12 | 1.4 Port and socket |
| 11, 14, 16, 17 | 1.6 DHCP |
| 18 | 1.4.1 + the 1.7 fault table |

---
---

# Phase 1 — Closing and the bridge to Phase 2

## What you carry from this phase

Phase 1 gave you the **three-step address funnel**: **MAC** "which device on this local network" (L2, flat,
changes at every hop), **IP** "which machine on the internet" (L3, hierarchical, constant end to end),
**port** "which application on that machine" (L4). You learned the 32-bit structure of an IPv4 address and
the network/host split; you saw why private and public addresses are different worlds; you grasped how the
socket and the 4-tuple make connections unique. And you learned where a machine gets its address from
(DHCP/DORA), that this mechanism hands out not only an IP but also **mask + gateway + DNS**, and that when it
breaks it leaves the `169.254.x.x` signature.

The most lasting sentence is this: **"I have no address" and "I have an address but it is wrong" are
different faults** — and the presence or absence of a single `inet` line in the `ip addr` output makes that
distinction instantly.

## Where Phase 2 connects to this

In Phase 1 we said an address has a "network part" and a "host part" (1.2.3) — but we kept postponing
**exactly where** that boundary is drawn. We wrote `/24` and `/20` and said "we will open it up in Phase 2".

Phase 2 is exactly that: **the subnet mask and CIDR.** You will learn how that boundary is drawn, how it is
shifted, and how to calculate how many machines count as being "on the same network" when you shift it. This
is the book's **most critical phase, the one to go through most slowly**, because the whole of cloud network
design — the VPC CIDR, splitting subnets, route tables, security group scopes, peering collisions — rests on
a single skill: **being able to read a prefix and say what it means.**

Phase 1 introduced the addresses; Phase 2 teaches you to **divide** them.

> **🤔 Phase output — ask yourself:** In 1.2.3 we said "the machine decides whether the destination is on its
> own network by comparing the network part". Before moving to Phase 2, think: what happens if two machines
> are given the **same** IP range but **different** prefixes (one `/24`, the other `/16`)? Can A consider B
> "on the same network" while B considers A "far away" — and how would such an asymmetry break the traffic?
> (Hint: the outbound path and the return path can make different decisions.)
>
> **🧪 Lab 1 idea (on your own machine, all 🟢):** (1) List your MAC addresses with `ip link` and your IPs and
> prefixes with `ip -4 addr` — see your L2 and L3 identities side by side. (2) Convert your own IP to binary
> on paper; draw a vertical line where the prefix says and mark the network and host parts. Keep this sheet,
> we will use it in Phase 2. (3) Find the word `dynamic` and the `valid_lft` lease time in the `ip -4 addr`
> output — confirm with your own eyes that your address came from DHCP. (4) List the listening sockets with
> `sudo ss -tulpn`; match the (IP:port) pair on each line with the process behind it. (5) Run
> `curl ifconfig.me` and compare the public IP that comes back with the private IP in `ip addr` — the two
> being different is the seed of Phase 7 (NAT). These five steps make the three address steps concrete in your
> hands.

---

> **Navigation:** [◀ Phase 0 — Mental Model](Phase_0_Mental_Model.md) · **Phase 1** · [Phase 2 — Subnetting and CIDR ▶](Phase_2_Subnetting_and_CIDR.md)
