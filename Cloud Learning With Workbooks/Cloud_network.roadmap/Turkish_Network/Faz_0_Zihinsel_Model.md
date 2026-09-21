# Faz 0 — Zihinsel Model: Neden Katmanlar Var?

> **Navigasyon:** **Faz 0** · [Faz 1 — Adresleme ▶](Faz_1_Adresleme.md)

---

## Nereden geliyoruz

Hiçbir yerden — burası başlangıç. Ama elinde sandığından fazlası var: bugüne kadar sayısız kez bir tarayıcıya
adres yazdın, bir video izledin, bir dosya indirdin. Bunların hepsi çalıştı ve sen hiç düşünmedin. Bu fazın
işi, o "çalışıyor" hissini bir **modele** çevirmek.

Bir uyarı ile başlayalım, çünkü bu kitabın tamamının tonunu belirler: **ağ öğrenmek protokol ezberlemek
değildir.** OSI'nin yedi katmanını sırayla sayabilen ama "bu servise bağlanamıyorum" dendiğinde nereye
bakacağını bilmeyen çok insan var. Fark, ezberde değil **zihinsel modeldedir**. Bu kitap ağı bir protokol
listesi olarak değil, bir **arıza haritası** olarak kurar: her konunun yanında "bu bilgi bozulunca sistemde
nasıl görünür?" sorusu vardır.

Eğer daha önce Linux tarafında çalıştıysan, oradaki "bir kaynağın **var olması** ile **erişilebilir olması**
ayrı şeylerdir" dersini hatırla. Ağ, o dersin en saf hâlidir: bir sunucu ayakta olabilir, servis çalışıyor
olabilir, ve paket yine de sana hiç ulaşmayabilir. Nerede öldüğünü bulmak, bu kitabın tek amacıdır.

## Bu fazın sorusu

> *"`ping` yazdığımda o veri makinemden çıkıp karşı tarafa nasıl gidiyor — ve yolda kim, neyi, ne zaman
> ekliyor?"*

Bu soru masum görünür ama cevabı bütün kitabın iskeletidir. Çünkü veri **tek parça hâlinde uçmaz**: her
katman ona kendi bilgisini ekler, karşı tarafta her katman kendi eklediğini geri söker. Bu sarma-açma
döngüsünü içselleştirmeden hiçbir troubleshooting içgüdüsü oluşmaz — çünkü bir arıza her zaman **belirli
bir katmanda** olur, ve belirti de o katmanın imzasını taşır.

Şöyle düşün: "internet çalışmıyor" bir teşhis değildir, bir **şikâyettir**. Bir mühendis onu şuna çevirir:
"L2'de mi (komşumu bulamıyorum), L3'te mi (hedefe giden yol yok), L4'te mi (bağlantı kurulmuyor), L7'de mi
(bağlantı var ama uygulama hata veriyor)?" Bu çeviriyi yapabilmek için önce katmanların **neden** var
olduğunu anlaman gerekir. Faz 0 tam olarak budur.

Bu fazın sonunda, bir paketin yolculuğunu kâğıda kendi kelimelerinle çizebileceksin — ve bir arıza belirtisi
duyduğunda içinden otomatik olarak "bu hangi katmanın işi?" sorusu geçecek.

---

## Bu fazın sonunda

- Katmanlı mimarinin **neden** var olduğunu (problemi bölmek, katmanları bağımsız değiştirebilmek)
  örnekle açıklayabileceksin
- OSI'nin 7 katmanı ile pratikte kullanılan TCP/IP'nin 4 katmanını eşleştirebilecek; hangisinin **model**,
  hangisinin **gerçek** olduğunu bileceksin
- "Her katman sadece kendi eşiyle konuşur" (peer-to-peer illüzyonu) cümlesinin ne anlama geldiğini, ve bunun
  neden bir **illüzyon** olduğunu anlatabileceksin
- Encapsulation'ı (sarma) ve decapsulation'ı (açma) adım adım, hangi katmanın hangi header'ı eklediğini
  söyleyerek çizebileceksin
- PDU adlarını (frame → packet → segment → data) doğru katmanla eşleştirebilecek; birinin "packet düştü"
  demesiyle "frame düştü" demesi arasındaki farkı bileceksin
- Bir arıza belirtisini duyduğunda **hangi katmanda** aramaya başlayacağını söyleyebileceksin
- **Cloud:** VPC'deki bir paketin de aynı katmanlardan geçtiğini; bulutun katmanları kaldırmadığını, sadece
  **soyutladığını** anlatabileceksin

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden burada |
|---|---|---|---|
| 0.1 | Katman fikri ve modeller (OSI / TCP-IP) | `[kavram]` | Neden dilimliyoruz — tüm haritanın çatısı |
| 0.2 | Encapsulation ve decapsulation | `[mekanizma]` | **Fazın kalbi** — verinin yolculuğunun motoru |
| 0.3 | PDU'lar — verinin katmandaki adı | `[kavram]` | Doğru terim = doğru katman = doğru teşhis |
| 0.4 | Bu faz bozulunca | — | Katman karıştırmanın arıza imzaları |

> **Bu fazda nasıl çalışmalı:** Bu faz neredeyse tamamen kavramsaldır, ve bu iyi bir şeydir — burada
> öğreneceğin şey bir komut değil, bir **bakış açısıdır**. Komut kutuları (🔧) burada azdır ve hepsi 🟢
> salt-okunurdur (`ip link`, `ping`, `tcpdump` ile tek bir paket izleme). Hiçbirini çalıştırmasan da fazı
> tamamlayabilirsin; çalıştırırsan model elinde somutlaşır. **Asıl çalışma aracın kâğıt kalemdir:** 0.2'yi
> okurken bir paketin sarılışını kendi elinle çiz. Bu fazı "okuyup geçmek" mümkündür ama boşunadır —
> Faz 1'den Faz 11'e kadar her faz bu fazın diline geri döner. Buradaki 20 dakikalık çizim, ileride saatlerce
> kafa karışıklığı kazandırır.

---
---

# 0.1 Katmanlar: Neden Problemi Dilimleriz

## 0.1.1 Katman fikri: bir problemi parçalara bölmek `[kavram]`

İki bilgisayarın konuşması gerçekten zor bir problemdir. Düşün: elektrik sinyalini kabloya nasıl koyacaksın,
hangi makinenin hangi olduğunu nasıl ayıracaksın, kablo bozulursa ne yapacaksın, dünyanın öbür ucundaki bir
makineye yol nasıl bulacaksın, veri yolda kaybolursa nasıl fark edeceksin, karşı taraftaki **hangi uygulamaya**
teslim edeceksin... Bunların hepsini tek bir dev program olarak yazmaya kalksan, ortaya kimsenin
değiştiremeyeceği, hiçbir parçasını ayrı test edemeyeceğin bir canavar çıkar.

