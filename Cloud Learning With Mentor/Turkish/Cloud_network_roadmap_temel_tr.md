# Cloud Engineering — Network Fundamentals Yol Haritası

---

## Mentor Başlangıç Talimatı — bu dosyayı okuyan AI için

Bu, network fundamentals yol haritasıdır ve sen bu haritayı işleyen öğrencinin **Socratic mentoru**sun. Bu dosya sana verildiyse aşağıdaki kuralları benimse ve öğrencinin yön vermesini bekle. Öğrenci "**Faz X'teyiz**" (fazın başından) ya da "**Faz X.Y'deyiz**" (alt-maddeden) dediğinde, o noktadan **kesintisiz** devam et — baştan özet çıkarma, izin isteme, dağılma.

**Öğrenme ritmi — en önemli kural: yapıcı (constructive) Socratic yöntem.** Bu ne saf soru-cevap ne de düz anlatımdır; ikisinin birleşimidir. Her kavramda sıra şudur: **(1) yönlendirici bir soru sor** — "sence X'in olması için ne gerekiyor olabilir?" gibi, öğrencinin **elindeki mevcut bilgiyle** tahmin yürütebileceği bir soru → **(2) öğrenci tahmin eder** ("şu şu olabilir mi?") → **(3) doğruysa hemen onayla ve terimi ver:** "evet, tam olarak — bunun adı **X**'tir, şu anlama gelir: …" → **(4) yanlışsa dolambaçlı soruyla uğraşma, direkt düzelt** → **(5) terim konduktan sonra üstüne birlikte kavram inşa edin.**

Kritik kural: **Öğrencinin elinde hiç olmayan bir kavramı tahmin ettirmeye çalışma.** Soru, keşif ettirmek için değil, bildiğinden bilmediğine **köprü kurmak** içindir. Cevap gelir gelmez adını koy, tanımını ver, kavramı havada bırakma. Bir kavram tamamlanmadan sonrakine geçilmez.

**Faz açılışı.** Bir faza girerken önce ~2 satır oryantasyon ver: bu faz neyi kapsıyor + önceki fazdan neyi bildiğini varsaydığın + genel bağlam. Ondan sonra ilk kavramın kısa anlatımına geç.

**Dil ve stil.**
- Türkçe. İngilizce teknik terim ilk geçişte parantez içinde Türkçe telaffuz + kısa tanım (dosyadaki "heydır", "sidır", "di-eyç-si-pi" örneklerindeki gibi).
- Derinlik etiketlerine uy: `[kavram]` tanımı bilecek kadar, `[mekanizma]` adım adım anlat, `[uygulama]` makinede gördür, `[atla]` sadece farkındalık.
- Cloud örnekleriyle bağla; her konunun **"Bozulunca"** açısını kullan — arıza içgüdüsü bu haritanın kalbidir.

**Düzeltme — direkt olsun.** Öğrenci yanlış cevap verdiğinde onu doğruya götürecek ikinci bir soru arama; **doğrusunu net biçimde söyle**, ardından kısa gerekçesini ver. Dolaylı yönlendirme yavaşlatır ve yorar. Öğrenci "direkt söyle" dediğinde tartışmasız direkt anlat.

**Lab.** İlgili `[uygulama]` kavramından **hemen sonra** o faza ait Lab'ı öner (dosyadaki komutlarla). Bu bir **öneridir, zorlama değildir.** Öğrenci komutu çalıştırıp çıktıyı getirirse birlikte okursunuz. "Şu an makinede değilim" veya "sonra" derse **zorlama, ısrar etme, akışı bozma** — kısa bir not düş ve devam et. Karar öğrencinindir; sen kararına uyarsın.

**Ara sınavlar.** ✅ ile işaretli checkpoint'lere (Faz 0–2, 3–4, 5–9 sonu) gelince ara sınavı **otomatik öner.** Öğrenci ertelemek isterse **ertele**, dayatma.

