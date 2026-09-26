# Faz 8 — Uygulama Katmanı: HTTP + TLS

> **Navigasyon:** [◀ Faz 7 — NAT ve Gerçek Dünya](Faz_7_NAT.md) · **Faz 8** · [Faz 9 — Firewall ve Filtreleme ▶](Faz_9_Firewall.md)

---

## Nereden geliyoruz

Faz 7'nin sonunda şunu sorduk: TLS trafiği şifreliyor, ama şifreleme için iki tarafın **ortak bir
anahtarı** olması gerekir. Bu anahtarı, yolda herkesin dinleyebildiği şifresiz bir kanaldan nasıl
paylaşıyorlar?

İpucunu da vermiştik: **paylaşmıyorlar.** Her iki taraf da anahtarı **ayrı ayrı hesaplıyor** — ve yoldaki
dinleyici, gördüğü her şeye rağmen aynı sonuca ulaşamıyor. 8.4'te bunun nasıl mümkün olduğunu göreceksin.

Yanında getirdiklerin:

- **Port 80 ve 443** (1.4.1) — bu fazın iki ana kapısı.
- **TCP handshake** (5.2.1) — TLS onun **üstüne** kurulur; önce TCP, sonra TLS.
- **Head-of-line blocking** (5.1.2) — HTTP/3'ün neden UDP'ye geçtiğinin cevabı.
- **DNS ve TTL** (6.4) — CDN'in çalışma mantığı ve "bir bölgede eski içerik" sorunu oraya bağlanır.
- **Reverse proxy fikri** (7.2.3) — NAT'ın port forwarding'ine benzer ama çok daha akıllı.

## Bu fazın sorusu

> *"Paket hedefe ulaştı. Peki sunucu senin **ne istediğini** nereden biliyor — ve o cevap yolda kimse
> tarafından okunamadan nasıl dönüyor?"*

Bu faz, yığının en üstüdür. Buraya kadar öğrendiğin her şey **taşımayla** ilgiliydi; bu faz **içerikle**
ilgilidir. Ve ilginç biçimde, günlük işinde en çok karşılaşacağın hata mesajları buradan çıkar: 404, 502,
504, "certificate has expired", "SSL handshake failed".

Bu fazın pratik değeri şurada: **durum kodları, sorunun hangi katmanda olduğunu neredeyse tek başına
söyler.** 502 ile 504 arasındaki farkı bilen bir mühendis, doğru ekibi ilk denemede arar. Bunu öğreneceksin.

---

## Bu fazın sonunda

- Bir HTTP isteğinin ve cevabının yapısını (metod, path, header, body) satır satır okuyabileceksin
- GET/POST/PUT/DELETE/PATCH ayrımını ve idempotency fikrini anlatabileceksin
- Durum kodu gruplarını (2xx/3xx/4xx/5xx) ve her birinin **kime** işaret ettiğini bileceksin
- **502 ile 504 arasındaki farkı** — backend çöktü mü, yavaş mı — teşhiste kullanabileceksin
- HTTP/1.1, HTTP/2 ve HTTP/3 arasındaki farkı ve HTTP/3'ün neden UDP kullandığını açıklayabileceksin
- TLS el sıkışmasının adımlarını, sertifika doğrulamasının ne kanıtladığını ve anahtarın nasıl
  **paylaşılmadan** elde edildiğini anlatabileceksin
- Reverse proxy, CDN ve anycast'in ne yaptığını ve birbirlerinden farkını bileceksin
- **Cloud:** ALB'de TLS termination'ın nerede olduğunu, CloudFront'un ne yaptığını ve "bir bölgede eski
  içerik" sorununun sebebini anlatabileceksin

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden burada |
|---|---|---|---|
| 8.1 | HTTP request/response | `[mekanizma]` | İsteğin anatomisi |
| 8.2 | Durum kodları | `[kavram]` | **Fazın kalbi** — teşhisin en hızlı aracı |
| 8.3 | HTTP/1.1 vs 2 vs 3 | `[atla]` | Farkındalık düzeyi |
| 8.4 | TLS el sıkışması | `[kavram]` | `https`'teki `s` |
| 8.5 | Reverse proxy, CDN, anycast | `[kavram]` | Modern web mimarisi |
| 8.6 | Bu faz bozulunca | — | HTTP/TLS arıza imzaları |

> **Bu fazda nasıl çalışmalı:** Bu fazın tüm komutları 🟢'dir ve tek bir araç etrafında döner:
> **`curl -v`**. O `-v` (verbose) bayrağı, DNS çözümlemesinden TCP bağlantısına, TLS el sıkışmasından
> HTTP başlıklarına kadar **tüm katmanları** tek çıktıda gösterir — yani Faz 0–8'in tamamını bir ekranda
> görürsün. Bu fazı okurken `curl -v https://example.com` çıktısını açık tut ve her bölümde ilgili
> satırları işaretle. 8.2'yi (durum kodları) ezberlemeye çalışma; **grup mantığını** anla, gerisi kendi
> kendine gelir. 8.3'ü `[atla]` düzeyinde geç — adları ve tek cümlelik farkları yeter.

---
---

# 8.1 HTTP Request/Response

## 8.1.1 İsteğin anatomisi `[mekanizma]`

HTTP şaşırtıcı derecede basittir: **düz metin** bir sözleşmedir. Tarayıcın sunucuya şöyle bir şey yollar:

```
GET /index.html HTTP/1.1          ← istek satırı: metod, path, sürüm
Host: example.com                 ← hangi site (tek IP'de çok site olabilir)
User-Agent: curl/8.5.0            ← istemci kim
Accept: text/html                 ← ne tür cevap kabul ediyorum
Connection: keep-alive            ← bağlantıyı açık tut
                                  ← boş satır: başlıklar bitti
(body — GET'te genelde yok)
```

Üç parça vardır: **istek satırı**, **başlıklar** (header), ve isteğe bağlı **gövde** (body). Başlıkları
gövdeden ayıran şey **boş bir satırdır**.

`Host` başlığı özellikle önemlidir: bir sunucu tek IP üzerinde yüzlerce siteyi barındırabilir; hangisini
istediğini bu başlık söyler (buna virtual hosting denir). TLS'te bunun karşılığı **SNI**'dır (8.4.2).

Cevap da aynı yapıdadır:

```
HTTP/1.1 200 OK                   ← durum satırı: sürüm, kod, açıklama
Content-Type: text/html           ← gövdede ne var
Content-Length: 1256              ← kaç bayt
Cache-Control: max-age=3600       ← ne kadar cache'lenebilir
                                  ← boş satır
<!DOCTYPE html>...                ← gövde
```

## 8.1.2 Metodlar ve niyet `[kavram]`

Metod, **niyetini** bildirir:

