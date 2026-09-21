# Ara Sınav 1 — Faz 0–2: Model, Adres ve Bölme

> **Navigasyon:** [◀ Faz 2 — Subnet ve CIDR](Faz_2_Subnet_ve_CIDR.md) · **Ara Sınav 1** · [Faz 3 — Yerel Ağ (L2) ▶](Faz_3_Yerel_Ag_L2.md)

---

## Bu sınav neyi ölçüyor?

Faz sonundaki "Kendini sına" testleri tek bir fazı yoklar. Ara sınav **farklı** bir şey ölçer: *"Bu
üç fazı birbirine bağlayabiliyor musun?"*

Faz 0 sana **haritayı** verdi: katmanlar, encapsulation, PDU'lar. Faz 1 haritanın üzerine **adresleri**
koydu: MAC, IP, port, DHCP. Faz 2 ise o adreslerin nasıl **bölündüğünü** öğretti: maske, CIDR,
kullanılabilir aralık.

Gerçek bir olayda bu üçü hiç ayrı durmaz. "Makine ağa çıkamıyor" dendiğinde aynı anda üç fazın
bilgisini kullanırsın: paketin hangi katmanda olduğunu (Faz 0), hangi adresleri taşıdığını (Faz 1) ve
o adreslerin aynı ağda olup olmadığını (Faz 2). Maske bir Faz 2 kavramıdır; ama "yanlış maske yüzünden
ARP yapıp gateway'e hiç gitmemek" üç fazın kesişimidir — ve bu, ağda en sık yapılan yapılandırma
hatasıdır.

**Nasıl çalışılır:**

- Kâğıt-kalemle çöz; cevabı yazmadan bir sonrakine geçme. Özellikle hesap sorularında **elle** hesapla.
- Cevap anahtarı her sorunun **hangi fazların kesişiminde** durduğunu söyler — kaçırdığın soru bir
  fazı değil, iki faz arasındaki **köprüyü** işaret eder.
- 21 soru var: Bölüm A (bağlantı muhakemesi, 1–9), Bölüm B (senaryo: "yeni makine ağa çıkamıyor"
  olayı, 10–15), Bölüm C (komut ve çıktı okuma, 16–21).
- Hedef süre: ~1 saat. Ama süre önemli değil; önemli olan her cevabın *neden* öyle olduğunu bir
  cümleyle gerekçelendirebilmek.

> **🤔 Başlamadan önce:** Şu cümleyi kendi kelimelerinle tamamla: *"Bir makinenin IP'si doğru, kablosu
> takılı, gateway'i doğru — ama maskesi yanlış. Paket neden hiç çıkmıyor?"* Bu tek soru üç fazı da
> içeriyor. Cevabını bir kenara yaz; Soru 5'te göreceğiz.

---

# Bölüm A — Bağlantı muhakemesi (1–9)

Bu bölümdeki her soru en az iki fazın bilgisini birleştirmeni ister. Kısa ama gerekçeli cevap ver.

**1.** Faz 0'da "her katman bir alt katmanın hizmetini kullanır" dedik. Faz 1'de MAC ve IP'yi ayrı ayrı
gördük. Bir paket ağlar arasında ilerlerken **hangi adres değişir, hangisi sabit kalır** — ve bu ayrım
Faz 0'ın hangi ilkesinin doğrudan sonucudur?

