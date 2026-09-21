# Phase 6 — From Name to Address: DNS

> **Navigation:** [◀ Phase 5 — The Transport Layer](Phase_5_Transport_Layer.md) · **Phase 6** · [Phase 7 — NAT and the Real World ▶](Phase_7_NAT.md)

---

## Where we come from

At the end of Phase 5 we asked: when you type `curl https://example.com`, your machine has to know the
**destination IP** in order to build the SYN packet (5.2.1) — but all it has is a **name**. Who will it ask,
and how will it know that "who"'s address?

We also gave you the hint: it is **one of the four things DHCP hands out** (1.6.1). DHCP gives you an IP, a
mask, a gateway and a **DNS server**. What that fourth one is good for is the subject of this phase.

What you brought with you:

- **Why UDP is the right choice for a single-packet question and answer** (5.1.2) — DNS is its purest
  example.
- **That there is one more step before the connection is established** — this phase fills that step in.
- **The idea of a cache** — you saw the switch's MAC table in Phase 3 and the ARP cache in Phase 4. DNS is
  the largest-scale application of the same idea, and it carries the same trap: **stale information.**

## The question of this phase

> *"There are billions of names in the world and no machine knows them all. How does a name turn into the
> address its owner chose — and do it this fast?"*

The answer lies in two ideas: **hierarchy** (nobody knows everything, but everyone knows the next step) and
**caching** (the same question is not asked twice).

There is a joke in the industry and it is not for nothing:

> **"It's always DNS."**

The truth behind the joke: when DNS breaks, the symptom looks like **"there is no internet"** — while IP,
routing and TCP are perfectly healthy. That misleading signature makes DNS the layer that wastes the most
time in diagnosis. By the end of this phase you will recognise that signature.

---

## By the end of this phase

- You will be able to explain the root → TLD → authoritative chain and why a distributed system was built
- You will be able to tell a recursive resolver from an iterative query — who is working on whose behalf
- You will be able to read `dig +trace` output line by line
- You will know the A, AAAA, CNAME, MX, TXT and NS records and the traps of a CNAME chain
- You will be able to explain what TTL is, at which levels caching happens and **how it affects
  deployments**
- You will be able to diagnose the symptom "ping to an IP works but names do not resolve" within seconds
- **Cloud:** You will be able to explain Route 53's role, alias records and the effect of the TTL setting on
  outage duration

---

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 6.1 | The DNS hierarchy | `[mechanism]` | Nobody knows everything |
| 6.2 | The resolution chain | `[mechanism]` | **The heart of the phase** — the journey of a question |
| 6.3 | Record types | `[concept]` | What is behind a name |
| 6.4 | Caching and TTL | `[mechanism]` | Both a source of speed and a source of failure |
| 6.5 | When this phase breaks | — | The "It's always DNS" signatures |

> **How to work through this phase:** Every command in this phase is 🟢 — `dig`, `nslookup`, `host` and
> `resolvectl` only ask, they change nothing. So you can experiment here without fear. The one exception is
> adding a line to `/etc/hosts` (you will see it in 6.2.3) — that is 🟡 and undoing it means deleting one
> line. The single most instructive command is **`dig example.com +trace`**: you see the whole chain,
> starting from the root, with your own eyes. Reading that output carefully once handles half of this phase.
> If `dig` is not installed: `sudo apt install dnsutils` (Debian/Ubuntu) or `sudo dnf install bind-utils`
> (RHEL/Fedora).

---
---

# 6.1 The DNS Hierarchy

## 6.1.1 Why there is no single centre `[concept]`

The first design that comes to mind would be this: one giant table with all the names and IPs in it, and
everybody asks it.

That design collapses for three reasons:

1. **Scale.** Hundreds of millions of domain names and millions of queries per second — no single system
   can carry that.
2. **Administration.** Who would change `example.com`'s IP? If it required permission from a central
   authority, the internet would not work. The owner of a name must be able to manage their own record
   **themselves**.
3. **Resilience.** If the single centre goes down, the internet ends.

The solution: **divide the responsibility.** Nobody knows everything; everybody knows only **the next
step**.

## 6.1.2 The links in the chain `[mechanism]`

DNS is a tree read from right to left. The structure of the name `www.example.com.`:

```
www   .   example   .   com   .
 │          │           │     └── root (the invisible dot)
 │          │           └──────── TLD (Top-Level Domain)
 │          └──────────────────── second-level domain (this is the owner's)
 └─────────────────────────────── subdomain / host
```

You do not normally type the trailing dot, but it **is there** — it is the root itself.

The roles in the chain:

| Level | Who | What it knows |
|---|---|---|
| **Root** | 13 logical root server clusters (hundreds of physical copies via anycast) | The servers for TLDs like `.com`, `.org`, `.uk` |
| **TLD** | The organisation that runs `.com` | `example.com`'s **authoritative** servers |
| **Authoritative** | The domain owner's DNS server (or a service like Route 53) | `www.example.com`'s **actual IP** |
| **Recursive resolver** | The ISP's, the company's, or a service like 8.8.8.8 | None of them — but it knows **how to ask** |

