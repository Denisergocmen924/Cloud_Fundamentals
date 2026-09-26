# Faz 4 — Sistem Bus'ları ve I/O

> **Navigasyon:** [◀ Faz 3 — Depolama](Faz_3_Depolama.md) · **Faz 4** · [Faz 5 — Ağ Donanımı ▶](Faz_5_Ag_Donanimi.md)

---

> ### Bu faz kısadır — ama atlanamaz.
>
> Faz 0–3'te parçaları tek tek inceledik: CPU, bellek, disk. Hiçbirini **birbirine nasıl
> bağlandıkları** açısından ele almadık. Faz 4 o boşluğu dolduruyor.
>
> Büyük kısmı `[kavram]` düzeyindedir ve doğrudan günlük karar üretmez. Ama **Faz 6
> (sanallaştırma) bu fazı ön koşul olarak kabul eder:** SR-IOV, virtio ve AWS Nitro'nun
> ne yaptığı, PCIe ve DMA bilinmeden anlaşılamaz. Ayrıca GPU ve yüksek paket hızlı ağ
> darboğazlarının açıklaması burada.

---

## Nereden geliyoruz

Faz 3'te iki cümle kurduk ama açıklamadık:

| Faz 3'te söylediğimiz | Buradaki karşılığı |
|---|---|
| "NVMe **PCIe** üzerinden çalışır" (3.3.2) | 4.2 — PCIe nedir |
| "NVMe'nin komut başına **CPU maliyeti düşük**" (3.3.2) | 4.3 — DMA |
| "Kuyruk derinliği paralelliği belirler" (3.2.5) | 4.4 — kesme ve yoklama |
| "Instance store sunucunun **içinde**" (3.3.4) | 4.5 — cihaz topolojisi |

Ve Faz 2'den taşıdığımız bir gerçek burada tekrar edecek: **paylaşılan bir yol, çekirdek
sayısı arttıkça darboğaza dönüşür.** Bellekte kanal, depolamada kuyruk, burada şerit
(lane).

---

## Bu fazın sonunda

- Bir bilgisayarın parçalarının birbirine nasıl bağlandığını anlatabileceksin
- PCIe şerit ve nesil hesabını yapıp bir cihazın tavanını bulabileceksin
- Neden GPU instance'larında veri transferinin darboğaz olabildiğini açıklayabileceksin
- DMA'nın olmadığı bir dünyanın neden çalışmayacağını göreceksin
- Yüksek paket hızlı ağda kesme maliyetinin neden problem olduğunu anlayacaksın
- Faz 6'daki sanallaştırma donanımını okumaya hazır olacaksın

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden önemli |
|---|---|---|---|
| 4.1 | Bus kavramı | `[kavram]` | Tüm iletişimin ortak dili |
| 4.2 | PCIe | `[mekanizma]` | Modern sistemin omurgası; GPU/NVMe/NIC hepsi burada |
| 4.3 | DMA | `[mekanizma]` | Nitro'nun ve yüksek performanslı I/O'nun temeli |
| 4.4 | Kesme (interrupt) | `[mekanizma]` / `[kavram]` | Yüksek paket hızında darboğazın kaynağı |
| 4.5 | Chipset mimarisi | `[kavram]` | Topolojiyi görmek; NUMA ile bağlantı |

---

# 4.1 Bus — veri yolu

## 4.1.1 Bus nedir `[kavram]`

**Bus**, birden fazla bileşenin paylaştığı bir iletişim yoludur. Her bileşeni her
bileşene ayrı kabloyla bağlamak yerine ortak bir hat kullanılır.

Klasik bus üç gruptan oluşur:

| Bus tipi | Ne taşır | Genişliği ne belirler |
|---|---|---|
| **Address bus** | "Hangi adres?" | Adreslenebilir bellek miktarı |
| **Data bus** | "Hangi veri?" | Tek seferde taşınan bit sayısı |
| **Control bus** | "Oku mu yaz mı, hazır mı?" | Koordinasyon sinyalleri |

> Faz 0.1.5'te 32-bit/64-bit konuşurken adres genişliğinin adreslenebilir belleği
> belirlediğini görmüştük. Adres bus'ı tam olarak o genişliğin fiziksel karşılığıdır.

## 4.1.2 Paylaşımlı bus'ın problemi ve terk edilişi `[kavram]`

Eski bilgisayarlarda gerçek bir **paylaşımlı** bus vardı: tüm cihazlar aynı hatta
bağlıydı (ISA, PCI). Bunun iki temel problemi vardı:

1. **Yarışma (arbitration).** Aynı anda sadece bir cihaz konuşabilir. Diğerleri bekler.
2. **Elektriksel sınır.** Hat uzadıkça ve cihaz sayısı arttıkça sinyal bozulur; frekans
   yükseltilemez.

> **Bu, Faz 2.4.3 ve Faz 3.3.2'deki desenin üçüncü örneğidir.** Tek bir paylaşımlı kaynak
> → yarışma → darboğaz. Bellekte çözüm çok kanal oldu, depolamada çok kuyruk oldu.
> Burada çözüm **noktadan noktaya bağlantı** oldu.

**Modern çözüm: bus değil, anahtarlanmış (switched) nokta-nokta bağlantı.**

PCI Express (PCIe) adında "bus" geçmesine rağmen aslında bir bus değildir: her cihazın
CPU'ya (veya bir switch'e) **kendi özel bağlantısı** vardır.

```
ESKİ (paylaşımlı bus):          MODERN (PCIe, anahtarlanmış):

  CPU                             CPU
   │                            ┌──┼──┬──────┐
  ═╪═══╪═══╪═══╪═  ← tek hat    │  │  │      │
   │   │   │   │               GPU NVMe NIC  ...
  GPU NVMe NIC ...              ↑ her biri kendi şeritlerinde
   ↑ sırayla konuşurlar           aynı anda tam hızda
```

> **Aynı çözüm, ağda da geçerlidir ve Faz 5.2'de göreceksin:** eski Ethernet hub'ları
> paylaşımlı bir hattı, modern switch'ler nokta-nokta bağlantıları kullanır. **Aynı
> problem, aynı çözüm, farklı katman** — haritanın tekrar eden deseni.

---

# 4.2 PCIe — modern sistemin omurgası

## 4.2.1 Şerit (lane) kavramı `[mekanizma]`

PCIe'nin temel birimi **şerit**tir. Bir şerit, iki çift diferansiyel kablodan oluşur:
biri gönderme, biri alma. Yani **her şerit tam çift yönlüdür (full duplex).**

Cihazlar birden fazla şerit kullanabilir:

| Yapılandırma | Şerit sayısı | Tipik cihaz |
|---|---|---|
| x1 | 1 | Ses kartı, basit genişletme |
| x4 | 4 | **NVMe SSD** |
| x8 | 8 | Yüksek hızlı ağ kartı (25–100 Gbps) |
| x16 | 16 | **GPU** |