**2.** Encapsulation'da (Faz 0.2) her katman kendi başlığını ekler. Faz 1'de port numarasını gördük.
Port numarası hangi katmanın başlığında taşınır, ve bir router (L3'te çalışan bir cihaz) bu bilgiyi
**neden normalde okumaz**?

**3.** Faz 1'de "IP adresi bir kimlik değil, bir konum bildirir" dedik. Faz 2'de maskeyi öğrendik. Bu
iki cümleyi birleştir: maske, bir IP adresinin hangi kısmının "konum", hangi kısmının "kimlik" olduğunu
nasıl belirler?

**4.** Faz 1.6'da DHCP'nin DORA akışını gördük. Bir makine `169.254.x.x` (APIPA) adresi almışsa, DORA'nın
hangi adımı gerçekleşmemiştir ve bu bilgi arızayı hangi **katmana** (Faz 0) daraltır?

**5.** Bir makinenin IP'si `10.0.1.50`, maskesi `/16` (yanlış; doğrusu `/24` olmalıydı), gateway'i
`10.0.1.1`. Hedef `10.0.5.20`. Makine bu pakete ne yapar ve **neden** gateway'e hiç göndermez? Cevabını
Faz 2'nin maske karşılaştırması ve Faz 0'ın katman mantığıyla birlikte ver.

**6.** Faz 2.3.2'de bir subnet'te iki adresin (network ve broadcast) kullanılamadığını gördük; AWS'te
ise **beş** adres rezerve edilir. Bu farkın sebebi nedir, ve rezerve edilen `.1` ile `.2` adresleri
hangi fazların hangi kavramlarına karşılık gelir?

**7.** Faz 1.4'te socket'in 4-tuple ile tanımlandığını gördük. İki ayrı tarayıcı sekmesi aynı sunucunun
aynı portuna bağlanıyor. Bu iki bağlantı birbirine karışmadan nasıl ayırt edilir — 4-tuple'ın **hangi**
elemanı farklıdır ve bu eleman nereden gelir?

**8.** Faz 2.4'te VLSM'i gördük: farklı boyutlarda subnet'ler. Neden herkese eşit boyutta subnet vermek
yerine değişken boyut kullanılır — ve bu tercih Faz 1.3'teki hangi kıtlıkla ilgilidir?

**9.** Faz 0'da PDU adlarını gördük (segment, packet, frame). Bir arıza raporunda "frame düşüyor"
ile "packet düşüyor" cümleleri **aynı şeyi** mi söyler? Fark, teşhiste hangi katmana bakacağını nasıl
değiştirir?

---

# Bölüm B — Senaryo: "yeni makine ağa çıkamıyor" olayı (10–15)

> Bir ofis ağında yeni bir Ubuntu makinesi kurdun. Ağ `192.168.10.0/24`, gateway `192.168.10.1`.
> Makine ağa çıkamıyor. Aşağıdaki altı soru bu makinede sırayla yaptığın teşhis.

**10.** İlk komutun `ip addr`. Çıktıda arayüz `UP` ama `inet` satırı **yok**. Bu tek gözlem sana ne
söyler ve ne söyle**mez**? Faz 1'in hangi mekanizması çalışmamıştır, ve bir sonraki adımın ne olur?

**11.** DHCP sunucusunu düzelttiler, makine yeniden IP aldı: `169.254.8.12/16`. Bu adres ne anlama
gelir? DORA'nın (Faz 1.6) hangi adımına kadar gelinmiş, hangisinde takılmıştır? Bu adresle makine
kimlerle konuşabilir?

**12.** DHCP sonunda çalıştı: makine `192.168.10.57/24`, gateway `192.168.10.1` aldı. Ama yöneticilerden
biri "sabit IP verelim" deyip elle şu ayarı yaptı: `192.168.10.57/16`. Şimdi makine kendi ağındaki
`192.168.10.30`'a ping atabiliyor mu? Ya `192.168.20.5`'e? Her iki cevabı da maske hesabıyla gerekçelendir.

**13.** Başka bir makineye yanlışlıkla aynı IP (`192.168.10.57`) verilmiş. Belirti "bağlantı bazen
çalışıyor, bazen kopuyor" şeklinde. Bu neden **aralıklı** bir arızadır — hangi tablo (Faz 1, Faz 3'e
bakacağız ama temeli burada) iki farklı cevap alıyor?

**14.** Yönetici subnet'i büyütmek istiyor ve `192.168.10.0/24` yerine `192.168.10.0/23` yapmayı
öneriyor. (a) Yeni aralık ne olur? (b) Kaç kullanılabilir adres verir? (c) Mevcut makinelerin ayarlarını
değiştirmek gerekir mi — neden?