The critical point: **a root server does not know `example.com`'s IP, and it does not need to.** It only
says "ask these servers about `.com`". Every level points to the next one. Responsibility is distributed —
and that is exactly why it scales.

> **⚠️ Common misconception: "There are 13 root servers, so there are 13 machines in the world."**
>
> No. There are **13 addresses** (A through M), but behind each address there are hundreds of physical
> servers distributed by **anycast**. Anycast (you will see it in Phase 8.5) means announcing the same IP
> address from many points in the world and letting routing take each user to the **nearest copy** (Phase
> 4.3 — longest prefix match and BGP come into play here). So when you ask the root from Istanbul, the
> answer probably comes from a copy in Istanbul. The number 13 is not a capacity limit; it is a leftover
> from an old packet-size constraint.

---
---

# 6.2 The Resolution Chain

## 6.2.1 The recursive resolver: the one that runs on your behalf `[mechanism]`

When your machine wants to resolve `www.example.com` it **does not walk the chain itself**. It asks one
single place: the **recursive resolver** DHCP gave it (or that you configured by hand) (1.6.1).

The difference between the two is the most important distinction in this section:

- **Your query: recursive** — *"Find me the answer. How you find it is not my concern."*
- **The resolver's queries: iterative** — at each step it asks one server, gets "ask the next one" back, and
  walks the chain itself.

So **the resolver does all the work**; you just ask one question and get one answer.

## 6.2.2 The chain step by step `[mechanism]`

Your machine asked for `www.example.com` and the resolver's cache is empty. What happens:

```
1. Machine  → Resolver   : "what is www.example.com?"          (recursive)
2. Resolver → Root       : "www.example.com?"
   Root     → Resolver   : "I don't know. Ask these NSes about .com."
3. Resolver → .com TLD   : "www.example.com?"
   TLD      → Resolver   : "I don't know. Ask these NSes about example.com."
4. Resolver → Auth. NS   : "www.example.com?"
   Auth.NS  → Resolver   : "A record: 93.184.216.34"           (authoritative answer)
5. Resolver → Machine    : "93.184.216.34"  (+ TTL)
```

Five steps — and all of them typically finish in **tens of milliseconds**. But the real speed does not come
from here: the resolver **caches** this answer (6.4), and the second time the same name is asked the chain
is not walked at all.

![Figure 6.1 — The DNS resolution chain: the client sends a single recursive query; the resolver visits the root, TLD and authoritative servers in turn (iteratively) to find the answer and keeps it in its cache for the duration of the TTL. The client never sees the chain.](../diagrams/png/nw-6-01-dns-resolution.png)

In the figure the single arrow leaving the client on the left is the recursive query, and the three arrows
leaving the resolver are the iterative chain. The three boxes on the right are the three levels of the
hierarchy (6.1.2). The dashed box under the resolver is the cache — on a second query only that box is
visited and the three arrows are never drawn.

> **🔧 See it on your machine** 🟢 — walk the chain with your own eyes
>
> ```
> $ dig example.com +trace
>
> .                   518400  IN  NS  a.root-servers.net.      ← the root NS list
> ...
> com.                172800  IN  NS  a.gtld-servers.net.      ← referral to the TLD
> ...
> example.com.        172800  IN  NS  a.iana-servers.net.      ← referral to the authoritative
> ...
> example.com.        86400   IN  A   93.184.216.34            ← the final answer
> ;; Received 56 bytes from 199.43.135.53#53(a.iana-servers.net)
> ```
>
> `+trace` repeats the resolver's work **on your machine**: it starts at the root and at each step asks the
> next NS set. When reading the output, look at three things: (1) **from which level to which level** it
> moves, (2) the **TTL** value on each line (518400 = 6 days, 86400 = 1 day — 6.4.1), (3) the **"Received
> ... from"** line at the bottom, which tells you which server the answer came from. This one command makes
> the whole of 6.1 and 6.2 visible.

## 6.2.3 The steps before the query `[concept]`

In fact, before it goes to the resolver your machine looks in a few more places. The order is usually:

1. **The application's own cache** (browsers keep their own DNS cache)
2. **The operating system's cache** (`systemd-resolved`, the DNS Client service on Windows)
3. **The `/etc/hosts` file** — hand-written mappings. If there is an entry here, DNS is **never** consulted.
4. **The recursive resolver** (the address written in `/etc/resolv.conf`)

The third item is critical in diagnosis: a single forgotten line in `/etc/hosts` creates a failure like "DNS
gives the right answer but the machine goes to the wrong place" — and `dig` **does not show it**, because
`dig` asks the resolver directly and never looks at `/etc/hosts`.