**İletişim.** Tetik/komut sistemi yok — doğal konuş. **Öğrenciyi anlamadığın anda anladığını varsayma; açıkça "seni tam anlamadım" de** ve netleştirici soru sor.

**Sınırlar.** Aynı anda tek kavram; konu atlama yok; istenmemiş roadmap sapması yok. Faz-içi ilerlemeyi **sen takip etme** — öğrenci kendi yönetir ve "Faz X.Y'deyiz" diyerek konumu sabitler. Geri bildirim dürüst ve direkt olur.

---

> **Kuzey yıldızı — üç içgüdü sorusu.**
> Bu haritanın tek amacı, bir gün bu üç soruyu düşünmeden cevaplayabilmen:
> 1. **Veri nasıl ilerliyor?** → encapsulation + paketin yolculuğu (Faz 0, 1, 4, 5)
> 2. **Packet neden düşmüş olabilir?** → congestion, MTU, firewall, TTL, buffer (Faz 5, 9, 10)
> 3. **Router neden saçmalıyor?** → routing table, ARP, NAT, MTU black hole (Faz 3, 4, 7)
>
> Tasarım prensibi: troubleshooting sona bırakılmadı. Her fazda
> **"bu bilgi bozulunca nasıl görünür?"** açısı var. İçgüdü ancak böyle oluşur.

## Derinlik etiketleri (fiziksel katman haritasıyla aynı sistem)

- `[kavram]` — sadece ne olduğunu bil, tanımını verebil.
- `[mekanizma]` — adım adım nasıl çalıştığını anlatabil.
- `[uygulama]` — kendi Ubuntu makinende elle çalıştır, gör.
- `[atla]` — farkındalık yeter, derine inme.
- **Cloud bağlantısı** — konunun AWS'te nereye map olduğu.
- **Bozulunca** — bu konu arızalanınca sistemde nasıl görünür (içgüdü tetikleyici).

---

## Faz 0 — Zihinsel Model: Neden Katmanlar Var?

Bu fazın amacı: ağı ezber bir protokol listesi olarak değil, bir **arıza haritası**
olarak kurmak. Encapsulation'ı içselleştirmeden hiçbir troubleshooting içgüdüsü oluşmaz.

### Konular

**0.1 OSI ve TCP/IP modeli**
- Katman fikri: neden problemi dilimlere böleriz `[kavram]`
- OSI 7 katman vs pratikte kullanılan 4 katmanlı TCP/IP `[kavram]`
- Her katman sadece **kendi eşiyle** konuşur (peer-to-peer illüzyonu) `[kavram]`

**0.2 Encapsulation ve Decapsulation**
- Her katmanın veriye kendi header'ını (heydır = başlık) sarması `[mekanizma]`
- Gönderirken sarma (encapsulation), alırken açma (decapsulation) `[mekanizma]`
- **Bozulunca:** yanlış katmanda header bozulursa hata hangi katmanda görünür?

**0.3 PDU'lar — verinin katmandaki adı**
- Frame (freym, L2) → Packet (paket, L3) → Segment (segmınt, L4) → Data (L7) `[kavram]`
- Neden aynı veri her katmanda farklı isim alır `[kavram]`

> **Faz 0 çıktısı:** "Ping attığımda o veri 7 katmandan nasıl iner, karşı tarafta nasıl çıkar" — kendi cümlelerinle çizebilmelisin.

---

## Faz 1 — Adresleme: Bir Makineyi Nasıl Buluruz?

Bu fazın amacı: bir cihazı ağda benzersiz olarak işaret etmenin **iki ayrı katmanda**
(L2 ve L3) niye gerektiğini anlamak. Bu, ARP ve routing'in ön koşulu.

### Konular

**1.1 MAC adresi (L2 kimlik)**
- 48-bit donanım adresi, NIC'e gömülü `[kavram]`
- OUI (üretici) + cihaz kısmı `[atla]`
- MAC neden sadece **yerel ağda** anlamlı `[kavram]`

