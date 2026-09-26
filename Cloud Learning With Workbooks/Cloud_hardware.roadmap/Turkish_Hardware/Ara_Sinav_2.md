# Ara Sınav 2 — Faz 4–7: Bus'tan Cloud Kararına

> **Navigasyon:** [◀ Faz 7 — Cloud Bağlantısı](Faz_7_Cloud_Baglantisi.md) · **Ara Sınav 2** · [Haritanın başı ▶](README.md)

---

## Bu sınav neyi ölçüyor?

Faz 0–3 makinenin **içini** kurdu: transistör, CPU, bellek, disk. Faz 4–7 bu makineyi **dış dünyaya**
bağlıyor ve sonra **kiraya veriyor**: veriyi taşıyan bus'lar ve DMA (Faz 4), ağ kartı ve tel (Faz 5), tek
sunucuyu çoğa bölen hypervisor (Faz 6) ve nihayet hepsinin bir instance tipine ve bir faturaya dönüştüğü cloud
kararları (Faz 7). Her fazın "Kendini sına" testi tek bir fazı yoklar. Bu sınav **farklı** bir şey ölçer:
*"Bir uygulama cloud'da yavaşladığında bütün yolu — CPU, bellek, disk, bus, ağ, hypervisor — yürüyüp gerçekten
suçlu olan katmanı adlandırabiliyor musun?"*

Bu fazlarda aynı üç fikrin ne kadar sık geri geldiğine dikkat et: **en pahalı bileşen, onu besleyen yol kadar
hızlıdır** (PCIe, GPU, EBS tavanı); **kuyruk eğrisi** (ring buffer, link, disk, kapasite planlaması);
**sanallaştırma vergisi, exit sayısı × maliyetidir** (VM exit, SR-IOV, Nitro). Bu sınav bölümleri değil, o üç
fikri görüp görmediğini kontrol eder.

**Nasıl çalışılır:**

- Kâğıt-kalemle çöz; kendi cevabını yazmadan cevap anahtarını okuma.
- Cevap anahtarı her sorunun **hangi fazların kesişiminde** durduğunu söyler — kaçırdığın soru bir fazı
  değil, iki faz arasındaki **köprüyü** işaret eder.
- 21 soru var: Bölüm A (bağlantı muhakemesi, 1–9), Bölüm B (senaryo: "tek bir API, bir çeyreklik olay",
  10–15), Bölüm C (çıktı okuma ve hesap, 16–21).
- İhtiyacın olan sayılar Ek A'da ve fazlarda. Hedef süre: ~1 saat. Süre önemli değil; her cevabı bir
  cümleyle gerekçelendirebilmek önemli.

> **🤔 Başlamadan önce:** Şu cümleyi kendi kelimelerinle tamamla: *"Uygulama yavaş, ama hiçbir kaynak dolu
> değil; ilk ölçtüğüm şey ___, çünkü ___."* Sonra ikinci bir cümle yaz: *"Daha pahalı bir ___ almak, onu besleyen
> yol ___ olduğunda yardım etmez."* Bu iki cümle 21 sorunun tamamından geçen ipliktir.

---

# Bölüm A — Bağlantı muhakemesi (1–9)

Bu bölümdeki her soru en az iki fazın bilgisini birleştirmeni ister. Kısa ama gerekçeli cevap ver.

**1.** Bir NVMe SSD 7.000 MB/s sıralı okuma için derecelendirilmiş (Faz 3.3.3) ve Gen4 x4 link için tasarlanmış.
Şuna takılırsa beklenebilecek en yüksek hız nedir: (a) Gen4 x4 slot, (b) Gen3 x4 slot, (c) ona yalnızca Gen4 x2
link veren bir M.2 slot? Faz 4.2.2'deki hangi cümle (b) ve (c)'de diske olanı anlatır?

**2.** Büyük bir dosya kopyalanırken `top` yüksek `wa` ve düşük `us` gösteriyor, ama makine CPU anlamında "meşgul"
değil. DMA (Faz 4.3) ve kesmeyi (Faz 4.4) kullanarak, diskin aldığı milisaniyelerde CPU'nun ne yaptığını anlat —
ve sistemde DMA olmasaydı (programmed I/O) `us`/`sy`'e ne olurdu?

**3.** Bir sunucu CPU'su 64 PCIe Gen4 lane sağlıyor. 2 GPU (her biri x16), 4 NVMe disk (her biri x4) ve bir
100 GbE NIC istiyorsun. (a) NIC'e Gen4 x8 link verirsen kaç lane kalır? (b) x8 ile **ikinci** bir 100 GbE NIC
ekleyebilir misin? (c) Gen4 x8 neden 100 GbE NIC için yeterli ama Gen3 x8 değil?

**4.** 25 Gbps bir NIC, 1500 baytlık paketleri hat hızında alıyor. Her paket ~2 μs CPU işiyle kendi kesmesini
tetiklerse, yalnızca kesmeler kaç çekirdeğin zamanını ister? Sistemin bunu atlatmasını sağlayan iki mekanizmayı —
biri Faz 4.4'ten, biri Faz 5.1'den — adlandır ve her birinin ne yaptığını söyle.