**15.** Kök neden bulundu: DHCP havuzu `192.168.10.100–192.168.10.150` aralığındaydı ve doluydu. Üç
çözüm öneriliyor: (a) havuzu genişletmek, (b) lease süresini kısaltmak, (c) subnet'i `/23`'e çıkarmak.
Her birini Faz 1 ve Faz 2 mantığıyla değerlendir — hangisi en hızlı, hangisi en doğru, hangisi en çok
iş yaratır?

---

# Bölüm C — Komut ve çıktı okuma (16–21)

Aşağıdaki çıktıların her birinde ilgili satırları okuyup soruyu cevapla.

**16.** `ip addr show` çıktısı:

```
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    link/ether 3c:52:82:1a:4f:07 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.57/24 brd 192.168.10.255 scope global dynamic enp3s0
       valid_lft 42318sec preferred_lft 42318sec
```

`link/ether`, `inet`, `/24`, `brd` ve `valid_lft` sırayla ne anlatıyor? `dynamic` kelimesi hangi fazın
hangi mekanizmasını ele veriyor? Bu makinenin IP'si sabit mi?

**17.** İkinci bir makineden `ip addr`:

```
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
    inet 169.254.112.9/16 scope link enp3s0
```

Bu çıktı 16'dakinden nasıl farklı? `scope link` ne demek ve neden `brd` satırı önemsiz? Bu makine
gateway'ine ping atabilir mi — neden?

**18.** Bir subnet hesabı. `172.16.40.0/22` bloğu veriliyor.

```
Ağ adresi:        ?
Broadcast:        ?
Kullanılabilir:   ? – ?
Toplam host:      ?
```

Dört değeri de elle hesapla ve nasıl bulduğunu bir satırla yaz. Ardından: `172.16.43.200` bu bloğun
**içinde** mi?

**19.** Bir mühendis şu iki adresi "aynı ağda" sanıyor:

```
A: 10.10.10.100/25
B: 10.10.10.200/25
```

Aynı ağdalar mı? Hesapla. Eğer değillerse, A'dan B'ye giden paket hangi yolu izler ve bu Faz 0'ın
hangi katmanında karar verilir?

**20.** `ss -tulpn` çıktısı:

```
Netid State   Local Address:Port   Peer Address:Port  Process
tcp   LISTEN  0.0.0.0:22           0.0.0.0:*          sshd
tcp   LISTEN  127.0.0.1:5432       0.0.0.0:*          postgres
udp   UNCONN  0.0.0.0:68           0.0.0.0:*          dhclient
```

Üç satırı da yorumla. (a) Hangi servise dışarıdan bağlanılabilir? (b) Hangisine bağlanılamaz ve neden?
(c) Üçüncü satırdaki `68` portu hangi fazın hangi mekanizmasına ait?

**21.** Bir ağ planı yapıyorsun. `10.20.0.0/16` bloğun var ve şu ihtiyaçlar:

```
web katmanı:  500 makine
app katmanı:  200 makine
db katmanı:    20 makine
```

Her katman için uygun prefix uzunluğunu seç ve çakışmayan üç blok öner. Neden hepsine aynı boyutu
vermedin — bu hangi tekniğin adı?

---

## Cevap Anahtarı

Her cevabın sonunda o sorunun **hangi fazların kesişiminde** durduğu belirtilmiştir.

**1.** **IP adresleri sabit kalır, MAC adresleri her hop'ta değişir.** Bu, Faz 0'ın katman
bağımsızlığı ilkesinin doğrudan sonucudur: L3 uçtan uca bir adresleme yapar (kaynak ve nihai hedef),
L2 ise yalnızca **bir sonraki fiziksel komşuya** teslim eder. Her router, frame'i açar, yeni bir L2
başlığıyla sarar ve yoluna devam ettirir — paketin içindeki IP'lere dokunmadan. · *Faz 0.1 × Faz 1.1/1.2*

**2.** Port, **L4 (transport) başlığında** taşınır — TCP veya UDP başlığında (Faz 0.2, 1.4). Bir router
L3'te çalışır: kararını **hedef IP'ye** göre verir ve L4 başlığını açmasına gerek yoktur. Encapsulation
mantığı gereği her katman yalnızca kendi başlığını okur. (İstisna: NAT/PAT yapan veya firewall görevi
gören cihazlar L4'e de bakar — Faz 7 ve 9'da göreceksin; bu "katman ihlali" kasıtlıdır.) · *Faz 0.2 ×
Faz 1.4*