**1.2 IPv4 anatomisi (L3 kimlik)**
- 32-bit adres, 4 oktet, dotted-decimal (`192.168.1.10`) `[mekanizma]`
- Binary ↔ decimal dönüşümü (Faz 0 dijital temelinden bağlanır) `[uygulama]`
- Network kısmı vs host kısmı ayrımı `[kavram]`

**1.3 Public vs Private IP**
- RFC 1918 private aralıkları (10/8, 172.16/12, 192.168/16) `[kavram]`
- Neden private IP internette route edilemez `[mekanizma]`
- **Cloud bağlantısı:** VPC içi private IP, EC2'nin public IP'si nereden gelir

**1.4 Port ve Socket**
- Port: aynı makinede farklı servisleri ayırma `[mekanizma]`
- Yaygın portlar (22, 53, 80, 443, 3306) `[kavram]`
- Socket = (IP : Port) ikilisi `[kavram]`
- **Bozulunca:** "port zaten kullanımda" hatası tam olarak ne demek

**1.5 IPv6 (farkındalık)**
- Neden var: IPv4 adres tükenmesi `[kavram]`
- 128-bit, hex gösterim `[atla]`

**1.6 DHCP — IP'yi otomatik almak**
- DHCP (di-eyç-si-pi = otomatik IP dağıtım protokolü): statik IP yerine dinamik atama `[kavram]`
- DORA döngüsü: Discover → Offer → Request → Ack `[mekanizma]`
- DHCP sadece IP değil; subnet mask + default gateway + DNS sunucusunu da dağıtır `[mekanizma]`
- **Cloud bağlantısı:** VPC'de DHCP option sets — EC2 private IP'sini ve DNS'ini buradan alır
- **Bozulunca:** DHCP çalışmazsa Windows kendine 169.254.x.x (APIPA) verir; Ubuntu (NetworkManager) genelde hiç IPv4 almaz → "IP yok, ağ yok"

> **Lab 1:** `ip addr` ve `ip link` ile kendi makinendeki MAC + IP'yi bul. `ss -tulpn` ile hangi portların dinlendiğini gör.

---

## Faz 2 — Subnet ve CIDR (Cloud'un Kalbi)

Bu fazın amacı: en kritik ve en yavaş ilerlenecek faz. VPC tasarımının, subnet
bölmenin, security group yazmanın tamamı buraya dayanır. Binary'yi Faz 0'dan biliyorsun;
şimdi onu adres uzayını bölmek için kullanacağız.

### Konular

**2.1 Subnet mask mantığı**
- Mask'in network/host sınırını çizmesi `[mekanizma]`
- `255.255.255.0` ↔ binary karşılığı `[mekanizma]`
- AND işlemi ile network adresi bulma `[mekanizma]`

**2.2 CIDR notasyonu**
- `/24`, `/16`, `/8` ne demek (sidır = ağ büyüklüğü notasyonu) `[mekanizma]`
- Prefix ne kadar büyükse ağ o kadar küçük (ters ilişki) `[kavram]`
- Adres sayısı hesabı: `2^(32 - prefix)` `[uygulama]`

**2.3 Network / Broadcast / Usable range**
- Her subnet'te ilk adres (network) ve son adres (broadcast) ayrılmış `[mekanizma]`
- Kullanılabilir host sayısı = toplam − 2 `[mekanizma]`
- **Cloud bağlantısı:** AWS her subnet'te **5 IP** ayırır (2 değil) — neden

**2.4 Subnetting pratiği**
- Bir `/16`'yı `/24`'lere bölme `[uygulama]`
- Subnet çakışması (overlap) neden felaket `[kavram]`
- **Bozulunca:** iki VPC peering yaparken CIDR'ler çakışırsa ne olur

> **Faz 2 çıktısı:** Sana `10.0.0.0/22` versem — kaç host, network adresi, broadcast adresi, kullanılabilir aralık — kağıt kalemle çıkarabilmelisin.
>
> **Lab 2:** `ipcalc 10.0.0.0/22` ile cevaplarını doğrula.

---

## ✅ Ara Sınav 1 (Faz 0–2)

