# Phase 7 — NAT and the Real World

> **Navigation:** [◀ Phase 6 — From Name to Address: DNS](Phase_6_DNS.md) · **Phase 7** · [Phase 8 — The Application Layer: HTTP + TLS ▶](Phase_8_HTTP_and_TLS.md)

---

## Where we come from

At the end of Phase 6 we asked: three devices at your home connected to the same site at the same time, all
three had their source IP replaced by the **same public IP**, and the three answers coming back all have the
same destination IP. **How** does the router deliver those answers to the right devices?

We gave the hint too: in Phase 1.4.2 we talked about the **four things** that make a connection unique —
source IP, source port, destination IP, destination port. The router still has one field it can change:
**the source port.** The whole of this phase is built on that idea.

What you brought with you:

- **Private addresses cannot be routed on the internet** (1.3.2) — the reason this phase exists.
- **A 4-tuple makes a connection unique** (1.4.2) — the operating principle of NAT.
- **Ephemeral ports are in the range 32768–60999** (1.4.1) — the pool the router uses.
- **The MTU black hole** (5.7.3) — in the tunnelling section (7.4) you will meet your old friend again.

## The question of this phase

> *"There are billions of devices but only ~4 billion addresses in IPv4 — and most of them are already
> allocated. How do this many devices get to the internet with this few addresses?"*

The answer: **they do not.** At least not with their own addresses. On the way out their addresses are
**changed**, and on the way back they are translated back. This translation is called **NAT** (Network
Address Translation) and it runs underneath every home network, every office network and every cloud VPC you
see today.

This phase is also a **concept-cleaning** phase: here you will learn exactly what expressions you will use
every day in the cloud — "public IP", "private subnet", "open/closed to the internet" — actually mean. Most
of the architectural mistakes made in the cloud come from confusing these concepts.

---

## By the end of this phase

- You will be able to explain why NAT exists and how private→public translation works
- You will be able to explain how returning traffic is matched to the right device — the logic of the
  translation table
- You will know the difference between PAT (port-based multiplexing) and basic NAT
- You will be able to explain what port forwarding is and **why** a service behind NAT cannot be reached
  from outside
- **Cloud:** You will be able to explain clearly the difference between an Internet Gateway and a NAT
  Gateway — the distinction between "public egress" and "outbound only"
- You will know what tunnelling is, what IPsec is for, and why the MTU problem in a tunnel is unavoidable
- You will be able to explain why NAT is mostly unnecessary in IPv6 and what dual-stack brings

---

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 7.1 | The logic of NAT | `[mechanism]` | The translation idea and the return problem |
| 7.2 | PAT and port forwarding | `[mechanism]` | **The heart of the phase** — port-based multiplexing |
| 7.3 | Cloud NAT | `[concept]` | IGW vs NAT GW — the basis of cloud architecture |
| 7.4 | VPN and tunnelling | `[concept]` | Putting a packet inside a packet |
| 7.5 | IPv6 and a world without NAT | `[concept]` | The future and the dual-stack traps |
| 7.6 | When this phase breaks | — | NAT failure signatures |

> **How to work through this phase:** The experiments in this phase are 🟢 and the most striking one is a
> single line: `curl ifconfig.me` — it tells you how your machine **looks from outside**. When you compare
> it with your own address in `ip addr` output you have seen NAT with your own eyes. In this phase
> conceptual clarity matters more than memorising commands: in particular do not skim over the IGW/NAT GW
> distinction in 7.3 — the most common architectural mistake in the cloud comes from there. And in 7.4 keep
> Phase 5.7 (MTU) next to you; the two are continuations of each other.

---
---

# 7.1 The Logic of NAT

## 7.1.1 The translation idea `[mechanism]`

Your computer at home has the address `192.168.1.50`. This is a **private** address (1.3.1) and it cannot be
routed on the internet — the first router on the way drops it.

So the router does this: before sending the packet out it **replaces the source IP field with its own public
address.**

```
On the way out:
  192.168.1.50:51234  →  93.184.216.34:443      (what the device sees)
  203.0.113.7:51234   →  93.184.216.34:443      (what the internet sees)
  └── the router replaced the source IP with its own public address
```

The server sends its answer to `203.0.113.7` — because that is the source address it knows. The answer
arrives at the router.

And this is where the real problem begins.

## 7.1.2 The return problem `[mechanism]`

The router holds a packet destined for `203.0.113.7:51234`. **Which device** should it give it to?

The answer: while making the translation, the router keeps a **record**. This is called the NAT table (or
the translation table / connection tracking table):

| Internal address:port | External address:port | Destination | State |
|---|---|---|---|
| 192.168.1.50:51234 | 203.0.113.7:51234 | 93.184.216.34:443 | ESTABLISHED |
| 192.168.1.51:49832 | 203.0.113.7:49832 | 93.184.216.34:443 | ESTABLISHED |
| 192.168.1.52:52001 | 203.0.113.7:52001 | 142.250.185.78:443 | ESTABLISHED |

When the answer arrives the router looks at the table, finds the matching row, **translates** the
destination IP back to the internal address and delivers the packet to the right device.

Three important consequences follow — and all three will come up in the field:

1. **NAT is a stateful mechanism.** The router has to remember open connections. This is the same idea as
   the stateful firewall in Phase 9.
