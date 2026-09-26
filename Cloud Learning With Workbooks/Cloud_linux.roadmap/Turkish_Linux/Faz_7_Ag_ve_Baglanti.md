# Faz 7 — Ağ (OS Katmanı): Sunucunun Dış Dünyası

> **Navigasyon:** [◀ Ara Sınav 3](Ara_Sinav_3.md) · **Faz 7** · [Faz 8 — Paketler, Yazılım ve Servisleştirme ▶](Faz_8_Paketler_ve_Servislestirme.md)

---

## Nereden geliyoruz

Faz 6'nın sonunda sana bir bilmece bıraktık: bir servis "port'u dinliyor" ama dışarıdan erişilemiyorsa,
bu, diskteki "cihaz var ama mount yok → `df`'te yok" arızasının hangi akrabası? Cevap tam bu fazın konusu.
Şimdiye kadar hep **tek bir makinenin içindeydik**: process (Faz 3), bellek (Faz 4), boot (Faz 5), disk
(Faz 6). Faz 7 makineyi **dış dünyaya bağlıyor** — ve depolamada öğrendiğin o ders burada aynen tekrar
ediyor: bir kaynağın **var olması** ile **erişilebilir olması** ayrı şeylerdir.

Üç fazın bilgisi burada doğrudan işe yarayacak:

- **Faz 5 — servisler unit'tir.** SSH, nginx, uygulaman: hepsi systemd'nin başlattığı `.service`
  unit'leridir (5.3). "Servis dinliyor mu" sorusu aslında "unit `active (running)` mı" sorusudur.
- **Faz 3 — dinleyen bir process.** Bir portu bir **process** açar ve dinler (3.1). `ss -tulpn`'in son
  sütunundaki PID, Faz 3'ün process'idir; "port kapalı" çoğu zaman "process ölmüş"tür.
- **Faz 2 — SSH anahtarları.** Faz 2.3'te SSH anahtarının yolculuğunu (public/private, `authorized_keys`,
  izinler) gördün. Bu faz o temeli günlük operasyona döküyor: `~/.ssh/config`, ssh-agent, sıkılaştırma.

## Bu fazın sorusu

Faz 6 "veri diskte nasıl kalıcı olur" dedi. Bu faz makineyi ağa açıyor:

> *"Bir servis makinede çalışıyor — dışarıdan bir bağlantı ona ulaşmak için hangi kapılardan geçmek
> zorunda, ve o kapılardan biri kapalıysa arızayı nasıl bulurum?"*

Bir cloud engineer'in günlük hayatının çoğu tek bir cümlenin etrafında döner: **"Servise bağlanamıyorum,
neden?"** Bu tek soru, ağ arızalarının %90'ıdır ve cevabı neredeyse hiç "internet bozuk" değildir. Cevap
bir **kapılar zinciridir**: bağlantı, güvenlik grubundan (bulut), host firewall'undan (OS), ve servisin
**hangi adrese bind ettiğinden** (127.0.0.1 mı 0.0.0.0 mı) geçmek zorundadır. Bu fazın merkezinde SSH
vardır — çünkü bir cloud engineer'in en çok kullandığı, ve en çok "neden bağlanamıyorum" dediği araç odur.

Bu fazın sonunda, "servise bağlanamıyorum" olayını panik yapmadan, katman katman (isim çözülüyor mu → port
dinleniyor mu → doğru adrese mi bind → firewall/SG izin veriyor mu) teşhis eden bir refleks kazanacaksın.
Ve SSH'ı yalnızca "bağlanma komutu" olarak değil, güvenli yapılandırılması gereken bir servis olarak
göreceksin.

---

## Bu fazın sonunda

- `ip addr` ve `ip route` ile bir makinenin ağ durumunu (arayüzler, IP'ler, varsayılan geçit) okuyabilecek;
  netplan'ın Ubuntu'da ağ config'ini nasıl tuttuğunu kavram düzeyinde bileceksin
- İsim çözümlemenin host tarafını (`/etc/hosts`, `/etc/resolv.conf`, systemd-resolved) açıklayabilecek;
  "IP ile ping çalışıyor ama isimle çalışmıyor" arızasını `resolv.conf`'a bağlayabileceksin
- SSH'ı derinlemesine kullanabileceksin: anahtar tabanlı kimlik, `~/.ssh/config`, ssh-agent, port
  yönlendirme/tünelleme; ve sunucuyu sıkılaştırabileceksin (root login kapalı, parola kapalı, sadece anahtar)
- `ss -tulpn` ile hangi servisin hangi portu dinlediğini okuyabilecek; "servis ayakta ama bağlanılamıyor"
  arızasında `127.0.0.1` ile `0.0.0.0` bind farkını ayırt edebileceksin
- Host firewall'un (ufw/firewalld/nftables) ne olduğunu ve bulut Security Group'undan neden **ayrı** bir
  katman olduğunu; ikisinin birden trafiği kesebileceğini açıklayabileceksin
- **Cloud:** EC2'ye SSH ile bağlanabilecek, keypair'i yönetebilecek; SSM Session Manager'ın SSH portu
  açmadan neden daha güvenli erişim sunduğunu; ve Security Group ile host firewall'un iki bağımsız kapı
  olduğunu anlatabileceksin

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden burada |
|---|---|---|---|
| 7.1 | Arayüz ve adres yönetimi | `[mekanizma]` | Makinenin ağ kimliği: `ip addr`, `ip route` |
| 7.2 | İsim çözümleme (host tarafı) | `[mekanizma]` | "IP çalışıyor ama isim çözülmüyor" arızası |
| 7.3 | SSH — derinlemesine | `[uygulama]` | **Fazın kalbi** — günlük en çok kullanılan araç |
| 7.4 | Port ve soket durumu | `[uygulama]` | **"Servis ayakta ama bağlanılamıyor"** |
| 7.5 | Host firewall ve Security Group | `[kavram]` | İki ayrı savunma katmanı |
| 7.6 | Bu faz bozulunca | — | Ağ arızalarının imzaları |

> **Bu fazda nasıl çalışmalı:** Bu fazın gözlem komutları neredeyse tamamen 🟢'dir (`ip addr`, `ip route`,
> `ss -tulpn`, `cat /etc/resolv.conf`, `ssh -v` — hepsi okur/dener, kalıcı değişiklik yapmaz). İki yerde
> dikkat: (1) **SSH sıkılaştırması** 🔴'dır — `sshd_config`'i yanlış değiştirip oturumu kapatırsan
> **kendini dışarı kilitlersin** (özellikle bulutta, konsol erişimin yoksa). Kural: `sshd_config`
> değişikliğini **ikinci bir açık oturum** dururken test et, ve her adımın bir geri alma yolu olsun. (2)
> **Firewall kuralları** 🔴'dır — yanlış bir kural (özellikle SSH portunu kapatan) seni kilitler. En
> aydınlatıcı deney: kendi test instance'ında bir servisi önce `127.0.0.1`'e, sonra `0.0.0.0`'a bind edip
> `ss -tulpn` çıktısındaki farkı ve dışarıdan erişilebilirliğin nasıl değiştiğini görmek.

---
---

# 7.1 Arayüz ve Adres Yönetimi

## 7.1.1 Makinenin ağ kimliği: arayüz, IP, geçit `[mekanizma]`

Bir makinenin ağdaki kimliği üç parçadan oluşur ve `ip` komutu üçünü de gösterir:

