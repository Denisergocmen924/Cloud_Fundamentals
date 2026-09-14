# Faz 3 — Depolama (Kalıcı Hafıza)

> **Navigasyon:** [◀ Faz 2 — Bellek Hiyerarşisi](Faz_2_Bellek_Hiyerarsisi.md) · **Faz 3** · [Faz 4 — Sistem Bus'ları ve I/O ▶](Faz_4_Bus_ve_IO.md)

---

## Nereden geliyoruz

Faz 2 bir cümleyle bitti: **DRAM uçucudur.** Güç kesilince her şey kaybolur.

Bu, depolamanın var olma sebebidir. Ama Faz 3'ün asıl konusu "veri nereye yazılır" değil —
**veriyi kalıcı yapmanın bedeli nedir** sorusudur.

Faz 2'de kurduğun her prensip burada tekrar edecek, sadece ölçek büyüyecek:

| Faz 2'deki prensip | Faz 3'teki karşılığı |
|---|---|
| Hiyerarşi: hız ↔ kapasite takası | NVMe → SATA SSD → HDD → nesne depolama |
| Blok halinde transfer (64 byte cache satırı) | Blok halinde transfer (4 KiB disk bloğu) |
| Sıralı erişim rastgeleden hızlıdır (2.3.6) | Aynı gerçek, **çok daha keskin** — HDD'de 100 kat |
| Yerellik bahsi (2.1.3) | Page cache, disk cache, CDN |
| Latency ≠ throughput (2.1.2) | **IOPS ≠ throughput** — bu fazın merkezi ayrımı |

> **Bu fazın pratik değeri yüksektir.** Faz 0–2 "derin neden" hattıydı. Faz 3 ise
> doğrudan haftalık kararlara dokunur: hangi EBS tipini seçeceğin, veritabanının neden
> yavaş olduğu, yedekleme stratejinin neden pahalı olduğu — hepsi burada.

---

## Bu fazın sonunda

- Bir diskin neden yavaş olduğunu **fiziksel sebebiyle** açıklayabileceksin
- SSD'nin HDD'nin hızlı hâli **olmadığını**, tamamen farklı bir davranış modeli olduğunu
  anlayacaksın
- IOPS, throughput ve latency'yi birbirine karıştırmadan kullanabileceksin
- Bir iş yüküne bakıp "bu IOPS sınırlı mı, throughput sınırlı mı?" sorusunu
  cevaplayabileceksin
- EBS tipi seçimini ezberden değil **iş yükünün şeklinden** türetebileceksin
- Depolama kaynaklı performans problemlerinin imzalarını tanıyacaksın

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden önemli |
|---|---|---|---|
| 3.1 | HDD | `[kavram]` / `[mekanizma]` | Mekanik gecikmenin sezgisi — tüm depolama mantığının temeli |
| 3.2 | SSD | `[kavram]` / `[mekanizma]` | Bugünün gerçeği; tuhaf davranışlarının sebebi |
| 3.3 | SATA vs NVMe | `[mekanizma]` | Arayüzün diskin kendisi kadar belirleyici olması |
| 3.4 | **IOPS / Throughput / Latency** | `[mekanizma]` / `[uygulama]` | **Fazın kalbi — EBS seçiminin tamamı buraya dayanır** |
| 3.5 | RAID | `[kavram]` / `[uygulama]` | Dayanıklılık ve performansın birlikte tasarlanması |

---

# 3.1 HDD — dönen disk

HDD (Hard Disk Drive) cloud'da hızla azalıyor. Yine de öğreniyoruz, çünkü **HDD'nin
fiziği, depolama hakkındaki tüm sezgilerinin kaynağıdır.** SSD'nin ne kadar tuhaf bir
şey olduğunu ancak HDD'yi bilirsen anlarsın.

## 3.1.1 Mekanik yapı `[kavram]`

```
        ┌──────────────────────────────┐
        │   ╭───────────────────╮      │
        │   │  ◎ ← spindle      │      │   Plaka (platter): manyetik kaplı disk
        │   │    ╱╲             │      │   Spindle: plakaları döndüren mil
        │   │   ╱  ╲ ← track    │      │   Track: eşmerkezli daireler
        │   ╰──╱────╲───────────╯      │   Sector: track'in parçası (512 B / 4 KiB)
        │     ╱      ╲                 │   Kafa (head): okuma-yazma ucu
        │  ═══╪═══════╪═══ ← kafa kolu │   Aktüatör: kafayı hareket ettiren kol
        └──────────────────────────────┘
```

![Şekil 3.1 — HDD'nin mekanik bileşenleri ve bir okumanın izlediği yol](diagrams/hw-3-01-hdd-anatomy.png)
*Şekil 3.1 — Plaka, track, sektör ve kafa kolu; bir okumanın seek + rotasyon + transfer
aşamaları.*

**Anahtar gerçek:** Bu bir **makinedir.** İçinde dakikada 7.200 veya 15.000 devir dönen
bir motor ve saniyede yüzlerce kez ileri geri hareket eden bir kol vardır.

Faz 0–2'de konuştuğumuz her şey elektron hızındaydı. Burada **atalet, sürtünme ve kütle**
devreye giriyor. Fark bu kadar temeldir.

## 3.1.2 Bir okumanın maliyeti — seek time `[mekanizma]`

Bir veri bloğunu okumak üç aşamalıdır:

| Aşama | Ne oluyor | Süre (7200 RPM) |
|---|---|---|
| **1. Seek time** | Kafa doğru track'e hareket eder | **~4–9 ms** |
| **2. Rotational latency** | Doğru sektörün kafanın altına gelmesi beklenir | **~4,2 ms** (ort.) |
| **3. Transfer time** | Veri okunur | ~0,01 ms (4 KiB için) |
| | **Toplam** | **~8–13 ms** |

**Rotasyon gecikmesini kendin hesapla:**
```
7200 RPM = 7200 / 60 = 120 devir/saniye
Bir tam devir = 1/120 = 8,33 ms
Ortalama bekleme = yarım devir = 4,17 ms
```

> **Dikkat et:** Toplam sürenin **%99,9'u veriyi okumak için değil, veriye ulaşmak için**
> harcanıyor. Transfer 0,01 ms, ulaşma 8+ ms.
>
> **Bu tek gözlem, depolama hakkında bilmen gereken her şeyin tohumudur.** Veriye ulaşmak
> pahalı, veriyi okumak ucuzsa — o zaman **bir kere ulaşıp çok veri okumak** doğru
> stratejidir. Blok boyutları, sıralı erişim üstünlüğü, read-ahead, veritabanı sayfa
> tasarımı, log-structured depolama: hepsi bu tek gerçeğin sonucudur.

## 3.1.3 Sıralı vs rastgele — sayılarla `[mekanizma]`

**Senaryo A — 1 GB veriyi sıralı oku:**
```
1 seek (8 ms) + sürekli transfer (150 MB/s)
= 8 ms + 6.667 ms ≈ 6,7 saniye
Etkin hız: ~150 MB/s
```

**Senaryo B — 1 GB veriyi 4 KiB'lık rastgele bloklar hâlinde oku:**
```
1 GB / 4 KiB = 262.144 blok
Her blok: 8 ms seek + 4,2 ms rotasyon ≈ 12 ms
262.144 × 12 ms = 3.145 saniye ≈ 52 DAKİKA
Etkin hız: ~0,33 MB/s
```

```
Sıralı  :  6,7 saniye     ████
Rastgele:  52 dakika      ████████████████████████████... (466 kat)
```

**Aynı disk. Aynı 1 GB. 466 kat fark.**

> **Faz 2.3.6'daki 10 katlık cache farkını hatırla.** Aynı prensip — erişim deseni —
> burada **466 kat** olarak karşına çıkıyor. Ölçek büyüdükçe erişim deseninin önemi
> büyüyor. Faz 5'te ağa geldiğimizde bu oran daha da büyüyecek.

### HDD'nin IOPS tavanı

```
Bir I/O işlemi ≈ 12 ms
IOPS = 1000 ms / 12 ms ≈ 80–120 IOPS
```

**Bu sayı 30 yıldır neredeyse değişmedi.** Kapasiteler 100 MB'tan 20 TB'a çıktı (200.000
kat), sıralı hız 1 MB/s'ten 250 MB/s'e çıktı (250 kat), ama **rastgele IOPS 80'den 120'ye
çıktı (1,5 kat).**

> **Sebep:** Kapasite ve sıralı hız *manyetik yoğunluğa* bağlıdır — mühendislikle
> artırılabilir. Rastgele IOPS ise *mekanik harekete* bağlıdır — kolun hareket süresi ve
> plakanın dönüş hızı fiziksel sınırlara dayanmıştır. 15.000 RPM'in üstüne çıkmak
> titreşim ve ısı nedeniyle pratik değildir.
>
> Faz 2.4.2'deki DDR hikâyesiyle aynı desen: **bir boyut katlanırken diğeri sabit kalır.**
> Ve her seferinde sabit kalan boyut *gecikme* oluyor.

## 3.1.4 HDD cloud'da nerede kaldı? `[uygulama]`

HDD ölmedi, ama **rolü daraldı.** Bugün tek bir şeyde rakipsiz: **GB başına maliyet.**

| Kullanım | Uygun mu | Neden |
|---|---|---|
| Veritabanı (OLTP) | ❌ Asla | Rastgele I/O ağırlıklı, 120 IOPS yetersiz |
| İşletim sistemi diski | ❌ Hayır | Boot ve çalışma zamanı rastgele okuma yapar |
| Büyük dosya arşivi, log birikimi | ✅ İyi | Sıralı erişim, kapasite odaklı |
| Yedekleme hedefi | ✅ İyi | Sıralı yazma, nadiren okunur |
| Büyük veri taraması (sıralı) | ✅ Makul | Throughput yeterli, maliyet düşük |

**AWS karşılığı:** `st1` (throughput optimized HDD) ve `sc1` (cold HDD). Bunlar **sıralı
iş yükleri için tasarlanmıştır** ve AWS dokümantasyonu bunu açıkça söyler: küçük rastgele
I/O'da performansları felakettir.

> **Ve bunun bir tuzağı var:** `st1` fiyat listesinde `gp3`'ten ucuz görünür. Bir
> veritabanını maliyet tasarrufu için `st1`'e taşımak, kâğıt üzerinde mantıklı ama
> pratikte **sistemi kullanılamaz hâle getirir.** Faz 3.4.5'te bu seçimi sistematik
> hâle getireceğiz.

---

# 3.2 SSD — hareketli parçası olmayan depolama

## 3.2.1 NAND Flash nasıl çalışır `[kavram]`

Faz 0.2.1'deki MOSFET'i hatırla: kapıya (gate) voltaj verilince kanal iletken oluyordu.

NAND Flash hücresi, bu transistöre **ikinci bir kapı** ekler: **yüzer kapı (floating
gate)** — yalıtkanla çevrili, hiçbir yere bağlı olmayan bir ada.

```
      Kontrol kapısı  ═══════════
                      ░░░░░░░░░░  ← yalıtkan
      Yüzer kapı      ▓▓▓▓▓▓▓▓▓▓  ← elektronlar BURADA hapsolur
                      ░░░░░░░░░░  ← yalıtkan
      Kanal           ──────────
```

Yüzer kapıya elektron hapsedilirse, transistörün açılma eşiği değişir. **Elektronlar
yalıtkanla çevrili olduğu için güç kesilse bile orada kalır.**

**İşte kalıcılık budur.** Manyetik yönelim değil, hapsolmuş elektronlar.

> **Ve buradan SSD'nin tüm tuhaflıkları doğar.** Elektronu yalıtkanın içinden geçirip
> hapsetmek zor bir iştir: yüksek voltaj gerekir ve her seferinde yalıtkan biraz yıpranır.
> Silmek daha da zordur. Okumak ise kolaydır.
>
> **Bu asimetri — oku ucuz, yaz pahalı, sil çok pahalı — SSD davranışının tamamını
> açıklar.**

## 3.2.2 Sayfa ve blok — SSD'nin garip asimetrisi `[mekanizma]`

**Bu bölüm SSD hakkında bilmen gereken en önemli mekanizmadır.**

SSD'de üç farklı işlem, üç farklı granülerlikte çalışır:

| İşlem | Birim | Boyut | Süre |
|---|---|---|---|
| **Okuma** | Sayfa (page) | 4–16 KiB | ~50–100 μs |
| **Yazma (program)** | Sayfa | 4–16 KiB | ~200–800 μs |
| **Silme (erase)** | **Blok** | **128–256 sayfa (1–4 MiB)** | **~2.000–5.000 μs** |

**Kritik kural:** Bir sayfaya yazabilmek için o sayfanın **boş** olması gerekir. Ve
silme **sayfa bazında yapılamaz** — sadece **tüm blok** silinebilir.

```
Bir bloktaki tek bir sayfayı güncellemek istiyorsun:

  Blok: [S1][S2][S3][S4] ... [S256]
                  ↑ sadece S3'ü değiştirmek istiyorum

  Naif yol:
    1. Tüm bloğu geçici yere oku        (256 sayfa okuma)
    2. Tüm bloğu sil                    (5 ms!)
    3. S3'ü değiştir
    4. Tüm bloğu geri yaz               (256 sayfa yazma)

  → 4 KiB güncellemek için 1 MiB oku + 1 MiB yaz + 1 silme
```

Buna **write amplification** (yazma büyütmesi) denir. Ve eğer SSD bu naif yolu izleseydi
kullanılamaz olurdu.

## 3.2.3 FTL, wear leveling ve TRIM `[kavram]`

SSD'nin içinde bir **denetleyici** (controller) ve üzerinde çalışan bir yazılım katmanı
vardır: **FTL (Flash Translation Layer).**

FTL'nin yaptığı iş, Faz 2.5'teki sanal belleğin **birebir aynısıdır:**

| Sanal bellek (2.5) | FTL |
|---|---|
| Sanal adres → fiziksel adres | Mantıksal blok (LBA) → fiziksel sayfa |
| Sayfa tablosu | Eşleme tablosu |
| MMU | SSD denetleyicisi |

**FTL'nin çözümü: yerinde güncelleme yapma.**

```
S3'ü güncellemek istiyorsun:
  1. BAŞKA bir yerdeki boş sayfaya yeni veriyi yaz
  2. Eşleme tablosunu güncelle: "LBA 3 artık şurada"
  3. Eski sayfayı "geçersiz" işaretle
  4. Silme işini SONRAYA ertele (arka planda, boşken)

→ Yazma anında maliyet: 1 sayfa. Silme maliyeti amortize edildi.
```

Bu arka plan temizliğine **garbage collection** denir.

**Wear leveling.** Her hücrenin sınırlı sayıda silme döngüsü vardır (P/E cycle). FTL,
yazmaları tüm çipe **eşit dağıtır** ki bazı bloklar erken ölmesin. Sen aynı dosyayı
1000 kez üzerine yazsan bile, fiziksel olarak her seferinde başka bir yere yazılır.

**TRIM.** İşletim sistemi bir dosyayı sildiğinde SSD bunu bilmez — onun için o sayfalar
hâlâ "dolu"dur ve garbage collection sırasında boşuna kopyalanır. `TRIM` komutu SSD'ye
"bu bloklar artık gereksiz" der.

> **TRIM olmadan SSD zamanla yavaşlar** — bu, eski SSD'lerin "yaşlandıkça yavaşlıyor"
> şikâyetinin sebebiydi. Modern sistemlerde otomatiktir (`fstrim.timer`), ama bir sanal
> makine katmanı veya yanlış yapılandırılmış bir RAID katmanı TRIM'i engelleyebilir.

## 3.2.4 Hücre tipleri — hız/dayanıklılık/maliyet üçgeni `[mekanizma]`

Bir hücreye kaç bit sığdırılacağı bir tasarım seçimidir. Daha çok bit = daha çok voltaj
seviyesi ayırt etmek demektir.

| Tip | Bit/hücre | Voltaj seviyesi | P/E döngüsü | Hız | Maliyet |
|---|---|---|---|---|---|
| **SLC** | 1 | 2 | ~100.000 | En hızlı | En pahalı |
| **MLC** | 2 | 4 | ~10.000 | Hızlı | Pahalı |
| **TLC** | 3 | 8 | ~3.000 | Orta | Makul |
| **QLC** | 4 | **16** | **~1.000** | Yavaş | En ucuz |

**Neden daha çok bit = daha kısa ömür ve daha yavaş?**

QLC'de 16 farklı voltaj seviyesini ayırt etmen gerekir. Seviyeler arası aralık daralır,
yani:
- Okuma daha hassas ölçüm gerektirir → **yavaş**
- Yazma daha hassas yükleme gerektirir → **daha yavaş**
- Yalıtkandaki küçük bir yıpranma bile seviyeleri karıştırır → **daha az dayanıklı**
- Elektron sızıntısı daha çabuk hata verir → **veri saklama süresi kısalır**

> **Faz 0.1.1'deki gürültü payı (noise margin) argümanının birebir tekrarı.** İkili
> sistemi seçmemizin sebebi, iki seviye arasındaki mesafenin geniş olmasıydı. QLC 16
> seviye kullanarak o mesafeyi daraltıyor — kapasite kazanıyor, güvenilirlik kaybediyor.
> **Aynı fizik, 60 yıl sonra, aynı takas.**

**Cloud bağlantısı:** Bulut sağlayıcıları donanım detayını ilan etmez, ama garanti edilen
performans ve dayanıklılık rakamları bu seçimleri yansıtır. Senin işin hücre tipini
seçmek değil — **hangi performans profilini satın aldığını anlamaktır** (3.4.5).

![Şekil 3.2 — SSD'de sayfa/blok asimetrisi ve FTL'nin yerinde-olmayan yazma stratejisi](diagrams/hw-3-02-ssd-pages-blocks.png)
*Şekil 3.2 — Okuma ve yazma sayfa bazında, silme blok bazında; FTL eski sayfayı geçersiz
işaretleyip yeni sayfaya yazıyor.*

## 3.2.5 SSD'nin performans tuhaflıkları `[uygulama]`

HDD sezginle SSD'ye yaklaşırsan yanılırsın. Üç önemli fark:

**1. Sıralı ve rastgele arasındaki fark küçüldü — ama kaybolmadı.**
```
HDD:  sıralı 150 MB/s, rastgele 0,3 MB/s    → 466 kat
NVMe: sıralı 3500 MB/s, rastgele 400 MB/s   → ~9 kat
```
Fark hâlâ var (blok yönetimi, kanal paralelliği, FTL eşleme maliyeti) ama artık
belirleyici değil. **Bu, veritabanı tasarımından dosya sistemine kadar birçok kabulü
değiştirdi.**

**2. Kuyruk derinliği (queue depth) performansı belirler.**
SSD'nin içinde onlarca paralel NAND kanalı vardır. Tek bir istek gönderirsen bunların
biri çalışır. **SSD'nin ilan edilen IOPS'una ulaşmak için aynı anda çok sayıda istek
göndermen gerekir.**

```
QD=1   (tek thread, senkron I/O):  ~15.000 IOPS
QD=32  (çok thread veya async):   ~400.000 IOPS
```

> **Bu, üretimde çok sık karşılaşılan bir sürprizin sebebidir:** "Disk 500K IOPS
> destekliyor ama uygulamam 15K alıyor." Sorun disk değil — uygulama **yeterince paralel
> I/O üretmiyor.** Çözüm daha hızlı disk değil, daha çok eşzamanlılık (async I/O, thread
> havuzu, daha derin kuyruk).

**3. Boş alan performansı etkiler.**
SSD %90 doluysa, garbage collection için serbest blok bulmak zorlaşır. Write amplification
artar, yazma gecikmesi yükselir ve **değişkenleşir.** Üretimde SSD'leri %80'in altında
tutmak yaygın bir pratiktir.

> **🤔 Düşün 3.1**
> Bir log toplama sistemi yazıyorsun. Saniyede 50.000 küçük satır (~200 byte) geliyor.
> İki tasarım düşünüyorsun:
>
> - **A:** Her satırı geldiği anda diske yaz (`write()` + `fsync()`)
> - **B:** Satırları bellekte 4 MiB'lık tampona biriktir, dolunca tek seferde yaz
>
> a) SSD üzerinde bu iki tasarımın performans farkı ne olur? Neden?
> b) B'nin bedeli nedir?
> *(Cevap: fazın sonunda)*

