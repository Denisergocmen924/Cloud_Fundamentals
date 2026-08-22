# Cloud Engineering — Network Fundamentals Roadmap

---

## Mentor Startup Instructions — for the AI reading this file

This is the network fundamentals roadmap, and you are the **Socratic mentor** of the learner working through it. If this file has been handed to you, adopt the rules below and wait for the learner to set the direction. When the learner says "**We're on Phase X**" (from the start of the phase) or "**We're on Phase X.Y**" (from a sub-item), continue **without interruption** from that point — don't summarize from the beginning, don't ask permission, don't wander.

**Learning rhythm — the most important rule: the constructive Socratic method.** This is neither pure Q&A nor plain lecturing; it is a blend of the two. For every concept the order is: **(1) ask a guiding question** — like "what do you think needs to be true for X to happen?", a question the learner can reason through **with the knowledge they already have** → **(2) the learner guesses** ("could it be such-and-such?") → **(3) if correct, confirm immediately and give the term:** "yes, exactly — this is called **X**, and it means: …" → **(4) if wrong, don't fuss with a roundabout question, correct it directly** → **(5) once the term is placed, build the concept up together.**

Critical rule: **Do not try to make the learner guess a concept they have no grip on at all.** The question is not for open-ended discovery, but to **build a bridge** from what they know to what they don't. As soon as the answer arrives, name it, define it, don't leave the concept hanging. You do not move to the next concept before the current one is complete.

**Phase opening.** When entering a phase, first give ~2 lines of orientation: what this phase covers + what you assume they know from the previous phase + the general context. Then move to a short explanation of the first concept.

**Language and style.**
- English. Give a technical term a short definition in parentheses on first use (as with the in-file examples like "header", "CIDR", "DHCP").
- Follow the depth labels: `[concept]` enough to know the definition, `[mechanism]` explain step by step, `[application]` have it seen on the machine, `[skip]` awareness only.
- Tie it to cloud examples; use the **"When it breaks"** angle of every topic — the failure instinct is the heart of this map.

**Correction — keep it direct.** When the learner answers wrong, don't hunt for a second question to lead them to the right answer; **state the correct answer clearly**, then give a brief rationale. Indirect steering slows things down and tires people out. When the learner says "just tell me directly," explain directly, no argument.

**Lab.** **Immediately after** the related `[application]` concept, suggest that phase's Lab (with the commands in the file). This is a **suggestion, not a demand.** If the learner runs the command and brings back the output, you read it together. If they say "I'm not at my machine right now" or "later," **don't force it, don't insist, don't break the flow** — jot a short note and move on. The decision is the learner's; you follow their decision.

**Checkpoint quizzes.** When you reach the checkpoints marked with ✅ (end of Phases 0–2, 3–4, 5–9), **suggest the checkpoint quiz automatically.** If the learner wants to defer, **defer** — don't impose.

**Communication.** No trigger/command system — talk naturally. **The moment you don't understand the learner, don't assume you do; say plainly "I didn't quite understand you"** and ask a clarifying question.

**Boundaries.** One concept at a time; no topic skipping; no unrequested roadmap drift. **Don't track** in-phase progress yourself — the learner manages it and pins the position by saying "We're on Phase X.Y." Feedback is honest and direct.

---

> **North star — three instinct questions.**
> The single goal of this map is that one day you can answer these three questions without thinking:
> 1. **How does data move forward?** → encapsulation + the packet's journey (Phases 0, 1, 4, 5)
> 2. **Why might a packet have been dropped?** → congestion, MTU, firewall, TTL, buffer (Phases 5, 9, 10)
> 3. **Why is the router acting up?** → routing table, ARP, NAT, MTU black hole (Phases 3, 4, 7)
>
> Design principle: troubleshooting was not left for the end. Every phase has the
> **"how does this show up when it breaks?"** angle. Instinct forms only this way.

## Depth labels (same system as the physical-layer map)

- `[concept]` — just know what it is, be able to give its definition.
- `[mechanism]` — be able to explain step by step how it works.
- `[application]` — run it by hand on your own Ubuntu/Linux machine, see it.
- `[skip]` — awareness is enough, don't go deep.
- **Cloud connection** — where the topic maps to in AWS.
- **When it breaks** — how this topic shows up in the system when it fails (an instinct trigger).

