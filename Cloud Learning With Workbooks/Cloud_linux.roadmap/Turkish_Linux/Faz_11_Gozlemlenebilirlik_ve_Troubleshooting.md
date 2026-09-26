# Faz 11 — Gözlemlenebilirlik ve Troubleshooting: Bir Şey Bozulunca

> **Navigasyon:** [◀ Faz 10 — Otomasyon ve Scripting](Faz_10_Otomasyon_ve_Scripting.md) · **Faz 11** · [Faz 12 — Cloud'a Köprü ▶](Faz_12_Cloud_a_Kopru.md)

---

## Nereden geliyoruz

Faz 0'dan Faz 10'a kadar her fazın sonunda bir **"Bu faz bozulunca"** tablosu vardı: Faz 2'de izin hatası,
Faz 3'te zombie/OOM, Faz 6'da mount kayması, Faz 7'de erişilemezlik, Faz 8'de kurulu-ama-çalışmıyor, Faz 9'da
"izinler doğru ama engelli", Faz 10'da sessizce yanlış iş yapan script. Her biri ayrı ayrı bir belirti-neden
eşlemesiydi. Faz 11 bunların **hepsini tek bir metodolojide** toplar. Çünkü gerçek dünyada bir şey bozulduğunda
sana "bu bir Faz 6 sorunudur" diye söyleyen kimse yoktur — sadece bir belirti vardır ("uygulama açılmıyor") ve
sen kanıtı katman katman bulmak zorundasın.

Bu faz özel olarak üç fazın üzerine oturur:

- **Faz 4 — kaynaklar.** `top`/`free`/`iostat` ile CPU/RAM/IO'yu okumayı öğrenmiştin. Faz 11 bunları bir
  **teşhis refleksinin** parçası yapar: "sistem yavaş" dendiğinde önce hangi kaynağa bakarsın.
- **Faz 5 — loglar.** journald ve `journalctl`'i görmüştün. Faz 11 logu **ilk kanıt kaynağı** yapar: bir servis
  neden başlamadı sorusunun cevabı neredeyse her zaman logdadır.
- **Faz 10 — sessiz başarısızlık.** Bir script'in hata verip devam edebileceğini gördün. Faz 11 bu sessiz
  hatanın kanıtını sistematik olarak nasıl bulacağını öğretir.

## Bu fazın sorusu

Faz 10'a kadar "sistemi nasıl kurar, çalıştırır, otomatikleştiririm" dedik. Faz 11 tam tersini sorar:

> *"Bir şey bozulduğunda — uygulama açılmıyor, servis başlamıyor, sistem yavaş — panik yapmadan, tahminle
> değil, kanıtla, katman katman (uygulama logu → servis durumu → kaynak → ağ → çekirdek) nasıl daralttarak
> gerçek nedeni bulurum; ve bunu bulmak için hangi araca ne zaman uzanırım?"*

Bu fazın merkezinde iki fikir var: **sistematik daralma** (rastgele denemek yerine katman katman elemek) ve
**üç içgüdü sorusu** ("Sistem şu an ne yapıyor?", "Neden erişilemiyor?", "Boot'tan servise nerede kırıldı?").
Bir bulut mühendisinin en değerli becerisi bir aracı ezbere bilmek değil, **kanıtı doğru sırada aramaktır**.

Bu fazın sonunda: bir sorunu katman katman daraltabilecek; her araca (top, ps, ss, lsof, strace, dmesg,
journalctl, iostat/vmstat) ne zaman uzanacağını bilecek; ve "instance sağlıklı ama uygulama değil" ayrımını
cloud'da kanıtla gösterebileceksin.

---

## Bu fazın sonunda

- Bir sorunu **katman katman daraltma** metodolojisini uygulayabileceksin (uygulama logu → servis durumu →
  kaynak → ağ → çekirdek)
- **Log ustalığı** kazanacaksın: `journalctl` (birim/zaman/boot filtreleri), `/var/log`, `logrotate`;
  auth.log/syslog/dmesg'in ne söylediği
- Her **teşhis aracına** ne zaman uzanacağını bileceksin: `top`/`htop`, `ps`, `ss`, `lsof`, `strace` (nihai
  hakem), `dmesg`, `iostat`/`vmstat`/`free`, `/proc/<pid>/`
- **Üç içgüdü sorusunu** bir karar ağacına çevirebileceksin ("neden erişilemiyor" → izin/ownership/mount/MAC/
  capability)
- **Cloud:** "instance sağlıklı ama uygulama değil" ayrımını yapabilecek; CloudWatch Logs + SSM ile kanıtı
  uzaktan toplayabileceksin

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden burada |
|---|---|---|---|
| 11.1 | Loglar — ilk kanıt kaynağı | `[uygulama]` | journalctl, /var/log, logrotate; auth/syslog/dmesg ne der; log'u kesmenin dört tarifi |
| 11.2 | Katman katman debugging metodolojisi | `[kavram]` | Panik değil, sistematik daralma: log→servis→kaynak→ağ→çekirdek; adım adım işlenmiş bir 502 olayı |
| 11.3 | Araç ustalığı | `[uygulama]` | Hangi araca ne zaman: top/ps/ss/lsof/strace/dmesg/iostat; vmstat/iostat/ss ve `/proc/<pid>` nasıl *okunur* |
| 11.4 | Üç içgüdü sorusu | `[kavram]` | "Ne yapıyor", "neden erişilemiyor", "boot'tan servise" |
| 11.5 | Cloud'da kanıt toplama | `[uygulama]` | CloudWatch Logs + SSM; "instance sağlıklı ama uygulama değil" |
| 11.6 | Bu faz bozulunca | — | Yanlış teşhis refleksinin imzaları |

> **Bu faz nasıl çalışılır:** Bu fazın komutlarının neredeyse tamamı 🟢 **salt-okunur**tur — `top`, `ps`, `ss`,
> `journalctl`, `dmesg`, `strace` sistemi gözlemler, değiştirmez. Bu, bu fazı en güvenli fazlardan biri yapar:
> gönül rahatlığıyla dene, hiçbir şeyi bozmazsın. Bu fazın en öğretici alışkanlığı şudur: bir sorunu çözerken
> **her adımda bir sonraki adıma geçmeden önce eldeki kanıtı yaz** — "log ne dedi", "servis durumu ne", "port
> dinleniyor mu". Rastgele komut denemek yerine, her komutu bir hipotezi test etmek için çalıştır. En değerli
> deney: kasıtlı olarak bir servisi boz (yanlış port'a bind et, config'de hata yap) ve kanıtı katman katman
> bul — logdan başlayıp gerçek nedene ulaşana kadar.

---
---

# 11.1 Loglar — İlk Kanıt Kaynağı

## 11.1.1 `journalctl`: modern sistemin hafızası `[uygulama]`

Faz 5'te journald'ı (systemd'nin merkezî log toplayıcısı) görmüştün. Troubleshooting'de o senin **ilk durağın**:
bir servis başlamadıysa, bir uygulama çöktüyse, kanıt neredeyse her zaman journal'dadır. Kritik filtreler:

```bash
journalctl -u nginx.service        # tek bir birimin logları (Faz 8!)
journalctl -u nginx -f             # canlı takip (tail -f gibi)
journalctl -u nginx --since "10 min ago"   # zaman filtresi
journalctl -b                      # bu boot'un logları
journalctl -b -1                   # bir önceki boot (çökme sonrası kritik)
journalctl -p err                  # sadece hata seviyesi ve üstü
journalctl -k                      # çekirdek mesajları (dmesg eşdeğeri)
```

Neden ilk durak? Çünkü bir servis "başlamadı" dediğinde `systemctl status` sana sadece son birkaç satırı
gösterir; asıl neden (config parse hatası, port çakışması, izin reddi) genellikle `journalctl -u <servis> -e`
ile görünür. Refleks: bir şey başlamadıysa **önce logu oku, sonra tahmin et**.

## 11.1.2 `/var/log` ve logrotate: geleneksel loglar ve boyut kontrolü `[uygulama]`

journald her şeyi topladıysa da, birçok geleneksel servis hâlâ `/var/log` altına yazar:

- **`/var/log/syslog`** (Debian/Ubuntu) veya **`/var/log/messages`** (RHEL) — genel sistem logları
- **`/var/log/auth.log`** — kimlik doğrulama: SSH girişleri, `sudo` kullanımı, başarısız girişler (Faz 9
  güvenlik forensiği burada başlar)
- **`dmesg`** / **`/var/log/kern.log`** — çekirdek halkası tamponu: donanım, sürücü, OOM killer (Faz 4!), disk
  hataları

Loglar sonsuza kadar büyürse diski doldurur (Faz 6'da gördüğün "disk %100 → sistem donuyor" senaryosu).
**`logrotate`** bu yüzden var: logları periyodik olarak döndürür, sıkıştırır, eskiyeni siler. `/etc/logrotate.d/`
altındaki config'ler her servisin log dosyasının ne zaman döndürüleceğini tanımlar.

> **🔧 Makinende gör** 🟢 — auth.log'da başarısız SSH girişlerini oku
>
> ```
> $ sudo journalctl -u ssh --since today | grep -i "failed\|invalid"
> $ sudo grep -i "failed password" /var/log/auth.log | tail -20    # Ubuntu
> ```
>
> Bir sunucuya kimin girmeye çalıştığı, `sudo`'yu kimin çalıştırdığı hep buradadır. Faz 9'un "kanıt" fikri
> burada somutlaşır: bir güvenlik olayından sonra ilk bakılan yer auth.log'dur. Kendi makinende birkaç kez
> yanlış parolayla `ssh localhost` dene ve girişlerin loga nasıl düştüğünü gör.

> **⚠️ Yaygın yanılgı: "Log yoksa sorun da yoktur."**
>
> Log**suzluk** bir kanıttır — ama sorunun yokluğunun değil, çoğu zaman sorunun **daha derinde** olduğunun
> kanıtı. Bir uygulama hiç log yazmadıysa üç olasılık var: (1) hiç başlamadı (servis birimi hatalı → journalctl
> -u), (2) başladı ama logu başka yere yazıyor (stdout yerine bir dosyaya → config'i kontrol et), (3) diskin
> logrotate'i loğu sildi ya da disk dolu olduğu için yazamıyor (`df -h` → Faz 6). "Log yok" bir cevap değil,
> yeni bir sorudur: neden yok?


## 11.1.3 Log'u iyi kesmek: dört tarif `[uygulama]`

Bayrakları bilmek, bir log'u **kesmeyi** bilmekle aynı şey değildir. Dört tarif olayların çoğunu karşılar:

| Durum | Tarif | Neden işe yarar |
|---|---|---|
| "Ölmeden hemen önce ne dedi?" | `journalctl -u myapp -e` | `-e` **sona** atlar; son satırdan yukarı doğru okursun |
| "03:10 civarı bozuldu" | `journalctl --since "03:00" --until "03:20"` | 20 dakikalık pencerede **tüm birimler** — birimler arası nedenler görünür (disk doldu, *sonra* uygulama çöktü) |
| "Sadece önemli olanlar" | `journalctl -u myapp -p warning -b` | Uyarı ve daha kötüsü, yalnızca bu boot |
| "Pid'e ya da birime göre tam filtre lazım" | `journalctl -u myapp -o json-pretty -n 1` | Journal'ın sakladığı her alanı gösterir (`_PID`, `_SYSTEMD_UNIT`, `PRIORITY`); neye göre filtreleyebileceğini görürsün |

İki alışkanlık bu tarifleri değerli kılar. Birincisi: **daralmadan önce genişlet** — neden çoğu zaman *başka* bir
birimdedir (dolu disk, ölü veritabanı, yeniden başlayan ağ); önce zaman penceresiyle başla, `-u`'yu sonra ekle.
İkincisi: `-b -1`'e güvenmeden önce **journal'ın reboot'tan sağ çıktığını kontrol et**: "no entries" ya da "no
persistent journal" cevabı geliyorsa journal *uçucudur* (yalnızca `/run`'da tutulur, her reboot'ta kaybolur).
Kalıcı yapmak için `sudo mkdir -p /var/log/journal && sudo systemctl restart systemd-journald` ya da
`/etc/systemd/journald.conf` içinde `Storage=persistent` — yine "çalışan durum vs kalıcı tanım" ikilisi, bu kez
kanıtın kendisi için.

> **🤔 Düşün 11.1** — Bir servis başlangıçta çöküyor ve `systemctl status myapp` yalnızca "failed" ile işe yaramaz iki satır gösteriyor. (a) Sonra ne çalıştırırsın ve neden? (b) Servis çöktü ve makine gece yeniden başladı — reboot öncesi log'a hangi filtre ulaştırır? (c) Hiç log bulamıyorsun: üç olasılık nedir?
>
> *(Cevap: fazın sonunda)*
---
---

# 11.2 Katman Katman Debugging Metodolojisi

## 11.2.1 Panik değil, sistematik daralma `[kavram]`

Yeni başlayanların en büyük hatası, bir şey bozulunca **rastgele komut denemektir**: "belki yeniden başlatırım",
"belki izinleri düzeltirim". Bu, gerçek nedeni bulmadan belirtiyi maskeleyebilir ya da işleri kötüleştirir. Usta
mühendis bunun yerine **katman katman daraltır** — her katmanda "sorun burada mı?" diye sorar ve kanıtla eler:

1. **Uygulama logu** — Uygulama ne diyor? (journalctl -u, /var/log, uygulamanın kendi logu) — çoğu zaman cevap
   burada.
2. **Servis durumu** — Servis çalışıyor mu, hangi durumda? (`systemctl status`, `is-active`, `is-failed` — Faz
   8) Enabled ama başlamamış mı, sürekli restart mı ediyor?
3. **Kaynak** — CPU/RAM/IO tükendi mi? (`top`, `free`, `iostat` — Faz 4) OOM killer devreye girdi mi (`dmesg`)?
4. **Ağ** — Port dinleniyor mu, DNS çözülüyor mu, ulaşılabilir mi? (`ss -tulpn`, `curl`, `dig` — Faz 7) Faz 7'nin
   üç merceği: dinliyor mu / güvenlik duvarı / bind adresi.
5. **Çekirdek** — Donanım/sürücü/disk hatası var mı? (`dmesg`, `journalctl -k`) En alt katman.

Bu sıra rastgele değil: **en olası ve en ucuz kontrolden en derin ve en nadire** doğru gider. Çoğu sorun ilk iki
katmanda çözülür; çekirdeğe kadar inmek nadirdir. Metodolojinin özü: her katmanda kanıt topla, bir sonrakine
ancak eldeki katmanı elediğinde geç.

![Şekil 11.1 — Katman katman debugging metodolojisi. "Uygulama açılmıyor" belirtisinden başlayan, beş katmandan geçen bir karar akışı: (1) Uygulama logu — journalctl -u / /var/log; (2) Servis durumu — systemctl status; (3) Kaynak — top / free / iostat / dmesg (OOM); (4) Ağ — ss -tulpn / curl / dig; (5) Çekirdek — dmesg / journalctl -k. Her katmanda "kanıt burada mı?" sorusu sorulur; kanıt bulunduysa neden tespit edilir, bulunmadıysa bir alt katmana inilir. Yanda "en ucuz ve en olası kontrolden en derin ve en nadire" yönünü gösteren bir ok. Altta ders: rastgele deneme değil, katman katman kanıtla daraltma.](../diagrams/png/lx-11-01-debugging-layers.png)

## 11.2.2 Adım adım bir olay: "502 Bad Gateway" tepeden tabana `[uygulama]`

Metodolojiye güvenmenin en kolay yolu onu iş başında görmektir. Kurulum: nginx, `127.0.0.1:8000`'deki bir
uygulamanın önünde duruyor; kullanıcılar **502 Bad Gateway** alıyor. Başka hiçbir şey bilinmiyor. Katmanlardan
in ve **ilk kanıt üreten katmanda dur**:

| Katman | Komut | Ne görürsün | Hüküm |
|---|---|---|---|
| 1. Log | `sudo tail -5 /var/log/nginx/error.log` | `connect() failed (111: Connection refused) while connecting to upstream ... "http://127.0.0.1:8000/"` | nginx **sağlam**; 8000'de kimse cevap vermiyor. Soru "neden 502?" iken "uygulama neden dinlemiyor?" olur |
| 2. Servis | `systemctl status myapp` | `Active: activating (auto-restart)` … `Main process exited, code=killed, status=9/KILL` | Uygulama **restart döngüsünde** ve kendi kendine çıkmıyor, **öldürülüyor** (sinyal 9) |
| 3. Kaynak | `sudo dmesg -T \| grep -i "out of memory"` | `Out of memory: Killed process 2114 (gunicorn) ...` ve `free -h` toplam 1 GiB, swap 0 gösteriyor | Katil çekirdeğin **OOM killer**'ı (Faz 4) |

4. ve 5. katmana hiç ihtiyaç duymadın. **Metot, kanıt bulununca durmayı söyler**, "eksik kalmasın diye her
katmanı gez" demez. Şimdi çözüm belirtiye değil *nedene* yapılır:

- `systemctl restart nginx` **değil** — nginx suçsuzdu ve restart gerçek sorunu birkaç dakika daha saklardı.
- Bellek talebini azalt (ör. daha az uygulama worker'ı), bellek ekle (daha büyük instance) ve tek bir process'in
  makineyi ele geçirememesi için servise tavan koy (unit'te `MemoryMax=` — Faz 8).
- Kanıtı kalıcı yap: uygulama log'unu **ve** OOM satırını CloudWatch'a gönder (11.5); bir dahaki sefere 3. katman
  bir login yerine tek bir sorgu olur.

Üç katmanın üç ayrı "dilde" konuştuğuna dikkat et — web sunucusu log'u, systemd durumu, çekirdek mesajı — ve her
birinin bir sonrakine soruyu nasıl *daralttığına*. Birinci katman `Permission denied` deseydi Faz 2'ye,
`Connection timed out` deseydi Faz 7'nin üç merceğine giderdin. **Sonraki adımı tahminin değil, kanıt seçer.**

> **🤔 Düşün 11.2** — Bir web uygulaması "açılmıyor" (tarayıcıda zaman aşımı). Elindeki tek bilgi bu. (a) Yukarıdaki
> beş katmanı sırayla uygularsan, her katmanda hangi tek komutu çalıştırırsın ve ne ararsın? (b) `systemctl
> status` "active (running)" diyorsa ama tarayıcı hâlâ açılmıyorsa, hangi katmana geçersin ve neden? (c) Neden
> bu sıra (log→servis→kaynak→ağ→çekirdek) "önce yeniden başlat" refleksinden daha iyidir?
>
> *(Cevap: fazın sonunda)*

---
---

# 11.3 Araç Ustalığı — Hangi Araca Ne Zaman

## 11.3.1 Gözlem araçları ve doğru anları `[uygulama]`

Her aracın bir **sorusu** vardır. Ustalık, aracı ezberlemek değil, hangi soruda hangisine uzanacağını
bilmektir:

| Araç | Cevapladığı soru | Ne zaman |
|---|---|---|
| `top` / `htop` | Sistem şu an ne yapıyor? Hangi process CPU/RAM yiyor? | "Sistem yavaş" (Faz 4) |
| `ps aux` | Anlık process listesi, durum (Z/D), parent | Belirli bir process'i ararken (Faz 3) |
| `free -h` | RAM/swap ne durumda? | "Bellek doldu mu" (Faz 4) |
| `iostat` / `vmstat` | Disk IO / genel sistem darboğazı | "Yavaşlık disk mi CPU mu" (Faz 4) |
| `ss -tulpn` | Hangi port'u kim dinliyor? | "Erişilemiyor" — dinleme merceği (Faz 7) |
| `lsof` | Hangi process hangi dosyayı/soketi açık tutuyor? | "Dosya silindi ama yer boşalmadı", "port kimde" |
| `dmesg` / `journalctl -k` | Çekirdek ne diyor? OOM, disk hatası, sürücü | Donanım/OOM şüphesi (Faz 4) |
| `journalctl -u` | Bu servis ne dedi? | Servis başlamadı/çöktü (Faz 5, 8) |
| `strace` | Bu process hangi **sistem çağrısını** yapıyor, nerede takılıyor? | **Nihai hakem** — başka her şey suskunsa |
| `/proc/<pid>/` | Bu process hakkında her şey (fd, limits, status, maps) | Derin inceleme |

## 11.3.2 `strace`: nihai hakem `[mekanizma]`

Diğer tüm araçlar suskun kaldığında — log yok, durum "running", kaynak bol ama uygulama takılıyor — `strace`
devreye girer. `strace` bir process'in yaptığı **her sistem çağrısını** (Faz 1'de gördüğün user space →
kernel space geçişleri) canlı gösterir:

```bash
strace -p 1234                     # çalışan process'e bağlan
strace -f -e trace=openat,connect ./program   # dosya açma ve ağ bağlantılarını izle
```

Neden "nihai hakem"? Çünkü process'in **gerçekte ne yapmaya çalıştığını** yalan söylemeden gösterir: hangi
dosyayı açmaya çalışıp `EACCES` (izin reddi — Faz 2!) aldığını, hangi adrese `connect` edip `ECONNREFUSED`
(Faz 7!) aldığını, hangi çağrıda `EAGAIN` ile takıldığını. "Uygulama config dosyasını bulamıyor" gibi belirsiz
bir sorun, `strace` ile "işte, `/etc/app/config.yml`'i açmaya çalışıp `ENOENT` alıyor — dosya yanlış yerde"
kesinliğine dönüşür. Ağırdır ve process'i yavaşlatır, o yüzden **son çare**dir; ama sustuğun her yerde konuşur.

> **🔧 Makinende gör** 🟢 — strace ile bir komutun neye dokunduğunu izle
>
> ```
> $ strace -e trace=openat cat /etc/hostname 2>&1 | grep -i hostname
> openat(AT_FDCWD, "/etc/hostname", O_RDONLY) = 3
> $ strace -e trace=openat cat /yok/dosya 2>&1 | tail -3
> openat(AT_FDCWD, "/yok/dosya", O_RDONLY) = -1 ENOENT (No such file or directory)
> ```
>
> İkinci örnekte `ENOENT` gördün mü? İşte `strace`'in gücü bu: soyut "dosya bulunamadı" hatası, **tam olarak
> hangi çağrının hangi hatayı** aldığına dönüşüyor. Faz 2'nin `EACCES`'i, Faz 7'nin `ECONNREFUSED`'ı burada
> somut kanıt olur.

> **💡 Cloud bağlantısı — SSH yoksa strace de yok:** `strace` güçlüdür ama makineye **girebiliyor olman**
> gerekir. Cloud'da bir instance'a SSH erişimin bozulmuşsa (Faz 7 — SG, key), `strace` yapamazsın. İşte bu
> yüzden cloud'da iki katmanlı gözlem kurarsın: (1) uygulama loglarını **dışarı** akıt (CloudWatch Logs —
> 11.5), makineye girmeden oku; (2) makineye girmek için SSM Session Manager kullan (SSH portu açık olmasa
> bile). Yerelde `strace` nihai hakemse, cloud'da "kanıtı makinenin dışına taşımak" nihai stratejidir.

## 11.3.3 `vmstat`, `iostat` ve `ss`'i tahmin etmeden okumak `[uygulama]`

11.3.1'deki tablo *hangi* araca uzanacağını söyler; burası *nasıl okunacağını*. Hangi sayılara bakacağını
bilmedikçe sayılar bir şey ifade etmez:

| Araç | Komut | Şunlara bak | Ne söylerler |
|---|---|---|---|
| `vmstat` | `vmstat 1 5` | `r`, `b`, `si`/`so`, `wa`, `st` | `r` (çalışmaya hazır) **CPU sayısından büyükse** → CPU-bound. `b` (kesilemez IO beklemesinde) → IO derdi. **Süregelen** sıfırdan farklı `si`/`so` → swap, bellek baskısı. `wa` yüksek → CPU *diski bekleyerek* boşta. `st` (steal) yüksek → hypervisor senden CPU alıyor (Hardware workbook'undaki gürültülü komşu) |
| `iostat` | `iostat -x 1` (paket `sysstat`) | `await`, `aqu-sz`, `%util` | `await` = bir isteğin beklemede + servis edilirken geçirdiği ortalama süre (ms) — uygulamanın *hissettiği* sayı. `aqu-sz` = kuyruk derinliği. HDD'de `%util` 100'e yakınsa doygunluk; SSD/NVMe'de **yanıltıcıdır** — `await`'e güven |
| `ss` | `ss -s` | duruma göre toplamlar | Hızlı sayım: kaç established, kaç `timewait` |
| `ss` | `ss -tan state established '( dport = :5432 )'` | bağlantı başına bir satır | Şu an veritabanına kaç bağlantı gidiyor — sızıntı, yalnızca büyüyen bir sayı olarak görünür |
| `lsof` | `sudo lsof +L1` | link sayısı 0 olan dosyalar | **Silinmiş ama hâlâ açık tutulan** dosyalar; `df` doluyken `du`'nun baytları bulamamasının sebebi (Faz 6) |

Birinci okuma kuralı: **tek satıra değil, eğilime bak.** `vmstat` ve `iostat`'ın ilk satırı açılıştan bu yana
ortalamadır; işe yarayanlar ondan sonrakilerdir. İkinci kural: **birleştir.** `vmstat`'tan yüksek `wa`, `iostat`'tan
yüksek `await` ve `ps`'te `D` durumunda bir process — üç bağımsız aracın aynı hikâyeyi anlatması kanıttır; hiçbiri
tek başına bir sezgidir.

> **🔧 Makinende gör** 🟡 — üç aracı birlikte oku (son adım /tmp'ye 500 MB yazar)
>
> ```
> $ vmstat 1 5                 # r, b, si, so, wa, st'ye bak (ilk satırı yok say)
> $ iostat -x 1 3              # await ve %util'e bak  (yoksa: sudo apt install sysstat)
> $ ss -s                      # kaç bağlantı, hangi durumlarda
> ```
>
> Boştaki bir laptop'ta her şey sıfıra yakındır ve mesele de budur: *sağlıklı* olan böyle görünür. Şimdi başka
> bir terminalde `dd if=/dev/zero of=/tmp/testfile bs=1M count=500 oflag=dsync` çalıştır ve `wa` ile `await`'in
> yükselişini izle — sonra `rm /tmp/testfile`. Bilerek bir disk darboğazı ürettin ve sayılarda nasıl göründüğünü
> öğrendin.

## 11.3.4 `/proc/<pid>/`: bir process'i içeriden okumak `[mekanizma]`

`/proc` **sanal** bir dosya sistemidir: içindeki hiçbir dosya diskte durmaz; çekirdek okunduğu anda üretir
(Faz 1'in "her şey bir dosyadır" fikri, teşhis aracına dönüşmüş hâli). `1234` numaralı process için:

| Yol | Ne okursun | Tipik kullanım |
|---|---|---|
| `/proc/1234/status` | Durum, `VmRSS` (bellekte duran), thread sayısı | "`D`'de mi? Gerçekte ne kadar RAM tutuyor?" |
| `/proc/1234/limits` | Kaynak limitleri, özellikle **Max open files** | "Too many open files" hatası: process'in gerçekten çalıştığı limit (kabuğunun `ulimit -n`'inden farklı olabilir) |
| `ls /proc/1234/fd \| wc -l` | Açık dosya tanımlayıcı sayısı | Yukarıdaki limitle karşılaştır — **sızıntı** ona doğru tırmanır |
| `/proc/1234/cmdline` | Tam komut satırı (NUL ile ayrılmış) | `tr '\0' ' ' < /proc/1234/cmdline` — *gerçekte* neyin hangi bayraklarla başlatıldığı |
| `/proc/1234/cwd`, `/proc/1234/exe` | Çalışma dizinine ve gerçek ikili dosyaya symlink | "Bu hangi ikili dosya, hangi dizinden?" |

Bir `systemd` servisinin pid'i `systemctl show myapp -p MainPID --value` ile gelir, yani zincir kısadır:
`cat /proc/$(systemctl show myapp -p MainPID --value)/limits`. "Servis *Too many open files* diyor ama kabuğumda
`ulimit`'i yükselttim" sorunu burada çözülür: servis senin kabuğunun altında çalışmaz, bu yüzden *onun* ne
aldığını yalnızca `/proc/<pid>/limits` gösterir (çözüm unit'tedir: `LimitNOFILE=`).

> **🤔 Düşün 11.3** — Bir servis `systemctl status` ile "active (running)" görünüyor, log yazmıyor, `top`'ta
> CPU/RAM normal, ama isteklere cevap vermiyor (takılmış). (a) Bu dört gözlem (durum, log, kaynak) neden
> yetersiz kaldı — her biri neyi göremez? (b) `strace -p <pid>` ile process'e bağlansan ve `read()` ya da
> `futex()` çağrısında "asılı" kaldığını görsen, bu sana ne söyler? (c) Neden `strace` bu durumda "nihai
> hakem"dir — diğer araçların göremediği neyi görür?
>
> *(Cevap: fazın sonunda)*

---
---

# 11.4 Üç İçgüdü Sorusu

Bir bulut mühendisinin refleksi üç soruya indirgenebilir. Her biri bir karar ağacıdır ve önceki fazlara
bağlanır.

## 11.4.1 "Sistem şu an ne yapıyor?" `[kavram]`

Bu soru **kaynak** sorusudur (Faz 4). Cevabı `top`/`htop` (CPU/RAM), `iostat`/`vmstat` (IO), `ss`
(ağ bağlantıları). "Sistem yavaş" belirsiz bir şikâyettir; bu soru onu somutlaştırır: CPU mı %100 (bir process
döngüde mi?), RAM mi doldu (swap'e mi düştü, OOM mu yakın?), IO mu darboğaz (disk mi yavaş?), yoksa ağ mı
(bağlantı mı bekliyor?). Refleks: "yavaş" → hangi kaynak → hangi process.

## 11.4.2 "Neden erişilemiyor?" — bir karar ağacı `[kavram]`

Bu, en sık ve en çok fazı birleştiren sorudur. "Erişemiyorum" (bir dosyaya, bir servise, bir port'a) dendiğinde
katman katman elenir:

- **İzin mi?** (Faz 2) — `ls -l`, kullanıcı/grup, rwx. `EACCES` mı alıyorsun?
- **Ownership mı?** (Faz 2) — dosya doğru kullanıcıya mı ait?
- **Mount mı?** (Faz 6) — dosya sistemi bağlı mı, `df`/`mount` ne diyor? "Var olmak ≠ erişilebilir olmak."
- **MAC mı?** (Faz 9) — izinler doğru ama AppArmor/SELinux mü engelliyor? `dmesg | grep -i denied`,
  `ausearch`. "İzinler doğru ama engelli."
- **Capability mi?** (Faz 9) — process'in gerekli capability'si var mı (`CAP_NET_BIND_SERVICE` port <1024)?
- **Ağ mı?** (Faz 7) — port dinleniyor mu, güvenlik duvarı, bind adresi (üç mercek)?

Bu karar ağacı Faz 2 + 6 + 7 + 9'u tek bir refleksde birleştirir: "erişilemiyor" tek bir neden değil, altı
farklı katmanın herhangi biri olabilir — ve doğru katmanı kanıtla bulursun.

## 11.4.3 "Boot'tan servise nerede kırıldı?" `[kavram]`

Bu soru **zaman/sıra** sorusudur (Faz 5). Bir makine açıldı ama uygulama çalışmıyor — zincirin neresinde
kırıldı? Boot → init (systemd) → hedef (target) → servis birimi → uygulama. `systemd-analyze` (boot süresi),
`systemctl list-units --failed` (başarısız birimler), `journalctl -b` (bu boot'un tüm logu). Faz 5'in "çalışan
durum vs kalıcı tanım" fikri burada kritiktir: servis **enabled** mı (boot'ta başlamalı) ama **failed** mı? Yoksa
hiç enabled değil mi? Zincirin hangi halkasının koptuğunu bulursun.

> **🤔 Düşün 11.4** — Bir EC2 instance'ı reboot sonrası açıldı, `ping` cevap veriyor, SSH çalışıyor, ama üzerinde
> koşması gereken web uygulaması yok. (a) "Boot'tan servise" zincirini (init → target → birim → uygulama) hangi
> komutlarla adım adım kontrol edersin? (b) `systemctl is-enabled myapp` "disabled" dönerse sorun nedir, ve bu
> Faz 5'in "çalışan durum vs kalıcı tanım" fikrine nasıl bağlanır? (c) Eğer "enabled" ama "failed" dönerse,
> hangi tek komutla nedenini bulursun?
>
> *(Cevap: fazın sonunda)*

---
---

# 11.5 Cloud'da Kanıt Toplama

## 11.5.1 "Instance sağlıklı ama uygulama değil" `[uygulama]`

Cloud'un en kafa karıştırıcı durumu budur ve bu fazın tüm metodolojisinin cloud karşılığıdır. AWS konsolu bir
EC2 instance için **"healthy"** (sağlıklı) diyor — ama uygulaman çalışmıyor. Neden? Çünkü AWS'in sağlık
kontrolleri (status checks) iki şeye bakar: (1) **System status** — altındaki fiziksel donanım/hipervizör iyi
mi; (2) **Instance status** — işletim sistemi ağ üzerinden ulaşılabilir, boot etmiş mi. **Hiçbiri senin
uygulamana bakmaz.** İşletim sistemi mükemmel açılmış, ama içindeki systemd servisi (Faz 8) failed durumda
olabilir — AWS bunu bilmez, "healthy" der.

Bu, Faz 8'in "kurulu olmak ≠ servis olarak çalışıyor olmak" ve Faz 5'in "çalışan durum vs kalıcı tanım"
fikirlerinin cloud'daki en keskin hâlidir: **instance sağlıklı olmak ≠ uygulama çalışıyor olmak.** Kanıtı
bulmak için AWS'in sağlık kontrolüne değil, kendi katman-katman metodolojine (11.2) inersin — ama makinenin
içine.

## 11.5.2 CloudWatch Logs ve SSM: makinenin içine SSH'siz ulaşmak `[uygulama]`

Yerelde loga `journalctl` ile bakarsın; cloud'da iki AWS aracı bunun yerini tutar:

- **CloudWatch Logs** — CloudWatch agent'ı kurarsan (Faz 10 — user-data ile), makinenin logları (journald,
  /var/log, uygulama logu) **AWS'e akar**. Böylece instance'a hiç girmeden, hatta instance çökmüş/silinmiş olsa
  bile logları okuyabilirsin. Bu, 11.3'teki "kanıtı makinenin dışına taşı" stratejisinin cloud uygulamasıdır.
- **SSM Session Manager** — makineye SSH portu (22) hiç açık olmadan, key olmadan, IAM üzerinden bir kabuk
  açmanı sağlar (Faz 7 + 9). SSH erişimin bozulsa bile (yanlış SG, kayıp key) SSM ile içeri girip `journalctl`,
  `systemctl status`, `strace` çalıştırabilirsin.

Refleks: cloud'da bir instance "sağlıklı ama uygulama yok" dediğinde — (1) önce CloudWatch Logs'a bak (SSH'siz),
(2) yetmezse SSM ile içeri gir, (3) içeride bu fazın katman-katman metodolojisini (log → servis → kaynak → ağ)
uygula. AWS'in "healthy" etiketi teşhisin başlangıcıdır, sonu değil.

> **💡 Cloud bağlantısı — gözlemi önceden kur:** Bu fazın en büyük cloud dersi şudur: kanıt toplama altyapısını
> **sorun çıkmadan önce** kurmalısın. Bir instance çöktükten sonra CloudWatch agent'ı kuramazsın; loglar zaten
> içeride kaybolmuştur. O yüzden CloudWatch agent'ı ve SSM'i AMI'ye (Faz 8) ya da user-data'ya (Faz 10) baştan
> koyarsın. "Gözlemlenebilirlik" bir düzeltme değil, bir **tasarım kararıdır** — sistemi kurarken kanıtın nasıl
> dışarı akacağını da tasarlarsın.


> **🤔 Düşün 11.5** — AWS konsolu bir instance için "2/2 kontrol geçti" gösteriyor, ama uygulama hata veriyor. Bir meslektaşın "AWS sağlıklı diyor, demek ki sorun makinede değil" diyor. (a) Neden yanılıyor? (b) SSH açmadan kanıtı hangi sırayla toplarsın? (c) Olaydan önce ne yapılmış olmalıydı?
>
> *(Cevap: fazın sonunda)*
---
---

# 11.6 Bu Faz Bozulunca — Yanlış Teşhis Refleksinin İmzaları

Bu fazın "bozulması", bir sistemin değil, **teşhis refleksinin** bozulmasıdır. Yanlış teşhis, doğru sorunu
gizler:

| Belirti | Muhtemel neden (yanlış refleks) | Doğrusu | İlgili bölüm |
|---|---|---|---|
| "Yeniden başlattım, düzeldi" ama tekrar bozuluyor | Belirtiyi maskeledin, nedeni bulmadın | Logdan başla, kök nedeni bul | 11.2.1 |
| Saatlerce rastgele komut denedin | Metodoloji yok, panik | Katman katman: log→servis→kaynak→ağ→çekirdek | 11.2.1 |
| "Log yok, o yüzden sorun yok" | Logsuzluğu yanlış okudun | "Neden log yok" yeni bir sorudur (başlamadı/başka yere/disk dolu) | 11.1.2 |
| CPU'ya baktın ama sorun IO'daydı | Yanlış kaynağa baktın | `iostat`/`vmstat` — hangi kaynak (Faz 4) | 11.3.1 |
| "Erişilemiyor" → sadece güvenlik duvarına baktın | Tek katman kontrol ettin | İzin/mount/MAC/capability/ağ — altı katman | 11.4.2 |
| AWS "healthy" diyor diye uygulama çalışıyor sandın | Sağlık kontrolünü yanlış anladın | Status check ≠ uygulama; içeri gir (SSM) | 11.5.1 |
| Instance çöktü, log yok | Gözlemi sonradan kurmaya çalıştın | CloudWatch agent'ı baştan kur | 11.5.2 |
| Her şey suskun, uygulama takılı, çare yok | strace'e uzanmadın | `strace -p <pid>` — nihai hakem | 11.3.2 |

> **Bu tablodan çıkan ders:** Bu fazın iki büyük fikri her satırın altında yatıyor. Birincisi **tahminle değil
> kanıtla**: bir sistemin en tehlikeli "düzeltmesi" nedeni bulmadan belirtiyi maskeleyendir ("yeniden başlattım,
> düzeldi") — çünkü sorun geri gelir ve bir dahaki sefere daha kötü zamanda. Katman-katman metodoloji tam da bu
> yüzden var: her adımda kanıt topla, bir sonrakine ancak eldeki katmanı eledikten sonra geç. İkincisi
> **kanıtı önceden dışarı taşı**: cloud'da makine kaybolduğunda içindeki kanıt da kaybolur; gözlemlenebilirliği
> sistemi kurarken tasarlarsın, kriz çıkınca değil. Ve unutma: bu fazın araçları neredeyse tamamen salt-okunur —
> gözlemlemekten korkma, korkulacak olan gözlemeden düzeltmeye kalkmaktır.

---
---

# Faz 11 — Düşün sorularının cevapları

## Cevap 11.1 — Önce journal'ı oku: `journalctl -u`, reboot sonrası `-b -1`

(a) `journalctl -u myapp -e` — `systemctl status` yalnızca son birkaç satırı gösterir; gerçek sebep (yapılandırma ayrıştırma hatası, port çakışması, permission denied) çoğunlukla journal'da görünür. Refleks: önce log'u oku, sonra tahmin et. (b) `journalctl -b -1 -u myapp` — bir önceki boot'un logları. (c) "Log yok" bir cevap değil yeni bir sorudur: (1) servis hiç başlamadı (bozuk unit — `journalctl -u`), (2) log'u başka yere yazıyor (stdout yerine dosyaya — yapılandırmaya bak), (3) logrotate log'u sildi ya da disk dolu olduğu için yazamıyor (`df -h`, Faz 6).
**İlgili bölüm:** 11.1.1-11.1.2 · **Devamı:** 11.2.1 (sistematik daraltma)

## Cevap 11.2 — Beş katmanı sırayla uygulamak, "önce yeniden başlat" refleksinden neden iyi

(a) Her katmanda tek komut ve aranan: **(1) Log** — `journalctl -u myapp -e` → hata satırı, exception,
"connection refused", "permission denied" ara. **(2) Servis** — `systemctl status myapp` → active mi, failed
mi, sürekli restart mı; "active (running)" mı yoksa "activating" mı. **(3) Kaynak** — `top` + `free -h` → CPU
%100 mü, RAM doldu mu, `dmesg | grep -i oom` → OOM killer devreye girdi mi. **(4) Ağ** — `ss -tulpn | grep
:8000` → port dinleniyor mu, hangi adrese bind (127.0.0.1 mı 0.0.0.0 mı — Faz 7). **(5) Çekirdek** — `dmesg -T
| tail` → disk hatası, sürücü, donanım. (b) `systemctl status` "active (running)" diyor ama tarayıcı açılmıyorsa
**ağ katmanına** (4) geçersin — çünkü servis süreç olarak çalışıyor demek "dinlediği port ve adres doğru" demek
değil; büyük ihtimalle uygulama `127.0.0.1`'e bind olmuş (Faz 7'nin klasik tuzağı) ya da güvenlik duvarı
engelliyor. (c) Bu sıra "önce yeniden başlat"tan iyidir çünkü yeniden başlatmak **nedeni yok etmeden belirtiyi
maskeler**: sorun geri gelir, üstelik yeniden başlatma logları da temizleyebilir, kanıtı yok edersin. Katman-
katman ise her adımda kanıt biriktirir, kök nedeni bulur ve kalıcı çözer.
**İlgili bölüm:** 11.2.1 · **Devamı:** 11.4 üç içgüdü sorusu.

## Cevap 11.3 — Durum/log/kaynak neden yetersiz kaldı; strace neden nihai hakem

(a) Üç gözlem şunu göremez: **`systemctl status`** sadece process'in **var olduğunu** söyler, ne yaptığını değil
— "running" bir process pekâlâ bir kilitte (deadlock) veya bir I/O'da sonsuza kadar bekliyor olabilir. **Log**
sadece uygulamanın **yazmayı seçtiğini** gösterir — uygulama bir çağrıda asılı kaldıysa ve log satırına hiç
ulaşamadıysa, log suskundur. **`top`** sadece **kaynak tüketimini** gösterir — bir process CPU/RAM yemeden de
takılabilir (örneğin bir soket okumasında `read()` beklerken CPU %0'dır ama iş yapmıyordur). (b) `strace -p
<pid>` ile process'in `read()` veya `futex()` çağrısında asılı kaldığını görmek şunu söyler: process **bir
şeyi bekliyor** — `read()` ise bir dosya/soketten hiç gelmeyen veri (belki karşı taraf cevap vermiyor — Faz 7),
`futex()` ise bir kilit (başka bir thread'in bırakmadığı lock → deadlock). (c) `strace` nihai hakemdir çünkü
process'in **gerçekte hangi sistem çağrısında, hangi argümanla, hangi dönüş değeriyle** takıldığını yalansız
gösterir — diğer araçların göremediği "process'in kernel'e ne sorduğu ve ne cevap aldığı" katmanını açar. Suskun
kalan her yerde o konuşur.
**İlgili bölüm:** 11.3.2 · **Devamı:** 11.5 cloud'da SSH'siz kanıt.

## Cevap 11.4 — Boot'tan servise zinciri; enabled vs failed; çalışan durum vs kalıcı tanım

(a) Zincir adım adım: `systemctl get-default` (hangi target'a boot etti) → `systemctl list-units --failed`
(başarısız birim var mı) → `systemctl status myapp` (birimin durumu) → `journalctl -u myapp -b` (bu boot'ta ne
dedi) → gerekirse uygulamanın kendi logu. (b) `systemctl is-enabled myapp` "disabled" dönerse sorun şudur:
servis **boot'ta otomatik başlamak üzere tanımlı değil** — birisi onu `systemctl start` ile elle başlatmış (Faz
5'in "çalışan durum") ama `systemctl enable` ile kalıcı tanıma (Faz 5'in "kalıcı tanım") eklememiş; reboot
edince "çalışan durum" kayboldu, "kalıcı tanım" hiç yoktu, servis geri gelmedi. Bu tam olarak Faz 5'in "çalışan
durum ≠ kalıcı tanım" dersidir. Çözüm: `systemctl enable myapp`. (c) "enabled" ama "failed" dönerse tek komut:
`journalctl -u myapp -b` — birim boot'ta başlamayı denedi ama neden başaramadığını (config hatası, eksik bağımlılık,
izin) log söyler.
**İlgili bölüm:** 11.4.3 · **Devamı:** 11.5 "instance sağlıklı ama uygulama değil".

## Cevap 11.5 — Sağlıklı instance ≠ çalışan uygulama: önce CloudWatch Logs, sonra SSM

(a) Durum kontrolleri yalnızca iki şeye bakar: sistem durumu (alttaki donanım/hypervisor) ve instance durumu (OS açıldı ve erişilebilir). İkisi de uygulamana bakmaz; içerideki systemd servisi failed durumunda olabilir ve AWS yine de "sağlıklı" der — "kurulu ≠ servis olarak çalışıyor"un cloud biçimi. (b) Önce CloudWatch Logs (SSH gerekmez), sonra IAM üzerinden içeri girmek için SSM Session Manager; içeride katman katman metodoloji: log → servis → kaynak → ağ. (c) CloudWatch agent'ı ve SSM AMI'de ya da user-data'da **önceden** olmalıydı — makine çöktükten sonra agent kurulamaz; gözlemlenebilirlik bir tasarım kararıdır.
**İlgili bölüm:** 11.5.1-11.5.2 · **Devamı:** 12.4.2 (CloudWatch)

---
---

# Faz 11 — Sık sorulan sorular

**S1 — Bir şey bozulunca ilk ne yapmalıyım?** Panik yapıp yeniden başlatma. Önce logu oku (`journalctl -u
<servis> -e`), sonra katman katman daralt: log → servis → kaynak → ağ → çekirdek (11.2.1).

**S2 — `journalctl` ile `/var/log` arasındaki fark ne?** journalctl systemd'nin merkezî journal'ını okur
(birim/zaman/boot filtreleriyle); `/var/log` geleneksel dosya loglarıdır (syslog, auth.log, kern.log). Modern
servisler journal'a, birçok geleneksel servis hâlâ /var/log'a yazar (11.1).

**S3 — `strace`'i ne zaman kullanırım?** Diğer her şey suskun kaldığında: log yok, durum "running", kaynak
normal ama uygulama takılı. `strace -p <pid>` process'in hangi sistem çağrısında asılı kaldığını gösterir —
nihai hakem (11.3.2).

**S4 — "Erişilemiyor" deyince nereye bakarım?** Tek yere değil, altı katmana: izin (Faz 2), ownership (Faz 2),
mount (Faz 6), MAC (Faz 9), capability (Faz 9), ağ (Faz 7). Karar ağacıyla ele (11.4.2).

**S5 — AWS "healthy" diyorsa uygulama çalışıyor demek mi?** Hayır. Status check'ler donanım ve OS'un
ulaşılabilirliğine bakar, senin uygulamana değil. "Instance sağlıklı ≠ uygulama çalışıyor" — içeri girip
kontrol et (11.5.1).

**S6 — Instance çökmüşse loglara nasıl bakarım?** Önceden CloudWatch Logs kurduysan loglar AWS'e akmıştır,
makine olmasa da okursun. SSH bozuksa SSM Session Manager ile içeri girersin. Gözlemi baştan kur (11.5.2).

**S7 — "Sistem yavaş" — nereden başlarım?** "Hangi kaynak" sorusuyla: `top` (CPU), `free` (RAM), `iostat` (IO),
`ss` (ağ). Yavaşlık tek şey değil; önce darboğazın hangi kaynakta olduğunu bul (11.4.1, Faz 4).

---
---

# Faz 11 — Kendini sına

Cevaplarını kâğıda yaz, sonra anahtarla karşılaştır. Hedef: 18 üzerinden 14+.

## Bölüm A — Tanım ve mekanizma

1. Katman-katman debugging metodolojisinin beş katmanını sırayla say. Neden bu sıra?
2. `journalctl -u`, `-b`, `-p err`, `-k` filtreleri ne yapar?
3. `strace` neyi gösterir ve neden "nihai hakem"dir?
4. "Erişilemiyor" karar ağacının altı katmanını say (hangi faz).
5. "Sistem şu an ne yapıyor" sorusunun aracı hangileri ve hangi kaynağa bakar?
6. auth.log ne içerir ve hangi fazın forensiğine bağlanır?
7. logrotate neden var, hangi Faz 6 senaryosunu önler?
8. `top` ile `iostat` arasındaki fark — hangi soruya hangisi cevap verir?

## Bölüm B — Uygulama ve teşhis

9. Bir web uygulaması "açılmıyor". İlk üç katmanda hangi komutları çalıştırırsın ve ne ararsın?
10. `systemctl status` "active (running)" diyor ama tarayıcı açılmıyor. Hangi katmana geçersin, hangi komut, ne
    ararsın?
11. Bir servis reboot sonrası gelmedi; `is-enabled` "disabled" diyor. Sorun ne, çözüm ne (Faz 5 bağlantısı)?
12. Uygulama takılı, log yok, durum "running", kaynak normal. Sıradaki adımın ne ve neden?
13. AWS konsolu "healthy" diyor ama uygulama çalışmıyor. Hangi iki AWS aracıyla kanıtı toplarsın?
14. Bir dosyaya "izniniz doğru ama erişilemiyor". Hangi iki katmanı (Faz 9) kontrol edersin ve hangi komutla?

## Bölüm C — Muhakeme ve bağlantı

15. Neden "yeniden başlattım, düzeldi" tehlikeli bir teşhis alışkanlığıdır?
16. "Log yok" neden bir cevap değil yeni bir sorudur? Üç olası nedeni say.
17. Cloud'da gözlemlenebilirliği neden "sorun çıkmadan önce" kurmalısın? "Instance çöktü, log yok" durumunu bağla.
18. Bu fazın araçlarının neredeyse tamamı salt-okunur olması teşhis felsefesini nasıl şekillendirir?

---

## Cevap anahtarı

1. Uygulama logu → servis durumu → kaynak → ağ → çekirdek; en ucuz/en olası kontrolden en derin/en nadire doğru
   (11.2.1). — 2. `-u` birim logu, `-b` bu boot, `-p err` hata seviyesi ve üstü, `-k` çekirdek mesajları
   (11.1.1). — 3. Process'in yaptığı her sistem çağrısını (dönüş değeriyle) gösterir; diğer her şey suskunken
   process'in gerçekte ne yaptığını yalansız açtığı için nihai hakem (11.3.2). — 4. İzin (Faz 2), ownership (Faz
   2), mount (Faz 6), MAC (Faz 9), capability (Faz 9), ağ (Faz 7) (11.4.2). — 5. `top`/`htop` (CPU/RAM),
   `iostat`/`vmstat` (IO), `ss` (ağ); "hangi kaynak darboğaz" (11.4.1). — 6. SSH girişleri, sudo kullanımı,
   başarısız girişler; Faz 9 güvenlik forensiği (11.1.2). — 7. Logların diski doldurmasını önler (döndürür,
   sıkıştırır, siler); Faz 6'daki "disk %100 → sistem donuyor" (11.1.2). — 8. `top` CPU/RAM/process ("ne
   yapıyor"), `iostat` disk IO ("yavaşlık disk mi") (11.3.1).

9. (1) `journalctl -u myapp -e` → hata satırı; (2) `systemctl status myapp` → failed/running; (3) `top`+`free`
   → CPU/RAM/OOM (11.2.1). — 10. Ağ katmanı; `ss -tulpn | grep <port>` → port dinleniyor mu, 127.0.0.1'e mi
   bind (Faz 7 tuzağı) (Cevap 11.2b). — 11. Servis "çalışan durum" olarak elle başlatılmış ama "kalıcı tanıma"
   (enable) eklenmemiş; reboot'ta kayboldu; çözüm `systemctl enable myapp` (Faz 5) (Cevap 11.4b). — 12. `strace
   -p <pid>` — hangi sistem çağrısında (read/futex) asılı kaldığını gösterir; diğer araçlar bunu göremez
   (11.3.2). — 13. CloudWatch Logs (SSH'siz log oku) + SSM Session Manager (içeri gir); status check ≠ uygulama
   (11.5). — 14. MAC (`dmesg | grep -i denied`, AppArmor/SELinux) ve capability (`CAP_NET_BIND_SERVICE`); Faz 9
   (11.4.2). — 15. Nedeni bulmadan belirtiyi maskeler; sorun geri gelir, üstelik yeniden başlatma kanıtı (logu)
   temizleyebilir (11.2.1, 11.6). — 16. Logsuzluk sorunun yokluğu değil derinliğinin kanıtı olabilir: (1) hiç
   başlamadı, (2) başka yere yazıyor, (3) disk dolu/logrotate sildi (11.1.2). — 17. Makine çökünce/silinince
   içindeki kanıt kaybolur; agent'ı sonradan kuramazsın; gözlemlenebilirlik bir tasarım kararıdır, CloudWatch
   agent'ı AMI/user-data'ya baştan koyarsın (11.5.2). — 18. Salt-okunurluk "gözlemlemekten korkma, sistemi
   değiştirmekten önce anla" felsefesini besler; kanıt topla, sonra kanıta dayalı tek bir kalıcı düzeltme yap
   (11.6).

## Puanlama

| Doğru | Anlamı |
|---|---|
| 16-18 | Sistematik teşhis refleksin oturmuş. Faz 12'ye (Cloud'a Köprü) hazırsın. |
| 13-15 | İyi. Kaçırdığın soruların bölümlerini tekrar oku (özellikle 11.2 ve 11.4). |
| 9-12 | Temel var ama araçlar dağınık. 11.3 tablosunu ve üç içgüdü sorusunu çalış. |
| 0-8 | Fazı yeniden gez; kendi test makinende bir servisi kasıtlı boz ve kanıtı katman katman bul. |

Kaçırdığın soru → dönmen gereken bölüm:

| Soru | Bölüm |
|---|---|
| 1, 9, 15 | 11.2.1 Metodoloji |
| 2, 6, 7, 16 | 11.1 Loglar |
| 3, 12 | 11.3.2 strace |
| 4, 14 | 11.4.2 "Neden erişilemiyor" |
| 5, 8 | 11.3.1 / 11.4.1 Araçlar ve kaynak |
| 10 | 11.2 + Faz 7 ağ |
| 11 | 11.4.3 Boot'tan servise |
| 13, 17 | 11.5 Cloud'da kanıt |
| 18 | 11.6 Teşhis felsefesi |

---
---

# Faz 11 — Kapanış ve Faz 12'ye Köprü

## Bu fazdan ne taşıyorsun

Faz 11 seni "bir şey bozulunca panikleyen"den "kanıtı katman katman bulan"a dönüştürdü. İki kalıcı fikir
kazandın: **tahminle değil kanıtla** (log → servis → kaynak → ağ → çekirdek sistematik daralması; her araca ne
zaman uzanacağın; strace'in nihai hakemliği) ve **kanıtı önceden dışarı taşı** (cloud'da CloudWatch Logs + SSM
ile makinenin içine SSH'siz ulaşmak, "instance sağlıklı ≠ uygulama çalışıyor"). Bu faz aynı zamanda önceki tüm
fazların "bu faz bozulunca" notlarını tek bir metodolojide birleştirdi: erişilemezlik altı katmandan biri, boot
sorunu zincirin bir halkası, yavaşlık bir kaynak darboğazı olabilir — ve doğru katmanı kanıtla bulursun.

## Faz 12 nereye bağlanıyor

Faz 11 sana "bozulanı bulmayı" verdi; **Faz 12 tüm yolculuğu tek bir hikâyede birleştirir**: boş bir Ubuntu
AMI'sinin boot'tan production'a yolculuğu. Faz 0-11'de öğrendiğin her parça (AMI, cloud-init, systemd, mount,
SG, IAM, capability, log) Faz 12'de bir AWS kavramına oturur ve hepsi tek bir zincirde buluşur. Faz 11'in
teşhis refleksi orada da işine yarar: o zincirin herhangi bir halkası koptuğunda, bu fazın katman-katman
metodolojisiyle nerede kırıldığını bulursun.

> **🤔 Faz çıktısı — kendine sor:** Bu fazın "kanıtı önceden dışarı taşı" fikri, Faz 12'nin "boş AMI'den
> production'a" hikâyesine nasıl bağlanıyor? Bir instance'ın gözlemlenebilirliğini (CloudWatch agent, SSM)
> boot yolculuğunun **hangi adımına** koyman gerektiğini düşün — AMI'ye mi (Faz 8), user-data'ya mı (Faz 10)? Ve
> neden bu bir "sonradan eklenecek özellik" değil, boot zincirinin baştan bir parçası olmalı?
>
> **🧪 Lab 11 fikri (kendi test makinende):** Kasıtlı bir arıza kur ve katman-katman çöz: (1) Basit bir web
> sunucusu servisi yaz (`python -m http.server 8000` bir systemd birimi olarak), enable+start et. (2) Şimdi onu
> **boz**: config'de `127.0.0.1:8000`'e bind ettir. (3) Başka bir makineden/terminalden erişmeye çalış — zaman
> aşımı. (4) Metodolojiyi uygula: `journalctl -u` (log ne diyor), `systemctl status` (running mi), `ss -tulpn`
> (hangi adrese bind — işte kanıt: 127.0.0.1!). (5) Düzelt (`0.0.0.0`), doğrula. Bu lab, Faz 7'nin bind tuzağını
> Faz 11'in teşhis refleksiyle birleştirir — kanıtı ağ katmanında bulursun.

---

> **Navigasyon:** [◀ Faz 10 — Otomasyon ve Scripting](Faz_10_Otomasyon_ve_Scripting.md) · **Faz 11** · [Faz 12 — Cloud'a Köprü ▶](Faz_12_Cloud_a_Kopru.md)