2. **The table has a capacity.** A large number of simultaneous connections can fill it, and new connections
   cannot be established.
3. **Entries have a timeout.** A row that has seen no traffic for a long time is **deleted** — and the next
   packet is rejected as an "unknown connection". This is exactly the cause of the "idle SSH session dying"
   problem you saw in Phase 5.6.1.

![Figure 7.1 — NAT/PAT translation: as three internal devices go out from behind the same public IP, the router assigns a unique external port to each connection and stores it in the translation table; returning answers are matched to the right internal device by looking at the destination port.](../diagrams/png/nw-7-01-nat-translation.png)

In the figure there are three internal devices on the left, the router and the translation table in the
middle, and the internet on the right. The outbound arrows mark how the source address changes; the return
arrows mark which field is examined to translate it back. The table itself is at the centre of the figure —
because NAT is entirely that table.

> **🔧 See it on your machine** 🟢 — find out how you look from outside
>
> ```
> $ ip addr show | grep 'inet '
>     inet 127.0.0.1/8 scope host lo
>     inet 192.168.1.50/24 brd 192.168.1.255 scope global eth0   ← your address
>
> $ curl -s ifconfig.me
> 203.0.113.7                                                     ← what the internet sees
> ```
>
> If the two addresses are **different** you are behind NAT — which you almost certainly are. `curl
> ifconfig.me` is in fact a very simple service: the other side looks at the **source IP field** of the
> incoming packet and writes it back to you. So it tells you "this is how I see you". If the two addresses
> are **the same**, your machine has a public IP directly (an instance in a public subnet in the cloud, or a
> server). This one comparison takes NAT out of the abstract.

> **⚠️ Common misconception: "NAT is a security feature."**
>
> Partly true, but a dangerous half-truth. As a side effect of NAT, a connection cannot be established from
> outside to inside **on its own** (7.2.2) — because there is no matching entry in the translation table.
> This provides some protection, but it is **not a deliberate security mechanism**: the purpose of NAT is
> saving addresses. Every connection established from the inside out opens a hole in the table, and traffic
> coming through that hole gets in. If malicious software establishes a connection from the inside out, NAT
> does not stop it at all. Real protection is provided by **firewall rules** (Phase 9) — and in the cloud a
> NAT Gateway does not even have a security group (7.3.2). The correct sentence is: *NAT makes unwanted
> inbound connections harder, but it is not a firewall.*

> **🤔 Think 7.1** — Hundreds of users at an office go out from behind a single public IP and one day the
> complaint comes in: "some sites do not open, some do, and it keeps changing". The router's CPU is normal
> and the link is idle. (a) What is the likely cause? (b) Where does the technical limit behind that cause
> come from? (c) Propose two solutions.
>
> *(Answer: at the end of the phase)*

---
---

# 7.2 PAT and Port Forwarding

## 7.2.1 Port-based multiplexing `[mechanism]`

What we described in 7.1 is in fact **PAT** (Port Address Translation) — the form used at home and in the
office. It is often called "NAT overload" or simply "NAT"; technically the correct name is PAT.

The difference:

- **Basic NAT (1:1):** One external address per internal address. No address saving — only translation. An
  Elastic IP assignment in the cloud works like this.
- **PAT (N:1):** Many internal addresses, one external address. They are distinguished by **port**.

PAT's critical detail is this: if two devices use **the same source port** (which can happen — ephemeral
port selection is random, 1.4.1), the router assigns one of them a **new external port**:

```
192.168.1.50:51234  →  203.0.113.7:51234     (the port stayed the same)
192.168.1.51:51234  →  203.0.113.7:62001     (a clash! the router assigned a new port)
                                    └── now the two can be told apart
```

That is why both the internal and the external port are stored in the translation table (7.1.2). And a
natural limit follows from this: **at most ~64,000 simultaneous connections** can be made behind a single
public IP — because the port field is 16 bits (1.4.1). In practice the limit is lower.

## 7.2.2 Why you cannot reach in from outside `[concept]`

Now think about the opposite direction. Someone on the internet wants to connect to your computer at home.

The only address they have is `203.0.113.7` — the router's address. The packet arrives at the router. The
router looks at the table: **there is no entry for this connection.** Because entries are created by
connections going from the **inside out**.

The only thing the router can do is **drop** the packet. It does not know which device to give it to.

This is NAT's most important behavioural consequence:

> **A service behind NAT cannot be reached from outside on its own — because there is no matching entry in
> the translation table.**

And this is the one-sentence answer to "I cannot connect to my home server from outside".

## 7.2.3 Port forwarding: punching a hole by hand `[concept]`

The solution is to write a **permanent row** into the table. This is called **port forwarding**:

```
Rule: 203.0.113.7:8080  →  192.168.1.50:80
```

You tell the router: *"Forward everything arriving at port 8080, without looking for any entry, to port 80
at this internal address."* Now `http://203.0.113.7:8080` works from outside.

What you need to know:

- The rule is **static** — it stands even when there is no connection.
- One external port can be forwarded to **one** internal target. If you have two servers you need two
  different external ports.
- This **removes** the implicit protection NAT provided for that port — the service is now open to the
  internet and its security rests entirely on the service itself and on firewall rules (Phase 9).

In the cloud the equivalent of this idea is not port forwarding directly; a **load balancer** (Phase 8.5) or
an **Elastic IP + security group** is used. But the logic is the same: access from outside to inside must be
**explicitly defined**.