---

## Phase 0 — Mental Model: Why Do Layers Exist?

The goal of this phase: to build the network not as a memorized list of protocols but as a **fault map**. Without internalizing encapsulation, no troubleshooting instinct forms.

### Topics

**0.1 The OSI and TCP/IP model**
- The layer idea: why we slice a problem into pieces `[concept]`
- OSI's 7 layers vs the 4-layer TCP/IP used in practice `[concept]`
- Each layer talks only **to its own peer** (the peer-to-peer illusion) `[concept]`

**0.2 Encapsulation and Decapsulation**
- Each layer wrapping the data in its own header `[mechanism]`
- Wrapping when sending (encapsulation), unwrapping when receiving (decapsulation) `[mechanism]`
- **When it breaks:** if a header is corrupted at the wrong layer, at which layer does the error show up?

**0.3 PDUs — the data's name at each layer**
- Frame (L2) → Packet (L3) → Segment (L4) → Data (L7) `[concept]`
- Why the same data takes a different name at each layer `[concept]`

> **Phase 0 output:** "When I send a ping, how does that data descend through the 7 layers and come back up on the other side" — you should be able to draw it in your own words.

---

## Phase 1 — Addressing: How Do We Find a Machine?

The goal of this phase: to understand why pointing to a device uniquely on the network is needed at **two separate layers** (L2 and L3). This is the precondition for ARP and routing.

### Topics

**1.1 MAC address (L2 identity)**
- 48-bit hardware address, burned into the NIC `[concept]`
- OUI (vendor) + device part `[skip]`
- Why MAC is meaningful only **on the local network** `[concept]`

**1.2 IPv4 anatomy (L3 identity)**
- 32-bit address, 4 octets, dotted-decimal (`192.168.1.10`) `[mechanism]`
- Binary ↔ decimal conversion (connects from the Phase 0 digital foundation) `[application]`
- The network part vs host part distinction `[concept]`

**1.3 Public vs Private IP**
- RFC 1918 private ranges (10/8, 172.16/12, 192.168/16) `[concept]`
- Why a private IP can't be routed on the internet `[mechanism]`
- **Cloud connection:** private IP inside a VPC, where an EC2's public IP comes from

**1.4 Port and Socket**
- Port: separating different services on the same machine `[mechanism]`
- Well-known ports (22, 53, 80, 443, 3306) `[concept]`
- Socket = the (IP : Port) pair `[concept]`
- **When it breaks:** what the "port already in use" error means exactly

**1.5 IPv6 (awareness)**
- Why it exists: IPv4 address exhaustion `[concept]`
- 128-bit, hex notation `[skip]`

**1.6 DHCP — getting an IP automatically**
- DHCP (automatic IP-assignment protocol): dynamic assignment instead of a static IP `[concept]`
- The DORA cycle: Discover → Offer → Request → Ack `[mechanism]`
- DHCP hands out not just the IP; it also distributes the subnet mask + default gateway + DNS server `[mechanism]`
- **Cloud connection:** DHCP option sets in a VPC — an EC2 gets its private IP and DNS from here
- **When it breaks:** if DHCP fails, Windows gives itself a 169.254.x.x (APIPA); Ubuntu (NetworkManager) usually gets no IPv4 at all → "no IP, no network"

> **Lab 1:** Find your machine's MAC + IP with `ip addr` and `ip link`. See which ports are listening with `ss -tulpn`.

---

## Phase 2 — Subnet and CIDR (The Heart of the Cloud)

The goal of this phase: the most critical phase, and the one to move through most slowly. The whole of VPC design, subnet splitting, and writing security groups rests on this. You know binary from Phase 0; now we'll use it to split the address space.

### Topics

**2.1 Subnet mask logic**
- The mask drawing the network/host boundary `[mechanism]`
- `255.255.255.0` ↔ its binary equivalent `[mechanism]`
- Finding the network address with the AND operation `[mechanism]`

**2.2 CIDR notation**
- What `/24`, `/16`, `/8` mean (network-size notation) `[mechanism]`
- The bigger the prefix, the smaller the network (inverse relationship) `[concept]`
- Address-count calculation: `2^(32 - prefix)` `[application]`

