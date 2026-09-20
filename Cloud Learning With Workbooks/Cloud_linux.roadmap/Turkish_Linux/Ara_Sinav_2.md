# Ara Sınav 2 — Faz 3–4: Process, Bellek ve Performans

> **Navigasyon:** [◀ Faz 4 — Bellek, I/O ve Performans](Faz_4_Bellek_IO_Performans.md) · **Ara Sınav 2** · [Faz 5 — Boot, Init ve systemd ▶](Faz_5_Boot_Init_systemd.md)

---

## Bu sınav neyi ölçüyor?

Faz sonundaki "Kendini sına" testleri tek bir fazı yoklar. Ara sınav **farklı** bir şey ölçer: *"Bu
iki fazı birbirine bağlayabiliyor musun?"*

Faz 3 sana **çalışan sistemin atomunu** verdi: process — nasıl doğar, hangi durumda olur, nasıl ölür,
kaynağı nasıl sınırlanır. Faz 4 aynı sisteme **kaynak** gözüyle baktı: bellek, I/O, darboğaz. Gerçek
bir olayda bu ikisi hiç ayrı durmaz. "Sunucu yavaş" dendiğinde aynı anda iki fazın bilgisini
kullanırsın: process'in hangi **durumda** olduğunu (Faz 3 — R/S/D/Z) ve o durumun hangi **kaynak**
darboğazına işaret ettiğini (Faz 4 — CPU/bellek/I/O). D-state bir Faz 3 kavramıdır; ama "yüksek load +
düşük CPU" bir Faz 4 teşhisidir — ve ikisi **aynı** olayın iki yüzüdür.

**Nasıl çalışılır:**

- Kâğıt-kalemle çöz; cevabı yazmadan bir sonrakine geçme.
- Cevap anahtarı her sorunun **hangi fazların kesişiminde** durduğunu söyler — kaçırdığın soru bir
  fazı değil, iki faz arasındaki **köprüyü** işaret eder.
- 21 soru var: Bölüm A (bağlantı muhakemesi, 1–9), Bölüm B (senaryo: "uygulama yavaş" olayı, 10–15),
  Bölüm C (komut ve çıktı okuma, 16–21).
- Hedef süre: ~1 saat. Ama süre önemli değil; önemli olan her cevabın *neden* öyle olduğunu bir
  cümleyle gerekçelendirebilmek.

> **🤔 Başlamadan önce:** Şu cümleyi kendi kelimelerinle tamamla: *"Load average 12, ama `top`'ta CPU
> %90 boşta — makine hem 'aşırı yüklü' hem 'boşta' nasıl olabilir?"* Bu tek soru iki fazı da içeriyor.
> Cevabını bir kenara yaz; Soru 3'te göreceğiz.

---

# Bölüm A — Bağlantı muhakemesi (1–9)

Bu bölümdeki her soru en az iki fazın (çoğunlukla Faz 3 × Faz 4) bilgisini birleştirmeni ister. Kısa
ama gerekçeli cevap ver.

**1.** Bir process `D` durumunda (Faz 3) takılı. Bu tek gerçek, Faz 4'ün hangi iki metriğini aynı anda
etkiler — load average ve `iowait`? Neden `kill -9` bu process'i (Faz 3) kurtarmaz ve doğru bakılacak
yer neden disktir (Faz 4)?

**2.** Faz 3'te "cgroup bellek limiti aşılınca OOM olur" dedik. Faz 4'te "OOM Killer en yüksek
`oom_score`'lu process'i öldürür" dedik. Bir container'ın **kendi** limiti dolduğunda sistem geneli
OOM'dan farkı ne, ve `dmesg`'te bunları hangi ibare ayırır?

**3.** Load average 12, ama `top`'ta CPU %90 boşta (`id`). Makine hem "aşırı yüklü" hem "boşta" nasıl
olabilir? Cevabını load'un tanımı (Faz 3: R + D) ile `wa` metriği (Faz 4) üzerinden ver.

**4.** `ps aux`'ta bir process'in `%MEM` sütunu (RSS'e dayanır, Faz 3) yüksek görünüyor ama sistemde
gerçek bellek baskısı yok. Bu çelişki Faz 4'ün hangi iki kavramıyla (RSS vs VSZ / shared memory /
page cache) açıklanabilir? En az iki olası neden yaz.