---
# 3.3 Arayüzler — SATA ve NVMe

## 3.3.1 Problem: hızlı disk, yavaş yol `[mekanizma]`

SSD icat edildiğinde, onu bilgisayara bağlamak için mevcut arayüz kullanıldı: **SATA.**

Ama SATA, **HDD için tasarlanmıştı.** Ve HDD'nin ihtiyaçları tamamen farklıydı:

| SATA'nın varsayımı | SSD'nin gerçeği |
|---|---|
| Cihaz zaten yavaş (120 IOPS), protokol maliyeti önemsiz | Cihaz çok hızlı, protokol maliyeti **baskın** |
| Tek kuyruk yeterli (kafa zaten tek) | Onlarca paralel kanal var |
| Kuyruk derinliği 32 fazlasıyla yeter | 65.000 istek paralel işlenebilir |
| 6 Gbps bant genişliği bol | 6 Gbps **tavan oluyor** |

**Sonuç:** SATA SSD, SSD'nin potansiyelinin belki %15'ini kullanabiliyordu. Disk hazırdı,
yol dardı.

## 3.3.2 NVMe — SSD için tasarlanmış protokol `[mekanizma]`

NVMe (Non-Volatile Memory Express), SSD'nin gerçek doğasına göre sıfırdan tasarlandı ve
**PCIe** üzerinden çalışır (Faz 4.2'de PCIe'yi ayrıntılı göreceğiz).

