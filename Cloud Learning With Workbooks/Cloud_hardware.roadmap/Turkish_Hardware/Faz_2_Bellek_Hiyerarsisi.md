# Faz 2 — Bellek Hiyerarşisi

> **Navigasyon:** [◀ Faz 1 — CPU Mimarisi](Faz_1_CPU_Mimarisi.md) · **Faz 2** · [Faz 3 — Depolama ▶](Faz_3_Depolama.md)

---

> ### ⚠️ Bu faz haritanın en kritik fazıdır. Acele etme.
>
> Diğer fazlar bir konuyu öğretir. Bu faz bir **düşünme biçimi** öğretir: *hız–kapasite–maliyet
> üçgeni*. Cloud'daki neredeyse her mimari karar bu üçgenin bir yansımasıdır — instance
> ailesi seçimi, EBS tipi, CDN kullanmak, Redis koymak, veritabanı index tasarımı, hatta
> mikroservis sınırlarını çizmek.
>
> Bu fazı yüzeysel geçersen sonraki fazlar "bilgi" olarak kalır. İçselleştirirsen **sezgi**
> olur.

---

## Nereden geliyoruz

Faz 1 boyunca aynı duvara defalarca çarptık:

| Nerede | Ne gördük |
|---|---|
| 1.2.3 | DRAM erişimi ~250 clock çevrimi sürüyor |
| 1.3.2 | Bellek yoğun iş yüklerinde IPC 0,2–0,7'ye düşüyor |
| 1.4 | Pipeline, beklemeyi *gizlemeye* çalışıyor ama çözmüyor |
| 1.5.2 | SMT, beklerken başka bir thread çalıştırıp boşluğu doldurmaya çalışıyor |

Hepsi aynı sorunun etrafından dolaşma girişimiydi. **Bu faz sorunun kendisine saldırıyor.**

## Bu fazın sonunda

- 64 byte'lık cache satırının neden dünyanın en önemli sayılarından biri olduğunu bileceksin
- Aynı veriyi aynı sayıda okuyan iki döngüden birinin neden **10 kat** yavaş olabileceğini
  açıklayabileceksin
- Noisy neighbor'ın **mekanizmasını** (sadece adını değil) anlatabileceksin
- Sanal adresten fiziksel adrese giden yolu adım adım çizebileceksin
- Üretimde swap'in neden tehlikeli olduğunu ve OOM Killer'ın nasıl karar verdiğini bileceksin
- NUMA'nın büyük instance'larda neden sessizce performans yediğini göreceksin

## Faz haritası

| Bölüm | Konu | Derinlik | Neden burada |
|---|---|---|---|
| 2.1 | Neden hiyerarşi var | `[mekanizma]` | Üçgenin kendisi |
| 2.2 | Register | `[kavram]` | Hiyerarşinin tepesi |
| 2.3 | **Cache (L1/L2/L3)** | `[mekanizma]` | **Fazın kalbi — en uzun bölüm** |
| 2.4 | RAM / DRAM | `[kavram]` / `[uygulama]` | Bant genişliği ve memory-optimized aileler |
| 2.5 | Sanal bellek | `[mekanizma]` | Adres çevirisi, TLB, huge page |
| 2.6 | Swap | `[mekanizma]` | Üretimdeki en sinsi performans katili |
| 2.7 | NUMA | `[mekanizma]` / `[uygulama]` | Büyük instance'ların gizli vergisi |

---
---

# 2.1 Neden Hiyerarşi Var

## 2.1.1 Uçurumun boyutu `[mekanizma]`

Faz 0.4.4'te SRAM ve DRAM'in fiziksel farkını öğrendik. Şimdi o farkın sonucuna rakamlarla
bakalım.

**Tam gecikme merdiveni** (3 GHz bir CPU varsayımıyla, yaklaşık değerler — nesle göre değişir):

| Seviye | Gecikme | Clock çevrimi | Tipik boyut |
|---|---|---|---|
| **Register** | ~0,1 ns | ~0 (aynı çevrim) | ~1 KB |
| **L1 cache** | ~1 ns | 4–5 | 32–48 KiB / çekirdek |
| **L2 cache** | ~4 ns | 12–20 | 512 KiB – 2 MiB / çekirdek |
| **L3 cache** | ~15–40 ns | 40–120 | 8–384 MiB (paylaşımlı) |
| **DRAM (RAM)** | ~80–100 ns | 240–300 | GB'lar |
| **NVMe SSD** | ~20–100 µs | ~60.000–300.000 | TB'lar |
| **SATA SSD** | ~100–500 µs | ~300.000–1.500.000 | TB'lar |
| **HDD** | ~5–10 ms | ~15.000.000–30.000.000 | TB'lar |
| **Aynı AZ ağ gidiş-dönüş** | ~0,25–0,5 ms | ~750.000–1.500.000 | — |
| **Kıtalararası ağ** | ~70–150 ms | ~200.000.000+ | — |

**Bu tabloyu bir kez insan ölçeğine çevirelim.** Register erişimini **1 saniye** kabul edelim:

| Seviye | İnsan ölçeğinde |
|---|---|
| Register | 1 saniye |
| L1 cache | 10 saniye |
| L2 cache | 40 saniye |
| L3 cache | 3–7 dakika |
| **DRAM** | **~15 dakika** |
| NVMe SSD | **~3–14 saat** |
| HDD | **~2–4 ay** |
| Kıtalararası ağ | **~2–5 yıl** |

> **Bu tabloyu ezberlemene gerek yok — ama ölçek hissini içselleştir.** Bir cloud
> engineer'ın kafasındaki en değerli sezgilerden biri şudur:
>
> **Cache → RAM → disk → ağ** geçişlerinin her biri, bir öncekine göre **kat kat** daha
> pahalıdır. Bir katman aşağı inmek "biraz yavaş" değil, "başka bir dünya" demektir.
>
> Bir sorgunun neden 200 ms sürdüğünü çözerken, hangi katmanlara kaç kez gidildiğini
> sayabilmek teşhisin yarısıdır.

![Şekil 2.1 — Bellek hiyerarşisi piramidi](../diagrams/png/hw-2-01-memory-pyramid.png)
*Şekil 2.1 — Hız, kapasite ve maliyet piramidi. Yukarı çıktıkça hız ve bit başına maliyet
artar, kapasite düşer.*

## 2.1.2 "Memory wall" — büyüyen uçurum `[kavram]`

Asıl sorun uçurumun büyüklüğü değil, **büyüme hızı**.

Kabaca 1980'den 2005'e:
- CPU hızı yılda ~%50 arttı
- DRAM gecikmesi yılda ~%7 iyileşti

Sonuç: aradaki makas yıllar içinde açıldı. 1980'de bir DRAM erişimi birkaç CPU çevrimiydi;
bugün **yüzlerce**.

Bu olguya **"memory wall"** (bellek duvarı) denir.

**Neden DRAM gecikmesi iyileşmedi?** Çünkü DRAM'in yavaşlığı bir mühendislik tembelliği
değil, **fiziksel bir kısıt**: Faz 0.4.4'ten hatırla — DRAM okuması, bir kondansatörün
minicik yükünü ölçmek ve yükseltmektir. Bu analog bir işlemdir ve ölçeklenmesi zordur.

**İyileşen ne oldu?** **Bant genişliği** (throughput). DDR nesilleri gecikmeyi pek
düşürmedi ama saniyede aktarılan veri miktarını kat kat artırdı.

> **Ve bu, bu haritada üçüncü kez karşına çıkan ayrım:** *latency* ve *throughput* farklı
> şeylerdir ve genelde farklı yönlerde iyileşirler.
>
> - Faz 1.4.1'de: pipeline throughput'u artırdı, latency'yi değil
> - Burada: DDR nesilleri bant genişliğini artırdı, gecikmeyi değil
> - Faz 3.4'te: IOPS ve throughput ayrımı
> - Faz 5.3'te: bandwidth ve latency ayrımı
>
> **Bu ayrımı otomatikleştir.** Bir performans iddiası duyduğunda ilk sorun şu olsun:
> "Bu latency iddiası mı, throughput iddiası mı?"

## 2.1.3 Hiyerarşiyi mümkün kılan şey: yerellik `[mekanizma]`

Şimdi kritik soru: **Küçük bir hızlı bellek (cache) nasıl oluyor da işe yarıyor?**

Program bellekte rastgele yerlere erişseydi, 32 KiB'lık bir L1 cache 16 GiB'lık bir bellek
uzayında hiçbir işe yaramazdı — isabet olasılığı binde birden az olurdu.

**Ama programlar rastgele erişmez.** İki güçlü desen sergilerler:

### Zamansal yerellik (temporal locality)

> *Şimdi eriştiğin bir veriye, yakın gelecekte tekrar erişme olasılığın yüksektir.*

Örnekler: döngü sayacı, sık çağrılan bir fonksiyonun kodu, bir yapılandırma nesnesi,
sıcak bir veritabanı sayfası.

### Mekânsal yerellik (spatial locality)

> *Bir adrese eriştiysen, komşu adreslere de erişme olasılığın yüksektir.*

Örnekler: dizi elemanlarını sırayla gezmek, bir struct'ın alanlarını okumak, kodun sıralı
çalışması.

**Bu iki desen cache'i mümkün kılar.** Cache'in tüm tasarımı bu iki varsayıma dayanır:

| Desen | Cache'in cevabı |
|---|---|
| Zamansal yerellik | Veriyi kullandıktan sonra **tut** (hemen atma) |
| Mekânsal yerellik | Tek byte değil, **komşularıyla birlikte bir blok getir** |

İkinci maddeden **cache satırı** (cache line) kavramı doğar — ve bu, bu fazın en pratik
kavramıdır.

> **⚠️ Buradan çıkan en önemli sonuç ve bu haritanın anahtar cümlelerinden biri:**
>
> **Cache "hızlı bellek" değildir. Cache, programın davranışı hakkında yapılmış bir
> bahistir.** Programın yerellik sergilediğine bahse girer. Program bu bahsi tutuyorsa
> cache dâhice çalışır; tutmuyorsa cache neredeyse işe yaramaz.
>
> Bu yüzden **aynı donanımda, aynı işi yapan iki program arasında 10 kat performans farkı
> olabilir.** Fark donanımda değil, programın yerelliğe uyup uymadığındadır. Faz 2.3.6'da
> bunu somut kodla göreceğiz.

> **🤔 Düşün 2.1**
> Redis veya Memcached gibi bir cache katmanı koymak, veritabanı sorgularını hızlandırır.
> CDN koymak, statik içeriği hızlandırır. Tarayıcı cache'i sayfa yüklemesini hızlandırır.
>
> Bu üçü ile CPU'nun L1/L2/L3 cache'i arasında **kavramsal olarak** ne fark var?
> Ve üçünün de çalışması, hangi iki varsayıma dayanıyor?
> *(Cevap: fazın sonunda)*

---

# 2.2 Register — Hiyerarşinin Tepesi `[kavram]`

Faz 1.1.2'de detaylı işledik; burada sadece hiyerarşideki yerini konumlandırıyoruz.

| Özellik | Değer |
|---|---|
| Erişim süresi | Etkin olarak sıfır — aynı clock çevriminde |
| Kapasite | x86-64'te 16 × 8 byte = **128 byte** (genel amaçlı) |
| Yönetim | **Derleyici** yönetir, donanım değil |
| Fiziksel malzeme | Flip-flop dizisi (Faz 0.4.3) |

**Hiyerarşideki diğer her seviyeden ayıran tek özellik:** register'lar **yazılım tarafından
açıkça adreslenir.** Derleyici hangi değerin hangi register'da duracağına karar verir.
Cache seviyelerinin hiçbiri böyle değildir — onlar **donanım tarafından şeffaf biçimde**
yönetilir; program cache'in varlığını bile bilmeden çalışır.

> **Bu ayrım önemli.** "Cache'i nasıl programlarım?" diye bir şey yoktur. Cache'i doğrudan
> kontrol edemezsin — ama **erişim desenini değiştirerek** davranışını dramatik biçimde
> etkileyebilirsin. Faz 2.3.6'nın tamamı bu dolaylı kontrol üzerine.

**Register spilling.** 16 register yetmediğinde derleyici bazı değerleri belleğe (aslında
stack'e, yani pratikte L1 cache'e) taşır. Buna **spill** denir. Bu yüzden ARM64'ün 31
register'ı gerçek bir avantajdır: daha az spill, daha az bellek trafiği (Faz 1.6.2).

---
---

# 2.3 Cache — L1, L2, L3

**Bu bölüm fazın kalbidir ve haritanın en uzun bölümüdür.** Cloud'da açıklayamadığın
performans farklarının büyük kısmı burada gizlidir.

## 2.3.1 Neden tek bir cache değil de seviyeler? `[mekanizma]`

Faz 0.4.4'ten biliyoruz: SRAM hızlı ama pahalı ve yer kaplıyor. O hâlde neden tek bir orta
boy cache yapmıyoruz da üç seviye kuruyoruz?

Çünkü **SRAM'in hızı da boyutuyla ters orantılıdır.** Bu üç sebepten kaynaklanır:

1. **Fiziksel mesafe.** Faz 1.3.1'de hesapladık: 3 GHz'de bir çevrimde sinyal ~5–7 cm yol
   alır. Büyük bir cache çipte daha fazla yer kaplar, yani veri daha uzaktan gelir.
2. **Adres çözme.** Cache ne kadar büyükse, içinde doğru satırı bulmak için gereken
   karşılaştırma ve çoklayıcı ağacı o kadar derinleşir (Faz 1.1.2'deki register dosyası
   argümanının aynısı).