Mühendislikte bu tür problemlerin standart çözümü **katmanlamadır** (layering): büyük problemi, her biri
**tek bir işten sorumlu** dilimlere böl, ve dilimler arasına net bir arayüz koy. Her katman:

- **altındaki katmandan bir hizmet alır** ("beni bu kabloya koy"),
- **kendi işini yapar** ("ben de hedefi belirleyeyim"),
- **üstündeki katmana bir hizmet sunar** ("senin verini hedefe götürürüm").

Bunun asıl kazancı şudur: **bir katmanı, diğerlerine dokunmadan değiştirebilirsin.** Wi-Fi ile ethernet
kablosu tamamen farklı fiziksel teknolojilerdir — ama üstlerindeki IP katmanı ikisini de umursamaz. Tarayıcın
"ben Wi-Fi'da mıyım kabloda mıyım" diye sormaz. Aynı şekilde IPv4'ten IPv6'ya geçerken tarayıcıyı yeniden
yazmak gerekmedi. Katmanlama, teknolojinin parça parça evrilebilmesinin tek sebebidir.

Ve senin için asıl kazanç: **katmanlar, arızayı sınırlar.** Bir sorun neredeyse her zaman tek bir katmanda
başlar. "İnternet yok" belirtisini katmanlara böldüğünde, aramayı tüm evrende değil tek bir dilimde
yaparsın. Bu kitabın tamamı bu tek cümlenin üstüne kuruludur.

> **⚠️ Yaygın yanılgı: "Katmanlar gerçek şeylerdir, paketin içinde fiziksel olarak vardır."**
>
> Katmanlar bir **tasarım fikridir**, fiziksel bir gerçeklik değil. Kabloda akan şey sadece elektrik (veya
> ışık, veya radyo dalgası) sinyalleridir; orada "katman 3" diye bir yer yoktur. Katmanlar, o sinyalin içindeki
> **bitlerin nasıl yorumlanacağına** dair bir anlaşmadır: ilk şu kadar bit adresleme içindir, sonraki şu kadar
> bit başka bir işe... Katman, verinin üzerinde bir etiket değil; **kimin hangi biti okuyacağına** dair bir
> sözleşmedir. Bu yüzden "paketin 3. katmanına bak" demek aslında "paketin şu ofsetindeki alanları oku"
> demektir. Bunu bilmek, ileride bir `tcpdump` çıktısına baktığında neye baktığını anlamanı sağlar.

## 0.1.2 İki model: OSI'nin 7'si ve TCP/IP'nin 4'ü `[kavram]`

Katmanlama fikrini iki farklı model somutlaştırır — ve ikisinin **rolü farklıdır**, bu ayrımı baştan
netleştirelim çünkü çok karıştırılır.

**OSI modeli (7 katman)** bir **referans modeldir**: akademik, eksiksiz, öğretmek ve konuşmak için
tasarlanmış bir ortak dil. Kimse "OSI protokolleri" çalıştırmıyor; ama herkes "bu bir L2 sorunu" derken
OSI'nin numaralandırmasını kullanıyor. OSI'nin asıl işi budur: **ortak bir numara dili.**

| # | Katman | Tek cümlelik işi | Tipik örnek |
|---|---|---|---|
| 7 | Application (uygulama) | Kullanıcının gerçekten istediği iş | HTTP, DNS, SSH |
| 6 | Presentation (sunum) | Veri biçimi, şifreleme, kodlama | TLS (pratikte buraya oturtulur) |
| 5 | Session (oturum) | Konuşmayı başlat/sürdür/bitir | (pratikte ayrı ele alınmaz) |
| 4 | Transport (taşıma) | Uçtan uca güvenilirlik + hangi uygulama | TCP, UDP |
| 3 | Network (ağ) | Ağlar arası adresleme ve yol bulma | IP, ICMP |
| 2 | Data link (veri bağı) | **Aynı** yerel ağda komşuya teslim | Ethernet, Wi-Fi, ARP |
| 1 | Physical (fiziksel) | Bitleri sinyale çevir ve taşı | Kablo, fiber, radyo |

**TCP/IP modeli (4 katman)** ise **gerçekten çalışan** modeldir — internet bunun üstünde koşar. OSI'nin
üst üç katmanını tek bir "Application" katmanında toplar, alt ikisini "Network Access"te birleştirir:

| TCP/IP katmanı | Karşılığı (OSI) | Ne yapar |
|---|---|---|
| Application | 7 + 6 + 5 | Uygulamanın işi (HTTP, DNS, SSH, TLS) |
| Transport | 4 | Uçtan uca: TCP / UDP |
| Internet | 3 | IP ile ağlar arası yol |
| Network Access (Link) | 2 + 1 | Yerel ağ + fiziksel taşıma |

Pratikte mühendisler **melez** konuşur: OSI'nin numaralarını (L2, L3, L4, L7) TCP/IP'nin gerçekliğiyle
kullanırlar. "L7 load balancer", "L4 firewall", "L2 switch" — bunların hepsi OSI numarası + gerçek dünya
cihazıdır. L5 ve L6'yı neredeyse hiç duymazsın; bu yüzden bu kitap da pratikte kullanılan dört numaraya
(L2, L3, L4, L7) odaklanır.

> **💡 Cloud bağlantısı — AWS servis adları OSI numarasıyla konuşur:** Bulut konsolunda karşına çıkan
> isimler doğrudan bu numaraları kullanır. **ALB** (Application Load Balancer) bir **L7** cihazıdır: HTTP
> path'ine, header'ına bakarak karar verir. **NLB** (Network Load Balancer) bir **L4** cihazıdır: sadece
> IP + port görür, içeriğe bakmaz — bu yüzden daha hızlı ama daha "kör"dür. Bir **Security Group** L3/L4
> seviyesinde filtreler (IP + port), içeriği okumaz. Yani AWS dokümantasyonunu okuyabilmek, bu tabloyu
> bilmeyi gerektirir. İleride Faz 11'de hepsini tek tek eşleyeceğiz; şimdilik tek not: bulut, katmanları
> **kaldırmaz** — sadece bazılarını senin yerine yönetir.

## 0.1.3 Peer-to-peer illüzyonu: her katman kendi eşiyle konuşur `[kavram]`

