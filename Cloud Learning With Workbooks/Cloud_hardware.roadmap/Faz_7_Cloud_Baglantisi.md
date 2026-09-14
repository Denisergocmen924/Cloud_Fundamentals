# Faz 7 — Cloud Bağlantısı

> **Navigasyon:** [◀ Faz 6 — Sanallaştırma Donanımı](Faz_6_Sanallastirma_Donanimi.md) · **Faz 7** · [Haritanın başı ▶](README.md)

---

> ### Bu faz yeni bilgi öğretmez.
>
> Faz 0'dan 6'ya kadar biriktirdiğin her şeyi **karara** çevirir.
>
> Buraya kadar sorular "bu nasıl çalışıyor?" biçimindeydi. Bundan sonra sorular şöyle:
> **"Bu iş yükü için hangi instance? Neden? Yavaşladığında nereye bakarsın?"**
>
> Ve cevabını fiyat listesinden değil, **fizikten** vereceksin. Bu haritanın tüm amacı
> buydu.

---

## Nereden geliyoruz

Bu faz, önceki yedi fazın hepsini aynı anda kullanır:

| Faz | Buraya ne getiriyor |
|---|---|
| **0** Sayısal temel | Terimlerin dili |
| **1** CPU | Clock, IPC, çekirdek/thread, vCPU'nun ne olduğu |
| **2** Bellek | Cache hiyerarşisi, bant genişliği, NUMA, **komşu gürültüsünün kökü** |
| **3** Depolama | IOPS/throughput/latency üçlüsü, EBS'in fiziği |
| **4** Bus/IO | PCIe bütçesi, DMA, kesme, Nitro'nun donanım temeli |
| **5** Ağ | Bant genişliği ≠ gecikme, BDP, burst modelleri |
| **6** Sanallaştırma | VM exit, steal time, SR-IOV, kotalanamayan kaynaklar |

---

## Bu fazın sonunda

- Bir iş yükü tarifini duyup **gerekçeli** bir instance önerisi yapabileceksin
- EC2 aile harflerini ezberle değil, **fiziksel karşılıklarıyla** okuyabileceksin
- Komşu gürültüsünün üç mekanizmasını ayırt edip her birine müdahale edebileceksin
- "Uygulamam yavaş" cümlesini 20 dakikada bir darboğaz sınıfına indirgeyebileceksin
- Kapasite planlamasını tahminle değil, ölçümle yapabileceksin
- Aşırı ve yetersiz tahsisin **gerçek** maliyetlerini karşılaştırabileceksin

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden önemli |
|---|---|---|---|
| 7.1 | Instance ailelerini fizikle okumak | `[uygulama]` | Her gün verilen karar |
| 7.2 | AWS Nitro | `[uygulama]` | Modern EC2'nin tamamı bunun üstünde |
| 7.3 | EBS seçimi | `[uygulama]` | En sık yanlış yapılan seçim |
| 7.4 | **Komşu gürültüsü — tam analiz** | `[uygulama]` | **Cloud'un en zor teşhisi** |
| 7.5 | **Darboğaz tespiti** | `[uygulama]` | **Fazın kalbi — günlük iş becerisi** |
| 7.6 | Kapasite planlaması | `[uygulama]` | Para ve güvenilirlik burada kesişir |

---

# 7.1 EC2 instance ailelerini fiziksel temelle okumak

## 7.1.0 İsmi çözmek `[uygulama]`

```
m 7 i . 2xlarge
│ │ │      │
│ │ │      └─ boyut (vCPU sayısı)
│ │ └─────── işlemci: i=Intel, a=AMD, g=Graviton(ARM)
│ └───────── nesil (7 = 7. nesil)
└─────────── aile (m = general purpose)
```

**Ek harfler:**

| Harf | Anlamı |
|---|---|
| `d` | Yerel NVMe (instance store) var |
| `n` | Yükseltilmiş ağ bant genişliği |
| `e` | Genişletilmiş bellek |
| `z` | Yüksek frekans (high clock) |

`m7gd.4xlarge` = 7. nesil, genel amaçlı, Graviton, yerel NVMe'li, 16 vCPU.

> **Nesil numarası önemsiz değildir.** Her nesil tipik olarak %10–20 fiyat/performans
> kazandırır — daha yeni CPU, daha geniş bellek bant genişliği, daha iyi Nitro
> kartları. **Eski nesilde kalmak sessiz bir maliyettir.**

## 7.1.1 Genel amaçlı — `m` ailesi `[uygulama]`

```
vCPU : RAM  =  1 : 4      (m7i.xlarge = 4 vCPU, 16 GB)
```

**Fiziksel anlamı:** Ne CPU ne bellek tarafında özel bir optimizasyon. Dengeli L3,
orta clock, standart bellek kanalı sayısı.

| Uygun | Uygun değil |
|---|---|
| Web sunucusu, uygulama sunucusu | Ağır hesaplama (c daha ucuz) |
| Küçük-orta veritabanı | Bellek içi veri seti (r gerekir) |
| Mikroservis | Yüksek IOPS (i gerekir) |
| **Bilmediğinde başlangıç noktası** | |

> **Pratik kural:** Yeni bir iş yükünde ne istediğini bilmiyorsan `m` ile başla, **ölç**,
> sonra taşı. Tahminle `c` veya `r` seçmek, ölçümle `m`'den taşınmaktan daha pahalıya
> gelir.

## 7.1.2 Hesaplama optimize — `c` ailesi `[uygulama]`

```
vCPU : RAM  =  1 : 2      (c7i.xlarge = 4 vCPU, 8 GB)
```

**Fiziksel anlamı — bu aileyi `m`'den ayıran şey RAM'in az olması değil:**

| Özellik | Neden önemli | Hangi fazdan |
|---|---|---|
| **Yüksek sürekli clock** | Tek iş parçacığı performansı | Faz 1.3.1 |
| **vCPU başına daha çok L3** | Cache'e sığma şansı artar | Faz 2.3.4 |
| Daha yeni CPU nesli (genelde) | Daha iyi IPC | Faz 1.3.2 |

```
Uygun iş yükleri:
  • Video kodlama (transcoding)
  • Bilimsel hesaplama
  • Toplu veri işleme
  • Yüksek trafikli web (CPU-bound)
  • Oyun sunucuları (düşük gecikme, tek thread)
```

> **Cache noktası çok önemli.** `c7i` ailesinde vCPU başına düşen L3, `m7i`'den
> belirgin şekilde fazladır. Faz 2.3'ün dersi: **çalışma seti L3'e sığarsa performans
> 5 kat değişir.** Bir iş yükü `m`'den `c`'ye geçince beklenenden çok hızlanıyorsa
> sebep genellikle clock değil, cache'tir.

## 7.1.3 Bellek optimize — `r`, `x`, `u` aileleri `[uygulama]`

```
r ailesi : 1 : 8    (r7i.xlarge = 4 vCPU, 32 GB)
x ailesi : 1 : 16   (x2idn.xlarge = 4 vCPU, 64 GB)
u ailesi : 1 : 24+  (u-6tb1.metal = 448 vCPU, 6 TB)
```

**İki farklı ihtiyaca hizmet eder — karıştırma:**

| İhtiyaç | Örnek |
|---|---|
| **Kapasite** — veri RAM'e sığmalı | Redis, Memcached, in-memory veritabanı |
| **Bant genişliği** — veri RAM'den hızlı akmalı | Analitik, büyük JOIN, sütunlu tarama |

**Bellek bant genişliği** *(Faz 2.4.3)*: r ailesi daha fazla bellek kanalına sahiptir.

```
Faz 2.4.3'ün formülü:
  Bant genişliği = Kanal sayısı × Kanal başına hız

8 kanal DDR5-4800 ≈ 307 GB/s
4 kanal DDR5-4800 ≈ 154 GB/s
```

> **Kritik uyarı (Faz 2.7 — NUMA):** Büyük `r` ve `x` instance'ları birden fazla NUMA
> düğümüne yayılır. `.24xlarge` almak, iki `.12xlarge` almaktan **her zaman iyi
> değildir** — uzak düğüm erişimi 1,5–2 kat yavaştır. İş yükün NUMA farkındaysa büyük
> instance kazandırır; değilse iki küçük instance daha öngörülebilirdir.

## 7.1.4 Depolama optimize — `i`, `d` aileleri `[uygulama]`

```
i ailesi : Yüksek IOPS NVMe instance store
d ailesi : Yüksek kapasite HDD/NVMe
```

**Fiziksel anlamı — EBS ile farkı:**

| | Instance store (i/d) | EBS |
|---|---|---|
| Bağlantı | **PCIe, doğrudan** *(Faz 4.2)* | **Ağ üzerinden** *(Faz 5)* |
| Gecikme | **~50–100 μs** | ~0,5–2 ms |
| IOPS | **Milyonlar** | gp3: 16.000, io2: 256.000 |
| Kalıcılık | ❌ **Instance durunca kaybolur** | ✅ Kalıcı |
| Anlık görüntü | ❌ | ✅ |

> **Bu tablodaki "ağ üzerinden" ifadesi, EBS hakkında bilinmesi gereken en önemli
> şeydir** *(Faz 3.3.4)*. EBS bir disk değil, ağ üzerinden erişilen bir depolama
> servisidir. Gecikmesinin alt sınırı ağ gecikmesidir; IOPS limiti bir ağ kotasıdır.

```
Uygun iş yükleri (i ailesi):
  • NoSQL (Cassandra, ScyllaDB, Aerospike)
  • Elasticsearch / OpenSearch data node
  • Yüksek IOPS gerektiren önbellek katmanı
  • Veri ambarı geçici alanı (spill, shuffle)
```

**Kalıcılık sorunu nasıl çözülür:** Uygulama katmanında çoğaltma (replication). Cassandra
ve Elasticsearch zaten bunu yapar — bir düğüm kaybolduğunda veri diğer düğümlerdedir.
**Instance store'u, çoğaltması olmayan tek kopya veri için kullanma.**

## 7.1.5 Hızlandırılmış — `p`, `g`, `inf`, `trn` aileleri `[uygulama]`

| Aile | İşlemci | Kullanım |
|---|---|---|
| `p` | NVIDIA (A100, H100) | **Eğitim** (training) |
| `g` | NVIDIA (A10G, L4) | **Çıkarım** (inference), grafik |
| `inf` | AWS Inferentia | Çıkarım (maliyet optimize) |
| `trn` | AWS Trainium | Eğitim (maliyet optimize) |

**Faz 4.2.4'ün dersi burada karar verir:**

```
GPU HBM bant genişliği : ~2000–3000 GB/s
PCIe Gen4 x16          :       ~32 GB/s
Fark                   :          ~70 kat
```

> **GPU instance'larında darboğaz neredeyse hiçbir zaman GPU değildir.** Sırasıyla
> şunlardır:
> 1. **Veri besleme** — diskten/ağdan GPU'ya veri akışı
> 2. **PCIe** — CPU↔GPU transferi
> 3. **Düğümler arası ağ** — çok düğümlü eğitimde *(Faz 5.4.3, EFA)*
> 4. GPU'nun kendisi
>
> `nvidia-smi` ile GPU kullanımı %40 görüyorsan, **daha büyük GPU almak para yakmaktır.**
> Önce veri hattını (data pipeline) düzelt.

## 7.1.6 Graviton — `g` soneki `[uygulama]`

AWS'in kendi ARM tabanlı işlemcisi.

