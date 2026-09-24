# Ara Sınav 3 — Faz 5–9: "Packet Neden Düştü?"

> **Navigasyon:** [◀ Faz 9 — Firewall, Filtreleme ve Güvenlik](Faz_9_Firewall.md) · **Ara Sınav 3** · [Faz 10 — Troubleshooting ▶](Faz_10_Troubleshooting.md)

---

## Bu sınav neyi ölçüyor?

Bu, kitabın en geniş ara sınavı — beş fazı birden kapsıyor. Ve bunun bir sebebi var: **Faz 5'ten 9'a
kadar olan her şey, tek bir sorunun parçalarıdır.**

O soru şu: *"Packet neden düştü?"*

Faz 5 sana güvenilir iletimin nasıl kurulduğunu verdi — ve nerede kırıldığını (handshake, yeniden
gönderim, MTU). Faz 6 ismin adrese çevrilmesini; bu çeviri yanlışsa paket **doğru yere hiç gitmez.**
Faz 7 tek adresin paylaşılmasını; çeviri tablosu dolarsa veya yön yanlışsa paket **geri dönemez.**
Faz 8 web'in bu katmanların üstüne nasıl kurulduğunu; 502 ile 504'ün farkı aslında bir **transport**
farkıdır. Faz 9 ise paketin **bilinçli olarak** düşürülmesini.

Gerçek bir olayda bunlar hiç ayrı durmaz. "Site açılmıyor" şikâyetinin arkasında DNS'in eski kaydı da
olabilir, NAT tablosunun dolması da, MTU black hole da, unutulmuş bir NACL kuralı da. Bu sınav, o
ayrımı yapabiliyor musun diye bakar.

**Nasıl çalışılır:**

- Kâğıt-kalemle çöz; cevabı yazmadan bir sonrakine geçme.
- Cevap anahtarı her sorunun **hangi fazların kesişiminde** durduğunu söyler — kaçırdığın soru bir
  fazı değil, fazlar arasındaki **köprüyü** işaret eder.
- 21 soru var: Bölüm A (bağlantı muhakemesi, 1–9), Bölüm B (senaryo: "ödeme servisi çöktü" olayı,
  10–15), Bölüm C (komut ve çıktı okuma, 16–21).
- Hedef süre: ~1.5 saat. Bu sınav diğerlerinden uzun; bölerek çözmek tamamen meşrudur.

> **🤔 Başlamadan önce:** Şu iki belirtiyi yan yana koy ve farkı bir cümleyle yaz: *"Bağlantı 30 saniye
> bekleyip timeout verdi"* ve *"Bağlantı anında 'connection refused' aldı."* Bu tek ayrım, aşağıdaki
> soruların en az altısının anahtarıdır. Cevabını bir kenara yaz; Soru 2'de göreceğiz.

---

# Bölüm A — Bağlantı muhakemesi (1–9)

**1.** Faz 5'te TCP'nin üç aşamalı el sıkışmasını, Faz 9'da stateful firewall'u gördük. Bir security
group'un dönen trafiğe kural gerektirmemesi **doğrudan hangi TCP özelliğine** dayanır? UDP'de aynı
mekanizma neden yapay bir zaman aşımıyla taklit edilmek zorundadır?

**2.** `curl https://api.example.com` iki farklı makinede iki farklı hata veriyor: birinde 30 saniye
sonra timeout, diğerinde anında "connection refused". Her birinin **hangi katmana** işaret ettiğini ve
hangi ekibi aramanı gerektirdiğini yaz.

**3.** Faz 6'da DNS TTL'i, Faz 8'de HTTP'yi gördük. Bir servisi yeni bir sunucuya taşıdın ve DNS
kaydını güncelledin, ama kullanıcıların bir kısmı hâlâ eski sunucuya gidiyor. Sebep ne, ve bu sorunu
**önceden** önlemenin doğru sırası neydi?