Encapsulation, adresleme, subnet. Bu üçünü geçemeden L2/L3'e girmek anlamsız.

---

## Faz 3 — Aynı Ağ İçinde (L2 Dünyası)

Bu fazın amacı: iki cihaz **aynı** subnet'teyse veri fiziksel olarak nasıl birbirini
bulur. Fiziksel katman roadmap'inde switch donanımına (Faz 5.2) dokunmuştuk; burada
protokol tarafı.

### Konular

**3.1 ARP (Address Resolution Protocol)**
- IP biliniyor ama MAC bilinmiyor — ARP bu boşluğu doldurur `[mekanizma]`
- ARP request (broadcast) → ARP reply (unicast) `[mekanizma]`
- ARP cache ve zaman aşımı `[kavram]`
- **Bozulunca:** ARP poisoning / yanlış ARP cache → trafik yanlış yere gider

**3.2 Switch ve MAC table**
- Switch MAC adreslerini öğrenerek tablo kurar `[mekanizma]`
- Bilinmeyen hedef → flooding `[kavram]`
- **Router neden saçmalıyor** sorusunun L2 kökü burada başlar

**3.3 Broadcast domain vs Collision domain**
- Switch collision domain'i böler, broadcast domain'i bölmez `[kavram]`
- Router broadcast domain'i böler `[kavram]`

**3.4 VLAN (farkındalık)**
- Tek fiziksel switch'te mantıksal ağ ayrımı `[atla]`

> **Lab 3:** `ip neigh` ile ARP cache'ini gör. `arping` ile bir komşuyu sorgula, request/reply'ı izle.

---

## Faz 4 — Ağlar Arası: Routing (L3)

Bu fazın amacı: hedef **farklı** bir ağdaysa veri oraya nasıl ulaşır. "Router neden
saçmalıyor" sorusunun çekirdeği. Bir cloud engineer'in en çok debug ettiği katman.

### Konular

**4.1 Default gateway**
- Yerel ağda olmayan her şey gateway'e gönderilir `[mekanizma]`
- Cihaz "bu hedef benim subnet'imde mi?" kararını nasıl verir `[mekanizma]`
- **Bozulunca:** yanlış/eksik gateway → "aynı ağ çalışıyor, internet çalışmıyor"

**4.2 Routing table**
- Her cihazın bir routing tablosu var (sadece router'ların değil) `[mekanizma]`
- Destination, gateway, interface, metric sütunları `[kavram]`
- **Cloud bağlantısı:** AWS route table = bu tablonun bulut karşılığı

**4.3 Longest prefix match**
- Birden fazla route eşleşirse **en spesifik** (en uzun prefix) kazanır `[mekanizma]`
- Router'ın karar mantığının kalbi `[mekanizma]`

**4.4 TTL ve ICMP**
- TTL (Time To Live) her hop'ta 1 azalır, 0'da paket ölür `[mekanizma]`
- Loop koruması: sonsuz dönen paketleri öldürür `[kavram]`
- ICMP: hata ve teşhis mesajları (ping bunu kullanır) `[kavram]`
- **Bozulunca:** routing loop → TTL exceeded seli

**4.5 Traceroute mekaniği**
- TTL'i 1, 2, 3... artırarak her hop'u ifşa etme hilesi `[mekanizma]`
- Neden bazı hop'lar `* * *` gösterir `[kavram]`
- Daha önce gerçek bir traceroute çıktısını (ISP → şehir → şehir hop zinciri) incelediysen buraya bağlanır

**4.6 Dinamik routing ve BGP (farkındalık)**
- Statik route (elle yazılan) vs dinamik route (protokolün öğrendiği) farkı `[kavram]`
- BGP (bi-ci-pi = internetin omurga yönlendirme protokolü): AS'ler (otonom sistem = bir ISP'nin ağ bloğu) arası yol duyurusu `[kavram]`
- İç yönlendirme protokolleri (OSPF, RIP) sadece isim farkındalığı `[atla]`
- **Cloud bağlantısı:** Direct Connect, Site-to-Site VPN ve route propagation BGP ile çalışır
- **Bozulunca:** yanlış BGP duyurusu (route leak/hijack) → trafik hatalı AS üzerinden akar veya kaybolur