**3.** Maske, adresin **kaç bitinin ağ kısmı** olduğunu söyler (2.1). Ağ kısmı "konum"dur — paketin
hangi ağa gideceğini belirler ve router'lar bu kısma bakar. Kalan bitler "kimlik"tir — o ağ içindeki
belirli makineyi seçer. Aynı IP adresi farklı maskeyle farklı bir ağa ait olur; bu yüzden **maske
olmadan bir IP adresi eksik bilgidir.** · *Faz 1.2 × Faz 2.1*

**4.** **Offer (veya Acknowledge) gelmemiştir** — makine Discover yayını yapmış ama cevap alamamıştır
(1.6.2). APIPA, "DHCP sunucusuna ulaşamadım, kendime bir adres uydurdum" demektir. Bu bilgi arızayı
**L2'ye ve altına** daraltır (Faz 0.1): kablo/arayüz sorunu, yanlış VLAN, veya DHCP sunucusunun kendisi.
L3 ve üstünü hiç düşünme — makinenin henüz geçerli bir L3 adresi yok. · *Faz 0.1 × Faz 1.6*

**5.** Makine, hedefi kendi ağında sanır ve **ARP yapar** — gateway'e hiç göndermez. Hesap: `/16`
maskesiyle kendi ağı `10.0.0.0/16`'dır ve `10.0.5.20` bu aralığın **içindedir**. Makine "komşum" der,
o adres için ARP yayını yapar, kimse cevap vermez (çünkü gerçekte başka bir ağdadır) ve paket düşer
(2.1.2, 3.1). Faz 0 tarafı şudur: karar **L3'te** verilir ve yanlış L3 kararı, paketin **L2'de**
kaybolmasına yol açar — belirti alt katmanda görünür, sebep üstte. · *Faz 0.1 × Faz 1.2 × Faz 2.1*