3. **Güç.** Büyük SRAM sürekli enerji yakar; her erişimde daha çok hat sürülür.

**Yani seçim şudur:** Ya küçük ve çok hızlı, ya büyük ve daha yavaş. İkisini birden
alamazsın.

**Çözüm: ikisini de al — ayrı seviyeler hâlinde.**

```
           küçük, çok hızlı, çekirdeğe yapışık
    L1  ←  32-48 KiB,  4-5 çevrim
     ↓
    L2  ←  512 KiB - 2 MiB,  12-20 çevrim
     ↓
    L3  ←  8-384 MiB,  40-120 çevrim,  TÜM çekirdekler paylaşır
     ↓
   DRAM ←  GB'lar,  240-300 çevrim
           büyük, yavaş, ucuz
```

Her seviye, bir alttakinin **cache'i** gibi davranır. Veri aranırken önce L1'e bakılır,
yoksa L2'ye, yoksa L3'e, yoksa DRAM'e gidilir — ve dönerken her seviyeye kopyası bırakılır.

## 2.3.2 L1 Cache `[mekanizma]`

| Özellik | Tipik değer |
|---|---|
| Boyut | 32–48 KiB **veri** + 32 KiB **talimat** (ayrı!) |
| Gecikme | 4–5 çevrim (~1 ns) |
| Sahiplik | **Her fiziksel çekirdeğin kendine ait** |
| SMT durumu | Aynı çekirdekteki iki thread **paylaşır** |

**Neden veri ve talimat cache'i ayrı?** (L1d ve L1i)

Çünkü Faz 1.2.1'deki pipeline'da `IF` (talimat getirme) ve `MEM` (veri erişimi) aşamaları
**aynı çevrimde** çalışır. Tek bir cache olsaydı her çevrim yapısal hazard (Faz 1.4.2)
yaşanırdı. Ayırmak bu çakışmayı ortadan kaldırır.

Bu tasarıma **Harvard mimarisi** denir — ama sadece cache seviyesinde. L2'den aşağısı
birleşiktir (unified), çünkü orada çakışma baskısı yok.

> **L1'in boyutu neden 20 yıldır neredeyse aynı?** 2005'te de ~32 KiB'dı, bugün de
> ~32–48 KiB. Sebep: L1'in 4 çevrimde erişilebilir kalması gerekiyor. Büyütürsen bu
> garantiyi kaybedersin. **L1'in boyutunu belirleyen kapasite ihtiyacı değil, gecikme
> bütçesidir.** Transistör bütçesi arttıkça L1'i büyütmek yerine L2 ve L3 büyütüldü.

## 2.3.3 L2 Cache `[mekanizma]`

| Özellik | Tipik değer |
|---|---|
| Boyut | 512 KiB – 2 MiB (sunucu CPU'larında üst uçta) |
| Gecikme | 12–20 çevrim (~4 ns) |
| Sahiplik | Genelde **her çekirdeğin kendine ait** (bazı eski tasarımlarda çift çekirdek paylaşırdı) |
| İçerik | Talimat + veri birlikte (unified) |

L2, L1'in kaçırdıklarını yakalayan ikinci savunma hattıdır. L1'den ~4 kat yavaş ama
~20–50 kat büyük.

**Sunucu CPU'larında L2 son yıllarda belirgin biçimde büyüdü** (Intel Sapphire Rapids'te
çekirdek başına 2 MiB). Sebep: çok çekirdekli sunucularda L3 üzerindeki yarışma arttıkça,
işi çekirdeğe yakın yerde bitirmek daha değerli hâle geldi.

## 2.3.4 L3 Cache — cloud'un ilgi merkezi `[mekanizma]`

| Özellik | Tipik değer |
|---|---|
| Boyut | 8–32 MiB (masaüstü) · **32–384 MiB (sunucu)** |
| Gecikme | 40–120 çevrim (~15–40 ns) |
| Sahiplik | **TÜM çekirdekler paylaşır** |
| Diğer adı | LLC (Last Level Cache) |

**L3'ün paylaşımlı olması, bu bölümün cloud açısından en önemli gerçeğidir.**

Paylaşımın iki yüzü var:

**Faydası:** Çekirdekler arası veri paylaşımı hızlanır. Bir çekirdeğin ürettiği veriyi
diğeri DRAM'e gitmeden L3'ten okuyabilir. Çok thread'li uygulamalarda bu büyük kazanç.

**Zararı:** **Yarışma.** L3 sabit boyutlu bir kaynaktır ve çekirdekler onu paylaşır. Bir
çekirdek büyük bir veri kümesini gezerse L3'ü kendi verisiyle doldurur ve **diğer
çekirdeklerin verisini dışarı atar (evict eder).**

> **Ve işte noisy neighbor'ın mekanizması budur.**
>
> Cloud'da "noisy neighbor" (gürültücü komşu) sık kullanılan ama nadiren açıklanan bir
> terimdir. Mekanizması tam olarak şu: **Aynı fiziksel sunucudaki başka bir kiracının
> instance'ı, senin instance'ınla aynı L3 cache'i paylaşır.** O kiracı bellek yoğun bir iş
> çalıştırdığında senin verin L3'ten atılır. Senin uygulaman kod değişmeden, yük değişmeden,
> **aniden yavaşlar.**
>
> Ve en sinsi tarafı: bunu **kendi metriklerinde göremezsin.** CPU kullanımın normal, bellek
> kullanımın normal, disk normal, ağ normal. Sadece gecikmen arttı. Faz 7.4'te bu konuyu
> tam olarak açacağız.

**Cloud bağlantısı `[uygulama]`:** Bu yüzden büyük L3'lü instance aileleri (örneğin `x` ve
`r` ailesinin bazı üyeleri, `c` ailesinin yeni nesilleri) çalışma kümesi cache'e sığan iş
yüklerinde orantısız iyi performans verir. Ve bu yüzden **bir instance'ı büyütmek bazen
beklenenden fazla hızlandırır**: sadece daha çok çekirdek değil, aynı zamanda çekirdek
başına daha çok L3 payı almış olursun.

> **🔧 Makinende gör** (isteğe bağlı)
>
> ```bash
> lscpu | grep -i cache
> ```
> **Beklenen çıktı (örnek):**
> ```
> L1d cache:    192 KiB (4 instances)   ← 4 çekirdek × 48 KiB
> L1i cache:    128 KiB (4 instances)
> L2 cache:     5 MiB   (4 instances)   ← çekirdek başına 1,25 MiB
> L3 cache:     12 MiB  (1 instance)    ← TEK instance = paylaşımlı
> ```
> `(1 instance)` ifadesi L3'ün paylaşımlı olduğunun doğrudan kanıtıdır.
>
> Daha detaylı görünüm:
> ```bash
> cat /sys/devices/system/cpu/cpu0/cache/index*/{level,type,size,shared_cpu_list} 2>/dev/null
> ```
> `shared_cpu_list` her cache seviyesini hangi CPU'ların paylaştığını gösterir — L1/L2 için
> genelde bir veya iki CPU (SMT çifti), L3 için hepsi.

## 2.3.5 Cache satırı (cache line) — 64 byte `[mekanizma]`

**Bu, bu fazın en pratik kavramıdır. Cloud engineer'ın bilmesi gereken tek bir cache
detayı seçilecek olsa, bu olurdu.**

CPU bellekten **tek byte okumaz.** En küçük transfer birimi bir **cache satırıdır** ve
x86-64 ile çoğu ARM64'te bu **64 byte**'tır.

```c
char veri[1000];
char x = veri[0];     // Tek bir byte istedin...
                      // ...ama CPU veri[0] ile veri[63] arasını getirdi.
```

**Neden?** İki sebep, ikisi de Faz 2.1.3'teki mekânsal yerelliğe dayanır:

1. **Komşu veriye erişme olasılığın yüksek** — bedavaya geleni al
2. **DRAM'den okuma zaten pahalı** — ~250 çevrimlik yolculuğu 1 byte için yapmak israf;
   aynı yolculukta 64 byte getirmek neredeyse aynı maliyettir (DRAM'in bant genişliği bol,
   gecikmesi pahalı — 2.4'te göreceğiz)

### Sonuç 1: Sıralı erişim, rastgele erişimden kat kat hızlıdır

```
Sıralı gezinti (dizi):
  veri[0] okundu  → cache miss → 64 byte geldi (veri[0..63])
  veri[1] okundu  → CACHE HIT  ✓
  veri[2] okundu  → CACHE HIT  ✓
  ...
  veri[63] okundu → CACHE HIT  ✓
  veri[64] okundu → cache miss → yeni satır

  → 64 erişimde 1 miss.  Hit oranı: %98,4

Rastgele gezinti (linked list, hash tablosu):
  Her erişim farklı bir satırda → her erişim miss
  → Hit oranı: ~%0
```

**Aynı sayıda veri okundu. Performans farkı 10–50 kat.**

### Sonuç 2: Veri yapısı seçimi bir performans kararıdır

| Yapı | Bellek düzeni | Cache davranışı |
|---|---|---|
| Dizi / `vector` / `slice` | Bitişik | **Mükemmel** — satır başına 64 byte faydalı |
| `struct` dizisi | Bitişik | İyi (tüm alanları kullanıyorsan) |
| Linked list | Dağınık, pointer'la bağlı | **Felaket** — her düğüm ayrı satır |
| Hash tablosu | Rastgele kova erişimi | Kötü (tasarımına bağlı) |
| Ağaç (BST) | Dağınık | Kötü |

> **Bu, "Big-O her şeyi anlatır" varsayımının çöktüğü yerdir.** Teoride O(n) olan dizi
> taraması ile O(n) olan linked list taraması aynıdır. Pratikte dizi taraması **10 kat
> daha hızlıdır**, çünkü cache satırını kullanır.
>
> Modern donanımda **algoritmanın karmaşıklığı kadar bellek erişim deseni de belirleyicidir.**
> Bu yüzden performans kritik kütüphaneler (veritabanı motorları, oyun motorları, sayısal
> hesap kütüphaneleri) veriyi mümkün olduğunca bitişik tutar.

### Sonuç 3: Hizalama (alignment) önemlidir

Bir veri yapısı cache satırı sınırını **aşarsa**, onu okumak iki satır getirir:

```
Satır sınırı:  |....64 byte....|....64 byte....|
Veri (8 byte):              [██|██]              ← iki satıra yayıldı!
                                                    2 cache erişimi
```

Derleyiciler bunu genelde otomatik yönetir. Ama manuel bellek düzeni kurduğunda veya
paylaşımlı veri yapıları tasarladığında farkında olmak gerekir — özellikle 2.3.8'deki
false sharing probleminde.

> **🤔 Düşün 2.2**
> Bir `struct` tanımlıyorsun:
> ```c
> struct Kullanici {
>     char  isim[56];   // 56 byte
>     int   id;         //  4 byte
>     int   yas;        //  4 byte
> };                    // toplam 64 byte
> ```
> Bu struct'lardan 1 milyon tanelik bir dizin var ve programın **sadece `id` alanlarını**
> taramak istiyor (örneğin bir arama için).
>
> a) Bu tarama sırasında cache'e ne kadar faydalı veri geliyor, ne kadar israf?
> b) Aynı veriyi cache dostu tutmak için yapıyı nasıl değiştirirdin?
> *(Cevap: fazın sonunda)*

## 2.3.6 Hit, miss ve yerelliğin somut kanıtı `[mekanizma]`

**Cache hit:** İstenen veri cache'te bulundu.
**Cache miss:** Bulunamadı, bir alt seviyeye gidildi.

### Miss tipleri (üç C)

| Tip | Sebep | Çözümü |
|---|---|---|
| **Compulsory (zorunlu)** | Veriye ilk kez erişiliyor, cache'te olması imkânsızdı | Prefetching (2.3.9) |
| **Capacity (kapasite)** | Çalışma kümesi cache'ten büyük | Çalışma kümesini küçült veya daha büyük cache |
| **Conflict (çakışma)** | Cache'te yer var ama o veri o sete sığmıyor | Erişim desenini/hizalamayı değiştir (2.3.7) |

### Hit oranının gerçek etkisi — ve sayı seni şaşırtacak

Ortalama erişim süresini hesaplayalım. L1 hit 1 ns, miss durumunda DRAM 100 ns diyelim:

```
Hit oranı %90:   0,90 × 1 ns + 0,10 × 100 ns = 10,9 ns
Hit oranı %95:   0,95 × 1 ns + 0,05 × 100 ns =  5,95 ns
Hit oranı %99:   0,99 × 1 ns + 0,01 × 100 ns =  1,99 ns
Hit oranı %99,9: 0,999 × 1 ns + 0,001 × 100 ns = 1,10 ns
```

**%90'dan %99'a çıkmak, ortalama erişimi 5,5 kat hızlandırıyor.**

> Faz 1.4.2'deki dal tahmini hesabıyla aynı matematik: **nadir ama pahalı olayların
> oranını azaltmak, orantısız kazanç verir.** %9'luk bir iyileşme %450 hızlanma
> getiriyor. Bu sezgiye aykırı ama cloud'da her yerde geçerli: p99 gecikmeyi düşürmek,
> ortalamayı düşürmekten çok daha değerlidir.

### Somut kanıt: aynı işi yapan iki döngü

İki boyutlu bir matrisi toplayalım. C'de matrisler **satır-öncelikli** (row-major) saklanır:
`m[0][0], m[0][1], m[0][2], ...` bellekte bitişiktir.

