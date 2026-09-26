# Ek B — Terim Sözlüğü

> **Navigasyon:** [◀ Ek A — Referans Tabloları](EK_A_Referans_Tablolari.md) · **Ek B** · [README ▶](README.md)

---

> **Nasıl kullanılır:** Her terimin yanında **hangi bölümde işlendiği** yazılıdır.
> Tanım hatırlatmak içindir; anlamadığın bir terimi burada okuyup geçme — **bölüme dön.**
>
> Terimler Türkçe alfabetik sıradadır. İngilizce karşılıklar parantez içindedir.

---

## A

**Adres bus** *(4.1.1)* — CPU'nun hangi bellek adresine erişmek istediğini taşıyan hat
grubu. Genişliği adreslenebilir bellek miktarını belirler.

**Anahtarlama (switching)** *(5.2.1)* — Veriyi paylaşımlı bir ortama yayınlamak yerine
yalnızca hedefe yönlendirme. Hub→switch, paylaşımlı bus→PCIe geçişlerinin ortak fikri.

**Associativity (küme ilişkilendirme)** *(2.3.7)* — Bir cache satırının kaç farklı yere
yerleşebileceği. Tam ilişkili, doğrudan eşlemeli ve n-yollu küme ilişkilendirmeli
varyantları vardır; çatışma ıskalarını etkiler.

**Asenkron** — İşlemi başlatıp sonucunu beklemeden devam etme. DMA'nın *(4.3)* ve NVMe
kuyruklarının *(3.3.2)* temel çalışma biçimi.

**await** *(3.4.4)* — `iostat -x` çıktısında bir I/O isteğinin **kuyrukta bekleme +
hizmet** toplam süresi. Yüksekse disk doygundur veya gecikme problemi vardır.

**AZ (Availability Zone)** *(5.3.4)* — Bir AWS bölgesi içindeki fiziksel olarak ayrı veri
merkezi. AZ'ler arası RTT ~1–2 ms.

## B

**Balloon driver** *(6.4.4)* — Misafir işletim sisteminin içine yerleştirilen, bellek
ayırarak "şişen" ve böylece misafiri sayfa bırakmaya zorlayan sürücü. Hypervisor'ün
misafirden dolaylı yoldan bellek geri alma yöntemi.

**Bandwidth (bant genişliği)** *(5.3.1)* — Birim zamanda taşınan veri miktarı. **Gecikmeden
bağımsızdır ve artırılabilir.**

**Bare metal** *(6.6.4, 7.2.3)* — Hypervisor olmadan, fiziksel sunucunun tamamının
müşteriye verildiği instance tipi (`.metal`).

**BDP (Bandwidth-Delay Product)** *(5.3.3)* — `Bant genişliği × RTT`. Hatta aynı anda
"uçmakta olan" veri miktarı. TCP penceresi bundan küçükse hat asla dolmaz.

**Bekleme bound** *(7.5.7)* — Hiçbir donanım kaynağı doygun değilken uygulamanın yavaş
olması. Kilit, uzak çağrı, havuz tükenmesi, GC duraklaması. **Donanım almak bu durumda
hiçbir şeyi düzeltmez.**

**Bit** *(0.1.2)* — İkili sistemin en küçük birimi: 0 veya 1.

**Blok (NAND)** *(3.2.2)* — SSD'de **silinebilen** en küçük birim (~1–4 MB). Yazma sayfa
(4–16 KB) biriminde yapılır — bu asimetri write amplification'ın kaynağıdır.

**Burst** *(5.2.3, 6.5.3)* — Kısa süreli, taban performansın üzerinde çalışabilme. Kredi
tabanlıdır ve tükenir. "Up to X" ifadeleri bunu anlatır.

**Bus** *(4.1.1)* — Bileşenler arası veri taşıyan ortak hat grubu. Adres, veri ve kontrol
bus'ı olmak üzere üçe ayrılır.

## C

**Cache** *(2.3)* — CPU ile RAM arasındaki küçük ve hızlı bellek. Programların
**yerellik** göstereceği bahsi üzerine kuruludur.

**Cache kirlenmesi (cache pollution)** *(2.3.4, 7.4.3)* — Bir iş yükünün, başka bir iş
yükünün sıcak verisini cache'ten atması. **Komşu gürültüsünün en sinsi mekanizması** —
hiçbir standart metrikte görünmez.