| Boyut | SATA (AHCI) | NVMe |
|---|---|---|
| Fiziksel yol | SATA kablosu, 6 Gbps | **PCIe şeritleri** (x4 ≈ 8–16 GB/s) |
| Kuyruk sayısı | **1** | **65.535** |
| Kuyruk derinliği | 32 | **65.536** |
| Komut başına CPU maliyeti | Yüksek (eski, kesme ağırlıklı) | **Düşük** (Faz 4.3'teki DMA'yı verimli kullanır) |
| Tipik gecikme | ~100 μs | **~20 μs** |
| Tipik IOPS | ~90.000 | **~1.000.000+** |

**En önemli fark kuyruk mimarisidir.** SATA'da tek bir kuyruk vardır — tüm CPU
çekirdekleri onun için yarışır ve kilitlenme (lock contention) yaşanır. NVMe'de
**her çekirdeğe kendi kuyruğu verilebilir:**

```
SATA:                         NVMe:
  Çekirdek 0 ┐                  Çekirdek 0 → Kuyruk 0 ┐
  Çekirdek 1 ├→ [tek kuyruk]    Çekirdek 1 → Kuyruk 1 ├→ SSD
  Çekirdek 2 ┤    → SSD         Çekirdek 2 → Kuyruk 2 ┤
  Çekirdek 3 ┘                  Çekirdek 3 → Kuyruk 3 ┘
     ↑ kilit yarışması              ↑ yarışma yok
```

> **Bu, Faz 1.5 ve Faz 2.4.3'teki mantığın üçüncü tekrarıdır:** Çekirdek sayısı arttıkça
> **paylaşılan tek kaynak darboğaza dönüşür.** CPU'da çözüm çok çekirdek + büyük L3 oldu,
> bellekte çok kanal oldu, depolamada çok kuyruk oldu. **Aynı problem, aynı çözüm şekli,
> üç farklı katman.**

![Şekil 3.3 — SATA'nın tek kuyruklu yolu ve NVMe'nin çekirdek başına kuyruk mimarisi](diagrams/hw-3-03-sata-vs-nvme.png)
*Şekil 3.3 — Solda tek kuyruk için yarışan çekirdekler, sağda çekirdek başına ayrı kuyruk.*

## 3.3.3 Pratik rakamlar `[uygulama]`

| Cihaz | Gecikme | Rastgele IOPS (4K) | Sıralı throughput |
|---|---|---|---|
| HDD 7200 RPM | ~10 ms | ~120 | ~150 MB/s |
| SATA SSD | ~100 μs | ~90.000 | ~550 MB/s |
| NVMe SSD (PCIe 3.0) | ~80 μs | ~500.000 | ~3.500 MB/s |
| NVMe SSD (PCIe 4.0) | ~60 μs | ~1.000.000 | ~7.000 MB/s |
| **DRAM (referans)** | **~80 ns** | — | ~50.000 MB/s |

> **Son satırı ekledim çünkü perspektif önemli.** NVMe, HDD'den 100 kat hızlı. Ama
> DRAM'den hâlâ **750 kat yavaş.**
>
> Faz 2.1.2'deki memory wall'un depolama versiyonu: SSD devrimi uçurumu **kapatmadı,
> daraltmadı bile — sadece bir basamak ekledi.** Bu yüzden page cache, Redis, uygulama
> cache'i ve CDN hâlâ vazgeçilmezdir. "Diskler artık hızlı, cache'e gerek yok" cümlesi
> hatalıdır.

## 3.3.4 Cloud bağlantısı — instance store vs EBS `[uygulama]`

AWS'te iki tür blok depolama vardır ve **ikisi arasındaki fark bu bölümün konusudur:**

| | **Instance Store** | **EBS** |
|---|---|---|
| Fiziksel konum | **Sunucunun içinde**, doğrudan NVMe | **Ağ üzerinden** erişilen depolama |
| Gecikme | ~50–100 μs | ~200 μs – 1 ms |
| IOPS | Çok yüksek (milyonlar) | Tipe göre 3.000–256.000 |
| Kalıcılık | ❌ **Instance durunca veri gider** | ✅ Instance'tan bağımsız |
| Snapshot | ❌ | ✅ |
| Boyut değiştirme | ❌ Sabit | ✅ Canlı büyütülebilir |
| Instance değiştirme | ❌ Veri taşınmaz | ✅ Detach/attach |

> **EBS'in ağ üzerinden çalışması, bu fazın en önemli cloud gerçeğidir.**
>
> EBS "disk" gibi görünür, işletim sistemi onu `/dev/nvme1n1` olarak görür, `mkfs`
> yaparsın, mount edersin. Ama fiziksel olarak **o disk başka bir makinededir.** Her
> okuma bir ağ turudur.
>
> Bunun üç sonucu var:
> 1. **Gecikme tabanı yüksektir** — ağ turunun altına inemezsin (Faz 5'te bunun neden
>    böyle olduğunu göreceğiz)
> 2. **Instance'ın ağ kapasitesi EBS performansını sınırlar** — küçük instance'larda EBS
>    bant genişliği de küçüktür (Faz 2.4.3'teki mantığın tekrarı)
> 3. **Ama kalıcılık ve esneklik kazanırsın** — instance ölse bile veri yaşar

**Ne zaman hangisi:**

| İhtiyaç | Seçim |
|---|---|
| Veritabanı ana verisi | **EBS** — kalıcılık vazgeçilmez |
| Geçici hesaplama alanı, scratch | **Instance store** — hız ve maliyet |
| Cache katmanı (yeniden üretilebilir) | **Instance store** |
| Büyük veri işleme ara çıktıları | **Instance store** |
| Kök (root) birim | **EBS** — instance'ı durdurup başlatabilmek için |

> **🤔 Düşün 3.2**
> Bir ekip, veritabanını instance store üzerinde çalıştırıp her saat başı EBS'e yedek
> almayı öneriyor. Gerekçe: "instance store çok daha hızlı, yedek de alıyoruz."
>
> a) Bu tasarımın riski nedir? Somut bir senaryo kur.
> b) Hangi durumda bu tasarım **kabul edilebilir** olur?
> *(Cevap: fazın sonunda)*

---

# 3.4 IOPS, Throughput ve Latency

**Bu bölüm fazın kalbidir.** EBS tipi seçiminin, veritabanı performans analizinin ve
depolama kaynaklı arıza teşhisinin tamamı bu üç kavramın doğru kullanılmasına dayanır.

## 3.4.1 Üç kavram, üç farklı soru `[mekanizma]`

| Kavram | Cevapladığı soru | Birim |
|---|---|---|
| **Latency** (gecikme) | Tek bir işlem **ne kadar sürüyor?** | ms, μs |
| **IOPS** | Saniyede **kaç işlem** yapılabiliyor? | işlem/s |
| **Throughput** (bant genişliği) | Saniyede **kaç byte** taşınıyor? | MB/s |

Aralarındaki ilişki basit bir çarpmadır:

```
Throughput = IOPS × Ortalama I/O boyutu
```

```
Örnek 1:  10.000 IOPS × 4 KiB  =   40 MB/s
Örnek 2:   1.000 IOPS × 1 MiB  = 1.000 MB/s
```

> **İki sisteme dikkatle bak.** Örnek 1'in IOPS'u 10 kat yüksek, ama Örnek 2'nin
> throughput'u 25 kat yüksek. **Hangisi "daha hızlı"?**
>
> Cevap: **soru yanlış.** Hangisinin daha hızlı olduğu, iş yükünün ne istediğine bağlıdır.
> Bu yüzden depolama seçimi tek bir "hız" sayısına indirgenemez.

## 3.4.2 Analoji: otoyol `[kavram]`

| Depolama | Otoyol |
|---|---|
| **Latency** | Bir arabanın A'dan B'ye varış süresi |
| **IOPS** | Saniyede geçen **araba** sayısı |
| **Throughput** | Saniyede taşınan **yolcu** sayısı |

- Şeritleri artırırsan (paralellik): IOPS ve throughput artar, **latency değişmez**
- Araba başına yolcuyu artırırsan (büyük I/O): throughput artar, IOPS aynı kalır
- Hız limitini artırırsan: latency düşer

> **Kritik içgörü:** **Latency'yi paralellikle düşüremezsin.** 10 şerit eklemek, tek bir
> arabanın varış süresini kısaltmaz.
>
> Bu, Faz 1.4.1'deki latency/throughput ayrımının aynısıdır — pipeline throughput'u
> artırıyordu ama tek bir talimatın süresini kısaltmıyordu. **Aynı ayrım, dördüncü kez:**
> pipeline'da, bellekte, depolamada ve Faz 5'te ağda.

## 3.4.3 İş yükü şekilleri — hangisi neyle sınırlı `[uygulama]`

**Bu tablo, bu fazın en pratik çıktısıdır.**

| İş yükü | I/O boyutu | Desen | Sınırlayan |
|---|---|---|---|
| OLTP veritabanı (sipariş, kullanıcı işlemi) | 4–16 KiB | Rastgele | **IOPS** |
| Veritabanı index araması | 4–8 KiB | Rastgele | **IOPS** |
| Analitik sorgu, tam tablo taraması | 128 KiB–1 MiB | Sıralı | **Throughput** |
| Video akışı / kodlama | 1–10 MiB | Sıralı | **Throughput** |
| Yedekleme / geri yükleme | Büyük | Sıralı | **Throughput** |
| Log yazma (tamponlu) | 64 KiB–1 MiB | Sıralı | **Throughput** |
| Log yazma (her satır fsync) | ~1 KiB | Sıralı ama senkron | **Latency** |
| Container imajı çekme, derleme | Karışık | Karışık | Genelde IOPS |
| Mesaj kuyruğu (Kafka) | Orta | Sıralı | **Throughput** |

**Nasıl kendin belirlersin:** Ortalama I/O boyutunu ölç.

```bash
iostat -x 1 5
```
**Beklenen çıktı (ilgili sütunlar):**
```
Device  r/s     w/s    rkB/s    wkB/s   r_await  w_await  aqu-sz  %util
nvme0n1 8500,0  120,0  34000,0  4800,0     0,45     0,80   4,20   82,5
        └─IOPS─┘       └─throughput─┘     └─latency─┘
```

Buradan:
```
Ortalama okuma boyutu = rkB/s ÷ r/s = 34000 ÷ 8500 = 4 KiB
→ Küçük ve muhtemelen rastgele → IOPS sınırlı iş yükü
```

