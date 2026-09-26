# Phase 10 — Automation and Scripting: Stop Doing It by Hand

> **Navigation:** [◀ Checkpoint Quiz 4](Checkpoint_Quiz_4.md) · **Phase 10** · [Phase 11 — Observability and Troubleshooting ▶](Phase_11_Observability_and_Troubleshooting.md)

---

## Where we are coming from

From Phase 0 to 9 you always did it **by hand**: typed commands in the shell (Phase 1), fixed permissions
(Phase 2), started processes (Phase 3), mounted disks (Phase 6), wrote services (Phase 8), hardened the system
(Phase 9). Each one on a single machine, one time, at your keyboard. Phase 10 breaks this habit. Because a
cloud engineer's real job is not a single machine — it is dozens, hundreds, machines that appear and vanish
automatically. Nothing done by hand can be repeated, audited, or trusted at that scale.

The lesson of three phases turns into a mindset here:

- **Phase 5 — cloud-init.** You saw the configuration that runs on first boot. cloud-init is really a
  **script**; Phase 10 teaches you to write that script correctly.
- **Phase 8 — immutable / baked AMI.** You saw the "don't patch the machine, rebuild it" idea. Phase 10 gives
  you how that "rebuild" is done **with code**.
- **Phase 9 — hardening steps.** Every hardening step you did by hand should really have been a repeatable line
  of code. Phase 10 systematizes this.

## The question of this phase

Up to Phase 9 we said "how do I set up, run and defend a machine." Phase 10 asks exactly the opposite:

> *"How do I turn everything I do by hand — installation, configuration, hardening — into repeatable, reliable
> code that does not break when run twice; and where is this on the road from 'patching servers' to 'producing
> infrastructure as code'?"*

Two ideas sit at the center of this phase: **robustness** (a script must not silently do the wrong thing — it
must stop early on error) and **idempotency** (running the same script twice must not break the system, it must
reach the same result). These two ideas are the bridge that carries you from Bash to IaC (Infrastructure as
Code). A cloud engineer does not patch servers by hand — they **produce** them.

By the end of this phase you will be able to turn a repetitive task into a reliable Bash script; explain why a
script can silently cause disaster (unquoted variable, unchecked exit code); and build the idempotency bridge
from cloud-init/user-data to Ansible/Terraform.

---

## By the end of this phase

- You will be able to write the **Bash scripting basics** (variables, conditionals, loops, functions, exit code
  `$?`) and explain at a mechanism level why quoting (`"$var"`) saves lives
- You will be able to write a **robust script**: stop early on error with `set -euo pipefail`, clean up with
  `trap`, logging
- You will recognize and prevent the unquoted-variable + spaced/empty-path disaster (`rm -rf $DIR/` where
  `$DIR` is empty)
- You will be able to decide **where Bash ends and Python begins** (logic beyond ~20 lines, JSON/HTTP work →
  Python)
- You will be able to explain **idempotency** and write a script idempotently
- **Cloud:** you will be able to explain the relationship of the user-data script / cloud-init to this phase;
  the road from there to Ansible/Terraform; and the "immutable rebuild" instead of "mutable server patching"
  philosophy

---

## The phase map

| Section | Topic | Depth | Why it is here |
|---|---|---|---|
| 10.1 | Bash scripting basics | `[application]` | The alphabet of automation: variables, conditionals, loops, exit code |
| 10.2 | Writing robust scripts | `[application]` | `set -euo pipefail`, `trap`, quoting — prevent silent disaster |
| 10.3 | Where Bash ends, Python begins | `[concept]` | Choosing the right tool: the ~20-line / JSON-HTTP threshold |
| 10.4 | Idempotency and the IaC bridge | `[concept]` | Same script twice → must not break; cloud-init → Ansible/Terraform |
| 10.5 | When this phase breaks | — | Automation/script failure signatures |

> **How to work through this phase:** The commands in this phase are mostly 🟢/🟡 — writing and running a
> script. But a script is a **blind force**: an error you write does damage **much faster and much wider** than
> doing it by hand (an `rm -rf` loop deletes a whole directory in seconds). So this phase's most critical habit
> is: for every destructive command (🔴), first "dry-run" it with `echo` (print what it would delete without
> actually deleting), then run the real thing. Test your scripts on your own test machine, in unimportant
> directories. The most instructive experiment: write a script without `set -euo pipefail` and deliberately
> make one of its commands fail — see the script **ignore the error and continue**; then add `set -e` and do
> the same, and feel the difference.