```c
#define N 4096
int m[N][N];   // 64 MiB

// A) Satır satır gez — bellekteki sırayla
long toplam = 0;
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        toplam += m[i][j];

// B) Sütun sütun gez — bellekte atlaya atlaya
long toplam = 0;
for (int j = 0; j < N; j++)
    for (int i = 0; i < N; i++)
        toplam += m[i][j];
```

**İki döngü tam olarak aynı 16.777.216 elemanı okuyor. Aynı sayıda toplama yapıyor.
Big-O aynı: O(N²).**

Ama:

```
A) Satır satır:
   m[i][0] okundu → miss → 64 byte geldi = 16 int (m[i][0..15])
   Sonraki 15 erişim HIT
   → 16 erişimde 1 miss

B) Sütun sütun:
   m[0][j] okundu → miss → 64 byte geldi (m[0][j..j+15])
   Sonraki erişim m[1][j] → 16 KB ötede → BAŞKA SATIR → miss
   → Her erişim miss
   Ve getirilen 64 byte'ın 60'ı hiç kullanılmadan atılıyor
```

**Ölçülen fark: tipik olarak 5–15 kat.** Aynı donanım, aynı algoritma, aynı veri.

![Şekil 2.2 — Satır-öncelikli ve sütun-öncelikli erişimin cache satırı üzerindeki etkisi](../diagrams/png/hw-2-02-row-vs-column.png)
*Şekil 2.2 — Solda satır taraması getirilen cache satırının tamamını kullanır; sağda sütun
taraması her satırdan yalnızca 4 byte kullanıp gerisini atar.*

> **🔧 Makinende gör** (isteğe bağlı — bu deneyi bir kez yapman şiddetle önerilir)
>
> ```bash
> cat > /tmp/cache_test.c <<'EOF'
> #include <stdio.h>
> #include <stdlib.h>
> #include <time.h>
> #define N 4096
> static int m[N][N];
> int main(void) {
>     struct timespec a, b;
>     long t = 0;
>     clock_gettime(CLOCK_MONOTONIC, &a);
>     for (int i = 0; i < N; i++) for (int j = 0; j < N; j++) t += m[i][j];
>     clock_gettime(CLOCK_MONOTONIC, &b);
>     printf("Satir sirali : %.3f s (toplam=%ld)\n",
>            (b.tv_sec-a.tv_sec)+(b.tv_nsec-a.tv_nsec)/1e9, t);
>     t = 0;
>     clock_gettime(CLOCK_MONOTONIC, &a);
>     for (int j = 0; j < N; j++) for (int i = 0; i < N; i++) t += m[i][j];
>     clock_gettime(CLOCK_MONOTONIC, &b);
>     printf("Sutun sirali : %.3f s (toplam=%ld)\n",
>            (b.tv_sec-a.tv_sec)+(b.tv_nsec-a.tv_nsec)/1e9, t);
>     return 0;
> }
> EOF
> gcc -O1 -o /tmp/cache_test /tmp/cache_test.c && /tmp/cache_test
> ```
> **Beklenen çıktı:** Sütun sıralı tarama, satır sıralı taramadan **5–15 kat yavaş.**
>
> *Not: `-O2` veya `-O3` ile derlersen derleyici döngüleri yeniden düzenleyip farkı
> kapatabilir — bu yüzden `-O1` kullanıyoruz. "Derleyici zaten hallediyor" demek her zaman
> doğru değildir; karmaşık gerçek kodda genelde halledemez.*
>
> Cache miss'leri doğrudan saymak istersen (bare metal veya PMU erişimi olan ortamda):
> ```bash
> perf stat -e cache-misses,cache-references /tmp/cache_test
> ```

**Cloud bağlantısı `[uygulama]`:** Bu 10 kat fark, hiçbir instance yükseltmesiyle elde
edemeyeceğin bir kazançtır. `c7i.large`'dan `c7i.16xlarge`'a çıkmak bile tek thread'li bir
işi 10 kat hızlandırmaz.

> **Ve buradan çıkan mimari ders:** Bir performans probleminde sıralama şu olmalı:
>
> 1. **Erişim deseni / algoritma** — bedava, 10–100 kat kazanç potansiyeli
> 2. **Yapılandırma** (thread sayısı, havuz boyutu, cache katmanı) — ucuz, 2–10 kat
> 3. **Instance yükseltmesi** — pahalı, tipik olarak 1,5–3 kat
>
> Sektörde çoğu zaman bu sıra **tersten** işletilir: önce instance büyütülür. Faz 7.5'te
> bu refleksi sistematik olarak kuracağız.

## 2.3.7 İlişkilendirme (associativity) `[kavram]`

Bu konu `[kavram]` düzeyinde — mekanizmasını bilmen yeterli, detayına girme.

**Soru:** Bellekten gelen bir cache satırı, cache'in **neresine** yerleşecek?

Üç yaklaşım:

| Tasarım | Nasıl | Artı / Eksi |
|---|---|---|
| **Direct-mapped** | Her adresin tek bir yeri var | Basit ve hızlı; ama çakışma çok |
| **Fully associative** | Herhangi bir yere koyulabilir | Çakışma yok; ama arama pahalı |
| **N-way set associative** | Adres bir "set"i belirler, set içinde N yer var | **Pratik denge — gerçek CPU'lar bunu kullanır** |

Modern CPU'larda tipik: L1 8-way, L2 8–16-way, L3 12–16-way.

**Ne zaman seni ilgilendirir?** Nadiren, ama bir durumda gerçekten:

**Cache thrashing.** Erişim desenin, hep aynı set'e düşen adreslerden oluşuyorsa (örneğin
2'nin kuvveti bir adım (stride) ile dizi geziyorsan), cache'te bol yer olduğu hâlde sürekli
conflict miss alırsın.

```c
// N = 4096 (2'nin kuvveti) → sütun taramasında adresler hep aynı sete düşebilir
// N = 4097 yaparak (padding) çakışma dağıtılır ve hız belirgin artabilir
```

Bu, yüksek performanslı sayısal kodda bilinen bir numaradır (ve `4096` yerine `4104` gibi
"garip" boyutlar görmenin sebebidir). Günlük cloud işinde karşına çıkmaz; **varlığını bil,
detayına girme.**

## 2.3.8 Cache tutarlılığı ve false sharing `[kavram]`

### Problem

Her çekirdeğin kendi L1'i var. Aynı bellek adresi iki farklı çekirdeğin L1'inde birden
bulunabilir. Biri değiştirirse diğerinin kopyası **eskir.**

Bu kabul edilemez — program açısından bellek tek ve tutarlı olmalıdır.

### Çözüm: MESI protokolü

Donanım, her cache satırına bir durum etiketi verir:

| Durum | Anlamı |
|---|---|
| **M**odified | Bu çekirdek değiştirdi, DRAM'deki kopya eski, sadece bende var |
| **E**xclusive | Sadece bende var, henüz değiştirmedim |
| **S**hared | Birden fazla çekirdekte var, hepsi aynı ve temiz |
| **I**nvalid | Bendeki kopya geçersiz, kullanılamaz |

Çekirdekler birbirlerinin cache işlemlerini dinler (snooping) veya merkezi bir dizin
üzerinden koordine olur. Bir çekirdek bir satırı yazmak istediğinde, diğerlerindeki
kopyaları **Invalid** yapar.

**Maliyet:** Bu koordinasyon trafiği ücretsiz değildir. Çok çekirdekli sistemlerde aynı
veriye yoğun yazma, **cache line ping-pong**'a yol açar: satır çekirdekler arasında
sürekli gidip gelir ve performans çöker.

### False sharing — en sinsi performans hatası

İki thread **farklı değişkenlere** yazıyor. Mantıksal olarak hiçbir paylaşım yok.
Ama o iki değişken **aynı 64 byte'lık cache satırında** duruyorsa, donanım bunu paylaşım
sanar.

```c
struct Sayaclar {
    long thread_a_sayac;   // offset 0
    long thread_b_sayac;   // offset 8   ← AYNI cache satırında!
};

// Thread A sürekli thread_a_sayac'ı artırıyor
// Thread B sürekli thread_b_sayac'ı artırıyor
// → Satır iki çekirdek arasında sürekli invalidate ediliyor
// → Her artırma bir cache coherence turu tetikliyor
// → Tek thread'li versiyondan DAHA YAVAŞ olabilir
```

**Çözüm: padding ile ayır.**

```c
struct Sayaclar {
    long thread_a_sayac;
    char _pad[56];         // satırın kalanını doldur (8 + 56 = 64)
    long thread_b_sayac;
};
// Artık iki değişken ayrı cache satırlarında → çakışma yok
```

> **Bu, "paralelleştirdim ama hızlanmadı" şikâyetinin en sık görülen ve en zor teşhis
> edilen sebeplerinden biridir.** Kod doğru, kilitler doğru, algoritma paralel — ama
> donanım seviyesinde thread'ler birbirini boğuyor.
>
> Yüksek performanslı kütüphanelerde `alignas(64)`, `__attribute__((aligned(64)))` veya
> Java'da `@Contended` gibi işaretler tam olarak bunun içindir. Gördüğünde artık ne
> olduğunu biliyorsun.

**Cloud bağlantısı:** Çekirdek sayısı arttıkça false sharing'in maliyeti **artar** (daha
çok çekirdek = daha çok invalidation trafiği). Yani 64 vCPU'lu bir instance'a geçmek,
false sharing'i olan bir uygulamayı **yavaşlatabilir.** "Daha büyük instance her zaman
daha hızlı" varsayımının çöktüğü yerlerden biri.

## 2.3.9 Prefetching `[kavram]`

Compulsory miss'lerin (ilk erişim) çaresi, veriyi **istenmeden önce** getirmektir.

**Donanım prefetcher'ı:** CPU, bellek erişim desenini izler. Düzenli bir adım (stride)
tespit ederse, bir sonraki satırları kendiliğinden getirmeye başlar.

- Sıralı tarama → prefetcher mükemmel çalışır, miss neredeyse sıfırlanır
- Sabit adımlı tarama (`i += 4`) → genelde yakalar
- Rastgele erişim (linked list, hash) → **yakalayamaz**

> **Bu, sıralı erişimin avantajını daha da büyütür.** 2.3.6'daki 10 katlık fark, sadece
> cache satırı kullanımından değil, **aynı zamanda prefetcher'ın bir desende çalışıp
> diğerinde çalışmamasından** kaynaklanır.
>
> Ve pointer'lı veri yapılarının (linked list, ağaç) neden modern donanımda beklenenden
> çok daha kötü performans gösterdiğini de bu açıklar: sadece cache miss almıyorlar,
> aynı zamanda prefetcher'ın da hiçbir yardımını alamıyorlar. "Pointer chasing" terimi
> bu durumu anlatır.

**Yazılım prefetch'i** de vardır (`__builtin_prefetch`) ama nadiren gerekir ve yanlış
kullanımı zarar verir. Farkındalık düzeyinde bil.

---
# 2.4 RAM / DRAM — ana bellek

## 2.4.1 DRAM nasıl çalışır `[kavram]`

Faz 0.4.4'te kurduk: bir DRAM hücresi **1 transistör + 1 kapasitör**dür. Kapasitörde yük
varsa 1, yoksa 0.

Kapasitörler **sızdırır.** Yük birkaç milisaniyede kaybolur. Bu yüzden DRAM'in sürekli
**tazelenmesi (refresh)** gerekir: her hücre saniyede on binlerce kez okunup geri yazılır.

Bunun üç sonucu var:

1. **Dinamik adı buradan gelir** — Dynamic RAM. SRAM ise Static: beslendiği sürece
   kendiliğinden korunur, tazeleme gerekmez.
2. **Refresh sırasında o bölge erişilemez.** Bu, DRAM gecikmesinin değişkenliğine katkıda
   bulunur.
3. **Güç kesilince her şey kaybolur.** DRAM **uçucudur (volatile).** Faz 3'teki depolamanın
   var olma sebebi budur.

## 2.4.2 DRAM'in organizasyonu ve gecikmenin kaynağı `[mekanizma]`

DRAM'e erişmek tek adımlı bir iş değildir. Katmanlı bir adresleme yapısı vardır:

```
DIMM (takılan çubuk)
 └─ Rank (çubuğun bir yüzü/grubu)
     └─ Bank (paralel çalışabilen bölüm)
         └─ Row (satır — binlerce hücre)
             └─ Column (sütun — istediğin veri)
```

Bir okuma şöyle ilerler:

1. **RAS** (Row Address Strobe) — satır seçilir ve tüm satır bir "row buffer"a açılır.
   Bu **yıkıcı** bir okumadır: hücreler boşalır, buffer'dan geri yazılır.
2. **CAS** (Column Address Strobe) — buffer'dan istenen sütun okunur.
3. **Precharge** — başka bir satıra geçmek için buffer kapatılır ve hatlar hazırlanır.

**Kritik gözlem:** Eğer bir sonraki erişim **aynı satırdaysa** sadece 2. adım yapılır —
buna **row hit** denir ve çok hızlıdır. Farklı satırdaysa 3+1+2 adımların hepsi gerekir —
**row miss**, kat kat yavaş.

> **Ve bu, cache satırının hikâyesini DRAM seviyesinde tekrarlar.** Sıralı erişim row
> hit üretir; rastgele erişim row miss üretir. Yani mekânsal yerelliğin ödülü **iki kez**
> ödenir: bir kez cache seviyesinde, bir kez DRAM seviyesinde.

**CAS Latency (CL)** — RAM ürün sayfalarında gördüğün `CL16`, `CL40` gibi sayılar bu 2.
adımın çevrim cinsinden süresidir. Ama dikkat: **çevrim süresi nesle göre değişir.**

