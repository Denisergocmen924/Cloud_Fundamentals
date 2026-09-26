# Ara Sınav 1 — Faz 0–3: Transistörden Diske

> **Navigasyon:** [◀ Faz 3 — Depolama](Faz_3_Depolama.md) · **Ara Sınav 1** · [Faz 4 — Sistem Bus'ları ve I/O ▶](Faz_4_Bus_ve_IO.md)

---

## Bu sınav neyi ölçüyor?

Faz sonundaki "Kendini sına" testleri tek bir fazı yoklar. Ara sınav **farklı** bir şey ölçer: *"Bu dört
fazı birbirine bağlayabiliyor musun?"*

Faz 0 sana **fiziksel kuralları** verdi: 2'nin kuvvetleri, kritik yol, ısı, "hızlı olan küçüktür". Faz 1 bu
kuralları **CPU'nun içine** koydu: pipeline, IPC, çekirdek, vCPU. Faz 2 CPU'nun **tek büyük zayıflığını**
gösterdi — bellek yavaştır — ve bunu gizlemek için kurulan hiyerarşiyi anlattı. Faz 3 aynı hiyerarşiyi
**diskte**, bin kat büyük bir ölçekte tekrarladı. Bunlar dört ayrı konu değil, **dört ölçekte anlatılan tek
bir hikâye**: az sayıda ilke (hiyerarşi, yerellik, blok transferi, kuyruk eğrisi) tekrar tekrar geri
geliyor. Bu sınav bölümleri değil, hikâyeyi görüp görmediğini ölçer.

**Nasıl çalışılır:**

- Kâğıt-kalemle çöz; cevabı yazmadan bir sonrakine geçme.
- Cevap anahtarı her sorunun **hangi fazların kesişiminde** durduğunu söyler — kaçırdığın soru bir fazı
  değil, iki faz arasındaki **köprüyü** işaret eder.
- 21 soru var: Bölüm A (bağlantı muhakemesi, 1–9), Bölüm B (senaryo: "buluta taşındıktan sonra yavaşlayan
  veritabanı", 10–15), Bölüm C (çıktı okuma ve hesap, 16–21).
- İhtiyacın olan sayılar Ek A'da. Hedef süre: ~1 saat. Ama süre önemli değil; önemli olan her cevabın
  *neden* öyle olduğunu bir cümleyle gerekçelendirebilmek.

> **🤔 Başlamadan önce:** Şu cümleyi kendi kelimelerinle tamamla: *"%100 CPU bir teşhis değildir, çünkü
> ___ de %100 CPU gibi görünür."* Sonra ikinci bir cümle ekle: *"Hızlı olan ___; büyük olan ___."* Bu iki
> cümle, 21 sorunun tamamından geçen ipliktir.

---

# Bölüm A — Bağlantı muhakemesi (1–9)

Bu bölümdeki her soru en az iki fazın bilgisini birleştirmeni ister. Kısa ama gerekçeli cevap ver.

**1.** Faz 0'da sınırların 2'nin kuvveti olduğunu gördün; Faz 2'de 64 baytlık cache satırıyla 4.096 baytlık
sayfayı tanıdın. Bir sayfaya kaç cache satırı sığar? İki boyutun da 2'nin kuvveti olması neden kolaylık —
donanım bir adresle, aksi hâlde bu kadar ucuza yapamayacağı hangi işi yapıyor?

**2.** Faz 0'da saat hızı **kritik yol** ile sınırlıydı; Faz 1'de pipeline geldi. Pipeline'ın kritik yola
saldırarak saat tavanını nasıl yükselttiğini anlat — ve neyi **iyileştirmediğini** söyle (ipucu: tek bir
komutun kendi gecikmesi).

**3.** Faz 0'da ısı bütçesi saat hızını durdurdu ve çekirdekler çoğaldı. Faz 1'de "vCPU ≠ çekirdek"
öğrendin. İkisi de 8 vCPU'lu bir x86 instance'ında ve bir Graviton instance'ında ne kadar fiziksel hesap gücü
alırsın ve "aynı vCPU sayısı" karşılaştırması neden tuzaktır?

**4.** Faz 1'de "CPU %100 bir teşhis değildir" dedin; Faz 2'de DRAM erişiminin maliyetini öğrendin. Bir servis
`top`'ta %100 CPU gösteriyor, ama CPU sayısını ikiye katlamak hiçbir şeyi değiştirmiyor. Faz 2'deki hangi
mekanizma "meşgul" görünen bir CPU'yu çoğunlukla boşta bırakabilir ve hangi komut (hangi iki sayaçla) gerçek
işi beklemekten ayırır?

**5.** Faz 2'de `ortalama erişim = hit × L1 + miss × RAM` formülünü kullandın. L1 = 4 döngü ve RAM = 200
döngü ile %90 ve %99 hit oranında ortalama erişim süresini hesapla. Hit oranı %90'dan %99'a çıkınca program
kaç kat hızlanır ve bu neden "cache optimizasyonunun kaldıracı"dır?

**6.** Faz 2'de swap'in diskin en kötü kullanımı olduğunu gördün (major page fault). Rakamlara dök: yerel bir
NVMe SSD'den okuma (50–100 μs), bir RAM erişiminden (~70 ns) kaç kat yavaştır? EBS gp3 (~1 ms) için?

**7.** Cache verileri 64 baytlık **satırlarla**, disk 4 KiB'lık **bloklarla** taşır. İki hiyerarşinin de blok
halinde taşımasının tek nedenini söyle. Sonra 7200 RPM bir HDD'nin 4 KiB blokları rastgele okumasını (yaklaşık
120 IOPS) aynı diskin sıralı okumasıyla (150–250 MB/s) karşılaştır: rastgele desen kabaca kaç kat yavaştır?

**8.** Kuyruk eğrisi Faz 3'te (`await`) göründü ve ağda ve kapasite planlamasında geri dönecek. Bir disk %50
kullanımda 1 ms'de cevap veriyorsa, %80, %90, %95 ve %99'da hangi gecikmeyi beklersin? `iostat -x`'in hangi iki
sütunu eğrinin dik kısmında olduğunu gösterir?

**9.** Faz 0'daki SRAM/DRAM ayrımı "hızlı olan küçüktür; büyük olan yavaştır" dedi. Şu beşini al: L3 cache,
DRAM, yerel NVMe instance store, EBS volume, HDD. Hangileri **uçucu**, hangileri **kalıcı**? Kalıcı olanlardan
hangisi bir instance **stop**'undan sağ çıkar ve Faz 3 neden "kalıcılık gerekmiyor" durumunu instance store'un
tek meşru kullanımı sayar?

---

# Bölüm B — Senaryo: "Buluta taşındıktan sonra yavaşlayan veritabanı" (10–15)

> **Olay:** Bir ekip, PostgreSQL veritabanını şirket içi bir sunucudan (256 GB RAM, yerel NVMe) AWS'ye taşıdı:
> **4 vCPU / 32 GiB RAM**'lik bir `r` ailesi instance'ı ve **500 GiB gp2** volume. Veritabanının sıcak çalışma
> kümesi yaklaşık **60 GB**. Taşımadan sonra milisaniyede biten sorgular saniyeler sürüyor. Mühendis şunu
> topladı:
>
> ```
> $ top
> %Cpu(s):  8.1 us,  2.0 sy,  0.0 ni, 19.5 id, 70.2 wa,  0.0 hi,  0.2 si,  0.0 st
> $ free -h
>                total   used   free  shared  buff/cache  available
> Mem:            31Gi   30Gi  180Mi    40Mi       900Mi      800Mi
> Swap:          4.0Gi  3.1Gi  0.9Gi
> $ vmstat 1          # si/so sütunları:  si 200   so 300
> $ iostat -x 1       # veri volume'u
> Device   r/s     w/s    rkB/s    wkB/s    await  aqu-sz
> nvme1n1  1420    80     22720    1280     45     68
> ```
>
> Ekibin ilk refleksi: "CPU çok zayıf — vCPU'ları ikiye katlayalım."

**10.** `top` satırını oku. Darboğaz CPU mu? Makinenin gerçekte ne yaptığını gösteren tek sayı hangisi ve
vCPU'ları ikiye katlamak neden neredeyse hiçbir şeyi değiştirmez?

**11.** gp2 volume `boyut × 3` IOPS verir. Bu 500 GiB'lık volume'un IOPS tavanını hesapla ve `iostat`
okumasıyla karşılaştır. Sonra `Throughput = IOPS × I/O boyutu` ile ortalama I/O boyutunu ve throughput'u bul;
volume hangi tavana (IOPS mu throughput mu) dayanmış, söyle.

**12.** `free -h` ve `vmstat`'a bak. `available` neden `free`'den daha anlamlı? `si`/`so` > 0 sana ne
söylüyor? Bir **bellek** yetersizliğinin (Faz 2) **disk** yükünü (Faz 3) nasıl ürettiğini adım adım anlat: 60
GB'lık çalışma kümesi 32 GiB'a sıkıştırılınca cache/page-cache hit oranına ne olur?

**13.** Faz 3'teki Little yasasıyla (`await ≈ aqu-sz ÷ IOPS`) `iostat` sayılarının birbiriyle tutarlı olup
olmadığını kontrol et. Sonuç, zamanın nerede harcandığı hakkında ne söylüyor?

**14.** Masada iki çözüm var: (a) 16.000 IOPS'lu bir gp3 volume'a geçmek; (b) en az 64 GiB RAM'i olan daha büyük
bir `r` boyutuna geçmek. **İlk** hangisini yaparsın, neden? Her çözüm hangi tek yeni sayıyı değiştirir ve bir
EBS tipi seçerken instance'ın kendisi hakkında neyi hâlâ doğrulaman gerekir?

**15.** Düzeltmeden sonra veritabanı normal günde iyi çalışıyor, ama yoğun saatlerde volume yaklaşık %95 dolu
olduğunda p99 gecikmesi sıçrıyor. Hangi eğri bunu açıklıyor? Hangi kullanım düzeyinde kalmayı planlarsın ve bu,
kapasiteyi "tam yetecek kadar" almak hakkında sana ne söylüyor?

---

# Bölüm C — Çıktı okuma ve hesap (16–21)

**16.** Varsayılan bir gp3 volume (3.000 IOPS, 125 MB/s) şunu gösteriyor:

```
Device   r/s     w/s     rkB/s    wkB/s    await  aqu-sz
nvme1n1  2900    100     92800    3200     12     36
```

(a) Toplam IOPS'u, throughput'u ve ortalama I/O boyutunu hesapla. (b) Volume hangi tavanda? (c) Aynı iş yükü
64 KiB I/O kullansaydı 3.000 IOPS'ta ne olurdu — hangi tavan önce dolardı?

**17.** Bir fiziksel sunucuda `lscpu` şunu yazdırıyor: `Thread(s) per core: 2`, `Core(s) per socket: 8`,
`Socket(s): 2`, `NUMA node(s): 2`. (a) İşletim sistemi kaç mantıksal CPU görür ve kaç fiziksel çekirdek var? (b)
Node 0'da çalışan bir process sürekli node 1'deki belleği okuyorsa bellek gecikmesine ne olur ve hangi komut
ailesi (Faz 2.7) mesafeleri gösterir ve düzeltmene izin verir?

**18.** Bu `vmstat 1` satırı yavaş bir sunucudan geliyor:

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 3  9 4194304  81920   2048 153600  850  1100  5200  4800  900 1500  4  3  5 88  0
```

Bu satırdaki **iki bağımsız alarmı**, her birinin ait olduğu fazı ve ikisini bağlayan neden-sonuç zincirini
söyle. Önce hangi kaynağı düzeltirsin?

**19.** Bir veritabanı 24 GiB'lık çalışma kümesini rastgele tarıyor. TLB 1.536 giriş tutuyor. 4 KiB sayfalarla
ve 2 MiB huge page'lerle TLB kapsamını (reach) ve her birinin çalışma kümesinin hangi kesrini kapsadığını
hesapla. Huge page'ler bu iş yüküne neden yardım eder?

**20.** (a) 7200 RPM bir HDD'nin rastgele IOPS'unu üç gecikme bileşeninden hesapla (seek ≈ 4 ms, transfer ≈
0,01 ms). (b) Aynı seek ve transfer süreleriyle 15.000 RPM'lik bir disk için yap: daha hızlı iş mili ne kadar
yardım etti ve bu mekanik diskler hakkında ne söylüyor? (c) Altı adet 7200 RPM disk (her biri 120 IOPS) RAID 5
olarak çalışıyor: 4× yazma cezası verildiğinde yaklaşık okuma ve yazma IOPS'u nedir?

**21.** Ek A.1'in insan ölçeğinde (1 döngü = 1 saniye, CPU 3 GHz) üç sayı koy: bir RAM erişimi (~70 ns), bir EBS
gp3 okuması (~1 ms) ve bir HDD rastgele okuması (~10 ms). Sonra merdivenin, "RAM hızlı, disk yavaş"ın
öğretmediği şeyi bir cümleyle söyle.

---

## Cevap anahtarı

**1.** 4096 ÷ 64 = bir sayfada **64 cache satırı**. İkisi de 2'nin kuvveti (2⁶ ve 2¹²), dolayısıyla donanım
bir adresi sadece **bit keserek** ayırır: alt 6 bit satır içi ofset, sonraki 6 bit sayfadaki satırı, üst
bitler sayfayı seçer. Bölme ya da çarpma devresi gerekmez — "her sınır 2'nin kuvvetidir" refleksi bir adres
çözme ekonomisidir. · *Faz 0.1 × Faz 2.3.5*

**2.** Saat periyodu, iki flip-flop arasındaki **en uzun sinyal yolundan** (kritik yol) kısa olamaz. Pipeline
bir komutun işini aşamalara böler; böylece her aşamanın kritik yolu kısalır ve saat yükseltilebilir; her
döngüde yeni bir komut girer. Bir komutun baştan sona **toplam süresini kısaltmaz** (gecikmesi aşağı yukarı
aynı kalır, hatta hafifçe artar) — **gecikmeyi** değil, **throughput'u** iyileştirir. · *Faz 0.3.3 × Faz 1.4.1*

**3.** Isı saat hızının yükselmesini durdurdu, üreticiler çekirdek ekledi. x86'da **1 vCPU = 1 SMT thread**
(fiziksel çekirdeğin yarısı), yani 8 vCPU ≈ 4 çekirdek; Graviton'da **1 vCPU = 1 fiziksel çekirdek**, yani 8
vCPU = 8 çekirdek. Aynı vCPU sayısı fiziksel hesap gücünde 2 kata kadar fark saklar; bu yüzden vCPU sayısı
mimariler arasında asla tek başına karşılaştırılmamalı. · *Faz 0.2.3 × Faz 1.5.3*

**4.** Bir DRAM erişimi ~200–250 döngü tutar; veri cache'te yoksa CPU **bekler** ve bekleme de "meşgul" sayılır
(CPU %100 bir teşhis değil). Daha fazla çekirdek beklemeyi kısaltmaz. `perf stat -e cache-misses,instructions,cycles
<komut>` bunu gösterir: düşük **döngü başına komut** (IPC) ve yüksek cache-miss sayısı, CPU'nun hesaplamak yerine
bellekte tıkandığı anlamına gelir. · *Faz 1.3.2 × Faz 2.1*

**5.** %90'da: 0,9 × 4 + 0,1 × 200 = 3,6 + 20 = **23,6 döngü**. %99'da: 0,99 × 4 + 0,01 × 200 = 3,96 + 2 =
**5,96 döngü**. 23,6 ÷ 5,96 ≈ **4 kat** hızlanma. Hit oranındaki 9 puanlık iyileşme hızı katlar, çünkü ortalamaya
nadir **miss'ler** hükmeder (bir hit'in 50 katı maliyetli): 100 erişimde 9 miss'i ortadan kaldırmak zamanın çoğunu
ortadan kaldırır. Cache optimizasyonunun kaldıracı budur. · *Faz 2.3.6*

**6.** NVMe: 50–100 μs ÷ 70 ns ≈ RAM'den **700–1.400 kat** yavaş. gp3: 1 ms ÷ 70 ns ≈ **14.000 kat**. Major page fault
(ya da swap-in) her miss'te bu bedeli öder — swap'in bir diskin olabilecek en kötü kullanımı olmasının nedeni bu. ·
*Faz 2.1.1 × Faz 2.6 × Faz 3.3.3*

**7.** Bir erişimin maliyeti çoğunlukla **sabit gecikmedir** (veriyi bulmak), her ek baytın transferi değil;
bu yüzden sabit bedel bir kez ödenir ve **yerelliğe** bahis oynayarak bir blok birlikte getirilir (cache satırında
64 B, diskte 4 KiB). Rastgele 4 KiB HDD okuması: 120 × 4 KiB ≈ 480 KiB/s ≈ 0,5 MB/s; sıralıda 150–250 MB/s —
rastgele desen kabaca **300–500 kat yavaş** (workbook'un daha ayrıntılı modeli 466 kat verir). ·
*Faz 2.3.5 × Faz 3.1.2–3.1.3*

**8.** %50'ye göre: %80 → 2,5 ms, %90 → 5 ms, %95 → **8 ms**, %99 → **40 ms**. Sütunlar **`await`** (istek başına
ortalama süre) ve **`aqu-sz`** (ortalama kuyruk uzunluğu); ikisi birlikte tırmanıyorsa disk kuyruk eğrisinin dik
kısmındadır. · *Faz 3.4.4 (eğri Faz 5'te ve 7.6.2'de geri döner)*

**9.** Uçucu: **L3 ve DRAM** (SRAM/DRAM gücü kaybedince içeriğini kaybeder) ve **instance store** (host üzerinde
yaşar; instance stop edilince ya da host arızalanınca veri kaybolur). Kalıcı: **EBS** ve genel olarak **HDD/SSD
diskler**. EBS bir instance stop'undan sağ çıkar. Instance store yalnızca kalıcılığın gerekmediği durumda doğrudur
(scratch, cache, replike veri) — karşılığında en yüksek IOPS'u alırsın. Kalıcılık hıza mal olur: hızlı olan
küçük ve uçucu, büyük olan yavaş ve kalıcıdır. · *Faz 0.4.4 × Faz 3.3.4*

**10.** Hayır. **`wa` = %70,2** — CPU boşta, **I/O bekliyor**; `us` yalnızca %8. vCPU sayısını ikiye katlamak, aynı
diski bekleyecek daha fazla işlemci ekler: makine **I/O-bound** (ve arkasında bellek-bound). CPU sayısı yanlış
koldu. · *Faz 1.5.4 × Faz 3.4.3*

**11.** 500 × 3 = **1.500 IOPS**. Ölçüm: r/s + w/s = 1.420 + 80 = **1.500** — tam tavanda. Throughput: 22.720 + 1.280 =
24.000 kB/s ≈ **24 MB/s**; yani ortalama I/O boyutu 24.000 ÷ 1.500 = **16 KiB**. Volume'un throughput sınırı yakın
bile değil; **IOPS tavanına** dayanmış. · *Faz 3.4.1 × Faz 3.4.5*

**12.** `available` (800 Mi), swap'e gitmeden bir process'e gerçekten verilebilecek miktardır (geri
kazanılabilir cache dahil); `free` yalnızca tamamen kullanılmayan sayfaları sayar ve sağlıklı bir makinede
neredeyse her zaman küçüktür, bu yüzden yanıltır. `si`/`so` > 0, sistemin **aktif olarak swap yaptığı** anlamına
gelir — acil bir işaret. Zincir: 60 GB'lık sıcak veri 32 GiB'a sığmaz → page cache/buffer cache **hit oranı düşer**
→ okumaların çoğu miss olur ve diske gider (70 ns yerine 1 ms) → bellek yetersizliği **disk yüküne** dönüşür (r/s
tavanda) → `wa` %70. · *Faz 2.6 × Faz 3.4.4*

**13.** 68 ÷ 1.500 ≈ 0,0453 s ≈ **45 ms** — bildirilen `await` ile birebir. Sayılar tutarlı: istekler zamanlarını
servisin içinde değil **kuyrukta** harcıyor; bekleyen 68 istek, saniyede en fazla 1.500 yapabilen bir volume'un
arkasında sıra bekliyor. · *Faz 3.4.4 (Little yasası, Cevap 3.3)*

**14.** **Önce belleği** düzelt: (b) ≥ 64 GiB'lık daha büyük bir `r` boyutu çalışma kümesinin RAM'de yaşamasına izin
verir; bu 1.500 IOPS'un çoğu artık **gerekmez** — nedeni ortadan kaldırır; daha hızlı bir volume (a) yalnızca
belirtiyi ucuzlatır (gp3 3.000–16.000 IOPS verir), her miss yine ~1 ms tutar. (b) **hit oranını**, (a) **IOPS
tavanını** değiştirir. Sonra **iki tavanı** da kontrol et: volume sınırı **ve** instance'ın kendi EBS bant genişliği
sınırı (`aws ec2 describe-instance-types … EbsInfo`). Sonrasında gp2 → gp3 geçişi ucuz ve mantıklıdır. ·
*Faz 2.6 × Faz 3.4.5 × Faz 7.3.5*

**15.** **Kuyruk eğrisi**: %95 kullanımda gecikme %50 değerinin yaklaşık **8 katı**, %99'da yaklaşık 40 katı. p99
sıçramaları bunun ilk görüldüğü yerdir. Kapasiteyi, tepe kullanım yaklaşık **%60–80**'de kalacak şekilde planla
(%95'te 8× değil, %80'de 2,5×): "tam yetecek kadar" kapasite, eğrinin dik kısmında çalışmak demektir. ·
*Faz 3.4.4 × Faz 7.6.2*

**16.** (a) IOPS = 2.900 + 100 = **3.000**; throughput = 92.800 + 3.200 = 96.000 kB/s = **96 MB/s**; I/O boyutu =
96.000 ÷ 3.000 = **32 KiB**. (b) **IOPS tavanı** (3.000) — 96 MB/s hâlâ 125 MB/s'nin altında. (c) 3.000 × 64 KiB =
**192 MB/s** > 125 MB/s: **throughput tavanı** önce dolardı, kabaca 125.000 ÷ 64 ≈ 2.000 IOPS'ta. (Little
kontrolü: 36 ÷ 3.000 = 12 ms = `await` ✓.) · *Faz 3.4.1*

**17.** (a) 2 × 8 × 2 = **32 mantıksal CPU** (x86 instance olarak satılırsa 32 vCPU), ama yalnızca **16 fiziksel
çekirdek** (2 soket × 8). (b) Uzak node belleği **~1,5–2 kat yavaştır** (~70 ns yerine ~140 ns). `numactl --hardware`
(mesafeler) ve `numastat -m` bunu gösterir; `numactl` process'i ve belleğini tek bir node'a sabitlemene izin
verir. · *Faz 1.5.1 × Faz 2.7*

**18.** Alarm 1: **`si` 850 / `so` 1100 > 0** — makine swap yapıyor (bellek, Faz 2.6; `free` yalnızca 80 MB). Alarm 2:
**`wa` 88** — CPU I/O bekliyor, `us` yalnızca %4 (disk, Faz 3.4). Zincir: bellek tükendi → sayfalar diske ve
diskten swap ediliyor → disk doydu → her process bekliyor (`b` = 9 bloklu). Önce **belleği** düzelt (daha fazla RAM
ya da daha küçük çalışma kümesi); daha hızlı bir disk yalnızca swap'i gizler. · *Faz 2.6 × Faz 3.4.4*

**19.** 4 KiB sayfalar: 1.536 × 4 KiB = **6 MiB** → 6 MiB ÷ 24 GiB ≈ çalışma kümesinin **%0,024**'ü. 2 MiB sayfalar:
1.536 × 2 MiB = **3 GiB** → 3 ÷ 24 = **%12,5** (512 kat daha geniş kapsam). 24 GiB üzerinde rastgele erişimde 4 KiB
sayfalar neredeyse her erişimi TLB miss'e (page table walk) çevirir; huge page'ler her sekiz erişimden birini
doğrudan TLB'den karşılar. · *Faz 2.5.3–2.5.4*

**20.** (a) Dönme = 60.000 ÷ 7.200 ÷ 2 = **4,17 ms**; toplam = 4 + 4,17 + 0,01 ≈ **8,2 ms** → 1 ÷ 0,0082 ≈ **120 IOPS**.
(b) Dönme = 60.000 ÷ 15.000 ÷ 2 = **2 ms**; toplam = 4 + 2 + 0,01 ≈ 6 ms → **~165 IOPS**: iş mili hızını ikiye katlamak
yalnızca ~%35 kazandırır, çünkü seek (kafayı hareket ettirmek) değişmez — mekanik bir sınır; HDD IOPS'u 30 yılda
büyümedi, kapasite 1000 kat büyüdü. (c) Okuma ≈ 6 × 120 = **720 IOPS**; yazma ≈ 720 ÷ 4 = **180 IOPS**. ·
*Faz 3.1.2–3.1.3 × Faz 3.5.2*

**21.** 3 GHz → 1 ns = 3 döngü. RAM: 70 ns ≈ 210 döngü ≈ **3,5 dakika**. gp3: 1 ms = 3.000.000 döngü ≈ 35 gün ≈ **yaklaşık bir
ay**. HDD: 10 ms = 30.000.000 döngü ≈ 347 gün ≈ **yaklaşık bir yıl**. Merdiven farkların "biraz" değil **büyüklük
mertebesi** olduğunu gösterir: bir disk okuması RAM'e göre, aylarca beklemenin bir kahve molasına oranı gibidir —
haritadaki her tasarım kararı bir alt basamağa inmemeye çalışır. · *Faz 2.1.1 × Faz 3.1.2*

---

## Puanlama

| Doğru sayısı | Değerlendirme |
|---|---|
| 18–21 | Dört fazı tek bir hikâye olarak görüyorsun. Faz 4'e hazırsın. |
| 14–17 | İyi. Kaçırdığın soruların işaret ettiği **köprü** bölümlerine (aşağıdaki tablo) bir tur dön. |
| 9–13 | Fazları ayrı ayrı biliyorsun ama kesişimde zorlanıyorsun. Bölüm B senaryosunu baştan çöz. |
| 0–8 | Faz 0, 1, 2 ve 3'ü yeniden gez; özellikle 2.3 (cache), 2.6 (swap) ve 3.4 (IOPS ve kuyruk eğrisi). |

**Hangi soruyu kaçırırsan nereye dön:**

| Kaçırdığın soru | Dön — bu köprü zayıf |
|---|---|
| 1 | 2'nin kuvvetleri × cache satırı ve sayfa (0.1 × 2.3.5) |
| 2 | Kritik yol × pipeline (0.3.3 × 1.4.1) |
| 3 | Isı bütçesi × çekirdek ve vCPU (0.2.3 × 1.5.3) |
| 4 | "CPU %100" × DRAM beklemeleri (1.3.2 × 2.1) |
| 5 | Hit oranı ve ortalama erişim süresi (2.3.6) |
| 6, 21 | Sayılarla gecikme merdiveni (2.1.1 × 3.1.2) |
| 7 | Blok transferi × sıralı ve rastgele (2.3.5 × 3.1) |
| 8, 15 | Kuyruk eğrisi (3.4.4) |
| 9 | Hızlı = küçük × uçucu ve kalıcı (0.4.4 × 3.3.4) |
| 10, 13, 16 | `iostat` okuma / Little yasası (3.4.3 × 3.4.4) |
| 11 | IOPS × I/O boyutu = throughput (3.4.1) |
| 12, 14, 18 | Bellek yetersizliği → swap → disk yükü (2.6 × 3.4) |
| 17 | Çekirdek, thread, NUMA (1.5.1 × 2.7) |
| 19 | TLB kapsamı ve huge page (2.5.3–2.5.4) |
| 20 | HDD bileşenleri ve RAID (3.1.2 × 3.5.2) |

---

## Kapanış — buradan Faz 4'e

Bu sınav dört fazı tek bir hikâye olarak sınadı: fiziksel kurallar (Faz 0), CPU (Faz 1), onu aç bırakan bellek
(Faz 2) ve belleği aç bırakan disk (Faz 3). Senaryonun dersi tek ve pahalı: **belirti merdivenin dibinde
görüldü (yavaş bir disk), ama neden bir basamak yukarıdaydı (yetersiz bellek).** "CPU zayıf, vCPU ekleyelim"
Faz 1'in seni uyardığı reflekstir; Faz 2 ve Faz 3'ün sayıları teşhisi aritmetiğe çevirdi. Üç dersi ileri taşı:
(1) bekleme de meşgul sayılır — *neyin* beklendiğini bul; (2) aynı eğri (hiyerarşi → yerellik → blok → kuyruk) her
ölçekte tekrar eder; (3) önce ölç, sonra hesapla, sonra satın al.

Faz 4, bilerek açık bıraktığımız boşluğu açar: **veri diskten CPU'ya gerçekte nasıl yolculuk eder** — PCIe
lane'leri, DMA ve kesmeler. Bu dört fazda tanıştığın her "hızlı" cihaz o yola bağlıdır.

---

> **Navigasyon:** [◀ Faz 3 — Depolama](Faz_3_Depolama.md) · **Ara Sınav 1** · [Faz 4 — Sistem Bus'ları ve I/O ▶](Faz_4_Bus_ve_IO.md)