---
---

# 10.1 Bash Scripting Basics

## 10.1.1 From a sequence of commands to a program: variables, conditionals, loops `[application]`

In Phase 1 you typed commands into the shell one by one. A **script** is putting those commands in a file and
running it like a program. The basic building blocks:

```bash
#!/usr/bin/env bash          # shebang: which interpreter should run this
name="web-01"                # variable (NO SPACE around =)
count=3

if [ "$count" -gt 0 ]; then  # conditional
  echo "$name has $count items"
fi

for i in 1 2 3; do           # loop
  echo "iteration $i"
done

backup() {                   # function
  local src="$1"             # first argument; local = specific to the function
  echo "backing up $src"
}
backup /etc
```

Two beginner traps: (1) there can be no space around `=` in assignment (`name = "x"` is an error); (2) you
call a variable with `$` when **using** it (`$name`), not when assigning it.

## 10.1.2 Exit code: every command's silent signal `[application]`

In Phase 3 we hinted that every process returns an exit code. This is the **heart** of script automation: as
soon as any command finishes, it returns `0` (success) or non-zero (error), and you read it with `$?`.

```bash
$ ls /var/log > /dev/null
$ echo $?
0                            # success
$ ls /no-such-place 2>/dev/null
$ echo $?
2                            # error (non-zero)
```

Why is this critical? Because if a script **ignores** success/failure, it silently does the wrong thing: it
says "I took a backup" but the command errored; it says "I stopped the service, now I'm deleting the file" but
the stop failed. The exit code is the script's only answer to "did it actually happen?" — and in 10.2 we will
turn this into an automatic safety net (`set -e`).

> **🔧 See it on your machine** 🟢 — watch the exit-code chain
>
> ```
> $ true;  echo $?          # 0
> $ false; echo $?          # 1
> $ grep -q root /etc/passwd && echo "found" || echo "missing"   # branch on exit code
> ```
>
> `&&` (run if the previous ones succeeded) and `||` (run if the previous ones failed) look directly at the
> exit code. Your scripts' flow is built on these two operators and `if` — all of them read `$?`.

## 10.1.3 Why quoting saves lives `[mechanism]`

This is Bash's most painful mechanism. When you use a variable unquoted (`$var`), Bash first **splits it into
words** (word splitting) and **expands globs** (turns characters like `*` into file names). Inside quotes
(`"$var"`) the value stays as-is, a single piece.

```bash
file="my report.txt"        # has a space in it
rm $file                    # ❌ Bash sees this as "rm my report.txt" → deletes TWO files: 'my' and 'report.txt'
rm "$file"                  # ✅ one file: 'my report.txt'
```

The rule is simple and absolute: **put every variable expansion inside double quotes** (`"$var"`, `"$@"`,
`"${arr[@]}"`) — except in the exceptional cases where you deliberately want splitting. This one habit prevents
the bulk of script disasters before they are born (in 10.2.3 we will see the `rm -rf` version of this).

> **⚠️ Common misconception: "My variable has no spaces, no need to quote."**
>
> None today — but your script will run tomorrow with another input, with another user's file name, or with an
> empty variable. Quoting is written not for "the current value" but for "every possible value." An empty
> variable (`$var` where var="") if unquoted becomes **no argument** and changes the meaning of the command;
> if quoted it becomes an **empty argument**. The two are very different, and the difference is exactly where
> disasters come from. The reflex: quote without exception.

> **🤔 Think 10.1** — A script has this line: `cp $SRC $DST`. `SRC="/tmp/a b.txt"` (with a space) and
> `DST="/backup/"`. (a) When this line runs unquoted, how many arguments does `cp` see and what happens? (b)
> When `SRC` is an empty variable (`SRC=""`), what does unquoted `cp $SRC $DST` do — what is the difference from
> `cp "$SRC" "$DST"`? (c) Write the rule in one sentence.
>
> *(Answer: at the end of the phase)*

---
---

# 10.2 Writing Robust Scripts

## 10.2.1 `set -euo pipefail`: stop early on error `[application]`