```
DDR4-3200 CL16  → 16 × (1/1600 MHz) = 10,0 ns
DDR5-6000 CL40  → 40 × (1/3000 MHz) = 13,3 ns   ← daha yüksek CL, ama...
```

İşte DRAM'in en önemli gerçeği:

> **DDR nesilleri bant genişliğini katladı, gecikmeyi neredeyse hiç iyileştirmedi.**
>
> ```
> DDR3 (2007):  ~13 ns gecikme,  ~12 GB/s
> DDR4 (2014):  ~13 ns gecikme,  ~25 GB/s
> DDR5 (2020):  ~13 ns gecikme,  ~50 GB/s
> ```
>
> Bu, 2.1.2'deki **memory wall**'un sayısal kanıtıdır. Fizik gecikmeyi sınırlıyor
> (kapasitörün boşalma süresi, sinyalin yol alması); paralellik ise bant genişliğini
> artırmaya izin veriyor. **Bant genişliği satın alınabilir; gecikme alınamaz.**

## 2.4.3 Kanal (channel) ve bant genişliği `[uygulama]`

Gecikmeyi düşüremiyorsan, **paralellik** kur. DRAM'de bunun adı **kanaldır.**

Her bellek kanalı CPU ile RAM arasında ayrı bir yoldur. 2 kanal ≈ 2 kat bant genişliği.

```
Tek kanal DDR5-4800:   1 × 38,4 GB/s
Çift kanal:            2 × 38,4 = 76,8 GB/s
Sunucu (8-12 kanal):   8 × 38,4 = 307 GB/s  ← EPYC/Xeon sınıfı
```

**Formül:**
```
Bant genişliği = Transfer hızı (MT/s) × 8 byte × Kanal sayısı
Örnek: 4800 MT/s × 8 × 8 kanal = 307 GB/s
```

> **Neden sunucu CPU'ları bu kadar çok kanala sahip?** Çünkü 64 çekirdek aynı anda bellek
> istiyor. Faz 1.5'teki mantığın aynısı: çekirdek sayısı arttıkça **paylaşılan kaynak
> darboğaza dönüşür.** Kanal sayısı artmasa, 64 çekirdekli bir CPU bellek beklemekten
> hiçbir iş yapamazdı.
>
> **Çekirdek başına bant genişliği**, sunucu CPU'larını değerlendirirken toplam bant
> genişliğinden daha anlamlı bir ölçüdür.

**Cloud bağlantısı `[uygulama]`:** İşte bu yüzden AWS'te bir instance'ın **ağ ve EBS bant
genişliği boyutuyla orantılı artar** — daha büyük instance, fiziksel sunucunun daha büyük
bir dilimini alır, dolayısıyla bellek kanallarının da daha büyük payını alır. Küçük
instance'lar bellek bant genişliği açısından da kısıtlıdır, sadece vCPU açısından değil.

## 2.4.4 ECC bellek `[kavram]`

DRAM hücreleri kozmik ışın, elektriksel gürültü veya yaşlanma nedeniyle **kendiliğinden
bit çevirebilir** (bit flip). Bu nadir ama gerçektir: büyük ölçekte, sunucu başına yılda
birkaç kez ölçülmüştür.

**ECC (Error Correcting Code)** bellek fazladan bit tutarak:
- **Tek bit hatasını düzeltir** (sessizce, uygulamaya yansımadan)
- **Çift bit hatasını tespit eder** (düzeltemez, ama sistemi durdurur — sessiz bozulmaya
  izin vermez)

> **ECC'nin asıl değeri hatayı düzeltmesi değil, sessiz veri bozulmasını (silent data
> corruption) engellemesidir.** Bir bit sessizce çevrilirse, bozuk veri veritabanına
> yazılır, yedeklenir, çoğaltılır — ve haftalar sonra keşfedilir. ECC olmayan sistemde
> bu hatayı fark etmenin bir yolu yoktur.

**Cloud bağlantısı:** **Tüm ciddi bulut sağlayıcılarının sunucuları ECC bellek kullanır.**
Bu, "neden cloud instance'ı kendi masaüstü makinemden pahalı?" sorusunun cevaplarından
biridir. Sen ECC'yi seçmezsin, zaten alırsın.

> **🤔 Düşün 2.3**
> Bir veritabanı sunucusu seçiyorsun. İki seçenek var:
> - **A:** 32 vCPU, 64 GiB RAM, çift kanal bellek
> - **B:** 16 vCPU, 64 GiB RAM, sekiz kanal bellek
>
> İş yükün: büyük tabloları tarayan analitik sorgular (her sorgu GB'larca veriyi baştan
> sona okuyor). Hangisini seçersin ve neden?
> *(Cevap: fazın sonunda)*

---

# 2.5 Sanal bellek — her programın kendi evreni

**Bu bölüm, işletim sistemi ile donanımın en sıkı el sıkıştığı yerdir.** Linux
haritasındaki bellek yönetimi konularının tamamı buraya dayanır.

## 2.5.1 Problem: neden sanal bellek var? `[kavram]`

Programlar doğrudan fiziksel bellek adreslerini kullansaydı üç problem çıkardı:

**1. Güvenlik.** Program A, program B'nin belleğini okuyabilirdi. Aynı sunucuda çalışan
iki uygulama birbirinin verisini (ve şifresini) görürdü.

**2. Yalıtım ve çökme.** Bir programın hatalı pointer'ı başka bir programın verisini
bozardı. Tek bir bug tüm sistemi düşürürdü.

**3. Yerleştirme.** Programın nereye yükleneceği derleme zamanında bilinemez. Her program
"0x400000 adresinden başlıyorum" diyemezdi — çakışırlardı.

**Çözüm: her programa kendi sanal adres uzayını ver.**

```
Program A'nın gördüğü:          Program B'nin gördüğü:
  0x0000...                       0x0000...
  ...                             ...
  0x7fff...  (kendi evreni)       0x7fff...  (kendi evreni)

Her ikisi de "0x400000" adresini kullanıyor —
ama farklı fiziksel adreslere çevriliyorlar.
```

**Program fiziksel belleği hiç görmez.** Gördüğü her adres sanaldır ve donanım tarafından
çevrilir.

## 2.5.2 Sayfa (page) ve sayfa tablosu `[mekanizma]`

Çeviriyi byte byte yapmak imkânsızdır (tablo belleğin kendisinden büyük olurdu). Bunun
yerine bellek **sayfa** denen bloklara bölünür.

**Standart sayfa boyutu: 4 KiB.**

| Terim | Anlamı |
|---|---|
| **Page** | Sanal bellekteki 4 KiB'lık blok |
| **Page frame** | Fiziksel bellekteki 4 KiB'lık blok |
| **Page table** | Sanal sayfa → fiziksel frame eşlemesi |
| **MMU** | Memory Management Unit — çeviriyi yapan **donanım** birimi (CPU içinde) |

Çeviri şöyle çalışır:

```
Sanal adres (48 bit):
┌──────────────────────────┬────────────────┐
│  Sayfa numarası (36 bit) │ Offset (12 bit)│
└──────────────────────────┴────────────────┘
            ↓                        ↓
      sayfa tablosundan        aynen kopyalanır
      frame numarası bul       (4 KiB = 2^12)
            ↓                        ↓
┌──────────────────────────┬────────────────┐
│  Frame numarası          │ Offset         │
└──────────────────────────┴────────────────┘
Fiziksel adres
```

**Çok seviyeli sayfa tablosu.** 48-bit adres uzayı için düz bir tablo 512 GiB yer kaplardı
— her process için. Bu yüzden tablo **4 seviyeli bir ağaç** olarak kurulur (x86-64'te
PML4 → PDPT → PD → PT). Sadece gerçekten kullanılan dallar bellekte tutulur.

**Ama bunun bedeli var:** Bir çeviri için **4 bellek erişimi** gerekir. Yani her bellek
erişimi aslında 5 erişim (4 tablo + 1 veri) olurdu. Bu kabul edilemez.

## 2.5.3 TLB — çevirinin cache'i `[mekanizma]`

**TLB (Translation Lookaside Buffer)**, yakın zamanda yapılmış sanal→fiziksel çevirileri
saklayan küçük ve çok hızlı bir cache'tir.

| Özellik | Tipik değer |
|---|---|
| Giriş sayısı | 64–1536 (seviyelere göre) |
| TLB hit maliyeti | ~0 çevrim (pipeline'a gömülü) |
| TLB miss maliyeti | **10–100+ çevrim** (page walk) |

> **TLB'yi anlamanın en temiz yolu: TLB, sayfa tablosunun cache'idir.** Cache verinin
> cache'iyse, TLB adresin cache'idir. Aynı yerellik prensipleri geçerlidir.

**TLB kapsamı (TLB reach) — kritik hesap:**

```
1536 giriş × 4 KiB sayfa = 6 MiB
```

**Yani TLB, aynı anda yalnızca ~6 MiB'lık bir bellek bölgesini "ucuz" tutabilir.**

Uygulaman 64 GiB veri üzerinde geziniyorsa? Neredeyse her erişim TLB miss. Ve her TLB
miss, 4 seviyeli bir page walk demek — üstelik o tablo sayfaları da cache'te olmayabilir.

**İşte huge page'in sebebi budur.**

## 2.5.4 Huge page `[uygulama]`

Sayfa boyutunu büyüterek TLB kapsamını genişletiriz:

| Sayfa boyutu | 1536 girişle kapsam |
|---|---|
| 4 KiB (standart) | 6 MiB |
| **2 MiB (huge page)** | **3 GiB** — 512 kat artış |
| **1 GiB (gigantic page)** | **1,5 TiB** |

**Kazanç:**
- TLB miss oranı çöker
- Sayfa tablosu küçülür (daha az bellek, daha iyi cache kullanımı)
- Page walk seviyeleri azalır

**Bedel:**
- **İç parçalanma (internal fragmentation):** 2 MiB sayfa ayırıp 100 KiB kullanırsan
  1,9 MiB israf
- Bellek daha kaba taneli yönetilir; parçalanmış bellekte huge page bulmak zorlaşabilir

**Kimlerin işine yarar:**

| İş yükü | Fayda |
|---|---|
| Veritabanı (PostgreSQL, MySQL, Oracle) | **Yüksek** — büyük paylaşımlı buffer havuzu |
| JVM (büyük heap) | **Yüksek** |
| In-memory cache (Redis, Memcached) | Orta–yüksek |
| Bilimsel hesaplama, ML | Yüksek |
| Web sunucusu, küçük servisler | Düşük veya negatif |

> **🔧 Makinende gör** (isteğe bağlı)
> ```bash
> grep -i huge /proc/meminfo
> cat /sys/kernel/mm/transparent_hugepage/enabled
> ```
> **Beklenen çıktı:**
> ```
> AnonHugePages:    212992 kB      ← THP ile otomatik verilenler
> HugePages_Total:       0         ← elle ayrılanlar (explicit)
> Hugepagesize:       2048 kB      ← 2 MiB
>
> [always] madvise never           ← THP modu (köşeli parantez = aktif)
> ```

> **THP (Transparent Huge Pages) uyarısı — üretimde bilmen gereken bir tuzak.**
> Linux, huge page'leri otomatik vermeye çalışır (THP). Genelde iyidir, **ama bazı
> veritabanları için zararlıdır:** THP'nin arka plan birleştirme (defrag) işlemi
> öngörülemez gecikme sıçramalarına yol açar. MongoDB, Redis ve Oracle dokümantasyonları
> THP'nin kapatılmasını (`never` veya `madvise`) açıkça önerir.
>
> Bu, "otomatik optimizasyon her zaman iyidir" varsayımının çöktüğü klasik örnektir:
> **ortalama performansı artıran bir mekanizma, kuyruk gecikmesini (p99) bozabilir.**

## 2.5.5 Page fault — üç tipi ayırt et `[mekanizma]`

**Page fault**, bir sanal sayfanın fiziksel bellekte karşılığı yokken erişilmesidir.
CPU bir kesme (Faz 4.4) üretir, işletim sistemi devralır.

**Üç tipi vardır ve bunları karıştırmak yaygın bir hatadır:**

| Tip | Ne oluyor | Maliyet | Sağlıklı mı? |
|---|---|---|---|
| **Minor fault** | Sayfa bellekte var ama bu process'e eşlenmemiş (paylaşımlı kütüphane, page cache, yeni ayrılan sıfır sayfası) | **~1–5 μs** | ✅ Tamamen normal, sürekli olur |
| **Major fault** | Sayfa **diskten** okunmalı (dosya eşlemesi veya swap) | **~50 μs – 10 ms** | ⚠️ Az olmalı, çoksa problem |
| **Invalid fault** | Geçersiz adres → **segmentation fault** | — | ❌ Program hatası |

> **Bu ayrım, üretimde metrik okurken hayati önemdedir.** "Page fault sayısı yüksek"
> tek başına hiçbir şey ifade etmez — minor fault sayısının yüksek olması sağlıklı bir
> sistemin normal davranışıdır. **Bakılacak metrik major fault'tur.**

> **🔧 Makinende gör** (isteğe bağlı)
> ```bash
> ps -o min_flt,maj_flt,cmd -p $$
> ```
> **Beklenen çıktı:**
> ```
>  MINFL  MAJFL CMD
>   4821      0 bash        ← binlerce minor, sıfır major = sağlıklı
> ```
> Sistem geneli:
> ```bash
> vmstat 1 3
> ```
> `si` (swap in) ve `so` (swap out) sütunları sürekli sıfırdan büyükse → 2.6'daki problem.

---
# 2.6 Swap — üretimdeki en sinsi performans katili

## 2.6.1 Swap nedir ve neden var `[kavram]`

**Swap**, fiziksel bellek yetmediğinde işletim sisteminin bazı bellek sayfalarını **diske**
taşımasıdır. Böylece programlar, fiziksel RAM'den daha fazla bellek kullanabilir.

Kulağa harika geliyor. Ama 2.1.1'deki merdiveni hatırla:

```
DRAM erişimi:      ~80 ns
NVMe SSD erişimi:  ~80.000 ns     ← 1.000 kat yavaş
SATA SSD:         ~200.000 ns     ← 2.500 kat
HDD:           ~8.000.000 ns      ← 100.000 kat
```

**Swap'e düşen bir sayfaya erişmek, RAM'e erişmekten en az 1.000 kat pahalıdır.**

## 2.6.2 Thrashing — sistemin ölüm spirali `[mekanizma]`

Swap'in tehlikesi tek bir yavaş erişim değildir. Tehlikeli olan **geri besleme
döngüsüdür:**

```
  Bellek dolar
       ↓
  OS sayfaları swap'e yazar
       ↓
  Program o sayfaya tekrar erişir  ← çünkü aktif kullanımdaydı
       ↓
  Major page fault — diskten okunmalı
       ↓
  Yer açmak için BAŞKA bir sayfa swap'e yazılır
       ↓
  O sayfaya da erişilir
       ↓
  ... döngü hızlanarak devam eder
```

Buna **thrashing** denir. Sistemin belirtileri:

| Belirti | Görünüm |
|---|---|
| CPU kullanımı | **Düşük** (%5–20) — CPU sürekli disk bekliyor |
| I/O wait (`wa`) | **Çok yüksek** (%50–90) |
| Disk aktivitesi | Sürekli yoğun |
| Uygulama yanıt süresi | 10–1000 kat artmış |
| SSH ile bağlanma | Dakikalar sürebilir veya zaman aşımına uğrar |

> **Thrashing'in en tehlikeli tarafı: sistem çökmez.** Çökse, sağlık kontrolü (health
> check) fark eder, load balancer trafiği keser, otomatik ölçekleme yeni instance açar.
> Ama thrashing yapan sunucu **"ayakta" görünür** — ping'e cevap verir, port açıktır,
> süreç çalışıyordur. Sadece her şeyi 100 kat yavaş yapar.
>
> Ve yavaş bir sunucu, çoğu mimaride ölü bir sunucudan **daha zararlıdır**: istekleri
> kabul eder, kuyruk biriktirir, zaman aşımları yayılır, bağlı servisleri de düşürür.
> Bu, dağıtık sistemlerde **"gray failure"** (gri arıza) olarak bilinir.

