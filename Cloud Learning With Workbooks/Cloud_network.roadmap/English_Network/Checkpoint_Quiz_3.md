# Checkpoint Quiz 3 — Phases 5–9: "Why Was the Packet Dropped?"

> **Navigation:** [◀ Phase 9 — Firewalls, Filtering and Security](Phase_9_Firewalls.md) · **Checkpoint Quiz 3** · [Phase 10 — Troubleshooting ▶](Phase_10_Troubleshooting.md)

---

## What does this quiz measure?

This is the book's widest checkpoint quiz — it covers five phases at once. And there is a reason for that:
**everything from Phase 5 to Phase 9 is a piece of a single question.**

That question is: *"Why was the packet dropped?"*

Phase 5 gave you how reliable delivery is built — and where it breaks (the handshake, retransmission, MTU).
Phase 6, the translation of a name into an address; if that translation is wrong the packet **never goes to
the right place at all.** Phase 7, the sharing of a single address; if the translation table fills up or the
direction is wrong the packet **cannot come back.** Phase 8, how the web is built on top of these layers; the
difference between a 502 and a 504 is really a **transport** difference. And Phase 9, the packet being dropped
**deliberately**.

In a real incident these never stand apart. Behind a "the site will not open" complaint there can be an old
DNS record, a full NAT table, an MTU black hole, or a forgotten NACL rule. This quiz looks at whether you can
make that distinction.

**How to work through it:**

- Solve it with paper and pen; do not move to the next question before writing an answer.
- The answer key says which **intersection of phases** each question stands at — a question you miss points
  not at a phase but at the **bridge** between phases.
