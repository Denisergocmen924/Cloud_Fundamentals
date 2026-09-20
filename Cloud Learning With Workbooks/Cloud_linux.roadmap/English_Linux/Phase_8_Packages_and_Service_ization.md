# Phase 8 — Packages, Software and Service-ization

> **Navigation:** [◀ Phase 7 — Networking and Connectivity](Phase_7_Networking_and_Connectivity.md) · **Phase 8** · [Phase 9 — Security and Hardening ▶](Phase_9_Security_and_Hardening.md)

---

## Where we are coming from

At the end of Phase 7 we left you a question: you installed an app with `apt install` but `systemctl
status` says "not found" — which step is missing between "being installed" and "running as a service"? The
answer is exactly this phase's subject. Throughout Phases 5-7 you worked with services that **already
existed**: SSH was listening, nginx was up, systemd was managing them. But how did that software **get onto**
the machine, and how does your own application become one of those "services managed by systemd"? Phase 8
tells you this.

The knowledge of three phases comes together directly here:

- **Phase 5 — services are units.** You saw how a `.service` unit file is started by systemd, its
  `active (running)` state, its log via `journalctl -u` (5.3). In this phase you'll **write** that unit
  file yourself — for your own application.
- **Phase 3 — a process and its death.** You'll understand why a process you throw into the background with
  `nohup python app.py &` is fragile (it dies when the session closes, doesn't come back when it crashes)
  through Phase 3's process model.
- **Phase 7 — a service opens to the network.** The outermost gate of the "I can't connect to the service"
  event was "is the service installed and running." This phase fills in what's behind that gate —
  installation and service-ization.

## The question of this phase

Phase 7 connected the service to the outside world. This phase brings the service onto the machine and
makes it persistent:

> *"How does software get onto a machine, and how do I turn my own application into a real service that
> comes up by itself even after the machine reboots and recovers itself when it crashes?"*

For a cloud engineer these two questions are intertwined. Software entering the machine almost always goes
through a **package manager** (`apt`) — and a package manager is not just a tool that "copies files" but a
system that manages versions, dependencies and signatures; when it breaks ("broken dependency", "repo
unreachable") it halts the whole installation. Running your own application is more than starting a file by
hand: in production an application must start **by itself**, **restart itself** when it crashes, and write
its logs to a **central place**. What provides this is using systemd — which in Phase 5 you saw as a
consumer — now as a **producer**: writing your own `.service` unit.

By the end of this phase you'll establish the distinction "being installed ≠ running as a service";
service-ize an application not through a fragile way like `nohup ... &` but properly for production with a
systemd unit; and understand the cloud's "don't patch the server by hand, rebuild it" (immutable)
philosophy.

---

## By the end of this phase

- You will be able to explain how `apt` (and the underlying `dpkg`) installs a package; and the concepts of
  repository, GPG signature and version pinning
- You will recognize the "broken dependency", "repo unreachable" and "version drift" failures and know
  where to look
- You will be able to write your own Python/FastAPI application as a systemd `.service` unit; and configure
  auto-restart and central logging
- You will be able to concretely explain why `nohup python app.py &` is not fit for production (session
  dependency, no restart, log scatter)
- You will know at the **concept level** what building from source is (`./configure && make && make
  install`); and tell apart when it's needed (and why it usually isn't)
- **Cloud:** you will be able to explain why pre-baking packages into a "baked-in" AMI, reproducibility via
  version pinning, and the "don't patch the server by hand, rebuild it" (immutable) philosophy are the
  foundation of cloud operations

---

## The phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 8.1 | Package managers | `[mechanism]` | How software gets onto the machine: `apt`, `dpkg`, repo, signature |
| 8.2 | Service-izing your own app | `[application]` | **The heart of the phase** — a systemd unit, not `nohup &` |
| 8.3 | Building from source (awareness) | `[skip]` | `./configure && make` — when it's needed |
| 8.4 | The immutable approach | `[concept]` | "Don't patch, rebuild" — the cloud philosophy |
| 8.5 | When this phase breaks | — | Package and service-ization failure signatures |

> **How to work in this phase:** This phase's observation commands are 🟢 (`apt list --installed`,
> `apt-cache policy`, `systemctl status`, `journalctl -u`). The installation and service-ization steps
> include 🟡/🔴: `apt install/remove` changes the system (🟡; `apt` keeps these reversible, but removing a
> package can stop dependent services 🔴). The most instructive experiment: write a simple `hello.service`
> unit, start it with `systemctl`, then deliberately crash it and watch auto-restart kick in via
> `journalctl`. Work on your own test machine — don't experiment with package removal on a production
> machine.