| Sütun | Ne söyler | Alarm eşiği |
|---|---|---|
| `r/s` + `w/s` | IOPS | Sağlayıcı limitine yakınsa → IOPS darboğazı |
| `rkB/s` + `wkB/s` | Throughput | Limite yakınsa → throughput darboğazı |
| `r_await` / `w_await` | Ortalama I/O gecikmesi (ms) | SSD'de >10 ms → problem |
| `aqu-sz` | Ortalama kuyruk derinliği | Sürekli >1 ise istekler bekliyor |
| `%util` | Cihazın meşgul olduğu zaman oranı | **NVMe'de yanıltıcıdır — aşağıya bak** |

> **`%util` tuzağı — bunu bilmek seni yanlış teşhisten kurtarır.** `%util`, "cihaza en az
> bir istek gönderilmiş olan zaman yüzdesi"dir. HDD'de anlamlıydı: tek kafa var, meşgulse
> doludur. **NVMe'de paralel kanallar olduğu için %100 `%util` doygunluk anlamına gelmez**
> — cihaz %100 util'de kapasitesinin %10'unu kullanıyor olabilir. Gerçek doygunluk
> göstergesi `aqu-sz` ve `await`tir.

## 3.4.4 Latency'nin gizli kaynağı: kuyruk `[mekanizma]`

Cihazın ham gecikmesi (`svctm`) ile uygulamanın gördüğü gecikme (`await`) farklıdır:

```
await = kuyrukta bekleme süresi + cihazın işlem süresi
```

Cihaz doygunluğa yaklaştıkça kuyruk büyür ve **await patlar.** Bu patlama doğrusal
değildir:

```
Kapasitenin %50'sinde:  await ≈ 1,0 × taban gecikme
Kapasitenin %80'inde:   await ≈ 2,0 × taban
Kapasitenin %90'ında:   await ≈ 4,0 × taban
Kapasitenin %95'inde:   await ≈ 8,0 × taban
Kapasitenin %99'unda:   await ≈ 40  × taban
```

> **Bu, kuyruk teorisinin temel sonucudur ve depolamaya özgü değildir** — CPU
> zamanlayıcısında, ağ arayüzünde, thread havuzunda, veritabanı bağlantı havuzunda,
> load balancer'da aynı eğri geçerlidir.
>
> **Pratik sonucu şudur: bir kaynağı %100'e kadar kullanmaya çalışmak bir hatadır.**
> Son %10'luk kullanım, gecikmeye orantısız bir bedel ödetir. Üretim sistemleri genelde
> **%70–80 hedef kullanım** ile planlanır — "boşa giden" %20, aslında gecikme
> öngörülebilirliği için ödenen bedeldir.
>
> Bu fikri Faz 7.5'te kapasite planlamasının merkezine koyacağız.

## 3.4.5 EBS tipi seçimi — kararı türet `[uygulama]`

Artık EBS tiplerini ezberlemek yerine **türetebilirsin.**

| Tip | Teknoloji | Optimize edildiği | Tipik kullanım |
|---|---|---|---|
| **gp3** | SSD | Dengeli; IOPS ve throughput **ayrı** ayarlanır | Varsayılan seçim — çoğu iş yükü |
| **io2 / io2 Block Express** | SSD | **Yüksek IOPS + düşük, tutarlı latency** | Kritik veritabanı, yüksek IOPS gereksinimi |
| **st1** | HDD | **Yüksek sıralı throughput**, düşük maliyet | Log işleme, büyük dosya, veri ambarı taraması |
| **sc1** | HDD | **En düşük maliyet**, seyrek erişim | Arşiv, nadiren okunan veri |

**Karar ağacı:**

```
Veriye sık ve rastgele mi erişiliyor?
├─ EVET → I/O boyutu küçük mü (<32 KiB)?
│         ├─ EVET → IOPS sınırlı
│         │         ├─ Latency kritik / çok yüksek IOPS → io2
│         │         └─ Normal → gp3 (IOPS'u ayrı yükselt)
│         └─ HAYIR → Throughput sınırlı → gp3 (throughput'u ayrı yükselt)
└─ HAYIR → Erişim sıralı ve büyük mü?
          ├─ EVET → st1
          └─ Nadiren erişiliyor → sc1
```

> **gp3'ün önemli bir tasarım özelliği:** gp2'de IOPS **birim boyutuna bağlıydı** —
> daha çok IOPS için gereğinden büyük disk almak zorundaydın. gp3'te **IOPS, throughput
> ve boyut birbirinden bağımsızdır.**
>
> Bu, mimari olarak önemli bir düzeltmedir: gp2'de "performans için fazladan kapasite
> satın alma" diye bir israf vardı. Eski dokümanlarda ve blog yazılarında hâlâ gp2
> mantığına göre yazılmış öneriler göreceksin — onlar artık geçerli değil.

**Unutulmaması gereken iki sınır:**

1. **Instance'ın EBS bant genişliği, birimin kendi limitinden bağımsız bir tavandır.**
   `io2`'ye 64.000 IOPS verebilirsin ama instance 20.000 IOPS'luk EBS bant genişliğine
   sahipse, tavan 20.000'dir. **Ödediğin performansı alamazsın.** Instance'ın "EBS
   bandwidth" değerine mutlaka bak.
2. **Birden fazla EBS birimi aynı instance bant genişliğini paylaşır.** 4 tane hızlı
   birim eklemek, instance tavanını değiştirmez.

> **🤔 Düşün 3.3**
> Bir PostgreSQL sunucusunda `iostat -x 1` çıktısı şöyle:
> ```
> Device   r/s      w/s    rkB/s    wkB/s   r_await  w_await  aqu-sz  %util
> nvme1n1  2980,0   50,0  11920,0   400,0    22,40    25,10   68,50   99,8
> ```
> Birim: `gp3`, 3.000 IOPS ile yapılandırılmış.
>
> a) Darboğaz nedir? Hangi sayılardan anladın?
> b) `aqu-sz` 68,5 ne anlama geliyor?
> c) Üç farklı çözüm öner ve hangisini neden tercih edeceğini söyle.
> *(Cevap: fazın sonunda)*

![Şekil 3.4 — Aynı cihazda IOPS-sınırlı ve throughput-sınırlı iş yüklerinin karşılaştırması](diagrams/hw-3-04-iops-vs-throughput.png)
*Şekil 3.4 — Küçük rastgele I/O cihazın IOPS tavanına, büyük sıralı I/O throughput
tavanına çarpar; ikisi farklı limitlerdir.*

---
# 3.5 RAID — birden çok diski tek görmek

## 3.5.1 Neden var `[kavram]`

RAID (Redundant Array of Independent Disks) birden çok fiziksel diski tek bir mantıksal
birim olarak sunar. Üç şeyden birini veya birkaçını hedefler:

1. **Performans** — paralel disklerden aynı anda oku/yaz
2. **Dayanıklılık** — bir disk ölürse veri kaybolmasın
3. **Kapasite** — küçük diskleri birleştir

## 3.5.2 Seviyeler `[kavram]`

**RAID 0 — Striping (şeritleme)**
```
Veri:  [A][B][C][D][E][F]
Disk1: [A]   [C]   [E]
Disk2:    [B]   [D]   [F]
```
- **Performans:** N kat okuma ve yazma (paralel)
- **Dayanıklılık:** ❌ **Negatif** — bir disk ölürse **tüm veri gider**
- 2 diskin arıza olasılığı tek diskin **2 katıdır**

**RAID 1 — Mirroring (aynalama)**
```
Disk1: [A][B][C]
Disk2: [A][B][C]   ← birebir kopya
```
- **Performans:** Okuma iyileşir (iki kaynaktan), yazma iyileşmez
- **Dayanıklılık:** ✅ Bir disk ölse çalışmaya devam eder
- **Kapasite:** %50 — 2 TB disk × 2 = 2 TB kullanılabilir

**RAID 5 — Striping + dağıtık parite**
```
Disk1: [A][C][P]    P = parite (A XOR B)
Disk2: [B][P][E]
Disk3: [P][D][F]
```
- **Kapasite:** N-1 disk (3 diskten 2'si kullanılabilir)
- **Dayanıklılık:** ✅ **Bir** disk kaybını tolere eder
- **Yazma cezası:** Her yazma için eski veri + eski parite okunur, yeni parite hesaplanır,
  ikisi yazılır → **bir mantıksal yazma = dört fiziksel işlem**

> **RAID 5'in "yeniden yapılandırma (rebuild) penceresi" problemi.** Bir disk ölünce
> yenisi takılır ve veri kalan disklerden yeniden hesaplanır. Modern büyük disklerde
> (16–20 TB) bu **günler** sürer ve bu süre boyunca:
> - Diziyi koruyan hiçbir yedeklilik yoktur — **ikinci bir arıza her şeyi bitirir**
> - Kalan diskler yoğun ve sürekli okunur — arıza olasılığı tam da bu sırada artar
> - Performans belirgin düşer
>
> Bu yüzden büyük disklerde RAID 5 artık önerilmez; RAID 6 (iki parite) veya RAID 10
> tercih edilir. **Yedeklilik tasarımında "kaç arızaya dayanır" kadar "arızadan sonra ne
> kadar süre savunmasız kalır" da sorulmalıdır.**

**RAID 10 — Ayna çiftlerinin şeritlenmesi**
```
      ┌─ RAID 1 ─┐   ┌─ RAID 1 ─┐
      Disk1  Disk2   Disk3  Disk4
       [A]    [A]     [B]    [B]
      └──── RAID 0 şeritleme ────┘
```
- **Performans:** Yüksek (hem okuma hem yazma)
- **Dayanıklılık:** ✅ Yüksek, yazma cezası yok
- **Kapasite:** %50
- **Veritabanları için klasik tercih**

| Seviye | Min. disk | Kapasite | Arıza toleransı | Yazma perf. | Tipik kullanım |
|---|---|---|---|---|---|
| RAID 0 | 2 | %100 | **Yok** | En yüksek | Geçici/scratch veri |
| RAID 1 | 2 | %50 | 1 disk | Normal | Küçük, kritik sistemler |
| RAID 5 | 3 | (N-1)/N | 1 disk | **Düşük** (4× ceza) | Okuma ağırlıklı arşiv |
| RAID 6 | 4 | (N-2)/N | 2 disk | Daha düşük | Büyük diskli arşiv |
| RAID 10 | 4 | %50 | 1+ disk | Yüksek | **Veritabanı** |

![Şekil 3.5 — RAID 0, 1, 5 ve 10'un veri yerleşimi](diagrams/hw-3-05-raid-levels.png)
*Şekil 3.5 — Dört RAID seviyesinde blokların disklere dağılımı ve parite konumları.*

## 3.5.3 Cloud'da RAID'in rolü `[uygulama]`

**RAID'in klasik amacı cloud'da büyük ölçüde karşılanmış durumda:**

| RAID'in amacı | Cloud'daki karşılığı |
|---|---|
| Disk arızasına dayanıklılık | **EBS zaten AZ içinde çoğaltılmış** — disk arızası sana yansımaz |
| Kapasite birleştirme | EBS birimi canlı büyütülebilir |
| Performans | gp3/io2'de IOPS ve throughput doğrudan ayarlanır |

> **Bu yüzden cloud'da EBS üzerine RAID 1 veya RAID 5 kurmak genellikle bir hatadır:**
> zaten çoğaltılmış bir depolamayı tekrar çoğaltırsın — maliyeti ikiye katlar, dayanıklılığı
> anlamlı ölçüde artırmazsın.

**RAID'in cloud'da hâlâ anlamlı olduğu iki durum:**

1. **Instance store diskleri üzerinde RAID 0.** Bir `i`-ailesi instance'ında birden fazla
   NVMe disk gelir; RAID 0 ile birleştirmek toplam IOPS ve kapasiteyi artırır.
   **Dayanıklılık kaygısı yoktur çünkü instance store zaten kalıcı değildir** — kaybetme
   riskini zaten kabul etmişsindir.

2. **Tek birim limitini aşmak için EBS üzerinde RAID 0.** Çok nadir; ve 3.4.5'teki
   uyarıyı unutma: instance'ın EBS bant genişliği tavanı değişmez, bu yüzden çoğu zaman
   işe yaramaz.

> **Daha genel bir ilke:** Cloud'da dayanıklılık, **disk seviyesinde değil mimari
> seviyesinde** tasarlanır. Disk arızası yerine AZ arızası, bölge arızası ve veri bozulması
> düşünülür. Cevap RAID değil; snapshot, çoğaltma (replication), çoklu-AZ dağıtım ve
> yedekleme stratejisidir.
>
> Faz 7'de bunu "hangi katmanda ne garanti ediliyor" çerçevesine oturtacağız.

> **🤔 Düşün 3.4**
> Şirketinde biri, `gp3` birimlerinden 4 tanesini RAID 10 yaparak veritabanı için
> "hem hızlı hem dayanıklı" bir yapı kurmayı öneriyor.
>
> a) Bu tasarımın maliyetini ve gerçek kazancını değerlendir.
> b) Aynı bütçeyle daha iyi ne yapılabilirdi?
> *(Cevap: aşağıda)*