| | x86 (`i`/`a`) | Graviton (`g`) |
|---|---|---|
| ISA | x86-64 *(Faz 1.6)* | ARM64 |
| **1 vCPU** | 1 SMT thread (**çekirdeğin yarısı**) | **1 fiziksel çekirdek** |
| Fiyat/performans | Taban | **%20–40 daha iyi** |
| Uyumluluk | Her şey çalışır | **ARM64 derlemesi gerekir** |

> **vCPU satırı en önemlisidir** *(Faz 1.5.3)*. `c7g.4xlarge`'ın 16 vCPU'su 16 **fiziksel
> çekirdektir**; `c7i.4xlarge`'ın 16 vCPU'su 8 fiziksel çekirdektir. Aynı sayı, farklı
> şey.
>
> **Sonuç:** Graviton'un avantajı SMT'den az faydalanan iş yüklerinde (yoğun hesaplama)
> ilan edilenden **daha büyüktür**; SMT'den çok faydalananlarda (çok sayıda hafif thread)
> daha küçüktür.

**Geçiş kontrol listesi:**
```
□ Uygulama dili ARM64 destekliyor mu? (Go, Java, Python, Node, Rust: ✅)
□ Native bağımlılıklar ARM64 derlemesine sahip mi?
□ Docker imajları multi-arch mi?
□ Üçüncü parti ajanlar (APM, log, güvenlik) ARM64 destekliyor mu?  ← en sık takılan yer
□ CI/CD ARM64 build yapabiliyor mu?
```

> **🤔 Düşün 7.1**
> Bir ekip, Redis önbelleğini `r6g.xlarge`'dan (4 vCPU, 32 GB) `c6g.4xlarge`'a (16 vCPU,
> 32 GB) taşıdı. Gerekçe: "CPU kullanımı bazen %80'e çıkıyordu, daha çok çekirdek
> alalım." Sonuç: **performans değişmedi, maliyet 2,5 kat arttı.**
>
> a) Neden değişmedi?
> b) %80 CPU neyi gösteriyordu?
> c) Doğru hamle ne olurdu?
> *(Cevap: fazın sonunda)*

---

# 7.2 AWS Nitro'yu anlamak

Faz 6.6.4'te mekanizmasını gördün. Burada **karar sonuçlarını** alıyoruz.

## 7.2.1 Ne değişti `[uygulama]`

| | Nitro öncesi (Xen) | Nitro |
|---|---|---|
| Sanallaştırma vergisi | %20–30 | **<%1** |
| Ağ | ~10 Gbps tavan | **100 Gbps'e kadar** |
| EBS gecikmesi | Yüksek, değişken | **Düşük, öngörülebilir** |
| Bare metal | ❌ | ✅ |
| Güvenlik sınırı | Yazılım | **Donanım** |

## 7.2.2 Neden komşu gürültüsünü azaltır `[uygulama]`

```
GELENEKSEL: Komşunun ağ trafiği → hypervisor CPU'su → SENİN CPU'nu da yiyor
NITRO     : Komşunun ağ trafiği → Nitro kartı → senin CPU'na dokunmuyor
```

**Ama tamamen çözmez.** Nitro'nun çözdüğü ve çözmediği:

| Kaynak | Nitro çözdü mü |
|---|---|
| Hypervisor CPU tüketimi | ✅ Çözdü |
| Ağ işleme yarışması | ✅ Çözdü (ayrı silikon) |
| EBS I/O yarışması | ✅ Büyük ölçüde |
| **L3 cache yarışması** | ❌ **Çözmedi** |
| **Bellek bant genişliği yarışması** | ❌ **Çözmedi** |

> **Son iki satır 7.4'ün konusudur** ve cloud'da kalan tek gerçek komşu gürültüsü
> kaynağıdır.

## 7.2.3 Bare metal'in varlık sebebi `[uygulama]`

Nitro kartları hypervisor olmadan da çalıştığı için AWS, müşteriye fiziksel sunucunun
tamamını verip yine de ENA ve EBS sağlayabiliyor.

**Ne zaman gerekir:** *(Faz 6, S2)* Kendi hypervisor'ün, fiziksel çekirdeğe bağlı lisans,
donanım performans sayaçları, uyumluluk zorunluluğu, veya yatay ölçeklenemeyen tek büyük
örnek.

**Ne zaman gerekmez:** Yatay ölçeklenebilen her şey. `c7i.metal` yerine 2× `c7i.24xlarge`
neredeyse her zaman daha iyi bir takastır *(Cevap 6.C7)*.

---

# 7.3 EBS seçimi fiziksel temelle

Faz 3'ün tamamı bu bölüm için hazırlıktı.

## 7.3.1 Tipler ve fiziksel karşılıkları `[uygulama]`

| Tip | Fiziksel | IOPS | Throughput | Gecikme | Ne için |
|---|---|---|---|---|---|
| **gp3** | SSD | 3.000 (baz) – 16.000 | 125 – 1.000 MB/s | ~1 ms | **Varsayılan seçim** |
| **gp2** | SSD | 3 IOPS/GB (maks 16.000) | Boyutla bağlı | ~1 ms | **Eski — gp3'e geç** |
| **io2 / Block Express** | SSD | 256.000'e kadar | 4.000 MB/s | **<1 ms** | Kritik veritabanı |
| **st1** | **HDD** | ~500 (burst) | 500 MB/s | Yüksek | **Sıralı** büyük veri |
| **sc1** | **HDD** | ~250 | 250 MB/s | Yüksek | Arşiv, nadir erişim |

## 7.3.2 gp3 vs gp2 — neden gp3 `[uygulama]`

```
gp2: IOPS = boyut × 3        → 1000 IOPS istiyorsan 334 GB almak zorundasın
gp3: IOPS boyuttan BAĞIMSIZ  → 100 GB + 3000 IOPS alabilirsin
```

**Somut örnek:**
```
İhtiyaç: 100 GB alan, 3.000 IOPS

gp2 ile: 1.000 GB disk almak zorundasın (3 IOPS/GB)
         → 900 GB'ı boşa ödüyorsun
gp3 ile: 100 GB disk + 3.000 IOPS (baz, ek ücretsiz)
         → ~%70 daha ucuz
```

> **gp2 kullanan her volume, gp3'e geçirilmelidir.** Kesinti gerektirmez, her durumda
> ya daha ucuzdur ya eşittir. Bu, cloud maliyet optimizasyonunun en kolay kazancıdır.

## 7.3.3 HDD tiplerini (st1/sc1) doğru okumak `[uygulama]`

**st1 ve sc1 gerçekten manyetik disktir** — Faz 3.1'deki tüm fizik geçerlidir:

```
Sıralı erişim : ✅ Hızlı (500 MB/s'e kadar)
Rastgele erişim: ❌ FELAKET (kafa hareketi, ~500 IOPS tavan)
```

| ✅ Uygun | ❌ Kesinlikle uygun değil |
|---|---|
| Log toplama, büyük dosya depolama | **Boot volume** |
| Veri ambarı sıralı tarama | **Veritabanı** |
| Büyük ETL girdi/çıktısı | Rastgele erişimli her şey |
| Yedek hedefi | |

> **En sık yapılan hata:** "sc1 çok ucuz, boot volume yapalım." Sonuç: sunucu açılışı
> dakikalar sürer, her paket kurulumu donar. **Boot volume rastgele erişimdir**
> *(Faz 3.4.3)*.

## 7.3.4 Karar ağacı `[uygulama]`

```
BAŞLA
  │
  ├─ Erişim deseni SIRALI mı? (log, yedek, büyük dosya)
  │    └─ EVET → st1 (sık) / sc1 (nadir)
  │
  ├─ IOPS > 16.000 gerekiyor mu?
  │    └─ EVET → io2 Block Express
  │
  ├─ Gecikme < 1 ms şart mı? (kritik veritabanı)
  │    └─ EVET → io2
  │
  ├─ Kalıcılık gerekmiyor + çok yüksek IOPS?
  │    └─ EVET → instance store (i ailesi)
  │
  └─ Diğer her durum → gp3  ← vakaların ~%80'i
```

## 7.3.5 Gözden kaçan tavan: instance EBS bant genişliği `[uygulama]`

**Volume'un limiti ile instance'ın limiti ayrıdır.** *(Faz 3.4.5)*

```
io2 volume  : 64.000 IOPS sağlayabilir
m7i.large   : ~10.000 IOPS'a izin verir  ← INSTANCE tavanı

→ Pahalı volume, ucuz instance = para boşa gider
```

```bash
# Instance'ın EBS limitlerini gör
aws ec2 describe-instance-types --instance-types m7i.2xlarge \
  --query 'InstanceTypes[0].EbsInfo'
```

> **Kural:** Disk performansı alırken **her iki tavanı da** kontrol et. Faz 4.2.4'ün
> dersiyle aynı: **en pahalı bileşen, onu besleyen yol kadar hızlıdır.**

> **🤔 Düşün 7.2**
> Bir PostgreSQL sunucusu `m7i.xlarge` üzerinde, 500 GB gp3 volume (3.000 IOPS baz).
> `iostat -x` gösteriyor: `r/s + w/s ≈ 2980`, `r_await ≈ 24 ms`, `aqu-sz ≈ 71`.
>
> a) Darboğaz nerede?
> b) Üç farklı çözüm öner ve maliyet/etki sırala.
> c) Hangi çözüm **hiç para harcamadan** işe yarayabilir?
> *(Cevap: fazın sonunda)*

---

# 7.4 Komşu gürültüsü — tam analiz

**Cloud'daki en zor teşhis budur.** Çünkü standart metriklerin hiçbirinde görünmez.

## 7.4.1 Üç ayrı mekanizma `[uygulama]`

Komşu gürültüsü tek bir şey değildir. **Üç farklı mekanizma, üç farklı belirti, üç farklı
çözüm:**

| # | Mekanizma | Belirti | Ölçülebilir mi | Nitro çözdü mü |
|---|---|---|---|---|
| **1** | **CPU zamanı yarışması** | **steal time** | ✅ `top`, `vmstat` | ⚠️ Kısmen |
| **2** | **L3 cache kirlenmesi** | Yanıt süresi ↑, metrikler normal | ❌ **Görünmez** | ❌ Hayır |
| **3** | **Bellek bant genişliği doyması** | Yanıt süresi ↑, metrikler normal | ❌ **Görünmez** | ❌ Hayır |

**Ek olarak (Nitro öncesi veya özel durumlar):**
| 4 | PCIe/ağ yarışması | I/O throughput ↓ | ⚠️ Kısmen | ✅ Çözdü |

## 7.4.2 Mekanizma 1 — CPU zamanı `[uygulama]`

