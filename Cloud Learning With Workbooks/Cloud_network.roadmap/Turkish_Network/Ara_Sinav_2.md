# Ara Sınav 2 — Faz 3–4: Yerel Ağ ve Yönlendirme

> **Navigasyon:** [◀ Faz 4 — Yönlendirme (L3)](Faz_4_Yonlendirme_L3.md) · **Ara Sınav 2** · [Faz 5 — Transport Katmanı ▶](Faz_5_Transport_Katmani.md)

---

## Bu sınav neyi ölçüyor?

Ara sınav tek bir fazı değil, **iki faz arasındaki köprüyü** ölçer.

Faz 3 sana **yerel teslimatı** verdi: ARP, MAC tablosu, broadcast domain, VLAN — yani "komşuma nasıl
ulaşırım". Faz 4 ise **uzak teslimatı**: varsayılan geçit, yönlendirme tablosu, en uzun ön ek
eşleşmesi, TTL, traceroute — yani "komşum olmayana nasıl ulaşırım".

Gerçek bir pakette bu ikisi **iç içedir.** Bir paket uzak bir hedefe giderken L3 kararı verilir
("gateway'e göndereceğim"), ama o kararı uygulamak için **yine L2 gerekir** ("gateway'in MAC'i ne?").
Yani her yönlendirme kararının ucunda bir ARP vardır. Bu iki fazı birbirine bağlamak, ağın en temel
mekaniğini kavramak demektir — ve pek çok kafa karışıklığı tam olarak bu ekleme yerinde başlar:
*"Paket uzağa gidiyorsa neden hâlâ ARP yapıyorum?"*

**Nasıl çalışılır:**

- Kâğıt-kalemle çöz; cevabı yazmadan bir sonrakine geçme.
- Cevap anahtarı her sorunun **hangi fazların kesişiminde** durduğunu söyler — kaçırdığın soru bir
  fazı değil, iki faz arasındaki **köprüyü** işaret eder.
- 21 soru var: Bölüm A (bağlantı muhakemesi, 1–9), Bölüm B (senaryo: "bir subnet'e erişilemiyor"
  olayı, 10–15), Bölüm C (komut ve çıktı okuma, 16–21).
- Hedef süre: ~1 saat. Önemli olan süre değil, her cevabın *neden* öyle olduğunu bir cümleyle
  gerekçelendirebilmek.

> **🤔 Başlamadan önce:** Şu cümleyi kendi kelimelerinle tamamla: *"Paketimi Japonya'daki bir sunucuya
> gönderiyorum. Peki neden ilk iş olarak **yan odadaki** router'ın MAC adresini arıyorum?"* Bu tek soru
> iki fazı da içeriyor. Cevabını bir kenara yaz; Soru 1'de göreceğiz.

---

# Bölüm A — Bağlantı muhakemesi (1–9)

**1.** Hedefin `8.8.8.8` (çok uzak bir adres). Makinen paketi göndermeden önce **kimin** MAC adresini
arar ve neden hedefin MAC'ini aramaz? Faz 3'ün ARP mekanizmasıyla Faz 4'ün gateway kavramını tek
cümlede birleştir.

**2.** Faz 3'te ARP'ın **yayın** (broadcast) ile çalıştığını gördük. Faz 4'te router'ların broadcast'i
geçirmediğini söyledik. Bu ikisi birlikte neden "her router bir broadcast domain sınırıdır" sonucunu
verir — ve bu, ağı büyütürken neden bir **tasarım** meselesidir?

**3.** Bir paket üç router üzerinden geçerek hedefine ulaşıyor. Yol boyunca **kaç kez ARP** yapılır
(kabaca) ve **kaç kez IP başlığı değişir**? Cevabın Faz 0.1'in hangi ilkesini doğruluyor?

**4.** Faz 4.3'te en uzun ön ek eşleşmesini (LPM) gördük. Bir route tablosunda hem `0.0.0.0/0` hem
`10.0.0.0/8` hem `10.0.5.0/24` satırları var. Hedef `10.0.5.7`. Hangisi seçilir ve **neden** sıra
değil spesifiklik kazanır?

**5.** Faz 3.2'de switch'in MAC tablosunu **öğrenerek** doldurduğunu, Faz 4.2'de router'ın route
tablosunun **yapılandırılarak veya protokolle** dolduğunu gördük. Bu fark, iki cihazın arıza
davranışını nasıl farklılaştırır — bilmediği bir hedefle karşılaşınca switch ne yapar, router ne yapar?

