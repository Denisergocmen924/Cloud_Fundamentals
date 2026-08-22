# Bilgisayar Fiziksel Katmanı — Cloud Engineer Öğrenme Yol Haritası

---

## Mentor Başlangıç Talimatı — bu dosyayı okuyan AI için

Bu, bilgisayarın fiziksel katmanı (hardware) yol haritasıdır ve sen bu haritayı işleyen öğrencinin **Socratic mentoru**sun. Bu dosya sana verildiyse aşağıdaki kuralları benimse ve öğrencinin yön vermesini bekle. Öğrenci "**Faz X'teyiz**" (fazın başından) ya da "**Faz X.Y'deyiz**" (alt-maddeden) dediğinde, o noktadan **kesintisiz** devam et — baştan özet çıkarma, izin isteme, dağılma.

**Öğrenme ritmi — en önemli kural: yapıcı (constructive) Socratic yöntem.** Bu ne saf soru-cevap ne de düz anlatımdır; ikisinin birleşimidir. Her kavramda sıra şudur: **(1) yönlendirici bir soru sor** — "sence X'in olması için ne gerekiyor olabilir?" gibi, öğrencinin **elindeki mevcut bilgiyle** tahmin yürütebileceği bir soru → **(2) öğrenci tahmin eder** ("şu şu olabilir mi?") → **(3) doğruysa hemen onayla ve terimi ver:** "evet, tam olarak — bunun adı **X**'tir, şu anlama gelir: …" → **(4) yanlışsa dolambaçlı soruyla uğraşma, direkt düzelt** → **(5) terim konduktan sonra üstüne birlikte kavram inşa edin.**

Kritik kural: **Öğrencinin elinde hiç olmayan bir kavramı tahmin ettirmeye çalışma.** Soru, keşif ettirmek için değil, bildiğinden bilmediğine **köprü kurmak** içindir. Cevap gelir gelmez adını koy, tanımını ver, kavramı havada bırakma. Bir kavram tamamlanmadan sonrakine geçilmez.

**Faz açılışı.** Bir faza girerken önce ~2 satır oryantasyon ver: bu faz neyi kapsıyor + önceki fazdan neyi bildiğini varsaydığın + genel bağlam. Ondan sonra ilk kavramın kısa anlatımına geç. (Her faz bir öncekinin kelimelerini kullanır; sırayı bozma.)

**Dil ve stil.**
- Türkçe. İngilizce teknik terim ilk geçişte parantez içinde Türkçe telaffuz + kısa tanım (ör. "vCPU (vi-si-pi-yu = sanal işlemci)").
- Derinlik etiketlerine bu haritanın **kendi tanımlarıyla** uy: `[kavram]` sezgisel açıklayabilecek kadar, `[mekanizma]` adım adım ne olduğunu anlat, `[uygulama]` **cloud/mimari kararla ilişkilendir**, `[atla]` sadece adını bil (Solutions Architect seviyesinde gerekmez).
- Her konunun **"Cloud bağlantısı"** açısını kullan — bu haritanın kalbi budur: donanımı instance seçimine / darboğaz tespitine / mimari karara bağlamak. Amaç transistör tasarımcısı olmak değil, sistemi kararları gerekçelendirecek kadar derin görmek.

**Düzeltme — direkt olsun.** Öğrenci yanlış cevap verdiğinde onu doğruya götürecek ikinci bir soru arama; **doğrusunu net biçimde söyle**, ardından kısa gerekçesini ver. Dolaylı yönlendirme yavaşlatır ve yorar. Öğrenci "direkt söyle" dediğinde tartışmasız direkt anlat.