**Cache satırı (cache line)** *(2.3.5)* — Cache'in transfer birimi, **64 byte.** Tek bir
byte istesen de 64 byte okunur. Erişim deseni optimizasyonunun temel birimi.

**Cache tutarlılığı (coherence)** *(2.3.8)* — Birden çok çekirdeğin cache'lerindeki aynı
verinin tutarlı kalmasını sağlayan protokol (MESI).

**CAS latency** *(2.4.2)* — DRAM'de sütun adresi verildikten sonra verinin gelmesine kadar
geçen çevrim sayısı.

**cgroup** *(6.7.2)* — Linux çekirdeğinin kaynak sınırlama mekanizması: CPU, bellek, I/O,
process sayısı. Container'ın "ne kadar kullanabilirsin" yarısı.

**Checkpoint** *(7.6.4)* — Uzun süren bir işin ara durumunu kaydetmesi. Spot instance
kullanımının ön şartı.

**Chipset** *(4.5)* — Anakarttaki CPU dışı bileşenleri birbirine bağlayan yonga seti.
Northbridge'in CPU'ya entegre edilmesi **NUMA'yı doğurmuştur** *(4.5.2)*.

**Clock (saat frekansı)** *(1.3.1)* — CPU'nun saniyedeki çevrim sayısı (GHz). Tek başına
performans göstergesi değildir — **IPC ile çarpılmalıdır.**

**Container** *(6.7)* — Namespace ve cgroup ile izole edilmiş process. **Hafif VM
değildir** — host çekirdeğini paylaşır.

**CRC** *(5.1.2)* — Ethernet çerçevesindeki hata tespit kodu. Bozuk çerçeve sessizce
atılır.

**Çevrim (cycle)** *(1.3.1)* — CPU saatinin bir vuruşu. 3 GHz'de 0,33 nanosaniye.

**Çıkarım (inference)** *(7.1.5)* — Eğitilmiş bir ML modelini çalıştırma. Eğitimden farklı
donanım profili ister (`g`, `inf` aileleri).

## D

**Dal tahmini (branch prediction)** *(1.4.2)* — Pipeline'ın dolu kalması için koşullu
dalların sonucunu önceden tahmin etme. Yanlış tahmin pipeline'ı boşaltır — Spectre'nin
temeli.

**Dedicated Host** *(7.4.6)* — Belirli bir fiziksel sunucunun müşteriye ayrılması.
Yerleşim görünürlüğü sağlar; fiziksel çekirdeğe bağlı lisanslar için gereklidir.

**DIMM** *(2.4.2)* — RAM modülünün fiziksel formu. Rank → Bank → Row → Column hiyerarşisi
içerir.

**DMA (Direct Memory Access)** *(4.3)* — Bir cihazın CPU'ya uğramadan doğrudan RAM'e
okuma/yazma yapması. Yüksek hızlı I/O'nun ön şartı.

**DRAM** *(0.4.4, 2.4.1)* — Kondansatörde şarj olarak veri tutan, **periyodik tazeleme
(refresh) gerektiren** bellek. Ucuz ve yoğun ama SRAM'den yavaş.

**Duplex** *(5.2.2)* — Half duplex: sırayla gönder/al. Full duplex: **aynı anda** — modern
standart.

## E

**EBS** *(3.3.4, 7.3)* — AWS'in kalıcı blok depolama servisi. **Ağ üzerinden erişilir** —
bu, gecikmesinin ve IOPS kotasının açıklamasıdır.

**ECC** *(2.4.4)* — Bellekte tek bit hatalarını düzelten, çift bit hatalarını tespit eden
kodlama. Sessiz veri bozulmasına karşı koruma.

**EFA (Elastic Fabric Adapter)** *(5.4.3)* — AWS'in RDMA benzeri düşük gecikmeli ağ
arayüzü. Çok düğümlü ML eğitimi ve HPC için belirleyici.

**Eğitim (training)** *(7.1.5)* — ML modelini veriden öğretme. Yüksek GPU ve düğümler
arası ağ ihtiyacı (`p`, `trn` aileleri).