**5.** Frankfurt'taki uygulama sunucusu Virginia'daki bir veritabanını çağırıyor ve sayfa yüklemesi 4 saniye
sürüyor. Biri linki 1 Gbps'ten 10 Gbps'e yükseltmeyi öneriyor. Faz 5.3.2'deki dört gecikme bileşenini kullanarak,
bunun 1500 baytlık bir paket için neden neredeyse hiçbir şeyi değiştirmediğini göster — ve 4 saniyenin nereden
geldiğini açıklayan hesabı ver.

**6.** Kuyruk eğrisi diskte (`await`, Faz 3.4.4), NIC'in içinde (ring buffer, Faz 5.1.3) ve linkte (Faz 5.3.2)
göründü. Bir ekip CPU ring'i yeterince hızlı boşaltamadığı için `rx_dropped` artıyor görüyor ve ring'i 512'den
4096'ya çıkarıyor. Kayıp azalıyor ama gecikme artıyor. Nedenini açıkla ve varış hızı servis hızını aştığında
gerçekten hangi tür değişikliğin yardım ettiğini söyle.

**7.** Bir guest VM, emüle edilmiş I/O ile saniyede 500.000 ağ paketi alıyor; her paket 2.500 döngülük bir VM
exit'e mal oluyor. CPU 3 GHz'de çalışıyor. VM exit'lere tek başına **bir çekirdeğin** kaçta kaçı harcanıyor?
"Sanallaştırma vergisi"ni hangi iki terim oluşturuyor ve SR-IOV/Nitro bu çarpımda neyi değiştirdi?

**8.** Çıplak metal adres çevirisi bir TLB miss'te en fazla 4 bellek erişimi gerektirir; EPT/NPT altında 24
gerekebilir. Bunun TLB'yi bir VM'de neden daha kritik yaptığını açıkla ve Faz 2.5.4'teki TLB kapsamı hesabını
(1.536 giriş; 4 KiB'a karşı 2 MiB sayfalar) kullanarak huge page'lerin veritabanı VM'leri için neden standart
öneri olduğunu anlat.

**9.** Faz 2.6, bellek yetersizliğinin performansı kademeli değil, uçurumdan düşer gibi bozduğunu öğretti
(thrashing). Faz 6.4.4, AWS'nin belleği **overcommit etmediğini** söylüyor. İkisini bağla: bu neden eksik bir
özellik değil de bir tasarım tercihi ve kurumsal VMware ortamlarından getirilen hangi alışkanlık AWS'de pahalıya
gelir?

---

# Bölüm B — Senaryo: "Tek bir API, bir çeyreklik olay" (10–15)

> **Olay:** Bir ekip AWS'de bir sipariş API'si çalıştırıyor. Bir çeyrek boyunca altı farklı sorunla
> karşılaşıyor, aşağıda karşılaşacağın sırayla. Her sorunun kendi kanıtı var. API, 100 ms'lik bir bütçe içinde
> cevap veriyor, Go ile yazılmış ve hayata bir `t3.large` (2 vCPU, 8 GiB) üzerinde başlamış.

**10.** API 5 dakikalık bir yük testini p99 = 40 ms ile geçti. Üretimde, yoğun trafiğin yaklaşık 40. dakikasında
p99 400 ms'ye sıçrıyor. O anda instance'taki `top`:

```
%Cpu(s): 19.8 us,  4.1 sy,  0.0 ni,  3.0 id,  0.3 wa,  0.0 hi,  0.8 si, 72.0 st
```

Teşhis hangi sayı, "gürültülü komşu" ile "kredi bitti"yi nasıl ayırırsın, instance'ı yeniden başlatmak her iki
durumda ne yapar ve 5 dakikalık yük testi neden yanılttı?

**11.** Faz 6.5.3'ün kredi modelini kullan (`t3.medium` sayıları: 2 vCPU, %20 baseline, saatte 24 kredi kazanılır,
tam hızda dakikada 2 kredi harcanır). (a) Hangi ortalama kullanım sonsuza dek sürdürülebilir? (b) Instance tepe
trafiğe 60 kredilik bir bakiyeyle başlıyor ve iki vCPU'yu %100'de çalıştırıyor: bakiye kaç dakikada sıfırlanır?
(c) 5 dakikalık bir yük testi kaç kredi harcar ve bu test hakkında ne söylüyor?

**12.** Ekip t ailesini bırakmaya karar veriyor. İki aday: `c6g.xlarge` ve `c6i.xlarge`. İkisinin adını çöz
(Faz 7.1.0). Her biri kaç **fiziksel çekirdek** verir ve "ikisi de 4 vCPU" neden tuzaktır? Graviton olanı seçmeden
önce doğrulanması gereken, geçiş kontrol listesinden üç maddeyi say.