**6.** Faz 4.4'te TTL'i gördük. Bir route döngüsü (loop) oluştuğunda paketler neden **sonsuza kadar**
dolaşmaz, ve bu koruma hangi katmanda çalışır? Aynı koruma L2'de (switch'ler arasında) var mıdır —
yoksa ne olur?

**7.** VLAN (Faz 3.4) ile subnet (Faz 2) arasındaki ilişkiyi kur: bir VLAN bir subnet midir? İki VLAN
arasındaki trafik neden bir **router** gerektirir ve bu, Faz 4'ün hangi kavramıyla aynı şeydir?

**8.** Faz 4.5'te traceroute'un TTL'i kademeli artırarak çalıştığını gördük. Traceroute çıktısında
ortadaki bir hop `* * *` gösteriyor ama sonraki hop'lar normal görünüyor. Bu, paketin orada
**düştüğü** anlamına gelir mi — cevabını ICMP'nin (Faz 4.4) doğasıyla gerekçelendir.

**9.** Faz 3.1'deki ARP tablosu ile Faz 4.2'deki route tablosu **ne zaman** birlikte kullanılır? Bir
`ping 10.0.5.7` komutunda çekirdeğin bu iki tabloya hangi **sırayla** baktığını yaz.

---

# Bölüm B — Senaryo: "bir subnet'e erişilemiyor" olayı (10–15)

> Bir şirket ağında iki subnet var: `10.10.1.0/24` (ofis) ve `10.10.2.0/24` (sunucular). Aralarında bir
> router var: ofis tarafında `10.10.1.1`, sunucu tarafında `10.10.2.1`. Ofisteki `10.10.1.50` makinesi,
> sunucu ağındaki `10.10.2.20`'ye erişemiyor. Aşağıdaki altı soru bu makinede sırayla yaptığın teşhis.