**ENA (Elastic Network Adapter)** *(6.6.4)* — AWS'in SR-IOV tabanlı yüksek performanslı ağ
arayüzü ("enhanced networking").

**EPT / NPT** *(6.4.3)* — Intel/AMD'nin ikinci adres çevirisini donanıma alan teknolojisi.
Gölge sayfa tablolarının VM exit maliyetini ortadan kaldırır; sayfa tablosu yürüyüşünü
uzatır.

**Ethernet çerçevesi** *(5.1.2)* — Fiziksel ağ katmanının veri birimi. MAC adresleri, tip
alanı, payload (maks 1500 byte) ve CRC içerir.

## F

**False sharing** *(2.3.8)* — Farklı çekirdeklerin, aynı cache satırındaki **farklı**
değişkenlere yazması. Mantıksal paylaşım yok ama donanım seviyesinde satır ping-pong
yapar. Padding ile çözülür.

**Flip-flop** *(0.4.2)* — Bir bit saklayan temel ardışıl devre. Register'ların yapı taşı.

**FTL (Flash Translation Layer)** *(3.2.3)* — SSD'nin mantıksal adresleri fiziksel NAND
konumlarına eşleyen katmanı. **Sanal belleğin cihaz içindeki karşılığı.**

## G

**Garbage collection (SSD)** *(3.2.3)* — SSD'de geçersiz sayfaları toplayıp blokları
yeniden kullanılabilir hâle getirme. Write amplification'ın kaynağı.

**Gecikme (latency)** *(5.3.1)* — Tek bir isteğin tamamlanma süresi. **Bant genişliğinden
bağımsızdır ve ışık hızıyla sınırlıdır.**

**Gölge sayfa tablosu (shadow page table)** *(6.4.2)* — EPT öncesi, hypervisor'ün misafir
sanal adresinden gerçek fiziksel adrese giden birleşik tablo tutması. Her sayfa tablosu
değişikliğinde VM exit gerektirir.

**Graviton** *(1.6.4, 7.1.6)* — AWS'in ARM64 tabanlı işlemcisi. **1 vCPU = 1 fiziksel
çekirdek** (x86'da yarım çekirdektir).

**gp3** *(7.3.2)* — AWS EBS SSD tipi. IOPS ve throughput **disk boyutundan bağımsız**
ayarlanır — gp2'ye göre temel avantajı budur.

**GRO / LRO** *(5.1.4)* — Gelen küçük paketleri birleştirip üst katmana daha az iş bırakan
boşaltma (offload) özelliği.

## H

**Halka tamponu (ring buffer)** *(5.1.3)* — NIC ile çekirdek arasındaki dairesel
tanımlayıcı dizisi. Dolarsa paket **sessizce düşer** (`rx_dropped`).

**HBM (High Bandwidth Memory)** *(4.2.4)* — GPU'larda kullanılan çok yüksek bant genişlikli
bellek (~2000–3000 GB/s). PCIe'den ~70 kat hızlı — GPU darboğazının kaynağı.

**HDD** *(3.1)* — Mekanik disk. Rastgele IOPS'i (80–120) **30 yıldır artmamıştır** çünkü
fiziksel kafa hareketiyle sınırlıdır.

**Hit oranı (hit rate)** *(2.3.6)* — İsteğin cache'te bulunma yüzdesi. %90→%99 geçişi
ortalama erişimi ~4 kat hızlandırır.

**Huge page** *(2.5.4)* — 2 MiB veya 1 GiB boyutlu bellek sayfası. TLB erişim alanını
512 kat genişletir; **sanallaştırmada daha da kritiktir** *(6.4.3)*.

**Hypervisor** *(6.2)* — Sanal makineleri yöneten yazılım katmanı. Type 1 doğrudan donanım
üzerinde, Type 2 bir host OS üzerinde çalışır.

## I

**Instance store** *(7.1.4)* — Fiziksel sunucuya doğrudan bağlı yerel NVMe. Çok hızlı ama
**instance durunca kaybolur.**

**Interrupt (kesme)** *(4.4.1)* — Bir cihazın CPU'nun dikkatini çekmek için gönderdiği
sinyal. Maliyeti pipeline boşaltma ve cache kirlenmesini içerir.

**IOMMU (VT-d / AMD-Vi)** *(Faz 4 S4, 6.6.3)* — Cihazlar için MMU: DMA erişimlerini çevirir
ve sınırlar. **SR-IOV'un varlık şartıdır** — olmadan izolasyon tamamen kalkar.

**IOPS** *(3.4.1)* — Saniyedeki I/O işlemi sayısı. `Throughput = IOPS × I/O boyutu`.

**IPC (Instructions Per Cycle)** *(1.3.2)* — Çevrim başına tamamlanan komut sayısı.
`Performans = Clock × IPC`. Cache kirlenmesinde **düşen şey budur** ve standart
metriklerde görünmez.

**ISA (Instruction Set Architecture)** *(1.6)* — CPU'nun anladığı komut kümesi (x86-64,
ARM64). Farklı ISA = farklı derleme gerekir.