By default Bash is **forgiving** — even if a command errors, the script continues to the next line. This is a
disaster in automation: in a "stop the service → delete the file" script, the delete runs even if the stop
failed. The solution is the first line of every robust script:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

Three separate protections:

- **`-e`** (errexit): when a command returns a non-zero code, the script **stops immediately**. No silent
  continuation.
- **`-u`** (nounset): error when an undefined variable is used. If `$DIR` in `rm -rf "$DIR/"` is undefined due
  to a typo, `-u` stops the script — it does not run `rm -rf /` on the root.
- **`-o pipefail`**: if any stage of a pipe (`a | b`) errors, the whole pipe counts as an error. By default
  only the **last** command's code is seen; this leads to thinking `curl ... | tar ...` succeeded even though
  `curl` crashed.

This one line is the script turning from "forgiving" into "meticulous" — instead of silently doing the wrong
thing, it stops at the first error and tells you.

## 10.2.2 Cleanup and logging with `trap` `[concept]`

If a script errors midway (or the user hits Ctrl-C), it can leave half-done work behind: a temp directory, a
lock file, a mounted disk. `trap` is the way to say "whatever reason the script ends for, do this cleanup":

```bash
tmpdir=$(mktemp -d)
trap 'rm -rf "$tmpdir"' EXIT   # delete the temp dir however the script ends
```

The `EXIT` signal is triggered both when the script ends normally and when it stops on error (`set -e`) — so
cleanup is guaranteed. Logging is also part of robustness: writing what the script does with lines like
`echo "[$(date)] step X"` or with `logger` to journald (Phase 5!) answers the "what happened" question later.

## 10.2.3 Unquoted variable + spaced path = disaster `[application]`

Here is this phase's most expensive failure, lived countless times in the real world. A cleanup script:

```bash
DIR="/var/tmp/cache"
rm -rf $DIR/                  # ❌ unquoted
```

Three separate disaster scenarios:

1. **`DIR` is empty or undefined** (typo, unset variable): it becomes `rm -rf /` — **deletes the whole system.**
   `set -u` catches the *undefined* case only (a typo in the name); a variable set to `""` slips through, so
   add `[ -n "$DIR" ]` (or `${DIR:?}`). Quoting + `set -u` + that check together save lives.
2. **`DIR` contains a space** (`/var/tmp/my cache`): `rm -rf /var/tmp/my cache/` → Bash sees two arguments:
   `/var/tmp/my` and `cache/` — deletes the wrong directories.
3. **`DIR` contains a glob** or expands: unexpected files match.

The correct form closes all three:

```bash
set -euo pipefail
DIR="/var/tmp/cache"
[ -n "$DIR" ] || { echo "DIR is empty, aborting"; exit 1; }   # extra safety: stop if empty
rm -rf "$DIR"/                # ✅ quoted
```

> **🔧 See it on your machine** 🟡 — dry-run before a destructive command
>
> ```
> $ DIR="/var/tmp/test space"
> $ printf '[%s]\n' rm -rf $DIR/     # ❌ unquoted: one line per argument
> [rm]
> [-rf]
> [/var/tmp/test]
> [space/]
> $ printf '[%s]\n' rm -rf "$DIR"/   # ✅ quoted: the path stays one argument
> [rm]
> [-rf]
> [/var/tmp/test space/]
> ```
>
> Putting `printf '[%s]\n'` in front of the command's arguments (plain `echo` would print both versions
> identically) shows what would **actually** be passed without deleting anything. This "dry-run" habit is
> the single most valuable safety reflex of this phase. See with your own eyes how the unquoted version
> splits into two arguments.

> **🤔 Think 10.2** — A colleague's script, **without** `set -e`, is: `cd "$WORKDIR"` then `rm -rf ./*`.
> `WORKDIR` points to a directory that no longer exists (deleted). (a) When `cd` fails and there is no `set
> -e`, what does the script do, and in **which directory** does `rm -rf ./*` run? (b) Why is this a disaster?
> (c) How would `set -euo pipefail` and `cd "$WORKDIR" || exit 1` prevent it?
>
> *(Answer: at the end of the phase)*

---
---

# 10.3 Where Bash Ends, Python Begins

## 10.3.1 Choosing the right tool `[concept]`