---
---

# 8.1 Package Managers

## 8.1.1 apt and dpkg: how a package is installed `[mechanism]`

The standard way to install software on Linux is the **package manager**. There are two big families: on
Debian/Ubuntu **`apt`** (with **`dpkg`** underneath), on RHEL/Amazon Linux **`dnf`/`yum`** (with **`rpm`**
underneath). This phase focuses on Ubuntu, but the mental model is the same for both — only the command
names change.

Separating the layers matters: **`dpkg`** is the lower layer; it unpacks a single `.deb` file, places its
files into the system (most under `/usr/bin`, `/etc`, `/lib`) and records it. But `dpkg` **does not resolve
dependencies** — if a package needs other packages it won't find and install them itself. That's where
**`apt`** is the upper layer: it resolves what a package needs (the dependency tree), downloads everything
required from a **repository**, and calls `dpkg` in the right order. So when you say `apt install nginx`:
apt computes nginx's dependencies, downloads them all, installs them in order via dpkg.

> **🔧 See it on your machine** 🟢 — read installed packages and a package's state
>
> ```
> $ apt list --installed | wc -l          # how many packages installed
> 1847
> $ apt-cache policy nginx                # which version installed, which available, from where
>   Installed: 1.18.0-6ubuntu14.4
>   Candidate: 1.18.0-6ubuntu14.4
>   Version table: ...
> $ dpkg -L nginx | head                  # which files this package placed
> /usr/sbin/nginx
> /etc/nginx/nginx.conf
> ...
> ```
>
> `apt-cache policy` gives you three critical pieces of information: the **installed** version, the
> **candidate** (newest installable) version, and which **repo** it comes from. `dpkg -L` shows exactly
> what a package placed on disk — the answer to "which package did this command come from" or "where is the
> config file."

## 8.1.2 Repository, GPG signature and version pinning `[concept]`

Where do packages come from? A **repository** (repo) is a server holding packages and their metadata
(version, dependencies). Your machine knows which repos to trust from `/etc/apt/sources.list` and the files
under `/etc/apt/sources.list.d/`. `apt update` refreshes those repos' **package list** (not the packages —
just the "what's available" information); `apt upgrade` upgrades installed packages to newer versions.

Two concepts are critical for security and stability:

- **GPG signature.** Packages from a repo are cryptographically signed; apt verifies with the signature that
  the package it downloaded really came from that repo and wasn't altered in transit. Errors like
  "repository is not signed" are a break in this chain of trust — apt refuses to install a package it can't
  verify (the correct behavior).
- **Version pinning.** **Locking** a package to a specific version. In production you don't want "every `apt
  upgrade` to change a service's version" — an unexpected version can break an application. With pinning you
  deliberately keep a package fixed; it's the basis of reproducibility (8.4).

> **⚠️ Common misconception: "`apt update` updates my packages."**
>
> No — this is the most commonly confused pair. **`apt update`** only refreshes the repos' **package
> list**: it downloads the information of "which packages have which new versions available," and updates
> not a single package. What actually upgrades packages is **`apt upgrade`**. The right order is always
> `apt update` (refresh the list) → then `apt install`/`apt upgrade` (install/upgrade). A common cause of
> the "can't find the package" error is trying to work with an old list without having run `apt update`.

> **🤔 Think 8.1** — On a server you say `apt install new-tool` but get "Unable to locate package
> new-tool" — even though you're sure this tool exists. On the same server someone else added this repo to
> `sources.list.d` yesterday. Which single command should you have run **first**, and what exactly does
> that command refresh (the packages, or something else)?
>
> *(Answer: at the end of the phase)*

---
---

# 8.2 Service-izing Your Own App

## 8.2.1 Why `nohup python app.py &` is not fit for production `[application]`

The fastest way to run your own app looks like this:

```
$ nohup python app.py &        # throw it to the background, don't die when the session closes
```

This is fine for a test but in production it has **three fatal shortcomings** — and each is explained by
the knowledge of previous phases:

1. **Doesn't come back when it crashes (Phase 3).** If the process hits an error and dies (segfault,
   unhandled exception, the OOM killer — Phase 4!), there's no one to restart it. Your app dies silently at
   three in the morning and stays down until someone notices in the morning.
2. **Doesn't start on boot (Phase 5).** If the machine reboots (kernel update, instance replace in the
   cloud) the process you started with `nohup` doesn't come back. There's no "start on boot" mechanism like
   `systemctl enable`.
3. **Its logs are scattered.** Output piles into a file called `nohup.out`; it isn't rotated, isn't
   central, can't be read together with other services via `journalctl -u`. Your single-window log-reading
   reflex from Phase 5 doesn't work here.

In short: `nohup ... &` **starts** a process but doesn't **manage** it. Production wants a managed service —
and what manages it is systemd. The figure below places the two ways side by side: the same app, two
different fates.

![Figure 8.1 — From a script to a managed service: the same app (app.py) can be run two ways. On the left `nohup python app.py &` — a process starts but nobody manages it: it stays down when it crashes, doesn't start on boot, its logs are scattered, it dies when the session closes. On the right `myapp.service` (systemd unit) — a managed service: auto-restarts with `Restart=on-failure`, starts on boot with `systemctl enable`, writes central logs to journald, is independent of the session. Lesson: being installed on disk is not the same as running as a managed service.](../diagrams/png/lx-8-01-script-to-service.png)

## 8.2.2 Writing a systemd unit: turn your app into a real service `[application]`

In Phase 5 you saw systemd as a **consumer** (you read units others wrote). Now you'll be a **producer**:
you'll write a `.service` unit file for your own application. The file goes under `/etc/systemd/system/`;
e.g. `/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My FastAPI app
After=network.target