**4.** Faz 7'de NAT'ı, Faz 5'te TCP'yi gördük. NAT cihazındaki çeviri tablosunun bir satırı, TCP
bağlantısı hâlâ açıkken zaman aşımına uğrarsa ne olur — istemci bunu nasıl yaşar, ve çözüm hangi
mekanizmadır?

**5.** Faz 8'de 502 ile 504 farkını gördük. Bu iki kodu **Faz 5'in** kavramlarıyla açıkla: hangisi
bağlantı kurulamamasına, hangisi bağlantı kurulup cevabın zamanında gelmemesine karşılık gelir?

**6.** Faz 5.7'de MTU'yu, Faz 9.3'te ICMP politikasını gördük. Bir güvenlik ekibi "tüm ICMP'yi
kapattık" diyor. Hangi belirti ortaya çıkar, neden bu belirti hiç firewall gibi görünmez, ve zinciri
adım adım yaz.

**7.** Faz 6'da CNAME'i, Faz 8'de TLS'i gördük. Bir alan adı bir CDN'e CNAME ile bağlı ve tarayıcı
"sertifika hatası" veriyor. TLS'in hangi mekanizması (Faz 8.4) devrede, ve hata DNS tarafında mı TLS
tarafında mı aranmalı?

**8.** Faz 7'de private subnet'i, Faz 9'da güvenlik katmanlarını gördük. "Makinelerimiz private
subnet'te, o yüzden güvendeler" cümlesindeki iki ayrı hatayı yaz — biri Faz 7'ye, biri Faz 9'a dayansın.

**9.** Faz 5'te ephemeral portu, Faz 9'da NACL'i gördük. NACL'in giden kuralında neden servisin portu
(`443`) değil, geniş bir aralık (`1024-65535`) yazılır? Security group'ta bu neden gerekmez?

---

# Bölüm B — Senaryo: "ödeme servisi çöktü" olayı (10–15)

> Bir e-ticaret sisteminde ödeme servisi, bir dış sağlayıcının API'sine (`api.payments.example`) HTTPS
> ile bağlanıyor. Saat 14:20'de alarmlar ötüyor: ödemelerin %40'ı başarısız. Servis private subnet'te,
> internete NAT Gateway üzerinden çıkıyor, önünde bir ALB var. Aşağıdaki altı soru sırayla teşhisin.

**10.** İlk log satırı: `dial tcp: i/o timeout` — yani bağlantı **kurulamıyor.** Bu tek satır hangi
ihtimalleri eler, hangilerini bırakır? Handshake'in hangi adımında takıldığını nereden bilirsin?

**11.** Servis makinesinde `dig api.payments.example` çalıştırdın, düzgün bir IP döndü. Bu gözlem
hangi fazı eler ve **hangi ihtimali hâlâ elemez**? (İpucu: doğru cevap dönmesi, doğru cevabın
**kullanıldığı** anlamına gelmez.)

**12.** `tcpdump -ni any 'tcp port 443 and host <api-ip>'` çalıştırdın ve yalnızca `[S]` satırları
görüyorsun, `[S.]` hiç yok. Teşhisin ne, ve bu gözlem Faz 9'un hangi ayrımını **kanıtlıyor**?

**13.** Hataların **%40'ı** başarısız, %60'ı başarılı. Bu oran neyi ele veriyor — tamamen kapalı bir
kural mı, yoksa kapasite/kaynak sınırına dayanan bir şey mi? Faz 7'nin hangi mekanizması bu tabloya
uyar?

**14.** VPC Flow Logs'ta NAT Gateway arayüzünde giden bağlantılar için **ACCEPT** satırları var ama
bazı akışlarda dönüş satırı **hiç yok**. Bu ne anlama gelir, ve NACL ile SG'den hangisi bu tabloya
daha uygun bir şüphelidir?