## J

**Jitter** *(5.3.2)* — Gecikmedeki dalgalanma. Kuyruk gecikmesinin imzasıdır; hat
doygunluğa yaklaştığında ortaya çıkar.

**Jumbo frame** *(5.1.2)* — MTU 9000 byte'lık Ethernet çerçevesi. Verimi ve **paket
sayısını** iyileştirir; yol üzerindeki **her** cihaz desteklemelidir.

## K

**Kanal (memory channel)** *(2.4.3)* — CPU ile RAM arasındaki bağımsız veri yolu.
`Bant genişliği = kanal sayısı × kanal hızı`.

**Komşu gürültüsü (noisy neighbor)** *(6.1.2, 7.4)* — Aynı fiziksel donanımı paylaşan
başka kiracının performansını etkilemesi. Üç mekanizması vardır: CPU zamanı (ölçülebilir),
**L3 cache** ve **bellek bant genişliği** (ölçülemez).

**Kredi (CPU credit)** *(6.5.3)* — t ailesinde biriken ve burst sırasında harcanan
performans birimi. Bitince taban performansa düşülür.

**KSM (Kernel Same-page Merging)** *(6.4.4)* — Aynı içerikli bellek sayfalarını tek fiziksel
sayfada birleştirme. Bellek tasarrufu sağlar, yan kanal riski taşır.

**Kuyruk derinliği (queue depth)** *(3.2.5, 3.3.2)* — Aynı anda cihaza gönderilen istek sayısı.
QD=1 ile QD=32 arasında NVMe'de kat kat fark vardır.

**Kuyruk eğrisi** *(3.4.4)* — Doluluk arttıkça gecikmenin **üstel** artması. %95 doluluk,
%50'nin ~8 katı gecikme. **Bu haritada üç ayrı katmanda tekrar eder.**

**KVM** *(6.2.3)* — Linux çekirdeğini Type 1 hypervisor'e dönüştüren modül. AWS Nitro
hypervisor'ünün temeli.

## L

**L1 / L2 / L3** *(2.3.2–2.3.4)* — Cache seviyeleri. L1 ve L2 çekirdeğe özel, **L3 tüm
çekirdekler tarafından paylaşılır** — komşu gürültüsü buradan doğar.

**Little Yasası** *(Cevap 3.3, Cevap 5.1)* — `Bekleme süresi = Kuyruk uzunluğu ÷ Hizmet hızı`. `iostat`
çıktısında `await ≈ aqu-sz ÷ IOPS` tutarlılık kontrolü için kullanılır.

## M

**MAC adresi** *(5.1.2)* — Ağ arayüzünün 48 bitlik donanım adresi. Yerel ağda adresleme
için kullanılır.

**Mantık kapısı** *(0.3)* — AND, OR, NOT gibi temel boole işlemini yapan devre. Tüm
hesaplamanın yapı taşı.

**Memory wall** *(2.1.2)* — CPU hızının bellek hızından çok daha hızlı artması sonucu
ortaya çıkan performans uçurumu. Cache hiyerarşisinin varlık sebebi.

**MESI** *(2.3.8)* — Cache tutarlılık protokolü: Modified, Exclusive, Shared, Invalid.

**MMU (Memory Management Unit)** *(2.5.2)* — Sanal adresleri fiziksel adreslere çeviren
donanım birimi.

**MSI-X** *(4.4.3)* — Mesaj tabanlı kesme mekanizması. Kesmelerin farklı çekirdeklere
dağıtılmasını sağlar — RSS'in donanım temeli.

