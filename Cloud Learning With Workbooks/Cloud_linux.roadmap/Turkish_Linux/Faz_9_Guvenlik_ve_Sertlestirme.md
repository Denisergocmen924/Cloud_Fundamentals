# Faz 9 — Güvenlik ve Sertleştirme (Hardening)

> **Navigasyon:** [◀ Faz 8 — Paketler ve Servisleştirme](Faz_8_Paketler_ve_Servislestirme.md) · **Faz 9** · [Ara Sınav 4 ▶](Ara_Sinav_4.md)

---

## Nereden geliyoruz

Faz 8'in sonunda sana kendi kurduğun bir tuzağı gösterdik: unit dosyasına `User=appuser` yazmıştın — root
yerine sınırlı bir kullanıcıyla çalıştırmak için. Bu aslında bir **güvenlik kararıydı** ve sen onu daha o
zaman verdin. Faz 9 bu içgüdüyü bir sisteme dönüştürüyor. Şimdiye kadar hep **daha fazla** yaptık: process
başlat (Faz 3), disk bağla (Faz 6), servisi ağa aç (Faz 7), paket kur (Faz 8). Faz 9 ilk kez tersini
soruyor: **neyi kısıtlamalıyım?**

Dört fazın bilgisi burada bir savunmaya dönüşüyor:

- **Faz 2 — kullanıcılar ve izinler.** DAC (Discretionary Access Control — sahip/grup/diğer, `rwx`) izin
  modelini gördün. Faz 9 bunun **üstüne** ikinci bir kilit (MAC) koyuyor ve "en az yetki" ilkesini
  sistematize ediyor.
- **Faz 7 — ağ yüzeyi.** SSH sıkılaştırma, `0.0.0.0` vs `127.0.0.1`, host firewall + Security Group. Faz 9
  bunları bir "yüzey daraltma" disiplini olarak topluyor.
- **Faz 8 — servisi kim çalıştırıyor.** `User=` satırı, servisin hangi kimlikle koştuğunu belirler. Faz 9
  bunu "ele geçirilirse ne kadar zarar verebilir" sorusuyla derinleştiriyor.

## Bu fazın sorusu

Faz 8 "yazılımı nasıl kurar ve çalıştırırım" dedi. Faz 9 tam tersini sorar:

> *"Kurduğum her paket, açtığım her port, verdiğim her yetki bir saldırı yüzeyi. Bu yüzeyi bilinçli olarak
> nasıl daraltırım — ve bir servis ele geçirilirse zararı nasıl sınırlarım?"*

