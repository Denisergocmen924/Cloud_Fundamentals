# Ek C — "Bozulunca" Hızlı Başvuru

> **Navigasyon:** [◀ Ek B — Kavram ve Dosya Haritası](EK_B_Kavram_Dosya_Haritasi.md) · **Ek C** · [Faz 11 — Cloud'a Köprü ▶](Faz_11_Clouda_Kopru.md)

---

> **Bu ek, tüm workbook'un "Bu faz bozulunca" tablolarını tek bir avlanma sayfasında birleştirir.** Bir belirti
> gördüğünde buraya bak: muhtemel neden, hangi komutla doğrularsın, hangi faza dönersin. Bu, Faz 10'un teşhis
> refleksinin cep hâlidir.

**İçindekiler**
- [C.1 Önce: teşhis refleksi (Faz 10)](#c1-önce-teşhis-refleksi-faz-10)
- [C.2 İki belirti, iki dünya: timeout vs refused](#c2-iki-belirti-iki-dünya-timeout-vs-refused)
- [C.3 Adresleme ve DHCP (Faz 1, 2)](#c3-adresleme-ve-dhcp-faz-1-2)
- [C.4 Yerel ağ — L2 (Faz 3)](#c4-yerel-ağ--l2-faz-3)
- [C.5 Yönlendirme — L3 (Faz 4)](#c5-yönlendirme--l3-faz-4)
- [C.6 Transport ve MTU (Faz 5)](#c6-transport-ve-mtu-faz-5)
- [C.7 DNS (Faz 6)](#c7-dns-faz-6)
- [C.8 NAT ve tünel (Faz 7)](#c8-nat-ve-tünel-faz-7)
- [C.9 HTTP ve TLS (Faz 8)](#c9-http-ve-tls-faz-8)
- [C.10 Firewall ve filtreleme (Faz 9)](#c10-firewall-ve-filtreleme-faz-9)
- [C.11 Bulut'a özgü (Faz 11)](#c11-buluta-özgü-faz-11)

---

## C.1 Önce: teşhis refleksi (Faz 10)

Panik yapma, rastgele kural açma. Aşağıdan yukarı ele:

| Sıra | Katman | İlk komut |
|---|---|---|
| 1 | Kendi adresim var mı | `ip addr` |
| 2 | Yerel ağ çalışıyor mu | `ping <gateway>` · `ip neigh` |
| 3 | Dışarı çıkabiliyor muyum | `ip route` · `ping 1.1.1.1` |
| 4 | İsim çözülüyor mu | `dig <alan>` |
| 5 | Port açık mı | `nc -zv <host> <port>` |
| 6 | Uygulama cevap veriyor mu | `curl -v <url>` |

> **Altın kural:** Her adımda bir katmanı **ele**, ancak ondan sonra yukarı çık. "Çalıştı sonra bozuldu"
> demek teşhis değildir; hangi adımda kırıldığını göster. **Kısayol (Faz 10.1.2):** Zamanın yoksa önce
> `ping 1.1.1.1` ve `dig <alan>` — bu ikisi ihtimallerin yarısını eler.

## C.2 İki belirti, iki dünya: timeout vs refused

*(Faz 9.2.3 — bu kitabın en çok işe yarayan tek ayrımı)*

| Belirti | Ne oldu | Nereye bak | Faz |
|---|---|---|---|
| **Timeout** (bekler, sonra hata) | Paket **sessizce düşürüldü** (DROP) | Firewall (SG/NACL/OS), route, NAT | 9.2, 4.2 |
| **Connection refused** (anında) | **RST** geldi — paket ulaştı, servis dinlemiyor | Uygulama, bind adresi, port | 5.2, 1.4 |
| **No route to host** (anında) | Yerel yönlendirme kararı bile verilemedi | `ip route`, gateway tanımı | 4.2 |
| **Name or service not known** | İsim çözülemedi, bağlantı hiç denenmedi | DNS, `/etc/resolv.conf` | 6.2 |

> **Tek cümle:** *Timeout = büyük ihtimalle firewall. Refused = firewall değil, servis dinlemiyor.*
> Bu ayrım, doğru ekibi ilk denemede bulmanı sağlar.

## C.3 Adresleme ve DHCP (Faz 1, 2)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| IP `169.254.x.x` | DHCP cevap vermedi (APIPA) | `ip addr`, `journalctl -u systemd-networkd` | 1.5 |
| Hiç IP yok | Arayüz down veya kablo/ENI sorunu | `ip link`, `ip -br addr` | 1.2 |
| IP var ama ağa çıkamıyor | Yanlış maske → komşuyu "uzak" sanıyor | `ip addr` (prefix'e bak), `ip route get` | 2.2 |
| Aynı IP iki makinede | Statik IP DHCP havuzuyla çakıştı | `arping -D`, `ip neigh` | 1.3, 3.1 |
| Subnet doldu, yeni makine açılmıyor | Adres alanı küçük planlanmış | `ipcalc`, kullanılan IP sayısı | 2.3 |
| Yeni subnet eklenemiyor | VPC CIDR'ı yetersiz (sonradan daraltılamaz) | `aws ec2 describe-vpcs` | 2.4, 11.1 |
| Dışarıdan erişilemiyor, port açık | Uygulama `127.0.0.1`'e bind olmuş | `ss -tulpn \| grep <port>` | 1.4 |

## C.4 Yerel ağ — L2 (Faz 3)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| Aynı subnet'teki makineye erişilemiyor | ARP çözülmüyor (L2 kopuk) | `ip neigh` → `INCOMPLETE`/`FAILED` | 3.1 |
| IP değişti, erişim kesildi | Bayat ARP kaydı | `ip neigh flush all`, tekrar dene | 3.1 |
| Ağ yavaş, herkes etkileniyor | Broadcast fırtınası / döngü | `tcpdump -ni any arp`, switch logları | 3.3 |
| Bazı makineler görünüyor bazıları yok | VLAN ayrımı (aynı switch, farklı segment) | VLAN yapılandırması, `ip -d link` | 3.4 |
| Erişim var ama çift yönlü değil | Yanlış/çakışan MAC ya da port güvenliği | `bridge fdb show`, arayüz sayaçları | 3.2 |

## C.5 Yönlendirme — L3 (Faz 4)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| Yerel ağ tamam, dışarı yok | Varsayılan rota eksik | `ip route` → `default` satırı var mı | 4.2 |
| Gidiyor ama cevap dönmüyor | **Dönüş rotası yok** (asimetrik) | Karşı tarafta `ip route get <senin-ip>` | 4.2 |
| Yanlış çıkışa gidiyor | Daha spesifik rota kazandı (LPM) | `ip route get <hedef>` | 4.3 |
| `traceroute` ortada `* * *` | Hop ICMP'ye cevap vermiyor — **arıza değil** | `mtr -rwc 100`, son hop'a bak | 4.4, 10.2 |
| Paket ulaşmıyor, TTL hatası | Yönlendirme döngüsü | `traceroute` → tekrarlayan hop'lar | 4.4 |
| Bir bölge erişilemiyor | Uzak taraftaki yönlendirme/BGP sorunu | `mtr`, sağlayıcı durum sayfası | 4.5 |

## C.6 Transport ve MTU (Faz 5)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| `SYN-SENT`'te takılı | Cevap hiç gelmiyor → DROP | `ss -tan state syn-sent`, `tcpdump` | 5.2, 9.2 |
| Bağlanıyor ama **donuyor** | MTU black hole (büyük paketler kayboluyor) | `ping -M do -s 1472 <hedef>` | 5.7 |
| Küçük istek çalışıyor, büyük çalışmıyor | Aynı sebep: MTU / MSS | `tracepath <hedef>` | 5.7 |
| Boşta kalan oturum ölüyor | NAT/firewall tablosundaki satır silindi | Keepalive ayarı, `conntrack -L` | 5.6, 7.1 |
| `CLOSE-WAIT` birikiyor | Uygulama soketi kapatmıyor (kod hatası) | `ss -tan state close-wait` | 5.6 |
| Aktarım yavaş ama kayıpsız | Pencere/RTT sınırı (bant genişliği değil) | `ss -i` (rtt, cwnd) | 5.4, 5.5 |
| UDP'de veri kayboluyor ve kimse haber vermiyor | UDP'de yeniden gönderim yok — normal | Uygulama katmanı sorumlu | 5.1 |

## C.7 DNS (Faz 6)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| Hiçbir isim çözülmüyor | Çözümleyici erişilemez | `cat /etc/resolv.conf`, `dig @1.1.1.1` | 6.2 |
| `dig @1.1.1.1` çalışıyor, `dig` çalışmıyor | Sorun **senin çözümleyicinde** | `resolvectl status` | 6.2 |
| Taşıdın ama eski sunucuya gidiyor | TTL süresince eski cevap cache'te | `dig` → TTL, `resolvectl flush-caches` | 6.4 |
| Bende çalışmıyor, herkeste çalışıyor | `/etc/hosts`'a unutulmuş satır | `getent hosts <isim>` vs `dig +short` | 6.2 |
| Apex alan adına CNAME eklenmiyor | Standart yasaklıyor | Route 53 **Alias** kullan | 6.3, 11.4 |
| İsim çözülüyor ama uygulama eski IP'ye gidiyor | Uygulama kendi cache'ini tutuyor | `ss -tan` → gerçek peer IP'ler | 6.4 |
| Alan adı bir anda kayboldu | NS delegasyonu veya kayıt süresi | `dig +trace`, `dig NS <alan>` | 6.1 |

## C.8 NAT ve tünel (Faz 7)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| İçeriden çıkılıyor, dışarıdan girilemiyor | NAT'ın doğası — eşleme yalnız içeriden başlar | `conntrack -L`, port yönlendirme | 7.1 |
| Yoğunlukta **kısmi** bağlantı hataları | Port tükenmesi / conntrack tablosu doldu | `conntrack -C` vs `nf_conntrack_max` | 7.2, 9.1 |
| Private makine güncelleme alamıyor | NAT Gateway veya rota yok | `ip route`, route table `0.0.0.0/0` | 7.3, 11.3 |
| VPN kurulu ama trafik akmıyor | Tünel ayakta, **rota tanımlı değil** | `ip route`, tünel arayüzü | 7.4 |
| VPN'de büyük paketler düşüyor | Tünel başlığı MTU'yu düşürdü | `ping -M do`, MSS clamping | 7.4, 5.7 |
| İki ağ birleşti, IP'ler çakışıyor | Aynı RFC 1918 bloğu iki tarafta | CIDR planı, yeniden adresleme | 7.3, 11.7 |
| IPv6'da çalışıyor, IPv4'te çalışmıyor (veya tersi) | Dual-stack'te tek taraf yapılandırılmış | `curl -4` / `curl -6` ile ayrıştır | 7.5 |

## C.9 HTTP ve TLS (Faz 8)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| **502** Bad Gateway | Proxy arkadakine bağlanamadı | Backend ayakta mı, SG kuralı | 8.2, 5.2 |
| **504** Gateway Timeout | Bağlandı, cevap zamanında gelmedi | Backend yavaş / takılı, timeout ayarı | 8.2 |
| 503 Service Unavailable | Sağlıklı hedef yok | Hedef grubu sağlık kontrolleri | 8.2, 11.6 |
| Sertifika hatası | Süresi dolmuş / alan adı kapsamıyor / SNI | `openssl s_client -servername ...` | 8.4 |
| Tarayıcıda çalışıyor, `curl`'de çalışmıyor | SNI, sertifika zinciri veya Host başlığı | `curl -v`, `--resolve` | 8.4 |
| Saat yanlış → TLS reddediliyor | Sertifika geçerlilik penceresi kayıyor | `timedatectl`, NTP | 8.4 |
| Değişiklik yaptım ama eski içerik geliyor | CDN veya tarayıcı cache'i | `curl -I` → cache başlıkları, invalidation | 8.5 |
| 301 sonrası yanlış adrese gidiyor | Kalıcı yönlendirme tarayıcıda cache'lenmiş | Özel pencerede dene | 8.2 |

## C.10 Firewall ve filtreleme (Faz 9)

| Belirti | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| Timeout, hiç cevap yok | DROP eden bir kural var | `iptables -L -n -v` (sayaçlar), `tcpdump` | 9.2 |
| SG açık ama erişilemiyor | NACL'de **giden ephemeral** kuralı eksik | NACL kuralları, Flow Logs REJECT | 9.4 |
| Kural ekledim, etkisi yok | Üstte daha genel bir kural yakalıyor | Kural sırası, `pkts` sayaçları | 9.2 |
| Ping çalışmıyor ama servis çalışıyor | ICMP kapalı — **arıza değil** | Port testi (`nc -zv`) ile doğrula | 9.3 |
| "Bağlanıyor ama donuyor" | ICMP Frag Needed engellenmiş | `ping -M do`, ICMP kuralları | 9.3, 5.7 |
| Uzaktan kural değiştirdim, kilitlendim | Kendi erişimini kesen kural | Konsol/Session Manager → `iptables-restore` | 9.5 |
| Herkes her yere erişebiliyor | Düz ağ: SG'ler birbirini referans almıyor | SG zinciri (alb-sg → app-sg → db-sg) | 9.5, 11.5 |

## C.11 Bulut'a özgü (Faz 11)

| Belirti (AWS) | Muhtemel neden | Doğrula | Faz |
|---|---|---|---|
| Public IP var ama erişilemiyor | Route table'da IGW rotası yok | `describe-route-tables` → `0.0.0.0/0` | 11.2, 11.3 |
| Private instance internete çıkamıyor | NAT Gateway yok veya rota eksik | NAT GW durumu + private route table | 11.3 |
| NAT GW var ama yine çıkamıyor | NAT GW **private** subnet'e konmuş | NAT GW'nin subnet'i public mi | 11.3 |
| S3'e erişim yavaş/pahalı | Trafik NAT üzerinden gidiyor | VPC Endpoint ekle | 11.3 |
| Peering var ama üçüncü VPC'ye erişilemiyor | Peering **geçişli değildir** | Rota tabloları, Transit Gateway | 11.7 |
| ALB 502 veriyor | Hedef SG, ALB SG'sinden trafiğe kapalı | Hedef grubu sağlık + SG zinciri | 11.5, 11.6 |
| Alan adı apex'te çalışmıyor | CNAME apex'te olmaz | Route 53 Alias kaydı | 11.4 |
| VPC içinde DNS çözülmüyor | `enableDnsSupport`/`enableDnsHostnames` kapalı | VPC DNS ayarları, `.2` resolver | 11.4 |
| Bağlantı düşüyor ama hata yok | Flow Logs'ta REJECT satırı | `filter-log-events` → ACCEPT/REJECT | 10.4 |
| Aynı AZ dışına çıkınca gecikme artıyor | Subnet'ler farklı AZ'de | Subnet-AZ eşlemesi | 11.2 |

> **Tüm workbook'un özeti — tek cümle:** Her belirtinin altında bir ağ gerçeği vardır. Panik yapma;
> paketin yolunu aşağıdan yukarı takip et, hangi tabloda hangi satırın eksik olduğunu bul, kök nedeni
> kalıcı düzelt. Bu refleks, bu workbook'un sana bıraktığı en değerli şeydir.

---

> **Navigasyon:** [◀ Ek B — Kavram ve Dosya Haritası](EK_B_Kavram_Dosya_Haritasi.md) · **Ek C** · [Faz 11 — Cloud'a Köprü ▶](Faz_11_Clouda_Kopru.md)
