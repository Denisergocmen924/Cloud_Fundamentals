# Faz 4 — Bellek, I/O ve Performans Sezgisi

> **Navigasyon:** [◀ Faz 3 — Process ve Kaynak](Faz_3_Process_ve_Kaynak.md) · **Faz 4** · [Ara Sınav 2 ▶](Ara_Sinav_2.md)

---

## Nereden geliyoruz

Faz 3'te çalışan sistemin atomunu — process'i — açtık: nasıl doğar, hangi durumda olur, nasıl ölür,
kaynağı nasıl sınırlanır. `top` açtığında gördüğün tabloyu artık ezberden değil modelden okuyorsun.
Ama Faz 3'ün sonunda bir tabloyu yarım bıraktık. Hatırla — bir sunucuda `top` açtın ve şunu gördün:

```
%Cpu(s):  4.0 us,  2.0 sy,  0.0 ni, 30.0 id, 63.5 wa, ...
```

Load 6.2'ydi ama CPU'nun çoğu `wa` (I/O bekleme) ve `id` (boşta) idi. O zaman "darboğaz CPU değil,
disk" dedik ama **neden** olduğunu açmadık. Bu faz tam olarak o `wa` satırının altını açıyor.

Faz 3'ten üç şey burada doğrudan işe yarayacak:

- **Load average = R + D process sayısı** (Faz 3.2.1). Şimdi bunun neden "CPU meşgul" demek olmadığını,
  D-state'in neden load'u şişirdiğini metrik metrik göreceğiz.
- **cgroup bellek limiti aşılınca OOM olur** (Faz 3.6.2). Şimdi OOM Killer'ın **kimi** öldüreceğine
  nasıl karar verdiğini (`oom_score`), swap'ın bu resme nasıl girdiğini açacağız.
- **`ps`'te `%MEM`, `VSZ`, `RSS` sütunları** (Faz 3.7). O sütunların ne anlama geldiğini,
  neden `VSZ`'nin "kullanılan bellek" olmadığını burada öğreneceğiz.

## Bu fazın sorusu

Faz 3'ün "sistem ne yapıyor" sorusunu bu faz **kaynak** eksenine taşıyor:

> *"Kaynağı ne yiyor — ve 'yavaş' dendiğinde darboğaz nerede?"*

Bir cloud engineer'in en sık duyduğu cümle "uygulama yavaş"tır. Ama "yavaş"ın dört ayrı sebebi
olabilir: CPU mu doygun, bellek mi bitti, disk mi boğuldu, ağ mı tıkandı? Bu dördü **farklı**
araçlarla teşhis edilir ve **farklı** çözümler ister. Yanlış teşhis, yanlış "daha büyük instance al"
kararına ve boşa harcanan paraya yol açar.

Bu fazın sonunda `free -h` çıktısına bakıp "RAM %95 dolu" panik cümlesini **düzelteceksin** — çünkü
o rakamın çoğu page cache'tir ve gerçek baskı `available` sütunundadır. Ve "uygulamam yavaş"
dendiğinde tek bir komutla başlayıp 60 saniyede darboğazın dört türünden hangisi olduğunu
daraltacaksın. Bu, ileride EC2 right-sizing ve CloudWatch metriklerini okumanın OS tarafıdır.

---

## Bu fazın sonunda

- Sanal bellek ile fiziksel bellek farkını; `RSS` (gerçekte kullanılan RAM) ile `VSZ` (ayrılan sanal
  alan) arasındaki farkı açıklayabilecek; neden process'lerin `RSS`'lerini toplamanın yanıltıcı
  olduğunu (shared memory) bileceksin
- Linux'un boş RAM'i neden disk cache'i (page cache) olarak kullandığını; `free` çıktısındaki
  `buff/cache`'in aslında **müsait** olduğunu ve gerçek baskının `available` sütununda okunduğunu
  açıklayabileceksin
- Swap'in ne olduğunu, neden yavaş olduğunu; `swappiness`'in ne ayarladığını; OOM Killer'ın RAM
  tükenince **kimi** ve neden öldürdüğünü (`oom_score`) açıklayabileceksin
- Load average'ı core sayısına göre doğru yorumlayabilecek; "yüksek load + düşük CPU" gördüğünde bunun
  neredeyse her zaman I/O wait olduğunu bileceksin
- `iowait`in ne anlattığını; blocking ve non-blocking I/O farkını; disk doygunluğunun neden tüm
  sistemi "yavaş" gösterdiğini açıklayabileceksin
- "Uygulamam yavaş" dendiğinde CPU-bound / memory-bound / I/O-bound / network-bound ayrımını hangi
  araçla (`free`, `vmstat`, `iostat`, `top`, `/proc/meminfo`) yapacağını metodolojiyle
  anlatabileceksin

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden burada |
|---|---|---|---|
| 4.1 | Sanal bellek ve process belleği: RSS vs VSZ | `[mekanizma]` | "Uygulama ne kadar RAM yiyor" sorusunun doğru cevabı |
| 4.2 | Page cache: "boş RAM" yanılgısı | `[mekanizma]` + `[uygulama]` | **Cloud'un en çok yanlış okunan metriği** |
| 4.3 | Swap ve OOM Killer | `[mekanizma]` | Bellek tükenince ne olur; kim ölür, neden |
| 4.4 | Load average'ı doğru okumak | `[mekanizma]` + `[uygulama]` | Faz 3'ten kalan `wa` satırının altı |
| 4.5 | I/O sezgisi: iowait, blocking I/O | `[kavram]` + `[uygulama]` | "Disk boğuldu, CPU boşta" nasıl olur |
| 4.6 | Darboğaz triyajı (OS gözüyle) | `[uygulama]` | **Fazın çıktısı** — 60 saniyede darboğazı daralt |
| 4.7 | Bu faz bozulunca | — | Bellek ve I/O arızalarının imzaları |

> **Bu fazda nasıl çalışmalı:** Bu fazın komutlarının **hemen hepsi** 🟢 (sadece okur — `free`,
> `vmstat`, `iostat`, `top`, `/proc/meminfo` okuma). Bu faz bir "gözlem" fazıdır; kalıcı bir şey
> değiştirmezsin. Yalnızca birkaç kutu 🟡: bir cgroup limitini `systemd-run` ile canlı test etmek
> veya `stress` aracıyla yapay yük bindirmek — bunlar geçicidir, işlem bitince kaybolur. `swappiness`
> veya OOM ayarlarını değiştirmek 🔴 olurdu ama bu fazda sadece **okuyoruz**, değiştirmiyoruz. En
> aydınlatıcı an: `free -h` açık dururken bir dosyayı `cat büyük_dosya > /dev/null` ile okuyup
> `buff/cache`'in canlı büyümesini izlemek.

---
---

# 4.1 Sanal Bellek ve Process Belleği

## 4.1.1 Sanal bellek: her process kendini yalnız sanır `[mekanizma]`

Faz 3'te process'in çekirdek gözünde bir "task" olduğunu gördük. Şimdi o task'ın **belleğine**
bakıyoruz. İlk kurmamız gereken model şu: **hiçbir process fiziksel RAM'e doğrudan dokunmaz.**

Her process, çekirdeğin ona sunduğu **sanal bir adres uzayında** yaşar. Bu uzay, process'in
kendisine ait, 0'dan başlayan, kesintisiz, kocaman bir bellek gibi görünür — sanki makinedeki tek
process oymuş, tüm bellek onunmuş gibi. Ama bu bir yanılsamadır: process'in gördüğü **sanal adresler**,
çekirdek ve donanım (MMU — Memory Management Unit) tarafından arka planda **fiziksel adreslere**
çevrilir. İki farklı process aynı sanal adresi (`0x400000`) kullanabilir, ama bunlar tamamen farklı
fiziksel RAM bölgelerine düşer.

Bu neden var? Üç sebep:

1. **İzolasyon.** Bir process başka bir process'in belleğini adresleyemez — çünkü onun sanal
   adresleri farklı fiziksel yerlere çevrilir. Faz 2'nin kimlik izolasyonunun bellek tarafı budur.
2. **Basitlik.** Her programcı "belleğim 0'dan başlar" varsayabilir; fiziksel RAM'in nerede boş
   olduğunu düşünmesi gerekmez.
3. **Esneklik.** Çekirdek, bir sanal sayfayı fiziksel RAM'e, diske (swap) veya hiçbir yere (henüz
   dokunulmamış) bağlayabilir. Bu esneklik, bu fazın yarısının temelidir.