**MTU (Maximum Transmission Unit)** *(5.1.2)* — Bir çerçevenin taşıyabileceği maksimum
payload. Standart 1500 byte. Uyuşmazlığı "bazı bağlantılar çalışıyor, bazıları takılıyor"
belirtisi verir.

## N

**Namespace** *(6.7.2)* — Linux çekirdeğinin görünürlük izolasyon mekanizması (PID, ağ,
mount, hostname). Container'ın "ne görüyorsun" yarısı.

**NAND** *(3.2.1)* — SSD'lerde kullanılan flash bellek teknolojisi. Floating gate'te
elektron tutarak veri saklar.

**NAPI** *(4.4.2, 5.1.4)* — Düşük trafikte kesme, yüksek trafikte yoklama (polling)
kullanan hibrit ağ işleme mekanizması. Kesme fırtınasını önler.

**NIC** *(5.1)* — Ağ arayüz kartı. Bellekteki bit'ler ile kablodaki sinyaller arasında
çeviri yapar.

**Nitro** *(4.3.3, 6.6.4, 7.2)* — AWS'in hypervisor işlevlerini ayrı fiziksel kartlara
taşıyan mimarisi. Sanallaştırma vergisini %1'in altına indirdi; güvenlik sınırını
**yazılımdan donanıma** taşıdı.

**NUMA (Non-Uniform Memory Access)** *(2.7)* — Her CPU soketinin kendi yerel belleğinin
olduğu mimari. Uzak düğüm erişimi 1,5–2 kat yavaştır. **Bellek denetleyicisinin CPU'ya
entegre edilmesinin sonucudur** *(4.5.2)*.

**NVMe** *(3.3.2)* — SSD'ler için tasarlanmış protokol. **65.535 kuyruk** (SATA'da 1),
her biri 65.536 derinlikte.

## O

**OOM Killer** *(2.6.3)* — Bellek tükendiğinde Linux çekirdeğinin bir process'i sonlandırma
mekanizması. Container'larda cgroup limiti aşıldığında da devreye girer.

**Overcommit** *(6.4.4)* — Fiziksel kaynaktan fazla taahhüt etme. **AWS EC2'de bellek
overcommit yapılmaz** — öngörülebilirlik için.

## Ö

**Ön getirme (prefetching)** *(2.3.9)* — CPU'nun gelecekte gerekecek veriyi önceden
cache'e çekmesi. Düzenli erişim desenlerinde çalışır; pointer chasing'de çalışmaz.

## P

**Paket kaybı** *(5.3.2)* — Kuyruk dolduğunda paketlerin düşmesi. TCP bunu tıkanıklık
sinyali olarak kullanır.

**Paravirtualization** *(6.3.2, 6.6.2)* — Misafir OS'in sanal olduğunu bilerek hypervisor
ile verimli protokol kullanması. virtio bunun standardıdır.

**PCIe** *(4.2)* — Modern nokta-nokta genişletme bus'ı. Şerit (x1–x16) ve nesil
(Gen3–Gen6) ile ölçeklenir.

**Pipeline** *(1.4)* — Komut işlemeyi aşamalara bölerek paralelleştirme. Hazard'lar
(veri, kontrol, yapısal) verimi düşürür.

**pps (paket/saniye)** *(4.4.2)* — Saniyede işlenen paket sayısı. Küçük paketli
iş yüklerinde bant genişliğinden önce bu limit dolar.

**Propagation delay (yayılım gecikmesi)** *(5.3.2)* — Sinyalin mesafeyi kat etme süresi.
**Işık hızıyla sınırlıdır — hiçbir teknoloji düşüremez.**

## R

**RAID** *(3.5)* — Birden çok diski tek mantıksal birim olarak kullanma. EBS üzerinde
genellikle gereksizdir (EBS zaten çoğaltılmıştır).

**RDMA** *(5.4.2)* — Uzak makinenin belleğine, onun CPU'su devreye girmeden doğrudan
erişim. DMA'nın ağ üzerinden uzatılmış hâli.

**Refresh (tazeleme)** *(2.4.1)* — DRAM hücrelerinin şarjını periyodik olarak yenileme.
DRAM'i SRAM'den ayıran temel özellik.