**2.3 Network / Broadcast / Usable range**
- In every subnet the first address (network) and last address (broadcast) are reserved `[mechanism]`
- Usable host count = total − 2 `[mechanism]`
- **Cloud connection:** AWS reserves **5 IPs** in each subnet (not 2) — why

**2.4 Subnetting practice**
- Splitting a `/16` into `/24`s `[application]`
- Why a subnet overlap is a disaster `[concept]`
- **When it breaks:** what happens if CIDRs overlap when peering two VPCs

> **Phase 2 output:** If I give you `10.0.0.0/22` — how many hosts, the network address, the broadcast address, the usable range — you should be able to work it out with pen and paper.
>
> **Lab 2:** Verify your answers with `ipcalc 10.0.0.0/22`.

---

## ✅ Checkpoint Quiz 1 (Phases 0–2)

Encapsulation, addressing, subnet. Entering L2/L3 before passing these three is meaningless.

---

## Phase 3 — Inside the Same Network (The L2 World)

The goal of this phase: if two devices are on the **same** subnet, how the data physically finds its way to each other. In the physical-layer roadmap we touched on switch hardware (Phase 5.2); here is the protocol side.

### Topics

**3.1 ARP (Address Resolution Protocol)**
- The IP is known but the MAC isn't — ARP fills this gap `[mechanism]`
- ARP request (broadcast) → ARP reply (unicast) `[mechanism]`
- ARP cache and timeout `[concept]`
- **When it breaks:** ARP poisoning / a wrong ARP cache → traffic goes to the wrong place

**3.2 Switch and MAC table**
- The switch builds a table by learning MAC addresses `[mechanism]`
- Unknown destination → flooding `[concept]`
- The L2 root of the question **why the router is acting up** starts here

**3.3 Broadcast domain vs Collision domain**
- A switch splits the collision domain, not the broadcast domain `[concept]`
- A router splits the broadcast domain `[concept]`

**3.4 VLAN (awareness)**
- Logical network separation on a single physical switch `[skip]`

> **Lab 3:** See your ARP cache with `ip neigh`. Query a neighbor with `arping`, watch the request/reply.

---

## Phase 4 — Between Networks: Routing (L3)

The goal of this phase: if the destination is on a **different** network, how the data reaches it. The core of the question "why is the router acting up." The layer a cloud engineer debugs most.

### Topics

**4.1 Default gateway**
- Everything not on the local network is sent to the gateway `[mechanism]`
- How a device decides "is this destination in my subnet?" `[mechanism]`
- **When it breaks:** a wrong/missing gateway → "the same network works, the internet doesn't"

**4.2 Routing table**
- Every device has a routing table (not just routers) `[mechanism]`
- The Destination, gateway, interface, metric columns `[concept]`
- **Cloud connection:** the AWS route table = the cloud counterpart of this table

**4.3 Longest prefix match**
- If multiple routes match, the **most specific** one (longest prefix) wins `[mechanism]`
- The heart of the router's decision logic `[mechanism]`

**4.4 TTL and ICMP**
- TTL (Time To Live) decreases by 1 at each hop, the packet dies at 0 `[mechanism]`
- Loop protection: kills packets circling forever `[concept]`
- ICMP: error and diagnostic messages (ping uses this) `[concept]`
- **When it breaks:** a routing loop → a flood of TTL exceeded

**4.5 Traceroute mechanics**
- The trick of exposing each hop by increasing TTL as 1, 2, 3... `[mechanism]`
- Why some hops show `* * *` `[concept]`
- If you've inspected a real traceroute output before (an ISP → city → city hop chain), it connects here

**4.6 Dynamic routing and BGP (awareness)**
- The difference between a static route (hand-written) vs a dynamic route (learned by a protocol) `[concept]`
- BGP (the internet's backbone routing protocol): route advertisement between ASes (autonomous system = an ISP's network block) `[concept]`
- Interior routing protocols (OSPF, RIP) — name awareness only `[skip]`
- **Cloud connection:** Direct Connect, Site-to-Site VPN, and route propagation work with BGP
- **When it breaks:** a wrong BGP advertisement (route leak/hijack) → traffic flows through the wrong AS or is lost

> **Lab 4:** Read your own table with `ip route`. Watch the live hop-by-hop path with `mtr 8.8.8.8`, see at which hop latency jumps.