## 2.6.3 OOM Killer `[mekanizma]`

Swap da dolarsa, Linux **OOM Killer** (Out Of Memory Killer) devreye girer ve bir process'i
**öldürür.**

Seçim, her process'e verilen `oom_score` ile yapılır. Puanı en çok bellek kullanan ve en
uzun süredir çalışan process'ler için yüksektir.

> **İronik sonuç:** OOM Killer genellikle **sunucudaki en önemli process'i** öldürür.
> Çünkü veritabanı veya uygulama sunucusu, tanımı gereği en çok belleği kullanan
> process'tir.

**Belirtisi:** Uygulaman "sebepsiz" yeniden başlıyor. Uygulama loglarında hiçbir hata yok
— çünkü uygulama kendi kapanmadı, `SIGKILL` ile öldürüldü; kapanış handler'ı bile
çalışmadı.

> **🔧 Makinende gör** (isteğe bağlı)
> ```bash
> # Swap durumu
> free -h
> # OOM Killer geçmişi
> dmesg -T | grep -i -E "out of memory|killed process"
> # Hangi process'ler swap kullanıyor
> for p in /proc/[0-9]*; do
>   s=$(awk '/^VmSwap/{print $2}' $p/status 2>/dev/null)
>   [ -n "$s" ] && [ "$s" -gt 0 ] && echo "$s kB  $(cat $p/comm 2>/dev/null)"
> done | sort -rn | head
> ```
> **Beklenen çıktı (sağlıklı sistem):**
> ```
>                total   used   free  shared  buff/cache  available
> Mem:            15Gi  4,2Gi  2,1Gi   412Mi       9,1Gi       10Gi
> Swap:          2,0Gi     0B  2,0Gi      ← used = 0 : sağlıklı
> ```
> `dmesg` çıktısı boşsa OOM olayı yaşanmamış demektir — bu iyi haberdir.

## 2.6.4 Swappiness `[uygulama]`

`vm.swappiness` (0–200, varsayılan 60) çekirdeğe şunu söyler: **bellek baskısı altında,
anonim sayfaları swap'e atmak ile page cache'i atmak arasında hangisini tercih edeyim?**

| Değer | Davranış | Uygun yer |
|---|---|---|
| 0 | Neredeyse hiç swap kullanma (OOM riski artar) | Agresif ayar, dikkatli kullan |
| 1–10 | Sadece son çare | **Veritabanı sunucuları** |
| 60 | Varsayılan denge | Genel amaçlı |
| 100+ | Agresif swap | Masaüstü, çok sayıda atıl uygulama |

```bash
cat /proc/sys/vm/swappiness        # oku
sudo sysctl -w vm.swappiness=10    # geçici ayarla
```

> **Yaygın yanlış anlama:** "swappiness=0 yaparsam swap hiç kullanılmaz." Hayır —
> bellek gerçekten biterse çekirdek yine de swap kullanır (veya OOM Killer'ı çağırır).
> Swappiness bir **tercih ağırlığıdır**, bir yasak değil.

## 2.6.5 Cloud'da swap: kasıtlı olarak yok `[uygulama]`

**Çoğu cloud instance'ı varsayılan olarak swap'siz gelir. Bu bir eksiklik değil, bilinçli
bir tasarım kararıdır.**

Mantık şudur:

| Swap'li | Swap'siz |
|---|---|
| Bellek biterse sistem **yavaşlar** | Bellek biterse process **ölür** |
| Gri arıza — tespiti zor | Net arıza — tespiti kolay |
| Sağlık kontrolü geçer, trafiği almaya devam eder | Sağlık kontrolü düşer, trafik kesilir |
| Saatlerce süren belirsiz yavaşlık | Saniyeler içinde yeni instance | 

> **Bu felsefeye "fail fast" denir ve dağıtık sistemlerin temel ilkelerindendir:**
> **Yavaş ve belirsiz bir arıza yerine hızlı ve net bir arıza tercih edilir** — çünkü
> otomasyon net arızaya tepki verebilir, belirsiz yavaşlığa veremez.
>
> Kubernetes bu prensibi daha da ileri götürür: swap'in kapalı olmasını uzun süre
> **zorunlu** tuttu (`--fail-swap-on`). Sebebi aynıdır: pod'un bellek limitinin anlamlı
> olabilmesi için, limite ulaşıldığında gerçekten bir şeyin olması gerekir.

**Pratik sonuç:** Cloud'da bellek yönetiminin cevabı "swap ekleyeyim" değildir. Cevap:
- Bellek kullanımını **ölç** ve **sınırla**
- Doğru boyutta instance seç (memory-optimized `r`/`x` aileleri)
- Bellek sızıntılarını bul, gizleme

> **🤔 Düşün 2.4**
> Bir uygulama sunucusunda bellek kullanımı yavaş yavaş artıyor (bellek sızıntısı var) ve
> 6 günde bir instance çöküyor. Ekipten biri "swap ekleyelim, çökmez" diyor.
>
> a) Swap eklenirse ne olur? Problem çözülür mü?
> b) Sen ne önerirsin ve neden?
> *(Cevap: fazın sonunda)*

---

# 2.7 NUMA — büyük instance'ların gizli vergisi

## 2.7.1 Problem `[kavram]`

Şimdiye kadar "bellek" tek ve homojen bir kaynak gibi konuştuk. Küçük sistemlerde doğru.
Büyük sunucularda **değil.**

Bir sunucuda 2 fiziksel CPU soketi ve 512 GiB RAM varsa, o RAM tek bir havuz değildir:
her soketin **kendi bellek denetleyicisi ve kendine bağlı RAM modülleri** vardır.

```
┌─────────────────┐        ┌─────────────────┐
│   SOKET 0       │◄──────►│   SOKET 1       │
│   32 çekirdek   │  UPI/  │   32 çekirdek   │
│   L3 cache      │Infinity│   L3 cache      │
└────────┬────────┘  Fabric└────────┬────────┘
         │                          │
    ┌────▼────┐                ┌────▼────┐
    │ 256 GiB │                │ 256 GiB │
    │ (yerel) │                │ (yerel) │
    └─────────┘                └─────────┘
```

**NUMA = Non-Uniform Memory Access** — bellek erişim süresi **tekdüze değildir.**