> **🔧 See it on your machine** 🟢 — find out who your resolver is
>
> ```
> $ cat /etc/resolv.conf
> nameserver 127.0.0.53          ← the local stub resolver (systemd-resolved)
> search lan
>
> $ resolvectl status | grep -A2 'Link 2'
> Current DNS Server: 192.168.1.1
> DNS Servers: 192.168.1.1
> ```
>
> On modern Linux `/etc/resolv.conf` usually shows `127.0.0.53` — that is a **local intermediary**
> (systemd-resolved), not the real resolver. To see the real address use `resolvectl status`. The IP you see
> there is most likely the home router DHCP gave you (1.6.1), and that in turn forwards to the ISP's
> resolver. These layers in the chain are why diagnosing "DNS is not working" requires asking **which
> layer** is broken.

> **🤔 Think 6.1** — `dig example.com` returns the right IP, but your browser still goes to the old server.
> (a) Write two possible causes. (b) Explain why `dig` cannot see one of them. (c) With which command or
> file would you confirm it?
>
> *(Answer: at the end of the phase)*

> **🤔 Think 6.2** — A user says "the internet is gone". You test: `ping 8.8.8.8` **works**, `ping
> google.com` **does not**. (a) Which layer is the problem in? (b) When you see this symptom, what do you
> know **for certain** about IP, routing and TCP? (c) What are your next two commands?
>
> *(Answer: at the end of the phase)*

---
---

# 6.3 Record Types

## 6.3.1 The basic records `[concept]`

DNS is not just name→IP. Different **types** of record live under a name; a query is always made with a
type.

| Type | What it returns | Example use |
|---|---|---|
| **A** | An IPv4 address | `example.com → 93.184.216.34` |
| **AAAA** | An IPv6 address | `example.com → 2606:2800:220:1:...` (1.5) |
| **CNAME** | Another **name** (an alias) | `www.example.com → example.com` |
| **MX** | A mail server (+ priority) | Where email is delivered |
| **NS** | This zone's authoritative servers | The referrals in the chain (6.1.2) |
| **TXT** | Free text | Domain-ownership verification, SPF/DKIM |
| **PTR** | IP → name (reverse resolution) | Who an IP in the logs belongs to |
| **SRV** | Service + port + host | Service discovery |

In practice you will deal mostly with **A**, **CNAME** and **NS**.

## 6.3.2 CNAME and its traps `[concept]`

A CNAME says "this name is really an alias of that name". It is useful: if you CNAME `www.example.com` to a
CDN's name, you do nothing at all when the CDN changes its IPs.

But it has three traps:

**1. A CNAME cannot be used at the apex (the root).** You cannot define a CNAME for `example.com` — because
the root **must** carry NS and SOA records, and the standard says a name that has a CNAME **can have no
other record**. You will hit this constantly in cloud services (the solution is the alias records in 6.3.3).

**2. The cost of a chain.** You can build CNAME → CNAME → A chains, but each link means another resolution
round. Long chains add latency.

**3. The danger of a broken chain.** If the name you CNAMEd to is deleted one day, your name's resolution
collapses too — and the reason is **invisible** in your own DNS. Worse: the deleted target can be claimed by
**someone else** at a cloud provider and your traffic goes to them (this is called a **dangling CNAME /
subdomain takeover**, and it is a real security vulnerability).

## 6.3.3 Managing names in the cloud `[concept]`