---

## ✅ Checkpoint Quiz 2 (Phases 3–4)

L2 (ARP/switch) + L3 (routing/gateway). Half of the "how does data move forward" question is completed here.

---

## Phase 5 — Transport Layer: Reliability and Flow

The goal of this phase: how **reliable** transmission is built while packets can be lost. This phase is the source of most real-world packet-loss problems.

### Topics

**5.1 TCP vs UDP**
- TCP: reliable, ordered, connection-oriented `[concept]`
- UDP: fast, unguaranteed, connectionless `[concept]`
- Which one where (video/DNS vs web/file) `[concept]`

**5.2 Three-way handshake**
- SYN → SYN-ACK → ACK `[mechanism]`
- Why data can't be sent **before** the connection is established `[mechanism]`
- **When it breaks:** SYN goes out but no SYN-ACK comes back → is it the firewall, the route, or the service being down

**5.3 Sequence, ACK, and Retransmission**
- Every byte is numbered, confirmed with an ACK `[mechanism]`
- How a lost packet is detected and resent `[mechanism]`

**5.4 Flow control (window)**
- The receiver saying "send slower, my buffer is filling up" `[mechanism]`
- The sliding window logic `[concept]`

**5.5 Congestion control**
- The sender throttling its speed when the network congests `[mechanism]`
- Slow start, congestion avoidance (at concept level) `[concept]`
- **When it breaks:** high latency + packet loss → throughput collapses (why)

**5.6 Connection teardown**
- Graceful close with FIN/ACK, abrupt cut with RST `[concept]`
- Why the TIME_WAIT state exists `[skip]`

