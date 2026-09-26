# Ek A — Komut Sözlüğü

> **Navigasyon:** [◀ Faz 12 — Cloud'a Köprü](Faz_12_Cloud_a_Kopru.md) · **Ek A** · [Ek B — Dosya ve Dizin Haritası ▶](EK_B_Dosya_Dizin_Haritasi.md)

---

> **Bu ek öğretmek için değil hatırlatmak için vardır.** Workbook'u bitirdikten sonra açık tutulacak sayfa
> budur. Her komutun yanında hangi fazdan geldiği yazılıdır — bir komut aklından çıktığında oraya dön. Risk
> işaretleri: 🟢 salt-okunur (güvenli), 🟡 geçici değişiklik, 🔴 kalıcı değişiklik (dikkatli ol).

**İçindekiler**
- [A.1 Shell ve dosya sistemi (Faz 1)](#a1-shell-ve-dosya-sistemi-faz-1)
- [A.2 Kullanıcılar ve izinler (Faz 2)](#a2-kullanıcılar-ve-izinler-faz-2)
- [A.3 Process ve kaynak (Faz 3-4)](#a3-process-ve-kaynak-faz-3-4)
- [A.4 Boot, systemd, servis (Faz 5, 8)](#a4-boot-systemd-servis-faz-5-8)
- [A.5 Depolama ve dosya sistemleri (Faz 6)](#a5-depolama-ve-dosya-sistemleri-faz-6)
- [A.6 Ağ (Faz 7)](#a6-ağ-faz-7)
- [A.7 Paketler (Faz 8)](#a7-paketler-faz-8)
- [A.8 Güvenlik ve sertleştirme (Faz 9)](#a8-güvenlik-ve-sertleştirme-faz-9)
- [A.9 Scripting (Faz 10)](#a9-scripting-faz-10)
- [A.10 Gözlemlenebilirlik ve teşhis (Faz 11)](#a10-gözlemlenebilirlik-ve-teşhis-faz-11)
- [A.11 Kalıcı değişiklikleri geri alma](#a11-kalıcı-değişiklikleri-geri-alma)

---

## A.1 Shell ve dosya sistemi (Faz 1)

| Komut | Ne yapar | Risk |
|---|---|---|
| `pwd` | Bulunulan dizini yazar | 🟢 |
| `ls -la` | Gizli dosyalar + izinler dahil listeler | 🟢 |
| `cd /path` | Dizin değiştirir | 🟢 |
| `which <komut>` | Komutun hangi dosyadan çalıştığını bulur (PATH araması) | 🟢 |
| `type <komut>` | Komut builtin mi, alias mı, dosya mı | 🟢 |
| `cat` / `less` / `head` / `tail` | Dosya içeriğini gösterir | 🟢 |
| `find /path -name "*.log"` | Dosya arar | 🟢 |
| `grep -r "desen" /path` | İçerik arar (recursive) | 🟢 |
| `cmd1 \| cmd2` | Pipe: cmd1'in stdout'unu cmd2'nin stdin'ine bağlar | 🟢 |
| `cmd > file` / `>> file` | stdout'u dosyaya yaz / ekle | 🟡 |
| `cmd 2>&1` | stderr'i stdout'a birleştir | 🟢 |
| `ln -s hedef isim` | Sembolik link oluşturur | 🟡 |

## A.2 Kullanıcılar ve izinler (Faz 2)

| Komut | Ne yapar | Risk |
|---|---|---|
| `id` | Mevcut kullanıcının UID/GID/grupları | 🟢 |
| `whoami` | Etkin kullanıcı adı | 🟢 |
| `ls -l` | Dosyanın rwx izinleri, sahip, grup | 🟢 |
| `stat <dosya>` | Dosyanın tüm meta verisi (izin, sahip, zaman) | 🟢 |
| `chmod 640 <dosya>` | İzin bitlerini değiştirir | 🔴 |
| `chown user:group <dosya>` | Sahip/grubu değiştirir | 🔴 |
| `sudo <komut>` | Komutu yükseltilmiş yetkiyle çalıştırır | 🔴 |
| `umask` | Yeni dosyaların varsayılan izin maskesi | 🟢 |
| `getfacl` / `setfacl` | Genişletilmiş ACL izinleri | 🟢/🔴 |

## A.3 Process ve kaynak (Faz 3-4)

| Komut | Ne yapar | Risk |
|---|---|---|
| `ps aux` | Tüm process'ler (durum, sahip, CPU/RAM) | 🟢 |
| `top` / `htop` | Canlı process ve kaynak izleme | 🟢 |
| `kill <pid>` / `kill -9 <pid>` | Process'e sinyal gönderir (TERM / KILL) | 🔴 |
| `pkill <isim>` | İsimle process öldürür | 🔴 |
| `nice` / `renice` | Process önceliğini ayarlar | 🟡 |
| `free -h` | RAM/swap durumu | 🟢 |
| `vmstat 1` | Sistem geneli (CPU/IO/bellek) canlı | 🟢 |
| `iostat -x 1` | Disk IO istatistikleri | 🟢 |
| `nohup cmd &` | Terminal kapansa da çalışmaya devam eder | 🟡 |
| `jobs` / `fg` / `bg` | Arka plan iş kontrolü | 🟢 |

## A.4 Boot, systemd, servis (Faz 5, 8)

| Komut | Ne yapar | Risk |
|---|---|---|
| `systemctl status <servis>` | Servisin durumu (active/failed) | 🟢 |
| `systemctl start/stop <servis>` | Servisi başlat/durdur (çalışan durum) | 🟡 |
| `systemctl enable/disable <servis>` | Boot'ta başlat/başlatma (kalıcı tanım) | 🔴 |
| `systemctl enable --now <servis>` | Hem başlat hem boot'a ekle | 🔴 |
| `systemctl daemon-reload` | Birim dosyası değişikliğini oku | 🟡 |
| `systemctl list-units --failed` | Başarısız birimler | 🟢 |
| `journalctl -u <servis>` | Bir birimin logları | 🟢 |
| `journalctl -b` / `-b -1` | Bu boot / önceki boot logları | 🟢 |
| `systemd-analyze` | Boot süresi analizi | 🟢 |
| `systemctl is-enabled/is-active <servis>` | Kalıcı/çalışan durumu sorgular | 🟢 |

## A.5 Depolama ve dosya sistemleri (Faz 6)

| Komut | Ne yapar | Risk |
|---|---|---|
| `lsblk` | Blok cihaz ağacı (disk, bölüm, mount) | 🟢 |
| `df -h` | Dosya sistemi doluluk oranları | 🟢 |
| `du -sh <dizin>` | Dizin boyutu | 🟢 |
| `mount /dev/xxx /mnt` | Dosya sistemini bağlar (çalışan durum) | 🔴 |
| `umount /mnt` | Bağlantıyı kaldırır | 🔴 |
| `mkfs.ext4 /dev/xxx` | Dosya sistemi oluşturur (**veri siler!**) | 🔴 |
| `blkid` | Cihazların UUID'leri | 🟢 |
| `findmnt` | Mount ağacını gösterir | 🟢 |
| `lsof <dosya>` | Dosyayı açık tutan process | 🟢 |

## A.6 Ağ (Faz 7)

| Komut | Ne yapar | Risk |
|---|---|---|
| `ss -tulpn` | Dinleyen port'lar + process (üç mercek: dinleme) | 🟢 |
| `ip a` | Ağ arayüzleri ve IP adresleri | 🟢 |
| `ip r` | Yönlendirme tablosu | 🟢 |
| `ping <host>` | Erişilebilirlik testi | 🟢 |
| `curl -v <url>` | HTTP isteği (ayrıntılı) | 🟢 |
| `dig <alan>` / `nslookup` | DNS çözümlemesi | 🟢 |
| `ufw status` / `ufw allow` | Host firewall durumu / kural | 🟢/🔴 |
| `traceroute <host>` | Paket yolu | 🟢 |
| `nc -zv host port` | Port erişim testi | 🟢 |

## A.7 Paketler (Faz 8)

| Komut | Ne yapar | Risk |
|---|---|---|
| `apt update` | Paket listesini yeniler | 🟡 |
| `apt install <paket>` | Paket kurar (+ bağımlılıklar) | 🔴 |
| `apt remove/purge <paket>` | Paket kaldırır | 🔴 |
| `dpkg -l` | Kurulu paketleri listeler | 🟢 |
| `dpkg -L <paket>` | Paketin kurduğu dosyalar | 🟢 |
| `apt list --installed` | Kurulu paketler | 🟢 |
| `systemctl` (bkz. A.4) | Kurulan servisin yaşam döngüsü | — |

## A.8 Güvenlik ve sertleştirme (Faz 9)

| Komut | Ne yapar | Risk |
|---|---|---|
| `ssh-keygen` | SSH anahtar çifti üretir | 🟡 |
| `sudo` / `visudo` | Yetki yükseltme / sudoers düzenleme | 🔴 |
| `aa-status` | AppArmor profil durumu (MAC) | 🟢 |
| `getenforce` / `sestatus` | SELinux durumu (MAC) | 🟢 |
| `dmesg \| grep -i denied` | MAC engel kayıtları | 🟢 |
| `getcap` / `setcap` | Dosya capability'leri | 🟢/🔴 |
| `auditctl` / `ausearch` | Denetim (audit) kayıtları | 🟢/🔴 |
| `chmod 600 ~/.ssh/id_*` | Özel anahtar izinlerini kısıtla | 🔴 |

## A.9 Scripting (Faz 10)

| Yapı | Ne yapar | Risk |
|---|---|---|
| `#!/usr/bin/env bash` | Shebang — yorumlayıcıyı belirtir | 🟢 |
| `set -euo pipefail` | Sağlam script: hata/tanımsız değişken/pipe hatasında dur | 🟢 |
| `"$var"` | Tırnaklı değişken — word splitting'i önler | 🟢 |
| `trap '...' EXIT` | Script bitince temizlik | 🟢 |
| `echo <destructive-cmd>` | Dry-run: ne yapacağını sil**meden** gör | 🟢 |
| `mkdir -p` / `grep -qxF \|\|` | Idempotent kalıplar | 🟡 |
| `$?` | Son komutun çıkış kodu | 🟢 |
| `cmd1 && cmd2` / `\|\|` | Çıkış koduna göre dallanma | değişir |

## A.10 Gözlemlenebilirlik ve teşhis (Faz 11)

| Komut | Ne yapar | Risk |
|---|---|---|
| `journalctl -u <servis> -e` | Servis logunun sonu (ilk kanıt durağı) | 🟢 |
| `journalctl -k` / `dmesg -T` | Çekirdek mesajları (OOM, disk, sürücü) | 🟢 |
| `systemctl status <servis>` | Servis durumu (2. katman) | 🟢 |
| `top` / `free` / `iostat` | Kaynak (3. katman) | 🟢 |
| `ss -tulpn` | Ağ (4. katman) | 🟢 |
| `strace -p <pid>` | Process'in sistem çağrıları (nihai hakem) | 🟡 |
| `lsof -p <pid>` | Process'in açık dosya/soketleri | 🟢 |
| `/proc/<pid>/status` | Process detayları | 🟢 |
| `grep -i "failed" /var/log/auth.log` | Başarısız girişler | 🟢 |

> **Teşhis refleksi (Faz 11.2):** Bir şey bozulunca sıra hep aynı — **log → servis → kaynak → ağ → çekirdek.**
> Rastgele komut deneme; her adımda kanıt topla, bir sonrakine ancak eldeki katmanı eledikten sonra geç.

## A.11 Kalıcı değişiklikleri geri alma

Yukarıdaki her 🔴 komutun bir geri dönüş yolu vardır — ya da geri dönüşün **olmadığı** açıkça yazılır.
Değiştirmeden önce **eski durumu kaydet.**

| Komut | Önce kaydet | Geri alma |
|---|---|---|
| `chmod` (her mod, `chmod 600 ~/.ssh/id_*` dahil) | `stat -c '%a' <dosya>` | `chmod <eski mod> <dosya>` |
| `chown user:group <dosya>` | `stat -c '%U:%G' <dosya>` | `chown <eski kullanıcı>:<eski grup> <dosya>` |
| `sudo <komut>` | — (`sudo` kendi başına bir şey değiştirmez) | `<komut>`'un yaptığını geri al |
| `setfacl` | `getfacl -R <yol> > acl.bak` | `setfacl -b <dosya>` (tüm ACL'leri sil) veya `setfacl --restore=acl.bak` |
| `setcap` | `getcap <dosya>` | `setcap -r <dosya>` |
| `kill` / `kill -9` / `pkill` | `ps -o pid,cmd -p <pid>` | **Process'in bellek durumu geri gelmez.** Yeniden başlat: `systemctl start <servis>`. `kill -9`'dan önce `kill` (TERM) dene |
| `systemctl enable` / `enable --now` | `systemctl is-enabled <servis>` | `systemctl disable` / `disable --now` |
| `systemctl disable` | `systemctl is-enabled <servis>` | `systemctl enable` |
| `mount /dev/xxx /mnt` | `findmnt` | `umount /mnt` |
| `umount /mnt` | `findmnt /mnt` | Yeniden `mount`; "target is busy" derse tutanı `lsof +f -- /mnt` ile bul |
| `mkfs.ext4 /dev/xxx` | `lsblk -f` — hedefi iki kez kontrol et | **Geri alma yok — veri gitti.** Sadece **önceden** alınmış snapshot/yedek kurtarır |
| `ufw allow` / `ufw enable` | `ufw status numbered` | `ufw delete <kural>` / `ufw disable`. SSH üzerindeyken etkinleştirmeden **önce** 22'yi izinle |
| `apt install <paket>` | `dpkg -l <paket>` | `apt remove <paket>`, sonra `apt autoremove` |
| `apt remove` / `purge` | `dpkg -L <paket>`, `/etc/<paket>`'i kopyala | `apt install <paket>`. **`purge` yapılandırmayı geri dönüşsüz siler** — önce yedekle |
| `visudo` / `/etc/sudoers` düzenleme | `cp /etc/sudoers /root/sudoers.bak` | Yedeği geri kopyala. Düzenlerken **ikinci bir root oturumu açık tut** |
| `auditctl` (kural ekle/sil) | `auditctl -l > rules.bak` | `auditctl -D` (çalışma anı kurallarını temizle); `/etc/audit/rules.d/`'ye yazılmayan kurallar yeniden başlatmada kaybolur |

---

> **Navigasyon:** [◀ Faz 12 — Cloud'a Köprü](Faz_12_Cloud_a_Kopru.md) · **Ek A** · [Ek B — Dosya ve Dizin Haritası ▶](EK_B_Dosya_Dizin_Haritasi.md)