Katmanlı modelin en zarif fikri budur ve bir kere oturduğunda ağ birden basitleşir.

Senin tarayıcın (L7), karşıdaki web sunucusunun L7'siyle konuşuyormuş gibi davranır. "Bana bu sayfayı ver"
der, sayfa gelir. Tarayıcı ne MAC adresi bilir, ne router'ların varlığından haberi vardır, ne de paketin kaç
hop geçtiğini umursar. Aynı şekilde senin makinendeki TCP (L4), karşı makinedeki TCP ile konuşur: "sana
gönderdiğim 1461. byte'ı aldın mı?" Aradaki router'lar TCP'yi hiç açıp bakmaz.

Buna **peer-to-peer iletişim illüzyonu** denir: **her katman, karşı taraftaki aynı numaralı katmanla
doğrudan konuşuyormuş gibi davranır.** Neden illüzyon? Çünkü gerçekte hiçbir veri yatay gitmez. Veri senin
makinende **aşağı iner** (L7→L4→L3→L2→L1), kablodan geçer, karşı makinede **yukarı çıkar** (L1→L2→L3→L4→L7).
Yatay ok, mantıksal bir kurgudur; dikey ok, fiziksel gerçektir.

Ama bu illüzyon boşuna değil — **teşhisin temelidir**. Çünkü şu soruyu sorabilir hâle gelirsin: "benim L4'üm
karşının L4'üne ulaşabiliyor mu?" Eğer TCP el sıkışması tamamlanmıyorsa (Faz 5), sorun L4 eşleşmesinde ya da
**altındaki** katmanlardadır — L7'yi kurcalamanın hiçbir anlamı yoktur. Bir katman, ancak altındaki katmanlar
çalışıyorsa çalışabilir. Bu tek kural, Faz 10'daki tüm troubleshooting metodolojisinin çekirdeğidir:
**aşağıdan yukarı doğrula.**

> **🤔 Düşün 0.1** — Bir meslektaşın "site açılmıyor" diyor ve sen `ping` ile karşı sunucunun IP'sine
> ulaşabildiğini görüyorsun (cevap geliyor). (a) `ping`'in çalışması hangi katmanların **sağlam** olduğunu
> kanıtlar? (b) Bu gözlem, hangi katmanları şüpheli listesinde **bırakır**? (c) Bu neden "peer-to-peer
> illüzyonu" fikrinin pratik bir uygulamasıdır?
>
> *(Cevap: fazın sonunda)*

---
---

# 0.2 Encapsulation ve Decapsulation

## 0.2.1 Sarma: her katman kendi zarfını ekler `[mekanizma]`

Şimdi fazın kalbine geldik. Yukarıda "her katman kendi işini yapar" dedik — peki bu iş **fiziksel olarak**
nasıl görünür? Cevap: her katman, üstünden aldığı veriyi bir **zarfa** koyar ve zarfın üstüne kendi bilgisini
yazar. Bu bilgiye **header** (heydır = başlık) denir, işleme de **encapsulation** (enkapsüleyşın = sarmalama)
denir.

Mektup benzetmesi burada gerçekten iyi çalışır. Bir mektup yazarsın (veri). Onu bir zarfa koyar, üstüne alıcı
adresini yazarsın (L3 — hedef nerede). O zarfı bir kargo poşetine koyar, poşetin üstüne "bu paket şu şubeye
gidecek" etiketi yapıştırırsın (L2 — bir sonraki durak). Kargo şubesi poşeti açar, etiketi atar, yeni bir
etiket yapıştırır ve bir sonraki şubeye yollar. **İçerideki zarf hiç açılmaz.** Sadece en dıştaki etiket
her durakta değişir.

Ağda da aynen böyle olur. Tarayıcın bir HTTP isteği üretir; o istek aşağı inerken her katman kendi header'ını
**önüne ekler**:

1. **L7 (Application)** — Veri üretilir: `GET /index.html HTTP/1.1` ve header'ları. Buna **data** denir.
2. **L4 (Transport)** — TCP, bu verinin önüne bir **TCP header** ekler. İçinde ne var? Kaynak port, hedef port
   (80), sıra numarası, ACK numarası, bayraklar. Artık bu birime **segment** denir. Buradaki hedef port,
   karşı makinede **hangi uygulamaya** teslim edileceğini söyler.
3. **L3 (Network)** — IP, segmentin önüne bir **IP header** ekler: kaynak IP, hedef IP, TTL, protokol
   numarası. Artık **packet** olmuştur. Buradaki hedef IP, **hangi makineye** gideceğini söyler.
4. **L2 (Data link)** — Ethernet, paketin önüne bir **ethernet header** ekler: kaynak MAC, hedef MAC, ve
   sonuna bir hata denetim alanı (FCS) koyar. Artık **frame** olmuştur. Buradaki hedef MAC, **bir sonraki
   fiziksel durağı** söyler — hedef makineyi değil.
5. **L1 (Physical)** — Frame'in bitleri sinyale çevrilir ve kabloya/havaya verilir.

Dikkat et: **veri hiç değişmedi, sadece etrafı kalınlaştı.** Kabloda akan şey, iç içe geçmiş bir dizi
zarftır. Bu yüzden ağdaki bir paket her zaman gerçek verisinden büyüktür — her katman "ek yük" (overhead)
getirir. Faz 5'te MTU'yu konuşurken bu ek yükün neden hayati olduğunu göreceksin.

![Şekil 0.1 — Encapsulation: bir HTTP isteği aşağı inerken her katmanın kendi header'ını ekleyişi (data → segment → packet → frame) ve karşı tarafta decapsulation ile aynı header'ların ters sırayla sökülüşü. Veri hiç değişmez; sadece etrafındaki zarflar eklenir ve çıkarılır.](../diagrams/png/nw-0-01-encapsulation.png)

Şekilde soldaki dikey ok senin makinendeki inişi (encapsulation), sağdaki dikey ok karşı makinedeki çıkışı
(decapsulation) gösterir. Ortadaki yatay ok, aslında **olmayan** ama her katmanın olduğunu varsaydığı
peer-to-peer bağlantıdır (0.1.3). Her satırdaki kutu, o katmanda verinin aldığı yeni adı taşır.