**5.** Faz 3'te sinyalleri (SIGTERM/SIGKILL) gördük. Bir process OOM Killer tarafından öldürülünce
(Faz 4) çekirdek ona hangi sinyali gönderir, ve bu sinyal neden yakalanamaz? Process "graceful
shutdown" yapabilir mi?

**6.** `top`'ta `VIRT` (VSZ) 20 GB, `RES` (RSS) 500 MB olan bir JVM process'i var (Faz 3 sütunları).
Bu makinede toplam RAM 8 GB. Nasıl oluyor da 20 GB "ayırmış" bir process 8 GB'lık makinede çalışıyor?
Faz 4'ün hangi mekanizması (sanal bellek / lazy allocation) bunu mümkün kılıyor?

**7.** Faz 3'te thread ile process farkını ve vCPU sayısını gördük. 8-thread'li bir uygulama 4-vCPU
makinede çalışıyor ve load 8. Bu load "sağlıklı tam kapasite" mi yoksa "aşırı yük" mü — cevabı hangi
tek sayıya (Faz 4) böInce netleşir?

**8.** Swap'ta thrashing (Faz 4) yaşayan bir sistemde process'ler hangi durumda (Faz 3: R/S/D)
birikir, ve bu neden load'u şişirir ama CPU'yu boşta bırakır? İki fazın kavramını tek cümlede bağla.

**9.** Faz 3'te `nice`/`renice` ile CPU önceliğini, cgroup ile CPU kotasını gördük. Bir process
CPU-bound (Faz 4) ise `renice` ile "yavaşlatmak" işe yarar mı; I/O-bound ise `renice` neden işe
yaramaz? Darboğaz türü (Faz 4) ile öncelik aracı (Faz 3) arasındaki ilişkiyi kur.

---

# Bölüm B — Senaryo: "uygulama yavaş" olayı (10–15)

> Bir üretim Ubuntu sunucusunda (4 vCPU, 16 GB RAM) bir web uygulaması çalışıyor. Saat 03:00'te
> nöbetçi sana yazdı: "site yavaş, bazen 502 dönüyor." SSH ile girdin. Aşağıdaki altı soru bu sunucuda
> sırayla yaptığın teşhis.

**10.** İlk komutun `uptime`: `load average: 11.5, 10.2, 7.8`. Bu tek satır sana ne söyler ve ne
söyle**mez**? Bir sonraki komutun ne olmalı ve neden — henüz "CPU yetmiyor, instance büyütelim"
diyemezsin, hangi metriği görmen lazım?

**11.** `top` açtın: `%Cpu: 5 us, 3 sy, 0 ni, 8 id, 84 wa`. Load 11.5, ama CPU'nun %84'ü `wa`.
Darboğaz dört türden (CPU/bellek/I/O/ağ) hangisi, ve nasıl anladın? "Daha büyük instance" (daha çok
vCPU) bu sorunu çözer mi?

**12.** `wa`'nın yüksek olduğunu gördün. Hangi iki komutla (biri process durumu — Faz 3, biri disk —
Faz 4) "hangi process diski bekletiyor ve disk gerçekten doygun mu" sorusunu daraltırsın? Her komutta
tam olarak hangi sütuna bakarsın?

**13.** `ps -eo stat,comm | grep '^D'` sana iki `D` process'i verdi: uygulamanın kendisi ve
`kworker`. `iostat -xz 1` çıktısında `nvme0n1` diski `%util 99, await 80ms`. Teşhisi tek cümlede yaz.
502'ler (Faz 3'ten hatırla: graceful shutdown / yük dengeleyici) bu tabloyla nasıl ilişkili olabilir?

**14.** Bu arada `free -h` de çalıştırdın: `used 3.2Gi, buff/cache 12Gi, available 12Gi`. Biri
"bak RAM neredeyse dolu, o yüzden yavaş" diyor. Bu doğru bir teşhis mi? Bir cümleyle çürüt (Faz 4),
ve bu sunucuda belleğin darboğaz **olmadığını** hangi sütun kanıtlıyor?