[Service]
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/venv/bin/uvicorn app:app --host 0.0.0.0 --port 8000
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Each line closes one shortcoming from 8.2.1: `Restart=on-failure` → systemd auto-restarts on crash
(shortcoming 1 solved); `WantedBy=multi-user.target` + `systemctl enable` → starts on boot (shortcoming 2
solved); stdout/stderr go to journald automatically → `journalctl -u myapp` (shortcoming 3 solved).
`After=network.target` is a nod to Phase 7: don't start before the network is ready. And `--host 0.0.0.0`
is Phase 7.4.2's lesson: be reachable from outside.

> **🔧 See it on your machine** 🟡 — register a service, start it, add it to boot
>
> ```
> $ sudo systemctl daemon-reload          # 🟡 make systemd read the new/changed unit file
> $ sudo systemctl start myapp            # 🟡 start it now
> $ sudo systemctl enable myapp           # 🟡 have it start on boot too
> $ systemctl status myapp                # 🟢 see the state (is it active running)
> $ journalctl -u myapp -f                # 🟢 follow its log live
> ```
>
> Critical reflex: **every time you change** a unit file you must `sudo systemctl daemon-reload` —
> otherwise systemd still uses the old definition. This is the number-one cause of the "I fixed the unit
> but nothing changed" failure. `start` and `enable` are different things (Phase 5.3): `start` runs it now,
> `enable` adds it to boot — you need both.

> **💡 Cloud connection — cloud-init and a service's first birth:** In the cloud, when an instance first
> boots, **cloud-init** runs (Phase 5): during boot it can install packages, place files, and install and
> `enable` your `myapp.service` — all without connecting by hand. So the whole flow of "start the instance,
> 30 seconds later your app is live on `0.0.0.0:8000`" is automatic. This is the basis of the next section
> (8.4 immutable): instead of logging into the server by hand and installing the app, you write the
> installation as a recipe (cloud-init/AMI) and the machine installs itself.

> **🤔 Think 8.2** — You wrote an application as `myapp.service`, said `systemctl start myapp`, and it ran.
> Then you fixed the `ExecStart` line in the unit file and said `systemctl restart myapp` again but the
> change **didn't take effect** — the service still runs with the old command. (a) Which single command did
> you skip? (b) Which "running state vs persistent definition" distinction from Phase 7 (e.g. `ip addr` vs
> netplan, `mount` vs fstab) is this failure a sibling of?
>
> *(Answer: at the end of the phase)*

---
---

# 8.3 Building from Source (Awareness)

## 8.3.1 `./configure && make && make install`: when it's needed `[skip]`

Sometimes a piece of software isn't in the package repository, or you're offered a very old version of it;
then **building from source** comes up. The classic trio:

```
$ ./configure        # inspect your system, prepare build settings
$ make               # compile the source (produce a binary)
$ sudo make install  # place the produced files into the system
```