> **Lab 4:** `ip route` ile kendi tablonu oku. `mtr 8.8.8.8` ile canlı hop-hop yolu izle, hangi hop'ta latency zıplıyor gör.

---

## ✅ Ara Sınav 2 (Faz 3–4)

L2 (ARP/switch) + L3 (routing/gateway). "Veri nasıl ilerliyor" sorusunun yarısı burada tamamlanır.

---

## Faz 5 — Transport Katmanı: Güvenilirlik ve Akış

Bu fazın amacı: paketler kaybolabilirken **güvenilir** iletim nasıl kurulur.
Gerçek dünyadaki packet loss sorunlarının çoğunun kaynağı bu faz.

### Konular

**5.1 TCP vs UDP**
- TCP: güvenilir, sıralı, connection-oriented `[kavram]`
- UDP: hızlı, güvencesiz, connectionless `[kavram]`
- Hangisi nerede (video/DNS vs web/dosya) `[kavram]`

**5.2 Three-way handshake**
- SYN → SYN-ACK → ACK `[mekanizma]`
- Bağlantı neden **kurulmadan** veri gönderilemez `[mekanizma]`
- **Bozulunca:** SYN gidip SYN-ACK gelmiyor → firewall mı, route mu, servis mi kapalı

**5.3 Sequence, ACK ve Retransmission**
- Her byte numaralanır, ACK ile teyit edilir `[mekanizma]`
- Kayıp paket nasıl tespit edilip yeniden gönderilir `[mekanizma]`

**5.4 Flow control (window)**
- Alıcının "yavaş gönder, buffer'ım doluyor" demesi `[mekanizma]`
- Sliding window mantığı `[kavram]`

**5.5 Congestion control**
- Ağ tıkanınca göndericinin hızını kısması `[mekanizma]`
- Slow start, congestion avoidance (kavram düzeyi) `[kavram]`
- **Bozulunca:** yüksek latency + packet loss → throughput çöker (neden)

**5.6 Connection teardown**
- FIN/ACK ile düzgün kapanış, RST ile ani kesme `[kavram]`
- TIME_WAIT durumu neden var `[atla]`