*(Faz 6.5.2'de tam işlendi)*

```bash
top      # st alanı
vmstat 1 # st kolonu
```

**Tek ölçülebilir olan budur** ve bu yüzden en çok konuşulanıdır — ama çoğu zaman **en
az zararlı** olanıdır.

## 7.4.3 Mekanizma 2 — L3 cache kirlenmesi `[uygulama]`

*(Faz 2.3.4 mekanizma, Cevap 6.1 senaryo)*

```
Komşu, L3'e sığmayan büyük bir veri setini tarıyor
   ↓
Her erişimi bir cache satırı tahliye ediyor
   ↓
SENİN sıcak verin L3'ten atılıyor
   ↓
Senin erişimlerin L3 (40 çevrim) yerine RAM'e (200 çevrim) gidiyor
   ↓
Aynı kod, 5 kat yavaş bellek erişimi
```

**Görünen tablo:**
| Metrik | Ne gösterir |
|---|---|
| CPU kullanımı | **Aynı veya yüksek** (aynı iş, daha çok çevrim) |
| Steal time | **%0 — hiç değişmez** |
| Bellek kullanımı | Değişmez |
| Disk/ağ | Değişmez |
| **Uygulama p99** | **Artar** |

> **"Hiçbir metrik değişmedi ama uygulama yavaş" cümlesinin en sık sebebi budur.**

## 7.4.4 Mekanizma 3 — bellek bant genişliği `[uygulama]`

*(Faz 2.4.3)*

Bellek kanalları da paylaşımlıdır. Komşu bant genişliğini doyurursa, senin RAM
erişimlerin **kuyruğa girer.**

```
Boş sistemde RAM erişimi    : ~200 çevrim
Bant genişliği doymuşken    : ~400–600 çevrim
```

Belirtisi mekanizma 2 ile **neredeyse aynıdır** — ve ikisini ayırt etmek pratikte
gereksizdir, çünkü **çözümleri aynıdır.**

## 7.4.5 Teşhis yordamı `[uygulama]`

Doğrudan ölçemezsin. **Eleme ile ilerlersin:**

```
1. Steal time kontrol et
      > %5  → Mekanizma 1. Yeniden başlat / taşı / dedicated.
      ≈ 0   → devam et
2. Standart metrikler normal mi? (CPU, RAM, disk, ağ)
      Hayır → komşu gürültüsü değil, normal darboğaz (7.5'e git)
      Evet  → devam et
3. Kod veya yük değişti mi? (deploy, trafik artışı)
      Evet  → önce onu ele
      Hayır → devam et
4. Aynı AMI'den 2-3 yeni instance başlat, aynı yükü ver
      Yeniler hızlı  → ✅ KOMŞU GÜRÜLTÜSÜ DOĞRULANDI
      Hepsi yavaş    → sistemik bir problem var, komşu değil
5. perf erişimin varsa doğrula:
      perf stat -e cache-misses,instructions,cycles ./uygulama
      cache-miss ↑ ve IPC ↓ → mekanizma 2/3 kesinleşti
```

> **4. adım en güçlü araçtır** ve para gerektirmez. **Karşılaştırma, cloud'da tek ölçüm
> yönteminin olmadığı yerde ölçüm yerine geçer.**

## 7.4.6 Çözümler ve maliyetleri `[uygulama]`

| Çözüm | Nasıl çalışır | Maliyet | Ne zaman |
|---|---|---|---|
| **Yeniden başlat** | Başka fiziksel hosta düşme ihtimali | **Ücretsiz** | **İlk deneme** |
| **Daha büyük instance** | Sunucunun daha büyük dilimi = az komşu | Orta | Tekrarlıyorsa |
| **`.metal`** | Komşu yok | **Yüksek (4×)** | Yatay ölçeklenemiyorsa |
| **Dedicated Instance** | Donanım sana ayrılmış | Yüksek | Uyumluluk gerekiyorsa |
| **Dedicated Host** | Belirli fiziksel sunucu + görünürlük | En yüksek | BYOL lisans |
| **Cluster placement group** | Yerleşim kontrolü | Düşük | Düğümler arası gecikme kritikse |
| **Kodu cache dostu yap** | Çalışma setini küçült *(Faz 2.3.6)* | **Ücretsiz** | **Her zaman değer** |

> **Sıralama önerisi: önce ücretsiz olanları dene.** Yeniden başlatmak vakaların önemli
> bir kısmını çözer; cache dostu kod her komşudan bağımsız olarak kazandırır.
>
> **`.metal`'e atlamak, teşhis yerine para harcamaktır.**

---

# 7.5 Darboğaz tespiti — bütünleşik

**Bu bölüm fazın kalbidir ve bu haritanın pratik özüdür.**

## 7.5.1 Dört darboğaz sınıfı `[uygulama]`

Her performans problemi dört sınıftan birine düşer:

| Sınıf | Ne sınırlıyor | Ana faz |
|---|---|---|
| **CPU bound** | İşlemci kapasitesi | Faz 1, 2 |
| **Memory bound** | RAM kapasitesi veya bant genişliği | Faz 2 |
| **I/O bound** | Disk | Faz 3 |
| **Network bound** | Ağ | Faz 5 |

**Beşinci bir kategori vardır ve en sık unutulandır:**

| **Bekleme bound** | Hiçbir kaynak dolu değil — **başka bir şey bekleniyor** | Kilit, uzak çağrı, havuz tükenmesi |

## 7.5.2 20 dakikalık teşhis yordamı `[uygulama]`

Bir ekip "uygulamam yavaş" dediğinde izlenecek sıra:

```
ADIM 0 — DOĞRU SORUYU SOR (2 dk)
   "Ne zamandan beri?"      → değişiklik var mı (deploy, trafik, veri büyümesi)
   "Her istek mi, bazıları mı?" → p50 mi p99 mu bozuldu
   "Ne kadar yavaş?"        → 2 kat mı 100 kat mı (farklı sebepler)
   "Hep mi, ara sıra mı?"   → sürekli mi periyodik mi

ADIM 1 — CPU (3 dk)
   top / mpstat -P ALL 1
   ├─ us yüksek (>%80)        → CPU BOUND → 7.5.3
   ├─ sy yüksek               → çekirdek: sistem çağrısı, kesme, bağlam değiştirme
   ├─ wa yüksek               → I/O BOUND → 7.5.5
   ├─ st yüksek               → KOMŞU / KREDİ → 7.4
   └─ hepsi düşük, yavaş      → BEKLEME BOUND → 7.5.7

ADIM 2 — BELLEK (3 dk)
   free -h / vmstat 1
   ├─ available düşük          → MEMORY BOUND (kapasite)
   ├─ si/so sıfırdan büyük     → SWAP → felaket, hemen ele al
   └─ dolu ama swap yok        → bant genişliği olabilir → 7.5.4

ADIM 3 — DİSK (3 dk)
   iostat -x 1
   ├─ await yüksek + aqu-sz yüksek → I/O BOUND → 7.5.5
   ├─ r/s+w/s ≈ volume limiti      → IOPS TAVANI
   └─ rkB/s+wkB/s ≈ throughput lim.→ THROUGHPUT TAVANI

ADIM 4 — AĞ (3 dk)
   ss -s / ethtool -S / sar -n DEV 1
   ├─ rx_dropped artıyor       → NIC/CPU → Faz 5.1.3
   ├─ retransmit yüksek        → paket kaybı
   └─ bant genişliği tavanda   → NETWORK BOUND → 7.5.6

ADIM 5 — HİÇBİRİ DEĞİLSE (6 dk)
   → BEKLEME BOUND → 7.5.7
```

> **Bu sıralama rastgele değil:** en ucuz ve en kesin ölçümden en pahalıya doğrudur.
> CPU/bellek ölçümü saniyeler sürer ve nettir; "bekleme bound" teşhisi uygulama
> profili gerektirir.

## 7.5.3 CPU bound `[uygulama]`

**İmza:** `us` %80+, run queue > çekirdek sayısı, steal ≈ 0

```bash
top -H              # hangi thread
perf top            # hangi fonksiyon (erişim varsa)
uptime              # load average
```

| Çözüm | Ne zaman |
|---|---|
| Kodu optimize et | **Her zaman önce bunu değerlendir** |
| Daha çok vCPU (dikey) | Paralelleşebiliyorsa |
| Daha yüksek clock (`c`, `z`) | **Tek thread sınırlıysa** — vCPU eklemek işe yaramaz |
| Yatay ölçekleme | Durumsuz (stateless) ise |
| Graviton'a geç | %20–40 fiyat/performans |

> **En sık hata:** Tek thread'li bir darboğazda vCPU eklemek. `top -H` ile **tek bir
> thread %100'de ve diğerleri boştaysa**, 64 vCPU da alsan hiçbir şey değişmez.
> *(Faz 1.5.1'in dersi)*

## 7.5.4 Memory bound `[uygulama]`

**İki farklı problem — ayırt et:**

| | Kapasite | Bant genişliği |
|---|---|---|
| İmza | `available` düşük, swap, OOM kill | RAM boş ama uygulama yavaş |
| Ölçüm | `free -h`, `dmesg` | Zor — `perf stat`, IPC düşüşü |
| Çözüm | Daha çok RAM (`r` ailesi) | Daha çok **kanal** (`r` ailesi), NUMA, cache dostu kod |

```bash
free -h
vmstat 1                        # si/so — swap aktivitesi
dmesg | grep -i "out of memory" # OOM kill geçmişi
numactl --hardware              # NUMA düğümleri (Faz 2.7)
```

> **Swap kullanımı görüyorsan bu bir uyarı değil, alarmdır** *(Faz 2.6)*. Cloud'da
> genellikle swap kapalıdır ve bu kasıtlıdır: **hızlı başarısız ol, yavaşça ölme.**
> Kubernetes varsayılan olarak swap'lı düğümde çalışmayı reddeder.

## 7.5.5 I/O bound `[uygulama]`

**İmza:** `wa` yüksek, `await` yüksek, `aqu-sz` yüksek

```bash
iostat -x 1
iotop                 # hangi process
```

**Faz 3.4.5'in okuma kılavuzu:**
```
r/s + w/s  ≈ volume IOPS limiti?     → IOPS tavanı
rkB/s+wkB/s ≈ throughput limiti?     → throughput tavanı
await yüksek, ikisi de limitte değil → gecikme problemi (io2 gerekebilir)
%util                                 → NVMe'de YANILTICI, kullanma
```

| Çözüm | Ne zaman |
|---|---|
| **Sorguları/erişimi düzelt** | **Her zaman önce** — indeks, N+1, tam tarama |
| IOPS artır (gp3 ayarı) | IOPS tavanındaysan |
| io2'ye geç | Gecikme kritikse |
| Instance store | Kalıcılık gerekmiyorsa, çok yüksek IOPS |
| **Instance EBS tavanını kontrol et** | **Volume'u büyütmeden önce** *(7.3.5)* |
| Daha çok RAM (önbellek) | Okuma ağırlıklıysa — **disk erişimini tamamen kaldırır** |

> **Son satır sık atlanır:** Okuma ağırlıklı bir veritabanında RAM eklemek, disk almaktan
> daha ucuz ve daha etkili olabilir. Faz 2.1'in hiyerarşi dersi: **en hızlı I/O,
> yapılmayan I/O'dur.**

## 7.5.6 Network bound `[uygulama]`

**İmza:** Bant genişliği tavanda, `rx_dropped`/retransmit artıyor, gecikme yüksek

```bash
sar -n DEV 1          # arayüz throughput
ss -s                 # soket özeti
ss -i                 # cwnd, rtt (BDP analizi)
ethtool -S eth0       # drop sayaçları
```

**Faz 5'in ayrımı kritik:**

| Problem | İmza | Çözüm |
|---|---|---|
| **Bant genişliği** | Hat tavanda | Daha büyük instance, jumbo frame |
| **Gecikme (mesafe)** | `ping` yüksek | Bölge/AZ yerleşimi, CDN — **bant genişliği işe yaramaz** |
| **Gecikme (kuyruk)** | `ping` dalgalı, hat %90+ | Yükü azalt, ölçekle |
| **TCP penceresi (BDP)** | Hat boş ama transfer yavaş | Tampon, paralel akış |
| **pps limiti** | Küçük paket çok, bant genişliği düşük | RSS, daha büyük instance |
| **Burst kredisi** | İlk dakikalar hızlı | Garanti ağ performanslı boyut |

## 7.5.7 Bekleme bound — en çok atlanan `[uygulama]`

**İmza: hiçbir kaynak dolu değil, ama uygulama yavaş.**

Bu, modern uygulamalarda **en sık görülen** durumdur ve yukarıdaki dört sınıfa girmez.

| Sebep | Nasıl teşhis edilir |
|---|---|
| **Uzak çağrı bekleme** | Çağrı sayısı × RTT ≈ toplam süre mi *(Faz 5, Cevap 5.2)* |
| **Kilit/mutex yarışması** | Thread dump, `perf lock` |
| **Bağlantı havuzu tükenmiş** | Havuz metrikleri — bekleyen istek sayısı |
| **Thread havuzu tükenmiş** | Aktif thread = maks thread |
| **Garbage collection duraklaması** | GC logları, duraklama süreleri |
| **Aşağı akış servisi yavaş** | Dağıtık izleme (tracing) |

```bash
# Çalışan bir process'in ne beklediği
cat /proc/<PID>/status | grep State
cat /proc/<PID>/wchan
strace -c -p <PID>        # hangi sistem çağrısında zaman geçiyor
```

> **Altın kural:** **Hiçbir kaynak dolu değilse donanım almak hiçbir şeyi
> düzeltmez.** Bu durumda cevap uygulama katmanındadır — ve bu haritanın verdiği en
> değerli beceri, **donanımı doğru şekilde elemek** olabilir.

> **🤔 Düşün 7.3**
> Bir API'nin p99 yanıt süresi 200 ms'den 2 saniyeye çıktı. Elindeki veriler:
> CPU %25, bellek %40, `iostat` await 1 ms, ağ %8 kullanım, steal time %0.
> Kod 3 gün önce deploy edildi ama sorun bugün başladı. Trafik normal.
>
> a) Hangi darboğaz sınıfı? Neden diğerleri elendi?
> b) Deploy 3 gün önceyse neden bugün başlamış olabilir — üç hipotez üret.
> c) Sırayla ne kontrol edersin?
> *(Cevap: fazın sonunda)*