Bash is a great **glue**: perfect for chaining commands, moving files, starting services. But past a threshold
Bash becomes a **burden**. General rules:

- **Logic beyond ~20 lines.** When complex conditionals, nested loops, data structures are needed, Bash becomes
  unreadable and error-prone. Python's clear syntax and real data structures (lists, dicts) come into play.
- **JSON / structured data.** Bash cannot parse JSON correctly ("parsing" with `grep`/`sed` is fragile); `jq`
  helps but for complex transformations Python's `json` module is incomparable.
- **HTTP / API work.** A few `curl` calls are fine in Bash; but when you need API pagination, retries, error
  handling, Python's `requests` is the right tool.
- **Testability.** If serious logic wants tests, Python's testing ecosystem is far ahead of Bash.

The reflex: "Is this a file/command gluing job, or a real program?" If the former, Bash; if the latter, Python.
Forcing the wrong tool (500 lines of Bash, or Python for a single `mv`) is a mistake in both directions.

> **💡 Cloud connection — which language where:** An EC2 user-data (Phase 5) usually has a short **Bash**
> script: install a package, start a service, download a config. But the application's real logic (a Lambda, a
> data processing step, a deploy tool) is almost always **Python**. The two live together: Bash brings the
> machine up, Python does the work. Drawing the right boundary is the foundation of automation that scales and
> is maintainable.


> **🤔 Think 10.3** — A script has grown to 400 lines: it calls a REST API with pagination, "parses" nested JSON with grep/sed pipelines and must retry on HTTP 429. Every few weeks a small change breaks it silently. (a) Which of the rules in 10.3.1 does it hit? (b) What should you do? (c) What legitimately stays in Bash?
>
> *(Answer: at the end of the phase)*
---
---

# 10.4 Idempotency and the IaC Bridge

## 10.4.1 Idempotency: the same script twice must not break `[concept]`

An **idempotent** operation brings the system to the **same** final state whether it runs once or ten times —
and the second run does not break anything. This is automation's most fundamental yet most often skipped
property. Example:

```bash
# ❌ NOT idempotent: the second run errors / adds the line twice
mkdir /opt/app                          # errors the second time if the dir exists
echo "export PATH=..." >> ~/.bashrc     # adds the line AGAIN on every run

# ✅ idempotent: the same result no matter how many times it runs
mkdir -p /opt/app                       # no problem if it exists
grep -qxF "export PATH=..." ~/.bashrc || echo "export PATH=..." >> ~/.bashrc  # add if missing
```

Why a must? Because real automation **runs again**: cloud-init sometimes re-runs, a deploy is triggered twice,
a configuration tool applies the "desired state" every minute. If it is not idempotent, each repeat breaks the
system a little more — doubled lines, erroring steps, inconsistent state. Idempotency means "describe the
desired state, reach it safely again and again."

![Figure 10.1 — Idempotency: what happens when the same script runs twice. On the left a non-idempotent script (mkdir /opt/app; echo line >> file): the first run creates the directory and adds the line; the second run gives a "directory exists" error and adds the same line again, corrupting the file — every repeat makes it worse. On the right an idempotent script (mkdir -p; grep -qxF || echo): the first run reaches the desired state; the second run changes nothing, the same final state is preserved. The lesson at the bottom: idempotency means "describe the desired state, reach it safely again and again" — the foundation of IaC.](../diagrams/png/lx-10-01-idempotency.png)

## 10.4.2 The bridge from Bash to IaC `[application]`

The idea of idempotency carries you directly to **IaC** (Infrastructure as Code). The journey goes like this:

1. **By hand** (Phase 0-9) — a single machine, not repeatable.
2. **A Bash script** — repeatable but fragile, you ensure idempotency.
3. **user-data / cloud-init** (Phase 5) — you run that script automatically at boot; the machine sets itself
   up.
4. **Ansible** — a tool that guarantees idempotency **itself**; you describe the "desired state" (package
   installed, service running), Ansible brings the system there and does not break it on re-run.
5. **Terraform** — you define the machine **itself** (and all cloud resources) as code; infrastructure is now
   a text file, under version control, reviewed.