**10.** İlk gözlem: `ping 10.10.1.1` (kendi gateway'i) **çalışıyor**, `ping 10.10.2.20` **timeout**
veriyor. Bu ikisi birlikte hangi katmanları kesin olarak eler, ve geriye hangi ihtimaller kalır?

**11.** `ip route` çıktısında `default via 10.10.1.1` satırı **var**. `ip neigh` çıktısında
`10.10.1.1 lladdr 00:1a:2b:3c:4d:5e REACHABLE` görünüyor. Bu iki satır birlikte ne kanıtlar — ve
sorunun **bu makinede olmadığını** nasıl gösterir?

**12.** Router'a girdin. `ip route` çıktısında `10.10.2.0/24` için bir satır **var** ve arayüz doğru.
Ama sunucu ağındaki makineye ping yine gitmiyor. Sorunun hangi yönde olabileceğini düşün: gidiş mi,
dönüş mü? Bunu ayırt etmek için hangi tek gözlemi ararsın?

**13.** Sunucu makinesine (`10.10.2.20`) girdin ve `ip route` çıktısında **`default` satırı yok**;
yalnızca `10.10.2.0/24 dev eth0` var. (a) Gelen ping sunucuya ulaşıyor mu? (b) Cevap ofise dönebiliyor
mu? (c) Belirti neden "hiç çalışmıyor" gibi görünüyor?

**14.** Yönetici "ama ben sunucu ağındaki iki makine arasında ping atabiliyorum, ağ çalışıyor" diyor.
Bu gözlem neden sorunu çürütmüyor — hangi tür teslimatı test etmiş oluyor (Faz 3 mü Faz 4 mü)?

**15.** Kök neden bulundu: sunucu makinesinde varsayılan geçit tanımlı değildi. Üç çözüm öneriliyor:
(a) sunucuya `default via 10.10.2.1` eklemek, (b) sunucuya yalnızca `10.10.1.0/24 via 10.10.2.1`
spesifik rotasını eklemek, (c) router'a bir şey eklemek. Her birini değerlendir: hangisi çalışır,
hangisi daha dar kapsamlıdır, hangisi hiç işe yaramaz?

---

# Bölüm C — Komut ve çıktı okuma (16–21)

**16.** `ip neigh` çıktısı:

```
10.10.1.1   dev enp3s0 lladdr 00:1a:2b:3c:4d:5e REACHABLE
10.10.1.30  dev enp3s0 lladdr 3c:52:82:1a:4f:07 STALE
10.10.1.77  dev enp3s0  FAILED
```

Üç satırı da yorumla. `REACHABLE`, `STALE` ve `FAILED` ne anlatıyor? Üçüncü satır hangi arızayı ele
veriyor ve o makinenin ayakta olmadığını mı kanıtlar?

**17.** `ip route` çıktısı:

```
default via 192.168.1.1 dev enp3s0 proto dhcp metric 100
10.8.0.0/24 dev tun0 proto kernel scope link src 10.8.0.6
192.168.1.0/24 dev enp3s0 proto kernel scope link src 192.168.1.42
```

(a) `10.8.0.15` hedefli paket hangi satırla eşleşir ve hangi arayüzden çıkar? (b) `8.8.8.8` hedefli
paket? (c) `scope link` ne demek — neden bu satırlarda `via` yok?

**18.** `ip route get` çıktıları:

```
$ ip route get 10.8.0.15
10.8.0.15 dev tun0 src 10.8.0.6 uid 1000

$ ip route get 172.20.5.5
172.20.5.5 via 192.168.1.1 dev enp3s0 src 192.168.1.42 uid 1000
```

Bu komut neden `ip route`'tan daha kesin bir teşhis aracıdır? İki çıktı arasındaki `via` farkı ne
anlatıyor?

**19.** `traceroute` çıktısı:

```
 1  192.168.1.1        1.2 ms   0.9 ms   1.1 ms
 2  10.20.0.1         12.4 ms  11.8 ms  12.0 ms
 3  * * *
 4  * * *
 5  * * *
```

Bu çıktı 8. sorudaki durumdan nasıl farklı? Yolun 3. hop'tan sonra **hiç** cevap gelmemesi ne anlama
gelir, ve bu tek başına "paket oraya kadar gidiyor ama ötesine geçmiyor" demek midir?

**20.** İki makinenin `ip addr` ve `ip route` çıktılarından özet:

```
Makine A: 10.10.1.50/24   default via 10.10.1.1
Makine B: 10.10.1.51/25   default via 10.10.1.1
```

B'nin maskesi `/25`. (a) A, B'ye doğrudan ulaşabilir mi? (b) B, A'ya doğrudan ulaşabilir mi?
(c) Bu asimetri neden ortaya çıkıyor ve belirtisi ne olur?

**21.** `tcpdump` ile yakalanmış bir ARP alışverişi:

```
12:04:11 ARP, Request who-has 10.10.1.1 tell 10.10.1.50, length 28
12:04:11 ARP, Reply 10.10.1.1 is-at 00:1a:2b:3c:4d:5e, length 46
12:04:11 IP 10.10.1.50 > 8.8.8.8: ICMP echo request, id 1, seq 1, length 64
```

Üç satırı sırayla anlat. Üçüncü satırda hedef IP `8.8.8.8` iken, bu paketin **frame'inin** hedef MAC'i
nedir? Bu, iki fazın hangi kavramlarını aynı anda gösteriyor?

---

## Cevap Anahtarı

**1.** **Varsayılan geçidin (router'ın) MAC adresini arar.** Çünkü L3 kararı "hedef benim ağımda değil,
gateway'e göndereceğim" der (4.1.1); ama o paketi kabloya koymak için bir L2 frame'i gerekir ve frame'in
hedef MAC'i **bir sonraki fiziksel komşudur** (3.1.1). Hedefin MAC'i aranmaz çünkü ARP yalnızca **aynı
broadcast domain** içinde çalışır — `8.8.8.8` oraya cevap veremez. Tek cümle: *L3 nereye gideceğine,
L2 kime vereceğine karar verir.* · *Faz 3.1 × Faz 4.1*

**2.** ARP bir **broadcast**'tir (3.1.1) ve router'lar broadcast'i **geçirmez** (3.3.2, 4.1). Dolayısıyla
bir router'ın her arayüzü ayrı bir broadcast domain'dir. Tasarım meselesi olmasının sebebi: broadcast
domain büyüdükçe her ARP isteği **tüm** makinelere ulaşır, gereksiz trafik ve işlem yükü doğar
(broadcast storm riski). Bu yüzden büyük ağlar VLAN'lara (3.4) ve subnet'lere (Faz 2) bölünür — bölme
yalnızca adres yönetimi değil, **yayın sınırlaması** içindir. · *Faz 3.1/3.3 × Faz 4.1*

**3.** **Dört kez ARP** (her segmentte bir: kaynak→R1, R1→R2, R2→R3, R3→hedef — her biri kendi
komşusunun MAC'ini bulur) ve **IP başlığı hiç değişmez** (TTL alanı azaltılır ve sağlama yeniden
hesaplanır, ama kaynak/hedef IP sabit kalır, 4.4.1). Bu, Faz 0.1'in **katman bağımsızlığı** ilkesini
doğrular: L2 bilgisi her hop'ta yeniden üretilir, L3 bilgisi uçtan uca taşınır. · *Faz 0.1 × Faz 3.1 ×
Faz 4.4*

**4.** **`10.0.5.0/24` seçilir.** LPM kuralı, eşleşen satırlar arasında **en uzun prefix'i** (en
spesifik olanı) seçer (4.3.1). Sıra değil spesifiklik kazanır çünkü daha uzun prefix, hedefe dair daha
**kesin** bir bilgi demektir: `/24` "bu tam olarak şu 256 adreslik ağ" derken `/8` "bu 16 milyonluk
blokta bir yerde" der. Yönlendirme, elindeki en kesin bilgiyi kullanır — bu yüzden `0.0.0.0/0`
(prefix uzunluğu 0) daima **son çaredir**. · *Faz 4.3 × Faz 2.2*

**5.** Switch bilmediği bir hedef MAC ile karşılaşınca **flood eder** — frame'i giriş portu dışındaki
tüm portlara gönderir ve cevaptan öğrenir (3.2.2). Yani bilgisizliği "herkese sorarak" çözer. Router
ise bilmediği bir hedefle karşılaşınca paketi **düşürür** ve `Destination Unreachable` döndürür
(4.4.2) — tahmin etmez, flood etmez. Fark, teşhiste kritiktir: L2 arızaları genelde "çalışıyor ama
yavaş/gürültülü" görünürken, L3 arızaları "hiç çalışmıyor" görünür. · *Faz 3.2 × Faz 4.2*

**6.** **TTL** koruması sayesinde (4.4.1): her router paketi iletirken TTL'i bir azaltır ve sıfıra
inince düşürüp `Time Exceeded` gönderir. Koruma **L3'te** (IP başlığında) çalışır. **L2'de böyle bir
alan yoktur** — Ethernet frame'inde TTL yoktur. Bu yüzden switch'ler arasında bir döngü oluşursa
frame'ler sonsuza kadar dolaşır ve ağı kilitler (**broadcast storm**, 3.3.2); bunu önlemek için ayrı
bir protokol (STP) kullanılır. · *Faz 3.3 × Faz 4.4*

**7.** Pratikte **bir VLAN bir subnet'e karşılık gelir** — VLAN L2'de yayın alanını böler, subnet L3'te
adres alanını böler ve ikisi birlikte tasarlanır (3.4.1). İki VLAN arasındaki trafik router gerektirir
çünkü farklı broadcast domain'lerdir: ARP karşıya geçemez, dolayısıyla doğrudan teslimat imkânsızdır.
Bu, Faz 4'ün **inter-VLAN routing** dediği şeydir ve mekanizma olarak iki subnet arasındaki
yönlendirmeyle **birebir aynıdır**. · *Faz 3.4 × Faz 4.1/4.2*

**8.** **Hayır, düştüğü anlamına gelmez.** `* * *`, o hop'un **ICMP Time Exceeded üretmediğini** söyler
— ya yapılandırma gereği ICMP'yi kapatmıştır ya da ICMP üretmeyi düşük öncelikli sayıp atlamıştır
(4.5.2). Ama paketi **iletmeye** devam etmiştir; bunun kanıtı, sonraki hop'ların cevap vermesidir.
Traceroute'ta yalnızca **son hop'a kadar hiçbir cevap gelmemesi** gerçek bir sorun işaretidir. · *Faz
4.4 × Faz 4.5*

**9.** Sıra: **önce route tablosu, sonra ARP tablosu.** Çekirdek `ping 10.0.5.7` için (i) route
tablosuna bakar ve hedefin doğrudan mı yoksa bir gateway üzerinden mi gideceğine karar verir (4.2.1);
(ii) çıkan **bir sonraki hop adresinin** MAC'ini ARP tablosunda arar, yoksa ARP isteği yayınlar
(3.1.2). Yani L3 kararı L2 sorgusunu belirler — ters sırada çalışmaz, çünkü kimin MAC'ini arayacağını
ancak route kararından sonra bilirsin. · *Faz 3.1 × Faz 4.2*

**10.** `ping 10.10.1.1`'in çalışması **L1, L2 ve yerel L3'ü** eler: kablo, arayüz, IP yapılandırması,
ARP ve yerel switch sağlamdır — gateway'e ulaşabiliyorsun (3.1, 4.1). Timeout ise "sessiz düşüş"
demektir (Faz 9'da göreceksin). Kalan ihtimaller: router'da `10.10.2.0/24` rotası yok, **dönüş rotası**
yok, hedef makine ayakta değil, veya araya giren bir filtreleme var. · *Faz 3.1 × Faz 4.1/4.2*

**11.** Birlikte şunu kanıtlarlar: makine **gateway'i biliyor** (route tablosunda default var, 4.1.1)
**ve ona L2 seviyesinde ulaşabiliyor** (ARP çözülmüş, durum `REACHABLE`, 3.1.2). Yani bu makinenin
yapması gereken her şey tamam: paket doğru yere, doğru MAC ile çıkıyor. Sorun bu makinede değil,
**daha ileride** — router'da, hedefte veya dönüş yolundadır. Teşhisi bir sonraki hop'a taşımak için
gereken kanıt budur. · *Faz 3.1 × Faz 4.1/4.2*

**12.** Sorun büyük ihtimalle **dönüş yönündedir** (4.1.1). Ayırt etmek için aranacak tek gözlem:
**hedef makineye paket ulaşıyor mu?** Hedefte `tcpdump -ni any icmp` çalıştırıp echo request'in
görünüp görünmediğine bakarsın. Request görünüyor ama ofise cevap dönmüyorsa **gidiş sağlam, dönüş
kırık**; request hiç görünmüyorsa sorun gidiş tarafındadır (router veya araya giren bir engel).
Tek yönü test etmek daima yarım testtir. · *Faz 4.2 × Faz 4.4*

**13.** (a) **Evet, ping sunucuya ulaşıyor** — router'ın `10.10.2.0/24` rotası var ve sunucu kendi
segmentinde erişilebilir. (b) **Hayır, cevap dönemiyor:** sunucunun route tablosunda yalnızca
`10.10.2.0/24` var; `10.10.1.50` hedefi hiçbir satırla eşleşmez ve `default` da olmadığı için paket
düşürülür (4.4.2 — "no route to host"). (c) Belirti "hiç çalışmıyor" gibi görünür çünkü ofisteki
kullanıcı yalnızca **cevap gelmediğini** görür; paketinin hedefe ulaştığını göremez. Asimetrik
arızaların kafa karıştırıcı olmasının sebebi tam olarak budur. · *Faz 4.1 × Faz 4.2*

**14.** Çünkü iki sunucu **aynı subnet'tedir**: o test yalnızca **doğrudan teslimatı** (Faz 3 — ARP ve
switch) doğrular, yönlendirmeyi hiç kullanmaz. Varsayılan geçit eksikliği yerel trafiği hiç etkilemez;
sorun ancak **farklı bir ağa** gitmek gerektiğinde ortaya çıkar (4.1.1). Yani gözlem doğru ama alakasız
bir şeyi test ediyor. Bu, sahada çok sık görülen bir muhakeme hatasıdır: *yerel çalışıyor demek,
yönlendirme çalışıyor demek değildir.* · *Faz 3.1/3.2 × Faz 4.1*

**15.** (a) **Çalışır** — default rota, sunucunun bilmediği tüm hedefleri router'a gönderir; en genel
ve en yaygın çözümdür (4.1.1). (b) **Çalışır ve daha dar kapsamlıdır** — yalnızca ofis ağına dönüşü
açar; sunucunun başka hiçbir ağa çıkamaması istenen bir güvenlik tercihiyse doğru seçimdir (LPM
gereği bu spesifik satır zaten default'tan önce seçilirdi, 4.3.1). (c) **Router'a bir şey eklemek işe
yaramaz** — router'ın her iki ağa da rotası zaten var; eksik olan bilgi **sunucunun** tablosundadır.
Doğru katmana müdahale etmek, teşhisin yarısıdır. · *Faz 4.1 × Faz 4.2/4.3*

**16.** `REACHABLE` = ARP kaydı doğrulanmış ve taze, iletişim var (3.1.2). `STALE` = kayıt var ama bir
süredir doğrulanmadı; kullanılır ve gerekirse yeniden doğrulanır — **arıza değildir.** `FAILED` = ARP
isteği yapıldı, **cevap gelmedi**. Bu, o IP'nin bulunduğu segmentte yanıt veren bir makine olmadığını
gösterir (3.1.2). Makinenin ayakta olmadığını **kanıtlamaz**: makine kapalı olabilir, ARP'a cevap
vermiyor olabilir, farklı bir VLAN'da olabilir (3.4), veya adres hiç kullanılmıyor olabilir. · *Faz
3.1 × Faz 3.4*

**17.** (a) `10.8.0.0/24` satırıyla eşleşir, **`tun0`** arayüzünden çıkar (bir VPN tüneli). (b)
`8.8.8.8` yalnızca `default` ile eşleşir → `192.168.1.1` üzerinden `enp3s0`'dan çıkar (4.3.1). (c)
`scope link`, o ağın **doğrudan bağlı** olduğunu söyler — hedefe ulaşmak için ara bir router gerekmez,
bu yüzden `via` alanı yoktur. `via` yalnızca paketin bir sonraki hop'a teslim edileceği durumlarda
bulunur (4.2.1). · *Faz 4.2 × Faz 4.3*

**18.** Çünkü `ip route` tabloyu **listeler**, `ip route get` ise çekirdeğin o hedef için **gerçekte
vereceği kararı** gösterir — LPM'i senin yerine uygular (4.3.1). Çok satırlı, üst üste binen
tablolarda hangi satırın kazandığını gözle bulmak hataya açıktır; bu komut tartışmayı bitirir. `via`
farkı: ilk çıktıda `via` **yok** → hedef doğrudan bağlı bir ağda (tünel arayüzünde). İkincisinde
`via 192.168.1.1` **var** → hedefe bir **sonraki hop** üzerinden gidiliyor (4.2.1). · *Faz 4.2 × Faz
4.3*

**19.** 8. sorudaki durumda ortada boşluk vardı ama **sonraki hop'lar cevap veriyordu** — orada iletim
sürüyordu. Burada 3. hop'tan sonra **hiç** cevap yok, yani yol gerçekten orada kesiliyor olabilir. Ama
bu **tek başına kesin değildir**: yol boyunca ICMP'nin engellendiği bir bölge de aynı görüntüyü verir
(4.5.2, Faz 9.3.1). Doğrulama yöntemi: hedefe gerçek protokol ve portla erişmeyi dene (`nc -zv host
port`). Erişim varsa `* * *`'lar yalnızca ICMP politikasıdır; yoksa yol gerçekten kopuktur. · *Faz
4.4 × Faz 4.5*

**20.** (a) **Evet** — A'nın maskesi `/24`, ağı `10.10.1.0–255`; B (`.51`) bu aralıktadır, A doğrudan
ARP yapar (2.1.2, 3.1). (b) **Evet** — B'nin maskesi `/25`, ağı `10.10.1.0–127`; A (`.50`) bu aralıkta
olduğu için B de doğrudan ulaşır. (c) Bu özel örnekte **asimetri çıkmaz**, çünkü her iki adres de
`/25`'in ilk bloğundadır. Asimetri, adreslerden biri `.128–.255` aralığında olsaydı doğardı: A onu
"komşum" sayıp ARP yapar, B ise "başka ağ" deyip gateway'e gönderirdi — belirti "bir yönde çalışıyor,
diğerinde çalışmıyor" veya router'ın gereksiz devreye girmesi olurdu. Ders: **maske uyuşmazlığı
sinsi arızalar üretir** ve tüm makinelerde aynı olmalıdır. · *Faz 2.1 × Faz 3.1 × Faz 4.1*

**21.** Satır 1: `10.10.1.50` **ARP yayını** yapıyor — "`10.10.1.1`'in MAC'i kimde?" (3.1.2). Satır 2:
gateway cevap veriyor — "benim, `00:1a:2b:3c:4d:5e`" (3.1.2). Satır 3: artık MAC bilindiği için ICMP
paketi gönderiliyor. Frame'in **hedef MAC'i `00:1a:2b:3c:4d:5e`**, yani **gateway'in MAC'i** — hedef
IP `8.8.8.8` olmasına rağmen. Bu üç satır, bu sınavın ana fikrinin kanıtıdır: **L3 hedefi uzaktır, L2
hedefi daima komşudur** (Soru 1). · *Faz 3.1 × Faz 4.1 × Faz 0.2*

---

## Puanlama

| Doğru sayısı | Değerlendirme |
|---|---|
| 19–21 | L2 ile L3'ü tek bir akışta birleştirmişsin. Faz 5'e güvenle geç. |
| 15–18 | Sağlam. Kaçırdığın soruların işaret ettiği **köprüye** bir tur dön. |
| 10–14 | İki fazı ayrı ayrı biliyorsun ama ekleme yeri zayıf. Aşağıdaki tabloyu kullan. |
| 0–9 | Faz 3.1 (ARP) ve Faz 4.1–4.3'ü (gateway, route, LPM) tekrar et; bu köprü oturmadan Faz 5 havada kalır. |

**Hangi soruyu kaçırırsan nereye dön:**

| Kaçırdığın soru | Dön — bu köprü zayıf |
|---|---|
| 1, 3, 9, 21 | Faz 3.1 × Faz 4.1 — ARP ile gateway'in birleştiği nokta |
| 2, 7 | Faz 3.3/3.4 × Faz 4.1 — broadcast domain, VLAN, inter-VLAN routing |
| 4, 17, 18 | Faz 4.2/4.3 — route tablosu ve LPM |
| 5, 16 | Faz 3.1/3.2 × Faz 4.2 — öğrenen switch vs yapılandırılan router |
| 6 | Faz 3.3 × Faz 4.4 — TTL, döngü, broadcast storm |
| 8, 19 | Faz 4.4/4.5 — ICMP'nin doğası ve traceroute okuma |
| 10, 12, 13, 14, 15 | Faz 4.1/4.2 — dönüş rotası ve asimetrik arızalar |
| 20 | Faz 2.1 × Faz 3.1 — maske uyuşmazlığının L2 sonucu |

---

## Kapanış — buradan Faz 5'e

Faz 3 ve 4 birlikte sana bir paketin **yolculuğunu** öğretti: komşuya nasıl teslim edildiğini, uzağa
nasıl yönlendirildiğini, ve her yönlendirme kararının ucunda neden yine bir ARP olduğunu. Artık bir
`ip route` çıktısına bakıp paketin nereden çıkacağını, bir `ip neigh` çıktısına bakıp L2 tarafının
sağlam olup olmadığını söyleyebiliyorsun.

Ama bir şeyi hiç sormadın: **paket yolda kaybolursa ne olur?**

Şimdiye kadar anlattığımız her mekanizma "elimden geleni yaparım" prensibiyle çalışıyor. IP, paketin
ulaştığını garanti etmez. Router doluysa düşürür, kablo gürültülüyse bozulur, TTL biterse yok edilir
— ve **kimse göndericiye haber vermek zorunda değildir.**

Faz 5 tam buraya bağlanır: bu güvenilmez zeminin üstüne **güvenilir** bir iletim nasıl inşa edilir?
Sıra numaraları, onaylar, yeniden gönderim, akış ve tıkanıklık kontrolü — TCP'nin tamamı, Faz 4'ün
bıraktığı bu boşluğu doldurmak için vardır. Ve orada karşına çıkacak bir şey daha var:
MTU (5.7.1), en sinsi arıza sınıflarından birinin (5.7.3) kaynağı olacak.

> **Devam etmeden önce:** Yukarıdaki 1. ve 21. soruların cevabını tereddütsüz verebiliyorsan — uzak bir
> hedefe giden paketin frame'inde neden gateway'in MAC'i olduğunu — Faz 5'e hazırsın. Veremiyorsan
> Faz 3.1 ve Faz 4.1'e bir tur dön; Faz 5, paketin hedefe **ulaştığını** varsayarak devam edecek.

---

> **Navigasyon:** [◀ Faz 4 — Yönlendirme (L3)](Faz_4_Yonlendirme_L3.md) · **Ara Sınav 2** · [Faz 5 — Transport Katmanı ▶](Faz_5_Transport_Katmani.md)