| Erişim | Gecikme | Bant genişliği |
|---|---|---|
| **Yerel** (kendi soketinin RAM'i) | ~80 ns | Tam |
| **Uzak** (diğer soketin RAM'i) | **~140 ns (1,5–2 kat)** | Düşük (soketler arası bağ sınırlı) |

## 2.7.2 Sonuçlar `[mekanizma]`

**Neden önemli:** Bir thread soket 0'da çalışıyor ama verisi soket 1'in RAM'indeyse,
**her bellek erişimi 2 kat pahalı.** Ve bu, uygulamaya hiçbir şekilde görünmez.

Daha kötüsü: Linux zamanlayıcısı thread'i soketler arasında **taşıyabilir**. Thread soket
0'da başlayıp belleğini oraya ayırır, sonra soket 1'e taşınır — artık tüm belleği uzaktır.

**Linux'un savunması:** İlk dokunma politikası (**first-touch**). Bir sayfa, ona **ilk
dokunan** thread'in soketine ayrılır — ayrıldığı yere değil, ilk yazıldığı yere.

> **Bu, çok bilinen bir tuzağı doğurur:** Program başlangıçta tek thread ile büyük bir
> tampon ayırıp sıfırlarsa, **tüm o bellek tek sokete düşer.** Sonra 64 thread onu
> paralel kullanmaya çalışır ve yarısı sürekli uzak erişim yapar.
>
> Doğrusu: tamponu **onu kullanacak thread'lere paralel olarak ilk dokundurmaktır.**

## 2.7.3 Cloud bağlantısı `[uygulama]`

**Ne zaman seni ilgilendirir?**

| Instance boyutu | NUMA durumu |
|---|---|
| Küçük/orta (`.large` – `.4xlarge`) | Tek NUMA düğümü — **ilgilendirmez** |
| Büyük (`.12xlarge` ve üstü, `metal`) | **Birden fazla NUMA düğümü — ilgilendirir** |

> **Ve buradan, bu fazın en pratik cloud dersi çıkar:**
>
> **Tek bir `.24xlarge` instance, iki `.12xlarge` instance'tan daha hızlı olmayabilir** —
> hatta NUMA'ya duyarsız bir uygulamada **daha yavaş olabilir.** Çünkü tek büyük instance
> NUMA sınırını içeride taşır ve uygulama bunu yönetmezse ceza öder; iki küçük instance
> ise her biri tek NUMA düğümünde kalır.
>
> "Dikey ölçekleme (scale up) her zaman daha basit ve hızlıdır" varsayımının donanım
> seviyesindeki karşı örneği budur.

> **🔧 Makinende gör** (isteğe bağlı)
> ```bash
> lscpu | grep -i numa
> numactl --hardware 2>/dev/null || echo "numactl kurulu degil"
> ```
> **Beklenen çıktı (tek soketli makine):**
> ```
> NUMA node(s):        1
> NUMA node0 CPU(s):   0-7        ← tek düğüm, NUMA problemi yok
> ```
> **Büyük sunucuda:**
> ```
> NUMA node(s):        2
> NUMA node0 CPU(s):   0-31,64-95
> NUMA node1 CPU(s):   32-63,96-127
>
> node distances:
> node   0   1
>   0:  10  21          ← uzak erişim ~2,1 kat maliyetli
>   1:  21  10
> ```
> `node distances` matrisi NUMA cezasının doğrudan ölçüsüdür: 10 = yerel referans,
> 21 = uzak.

**Çözüm araçları (farkındalık düzeyinde):**
```bash
numactl --cpunodebind=0 --membind=0 ./uygulama   # tek düğüme sabitle
```
Veritabanları ve JVM'ler genelde kendi NUMA farkındalıklarına sahiptir; ayarlarını
açmak yeterli olur.

---
# 2.8 Bu faz bozulunca — arıza imzaları

Haritanın "Bozulunca" ekseni. Her satır, üretimde göreceğin bir belirti ve arkasındaki
mekanizma.

| Belirti | Olası mekanizma | Nerede anlatıldı | İlk bakılacak |
|---|---|---|---|
| CPU %100 ama iş bitmiyor, IPC düşük | Cache miss / bellek bekleme | 2.1.2, 2.3.6 | Erişim deseni, çalışma kümesi boyutu |
| Kod değişmeden ara ara yavaşlama | Noisy neighbor — L3 yarışması | 2.3.4 | `steal time`, aynı iş için p99 sapması |
| Thread sayısını artırdım, hızlanmadı | False sharing veya bellek bant genişliği doygunluğu | 2.3.8, 2.4.3 | Paylaşımlı sayaç/yapılar, padding |
| Büyük instance'a geçtim, yavaşladı | NUMA uzak erişim veya artan coherence trafiği | 2.7, 2.3.8 | `numactl --hardware`, thread sabitleme |
| CPU %10, I/O wait %80, her şey donuk | **Thrashing** | 2.6.2 | `vmstat` si/so, `free -h` |
| Uygulama loglara hiçbir şey yazmadan öldü | **OOM Killer** | 2.6.3 | `dmesg -T \| grep -i "killed process"` |
| Veritabanında öngörülemez gecikme sıçraması | THP defrag | 2.5.4 | `transparent_hugepage/enabled` → `madvise` |
| Büyük veri setinde beklenmedik yavaşlık | TLB miss fırtınası | 2.5.3 | Huge page değerlendir |
| `free` az "free" gösteriyor, panik | Page cache — **normal** | 2.5.5 not | `available` sütununa bak, `free`'ye değil |

> **Son satır özellikle önemli.** `free -h` çıktısında `free` sütununun düşük olması bir
> problem değildir: Linux boş belleği page cache olarak kullanır, çünkü kullanılmayan RAM
> israf edilmiş RAM'dir. Baktığın sütun **`available`** olmalıdır — o, gerektiğinde
> uygulamalara verilebilecek belleği gösterir.

---

> **🤔 Düşün 2.5 — sentez sorusu**
> Bu fazda dört farklı "cache" gördün: CPU cache (2.3), TLB (2.3 / 2.5.3), DRAM row
> buffer (2.4.2) ve page cache (2.6, 2.8).
>
> Dördünün de ortak olan tek bir prensibi var. O nedir? Ve bu prensip **ne zaman
> çalışmaz?**
> *(Cevap: aşağıda)*

> **🤔 Düşün 2.6 — mimari kararı**
> Bir Redis cache sunucusu kuracaksın. Veri seti 50 GiB. İki seçenek:
> - **A:** `r7g.2xlarge` (8 vCPU, 64 GiB) — tek düğüm
> - **B:** 4 × `r7g.large` (2 vCPU, 16 GiB) — shard'lı küme, toplam 64 GiB
>
> Bu fazda öğrendiklerinle her iki seçeneğin **donanım açısından** artı ve eksilerini
> sırala. (Operasyonel karmaşıklığı bir kenara bırak, sadece bellek fiziği açısından bak.)
> *(Cevap: aşağıda)*

---

# Faz 2 — Düşün Sorularının Cevapları

## Cevap 2.1 — Redis/CDN/tarayıcı cache'i ile CPU cache'i

**Ortak prensip: hepsi 2.1.3'teki yerelliğe yapılmış bir bahistir.** Hepsi şunu varsayar:
*yakın zamanda erişilen veriye tekrar erişilecek (zamansal yerellik) ve komşu veriye
erişilecek (mekânsal yerellik).*

Ve hepsi aynı üç soruyu cevaplamak zorundadır:
1. **Neyi tutayım?** (yerleştirme politikası)
2. **Yer bitince neyi atayım?** (tahliye politikası — LRU, LFU, rastgele)
3. **Alttaki veri değişirse ne olacak?** (tutarlılık / invalidation)

**Ama kritik farklar var:**

| Boyut | CPU cache | Redis / CDN |
|---|---|---|
| Kim yönetiyor | **Donanım** — yazılıma görünmez | **Sen** — açıkça kod yazıyorsun |
| Tutarlılık | Donanım garanti ediyor (MESI, 2.3.8) | **Senin sorunun** — invalidation stratejisi yazman lazım |
| Granülerlik | 64 byte, sabit | Anahtar bazlı, değişken |
| Miss maliyeti | ~100 ns | ~1–100 ms (10.000–1.000.000 kat) |
| Hata modu | Sadece yavaşlama | **Bayat veri servis etme** |

**En önemli fark:** CPU cache yanlış sonuç veremez — donanım tutarlılığı garanti eder.
Uygulama cache'i **verebilir**, ve yazılım mühendisliğinin en meşhur zor problemlerinden
biri (cache invalidation) buradan doğar.

> **Buradan çıkan içgörü:** Cache bir mimari desen değil, **bilgisayar biliminin her
> katmanında tekrar eden temel bir yanıttır** — hız farkı olan iki seviye varsa, araya
> cache girer. CPU↔DRAM, DRAM↔disk, disk↔ağ, uygulama↔veritabanı, tarayıcı↔sunucu.
> Aynı problem, aynı çözüm, farklı ölçek.

## Cevap 2.2 — 64 byte'lık struct ve yalnızca `id` taraması

**a) Ne kadar israf?**

Struct tam 64 byte, yani **tam bir cache satırı.** Sadece `id` alanını (4 byte) okuyorsun.

```
Her kullanıcı için:
  Getirilen : 64 byte (tam bir cache satırı)
  Kullanılan:  4 byte (id)
  İsraf     : 60 byte

Faydalı kullanım oranı: %6,25
```

1 milyon kullanıcı için:
```
Taşınan veri : 64 MB
Gereken veri :  4 MB
16 kat fazla bellek trafiği, 16 kat fazla cache satırı işgali
```

Ayrıca: 1 milyon × 64 byte = 64 MB, tipik bir L3 cache'ten (32 MB) büyük. Yani tarama
sırasında **tüm L3 süpürülür** — programın diğer verileri de dışarı atılır (cache
pollution).

**b) Nasıl düzeltilir? — SoA dönüşümü**

Veriyi **Array of Structs (AoS)** yerine **Struct of Arrays (SoA)** olarak düzenle:

```c
// ÖNCE — AoS (Array of Structs)
struct Kullanici { char isim[56]; int id; int yas; };
struct Kullanici kullanicilar[1000000];
// Bellekte: [isim0 id0 yas0][isim1 id1 yas1][isim2 id2 yas2]...

// SONRA — SoA (Struct of Arrays)
struct Kullanicilar {
    char isim[1000000][56];
    int  id[1000000];        ← hepsi bitişik!
    int  yas[1000000];
};
// Bellekte: [isim0 isim1 ...][id0 id1 id2 ...][yas0 yas1 ...]
```

Artık `id` taraması:
```
Bir cache satırı = 64 byte = 16 adet id
Faydalı kullanım oranı: %100     ← 16 kat iyileşme
Taranan toplam veri: 4 MB (64 MB yerine)  ← L3'e rahat sığar
```

Üstelik prefetcher (2.3.9) mükemmel çalışır ve SIMD talimatları da kullanılabilir hâle
gelir.

> **Bu dönüşüm, yüksek performanslı sistemlerde standart bir tekniktir.** Sütunlu
> veritabanlarının (ClickHouse, Parquet, Redshift) analitik sorgularda satır tabanlı
> veritabanlarını neden 10–100 kat geçtiğinin **aynı sebebi** budur: sütunlu depolama,
> disk ve bellek seviyesinde SoA'dır.
>
> Faz 0'dan beri kurduğumuz zincirin güzel bir örneği: *cache satırı* gibi bir donanım
> detayı, *hangi veritabanını seçmen gerektiği* gibi bir mimari karara kadar uzanıyor.

## Cevap 2.3 — 32 vCPU/çift kanal mı, 16 vCPU/sekiz kanal mı?

**Cevap: B (16 vCPU, sekiz kanal).**

**Gerekçe:**

İş yükü tanımı kritik: *"büyük tabloları baştan sona tarayan analitik sorgular."* Bu,
tanımı gereği **bellek bant genişliği sınırlı (memory-bandwidth-bound)** bir iş yüküdür:

- Veri seti cache'e sığmıyor → 2.3'teki cache neredeyse işe yaramıyor
- Erişim sıralı → her cache satırı tam kullanılıyor, prefetcher çalışıyor
- Yani her çekirdek **sürekli DRAM'den veri çekiyor**

Bant genişliğini hesaplayalım (DDR5-4800 varsayımıyla):

```
A: 2 kanal × 38,4 GB/s = 76,8 GB/s  ÷ 32 vCPU = 2,4 GB/s per vCPU
B: 8 kanal × 38,4 GB/s = 307 GB/s   ÷ 16 vCPU = 19,2 GB/s per vCPU
                                                 ↑ 8 kat fazla
```

**A seçeneğinde 32 çekirdek, 76,8 GB/s'lik bir boruyu paylaşmak için yarışır.** Çekirdek
sayısını ikiye katlamak burada hiçbir şey kazandırmaz — darboğaz çekirdek değil, borudur.
Faz 1.5.4'teki mantığın bellek versiyonu: **kaynağı artırmak, darboğaz orada değilse
işe yaramaz.**

**Bunu ne zaman tersine çevirirsin?** İş yükü değişirse:
- Çalışma kümesi L3'e sığıyorsa → bant genişliği önemsizleşir, A daha iyi
- İş CPU yoğunsa (sıkıştırma, şifreleme, hesaplama) → A daha iyi
- Çok sayıda küçük eşzamanlı sorgu varsa (OLTP) → A daha iyi

> **Genel ders:** "Kaç vCPU?" sorusu tek başına eksiktir. Doğru soru: **"Bu iş yükünün
> darboğazı hangi kaynak?"** Cloud'da instance ailesi seçimi (`c` = compute, `r` =
> memory, `m` = dengeli, `i` = storage) tam olarak bu soruya verilen cevaptır. Faz 7.2'de
> bunu sistematikleştireceğiz.

## Cevap 2.4 — Bellek sızıntısına swap çözüm mü?

**a) Swap eklenirse ne olur?**

**Problem çözülmez, daha kötü bir hâle dönüşür.**

Sızıntı tanımı gereği **büyümeye devam eder.** Swap sadece çöküşü geciktirir:

```
Swap'siz:              Swap'li:
  6 gün normal çalışma   6 gün normal çalışma
  → çöküş (net)          → swap kullanımı başlar
  → yeniden başlar       → 1-3 gün THRASHING (2.6.2)
  → normale döner        → uygulama 100 kat yavaş ama "ayakta"
                         → sonunda swap de dolar
                         → OOM Killer (2.6.3) veya çöküş
```

**Takas ettiğin şey:** 6 günde bir *kısa ve net* bir kesinti yerine, *günlerce süren
belirsiz bir yavaşlık.* 2.6.5'teki tabloya göre bu **kötü bir takas**:

- Sağlık kontrolü thrashing'i yakalayamaz → bozuk instance trafiği almaya devam eder
- Zaman aşımları yukarı yayılır → bağlı servisler de bozulur
- Teşhis zorlaşır: çöküş `dmesg`'te görünür, thrashing sadece "her şey yavaş" olarak

**b) Ne önerirsin?**

Üç katmanlı bir cevap — ve sırası önemli:

**1. Hemen (bant yardımı, saatler):**
- Bellek kullanımına **alarm** koy (%80 eşiği) — çöküşü sürpriz olmaktan çıkar
- Otomatik yeniden başlatmayı **kontrollü** hâle getir: yükün düşük olduğu saatte,
  rolling restart ile, trafiği önce çekerek. Çöküş yaşanacaksa en azından sen seç.
- Process'e bellek limiti koy (systemd `MemoryMax=`, container limiti) → OOM Killer'ın
  rastgele kurban seçmesini engelle, sadece bu process ölsün

**2. Asıl iş (günler):**
- **Sızıntıyı bul.** Bellek profilleyici kullan (JVM: heap dump + MAT; Python:
  `tracemalloc`; Go: `pprof`; C/C++: `valgrind`, ASan)
- Sızıntı grafiği lineer mi, kademeli mi? Yükle orantılı mı? Bu, hangi kod yolunda
  olduğuna dair ilk ipucudur

**3. Yapmayacağın şey:**
- Swap eklemek
- Instance'ı büyütmek — bu da sadece geciktirir (6 gün yerine 12 gün), sızıntı devam eder

> **Buradaki genel ilke:** Swap, bellek sızıntısının **semptomunu gizler, hastalığı
> tedavi etmez.** Ve gizlenmiş bir semptom, açık bir semptomdan daha tehlikelidir —
> çünkü ne sen ne de otomasyonun ona tepki verebilir.
>
> Bu, Faz 7'de tekrar karşımıza çıkacak bir düşünce biçimi: **gözlemlenebilirliği
> azaltan hiçbir "çözüm" gerçek bir çözüm değildir.**

## Cevap 2.5 — Dört cache'in ortak prensibi ve ne zaman çalışmadığı

**Ortak prensip: yerellik bahsi.**

| Cache | Neyin cache'i | Bahsi |
|---|---|---|
| CPU cache | DRAM verisi | Yakın zamanda kullanılan veri tekrar kullanılacak |
| TLB | Sayfa tablosu girişleri | Yakın zamanda çevrilen adres tekrar çevrilecek |
| DRAM row buffer | Aktif DRAM satırı | Bir sonraki erişim aynı satırda olacak |
| Page cache | Disk blokları | Okunan dosya bloğu tekrar okunacak |

Dördü de aynı yapıyı paylaşır: **hızlı ve küçük bir katman + yavaş ve büyük bir katman +
yerelliğe dayalı bir tahmin.**

**Ne zaman çalışmaz? — Dört senaryo:**

**1. Erişim gerçekten rastgele olduğunda.** Yerellik yoksa tahmin tutmaz. Büyük bir hash
tablosunda rastgele arama, kriptografik anahtar üretimi, pointer chasing (2.3.9).

**2. Çalışma kümesi cache'ten biraz büyük olduğunda — ve bu en can sıkıcısıdır.**
Veri seti cache'ten biraz büyükse, döngüsel bir tarama her turda her şeyi tahliye eder:
her erişimden önce o veri **tam da atılmış olur.** Hit oranı %0'a yaklaşır — cache hiç
olmamasından **daha kötüdür**, çünkü tahliye trafiğini de ödersin.