**15.** Kök neden: NAT Gateway'in aynı hedef IP:port çiftine kurabileceği eşzamanlı bağlantı sayısı
dolmuş (port tükenmesi). Üç çözüm değerlendir: (a) bağlantı havuzu kullanıp bağlantıları yeniden
kullanmak, (b) ikinci bir NAT Gateway eklemek, (c) TCP keepalive süresini uzatmak. Her birini Faz 5
ve Faz 7 mantığıyla gerekçelendir — hangisi kök nedene dokunur, hangisi durumu kötüleştirir?

---

# Bölüm C — Komut ve çıktı okuma (16–21)

**16.** `dig api.example.com` çıktısı (kısaltılmış):

```
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 14203
;; ANSWER SECTION:
api.example.com.      45   IN  CNAME  d3kx9.cloudfront.net.
d3kx9.cloudfront.net. 60   IN  A      13.224.10.7
;; SERVER: 127.0.0.53#53(127.0.0.53)
```

(a) `45` ve `60` sayıları ne? (b) `SERVER:` satırı neden önemli? (c) Bu zincirde bir **dangling CNAME**
riski olsaydı belirtisi ne olurdu?

**17.** `ss -tan` çıktısı:

```
State        Recv-Q  Send-Q  Local Address:Port    Peer Address:Port
ESTAB        0       0       10.0.1.50:51234       93.184.216.34:443
SYN-SENT     0       1       10.0.1.50:51290       203.0.113.9:443
TIME-WAIT    0       0       10.0.1.50:51201       93.184.216.34:443
CLOSE-WAIT   0       0       10.0.1.50:51188       93.184.216.34:443
```

Dört durumu da yorumla. Hangisi bir **arıza** işaretidir, hangisi normaldir? `SYN-SENT`'te takılı
kalmak neyi gösterir (Faz 5 ve Faz 9 birlikte)?

**18.** `curl -v https://example.com` çıktısı:

```
* Connected to example.com (93.184.216.34) port 443
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* SSL certificate verify ok.
> GET / HTTP/1.1
> Host: example.com
< HTTP/1.1 504 Gateway Timeout
```

Bu çıktı hangi katmanlara kadar **başarılı**? 504 kodu hangi tarafa işaret ediyor, ve sorunun
istemcide olmadığını hangi satırlar kanıtlıyor?

**19.** İki ping testi:

```
$ ping -c2 -M do -s 1400 10.20.0.9
2 packets transmitted, 2 received, 0% packet loss

$ ping -c2 -M do -s 1472 10.20.0.9
ping: local error: message too long, mtu=1436
```

Bu iki çıktı birlikte ne söylüyor? Efektif MTU kaç, neden 1500 değil, ve bu hangi arızaya yol açar?

**20.** VPC Flow Logs'tan iki satır:

```
2 111... eni-0a1 10.0.1.50 10.0.2.20 51234 5432 6 8 1024 ... ACCEPT OK
2 111... eni-0b2 10.0.2.20 10.0.1.50 5432 51234 6 5  640 ... REJECT OK
```

İki satırı birlikte okuyup teşhis koy. Hangi yön reddedilmiş, hangi kural eksik, ve istemci tarafında
belirti ne olur?

**21.** `iptables -L -n -v` çıktısı (kısaltılmış):

```
Chain INPUT (policy DROP 1204 packets, 96K bytes)
 pkts bytes target  prot opt source        destination
 8412  982K ACCEPT  all  --  0.0.0.0/0     0.0.0.0/0    ctstate RELATED,ESTABLISHED
   14   840 ACCEPT  tcp  --  0.0.0.0/0     0.0.0.0/0    tcp dpt:22
    0     0 ACCEPT  tcp  --  0.0.0.0/0     0.0.0.0/0    tcp dpt:8080
```

(a) `policy DROP` ne anlatıyor? (b) İlk kural neden bu kadar çok paket saymış? (c) Üçüncü kuralın
`pkts` değeri 0 — bu ne demek ve hangi iki farklı sebebi olabilir?

---

## Cevap Anahtarı

