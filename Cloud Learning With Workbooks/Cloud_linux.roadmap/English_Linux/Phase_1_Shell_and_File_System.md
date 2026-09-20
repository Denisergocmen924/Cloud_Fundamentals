# Phase 1 — Shell and File System: Finding Your Way in Linux

> **Navigation:** [◀ Phase 0 — Mental Model](Phase_0_Mental_Model.md) · **Phase 1** · [Phase 2 — Users and Permissions ▶](Phase_2_Users_and_Permissions.md)

---

## Where we are coming from

In Phase 0 we built three ideas: **two worlds** (kernel space and user space), the **single door**
between them (the syscall), and the **file interface** through which the kernel shows you the world.
We ran few commands in that phase; the goal was to build the model.

In this phase we start **walking** on top of that model. You will learn what the black screen you
reach over SSH every day really is, how a word you type turns into a program, why files live in
exactly those directories, and how to pull a single error out of thousands of log lines.

Three things from Phase 0 are directly useful here:

- **"Every command is a program"** — because programs run in user space and ask the kernel to
  start them with `execve`.
- **"File descriptors 0, 1, 2"** — because redirection and pipes are a game played with these
  three numbers.
- **"`/proc` is read like a file"** — because in this phase we put it in its place in the
  directory tree.

## The question of this phase

The moment you live most often when you connect to a server in the cloud is this:

> *"The application returns 502. I got onto the machine with SSH. Now where do I look, and how do I
> search that hundred-thousand-line log for what matters?"*

By the end of this phase you will answer this not with "I'll look around", but by saying **which
directory you go to, in which order, and which pipeline you write.**

---

## By the end of this phase

- You will be able to explain the difference between a terminal, a shell and a command
- You will be able to describe step by step how the shell **finds** a command (alias → builtin →
  PATH) and how it runs it (fork + exec)
- You will know the cause of "the script works in the terminal but says `command not found` in cron"
- You will know **what** the main FHS directories are for and which one to look at during an outage
- You will be able to find the file filling a disk, the config that changed recently or the log that
  is growing, with `find`
- You will be able to explain why stdout and stderr are separate and why the **order** of `2>&1`
  matters
- You will know what a pipe is in the kernel and how it connects two programs
- You will be able to pull a meaningful summary out of a log with `grep`, `cut`, `sort`, `uniq`,
  `wc`, `head`, `tail`, `sed` and `awk`

---

## Phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 1.1 | What a shell is, how a command is found and run | `[concept]` + `[mechanism]` | The model behind everything you type |
| 1.2 | File system hierarchy (FHS) | `[mechanism]` + `[application]` | The answer to "where do I look" during an outage |
| 1.3 | Navigation and `find` | `[application]` | Everyday hand skill and hunting for faults |
| 1.4 | stdin/stdout/stderr, redirection, pipes | `[mechanism]` | **The heart of the phase** — the combining power of Unix |
| 1.5 | Text-processing tools | `[application]` + `[concept]` | Getting answers out of logs |
| 1.6 | When this phase breaks | — | Signatures of shell and log failures |

> **How to work through this phase:** In every section labeled `[application]`, run the commands **on
> your own machine**. If you don't have Ubuntu at hand, a free-tier EC2 instance, a virtual machine or
> WSL is enough. Every command in this phase is marked 🟢 (read-only) or 🟡 (creates a temporary file);
> none of them change the system permanently.

---
---

# 1.1 What Is a Shell?

## 1.1.1 Terminal, shell and REPL `[concept]`

When you connect to a server over SSH, you see this:

```
ubuntu@ip-172-31-20-14:~$
```

Behind this screen there are three separate things, and in everyday speech all three are called
"the terminal":

| Part | What it does | Example |
|---|---|---|
| **Terminal (emulator)** | Takes your keyboard input and draws incoming characters on the screen. It does **not** understand commands. | GNOME Terminal, Windows Terminal, iTerm2, the VS Code terminal |
| **Connection** | Carries characters between the terminal and the remote machine | SSH |
| **Shell** | **Interprets** the line you type and runs programs | `bash`, `zsh`, `sh` (`dash`) |

So when you type `ls`, the terminal does not understand those letters; it only sends them across
SSH. The program called **bash** running on the other machine reads that line and decides what to do.

The shell's job is a loop. It is called a **REPL** (Read–Eval–Print Loop):

1. **Read** — Show the prompt, read a line.
2. **Eval** — Split the line, expand it, find the command, run it.
3. **Print** — The running program's output reaches the screen.
4. **Loop** — When the program ends, show a new prompt.

In the language of Phase 0: **the shell is just a user-space program too.** It has no special
privilege. To write to disk, start another program or connect to the network, it does what every
program does — it asks the kernel with a syscall.

**Reading the prompt:**

```
ubuntu@ip-172-31-20-14:~$
└─┬──┘ └──────┬──────┘ │ │
  │           │        │ └─ $ = normal user  (# = root)
  │           │        └─── ~ = the directory you are in (your home directory)
  │           └──────────── the machine's hostname (on EC2, derived from the private IP)
  └──────────────────────── the logged-in user
```

The last character of the prompt is small but important: **if you see `#`, you are root**, and every
command you type can change the system. We will look at root in detail in Phase 2.

> **🔧 See it on your machine** 🟢 — Which shell are you in?
>
> ```
> $ echo $SHELL
> /bin/bash
> $ ps -p $$
>     PID TTY          TIME CMD
>    1432 pts/0    00:00:00 bash
> ```
>
> - `$SHELL` → your **login shell** (the one recorded for your user in `/etc/passwd`). It does not always
>   show the shell you are in right now; even if you start `zsh` from `bash`, `$SHELL` does not change.
> - `$$` → **the PID of the shell running right now.** That is why `ps -p $$` shows the shell you are
>   really in.
> - `pts/0` → a pseudo-terminal. The kernel opens one for you when you connect over SSH. With the
>   "everything is a file" idea from Phase 0: it is a file called `/dev/pts/0`.

**❓ A question that comes to mind: bash, sh, zsh — which one should I learn?**

On servers, learn **bash**. On Ubuntu, Debian, Amazon Linux and RHEL, bash is the default interactive
shell for users. There is a trap: on Ubuntu `/bin/sh` is **not bash**, but a smaller and faster shell
called `dash`. If a script starting with `#!/bin/sh` uses a bash-only feature (such as `[[ ]]` or
arrays), you get an error on Ubuntu. We will come back to this when writing scripts in Phase 10. The
default `zsh` on macOS is comfortable on your personal machine, but assume you won't find it on a server.

---

## 1.1.2 A command = a program call: how does the shell find a command? `[mechanism]`

You typed `ls -l /var/log` and pressed Enter. Until the list appears on the screen, this happens
inside the shell:

**Step 1 — Split (tokenize).** The line is split on spaces: `ls`, `-l`, `/var/log`. The first piece
is the **command name**, the rest are **arguments**. Quotes change the splitting: `echo "a  b"` is one
argument, `echo a  b` is two.

**Step 2 — Expand.** Before running the program, the shell resolves special characters **itself**:

- `$HOME` → `/home/ubuntu` (variable expansion)
- `~` → `/home/ubuntu`
- `*.log` → `auth.log syslog.log ...` (glob — file name expansion)
- `$(date +%F)` → `2026-09-15` (command substitution)

This is a critical point: **when you type `ls *.log`, the `ls` program never sees the `*` character.**
The shell turns the asterisk into file names and gives `ls` the list of files.

**Step 3 — Find the command.** The shell searches for the command name in this order and **stops at
the first match:**

1. **Alias** — a shortcut (`ll` → `ls -alF`)
2. **Function** — a function defined in the shell
3. **Builtin** — a command inside the shell itself (`cd`, `echo`, `export`, `exit`)
4. **PATH search** — an executable file with this name in the directories of `$PATH`, **from left to
   right**

**Step 4 — Run it.** If the command is a builtin, the shell does the work itself; there is no new
process. Otherwise:

1. The shell **copies** itself (`fork`; on Linux the `clone` syscall). Now there are two shells: the
   parent and the child.
2. The child calls `execve("/usr/bin/ls", ["ls", "-l", "/var/log"], ...)`. The kernel empties the
   child's memory and loads the `ls` program into it. The child is no longer bash; it is `ls`.
3. The parent waits for the child to finish with `wait`.

**Step 5 — Exit code.** When `ls` finishes, it leaves a number with the kernel. The shell puts it into
the `$?` variable and shows a new prompt. **`0` = success, anything other than `0` = some kind of
error.**

![Figure 1.1 — How the shell finds and runs a command](../diagrams/png/lx-1-01-command-lookup.png)
*Figure 1.1 — Left to right: the line is split and expanded; the command is searched as alias, function,
builtin, then PATH; if it is not a builtin, the shell copies itself with fork, the child turns into the program
with execve, and the parent waits for the exit code.*

