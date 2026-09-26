# Checkpoint Quiz 1 — Phases 0–2: Model, Shell, and Access

> **Navigation:** [◀ Phase 2 — Users, Permissions and Identity](Phase_2_Users_and_Permissions.md) · **Checkpoint Quiz 1** · [Phase 3 — Processes and Resources ▶](Phase_3_Processes_and_Resources.md)

---

## What does this quiz measure?

The "Test yourself" tests at the end of each phase probe a single phase: *"Did you understand this
phase?"* A checkpoint quiz measures something **different**: *"Can you connect these three phases to
each other?"*

On a real Linux server no problem stays inside one phase. When you see "Permission denied" you use
knowledge from three phases at once: **where** the problem arose (Phase 0 — the kernel/user-space
boundary, the syscall), **which tool** you're looking with (Phase 1 — the shell, pipes, streams),
and **who** is trying to reach **what** (Phase 2 — identity, permission class). The questions in
this quiz mostly can't be answered from a single phase — which is exactly why they're here.

**How to work through it:**

- Solve it with pen and paper; don't move on before writing your answer.
- The answer key states which **intersection of phases** each question sits at — a question you miss
  points not to a phase but to the **bridge** between two phases.
- There are 21 questions: Section A (connection reasoning, 1–9), Section B (a scenario, 10–15),
  Section C (reading commands and output, 16–21).
- Target time: ~1 hour. But time doesn't matter; what matters is being able to justify in one
  sentence *why* each answer is what it is.

