# Faz 1 — Adresleme: Bir Makineyi Nasıl Buluruz?

> **Navigasyon:** [◀ Faz 0 — Zihinsel Model](Faz_0_Zihinsel_Model.md) · **Faz 1** · [Faz 2 — Subnet ve CIDR ▶](Faz_2_Subnet_ve_CIDR.md)

---

## Nereden geliyoruz

Faz 0'ın sonunda sana bir soru bıraktık: MAC her hop'ta değişiyorsa, makinen uzak bir sunucuya paket
yollarken **hangi MAC'i** yazar? O soruyu cevaplamaya bu fazda başlıyoruz — ama tam cevap Faz 3'te
gelecek, çünkü önce adreslerin **kendisini** tanıman gerekiyor.

Faz 0'dan üç şey taşıyorsun ve üçü de burada doğrudan işe yarayacak:

- **Her katmanın kendi adresi var.** L2'de MAC, L3'te IP, L4'te port. Bu faz o üç adresin üçünü de tek tek
  açıyor — ne oldukları, neye benzedikleri, hangi soruya cevap verdikleri.
- **MAC yereldir, IP uçtan ucadır** (0.2.2). Bu cümlenin *sebebini* burada öğreneceksin: MAC'in yapısı
  gereği neden route edilemeyeceğini.
- **Header, bir sonrakini işaret eder** (0.2.2). L4 header'ındaki **port**, "hangi uygulamaya teslim
  edeyim" sorusunun cevabıdır — ve bu fazda portu tam olarak bu gözle tanıyacağız.

## Bu fazın sorusu

> *"Dünyada milyarlarca cihaz var. Benim paketim, aralarından **tam olarak bir tanesine** — hatta o
> cihazdaki tam olarak bir **uygulamaya** — nasıl ulaşıyor?"*

Cevap tek bir adres değil, **üç kademeli bir huni**: MAC "bu yerel ağdaki hangi cihaz", IP "internetteki
hangi makine", port "o makinedeki hangi uygulama". Üçü olmadan teslimat tamamlanmaz, ve üçü **farklı
katmanlarda** yaşadığı için üçü **farklı şekillerde bozulur**.

Bu ayrım, sahada sürekli karşına çıkacak: "makineye ping atabiliyorum ama servise bağlanamıyorum" tam olarak
"IP doğru, port yanlış/kapalı" demektir. "Aynı ağdakilere erişiyorum ama internete çıkamıyorum" tam olarak
"L2 çalışıyor, L3 yolu yok" demektir. Adres kademelerini ayırt edebilmek, bu cümleleri **çevirebilmektir**.

Bu fazın sonunda bir `ip addr` çıktısına baktığında satırları ezberden değil **anlamlarıyla** okuyacak; bir
makinenin IP'sini nereden aldığını (DHCP) ve o mekanizma bozulunca ne göründüğünü bileceksin.

---

## Bu fazın sonunda

- MAC adresinin ne olduğunu, kaç bit olduğunu ve **neden sadece yerel ağda anlamlı** olduğunu
  açıklayabileceksin
- Bir IPv4 adresinin anatomisini (32 bit, 4 oktet, dotted-decimal) çözebilecek; binary ↔ decimal dönüşümünü
  elle yapabileceksin
- Bir adresin **network** kısmı ile **host** kısmı ayrımını anlatabilecek — Faz 2'nin tüm temeli budur
- RFC 1918 private aralıklarını (10/8, 172.16/12, 192.168/16) tanıyabilecek; private bir IP'nin internette
  neden route **edilemeyeceğini** açıklayabileceksin
- Port ve socket kavramlarını ayırt edebilecek; well-known portları tanıyacak; "address already in use"
  hatasının tam olarak ne anlama geldiğini bileceksin
- IPv6'nın neden var olduğunu farkındalık düzeyinde anlatabileceksin
- DHCP'nin DORA döngüsünü adım adım anlatabilecek; DHCP'nin sadece IP değil **mask + gateway + DNS** de
  dağıttığını bilecek; DHCP bozulunca ortaya çıkan imzayı (169.254.x.x / hiç IP yok) tanıyacaksın
- **Cloud:** VPC içindeki private IP'nin nereden geldiğini, EC2'nin public IP'sinin **makinenin içinde neden
  görünmediğini** ve DHCP option set'lerin ne işe yaradığını anlatabileceksin

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden burada |
|---|---|---|---|
| 1.1 | MAC adresi (L2 kimlik) | `[kavram]` | Yerel teslimatın adresi — Faz 3'ün ön koşulu |
| 1.2 | IPv4 anatomisi (L3 kimlik) | `[mekanizma]` | **Fazın kalbi** — network/host ayrımı Faz 2'yi doğurur |
| 1.3 | Public vs private IP | `[kavram]` | Bulut ağının tüm mimarisi bu ayrım üstünde |
| 1.4 | Port ve socket | `[mekanizma]` | "Hangi uygulama" sorusu — L4 kimliği |
| 1.5 | IPv6 (farkındalık) | `[atla]` | Neden var olduğunu bil, derine inme |
| 1.6 | DHCP — IP'yi otomatik almak | `[mekanizma]` | Makine adresini nasıl alır; en sık ilk arıza |
| 1.7 | Bu faz bozulunca | — | Adresleme arızalarının imzaları |

> **Bu fazda nasıl çalışmalı:** Bu fazın tüm gözlem komutları 🟢'dir (`ip addr`, `ip link`, `ss -tulpn`,
> `ip -4 addr show`) — hiçbiri sistemini değiştirmez, hepsini gönül rahatlığıyla çalıştır. Tek dikkat
> noktası 1.2'deki **binary dönüşümüdür**: bunu okuyup geçme, **elle yap**. Üç dört adresi kâğıtta binary'ye
> çevirmeden Faz 2'ye (subnet) girme — orada bu beceri her satırda gerekecek ve eksikse Faz 2 bir işkenceye
> döner. En aydınlatıcı deney: `ip addr` çıktısındaki kendi IP'ni al, binary'ye çevir, sonra yanındaki `/24`
> gibi prefix ile hangi kısmının "network", hangi kısmının "host" olduğunu işaretle. Faz 2'ye bu kâğıtla
> gel.

---
---

# 1.1 MAC Adresi — L2 Kimliği

## 1.1.1 48 bitlik donanım adresi `[kavram]`

Faz 0'da L2'nin işini "aynı yerel ağda komşuya teslim" diye tanımlamıştık. O teslimatın adresi **MAC
adresidir** (Media Access Control — *mek adres*). Üç özelliğiyle tanı:

**48 bittir** ve genelde altı onaltılık (hex) çiftle yazılır: `3c:52:82:1a:0b:77`. 48 bit, yaklaşık 281
trilyon farklı adres demektir — yani dünyadaki her ağ arayüzüne benzersiz bir tane düşecek kadar geniş.

**Donanıma gömülüdür.** MAC, ağ arayüz kartının (NIC) üreticisi tarafından üretim anında yazılır. Bu yüzden
"donanım adresi" (hardware address) veya "fiziksel adres" de denir. Yazılımla değiştirilebilir (MAC spoofing)
ama varsayılan olarak kartla birlikte gelir ve makineyle birlikte hareket eder.

**İki parçadan oluşur.** İlk 24 bit **OUI**'dir (Organizationally Unique Identifier) — üreticiyi gösterir;
son 24 bit o üreticinin o karta verdiği seri numarasıdır. Bir MAC'in ilk üç oktetinden hangi firmanın kartı
olduğunu bulabilirsin. Bu detay `[atla]` düzeyindedir: bilmen yeterli, ezberlemen gereksiz.

Bir de özel bir adres var, şimdiden tanı: **`ff:ff:ff:ff:ff:ff`** = **broadcast** adresi. Bu hedefle
gönderilen bir frame, yerel ağdaki **herkese** gider. Faz 3'te ARP'nin tam olarak bunu kullandığını
göreceksin.