> **🔧 Makinende gör** 🟢 — bir frame'in içindeki katmanları aynı anda gör
>
> ```
> $ sudo tcpdump -n -c 1 -v icmp
> listening on ens5 ...
> 14:02:11.377 IP (tos 0x0, ttl 64, id 4711, proto ICMP (1), length 84)
>     172.31.20.15 > 8.8.8.8: ICMP echo request, id 3, seq 1, length 64
> ```
>
> Tek satırda üç katman birden görünüyor: `IP (... ttl 64 ... proto ICMP)` **L3** header'ıdır;
> `172.31.20.15 > 8.8.8.8` L3'ün kaynak/hedef adresleridir; `ICMP echo request` ise L3'ün **içindeki**
> yüktür. `length 84` = IP header (20) + ICMP yükü (64). `tcpdump` tam olarak bunu yapar: dıştan içe zarfları
> açıp sana gösterir. Bu fazda anlamak zorunda değilsin — sadece **zarfların gerçekten orada olduğunu** gör.
> Faz 10'da `tcpdump` senin nihai hakemin olacak.

## 0.2.2 Açma: karşı tarafta ters yönde sökülür `[mekanizma]`

Karşı makineye ulaşan şey bir bit akışıdır. Orada tam tersi olur — buna **decapsulation** (dekapsüleyşın =
zarfları açma) denir:

1. **L1** bitleri sinyalden geri okur ve L2'ye verir.
2. **L2** ethernet header'ına bakar: "hedef MAC benim mi?" Değilse frame'i **atar** (switch'ler bunu farklı
   yapar — Faz 3). Benimse header'ı söker, FCS ile bütünlüğü denetler, ve içindekini L3'e verir. Header'daki
   bir alan "içeride IP var" der; L2 bu sayede kime vereceğini bilir.
3. **L3** IP header'ına bakar: "hedef IP benim mi?" Değilse ya atar ya da (router ise) yönlendirir — Faz 4.
   Benimse header'ı söker ve `proto` alanına bakar: TCP mi, UDP mi, ICMP mi? İçindekini doğru L4'e verir.
4. **L4** TCP header'ına bakar: **hedef port** kaç? 80 ise, 80'i dinleyen process'e (Faz 1.4) teslim eder.
   Sıra numaralarını denetler, gerekirse ACK üretir (Faz 5).
5. **L7** sonunda saf veriyi alır: `GET /index.html HTTP/1.1`. Web sunucusu artık işini yapabilir.

Buradaki tek fikir çok güçlüdür: **her katmanın header'ı, bir sonraki katmanın kim olduğunu söyler.** L2
header'ı "içeride IP var" der, L3 header'ı "içeride TCP var" der, L4 header'ı "hedef port 80" der. Zincir
böyle çözülür. Bu, zarfın üstündeki "kime" bilgisinin her seviyede bir kademe daha keskinleştiği bir
huniye benzer: **hangi makine → hangi protokol → hangi uygulama.**

> **⚠️ Yaygın yanılgı: "Yol boyunca her cihaz tüm katmanları açar."**
>
> Hayır — ve bu, ağın nasıl hızlı olabildiğinin cevabıdır. Aradaki cihazlar **sadece ihtiyaç duydukları
> katmana kadar** açar. Bir **switch** (L2 cihazı) sadece ethernet header'ına bakar; içerideki IP'yi hiç
> görmez, umursamaz. Bir **router** (L3 cihazı) L2'yi söküp IP header'ına bakar, kararını verir, sonra
> **yeni** bir L2 header'ı takıp yollar — ama TCP'yi hiç açmaz. TCP header'ını sadece **iki uç** okur.
> Bu yüzden TCP'ye "uçtan uca" (end-to-end) protokol denir: aradakiler onu taşır ama okumaz. Ve bu yüzden
> bir TCP sorununu router loglarında aramak boşunadır. (Faz 9'da göreceğin **stateful firewall**'lar bu
> kuralın bilinçli istisnasıdır — tam da bu yüzden özel bir cihaz sayılırlar.)

> **❓ Akla gelen soru: "Router L2 header'ını söküp yenisini takıyorsa, hedef MAC adresi yolda değişiyor mu?"**
>
> Evet — ve bu, Faz 3 ile Faz 4'ün tamamının anahtarıdır. **Kaynak ve hedef IP** (L3) yolculuk boyunca
> genelde **sabit** kalır: uçtan uca kimlik onlardır. Ama **kaynak ve hedef MAC** (L2) **her hop'ta
> değişir**: çünkü MAC, "bir sonraki fiziksel durak" demektir, "nihai hedef" değil. Kargo benzetmesinde:
> zarfın üstündeki ev adresi hiç değişmez, ama her şubede poşetin üstündeki "bir sonraki şube" etiketi
> yenilenir. Bu ayrımı şimdi yerleştir — Faz 1'de MAC'in neden sadece yerel ağda anlamlı olduğunu, Faz 3'te
> ARP'nin neden var olduğunu, Faz 4'te router'ın tam olarak ne yaptığını bu cümle üzerine kuracağız.

> **🤔 Düşün 0.2** — Bir HTTP isteğinin verisi 100 byte. Bu veri kablodan çıkarken toplam boyutu 100
> byte'tan **büyük** olacaktır. (a) Hangi üç header eklendiği için? (b) Karşı tarafta bu header'lar hangi
> sırayla sökülür? (c) Aradaki bir switch bu üç header'dan kaçını okur, bir router kaçını okur?
>
> *(Cevap: fazın sonunda)*

---
---

# 0.3 PDU'lar — Verinin Katmandaki Adı

## 0.3.1 Frame, packet, segment, data: dört isim, tek veri `[kavram]`

Yukarıda fark etmiş olabilirsin: aynı veri her katmanda **farklı bir isim** aldı. Bu isimlere **PDU**
(Protocol Data Unit — protokol veri birimi) denir. Tablo kısa ama ezberden fazlasını hak ediyor:

| Katman | PDU adı | Türkçe okunuşu | İçinde ne var | Adresi ne |
|---|---|---|---|---|
| L7 Application | **Data** | *deyta* | Saf uygulama verisi | — |
| L4 Transport | **Segment** (TCP) / **Datagram** (UDP) | *segmınt* | TCP/UDP header + data | Port (hangi uygulama) |
| L3 Network | **Packet** | *paket* | IP header + segment | IP (hangi makine) |
| L2 Data link | **Frame** | *freym* | Ethernet header + packet + FCS | MAC (hangi komşu) |
| L1 Physical | **Bit** | *bit* | Sinyale çevrilmiş 1'ler ve 0'lar | — |