> **🤔 Before you start:** Complete this sentence in your own words: *"Why does `sudo echo hello >
> /etc/test.txt` give `Permission denied`, when the one writing the file is root?"* This single
> question contains all three phases. Set your answer aside; we'll see it in Question 4.

---

# Section A — Connection reasoning (1–9)

Each question here asks you to combine knowledge from at least two phases. Give a short but
justified answer.

**1.** How do the "everything is a file" principle (Phase 0) and the permission model (Phase 2) meet
in the `/proc/1/environ` file? Why can only root read this file, while `status` under `/proc/1/` is
readable by most users?

**2.** A program's `open("/etc/shadow", O_RDONLY)` call returns `EACCES`. **Who** issues this
refusal — the shell, the program, or the kernel? Relate your answer to the user-space / kernel-space
boundary from Phase 0.

**3.** In the pipeline `cat /var/log/syslog | grep error`, if `cat` gives `Permission denied`, what
does `grep` see? Do the two commands run with the same identity, and if you wrote `sudo cat ... |
grep ...`, which side would be root?

**4.** Why does `sudo echo hello > /etc/test.txt` give `Permission denied`? Who performs the
redirection (`>`), and with which identity? Write **two** different commands that do the job
correctly.

**5.** In Phase 1, `which` finds the first matching executable in PATH. What would happen if an
attacker could get your `~/bin` directory prepended to the **front** of PATH and put a fake `ls`
there? Why is this also a Phase 2 (identity/permission) question?

**6.** Why is `/usr/bin/passwd` both in `/usr/bin` per the FHS (Phase 1) and setuid (Phase 2)?
Connect these two facts in a single sentence: "how does an ordinary user update a file they can't
even read?"

**7.** In `ls -l /etc/shadow` the first character is `-`, the permissions are `rw-r-----`, the group
is `shadow`. Which principle of Phase 0, which tool of Phase 1, and which three concepts of Phase 2
does this single line show at once?

**8.** The form `command 2>/dev/null` is used in Phase 1 to swallow stderr. Why can this habit be
dangerous while hunting a permission problem (`Permission denied` is a stderr message)? Relate it to
Phase 2's decision tree.

**9.** `adduser` on Ubuntu, `useradd` on Amazon Linux (Phase 2) — this difference is really a direct
consequence of which Phase 0 concept (distro family)? Does it come from the same root as the `apt`
vs `dnf` difference?

---

# Section B — Scenario: first login to a new server (10–15)

> You've SSH'd into a freshly created Ubuntu 24.04 EC2 instance as the `ubuntu` user. There's an
> application on it: `/opt/app/run.sh`, writing its logs under `/var/log/app/`. The six questions
> below are what happens to you on this server, in order.

**10.** You ran `ssh ubuntu@<ip>` but got `Permission denied (publickey)`. Which Phase 2 mechanism
does this message point to, and how is the permission of `~/.ssh/authorized_keys` related to it?
(Hint: StrictModes.)

**11.** You got in. You ran `docker ps` and got `permission denied while trying to connect to the
Docker daemon socket`. You ran `sudo usermod -aG docker ubuntu` but in the same terminal it's still
refused. Why, and what's your fix without logging out and back in? Which Phase 2 principle (when is
identity fixed) is this a consequence of?

**12.** You ran `cat /var/log/app/app.log` and got `Permission denied`. `ls -l` shows
`-rw-r----- 1 root adm`. You're not in the `adm` group. Which permission class did the kernel look
at to refuse — owner, group, or other? Write two different solutions (one for lasting access, one
one-off).

**13.** `/opt/app/run.sh` is `755` and you're not its owner but you have read permission. You ran
`./run.sh` and got `Permission denied`; `bash /opt/app/run.sh` worked. `/opt` is mounted on a
separate disk. What's the likely cause, and with which single command do you confirm it? (This
question combines Phase 1's file system with Phase 2's execute bit.)

**14.** To fix the app a colleague suggests `sudo chmod -R 777 /opt/app`. Why is this a bad idea from
both a Phase 2 (permission model) and a Phase 0 (every process is on this machine) standpoint? Name
at least two concrete harms.

**15.** To find the problem's log you ran `sudo journalctl -u app | tail -50` and the output opened
in a pager (`less`). Why can this be a security problem for a user who's been given limited `sudo`
privilege? (Connect it to the shell-escape idea from Phase 2.)

---

# Section C — Reading commands and output (16–21)

For each of the outputs below, read the **single line** that matters and answer the question.

**16.** What does the difference between these two outputs prove?

```
$ id
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),27(sudo)
$ id ubuntu
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),27(sudo),988(docker)
```

**17.** In `ls -l` output a line reads: `lrwxrwxrwx 1 root root 7 ... /bin -> usr/bin`. Which Phase 0
concept (file types) and which Phase 1 fact (what `/bin` is in the modern FHS) does this line show
at once? Say why the permissions (`rwxrwxrwx`) are misleading here.

**18.** `stat -c '%A %U:%G %n' /tmp` outputs: `drwxrwxrwt root:root /tmp`. What is the trailing `t`,
which concrete behavior does it provide, and why is it essential in a world-open directory like
`/tmp`?

**19.** `findmnt -T /opt/app/run.sh` outputs:

```
TARGET SOURCE       FSTYPE OPTIONS
/opt   /dev/nvme1n1 ext4   rw,nosuid,nodev,noexec,relatime
```

Which word in this output explains the `./run.sh` refusal from Question 13? In one sentence, say why
`bash run.sh` still works.

**20.** `namei -l /home/ubuntu/site/index.html` outputs:

```
f: /home/ubuntu/site/index.html
drwxr-xr-x root   root   /
drwxr-xr-x root   root   home
drwxr-x--- ubuntu ubuntu ubuntu
drwxrwxr-x ubuntu ubuntu site
-rw-r--r-- ubuntu ubuntu index.html
```

Nginx (the `www-data` user) can't read this file. Which **line** places the barrier, and why doesn't
the file's own permission (`-rw-r--r--`, world-readable) save the situation?

**21.** What does the following command chain do, and which phase does each piece belong to?

```
$ ps aux | grep nginx | awk '{print $1}' | sort -u
```

What does the output containing both `root` and `www-data` tell you from a Phase 2 standpoint (why
does nginx run processes under two different identities)?

---

## Answer key

Each answer ends with which **intersection of phases** the question sits at.

**1.** Both are consequences of "everything is a file": `/proc/<pid>/environ` and `status` present
in-memory process information **as if it were a file** (Phase 0). But these virtual files also have
permissions (Phase 2): `environ` carries a process's environment variables (which may hold passwords
or tokens), so it is `0400` and owned by the process's owner — since PID 1 belongs to root, only
root reads it. `status` is more harmless metadata, so it's open to everyone. · *Phase 0 × Phase 2*

**2.** The **kernel** issues the refusal. `open()` is a syscall (Phase 0); the user program makes the
request, but the permission check happens not in user space but at the boundary the call crosses
into kernel space. The shell isn't in the picture — it launched the program and stepped aside. The
program only receives the `EACCES` errno; it doesn't make the decision. · *Phase 0 × Phase 2*

**3.** `grep` sees **empty input** (or nothing) and finds no match; `cat`'s `Permission denied`
message goes to stderr, not the pipe — so it never reaches `grep`, it lands on the terminal. The
pipe carries only stdout (Phase 1). Both commands run with **your** identity (Phase 2). If you wrote
`sudo cat ... | grep ...`, **only `cat`** would be root; `grep` would still be you — because `sudo`
elevates only the command that is its argument. · *Phase 1 × Phase 2*

**4.** The **shell** performs the redirection (`>`), not the command — and the shell runs with
**your** identity (Phase 1 + Phase 2). `sudo` makes only `echo` root; but the `>` operator that
opens the file is still you, and you have no write permission on `/etc`. The correct ways:
(a) `echo hello | sudo tee /etc/test.txt` — `tee` runs as root and opens the file.
(b) `sudo sh -c 'echo hello > /etc/test.txt'` — puts the redirection inside a shell that root runs.
· *Phase 1 × Phase 2*

**5.** When you type a command (`ls`) the shell scans PATH from the start; if `~/bin` is at the
front, the fake `ls` runs — this is **PATH hijacking** (Phase 1). It's a Phase 2 question because
that fake `ls` runs **with your identity**, so it can read anything you can read, and if you type
`sudo ls` it becomes root. Privilege depends on the identity of the running process — not on where
the file came from. · *Phase 1 × Phase 2*

**6.** `/usr/bin` is the FHS home of "secondary commands for all users" (Phase 1); `passwd` goes
there because it's a tool everyone needs to run. The setuid bit (Phase 2) then solves this: when
`passwd` runs, its **effective UID rises to root**, so while an ordinary user can't read
`/etc/shadow` themselves, they can update their own line through `passwd`. In one sentence: *the user
doesn't reach the file, but a trusted program that opens the file with root's privilege.* ·
*Phase 1 × Phase 2*

**7.** First character `-`: an ordinary file — Phase 0's "everything is a file" and file-type
concept. `ls -l`: Phase 1's fundamental observation tool. `rw-r-----` + `root` owner + `shadow`
group: Phase 2's three concepts — permission bits (rwx), ownership (user), and group ownership. One
line, the intersection of three phases. · *Phase 0 × Phase 1 × Phase 2*

**8.** `2>/dev/null` throws stderr away (Phase 1). It's dangerous while hunting a permission problem
because `Permission denied` messages are written to **stderr** — silence them and you lose the most
important clue of the Phase 2 decision tree (on which path, with which errno, was it refused). `find
/ -name x 2>/dev/null` is common for reducing noise, but while diagnosing you need to **look** at
stderr. · *Phase 1 × Phase 2*

**9.** Yes, the same root: the distro **family** (Phase 0). Ubuntu is in the Debian family → `apt` +
`adduser`; Amazon Linux/RHEL is in the Red Hat family → `dnf` + `useradd`. The difference in package
manager and user tools is a consequence of different ecosystems built on top of the same kernel
(Linux); the question "which distro family" answers both at once. · *Phase 0 × Phase 2*

**10.** `Permission denied (publickey)`: the server couldn't find a public key matching your private
key in `~/.ssh/authorized_keys`, or found one but tripped on the **StrictModes** check (Phase 2,
2.5.1). sshd requires `.ssh` to be `700` and `authorized_keys` to be `600` (and the home directory
not writable by others); if these are loose it refuses even a valid key. So permission that's *too
loose* also blocks login. · *Phase 2 (× cloud-init, bridge to Phase 5)*

**11.** The group list is fixed at login and inherited through `fork` (Phase 2, 2.1.1); the running
shell has no knowledge of the new `docker` group written after `usermod`. Fix: `newgrp docker` (a new
group context in the current shell) or a full logout/login. `id` shows the old list, `id ubuntu` the
new — the difference is exactly the source of the problem. · *Phase 2, Answer 2.1*

**12.** The kernel walked the classes in order — owner, then group — and refused at the **other** class: you're
not the owner (`root` is) and not in the `adm` group either, so you fall into "other", and there is no
permission for other (`---`). Solutions: (a) lasting — `sudo usermod -aG adm ubuntu` (then log in again) to
join the `adm` group; (b) one-off — `sudo cat /var/log/app/app.log` or `sudo less ...`. · *Phase 2,
2.2.1–2.2.3*

**13.** The `/opt` mount is probably `noexec`; `./run.sh` invokes an `execve`, and even with `755`
bits, execution is refused on a `noexec` mount (Phase 2, 2.5.3). `bash run.sh` works because the
thing executed there is `bash` (on an exec-allowed mount); `run.sh` is only **read**. Confirm with:
`findmnt -T /opt/app/run.sh`. · *Phase 1 (mount) × Phase 2 (execve)*

**14.** (a) `777` makes every file in the directory writable by **every process on the machine** —
even a compromised service could change the application code (Phase 0: every process runs with an
identity; Phase 2: 2.2.2). (b) With `-R`, private keys/config files inside are opened to everyone
too. (c) It "solves" the problem without ever learning who was refused and why — it hides the actual
mechanism. The right way is to answer narrowly: who, to what, with which permission should have
access. · *Phase 0 × Phase 2*

**15.** From within `less` (a pager) you can open a shell by typing `!sh`. If `sudo journalctl` runs
as root and the output opens in a pager root started, that `!sh` gives a **root shell** — the
seemingly limited `sudo journalctl` privilege turns into full root (Phase 2, 2.4.1, shell escape).
This is why you check whether another command can be run from within a granted `sudo` command (the
`SYSTEMD_PAGER`, `sudoedit` logic). · *Phase 2, 2.4*

**16.** `id` shows the running shell's **current** identity (NO docker); `id ubuntu` reads **fresh**
from `/etc/group` (docker IS there). The difference is that identity is fixed in the session while
the file is current — that is, the proof of the "I added myself to the group but it doesn't work"
situation. · *Phase 2, 2.1.1*

**17.** Starting with `l`: a **symbolic link** (Phase 0, file types). `/bin -> usr/bin`: in the modern
FHS `/bin` is no longer a separate directory but a link to `/usr/bin` (Phase 1, the "usr merge").
The permissions `rwxrwxrwx` are misleading because **a link's own permissions aren't used** — access
follows the permissions of the **target** it points to. · *Phase 0 × Phase 1*

**18.** The trailing `t` is the **sticky bit** (Phase 2, 2.3.1). The behavior it provides: everyone
can create files in the directory but a file can be deleted **only by its owner** (or root). It's
essential in `/tmp` because `/tmp` is world-writable (`rwxrwxrwx`) — without sticky, any user could
delete another's temporary files. · *Phase 2, 2.3.1*

**19.** `noexec`. `./run.sh` wants to **execute** the file (execve), and a `noexec` mount refuses
this independently of the bits. `bash run.sh` works because what's executed there is `bash`;
`run.sh` is just a data file being read. · *Phase 2, 2.5.3 × bridge to Phase 6*

**20.** The line placing the barrier: `drwxr-x--- ubuntu ubuntu ubuntu` — that is, the
`/home/ubuntu` directory. `www-data` is neither the owner (`ubuntu`) nor in the group, so it falls
into the "other" class, and there it's `---`; that is, **no traverse permission** through this
directory. If any directory along the path is missing `x`, the kernel refuses before it can even
reach the file below — whatever that file's permission. The file being `-rw-r--r--` doesn't save it
because the problem isn't reading, it's **entering the path**. · *Phase 2, 2.2.3*

**21.** `ps aux` lists running processes (a preview of Phase 3, but used here as a Phase 1 tool),
`grep nginx` filters the nginx lines, `awk '{print $1}'` takes the first column (the username),
`sort -u` deduplicates — all Phase 1's stream/pipe/text-processing arsenal. `root` and `www-data`
appearing: the nginx **master** process starts as root (to bind to a privileged port like 80), then
runs its **worker** processes under the unprivileged `www-data` identity — the least-privilege
principle (Phase 2, 2.4.2). · *Phase 1 × Phase 2 (× bridge to Phase 3)*

---

## Scoring

| Number correct | Assessment |
|---|---|
| 19–21 | You've connected three phases into a single model. Move on to Phase 3 with confidence. |
| 15–18 | Solid. Take a pass through the **bridge** (not the single phase) that a missed question points to. |
| 10–14 | You know the phases separately but the links between them are weak. Use the table below. |
| 0–9 | Repeat the relevant phases, especially the "When this breaks" and "Closing" sections; a checkpoint quiz isn't passed until these bridges settle in. |

**Which question you missed → where to return:**

| Question you missed | Return to — this bridge is weak |
|---|---|
| 1, 2, 7, 17 | Phase 0 × Phase 2 — "everything is a file" + permission/file type |
| 3, 4, 5, 8 | Phase 1 × Phase 2 — shell/pipe/redirection runs **with your identity** |
| 6, 9 | Phase 0/1 × Phase 2 — FHS + setuid, distro family + tools |
| 10, 11, 16 | Phase 2 (2.1, 2.5) — when identity is fixed, SSH/StrictModes |
| 12, 20 | Phase 2 (2.2.1–2.2.3) — permission-class selection and path traverse |
| 13, 19 | Phase 1/6 × Phase 2 — mount options (noexec) + execve |
| 14, 15, 18, 21 | Phase 2 (2.3, 2.4) — special bits, least privilege, shell escape |

---

## Closing — from here to Phase 3

These three phases taught you to enter a server and read its **static** picture: which file, whose,
with which permission; which command, with which identity. But there's one thing you couldn't yet
ask: **"what is this machine doing right now?"** A process has an identity (Phase 2) but you haven't
yet seen the process itself — its PID, its parent, its state, the resources it consumes.

Phase 3 connects right here: it takes Phase 2's sentence "every process carries a UID/GID" and lays
on top of it "every process carries a PID, a parent, and a state"; you saw that `sudo` is "starting
a process with a different identity," and now you'll see how those processes are born (`fork`/`exec`),
how they die (signals, zombies), and how their resources are limited (cgroups — the reality beneath
containers).

> **Before you continue:** If you can answer Question 21 above without hesitation — why nginx runs
> processes under both `root` and `www-data` at once — you're ready for Phase 3. If you can't, take a
> minute back at Phase 2's section 2.4.2 (least privilege); Phase 3 will be built on top of that
> sentence.

---

> **Navigation:** [◀ Phase 2 — Users, Permissions and Identity](Phase_2_Users_and_Permissions.md) · **Checkpoint Quiz 1** · [Phase 3 — Processes and Resources ▶](Phase_3_Processes_and_Resources.md)