> **🔧 Makinende gör** 🟢 — arayüzlerini ve MAC adreslerini oku
>
> ```
> $ ip link
> 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
>     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
> 2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 ...
>     link/ether 06:3c:52:82:1a:0b brd ff:ff:ff:ff:ff:ff
> ```
>
> `link/ether` satırındaki ilk adres bu arayüzün **MAC adresidir** — L2 kimliğin. Yanındaki
> `brd ff:ff:ff:ff:ff:ff` ise bu arayüzün broadcast adresidir (yerel ağda herkes). `lo` (loopback) arayüzünün
> MAC'i sıfırdır çünkü fiziksel bir ağa hiç çıkmaz. `mtu 9001` alanını şimdilik görmezden gel — Faz 5.7'de
> onun ne kadar önemli olduğunu göreceksin.

## 1.1.2 MAC neden sadece yerel ağda anlamlı `[kavram]`

Şimdi Faz 0'da söz verdiğimiz cevabı verelim: neden MAC ile internet üzerinden haberleşemiyoruz?

Cevap tek kelime: **hiyerarşi yok.** MAC adresleri **düzdür** (flat). `3c:52:82:...` adresinden o cihazın
hangi ülkede, hangi şehirde, hangi ağda olduğuna dair **hiçbir bilgi çıkarılamaz**. Adres sadece "kim"
der, "nerede" demez.

Bunun sonucu felakettir: eğer internet MAC ile çalışsaydı, dünyadaki her router'ın tablosunda **her
cihazın** MAC'i olması gerekirdi — milyarlarca satır, ve her biri gruplanamaz. Posta sisteminin, ülke ve
şehir olmadan sadece kimlik numaralarıyla çalışmaya çalışması gibi.

IP adresi bu sorunu **hiyerarşiyle** çözer: `10.0.5.20` adresinin bir "network" kısmı vardır ve o kısım,
tüm bir adres grubunu tek satırda temsil eder. Bir router "10.0.5.0 ile başlayan her şey şu yöne" diyebilir
— tek satır, 254 makine. MAC'te böyle bir gruplama **mümkün değildir**.

İşte bu yüzden iki adres birden var (Faz 0, S4): **MAC yerel teslimat, IP küresel yol bulma içindir.**
Ve bu yüzden MAC her hop'ta değişir — her hop yeni bir "yerel teslimattır".

> **⚠️ Yaygın yanılgı: "MAC adresi makineye aittir."**
>
> Hayır, **arayüze** aittir. Bir makinede üç ağ kartı varsa (ethernet, Wi-Fi, sanal bir köprü) **üç ayrı MAC**
> vardır. `ip link` çıktında her satırın kendi `link/ether` değeri olması bunun kanıtıdır. Bulutta bu ayrım
> daha da görünür hâle gelir: bir EC2 instance'ına ikinci bir ENI (Elastic Network Interface) eklersen,
> makine ikinci bir MAC **ve** ikinci bir IP kazanır. Kimlik makinenin değil, **arayüzün** özelliğidir —
> ve bu, bir makinenin aynı anda birden fazla ağda durabilmesinin temelidir.

> **🤔 Düşün 1.1** — Aynı yerel ağda A ve B makineleri var, ve aralarında bir switch. A, uzaktaki bir
> internet sunucusuna (`93.184.216.34`) paket yolluyor. (a) A'nın yazdığı **hedef IP** nedir? (b) A'nın
> yazdığı **hedef MAC** hangi cihaza aittir — uzak sunucuya mı, yoksa başka bir şeye mi? (c) Cevabını
> Faz 0.2.2'deki "MAC her hop'ta değişir" cümlesine bağla.
>
> *(Cevap: fazın sonunda)*

---
---

# 1.2 IPv4 Anatomisi — L3 Kimliği

## 1.2.1 32 bit, 4 oktet, dotted-decimal `[mekanizma]`

Bir IPv4 adresi aslında **32 bitlik tek bir sayıdır**. Ama 32 bit uzunluğunda bir ikili diziyi insanların
okuması imkânsız olduğu için, onu dörde bölüp her parçayı ondalık yazarız:

```
11000000 10101000 00000001 00001010     ← gerçekte bu (32 bit)
   192  .   168  .    1   .   10        ← biz böyle yazıyoruz
```

Her 8 bitlik parçaya **oktet** (octet) denir. Bir oktet 8 bit olduğu için alabileceği değer aralığı
`00000000` ile `11111111` arasıdır — yani **0 ile 255**. Bu yüzden `192.168.1.300` diye bir adres olamaz:
300, bir oktete sığmaz. Bu yazım biçimine **dotted-decimal** (noktalı ondalık) denir.

Burada anlaman gereken tek şey şu: **noktalar sadece okumak içindir.** Ağ donanımı için adres tek bir 32
bitlik sayıdır; noktaların matematiksel bir anlamı yoktur. Bu, Faz 2'de kritik olacak — çünkü subnet
sınırları noktalarla **hizalı olmak zorunda değildir**.

## 1.2.2 Binary ↔ decimal: elle dönüştürmek `[uygulama]`

Bunu okuyup geçme — Faz 2'nin tamamı bu beceriye dayanıyor. Neyse ki sadece sekiz sayı ezberlemen yeterli:
bir oktetteki her bitin **ağırlığı**.

| Bit pozisyonu | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
|---|---|---|---|---|---|---|---|---|
| **Değeri** | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |

**Binary → decimal:** 1 olan bitlerin değerlerini topla.

```
11000000  →  128 + 64                      = 192
10101000  →  128 +      32 +        8      = 168
00001010  →                 8 +        2   = 10
11111111  →  128+64+32+16+8+4+2+1          = 255
```

**Decimal → binary:** soldan başla, sayı o bitin değerinden büyük veya eşitse 1 yaz ve çıkar, değilse 0 yaz.

```
172 →  128? evet (1), kalan 44
       64?  hayır (0)
       32?  evet (1), kalan 12
       16?  hayır (0)
       8?   evet (1), kalan 4
       4?   evet (1), kalan 0
       2?   hayır (0)
       1?   hayır (0)
       = 10101100
```

Birkaç kez yaptıktan sonra bu refleks olur. Ezberlemeye değecek üç değer: `255 = 11111111` (hepsi 1),
`0 = 00000000` (hepsi 0), `128 = 10000000` (sadece ilk bit).

## 1.2.3 Network kısmı vs host kısmı `[kavram]`

Şimdi bu fazın en önemli fikrine geldik — ve bu, doğrudan Faz 2'nin kapısıdır.

Bir IP adresi tek parça değildir; **iki mantıksal parçaya** bölünür:

- **Network kısmı** (soldaki bitler) — "hangi ağ". Aynı ağdaki tüm makinelerde **aynıdır**.
- **Host kısmı** (sağdaki bitler) — "o ağdaki hangi makine". Her makinede **farklıdır**.