Knowing this at the concept level is enough — we're not going deep in this phase (`[skip]`). What's
critical is telling apart **when it's needed and why it usually isn't**. Building from source brings these
costs: because it stays **outside** the package manager it doesn't show in `apt`'s records (`dpkg -L` won't
find it), doesn't auto-update, and the files that `make install` spreads all over the disk are hard to
remove cleanly. So the modern preference order is: **package first** (`apt`), else **add an official repo**,
else a **prebuilt binary** or a **container**, and as a last resort **build from source**.

> **❓ Question that comes to mind: "If I can build from source, why always use a package?"**
>
> Because a package manager gives you not just "installation" but a **lifecycle**: version tracking,
> dependency resolution, security updates, clean removal, and signature-verified trust. Building from source
> means shouldering all of these by hand — dozens of hand-compiled tools on a server become, a few months
> later, a junkyard where nobody knows who put which version where. The package manager is discipline;
> building from source is an exception you reach for only when it's really needed.

---
---

# 8.4 The Immutable Approach

## 8.4.1 "Don't patch the server by hand, rebuild it" `[concept]`

Everything so far (install a package, write a unit, edit config) was about touching a server **by hand**.
The mature form of cloud operations reverses this habit: it sees touching a server by hand as a **source of
failure** and puts **reproducibility** in its place. This is called **immutable infrastructure**.

The idea is this: instead of setting up a server by hand once and then tweaking it for months, you write
the whole setup as a **recipe** — which packages, which versions (pinning!), which unit files, which config
— and from that recipe you produce a pre-baked **image** (AMI). When you need a new machine you start from
this image; the machine comes correctly configured from the very first second. If something breaks, you
**don't repair it by hand** — you throw it away and start a new one from the image.

Two concepts make this possible, and both come from this phase:

- **Version pinning (8.1.2).** If every package in the recipe is locked to a fixed version, the same recipe
  produces the **exact same** machine today and three months from now. Without pinning there's no immutable
  — if "the same recipe" pulls different versions each time, reproducibility is a lie.
- **Service-ization (8.2).** If your app is written as a systemd unit in the recipe, the machine born from
  the image `enable`s and starts it by itself — no manual `start` needed.

> **💡 Cloud connection — baked AMI and "cattle, not pets":** The cloud counterpart of this philosophy is
> the **baked-in AMI**: your app, its dependencies and config are pre-"baked" into a machine image. An Auto
> Scaling group starts as many identical instances as it wants from this AMI. The industry's slogan:
> servers should be **"cattle, not pets"** — not pets you name and hand-feed and treat when they get sick,
> but numbered, identical herd animals you replace without hesitation when one breaks. The sentence "this
> server has a special setting, don't ever delete it" should never be heard in an immutable infrastructure
> — every setting is in the recipe, every machine is disposable. In Phase 12 we'll fully merge this
> philosophy with cloud architecture.

---
---

# 8.5 When This Phase Breaks — Package and Service-ization Failure Signatures