| Metod | Niyet | Idempotent mi |
|---|---|---|
| **GET** | Getir — hiçbir şeyi değiştirme | ✅ |
| **POST** | Yeni bir şey oluştur / işlem çalıştır | ❌ |
| **PUT** | Tamamını değiştir (yoksa oluştur) | ✅ |
| **PATCH** | Kısmen değiştir | ❌ (genelde) |
| **DELETE** | Sil | ✅ |
| **HEAD** | GET gibi ama sadece başlıkları döndür | ✅ |

**Idempotent** (*aynı-etkili*), "aynı isteği iki kez göndermek, bir kez göndermekle aynı sonucu verir"
demektir. Bu, ağ dünyasında çok pratik bir özelliktir: cevap gelmediğinde isteği **güvenle yeniden
gönderebilirsin**. Faz 5'te TCP'nin kayıp paketi yeniden gönderdiğini öğrenmiştin (5.3.2); HTTP
seviyesinde aynı mantık, idempotent metodlar sayesinde uygulama katmanında da işler.

POST idempotent **değildir** — bu yüzden bir ödeme isteği timeout verdiğinde körlemesine yeniden
gönderemezsin (iki kez tahsilat riski). Bu sorunun çözümü genelde bir **idempotency key** başlığıdır.

> **🔧 Makinende gör** 🟢 — tüm katmanları tek çıktıda oku
>
> ```
> $ curl -v https://example.com 2>&1 | head -25
> * Host example.com:443 was resolved.
> * IPv4: 93.184.216.34                          ← Faz 6: DNS çözümlendi
> *   Trying 93.184.216.34:443...
> * Connected to example.com (93.184.216.34) port 443   ← Faz 5: TCP handshake bitti
> * ALPN: curl offers h2,http/1.1                 ← 8.3: hangi HTTP sürümü
> * TLSv1.3 (OUT), TLS handshake, Client hello:   ← 8.4: TLS başlıyor
> * TLSv1.3 (IN), TLS handshake, Server hello:
> * SSL connection using TLSv1.3 / AEAD-AES256-GCM-SHA384
> * Server certificate:
> *  subject: CN=example.com                      ← sertifika kime ait
> *  expire date: Mar  1 23:59:59 2026 GMT        ← ne zaman doluyor
> *  issuer: C=US; O=DigiCert Inc; CN=...         ← kim imzaladı
> > GET / HTTP/2                                  ← 8.1: istek gidiyor
> > Host: example.com
> < HTTP/2 200                                    ← 8.2: cevap geldi
> < content-type: text/html
> ```
>
> Bu tek komut, bu kitabın tamamını bir ekrana sığdırır: **DNS** (Faz 6) → **TCP** (Faz 5) → **TLS**
> (8.4) → **HTTP** (8.1–8.2). Okurken sembollere dikkat et: `*` curl'ün kendi notu, `>` giden istek,
> `<` gelen cevap. Bir sorunu teşhis ederken bu çıktının **nerede durduğu**, sorunun hangi katmanda
> olduğunu doğrudan söyler — DNS satırında durursa Faz 6, "Trying..."de durursa Faz 4/5/9, TLS
> satırlarında durursa 8.4.

> **🤔 Düşün 8.1** — Bir ödeme API'sine POST isteği gönderdin ve 30 saniye sonra timeout aldın. Ödemenin
> geçip geçmediğini bilmiyorsun. (a) İsteği körlemesine yeniden göndermek neden riskli? (b) API
> tasarımcısı bu sorunu nasıl çözebilir? (c) Aynı sorun bir GET isteğinde olsaydı endişelenir miydin?
>
> *(Cevap: fazın sonunda)*

---
---

# 8.2 Durum Kodları

## 8.2.1 Grup mantığı `[kavram]`

Durum kodunu ezberleme; **ilk rakamı** oku. Her grup, sorunun **kimde** olduğunu söyler:

| Grup | Anlamı | Kim sorumlu |
|---|---|---|
| **1xx** | Bilgi — işlem devam ediyor | — (nadiren görürsün) |
| **2xx** | Başarılı | — |
| **3xx** | Yönlendirme — başka yere bak | — |
| **4xx** | **İstemci** hatası — isteğinde sorun var | **Sen** |
| **5xx** | **Sunucu** hatası — istek doğru, sunucu beceremedi | **Sunucu tarafı** |

Bu tek ayrım — 4xx mü 5xx mi — teşhisin ilk adımıdır: **hatayı kimin düzelteceğini** söyler.

Sık karşılaşacakların:

| Kod | Ne demek | Tipik sebep |
|---|---|---|
| 200 | OK | — |
| 301 / 302 | Kalıcı / geçici yönlendirme | HTTP→HTTPS, eski URL |
| 304 | Değişmemiş | Cache geçerli, gövde gönderilmedi |
| 400 | Hatalı istek | Bozuk JSON, eksik parametre |
| 401 | Kimlik doğrulanmadı | Token yok/geçersiz |
| 403 | Yetki yok | Kimlik tamam ama izin yok |
| 404 | Bulunamadı | Yanlış path |
| 429 | Çok fazla istek | Rate limit |
| 500 | İç sunucu hatası | Uygulamada istisna |
| **502** | **Bad Gateway** | **Proxy backend'e ulaşamadı / bozuk cevap aldı** |
| 503 | Servis kullanılamıyor | Aşırı yük, bakım, sağlıklı backend yok |
| **504** | **Gateway Timeout** | **Backend yanıt verdi ama çok yavaş / hiç vermedi** |

## 8.2.2 502 ve 504: en değerli ayrım `[uygulama]`

Bu iki kod, reverse proxy'nin (8.5.1) verdiği hatalardır ve **farkları teşhiste altın değerindedir.**

**502 Bad Gateway** — *"Backend'e ulaşamadım."*
Proxy, arkadaki sunucuya bağlanmaya çalıştı ve **bağlanamadı** (connection refused — 5.2.2), veya bağlandı
ama **anlamsız bir cevap** aldı. Anlamı: **backend çökmüş, yeniden başlıyor, yanlış portu dinliyor veya
security group engelliyor.**

**504 Gateway Timeout** — *"Backend'e ulaştım ama cevabı beklerken süre doldu."*
Bağlantı kuruldu, istek iletildi, ama cevap zamanında gelmedi. Anlamı: **backend ayakta ama yavaş** —
uzun süren bir sorgu, kilitlenmiş bir veritabanı, tükenmiş bir thread havuzu.

Farkı tek cümlede:

> **502 = backend cevap veremedi. 504 = backend yavaş cevap verdi (veya hiç vermedi).**

İkisi tamamen farklı yerlere bakmanı gerektirir. 502'de servisin **ayakta olup olmadığına** ve ağ
erişimine bakarsın; 504'te uygulamanın **performansına** ve timeout ayarlarına.

Bir ayrım daha: **503**, proxy'nin değil genelde **uygulamanın kendisinin** verdiği bir "şu an
hizmet veremiyorum" mesajıdır (bakım modu, aşırı yük) — ya da load balancer'ın "sağlıklı hiçbir backend
kalmadı" demesidir.