**15.** Kök neden: gece 03:00'te çalışan bir yedekleme işi diski doldurmuş. Kalıcı çözüm için üç
seçenek düşün ve her birini bir darboğaz-türü/araç mantığıyla gerekçelendir: (a) yedeklemeyi
`ionice`/`nice` ile düşük önceliğe almak, (b) daha yüksek IOPS'li disk (gp3 → io2), (c) instance'ı
vCPU olarak büyütmek. Hangisi bu **özel** darboğazı çözmez ve neden?

---

# Bölüm C — Komut ve çıktı okuma (16–21)

Aşağıdaki çıktıların her birinde ilgili satırları okuyup soruyu cevapla.

**16.** `vmstat 1` çıktısı:

```
 r  b   swpd   free   buff  cache   si   so   bi    bo   us sy id wa
 9  0      0 480000  40000 3800000   0    0   16    48   94  4  2  0
 8  1      0 470000  40000 3810000   0    0    8    40   91  6  3  0
```

`r`, `b`, `si/so`, `wa`, `us` sütunlarını okuyarak darboğazın türünü söyle. Bu **CPU-bound** mı
I/O-bound mı, ve `si/so`'nun sıfır olması neyi eler?

**17.** İkinci bir sunucudan `vmstat 1`:

```
 r  b   swpd   free   buff  cache   si   so   bi     bo    us sy id wa
 1  6  980000  90000  12000  140000  520  610  8200   240   4  5 10 81
```

Bu çıktı 16. sorudakinden nasıl farklı? `si/so`, `b`, `wa`, `free`, `buff/cache` sütunları birlikte
hangi **iki** darboğazı aynı anda gösteriyor (ipucu: hem bellek hem I/O)? Kök neden büyük olasılıkla
ne?

**18.** `dmesg -T | tail` çıktısı:

```
[Tue 03:14:22] Out of memory: Killed process 2481 (python3) total-vm:8123400kB,
  anon-rss:7981200kB, file-rss:0kB, oom_score_adj:0
```

Bu satır Faz 4'ün hangi mekanizmasını gösteriyor? `python3` gerçekten "suçlu" mu, yoksa sadece en
büyük mü — nasıl karar verirsin? `anon-rss` ~8 GB'ın "anonim" olması neden önemli (page cache ile
farkı)?

**19.** `ps -eo pid,comm,vsz,rss --sort=-rss | head -3`:

```
   PID COMMAND        VSZ      RSS
  1102 java       20918000  512000
   890 postgres    2400000  410000
```

`java`'nın VSZ'si ~20 GB, RSS'i ~512 MB. "Java 20 GB RAM yiyor" diyen birine bir cümleyle ne dersin?
Hangi sütun gerçek RAM kullanımını verir, ve VSZ'nin bu kadar büyük olması neyin sonucu?

**20.** `top` üst bloğu:

```
top - 03:20:01 up 5 days,  load average: 0.80, 0.75, 0.70
Tasks: 180 total, 1 running, 176 sleeping, 0 stopped, 3 zombie
%Cpu(s): 12.0 us, 3.0 sy, 0.0 ni, 84.0 id, 1.0 wa
```