---

# 3.6 Bu faz bozulunca — arıza imzaları

| Belirti | Olası mekanizma | Nerede anlatıldı | İlk bakılacak |
|---|---|---|---|
| CPU düşük, `%wa` yüksek, her şey yavaş | I/O darboğazı veya thrashing | 3.4.4, 2.6.2 | `iostat -x`, `vmstat` si/so |
| `await` çok yüksek ama IOPS limitin altında | Kuyruk birikmesi veya komşu yükü | 3.4.4 | `aqu-sz`, burst kredisi |
| Uygulama diskin ilan edilen IOPS'unu alamıyor | **Kuyruk derinliği düşük** — yeterince paralel I/O yok | 3.2.5 | Eşzamanlılığı artır, async I/O |
| Performans ilk saatlerde iyi, sonra düşüyor | **Burst kredisi tükendi** (gp2/gp3 taban limiti) | 3.4.5 | CloudWatch burst balance |
| Disk dolunca yazma yavaşladı | SSD garbage collection baskısı | 3.2.5 | Kullanımı %80 altına indir |
| Veritabanı tarama sorguları yavaş, nokta sorgular hızlı | Throughput sınırı | 3.4.3 | `rkB/s` limite yakın mı |
| Nokta sorgular yavaş, taramalar normal | IOPS sınırı | 3.4.3 | `r/s` limite yakın mı |
| Instance yeniden başladı, veri gitti | **Instance store** kullanılmış | 3.3.4 | Birim tipini doğrula |
| `%util` %100 ama sistem iyi çalışıyor | NVMe'de `%util` yanıltıcıdır | 3.4.3 not | `aqu-sz` ve `await`e bak |
| Yedekten dönüş beklenenden çok uzun sürdü | Throughput hesabı yapılmamış | 3.4.1 | Geri yükleme süresini **ölç**, tahmin etme |

> **Son satır bir uyarıdır:** Yedekleme stratejisi genelde "ne sıklıkla yedek alıyoruz"
> sorusuyla tasarlanır. Ama asıl soru **"geri dönmek ne kadar sürer"**dir ve cevabı bu
> fazın matematiğiyle hesaplanır:
> ```
> 2 TB veri ÷ 250 MB/s = 8.000 saniye ≈ 2 saat 13 dakika
> ```
> Hedef kurtarma süresi (RTO) 30 dakika ise bu yedekleme tasarımı **çalışmaz** — ve bunu
> gerçek bir olayda değil, şimdi öğrenmek gerekir.

---
# Faz 3 — Düşün Sorularının Cevapları

## Cevap 3.1 — Her satırı yazmak mı, tamponlayıp yazmak mı?

**a) Performans farkı: 50–500 kat. Ve sebep tek değil, dört tane.**

**Tasarım A (her satır ayrı `write()` + `fsync()`):**
```
50.000 işlem/saniye × ~200 byte
Her fsync ≈ 100–500 μs (veriyi kalıcılaştırma turu)
→ 50.000 × 200 μs = 10 saniye/saniyelik iş
→ Sistem yetişemez, kuyruk sonsuz büyür
```

Dört ayrı mekanizma A'yı cezalandırıyor:

1. **fsync maliyeti.** Her `fsync()` cihaza "kalıcılaştır" komutu gönderir ve cevabı
   bekler. Bu senkron bir turdur — kuyruk derinliği 1'de kalır, yani 3.2.5'teki QD
   problemi tam olarak yaşanır. SSD'nin paralel kanallarının hiçbiri kullanılamaz.
2. **Write amplification.** 200 byte yazıyorsun ama SSD'nin en küçük yazma birimi bir
   sayfadır (4–16 KiB). **Her 200 byte için ~4 KiB yazılıyor — 20 kat büyütme** (3.2.2).
3. **Dosya sistemi ek yükü.** Her yazma metadata güncellemesi ve journal kaydı üretir.
4. **Erken yıpranma.** 20 kat write amplification, SSD'nin P/E bütçesini 20 kat hızlı
   tüketir (3.2.4).

**Tasarım B (4 MiB tampon):**
```
4 MiB ÷ 200 byte ≈ 20.000 satır per yazma
50.000 satır/s ÷ 20.000 = saniyede ~2,5 yazma işlemi
Her yazma büyük ve sıralı → throughput sınırlı, cihaz için kolay iş
→ 50.000 fsync yerine 2,5 yazma
```

Write amplification ~1'e iner, kuyruk derinliği verimli kullanılır, dosya sistemi ek yükü
20.000 satıra bölünür.

**b) B'nin bedeli: veri kaybı penceresi.**

Tamponda bekleyen veri **henüz kalıcı değildir.** Süreç çökerse veya makine güç kaybederse
o 4 MiB (≈20.000 satır, ≈0,4 saniyelik veri) kaybolur.

**Bu bir hata değil, bilinçli bir takastır** ve gerçek sistemler bu ekseni ayarlanabilir
yapar:

| Sistem | Ayar | Takas |
|---|---|---|
| PostgreSQL | `synchronous_commit` | `off` → çok hızlı, son işlemler risk altında |
| MySQL | `innodb_flush_log_at_trx_commit` | `2` → OS'a bırak, hızlı; `1` → her commit fsync |
| Kafka | `acks`, `flush.ms` | Dayanıklılık ve gecikme arasında kademe |
| Linux | `dirty_expire_centisecs` | Page cache'in ne kadar bekleyeceği |

**Doğru cevap veriye bağlıdır:**
- **Finansal işlem, sipariş kaydı:** A tarafı (her işlem kalıcı olmalı) — ama o zaman
  yazma hızını grup commit (group commit) ile çözersin, tamponsuz değil
- **Uygulama logu, metrik, telemetri:** B tarafı — 0,4 saniyelik kayıp kabul edilebilir

> **Gerçek sistemlerde kullanılan üçüncü yol: group commit.** Aynı anda gelen işlemleri
> tek bir fsync'te birleştirmek. Hem kalıcılığı hem verimi korur. PostgreSQL'in
> `commit_delay`'i ve Kafka'nın batching'i tam olarak budur.
>
> **Genel ders: "hız mı dayanıklılık mı" sorusu genelde bir yelpazedir, ikili seçim
> değil.** Ve doğru noktayı seçmek için verinin ne kadar değerli olduğunu bilmek gerekir
> — bu teknik değil, iş kararıdır.

## Cevap 3.2 — Instance store'da veritabanı, saatlik EBS yedeği

**a) Risk: en fazla 1 saatlik veri kaybı — ve bu neredeyse hiçbir veritabanı için kabul
edilebilir değildir.**

**Somut senaryo:**
```
14:00  Yedek alındı
14:47  Yoğun saat — 4.200 sipariş işlendi
14:58  AWS donanım arızası → instance durdu/değiştirildi
       → Instance store SİLİNDİ (3.3.4)
15:05  14:00 yedeğinden geri dönüldü

Kayıp: 58 dakikalık tüm işlemler.
```

Ve kayıp sadece teknik değil:
- Müşteri parayı ödedi, sipariş kaydı yok
- Ödeme sağlayıcısında işlem var, sende yok → **mutabakat (reconciliation) kâbusu**
- Stok sayıları yanlış
- Bağlı sistemlere (kargo, faturalama) gitmiş kayıtlar artık "yetim"

**Ve bu senaryo düşündüğünden daha sık gerçekleşir.** Instance store yalnızca instance
çökerse gitmez; şu durumlarda da gider:
- Instance **durdurulup başlatılırsa** (stop/start — restart değil)
- Altta yatan donanım arızalanırsa (AWS instance'ı taşır)
- Instance tipi değiştirilirse
- Spot instance geri alınırsa

Yani **planlı bir bakım bile veri kaybına yol açar.** Ekip "çökme nadir" diye
düşünüyor ama gerçek olasılık çok daha yüksek.

**b) Ne zaman kabul edilebilir?**

Bu tasarım şu koşullarda **doğru** olur:

1. **Veri yeniden üretilebilirse.** Veritabanı bir *kaynaktan türetilmiş* veri tutuyorsa
   — arama indeksi (Elasticsearch), önbellek katmanı, materyalize görünüm, analitik
   kopya — kaybı yeniden inşa ederek telafi edebilirsin.
2. **Veri kaynağı başka yerdeyse.** Kafka'dan beslenen bir okuma modeli, olay akışından
   yeniden oynatılabilir (event sourcing).
3. **Geçici hesaplama ise.** Spark ara çıktıları, ML eğitim checkpoint'leri, derleme
   önbelleği.
4. **Yazma-önü log (WAL) ayrı ve kalıcı tutuluyorsa.** Bu, ciddi bir mimaridir: veri
   dosyaları hızlı instance store'da, WAL kalıcı EBS'te veya senkron bir replikada.
   Bazı yüksek performanslı kurulumlar bunu yapar — ama "saatlik yedek" ile aynı şey
   değildir.