**Şeritler paraleldir:** x4 bağlantı, x1'in dört katı bant genişliği verir.

## 4.2.2 Nesiller ve bant genişliği hesabı `[mekanizma]`

Her PCIe nesli şerit başına hızı yaklaşık ikiye katlar:

| Nesil | Yıl | Şerit başına (tek yön) | x4 (NVMe) | x16 (GPU) |
|---|---|---|---|---|
| Gen 3 | 2010 | ~1,0 GB/s | ~4 GB/s | ~16 GB/s |
| Gen 4 | 2017 | ~2,0 GB/s | ~8 GB/s | ~32 GB/s |
| Gen 5 | 2019 | ~4,0 GB/s | ~16 GB/s | ~64 GB/s |
| Gen 6 | 2022 | ~8,0 GB/s | ~32 GB/s | ~128 GB/s |

**Hesap kuralı:**
```
Bant genişliği ≈ Şerit sayısı × Nesil hızı  (her yön için ayrı)
Örnek: Gen4 x4 NVMe = 4 × 2,0 = 8 GB/s
```

**Faz 3'e bağlanıyor:** Bir NVMe SSD'nin 7.000 MB/s sıralı okuma yapabilmesi (3.3.3),
Gen4 x4 bağlantısının 8 GB/s tavanıyla mümkündür. Aynı SSD'yi Gen3 x4'e takarsan tavan
4 GB/s'e iner ve **disk yavaşlamaz, yol darlaşır.**

> **Bu, pratikte sık karşılaşılan bir sürprizdir:** "Yeni SSD aldım ama ilan edilen hızı
> alamıyorum." Sebep genelde diskin kendisi değil, takıldığı yuvanın nesli veya şerit
> sayısıdır. Anakartlarda bazı M.2 yuvaları x2 çalışır veya bir yuvayı doldurunca başka
> bir yuva şerit kaybeder — çünkü **CPU'nun toplam şerit sayısı sınırlıdır.**

## 4.2.3 Şerit bütçesi — sınırlı bir kaynak `[mekanizma]`

Bir CPU'nun sağladığı toplam PCIe şerit sayısı sabittir:

```
Masaüstü CPU:     ~20–28 şerit
Sunucu CPU:       ~64–128 şerit (soket başına)
```

Ve bunlar paylaşılır:
```
Sunucu örneği (128 şerit):
  2 × GPU      = 32 şerit
  4 × NVMe     = 16 şerit
  2 × 100G NIC = 32 şerit
  Chipset      = 16 şerit
  ─────────────────────────
  Toplam         96 şerit  (32 kaldı)
```

> **Şerit sayısı, sunucu CPU'larını masaüstü CPU'larından ayıran en önemli özelliklerden
> biridir** — çekirdek sayısı kadar önemlidir ama çok daha az konuşulur. Çok sayıda NVMe
> ve ağ kartı barındıran bir sunucu, çekirdek sayısı yetse bile şerit bütçesi yüzünden
> yapılandırılamayabilir.
>
> Faz 2.4.3'teki bellek kanalı argümanının birebir aynısı: **sunucu sınıfı donanımın
> farkı sadece "daha güçlü" olması değil, daha çok paralel yola sahip olmasıdır.**

## 4.2.4 Cloud bağlantısı — GPU instance'larında darboğaz `[uygulama]`

GPU'lu bir iş yükünde veri şu yolu izler:

```
Depolama → CPU belleği (RAM) → PCIe → GPU belleği (VRAM) → hesaplama
                                 ↑
                          BURASI DARBOĞAZ
```

Sayılarla:
```
GPU'nun kendi bellek bant genişliği : ~2.000–3.000 GB/s  (HBM)
PCIe Gen4 x16 üzerinden veri aktarımı:      ~32 GB/s
                                              ↑ ~70 kat dar
```

**Sonuç:** GPU, veriyi beklemekten hesaplamaktan daha çok zaman harcayabilir.

> **Bu, ML eğitiminde "GPU kullanımı %30" şikâyetinin en yaygın sebebidir.** GPU
> yavaş değil — **besleme hattı (data pipeline) yetişemiyor.** Veri diskten okunuyor,
> CPU'da ön işleniyor, PCIe'den geçiyor... ve GPU bekliyor.
>
> Çözümler donanım değiştirmekle ilgili değildir: veri yükleyicide daha çok işçi
> (worker), önceden getirme (prefetch), veriyi GPU belleğinde tutmak, daha büyük batch,
> ön işlemeyi GPU'ya taşımak, veriyi sıkıştırılmış/uygun formatta saklamak.
>
> **Faz 2.3.6 ve Cevap 3.3'teki dersin üçüncü tekrarı:** darboğaz genelde en pahalı
> bileşende değil, onu besleyen yoldadır.

**NVLink.** NVIDIA, GPU'lar arası iletişim için PCIe'yi atlayan özel bir bağlantı sunar
(~600–900 GB/s). Çok GPU'lu eğitimde GPU'lar birbirleriyle sürekli gradyan alışverişi
yapar; PCIe üzerinden bu mümkün olmazdı. AWS'in `p4d`/`p5` gibi aileleri NVLink'li
GPU'lar içerir — **"8 GPU'lu instance" ile "NVLink'li 8 GPU'lu instance" çok farklı
şeylerdir.**

![Şekil 4.1 — PCIe topolojisi: CPU, şeritler ve bağlı cihazlar](../diagrams/png/hw-4-01-pcie-topology.png)
*Şekil 4.1 — CPU'dan çıkan şeritlerin GPU, NVMe, NIC ve chipset arasında dağılımı.*

> **🤔 Düşün 4.1**
> Bir ML ekibi, eğitim işlerinin yavaş olduğunu söylüyor. `nvidia-smi` çıktısında GPU
> kullanımı sürekli %25–35 arasında dalgalanıyor, GPU belleği ise neredeyse dolu.
>
> a) Sorun GPU'nun gücü mü? Nasıl anlarsın?
> b) Daha pahalı GPU'lu bir instance'a geçmek işe yarar mı?
> c) Hangi üç yeri incelerdin?
> *(Cevap: fazın sonunda)*

---

# 4.3 DMA — CPU'yu aradan çıkarmak

## 4.3.1 DMA olmasaydı `[kavram]`

Bir diskten 1 MB okuduğunu düşün. DMA olmasaydı akış şöyle olurdu:

```
CPU: "Disk, bana ilk kelimeyi ver"
Disk: [veri]
CPU: veriyi al → RAM'e yaz
CPU: "Sıradaki kelimeyi ver"
Disk: [veri]
CPU: veriyi al → RAM'e yaz
... 262.144 kez (1 MB ÷ 4 byte)
```

Buna **programlı I/O (PIO)** denir ve iki felaketi vardır:

1. **CPU tamamen meşgul.** Veri transferi boyunca başka hiçbir iş yapamaz.
2. **CPU en yavaş bileşenin hızında çalışır.** Faz 1'de inşa ettiğimiz pipeline, dal
   tahmini, süperskalar yürütme — hepsi diskin hızında beklemeye indirgenir.

Tek bir dosya kopyalama işlemi sunucuyu kilitlerdi. Modern sistemler bu şekilde
çalışamazdı.

## 4.3.2 DMA nasıl çalışır `[mekanizma]`

**DMA (Direct Memory Access)**, cihazların **CPU'ya uğramadan** doğrudan RAM'e yazmasını
sağlayan bir denetleyicidir.

```
1. CPU  → DMA denetleyicisine talimat:
          "Diskten 1 MB oku, RAM'de 0x7f3a... adresine yaz,
           bitince haber ver"

2. CPU  → BAŞKA İŞE GEÇER  ← kritik nokta

3. DMA  → Disk ile RAM arasında veriyi taşır (CPU dahil değil)

4. DMA  → Bitince CPU'ya KESME (interrupt) gönderir

5. CPU  → "Veri hazır" — işlemeye devam eder
```

**CPU'nun toplam katılımı: iki kısa an.** Başlatma ve tamamlanma bildirimi. Arada geçen
milisaniyelerde CPU başka thread'leri çalıştırır.

> **DMA olmadan modern bilgisayar mümkün değildir** — ve bu abartı değil. 10 Gbps ağ
> trafiğini PIO ile almak, tek başına birden fazla CPU çekirdeğini tüketirdi. Kullanıcıya
> hiç çekirdek kalmazdı.

**DMA ve cache tutarlılığı.** Cihaz RAM'e doğrudan yazınca, o adresin CPU cache'indeki
kopyası eskir (Faz 2.3.8'deki problem, bu kez cihazla CPU arasında). Modern sistemlerde
donanım bunu halleder (**cache-coherent DMA**); halletmediği durumlarda işletim
sisteminin ilgili cache satırlarını geçersiz kılması gerekir.

## 4.3.3 Cloud bağlantısı — AWS Nitro `[uygulama]`

**Geleneksel sanallaştırmada** (Faz 6.1'de detaylandıracağız) ağ ve depolama I/O'su
hypervisor yazılımından geçerdi:

```
Misafir VM → hypervisor (YAZILIM) → fiziksel cihaz
                    ↑
          CPU çevrimi tüketir, gecikme ekler
```

Bu, sunucu CPU'sunun **%20–30'una** mal olabiliyordu. Yani satabileceğin kapasitenin
üçte biri sanallaştırma vergisi olarak gidiyordu.

**AWS Nitro'nun yaptığı:** Bu işi ayrı bir donanım kartına taşımak.

```
Misafir VM → Nitro kartı (AYRI DONANIM) → ağ / depolama
                    ↑
          Ana CPU'nun neredeyse hiç katılımı yok
```

Nitro kartı kendi işlemcisine, belleğine ve DMA motoruna sahiptir. Ana sunucunun CPU'su
neredeyse tamamen müşteriye ayrılır.

| | Geleneksel hypervisor | Nitro |
|---|---|---|
| I/O işleme | Ana CPU'da yazılım | Ayrı donanım kartı |
| Sanallaştırma vergisi | %20–30 | **%1'in altı** |
| Ağ gecikmesi | Yüksek, değişken | Düşük, tutarlı |
| Güvenlik sınırı | Yazılım | **Fiziksel ayrım** |

> **Ve bu, cloud'da bir mimari kararın donanıma dönüştüğü en net örnektir.** AWS,
> yazılımla çözülen bir problemi donanıma taşıyarak hem müşteriye daha çok kapasite
> sattı hem de güvenlik sınırını güçlendirdi. `.metal` instance tiplerinin var
> olabilmesi de buna dayanır — hypervisor olmadan da ağ ve depolama çalışabiliyor,
> çünkü onları hypervisor yapmıyordu.
>
> Faz 6.4'te bu konuyu sanallaştırma bağlamında tamamlayacağız.

---

# 4.4 Kesme (interrupt) — donanım dikkat çekiyor

## 4.4.1 Kesme nedir `[mekanizma]`

**Kesme**, bir donanım cihazının CPU'nun dikkatini çekme yöntemidir.

```
CPU bir programı çalıştırıyor...
        ↓
Ağ kartı: "Paket geldi!" → IRQ sinyali
        ↓
CPU: mevcut durumu kaydeder (register'lar, PC — Faz 1.1.3)
        ↓
CPU: kesme vektör tablosundan handler adresini bulur
        ↓
CPU: kesme işleyicisini (handler) çalıştırır
        ↓
CPU: durumu geri yükler, kaldığı yerden devam eder
```

**IRQ (Interrupt Request):** Her cihaz tipinin bir kesme numarası vardır; işletim sistemi
hangi handler'ı çağıracağını böyle bilir.

**Kesmenin maliyeti — ve bu maliyet Faz 1'e dayanır:**

| Maliyet kalemi | Neden |
|---|---|
| Bağlam kaydetme/geri yükleme | ~1–2 μs |
| **Pipeline boşalması** | Faz 1.4 — dolu pipeline atılır |
| **Cache kirlenmesi** | Faz 2.3 — handler'ın verisi, programın verisini atar |
| Kullanıcı/çekirdek modu geçişi | Faz 1.4.4 sonrası Spectre azaltmaları bunu pahalılaştırdı |

> **Kesmenin gerçek maliyeti, bağlam değiştirmenin kendisi değil, Faz 1 ve 2'de
> öğrendiğin dolaylı zararlardır:** pipeline yeniden dolmalı, cache ısınmalı, dal
> tahmincisi yeniden öğrenmeli. Bu yüzden mikrosaniyelik bir işlem, etkisini
> mikrosaniyelerce sonra da hissettirir.

## 4.4.2 Yoklama (polling) ve kesme fırtınası `[kavram]`

**Yoklama (polling):** CPU cihazı düzenli olarak kendisi kontrol eder.

| | Kesme | Yoklama |
|---|---|---|
| Olay **seyrekse** | ✅ Verimli — CPU boşta kalmaz | ❌ Boşuna kontrol, çevrim israfı |
| Olay **sıksa** | ❌ **Kesme fırtınası** | ✅ Verimli — zaten hep veri var |
| Gecikme | Düşük | Yoklama aralığına bağlı |

**Kesme fırtınası (interrupt storm)** — somut bir hesap:

```
10 Gbps hat, 1500 byte paketler:
  10.000.000.000 bit/s ÷ (1500 × 8 bit) ≈ 833.000 paket/saniye

Her paket için ayrı kesme olsaydı:
  833.000 kesme/s × ~2 μs = 1,67 saniye/saniye
  → BİR ÇEKİRDEK YETMEZ. Sistem paket işlemekten başka iş yapamaz.
```

Küçük paketlerle (64 byte) durum çok daha kötüdür: ~14,8 milyon paket/saniye.

**Çözüm: NAPI — melez yaklaşım.**

Linux'un ağ katmanı ikisini birleştirir:
```
Trafik düşükken  → kesme modu (CPU boşta kalabilsin, gecikme düşük olsun)
Paket gelince    → kesmeleri KAPAT, yoklama moduna geç
Kuyruk boşalınca → kesmeleri tekrar aç
```

Böylece düşük trafikte kesmenin gecikme avantajı, yüksek trafikte yoklamanın verim
avantajı elde edilir.

> **Bu "trafik yoğunken toplu işle, seyrekken anında tepki ver" deseni her katmanda
> tekrar eder:** Cevap 3.1'deki log tamponlama, veritabanlarındaki group commit, Kafka'nın
> batching'i, TCP'nin Nagle algoritması. **Aynı takas: gecikme mi, verim mi.**

Faz 5.1.4'te NAPI'yi ağ kartı bağlamında tekrar göreceğiz.

## 4.4.3 Kesme dağıtımı ve CPU yakınlığı `[uygulama]`

Çok çekirdekli sistemde kesmeler tek bir çekirdeğe düşerse o çekirdek boğulur.

**MSI-X** (Message Signaled Interrupts eXtended) sayesinde modern cihazlar **birden fazla
kesme hattı** sunar ve bunlar farklı çekirdeklere dağıtılabilir.

```bash
cat /proc/interrupts | head -20
```
**Beklenen çıktı (ağ kartı satırları):**
```
            CPU0       CPU1       CPU2       CPU3
 24:     1245678          0          0          0   PCI-MSI  eth0-rx-0
 25:           0    1198234          0          0   PCI-MSI  eth0-rx-1
 26:           0          0    1201456          0   PCI-MSI  eth0-rx-2
 27:           0          0          0    1189012   PCI-MSI  eth0-rx-3
              ↑ dengeli dağılım = sağlıklı
```

Tüm sayılar tek bir sütunda toplanmışsa, o çekirdek darboğazdır.

> **Cloud bağlantısı:** Yüksek paket hızlı iş yüklerinde (yük dengeleyici, API ağ geçidi,
> proxy) kesme dengesizliği **gerçek bir darboğazdır ve toplam CPU metriğinde görünmez** —
> `top`'ta ortalama %30 CPU görürsün ama bir çekirdek %100'dedir. `mpstat -P ALL 1`
> ile çekirdek bazında bakmak gerekir.
>
> Faz 5.1.4'te bu konuyu RSS (Receive Side Scaling) ile tamamlayacağız.

> **🤔 Düşün 4.2**
> Bir API ağ geçidi sunucusunda `top` ortalama %35 CPU gösteriyor ama gecikme yüksek ve
> paket kaybı var.
>
> a) Bu faz ve önceki fazlardan hangi üç sebebi araştırırsın?
> b) Hangi komutlarla doğrularsın?
> *(Cevap: fazın sonunda)*