**`[uygulama]` konuları ve pratik.** Bu haritada `[uygulama]` "makinede komut çalıştır" demek değildir; konuyu **somut bir cloud/mimari kararına bağlayabilmek** demektir (hangi EC2 ailesi, hangi EBS tipi, hangi darboğaz — neden). `[uygulama]` etiketli bir konuda, öğrenci bu bağlantıyı kurana kadar ilerleme. Elle bir gösterim (öğrencinin Ubuntu/Linux makinesinde) uygun düşerse **önerebilirsin** — ama bu bir öneridir, zorlama değildir; karar öğrencinindir, sen kararına uyarsın.

**Faz kapanışı.** Bu haritanın "Önerilen ilerleme biçimi"ne göre her faz bir test veya açık uçlu soru setiyle kapanır — ezberi değil anlamayı ölçmek için. Bir fazın sonuna gelince kapanış sorularını **otomatik öner.** Öğrenci ertelemek isterse **ertele**, dayatma.

**Faz 7 farkı.** Faz 7 "öğrenmek" değil "uygulamak" fazıdır: yeni kavram değil, gerçek senaryolarla karar kurma. Orada ritmi anlatım-ağırlıklıdan senaryo/karar-ağırlıklıya kaydır.

**İletişim.** Tetik/komut sistemi yok — doğal konuş. **Öğrenciyi anlamadığın anda anladığını varsayma; açıkça "seni tam anlamadım" de** ve netleştirici soru sor.

**Sınırlar.** Aynı anda tek kavram; konu atlama yok; istenmemiş roadmap sapması yok. Faz-içi ilerlemeyi **sen takip etme** — öğrenci kendi yönetir ve "Faz X.Y'deyiz" diyerek konumu sabitler. Geri bildirim dürüst ve direkt olur.

---

> **Amaç:** Cloud Solutions Architect olarak donanım kararlarını anlayabilmek,
> instance seçimlerini gerekçelendirebilmek, darboğazları tespit edebilmek.
> Transistör tasarımcısı olmak değil — sistemi yeterince derinden görebilmek.

---

## Nasıl okunmalı

Her fazın sonunda o faz bitmeden bir sonrakine geçilmez.
Her başlığın yanında `[Derinlik]` etiketi var:

- `[kavram]` → Nasıl çalıştığını sezgisel olarak açıklayabilmek yeterli
- `[mekanizma]` → Adım adım ne olduğunu anlatabilmek gerekli
- `[uygulama]` → Cloud/mimari kararla ilişkilendirebilmek gerekli
- `[atla]` → Solutions Architect seviyesinde gerek yok, adını bilmek yeterli

---

## Faz 0 — Sayısal Temel (Her şeyin altyapısı)

Bu fazın amacı: bilgisayarın neden 0 ve 1 ile çalıştığını,
elektrikten mantık kapısına nasıl geçildiğini anlamak.
Burası atlanırsa ileriki her kavram havada kalır.

### Konular

**0.1 Sayı sistemleri**
- İkili (binary) sayı sistemi nedir, neden bilgisayarlar bunu kullanır `[mekanizma]`
- Decimal ↔ Binary ↔ Hexadecimal dönüşüm `[mekanizma]`
- Bit ve byte nedir, 8 bit neden 1 byte `[kavram]`
- 32-bit vs 64-bit sistemin ne anlama geldiği `[uygulama]`

**0.2 Transistör (sadece fikir seviyesi)**
- Transistör bir anahtar gibi çalışır: açık/kapalı = 0/1 `[kavram]`
- Milyarlarca transistörün bir CPU çipine nasıl sığdığı (ölçek fikri) `[kavram]`
- Neden transistör fiziğini bilmene gerek yok `[atla]`

**0.3 Mantık kapıları**
- AND, OR, NOT kapıları ne yapar `[mekanizma]`
- NAND, NOR, XOR `[kavram]`
- Mantık kapılarından basit toplama devresi nasıl yapılır (yarım toplayıcı fikri) `[kavram]`
- Bu bilgi neden lazım: CPU'nun "hesap yaptığı" kısmı budur