**1.** Doğrudan **TCP'nin durum bilgisine** dayanır (5.2.1): SYN bir bağlantının başlangıcı, SYN-ACK
cevabı, ACK ise kuruluşudur. Stateful firewall bu bayrakları okuyup bağlantıyı bir tabloya yazar ve
dönen paketin "zaten izin verdiğim bağlantının parçası" olduğunu **bilir** (9.1.3). UDP'de böyle
bayraklar yoktur (5.1.1) — bağlantı kavramı yoktur — bu yüzden firewall yapay bir zaman aşımı tutar:
bir süre trafik görmezse satırı siler. · *Faz 5.1/5.2 × Faz 9.1*

**2.** **Timeout** → paket **sessizce düşürülmüş** (DROP, 9.2.3): firewall (SG/NACL/OS), eksik route
(4.2) veya erişilemeyen host. Ağ/güvenlik tarafına bakarsın. **Connection refused** → bir **RST**
gelmiş (5.2.2), yani paket hedefe **ulaşmış** ve makine cevap vermiş; ağ katmanı kanıtlanmıştır, sorun
servistedir (dinlemiyor veya yanlış adrese bağlanmış, 1.4.2). Uygulama ekibine gidersin. Tek kelimelik
bu fark, doğru ekibi ilk denemede bulmanı sağlar. · *Faz 5.2 × Faz 9.2*

**3.** Sebep **DNS cache'i ve TTL'dir** (6.4.1): istemciler ve ara çözümleyiciler eski cevabı TTL
süresince saklar; sen kaydı değiştirsen bile onlar eskisini kullanmaya devam eder. Doğru sıra (6.4.3):
**önce TTL'i düşür** (örneğin 3600 → 60), **eski TTL süresi kadar bekle** (tüm cache'ler yenilensin),
**sonra kaydı değiştir**, geçiş tamamlanınca TTL'i tekrar yükselt. TTL'i değişiklikle **aynı anda**
düşürmek işe yaramaz — çünkü dışarıdaki cache'ler hâlâ eski, uzun TTL'i taşır. · *Faz 6.4 × Faz 8.1*

**4.** NAT tablosundaki satır silinince, o bağlantıya ait dönen paketler **eşleşecek bir kayıt
bulamaz** ve düşürülür (7.1.2). İstemci bunu şöyle yaşar: bağlantı sanki donmuş gibidir — veri gider
ama cevap gelmez; sonunda TCP zaman aşımıyla ya da RST ile kopar. Klasik belirti: **uzun süre boşta
kalan SSH oturumunun ölmesi** (5.6.1). Çözüm **TCP keepalive**'dir: düzenli aralıklarla küçük paketler
gönderilerek NAT/firewall tablosundaki satır taze tutulur. · *Faz 5.6 × Faz 7.1*

**5.** **502 Bad Gateway** = proxy/yük dengeleyici, arkadaki sunucuya **bağlanamadı** veya geçersiz
cevap aldı — Faz 5 diliyle: **el sıkışma kurulamadı** ya da bağlantı erken koptu (RST/FIN, 5.2.2,
5.6.1). **504 Gateway Timeout** = bağlantı **kuruldu** ama cevap zamanında gelmedi — yani handshake
başarılı, veri akışı yavaş veya sunucu yanıt üretemiyor (8.2.2). Tek cümle: *502 bağlanamadım, 504
bekledim.* · *Faz 5.2/5.6 × Faz 8.2*

**6.** Belirti: **küçük paketler geçiyor, büyük paketler geçmiyor** — MTU black hole (5.7.3). Zincir:
yoldaki bir cihazın MTU'su daha küçüktür → büyük paketi iletemez → normalde ICMP **Fragmentation
Needed (tip 3 kod 4)** ile göndericiye haber verir (4.4.2) → ama ICMP engellendiği için bu mesaj
ulaşmaz → gönderici aynı boyutta yeniden dener → paketler **sessizce** kaybolur. Firewall gibi
görünmemesinin sebebi belirtinin uygulama diliyle konuşmasıdır: *"SSH bağlanıyor ama donuyor"*,
*"sorgular takılıyor"*, *"site açılıyor ama resimler gelmiyor"* (9.3.1). · *Faz 5.7 × Faz 9.3*