---

# 4.5 Chipset ve anakart mimarisi `[kavram]`

## 4.5.1 Tarihsel yapı: Northbridge / Southbridge

Eskiden CPU dış dünyaya iki çip üzerinden bağlanırdı:

```
        CPU
         │
    ┌────┴────┐
    │Northbr. │──── RAM        ← HIZLI cihazlar
    │         │──── AGP/PCIe (grafik)
    └────┬────┘
         │
    ┌────┴────┐
    │Southbr. │──── SATA, USB, ses, ağ, BIOS   ← YAVAŞ cihazlar
    └─────────┘
```

**Northbridge** hızlı olanları (bellek, grafik), **Southbridge** yavaş olanları yönetirdi.

## 4.5.2 Modern yapı: entegrasyon

Bugün Northbridge'in işlevi **CPU'nun içine taşındı:**

```
   ┌─────────────────────────────┐
   │  CPU                        │
   │  ├─ Çekirdekler + cache     │
   │  ├─ BELLEK DENETLEYİCİSİ    │──── RAM (doğrudan)
   │  └─ PCIe DENETLEYİCİSİ      │──── GPU, NVMe, NIC (doğrudan)
   └──────────────┬──────────────┘
                  │ (DMI / Infinity Fabric)
          ┌───────┴────────┐
          │ Chipset (PCH)  │──── USB, SATA, ses, düşük hızlı PCIe
          └────────────────┘
```

**Neden bellek denetleyicisi CPU'ya taşındı?** Faz 1.3.1'deki hesap: 3 GHz'de bir çevrimde
sinyal ~5–7 cm yol alır. Bellek denetleyicisi ayrı bir çipteyken her erişim o çipe gidip
dönüyordu — **ek 20–30 çevrim.** İçeri taşımak doğrudan gecikme kazancı sağladı.

> **Ve bu, Faz 2.7'deki NUMA'nın sebebidir.** Bellek denetleyicisi CPU'ya taşındığı için,
> iki soketli bir sistemde **iki ayrı bellek denetleyicisi** olur — ve her biri kendi
> RAM'ine bağlıdır. Uzak sokete erişmek, diğer CPU'nun denetleyicisinden geçmek demektir.
>
> **NUMA, entegrasyonun kaçınılmaz sonucudur.** Faz 2'de NUMA'yı bir olgu olarak
> öğrenmiştin; şimdi neden var olduğunu biliyorsun.

**Soketler arası bağlantı:** Intel'de **UPI**, AMD'de **Infinity Fabric**. Bunlar bir
CPU'nun diğerinin belleğine ve cihazlarına erişmesini sağlar — Faz 2.7.1'deki şemadaki
oktur.

> **Cloud'da bunun görünümü:** Bir cihaz (NVMe, NIC) fiziksel olarak bir sokete bağlıdır.
> O cihazı kullanan iş yükü diğer sokette çalışırsa, **her I/O soketler arası bağlantıyı
> geçer.** Bu, "NUMA-aware I/O" konusunun temelidir ve `.metal` veya çok büyük
> instance'larda ölçülebilir fark yaratır.

![Şekil 4.2 — Tarihsel Northbridge/Southbridge yapısı ve modern entegre mimarinin karşılaştırması](../diagrams/png/hw-4-02-chipset-evolution.png)
*Şekil 4.2 — Bellek ve PCIe denetleyicilerinin CPU'ya taşınması ve bunun NUMA ile ilişkisi.*

---

# 4.6 Bu faz bozulunca — arıza imzaları