Neden aynı veriye dört isim? Çünkü **isim, o an hangi zarfın en dışta olduğunu söyler.** "Frame" dediğinde
ethernet header'ı olan bir şeyden bahsediyorsundur; "packet" dediğinde IP header'ı olan bir şeyden. Yani PDU
adı bir **katman etiketidir** — ve bu, mühendisler arası konuşmanın hassasiyetini belirler.

Pratikte günlük konuşmada herkes her şeye "paket" der ("paket düştü", "paket yakala"). Bu genelde zararsızdır.
Ama teşhis anında fark kritik olur:

- **"Frame düşüyor"** → L2 sorunu: kablo, NIC, switch portu, duplex uyuşmazlığı, CRC hataları. Yerel.
- **"Packet düşüyor"** → L3 sorunu: routing, TTL, MTU, firewall, uzak bir hop. Uçtan uca yol.
- **"Segment retransmit ediliyor"** → L4 sorunu: kayıp + yeniden gönderim, congestion. Güvenilirlik katmanı.

Üçü de "veri kayboluyor" belirtisi verir ama üçünün **bakılacak yeri farklıdır**. Bir mühendisin "packet
loss var" demesiyle "CRC error sayacı artıyor" demesi arasındaki fark, aramayı kilometrelerce daraltır.

> **💡 Cloud bağlantısı — bulutta hangi PDU'yu görürsün:** Bulutta L1 ve L2'yi genelde **hiç görmezsin** —
> kablo yok, switch yok, NIC sanal. Bu iyi bir haber: kablo/duplex sınıfı arızalar hayatından çıkar. Ama
> bir bedeli var: **L2'yi göremediğin için teşhis araçların L3'ten başlar.** VPC Flow Logs (Faz 10.4) sana
> **packet** seviyesinde kayıt verir — kaynak IP, hedef IP, port, ACCEPT/REJECT. Frame göremezsin, çünkü
> bulut sağlayıcı o katmanı senin adına yönetir. Yani bulutta "paket" demek çoğu zaman gerçekten doğrudur;
> gördüğün en alt birim odur.

> **🤔 Düşün 0.3** — Bir sunucunun NIC istatistiklerinde "CRC error" sayacının arttığını görüyorsun. Aynı
> anda uygulama ekibi "TCP retransmit çok yüksek" diyor. (a) CRC hatası hangi PDU/katmanla ilgilidir?
> (b) TCP retransmit hangi PDU/katmanla? (c) İkisi arasında bir **neden-sonuç** ilişkisi kurulabilir mi —
> hangisi hangisine yol açar?
>
> *(Cevap: fazın sonunda)*

---
---

# 0.4 Bu Faz Bozulunca — Katman Karıştırmanın İmzaları

Faz 0 bir komut fazı değil, bir **model** fazıdır — dolayısıyla "bozulması" da farklı görünür. Burada bozulan
şey sistemin kendisi değil, **senin haritandır**: yanlış katmanda aramak. Aşağıdaki tablo, bu fazın
kavramları oturmadığında sahada nasıl vakit kaybedildiğini gösterir.

| Belirti / davranış | Altında yatan model hatası | Doğru refleks | İlgili bölüm |
|---|---|---|---|
| "İnternet yok" deyip modem resetlemek | Katman ayrımı yok; tek bir "internet" var sanılıyor | Önce hangi katman: L2 mi L3 mü L4 mü L7 mi | 0.1.1 |
| `ping` çalışıyor diye "ağ sağlam" demek | `ping` L3'ü test eder, L4/L7'yi değil | L3 sağlam ≠ servis erişilebilir | 0.1.3 |
| Uygulama hatasını router loglarında aramak | Router'ın L4/L7'yi okuduğu sanılıyor | Router L3'te durur; L4 uçtan ucadır | 0.2.2 |
| "Paket büyüdü" sürprizi (MTU aşımı) | Header'ların ek yük getirdiği unutulmuş | Her katman byte ekler; toplam boyutu hesapla | 0.2.1 |
| MAC adresini uzak sunucu için aramak | MAC'in uçtan uca sanılması | MAC yereldir, her hop'ta değişir; IP uçtan uca | 0.2.2 |
| "Packet loss" deyip kablo değiştirmek | PDU adları katmanla eşleşmiyor | Frame mi packet mi segment mi — hangisi düşüyor | 0.3.1 |
| Sorunu L7'den (uygulama) aramaya başlamak | Alt katmanlar doğrulanmadan üste çıkılıyor | Aşağıdan yukarı doğrula: link → IP → yol → port → app | 0.1.3 |

