# Appendix B — Concept and File Map

> **Navigation:** [◀ Appendix A — Command Glossary](Appendix_A_Command_Glossary.md) · **Appendix B** · [Appendix C — "When It Breaks" Quick Reference ▶](Appendix_C_When_It_Breaks_Quick_Reference.md)

---

> **This appendix is the quick answer to "what was this concept, where does this setting live".** Next to
> every row is written which phase it comes from. In networking everything lives in a **layer** (Phase 0) —
> understanding a problem starts with looking at the right layer and the right table.

**Contents**
- [B.1 Layers and PDUs (Phase 0)](#b1-layers-and-pdus-phase-0)
- [B.2 Address types and special blocks (Phases 1, 2)](#b2-address-types-and-special-blocks-phases-1-2)
- [B.3 The prefix length table (Phase 2)](#b3-the-prefix-length-table-phase-2)
- [B.4 Commonly used ports (Phases 1, 5, 8)](#b4-commonly-used-ports-phases-1-5-8)
- [B.5 The network files on the machine (Phases 1, 4, 6)](#b5-the-network-files-on-the-machine-phases-1-4-6)
- [B.6 The tables on the machine — which question in which table (Phases 3, 4, 5, 7)](#b6-the-tables-on-the-machine--which-question-in-which-table-phases-3-4-5-7)
- [B.7 DNS record types (Phase 6)](#b7-dns-record-types-phase-6)
- [B.8 HTTP status codes (Phase 8)](#b8-http-status-codes-phase-8)
- [B.9 ICMP message types (Phases 4, 5, 9)](#b9-icmp-message-types-phases-4-5-9)
- [B.10 Fundamental concept → the AWS counterpart (Phase 11)](#b10-fundamental-concept--the-aws-counterpart-phase-11)

---

## B.1 Layers and PDUs (Phase 0)

*(Phase 0.2 — each layer puts the previous one in an envelope)*

| Layer | Address | PDU name | Device | Example protocol | Phase |
|---|---|---|---|---|---|
| L7 Application | URL / name | Message | — | HTTP, DNS, SSH | 6, 8 |
| L4 Transport | Port | Segment (TCP) / Datagram (UDP) | Load balancer (L4) | TCP, UDP | 5 |
| L3 Network | IP address | Packet | Router | IP, ICMP | 1, 4 |
| L2 Data link | MAC address | Frame | Switch | Ethernet, ARP | 1, 3 |
| L1 Physical | — | Bit | Cable, hub | — | 0 |

> **The rule (Phase 0.3):** Encapsulation **adds a header** on the way down and **strips one** on the way up.
> Everything that happens to a packet happens in one of these five rows. The first question in diagnosis:
> *"Which layer does this symptom belong to?"*

## B.2 Address types and special blocks (Phases 1, 2)

| Block / Address | What it means | Phase |
|---|---|---|
| `10.0.0.0/8` | Private (RFC 1918) — a large block, the most common in the cloud | 1.3 |
| `172.16.0.0/12` | Private (RFC 1918) — Docker's defaults are here | 1.3 |
| `192.168.0.0/16` | Private (RFC 1918) — home/office networks | 1.3 |
| `127.0.0.0/8` | Loopback — the machine itself (never leaves it) | 1.2 |
| `169.254.0.0/16` | APIPA / link-local — it means **DHCP failed** | 1.5 |
| `169.254.169.254` | The cloud instance metadata service | 11.1 |
| `0.0.0.0/0` | "Everywhere" — the default route or an "open to all" rule | 4.2, 9.2 |
| `0.0.0.0` (as a bind address) | "Listen on all interfaces" | 1.4 |
| `255.255.255.255` | Limited broadcast (DHCP Discover) | 1.5, 3.3 |
| `224.0.0.0/4` | Multicast | 3.3 |
| `fe80::/10` | IPv6 link-local | 7.5 |
| `::1` | IPv6 loopback | 7.5 |

> **The reflex (Phase 1.5):** If you see `169.254.x.x` on a machine, DHCP did not answer — the IP
> configuration **never happened**, it is not wrong. The place to look is the DHCP server or the L2
> connection.

## B.3 The prefix length table (Phase 2)

*(Phase 2.2 — how many bits the mask gives to the network)*

| Prefix | Mask | Total addresses | Usable in AWS | Typical use |
|---|---|---|---|---|
| `/16` | 255.255.0.0 | 65,536 | 65,531 | A whole VPC |
| `/20` | 255.255.240.0 | 4,096 | 4,091 | A large subnet |
| `/22` | 255.255.252.0 | 1,024 | 1,019 | A medium subnet |
| `/24` | 255.255.255.0 | 256 | 251 | A standard subnet |
| `/26` | 255.255.255.192 | 64 | 59 | A small subnet |
| `/28` | 255.255.255.240 | 16 | 11 | The smallest AWS subnet |
| `/32` | 255.255.255.255 | 1 | — | A single host (a rule/route target) |

> **AWS reserves 5 addresses (Phase 2.3.2):** `.0` the network address · `.1` the router · `.2` the DNS
> resolver · `.3` future use · **the last address** broadcast. You cannot go below a `/28`.

## B.4 Commonly used ports (Phases 1, 5, 8)

| Port | Protocol | Service | Note | Phase |
|---|---|---|---|---|
| 22 | TCP | SSH | Management access — never open it to `0.0.0.0/0` | 1.4, 9.5 |
| 53 | UDP (+TCP) | DNS | Large answers fall back to TCP | 6.1 |
| 67/68 | UDP | DHCP | Broadcast based (DORA) | 1.5 |
| 80 | TCP | HTTP | Usually redirected to 443 | 8.1 |
| 123 | UDP | NTP | Clock synchronisation (critical for TLS) | 8.4 |
| 443 | TCP (+UDP/QUIC) | HTTPS | HTTP/3 runs over UDP | 8.3, 8.4 |
| 3306 | TCP | MySQL | Opened only to the application SG | 11.5 |
| 5432 | TCP | PostgreSQL | Opened only to the application SG | 11.5 |
| 6379 | TCP | Redis | Its default has no authentication — do not expose it | 9.5 |
| 1024–65535 | TCP/UDP | The ephemeral range | The client's temporary source port | 1.4, 5.1 |

> **The critical distinction (Phase 9.4.1):** In a NACL's **outbound** rule it is not the service's port that
> is written but this **ephemeral range** — because the answer goes to the client's temporary port and a NACL
> is stateless.

## B.5 The network files on the machine (Phases 1, 4, 6)

| File | What it does | Phase |
|---|---|---|
| `/etc/hosts` | Local name→IP mapping; consulted **before** DNS | 6.2 |
| `/etc/resolv.conf` | Which resolver is asked (the `nameserver` lines) | 6.2 |
| `/etc/nsswitch.conf` | The **order** of name resolution (`files dns`) | 6.2 |
| `/etc/netplan/*.yaml` | Permanent network configuration (Ubuntu) | 1.2 |
| `/etc/systemd/network/*.network` | systemd-networkd configuration | 1.2 |
| `/etc/ssh/sshd_config` | The SSH listening address and port (`ListenAddress`) | 1.4 |
| `/proc/sys/net/ipv4/ip_forward` | Is the machine forwarding packets (router behaviour) | 4.1 |
| `/proc/sys/net/ipv4/tcp_keepalive_time` | The keepalive interval (for the NAT table) | 5.6, 7.1 |
| `/proc/sys/net/netfilter/nf_conntrack_max` | The connection tracking table's limit | 7.2, 9.1 |
| `/sys/class/net/<interface>/mtu` | The interface's MTU value | 5.7 |
| `/sys/class/net/<interface>/address` | The interface's MAC address | 1.1 |

> **Note (Phase 6.2.2):** `/etc/hosts` is read **before** DNS. The classic cause of "the site works for
> everyone else but not for me" is a line forgotten here. If `getent hosts <name>` and `dig +short <name>`
> give different results, this is the suspect.

## B.6 The tables on the machine — which question in which table (Phases 3, 4, 5, 7)

| Question | Table | Command | Phase |
|---|---|---|---|
| "What is this IP's MAC?" | The ARP table | `ip neigh` | 3.1 |
| "By which path do I get to this destination?" | The routing table | `ip route get <ip>` | 4.2 |
| "Which connections are open right now?" | The socket table | `ss -tan` | 5.2 |
| "Which ports are being listened on?" | The listening sockets | `ss -tulpn` | 1.4 |
| "Which translation did NAT make?" | The conntrack table | `conntrack -L` | 7.1 |
| "Which rule did the firewall apply?" | The rule chain + counters | `iptables -L -n -v` | 9.2 |
| "Where did the switch learn this MAC from?" | The MAC (FDB) table | `bridge fdb show` | 3.2 |

> **The diagnostic logic (Phase 10.1):** Every fault shows up as a **missing or stale row** in one of these
> tables. Instead of guessing, open the relevant table and look.

## B.7 DNS record types (Phase 6)

| Record | What it does | Watch out | Phase |
|---|---|---|---|
| `A` | Name → IPv4 | The most basic record | 6.3 |
| `AAAA` | Name → IPv6 | On dual-stack both exist together | 6.3, 7.5 |
| `CNAME` | Name → another name | **Cannot be used at the apex (the root domain)** | 6.3 |
| `ALIAS` / Route 53 `Alias` | CNAME-like behaviour at the apex | An AWS-specific solution | 11.4 |
| `MX` | The mail server | Carries a priority value | 6.3 |
| `NS` | The authoritative servers for this zone | A link in the delegation chain | 6.1 |
| `TXT` | Free text | SPF/DKIM, domain verification | 6.3 |
| `PTR` | IP → name (reverse) | Important for mail servers | 6.3 |
| `SOA` | The zone's authority information | The default negative TTL is here | 6.1 |
| `SRV` | A service's location (host + port) | Used in service discovery | 6.3 |

> **A dangling CNAME (Phase 6.3.2):** If a CNAME's target is deleted but the record stays, someone else can
> take that name — **subdomain takeover.** When you delete the resource, delete the DNS record too.

## B.8 HTTP status codes (Phase 8)

| Code | What it means | Where to look | Phase |
|---|---|---|---|
| 200 | Success | — | 8.2 |
| 301 / 302 | Permanent / temporary redirect | A 301 is **cached** in the browser, be careful | 8.2 |
| 400 | Bad request | The client | 8.2 |
| 401 / 403 | Not authenticated / not authorised | 401 "who are you", 403 "you are not allowed" | 8.2 |
| 404 | Not found | The path or the routing rule | 8.2 |
| 429 | Too many requests | A rate limit | 8.5 |
| 500 | Internal server error | The application code | 8.2 |
| **502** | **Bad Gateway — the proxy could not connect to the backend** | Is the backend up, is the SG open | 8.2, 5.2 |
| **504** | **Gateway Timeout — it connected, no answer came** | The backend is slow / stuck | 8.2, 5.2 |
| 503 | Service unavailable | No healthy target (LB) | 8.2, 11.6 |

## B.9 ICMP message types (Phases 4, 5, 9)

| Type/Code | Message | What it tells you | Phase |
|---|---|---|---|
| 8 / 0 | Echo Request / Reply | `ping` itself | 4.4 |
| 11 code 0 | Time Exceeded | The TTL ran out — `traceroute` uses this | 4.4 |
| 3 code 0/1 | Network/Host Unreachable | There is no route or the destination is unreachable | 4.4 |
| 3 code 3 | Port Unreachable | The UDP port is closed | 5.1 |
| **3 code 4** | **Fragmentation Needed** | **The MTU is small — never block it** | 5.7, 9.3 |
| 5 | Redirect | A suggestion of a better route (usually turned off) | 4.2 |

> **The golden rule (Phase 9.3):** You may restrict ping, but **never block Fragmentation Needed** — if you
> do, an MTU black hole forms and the symptom looks nothing like a firewall ("it connects but freezes").

## B.10 Fundamental concept → the AWS counterpart (Phase 11)

| Fundamental concept | The AWS counterpart | Phase |
|---|---|---|
| An address space / a private network | VPC (a CIDR block) | 11.1 |
| Subnet | Subnet (attached to one AZ) | 11.2 |
| The routing table | Route Table (associated with a subnet) | 11.2 |
| The default gateway | Internet Gateway / NAT Gateway | 11.3 |
| NAT / PAT | NAT Gateway | 11.3 |
| A stateful firewall | Security Group | 11.5 |
| A stateless filter | Network ACL | 11.5 |
| A DNS resolver | The VPC `.2` resolver / Route 53 Resolver | 11.4 |
| Authoritative DNS | Route 53 Hosted Zone | 11.4 |
| An L4 load balancer | NLB | 11.6 |
| An L7 load balancer | ALB | 11.6 |
| CDN / anycast | CloudFront | 11.6 |
| A connection between private networks | VPC Peering / Transit Gateway | 11.7 |
| An encrypted tunnel | Site-to-Site VPN | 11.7 |
| Private service access | VPC Endpoint / PrivateLink | 11.3 |
| Packet recording | VPC Flow Logs | 10.4 |

> **In one sentence (Phase 11.8):** The cloud did not invent a new network; it **put the network you learned
> in this book behind an API.** Under every box you see in the console there is a fundamental concept listed
> in this appendix.

---

> **Navigation:** [◀ Appendix A — Command Glossary](Appendix_A_Command_Glossary.md) · **Appendix B** · [Appendix C — "When It Breaks" Quick Reference ▶](Appendix_C_When_It_Breaks_Quick_Reference.md)