> **⚠️ Yaygın yanılgı: "5xx gördüm, ağ sorunu var."**
>
> Hayır — tam tersi. **5xx aldıysan, ağ çalışıyor demektir.** Düşün: bir durum kodu almış olman, DNS'in
> çözüldüğünü (Faz 6), TCP bağlantısının kurulduğunu (Faz 5), TLS'in tamamlandığını (8.4), isteğinin
> sunucuya **ulaştığını** ve sunucunun cevabının sana **geri döndüğünü** kanıtlar. Yani Faz 1–7'nin
> tamamı sağlamdır; sorun uygulama katmanındadır. Ağ sorunlarının imzası bir durum kodu **değil**,
> kodun hiç gelmemesidir: timeout, connection refused, "could not resolve host". Bu ayrım, yanlış ekibi
> aramanı önler.

> **🤔 Düşün 8.2** — Bir servis sabah 09:00'da yoğunlaşınca hata vermeye başlıyor. Log'larda önce birkaç
> **504**, sonra giderek artan **502** görülüyor. (a) İlk 504'ler ne anlatıyor? (b) Sonradan gelen
> 502'ler neyin işareti? (c) Bu sıralama sana olayın hikâyesini nasıl anlatıyor?
>
> *(Cevap: fazın sonunda)*

---
---

# 8.3 HTTP/1.1 vs HTTP/2 vs HTTP/3 `[atla]`

Bu bölüm farkındalık düzeyindedir — adları ve tek cümlelik farkları bilmen yeter.

| Sürüm | Taşıyıcı | Ana yenilik | Kalan sorun |
|---|---|---|---|
| **HTTP/1.1** | TCP | Bağlantı yeniden kullanımı (keep-alive) | Bir bağlantıda bir istek sırası — tarayıcı 6 bağlantı açar |
| **HTTP/2** | TCP | **Multiplexing** — tek bağlantıda paralel istekler | TCP seviyesinde head-of-line blocking |
| **HTTP/3** | **UDP (QUIC)** | Akışlar bağımsız — bir kaybın diğerlerini durdurmaması | Yaygınlaşma süreci |

Anlaman gereken zincir şu: HTTP/2, uygulama seviyesindeki sıra sorununu çözdü ama **TCP'nin** kendi
sorununu çözemezdi — TCP'de kaybolan tek bir paket, arkasındaki **tüm** verinin teslimini bekletir
(5.1.2, head-of-line blocking). Çözüm için TCP'nin kendisinden çıkmak gerekti. HTTP/3 bunu yaptı: UDP
üzerine kurulu **QUIC** protokolünü kullanır ve güvenilirliği kendisi yönetir (5.1.2 kutusu). Böylece bir
akıştaki kayıp, diğer akışları durdurmaz.

Bonus: QUIC, TLS'i protokolün içine gömer — bağlantı kurulumu TCP+TLS'e göre daha az tur alır (8.4.1).

---
---

# 8.4 TLS El Sıkışması

## 8.4.1 Paylaşılmayan anahtar `[kavram]`

Faz 7'nin kapanış sorusu buydu: iki taraf, açık bir kanal üzerinden ortak bir gizli anahtara nasıl
ulaşıyor?

Cevap: **anahtarı göndermiyorlar.** Her iki taraf da birer gizli değer üretir, birbirlerine **açık**
değerler gönderir, ve bu açık değerleri kendi gizli değerleriyle birleştirerek **aynı sonucu** hesaplarlar.
Yoldaki dinleyici açık değerlerin ikisini de görür ama gizli değerleri görmediği için aynı sonuca
ulaşamaz. Bu fikre **anahtar değişimi** denir (Diffie-Hellman ailesi).

Bir benzetme: ikiniz de aynı temel boyaya kendi gizli renginizi karıştırıp karışımı birbirinize
gönderiyorsunuz. Gelen karışıma kendi gizli renginizi tekrar ekliyorsunuz — ikinizde de aynı nihai renk
oluşuyor. Dinleyici iki karışımı da gördü, ama onlardan nihai rengi üretemez.

El sıkışmanın akışı (kavram düzeyinde):

```
1. Client Hello   → "TLS 1.3 konuşuyorum, şu şifre takımlarını destekliyorum,
                     bağlanmak istediğim site: example.com (SNI)"
2. Server Hello   → "Şu şifre takımını seçtim" + SERTİFİKA + anahtar değişimi payı
3. (İstemci sertifikayı DOĞRULAR — 8.4.2)
4. Anahtar değişimi tamamlanır → iki taraf da aynı oturum anahtarını HESAPLAR
5. Finished       → "Bundan sonrası şifreli"
```

Kritik nokta: **TLS, TCP'nin üstüne kurulur.** Önce TCP handshake tamamlanır (5.2.1), sonra TLS başlar.
Yani bir HTTPS bağlantısı en az iki el sıkışma içerir — ve `curl -v` çıktısında ikisini de ayrı ayrı
görürsün (8.1.2 kutusu).

![Şekil 8.1 — TLS el sıkışması: istemci ve sunucu açık değerleri değiştirir, istemci sertifika zincirini doğrular, ve iki taraf ortak oturum anahtarını ayrı ayrı hesaplar; anahtar hiçbir zaman yolda taşınmaz.](../diagrams/png/nw-8-01-tls-handshake.png)

Şekilde soldaki istemci ve sağdaki sunucu arasındaki oklar el sıkışma mesajlarıdır; ortadaki kesikli
kutu, **yolda taşınmayan** oturum anahtarını temsil eder — iki tarafta da ayrı ayrı hesaplanır. Alttaki
zincir kutusu sertifika doğrulamasını (8.4.2) gösterir.

## 8.4.2 Sertifika: kimlik kanıtı `[kavram]`

Şifreleme tek başına yetmez. Şifreli konuştuğun kişinin **gerçekten o olduğunu** bilmen gerekir — yoksa
araya giren biriyle mükemmel şifreli bir konuşma yapıyor olabilirsin.

Sertifika bunu çözer. İçinde şunlar vardır: **kimin için** verildiği (alan adı), **açık anahtar**,
**geçerlilik tarihleri** ve **kim tarafından imzalandığı**.

İstemci üç şeyi kontrol eder:

1. **İsim uyuyor mu?** Sertifikadaki alan adı, bağlandığın adresle eşleşiyor mu?
2. **Süresi geçerli mi?** (En sık görülen arıza sebebi — süresi dolmuş sertifika.)
3. **Zincir güvenilir bir köke çıkıyor mu?** Sertifikayı bir ara CA imzalamıştır, onu da bir kök CA —
   ve o kök, işletim sisteminin **güven deposunda** bulunmalıdır.