![Şekil 7.1 — Dört darboğaz sınıfı ve teşhis yordamı akış şeması](diagrams/hw-7-01-bottleneck-decision-tree.png)
*Şekil 7.1 — "Uygulamam yavaş" cümlesinden darboğaz sınıfına giden eleme yolu.*

---

# 7.6 Kapasite planlaması

## 7.6.1 Instance seçme süreci `[uygulama]`

```
1. İŞ YÜKÜNÜ KARAKTERİZE ET
   • CPU mu bellek mi I/O mu ağ mı ağırlıklı?
   • Sürekli mi aralıklı mı?          → t ailesi mi değil mi (Faz 6.5.3)
   • Tek thread mi paralel mi?         → clock mu çekirdek mi (Faz 1.5.1)
   • Çalışma seti ne kadar?            → cache ve RAM (Faz 2.3)
   • Durumlu mu durumsuz mu?           → yatay ölçeklenebilir mi

2. TAHMİN ETME — ÖLÇ
   • m ailesinde başla
   • Gerçekçi yükle çalıştır (EN AZ 30 DAKİKA — kredi/burst tuzağı)
   • Darboğazı bul (7.5)

3. DARBOĞAZA GÖRE TAŞI
   CPU bound + tek thread  → c (yüksek clock)
   CPU bound + paralel     → daha çok vCPU / Graviton
   Memory bound            → r
   I/O bound               → i ailesi veya EBS ayarı
   Network bound           → daha büyük boyut / garanti ağ

4. DOĞRULA VE TEKRARLA
   • Yeni darboğaz nerede? (darboğaz her zaman yer değiştirir)
   • Maliyet/performans kabul edilebilir mi?
```

> **2. adımdaki "en az 30 dakika" bir detay değildir.** `t` ailesinin kredisi ve ağın
> burst kotası 5 dakikalık testte tükenmez; üretimde 20. dakikada tükenir. *(Faz 5.2.3,
> 6.5.3)*

## 7.6.2 Right-sizing `[uygulama]`

**Doğru boyutlandırma = gerçek kullanıma göre ayarlama.**

```
Tipik bulgu: instance'ların %60-70'i aşırı tahsisli
```

```bash
# CloudWatch üzerinden temel metrikler
# CPUUtilization, MemoryUtilization (CW agent gerekir), NetworkIn/Out
# EBS: VolumeReadOps, VolumeWriteOps, VolumeQueueLength
# t ailesi: CPUCreditBalance  ← EN KRİTİK METRİK
```

**Hedef aralıklar:**

| Metrik | Hedef | Neden |
|---|---|---|
| CPU ortalama | **%40–60** | Burst payı bırak |
| CPU p99 | < %80 | Tepe anlarında yığılma olmasın |
| Bellek | < %80 | OOM ve swap payı |
| Disk kuyruk derinliği | < 2 | Kuyruk eğrisi *(Faz 3.4.4)* |
| Ağ | < %70 | Kuyruk gecikmesi *(Faz 5.3.2)* |

> **Neden %100 hedeflenmez?** Faz 3.4.4'teki kuyruk eğrisi. **%95 doluluk, %50
> doluluğun 8 katı gecikme demektir.** Kaynağın son %20'si ucuz değildir — p99'u
> patlatarak ödenir.
>
> **Bu, bu haritanın belki de en pratik tek dersidir:** kapasite planlaması, ortalamayı
> değil **kuyruk davranışını** yönetmektir.

## 7.6.3 Aşırı vs yetersiz tahsis `[uygulama]`

| | Aşırı tahsis (over-provisioning) | Yetersiz tahsis (under-provisioning) |
|---|---|---|
| Doğrudan maliyet | **Boşa giden para** | Düşük fatura |
| Performans | ✅ Güvenli | ❌ Yavaş, dalgalı |
| Arıza riski | Düşük | **Yüksek** |
| Görünürlük | **Faturada görünür** | **Görünmez — müşteri kaybında** |
| Düzeltme | Kolay, düşür | Kolay ama **geç kalınmış olur** |

```
Aşırı tahsis maliyeti  : ÖLÇÜLEBİLİR   (fatura)
Yetersiz tahsis maliyeti: GİZLİ        (kaybedilen müşteri, itibar, olay yönetimi)
```

> **Bu asimetri yüzünden makul miktarda aşırı tahsis rasyoneldir.** Ama "makul" kelimesi
> ölçümle tanımlanmalıdır — %60 kullanımda çalışmak makuldür, %8 kullanımda çalışmak
> ölçmemektir.
>
> **Doğru yaklaşım: otomatik ölçekleme + makul taban.** Tabanı p50'ye göre, ölçeklemeyi
> p99'a göre ayarla.

## 7.6.4 Maliyet modelleri `[uygulama]`

| Model | İndirim | Taahhüt | Risk |
|---|---|---|---|
| On-Demand | — | Yok | Yok |
| Savings Plans / Reserved | %30–70 | 1–3 yıl | Kullanmazsan ödersin |
| **Spot** | **%70–90** | Yok | **2 dakika uyarıyla kesilir** |

**Spot'un teknik gereksinimi:** İş yükü **kesintiye dayanıklı** olmalı — durumsuz,
kontrol noktalı (checkpoint) veya kuyruk tabanlı.

```
✅ Spot uygun : toplu işleme, CI/CD runner, ML eğitimi (checkpoint'li),
                durumsuz web katmanı (karışık filo ile)
❌ Spot uygun değil: veritabanı, durumlu tek örnek, kesinti kabul edilmeyen üretim
```

> **Pratik desen:** Taban kapasiteyi Savings Plan ile, tepe kapasiteyi Spot ile karşıla.
> Bu, çoğu iş yükünde %50+ tasarruf sağlar ve dayanıklılığı bozmaz.

> **🤔 Düşün 7.4**
> Bir ekip, 20 adet `m5.2xlarge` üzerinde çalışan durumsuz bir API'ye sahip. Ortalama
> CPU kullanımı %12, p99 CPU %35. Hepsi On-Demand. Aylık maliyet yüksek diye şikâyet
> ediyorlar.
>
> a) Buradaki üç ayrı optimizasyon fırsatını bul.
> b) Her birinin tahmini kazancını ve riskini belirt.
> c) Hangi sırayla uygularsın ve neden?
> *(Cevap: fazın sonunda)*

---

# 7.7 Bu faz bozulunca — karar arızaları

Bu faz bir mekanizma değil, bir **karar süreci** öğrettiği için "bozulması" da karar
hatası biçiminde olur:

| Karar hatası | Belirtisi | Nereden kaçınılır |
|---|---|---|
| Üretim yükünü `t` ailesine koymak | İlk dakikalar hızlı, sonra çöküş | 7.1, Faz 6.5.3 |
| Ölçmeden instance tipi seçmek | Yanlış ailede aşırı ödeme | 7.6.1 |
| gp2'de kalmak | Gereksiz büyük disk, gereksiz maliyet | 7.3.2 |
| Boot volume'u st1/sc1 yapmak | Açılış dakikalar sürer | 7.3.3 |
| Volume IOPS'ini artırıp instance tavanını unutmak | Para harcandı, hız değişmedi | 7.3.5 |
| GPU kullanımı %40'ken daha büyük GPU almak | Maliyet arttı, hız değişmedi | 7.1.5 |
| Tek thread darboğazında vCPU eklemek | Hiçbir şey değişmedi | 7.5.3 |
| "Yavaş" deyip donanım almak (bekleme bound) | Para harcandı, sorun duruyor | 7.5.7 |
| 5 dakikalık yük testine güvenmek | Üretimde 20. dakikada çöküş | 7.6.1 |
| Komşu gürültüsünde doğrudan `.metal` | 4× maliyet, %12 kazanç | 7.4.6 |
| Instance store'u tek kopya veriye kullanmak | **Veri kaybı** | 7.1.4 |
| Kaynağı %95'te çalıştırmak | p99 patlar | 7.6.2 |

---

# Faz 7 — Düşün Sorularının Cevapları

## Cevap 7.1

**a) Neden değişmedi?**

**Redis tek thread'lidir.**

```
r6g.xlarge  :  4 vCPU — Redis 1 tanesini kullanıyor, 3'ü boşta
c6g.4xlarge : 16 vCPU — Redis yine 1 tanesini kullanıyor, 15'i boşta
```

Ek 12 vCPU'nun Redis'e hiçbir faydası yoktur. *(Faz 1.5.1'in dersi: paralelleşmeyen iş
yükünde çekirdek eklemek işe yaramaz.)*

**b) %80 CPU neyi gösteriyordu?**

**Muhtemelen bir çekirdeğin %80'ini — ama metrik onu ortalama olarak gösteriyordu.**

```
4 vCPU'lu makinede tek thread %100 çalışıyorsa:
  CloudWatch CPUUtilization = %25 gösterir

%80 görünüyorsa ya:
  • Redis'in kendi thread'i + arka plan işleri (AOF, RDB) yükleniyordu, veya
  • Ölçüm tepe anlarına aitti
```

**Doğru ölçüm:**
```bash
top -H                    # thread bazında — hangi thread doygun?
redis-cli --latency       # Redis'in kendi gecikme ölçümü
redis-cli INFO commandstats
```

Eğer tek bir thread %100'deyse **CPU gerçekten darboğazdır — ama vCPU ekleyerek
çözülmez.**

**c) Doğru hamle ne olurdu?**

