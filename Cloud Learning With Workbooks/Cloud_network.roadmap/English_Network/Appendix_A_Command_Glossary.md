# Appendix A — Command Glossary

> **Navigation:** [◀ Phase 11 — The Bridge to the Cloud](Phase_11_Bridge_to_the_Cloud.md) · **Appendix A** · [Appendix B — Concept and File Map ▶](Appendix_B_Concept_File_Map.md)

---

> **This appendix exists not to teach but to remind.** This is the page to keep open after you finish the
> workbook. Next to every command is written which phase it comes from — when a command slips your mind, go
> back there. Risk marks: 🟢 read-only (safe), 🟡 a temporary change, 🔴 a permanent change (be careful).

**Contents**
- [A.1 Addresses and interfaces (Phase 1)](#a1-addresses-and-interfaces-phase-1)
- [A.2 Subnet and CIDR arithmetic (Phase 2)](#a2-subnet-and-cidr-arithmetic-phase-2)
- [A.3 The local network — L2 (Phase 3)](#a3-the-local-network--l2-phase-3)
- [A.4 Routing — L3 (Phase 4)](#a4-routing--l3-phase-4)
- [A.5 Transport and connections (Phase 5)](#a5-transport-and-connections-phase-5)
- [A.6 DNS (Phase 6)](#a6-dns-phase-6)
- [A.7 NAT, tunnels, IPv6 (Phase 7)](#a7-nat-tunnels-ipv6-phase-7)
- [A.8 HTTP and TLS (Phase 8)](#a8-http-and-tls-phase-8)
- [A.9 Firewalls and filtering (Phase 9)](#a9-firewalls-and-filtering-phase-9)
- [A.10 Packet capture and diagnosis (Phase 10)](#a10-packet-capture-and-diagnosis-phase-10)
- [A.11 The cloud side (Phase 11)](#a11-the-cloud-side-phase-11)

---

## A.1 Addresses and interfaces (Phase 1)

| Command | What it does | Risk |
|---|---|---|
| `ip addr` (`ip a`) | The interfaces and the IPs/prefixes on them | 🟢 |
| `ip -br addr` | The same, as a one-line-per-interface summary | 🟢 |
| `ip link` | The L2 state of the interfaces and their MAC addresses | 🟢 |
| `ip link set eth0 up/down` | Bring the interface up/down (temporary) | 🟡 |
| `ip addr add 10.0.0.5/24 dev eth0` | Adds a temporary IP (lost on reboot) | 🟡 |
| `hostname -I` | The machine's IP address(es) | 🟢 |
| `ip -4 addr show eth0` | IPv4 addresses only | 🟢 |
| `cat /sys/class/net/eth0/address` | The interface's MAC address (from the file) | 🟢 |
| `dhclient -v eth0` | Renews the DHCP lease (DORA) | 🟡 |
| `ss -tulpn` | Listening ports + which process (the bind address!) | 🟢 |

## A.2 Subnet and CIDR arithmetic (Phase 2)

| Command | What it does | Risk |
|---|---|---|
| `ipcalc 10.0.4.0/22` | Network/broadcast/usable range arithmetic | 🟢 |
| `sipcalc 10.0.4.0/22` | The same job with more detailed output | 🟢 |
| `ipcalc -s 10.0.0.0/16 256 256` | Splits it into the given sizes (VLSM) | 🟢 |
| `python3 -c "import ipaddress;print(ipaddress.ip_network('10.0.4.0/22'))"` | When there is no calculator | 🟢 |
| `python3 -c "import ipaddress;print(ipaddress.ip_address('10.0.5.7') in ipaddress.ip_network('10.0.4.0/22'))"` | "Is this IP inside this block" | 🟢 |

> **A check in your head (Phase 2.3):** `/24` → 256 addresses, `/25` → 128, `/26` → 64, `/27` → 32, `/28` →
> 16. AWS **reserves 5 addresses** in every subnet (`.0` network, `.1` router, `.2` DNS, `.3` future, `.255`
> broadcast).

## A.3 The local network — L2 (Phase 3)

| Command | What it does | Risk |
|---|---|---|
| `ip neigh` (`arp -n`) | The ARP table: IP → MAC mappings | 🟢 |
| `ip neigh flush all` | Clears the ARP cache (when you suspect a stale entry) | 🟡 |
| `arping -I eth0 10.0.0.1` | A reachability test directly at the L2 level | 🟢 |
| `ip -s link show eth0` | The interface's packet/error counters | 🟢 |
| `bridge fdb show` | The bridge/switch MAC learning table | 🟢 |
| `ip link add link eth0 name eth0.10 type vlan id 10` | Creates a VLAN sub-interface | 🔴 |
| `tcpdump -ni eth0 arp` | Watches live ARP traffic | 🟢 |

## A.4 Routing — L3 (Phase 4)

| Command | What it does | Risk |
|---|---|---|
| `ip route` (`ip r`) | The routing table (the first place to look) | 🟢 |
| `ip route get 8.8.8.8` | **Which row is chosen for this destination** (the LPM decision) | 🟢 |
| `ip route add 10.2.0.0/16 via 10.0.0.1` | Adds a temporary route | 🟡 |
| `ip route add default via 10.0.0.1` | Defines the default gateway | 🟡 |
| `ip route del <network>` | Deletes a route | 🔴 |
| `ping -c4 <destination>` | A reachability test (ICMP Echo) | 🟢 |
| `ping -t 1 <destination>` | Limit the TTL by hand (the logic of traceroute) | 🟢 |
| `traceroute <destination>` | The hops along the path (UDP based) | 🟢 |
| `traceroute -I <destination>` / `-T -p 443` | Tracing with ICMP / TCP (on filtered networks) | 🟢 |
| `cat /proc/sys/net/ipv4/ip_forward` | Is the machine behaving like a router | 🟢 |

## A.5 Transport and connections (Phase 5)

| Command | What it does | Risk |
|---|---|---|
| `ss -tan` | TCP connections and their states (ESTAB, SYN-SENT…) | 🟢 |
| `ss -tulpn` | Listening TCP/UDP ports + the process | 🟢 |
| `ss -tan state syn-sent` | Only the handshakes that are stuck | 🟢 |
| `nc -zv <host> 443` | Whether a single port is open (connect and close) | 🟢 |
| `nc -zvu <host> 53` | Tries a UDP port (no answer ≠ closed) | 🟢 |
| `nc -l 8080` | Listens on a port for testing | 🟡 |
| `ping -M do -s 1472 <destination>` | An MTU test (fragmentation forbidden) | 🟢 |
| `ip link show eth0 \| grep mtu` | The interface's MTU value | 🟢 |
| `tracepath <destination>` | Shows the MTU change along the path | 🟢 |
| `ss -i` | Detail inside a connection (rtt, cwnd, retransmits) | 🟢 |

> **Reading the states (Phase 5.2/5.6):** `SYN-SENT` = no answer is coming (a DROP is suspected) · `ESTAB` =
> healthy · `TIME-WAIT` = the remains of a normal close · `CLOSE-WAIT` **piling up** means the application is
> not closing its sockets.

## A.6 DNS (Phase 6)

| Command | What it does | Risk |
|---|---|---|
| `dig example.com` | The full DNS answer (including TTL, ANSWER, SERVER) | 🟢 |
| `dig +short example.com` | Just the result | 🟢 |
| `dig @1.1.1.1 example.com` | Asks **a specific resolver** (skips the cache) | 🟢 |
| `dig example.com MX` / `NS` / `TXT` / `CNAME` | A query by record type | 🟢 |
| `dig +trace example.com` | The whole resolution chain from the root down | 🟢 |
| `dig -x 93.184.216.34` | Reverse resolution (PTR) | 🟢 |
| `host` / `nslookup` | A quick query (does not give as much detail as dig) | 🟢 |
| `resolvectl status` | The DNS servers the system is using | 🟢 |
| `resolvectl flush-caches` | Clears the local DNS cache | 🟡 |
| `cat /etc/resolv.conf` | The resolver setting (who is being asked) | 🟢 |
| `getent hosts example.com` | Uses the system resolution path (including `/etc/hosts`) | 🟢 |

> **The distinction (Phase 6.2.3):** If `dig @1.1.1.1` works and `dig` does not, the problem is not in DNS but
> in **your resolver.** If `getent` and `dig` give different results, `/etc/hosts` is in play.

## A.7 NAT, tunnels, IPv6 (Phase 7)

| Command | What it does | Risk |
|---|---|---|
| `curl -s ifconfig.me` | Your **public IP** as seen from outside (the result of NAT) | 🟢 |
| `ip addr` vs the above | Compare it with the private IP inside (RFC 1918) | 🟢 |
| `conntrack -L` | The connection tracking table (NAT/stateful records) | 🟢 |
| `conntrack -C` | The number of records in the table (when you suspect exhaustion) | 🟢 |
| `iptables -t nat -L -n -v` | The NAT rules (MASQUERADE/DNAT) | 🟢 |
| `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE` | Adds a source NAT rule | 🔴 |
| `ip -6 addr` / `ip -6 route` | IPv6 addresses and routes | 🟢 |
| `ping6 <destination>` / `curl -6` | Testing over IPv6 | 🟢 |
| `curl -4 <url>` | Force IPv4 on a dual-stack host (to tell them apart) | 🟢 |
| `ip tunnel show` / `wg show` | Tunnel / WireGuard status | 🟢 |
| `cat /proc/sys/net/netfilter/nf_conntrack_max` | The upper limit of the tracking table | 🟢 |

## A.8 HTTP and TLS (Phase 8)

| Command | What it does | Risk |
|---|---|---|
| `curl -I <url>` | Only the response headers (the status code) | 🟢 |
| `curl -v <url>` | The whole connection + TLS + request/response chain | 🟢 |
| `curl -w "%{time_total} %{http_code}\n" -o /dev/null -s <url>` | Measuring the time and the code | 🟢 |
| `curl --resolve example.com:443:1.2.3.4 https://example.com` | Skips DNS and tests **a specific server** | 🟢 |
| `curl -H "Host: example.com" http://1.2.3.4/` | A virtual host / routing test | 🟢 |
| `openssl s_client -connect example.com:443 -servername example.com` | See the TLS handshake raw (including SNI) | 🟢 |
| `openssl s_client ... \| openssl x509 -noout -dates -subject` | The certificate's validity dates and owner | 🟢 |
| `curl -vk <url>` | Skip certificate validation (**for diagnosis only**) | 🟡 |
| `curl --http1.1` / `--http2` | Force the protocol version | 🟢 |

> **Reading the codes (Phase 8.2.2):** **502** = the proxy **could not connect** to the backend · **504** = it
> connected but **no answer came** · **4xx** = the client side · **5xx** = the server side.

## A.9 Firewalls and filtering (Phase 9)

| Command | What it does | Risk |
|---|---|---|
| `iptables -L -n -v` | The rules + **the packet counters** (which rule is firing) | 🟢 |
| `nft list ruleset` | The whole nftables rule set | 🟢 |
| `ufw status verbose` | A simplified firewall state | 🟢 |
| `ufw allow 22/tcp` | Opens a port (permanent) | 🔴 |
| `iptables -I INPUT -p tcp --dport 8080 -j ACCEPT` | Adds the rule **at the top** (order matters) | 🔴 |
| `iptables -P INPUT DROP` | Makes the default policy deny (**you can lock yourself out**) | 🔴 |
| `iptables-save > /tmp/rules.bak` | Backs up the rules (**before** a change) | 🟢 |
| `iptables-restore < /tmp/rules.bak` | Restores the backup (recovery) | 🔴 |
| `conntrack -L -p tcp` | The open connections in the stateful table | 🟢 |

> **🔴 The rule (Phase 9.5.3):** Before changing rules remotely, **leave a separate session open** and take a
> backup with `iptables-save`. **Undo:** from the console/Session Manager, `iptables-restore < backup`.

## A.10 Packet capture and diagnosis (Phase 10)

| Command | What it does | Risk |
|---|---|---|
| `tcpdump -ni any 'tcp port 443'` | Live packet capture, filtered | 🟢 |
| `tcpdump -ni any 'host 10.0.2.20 and port 5432'` | Watches one specific flow | 🟢 |
| `tcpdump -ni any icmp` | ICMP messages (including Fragmentation Needed) | 🟢 |
| `tcpdump -w /tmp/capture.pcap ...` | Writes to a file (to examine with Wireshark) | 🟡 |
| `mtr <destination>` | A continuous traceroute + loss statistics | 🟢 |
| `mtr -rwc 100 <destination>` | Produces a 100-round report | 🟢 |
| `ping -c 100 -i 0.2 <destination>` | Measuring the loss rate | 🟢 |
| `ss -s` | A socket summary (total connection counts) | 🟢 |
| `journalctl -u <service> -e` | The evidence from the application side | 🟢 |

> **Reading tcpdump flags (Phase 10.2.2):** `[S]` SYN · `[S.]` SYN-ACK · `[.]` ACK · `[P.]` data · `[F.]` FIN
> · `[R]` RST. **If you see only `[S]`** it means no answer is coming → a DROP is suspected.

## A.11 The cloud side (Phase 11)

| Command | What it does | Risk |
|---|---|---|
| `aws ec2 describe-vpcs` | The VPCs and their CIDR blocks | 🟢 |
| `aws ec2 describe-subnets --filters Name=vpc-id,Values=<vpc>` | Subnets, AZs, CIDRs | 🟢 |
| `aws ec2 describe-route-tables` | The route tables (the public/private distinction is here) | 🟢 |
| `aws ec2 describe-security-groups --group-ids <sg>` | The SG rules | 🟢 |
| `aws ec2 describe-network-acls` | The NACL rules (numbered order!) | 🟢 |
| `aws ec2 authorize-security-group-ingress ...` | Adds a rule to an SG | 🔴 |
| `aws logs filter-log-events --log-group-name <flow-logs> ...` | Queries VPC Flow Logs (ACCEPT/REJECT) | 🟢 |
| `aws route53 list-resource-record-sets --hosted-zone-id <id>` | The DNS records | 🟢 |
| `curl -s http://169.254.169.254/latest/meta-data/local-ipv4` | The instance's own private IP (metadata) | 🟢 |

> **The diagnostic reflex (Phase 10.1):** When something breaks the order is always the same — **address →
> local network → routing → name → port → application.** Do not try commands at random; eliminate one layer
> at each step and only then climb up.

---

> **Navigation:** [◀ Phase 11 — The Bridge to the Cloud](Phase_11_Bridge_to_the_Cloud.md) · **Appendix A** · [Appendix B — Concept and File Map ▶](Appendix_B_Concept_File_Map.md)