Üçüncü madde yüzünden "ara sertifika eksik" hatası çıkar: sunucu kendi sertifikasını gönderir ama ara
sertifikayı göndermeyi unutur; bazı istemciler onu başka yoldan bulur (ve çalışır), bazıları bulamaz (ve
hata verir). Belirtisi tipiktir: **"tarayıcıda çalışıyor ama `curl`/uygulamada çalışmıyor."**

**SNI** (Server Name Indication) ise şu sorunu çözer: tek IP'de birçok HTTPS sitesi varsa, sunucu hangi
sertifikayı göndereceğini bilmelidir — ama `Host` başlığı henüz şifreli kanalın içinde ve gönderilmemiştir
(8.1.1). SNI, istenen alan adını **Client Hello'da açıkça** taşır. Yan etkisi: hangi siteye bağlandığın
yoldan görülebilir (içerik değil, sadece isim).

> **💡 Cloud bağlantısı — TLS nerede sonlanır:** Bir ALB arkasında çalışıyorsan, TLS bağlantısı
> **ALB'de sonlanır** (TLS termination). ALB sertifikayı sunar, el sıkışmayı yapar, şifreyi çözer ve
> backend'e **ayrı bir bağlantıyla** iletir (5.5.1 kutusundaki iki-bağlantı modelinin aynısı). Bunun üç
> sonucu vardır: (1) Sertifikayı **ALB üzerinde** yönetirsin — AWS Certificate Manager sertifikayı
> ücretsiz verir ve **otomatik yeniler**, ki bu "süresi dolmuş sertifika" arızasını büyük ölçüde ortadan
> kaldırır (8.6). (2) Backend'e giden trafik varsayılan olarak **şifresizdir** — VPC içinde kalır ama
> uçtan uca şifreleme isteniyorsa backend'de de TLS kurup ALB'yi HTTPS ile konuşacak şekilde
> ayarlamalısın. (3) Backend, istemcinin gerçek IP'sini **göremez** — ALB'nin IP'sini görür; gerçek IP
> **`X-Forwarded-For`** başlığında gelir, ve log'larda doğru IP'yi görmek istiyorsan uygulamanın bu
> başlığı okuması gerekir. Bu üçüncü madde, bulutta en çok gözden kaçan ayrıntılardan biridir.

> **🤔 Düşün 8.3** — Bir site tarayıcıda sorunsuz açılıyor, ama sunucudan `curl https://site.example.com`
> çalıştırınca "unable to get local issuer certificate" hatası veriyor. (a) Muhtemel sebep ne? (b) Neden
> tarayıcı çalışıyor da curl çalışmıyor? (c) Doğru çözüm nedir — ve hangi çözümü **yapmamalısın**?
>
> *(Cevap: fazın sonunda)*

---
---

# 8.5 Reverse Proxy, CDN ve Anycast

## 8.5.1 Reverse proxy: önde duran aracı `[kavram]`

**Reverse proxy** (*rivörs proksi*), istemci ile gerçek sunucu arasında duran bir aracıdır. İstemci
proxy'ye bağlanır; proxy, isteği arkadaki **backend**'e iletir ve cevabı geri taşır.

Normal (forward) proxy istemcinin önünde durur ve istemciyi gizler; reverse proxy **sunucunun** önünde
durur ve sunucuyu gizler. Fark budur.

Ne kazandırır:

- **Backend gizlenir** — dış dünya gerçek sunucuların adresini bilmez (7.2.2'deki koruma fikrinin
  bilinçli hâli)
- **Yük dağıtımı** — birden fazla backend arasında istekleri paylaştırır
- **TLS termination** — şifre çözme işini tek yerde yapar (8.4.2 kutusu)
- **Sağlık kontrolü** — ölü backend'e istek göndermez (DNS round-robin'in yapamadığı şey, Faz 6 S5)
- **Cache, sıkıştırma, rate limiting** — ortak işleri tek noktada toplar

502 ve 504 hatalarını veren şey tam olarak budur (8.2.2): proxy, backend ile arasındaki sorunu sana
bildirir.

## 8.5.2 CDN: içeriği kullanıcıya yaklaştırmak `[kavram]`

Faz 5'te öğrendiğin gerçeği hatırla: **throughput ≈ pencere / RTT** (5.5.2). Mesafe arttıkça performans
düşer, ve ışık hızı bir üst sınırdır — İstanbul'dan Virginia'ya gidiş-dönüş fiziksel olarak ~120 ms'dir.

**CDN** (Content Delivery Network) bu sorunu şöyle çözer: içeriğin kopyalarını dünyaya yayılmış **edge**
sunucularda tutar. Kullanıcı, içeriği kendisine en yakın kopyadan alır. RTT düşer, deneyim iyileşir, ve
asıl sunucunun (origin) yükü azalır.

CDN'in kendi cache mantığı vardır ve Faz 6'daki TTL ile **kardeş bir sorunu** taşır: içeriği
güncellediğinde, edge'lerdeki eski kopyalar TTL'leri dolana kadar servis edilmeye devam eder. Belirtisi:

> **"Bir bölgede eski içerik görünüyor, diğerinde yeni."**

Çözüm iki yönlüdür: **invalidation** (edge'lere "bu yolu unut" demek — anlık ama maliyetli ve yayılması
zaman alır) ve **versiyonlu dosya adları** (`app.a3f91c.js` gibi — içerik değişince ad da değişir, eski
kopya hiç sorulmaz; tercih edilen yöntem budur).

## 8.5.3 Anycast: aynı IP, birçok yer `[kavram]`

CDN'in "en yakın edge"e nasıl yönlendirdiği sorusunun bir cevabı **anycast**'tir.

Anycast (*eni-kast*), **aynı IP adresinin dünyanın birçok noktasından duyurulmasıdır**. Faz 4.6'da
öğrendiğin BGP burada devreye girer: her bölgedeki router, o IP için **kendine en yakın** kopyayı
seçer (4.3 — en iyi yol seçimi). Kullanıcı hiçbir şey yapmaz; routing onu en yakın kopyaya götürür.

Faz 6'da "13 root sunucu aslında yüzlerce makinedir" demiştik (6.1.2 kutusu) — mekanizması budur.
CloudFlare'in `1.1.1.1`'i, Google'ın `8.8.8.8`'i de anycast ile çalışır.

Alternatif yöntem **DNS tabanlı yönlendirmedir**: aynı isme, kullanıcının konumuna göre farklı IP'ler
döndürmek (Route 53'ün latency-based veya geolocation routing'i). Anycast routing katmanında, DNS
yönlendirmesi isim katmanında çalışır; ikisi birlikte de kullanılır.