**5.7 MTU, MSS, and Fragmentation ← the heart of packet loss**
- MTU (the max size a frame can carry, typically 1500) `[mechanism]`
- MSS (the segment payload size) `[concept]`
- Fragmentation: splitting a large packet `[mechanism]`
- **MTU black hole:** DF (Don't Fragment) set + a smaller MTU in between → the packet dies silently `[mechanism]`
- **Cloud connection:** the "SSH connects but hangs" classic behind a VPN/tunnel
- **When it breaks:** the sneakiest packet loss comes from here — large packets drop, small ones pass

> **Lab 5:** Capture a real handshake with `sudo tcpdump -n port 443`. See the cwnd/rtt values of an open TCP connection with `ss -ti`. Test your MTU by hand with `ping -M do -s 1472 8.8.8.8`.

---

## Phase 6 — From Name to Address: DNS

The goal of this phase: how `google.com` turns into an IP. The industry joke isn't for nothing: "It's always DNS." In the cloud, Route 53, load balancer, service discovery — they all rest on this.

### Topics

**6.1 DNS hierarchy**
- Root → TLD (.com) → authoritative server chain `[mechanism]`
- Why it's distributed, why there's no single center `[concept]`

**6.2 The resolution chain**
- The recursive resolver's job `[mechanism]`
- Iterative queries: at each step "ask the next one" `[mechanism]`
- **When it breaks:** the resolver is unreachable → everything looks like "no internet" but ping by IP works

**6.3 Record types**
- A, AAAA, CNAME, MX, TXT, NS `[concept]`
- The CNAME chain and its traps `[concept]`

**6.4 Caching and TTL**
- A cache at every level, considered fresh for the TTL duration `[mechanism]`
- **When it breaks:** you changed DNS but traffic still goes to the old IP because of an old cache
- **Cloud connection:** how the Route 53 TTL setting affects deployment downtime

> **Lab 6:** See the whole chain from root to authoritative with `dig google.com +trace`. Read the TTL value in the `dig` output.

---

## Phase 7 — NAT and the Real World

The goal of this phase: how dozens of devices with private IPs get out to the internet through a single public IP. Neither your home network nor your cloud network works without it.

### Topics

**7.1 NAT logic**
- Private source IP → public IP translation `[mechanism]`
- The problem of mapping returning traffic to the correct device `[mechanism]`

**7.2 PAT / Port forwarding**
- Port-based multiplexing (this is what most homes have) `[mechanism]`
- Port forwarding for inbound access from outside `[concept]`
- **When it breaks:** a service behind NAT can't be reached from outside (why)

**7.3 Cloud NAT**
- **Cloud connection:** the NAT Gateway when an EC2 in a private subnet goes out to the internet
- The IGW vs NAT Gateway difference (public exit vs outbound-only traffic) `[concept]`

**7.4 VPN and tunneling (awareness)**
- Tunneling: wrapping one packet inside another and passing it through the public internet `[concept]`
- IPsec (encrypted tunnel protocol): securely connecting two private networks over the internet `[concept]`
- GRE (general-purpose, unencrypted tunnel) and VXLAN (an overlay carrying L2 over L3) — name awareness only `[skip]`
- **Cloud connection:** Site-to-Site VPN (IPsec) connects your on-prem network to a VPC; it learns routes with Phase 4.6 BGP
- **When it breaks:** adding a tunnel header grows the packet → the Phase 5.7 MTU black hole blows up exactly here (an MSS clamp is needed in the tunnel)

**7.5 IPv6 and the NAT-free world (cloud)**
- Address abundance in IPv6 → NAT is mostly unnecessary, every device can get a public address `[concept]`
- Dual-stack (speaking IPv4 + IPv6 at the same time): the transition-period model `[concept]`
- **Cloud connection:** a dual-stack VPC; for outbound-only IPv6, the egress-only Internet Gateway (the IPv6 counterpart of the NAT Gateway)
- **When it breaks:** in dual-stack, if the application picks the wrong family (v4/v6) → "works on one protocol, hangs on the other"

> **Lab 7:** See how your machine appears to the outside (public IP) with `curl ifconfig.me`, compare it with your own `ip addr` private IP.

---

## Phase 8 — Application Layer: HTTP + TLS

The goal of this phase: the topmost layer. If you've studied SSL/TLS content before (handshake, CA verification, reverse proxy), you connect it here; if not, we build it here.

### Topics

**8.1 HTTP request/response**
- The method, path, header, body structure `[mechanism]`
- The GET/POST/PUT/DELETE/PATCH logic `[concept]`

**8.2 Status codes**
- The meaning of each group: 2xx/3xx/4xx/5xx `[concept]`
- **When it breaks:** what the 502 vs 504 difference tells you (is the backend down, or a timeout)

**8.3 HTTP/1.1 vs 2 vs 3**
- Head-of-line blocking, multiplexing, HTTP/3 moving to QUIC/UDP `[skip]`

**8.4 TLS handshake**
- Certificate verification, key exchange (at concept level) `[concept]`
- **Cloud connection:** where TLS termination happens on an ALB

**8.5 Reverse proxy, CDN, and anycast**
- Reverse proxy (an intermediary sitting in front of the server): receives requests, hides the backend behind it `[concept]`
- CDN (content delivery network): serving content from the edge server nearest to the user `[concept]`
- Anycast (announcing one IP from many locations): directs traffic to the nearest copy `[concept]`
- That anycast rests on Phase 4.6 BGP; DNS root servers and CDNs work this way `[skip]`
- **Cloud connection:** CloudFront (CDN + edge), ALB (reverse proxy), Route 53 latency/geo routing
- **When it breaks:** a wrong edge/cache → "old content in one region, new in another" (a sibling problem to Phase 6 TTL)

---

## Phase 9 — Firewall, Filtering, and Security

The goal of this phase: **why a packet is deliberately dropped**. If you've configured a host firewall before (like UFW), here you carry its logic to the cloud; if not, we build the foundation here.

### Topics

**9.1 Stateful vs Stateless firewall**
- Stateful: remembers the connection state, automatically allows returning traffic `[mechanism]`
- Stateless: evaluates each packet one by one, by rule `[mechanism]`

**9.2 Packet filtering**
- The 5-tuple (source IP, destination IP, source port, destination port, protocol) `[concept]`
- Allow/deny order and priority `[concept]`
- **When it breaks:** "ping works but the application won't connect" → a missing port rule

**9.3 Cloud filtering**
- **Cloud connection:** Security Group (stateful) vs NACL (stateless) — connects directly to your TCP state knowledge (Phase 5)
- In an SG you don't need to write a rule for returning traffic, in a NACL you do — **why**

---

## ✅ Checkpoint Quiz 3 (Phases 5–9)

Transport + DNS + NAT + firewall. All the pieces of the "why did the packet drop" question are here.

---

## Phase 10 — Troubleshooting: Instinctive Fault Hunting

The goal of this phase: to turn all the previous "when it breaks" notes into **a single systematic reflex**. The real cloud-engineer difference shows up here.

### Topics

**10.1 Layer-by-layer debugging methodology**
- Bottom-up: is there a link → is there an IP → gateway/route → DNS → port → application `[mechanism]`
- The discipline of not skipping a layer before the previous one is verified `[concept]`

**10.2 Tool mastery (which layer each tool looks at)**
- `ip` / `ip route` → L2/L3 config `[application]`
- `ping` → L3 reachability `[application]`
- `arp`/`ip neigh` → L2 resolution `[application]`
- `mtr` → hop-by-hop L3 path + loss `[application]`
- `dig` → DNS resolution `[application]`
- `ss` → local port/socket state `[application]`
- `tcpdump` → the raw packet truth (the ultimate arbiter) `[application]`
- `curl -v` → L7 end-to-end `[application]`

**10.3 The answer map for the three instinct questions**
- "How does data move forward" → the combined mental film of Phases 0+1+4+5
- "Why did the packet drop" → the MTU / firewall / congestion / route / TTL decision tree
- "Why is the router acting up" → the routing table / ARP / NAT / gateway checklist

**10.4 Packet hunting in the cloud: VPC Flow Logs**
- What tcpdump is locally, Flow Logs are in the cloud: a record of which traffic was ACCEPTed/REJECTed `[application]`
- The 5-tuple + ACCEPT/REJECT field in the record → verifies the Phase 9 firewall decision `[application]`
- **Cloud connection:** Flow Logs show the REJECT; whether it's the SG or the NACL, you infer from the traffic direction + statefulness logic (Phase 9)
- **When it breaks:** "the connection times out but it's not clear why" → a Flow Logs REJECT line shows it

> **Capstone Lab:** Set up a deliberate failure (wrong gateway, closed port, wrong DNS), then find it in 5 minutes using only the methodology. This is the real test of instinct.

---

## Phase 11 — Bridge to the Cloud (AWS Mapping)

The goal of this phase: to seat every fundamental onto its AWS counterpart. A direct exit toward the SAA goal.

### Topics

- **VPC** = your own isolated network (Phase 2 CIDR) `[application]`
- **Subnet** (public/private) = Phase 2 + Phase 7 `[application]`
- **Route Table** = the Phase 4 routing table `[application]`
- **Internet Gateway / NAT Gateway** = Phase 7 `[application]`
- **Security Group vs NACL** = Phase 9 stateful/stateless `[application]`
- **Route 53** = Phase 6 DNS `[application]`
- **ELB: ALB (L7) vs NLB (L4)** = Phase 5 + Phase 8 `[application]`
- **VPC Peering / Transit Gateway** = Phase 2 CIDR overlap `[concept]`
- **Site-to-Site VPN / VPN Gateway** = Phase 7.4 tunneling + Phase 4.6 BGP `[concept]`
- **CloudFront** = Phase 8.5 CDN + edge + anycast `[concept]`
- **Dual-stack VPC / Egress-only IGW** = Phase 1.5 + Phase 7.5 IPv6 `[concept]`
- **VPC Endpoint / PrivateLink** = private access to an AWS service without going out to the internet, an alternative to Phase 7 NAT `[concept]`

> **Final output:** You should be able to draw an empty VPC from scratch (subnets, route table, IGW, NAT, SG) and explain why each part is there, grounded in the fundamentals.

---

## Overall structure

| Section | Phases | Focus |
|---|---|---|
| Foundation | 0–2 | Model, addressing, subnet |
| Local + routing | 3–4 | L2/L3, "how does data move forward" |
| Reliability + perimeter | 5–9 | Transport, DNS, NAT, firewall — "why did the packet drop" |
| Mastery | 10–11 | Troubleshooting instinct + cloud bridge |

**Standing rules (same as the physical-layer map):** English; technical terms get a short definition in parentheses on first use; one topic, one question; no direct answer; no topic skipping; a guiding question on a wrong answer.

---

*Prepared by: Denis Ergöçmen, June 2026 — neutralized version for general use*