| Belirti | Olası mekanizma | Nerede | İlk bakılacak |
|---|---|---|---|
| NVMe ilan edilen hızın yarısını veriyor | Yuva Gen3 veya x2 çalışıyor | 4.2.2 | `lspci -vv` LnkSta satırı |
| GPU kullanımı %30, VRAM dolu | PCIe / veri hattı darboğazı | 4.2.4 | Veri yükleyici, batch boyutu |
| `top` %35 ama bir çekirdek %100 | Kesme tek çekirdekte toplanmış | 4.4.3 | `mpstat -P ALL 1`, `/proc/interrupts` |
| Yüksek paket hızında CPU tükeniyor | Kesme fırtınası | 4.4.2 | NAPI/coalescing ayarları |
| Büyük instance'ta I/O beklenenden yavaş | NUMA'ya uzak cihaz | 4.5.2 | `lstopo`, cihaz-soket yakınlığı |
| Sanallaştırma katmanı CPU yiyor | Yazılım I/O yolu (Nitro öncesi mimari) | 4.3.3 | Nitro tabanlı nesle geç |

> **🔧 PCIe bağlantını doğrula** (isteğe bağlı)
> ```bash
> lspci | grep -i -E "nvme|ethernet|vga"
> sudo lspci -vv -s <adres> 2>/dev/null | grep -E "LnkCap|LnkSta"
> ```
> **Beklenen çıktı:**
> ```
> LnkCap: Speed 16GT/s, Width x4      ← cihazın DESTEKLEDİĞİ
> LnkSta: Speed 16GT/s, Width x4      ← şu an ÇALIŞTIĞI
> ```
> İkisi farklıysa (örn. `LnkCap: 16GT/s` ama `LnkSta: 8GT/s`) cihaz düşük hızda
> çalışıyor demektir — yuva, kablo veya güç yönetimi kaynaklı. Sanal makinelerde bu
> bilgi genelde görünmez; bare metal ve `.metal` instance'larda anlamlıdır.

---
# Faz 4 — Düşün sorularının cevapları

## Cevap 4.1 — GPU kullanımı %25–35, VRAM dolu

**a) Sorun GPU'nun gücü değil. Kanıt, sorunun kendi içinde.**

GPU kullanımı **%100'e çıkamıyor** ve **dalgalanıyor.** Bu iki gözlem birlikte tek bir
şeyi söyler: **GPU bekliyor.** Eğer GPU gücü yetersiz olsaydı kullanım %95–100'de sabit
kalır, iş sadece uzun sürerdi.

VRAM'in dolu olması yanıltıcı bir rahatlıktır — bellek *ayrılmış* olabilir ama işlem
birimleri boşta. `nvidia-smi` çıktısında `GPU-Util` ve `Memory-Used` **farklı şeylerdir**;
biri hesaplama, diğeri tahsis.

**Nasıl kesinleştirirsin:**
```bash
nvidia-smi dmon -s pucm        # zaman içinde util/power/clock/bellek
```
- Güç tüketimi TDP'nin çok altındaysa → GPU zorlanmıyor, bekliyor
- `sm` (streaming multiprocessor) kullanımı düşük ama bellek kopyası (`mem`) yüksekse →
  transfer darboğazı

Ayrıca profilleyici (PyTorch Profiler, Nsight Systems) zaman çizelgesinde GPU'nun boş
kaldığı boşlukları doğrudan gösterir.

**b) Daha pahalı GPU işe yaramaz — muhtemelen durumu kötüleştirir.**

Daha hızlı bir GPU, aynı veriyi daha hızlı işler ve **daha uzun süre bekler.** Kullanım
oranı %30'dan %15'e düşer, toplam süre neredeyse hiç kısalmaz, maliyet iki katına çıkar.

> **Bu, Faz 2.3.6, Cevap 3.3 ve 4.2.4'ün aynı dersidir:** darboğaz olmayan bileşeni
> güçlendirmek para harcar, sorunu çözmez. Amdahl'ın yasasının pratik hâli.

**c) İncelenecek üç yer — ve sırası:**

**1. Veri yükleme hattı (en sık sebep)**
```python
DataLoader(dataset,
    num_workers=8,        # tek işçi CPU'da darboğaz yapar
    pin_memory=True,      # sabitlenmiş bellek → DMA daha verimli (4.3.2)
    prefetch_factor=4,    # GPU işlerken bir sonraki batch hazırlansın
    persistent_workers=True)
```
`num_workers=0` (varsayılan) ise ön işleme ana thread'de yapılır ve GPU her batch
arasında bekler. Bu tek satır çoğu zaman 2–4 kat kazandırır.

**2. Depolama hattı (Faz 3)**
- Veri EBS'te ve küçük dosyalar hâlindeyse → **IOPS sınırlı** (3.4.3). Milyonlarca küçük
  görüntü dosyası, klasik ML darboğazıdır.
- Çözüm: veriyi paketli formatlarda sakla (TFRecord, WebDataset, Parquet) → küçük rastgele
  okuma yerine **büyük sıralı okuma** (3.1.3'ün dersi)
- Veya veriyi instance store'a (yerel NVMe) kopyala — tam olarak SSS S5'teki
  "yeniden üretilebilir veri" durumu

**3. PCIe transferi (4.2.4)**
- Batch boyutunu artır → transfer başına daha çok iş
- Ön işlemeyi GPU'ya taşı (NVIDIA DALI, GPU'da augmentation)
- `pin_memory=True` → sayfalanabilir bellekten kopyalama adımını atlar
- Mümkünse veri setini tamamen VRAM'de tut

> **Sıralama önemli:** 1 ve 2 bedava veya ucuzdur ve çoğu vakayı çözer. 3 kod
> değişikliği gerektirir. Instance yükseltmesi listede **yok** — çünkü darboğaz orada
> değil.

## Cevap 4.2 — `top` %35 ama gecikme yüksek ve paket kaybı var

**a) Üç sebep — hepsi "ortalama metrik yalan söyler" ailesinden:**

**1. Kesme dengesizliği (4.4.3) — en olası.**
Ağ kesmeleri tek bir çekirdeğe düşüyorsa o çekirdek %100'de, diğerleri boşta. 8
çekirdekli bir makinede bir çekirdeğin doymuş olması, ortalamada **%12,5** olarak görünür.
`top`'un gösterdiği %35, içinde bir %100 barındırıyor olabilir.

**2. Kesme fırtınası ve softirq yükü (4.4.2).**
Yüksek paket hızında CPU zamanının büyük kısmı `si` (softirq) kategorisine gider. Bu
zaman `top`'un genel yüzdesinde görünür ama **kullanıcı işine ayrılmış gibi okunmaz** —
sistem paket işlemekten uygulamayı çalıştıramıyordur.

**3. Kuyruk doygunluğu — CPU'yla ilgisiz (3.4.4'ün eğrisi).**
NIC halka tamponu (ring buffer) dolmuş olabilir. Paketler düşüyor ama CPU meşgul değil,
çünkü paketler CPU'ya hiç ulaşmıyor. Aynı şekilde bağlantı havuzu, thread havuzu veya
`somaxconn` kuyruğu dolmuş olabilir.

