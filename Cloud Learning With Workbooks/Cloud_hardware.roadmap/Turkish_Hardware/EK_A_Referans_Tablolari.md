# Ek A — Referans Tabloları

> **Navigasyon:** [◀ README](README.md) · **Ek A** · [Ek B — Terim Sözlüğü ▶](EK_B_Terim_Sozlugu.md)

---

> **Bu ek, öğretmek için değil hatırlatmak için vardır.** Haritayı bitirdikten sonra
> açık tutulacak sayfa budur. Her tablonun yanında hangi bölümden geldiği yazılıdır —
> bir rakam anlamsız geldiğinde oraya dön.

**İçindekiler**
- [A.1 Gecikme merdiveni](#a1-gecikme-merdiveni)
- [A.2 Birimler ve çevrimler](#a2-birimler-ve-çevrimler)
- [A.3 Bellek referansı](#a3-bellek-referansı)
- [A.4 Depolama referansı](#a4-depolama-referansı)
- [A.5 Bus ve PCIe referansı](#a5-bus-ve-pcie-referansı)
- [A.6 Ağ referansı](#a6-ağ-referansı)
- [A.7 Sanallaştırma referansı](#a7-sanallaştırma-referansı)
- [A.8 AWS instance aileleri](#a8-aws-instance-aileleri)
- [A.9 EBS tipleri](#a9-ebs-tipleri)
- [A.10 Komut referansı](#a10-komut-referansı)
- [A.11 Teşhis akış şeması](#a11-teşhis-akış-şeması)
- [A.12 Formül özeti](#a12-formül-özeti)

---

## A.1 Gecikme merdiveni

*(Faz 2.1.1, 3.1.3, 5.3.4)* — **Bu haritanın tek en önemli tablosu.**

| İşlem | Süre | İnsan ölçeğinde (1 çevrim = 1 saniye) |
|---|---|---|
| 1 CPU çevrimi (3 GHz) | 0,3 ns | 1 saniye |
| L1 cache erişimi | ~1 ns (4 çevrim) | 4 saniye |
| L2 cache erişimi | ~4 ns (12 çevrim) | 12 saniye |
| L3 cache erişimi | ~13 ns (40 çevrim) | 40 saniye |
| **RAM erişimi** | **~70 ns (200 çevrim)** | **3,5 dakika** |
| Uzak NUMA düğümü RAM | ~120 ns | 6 dakika |
| **VM exit** | **~0,3–1,7 μs** | **17 dakika – 1,5 saat** |
| NVMe SSD okuma | ~50–100 μs | **1,5–3 gün** |
| SATA SSD okuma | ~100–200 μs | 3–6 gün |
| **EBS gp3 okuma** | **~1 ms** | **~1 ay** |
| Aynı AZ ağ RTT | ~0,3–0,5 ms | ~2 hafta |
| AZ'ler arası RTT | ~1–2 ms | ~1–2 ay |
| **HDD rastgele okuma** | **~10 ms** | **~1 yıl** |
| Kıtalar arası RTT | ~100–200 ms | **10–20 yıl** |

> **Bu tabloyu ezberleme, iç geçir.** "RAM erişimi yavaş" cümlesi ancak bu merdivende
> anlam kazanır.

---

## A.2 Birimler ve çevrimler

*(Faz 0.1, 5.2.3)*

**Veri boyutu**
| Birim | Değer |
|---|---|
| 1 byte | 8 bit |
| 1 KiB | 1.024 byte |
| 1 MiB | 1.024 KiB |
| 1 GiB | 1.024 MiB |
| 1 TiB | 1.024 GiB |

> **KiB vs KB:** Depolama üreticileri 1 KB = 1.000 byte kullanır; işletim sistemleri
> 1 KiB = 1.024 byte. **1 TB disk, işletim sisteminde 931 GiB görünür** — eksik değildir.

**Hız çevrimleri**
```
Gbps ÷ 8 = GB/s          (1 Gbps = 125 MB/s)
GB/s × 8 = Gbps

Pratikte protokol ek yüküyle ~%95'i kullanılabilir.
```

| Hat | Teorik | Pratik |
|---|---|---|
| 1 Gbps | 125 MB/s | ~118 MB/s |
| 10 Gbps | 1,25 GB/s | ~1,18 GB/s |
| 25 Gbps | 3,125 GB/s | ~2,96 GB/s |
| 100 Gbps | 12,5 GB/s | ~11,8 GB/s |

**Zaman**
| Birim | Saniye cinsinden |
|---|---|
| 1 ms (milisaniye) | 10⁻³ |
| 1 μs (mikrosaniye) | 10⁻⁶ |
| 1 ns (nanosaniye) | 10⁻⁹ |

---

## A.3 Bellek referansı

### Cache hiyerarşisi *(Faz 2.3)*

| Seviye | Tipik boyut | Gecikme | Paylaşım |
|---|---|---|---|
| Register | ~1 KB | 0 çevrim | Çekirdeğe özel |
| L1i / L1d | 32–64 KB (her biri) | ~4 çevrim | **Çekirdeğe özel** |
| L2 | 512 KB – 2 MB | ~12 çevrim | Çekirdeğe özel |
| **L3** | 8–100+ MB | ~40 çevrim | **TÜM çekirdekler — komşu gürültüsü burada** |

| Sabit | Değer |
|---|---|
| **Cache satırı** | **64 byte** — her erişimde bu kadar okunur |
| Sayfa boyutu | 4 KiB (standart), 2 MiB (huge), 1 GiB (gigantic) |
| TLB giriş sayısı | ~1.536 (tipik) |
| **TLB erişim alanı (4 KiB)** | 1.536 × 4 KiB = **6 MiB** |
| **TLB erişim alanı (2 MiB huge)** | 1.536 × 2 MiB = **3 GiB** (512×) |

### Hit oranı etkisi *(Faz 2.3.6)*

```
Ortalama erişim = (hit_oranı × L1) + (ıska_oranı × RAM)
```

| Hit oranı | Ortalama erişim (4 / 200 çevrim) | Göreli hız |
|---|---|---|
| %90 | 23,6 çevrim | 1× |
| %95 | 13,8 çevrim | 1,7× |
| %99 | 5,96 çevrim | **4,0×** |
| %99,9 | 4,2 çevrim | 5,6× |

> **%90'dan %99'a çıkmak 4 kat hızlandırır.** Cache optimizasyonunun kaldıracı budur.

### DRAM ve bant genişliği *(Faz 2.4)*

| Nesil | Tipik hız | Kanal başına bant genişliği |
|---|---|---|
| DDR3-1600 | 1600 MT/s | 12,8 GB/s |
| DDR4-3200 | 3200 MT/s | 25,6 GB/s |
| DDR5-4800 | 4800 MT/s | 38,4 GB/s |

```
Toplam bant genişliği = Kanal sayısı × Kanal başına hız

8 kanal DDR5-4800 = 307 GB/s
4 kanal DDR5-4800 = 154 GB/s
```

> **Nesiller arasında bant genişliği katlanır, gecikme neredeyse sabit kalır** (~70 ns).
> Bu, memory wall'un özüdür *(Faz 2.1.2)*.

### NUMA *(Faz 2.7)*

| Erişim | Göreli mesafe | Göreli gecikme |
|---|---|---|
| Yerel düğüm | 10 | 1× |
| Komşu düğüm | 21 | **~1,5–2×** |

---

## A.4 Depolama referansı

### HDD vs SSD *(Faz 3.1, 3.2)*

| | HDD (7200 RPM) | SATA SSD | NVMe SSD |
|---|---|---|---|
| **Rastgele IOPS** | **80–120** | 50.000–100.000 | **500.000–1.000.000+** |
| Sıralı throughput | 150–250 MB/s | ~550 MB/s | 3.000–7.000 MB/s |
| Gecikme | ~10 ms | ~100 μs | **~50 μs** |
| Kuyruk sayısı | 1 | 1 (32 derinlik) | **65.535 (her biri 65.536)** |

**HDD gecikme bileşenleri (7200 RPM):**
```
Seek (arama)     : ~4 ms    ← kafa hareketi
Rotational       : ~4,17 ms ← yarım tur (60000/7200/2)
Transfer         : ~0,1 ms
─────────────────────────
Toplam           : ~8,3 ms  → ~120 IOPS
```

> **HDD'nin rastgele IOPS'i 30 yıldır artmadı** — çünkü mekanik. Kapasite 1000 kat
> arttı, IOPS aynı kaldı.

### NAND tipleri *(Faz 3.2.4)*

| Tip | Bit/hücre | Yazma dayanıklılığı | Hız | Maliyet |
|---|---|---|---|---|
| SLC | 1 | ~100.000 döngü | En hızlı | En pahalı |
| MLC | 2 | ~10.000 | Hızlı | Pahalı |
| TLC | 3 | ~3.000 | Orta | Orta |
| QLC | 4 | ~1.000 | Yavaş | **En ucuz** |

### IOPS / Throughput ilişkisi *(Faz 3.4.1)*

```
Throughput = IOPS × I/O boyutu
```

| I/O boyutu | 3.000 IOPS'te throughput |
|---|---|
| 4 KiB | 12 MB/s |
| 16 KiB | 48 MB/s |
| 64 KiB | 192 MB/s |
| 1 MiB | 3.000 MB/s (throughput limitine takılır) |

> **Aynı IOPS, farklı throughput.** "Hangisi darboğaz" sorusu I/O boyutuyla cevaplanır.

### Kuyruk eğrisi *(Faz 3.4.4)* — **üç fazda tekrar eder**

| Doluluk | Göreli gecikme |
|---|---|
| %50 | 1× |
| %80 | 2,5× |
| %90 | 5× |
| **%95** | **8×** |
| **%99** | **40×** |

### RAID seviyeleri *(Faz 3.5)*

| Seviye | Min. disk | Kapasite | Okuma | Yazma | Dayanıklılık |
|---|---|---|---|---|---|
| 0 | 2 | %100 | ✅ Hızlı | ✅ Hızlı | ❌ **Hiç** |
| 1 | 2 | %50 | ✅ Hızlı | Normal | ✅ 1 disk |
| 5 | 3 | (n-1)/n | ✅ Hızlı | ❌ **4× ceza** | ✅ 1 disk |
| 10 | 4 | %50 | ✅ Hızlı | ✅ Hızlı | ✅ Her mirror'dan 1 |

> **EBS üzerinde RAID genelde yanlıştır** — EBS zaten çoğaltılmıştır. RAID 0 sadece
> instance tavanına kadar IOPS toplamak için anlamlı olabilir *(Faz 3.5.3)*.

---

## A.5 Bus ve PCIe referansı

*(Faz 4.2)*

| Nesil | Şerit başına | x4 | x8 | **x16** |
|---|---|---|---|---|
| Gen3 | 0,985 GB/s | 3,9 GB/s | 7,9 GB/s | **15,8 GB/s** |
| Gen4 | 1,969 GB/s | 7,9 GB/s | 15,8 GB/s | **31,5 GB/s** |
| Gen5 | 3,938 GB/s | 15,8 GB/s | 31,5 GB/s | **63 GB/s** |
| Gen6 | 7,563 GB/s | 30,3 GB/s | 60,5 GB/s | **121 GB/s** |

**Cihaz ihtiyaçları:**

| Cihaz | Gereken |
|---|---|
| NVMe SSD | Gen3/Gen4 x4 |
| 10 Gbps NIC | Gen3 x4 |
| **25 Gbps NIC** | **Gen3 x4 (3,9 ≥ 3,1 GB/s); boşluk payı için veya çift portlu kartta x8** |
| **100 Gbps NIC** | **Gen4 x8 veya Gen3 x16** |
| GPU | Gen4/Gen5 x16 |

```bash
lspci -vv | grep -E "LnkCap|LnkSta"   # yuvanın kapasitesi ve gerçek durumu
```

> **GPU bant genişliği karşılaştırması** *(Faz 4.2.4)*:
> ```
> GPU HBM belleği : ~2000–3000 GB/s
> PCIe Gen4 x16   :        ~32 GB/s
> Fark            :         ~70 kat
> ```
> **GPU'nun kendi belleği hızlıdır; ona veri götürmek yavaştır.**

---

## A.6 Ağ referansı

### Ethernet *(Faz 5.1.2, 5.2.3)*

| Standart | Gbps | GB/s |
|---|---|---|
| 1 GbE | 1 | 0,125 |
| 10 GbE | 10 | 1,25 |
| 25 GbE | 25 | 3,125 |
| 100 GbE | 100 | 12,5 |
| 400 GbE | 400 | 50 |

**Çerçeve ek yükü:**
```
Preamble + SFD        :  8 byte   (MTU dışında, hatta)
Ethernet başlık + CRC : 18 byte   (MTU dışında)
Çerçeveler arası boşluk: 12 byte   (MTU dışında)
IP başlık             : 20 byte   (MTU içinde)
TCP başlık            : 20 byte   (MTU içinde)
────────────────────────────────
Toplam                : 78 byte

MTU 1500 → verim %94,9  (1460 / 1538)
MTU 9000 → verim %99,1  (8960 / 9038)  + paket sayısı 6 kat az
```

### Gecikmenin dört bileşeni *(Faz 5.3.2)*

| Bileşen | Neye bağlı | Müdahale |
|---|---|---|
| **Yayılım** | Mesafe (ışık hızı) | ❌ Sadece yaklaş (CDN) |
| **İletim** | Paket ÷ hat hızı | ✅ Hızlı hat |
| **İşleme** | Cihaz sayısı | ⚠️ Az hop |
| **Kuyruk** | Yük | ✅ **En müdahale edilebilir** |

### Gerçek RTT değerleri *(Faz 5.3.4)*

| Yol | RTT |
|---|---|
| Loopback | ~0,02 ms |
| Aynı rack | ~0,1 ms |
| **Aynı AZ** | **~0,3–0,5 ms** |
| **AZ'ler arası** | **~1–2 ms** |
| Aynı kıtada bölgeler | ~20–40 ms |
| **Kıtalar arası** | **~100–200 ms** |

### BDP *(Faz 5.3.3)*

```
BDP = Bant genişliği × RTT
Maksimum throughput = Pencere boyutu ÷ RTT
```

| Senaryo | Hesap | Sonuç |
|---|---|---|
| 10 Gbps, 100 ms RTT | 1,25 GB/s × 0,1 s | **BDP = 125 MB** |
| 64 KB pencere, 100 ms | 65.536 ÷ 0,1 | **655 KB/s = 5,2 Mbps** |
| 64 KB pencere, 0,4 ms | 65.536 ÷ 0,0004 | 164 MB/s |

> **Aynı pencere, aynı hat — sadece RTT farkı 250 kat throughput farkı yaratıyor.**

### Paket hızı *(Faz 4.4.2, Cevap 6.2)*

```
pps = Bant genişliği ÷ (Paket boyutu × 8)

10 Gbps, 1500 byte  →  833.000 pps
100 Gbps, 1500 byte →  8.333.000 pps
```

---

## A.7 Sanallaştırma referansı

### VM exit maliyeti *(Faz 6.3.4)*

| Dönem | Sanallaştırma vergisi |
|---|---|
| İkili çeviri (2000'ler) | %30–50 |
| VT-x + EPT (2010'lar) | %5–15 |
| **Nitro / SR-IOV** | **<%1** |

```
Tek VM exit: 1.000–5.000 çevrim (~0,3–1,7 μs)
```

### I/O sanallaştırma kuşakları *(Faz 6.6)*

| Yöntem | VM exit | Performans | Canlı göç |
|---|---|---|---|
| Emülasyon | Çok yüksek | %20–40 | ✅ |
| virtio | Orta | %70–90 | ✅ |
| **SR-IOV** | **~Sıfır** | **%95–99** | ❌ |

### Steal time yorumu *(Faz 6.5.2)*

| Değer | Anlamı | Eylem |
|---|---|---|
| %0–2 | Normal | — |
| %2–10 | Hafif yarışma | İzle |
| **%10–25** | **Ciddi** | Yeniden başlat / tip değiştir |
| **%25+** | **Kabul edilemez** | Acil taşı |

> **İki sebebi ayırt et:** host yarışması (m/c/r ailesi) vs **kredi tükenmesi** (t ailesi).
> İkincisinde yeniden başlatmak işe yaramaz.

### Container vs VM *(Faz 6.7.3)*

| | VM | Container |
|---|---|---|
| Başlangıç | 30–60 s | **50–500 ms** |
| Bellek ek yükü | 512 MB – 2 GB | **1–10 MB** |
| Yoğunluk (64 GB host) | ~30 | **~1000+** |
| **İzolasyon sınırı** | **Hypervisor (~100K satır)** | Çekirdek (~30M satır) |

---

## A.8 AWS instance aileleri

*(Faz 7.1)*

### İsim çözümü
```
m 7 g d . 2xlarge
│ │ │ │      └─ 8 vCPU
│ │ │ └──────── d = yerel NVMe
│ │ └────────── g = Graviton (i=Intel, a=AMD)
│ └──────────── 7. nesil
└────────────── m = genel amaçlı
```

### Aileler

| Aile | vCPU:RAM | Fiziksel öne çıkan | Ne için |
|---|---|---|---|
| **m** | 1:4 | Dengeli | **Bilmiyorsan buradan başla** |
| **c** | 1:2 | **Yüksek clock + vCPU başına çok L3** | CPU-bound, tek thread |
| **r** | 1:8 | Bellek kapasitesi + kanal sayısı | Önbellek, analitik |
| **x** | 1:16 | Çok büyük RAM | In-memory veritabanı |
| **i** | — | **NVMe instance store** | Yüksek IOPS NoSQL |
| **d** | — | Yüksek kapasite yerel disk | Veri ambarı |
| **p / g** | — | GPU | Eğitim / çıkarım |
| **t** | değişken | **Kredi tabanlı** | ⚠️ Sadece aralıklı yük |

### vCPU eşleniği *(Faz 1.5.3, 7.1.6)*

| İşlemci | 1 vCPU = |
|---|---|
| Intel / AMD (x86) | **1 SMT iş parçacığı (çekirdeğin yarısı)** |
| **Graviton (ARM)** | **1 fiziksel çekirdek** |

### t ailesi taban performansı *(Faz 6.5.3)*

| Instance | vCPU | Taban |
|---|---|---|
| t3.micro | 2 | %10 |
| t3.small | 2 | %20 |
| t3.medium | 2 | %20 |
| t3.large | 2 | %30 |
| t3.xlarge | 4 | %40 |
| t3.2xlarge | 8 | %40 |

> **Kural: ortalama CPU kullanımın tabandan belirgin düşükse t doğrudur.** Değilse
> t ailesi ucuz değil, bozuktur.

### Ağ performansı *(Faz 5.2.3)*

| Boyut | Ağ |
|---|---|
| `.large` | "Up to 10 Gigabit" — **burst, garanti değil** |
| `.4xlarge` | "Up to 25 Gigabit" |
| `.12xlarge` | 25 Gbps **garanti** |
| `.24xlarge` / `.metal` | 50–100 Gbps garanti |

---

## A.9 EBS tipleri

*(Faz 7.3)*

| Tip | Fiziksel | IOPS | Throughput | Gecikme | Ne için |
|---|---|---|---|---|---|
| **gp3** | SSD | 3.000–**16.000** | 125–1.000 MB/s | ~1 ms | **Varsayılan** |
| gp2 | SSD | boyut×3 (maks 16.000) | boyutla bağlı | ~1 ms | ⚠️ **gp3'e geç** |
| **io2 / BX** | SSD | **256.000'e kadar** | 4.000 MB/s | **<1 ms** | Kritik DB |
| st1 | **HDD** | ~500 burst | 500 MB/s | Yüksek | **Sıralı** |
| sc1 | **HDD** | ~250 | 250 MB/s | Yüksek | Arşiv |

### Karar ağacı
```
Sıralı erişim?        → st1 / sc1
IOPS > 16.000?        → io2 Block Express
Gecikme < 1 ms şart?  → io2
Kalıcılık gerekmiyor? → instance store (i ailesi)
Diğer her şey         → gp3     (~%80 vaka)
```

> **İki tavanı da kontrol et:** volume limiti **ve** instance EBS bant genişliği limiti
> *(Faz 7.3.5)*.
> ```bash
> aws ec2 describe-instance-types --instance-types <tip> --query 'InstanceTypes[0].EbsInfo'
> ```

---

## A.10 Komut referansı

### CPU
```bash
lscpu                      # CPU modeli, çekirdek, cache, NUMA
top                        # us/sy/wa/st — st = steal time
top -H                     # thread bazında (tek thread darboğazı için)
mpstat -P ALL 1            # çekirdek bazında, %soft = ağ işleme
vmstat 1                   # r (run queue), st, si/so
uptime                     # load average
perf stat -e cache-misses,instructions,cycles <komut>
```

### Bellek
```bash
free -h                    # available'a bak, free'ye değil
vmstat 1                   # si/so = swap aktivitesi → alarm
cat /proc/meminfo
numactl --hardware         # NUMA düğümleri ve mesafeler
numastat -m
dmesg | grep -i "out of memory"
```

### Depolama
```bash
iostat -x 1                # await, aqu-sz, r/s, w/s
iotop -o                   # hangi process I/O yapıyor
lsblk                      # blok cihaz ağacı
df -h                      # dosya sistemi doluluk
nvme list                  # NVMe cihazları
smartctl -a /dev/sda       # disk sağlığı
```

### Ağ
```bash
ip a; ip r                 # adres ve yönlendirme
ss -s                      # soket özeti
ss -i                      # cwnd, rtt (BDP analizi)
ethtool eth0               # hız, duplex
ethtool -S eth0            # drop/error sayaçları
ethtool -g eth0            # halka tamponu
ethtool -l eth0            # kuyruk (kanal) sayısı
ethtool -k eth0            # offload durumu
sar -n DEV 1               # arayüz throughput
cat /proc/interrupts       # kesme dağılımı
ping -M do -s 1472 <hedef> # MTU testi
iperf3 -c <hedef> -P 10    # paralel akış testi
```

### Sanallaştırma / container
```bash
systemd-detect-virt        # hangi hypervisor
lscpu | grep -i hypervisor
cat /sys/fs/cgroup/memory.max      # container gerçek bellek limiti
cat /sys/fs/cgroup/memory.current
cat /sys/fs/cgroup/cpu.max
```

### Bus / cihaz
```bash
lspci                      # PCI cihazları
lspci -vv | grep -E "LnkCap|LnkSta"   # PCIe şerit ve nesil
lsusb
dmidecode -t memory        # RAM modülleri, kanal, hız
```

---

## A.11 Teşhis akış şeması

*(Faz 7.5.2)* — **"Uygulamam yavaş" duyduğunda**

```
ADIM 0 — SORULAR (2 dk)
  Ne zamandan beri? / p50 mi p99 mu? / Ne kadar? / Sürekli mi?

ADIM 1 — CPU (3 dk)          top, mpstat -P ALL 1
  us yüksek      → CPU BOUND
  sy yüksek      → çekirdek: syscall, kesme, bağlam değiştirme
  wa yüksek      → I/O BOUND
  st yüksek      → KOMŞU veya KREDİ
  hepsi düşük    → ↓ devam

ADIM 2 — BELLEK (3 dk)       free -h, vmstat 1
  available düşük → MEMORY BOUND (kapasite)
  si/so > 0       → SWAP — acil
  dolu, swap yok  → bant genişliği olabilir

ADIM 3 — DİSK (3 dk)         iostat -x 1
  await + aqu-sz yüksek     → I/O BOUND
  r/s+w/s ≈ limit           → IOPS tavanı
  kB/s ≈ limit              → throughput tavanı
  %util                     → NVMe'de YANILTICI

ADIM 4 — AĞ (3 dk)           ss -s, ethtool -S, sar -n DEV 1
  rx_dropped ↑    → NIC/CPU
  retransmit ↑    → paket kaybı
  bant tavanda    → NETWORK BOUND
  hat boş, yavaş  → BDP / pencere

ADIM 5 — HİÇBİRİ (6 dk)      → BEKLEME BOUND
  Havuz metrikleri / dağıtık izleme / GC / kilit / aşağı akış
  ⚠️ Donanım almak BU DURUMDA HİÇBİR ŞEYİ DÜZELTMEZ
```

---

## A.12 Formül özeti

| Formül | Nerede | Ne için |
|---|---|---|
| `Ortalama erişim = hit×L1 + ıska×RAM` | 2.3.6 | Cache kazancı |
| `Bant genişliği = kanal × kanal_hızı` | 2.4.3 | Bellek kapasitesi |
| `TLB erişim alanı = giriş × sayfa_boyutu` | 2.5.4 | Huge page kararı |
| `HDD gecikme = seek + rotational + transfer` | 3.1.2 | HDD IOPS tavanı |
| `Rotational = 60000 / RPM / 2` (ms) | 3.1.2 | 7200 RPM → 4,17 ms |
| **`Throughput = IOPS × I/O boyutu`** | **3.4.1** | **Hangi limit darboğaz** |
| `await ≈ aqu-sz ÷ IOPS` (Little) | Cevap 3.3 | `iostat` tutarlılık kontrolü |
| `PCIe BW = şerit × nesil_hızı` | 4.2.2 | Yuva yeterli mi |
| `pps = Gbps ÷ (paket_byte × 8)` | 4.4.2 | Kesme yükü |
| `İletim gecikmesi = paket_bit ÷ hat_bps` | 5.3.2 | Gecikme bileşeni |
| **`BDP = bant_genişliği × RTT`** | **5.3.3** | **TCP pencere ihtiyacı** |
| `Maks throughput = pencere ÷ RTT` | 5.3.3 | "Hızlı hat, yavaş transfer" |
| `VM exit yükü = pps × exit_maliyeti ÷ clock` | 6.3.4 | SR-IOV gerekçesi |
| `Gereken vCPU = istek/s × vCPU-s/istek ÷ hedef_kullanım` | 7.6 | Kapasite planı |

---

*Ek A sonu.* → **[Ek B — Terim Sözlüğü](EK_B_Terim_Sozlugu.md)** · **[README](README.md)**

---

> **Navigasyon:** [◀ README](README.md) · **Ek A** · [Ek B — Terim Sözlüğü ▶](EK_B_Terim_Sozlugu.md)
