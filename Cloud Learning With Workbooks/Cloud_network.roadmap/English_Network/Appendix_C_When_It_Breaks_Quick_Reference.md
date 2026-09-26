# Appendix C — "When It Breaks" Quick Reference

> **Navigation:** [◀ Appendix B — Concept and File Map](Appendix_B_Concept_File_Map.md) · **Appendix C** · [README ▶](README_en.md)

---

> **This appendix brings all of the workbook's "When this phase breaks" tables together on a single hunting
> page.** When you see a symptom, look here: the likely cause, the command you verify it with, the phase you
> go back to. This is the pocket form of Phase 10's diagnostic reflex.

**Contents**
- [C.1 First: the diagnostic reflex (Phase 10)](#c1-first-the-diagnostic-reflex-phase-10)
- [C.2 Two symptoms, two worlds: timeout vs refused](#c2-two-symptoms-two-worlds-timeout-vs-refused)
- [C.3 Addressing and DHCP (Phases 1, 2)](#c3-addressing-and-dhcp-phases-1-2)
- [C.4 The local network — L2 (Phase 3)](#c4-the-local-network--l2-phase-3)
- [C.5 Routing — L3 (Phase 4)](#c5-routing--l3-phase-4)
- [C.6 Transport and MTU (Phase 5)](#c6-transport-and-mtu-phase-5)
- [C.7 DNS (Phase 6)](#c7-dns-phase-6)
- [C.8 NAT and tunnels (Phase 7)](#c8-nat-and-tunnels-phase-7)
- [C.9 HTTP and TLS (Phase 8)](#c9-http-and-tls-phase-8)
- [C.10 Firewalls and filtering (Phase 9)](#c10-firewalls-and-filtering-phase-9)
- [C.11 Cloud-specific (Phase 11)](#c11-cloud-specific-phase-11)

---

## C.1 First: the diagnostic reflex (Phase 10)

Do not panic, do not open rules at random. Eliminate from the bottom up:

| Order | Layer | The first command |
|---|---|---|
| 1 | Do I have an address of my own | `ip addr` |
| 2 | Is the local network working | `ping <gateway>` · `ip neigh` |
| 3 | Can I get out | `ip route` · `ping 1.1.1.1` |
| 4 | Do names resolve | `dig <domain>` |
| 5 | Is the port open | `nc -zv <host> <port>` |
| 6 | Is the application answering | `curl -v <url>` |

> **The golden rule:** **Eliminate** one layer at each step and only then climb up. Saying "it worked and then
> it broke" is not a diagnosis; show which step it breaks at. **The shortcut (Phase 10.1.2):** if you are
> short of time, `ping 1.1.1.1` and `dig <domain>` first — those two eliminate half the possibilities.

## C.2 Two symptoms, two worlds: timeout vs refused

*(Phase 9.2.3 — the single most useful distinction in this book)*

| Symptom | What happened | Where to look | Phase |
|---|---|---|---|
| **Timeout** (waits, then an error) | The packet was **dropped silently** (DROP) | The firewall (SG/NACL/OS), routing, NAT | 9.2, 4.2 |
| **Connection refused** (instantly) | An **RST** came — the packet arrived, the service is not listening | The application, the bind address, the port | 5.2, 1.4 |
| **No route to host** (instantly) | Not even a local routing decision could be made | `ip route`, the gateway definition | 4.2 |
| **Name or service not known** | The name did not resolve, the connection was never attempted | DNS, `/etc/resolv.conf` | 6.2 |

> **In one sentence:** *Timeout = most likely a firewall. Refused = not a firewall, the service is not
> listening.* This distinction lets you find the right team on the first try.

## C.3 Addressing and DHCP (Phases 1, 2)

| Symptom | Likely cause | Verify | Phase |
|---|---|---|---|
| The IP is `169.254.x.x` | DHCP did not answer (APIPA) | `ip addr`, `journalctl -u systemd-networkd` | 1.5 |
| There is no IP at all | The interface is down or a cable/ENI problem | `ip link`, `ip -br addr` | 1.2 |
| There is an IP but it cannot reach the network | The wrong mask → it thinks a neighbour is "remote" | `ip addr` (look at the prefix), `ip route get` | 2.2 |
| The same IP on two machines | A static IP clashed with the DHCP pool | `arping -D`, `ip neigh` | 1.3, 3.1 |
| The subnet is full, no new machine starts | The address space was planned too small | `ipcalc`, the number of IPs in use | 2.3 |
| A new subnet cannot be added | The VPC CIDR is insufficient (it cannot be narrowed later) | `aws ec2 describe-vpcs` | 2.4, 11.1 |
| Unreachable from outside, the port is open | The application is bound to `127.0.0.1` | `ss -tulpn \| grep <port>` | 1.4 |

## C.4 The local network — L2 (Phase 3)

| Symptom | Likely cause | Verify | Phase |
|---|---|---|---|
| A machine in the same subnet is unreachable | ARP is not resolving (L2 is broken) | `ip neigh` → `INCOMPLETE`/`FAILED` | 3.1 |
| The IP changed and access was cut off | A stale ARP entry | `ip neigh flush all`, try again | 3.1 |
| The network is slow, everyone is affected | A broadcast storm / a loop | `tcpdump -ni any arp`, the switch logs | 3.3 |
| Some machines are visible, some are not | VLAN separation (the same switch, different segments) | The VLAN configuration, `ip -d link` | 3.4 |
| There is access but it is not two-way | A wrong/clashing MAC or port security | `bridge fdb show`, the interface counters | 3.2 |

## C.5 Routing — L3 (Phase 4)

| Symptom | Likely cause | Verify | Phase |
|---|---|---|---|
| The local network is fine, nothing gets out | The default route is missing | `ip route` → is there a `default` row | 4.2 |
| It goes but no answer comes back | **There is no return route** (asymmetric) | On the far side, `ip route get <your-ip>` | 4.2 |
| It goes out the wrong way | A more specific route won (LPM) | `ip route get <destination>` | 4.3 |
| `* * *` in the middle of `traceroute` | The hop does not answer ICMP — **not a fault** | `mtr -rwc 100`, look at the last hop | 4.4, 10.2 |
| The packet does not arrive, a TTL error | A routing loop | `traceroute` → repeating hops | 4.4 |
| One region is unreachable | A routing/BGP problem on the far side | `mtr`, the provider's status page | 4.5 |

## C.6 Transport and MTU (Phase 5)

| Symptom | Likely cause | Verify | Phase |
|---|---|---|---|
| Stuck in `SYN-SENT` | No answer is coming at all → a DROP | `ss -tan state syn-sent`, `tcpdump` | 5.2, 9.2 |
| It connects but **freezes** | An MTU black hole (large packets are disappearing) | `ping -M do -s 1472 <destination>` | 5.7 |
| Small requests work, large ones do not | The same cause: MTU / MSS | `tracepath <destination>` | 5.7 |
| An idle session dies | The row in the NAT/firewall table was deleted | The keepalive setting, `conntrack -L` | 5.6, 7.1 |
| `CLOSE-WAIT` is piling up | The application is not closing its sockets (a code bug) | `ss -tan state close-wait` | 5.6 |
| The transfer is slow but lossless | The window/RTT limit (not the bandwidth) | `ss -i` (rtt, cwnd) | 5.4, 5.5 |
| Data is lost in UDP and nobody reports it | There is no retransmission in UDP — that is normal | The application layer is responsible | 5.1 |

## C.7 DNS (Phase 6)

| Symptom | Likely cause | Verify | Phase |
|---|---|---|---|
| No name resolves at all | The resolver is unreachable | `cat /etc/resolv.conf`, `dig @1.1.1.1` | 6.2 |
| `dig @1.1.1.1` works, `dig` does not | The problem is **in your resolver** | `resolvectl status` | 6.2 |
| You migrated but it still goes to the old server | The old answer is in the cache for the length of the TTL | `dig` → the TTL, `resolvectl flush-caches` | 6.4 |
| It does not work for me, it works for everyone else | A line forgotten in `/etc/hosts` | `getent hosts <name>` vs `dig +short` | 6.2 |
| A CNAME cannot be added at the apex domain | The standard forbids it | Use a Route 53 **Alias** | 6.3, 11.4 |
| The name resolves but the application goes to the old IP | The application keeps its own cache | `ss -tan` → the real peer IPs | 6.4 |
| The domain suddenly disappeared | The NS delegation or the registration period | `dig +trace`, `dig NS <domain>` | 6.1 |

## C.8 NAT and tunnels (Phase 7)

| Symptom | Likely cause | Verify | Phase |
|---|---|---|---|
| It gets out from inside, nothing gets in from outside | The nature of NAT — the mapping starts only from inside | `conntrack -L`, port forwarding | 7.1 |
| **Partial** connection errors under load | Port exhaustion / the conntrack table is full | `conntrack -C` vs `nf_conntrack_max` | 7.2, 9.1 |
| A private machine cannot get updates | There is no NAT Gateway or route | `ip route`, the route table's `0.0.0.0/0` | 7.3, 11.3 |
| The VPN is up but no traffic flows | The tunnel is up, **the route is not defined** | `ip route`, the tunnel interface | 7.4 |
| Large packets are dropped over the VPN | The tunnel header lowered the MTU | `ping -M do`, MSS clamping | 7.4, 5.7 |
| Two networks merged, the IPs clash | The same RFC 1918 block on both sides | The CIDR plan, re-addressing | 7.3, 11.7 |
| It works on IPv6 but not IPv4 (or the reverse) | Only one side of the dual stack is configured | Separate them with `curl -4` / `curl -6` | 7.5 |

## C.9 HTTP and TLS (Phase 8)

| Symptom | Likely cause | Verify | Phase |
|---|---|---|---|
| **502** Bad Gateway | The proxy could not connect to the backend | Is the backend up, the SG rule | 8.2, 5.2 |
| **504** Gateway Timeout | It connected, the answer did not come in time | The backend is slow / stuck, the timeout setting | 8.2 |
| 503 Service Unavailable | There is no healthy target | The target group's health checks | 8.2, 11.6 |
| A certificate error | Expired / does not cover the domain / SNI | `openssl s_client -servername ...` | 8.4 |
| It works in the browser, not with `curl` | SNI, the certificate chain or the Host header | `curl -v`, `--resolve` | 8.4 |
| The clock is wrong → TLS is rejected | The certificate's validity window is shifted | `timedatectl`, NTP | 8.4 |
| I made a change but the old content comes back | The CDN or the browser cache | `curl -I` → the cache headers, invalidation | 8.5 |
| After a 301 it goes to the wrong address | The permanent redirect is cached in the browser | Try it in a private window | 8.2 |

## C.10 Firewalls and filtering (Phase 9)

| Symptom | Likely cause | Verify | Phase |
|---|---|---|---|
| A timeout, no answer at all | There is a rule dropping it | `iptables -L -n -v` (the counters), `tcpdump` | 9.2 |
| The SG is open but it is unreachable | The **outbound ephemeral** rule is missing in the NACL | The NACL rules, a REJECT in Flow Logs | 9.4 |
| I added a rule, it has no effect | A more general rule above it is catching the traffic | The rule order, the `pkts` counters | 9.2 |
| Ping does not work but the service does | ICMP is closed — **not a fault** | Verify with a port test (`nc -zv`) | 9.3 |
| "It connects but freezes" | ICMP Frag Needed is blocked | `ping -M do`, the ICMP rules | 9.3, 5.7 |
| I changed a rule remotely and locked myself out | A rule that cut off your own access | Console/Session Manager → `iptables-restore` | 9.5 |
| Everyone can reach everything | A flat network: the SGs do not reference each other | The SG chain (alb-sg → app-sg → db-sg) | 9.5, 11.5 |

## C.11 Cloud-specific (Phase 11)

| Symptom (AWS) | Likely cause | Verify | Phase |
|---|---|---|---|
| There is a public IP but it is unreachable | There is no IGW route in the route table | `describe-route-tables` → `0.0.0.0/0` | 11.2, 11.3 |
| A private instance cannot get out to the internet | There is no NAT Gateway or the route is missing | The NAT GW's state + the private route table | 11.3 |
| There is a NAT GW but it still cannot get out | The NAT GW was put in a **private** subnet | Is the NAT GW's subnet public | 11.3 |
| Access to S3 is slow/expensive | The traffic goes through the NAT | Add a VPC Endpoint | 11.3 |
| There is peering but a third VPC is unreachable | Peering **is not transitive** | The route tables, Transit Gateway | 11.7 |
| The ALB returns a 502 | The target SG is closed to traffic from the ALB's SG | The target group's health + the SG chain | 11.5, 11.6 |
| The domain does not work at the apex | A CNAME cannot be at the apex | A Route 53 Alias record | 11.4 |
| DNS does not resolve inside the VPC | `enableDnsSupport`/`enableDnsHostnames` is off | The VPC DNS settings, the `.2` resolver | 11.4 |
| The connection drops but there is no error | A REJECT line in Flow Logs | `filter-log-events` → ACCEPT/REJECT | 10.4 |
| Latency rises when you leave the AZ | The subnets are in different AZs | The subnet-to-AZ mapping | 11.2 |

> **The summary of the whole workbook — in one sentence:** Under every symptom there is a networking fact. Do
> not panic; follow the packet's path from the bottom up, find which row is missing in which table, and fix
> the root cause for good. That reflex is the most valuable thing this workbook leaves you with.

---

> **Navigation:** [◀ Appendix B — Concept and File Map](Appendix_B_Concept_File_Map.md) · **Appendix C** · [README ▶](README_en.md)
