# Appendix C — "When It Breaks" Quick Reference

> **Navigation:** [◀ Appendix B — File and Directory Map](Appendix_B_File_Directory_Map.md) · **Appendix C** · [Phase 12 — Bridge to the Cloud ▶](Phase_12_Bridge_to_the_Cloud.md)

---

> **This appendix merges every "When this phase breaks" table in the workbook into a single hunting page.** When
> you see a symptom, look here: likely cause, which command confirms it, which phase you go back to. This is the
> pocket form of Phase 11's diagnostic reflex.

**Contents**
- [C.1 First: the diagnostic reflex (Phase 11)](#c1-first-the-diagnostic-reflex-phase-11)
- [C.2 Access and permissions (Phase 2, 6, 9)](#c2-access-and-permissions-phase-2-6-9)
- [C.3 Processes and resources (Phase 3, 4)](#c3-processes-and-resources-phase-3-4)
- [C.4 Boot and services (Phase 5, 8)](#c4-boot-and-services-phase-5-8)
- [C.5 Storage (Phase 6)](#c5-storage-phase-6)
- [C.6 Networking (Phase 7)](#c6-networking-phase-7)
- [C.7 Scripts and automation (Phase 10)](#c7-scripts-and-automation-phase-10)
- [C.8 Cloud-specific (Phase 12)](#c8-cloud-specific-phase-12)

---

## C.1 First: the diagnostic reflex (Phase 11)

Don't panic, don't reboot. Take it in order:

| Order | Layer | First command |
|---|---|---|
| 1 | Application log | `journalctl -u <service> -e` |
| 2 | Service status | `systemctl status <service>` |
| 3 | Resource | `top` · `free -h` · `dmesg \| grep -i oom` |
| 4 | Network | `ss -tulpn` · `curl localhost` |
| 5 | Kernel | `dmesg -T \| tail` |

> **Golden rule:** Gather evidence at each layer, and move to the next only after you've ruled out the one in
> hand. "I rebooted and it fixed itself" is not a diagnosis — it's destroying the evidence.

## C.2 Access and permissions (Phase 2, 6, 9)

| Symptom | Likely cause | Confirm | Phase |
|---|---|---|---|
| "Permission denied" (file) | Wrong rwx permission / owner | `ls -l <file>`, `id` | 2 |
| "Permission denied" but permissions look right | AppArmor/SELinux (MAC) blocking | `dmesg \| grep -i denied`, `aa-status` | 9 |
| File exists but "No such file" | Not mounted (existing ≠ accessible) | `df -h`, `mount`, `findmnt` | 6 |
| Can't open port <1024 (non-root) | Missing capability | `getcap`, `CAP_NET_BIND_SERVICE` | 9 |
| SSH "permissions too open" | Loose `~/.ssh` / key permissions | `ls -l ~/.ssh` → `chmod 600` | 9 |
| sudo doesn't work | sudoers rule / missing group | `sudo -l`, `visudo` | 2, 9 |

## C.3 Processes and resources (Phase 3, 4)

| Symptom | Likely cause | Confirm | Phase |
|---|---|---|---|
| System slow, CPU 100% | A process in a loop | `top` → which process | 4 |
| Memory full, system freezing | RAM exhausted, dropped to swap | `free -h`, `vmstat 1` | 4 |
| Process died suddenly, log "Killed" | OOM killer stepped in | `dmesg \| grep -i oom` | 4 |
| Slow but CPU/RAM normal | Disk IO bottleneck | `iostat -x 1` | 4 |
| Zombie processes (Z) piling up | Parent not calling wait() | `ps aux \| grep defunct` | 3 |
| Process stuck, unresponsive | Waiting in a system call | `strace -p <pid>` | 3, 11 |

## C.4 Boot and services (Phase 5, 8)

| Symptom | Likely cause | Confirm | Phase |
|---|---|---|---|
| Service won't start | Config error / dependency / permission | `journalctl -u <service> -e` | 8 |
| Service gone after reboot | Not enabled (running ≠ persistent) | `systemctl is-enabled <service>` | 5 |
| Service keeps restarting | Application crashing (Restart=on-failure) | `journalctl -u <service> -f` | 8 |
| Unit file changed but no effect | daemon-reload not run | `systemctl daemon-reload` | 8 |
| Crash before PID 1 (GRUB/initramfs) | systemctl/journal can't see it | Read the console | 5 |
| Boot very slow | A service hanging | `systemd-analyze blame` | 5 |

## C.5 Storage (Phase 6)

| Symptom | Likely cause | Confirm | Phase |
|---|---|---|---|
| Mount gone after reboot | Not added to fstab | `cat /etc/fstab`, `blkid` | 6 |
| Disk 100%, system can't write | Log/data filled the disk | `df -h`, `du -sh /var/log/*` | 6, 11 |
| File deleted but space not freed | A process holds the file open | `lsof \| grep deleted` | 6, 11 |
| `/data` came up empty | EBS not mounted / wrong device | `lsblk`, `findmnt` | 6, 12 |
| Device name changed (nvme) | Use UUID instead of device name | `blkid` → fstab UUID | 6 |

## C.6 Networking (Phase 7)

| Symptom | Likely cause | Confirm | Phase |
|---|---|---|---|
| Unreachable, from outside | App bound to 127.0.0.1 (three lenses) | `ss -tulpn \| grep <port>` | 7 |
| Port listening but unreachable | Host firewall blocking | `ufw status` | 7 |
| DNS not resolving | resolv.conf / DNS server | `dig <domain>`, `cat /etc/resolv.conf` | 7 |
| `curl localhost` works, remote doesn't | Local is fine, network layer is the issue | `ss -tulpn` (bind address) | 7 |
| Connection refused (ECONNREFUSED) | Other side isn't listening | `ss -tulpn`, `nc -zv` | 7 |

## C.7 Scripts and automation (Phase 10)

| Symptom | Likely cause | Confirm | Phase |
|---|---|---|---|
| Script errored but kept going | No `set -e` | First line `set -euo pipefail` | 10 |
| `rm`/`cp` hit the wrong file | Unquoted variable (word splitting) | `"$var"`, dry-run with `echo` | 10 |
| `rm -rf /` disaster | Empty/unset variable, unquoted | `set -u` + `[ -n "$DIR" ]` | 10 |
| Broke on the second run | Not idempotent | `mkdir -p`, `grep -qxF \|\|` | 10 |
| Pipe thought it succeeded but crashed | No `pipefail` | `set -o pipefail` | 10 |

## C.8 Cloud-specific (Phase 12)

| Symptom (AWS) | Likely cause | Confirm | Phase |
|---|---|---|---|
| Instance "healthy" but no application | systemd service failed | `systemctl status`, `journalctl -u` | 8, 11, 12 |
| user-data didn't run | cloud-init script failed silently | `/var/log/cloud-init-output.log` | 10, 12 |
| Can't write to S3 but SG is open | IAM role missing/wrong (identity layer) | IAM console, instance metadata | 9, 12 |
| Unreachable but SG is 0.0.0.0/0 | App bound to 127.0.0.1 | `ss -tulpn` | 7, 12 |
| Instance crashed, no logs at all | CloudWatch agent never installed | Add agent to AMI/user-data | 11, 12 |
| Container keeps getting OOM killed | cgroup memory limit exceeded | Task definition memory limit | 3.6, 4, 12 |

> **The whole workbook in one sentence:** Beneath every symptom is a Linux truth. Don't panic; drop to the layer
> under the abstraction, find the evidence with the right command, and fix the root cause permanently. That
> reflex is the most valuable thing this workbook leaves you.

---

> **Navigation:** [◀ Appendix B — File and Directory Map](Appendix_B_File_Directory_Map.md) · **Appendix C** · [Phase 12 — Bridge to the Cloud ▶](Phase_12_Bridge_to_the_Cloud.md)