> **🤔 Think 7.2** — You are running a web server at home (`192.168.1.50:80`). `http://192.168.1.50` works
> from the local network, but when your friend types `http://203.0.113.7` from outside nothing happens.
> (a) Why? (b) What is the solution? (c) What new responsibility have you taken on after applying it?
>
> *(Answer: at the end of the phase)*

---
---

# 7.3 Cloud NAT

This section contains the most confused distinction in cloud networking. Read it carefully — most of the
security mistakes made in the cloud come from here.

## 7.3.1 Public subnet, private subnet `[concept]`

First some concept-cleaning. In the cloud you hear two terms, "public subnet" and "private subnet". These
are **not a property of the subnet** — both are ordinary subnets (Phase 2). The difference is in one single
thing:

> **If its route table has a `0.0.0.0/0 → Internet Gateway` row, that subnet is "public". If not, it is
> "private".**

So the distinction is in the **routing table** you learned about in Phase 4.2. Nowhere else. That single
sentence puts a lot of cloud networking in place.

## 7.3.2 IGW vs NAT Gateway `[concept]`

Two different components, two different jobs:

| | **Internet Gateway (IGW)** | **NAT Gateway** |
|---|---|---|
| Where it sits | Attached to the VPC | Sits in a **public** subnet |
| What it does | Opens a path between resources with a public IP and the internet | Translates private IPs with its own public IP |
| Outbound traffic | ✅ | ✅ |
| **Inbound (initiated) traffic** | ✅ (if there is a public IP) | ❌ **never** |
| Who it is for | Resources in a public subnet that **have a public IP** | Resources in a private subnet |

The most critical row is the bold one: **a connection cannot be established from outside to inside through a
NAT Gateway.** Only the answers to connections initiated from inside come back (7.2.2 — the same mechanism).
This is not a shortcoming, it is **the purpose of the design**: a database in a private subnet should be
able to download updates but must not be reachable from the internet.

And prevent a common mistake here: **an IGW alone is not enough.** Three things are needed together for an
instance to reach the internet:

1. A `0.0.0.0/0 → igw-...` row in the subnet's route table (7.3.1)
2. A **public IP** or Elastic IP on the instance (with no public IP the IGW cannot take it out — there is no
   address to translate to)
3. The security group and NACL allowing it (Phase 9)

If one of the three is missing you get the "I cannot reach the internet" problem — and this is one of the
most frequently diagnosed problems in the cloud.

## 7.3.3 The typical architecture `[concept]`

A standard VPC is built like this:

```
VPC 10.0.0.0/16
│
├── Public subnet  10.0.1.0/24
│     route: 0.0.0.0/0 → IGW
│     inside: the ALB, the bastion host, and the NAT Gateway (with an Elastic IP)
│
└── Private subnet 10.0.2.0/24
      route: 0.0.0.0/0 → NAT Gateway
      inside: the application servers, the database
```

When a server in the private subnet runs `apt update`: the packet goes to the NAT Gateway (the route table),
the NAT GW replaces the source IP with its own Elastic IP (7.1.1), it goes out through the IGW, and the
answer comes back the same way. **Nobody** can connect to that server from outside.

> **💡 Cloud connection — the cost of a NAT Gateway and VPC Endpoints:** A NAT Gateway charges an hourly fee
> **and** a data-processing fee for every GB that passes through it. If hundreds of instances in a private
> subnet are writing large volumes of data to S3, all of that traffic goes through the NAT Gateway and the
> bill grows fast — even though the traffic stays inside AWS. The solution is a **VPC Endpoint**: it
> provides access to services like S3 and DynamoDB from inside the VPC without ever going out to the
> internet. A Gateway-type endpoint (S3, DynamoDB) **adds a row to the route table** — that is, exactly the
> mechanism from Phase 4.2; an Interface-type endpoint (PrivateLink) places an ENI in the subnet and gives
> the service a **private IP**. Both take the NAT Gateway out of the path, lower the cost and keep the
> traffic off the internet. What you need to know for diagnosis: once an endpoint is defined, traffic to
> that service **no longer passes through the NAT GW** — the route table has changed (Phase 11.8).

> **🤔 Think 7.3** — `apt update` does not work on an EC2 instance in a private subnet. The route table has
> a `0.0.0.0/0 → nat-...` row, and the NAT Gateway is up and in a public subnet. (a) Write three possible
> causes. (b) What must be in the route table of the subnet the NAT Gateway sits in? (c) Can you SSH into
> this instance from outside — and why?
>
> *(Answer: at the end of the phase)*

---
---

# 7.4 VPN and Tunnelling

## 7.4.1 Tunnelling: putting a packet inside a packet `[concept]`

You want to connect two private networks (your office and your VPC, say) to each other. But there is the
**internet** between them, and private addresses cannot be routed on the internet (1.3.2).

The solution is surprisingly simple: **put the private packet inside a public packet.**

```
[ Public IP header ] [ Encryption ] [ Private IP header ] [ TCP ] [ data ]
 └── this is what the internet sees ──┘  └── only the ends see this ──┘
```

This is called **tunnelling**. The outer packet is routed normally between two public addresses; the routers
along the way do not know and do not care what is inside. On arrival the outer envelope is stripped and the
inner private packet carries on as if it had never crossed the internet.