**0.4 Flip-flop ve latch (hafızanın temeli)**
- Bir bit nasıl saklanır `[kavram]`
- SR latch, D flip-flop fikri `[kavram]`
- Bu bilgi neden lazım: register ve cache'in fiziksel temeli burası

---

## Faz 1 — CPU Mimarisi (Düşünen kısım)

Bu fazın amacı: CPU'nun içinde ne olduğunu, bir talimatın nasıl işlendiğini,
"daha fazla core" veya "daha hızlı clock" demek ne anlama gelir — bunu anlamak.

### Konular

**1.1 CPU'nun temel bileşenleri**
- ALU (Arithmetic Logic Unit) — hesap yapan kısım `[kavram]`
- Register — CPU'nun içindeki geçici hafıza `[mekanizma]`
- Program Counter (PC) — sıradaki talimatı takip eder `[kavram]`
- Instruction Register — şu anki talimatı tutar `[kavram]`
- Control Unit — tüm parçaları koordine eder `[kavram]`

**1.2 Fetch-Decode-Execute döngüsü**
- Bir CPU talimatı nasıl alır, yorumlar ve çalıştırır `[mekanizma]`
- Bu döngünün hızı clock speed ile nasıl ilişkili `[mekanizma]`
- Cloud bağlantısı: neden "yüksek clock speed her zaman iyi değil"

**1.3 Clock speed ve IPC**
- GHz ne demek — saniyede kaç döngü `[mekanizma]`
- IPC (Instructions Per Clock) kavramı `[kavram]`
- Clock × IPC = gerçek performans fikri `[uygulama]`
- Cloud bağlantısı: tek thread'li uygulama vs çok thread'li uygulamada hangisi önemli

**1.4 Pipeline**
- Assembly bandı benzetmesi `[kavram]`
- Pipeline hazard nedir (veri bağımlılığı, dal tahmini) `[kavram]`
- Neden bazı iş yükleri pipeline'dan daha fazla yararlanır `[kavram]`

**1.5 Core, Thread, Hyper-Threading, vCPU**
- Fiziksel core nedir `[mekanizma]`
- Logical core / Hyper-Threading nasıl çalışır `[mekanizma]`
- vCPU = ne — EC2'de vCPU fiziksel olarak ne demek `[uygulama]`
- 8 vCPU alan bir EC2 instance gerçekte ne kullanıyor `[uygulama]`
- Cloud bağlantısı: CPU-bound vs I/O-bound workload'da core sayısının etkisi

**1.6 CPU ailelerini tanıma (mimari düzey)**
- Intel vs AMD vs ARM (Graviton) mimarilerinin farkı neden önemli `[uygulama]`
- Instruction Set Architecture (ISA) kavramı `[kavram]`
- x86-64 ve ARM arasındaki güç tüketimi farkı ve cloud maliyetiyle ilişkisi `[uygulama]`

---

## Faz 2 — Bellek Hiyerarşisi (En kritik faz)

Bu fazın amacı: hız-kapasite-maliyet üçgenini anlamak.
Cloud'daki neredeyse her mimari karar bu üçgenin bir yansımasıdır.

### Konular

**2.1 Neden hiyerarşi var**
- CPU ne kadar hızlı, RAM ne kadar yavaş — rakamlarla `[mekanizma]`
- "Memory wall" problemi `[kavram]`
- Hiyerarşinin mantığı: hızlı-küçük-pahalı / yavaş-büyük-ucuz `[mekanizma]`

**2.2 Register**
- CPU içinde, nanosaniyenin altında erişim `[kavram]`
- Kapasite: birkaç onlarca byte `[kavram]`