1. **Arayüz (interface).** Makinenin ağa açılan "kapısı" — fiziksel bir kart veya bulutta sanal bir NIC.
   İsimleri: `lo` (loopback — makinenin kendisiyle konuştuğu iç arayüz, hep `127.0.0.1`), `eth0` / `ens5` /
   `enX0` (asıl ağ arayüzü; bulutta genelde tek bir birincil arayüz).
2. **IP adresi.** Arayüze atanmış adres — makinenin ağdaki "ev numarası". Bulutta bu genelde **özel** (private)
   bir IP'dir (`172.31.x.x` gibi); dışarıya açık **public** IP çoğu zaman ayrı bir katmanda (NAT/Elastic IP)
   durur ve makinenin içinde görünmeyebilir.
3. **Varsayılan geçit (default gateway) ve rota.** "Bu ağda olmayan her yere giden trafiği nereye
   göndereyim?" sorusunun cevabı. `ip route`'taki `default via ...` satırı budur; yanlışsa makine yerel ağı
   görür ama internete çıkamaz.

> **🔧 Makinende gör** 🟢 — makinenin ağ kimliğini oku
>
> ```
> $ ip addr
> 1: lo: <LOOPBACK,UP> ... inet 127.0.0.1/8 scope host lo
> 2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
>     inet 172.31.20.15/20 ... scope global ens5
> $ ip route
> default via 172.31.16.1 dev ens5
> 172.31.16.0/20 dev ens5 proto kernel scope link src 172.31.20.15
> ```
>
> `ens5` arayüzünün özel IP'si `172.31.20.15`. `default via 172.31.16.1` = internete/diğer ağlara giden
> trafik bu geçide gider. `lo`'nun `127.0.0.1`'i ise hep vardır ve dış dünyayla ilgisi yoktur — 7.4'te bu
> ayrım kritik olacak.

> **⚠️ Yaygın yanılgı: "`ip addr`'da public IP'imi göreceğim."**
>
> Genelde göremezsin. Bulutta makinenin **içinden** bakınca sadece **özel** IP görünür; public IP,
> makinenin dışında bir NAT katmanında ona eşlenir (1:1 map). Bu yüzden `ip addr` `172.31.x.x` gösterirken
> insanlar internetten `3.120.x.x` ile bağlanır — ikisi aynı makinedir. "Public IP'm nerede" sorusunun
> cevabı makinenin içinde değil, bulut konsolundadır (veya bir metadata sorgusuyla). Bu ayrım, "makinem
> internete çıkıyor ama ben ona bağlanamıyorum" gibi arızaları çözerken hayatidir.

## 7.1.2 netplan: Ubuntu'da ağ config'i `[kavram]`

`ip addr` sana ağın **o anki** durumunu gösterir — ama bu durum, tıpkı Faz 6'daki `mount` gibi, kalıcı
değildir (reboot'ta yeniden kurulur). Ağ ayarlarının **kalıcı** tanımı Ubuntu'da **netplan**'dadır:
`/etc/netplan/*.yaml` dosyaları, boot'ta hangi arayüzün nasıl yapılandırılacağını (DHCP mi, statik IP mi)
söyler. Bulutta bu dosya genelde "DHCP kullan" der ve gerisini bulutun DHCP sunucusu halleder — bu yüzden
bir EC2 instance'ı IP'sini otomatik alır.

Buradaki kavramsal bağ Faz 6 ile aynıdır: **çalışan durum** (`ip addr`, `mount`) ile **kalıcı config**
(netplan, `fstab`) ayrı katmanlardır. `ip addr add` ile elle eklediğin bir IP reboot'ta gider; netplan'a
yazarsan kalır. Bu fazda netplan'ı derinlemesine öğrenmen gerekmiyor — sadece "ağın kalıcı ayarı burada,
ve bulutta genelde DHCP olduğu için ona nadiren dokunursun" demeni bilmen yeter.

> **🤔 Düşün 7.1** — Bir makinede `ping 8.8.8.8` çalışıyor (IP ile internete çıkıyor) ama `ping google.com`
> "Name or service not known" veriyor. Sorun 7.1'deki üç parçadan (arayüz, IP, geçit) **hangisinde
> değildir**, ve bu gözlem seni bir sonraki bölümün (7.2) hangi dosyasına yönlendirir? (İpucu: IP ile
> çıkabiliyorsa arayüz/IP/geçit çalışıyor demektir.)
>
> *(Cevap: fazın sonunda)*

---
---

# 7.2 İsim Çözümleme (Host Tarafı)

## 7.2.1 İsimden IP'ye: çözümlemenin iki durağı `[mekanizma]`

İnsanlar `google.com` yazar, ama ağ `142.250.x.x` gibi bir IP ister. Bu çeviriye **isim çözümleme** (name
resolution / DNS) denir. Bir Linux makinesinde çözümleme sırayla iki yere bakar:

1. **`/etc/hosts`** — elle yazılmış, yerel bir "isim → IP" defteri. Buraya `10.0.0.5  db.internal` yazarsan,
   o makine `db.internal`'ı hiç DNS'e sormadan doğrudan `10.0.0.5` yapar. DNS'ten **önce** bakılır; bir ismi
   test için veya bir DNS'i geçici olarak ezmek için kullanılır.
2. **DNS sunucusu** — `/etc/hosts`'ta yoksa, makine yapılandırılmış bir DNS sunucusuna sorar. Hangi sunucuya
   soracağı `/etc/resolv.conf` dosyasındaki `nameserver` satırında yazar.

Modern Ubuntu'da araya bir de **systemd-resolved** girer: `/etc/resolv.conf` genelde `127.0.0.53` (yerel
resolved servisi) gösterir, o da asıl DNS sunucusuna (bulutta genelde VPC'nin resolver'ı) yönlendirir.
Detay önemli değil; kritik olan zincir: **isim → `/etc/hosts` → DNS sunucusu (`resolv.conf`) → IP.**

## 7.2.2 "IP çalışıyor ama isim çözülmüyor" — resolv.conf arızası `[mekanizma]`

Bu, ağ arızalarının en klasik ve en yanıltıcı olanlarından biridir çünkü "internet çalışıyor gibi ama
çalışmıyor" hissi verir. Belirti: `ping 8.8.8.8` **çalışır** (IP ile paket gidip geliyor — yani arayüz, IP,
geçit, internet yolu **sağlam**), ama `ping google.com` "Name or service not known" **verir**. Tek fark:
ikincisi bir **isim çözümlemesi** gerektiriyor. Demek ki arıza tam olarak çözümleme katmanındadır — büyük
olasılıkla `/etc/resolv.conf` boş, yanlış bir `nameserver` içeriyor, veya DNS sunucusu erişilemez.

> **🔧 Makinende gör** 🟢 — çözümleme zincirini denetle
>
> ```
> $ ping -c1 8.8.8.8            # IP ile: çalışıyorsa ağ yolu sağlam
> 64 bytes from 8.8.8.8: ...
> $ ping -c1 google.com        # isim ile: burada patlıyorsa çözümleme sorunu
> ping: google.com: Name or service not known
> $ cat /etc/resolv.conf       # hangi DNS sunucusu yazılı?
> nameserver 127.0.0.53
> $ resolvectl status          # systemd-resolved gerçekte hangi sunucuyu kullanıyor
> ```
>
> Teşhis mantığı: **IP çalışıp isim çalışmıyorsa, sorun her zaman çözümlemededir** — ağ yolunda değil.
> Bakılacak yer `/etc/resolv.conf` (ve `resolvectl status`). Sık sebep: `resolv.conf`'un boş olması veya
> erişilemez bir DNS sunucusu göstermesi.