**7.** Devrede olan mekanizma **sertifika doğrulamasıdır** (8.4.2): tarayıcı, sunucunun sunduğu
sertifikanın **istenen alan adını** kapsayıp kapsamadığına bakar. CNAME zinciri DNS'te sorunsuz çözülse
bile, CDN senin alan adın için bir sertifika sunmuyorsa doğrulama başarısız olur. Yani hata **TLS
tarafındadır** — DNS görevini yapmıştır. Genellikle sebep, CDN'de alan adı için sertifika
yapılandırılmamış olması veya **SNI** ile yanlış sertifikanın sunulmasıdır (8.4.2). · *Faz 6.3 × Faz
8.4*

**8.** Hata 1 (**Faz 7**): private subnet yalnızca **internetten gelen** bağlantıyı engeller (7.3.2);
makineler NAT üzerinden **dışarı çıkabilir**, yani ele geçirilmiş bir makine dışarıyla haberleşebilir
ve NAT bunu engellemez (7.1.2). Hata 2 (**Faz 9**): private subnet **VPC içinden** gelen bağlantıyı
engellemez — route table'daki `local` satırı gereği aynı VPC'deki her instance erişebilir; asıl koruma
SG/NACL'dir ve tek katmana güvenmek katmanlı savunma ilkesini ihlal eder (9.5.2). · *Faz 7.3 × Faz 9.5*

**9.** Çünkü **NACL stateless'tır** (9.1.2): dönen paketi bağımsız değerlendirir ve o paketin **hedef
portu** sunucunun portu değil, istemcinin **ephemeral portudur** (1.4.1, 5.1). Hangi portun
seçileceğini önceden bilemeyeceğin için aralığın tamamını açmak zorundasın. SG'de gerekmez çünkü SG
**stateful'dur**: bağlantıyı tablosunda tutar ve dönen paketi kural kontrolüne hiç sokmadan geçirir
(9.1.3, 9.4.1). · *Faz 5.1 × Faz 9.1/9.4*

**10.** `i/o timeout` **hiçbir cevap gelmediğini** söyler: ne SYN-ACK, ne RST, ne ICMP. Bu, RST
döndüren senaryoları (servis dinlemiyor, 5.2.2) ve "no route to host" gibi anında hata verenleri
**eler**. Geriye şunlar kalır: firewall DROP (SG/NACL/OS, 9.2.3), route eksikliği (gidiş **veya
dönüş**, 4.2), NAT tarafında bir sorun (7.1.2), veya hedefin ayakta olmaması. Handshake'in hangi
adımında takıldığını **`tcpdump` ile** görürsün: yalnızca `[S]` varsa ilk adımda takılmıştır (5.2.1).
· *Faz 5.2 × Faz 9.2*