This phase's failures split into two groups: **package** (software can't get onto the machine) and
**service-ization** (software got on but doesn't run properly). The table below takes the symptom to the
right layer:

| Symptom | Likely cause | Where to look | Related section |
|---|---|---|---|
| `apt install X` → "Unable to locate package" | `apt update` not run / no repo | `apt update`, `sources.list` | 8.1.2 |
| `apt install` → "unmet dependencies" / broken dependency | Dependency conflict / half-finished install | `apt -f install`, `dpkg --configure -a` | 8.1.1 |
| `apt update` → "repository is not signed" | Missing/invalid GPG key | Add the repo's GPG key | 8.1.2 |
| Package version changed unexpectedly | No pinning, `apt upgrade` bumped it | `apt-cache policy X`, add a pin | 8.1.2 |
| Installed with `apt install` but `systemctl status` "not found" | Package defines no systemd service | `dpkg -L X` (is there a unit file) | 8.2.2 |
| Fixed the unit but the change has no effect | `daemon-reload` not run | `sudo systemctl daemon-reload` | 8.2.2 |
| App doesn't come back when it crashes | No `Restart=` / started with `nohup &` | Add `Restart=on-failure` to the unit | 8.2.1 |
| App doesn't start on boot | `systemctl enable` not run | `systemctl enable X`, `is-enabled` | 8.2.2 |

> **The lesson from this table:** This phase's two big distinctions lie beneath every failure. The first is
> **`apt update` ≠ `apt upgrade`**: one refreshes the list, the other upgrades packages — confusing them is
> the origin of the "can't find the package" and "unexpected version" failures. The second, one level up
> from the line carried over from Phase 7: **being installed ≠ running as a service**. Installing a package
> places files on disk; what turns it into a service managed by systemd, one that recovers on crash and
> starts on boot, is the unit you wrote and `enable`. And remember: every time you change a unit,
> `daemon-reload` — this is the same as Phase 7's `mount` vs fstab, `ip addr` vs netplan distinction: the
> **running state** and the **persistent definition** are separate layers.

---
---

# Phase 8 — Answers to the think questions

## Answer 8.1 — First `apt update`; it refreshes the package list (not the packages)

You should have run **`sudo apt update`** first. The newly added repo was written to `sources.list.d` but
your machine is still **unaware** of the packages in that repo — because the package list it has is old.
`apt update` refreshes exactly this: it re-downloads the **package metadata** of all configured repos
(which package, which version, which dependency is available). It updates/installs not a single package —
it only refreshes the "what's available" information. After `update`, `apt install new-tool` now finds the
package. The right order is always: **`apt update` → then `install`/`upgrade`**.
**Related section:** 8.1.2 · **Continues in:** the 8.5 failure table (row 1).

## Answer 8.2 — You skipped `daemon-reload`; it's in the "running state vs persistent definition" family

(a) You skipped **`sudo systemctl daemon-reload`**. You changed the unit file on disk but systemd is still
using the **old definition** it read into memory earlier; `restart` restarted with that old definition.
`daemon-reload` tells systemd "re-read the unit files from disk"; after it, `restart` uses the new
`ExecStart`. The right order: fix the file → `daemon-reload` → `restart`. (b) This is an exact sibling of
the **running state vs persistent definition** distinction from Phase 7 (and Phase 6): systemd's active
in-memory definition = the "running state"; the `.service` file on disk = the "persistent definition." Just
like `ip addr` (running) vs netplan (persistent), or `mount` (running) vs fstab (persistent) — changing the
disk doesn't automatically update the running state; there's a "re-read" step in between (`daemon-reload` /
`mount -a` / netplan apply).
**Related section:** 8.2.2 · **Continues in:** the 8.5 failure table (row 6).

---
---

# Phase 8 — Frequently asked questions

**Q1 — What's the difference between `apt update` and `apt upgrade`?** `apt update` refreshes the repos'
**package list** (changes no package); `apt upgrade` **upgrades** installed packages to newer versions. The
right order is always `update` → then `install`/`upgrade` (8.1.2).

**Q2 — Should I use `apt` or `dpkg`?** Almost always `apt`. `apt` resolves dependencies and downloads from
repos; `dpkg` only installs a single `.deb` file and doesn't resolve dependencies. You usually use `dpkg`
just to query (`dpkg -L`, `dpkg -l`) (8.1.1).

**Q3 — How do I find which package a command came from?** If installed, `dpkg -S $(which command)`; if not
yet installed, `apt-file search command` (8.1.1).

**Q4 — Why isn't `nohup python app.py &` enough?** Three shortcomings: doesn't come back when it crashes,
doesn't start on boot, its logs are scattered. A systemd unit solves all three (`Restart=`, `enable`,
journald) (8.2.1).

**Q5 — I changed the unit file but it has no effect, why?** You didn't run `sudo systemctl daemon-reload`.
systemd doesn't automatically see the change on disk; `daemon-reload` is mandatory after every unit change
(8.2.2).

**Q6 — The difference between `systemctl start` and `enable`?** `start` starts the service **now** (won't
come back after reboot); `enable` marks it to **start on boot**. For a persistent service you need both
(8.2.2, Phase 5.3).

**Q7 — What is immutable infrastructure, briefly?** Instead of patching the server by hand, write the setup
as a recipe (pinned packages + units) and start from a pre-baked image (AMI); instead of repairing what
breaks, throw it away and rebuild. "Cattle, not pets" (8.4.1).

---
---

# Phase 8 — Test yourself

Write your answers on paper, then compare against the answer key. Target: 14+ out of 18.

## Section A — Definition and mechanism

1. Explain the layer difference between `apt` and `dpkg`: which resolves dependencies, which installs a
   single `.deb`?