**6.** Klasik bir ağda yalnızca **network** (tüm host bitleri 0) ve **broadcast** (tüm host bitleri 1)
adresleri kullanılamaz (2.3.2). AWS ek olarak üç adres daha rezerve eder çünkü VPC yazılımsal bir ağdır
ve altyapı servisleri için adres ayırır: **`.1` = router/varsayılan geçit** (Faz 4.1'deki gateway
kavramı), **`.2` = VPC DNS çözümleyicisi** (Faz 6'da göreceğin DNS), `.3` ise gelecekteki kullanım için
ayrılmıştır. · *Faz 2.3 × Faz 1.2 (+ ileride Faz 4, 6)*

**7.** Farklı olan eleman **istemcinin kaynak portudur** (ephemeral port, 1.4.1). Her yeni bağlantı
için işletim sistemi kullanılmayan bir port seçer (Linux'ta tipik aralık 32768–60999). Dolayısıyla iki
sekmenin 4-tuple'ları `(kaynak IP, **farklı kaynak port**, hedef IP, 443)` şeklinde ayrışır ve çekirdek
gelen paketleri doğru sokete yönlendirir. Diğer üç eleman aynıdır. · *Faz 1.4 × Faz 0.2*

**8.** Çünkü **adres israf olur** (2.4.2). Herkese `/24` verirsen, 20 makinelik bir katman 254 adres
işgal eder ve 234 adres boşa gider. VLSM ihtiyaca göre boyutlandırır. Bu tercih Faz 1.3'teki **IPv4
adres kıtlığıyla** doğrudan ilgilidir: public adresler tükendiği için özel bloklar dikkatli kullanılır
— ve bulutta da VPC CIDR'ı sınırlı olduğu için aynı disiplin geçerlidir. · *Faz 1.3 × Faz 2.4*

**9.** **Aynı şeyi söylemezler.** Frame bir **L2** PDU'sudur, packet bir **L3** PDU'sudur (0.3). "Frame
düşüyor" dendiğinde switch, kablo, MTU veya arayüz seviyesine bakarsın; "packet düşüyor" dendiğinde
yönlendirme, route tablosu veya filtreleme seviyesine bakarsın. Doğru terim, teşhisin **hangi
katmandan** başlayacağını belirler — bu yüzden PDU adları bir formalite değil, bir teşhis kısayoludur.
· *Faz 0.3 × Faz 0.1*

**10.** Söyler: **fiziksel katman ve arayüz sağlam** — kablo takılı, sürücü çalışıyor (`UP` ve
`LOWER_UP`). Söylemez: neden adres alınamadığını. Çalışmayan mekanizma **DHCP'dir** (1.6): makine ya
Discover gönderemiyor ya cevap alamıyor. Bir sonraki adım: `journalctl -u systemd-networkd` veya
`dhclient` log'una bakmak; paralelde DHCP sunucusunun ayakta olup olmadığını doğrulamak. · *Faz 0.1 ×
Faz 1.6*

**11.** `169.254.x.x` = **APIPA / link-local** (1.6.3): "DHCP sunucusundan cevap alamadım" demektir.
DORA'da **Discover gönderilmiş, Offer alınamamıştır** (1.6.2). Bu adresle makine yalnızca aynı fiziksel
segmentteki diğer link-local makinelerle konuşabilir — gateway'e, başka bir subnet'e veya internete
erişemez, çünkü bu adres yönlendirilebilir değildir. · *Faz 1.6 × Faz 0.1*

**12.** **Kendi ağındaki `192.168.10.30`'a ping atabilir** — `/16` maskesiyle ağı `192.168.0.0/16`
olur ve `.10.30` bu aralıktadır, doğrudan ARP ile ulaşır (yerel switch aynı fiziksel segmentte olduğu
için çalışır). **`192.168.20.5`'e ise atamaz**: makine onu da "kendi ağımda" sanır (çünkü `/16` onu da
kapsar), ARP yapar, cevap gelmez ve paket düşer — gateway'e hiç gitmez (2.1.2). Bu, Soru 5'teki hatanın
aynısıdır ve sahada en sık görülen maske hatasıdır. · *Faz 2.1 × Faz 1.2*

**13.** Çünkü iki makine aynı IP'yi duyurduğunda, **ARP tablosu** (Faz 3.1'de detayını göreceğiz) iki
farklı MAC cevabı alır ve hangisini önbelleğe aldığına göre trafik bir o makineye bir diğerine gider.
Kim en son ARP cevabı verdiyse trafiği o çeker. Belirti aralıklıdır çünkü ARP kayıtlarının **zaman
aşımı** vardır ve her yenilenmede kazanan değişebilir. Bu, Faz 1'in "adres benzersiz olmalıdır"
kuralının ihlalidir. · *Faz 1.2 × Faz 3.1 (önizleme)*

**14.** (a) `192.168.10.0/23` → `192.168.10.0` – `192.168.11.255`. (b) 512 toplam, **510
kullanılabilir** (network ve broadcast düşülür, 2.3.2). (c) **Evet, gerekir:** mevcut makineler hâlâ
`/24` maskesiyle çalışır ve `192.168.11.x`'teki makineleri "başka ağ" sayıp gateway'e gönderirler
(2.1.2). Maske değişikliği **her makinede** yapılmalıdır — DHCP ile dağıtılıyorsa lease yenilendiğinde
otomatik gelir; elle ayarlananlar tek tek düzeltilmelidir. · *Faz 2.2 × Faz 2.3*

**15.** (a) **Havuzu genişletmek** — en hızlı ve en az riskli; `/24` içinde boş adres varsa (örneğin
`.51–.99` ve `.151–.250`) tek ayarla çözülür. (b) **Lease süresini kısaltmak** — ölü kayıtları daha
hızlı geri kazandırır (1.6.4), misafir cihazların çok olduğu ağlarda doğru bir hamledir, ama havuz
gerçekten küçükse yalnızca zaman kazandırır. (c) **`/23`'e çıkarmak** — adres sorununu kökten çözer ama
**en çok iş yaratır**: tüm makinelerin maskesi değişmeli (Soru 14c). Doğru sıra: önce (a), kalıcı
büyüme varsa planlı biçimde (c). · *Faz 1.6 × Faz 2.2/2.4*