> **🔧 Makinende gör** 🟢 — bir process'in sanal bellek haritası
>
> ```
> $ cat /proc/self/maps | head -5
> 55a3c1e00000-55a3c1e21000 r--p 00000000 08:01 1183     /usr/bin/cat
> 55a3c1e21000-55a3c1ea6000 r-xp 00021000 08:01 1183     /usr/bin/cat
> 55a3c1ea6000-55a3c1ecb000 r--p 000a6000 08:01 1183     /usr/bin/cat
> ...
> 7fff...       rw-p 00000000 00:00 0        [stack]
> ```
>
> `/proc/self/maps` o an okuyan process'in (yani bu `cat`'in) sanal adres haritasıdır. Her satır bir
> **bölge**: soldaki adres aralığı sanaldır, `r-xp` izinleri (Faz 2'nin mod bitleri gibi ama bellek
> için), sağdaki ise o bölgenin **neyle desteklendiği** — bir dosya (`/usr/bin/cat`), stack, heap.
> "Her şey dosyadır" (Faz 0) burada da geçerli: process'in belleği bile bir dosya gibi okunuyor.

**Sayfa (page).** Çekirdek belleği bayt bayt değil, sabit boyutlu bloklar hâlinde yönetir: bunlara
**sayfa** (page) denir, tipik boyutu 4 KB'dir. Sanaldan fiziksele çeviri sayfa granülaritesinde
yapılır. Bu detay şimdilik küçük görünebilir ama page cache, swap ve OOM'un hepsi "sayfa" biriminde
çalışır — o yüzden kelimeyi şimdiden tanıyor ol.

---

## 4.1.2 RSS vs VSZ: "ayrılan" ile "gerçekten kullanılan" farkı `[mekanizma]`

Şimdi `ps` ve `top`'ta gördüğün iki bellek sütununu ayırabiliriz — bu ayrım, "bu uygulama ne kadar
RAM yiyor" sorusunun doğru cevabının temelidir.

- **VSZ (Virtual Size — vi-es-zed).** Process'in sanal adres uzayının **toplam** boyutu. Yani
  process'in çekirdekten "ayırdığı" tüm sanal alan: kod, veri, stack, yüklenen kütüphaneler,
  `malloc` ile istenmiş ama henüz **dokunulmamış** bölgeler — hepsi. Bu sayı çoğu zaman çok
  büyüktür ve **fiziksel RAM ile doğrudan ilgisi yoktur.**
- **RSS (Resident Set Size — re-zi-dent set sayz).** Process'in sanal sayfalarından şu an **gerçekten
  fiziksel RAM'de** oturanların toplamı. "Resident" = RAM'de ikamet eden. Uygulamanın o an RAM'e
  gerçekte ne kadar bindiğini bu sayı gösterir — VSZ değil.

Farkın kaynağı sanal belleğin doğasıdır: bir process `malloc(1 GB)` yapabilir ve VSZ 1 GB artar — ama
o 1 GB'a **dokunana** kadar çekirdek tek bir fiziksel sayfa bile ayırmaz. Buna **lazy allocation**
(tembel ayırma) denir. Sadece process bir sayfaya gerçekten yazdığında çekirdek o an bir fiziksel
sayfa bağlar (page fault ile) ve RSS o kadar artar. Yani:

> **VSZ = process'in söz verdiği (ayırdığı) alan; RSS = process'in gerçekten kullandığı RAM.**