**Dördüncü ihtimal (Faz 2'den):** Cache/bellek darboğazı — CPU "meşgul" görünmüyor ama
IPC düşük. Bu senaryoda daha az olası, yine de tabloda tut.

**b) Doğrulama komutları — sırayla:**

```bash
# 1. Çekirdek bazında dağılım — ortalamanın gizlediği
mpstat -P ALL 1 5
```
**Beklenen (problem varsa):**
```
CPU   %usr  %sys  %soft  %idle
all   18,2   9,1   7,8    64,9      ← ortalama masum görünüyor
  0    5,1   3,2  91,5     0,2      ← CPU0 softirq'te BOĞULMUŞ
  1   21,0  10,1   0,1    68,8
  2   20,5   9,8   0,0    69,7
```

```bash
# 2. Kesmeler hangi çekirdekte
cat /proc/interrupts | grep -E "eth|ens|enp"

# 3. NIC seviyesinde düşen paket var mı
ip -s link show eth0
ethtool -S eth0 | grep -i -E "drop|discard|error|miss|overrun"

# 4. Halka tamponu doluyor mu
ethtool -g eth0

# 5. Soket seviyesi kuyruklar
ss -s
netstat -s | grep -i -E "overflow|dropped|pruned"
```

**Tipik bulgu ve çözümleri:**

| Bulgu | Çözüm |
|---|---|
| Tek çekirdekte softirq %90+ | **RSS/RPS ile kesmeleri dağıt** (Faz 5.1.4) |
| `rx_dropped` artıyor | Halka tamponunu büyüt (`ethtool -G`) |
| Kesme sayısı çok yüksek | **Interrupt coalescing** aç (`ethtool -C`) — gecikme karşılığı verim |
| Hepsi normal, yine de kayıp | Uygulama katmanına bak: kuyruklar, thread havuzu, GC duraklamaları |

> **Bu sorunun asıl dersi:** Ortalama metrikler dağılımı gizler. `top`'un %35'i,
> "kapasitenin %65'i boşta" demek **değildir** — tek bir çekirdeğin doymuş olması sistemi
> kilitlemeye yeter.
>
> Aynı hata her katmanda yapılır: ortalama gecikme iyi ama p99 berbat; ortalama disk
> kullanımı düşük ama bir birim doygun; ortalama bellek boş ama bir NUMA düğümü dolu.
> **Ortalamaya değil, dağılımın kuyruğuna bak.**

---

# Faz 4 — Sık sorulan sorular

> **S1: PCIe nesli gerçekten fark eder mi? Gen3 yeterli değil mi?**
>
> Cihaza bağlı:
> - **NVMe SSD (x4):** Gen3 = 4 GB/s. Üst düzey bir SSD 7 GB/s yapabiliyorsa **yarısını
>   kaybedersin.** Fark eder.
> - **GPU (x16):** Gen3 = 16 GB/s. Model VRAM'e sığıyor ve transfer az ise fark küçüktür;
>   veri sürekli akıyorsa (4.2.4) belirgin fark eder.
> - **Ağ kartı:** 25 Gbps ≈ 3 GB/s → Gen3 x8 fazlasıyla yeter. 100 Gbps için Gen4
>   gerekir.
>
> **Kural:** Cihazın ihtiyacını hesapla, yolun kapasitesiyle karşılaştır. Yol cihazdan
> genişse nesil önemsizdir.

> **S2: Cloud instance'ında PCIe'yi görebilir miyim?**
>
> Kısmen. `lspci` cihazları listeler ama sanallaştırma katmanı gerçek topolojiyi gizler —
> gördüğün şey çoğu zaman sanal bir cihazdır (virtio) veya SR-IOV ile sunulmuş bir sanal
> fonksiyondur (Faz 6.5).
>
> **`.metal` instance'larda** gerçek donanımı görürsün: `lspci -vv`, `lstopo`, PCIe
> bağlantı hızları, NUMA topolojisi. Bu, `.metal` tiplerinin donanım-duyarlı iş yükleri
> için tercih edilme sebeplerinden biridir.
>
> Pratikte: normal instance'larda PCIe'yi ayarlayamazsın; **bilgi, davranışı açıklamak
> için işe yarar, müdahale etmek için değil.**

> **S3: Interrupt coalescing'i açmalı mıyım?**
>
> **Coalescing**, NIC'in birden fazla paketi biriktirip tek kesme üretmesidir. Klasik
> gecikme/verim takası:
>
> | | Coalescing kapalı | Coalescing açık |
> |---|---|---|
> | Gecikme | **Düşük** | Yüksek (+10–200 μs) |
> | CPU maliyeti | Yüksek | **Düşük** |
> | Yüksek pps'te | CPU tükenir | Dayanır |
>
> **Karar:** Yüksek verim odaklı iş yükü (veri aktarımı, yedekleme, toplu iş) → aç.
> Düşük gecikme odaklı (ticaret sistemi, gerçek zamanlı oyun, düşük gecikmeli API) →
> kapat veya agresif ayarla.
>
> Cloud'da genelde sağlayıcının varsayılanı makuldür; değiştirmeden önce **ölç.**

> **S4: DMA bir güvenlik riski değil mi? Cihaz RAM'in her yerine yazabiliyorsa?**
>
> Evet, gerçek bir risktir ve adı vardır: **DMA saldırıları** (Thunderbolt üzerinden
> yapılan saldırılar meşhur örnektir).
>
> **Çözüm: IOMMU** (Intel VT-d, AMD-Vi). Cihazlar için bir MMU'dur — Faz 2.5'teki sanal
> belleğin cihaz versiyonu. Her cihaza yalnızca izin verilen bellek bölgelerine erişim
> tanır.
>
> **Ve IOMMU cloud için kritiktir:** Faz 6.6.3'te göreceğin SR-IOV, bir fiziksel cihazı
> birden fazla VM'e doğrudan vermeyi sağlar. IOMMU olmadan bir VM'in cihazı, başka bir
> VM'in belleğine DMA yapabilirdi. **IOMMU, donanım seviyesinde kiracı yalıtımını mümkün
> kılan şeydir.**

> **S5: Bu faz cloud engineer için gerçekten gerekli mi? Hiç PCIe ayarı yapmayacağım.**
>
> Ayar yapmayacaksın, doğru. Ama üç yerde doğrudan işine yarayacak:
>
> 1. **GPU/ML iş yüklerinde** — "GPU kullanımı neden düşük" sorusu bu faz olmadan
>    cevaplanamaz (Cevap 4.1), ve bu soru ML ekipleriyle çalışan her cloud engineer'ın
>    karşısına çıkar
> 2. **Yüksek paket hızlı ağda** — kesme dengesizliği ortalama metriklerde görünmez
>    (Cevap 4.2)
> 3. **Faz 6'nın tamamı için** — Nitro, SR-IOV, virtio, IOMMU: hepsi bu fazın üstüne
>    kurulu
>
> Ayrıca daha genel bir kazanç var: bu faz, **"bileşenler arasındaki yol da bir kaynaktır"**
> fikrini yerleştirir. CPU, bellek ve diski ayrı ayrı boyutlandırıp aralarındaki yolu
> unutmak, mimaride sık yapılan bir hatadır.