2. What exactly do `apt update` and `apt upgrade` do? What is the right order?
3. What is a repository, and what does the GPG signature guarantee?
4. What is version pinning and why does it matter in production?
5. Name the three production shortcomings of `nohup python app.py &`; tie each to a previous phase.
6. What do the `Restart=on-failure`, `WantedBy=multi-user.target` and `ExecStart` lines do in a systemd
   unit?
7. What is the difference between `systemctl start` and `systemctl enable`?
8. What is the core idea of immutable infrastructure? What does "cattle, not pets" convey?

## Section B — Apply and diagnose

9. `apt install toolname` → "Unable to locate package". What's the first command you run and why?
10. You installed an app with `apt install` but `systemctl status app` says "not found". What does this
    mean, and how do you verify it?
11. You fixed `ExecStart` in the unit file, ran `systemctl restart`, but the change has no effect. Which
    command did you skip?
12. Your app crashed at night and stayed down until morning. Which single line do you add to the unit to
    prevent this?
13. After a reboot your app doesn't come up. With which command do you check the state, and with which do
    you fix it?
14. How do you find which package a command came from, and the files that package placed on disk?

## Section C — Reasoning and connection

15. Explain "being installed ≠ running as a service" with an example. Which distinction from Phase 7 is this
    one level up from?
16. How does the need for `daemon-reload` fall into the same family as Phase 6's `mount` vs fstab and Phase
    7's `ip addr` vs netplan distinctions? Write the shared principle in one sentence.
17. Which two concepts of this phase (from 8.1 and 8.2) does immutable infrastructure rest on, and why are
    both required?
18. When an instance first boots, which steps of this phase (package + service) does a cloud-init recipe
    automate?

---

## Answer key

1. `apt` is the upper layer: resolves dependencies, downloads from repos; `dpkg` is the lower layer:
   installs a single `.deb`, doesn't resolve dependencies (8.1.1). — 2. `apt update` refreshes the repos'
   package list (changes no package); `apt upgrade` upgrades installed packages; order: `update` →
   `upgrade`/`install` (8.1.2). — 3. A repo is a server holding packages and their metadata; the GPG
   signature guarantees the package really came from that repo and wasn't altered in transit (8.1.2). — 4.
   Locking a package to a specific version; in production it prevents unexpected version changes (and
   breakage), the basis of reproducibility (8.1.2). — 5. (1) Doesn't come back when it crashes (Phase 3/4),
   (2) doesn't start on boot (Phase 5), (3) logs scattered (Phase 5 journald); systemd solves all three
   (8.2.1). — 6. `Restart=on-failure` auto-restarts on crash; `WantedBy=multi-user.target` (+enable) starts
   on boot; `ExecStart` is the command to run (8.2.2). — 7. `start` runs it now (not persistent); `enable`
   adds it to boot; for a persistent service you need both (8.2.2). — 8. Don't patch the server by hand,
   start from a pre-baked image from a recipe, throw away and rebuild what breaks; "cattle, not pets" =
   servers are not named pets but disposable herd (8.4.1).

9. `sudo apt update` — the machine's package list is old; for it to see the newly added repo/new versions
   you must refresh the list first (8.1.2). — 10. The package **defined no** systemd service (not every
   package does); with `dpkg -L app | grep '\.service'` you check whether it placed a unit file; if not,
   you write your own unit (8.2.2). — 11. `sudo systemctl daemon-reload` (8.2.2). — 12. `Restart=on-failure`
   (with `RestartSec=` if you like) (8.2.1). — 13. Check: `systemctl is-enabled app` / `systemctl status`;
   fix: `sudo systemctl enable app` (8.2.2). — 14. `dpkg -S $(which command)` (which package); `dpkg -L
   package` (the files it placed) (8.1.1).

15. E.g.: `apt install postgresql` places files on disk (installed) but it isn't "running" until you start
    and `enable` the service yourself, or write a unit for your own app. This is one level up from Phase 7's
    "existing ≠ reachable" (process listening but not bound to `0.0.0.0`): here it's "existing on disk ≠
    running as a managed service" (8.5, Phase 7.4.2). — 16. In all of them, changing the **persistent
    definition on disk** doesn't automatically update the **in-memory/running state**; a "re-read" step is
    needed in between: `daemon-reload` / `mount -a` / netplan apply. Shared principle: the running state and
    the persistent config are separate layers (8.5). — 17. Version pinning (8.1.2) → the same recipe
    produces the exact same machine; and service-ization (8.2) → because the app is in the recipe as a unit,
    the machine born from the image starts it by itself. Without pinning reproducibility is a lie, without a
    unit the app won't come up automatically (8.4.1). — 18. Package side: installs packages via `apt update`
    + `apt install` (pinned); service side: places the unit file, brings the app up via `daemon-reload` +
    `enable` + `start` — all without connecting by hand (8.2.2, 8.4.1).