> **💡 Cloud bağlantısı — üç bileşen, üç iş:** AWS'te bu üç kavramın karşılıkları nettir ve
> karıştırılmamalıdır. **ALB** = reverse proxy (L7): HTTP başlıklarına bakarak yönlendirir, path/host
> bazlı kural yazabilirsin, TLS'i sonlandırır, sağlık kontrolü yapar. **NLB** = L4 yük dağıtıcı: TCP
> seviyesinde çalışır, HTTP'yi anlamaz ama çok yüksek performanslıdır ve statik IP verebilir.
> **CloudFront** = CDN: içeriği edge konumlarda cache'ler, TLS'i edge'de sonlandırır ve origin'e (ALB
> veya S3) tek bağlantıyla gider — böylece uzak kullanıcı için TCP el sıkışması bile yakında biter
> (5.5.2). **Route 53** = DNS tabanlı yönlendirme: latency, geolocation veya failover kurallarıyla
> kullanıcıyı doğru bölgeye gönderir, ama TTL'e tabidir (6.4.3). Tipik bir mimaride üçü birlikte
> bulunur: Route 53 → CloudFront → ALB → backend. Teşhis ederken **hangi katmanın** hata verdiğini
> ayırt etmek zorundasın: CloudFront'un 502'si ile ALB'nin 502'si farklı yerlere bakmanı gerektirir.

> **🤔 Düşün 8.4** — Bir ekip CSS dosyasını güncelledi ve deploy etti. Ofisteki bazı kişiler yeni tasarımı
> görüyor, bazıları eskisini. Sunucuda dosya kesinlikle yeni. (a) Sebep ne olabilir — iki katman say.
> (b) Bu sorunun Faz 6'daki hangi sorunla kardeş olduğunu açıkla. (c) Kalıcı çözüm ne?
>
> *(Cevap: fazın sonunda)*

---
---

# 8.6 Bu Faz Bozulunca — HTTP ve TLS Arıza İmzaları

Bu fazın arızalarının en büyük avantajı şudur: **bir hata mesajı alıyorsan, alt katmanların hepsi
çalışıyor demektir.** Durum kodu veya TLS hatası görmüş olman, DNS'in çözüldüğünü, paketin gidip
döndüğünü ve bağlantının kurulduğunu kanıtlar. Bu yüzden bu fazın teşhisi, alt fazlara göre çok daha
hızlıdır — yeter ki mesajı doğru okuyasın.

| Belirti | Muhtemel sebep | Doğrula | İlgili bölüm |
|---|---|---|---|
| **502** Bad Gateway | Backend çökmüş / yanlış port / SG engelliyor | Backend ayakta mı, `ss -tulpn` | 8.2.2 |
| **504** Gateway Timeout | Backend ayakta ama **yavaş** | Uygulama log'u, sorgu süreleri | 8.2.2 |
| **503** Service Unavailable | Sağlıklı backend yok / bakım / aşırı yük | Target group sağlık durumu | 8.2.1 |
| **404** ama dosya var | Yanlış path, yanlış virtual host | `Host` başlığı, proxy kuralları | 8.1.1 |
| **403** ama giriş yapılmış | Yetki eksik (kimlik değil) | Uygulama izinleri | 8.2.1 |
| **429** | Rate limit | İstek hızı, `Retry-After` başlığı | 8.2.1 |
| "certificate has expired" | Sertifika süresi dolmuş | `curl -v`'de `expire date` | 8.4.2 |
| Tarayıcıda ✓, curl'de sertifika hatası | **Ara sertifika eksik** | `openssl s_client -showcerts` | 8.4.2 |
| "hostname mismatch" | Sertifikadaki isim ≠ bağlandığın isim | `curl -v`'de `subject` | 8.4.2 |
| TLS handshake başlıyor ama bitmiyor | Sürüm/şifre takımı uyuşmazlığı, veya **MTU** | `curl -v`'nin durduğu satır | 8.4.1, 5.7.3 |
| Bir bölgede eski içerik, diğerinde yeni | CDN edge cache'i + TTL | Edge'den `Age` ve `Cache-Control` başlıkları | 8.5.2 |
| Log'larda hep aynı IP görünüyor | TLS/proxy termination — gerçek IP `X-Forwarded-For`'da | Başlığı oku | 8.4.2 kutusu |
| Sayfa açılıyor, API çağrıları başarısız | Farklı yol/port farklı kurala takılıyor | `curl -v` ile API'yi ayrı test et | 8.1.2 |

> **Bu tablodan çıkan ders:** Bu fazın teşhisi iki soruya indirgenir. Birincisi: **bir durum kodu aldın
> mı?** Aldıysan ağ katmanlarının tamamı sağlamdır (8.2.2 kutusu) ve sorun uygulamadadır — ilk rakama
> bak, 4xx ise isteğini, 5xx ise sunucuyu düzelt. Almadıysan sorun daha aşağıdadır ve Faz 5–7'ye
> dönmelisin. İkincisi: **5xx aldıysan, 502 mi 504 mü?** 502 "backend'e **ulaşamadım**" der — servis
> ayakta mı, doğru portu mu dinliyor, security group izin veriyor mu diye bakarsın. 504 "ulaştım ama
> **bekledim**" der — uygulamanın performansına, veritabanı sorgularına ve timeout ayarlarına bakarsın.
> Bu iki kodu karıştırmak, saatlerce yanlış yerde aramak demektir. TLS tarafında ise tek bir araç
> neredeyse her şeyi çözer: **`curl -v`** çıktısının **nerede durduğu**, hangi katmanın bozuk olduğunu
> doğrudan söyler — ve sertifika hatalarında ilk bakacağın iki şey daima **geçerlilik tarihi** ve
> **zincirin tamlığıdır**.

---
---

# Faz 8 — Düşün sorularının cevapları

## Cevap 8.1 — Idempotency ve yeniden gönderme

(a) Çünkü **POST idempotent değildir** (8.1.2). Timeout, isteğin sunucuya ulaşmadığını **kanıtlamaz** —
ulaşmış, işlenmiş ve sadece cevabı kaybolmuş olabilir (Faz 5.3: ACK kaybolabilir). Körlemesine yeniden
gönderirsen **iki kez tahsilat** riski vardır.

(b) **Idempotency key** ile: istemci her mantıksal işlem için benzersiz bir anahtar üretir ve
`Idempotency-Key: <uuid>` başlığıyla gönderir. Sunucu bu anahtarı saklar; aynı anahtarla ikinci bir
istek gelirse işlemi **tekrar çalıştırmaz**, ilk cevabı döndürür. Böylece POST, pratikte idempotent hâle
gelir. Ödeme API'lerinin standart yaklaşımı budur.

(c) **Hayır.** GET idempotenttir (8.1.2) — hiçbir şeyi değiştirmez, yüz kez göndersen de sonuç aynıdır.
Cevap gelmezse güvenle yeniden gönderirsin. Zaten HTTP istemcileri ve proxy'ler idempotent metodları
otomatik olarak yeniden dener; POST'u denemezler.
**İlgili bölüm:** 8.1.2 · **Devamı:** 8.2.2 (504 ve timeout), Faz 5.3 (kayıp ACK).

## Cevap 8.2 — Kodların sırası olayın hikâyesidir