**11.** **Faz 6'yı (DNS çözümlemesini) eler** — isim doğru IP'ye çözülüyor. Ama şunu **elemez**:
uygulamanın gerçekten o IP'yi kullanıp kullanmadığı. Uygulama kendi içinde **eski bir DNS cache'i**
tutuyor olabilir (bazı çalışma zamanları çözülmüş IP'leri uzun süre saklar, 6.4.2), ya da bağlantı
havuzundaki mevcut bağlantılar eski bir IP'ye kurulmuş olabilir. `dig`'in sonucu **çözümleyicinin**
cevabıdır, **uygulamanın** kullandığı adres değil. Kesin doğrulama: `ss -tan` ile uygulamanın gerçekten
hangi peer IP'lere bağlandığına bak. · *Faz 6.2/6.4 × Faz 5.3*

**12.** Teşhis: **SYN gidiyor, SYN-ACK dönmüyor** — paket ya hedefe ulaşmıyor ya da cevap
düşürülüyor; her iki hâlde de araya giren bir şey **sessizce** düşürüyor (DROP). Kanıtladığı ayrım:
9.2.3'teki **DROP vs REJECT** ayrımıdır — REJECT olsaydı `[R]` (RST) görürdük ve belirti "connection
refused" olurdu (5.2.2). Sessizlik, bilinçli bir filtreleme politikasının imzasıdır. · *Faz 5.2 × Faz
9.2*

**13.** **Tamamen kapalı bir kural olamaz** — kural kapalı olsaydı oran %100 başarısızlık olurdu.
%40'lık kısmi başarısızlık, bir **kapasite sınırına** işaret eder: yoğunlukta dolan, boşaldıkça
çalışan bir kaynak. Faz 7'nin uyan mekanizması **NAT/PAT'in port sınırıdır** (7.2.1): 16 bitlik port
alanı nedeniyle aynı hedefe kurulabilecek eşzamanlı bağlantı sayısı sınırlıdır; tablo dolunca yeni
bağlantılar kurulamaz, mevcutlar çalışmaya devam eder. Aynı sınıf sorun stateful firewall'ın bağlantı
tablosunda da görülür (9.1.3). · *Faz 7.2 × Faz 9.1*

**14.** Giden **ACCEPT** var demek, paketin NAT Gateway'e kadar geldiği ve kabul edildiği demektir.
Dönüş satırının **hiç olmaması**, cevabın geri gelmediğini gösterir — yani sorun ya karşı tarafta ya da
dönüş yolunda (10.3.3, 10.4.3). Şüpheli olarak **SG değil, NACL daha uygundur**: SG stateful olduğu
için dönen trafiği zaten geçirirdi; stateless NACL'de ise giden ephemeral kuralı eksikse dönüş
reddedilir veya düşer (9.4.1). Not: dönüş satırının hiç olmaması REJECT'ten farklıdır — REJECT
görseydin kuralı doğrudan işaret ederdi; satırın hiç olmaması "cevap gelmedi" demektir. · *Faz 7.1 ×
Faz 9.4*

**15.** (a) **Bağlantı havuzu — kök nedene dokunan tek çözüm.** Her istek için yeni bağlantı açmak
yerine mevcutları yeniden kullanmak, eşzamanlı bağlantı sayısını doğrudan düşürür (5.2.1'deki
handshake maliyetinden de kurtarır; 8.3'teki HTTP keep-alive fikri). (b) **İkinci NAT Gateway** —
çalışır ama **palyatiftir**: sınırı ikiye katlar, kökü çözmez ve maliyeti artırır; trafik büyüyünce
aynı duvara çarparsın. (c) **Keepalive süresini uzatmak — durumu kötüleştirir:** bağlantılar tabloda
daha uzun kalır, port tükenmesi daha hızlı olur. (Keepalive, Soru 4'teki **erken silinme** sorununun
çözümüdür; burada sorun tam tersidir.) · *Faz 5.2/5.6 × Faz 7.2*

**16.** (a) **TTL değerleri** — cevabın kaç saniye cache'leneceği (6.4.1); burada düşük tutulmuş, yani
hızlı değişime hazır bir yapılandırma. (b) `SERVER:` satırı **hangi çözümleyiciye sorduğunu** söyler
(6.2.3); `127.0.0.53` yerel stub çözümleyicidir. Teşhiste kritiktir: `dig @1.1.1.1` çalışıp yerel
sorgu çalışmıyorsa sorun DNS'in kendisinde değil, **senin çözümleyicinde**dir. (c) **Dangling CNAME**
riski: CNAME hedefi (CDN dağıtımı) silinir ama kayıt kalırsa, isim çözülemez (NXDOMAIN) — ya da daha
kötüsü, o hedef adı başkası tarafından ele geçirilirse **subdomain takeover** olur (6.3.2). · *Faz 6.3
× Faz 6.4*

**17.** **ESTAB** = kurulmuş, sağlıklı bağlantı. **SYN-SENT** = SYN gönderildi, **SYN-ACK bekleniyor**
— burada takılı kalmak, cevabın hiç gelmediğini gösterir: firewall DROP veya erişilemeyen hedef
(5.2.2, 9.2.3). **Asıl arıza işareti budur.** **TIME-WAIT** = bağlantıyı kapatan taraf, gecikmiş
paketler için bekliyor — **normaldir** (5.6.1); sayısı çok yükselirse kısa ömürlü bağlantıların
fazlalığına işaret eder. **CLOSE-WAIT** = karşı taraf FIN gönderdi ama **yerel uygulama soketi
kapatmadı** — tek tük normal, ama biriktiğinde bir **uygulama hatasıdır** (soket sızıntısı, 5.6.1).
· *Faz 5.2/5.6 × Faz 9.2*

**18.** Başarılı olan katmanlar: **DNS** (isim çözüldü), **TCP** (`Connected ... port 443` — handshake
tamam, 5.2.1), **TLS** (`certificate verify ok` — el sıkışma ve doğrulama tamam, 8.4.2) ve **HTTP
isteği gönderildi.** 504, **sunucu tarafına** işaret eder: proxy/yük dengeleyici arkadaki servise
bağlanmış ama cevabı zamanında alamamıştır (8.2.2). İstemcide sorun olmadığını kanıtlayan satırlar
tam olarak bunlardır — ağ, TLS ve istek zinciri eksiksiz çalışmıştır; kalan tek şüpheli backend'dir.
· *Faz 8.2/8.4 × Faz 5.2*

**19.** `-M do` parçalanmayı yasaklar, `-s` veri boyutunu belirler (5.7.3). 1400 bayt geçiyor, 1472
geçmiyor ve çekirdek **`mtu=1436`** diyor. Yani efektif MTU **1436**'dır, 1500 değil — aradaki 64
bayt, bir **tünelin** (VPN/IPsec/VXLAN) eklediği başlık yüküdür (7.4.2). Yol açtığı arıza: **MTU black
hole** — ICMP Fragmentation Needed engellenirse büyük paketler sessizce kaybolur ve belirti
"bağlanıyor ama takılıyor" olur (5.7.3, 9.3.1). Çözüm genelde **MSS clamping**'dir (5.7.3). · *Faz 5.7
× Faz 7.4*

**20.** İlk satır: `10.0.1.50 → 10.0.2.20:5432` **ACCEPT** — istek veritabanına ulaşmış. İkinci satır:
`10.0.2.20:5432 → 10.0.1.50:51234` **REJECT** — **dönüş** reddedilmiş. Eksik kural: hedef subnet'in
**NACL giden (outbound) ephemeral** kuralı (`ALLOW TCP 1024-65535`), çünkü cevap istemcinin ephemeral
portuna gider (9.4.1). İstemci tarafında belirti: SYN gönderilir, SYN-ACK dönemez → **timeout**
(9.2.3) — yani "veritabanına bağlanamıyorum", oysa istek veritabanına ulaşmıştır. · *Faz 5.1 × Faz 9.4
× Faz 10.4*

**21.** (a) `policy DROP`, hiçbir kurala uymayan paketlerin **sessizce düşürüleceğini** söyler —
doğru varsayılan olan **default deny** (9.2.2); yanındaki sayaç (1204 paket) bunun aktif olarak
çalıştığını gösterir. (b) İlk kural `ctstate RELATED,ESTABLISHED` ile **stateful takibi** uygular
(9.1.3): makinenin kendi başlattığı tüm bağlantıların dönen trafiği bu kuraldan geçer — normal
trafiğin ezici çoğunluğu budur. (c) `pkts = 0`, o kuralın **hiç eşleşmediğini** söyler. İki sebebi
olabilir: (i) 8080'e **hiç trafik gelmemiştir** (servis kullanılmıyor veya istemci hiç denemiyor),
(ii) trafik geliyor ama **daha üstteki bir kural** onu önce yakalıyordur (sıra sorunu, 9.2.2). Ayrım
için `tcpdump` ile 8080'e paket gelip gelmediğine bakarsın (10.2.2). · *Faz 9.1/9.2 × Faz 10.2*

---

## Puanlama

| Doğru sayısı | Değerlendirme |
|---|---|
| 19–21 | "Packet neden düştü" sorusunun tüm parçaları sende. Faz 10'a güvenle geç. |
| 15–18 | Sağlam. Kaçırdığın soruların işaret ettiği **köprüye** bir tur dön. |
| 10–14 | Fazları ayrı ayrı biliyorsun ama birleşim noktaları zayıf. Aşağıdaki tabloyu kullan. |
| 0–9 | Faz 5.2 (handshake), 9.1 (stateful) ve 9.2.3 (DROP/REJECT) üçlüsünü tekrar et; bu üçü olmadan Faz 10'un metodolojisi işe yaramaz. |

**Hangi soruyu kaçırırsan nereye dön:**

| Kaçırdığın soru | Dön — bu köprü zayıf |
|---|---|
| 1, 9, 14, 20 | Faz 5.1/5.2 × Faz 9.1/9.4 — stateful/stateless ve ephemeral port |
| 2, 10, 12, 17 | Faz 5.2 × Faz 9.2 — timeout vs refused, handshake imzaları |
| 3, 11, 16 | Faz 6.2/6.3/6.4 — çözümleme, CNAME, TTL ve cache |
| 4, 13, 15 | Faz 5.6 × Faz 7.1/7.2 — NAT tablosu, port tükenmesi, keepalive |
| 5, 18 | Faz 5.2/5.6 × Faz 8.2 — 502 vs 504'ün transport karşılığı |
| 6, 19 | Faz 5.7 × Faz 7.4 × Faz 9.3 — MTU, tünel, ICMP politikası |
| 7 | Faz 6.3 × Faz 8.4 — CNAME zinciri ve sertifika doğrulama |
| 8, 21 | Faz 7.3 × Faz 9.5 — katmanlı savunma ve default deny |

---

## Kapanış — buradan Faz 10'a

Faz 5'ten 9'a kadar olan beş faz, sana bir paketin başına gelebilecek **her şeyi** öğretti: yolda
kaybolmasını ve yeniden gönderilmesini (Faz 5), yanlış adrese gitmesini (Faz 6), geri dönememesini
(Faz 7), yanlış cevap almasını (Faz 8) ve bilinçli olarak düşürülmesini (Faz 9).

Artık her birinin **imzasını** tanıyorsun: timeout mu refused mu, hangi TCP durumunda takıldı, TTL
dolu muydu, cevap eski cache'ten mi geldi, dönüş kuralı var mıydı.

Ama bir şey eksik. Bu imzaların hepsi ayrı ayrı zihninde duruyor — ve gerçek bir arızada elinde otuz
ihtimal, bir kullanıcı ve az vakit olacak.

Faz 10 tam buraya bağlanır: bu beş fazın bilgisini **bir sıraya** koyar. Aşağıdan yukarı disiplinli
bir metodoloji, her aracın hangi katmana baktığı, ve bulutta paketin neden düştüğünü tahmin etmek
yerine **Flow Logs'tan okumak.** Bu sınavda tek tek cevapladığın soruların hepsi, orada tek bir karar
ağacında toplanacak.

> **Devam etmeden önce:** Yukarıdaki 2. ve 12. soruların cevabını tereddütsüz verebiliyorsan —
> timeout ile refused arasındaki farkın neden iki farklı ekibe işaret ettiğini — Faz 10'a hazırsın.
> Veremiyorsan Faz 5.2 ve Faz 9.2.3'e bir tur dön; Faz 10'un tüm metodolojisi bu ayrımın üstünde duruyor.

---

> **Navigasyon:** [◀ Faz 9 — Firewall, Filtreleme ve Güvenlik](Faz_9_Firewall.md) · **Ara Sınav 3** · [Faz 10 — Troubleshooting ▶](Faz_10_Troubleshooting.md)