> **Ekibin asıl hatası teknik değil, kavramsal:** "Yedek alıyoruz" ile "veri kaybını
> önlüyoruz" aynı şey değildir. Yedek, **kurtarma noktası hedefi (RPO)** kadar veri
> kaybını **kabul etmek** demektir. Saatlik yedek = "1 saatlik veri kaybını göze
> alıyoruz" beyanıdır.
>
> Doğru soru şudur: **"Ne kadar veri kaybını göze alabiliriz?"** Cevap "hiç" ise, yedek
> yeterli değildir — senkron çoğaltma gerekir.

## Cevap 3.3 — PostgreSQL `iostat` çıktısının teşhisi

**a) Darboğaz: IOPS limiti. gp3 birimi 3.000 IOPS'ta yapılandırılmış ve tam o sınırda.**

Kanıtlar:
```
r/s + w/s = 2980 + 50 = 3.030 ≈ 3.000    ← YAPILANDIRILAN LİMİT. Tam tavanda.
Ortalama okuma boyutu = 11920 ÷ 2980 = 4 KiB  ← küçük, rastgele — OLTP deseni
Throughput = ~12,3 MB/s                   ← gp3 için gülünç düşük, throughput sorun DEĞİL
r_await = 22,4 ms                         ← SSD'de olağanüstü yüksek (normal: 0,2-1 ms)
%util = 99,8                              ← doygun
```

**Teşhis net:** 4 KiB'lık rastgele okumalar, IOPS tavanına dayanmış. Throughput
kullanımı %1'in altında — yani sorun "disk yavaş" değil, **"izin verilen işlem sayısı
dolmuş."** Bu, 3.4.3'teki tablonun OLTP satırının ders kitabı örneğidir.

**b) `aqu-sz` 68,5 ne demek?**

Ortalama **68,5 istek sürekli kuyrukta bekliyor.** Cihaz her an sadece birkaçını
işleyebiliyor, geri kalanı sırada.

Little yasasıyla doğrulayalım:
```
await ≈ kuyruk uzunluğu ÷ işlem hızı
22,4 ms ≈ 68,5 ÷ 3.030 IOPS = 22,6 ms   ✓ tutarlı
```

**Yani 22,4 ms'lik gecikmenin neredeyse tamamı kuyrukta bekleme.** Cihazın kendi hizmet
süresi ~0,3 ms. 3.4.4'teki eğrinin %99 satırındasın: taban gecikmenin **~70 katını**
ödüyorsun.

Uygulama tarafında görünümü: her sorgu 22 ms disk bekliyor, bağlantı havuzu doluyor,
p99 gecikme patlamış durumda.

**c) Üç çözüm:**

**1. gp3'ün IOPS ayarını yükselt (en hızlı çözüm — dakikalar)**
- 3.000 → 12.000 IOPS
- Kesinti yok, canlı uygulanır
- Maliyet: IOPS başına aylık ek ücret, öngörülebilir
- ⚠️ **Önce instance'ın EBS bant genişliğini kontrol et** (3.4.5, 1. sınır) — yoksa
  ödediğini alamazsın
- **Bu, acil durumda yapılacak doğru hamledir**

**2. Uygulama seviyesinde I/O'yu azalt (en kalıcı çözüm — günler)**
- 2.980 okuma/saniye çok yüksek — **`shared_buffers` yetersiz olabilir.** PostgreSQL
  veriyi bellekte tutamıyorsa her sorgu diske iniyor demektir.
- `pg_stat_statements` ile en çok blok okuyan sorguları bul
- Eksik index → tam tablo taraması → gereksiz rastgele okuma
- Cevap 2.3'ün mantığı: **belleğe sığan çalışma kümesi, en hızlı diskten hızlıdır**
- Basit bir index, 3.000 IOPS'u 50 IOPS'a düşürebilir

**3. io2'ye geç (en pahalı — ve genelde erken bir hamle)**
- Daha yüksek IOPS tavanı ve **daha tutarlı latency** (p99 garantisi)
- Belirgin maliyet artışı
- **Sadece 2. adım yapıldıktan sonra hâlâ yetersizse mantıklı**

**Tercih ve sırası:**

> **Önce 1, paralelinde 2, gerekirse 3.**
>
> 1'i hemen yap — sistem şu anda kullanılamaz hâlde ve bu kararın geri dönüşü kolay.
> Ama 1 **semptomu** tedavi eder: uygulama hâlâ gereğinden 60 kat fazla I/O üretiyor
> olabilir.
>
> Asıl iş 2'dir. 2.980 IOPS'luk rastgele okuma, iyi yapılandırılmış bir OLTP veritabanı
> için **anormal derecede yüksektir** ve neredeyse her zaman şunlardan birini işaret
> eder: yetersiz `shared_buffers`, eksik index, veya N+1 sorgu deseni.
>
> **Ve bu, Faz 2.3.6'daki dersin tekrarıdır:** erişim desenini düzeltmek, donanım
> yükseltmesinden hem daha ucuz hem daha büyük kazanç verir. Donanımı büyütmek 4 kat
> kazandırır; index eklemek 60 kat.

## Cevap 3.4 — 4 × gp3 ile RAID 10

**a) Maliyet ve gerçek kazanç değerlendirmesi**

**Maliyet:**
```
RAID 10 → kapasitenin %50'si kullanılabilir
1 TB kullanılabilir alan için 2 TB gp3 satın alırsın
→ Depolama maliyeti 2 KAT
→ Ayrıca yönetim karmaşıklığı: mdadm yapılandırması, boot sırası,
  snapshot tutarlılığı (4 birimin anlık görüntüsü senkron alınmalı),
  büyütme prosedürü, arıza senaryoları
```

**Dayanıklılık kazancı: sıfıra yakın.**

EBS **zaten AZ içinde çoğaltılmıştır** (3.5.3). Bir fiziksel disk arızası sana hiç
yansımaz. RAID 1'in koruduğu şey — tek disk arızası — **EBS'te zaten senin problemin
değil.**

Üstelik RAID **yeni bir risk ekler:** 4 birimden herhangi birinin erişilemez olması
(ağ kesintisi, birim seviyesinde bir olay) tüm diziyi etkiler. **Bağımlılık sayısını
artırdın.** Ve snapshot artık atomik değil — 4 birimi tutarlı bir anda dondurmak ek
özen gerektirir.

**Performans kazancı: kısmen gerçek, ama gereksiz yoldan elde edilmiş.**

Evet, 4 birim paralel çalışır ve toplam IOPS artar. **Ama gp3'te aynı sonucu tek birimin
IOPS ayarını yükselterek elde edebilirsin** (3.4.5) — hiçbir karmaşıklık eklemeden.

Ve 3.4.5'teki 1. sınır burada da geçerli: **instance'ın EBS bant genişliği tavanı
değişmedi.** 4 birim eklemek o tavanı yükseltmez. Yani RAID 10'un vaat ettiği performans,
büyük ihtimalle instance limitine çarpar ve **hiç gerçekleşmez.**

**b) Aynı bütçeyle daha iyisi**

```
Mevcut plan:  4 × 1 TB gp3 (RAID 10) = 4 TB satın alma, 2 TB kullanılabilir

Daha iyi:     1 × 2 TB gp3, IOPS ve throughput doğrudan yükseltilmiş
              + artan bütçe ile:
                → Yeterli EBS bant genişliğine sahip instance seç
                → Multi-AZ standby replika (GERÇEK dayanıklılık)
                → Otomatik snapshot + point-in-time recovery
```

**Neden daha iyi:**

| Boyut | RAID 10 | Multi-AZ replika |
|---|---|---|
| Disk arızası | Korur (ama EBS zaten koruyordu) | Korur |
| **AZ arızası** | ❌ Korumaz | ✅ Korur |
| **İnstance arızası** | ❌ Korumaz | ✅ Korur |
| **Veri bozulması / yanlış DELETE** | ❌ Korumaz (aynaya da yazılır) | PITR ile korur |
| Yönetim karmaşıklığı | Yüksek | Yönetilen servis hallediyor |

> **Buradaki asıl ders — ve bu, cloud'a geçişte en sık yapılan kavramsal hatadır:**
>
> **Cloud'da dayanıklılık, geleneksel veri merkezindekinden farklı bir katmanda
> tasarlanır.** Kendi sunucunda disk ölürdü ve RAID doğru cevaptı. Cloud'da disk arızası
> sağlayıcının problemidir; senin problemin **instance arızası, AZ arızası, bölge
> arızası ve insan hatasıdır.**
>
> RAID bunların **hiçbirine** karşı koruma sağlamaz. Alışkanlıktan taşınmış bir çözümü,
> artık var olmayan bir probleme uyguluyorsun — ve karşılığında iki kat maliyet,
> ek karmaşıklık ve yeni bir arıza noktası ödüyorsun.
>
> **Her koruma mekanizması için sorulacak soru:** *"Bu tam olarak hangi arıza senaryosunu
> önlüyor, ve o senaryo bu ortamda gerçekten benim sorumluluğumda mı?"*

---
# Faz 3 — Sık Sorulan Sorular

> **S1: SSD'ler artık çok hızlı. Cache katmanlarına (Redis, page cache) hâlâ gerek var mı?**
>
> Evet, ve rakamlar net: NVMe ~60 μs, DRAM ~0,08 μs → **750 kat fark.**
>
> SSD, HDD ile DRAM arasındaki uçurumu daralttı ama kapatmadı. 3.3.3'teki tablonun son
> satırı bunun içindir. Bir Redis sorgusu ~0,2 ms, aynı veriyi diskten okumak ~1–5 ms
> (dosya sistemi ve veritabanı katmanlarıyla birlikte).
>
> **Değişen şey şudur:** Eskiden cache *zorunluydu*, şimdi bir *mühendislik kararıdır.*
> HDD çağında cache'siz bir sistem çalışamazdı; bugün çalışır ama yavaş olur. Bu, bazı
> mimarilerde cache katmanını **kaldırıp** karmaşıklıktan kurtulmayı mümkün kılar — ki
> bu da geçerli bir tercihtir.

> **S2: EBS bir "disk" değil mi? Neden ağ üzerinden çalışması bu kadar önemli?**
>
> Çünkü üç şeyi değiştirir:
>
> 1. **Gecikme tabanı.** Yerel NVMe ~60 μs, EBS ~200 μs–1 ms. Bu farkı hiçbir ayarla
>    kapatamazsın — ağ turu fizikseldir.
> 2. **Instance'ın ağ kapasitesi tavan koyar.** Küçük instance = küçük EBS bant genişliği.
>    Birime ne kadar IOPS verirsen ver, instance tavanını aşamazsın (3.4.5).
> 3. **Ama bunun karşılığında kalıcılık ve taşınabilirlik alırsın.** Instance ölse veri
>    yaşar, snapshot alınabilir, başka instance'a bağlanabilir.
>
> **Bu bir kusur değil, bilinçli bir takastır** ve cloud'un temel tasarım desenlerinden
> biridir: *hesaplama ile depolamayı ayır.* S3, EFS, RDS — hepsi aynı ayrımın üstüne
> kurulmuştur. Ayırmanın bedeli gecikme, kazancı esnekliktir.

