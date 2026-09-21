# Ek B — Kavram ve Dosya Haritası

> **Navigasyon:** [◀ Ek A — Komut Sözlüğü](EK_A_Komut_Sozlugu.md) · **Ek B** · [Ek C — "Bozulunca" Hızlı Başvuru ▶](EK_C_Bozulunca_Hizli_Basvuru.md)

---

> **Bu ek, "bu kavram neydi, bu ayar nerede duruyor" sorusunun hızlı cevabıdır.** Her satırın yanında hangi
> fazdan geldiği yazılıdır. Ağda her şey bir **katmanda** durur (Faz 0) — bir sorunu anlamak, doğru katmana
> ve doğru tabloya bakmakla başlar.

**İçindekiler**
- [B.1 Katmanlar ve PDU'lar (Faz 0)](#b1-katmanlar-ve-pdular-faz-0)
- [B.2 Adres tipleri ve özel bloklar (Faz 1, 2)](#b2-adres-tipleri-ve-özel-bloklar-faz-1-2)
- [B.3 Prefix uzunluğu tablosu (Faz 2)](#b3-prefix-uzunluğu-tablosu-faz-2)
- [B.4 Sık kullanılan portlar (Faz 1, 5, 8)](#b4-sık-kullanılan-portlar-faz-1-5-8)
- [B.5 Makinedeki ağ dosyaları (Faz 1, 4, 6)](#b5-makinedeki-ağ-dosyaları-faz-1-4-6)
- [B.6 Makinedeki tablolar — hangi soru hangi tabloda (Faz 3, 4, 5, 7)](#b6-makinedeki-tablolar--hangi-soru-hangi-tabloda-faz-3-4-5-7)
- [B.7 DNS kayıt tipleri (Faz 6)](#b7-dns-kayıt-tipleri-faz-6)
- [B.8 HTTP durum kodları (Faz 8)](#b8-http-durum-kodları-faz-8)
- [B.9 ICMP mesaj tipleri (Faz 4, 5, 9)](#b9-icmp-mesaj-tipleri-faz-4-5-9)
- [B.10 Temel kavram → AWS karşılığı (Faz 11)](#b10-temel-kavram--aws-karşılığı-faz-11)

---

## B.1 Katmanlar ve PDU'lar (Faz 0)

*(Faz 0.2 — her katman bir öncekini zarfa koyar)*

| Katman | Adres | PDU adı | Cihaz | Örnek protokol | Faz |
|---|---|---|---|---|---|
| L7 Uygulama | URL / isim | Mesaj | — | HTTP, DNS, SSH | 6, 8 |
| L4 Transport | Port | Segment (TCP) / Datagram (UDP) | Yük dengeleyici (L4) | TCP, UDP | 5 |
| L3 Ağ | IP adresi | Paket | Router | IP, ICMP | 1, 4 |
| L2 Veri bağlantısı | MAC adresi | Frame | Switch | Ethernet, ARP | 1, 3 |
| L1 Fiziksel | — | Bit | Kablo, hub | — | 0 |

> **Kural (Faz 0.3):** Encapsulation aşağı inerken **başlık ekler**, yukarı çıkarken **soyar.** Bir
> paketin başına gelen her şey, bu beş satırdan birinde olur. Teşhiste ilk soru: *"Bu belirti hangi
> katmana ait?"*

## B.2 Adres tipleri ve özel bloklar (Faz 1, 2)

| Blok / Adres | Ne anlama gelir | Faz |
|---|---|---|
| `10.0.0.0/8` | Private (RFC1918) — büyük blok, bulutta en yaygın | 1.3 |
| `172.16.0.0/12` | Private (RFC1918) — Docker varsayılanları burada | 1.3 |
| `192.168.0.0/16` | Private (RFC1918) — ev/ofis ağları | 1.3 |
| `127.0.0.0/8` | Loopback — makinenin kendisi (dışarı çıkmaz) | 1.2 |
| `169.254.0.0/16` | APIPA / link-local — **DHCP başarısız** demektir | 1.5 |
| `169.254.169.254` | Bulut instance metadata servisi | 11.1 |
| `0.0.0.0/0` | "Her yer" — varsayılan rota veya "herkese açık" kural | 4.2, 9.2 |
| `0.0.0.0` (bind adresi) | "Tüm arayüzlerden dinle" | 1.4 |
| `255.255.255.255` | Sınırlı broadcast (DHCP Discover) | 1.5, 3.3 |
| `224.0.0.0/4` | Multicast | 3.3 |
| `fe80::/10` | IPv6 link-local | 7.5 |
| `::1` | IPv6 loopback | 7.5 |

> **Refleks (Faz 1.5):** Bir makinede `169.254.x.x` görüyorsan DHCP cevap vermemiştir — IP yapılandırması
> **hiç olmamıştır**, yanlış olmamıştır. Aranacak yer DHCP sunucusu veya L2 bağlantısıdır.

## B.3 Prefix uzunluğu tablosu (Faz 2)

*(Faz 2.2 — maske kaç biti ağa verir)*

| Prefix | Maske | Toplam adres | AWS'de kullanılabilir | Tipik kullanım |
|---|---|---|---|---|
| `/16` | 255.255.0.0 | 65.536 | 65.531 | VPC tamamı |
| `/20` | 255.255.240.0 | 4.096 | 4.091 | Büyük subnet |
| `/22` | 255.255.252.0 | 1.024 | 1.019 | Orta subnet |
| `/24` | 255.255.255.0 | 256 | 251 | Standart subnet |
| `/26` | 255.255.255.192 | 64 | 59 | Küçük subnet |
| `/28` | 255.255.255.240 | 16 | 11 | En küçük AWS subnet'i |
| `/32` | 255.255.255.255 | 1 | — | Tek host (kural/rota hedefi) |

> **AWS 5 adres ayırır (Faz 2.3.2):** `.0` ağ adresi · `.1` router · `.2` DNS çözümleyici ·
> `.3` gelecek kullanım · **son adres** broadcast. `/28` altına inemezsin.

## B.4 Sık kullanılan portlar (Faz 1, 5, 8)

| Port | Protokol | Servis | Not | Faz |
|---|---|---|---|---|
| 22 | TCP | SSH | Yönetim erişimi — asla `0.0.0.0/0`'a açma | 1.4, 9.5 |
| 53 | UDP (+TCP) | DNS | Büyük cevaplar TCP'ye düşer | 6.1 |
| 67/68 | UDP | DHCP | Broadcast tabanlı (DORA) | 1.5 |
| 80 | TCP | HTTP | Genelde 443'e yönlendirilir | 8.1 |
| 123 | UDP | NTP | Saat senkronizasyonu (TLS için kritik) | 8.4 |
| 443 | TCP (+UDP/QUIC) | HTTPS | HTTP/3 UDP üzerinde çalışır | 8.3, 8.4 |
| 3306 | TCP | MySQL | Yalnız uygulama SG'sine açılır | 11.5 |
| 5432 | TCP | PostgreSQL | Yalnız uygulama SG'sine açılır | 11.5 |
| 6379 | TCP | Redis | Varsayılanı kimlik doğrulamasız — dışa açma | 9.5 |
| 1024–65535 | TCP/UDP | Ephemeral aralık | İstemcinin geçici kaynak portu | 1.4, 5.1 |

> **Kritik ayrım (Faz 9.4.1):** NACL'in **giden** kuralında servisin portu değil, bu **ephemeral aralık**
> yazılır — çünkü cevap istemcinin geçici portuna gider ve NACL stateless'tır.

## B.5 Makinedeki ağ dosyaları (Faz 1, 4, 6)

| Dosya | Ne yapar | Faz |
|---|---|---|
| `/etc/hosts` | Yerel isim→IP eşlemesi; **DNS'ten önce** bakılır | 6.2 |
| `/etc/resolv.conf` | Hangi çözümleyiciye sorulacağı (`nameserver` satırları) | 6.2 |
| `/etc/nsswitch.conf` | İsim çözümleme **sırası** (`files dns`) | 6.2 |
| `/etc/netplan/*.yaml` | Kalıcı ağ yapılandırması (Ubuntu) | 1.2 |
| `/etc/systemd/network/*.network` | systemd-networkd yapılandırması | 1.2 |
| `/etc/ssh/sshd_config` | SSH dinleme adresi ve portu (`ListenAddress`) | 1.4 |
| `/proc/sys/net/ipv4/ip_forward` | Makine paket yönlendiriyor mu (router davranışı) | 4.1 |
| `/proc/sys/net/ipv4/tcp_keepalive_time` | Keepalive aralığı (NAT tablosu için) | 5.6, 7.1 |
| `/proc/sys/net/netfilter/nf_conntrack_max` | Bağlantı takip tablosu sınırı | 7.2, 9.1 |
| `/sys/class/net/<arayüz>/mtu` | Arayüzün MTU değeri | 5.7 |
| `/sys/class/net/<arayüz>/address` | Arayüzün MAC adresi | 1.1 |

> **Not (Faz 6.2.2):** `/etc/hosts` DNS'ten **önce** okunur. "Site herkeste çalışıyor, bende çalışmıyor"
> vakalarının klasik sebebi buraya unutulmuş bir satırdır. `getent hosts <isim>` ile `dig +short <isim>`
> farklı sonuç veriyorsa şüpheli budur.

## B.6 Makinedeki tablolar — hangi soru hangi tabloda (Faz 3, 4, 5, 7)

| Soru | Tablo | Komut | Faz |
|---|---|---|---|
| "Bu IP'nin MAC'i ne?" | ARP tablosu | `ip neigh` | 3.1 |
| "Bu hedefe hangi yoldan giderim?" | Yönlendirme tablosu | `ip route get <ip>` | 4.2 |
| "Şu an hangi bağlantılar açık?" | Soket tablosu | `ss -tan` | 5.2 |
| "Hangi portlar dinleniyor?" | Dinleyen soketler | `ss -tulpn` | 1.4 |
| "NAT hangi çeviriyi yaptı?" | Conntrack tablosu | `conntrack -L` | 7.1 |
| "Firewall hangi kuralı uyguladı?" | Kural zinciri + sayaç | `iptables -L -n -v` | 9.2 |
| "Switch bu MAC'i nereden öğrendi?" | MAC (FDB) tablosu | `bridge fdb show` | 3.2 |

> **Teşhis mantığı (Faz 10.1):** Her arıza bu tablolardan birinde bir **eksik veya bayat satır**
> olarak görünür. Tahmin etmek yerine ilgili tabloyu aç ve bak.

## B.7 DNS kayıt tipleri (Faz 6)

| Kayıt | Ne yapar | Dikkat | Faz |
|---|---|---|---|
| `A` | İsim → IPv4 | En temel kayıt | 6.3 |
| `AAAA` | İsim → IPv6 | Dual-stack'te ikisi birden bulunur | 6.3, 7.5 |
| `CNAME` | İsim → başka isim | **Apex'te (kök alan adında) kullanılamaz** | 6.3 |
| `ALIAS` / Route 53 `Alias` | Apex'te CNAME benzeri davranış | AWS'ye özgü çözüm | 11.4 |
| `MX` | Posta sunucusu | Öncelik değeri taşır | 6.3 |
| `NS` | Bu bölgenin yetkili sunucuları | Delegasyon zincirinin halkası | 6.1 |
| `TXT` | Serbest metin | SPF/DKIM, alan adı doğrulaması | 6.3 |
| `PTR` | IP → isim (ters) | Posta sunucuları için önemlidir | 6.3 |
| `SOA` | Bölgenin yetki bilgisi | Varsayılan negatif TTL burada | 6.1 |
| `SRV` | Servis konumu (host + port) | Servis keşfinde kullanılır | 6.3 |

> **Dangling CNAME (Faz 6.3.4):** CNAME'in hedefi silinir ama kayıt kalırsa, o adı başkası alabilir —
> **subdomain takeover.** Kaynağı silerken DNS kaydını da sil.

## B.8 HTTP durum kodları (Faz 8)

| Kod | Anlamı | Nerede aranır | Faz |
|---|---|---|---|
| 200 | Başarılı | — | 8.2 |
| 301 / 302 | Kalıcı / geçici yönlendirme | 301 tarayıcıda **cache'lenir**, dikkat | 8.2 |
| 400 | Hatalı istek | İstemci | 8.2 |
| 401 / 403 | Kimlik doğrulanmamış / yetki yok | 401 "kimsin", 403 "yetkin yok" | 8.2 |
| 404 | Bulunamadı | Yol veya yönlendirme kuralı | 8.2 |
| 429 | Çok fazla istek | Hız sınırı (rate limit) | 8.5 |
| 500 | Sunucu içi hata | Uygulama kodu | 8.2 |
| **502** | **Bad Gateway — proxy arkadakine bağlanamadı** | Backend ayakta mı, SG açık mı | 8.2, 5.2 |
| **504** | **Gateway Timeout — bağlandı, cevap gelmedi** | Backend yavaş / takılı | 8.2, 5.2 |
| 503 | Servis kullanılamıyor | Sağlıklı hedef yok (LB) | 8.2, 11.6 |

## B.9 ICMP mesaj tipleri (Faz 4, 5, 9)

| Tip/Kod | Mesaj | Ne anlatır | Faz |
|---|---|---|---|
| 8 / 0 | Echo Request / Reply | `ping`'in kendisi | 4.4 |
| 11 kod 0 | Time Exceeded | TTL bitti — `traceroute` bunu kullanır | 4.4 |
| 3 kod 0/1 | Network/Host Unreachable | Rota yok veya hedefe ulaşılamıyor | 4.4 |
| 3 kod 3 | Port Unreachable | UDP portu kapalı | 5.1 |
| **3 kod 4** | **Fragmentation Needed** | **MTU küçük — asla engelleme** | 5.7, 9.3 |
| 5 | Redirect | Daha iyi rota önerisi (genelde kapatılır) | 4.2 |

> **Altın kural (Faz 9.3):** Ping'i kısıtlayabilirsin, ama **Fragmentation Needed'ı asla engelleme** —
> engellersen MTU black hole oluşur ve belirti hiç firewall'a benzemez ("bağlanıyor ama donuyor").

## B.10 Temel kavram → AWS karşılığı (Faz 11)

| Temel kavram | AWS karşılığı | Faz |
|---|---|---|
| Adres alanı / özel ağ | VPC (CIDR bloğu) | 11.1 |
| Subnet | Subnet (bir AZ'ye bağlı) | 11.2 |
| Yönlendirme tablosu | Route Table (subnet'e ilişkilendirilir) | 11.2 |
| Varsayılan ağ geçidi | Internet Gateway / NAT Gateway | 11.3 |
| NAT / PAT | NAT Gateway | 11.3 |
| Stateful firewall | Security Group | 11.5 |
| Stateless filtre | Network ACL | 11.5 |
| DNS çözümleyici | VPC `.2` resolver / Route 53 Resolver | 11.4 |
| Yetkili DNS | Route 53 Hosted Zone | 11.4 |
| L4 yük dengeleyici | NLB | 11.6 |
| L7 yük dengeleyici | ALB | 11.6 |
| CDN / anycast | CloudFront | 11.6 |
| Özel ağlar arası bağlantı | VPC Peering / Transit Gateway | 11.7 |
| Şifreli tünel | Site-to-Site VPN | 11.7 |
| Özel servis erişimi | VPC Endpoint / PrivateLink | 11.3 |
| Paket kaydı | VPC Flow Logs | 10.4 |

> **Tek cümle (Faz 11.8):** Bulut yeni bir ağ icat etmedi; bu kitapta öğrendiğin ağı **bir API'nin
> arkasına koydu.** Konsolda gördüğün her kutunun altında bu ekte listelenen bir temel kavram vardır.

---

> **Navigasyon:** [◀ Ek A — Komut Sözlüğü](EK_A_Komut_Sozlugu.md) · **Ek B** · [Ek C — "Bozulunca" Hızlı Başvuru ▶](EK_C_Bozulunca_Hizli_Basvuru.md)