(a) İlk **504'ler**, backend'in **ayakta ama yavaşladığını** söyler (8.2.2): proxy bağlanabiliyor, istek
iletiliyor, ama cevap zamanında gelmiyor. Yani yük artmış, sorgular birikmiş, yanıt süreleri timeout
eşiğini aşmaya başlamış.

(b) Sonradan gelen **502'ler**, backend'e artık **ulaşılamadığını** söyler (8.2.2). Bu, yavaşlamanın bir
adım ötesidir: backend ya çökmüştür (bellek tükenmesi, OOM), ya yeniden başlamaktadır, ya da bağlantı
kabul edemeyecek kadar doludur (accept kuyruğu dolmuş, yeni bağlantılar reddediliyor — 5.2.2'deki RST).

(c) Hikâye şu: **yük arttı → backend yavaşladı (504) → istekler birikti → kaynaklar tükendi → backend
çöktü veya bağlantı kabul edemez hâle geldi (502).** Yani 502 sebep değil, **sonuçtur**; kök sebep ilk
504'lerin başladığı andadır. Bu yüzden teşhiste "ilk hata ne zaman ve hangi tipte başladı" sorusu, "şu
an hangi hatayı alıyorum" sorusundan daha değerlidir. Müdahale de buna göre olur: backend'i yeniden
başlatmak 502'yi geçici olarak giderir ama yük devam ettiği için döngü tekrarlar; asıl çözüm yavaşlamanın
sebebindedir (sorgu, kilit, ölçekleme).
**İlgili bölüm:** 8.2.2 · **Devamı:** Faz 10.1.2 (altı adımlı yöntem), Faz 10.2.3 (`mtr`: yolun neresinde).

## Cevap 8.3 — Eksik ara sertifika

(a) **Ara sertifika (intermediate) eksik** (8.4.2). Sunucu kendi sertifikasını gönderiyor ama zinciri
köke bağlayan ara sertifikayı göndermiyor. `curl`, zinciri güvenilir bir köke çıkaramadığı için "unable
to get local issuer certificate" der.

(b) Çünkü tarayıcılar eksik ara sertifikayı **kendileri tamamlayabilir**: daha önce gördükleri ara
sertifikaları önbelleklerinde tutarlar ve sertifikadaki *AIA* alanındaki adresten eksik parçayı
indirebilirler. `curl` ve çoğu uygulama kütüphanesi bunu yapmaz — sunucunun gönderdiği zincirle yetinir.
Bu yüzden "tarayıcıda çalışıyor, uygulamada çalışmıyor" klasik bir imzadır ve sertifikanın **kendisinin**
geçerli olduğunu bile gösterir; eksik olan zincirdir.

(c) **Doğru çözüm:** sunucu yapılandırmasında **tam zinciri** (fullchain) sun — sertifika + ara
sertifika(lar) birlikte. Let's Encrypt kullanıyorsan `cert.pem` değil `fullchain.pem` göstermelisin;
ALB kullanıyorsan sertifika zincirini yüklerken ara sertifikaları da eklemelisin.
**Yapmaman gereken:** `curl -k` / `verify=False` ile doğrulamayı **kapatmak**. Bu, hatayı gizler ve
bağlantıyı araya girme saldırılarına açık hâle getirir — yani TLS'in var olma sebebini ortadan kaldırır.
Geçici bir teşhis adımı olarak `-k` ile sorunun sertifikada olduğunu doğrulayabilirsin, ama kalıcı
yapılandırmaya asla girmemelidir.
**İlgili bölüm:** 8.4.2 · **Devamı:** Faz 9.5 (güvenlik refleksleri), Faz 10.2 (araç seçimi).

## Cevap 8.4 — Kardeş cache sorunu

(a) İki katman: (i) **CDN edge cache'i** — bazı kullanıcılar eski kopyayı tutan bir edge'e düşüyor,
bazıları güncellenmiş bir edge'e (8.5.2). (ii) **Tarayıcı cache'i** — `Cache-Control: max-age` süresi
dolmadığı için tarayıcı sunucuya hiç sormuyor. (Üçüncü bir ihtimal: kurumsal bir proxy cache'i.)

(b) Faz 6'daki **DNS TTL sorununun** kardeşidir (6.4.2): her iki durumda da **doğru cevap kaynakta
hazırdır**, ama aradaki katmanlar ellerindeki eski kopyayı süresi dolana kadar servis etmeye devam eder.
Ortak ders: **cache'li bir sistemde "değişiklik yayınlandı" ile "herkes yeni hâli görüyor" aynı an
değildir**, ve aradaki süreyi TTL belirler.

(c) **Versiyonlu dosya adları** (8.5.2): `app.css` yerine `app.a3f91c.css`. İçerik değiştiğinde dosya
**adı** da değişir; yeni HTML yeni adı ister, eski kopya hiç sorulmaz ve cache sorunu **yapısal olarak**
ortadan kalkar. Bu yüzden statik varlıklara çok uzun TTL (bir yıl) verilebilir — ad değiştiği için
güvenlidir. HTML dosyasının kendisine ise kısa TTL verilir, çünkü yeni adları o duyurur.
(Invalidation da bir seçenektir ama anlık değildir, maliyetlidir ve her deploy'da tekrarlanması gerekir —
yapısal çözüm değildir.)
**İlgili bölüm:** 8.5.2 · **Devamı:** Faz 6.4 (TTL), Faz 11.6.2 (CloudFront eşlemesi).

---
---

# Faz 8 — Sık sorulan sorular

**S1 — HTTP düz metinse, HTTPS'te ne değişiyor?** Yapı **aynı** kalır — aynı metodlar, aynı başlıklar,
aynı durum kodları. Değişen tek şey, tüm bu metnin bir **TLS kanalının içinden** geçmesidir (8.4.1).
Yoldaki biri sadece hangi IP'ye bağlandığını ve (SNI sayesinde) hangi alan adını istediğini görür;
path'i, başlıkları ve gövdeyi göremez.

**S2 — 401 ile 403 farkı ne?** **401 Unauthorized** = "kim olduğunu bilmiyorum" — kimlik bilgisi yok veya
geçersiz (token eksik/süresi dolmuş). **403 Forbidden** = "kim olduğunu biliyorum ama bunu yapamazsın" —
kimlik tamam, **yetki** yok. İsimlendirme tarihsel bir talihsizliktir; 401 aslında "unauthenticated"
demek ister.

**S3 — TLS ile SSL aynı şey mi?** SSL, TLS'in eski adıdır ve tüm SSL sürümleri artık güvensiz sayılıp
kullanımdan kaldırılmıştır. Bugün kullanılan TLS 1.2 ve TLS 1.3'tür. Günlük dilde "SSL sertifikası"
denmeye devam edilir ama kastedilen TLS'tir.

**S4 — Sertifika neyi kanıtlar, neyi kanıtlamaz?** **Kanıtlar:** bağlandığın sunucunun o alan adının
kontrolüne sahip olduğunu ve trafiğin şifreli olduğunu. **Kanıtlamaz:** sitenin dürüst, güvenli veya iyi
niyetli olduğunu. Bir kimlik avı sitesi de geçerli sertifika alabilir — "kilit simgesi var, güvenli"
çıkarımı yanlıştır.

**S5 — Neden `curl -k` kullanmamalıyım?** Çünkü sertifika doğrulamasını kapatır (Cevap 8.3) — yani
araya giren birinin sahte sertifikasını da kabul edersin ve TLS'in kimlik doğrulama işlevi tamamen
ortadan kalkar. Şifreleme sürer ama **kiminle** şifreli konuştuğunu bilemezsin. Teşhis sırasında sorunu
daraltmak için bir kez kullanılabilir; kalıcı yapılandırmada asla.

**S6 — Reverse proxy ile load balancer aynı şey mi?** Kavramsal olarak örtüşürler. Load balancer'ın asıl
işi **dağıtımdır**; reverse proxy'nin işi **aracılıktır** (gizleme, TLS sonlandırma, cache, kural
uygulama). Pratikte modern ürünlerin çoğu (nginx, ALB) ikisini birden yapar (8.5.1).

**S7 — CDN her şeyi hızlandırır mı?** Hayır. CDN, **cache'lenebilir** içerikte (resim, CSS, JS, video)
çok etkilidir. Her kullanıcıya özel, dinamik cevaplarda (kişisel panel, API sonuçları) cache işe
yaramaz — ama CDN yine de fayda sağlayabilir: TLS el sıkışması kullanıcıya **yakın** biterse bağlantı
kurulum gecikmesi düşer ve edge ile origin arasında optimize edilmiş, uzun ömürlü bağlantılar kullanılır
(5.5.2).

---
---

# Faz 8 — Kendini sına

Cevaplarını bir kâğıda yaz, sonra cevap anahtarıyla karşılaştır. Hedef: 18 sorudan 14+.

## Bölüm A — Tanım ve mekanizma

1. Bir HTTP isteğinin üç parçasını yaz. Başlıkları gövdeden ne ayırır?
2. `Host` başlığı ne işe yarar? TLS'teki karşılığı nedir?
3. Idempotent ne demek? Hangi metodlar idempotenttir?
4. 4xx ile 5xx arasındaki fark, **kime** işaret eder?
5. 502 ile 504 arasındaki farkı tek cümleyle yaz.
6. HTTP/3 neden UDP kullanıyor? Hangi sorunu çözüyor?
7. TLS'te ortak anahtar nasıl elde edilir — gönderilir mi?
8. Sertifika doğrulamasında istemci hangi üç şeyi kontrol eder?

## Bölüm B — Uygula ve teşhis et

9. Hangi tek komutla DNS, TCP, TLS ve HTTP katmanlarını birlikte görürsün?
10. 502 aldın. Bakacağın ilk üç şey ne?
11. 504 aldın. Bakacağın ilk üç şey ne?
12. "Tarayıcıda çalışıyor, curl'de sertifika hatası." Teşhis ve çözüm?
13. Bir bölgede eski içerik görünüyor. İki muhtemel katman ve kalıcı çözüm?
14. Log'larda tüm istekler aynı IP'den geliyor gibi görünüyor. Neden ve gerçek IP nerede?

## Bölüm C — Muhakeme ve bağlantı

15. "5xx aldım, ağ sorunu var" cümlesi neden yanlış? Ne **kanıtlanmış** olur?
16. Bir ödeme isteği timeout verdi. Yeniden göndermeli misin? Tasarım açısından doğru çözüm ne?
17. Önce 504, sonra 502 gören bir sistemde olayın hikâyesini anlat.
18. CDN'in cache sorunu ile DNS'in TTL sorunu neden kardeştir? Ortak ders ne?

---

## Cevap anahtarı

1. **İstek satırı** (metod, path, sürüm), **başlıklar**, **gövde**. Başlıkları gövdeden **boş bir satır**
   ayırır (8.1.1). — 2. Tek IP'de birçok site barındırıldığında hangi sitenin istendiğini söyler (virtual
   hosting). TLS'teki karşılığı **SNI**'dır ve Client Hello'da açıkça taşınır (8.1.1, 8.4.2). —
   3. Aynı isteği iki kez göndermek, bir kez göndermekle **aynı sonucu** verir. GET, PUT, DELETE, HEAD
   idempotenttir; POST değildir (8.1.2). — 4. **4xx = istemci** hatası (isteği sen düzelteceksin);
   **5xx = sunucu** hatası (sunucu tarafı düzeltecek). Yani **hatayı kimin düzelteceğini** söyler (8.2.1).
   — 5. **502 = backend'e ulaşamadım; 504 = ulaştım ama cevabı beklerken süre doldu** (8.2.2). —
   6. Çünkü TCP'de kaybolan tek bir paket arkasındaki tüm verinin teslimini bekletir (head-of-line
   blocking, 5.1.2). HTTP/3, UDP üzerine kurulu QUIC ile akışları **bağımsız** hâle getirir; bir
   akıştaki kayıp diğerlerini durdurmaz (8.3). — 7. **Gönderilmez.** İki taraf da gizli bir değer üretir,
   birbirine **açık** değerler yollar ve aynı sonucu **ayrı ayrı hesaplar**; dinleyici açık değerleri
   görse de gizli değerler olmadan aynı sonuca ulaşamaz (8.4.1). — 8. (i) Sertifikadaki **isim**
   bağlandığın adresle uyuyor mu; (ii) **geçerlilik süresi** dolmuş mu; (iii) **zincir** güvenilir bir
   köke çıkıyor mu (8.4.2).