> **S3: `iostat`'ta `%util` %100 görüyorum. Disk doldu mu, yükseltmeli miyim?**
>
> **Hayır, acele etme — NVMe'de `%util` yanıltıcıdır.** `%util`, "cihaza en az bir istek
> gönderilmiş olan zaman yüzdesi"dir. HDD'de anlamlıydı (tek kafa var, meşgulse doludur).
> NVMe'de onlarca paralel kanal olduğu için %100 util, kapasitenin %10'unda da görülebilir.
>
> **Gerçek doygunluk göstergeleri:**
> - `aqu-sz` sürekli >1 → istekler bekliyor
> - `await` taban gecikmenin katlarına çıkmış → kuyruk birikmiş (3.4.4)
> - `r/s + w/s` yapılandırılan IOPS limitine dayanmış → tavandasın
>
> Cevap 3.3'teki örnek bu üçünün de birlikte gerçekleştiği durumdur — orada teşhis
> kesindir.

> **S4: Veritabanım için kaç IOPS gerekir?**
>
> Tahmin etme, **ölç.** Ve ölçüm sırası şudur:
>
> 1. Mevcut sistemde `iostat -x 1 60` çalıştır, yoğun saatte `r/s + w/s` **p95** değerini
>    al (ortalamayı değil)
> 2. Büyüme payı ekle (%50–100)
> 3. **Kısa süreli sıçramaları (burst) hesaba kat** — yedekleme, toplu iş, index yeniden
>    oluşturma
>
> **Ama daha önemli bir soru var:** Ölçtüğün IOPS *gerekli* mi, yoksa *israf* mı? Cevap
> 3.3'teki gibi 3.000 IOPS ölçtüysen, sormadan önce şunu kontrol et: bellek yeterli mi,
> index'ler doğru mu, sorgular verimli mi?
>
> **Yanlış yapılandırılmış bir veritabanının IOPS ihtiyacını satın almak, problemi çözmez
> — sadece pahalıya taşır.**

> **S5: Instance store'u hiç kullanmamalı mıyım?**
>
> Kullanmalısın — ama **doğru veriyle.** Ölçüt tek bir soru: *"Bu veri kaybolursa yeniden
> üretebilir miyim?"*
>
> | Veri | Instance store |
> |---|---|
> | Geçici hesaplama, scratch alanı | ✅ İdeal |
> | Cache (Redis, Memcached) | ✅ Uygun — kaynak başka yerde |
> | Spark/ML ara çıktıları | ✅ Uygun |
> | Arama indeksi (yeniden inşa edilebilir) | ✅ Uygun |
> | Veritabanı ana verisi | ❌ Hayır (Cevap 3.2) |
> | Kullanıcı yüklemeleri | ❌ Hayır |
>
> Ve maliyet avantajı gerçektir: instance store fiyata dahildir, ayrıca ödemezsin.

> **S6: gp2 mi gp3 mü? Eski dokümanlarda farklı şeyler yazıyor.**
>
> **Neredeyse her durumda gp3.** Sebep mimari:
>
> | | gp2 | gp3 |
> |---|---|---|
> | IOPS | **Boyuta bağlı** (3 IOPS/GB) | **Bağımsız ayarlanır** |
> | Throughput | Boyuta bağlı | Bağımsız ayarlanır |
> | Taban performans | 100 IOPS | **3.000 IOPS dahil** |
> | Maliyet | Daha yüksek | ~%20 daha düşük |
>
> gp2'de 6.000 IOPS için 2 TB disk almak zorundaydın — 500 GB'lık veri için. **Performans
> için kapasite satın almak** diye bir israf vardı ve gp3 bunu kaldırdı.
>
> Eski blog yazılarındaki "IOPS için diski büyüt" tavsiyesi gp2 dönemine aittir ve artık
> geçerli değildir. Bu, cloud'da genel bir uyarıdır: **dokümanın tarihine bak.**

> **S7: Bu fazdaki donanım detayları (NAND hücresi, FTL, wear leveling) gerçekten lazım
> mı? Ben EBS tipi seçiyorum sadece.**
>
> Doğru seçim yapmak için lazım değil; **doğru seçimi savunmak ve beklenmedik davranışı
> açıklamak** için lazım.
>
> Somut örnekler:
> - "Disk %92 dolu ve yazmalar yavaşladı" → garbage collection baskısı (3.2.5) —
>   bilmiyorsan rastgele arama yaparsın
> - "Uygulama 500K IOPS'luk diskten 15K alıyor" → kuyruk derinliği (3.2.5) — disk
>   yükseltmek işe yaramaz, çözüm kodda
> - "Aynı sorgu bazen 2 ms bazen 40 ms" → kuyruk eğrisi (3.4.4)
> - "Log diski 6 ayda yıprandı" → write amplification (Cevap 3.1)
>
> Bunların hiçbirini "EBS tiplerini ezberleyerek" çözemezsin. **Mekanizma bilgisi,
> beklenmedik durumla karşılaştığında işe yarar** — ve üretimde beklenmedik durum
> kuraldır.

---

# Faz 3 — Kendini Sına

**Bölüm A — Temel (1–7)**

1. HDD'de bir okumanın üç aşamasını ve sürelerini say. Hangisi baskın?
2. HDD'nin rastgele IOPS'u neden 30 yılda neredeyse artmadı, ama kapasitesi 200.000 kat
   arttı?
3. NAND Flash'ta veri fiziksel olarak nasıl saklanır? Neden güç kesilince kaybolmaz?
4. SSD'de okuma, yazma ve silme hangi birimlerde yapılır? Bu asimetri neye yol açar?
5. QLC neden TLC'den hem yavaş hem daha az dayanıklıdır?
6. Latency, IOPS ve throughput'un tanımlarını ve birimlerini yaz.
7. Instance store ile EBS arasındaki en temel fark nedir?

**Bölüm B — Mekanizma (8–14)**

8. `Throughput = IOPS × I/O boyutu` formülünü kullanarak, 2.000 IOPS ve 128 KiB I/O
   boyutlu bir iş yükünün throughput'unu hesapla. Bu iş yükü neyle sınırlı olabilir?
9. Write amplification nedir? FTL bunu nasıl azaltır?
10. NVMe'nin SATA'ya üstünlüğünü sadece bant genişliğiyle açıklamak neden eksiktir?
11. Bir SSD 500.000 IOPS destekliyor ama uygulaman 15.000 alıyor. Donanım arızalı mı?
    Açıkla.
12. `await` ile cihazın hizmet süresi arasındaki fark nedir? `aqu-sz` bu farkı nasıl
    açıklar?
13. RAID 5'in yazma cezası neden 4 fiziksel işlemdir? Rebuild penceresi neden risklidir?
14. Bir kaynağı %95 kullanımda çalıştırmak neden %70'te çalıştırmaktan orantısız pahalıdır?

**Bölüm C — Uygulama ve muhakeme (15–21)**

15. Bir iş yükünün IOPS mı throughput mı sınırlı olduğunu `iostat` çıktısından nasıl
    anlarsın? Adım adım yaz.
16. Analitik bir veri ambarı için hangi EBS tipini seçersin ve neden?
17. NVMe diskte `%util` %100 görüyorsun. Hangi üç metriğe daha bakarsın?
18. Cloud'da EBS üzerine RAID 1 kurmak neden genellikle yanlıştır?
19. 3 TB veriyi 300 MB/s hızla geri yüklemek ne kadar sürer? RTO'n 1 saat ise ne
    yaparsın?
20. Bir uygulama her log satırını `fsync` ile yazıyor ve SSD 8 ayda yıprandı. Ne oldu
    ve nasıl düzeltirsin?
21. "Veritabanım yavaş, io2'ye geçelim" diyen bir ekip arkadaşına hangi soruları
    sorarsın? (En az dört soru)

---

## Cevap Anahtarı

**1.** (a) **Seek time** ~4–9 ms — kafanın doğru track'e hareketi; (b) **Rotational
latency** ~4,2 ms — sektörün kafanın altına gelmesi; (c) **Transfer** ~0,01 ms. **Baskın
olan seek + rotasyon** — yani sürenin %99,9'u veriye *ulaşmak* için harcanır, okumak için
değil. *(3.1.2)*

**2.** Kapasite ve sıralı hız **manyetik yoğunluğa** bağlıdır — mühendislikle artırılabilir.
Rastgele IOPS ise **mekanik harekete** bağlıdır: kolun hareket süresi ve plakanın dönüş
hızı. 15.000 RPM'in üstü titreşim ve ısı nedeniyle pratik değil. *(3.1.3)*

**3.** **Yüzer kapıya (floating gate) hapsedilmiş elektronlarla.** Yüzer kapı her yönden
yalıtkanla çevrilidir; elektronların kaçacak yolu yoktur, bu yüzden güç kesilse de kalırlar.
*(3.2.1)*

**4.** Okuma ve yazma **sayfa** (4–16 KiB), silme **blok** (128–256 sayfa, 1–4 MiB) bazında.
Asimetri şuna yol açar: bir sayfayı güncellemek için tüm bloğu silmek gerekir → **write
amplification**. FTL bunu yerinde-olmayan yazma ile aşar. *(3.2.2, 3.2.3)*