**13.** Taşımadan sonra ekip API'yi dayanıklılık için birden çok AZ'ye dağıtıyor. Her istek artık başka bir
AZ'deki bir servise 12 **sıralı** çağrı yapıyor. Faz 5.3.4'teki RTT tablosunu kullanarak, tek başına ağ 100
ms'lik bütçenin ne kadarını tüketiyor, aynı AZ durumuyla karşılaştır. Zincir sonradan 20 çağrıya çıkarsa ne olur?
Bu kararı tartmanın doğru yolu nedir?

**14.** Gece çalışan bir iş, 100 ms uzaktaki bir ortağın sunucusundan 10 GiB'lık bir dosya çekiyor. Link 10 Gbps,
ama transfer saniyede birkaç yüz KB'de yürüyor. Host, 64 KiB pencereli tek bir TCP bağlantısı kullanıyor. (a) Bu
yolun BDP'si nedir? (b) 64 KiB'lık pencere hangi throughput'a izin verir ve 10 GiB'lık transfer ne kadar sürer?
(c) Üç çözüm ver ve `aws s3 cp`'nin hangisini zaten kullandığını söyle.

**15.** API gateway katmanında `top` ortalama yaklaşık %40 CPU gösteriyor, ama paket kaybı ve yüksek gecikme var.
Kanıt:

```
$ mpstat -P ALL 1                      (bir örnek)
CPU   %usr  %sys  %soft  %idle
all    9.7   5.8   23.7   60.8
  0    3.0   5.0   92.0    0.0
  1   12.0   6.0    1.0   81.0
  2   12.0   6.0    1.0   81.0
  3   12.0   6.0    1.0   81.0

$ cat /proc/interrupts | grep eth0
 24:   98234511          0          0          0   PCI-MSI  eth0-rx-0
 25:   97120344          0          0          0   PCI-MSI  eth0-rx-1
 26:   96877210          0          0          0   PCI-MSI  eth0-rx-2
 27:   97456002          0          0          0   PCI-MSI  eth0-rx-3

$ ethtool -g eth0      →  RX current 512 / max 4096
$ ethtool -S eth0 | grep drop  →  rx_dropped: 1,204,331 (artıyor)
```

Ne oluyor, ortalama bunu neden saklıyor, çözüm ne ve hangi sırayla harekete geçersin, ve ring buffer'ı büyütmek
neden denenecek **son** şey?

---

# Bölüm C — Çıktı okuma ve hesap (16–21)

**16.** Bir ML eğitim işi, GPU belleği neredeyse dolu iken GPU kullanımını yaklaşık %33 gösteriyor. Eğitim
adımı başına GPU 0,5 s hesaplıyor; veri hattı (disk → CPU ön işleme → PCIe) sonraki batch'i teslim etmek için 1,0
s ihtiyaç duyuyor ve hiçbir şey örtüşmüyor. (a) %33 rakamını doğrula. (b) Yükleme hesaplamayla örtüşecek şekilde
prefetching olursa adım süresi ve kullanım nedir? (c) GPU'nun darboğaz olması için hat kaç kat hızlanmalı? (d)
Daha pahalı bir GPU yardım eder mi? (e) GPU'nun kendi bellek bant genişliğini (~2.500 GB/s) Gen4 x16 ile (Faz
4.2.2) karşılaştır: besleme hattı ne kadar dar?

**17.** Bir 100 GbE NIC (Faz 5.2.3), **PCIe Gen3 x8** olarak eğitilen bir slota takılmış. (a) NIC'in hat hızı
GB/s cinsinden nedir ve slotun tavanı nedir? (b) Sistem gerçekte Gbps cinsinden hangi throughput'a ulaşabilir? (c)
Hangi slot yapılandırmaları düzeltir (Gen4 x8, Gen3 x16, Gen4 x4)?

**18.** Üç sunucu steal time bildiriyor:

| Sunucu | Instance | `st` | Ek kanıt |
|---|---|---|---|
| A | `m6i.large` | %1,5 | — |
| B | `m6i.large` | %14 | Host'ta başka kiracılar; `CPUCreditBalance` uygulanamaz |
| C | `t3.medium` | %61 | `CPUCreditBalance` = 0 |

Her birini Faz 6.5.2'nin yorum tablosuyla sınıflandır ve eylemi ver. C'nin çaresi B'ninkinden neden farklı ve C'yi
yeniden başlatmak neden hiçbir şey yapmaz?

**19.** Bir `io2` volume (64.000 IOPS), EBS sınırı yaklaşık 10.000 IOPS olan bir `m7i.large`'a takılmış.
`iostat -x`:

```
Device    r/s     w/s    await  aqu-sz
nvme1n1   8400    1550    6.0     60
```

(a) Sistem hangi tavanda? (b) Sayıları Little yasasıyla kontrol et. (c) Nerede israf var ve iki makul çözüm
nedir? Hangi komut instance'ın sınırını gösterir?