---

# Faz 4 — Kendini sına

**Bölüm A — Temel (1–6)**

1. Address bus, data bus ve control bus'ın taşıdığı bilgiyi yaz.
2. PCIe neden gerçek anlamda bir "bus" değildir?
3. PCIe Gen4 x4 bir NVMe'nin teorik tek yönlü bant genişliği nedir? Hesapla.
4. DMA olmasaydı 1 MB'lık bir disk okuması CPU açısından nasıl geçerdi?
5. Kesme (interrupt) ile yoklama (polling) arasındaki temel farkı ve her birinin uygun
   olduğu durumu yaz.
6. Bellek denetleyicisinin CPU'ya taşınmasının iki sonucunu söyle (biri olumlu, biri
   olumsuz).

**Bölüm B — Mekanizma (7–12)**

7. Bir GPU'nun kendi bellek bant genişliği ~2.000 GB/s iken PCIe Gen4 x16 ~32 GB/s.
   Bu fark hangi problemi doğurur?
8. Kesmenin maliyeti neden sadece bağlam değiştirme süresi değildir? Faz 1 ve 2'den iki
   sebep say.
9. 10 Gbps hatta 1500 byte paketlerle saniyede kaç paket düşer? Her paket için ayrı
   kesme neden çalışmaz?
10. NAPI nasıl çalışır ve hangi iki dünyanın iyi yanını birleştirir?
11. NUMA'nın var olmasının sebebi bu fazda hangi tasarım kararına dayanıyor?
12. IOMMU nedir ve cloud'da neden kritiktir?

**Bölüm C — Uygulama ve muhakeme (13–18)**

13. Yeni aldığın NVMe ilan edilen hızın yarısını veriyor. Hangi komutla ve neye bakarsın?
14. `top` ortalama %35 CPU gösteriyor ama sistem paket kaybediyor. Hangi komutu
    çalıştırır, ne ararsın?
15. Bir ML ekibi "daha güçlü GPU alalım" diyor, GPU kullanımı %30. Ne sorarsın?
16. Interrupt coalescing'i hangi iş yükünde açar, hangisinde kapatırsın?
17. AWS Nitro'nun çözdüğü problem neydi ve nasıl çözdü?
18. Bir cihazın performansını değerlendirirken neden sadece cihazın kendi hızına bakmak
    yetersizdir? Bu fazdan ve Faz 3'ten birer örnek ver.

---

## Cevap anahtarı

**1. Address bus:** hangi adrese erişileceği (genişliği adreslenebilir bellek miktarını
belirler). **Data bus:** taşınan verinin kendisi (genişliği tek seferde taşınan bit
sayısını belirler). **Control bus:** işlemin türü ve koordinasyon sinyalleri (oku/yaz,
hazır, kesme). *(4.1.1)*

**2.** Çünkü **paylaşımlı bir hat değil, anahtarlanmış nokta-nokta bağlantılardan** oluşur.
Her cihazın CPU'ya (veya bir switch'e) kendi özel şeritleri vardır; cihazlar aynı anda tam
hızda iletişim kurabilir. "Bus" adı tarihsel bir devamlılıktır. *(4.1.2)*

**3.** `4 şerit × 2,0 GB/s = 8 GB/s` (her yön için ayrı — PCIe tam çift yönlüdür).
*(4.2.2)*

**4.** **Programlı I/O (PIO)** ile: CPU her kelimeyi tek tek diskten okuyup RAM'e
yazardı — 1 MB için ~262.000 döngü. Bu süre boyunca CPU **başka hiçbir iş yapamaz** ve
en yavaş bileşenin hızında çalışır. Faz 1'de inşa edilen pipeline, dal tahmini ve
süperskalar yürütmenin tamamı anlamsızlaşır. *(4.3.1)*

**5. Kesme:** Cihaz hazır olduğunda CPU'ya sinyal gönderir; CPU o ana kadar başka iş
yapar. **Yoklama:** CPU cihazı düzenli olarak kendisi kontrol eder. **Olay seyrekse kesme**
verimlidir (CPU boşa dönmez); **olay sıksa yoklama** verimlidir (kesme maliyeti ödenmez).
*(4.4.2)*

**6. Olumlu:** Bellek erişim gecikmesi belirgin düştü — ayrı çipe gidiş-dönüşün ~20–30
çevrimi ortadan kalktı. **Olumsuz:** Çok soketli sistemlerde her CPU'nun kendi bellek
denetleyicisi olduğu için **NUMA** doğdu — uzak sokete erişim 1,5–2 kat pahalı hâle geldi.
*(4.5.2)*

**7.** GPU, veriyi işlemekten çok **beklemek** zorunda kalır. Hesaplama kapasitesi
besleme hızının ~70 katıdır, dolayısıyla veri sürekli akan iş yüklerinde GPU kullanımı
düşük kalır. Klasik belirtisi: `nvidia-smi`'de %25–35 kullanım. *(4.2.4)*

**8.** (a) **Pipeline boşalması** — Faz 1.4: dolu pipeline atılır ve yeniden doldurulması
onlarca çevrim alır. (b) **Cache kirlenmesi** — Faz 2.3: kesme işleyicisinin verisi,
çalışan programın cache satırlarını dışarı atar; program devam ettiğinde cache'i yeniden
ısıtması gerekir. Ayrıca dal tahmincisi de bozulur. *(4.4.1)*

**9.** `10.000.000.000 ÷ (1500 × 8) ≈ 833.000 paket/saniye.` Her paket için ~2 μs'lik bir
kesme, saniyede 1,67 saniyelik iş demektir — **bir çekirdek yetmez**, sistem paket
işlemekten başka iş yapamaz. 64 byte'lık paketlerde durum ~14,8 milyon pps ile çok daha
kötüdür. *(4.4.2)*

**10.** Trafik düşükken **kesme** modunda çalışır (CPU boşta kalabilir, gecikme düşük);
paket gelince kesmeleri **kapatıp yoklama** moduna geçer (yüksek trafikte kesme maliyeti
ödenmez); kuyruk boşalınca kesmeleri tekrar açar. Böylece **düşük trafikte kesmenin
gecikme avantajı, yüksek trafikte yoklamanın verim avantajı** elde edilir. *(4.4.2)*

**11.** **Bellek denetleyicisinin CPU'ya entegre edilmesine.** Denetleyici ayrı bir
çipteyken tüm CPU'lar aynı denetleyiciyi paylaşıyordu (tekdüze erişim). İçeri taşınınca
her soketin kendi denetleyicisi ve kendi RAM'i oldu → uzak soketin RAM'ine erişim
farklılaştı → NUMA. *(4.5.2)*