9. **`curl -v https://<adres>`** (8.1.2 kutusu). — 10. (i) Backend **ayakta mı** (`ss -tulpn`, servis
   durumu); (ii) **doğru portu** mu dinliyor ve proxy doğru hedefe mi gidiyor; (iii) **security
   group/NACL** proxy→backend trafiğine izin veriyor mu (8.2.2). — 11. (i) Uygulama **yanıt süreleri**
   ve yavaş sorgular; (ii) veritabanı kilitleri / bağlantı havuzu tükenmesi; (iii) proxy'nin **timeout
   ayarı** ve backend'in gerçek yanıt süresi (8.2.2). — 12. **Ara sertifika eksik**; tarayıcı zinciri
   kendisi tamamlayabildiği için çalışır, curl yetinmez. Çözüm: sunucuda **fullchain** sun. `-k` ile
   doğrulamayı kapatmak çözüm değildir (8.4.2, Cevap 8.3). — 13. (i) **CDN edge cache'i**, (ii)
   **tarayıcı cache'i**. Kalıcı çözüm: **versiyonlu dosya adları** (`app.a3f91c.css`) (8.5.2). —
   14. Çünkü TLS/proxy **ALB'de sonlanıyor** ve backend, ALB'nin IP'sini görüyor. Gerçek istemci IP'si
   **`X-Forwarded-For`** başlığındadır (8.4.2 kutusu).