Bu makine sağlıklı mı? Load (4 vCPU'da 0.80), `wa` (1.0) ve `3 zombie` sırayla ne anlatıyor? `3
zombie` bir alarm sebebi mi — ne zaman olurdu (Faz 3)?

**21.** Aşağıdaki komut zinciri ne yapıyor, ve çıktısı bir "yavaşlık" olayında neyi ele verir?

```
$ ps -eo pid,ppid,stat,rss,comm --sort=-rss | awk '$3 ~ /^D/'
```

`stat` sütununun `D` ile başlayanları süzmek neden "I/O darboğazı" teşhisinin (Faz 4) Faz 3 tarafıdır?
Bu zincir hangi fazların araçlarını birleştiriyor?

---

## Cevap Anahtarı

Her cevabın sonunda o sorunun **hangi fazların kesişiminde** durduğu belirtilmiştir.

**1.** `D` durumundaki process hem **load average**'ı şişirir (Linux load'a R + D'yi sayar) hem de
`iowait`'i yükseltir (CPU boşta ama process I/O bekliyor). `kill -9` kurtarmaz çünkü process kernel'de
I/O'yu bekliyor ve sinyal ancak user space'e dönünce teslim edilir (Faz 3.2.3); doğru bakılacak yer
diskin kendisidir (`iostat`, Faz 4.5) — process kilitli değil, altındaki depolama yavaş/doygun. ·
*Faz 3.2 × Faz 4.4/4.5*

**2.** Container'ın kendi cgroup bellek limiti dolunca **grup içi** OOM olur: host'ta bol RAM olsa
bile grup kendi kotasını aştığı için çekirdek grup içinde bir process öldürür. Sistem geneli OOM ise
makinenin **tüm** RAM'i (+swap) bittiğinde olur. `dmesg`'te ayrım: grup OOM'unda "Memory cgroup out of
memory" ibaresi geçer; sistem OOM'unda geçmez. · *Faz 3.6 × Faz 4.3*

**3.** Çelişki değil, çünkü load "CPU meşgul" demek **değildir**. Load = çalışmak isteyen (R) +
kesintisiz uykuda I/O bekleyen (D) process sayısı (Faz 3.2.1). CPU %90 boşta ama load 12 ise,
process'lerin çoğu D durumundadır — CPU'yu değil **diski** bekliyorlar (`wa` yüksektir, Faz 4.4.2).
Yani makine "iş kuyruğu dolu" ama "iş disk yüzünden ilerlemiyor". · *Faz 3.2 × Faz 4.4*

**4.** En az iki neden: (a) `%MEM`/RSS **paylaşılan** sayfaları (shared libraries, CoW) her process'e
tam sayar; gerçek fiziksel kullanım daha az olabilir (Faz 4.1.3). (b) Görünen "dolu" RAM aslında page
cache olabilir — geri alınabilir, baskı yaratmaz (Faz 4.2). (c) RSS gerçek olsa bile `available`
yüksekse sistem rahattır. Kısaca: tek process'in `%MEM`'i sistem baskısını göstermez;
`available`'a bakılır. · *Faz 3.7 × Faz 4.1/4.2*

**5.** Çekirdek OOM Killer'da process'e **SIGKILL** (9) gönderir. Bu sinyal yakalanamaz (Faz 3.4.1) —
tam da bu yüzden OOM'da process "graceful shutdown" **yapamaz**, temizlik yapmadan anında ölür. OOM
acil bir bellek kurtarma işlemidir; çekirdek nazik davranmaz çünkü sistem zaten bellek çıkmazındadır.
· *Faz 3.4 × Faz 4.3*

**6.** VSZ, process'in **ayırdığı sanal** alandır; fiziksel RAM ile doğrudan ilgisi yoktur (Faz
4.1.2). JVM başlangıçta devasa sanal alan ayırır ama **lazy allocation** sayesinde bu alana
dokunulana kadar tek fiziksel sayfa bile harcanmaz — RSS 500 MB kalır. 20 GB sadece "söz verilen"
alandır; 8 GB makinede çalışması normaldir çünkü gerçekte tuttuğu 500 MB'dir. · *Faz 3.7 × Faz 4.1*

**7.** "Sağlıklı tam kapasite." Load, **core sayısına** bölünerek okunur (Faz 4.4.1): 4 vCPU'da load
8 → kabaca kapasitenin 2 katı, yani sıkışık — ama "aşırı yük" olup olmadığı `%Cpu` satırına bağlıdır.
`us` yüksekse gerçekten CPU-bound (8-thread'li uygulama 4 çekirdeği doldurup kuyruk yapmış olabilir);
`wa` yüksekse load'un kaynağı I/O'dur. Netleşme: load / `nproc` = 8/4 = 2. · *Faz 3.3 × Faz 4.4*

**8.** Thrashing'de process'ler **D** durumunda birikir: aktif bellek sayfaları swap'a inip çıkarken
process'ler disk (swap) I/O'sunu bekler (Faz 4.3.1). D-state load'a sayıldığı için load şişer (Faz
3.2.1); ama iş CPU'da değil diskte olduğu için CPU boşta kalır (`wa` yüksek). Tek cümle: *swap I/O'su
process'leri D'ye düşürür, D hem load'u şişirir hem CPU'yu boşta bırakır.* · *Faz 3.2 × Faz 4.3*

**9.** `renice` yalnızca **CPU** için sıra önceliği ayarlar (Faz 3.5.2). CPU-bound bir process'i
`renice` ile geri plana atmak işe yarar — CPU'yu diğerlerine bırakır. Ama I/O-bound bir process
CPU'yu zaten kullanmıyordur (diski bekliyordur); `renice` onun disk davranışını değiştirmez, işe
yaramaz. Disk önceliği için ayrı bir araç (`ionice`) gerekir. Ders: aracı **darboğaz türüne** göre
seç — CPU önceliği CPU darboğazını, I/O önceliği I/O darboğazını hedefler. · *Faz 3.5 × Faz 4.5/4.6*

**10.** `uptime` sana load'un yüksek olduğunu (11.5, 4 vCPU'da ~2.9×) söyler — yani bir baskı var ve
artıyor (1dk > 5dk > 15dk). Ama **ne söylemez:** bu baskının CPU mu, bellek mi, I/O mu olduğunu. Load
tek başına darboğaz türünü vermez (Faz 4.4). Bir sonraki komut `top` (veya `vmstat 1`) olmalı: `%Cpu`
satırında `us`+`sy` mi yoksa `wa` mı yüksek — bunu görmeden "instance büyütelim" demek yanlış teşhis
riskidir. · *Faz 4.4 × 4.6*

**11.** Darboğaz **I/O-bound**. `wa` %84: CPU'nun büyük kısmı boşta oturup disk I/O'sunu bekliyor
(`us`+`sy` sadece %8). Load'un kaynağı D-state process'ler (Faz 4.4.2). "Daha büyük instance" (daha
çok vCPU) bu sorunu **çözmez** — CPU zaten boşta; sorun diskte. Doğru yön: diski hızlandırmak veya
I/O'yu azaltmak. · *Faz 4.4 × 4.6*

**12.** (a) `ps -eo stat,comm | grep '^D'` — `STAT` sütununda `D` ile başlayan (kesintisiz uyku)
process'leri bulur; bunlar diski bekleyenlerdir (Faz 3.2). (b) `iostat -xz 1` — disk tarafında
`%util` (doygunluk, %100'e yakınsa disk dolu) ve `await` (istek başına bekleme ms) sütunlarına
bakılır (Faz 4.5.2). İlki "kim bekliyor", ikincisi "disk gerçekten doygun mu" sorusunu cevaplar. ·
*Faz 3.2 × Faz 4.5*

**13.** Teşhis: **disk doygun (I/O-bound darboğaz)** — `nvme0n1` %util 99, await 80 ms, ve iki process
D durumunda diski bekliyor. 502'lerle ilişki: uygulama disk I/O'sunda bloke olunca istekleri zamanında
işleyemez, yanıt süreleri uzar; önündeki yük dengeleyici/proxy zaman aşımına düşen backend'e "erişemedim"
deyip 502 döndürür (Faz 3.4.2 graceful/timeout mantığı). Yani 502'ler CPU değil, diskin uygulamayı
bekletmesinin sonucudur. · *Faz 3.2/3.4 × Faz 4.5*

**14.** Hayır, yanlış teşhis. `used` sadece 3.2 GB; 12 GB **page cache** (geri alınabilir); en önemlisi
`available` 12 GB — yani sistemde gerçek bellek baskısı yok (Faz 4.2.2). "RAM neredeyse dolu" görüntüsü
page cache yanılgısıdır. Belleğin darboğaz **olmadığını** kanıtlayan sütun: **`available`** (12/16 GB,
rahat). Darboğaz bellekte değil, diskte. · *Faz 4.2*

**15.** (a) `ionice`/`nice` ile yedeklemeyi düşük I/O önceliğine almak — **işe yarar**, çünkü darboğaz
I/O ve `ionice` disk önceliğini hedefler; yedekleme uygulamanın I/O'suna yol verir. (b) Daha yüksek
IOPS'li disk — **işe yarar**, çünkü doğrudan I/O tavanını (Faz 4.5.2 cloud) yükseltir. (c) vCPU olarak
büyütmek — **çözmez**, çünkü darboğaz CPU değil disk; CPU zaten `wa`'da boşta oturuyor. (c) klasik
"yanlış eksende büyütme" hatasıdır ve para israfıdır. · *Faz 3.5 × Faz 4.5/4.6*

**16.** **CPU-bound.** `r=9` (4 vCPU'da run queue'da 9 process, CPU'ya baskı), `us≈91–94` (CPU
kullanıcı kodunu çalıştırmakla dolu), `wa=0` (disk beklemesi yok), `b=0/1` (bloke process yok),
`si/so=0` (swap yok). `si/so`'nun sıfır olması **bellek** darboğazını eler; `wa`'nın sıfır olması
**I/O**'yu eler. Geriye net CPU kalır: vCPU eklemek/kodu optimize etmek mantıklı. · *Faz 4.4/4.6*

**17.** Bu çıktı 16'dan tam tersi: `us` düşük (4), ama `wa=81` (I/O bekleme) **ve** `si/so` sıfır değil
(520/610, aktif swap). Yani aynı anda **iki darboğaz**: bellek baskısı (swap in/out aktif, `free` çok
düşük 90 MB) ve bunun tetiklediği I/O doygunluğu (`bi=8200`, `wa=81`, `b=6`). Kök neden büyük olasılıkla:
**bellek tükendi → sistem swap'a girdi → swap disk I/O'sunu doyurdu → thrashing.** Çözüm CPU değil,
RAM (veya belleği tüketen process). · *Faz 4.3 × 4.4/4.5*

**18.** **OOM Killer** (Faz 4.3.2). `python3` gerçekten suçlu olmayabilir — OOM Killer en yüksek
`oom_score`'lu, yani en çok RAM tutan process'i seçer; başka bir şey RAM'i tüketip `python3`'ü en
büyük kurban yapmış olabilir. Karar için: OOM anına kadar `available`'ın eğimine ve o sırada RSS'i
büyüyen process'e bak. `anon-rss` ~8 GB'ın **anonim** olması önemli çünkü anonim bellek page cache
gibi geri alınamaz — gerçek, sıkıştıran bellektir; cache olsaydı çekirdek onu bırakır, OOM
tetiklenmezdi. · *Faz 4.3 × 4.1/4.2*

**19.** "Java 20 GB RAM yiyor" yanlış — o VSZ (ayrılan **sanal** alan), fiziksel RAM değil. Gerçek RAM
kullanımını **RSS** sütunu verir: ~512 MB. VSZ'nin bu kadar büyük olması, JVM'in başlangıçta devasa
sanal heap/arena ayırmasının ve lazy allocation'ın sonucudur — dokunulmayan alan fiziksel RAM
harcamaz (Faz 4.1.2). Doğru cümle: "Java ~512 MB RSS kullanıyor; 20 GB sadece ayrılan sanal alan." ·
*Faz 3.7 × Faz 4.1*

**20.** Makine **sağlıklı.** Load 0.80 (4 vCPU'da ~%20 kullanım — çok rahat); `wa` 1.0 (disk beklemesi
yok denecek kadar az); `us` 12 (hafif CPU işi). `3 zombie` tek başına alarm **değildir** — zombie'ler
kaynak tüketmez, sadece process tablosunda küçük bir kayıttır (Faz 3.2.2). Alarm olurdu **eğer** sayı
sürekli **artıyorsa** (bir ebeveyn çocuklarını `wait` etmiyor demektir) — o zaman ebeveyni aranır.
Sabit 3 zombie zararsızdır. · *Faz 3.2 × Faz 4.4*

**21.** Zincir: `ps` tüm process'leri PID/PPID/durum/RSS/isim sütunlarıyla RSS'e göre sıralı listeler,
`awk '$3 ~ /^D/'` üçüncü sütunun (`stat`) `D` ile başlayanlarını süzer — yani **kesintisiz uykuda,
disk bekleyen** process'leri verir. Bu, "I/O darboğazı" teşhisinin (Faz 4.5) **Faz 3 tarafıdır**:
darboğaz metrikte `wa` olarak görünür (Faz 4), ama onu **yaratan** somut process'ler D durumundadır
(Faz 3.2). Zincir Faz 1 (pipe/awk metin işleme) + Faz 3 (process durumu) + Faz 4 (I/O teşhisi)
araçlarını birleştirir. · *Faz 1 × Faz 3 × Faz 4*

---

## Puanlama

| Doğru sayısı | Değerlendirme |
|---|---|
| 19–21 | Process ve kaynak modelini tek bir teşhis refleksine bağlamışsın. Faz 5'e güvenle geç. |
| 15–18 | Sağlam. Kaçırdığın soruların işaret ettiği **köprüye** (tek faza değil) bir tur dön. |
| 10–14 | İki fazı ayrı ayrı biliyorsun ama arasındaki bağ zayıf. Aşağıdaki tabloyu kullan. |
| 0–9 | Faz 3 ve 4'ü, özellikle "Bu faz bozulunca" ve "Kapanış" bölümlerini tekrar et; ara sınav bu köprüler oturmadan geçilmez. |

**Hangi soruyu kaçırırsan nereye dön:**

| Kaçırdığın soru | Dön — bu köprü zayıf |
|---|---|
| 1, 3, 8, 21 | Faz 3.2 × Faz 4.4 — D-state, load average, iowait |
| 2, 5, 18 | Faz 3.4/3.6 × Faz 4.3 — sinyaller, cgroup, OOM Killer |
| 4, 6, 19 | Faz 3.7 × Faz 4.1 — RSS/VSZ, sanal bellek, shared memory |
| 10, 11, 16, 17 | Faz 4.4/4.6 — load'u %Cpu ile okumak, darboğaz triyajı |
| 12, 13, 15 | Faz 3.2 × Faz 4.5 — D-state process'i diske bağlamak, iostat |
| 7, 9 | Faz 3.3/3.5 × Faz 4.4/4.6 — vCPU, öncelik, darboğaz türüne göre araç |
| 14, 20 | Faz 4.2 × Faz 3.2 — page cache/available, zombie yanılgıları |

---

## Kapanış — buradan Faz 5'e

Faz 3 ve 4 birlikte sana bir sunucunun **canlı** resmini okumayı öğretti: sistem şu an ne yapıyor
(process durumları) ve kaynağı ne yiyor (bellek, I/O, darboğaz türü). Artık "yavaş" tek kelimesini
dört ayrı teşhise ayırabiliyor, bir metriğe tepki vermeden önce onu doğrulayan ikinci metriğe
bakıyorsun. Ama hâlâ bir şeyi sormadın: **bu makine baştan nasıl bu çalışan sisteme geldi?** Process
ağacının kökünde PID 1 = systemd'yi gördün ama systemd'nin kendisini — makineyi boot'tan çalışır
duruma nasıl getirdiğini, servisleri hangi sırayla başlattığını — daha açmadın.

Faz 5 tam buraya bağlanır: Faz 3'ün "PID 1 = systemd, ağacın kökü" cümlesini alıp üstüne "systemd
makinenin tüm yaşam döngüsünü yönetir"i koyar. Faz 4'te bir process'i OOM'dan `OOMScoreAdjust` ile
koruduk — o ayarın nereye (systemd unit dosyası) yazıldığını göreceksin. `dmesg` çekirdek olaylarını
gösteriyordu; şimdi `journalctl` ile systemd'nin merkezî log defterini okuyacaksın.

> **Devam etmeden önce:** Yukarıdaki 11. sorunun cevabını tereddütsüz verebiliyorsan — load 11.5 ama
> `wa` %84 iken darboğazın neden CPU değil disk olduğunu ve "instance büyütmenin" neden çözmediğini —
> Faz 5'e hazırsın. Veremiyorsan, Faz 4'ün 4.4 (load) ve 4.6 (triyaj) bölümlerine bir tur dön; Faz 5
> çalışan sistemin *nasıl kurulduğunu* bu teşhis zemini üstüne inşa edecek.

---

> **Navigasyon:** [◀ Faz 4 — Bellek, I/O ve Performans](Faz_4_Bellek_IO_Performans.md) · **Ara Sınav 2** · [Faz 5 — Boot, Init ve systemd ▶](Faz_5_Boot_Init_systemd.md)