> **🔧 Makinende gör** 🟢 — VSZ ile RSS'in farkını yakala
>
> ```
> $ ps -eo pid,comm,vsz,rss --sort=-rss | head -6
>    PID COMMAND            VSZ    RSS
>   1421 mysqld         2148372 512400
>    987 java           4821960 398210
>    623 systemd-journ   58210  18400
> ```
>
> `mysqld`'in VSZ'si ~2.1 GB ama RSS'i ~512 MB. Yani MySQL 2.1 GB **sanal** alan ayırmış ama fiziksel
> RAM'de gerçekten ~512 MB tutuyor. "MySQL 2 GB RAM yiyor" demek yanlış olurdu — VSZ okunmuş olurdu.
> Doğru cümle: "MySQL ~512 MB RSS kullanıyor." **Kural: bellek kullanımını daima RSS'ten oku, VSZ'den
> değil.** (`top`'ta da `RES` = RSS, `VIRT` = VSZ.)

> **⚠️ Yaygın yanılgı: "VSZ büyükse process çok RAM yiyor demektir."**
> Hayır. VSZ'nin büyük olması sadece process'in çok **sanal alan ayırdığını** gösterir — ki bu bedava
> gibidir, dokunulmadıkça fiziksel RAM harcamaz. Özellikle JVM ve Go gibi runtime'lar başlangıçta
> devasa sanal alan ayırır (VSZ = onlarca GB olabilir) ama RSS küçük kalır. Panik ölçütün her zaman
> RSS'tir.

---

## 4.1.3 Shared memory: neden RSS'leri toplayamazsın `[kavram]`

Bir adım daha: diyelim 10 tane Apache/PHP worker process'in var, her birinin RSS'i 200 MB görünüyor.
"O zaman 10 × 200 MB = 2 GB RAM yiyor" demek doğru mu? **Hayır** — ve sebebi bu fazın en kurnaz
detayı.

Process'lerin RSS'inin büyük kısmı çoğu zaman **paylaşılan** sayfalardır. En tipik örnek: **paylaşılan
kütüphaneler** (shared libraries — `libc`, `libssl` vb.). 10 worker da aynı `libc`'yi kullanır ve
çekirdek bu kütüphaneyi **fiziksel RAM'de tek bir kopya** olarak tutup 10 process'in sanal uzayına
aynı fiziksel sayfaları bağlar. Ama her process'in RSS'i bu paylaşılan sayfaları **kendi payına
sayar**. Yani aynı 50 MB'lık `libc`, 10 process'in RSS toplamında 500 MB gibi görünür — oysa fiziksel
RAM'de tek 50 MB var.

Aynı durum `fork`'ta da geçerli (Faz 3.1.2 hatırla): `fork` ile doğan çocuk, ebeveynin bellek
sayfalarını başta **paylaşır** (Copy-on-Write). Ebeveyn ve çocuk aynı sayfaya yazana kadar tek kopya
kullanılır. Yani ebeveyn + çocuğun RSS toplamı, gerçek fiziksel kullanımdan büyük görünür.

> **🤔 Düşün 4.1** — Bir sunucuda `ps` ile 8 tane `php-fpm` worker görüyorsun, her birinin RSS'i
> 180 MB. Bir arkadaşın "bunlar 8 × 180 = 1.44 GB RAM yiyor, instance küçük kalmış" diyor. Bu hesap
> neden yanlış olabilir, ve gerçek fiziksel kullanımı hangi kavramla açıklarsın?
>
> *(Cevap: fazın sonunda)*

Bunun pratik sonucu: **process'lerin RSS'lerini toplayarak "toplam RAM kullanımı" bulmaya çalışma.**
Bu her zaman fazla tahmin eder. Toplam gerçek kullanımı sistem düzeyinden — `free` veya
`/proc/meminfo`'dan — okumak gerekir. Ve işte bu bizi bu fazın kalbine, "boş RAM" yanılgısına
getiriyor.

> **❓ Akla gelen soru: "Peki bir process'in gerçekten 'özel' (paylaşılmayan) kullandığı RAM'i nasıl
> görürüm?"**
> `RSS` yerine `PSS` (Proportional Set Size) veya daha net olarak `/proc/<PID>/smaps_rollup`'taki
> `Private` satırlarına bakılır — paylaşılan sayfalar process sayısına bölünür. Pratikte `smem`
> aracı bunu okunur biçimde verir. Bu fazda derine inmiyoruz ama "gerçek kullanım için PSS'e bak"
> refleksini not et.

---
---

# 4.2 Page Cache — "Boş RAM" Yanılgısı

## 4.2.1 Linux boş RAM'i çöp olarak görmez: page cache `[mekanizma]`

Şimdi bu fazın — belki tüm haritanın — en çok yanlış okunan konusuna geldik. Yeni bir sunucuya girip
`free -h` çalıştırırsın ve RAM'in %90'ının "kullanımda" göründüğünü görüp panikleyebilirsin. Neredeyse
her zaman bu bir yanılgıdır. Nedenini anlamak için Linux'un bir felsefesini kurmak lazım:

> **Boş RAM, boşa harcanan RAM'dir.**

Linux, kullanılmayan RAM'i öylece boş bırakmayı israf sayar. Onun yerine, diskten okunan her şeyi —
dosya içeriklerini — RAM'de **kopya olarak** tutar. Buna **page cache** denir. Mantık şu: disk
RAM'den binlerce kat yavaştır (Faz 0/donanım sezgisi). Bir dosyayı bir kez diskten okudunsa, çekirdek
onu page cache'te tutar; ikinci kez okuduğunda diske hiç gitmeden RAM'den verir — yani **uçar**.

Bu yüzden çalışan bir sistemde RAM zamanla "dolar": çekirdek okunan dosyaları cache'te biriktirir.
Ama bu dolu görünen RAM **her an geri alınabilir**: yeni bir process RAM isterse, çekirdek page
cache'ten bir kısmını anında bırakır (çünkü orası diskte zaten var, kaybolmaz) ve process'e verir.
Yani page cache "kullanımda" ama aynı zamanda "müsait".

> **🔧 Makinende gör** 🟢 — page cache'i canlı büyüt
>
> ```
> $ free -h
>                total        used        free      buff/cache   available
> Mem:            7.7Gi       1.2Gi       5.1Gi        1.4Gi        6.1Gi
> $ cat /var/log/*.log > /dev/null      # büyük dosyaları okut (diske değil, cache'e)
> $ free -h
>                total        used        free      buff/cache   available
> Mem:            7.7Gi       1.2Gi       3.0Gi        3.5Gi        6.1Gi
> ```
>
> Dikkat: `used` neredeyse **hiç değişmedi** (1.2 GB), `free` düştü, `buff/cache` şişti (1.4 → 3.5
> GB) — çünkü okuttuğun dosyalar page cache'e girdi. Ama en önemli sütun: **`available` aynı kaldı
> (6.1 GB).** Yani "gerçekten müsait" bellek değişmedi; sadece boş RAM cache'e dönüştü. Bu, page
> cache'in "geri alınabilir" olduğunun canlı kanıtıdır.

`free` çıktısındaki sütunları netleştirelim:

- **`used`** — process'lerin gerçekten tuttuğu (anonim) bellek + çekirdek. Asıl "harcanan" budur.
- **`buff/cache`** — page cache + buffer'lar. Dolu görünür ama **geri alınabilir**.
- **`free`** — hiç dokunulmamış, tamamen boş RAM. Çalışan bir sistemde bunun küçük olması **normaldir**.
- **`available`** — çekirdeğin tahmini: yeni bir uygulama başlatırsan diske swap etmeden ne kadar RAM
  verebilir. **Gerçek baskıyı ölçen sütun budur.**

![Şekil 4.1 — `free -h` gerçekte ne gösterir: used (gerçek baskı) + buff/cache (geri alınabilir) → available](../diagrams/png/lx-4-01-free-memory-anatomy.png)

*Şekil 4.1 — Sol: `free -h` çıktısına saf bakışın kurduğu yanlış model ("RAM %81 dolu, panik"). Sağ:
gerçek model — `used` küçük, `buff/cache` geri alınabilir, ve gerçek karar sütunu `available`. Aynı
RAM, iki farklı okuma.*

## 4.2.2 `free`'yi doğru okumak: `available` sütunu `[uygulama]`

Kuralı tek cümleye indirelim:

> **"RAM ne kadar dolu?" sorusunun cevabı `used`/`total` değil, `available`/`total`'dir.**

`available` yüksekse (örneğin total'in %40+'ı), RAM'in ne kadar "dolu" göründüğü önemsizdir — sistem
rahattır. `available` sıfıra yaklaşıyorsa, işte o zaman gerçek bellek baskısı var demektir ve swap /
OOM riski başlar.

> **⚠️ Yaygın yanılgı: "`free`'de `used` yüksek / `free` düşük, RAM bitiyor, daha büyük instance
> lazım."**
> Bu, cloud'da en pahalı yanlış okumalardan biridir. `used` düşükse ve düşen tek şey `free` sütunuysa,
> gördüğün page cache'tir — sistem sağlıklıdır ve daha büyük instance **para israfıdır**. Karar
> vermeden önce daima `available`'a bak. CloudWatch'ın `MemoryUtilization` metriği bu yanılgının
> uzaktan hâlidir: varsayılan hesap çoğu zaman cache'i "kullanımda" sayar; doğrusu için
> `mem_available_percent` benzeri bir metrik kullanılır.

> **🤔 Düşün 4.2** — Bir monitoring paneli "RAM kullanımı %94" diye kırmızı alarm veriyor. Sunucuya
> girip `free -h` çalıştırıyorsun: `used` 2 GB, `buff/cache` 12 GB, `available` 11 GB, total 16 GB.
> Alarm haklı mı? Bir cümleyle ne dersin, ve gerçekten alarm vermesi gereken durum ne olurdu?
>
> *(Cevap: fazın sonunda)*

---
---

# 4.3 Swap ve OOM Killer

## 4.3.1 Swap: RAM taşınca diske sarkan alan `[mekanizma]`

`available` gerçekten sıfıra yaklaştığında ne olur? Çekirdeğin iki kozu vardır. İlki **swap**.

**Swap**, diskte ayrılmış özel bir alandır (bir bölüm veya dosya). RAM dolduğunda ve çekirdek daha
fazla fiziksel sayfaya ihtiyaç duyduğunda, o an **az kullanılan** bellek sayfalarını RAM'den alıp bu
disk alanına yazar (buna "swap out" denir) ve boşalan RAM'i acil ihtiyaç için kullanır. O sayfaya
tekrar dokunulduğunda diskten geri okunur ("swap in").

Swap'ın kritik özelliği: **diske yazılan bellek, disk hızındadır.** Yani RAM'den binlerce kat yavaş.
Bir process'in aktif çalışan sayfaları swap'a inip çıkmaya başlarsa (buna **thrashing** denir), sistem
görünürde "çalışır" ama her bellek erişimi disk erişimine dönüştüğü için felç olmuş gibi yavaşlar.
CPU boşta görünür, ama hiçbir iş ilerlemez.

> **🔧 Makinende gör** 🟢 — swap kullanımını ve aktivitesini oku
>
> ```
> $ free -h
>                total        used        free      buff/cache   available
> Mem:            3.8Gi       3.4Gi       120Mi        280Mi        180Mi
> Swap:           2.0Gi       1.6Gi       0.4Gi
> $ vmstat 1 3
> procs -----------memory---------- ---swap-- ...  ----cpu----
>  r  b   swpd   free   buff  cache   si   so  ...  us sy id wa
>  2  1 1600000 120000  ...          420  380  ...   5  3 12 80
> ```
>
> Buradaki tehlike işareti: `available` sadece 180 MB (RAM baskı altında) **ve** `vmstat`'ın `si`/`so`
> (swap-in / swap-out) sütunları sıfır değil — yani sistem aktif olarak swap'a girip çıkıyor. `wa`
> (I/O wait) 80: CPU'nun çoğu diski (swap'ı) beklemekle geçiyor. Bu, klasik "thrashing" imzasıdır.

> **💡 Cloud bağlantısı — üretimde swap neden tehlikeli:** Çoğu üretim ortamında (özellikle
> Kubernetes) swap **kapatılır**. Sebep: swap, bir uygulamanın "bellek yetmiyor" gerçeğini gizler —
> uygulama ölmek yerine yavaşlar, ve bu yavaşlık teşhisi çok zor bir performans sorununa dönüşür.
> Swap'sız bir sistemde bellek biterse uygulama **hızlıca** OOM ile ölür (net bir sinyal); swap'lı
> sistemde ise saatlerce sürünerek herkesi yanıltır. "Hızlı ve net başarısızlık" çoğu zaman "yavaş ve
> belirsiz"e tercih edilir.

**`swappiness`** — çekirdeğin swap'a ne kadar "istekli" olduğunu ayarlayan 0–100 arası bir değer
(`/proc/sys/vm/swappiness`). Yüksek değer: çekirdek page cache'i korumak için process belleğini daha
erken swap'a atar. Düşük değer (örn. 10): mümkün olduğunca RAM'de tut, son çare olarak swap'la.
Sunucularda genellikle düşük tutulur. Bu fazda sadece **okuyoruz** (`cat /proc/sys/vm/swappiness`);
değiştirmek 🔴 bir işlemdir ve üretim davranışını etkiler.

## 4.3.2 OOM Killer: RAM tükenince kim ölür `[mekanizma]`

Çekirdeğin ikinci — ve son — kozu: swap da bittiyse veya swap yoksa ve RAM tamamen tükendiyse,
çekirdek imkânsız bir durumdadır. Bir process daha fazla bellek istiyor ama verecek hiçbir sayfa yok.
Bu çıkmazda çekirdek sert bir karar verir: **bir process'i öldürüp belleğini geri alır.** Bu
mekanizmaya **OOM Killer** (Out Of Memory Killer — a-u-t-of-memory) denir.

Faz 3.6.2'de cgroup limiti aşılınca "grup içi OOM" olduğunu görmüştük. Buradaki **sistem geneli**
OOM aynı mantığın makine ölçeğidir. Ama kritik soru: çekirdek **kimi** öldürür?