**2.3 Cache (L1 / L2 / L3)**
- SRAM nedir, DRAM'den farkı `[kavram]`
- L1: hangi core'a ait, boyut, hız `[mekanizma]`
- L2: core'a özel mi paylaşımlı mı, boyut, hız `[mekanizma]`
- L3: tüm core'lar arasında paylaşımlı, boyut, hız `[mekanizma]`
- Cache hit ve cache miss ne anlama gelir `[mekanizma]`
- Spatial locality ve temporal locality kavramları `[kavram]`
- Cache coherence: çok core'da aynı veri sorunları `[kavram]`
- Cloud bağlantısı: noisy neighbor'ın L3 üzerindeki etkisi, büyük L3 olan instance'ların avantajı

**2.4 RAM (Ana Bellek)**
- DRAM nasıl çalışır — kondansatör fikri, neden yavaş `[kavram]`
- Row, column, DRAM erişim süresi `[kavram]`
- DDR4 vs DDR5 farkı neden önemli `[kavram]`
- Volatile: güç kesilince neden silinir `[mekanizma]`
- Memory channel ve bandwidth kavramı `[uygulama]`
- Cloud bağlantısı: memory-optimized instance'ların varlık nedeni

**2.5 Sanal bellek**
- Fiziksel RAM her zaman yetmez — sanal adres uzayı neden icat edildi `[mekanizma]`
- Page ve page table kavramları `[mekanizma]`
- TLB (Translation Lookaside Buffer) — page table'ı hızlandıran cache `[kavram]`
- Page fault nedir, nasıl çözülür `[mekanizma]`

**2.6 Swap**
- RAM dolduğunda ne olur `[mekanizma]`
- Swap alanı neden disk üzerinde ve neden yavaş `[mekanizma]`
- OOM Killer — Linux RAM tamamen dolunca ne yapar `[kavram]`
- Cloud bağlantısı: üretimde swap neden tehlikeli, memory leak senaryosu

**2.7 NUMA (Non-Uniform Memory Access)**
- Çok CPU-socket'li sunucularda bellek erişimi neden eşit değil `[mekanizma]`
- Local vs remote NUMA node erişim farkı `[mekanizma]`
- Cloud bağlantısı: büyük EC2 instance'larında NUMA'nın performansa etkisi

---

## Faz 3 — Depolama (Kalıcı Hafıza)

Bu fazın amacı: IOPS, throughput ve latency üçlüsünü içselleştirmek.
EBS tipi seçimi, instance store tercihi, veritabanı storage tasarımı buraya dayanır.

### Konular

**3.1 HDD**
- Mekanik yapı: plaka, okuma kafası, spindle `[kavram]`
- Seek time nedir ve neden rastgele okuma yavaş `[mekanizma]`
- Sequential vs random okuma farkı rakamlarla `[mekanizma]`
- HDD'nin neden cloud'da neredeyse ölmekte olduğu `[uygulama]`

**3.2 SSD**
- NAND Flash nasıl çalışır — yüzer kapı transistörü fikri `[kavram]`
- Cell tipleri: SLC, MLC, TLC, QLC — hız, dayanıklılık, maliyet dengesi `[mekanizma]`
- Neden SSD'de seek time kavramı farklı anlam taşır `[kavram]`
- Write amplification ve erase block kavramı `[kavram]`
- Wear leveling nedir `[kavram]`

**3.3 Arayüzler: SATA vs NVMe**
- SATA: eski arayüz, bant genişliği sınırlı, SSD'yi yavaşlatır `[mekanizma]`
- NVMe: PCIe üzerinde çalışır, sıra ve kuyruk derinliği avantajı `[mekanizma]`
- Pratik gecikme ve throughput rakamları: SATA SSD vs NVMe SSD `[uygulama]`
- Cloud bağlantısı: instance store = NVMe, EBS = ağ üzerinden blok depolama