> **💡 Cloud bağlantısı — bulutta DNS ve VPC resolver:** Bir VPC içindeki instance'lar, AWS'in sağladığı
> bir DNS resolver'ı kullanır (VPC'nin `.2` adresi, örn. `172.31.0.2`). Bu resolver hem internet isimlerini
> hem de VPC içi özel isimleri (örn. bir RDS endpoint'inin adı) çözer. "Instance internete çıkıyor ama bir
> AWS servis endpoint'ini çözemiyor" gibi arızalar genelde ya bu resolver'a giden yolun (route/SG) ya da
> `resolv.conf`/resolved config'inin bozulmasıdır. Kural yine aynı: önce `ping IP` ile ağ yolunu ayır,
> sonra isim çözümlemesini ayrı test et — ikisini karıştırma.

> **🤔 Düşün 7.2** — Bir uygulama "could not connect to database `db.prod.internal`" hatası veriyor. Sen
> `ping db.prod.internal` deniyorsun: "Name or service not known". Ama veritabanının IP'sini
> (`10.0.5.20`) biliyorsun ve `ping 10.0.5.20` **çalışıyor**. (a) Arıza hangi katmanda (ağ yolu mu,
> çözümleme mi)? (b) `/etc/hosts`'a hangi tek satırı ekleyerek uygulamayı **geçici** olarak çalışır hâle
> getirebilirsin, (c) bu neden kalıcı bir çözüm değil?
>
> *(Cevap: fazın sonunda)*

---
---

# 7.3 SSH — Derinlemesine

## 7.3.1 Anahtar tabanlı kimlik: neden parola değil `[uygulama]`

Faz 2.3'te SSH anahtarının yolculuğunu kavram olarak gördün; şimdi onu günlük operasyona döküyoruz. SSH,
uzak bir makinede güvenli bir kabuk (shell) açmanın standart yoludur ve bir cloud engineer'in en çok
kullandığı araçtır. Kimliğin iki yolu var, ama bulutta pratikte tek biri kullanılır:

- **Parola** — hatırlanır ama zayıf: tahmin edilebilir, brute-force edilebilir, sızdırılabilir. Bulutta
  genelde **kapatılır**.
- **Anahtar çifti (key pair)** — matematiksel olarak bağlı iki dosya: **private key** (sende kalır, asla
  paylaşılmaz) ve **public key** (sunucuya, `~/.ssh/authorized_keys`'e konur). Bağlanırken sunucu, sadece
  doğru private key'in sahibinin çözebileceği bir meydan okuma gönderir; parola hiç ağdan geçmez. Faz 2.3'in
  diyagramı tam bu akıştı.

Kritik izin kuralı (Faz 2'nin izinleri burada geri geliyor): SSH, güvenlik gereği **çok katıdır**. Private
key `600` (sadece sahibi okur), `~/.ssh` dizini `700` olmalıdır; aksi hâlde SSH anahtarı **sessizce
reddeder** ("izinler çok açık" der). "Anahtarım doğru ama yine parola soruyor / reddediyor" arızasının en
sık sebebi budur.

> **🔧 Makinende gör** 🟢 — anahtarla bağlan ve izinleri doğrula
>
> ```
> $ ls -l ~/.ssh/id_ed25519           # private key 600 olmalı
> -rw------- 1 sen sen 411 ... id_ed25519
> $ ssh -i ~/.ssh/id_ed25519 ubuntu@3.120.0.10
> # bağlanamazsan neden'i gör:
> $ ssh -v -i ~/.ssh/id_ed25519 ubuntu@3.120.0.10   # -v = adım adım el sıkışma
> ```
>
> `ssh -v` (veya `-vvv`) bağlantının **hangi adımda** takıldığını gösterir: TCP kuruldu mu, sunucu anahtarı
> sunuldu mu, hangi kimlik yöntemi denendi, neden reddedildi. "Bağlanamıyorum"u "şu adımda, şu sebeple
> bağlanamıyorum"a çeviren tek araç budur — 7.6'nın teşhis refleksinin merkezinde durur.

## 7.3.2 `~/.ssh/config` ve ssh-agent: günlük ergonomi `[uygulama]`

Her seferinde `ssh -i uzun/yol/key.pem ubuntu@3.120.0.10 -p 22` yazmak hem yorucu hem hataya açıktır.
`~/.ssh/config` bunu isimlendirilmiş kısayollara çevirir:

```
Host prod-web
    HostName 3.120.0.10
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
```

Artık sadece `ssh prod-web` yeter. Bu sadece konfor değil; doğru anahtarı doğru sunucuyla eşleştirerek
"yanlış key'i yolladım" hatalarını da azaltır.

**ssh-agent** ise private key'ini (parola korumalıysa) bir kez açıp bellekte tutan bir yardımcıdır: her
bağlantıda anahtarın parolasını tekrar tekrar girmezsin, ama anahtar diske şifresiz yazılmaz. `ssh-add` ile
anahtarı agent'a eklersin.

> **💡 Cloud bağlantısı — EC2 keypair yönetimi:** Bir EC2 instance'ı başlatırken bir **keypair** seçersin;
> AWS, seçtiğin public key'i instance'ın `~/.ssh/authorized_keys`'ine (genelde `ubuntu` veya `ec2-user`
> kullanıcısı için) otomatik koyar. Private key **sende** kalır — AWS onu saklamaz, kaybedersen yeniden
> indiremezsin. Bu yüzden "instance'a bağlanamıyorum, key'i kaybettim" ciddi bir durumdur (çözüm genelde:
> instance'ı durdur, kök diski başka instance'a bağla, `authorized_keys`'i düzelt — Faz 6 bilgisi!). Kural:
> private key'lerini bir kez üret, güvenle sakla, ve sunuculara sadece **public** tarafı dağıt.

## 7.3.3 Port yönlendirme (tünelleme): kapalı bir servise güvenli erişim `[kavram]`

Bazen bir servise doğrudan erişemezsin çünkü o servis sadece **makinenin içinden** (`127.0.0.1`) dinliyor
veya firewall/SG dışarıya kapalı — ama SSH'ın var. SSH **port yönlendirme** (tünelleme) tam bu durumu
çözer: SSH bağlantısının içinden bir tünel açıp, uzak makinedeki bir portu **kendi yerel makinene** taşırsın.

Örnek: uzak sunucuda bir veritabanı `127.0.0.1:5432`'de dinliyor (dışarıya kapalı, güvenli). Şu komut:

```
$ ssh -L 5432:127.0.0.1:5432 prod-web
```

kendi makinendeki `localhost:5432`'yi, SSH tüneli üzerinden uzak sunucunun `127.0.0.1:5432`'sine bağlar.
Artık yerel bir DB aracıyla `localhost:5432`'ye bağlanırsın, trafik şifreli SSH tünelinden akar. Bu, "DB
portunu internete açmadan" ona erişmenin güvenli yoludur ve bulutta çok kullanılır (bastion/jump host
mantığı).

## 7.3.4 SSH sıkılaştırma: sunucuyu kilitlemek `[uygulama]`

SSH açık bir kapıdır; internete bakan her sunucu sürekli SSH brute-force denemeleri alır. Sıkılaştırmanın
üç temel adımı (`/etc/ssh/sshd_config`'te) vardır:

1. **Parola girişini kapat** (`PasswordAuthentication no`) — sadece anahtar. Brute-force'u anlamsız kılar.
2. **root ile doğrudan girişi kapat** (`PermitRootLogin no`) — önce normal kullanıcı, sonra `sudo`. Saldırı
   yüzeyini daraltır ve denetim izini iyileştirir.
3. Değişiklikten sonra `sudo systemctl restart ssh` (Faz 5!) ile sshd'yi yeniden başlat.

> **🔧 Makinende gör** 🔴 — SSH sıkılaştır (kendini kilitlemeden)
>
> ```
> # /etc/ssh/sshd_config içinde:
> #   PasswordAuthentication no
> #   PermitRootLogin no
> $ sudo sshd -t                       # 🟢 önce config'i SÖZDİZİMİ açısından doğrula
> $ sudo systemctl reload ssh          # 🟡 uygula (mevcut oturumları düşürmez)
> ```
>
> **Geri alma:** Bu 🔴'dır çünkü yanlış bir `sshd_config` seni **dışarı kilitleyebilir**. Altın kural:
> değişikliği uygulamadan önce `sudo sshd -t` ile sözdizimini doğrula, ve **ikinci bir açık SSH oturumu
> dururken** test et — yeni bir oturum açabildiğini görmeden mevcut oturumunu kapatma. Kilitlenirsen geri
> dönüş: bulut konsolu / seri konsol ile gir ve `sshd_config`'i eski hâline al (Faz 6'daki kurtarma
> mantığının ağ versiyonu).

> **💡 Cloud bağlantısı — SSM Session Manager: SSH portu hiç açmadan erişim:** AWS SSM Session Manager,
> instance'a **22 portunu internete hiç açmadan** kabuk erişimi verir: instance üstündeki bir ajan AWS'e
> giden (outbound) bir bağlantı kurar, sen de AWS konsolu/CLI üzerinden o ajana bağlanırsın. Avantajları:
> (1) inbound SSH portu kapalı kalır → saldırı yüzeyi neredeyse sıfır; (2) erişim IAM ile yetkilendirilir
> (Faz 9) → SSH anahtarı dağıtmak/döndürmek gerekmez; (3) her oturum loglanır → denetim. "SSH'ı sıkılaştırmak"
> yerine "SSH'ı hiç açmamak" çoğu modern kurulumda tercih edilen yoldur. Yine de SSH'ı bilmek şarttır —
> hem eski sistemler hem de tünelleme için.

> **🤔 Düşün 7.3** — Yeni ürettiğin bir anahtarla EC2 instance'ına bağlanmaya çalışıyorsun ama sürekli
> "Permission denied (publickey)" alıyorsun. `ssh -v` çıktısı doğru anahtarı sunduğunu ama sunucunun
> reddettiğini gösteriyor. İki olası sebebi say: biri **sunucudaki** `authorized_keys`/izinlerle (Faz 2),
> biri **senin tarafındaki** private key izinleriyle (7.3.1) ilgili olsun. Her biri için doğrulama komutunu
> yaz.
>
> *(Cevap: fazın sonunda)*

---
---

# 7.4 Port ve Soket Durumu

## 7.4.1 Hangi servis hangi portu dinliyor: `ss -tulpn` `[uygulama]`

Bir servisin "ağa açık" olması demek, bir **portu dinliyor** olması demektir. Port, bir makinedeki
numaralandırılmış bir bağlantı noktasıdır (SSH 22, HTTP 80, HTTPS 443, PostgreSQL 5432...). Bir process
(Faz 3!) bir portu açar ve o porta gelen bağlantıları dinler. Hangi portların dinlendiğini ve arkasındaki
process'i görmenin standart aracı `ss`'tir:

> **🔧 Makinende gör** 🟢 — dinlenen portları ve process'lerini oku
>
> ```
> $ sudo ss -tulpn
> Netid State  Local Address:Port   Process
> tcp   LISTEN 0.0.0.0:22           users:(("sshd",pid=712,...))
> tcp   LISTEN 127.0.0.1:5432       users:(("postgres",pid=980,...))
> tcp   LISTEN 0.0.0.0:80           users:(("nginx",pid=1120,...))
> ```
>
> Bayrakların anlamı: `-t` TCP, `-u` UDP, `-l` sadece dinleyenler (listening), `-p` process adı/PID, `-n`
> isimleri çözmeden sayısal göster. Son sütun seni doğrudan Faz 3'e bağlar: her portun arkasında bir PID
> vardır. "Port kapalı" gördüğünde ilk soru: process ayakta mı (Faz 5: `systemctl status`, Faz 3: `ps`)?

## 7.4.2 "Servis ayakta ama bağlanılamıyor": 127.0.0.1 vs 0.0.0.0 `[mekanizma]`

Şimdi bu fazın en önemli tek ayrımına geldik — ve bu, Faz 6'da söz verdiğimiz "var olmak ≠ erişilebilir
olmak" boşluğunun ağ versiyonudur. Yukarıdaki çıktıda iki servis dinliyor ama **farklı adreslerde**:

- **`0.0.0.0:22` (sshd, nginx)** — `0.0.0.0` "**tüm** arayüzlerde dinle" demektir; hem `lo` (127.0.0.1) hem
  de dış arayüz (`ens5`, 172.31.x.x) üzerinden gelen bağlantıları kabul eder. Yani **dışarıdan erişilebilir**
  (firewall/SG izin verdiği sürece).
- **`127.0.0.1:5432` (postgres)** — `127.0.0.1` "**sadece** loopback'te, yani sadece makinenin **kendi
  içinden** dinle" demektir. Bu porta dışarıdan hiçbir bağlantı ulaşamaz — process ayakta, port "açık", ama
  yalnızca yerel.

İşte "servis çalışıyor (`systemctl` yeşil), port dinleniyor (`ss` gösteriyor), ama dışarıdan bağlanamıyorum"
arızasının bir numaralı sebebi: servis `127.0.0.1`'e **bind** etmiş, `0.0.0.0`'a değil. Çözüm servisin kendi
config'inde bind adresini değiştirmektir (örn. PostgreSQL'de `listen_addresses`, birçok uygulamada
`--host 0.0.0.0`). Bu bilinçli bir güvenlik tercihi de olabilir: bir DB'yi kasten sadece yerelde tutup ona
SSH tüneliyle (7.3.3) erişmek yaygın ve güvenli bir kalıptır.

> **❓ Akla gelen soru: "`systemctl status` yeşil (`active running`) diyor, o hâlde servis erişilebilir
> olmalı, değil mi?"**
>
> Hayır — bu üç ayrı sorudur ve `systemctl` sadece birincisini yanıtlar. (1) **Process çalışıyor mu?**
> → `systemctl status` (Faz 5). (2) **Doğru adreste dinliyor mu?** → `ss -tulpn` (`0.0.0.0` mı `127.0.0.1`
> mi). (3) **Firewall/SG izin veriyor mu?** → 7.5. "Yeşil" sadece (1)'dir. Erişilebilirlik üçünün de
> yeşil olmasını gerektirir. Bir engineer'i acemi'den ayıran şey, "yeşil ama çalışmıyor"da bu üç katmanı
> sırayla eleyebilmektir.

## 7.4.3 Bağlantının geçmek zorunda olduğu kapılar `[mekanizma]`

Yukarıdaki üç soruyu tek bir zihinsel modele dizelim. Dışarıdan gelen bir bağlantının servise ulaşması için
**sırayla** birkaç kapıdan geçmesi gerekir; herhangi biri kapalıysa bağlantı sessizce ölür:

![Şekil 7.1 — Bir bağlantının geçmek zorunda olduğu kapılar: DNS çözümlemesi → Security Group (bulut) → host firewall (ufw) → servisin bind adresi (0.0.0.0 mı 127.0.0.1 mi) → process. Herhangi bir kapı kapalıysa "bağlanamıyorum"; teşhis, kapıları sırayla elemektir.](../diagrams/png/lx-7-01-connection-gates.png)

Bu model, "servise bağlanamıyorum" olayının teşhis haritasıdır. Bir bağlantı istemciden servise giderken
şu kapılardan geçer: (0) **isim çözümlenir mi** (7.2 — yoksa hiç başlamaz); (1) **Security Group** bulut
tarafında portu açıyor mu (7.5); (2) **host firewall** OS tarafında portu açıyor mu (7.5); (3) servis
**doğru adrese bind** etmiş mi (7.4.2 — `0.0.0.0`); (4) arkasındaki **process ayakta mı** (Faz 3/5).
Teşhis, bu kapıları dıştan içe (veya içten dışa) sırayla elemektir — panik değil, sıra.

> **🤔 Düşün 7.4** — Bir web uygulamasını başlattın ve `systemctl status` yeşil. `ss -tulpn` şunu
> gösteriyor: `127.0.0.1:8080  users:(("app",pid=2210,...))`. Laptop'ından `curl http://<public-ip>:8080`
> `Connection refused` diyor, oysa Security Group 8080'i açık ve `ufw` kapalı. (a) 7.4.2'deki üç sorudan
> hangisi kırmızı ve `ss` çıktısının neresi bunu gösteriyor? (b) Neyi, nerede değiştirirsin? (c) Bir
> servis için `127.0.0.1` ne zaman *doğru* cevaptır?
>
> *(Cevap: fazın sonunda)*

---
---

# 7.5 Host Firewall ve Security Group — İki Ayrı Katman

## 7.5.1 İki bağımsız kapı: OS firewall ve bulut Security Group `[kavram]`

Bir bağlantının geçmek zorunda olduğu "kapılardan" ikisi firewall'dur ve bunlar **iki ayrı katmandır** —
bu ayrımı karıştırmak bulutta en sık kafa karışıklığıdır:

- **Host firewall (OS seviyesi)** — makinenin **içinde** çalışan bir paket filtresi: **ufw** (Ubuntu'nun
  basit arayüzü), **firewalld** (RHEL ailesi), veya alttaki **nftables/iptables**. Kuralları makinenin
  kendisi uygular; `sudo ufw status` ile görülür.
- **Security Group (bulut seviyesi)** — makinenin **dışında**, bulut sağlayıcının ağ katmanında çalışan bir
  sanal firewall. Instance'a hiç girmeden, ona giden/gelen trafiği bulut tarafında filtreler. AWS konsolundan
  yönetilir, makinenin içinden **görünmez**.

Kritik nokta: **ikisi de trafiği kesebilir, ve ikisi de açık olmalıdır.** Bir bağlantının geçmesi için hem
Security Group hem host firewall o portu açık tutmalıdır — biri kapatırsa bağlantı ölür. Bu yüzden "SG'de
80 portunu açtım ama hâlâ bağlanamıyorum" arızasının cevabı sık sık host firewall'dur (veya tersi).

> **🔧 Makinende gör** 🟢 — host firewall durumunu oku
>
> ```
> $ sudo ufw status verbose
> Status: active
> To                         Action      From
> --                         ------      ----
> 22/tcp                     ALLOW IN    Anywhere
> 80/tcp                     ALLOW IN    Anywhere
> # 5432 burada YOK → host firewall onu kesiyor (SG açık olsa bile)
> ```
>
> Bu çıktı sadece **host** katmanını gösterir. Security Group'u buradan **göremezsin** — onu bulut
> konsolundan kontrol etmen gerekir. Teşhiste ikisini ayrı ayrı ele: `ufw status` (host) + konsol/CLI (SG).

> **💡 Cloud bağlantısı — neden iki katman (derinlemesine savunma):** İki firewall katmanı gereksiz bir
> tekrar değil, kasıtlı bir **derinlemesine savunma** (defense in depth) tasarımıdır. Security Group geniş,
> ağ-seviyesi bir sınırdır (hangi instance'lar hangi portlarda konuşabilir); host firewall ise o instance'a
> özgü, ince ayarlı bir ikinci sınırdır. Biri yanlış yapılandırılsa (veya bir instance yanlış SG'ye düşse)
> diğeri hâlâ koruyabilir. Faz 9'da bunu AppArmor/SELinux (uygulama-seviyesi zorlama) ile birlikte tam bir
> katmanlı savunma olarak göreceksin. Şimdilik kural: bir bağlantı arızasında **her iki firewall katmanını
> da** kontrol et — biri "açık" diye diğerini varsayma.

> **🤔 Düşün 7.5** — Security Group'ta 80. portu açtın ve `ss -tulpn` nginx'i `0.0.0.0:80`'de gösteriyor.
> Yine de dışarıdan `curl` zaman aşımına kadar takılı kalıyor. `sudo ufw status verbose` çıktısı `Status:
> active` ve yalnızca `22/tcp` izinli. (a) Hangi kapı kapanıyor ve Security Group konsolu bunu neden
> göstermiyor? (b) Çözüm nedir ve dışarıdan nasıl doğrularsın? (c) Başka bir instance da zaman aşımına
> uğruyor ama `ufw`'su kapalı. Sonra nereye bakarsın?
>
> *(Cevap: fazın sonunda)*

---
---

# 7.6 Bu Faz Bozulunca — Ağ Arıza İmzaları

Ağ arızaları acemi mühendisi en çok panikleten türdür çünkü belirti hep aynıdır: "bağlanamıyorum". Ama
neredeyse hepsi, 7.4.3'teki **kapılar zincirinin** bir halkasına oturur. Aşağıdaki tablo belirtiyi doğru
kapıya götürür:

| Belirti | Muhtemel sebep | Bakılacak yer | İlgili bölüm |
|---|---|---|---|
| `ping IP` çalışıyor, `ping isim` çalışmıyor | İsim çözümleme bozuk (resolv.conf/DNS) | `cat /etc/resolv.conf`, `resolvectl status` | 7.2.2 |
| Servis `active (running)` ama dışarıdan bağlanılamıyor | `127.0.0.1`'e bind etmiş, `0.0.0.0`'a değil | `ss -tulpn` (Local Address sütunu) | 7.4.2 |
| `ss`'te port hiç görünmüyor | Process ölmüş / servis çökmüş | `systemctl status`, `ps`, `journalctl -u` | 7.4.1 (Faz 5) |
| Port dinleniyor, host firewall açık, hâlâ yok | Security Group portu kapalı (bulut katmanı) | Bulut konsolu / CLI (SG kuralları) | 7.5.1 |
| SG açık ama hâlâ bağlanılamıyor | Host firewall (ufw) portu kesiyor | `sudo ufw status` | 7.5.1 |
| SSH "Permission denied (publickey)" | Yanlış anahtar / `authorized_keys` / izinler | `ssh -v`, sunucuda `ls -l ~/.ssh` | 7.3.1 (Faz 2) |
| SSH "Connection timed out" | Bağlantı kapıya hiç ulaşmıyor (SG/firewall/yanlış IP) | `ssh -v` (nerede takıldı), SG, IP | 7.5.1 |
| SSH sonrası kilitlendim | `sshd_config` yanlış / firewall SSH'ı kesti | Seri/bulut konsolu ile gir, geri al | 7.3.4 |

> **Bu tablodan çıkan ders:** "Bağlanamıyorum" bir teşhis değil, bir **başlangıç noktasıdır**. Her ağ
> arızası 7.4.3'teki kapılar zincirinin bir halkasıdır, ve doğru refleks paniklemek değil **sırayla
> elemektir**: (0) isim çözülüyor mu (`ping IP` vs `ping isim`) → (1/2) firewall katmanları açık mı (`ufw` +
> SG) → (3) servis doğru adrese mi bind (`ss -tulpn`) → (4) process ayakta mı (`systemctl status`). İki
> ayrımı asla karıştırma: **IP vs isim** (ağ yolu mu çözümleme mi) ve **127.0.0.1 vs 0.0.0.0** (yerel mi
> dışa açık mı). Ve unutma: bir makine erişilemez hâle geldiyse (özellikle SSH sıkılaştırma/firewall
> hatasıyla), kurtarma yolun Faz 6'dakiyle aynı ailedendir — **seri/bulut konsolu**, çünkü ağ katmanı
> çöktüğünde `ssh` zaten işe yaramaz.

---
---

# Faz 7 — Düşün sorularının cevapları

## Cevap 7.1 — Sorun arayüz/IP/geçitte değil; çözümlemededir

`ping 8.8.8.8` çalışıyorsa, paket makinenden çıkıp bir IP'ye gidip dönebiliyor demektir — yani **arayüz**
(paket çıkabiliyor), **IP** (makinenin adresi var) ve **geçit** (dış ağa yönlendirme çalışıyor) üçü de
sağlamdır. Sorun bu üç parçanın **hiçbirinde değildir**. `ping google.com`'un "Name or service not known"
vermesinin tek farkı, ikincisinin bir **isim→IP çevirisi** gerektirmesidir. Demek ki arıza tam olarak isim
çözümleme katmanındadır. Bu gözlem seni doğrudan 7.2'nin dosyasına yönlendirir: **`/etc/resolv.conf`** (ve
`resolvectl status`) — DNS sunucusu boş, yanlış veya erişilemez.
**İlgili bölüm:** 7.1.1, 7.2.2 · **Devamı:** Cevap 7.2.

## Cevap 7.2 — Çözümleme arızası; /etc/hosts ile geçici kurtarma

(a) Arıza **çözümleme** katmanındadır, ağ yolunda değil: `ping 10.0.5.20` (IP) çalışıyor → veritabanına
giden ağ yolu (arayüz/geçit/SG) sağlam; ama `ping db.prod.internal` (isim) patlıyor → isim IP'ye
çevrilemiyor. Uygulama da aynı ismi kullandığı için bağlanamıyor. (b) `/etc/hosts`'a şu tek satır uygulamayı
geçici olarak çalışır kılar: `10.0.5.20  db.prod.internal` — bu, ismi hiç DNS'e sormadan doğrudan IP'ye
eşler (7.2.1'de `/etc/hosts` DNS'ten önce bakılır). (c) Kalıcı çözüm değildir çünkü: veritabanının IP'si
değişirse (bulutta failover/replace'te olur) `/etc/hosts` eski IP'yi göstermeye devam eder ve arıza sessizce
geri gelir; ayrıca bu satırı her makineye tek tek koyman gerekir. Kalıcı çözüm, asıl DNS/`resolv.conf`
sorununu düzeltmektir.
**İlgili bölüm:** 7.2.1-7.2.2 · **Devamı:** 7.6 arıza tablosu (satır 1).

## Cevap 7.3 — İki taraf, iki farklı izin/kimlik sorunu

**Sunucu tarafı (Faz 2):** Public key sunucuda `~/.ssh/authorized_keys`'e hiç eklenmemiş olabilir, ya da
eklenmiş ama izinler yanlış — sshd, `~/.ssh` `700` ve `authorized_keys` `600` değilse (dizin/dosya "çok
açık"sa) anahtarı **sessizce reddeder**. Doğrulama: sunucuda `ls -ld ~/.ssh` ve `ls -l ~/.ssh/authorized_keys`
(izinler + public key satırın gerçekten orada mı). **İstemci tarafı (7.3.1):** Senin private key'in çok açık
izinlerde olabilir (`600` değil); SSH bunu tehlikeli bulup anahtarı kullanmayı reddeder. Doğrulama:
`ls -l ~/.ssh/id_ed25519` — `-rw-------` (600) değilse `chmod 600 ~/.ssh/id_ed25519`. `ssh -v` her iki
durumda da hangi anahtarın sunulduğunu ve sunucunun neden reddettiğini gösterir.
**İlgili bölüm:** 7.3.1 (Faz 2.3) · **Devamı:** 7.6 arıza tablosu (satır 6).

## Cevap 7.4 — `127.0.0.1`'e bağlı: yeşil `systemctl` üç sorudan yalnızca birincisini cevaplar

(a) **İkinci** soru — "doğru adreste mi dinliyor?" `systemctl` yalnızca birincisini (process çalışıyor
mu) cevaplar. `ss` satırı `127.0.0.1:8080` gösteriyor: yalnızca loopback, yani dışarıdan hiçbir bağlantı
ona ulaşamaz. Bu, mesajı da açıklar: paket makineye **ulaştı** (SG ve `ufw` geçirdi) ama o arayüzde
dinleyen bir şey yoktu; sessizce düşürülmek yerine reddedildi. (b) **Uygulamanın kendi
yapılandırmasındaki bind adresini** değiştir (örneğin `--host 0.0.0.0` ya da `listen` ayarı), servisi
yeniden başlat ve `ss -tulpn` çıktısında artık `0.0.0.0:8080` göründüğünü doğrula. SG'yi ya da `ufw`'yu
daha fazla açmak hiçbir şeyi değiştirmez — zaten açıklar. (c) Servis yalnızca iç kullanım içinse: yerel
process'lerin ya da bir SSH tüneli (7.3.3) üzerinden erişilen bir veritabanı — loopback'te tutmak o
zaman bilinçli bir güvenlik tercihidir.
**İlgili bölüm:** 7.4.2 · **Devamı:** 7.5.1 (diğer iki kapı).

## Cevap 7.5 — İki katman, ikisi de açık olmalı: SG'nin izin verdiğini host firewall kesti

(a) **Host firewall**: `ufw` aktif ve yalnızca 22'yi listeliyor; SG paketin makineye ulaşmasına izin
verse de 80. port makinenin içinde düşürülüyor. Security Group makinenin dışında yaşayan ayrı bir
katmandır ve konsolu yalnızca SG'yi yansıtır — ikisi bağımsızdır ve **ikisi de açık olmalıdır** (7.5.1).
(b) Portu host'ta `sudo ufw allow 80/tcp` ile aç (geri alma: `sudo ufw delete allow 80/tcp`), `sudo ufw
status verbose` ile kontrol et ve **dışarıdan** `curl -I http://<ip>` ile doğrula — makinenin içinden
yapılan bir kontrol SG'den hiç geçmez. (c) `ufw` kapalıysa host katmanı sebep değildir; **Security
Group**'u konsoldan ya da CLI'dan kontrol et: port açık mı, kaynak aralığı doğru mu, instance gerçekten
düşündüğün SG'de mi? Sonra 7.4.3'teki kapı haritasına dön.
**İlgili bölüm:** 7.5.1 · **Devamı:** Faz 9 (defense in depth).

---
---

# Faz 7 — Sık sorulan sorular

**S1 — `ip addr` neden public IP'mi göstermiyor?** Bulutta makinenin içinden sadece **özel** IP görünür;
public IP dışarıda bir NAT katmanında ona eşlenir. Public IP'yi bulut konsolundan veya instance metadata'dan
öğrenirsin, makinenin içinden değil (7.1.1).

**S2 — `ping` çalışıyor ama servise bağlanamıyorum, çelişki değil mi?** Hayır. `ping` (ICMP) ile bir
**servis portuna** TCP bağlantısı ayrı şeylerdir. `ping` makinenin ağda **var** olduğunu gösterir; ama
servis o portu dinlemiyorsa, `0.0.0.0` yerine `127.0.0.1`'e bind'liyse, veya firewall/SG portu kapatıyorsa,
ping'e rağmen bağlanamazsın. `ping`, kapılar zincirinin sadece en dışını test eder (7.4.3).

**S3 — `127.0.0.1` ile `0.0.0.0` arasındaki farkı bir cümlede?** `127.0.0.1` = "sadece makinenin kendi
içinden erişilebilir dinle" (dışarı kapalı); `0.0.0.0` = "tüm arayüzlerde dinle" (firewall/SG izin verirse
dışarıdan erişilebilir). Bir servis dışarıdan erişilecekse `0.0.0.0`'a bind etmeli (7.4.2).

**S4 — SSH anahtarım doğru ama yine reddediliyor / parola soruyor. Neden?** En sık sebep **izinlerdir**:
private key `600`, `~/.ssh` `700` değilse SSH güvenlik gereği sessizce reddeder. Sunucuda `authorized_keys`
izinleri de aynı şekilde katıdır. `ssh -v` ile tam adımı gör, izinleri `ls -l` ile doğrula (7.3.1).

**S5 — Security Group'u makinenin içinden nasıl görürüm?** Göremezsin. Security Group makinenin **dışında**,
bulut ağ katmanında yaşar ve OS'e görünmez. `ufw`/`nftables` sana sadece **host** firewall'unu gösterir. SG'yi
bulut konsolu veya CLI ile kontrol etmelisin — ve bir bağlantı arızasında **ikisini de** ayrı ayrı ele
(7.5.1).

**S6 — DB portunu dışarıya açmadan ona nasıl bağlanırım?** SSH port yönlendirme (tünelleme):
`ssh -L 5432:127.0.0.1:5432 sunucu` ile uzak DB'nin yerel portunu kendi makinene taşırsın; trafik şifreli
SSH tünelinden akar, DB portu internete hiç açılmaz. Bastion/jump host kalıbının temeli budur (7.3.3).

**S7 — SSH mi yoksa SSM Session Manager mı kullanmalıyım?** Mümkünse SSM: 22 portunu internete hiç açmaz
(saldırı yüzeyi ~sıfır), erişimi IAM ile yetkilendirir (anahtar dağıtmak yok), her oturumu loglar. SSH'ı yine
de bil — eski sistemler ve tünelleme için gerekir. Modern kurulumda "SSH'ı sıkılaştır" yerine giderek "SSH'ı
hiç açma" tercih ediliyor (7.3.4).

---
---

# Faz 7 — Kendini sına

Cevaplarını bir kâğıda yaz, sonra cevap anahtarıyla karşılaştır. Hedef: 18 sorudan 14+.

## Bölüm A — Tanım ve mekanizma

1. Bir makinenin ağ kimliğinin üç parçasını (arayüz, IP, geçit) ve her birini gösteren komutu say.
2. `ip addr` neden bulutta genelde public IP'yi göstermez? Public IP nerede yaşar?
3. İsim çözümleme zincirini sırayla yaz: bir isim IP'ye çevrilirken hangi iki yere (hangi sırayla) bakılır?
4. "IP ile ping çalışıyor, isimle çalışmıyor" gözlemi arızayı hangi tek katmana daraltır ve hangi dosyaya
   bakarsın?
5. SSH'ta anahtar tabanlı kimlik neden paroladan güvenlidir? Private ve public key'in her biri nerede durur?
6. `ss -tulpn`'deki `t`, `u`, `l`, `p`, `n` bayraklarının her biri ne yapar?
7. `0.0.0.0:80` ile `127.0.0.1:5432` dinleme adresleri arasındaki fark nedir — hangisi dışarıdan erişilebilir?
8. Host firewall ile Security Group iki ayrı katman derken ne kastediyoruz? Hangisi makinenin içinden görünmez?

## Bölüm B — Uygula ve teşhis et

9. Bir servis `systemctl status` yeşil ama dışarıdan bağlanılamıyor. Erişilebilirlik için sırayla kontrol
   etmen gereken üç ayrı soruyu (ve komutunu) yaz.
10. `ping 8.8.8.8` çalışıyor, `ping github.com` "Name or service not known". Hangi tek dosyaya bakarsın ve
    orada ne ararsın?
11. Yeni bir anahtarla EC2'ye bağlanamıyorsun, "Permission denied (publickey)". İlk çalıştıracağın komut
    nedir, ve kontrol edeceğin iki izin (biri sende, biri sunucuda) hangileri?
12. `ss -tulpn` çıktısında uygulaman `127.0.0.1:8080`'de dinliyor ama dışarıdan erişilmesini istiyorsun. Kök
    sebep nedir ve nerede düzeltilir?
13. Bir DB portunu (5432) internete hiç açmadan yerel bir araçla ona bağlanmak istiyorsun. Hangi komutu
    kullanırsın?
14. `sshd_config`'i sıkılaştıracaksın. Kendini kilitlememek için uygulamadan önce ve sırasında yapman gereken
    iki şey nedir?

## Bölüm C — Muhakeme ve bağlantı

15. "Servise bağlanamıyorum" olayında bağlantının geçmek zorunda olduğu kapıları (dıştan içe) sırala. Her
    kapı hangi bölüme ait?
16. Faz 6'da "cihaz var ama mount yok → `df`'te yok" arızasını gördün. Bunun bu fazdaki tam karşılığı hangi
    arızadır? İki durumun ortak dersini bir cümlede yaz.
17. Bir makine SSH sıkılaştırma hatasıyla erişilemez oldu. Neden `ssh` ile kurtaramazsın, ve Faz 6'daki
    hangi kurtarma mantığının ağ versiyonunu kullanırsın?
18. SSM Session Manager'ın SSH'a göre üç güvenlik avantajını say.

---

## Cevap anahtarı

1. Arayüz (`ip addr` / `ip link`), IP (`ip addr`), varsayılan geçit (`ip route` → `default via`) (7.1.1). —
2. Çünkü içeriden sadece özel IP görünür; public IP dışarıda bir NAT katmanında eşlenir — bulut konsolunda/
   metadata'da yaşar (7.1.1, S1). — 3. Önce `/etc/hosts` (yerel defter), sonra `/etc/resolv.conf`'taki DNS
   sunucusu (7.2.1). — 4. İsim **çözümleme** katmanına; `/etc/resolv.conf` (ve `resolvectl status`) (7.2.2).
   — 5. Parola ağdan geçmez ve brute-force edilemez; private key sende kalır (asla paylaşılmaz), public key
   sunucuda `authorized_keys`'te durur (7.3.1). — 6. `-t` TCP, `-u` UDP, `-l` sadece dinleyenler, `-p`
   process/PID, `-n` sayısal (isim çözmeden) (7.4.1). — 7. `0.0.0.0` tüm arayüzlerde dinler → dışarıdan
   erişilebilir; `127.0.0.1` sadece loopback'te → sadece makinenin içinden (7.4.2). — 8. Host firewall OS'in
   içinde (`ufw`/`nftables`), SG bulut ağ katmanında; **SG makinenin içinden görünmez**, ikisi de trafiği
   kesebilir (7.5.1).

9. (1) Process çalışıyor mu → `systemctl status` (Faz 5); (2) doğru adreste dinliyor mu → `ss -tulpn`
   (`0.0.0.0` vs `127.0.0.1`); (3) firewall/SG izin veriyor mu → `ufw status` + bulut konsolu (7.4.2). — 10.
   `/etc/resolv.conf`; geçerli/erişilebilir bir `nameserver` satırı var mı diye bakarsın (7.2.2). — 11. Önce
   `ssh -v ...` (hangi adımda reddedildi); izinler: sende `~/.ssh/id_*` `600` mü, sunucuda `~/.ssh` `700` /
   `authorized_keys` `600` ve public key orada mı (7.3.1). — 12. Uygulama `127.0.0.1`'e bind etmiş; dışarı
   açmak için uygulamanın config'inde bind adresini `0.0.0.0` yap (7.4.2). — 13. `ssh -L 5432:127.0.0.1:5432
   sunucu` (SSH tüneli) (7.3.3). — 14. `sudo sshd -t` ile sözdizimini doğrula; **ikinci bir açık oturum**
   dururken test et, yeni oturum açabildiğini görmeden mevcut oturumu kapatma (7.3.4).

15. (0) İsim çözülüyor mu (7.2) → (1) Security Group açık mı (7.5) → (2) host firewall açık mı (7.5) → (3)
    servis `0.0.0.0`'a mı bind (7.4.2) → (4) process ayakta mı (Faz 3/5) (7.4.3). — 16. Tam karşılığı: "servis
    çalışıyor / port dinleniyor ama `127.0.0.1`'e bind veya firewall kapalı → dışarıdan erişilemiyor." Ortak
    ders: bir kaynağın **var olması** (mount'lu / process dinliyor) ile **erişilebilir olması** (df'te /
    dışarıdan ulaşılabilir) ayrı şeylerdir (7.4.2, Faz 6.6). — 17. Çünkü ağ/SSH katmanı çöktüğünde `ssh`
    zaten çalışmaz; Faz 6'daki "erişilemez makineye seri/bulut konsolundan gir" kurtarma mantığının ağ
    versiyonunu kullanırsın (7.3.4, 7.6). — 18. (1) 22 portunu internete hiç açmaz (saldırı yüzeyi ~sıfır);
    (2) erişimi IAM ile yetkilendirir (anahtar dağıtmak/döndürmek yok); (3) her oturumu loglar (denetim)
    (7.3.4).

## Puanlama

| Doğru sayısı | Ne anlama geliyor |
|---|---|
| 16-18 | "Bağlanamıyorum"u katman katman elemeyi öğrendin. Faz 8'e hazırsın. |
| 13-15 | İyi. Kaçırdığın soruların bölümlerini tekrar oku (özellikle 7.4.2 ve 7.5). |
| 9-12 | Temel var ama kırılgan. 7.2 (çözümleme) ve 7.4 (bind adresi) üzerine çalış. |
| 0-8 | Fazı yeniden gez; `ip addr`/`ss -tulpn`/`ssh -v` komutlarını bir test makinesinde bizzat çalıştır. |

Kaçırdığın soru → dönmen gereken bölüm:

| Soru | Bölüm |
|---|---|
| 1, 2 | 7.1 Arayüz ve adres |
| 3, 4, 10 | 7.2 İsim çözümleme |
| 5, 11, 14 | 7.3 SSH |
| 6, 7, 9, 12 | 7.4 Port ve bind adresi |
| 8, 15 | 7.5 Firewall / SG |
| 13 | 7.3.3 Tünelleme |
| 16, 17 | 7.4.3 + Faz 6 bağlantısı |
| 18 | 7.3.4 SSM |

---
---

# Faz 7 — Kapanış ve Faz 8'e Köprü

## Bu fazdan ne taşıyorsun

Faz 7 sana **erişilebilirlik refleksini** verdi. Artık "bağlanamıyorum" dediğinde paniklemek yerine kapılar
zincirini sırayla eliyorsun: isim çözülüyor mu (`resolv.conf`) → firewall katmanları açık mı (host `ufw` +
bulut SG) → servis doğru adrese mi bind (`ss -tulpn`: `0.0.0.0` vs `127.0.0.1`) → process ayakta mı
(`systemctl`). İki ayrımı kalıcı olarak edindin: **IP vs isim** (ağ yolu mu çözümleme mi) ve **127.0.0.1 vs
0.0.0.0** (yerel mi dışa açık mı). SSH'ı da sadece bir komut değil, güvenli yapılandırılması gereken bir
servis olarak görüyorsun — ve bulutta çoğu zaman "SSH'ı sıkılaştır" yerine "SSM ile SSH'ı hiç açma" yolunun
tercih edildiğini biliyorsun.

## Faz 8 bunun neresine bağlanıyor

Faz 5'te servislerin systemd tarafından yönetildiğini, Faz 7'de o servislerin ağa nasıl açıldığını gördün.
Ama o servisler makineye **nasıl geliyor**? Faz 8 tam bunu anlatıyor: paket yöneticileri (`apt`), yazılımın
kurulması, ve kendi uygulamanı bir systemd servisi hâline getirmek (servisleştirme). Bir "servise
bağlanamıyorum" olayının en dıştaki kapısı "servis kurulu ve çalışıyor mu"dur — ve o servisin oraya nasıl
geldiği (paket mi, kendi binary'in mi, hangi sürüm) Faz 8'in konusudur. Ağ (Faz 7) servisi **dışarıya**
bağlar; paketler (Faz 8) servisi **makineye** getirir. İkisi birleşince, bulutta bir uygulamayı sıfırdan
ayağa kaldırıp dış dünyaya güvenle açmanın tüm zinciri tamamlanır.

> **🤔 Faz çıktısı — kendine sor:** Faz 7'de bir servisin "çalışması" ile "erişilebilir olması"nı ayırdın.
> Faz 8'e geçmeden düşün: bir uygulamayı `apt install` ile kurdun ama `systemctl status` "not found" diyor.
> Bu, Faz 7'deki hangi ayrımın bir üst basamağı — "kurulu olmak" ile "bir servis olarak çalışıyor olmak"
> arasında hangi adım eksik olabilir? (İpucu: her paket otomatik bir systemd servisi tanımlamaz.)
>
> **🧪 Lab 7 fikri (kendi test instance'ında):** (1) `ip addr` ve `ip route` ile ağ kimliğini oku, özel
> IP'ni bul. (2) `ss -tulpn` ile dinlenen portları ve arkalarındaki process'leri gör. (3) `cat /etc/resolv.conf`
> ile DNS sunucunu bul; `ping 8.8.8.8` ve `ping google.com` farkını yaşa. (4) Basit bir web sunucusunu
> (örn. `python3 -m http.server`) önce `127.0.0.1`'e, sonra `0.0.0.0`'a bind edip `ss` çıktısındaki farkı
> ve dışarıdan erişilebilirliği gör. (5) `ssh -v` ile bir bağlantıyı adım adım izle — el sıkışmanın nerede
> tamamlandığını gör. Bu beş adım, bu fazın erişilebilirlik refleksini elinde toplar.

---

> **Navigasyon:** [◀ Ara Sınav 3](Ara_Sinav_3.md) · **Faz 7** · [Faz 8 — Paketler, Yazılım ve Servisleştirme ▶](Faz_8_Paketler_ve_Servislestirme.md)