| Öncelik | Hamle | Neden |
|---|---|---|
| **1** | **Ölç: gerçekten CPU mu?** | Redis genelde **bellek veya ağ** sınırlıdır, CPU değil |
| **2** | Yüksek clock'lu instance (`c7g`, `z` serisi) | Tek thread → **clock önemli, çekirdek değil** *(7.5.3)* |
| **3** | Redis Cluster / sharding | **Tek thread sınırını aşmanın tek gerçek yolu** |
| **4** | Yavaş komutları bul | `KEYS`, büyük `HGETALL`, O(n) komutlar |
| **5** | Ağ tarafına bak | Küçük istek çok → pps limiti *(Faz 5)* |

> **Asıl ders:** Ekip `r`'den `c`'ye geçerken **RAM'i aynı tutup vCPU'yu 4 katladı.**
> Redis için doğru eksen buydu diye düşündüler — ama Redis'in ölçekleme ekseni vCPU
> değil, **sharding**'dir.
>
> **Bu haritanın temel mesajı burada:** Instance seçimi, iş yükünün **hangi kaynağa**
> bağlı olduğunu bilmeyi gerektirir. Bunu bilmeden yapılan her seçim kumar oynamaktır.

---

## Cevap 7.2

**a) Darboğaz nerede?**

**gp3 volume'un IOPS limiti.** *(Faz 3.4.5)*

```
r/s + w/s ≈ 2980 ≈ 3000  ← gp3 baz IOPS limiti
aqu-sz = 71               ← çok derin kuyruk
r_await = 24 ms           ← SSD için felaket (normal: ~1 ms)

Doğrulama (Little Yasası):
  await ≈ kuyruk ÷ IOPS = 71 ÷ 2980 = 23,8 ms  ✓ tutarlı
```

**Teşhis kesin:** Disk 3.000 IOPS'i sunuyor, uygulama daha fazlasını istiyor, fark
kuyrukta birikiyor. Bu bir **kapasite tavanıdır**, gecikme problemi değil.

**b) Üç çözüm — maliyet/etki sıralı**

| # | Çözüm | Maliyet | Etki | Not |
|---|---|---|---|---|
| **1** | **gp3 IOPS'ini artır** (3.000 → 12.000) | **Düşük** — IOPS ayrı fiyatlanır | ✅ Anında | **Kesintisiz yapılabilir** |
| **2** | **Sorguları düzelt** (indeks, N+1) | **Ücretsiz** | ✅ Kalıcı, en büyük | Zaman ister |
| **3** | **RAM ekle** (`r7i.xlarge` → shared_buffers) | Orta | ✅ Okumaları diskten kaldırır | Okuma ağırlıklıysa çok etkili |
| 4 | io2'ye geç | Yüksek | ✅ Gecikme de düşer | Ancak IOPS > 16.000 gerekiyorsa |
| 5 | Instance store'a taşı | Yüksek + risk | ✅ En hızlı | **Kalıcılık yok** — PostgreSQL için tehlikeli |

**Ama önce 7.3.5'i kontrol et:**
```
m7i.xlarge'ın EBS IOPS tavanı nedir?
Volume'u 12.000'e çıkarırsan instance izin veriyor mu?
→ aws ec2 describe-instance-types ... EbsInfo
```
Instance tavanı 10.000 ise, volume'u 16.000 yapmak **para israfıdır.**

**c) Hiç para harcamadan işe yarayabilecek çözüm**

**Sorguları düzeltmek (2 numara) — ve genellikle en büyük kazancı verir.**

```bash
# PostgreSQL'de somut adımlar
# 1. Hangi sorgular disk okuyor?
SELECT query, calls, shared_blks_read, shared_blks_hit
FROM pg_stat_statements ORDER BY shared_blks_read DESC LIMIT 10;

# 2. Eksik indeks var mı? (sıralı tarama sayısı yüksek tablolar)
SELECT relname, seq_scan, idx_scan, seq_tup_read
FROM pg_stat_user_tables WHERE seq_scan > idx_scan ORDER BY seq_tup_read DESC;

# 3. Cache hit oranı
SELECT sum(blks_hit)*100/sum(blks_hit+blks_read) AS hit_ratio FROM pg_stat_database;
# %99'un altındaysa shared_buffers yetersiz
```

**Tek bir eksik indeks, 3.000 IOPS'lik bir yükü 50 IOPS'e düşürebilir.**

> **Bu, bu haritanın en çok tekrarlanan dersidir ve altıncı kez karşına çıkıyor:**
>
> | Faz | Aynı ders |
> |---|---|
> | 2.3.6 | Satır/sütun sıralı erişim — **kod değişti, 10× hızlandı** |
> | 3.4 | Erişim desenini düzelt, disk alma |
> | 4.2.4 | Darboğaz GPU'da değil, besleyen yolda |
> | 5.3.3 | Bant genişliği değil, TCP penceresi |
> | 6.5.3 | Instance tipi yanlış, daha büyüğü değil |
> | **7.3** | **Sorgu deseni yanlış, disk yavaş değil** |
>
> **Donanım almadan önce erişim desenini düzelt.**

---

## Cevap 7.3

**a) Hangi darboğaz sınıfı?**

**Bekleme bound (7.5.7).**

Eleme:
| Sınıf | Neden elendi |
|---|---|
| CPU bound | %25 — bol kapasite var |
| Memory bound | %40 — bol kapasite, swap yok |
| I/O bound | `await` 1 ms — normal, disk rahat |
| Network bound | %8 kullanım — hat neredeyse boş |
| Komşu gürültüsü | steal %0 **ve** p99 bozulurken p50 normal olabilir — ama hiçbir kaynak dolu değil |

**Hiçbir kaynak doygun değil ama uygulama 10 kat yavaş.** Bu tanım gereği bekleme
bound'dur: sistem bir şeyi **bekliyor**, hiçbir şeyi **tüketmiyor.**

> **Ek ipucu: p99 bozuldu.** p50 de bozulsaydı sistemik bir yavaşlama olurdu. Sadece p99
> bozulması, **isteklerin bir kısmının bir kuyrukta beklediğini** gösterir — havuz
> tükenmesi, kilit yarışması veya yavaş bir aşağı akış bağımlılığının imzası.

**b) Deploy 3 gün önceyse neden bugün başladı — üç hipotez**

| # | Hipotez | Mekanizma |
|---|---|---|
| **1** | **Kaynak sızıntısı** | Bağlantı/thread/dosya tanıtıcı sızıntısı, havuzun dolması 3 gün sürdü |
| **2** | **Veri büyümesi eşiği** | Bir tablo indekssiz büyüdü; 3 gün sonra sorgu planı değişti veya cache'e sığmaz oldu |
| **3** | **Aşağı akış bağımlılığı değişti** | Başka bir servis/veritabanı bugün yavaşladı; senin uygulaman sadece bekliyor |

**Ek hipotezler:**
- Zamanlanmış bir iş (haftalık batch, yedek) bugün çalıştı
- Sertifika/DNS/TLS yeniden anlaşması
- Bağımlılığın rate limit'ine bugün ulaşıldı
- GC davranışı: heap 3 günde doldu, artık sürekli full GC

> **1 ve 2 numara "3 gün" ipucunu doğrudan açıklar** ve bu yüzden en güçlü
> hipotezlerdir. **Zamanın kendisi bir kanıttır:** yavaş biriken bir şey var.

**c) Sırayla ne kontrol edersin**

```
1. HAVUZ METRİKLERİ (en olası, en hızlı)
   • Bağlantı havuzu: aktif / bekleyen / maks
   • Thread havuzu: aktif / kuyruk uzunluğu
   • Veritabanı tarafı: SELECT count(*) FROM pg_stat_activity;
   → Havuz dolmuşsa hipotez 1 doğrulandı

2. DAĞITIK İZLEME (tracing)
   • Bir yavaş isteğin span'lerine bak
   • Süre NEREDE geçiyor? → hipotez 3'ü doğrular veya eler
   → Bu adım genelde cevabı tek başına verir

3. KAYNAK SAYAÇLARI (sızıntı arıyoruz)
   ls /proc/<PID>/fd | wc -l        # dosya tanıtıcı sayısı
   cat /proc/<PID>/status | grep Threads
   ss -s                            # soket sayısı, TIME_WAIT
   → 3 gün boyunca artan bir grafik varsa hipotez 1 kesinleşti

4. GC / ÇALIŞMA ZAMANI
   • GC duraklama süreleri ve sıklığı
   • Heap kullanım eğilimi (3 günlük grafik)

5. VERİTABANI PLAN DEĞİŞİKLİĞİ
   • Yavaş sorgu logu — 3 gün önce yoktu, bugün var mı?
   • EXPLAIN ANALYZE — indeks kullanılıyor mu, tam tarama mı?
   • pg_stat_user_tables: seq_scan aniden arttı mı?
```

> **En önemli adım 2'dir.** Dağıtık izleme varsa, "süre nerede geçiyor" sorusunu
> doğrudan cevaplar ve diğer adımları gereksiz kılabilir.
>
> **İzleme yoksa, bu olayın asıl dersi izleme kurmaktır.** Bekleme bound problemler
> altyapı metrikleriyle teşhis edilemez — çünkü altyapı metrikleri **her şey yolunda**
> der.

---

## Cevap 7.4

**a) Üç optimizasyon fırsatı**

```
Mevcut durum: 20 × m5.2xlarge (8 vCPU) = 160 vCPU
              Ortalama CPU %12, p99 CPU %35
              Hepsi On-Demand
```

| # | Fırsat | Kanıt |
|---|---|---|
| **1** | **Ciddi aşırı tahsis** | %12 ortalama, %35 p99 → kapasitenin ~3 katı fazla |
| **2** | **Eski nesil** | `m5` → `m7g` (Graviton) %20–40 fiyat/performans |
| **3** | **Fiyatlandırma modeli** | Hepsi On-Demand → Savings Plan + Spot karması |

**b) Kazanç ve risk**

| # | Optimizasyon | Tahmini kazanç | Risk |
|---|---|---|---|
| **1** | 20 → 8 instance (p99 %35 × 20/8 = %87... çok agresif) → **20 → 10** | **~%50** | ⚠️ Tepe yükte pay azalır |
| **2** | `m5.2xlarge` → `m7g.2xlarge` | **~%20–30** | ⚠️ ARM64 uyumluluğu *(7.1.6 kontrol listesi)* |
| **3a** | Taban kapasiteye Savings Plan (1 yıl) | **~%30** taban üzerinde | Taahhüt — kullanmazsan ödersin |
| **3b** | Durumsuz olduğu için tepe kapasiteye Spot | **~%70** o kısımda | 2 dk uyarı — durumsuz olduğu için kabul edilebilir |

**Birleşik etki (çarpımsal):**
```
Taban        : 1,00
Right-sizing : × 0,50
Graviton     : × 0,75
Fiyat modeli : × 0,65
──────────────────────
Sonuç        : ≈ 0,24  →  ~%76 tasarruf
```

**c) Hangi sırayla ve neden**

```
1. ÖNCE: Otomatik ölçekleme kur (henüz yoksa)
   → Sabit 20 instance, right-sizing'in düşmanıdır.
   → Ölçekleme olmadan azaltmak, tepe yükte kesinti riski demektir.

2. RIGHT-SIZING (%50 kazanç, sıfır risk, hemen)
   → 20 → 14 → 12 → 10 kademeli in, her adımda p99'u izle
   → Kademeli olması önemli: tek adımda yarıya inmek kör bir bahistir

3. SAVINGS PLAN (taban kapasite netleştikten SONRA)
   → Sırayı bozma: önce taahhüt edip sonra küçültürsen
     kullanmadığın kapasiteye 1 yıl para ödersin
   → Bu, en sık yapılan sıralama hatasıdır

4. SPOT (tepe kapasite için)
   → Durumsuz olduğu için uygun
   → Karışık filo: taban On-Demand/SP, tepe Spot

5. GRAVITON (en son — en çok test gerektiren)
   → Kod ve bağımlılık uyumluluğu test edilmeli
   → Kanarya dağıtımıyla kademeli geçiş
   → Diğerlerinden bağımsız, paralel yürütülebilir
```