> Bunun adı **LRU'nun patolojik durumudur** ve gerçek sistemlerde sık görülür: veri seti
> 40 MB, L3 32 MB → performans, veri seti 200 MB olduğundakinden farksız. Ama veri setini
> 30 MB'a düşürebilirsen **10 kat hızlanır.** Performans, veri boyutuna doğrusal değil
> **basamaklı** tepki verir; basamaklar cache sınırlarındadır.

**3. Streaming erişimde.** Veriyi bir kez okuyup bir daha dokunmuyorsan (log işleme,
yedekleme, ETL), cache sadece yer işgal eder ve **faydalı veriyi dışarı atar.** Bu yüzden
bazı sistemler "bu veriyi cache'leme" ipuçları sunar (`O_DIRECT`, `posix_fadvise`,
non-temporal store talimatları).

**4. Yazma ağırlıklı ve paylaşımlı erişimde.** 2.3.8'deki coherence trafiği, cache'i
kazançtan zarara çevirebilir.

> **Ve bu, cache hakkında bilmen gereken en olgun cümledir:** Cache bir garanti değil,
> **bir bahistir.** Bahis tuttuğunda 100 kat kazanırsın, tutmadığında sadece kaybetmezsin
> — bazen ekstra maliyet de ödersin. Bu yüzden "cache ekleyelim" hiçbir zaman düşünmeden
> verilecek bir karar değildir; önce **erişim deseninin** ne olduğunu sormak gerekir.

## Cevap 2.6 — Tek büyük Redis mi, dört küçük shard mı?

Sadece bellek fiziği açısından:

**A — Tek `r7g.2xlarge` (8 vCPU, 64 GiB)**

*Artıları:*
- **Ağ yok.** Shard'lar arası iletişim maliyeti sıfır. 2.1.1'deki merdivende aynı-AZ ağ
  gecikmesi (~0,5 ms) DRAM'den (~80 ns) **6.000 kat** pahalı — bu tasarrufun büyüklüğünü
  hafife alma.
- Tek NUMA düğümü (2.7.3 tablosu: `.2xlarge` küçük sınıfta) → uzak erişim cezası yok
- Çapraz-shard sorgu problemi yok

*Eksileri:*
- 50 GiB veri, 64 GiB RAM'in **%78'i.** İşletim sistemi, Redis'in kendi ek yükü ve
  fragmantasyon payı ile bu **dar bir marj.** Ve 2.6.5'e göre cloud'da swap yok → bellek
  biterse OOM Killer.
- Tek bellek denetleyicisi seti → bant genişliği bir instance'ın payıyla sınırlı
- Redis büyük ölçüde tek thread'li; 8 vCPU'nun çoğu atıl kalır

**B — 4 × `r7g.large` (her biri 2 vCPU, 16 GiB)**

*Artıları:*
- Her shard ~12,5 GiB veri tutar, 16 GiB'ın %78'i — ama **hata alanı 4'e bölünmüş.**
  Bir shard şişerse sadece o düşer.
- **Toplam bellek bant genişliği 4 kat** (4 ayrı fiziksel sunucu, 4 ayrı bellek
  denetleyicisi seti — 2.4.3)
- 4 kat toplam ağ bant genişliği (her instance kendi NIC payını alır — Faz 5)
- Her shard'ın çalışma kümesi küçük → L3 hit oranı **daha yüksek** (2.3.4). Cevap 2.5'teki
  basamak etkisi burada **lehine** çalışır.
- Yatay ölçeklenebilir: 5. shard eklenebilir

*Eksileri:*
- Her istek bir ağ turu daha ekler (istemci → doğru shard)
- Çapraz-shard işlemler (çoklu anahtar, transaction) zor veya imkânsız
- Sıcak anahtar (hot key) tek bir shard'ı boğabilir — dengesiz yük

**Donanım açısından karar:**

| Kullanım deseni | Tercih |
|---|---|
| Küçük, bağımsız anahtar erişimleri (tipik cache) | **B** — bant genişliği ve hata izolasyonu kazanır |
| Çoklu anahtar işlemleri, Lua script, transaction | **A** — ağ turu ve dağıtık koordinasyon cezasından kaçın |
| Veri 50→60 GiB büyüyecekse | **B** — A'da marj yok |
| Gecikmenin p99'u kritikse | **B** — küçük çalışma kümesi, daha iyi cache davranışı |

> **Bu sorunun asıl dersi şudur:** "Daha büyük instance mi, daha çok instance mı?"
> sorusunun cevabı bir tercih meselesi değildir — **iş yükünün bellek erişim desenine
> bağlıdır.** Ve bu deseni anlamak için tam olarak bu fazda öğrendiklerin gerekir.
>
> Faz 7.3 ve Faz 7.5'te bu kararı sistematik bir çerçeveye oturtacağız.

---
# Faz 2 — Sık Sorulan Sorular

> **S1: Daha çok RAM almak her zaman performansı artırır mı?**
>
> Hayır — ve bu, bu fazın en önemli düzeltmelerinden biridir. RAM miktarı bir **kapasite**
> ölçüsüdür, hız ölçüsü değil. Uygulaman zaten belleğe sığıyorsa ve swap kullanmıyorsa,
> RAM eklemek **hiçbir şey** kazandırmaz.
>
> RAM eklemek şu durumlarda işe yarar:
> - Swap kullanıyorsun (2.6) → devasa kazanç
> - Page cache yetersiz, disk okuma çok (Faz 3) → iyi kazanç
> - Uygulama bellek limiti yüzünden veri setini bölerek işliyor → iyi kazanç
>
> Hızı belirleyen şeyler ise: cache hit oranı (2.3), bellek bant genişliği (2.4.3),
> erişim deseni (2.3.6) ve NUMA yerleşimi (2.7). Bunların hiçbiri RAM miktarıyla
> doğrudan ilgili değil.

> **S2: `free -h` "sadece 2 GB boş" diyor, 16 GB RAM'im var. Bellek sızıntısı mı var?**
>
> Hayır, büyük ihtimalle sistem **doğru** çalışıyor. Linux boş belleği page cache olarak
> kullanır — okunan dosyalar bellekte tutulur. Uygulama bellek isterse bu cache anında
> boşaltılır.
>
> Bakman gereken sütun **`available`**dır:
> ```
>               total   used   free  shared  buff/cache  available
> Mem:           15Gi  4,2Gi  2,1Gi   412Mi       9,1Gi       10Gi
>                                                              ↑ bak
> ```
> `free` = 2,1 GiB ama `available` = 10 GiB. Sistem sağlıklı.
>
> **"Kullanılmayan RAM, israf edilmiş RAM'dir"** Linux'un bilinçli tasarım felsefesidir.