Telefon numarası benzetmesi iyi çalışır: `0212` alan kodudur (network — tüm İstanbul'da aynı), kalan
kısım abone numarasıdır (host — her hatta farklı).

Örnek: `192.168.1.10` adresi ve `/24` sınırı (bunu Faz 2'de tam açacağız) ile:

```
 192  .  168  .   1  .   10
 └──────── network ────────┘ └host┘
      ilk 24 bit                 son 8 bit
```

Yani `192.168.1.10` ve `192.168.1.77` **aynı ağdadır** (ilk 24 bit aynı), ama `192.168.2.10` **farklı
ağdadır** (üçüncü oktet farklı).

Bu ayrımın neden bu kadar önemli olduğunu şimdi söyleyelim, çünkü Faz 4'e kadar taşıyacaksın: **bir makine,
paketi göndermeden önce ilk olarak şu kararı verir — "hedef benim ağımda mı, değil mi?"** Cevap:

- **Aynı ağdaysa** → doğrudan teslim et (L2 ile, komşuna — Faz 3).
- **Farklı ağdaysa** → varsayılan geçide (router'a) yolla (Faz 4).

Bu tek karar, ağın çalışma mantığının çekirdeğidir ve makine onu **network kısmını karşılaştırarak** verir.
"Network kısmı nereden nereye" sorusunun cevabı ise **subnet mask'tir** — Faz 2'nin konusu.

> **🔧 Makinende gör** 🟢 — kendi IP'ni ve prefix'ini oku
>
> ```
> $ ip -4 addr show
> 1: lo: <LOOPBACK,UP,LOWER_UP> ...
>     inet 127.0.0.1/8 scope host lo
> 2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
>     inet 172.31.20.15/20 brd 172.31.31.255 scope global dynamic ens5
> ```
>
> `inet 172.31.20.15/20` satırını parçalarına ayır: `172.31.20.15` makinenin IP'si, `/20` ise network
> kısmının **kaç bit** olduğu — yani ilk 20 bit network, kalan 12 bit host. `brd 172.31.31.255` bu ağın
> broadcast adresi (Faz 2.3). `scope global` "bu adres dış dünyaya yönelik" demek; `lo`'daki `scope host`
> ise "sadece bu makinenin içinde geçerli". `dynamic` kelimesi de önemli: bu adres elle değil **DHCP ile**
> alınmış (1.6). Bu tek satır, bu fazın üç konusunu birden içeriyor.

> **❓ Akla gelen soru: "`/20` ve `/24` neyi değiştiriyor — adres aynı kalıyor gibi görünüyor?"**
>
> Adres aynı kalıyor ama **"aynı ağ" tanımı** değişiyor, ve bu her şeyi değiştirir. `/24` ise network
> kısmı ilk 24 bittir: `172.31.20.15` ile `172.31.21.15` **farklı ağlardadır** (üçüncü oktet farklı).
> `/20` ise network kısmı ilk 20 bittir ve üçüncü oktetin sadece bir kısmını kapsar: bu ikisi **aynı
> ağdadır**. Aynı adres, farklı prefix, tamamen farklı komşuluk. Ve "aynı ağda mı" sorusunun cevabı
> değişince (1.2.3), makinenin paketi doğrudan mı yoksa gateway'e mi yollayacağı da değişir. İşte bu yüzden
> Faz 2 ("en kritik ve en yavaş ilerlenecek faz") tamamen bu prefix hesabına ayrılmıştır.

> **🤔 Düşün 1.2** — `10.0.4.130` adresini binary'ye çevir. Sonra: eğer network kısmı ilk **24** bit ise
> bu makine `10.0.4.200` ile aynı ağda mıdır? Peki `10.0.5.130` ile? (a) İkisini de binary karşılaştırmayla
> gerekçelendir. (b) Bu karşılaştırmanın, makinenin hangi kararını belirlediğini yaz.
>
> *(Cevap: fazın sonunda)*

---
---

# 1.3 Public vs Private IP

## 1.3.1 RFC 1918: internette route edilmeyen adresler `[kavram]`

IPv4 adres uzayı 32 bittir — yani yaklaşık **4.3 milyar** adres. Dünyadaki cihaz sayısı düşünüldüğünde bu
korkunç derecede azdır. Bu darlığı hafifleten çözümlerden biri, adres uzayının bir kısmını **"herkes
istediği gibi kullansın, ama sadece kendi içinde"** diye ayırmaktır.

Bu ayrılmış aralıklara **private** (özel) adresler denir ve **RFC 1918** belgesiyle tanımlanmışlardır:

| Aralık | CIDR gösterimi | Kaç adres | Tipik kullanım |
|---|---|---|---|
| `10.0.0.0` – `10.255.255.255` | `10.0.0.0/8` | ~16.7 milyon | Büyük kurumsal ağlar, bulut VPC'leri |
| `172.16.0.0` – `172.31.255.255` | `172.16.0.0/12` | ~1 milyon | Orta ölçekli ağlar, AWS varsayılan VPC |
| `192.168.0.0` – `192.168.255.255` | `192.168.0.0/16` | ~65 bin | Ev ve küçük ofis ağları |

Bunların dışındaki (ve özel amaçlar için ayrılmamış) her adres **public**'tir: internette benzersizdir ve
bir otorite tarafından tahsis edilmiştir.

Bir tanıdık aralık daha: `127.0.0.0/8` — **loopback**. `127.0.0.1` her makinenin "kendisi"dir; bu adrese
giden trafik hiçbir zaman kabloya çıkmaz. Ve `169.254.0.0/16` — **link-local**; 1.6'da DHCP arızasının
imzası olarak geri gelecek.

## 1.3.2 Private IP neden internette route edilemez `[mekanizma]`

Buradaki mekanizma teknik değil, **anlaşmaya dayalıdır** — ve tam da bu yüzden sağlamdır.

İnternet omurgasındaki router'lar, RFC 1918 aralıklarını **kasten tanımaz**. Bir ISP router'ına hedefi
`192.168.1.10` olan bir paket gelirse, onu yönlendirmez — **düşürür**. Sebebi basit: o adres benzersiz
değildir. Şu anda dünyada milyonlarca ev ağında `192.168.1.10` adresli bir cihaz var. Router hangisine
yollayacağını **bilemez**, çünkü adresin kendisi belirsizdir.

Yani private adresler "bloklanmıştır" değil, **anlamsızdır**. Bu bir kısıtlama değil, bir tasarım kararıdır:
sayesinde her ev, her şirket, her VPC aynı adres bloklarını çakışma korkusu olmadan yeniden kullanabilir.

Peki o zaman ev ağındaki telefonun internete nasıl çıkıyor? Cevap **NAT**'tır (Network Address Translation):
çıkışta private kaynak adresi, router'ın public adresiyle **değiştirilir**. Bu, Faz 7'nin tamamıdır; şimdilik
tek cümle yeterli: **private adresler internete çıkarken bir public adresin arkasına gizlenir.**

> **💡 Cloud bağlantısı — VPC'de private IP ve EC2'nin public IP'si:** Bir VPC (Virtual Private Cloud)
> oluştururken ona bir private CIDR bloğu verirsin (örn. `10.0.0.0/16`). İçindeki her EC2, bu bloktan bir
> **private IP** alır ve bu, instance'ın gerçek, kalıcı ağ kimliğidir — `ip addr` çıktısında gördüğün
> adres budur. Instance'a bir **public IP** atanmışsa, o adres instance'ın **içinde görünmez**: AWS onu
> dışarıda, bir NAT katmanında private IP'ye 1:1 eşler. Bu yüzden `ip addr` ile `172.31.x.x` görürken
> internetten `3.120.x.x` ile bağlanırsın — ikisi aynı makinedir. "Public IP'm nerede" sorusunun cevabı
> makinenin içinde değil, **bulut konsolunda** veya instance metadata servisindedir. Bu tek gerçek, bulutta
> en sık kafa karıştıran şeylerden biridir ve Faz 7'de NAT'ı öğrenince tam oturacak.

> **🤔 Düşün 1.3** — Bir mühendis, ofis ağında `192.168.1.50` adresli bir sunucu kurmuş ve arkadaşına
> "evinden şu IP'ye bağlan" demiş. Arkadaşı bağlanamıyor. (a) Neden? (b) Arkadaşının ev router'ındaki
> cihazlardan birinin de `192.168.1.50` olma ihtimali nedir, ve bu neden sorunu daha da netleştirir?
> (c) Doğru çözüm hangi kavramla ilgilidir?
>
> *(Cevap: fazın sonunda)*

---
---

# 1.4 Port ve Socket

## 1.4.1 Port: aynı makinede farklı servisleri ayırmak `[mekanizma]`

IP adresi paketi **makineye** kadar getirir. Peki makinede aynı anda bir web sunucusu, bir SSH sunucusu ve
bir veritabanı çalışıyorsa — paket **hangisine** teslim edilecek?

Cevap **port**'tur. Port, 16 bitlik bir sayıdır (yani **0–65535** arası) ve L4 header'ında (TCP/UDP) taşınır.
Faz 0'daki huni metaforunu hatırla: IP "hangi makine", port "o makinedeki hangi uygulama".

Bina benzetmesi: IP = binanın sokak adresi, port = daire numarası. Postacı binayı adresle bulur, daireyi
numarayla.

Portlar üç gruba ayrılır:

| Aralık | Adı | Kim kullanır |
|---|---|---|
| 0 – 1023 | **Well-known** (iyi bilinen) | Standart servisler; Linux'ta açmak için root yetkisi gerekir |
| 1024 – 49151 | **Registered** (kayıtlı) | Uygulamalara tahsis edilmiş (örn. 3306 MySQL, 5432 PostgreSQL) |
| 49152 – 65535 | **Ephemeral** (geçici) | İstemcinin her bağlantı için rastgele aldığı kaynak portu |

Ezberlemeye değer portlar — bunlar sahada sürekli karşına çıkar (ilk dördü well-known; 3306 ve 5432 *registered* port, ama en az onlar kadar yaygın):

| Port | Servis | Nerede görürsün |
|---|---|---|
| 22 | SSH | Sunucuya bağlanma |
| 53 | DNS | İsim çözümleme (Faz 6) |
| 80 | HTTP | Web, şifresiz (Faz 8) |
| 443 | HTTPS | Web, TLS'li (Faz 8) |
| 3306 | MySQL | Veritabanı |
| 5432 | PostgreSQL | Veritabanı |

Son bir ayrıntı, çok önemli: bir bağlantıda **iki** port vardır. Hedef port sabittir (443 gibi), ama
**kaynak portu** istemci her bağlantı için ephemeral aralıktan rastgele seçer. Bu yüzden aynı anda aynı
siteye on sekme açabilirsin: on ayrı kaynak portu, on ayrı bağlantı.

## 1.4.2 Socket = (IP : Port) `[kavram]`

Bir IP adresi ile bir portu birleştirdiğinde elde ettiğin ikiliye **socket** (soket) denir:
`172.31.20.15:443`. Socket, ağdaki bir **iletişim uç noktasıdır**.

Ama asıl önemli olan şu: bir **bağlantı**, tek bir socket'le değil, **dört değerle** benzersiz hâle gelir:

```
(kaynak IP, kaynak port, hedef IP, hedef port)
```

Buna bağlantının **4-tuple**'ı denir. (Faz 9'da buna protokol de eklenip **5-tuple** olacak — firewall
kurallarının temeli budur.) İşte bu yüzden bin farklı kullanıcı aynı web sunucusunun **aynı** 443 portuna
bağlanabilir: hedef (IP:port) hepsinde aynıdır ama kaynak (IP:port) her birinde farklıdır, dolayısıyla her
bağlantı benzersizdir.

> **🔧 Makinende gör** 🟢 — dinlenen portları ve socket'leri oku
>
> ```
> $ sudo ss -tulpn
> Netid State  Local Address:Port   Peer Address:Port  Process
> tcp   LISTEN 0.0.0.0:22           0.0.0.0:*          users:(("sshd",pid=712,...))
> tcp   LISTEN 127.0.0.1:5432       0.0.0.0:*          users:(("postgres",pid=980,...))
> udp   UNCONN 127.0.0.53:53        0.0.0.0:*          users:(("systemd-resolve",...))
> ```
>
> Her satır bir **socket**'tir. `Local Address:Port` sütununda socket'in iki parçasını görüyorsun.
> `0.0.0.0:22` = "tüm arayüzlerde 22'yi dinle"; `127.0.0.1:5432` = "sadece makinenin içinden erişilebilir".
> Bu ayrım hayati ama bu fazın konusu değil — Faz 9 ve Faz 10'da geri gelecek. `LISTEN` durumu "bu socket
> bağlantı bekliyor" demek. Son sütun, socket'in arkasındaki process'i gösterir: her açık portun arkasında
> **çalışan bir program** vardır.

> **⚠️ Yaygın yanılgı: "Port açmak" firewall'da bir kural yazmaktır.**
>
> Bu ifade iki **tamamen ayrı** şeyi karıştırır ve sahada sürekli kafa karışıklığı yaratır. (1) Bir portun
> **dinleniyor olması**: bir process o portu açmış ve bağlantı bekliyor (`ss -tulpn`'de görünür). (2) Bir
> portun **firewall'dan geçebilmesi**: bir kural o porta gelen trafiğe izin veriyor. Bunlar bağımsızdır:
> firewall'da 8080'i "açabilirsin" ama hiçbir process onu dinlemiyorsa bağlantı yine reddedilir
> (**connection refused**). Ya da process dinliyor olabilir ama firewall kesiyordur (**timeout**). İki
> belirtinin farklı olması tesadüf değil: *refused* "makineye ulaştım, kimse cevap vermedi", *timeout*
> "makineye hiç ulaşamadım" demektir. Bu ayrımı şimdi yerleştir — Faz 9 ve Faz 10'da teşhisin yarısıdır.

> **❓ Akla gelen soru: "'Address already in use' hatası tam olarak ne demek?"**
>
> Bir process bir portu dinlemeye başladığında, o (IP, port) ikilisini **tekel olarak** alır — aynı adres
> ve port çiftini ikinci bir process aynı anda dinleyemez. Uygulamanı başlatırken bu hatayı alıyorsan,
> anlamı nettir: **o portu zaten birisi tutuyor.** Genelde iki sebepten olur: (1) uygulamanın eski bir
> kopyası hâlâ çalışıyordur (`sudo ss -tulpn | grep :8080` ile PID'sini bul, Faz 0'daki "her socket'in
> arkasında bir process vardır" kuralı), veya (2) bir önceki bağlantı `TIME_WAIT` durumundadır (Faz 5.6 —
> `[atla]` düzeyinde, ama adını duymuş ol). Not: farklı **IP**'lerde aynı port sorun değildir —
> `127.0.0.1:8080` ile `10.0.1.5:8080` iki ayrı socket'tir, çakışmaz.

> **🤔 Düşün 1.4** — Bir web sunucusu `0.0.0.0:443`'ü dinliyor ve aynı anda 5.000 kullanıcı bağlı.
> (a) Sunucu tarafında kaç farklı **hedef** socket var? (b) 5.000 bağlantı birbirinden nasıl ayırt ediliyor —
> hangi değerler farklı? (c) Bu, "port tükenmesi" (port exhaustion) sorununun neden **istemci** tarafında
> ortaya çıktığını nasıl açıklar?
>
> *(Cevap: fazın sonunda)*

---
---

# 1.5 IPv6 — Farkındalık

## 1.5.1 Neden var: IPv4 adres tükenmesi `[kavram]`

IPv4'ün 32 biti ~4.3 milyar adres verir. 1980'lerde bu sonsuz görünüyordu; 2010'larda resmen tükendi —
adres tahsis eden bölgesel otoritelerin elinde yeni blok kalmadı. NAT (Faz 7) bu çöküşü yıllarca erteledi
ama kökten çözmedi.

**IPv6** kökten çözer: adres uzunluğu **128 bit**tir. Bu, yaklaşık 3.4×10³⁸ adres demektir — dünyadaki her
kum tanesine milyarlarca adres düşecek kadar. Adresler hex yazılır ve iki nokta üst üste ile ayrılır:

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
2001:db8:85a3::8a2e:370:7334        ← kısaltılmış hâli (sıfırlar atlanır)
```

Bu faz için bilmen gereken bu kadar — derinlik etiketi `[atla]`. Üç pratik not yeterli: (1) IPv6'da adres
bolluğu olduğu için **NAT'a çoğunlukla gerek yoktur**, her cihaz public adres alabilir (Faz 7.5). (2)
Geçiş dönemi modeli **dual-stack**'tir: bir makine aynı anda hem IPv4 hem IPv6 konuşur. (3) Dual-stack'te
en sık arıza, uygulamanın **yanlış aileyi** seçmesidir — "bir protokolde çalışıyor, diğerinde takılıyor".

---
---

# 1.6 DHCP — IP'yi Otomatik Almak

## 1.6.1 Statik vs dinamik: adres nereden geliyor `[kavram]`

Buraya kadar adreslerin **ne olduğunu** konuştuk. Peki bir makine adresini **nereden alıyor**?

İki yol var. **Statik**: elle yazarsın (Ubuntu'da netplan dosyasına). Sunucular için kullanılır — adresin
değişmemesi gerektiğinde. **Dinamik**: ağdaki bir sunucudan otomatik alırsın. Bunu yapan protokole **DHCP**
(Dynamic Host Configuration Protocol — *di-eyç-si-pi*) denir.

DHCP, ağa takılan her cihazın elle yapılandırılmasını gereksiz kılar: bir kafeye girip Wi-Fi'a bağlandığında
telefonunun saniyeler içinde internete çıkabilmesinin sebebi budur.

Ve burada kritik bir nokta var, çoğu kişi bunu bilmez: **DHCP sadece IP dağıtmaz.** Bir makinenin ağda
çalışabilmesi için gereken dört bilginin **hepsini** verir:

1. **IP adresi** — makinenin kimliği (1.2)
2. **Subnet mask** — "aynı ağ" sınırının nerede olduğu (Faz 2)
3. **Default gateway** — yerel ağda olmayan trafiğin gideceği adres (Faz 4)
4. **DNS sunucusu** — isimleri IP'ye çevirecek adres (Faz 6)

Bu dördü, bu kitabın dört ayrı fazına karşılık gelir. DHCP, o dört fazın girdilerini tek hamlede teslim
eder — bu yüzden DHCP bozulunca belirti "ağ tamamen yok" gibi görünür.

## 1.6.2 DORA döngüsü: dört adımda adres almak `[mekanizma]`

DHCP'nin adres tahsisi dört mesajdan oluşur ve baş harfleriyle **DORA** diye anılır:

1. **Discover** (keşfet) — İstemcinin henüz IP'si yoktur. Yerel ağa bir **broadcast** yollar: "burada DHCP
   sunucusu var mı?" Kaynak IP `0.0.0.0` (henüz adresi yok), hedef `255.255.255.255` (herkese). Bu adımın
   broadcast olması **zorunludur** — istemci sunucunun adresini bilmiyor.
2. **Offer** (teklif et) — DHCP sunucusu cevap verir: "sana `10.0.1.47` adresini önerebilirim; mask şu,
   gateway şu, DNS şu, süre (lease) 24 saat."
3. **Request** (talep et) — İstemci teklifi kabul eder ve yine broadcast ile duyurur: "`10.0.1.47`'yi
   istiyorum." Broadcast olmasının sebebi, ağda birden fazla DHCP sunucusu varsa diğerlerinin de tekliflerini
   geri çekebilmesidir.
4. **Ack** (onayla) — Sunucu onaylar: "tamam, senin. Lease süresi başladı." İstemci artık adresini
   yapılandırır ve ağa katılır.

Adres **ödünç verilir**, sonsuza kadar değil: buna **lease** (kiralama) denir. Süre dolmadan istemci
yenilemek için tekrar Request yollar. Bu yüzden bir makine uzun süre kapalı kalırsa geri döndüğünde farklı
bir IP alabilir.

![Şekil 1.1 — DHCP DORA döngüsü: istemci ve DHCP sunucusu arasındaki dört mesaj (Discover → Offer → Request → Ack) ve her adımın broadcast mi unicast mi olduğu. Ack ile birlikte istemci sadece IP değil, subnet mask, default gateway ve DNS sunucusunu da almış olur.](../diagrams/png/nw-1-01-dhcp-dora.png)

Şekilde soldaki dikey çizgi istemci, sağdaki DHCP sunucusudur; zaman yukarıdan aşağı akar. İlk mesajın
kaynak adresinin `0.0.0.0` olduğuna dikkat et — istemcinin henüz kimliği yoktur. Sağ alttaki kutu, Ack ile
birlikte teslim edilen dört bilgiyi listeler: bunların her biri bu kitabın farklı bir fazının konusudur.

> **💡 Cloud bağlantısı — VPC'de DHCP option sets:** Bir EC2 instance'ı başlattığında IP'sini elle
> yazmazsın; AWS'in VPC içindeki DHCP servisi ona private IP'sini, mask'ini, gateway'ini ve DNS sunucusunu
> verir. `ip addr` çıktısındaki `dynamic` kelimesi tam olarak bunun izidir. Bu dağıtımın içeriğini **DHCP
> option set** ile yapılandırabilirsin — en yaygın kullanımı, VPC'nin varsayılan resolver'ı yerine kendi
> DNS sunucularını (örn. şirket içi bir Active Directory DNS'i) dağıtmaktır. Buradaki pratik uyarı: bir DHCP
> option set'i yanlış yapılandırmak, **instance'ların DNS çözümlemesini topluca bozar** — ve belirti
> "internet yok" gibi görünür, oysa IP ve routing gayet sağlamdır (Faz 6'da bu ayrımı tam oturtacağız).

## 1.6.3 DHCP bozulunca: 169.254.x.x ve "hiç IP yok" `[mekanizma]`

DHCP başarısız olursa — sunucu yoksa, ulaşılamıyorsa, havuzunda adres kalmadıysa — istemcinin adresi
**olmaz**. Bu durumda işletim sistemine göre iki farklı imza görürsün:

- **Windows:** Kendine `169.254.x.x` aralığından bir adres verir. Buna **APIPA** (Automatic Private IP
  Addressing) veya **link-local** denir. Bu adresle sadece aynı segmentteki diğer link-local cihazlara
  ulaşabilirsin; gateway yoktur, DNS yoktur, internet yoktur.
- **Ubuntu (NetworkManager/systemd-networkd):** Genellikle hiç IPv4 adresi almaz. `ip addr` çıktısında o
  arayüzde `inet` satırı **hiç görünmez**.

İkisi de aynı şeyi söyler: **"DHCP cevap vermedi."** Ve teşhis açısından bu çok değerli bir imzadır, çünkü
`169.254.x.x` gören bir mühendis aramayı anında daraltır — sorun uygulamada, DNS'te veya routing'de değil,
**adres tahsisindedir**.

> **🔧 Makinende gör** 🟢 — adresin DHCP'den mi geldiğini anla
>
> ```
> $ ip -4 addr show ens5
> 2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
>     inet 172.31.20.15/20 brd 172.31.31.255 scope global dynamic ens5
>        valid_lft 2591998sec preferred_lft 2591998sec
> ```
>
> İki ipucu: `dynamic` kelimesi adresin DHCP ile alındığını söyler (statik olsaydı bu kelime olmazdı).
> `valid_lft` (valid lifetime) ise **lease süresidir** — bu örnekte ~30 gün kalmış. Süre saniye saniye azalır
> ve yenilenir. Eğer bu satırda `inet` hiç görmüyorsan veya adres `169.254.x.x` ise, teşhisin hazır:
> DHCP cevap vermemiş.

> **🤔 Düşün 1.5** — Bir kullanıcı "internet çalışmıyor" diyor. `ip addr` çıktısında arayüzünün IP'si
> `169.254.8.31`. (a) Bu adres sana ne söylüyor? (b) Bu makineden `ping 8.8.8.8` denesen ne beklersin,
> ve neden? (c) Aynı ağdaki başka bir makine sorunsuz çalışıyorsa, şüphenin yönü nasıl değişir?
>
> *(Cevap: fazın sonunda)*

---
---

# 1.7 Bu Faz Bozulunca — Adresleme Arıza İmzaları

Adresleme arızalarının ortak özelliği şudur: belirti neredeyse her zaman "internet yok" gibi görünür ama
**kök sebep üç farklı kademenin birinde** yatar — L2 kimlik (MAC), L3 kimlik (IP), veya L4 kimlik (port).
Aşağıdaki tablo belirtiyi doğru kademeye götürür.

| Belirti | Muhtemel sebep | Bakılacak yer | İlgili bölüm |
|---|---|---|---|
| Arayüzde hiç `inet` satırı yok | DHCP cevap vermedi / arayüz down | `ip addr`, `ip link` (UP mı) | 1.6.3 |
| IP `169.254.x.x` (APIPA) | DHCP sunucusuna ulaşılamıyor | DHCP sunucusu, L2 bağlantı | 1.6.3 |
| IP var ama internete çıkılmıyor | DHCP mask/gateway'i yanlış vermiş | `ip route` (`default via` var mı) | 1.6.1 (Faz 4) |
| IP var, internet var, isimler çözülmüyor | DHCP DNS sunucusunu yanlış verdi | `/etc/resolv.conf` | 1.6.1 (Faz 6) |
| Aynı ağda iki cihaz kopuyor | IP çakışması (aynı adres iki makinede) | `ip neigh`, DHCP lease kayıtları | 1.2.3 |
| Uzak private IP'ye erişilemiyor | Private adres internette route edilmez | Adres RFC 1918 aralığında mı | 1.3.2 |
| "Address already in use" | Portu başka bir process tutuyor | `ss -tulpn \| grep :<port>` | 1.4.2 |
| "Connection refused" | Hiçbir process o portu dinlemiyor | `ss -tulpn` (LISTEN var mı) | 1.4.1 |
| Bulutta `ip addr` public IP göstermiyor | Public IP dışarıda NAT ile eşlenir | Bulut konsolu / metadata | 1.3.1 |
| Makine her reboot'ta farklı IP alıyor | DHCP lease yenilenmemiş / rezervasyon yok | DHCP lease süresi, statik/rezerve IP | 1.6.2 |

> **Bu tablodan çıkan ders:** "Adresim yok" ile "adresim var ama yanlış" **tamamen farklı** iki arızadır ve
> ilk ayırman gereken budur. `ip addr` çıktısında `inet` satırı **hiç yoksa** arıza adres tahsisindedir
> (DHCP/L2) — daha yukarı bakmanın anlamı yok. `inet` **varsa**, adres tahsisi çalışmış demektir ve şüphe
> otomatik olarak bir üst kademeye kayar: mask mı yanlış (Faz 2), gateway mi eksik (Faz 4), DNS mi bozuk
> (Faz 6)? Üç kademeli huniyi (MAC → IP → port) daima sırayla ele: **önce kimlik var mı, sonra doğru mu,
> en son hedeflenen uygulamaya ulaşıyor mu.** Ve unutma: `169.254.x.x` bir adres değil, bir **arıza
> raporudur**.

---
---

# Faz 1 — Düşün sorularının cevapları

## Cevap 1.1 — Hedef IP uzak sunucu, hedef MAC ise kendi router'ı

(a) Hedef IP **`93.184.216.34`**'tür — yani uzak sunucunun kendisi. L3 adresi uçtan uca sabittir ve nihai
hedefi gösterir (Faz 0.2.2).

(b) Hedef MAC, uzak sunucunun MAC'i **değildir** — olamaz da, çünkü A o MAC'i asla öğrenemez (uzak sunucu
başka bir yerel ağdadır ve MAC'ler route edilmez, 1.1.2). A'nın yazdığı hedef MAC, **kendi yerel ağındaki
varsayılan geçidin (router'ın) MAC'idir**. Yani frame fiziksel olarak router'a gider; router zarfı açar,
IP header'ındaki gerçek hedefi görür, ve paketi bir sonraki hop'a yeni bir L2 zarfıyla yollar.

(c) Bu tam olarak "MAC her hop'ta değişir" cümlesinin somut hâlidir: A→router adımında bir MAC çifti, →bir
sonraki hop'ta bambaşka bir MAC çifti kullanılır. IP ise yolculuk boyunca hiç değişmez. Bir sonraki soru
doğal olarak şu olur: *A, router'ın MAC'ini nereden bildi?* Cevabı **ARP**'dır ve Faz 3.1'in konusudur.
**İlgili bölüm:** 1.1.2 (Faz 0.2.2) · **Devamı:** Faz 3.1 (ARP), Faz 4.1 (gateway).

## Cevap 1.2 — Binary karşılaştırma, "aynı ağ mı" kararını verir

`10.0.4.130` binary olarak: `00001010.00000000.00000100.10000010`
(10 = `00001010`, 0 = `00000000`, 4 = `00000100`, 130 = `10000010`).

(a) **`10.0.4.200` ile:** ilk 24 bit `00001010.00000000.00000100` — ikisinde de **aynı** (ilk üç oktet
`10.0.4`). Dolayısıyla **aynı ağdalar**. Sadece host kısmı (son oktet: 130 vs 200) farklı.
**`10.0.5.130` ile:** ilk 24 bitin son oktet'i farklı — `00000100` (4) vs `00000101` (5). Üçüncü oktet
network kısmının içinde olduğu için bu ikisi **farklı ağlardadır**.

(b) Bu karşılaştırma, makinenin en temel kararını belirler: **"hedef benim ağımda mı?"** Aynı ağdaysa paket
doğrudan komşuya teslim edilir (L2, Faz 3); farklı ağdaysa **varsayılan geçide** yollanır (Faz 4). Yani
`10.0.4.130` makinesi `10.0.4.200`'e doğrudan ulaşmaya çalışır, `10.0.5.130`'a ise router üzerinden. Prefix
yanlış yapılandırılmışsa bu karar da yanlış verilir — bu, Faz 2'deki en pahalı hata sınıfıdır.
**İlgili bölüm:** 1.2.2-1.2.3 · **Devamı:** Faz 2.1 (subnet mask), Faz 4.1 (gateway kararı).

## Cevap 1.3 — Private adres internette anlamsızdır

(a) `192.168.1.50` bir **RFC 1918 private** adresidir (1.3.1). İnternet router'ları bu aralığı kasten
yönlendirmez — paketi düşürürler. Arkadaşının paketi ofis ağına **hiç ulaşmaz**; internet omurgasında ölür.

(b) İhtimal çok yüksek, çünkü `192.168.1.0/24` ev router'larının en yaygın varsayılan bloğudur. Ve bu,
sorunu daha da netleştirir: eğer arkadaşı o adrese bağlanmaya çalışırsa, makinesi "bu adres **benim** yerel
ağımda" diye karar verir (1.2.3) ve paketi hiç dışarı yollamaz — kendi ağındaki `192.168.1.50`'ye gider
(varsa bambaşka bir cihaza, yoksa hiçbir yere). Yani adres "ulaşılamaz" bile değil, **yanlış yere
ulaşabilir**. Private adreslerin benzersiz olmamasının pratik sonucu tam olarak budur.

(c) Doğru çözüm **NAT + port forwarding** ile ilgilidir (Faz 7.2): ofis router'ının **public** IP'sine
bağlanılır, router da gelen trafiği içerideki `192.168.1.50`'ye yönlendirir. Alternatif olarak bir **VPN**
(Faz 7.4) kurulur ve arkadaşı mantıksal olarak ofis ağının içine alınır.
**İlgili bölüm:** 1.3.1-1.3.2 · **Devamı:** Faz 7.1-7.2 (NAT ve port forwarding).

## Cevap 1.4 — Tek hedef socket, 5.000 farklı 4-tuple

(a) Sunucu tarafında **tek bir** hedef socket vardır: `<sunucu IP>:443`. 5.000 bağlantı için 5.000 ayrı
dinleme socket'i açılmaz — bir tane dinler, gelen bağlantıları kabul eder.

(b) Bağlantılar **4-tuple** ile ayırt edilir: (kaynak IP, kaynak port, hedef IP, hedef port). Hedef IP ve
hedef port 5.000 bağlantıda da **aynıdır** (sunucu:443). Farklı olan **kaynak** taraftır: ya farklı kaynak
IP'ler (farklı kullanıcılar), ya aynı kullanıcının farklı **ephemeral kaynak portları**. Bu dördü birlikte
her bağlantıyı benzersiz kılar (1.4.2).

(c) Çünkü darboğaz kaynak tarafındadır: ephemeral port aralığı sınırlıdır (~49152–65535, yani ~16.000 port).
**Tek bir istemci IP'si**, tek bir hedef socket'e doğru en fazla o kadar eşzamanlı bağlantı açabilir —
sonra 4-tuple'lar tükenir. Sunucu tarafında böyle bir sınır yoktur, çünkü sunucu tek port kullanır ve
çeşitlilik kaynak tarafından gelir. Bu yüzden port tükenmesi tipik olarak **NAT arkasındaki bir çıkış
noktasında** (Faz 7.2) veya yoğun bir proxy/istemci makinede görülür.
**İlgili bölüm:** 1.4.1-1.4.2 · **Devamı:** Faz 7.2 (PAT), Faz 9.2 (5-tuple).

## Cevap 1.5 — 169.254 bir adres değil, bir arıza raporudur

(a) `169.254.8.31` bir **link-local / APIPA** adresidir (1.3.1, 1.6.3). Anlamı nettir: makine DHCP'den
adres **alamamıştır** ve kendine geçici bir adres uydurmuştur. Yani arıza, adres tahsisi kademesindedir —
DNS, routing veya uygulama katmanında değil.

(b) `ping 8.8.8.8` **başarısız** olur ("Network is unreachable" veya benzeri). Sebep: link-local adresle
birlikte **varsayılan geçit gelmez** (1.6.1 — gateway'i de DHCP dağıtır). Makine, yerel ağı olmayan bir
hedefe paketi nereye yollayacağını bilmez; `ip route` çıktısında `default via ...` satırı **yoktur**
(Faz 4.1). Paket makineden hiç çıkmaz.

(c) Şüphe **makinenin kendisine** kayar. Aynı ağdaki diğer makineler DHCP'den sorunsuz adres alabiliyorsa,
DHCP sunucusu ayakta ve ulaşılabilir demektir. O hâlde sorun bu makineye özgüdür: kablo/port arızası
(L2 — `ip link`'te `LOWER_UP` var mı?), arızalı NIC, yanlış VLAN'a düşmüş bir switch portu, veya
istemcinin DHCP servisinin çalışmaması. Teşhis sırası yine aşağıdan yukarıdır (Faz 0.1.3): önce **link var
mı**, sonra adres alınabiliyor mu.
**İlgili bölüm:** 1.6.3 · **Devamı:** Faz 4.1 (default gateway), Faz 10.1 (katman-katman teşhis).

---
---

# Faz 1 — Sık sorulan sorular

**S1 — MAC adresi gerçekten değiştirilemez mi?** Donanıma gömülüdür ama yazılımla **geçersiz kılınabilir**
(MAC spoofing; Linux'ta `ip link set dev ens5 address ...`). Pratikte bu, bazı ağ erişim kontrollerini aşmak
veya bir yedek cihazı aynı kimlikle devreye almak için kullanılır. Bulutta ise genelde **yasaklıdır**:
hypervisor, ENI'ye atanan MAC dışında bir kaynak MAC'e izin vermez (1.1.1).

**S2 — `192.168.1.300` neden geçersiz bir adres?** Çünkü her oktet **8 bittir** ve 8 bitin alabileceği en
büyük değer 255'tir. 300 bir oktete sığmaz. Bu, adresin "uydurma" olduğunu anlamanın en hızlı yoludur —
bir oktette 255'ten büyük bir sayı görürsen adres geçersizdir (1.2.1).

**S3 — Private IP kullanmak güvenlik sağlar mı?** Kısmen ve **yanıltıcı** biçimde. Private adresler
internetten doğrudan route edilemez, bu doğal bir engeldir. Ama bu bir **firewall değildir**: NAT/port
forwarding ile o servis dışarı açılabilir, VPN ile ağa girilebilir, ve ağın içine bir kez giren bir saldırgan
için private adres hiçbir koruma sağlamaz. Güvenlik Faz 9'un işidir; private adres bir güvenlik özelliği
değil, bir **adresleme** özelliğidir (1.3.2).

**S4 — Kaynak portu neden rastgele?** Çünkü bir bağlantının benzersizliği 4-tuple'a dayanır (1.4.2). Aynı
istemci aynı sunucunun aynı portuna birden çok bağlantı açacaksa, tek ayırt edici değer **kaynak portudur**.
Rastgele (ephemeral) seçim, bu çeşitliliği sağlar. Ayrıca tahmin edilebilir kaynak portları bazı saldırıları
kolaylaştırdığı için rastgelelik bir güvenlik katkısı da sağlar.

**S5 — "Connection refused" ile "timeout" arasındaki fark nedir?** Kritik bir ayrım. **Refused**: paket
makineye **ulaştı**, ama o portu dinleyen bir process yok — makine aktif olarak "burada kimse yok" cevabı
(TCP RST) yolladı. **Timeout**: paket hedefe hiç ulaşmadı veya cevap dönmedi — genelde bir firewall sessizce
düşürmüştür, ya da routing/adres yanlıştır. Kabaca: *refused* = L4 sorunu (servis yok), *timeout* = L3/
firewall sorunu (yol yok) (1.4.1).

**S6 — DHCP olmadan da ağ çalışır mı?** Evet — tüm ayarları elle (statik) yazarsan. Ama dört şeyi birden
doğru yazman gerekir: IP, mask, gateway, DNS (1.6.1). Sunucularda ve ağ cihazlarında statik yapılandırma
yaygındır (adresin değişmemesi gerekir); istemcilerde neredeyse her zaman DHCP kullanılır. Bulutta ise
instance'a elle IP yazmak genelde **yanlıştır** — adres tahsisi bulut tarafından yönetilir.

**S7 — IPv6'yı şimdi öğrenmem gerekir mi?** Hayır, bu fazda `[atla]` düzeyindedir. Bilmen gerekenler: neden
var olduğu (IPv4 tükenmesi), 128 bit olduğu, ve dual-stack'te "bir protokolde çalışıyor, diğerinde
takılıyor" arızasının var olduğu. Bulut tarafındaki karşılıklarını (dual-stack VPC, egress-only IGW) Faz 7.5
ve Faz 11'de göreceksin (1.5.1).

---
---

# Faz 1 — Kendini sına

Cevaplarını bir kâğıda yaz, sonra cevap anahtarıyla karşılaştır. Hedef: 18 sorudan 14+.

## Bölüm A — Tanım ve mekanizma

1. MAC adresi kaç bittir, nasıl yazılır, ve hangi katmanın kimliğidir?
2. MAC neden internet ölçeğinde route edilemez? Tek kelimelik sebebi ve bir cümlelik açıklamasını yaz.
3. Bir IPv4 adresi kaç bittir, kaç oktetten oluşur, bir oktetin alabileceği değer aralığı nedir?
4. `11000000.10101000.00001010.00000001` adresini dotted-decimal yaz.
5. Bir IP adresindeki "network kısmı" ile "host kısmı" ne demektir? Hangisi aynı ağdaki tüm makinelerde
   aynıdır?
6. RFC 1918'in üç private aralığını CIDR gösterimiyle yaz.
7. Port kaç bittir, aralığı nedir, ve üç port grubunun adını + kullanım amacını yaz.
8. Socket nedir? Bir **bağlantıyı** benzersiz kılan dört değer hangileridir?

## Bölüm B — Uygula ve teşhis et

9. `172` sayısını elle binary'ye çevir, adımlarını göster.
10. `ip addr` çıktısında `inet 10.0.1.25/24 ... dynamic` satırını görüyorsun. Bu satırdan çıkarabileceğin
    **üç** ayrı bilgiyi yaz.
11. Bir makinenin IP'si `169.254.3.44`. Teşhisin nedir, ve doğrulamak için bakacağın ilk iki şey nedir?
12. Uygulamanı başlatırken "address already in use" hatası alıyorsun. Hangi komutla kök sebebi bulursun,
    ve çıktıda neye bakarsın?
13. `ping <ip>` çalışıyor ama `curl http://<ip>` "connection refused" diyor. Hangi adres kademesi sağlam,
    hangisi sorunlu?
14. DHCP'nin dağıttığı dört bilgiyi say ve her birinin bu kitabın hangi fazına karşılık geldiğini yaz.

## Bölüm C — Muhakeme ve bağlantı

15. Bir makine uzak bir sunucuya paket yollarken hedef IP ve hedef MAC alanlarına ne yazar? İkisi arasındaki
    farkı Faz 0'daki "her hop'ta değişir" kuralına bağla.
16. DORA döngüsünün ilk mesajı neden **broadcast** olmak zorundadır?
17. Bir kullanıcı IP alabiliyor, `ping 8.8.8.8` çalışıyor, ama `ping google.com` çalışmıyor. DHCP'nin
    dağıttığı hangi bilgi yanlış olabilir, ve hangi bilgiler doğru çalışıyor demektir?
18. "Connection refused" ile "timeout" farkını, ağın hangi katmanlarına işaret ettiklerini de belirterek
    açıkla.

---

## Cevap anahtarı

1. **48 bit**, altı hex çiftiyle (`3c:52:82:1a:0b:77`), **L2** (data link) kimliğidir (1.1.1). — 2. Sebep:
   **hiyerarşi yok**. MAC düzdür; adresten cihazın nerede olduğu çıkarılamaz, dolayısıyla gruplanamaz ve
   router tablosuna sığmaz (1.1.2). — 3. **32 bit**, **4 oktet**, her oktet **0–255** (1.2.1). —
   4. `192.168.10.1` (1.2.2). — 5. Network kısmı "hangi ağ"dır ve aynı ağdaki **tüm** makinelerde aynıdır;
   host kısmı "o ağdaki hangi makine"dir ve her makinede farklıdır (1.2.3). — 6. `10.0.0.0/8`,
   `172.16.0.0/12`, `192.168.0.0/16` (1.3.1). — 7. **16 bit**, 0–65535. Well-known (0–1023, standart
   servisler, root gerekir), registered (1024–49151, tahsisli uygulamalar), ephemeral (49152–65535, istemci
   kaynak portu) (1.4.1). — 8. Socket = **(IP : port)** ikilisi. Bağlantıyı benzersiz kılan: **kaynak IP,
   kaynak port, hedef IP, hedef port** (4-tuple) (1.4.2).

9. 128? evet (1), kalan 44 → 64? hayır (0) → 32? evet (1), kalan 12 → 16? hayır (0) → 8? evet (1), kalan 4
   → 4? evet (1), kalan 0 → 2? hayır (0) → 1? hayır (0). Sonuç: **`10101100`** (1.2.2). — 10. (i) Makinenin
   IP'si `10.0.1.25`; (ii) network kısmı ilk **24 bit** (`/24`), yani `10.0.1.x` aynı ağ; (iii) adres
   **DHCP** ile alınmış (`dynamic`) (1.2.3, 1.6.3). — 11. Teşhis: **DHCP cevap vermedi** (APIPA/link-local
   adres). Bakılacaklar: (a) `ip link` ile arayüz gerçekten UP/LOWER_UP mı (L2 bağlantı var mı), (b) aynı
   ağdaki başka bir makine DHCP'den adres alabiliyor mu (sunucu mu arızalı, makine mi) (1.6.3). —
   12. `sudo ss -tulpn | grep :<port>` — çıktıda o portu `LISTEN` durumunda tutan process'in adına ve
   **PID**'sine bakarsın (1.4.2). — 13. **L3 sağlam** (IP doğru, makineye ulaşılıyor — ping cevap veriyor);
   sorun **L4'te**: o portu dinleyen bir process yok, makine RST ile reddediyor (1.4.1). — 14. IP adresi
   (Faz 1), subnet mask (Faz 2), default gateway (Faz 4), DNS sunucusu (Faz 6) (1.6.1).

15. Hedef IP alanına **nihai hedefin IP'si** yazılır ve yolculuk boyunca değişmez. Hedef MAC alanına ise
    **bir sonraki hop'un** (uzak hedefse: varsayılan geçidin) MAC'i yazılır ve **her hop'ta yenilenir**.
    IP = uçtan uca kimlik, MAC = yerel teslimat etiketi (1.1.2, Faz 0.2.2). — 16. Çünkü istemcinin o anda
    ne kendi IP'si ne de DHCP sunucusunun adresi vardır; kime soracağını bilmediği için **herkese** sormak
    zorundadır (kaynak `0.0.0.0`, hedef `255.255.255.255`) (1.6.2). — 17. Yanlış olan **DNS sunucusu**
    bilgisidir. Doğru çalışanlar: IP adresi (makinenin kimliği var), subnet mask ve default gateway (IP ile
    internete çıkabiliyor). Yani adres tahsisi ve routing sağlam, **isim çözümleme** bozuk (1.6.1, Faz 6).
    — 18. **Refused:** paket makineye ulaştı ama o portu dinleyen process yok → makine RST döndürdü; bu bir
    **L4** sorunudur (servis yok/ölü). **Timeout:** cevap hiç dönmedi → paket yolda düşürüldü veya hedefe
    ulaşmadı; bu bir **L3/firewall** sorunudur (yol yok veya sessizce engelleniyor) (1.4.1, S5).

## Puanlama

| Doğru sayısı | Ne anlama geliyor |
|---|---|
| 16-18 | Üç adres kademesini (MAC/IP/port) ayırt ediyorsun. Faz 2'ye hazırsın. |
| 13-15 | İyi. Kaçırdığın bölümleri tekrar oku — özellikle 1.2.3 (network/host ayrımı). |
| 9-12 | Temel var ama kırılgan. **1.2.2 binary dönüşümünü elle çalış** — Faz 2 bunsuz yürümez. |
| 0-8 | Fazı yeniden gez. Hedef: `ip addr` çıktısının her alanını ezbersiz açıklayabilmek. |

Kaçırdığın soru → dönmen gereken bölüm:

| Soru | Bölüm |
|---|---|
| 1, 2, 15 | 1.1 MAC adresi |
| 3, 4, 9 | 1.2.1-1.2.2 IPv4 ve binary |
| 5, 10 | 1.2.3 Network/host ayrımı |
| 6, 13 | 1.3 Public vs private |
| 7, 8, 12 | 1.4 Port ve socket |
| 11, 14, 16, 17 | 1.6 DHCP |
| 18 | 1.4.1 + 1.7 arıza tablosu |

---
---

# Faz 1 — Kapanış ve Faz 2'ye Köprü

## Bu fazdan ne taşıyorsun

Faz 1 sana **üç kademeli adres huniyi** verdi: **MAC** "bu yerel ağdaki hangi cihaz" (L2, düz, her hop'ta
değişir), **IP** "internetteki hangi makine" (L3, hiyerarşik, uçtan uca sabit), **port** "o makinedeki
hangi uygulama" (L4). Bir IPv4 adresinin 32 bitlik yapısını ve network/host ayrımını öğrendin; private ile
public adresin neden farklı dünyalar olduğunu gördün; socket'in ve 4-tuple'ın bağlantıları nasıl benzersiz
kıldığını kavradın. Ve bir makinenin adresini nereden aldığını (DHCP/DORA), o mekanizmanın sadece IP değil
**mask + gateway + DNS** de dağıttığını, bozulunca `169.254.x.x` imzasını bıraktığını öğrendin.

En kalıcı cümle şu: **"Adresim yok" ile "adresim var ama yanlış" farklı arızalardır** — ve `ip addr`
çıktısındaki tek bir `inet` satırının varlığı/yokluğu bu ayrımı anında yapar.

## Faz 2 bunun neresine bağlanıyor

Faz 1'de bir adresin "network kısmı" ve "host kısmı" olduğunu söyledik (1.2.3) — ama o sınırın **tam olarak
nerede** çizildiğini hep erteledik. `/24` ve `/20` yazdık, "Faz 2'de açacağız" dedik.

Faz 2 tam olarak budur: **subnet mask ve CIDR.** O sınırın nasıl çizildiğini, nasıl kaydırıldığını, ve
kaydırdığında kaç makinenin "aynı ağda" sayıldığını hesaplamayı öğreneceksin. Bu, kitabın **en kritik ve en
yavaş ilerlenecek** fazıdır, çünkü bulut ağ tasarımının tamamı — VPC CIDR'ı, subnet bölme, route table,
security group kapsamları, peering çakışmaları — tek bir beceriye dayanır: **bir prefix'i okuyup ne anlama
geldiğini söyleyebilmek.**

Faz 1 adresleri tanıttı; Faz 2 onları **bölmeyi** öğretiyor.

> **🤔 Faz çıktısı — kendine sor:** 1.2.3'te "makine, hedefin kendi ağında olup olmadığına network kısmını
> karşılaştırarak karar verir" dedik. Faz 2'ye geçmeden düşün: eğer iki makineye **aynı** IP aralığı ama
> **farklı** prefix (biri `/24`, diğeri `/16`) verilirse ne olur? A, B'yi "aynı ağda" sanarken B, A'yı
> "uzakta" sayabilir mi — ve böyle bir asimetri trafiği nasıl bozar? (İpucu: gidiş yolu ile dönüş yolu
> farklı kararlar alabilir.)
>
> **🧪 Lab 1 fikri (kendi makinende, hepsi 🟢):** (1) `ip link` ile MAC adreslerini, `ip -4 addr` ile
> IP'lerini ve prefix'lerini listele — L2 ve L3 kimliğini yan yana gör. (2) Kendi IP'ni kâğıtta binary'ye
> çevir; prefix'in gösterdiği yerde bir dikey çizgi çekip network ve host kısımlarını işaretle. Bu kâğıdı
> sakla, Faz 2'de kullanacağız. (3) `ip -4 addr` çıktısında `dynamic` kelimesini ve `valid_lft` lease
> süresini bul — adresinin DHCP'den geldiğini kendi gözünle doğrula. (4) `sudo ss -tulpn` ile dinlenen
> socket'leri listele; her satırdaki (IP:port) ikilisini ve arkasındaki process'i eşleştir. (5) `curl
> ifconfig.me` çalıştırıp dönen public IP ile `ip addr`'daki private IP'yi karşılaştır — ikisinin farklı
> olması Faz 7'nin (NAT) tohumudur. Bu beş adım, üç adres kademesini elinde somutlaştırır.

---

> **Navigasyon:** [◀ Faz 0 — Zihinsel Model](Faz_0_Zihinsel_Model.md) · **Faz 1** · [Faz 2 — Subnet ve CIDR ▶](Faz_2_Subnet_ve_CIDR.md)