Çekirdek her process'e bir **`oom_score`** verir. Bu skor kabaca "bu process'i öldürürsem ne kadar RAM
kurtulur" ölçüsüne dayanır — yani en çok bellek tutan process en yüksek skoru alır ve ilk öldürülme
adayıdır. (Yönetici bir process'in skorunu `oom_score_adj` ile aşağı/yukarı çekebilir; örneğin kritik
bir veritabanını "en son öldürülecekler" listesine koyabilir.)

> **🔧 Makinende gör** 🟢 — bir OOM olayını `dmesg`'ten oku
>
> ```
> $ dmesg -T | grep -i "killed process"
> [Tue Sep 16 03:14:22] Out of memory: Killed process 2481 (python3)
>   total-vm:8123400kB, anon-rss:7981200kB, file-rss:0kB, oom_score_adj:0
> ```
>
> Çekirdek OOM olayını her zaman kayda geçer. Buradan okunacaklar: **hangi** process öldü (`python3`,
> PID 2481), ne kadar tutuyordu (`anon-rss` ~8 GB — anonim, yani cache olmayan gerçek bellek), ve
> `oom_score_adj:0` (ayarlanmamış). "Uygulamam durup dururken ölmüş" şikâyetinin cevabı neredeyse her
> zaman buradadır — önce `dmesg | grep -i oom`.

> **🤔 Düşün 4.3** — Bir sunucuda gece 3'te veritabanı process'i (en çok RAM tutan) aniden ölmüş,
> restart olmuş. `dmesg`'te "Out of memory: Killed process ... (postgres)" görüyorsun. (1) Veritabanı
> mı bozuldu, yoksa başka bir şey mi RAM'i tüketti? Nasıl anlarsın? (2) Veritabanının bu listede ilk
> ölen olmasını nasıl engellersin?
>
> *(Cevap: fazın sonunda)*

> **💡 Cloud bağlantısı — memory leak senaryosu:** Üretimde en sık OOM sebebi, bir uygulamanın zamanla
> sızdırdığı (leak) bellektir: RSS saatler içinde yavaşça büyür, `available` düşer, sonunda OOM
> Killer devreye girer ve genellikle **en çok RAM tutan** process'i — çoğu zaman sızdıran uygulamanın
> kendisini, bazen de masum ama iri bir komşuyu — öldürür. Doğru teşhis: RSS'in zaman içindeki
> **eğimini** izlemek (CloudWatch/Prometheus). Tek anlık ölçüm yanıltır; leak bir **trend**tir.

---
---

# 4.4 Load Average'ı Doğru Okumak

## 4.4.1 Load = run queue uzunluğu, "CPU meşgul" değil `[mekanizma]`

Faz 3'te load average'ın "R + D process sayısının zaman-ortalaması" olduğunu söylemiştik. Şimdi bu
tanımı sağlamlaştıralım, çünkü Linux dünyasının en yanlış anlaşılan metriğidir.

`uptime` veya `top`'ın üstündeki üç sayı — örneğin `load average: 2.10, 1.80, 1.20` — sırasıyla son
**1, 5 ve 15 dakikanın** ortalamasıdır. Peki neyin ortalaması? "Çalışmak isteyen" process sayısının:

- **çalışan** (R — o an CPU'da) +
- **çalışmaya hazır, sıra bekleyen** (R — CPU boşalsa hemen koşacak) +
- **kesintisiz uykuda** (D — genellikle disk/NFS I/O'da kilitli).

Kritik nokta iki tanedir. **Birincisi:** load bir yüzde değil, bir **sayı**dır ve **core sayısına
göre** okunur. 4 çekirdekli bir makinede load 4.0 = "tam dolu, tam kapasite" (sağlıklı); load 8.0 =
"kapasitenin iki katı iş kuyrukta bekliyor" (sıkışık). Tek çekirdekli makinede load 4.0 = "sistem
boğuluyor". Pratik kural: **load'u `nproc`'a böl** — sonuç 1'in altındaysa rahat, ~1 ise tam dolu,
1'in çok üstündeyse kuyruk birikiyor.

**İkincisi** — ve Linux'a özgü olan: load'a **D-state de sayılır.** Bu yüzden load "CPU meşgul"
demek **değildir**. Diski bekleyen (D) process'ler CPU'yu hiç kullanmadan load'u şişirebilir.

## 4.4.2 Yüksek load + düşük CPU = I/O wait `[uygulama]`

İşte Faz 3'ten kalan tabloyu şimdi çözebiliriz. Yüksek load ama boşta CPU gördüğünde, suçlu CPU değil
**I/O**'dur:

> **Yüksek load + düşük CPU kullanımı → neredeyse her zaman I/O bekleme (D-state process'ler).**

`top`'ın `%Cpu` satırındaki `wa` (I/O wait) yüzdesi bunu doğrudan söyler. `wa` yüksekse, CPU
"boşta oturup diski bekliyor" demektir — iş yok değil, ama iş **disk yüzünden** ilerlemiyor.

> **🔧 Makinende gör** 🟢 — load'u ayrıştır
>
> ```
> $ uptime
>  03:20:11 up 5 days,  load average: 9.40, 8.10, 6.50
> $ nproc
> 4
> $ top -bn1 | grep '%Cpu'
> %Cpu(s):  6.0 us,  3.0 sy,  0.0 ni, 11.0 id, 79.0 wa, ...
> $ ps -eo stat,comm | grep '^D'          # D-state process'leri bul
> D    postgres
> D    kworker/u8:2
> ```
>
> Load 9.4, ama makine 4 çekirdekli — yani kapasitenin ~2.3 katı. CPU'ya bakınca `us`+`sy` sadece %9,
> ama `wa` %79. Yani CPU aslında **boşta**, iş diski beklemekle tıkanmış. `ps ... grep '^D'` diskte
> kilitli process'leri (burada `postgres`) verir. Karar: CPU eklemek (daha büyük instance) bu sorunu
> **çözmez** — daha hızlı disk (IOPS) veya I/O'yu azaltmak gerekir.

> **⚠️ Yaygın yanılgı: "Load 9, CPU yetmiyor, daha çok vCPU alalım."**
> Load'un yüksek olması CPU darboğazı **demek değildir**. Önce `wa`'ya bak: yüksekse sorun disktedir,
> vCPU eklemek para israfıdır ve sorunu çözmez. Load'u her zaman `%Cpu` satırıyla **birlikte** oku:
> yüksek `us`+`sy` → gerçekten CPU-bound; yüksek `wa` → I/O-bound.

---
---

# 4.5 I/O Sezgisi

## 4.5.1 iowait ne anlatır: CPU'nun "diski beklerken" geçen zamanı `[kavram]`

`wa` (iowait) sütununu netleştirelim, çünkü sık yanlış yorumlanır. **iowait, CPU'nun "yapacak başka
işi olmadığı **ve** en az bir process'in disk I/O'sunu beklediği" zamanın yüzdesidir.**

Dikkat edilecek incelik: iowait bir "CPU meşgul" ölçüsü **değildir** — tam tersine, CPU o sırada
**boştadır**. Sadece boşta olmasının sebebi "yapacak iş yok" değil, "iş var ama disk yüzünden
ilerleyemiyor"dur. Yani iowait = **gizli disk darboğazı** göstergesi.

Bunun altında **blocking I/O** kavramı yatar. Bir process bir dosyadan okumak istediğinde `read()`
syscall'ı yapar (Faz 0). Veri diskten gelene kadar process **bloke** olur — D durumuna girer (Faz
3.2), CPU'yu bırakır ve bekler. Disk yavaşsa veya doygunsa bu bekleme uzar; çok sayıda process aynı
anda beklerse load şişer, CPU `wa`'da oturur. (Bunun alternatifi **non-blocking / asenkron I/O**'dur:
process "veri hazır olunca haber ver" der ve beklemeden başka işe geçer — yüksek performanslı
sunucuların, örneğin nginx'in, temeli budur. Bu fazda kavram olarak tanıman yeter.)

## 4.5.2 Disk doygunluğu: tüm sistem "yavaş" ama CPU boşta `[uygulama]`

Bir diskin de bir kapasitesi vardır: saniyede kaç işlem (IOPS) ve kaç MB (throughput) taşıyabileceği.
Bu tavan dolduğunda disk **doygun** (saturated) hâle gelir — her yeni I/O isteği kuyruğa girer ve
bekler. Sonuç: diske dokunan **her** process yavaşlar, sistem geneli "yavaş" hissedilir, ama CPU
grafiği boştadır. Klasik ve kafa karıştırıcı bir tablodur.

Bunu ele veren araç `iostat`'tır:

> **🔧 Makinende gör** 🟢 — disk doygunluğunu ölç
>
> ```
> $ iostat -xz 1 3
> Device   r/s    w/s   rkB/s   wkB/s  await  %util
> nvme0n1  12.0  380.0   480.0  152000  45.20   99.3
> ```
>
> Okunacak iki sütun: **`%util`** — diskin ne kadar zamanının meşgul geçtiği; %100'e yakınsa disk
> doygun demektir. **`await`** — bir I/O isteğinin ortalama tamamlanma süresi (ms); bu sayı
> tırmanıyorsa disk isteklere yetişemiyordur. Burada `%util` 99.3 ve `await` 45 ms — disk boğulmuş.
> `w/s` ve `wkB/s`'ye bakınca yoğun **yazma** var (belki bir log seli veya toplu iş). Bu tablo "CPU
> boşta ama sistem yavaş" bilmecesinin cevabıdır.

> **💡 Cloud bağlantısı:** AWS'te EBS disklerinin IOPS ve throughput tavanı **satın alınan tipe** göre
> bellidir (gp3, io2 vb.). Bir gp3 diski IOPS tavanına dayandığında tam bu tablo çıkar: `%util`
> ~%100, `await` yükselir, uygulama "yavaş"tır ama CPU metriği masumdur. Çözüm CPU değil, diskin
> IOPS/throughput'unu artırmak veya I/O desenini düzeltmektir. CloudWatch'ın `VolumeQueueLength` ve
> `VolumeReadOps`/`WriteOps` metrikleri `iostat`'ın uzaktan hâlidir.

---
---

# 4.6 Darboğaz Triyajı (OS Gözüyle)

## 4.6.1 Dört darboğaz türü ve hangi araç ele verir `[uygulama]`

Bu faz buraya, tek bir pratik refleks için tırmandı: "uygulamam yavaş" dendiğinde, **60 saniyede**
darboğazın dört türünden hangisi olduğunu daraltmak. Dört tür şunlardır:

| Darboğaz türü | İmza | İlk bakılacak araç |
|---|---|---|
| **CPU-bound** | Yüksek load + yüksek `us`+`sy`, düşük `wa` | `top` (%Cpu satırı), `mpstat` |
| **Memory-bound** | `available` düşük, swap `si/so` aktif, OOM riski | `free -h`, `vmstat` (si/so) |
| **I/O-bound** | Yüksek load + düşük CPU + yüksek `wa`, D-state | `iostat -xz` (%util, await) |
| **Network-bound** | CPU/disk boşta ama transfer yavaş, retransmit | `ss`, `iftop`, `sar -n` (Faz 7) |

Buradaki mantık: **her darboğazın farklı bir imzası ve farklı bir aracı vardır.** Yanlış aracı
kullanırsan yanlış teşhis koyarsın — ve cloud'da yanlış teşhis, yanlış "daha büyük instance" kararı
demektir.

## 4.6.2 60 saniyelik triyaj sırası `[uygulama]`

Pratikte izlenen sıra şudur — her adım bir sonrakini eler:

1. **`uptime`** — load'a bak, `nproc`'a böl. Rahat mı, sıkışık mı? (Sorun var mı, ne kadar?)
2. **`top` (veya `vmstat 1`)** — `%Cpu` satırını oku: `us`+`sy` mi yüksek (CPU-bound), yoksa `wa` mı
   yüksek (I/O-bound)?
3. **`free -h`** — `available` düşük mü? `vmstat`'ta `si/so` var mı? (Memory-bound / swap.)
4. **`iostat -xz 1`** — `wa` yüksekse: hangi disk, `%util` ve `await` ne? (I/O darboğazını doğrula.)
5. **`ss` / ağ araçları** — hepsi boştaysa ama uygulama yavaşsa, darboğaz ağda veya uygulamanın
   kendi mantığında olabilir (Faz 7).

> **🔧 Makinende gör** 🟢 — tek ekranda triyaj: `vmstat`
>
> ```
> $ vmstat 1 5
> procs -----------memory---------- ---swap-- -----io---- --system-- ------cpu-----
>  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs  us sy id wa st
>  1  0      0 512000  40000 3800000   0    0    12    40  520 1100  15  4 80  1  0
>  8  0      0 480000  40000 3810000   0    0     8    32 1200 3400  92  6  2  0  0
> ```
>
> `vmstat` dört darboğazı tek satırda özetler: `r` (run queue — CPU baskısı), `si/so` (swap — bellek
> baskısı), `bi/bo` (blok I/O), `wa` (I/O bekleme), `us/sy` (CPU kullanımı). İkinci örnek satırda
> `r=8` ve `us=92` → net **CPU-bound**; `si/so=0` ve `wa=0` → bellek ve disk masum. Teşhis: CPU
> yetmiyor, vCPU eklemek (veya kodu optimize etmek) mantıklı — çünkü bu sefer imza **gerçekten** CPU.

> **🤔 Düşün 4.4** — İki farklı sunucudan `top` üst satırları:
> **A:** `load average: 8.0` · `%Cpu: 95 us, 3 sy, 0 wa`
> **B:** `load average: 8.0` · `%Cpu: 4 us, 2 sy, 82 wa`
> Load ikisinde de aynı (8.0). Hangisi CPU-bound, hangisi I/O-bound? Her biri için doğru "çözüm"
> ne olurdu, ve hangisine daha büyük/hızlı disk almak boşuna olurdu?
>
> *(Cevap: fazın sonunda)*

---
---

# 4.7 Bu Faz Bozulunca — Arıza İmzaları

Bu faz "yavaş" ve "bellek bitti" şikâyetlerinin altını açtı. Aşağıdaki tablo, sahada gördüğün
belirtiyi doğru mekanizmaya bağlar — panik yerine teşhis için.

| Belirti | Muhtemel mekanizma | Nerede anlatıldı | Önce şuna bak |
|---|---|---|---|
| `free`'de RAM %90 dolu, panik | Page cache (geri alınabilir) | 4.2.1 | `available` sütunu — düşük mü? |
| Uygulama durup dururken ölmüş | OOM Killer | 4.3.2 | `dmesg -T \| grep -i oom` |
| Sistem çok yavaş, CPU boşta | Swap thrashing veya I/O doygunluğu | 4.3.1, 4.5.2 | `vmstat` si/so ve `iostat` %util |
| Yüksek load ama CPU %10 | D-state / I/O wait | 4.4.2 | `top` %Cpu satırı `wa`; `ps grep '^D'` |
| "MySQL 2 GB RAM yiyor" (yanlış) | VSZ ile RSS karışması | 4.1.2 | `RES`/`RSS` sütunu, VSZ değil |
| 8 worker'ın RSS toplamı > gerçek | Shared memory / CoW | 4.1.3 | `smem` / PSS |
| Instance büyüttük ama hâlâ yavaş | Yanlış darboğaz teşhisi | 4.6 | Triyaj sırasını baştan çalıştır |
| gp3 diski "yavaş", CPU boşta | EBS IOPS tavanı doygun | 4.5.2 | `iostat await`/`%util`; EBS metrikleri |

> **Bu tablodan çıkan ders:** Bu fazın tüm arızaları **aynı kök yanılgıdan** doğar — bir metriği
> yanlış eksende okumak. "RAM dolu" `available`'a bakmadan okununca panik; "load yüksek" `wa`'ya
> bakmadan okununca yanlış CPU kararı; "bellek çok" VSZ'den okununca abartı. Refleks tek: **bir metriğe
> tepki vermeden önce, onu doğrulayan ikinci metriğe bak.** `used` → `available`, load → `%Cpu`, VSZ →
> RSS, "yavaş" → `vmstat`/`iostat`.

---
---

# Faz 4 — Düşün sorularının cevapları

## Cevap 4.1 — 8 worker × 180 MB neden 1.44 GB etmez

**Soru:** `ps` 8 tane `php-fpm` worker gösteriyor, her birinin RSS'i 180 MB. "8 × 180 = 1.44 GB"
hesabı neden yanlış olabilir?

Çünkü RSS **paylaşılan sayfaları her process'in kendi payına sayar** (4.1.3). 8 worker'ın büyük kısmı
aynı kodu paylaşır: aynı PHP yorumlayıcısı, aynı paylaşılan kütüphaneler (`libc`, PHP eklentileri),
ve `php-fpm`'de sık görüldüğü gibi ana process'ten `fork` edilmiş oldukları için Copy-on-Write ile
paylaşılan başlangıç sayfaları. Bu paylaşılan sayfalar fiziksel RAM'de **tek kopya** durur, ama her
worker'ın RSS'inde tam olarak sayılır.

Yani her worker'ın 180 MB'ının belki 120 MB'ı ortak (tek fiziksel kopya), 60 MB'ı gerçekten özeldir.
Gerçek fiziksel kullanım kabaca: `120 MB (bir kez) + 8 × 60 MB (özel) ≈ 600 MB` — 1.44 GB değil.
Doğru ölçüm için RSS toplamı değil, **PSS** (Proportional Set Size, paylaşılanı process sayısına
bölen ölçü) veya sistem düzeyinde `free`'deki `used` okunur. Kural: **RSS'leri toplayarak toplam RAM
bulma; bu her zaman abartır.**

**İlgili bölüm:** 4.1.3, 4.1.2 · **Devamı:** 4.2 (sistem düzeyi ölçüm), Faz 6 (/proc)

---

## Cevap 4.2 — "%94 RAM" alarmı haklı mı

**Soru:** Panel "RAM %94" alarmı veriyor. `free -h`: `used` 2 GB, `buff/cache` 12 GB, `available` 11
GB, total 16 GB. Alarm haklı mı?

**Hayır, alarm yanlış okuyor.** %94, muhtemelen `(total − free) / total` formülünden geliyor — yani
page cache'i "kullanımda" sayıyor. Ama gerçek tabloya bak: process'lerin gerçekten tuttuğu
(`used`) sadece 2 GB, geri kalan 12 GB **page cache** (geri alınabilir), ve çekirdeğin kendi tahmini
`available` = 11 GB (total'in ~%69'u). Yani sistem baskı altında **değil**; yeni bir uygulama
başlatırsan swap'a inmeden 11 GB verilebilir (4.2.1, 4.2.2).

Doğru cümle: "RAM %94 dolu görünüyor ama bunun 12 GB'ı geri alınabilir page cache; gerçek müsait
bellek 11 GB, sistem rahat." **Gerçekten alarm vermesi gereken durum:** `available` toplamın küçük
bir yüzdesine (örneğin %10'un altına) düştüğünde — çünkü asıl bellek baskısını `available` ölçer,
`used`/`free` değil. Alarmın eşiği `used` değil `available` üzerine kurulmalı.

**İlgili bölüm:** 4.2.1, 4.2.2 · **Devamı:** 4.3 (available bitince ne olur), Faz 11 (metrik alarmları)

---

## Cevap 4.3 — Gece 3'teki OOM: veritabanı mı suçlu

**Soru:** Gece 3'te en çok RAM tutan `postgres` OOM ile öldü. (1) Veritabanı mı bozuldu, yoksa başka
bir şey mi RAM'i tüketti? (2) Bir daha ilk ölen o olmasın diye ne yaparsın?

**(1) OOM Killer'ın öldürdüğü process, çoğu zaman suçlu değildir — sadece en iri olandır.** OOM
Killer `oom_score`'a göre seçim yapar ve bu skor kabaca "en çok RAM kurtaracak" process'i işaret eder
(4.3.2). `postgres` bir veritabanı olarak doğal olarak en çok RAM tutandır, bu yüzden RAM'i **başka
bir şey** tükettiğinde bile kurban `postgres` olur. Gerçek suçluyu bulmak için: OOM anına kadar
`available`'ın **zaman içindeki eğimine** bak (memory leak bir trenddir, 4.3.2 cloud kutusu).
Örneğin gece 3'te çalışan bir cron/batch işi yavaşça RAM'i tüketip `available`'ı sıfıra indirmiş,
son darbeyi OOM Killer `postgres`'e vurmuş olabilir. `dmesg`'te OOM anındaki process listesi ve
o sırada RSS'i büyüyen process bunu ele verir.

**(2) `postgres`'in ilk ölen olmasını engellemek:** `oom_score_adj` ile onun OOM skorunu **aşağı**
çekersin (örneğin `-900`), böylece çekirdek başka adayları önce öldürür. systemd servisi olarak
çalışıyorsa unit dosyasında `OOMScoreAdjust=-900` ayarlanır. Ama bu sadece **belirtiyi** taşır —
asıl çözüm RAM'i tüketen gerçek süreci bulup düzeltmek veya belleği doğru boyutlandırmaktır. "En
kritik process en son ölsün" ayarı bir emniyet kemeridir, kaza sebebini ortadan kaldırmaz.

**İlgili bölüm:** 4.3.2, 4.3.1 · **Devamı:** Faz 5 (systemd unit, OOMScoreAdjust), Faz 11 (trend
izleme)

---

## Cevap 4.4 — Aynı load, iki farklı darboğaz

**Soru:** A: `load 8.0`, `%Cpu: 95 us`. B: `load 8.0`, `%Cpu: 82 wa`. Hangisi CPU-bound, hangisi
I/O-bound?

Load ikisinde de aynı (8.0) ama **load tek başına darboğaz türünü söylemez** — onu `%Cpu` satırıyla
birlikte okuman gerekir (4.4.2). 

**A = CPU-bound.** `us` %95: CPU'nun neredeyse tamamı kullanıcı-uzayı kodunu çalıştırmakla geçiyor.
Load 8, gerçek koşan/kuyrukta bekleyen process'lerden geliyor. Çözüm: daha fazla/hızlı CPU (vCPU
ekle) veya kodu optimize et / paralelize et. Burada daha hızlı disk almak **tamamen boşuna** olurdu —
disk zaten darboğaz değil.

**B = I/O-bound.** `wa` %82: CPU aslında **boşta**, iş diski beklemekle tıkanmış (4.5.1). Load 8,
D-state'te disk bekleyen process'lerden geliyor. Çözüm: daha hızlı disk (IOPS/throughput artır), I/O
desenini düzelt (batch'le, cache'le), veya diske yükü azalt. Burada vCPU eklemek **boşuna** olurdu —
CPU zaten boşta oturuyor.

Dersin özü: iki sunucunun load'u aynı ama **çözümleri taban tabana zıt.** Load'u asla `%Cpu`
satırından ayrı okuma — yüksek `us` → CPU, yüksek `wa` → disk.

**İlgili bölüm:** 4.4.2, 4.6.1 · **Devamı:** 4.6 (triyaj), Faz 11 (troubleshooting)

---
---

# Faz 4 — Sık sorulan sorular

### S1. `free`'de neden `free` sütunu hep küçük? Bir sorun mu?

Hayır, tam tersine sağlık işaretidir. Linux boş RAM'i page cache olarak kullanır (4.2.1) — "boş RAM,
boşa harcanan RAM". Bu yüzden bir süre çalışmış sistemde `free` sütununun küçük, `buff/cache`'in büyük
olması **normaldir**. Bakman gereken sütun `free` değil `available`'dır. `free` sütununun büyük olması
bir sistemin ya yeni boot olduğunu ya da hiç iş yapmadığını gösterir — övünülecek bir şey değil.

### S2. RSS ile PSS arasındaki fark nedir, hangisini ne zaman kullanırım?

**RSS**, process'in RAM'de oturan sayfalarının toplamıdır ama paylaşılan sayfaları **tam** sayar (4.1.3).
**PSS** (Proportional Set Size), paylaşılan sayfaları o sayfayı paylaşan process sayısına **böler** —
yani her process paylaşılandan "kendi payına düşeni" alır. Tek bir process'in kabaca ne kadar tuttuğuna
bakarken RSS yeterli; ama **birçok benzer process'in toplamını** çıkarmaya çalışıyorsan (worker'lar,
fork'lar) RSS abartır, PSS gerçeğe yakındır. PSS'i `smem` veya `/proc/<PID>/smaps_rollup` verir.

### S3. Swap'ı kapatmalı mıyım?

Duruma bağlı. Üretim sunucularında (özellikle Kubernetes node'larında) swap genellikle **kapatılır**:
swap, "bellek yetmiyor" gerçeğini yavaş bir performans sorununa çevirerek gizler (4.3.1 cloud kutusu).
Swap'sız sistem bellek bitince hızlı ve net OOM verir — teşhis kolaydır. Ama masaüstü/geliştirme
makinesinde küçük bir swap, nadir bellek zirvelerini yumuşatabilir. Kural: üretimde "hızlı ve net
başarısızlık" istiyorsan swap'ı küçük tut veya kapat; asla swap'a *sürekli* güvenerek RAM'i az verme.

### S4. `top`'ta `VIRT` (VSZ) neden bu kadar büyük, endişelenmeli miyim?

Hayır. `VIRT`/`VSZ`, process'in ayırdığı **sanal** alandır — dokunulmadıkça fiziksel RAM harcamaz
(4.1.2). JVM, Go, veritabanları başlangıçta onlarca GB sanal alan ayırır; bu bedava gibidir. Endişe
ölçütün her zaman `RES`/`RSS`'tir (gerçekte RAM'de oturan). "VIRT 20 GB!" diye panik yapma; `RES`'e
bak.

### S5. OOM Killer bir process'i öldürdü ama makinede boş RAM vardı, nasıl olur?

İki yaygın sebep. **(1) cgroup limiti:** process bir cgroup içindeyse (container, systemd servisi),
o grubun **kendi** bellek limiti dolunca host'ta bol RAM olsa bile grup içinde OOM olur (Faz 3.6.2).
`dmesg`'te "Memory cgroup out of memory" ibaresi bunu belli eder. **(2) Anlık zirve:** ölçtüğün an
RAM boştu ama saniyeler önce bir process ani bir zirveyle tüm RAM'i isteyip OOM'u tetiklemiş olabilir;
öldükten sonra RAM boşalmış görünür. Her iki durumda da `dmesg -T | grep -i oom` gerçek anı gösterir.

### S6. `iowait` yüksek ama `iostat`'ta disk boşta görünüyor, çelişki mi?

Her zaman değil. `iowait`, CPU'nun "boşta ama en az bir process I/O bekliyor" zamanıdır (4.5.1).
Beklenen I/O yerel disk olmayabilir: **NFS/ağ dosya sistemi**, **EBS gibi ağ üzerinden bağlı disk**,
veya çok sayıda küçük eşzamanlı istek. Yerel `iostat` yerel diski boş gösterip beklemenin ağ
depolamasında olduğunu kaçırabilir. Ayrıca çok çekirdekli makinede `iowait` yüzdesi çekirdekler
arası ortalandığı için yanıltıcı olabilir. Kural: `iowait` yüksekse D-state process'leri bul
(`ps ... grep '^D'`) ve onların **neyi** beklediğine (yerel disk mi, ağ deposu mu) bak.

### S7. `vmstat 1`'deki `r` ve `b` sütunları tam olarak ne?

`vmstat`'ın `procs` başlığı altındaki iki sayı darboğaz triyajının özüdür: **`r`** = çalışan +
çalışmaya hazır (run queue) process sayısı — CPU baskısını gösterir (core sayısını aşarsa CPU
sıkışık). **`b`** = kesintisiz uykuda (blocked, D-state) bekleyen process sayısı — I/O baskısını
gösterir. Yani tek satırda: `r` yüksek → CPU'ya bak; `b` yüksek → diske bak. Load average'ın anlık,
ayrıştırılmış hâli gibidir.

---
---

# Faz 4 — Kendini sına

Aşağıdaki 18 soruyu cevapla. Cevap anahtarı ve puanlama hemen altında. Amaç ezber değil, "kaynağı ne
yiyor" sorusunu modelden cevaplayabilmek.

## A Bölümü — Temel (1–6)

**1.** `ps`/`top` çıktısında `VSZ` (VIRT) ile `RSS` (RES) arasındaki fark nedir, hangisi "gerçekte
kullanılan RAM"dır?

**2.** `free -h` çıktısındaki `buff/cache` sütunu neyi gösterir, ve bu bellek "kullanımda" mı yoksa
"müsait" mi?

**3.** `free -h`'te RAM'in ne kadar dolu olduğuna karar vermek için hangi sütuna bakılır?

**4.** Swap nedir ve neden RAM'den çok daha yavaştır?

**5.** OOM Killer ne zaman devreye girer ve kabaca kimi öldürmeyi seçer?

**6.** Load average 4.0 olan iki makineden biri 2 çekirdekli, diğeri 8 çekirdekli. Hangisi daha çok
baskı altında?

## B Bölümü — Mekanizma (7–12)

**7.** Bir process `malloc(1GB)` yapıyor ama VSZ 1 GB artarken RSS neredeyse hiç artmıyor. Neden?
Bu kavramın adı ne?

**8.** 10 worker'ın her birinin RSS'i 200 MB. Gerçek fiziksel RAM kullanımı neden 2 GB'dan az
olabilir?

**9.** "RAM %90 dolu" görünen sağlıklı bir sistemde, o dolu belleğin çoğu nedir ve yeni bir uygulama
RAM isterse ne olur?

**10.** Load average yüksek (örn. 9) ama `top`'ta `us`+`sy` düşük, `wa` yüksek. Process'ler hangi
durumda ve neyi bekliyor?

**11.** `iowait` (`wa`) tam olarak neyi ölçer — CPU meşgul mü, boşta mı?

**12.** `swappiness` değeri neyi ayarlar; yüksek ve düşük değer ne anlama gelir?

## C Bölümü — Uygulama ve muhakeme (13–18)

**13.** Bir monitoring paneli "RAM %95" kırmızı alarm veriyor. Sunucuda `free -h`: used 2 GB,
buff/cache 13 GB, available 12 GB. Alarm haklı mı, bir cümleyle ne dersin?

**14.** İki sunucunun ikisinde de load 8.0. A: `%Cpu 95 us`. B: `%Cpu 80 wa`. Hangisi CPU-bound,
hangisi I/O-bound, ve hangisine vCPU eklemek boşuna?

**15.** Gece 3'te en çok RAM tutan `postgres` OOM ile öldü. İlk sorman gereken iki şey ne, ve gerçek
suçluyu nasıl ararsın?

**16.** "Uygulamam yavaş" şikâyeti geldi. 60 saniyede darboğazın türünü daraltmak için hangi
komutları hangi sırayla çalıştırırsın?

**17.** `iostat -xz 1` çıktısında bir diskin `%util` değeri 99, `await` 60 ms. Bu ne anlatır ve
çözüm CPU eklemek midir?

**18.** Bir monitoring aracı bir container için "MemoryUtilization %98" diyor ama uygulama sağlıklı
çalışıyor. Bu neden yanıltıcı olabilir, gerçek limiti nereden okursun?

---

## Cevap anahtarı

**1.** `VSZ` = process'in ayırdığı toplam **sanal** alan (fiziksel RAM ile doğrudan ilgisi yok);
`RSS` = o sanal sayfalardan **gerçekten fiziksel RAM'de** oturanlar. "Gerçekte kullanılan RAM" =
**RSS**. · *4.1.2*

**2.** `buff/cache` = page cache + buffer'lar, yani diskten okunanların RAM'deki kopyası. Hem
"kullanımda" görünür hem **geri alınabilir** — bir process RAM isterse çekirdek anında bırakır. · *4.2.1*

**3.** **`available`** sütunu — çekirdeğin "swap'a inmeden yeni uygulamaya ne kadar RAM verebilirim"
tahmini. `used`/`free` değil, gerçek baskıyı bu ölçer. · *4.2.2*

**4.** Swap, RAM taştığında az kullanılan bellek sayfalarının sarkıtıldığı **disk** alanıdır. Disk
RAM'den binlerce kat yavaş olduğu için, aktif bellek swap'a girip çıkmaya başlarsa (thrashing) sistem
felç olur. · *4.3.1*

**5.** RAM (ve swap) tamamen tükenip çekirdek verecek sayfa bulamayınca. Kabaca en **yüksek
`oom_score`**'a sahip — yani en çok RAM tutan — process'i öldürüp belleğini geri alır. · *4.3.2*

**6.** **2 çekirdekli** olan. Load core sayısına göre okunur: 2 çekirdekte load 4.0 = kapasitenin 2
katı (sıkışık); 8 çekirdekte load 4.0 = %50 kullanım (rahat). Kural: load / `nproc`. · *4.4.1*

**7.** Çünkü Linux **lazy allocation** (tembel ayırma) yapar: `malloc` sadece sanal alan ayırır (VSZ
artar), ama o sayfalara **dokunulana** kadar çekirdek fiziksel RAM bağlamaz (RSS artmaz). Fiziksel
sayfa ancak yazıldığında (page fault) bağlanır. · *4.1.2*

**8.** Çünkü RSS **paylaşılan** sayfaları (shared libraries, `fork`'tan gelen CoW sayfaları) her
process'in payına tam sayar, ama fiziksel RAM'de tek kopya vardır. Gerçek kullanım için RSS'leri
toplama; PSS'e veya `free`'deki `used`'a bak. · *4.1.3*

**9.** Çoğu **page cache** (diskten okunanların geri alınabilir kopyası). Yeni bir uygulama RAM
isterse çekirdek page cache'ten anında bir kısmını bırakıp verir — çünkü orası diskte zaten var. · *4.2.1*

**10.** Büyük olasılıkla **D (kesintisiz uyku)** durumunda, **disk/ağ I/O**'sunu bekliyorlar. Yüksek
load'u bu D-state process'ler şişiriyor; CPU aslında boşta (`wa` yüksek). · *4.4.2*

**11.** `iowait`, CPU'nun **boşta** olduğu **ama** en az bir process'in I/O beklediği zamanın
yüzdesidir. CPU meşgul değil — boşta ama iş disk yüzünden ilerlemiyor. "Gizli disk darboğazı"
göstergesi. · *4.5.1*

**12.** `swappiness` (0–100), çekirdeğin swap'a ne kadar istekli olduğunu ayarlar. Yüksek = page
cache'i korumak için process belleğini erken swap'a atar; düşük = mümkün olduğunca RAM'de tut, son
çare swap. Sunucularda genelde düşük. · *4.3.1*

**13.** Hayır, alarm yanlış okuyor. Gerçek `used` sadece 2 GB; 13 GB **page cache** (geri
alınabilir); `available` 12 GB → sistem rahat. Doğru: "%95 dolu görünüyor ama çoğu geri alınabilir
cache, gerçek müsait 12 GB." Alarm `available` üzerine kurulmalı. · *4.2.2*

**14.** **A = CPU-bound** (`us` %95, CPU dolu); **B = I/O-bound** (`wa` %80, CPU boşta, disk
bekliyor). **B'ye vCPU eklemek boşuna** — CPU zaten boşta; ona hızlı disk gerekir. A'ya ise disk
almak boşuna. · *4.4.2, 4.6.1*

**15.** (1) "Suçlu gerçekten postgres mi, yoksa başka bir şey mi RAM'i tüketip onu en iri kurban mı
yaptı?" (2) "OOM anına kadar `available` nasıl bir eğimle düştü?" Gerçek suçlu için RSS'i zamanla
büyüyen process'i / o saatte çalışan işi ara (`dmesg` + trend). Koruma: `postgres`'e
`OOMScoreAdjust=-900`. · *4.3.2*

**16.** (1) `uptime` — load / `nproc`; (2) `top`/`vmstat 1` — `us+sy` mi `wa` mı yüksek; (3) `free -h`
— `available` düşük mü, `si/so` var mı; (4) `iostat -xz 1` — hangi disk `%util`/`await`; (5) ağ
araçları. Her adım bir darboğaz türünü eler. · *4.6.2*

**17.** Disk **doygun** (%util ~100) ve her I/O ortalama 60 ms bekliyor (await yüksek) — disk
isteklere yetişemiyor. Çözüm CPU **değil**; daha hızlı disk (IOPS/throughput), I/O desenini düzeltmek
veya yükü azaltmak. · *4.5.2*

**18.** Çünkü container içindeki bellek metrikleri çoğu zaman host değerini veya cache'i "kullanımda"
sayar; ayrıca `free`/`top` container içinde host'u gösterebilir (Faz 3.6). Gerçek limiti cgroup'tan
oku: cgroup v2'de `/sys/fs/cgroup/memory.max` ve `memory.current`. · *4.2.2, Faz 3.6.4*

---

## Puanlama

| Doğru sayısı | Değerlendirme |
|---|---|
| 16–18 | "Kaynağı ne yiyor" sorusunu modelle cevaplıyorsun. Ara Sınav 2'ye geç. |
| 12–15 | Sağlam. Kaçırdığın soruların bölümlerine bir tur dön. |
| 8–11 | Temel oturmuş ama metrikleri karıştırıyorsun. Aşağıdaki tabloyu kullan. |
| 0–7 | Fazı baştan, 🔧 kutularını (özellikle 4.2, 4.4, 4.6) makinende çalıştırarak tekrar et. |

**Hangi soruyu kaçırırsan hangi bölüme dön:**

| Kaçırdığın soru | Dön |
|---|---|
| 1, 7 | 4.1.2 (RSS vs VSZ, lazy allocation) |
| 8, 18 | 4.1.3 (shared memory, RSS toplamı) |
| 2, 3, 9, 13 | 4.2 (page cache, available sütunu) |
| 4, 5, 12, 15 | 4.3 (swap, swappiness, OOM Killer) |
| 6, 10 | 4.4 (load average, core sayısı, D-state) |
| 11, 17 | 4.5 (iowait, disk doygunluğu, iostat) |
| 14, 16 | 4.6 (darboğaz triyajı, dört tür) |

---
---

# Faz 4 — Kapanış ve Faz 5'e Köprü

## Bu fazdan ne taşıyorsun

Faz 3 sana "sistem ne yapıyor" sorusunu process ekseninde verdi. Faz 4 aynı soruyu **kaynak** eksenine
taşıdı: bellek ve I/O. Artık bir metriğe tek başına tepki vermiyorsun — her rakamı doğrulayan ikinci
rakama bakıyorsun. Yanında dört refleks götürüyorsun:

1. **Bellek gerçeği RSS'te, `available`'da.** `VSZ` söz verilen, `RSS` kullanılan; RSS'leri toplama
   (shared memory). "RAM dolu" `used`/`free` değil `available` ile ölçülür — page cache geri
   alınabilir.
2. **Bellek bitince swap → OOM.** Swap yavaştır ve gerçeği gizler; OOM Killer en iri process'i öldürür
   (suçluyu değil). `dmesg | grep oom` ilk bakılacak yerdir.
3. **Load'u `%Cpu` ile birlikte oku.** Load core sayısına bölünür; yüksek `us` → CPU-bound, yüksek
   `wa` → I/O-bound. Aynı load, zıt çözümler.
4. **Darboğaz triyajı.** Dört tür (CPU/memory/I/O/network), her birinin imzası ve aracı ayrı.
   `uptime → top → free → iostat` sırası 60 saniyede türü daraltır.

## Faz 5 bunun neresine bağlanıyor

Faz 3 ve 4 birlikte "çalışan bir makine ne yapıyor, kaynağı ne yiyor" sorusunu tam kapladı — bu
ikisi **Ara Sınav 2**'nin konusudur. Sıradaki büyük soru ise: **bu makine baştan nasıl çalışan bir
sisteme geldi?** Faz 5 (Boot, Init ve systemd) makinenin yaşam döngüsünü açar ve bu fazın bıraktığı
iplikleri toplar:

| Faz 4'te öğrendiğin | Faz 5'te üstüne konacak |
|---|---|
| PID 1 = systemd, ağacın kökü (Faz 3'ten) | systemd'nin bir "başlatıcı"dan fazlası: unit, servis, hedef |
| OOM'da `OOMScoreAdjust` ile process korunur | systemd unit dosyasında kaynak ve OOM ayarları |
| Bir servis nasıl "çalışır durumda" tutulur | systemd servis yaşam döngüsü, restart politikaları |
| `dmesg` çekirdek olaylarını gösterir | `journalctl` — systemd'nin merkezî log defteri |
| Bir process'i doğru başlatmak/durdurmak | `systemctl start/stop/enable`, boot sırası |

> **Faz 4 çıktısı — devam etmeden önce kendine sor:**
> Bir geliştirici "prod sunucusu çok yavaş, RAM %97 dolu, acil daha büyük instance alalım" diye yazdı.
> Sunucuya girdin:
>
> ```
> $ free -h
>                total   used   free   buff/cache   available
> Mem:            16Gi   2.4Gi  0.3Gi     13Gi         12Gi
> $ uptime
>  ... load average: 11.20, 10.80, 9.90     # (4 vCPU makine)
> $ top -bn1 | grep %Cpu
> %Cpu(s):  5.0 us, 2.0 sy, 0.0 ni, 8.0 id, 84.0 wa, ...
> ```
>
> 1. "RAM %97 dolu" doğru bir gerekçe mi? Bir cümleyle geliştiriciye ne yazarsın?
> 2. Asıl darboğaz ne — CPU, bellek, yoksa disk? Hangi iki rakamdan anladın?
> 3. "Daha büyük instance" bu sorunu çözer mi? Değilse doğru adım ne?
>
> Bu üçünü tereddütsüz cevaplayabiliyorsan, bellek ve I/O sezgisi oturmuş demektir — Ara Sınav 2'ye
> hazırsın.
>
> **Lab 4 fikri:** `free -h` açık dururken `cat büyük_bir_dosya > /dev/null` çalıştırıp `buff/cache`
> ile `available`'ın nasıl değiştiğini izle (used değişmez!). `vmstat 1` ile bir dosya kopyalarken
> `bi/bo` (blok I/O) ve `wa` sütunlarını canlı gör. Makinen varsa `stress-ng --vm 2 --vm-bytes 1G
> --timeout 20s` ile yapay bellek baskısı bindirip `si/so`'nun (swap) uyanmasını izle. `uptime`
> load'unu her zaman `nproc` ile kıyasla.

---

> **Navigasyon:** [◀ Faz 3 — Process ve Kaynak](Faz_3_Process_ve_Kaynak.md) · **Faz 4** · [Ara Sınav 2 ▶](Ara_Sinav_2.md)