> **Sıralamanın mantığı: önce ölçüm ve esneklik, sonra taahhüt.**
>
> **En kritik nokta 3. adımdır.** Savings Plan'ı right-sizing'den **önce** almak, çok
> sık yapılan ve pahalıya mal olan bir hatadır: 20 instance'lık taahhüt verip 10'a
> düşersen, 1 yıl boyunca kullanmadığın 10 instance'ı ödersin. **Taahhüt her zaman en
> son gelir.**

---

# Faz 7 — Sık Sorulan Sorular

> **S1: "Hangi instance tipini seçeceğimi nasıl bilirim — bir kısayol var mı?"**
>
> **Kısayol yok, ama başlangıç noktası var:**
>
> | Bildiğin | Başlangıç |
> |---|---|
> | Hiçbir şey | **`m` ailesi, orta boy** — ölç, sonra taşı |
> | CPU ağırlıklı | `c` |
> | Bellek ağırlıklı | `r` |
> | Yüksek IOPS, kalıcılık yok | `i` |
> | GPU | `g` (çıkarım) / `p` (eğitim) |
>
> **Asıl cevap:** Tahmin etme, **ölç.** *(7.6.1)* İş yükünü `m`'de çalıştır, darboğazı
> bul, o darboğaza göre taşı. Bu süreç birkaç saat sürer ve aylarca yanlış instance
> ödemekten ucuzdur.

> **S2: "Graviton'a geçmeli miyim?"**
>
> **Muhtemelen evet** — %20–40 fiyat/performans kazancı gerçektir.
>
> **Kontrol listesi (7.1.6):** Dil desteği, native bağımlılıklar, multi-arch Docker
> imajları, üçüncü parti ajanlar, CI/CD.
>
> **En sık takılan yer: üçüncü parti ajanlar** (APM, log toplayıcı, güvenlik ajanı).
> Uygulaman ARM64'te sorunsuz çalışır ama izleme ajanın çalışmaz.
>
> **Yaklaşım:** Tek bir servisle kanarya dağıtımı yap, ölç, sonra yay.

> **S3: "CloudWatch'ta bellek kullanımı neden yok?"**
>
> Çünkü hypervisor **misafirin belleğinin içini göremez.** *(Faz 6.4)* Instance'a 32 GB
> ayrılmıştır; misafir OS'in bunun ne kadarını kullandığı, ne kadarının cache olduğu
> hypervisor'e görünmez.
>
> **Çözüm:** CloudWatch Agent kur — misafir içinden metrik gönderir.
>
> Aynı sebeple disk doluluk oranı da varsayılan olarak yoktur (EBS I/O metrikleri vardır
> ama dosya sistemi doluluğu yoktur).

> **S4: "Dikey mi yatay mı ölçeklemeliyim?"**
>
> | | Dikey (daha büyük instance) | Yatay (daha çok instance) |
> |---|---|---|
> | Kolaylık | ✅ Kod değişikliği yok | ⚠️ Durumsuzluk gerekir |
> | Tavan | ❌ **Var** — en büyük boyut | ✅ Pratikte yok |
> | Dayanıklılık | ❌ **Tek hata noktası** | ✅ Bir düğüm ölse devam |
> | Maliyet verimliliği | ⚠️ Büyük boyutlar orantısız pahalı | ✅ Daha iyi |
> | Kesinti | ⚠️ Yeniden başlatma gerekir | ✅ Kesintisiz |
>
> **Kural:** Mümkünse **yatay.** Dikey ölçeklemeyi durumlu bileşenler (veritabanı birincil
> düğümü) ve yatay ölçeklenemeyen şeyler için sakla.
>
> **Not:** Faz 2.7'deki NUMA uyarısı burada devreye girer — çok büyük instance'lar NUMA
> düğümlerine yayılır ve lineer kazanç vermez.

> **S5: "Aynı uygulama neden bazı instance'larda daha hızlı çalışıyor?"**
>
> Üç ihtimal, **sırayla kontrol et:**
>
> | Sebep | Nasıl doğrularsın |
> |---|---|
> | **Komşu gürültüsü** *(7.4)* | Yeni instance başlat, aynı yükü ver, karşılaştır |
> | **Farklı fiziksel CPU** | `lscpu` — aynı instance tipi farklı CPU modellerine düşebilir |
> | **NUMA yerleşimi** *(Faz 2.7)* | `numactl --hardware`, bellek/vCPU aynı düğümde mi |
>
> İkinci satır az bilinir: AWS aynı instance tipini farklı nesil fiziksel CPU'lar
> üzerinde sunabilir. `lscpu | grep "Model name"` ile kontrol edilebilir.