**16.** `link/ether` = **MAC adresi** (L2 kimliği, 1.1); `inet` = **IPv4 adresi** (L3, 1.2); `/24` =
**maske** — ağ kısmı 24 bit (2.2); `brd 192.168.10.255` = bu subnet'in **broadcast** adresi (2.3.2);
`valid_lft 42318sec` = **DHCP lease'inin kalan süresi** (1.6.4). `dynamic` kelimesi adresin **DHCP ile**
alındığını ele verir. Hayır, bu IP **sabit değildir** — lease süresi dolduğunda yenilenir ve (nadiren
de olsa) değişebilir. · *Faz 1.1/1.2 × Faz 1.6 × Faz 2.3*

**17.** Fark: adres `169.254.x.x` (APIPA) ve `scope global` yerine **`scope link`**. `scope link`, bu
adresin yalnızca **doğrudan bağlı segmentte** geçerli olduğunu, yönlendirilemeyeceğini söyler. `brd`
satırının önemsiz olmasının sebebi, bu adresle zaten ağ dışına çıkılamamasıdır. **Gateway'ine ping
atamaz:** gateway `192.168.10.1` farklı bir ağdadır ve makinenin ona giden ne bir rotası ne geçerli bir
kaynak adresi vardır (1.6.3). · *Faz 1.6 × Faz 2.1*

**18.** `/22` → son iki oktette 10 host biti; blok boyutu 3. oktette **4**'tür. `40` sayısı 4'ün
katıdır, dolayısıyla: **Ağ = `172.16.40.0`**, **Broadcast = `172.16.43.255`**, **Kullanılabilir =
`172.16.40.1` – `172.16.43.254`**, **Toplam host = 1022** (1024 − 2). Nasıl: `32 − 22 = 10` host biti
→ `2^10 = 1024` adres → blok 3. oktette 4 birim (`40–43`). **Evet**, `172.16.43.200` bloğun içindedir
(`40–43` aralığında ve broadcast değil). · *Faz 2.2 × Faz 2.3*

**19.** **Aynı ağda değiller.** `/25` blok boyutu 128'dir: birinci blok `10.10.10.0–127`, ikinci blok
`10.10.10.128–255`. A (`.100`) birinci blokta, B (`.200`) ikinci bloktadır. Dolayısıyla A, B'yi "başka
ağ" sayar ve paketi **varsayılan geçide** gönderir; oradan yönlendirilir (Faz 4'te detayı). Karar
**L3'te** verilir: makine hedef IP'yi kendi maskesiyle karşılaştırır (2.1.2). Aynı fiziksel switch'e
takılı olsalar bile sonuç değişmez — bu ayrım mantıksaldır, fiziksel değil. · *Faz 2.1/2.3 × Faz 0.1*

**20.** (a) **`sshd`** — `0.0.0.0:22` tüm arayüzleri dinler, dışarıdan bağlanılabilir (1.4.2). (b)
**`postgres`** — `127.0.0.1:5432` yalnızca **loopback**'i dinler; dışarıdan gelen bağlantılar makineye
ulaşsa bile çekirdek onları bu sokete yönlendirmez. Firewall kuralı değil, **bind adresi** sorunudur ve
sahada sık sık firewall sanılır. (c) UDP **68**, DHCP **istemci** portudur (sunucu 67) — Faz 1.6'daki
DORA akışı bu port çifti üzerinden yürür; `UNCONN` durumu UDP'nin bağlantısız olduğunu gösterir. ·
*Faz 1.4 × Faz 1.6*

