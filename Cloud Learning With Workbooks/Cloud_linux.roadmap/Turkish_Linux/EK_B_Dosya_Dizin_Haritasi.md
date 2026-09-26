# Ek B — Dosya ve Dizin Haritası

> **Navigasyon:** [◀ Ek A — Komut Sözlüğü](EK_A_Komut_Sozlugu.md) · **Ek B** · [Ek C — "Bozulunca" Hızlı Başvuru ▶](EK_C_Bozulunca_Hizli_Basvuru.md)

---

> **Bu ek, "bu dosya nerede, ne işe yarar" sorusunun hızlı cevabıdır.** Her satırın yanında hangi fazdan
> geldiği yazılıdır. Linux'ta "her şey bir dosyadır" (Faz 0) — bu yüzden bir sistemi anlamak, doğru dosyaya
> bakmakla başlar.

**İçindekiler**
- [B.1 Kök dizin ağacı — FHS (Faz 1)](#b1-kök-dizin-ağacı--fhs-faz-1)
- [B.2 Yapılandırma dosyaları /etc (Faz 2, 5, 6, 7, 9)](#b2-yapılandırma-dosyaları-etc)
- [B.3 Log dosyaları /var/log (Faz 5, 11)](#b3-log-dosyaları-varlog-faz-5-11)
- [B.4 /proc ve /sys — çekirdeğin penceresi (Faz 3, 4)](#b4-proc-ve-sys--çekirdeğin-penceresi-faz-3-4)
- [B.5 systemd birim dosyaları (Faz 5, 8)](#b5-systemd-birim-dosyaları-faz-5-8)
- [B.6 Kullanıcı ve SSH dosyaları (Faz 2, 7)](#b6-kullanıcı-ve-ssh-dosyaları-faz-2-7)

---

## B.1 Kök dizin ağacı — FHS (Faz 1)

*(Faz 1.2 — Filesystem Hierarchy Standard)*

| Dizin | Ne içerir | Faz |
|---|---|---|
| `/` | Kök — her şeyin başladığı yer | 1 |
| `/bin`, `/usr/bin` | Kullanıcı komutları (ls, cat, grep) | 1 |
| `/sbin`, `/usr/sbin` | Sistem komutları (mount, systemctl) | 1 |
| `/etc` | Sistem geneli yapılandırma dosyaları | 1, 2 |
| `/var` | Değişen veri (loglar, kuyruklar, DB) | 1, 11 |
| `/var/log` | Sistem ve servis logları | 5, 11 |
| `/tmp` | Geçici dosyalar (reboot'ta silinebilir) | 1 |
| `/home/<user>` | Kullanıcı ev dizinleri | 2 |
| `/root` | root kullanıcısının ev dizini | 2 |
| `/opt` | Üçüncü parti / kendi uygulamaların | 8 |
| `/proc` | Çekirdek ve process'lerin sanal dosya sistemi | 3, 4 |
| `/sys` | Çekirdek/donanım sanal dosya sistemi | 4 |
| `/dev` | Cihaz dosyaları (diskler, terminaller) | 6 |
| `/mnt`, `/media` | Geçici mount noktaları | 6 |
| `/boot` | Çekirdek, initramfs, GRUB | 5 |
| `/lib`, `/usr/lib` | Paylaşılan kütüphaneler | 8 |

## B.2 Yapılandırma dosyaları /etc

| Dosya/Dizin | Ne yapar | Faz |
|---|---|---|
| `/etc/passwd` | Kullanıcı hesapları (parola değil) | 2 |
| `/etc/shadow` | Parola hash'leri (kısıtlı erişim) | 2 |
| `/etc/group` | Grup tanımları | 2 |
| `/etc/sudoers` (+ `/etc/sudoers.d/`) | sudo yetki kuralları (`visudo` ile düzenle) | 2, 9 |
| `/etc/fstab` | Kalıcı mount tanımları (UUID ile) | 6 |
| `/etc/hostname` | Makine adı | 5 |
| `/etc/hosts` | Yerel isim→IP eşlemesi | 7 |
| `/etc/resolv.conf` | DNS sunucu ayarları | 7 |
| `/etc/netplan/*.yaml` | Ağ yapılandırması (Ubuntu) | 7 |
| `/etc/ssh/sshd_config` | SSH sunucu ayarları (sertleştirme) | 7, 9 |
| `/etc/systemd/system/` | Özel systemd birim dosyaları | 5, 8 |
| `/etc/logrotate.d/` | Log döndürme kuralları | 11 |
| `/etc/apparmor.d/` | AppArmor profilleri (MAC) | 9 |
| `/etc/crontab`, `/etc/cron.d/` | Zamanlanmış görevler | 5 |

## B.3 Log dosyaları /var/log (Faz 5, 11)

*(Faz 11.1 — ilk kanıt kaynağı)*

| Dosya | Ne içerir | Faz |
|---|---|---|
| `/var/log/syslog` | Genel sistem logları (Debian/Ubuntu) | 11 |
| `/var/log/auth.log` | Kimlik doğrulama: SSH, sudo, başarısız giriş | 9, 11 |
| `/var/log/kern.log` | Çekirdek mesajları (`dmesg`'in doğrudan okuduğu ring buffer'ın rsyslog'un yazdığı kopyası) | 4, 11 |
| `/var/log/cloud-init-output.log` | cloud-init/user-data çıktısı (AWS boot) | 5, 10, 12 |
| `/var/log/dpkg.log` | Paket kurulum/kaldırma geçmişi | 8 |
| `journalctl` (dosya değil) | systemd journal — merkezî log | 5, 11 |

> **Not:** Modern servisler journald'a yazar (`journalctl -u <servis>`), birçok geleneksel servis hâlâ
> `/var/log`'a. İkisine de bakmayı bil (Faz 11.1).

## B.4 /proc ve /sys — çekirdeğin penceresi (Faz 3, 4)

| Yol | Ne gösterir | Faz |
|---|---|---|
| `/proc/<pid>/status` | Process durumu, bellek, sahip | 3 |
| `/proc/<pid>/fd/` | Process'in açık dosya tanıtıcıları | 3, 11 |
| `/proc/<pid>/limits` | Process kaynak sınırları | 4 |
| `/proc/<pid>/cgroup` | Process'in cgroup'u (kaynak sınırı) | 3, 4 |
| `/proc/meminfo` | Sistem bellek durumu (`free` bunu okur) | 4 |
| `/proc/cpuinfo` | CPU bilgisi | 4 |
| `/proc/mounts` | Aktif mount'lar | 6 |
| `/sys/fs/cgroup/` | cgroup hiyerarşisi (container kaynak sınırları) | 3.6, 4 |

## B.5 systemd birim dosyaları (Faz 5, 8)

*(Faz 8.2 — script'ten servise)*

| Yol | Ne yapar | Faz |
|---|---|---|
| `/etc/systemd/system/<isim>.service` | Senin yazdığın özel servis birimleri | 8 |
| `/lib/systemd/system/` | Paketlerin kurduğu birimler | 8 |
| `/etc/systemd/system/*.target.wants/` | Bir hedefte hangi servisler başlar (enable) | 5 |

**Bir `.service` dosyasının iskeleti:**
```ini
[Unit]
Description=...
After=network.target          # bağımlılık sırası (Faz 5)

[Service]
ExecStart=/yol/uygulama
Restart=on-failure            # çökünce yeniden başlat (Faz 8)
User=myapp                    # root değil (Faz 9)

[Install]
WantedBy=multi-user.target    # boot'ta başla (Faz 5 — kalıcı tanım)
```

## B.6 Kullanıcı ve SSH dosyaları (Faz 2, 7)

| Dosya | Ne yapar | Faz |
|---|---|---|
| `~/.ssh/authorized_keys` | Bu kullanıcıya girebilecek açık anahtarlar | 7 |
| `~/.ssh/id_ed25519` (`.pub`) | Özel/açık anahtar çifti | 7 |
| `~/.ssh/config` | SSH istemci kısayolları | 7 |
| `~/.bashrc`, `~/.bash_profile` | Kullanıcı shell yapılandırması | 1 |
| `~/.bash_history` | Komut geçmişi | 1 |

> **İzin uyarısı (Faz 9):** SSH `~/.ssh/` dizinini `700`, özel anahtarı `600`, `authorized_keys`'i `600`
> değilse **reddeder** — "izinler çok açık" hatası tam olarak budur. Faz 2'nin izin bitleri burada güvenlik
> kuralı olur.

---

> **Navigasyon:** [◀ Ek A — Komut Sözlüğü](EK_A_Komut_Sozlugu.md) · **Ek B** · [Ek C — "Bozulunca" Hızlı Başvuru ▶](EK_C_Bozulunca_Hizli_Basvuru.md)