> **S6: "Spot instance gerçekten güvenli mi?"**
>
> **İş yüküne bağlı, altyapıya değil.**
>
> Spot, 2 dakika uyarıyla kesilir. Soru şu: **iş yükün 2 dakikada güvenli şekilde
> durabilir mi?**
>
> | ✅ Evet | ❌ Hayır |
> |---|---|
> | Durumsuz web (yük dengeleyici arkasında) | Veritabanı birincil düğümü |
> | Toplu işleme (checkpoint'li) | Durumlu tek örnek |
> | CI/CD runner | Uzun süren, kaydedilmeyen hesap |
> | ML eğitimi (checkpoint'li) | Kesinti kabul edilmeyen üretim |
>
> **Pratik desen:** Karışık filo — taban kapasite On-Demand/Savings Plan, tepe kapasite
> Spot. Spot kesilse bile taban ayakta kalır.

> **S7: "Bu haritayı bitirdim. Şimdi ne öğrenmeliyim?"**
>
> Bu harita **temel katmandır** — yolculuğun kabaca ilk üçte biri.
>
> **Paralel tamamlayıcılar (aynı seviyede):**
> - **Linux yol haritası** — çalışan sunucuyu okumak, arıza avlamak
> - **Network yol haritası** — protokoller: IP, TCP, DNS, TLS, yönlendirme
>
> **Sonraki katman (uygulama):**
> ```
> 4. AWS Core Services (EC2/VPC/S3/IAM/RDS — SAA odaklı)
> 5. IaC / Terraform
> 6. Docker → Kubernetes / ECS-EKS
> 7. CI/CD + Git + Python (boto3) otomasyon
> 8. Mimari desenler + gözlemlenebilirlik + FinOps
> ```
>
> **Bu haritanın sana kazandırdığı şey, o katmanın üstünde durduğu zemini görmektir.**
> Kubernetes'in kaynak limitlerini öğrenirken cgroup'u *(6.7.2)*, EBS seçerken IOPS
> fiziğini *(Faz 3)*, pod yerleşimi kararlarında NUMA'yı *(Faz 2.7)* bileceksin.

---

# Faz 7 — Kendini Sına

## Bölüm A — Temel

**A1.** `m7gd.2xlarge` isminin her parçasını çöz.

**A2.** `c` ailesini `m`'den ayıran iki fiziksel özellik nedir?

**A3.** Instance store ile EBS arasındaki üç temel fark nedir?

**A4.** gp3'ün gp2'ye göre temel avantajı nedir?

**A5.** Komşu gürültüsünün üç mekanizmasını say.

**A6.** Beş darboğaz sınıfını say.

**A7.** Right-sizing'de CPU için hedef aralık nedir ve neden %100 değil?

## Bölüm B — Mekanizma

**B1.** Graviton'da 1 vCPU ile x86'da 1 vCPU neden farklı şeylerdir? Sonucu ne?

**B2.** L3 cache kirlenmesi neden hiçbir standart metrikte görünmez?

**B3.** "Uygulamam yavaş" duyduğunda izleyeceğin 5 adımlık yordamı sırala.

**B4.** Bekleme bound nedir ve diğer dört sınıftan nasıl ayrılır?

**B5.** Instance EBS bant genişliği tavanı neden ayrı bir kontrol gerektirir?

**B6.** Nitro hangi komşu gürültüsü mekanizmalarını çözdü, hangilerini çözmedi? Neden?

**B7.** Aşırı ve yetersiz tahsisin maliyetleri neden asimetriktir?

## Bölüm C — Uygulama ve muhakeme

**C1.** Bir ekip video kodlama servisi kuruyor. İş: kullanıcı video yükler, 4 farklı
çözünürlüğe dönüştürülür. İşler kuyrukta birikebilir, gecikme kritik değil. Günde 3 saat
yoğun, 21 saat sakin.
Instance ailesi, boyut, fiyatlandırma modeli ve depolama önerini gerekçeleriyle yaz.

**C2.** Bir PostgreSQL birincil düğümü: 500 GB veri, çalışma seti ~120 GB, %85 okuma,
p99 sorgu süresi 50 ms hedefi.
Instance ve EBS önerini gerekçelendir. Hangi metrikleri izlersin?

**C3.** `iostat -x` çıktısı:
```
Device  r/s     w/s    rkB/s    wkB/s  r_await w_await aqu-sz %util
nvme1n1 15980,0 20,0  63920,0   80,0    0,62    0,71   10,20  99,9
```
Volume: gp3, 16.000 IOPS. Instance: `m7i.4xlarge`. Teşhisin?

**C4.** Bir ML ekibi `p4d.24xlarge` (8× A100) kullanıyor. `nvidia-smi` GPU kullanımını
%35 gösteriyor. Ekip `p5.48xlarge`'a (8× H100) geçmek istiyor.
Bu mantıklı mı? Önce ne yapılmalı?

**C5.** Bir API'nin p99'u gece 02:00'de her gün 5 kata çıkıyor, 03:00'te normale dönüyor.
Trafik o saatte en düşük seviyede. CPU, bellek, ağ normal. `iostat` await 15 ms (normalde
1 ms).
Teşhisin ve çözümün?

**C6.** Bir ekip maliyeti düşürmek için tüm üretim instance'larını `m6i`'den
`t3`'e taşımayı öneriyor. Ortalama CPU kullanımı %35.
Değerlendir ve alternatif öner.

**C7.** Bir e-ticaret sitesi Black Friday'e hazırlanıyor. Normal trafik 1.000 istek/s,
beklenen tepe 15.000 istek/s. Mevcut: 10 × `m7i.2xlarge`, ortalama CPU %30.
Kapasite planını yaz: kaç instance, hangi model, hangi riskler?

---

## Cevap Anahtarı

### Bölüm A

**A1.** *(7.1.0)*
```
m   = genel amaçlı aile
7   = 7. nesil
g   = Graviton (ARM64)
d   = yerel NVMe (instance store) var
2xlarge = 8 vCPU
```

**A2.** *(7.1.2)* **Yüksek sürekli clock** (tek thread performansı) ve **vCPU başına daha
fazla L3 cache**. RAM oranının düşük olması bir sonuçtur, sebep değil.

**A3.** *(7.1.4)*
| Instance store | EBS |
|---|---|
| PCIe doğrudan bağlı | **Ağ üzerinden** |
| ~50–100 μs, milyonlarca IOPS | ~1 ms, kotalı IOPS |
| **Instance durunca kaybolur** | Kalıcı, snapshot alınabilir |

**A4.** *(7.3.2)* **IOPS, disk boyutundan bağımsız ayarlanır.** gp2'de IOPS = boyut × 3
olduğu için yüksek IOPS istemek gereksiz büyük disk almayı zorunlu kılar.

**A5.** *(7.4.1)* CPU zamanı yarışması (steal time), **L3 cache kirlenmesi**, **bellek
bant genişliği doyması**. Son ikisi ölçülemez ve Nitro tarafından çözülmemiştir.

**A6.** *(7.5.1)* CPU bound, memory bound, I/O bound, network bound, **bekleme bound**.

**A7.** *(7.6.2)* **%40–60 ortalama.** %100 hedeflenmez çünkü kuyruk eğrisi üsteldir —
%95 doluluk, %50 doluluğun ~8 katı gecikme demektir *(Faz 3.4.4)*. Son %20 kapasite,
p99'u patlatarak ödenir.

### Bölüm B

**B1.** *(7.1.6, Faz 1.5.2)* x86'da 1 vCPU = 1 SMT iş parçacığı = **fiziksel çekirdeğin
yarısı.** Graviton'da 1 vCPU = **1 tam fiziksel çekirdek.**

**Sonuç:** `c7g.4xlarge` (16 fiziksel çekirdek) ile `c7i.4xlarge` (8 fiziksel çekirdek)
aynı vCPU sayısına sahip ama farklı donanımdır. Graviton'un avantajı SMT'den az
faydalanan iş yüklerinde ilan edilenden **büyük**, çok faydalananlarda **küçüktür**.

**B2.** *(7.4.3)* Çünkü kirlenme **komut sayısını değil, komut başına süreyi** artırır.
- CPU kullanımı aynı veya yüksek görünür (aynı iş için daha çok çevrim harcanıyor)
- Steal time değişmez (vCPU çekirdeği **alıyor**, sadece bellek bekliyor)
- Bellek/disk/ağ kullanımı değişmez

Düşen şey **IPC**'dir *(Faz 1.3.2)* ve onu ölçmek donanım performans sayaçları gerektirir
— cloud instance'larında genelde kısıtlıdır.

**B3.** *(7.5.2)*
```
0. Doğru soruyu sor (ne zamandan beri, p50 mi p99 mu, ne kadar, sürekli mi)
1. CPU     — top/mpstat: us, sy, wa, st
2. Bellek  — free/vmstat: available, si/so
3. Disk    — iostat -x: await, aqu-sz, limitler
4. Ağ      — ss/ethtool/sar: drop, retransmit, throughput
5. Hiçbiri değilse → bekleme bound
```

**B4.** *(7.5.7)* **Hiçbir kaynak doygun değilken uygulamanın yavaş olmasıdır.** Diğer
dördünde bir kaynak tavana dayanır; burada sistem bir şeyi **bekler**: kilit, uzak çağrı,
bağlantı/thread havuzu, GC duraklaması, yavaş aşağı akış servisi.

**Ayırt edici:** Donanım eklemek hiçbir şeyi düzeltmez.

**B5.** *(7.3.5)* Volume'un ve instance'ın limitleri **ayrı ayrı** uygulanır. 64.000 IOPS
sağlayabilen bir io2 volume, 10.000 IOPS'a izin veren bir instance'a takılırsa efektif
limit 10.000'dir — fark boşa ödenir.

**B6.** *(7.2.2)*
| Çözdü | Çözmedi |
|---|---|
| Hypervisor CPU tüketimi | **L3 cache yarışması** |
| Ağ işleme yarışması | **Bellek bant genişliği yarışması** |
| EBS I/O yarışması (büyük ölçüde) | |

**Neden:** Nitro, işleri ayrı silikona taşıyabildiği kaynakları çözdü. Ama L3 ve bellek
kanalları **CPU paketinin içindedir** ve fiziksel olarak paylaşılır; taşınamaz ve
donanım seviyesinde kotalanmaz *(Faz 6.1.2)*.

**B7.** *(7.6.3)*
```
Aşırı tahsis  : maliyet FATURADA görünür, ölçülebilir, kolay düzeltilir
Yetersiz tahsis: maliyet GİZLİDİR — kaybedilen istek, müşteri, itibar, olay yönetimi
```
Bu asimetri, makul miktarda aşırı tahsisi rasyonel kılar. Ama "makul" ölçümle
tanımlanmalıdır.

### Bölüm C

**C1.** *(7.1.2, 7.6.4)*

| Karar | Öneri | Gerekçe |
|---|---|---|
| **Aile** | **`c` (c7g veya c7i)** | Video kodlama saf CPU işidir; bellek az gerekir |
| **İşlemci** | **Graviton (`c7g`)** denenmeli | Kodlama kütüphaneleri ARM64'te iyi; %20–40 kazanç |
| **Boyut** | Orta (`.4xlarge`), **yatayda ölçekle** | Kuyruk tabanlı → mükemmel yatay ölçeklenir |
| **Fiyatlandırma** | **Spot (ağırlıklı)** | Kuyruk + gecikme kritik değil = **ideal Spot iş yükü** |
| **Ölçekleme** | Kuyruk uzunluğuna göre otomatik | 3 saat yoğun / 21 saat sakin → sabit kapasite israf |
| **Depolama** | Geçici iş için **instance store** (`c7gd`), çıktı **S3** | Kodlama sırasında yüksek I/O, kalıcılık gerekmez |

> **Bu iş yükü Spot için ders kitabı örneğidir:** durumsuz, kuyruk tabanlı, kesintiye
> dayanıklı (iş kuyruğa geri döner), gecikmeye duyarsız. %70–90 tasarruf, neredeyse
> sıfır risk.

**C2.** *(7.1.3, 7.3, 7.5.4)*

| Karar | Öneri | Gerekçe |
|---|---|---|
| **Aile** | **`r` (r7i/r7g)** | Çalışma seti 120 GB — **RAM'e sığmalı** |
| **Boyut** | `r7i.4xlarge` (16 vCPU, **128 GB**) | 120 GB çalışma seti + OS + bağlantılar |
| **EBS** | **gp3, 500 GB, 12.000 IOPS, 500 MB/s** | Çalışma seti RAM'de → disk yükü orta |
| **Alternatif** | p99 tutmuyorsa **io2** | Sub-ms gecikme gerekiyorsa |

**Kritik gerekçe:** Çalışma seti (120 GB) RAM'e sığdığında **disk erişimi neredeyse
tamamen ortadan kalkar** *(Faz 2.1'in dersi: en hızlı I/O yapılmayan I/O'dur)*.
%85 okuma oranı bunu daha da güçlendirir. Bu yüzden **para diske değil RAM'e
harcanmalıdır.**

`shared_buffers` ≈ RAM'in %25'i (32 GB), geri kalanı OS sayfa cache'i olarak çalışır.

**İzlenecek metrikler:**
```
□ Cache hit oranı (>%99 hedef)  ← en önemli tek metrik
□ p99 sorgu süresi
□ EBS: VolumeQueueLength (<2), read/write IOPS
□ Bellek: available, swap (0 olmalı)
□ Bağlantı sayısı / havuz doygunluğu
□ Replikasyon gecikmesi (replika varsa)
□ Checkpoint sıklığı ve süresi
```

**C3.** *(7.3.5, Faz 3.4.5)*

**Okuma:**
```
r/s + w/s = 16.000  ← TAM OLARAK gp3 limitinde
rkB/s ÷ r/s = 63920 ÷ 15980 = 4 KiB ortalama → OLTP/rastgele desen
await 0,62 ms  ← düşük, disk sağlıklı yanıt veriyor
aqu-sz 10,2    ← makul
%util 99,9     ← NVMe'de YANILTICI, dikkate alma (Faz 3.4.5)
```

**Teşhis: IOPS tavanına dayanılmış, ama sistem sağlıklı çalışıyor.**

Önceki örnekten (Düşün 7.2) farkı önemli: orada `await` 24 ms ve `aqu-sz` 71'di —
**patolojik.** Burada await 0,62 ms; disk isteklere hızlı cevap veriyor, sadece **daha
fazlasını kabul etmiyor.**

**Ne yapmalı:**
1. **Önce instance tavanını kontrol et** — `m7i.4xlarge` 16.000 IOPS'a izin veriyor mu?
   Vermiyorsa volume'u büyütmek işe yaramaz.
2. İzin veriyorsa **gp3 IOPS'ini artır** (16.000 → daha fazlası için io2 gerekir; gp3
   tavanı 16.000'dir)
3. **16.000'den fazlası gerekiyorsa io2 Block Express**
4. Paralel olarak: **erişim desenini incele** — 4 KiB rastgele okuma çok; indeks veya
   önbellek eksik olabilir

> **Not: gp3'ün tavanı 16.000 IOPS'tir.** Tam oradasın, yani gp3 ile yapabileceğin bir
> şey kalmadı — ya io2'ye geçeceksin ya da IOPS ihtiyacını azaltacaksın.

**C4.** *(7.1.5, Faz 4.2.4)*

**Mantıklı değil — %35 GPU kullanımıyla daha hızlı GPU almak, boşta duran kapasiteyi
büyütmektir.**

```
Mevcut: A100'lerin %35'i kullanılıyor → %65 boşta
H100'e geçsen: daha hızlı GPU'nun daha büyük kısmı boşta kalacak
Fiyat 2 katına çıkacak, kullanım oranı DÜŞECEK
```

**Önce yapılması gerekenler:**

```
1. DARBOĞAZI BUL
   nvidia-smi dmon        # GPU kullanımı, bellek, PCIe
   • GPU %35, CPU %100     → veri ön işleme darboğazı (en sık)
   • GPU %35, disk doygun  → veri yükleme darboğazı
   • GPU %35, ağ doygun    → çok düğümlü senkronizasyon (Faz 5.4.3)

2. VERİ HATTINI DÜZELT (en sık çözüm)
   • DataLoader worker sayısını artır
   • Veriyi önceden işle, tekrar tekrar yapma
   • Veriyi instance store'a (NVMe) koy, EBS'ten okuma
   • Prefetch ve pin_memory kullan

3. BATCH BOYUTUNU ARTIR
   • Küçük batch = GPU sürekli veri bekler
   • GPU belleği dolana kadar büyüt

4. KARMA HASSASİYET (mixed precision)
   • Hem hızlandırır hem bellek açar

5. PCIe KONTROLÜ (Faz 4.2.4)
   • CPU↔GPU transferi darboğaz mı?
   • Gereksiz transferleri kaldır, veriyi GPU'da tut
```

> **GPU kullanımını %80+'a çıkarmak, çoğu zaman daha büyük GPU almaktan hem ucuz hem
> etkilidir.** Faz 4.2.4'ün dersi: **darboğaz en pahalı bileşende değil, onu besleyen
> yoldadır.**
>
> Ve eğer %80'e çıkardıktan sonra hâlâ yetmiyorsa, **o zaman** H100 mantıklı bir
> karardır — çünkü artık gerçekten GPU sınırlısın.

**C5.** *(7.5.5)*

**Teşhis: Zamanlanmış bir gece işi diski doyuruyor.**

Kanıt zinciri:
```
Her gün AYNI SAATTE       → zamanlanmış iş (cron, yedek, batch)
Trafik EN DÜŞÜK seviyede  → yük kaynaklı değil
await 1 ms → 15 ms        → disk kuyruğu doldu
CPU/bellek/ağ normal      → sadece I/O
```

**En olası suçlular:**
| Aday | Nasıl doğrularsın |
|---|---|
| **EBS snapshot** | İlk snapshot veya büyük değişiklik → yoğun okuma |
| **Veritabanı yedeği** (`pg_dump`, mysqldump) | Tam tablo taraması |
| **Log rotasyonu / sıkıştırma** | Büyük dosya okuma-yazma |
| **VACUUM / ANALYZE** (PostgreSQL) | Otomatik bakım penceresi |
| **Antivirüs/uyumluluk taraması** | Tüm dosya sistemi taraması |
| **Batch ETL işi** | Büyük veri okuma |

```bash
# Doğrulama
crontab -l; ls /etc/cron.d/
systemctl list-timers
iotop -o          # 02:00'de çalıştır — hangi process
```

**Çözümler (etki sırasıyla):**
1. **İşi yeniden zamanla** — trafiğin p99'unun önemli olmadığı bir saate (ücretsiz)
2. **I/O önceliğini düşür** — `ionice -c3 <komut>` (ücretsiz)
3. **Hızını sınırla** — yedek/dump araçlarının throttle seçenekleri
4. **İşi ayrı bir replikaya taşı** — üretim diskine hiç dokunmasın
5. IOPS'i artır — **en son çare, para harcar ve kök sebebi çözmez**

> **1 ve 2 numara ücretsizdir ve vakaların çoğunu çözer.** "p99 arttı, io2'ye geçelim"
> demek burada **kök sebebi görmeden para harcamaktır.**

**C6.** *(7.1, Faz 6.5.3)*

**Değerlendirme: Bu öneri yanlıştır ve üretimde arızaya yol açar.**

```
Ortalama CPU kullanımı: %35

t3.large  tabanı: %30    → %35 > %30 → kredi TÜKENİR
t3.xlarge tabanı: %40    → %35 < %40 → sınırda, riskli
```

**Kural** *(Faz 6.5.3)*: t ailesi, **ortalama kullanım taban performansın belirgin
altındaysa** doğrudur. %35 ortalama ile t3.large'a geçmek, her gün kredi tükenmesi ve
%30 performansa düşüş demektir.

**Ayrıca:** Ortalama %35 demek, tepe anlarının çok daha yüksek olduğu anlamına gelir —
ve kredi tam da tepe anlarda tükenir.

**Doğru alternatifler (etki sırasıyla):**

| # | Alternatif | Kazanç | Risk |
|---|---|---|---|
| **1** | **Right-sizing** — %35 kullanım zaten fazla tahsis, boyut düşür | **~%30–40** | Düşük (kademeli) |
| **2** | **Graviton** — `m6i` → `m7g` | **~%20–30** | ARM64 testi |
| **3** | **Savings Plan** — taban kapasiteye | **~%30** | 1 yıl taahhüt |
| **4** | **Otomatik ölçekleme** — sabit kapasite yerine | Değişken | Kurulum işi |
| **5** | Spot (durumsuz katmanlar için) | %70–90 o kısımda | Kesinti toleransı |

**Birleşik: %60+ tasarruf, t3'ün riski olmadan.**

> **Asıl ders:** Ekip doğru problemi (maliyet) görmüş ama **yanlış aracı** seçmiş.
> t ailesi bir **indirim** değil, **farklı bir performans modelidir.** İş yükü o modele
> uymuyorsa ucuz değil, sadece bozuktur.

**C7.** *(7.6)*

**Mevcut durum analizi:**
```
10 × m7i.2xlarge (8 vCPU) = 80 vCPU
1.000 istek/s'de CPU %30
→ İstek başına yaklaşık: 80 vCPU × 0,30 / 1000 = 0,024 vCPU-s/istek
```

**Tepe için gereken:**
```
15.000 istek/s × 0,024 = 360 vCPU  (%100 kullanımda)

%60 hedef kullanımda (7.6.2 — kuyruk eğrisi payı):
  360 / 0,60 = 600 vCPU
  = 75 × m7i.2xlarge     ← mevcut 10'dan 7,5 kat

Alternatif: 38 × m7i.4xlarge (16 vCPU) — daha az örnek, daha kolay yönetim
```

**Kapasite planı:**

| Katman | Yapılandırma | Gerekçe |
|---|---|---|
| **Taban** | 15 instance, Savings Plan | Normal trafik + emniyet payı |
| **Ölçekleme** | Otomatik, 15 → 80, On-Demand | Tepeye çıkış |
| **Ek pay** | Maks'ı 100'e ayarla | Tahmin %25 yanılırsa |
| **Isıtma** | **Etkinlikten 1 saat önce ön-ölçekleme** | **Kritik — aşağıda** |

**Riskler ve önlemler:**

| Risk | Önlem |
|---|---|
| **Otomatik ölçekleme yetişemez** (yeni instance 2–5 dk) | **Zamanlanmış ön-ölçekleme** — tepeyi bekleme |
| **Veritabanı darboğaz olur** | ⚠️ **En büyük risk** — uygulama katmanı ölçeklenir, veritabanı ölçeklenmez. Okuma replikaları, bağlantı havuzu (pgbouncer), önbellek |
| **Bağlantı havuzu tükenir** | 80 instance × N bağlantı = veritabanı limitini aşabilir. **Hesapla.** |
| **Hesap/servis kotaları** | vCPU kotası, ELB hedef sayısı, NAT gateway bant genişliği — **önceden artır** |
| **Yük dengeleyici ısınmamış** | AWS'e önceden bildir veya trafiği kademeli artır |
| **Tahmin yanlış** (30.000 istek/s gelirse) | Maks kapasiteyi cömert ayarla + kademeli yanıt bozulma (graceful degradation) |
| **Aşağı akış bağımlılıkları** | Ödeme, envanter, kargo servisleri de ölçeklenebiliyor mu? |

**Doğrulama — en önemli adım:**
```
□ 20.000 istek/s ile yük testi yap (hedefin %133'ü)
□ Testi EN AZ 1 SAAT sürdür (burst/kredi tuzağı — 7.6.1)
□ Ölçeklemenin gerçekten çalıştığını gör, varsayma
□ Veritabanı davranışını ayrıca izle
□ Geri dönüş (rollback) planı hazır olsun
```

> **En kritik uyarı:** **Uygulama katmanı ölçeklenir, veritabanı ölçeklenmez.** 80
> instance başarıyla ayağa kalkar ve hepsi aynı veritabanına yüklenir. Black Friday
> arızalarının çoğu uygulama katmanında değil, **veritabanı bağlantı havuzunda** olur.
>
> Bu, bu haritanın kapanış dersidir: **darboğaz her zaman yer değiştirir.** Birini
> çözdüğünde bir sonrakini bulursun — ve hazırlık, bir sonrakinin nerede olacağını
> önceden bilmektir.

---

## Puanlama

| Doğru sayısı | Değerlendirme |
|---|---|
| 18–21 | **Haritayı bitirdin.** Bu bilgi artık iş çıktısı. |
| 14–17 | Çok iyi. 7.5'i (darboğaz tespiti) bir kez daha oku. |
| 10–13 | 7.4 ve 7.5'i tekrar çalış — bunlar günlük iş becerileri. |
| 0–9 | C sorularını cevaplarıyla birlikte tekrar çöz. Hangi fazlara dönmen gerektiğini cevaplar söylüyor. |

> **C bölümü bu haritanın gerçek sınavıdır.** A ve B bilgiyi ölçer; C **kararı** ölçer.
> Bir C sorusunu kaçırdıysan, cevabındaki faz referanslarını takip et — eksik olan bilgi
> değil, bağlantıdır.

---

# Harita Kapanışı

## Nereden nereye

| Faz | Ne öğrendin | Hangi karara dönüştü |
|---|---|---|
| **0** | Transistör, kapı, flip-flop, SRAM/DRAM | Cache'in neden pahalı, RAM'in neden ucuz olduğu |
| **1** | Pipeline, IPC, çekirdek/thread, ISA | vCPU ne demek, clock mu çekirdek mi |
| **2** | Cache hiyerarşisi, sanal bellek, NUMA | **En sık mimari kararı etkileyen alan** |
| **3** | HDD/SSD fiziği, IOPS/throughput/latency | Her EBS kararı |
| **4** | PCIe, DMA, kesme, chipset | GPU ve yüksek I/O iş yükleri |
| **5** | NIC, switching, bant genişliği vs gecikme | Bölge/AZ yerleşimi, ağ darboğazları |
| **6** | Hypervisor, VM exit, steal time, container | EC2'nin altında ne olduğu |
| **7** | Instance aileleri, darboğaz tespiti, kapasite | **Hepsinin karara dönüşmesi** |

## Bu haritanın taşıdığı beş fikir

Yedi faz boyunca aynı beş fikir farklı kılıklarda tekrar etti. Kalıcı olan bunlardır:

**1. Hiyerarşi, hız ile maliyet arasındaki takasın ürünüdür.**
Register → L1 → L2 → L3 → RAM → SSD → HDD → ağ. Her basamak bir öncekinden ucuz ve yavaş.
Aynı yapı cache'te, depolamada, CDN'de, veritabanı önbelleğinde tekrar eder.

**2. Paylaşılan tek kaynak, ölçek büyüdükçe darboğaz olur; çözüm anahtarlamadır.**
Bellek kanalı, depolama kuyruğu, PCIe şeridi, ağ switch'i — dördü de aynı evrimi yaşadı:
paylaşımlı hat → nokta-nokta.

**3. Kuyruklar üstel davranır.**
%50 doluluk 1×, %95 doluluk 8×, %99 doluluk 40× gecikme. Diskte `await`, ağda jitter,
CPU'da run queue. **Kaynağı %100'e kadar doldurmak bir tasarruf değil, bir arızadır.**

**4. Darboğaz en pahalı bileşende değil, onu besleyen yoldadır.**
GPU değil PCIe, disk değil sorgu deseni, hat değil TCP penceresi, CPU değil bellek erişimi.

**5. Donanım almadan önce erişim desenini düzelt.**
Satır/sütun sıralı erişim (10×), eksik indeks (60×), N+1 sorgu (100×), cache dostu veri
yapısı (5×). **Bu haritadaki en büyük performans kazançlarının hiçbiri para
gerektirmedi.**

## Şimdi ne yapmalı

**Paralel tamamlayıcılar (aynı temel katman):**
- **Linux yol haritası** — çalışan bir sunucuyu okumak, arızayı avlamak
- **Network yol haritası** — protokoller: IP, TCP, DNS, TLS, yönlendirme, firewall

**Sonraki katman (uygulama):**
```
4. AWS Core Services (EC2/VPC/S3/IAM/RDS — SAA odaklı)   ← en acil
5. IaC / Terraform
6. Docker → Kubernetes / ECS-EKS
7. CI/CD + Git + Python (boto3) otomasyon
8. Mimari desenler + gözlemlenebilirlik + FinOps         ← capstone
```

> **Bu haritanın sana kazandırdığı şey servis bilgisi değil, zemindir.**
>
> Kubernetes'in `resources.limits`'ini öğrenirken cgroup'u göreceksin *(6.7.2)*. RDS
> instance sınıfı seçerken bellek bant genişliğini *(2.4.3)*. S3 çok parçalı yüklemeyi
> anlarken BDP'yi *(5.3.3)*. Lambda'nın soğuk başlangıcını okurken Firecracker'ı
> *(6.7.4)*.
>
> **Servisler değişir, katman adları değişir, fiyatlar değişir. Fizik değişmez.**

---

*Faz 7 tamamlandı. **Hardware yol haritası bitti.***

→ **[Haritanın başına dön](README.md)** · **[Ek A — Referans Tabloları](EK_A_Referans_Tablolari.md)** · **[Ek B — Terim Sözlüğü](EK_B_Terim_Sozlugu.md)**