**Register** *(2.2)* — CPU içindeki en hızlı saklama birimi. Sıfır çevrim gecikme.

**Right-sizing** *(7.6.2)* — Gerçek kullanıma göre instance boyutu ayarlama. Hedef: CPU
ortalama %40–60.

**Ring seviyesi** *(6.3.1)* — x86'daki ayrıcalık katmanı. Ring 0 çekirdek, ring 3
kullanıcı.

**RSS (Receive Side Scaling)** *(5.1.4)* — NIC'in gelen paketleri başlık hash'ine göre
farklı kuyruklara ve çekirdeklere dağıtması. **Hash kullanılır çünkü aynı bağlantı aynı
çekirdekte kalmalıdır** (sıralama + cache yerelliği).

**RTT (Round-Trip Time)** *(5.3.4)* — Gidiş-dönüş süresi. Aynı AZ ~0,4 ms, kıtalar arası
~100–200 ms.

## S

**Sanal bellek** *(2.5)* — Her process'e kendi sürekli adres alanını verme mekanizması.
Üç problemi çözer: izolasyon, parçalanma, fiziksel RAM'den büyük adres alanı.

**Sayfa (page)** *(2.5.2)* — Sanal belleğin transfer ve eşleme birimi. Standart 4 KiB.

**Sayfa hatası (page fault)** *(2.5.5)* — Erişilen sayfanın fiziksel bellekte olmaması.
Minor (bellekte ama eşlenmemiş), major (diskten okunmalı), invalid (geçersiz — segfault).

**SMT / Hyper-Threading** *(1.5.2)* — Bir fiziksel çekirdeğin iki iş parçacığı olarak
görünmesi. **x86 instance'larda 1 vCPU = 1 SMT thread = yarım çekirdek.**

**Spot instance** *(7.6.4)* — %70–90 indirimli ama 2 dakika uyarıyla kesilebilen kapasite.
Durumsuz ve checkpoint'li iş yükleri için uygundur.

**SR-IOV** *(6.6.3)* — Fiziksel cihazın kendini birden çok sanal fonksiyon olarak gösterip
her birini bir VM'e doğrudan atanması. **VM exit'i ortadan kaldırır**; IOMMU gerektirir.

**SRAM** *(0.4.4)* — Transistörlerle veri tutan, tazeleme gerektirmeyen hızlı bellek.
Cache'lerde kullanılır; pahalı ve az yoğundur.

**Steal time** *(6.5.2)* — vCPU'nun çalışmaya hazır olduğu ama fiziksel çekirdek bulamadığı
sürenin yüzdesi. **İki sebebi vardır:** host yarışması ve t-ailesi kredi tükenmesi.

**Swap** *(2.6)* — Bellek sayfalarını diske taşıma. Cloud'da genellikle kapalıdır — **hızlı
başarısız ol, yavaşça ölme** ilkesi.

## T

**TCP penceresi** *(5.3.3)* — Onay beklemeden gönderilebilecek veri miktarı. BDP'den
küçükse hat asla dolmaz.

**Thrashing** *(2.6.2)* — Sistemin çalışmaktan çok sayfa takasıyla uğraşması. Pozitif geri
besleme döngüsüdür — sistem yavaşça değil **uçurumdan** düşer.

**Throughput** *(3.4.1)* — Birim zamanda taşınan veri miktarı. `IOPS × I/O boyutu`.

**TLB (Translation Lookaside Buffer)** *(2.5.3)* — Sayfa tablosu çevirilerinin cache'i.
Erişim alanı `giriş sayısı × sayfa boyutu` ile sınırlıdır — huge page'in gerekçesi.

**TRIM** *(3.2.3)* — İşletim sisteminin SSD'ye "bu sayfalar artık geçersiz" bildirmesi.
Garbage collection verimini artırır.

**TSO / GSO** *(5.1.4)* — Büyük veri bloğunu NIC'in paketlere bölmesi. Paket başına CPU
maliyetini düşürür.

## V

**vCPU** *(1.5.3, 6.5.1)* — Sanal işlemci. x86'da 1 SMT thread, Graviton'da 1 fiziksel
çekirdek. **Ve her durumda, fiziksel çekirdek üzerinde zamanlanan bir iş parçacığıdır** —
sürekli sana ait değildir.