> **Bu tablodan çıkan ders:** Bu fazın tek çıktısı bir refleks: **"bu hangi katmanın işi?"** Bir belirti
> duyduğunda ilk yapacağın şey onu bir katmana oturtmaktır — çünkü katman, aramanın **yerini** belirler.
> İki kural bunu taşır: (1) **Bir katman, ancak altındaki katmanlar çalışıyorsa çalışır** — bu yüzden teşhis
> daima aşağıdan yukarı yapılır (Faz 10'un tüm metodolojisi budur). (2) **Terimler katman etiketidir** —
> "frame", "packet", "segment" demek, aramayı farklı yerlere yollar; gelişigüzel kullanma. Bu iki kuralı
> şimdi al; Faz 1'den itibaren her fazda üstüne et bağlayacağız.

---
---

# Faz 0 — Düşün sorularının cevapları

## Cevap 0.1 — `ping` L3'ü kanıtlar, L4 ve L7'yi şüpheli bırakır

(a) `ping` bir **ICMP** mesajıdır ve **L3**'te çalışır. Cevabın gelmesi şunları kanıtlar: fiziksel bağlantı
var (**L1**), yerel ağda komşuna/geçidine ulaşabiliyorsun (**L2**), ve hedef IP'ye giden **yol** iki yönde de
çalışıyor (**L3** — gidiş *ve* dönüş, çünkü cevap geri geldi). Yani alt üç katman sağlamdır.

(b) Şüpheli kalan katmanlar **L4 ve L7**'dir: hedef makine ağda var ama (i) servis o portu dinlemiyor
olabilir, (ii) bir firewall o **portu** kapatıyor olabilir (ICMP'ye izin verip TCP 443'ü kesen kural çok
yaygındır), (iii) TCP el sıkışması tamamlanmıyor olabilir, (iv) bağlantı kuruluyor ama uygulama hata
döndürüyor olabilir. `ping`'in başarısı bunların **hiçbirini** dışlamaz.

(c) Bu, peer-to-peer illüzyonunun doğrudan pratik hâlidir: `ping` ile senin **L3'ün**, karşının **L3'üyle**
konuşabildiğini test ettin — sadece o eşleşmeyi. Her katman kendi eşiyle ayrı ayrı "konuşur", dolayısıyla
her katman ayrı ayrı **test edilir**. Bir katmanın çalışması, üstündekilerin çalıştığını asla kanıtlamaz;
sadece altındakilerin çalıştığını kanıtlar.
**İlgili bölüm:** 0.1.3 · **Devamı:** Faz 10.1 (katman-katman metodoloji).

## Cevap 0.2 — Üç header eklenir, ters sırayla sökülür, aradakiler farklı derinliğe bakar

(a) Üç header eklenir: **TCP header** (L4 — kaynak/hedef port, sıra no), **IP header** (L3 — kaynak/hedef IP,
TTL), **Ethernet header** (L2 — kaynak/hedef MAC; ayrıca sona FCS eklenir). Yani 100 byte'lık veri, kabloya
çıkarken her katmanın ek yüküyle birlikte belirgin şekilde büyür. Bu "büyüme" Faz 5.7'de MTU'yu konuşurken
kritik olacak: bir frame'in taşıyabileceği toplam boyut sınırlıdır ve header'lar o bütçeden yer yer.

(b) Ters sırayla, **dıştan içe**: önce Ethernet header (L2) sökülür, sonra IP header (L3), sonra TCP header
(L4), en sonda saf veri L7'ye teslim edilir. Sıra rastgele değil zorunludur — her header, bir sonrakinin
ne olduğunu söyler.

(c) Bir **switch** (L2) sadece **bir** header okur: ethernet header'ı (hedef MAC). İçerideki IP'yi hiç
görmez. Bir **router** (L3) **iki** header okur: ethernet header'ını söker, IP header'ına bakıp kararını
verir — sonra yeni bir ethernet header takıp yollar. TCP header'ını **ikisi de okumaz**; onu sadece iki uç
makine okur.
**İlgili bölüm:** 0.2.1-0.2.2 · **Devamı:** Faz 5.7 (MTU), Faz 3-4 (switch ve router).

## Cevap 0.3 — CRC L2'dir, retransmit L4'tür, ve biri diğerine yol açar

(a) **CRC error** bir **L2 / frame** olayıdır: ethernet frame'inin sonundaki FCS alanı, frame'in yolda
bozulup bozulmadığını denetler. Tutmuyorsa frame **atılır**. Sebepleri fizikseldir: kötü kablo, gevşek konektör,
elektriksel gürültü, arızalı NIC/switch portu, duplex uyuşmazlığı.

(b) **TCP retransmit** bir **L4 / segment** olayıdır: gönderilen bir segment için ACK gelmediğinde TCP onu
yeniden gönderir (Faz 5.3). TCP, verinin kaybolduğunu bilir ama **neden** kaybolduğunu bilmez.

(c) Evet, net bir neden-sonuç var ve yön **aşağıdan yukarıdır**: L2'de atılan her frame, içindeki paketi ve
onun içindeki segmenti de götürür. TCP o segment için ACK alamaz, zaman aşımına uğrar ve **yeniden gönderir**.
Yani **CRC hataları sebep, retransmit'ler sonuçtur.** Doğru müdahale L4'te (TCP ayarı) değil **L2'dedir**:
kabloyu/portu/NIC'i düzelt. Bu, "aşağıdan yukarı teşhis" kuralının en temiz örneğidir — üst katmandaki
gürültülü belirti, alt katmandaki sessiz arızanın gölgesidir.
**İlgili bölüm:** 0.3.1, 0.1.3 · **Devamı:** Faz 5.3 (retransmission), Faz 10.1.

---
---

# Faz 0 — Sık sorulan sorular

**S1 — OSI'yi gerçekten ezberlemem gerekiyor mu?** Yedi katmanı sırayla saymak bir sınav becerisidir; işe
yarayan şey **numaraların ne anlama geldiğidir**. Pratikte dört numarayı içselleştirmen yeter: **L2** yerel
ağ/MAC, **L3** IP ve yol, **L4** port ve güvenilirlik, **L7** uygulama. L5 ve L6'yı sahada neredeyse hiç
duymazsın (TLS genelde "L6 civarı" diye geçiştirilir) (0.1.2).

**S2 — TCP/IP modeli varken OSI neden hâlâ kullanılıyor?** Çünkü OSI bir **konuşma dilidir**. Cihazlar
TCP/IP çalıştırır, insanlar OSI numaralarıyla konuşur: "L7 load balancer", "L2 switch", "L3 routing". İkisi
rakip değil — biri gerçeklik, diğeri ortak terminoloji (0.1.2).

**S3 — Encapsulation veriyi şifreler mi?** Hayır, ikisi tamamen farklı şeylerdir. Encapsulation sadece
**sarar**: verinin önüne adresleme ve kontrol bilgisi ekler, içeriğe dokunmaz — `tcpdump` ile içeriği
okuyabilirsin. Şifreleme ayrı bir iştir ve genelde **L7 civarında** (TLS ile) yapılır (Faz 8.4). Şifrelenmiş
bir HTTPS trafiğinde bile IP ve TCP header'ları **açıktır**; sadece taşınan yük şifrelidir (0.2.1).

**S4 — Neden hem IP hem MAC adresi var, biri yetmez mi?** İkisi farklı sorulara cevap verir. **MAC** "bu
yerel ağdaki hangi fiziksel cihaz" der ve **her hop'ta değişir**; **IP** "internetteki hangi makine" der ve
uçtan uca **sabit** kalır. IP hiyerarşiktir (gruplanabilir, route edilebilir), MAC düzdür (gruplanamaz).
Tek başına MAC ile internet ölçeğinde yol bulunamaz — bu, Faz 1 ve Faz 4'ün konusu (0.2.2).

**S5 — Frame, packet, segment arasındaki farkı bir cümlede?** Aynı verinin farklı zarf katmanlarındaki
adları: **segment** = TCP header'lı (L4), **packet** = IP header'lı (L3), **frame** = ethernet header'lı (L2).
İsim, o an **en dıştaki header'ın** hangi katmana ait olduğunu söyler (0.3.1).

**S6 — Bulutta katmanlar da var mı, yoksa AWS bunları kaldırdı mı?** Hepsi var — AWS onları **kaldırmaz**,
bir kısmını senin yerine **yönetir**. VPC içindeki bir paket de aynen encapsulate edilir, aynı header'ları
taşır. Fark: L1/L2'yi sen görmezsin ve yönetmezsin. Bu yüzden bulut teşhisin L3'ten başlar (Flow Logs
packet seviyesindedir) (0.3.1).

**S7 — "Bu bir L3 sorunu" cümlesini ne zaman kurabilirim?** Belirti **adresleme veya yol bulma** ile
ilgiliyse: hedefe hiç ulaşılamıyor, `ping` cevapsız, TTL exceeded geliyor, farklı bir ağa çıkılamıyor ama
yerel ağ çalışıyor. Belirti "bağlantı kuruluyor ama veri akmıyor" ise L4'e, "bağlanıyor ama hata dönüyor"
ise L7'ye bakarsın. Bu eşleştirmeyi Faz 10 tam bir karar ağacına çevirecek (0.1.3, 0.4).

---
---

# Faz 0 — Kendini sına

Cevaplarını bir kâğıda yaz, sonra cevap anahtarıyla karşılaştır. Hedef: 18 sorudan 14+.

## Bölüm A — Tanım ve mekanizma

1. Katmanlı mimarinin iki temel faydasını yaz (biri tasarım, biri teşhis ile ilgili olsun).
2. OSI'nin 7 katmanını sırayla say ve her birinin işini **tek kelimeyle** yaz.
3. TCP/IP'nin 4 katmanı OSI'nin hangi katmanlarına karşılık gelir? Tablo hâlinde eşleştir.
4. "Peer-to-peer illüzyonu" ne demektir, ve neden bir **illüzyondur**?
5. Encapsulation sırasında L4, L3 ve L2'nin eklediği header'larda **sırasıyla** hangi adres/kimlik bilgisi
   bulunur?
6. Decapsulation'da bir katman, içindekini hangi katmana vereceğini nasıl bilir?
7. Dört PDU adını (data, segment, packet, frame) katmanlarıyla eşleştir ve her birinin adres tipini yaz.
8. Bir router ile bir switch, gelen bir frame'de kaç header'a kadar açar? Farkı açıkla.

## Bölüm B — Uygula ve teşhis et

9. `ping 8.8.8.8` çalışıyor ama `curl https://8.8.8.8` bağlanamıyor. Hangi katmanlar sağlam, hangileri
   şüpheli?
10. Bir NIC'te CRC error sayacı artıyor. Bu hangi katmanın arızasıdır, ve üst katmanda hangi belirti olarak
    görünür?
11. 200 byte'lık bir veri gönderiyorsun ama kabloda ölçülen boyut çok daha büyük. Nedenini ve hangi
    header'ların eklendiğini yaz.
12. Bir paket İstanbul'dan Frankfurt'a 12 hop geçerek gidiyor. Yolculuk boyunca **kaç kez** hedef IP değişti,
    **kaç kez** hedef MAC değişti?
13. Bir meslektaşın "uygulama yavaş, router'ın TCP ayarlarına bakalım" diyor. Bu cümlede hangi model hatası
    var?
14. "Paket düşüyor" diyen birine, aramayı daraltmak için soracağın tek soruyu yaz (ipucu: PDU adı).

## Bölüm C — Muhakeme ve bağlantı

15. Bir katmanın çalışması, üstündeki katmanların çalıştığını kanıtlar mı? Altındakiler için ne söyler?
    Bu kural teşhis sırasını nasıl belirler?
16. HTTPS kullanıldığında IP ve TCP header'ları da şifrelenir mi? Cevabın, bir firewall'un (Faz 9) nasıl
    çalışabildiğini nasıl açıklar?
17. Bulutta L1/L2'yi göremiyorsun. Bu, hangi arıza sınıfını hayatından çıkarır ve hangi teşhis aracını
    başlangıç noktan yapar?
18. Katmanlama olmasaydı IPv4'ten IPv6'ya geçmek neden çok daha zor olurdu? Bir cümleyle açıkla.

---

## Cevap anahtarı

1. (i) **Tasarım:** bir katman diğerlerine dokunmadan değiştirilebilir (Wi-Fi ↔ ethernet, IPv4 ↔ IPv6);
   (ii) **Teşhis:** arıza tek bir katmanda sınırlanır, arama alanı daralır (0.1.1). — 2. 7 Application (iş),
   6 Presentation (biçim), 5 Session (oturum), 4 Transport (güvenilirlik), 3 Network (yol), 2 Data link
   (komşu), 1 Physical (sinyal) (0.1.2). — 3. Application = 7+6+5; Transport = 4; Internet = 3; Network
   Access = 2+1 (0.1.2). — 4. Her katman karşıdaki aynı numaralı katmanla doğrudan konuşuyormuş gibi
   davranır; illüzyondur çünkü veri gerçekte yatay gitmez — bir uçta aşağı iner, diğerinde yukarı çıkar
   (0.1.3). — 5. L4: **port** (hangi uygulama); L3: **IP** (hangi makine); L2: **MAC** (hangi komşu/sonraki
   durak) (0.2.1). — 6. Her header, içinde hangi protokolün taşındığını belirten bir alan taşır (L2 "içeride
   IP var", L3 `proto` alanı "içeride TCP var", L4 hedef portu "hangi uygulama") (0.2.2). — 7. Data = L7
   (adres yok); segment = L4 (port); packet = L3 (IP); frame = L2 (MAC) (0.3.1). — 8. Switch **bir** header
   açar (L2/ethernet); router **iki** (L2'yi söker, L3'e bakar, yeni L2 takar). İkisi de L4'ü okumaz (0.2.2).

9. Sağlam: L1, L2, L3 (ICMP gidip dönüyor). Şüpheli: **L4** (port kapalı / firewall portu kesiyor / TCP el
   sıkışması olmuyor) ve **L7** (TLS/uygulama hatası) (0.1.3). — 10. **L2 / frame** arızası (kablo, konektör,
   NIC, switch portu, duplex). Üstte **TCP retransmit** artışı olarak görünür — sebep L2, sonuç L4 (0.3.1).
   — 11. Her katman kendi header'ını ekler: TCP header (L4) + IP header (L3) + ethernet header ve FCS (L2).
   Bu ek yük her paketin sabit maliyetidir (0.2.1). — 12. Hedef **IP hiç değişmedi** (0 kez — uçtan uca
   sabit); hedef **MAC her hop'ta değişti** (12 kez — her adımda bir sonraki durak) (0.2.2). — 13. Router
   L3'te durur, TCP header'ını **okumaz**; TCP uçtan uca bir protokoldür ve sadece iki uç onu yorumlar.
   Aranacak yer router değil, uç makinelerdir (0.2.2). — 14. "**Frame** mi, **packet** mi, **segment** mi
   düşüyor?" — yani kaybı hangi sayaçta görüyorsun: NIC CRC/drop (L2), hop'ta kayıp (L3), TCP retransmit
   (L4)? Cevap, bakılacak yeri belirler (0.3.1).

15. Hayır — bir katmanın çalışması **üstündekiler** hakkında hiçbir şey söylemez; ama **altındakilerin**
    çalıştığını kanıtlar. Bu yüzden teşhis **aşağıdan yukarı** yapılır: alt katmanı doğrulamadan üste
    çıkmak, kanıtsız tahmin yürütmektir (0.1.3, 0.4). — 16. Hayır. TLS **yükü** şifreler; IP ve TCP
    header'ları **açık** kalır. Bir firewall tam bu yüzden çalışabilir: içeriği hiç görmeden kaynak/hedef IP
    ve portu okuyup karar verir (Faz 9'daki 5-tuple) (0.2.1, S3). — 17. Kablo/konektör/duplex/CRC sınıfı
    **L2 fiziksel arızaları** hayatından çıkar; başlangıç teşhis aracın L3 seviyesindeki kayıt olur (VPC
    Flow Logs — packet seviyesi) (0.3.1). — 18. Çünkü IP katmanını değiştirmek, üstündeki TCP'yi ve tüm
    uygulamaları da yeniden yazmayı gerektirirdi; katmanlama sayesinde sadece L3 değişti, L4 ve L7 aynı
    kaldı (0.1.1).

## Puanlama

| Doğru sayısı | Ne anlama geliyor |
|---|---|
| 16-18 | Model oturdu. Artık her belirtiyi bir katmana oturtabilirsin. Faz 1'e hazırsın. |
| 13-15 | İyi. Kaçırdığın soruların bölümlerini tekrar oku (özellikle 0.2 encapsulation). |
| 9-12 | Temel var ama kırılgan. 0.2'yi kâğıt kalemle **çizerek** tekrarla — okumak yetmez. |
| 0-8 | Fazı yeniden gez. Tek hedef: bir paketin iniş-çıkış yolculuğunu ezbersiz çizebilmek. |

Kaçırdığın soru → dönmen gereken bölüm:

| Soru | Bölüm |
|---|---|
| 1, 18 | 0.1.1 Katman fikri |
| 2, 3 | 0.1.2 OSI ve TCP/IP |
| 4, 9, 15 | 0.1.3 Peer-to-peer illüzyonu |
| 5, 11, 16 | 0.2.1 Encapsulation |
| 6, 8, 12, 13 | 0.2.2 Decapsulation ve hop'lar |
| 7, 10, 14 | 0.3.1 PDU'lar |
| 17 | 0.3.1 Cloud bağlantısı |

---
---

# Faz 0 — Kapanış ve Faz 1'e Köprü

## Bu fazdan ne taşıyorsun

Faz 0 sana bir komut değil, bir **bakış açısı** verdi: ağ, tek bir "internet" değil, birbirinin üstünde duran
ve her biri tek bir işten sorumlu **katmanlardır**. Veri bu katmanlardan inerken sarılır (encapsulation),
çıkarken açılır (decapsulation), ve her aşamada farklı bir isim alır (data → segment → packet → frame).
Her katmanın header'ı bir sonrakini işaret eder; her katman karşıdaki eşiyle konuşuyormuş gibi davranır ama
gerçekte veri hep dikey hareket eder.

En kalıcı iki kural şunlar: (1) **Bir katman, ancak altındakiler çalışıyorsa çalışır** — bu yüzden teşhis
aşağıdan yukarı yapılır. (2) **Adresler farklı ömürlüdür** — IP uçtan uca sabittir, MAC her hop'ta değişir.
Bu iki cümleyi ezberleme, **kullan**: sonraki on bir fazın tamamı bunların üstüne et bağlıyor.

## Faz 1 bunun neresine bağlanıyor

Faz 0 "veri katmanlardan geçer" dedi ve her katmanın bir **adres** taşıdığını gösterdi: L2'de MAC, L3'te IP,
L4'te port. Ama bu adreslerin **kendisini** hiç açmadık: MAC tam olarak nedir, IP nasıl yazılır, neden
"private" ve "public" diye ikiye ayrılır, bir makine IP'sini nereden alır?

Faz 1 tam bunu yapıyor: **bir makineyi ağda nasıl işaret ederiz.** Orada MAC'in neden sadece yerel ağda
anlamlı olduğunu (Faz 0'daki "MAC her hop'ta değişir" cümlesinin sebebi), IP'nin neden iki parçadan (network
+ host) oluştuğunu — ki bu, Faz 2'deki subnet'in tohumudur — ve portun neden "hangi uygulama" sorusunun
cevabı olduğunu göreceksin. Faz 0 zarfları gösterdi; Faz 1 zarfların üstündeki **adresleri** okumayı
öğretiyor.

> **🤔 Faz çıktısı — kendine sor:** Faz 0'da "MAC her hop'ta değişir, IP uçtan uca sabit kalır" dedik.
> Faz 1'e geçmeden düşün: eğer MAC her hop'ta değişiyorsa, senin makinen uzak bir sunucuya paket yollarken
> **hangi MAC adresini** yazar? Bilmediği bir MAC'i nasıl bulabilir — ve bu, hangi yeni sorunun kapısını
> aralar? (İpucu: cevap bir protokolün adıdır ve Faz 3'te göreceğiz; ama sorusu Faz 1'de doğar.)
>
> **🧪 Lab 0 fikri (kendi makinende, hepsi 🟢):** (1) `ip link` ile arayüzlerini ve MAC adreslerini gör —
> bunlar L2 kimliğin. (2) `ip addr` ile IP adreslerini gör — bunlar L3 kimliğin. İkisinin aynı arayüzde,
> iki ayrı satırda durduğuna dikkat et: bir arayüz, iki katman. (3) `ping -c 3 8.8.8.8` çalıştır ve aynı
> anda başka bir terminalde `sudo tcpdump -n -c 3 icmp` ile paketleri yakala — Faz 0'da anlattığımız
> zarfların gerçekten orada olduğunu gör. (4) Kâğıda, tarayıcına `example.com` yazdığın andan paketin
> kablodan çıkışına kadar olan iniş zincirini çiz; her kutunun yanına o katmanın eklediği header'ı ve PDU
> adını yaz. Bu dört adım, bu fazın modelini elinde somutlaştırır.

---

> **Navigasyon:** **Faz 0** · [Faz 1 — Adresleme ▶](Faz_1_Adresleme.md)