Think of it as one more application of the **encapsulation** idea you learned in Phase 0.2: each layer adds
its own envelope; tunnelling puts **an entire packet** inside a new envelope.

Types of tunnel:

| Name | What it does | Depth |
|---|---|---|
| **IPsec** | An encrypted tunnel — connects two private networks securely | `[concept]` |
| **GRE** | A general-purpose, **unencrypted** tunnel | `[skip]` — know the name |
| **VXLAN** | An overlay carrying L2 over L3 | `[skip]` — know the name |
| **WireGuard / OpenVPN** | Modern remote-access VPNs | `[concept]` |

IPsec has two jobs and do not confuse them: **encryption** (hides the content) and **authentication**
(proves the other side really is who it says). GRE has neither — GRE only carries; if security is wanted it
is generally used together with IPsec.

## 7.4.2 The price of a tunnel: MTU `[mechanism]`

And here is where the problem you met in Phase 5.7 explodes.

The outer envelope **takes up space.** The headers and encryption overhead IPsec adds are typically between
50 and 100 bytes. That is:

```
Normal:      1500-byte MTU  →  1460 bytes of data (MSS)
In a tunnel: 1500-byte MTU  →  ~1400 bytes of data or less
             └── the outer header + encryption overhead ate this
```

If the machines at the ends do not know this — and by default they do not — they keep sending 1500-byte
packets. At the tunnel entrance those packets **do not fit**, they are dropped, and an **ICMP Fragmentation
Needed** should be sent to the sender (4.4.2). If that ICMP is blocked: an **MTU black hole** (5.7.3).

The symptom is the same again and you recognise it now:

> **Ping works, SSH connects, but a large data transfer freezes.**

Two solutions (the same ones you saw in 5.7.3): **MSS clamping** (the tunnel device lowers the MSS in SYN
packets — it is a standard setting on VPN devices and should generally be on by default) or **allowing ICMP
type 3 code 4** (so path MTU discovery works by itself).

That is why these two phases are continuations of each other: **if you have built a tunnel, think about
MTU.**

> **💡 Cloud connection — Site-to-Site VPN and route learning:** In AWS the standard way to connect an
> office network to a VPC is a **Site-to-Site VPN**, and IPsec runs underneath. The setup has three parts: a
> **Virtual Private Gateway** (or Transit Gateway) on the VPC side, a **Customer Gateway** on the office
> side, and two **tunnels** between them (two are always built, for redundancy). There are two options for
> how routes are learned: **static** (you write them by hand) or dynamic via **BGP** (Phase 4.6) — the
> second is preferred, because when one tunnel goes down traffic shifts to the other by itself. And the
> first problem you meet after setup will most likely be MTU (7.4.2): the tunnel is up, ping works, but
> database queries freeze. If you need higher bandwidth and stable latency, **Direct Connect** (a dedicated
> physical link) is used; it too learns routes with BGP. If you need to connect several VPCs and offices,
> instead of peering each pair individually a **Transit Gateway** is used — it works like a central router
> in a hub-and-spoke topology (Phase 11.7).

> **🤔 Think 7.4** — Over a newly built Site-to-Site VPN, someone at the office connects to a file server in
> the VPC. `ping` works, the directory listing comes through, but when a 10 MB file starts downloading the
> transfer freezes at 2%. (a) What is your diagnosis and why are you so sure? (b) With which command do you
> confirm it? (c) Which setting on the VPN device fixes this permanently?
>
> *(Answer: at the end of the phase)*

---
---

# 7.5 IPv6 and a World Without NAT

## 7.5.1 Abundance, the cure for scarcity `[concept]`

The reason NAT exists is **address scarcity** (7.1). In IPv6 that problem does not exist.

IPv4: 32 bits → ~4.3 billion addresses.
IPv6: 128 bits → a number that is practically impossible to exhaust (1.5).

The result: in IPv6 **every device can have its own public address.** No translation is needed. This brings
three things:

1. **End-to-end connectivity comes back.** Devices can reach each other directly; the protocols NAT broke
   (peer-to-peer applications, some VoIP protocols) work properly.
2. **There is no translation table** — filling up, timeouts and port exhaustion disappear (7.1.2).
3. **Security becomes entirely the firewall's job.** There is no implicit protection from NAT (the box in
   7.1.2) — every device is theoretically reachable, so **firewall rules are mandatory** (Phase 9). This is
   not a weakness; it is moving the responsibility to the right place.

## 7.5.2 Dual-stack and its trap `[concept]`

The transition did not happen in a day and will not. So most systems run **dual-stack**: they speak IPv4 and
IPv6 **at the same time**. When a name is queried both an A and an AAAA record can come back (6.3.1) and the
client picks one.

And the trap is exactly here: **if the application picks the wrong family**, the symptom is:

> **"It works on one protocol and hangs on the other."**

The typical scenario: the server publishes an AAAA record, the client prefers IPv6 (that is the default on
modern systems), but the IPv6 connectivity on the path is broken or the firewall never thought about IPv6.
The client tries IPv6, waits for a timeout, then falls back to IPv4 — and the user experiences this as
**"the site is very slow to open"**. (Modern clients mitigate this with a method called **Happy Eyeballs**:
they try both families almost simultaneously and use whichever answers first.)