**20.** Tepe trafik saniyede 3.800 istek. Bir instance %100 CPU'da saniyede 1.000 istek işliyor. İki kapasite
planını karşılaştır: her instance'ı tepede ~%95'te çalıştırmak ya da ~%60'ta. (a) Her plan kaç instance ister? (b)
Tepede bir instance arızalanırsa hayatta kalanlar hangi kullanımla karşılaşır? (c) Kuyruk eğrisi rakamlarıyla %95
hedefinin neyi satın aldığını ve neyi riske attığını açıkla; ve fazla provizyon neden "görünür", eksik provizyon
neden değil?

**21.** Her makineyi Faz 7.5'in beş darboğaz sınıfından birine (CPU, bellek, I/O, ağ, bekleme) yerleştir ve
teşhis prosedürünün onu gösterecek ilk komutunu adlandır:

- (a) `top`: %92 `us`; `perf stat` IPC 1,8 gösteriyor.
- (b) `top`: %6 `us`, %90 `id`, `wa` 0, `st` 0; `iostat` boşta; `free` sağlıklı; yine de p99 2 s ve thread'ler bir
  veritabanı bağlantı havuzunda bloklu bekliyor.
- (c) `vmstat 1`: `si` 300, `so` 400; `free -h`: `available` 90 Mi.
- (d) `iostat -x`: r/s + w/s volume'un sınırında; `await` ve `aqu-sz` ikisi de yüksek.
- (e) `sar -n DEV 1`: 10 Gbps linkte 9,4 Gbps; `rx_dropped` artıyor.

---

## Cevap anahtarı

**1.** (a) Gen4 x4 = 4 × 2,0 = **8 GB/s** tavan → disk derecelendirilmiş **~7.000 MB/s**'ye ulaşır. (b) Gen3 x4 = 4 × 1,0
= **4 GB/s** → yaklaşık **4.000 MB/s**. (c) Gen4 x2 = 2 × 2,0 = **4 GB/s** → yine yaklaşık **4.000 MB/s**. Workbook'un
cümlesi: *"disk yavaşlamaz, yol daralır."* "Hızlı SSD aldım ama hızı alamıyorum" şikâyetinin nedeni genellikle
disk değil, takıldığı slotun nesli ya da lane sayısıdır. · *Faz 3.3.3 × Faz 4.2.2*

**2.** DMA ile CPU'nun katkısı iki kısa andır: denetleyiciye talimat verir ("1 MB'ı şu adrese oku, bitince haber
ver") ve tamamlanma **kesmesini** işler. Arada disk veriyi kendi başına RAM'e taşır ve CPU başka thread'leri
çalıştırır; bekleyen process uyur. Çalıştıracak başka bir şey yoksa süre `us`/`sy` değil **`wa`** (I/O bekleme)
olarak hesaplanır. DMA olmasaydı (programmed I/O) CPU her kelimeyi kendisi kopyalardı — 1 MB için 262.144 transfer —
bu da meşgul süre olarak görünür ve sunucuyu kilitlerdi. · *Faz 3.4.3 × Faz 4.3.2*

**3.** (a) 2 × 16 + 4 × 4 + 8 = 32 + 16 + 8 = 56 lane kullanıldı → **8 lane kalır**. (b) Evet — x8'lik ikinci bir NIC tam
kalan 8 lane'i kullanır ve sonra bütçe **tamamen harcanmış** olur (başka hiçbir cihaz için yer kalmaz). (c) 100 GbE =
12,5 GB/s (protokol verimliliği sonrası ≈ 11,9 GB/s kullanılabilir). Gen4 x8 = 8 × 2,0 = **16 GB/s** ≥ 12,5 → sığar;
Gen3 x8 = 8 × 1,0 = **8 GB/s** < 12,5 → sınır NIC değil slot olur. Lane sayısı, sunucu CPU'larını masaüstü
CPU'lardan ayıran birinci sınıf bir özelliktir. · *Faz 4.2.3 × Faz 5.2.3*

**4.** 25 Gbps ÷ (1500 × 8 bit) ≈ **saniyede 2,08 milyon paket**; × 2 μs = **saniyede ~4,2 saniye CPU** — yalnızca kesme
almak için dört çekirdekten fazla. Hayatta kalma mekanizmaları: **NAPI** (Faz 4.4.2 / 5.1.4) — düşük trafikte kesme
kullan (düşük gecikme, CPU boşta kalabilir), yüksek trafikte **kesmeleri kapat ve** toplu olarak **poll et** (tur
başına ~64–300 paket bütçesi), kuyruk boşalınca kesmeleri yeniden aç; ve **RSS** (Faz 5.1.4) — NIC bağlantı
demetini hash'leyip paketleri farklı çekirdeklere bağlı birkaç RX kuyruğuna dağıtır, böylece iş paralel olur ve bir
bağlantı hep aynı çekirdekte kalır (sıralama ve cache yerelliği). · *Faz 4.4.2 × Faz 5.1.4*

