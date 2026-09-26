# Linux — Offline Çalışma Kitabı

> **Cloud Engineer Temel Yol Haritası — Linux / İşletim Sistemi**
> Bu, [`Cloud_linux_roadmap_temel_tr.md`](../../../Cloud%20Learning%20With%20Mentor/Turkish/Cloud_linux_roadmap_temel_tr.md) yol
> haritasının **tek başına, YZ mentoru olmadan, internetsiz** baştan sona işlenebilir hâlidir.

> 🇬🇧 English version: **[README_en.md](../English_Linux/README_en.md)**

---

## Bu kitap nedir, ana haritadan farkı ne?

Mentor klasöründeki yol haritası dosyası bir **iskelettir**: bir YZ mentoruna verilir, mentor
konuları sırayla açar. İskelet tek başına okunduğunda "ne öğrenmem gerektiğini" söyler ama
"onu öğretmez".

Bu kitap o iskeletin **eti**dir. Aynı fazlar, aynı alt maddeler, aynı derinlik etiketleri —
ama her madde burada:

- **anlatılır** (sezgi → mekanizma → komut çıktısı),
- **sorulur** (senin düşünmen için bırakılan sorular),
- **cevaplanır** (aynı dosyanın ilerleyen bölümünde),
- **bağlanır** (önceki maddeye ve AWS'teki karşılığına).

Hiçbir yerde "bunu mentoruna sor" demez. Dosyayı bir uçakta, internetsiz, tek başına
baştan sona işleyebilirsin.

---

## Bu kitabın donanım kitabından farkı — önemli

Donanım kitabında `[uygulama]` "bir mimari kararla ilişkilendir" demekti.
**Burada `[uygulama]` gerçekten "kendi makinende çalıştır" demektir.**

Linux öğrenmenin başka yolu yok. Bir `ps aux` çıktısını okumayı okuyarak öğrenemezsin —
kendi makinendeki çıktıya bakman gerekir. Bu yüzden bu kitapta **her komutun beklenen
çıktısı yazılıdır**: makinen yoksa çıktıyı okur, satır satır yorumunu takip edersin;
makinen varsa çalıştırır, kendi çıktınla karşılaştırırsın.

> **Makinen yoksa ne yapmalısın?** Hiçbir şey. Bu kitap makinesiz de baştan sona
> işlenir — her çıktı yazılı ve yorumlanmıştır. Ama bir Ubuntu makinesi (fiziksel,
> VM, WSL2 veya bir EC2 t3.micro) bulabilirsen öğrenme hızın **iki katına çıkar**.

---

## Dosyalar

| Dosya | Faz | Konu | Yaklaşık süre |
|---|---|---|---|
| [Faz_0_Zihinsel_Model.md](Faz_0_Zihinsel_Model.md) | 0 | Kernel/user space, syscall, "her şey dosyadır", distro manzarası | 3–4 saat |
| [Faz_1_Shell_ve_Dosya_Sistemi.md](Faz_1_Shell_ve_Dosya_Sistemi.md) | 1 | Shell, FHS, akışlar, pipe, metin işleme cephanesi | 5–7 saat |
| [Faz_2_Kullanicilar_ve_Izinler.md](Faz_2_Kullanicilar_ve_Izinler.md) | 2 | UID/GID, rwx, özel bitler, sudo, OS kimliği vs IAM | 4–6 saat |
| [**Ara_Sinav_1.md**](Ara_Sinav_1.md) | 0–2 | Birleşik sınav: model + shell + izinler | 1 saat |
| [Faz_3_Process_ve_Kaynak.md](Faz_3_Process_ve_Kaynak.md) | 3 | PID, fork/exec, durumlar, sinyaller, cgroup, namespace | 6–8 saat |
| [Faz_4_Bellek_IO_Performans.md](Faz_4_Bellek_IO_Performans.md) | 4 | RSS/VSZ, page cache, swap/OOM, load average, darboğaz triyajı | 6–8 saat |
| [**Ara_Sinav_2.md**](Ara_Sinav_2.md) | 3–4 | Birleşik sınav: "kaynağı ne yiyor" | 1 saat |
| [Faz_5_Boot_Init_systemd.md](Faz_5_Boot_Init_systemd.md) | 5 | Boot zinciri, systemd unit'leri, journald, cloud-init | 6–8 saat |
| [Faz_6_Depolama_ve_Dosya_Sistemleri.md](Faz_6_Depolama_ve_Dosya_Sistemleri.md) | 6 | Blok cihaz, mount, fstab, ext4/xfs, inode, LVM, EBS akışı | 5–7 saat |
| [**Ara_Sinav_3.md**](Ara_Sinav_3.md) | 5–6 | Birleşik sınav: "boot'tan servise" + "veri nerede" | 1 saat |
| [Faz_7_Ag_ve_Baglanti.md](Faz_7_Ag_ve_Baglanti.md) | 7 | `ip`, DNS istemcisi, SSH derinlemesine, `ss`, host firewall | 5–7 saat |
| [Faz_8_Paketler_ve_Servislestirme.md](Faz_8_Paketler_ve_Servislestirme.md) | 8 | apt/dnf, kendi uygulamanı systemd servisi yapmak, immutable AMI | 4–6 saat |
| [Faz_9_Guvenlik_ve_Sertlestirme.md](Faz_9_Guvenlik_ve_Sertlestirme.md) | 9 | En az yetki, SSH sıkılaştırma, AppArmor/SELinux, auditd, capabilities, sırlar | 5–7 saat |
| [**Ara_Sinav_4.md**](Ara_Sinav_4.md) | 7–9 | Birleşik sınav: dış dünya + savunma | 1 saat |
| [Faz_10_Otomasyon_ve_Scripting.md](Faz_10_Otomasyon_ve_Scripting.md) | 10 | Bash temeli, `set -euo pipefail`, quoting, idempotency, IaC köprüsü | 5–7 saat |
| [Faz_11_Gozlemlenebilirlik_ve_Troubleshooting.md](Faz_11_Gozlemlenebilirlik_ve_Troubleshooting.md) | 11 | Loglar, katman-katman metodoloji, araç ustalığı, adli refleks | 6–8 saat |
| [Faz_12_Cloud_a_Kopru.md](Faz_12_Cloud_a_Kopru.md) | 12 | AMI → cloud-init → SSH → EBS → systemd → log → hardening | 4–5 saat |

**Ekler:**

| Dosya | Ne işe yarar |
|---|---|
| [EK_A_Komut_Sozlugu.md](EK_A_Komut_Sozlugu.md) | Tüm komutlar, amacı ve risk işaretiyle (🟢🟡🔴), geçtiği fazla — masaüstünde açık tut |
| [EK_B_Dosya_Dizin_Haritasi.md](EK_B_Dosya_Dizin_Haritasi.md) | Hangi dosya nerede, ne işe yarar — FHS, `/etc`, `/var/log`, `/proc`, systemd birimleri, SSH dosyaları |
| [EK_C_Bozulunca_Hizli_Basvuru.md](EK_C_Bozulunca_Hizli_Basvuru.md) | Belirti → muhtemel neden → doğrulama komutu → faz; tüm "Bozulunca" tablolarının tek sayfalık birleşimi |

---

## Nasıl çalışılır

### 1. Sırayı bozma

Her faz bir öncekinin **kelimeleriyle** konuşur. Faz 4 "OOM Killer"ı anlatırken Faz 3'teki
process durumlarını, Faz 0'daki kernel/user space ayrımını kullanır. Faz 11'in tamamı
önceki on bir fazın "Bozulunca" notlarının üstüne kuruludur — tek başına okunursa bir
komut listesine dönüşür.

**Tek istisna:** Faz 1'in navigasyon kısmı (`cd`, `ls`, `cp`, `mv`). Bunları biliyorsan
hızlı geç — ama `1.4` (akışlar ve yönlendirme) ve `1.5` (metin işleme) bilinen konular
değildir, atlanmamalıdır.

### 2. Kutuları ciddiye al

Metinde dört tür kutu var. Hepsinin farklı bir işi var:

> **🤔 Düşün 3.2** — Buradaki soruyu **okur okumaz cevaplama.** Kalem al, 2 dakika düşün,
> tahminini yaz. Cevabı fazın sonundaki *"Düşün sorularının cevapları"* bölümünde.
> Tahminin yanlış çıkarsa bu iyi bir şey — yanlış tahmin, doğru cevabı kalıcı yapar.

**❓ Akla gelen soru:** Okurken doğal olarak oluşan soru. Cevabı **hemen altında**.
Bunlar beklemez, çünkü cevaplanmazsa okuma akışını kırar.

**⚠️ Yaygın yanılgı:** Çoğu insanın yanlış bildiği şey. Bunları okurken "ben de böyle
biliyordum" diyorsan, o cümleyi işaretle.

**🔧 Makinende gör:** Komut + **beklenen çıktı** + çıktının satır satır yorumu.
Kutunun başında üç işaretten biri vardır:

| İşaret | Anlamı |
|---|---|
| 🟢 | **Salt okunur.** Sistemini değiştirmez, gönül rahatlığıyla çalıştır. |
| 🟡 | **Geçici değişiklik.** Makine yeniden başlayınca eski hâline döner. |
| 🔴 | **Kalıcı değişiklik.** Üretim makinesinde yapma. Ne yaptığını anlamadan çalıştırma. |

Bu kitapta 🔴 işaretli komutların yanında **her zaman geri alma adımı** yazılıdır — ya da geri
almanın olmadığı açıkça söylenir (`mkfs`, `apt purge`). Hepsi Ek A.11'de toplanmıştır.

### 3. Derinlik etiketlerine uy

| Etiket | Ne bekleniyor | Nasıl test edersin |
|---|---|---|
| `[kavram]` | Sezgisel açıklayabilmek | "Bunu bir arkadaşıma 30 saniyede anlatabilir miyim?" |
| `[mekanizma]` | Adım adım anlatabilmek | "Kalem kâğıtla akışı çizebilir miyim?" |
| `[uygulama]` | **Makinende çalıştırıp çıktıyı okuyabilmek** | "Bu çıktıda anormal olanı görür müydüm?" |
| `[atla]` | Sadece adını bilmek | Adını duyunca "o şu alandaydı" diyebilmek yeter |

### 4. Faz sonunu atlama

Her faz şununla biter:

1. **Bu faz bozulunca — arıza imzaları** — belirti → mekanizma → ilk bakılacak yer tablosu
2. **Düşün sorularının cevapları** — kendi tahminlerinle karşılaştır
3. **Sık sorulan sorular** — fazın etrafındaki "peki ya şu?" soruları
4. **Kendini sına** — 16–18 soruluk test + tam cevap anahtarı
5. **Kapanış ve köprü** — bu fazdan ne kaldı, sonraki faz bunun neresine bağlanıyor

Testte %70'in altında kaldıysan **fazı tekrar etmek yerine**, yanlış yaptığın sorunun
işaret ettiği bölüme geri dön. Her cevap anahtarı hangi bölüme ait olduğunu söyler.

### 5. Ara sınavları atlama — bunlar farklı

Dört ara sınav (Faz 0–2, 3–4, 5–6, 7–9) faz sınavlarından **farklı bir şey ölçer.**
Faz sınavı "bu fazı anladın mı?" diye sorar; ara sınav "**bu fazları birbirine
bağlayabiliyor musun?**" diye sorar. Soruların çoğu tek bir fazdan cevaplanamaz.

Gerçek iş de böyledir: hiçbir arıza tek bir fazın içinde kalmaz.

---

## İlerleme takibi

```
[ ] Faz 0  — Zihinsel Model
[ ] Faz 1  — Shell ve Dosya Sistemi
[ ] Faz 2  — Kullanıcılar, İzinler ve Kimlik
[ ] ✅ Ara Sınav 1 (Faz 0–2)
[ ] Faz 3  — Process ve Kaynak Yönetimi
[ ] Faz 4  — Bellek, I/O ve Performans Sezgisi   ← en çok yanlış okunan metrikler
[ ] ✅ Ara Sınav 2 (Faz 3–4)
[ ] Faz 5  — Boot, Init ve systemd
[ ] Faz 6  — Depolama ve Dosya Sistemleri        ← fstab tuzağı burada
[ ] ✅ Ara Sınav 3 (Faz 5–6)
[ ] Faz 7  — Ağ (OS Katmanı)
[ ] Faz 8  — Paketler ve Servisleştirme
[ ] Faz 9  — Güvenlik ve Sertleştirme
[ ] ✅ Ara Sınav 4 (Faz 7–9)
[ ] Faz 10 — Otomasyon ve Scripting
[ ] Faz 11 — Gözlemlenebilirlik ve Troubleshooting ← haritanın zirvesi
[ ] Faz 12 — Cloud'a Köprü
```

---

## Kuzey yıldızı — üç içgüdü sorusu

Bu kitabın tamamı şu üç soruyu **düşünmeden** cevaplayabilmen için var:

| # | Soru | Nerede cevaplanıyor |
|---|---|---|
| 1 | **Sistem şu an ne yapıyor, kaynağı ne yiyor?** | Faz 3, 4, 11 |
| 2 | **Neden erişemiyorum / neden "permission denied"?** | Faz 2, 6, 7, 9 |
| 3 | **Bu makine boot'tan çalışan servise nasıl geliyor?** | Faz 5, 8, 12 |

Bu üç soru, bir cloud engineer'in işinin yaklaşık %80'idir. Geri kalan %20 bunları
otomatikleştirmektir (Faz 10).

**Tasarım prensibi:** troubleshooting sona bırakılmadı. Her fazda *"bu bilgi bozulunca
nasıl görünür?"* açısı var. İçgüdü ancak böyle oluşur — Faz 11 bu dağınık notları tek
bir reflekse dönüştürür.

---

## Bu kitabın hedefi ve sınırı

**Hedef:** Bir Linux sunucusuna SSH ile girdiğinde, "bu makinede ne oluyor" sorusunu
komut ezberiyle değil **sistem modeliyle** cevaplayabilmek; bir arızayı tahminle değil
**metodolojiyle** daraltabilmek.

**Sınır:** Bu kitap kernel geliştiricisi yetiştirmez. Kernel modülü yazma, scheduler
iç yapısı, device driver geliştirme kapsam dışıdır. Amaç, sistemi **kararları
gerekçelendirecek ve arızayı avlayacak** kadar derinden görmektir.

**Dağıtım varsayımı:** Örnekler **Ubuntu 22.04/24.04** üzerindedir. Amazon Linux /
RHEL farkları ilgili yerde ayrıca not edilir — bu fark cloud'da sürekli karşına çıkar.

---

## Ana haritayla ilişki

| Bu kitap | Ana harita |
|---|---|
| Offline, tek başına işlenir | YZ mentoruna verilir |
| Anlatım + soru + cevap | Konu listesi + mentor talimatı |
| Sabit içerik | Öğrencinin seviyesine göre mentor kalibre eder |
| Kendini sına testleri | Mentor sözlü yoklar |

İkisi **rakip değil**. Sıralı kullanım önerilir: önce bu kitapla fazı işle, sonra ana
haritayı bir YZ mentoruna verip aynı fazı **sözlü** tekrar et.

---

## Diğer kitaplarla ilişki

Bu seri üç kitaptan oluşur ve üçü birbirini **akıllıca sınırlar**:

| Kitap | Neyi alır | Bu kitapla kesişimi |
|---|---|---|
| [Donanım](../../Cloud_hardware.roadmap/Turkish_Hardware/README.md) | Fiziksel katman: CPU, bellek, disk, NIC, hypervisor | Faz 4'ün "neden yavaş" sorusunun **altı**: cache, IOPS, steal time |
| **Linux** (bu kitap) | İşletim sistemi: process, bellek yönetimi, boot, izin, araçlar | — |
| [Ağ](../../Cloud_network.roadmap/Turkish_Network/README.md) | Protokol teorisi: OSI, TCP/IP, subnet, DNS, TLS | Faz 7 burada sadece **OS tarafını** alır; protokolün kendisi orada |

Bu kitap diğer ikisinden **bağımsız yürür.** Donanım veya ağ kitabını okumadıysan hiçbir
yerde takılmazsın — gereken yerde ilgili kavram burada kısaca açıklanır, derinleşmek
istersen ilgili kitap işaret edilir.

---

*Bu offline sürüm, Denis Ergöçmen'in Cloud Engineer temel yol haritası serisine dayanır.*