The reflex in diagnosis is simple: **test the two families separately** with `curl -4` and `curl -6`. If one
works and the other does not, you have found the problem.

> **💡 Cloud connection — dual-stack VPCs and the egress-only IGW:** In AWS you can add an IPv6 block to a
> VPC and the subnets run dual-stack. But there is a conceptual difference here: because there is no NAT in
> IPv6, the idea of a "private subnet" works differently too — every instance can have a public IPv6
> address. If you still want "outbound only" (the IPv6 equivalent of a private subnet), an **egress-only
> Internet Gateway** is used: it is the IPv6 equivalent of a NAT Gateway, it translates no addresses but
> **blocks connections initiated from outside** (the same logic as 7.3.2, without the translation). You also
> have to write security group and NACL rules **separately for the two families** — `0.0.0.0/0` does not
> cover IPv6, `::/0` must be written separately. This is exactly the most common cause of "it works on IPv4
> but not on IPv6" problems in the cloud: a forgotten `::/0` rule (Phase 9.4).

---
---

# 7.6 When This Phase Breaks — NAT Failure Signatures

The common theme of NAT failures is this: **the translation table.** The table fills up, the table forgets,
or there is no entry in the table at all. All three have a different signature and all of them lead back to
that table.

| Symptom | Likely cause | Verify | Related section |
|---|---|---|---|
| An internal service cannot be reached from outside | No entry in the NAT table — normal behaviour | Is port forwarding/an LB defined | 7.2.2 |
| Idle connections drop after a while | A NAT session timeout | Add keepalive, find the source of the RST | 7.1.2, 5.6.1 |
| Random connection errors at peak hours | The NAT table / port pool is full | The simultaneous connection count, an extra public IP | 7.2.1 |
| `curl ifconfig.me` ≠ `ip addr` | **Normal** — you are behind NAT | — | 7.1.1 |
| An instance with a public IP cannot reach the internet | No IGW in the route table, or no public IP, or the SG | Check the three conditions one by one | 7.3.2 |
| A private instance cannot reach the internet | No `0.0.0.0/0 → nat-...` route, or the NAT GW is in a private subnet | The NAT GW's subnet route table | 7.3.3 |
| The NAT Gateway is up but no traffic passes | No IGW route in the NAT GW's subnet | That subnet must be public | 7.3.2 |
| The VPN tunnel is up, ping ✓, transfers freeze | **The tunnel's MTU** — an MTU black hole | `ping -M do -s 1472` | 7.4.2, 5.7.3 |
| The VPN is up but some subnets are unreachable | Routes not learned (static missing / no BGP) | The route table, the BGP neighbourship | 7.4.2, 4.6 |
| It works on IPv4 but not on IPv6 | Dual-stack — a missing `::/0` rule or a broken IPv6 path | `curl -4` vs `curl -6` | 7.5.2 |
| The site is slow to open, then normal | IPv6 is tried, times out, falls back to IPv4 | Test directly with `curl -6` | 7.5.2 |
| The bill explodes when writing to S3 | The traffic goes through the NAT Gateway | Define a VPC Endpoint | 7.3.3 |

> **The lesson from this table:** In NAT diagnosis ask this first: **who initiated the connection?** If it
> was initiated from the inside there **is** a row in the translation table and the return should work — if
> it does not, either the row has timed out (7.1.2) or the table is full (7.2.1). If it was initiated from
> outside there is **no** row and the packet is dropped; that is not a failure, it is **the design** (7.2.2)
> — the solution is an explicitly defined entry point (port forwarding, a load balancer, an Elastic IP). In
> the cloud the second question is: **what does the route table say?** "Public subnet" and "private subnet"
> are not properties of the subnet but the contents of its route table (7.3.1) — and most "I cannot reach
> the internet" problems are solved there, in a single missing row. Finally: **if you have built a tunnel,
> think about MTU** (7.4.2). The moment you hear the sentence "ping works but transfers freeze", the VPN
> having been recently installed makes the diagnosis nearly certain.

---
---

# Answers to the Think questions

## Answer 7.1 — The port pool is exhausted

**(a)** **Exhaustion of the NAT/PAT port pool** (7.2.1). If hundreds of users are behind a single public IP
and each user opens dozens of simultaneous connections (a modern web page alone can open 20–50), the total
number of simultaneous connections fills the port pool. The router cannot create a new translation row and
the connection fails **silently**. The fact that it is "random and changing" is the typical signature: which
connection fails depends on whether there is room in the pool at that moment.

**(b)** **The port field is 16 bits** (1.4.1) — behind a single public IP about 65,000 simultaneous
translations are theoretically possible. In practice fewer: some are reserved, and rows for closed
connections linger in the table for a while (a TIME_WAIT-like delay, 5.6.1). The router's own table capacity
is a separate limit too.

