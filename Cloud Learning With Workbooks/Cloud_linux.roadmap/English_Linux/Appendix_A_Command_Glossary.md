# Appendix A — Command Glossary

> **Navigation:** [◀ Phase 12 — Bridge to the Cloud](Phase_12_Bridge_to_the_Cloud.md) · **Appendix A** · [Appendix B — File and Directory Map ▶](Appendix_B_File_Directory_Map.md)

---

> **This appendix exists to remind, not to teach.** This is the page you keep open after you finish the
> workbook. Next to each command is the phase it came from — when a command slips your mind, go back there.
> Risk marks: 🟢 read-only (safe), 🟡 temporary change, 🔴 permanent change (be careful).

**Contents**
- [A.1 Shell and filesystem (Phase 1)](#a1-shell-and-filesystem-phase-1)
- [A.2 Users and permissions (Phase 2)](#a2-users-and-permissions-phase-2)
- [A.3 Processes and resources (Phase 3-4)](#a3-processes-and-resources-phase-3-4)
- [A.4 Boot, systemd, services (Phase 5, 8)](#a4-boot-systemd-services-phase-5-8)
- [A.5 Storage and filesystems (Phase 6)](#a5-storage-and-filesystems-phase-6)
- [A.6 Networking (Phase 7)](#a6-networking-phase-7)
- [A.7 Packages (Phase 8)](#a7-packages-phase-8)
- [A.8 Security and hardening (Phase 9)](#a8-security-and-hardening-phase-9)
- [A.9 Scripting (Phase 10)](#a9-scripting-phase-10)
- [A.10 Observability and troubleshooting (Phase 11)](#a10-observability-and-troubleshooting-phase-11)
- [A.11 Undo steps for permanent changes](#a11-undo-steps-for-permanent-changes)

---

## A.1 Shell and filesystem (Phase 1)

| Command | What it does | Risk |
|---|---|---|
| `pwd` | Prints the current directory | 🟢 |
| `ls -la` | Lists including hidden files + permissions | 🟢 |
| `cd /path` | Changes directory | 🟢 |
| `which <cmd>` | Finds which file a command runs from (PATH search) | 🟢 |
| `type <cmd>` | Is the command a builtin, alias, or file | 🟢 |
| `cat` / `less` / `head` / `tail` | Shows file contents | 🟢 |
| `find /path -name "*.log"` | Searches for files | 🟢 |
| `grep -r "pattern" /path` | Searches contents (recursive) | 🟢 |
| `cmd1 \| cmd2` | Pipe: connects cmd1's stdout to cmd2's stdin | 🟢 |
| `cmd > file` / `>> file` | Write stdout to file / append | 🟡 |
| `cmd 2>&1` | Merge stderr into stdout | 🟢 |
| `ln -s target name` | Creates a symbolic link | 🟡 |

## A.2 Users and permissions (Phase 2)

| Command | What it does | Risk |
|---|---|---|
| `id` | Current user's UID/GID/groups | 🟢 |
| `whoami` | Effective username | 🟢 |
| `ls -l` | A file's rwx permissions, owner, group | 🟢 |
| `stat <file>` | All of a file's metadata (perms, owner, times) | 🟢 |
| `chmod 640 <file>` | Changes permission bits | 🔴 |
| `chown user:group <file>` | Changes owner/group | 🔴 |
| `sudo <cmd>` | Runs the command with elevated privilege | 🔴 |
| `umask` | Default permission mask for new files | 🟢 |
| `getfacl` / `setfacl` | Extended ACL permissions | 🟢/🔴 |

## A.3 Processes and resources (Phase 3-4)

| Command | What it does | Risk |
|---|---|---|
| `ps aux` | All processes (state, owner, CPU/RAM) | 🟢 |
| `top` / `htop` | Live process and resource monitoring | 🟢 |
| `kill <pid>` / `kill -9 <pid>` | Sends a signal to a process (TERM / KILL) | 🔴 |
| `pkill <name>` | Kills a process by name | 🔴 |
| `nice` / `renice` | Sets process priority | 🟡 |
| `free -h` | RAM/swap status | 🟢 |
| `uptime` | Load average (R + D processes) and uptime | 🟢 |
| `nproc` | Core count the machine sees (load ÷ `nproc`) | 🟢 |
| `vmstat 1` | System-wide (CPU/IO/memory) live | 🟢 |
| `iostat -x 1` | Disk IO statistics | 🟢 |
| `nohup cmd &` | Keeps running even if the terminal closes | 🟡 |
| `jobs` / `fg` / `bg` | Background job control | 🟢 |

## A.4 Boot, systemd, services (Phase 5, 8)

| Command | What it does | Risk |
|---|---|---|
| `systemctl status <service>` | Service status (active/failed) | 🟢 |
| `systemctl start/stop <service>` | Start/stop service (running state) | 🟡 |
| `systemctl enable/disable <service>` | Start/don't start at boot (persistent definition) | 🔴 |
| `systemctl enable --now <service>` | Both start and add to boot | 🔴 |
| `systemctl daemon-reload` | Read unit-file changes | 🟡 |
| `systemctl list-units --failed` | Failed units | 🟢 |
| `journalctl -u <service>` | A unit's logs | 🟢 |
| `journalctl -b` / `-b -1` | This boot / previous boot logs | 🟢 |
| `systemd-analyze` | Boot-time analysis | 🟢 |
| `systemctl is-enabled/is-active <service>` | Query persistent/running state | 🟢 |

## A.5 Storage and filesystems (Phase 6)

| Command | What it does | Risk |
|---|---|---|
| `lsblk` | Block-device tree (disk, partition, mount) | 🟢 |
| `df -h` | Filesystem usage ratios | 🟢 |
| `du -sh <dir>` | Directory size | 🟢 |
| `mount /dev/xxx /mnt` | Mounts a filesystem (running state) | 🟡 |
| `umount /mnt` | Unmounts | 🟡 |
| `mkfs.ext4 /dev/xxx` | Creates a filesystem (**erases data!**) | 🔴 |
| `blkid` | Devices' UUIDs | 🟢 |
| `findmnt` | Shows the mount tree | 🟢 |
| `lsof <file>` | Process holding a file open | 🟢 |

## A.6 Networking (Phase 7)

| Command | What it does | Risk |
|---|---|---|
| `ss -tulpn` | Listening ports + process (three lenses: listening) | 🟢 |
| `ip a` | Network interfaces and IP addresses | 🟢 |
| `ip r` | Routing table | 🟢 |
| `ping <host>` | Reachability test | 🟢 |
| `curl -v <url>` | HTTP request (verbose) | 🟢 |
| `dig <domain>` / `nslookup` | DNS resolution | 🟢 |
| `ufw status` / `ufw allow` | Host firewall status / rule | 🟢/🔴 |
| `traceroute <host>` | Packet path | 🟢 |
| `nc -zv host port` | Port reachability test | 🟢 |
| `ssh [-i key] user@host` | Opens a remote shell (`-v` shows the handshake) | 🟢 |
| `netplan apply` | Applies `/etc/netplan/*.yaml` to the running network | 🔴 |

## A.7 Packages (Phase 8)

| Command | What it does | Risk |
|---|---|---|
| `apt update` | Refreshes the package list | 🟡 |
| `apt upgrade` | Upgrades installed packages to newer versions | 🔴 |
| `apt install <pkg>` | Installs a package (+ dependencies) | 🔴 |
| `apt remove/purge <pkg>` | Removes a package | 🔴 |
| `dpkg -l` | Lists installed packages | 🟢 |
| `dpkg -L <pkg>` | Files a package installed | 🟢 |
| `apt list --installed` | Installed packages | 🟢 |
| `systemctl` (see A.4) | Lifecycle of the installed service | — |

## A.8 Security and hardening (Phase 9)

| Command | What it does | Risk |
|---|---|---|
| `ssh-keygen` | Generates an SSH key pair | 🟡 |
| `sudo` / `visudo` | Privilege elevation / editing sudoers | 🔴 |
| `aa-status` | AppArmor profile status (MAC) | 🟢 |
| `getenforce` / `sestatus` | SELinux status (MAC) | 🟢 |
| `dmesg \| grep -i denied` | MAC denial records | 🟢 |
| `getcap` / `setcap` | File capabilities | 🟢/🔴 |
| `auditctl` / `ausearch` | Audit records | 🟢/🔴 |
| `chmod 600 ~/.ssh/id_*` | Restrict private-key permissions | 🔴 |

## A.9 Scripting (Phase 10)

| Construct | What it does | Risk |
|---|---|---|
| `#!/usr/bin/env bash` | Shebang — specifies the interpreter | 🟢 |
| `set -euo pipefail` | Robust script: stop on error/unset var/pipe failure | 🟢 |
| `"$var"` | Quoted variable — prevents word splitting | 🟢 |
| `trap '...' EXIT` | Cleanup when the script ends | 🟢 |
| `echo <destructive-cmd>` | Dry-run: see what it will do **without** doing it | 🟢 |
| `mkdir -p` / `grep -qxF \|\|` | Idempotent patterns | 🟡 |
| `$?` | Exit code of the last command | 🟢 |
| `cmd1 && cmd2` / `\|\|` | Branch on exit code | varies |

## A.10 Observability and troubleshooting (Phase 11)

| Command | What it does | Risk |
|---|---|---|
| `journalctl -u <service> -e` | End of a service log (first evidence stop) | 🟢 |
| `journalctl -k` / `dmesg -T` | Kernel messages (OOM, disk, driver) | 🟢 |
| `systemctl status <service>` | Service status (2nd layer) | 🟢 |
| `top` / `free` / `iostat` | Resource (3rd layer) | 🟢 |
| `ss -tulpn` | Network (4th layer) | 🟢 |
| `strace -p <pid>` | A process's system calls (final arbiter) | 🟡 |
| `lsof -p <pid>` | A process's open files/sockets | 🟢 |
| `/proc/<pid>/status` | Process details | 🟢 |
| `grep -i "failed" /var/log/auth.log` | Failed logins | 🟢 |

> **Diagnostic reflex (Phase 11.2):** When something breaks, the order is always the same — **log → service →
> resource → network → kernel.** Don't try random commands; gather evidence at each layer, and move to the next
> only after you've ruled out the one in hand.

## A.11 Undo steps for permanent changes

Every 🔴 command above has a way back — or an explicit statement that there is none. **Record the old state
first**, then change it.

| Command | Record first | Undo |
|---|---|---|
| `chmod` (any mode, incl. `chmod 600 ~/.ssh/id_*`) | `stat -c '%a' <file>` | `chmod <old mode> <file>` |
| `chown user:group <file>` | `stat -c '%U:%G' <file>` | `chown <old user>:<old group> <file>` |
| `sudo <cmd>` | — (`sudo` itself changes nothing) | Undo whatever `<cmd>` did |
| `setfacl` | `getfacl -R <path> > acl.bak` | `setfacl -b <file>` (drop all ACLs) or `setfacl --restore=acl.bak` |
| `setcap` | `getcap <file>` | `setcap -r <file>` |
| `kill` / `kill -9` / `pkill` | `ps -o pid,cmd -p <pid>` | **No undo for the process's memory state.** Restart it: `systemctl start <service>`. Try `kill` (TERM) before `kill -9` |
| `systemctl enable` / `enable --now` | `systemctl is-enabled <service>` | `systemctl disable` / `disable --now` |
| `systemctl disable` | `systemctl is-enabled <service>` | `systemctl enable` |
| `mount /dev/xxx /mnt` | `findmnt` | `umount /mnt` |
| `umount /mnt` | `findmnt /mnt` | `mount` again; if "target is busy", find the holder with `lsof +f -- /mnt` |
| `mkfs.ext4 /dev/xxx` | `lsblk -f` — check the target twice | **No undo — the data is gone.** Only a snapshot/backup taken *before* saves you |
| `ufw allow` / `ufw enable` | `ufw status numbered` | `ufw delete <rule>` / `ufw disable`. Over SSH, allow port 22 **before** enabling |
| `netplan apply` | `ip a`, `ip r`, `cp /etc/netplan/<file>.yaml <file>.bak` | Copy the backup back, then `netplan apply`. A wrong config over SSH can cut your own connection — keep console access (EC2 serial console / SSM) ready |
| `apt install <pkg>` | `dpkg -l <pkg>` | `apt remove <pkg>` then `apt autoremove` |
| `apt upgrade` | `apt list --upgradable`, `dpkg -l > pkgs.before` | **No single undo.** Reinstall the old version: `apt install <pkg>=<old version>` (see versions with `apt-cache policy <pkg>`); version pinning prevents the surprise |
| `apt remove` / `purge` | `dpkg -L <pkg>`, copy `/etc/<pkg>` | `apt install <pkg>`. **`purge` deletes the config with no undo** — back it up first |
| `visudo` / edits to `/etc/sudoers` | `cp /etc/sudoers /root/sudoers.bak` | Copy the backup back. Keep a **second root session open** while editing |
| `auditctl` (add/delete rules) | `auditctl -l > rules.bak` | `auditctl -D` (clear runtime rules); rules not written to `/etc/audit/rules.d/` vanish at reboot |

---

> **Navigation:** [◀ Phase 12 — Bridge to the Cloud](Phase_12_Bridge_to_the_Cloud.md) · **Appendix A** · [Appendix B — File and Directory Map ▶](Appendix_B_File_Directory_Map.md)