**3.4 IOPS, Throughput ve Latency**
- Latency: tek işlemin ne kadar sürdüğü (ms, μs, ns) `[mekanizma]`
- IOPS: saniyedeki işlem sayısı `[mekanizma]`
- Throughput: saniyedeki veri miktarı (MB/s, GB/s) `[mekanizma]`
- IOPS ve throughput arasında ne zaman hangisi darboğaz olur `[uygulama]`
- Küçük rastgele I/O = IOPS sınırlı, büyük sıralı I/O = throughput sınırlı `[uygulama]`
- Cloud bağlantısı: EBS gp3, io2, st1, sc1 seçimi bu bilgiye dayanır

**3.5 RAID (genel fikir)**
- RAID 0, 1, 5, 10 kavramsal farkları `[kavram]`
- Cloud'da RAID'in rolü vs managed storage servisleri `[uygulama]`

---

## Faz 4 — Sistem Bus'ları ve I/O (Parçaların birbirine bağlandığı yer)

Bu fazın amacı: CPU, RAM ve disk'in birbirleriyle nasıl konuştuğunu anlamak.
GPU workload'larındaki darboğazı, PCIe bant genişliğini bu faz açıklar.

### Konular

**4.1 Bus kavramı**
- Bus nedir — veri yolu fikri `[kavram]`
- Address bus, data bus, control bus `[kavram]`
- Bant genişliği ve gecikme bağlantısı `[kavram]`

**4.2 PCIe (Peripheral Component Interconnect Express)**
- Şerit (lane) kavramı — x1, x4, x8, x16 `[mekanizma]`
- PCIe nesilleri: Gen 3, Gen 4, Gen 5 — bant genişliği farkları `[mekanizma]`
- NVMe, GPU ve ağ kartı PCIe'ye nasıl bağlanır `[mekanizma]`
- Cloud bağlantısı: GPU instance'larında CPU-GPU bant genişliği darboğazı

**4.3 DMA (Direct Memory Access)**
- CPU olmadan cihaz verisi RAM'e nasıl gider `[mekanizma]`
- Neden DMA olmadan her I/O işlemi CPU'yu meşgul eder `[kavram]`
- Cloud bağlantısı: Nitro System'in DMA kullanımı

**4.4 Interrupt mekanizması**
- Donanım interrupt'ı nedir `[mekanizma]`
- Polling vs interrupt karşılaştırması `[kavram]`
- IRQ, interrupt handler kavramları `[kavram]`
- Cloud bağlantısı: yüksek paket hızlı ağ işlemlerinde interrupt overhead

**4.5 Chipset ve anakart mimarisi (genel bakış)**
- Northbridge, Southbridge kavramı (tarihsel) `[kavram]`
- Modern tek-chip mimarisi `[kavram]`
- Memory controller'ın CPU'ya entegrasyonu `[kavram]`

---

## Faz 5 — Ağ Donanımı (Cloud'un kan damarları)

Bu fazın amacı: bir paketin fiziksel olarak nasıl yolculuk ettiğini,
ağ kartının CPU ile nasıl çalıştığını, bandwidth ve latency'nin donanımsal temelini anlamak.

### Konular

**5.1 NIC (Network Interface Card)**
- NIC'in temel görevi `[mekanizma]`
- Ethernet frame ve MAC adresi `[mekanizma]`
- Transmit ve receive buffer'ları `[mekanizma]`
- Interrupt vs polling modları (NAPI) `[kavram]`

**5.2 Switching ve fiziksel ağ**
- Switch nasıl çalışır — MAC tablosu `[mekanizma]`
- Full duplex ve half duplex `[kavram]`
- 1G, 10G, 25G, 100G Ethernet farkı `[kavram]`

**5.3 Bandwidth ve latency (donanımsal açıdan)**
- Bandwidth: fiziksel hattın kapasitesi `[mekanizma]`
- Latency: elektrik sinyalinin tur süresi `[mekanizma]`
- Neden bandwidth yüksek olsa bile latency yüksek olabilir `[mekanizma]`
- Propagation delay, transmission delay, queuing delay `[kavram]`