Güvenlik bir ürün değil, bir **disiplindir** ve tek bir sihirli ayar yoktur. Onun yerine **katmanlı bir
savunma** (defense in depth) vardır: ağ katmanı (firewall + SG), erişim katmanı (SSH key-only, root
kapalı), yetki katmanı (en az yetki, non-root, capabilities), zorunlu erişim kontrolü (AppArmor/SELinux),
ve sırların yönetimi (diskte değil, IAM/Secrets Manager'da). Bir katman aşılsa bile diğerleri hâlâ ayakta
kalır. Bu fazın merkezinde iki fikir vardır: **en az yetki** (her şeye sadece gerektiği kadar) ve **patlama
yarıçapını daraltmak** (bir şey ele geçirilirse hasar ne kadar yayılabilir).

Bu fazın sonunda, bir sistemi bilinçli olarak sertleştirebilecek; "izinler doğru ama yine engelleniyor"
gibi kafa karıştıran bir arızayı MAC katmanına (AppArmor/SELinux) bağlayabilecek; ve sırları asla diske/koda
gömmeden bulut kimlik mekanizmalarıyla (IAM role) yönetmenin neden temel olduğunu anlatabileceksin.

---

## Bu fazın sonunda

- **En az yetki ilkesini** (least privilege) açıklayabilecek ve her servis/kullanıcı için "sadece gereken
  yetki" kararını verebileceksin
- SSH ve ağ yüzeyini sistematik olarak sıkılaştırabileceksin: key-only, root kapalı, gereksiz portlar
  kapalı, firewall (Faz 7'nin devamı)
- MAC'i (Mandatory Access Control) — AppArmor (Ubuntu) ve SELinux (RHEL) — DAC izinlerinin **üstünde ikinci
  bir kilit** olarak açıklayabilecek; "izinler doğru ama yine engelleniyor" arızasını buna bağlayabileceksin
- Denetim ve bütünlük araçlarını (auditd, rkhunter, dosya bütünlüğü) kavram düzeyinde bilecek; auditd log
  patlamasının journald'ı boğup sistemi dondurabileceği **gerçek** senaryoyu ve rate-limit/rotation gereğini
  açıklayabileceksin
- **Capabilities**'i root/non-root ikiliğini kıran ince yetkiler olarak (ör. yalnız port-bind) tanıyabileceksin
- Sırları diskte/koda gömmenin neden yanlış olduğunu; **Cloud:** IAM instance role (diskte key tutmadan
  yetki), SSM Parameter Store / Secrets Manager ve CIS benchmark'lı hardened AMI'nin bunu nasıl çözdüğünü
  anlatabileceksin

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden burada |
|---|---|---|---|
| 9.1 | En az yetki ilkesi | `[kavram]` | Tüm fazın omurgası: sadece gereken yetki |
| 9.2 | SSH ve ağ yüzeyi sıkılaştırma | `[uygulama]` | Faz 7'nin savunma disiplinine dönüşmesi |
| 9.3 | MAC — AppArmor / SELinux | `[mekanizma]` | DAC'ın üstünde ikinci kilit; "izin doğru ama engelli" |
| 9.4 | Denetim ve bütünlük | `[kavram]` | auditd/rkhunter; log patlaması tuzağı |
| 9.5 | Capabilities | `[kavram]` | root/non-root ikiliğini kıran ince yetkiler |
| 9.6 | Sırların yönetimi | `[uygulama]` | **Diske key gömme** — IAM role ile çöz |
| 9.7 | Bu faz bozulunca | — | Güvenlik/hardening arıza imzaları |

> **Bu fazda nasıl çalışmalı:** Bu fazın gözlem komutları 🟢'dir (`aa-status`, `ss -tulpn`,
> `systemctl status auditd`, `getcap`). Ama sertleştirme adımları doğası gereği 🔴'dır — yanlış bir SSH
> kuralı, yanlış bir AppArmor profili veya yanlış bir firewall kuralı seni **dışarı kilitleyebilir** ya da
> bir servisi durdurabilir. Faz 7'nin altın kuralı burada da geçerli: kritik değişiklikleri **ikinci bir
> açık oturum** dururken test et, her adımın bir geri alma yolu olsun. En öğretici deney: bir AppArmor
> profilini `complain` moduna alıp neyi engellediğini `journalctl`'de izlemek — "izinler doğru ama engelli"
> arızasını canlı görmek. Kendi test makinende çalış.

---
---

# 9.1 En Az Yetki İlkesi

## 9.1.1 "Sadece gerekeni ver": savunmanın omurgası `[kavram]`

Tüm güvenlik disiplininin tek cümlelik özü: **her kullanıcı, her servis, her process yalnızca işini yapmak
için gereken en az yetkiye sahip olmalı — bir fazla değil.** Buna **en az yetki ilkesi** (principle of
least privilege) denir ve bu fazdaki her şey aslında bunun bir uygulamasıdır.

Neden bu kadar merkezi? Çünkü güvenlik "hiç ele geçirilmemek" değildir — er ya da geç bir açık bulunur. Asıl
soru şudur: **bir şey ele geçirilirse hasar ne kadar yayılır?** Buna **patlama yarıçapı** (blast radius)
denir. Bir servis root olarak çalışıyorsa ve ele geçirilirse, saldırgan tüm makineyi ele geçirir. Aynı
servis sınırlı bir kullanıcıyla (Faz 8'deki `User=appuser`!) çalışıyorsa, saldırgan yalnızca o kullanıcının
görebildiği kadarını görür — patlama yarıçapı küçüktür. En az yetki, ele geçirilmeyi önlemez; ele geçirilme
olduğunda **hasarı sınırlar**.

> **⚠️ Yaygın yanılgı: "Her şeyi root çalıştırmak daha pratik, izin dertleriyle uğraşmam."**
>
> Tersine — bu, patlama yarıçapını maksimuma çıkarmaktır. Root çalışan bir servisteki tek bir açık = tüm
> makinenin düşmesi. Faz 8'de `User=appuser` yazman "pratiklikten ödün vermek" değil, bilinçli bir güvenlik
> yatırımıydı: o servis ele geçirilse bile saldırgan root olamaz, başka kullanıcıların dosyalarını okuyamaz,
> sistem dosyalarını değiştiremez. "İzinlerle uğraşmak" güvenliğin bir dert değil, ta kendisidir. Kural:
> bir servise root vermeden önce hep sor — "gerçekten root gerekiyor mu, yoksa sadece tek bir yetki mi?"
> (bu sorunun cevabı çoğu zaman 9.5'teki capabilities'tir).

## 9.1.2 Katmanlı savunma: tek kilit değil, çok kilit `[kavram]`

En az yetki tek başına yeterli değildir; onu **katmanlı bir savunmaya** (defense in depth) yerleştirirsin.
Fikir şudur: bir saldırganın hedefe (uygulaman ve verisi) ulaşması için **birden çok bağımsız katmanı**
aşması gerekir, ve bir katman aşılsa bile diğerleri hâlâ ayakta kalır. Bu fazın bölümleri aslında bu
katmanlardır — dıştan içe:

![Şekil 9.1 — Katmanlı savunma (defense in depth): bir saldırganın uygulamaya ve sırlarına ulaşması için sırayla bağımsız katmanları aşması gerekir. Dıştan içe: ① Ağ (Security Group + host firewall, Faz 7) → ② Erişim (SSH key-only, root kapalı, 9.2) → ③ En az yetki (non-root kullanıcı, capabilities, 9.1/9.5) → ④ MAC (AppArmor/SELinux, izinlerin üstünde ikinci kilit, 9.3) → ⑤ çekirdekte uygulama + sırlar (diske gömülmez, IAM role ile erişilir, 9.6). Her katman bağımsızdır; birinin aşılması tam ele geçirilme demek değildir — en az yetki patlama yarıçapını daraltır.](../diagrams/png/lx-9-01-defense-in-depth.png)

Bu model bütün fazın haritasıdır. Faz 7'de ilk iki katmanı (ağ + erişim) kurdun; Faz 9 içteki katmanları
(yetki, MAC, sırlar) ekler. Kritik nokta: katmanlar **bağımsızdır** — biri yanlış yapılandırılsa bile
(bir instance yanlış SG'ye düşse, bir izin fazla verilse) diğerleri hâlâ koruyabilir. "Tek bir güvenlik
duvarım var" demek, tek kilitli bir kapı demektir; katmanlı savunma, arka arkaya beş kilittir.

> **🤔 Düşün 9.1** — İki sunucun var. A'da web uygulaması `User=root` ile çalışıyor, tek savunma bir host
> firewall. B'de aynı uygulama `User=appuser` ile çalışıyor, üstelik SG + host firewall + AppArmor profili
> var. Uygulamanın kodunda uzaktan kod çalıştırmaya izin veren bir açık bulunuyor (ikisinde de aynı açık).
> (a) Saldırgan her iki sunucuda da bu açığı kullanabilir mi? (b) Ele geçirdikten **sonra** iki sunucudaki
> patlama yarıçapı neden farklı — hangi katmanlar B'de saldırganı sınırlar?
>
> *(Cevap: fazın sonunda)*

---
---

# 9.2 SSH ve Ağ Yüzeyi Sıkılaştırma

## 9.2.1 Yüzey daraltma: Faz 7'nin bir disipline dönüşmesi `[uygulama]`

Faz 7'de SSH'ı ve ağı **nasıl çalıştırdığını** öğrendin; Faz 9 aynı bilgiyi bir **daraltma disiplinine**
çeviriyor. Temel soru: "makinede dışarıya bakan ne kadar yüzey var, ve bunun ne kadarı gerçekten gerekli?"
Her açık port, her aktif servis, her izin verilen giriş yöntemi bir saldırı yüzeyidir. Sıkılaştırma bu
yüzeyi bilinçli olarak küçültmektir. Ağ tarafında dört temel adım (çoğu Faz 7'den tanıdık):

1. **SSH key-only + root kapalı** (Faz 7.3.4): `PasswordAuthentication no`, `PermitRootLogin no`.
   Brute-force'u anlamsız kılar, doğrudan root girişini keser.
2. **Gereksiz servisleri kapat.** `ss -tulpn` ile dinlenen portları denetle; ihtiyaç olmayan her servisi
   durdur ve `disable` et (Faz 5/8). Dinlemeyen bir port, saldırılamayan bir porttur.
3. **Firewall'u varsayılan-reddet yap.** Host firewall'da (ufw) ve Security Group'ta varsayılan politika
   "her şeyi kapat, sadece gerekeni aç" olmalı — beyaz liste, kara liste değil.
4. **Portları en dar kaynağa aç.** SSH'ı `0.0.0.0/0`'a (tüm internete) değil, sadece kendi ofis/VPN IP'ne
   aç; ya da hiç açma (SSM — Faz 7.3.4).

> **🔧 Makinende gör** 🟢 — dışa açık yüzeyi denetle
>
> ```
> $ sudo ss -tulpn                 # hangi portlar dışarı dinliyor (0.0.0.0)?
> $ systemctl list-units --type=service --state=running   # kaç servis koşuyor
> $ sudo ufw status verbose        # host firewall politikası (default deny mi?)
> ```
>
> İlk çıktıdaki her `0.0.0.0:<port>` satırı bir sorudur: "bu gerçekten dışarıdan erişilebilir olmalı mı?"
> Cevap "hayır"sa ya servisi `127.0.0.1`'e bind et (Faz 7.4.2) ya da firewall'da kapat. Yüzey daraltmanın
> en somut hâli budur.

## 9.2.2 Otomatik güvenlik güncellemeleri `[kavram]`

Yüzeyin bir boyutu da **zamandır**: bugün güvenli bir paket, yarın bir açık (CVE) yüzünden savunmasız hâle
gelebilir. `unattended-upgrades` gibi mekanizmalar güvenlik yamalarını otomatik uygular. Ama bu Faz 8'in
gerilimini geri getirir: otomatik yükseltme, sürüm pinning'in (8.1.2) tekrarlanabilirliğiyle çatışabilir.
Olgun pratik: **güvenlik yamaları** için otomatik güncelleme aç, ama **uygulama sürümlerini** immutable
imaj (8.4) ile sabit tut. İkisi farklı katmanlardır.

---
---

# 9.3 MAC — Mandatory Access Control

## 9.3.1 DAC'ın üstünde ikinci bir kilit `[mekanizma]`

Faz 2'de öğrendiğin izin modeli (`rwx`, sahip/grup/diğer) **DAC**'tır — Discretionary Access Control
(isteğe bağlı erişim kontrolü). "Discretionary" (isteğe bağlı) çünkü bir dosyanın sahibi izinleri
**kendi** belirler ve değiştirebilir. Bu güçlüdür ama bir zayıflığı vardır: bir process root olarak
çalışıyorsa (veya bir kullanıcı kandırıldıysa), DAC onu durduramaz — root her şeyi yapabilir.

**MAC** (Mandatory Access Control — zorunlu erişim kontrolü) bunun üstüne **ikinci, bağımsız bir kilit**
koyar. "Mandatory" (zorunlu) çünkü kuralları merkezi bir politika belirler ve process'in kendisi (root
bile) bunları gevşetemez. İki büyük uygulama: **AppArmor** (Ubuntu — dosya yoluna dayalı profiller) ve
**SELinux** (RHEL/Amazon Linux — etikete dayalı, daha ayrıntılı). İkisinin de fikri aynıdır: her programa,
"sen sadece şu dosyalara, şu ağ işlemlerine, şu yeteneklere erişebilirsin" diyen bir **profil** atarsın.
Profilin izin vermediği şeyi, DAC izinleri sonuna kadar açık olsa bile, program **yapamaz**.

## 9.3.2 "İzinler doğru ama yine engelleniyor" `[mekanizma]`

Bu, MAC'ın en klasik ve en kafa karıştırıcı arızasıdır — çünkü Faz 2 refleksin ("izinleri kontrol et")
seni yanıltır. Belirti: bir servis bir dosyaya/porta erişmeye çalışıyor, `ls -l` izinleri **kusursuz**
(`chmod`/`chown` doğru), ama erişim yine de "Permission denied" ile reddediliyor. DAC katmanında hiçbir
sorun yok — çünkü engel DAC'ta değil, **üstündeki MAC katmanındadır**. AppArmor/SELinux profili o process'in
o dosyaya/işleme erişmesine izin vermiyor.

> **🔧 Makinende gör** 🟢 — MAC katmanını denetle
>
> ```
> $ sudo aa-status                 # AppArmor: hangi profiller yüklü, hangileri enforce
> $ sudo journalctl -k | grep -i apparmor | tail    # engellenen işlemler (DENIED)
> # SELinux tarafı (RHEL):
> $ getenforce                     # Enforcing / Permissive / Disabled
> $ sudo ausearch -m avc -ts recent   # SELinux'un engellediği erişimler (AVC denials)
> ```
>
> Teşhis refleksi: DAC izinleri doğru görünüyor ama erişim engelleniyorsa, **bir üst katmana bak**.
> `journalctl -k | grep apparmor` (veya SELinux'ta `ausearch -m avc`) tam olarak neyin engellendiğini
> söyler. Çözüm profili düzeltmektir (izin verilen yola dosyayı ekle) — profili tamamen kapatmak (`disable`)
> değil, çünkü o zaman ikinci kilidi kaybedersin.

> **💡 Cloud bağlantısı — hardened AMI ve CIS benchmark:** Bulutta güvenlik profillerini sıfırdan
> yazmazsın; **CIS benchmark** gibi standartlara göre önceden sertleştirilmiş **hardened AMI**'ler
> kullanırsın. Bu imajlar makul AppArmor/SELinux profilleri, kapatılmış gereksiz servisler, sıkılaştırılmış
> SSH ayarları ve denetim yapılandırmasıyla gelir. Faz 8'in immutable felsefesiyle birleşince: güvenliği
> her makinede elle kurmak yerine, bir kez sertleştirilmiş imajı tarifeye koyarsın ve her makine güvenli
> doğar. "Güvenlik bir kurulum adımı değil, imajın bir özelliğidir."

> **🤔 Düşün 9.2** — Yeni kurduğun bir web sunucusu, `/var/www/data/` altındaki bir dizinden dosya
> okumaya çalışıyor ama "Permission denied" alıyor. `ls -ld /var/www/data` çıktısı `drwxr-xr-x` ve dosya
> sahibi doğru — DAC açısından sorun yok. `sudo -u www-data cat /var/www/data/x.txt` bile çalışıyor. (a)
> Engel büyük olasılıkla hangi katmanda? (b) Hangi tek komutla bunu doğrularsın, ve (c) doğru çözüm neden
> "profili kapatmak" değildir?
>
> *(Cevap: fazın sonunda)*

---
---

# 9.4 Denetim ve Bütünlük

## 9.4.1 Ne oldu, ve bir şey değişti mi: auditd, rkhunter `[kavram]`

Güvenliğin ikinci yarısı **görünürlüktür**: bir şey olduğunda (veya olduktan sonra) ne olduğunu görebilmek.
Üç araç kategorisi:

- **auditd** — çekirdek seviyesi bir **denetim günlüğü**. "Kim, ne zaman, hangi dosyaya erişti / hangi
  sistem çağrısını yaptı" gibi olayları kaydeder. Bir olay sonrası adli inceleme (forensics) ve uyumluluk
  (compliance) için kritiktir.
- **rkhunter / chkrootkit** — **rootkit tarayıcıları**. Sisteme gizlenmiş kötü amaçlı yazılım izlerini arar.
- **Dosya bütünlüğü** (AIDE gibi) — kritik dosyaların bir "parmak izini" (hash) alır ve sonradan
  değişip değişmediğini kontrol eder. "Birisi `/etc/passwd`'ü değiştirdi mi?" sorusunun cevabı.

## 9.4.2 Denetimin kendisi bir arıza kaynağı: log patlaması `[uygulama]`

İşte gerçek dünyadan, yaşanmış bir tuzak: denetim aracının **kendisi** sistemi çökertebilir. Senaryo:
auditd çok geniş bir kural setiyle yapılandırılır (her dosya erişimini logla gibi), sistem yoğun çalışır,
ve auditd saniyede binlerce olay üretmeye başlar. Bu loglar journald'a akar; journald boğulur, disk hızla
dolar (Faz 6'daki `df` dolması!), ve I/O darboğazı tüm sistemi yavaşlatır — sonunda makine ya donar ya da
disk dolduğu için servisler ölür. Güvenlik için kurduğun araç, bir **denial-of-service** kaynağına dönüşür.

Ders çift yönlüdür: (1) denetim kurallarını **dar** tut — her şeyi değil, gerçekten önemli olayları logla;
(2) **rate-limit ve rotation** şart — auditd'nin olay hızını sınırla, journald/log dosyalarını döndür
(Faz 5/6). Bu, güvenlik araçlarının bile "en az yetki / en az yük" ilkesine tabi olduğunu gösterir.

> **🔧 Makinende gör** 🟢 — denetim servisini ve log yükünü kontrol et
>
> ```
> $ sudo systemctl status auditd            # denetim servisi ayakta mı
> $ sudo journalctl --disk-usage            # journald ne kadar disk yiyor
> $ df -h /var/log                          # log diski doluyor mu (Faz 6!)
> ```
>
> `journalctl --disk-usage` ve `df -h /var/log` birlikte "loglar diski dolduruyor mu" sorusunu yanıtlar —
> bu, 9.4.2'deki log patlaması tuzağının erken uyarı göstergesidir.

---
---

# 9.5 Capabilities

## 9.5.1 root/non-root ikiliğini kırmak `[kavram]`

Şimdiye kadar dünya ikiye bölünmüş gibiydi: ya root'sun (her şeyi yaparsın) ya değilsin (kısıtlısın). Ama
gerçek ihtiyaçlar çoğu zaman ikisinin arasındadır. Klasik örnek: bir web sunucusu **80 portunu** dinlemek
ister; 1024 altındaki portları açmak geleneksel olarak root yetkisi gerektirir. Bu yüzden insanlar tüm
sunucuyu root çalıştırır — sırf tek bir yetki için! Bu, en az yetki ilkesinin (9.1) ihlalidir.

**Linux capabilities** bu ikiliği kırar: root'un tüm gücünü ~40 ayrı **ince yetkiye** böler ve bunları
tek tek verebilirsin. Web sunucusu örneğinde, tüm root yerine sadece `CAP_NET_BIND_SERVICE` yetkisini
verirsin — "sadece ayrıcalıklı portlara bind edebilirsin, başka hiçbir root gücün yok." Böylece process
80 portunu açar ama root'un geri kalan tehlikeli güçlerine (herhangi bir dosyayı okuma, kernel modülü
yükleme, başka process'leri öldürme) **sahip olmaz**. Patlama yarıçapı dramatik biçimde küçülür.

> **🔧 Makinende gör** 🟢 — bir binary'nin capabilities'ini gör
>
> ```
> $ getcap /usr/bin/ping
> /usr/bin/ping cap_net_raw=ep       # ping root değil ama raw soket açabiliyor
> $ sudo setcap 'cap_net_bind_service=+ep' /opt/myapp/server   # 🔴 ince yetki ver
> ```
>
> `ping`'in kendisi güzel bir örnektir: eskiden setuid-root'tu (tehlikeli); modern sistemlerde sadece
> `cap_net_raw` capability'siyle çalışır — ihtiyacı olan tek yetki. systemd unit'inde bunu
> `AmbientCapabilities=CAP_NET_BIND_SERVICE` ile de verebilirsin (Faz 8 bağlantısı).

---
---

# 9.6 Sırların Yönetimi

## 9.6.1 Ne YAPILMAMALI: koda/diske gömülü sırlar `[uygulama]`

Bir uygulama neredeyse her zaman **sırlara** ihtiyaç duyar: veritabanı parolası, API anahtarı, TLS özel
anahtarı. Yeni başlayan herkesin yaptığı ve en tehlikeli hata, bu sırları **koda veya diske gömmektir**:

```python
# ASLA BÖYLE YAPMA:
DB_PASSWORD = "s3cr3t-prod-password"      # kodun içinde
API_KEY = "AKIA...."                       # bir config dosyasında, düz metin
```

Neden felaket? Çünkü bu sır artık: git geçmişine girer (silsen bile geçmişte kalır), her kod kopyasında
bulunur, loglara sızabilir, ve makineye erişen herkes tarafından okunur. Bir kez sızmış sır, sızmış
sırdır — döndürmekten (rotate) başka çare yoktur. Faz 2'nin izin dersi burada yetmez: `chmod 600` bir
config dosyasını korur ama sır hâlâ **diskte düz metindir** ve root (veya ele geçiren) okur.

## 9.6.2 Doğru yol: sırlar diskte değil, kimlikte `[uygulama]`

Doğru model, sırrı makineden tamamen **çıkarmaktır**. İki yaklaşım:

- **Secrets yönetim servisi.** Sırları merkezi, şifreli bir kasada (AWS Secrets Manager, SSM Parameter
  Store, HashiCorp Vault) tut. Uygulama sırrı çalışma anında oradan çeker; disk hiçbir zaman düz metin sır
  tutmaz. Sır döndürüldüğünde tek yerde döner.
- **Kimlik tabanlı yetki (IAM role).** En güçlü yol — sırrı tamamen ortadan kaldırır. Uygulamaya bir
  parola vermek yerine, çalıştığı makineye bir **kimlik** (IAM instance role) verirsin. Uygulama "ben bu
  makineyim" der ve bulut ona geçici, otomatik dönen kimlik bilgileri verir. Diske hiçbir key yazılmaz.

> **💡 Cloud bağlantısı — IAM instance role: diskte sıfır sır:** Bir EC2 instance'ına bir **IAM role**
> eklediğinde, üstündeki uygulama S3'e/RDS'e/Secrets Manager'a erişmek için hiçbir anahtar saklamak zorunda
> kalmaz. Instance metadata servisi ona geçici, kısa ömürlü, otomatik dönen kimlik bilgileri verir. Bu, Faz
> 7.3'teki "SSH anahtarını hiç dağıtma, SSM kullan" fikrinin veri erişimi versiyonudur: **anahtar dağıtmak
> yerine kimlik doğrulamak.** Sonuç: diskte hiç kalıcı sır yoktur → çalınacak sır yoktur → sır sızması diye
> bir arıza sınıfı büyük ölçüde ortadan kalkar. "Var olmayan sır sızmaz" — bu fazın en güçlü tek dersidir.

> **🤔 Düşün 9.3** — Bir uygulaman S3'ten dosya okuyacak. İki tasarım: (A) bir IAM kullanıcısının access
> key'ini uygulamanın config dosyasına yazmak; (B) instance'a bir IAM role eklemek ve uygulamanın
> anahtarsız erişmesi. (a) (A)'da access key sızarsa (ör. config git'e düştü) ne olur, ve zararı sınırlamak
> için ne yapman gerekir? (b) (B) bu risk sınıfını neden büyük ölçüde ortadan kaldırır — diske ne yazılıyor?
>
> *(Cevap: fazın sonunda)*

---
---

# 9.7 Bu Faz Bozulunca — Güvenlik/Hardening Arıza İmzaları

Bu fazın arızaları özeldir: çoğu "bir şey **fazla** açık" (güvenlik açığı) veya "bir şey **fazla** kapalı"
(sertleştirme bir işlevi kırdı) biçimindedir. Tablo belirtiyi doğru katmana götürür:

| Belirti | Muhtemel sebep | Bakılacak yer | İlgili bölüm |
|---|---|---|---|
| İzinler doğru ama "Permission denied" | MAC (AppArmor/SELinux) engelliyor | `aa-status`, `journalctl -k \| grep apparmor`, `ausearch -m avc` | 9.3.2 |
| Servis 80'e bind edemiyor, root vermek istemiyorum | Ayrıcalıklı port yetkisi eksik | `setcap cap_net_bind_service` / `AmbientCapabilities` | 9.5.1 |
| Sistem yavaşladı/dondu, disk hızla doluyor | auditd log patlaması → journald/disk | `journalctl --disk-usage`, `df -h /var/log`, audit kuralları | 9.4.2 |
| Servis ele geçirildi, tüm makine düştü | Servis root çalışıyordu (patlama yarıçapı max) | Unit'te `User=`, capabilities | 9.1.1 |
| Bir sır sızdı (git'e düştü / logda) | Sır diske/koda gömülü | Sırrı döndür, Secrets Manager/IAM role'e taşı | 9.6.1 |
| SSH sertleştirmesi sonrası kilitlendim | Yanlış `sshd_config` / firewall | Seri/bulut konsolu ile gir, geri al (Faz 7) | 9.2.1 |
| Otomatik yükseltme bir uygulamayı bozdu | Güvenlik yaması sürümü değiştirdi | Pinning (8.1.2), immutable imaj (8.4) | 9.2.2 |

> **Bu tablodan çıkan ders:** Bu fazın iki büyük fikri her satırın altında yatar. Birincisi **katman
> ayrımı**: "izinler doğru ama engelli" arızası, DAC (Faz 2) ile MAC'ı (9.3) karıştırmaktan doğar — bir
> engelle karşılaşınca hep sor "bu hangi katman: izin mi, MAC mı, firewall mı, capability mi?" İkincisi
> **patlama yarıçapı**: güvenlik ele geçirilmeyi tümüyle önlemek değil, ele geçirilme olduğunda hasarı
> sınırlamaktır — ve bunun tek en güçlü aracı en az yetkidir (non-root servis, ince capabilities, diskte
> sıfır sır). Ve unutma: sertleştirmenin kendisi 🔴'dır — her sıkılaştırma adımı bir şeyi kırabilir veya
> seni kilitleyebilir, bu yüzden Faz 7'nin altın kuralı (ikinci oturum, geri alma yolu) burada da geçerli.

---
---

# Faz 9 — Düşün sorularının cevapları

## Cevap 9.1 — Açık ikisinde de kullanılır; fark patlama yarıçapındadır

(a) Evet — açık kodun içinde olduğu için saldırgan **her iki** sunucuda da onu kullanıp uzaktan kod
çalıştırabilir. Katmanlı savunma açığın **kullanılmasını** her zaman önlemez; asıl işi, kullanıldıktan
sonra hasarı sınırlamaktır. (b) Ele geçirdikten sonra patlama yarıçapı çok farklıdır: A'da uygulama
`User=root` çalıştığı için saldırgan **anında root** olur — tüm dosyaları okur/değiştirir, başka servisleri
öldürür, kalıcılık sağlar; tek savunma olan host firewall zaten **içeri girmiş** bir saldırgana karşı işe
yaramaz. B'de ise saldırgan yalnızca `appuser` yetkisiyle başlar (9.1: sistem dosyalarını değiştiremez,
başka kullanıcıların verisini okuyamaz), AppArmor profili (9.3) o process'in erişebileceği dosya/işlemleri
zaten kısıtlar, ve SG dışa doğru bağlantıları da sınırlayabilir (veri sızdırmayı zorlaştırır). Aynı açık,
B'de çok daha küçük bir hasarla sonuçlanır — katmanlı savunmanın tüm amacı budur.
**İlgili bölüm:** 9.1.1-9.1.2 · **Devamı:** 9.7 arıza tablosu (satır 4).

## Cevap 9.2 — Engel MAC katmanındadır; `disable` değil, profili düzelt

(a) DAC (`ls -ld` izinleri doğru, hatta `sudo -u www-data cat` çalışıyor) sorunsuz olduğuna göre engel bir
**üst katmandadır** — büyük olasılıkla **MAC** (Ubuntu'da AppArmor). Web sunucusunun AppArmor profili,
`/var/www/data/` yolunu okumasına izin vermiyor. (b) Doğrulama: `sudo aa-status` (profil enforce mi) ve
`sudo journalctl -k | grep -i apparmor | tail` — engellenen erişim "DENIED" satırı olarak tam bu yolu
gösterir (SELinux'ta `sudo ausearch -m avc -ts recent`). (c) Doğru çözüm profili tamamen kapatmak
(`aa-disable`) **değildir**, çünkü o zaman o servisin ikinci kilidini tümüyle kaybedersin — ele geçirilirse
hiçbir MAC sınırı kalmaz. Doğru çözüm profili **düzeltmektir**: `/var/www/data/` yolunu profile izinli
olarak eklemek (gerekirse önce `complain` moduyla neyin gerektiğini görmek). En az yetki: sadece gereken
yolu aç, profili değil.
**İlgili bölüm:** 9.3.1-9.3.2 · **Devamı:** 9.7 arıza tablosu (satır 1).

## Cevap 9.3 — Sızan key döndürülmeli; IAM role diske hiç key yazmaz

(a) (A)'da access key config dosyasına düz metin yazıldığı için, config git'e düştüğü an anahtar **sızmış**
sayılır — geçmişten silsen bile git geçmişinde ve her klonda kalır. Bir kez sızmış sır artık güvenilmezdir;
tek doğru tepki anahtarı **hemen döndürmektir** (eskisini iptal et, yenisini üret) ve o key'in eriştiği
kaynaklarda kötüye kullanım olup olmadığını denetlemektir. Silmek yetmez, döndürmek şarttır. (b) (B) bu
risk sınıfını büyük ölçüde ortadan kaldırır çünkü **diske hiçbir kalıcı key yazılmaz**: instance'a bir IAM
role eklenir, uygulama kimlik bilgilerini çalışma anında instance metadata servisinden alır, ve bu bilgiler
**geçici ve otomatik dönerdir**. Sızacak kalıcı bir sır olmadığı için "config git'e düştü → key sızdı"
arıza sınıfı ortadan kalkar. "Var olmayan sır sızmaz" (9.6.2).
**İlgili bölüm:** 9.6.1-9.6.2 · **Devamı:** 9.7 arıza tablosu (satır 5).

---
---

# Faz 9 — Sık sorulan sorular

**S1 — En az yetki ilkesi tek cümlede nedir?** Her kullanıcı/servis/process yalnızca işini yapmak için
gereken en az yetkiye sahip olmalı — bir fazla değil. Amaç ele geçirilmeyi önlemek değil, olduğunda patlama
yarıçapını daraltmaktır (9.1.1).

**S2 — DAC ile MAC farkı nedir?** DAC (Faz 2: `rwx`, sahip/grup/diğer) izinleri dosya sahibi belirler ve
root aşabilir. MAC (AppArmor/SELinux) üstte, merkezi bir politikayla zorlanır ve root bile gevşetemez —
DAC'ın üstünde ikinci bir kilit (9.3.1).

**S3 — "İzinler doğru ama Permission denied" — nereye bakarım?** MAC katmanına: `aa-status` +
`journalctl -k | grep apparmor` (AppArmor), veya `getenforce` + `ausearch -m avc` (SELinux). Engel DAC'ta
değil, profildedir (9.3.2).

**S4 — Bir servisi 80 portunda çalıştırmak için root vermek zorunda mıyım?** Hayır. Sadece
`CAP_NET_BIND_SERVICE` capability'sini ver (`setcap` veya systemd `AmbientCapabilities`) — ayrıcalıklı porta
bind eder ama root'un geri kalan gücüne sahip olmaz (9.5.1).

**S5 — Sırları nereye koymalıyım?** Koda/diske düz metin ASLA. Secrets Manager / SSM Parameter Store gibi
bir kasaya, ya da en iyisi bir IAM role ile hiç sır saklamadan kimlikle eriş (9.6).

**S6 — Bir API key yanlışlıkla git'e düştü, sildim, yeter mi?** Hayır — git geçmişinde ve klonlarda kalır.
Tek doğru tepki key'i **döndürmektir** (iptal + yeni). Silmek sızmayı geri almaz (9.6.1).

**S7 — auditd'yi açtım, sistem yavaşladı, neden?** Muhtemelen çok geniş kurallarla log patlaması yaşıyorsun:
auditd binlerce olay üretiyor, journald/disk boğuluyor. Kuralları daralt, rate-limit + rotation uygula
(9.4.2).

---
---

# Faz 9 — Kendini sına

Cevaplarını bir kâğıda yaz, sonra cevap anahtarıyla karşılaştır. Hedef: 18 sorudan 14+.

## Bölüm A — Tanım ve mekanizma

1. En az yetki ilkesini tanımla. "Patlama yarıçapı" ne demektir ve en az yetki onu nasıl etkiler?
2. Katmanlı savunma (defense in depth) nedir? Bir katmanın aşılması neden tam ele geçirilme değildir?
3. DAC ile MAC arasındaki temel farkı açıkla: hangisini root aşabilir, hangisini aşamaz?
4. AppArmor ve SELinux ne yapar? "Profil" kavramı neyi tanımlar?
5. Linux capabilities root/non-root ikiliğini nasıl kırar? `CAP_NET_BIND_SERVICE` örneğini ver.
6. Sırları koda/diske gömmek neden felakettir? Bir kez sızan sır için tek doğru tepki nedir?
7. IAM instance role, diske sır yazmayı nasıl ortadan kaldırır?
8. auditd log patlaması senaryosunu anlat: denetim aracı sistemi nasıl çökertebilir?

## Bölüm B — Uygula ve teşhis et

9. Bir servisin dosya erişimi `ls -l` izinleri kusursuz olmasına rağmen "Permission denied" veriyor. Hangi
   katmana bakarsın ve hangi komutla doğrularsın?
10. Web sunucun 80 portuna bind edemiyor ama tüm servisi root çalıştırmak istemiyorsun. Çözüm nedir?
11. Bir uygulama ele geçirildi ve saldırgan anında tüm makineyi kontrol etti. Unit dosyasındaki hangi ayar
    bunu önleyebilirdi?
12. `df -h /var/log` diskin dolduğunu, `journalctl --disk-usage` devasa bir boyut gösteriyor. Muhtemel
    güvenlik-aracı sebebi nedir ve iki düzeltme adımı?
13. Bir access key'in config dosyasında olduğunu ve dosyanın git'e commit edildiğini fark ettin. İlk iki
    adımın ne?
14. Dışa açık saldırı yüzeyini denetlemek için hangi üç komutu çalıştırırsın (portlar, servisler, firewall)?

## Bölüm C — Muhakeme ve bağlantı

15. Faz 8'de `User=appuser` yazmıştın. Bunun Faz 9'daki hangi ilkenin bir uygulaması olduğunu ve neden bir
    güvenlik kararı olduğunu açıkla.
16. "İzinler doğru ama engelli" arızası, Faz 2'nin izin modeliyle Faz 9'un MAC'ını nasıl birbirine bağlar?
    İki katmanın ilişkisini bir cümlede yaz.
17. IAM role fikri, Faz 7.3'teki hangi fikrin (SSH/SSM) veri erişimi versiyonudur? Ortak prensibi yaz.
18. Bir CIS-hardened AMI, Faz 8'in immutable felsefesiyle nasıl birleşir? "Güvenlik imajın bir özelliğidir"
    ne demek?

---

## Cevap anahtarı

1. Her kullanıcı/servis sadece gereken en az yetkiye sahip olmalı; patlama yarıçapı = bir şey ele
   geçirilince hasarın yayılım büyüklüğü; en az yetki onu küçültür (9.1.1). — 2. Hedefe ulaşmak için birden
   çok bağımsız katmanı aşmak gerekir; katmanlar bağımsız olduğu için biri aşılsa da diğerleri hâlâ korur
   (9.1.2). — 3. DAC izinlerini sahip belirler, root aşabilir; MAC merkezi politikayla zorlanır, root bile
   aşamaz — DAC'ın üstünde ikinci kilit (9.3.1). — 4. Her programa "sadece şu dosya/işlem/yeteneklere
   erişebilirsin" diyen bir profil atar; profilin izin vermediğini DAC açık olsa bile yapamaz (9.3.1). — 5.
   root'un gücünü ~40 ince yetkiye böler, tek tek verilir; `CAP_NET_BIND_SERVICE` = "sadece ayrıcalıklı
   porta bind et, başka root gücü yok" (9.5.1). — 6. Sır git geçmişine/kopyalara/loglara sızar ve geri
   alınamaz; tek doğru tepki döndürmektir (rotate) (9.6.1). — 7. Uygulamaya parola yerine makineye kimlik
   (role) verir; kimlik bilgileri metadata'dan geçici/otomatik-döner gelir, diske key yazılmaz (9.6.2). — 8.
   Çok geniş kurallar → saniyede binlerce olay → journald/disk boğulur → disk dolar → sistem yavaşlar/donar;
   güvenlik aracı DoS'a döner (9.4.2).

9. MAC katmanına; `sudo aa-status` + `sudo journalctl -k | grep -i apparmor` (veya `ausearch -m avc`)
   (9.3.2). — 10. Tüm servise root değil, sadece `CAP_NET_BIND_SERVICE` capability'si ver (`setcap` /
   `AmbientCapabilities`) (9.5.1). — 11. `User=` ile non-root çalıştırmak (ve gerekiyorsa ince
   capabilities); root yerine sınırlı kullanıcı patlama yarıçapını küçültürdü (9.1.1). — 12. auditd log
   patlaması; (i) audit kurallarını daralt, (ii) rate-limit + log rotation uygula (9.4.2). — 13. (i) Key'i
   hemen **döndür** (iptal + yeni üret); (ii) sırrı Secrets Manager/SSM'e veya IAM role'e taşı, koddan
   çıkar; (git geçmişini temizlemek ikincil — key zaten sızmış say) (9.6.1). — 14. `sudo ss -tulpn`
   (portlar), `systemctl list-units --type=service --state=running` (servisler), `sudo ufw status verbose`
   (firewall) (9.2.1).

15. En az yetki ilkesinin (9.1) uygulamasıdır: servisi root yerine sınırlı bir kullanıcıyla çalıştırmak,
    ele geçirilme hâlinde patlama yarıçapını küçültür — yani `User=appuser` bilinçli bir hardening kararıdır
    (9.1.1, Faz 8.2.2). — 16. DAC (Faz 2) ve MAC (9.3) iki bağımsız, üst üste katmandır; bir erişim için
    **ikisinin de** izin vermesi gerekir — DAC geçse bile MAC reddedebilir, "izin doğru ama engelli"
    tam budur (9.3.2). — 17. Faz 7.3'teki "SSH anahtarı dağıtma, SSM/kimlik kullan" fikrinin veri erişimi
    versiyonudur; ortak prensip: **anahtar dağıtmak yerine kimlik doğrulamak** — kalıcı sır yok (9.6.2).
    — 18. Sertleştirmeyi (profiller, kapalı servisler, sıkı SSH) bir kez imaja "pişirir"; immutable felsefeyle
    her makine güvenli doğar, elle sertleştirme gerekmez → "güvenlik bir kurulum adımı değil, imajın bir
    özelliğidir" (9.3.2, Faz 8.4.1).

## Puanlama

| Doğru sayısı | Ne anlama geliyor |
|---|---|
| 16-18 | Katmanlı savunmayı ve en az yetkiyi kavradın. Ara Sınav 4'e (Faz 7-9) hazırsın. |
| 13-15 | İyi. Kaçırdığın soruların bölümlerini tekrar oku (özellikle 9.3 ve 9.6). |
| 9-12 | Temel var ama kırılgan. DAC vs MAC ve sır yönetimi üzerine çalış. |
| 0-8 | Fazı yeniden gez; bir test makinesinde `aa-status`, `getcap`, `ss -tulpn` çıktılarını bizzat oku. |

Kaçırdığın soru → dönmen gereken bölüm:

| Soru | Bölüm |
|---|---|
| 1, 11, 15 | 9.1 En az yetki |
| 2 | 9.1.2 Katmanlı savunma |
| 3, 4, 9, 16 | 9.3 MAC |
| 5, 10 | 9.5 Capabilities |
| 6, 7, 13, 17 | 9.6 Sırların yönetimi |
| 8, 12 | 9.4 Denetim / log patlaması |
| 14 | 9.2 Yüzey daraltma |
| 18 | 9.3.2 + Faz 8 immutable |

---
---

# Faz 9 — Kapanış ve Ara Sınav 4'e Köprü

## Bu fazdan ne taşıyorsun

Faz 9 sana güvenliği bir **disiplin** olarak verdi: tek bir ayar değil, katmanlı bir savunma. İki kalıcı
fikir edindin: **en az yetki** (sadece gerekeni ver — non-root servis, ince capabilities, diskte sıfır sır)
ve **patlama yarıçapı** (güvenlik ele geçirilmeyi değil, hasarı sınırlar). Faz 2'nin DAC izinlerinin üstüne
MAC'ı (AppArmor/SELinux) koydun ve "izinler doğru ama engelli" arızasını doğru katmana bağlamayı öğrendin.
Sırları diskten tamamen çıkaran IAM role fikrini gördün — Faz 7.3'teki "anahtar dağıtma, kimlik doğrula"
prensibinin veri versiyonu. Ve tüm bu katmanların (ağ → erişim → yetki → MAC → sırlar) tek bir savunma
mimarisi olduğunu Şekil 9.1'de topladın.

## Ara Sınav 4 (Faz 7-9) bunun neresine bağlanıyor

Faz 7-8-9 birlikte bir bütün oluşturur: sunucunun **dış dünyayla ilişkisi ve savunması**. Faz 7 servisi
ağa açtı (erişilebilirlik), Faz 8 servisi makineye getirdi ve yönetilebilir kıldı (kurulum + servisleştirme),
Faz 9 tüm bunları bilinçli olarak sınırladı (sertleştirme). Ara Sınav 4 bu üç fazı **kesiştirir**: örneğin
"bir servise bağlanamıyorum" (Faz 7) arızasının aslında bir güvenlik grubu/AppArmor kararı (Faz 9) olması;
ya da `nohup` yerine unit yazmanın (Faz 8) hem operasyonel hem güvenlik (`User=`, Faz 9) bir karar olması.
Sınav, bu üç fazın tek bir olayın farklı yüzleri olduğunu görüp göremediğini ölçer.

> **🤔 Faz çıktısı — kendine sor:** Bu üç fazda (7-8-9) aynı komut, `ss -tulpn`, üç farklı gözle karşına
> çıktı: Faz 7'de "servis doğru adreste dinliyor mu" (erişilebilirlik), Faz 8'de "servisim ayakta mı"
> (operasyon), Faz 9'da "gereksiz kaç port dışa açık" (saldırı yüzeyi). Ara Sınav 4'e girmeden düşün: tek
> bir `ss -tulpn` çıktısına bakan bir cloud engineer, bu üç soruyu **aynı anda** nasıl sorar? Bu, üç fazın
> tek bir refleks hâline gelmesidir.
>
> **🧪 Lab 9 fikri (kendi test instance'ında):** (1) `sudo aa-status` ile yüklü AppArmor profillerini gör.
> (2) `sudo ss -tulpn` ile dışa açık portları denetle — her `0.0.0.0` satırı için "gerçekten gerekli mi"
> diye sor. (3) `sudo systemctl status auditd` ile denetim servisini kontrol et; `journalctl --disk-usage`
> ile log yükünü gör. (4) `getcap /usr/bin/ping` ile bir capability örneğini incele. (5) Bir test
> servisini önce `User=root`, sonra `User=nobody` ile çalıştırıp erişebildiği dosyalardaki farkı gör —
> patlama yarıçapını bizzat daralt. Bu adımlar bu fazın katmanlı savunma refleksini elinde toplar.

---

> **Navigasyon:** [◀ Faz 8 — Paketler ve Servisleştirme](Faz_8_Paketler_ve_Servislestirme.md) · **Faz 9** · [Ara Sınav 4 ▶](Ara_Sinav_4.md)