**5.** Dört bileşen: yayılma + iletim + işleme + kuyruklama. 1500 baytın iletimi: 1.500 × 8 ÷ 1 Gbps = **12 μs**; 10
Gbps'te = **1,2 μs**. Kıtalararası yayılma tek yönde ~50.000 μs olduğundan toplam ~50.012 μs'den ~50.001 μs'ye iner —
yaklaşık %0,02: **hiçbir şey**. 4 saniye **gidiş-dönüşlerden** gelir: RTT ~100 ms iken (Frankfurt–Virginia
kıtalararası ~100–200 ms sınıfında) ~40 sıralı sorgu yapan bir sayfa 40 × 100 ms = **4 s** saf yayılma öder. Bant
genişliği yanlış koldur; gidiş-dönüş sayısını azalt (sorguları topla, cache'le, veriyi yaklaştır). ·
*Faz 5.3.2 × Faz 5.3.4*

**6.** Ring, CPU'nun hızıyla boşalır; varışlar onu aşar, yani kuyruk hep doludur. Daha derin bir ring paketleri
yalnızca **daha uzun bir kuyrukta daha uzun bekletir** (bufferbloat) — daha az paket düşer, her biri daha geç
gelir. Diskteki `await` ile aynı kuyruk eğrisidir. Yardım eden şey **servis hızını** değiştirmektir (ring'i RSS/IRQ
dağıtımıyla boşaltan daha fazla çekirdek, paket başına daha az iş, daha hızlı CPU/NIC) ya da **varış hızını**
azaltmak; tamponu büyütmek yalnızca kısa patlamalara yardım eder, kalıcı olarak aşırı yüklü bir tüketiciye değil. ·
*Faz 3.4.4 × Faz 5.1.3 × Faz 5.3.2*

**7.** 500.000 × 2.500 = 1,25 × 10⁹ döngü/s; ÷ 3 × 10⁹ = **≈ bir çekirdeğin %42'si**, guest daha hiçbir iş yapmadan.
Vergi **VM exit sayısı × bir exit'in maliyeti**dir (1.000–5.000 döngü; bir pipeline flush artı cache/TLB kirliliği —
branch misprediction cezasıyla aynı mekanizma). SR-IOV (bir sanal fonksiyonun doğrudan guest'e atanması) ve Nitro
(I/O'nun ayrı kartlarda yapılması) I/O yolundaki exit'lerin çoğunu kaldırır; vergiyi %30–50'den (binary translation)
ve %5–15'ten (VT-x + EPT) **%1'in altına** indirir. · *Faz 1.4.2 × Faz 6.3.4 × Faz 6.6.4*

**8.** MMU iki tabloyu yürür (guest → GPA, EPT → HPA); tam bir yürüyüş 4 yerine **24** erişim gerektirebilir — bir TLB
miss 6 kata kadar daha pahalıdır. Miss oranını düşüren her şey bu yüzden daha çok kazandırır. TLB kapsamı: 1.536 ×
4 KiB = **6 MiB**; 1.536 × 2 MiB = **3 GiB** (512 kat geniş). Huge page'ler hem TLB miss'lerini azaltır hem de iki
yürüyüşün seviyelerini kısaltır — veritabanı VM'lerinde huge page'lerin standart öneri olmasının nedeni budur. ·
*Faz 2.5.3–2.5.4 × Faz 6.4.3*

**9.** Overcommit öngörülemez performans demektir ve öngörülebilirlik cloud sözleşmesinin temelidir: bir guest
hypervisor seviyesinde sıkıştırılırsa (balloon, KSM ya da hypervisor swap), guest nedenini bilmeden thrashing
uçurumuna düşer. Yani AWS 32 GB dediğinde 32 GB verir — gerçekten ayrılmış ve gerçekten faturalanmış. Pahalı
alışkanlık: kurumsal VMware'den gelen **"RAM'i cömertçe ver, zaten overcommit var"** — AWS'de her GB ödenir, bu
yüzden belleği şişirmek yerine doğru boyutlandır. · *Faz 2.6.2 × Faz 6.4.4*

**10.** Teşhis **`st` = %72** ("kabul edilemez" %25+ bandının üstünde): vCPU'lar çalışmaya hazır ama fiziksel çekirdek
bekliyor; `us` yalnızca ~%20. İki nedeni ayır: CloudWatch **`CPUCreditBalance`** — t ailesi instance'ta 0 ise AWS
seni baseline'a doğru **bilerek** kısıyor; sağlıklı bakiyeyle yüksek `st` aşırı yüklü bir host'a işaret eder.
Yeniden başlatmak kredi tükenmesinde **hiçbir şey yapmaz** (yalnızca t ailesinden çıkmak ya da `unlimited` için
ödemek yardım eder); gürültülü host'ta yeniden başlatma başka bir host'a düşürebilir. 5 dakikalık test yanılttı,
çünkü kredi ve burst kotaları 5 dakikada bitmez — üretimde 20–40. dakika civarında biter; **en az 30 dakika** yük
testi yap ve asla t instance'ında yapma. · *Faz 6.5.2–6.5.3 × Faz 7.6.1*

**11.** (a) Baseline: instance'ın **%20**'si — dakikada 0,4 vCPU-dakika = saatte 24 kredi kazanç. (b) Tam hız dakikada 2
kredi harcar, 0,4/dk hâlâ kazanılır: net tüketim 1,6/dk → 60 ÷ 1,6 = **37,5 dakika**. (c) 5 dakikalık test ~10 kredi
harcar (net ~8) — duvara hiç yaklaşmaz; instance tam da test bakiyeden kısa olduğu için "hızlı" görünür. Karar
kuralı: sürekli yük (üretim API'si) → m/c/r; t yalnızca aralıklı, düşük ortalamalı işler için. · *Faz 6.5.3*

**12.** `c6g.xlarge`: **c** = compute optimized, **6** = 6. nesil, **g** = Graviton (ARM), **xlarge** = 4 vCPU. `c6i.xlarge`:
aynısı, ama **i** = Intel. Graviton'da **1 vCPU = 1 fiziksel çekirdek** → 4 çekirdek; x86'da **1 vCPU = 1 SMT thread
(çekirdeğin yarısı)** → **2 fiziksel çekirdek**. "Aynı vCPU sayısı" fiziksel hesap gücünde 2 kata kadar fark saklar
(Graviton'un fiyat/performansının %20–40 daha iyi olmasının bir nedeni). Geçmeden önce doğrula (herhangi üçü):
dil ARM64'ü destekliyor (Go ✅), native bağımlılıkların ARM64 build'i var, Docker imajları multi-arch, üçüncü
taraf ajanlar (APM/log/güvenlik) ARM64'ü destekliyor — *en sık engel* — ve CI/CD ARM64 build üretebiliyor. ·
*Faz 1.5.3 × Faz 7.1.6*

**13.** AZ'ler arası RTT ≈ 1–2 ms (1,5 al): 12 × 1,5 = **18 ms** yalnızca ağ, yani bütçenin %18'i; aynı AZ (0,3–0,5 ms,
0,4 al): 12 × 0,4 = **4,8 ms**. 20 çağrıyla **30 ms** — hesaplama daha başlamadan bütçenin üçte biri. Dayanıklılık
gerçek bir kazanç ve gecikme bütçesi de gerçek: ikisini **bilerek, refleksle değil** tart — sıralı AZ'ler arası
çağrıları azalt (toplu ya da paralel), gevezelik eden servisleri aynı AZ'de tut, yalnızca dayanıklılık gereken
şeyi AZ'lere yay. · *Faz 5.3.4 × Faz 7.6*

**14.** (a) BDP = bant genişliği × RTT = 1,25 GB/s × 0,1 s = **125 MB** — TCP'nin linki doldurması için ihtiyaç duyduğu
pencere. (b) 64 KiB ÷ 0,1 s = 655.360 B/s ≈ **640 KiB/s ≈ 5 Mbps** — linkin yaklaşık **%0,05**'i. 10 GiB ÷ 640 KiB/s =
10.485.760 KiB ÷ 640 = 16.384 s ≈ **4,5 saat**. (c) Çözümler: TCP window scaling (modern sistemlerde varsayılan
açık) ve daha büyük `net.ipv4.tcp_rmem`/`tcp_wmem`; **paralel akışlar** (10 akış = 10× pencere; `aws s3 cp` ve
indirme hızlandırıcıları tam bunu yapar); uzun mesafeye daha uygun bir tıkanıklık kontrolü (**BBR**). Sorun link
değil, penceredir. · *Faz 5.3.3*

**15.** **Kesme dengesizliği.** Dört RX kuyruğunun kesmelerinin hepsi **CPU0**'a düşüyor: çekirdek 0 %92 `%soft` ve
%0 boşta iken çekirdek 1–3 %81 boşta. Ortalama (`top` ~%39 meşgul diyor) boğulan çekirdeği saklıyor ve ring düşüyor,
çünkü o tek çekirdek onu boşaltamıyor. Eylem sırası: (1) `mpstat -P ALL 1` ile çekirdek başına doğrula; (2)
`/proc/interrupts`'a bak — tüm sayılar tek bir sütunda; (3) IRQ'ları çekirdekler arasında **dağıt** (MSI-X kuyrukları
`smp_affinity`/`irqbalance` ile farklı CPU'lara bağla) ve RSS'yi `ethtool -l eth0` ile kontrol et; (4) ancak
sonra ring boyutunu düşün. Ring buffer'ı büyütmek aşırı yüklü bir tüketicinin önündeki kuyruğu derinleştirir (daha
fazla gecikme, yük altında aynı kayıp) — yine kuyruk eğrisi. · *Faz 4.4.3 × Faz 5.1.3–5.1.4*

**16.** (a) Kullanım = 0,5 ÷ (0,5 + 1,0) = **%33** ✓. (b) Örtüşmeyle adım max(0,5; 1,0) = **1,0 s** sürer; kullanım =
0,5 ÷ 1,0 = **%50**. (c) Hat, GPU sınır olmadan önce bir batch'i ≤ 0,5 s'de teslim etmeli — **2 kat daha hızlı**. (d)
**Hayır** — daha pahalı bir GPU yalnızca daha çok bekler; veri yükleyicide daha fazla worker, prefetching, daha hızlı
depolama/format, ön işlemeyi GPU'ya taşımak, daha büyük batch ya da veriyi GPU belleğinde tutmak çözümlerdir. (e)
~2.500 GB/s ÷ 32 GB/s ≈ **78 kat** daha dar (workbook ~70 der). `nvidia-smi` kullanımını, çekirdek başına
`top`/`mpstat`'ı, disk `iostat`'ı izle. · *Faz 3.4.4 × Faz 4.2.4*

**17.** (a) 100 Gbps ÷ 8 = **12,5 GB/s** (≈ 11,9 GB/s kullanılabilir); Gen3 x8 = 8 × 1,0 = **8 GB/s**. (b) Slot
sistemi **8 GB/s ≈ 64 Gbps**'e sınırlar — NIC'in yaklaşık üçte biri boşa gider. (c) **Gen4 x8** (16 GB/s ✓), **Gen3
x16** (16 GB/s ✓); Gen4 x4 = 8 GB/s ✗. Slotun kapasitesi NIC'in kendi hızı kadar belirleyicidir. ·
*Faz 4.2.2 × Faz 5.2.3*

**18.** **A: %1,5 → normal, hiçbir şey yapma** (%0–2). **B: %14 → ciddi** (%10–25): host aşırı yüklü → yeniden başlat
(instance başka bir host'a düşebilir), daha büyük ya da dedicated instance. **C: %61 → kabul edilemez** (%25+) ve
`CPUCreditBalance` = 0, **AWS'nin bilerek kıstığını** gösterir — komşu yok — bu yüzden yeniden başlatmak **hiçbir şey
yapmaz**; tek çare **aileyi değiştirmek** (t → m/c) ya da `unlimited` için ödemektir. Neden çareyi belirler; bu
yüzden t instance'larında steal time hep kredi metriğiyle birlikte okunmalıdır. · *Faz 6.5.2–6.5.3*

**19.** (a) r/s + w/s = 8.400 + 1.550 = **9.950 IOPS** — volume'un (64.000) değil, **instance'ın EBS tavanında**
(~10.000). (b) 60 ÷ 9.950 ≈ 0,00603 s ≈ **6,0 ms** = `await` ✓ — istekler zamanlarını kuyrukta bekleyerek geçiriyor.
(c) Pahalı io2 yeteneği **israf** (64.000 ödenmiş, 10.000 kullanılabilir). Çözümler: EBS bant genişliği/IOPS tavanı
daha yüksek **daha büyük bir instance** ya da instance'ın kullanabildiğine uyan **daha ucuz bir volume** (örn.
~10.000 IOPS'a provizyonlanmış gp3). `aws ec2 describe-instance-types --instance-types … --query
'InstanceTypes[0].EbsInfo'` instance'ın sınırını gösterir; disk performansı satın alırken **iki tavanı** da kontrol
et. · *Faz 3.4.5 × Faz 7.3.5*

**20.** (a) %95 planı: 3.800 ÷ 950 = **4 instance**. %60 planı: 3.800 ÷ 600 = 6,33 → **7 instance** (1,75× filo). (b) %95
planı: hayatta kalan 3 instance 3.800'ü taşımalı → her biri 1.267 istek/s = **%127 — aşırı yük, çığ**. %60 planı: 6
hayatta kalan → her biri 633 istek/s = **%63** — sorun yok. (c) %95 hedefi kâğıt üzerinde daha düşük fatura satın alır
ama kuyruk eğrisinin dik kısmında oturur: **%95 kullanım ≈ %50'nin 8 katı gecikme**, ve herhangi bir arıza ya da
sıçrama uçurumdan aşağı düşer; CPU ortalaması %40–60 ve p99 < %80 hedefleri boşluk bırakır. Fazla provizyon
**faturada görünür** ve düzeltmesi kolaydır; eksik provizyon **görünmezdir** — kaybedilen müşteri ve olaylarla
ödenir — bu yüzden makul miktarda fazla provizyon rasyoneldir, "makul" ölçümle tanımlanır, üstüne auto scaling
konur. · *Faz 3.4.4 × Faz 7.6.2–7.6.3*

**21.** (a) **CPU-bound** (IPC yüksek, `us` yüksek) — Adım 1 `top`/`mpstat`. (b) **Bekleme-bound** — her şey boşta ama
uygulama yavaş; bir kilit/havuzda bloklu — Adım 5 (uygulama profillemesi), eleme yoluyla varılır. (c) **Bellek-bound**
ve **swap** — `si`/`so` > 0 hemen ele alınacak bir felakettir — Adım 2 `free -h`/`vmstat 1`. (d) **I/O-bound** —
`await` + `aqu-sz` yüksek, IOPS tavanında — Adım 3 `iostat -x 1`. (e) **Ağ-bound** — bant genişliği tavanda, kayıplar
artıyor — Adım 4 `ss -s`/`ethtool -S`/`sar -n DEV`. Sıra (CPU → bellek → disk → ağ → bekleme) en ucuz ve en kesin
ölçümden en pahalıya doğru gider. · *Faz 7.5.1–7.5.2*

---

## Puanlama

| Doğru sayısı | Değerlendirme |
|---|---|
| 18–21 | Bus'tan faturaya bütün yolu yürüyebiliyorsun. Donanım haritası senin. |
| 14–17 | İyi. Kaçırdığın soruların işaret ettiği **köprü** bölümlerine (aşağıdaki tablo) bir tur dön. |
| 9–13 | Fazları ayrı ayrı biliyorsun ama kesişimde zorlanıyorsun. Bölüm B senaryosunu baştan, soru soru çöz. |
| 0–8 | Faz 4, 5, 6 ve 7'yi yeniden gez; özellikle 4.2 (PCIe), 5.3 (gecikme ve BDP), 6.3–6.5 (VM exit, steal) ve 7.5 (teşhis prosedürü). |

**Hangi soruyu kaçırırsan nereye dön:**

| Kaçırdığın soru | Dön — bu köprü zayıf |
|---|---|
| 1, 17 | PCIe nesilleri ve lane'ler × beslediği cihaz (3.3.3 × 4.2.2 × 5.2.3) |
| 2 | DMA, kesme ve `wa` (3.4.3 × 4.3 × 4.4) |
| 3 | Lane bütçesi (4.2.3) |
| 4 | Kesme fırtınası, NAPI, RSS (4.4.2 × 5.1.4) |
| 5, 13 | Gecikmenin dört bileşeni, RTT tablosu (5.3.2 × 5.3.4) |
| 6 | Ring buffer'da kuyruk eğrisi (3.4.4 × 5.1.3) |
| 7 | VM exit maliyeti ve sanallaştırma vergisi (6.3.4 × 6.6) |
| 8 | TLB × EPT × huge page (2.5.3–2.5.4 × 6.4.3) |
| 9 | Bellek overcommit'e karşı thrashing (2.6.2 × 6.4.4) |
| 10, 11, 18 | Steal time ve t ailesi kredileri (6.5.2–6.5.3 × 7.6.1) |
| 12 | vCPU ≠ çekirdek, Graviton (1.5.3 × 7.1.6) |
| 14 | BDP ve TCP penceresi (5.3.3) |
| 15 | Kesme dağıtımı, çekirdek başına okuma (4.4.3 × 5.1.4) |
| 16 | GPU besleme hattı (4.2.4) |
| 19 | İki EBS tavanı (3.4.5 × 7.3.5) |
| 20 | Kapasite planlaması ve kuyruk eğrisi (7.6.2–7.6.3) |
| 21 | 20 dakikalık teşhis prosedürü (7.5) |

---

## Kapanış — buradan sonraki haritaya

Bu sınav tek bir uygulamayı dört katmandan yürüttü: yolu daraltabilen bir **bus** (Faz 4), gecikmesi çoğunlukla
coğrafya olan bir **ağ** (Faz 5), vergisi exit'ler × maliyet olan bir **hypervisor** (Faz 6) ve hepsinin bir
instance tipine ve faturaya dönüştüğü **kararlar** (Faz 7). Senaryonun dersi ilk sınavınkini tekrarlar: **belirti
bir yerde görünür, neden bir katman ötede oturur** — "meşgul görünen" CPU kısılıyordu (`st`), "yavaş ağ" bir pencere
ve gidiş-dönüş sayısıydı, "paket kaybı" kesmelerde boğulan tek bir çekirdekti. Üç alışkanlığı ileri taşı: (1)
katman katman ölç, sırayla CPU → bellek → disk → ağ → bekleme; (2) satın almadan önce aritmetiği yap (BDP, kredi,
lane bütçesi, exit maliyeti); (3) hiçbir kaynağı kuyruk eğrisinin dik kısmında çalıştırma.

Donanım haritası burada biter. Zemini, sonraki katmanların üzerinde durduğu şeydir: **Linux haritası** (çalışan bir
sunucuyu okumak) ve **Ağ haritası** (IP, TCP, DNS, TLS) ve onlardan sonra AWS servisleri, Terraform, container'lar ve
gözlemlenebilirlik. Servisler değişir, fiyatlar değişir; fizik değişmez.

---

> **Navigasyon:** [◀ Faz 7 — Cloud Bağlantısı](Faz_7_Cloud_Baglantisi.md) · **Ara Sınav 2** · [Haritanın başı ▶](README.md)