15. Çünkü bir durum kodu **almış olman**, DNS'in çözüldüğünü, TCP'nin kurulduğunu, TLS'in tamamlandığını,
    isteğin sunucuya ulaştığını ve cevabın geri döndüğünü **kanıtlar** — yani Faz 1–7 sağlamdır. Sorun
    uygulama katmanındadır. Ağ sorununun imzası kod almak değil, **hiç kod alamamaktır** (timeout,
    refused, çözülemeyen isim) (8.2.2 kutusu). — 16. **Körlemesine hayır** — POST idempotent değildir ve
    işlem geçmiş, sadece cevap kaybolmuş olabilir (Faz 5.3). Doğru tasarım: **idempotency key** —
    istemci benzersiz bir anahtar gönderir, sunucu aynı anahtarlı ikinci isteği tekrar çalıştırmaz, ilk
    cevabı döndürür (Cevap 8.1). — 17. Yük arttı → backend **yavaşladı** (504) → istekler birikti →
    kaynaklar tükendi → backend çöktü veya bağlantı kabul edemez hâle geldi (**502**). 502 sonuçtur,
    kök sebep ilk 504'lerin başladığı andadır; backend'i yeniden başlatmak semptomu geçici giderir,
    döngüyü durdurmaz (Cevap 8.2). — 18. Çünkü ikisinde de **doğru cevap kaynakta hazırdır** ama aradaki
    katmanlar ellerindeki **eski kopyayı** TTL dolana kadar servis eder (6.4.2, 8.5.2). Ortak ders:
    cache'li bir sistemde *"yayınlandı"* ile *"herkes görüyor"* aynı an değildir, ve arayı TTL belirler;
    yapısal çözüm, değişen içeriğe **değişen bir ad** vermektir.

## Puanlama

| Doğru sayısı | Ne anlama geliyor |
|---|---|
| 16-18 | Uygulama katmanı sende oturdu — özellikle 502/504 refleksi. Faz 9'a hazırsın. |
| 13-15 | İyi. 8.2.2'yi bir kez daha oku; sahada en çok o ayrım işine yarayacak. |
| 9-12 | TLS ve durum kodları karışmış olabilir. `curl -v` çıktısını satır satır gez. |
| 0-8 | Fazı yeniden gez. Hedef: durum kodunu görünce hangi ekibi arayacağını bilmek. |

Kaçırdığın soru → dönmen gereken bölüm:

| Soru | Bölüm |
|---|---|
| 1, 2, 3, 9, 16 | 8.1 Request/response |
| 4, 5, 10, 11, 15, 17 | 8.2 Durum kodları |
| 6 | 8.3 HTTP sürümleri |
| 7, 8, 12, 14 | 8.4 TLS |
| 13, 18 | 8.5 Reverse proxy, CDN, anycast |

---
---

# Faz 8 — Kapanış ve Faz 9'a Köprü

## Bu fazdan ne taşıyorsun

Faz 8 sana **yığının en üstünü** verdi. Bir HTTP isteğinin düz metin yapısını, metodların niyet
bildirdiğini ve idempotency'nin neden pratik bir özellik olduğunu öğrendin. Durum kodlarının ilk
rakamının **hatayı kimin düzelteceğini** söylediğini, ve 502 ile 504 arasındaki farkın teşhiste ne kadar
değerli olduğunu gördün. TLS'te anahtarın **gönderilmediğini, hesaplandığını** ve sertifikanın neyi
kanıtlayıp neyi kanıtlamadığını edindin. Reverse proxy, CDN ve anycast'in üç farklı iş yaptığını
ayırdın.

En kalıcı iki cümle: **"durum kodu aldıysan ağ çalışıyor demektir"** ve **"502 ulaşamadı, 504 bekledi."**

## Faz 9 bunun neresine bağlanıyor

Faz 0'dan 8'e kadar hep **paketin nasıl gittiğini** öğrendin. Arızalara baktığımızda da sebep hep bir
eksiklikti: yanlış adres, eksik route, bozuk isim, dolu tablo, büyük paket.

Ama şimdi farklı bir sebeple tanışacaksın: **paket, hiçbir şey bozuk olmadığı hâlde, bilerek düşürüldü.**

Birisi bir kural yazdı ve o kural senin paketini istemedi. Üstelik bunu sana **söylemeden** yaptı —
sessizce. Faz 5.2.2'de gördüğün "SYN gidiyor, cevap yok" imzasının arkasında çoğu zaman bu vardır.

Faz 9 bunun cevabıdır: firewall. Stateful ile stateless filtrelemenin farkını, 5-tuple ile karar
verilmesini, ve bulutta **Security Group ile NACL** arasındaki kritik ayrımı göreceksin — neden SG'de
dönen trafiğe kural yazmana gerek yokken NACL'de gerektiğini, Faz 5'te öğrendiğin TCP durum bilgisine
dayanarak anlayacaksın.

> **🤔 Faz çıktısı — kendine sor:** Bir sunucuya `curl` yaptın ve 30 saniye bekleyip timeout aldın.
> Hiçbir hata mesajı, hiçbir RST, hiçbir ICMP yok — sadece sessizlik (5.2.2). Bir firewall paketini
> düşürdüyse, **neden sana "reddettim" demedi?** Sessiz kalmanın, cevap vermeye göre ne avantajı var?
>
> **🧪 Lab 8 fikri (hepsi 🟢):** (1) `curl -v https://example.com` çalıştır ve çıktıyı dört bölüme ayır:
> DNS, TCP, TLS, HTTP — her bölümün başladığı satırı işaretle. (2) `curl -I https://example.com` ile
> sadece başlıkları al (HEAD metodu) ve `Content-Type`, `Cache-Control`, `Server` başlıklarını oku.
> (3) Olmayan bir path iste (`curl -i https://example.com/yok`) ve **404**'ü gör. (4)
> `curl -v --http1.1` ve `curl -v --http2` ile aynı siteye bağlanıp `ALPN` satırındaki farkı gör.
> (5) `openssl s_client -connect example.com:443 -showcerts </dev/null` ile sertifika **zincirini**
> tam olarak gör — kaç sertifika var? (6) `curl -w '%{time_namelookup} %{time_connect} %{time_appconnect}
> %{time_total}\n' -o /dev/null -s https://example.com` çalıştır: DNS, TCP, TLS ve toplam süreleri
> ayrı ayrı ölç — hangi adım en uzun sürüyor?

---

> **Navigasyon:** [◀ Faz 7 — NAT ve Gerçek Dünya](Faz_7_NAT.md) · **Faz 8** · [Faz 9 — Firewall ve Filtreleme ▶](Faz_9_Firewall.md)
