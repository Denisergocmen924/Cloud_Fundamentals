# Ek A — Komut Sözlüğü

> **Navigasyon:** [◀ Faz 11 — Cloud'a Köprü](Faz_11_Clouda_Kopru.md) · **Ek A** · [Ek B — Kavram ve Dosya Haritası ▶](EK_B_Kavram_Dosya_Haritasi.md)

---

> **Bu ek öğretmek için değil hatırlatmak için vardır.** Workbook'u bitirdikten sonra açık tutulacak sayfa
> budur. Her komutun yanında hangi fazdan geldiği yazılıdır — bir komut aklından çıktığında oraya dön. Risk
> işaretleri: 🟢 salt-okunur (güvenli), 🟡 geçici değişiklik, 🔴 kalıcı değişiklik (dikkatli ol).

**İçindekiler**
- [A.1 Adres ve arayüz (Faz 1)](#a1-adres-ve-arayüz-faz-1)
- [A.2 Subnet ve CIDR hesabı (Faz 2)](#a2-subnet-ve-cidr-hesabı-faz-2)
- [A.3 Yerel ağ — L2 (Faz 3)](#a3-yerel-ağ--l2-faz-3)
- [A.4 Yönlendirme — L3 (Faz 4)](#a4-yönlendirme--l3-faz-4)
- [A.5 Transport ve bağlantı (Faz 5)](#a5-transport-ve-bağlantı-faz-5)
- [A.6 DNS (Faz 6)](#a6-dns-faz-6)
- [A.7 NAT, tünel, IPv6 (Faz 7)](#a7-nat-tünel-ipv6-faz-7)
- [A.8 HTTP ve TLS (Faz 8)](#a8-http-ve-tls-faz-8)
- [A.9 Firewall ve filtreleme (Faz 9)](#a9-firewall-ve-filtreleme-faz-9)
- [A.10 Paket yakalama ve teşhis (Faz 10)](#a10-paket-yakalama-ve-teşhis-faz-10)
- [A.11 Bulut tarafı (Faz 11)](#a11-bulut-tarafı-faz-11)
- [A.12 Kalıcı değişiklikleri geri alma](#a12-kalıcı-değişiklikleri-geri-alma)

---

## A.1 Adres ve arayüz (Faz 1)

| Komut | Ne yapar | Risk |
|---|---|---|
| `ip addr` (`ip a`) | Arayüzler ve üzerlerindeki IP/prefix'ler | 🟢 |
| `ip -br addr` | Aynısı, tek satırlık özet biçimde | 🟢 |
| `ip link` | Arayüzlerin L2 durumu ve MAC adresleri | 🟢 |
| `ip link set eth0 up/down` | Arayüzü aç/kapat (geçici) | 🟡 |
| `ip addr add 10.0.0.5/24 dev eth0` | Geçici IP ekler (reboot'ta kaybolur) | 🟡 |
| `hostname -I` | Makinenin IP adres(ler)i | 🟢 |
| `ip -4 addr show eth0` | Yalnız IPv4 adresleri | 🟢 |
| `cat /sys/class/net/eth0/address` | Arayüzün MAC adresi (dosyadan) | 🟢 |
| `dhclient -v eth0` | DHCP kiralamasını yeniler (DORA) | 🟡 |
| `ss -tulpn` | Dinleyen portlar + hangi process (bind adresi!) | 🟢 |

## A.2 Subnet ve CIDR hesabı (Faz 2)

| Komut | Ne yapar | Risk |
|---|---|---|
| `ipcalc 10.0.4.0/22` | Ağ/broadcast/kullanılabilir aralık hesabı | 🟢 |
| `sipcalc 10.0.4.0/22` | Aynı işin daha ayrıntılı çıktısı | 🟢 |
| `ipcalc -s 10.0.0.0/16 256 256` | Verilen boyutlara böler (VLSM) | 🟢 |
| `python3 -c "import ipaddress;print(ipaddress.ip_network('10.0.4.0/22'))"` | Hesap makinesi yoksa | 🟢 |
| `python3 -c "import ipaddress;print(ipaddress.ip_address('10.0.5.7') in ipaddress.ip_network('10.0.4.0/22'))"` | "Bu IP bu bloğun içinde mi" | 🟢 |

> **Kafadan kontrol (Faz 2.3):** `/24` → 256 adres, `/25` → 128, `/26` → 64, `/27` → 32, `/28` → 16.
> AWS her subnet'te **5 adres ayırır** (`.0` ağ, `.1` router, `.2` DNS, `.3` gelecek, `.255` broadcast).

## A.3 Yerel ağ — L2 (Faz 3)

| Komut | Ne yapar | Risk |
|---|---|---|
| `ip neigh` (`arp -n`) | ARP tablosu: IP → MAC eşlemeleri | 🟢 |
| `ip neigh flush all` | ARP önbelleğini temizler (bayat kayıt şüphesi) | 🟡 |
| `arping -I eth0 10.0.0.1` | Doğrudan L2 seviyesinde erişim testi | 🟢 |
| `ip -s link show eth0` | Arayüz paket/hata sayaçları | 🟢 |
| `bridge fdb show` | Köprü/switch MAC öğrenme tablosu | 🟢 |
| `ip link add link eth0 name eth0.10 type vlan id 10` | VLAN alt arayüzü oluşturur | 🔴 |
| `tcpdump -ni eth0 arp` | Canlı ARP trafiğini izler | 🟢 |

## A.4 Yönlendirme — L3 (Faz 4)

| Komut | Ne yapar | Risk |
|---|---|---|
| `ip route` (`ip r`) | Yönlendirme tablosu (ilk bakılacak yer) | 🟢 |
| `ip route get 8.8.8.8` | **Bu hedef için hangi satır seçilir** (LPM kararı) | 🟢 |
| `ip route add 10.2.0.0/16 via 10.0.0.1` | Geçici rota ekler | 🟡 |
| `ip route add default via 10.0.0.1` | Varsayılan ağ geçidi tanımlar | 🟡 |
| `ip route del <ağ>` | Rota siler | 🔴 |
| `ping -c4 <hedef>` | Erişilebilirlik testi (ICMP Echo) | 🟢 |
| `ping -t 1 <hedef>` | TTL'i elle sınırla (traceroute'un mantığı) | 🟢 |
| `traceroute <hedef>` | Yol üzerindeki hop'lar (UDP tabanlı) | 🟢 |
| `traceroute -I <hedef>` / `-T -p 443` | ICMP / TCP ile izleme (filtreli ağlarda) | 🟢 |
| `cat /proc/sys/net/ipv4/ip_forward` | Makine router gibi davranıyor mu | 🟢 |

## A.5 Transport ve bağlantı (Faz 5)

| Komut | Ne yapar | Risk |
|---|---|---|
| `ss -tan` | TCP bağlantıları ve durumları (ESTAB, SYN-SENT…) | 🟢 |
| `ss -tulpn` | Dinleyen TCP/UDP portları + process | 🟢 |
| `ss -tan state syn-sent` | Yalnız takılı kalan handshake'ler | 🟢 |
| `nc -zv <host> 443` | Tek portun açık olup olmadığı (bağlan ve kapat) | 🟢 |
| `nc -zvu <host> 53` | UDP portu dener (cevapsızlık ≠ kapalı) | 🟢 |
| `nc -l 8080` | Test için port dinler | 🟡 |
| `ping -M do -s 1472 <hedef>` | MTU testi (parçalanma yasak) | 🟢 |
| `ip link show eth0 \| grep mtu` | Arayüzün MTU değeri | 🟢 |
| `tracepath <hedef>` | Yol boyunca MTU değişimini gösterir | 🟢 |
| `ss -i` | Bağlantı içi detay (rtt, cwnd, retransmit) | 🟢 |

> **Durum okuma (Faz 5.2/5.6):** `SYN-SENT` = cevap gelmiyor (DROP şüphesi) · `ESTAB` = sağlıklı ·
> `TIME-WAIT` = normal kapanış artığı · `CLOSE-WAIT` **birikiyorsa** uygulama soketi kapatmıyor.

## A.6 DNS (Faz 6)

| Komut | Ne yapar | Risk |
|---|---|---|
| `dig example.com` | Tam DNS cevabı (TTL, ANSWER, SERVER dahil) | 🟢 |
| `dig +short example.com` | Sadece sonuç | 🟢 |
| `dig @1.1.1.1 example.com` | **Belirli bir çözümleyiciye** sorar (cache'i atlar) | 🟢 |
| `dig example.com MX` / `NS` / `TXT` / `CNAME` | Kayıt tipine göre sorgu | 🟢 |
| `dig +trace example.com` | Kök'ten aşağı tüm çözümleme zinciri | 🟢 |
| `dig -x 93.184.216.34` | Ters çözümleme (PTR) | 🟢 |
| `host` / `nslookup` | Hızlı sorgu (dig kadar ayrıntı vermez) | 🟢 |
| `resolvectl status` | Sistemin kullandığı DNS sunucuları | 🟢 |
| `resolvectl flush-caches` | Yerel DNS önbelleğini temizler | 🟡 |
| `cat /etc/resolv.conf` | Çözümleyici ayarı (nereye soruluyor) | 🟢 |
| `getent hosts example.com` | Sistem çözümleme yolunu kullanır (`/etc/hosts` dahil) | 🟢 |

> **Ayrım (Faz 6.2.3):** `dig @1.1.1.1` çalışıp `dig` çalışmıyorsa sorun DNS'te değil, **senin
> çözümleyicindedir.** `getent` ile `dig` farklı sonuç veriyorsa `/etc/hosts` devrededir.

## A.7 NAT, tünel, IPv6 (Faz 7)

| Komut | Ne yapar | Risk |
|---|---|---|
| `curl -s ifconfig.me` | Dışarıdan görünen **public IP**'n (NAT'ın sonucu) | 🟢 |
| `ip addr` vs yukarıdaki | İçerideki private IP ile karşılaştır (RFC 1918) | 🟢 |
| `conntrack -L` | Bağlantı takip tablosu (NAT/stateful kayıtları) | 🟢 |
| `conntrack -C` | Tablodaki kayıt sayısı (tükenme şüphesi) | 🟢 |
| `iptables -t nat -L -n -v` | NAT kuralları (MASQUERADE/DNAT) | 🟢 |
| `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE` | Kaynak NAT kuralı ekler | 🔴 |
| `ip -6 addr` / `ip -6 route` | IPv6 adres ve rotaları | 🟢 |
| `ping6 <hedef>` / `curl -6` | IPv6 üzerinden test | 🟢 |
| `curl -4 <url>` | Dual-stack'te IPv4'ü zorla (ayrıştırma için) | 🟢 |
| `ip tunnel show` / `wg show` | Tünel / WireGuard durumu | 🟢 |
| `cat /proc/sys/net/netfilter/nf_conntrack_max` | Takip tablosu üst sınırı | 🟢 |

## A.8 HTTP ve TLS (Faz 8)

| Komut | Ne yapar | Risk |
|---|---|---|
| `curl -I <url>` | Sadece yanıt başlıkları (durum kodu) | 🟢 |
| `curl -v <url>` | Bağlantı + TLS + istek/yanıt zincirinin tamamı | 🟢 |
| `curl -w "%{time_total} %{http_code}\n" -o /dev/null -s <url>` | Süre ve kod ölçümü | 🟢 |
| `curl --resolve example.com:443:1.2.3.4 https://example.com` | DNS'i atlayıp **belirli sunucuyu** test eder | 🟢 |
| `curl -H "Host: example.com" http://1.2.3.4/` | Sanal host / yönlendirme testi | 🟢 |
| `openssl s_client -connect example.com:443 -servername example.com` | TLS el sıkışmasını ham hâliyle gör (SNI dahil) | 🟢 |
| `openssl s_client ... \| openssl x509 -noout -dates -subject` | Sertifika geçerlilik tarihi ve sahibi | 🟢 |
| `curl -vk <url>` | Sertifika doğrulamasını atla (**yalnız teşhis için**) | 🟡 |
| `curl --http1.1` / `--http2` | Protokol sürümünü zorla | 🟢 |

> **Kod okuma (Faz 8.2.2):** **502** = proxy arkadakine **bağlanamadı** · **504** = bağlandı ama
> **cevap gelmedi** · **4xx** = istemci tarafı · **5xx** = sunucu tarafı.

## A.9 Firewall ve filtreleme (Faz 9)

| Komut | Ne yapar | Risk |
|---|---|---|
| `iptables -L -n -v` | Kurallar + **paket sayaçları** (hangi kural çalışıyor) | 🟢 |
| `nft list ruleset` | nftables kural kümesinin tamamı | 🟢 |
| `ufw status verbose` | Basitleştirilmiş firewall durumu | 🟢 |
| `ufw allow 22/tcp` | Port açar (kalıcı) | 🔴 |
| `iptables -I INPUT -p tcp --dport 8080 -j ACCEPT` | Kuralı **en üste** ekler (sıra önemli) | 🔴 |
| `iptables -P INPUT DROP` | Varsayılan politikayı reddet yapar (**kendini kilitleyebilirsin**) | 🔴 |
| `iptables-save > /tmp/rules.bak` | Kuralları yedekler (değişiklikten **önce**) | 🟢 |
| `iptables-restore < /tmp/rules.bak` | Yedeği geri yükler (kurtarma) | 🔴 |
| `conntrack -L -p tcp` | Stateful tablodaki açık bağlantılar | 🟢 |

> **🔴 Kural (Faz 9.5.3):** Uzaktan kural değiştirmeden önce **ayrı bir oturum açık bırak** ve
> `iptables-save` ile yedek al. **Geri alma:** Konsol/Session Manager'dan `iptables-restore < yedek`.

## A.10 Paket yakalama ve teşhis (Faz 10)

| Komut | Ne yapar | Risk |
|---|---|---|
| `tcpdump -ni any 'tcp port 443'` | Canlı paket yakalama, filtreli | 🟢 |
| `tcpdump -ni any 'host 10.0.2.20 and port 5432'` | Belirli akışı izler | 🟢 |
| `tcpdump -ni any icmp` | ICMP mesajları (Fragmentation Needed dahil) | 🟢 |
| `tcpdump -w /tmp/yakala.pcap ...` | Dosyaya yazar (Wireshark ile incelemek için) | 🟡 |
| `mtr <hedef>` | Sürekli traceroute + kayıp istatistiği | 🟢 |
| `mtr -rwc 100 <hedef>` | 100 turluk rapor üretir | 🟢 |
| `ping -c 100 -i 0.2 <hedef>` | Kayıp oranı ölçümü | 🟢 |
| `ss -s` | Soket özeti (toplam bağlantı sayıları) | 🟢 |
| `journalctl -u <servis> -e` | Uygulama tarafının kanıtı | 🟢 |

> **tcpdump bayrak okuma (Faz 10.2.2):** `[S]` SYN · `[S.]` SYN-ACK · `[.]` ACK · `[P.]` veri ·
> `[F.]` FIN · `[R]` RST. **Sadece `[S]` görüyorsan** cevap gelmiyor demektir → DROP şüphesi.

## A.11 Bulut tarafı (Faz 11)

| Komut | Ne yapar | Risk |
|---|---|---|
| `aws ec2 describe-vpcs` | VPC'ler ve CIDR blokları | 🟢 |
| `aws ec2 describe-subnets --filters Name=vpc-id,Values=<vpc>` | Subnet'ler, AZ'ler, CIDR'ler | 🟢 |
| `aws ec2 describe-route-tables` | Route table'lar (public/private ayrımı burada) | 🟢 |
| `aws ec2 describe-security-groups --group-ids <sg>` | SG kuralları | 🟢 |
| `aws ec2 describe-network-acls` | NACL kuralları (numaralı sıra!) | 🟢 |
| `aws ec2 authorize-security-group-ingress ...` | SG'ye kural ekler | 🔴 |
| `aws logs filter-log-events --log-group-name <flow-logs> ...` | VPC Flow Logs sorgular (ACCEPT/REJECT) | 🟢 |
| `aws route53 list-resource-record-sets --hosted-zone-id <id>` | DNS kayıtları | 🟢 |
| `curl -s http://169.254.169.254/latest/meta-data/local-ipv4` | Instance'ın kendi private IP'si (metadata) | 🟢 |

> **Teşhis refleksi (Faz 10.1):** Bir şey bozulunca sıra hep aynı — **adres → yerel ağ → yönlendirme →
> isim → port → uygulama.** Rastgele komut deneme; her adımda bir katmanı ele ve ancak ondan sonra
> yukarı çık.

## A.12 Kalıcı değişiklikleri geri alma

Yukarıdaki her 🔴 komutun bir geri dönüş yolu vardır. Değiştirmeden önce **eski durumu kaydet** — uzak
makinede ise ikinci bir oturumu açık tut (Faz 9.5.3).

| Komut | Önce kaydet | Geri alma |
|---|---|---|
| `ip link add link eth0 name eth0.10 type vlan id 10` | `ip -d link show` | `ip link del eth0.10` |
| `ip route del <ağ>` | `ip route show > routes.bak` | `ip route add <ağ> via <gw> dev <arayüz>` (kaydedilen satırdan) |
| `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE` | `iptables-save > rules.bak` | `iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE` |
| `ufw allow 22/tcp` | `ufw status numbered` | `ufw delete allow 22/tcp` |
| `iptables -I INPUT -p tcp --dport 8080 -j ACCEPT` | `iptables-save > rules.bak` | `iptables -D INPUT -p tcp --dport 8080 -j ACCEPT` |
| `iptables -P INPUT DROP` | `iptables -S \| head -3` | `iptables -P INPUT ACCEPT` (veya `iptables-restore < rules.bak`) |
| `aws ec2 authorize-security-group-ingress ...` | `aws ec2 describe-security-groups --group-ids <sg>` | Aynı argümanlarla `aws ec2 revoke-security-group-ingress ...` |

---

> **Navigasyon:** [◀ Faz 11 — Cloud'a Köprü](Faz_11_Clouda_Kopru.md) · **Ek A** · [Ek B — Kavram ve Dosya Haritası ▶](EK_B_Kavram_Dosya_Haritasi.md)