- There are 21 questions: Part A (connection reasoning, 1–9), Part B (a scenario: the "the payment service
  went down" incident, 10–15), Part C (reading commands and output, 16–21).
- Target time: ~1.5 hours. This quiz is longer than the others; solving it in pieces is entirely legitimate.

> **🤔 Before you start:** Put these two symptoms side by side and write the difference in one sentence: *"The
> connection waited 30 seconds and timed out"* and *"The connection got an immediate 'connection refused'."*
> That single distinction is the key to at least six of the questions below. Write your answer down; we will
> see it in Question 2.

---

# Part A — Connection reasoning (1–9)

**1.** In Phase 5 we saw TCP's three-way handshake, in Phase 9 the stateful firewall. Which **TCP property
directly** underlies a security group not needing a rule for return traffic? Why does the same mechanism have
to be imitated with an artificial timeout in UDP?

**2.** `curl https://api.example.com` gives two different errors on two different machines: on one a timeout
after 30 seconds, on the other an immediate "connection refused". Write which **layer** each one points at and
which team it requires you to call.

**3.** In Phase 6 we saw the DNS TTL, in Phase 8 HTTP. You moved a service to a new server and updated the DNS
record, but some of your users still go to the old server. What is the cause, and what was the right order for
preventing this problem **beforehand**?

**4.** In Phase 7 we saw NAT, in Phase 5 TCP. What happens if a row in the translation table on the NAT device
times out while the TCP connection is still open — how does the client experience it, and which mechanism is
the solution?

**5.** In Phase 8 we saw the difference between a 502 and a 504. Explain those two codes with **Phase 5's**
concepts: which one corresponds to the connection not being established, and which to the connection being
established but the answer not arriving in time?

**6.** In 5.7 we saw MTU, in 9.3 ICMP policy. A security team says "we turned off all ICMP". Which symptom
appears, why does that symptom not look like a firewall at all, and write the chain step by step.

**7.** In Phase 6 we saw CNAME, in Phase 8 TLS. A domain name is attached to a CDN with a CNAME and the browser
gives a "certificate error". Which mechanism of TLS (8.4) is in play, and should the error be looked for on the
DNS side or the TLS side?

**8.** In Phase 7 we saw the private subnet, in Phase 9 the security layers. Write the two separate errors in
the sentence "our machines are in a private subnet, so they are safe" — let one rest on Phase 7 and one on
Phase 9.

**9.** In Phase 5 we saw the ephemeral port, in Phase 9 the NACL. Why is a wide range (`1024-65535`) written in
a NACL's outbound rule rather than the service's port (`443`)? Why is that not needed in a security group?

---

# Part B — Scenario: the "the payment service went down" incident (10–15)

> In an e-commerce system the payment service connects over HTTPS to an external provider's API
> (`api.payments.example`). At 14:20 the alarms go off: 40% of payments are failing. The service is in a
> private subnet, goes out to the internet over a NAT Gateway, and has an ALB in front of it. The six
> questions below are your diagnosis, in order.

**10.** The first log line: `dial tcp: i/o timeout` — that is, the connection **cannot be established**. Which
possibilities does that single line eliminate, and which does it leave? How do you know at which step of the
handshake it got stuck?

**11.** You ran `dig api.payments.example` on the service machine and a proper IP came back. Which phase does
that observation eliminate, and which possibility does it **still not** eliminate? (Hint: the right answer
coming back does not mean the right answer is **being used**.)

**12.** You ran `tcpdump -ni any 'tcp port 443 and host <api-ip>'` and you see only `[S]` lines, no `[S.]` at
all. What is your diagnosis, and which distinction from Phase 9 does this observation **prove**?

**13.** **40%** of the requests fail and 60% succeed. What does that ratio give away — a completely closed
rule, or something resting on a capacity/resource limit? Which mechanism from Phase 7 fits this picture?

**14.** In VPC Flow Logs there are **ACCEPT** lines for the outbound connections on the NAT Gateway interface,
but for some flows there is **no return line at all**. What does that mean, and which of the NACL and the SG is
a more fitting suspect for this picture?

**15.** The root cause: the number of simultaneous connections the NAT Gateway can make to the same destination
IP:port pair has been exhausted (port exhaustion). Evaluate three solutions: (a) using a connection pool and
reusing connections, (b) adding a second NAT Gateway, (c) lengthening the TCP keepalive interval. Justify each
one with Phase 5 and Phase 7 reasoning — which one touches the root cause, and which makes things worse?

---

# Part C — Reading commands and output (16–21)

**16.** `dig api.example.com` output (abbreviated):

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 14203
;; ANSWER SECTION:
api.example.com.      45   IN  CNAME  d3kx9.cloudfront.net.
d3kx9.cloudfront.net. 60   IN  A      13.224.10.7
;; SERVER: 127.0.0.53#53(127.0.0.53)
```

(a) What are the numbers `45` and `60`? (b) Why does the `SERVER:` line matter? (c) If there were a **dangling
CNAME** risk in this chain, what would its symptom be?

**17.** `ss -tan` output:

```
State        Recv-Q  Send-Q  Local Address:Port    Peer Address:Port
ESTAB        0       0       10.0.1.50:51234       93.184.216.34:443
SYN-SENT     0       1       10.0.1.50:51290       203.0.113.9:443
TIME-WAIT    0       0       10.0.1.50:51201       93.184.216.34:443
CLOSE-WAIT   0       0       10.0.1.50:51188       93.184.216.34:443
```

Interpret all four states. Which one is a sign of **a fault** and which is normal? What does being stuck in
`SYN-SENT` show (Phase 5 and Phase 9 together)?

**18.** `curl -v https://example.com` output:

```
* Connected to example.com (93.184.216.34) port 443
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* SSL certificate verify ok.
> GET / HTTP/1.1
> Host: example.com
< HTTP/1.1 504 Gateway Timeout
```

Up to which layers is this output **successful**? Which side does the 504 point at, and which lines prove that
the problem is not at the client?

**19.** Two ping tests:

```
$ ping -c2 -M do -s 1400 10.20.0.9
2 packets transmitted, 2 received, 0% packet loss

$ ping -c2 -M do -s 1472 10.20.0.9
ping: local error: message too long, mtu=1436
```

What do these two outputs say together? What is the effective MTU, why is it not 1500, and which failure does
this lead to?

**20.** Two lines from VPC Flow Logs:

```
2 111... eni-0a1 10.0.1.50 10.0.2.20 51234 5432 6 8 1024 ... ACCEPT OK
2 111... eni-0b2 10.0.2.20 10.0.1.50 5432 51234 6 5  640 ... REJECT OK
```

Read the two lines together and make a diagnosis. Which direction was refused, which rule is missing, and what
is the symptom on the client side?

**21.** `iptables -L -n -v` output (abbreviated):

```
Chain INPUT (policy DROP 1204 packets, 96K bytes)
 pkts bytes target  prot opt source        destination
 8412  982K ACCEPT  all  --  0.0.0.0/0     0.0.0.0/0    ctstate RELATED,ESTABLISHED
   14   840 ACCEPT  tcp  --  0.0.0.0/0     0.0.0.0/0    tcp dpt:22
    0     0 ACCEPT  tcp  --  0.0.0.0/0     0.0.0.0/0    tcp dpt:8080
```

(a) What does `policy DROP` tell you? (b) Why has the first rule counted so many packets? (c) The third rule's
`pkts` value is 0 — what does that mean and which two different causes could it have?

---

## Answer key

**1.** It rests directly on **TCP's state information** (5.2.1): a SYN is the beginning of a connection, the
SYN-ACK is the answer, the ACK is its establishment. A stateful firewall reads those flags, writes the
connection into a table and **knows** that the returning packet is "part of a connection I already allowed"
(9.1.3). In UDP there are no such flags (5.1.1) — there is no concept of a connection — so the firewall keeps
an artificial timeout: if it sees no traffic for a while it deletes the row. · *Phase 5.1/5.2 × Phase 9.1*

**2.** **Timeout** → the packet was **dropped silently** (DROP, 9.2.3): a firewall (SG/NACL/OS), a missing
route (4.2) or an unreachable host. You look at the network/security side. **Connection refused** → an **RST**
came back (5.2.2), so the packet **reached** the target and the machine answered; the network layer is proved
and the problem is in the service (it is not listening or it is bound to the wrong address, 1.4.2). You go to
the application team. That one-word difference lets you find the right team on the first try. · *Phase 5.2 ×
Phase 9.2*

**3.** The cause is **the DNS cache and TTL** (6.4.1): clients and intermediate resolvers keep the old answer
for the duration of the TTL; even though you changed the record they carry on using the old one. The right
order (6.4.3): **first lower the TTL** (for example 3600 → 60), **wait as long as the old TTL** (so that all
the caches refresh), **then change the record**, and raise the TTL again once the move is complete. Lowering
the TTL **at the same moment** as the change does not work — because the caches out there still carry the old,
long TTL. · *Phase 6.4 × Phase 8.1*

**4.** When the row in the NAT table is deleted, the returning packets belonging to that connection **find no
record to match** and are dropped (7.1.2). The client experiences it like this: the connection seems to freeze
— data goes out but no answer comes; in the end it breaks with a TCP timeout or an RST. The classic symptom: **a
long-idle SSH session dying** (5.6.1). The solution is **TCP keepalive**: small packets sent at regular
intervals keep the row in the NAT/firewall table fresh. · *Phase 5.6 × Phase 7.1*

**5.** **502 Bad Gateway** = the proxy/load balancer **could not connect** to the server behind it or got an
invalid answer — in Phase 5's language: **the handshake could not be established** or the connection broke
early (RST/FIN, 5.2.2, 5.6.1). **504 Gateway Timeout** = the connection **was established** but the answer did
not arrive in time — that is, the handshake succeeded and the data flow is slow or the server cannot produce a
reply (8.2.2). In one sentence: *502 I could not connect, 504 I waited.* · *Phase 5.2/5.6 × Phase 8.2*

**6.** The symptom: **small packets get through, large packets do not** — an MTU black hole (5.7.3). The chain:
a device on the path has a smaller MTU → it cannot forward the large packet → normally it tells the sender with
an ICMP **Fragmentation Needed (type 3 code 4)** (4.4.2) → but because ICMP is blocked that message never
arrives → the sender retries at the same size → the packets disappear **silently**. The reason it does not look
like a firewall is that the symptom speaks the application's language: *"SSH connects but freezes"*, *"the
queries hang"*, *"the site opens but the images do not load"* (9.3.1). · *Phase 5.7 × Phase 9.3*

**7.** The mechanism in play is **certificate validation** (8.4.2): the browser looks at whether the
certificate the server presents covers the **requested domain name**. Even if the CNAME chain resolves
perfectly in DNS, if the CDN is not presenting a certificate for your domain name the validation fails. So the
error is **on the TLS side** — DNS did its job. The cause is usually that the certificate for the domain name
has not been configured on the CDN, or that the wrong certificate is being presented through **SNI** (8.4.2).
· *Phase 6.3 × Phase 8.4*

**8.** Error 1 (**Phase 7**): a private subnet only blocks connections **coming from the internet** (7.3.2);
the machines **can go out** over NAT, so a compromised machine can communicate with the outside and NAT does
not block that (7.1.2). Error 2 (**Phase 9**): a private subnet does not block connections **from inside the
VPC** — because of the `local` row in the route table every instance in the same VPC can reach it; the real
protection is the SG/NACL, and trusting a single layer violates the layered-defence principle (9.5.2). · *Phase
7.3 × Phase 9.5*

**9.** Because **a NACL is stateless** (9.1.2): it evaluates the returning packet independently, and that
packet's **destination port** is not the server's port but the client's **ephemeral port** (1.4.1, 5.1). Since
you cannot know in advance which port will be chosen, you have to open the whole range. It is not needed in an
SG because an SG is **stateful**: it holds the connection in its table and lets the returning packet through
without putting it into the rule check at all (9.1.3, 9.4.1). · *Phase 5.1 × Phase 9.1/9.4*

**10.** `i/o timeout` says **no answer came at all**: no SYN-ACK, no RST, no ICMP. That **eliminates** the
scenarios that return an RST (the service is not listening, 5.2.2) and those that give an immediate error like
"no route to host". What remains: a firewall DROP (SG/NACL/OS, 9.2.3), a missing route (outbound **or
return**, 4.2), a problem on the NAT side (7.1.2), or the target not being up. You see at which step of the
handshake it got stuck **with `tcpdump`**: if there are only `[S]` lines it is stuck at the first step (5.2.1).
· *Phase 5.2 × Phase 9.2*

**11.** **It eliminates Phase 6 (DNS resolution)** — the name resolves to the right IP. But it does **not**
eliminate this: whether the application is really using that IP. The application may be holding **an old DNS
cache** of its own (some runtimes keep resolved IPs for a long time, 6.4.2), or the existing connections in the
connection pool may have been made to an old IP. `dig`'s result is the **resolver's** answer, not the address
**the application** is using. The definitive check: look with `ss -tan` at which peer IPs the application is
really connected to. · *Phase 6.2/6.4 × Phase 5.3*

**12.** The diagnosis: **the SYN goes out, no SYN-ACK comes back** — either the packet is not reaching the
target or the answer is being dropped; in either case something in between is dropping it **silently** (DROP).
The distinction it proves: the **DROP vs REJECT** distinction in 9.2.3 — with a REJECT we would see an `[R]`
(RST) and the symptom would be "connection refused" (5.2.2). Silence is the signature of a deliberate filtering
policy. · *Phase 5.2 × Phase 9.2*

**13.** **It cannot be a completely closed rule** — if the rule were closed the failure rate would be 100%. A
partial 40% failure points at a **capacity limit**: a resource that fills up under load and works again as it
empties. The mechanism from Phase 7 that fits is **NAT/PAT's port limit** (7.2.1): because of the 16-bit port
space, the number of simultaneous connections that can be made to the same destination is limited; when the
table fills new connections cannot be made while the existing ones keep working. The same class of problem is
seen in a stateful firewall's connection table too (9.1.3). · *Phase 7.2 × Phase 9.1*

**14.** An outbound **ACCEPT** means the packet reached the NAT Gateway and was accepted. **No return line at
all** shows that the answer did not come back — so the problem is either on the far side or on the return path
(10.3.3, 10.4.3). As a suspect, **the NACL fits better than the SG**: because an SG is stateful it would
already let the return traffic through; in a stateless NACL, if the outbound ephemeral rule is missing the
return is refused or dropped (9.4.1). A note: no return line at all is different from a REJECT — if you saw a
REJECT it would point straight at the rule; no line at all means "no answer came". · *Phase 7.1 × Phase 9.4*

**15.** (a) **The connection pool — the only solution that touches the root cause.** Reusing existing
connections instead of opening a new one for every request directly lowers the number of simultaneous
connections (and saves the handshake cost of 5.2.1; the HTTP keep-alive idea in 8.3). (b) **A second NAT
Gateway** — it works but it is **palliative**: it doubles the limit, does not solve the root and increases the
cost; when the traffic grows you hit the same wall. (c) **Lengthening the keepalive interval — it makes things
worse:** connections stay in the table longer and port exhaustion comes faster. (Keepalive is the solution to
the **premature deletion** problem in Question 4; here the problem is the exact opposite.) · *Phase 5.2/5.6 ×
Phase 7.2*

**16.** (a) **The TTL values** — how many seconds the answer will be cached (6.4.1); they have been kept low
here, so it is a configuration ready for fast change. (b) The `SERVER:` line says **which resolver you asked**
(6.2.3); `127.0.0.53` is the local stub resolver. It is critical in diagnosis: if `dig @1.1.1.1` works and the
local query does not, the problem is not in DNS itself but in **your resolver**. (c) The **dangling CNAME**
risk: if the CNAME's target (the CDN distribution) is deleted but the record stays, the name cannot be resolved
(NXDOMAIN) — or worse, if that target name is taken over by someone else it becomes a **subdomain takeover**
(6.3.3). · *Phase 6.3 × Phase 6.4*

**17.** **ESTAB** = an established, healthy connection. **SYN-SENT** = the SYN was sent and the **SYN-ACK is
awaited** — being stuck here shows that the answer never came: a firewall DROP or an unreachable target (5.2.2,
9.2.3). **This is the real sign of a fault.** **TIME-WAIT** = the side that closed the connection is waiting
for delayed packets — it is **normal** (5.6.2); if its count rises very high it points at an excess of
short-lived connections. **CLOSE-WAIT** = the other side sent a FIN but **the local application has not closed
the socket** — a few are normal, but when they pile up it is **an application bug** (a socket leak, 5.6.1). ·
*Phase 5.2/5.6 × Phase 9.2*

**18.** The successful layers: **DNS** (the name resolved), **TCP** (`Connected ... port 443` — the handshake
completed, 5.2.1), **TLS** (`certificate verify ok` — the handshake and validation completed, 8.4.2) and **the
HTTP request was sent.** The 504 points at the **server side**: the proxy/load balancer connected to the
service behind it but could not get its answer in time (8.2.2). Those are exactly the lines proving there is no
problem at the client — the network, TLS and request chain all worked completely; the only remaining suspect is
the backend. · *Phase 8.2/8.4 × Phase 5.2*

**19.** `-M do` forbids fragmentation, `-s` sets the data size (5.7.3). 1400 bytes gets through, 1472 does not,
and the kernel says **`mtu=1436`**. So the effective MTU is **1436**, not 1500 — the 64 bytes in between are
the header overhead added by a **tunnel** (VPN/IPsec/VXLAN) (7.4.2). The failure it leads to: an **MTU black
hole** — if ICMP Fragmentation Needed is blocked, large packets disappear silently and the symptom becomes "it
connects but it hangs" (5.7.3, 9.3.1). The solution is usually **MSS clamping** (5.7.3). · *Phase 5.7 × Phase
7.4*

**20.** The first line: `10.0.1.50 → 10.0.2.20:5432` **ACCEPT** — the request reached the database. The second
line: `10.0.2.20:5432 → 10.0.1.50:51234` **REJECT** — the **return** was refused. The missing rule: the target
subnet's **NACL outbound ephemeral** rule (`ALLOW TCP 1024-65535`), because the answer goes to the client's
ephemeral port (9.4.1). The symptom on the client side: the SYN is sent, the SYN-ACK cannot come back → a
**timeout** (9.2.3) — that is, "I cannot connect to the database", even though the request reached the
database. · *Phase 5.1 × Phase 9.4 × Phase 10.4*

**21.** (a) `policy DROP` says that packets matching no rule will be **dropped silently** — the right default,
**default deny** (9.2.2); the counter beside it (1204 packets) shows it is actively working. (b) The first rule
applies **stateful tracking** with `ctstate RELATED,ESTABLISHED` (9.1.3): the return traffic of every
connection the machine started itself passes through this rule — that is the overwhelming majority of normal
traffic. (c) `pkts = 0` says that rule has **never matched**. It can have two causes: (i) **no traffic has come
to 8080 at all** (the service is not being used or the client never tries), (ii) traffic is coming but **a rule
above it** is catching it first (an ordering problem, 9.2.2). To tell them apart you look with `tcpdump` at
whether packets are arriving at 8080 (10.2.2). · *Phase 9.1/9.2 × Phase 10.2*

---

## Scoring

| Correct answers | Assessment |
|---|---|
| 19–21 | You have all the pieces of the "why was the packet dropped" question. Move on to Phase 10 with confidence. |
| 15–18 | Solid. Take one pass back over the **bridge** the questions you missed point at. |
| 10–14 | You know the phases separately but the joins are weak. Use the table below. |
| 0–9 | Go over the trio of 5.2 (the handshake), 9.1 (stateful) and 9.2.3 (DROP/REJECT); without those three Phase 10's methodology does not work. |

**Where to go back to for a question you missed:**

| Question missed | Go back — this bridge is weak |
|---|---|
| 1, 9, 14, 20 | Phase 5.1/5.2 × Phase 9.1/9.4 — stateful/stateless and the ephemeral port |
| 2, 10, 12, 17 | Phase 5.2 × Phase 9.2 — timeout vs refused, handshake signatures |
| 3, 11, 16 | Phase 6.2/6.3/6.4 — resolution, CNAME, TTL and the cache |
| 4, 13, 15 | Phase 5.6 × Phase 7.1/7.2 — the NAT table, port exhaustion, keepalive |
| 5, 18 | Phase 5.2/5.6 × Phase 8.2 — the transport equivalent of 502 vs 504 |
| 6, 19 | Phase 5.7 × Phase 7.4 × Phase 9.3 — MTU, tunnels, ICMP policy |
| 7 | Phase 6.3 × Phase 8.4 — the CNAME chain and certificate validation |
| 8, 21 | Phase 7.3 × Phase 9.5 — layered defence and default deny |

---

## Closing — from here to Phase 10

The five phases from 5 to 9 taught you **everything** that can happen to a packet: getting lost on the way and
being retransmitted (Phase 5), going to the wrong address (Phase 6), not being able to come back (Phase 7),
getting the wrong answer (Phase 8) and being dropped deliberately (Phase 9).

You now recognise each one's **signature**: timeout or refused, which TCP state it got stuck in, whether the
TTL had expired, whether the answer came from an old cache, whether the return rule was there.

But something is missing. All these signatures stand separately in your mind — and in a real incident you will
have thirty possibilities, one user and little time.

Phase 10 connects exactly here: it puts the knowledge of these five phases **into an order**. A disciplined
bottom-up methodology, which layer each tool looks at, and in the cloud, **reading** why a packet was dropped
**from Flow Logs** instead of guessing. Every question you answered one by one in this quiz will be gathered
there into a single decision tree.

> **Before you go on:** If you can answer questions 2 and 12 above without hesitation — why the difference
> between timeout and refused points at two different teams — you are ready for Phase 10. If you cannot, take a
> pass back over 5.2 and 9.2.3; the whole methodology of Phase 10 stands on that distinction.

---

> **Navigation:** [◀ Phase 9 — Firewalls, Filtering and Security](Phase_9_Firewalls.md) · **Checkpoint Quiz 3** · [Phase 10 — Troubleshooting ▶](Phase_10_Troubleshooting.md)