**12. IOMMU**, cihazlar için bir MMU'dur — Faz 2.5'teki sanal belleğin cihaz versiyonu.
Her cihazın DMA ile erişebileceği bellek bölgelerini sınırlar. **Cloud'da kritiktir**
çünkü SR-IOV ile bir fiziksel cihaz birden fazla VM'e doğrudan verilir; IOMMU olmadan bir
VM'in cihazı başka bir VM'in belleğine yazabilirdi. **Donanım seviyesinde kiracı
yalıtımını mümkün kılan mekanizmadır.** *(SSS S4)*

**13.**
```bash
sudo lspci -vv -s <adres> | grep -E "LnkCap|LnkSta"
```
**`LnkCap`** cihazın desteklediğini, **`LnkSta`** şu an çalıştığını gösterir. İkisi
farklıysa (örn. Cap 16GT/s x4, Sta 8GT/s x2) yuva, nesil veya şerit paylaşımı kaynaklı
bir düşüş vardır. Sanal makinelerde bu bilgi genelde görünmez. *(4.6)*

**14.**
```bash
mpstat -P ALL 1 5
```
**Aranan:** Tek bir çekirdekte `%soft` veya `%sys` değerinin çok yüksek (>%80) olması.
Ortalama %35 masum görünürken bir çekirdek doymuş olabilir. Ardından `/proc/interrupts`
ile kesmelerin dağılımına ve `ethtool -S` ile düşen paketlere bakılır. *(4.4.3, Cevap 4.2)*

**15.** En az dört soru: (a) **"GPU kullanımı %100'e çıkıyor mu, yoksa dalgalanıyor mu?"**
— dalgalanıyorsa GPU bekliyordur, güç sorunu yoktur. (b) **"`num_workers` kaç,
`pin_memory` açık mı?"** — veri yükleyici tek thread'liyse darboğaz orada. (c) **"Veri
nerede duruyor, dosya boyutları ne?"** — milyonlarca küçük dosya = IOPS darboğazı (3.4.3).
(d) **"Profilleyici ile GPU'nun boş kaldığı boşlukları gördük mü?"** — ölçmeden karar
verilmemeli. *(Cevap 4.1)*

**16. Açarım:** Yüksek verim odaklı, gecikmeye duyarsız iş yüklerinde — veri aktarımı,
yedekleme, toplu işleme, akış işleme. **Kapatırım/agresif ayarlarım:** Düşük gecikmeli
iş yüklerinde — gerçek zamanlı API, finansal işlem, oyun sunucusu. Klasik gecikme/verim
takasıdır. *(SSS S3)*

**17. Problem:** Geleneksel sanallaştırmada ağ ve depolama I/O'su hypervisor
**yazılımından** geçiyordu ve bu sunucu CPU'sunun **%20–30'unu** tüketiyordu — satılabilir
kapasitenin üçte biri kaybediliyordu. **Çözüm:** Bu işi kendi işlemcisi, belleği ve DMA
motoru olan ayrı bir donanım kartına taşımak. Sonuç: sanallaştırma vergisi %1'in altına
indi, gecikme tutarlılaştı ve güvenlik sınırı yazılımdan **fiziksel ayrıma** taşındı.
*(4.3.3)*

**18.** Çünkü **cihaz kadar ona giden yol da bir kaynaktır ve o yol darboğaz olabilir.**
- **Bu fazdan:** Gen4 x4 destekleyen bir NVMe, Gen3 x2 yuvaya takılırsa hızının dörtte
  birini verir (4.2.2). GPU, PCIe bant genişliği yüzünden kapasitesinin %30'unda çalışır
  (4.2.4).
- **Faz 3'ten:** `io2` birimine 64.000 IOPS verirsin ama instance'ın EBS bant genişliği
  20.000'de tavan yapar — ödediğini alamazsın (3.4.5).

---

## Puanlama

| Doğru sayısı | Değerlendirme |
|---|---|
| 16–18 | Faz 5'e geç. |
| 12–15 | Faz 5'e geçebilirsin. 4.3 ve 4.4'ü bir kez daha oku. |
| 8–11 | 4.2 ve 4.4'ü tekrar et — Faz 6 bunlara dayanıyor. |
| 0–7 | Fazı baştan geç. Özellikle 4.3 (DMA) ve 4.4 (kesme) olmadan Faz 6 anlaşılmaz. |

---

# Faz 4 — Kapanış ve Faz 5'e Köprü

## Bu fazdan ne taşıyorsun

| Kavram | Neden taşınıyor |
|---|---|
| **Yol da bir kaynaktır** | Faz 5'te ağ, Faz 7'de instance limitleri |
| **Paylaşımlı hat → nokta-nokta** | Faz 5.2'de hub → switch geçişi birebir aynı hikâye |
| **DMA** | Faz 5.1'de NIC'in paketleri nasıl aldığı; Faz 6'da Nitro/SR-IOV |
| **Kesme vs yoklama, NAPI** | Faz 5.1.4'te doğrudan devam ediyor |
| **Kesme dağıtımı (MSI-X)** | Faz 5.1.4'te RSS olarak genişleyecek |
| **IOMMU** | Faz 6.6.3'te SR-IOV'un ön koşulu |
| **Ortalama metrik dağılımı gizler** | Faz 7'de gözlemlenebilirliğin temel uyarısı |

## Faz 5 bunun neresine bağlanıyor

Faz 5, bu fazın ağ kartı özelindeki devamıdır. Kesme, DMA ve tampon kavramlarını
bırakmadığın yerden alacak:

| Faz 4'te öğrendiğin | Faz 5'te karşına şöyle çıkacak |
|---|---|
| 4.3 DMA | NIC gelen paketi doğrudan RAM'e nasıl yazar |
| 4.4.2 NAPI | Ağ katmanında kesme/yoklama geçişinin tamamı |
| 4.4.3 MSI-X dağıtımı | RSS — paketleri çekirdeklere dağıtmak |
| 4.1.2 anahtarlanmış bağlantı | Hub → switch: aynı evrimin ağ versiyonu |
| 4.2.4 PCIe darboğazı | 100 Gbps NIC'in PCIe ihtiyacı |
| Kuyruk eğrisi (3.4.4'ten) | Ağ kuyruklarında gecikme ve paket kaybı |

> **Faz 5, üç haritanın kesiştiği yerdir.** Ağ *donanımını* burada işliyoruz; ağ
> *protokollerini* (IP, TCP, DNS, TLS) network haritası alıyor. Bu faz, o haritanın
> fiziksel zeminini kuruyor — **bir paketin neden geciktiğini anlamak için önce onun
> fiziksel yolculuğunu bilmek gerekir.**

---

> **Navigasyon:** [◀ Faz 3 — Depolama](Faz_3_Depolama.md) · **Faz 4** · [Faz 5 — Ağ Donanımı ▶](Faz_5_Ag_Donanimi.md)