> **🔧 See it on your machine** 🟢 — Where does a command come from?
>
> ```
> $ type cd
> cd is a shell builtin
> $ type -a echo
> echo is a shell builtin
> echo is /usr/bin/echo
> echo is /bin/echo
> $ type -a ls
> ls is aliased to `ls --color=auto'
> ls is /usr/bin/ls
> ls is /bin/ls
> $ command -v ls
> alias ls='ls --color=auto'
> ```
>
> - `type` is a builtin and shows you the shell's **search order**. `-a` lists every match; the top one
>   wins.
> - `echo` exists both as a builtin and as a program. The shell uses the builtin; `/usr/bin/echo` never
>   runs.
> - `ls` is an alias first. Ubuntu's `~/.bashrc` defines it for colored output.
> - `/usr/bin/ls` and `/bin/ls` are the same file: on modern Ubuntu `/bin` is just a link to `/usr/bin`
>   (we will see this in 1.2.1).

> **🔧 See it on your machine** 🟢 — PATH itself
>
> ```
> $ echo $PATH
> /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
> ```
>
> Directories separated by colons (`:`), searched from left to right. `/usr/local/bin` comes **before**
> `/usr/bin`. This is a deliberate decision: if you put a `python3` you compiled or installed by hand in
> `/usr/local/bin`, it **shadows** the system's `python3`. This is both a feature and a common cause of
> "the wrong version is running on the server" failures.

> **🔧 See it on your machine** 🟢 — Watching fork and exec live
>
> ```
> $ strace -f -e trace=execve,clone,wait4 bash -c 'ls /tmp > /dev/null; echo done'
> execve("/usr/bin/bash", ["bash", "-c", "ls /tmp > /dev/null; echo done"], 0x7fff... /* 25 vars */) = 0
> clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7fc9...) = 14950
> strace: Process 14950 attached
> [pid 14949] wait4(-1 <unfinished ...>
> [pid 14950] execve("/usr/bin/ls", ["ls", "/tmp"], 0x5aea... /* 25 vars */) = 0
> [pid 14950] +++ exited with 0 +++
> <... wait4 resumed>, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 14950
> --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=14950, ...} ---
> done
> +++ exited with 0 +++
> ```
>
> Line by line:
>
> - `execve(".../bash"...)` → strace starts bash.
> - `clone(...) = 14950` → bash (PID 14949) copied itself; the child's PID is 14950.
> - `[pid 14949] wait4(-1 ...` → the parent waits.
> - `[pid 14950] execve("/usr/bin/ls", ["ls", "/tmp"], ...)` → the child turned into `ls`. There is **no**
>   `>` or `/dev/null` in the argument list: the shell did the redirection before `ls` started (1.4.2).
> - `WEXITSTATUS(s) == 0` → `ls` exited with 0; this number becomes `$?`.
> - `done` was printed, but **there is no `clone` or `execve` before it.** Because `echo` is a builtin;
>   bash did the work internally.
>
> If `strace` is not installed: `sudo apt install strace` (Ubuntu) or `sudo dnf install strace`
> (Amazon Linux).

> **⚠️ Common misconception: "`cd` is a program too, like `/usr/bin/cd`."**
>
> It cannot be — and the reason comes from the process model in Phase 0. Every process has its own
> **current working directory**. If `cd` were a separate program, the shell would run it as a **child**
> process with fork + exec. The child would change its own directory, then die. The parent shell's
> directory would never change.
>
> A process cannot change another process's working directory. That is why every command that changes
> **the shell's own state** — `cd`, `export`, `source`, `exit` — has to be a builtin.

**❓ A question that comes to mind: Why do I have to type `./script.sh`? Isn't `script.sh` enough?**

Because the current directory (`.`) is **not** in PATH, and this is a deliberate security decision. If
`.` were in PATH, someone could drop a malicious script named `ls` into `/tmp`, and when you typed
`cd /tmp && ls`, their script would run instead of the system's `ls`. `./script.sh` means "don't look at
PATH, run the file **in exactly this directory**." Any command name that contains a `/` skips the PATH
search.

**❓ A question that comes to mind: What happens if a glob matches no file?**

Bash's default behavior is surprising: **it leaves the asterisk as it is.**

```
$ echo *.txt
*.txt
$ ls *.txt
ls: cannot access '*.txt': No such file or directory
```

Here `ls` is looking for a **real file** named `*.txt`, because the shell passed the asterisk without
expanding it. The quoted `'*.txt'` in the error message is the sign. This behavior is the key to the next
Think question.

> **🤔 Think 1.1**
> You wrote a script. It runs fine in the terminal with `./backup.sh`. It contains an `aws s3 cp`
> command, and you installed the `aws` CLI under `/usr/local/bin`. You added the same script to cron to
> run every night at 03:00. In the morning you see this in the log file:
>
> ```
> backup.sh: line 4: aws: command not found
> ```
>
> Same command, same script, same user. Why can't cron find it? Suggest at least two fixes.
> *(Answer: at the end of the phase)*

> **🤔 Think 1.2**
> You are in `/home/ubuntu`, and there is a file called `notes.log` here. You type:
>
> ```
> $ find /var/log -name *.log
> ```
>
> What you expect: every `.log` file under `/var/log`. What you get: it searches only for a file named
> `notes.log` and finds nothing. In another directory with **two** `.log` files, the same command gives
> the error `find: paths must precede expression`. And in a directory with no `.log` file, it **works
> correctly.** What is the single cause of these three different behaviors?
> *(Answer: at the end of the phase)*

---
---

# 1.2 The File System Hierarchy (FHS)

## 1.2.1 One tree, deliberate directories `[mechanism]`

On Windows every disk is a separate letter: `C:\`, `D:\`. Linux has no such thing. **Everything is
inside a single tree**, and the root of the tree is written as `/` (the root directory). When a second
disk is attached, it is **connected to a branch** of the tree (mounted), for example at `/data`. For
the user there is no difference between `/data/report.csv` and `/home/ubuntu/report.csv` except the
path, even if one is on another disk. We will look at mounting in detail in Phase 6.

The directories of this tree are not random. The **FHS** (Filesystem Hierarchy Standard) defines what
each directory is for. Ubuntu, Debian, Amazon Linux and RHEL follow this standard closely. That is why
"config is in `/etc`, logs are in `/var/log`", learned on one distro, also works on another.

The logic behind the FHS rests on two questions:

1. **Does this data change?** Programs (`/usr`) are fixed; logs and databases (`/var`) grow all the
   time.
2. **Is this data persistent?** `/run` and `/tmp` may be reset at boot; `/etc` and `/var/lib` must live
   on.

These two questions matter a lot in practice. If you put `/var` on a separate disk, a log file growing
out of control does not fill **the rest of the system**. If you mount `/usr` read-only, nobody can
change the programs.

| Directory | What it holds | Does it change? | Why it matters in the cloud |
|---|---|---|---|
| `/etc` | System-wide **config** files (text) | Rarely, by hand | Where you change a service's behavior |
| `/var/log` | Log files | Grows all the time | The first place to look during an outage; suspect number one for a full disk |
| `/var/lib` | Programs' **persistent state data** | All the time | Docker images, database files, the package database |
| `/var/cache` | Cache that can be deleted | Yes | `apt` downloads; can be safely cleaned to free space |
| `/tmp` | Temporary files, writable by everyone | Yes | Cleaned at boot; don't put anything important there |
| `/var/tmp` | Temporary files that **survive a reboot** | Yes | The difference from `/tmp` is persistence |
| `/home` | Home directories of normal users | Yes | `/home/ubuntu/.ssh/authorized_keys` |
| `/root` | The root user's home directory | Yes | **Not** `/home/root` |
| `/usr/bin` | User programs | When packages are installed | `ls`, `grep`, `python3` live here |
| `/usr/sbin` | System administration programs | When packages are installed | `sshd`, `useradd` |
| `/usr/lib` | Libraries and package files | When packages are installed | Default systemd unit files are here too |
| `/usr/local` | Things installed by hand **outside the package manager** | By hand | Tools you compiled yourself or installed with a script |
| `/opt` | Third-party software that brings its own directory layout | By hand | `/opt/aws/...`, vendor agents |
| `/boot` | The kernel and boot files | On kernel updates | If it fills up, kernel updates fail (Phase 5) |
| `/dev` | Device files | Managed by the kernel | `/dev/nvme1n1` — an attached EBS disk |
| `/proc` | Live state of processes and the kernel | **Not** on disk | Phase 0.3.2 |
| `/sys` | The device and driver tree | **Not** on disk | Phase 0.3.3 |
| `/run` | Runtime data (PID files, sockets) | Reset at boot | Lives in RAM |
| `/mnt`, `/media` | Mount points for temporary or removable media | — | Disks mounted by hand |
| `/srv` | Data being served (web roots and so on) | — | Rarely used; most setups use `/var/www` |

![Figure 1.2 — The Linux directory tree and the roles of directories](../diagrams/png/lx-1-02-fhs-map.png)
*Figure 1.2 — Under a single root, directories are grouped by the kind of data they hold: config
(`/etc`), changing data (`/var`), programs (`/usr`), user data (`/home`), and virtual directories that
are not on disk but produced by the kernel (`/proc`, `/sys`, `/dev`, `/run`).*

> **🔧 See it on your machine** 🟢 — Inside the root directory
>
> ```
> $ ls -l /
> lrwxrwxrwx   1 root root     7 Apr 20 11:46 bin -> usr/bin
> drwxr-xr-x   4 root root  4096 Sep 15 13:12 boot
> drwxr-xr-x  17 root root  3300 Sep 15 17:02 dev
> drwxr-xr-x 104 root root  4096 Sep 15 13:12 etc
> drwxr-xr-x   3 root root  4096 Aug 30 21:59 home
> lrwxrwxrwx   1 root root     7 Apr 20 11:46 lib -> usr/lib
> lrwxrwxrwx   1 root root     9 Apr 20 11:46 lib64 -> usr/lib64
> drwx------   2 root root 16384 May 20 21:13 lost+found
> dr-xr-xr-x 181 root root     0 Sep 15 17:02 proc
> drwx------   5 root root  4096 Sep 15 13:11 root
> drwxr-xr-x  29 root root   900 Sep 15 17:05 run
> lrwxrwxrwx   1 root root     8 Apr 20 11:46 sbin -> usr/sbin
> drwxrwxrwt  12 root root  4096 Sep 15 17:37 tmp
> drwxr-xr-x  12 root root  4096 May 26 00:34 usr
> drwxr-xr-x  13 root root  4096 May 26 11:18 var
> ```
>
> - `bin -> usr/bin` → lines starting with `l` are **symbolic links**. `/bin` and `/usr/bin` used to be
>   separate directories; modern distros merged them (usrmerge). That is why `/bin/ls` and `/usr/bin/ls`
>   are the same file.
> - The `proc` line shows size `0` and permissions `dr-xr-xr-x` → a directory produced by the kernel, not
>   on disk.
> - The `t` at the end of the `tmp` permissions → the **sticky bit.** Everyone can write, but everyone can
>   delete only their own files. We will see it in Phase 2.3.
> - The `root` line is `drwx------` → nobody else can enter root's home directory.
> - `lost+found` → the ext4 file system's recovery directory; pieces recovered during a repair land here.

> **🔧 See it on your machine** 🟢 — Which directory is really on disk?
>
> ```
> $ df -hT
> Filesystem      Type   Size  Used Avail Use% Mounted on
> /dev/root       ext4    29G  4.1G   25G  15% /
> tmpfs           tmpfs  1.9G     0  1.9G   0% /dev/shm
> tmpfs           tmpfs  766M  1.0M  765M   1% /run
> tmpfs           tmpfs  5.0M     0  5.0M   0% /run/lock
> /dev/nvme0n1p15 vfat   105M  6.1M   99M   6% /boot/efi
> tmpfs           tmpfs  383M  4.0K  383M   1% /run/user/1000
> ```
>
> - `/dev/root ext4 ... /` → the root EBS disk on EC2; the rest of the tree is here.
> - `tmpfs` → a file system living **in RAM**. `/run` is here; that is why it is reset on reboot.
> - `/boot/efi` → a separate, small partition. Boot files read by the firmware.
> - `/proc` and `/sys` don't appear in this list because `df` hides them by default; you can see them with
>   `df -a`.

> **⚠️ Common misconception: "`/tmp` is always in RAM" or "`/tmp` is always on disk."**
>
> **It depends on the distro.** On Ubuntu 22.04 and 24.04, `/tmp` is a directory on the root disk and is
> cleaned at boot. On Amazon Linux 2023 (and on some newer Ubuntu releases), `/tmp` is a **tmpfs**: it
> lives in RAM and its size is limited by RAM. Two practical consequences:
>
> - On Amazon Linux, a script trying to unpack a 3 GB archive into `/tmp` on a small instance with 2 GB
>   of RAM crashes with "No space left on device" — even if the root disk is empty.
> - Whatever distro you are on, **don't rely on a file in `/tmp` surviving a reboot.** Use `/var/tmp` for
>   persistent temporary files.
>
> On your own machine, `df -hT /tmp` tells you which one it is right away.

**❓ A question that comes to mind: `/usr/local/bin`, `/opt` or `/usr/bin` — where should I install a
tool?**

The rule is simple: **`/usr/bin` belongs to the package manager.** If you put a file there by hand, the
next `apt upgrade` may overwrite it, or the package manager won't know why that file is there.
Single-file tools you install by hand (the `aws` CLI, `kubectl`, your own scripts) go into
`/usr/local/bin`; large software that brings its own directory layout goes into `/opt/<name>`. Thanks to
the PATH order (`/usr/local/bin` first), the version you install by hand shadows the system's.

---

## 1.2.2 The three most visited places: `/etc`, `/var/log`, `/proc` `[application]`

When you hunt for a fault on a server, most of your time is spent in these three directories. Each has a
different "reading habit".

### `/etc` — how the system should behave

Almost everything in `/etc` is **plain text**. This is a deliberate Unix choice: to read and change
config you need a text editor and `grep`, not a special tool.

Learn two patterns:

**1. `.d` directories (drop-in).** `/etc/ssh/sshd_config` is the main file; but the `*.conf` files under
`/etc/ssh/sshd_config.d/` are **read too** and override it. Why? Because a package update may want to
change the main file. If you put your own setting in a separate file, there is no conflict with the
update. cloud-init adds its own settings this way too.

> **🔧 See it on your machine** 🟢 — Drop-in directories
>
> ```
> $ ls -d /etc/*.d | head -8
> /etc/apparmor.d
> /etc/apt/apt.conf.d
> /etc/cron.d
> /etc/init.d
> /etc/ld.so.conf.d
> /etc/logrotate.d
> /etc/profile.d
> /etc/rsyslog.d
> $ ls /etc/ssh/sshd_config.d/
> 50-cloud-init.conf  60-cloudimg-settings.conf
> ```
>
> Even if `sshd_config` says `PasswordAuthentication yes`, if `50-cloud-init.conf` says
> `PasswordAuthentication no`, you need to know **which one wins**. So before changing a config, always
> look at both the main file and the `.d` directory. (For sshd, the first value read wins; details in
> Phase 9.)

**2. Files managed through a link.** Some files are really produced somewhere else:

```
$ ls -l /etc/resolv.conf
lrwxrwxrwx 1 root root 39 Aug 21 10:02 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
```

If you edit `/etc/resolv.conf` by hand, your change is **lost**, because the real file is under `/run`
and systemd-resolved rewrites it. We will come back to this link when we reach DNS in Phase 7. For now
the lesson: **before editing a file in `/etc`, check with `ls -l` whether it is a link.**

### `/var/log` — what happened on the system

> **🔧 See it on your machine** 🟢 — The log directory on Ubuntu
>
> ```
> $ ls /var/log
> README               auth.log.2.gz          dmesg                 kern.log.1
> alternatives.log     cloud-init-output.log  dpkg.log              landscape
> amazon               cloud-init.log         journal               lastlog
> apt                  dist-upgrade           kern.log              syslog
> auth.log             dmesg.0                syslog.1              unattended-upgrades
> auth.log.1           ...
> ```
>
> | File | What is inside | When you look at it |
> |---|---|---|
> | `syslog` | General system messages | "Something happened but I don't know what" |
> | `auth.log` | Login attempts, `sudo`, SSH | "Who logged in?", "Why did SSH refuse?" |
> | `kern.log` | Kernel messages | Disk errors, the OOM killer (Phase 4) |
> | `cloud-init.log` | What cloud-init did | "Did user-data run?" |
> | `cloud-init-output.log` | **The output of the user-data script** | "Why did user-data fail?" |
> | `journal/` | systemd journal binary files | Read with `journalctl` (1.5.3) |
> | `apt/`, `dpkg.log` | Package installations | "What was updated last night?" |
>
> - `auth.log.1`, `auth.log.2.gz` → old logs rotated by **logrotate**. `.1` is yesterday's, `.2.gz` is the
>   compressed one from the day before. In 1.5.4 you will see why these files matter.
>
> **On Amazon Linux 2023** most of this list does not exist: there are no `syslog`, `auth.log` or
> `kern.log` files, because rsyslog is not installed by default. Everything is in the **journal** and is
> read with `journalctl`. The cloud-init logs exist on both.

### `/proc` — what is happening right now

In Phase 0 we saw `/proc/meminfo` and `/proc/loadavg`. Here, let's look at the directory opened for each
process. Every running process is a directory under `/proc/<PID>/`:

> **🔧 See it on your machine** 🟢 — A process's identity
>
> ```
> $ sleep 300 &
> [1] 2214
> $ ls /proc/2214
> attr  cgroup  cmdline  comm  cwd  environ  exe  fd  io  limits  maps  mem  mounts
> net  ns  oom_score  root  sched  smaps  stack  stat  statm  status  task  wchan ...
> $ tr '\0' ' ' < /proc/2214/cmdline; echo
> sleep 300
> $ readlink /proc/2214/exe /proc/2214/cwd
> /usr/bin/sleep
> /home/ubuntu
> $ kill %1
> ```
>
> | File | What it tells you |
> |---|---|
> | `cmdline` | The process's **full command line** (arguments separated by `\0`; hence `tr`) |
> | `exe` | A link to the program file actually running |
> | `cwd` | A link to the process's working directory |
> | `environ` | The environment variables at the moment it started (readable only by its owner and root) |
> | `fd/` | Open files (in detail in 1.4.1) |
> | `status` | State, memory, UIDs — a human-readable summary |
>
> Practical use: if a process behaves strangely, `cat /proc/<PID>/cmdline` shows which arguments it
> started with, and `/proc/<PID>/environ` shows which environment variables it started with — **without
> guessing.** Half of "which config file does this service read?" is answered here.

> **🤔 Think 1.3**
> An alert arrives at 02:00: the root file system of an EC2 instance is `100%` full. The application can't
> write new files; you can get in over SSH, but even `sudo apt install` doesn't work.
>
> 1. According to the FHS, **which two directories** do you look at first, and why?
> 2. A teammate says "let's run `du -sh /*`". Does this command also walk under `/proc`? Is that a
>    problem? And what happens if a separate 500 GB `/data` disk is attached to the instance?
> *(Answer: at the end of the phase)*

---
---

# 1.3 Navigation and File Operations

## 1.3.1 `cd`, `ls`, `cp`, `mv`, `rm` — a quick check `[application]`

You probably know these commands. Instead of teaching them, this section shows the **mistakes commonly
made on servers** and the mechanisms behind them.

| Command | What you need to know | Why |
|---|---|---|
| `cd -` | Goes back to the previous directory | When jumping between two distant directories |
| `ls -la` | Also shows hidden files (starting with `.`) | `.ssh`, `.bashrc`, `.env` are hidden |
| `ls -lh` | Writes sizes with `K`, `M`, `G` | Spotting a big file at a glance |
| `ls -ltr` | Sorts by modification time, **newest at the bottom** | "Which log changed last?" — the bottom one |
| `cp -a` | **Preserves** permissions, owner, times and links | Use instead of `cp -r` when backing up config |
| `cp file{,.bak}` | Same as `cp file file.bak` | The shell's brace expansion; less typing |
| `mv` | Within the same file system, only changes the **name** | Moves even a 50 GB file instantly |
| `rm -r` | Deletes a directory with its contents | **There is no trash bin** |
| `rm -i` | Asks for confirmation for every file | When you are not sure |

**Why is `mv` so fast?** We will see the details in Phase 6, but in short: a file's data sits somewhere on
disk, while its **name** is an entry inside a directory. Within the same file system, `mv` never touches
the data; it just takes this entry from one directory and puts it into another. When moving to a different
file system (for example from the `/` disk to the `/data` disk), it **copies** the data from start to end
and deletes the old one; that is why it is slow.

> **🔧 See it on your machine** 🟢 — `mv` does not move data
>
> ```
> $ touch test.txt
> $ stat -c '%i %n' test.txt
> 655 test.txt
> $ mv test.txt new-name.txt
> $ stat -c '%i %n' new-name.txt
> 655 new-name.txt
> $ rm new-name.txt
> ```
>
> `%i` is the file's **inode number** — the file's identity on disk. The name changed, the identity stayed
> the same. Your number will be different; what matters is that it is the same on both lines.

> **⚠️ Common misconception: "I deleted the big log file with `rm`, so disk space must have been freed."**
>
> If a process still **holds that file open** (such as the application writing the log), no space is
> freed. `rm` only removes the **name** from the directory. With the model from Phase 0: the kernel does not
> free the data of a file that has an open file descriptor until that fd is closed. The file is nameless
> but alive.
>
> Symptom: `df -h` shows the disk at 100%, while `du` finds a much smaller total for the files.

> **🔧 See it on your machine** 🟡 — A deleted but open file
>
> ```
> $ dd if=/dev/zero of=big.dat bs=1M count=50 status=none
> $ tail -f big.dat > /dev/null &
> [1] 3071
> $ rm big.dat
> $ ls -l /proc/3071/fd | grep deleted
> lr-x------ 1 ubuntu ubuntu 64 Sep 15 17:40 3 -> /home/ubuntu/big.dat (deleted)
> $ sudo lsof +L1
> COMMAND  PID   USER   FD   TYPE DEVICE SIZE/OFF NLINK   NODE NAME
> tail    3071 ubuntu    3r   REG  259,1 52428800     0 262311 /home/ubuntu/big.dat (deleted)
> $ kill %1
> ```
>
> - `(deleted)` → there is no name, but the `tail` process still holds the file through fd 3.
> - `lsof +L1` → lists open files with a link count below 1 (that is, **0**: the name was deleted). The
>   `SIZE/OFF` column shows that 52428800 bytes = 50 MB are still on disk.
> - When `tail` is closed with `kill %1`, the kernel frees the 50 MB.
>
> **Undo:** if you forget to run `kill %1`, `tail` stays in the background; see it with `jobs` and close it
> with `kill %1`. In production the fix is usually restarting the service, or logrotate sending the service a
> "reopen the file" signal (1.5.4).

---

## 1.3.2 `find` — for hunting faults `[application]`

`ls` looks at one directory. `find` walks a **tree** and tests every file against the conditions you give.
Most of the questions you ask on a server are really `find` queries:

| Question | Command |
|---|---|
| "Where are the files over 100 MB that are filling the disk?" | `sudo find / -xdev -type f -size +100M` |
| "Which config changed in the last 30 minutes?" | `sudo find /etc -type f -mmin -30` |
| "Which logs are older than 7 days?" | `find /var/log -name '*.gz' -mtime +7` |
| "Is there an `.env` file in this directory?" | `find /srv -name '.env'` |
| "Files with no owner?" | `sudo find / -xdev -nouser` |

**Structure:** `find <where from> <tests> <action>`

| Part | Meaning |
|---|---|
| `-name '*.log'` | Name match (case-sensitive; `-iname` is case-insensitive). **Quote the pattern.** |
| `-type f` / `-type d` | Files only / directories only |
| `-size +100M` | Larger than 100 MB (`-` smaller; `k`/`M`/`G` units) |
| `-mtime +7` | Content changed **more than** 7 days ago (`-7` = within the last 7 days) |
| `-mmin -30` | Changed within the last 30 minutes |
| `-xdev` | **Don't cross into other file systems** (`/proc`, `/sys`, separate disks) |
| `-maxdepth 2` | Go down at most 2 levels |
| `-exec cmd {} +` | Pass the files found to a command in batches |
| `-delete` | Delete what was found — **run it without `-delete` first** |

> **🔧 See it on your machine** 🟢 — Finding big files
>
> ```
> $ sudo find / -xdev -type f -size +100M -exec ls -lh {} + 2>/dev/null
> -rw-r----- 1 syslog adm  1.3G Sep 15 17:41 /var/log/syslog
> -rw-r--r-- 1 root   root 180M Sep  3 10:12 /var/lib/snapd/snaps/core22_1586.snap
> ```
>
> - `-xdev` → stay on the root disk. Don't descend into virtual file systems like `/proc` or into other
>   attached disks. Both fast and correct: it answers "why is **this** disk full?"
> - `-exec ls -lh {} +` → gives the files found to `ls -lh` **in one go**. `{}` stands for the file names;
>   `+` means "all at once". (If you end it with `\;`, a separate `ls` process starts for every file.)
> - `2>/dev/null` → hides permission errors (in detail in 1.4.2).
>
> A 1.3 GB `syslog` here is suspicious: something is printing logs to the system fast. The next question
> is "what is printing?", and the answer is in the tools of 1.5.

**Looking per directory: `du`.** `find` finds individual files; but if the problem is made of thousands
of small files (a cache directory, for example), you need to look at **directory totals**:

> **🔧 See it on your machine** 🟢 — Which directory is big?
>
> ```
> $ sudo du -xh --max-depth=1 /var | sort -h | tail -5
> 39M     /var/snap
> 174M    /var/cache
> 1.6G    /var/log
> 2.3G    /var/lib
> 4.1G    /var
> ```
>
> - `-x` → the same as `-xdev` in `find`: don't cross into another file system.
> - `--max-depth=1` → totals of one sub-level only.
> - `sort -h` → "human" sort: it knows `174M` is smaller than `1.6G`. Plain `sort` doesn't (1.5.1).
> - The biggest is at the bottom. The next step is to go into the biggest directory and repeat the same
>   command: `sudo du -xh --max-depth=1 /var/log | sort -h | tail -5`.

> **🤔 Think 1.4**
> A teammate suggests adding this command to cron for disk cleanup:
>
> ```
> find / -name *.log -mtime +7 -delete
> ```
>
> Find **at least four** separate problems in this command. For each, write what can go wrong and how you
> would fix it.
> *(Answer: at the end of the phase)*

---
---

# 1.4 stdin, stdout, stderr and Pipes

## 1.4.1 Three standard streams: 0, 1, 2 `[mechanism]`

In Phase 0.3.1 we saw that every process has a **file descriptor (fd) table**: the number of each open
file inside the process. Unix gives the first three numbers of this table a special meaning by
**convention**:

| fd | Name | Connected by default to | What for |
|---|---|---|---|
| **0** | stdin (standard input) | Terminal (keyboard) | The data the program **reads** |
| **1** | stdout (standard output) | Terminal (screen) | The **result** the program produces |
| **2** | stderr (standard error) | Terminal (screen) | **Error and diagnostic messages** |

There is a subtle but very important point here: **the program does not know about the terminal.** `ls`
only says "write to fd 1". The kernel knows whether a terminal, a file or another program is behind fd 1;
`ls` does not. This is the practical form of the "one interface" idea from Phase 0, and the rest of this
phase is built entirely on this fact.

**Why is stderr separate?** Look at this command:

```
$ ls /etc/hostname /nope
ls: cannot access '/nope': No such file or directory
/etc/hostname
```

Both lines are on the screen, but they came from **different channels**: the first from fd 2 (error),
the second from fd 1 (result). They look the same on screen because both are connected to the same
terminal. Now let's send the result to a file:

```
$ ls /etc/hostname /nope > result.txt
ls: cannot access '/nope': No such file or directory
$ cat result.txt
/etc/hostname
```

The error is **still on the screen**, the result is in the file. This is exactly the purpose of separate
channels: when you give a program's output to a file or another program, **error messages don't get mixed
into that data**, and you keep seeing them. If errors also went to stdout, the next program would take the
line `ls: cannot access` for a file name and try to process it.

> **🔧 See it on your machine** 🟢 — Your shell's three streams
>
> ```
> $ ls -l /proc/$$/fd
> lrwx------ 1 ubuntu ubuntu 64 Sep 15 17:45 0 -> /dev/pts/0
> lrwx------ 1 ubuntu ubuntu 64 Sep 15 17:45 1 -> /dev/pts/0
> lrwx------ 1 ubuntu ubuntu 64 Sep 15 17:45 2 -> /dev/pts/0
> lrwx------ 1 ubuntu ubuntu 64 Sep 15 17:45 255 -> /dev/pts/0
> ```
>
> - **All three** of 0, 1 and 2 go to the same place: the virtual terminal of your SSH session.
> - `255` → a spare fd bash keeps on the terminal for its own internal use; you can ignore it.
> - When the shell starts a command (fork), the child **inherits** this table. That is why `ls`'s fd 1 is
>   automatically your terminal.

---

## 1.4.2 Redirection and pipes `[mechanism]`

Redirection is the shell changing the child's fd table **just before starting it**. Remember the strace
output from 1.1.2: for `ls /tmp > /dev/null`, there was no `>` in the arguments passed to `execve`.
Because the order is:

```
the shell forks
  └── child (still bash):
        1. opens /dev/null → it becomes fd 3
        2. dup2(3, 1)  → fd 1 now points to /dev/null
        3. close(3)
        4. execve("/usr/bin/ls", ["ls", "/tmp"])   ← ls finds fd 1 ready
```

`ls` is unaware of anything. It writes to fd 1 as always.

| Syntax | Meaning | Mechanism |
|---|---|---|
| `cmd > file` | Write stdout to the file, **truncate the file first** | `open(O_TRUNC)` + `dup2(fd, 1)` |
| `cmd >> file` | **Append** stdout to the end of the file | `open(O_APPEND)` + `dup2(fd, 1)` |
| `cmd 2> file` | Write stderr to the file | `dup2(fd, 2)` |
| `cmd 2>&1` | Connect fd 2 to wherever fd 1 points **right now** | `dup2(1, 2)` |
| `cmd &> file` | stdout and stderr together into the file (bash) | Short for `> file 2>&1` |
| `cmd < file` | Read stdin from the file | `dup2(fd, 0)` |
| `cmd > /dev/null` | Throw stdout away | `/dev/null`: a character device that swallows what is written |

**Rule 1 — `>` truncates the file BEFORE the command runs.** The shell empties the file while opening it;
the command has not even started yet.

```
$ sort records.log > records.log
$ ls -l records.log
-rw-rw-r-- 1 ubuntu ubuntu 0 Sep 15 17:46 records.log
```

The file is **empty.** By the time `sort` started, the file it was going to read had already been
truncated. Result: data loss, no error message.

**Rule 2 — redirections are applied from left to right, in order.** `2>&1` does not mean "connect fd 2
to fd 1"; it means "connect fd 2 **to wherever fd 1 points right now**":

```
$ ls /nope . > o1.txt 2>&1      # 1) fd1 → o1.txt   2) fd2 → (fd1's target) o1.txt
$ cat o1.txt
ls: cannot access '/nope': No such file or directory
.:
access.log
...

$ ls /nope . 2>&1 > o2.txt      # 1) fd2 → (fd1's target) terminal   2) fd1 → o2.txt
ls: cannot access '/nope': No such file or directory
$ cat o2.txt
.:
access.log
...
```

In the second command the error landed on the **terminal**, because when `2>&1` was processed, fd 1 was
still the terminal. When logging cron and systemd output, this ordering mistake is the classic cause of
"why are the error logs empty?"

> **⚠️ Common misconception: "`sudo echo 'x' > /etc/file` writes as root."**
>
> It doesn't, and you get `Permission denied`. The reason is the mechanism above: **the shell does the
> redirection**, and the shell runs as your normal user. `sudo` only makes the `echo` program root; but
> the attempt to open `/etc/file` happens before `sudo` starts, with your permissions.
>
> The right way is to hand the job of opening the file to **a program running as root**:
>
> ```
> $ echo 'x' | sudo tee /etc/file > /dev/null      # overwrite
> $ echo 'x' | sudo tee -a /etc/file > /dev/null   # append
> ```
>
> `tee` writes what it reads from stdin both to the file and to stdout; `> /dev/null` stops it from
> printing to the screen again.

### Pipe: one program's output, another's input

When you type `cmd1 | cmd2`, the shell does this:

1. It asks the kernel for a **pipe** (the `pipe()` syscall). The kernel creates a small buffer in RAM
   and returns two fds: a **write end** and a **read end.**
2. It forks twice.
3. In the first child it connects fd 1 to the write end of the pipe and execs `cmd1`.
4. In the second child it connects fd 0 to the read end of the pipe and execs `cmd2`.

![Figure 1.3 — A pipe connects the fds of two processes](../diagrams/png/lx-1-03-pipe-fds.png)
*Figure 1.3 — `ls | grep log`: ls's fd 1 is connected to the write end of the pipe buffer in the kernel,
grep's fd 0 to the read end. The stderr (fd 2) of both processes does not enter the pipe; it goes to the
terminal.*

Four important consequences follow from this model:

**1. A pipe is not a file; nothing is written to disk.** It is a buffer in the kernel; on Linux its
default capacity is 64 KiB.

**2. The two programs run at the same time.** `cmd2` has started before `cmd1` finishes. If the buffer
fills up, the writer (`cmd1`) is **made to wait** until the reader makes room; if the buffer is empty, the
reader waits. That is why `tail -f log | grep ERROR` can flow forever: the data doesn't pile up anywhere.

**3. If the reader exits early, the writer dies with SIGPIPE.** When you type `yes | head -2`, `yes`
prints `y` forever. `head` takes two lines and exits. The read end of the pipe closes. On its next write
attempt, `yes` receives the **SIGPIPE** signal from the kernel and dies quietly. This is not a bug, it is
by design: if nobody is reading, there is no point in producing.

**4. Only stdout enters the pipe.** stderr goes to the terminal unless you say `2>&1`. `cmd 2>&1 | grep x`
gives the error messages to grep as well.

> **🔧 See it on your machine** 🟢 — Seeing the pipe in the fd table
>
> ```
> $ ls -l /proc/self/fd | cat
> total 0
> lrwx------ 1 ubuntu ubuntu 64 Sep 15 17:48 0 -> /dev/pts/0
> l-wx------ 1 ubuntu ubuntu 64 Sep 15 17:48 1 -> pipe:[65627]
> lrwx------ 1 ubuntu ubuntu 64 Sep 15 17:48 2 -> /dev/pts/0
> lr-x------ 1 ubuntu ubuntu 64 Sep 15 17:48 3 -> /proc/3140/fd
> ```
>
> - `/proc/self` → "the process reading this file right now". Here that process is `ls`.
> - `1 -> pipe:[65627]` → `ls`'s stdout is not the terminal but **pipe number 65627.** `cat` reads at the
>   other end.
> - `0` and `2` are still the terminal: the pipe changed only stdout.
> - `3 -> /proc/3140/fd` → the directory `ls` opened in order to list it.

> **🔧 See it on your machine** 🟢 — SIGPIPE and the exit codes of a pipeline
>
> ```
> $ yes | head -2; echo "${PIPESTATUS[@]}"
> y
> y
> 141 0
> ```
>
> - `PIPESTATUS` → in bash, the exit code of **every** command in the pipeline, in order.
> - `141` = 128 + 13. When a process is killed by a signal, the shell reports the exit code as 128 + the
>   signal number; 13 is SIGPIPE.
> - `0` → `head` finished successfully.
> - Plain `$?` gives only the code of the **last** command. In the pipeline `wrong_command | sort`, `$?`
>   returns 0 because `sort` succeeded. In scripts the fix is `set -o pipefail`; we will use it in
>   Phase 10.

**❓ A question that comes to mind: why is `ls` colorful in the terminal but colorless in `ls | cat`?**

Because a program can **ask** whether fd 1 is a terminal (`isatty()`). `ls --color=auto` works like this:
"if fd 1 is a terminal, print color codes; otherwise don't." If color codes were printed into a pipe or a
file, the data read by the next program would contain garbage characters like `\033[01;34m`. You can do
the same check in the shell:

```
$ [ -t 1 ] && echo tty || echo notty
tty
$ ( [ -t 1 ] && echo tty || echo notty ) | cat
notty
```

> **🤔 Think 1.5**
> A cron job runs this line:
>
> ```
> /opt/app/report.sh 2>&1 > /var/log/report.log
> ```
>
> The script sometimes crashes, but there are no error messages in `/var/log/report.log`; only normal
> output.
>
> 1. Where did the error messages go?
> 2. Write the correct line.
> 3. To see the log sorted, a teammate ran `sort /var/log/report.log > /var/log/report.log` and the file
>    was emptied. Why? What is the safe way?
> *(Answer: at the end of the phase)*

---

## 1.4.3 The Unix philosophy: small tools, the power of combining `[concept]`

The people who designed Unix in the 1970s adopted this idea:

1. **Each program should do one thing well.**
2. **Programs should work together** — one's output can be another's input.
3. **The universal interface should be a text stream.**

That is why `sort` only sorts, `uniq` only counts repeats, and `head` only takes the first lines. None of
them has a feature called "find the IP with the most requests". But combined, they answer that question —
and they also answer questions **nobody thought of in advance**.

Let's build this with a real example. A web server's `access.log` file (nginx format):

```
203.0.113.7 - - [14/Sep/2026:10:15:01 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/8.5.0"
198.51.100.23 - - [14/Sep/2026:10:15:02 +0000] "GET /api/orders HTTP/1.1" 200 1534 "-" "Mozilla/5.0"
203.0.113.7 - - [14/Sep/2026:10:15:04 +0000] "POST /api/orders HTTP/1.1" 502 157 "-" "curl/8.5.0"
192.0.2.44 - - [14/Sep/2026:10:15:05 +0000] "GET /health HTTP/1.1" 200 2 "-" "ELB-HealthChecker/2.0"
203.0.113.7 - - [14/Sep/2026:10:15:07 +0000] "GET /api/orders HTTP/1.1" 200 1534 "-" "curl/8.5.0"
198.51.100.23 - - [14/Sep/2026:10:15:09 +0000] "GET /login HTTP/1.1" 404 153 "-" "Mozilla/5.0"
192.0.2.44 - - [14/Sep/2026:10:15:10 +0000] "GET /health HTTP/1.1" 200 2 "-" "ELB-HealthChecker/2.0"
203.0.113.7 - - [14/Sep/2026:10:15:12 +0000] "POST /api/orders HTTP/1.1" 502 157 "-" "curl/8.5.0"
198.51.100.61 - - [14/Sep/2026:10:15:15 +0000] "GET /api/orders HTTP/1.1" 500 89 "-" "Mozilla/5.0"
192.0.2.44 - - [14/Sep/2026:10:15:15 +0000] "GET /health HTTP/1.1" 200 2 "-" "ELB-HealthChecker/2.0"
```

Save these 10 lines on your own machine as `access.log`; all the examples in 1.5 use this file. The fields
are separated by spaces: field 1 is the IP, field 7 the path, field 9 the status code, field 10 the
response size (bytes).

**Question: which IP made the most requests?** Let's build the pipeline step by step. Checking the output
of the previous command with your own eyes at each step is the safest way to write a pipeline.

> **🔧 See it on your machine** 🟢 — Building a pipeline step by step
>
> ```
> $ awk '{print $1}' access.log | head -3          # step 1: only the IPs
> 203.0.113.7
> 198.51.100.23
> 203.0.113.7
>
> $ awk '{print $1}' access.log | sort | uniq -c   # step 2: sort, count
>       3 192.0.2.44
>       2 198.51.100.23
>       1 198.51.100.61
>       4 203.0.113.7
>
> $ awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -3   # step 3: most to least
>       4 203.0.113.7
>       3 192.0.2.44
>       2 198.51.100.23
> ```
>
> - `awk '{print $1}'` → prints field 1 of each line (1.5.2).
> - `sort` → brings identical IPs **next to each other**.
> - `uniq -c` → collapses identical adjacent lines into one line and writes how many there were.
> - `sort -rn` → sorts numerically (`-n`), in reverse (`-r`).
> - `head -3` → the first three.
>
> Answer: `203.0.113.7`, 4 requests. Five small tools, none of which knows this question.

Now we can get to know the tools in 1.5 one by one. But first, let's be honest about one limit of the
philosophy.

**❓ A question that comes to mind: isn't cutting and slicing text fragile?**

Yes, it is fragile. The pipeline above rests on the assumption "the IP is in field 1". If someone adds a
field to the start of the nginx log format, the pipeline produces **a wrong answer without an error**. That
is why modern tools also offer structured output:

- `journalctl -o json` → each log record as a JSON object with field names
- `ip -j addr` → network information as JSON (Phase 7)
- `aws ... --output json` + `jq` → AWS CLI output

The rule: **for interactive diagnosis, text tools are fast and good enough. In a script that will run for
years, use structured output where possible.**

---
---

# 1.5 Text Processing Tools

## 1.5.1 `grep`, `cut`, `sort`, `uniq`, `wc`, `head`, `tail` `[application]`

These seven tools cover 90% of log analysis on a server. They all follow the same contract: **if you give a
file name they read from the file; if you don't, they read from stdin; they write the result to stdout.**
That is why they can be plugged in anywhere in a pipeline.

### `grep` — filter lines

`grep PATTERN file` → prints the lines containing the pattern.

| Option | Meaning | When |
|---|---|---|
| `-i` | Case-insensitive | Catching `ERROR`, `Error` and `error` all at once |
| `-v` | Print lines that **don't match** | Removing noise: `grep -v health` |
| `-n` | Show the line number | Finding the exact place in the file |
| `-c` | Only the **count** of matching lines | "How many 502s are there?" |
| `-w` | Whole-word match | Not catching `errors` and `no_error` when searching for `error` |
| `-E` | Extended regex (`+`, `?`, `\|`, `{n}`) | Patterns like `' 5[0-9]{2} '` |
| `-r` | Search a directory recursively | `grep -r 'listen' /etc/nginx/` |
| `-l` | Print only the matching **file names** | "Which config file has this setting?" |
| `-A n` / `-B n` / `-C n` | n lines after / before / around the match | Seeing the context of an error |

> **🔧 See it on your machine** 🟢 — Searching the log for 5xx
>
> ```
> $ grep -c ' 502 ' access.log
> 2
> $ grep -n -E '" 5[0-9]{2} ' access.log
> 3:203.0.113.7 - - [14/Sep/2026:10:15:04 +0000] "POST /api/orders HTTP/1.1" 502 157 "-" "curl/8.5.0"
> 8:203.0.113.7 - - [14/Sep/2026:10:15:12 +0000] "POST /api/orders HTTP/1.1" 502 157 "-" "curl/8.5.0"
> 9:198.51.100.61 - - [14/Sep/2026:10:15:15 +0000] "GET /api/orders HTTP/1.1" 500 89 "-" "Mozilla/5.0"
> $ grep -n -A1 ' 404 ' access.log
> 6:198.51.100.23 - - [14/Sep/2026:10:15:09 +0000] "GET /login HTTP/1.1" 404 153 "-" "Mozilla/5.0"
> 7-192.0.2.44 - - [14/Sep/2026:10:15:10 +0000] "GET /health HTTP/1.1" 200 2 "-" "ELB-HealthChecker/2.0"
> $ grep -v health access.log | wc -l
> 7
> ```
>
> - `' 502 '` → the spaces are deliberate. If you had written just `502`, lines with a response size of
>   `1502` or with `502` in the path would also match.
> - `'" 5[0-9]{2} '` → a quote, a space, `5` and two digits: the status code field. `{2}` doesn't work
>   without `-E`.
> - In the `-A1` output, `6:` is the matching line and `7-` is a context line. Colon = match, dash =
>   context.
> - `grep -v health | wc -l` → after dropping the load balancer health checks, 7 real requests remain.

**`grep`'s exit code is an answer too:**

| `$?` | Meaning |
|---|---|
| `0` | At least one match was found |
| `1` | **No** match (not an error) |
| `2` | A real error (file missing, no permission, broken regex) |

```
$ grep -q 999 access.log; echo $?
1
$ grep -q 999 nope.log; echo $?
grep: nope.log: No such file or directory
2
```

`-q` prints nothing and answers only with the exit code. In scripts, the question "is there a `FATAL` in the
log file?" is asked with `if grep -q FATAL app.log; then ...` (Phase 10).

### `cut` — cut columns

`cut -d DELIMITER -f FIELDS` → takes specific fields from each line.

```
$ cut -d' ' -f1,9 access.log | head -3
203.0.113.7 200
198.51.100.23 200
203.0.113.7 502
$ cut -d: -f1,7 /etc/passwd | head -3
root:/bin/bash
daemon:/usr/sbin/nologin
bin:/usr/sbin/nologin
```

`cut`'s weak spot: it accepts the delimiter as **exactly one character**. If there are sometimes one and
sometimes three spaces between two fields (as in `ps` or `df` output), `cut` gets the fields wrong. In that
case use `awk`; it treats consecutive spaces as a single delimiter (1.5.2).

### `sort` and `uniq` — sort, count

| Option | Meaning |
|---|---|
| `sort` | Alphabetical (dictionary) sort |
| `sort -n` | **Numeric** sort (`9 < 10`; alphabetically `10 < 9`) |
| `sort -h` | Human-readable size sort (`900K < 1.2M < 2G`) |
| `sort -r` | Reverse order |
| `sort -t' ' -k10,10 -n` | Delimiter is space; sort numerically **by field 10** |
| `sort -u` | Sort and drop duplicates |
| `uniq` | Collapse **adjacent** identical lines into one |
| `uniq -c` | ... and write how many there were |

> **⚠️ Common misconception: "`uniq` finds repeated lines in a file."**
>
> It finds only **consecutive** repeats. `uniq` keeps nothing in memory; it compares each line with the
> previous one. On unsorted data the result is wrong — and there is no error:
>
> ```
> $ awk '{print $1}' access.log | uniq -c
>       1 203.0.113.7
>       1 198.51.100.23
>       1 203.0.113.7
>       1 192.0.2.44
>       1 203.0.113.7
>       ...
> ```
>
> Every IP appears once, because the same IP never came twice in a row. The rule: **always `sort` before
> `uniq`.**

> **🔧 See it on your machine** 🟢 — Status code distribution
>
> ```
> $ awk '{print $9}' access.log | sort | uniq -c | sort -rn
>       6 200
>       2 502
>       1 500
>       1 404
> ```
>
> The question to ask in the first minute of an outage: "how widespread are the errors?" 3 out of 10 requests
> are 5xx. This is **not a single broken client**; three different requests, two from the same IP but one
> from another IP. All of them are on the `/api/orders` path — the problem may be in the backend.

### `wc`, `head`, `tail` — count, take from the start, take from the end

| Command | Meaning |
|---|---|
| `wc -l` | Count lines |
| `head -n 20` | First 20 lines (default 10) |
| `tail -n 50` | Last 50 lines |
| `tail -f file` | **Follow** the end of the file; print new lines as they arrive |
| `tail -F file` | `-f` + if the file is deleted and recreated, **open the new one** |

```
$ wc -l access.log
10 access.log
$ grep -c POST access.log
2
```

`grep PATTERN | wc -l` and `grep -c PATTERN` give the same result; the second has one process fewer.

**The difference between `tail -f` and `tail -F`** looks unimportant at first glance, but it causes real
outages. `tail -f` follows the **fd** of the file it opened. If logrotate renames `app.log` to `app.log.1` at
night and creates a new, empty `app.log`, `tail -f` keeps looking at the old file (now `app.log.1`) and
**never shows** the new lines. `tail -F`, on the other hand, follows the **file name**; when the name points
to a new file, it opens it. Another face of the "name and data are separate" idea from 1.3.1.

---

## 1.5.2 `sed` and `awk` — know what they are `[concept]`

Both are programming languages in their own right. The goal in this phase is not to learn them, but to **be
able to read and write their most common one-line uses.**

### `sed` — the stream editor

`sed` reads each line, applies an operation and prints it. The most common operation is **find and
replace**:

| Usage | Meaning |
|---|---|
| `sed 's/old/new/'` | On each line, change the **first** `old` to `new` |
| `sed 's/old/new/g'` | On each line, change **all** of them |
| `sed -n '3,4p'` | Print only lines 3 and 4 |
| `sed -n '/ 502 /p'` | Print only lines containing the pattern (like `grep`) |
| `sed '/^#/d'` | Delete lines starting with `#` (dropping comments in a config) |
| `sed -i.bak 's/a/b/' file` | Change **the file itself**, taking a `file.bak` backup first |

> **🔧 See it on your machine** 🟢 — Masking an IP in the log
>
> ```
> $ sed 's/203\.0\.113\.7/[HIDDEN]/' access.log | head -2
> [HIDDEN] - - [14/Sep/2026:10:15:01 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/8.5.0"
> 198.51.100.23 - - [14/Sep/2026:10:15:02 +0000] "GET /api/orders HTTP/1.1" 200 1534 "-" "Mozilla/5.0"
> ```
>
> - `\.` → in regex, `.` means "any character"; a real dot needs escaping.
> - The original file **did not change.** `sed` wrote the result to stdout. You can mask a log like this
>   before sharing it with a support team.

> **🔧 See it on your machine** 🔴 — Changing a file in place (`-i`)
>
> ```
> $ cp access.log copy.log
> $ sed -i.bak 's/HTTP\/1.1/HTTP\/2/g' copy.log
> $ ls copy.log*
> copy.log  copy.log.bak
> $ head -1 copy.log
> 203.0.113.7 - - [14/Sep/2026:10:15:01 +0000] "GET / HTTP/2" 200 612 "-" "curl/8.5.0"
> ```
>
> - `-i` changes the file **permanently**; if you give a `.bak` suffix, it takes a backup first. On a real
>   config, **never use `-i` without a suffix.**
> - This example is safe because it works on a copy; but `-i` on a real file is 🔴.
> - **Undo:** `mv copy.log.bak copy.log`. Cleanup when done: `rm copy.log copy.log.bak`.
> - Run it without `-i` first and check the output with your own eyes, **then** add `-i`.

### `awk` — the language that thinks in fields

`awk` automatically splits each line into fields: `$1`, `$2`, ... `$NF` (the last field), `$0` (the whole
line). Its default delimiter is **one or more spaces** — the thing `cut` can't do.

Structure: `awk 'CONDITION { ACTION }'`. If the condition is true, the action runs; with no condition it runs
on every line. The `END { ... }` block runs once after all lines are done.

| Usage | Meaning |
|---|---|
| `awk '{print $1}'` | Print field 1 |
| `awk '$9 >= 500 {print $9, $7}'` | If field 9 is 500 or above, print the code and the path |
| `awk '{s += $10} END {print s}'` | Sum field 10, print at the end |
| `awk -F: '$3 >= 1000 {print $1}'` | Delimiter `:`; if field 3 is 1000+, print field 1 |
| `awk '{print $NF}'` | Print the last field |

> **🔧 See it on your machine** 🟢 — Extracting numbers from a log
>
> ```
> $ awk '$9 >= 500 {print $9, $7}' access.log
> 502 /api/orders
> 502 /api/orders
> 500 /api/orders
> $ awk '$9 >= 500 {n++} END {print n}' access.log
> 3
> $ awk '{s+=$10; n++} END {printf "%d requests, average %.1f bytes\n", n, s/n}' access.log
> 10 requests, average 424.2 bytes
> $ awk -F: '$3 >= 1000 {print $1, $3, $7}' /etc/passwd
> ubuntu 1000 /bin/bash
> nobody 65534 /usr/sbin/nologin
> ```
>
> - `$9 >= 500` → awk compares the field as a number. The "greater than" question that `grep` can't ask.
> - `n++` → increment the counter on each line where the condition is true; print it in `END`.
> - `/etc/passwd` → users with a UID of 1000 and above are real (human) users. `nobody` is a special
>   exception. In Phase 2.1 we will read this file field by field.

**❓ A question that comes to mind: should I use `grep | awk | sed`, or do everything with `awk`?**

Choose the readable one. `grep ERROR app.log | awk '{print $1}'` is a pipeline anyone will understand.
`awk '/ERROR/ {print $1}' app.log` starts one process fewer, but requires the next person to know awk. In the
terminal it doesn't matter; in a shared script, think of the reader.

---

## 1.5.3 The cloud connection: reading logs remotely and the journal `[application]`

### A single command over SSH

Without logging in to a server, you can run a command remotely and get the output on your own machine:

```
$ ssh ubuntu@10.0.1.25 'sudo tail -n 200 /var/log/nginx/error.log' | grep -i upstream
```

Here it matters **which side the pipe runs on**:

- Everything **inside** the quotes runs on the remote server: `sudo tail`.
- The `| grep` **outside** the quotes runs on your machine.

200 lines come to you over the network, then get filtered. If the log were 5 GB and you had written `cat`
instead of `tail`, you would pull 5 GB over the network and filter it on your own machine. The rule: **filter
where the data is:**

```
$ ssh ubuntu@10.0.1.25 'sudo grep -i upstream /var/log/nginx/error.log | tail -n 20'
```

### `journalctl` — systemd's log

On modern distros, most services write their logs not to a file but to the **journal**. On Amazon Linux 2023
almost everything is only here. In Phase 5 we will see how the journal works; for now, learn to read it:

| Command | Meaning |
|---|---|
| `journalctl -u nginx` | Only the log of the `nginx` service |
| `journalctl -p err` | Only priority `err` and more serious |
| `journalctl -n 50` | The last 50 entries |
| `journalctl -f` | Follow live (like `tail -f`) |
| `journalctl --since "10 min ago"` | The last 10 minutes |
| `journalctl -b` | Only since this boot |
| `journalctl --no-pager` | Print plainly without paging (automatic in pipes) |

> **🔧 See it on your machine** 🟢 — Recent errors
>
> ```
> $ journalctl -p err -n 3 --no-pager
> Sep 15 09:12:40 ip-172-31-20-14 nginx[1210]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
> Sep 15 09:12:40 ip-172-31-20-14 systemd[1]: Failed to start nginx.service - A high performance web server.
> Sep 15 09:14:02 ip-172-31-20-14 sshd[1502]: error: kex_exchange_identification: Connection closed by remote host
> ```
>
> The structure of each line: **time · hostname · program[PID] · message.** The lines on your machine will
> be different; what matters is being able to read the fields. Thanks to this structure, text tools work
> on the journal too:
>
> ```
> $ journalctl -p err --since today --no-pager | awk '{print $5}' | sort | uniq -c | sort -rn
>       4 nginx[1210]:
>       2 systemd[1]:
>       1 sshd[1502]:
> ```
>
> "Which program printed the most errors today?" — field 5 is the program name.
>
> As a normal user, `journalctl` may show only your own entries; for all system logs you need `sudo` or
> membership in the `adm` / `systemd-journal` group (Phase 2).

**The three commands of Lab 1 in the roadmap** are now familiar:

```
$ journalctl -p err -n 50 --no-pager | grep -v sshd | awk '{print $5}' | sort | uniq -c
$ sudo find /var/log -type f -size +50M -exec ls -lh {} +
$ df -h /var
```

The first answers "among the last 50 errors, who is there apart from SSH noise?", the second "which log file
has swollen?", the third "how full is the disk `/var` sits on?"

---

## 1.5.4 When it breaks: "I can't find the error in the log" `[application]`

The application is failing, you type `grep -i error /var/log/app.log | tail` and **nothing comes out.** Is
the error really not there, or are you looking in the wrong place? The checklist that goes through an
experienced engineer's mind:

| # | Possible cause | How you can tell | Fix |
|---|---|---|---|
| 1 | **The log was rotated** — the line you want is in yesterday's file | `ls -ltr /var/log/app.log*` | `grep` → the `.1` file; `zgrep` for `.gz` |
| 2 | **Wrong file** — the application writes elsewhere or to the journal | `ls -l /proc/<PID>/fd \| grep log` | The right file, or `journalctl -u app` |
| 3 | **stderr is not captured** — the error isn't in the file stdout was redirected to | Is there a `2>&1` in the service definition? | The ordering rule from 1.4.2 |
| 4 | **Different word** — not `error` but `ERR`, `FATAL`, `Traceback`, `panic`, `exception` | `grep -iE 'err\|fatal\|traceback\|panic\|exception'` | Look at the application's log format |
| 5 | **Wrong time window** — the error was 3 hours ago, you are looking at the last 10 lines | A time-pattern `grep` instead of `tail` | `journalctl --since` |
| 6 | **Buffering** — the application wrote the log but it hasn't reached the disk yet | Lines arrive in batches | Phase 0.2.2: the user space buffer |
| 7 | **Multi-line error** — there is an `error` line but the real information is in the lines below | Stack trace | `grep -A 20` |
| 8 | **Permission** — you can't read the file, `grep` went quietly to `2>/dev/null` | `$?` = 2 | `sudo` (Phase 2) |

> **🔧 See it on your machine** 🟢 — Searching rotated logs
>
> ```
> $ ls -ltr /var/log/syslog*
> -rw-r----- 1 syslog adm  301512 Sep 12 00:00 /var/log/syslog.4.gz
> -rw-r----- 1 syslog adm  288104 Sep 13 00:00 /var/log/syslog.3.gz
> -rw-r----- 1 syslog adm  312990 Sep 14 00:00 /var/log/syslog.2.gz
> -rw-r----- 1 syslog adm 2841203 Sep 15 00:00 /var/log/syslog.1
> -rw-r----- 1 syslog adm  912044 Sep 15 17:50 /var/log/syslog
> $ sudo zgrep -c 'Out of memory' /var/log/syslog*
> /var/log/syslog:0
> /var/log/syslog.1:0
> /var/log/syslog.2.gz:3
> /var/log/syslog.3.gz:0
> /var/log/syslog.4.gz:0
> ```
>
> - The files were rotated at midnight; the oldest is at the top (`-ltr`).
> - `zgrep` searches compressed and plain files together. Plain `grep` can't read inside a `.gz` file — and
>   finds no match **without an error**.
> - The 3 OOM events are in the file from 2 days ago. If you had looked only at `syslog`, you would have said
>   "OOM never happened".

> **🤔 Think 1.6**
> At 23:30 in the evening, a teammate leaves this command running in a `tmux` session to watch a live problem,
> and goes home:
>
> ```
> tail -f /var/log/app/app.log | grep --line-buffered ERROR >> /home/ubuntu/errors.txt
> ```
>
> In the morning, `errors.txt` has only the errors between 23:30 and 00:00. Yet the application log certainly
> had lots of `ERROR` lines at 03:00. The `tail` process is still running and hasn't printed an error.
>
> 1. What happened at midnight?
> 2. Which file must `tail` be following, and how do you prove it?
> 3. How do you fix the command?
> *(Answer: at the end of the phase)*

---
---

# 1.6 When this phase breaks — failure signatures

Most failures in this phase are not "the system is broken" but the gap between your expectation and **the
shell or the tool doing exactly what it was told**. Behind every row of the table is a mechanism you built in
this phase.

| Symptom | Likely mechanism | Where it was covered | First place to look |
|---|---|---|---|
| The script works in the terminal, `command not found` in cron/systemd | Different PATH; cron uses `/usr/bin:/bin` | 1.1.2, Answer 1.1 | `type -a <command>`, a full path or a PATH definition in the script |
| The "wrong version" runs on the server | Another copy in a directory earlier in PATH | 1.1.2 | `type -a <command>`, `hash -r` |
| `find ... -name *.log` sometimes finds nothing, sometimes says `paths must precede expression` | The shell expanded the pattern | 1.1.2, Answer 1.2 | Quote the pattern: `'*.log'` |
| `cd` works inside a script, but the directory hasn't changed after the script ends | The script ran in a child process | 1.1.2 | `source script.sh`, or change the expectation |
| Your change to `/etc/resolv.conf` or a config disappears | The file is a link and another service regenerates it; or a `.d` directory overrides it | 1.2.2 | `ls -l`, the `.d` directory |
| Disk at `100%`, `du` total much lower | A deleted but still open file | 1.3.1 | `sudo lsof +L1` |
| "No space left" when writing a big file to `/tmp`, root disk is empty | `/tmp` is tmpfs (RAM) | 1.2.1 | `df -hT /tmp` |
| `mv` takes minutes to move a directory | Moving to a different file system = copy + delete | 1.3.1 | `df <source> <target>` |
| File empty after `sort f > f` (or `sed ... f > f`) | `>` truncated the file before the command started | 1.4.2 | Write to a temporary file, then `mv` |
| No error messages in the cron log file | `2>&1 > log` ordering; stderr was not captured | 1.4.2, Answer 1.5 | `> log 2>&1` |
| `sudo echo ... > /etc/x` → `Permission denied` | The shell running as the normal user does the redirection | 1.4.2 | `sudo tee` |
| The pipeline failed but `$?` = 0 | `$?` is only the last command's code | 1.4.2 | `PIPESTATUS`, `set -o pipefail` |
| `uniq -c` counts every line as 1 | The input isn't sorted | 1.5.1 | `sort` first |
| `sort` puts `10` before `9`, `2G` before `900M` | Alphabetical sort | 1.5.1 | `sort -n` / `sort -h` |
| `grep` can't find the error | Rotation, wrong file, different word, `.gz` | 1.5.4 | `ls -ltr log*`, `zgrep`, `journalctl` |
| `tail -f` shows no new lines after midnight | Logrotate replaced the file, `tail -f` follows the old fd | 1.5.1, Answer 1.6 | `tail -F` |
| No `/var/log/syslog` on Amazon Linux | rsyslog isn't installed, everything is in the journal | 1.2.2 | `journalctl` |

> **The lesson from this table:** almost none of the failures in this phase give an error message. `uniq`
> counts wrong, `>` quietly empties the file, `grep` says "not found" and falls silent. So build two habits:
> **(1) build a pipeline step by step and check each step's output with your own eyes; (2) treat "nothing came
> out" not as an answer but as a question** — "is it really not there, or am I looking in the wrong place?"

---

# Phase 1 — Answers to the think questions

## Answer 1.1 — A script that works in the terminal says `command not found` in cron

**Question:** The `aws` CLI is in `/usr/local/bin`. The script works in the terminal but says
`aws: command not found` in cron. Why, and what is the fix?

**Cause: cron's PATH is not your PATH.**

When you log in to a terminal, bash reads a series of startup files (`/etc/profile`, `~/.profile`,
`~/.bashrc`) and these set up PATH. Your PATH looks something like this:

```
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

cron, on the other hand, does not log in, does **not read** these files, and starts user crontabs with a
very short default PATH:

```
/usr/bin:/bin
```

`/usr/local/bin` is not in this list. The shell applies the lookup order from 1.1.2: no alias, not a
builtin, no `aws` in the two PATH directories → `command not found`. The same reason applies to systemd
services: systemd also starts them with its own, limited environment.

To prove it, add a temporary line to cron and dump the environment into a file:

```
* * * * * env > /tmp/cron-env.txt
```

A minute later, `grep PATH /tmp/cron-env.txt` shows the PATH cron really sees. (Delete the line
afterwards.)

**Fixes (from most robust to least robust):**

1. **Use the full path inside the script:** `/usr/local/bin/aws s3 cp ...`. No dependency on the
   environment at all.
2. **Define PATH yourself at the top of the script:**
   ```
   #!/bin/bash
   PATH=/usr/local/bin:/usr/bin:/bin
   ```
   Now the script behaves the same no matter who runs it.
3. **Add a PATH line at the top of the crontab:** `PATH=/usr/local/bin:/usr/bin:/bin`. It works, but the
   problem comes back when the script runs somewhere else (such as a systemd timer).

**A "fix" to avoid:** adding `source ~/.bashrc` to the crontab line. Ubuntu's `.bashrc` exits immediately
in a non-interactive shell; it also makes the script depend on your personal settings.

**The general lesson:** the sentence "it works for me" usually means "it works in **my environment**".
Cron, systemd, a CI pipeline, a container and running a command remotely over SSH **all** start with a
different environment.

**Related section:** 1.1.2 · **Continues in:** Phase 5 (the systemd environment), Phase 10 (a robust
environment in scripts)

---

## Answer 1.2 — Three different behaviors of `find /var/log -name *.log`

**Question:** The same command finds nothing when there is one `.log` in the directory, errors when there
are two, and works properly when there are none. Why?

**Cause: `*.log` is expanded by the shell, not by `find` — and it does so based on the directory you are
in.**

Remember Step 2 in 1.1.2: a glob is resolved **before** the program runs, **in the current directory**.
`find` never sees the star; it sees what the shell gave it.

**Case 1 — there is one `.log` in the directory (`notes.log`):**

```
What you typed:    find /var/log -name *.log
What find got:     find /var/log -name notes.log
```

`find` searches under `/var/log` for a file named `notes.log`. There is none. It quietly prints nothing.

**Case 2 — there are two `.log` files in the directory (`a.log`, `b.log`):**

```
What find got:     find /var/log -name a.log b.log
find: paths must precede expression: `b.log'
find: possible unquoted pattern after predicate `-name'?
```

`-name` takes a single argument. `b.log` is now an extra argument and `find` can't make sense of it. Even
`find` itself gives a hint: "**unquoted pattern**?"

**Case 3 — there is no `.log` in the directory:**

When bash finds no match, it leaves the star **as it is** (1.1.2, "If a glob matches no file"). `find`
really receives the `*.log` pattern and works as you expect.

The third case is the most dangerous: the command is **wrong but looks like it works**. You put the same
line into a script; one day the script is run from a directory that contains a `.log` file, and it quietly
gives a wrong result.

**Fix:** **always quote** a pattern meant for the program:

```
$ find /var/log -name '*.log'
```

Single quotes tell the shell "don't touch these characters, pass them as they are". The same rule applies
to `grep -E 'a|b'`, `awk '{print $1}'` and `sed 's/a/b/'`: quote every argument containing special
characters.

**Related section:** 1.1.2, 1.3.2 · **Continues in:** Phase 10 (quoting in scripts)

---

## Answer 1.3 — Root disk at 100%: where to start?

**Question:** Which two directories do you look at first? Does `du -sh /*` walk through `/proc`, and is
that a problem? What happens if a separate `/data` disk is attached?

**1. The first two directories: `/var/log` and `/var/lib`.**

The logic of the FHS gives the answer: fixed data (`/usr`, `/etc`) doesn't grow by itself; unless you
install packages, its size stays the same. **Changing data is in `/var`.**

- `/var/log` → an application out of control can print thousands of lines per second and fill the disk
  overnight. The most common cause.
- `/var/lib` → Docker images and container layers (`/var/lib/docker`), database files, old snap
  revisions.

Then come `/tmp` and `/home` (a big dump or archive someone left behind).

```
$ sudo du -xh --max-depth=1 / | sort -h | tail -5
$ sudo du -xh --max-depth=1 /var | sort -h | tail -5
$ sudo find / -xdev -type f -size +500M -exec ls -lh {} +
```

**2. `du -sh /*` has two problems.**

- **It also goes into `/proc`, `/sys` and `/run`.** Files in `/proc` take no space on disk, but `du` tries
  to open thousands of files to walk them. It slows down, fills the screen with permission errors, and
  may produce meaningless numbers for some virtual files.
- **It also counts the `/data` disk.** If 400 GB of the 500 GB `/data` disk is full, `du` shows `/data`
  as the biggest directory. But you are looking for why **the root disk** is full; `/data` is a separate
  disk and its being full does not affect the root disk. You follow the wrong trail.

The fix for both is **`-x`** (`du`) / **`-xdev`** (`find`): "stay on the file system you started on". If
`/` is on the root disk, `/proc`, `/sys`, `/run` and `/data` are skipped because they are separate file
systems.

**3. One step further: if `du` shows little and `df` shows a lot** — for example `df` says 29 GB used and
`du -x /` finds 12 GB — the missing 17 GB is most likely in **deleted but open** files (1.3.1).
`sudo lsof +L1`.

**Why it matters:** when the disk fills up, deleting things at random with `rm -rf` in a panic both loses
data and (if the file is open) frees no space. The right order: which disk with `df` → which directory with
`du -x` → which file with `find -xdev` → hidden space with `lsof +L1`.

**Related section:** 1.2.1, 1.3.1, 1.3.2 · **Continues in:** Phase 6 (file systems, running out of
inodes), Phase 11

---

## Answer 1.4 — The problems in `find / -name *.log -mtime +7 -delete`

**Question:** Find at least four problems in this cleanup command.

**Problem 1 — The pattern is unquoted.** The same as Answer 1.2. If there is a `.log` file in the directory
cron runs in (usually the user's home directory), the pattern turns into that file's name. In the best case
nothing is deleted; if there are two `.log` files, the command errors. **Fix:** `-name '*.log'`.

**Problem 2 — It starts from the root directory (`/`) with no file system boundary.** It walks every disk,
mounted network file systems (NFS, EFS) and `/proc`. It is slow; if an EFS disk is mounted it may delete
**other servers'** logs. **Fix:** narrow the target directory (`/var/log/app`) and add `-xdev`.

**Problem 3 — There is no `-type f`.** If there is a **directory** whose name ends in `.log` (a badly named
directory like `/var/log/nginx.log/`), `find` matches it too. If it is not empty `-delete` fails; if it is
empty it deletes it. **Fix:** `-type f`.

**Problem 4 — You never saw which files would be deleted.** `-delete` cannot be undone; there is no trash
bin. A log file that is **actively being written to** can also be `-mtime +7` (unchanged for 7 days but
held open by a process). If you delete it, no space is freed (1.3.1) and the application loses its log.
**Fix:** run it without `-delete` first, read the list, then add it.

**Problem 5 — It does logrotate's job by hand.** Most files under `/var/log` are already rotated and deleted
by **logrotate**. Deleting by hand can conflict with logrotate's state file. The right fix is usually to
correct the logrotate rule (Phase 8).

**Problem 6 — No errors and no record.** When it runs in cron, permission errors and what was deleted are
written nowhere. **Fix:** add `-print` and redirect the output to a log file.

**A safer version:**

```
find /var/log/app -xdev -type f -name '*.log.*' -mtime +7 -print -delete >> /var/log/cleanup.log 2>&1
```

It deletes only in the application's own directory, only files, only old rotated copies (like `app.log.3`;
not the active `app.log`), and it records what it deleted and any errors.

**Related section:** 1.1.2, 1.3.1, 1.3.2, 1.4.2 · **Continues in:** Phase 8 (logrotate), Phase 10 (safe
scripts)

---

## Answer 1.5 — No errors in the cron log; `sort f > f` emptied the file

**Question:** Where did `report.sh 2>&1 > /var/log/report.log` send the errors? What is the correct line?
Why did `sort` empty the file?

**1. The errors went to cron's own stderr.**

Redirections are applied from left to right (1.4.2):

```
start:          fd1 → (cron's output)    fd2 → (cron's output)
2>&1:           fd2 → where fd1 points RIGHT NOW = (cron's output)
> report.log:   fd1 → /var/log/report.log
result:         fd1 → report.log         fd2 → (cron's output)
```

cron either tries to send a job's uncaptured output to the user by **email** (lost if mail isn't set up on
the server), or writes it to the journal. So the error messages are most likely nowhere.

**2. The correct line:**

```
/opt/app/report.sh > /var/log/report.log 2>&1
```

First fd 1 to the file, **then** fd 2 to "where fd 1 points" — that is, the same file. If you don't want to
wipe the old log on every run, use `>>`.

**3. `sort f > f` emptied the file, because `>` truncates the file before the command starts.**

The order: the shell opens `report.log` with `O_TRUNC` (the file becomes 0 bytes) → fork + exec → `sort`
reads the empty file → writes empty output. The data was gone before `sort` started.

**Safe ways:**

```
$ sort report.log > report.log.tmp && mv report.log.tmp report.log
$ sort -o report.log report.log        # a feature of sort itself: reads first, then writes
```

The first is the general pattern that works with any tool: **write to a temporary file, and if that
succeeds (`&&`), put it in place.** `mv` is atomic within the same file system (1.3.1): the file is either
in its old or its new state, never half-done. (Note: if a running service is still writing to this log
file, after the `mv` the service keeps writing to the old inode — the same mechanism as Answer 1.6.)

**Related section:** 1.4.2 · **Continues in:** Phase 5 (systemd service logs), Phase 10

---

## Answer 1.6 — `tail -f` went silent after midnight

**Question:** `tail -f app.log | grep ERROR >> errors.txt` caught nothing after midnight, but the process is
running. What happened, how do you prove it, and how do you fix it?

**1. logrotate ran at midnight.**

Default logrotate behavior (`create` mode):

```
00:00  mv app.log app.log.1             ← same file, new name (same inode)
00:00  create a new, empty app.log      ← new file, new inode
00:00  send the application a "reopen your log" signal
```

The application receives the signal and starts writing to the **new** `app.log`. But when `tail -f`
started, it opened the file and got an **fd**. An fd is tied not to a name but to **the file itself** (the
inode). Since `mv` only changed the name, `tail`'s fd still points to the old file — the one that now
carries the name `app.log.1` and that nobody writes to. `tail` gives no error; its file simply isn't growing
any more.

**2. Proof: look at `tail`'s fd table.**

```
$ pgrep -a tail
4120 tail -f /var/log/app/app.log
$ ls -l /proc/4120/fd | grep app
lr-x------ 1 ubuntu ubuntu 64 Sep 15 08:05 3 -> /var/log/app/app.log.1
```

The command line says `app.log`, but fd 3 points to `app.log.1`. Live proof of the "an fd is tied to an
identity" idea from Phase 0. (If logrotate has compressed and deleted `.1`, you see `(deleted)` on this
line.)

**3. The fix: `tail -F`.**

```
tail -F /var/log/app/app.log | grep --line-buffered ERROR >> /home/ubuntu/errors.txt
```

`-F` = `--follow=name --retry`: `tail` periodically checks the **name**; when the name points to a new
file, it opens it and prints `tail: '/var/log/app/app.log' has been replaced; following new file`.

A detail that was already right in the question: `grep --line-buffered`. When `grep`'s output goes to a
file it writes lines in batches (buffering, Phase 0.2.2); `--line-buffered` makes it write each line
immediately. Without it, even if errors were caught during the night, they would reach `errors.txt` late.

**The long-term fix:** for permanent monitoring, use not a pipeline forgotten in `tmux` but a log
collection agent (CloudWatch Agent, Fluent Bit). These agents already follow rotation correctly
(Phase 11).

**Related section:** 1.3.1, 1.5.1 · **Continues in:** Phase 8 (logrotate), Phase 11 (log collection)

---

# Phase 1 — Frequently asked questions

### Q1. Terminal, console, shell, TTY — aren't they all the same thing?

In everyday language yes, technically no. The **terminal** is the program that draws the screen (or, in the
past, a physical device); the **shell** is the program that interprets commands; the **console** is the main
terminal connected directly to the machine ("EC2 Serial Console" on EC2); and **TTY** is the name the kernel
uses for terminal devices (`/dev/tty1`, `/dev/pts/0`). The distinction helps when hunting faults: "I can't get
in over SSH but I can get in from the serial console" tells you the problem is not in the shell but in the
network or SSH.

### Q2. Why does my script starting with `#!/bin/sh` say `[[: not found` on Ubuntu?

Because on Ubuntu `/bin/sh` is not bash but **dash** (`ls -l /bin/sh` → `sh -> dash`). `[[ ]]`, arrays,
`{1..10}` and the `function` keyword are specific to bash ("bashisms"). If your script uses bash features,
make the first line `#!/bin/bash`. Running it with `sh script.sh` also ignores the first line and uses dash;
use `bash script.sh` or `./script.sh`. On Amazon Linux `/bin/sh` points to bash, so the same script works
there — the classic cause of the "broken on Ubuntu, works on Amazon Linux" failure.

### Q3. What is the difference between `man` and `--help`? Which should I look at?

`command --help` is a short summary printed by the program itself; it is fast and enough for "which option was
it?" `man command` is the full manual; inside it you search with `/`, go to the next match with `n` and quit
with `q`. Builtins have no man page: `help cd`. Minimal cloud images may not have man pages installed (the
`unminimize` command brings them back on Ubuntu). The `tldr` tool is also handy for quick examples.

### Q4. Ctrl+C, Ctrl+Z, Ctrl+D — what are the differences?

| Key | What it does | Mechanism |
|---|---|---|
| **Ctrl+C** | Stops the running command | The terminal sends the **SIGINT** signal to the foreground process |
| **Ctrl+Z** | **Suspends** the command (doesn't kill it) | **SIGTSTP**; continue with `fg`, continue in the background with `bg` |
| **Ctrl+D** | "End of input" | Not a signal; end of file (EOF) on stdin. On an empty prompt it exits the shell |
| **Ctrl+R** | Search backwards in history | A bash feature; Ctrl+R again for the previous match |
| **Ctrl+L** | Clear the screen | Same as `clear` |

The Ctrl+Z trap: if you press Ctrl+Z to "get out of" `vim` and then leave SSH, vim stays suspended in the
background and leaves a lock (`.swp`) on the file. `jobs` shows suspended jobs. We will look at signals in
detail in Phase 3.

### Q5. I accidentally deleted a file with `rm`. Can I get it back?

The general answer: **no**, there is no trash bin. The only exception is the mechanism from 1.3.1: if a process
still holds the file open, the data is still on disk and can be copied through `/proc`:

```
$ sudo lsof +L1 | grep mydata
app     2210 ubuntu  4w  REG  259,1  81234  0  262400 /srv/app/mydata.db (deleted)
$ sudo cp /proc/2210/fd/4 /srv/app/mydata.db.recovered
```

If the process exits, that chance is gone too. The real fix in the cloud is an **EBS snapshot** taken in
advance, or S3 versioning. Checking the target with `ls` before `rm` and using `rm -i` in critical directories
is the cheapest insurance.

### Q6. `vim` opened and I can't get out.

Press `Esc` (return to command mode), then:

- `:q` + Enter → quit (if there are no changes)
- `:q!` + Enter → quit **without saving**
- `:wq` + Enter → save and quit

`git commit` or `crontab -e` can drop you into vim when you don't expect it. To change the default editor:
`export EDITOR=nano` (add it to `~/.bashrc` to make it permanent), or on Ubuntu
`sudo update-alternatives --config editor`.

### Q7. Why doesn't my `history` show some commands? I typed a password on the command line — what now?

Bash writes history to `~/.bash_history` **when the session closes**; two sessions open at the same time don't
see each other's history right away. On Ubuntu, commands starting with a space are not written to history
(`HISTCONTROL=ignoreboth`).

If you typed a password or token on the command line, it may have landed in three places: `~/.bash_history`,
`/proc/<PID>/cmdline` for as long as the command ran (**everyone on the machine** can see it with `ps aux`),
and `auth.log` if you used `sudo`. To remove it from history, use `history -d <number>`; but the real
precaution is keeping secrets not on the command line but in an environment variable or a file (Phase 9, and
AWS Secrets Manager in Phase 12).

---

# Phase 1 — Test yourself

Do not go back to the sections before writing your answers. If you can, answer the questions in Part C by trying
them on your machine. The numbers of the questions you struggle with point to the sections to review.

**Part A — Basics (1–6)**

1. What is the difference between a terminal and a shell? Explain the four parts of the prompt
   `ubuntu@ip-172-31-20-14:~$`.
2. In what order does the shell look up a command name? Which command tells you where a command comes from?
3. What do the `/etc`, `/var/log`, `/var/lib` and `/usr/local/bin` directories hold? Give an example file for
   each.
4. What do the `/proc` and `/run` directories have in common? What does this mean at a reboot?
5. What do fd 0, 1 and 2 represent? Why is stderr a separate channel from stdout?
6. What do `grep`'s exit codes 0, 1 and 2 mean?

**Part B — Mechanism (7–12)**

7. Why can't `cd` be a separate program (`/usr/bin/cd`)? Explain with the process model.
8. Describe, in order, the steps the shell takes when you type `ls -l /var/log` and press Enter. Which syscalls
   are used?
9. Explain the difference between `cmd > out 2>&1` and `cmd 2>&1 > out` in terms of the fd table.
10. What does the kernel create when you type `cmd1 | cmd2`? Do the two commands run one after the other or at
    the same time? What happens to `cmd1` if `cmd2` exits early?
11. Why does moving a 50 GB file with `mv` within the same disk finish instantly, while moving it to another
    disk takes minutes?
12. Why does `sudo echo "x" > /etc/file` give `Permission denied`? What is the right method, and why does it
    work?

**Part C — Application and reasoning (13–18)**

13. In the `access.log` file (1.4.3), find with a single pipeline how many of the requests to the `/api/orders`
    path returned 5xx. What is the expected result?
14. On a server, `df -h /` shows 97% used, but `sudo du -xsh /` finds only 11 GB (less than half the disk).
    What is the most likely cause, and which command confirms it?
15. Write the command that lists the files under `/etc` changed within the last hour.
16. A teammate runs `awk '{print $1}' access.log | uniq -c | sort -rn` and says "every IP came only once, there
    is no attack". What did they do wrong?
17. On an Amazon Linux 2023 instance you type `sudo grep -i 'failed password' /var/log/auth.log` and get
    `No such file or directory`. How do you find the failed SSH login attempts?
18. A service writes its logs to `/var/log/app/app.log`. While watching a failure live, you want to see only
    lines containing `ERROR` or `FATAL`, and you want this watching not to be affected by the midnight
    logrotate. Write the command and explain the two critical options.

---

## Answer key

**1.** A terminal is the program that takes keyboard input and draws characters on the screen; it doesn't
understand commands. A shell is the program that interprets the typed line and runs programs (bash). Prompt:
`ubuntu` = user, `ip-172-31-20-14` = hostname, `~` = current directory (the home directory), `$` = normal user
(`#` = root). · *1.1.1*

**2.** Alias → function → builtin → PATH (left to right). `type -a <command>` (or `command -v`). · *1.1.2*

**3.** `/etc` = config (`/etc/ssh/sshd_config`); `/var/log` = logs (`/var/log/auth.log`); `/var/lib` = programs'
persistent state data (`/var/lib/docker`, `/var/lib/mysql`); `/usr/local/bin` = programs installed by hand
outside the package manager (the `aws` CLI). · *1.2.1*

**4.** Neither is on disk: `/proc` is a virtual file system produced by the kernel, `/run` is a tmpfs in RAM. At
a reboot the contents of both are built from scratch; a file put in `/run` does not survive a reboot. · *1.2.1*

**5.** 0 = stdin (input), 1 = stdout (result), 2 = stderr (error and diagnostic messages). stderr is separate
because when output is redirected to a file or another program, error messages must not get mixed into the data,
and the user must keep seeing them. · *1.4.1*

**6.** 0 = at least one match, 1 = no match (not an error), 2 = a real error (file missing, no permission, broken
pattern). · *1.5.1*

**7.** Every process has its own working directory, and a process cannot change another's. If `cd` were a
separate program, the shell would run it as a child process; the child would change its own directory and die,
and the parent shell's directory would stay the same. That is why `cd` has to be a builtin. · *1.1.2*

**8.** (1) Split the line into words (`ls`, `-l`, `/var/log`); (2) variable and glob expansion (none here);
(3) look up `ls`: an alias (`ls --color=auto`) is found and expanded, then `/usr/bin/ls` in PATH; (4) copy itself
with `fork` (`clone` on Linux); (5) the child turns into `ls` with `execve("/usr/bin/ls", ...)`; (6) the parent
waits with `wait4`; (7) the exit code is stored in `$?`, a new prompt appears. · *1.1.2*

**9.** `> out 2>&1`: first fd 1 → `out`, then fd 2 → where fd 1 points = `out`; both in the file. `2>&1 > out`:
first fd 2 → where fd 1 points at that moment = the terminal, then fd 1 → `out`; the errors stay on the terminal.
Redirections are applied from left to right, and `2>&1` is a copy, not a permanent link. · *1.4.2*

**10.** The kernel creates a **pipe buffer** in RAM (64 KiB by default) and two fds (a write end and a read
end). The two commands run **at the same time**; if the buffer is full the writer waits, if it is empty the
reader waits. If `cmd2` exits early, `cmd1` receives **SIGPIPE** on its next write attempt and dies (exit code
141). · *1.4.2*

**11.** On the same file system `mv` only changes the name entry in the directory; it doesn't touch the data
blocks (the inode stays the same). When moving to a different file system, the data is copied byte by byte and
the source is deleted. · *1.3.1*

**12.** The `>` redirection is done not by `sudo` but by the **shell** running as the normal user; the file is
opened with your permissions before `sudo` starts. The right way: `echo "x" | sudo tee /etc/file > /dev/null`.
Here the program that opens the file is `tee`, and it runs as root. · *1.4.2*

**13.**

```
$ awk '$7 == "/api/orders" && $9 >= 500' access.log | wc -l
3
```

(or `grep '/api/orders' access.log | awk '$9 >= 500' | wc -l`). Two 502s, one 500. · *1.5.1, 1.5.2*

**14.** **Deleted but still open files** — usually a big log that a service writes to and someone deleted with
`rm`. `du` walks names and can't find it; `df` shows the kernel's real block usage. Confirm with:
`sudo lsof +L1`. Fix: restart the service involved (or logrotate's reopen signal). · *1.3.1, Answer 1.3*

**15.** `sudo find /etc -type f -mmin -60` · *1.3.2*

**16.** `uniq` counts only **consecutive** repeats; because the input wasn't sorted, every IP shows as 1. The
right way: `awk '{print $1}' access.log | sort | uniq -c | sort -rn`. · *1.5.1*

**17.** On Amazon Linux 2023 rsyslog is not installed by default; there is no `auth.log`, the logs are in the
**journal**: `sudo journalctl -u sshd --since today | grep -i 'failed'` (or `journalctl _COMM=sshd`).
· *1.2.2, 1.5.3*

**18.**

```
tail -F /var/log/app/app.log | grep --line-buffered -E 'ERROR|FATAL'
```

`tail -F`: follows the file **by name**, not by fd; when logrotate creates a new file, it opens it (`-f` would
stay on the old file). `grep -E 'ERROR|FATAL'`: searches for either of the two words; the quotes stop the shell
from taking `|` for a pipe. (`--line-buffered` makes lines flow without delay if the output goes to another pipe
or a file.) · *1.5.1, Answer 1.6*

---

## Scoring

| Correct answers | What to do |
|---|---|
| 16–18 | Move on to Phase 2. Your model of the shell and the file system is solid. |
| 12–15 | Move on to Phase 2, but re-read the sections your mistakes point to and run the commands in those sections on your machine again. |
| 8–11 | Review 1.1.2 and 1.4.2 — these two sections will be used in every remaining phase (especially Phase 5 and Phase 10). |
| 0–7 | Work through the phase from the start, this time running **every** 🔧 box on your machine. The skills in this phase settle in by typing, not by reading. |

**Which section to go back to for each missed question:**

| Question | Section |
|---|---|
| 1 | 1.1.1 — Terminal, shell, REPL |
| 2, 7, 8 | 1.1.2 — Command lookup, fork + exec |
| 3, 4 | 1.2.1 — FHS |
| 17 | 1.2.2 + 1.5.3 — `/var/log` and the journal |
| 11, 14 | 1.3.1 — `mv`, deleted but open files |
| 15 | 1.3.2 — `find` |
| 5 | 1.4.1 — The three standard streams |
| 9, 10, 12 | 1.4.2 — Redirection and pipes |
| 6, 16, 18 | 1.5.1 — Text tools |
| 13 | 1.4.3 + 1.5.2 — Pipelines and `awk` |

---

# Phase 1 — Closing and bridge to Phase 2

## What you carry from this phase

When you finish this phase you have **four tools** in hand:

**1. The "the shell interprets first, then the program runs" model.**
Expansion, redirection and command lookup happen **before the program**. When a command behaves unexpectedly,
your first question will not be "what did the program do?" but "**what did the shell give** the program?" The
traps of the unquoted glob, the `2>&1` order and redirection with `sudo` all come from this single idea.

**2. The "the environment doesn't carry over" reflex.**
The PATH, variables and aliases in your terminal don't exist in cron, systemd, a container or CI. You will use
this reflex when writing service definitions in Phase 5 and scripts in Phase 10.

**3. The FHS map.**
You can answer "where is the config, where is the log, where is the state data, why is the disk full" with a
directory name. In Phase 6 you will see the disks and mounts under this tree, and in Phase 8 how packages spread
their files across it.

**4. The habit of building pipelines step by step.**
Splitting a question into small tools and checking each step's output with your own eyes. All the diagnostic
flows in Phase 11 run on top of this skill.

## Where Phase 2 connects to this

In Phase 2 we get into users, groups and permissions. You will see these connections:

| What you learned in Phase 1 | What happens in Phase 2 |
|---|---|
| `$` and `#` in the prompt (1.1.1) | What root is, why UID 0 is special (2.1) |
| `drwx------` and `drwxrwxrwt` in the `ls -l /` output (1.2.1) | Reading permission bits; the sticky bit (2.2, 2.3) |
| `cut -d: -f1,7 /etc/passwd`, `awk -F: '$3 >= 1000'` (1.5) | `/etc/passwd` and `/etc/shadow` field by field (2.1) |
| The `sudo echo > file` trap (1.4.2) | What `sudo` really does, sudoers (2.4) |
| `/proc/<PID>/environ` is open only to its owner (1.2.2) | Which user processes run as, how permission checks are done (2.2) |
| `grep` exit code 2 = permission error (1.5.1, 1.5.4) | The "Permission denied" decision tree (2.5) |
| `auth.log` and `journalctl -u sshd` (1.2.2, 1.5.3) | How the SSH key gets onto EC2, default users (2.5) |

> **Phase 1 output — ask yourself before moving on:**
> "I logged in to a server over SSH and the application returns 502. Which directories do I look at, and which
> pipeline do I write?" Can I explain this, with the commands?
>
> An example chain of answers: **see that the disk isn't full with `df -h /var` → find the most recently written
> log with `ls -ltr /var/log/nginx/` → count how widespread it is with `grep -c '" 502 ' access.log` → find
> which paths it is on with `awk '$9 == 502 {print $7}' access.log | sort | uniq -c | sort -rn` → read the
> backend error with `grep -i upstream error.log | tail` → move on to the backend service's own log with
> `journalctl -u <application> --since "30 min ago"`.**
>
> If you can build this chain, you are ready for Phase 2.

---

> **Navigation:** [◀ Phase 0 — Mental Model](Phase_0_Mental_Model.md) · **Phase 1** · [Phase 2 — Users and Permissions ▶](Phase_2_Users_and_Permissions.md)