The philosophy under this bridge is Phase 8's immutable idea: **"immutable rebuild" instead of "mutable server
patching".** When something breaks on a server, you do not repair it by hand (mutable patching — makes the
state unknown); you **destroy it and rebuild it from code** (immutable rebuild — the state always equals the
code). So this phase teaches not writing a single script, but **thinking of all infrastructure as repeatable
code**.

> **💡 Cloud connection — from user-data to Terraform:** An EC2 instance's user-data (Phase 5) is often your
> first "IaC" step: a Bash script that runs at boot. But as scale grows this is not enough — you want to keep
> 50 machines consistent, review changes, roll back. That is when **Ansible** (configures the inside of the
> machine) and **Terraform** (produces the machine itself and its network/SG/IAM) come into play. But under
> all of them lie this phase's two lessons: robustness (do not break silently) and idempotency (the same result
> on re-run). Understanding these two ideas before learning any tool pays off whichever tool you use.

> **🤔 Think 10.4** — A cloud-init user-data script of yours installs a package and adds a configuration line
> to `/etc/app.conf`. One day the instance reboots and part of cloud-init re-runs. (a) If the script is not
> idempotent (adds the line with `>>`), what happens to the config file? (b) Which single pattern would you use
> to make it idempotent? (c) Why is this a small example of the "immutable rebuild" philosophy — how do
> idempotency and immutability serve the same goal (state = code)?
>
> *(Answer: at the end of the phase)*

---
---

# 10.5 When This Phase Breaks — Automation/Script Failure Signatures

This phase's failures are insidious: a script "looks like it's working" but silently does the wrong thing. The
table connects the symptom to the cause:

| Symptom | Likely cause | Where to look | Related section |
|---|---|---|---|
| Script continued after an erroring command, wrong result | No `set -e` | Add `set -euo pipefail` to the first line | 10.2.1 |
| `rm`/`cp` processed the wrong files (spaced path) | Unquoted variable | All `$var` → `"$var"`; dry-run with `echo` | 10.1.3, 10.2.3 |
| `rm -rf /`-like disaster | Empty/undefined variable unquoted | `set -u` + `[ -n "$DIR" ]` check | 10.2.3 |
| `curl ... \| tar ...` thought it succeeded but curl crashed | No `pipefail` | `set -o pipefail` | 10.2.1 |
| Script errored / broke on the second run | Not idempotent | `mkdir -p`, `grep -qxF \|\|`, conditional adding | 10.4.1 |
| Config line added again on every boot | cloud-init not idempotent | Presence check before adding | 10.4.1 |
| Leftover temp dir/lock file piled up | No cleanup with `trap` | `trap '...' EXIT` | 10.2.2 |
| 400-line Bash impossible to maintain | Wrong tool | JSON/HTTP/complex logic → Python | 10.3.1 |

> **The lesson from this table:** The two big ideas of this phase lie beneath every row. The first is **silent
> failure is the enemy**: a script's most dangerous state is not crashing but erroring and **continuing** —
> `set -euo pipefail` and quoting prevent exactly this, turning the script from "forgiving" into "meticulous."
> The second is **repeatability is reliability**: a non-idempotent script may work once but is untrustworthy on
> the second run — and real automation always runs again. And remember: a script is a blind force; a mistake
> you make by hand hits one file, the same mistake in a script hits a fleet — so dry-run destructive commands
> with `echo` first.

---
---

# Phase 10 — Answers to the think questions

## Answer 10.1 — An unquoted variable is split into words; the rule: always quote

(a) When `cp $SRC $DST` is unquoted and `SRC="/tmp/a b.txt"` has a space, Bash does word splitting: `cp` sees
**three** arguments — `/tmp/a`, `b.txt`, `/backup/`. `cp` thinks this is "copy two sources into a destination,"
looks for files `/tmp/a` and `b.txt` (neither exists), errors; or copies unexpected files. `cp "$SRC" "$DST"`
sees two arguments and copies the right file. (b) When `SRC=""` (empty), unquoted `cp $SRC $DST` produces **no
argument** for `SRC` — the command becomes `cp /backup/`, a single-argument `cp` errors (or worse, misbehaves).
`cp "$SRC" "$DST"` passes an empty argument (`cp "" "/backup/"`) — still an error but predictable behavior (and if
the variable was *unset* rather than `""`, `set -u` would catch it). (c) The rule: **put every variable expansion inside double quotes without exception**
(`"$var"`) — because quoting keeps the value as a single piece "under every possible input"; leaving splitting
unquoted is done only when you deliberately want it.
**Related section:** 10.1.3 · **Continues in:** 10.2.3 (`rm -rf` disaster).