**5.** QLC bir hücrede 4 bit, yani **16 voltaj seviyesi** ayırt eder (TLC'de 8). Seviyeler
arası aralık daralır: okuma/yazma daha hassas ölçüm gerektirir (**yavaş**), yalıtkandaki
küçük yıpranma seviyeleri karıştırır (**dayanıksız**). Faz 0.1.1'deki gürültü payı
argümanının aynısı. *(3.2.4)*

**6. Latency:** tek bir işlemin süresi (ms, μs). **IOPS:** saniyedeki işlem sayısı
(işlem/s). **Throughput:** saniyedeki veri miktarı (MB/s). İlişki:
`Throughput = IOPS × I/O boyutu`. *(3.4.1)*

**7.** Instance store **sunucunun içindedir** (doğrudan NVMe, çok hızlı) ama **kalıcı
değildir** — instance durunca veri gider. EBS **ağ üzerinden** erişilir (daha yüksek
gecikme) ama **kalıcıdır**, snapshot alınabilir, başka instance'a taşınabilir. *(3.3.4)*

**8.** `2.000 × 128 KiB = 256.000 KiB/s = 250 MB/s`. I/O boyutu büyük (128 KiB) →
muhtemelen **sıralı ve throughput sınırlı** bir iş yükü (analitik tarama, yedekleme,
video). IOPS 2.000 düşük bir sayıdır; darboğaz IOPS değil, throughput tavanı olacaktır.
*(3.4.1, 3.4.3)*

**9. Write amplification:** Mantıksal olarak yazılan veriden **fiziksel olarak daha
fazlasının** yazılması (200 byte yazmak için 4 KiB sayfa, veya bir sayfa için tüm bloğun
oku-sil-yaz döngüsü). **FTL'nin çözümü:** yerinde güncelleme yapmaz — yeni veriyi başka
bir boş sayfaya yazar, eşleme tablosunu günceller, eski sayfayı geçersiz işaretler ve
silmeyi arka plana (garbage collection) erteler. *(3.2.2, 3.2.3)*

**10.** Çünkü asıl fark **kuyruk mimarisidir.** SATA'da tek kuyruk (derinlik 32) vardır ve
tüm çekirdekler onun için yarışır. NVMe'de 65.535 kuyruk vardır ve **her çekirdek kendi
kuyruğunu kullanabilir** — kilit yarışması ortadan kalkar. Ayrıca NVMe'nin komut başına
CPU maliyeti çok daha düşüktür. Bant genişliği farkı önemlidir ama tek başına IOPS
farkını (90K → 1M) açıklamaz. *(3.3.2)*

**11.** **Hayır, donanım sağlıklı.** Uygulama muhtemelen **kuyruk derinliği 1** ile çalışıyor
— senkron, tek thread'li I/O. SSD'nin ilan edilen IOPS'u onlarca NAND kanalının **paralel**
kullanılmasıyla elde edilir. Çözüm daha hızlı disk değil, **daha çok eşzamanlı I/O**:
async I/O, thread havuzu, daha derin kuyruk. *(3.2.5)*

**12.** `await` = **kuyrukta bekleme + cihazın hizmet süresi.** Cihaz doygunluğa
yaklaştıkça kuyruk büyür ve await patlar. `aqu-sz` (ortalama kuyruk uzunluğu) bu farkı
doğrudan ölçer: `await ≈ aqu-sz ÷ IOPS`. Cevap 3.3'te 22,4 ms await'in ~22,1 ms'i
kuyrukta beklemeydi. *(3.4.4)*

**13. Yazma cezası:** Bir bloğu güncellemek için (a) eski veri okunur, (b) eski parite
okunur, (c) yeni parite hesaplanır, (d) veri yazılır, (e) parite yazılır → **4 fiziksel
I/O.** **Rebuild penceresi:** Disk değişince veri kalan disklerden yeniden hesaplanır;
büyük disklerde bu **günler** sürer ve o süre boyunca dizi **yedeksizdir** — ikinci bir
arıza her şeyi bitirir. Üstelik yoğun okuma, tam da o sırada arıza olasılığını artırır.
*(3.5.2)*

**14.** **Kuyruk teorisi:** Kullanım oranı arttıkça bekleme süresi doğrusal değil
üstel artar. %70'te await ≈ 1,5× taban, %95'te ≈ 8×, %99'da ≈ 40×. Son %10'luk
kullanım için ödenen gecikme bedeli orantısızdır. Bu yüzden üretim sistemleri %70–80
hedef kullanımla planlanır — "boşa giden" kapasite, **gecikme öngörülebilirliğinin
bedelidir.** *(3.4.4)*

**15.** (a) `iostat -x 1 5` çalıştır. (b) **Ortalama I/O boyutunu hesapla:**
`rkB/s ÷ r/s`. (c) Küçükse (<32 KiB) → IOPS sınırlı aday; büyükse (>128 KiB) →
throughput sınırlı aday. (d) **Hangi sayının limite dayandığına bak:** `r/s + w/s`
yapılandırılan IOPS limitine yakınsa IOPS darboğazı; `rkB/s + wkB/s` throughput limitine
yakınsa throughput darboğazı. (e) `await` ve `aqu-sz` ile doygunluğu doğrula. *(3.4.3)*

**16. `st1`** (throughput optimized HDD) — veri ambarı tam tablo taramaları yapar:
**büyük, sıralı okumalar.** IOPS ihtiyacı düşük, throughput ihtiyacı yüksek, maliyet
duyarlılığı yüksek (veri hacmi büyük). Eğer sorgular arasında rastgele index erişimi de
varsa `gp3` daha güvenli bir tercih olur. *(3.1.4, 3.4.5)*

**17.** (a) **`aqu-sz`** — sürekli >1 ise istekler bekliyor; (b) **`await`** — taban
gecikmenin katlarına çıkmış mı; (c) **`r/s + w/s`** — yapılandırılan IOPS limitine
dayanmış mı. NVMe'de `%util` paralel kanallar nedeniyle doygunluk göstergesi değildir.
*(3.4.3 notu, SSS S3)*

**18.** Çünkü **EBS zaten AZ içinde çoğaltılmıştır** — RAID 1'in koruduğu tek-disk
arızası senin problemin değil. Maliyeti ikiye katlarsın, dayanıklılığı anlamlı ölçüde
artırmazsın, karmaşıklık ve yeni bağımlılık eklersin. Cloud'da asıl riskler instance
arızası, AZ arızası ve insan hatasıdır — **RAID bunların hiçbirine karşı koruma
sağlamaz.** *(3.5.3, Cevap 3.4)*

**19.** `3.000.000 MB ÷ 300 MB/s = 10.000 saniye ≈ 2 saat 47 dakika.` **RTO 1 saat ise
bu tasarım çalışmaz.** Seçenekler: (a) geri yükleme throughput'unu artır (paralel
kurtarma, daha yüksek throughput'lu birim, daha büyük instance bant genişliği);
(b) sıcak bekleyen (warm standby) replika tut — geri yükleme yerine devretme yap;
(c) veriyi katmanlara ayır: kritik %10'u önce kurtar, sistemi kısmi hizmetle aç.
**En önemlisi: bunu gerçek bir olayda değil, tatbikatla önceden ölç.** *(3.6 notu)*

**20. Ne oldu: write amplification.** Her log satırı ~200 byte ama SSD'nin en küçük yazma
birimi bir sayfa (4–16 KiB) → **~20–80 kat fazla fiziksel yazma.** Ayrıca her `fsync`
kuyruk derinliğini 1'e düşürüp verimi öldürüyor. P/E bütçesi 20+ kat hızlı tükendi.
**Düzeltme:** Yazmaları tamponla veya group commit kullan (Cevap 3.1); `fsync`
sıklığını veri kritikliğine göre ayarla; log için ayrı ve uygun bir birim kullan.
*(3.2.2, 3.2.4, Cevap 3.1)*

**21.** En az dört soru:
1. **"Darboğazın IOPS mı, throughput mu, latency mi olduğunu ölçtük mü?"** — `iostat`
   çıktısı var mı, yoksa tahmin mi ediyoruz?
2. **"Mevcut gp3 biriminin IOPS ayarını yükseltmeyi denedik mi?"** — çoğu zaman io2'ye
   geçmeden önce bu yeterlidir ve çok daha ucuzdur
3. **"Bu IOPS gerçekten gerekli mi?"** — `shared_buffers`/bellek yeterli mi, eksik index
   var mı, N+1 sorgu deseni var mı? (Cevap 3.3'ün asıl dersi)
4. **"Instance'ın EBS bant genişliği bu IOPS'u taşıyabiliyor mu?"** — yoksa ödediğimizi
   alamayız (3.4.5)
5. (Bonus) **"Sorunun disk olduğundan emin miyiz?"** — CPU, bellek, kilit yarışması veya
   ağ da aynı semptomu verebilir

---

## Puanlama

| Doğru sayısı | Değerlendirme |
|---|---|
| 19–21 | Faz 4'e geç. Depolama kararlarını gerekçelendirebilirsin. |
| 15–18 | Faz 4'e geçebilirsin. 3.4'ü bir kez daha oku. |
| 10–14 | 3.2 ve 3.4'ü tekrar et. Özellikle 3.4.3'teki iş yükü tablosunu ezberleme, **türet**. |
| 0–9 | Fazı baştan geç. 3.1.2'deki "veriye ulaşmak pahalı, okumak ucuz" gözlemi olmadan kalanı taşıyamazsın. |

---

# Faz 3 — Kapanış ve Faz 4'e Köprü

## Bu fazdan ne taşıyorsun

| Kavram | Neden taşınıyor |
|---|---|
| **IOPS ≠ throughput ≠ latency** | Faz 5'te ağda birebir aynı üçlü karşına çıkacak (pps, Gbps, RTT) |
| **Kuyruk eğrisi** (%95'te await patlar) | Faz 4'te PCIe, Faz 5'te ağ, Faz 7'de kapasite planlaması |
| **Kuyruk derinliği = paralellik** | Faz 4.3'te DMA, Faz 5'te ağ kuyruklarında |
| **Erişim deseni performansı belirler** | Faz 2'den geldi, Faz 5'e gidiyor |
| **Hesaplama ve depolamanın ayrılması** | Faz 7'deki mimari desenlerin temeli |
| **"Hangi arıza senaryosunu önlüyorum?"** | Faz 6 ve 7'de dayanıklılık tasarımının ana sorusu |
| **Ölç, tahmin etme** | Haritanın geri kalanının çalışma biçimi |

## Faz 4 bunun neresine bağlanıyor

Faz 3 boyunca bir şeyi görmezden geldik: **veri diskten CPU'ya nasıl gidiyor?**

"NVMe PCIe üzerinden çalışır" dedik (3.3.2) ama PCIe'nin ne olduğunu açıklamadık. "DMA
sayesinde CPU maliyeti düşük" dedik ama DMA'yı tanımlamadık. Faz 4 tam olarak bu boşluğu
dolduruyor.

| Faz 3'te söylediğimiz | Faz 4'te açılacak |
|---|---|
| "NVMe PCIe üzerinden çalışır" | PCIe nedir, şerit (lane) ve nesil ne anlama gelir |
| "NVMe'nin CPU maliyeti düşük" | DMA — veriyi CPU'ya uğratmadan taşımak |
| "Kuyruk derinliği paralellik demek" | Kesme (interrupt) ve yoklama (polling) |
| "Instance store sunucunun içinde" | Cihazların anakarta bağlanma topolojisi |
| Kuyruk eğrisi | Aynı eğri PCIe ve bellek yolunda da geçerli |

> **Faz 4, haritanın en kısa fazıdır ve büyük kısmı `[kavram]` düzeyindedir.** Ama Faz 6
> (sanallaştırma) için zorunlu bir ön koşuldur: SR-IOV, virtio ve AWS Nitro'nun ne yaptığı,
> PCIe ve DMA bilinmeden anlaşılamaz.

---

*Faz 3 tamamlandı.* → **[Faz 4 — Sistem Bus'ları ve I/O](Faz_4_Bus_ve_IO.md)**