**5.4 RDMA ve high-performance networking (farkındalık)**
- RDMA nedir, neden HPC ve ML cluster'larında kullanılır `[kavram]`
- InfiniBand ve RoCE `[kavram]`
- Cloud bağlantısı: AWS EFA (Elastic Fabric Adapter) varlık nedeni

---

## Faz 6 — Sanallaştırma Donanımı (Cloud'un teknik temeli)

Bu fazın amacı: EC2'nin altında gerçekte ne olduğunu anlamak.
Bu faz olmadan cloud kararları körü körüne alınır.

### Konular

**6.1 Sanallaştırma neden gerekli**
- Tek fiziksel sunucuyu birden fazla müşteriye vermek `[kavram]`
- İzolasyon ve kaynak paylaşımı gerginliği `[mekanizma]`

**6.2 Hypervisor tipleri**
- Type 1 (bare metal): doğrudan donanım üzerinde `[mekanizma]`
- Type 2 (hosted): host OS üzerinde `[mekanizma]`
- Örnekler: KVM, Xen, VMware ESXi, Hyper-V, VirtualBox `[kavram]`
- Cloud bağlantısı: AWS Nitro = KVM tabanlı Type 1

**6.3 Hardware-assisted virtualization**
- Intel VT-x ve AMD-V nedir, neden önemli `[mekanizma]`
- Ring seviyesi kavramı: ring 0 (kernel), ring 3 (user) `[mekanizma]`
- VMX root / non-root mod kavramı `[kavram]`
- VM exit nedir ve neden maliyetlidir `[mekanizma]`

**6.4 Memory virtualization**
- Shadow page table kavramı `[kavram]`
- Extended Page Tables (EPT) / Nested Page Tables (NPT) `[kavram]`
- Memory overcommit mekanizması `[mekanizma]`
- Balloon driver, KSM (Kernel Same-page Merging) `[kavram]`
- Cloud bağlantısı: bir EC2 instance'ının fiziksel RAM'deki yeri

**6.5 CPU sanallaştırması**
- vCPU'nun fiziksel core üzerine schedule edilmesi `[mekanizma]`
- CPU steal time nedir, nasıl ölçülür `[uygulama]`
- Credit bazlı CPU (t serisi) vs dedicated CPU (c, m, r serisi) `[uygulama]`

**6.6 I/O sanallaştırması**
- Emulated I/O vs paravirtualization `[kavram]`
- Virtio nedir `[kavram]`
- SR-IOV (Single Root I/O Virtualization) `[kavram]`
- Cloud bağlantısı: AWS Nitro, NIC ve storage'ı ayrı donanıma taşır, bu neden performans kazancı sağlar

**6.7 Container vs VM (donanım gözüyle)**
- cgroup ve namespace Linux kernel özellikleri `[mekanizma]`
- Container'lar neden kernel paylaşır, VM'ler neden paylaşmaz `[mekanizma]`
- Güvenlik izolasyon farkı `[uygulama]`
- Başlangıç süresi ve bellek overhead farkı rakamlarla `[uygulama]`

---

## Faz 7 — Cloud Bağlantısı (Her şeyin bir araya geldiği yer)

Bu fazın amacı: öğrenilen her konuyu gerçek cloud kararlarına dönüştürmek.
Bu faz "öğrenmek" değil, "uygulamak" fazıdır.

### Konular

**7.1 EC2 instance ailelerini fiziksel temelle okumak**
- General purpose (m): dengeli CPU/RAM ratio neden bu anlama gelir `[uygulama]`
- Compute optimized (c): clock speed ve L3 cache önem kazanıyor `[uygulama]`
- Memory optimized (r, x): bellek bandwidth ve kapasite önceliği `[uygulama]`
- Storage optimized (i, d): NVMe instance store, yüksek IOPS `[uygulama]`
- Accelerated (p, g, trn): GPU ve PCIe bant genişliği `[uygulama]`
- Graviton (arm): ARM mimarisi, güç/performans dengesi `[uygulama]`