## Answer 10.2 — `cd` fails, `rm -rf ./*` runs in the wrong directory

(a) Without `set -e`, `cd "$WORKDIR"` fails (the directory does not exist) but the script **does not stop, it
continues** — and because the current working directory did not change, `rm -rf ./*` runs in **the directory
the script was started in** (e.g. the user's home directory or a project root). (b) This is a disaster because,
instead of the intended directory, a completely unrelated, perhaps critical directory is deleted — `cd`'s
silent failure directs the destructive command to the wrong place. (c) With `set -euo pipefail`, when `cd`
fails the script **stops immediately** and never reaches `rm`; also writing `cd "$WORKDIR" || exit 1`
explicitly gives the same guarantee. Two layers (fail-fast + explicit check) cut off the silent failure before
the destructive command.
**Related section:** 10.2.1, 10.2.3 · **Continues in:** the 10.5 failure table (rows 1, 3).

## Answer 10.3 — It hits all four rules: move the logic to Python, keep Bash as glue

(a) All four: logic beyond ~20 lines, structured JSON data parsed with fragile `grep`/`sed`, HTTP/API work (pagination, retries) and testability — nobody can write tests for it. (b) Rewrite the core in Python (`requests` for pagination and retries, the `json` module for parsing) and add tests; the silent breakage is the price of forcing the wrong tool. (c) Bash keeps the glue: the user-data one-liner that installs Python and runs `python3 tool.py`, moving files, starting services. Bash brings the machine up, Python does the work.
**Related section:** 10.3.1 · **Continues in:** 12.2.1 (cloud-init user-data)

## Answer 10.4 — The config file bloats; conditional adding; idempotency = state equals code

(a) If the script is not idempotent and adds the line with `>>`, then every time cloud-init re-runs it adds
**the same line again** — the config file bloats over time with copies of the same line, and even
conflicting/doubled settings can break the application. (b) Idempotent pattern: check whether the line exists
before adding it — `grep -qxF "line" /etc/app.conf || echo "line" >> /etc/app.conf` (add if missing). (c) This
is a small example of "immutable rebuild" because both serve the same goal: **the system's state must always
equal the code.** Idempotency ensures that re-running the same code does not change the state (state = the
final form the code describes); immutability fixes the state to the code by rebuilding the machine from code
instead of patching it. Together they prevent "state drift" — whether you re-run the same script or rebuild the
machine, the result is always the same.
**Related section:** 10.4.1-10.4.2 · **Continues in:** the 10.5 failure table (row 6).

---
---

# Phase 10 — Frequently asked questions

**Q1 — What does `set -euo pipefail` do in one sentence?** It turns the script from "forgiving" into
"meticulous": `-e` stops on error, `-u` stops on an undefined variable, `-o pipefail` counts any stage of a
pipe crashing as an error. It prevents silent wrong work (10.2.1).

**Q2 — Why should I quote every variable?** An unquoted variable undergoes word splitting + glob expansion; with
spaced, empty, or glob-containing values the command sees the wrong arguments. `"$var"` keeps the value as a
single piece under every input (10.1.3).

**Q3 — What is `$?`?** The exit code of the last command run: `0` success, non-zero error. The script's answer
to "did it actually happen"; `&&`, `||`, `if` all look at it (10.1.2).

**Q4 — When should I move from Bash to Python?** When there is logic beyond ~20 lines, JSON/structured data,
serious HTTP/API work, or a testing need. Bash is glue, Python is a program (10.3.1).

**Q5 — Why does idempotency matter?** Real automation runs again (cloud-init, deploy, config tool). If it is
not idempotent, every repeat breaks the state (doubled lines, errors). Idempotent = the same final state no
matter how many times it runs (10.4.1).

**Q6 — What is `trap` for?** It runs a cleanup however the script ends (normal, error, Ctrl-C) — e.g. deleting a
temp dir: `trap 'rm -rf "$tmpdir"' EXIT`. Leaves no half-done work (10.2.2).

**Q7 — The difference between user-data, Ansible and Terraform?** user-data runs a script at boot (inside the
machine, once); Ansible applies the desired state idempotently (inside the machine, again and again); Terraform
produces the machine and cloud resources **themselves** as code (all of the infrastructure) (10.4.2).

---
---

# Phase 10 — Test yourself

Write your answers on paper, then compare with the answer key. Target: 14+ out of 18.

## Section A — Definition and mechanism

1. What is the exit code (`$?`) and why is it the heart of script automation? What does `0` mean?
2. What does quoting (`"$var"`) do compared to unquoted? Explain word splitting with an example.
3. What do `set -e`, `set -u`, `set -o pipefail` do separately?
4. What is idempotency? Give an example of a non-idempotent command and its idempotent counterpart.
5. When and why does `trap '...' EXIT` run?
6. Where does Bash end and Python begin — write three threshold criteria.
7. Explain the difference between "mutable server patching" and "immutable rebuild."
8. How do the `&&` and `||` operators use the exit code?

## Section B — Apply and diagnose

9. A script continues after an erroring command and produces a wrong result. What do you add to the first line?
10. Write the two separate disaster scenarios in the line `rm -rf $DIR/` (empty `DIR`, spaced `DIR`) and the
    fix.
11. A cloud-init script adds the same line to the config file on every boot. The cause and the idempotent fix?
12. A `curl ... | tar ...` pipeline thinks the script succeeded even though `curl` crashed. Which setting is
    missing?
13. How do you safely see what a destructive `rm` command would run before running it?
14. You inherited a 500-line Bash script that parses JSON and calls an API. What do you recommend and why?

## Section C — Reasoning and connection

15. Which two lessons of this phase (robustness, idempotency) does Phase 5's cloud-init directly require? Why?
16. How does Phase 8's immutable/baked AMI philosophy serve the same goal (state = code) as this phase's
    idempotency idea?
17. Why should the hardening steps you did by hand in Phase 9 really have been code? How does IaC solve this?
18. Combine the trio `set -euo pipefail` + quoting + idempotency in one sentence: what is the essence of
    reliable automation?

---

## Answer key

1. The success/failure signal of the last command; the script answers "did it actually happen" based on it;
   `0` = success, non-zero = error (10.1.2). — 2. An unquoted variable is split into words + globs expand; `rm
   $f` where `f="a b"` → two files; `"$var"` keeps the value as a single piece (10.1.3). — 3. `-e` stops on
   error, `-u` stops on an undefined variable, `-o pipefail` counts the whole pipe as an error when any stage
   crashes (10.2.1). — 4. The same operation gives the same final state no matter how many times it runs;
   `mkdir /x` (errors the second time) vs `mkdir -p /x` (idempotent) (10.4.1). — 5. It is triggered however the
   script ends (normal/error/signal); for guaranteed cleanup (temp dir/lock) (10.2.2). — 6. Logic beyond ~20
   lines; JSON/structured data; serious HTTP/API or a testing need → Python (10.3.1). — 7. Mutable: repair the
   broken server by hand (state becomes unknown); immutable: destroy and rebuild from code (state = code)
   (10.4.2). — 8. `&&` runs if the previous succeeded (exit 0), `||` runs if the previous failed — both look at
   `$?` (10.1.2).

9. `set -euo pipefail` (10.2.1). — 10. Empty `DIR` → `rm -rf /` (whole system); spaced `DIR` → wrong
   directories; fix: `set -u` + `[ -n "$DIR" ]` + `rm -rf "$DIR"/` (10.2.3). — 11. Not idempotent (`>>` adds
   unconditionally); fix: `grep -qxF "line" file || echo "line" >> file` (10.4.1). — 12. `set -o pipefail` is
   missing; without it only the last command's (tar) code is seen (10.2.1). — 13. Put `echo` in front of the
   command (dry-run) — shows what would actually run without deleting anything (10.2.3). — 14. Recommend moving
   to Python: JSON/HTTP/complex logic + testability exceed Bash's threshold; Bash is glue, this is a program
   (10.3.1).

15. Robustness (if it errors silently at boot the machine is set up wrong and no one sees it) and idempotency
    (cloud-init can re-run; if not idempotent every repeat breaks it) — both are critical because it runs
    automatically, unattended (10.4.2, Phase 5). — 16. Both serve the "state = code" goal: idempotency ensures
    re-running the same code does not change the state; immutability fixes the state to the code by rebuilding
    the machine from code; together they prevent drift (10.4.2, Phase 8.4). — 17. Hardening done by hand is not
    repeatable, not auditable, inconsistent across 50 machines; written as code (Ansible/Terraform) it becomes
    repeatable, reviewable, version-controlled and idempotent (10.4.2, Phase 9). — 18. The essence of reliable
    automation: do not fail silently (`set -euo pipefail` + quoting) and do not break on re-run (idempotency) —
    that is, predictable, repeatable behavior under every condition (10.2, 10.4).

## Scoring

| Correct | What it means |
|---|---|
| 16-18 | You can write robust and idempotent automation. You are ready for Phase 11 (troubleshooting). |
| 13-15 | Good. Re-read the sections of the questions you missed (especially 10.2 and 10.4). |
| 9-12 | The basics are there but fragile. Study `set -euo pipefail`, quoting and idempotency. |
| 0-8 | Walk through the phase again; write small scripts on your own test machine, dry-running with `echo`. |

Missed question → section to return to:

| Question | Section |
|---|---|
| 1, 8 | 10.1.2 Exit code |
| 2, 10 | 10.1.3 / 10.2.3 Quoting |
| 3, 9, 12 | 10.2.1 `set -euo pipefail` |
| 5 | 10.2.2 `trap` |
| 4, 11, 16 | 10.4.1 Idempotency |
| 6, 14 | 10.3.1 Bash vs Python |
| 7, 15, 17 | 10.4.2 IaC bridge |
| 13 | 10.2.3 dry-run |
| 18 | 10.2 + 10.4 combined |

---
---

# Phase 10 — Closing and Bridge to Phase 11

## What you carry from this phase

Phase 10 turned you from "someone who does it by hand" into "someone who produces." You gained two lasting
ideas: **robustness** (a script must not silently do the wrong thing — `set -euo pipefail`, quoting, `trap`,
exit-code checking) and **idempotency** (re-running the same code gives the same final state). You saw that
these two ideas are the bridge from a single script to IaC that manages all infrastructure as code (cloud-init
→ Ansible → Terraform) — and that under it lies Phase 8's immutable philosophy: "don't patch the server,
rebuild it." Now you can turn every repetitive task you do by hand into repeatable and reliable code.

## Where Phase 11 connects

Phase 10 gave you "automating the work"; Phase 11 gives you "finding the evidence when the work breaks." The two
are two faces of a coin: automation **builds** systems, troubleshooting **repairs** them. And it is precisely
because of automation that troubleshooting becomes even more critical — when a script silently does the wrong
thing (10.2), you catch it only with a systematic forensic reflex (log → status → resource → network). Phase 11
gathers all the previous phases' "when this phase breaks" notes into a single methodology: when something
breaks, not panic but layer-by-layer evidence.

> **🤔 Phase output — ask yourself:** How does the "silent failure" you saw in this phase (a script errored and
> continued) connect to Phase 11's "where is the evidence" question? A cloud-init script of yours silently
> stopped halfway on a machine and the machine "looks healthy" but the application does not work. Before
> entering Phase 11, think: in what order (which log, which status command) would you look for the evidence of
> this silent failure — and why is this exactly the "instance is healthy but the application is not"
> distinction?
>
> **🧪 Lab 10 idea (on your own test instance):** Write a short backup/cleanup script: (1) start with `set -euo
> pipefail`. (2) Find logs older than 7 days in a directory (`find /var/log/test -type f -mtime +7`) — first
> list them with just `-print` (dry-run!), then add `-delete`. (3) Before every destructive command, see what
> would be deleted with `echo`. (4) Check the exit code (`echo $?`). (5) Run the script **twice** — does the
> second run error, or is it idempotent? If not idempotent, fix it. This lab gathers this phase's robustness +
> idempotency reflex in your hands.

---

> **Navigation:** [◀ Checkpoint Quiz 4](Checkpoint_Quiz_4.md) · **Phase 10** · [Phase 11 — Observability and Troubleshooting ▶](Phase_11_Observability_and_Troubleshooting.md)
