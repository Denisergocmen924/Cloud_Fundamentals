# Ağ — Offline Çalışma Kitabı

> **Cloud Engineer Temel Yol Haritası — Ağ / Network**
> Bu, [`Cloud_network_roadmap_temel_tr.md`](../../../Cloud%20Learning%20With%20Mentor/Turkish/Cloud_network_roadmap_temel_tr.md) yol
> haritasının **tek başına, YZ mentoru olmadan, internetsiz** baştan sona işlenebilir hâlidir.

> 🇬🇧 English version: **[README_en.md](../English_Network/README_en.md)**

---

## Bu kitap nedir, ana haritadan farkı ne?

Mentor klasöründeki yol haritası dosyası bir **iskelettir**: bir YZ mentoruna verilir, mentor
konuları sırayla açar. İskelet tek başına okunduğunda "ne öğrenmem gerektiğini" söyler ama
"onu öğretmez".

Bu kitap o iskeletin **eti**dir. Aynı fazlar, aynı alt maddeler, aynı derinlik etiketleri —
ama her madde burada:

- **anlatılır** (sezgi → mekanizma → komut çıktısı),
- **sorulur** (senin düşünmen için bırakılan sorular),
- **cevaplanır** (aynı dosyanın ilerleyen bölümünde),
- **bağlanır** (önceki maddeye ve AWS'teki karşılığına).

Hiçbir yerde "bunu mentoruna sor" demez. Dosyayı bir uçakta, internetsiz, tek başına
baştan sona işleyebilirsin.

---

## Bu kitabın diğer iki kitaptan farkı — önemli

Donanım kitabında `[uygulama]` "bir mimari kararla ilişkilendir", Linux kitabında "kendi
makinende çalıştır" demekti.

**Burada `[uygulama]` çoğunlukla "kendi makinende çalıştır", bazen de "kâğıtta hesapla"
demektir.**

Ağın bir kısmı gözle görülmez: bir paketin router'da nasıl karar gördüğünü kendi makinende
izleyemezsin. Ama subnet hesabını **elle yapabilirsin**, ARP tablosuna **bakabilirsin**,
bir TCP el sıkışmasını `tcpdump` ile **yakalayabilirsin**. Bu yüzden bu kitapta iki tür
uygulama vardır: **`🔧 Makinende gör` kutuları** (komut + beklenen çıktı + satır satır
yorum) ve **kâğıt üstü hesaplar** (özellikle Faz 2).

> **Makinen yoksa ne yapmalısın?** Hiçbir şey. Bu kitap makinesiz de baştan sona işlenir —
> her çıktı yazılı ve yorumlanmıştır. Ama bir Linux makinesi (fiziksel, VM, WSL2 veya bir
> EC2 t3.micro) bulabilirsen öğrenme hızın **iki katına çıkar**. Faz 10'un bitirme lab'ı
> için bir makine neredeyse şarttır.

---

## Dosyalar

| Dosya | Faz | Konu | Yaklaşık süre |
|---|---|---|---|
| [Faz_0_Zihinsel_Model.md](Faz_0_Zihinsel_Model.md) | 0 | Katmanlar, encapsulation, PDU'lar, teşhis sırası olarak OSI | 3–4 saat |
| [Faz_1_Adresleme.md](Faz_1_Adresleme.md) | 1 | MAC, IPv4, port, bind adresi, özel bloklar, DHCP/DORA, APIPA | 5–7 saat |
| [Faz_2_Subnet_ve_CIDR.md](Faz_2_Subnet_ve_CIDR.md) | 2 | Maske, CIDR, kullanılabilir adres, VLSM, supernetting, çakışma | 6–8 saat |
| [**Ara_Sinav_1.md**](Ara_Sinav_1.md) | 0–2 | Birleşik sınav: model + adres + subnet | 1 saat |
| [Faz_3_Yerel_Ag_L2.md](Faz_3_Yerel_Ag_L2.md) | 3 | ARP, switch MAC tablosu, broadcast domain, VLAN | 4–6 saat |
| [Faz_4_Yonlendirme_L3.md](Faz_4_Yonlendirme_L3.md) | 4 | Varsayılan geçit, yönlendirme tablosu, LPM, TTL, ICMP, BGP | 6–8 saat |
| [**Ara_Sinav_2.md**](Ara_Sinav_2.md) | 3–4 | Birleşik sınav: "paket nereye gidiyor" | 1 saat |
| [Faz_5_Transport_Katmani.md](Faz_5_Transport_Katmani.md) | 5 | TCP vs UDP, el sıkışma, pencere, durumlar, MTU/MSS | 6–8 saat |
| [Faz_6_DNS.md](Faz_6_DNS.md) | 6 | Hiyerarşi, çözümleme zinciri, kayıt tipleri, TTL, cache | 5–6 saat |
| [Faz_7_NAT.md](Faz_7_NAT.md) | 7 | NAT/PAT, public-private ayrımı, IPsec tüneli, IPv6 | 5–7 saat |
| [Faz_8_HTTP_ve_TLS.md](Faz_8_HTTP_ve_TLS.md) | 8 | HTTP anatomisi, durum kodları, TLS el sıkışması, proxy, CDN | 5–7 saat |
| [Faz_9_Firewall.md](Faz_9_Firewall.md) | 9 | Stateful vs stateless, 5-tuple, DROP vs REJECT, SG vs NACL | 5–7 saat |
| [**Ara_Sinav_3.md**](Ara_Sinav_3.md) | 5–9 | Birleşik sınav: "paket neden düştü?" | 1–1.5 saat |
| [Faz_10_Troubleshooting.md](Faz_10_Troubleshooting.md) | 10 | Katman katman metodoloji, araç ustalığı, `tcpdump`, Flow Logs | 6–8 saat |
| [Faz_11_Clouda_Kopru.md](Faz_11_Clouda_Kopru.md) | 11 | VPC, route table, IGW/NAT GW, Route 53, SG/NACL, ALB/NLB | 4–6 saat |

**Ekler:**

| Dosya | Ne işe yarar |
|---|---|
| [EK_A_Komut_Sozlugu.md](EK_A_Komut_Sozlugu.md) | Tüm komutlar, amacı ve risk işaretiyle (🟢🟡🔴), geçtiği fazla — masaüstünde açık tut |
| [EK_B_Kavram_Dosya_Haritasi.md](EK_B_Kavram_Dosya_Haritasi.md) | Katmanlar, özel adres blokları, prefix tablosu, portlar, DNS kayıtları, HTTP/ICMP kodları, AWS eşlemesi |
| [EK_C_Bozulunca_Hizli_Basvuru.md](EK_C_Bozulunca_Hizli_Basvuru.md) | Belirti → muhtemel neden → doğrulama komutu → faz; tüm "Bozulunca" tablolarının tek sayfalık birleşimi |

**Şekiller:** Her fazın kilit mekanizması için bir diyagram vardır ([`../diagrams/png/`](../diagrams/png)).
Diyagramlar **dilden bağımsızdır** (etiketler İngilizce) — Türkçe ve İngilizce sürümler aynı
görselleri kullanır.

---

## Nasıl çalışılır

### 1. Sırayı bozma

Her faz bir öncekinin **kelimeleriyle** konuşur. Faz 4 "en uzun ön ek eşleşmesi"ni anlatırken
Faz 2'nin maske mantığını, Faz 9 "stateful firewall"ı anlatırken Faz 5'in TCP durum bilgisini
kullanır. Faz 10'un tamamı önceki on fazın "Bozulunca" notlarının üstüne kuruludur — tek
başına okunursa bir komut listesine dönüşür.

**Tek istisna:** Faz 8.3 (HTTP sürümleri) `[atla]` etiketlidir; adını bilmen yeter.

### 2. Kutuları ciddiye al

Metinde beş tür kutu var. Hepsinin farklı bir işi var:

> **🤔 Düşün 4.2** — Buradaki soruyu **okur okumaz cevaplama.** Kalem al, 2 dakika düşün,
> tahminini yaz. Cevabı fazın sonundaki *"Düşün sorularının cevapları"* bölümünde.
> Tahminin yanlış çıkarsa bu iyi bir şey — yanlış tahmin, doğru cevabı kalıcı yapar.

**❓ Akla gelen soru:** Okurken doğal olarak oluşan soru. Cevabı **hemen altında**.
Bunlar beklemez, çünkü cevaplanmazsa okuma akışını kırar.

**⚠️ Yaygın yanılgı:** Çoğu insanın yanlış bildiği şey. Bunları okurken "ben de böyle
biliyordum" diyorsan, o cümleyi işaretle.

**💡 Cloud bağlantısı:** Öğrendiğin mekanizmanın AWS'teki karşılığı. Faz 11 bu kutuların
birleşimidir — o yüzden her birini not al.

**🔧 Makinende gör:** Komut + **beklenen çıktı** + çıktının satır satır yorumu.
Kutunun başında üç işaretten biri vardır:

| İşaret | Anlamı |
|---|---|
| 🟢 | **Salt okunur.** Sistemini değiştirmez, gönül rahatlığıyla çalıştır. |
| 🟡 | **Geçici değişiklik.** Makine yeniden başlayınca eski hâline döner. |
| 🔴 | **Kalıcı değişiklik.** Üretim makinesinde yapma. Ne yaptığını anlamadan çalıştırma. |

Bu kitapta 🔴 işaretli komutların yanında **her zaman geri alma adımı** yazılıdır (Ek A.12'de toplanmıştır). Ağda
ekstra bir tehlike vardır: **kendi erişimini kesebilirsin.** Uzak makinede firewall veya
rota değiştirmeden önce Faz 9.5.3'ü oku.

### 3. Derinlik etiketlerine uy

| Etiket | Ne bekleniyor | Nasıl test edersin |
|---|---|---|
| `[kavram]` | Sezgisel açıklayabilmek | "Bunu bir arkadaşıma 30 saniyede anlatabilir miyim?" |
| `[mekanizma]` | Adım adım anlatabilmek | "Kalem kâğıtla akışı çizebilir miyim?" |
| `[uygulama]` | **Çalıştırıp çıktıyı okuyabilmek / elle hesaplayabilmek** | "Bu çıktıda anormal olanı görür müydüm?" |
| `[atla]` | Sadece adını bilmek | Adını duyunca "o şu alandaydı" diyebilmek yeter |

### 4. Faz sonunu atlama

Her faz şununla biter:

1. **Bu faz bozulunca — arıza imzaları** — belirti → mekanizma → ilk bakılacak yer tablosu
2. **Düşün sorularının cevapları** — kendi tahminlerinle karşılaştır
3. **Sık sorulan sorular** — fazın etrafındaki "peki ya şu?" soruları
4. **Kendini sına** — 18 soruluk test + tam cevap anahtarı + puanlama
5. **Kapanış ve köprü** — bu fazdan ne kaldı, sonraki faz bunun neresine bağlanıyor

Testte %70'in altında kaldıysan **fazı tekrar etmek yerine**, yanlış yaptığın sorunun
işaret ettiği bölüme geri dön. Her fazın sonunda "kaçırdığın soru → dönmen gereken bölüm"
tablosu vardır.

### 5. Ara sınavları atlama — bunlar farklı

Üç ara sınav (Faz 0–2, 3–4, 5–9) faz sınavlarından **farklı bir şey ölçer.**
Faz sınavı "bu fazı anladın mı?" diye sorar; ara sınav "**bu fazları birbirine
bağlayabiliyor musun?**" diye sorar. Soruların çoğu tek bir fazdan cevaplanamaz.

Gerçek iş de böyledir: hiçbir arıza tek bir fazın içinde kalmaz.

---

## İlerleme takibi

```
[ ] Faz 0  — Zihinsel Model: Katmanlar ve Zarflar
[ ] Faz 1  — Adresleme: Kim Kimdir
[ ] Faz 2  — Subnet ve CIDR                      ← kâğıt-kalem fazı, atlanmaz
[ ] ✅ Ara Sınav 1 (Faz 0–2)
[ ] Faz 3  — Yerel Ağ (L2): ARP ve Switch
[ ] Faz 4  — Yönlendirme (L3): Paket Nereye Gidiyor
[ ] ✅ Ara Sınav 2 (Faz 3–4)
[ ] Faz 5  — Transport Katmanı: TCP ve UDP
[ ] Faz 6  — DNS: İsimden Adrese
[ ] Faz 7  — NAT ve Gerçek Dünya
[ ] Faz 8  — Uygulama Katmanı: HTTP + TLS
[ ] Faz 9  — Firewall ve Filtreleme              ← "timeout mu refused mu" burada
[ ] ✅ Ara Sınav 3 (Faz 5–9)
[ ] Faz 10 — Troubleshooting                     ← haritanın zirvesi
[ ] Faz 11 — Cloud'a Köprü
```

---

## Kuzey yıldızı — üç içgüdü sorusu

Bu kitabın tamamı şu üç soruyu **düşünmeden** cevaplayabilmen için var:

| # | Soru | Nerede cevaplanıyor |
|---|---|---|
| 1 | **Veri nasıl ilerliyor?** | Faz 0, 1, 3, 4, 5 |
| 2 | **Packet neden düştü?** | Faz 4, 9, 10 |
| 3 | **Router neden saçmalıyor?** | Faz 4, 7, 10 |

Bu üç soru, bir cloud engineer'in ağ tarafındaki işinin neredeyse tamamıdır. Faz 10.3 bu
üçünün karar ağaçlarını tek yerde toplar.

**Tasarım prensibi:** troubleshooting sona bırakılmadı. Her fazda *"bu bilgi bozulunca nasıl
görünür?"* açısı var. İçgüdü ancak böyle oluşur — Faz 10 bu dağınık notları tek bir reflekse
dönüştürür.

---

## Bu kitabın hedefi ve sınırı

**Hedef:** Bir "bağlanamıyorum" şikâyeti geldiğinde, hangi katmanda ne olduğunu tahminle
değil **metodolojiyle** daraltabilmek; bir VPC tasarımındaki her kutunun altında hangi
temel kavramın durduğunu görebilmek.

**Sınır:** Bu kitap ağ mühendisi (CCNA/CCNP) yetiştirmez. Switch/router yapılandırma
sözdizimi, OSPF/EIGRP iç yapısı, MPLS ve operatör tarafı kapsam dışıdır. Amaç, bir cloud
engineer'in **kararları gerekçelendirecek ve arızayı avlayacak** kadar derinden görmesidir.

**Ortam varsayımı:** Komut örnekleri **Linux** (Ubuntu 22.04/24.04) üzerindedir; bulut
örnekleri **AWS** terminolojisini kullanır. Azure/GCP karşılıkları aynı kavramların başka
adlarıdır — Faz 11.8'deki eşleme tablosu o çeviriyi yapmanı sağlar.

---

## Ana haritayla ilişki

| Bu kitap | Ana harita |
|---|---|
| Offline, tek başına işlenir | YZ mentoruna verilir |
| Anlatım + soru + cevap | Konu listesi + mentor talimatı |
| Sabit içerik | Öğrencinin seviyesine göre mentor kalibre eder |
| Kendini sına testleri | Mentor sözlü yoklar |

İkisi **rakip değil**. Sıralı kullanım önerilir: önce bu kitapla fazı işle, sonra ana
haritayı bir YZ mentoruna verip aynı fazı **sözlü** tekrar et.

---

## Diğer kitaplarla ilişki

Bu seri üç kitaptan oluşur ve üçü birbirini **akıllıca sınırlar**:

| Kitap | Neyi alır | Bu kitapla kesişimi |
|---|---|---|
| [Donanım](../../Cloud_hardware.roadmap/Turkish_Hardware/README.md) | Fiziksel katman: CPU, bellek, disk, NIC, hypervisor | NIC, bant genişliği ve gecikmenin **altı**: Faz 5'in "neden yavaş" sorusu |
| [Linux](../../Cloud_linux.roadmap/Turkish_Linux/README.md) | İşletim sistemi: process, bellek, boot, izin, araçlar | Linux Faz 7 ağın **OS tarafını** alır (`ip`, `ss`, SSH); protokolün kendisi burada |
| **Ağ** (bu kitap) | Protokol teorisi: OSI, adresleme, yönlendirme, TCP, DNS, TLS, firewall | — |

Bu kitap diğer ikisinden **bağımsız yürür.** Donanım veya Linux kitabını okumadıysan hiçbir
yerde takılmazsın — gereken yerde ilgili kavram burada kısaca açıklanır, derinleşmek
istersen ilgili kitap işaret edilir.

---

*Bu offline sürüm, Denis Ergöçmen'in Cloud Engineer temel yol haritası serisine dayanır.*