**21.** İhtiyaca göre en küçük yeterli blok (VLSM, 2.4.2): **web 500 → `/23`** (510 kullanılabilir),
**app 200 → `/24`** (254), **db 20 → `/27`** (30). Çakışmayan bir öneri: `web 10.20.0.0/23`
(`10.20.0.0–10.20.1.255`), `app 10.20.2.0/24`, `db 10.20.3.0/27`. Hepsine aynı boyutu vermedim çünkü
`/23`'ü herkese vermek 1000'den fazla adresi 20 makine için israf ederdi; tekniğin adı **VLSM
(Variable Length Subnet Masking)**. Not: `/24` 500 makineye yetmez — bu yüzden web katmanı bir üst
bloğa çıkar. · *Faz 2.4 × Faz 2.3*

---

## Puanlama

| Doğru sayısı | Değerlendirme |
|---|---|
| 19–21 | Model, adres ve bölme tek bir zihinsel haritada birleşmiş. Faz 3'e güvenle geç. |
| 15–18 | Sağlam. Kaçırdığın soruların işaret ettiği **köprüye** (tek faza değil) bir tur dön. |
| 10–14 | Fazları ayrı ayrı biliyorsun ama arasındaki bağ zayıf. Aşağıdaki tabloyu kullan. |
| 0–9 | Faz 1 ve 2'yi, özellikle maske hesabını ve "Bu faz bozulunca" bölümlerini tekrar et; bu köprüler oturmadan Faz 3 anlamsız gelir. |

**Hangi soruyu kaçırırsan nereye dön:**

| Kaçırdığın soru | Dön — bu köprü zayıf |
|---|---|
| 1, 2, 9 | Faz 0.1/0.2/0.3 — katmanlar, encapsulation, PDU adları |
| 3, 5, 12, 19 | Faz 1.2 × Faz 2.1 — maske ile "aynı ağda mı" kararı |
| 4, 10, 11 | Faz 1.6 × Faz 0.1 — DHCP/DORA ve APIPA'nın katman anlamı |
| 6, 14, 18 | Faz 2.2/2.3 — CIDR hesabı, network/broadcast, rezerve adresler |
| 7, 20 | Faz 1.4 — port, socket, 4-tuple, bind adresi |
| 8, 21 | Faz 1.3 × Faz 2.4 — adres kıtlığı ve VLSM |
| 13, 16, 17 | Faz 1.1/1.2 × Faz 1.6 — adres benzersizliği, lease, link-local |

---

## Kapanış — buradan Faz 3'e

Faz 0, 1 ve 2 birlikte sana ağın **statik** resmini verdi: katmanlar, adresler ve o adreslerin nasıl
bölündüğü. Artık bir makineye bakıp "bu adres hangi ağda, kiminle doğrudan konuşabilir, kiminle
konuşmak için yardıma ihtiyacı var" sorusunu cevaplayabiliyorsun.

Ama bir şeyi hiç sormadın: **makine, "komşum" dediği o adrese paketi gerçekte nasıl ulaştırıyor?**
Faz 2'de defalarca "aynı ağdaysa doğrudan gönderir" dedik — ama IP adresini bilmek yetmez. Kablonun
üstünde yürüyen şey frame'dir ve frame'in **MAC adresine** ihtiyacı vardır (Faz 0.3, 1.1). O MAC
nereden geliyor?

Faz 3 tam buraya bağlanır: **ARP.** "Aynı ağdaysa doğrudan gönderir" cümlesinin altındaki mekanizmayı
açacak, switch'in MAC tablosunu nasıl öğrendiğini görecek, ve Soru 13'te değindiğimiz çift IP
sorununun neden aralıklı olduğunu tam olarak anlayacaksın.

> **Devam etmeden önce:** Yukarıdaki 5. ve 12. soruların cevabını tereddütsüz verebiliyorsan — yanlış
> maskeli bir makinenin neden gateway'e hiç gitmediğini — Faz 3'e hazırsın. Veremiyorsan Faz 2.1 ve
> 2.3'e bir tur dön; Faz 3 "aynı ağda mı?" kararının **doğru** verildiğini varsayarak devam edecek.

---

> **Navigasyon:** [◀ Faz 2 — Subnet ve CIDR](Faz_2_Subnet_ve_CIDR.md) · **Ara Sınav 1** · [Faz 3 — Yerel Ağ (L2) ▶](Faz_3_Yerel_Ag_L2.md)