**(c)** Two solutions: (i) **Widen the public IP pool** — define more than one public IP and let the router
use them in turn; each extra address means ~65,000 more translations. (ii) **Reduce the number of
connections** — spread the use of connection pooling / keep-alive at the application level, and shorten the
NAT timeout values so unused rows are freed faster. (The cloud equivalent of the same problem: hitting a
single NAT Gateway's port limit — the solution is several NAT Gateways or extra Elastic IPs.)

**Related section:** 7.1.2, 7.2.1 · **Next:** Phase 9.1 (the stateful table), Phase 10.4 (capacity
diagnosis)

## Answer 7.2 — No entry in the table

**(a)** Because your friend's packet is coming **from outside to inside** and there is **no** entry for this
connection in the router's translation table (7.2.2). Entries are created only by connections going from the
**inside out**. The router cannot know which internal device to give the packet to, so it **drops** it.

**(b)** **Port forwarding** (7.2.3): write a permanent rule on the router — `203.0.113.7:80 →
192.168.1.50:80`. Now every packet arriving at that port is forwarded straight to that internal address
without looking for an entry. (Alternatives: using a reverse proxy/tunnel service, or moving the server to
the cloud and giving it a public IP.)

**(c)** **The responsibility for security.** Port forwarding **removes** the implicit protection NAT
provided for that port (the box in 7.1.2). That service is now open to the whole internet: automated
scanners will find it within minutes. Your responsibility: keep the service up to date, put authentication
in front of it, turn off unnecessary features and, if possible, restrict source addresses with a firewall
rule (Phase 9). NAT was protecting you; it no longer is.

**Related section:** 7.2.2–7.2.3 · **Next:** Phase 9.2 (writing rules), Phase 8.5 (reverse proxies)

## Answer 7.3 — The NAT Gateway's own path

**(a)** Three causes: (i) **the route table of the subnet the NAT Gateway sits in has no `0.0.0.0/0 → IGW`**
— a NAT GW has to be in a public subnet, otherwise it cannot reach the internet itself and nothing gets
through (7.3.2). (ii) **A security group or NACL** is blocking outbound traffic (or the returning traffic —
NACLs are stateless) (Phase 9). (iii) **DNS is not resolving** — `apt update` requires name resolution; if
the VPC resolver is unreachable the command fails even with NAT intact (6.5, 6.3.3). A fourth possibility:
the NAT Gateway has **no Elastic IP**, or it is in a different AZ and the cross-AZ path is down.

**(b)** The NAT Gateway's subnet must be **public**: its route table must have a **`0.0.0.0/0 → igw-...`**
row (7.3.1, 7.3.3). This is the most common setup mistake — if the NAT Gateway is placed in a private
subnet, nothing works.

**(c)** **No.** A NAT Gateway **never** lets a connection initiated from outside in (7.3.2) — there is no
matching entry in the translation table (7.2.2, the same mechanism). This is not a failure but the purpose
of the design. To connect you either hop through a **bastion host** in a public subnet, or use a service
that builds a reverse connection such as **SSM Session Manager** (the instance connects outward itself and
you join that session — no inbound port is opened at all).

**Related section:** 7.3.1–7.3.3 · **Next:** Phase 9.4 (SG vs NACL), Phase 11.4 (the IGW/NAT GW mapping)

## Answer 7.4 — Tunnel + MTU = the classic

**(a)** An **MTU black hole** (7.4.2, 5.7.3). The reason you can be this certain is that **the signature
fits exactly**: small packets get through (ping ✓, the directory listing = small data ✓), large packets do
not (the file transfer ✗). And the VPN was **recently built** — the 50–100 bytes of header IPsec adds have
brought the path MTU below 1500 and the ends do not know it. The transfer "freezing at 2%" is typical too:
the first small packets get through, and the moment the window grows and full-sized segments start being
sent the flow stops (5.5.1).

**(b)** **`ping -M do -s 1472 <target>`** (5.7.3). If that does not get through over the tunnel, the path
MTU is below 1500. Reduce the size to find the largest value that gets through and add 28 — that gives you
the tunnel's real MTU (typically around 1400). An additional confirmation: the `retrans` value in `ss -ti`
output climbing quickly during the transfer (5.3.2).

**(c)** **MSS clamping** (it goes by the names *TCP MSS adjust* or *clamp MSS to PMTU*). The VPN device
lowers the MSS value in SYN packets passing through the tunnel according to the tunnel's MTU (5.7.2); so the
two ends agree **from the start** to use small segments and a large packet is never created. This setting
exists on most VPN devices and should be considered a **standard part** of tunnel setup. A complementary
measure: **allowing ICMP type 3 code 4** (4.4.2), so path MTU discovery works too. The two are applied
together.

**Related section:** 7.4.2 · **Next:** Phase 9.3 (ICMP policy), Phase 10.3 (layer-by-layer diagnosis)

---
---

# Frequently asked questions

**Q1 — What exactly is the difference between NAT and PAT?** **NAT (1:1)** maps one external address to each
internal address — there is no address saving, only translation (an Elastic IP assignment in the cloud works
like this). **PAT (N:1)** squeezes many internal addresses onto a single external address and distinguishes
them by **port** (7.2.1). In everyday speech "NAT" almost always means PAT.

**Q2 — Does NAT provide security?** As a side effect, a connection cannot be established from outside to
inside on its own (7.2.2), but this is **not a deliberate security mechanism**. Every connection established
from the inside out opens a hole, and malicious software uses that freely. Real protection is provided by
firewall rules (Phase 9). The correct sentence: *NAT makes it harder, it does not protect.*

**Q3 — Why do some applications not work behind NAT?** Because some protocols carry an IP address **inside**
the packet (old FTP, some VoIP/SIP protocols) or expect the other side to **initiate a connection** to them.
NAT changes the outer address but knows nothing about the inner information, and it does not accept an
inbound connection either. That is why special "NAT traversal" methods (STUN/TURN, UPnP, hole punching) were
developed. In IPv6 this problem largely disappears (7.5.1).

**Q4 — What is the difference between a NAT Gateway and a NAT instance?** A NAT Gateway is an AWS-managed,
scaling, highly available service. A NAT instance is an ordinary EC2 configured to do NAT — it is cheap but
it is alone (if it fails, egress stops), scaling it is your job, and if you forget to disable the
"source/destination check" setting it does not work at all. In production a NAT Gateway is preferred.

**Q5 — How do I connect to an instance in a private subnet?** Three ways: (i) an SSH hop through a **bastion
host** in a public subnet; (ii) **SSM Session Manager** — the instance connects outward itself and no
inbound port is opened (the safest and the preferred way today); (iii) connecting the office network to the
VPC with a VPN or Direct Connect and reaching it directly over its private IP (the box in 7.4.2).

**Q6 — Why is there always an MTU problem in a tunnel?** Because a tunnel puts **an entire packet** inside a
new envelope (7.4.1) and that envelope takes up space. The machines at the ends are unaware the tunnel
exists; they still send 1500 bytes. The notification channel is ICMP, and ICMP is frequently blocked
(5.7.3). That is why MSS clamping should be considered a standard part of tunnel setup (7.4.2).

**Q7 — Will NAT disappear entirely once we move to IPv6?** Largely yes — since there is no address scarcity,
no translation is needed (7.5.1). But the need for "outbound only" remains, and its equivalent is not
translation but **filtering** (an egress-only Internet Gateway in AWS, the box in 7.5.2). So NAT goes and
the firewall stays — and the responsibility moves to the right place, to explicit rules.

---
---

# Test yourself

Write your answers on paper, then compare them with the key. Target: 14+ out of 18.

## Part A — Definitions and mechanisms

1. Why was NAT invented? What concrete constraint required it?
2. How does a router match a returning answer to the right internal device? What structure does it use?
3. What is the difference between NAT (1:1) and PAT (N:1)?
4. How many simultaneous connections can be made behind a single public IP, and where does that limit come
   from?
5. What is port forwarding and how does it differ from a normal NAT entry?
6. What is tunnelling? Describe it in one sentence.
7. What are IPsec's two jobs? How does it differ from GRE?
8. What does dual-stack mean?

## Part B — Apply and diagnose

9. With which single command do you find out how your machine looks from outside?
10. How do you tell that a subnet in the cloud is "public"?
11. An instance with a public IP cannot reach the internet. What three things must you check?
12. An instance in a private subnet cannot reach the internet and the route has `0.0.0.0/0 → nat-...`. Where
    do you look next?
13. A new VPN has been built, ping ✓, file transfer ✗. Diagnosis and the confirming command?
14. "It works on IPv4 but not on IPv6" — what are your first two commands?

## Part C — Reasoning and connections

15. Why is the sentence "NAT is a firewall" wrong? How is it set up correctly?
16. Is the fact that a connection cannot be established from outside through a NAT Gateway a shortcoming or
    the design? Explain with the mechanism.
17. Why is the MTU problem in a tunnel **unavoidable**? Which two measures are taken?
18. When moving to IPv6, which function of NAT disappears and which need remains?

---

## Answer key

1. **IPv4 address scarcity** — 32 bits, ~4.3 billion addresses, billions of devices. Because private
   addresses cannot be routed on the internet (1.3.2), translation on the way out became necessary (7.1.1).
   — 2. By looking at the **translation table** (the NAT/connection tracking table): it stores the internal
   address:port ↔ external address:port ↔ destination mapping, and on a returning packet it looks at the
   destination field, finds the row and translates it back (7.1.2). — 3. **NAT (1:1):** one external address
   per internal address, no saving. **PAT (N:1):** many internal addresses on one external address,
   distinguished by **port** (7.2.1). — 4. Theoretically ~65,000 (fewer in practice); the limit comes from
   the **port field being 16 bits** (1.4.1, 7.2.1). — 5. A **permanent, static** row written into the
   translation table by hand: it forwards every packet arriving at a particular external port to a
   particular internal target without looking for an entry. Normal entries are created **temporarily** by
   connections going from the inside out (7.2.3). — 6. Putting a packet (headers and all) **inside another
   packet** and sending it across the public internet; at the destination the outer envelope is stripped
   (7.4.1). — 7. IPsec: **encryption** (hides the content) + **authentication** (proves the other side). GRE
   has neither, it only carries (7.4.1). — 8. Speaking IPv4 and IPv6 **at the same time**; both an A and an
   AAAA record can come back for a name (7.5.2).

9. **`curl ifconfig.me`** — compare the address it returns with `ip addr` (7.1.1). — 10. **If its route
   table has a `0.0.0.0/0 → Internet Gateway` row** it is public. It is not a property of the subnet itself
   (7.3.1). — 11. (i) `0.0.0.0/0 → igw-...` in the route table; (ii) does the instance have a **public/
   Elastic IP**; (iii) do the security group and NACL allow it (7.3.2). — 12. At **the route table of the
   subnet the NAT Gateway sits in** — it must have `0.0.0.0/0 → igw-...`, that is, the NAT GW must sit in a
   public subnet (7.3.2, 7.3.3). Then the SG/NACL and DNS. — 13. An **MTU black hole**; confirmation: **`ping
   -M do -s 1472 <target>`** (7.4.2, 5.7.3). — 14. **`curl -4 <target>`** and **`curl -6 <target>`** — test
   the two families separately (7.5.2).

15. Because the purpose of NAT is **saving addresses**; the inability to connect from outside is a **side
    effect** (the box in 7.1.2). Every connection established from the inside out opens a hole in the table
    and traffic through that hole gets in; malicious software uses it freely. The correct setup: access
    decisions are made with **explicit firewall rules** (Phase 9) and NAT only translates. — 16. **The
    design.** A NAT Gateway's translation table only has entries for connections **initiated from inside**;
    because there is no row corresponding to a packet arriving from outside, it cannot be known which
    internal address to give it to and the packet is dropped (7.2.2, 7.3.2). That is the whole purpose:
    private resources should be able to download updates but not be reachable from the internet. — 17.
    Because a tunnel puts the entire packet inside a new envelope and the envelope **takes up space** (50–100
    bytes); the ends are unaware of the tunnel and still send 1500 bytes. The notification channel is ICMP
    and it is frequently blocked (5.7.3). Two measures: **MSS clamping** and **allowing ICMP type 3 code 4**
    (7.4.2). — 18. What disappears: **address translation** — thanks to address abundance every device can
    have a public address (7.5.1). What remains: the **"outbound only"** restriction — but it is now provided
    by **filtering** rather than translation (an egress-only IGW, firewall rules) (7.5.2).

## Scoring

| Correct answers | What it means |
|---|---|
| 16–18 | NAT and cloud network architecture have settled. You are ready for Phase 8. |
| 13–15 | Good. Read 7.3 (IGW vs NAT GW) once more — that is the most critical distinction in the cloud. |
| 9–12 | The translation-table idea may not have fully settled. Redo 7.1.2 and 7.2.2 together. |
| 0–8 | Walk the phase again. The goal: to think of the table the moment you hear "it cannot be reached from outside". |

**Which section to go back to for a question you missed:**

| Question | Section |
|---|---|
| 1, 2, 9 | 7.1 The logic of NAT |
| 3, 4, 5, 15, 16 | 7.2 PAT and port forwarding |
| 10, 11, 12 | 7.3 Cloud NAT |
| 6, 7, 13, 17 | 7.4 Tunnelling and MTU |
| 8, 14, 18 | 7.5 IPv6 and dual-stack |

---
---

# Closing and the Bridge to Phase 8

## What you carry from this phase

Phase 7 gave you **the address arrangement of the real world**. You learned why NAT exists, how the
translation table works and why returning traffic reaches the right device thanks to the port. You saw that
being unreachable from outside is not a failure but **the design**. You clarified that the distinction
between a "public subnet" and a "private subnet" in the cloud is really **a single row in a route table** —
and that an IGW and a NAT Gateway do different jobs. You learned tunnelling and its unavoidable MTU price.
Finally you saw why IPv6 makes NAT unnecessary and what trap dual-stack brings.

The three most durable sentences: **"NAT makes it harder, it does not protect"**, **"a public subnet means a
subnet whose route table has an IGW row"** and **"if you have built a tunnel, think about MTU."**

## Where Phase 8 connects

Up to here you have learned how to **deliver** a packet to its destination: the address (Phase 1), the range
(Phase 2), local delivery (Phase 3), routing (Phase 4), reliability (Phase 5), the name (Phase 6),
translation (Phase 7).

So when you type `curl https://example.com` you can now explain **from start to finish** how the packet
travels.

But we have never talked about one thing: **what is inside the packet?**

How does the server know what you wanted? How is "give me this page" expressed? Why is the difference
between a 404 and a 502 so valuable in diagnosis? And that `s` in `https` — traffic being unreadable by
anyone along the way — exactly **when and how** is it established?

Phase 8 is the answer: HTTP and TLS. You will see the request/response structure, the diagnostic value of
status codes, the steps of the TLS handshake, certificate validation, and the reverse proxy / CDN / anycast
architecture.

> **🤔 Phase output — ask yourself:** TLS encrypts the traffic. But for encryption both sides need a **shared
> key**. How will they share that key — over a channel where everyone along the way can listen and which is
> not yet encrypted? (Hint: they do not send the key. Both sides **compute it separately**.)
>
> **🧪 Lab 7 idea (all 🟢):** (1) Note your private address with `ip addr show`, get your public appearance
> with `curl -s ifconfig.me`, and compare the two. (2) See which interface and gateway you go out through
> with `ip route get 8.8.8.8` (4.3.1) — that is the device doing the NAT. (3) Repeat `curl -s ifconfig.me`
> while connected to your phone's hotspot; did the public IP change? (4) Open your home router's interface
> and find the NAT/connection table (usually under "NAT table", "Active connections" or "Session list") — see
> the table from 7.1.2 **in the flesh**. (5) Run `curl -4 example.com` and `curl -6 example.com`; do both
> work, does one give an error? (6) Measure your path MTU again with `ping -M do -s 1472 8.8.8.8` — does the
> result change with the VPN on and off?

---

> **Navigation:** [◀ Phase 6 — From Name to Address: DNS](Phase_6_DNS.md) · **Phase 7** · [Phase 8 — The Application Layer: HTTP + TLS ▶](Phase_8_HTTP_and_TLS.md)