**5.7 MTU, MSS ve Fragmentation ← packet loss'un kalbi**
- MTU (em-ti-yu = bir frame'in taşıyabileceği max boyut, tipik 1500) `[mekanizma]`
- MSS (segment payload boyutu) `[kavram]`
- Fragmentation: büyük paketin bölünmesi `[mekanizma]`
- **MTU black hole:** DF (Don't Fragment) set + araya küçük MTU → paket sessizce ölür `[mekanizma]`
- **Cloud bağlantısı:** VPN/tünel arkasında "SSH bağlanıyor ama takılıyor" klasiği
- **Bozulunca:** en sinsi packet loss buradan gelir — büyük paketler düşer, küçükler geçer

> **Lab 5:** `sudo tcpdump -n port 443` ile gerçek handshake'i yakala. `ss -ti` ile açık bir TCP bağlantısının cwnd/rtt değerlerini gör. `ping -M do -s 1472 8.8.8.8` ile MTU'nu elle test et.

---

## Faz 6 — İsimden Adrese: DNS

Bu fazın amacı: `google.com`'un IP'ye nasıl döndüğü. Sektörün şakası boşuna değil:
"It's always DNS." Cloud'da Route 53, load balancer, service discovery — hepsi buna dayanır.

### Konular

**6.1 DNS hiyerarşisi**
- Root → TLD (.com) → authoritative sunucu zinciri `[mekanizma]`
- Neden dağıtık, neden tek merkez yok `[kavram]`

**6.2 Çözümleme zinciri**
- Recursive resolver'ın işi `[mekanizma]`
- Iterative sorgular: her adımda "bir sonrakine sor" `[mekanizma]`
- **Bozulunca:** resolver ulaşılamıyor → her şey "internet yok" gibi görünür ama ping IP çalışır

**6.3 Kayıt tipleri**
- A, AAAA, CNAME, MX, TXT, NS `[kavram]`
- CNAME zinciri ve tuzakları `[kavram]`

**6.4 Caching ve TTL**
- Her seviyede cache, TTL süresince taze sayılır `[mekanizma]`
- **Bozulunca:** DNS değiştirdin ama eski cache yüzünden trafik hâlâ eski IP'ye gidiyor
- **Cloud bağlantısı:** Route 53 TTL ayarı deployment kesintisini nasıl etkiler

> **Lab 6:** `dig google.com +trace` ile root'tan authoritative'e tüm zinciri gör. `dig` çıktısındaki TTL değerini oku.

---

## Faz 7 — NAT ve Gerçek Dünya

Bu fazın amacı: private IP'li onlarca cihazın tek public IP ile internete nasıl çıktığı.
Ev ağın da bulut ağın da bunsuz çalışmaz.

### Konular

**7.1 NAT mantığı**
- Private kaynak IP → public IP çevirisi `[mekanizma]`
- Dönen trafiği doğru cihaza eşleme problemi `[mekanizma]`

**7.2 PAT / Port forwarding**
- Port bazlı çoklama (çoğu evde olan bu) `[mekanizma]`
- Dışarıdan içeriye erişim için port forwarding `[kavram]`
- **Bozulunca:** NAT arkasındaki servise dışarıdan erişilemez (neden)

**7.3 Cloud NAT**
- **Cloud bağlantısı:** private subnet'teki EC2 internete çıkarken NAT Gateway
- IGW vs NAT Gateway farkı (public çıkış vs sadece giden trafik) `[kavram]`

**7.4 VPN ve tünelleme (farkındalık)**
- Tünelleme (tunneling): bir paketi başka bir paketin içine sarıp public internetten geçirmek `[kavram]`
- IPsec (ay-pi-sek = şifreli tünel protokolü): iki private ağı internet üstünden güvenli bağlama `[kavram]`
- GRE (ci-ar-i = genel amaçlı, şifresiz tünel) ve VXLAN (vi-eks-lan = L2'yi L3 üstünde taşıyan overlay) sadece isim farkındalığı `[atla]`
- **Cloud bağlantısı:** Site-to-Site VPN (IPsec) on-prem ağını VPC'ye bağlar; Faz 4.6 BGP ile route öğrenir
- **Bozulunca:** tünel header'ı ekleyince paket büyür → Faz 5.7 MTU black hole tam burada patlar (tünelde MSS clamp gerekir)

**7.5 IPv6 ve NAT'sız dünya (cloud)**
- IPv6'da adres bolluğu → NAT'a çoğunlukla gerek yok, her cihaz public adres alabilir `[kavram]`
- Dual-stack (dual-stak = IPv4 + IPv6'yı aynı anda konuşma): geçiş dönemi modeli `[kavram]`
- **Cloud bağlantısı:** dual-stack VPC; sadece giden IPv6 için egress-only Internet Gateway (NAT Gateway'in IPv6 karşılığı)
- **Bozulunca:** dual-stack'te uygulama yanlış aileyi (v4/v6) seçerse "bir protokolde çalışıyor, diğerinde takılıyor"

> **Lab 7:** `curl ifconfig.me` ile makinenin dışarıya nasıl göründüğünü (public IP) gör, kendi `ip addr` private IP'nle karşılaştır.

---

## Faz 8 — Uygulama Katmanı: HTTP + TLS

Bu fazın amacı: en üst katman. SSL/TLS içeriğini (handshake, CA doğrulama, reverse
proxy) daha önce çalıştıysan buraya bağlarsın; çalışmadıysan burada kurarız.

### Konular

**8.1 HTTP request/response**
- Metod, path, header, body yapısı `[mekanizma]`
- GET/POST/PUT/DELETE/PATCH mantığı `[kavram]`

**8.2 Status kodları**
- 2xx/3xx/4xx/5xx her grubun anlamı `[kavram]`
- **Bozulunca:** 502 vs 504 farkı ne anlatır (backend down mı, timeout mu)

**8.3 HTTP/1.1 vs 2 vs 3**
- Head-of-line blocking, multiplexing, HTTP/3'ün QUIC/UDP'ye geçmesi `[atla]`

**8.4 TLS handshake**
- Sertifika doğrulama, anahtar değişimi (kavram düzeyi) `[kavram]`
- **Cloud bağlantısı:** ALB'de TLS termination nerede olur

**8.5 Reverse proxy, CDN ve anycast**
- Reverse proxy (rivörs proksi = sunucu önünde duran aracı): istekleri karşılar, arkadaki backend'i gizler `[kavram]`
- CDN (si-di-en = içerik dağıtım ağı): içeriği kullanıcıya en yakın edge sunucudan servis etme `[kavram]`
- Anycast (eni-kast = tek IP'nin birçok konumdan duyurulması): trafiği en yakın kopyaya yönlendirir `[kavram]`
- Anycast'in Faz 4.6 BGP'ye dayandığı; DNS root sunucuları ve CDN'ler böyle çalışır `[atla]`
- **Cloud bağlantısı:** CloudFront (CDN + edge), ALB (reverse proxy), Route 53 latency/geo routing
- **Bozulunca:** yanlış edge/cache → "bir bölgede eski içerik, diğerinde yeni" (Faz 6 TTL ile kardeş sorun)

---

## Faz 9 — Firewall, Filtreleme ve Güvenlik

Bu fazın amacı: paketin **neden bilerek düşürüldüğü**. Daha önce bir host firewall (UFW gibi)
yapılandırdıysan burada mantığını cloud'a taşırsın; taşımadıysan temelini burada kurarız.

### Konular

**9.1 Stateful vs Stateless firewall**
- Stateful: bağlantı durumunu hatırlar, dönen trafiğe otomatik izin `[mekanizma]`
- Stateless: her paketi tek tek, kuralla değerlendirir `[mekanizma]`

**9.2 Packet filtering**
- 5-tuple (kaynak IP, hedef IP, kaynak port, hedef port, protokol) `[kavram]`
- Allow/deny sırası ve öncelik `[kavram]`
- **Bozulunca:** "ping çalışıyor ama uygulama bağlanmıyor" → port kuralı eksik

**9.3 Cloud filtreleme**
- **Cloud bağlantısı:** Security Group (stateful) vs NACL (stateless) — TCP state bilgine (Faz 5) doğrudan bağlanır
- SG'de dönen trafiğe kural yazmana gerek yok, NACL'de gerek var — **neden**

---

## ✅ Ara Sınav 3 (Faz 5–9)

Transport + DNS + NAT + firewall. "Packet neden düştü" sorusunun tüm parçaları burada.

---

## Faz 10 — Troubleshooting: İçgüdüsel Hata Avı

Bu fazın amacı: önceki tüm "bozulunca" notlarını **tek bir sistematik refleks** haline
getirmek. Gerçek cloud engineer farkı burada ortaya çıkar.

### Konular

**10.1 Katman-katman debugging metodolojisi**
- Aşağıdan yukarı: link var mı → IP var mı → gateway/route → DNS → port → uygulama `[mekanizma]`
- Her katmanı bir önceki doğrulanmadan atlamama disiplini `[kavram]`

**10.2 Araç ustalığı (her araç hangi katmana bakar)**
- `ip` / `ip route` → L2/L3 config `[uygulama]`
- `ping` → L3 erişilebilirlik `[uygulama]`
- `arp`/`ip neigh` → L2 çözümleme `[uygulama]`
- `mtr` → hop-hop L3 yolu + kayıp `[uygulama]`
- `dig` → DNS çözümleme `[uygulama]`
- `ss` → yerel port/socket durumu `[uygulama]`
- `tcpdump` → ham paket gerçeği (nihai hakem) `[uygulama]`
- `curl -v` → L7 uçtan uca `[uygulama]`

**10.3 Üç içgüdü sorusunun cevap haritası**
- "Veri nasıl ilerliyor" → Faz 0+1+4+5'in birleşik zihinsel filmi
- "Packet neden düştü" → MTU / firewall / congestion / route / TTL karar ağacı
- "Router neden saçmalıyor" → routing table / ARP / NAT / gateway kontrol listesi

**10.4 Cloud'da paket avı: VPC Flow Logs**
- Yerelde tcpdump neyse, cloud'da Flow Logs o: hangi trafik ACCEPT/REJECT edildi kaydı `[uygulama]`
- Kayıttaki 5-tuple + ACCEPT/REJECT alanı → Faz 9 firewall kararını doğrular `[uygulama]`
- **Cloud bağlantısı:** Flow Logs REJECT'i gösterir; SG mi NACL mı olduğunu trafiğin yönü + statefulness mantığıyla (Faz 9) sen çıkarırsın
- **Bozulunca:** "bağlantı zaman aşımına uğruyor ama neden belli değil" → Flow Logs REJECT satırı gösterir

> **Bitirme Lab'ı:** Kasıtlı bir arıza kur (yanlış gateway, kapalı port, yanlış DNS), sonra sadece metodolojiyle 5 dakikada bul. Bu, içgüdünün gerçek testi.

---

## Faz 11 — Cloud'a Köprü (AWS Mapping)

Bu fazın amacı: her fundamental'i AWS karşılığına oturtmak. SAA hedefine doğrudan çıkış.

### Konular

- **VPC** = kendi izole ağın (Faz 2 CIDR) `[uygulama]`
- **Subnet** (public/private) = Faz 2 + Faz 7 `[uygulama]`
- **Route Table** = Faz 4 routing table `[uygulama]`
- **Internet Gateway / NAT Gateway** = Faz 7 `[uygulama]`
- **Security Group vs NACL** = Faz 9 stateful/stateless `[uygulama]`
- **Route 53** = Faz 6 DNS `[uygulama]`
- **ELB: ALB (L7) vs NLB (L4)** = Faz 5 + Faz 8 `[uygulama]`
- **VPC Peering / Transit Gateway** = Faz 2 CIDR çakışması `[kavram]`
- **Site-to-Site VPN / VPN Gateway** = Faz 7.4 tünelleme + Faz 4.6 BGP `[kavram]`
- **CloudFront** = Faz 8.5 CDN + edge + anycast `[kavram]`
- **Dual-stack VPC / Egress-only IGW** = Faz 1.5 + Faz 7.5 IPv6 `[kavram]`
- **VPC Endpoint / PrivateLink** = internete çıkmadan AWS servisine private erişim, Faz 7 NAT'a alternatif `[kavram]`

> **Final çıktısı:** Boş bir VPC'yi sıfırdan çizip (subnet'ler, route table, IGW, NAT, SG) neden her parçanın orada olduğunu fundamental'lere dayanarak açıklayabilmelisin.

---

## Toplam yapı

| Bölüm | Fazlar | Odak |
|---|---|---|
| Temel | 0–2 | Model, adresleme, subnet |
| Yerel + yönlendirme | 3–4 | L2/L3, "veri nasıl ilerliyor" |
| Güvenilirlik + çevre | 5–9 | Transport, DNS, NAT, firewall — "packet neden düştü" |
| Usta | 10–11 | Troubleshooting içgüdüsü + cloud köprüsü |

**Standing kurallar (fiziksel katman haritasıyla aynı):** Türkçe; teknik terimler İngilizce
kalır, ilk geçişte parantez içinde Türkçe telaffuz + kısa tanım; tek konu, tek soru; direkt
cevap yok; konu atlama yok; yanlışta yönlendiren soru.

---

*Hazırlayan: Denis Ergöçmen, Haziran 2026 — genel kullanım için nötrleştirilmiş sürüm*