> **💡 Cloud connection — Route 53 and alias records:** AWS's DNS service is **Route 53** and it plays two
> roles at once: it is both an **authoritative server** (it holds your zone's records, 6.1.2) and a
> **recursive resolver** (the `.2` address inside the VPC — one of the reserved addresses you saw in 2.3).
> Route 53's most important addition is the **alias record**: it solves the problem of CNAME not being
> usable at the apex (6.3.2). You can alias `example.com` straight to an ALB, a CloudFront distribution or
> an S3 website — as far as the DNS standard is concerned it is answered like an A record, but behind the
> scenes the target's current IP is looked up. Its second difference: alias queries are free and AWS manages
> the TTL. Inside a VPC there is also a **private hosted zone** — internal names resolvable only from that
> VPC (the basis of service discovery). What you need to know for diagnosis: if a name does not resolve from
> outside but does resolve from an instance, you are probably in a private hosted zone.

> **🤔 Think 6.3** — A team wants to point `example.com` (the apex) at a load balancer and gets the error
> "CNAME not accepted". (a) Why is it not accepted? (b) What is the solution in AWS? (c) If they were at a
> provider other than AWS, what options would they have?
>
> *(Answer: at the end of the phase)*

---
---

# 6.4 Caching and TTL

## 6.4.1 TTL: "this answer is fresh for this long" `[mechanism]`

Every DNS record has a **TTL** (Time To Live) value, in seconds. Its meaning: *"You may keep this answer for
this long; you do not need to ask again."*

```
example.com.   300   IN  A   93.184.216.34
               └── TTL: 300 seconds = 5 minutes
```

This value is set by **the record's owner** (on the authoritative server) and everyone in the chain obeys
it.

Typical values and what they mean:

| TTL | Duration | When it is used |
|---|---|---|
| 60 | 1 min | Frequently changing records, before a failover |
| 300 | 5 min | Actively managed services |
| 3600 | 1 hour | Ordinary websites |
| 86400 | 1 day | Rarely changing records (NS, MX) |

The trade-off is clear: **low TTL = fast change but more queries; high TTL = few queries but slow change.**

## 6.4.2 There is a cache at every level `[mechanism]`

This is the reason DNS failures are so confusing. One answer is stored in **many places at once**:

```
The browser's cache
   └── The operating system's cache (systemd-resolved)
          └── The local/corporate resolver's cache
                 └── The ISP resolver's cache
                        └── (the authoritative server — the single source of truth)
```

When you change a record, the authoritative server gives the new answer **immediately**. But every layer
above keeps using the old answer it holds **until the TTL expires**. So:

> The change reaching everybody takes **as long as the old TTL**.

And here is the most common mistake: **lowering the TTL at the moment of the change does not help.** Because
the old record sitting in caches was stored with the **old TTL**. The time to lower the TTL is one **old
TTL** period **before** the change.

## 6.4.3 The correct deployment order `[application]`

If you are planning a server move (an IP change), the correct order is:

1. **At least one TTL before the change**, lower the TTL (e.g. 3600 → 60). Within an hour the old records
   expire and are refreshed with the new, short-TTL version.
2. **Wait** for everybody to move to the short TTL (as long as the old TTL).
3. **Change the record.** Now the world sees the new address within 60 seconds at most.
4. Once the transition has settled, raise the TTL back to normal.
5. **Do not shut the old server down immediately** — keep it up for at least a few TTLs; there will be
   latecomers.

The fifth item matters especially: in DNS there is **no** moment when "everybody has moved", only "very few
are left". Some clients do not obey TTLs (some applications cache a record indefinitely).

> **🔧 See it on your machine** 🟢 — watch the TTL count down
>
> ```
> $ dig example.com | grep -A1 'ANSWER SECTION'
> ;; ANSWER SECTION:
> example.com.    3542    IN  A   93.184.216.34
>
> # the same command 10 seconds later:
> example.com.    3532    IN  A   93.184.216.34
>                 └── it went down: this is coming from the resolver's cache
> ```
>
> If the TTL value **goes down on every query**, the answer is coming from the cache — the number shows how
> much longer that record will be considered fresh. When it reaches zero the resolver walks the chain again
> and the TTL returns to its full value. This small experiment makes 6.4.1 and 6.4.2 concrete in one go. If
> you ask a different resolver with `dig @8.8.8.8 example.com` you will see a **different TTL** — because
> its cache was filled at a different time.

> **⚠️ Common misconception: "I changed the DNS record and it still goes to the old place — the change did
> not take effect."**
>
> It did take effect. If you ask the authoritative server directly with `dig @<authoritative-ns>
> example.com` you will see the new answer. The old answer you are seeing is coming from one of the **cache
> layers** in between (6.4.2). The right reflex: ask the authoritative directly first — that is what
> separates "was it really published" from everything else. Then clear the cache on your own side (`sudo
> resolvectl flush-caches` 🟡, no undo needed — the cache refills by itself). You cannot clear the ISP's or
> your users' caches; there the only solution is **waiting**, and how long is determined by the old TTL.

> **💡 Cloud connection — TTL and seamless transitions:** In Route 53 a record's TTL is a direct part of
> your deployment strategy. If you switch traffic to a new environment with DNS in a blue/green deployment,
> your switching speed is **equal to the TTL** — with a TTL of 3600 a rollback takes an hour, which is
> unacceptable during an incident. That is why 60 seconds is typical for actively managed records. But there
> is a better approach: **do not use DNS as a traffic switch.** Instead keep a fixed name (the ALB's alias)
> and switch traffic between the load balancer's **target groups** — there the transition happens in seconds
> and is independent of caches (Phase 8.5). Save DNS-level redirection for coarser decisions such as
> regional failover. Route 53 health checks + failover routing exist exactly for that, and they too are
> subject to the TTL.

> **🤔 Think 6.4** — Tomorrow morning you are moving a web server to a new IP. The record's TTL is currently
> 86400 (1 day). (a) What should you do today and why? (b) Write the order of operations for the move day.
> (c) When can you shut the old server down?
>
> *(Answer: at the end of the phase)*

---
---

# 6.5 When This Phase Breaks — The "It's Always DNS" Signatures

DNS failures have one thing in common: **the symptom looks nothing like the cause.** The user says "there is
no internet", the application says "connection timeout", the developer calls the network team — while IP,
routing and TCP are working flawlessly. The most valuable reflex of this phase is lifting that mask
quickly.

| Symptom | Likely cause | Verify | Related section |
|---|---|---|---|
| `ping 8.8.8.8` ✓ but `ping google.com` ✗ | The resolver is unreachable or wrong | `cat /etc/resolv.conf`, `dig @8.8.8.8 google.com` | 6.2.1 |
| Everything is slow to open, then normal | A slow/timing-out resolver, falling back to the second | The `Query time` in `dig` output | 6.2.2 |
| The record changed but the old IP is used | Cache + TTL | Ask directly with `dig @<auth-ns>` | 6.4.2 |
| `dig` is right, the browser goes to the wrong place | `/etc/hosts` or the application cache | `cat /etc/hosts` | 6.2.3 |
| A CNAME at the apex is rejected | A standard restriction (apex) | The provider's alias record | 6.3.2 |
| A subdomain suddenly goes to someone else | A dangling CNAME / subdomain takeover | `dig CNAME <name>`, check the target | 6.3.2 |
| Internal names resolve, external ones do not | The internal resolver cannot get out (firewall/UDP 53) | Compare with `dig @8.8.8.8` | 6.2.1 |
| External names resolve, internal ones do not | The wrong resolver, or no private-zone access | `resolvectl status` | 6.3.3 |
| Some users see the new IP, some the old | Different resolvers' caches filled at different times | `dig` from different resolvers | 6.4.2 |
| `NXDOMAIN` is returned | The name really does not exist — a typo or a deleted record | Walk the chain with `dig +trace` | 6.2.2 |
| `SERVFAIL` is returned | The authoritative server is not answering, or a DNSSEC error | `dig +trace`, look at the last step | 6.2.2 |
| Mail does not go but the site works | The MX record is missing/wrong (the A record is separate) | `dig MX <domain>` | 6.3.1 |

> **The lesson from this table:** In DNS diagnosis **a single test** splits most cases in two: **if `ping
> 8.8.8.8` works and `ping google.com` does not, the problem is DNS — and that is proof that all the layers
> below are healthy** (there is an IP, there is routing, packets go out and come back). That one observation
> makes calling the network team unnecessary. The second reflex: **where is the answer coming from?** Ask
> the resolver with `dig`, ask the source with `dig @<authoritative>` — if the two differ the problem is the
> **cache** and the fix is waiting or flushing; if they are the same the problem is **the record itself**
> and the fix is on the authoritative. Third: if `dig` gives the right answer and the application misbehaves,
> your suspects should be `/etc/hosts` and the application cache (6.2.3) — because `dig` **skips** those
> layers. Finally, the real lesson of the "It's always DNS" joke is not that DNS is fragile: because DNS is
> **the first step of everything**, when it breaks every layer after it appears to fail too. You can now
> test the first step separately.

---
---

# Answers to the Think questions

## Answer 6.1 — The layers `dig` does not see

**(a)** Two causes: (i) there is an old hand-written line for that name in **`/etc/hosts`** — the system
never goes to DNS at all (6.2.3, item 3). (ii) **The browser's own DNS cache** still holds the old answer
(6.2.3, item 1; 6.4.2).

**(b)** Because **`dig` does not follow the resolution order** — it sends a UDP 53 query straight to the
resolver (or to the server you name with `@`). It **skips** `/etc/hosts`, the operating system cache and the
application cache entirely. That is why "dig is right but the application is wrong" is exactly the sign of a
problem in one of those layers. (If you want, use `getent hosts example.com` — that follows the system's
real resolution order and takes `/etc/hosts` into account.)

**(c)** Check for a hand-written entry with `cat /etc/hosts`; see what the system actually finds with
`getent hosts <name>`; for the browser cache close and reopen the browser or try a private window; for the
system cache `sudo resolvectl flush-caches` 🟡 *(No undo needed — the cache refills by itself.)*

**Related section:** 6.2.3 · **Next:** 6.4.2 (the cache layers), Phase 10.2 (choosing a tool)

## Answer 6.2 — The single most valuable test

**(a)** **DNS** (6.5). Name resolution is not working.

**(b)** You know a great deal, and all of it is **certain**: the machine has an IP and it is configured
correctly (Phase 1); the subnet/mask is right, otherwise even the local network would not work (Phase 2);
ARP and L2 are working, the gateway's MAC has been found (Phase 3); **routing works** — the packet leaves
through the default gateway, goes to the internet and **comes back** (Phase 4); there is no firewall
blocking packets along the way, at least for ICMP. So the whole of Phases 1–4 is healthy. Only one thing
remains: the name→IP translation.

**(c)** (i) **`cat /etc/resolv.conf`** or `resolvectl status` — who is the resolver, is its address
plausible? (ii) **`dig @8.8.8.8 google.com`** — ask a resolver known to work. If an answer comes back the
problem is **your resolver** (unreachable, crashed, or the wrong address was handed out — the value from
DHCP may be broken, 1.6.1). If no answer comes back, UDP 53 traffic may be blocked (Phase 9).

**Related section:** 6.2.1, 6.5 · **Next:** Phase 10.3 (layer-by-layer diagnosis)

## Answer 6.3 — No CNAME at the apex

**(a)** Because the standard says a name that has a CNAME **can have no other record**. And at the apex
(`example.com`) there **must** be NS and SOA records — the records that point to the zone's own authoritative
servers (6.1.2). The two cannot coexist, so a CNAME at the apex is forbidden (6.3.2).

**(b)** In AWS, an **alias record** (6.3.3). It is Route 53's extension outside the DNS standard: it answers
like an A record on the outside but looks up the current IP of the ALB/CloudFront/S3 target behind the
scenes. It is used at the apex without trouble, and there is no query charge either.

**(c)** Three options: (i) The provider's own equivalent — most modern DNS providers have the same feature
under the name **ALIAS** or **ANAME**. (ii) Write a plain **A record** at the apex — but then you have to
update it by hand whenever the load balancer's IP changes, and cloud LB IPs are volatile, so this is a
fragile solution. (iii) Send the apex to `www` with an **HTTP redirect** (`example.com` → 301 →
`www.example.com`) and keep the real record as a CNAME on `www` — a common, working pattern, but it adds a
round trip.

**Related section:** 6.3.2–6.3.3 · **Next:** Phase 8.5 (load balancers), Phase 11.6 (the Route 53 mapping)

## Answer 6.4 — Lowering the TTL in advance

**(a)** **Lower the TTL today** (86400 → 60). The reason: records sitting in caches were stored with the
**old TTL** (6.4.2). If you lower it today, over the next 24 hours all the old records in all the caches
expire and are replaced by their new **60-second** versions. When you make the change tomorrow, the world
sees you within 60 seconds. If you had lowered the TTL at the moment of the move, it would have achieved
nothing — the cached copies would still have used the old IP for a whole day.

**(b)** The order on the move day (6.4.3): (1) confirm the new server is ready and tested; (2) change the A
record to the new IP; (3) confirm the change is published **at the source** with `dig @<authoritative-ns>`;
(4) check from several different resolvers (`dig @8.8.8.8`, `dig @1.1.1.1`); (5) watch the logs to see
traffic arriving at the new server; (6) once the transition has settled, raise the TTL back to normal (3600
or 86400).

**(c)** **Not immediately.** Keep it up for at least a few TTLs — in practice a few hours, if possible a
day. The reason: in DNS there is no moment when "everybody has moved" (6.4.3, item 5); there will be
applications that ignore TTLs and long-lived connections. Shut it down **after** traffic in the old server's
logs has dropped to zero — that is the most reliable criterion.

**Related section:** 6.4.1–6.4.3 · **Next:** Phase 8.5 (transitions with an LB), Phase 11.6 (Route 53)

---
---

# Frequently asked questions

**Q1 — Why does DNS use UDP, and what happens if an answer is lost?** Because DNS is typically a
**single-packet question and answer**, and the cost of establishing a handshake is far higher than asking a
rarely lost query again (5.1.2). If no answer comes the client **asks again** or moves to the second
resolver. If the answer does not fit in 512 bytes (large record sets, DNSSEC) DNS falls back to **TCP 53** —
which is why firewall rules must open both UDP and TCP 53, otherwise you see the odd failure "some names
resolve, some do not".

**Q2 — What is the difference between `dig` and `nslookup`?** `dig` gives more detailed, machine-readable
output (the TTL, which server it came from, the time); `nslookup` is older and misleading in some cases.
Also, both **skip** `/etc/hosts` (Answer 6.1). If you want to see the system's real behaviour, use `getent
hosts <name>`.

**Q3 — What is the difference between `NXDOMAIN` and `SERVFAIL`?** **NXDOMAIN** = "this name **does not
exist**" — the authoritative server is saying so definitively. Usually a typo or a deleted record.
**SERVFAIL** = "I **could not get** an answer" — the authoritative server is unreachable, there is a break
in the chain, or DNSSEC validation failed. The first is an answer, the second is a failure (6.5).

**Q4 — Does clearing the cache solve DNS problems?** It can solve the problem on **your own** machine
(`sudo resolvectl flush-caches` 🟡). But you cannot clear other people's caches — the ISP's, your users',
the intermediate resolvers'. There the only way is **waiting**, and how long is determined by the **old
TTL** (6.4.2). So flushing the cache is not a solution but a verification tool: if you get the right answer
after flushing, you have proved the problem was the cache.

**Q5 — Can a name have more than one A record?** Yes, and it is common. The resolver returns them all and
the client usually tries the first; servers rotate the order (round-robin) for a crude form of load
distribution. But this is **not real load balancing**: there is no health checking, a dead IP stays in the
list and the client tries to connect to it (some clients try the next one, some do not). For real
distribution a load balancer is used (Phase 8.5).

**Q6 — Who secures DNS, can an answer be altered?** Classic DNS is unencrypted and can be forged (DNS
spoofing / cache poisoning). There are two layers of defence: **DNSSEC** signs the answers — it proves the
answer was not altered but does not hide the content; **DoH/DoT** (DNS over HTTPS/TLS) encrypts the query —
it hides the content but does not prove the source. The two solve different problems and can be used
together.

**Q7 — Why is the `.2` address the DNS server in the cloud?** Because AWS reserves the first four and the
last address in every subnet (2.3) and gives the second one (like `10.0.0.2`) to the VPC's own resolver.
That is the address you see in an instance's `/etc/resolv.conf`, and it is handed out by DHCP (1.6.1). This
resolver both resolves internet names and reaches **private hosted zone** records (6.3.3) — which is why
internal names resolve only from inside the VPC.

---
---

# Test yourself

Write your answers on paper, then compare them with the key. Target: 14+ out of 18.

## Part A — Definitions and mechanisms

1. Write the four roles in the DNS hierarchy (root, TLD, authoritative, resolver) and **what each one
   knows**.
2. What is the difference between a recursive query and an iterative one? Which one do you make, which does
   the resolver make?
3. What does the dot at the end of `www.example.com.` represent?
4. Write what each of the A, CNAME, NS and MX records returns.
5. What does TTL mean and who sets it?
6. In how many different places can a DNS answer be cached? Name at least four.
7. What is the difference between `NXDOMAIN` and `SERVFAIL`?
8. Why does DNS use UDP, and when does it fall back to TCP?

## Part B — Apply and diagnose

9. `ping 8.8.8.8` works, `ping google.com` does not. What is your diagnosis and your first two commands?
10. Which command shows you the entire resolution chain starting from the root?
11. `dig` gives the right IP but the browser goes to the old server. Write two causes and how to verify.
12. You changed a record; some users see the new IP and some do not. Why?
13. How do you confirm **at the source** that the change was really published?
14. You are changing an IP tomorrow, the TTL is 86400. What should you do today?

## Part C — Reasoning and connections

15. What does the symptom "ping to an IP works, names do not resolve" **prove** to you about Phases 1–5?
16. Why can a CNAME not be used at the apex? How do cloud providers solve it?
17. What happens if the target a subdomain CNAMEs to is deleted — and why is that a **security** problem?
18. Why is switching traffic with DNS risky in a blue/green deployment? What is the better alternative?

---

## Answer key

1. **Root:** knows the TLDs' servers. **TLD (.com):** knows that zone's authoritative servers.
   **Authoritative:** knows the real records (the IP) — the single source of truth. **Resolver:** knows none
   of them but knows **how to ask**, and caches the answer (6.1.2). — 2. **Recursive** = "find me the
   answer" (this is what you ask the resolver). **Iterative** = walking the chain step by step, getting
   "ask the next one" at each step (this is what the resolver does) (6.2.1). — 3. **The root** — the top of
   the hierarchy. It is not normally typed but it is there (6.1.2). — 4. **A** → an IPv4 address; **CNAME**
   → another **name** (an alias); **NS** → that zone's authoritative servers; **MX** → the mail server and
   its priority (6.3.1). — 5. **How many seconds the answer is considered fresh**; **the record's owner**
   sets it on the authoritative server and everyone in the chain obeys (6.4.1). — 6. The browser cache, the
   operating system cache (systemd-resolved), the local/corporate resolver, the ISP resolver — and
   `/etc/hosts` is a kind of permanent hand-written entry as well (6.2.3, 6.4.2). — 7. **NXDOMAIN** = the
   name **does not exist** (a definitive answer, usually a typo/deleted record). **SERVFAIL** = an answer
   **could not be obtained** (the server is unreachable, the chain is broken, or a DNSSEC error) (6.5). —
   8. UDP, because it is a single-packet question and answer and a handshake would be a pointless cost
   (5.1.2). If the answer **does not fit in 512 bytes** (large record sets, DNSSEC) it falls back to **TCP
   53** (Q1).

9. Diagnosis: **DNS**. Commands: `cat /etc/resolv.conf` (or `resolvectl status`) to see who the resolver is;
   `dig @8.8.8.8 google.com` to ask a resolver known to work (6.2.1, 6.5). — 10. **`dig example.com
   +trace`** (6.2.2). — 11. (i) A hand-written entry in `/etc/hosts` → `cat /etc/hosts` or `getent hosts
   <name>`; (ii) the browser/OS cache → refresh the browser, `sudo resolvectl flush-caches`. `dig` cannot
   see them because it **skips** those layers (6.2.3). — 12. Because every resolver's cache filled at a
   **different time**; everyone waits for their own TTL to expire. There is no moment when "everybody has
   moved" (6.4.2). — 13. **`dig @<authoritative-ns> <name>`** — skipping the cache layers and asking the
   source directly (the box in 6.4.2). — 14. **Lower the TTL today** (86400 → 60). Because cached copies
   were stored with the old TTL, lowering it takes one **old TTL** period to take effect (6.4.3).

15. It proves a great deal: the IP configuration is correct (Phase 1), the mask/subnet is correct (Phase 2),
    ARP and L2 work (Phase 3), **routing works and packets come back** (Phase 4), there is no basic blocking
    along the path. So Phases 1–4 are healthy; the problem is only in the **name→IP** step (6.5). — 16.
    Because the standard says a name with a CNAME **can have no other record**, and the apex **must** have
    NS and SOA (6.3.2). Cloud providers solve it with **alias/ALIAS/ANAME** records: the outside is answered
    like an A record while the target's current IP is looked up behind the scenes (6.3.3). — 17. Your name
    becomes unresolvable too, and the reason is **invisible in your own DNS**. It is a security problem
    because the deleted target can be **claimed by someone else** at a cloud provider — and then the traffic
    to your subdomain goes to the attacker (a dangling CNAME / subdomain takeover) (6.3.2). — 18. Because
    your switching speed is **equal to the TTL**: with a TTL of 3600 a rollback takes an hour, and clients
    that ignore TTLs hang on even longer. Better: keep a fixed name (the LB's alias) and switch traffic
    between the **load balancer's target groups** — within seconds and independent of caches (the cloud box
    in 6.4.3, Phase 8.5).

## Scoring

| Correct answers | What it means |
|---|---|
| 16–18 | DNS has settled. You have the "It's always DNS" reflex. You are ready for Phase 7. |
| 13–15 | Good. Read 6.4 (caching and TTL) once more — that is what misleads most in the field. |
| 9–12 | The chain and the cache may be getting mixed up. Do the `dig +trace` and TTL experiments yourself. |
| 0–8 | Walk the phase again. The goal: to say "DNS" the moment you hear "ping to the IP works, names do not resolve". |

**Which section to go back to for a question you missed:**

| Question | Section |
|---|---|
| 1, 3 | 6.1 The hierarchy |
| 2, 8, 10 | 6.2 The resolution chain |
| 9, 15 | 6.2.1 + 6.5 Diagnosis |
| 11 | 6.2.3 The steps before the query |
| 4, 16, 17 | 6.3 Record types |
| 5, 6, 12, 13, 14, 18 | 6.4 Caching and TTL |
| 7 | 6.5 Failure signatures |

---
---

# Closing and the Bridge to Phase 7

## What you carry from this phase

Phase 6 gave you **how a name turns into an address**. You learned why the hierarchy was built distributed —
that nobody knows everything but everybody knows the next one. You saw the recursive/iterative distinction,
that the resolver does all the work. You picked up the record types, the apex restriction on CNAME and why a
dangling CNAME is a security vulnerability. And most practically: you understood that **TTL is a deployment
parameter**, that it determines how fast a change spreads to the world.

The two most durable sentences: **"if `ping 8.8.8.8` works and `ping google.com` does not, the problem is
DNS and every layer below is healthy"** and **"lower the TTL before the change, not after."**

## Where Phase 7 connects

In Phase 1 we said one thing in passing: **private** addresses like `10.0.1.50` cannot be routed on the
internet (1.3.2). Routers drop packets carrying these addresses.

But your computer at home almost certainly has the address `192.168.1.x`. And you can get to the internet.

How? What is more, there are ten devices on the same network and all of them can go out at the same time —
all from behind a single public IP. How do the returning answers know which device they belong to?

Phase 7 is the answer: NAT. You will see private→public translation, port-based multiplexing (PAT), port
forwarding, the difference between a NAT Gateway and an Internet Gateway in the cloud, and tunnelling. And
you will meet an old friend again: **a tunnel header makes the packet bigger, and the MTU black hole from
Phase 5.7 explodes right there.**

> **🤔 Phase output — ask yourself:** Three devices at your home opened `https://example.com` at the same
> time. The source IP of all three was replaced by the router with the **same public IP**. Three answers are
> coming back from the server and all of them have the same destination IP. How will the router deliver
> those three answers to the **right devices**? (Hint: in Phase 1.4.2 we talked about the **four things**
> that make a connection unique — the router still has one field it can change.)
>
> **🧪 Lab 6 idea (all 🟢):** (1) Run `dig example.com +trace` and mark the root → TLD → authoritative
> transitions in the output. (2) Run `dig example.com` twice, 10 seconds apart; see the TTL **going down**.
> (3) Compare with `dig @8.8.8.8 example.com` — why is the TTL different? (4) Run `dig MX gmail.com`, `dig
> NS example.com` and `dig TXT example.com` to see different record types. (5) Find your real resolver with
> `resolvectl status`, then ask it directly with `dig @<that address> example.com`. (6) Ask for a name that
> does not exist (`dig this-name-does-not-exist-12345.com`) and see **NXDOMAIN**.

---

> **Navigation:** [◀ Phase 5 — The Transport Layer](Phase_5_Transport_Layer.md) · **Phase 6** · [Phase 7 — NAT and the Real World ▶](Phase_7_NAT.md)
