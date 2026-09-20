# Ek C — "Bozulunca" Hızlı Başvuru

> **Navigasyon:** [◀ Ek B — Dosya ve Dizin Haritası](EK_B_Dosya_Dizin_Haritasi.md) · **Ek C** · [Faz 12 — Cloud'a Köprü ▶](Faz_12_Cloud_a_Kopru.md)

---

> **Bu ek, tüm workbook'un "Bu faz bozulunca" tablolarını tek bir avlanma sayfasında birleştirir.** Bir belirti
> gördüğünde buraya bak: muhtemel neden, hangi komutla doğrularsın, hangi faza dönersin. Bu, Faz 11'in teşhis
> refleksinin cep hâlidir.

**İçindekiler**
- [C.1 Önce: teşhis refleksi (Faz 11)](#c1-önce-teşhis-refleksi-faz-11)
- [C.2 Erişim ve izin (Faz 2, 6, 9)](#c2-erişim-ve-izin-faz-2-6-9)
- [C.3 Process ve kaynak (Faz 3, 4)](#c3-process-ve-kaynak-faz-3-4)
- [C.4 Boot ve servis (Faz 5, 8)](#c4-boot-ve-servis-faz-5-8)
- [C.5 Depolama (Faz 6)](#c5-depolama-faz-6)
- [C.6 Ağ (Faz 7)](#c6-ağ-faz-7)
- [C.7 Script ve otomasyon (Faz 10)](#c7-script-ve-otomasyon-faz-10)
- [C.8 Cloud'a özgü (Faz 12)](#c8-clouda-özgü-faz-12)

---

## C.1 Önce: teşhis refleksi (Faz 11)

Panik yapma, yeniden başlatma. Sırayla ele:

| Sıra | Katman | İlk komut |
|---|---|---|
| 1 | Uygulama logu | `journalctl -u <servis> -e` |
| 2 | Servis durumu | `systemctl status <servis>` |
| 3 | Kaynak | `top` · `free -h` · `dmesg \| grep -i oom` |
| 4 | Ağ | `ss -tulpn` · `curl localhost` |
| 5 | Çekirdek | `dmesg -T \| tail` |

> **Altın kural:** Her adımda kanıt topla, bir sonrakine ancak eldeki katmanı eledikten sonra geç. "Yeniden
> başlattım, düzeldi" bir teşhis değil, kanıtı yok etmektir.

## C.2 Erişim ve izin (Faz 2, 6, 9)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| "Permission denied" (dosya) | rwx izni / sahip yanlış | `ls -l <dosya>`, `id` | 2 |
| "Permission denied" ama izinler doğru | AppArmor/SELinux (MAC) engelliyor | `dmesg \| grep -i denied`, `aa-status` | 9 |
| Dosya var ama "No such file" | mount edilmemiş (var ≠ erişilebilir) | `df -h`, `mount`, `findmnt` | 6 |
| Port <1024 açılamıyor (non-root) | capability eksik | `getcap`, `CAP_NET_BIND_SERVICE` | 9 |
| SSH "permissions too open" | `~/.ssh` / anahtar izni gevşek | `ls -l ~/.ssh` → `chmod 600` | 9 |
| sudo çalışmıyor | sudoers kuralı / grup eksik | `sudo -l`, `visudo` | 2, 9 |

## C.3 Process ve kaynak (Faz 3, 4)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| Sistem yavaş, CPU %100 | Bir process döngüde | `top` → hangi process | 4 |
| Bellek doldu, sistem donuyor | RAM tükendi, swap'e düştü | `free -h`, `vmstat 1` | 4 |
| Process aniden öldü, log "Killed" | OOM killer devreye girdi | `dmesg \| grep -i oom` | 4 |
| Yavaşlık ama CPU/RAM normal | Disk IO darboğazı | `iostat -x 1` | 4 |
| Zombie process (Z) birikiyor | Parent wait() çağırmıyor | `ps aux \| grep defunct` | 3 |
| Process takılı, cevap yok | Bir sistem çağrısında bekliyor | `strace -p <pid>` | 3, 11 |

## C.4 Boot ve servis (Faz 5, 8)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| Servis başlamıyor | Config hatası / bağımlılık / izin | `journalctl -u <servis> -e` | 8 |
| Reboot sonrası servis gelmedi | enable edilmemiş (çalışan ≠ kalıcı) | `systemctl is-enabled <servis>` | 5 |
| Servis sürekli restart ediyor | Uygulama çöküyor (Restart=on-failure) | `journalctl -u <servis> -f` | 8 |
| Birim dosyası değişti ama etkisiz | daemon-reload yapılmadı | `systemctl daemon-reload` | 5 |
| PID 1'den önce çökme (GRUB/initramfs) | systemctl/journal göremez | Konsolu oku | 5 |
| Boot çok yavaş | Bir servis takılıyor | `systemd-analyze blame` | 5 |

## C.5 Depolama (Faz 6)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| Reboot sonrası mount kayboldu | fstab'a eklenmedi | `cat /etc/fstab`, `blkid` | 6 |
| Disk %100, sistem yazamıyor | Log/veri diski doldurdu | `df -h`, `du -sh /var/log/*` | 6, 11 |
| Dosya silindi ama yer boşalmadı | Process dosyayı açık tutuyor | `lsof \| grep deleted` | 6, 11 |
| `/data` boş geldi | EBS mount edilmedi / yanlış cihaz | `lsblk`, `findmnt` | 6, 12 |
| Cihaz adı değişti (nvme) | Cihaz adı yerine UUID kullan | `blkid` → fstab UUID | 6 |

## C.6 Ağ (Faz 7)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| Erişilemiyor, dışarıdan | Uygulama 127.0.0.1'e bind (üç mercek) | `ss -tulpn \| grep <port>` | 7 |
| Port dinliyor ama erişilemiyor | Host firewall engelliyor | `ufw status` | 7 |
| DNS çözülmüyor | resolv.conf / DNS sunucu | `dig <alan>`, `cat /etc/resolv.conf` | 7 |
| `curl localhost` çalışıyor, uzak çalışmıyor | Yerelde tamam, ağ katmanı sorunlu | `ss -tulpn` (bind adresi) | 7 |
| Bağlantı reddedildi (ECONNREFUSED) | Karşı taraf dinlemiyor | `ss -tulpn`, `nc -zv` | 7 |

## C.7 Script ve otomasyon (Faz 10)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| Script hata verdi ama devam etti | `set -e` yok | İlk satıra `set -euo pipefail` | 10 |
| `rm`/`cp` yanlış dosyayı işledi | Tırnaksız değişken (word splitting) | `"$var"`, `echo` ile dry-run | 10 |
| `rm -rf /` felaketi | Boş/tanımsız değişken tırnaksız | `set -u` + `[ -n "$DIR" ]` | 10 |
| İkinci çalıştırmada bozuldu | Idempotent değil | `mkdir -p`, `grep -qxF \|\|` | 10 |
| Pipe başarılı sandı ama çöktü | `pipefail` yok | `set -o pipefail` | 10 |

## C.8 Cloud'a özgü (Faz 12)

| Belirti (AWS) | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| Instance "healthy" ama uygulama yok | systemd servisi failed | `systemctl status`, `journalctl -u` | 8, 11, 12 |
| user-data çalışmadı | cloud-init script'i sessizce hata verdi | `/var/log/cloud-init-output.log` | 10, 12 |
| S3'e yazamıyor ama SG açık | IAM role eksik/yanlış (kimlik katmanı) | IAM konsolu, instance metadata | 9, 12 |
| Erişilemiyor ama SG 0.0.0.0/0 | Uygulama 127.0.0.1'e bind | `ss -tulpn` | 7, 12 |
| Instance çöktü, hiç log yok | CloudWatch agent baştan konmamış | Agent'ı AMI/user-data'ya ekle | 11, 12 |
| Container sürekli OOM killed | cgroup bellek sınırı aşıldı | Task tanımı bellek limiti | 3.6, 4, 12 |

> **Tüm workbook'un özeti — tek cümle:** Her belirtinin altında bir Linux gerçeği vardır. Panik yapma;
> soyutlamanın altındaki katmana in, doğru komutla kanıtı bul, kök nedeni kalıcı düzelt. Bu refleks, bu
> workbook'un sana bıraktığı en değerli şeydir.

---

> **Navigasyon:** [◀ Ek B — Dosya ve Dizin Haritası](EK_B_Dosya_Dizin_Haritasi.md) · **Ek C** · [Faz 12 — Cloud'a Köprü ▶](Faz_12_Cloud_a_Kopru.md)