**Veri bus** *(4.1.1)* — Gerçek veriyi taşıyan hat grubu.

**virtio** *(6.6.2)* — Paravirtualize I/O standardı. Paylaşımlı halka tamponuyla toplu
bildirim yaparak VM exit sayısını düşürür.

**VM exit** *(6.3.4)* — Misafirin ayrıcalıklı bir işlemi üzerine kontrolün hypervisor'e
geçmesi. **Sanallaştırmanın temel maliyet birimi** (1.000–5.000 çevrim). Tüm optimizasyon
bunu azaltmaya yöneliktir.

**VMX root / non-root** *(6.3.3)* — VT-x'in getirdiği, ring seviyelerine dik mod ayrımı.
Misafir ring 0'da çalışır ama non-root moddadır.

**VT-x / AMD-V** *(6.3.3)* — Donanım destekli sanallaştırma teknolojisi. Misafir OS'in
değiştirilmeden çalışmasını sağlar.

## W

**Wear leveling** *(3.2.3)* — SSD'de yazmaları bloklar arasında dengeleyerek ömrü uzatma.

**Write amplification** *(3.2.2)* — Uygulamanın yazdığından daha fazla verinin NAND'a
yazılması. Sayfa/blok asimetrisinden doğar.

## Y

**Yazma cezası (write penalty)** *(3.5.2)* — RAID 5'te tek mantıksal yazmanın 4 fiziksel
işlem gerektirmesi (oku-oku-yaz-yaz).

**Yerellik (locality)** *(2.1.3)* — Programların belleğe erişim deseni. **Zamansal**
(yakın zamanda erişilen tekrar erişilir) ve **mekânsal** (yakın adresler birlikte
erişilir). Cache'in üzerine kurulduğu bahis budur.

**Yoklama (polling)** *(4.4.2)* — Kesme beklemek yerine cihazı düzenli sorgulama. Yüksek
trafikte kesmeden verimlidir.

---

## Ek: Kolayca karıştırılan çiftler

| Terim çifti | Fark | Bölüm |
|---|---|---|
| **Bant genişliği vs Gecikme** | Biri artırılabilir, diğeri ışık hızıyla sınırlı | 5.3.1 |
| **IOPS vs Throughput** | `Throughput = IOPS × I/O boyutu` | 3.4.1 |
| **Çekirdek vs Thread vs vCPU** | Fiziksel / SMT / ayrılan dilim | 1.5 |
| **Steal time vs Cache kirlenmesi** | Biri ölçülebilir, diğeri görünmez | 7.4 |
| **Sanal bellek vs Swap** | Mekanizma vs onun bir kullanımı | 2.5, 2.6 |
| **Container vs VM** | Paylaşılan çekirdek vs ayrı çekirdek | 6.7 |
| **Instance store vs EBS** | PCIe doğrudan vs ağ üzerinden | 7.1.4 |
| **gp3 vs io2** | Genel amaç vs sub-ms gecikme | 7.3 |
| **RSS vs RPS** | Donanımda vs yazılımda dağıtım | 5.1.4, Faz 5 S6 |
| **DMA vs RDMA** | Yerel CPU atlama vs uzak CPU atlama | 4.3, 5.4.2 |
| **Emülasyon vs virtio vs SR-IOV** | Üç I/O sanallaştırma kuşağı | 6.6 |
| **Overcommit vs Right-sizing** | Sağlayıcının fazla satması vs senin doğru almanı | 6.4.4, 7.6.2 |
| **Type 1 vs Type 2 hypervisor** | Donanım üzerinde vs host OS üzerinde | 6.2 |
| **Minor vs Major page fault** | Bellekte ama eşlenmemiş vs diskten okunmalı | 2.5.5 |
| **SRAM vs DRAM** | Transistör, tazeleme yok vs kondansatör, tazeleme var | 0.4.4 |

---

*Ek B sonu.* → **[Ek A — Referans Tabloları](EK_A_Referans_Tablolari.md)** · **[README](README.md)**

---

> **Navigasyon:** [◀ Ek A — Referans Tabloları](EK_A_Referans_Tablolari.md) · **Ek B** · [README ▶](README.md)
