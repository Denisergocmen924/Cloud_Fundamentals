# Appendix B — File and Directory Map

> **Navigation:** [◀ Appendix A — Command Glossary](Appendix_A_Command_Glossary.md) · **Appendix B** · [Appendix C — "When It Breaks" Quick Reference ▶](Appendix_C_When_It_Breaks_Quick_Reference.md)

---

> **This appendix is the quick answer to "where is this file, and what does it do?"** Next to each line is the
> phase it came from. On Linux "everything is a file" (Phase 0) — so understanding a system starts with looking
> at the right file.

**Contents**
- [B.1 Root directory tree — FHS (Phase 1)](#b1-root-directory-tree--fhs-phase-1)
- [B.2 Configuration files /etc (Phase 2, 5, 6, 7, 9)](#b2-configuration-files-etc)
- [B.3 Log files /var/log (Phase 5, 11)](#b3-log-files-varlog-phase-5-11)
- [B.4 /proc and /sys — the kernel's window (Phase 3, 4)](#b4-proc-and-sys--the-kernels-window-phase-3-4)
- [B.5 systemd unit files (Phase 5, 8)](#b5-systemd-unit-files-phase-5-8)
- [B.6 User and SSH files (Phase 2, 7)](#b6-user-and-ssh-files-phase-2-7)

---

## B.1 Root directory tree — FHS (Phase 1)

*(Phase 1.2 — Filesystem Hierarchy Standard)*

| Directory | What it contains | Phase |
|---|---|---|
| `/` | Root — where everything begins | 1 |
| `/bin`, `/usr/bin` | User commands (ls, cat, grep) | 1 |
| `/sbin`, `/usr/sbin` | System commands (mount, systemctl) | 1 |
| `/etc` | System-wide configuration files | 1, 2 |
| `/var` | Changing data (logs, queues, DB) | 1, 11 |
| `/var/log` | System and service logs | 5, 11 |
| `/tmp` | Temporary files (may be cleared on reboot) | 1 |
| `/home/<user>` | User home directories | 2 |
| `/root` | The root user's home directory | 2 |
| `/opt` | Third-party / your own applications | 8 |
| `/proc` | Virtual filesystem of the kernel and processes | 3, 4 |
| `/sys` | Kernel/hardware virtual filesystem | 4 |
| `/dev` | Device files (disks, terminals) | 6 |
| `/mnt`, `/media` | Temporary mount points | 6 |
| `/boot` | Kernel, initramfs, GRUB | 5 |
| `/lib`, `/usr/lib` | Shared libraries | 8 |

## B.2 Configuration files /etc

| File/Directory | What it does | Phase |
|---|---|---|
| `/etc/passwd` | User accounts (not passwords) | 2 |
| `/etc/shadow` | Password hashes (restricted access) | 2 |
| `/etc/group` | Group definitions | 2 |
| `/etc/sudoers` (+ `/etc/sudoers.d/`) | sudo privilege rules (edit with `visudo`) | 2, 9 |
| `/etc/fstab` | Persistent mount definitions (by UUID) | 6 |
| `/etc/hostname` | Machine name | 5 |
| `/etc/hosts` | Local name→IP mapping | 7 |
| `/etc/resolv.conf` | DNS server settings | 7 |
| `/etc/netplan/*.yaml` | Network configuration (Ubuntu) | 7 |
| `/etc/ssh/sshd_config` | SSH server settings (hardening) | 7, 9 |
| `/etc/systemd/system/` | Custom systemd unit files | 5, 8 |
| `/etc/logrotate.d/` | Log-rotation rules | 11 |
| `/etc/apparmor.d/` | AppArmor profiles (MAC) | 9 |
| `/etc/crontab`, `/etc/cron.d/` | Scheduled tasks | 5 |

## B.3 Log files /var/log (Phase 5, 11)

*(Phase 11.1 — the first source of evidence)*

| File | What it contains | Phase |
|---|---|---|
| `/var/log/syslog` | General system logs (Debian/Ubuntu) | 11 |
| `/var/log/auth.log` | Authentication: SSH, sudo, failed logins | 9, 11 |
| `/var/log/kern.log` | Kernel messages (a copy of the ring buffer that `dmesg` reads directly; written by rsyslog) | 4, 11 |
| `/var/log/cloud-init-output.log` | cloud-init/user-data output (AWS boot) | 5, 10, 12 |
| `/var/log/dpkg.log` | Package install/remove history | 8 |
| `journalctl` (not a file) | systemd journal — central log | 5, 11 |

> **Note:** Modern services write to journald (`journalctl -u <service>`); many traditional services still write
> to `/var/log`. Know how to look at both (Phase 11.1).

## B.4 /proc and /sys — the kernel's window (Phase 3, 4)

| Path | What it shows | Phase |
|---|---|---|
| `/proc/<pid>/status` | Process state, memory, owner | 3 |
| `/proc/<pid>/fd/` | The process's open file descriptors | 3, 11 |
| `/proc/<pid>/limits` | Process resource limits | 4 |
| `/proc/<pid>/cgroup` | The process's cgroup (resource limit) | 3, 4 |
| `/proc/meminfo` | System memory status (`free` reads this) | 4 |
| `/proc/cpuinfo` | CPU information | 4 |
| `/proc/mounts` | Active mounts | 6 |
| `/sys/fs/cgroup/` | cgroup hierarchy (container resource limits) | 3.6, 4 |

## B.5 systemd unit files (Phase 5, 8)

*(Phase 8.2 — from script to service)*

| Path | What it does | Phase |
|---|---|---|
| `/etc/systemd/system/<name>.service` | Custom service units you write | 8 |
| `/lib/systemd/system/` | Units installed by packages | 8 |
| `/etc/systemd/system/*.target.wants/` | Which services start in a target (enable) | 5 |

**The skeleton of a `.service` file:**
```ini
[Unit]
Description=...
After=network.target          # dependency order (Phase 5)

[Service]
ExecStart=/path/app
Restart=on-failure            # restart on crash (Phase 8)
User=myapp                    # not root (Phase 9)

[Install]
WantedBy=multi-user.target    # start at boot (Phase 5 — persistent definition)
```

## B.6 User and SSH files (Phase 2, 7)

| File | What it does | Phase |
|---|---|---|
| `~/.ssh/authorized_keys` | Public keys allowed to log in as this user | 7 |
| `~/.ssh/id_ed25519` (`.pub`) | Private/public key pair | 7 |
| `~/.ssh/config` | SSH client shortcuts | 7 |
| `~/.bashrc`, `~/.bash_profile` | User shell configuration | 1 |
| `~/.bash_history` | Command history | 1 |

> **Permission warning (Phase 9):** SSH **refuses** the `~/.ssh/` directory unless it is `700`, the private key
> `600`, and `authorized_keys` `600` — the "permissions too open" error is exactly this. Phase 2's permission
> bits become a security rule here.

---

> **Navigation:** [◀ Appendix A — Command Glossary](Appendix_A_Command_Glossary.md) · **Appendix B** · [Appendix C — "When It Breaks" Quick Reference ▶](Appendix_C_When_It_Breaks_Quick_Reference.md)