## Scoring

| Correct | What it means |
|---|---|
| 16-18 | You grasped bringing software onto the machine and service-izing it. You're ready for Phase 9. |
| 13-15 | Good. Re-read the sections of the questions you missed (especially 8.1.2 and 8.2.2). |
| 9-12 | The basics are there but fragile. Work on the `apt update`/`upgrade` distinction and unit writing. |
| 0-8 | Walk the phase again; write and start a simple `hello.service` on a test machine, crash it, watch it. |

Missed question → the section to return to:

| Question | Section |
|---|---|
| 1, 2, 3 | 8.1 Package managers |
| 4, 17 | 8.1.2 Pinning + 8.4 Immutable |
| 5, 12 | 8.2.1 Why not nohup |
| 6, 7, 11, 13 | 8.2.2 Writing a unit |
| 8, 18 | 8.4 Immutable / cloud-init |
| 9, 14 | 8.1.1 apt/dpkg |
| 10, 15 | Installed ≠ service (8.2.2, 8.5) |
| 16 | Running state vs persistent definition (8.5) |

---
---

# Phase 8 — Closing and Bridge to Phase 9

## What you carry from this phase

Phase 8 gave you the **lifecycle** of software: how it gets onto the machine (`apt`/`dpkg`, repo,
signature, pinning), and how your app becomes a real service (systemd unit, `Restart=`, `enable`,
journald). You acquired two lasting distinctions: **`apt update` ≠ `apt upgrade`** (list vs upgrade) and
**being installed ≠ running as a service**. And with the `daemon-reload` reflex you saw the "running state
vs persistent definition" principle for a third time (Phase 6 fstab, Phase 7 netplan, Phase 8 unit) — it's
a pattern now. Finally you met the immutable philosophy: don't patch by hand, build from a recipe, "cattle,
not pets."

## Where Phase 9 connects to this

Throughout Phase 8 we always did **more**: install a package, open a service, bring the app up. Phase 9
asks the opposite: **what should I restrict?** Every package you install, every port you open, every
permission you grant is an **attack surface**. Phase 9 (Security and Hardening) teaches deliberately
narrowing that surface: the principle of least privilege, SSH/network hardening (the continuation of Phase
7), AppArmor/SELinux (a second lock above DAC permissions — the "permissions are right but it's still
blocked" failure), and managing secrets not on disk but in IAM/Secrets Manager. In Phase 8 you bound a
service to `0.0.0.0`; Phase 9 asks "and who runs this service, with what privilege, and how much damage can
it do if it's compromised?" Building and protecting are two sides of the same coin.

> **🤔 Phase output — ask yourself:** In Phase 8 you wrote `User=appuser` in the unit file — to run the app
> as a limited user instead of root. Before moving to Phase 9, think: if this app were compromised, what
> could an attacker do if it ran as `User=root`, and what can't they with `User=appuser`? This is the very
> "principle of least privilege" of Phase 9 — you actually already made a hardening decision back in Phase
> 8.
>
> **🧪 Lab 8 idea (on your own test instance):** (1) See how many packages are installed with `apt list
> --installed | wc -l`; read version/repo info with `apt-cache policy nginx`. (2) Write a simple script
> (e.g. a loop printing the date every 5 seconds), put it as a `hello.service` unit into
> `/etc/systemd/system/`. (3) Apply the chain `daemon-reload` → `start` → `enable` → `status` →
> `journalctl -u hello -f`. (4) Deliberately crash the script (`exit 1`), watch systemd bring it back with
> `Restart=on-failure` via `journalctl`. (5) Start the same thing with `nohup ./script &`, close the
> session, come back — see the difference for yourself. These steps gather this phase's "installed ≠
> service" reflex into your hands.

---

> **Navigation:** [◀ Phase 7 — Networking and Connectivity](Phase_7_Networking_and_Connectivity.md) · **Phase 8** · [Phase 9 — Security and Hardening ▶](Phase_9_Security_and_Hardening.md)