**7.2 AWS Nitro System'i anlamak**
- Nitro card nedir — NIC ve storage'ın ayrı donanıma taşınması `[mekanizma]`
- Neden bu mimari noisy neighbor'ı azaltır `[uygulama]`
- Bare metal instance'ların varlık nedeni `[uygulama]`

**7.3 EBS seçimi fiziksel temelle**
- gp3: IOPS ve throughput bağımsız ayarlanır `[uygulama]`
- io2 / io2 Block Express: sub-millisecond latency, yüksek IOPS `[uygulama]`
- st1: throughput optimized HDD — sıralı erişim `[uygulama]`
- sc1: cold HDD — arşiv `[uygulama]`
- Hangi workload hangisini ister — karar tablosu `[uygulama]`

**7.4 Noisy neighbor — tam analiz**
- L3 cache kirlenmesi mekanizması `[uygulama]`
- Memory bandwidth yarışması `[uygulama]`
- PCIe bus satürasyonu `[uygulama]`
- AWS çözümleri: Dedicated Host, Placement Groups, Nitro `[uygulama]`

**7.5 Darboğaz tespiti (bütünleşik)**
- CPU bound, memory bound, I/O bound, network bound ayrımı `[uygulama]`
- Her darboğazın monitoring metrikleri (CloudWatch gözüyle) `[uygulama]`
- "Uygulamam yavaş" → doğru soruları sormak `[uygulama]`

**7.6 Kapasite planlaması**
- Bir iş yükü için instance tipi seçme süreci `[uygulama]`
- Right-sizing kavramı `[uygulama]`
- Over-provisioning vs under-provisioning maliyetleri `[uygulama]`

---

## Öğrenme derinliği özeti

| Konu | Gereken Derinlik | Neden |
|---|---|---|
| Sayısal sistemler | Mekanizma | Sonraki her şeyin dili |
| Mantık kapıları | Kavram | CPU'nun neyi yaptığını anlamak |
| CPU pipeline / core | Mekanizma | Instance seçimi, CPU bound analizi |
| Cache hiyerarşisi | Mekanizma | En sık mimari kararı etkileyen alan |
| RAM ve sanal bellek | Mekanizma | Memory leak, swap, OOM senaryoları |
| NUMA | Kavram | Büyük instance'larda performans |
| HDD / SSD fiziği | Kavram | EBS tipi seçimini gerekçelendirmek |
| IOPS / Throughput | Uygulama | Her storage kararı buraya dayanır |
| PCIe | Kavram | GPU instance'larını anlamak |
| DMA / Interrupt | Kavram | Yüksek I/O workload'larını anlamak |
| NIC ve ağ donanımı | Kavram | Bandwidth, latency kararları |
| Hypervisor | Mekanizma | EC2'nin temeli |
| Sanallaştırma donanımı | Mekanizma | vCPU, steal time, noisy neighbor |
| Container vs VM | Uygulama | Her gün alınan kararlar |
| EC2 instance aileleri | Uygulama | Doğrudan iş çıktısı |
| CMOS, transistör fiziği | Atla | Solutions Architect'in işi değil |

---

## Önerilen ilerleme biçimi

- Her faz bir öncekinin temel kelimelerini kullanır — sırayı atlamak birikimi kırar
- Faz 0-2 en yoğun faz, sabır ister
- Faz 3-5 nispeten hızlı ilerler — sezgi kazanıldıktan sonra
- Faz 6 en bağlayıcı fazdır — "bu yüzden böyle çalışıyor" anları burada olur
- Faz 7 teknik öğrenme değil, bağlam kurma fazıdır — gerçek senaryolar
- Her fazı bir testle veya açık uçlu soru setiyle kapatmak, ezberi değil anlamayı ölçer

---

*Hazırlayan: Denis Ergöçmen, Haziran 2026 — genel kullanım için nötrleştirilmiş sürüm*