> **S3: Cache'i elle kontrol edebilir miyim? Verimi L1'e sabitleyebilir miyim?**
>
> Hayır. CPU cache **yazılıma tamamen şeffaftır** — ne içinde ne olduğunu görebilir ne de
> ne gireceğini seçebilirsin (2.2'deki register ile temel farkı budur).
>
> Cache'i **dolaylı olarak** etkilersin:
> - Erişim desenini değiştirerek (2.3.6) — en güçlü kaldıraç
> - Veri yapısını sıkıştırarak (Cevap 2.2'deki SoA)
> - Çalışma kümesini cache boyutunun altına indirerek (Cevap 2.5'teki basamak etkisi)
> - Prefetch ipucu vererek (`__builtin_prefetch`) — nadiren gerekli
> - Cache bypass ederek (non-temporal store) — streaming iş yükleri için
>
> Bazı sunucu CPU'larında **cache partitioning** (Intel CAT) vardır ve L3'ün bir dilimini
> bir uygulamaya ayırabilir. Bulut sağlayıcıları bunu kiracıya açmaz; kendi noisy neighbor
> kontrollerinde kullanırlar.

> **S4: L3 cache boyutu instance seçerken bakmam gereken bir şey mi?**
>
> Evet, ama dolaylı olarak — çünkü AWS bunu ilan etmez. Yine de iki pratik sonucu var:
>
> 1. **Aynı ailede daha büyük instance = daha büyük L3 payı.** `c7i.8xlarge`,
>    `c7i.4xlarge`'ın iki katı vCPU verir ama L3 payı da artar. Bu, ölçeklemenin bazen
>    doğrusaldan **iyi** çıkmasının sebebidir.
> 2. **Nesil atlaması L3'ü büyütür.** c6i → c7i geçişinde saat hızı aynı kalsa bile cache
>    ve bellek alt sistemi iyileştiği için gerçek kazanç olur (Faz 1.6.4'teki Graviton
>    tablosuna benzer mantık).
>
> Kesin sayı için CPU modelini öğrenip (`lscpu`) üreticinin sayfasına bakman gerekir.

> **S5: Swap tamamen kötü müdür? Hiç kullanmamalı mıyım?**
>
> Bağlama göre değişir:
>
> | Ortam | Öneri |
> |---|---|
> | Cloud sunucusu, üretim | **Swap yok** — fail fast (2.6.5) |
> | Kubernetes düğümü | **Swap yok** — limitlerin anlamlı olması için |
> | Masaüstü / geliştirme makinesi | **Swap var** — atıl uygulamalar diske gidebilir, sorun değil |
> | Bellek yoğun toplu iş (batch), gecikme önemsiz | Swap makul olabilir |
>
> Ayrım şu: **gecikmenin önemli olduğu her yerde swap zararlıdır.** Toplu bir gece işinde
> 2 kat yavaşlık kabul edilebilir; API sunucusunda edilemez.

> **S6: Huge page'i her yerde açmalı mıyım? Bedava kazanç gibi görünüyor.**
>
> Hayır. Kazanç **TLB baskısı olan iş yüklerine** özgüdür: büyük ve yoğun erişilen bellek
> bölgeleri (veritabanı buffer havuzu, büyük JVM heap'i, bilimsel hesaplama).
>
> Zararları:
> - **İç parçalanma** — küçük tahsislerde bellek israfı
> - **THP defrag gecikmesi** — 2.5.4'teki uyarı; öngörülemez duraklamalar
> - Parçalanmış bellekte huge page bulmak zorlaşır, bu da gecikme yaratır
>
> **Pratik kural:** Varsayılanı (`madvise`) bırak. Uygulama dokümantasyonu açıkça
> istiyorsa (PostgreSQL, Oracle, büyük JVM) elle huge page ayır. MongoDB/Redis gibi
> THP'yi açıkça reddeden uygulamalarda `never` yap. **Ölçmeden değiştirme.**

> **S7: Bu fazda öğrendiklerim yazılımcı işi gibi görünüyor. Cloud engineer olarak
> gerçekten lazım mı?**
>
> Evet — ama farklı bir sebeple. Sen bu optimizasyonları **yapmayacaksın**; onları
> **tanıyacaksın.**
>
> Cloud engineer olarak karşına çıkacak gerçek sorular:
> - "Uygulama yavaş, instance büyütelim mi?" → **Darboğaz nerede?** (Cevap 2.3)
> - "Thread sayısını artırdık, hızlanmadı, neden?" → false sharing veya bant genişliği
> - "Aynı kod bazen yavaş çalışıyor" → noisy neighbor (2.3.4)
> - "Sunucu ayakta ama her şey donuk" → thrashing (2.6.2)
> - "Uygulama sessizce ölüyor" → OOM Killer (2.6.3)
> - "Tek büyük mü, çok küçük mü?" → Cevap 2.6
>
> Bu soruların hiçbirine kod yazarak cevap vermiyorsun. Ama **mekanizmayı bilmeden doğru
> cevabı veremezsin** — ve bu, "instance büyüt" refleksiyle çalışan mühendis ile darboğazı
> bulan mühendis arasındaki farktır.

---

# Faz 2 — Kendini Sına

Cevaplarını yazmadan önce bölümlere dönme. Zorlandığın soruların numarası, tekrar etmen
gereken bölümü gösterir.

**Bölüm A — Temel (1–8)**

1. Bellek hiyerarşisi neden var? Tek cümleyle temel sebebi söyle.
2. Bir cache satırı kaç byte'tır ve bu sayı neden önemlidir?
3. L1, L2 ve L3 arasındaki üç temel farkı say (boyut dışında).
4. Zamansal ve mekânsal yerellik arasındaki farkı bir örnekle açıkla.
5. Neden L1 cache veri ve talimat olarak ikiye ayrılmıştır?
6. DRAM neden sürekli tazelenmek zorundadır? SRAM neden değildir?
7. Sanal bellek hangi üç problemi çözer?
8. Minor page fault ile major page fault arasındaki fark nedir? Hangisi problem işaretidir?

**Bölüm B — Mekanizma (9–15)**

9. Bir dizi taraması ile bir linked list taraması neden 10 kat farklı performans verir?
   Üç ayrı mekanizma say.
10. False sharing nedir ve neden "sinsi" olarak nitelenir?
11. TLB nedir, neyin cache'idir ve neden huge page onun performansını artırır?
12. Thrashing'in geri besleme döngüsünü adım adım anlat.
13. MESI protokolü hangi problemi çözer? Bedeli nedir?
14. NUMA nedir ve neden küçük instance'larda önemli değildir?
15. DDR nesilleri arasında bant genişliği katlanırken gecikme neden neredeyse sabit
    kaldı?

**Bölüm C — Uygulama ve muhakeme (16–22)**

16. Bir sunucuda CPU %100 görünüyor ama iş çıktısı beklenenin dörtte biri. Bu fazdan
    hangi üç sebebi sıralarsın?
17. Bir uygulamayı 8 thread'den 32 thread'e çıkardın, performans **düştü.** Muhtemel
    donanım sebepleri neler?
18. `free -h` çıktısında hangi sütuna bakarsın ve neden `free` sütunu yanıltıcıdır?
19. Cloud instance'larında neden varsayılan olarak swap yoktur? Bu kararın arkasındaki
    ilkeyi adlandır.
20. Bir iş yükünün "bellek bant genişliği sınırlı" olduğunu nasıl anlarsın? Hangi instance
    ailesini seçersin?
21. Bir veri seti 40 MB, L3 cache 32 MB. Veri setini 30 MB'a düşürmenin performans etkisi
    ne olur ve neden doğrusal değildir?
22. Noisy neighbor'ın donanım seviyesindeki mekanizmasını açıkla. Neden kendi metriklerinde
    göremezsin?

---

## Cevap Anahtarı

**1.** Hızlı bellek pahalı ve küçük, ucuz bellek yavaş ve büyüktür; hiyerarşi bu iki
gerçeği **yerellik sayesinde** birleştirir — küçük hızlı katman, büyük yavaş katmanın
sık kullanılan kısmını tutar. *(2.1.1, 2.1.3)*

**2.** 64 byte. Önemlidir çünkü **CPU bellekten asla tek byte okumaz** — en küçük transfer
birimi budur. Bu yüzden sıralı erişim neredeyse bedava komşu veri getirir, rastgele erişim
ise getirdiğinin çoğunu israf eder. *(2.3.5)*

**3.** (a) **Sahiplik:** L1/L2 çekirdeğe özel, L3 tüm çekirdekler tarafından paylaşılır.
(b) **Gecikme:** ~4 / ~14 / ~50+ çevrim. (c) **İçerik ayrımı:** L1 veri ve talimat olarak
ayrık, L2/L3 birleşik. *(2.3.2–2.3.4)*

**4. Zamansal:** Az önce eriştiğin veriye yakında tekrar erişeceksin — örn. bir döngü
değişkeni. **Mekânsal:** Eriştiğin verinin komşusuna erişeceksin — örn. dizi taraması.
Cache satırı mekânsal yerelliği, tahliye politikası (LRU) zamansal yerelliği sömürür.
*(2.1.3)*

**5.** Pipeline'da `IF` (talimat getirme) ve `MEM` (veri erişimi) aşamaları aynı çevrimde
çalışır (Faz 1.2.1). Tek cache olsaydı her çevrim **yapısal hazard** yaşanırdı. *(2.3.2)*

**6.** DRAM hücresi bir **kapasitördür** ve yükü milisaniyeler içinde sızar; tazeleme
olmadan veri kaybolur. SRAM ise **6 transistörlü bir geri beslemeli devredir** (Faz 0.4.4)
— beslendiği sürece durumunu kendi tutar. *(2.4.1)*

**7.** (a) **Güvenlik/yalıtım** — programlar birbirinin belleğini göremez; (b) **Koruma** —
hatalı pointer başka programı bozamaz; (c) **Yerleştirme** — program nereye yükleneceğini
bilmek zorunda değil. *(2.5.1)*

**8. Minor:** Sayfa zaten fizikselde var, sadece bu process'e eşlenmemiş — ~1–5 μs,
**tamamen normal.** **Major:** Sayfa **diskten** okunmalı — ~50 μs–10 ms. **Problem
işareti major fault'tur**; minor fault sayısının yüksek olması sağlıklı bir sistemin
normalidir. *(2.5.5)*

**9.** (a) **Cache satırı kullanımı:** Dizide bir miss 16 elemanı getirir; listede her
düğüm ayrı satırdır. (b) **Prefetcher:** Dizinin düzenli adımını yakalar, pointer
zincirini yakalayamaz. (c) **DRAM row buffer:** Sıralı erişim row hit üretir, rastgele
erişim row miss. Üçü birleşince 10–50 kat fark çıkar. *(2.3.5, 2.3.9, 2.4.2)*

**10.** İki thread **farklı** değişkenlere yazıyor ama o değişkenler **aynı 64 byte'lık
cache satırında**; donanım bunu paylaşım sanıp satırı sürekli invalidate ediyor. Sinsidir
çünkü **kod tamamen doğrudur** — mantıksal paylaşım yoktur, kilit hatası yoktur, profiler
tek bir yavaş fonksiyon göstermez. Sadece "paralelleştirdim ama hızlanmadı" olarak görünür.
*(2.3.8)*

**11.** TLB, **sayfa tablosunun cache'idir** — sanal→fiziksel çevirileri tutar. Miss
maliyeti 10–100+ çevrimlik bir page walk'tır. Huge page, giriş başına kapsanan bellek
miktarını 4 KiB'dan 2 MiB'a çıkarır: aynı giriş sayısıyla TLB kapsamı 6 MiB'dan 3 GiB'a
**512 kat** büyür. *(2.5.3, 2.5.4)*

**12.** Bellek dolar → OS sayfaları swap'e yazar → program o sayfalara **tekrar erişir**
(çünkü aktiftiler) → major fault → sayfa geri okunurken yer açmak için **başka** bir sayfa
swap'e yazılır → ona da erişilir → döngü hızlanır. Sistem CPU %10, I/O wait %80'de kilitlenir
ama **çökmez.** *(2.6.2)*

**13.** Aynı bellek adresinin birden fazla çekirdeğin L1'inde bulunabilmesinden doğan
**tutarsızlık** problemini çözer; her satıra Modified/Exclusive/Shared/Invalid durumu
verir. **Bedeli:** koordinasyon trafiği. Yoğun paylaşımlı yazmada cache line ping-pong
oluşur ve çekirdek sayısı arttıkça bu maliyet **büyür.** *(2.3.8)*

**14.** Çok soketli sunucularda her soketin kendi RAM'i vardır; uzak sokete erişim ~1,5–2
kat pahalıdır. Küçük instance'lar **tek bir NUMA düğümüne sığar** — fiziksel sunucunun
tek bir soketinin diliminde çalışırlar, dolayısıyla uzak erişim hiç oluşmaz. Sorun
`.12xlarge` ve üstünde başlar. *(2.7)*

**15.** Gecikmeyi **fizik** sınırlar: kapasitörün boşalma süresi, sinyalin mesafe kat
etmesi, satır açma işleminin analog doğası. Bant genişliği ise **paralellikle** artırılır:
daha çok bank, daha çok kanal, daha yüksek transfer hızı. Paralellik satın alınabilir,
gecikme alınamaz — bu, memory wall'un (2.1.2) sayısal ifadesidir. *(2.4.2)*

**16.** (a) **Cache miss ağırlıklı iş** — CPU "meşgul" görünüyor ama çevrimlerin çoğunu
bellek bekleyerek geçiriyor, IPC düşük (Faz 1.3.2). (b) **Bellek bant genişliği
doygunluğu** — çekirdekler DRAM yolunu paylaşmak için yarışıyor (2.4.3). (c) **Noisy
neighbor** — L3 payın başka bir kiracı tarafından süpürülüyor (2.3.4). Ayrıca SMT
kardeşiyle yarışma da olabilir (Faz 1.5.2).

**17.** (a) **False sharing** — thread sayısı arttıkça invalidation trafiği katlanır
(2.3.8). (b) **Bellek bant genişliği doygunluğu** — 8 thread boruyu zaten doldurmuştu
(2.4.3). (c) **NUMA** — 32 thread artık ikinci sokete taştı, verinin yarısı uzakta (2.7).
(d) **Cache basıncı** — her thread kendi çalışma kümesini getiriyor, paylaşılan L3 yetmiyor.

**18. `available`** sütununa bakarsın. `free` yanıltıcıdır çünkü Linux boş belleği **page
cache** olarak kullanır; bu bellek uygulama istediği anda geri alınabilir. `free` sütununun
düşük olması sağlıklı bir sistemin normal görüntüsüdür. *(2.8, SSS S2)*

**19.** Çünkü swap, bellek yetersizliğini **net bir arızadan belirsiz bir yavaşlığa**
çevirir; sağlık kontrolleri bunu yakalayamaz ve bozuk instance trafik almaya devam eder.
İlkenin adı **"fail fast"**: otomasyon net arızaya tepki verebilir, gri arızaya veremez.
*(2.6.5)*

**20.** **Belirtiler:** Çekirdek sayısını artırınca performans doğrusal artmıyor veya
düşüyor; CPU kullanımı yüksek ama IPC düşük; çalışma kümesi L3'ten çok büyük; erişim
sıralı ve veri yoğun (tarama, ETL, analitik). **Seçim:** Yüksek bellek bant genişliğine
sahip aileler — memory-optimized (`r`, `x`) veya yeni nesil compute (`c7g`/`c8g`);
vCPU başına bant genişliğine bak, toplam vCPU'ya değil. *(2.4.3, Cevap 2.3)*

**21.** **Büyük ve doğrusal olmayan bir sıçrama** — tipik olarak birkaç kat. Çünkü 40 MB
veri seti 32 MB L3'e sığmaz: döngüsel tarama her turda kendi verisini tahliye eder ve hit
oranı ~%0'a düşer. 30 MB'a inince veri **tamamen cache'te kalır** ve hit oranı ~%100'e
çıkar. Performans, veri boyutuna **basamaklı** tepki verir ve basamaklar cache
sınırlarındadır. *(Cevap 2.5)*

**22. Mekanizma:** Aynı fiziksel sunucudaki başka bir kiracının instance'ı **aynı L3
cache'i** (ve bellek kanallarını) paylaşır. O kiracı bellek yoğun bir iş çalıştırdığında
senin cache satırların tahliye edilir; senin erişimlerin L3 hit yerine DRAM'e gider.
**Göremezsin** çünkü senin CPU, bellek, disk ve ağ metriklerinin hepsi normaldir — tek
değişen gecikmedir. Ölçülebilen dolaylı ipuçları: `steal time`, aynı iş için p99 sapması,
IPC düşüşü. *(2.3.4)*

---

## Puanlama

| Doğru sayısı | Değerlendirme |
|---|---|
| 20–22 | Faz 3'e geç. Bu fazı sağlam oturtmuşsun. |
| 16–19 | Faz 3'e geçebilirsin. Yanlışların bölümlerini bir kez daha oku. |
| 11–15 | 2.3 ve 2.6'yı tekrar et — bunlar fazın taşıyıcı bölümleri. Sonra tekrar dene. |
| 0–10 | Fazı baştan geç. Özellikle 2.1 (neden hiyerarşi) ve 2.3 (cache) üzerinde dur; bu ikisi olmadan kalanı taşıyamazsın. |

**Bölüm C'de (16–22) zorlandıysan** — bu normaldir. Bölüm C mekanizmayı bilmekten
muhakemeye geçişi ölçer ve asıl hedef odur. A ve B'yi biliyorsan mekanizma yerindedir;
C zamanla, gerçek sistemlerle uğraştıkça oturur.

---

# Faz 2 — Kapanış ve Faz 3'e Köprü

## Bu fazdan ne taşıyorsun

| Kavram | Neden taşınıyor |
|---|---|
| **Hız/maliyet takası hiyerarşi doğurur** | Faz 3'te aynı prensip disk katmanlarında tekrarlanacak |
| **Latency ≠ throughput** | Faz 3'ün IOPS/throughput ayrımının tam karşılığı |
| **Yerellik bahsi** | Disk cache, page cache, CDN — hepsi aynı bahis |
| **Blok halinde transfer** (cache satırı) | Diskte "blok boyutu" olarak birebir geri gelecek |
| **Uçuculuk (volatility)** | DRAM güçle kaybolur → kalıcı depolamanın var olma sebebi |
| **Sanal bellek ve page cache** | Faz 3'teki dosya I/O'sunun tamamı buradan geçiyor |
| **Fail fast ilkesi** | Faz 7'de mimari karar çerçevesinin parçası olacak |
| **"Darboğaz nerede?" refleksi** | Haritanın geri kalanının omurgası |

## Faz 3 bunun neresine bağlanıyor

| Faz 2'de öğrendiğin | Faz 3'te karşına şöyle çıkacak |
|---|---|
| 2.1.1 gecikme merdiveni | Merdivenin disk basamakları açılacak: NVMe / SATA SSD / HDD |
| 2.4.1 DRAM uçucudur | Kalıcı depolamanın (persistent storage) neden var olduğu |
| 2.3.5 cache satırı = 64 byte | Disk blok boyutu = 4 KiB; aynı mantık, 64 kat büyük tane |
| 2.5.5 major page fault | Diskten okumanın gerçek maliyeti; mmap ve sayfa cache'i |
| 2.6 swap | Swap'in neden diskin en kötü kullanımı olduğu, sayılarla |
| 2.1.2 memory wall | Storage wall: SSD bile CPU'ya göre hâlâ çok yavaş |
| 2.3.6 sıralı vs rastgele | Diskte bu fark **daha da büyük** — HDD'de 100 kat |

> **Faz 3'te göreceğin en önemli devamlılık şudur:** Bu fazda öğrendiğin her prensip —
> hiyerarşi, yerellik, blok transferi, sıralı erişimin üstünlüğü, cache'in bir bahis
> olması — **diskte aynen tekrarlanır, sadece ölçek 1000 kat büyür.** Aynı fikirleri
> yeniden öğrenmeyeceksin; onları yeni bir ölçekte tanıyacaksın.
>
> Ve bu, haritanın tasarım mantığıdır: Faz 0'daki transistörden Faz 7'deki instance
> seçimine kadar **az sayıda prensip, giderek büyüyen ölçeklerde tekrar eder.**

---

*Faz 2 tamamlandı.* → **[Faz 3 — Depolama](Faz_3_Depolama.md)**
