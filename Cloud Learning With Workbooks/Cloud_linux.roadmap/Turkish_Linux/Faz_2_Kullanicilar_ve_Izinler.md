# Faz 2 — Kullanıcılar, İzinler ve Kimlik: Erişim Modeli

> **Navigasyon:** [◀ Faz 1 — Shell ve Dosya Sistemi](Faz_1_Shell_ve_Dosya_Sistemi.md) · **Faz 2** · [Ara Sınav 1 ▶](Ara_Sinav_1.md)

---

## Nereden geliyoruz

Faz 0'da çekirdeğin **tek kapısını** (syscall) kurduk. Faz 1'de o kapıya shell üzerinden
nasıl gittiğimizi, dosyaların hangi dizinde durduğunu ve bir logdan nasıl cevap
çıkaracağımızı öğrendik.

Ama o fazda birkaç kez bir duvara çarptık ve üstünden geçtik:

- `cat /proc/1/environ` → **Permission denied**
- `grep` çıkış kodu 2 → "dosya var ama okuyamıyorum"
- `sudo echo "x" > /etc/dosya` → root olmama rağmen **Permission denied**
- `ls -l /` çıktısında `drwx------` ve `drwxrwxrwt` gibi anlamını bilmediğimiz dizeler

Bu fazda o duvarın kendisini inceliyoruz. Faz 0'daki modele bir parça ekliyoruz: çekirdek bir
syscall'ı yerine getirmeden önce **"bunu isteyen kim?"** diye sorar. Bu fazın tamamı bu sorunun
cevabıdır.

Faz 0 ve Faz 1'den üç şey burada doğrudan işe yarayacak:

- **"Her istek bir syscall'dır"**. İzin kontrolü, çekirdeğin `open` veya `execve` isteğini
  karşıladığı anda yapılır.
- **"Her process'in bir kimliği vardır"**. `/proc/<PID>/status` dosyasında gördüğün `Uid` satırı
  bu fazın merkezindedir.
- **"Dizin = isim → inode tablosu"**. Faz 1'de `mv`'nin neden anında bittiğini açıklayan bu fikir,
  dizin izinlerinin neden garip davrandığını da açıklar.

## Bu fazın sorusu

Cloud'da en sık duyacağın iki kelime şunlardır:

> *"Permission denied."*

Nginx 403 döndürüyor. Deploy script'i `Permission denied` diyor. SSH `Permission denied
(publickey)` ile kapıyı kapatıyor. `docker ps` çalışmıyor. IAM'de yönetici olmana rağmen sunucuda
bir dosyayı okuyamıyorsun.

Bu fazın sonunda bu mesajı gördüğünde "`chmod 777` deneyeyim" yerine **hangi kontrolün
reddettiğini sırayla bulabileceksin**: kim olarak çalışıyorum → yoldaki dizinler → dosyanın
bitleri → grup üyeliğim → mount seçenekleri → güvenlik modülleri.

---

## Bu fazın sonunda

- Çekirdeğin kullanıcıları **isimle değil sayıyla** (UID/GID) tanıdığını ve bunun bir diski başka
  bir sunucuya taktığında neden sorun çıkardığını açıklayabileceksin
- `/etc/passwd`, `/etc/shadow` ve `/etc/group` dosyalarını alan alan okuyabileceksin
- `-rwxr-x---` gibi bir izin dizesini ve `750` gibi octal bir sayıyı anında birbirine
  çevirebileceksin
- Çekirdeğin izin kontrolünü **hangi sırayla** yaptığını ve "sahip gruptan daha az yetkili"
  durumunun nasıl mümkün olduğunu bileceksin
- Bir **dizinde** `r`, `w` ve `x` bitlerinin dosyadakinden farklı ne anlama geldiğini
  açıklayabileceksin
- setuid, setgid ve sticky bit'in hangi problemi çözdüğünü ve yanlış setuid'in neden bir yetki
  yükseltme açığı olduğunu bileceksin
- `sudo`'nun gerçekte ne yaptığını, sudoers kuralını okumayı ve "sadece bir komuta sudo izni
  vermenin" neden tehlikeli olabileceğini bileceksin
- EC2'de SSH key'inin sunucuya nasıl yerleştiğini ve OS izinlerinin IAM'den neden **ayrı bir
  katman** olduğunu açıklayabileceksin
- "Permission denied" gördüğünde sırayla hangi kontrolleri yapacağını bileceksin

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden burada |
|---|---|---|---|
| 2.1 | Kullanıcı ve grup: UID, GID, root, kimlik dosyaları | `[kavram]` + `[mekanizma]` | Çekirdeğin "kim?" sorusuna verdiği cevap |
| 2.2 | İzin bitleri: rwx, octal, dizin izinleri | `[mekanizma]` + `[uygulama]` | **Fazın kalbi** — her izin kararı buradan geçer |
| 2.3 | Özel bitler: setuid, setgid, sticky | `[kavram]` + `[uygulama]` | Normal kullanıcının parolasını nasıl değiştirebildiği; ve bir açığın anatomisi |
| 2.4 | `sudo` ve sudoers; uygulamayı root çalıştırmamak | `[mekanizma]` + `[uygulama]` | Yetkiyi kontrollü yükseltmek |
| 2.5 | OS kimliği ve cloud kimliği; "Permission denied" karar ağacı | `[uygulama]` + `[kavram]` | EC2'de gerçek hayat |
| 2.6 | Bu faz bozulunca | — | İzin arızalarının imzaları |

> **Bu fazda nasıl çalışmalı:** Bu fazdaki komutların çoğu 🟢 (sadece okur) veya 🟡 (kendi ev
> dizininde geçici dosya oluşturur). Birkaç kutu 🔴 işaretli: kullanıcı oluşturur veya sudoers'a
> kural ekler. Bu kutuları **sadece kendine ait bir test makinesinde** (VM, WSL veya bir EC2
> t3.micro) çalıştır ve her birinin sonundaki geri alma adımını atlama. Özellikle sudoers
> dosyasındaki bir yazım hatası, parolası olmayan bir EC2 sunucusunda seni **kalıcı olarak
> yetkisiz** bırakabilir (2.4.1).

---
---

# 2.1 Kullanıcı ve Grup

## 2.1.1 Kimlik bir sayıdır: UID, GID ve root `[kavram]`

`ls -l` yazdığında dosya sahibinin adını görürsün:

```
$ ls -l /etc/hostname
-rw-r--r-- 1 root root 16 Sep 15 08:02 /etc/hostname
```

Ama çekirdek `root` kelimesini **bilmez**. Diskteki inode'da sahip olarak bir **sayı** yazılıdır.
Çekirdeğin çalışan her process için tuttuğu kimlik de sayılardan oluşur:

| Parça | Açılımı | Ne tutar |
|---|---|---|
| **UID** | User ID | Kullanıcının numarası |
| **GID** | Group ID | Birincil (primary) grubunun numarası |
| **Ek gruplar** | Supplementary groups | Üyesi olduğu diğer grupların numaraları |

`root`, `ubuntu` ve `www-data` gibi isimler **insanlar için** vardır. `ls` her dosyanın UID'sini
alır, `/etc/passwd`'de o sayıya karşılık gelen ismi bulur ve ekrana ismi basar. Bu çeviri
tamamen user space'te, `ls` programının içinde yapılır.

**Numara aralıkları (Ubuntu ve Amazon Linux):**

| UID | Kim | Örnek |
|---|---|---|
| **0** | root — süper kullanıcı | `root` |
| **1–999** | Sistem hesapları: servisler için, insanlar için değil | `daemon` (1), `www-data` (33), `syslog`, `sshd` |
| **1000+** | İnsan kullanıcılar | EC2'de `ubuntu` veya `ec2-user` = 1000 |
| **65534** | `nobody` — "hiç kimse" | NFS'te tanınmayan kullanıcı, kısıtlı servisler |

**root neden özeldir?** İsmi yüzünden değil, **numarası 0 olduğu için.** Çekirdek bir izin
kontrolü yaparken process'in UID'si 0 ise dosya izinlerinin çoğunu atlar. `/etc/passwd`'ye
`yedek:x:0:0:...` diye bir satır eklenirse sistemde **iki root** olur; adının `root` olmaması
hiçbir şeyi değiştirmez. (Teknik ayrıntı: modern çekirdekte root'un gücü "capabilities" denen
parçalara bölünmüştür. Bunu Faz 9'da göreceğiz; bu faz için "UID 0 = kontrollerin çoğu atlanır"
modeli yeterli.)

> **🔧 Makinende gör** 🟢 — Kimliğin
>
> ```
> $ id
> uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),24(cdrom),27(sudo),30(dip),110(lxd)
> $ grep -E '^(Uid|Gid|Groups)' /proc/self/status
> Uid:	1000	1000	1000	1000
> Gid:	1000	1000	1000	1000
> Groups:	4 24 27 30 110 1000
> ```
>
> - `id` sayıları **ve** isimleri gösterir; isimleri `/etc/passwd` ile `/etc/group`'tan çevirir.
> - `/proc/self/status` çekirdeğin gerçekte tuttuğunu gösterir: **sadece sayılar.** (`self`,
>   dosyayı okuyan process'in kendisi, burada `grep`.)
> - `Uid` satırında dört sayı var: **real, effective, saved, filesystem.** Şimdilik hepsi aynı.
>   İzin kontrolünde kullanılan **effective** UID'dir (ikinci sütun). Bu dört sayının neden
>   ayrıldığını 2.3.1'de setuid ile göreceğiz.
> - `27(sudo)`: bu kullanıcı `sudo` grubunda. Ubuntu'da `sudo` kullanabilmenin sebebi budur
>   (2.4.1).

**Process kimliği nereden gelir?** Kimlik kalıtımla geçer. Faz 1'deki `fork` process'in bir
kopyasını çıkarır ve kopya UID, GID ve ek grupları da **aynen alır.** SSH ile girdiğinde `sshd`
root olarak çalışır. Parolanı veya key'ini doğruladıktan sonra oturum için açtığı process'in
kimliğini **bir kez** seninkine çevirir, sonra shell'ini başlatır. O andan sonra başlattığın her
komut o shell'in çocuğudur ve aynı kimliği taşır.

Bu küçük detayın önemli bir sonucu var: **bir kullanıcıyı bir gruba eklemek, zaten çalışan
process'lerin kimliğini değiştirmez.** Bunu Düşün 2.1'de göreceksin.

> **⚠️ Yaygın yanılgı: "root bir kullanıcı adıdır; adı değiştirilirse veya başka bir adla
> girilirse root yetkisi olmaz."**
>
> Çekirdek sadece sayıya bakar. UID'si 0 olan her hesap root'tur. Bu yüzden bir sunucunun
> güvenlik denetiminde ilk bakılan şeylerden biri şudur:
>
> ```
> $ awk -F: '$3 == 0' /etc/passwd
> root:x:0:0:root:/root:/bin/bash
> ```
>
> Bu komut **tek satır** döndürmelidir. İkinci bir satır varsa ya çok eski bir yapılandırma
> ya da bir saldırganın bıraktığı arka kapıdır.

**❓ Akla gelen soru: Neden `www-data`, `mysql`, `syslog` gibi ayrı kullanıcılar var? Hepsi root
olarak çalışsa daha basit olmaz mıydı?**

Daha basit olurdu, ama her servis diğerlerinin ve sistemin tamamının dosyalarına erişebilirdi.
Ayrı kullanıcılar bir **bölme duvarı** kurar. Nginx'te bir açık bulunursa saldırgan `www-data`
olur ve sadece `www-data`'nın okuyabildiği dosyaları görür. Veritabanının dosyalarına veya
`/etc/shadow`'a erişemez. Bu fikrin adı **en az yetki prensibi**dir (least privilege). 2.4.2'de
ve Faz 9'da ona döneceğiz. Sistem hesaplarının çoğunun shell'i `/usr/sbin/nologin`'dir, yani bu
hesaplarla giriş yapılamaz. Bunlar sadece servislerin "kimlik kartı"dır.

---

## 2.1.2 Kimlik nerede tutulur: `/etc/passwd`, `/etc/shadow`, `/etc/group` `[mekanizma]`

Faz 1'de FHS'yi öğrenirken "config `/etc`'de" demiştik. Kimlik de bir config'dir ve üç düz metin
dosyasında durur.

### `/etc/passwd` — kullanıcı listesi

Her satır bir kullanıcıdır; alanlar `:` ile ayrılır:

```
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
│      │ │    │    │      │            └─ 7. giriş shell'i
│      │ │    │    │      └────────────── 6. ev dizini
│      │ │    │    └───────────────────── 5. GECOS (açıklama, tam ad)
│      │ │    └────────────────────────── 4. birincil GID
│      │ └─────────────────────────────── 3. UID
│      └───────────────────────────────── 2. parola alanı: "x" = shadow'a bak
└──────────────────────────────────────── 1. kullanıcı adı
```

İsminin aksine bu dosyada **parola yoktur.** 2. alandaki `x`, "parola `/etc/shadow`'da" demektir.
Eskiden parolanın hash'i gerçekten burada dururdu. Sorun şuydu: `/etc/passwd` **herkes tarafından
okunabilir olmak zorundadır**, çünkü `ls -l`'den `ps`'e kadar her program UID'yi isme çevirmek
için bu dosyayı okur. Herkesin okuyabildiği bir dosyada duran hash'ler çevrimdışı kırılabiliyordu.
Çözüm hash'leri ayrı ve korumalı bir dosyaya taşımak oldu.

### `/etc/shadow` — parola hash'leri

```
ubuntu:!:19980:0:99999:7:::
│      │ │     │ │     └─ 6. kaç gün önceden uyar
│      │ │     │ └─────── 5. en fazla kaç günde bir değişmeli
│      │ │     └───────── 4. en az kaç gün sonra değiştirilebilir
│      │ └─────────────── 3. son değişim (1 Ocak 1970'ten beri gün)
│      └───────────────── 2. parola hash'i
└──────────────────────── 1. kullanıcı adı
```

2. alanın değerleri:

| Değer | Anlamı |
|---|---|
| `$y$j9T$...` | yescrypt hash'i (Ubuntu 22.04 ve sonrası) |
| `$6$...` | SHA-512 hash'i (Amazon Linux 2023, RHEL ailesi, eski Ubuntu) |
| `!` veya `!$y$...` | Hesap **parola ile kilitli**. Parolayla giriş olmaz, ama SSH key ile olabilir |
| `*` | Parola hiç tanımlanmamış (sistem hesapları) |

EC2'deki `ubuntu` kullanıcısının parola alanı `!`'dir: **parolası yoktur.** Tek giriş yolu SSH
key'idir (2.5.1). `sudo`'nun ondan parola sormamasının sebebi de budur (2.4.1).

### `/etc/group` — gruplar ve üyeleri

```
sudo:x:27:ubuntu
docker:x:988:ubuntu,deploy
│      │ │   └─ 4. ek üyeler (virgülle)
│      │ └───── 3. GID
│      └─────── 2. grup parolası (kullanılmaz)
└────────────── 1. grup adı
```

Bir kullanıcının **birincil grubu** `/etc/passwd`'nin 4. alanındadır ve genellikle `/etc/group`'ta
üye olarak **yazmaz**. Ubuntu her kullanıcı için aynı adda bir grup oluşturur (`ubuntu` kullanıcısı
→ `ubuntu` grubu, GID 1000). **Ek grupları** ise `/etc/group`'taki üye listesindedir.

> **🔧 Makinende gör** 🟢 — Neden `/etc/shadow` okunamıyor?
>
> ```
> $ ls -l /etc/passwd /etc/shadow /etc/group
> -rw-r--r-- 1 root root   1322 Aug 30 21:59 /etc/group
> -rw-r--r-- 1 root root   3097 Aug 30 21:59 /etc/passwd
> -rw-r----- 1 root shadow 1386 Aug 30 21:59 /etc/shadow
> $ cat /etc/shadow
> cat: /etc/shadow: Permission denied
> $ stat /etc/shadow
>   File: /etc/shadow
>   Size: 1386      	Blocks: 8          IO Block: 4096   regular file
> Device: 259,2	Inode: 1575655     Links: 1
> Access: (0640/-rw-r-----)  Uid: (    0/    root)   Gid: (   42/  shadow)
> ...
> ```
>
> - `passwd` ve `group`: `-rw-r--r--` → herkes okur, sadece root yazar.
> - `shadow`: `-rw-r-----` → root okur ve yazar, `shadow` grubu okur, **diğerleri hiçbir şey
>   yapamaz.** Sen `shadow` grubunda değilsin; bu yüzden `cat` reddedildi.
> - `stat` aynı bilgiyi hem sayıyla (`0640`, `Uid: 0`, `Gid: 42`) hem isimle gösterir. İzin
>   dizesini okumayı 2.2.1'de adım adım yapacağız.

> **🔧 Makinende gör** 🟢 — Kullanıcıları ve grupları listelemek
>
> ```
> $ getent passwd ubuntu
> ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
> $ awk -F: '$3 >= 1000 && $3 < 65534 {print $1, $3, $7}' /etc/passwd
> ubuntu 1000 /bin/bash
> $ getent group sudo
> sudo:x:27:ubuntu
> $ groups ubuntu
> ubuntu : ubuntu adm cdrom sudo dip lxd
> ```
>
> - Faz 1'deki `awk -F:` burada işe yarıyor: 3. alanı (UID) 1000 ile 65534 arasında olan insan
>   kullanıcıları seçtik.
> - **`cat /etc/passwd` yerine `getent passwd` kullan.** Kurumsal ortamlarda kullanıcılar LDAP
>   veya Active Directory'den (SSSD üzerinden) gelebilir ve `/etc/passwd`'de hiç yazmaz.
>   `getent` sistemin gerçekte kullandığı bütün kaynaklara sorar (`/etc/nsswitch.conf`). `cat`
>   sadece dosyayı okur.

**Kullanıcı ve grup yönetimi komutları:**

| Komut | Ne yapar |
|---|---|
| `useradd -m -s /bin/bash ali` | Kullanıcı oluşturur (`-m` = ev dizini, `-s` = shell) |
| `adduser ali` | Debian/Ubuntu'da etkileşimli, daha dostça bir sarmalayıcı |
| `passwd ali` | Parola koyar veya değiştirir |
| `usermod -aG docker ali` | Ek gruba **ekler** (`-a` = append) |
| `userdel -r ali` | Kullanıcıyı ve ev dizinini siler |
| `groupadd devops` | Grup oluşturur |
| `gpasswd -d ali docker` | Kullanıcıyı gruptan çıkarır |

Bu dosyaları elle düzenleme; yukarıdaki komutları kullan. Komutlar dosyayı kilitler, dört dosyayı
(`passwd`, `shadow`, `group`, `gshadow`) tutarlı tutar ve yazım hatasına izin vermez. Elle
düzenlemen gerekiyorsa `vipw` ve `vigr` kullan.

> **⚠️ Yaygın yanılgı: "`usermod -G docker ali` kullanıcıyı docker grubuna ekler."**
>
> `-a` olmadan `-G`, kullanıcının ek gruplarını verdiğin listeyle **değiştirir.** `ali`'nin
> `sudo` grubundaki üyeliği sessizce silinir. Bu hatayı kendi kullanıcına yaparsan ve makinede
> başka yönetici yoksa, sudo yetkini kendi elinle kaybetmiş olursun. Doğrusu: **`usermod -aG`**.

> **🔧 Makinende gör** 🔴 — Kullanıcı oluştur, incele, sil
>
> *Sadece test makinesinde.*
>
> ```
> $ sudo useradd -m -s /bin/bash deneme
> $ getent passwd deneme
> deneme:x:1001:1001::/home/deneme:/bin/bash
> $ sudo grep deneme /etc/shadow
> deneme:!:20711:0:99999:7:::
> $ ls -ld /home/deneme
> drwxr-x--- 2 deneme deneme 4096 Sep 15 09:10 /home/deneme
> $ sudo -u deneme id
> uid=1001(deneme) gid=1001(deneme) groups=1001(deneme)
> ```
>
> - Yeni kullanıcı bir sonraki boş UID'yi (1001) ve aynı adda bir grubu aldı.
> - Parola alanı `!`: parola koymadığımız için hesap kilitli.
> - Ev dizini `drwxr-x---`: Ubuntu 21.04'ten beri ev dizinleri başka kullanıcılara kapalıdır
>   (`/etc/login.defs` → `HOME_MODE 0750`). Bu detay Düşün 2.3'te önemli olacak.
> - `sudo -u deneme id`: bir komutu başka bir kullanıcı olarak çalıştırdık (2.4.1).
>
> **Geri al:**
>
> ```
> $ sudo userdel -r deneme
> $ getent passwd deneme || echo "silindi"
> silindi
> ```

> **🤔 Düşün 2.1**
> Bir EC2 sunucusunda `ubuntu` kullanıcısıyla çalışıyorsun. Docker kurdun ve kendini gruba
> ekledin:
>
> ```
> $ sudo usermod -aG docker ubuntu
> $ getent group docker
> docker:x:988:ubuntu
> $ docker ps
> permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
> ```
>
> `/etc/group` dosyası seni üye olarak gösteriyor, ama `docker ps` hâlâ reddediliyor.
>
> 1. Neden? `id` komutunun çıktısında ne görmeyi beklersin?
> 2. Sorunu çözmenin en az iki yolunu söyle.
> 3. SSH bağlantını kapatıp açtın ama `tmux` oturumuna geri bağlandın ve sorun devam ediyor.
>    Neden?
>
> *(Cevap: fazın sonunda)*

> **🤔 Düşün 2.2**
> Eski bir sunucudan (Amazon Linux) bir EBS diskini ayırıp yeni bir Ubuntu sunucusuna taktın.
> Eski sunucuda uygulama `appuser` (UID 1001) kullanıcısıyla çalışıyordu. Yeni sunucuda önce
> `deploy` adında bir kullanıcı oluşturdun, sonra `appuser`'ı oluşturdun. Diski `/data`'ya
> bağladın:
>
> ```
> $ ls -l /data/app
> -rw-r----- 1 deploy deploy 5120 Sep 10 14:22 settings.yml
> drwx------ 2 deploy deploy 4096 Sep 10 14:22 uploads
> ```
>
> Dosyaları `deploy` kullanıcısı hiç oluşturmadı. Uygulama `appuser` olarak başlatılınca
> `Permission denied` alıyor.
>
> 1. Dosyalar neden `deploy`'a ait görünüyor?
> 2. İki farklı çözüm öner ve hangisini ne zaman seçeceğini söyle.
>
> *(Cevap: fazın sonunda)*

---
---

# 2.2 İzin Bitleri

## 2.2.1 rwx × sahip, grup, diğer `[mekanizma]`

`ls -l` çıktısındaki her satır, çekirdeğin o dosya için bildiği izin bilgisinin tamamını taşır:

```
-rwxr-x--- 1 root devops 1832 Sep 15 09:12 deploy.sh
│└┬┘└┬┘└┬┘ │ │    │      │    │            └─ isim
│ │  │  │  │ │    │      │    └────────────── son değişim zamanı
│ │  │  │  │ │    │      └─────────────────── boyut (byte)
│ │  │  │  │ │    └────────────────────────── grup
│ │  │  │  │ └─────────────────────────────── sahip (owner)
│ │  │  │  └───────────────────────────────── hard link sayısı
│ │  │  └──────────────────────────────────── diğerleri (other)
│ │  └─────────────────────────────────────── grup (group)
│ └────────────────────────────────────────── sahip (user/owner)
└──────────────────────────────────────────── dosya türü
```

**İlk karakter: dosya türü.** Faz 0'daki "her şey dosyadır" fikri burada görünür:

| Karakter | Tür | Örnek |
|---|---|---|
| `-` | Normal dosya | `/etc/hostname` |
| `d` | Dizin | `/etc` |
| `l` | Sembolik link | `/bin -> usr/bin` |
| `c` | Karakter cihazı | `/dev/null` |
| `b` | Blok cihazı | `/dev/nvme0n1` |
| `s` | Socket | `/run/systemd/journal/stdout` |
| `p` | Named pipe (FIFO) | `mkfifo` ile oluşturulan |

**Sonraki dokuz karakter: üç sınıf × üç izin.** Her üçlü sırasıyla `r`, `w`, `x`'tir. Harf yerine
`-` varsa o izin **yoktur.**

| Harf | Dosyada anlamı |
|---|---|
| `r` (read) | İçeriği okuyabilir (`open` ile okuma modunda açabilir) |
| `w` (write) | İçeriği değiştirebilir |
| `x` (execute) | Program olarak çalıştırabilir (`execve`) |

Örneğimizde `rwx` sahip (`root`), `r-x` grup (`devops`) ve `---` diğerleri içindir.

![Şekil 2.1 — İzin dizesinin anatomisi](../diagrams/png/lx-2-01-mode-bits.png)
*Şekil 2.1 — `-rwxr-x---` dizesi bir tür karakteri ve üç üçlüye ayrılır. Her üçlüde r = 4, w = 2,
x = 1; üçlünün toplamı octal gösterimin bir basamağıdır: 7, 5, 0 → `750`.*

### Octal gösterim

Her üçlü üç bitten oluşur, yani 0 ile 7 arasında tek bir sayıyla yazılabilir. Bitlerin
değerleri: **r = 4, w = 2, x = 1.** Açık olan bitleri toplarsın:

```
rwx = 4+2+1 = 7        r-x = 4+0+1 = 5        --- = 0
-rwxr-x---  →  750
```

En sık göreceğin değerler:

| Octal | Dize | Tipik kullanım |
|---|---|---|
| `755` | `rwxr-xr-x` | Programlar, script'ler, herkese açık dizinler |
| `644` | `rw-r--r--` | Normal config ve içerik dosyaları |
| `750` | `rwxr-x---` | Sadece bir grubun erişmesi gereken dizin veya script |
| `640` | `rw-r-----` | Grubun okuyacağı hassas dosya (`/etc/shadow`, `auth.log`) |
| `700` | `rwx------` | Kişisel dizin (`~/.ssh`) |
| `600` | `rw-------` | Özel anahtar, parola içeren config (`~/.ssh/id_ed25519`) |
| `400` | `r--------` | Sadece sahibinin okuyacağı, değişmemesi gereken dosya (AWS'nin indirdiğin `.pem` için önerdiği) |
| `777` | `rwxrwxrwx` | **Neredeyse hiçbir zaman doğru cevap değil** (2.2.2) |

> **🔧 Makinende gör** 🟢 — İzinleri sayı ve dize olarak okumak
>
> ```
> $ stat -c '%a %A %U:%G %n' /etc/passwd /etc/shadow /usr/bin/ls ~/.ssh ~
> 644 -rw-r--r-- root:root /etc/passwd
> 640 -rw-r----- root:shadow /etc/shadow
> 755 -rwxr-xr-x root:root /usr/bin/ls
> 700 drwx------ ubuntu:ubuntu /home/ubuntu/.ssh
> 750 drwxr-x--- ubuntu:ubuntu /home/ubuntu
> ```
>
> - `stat -c` biçim dizesiyle istediğin alanları seçer: `%a` octal, `%A` dize, `%U:%G` sahip ve
>   grup.
> - Her satırda dizeyi kafandan sayıya çevir, sonra ilk sütunla karşılaştır. Bu çeviriyi
>   düşünmeden yapabilene kadar tekrarla; bu fazın geri kalanı bunun üstüne kurulu.

### Çekirdek hangi üçlüye bakar?

Bu, fazın en önemli ve en çok yanlış bilinen kuralıdır. Bir process bir dosyayı açmak
istediğinde çekirdek **tek bir üçlü seçer** ve sadece ona bakar:

1. Process'in effective UID'si **0** mı? → root. Okuma ve yazma izin bitlerinden bağımsız olarak
   verilir. (Çalıştırma için dosyada **en az bir** `x` biti olmalıdır.)
2. Process'in UID'si dosyanın **sahibi** mi? → **sadece sahip üçlüsü** kullanılır.
3. Process'in GID'si veya ek gruplarından biri dosyanın **grubu** mu? → **sadece grup üçlüsü**
   kullanılır.
4. Hiçbiri değilse → **diğerleri üçlüsü** kullanılır.

İstenen izin seçilen üçlüde varsa işlem yapılır; yoksa syscall `EACCES` hatasıyla döner ve shell
bunu `Permission denied` olarak basar. **İzinler toplanmaz ve bir sınıftan diğerine geçilmez.**
Sahip üçlüsü reddediyorsa, grup üçlüsü `rwx` olsa bile sonuç reddir.

![Şekil 2.2 — Çekirdek hangi izin üçlüsünü kullanır](../diagrams/png/lx-2-02-permission-check.png)
*Şekil 2.2 — Kontrol yukarıdan aşağı ilerler ve ilk "evet" cevabında durur. Seçilen üçlü dışındaki
bitlere hiç bakılmaz; bu yüzden sahip, grubundan veya diğerlerinden daha az yetkili olabilir.*

(Teknik not: çekirdek kodunda root kontrolü aslında sona yakın, "capability" kontrolü olarak
yapılır. Sonuç bu dört adımlı modelle aynıdır; ayrıntısı Faz 9'da.)

> **🔧 Makinende gör** 🟡 — Sahibin gruptan daha az yetkili olması
>
> ```
> $ echo gizli > sahip-testi
> $ chmod 070 sahip-testi
> $ ls -l sahip-testi
> ----rwx--- 1 ubuntu ubuntu 6 Sep 15 09:20 sahip-testi
> $ id -gn
> ubuntu
> $ cat sahip-testi
> cat: sahip-testi: Permission denied
> ```
>
> - Grup `ubuntu`, sen de `ubuntu` grubundasın ve grup üçlüsü `rwx`. Ama çekirdek önce "sahip
>   mi?" diye sordu, cevap **evet**, sahip üçlüsü `---` → **ret.** Grup üçlüsüne hiç bakmadı.
> - Gerçek hayatta bu durum nadiren kasıtlıdır. Ama bir `chmod` hatasının neden "garip"
>   göründüğünü anlamanı sağlar.
>
> **Geri al:** `rm sahip-testi` (dosyayı silmek için dosyaya değil, **dizine** yazma izni gerekir;
> 2.2.3'te göreceğiz).

> **⚠️ Yaygın yanılgı: "Root her şeyi yapabilir; `x` biti olmayan bir script'i de çalıştırır."**
>
> Root okuma ve yazma izinlerini atlar, ama çalıştırmada bir istisna vardır: dosyanın **üç
> üçlüsünden en az birinde** `x` biti olmalıdır. `-rw-r--r--` bir script'i root da `./script.sh`
> ile çalıştıramaz ve `Permission denied` alır. Bu kuralın amacı "çalıştırılabilir değil"
> işaretli bir dosyanın kazara program gibi çalıştırılmasını önlemektir. `bash script.sh`
> yazmak ise her kullanıcı için işe yarar, çünkü o durumda çalıştırılan program `bash`'tir;
> script sadece **okunan** bir dosyadır.

**❓ Akla gelen soru: Bir script için `x` yeterli mi, yoksa `r` de gerekiyor mu?**

Derlenmiş bir program (`/usr/bin/ls` gibi) için `x` yeterlidir; çekirdek dosyayı kendisi belleğe
yükler. Script için **`r` de gerekir.** `./script.sh` yazdığında çekirdek dosyanın ilk satırındaki
`#!/bin/bash`'i görür ve `bash`'i çalıştırıp dosya yolunu ona argüman olarak verir. Sonra `bash`
dosyayı normal bir process gibi **okumak** zorundadır. `--x` izinli bir script'te hata mesajı
`/bin/bash: ./script.sh: Permission denied` olur; mesajın başındaki `/bin/bash`, reddedilenin shell'in
değil, çekirdeğin başlattığı yeni `bash` process'inin okuma denemesi olduğunu gösterir. Özetle: program için `x`, script için `r` + `x`.

---

## 2.2.2 `chmod`, `chown`, `chgrp` ve umask `[uygulama]`

### `chmod` — izin bitlerini değiştirmek

İki yazım biçimi vardır:

| Biçim | Örnek | Ne zaman |
|---|---|---|
| **Octal** | `chmod 640 dosya` | Bütün izni baştan tanımlarken |
| **Sembolik** | `chmod g+w dosya` | Mevcut izinde tek bir şeyi değiştirirken |

Sembolik biçimin parçaları: **kim** (`u` sahip, `g` grup, `o` diğerleri, `a` hepsi), **işlem**
(`+` ekle, `-` çıkar, `=` tam olarak bu yap) ve **izin** (`r`, `w`, `x`). Virgülle birleştirilebilir:

```
$ chmod 640 s
$ chmod u+x,g-r,o=r s
$ stat -c '%a %A' s
704 -rwx---r--
```

`640` (`rw-r-----`) → sahibe `x` eklendi → gruptan `r` çıkarıldı → diğerleri tam olarak `r` yapıldı
→ `704`.

**`chmod -R` ve büyük `X`.** `chmod -R 755 dizin` bütün dosyaları çalıştırılabilir yapar; `.txt`
ve `.yml` dosyaları da dahil. Büyük `X` bunun için vardır: **dizinlere ve zaten en az bir `x` biti
olan dosyalara** `x` ekler, diğer dosyalara dokunmaz:

```
$ chmod -R u=rwX,go=rX /srv/site
```

Dikkat: `X` yanlışlıkla çalıştırılabilir yapılmış dosyaları **düzeltmez**, sadece yenisini
eklemez. Bir dizinin izinlerini baştan kurmanın en net yolu dosya ve dizinleri ayırmaktır:

```
$ find /srv/site -type d -exec chmod 755 {} +
$ find /srv/site -type f -exec chmod 644 {} +
```

### `chown` ve `chgrp` — sahipliği değiştirmek

```
$ sudo chown www-data:www-data /srv/site/index.html    # sahip ve grup
$ sudo chown -R appuser: /srv/app                      # sahip = appuser, grup = appuser'ın birincil grubu
$ chgrp devops rapor.txt                                # sadece grup
```

**Kural:** Bir dosyanın sahibini **sadece root** değiştirebilir. Normal bir kullanıcı kendi
dosyasının grubunu, **sadece üyesi olduğu** bir gruba çevirebilir:

```
$ chgrp root g
chgrp: changing group of 'g': Operation not permitted
$ chown root g
chown: changing ownership of 'g': Operation not permitted
```

Neden normal kullanıcı dosyasını başkasına "hediye" edemiyor? Çünkü o zaman disk kotasını
başkasının hesabına doldurabilir, ya da 2.3'te göreceğin setuid bitiyle bir dosyayı başkası
adına çalışır hâle getirmeye çalışabilirdi. Mesajın `Permission denied` değil **`Operation not
permitted`** olduğuna dikkat et. Bu fark 2.5.3'te önemli olacak.

### umask — yeni dosyalar hangi izinle doğar?

Bir program dosya oluştururken genellikle `666` (`rw-rw-rw-`), dizin oluştururken `777` ister.
Çekirdek bu isteği process'in **umask**'ı ile maskeler: umask'ta **açık olan bitler, yeni dosyada
kapatılır.**

| umask | Yeni dosya | Yeni dizin | Nerede görülür |
|---|---|---|---|
| `022` | `644` `rw-r--r--` | `755` `rwxr-xr-x` | Root, systemd servisleri, çoğu dağıtımın varsayılanı |
| `002` | `664` `rw-rw-r--` | `775` `rwxrwxr-x` | Ubuntu'da normal kullanıcılar (her kullanıcının kendi grubu olduğu için) |
| `077` | `600` `rw-------` | `700` `rwx------` | Hassas veri üreten script'ler |

> **🔧 Makinende gör** 🟡 — umask'ı değiştirip sonucu görmek
>
> ```
> $ umask
> 0002
> $ (umask 022; touch f1; mkdir d1; ls -ld f1 d1)
> drwxr-xr-x 2 ubuntu ubuntu 4096 Sep 15 09:25 d1
> -rw-r--r-- 1 ubuntu ubuntu    0 Sep 15 09:25 f1
> $ (umask 077; touch f2; mkdir d2; ls -ld f2 d2)
> drwx------ 2 ubuntu ubuntu 4096 Sep 15 09:25 d2
> -rw------- 1 ubuntu ubuntu    0 Sep 15 09:25 f2
> ```
>
> - Parantez, komutları bir **alt shell'de** çalıştırır (Faz 1, fork). umask da kimlik gibi
>   process'e aittir ve kalıtımla geçer; alt shell'deki değişiklik ana shell'ini etkilemez.
> - `touch` dosyada `x` istemediği için umask'la `x` "kazanmak" mümkün değildir. umask sadece
>   **kapatır.**
>
> **Geri al:** `rm -r f1 d1 f2 d2`

umask kalıcı olarak `/etc/login.defs`, `~/.bashrc` (kullanıcı) veya servis tanımındaki `UMask=`
(systemd, Faz 5) ile ayarlanır. Bir servis dosya oluşturuyor ve dosyaların izni beklediğin gibi
değilse ilk bakacağın yer servisin umask'ıdır.

**❓ Akla gelen soru: `chmod -R 777` yaptım ve sorun çözüldü. Neden herkes bunun kötü olduğunu
söylüyor?**

Çünkü sorunu çözmedin, **gizledin** ve üç yeni sorun yarattın:

1. **Sebebi hiç öğrenmedin.** Hangi kullanıcının hangi kontrolde reddedildiğini bilmiyorsun; aynı
   hata başka bir dosyada tekrar gelecek.
2. **Makinedeki her process artık o dosyaları değiştirebilir.** Nginx'te bir açık bulan saldırgan
   (`www-data`) artık uygulamanın koduna yazabilir. Root'un cron ile çalıştırdığı bir script
   `777` ise **herhangi bir kullanıcı** o script'e bir satır ekleyerek root olarak komut
   çalıştırır. Bu bir yetki yükseltme açığıdır.
3. **Bazı programlar gevşek izinli dosyayı reddeder.** SSH, `~/.ssh` veya ev dizinin herkese
   yazılabilirse key'ini kabul etmez (2.5.1). Yani `chmod -R 777 ~` seni sunucudan kilitleyebilir.

Doğru yol her zaman aynıdır: **kim** (hangi kullanıcı ve grup) **neye** (hangi dosya, hangi dizin)
**hangi izinle** erişmeli? Sadece o izni ver.

---

## 2.2.3 Dizinde `r`, `w`, `x` `[mekanizma]`

Faz 1'de `mv`'nin aynı disk içinde neden anında bittiğini görmüştük: **dizin, isimleri inode
numaralarına bağlayan bir tablodur.** Dosyanın içeriği o tabloda değil, inode'un gösterdiği
bloklardadır. Bu model akılda tutulunca dizin izinleri kendiliğinden anlaşılır. Dizinin bitleri
**tablo** üzerindeki işlemleri kontrol eder, dosyaların içeriğini değil:

| Bit | Dizinde anlamı | Hangi işlem |
|---|---|---|
| `r` | Tablodaki **isimleri listeleyebilir** | `ls dizin` |
| `w` | Tabloya **isim ekleyebilir, silebilir, değiştirebilir** (`x` ile birlikte) | dosya oluşturmak, `rm`, `mv` |
| `x` | Tablodaki bir isimden **inode'a geçebilir** (search/traverse) | `cd dizin`, `cat dizin/dosya`, yoldan geçmek |

`x` bitinin dizindeki adı "search"tür, ama "içinden geçme izni" demek daha açıklayıcıdır. `r`
olmadan `x` = "ismini biliyorsan girebilirsin ama listeleyemezsin". `x` olmadan `r` = "isimleri
görebilirsin ama hiçbirine dokunamazsın".

> **🔧 Makinende gör** 🟡 — Dizin bitlerini tek tek denemek
>
> ```
> $ mkdir d && echo merhaba > d/f
>
> $ chmod 600 d          # rw- : x yok
> $ ls d
> f
> $ ls -l d
> ls: cannot access 'd/f': Permission denied
> total 0
> -????????? ? ? ? ?            ? f
> $ cat d/f
> cat: d/f: Permission denied
>
> $ chmod 100 d          # --x : sadece geçiş
> $ ls d
> ls: cannot open directory 'd': Permission denied
> $ cat d/f
> merhaba
>
> $ chmod 500 d          # r-x : w yok
> $ touch d/yeni
> touch: cannot touch 'd/yeni': Permission denied
> ```
>
> - **`rw-`:** İsimleri okuyabildin (`f`). Ama `ls -l` her dosyanın inode'una gidip boyutunu,
>   sahibini okumak ister; `x` olmadığı için gidemedi ve her alanı `?` bastı. `cat` de dosyaya
>   ulaşamadı.
> - **`--x`:** Listeleme reddedildi, ama ismini **bildiğin** dosyayı okudun.
> - **`r-x`:** Tabloya yeni isim eklenemedi.
>
> **Geri al:** `chmod 700 d && rm -r d`

### Silmek bir dizin işlemidir

`rm dosya` dosyanın içeriğine dokunmaz; dizin tablosundan bir **satırı** siler. Bu yüzden silme
izni dosyanın bitlerine değil, **dizinin** `w` ve `x` bitlerine bağlıdır:

```
$ ls -l salt-okunur.txt
-r--r--r-- 1 ubuntu ubuntu 12 Sep 15 09:30 salt-okunur.txt
$ rm salt-okunur.txt
rm: remove write-protected regular file 'salt-okunur.txt'? y
$ ls salt-okunur.txt
ls: cannot access 'salt-okunur.txt': No such file or directory
```

`rm` sadece **nezaketen** sordu; dizine yazma iznin olduğu için çekirdek silmeye izin verdi. Aynı
mantıkla, kendi ev dizinine root'un bıraktığı bir dosyayı da silebilirsin.

> **⚠️ Yaygın yanılgı: "Bir dosyayı korumak için dosyayı salt okunur yapmak (`chmod 444`) yeterlidir."**
>
> `444` dosyanın **içeriğinin** değiştirilmesini engeller. Dizine yazma izni olan herkes dosyayı
> yine **silebilir**, **yeniden adlandırabilir** veya aynı isimle yeni bir dosya koyabilir. Bir
> dosyanın varlığını korumak istiyorsan dizinin izinlerine bakmalısın. Herkesin yazabildiği ortak
> dizinler (`/tmp` gibi) için çözüm 2.3.1'deki sticky bit'tir.

### Yoldaki her dizin sayılır

`/var/log/auth.log` dosyasını açmak için çekirdek yolu parça parça yürür: `/` → `var` → `log` →
`auth.log`. **Yoldaki her dizinde `x` iznin olmalıdır.** Dosya `644` olsa bile yolun ortasındaki
bir dizin seni geçirmezse dosyaya ulaşamazsın. Yolu tek komutla görmek için:

> **🔧 Makinende gör** 🟢 — Yolun her adımının izni
>
> ```
> $ namei -l /var/log/auth.log
> f: /var/log/auth.log
> drwxr-xr-x root   root   /
> drwxr-xr-x root   root   var
> drwxrwxr-x root   syslog log
> -rw-r----- syslog adm    auth.log
> ```
>
> - `/`, `var` ve `log` dizinlerinin hepsinde diğerleri için `x` var; yol açık.
> - Son adım: `auth.log` sahibi `syslog`, grubu `adm`, `640`. Diğerlerine hiçbir şey yok.
> - `id` çıktında `4(adm)` varsa bu dosyayı `sudo` olmadan okuyabilirsin. EC2'deki `ubuntu`
>   kullanıcısı `adm` grubundadır, bu yüzden logları okuyabilir. Bu grubun **amacı** budur:
>   yönetici olmadan log okumak.
> - `namei -l`, "Permission denied" aramasında en hızlı araçtır; hangi adımın seni durdurduğunu
>   tek bakışta gösterir.

> **🤔 Düşün 2.3**
> Bir ekip arkadaşın statik bir siteyi hızlıca yayınlamak için dosyaları ev dizinine koydu ve
> Nginx'in `root` ayarını oraya yönlendirdi. Tarayıcı **403 Forbidden** gösteriyor. Nginx'in hata
> logunda:
>
> ```
> open() "/home/ubuntu/site/index.html" failed (13: Permission denied)
> ```
>
> İzinler:
>
> ```
> $ ls -ld /home/ubuntu /home/ubuntu/site /home/ubuntu/site/index.html
> drwxr-x--- 6 ubuntu ubuntu 4096 Sep 15 09:40 /home/ubuntu
> drwxrwxr-x 2 ubuntu ubuntu 4096 Sep 15 09:41 /home/ubuntu/site
> -rw-rw-r-- 1 ubuntu ubuntu  512 Sep 15 09:41 /home/ubuntu/site/index.html
> ```
>
> Nginx'in worker process'leri `www-data` kullanıcısıyla çalışıyor. Arkadaşın "`sudo chmod -R 777
> /home/ubuntu` yapıp geçelim" diyor.
>
> 1. `index.html` herkese okunabilir (`r--` diğerleri için). Çekirdek hangi adımda ve hangi
>    üçlüye bakarak reddetti?
> 2. `chmod -R 777 /home/ubuntu` siteyi açar. Ama bir sonraki SSH girişinde ne olur, neden?
> 3. İki doğru çözüm öner.
>
> *(Cevap: fazın sonunda)*

---
---

# 2.3 Özel Bitler

## 2.3.1 setuid, setgid ve sticky `[kavram]`

### Bir bulmaca: `passwd` nasıl çalışıyor?

2.1.2'de gördük: `/etc/shadow` dosyası `640 root:shadow`. Normal bir kullanıcı bu dosyayı
**okuyamaz** bile. Ama aynı kullanıcı `passwd` yazıp kendi parolasını değiştirebiliyor, yani
`/etc/shadow`'a **yazıyor.** 2.2.1'deki kurallara göre bu imkânsız olmalı.

Çözüm, `passwd` programının kendisinde:

```
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 93640 Feb  2 10:14 /usr/bin/passwd
```

Sahip üçlüsünde `x` yerine **`s`** var. Bu **setuid** bitidir ve şu anlama gelir: **bu programı
kim çalıştırırsa çalıştırsın, process dosyanın sahibinin (root'un) kimliğiyle çalışır.**

### Real UID ve effective UID

2.1.1'de `/proc/self/status` içinde dört `Uid` değeri görmüştük. Şimdi ilk ikisi anlam kazanıyor:

| Alan | Anlamı | `passwd` çalışırken |
|---|---|---|
| **Real UID** | Process'i **kim başlattı** | `1000` (ubuntu) |
| **Effective UID** | İzin kontrollerinde **kim sayılır** | `0` (root) |

Çekirdek 2.2.1'deki kontrolleri **effective UID** ile yapar. `passwd` root gibi kontrol edildiği
için `/etc/shadow`'a yazabilir. Real UID ise programın "beni kim çağırdı?" sorusunu cevaplamasını
sağlar: `passwd` sadece **real UID'nin** parolasını değiştirmene izin verir; başka birinin
parolasını değiştirmek için real UID'nin de 0 olması gerekir.

İkinci bir terminalde `passwd` parola sorarken bekletip şunu çalıştırırsan farkı görürsün:

```
$ ps -o pid,ruser,euser,cmd -C passwd
    PID RUSER    EUSER    CMD
   2841 ubuntu   root     passwd
```

Bu, setuid programların neden **tehlikeli** olduğunu da gösterir: program, onu çalıştıran
kullanıcıya root'un gücünü **ödünç verir.** Programın tek bir hatası (başka bir dosyaya yazmasına
veya bir shell açmasına izin veren bir açık), o kullanıcının root olması demektir. Bu yüzden
setuid programlar küçük, dikkatle yazılmış ve sayıca az tutulur.

> **🔧 Makinende gör** 🟢 — Makinedeki setuid programlar
>
> ```
> $ find / -xdev -perm -4000 -type f 2>/dev/null
> /usr/bin/chfn
> /usr/bin/su
> /usr/bin/gpasswd
> /usr/bin/umount
> /usr/bin/newgrp
> /usr/bin/chsh
> /usr/bin/mount
> /usr/bin/pkexec
> /usr/bin/passwd
> /usr/bin/sudo
> /usr/lib/openssh/ssh-keysign
> ...
> ```
>
> - `-perm -4000`: "en az setuid biti açık olanlar". `-xdev` başka dosya sistemlerine
>   (`/proc` gibi) inmeyi engeller; `2>/dev/null` erişemediğin dizinlerin hata mesajlarını atar.
> - Listedeki her programın bir sebebi var: `su` ve `sudo` kimlik değiştirir, `mount` disk
>   bağlar, `passwd`/`chsh`/`chfn` kullanıcı veritabanına yazar.
> - Listeyi bir dosyaya kaydet. 2.3.2'de bu listenin neden bir **güvenlik referansı** olduğunu
>   göreceğiz.

### setgid: aynı fikir, grup için

Grup üçlüsünde `s` görürsen bu **setgid** bitidir. Dosyada ve dizinde farklı işler yapar.

**Dosyada:** process dosyanın **grubunun** kimliğiyle çalışır. Örnek `crontab`:

```
$ ls -l /usr/bin/crontab
-rwxr-sr-x 1 root crontab 39744 Nov  5 11:02 /usr/bin/crontab
$ sudo ls -ld /var/spool/cron/crontabs
drwx-wx--T 2 root crontab 4096 May 30 03:20 /var/spool/cron/crontabs
```

Kullanıcıların cron tablolarının durduğu dizine sadece `crontab` grubu yazabilir (`-wx`, listeleme
yok). `crontab -e` çalıştırdığında process `crontab` grubunda sayılır ve dosyanı oraya yazar. Root
yetkisinin tamamı yerine **tek bir grubun** yetkisi ödünç verilir; setuid root'a göre çok daha
dar bir yetkidir.

**Dizinde:** setgid bitli bir dizinde oluşturulan her dosya, oluşturan kullanıcının grubunu değil,
**dizinin grubunu** alır. Paylaşılan proje dizinlerinin temelidir:

> **🔧 Makinende gör** 🟡 — setgid dizininde grup kalıtımı
>
> ```
> $ mkdir ortak && chmod 2775 ortak
> $ ls -ld ortak
> drwxrwsr-x 2 ubuntu ubuntu 4096 Sep 15 10:05 ortak
> ```
>
> - `2775`: baştaki `2` setgid, `775` normal izinler. Grup üçlüsü `rwx` yerine `rws` gösteriyor.
> - Gerçek bir senaryoda `sudo chgrp devops ortak` ile dizinin grubu `devops` yapılır. Sonra
>   `devops`'taki herkesin oluşturduğu dosyalar `devops` grubunda doğar ve umask `002` ise grup
>   üyeleri birbirinin dosyalarını düzenleyebilir.
> - Setgid olmasa her dosya oluşturanın kişisel grubunda (`alice:alice`) doğardı ve diğer
>   üyeler için grup üçlüsü işe yaramazdı.
>
> **Geri al:** `rm -r ortak`

### Sticky bit: ortak dizinde "sadece kendi dosyanı sil"

2.2.3'te öğrendik: dizine yazma izni olan herkes içindeki **her** dosyayı silebilir. `/tmp`
herkese yazılabilir olmak **zorunda**, çünkü her kullanıcı ve servis oraya geçici dosya koyar. O
zaman herkes birbirinin dosyasını silebilir mi?

```
$ ls -ld /tmp
drwxrwxrwt 18 root root 4096 Sep 15 10:07 /tmp
```

Diğerleri üçlüsündeki **`t`**, **sticky bit**'tir. Sticky bitli bir dizinde bir dosyayı silmek
veya yeniden adlandırmak için dizine yazma izni **yetmez**: dosyanın sahibi, dizinin sahibi veya
root olman gerekir.

### Özet: dört basamaklı octal ve büyük harfler

Özel bitler octal gösterimde **dördüncü basamak** olarak en başa yazılır:

| Değer | Bit | Nerede görünür | Örnek |
|---|---|---|---|
| `4000` | setuid | Sahip `x` yerinde `s` | `4755` → `-rwsr-xr-x` |
| `2000` | setgid | Grup `x` yerinde `s` | `2775` → `drwxrwsr-x` |
| `1000` | sticky | Diğerleri `x` yerinde `t` | `1777` → `drwxrwxrwt` |

**Küçük harf / büyük harf:** özel bit `x` ile aynı yeri paylaştığı için iki bilgiyi tek harfle
gösterir. `s`/`t` = özel bit **ve** `x` açık. `S`/`T` = özel bit açık **ama** `x` kapalı.

```
$ chmod 4644 dosya          → -rwSr--r--
$ chmod 1776 dizin          → drwxrwxrwT
```

Büyük harf neredeyse her zaman bir hatanın işaretidir: `x` olmayan bir dosyanın setuid biti hiçbir
şey yapmaz. Az önceki `/var/spool/cron/crontabs` gibi bilinçli istisnalar vardır, ama bir `S`
gördüğünde önce "bu kasıtlı mı?" diye sor.

> **⚠️ Yaygın yanılgı: "Bir script'e setuid verirsem root olarak çalışır."**
>
> Linux, `#!` ile başlayan script'lerde setuid ve setgid bitlerini **yok sayar.** Sebebi bir yarış
> durumu (race condition): çekirdek script'i açıp yorumlayıcıyı başlatması ile yorumlayıcının
> dosyayı tekrar açması arasında, bir saldırgan dosyayı (veya ona giden linki) başka bir script'le
> değiştirebilirdi. Script'in root olarak çalışması gerekiyorsa doğru araç `sudo` kuralıdır
> (2.4.1) veya script'i root olarak çalışan bir systemd servisine koymaktır (Faz 5).
>
> Ek bir koruma katmanı: `nosuid` seçeneğiyle bağlanmış dosya sistemlerinde (`/tmp`, `/dev/shm`,
> çıkarılabilir diskler sıklıkla böyledir) setuid bitleri de yok sayılır. `findmnt -no OPTIONS -T
> /dev/shm` çıktısında `nosuid` görürsün.

**❓ Akla gelen soru: `sudo` da setuid mi? O zaman `sudo` ile `passwd` arasında ne fark var?**

Evet, `sudo` da setuid root bir programdır; root olarak başlamasının başka yolu yok. Fark,
programın root gücünü **ne için** kullandığıdır. `passwd` bu gücü tek bir sabit iş için
kullanır: senin parolanı değiştirmek. `sudo` ise gücü, bir **kural dosyasına** (sudoers) bakarak
**başka programları** root olarak çalıştırmak için kullanır. Yani `sudo` "root gücü verilen
program" değil, "root gücünü kurallara göre dağıtan program"dır. Nasıl karar verdiğini 2.4.1'de
göreceğiz.

---

## 2.3.2 Bozulunca: yanlış yerdeki setuid biti `[uygulama]`

setuid bitinin gücü, yanlış programda bir felakete dönüşür. Bir yöneticinin "kullanıcılar log
aramak için `find`'ı sudo'suz kullanabilsin" diye şunu yaptığını düşün:

```
$ sudo chmod u+s /usr/bin/find        # YAPMA
$ ls -l /usr/bin/find
-rwsr-xr-x 1 root root 204264 Mar 31 12:10 /usr/bin/find
```

Niyet: `find` root yetkisiyle her dizinde arama yapabilsin. Sonuç: `find`'ın `-exec` seçeneği
**herhangi bir programı** çalıştırabilir. Makinedeki her kullanıcı artık şunu yazabilir:

```
$ find . -maxdepth 0 -exec /bin/sh -p \;
# id
uid=1000(ubuntu) euid=0(root) gid=1000(ubuntu) egid=0(root) ...
# whoami
root
```

`find`'ın `-exec` özelliği herhangi bir komutu çalıştırır; `find` root efektif UID'siyle
çalıştığı için başlattığı `/bin/sh` de root olur. `-p` bayrağı kabuğun efektif UID'yi başlangıçta
gerçek UID'ye düşürmesini engeller. Aynı tuzak `vim` (`:!sh`), `less` (`!sh`), `awk`
(`BEGIN{system("sh")}`), `tar --to-command` ve onlarca başka araçta vardır. Bir programdan kabuğa
kaçış yollarının kataloğu **GTFOBins** adıyla bilinir ve bir saldırganın makineye düştüğünde ilk
baktığı yerlerden biridir.

> **⚠️ Yaygın yanılgı: "setuid'i sadece root koyabildiği için setuid binary güvenlidir."**
>
> setuid bitini kimin koyduğu değil, **hangi programa** konduğu önemlidir. Root'un iyi niyetle
> `vim`'e veya `find`'a koyduğu bir setuid, o sunucudaki her kullanıcıya root verir. Kural: setuid
> yalnızca **tek ve dar bir iş** yapan, içinden kabuk veya rastgele komut çalıştırılamayan
> programlarda olmalıdır. Genel amaçlı hiçbir araca (editör, arşivleyici, yorumlayıcı, arama
> aracı) setuid konmaz.

### Tespit: bilinen-iyi bir listeye karşı fark

Bir sunucuda beklenmedik setuid binary'lerini bulmanın yolu, ilk kurulumda alınan bilinen-iyi bir
envanterle karşılaştırmaktır:

> **🔧 Makinende gör** 🟢 — setuid envanterini bir baseline'a karşı tutmak
>
> ```
> $ find / -xdev \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null | sort > suid.now
> $ diff suid.baseline suid.now
> > /usr/bin/vim.basic          # ← baseline'da yoktu: incele
> ```
>
> - `-xdev` aramayı tek dosya sistemiyle sınırlar (network mount'lara veya `/proc`'a dalmaz).
> - İlk kurulumda `suid.now`'ı `suid.baseline` olarak saklarsın; sonra düzenli olarak
>   karşılaştırırsın. Yeni bir satır ya meşru bir paket kurulumudur ya da bir yetki-yükseltme
>   izidir. Bu, Faz 9'daki sertleştirme ve Faz 11'deki gözlemlenebilirlik fikirlerinin küçük bir
>   habercisidir.
> - En sağlam savunma önce mount'tadır: kullanıcının yazabildiği dosya sistemlerini (`/tmp`,
>   `/home`, ayrı veri diskleri) `nosuid` ile mount etmek, oralara bırakılan hiçbir setuid
>   binary'nin işlememesini garanti eder.

> **🤔 Düşün 2.4**
> Bir ekip `/srv/raporlar` dizininde ortak çalışacak: `ayse`, `mehmet` ve `deploy` kullanıcıları
> buraya rapor dosyaları yazacak; **birbirlerinin** dosyalarını okuyup güncelleyebilmeli, ama
> yanlışlıkla birbirlerinin dosyalarını **silmemeli.** Bir yönetici sorunu çözmek için
> `chmod -R 777 /srv/raporlar` yaptı.
>
> 1. `777` neyi çözdü, hangi gereksinimi bozdu? (Özellikle "silmemeli" şartını düşün.)
> 2. Üç kullanıcının aynı `raporcu` grubuna alındığını varsay. Dizini hangi izinler ve hangi özel
>    bit(ler)le kurarsın ki: (a) grup üyeleri yeni dosya oluşturabilsin, (b) yeni dosyalar otomatik
>    olarak `raporcu` grubuna ait olsun, (c) herkes birbirinin dosyasını okuyup yazabilsin ama
>    yalnızca sahibi silebilsin?
> 3. Octal modu yaz ve `ls -ld /srv/raporlar` çıktısının nasıl görüneceğini tahmin et.
>
> *(Cevap: fazın sonunda)*

---
---

# 2.4 Yükseltilmiş Yetki: su ve sudo

## 2.4.1 sudo ve sudoers `[mekanizma]`

Bir işlem gerçekten root gerektirdiğinde iki yol vardır. Eskisi `su` (**s**witch **u**ser):
root'un parolasını girip tam bir root kabuğu açarsın. Modern sunucularda tercih edilen ise
**sudo**'dur ve farkı önemlidir:

| | `su -` | `sudo <komut>` |
|---|---|---|
| Hangi parola? | **Hedef** kullanıcının (root) parolası | **Kendi** parolan |
| Ne alırsın? | Tam root kabuğu, süresiz | Tek komut, root olarak |
| Kim yetkili? | Root parolasını bilen herkes | `sudoers`'ta adı geçen kullanıcılar |
| İz kalır mı? | Sadece "su oturumu açıldı" | **Her komut** loglanır |

Bulut sunucularında root'un parolası çoğunlukla **hiç yoktur** (Ubuntu'da `disable_root: true` ve
root satırı `!` ile kilitlidir), bu yüzden `su -` zaten çalışmaz. Erişim tamamen `sudo` üstünden
verilir.

### Bir `sudo` komutu adım adım

`sudo systemctl restart nginx` yazdığında olanlar:

1. **`sudo` bir setuid binary'dir** (`-rwsr-xr-x root root`). Sen çalıştırırsın, process anında
   root efektif UID'siyle başlar. (2.3'teki setuid modeli burada işe yarıyor.)
2. Kural dosyasını okur: `/etc/sudoers` ve `/etc/sudoers.d/*`. Bu dosyalar `440 root:root`'tur —
   normal kullanıcı okuyamaz bile.
3. Seni **PAM** üzerinden doğrular (parolanı sorar, sonucu bir süre cache'ler).
4. Ortamı temizler: `env_reset` çoğu değişkeni atar, `secure_path` `PATH`'i güvenli bir sabit
   değere kurar. Bu yüzden `sudo` altında `PATH`'in farklıdır.
5. İzin verilmişse komutu root olarak `exec` eder.
6. İşlemi loglar: `journalctl -t sudo` veya `/var/log/auth.log`:
>
> ```
> sudo: ubuntu : TTY=pts/0 ; PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/systemctl restart nginx
> ```

### sudoers kural sözdizimi

Bir sudoers satırının anatomisi:

```
ubuntu   ALL = (ALL:ALL)   ALL
  │       │      │     │     └─ hangi komutlar (ALL = her komut)
  │       │      │     └─────── hangi hedef grup olarak
  │       │      └───────────── hangi hedef kullanıcı olarak
  │       └──────────────────── hangi host'ta (ALL = her yerde)
  └──────────────────────────── kim
```

Kullanıcı yerine `%grup` yazılırsa kural o grubun tüm üyelerine uygular. Dağıtımların varsayılan
kuralları:

| Dağıtım | Kural | Anlamı |
|---|---|---|
| Ubuntu / Debian | `%sudo ALL=(ALL:ALL) ALL` | `sudo` grubundaki herkes her şeyi yapabilir (parolayla) |
| RHEL / Amazon Linux | `%wheel ALL=(ALL) ALL` | `wheel` grubundaki herkes (parolayla) |
| EC2 (cloud-init) | `ubuntu ALL=(ALL) NOPASSWD:ALL` | `/etc/sudoers.d/90-cloud-init-users` içinde; **parolasız** |

Son satır EC2'de neden `ubuntu` kullanıcısının hiç parola sormadan `sudo` yapabildiğini açıklar:
cloud-init ilk boot'ta bu dosyayı yazar. `NOPASSWD` kolaylık sağlar ama SSH anahtarını ele
geçiren birinin anında root olması demektir — bu yüzden anahtar dosyasının izinleri (2.5.1) o
kadar kritiktir.

> **🔧 Makinende gör** 🟢 — Kendi sudo yetkilerini listelemek
>
> ```
> $ sudo -l
> Matching Defaults entries for ubuntu on ip-10-0-1-5:
>     env_reset, mail_badpass, secure_path=/usr/local/sbin\:...
>
> User ubuntu may run the following commands on ip-10-0-1-5:
>     (ALL) NOPASSWD: ALL
> ```
>
> - `sudo -l` sana **hangi komutları hangi kullanıcı olarak** çalıştırabileceğini söyler. Kısıtlı
>   bir kullanıcıda burada tek tek izin verilen komutları görürsün.
> - `Defaults` satırında `secure_path` ve `env_reset` görünür — 4. adımdaki ortam temizliğinin
>   kanıtı.

### `sudoers`'ı güvenle düzenlemek

`/etc/sudoers` dosyasını **asla** doğrudan bir editörle açma. Sözdiziminde tek bir hata dosyayı
bozarsa `sudo` çalışmayı reddeder ve makinede root'a **hiçbir yolun kalmayabilir.** Bunun yerine:

- `sudo visudo` — dosyayı düzenler, kaydederken sözdizimini **doğrular**, hatalıysa kaydetmez.
- Kendi kurallarını ana dosyaya değil, `/etc/sudoers.d/` altında ayrı bir dosyaya koy:
  `sudo visudo -f /etc/sudoers.d/90-deploy`. Ana dosyadaki `@includedir /etc/sudoers.d` bunları
  otomatik dahil eder.

> **🔧 Makinende gör** 🔴 — Kısıtlı bir sudo kuralı eklemek
>
> ```
> $ sudo visudo -f /etc/sudoers.d/91-deneme
> # editöre şunu yaz ve kaydet:
> ubuntu ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx
>
> $ sudo -l | grep systemctl
>     (root) NOPASSWD: /usr/bin/systemctl restart nginx
> ```
>
> - `visudo -f` kaydederken sözdizimini doğrular; hata varsa "What now?" diye sorar, `e` ile
>   düzeltir veya `x` ile iptal edersin. Doğrudan editörle olsaydı bozuk dosya kaydedilirdi.
> - Bu kural `ubuntu`'ya sadece **o tek komutu** verir; başka bir şeyi root yapamaz.
>
> **Geri al:** `sudo rm /etc/sudoers.d/91-deneme`

> **⚠️ Yaygın yanılgı: "Tek bir komuta sudo izni vermek güvenlidir."**
>
> Verdiğin komut **içinden başka komut çalıştırabiliyorsa** güvenli değildir. `sudo vim`, `sudo
> less`, `sudo find`, `sudo awk`, `sudo tar`, `sudo systemctl` (pager üzerinden) — hepsinden bir
> kabuğa kaçılabilir ve o kabuk root olur. Bir kullanıcıya `sudo vim /etc/app.conf` izni verdiysen,
> o kullanıcı `:!sh` yazıp tam root kabuğu alır. Bu tam olarak Düşün 2.5'in konusu. Bir dosyayı
> güvenle düzenletmek için doğru araç `sudoedit`'tir (Cevap 2.5).

---

## 2.4.2 Uygulamayı neden root çalıştırmamalı `[uygulama]`

Yeni başlayanların en sık yaptığı hata, "izin sorunu çıkmasın" diye uygulamayı root olarak
çalıştırmaktır. Bunun bedeli **patlama yarıçapıdır** (blast radius): uygulamada bir güvenlik açığı
bulunursa, saldırgan uygulamanın yetkisiyle ne yapabiliyorsa onu yapar. Uygulama root'sa,
saldırgan da root olur — tüm makine gider. Uygulama `www-data` gibi yetkisiz bir kullanıcıysa,
saldırgan sadece o kullanıcının erişebildiği dosyalarla sınırlı kalır.

Bu yüzden iyi tasarlanmış servisler yetkiyi **en aza indirir**. Nginx'in klasik deseni:

```
$ ps -eo user,comm | grep nginx
root     nginx        ← master: 80/443 portunu açar, sonra bekler
www-data nginx        ← worker: gerçek istekleri işler, yetkisiz
www-data nginx
```

Master process root'tur çünkü **1024'ün altındaki portları** (80, 443) sadece root açabilir. Ama
portu açtıktan sonra gerçek işi yapan worker'lar yetkisiz `www-data` kullanıcısına düşer. Bir
saldırgan bir worker'ı ele geçirse bile root olamaz.

### 1024 altı port sorununun çözümleri

Uygulamanı root yapmadan düşük port dinlemenin yolları:

1. **Reverse proxy:** Nginx/ALB 80/443'ü dinler, uygulamana 8080'de proxy'ler. En yaygın bulut
   deseni; uygulaman hiç root olmaz.
2. **Capability:** `setcap CAP_NET_BIND_SERVICE=+ep /path/to/app` — root olmadan sadece "düşük
   port açma" yeteneğini verir (Faz 9).
3. **systemd socket activation:** systemd portu root olarak açar, soketi yetkisiz servise devreder.

### Servis için özel kullanıcı

Bir servisin kendi yetkisiz kullanıcısını oluşturmanın standart yolu:

```
$ sudo useradd --system --shell /usr/sbin/nologin --no-create-home appuser
```

- `--system` UID'yi sistem aralığından (1000 altı) verir; bu kullanıcı bir insan değildir.
- `--shell /usr/sbin/nologin` interaktif giriş yapamaz — sadece servisi çalıştırmak için var.
- systemd servis dosyasında `User=appuser` yazarak servisi bu kimlikle başlatırsın (Faz 8).

> **🔧 Makinende gör** 🟢 — Hangi process hangi kullanıcıyla çalışıyor
>
> ```
> $ ps -eo user= | sort | uniq -c | sort -rn
>     142 root
>      38 ubuntu
>      12 www-data
>       9 systemd+
>       4 messagebus
> ```
>
> - Sağlıklı bir sunucuda root process'leri çekirdek işleri ve servis yöneticileridir; **uygulama**
>   process'leri kendi yetkisiz kullanıcılarında görünmelidir.
> - Uygulamanı `root` satırında görüyorsan, bu düzeltmen gereken bir bulgudur.

> **🤔 Düşün 2.5**
> Bir deploy otomasyonu için `deploy` kullanıcısına şu sudoers kuralı verildi:
>
> ```
> deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart app, /usr/bin/vim /etc/app/config.yml
> ```
>
> Niyet: `deploy` sadece uygulamayı yeniden başlatabilsin ve tek bir config dosyasını
> düzenleyebilsin — tam root olmasın.
>
> 1. `deploy` kullanıcısı bu iki komutla nasıl **tam root** olur? Tam adımı yaz.
> 2. Bu, 2.4.1'deki hangi yanılgının somut hâli?
> 3. Config dosyasını güvenle düzenletmek için kuralı nasıl değiştirirsin? (İpucu: `sudoedit`.)
>
> *(Cevap: fazın sonunda)*

---
---

# 2.5 OS Kimliği ve Bulut Kimliği

## 2.5.1 EC2 varsayılan kullanıcıları ve SSH anahtarının yolculuğu `[uygulama]`

Yeni bir EC2 instance'ı başlattığında bir anahtar çifti seçersin ve sonra `ssh -i key.pem
ubuntu@<ip>` ile bağlanırsın. Bu "ubuntu" kullanıcısı nereden geldi, anahtar oraya nasıl ulaştı?
Bu zincir bulut kimliğinin OS kimliğine nasıl dönüştüğünü gösterir.

**Dağıtıma göre varsayılan kullanıcı** farklıdır (root'la giriş her zaman kapalıdır):

| AMI | Varsayılan kullanıcı |
|---|---|
| Ubuntu | `ubuntu` |
| Amazon Linux | `ec2-user` |
| Debian | `admin` |
| RHEL / CentOS | `ec2-user` / `centos` |
| Fedora | `fedora` |

Root olarak bağlanmayı denersen açık bir mesaj alırsın:

```
Please login as the user "ubuntu" rather than the user "root".
```

Bu mesaj bir OS özelliği değildir: cloud-init, root'un `authorized_keys` dosyasına bu uyarıyı
yazdırıp bağlantıyı kapatan bir komut koyar.

### Anahtarın yolculuğu

![Şekil 2.3 — SSH anahtarının konsoldan kabuğa yolculuğu](../diagrams/png/lx-2-03-ssh-key-journey.png)
*Şekil 2.3 — Açık anahtar konsolda seçilir, IMDS üzerinden instance'a ulaşır, cloud-init ilk
boot'ta kullanıcıyı ve `authorized_keys`'i oluşturur, sonraki her girişte sshd imzayı doğrular.*

1. **Konsol:** Bir anahtar çifti oluşturursun. AWS **açık** anahtarı saklar, **özel** anahtarı
   (`.pem`) sana bir kez indirtir. Özel anahtar bir daha asla AWS'de olmaz.
2. **IMDS:** Instance başlarken açık anahtar instance metadata servisinde belirir:
   `http://169.254.169.254/latest/meta-data/public-keys/0/openssh-key`.
3. **cloud-init:** İlk boot'ta (per-instance, sadece bir kez) cloud-init varsayılan kullanıcıyı
   oluşturur, `sudo` yetkisini verir ve açık anahtarı `~/.ssh/authorized_keys` dosyasına yazar.
4. **sshd:** Bağlandığında sshd, `authorized_keys`'teki açık anahtarla senin özel anahtarınla
   imzaladığın veriyi **doğrular.** Parola hiç kullanılmaz.

### StrictModes: neden 600 ve 700 şart

sshd, anahtar dosyalarının izinlerini **kontrol eder** (`StrictModes yes`, varsayılan). Anahtar
dosyanı veya `.ssh` dizinini başkalarının okuyabileceği kadar gevşek bırakırsan sshd anahtarı
**reddeder** — çünkü gevşek izinli bir özel anahtar zaten güvenli değildir:

| Yol | Gereken izin |
|---|---|
| `~` (ev dizini) | başkaları yazamamalı (bu yüzden EC2'de `750`) |
| `~/.ssh` | `700` (`drwx------`) |
| `~/.ssh/authorized_keys` | `600` (`-rw-------`) |
| İndirdiğin `.pem` (istemci tarafı) | `400` veya `600` |

Yanlış izinde tipik hata mesajları:

```
# Sunucuda /var/log/auth.log:
Authentication refused: bad ownership or modes for directory /home/ubuntu/.ssh
# İstemcide:
Permissions 0644 for 'key.pem' are too open.
```

> **⚠️ Yaygın yanılgı: "Konsoldan instance'ın anahtar çiftini değiştirince eski anahtar geçersiz olur."**
>
> Konsoldaki anahtar çifti sadece **ilk boot'ta**, cloud-init tarafından okunur. Çalışan bir
> instance'ın `authorized_keys` dosyası zaten diskte yazılıdır; konsoldan anahtarı "değiştirmek"
> o dosyaya dokunmaz. Erişimi gerçekten değiştirmek için `authorized_keys`'i elle düzenlemen (veya
> aşağıdaki kurtarma yollarından biriyle girmen) gerekir.

### Kilitlendiğinde: kurtarma yolları

SSH anahtarını kaybettiysen veya izinleri bozup kendini kilitlediysen, AWS OS'tan bağımsız
kurtarma yolları sunar:

- **EC2 Instance Connect:** Konsoldan geçici bir anahtar enjekte eder (Amazon Linux'ta ve
  Ubuntu'da agent kuruluysa).
- **SSM Session Manager:** SSM agent ve doğru IAM rolü varsa, SSH hiç kullanmadan bir kabuk
  açar — port 22 kapalı olsa bile.
- **Volume detach:** Kök diski ayırıp başka bir instance'a bağlar, `authorized_keys`'i veya
  bozuk `sudoers`'ı elle düzeltir, geri takarsın. En son çare ama her zaman çalışır.

**❓ Akla gelen soru: `authorized_keys` diskteyse, SSH anahtarını değiştirmek için neden IMDS'e
gidilmiyor?**

Çünkü IMDS'teki açık anahtar sadece cloud-init'in ilk boot referansıdır; canlı erişimi belirleyen
şey **diskteki** `authorized_keys` dosyasıdır. Yeni bir anahtarı tanıtmak için o dosyaya bir satır
eklemek yeterlidir (`ssh-copy-id` veya elle). IMDS'i güncellemek çalışan instance'ı etkilemez.

---

## 2.5.2 OS izinleri IAM değildir `[kavram]`

Bulutta iki ayrı **yetkilendirme dünyası** vardır ve bunları karıştırmak en sık kafa karışıklığı
kaynağıdır:

| | OS izinleri (bu faz) | IAM (AWS) |
|---|---|---|
| Neyi korur? | Instance **içindeki** dosya, process, port | AWS **API'leri**: S3, EC2, RDS... |
| Kim uygular? | Linux çekirdeği | AWS kontrol düzlemi |
| Kimlik nedir? | UID/GID, gruplar | IAM kullanıcısı/rolü, policy |
| Nerede kontrol edilir? | `open()`, `execve()` syscall'ları | Her AWS API çağrısı |

Bu ikisi **birbirini tanımaz.** Instance'a bağlı IAM rolü S3'e tam erişim verebilir ama bu, OS
içinde `/etc/shadow`'u okuyabileceğin anlamına gelmez. Tersi de doğru: OS'ta root olman, `aws s3
ls` komutunun çalışacağı anlamına gelmez — o komut instance'ın IAM rolüne bağlıdır.

### Kritik köprü: IMDS'teki rol kimlik bilgileri

İki dünya bir noktada tehlikeli biçimde buluşur. Instance'a bir IAM rolü bağlıysa, o rolün geçici
kimlik bilgileri IMDS'ten okunabilir:

```
$ curl http://169.254.169.254/latest/meta-data/iam/security-credentials/<rol-adı>
```

Bu endpoint'i **instance üzerindeki herhangi bir yerel kullanıcı** okuyabilir — root olmak
gerekmez. Yani uygulamanda (yetkisiz `www-data` bile) bir SSRF veya komut-enjeksiyonu açığı
bulan saldırgan, instance'ın IAM rolünün tüm AWS yetkilerini ele geçirir. Bu yüzden:

- Instance rolüne mümkün olan **en dar** IAM policy'sini ver (least privilege — Faz 12).
- **IMDSv2**'yi zorunlu kıl (token gerektirir, SSRF'i büyük ölçüde engeller).

### SSM ve Instance Connect: iki dünya arasındaki köprüler

- **SSM Session Manager** bağlandığında seni `ssm-user` olarak bir kabuğa düşürür; bu kullanıcının
  sudoers yetkisi vardır. IAM izni ("bu instance'a SSM ile bağlanabilir misin?") OS kimliğine
  (`ssm-user`) çevrilir.
- **EC2 Instance Connect** IAM iznini kontrol eder, sonra geçici SSH anahtarını OS'un
  `authorized_keys`'ine enjekte eder. Yine bir IAM → OS köprüsü.

**❓ Akla gelen soru: IAM'de admin'im ama sunucuda `Permission denied` alıyorum. Nasıl olur?**

Çünkü IAM admin'liğin AWS **API'lerinde** geçerlidir; instance'ın **içinde** sıradan bir Linux
kullanıcısısın. `/etc/nginx/nginx.conf`'u düzenlemek için OS'ta `sudo` yetkisi gerekir, IAM policy
değil. İki dünya ayrıdır; birindeki yetki diğerine geçmez. (Tek istisna: IAM ile SSM/Instance
Connect üzerinden makineye **girip**, oradaki OS yetkilerini kullanmaktır.)

---

## 2.5.3 "Permission denied" karar ağacı `[uygulama]`

Bu faz boyunca reddedilmenin onlarca farklı nedenini gördük. Bir sunucuda `Permission denied`
gördüğünde panik yapmak yerine sırayla eleyeceğin bir karar ağacı şudur. Her adım Faz 2'nin bir
bölümüne bağlanır:

1. **Kim olarak çalışıyorum?** `id`, ya da process için `ps -o pid,user,cmd -p <pid>`. Beklediğin
   kullanıcı mı? (2.1)
2. **Yola erişebiliyor muyum?** `namei -l /tam/yol` — yolun her dizininde `x` var mı? İlk `?`
   veya reddedilen dizin suçludur. (2.2.3)
3. **Doğru izin sınıfı mı?** `stat -c '%a %A %U:%G' dosya` — sahip miyim, grup üyesi miyim, yoksa
   "diğer" miyim? Çekirdek hangi üçlüye bakar? (2.2.1)
4. **Grup gerçekten aktif mi?** `id` çıktısında grubu görüyorum ama erişemiyorum → gruba
   girişten **sonra** mı eklendim? Yeni oturum veya `newgrp` gerekli. (2.1.1, Cevap 2.1)
5. **Dosya sistemi izin veriyor mu?** `findmnt -T /tam/yol` — `ro` (salt okunur), `noexec`
   (çalıştırma yok) veya `nosuid` mi? Bit'ler doğru olsa da mount reddedebilir. (2.3.1)
6. **Dosya kilitli mi?** `lsattr dosya` — `i` (immutable) bayrağı varsa root bile yazamaz;
   `chattr -i` ile kaldırılır. (Faz 6)
7. **ACL var mı?** `ls -l` çıktısında iznin sonunda `+` varsa standart bit'lerin ötesinde bir ACL
   vardır; `getfacl dosya` ile bak. (Faz 9)
8. **MAC devrede mi?** AppArmor/SELinux bit'ler doğru olsa da reddedebilir; log'da `apparmor` veya
   `avc: denied` ara. (Faz 9)
9. **Özel durum mu?** SSH publickey reddi (StrictModes), docker soketine erişim (docker grubu),
   1024 altı port bağlama — hepsinin kendi kuralı var. (2.4.2, 2.5.1)

### errno'yu oku: EACCES ≠ EPERM ≠ EROFS

Hata mesajının **tam metni** hangi dünyada olduğunu söyler:

| errno | Mesaj | Anlamı | İlk bakılacak |
|---|---|---|---|
| `EACCES` | `Permission denied` | İzin bit'leri yetersiz | `stat`, `namei -l` (2.2) |
| `EPERM` | `Operation not permitted` | İşlem yetki (capability) gerektiriyor | root/capability gerek (2.2.2, `chown`) |
| `EROFS` | `Read-only file system` | Mount salt okunur | `findmnt -T` (2.3.1) |

2.2.2'de `chgrp root` denemesinin `Operation not permitted` (EPERM) verdiğini, ama bir dosyayı
okuyamamanın `Permission denied` (EACCES) verdiğini hatırla. Bu fark, karar ağacında hangi daldan
gideceğini söyler.

> **🤔 Düşün 2.6**
> Bir deploy script'i `/data/scripts/backup.sh` yolunda, izinleri `-rwxr-xr-x` (yani `755`). Onu
> doğrudan çalıştırmak istiyorsun:
>
> ```
> $ ./backup.sh
> bash: ./backup.sh: Permission denied
> $ bash backup.sh
> [script normal çalışıyor]
> ```
>
> Dosyada `x` var, sen sahipsin, yol da açık. Yine de `./backup.sh` reddediliyor ama `bash
> backup.sh` çalışıyor.
>
> 1. `bash backup.sh` neden çalışırken `./backup.sh` neden reddediliyor? İkisi arasındaki fark tam
>    olarak ne?
> 2. Karar ağacındaki hangi adım bunu yakalar? Hangi komutu çalıştırırdın?
> 3. Asıl nedeni bulduğunda, script'i doğrudan çalıştırmak için kalıcı çözüm ne olur?
>
> *(Cevap: fazın sonunda)*

---
---

# 2.6 Bu Faz Bozulunca — Arıza İmzaları

Erişim modelini kavradığında, bir sürü "gizemli" hata aslında aynı birkaç mekanizmanın
belirtisine dönüşür. Aşağıdaki tablo, sahada göreceğin belirtileri bu fazın mekanizmalarına
bağlar:

| Belirti | Olası mekanizma | Nerede anlatıldı | İlk bakılacak |
|---|---|---|---|
| `sudo: unable to resolve host` / `sudo` hiç çalışmıyor | Bozuk `/etc/sudoers`, veya kullanıcı `sudo` grubunda değil | 2.4.1 | `sudo -l`; Instance Connect/SSM/volume detach ile gir, `visudo -c` |
| `usermod -aG` yaptım ama `docker ps` hâlâ denied | Kimlik oturum açışında sabitlenir; grup henüz aktif değil | 2.1.1 | `id`; yeniden giriş veya `newgrp docker` |
| Dosya izinleri doğru ama `cat` yine `Permission denied` | Yoldaki bir dizinde `x` yok | 2.2.3 | `namei -l /tam/yol` |
| `chmod` yaptım ama erişim değişmedi | Çekirdek sahip üçlüsüne bakıyor, sen grup sandın | 2.2.1 | `stat -c '%A %U:%G'`; hangi sınıftayım? |
| `./script.sh: Permission denied`, `bash script.sh` çalışıyor | Dosyada `x` yok **ya da** mount `noexec` | 2.2.1 / 2.3.1 | `ls -l`; `findmnt -T` |
| Nginx `13: Permission denied`, dosya herkese açık | Ev dizini `750`, yol `x` vermiyor | 2.2.3 | `namei -l`; siteyi `/var/www`'ye taşı |
| SSH `Authentication refused: bad ownership or modes` | `.ssh` 700 / `authorized_keys` 600 değil | 2.5.1 | `ls -ld ~/.ssh`; StrictModes |
| `chown: Operation not permitted` (EPERM, EACCES değil) | Sahiplik değiştirmek root/capability ister | 2.2.2, 2.5.3 | `id`; `sudo` gerekli mi? |
| `Read-only file system` (EROFS) | Mount `ro`; izin bit'leriyle ilgisi yok | 2.5.3 | `findmnt -T`; neden `ro`? |
| Beklenmedik setuid binary; kullanıcılar root oluyor | Yanlış yere konmuş setuid | 2.3.2 | `find / -perm -4000`; baseline farkı |
| IAM'de admin'im ama sunucuda denied | OS izinleri ≠ IAM; farklı uygulayıcılar | 2.5.2 | `id`; OS'ta `sudo` yetkim var mı? |
| Ortak dizinde herkes birbirinin dosyasını siliyor | Sticky bit yok | 2.3.1 | `ls -ld`; `chmod +t` |

> **Bu tablodan çıkarılacak ders:** "Permission denied" tek bir hata değil, en az beş farklı
> mekanizmanın ortak yüzüdür — yol üzerinde `x` eksikliği, yanlış izin sınıfı, henüz aktif olmayan
> grup, mount kısıtı ve OS-dışı bir uygulayıcı (IAM, StrictModes). Doğru refleks izinleri rastgele
> gevşetmek (`chmod 777`) değil, 2.5.3'teki karar ağacını sırayla yürüyüp **hangi** mekanizmanın
> reddettiğini bulmaktır. Sebebi bulunca çözüm neredeyse her zaman tek ve dar bir düzeltmedir.

---
---

# Faz 2 — Düşün Sorularının Cevapları

## Cevap 2.1 — Gruba eklendim ama hâlâ erişemiyorum

**Soru:** `sudo usermod -aG docker ubuntu` çalıştırdın, ama aynı terminalde `docker ps` hâlâ
`permission denied` veriyor. Neden? `id` ne gösterir? İki farklı çözüm nedir? Neden `tmux`
oturumunu yeniden bağlamak da işe yaramaz?

**Kimlik oturum açışında sabitlenir.** Bir process'in UID ve grup listesi, o process başlatılırken
(login sırasında sshd tarafından) belirlenir ve `fork` ile çocuklarına **miras** kalır. Zaten
çalışan bir kabuğun grup listesini `usermod` **geriye dönük değiştiremez** — yeni `docker` grubu
`/etc/group`'ta yazılıdır ama senin çalışan kabuğun onu okumamıştır.

**`id` iki farklı şey gösterebilir:**

```
$ id                    # çalışan kabuk: eski grup listesi, docker YOK
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),27(sudo)
$ id ubuntu             # /etc/group'tan taze okur: docker VAR
uid=1000(ubuntu) ... groups=...,27(sudo),988(docker)
```

Aradaki fark tam olarak sorunun kaynağıdır: dosyada var, çalışan process'te yok.

**İki çözüm:**
1. **Yeniden giriş yap** (SSH oturumunu kapat-aç). sshd yeni bir kabuğu güncel grup listesiyle
   başlatır. En temiz çözüm.
2. **`newgrp docker`** — mevcut kabukta yeni bir grup bağlamı açar; anında etki eder. (Veya `su -
   ubuntu` ile taze bir giriş kabuğu.)

**Neden tmux reattach işe yaramaz?** Çünkü `tmux attach` yeni bir kimlik oluşturmaz; sadece zaten
çalışan **tmux server**'a bağlanır. O server sen ilk `tmux` yazdığında başladı ve o anki (eski)
grup listesini taşıyor. Server'ı öldürüp (`tmux kill-server`) yeniden başlatana kadar eski
kimlik yaşar.

**Bir uyarı:** `docker` grubu pratikte **root ile eşdeğerdir** — docker soketine erişen herkes
bir konteyneri host'un köküne mount edip root olabilir. "Sadece docker çalıştırsın diye" gruba
eklemek, sessizce tam yetki vermektir.

**İlgili bölüm:** 2.1.1 · **Devamı:** 2.4.2 (least privilege), Faz 8 (docker)

---

## Cevap 2.2 — Diskteki dosyalar yanlış kullanıcıda görünüyor

**Soru:** Bir EBS diski Amazon Linux'tan (orada `appuser` UID 1001'di) çıkarıp Ubuntu'ya taktın.
Ubuntu'da önce `deploy` kullanıcısı oluşturulmuş (o 1001 olmuş), `appuser` sonradan gelmiş (1002).
Şimdi `ls -l` diskteki dosyaları `deploy`'a ait gösteriyor. Neden? Nasıl düzeltirsin?

**inode kullanıcı adını değil, sayıyı saklar.** Dosya sisteminde her dosyanın sahibi bir **UID
sayısı** olarak yazılıdır, "appuser" metni olarak değil. Diskteki dosyalar `1001` UID'sini taşıyor.
Amazon Linux'ta `1001 = appuser`'dı; Ubuntu'da `1001 = deploy`. `ls -l`, `1001`'i Ubuntu'nun
`/etc/passwd`'ine sorup "deploy" yazdırıyor. Dosyalar değişmedi — sadece **aynı sayının farklı
makinelerde farklı isme çözülmesi** söz konusu.

```
$ ls -ln /mnt/data     # -n: isim yerine sayı göster
-rw-r--r-- 1 1001 1001 ... rapor.csv     ← sayı hep 1001
$ ls -l /mnt/data
-rw-r--r-- 1 deploy deploy ... rapor.csv ← Ubuntu 1001'i "deploy" çözüyor
```

**İki çözüm:**
1. **Dosyaların sahibini düzelt:** `sudo chown -R appuser:appuser /mnt/data`. Dosyaları yeni
   makinedeki doğru UID'ye (1002) yeniden yazar. Diskin bu makineye ait olması amaçlanıyorsa
   doğru yol budur.
2. **UID'leri eşle:** Kullanıcıyı baştan doğru numarayla oluştur — `useradd -u 1001 appuser`.
   Disk birden çok makine arasında taşınıyorsa veya NFS ile paylaşılıyorsa, UID'leri her yerde
   tutarlı tutmak asıl çözümdür.

**İlgili bölüm:** 2.1.1, 2.1.2 · **Devamı:** Faz 6 (dosya sistemleri, mount)

---

## Cevap 2.3 — Nginx 403: ev dizinindeki site

**Soru:** Statik site `/home/ubuntu/site` altında, dosya `644`, dizin `775`, ama ev dizini `750`.
Nginx (`www-data`) `13: Permission denied` veriyor. Arkadaşın `chmod -R 777 /home/ubuntu` öneriyor.
Çekirdek nerede reddetti? `777` bir sonraki SSH girişinde ne bozar? İki doğru çözüm?

**Reddin yeri: yol üzerindeki `x`.** `index.html` herkese okunabilir (`r--` diğerleri için), ama
Nginx'in ona ulaşmak için önce yolu yürümesi gerekir: `/` → `home` → `ubuntu` → `site` →
`index.html`. `/home/ubuntu` dizini `750` (`drwxr-x---`): sahip `ubuntu`, grup `ubuntu`, **diğerleri
için hiçbir şey.** `www-data` ne sahip ne de `ubuntu` grubunda, yani "diğer" sınıfına düşüyor ve
`x` bulamıyor. Çekirdek daha `site` dizinine ulaşamadan, `/home/ubuntu` adımında **traverse**
iznini reddediyor (2.2.3). Dosyanın kendi izni ne kadar açık olursa olsun, yol kapalı.

**`chmod -R 777 /home/ubuntu` neyi bozar:** Ev dizinini ve altındaki `~/.ssh`'i herkese
yazılabilir yapar. Bir sonraki SSH girişinde sshd `StrictModes` kontrolünü çalıştırır, `.ssh`'in
(veya ev dizininin) gevşek iznini görür ve anahtarı **reddeder** — `Authentication refused: bad
ownership or modes` (2.5.1). Yani siteyi açayım derken kendini sunucudan **kilitlersin.** Üstelik
`-R` ev dizinindeki tüm özel anahtarları, `.aws/credentials`'ı ve her şeyi tüm kullanıcılara
okunur yapmış olur.

**İki doğru çözüm:**
1. **Siteyi doğru yere taşı:** `/var/www/site`. Bu dizin zaten web içeriği için tasarlanmıştır;
   yol boyunca `x` açıktır ve içeriği `www-data`'ya (veya uygun bir gruba) verirsin. Standart ve
   temiz çözüm.
2. **Yola traverse izni ver:** Siteyi ev dizininde tutman şartsa, sadece **traverse** için
   `chmod o+x /home/ubuntu` (veya daha iyisi Nginx'i `ubuntu` grubuna alıp `chmod g+x`). Bu, ev
   dizinini listelenebilir yapmaz (`r` vermez), sadece içinden geçmeye izin verir. `-R` yok, `777`
   yok.

**İlgili bölüm:** 2.2.3 · **Devamı:** 2.2.2 (neden 777 kötü), 2.5.1 (StrictModes)

---

## Cevap 2.4 — Ortak çalışma dizini tasarımı

**Soru:** `/srv/raporlar`'da `ayse`, `mehmet`, `deploy` ortak yazacak, birbirlerinin dosyalarını
okuyup güncelleyebilecek ama **silemeyecek.** `chmod -R 777` neyi bozdu? Doğru octal ve özel
bit(ler) ne?

**`777` neyi bozdu:** Yazma ve okumayı çözdü ama **silmeyi kısıtlayamadı.** 2.2.3'te gördük: bir
dosyayı silmek dosyanın değil **dizinin** yazma iznine bağlıdır. `777` dizinde herkese `w` verdiği
için, herkes herkesin dosyasını silebilir — tam da istenmeyen şey. Ayrıca `777` makinedeki her
process'e (ele geçirilmiş bir servis dahil) bu dizine yazma verir.

**Doğru tasarım — üç parça:**
1. **Grup sahipliği:** Dizini `raporcu` grubuna ver: `sudo chgrp raporcu /srv/raporlar`. Üç
   kullanıcı da bu grupta.
2. **setgid (2000):** Yeni dosyalar oluşturanın birincil grubunu değil, dizinin grubunu (`raporcu`)
   miras alsın diye. Böylece `ayse`'nin oluşturduğu dosya otomatik `raporcu` grubuna ait olur ve
   `mehmet` grup üzerinden okuyup yazabilir.
3. **sticky (1000):** Herkes yeni dosya oluşturabilsin ama sadece **sahibi** kendi dosyasını
   silebilsin diye.

**Octal:** `chmod 3770 /srv/raporlar` — yani `2000 + 1000 + 770`:
- `770`: sahip ve grup tam yetki (`rwx`), diğerleri hiçbir şey.
- `2000`: setgid → grup mirası.
- `1000`: sticky → silme koruması.

```
$ sudo chgrp raporcu /srv/raporlar
$ sudo chmod 3770 /srv/raporlar
$ ls -ld /srv/raporlar
drwxrws--T 2 root raporcu 4096 Sep 15 10:00 /srv/raporlar
```

Grup üçlüsünde `s` (setgid), diğerler üçlüsünde `T` (sticky, `x` olmadan çünkü diğerlerine erişim
yok). Artık grup üyeleri serbestçe çalışır, kimse başkasının raporunu silemez, gruba girmeyen
kimse dizine dokunamaz.

**İlgili bölüm:** 2.3.1 · **Devamı:** 2.2.3 (silme = dizin işlemi)

---

## Cevap 2.5 — Kısıtlı sudo'dan tam root'a kaçış

**Soru:** `deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart app, /usr/bin/vim /etc/app/config.yml`.
`deploy` bu iki komutla nasıl tam root olur? Hangi yanılgının somut hâli? Nasıl düzeltirsin?

**Kaçış: `vim`'in içinden kabuk.** `sudo vim /etc/app/config.yml` çalıştığında `vim` root efektif
UID'siyle açılır. `vim` içinde ise kabuk çalıştırma komutu vardır:

```
$ sudo vim /etc/app/config.yml
:!sh
# id
uid=0(root) gid=0(root) groups=0(root)
```

`:!sh` (veya `:!bash`), `vim`'in root process'inin içinden bir kabuk başlatır; o kabuk root olur.
`deploy` artık config'i düzenlemekle sınırlı değil — **tam root.** Aynı kaçış `systemctl` için de
mümkündür: uzun çıktı bir pager'da (`less`) açılırsa, `less` içinden `!sh` yazılabilir (bu yüzden
`sudo systemctl` için de `SYSTEMD_PAGER` dikkate alınır).

**Hangi yanılgı:** 2.4.1'deki *"tek bir komuta sudo izni vermek güvenlidir"* yanılgısının tam
karşılığı. Güvenlik, verilen komutun **içinden başka komut çalıştırılıp çalıştırılamadığına**
bağlıdır. `vim`, `less`, `find`, `awk`, `tar` — hepsi kabuğa kaçış sunar; bunlara verilen dar
görünümlü bir sudo izni aslında sınırsızdır.

**Doğru düzeltme — `sudoedit`:**

```
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart app, sudoedit /etc/app/config.yml
```

`sudoedit` (veya `sudo -e`) dosyanın bir **kopyasını** `deploy`'un **kendi** yetkisiyle, kendi
editöründe açar; `deploy` kaydedince `sudoedit` root olarak kopyayı yerine yazar. Editör hiçbir
zaman root çalışmaz, dolayısıyla `:!sh` yazsa bile açılan kabuk `deploy` kalır. Kullanıcı
editörünü `EDITOR` ile kendi seçer ve root süreç içinde hiç kod çalıştırmaz.

**İlgili bölüm:** 2.4.1, 2.4.2 · **Devamı:** Faz 9 (sertleştirme)

---

## Cevap 2.6 — `./backup.sh` denied ama `bash backup.sh` çalışıyor

**Soru:** `/data/scripts/backup.sh` `755`, sen sahipsin, yol açık. `./backup.sh` → `Permission
denied`, ama `bash backup.sh` çalışıyor. Neden? Karar ağacında hangi adım yakalar? Kalıcı çözüm?

**Fark: çalıştırma (`execve`) ile okuma.** `./backup.sh` yazdığında çekirdek dosyayı **çalıştırmak**
ister — `execve` syscall'ı çağrılır. `bash backup.sh` yazdığında ise çalıştırılan program
`bash`'tir; `backup.sh` sadece `bash`'in **okuduğu** bir veri dosyasıdır. Yani ilk yol dosyanın
"çalıştırılabilir" olmasını gerektirir, ikincisi sadece "okunabilir" olmasını.

Dosyada `x` var (`755`), sahipsin — o zaman `execve` neden reddedildi? Çünkü izin bitleri tek
kontrol değildir: **dosyanın bulunduğu mount `noexec` ile bağlanmışsa**, o dosya sistemindeki
hiçbir dosya çalıştırılamaz, bit'ler ne olursa olsun. `bash backup.sh` çalışır çünkü orada
çalıştırılan `bash`'tir (o `/usr/bin`'de, exec'e izin veren bir mount'ta); `noexec` mount'taki
dosya sadece okunur — ki bu izin veriliyor.

**Karar ağacındaki adım: 5 — dosya sistemi izin veriyor mu?** Komut:

```
$ findmnt -T /data/scripts/backup.sh
TARGET SOURCE      FSTYPE OPTIONS
/data  /dev/nvme1n1 ext4  rw,nosuid,nodev,noexec,relatime
                                        ^^^^^^ suçlu
```

`noexec` bayrağı doğrudan görünüyor. (Bu, `EACCES` verir ama izin bit'leriyle ilgisi yoktur —
2.5.3'teki errno okuma alışkanlığı burada seni yol yerine mount'a yönlendirir.)

**Kalıcı çözümler:**
1. **Script'i exec'e izin veren bir yere taşı:** `/usr/local/bin` veya `/opt`. Ayrı veri diskleri
   çoğu zaman kasıtlı olarak `noexec`'tir (güvenlik sertleştirmesi); script'ler oraya değil, kod
   dizinlerine konur. Tercih edilen çözüm.
2. **`bash backup.sh` ile çağır:** Yorumlayıcıyı açıkça vererek `noexec`'i atlarsın. Hızlı ama
   `noexec`'in var olma nedenini (o diskte bırakılan bir şeyin çalıştırılmasını önlemek) zayıflatır.
3. **Mount'u `exec` yap:** `/etc/fstab`'da `noexec`'i kaldırıp yeniden mount et — ama önce o diskin
   neden `noexec` kurulduğunu sorgula; genellikle kasıtlıdır ve dokunmamak gerekir.

**İlgili bölüm:** 2.5.3, 2.3.1 · **Devamı:** Faz 6 (fstab, mount seçenekleri), Faz 9 (noexec sertleştirme)

---
---

# Faz 2 — Sık Sorulan Sorular

### S1. `sudo`, `sudo -i`, `sudo -s` ve `su -` arasındaki fark ne?

- `sudo <komut>`: tek komutu root olarak çalıştırır, kendi bulunduğun ortamda kalırsın.
- `sudo -s`: root olarak bir kabuk açar ama **senin** ortam değişkenlerini ve `$HOME`'unu korur.
- `sudo -i`: root'un tam **giriş** kabuğunu açar — root'un `~/.bashrc`'si okunur, `$HOME` `/root`
  olur, temiz bir root ortamı gibi davranır. Günlük yönetim için en temizi budur.
- `su -`: root **parolasını** ister (Ubuntu/EC2'de bu parola kilitli olduğu için genelde çalışmaz)
  ve root'un giriş kabuğunu açar. `sudo -i` bunun parola-yerine-sudoers ile çalışan modern
  karşılığıdır.

### S2. Bu makinede `sudo` biraz farklı davranıyor — `sudo-rs` de ne?

Bazı modern dağıtımlarda (bu belgeyi hazırladığımız makine dahil) `/usr/bin/sudo`, klasik
`sudo.ws` yerine **`sudo-rs`** (Rust ile yeniden yazılmış sudo) olabilir. Davranış ve `sudoers`
sözdizimi büyük ölçüde aynıdır; fark çoğunlukla hata mesajlarının biçimindedir (örneğin bazı
hataların sonuna `(os error 1)` eklenmesi). Referans dağıtımlarımızda (Ubuntu 24.04, Amazon Linux
2023) hâlâ klasik `sudo` vardır; bu fazdaki her şey iki uygulamada da geçerlidir.

### S3. Standart izinler yetmiyor; bir dosyayı sadece **bir** kullanıcıya açmam gerekiyor. Nasıl?

Bunun için **ACL** (Access Control List) vardır. Klasik "sahip/grup/diğer" modeli üç sınıfla
sınırlıdır; ACL ise dosya bazında tek tek kullanıcı/grup kuralı eklemene izin verir:

```
$ setfacl -m u:ayse:rw rapor.txt      # sadece ayse'ye okuma+yazma
$ getfacl rapor.txt                    # kuralları listele
$ ls -l rapor.txt                      # izin sonunda '+' → ACL var
-rw-rw-r--+ 1 ubuntu ubuntu ...
```

`ls -l`'de iznin sonundaki `+` ACL olduğunun işaretidir. Ayrıntısı Faz 9'da; ama karar ağacında
(2.5.3) `+` gördüğünde `getfacl`'a bakmayı unutma.

### S4. Ubuntu'da root'un parolası ne? Neden `su` çalışmıyor?

Ubuntu (ve EC2 imajları) root hesabını **kilitli** kurar: `/etc/shadow`'da root satırının parola
alanı `!`'tir, yani hiçbir parola eşleşmez. Bu kasıtlıdır — root'a doğrudan giriş yerine her şey
`sudo` üzerinden, iz bırakarak yapılır. `su -` bu yüzden "Authentication failure" verir. Gerçekten
root parolası kurmak istersen `sudo passwd root` (çoğu bulut kurulumunda önerilmez).

### S5. `ping` neden setuid değil ama yine çalışıyor? Hani düşük seviye işlemler root ister?

Eskiden `ping` setuid'ti çünkü ICMP için ham (raw) soket açması gerekiyordu ve bu root işiydi.
Modern Linux'ta bunun yerine bir çekirdek ayarı var:

```
$ ls -l /usr/bin/ping           # setuid YOK
-rwxr-xr-x 1 root root ... /usr/bin/ping
$ sysctl net.ipv4.ping_group_range
net.ipv4.ping_group_range = 0 2147483647
```

`ping_group_range` bu aralıktaki GID'lere ICMP soketi açma izni verir; aralık tüm grupları
kapsadığı için herkes `ping` atabilir, setuid'e gerek kalmaz. Bu, "root gerektiren işi setuid
yerine dar bir çekirdek yeteneğine bağlamak" deseninin güzel bir örneğidir (setcap/capability
mantığı, Faz 9).

### S6. IAM'de admin'im ama sunucuda `sudo` çalışmıyor. Bu nasıl mümkün?

Çünkü IAM ve OS izinleri **ayrı dünyalardır** (2.5.2). IAM admin'liğin AWS API'lerinde geçerli —
instance başlatabilir, S3'e yazabilirsin. Ama instance'ın **içinde** sıradan bir Linux
kullanıcısısın; `sudo` yetkisi `/etc/sudoers`'a bağlıdır, IAM'e değil. IAM admin'liğini OS
yetkisine çevirmenin yolu, SSM Session Manager veya Instance Connect ile makineye girip oradaki
`sudo` grubuna dahil olmaktır — yoksa IAM'deki gücün OS'un içine sızmaz.

### S7. Yeni kullanıcı için `adduser` mı `useradd` mı? Ve neden `777` çözüm değil?

- `useradd` düşük seviyeli, minimal komuttur — ev dizini bile oluşturmaz (`-m` gerekir), parola
  ayarlamaz. Script'ler ve servis kullanıcıları (`--system`) için idealdir.
- `adduser` (Debian/Ubuntu) `useradd` üstüne kurulu, interaktif ve dostane bir sarmalayıcıdır:
  ev dizinini oluşturur, parola sorar, varsayılan grupları ayarlar. İnsan kullanıcılar için
  önerilir.
- **`777` neden çözüm değil:** Bir erişim sorununu `chmod 777` ile "çözmek", hangi kullanıcının
  hangi kontrolde reddedildiğini hiç öğrenmeden makinedeki **her process'e** tam yetki vermektir
  (2.2.2). Doğru soru her zaman aynıdır: *kim, neye, hangi izinle* erişmeli? Bu fazın tüm araçları
  (grup, setgid, ACL, sudoers) bu soruyu **dar** biçimde cevaplamak içindir; `777` ise soruyu
  cevaplamak yerine yok sayar.

---
---

# Faz 2 — Kendini Sına

Aşağıdaki 18 soruyu cevapla. Cevap anahtarı ve puanlama hemen altındadır. Amaç ezber değil,
mekanizmayı muhakeme edebilmek.

## Bölüm A — Temel (1–6)

**1.** `-rwxr-x---` izin dizesinin octal karşılığı nedir?

**2.** `ls -l` çıktısında bir satır `l` ile başlıyorsa dosya türü nedir?

**3.** UID 0 hangi kullanıcıya aittir ve bu kullanıcının izin kontrollerindeki özelliği nedir?

**4.** Kullanıcı hesaplarının parola hash'leri hangi dosyada tutulur ve bu dosyanın izni tipik
olarak nedir?

**5.** `chmod 640 dosya` komutundan sonra grubun bu dosya üzerindeki hakları nelerdir?

**6.** `umask 022` iken oluşturulan yeni bir **dosyanın** izni ne olur?

## Bölüm B — Mekanizma (7–12)

**7.** Bir process bir dosyanın sahibiyse ama sahip üçlüsü `---`, grup üçlüsü `rwx` ise ve process
o grubun üyesiyse — dosyayı okuyabilir mi? Neden?

**8.** Bir dizinde sadece `x` izni (`--x`) olan bir kullanıcı, ismini bildiği bir dosyayı okuyabilir
mi? Aynı dizini `ls` ile listeleyebilir mi?

**9.** `444` (salt okunur) bir dosya, yazma izni olan bir dizinde duruyor. Başka bir kullanıcı bu
dosyayı silebilir mi? Neden?

**10.** `passwd` komutu, normal kullanıcı `/etc/shadow`'u okuyamazken o dosyaya nasıl yazabiliyor?
Hangi bit ve hangi UID kavramı devrede?

**11.** setgid biti bir **dizine** konduğunda ne etki yapar?

**12.** `sudo` ile `su -` arasındaki en temel iki fark nedir (parola ve kapsam açısından)?

## Bölüm C — Uygulama ve muhakeme (13–18)

**13.** Bir kullanıcıya `sudo usermod -aG docker kullanici` yaptın ama `docker ps` hâlâ reddediyor.
Kullanıcı çıkış-giriş yapmadan bunu nasıl çözebilir ve neden gerekli?

**14.** Bir dosya `644` ve herkese okunabilir, ama `cat /a/b/c/dosya` yine `Permission denied`
veriyor. Nedeni bulmak için hangi tek komutu çalıştırırsın?

**15.** `sudo vim /etc/app.conf` iznine sahip bir kullanıcı nasıl tam root olur? Doğru düzeltme
nedir?

**16.** Üç kişilik bir ekip için ortak bir dizin kuruyorsun: herkes yazsın, birbirinin dosyasını
okusun/güncellesin, ama sadece sahibi silsin. Hangi octal modu kullanırsın?

**17.** IAM'de admin olan biri, EC2 instance'ında `/etc/nginx/nginx.conf`'u düzenlemeye çalışırken
`Permission denied` alıyor. Bu neden IAM ile çelişmiyor?

**18.** `./deploy.sh` (izni `755`, sahibisin) `Permission denied` veriyor ama `bash deploy.sh`
çalışıyor. Muhtemel neden nedir ve hangi komutla doğrularsın?

---

## Cevap Anahtarı

**1.** `750` — rwx=7, r-x=5, ---=0. · *2.2.1*

**2.** Sembolik link (symlink). · *2.2.1*

**3.** root. İzin kontrollerinde okuma ve yazma bit'lerini atlar (çalıştırma için dosyada en az bir
`x` biti gerekir). · *2.1.1, 2.2.1*

**4.** `/etc/shadow`; izni `640` (`rw-r-----`), sahip `root`, grup `shadow`. · *2.1.2*

**5.** Grup sadece okuyabilir (`r--`); yazamaz, çalıştıramaz. · *2.2.1, 2.2.2*

**6.** `644` (`rw-r--r--`). Program `666` ister, umask `022` `w` bit'lerini gruptan ve diğerinden
düşürür. · *2.2.2*

**7.** Okuyamaz. Çekirdek "sahip mi?" sorusuna **evet** alır almaz sadece **sahip üçlüsüne**
bakar; `---` olduğu için reddeder. Grup üçlüsüne hiç bakmaz — izinler toplanmaz. · *2.2.1*

**8.** İsmini bildiği dosyayı **okuyabilir** (`x` = traverse). Ama `ls` ile **listeleyemez** —
listeleme `r` gerektirir. · *2.2.3*

**9.** Evet, silebilir. Silme dosyanın değil, **dizinin** `w` (+`x`) iznine bağlıdır; dosyanın
`444` olması içeriğini korur, varlığını değil. (Sticky bit olmadıkça.) · *2.2.3, 2.3.1*

**10.** `passwd` bir **setuid** binary'dir (`-rwsr-xr-x root root`). Çalıştırıldığında **efektif
UID**'si dosyanın sahibi olan root'a yükselir; kontroller efektif UID'ye baktığı için `/etc/shadow`'a
yazabilir. · *2.3.1*

**11.** O dizinde oluşturulan yeni dosya ve dizinler, oluşturanın birincil grubunu değil,
**dizinin grubunu** miras alır (ve alt dizinler setgid bit'ini de devralır). Ortak çalışma
dizinleri için kullanılır. · *2.3.1*

**12.** (a) Parola: `sudo` **kendi** parolanı ister, `su -` **root'un** parolasını. (b) Kapsam:
`sudo` tek komutu (veya oturumu) verir ve loglar; `su -` tam ve süresiz bir root kabuğu açar. ·
*2.4.1*

**13.** `newgrp docker` (veya `su - kullanici`) ile mevcut kabukta yeni grup bağlamı açar. Gerekli,
çünkü grup listesi oturum açılışında sabitlenir; çalışan kabuk yeni grubu görmez. · *2.1.1, Cevap 2.1*

**14.** `namei -l /a/b/c/dosya` — yolun her dizinindeki izni gösterir; `x` eksik olan dizin
suçludur. · *2.2.3, 2.5.3*

**15.** `vim` root olarak açıldığı için `:!sh` yazıp root kabuğu alır. Düzeltme: `sudoedit`
(`sudo -e`) kullan — editör kullanıcının kendi yetkisiyle çalışır, root süreçte hiç kod
çalışmaz. · *2.4.1, Cevap 2.5*

**16.** `3770` (setgid 2000 + sticky 1000 + rwx grup, diğerine hiçbir şey), dizin `raporcu` grubuna
ait olacak şekilde. · *2.3.1, Cevap 2.4*

**17.** IAM AWS **API'lerini** yönetir; instance'ın **içinde** kullanıcı sıradan bir Linux
kimliğidir. Dosyayı düzenlemek OS `sudo` yetkisi ister, IAM policy değil — iki ayrı uygulayıcı. ·
*2.5.2*

**18.** Dosyanın bulunduğu mount muhtemelen `noexec`'tir; `execve` reddedilir ama `bash`'in dosyayı
**okuması** serbesttir. `findmnt -T deploy.sh` ile doğrularsın. · *2.5.3, Cevap 2.6*

---

## Puanlama

| Doğru sayısı | Değerlendirme |
|---|---|
| 16–18 | Erişim modelini içselleştirmişsin. Faz 3'e geçebilirsin. |
| 12–15 | Sağlam. Kaçırdığın soruların bölümlerine bir tur dön. |
| 8–11 | Temel oturmuş ama mekanizma boşlukları var. Aşağıdaki tabloyu kullan. |
| 0–7 | Fazı baştan, 🔧 kutularını makinende çalıştırarak tekrar et. |

**Hangi soruyu kaçırırsan hangi bölüme dön:**

| Kaçırdığın soru | Dön |
|---|---|
| 1, 5, 6 | 2.2.1–2.2.2 (izin bit'leri, chmod, umask) |
| 2, 4 | 2.1.2, 2.2.1 (kimlik dosyaları, dosya türleri) |
| 3, 10 | 2.1.1, 2.3.1 (root, setuid, efektif UID) |
| 7, 8, 9 | 2.2.1, 2.2.3 (üçlü seçimi, dizin bit'leri) |
| 11, 16 | 2.3.1 (setgid, sticky) |
| 12, 15 | 2.4.1 (sudo, sudoers, kabuğa kaçış) |
| 13 | 2.1.1 (kimliğin oturumda sabitlenmesi) |
| 14, 18 | 2.2.3, 2.5.3 (yol traverse, noexec, karar ağacı) |
| 17 | 2.5.2 (OS izinleri ≠ IAM) |

---
---

# Faz 2 — Kapanış ve Faz 3'e Köprü

## Bu fazdan ne taşıyorsun

Faz 1'de "her şey dosyadır" ve süreçlerin nasıl doğduğunu gördün. Faz 2 buna **kim** sorusunu
ekledi: her dosyanın ve her process'in bir kimliği var, ve çekirdek bu kimliğe bakarak her erişimi
tek tek onaylıyor. Yanında dört araç götürüyorsun:

1. **Kimlik = sayı.** UID/GID, root'un `0` olması, kimliğin oturumda sabitlenip `fork` ile miras
   kalması. "Gruba ekledim ama çalışmıyor" artık gizem değil.
2. **Üçlü seçimi.** Çekirdek sahip → grup → diğer sırasıyla **tek** üçlü seçer ve sadece ona bakar;
   izinler toplanmaz. `chmod`, octal, umask ve dizin bit'leri (traverse, silme) bu modelin üstünde.
3. **Yetki yükseltme.** setuid/setgid/sticky ve `sudo`/`sudoers` — yetkiyi **dar** ve **izlenebilir**
   biçimde vermek; `chmod 777` ve "uygulamayı root çalıştır" tuzaklarından kaçınmak.
4. **"Permission denied" karar ağacı.** Kim → yol traverse → izin sınıfı → grup aktif mi → mount →
   ACL/MAC → özel durum. Ve errno okuma: EACCES ≠ EPERM ≠ EROFS. Ve bulutta OS ≠ IAM.

## Faz 3 bunun neresine bağlanıyor

Faz 3 process'leri ve kaynak kontrolünü derinleştirir. Faz 2'nin kimlik ve yetki modeli oraya
doğrudan bağlanır:

| Faz 2'de öğrendiğin | Faz 3'te üstüne konacak |
|---|---|
| Her process'in bir UID/GID'si var (kimlik) | Her process'in bir PID'si, ebeveyni ve durumu var; `ps`, `/proc` |
| setuid ile efektif UID değişir | Process kredileri, `nice`/öncelik, kaynak limitleri (`ulimit`) |
| Uygulamayı yetkisiz kullanıcıyla çalıştırmak (least privilege) | Servisleri cgroup'larla izole etmek, kaynak sınırlamak |
| `sudo` bir process'i farklı kimlikle başlatır | Sinyaller, `kill`, process ağacı ve yetim/zombi süreçler |

> **Faz 2 çıktısı — devam etmeden önce kendine sor:**
> Bir sunucuda şu dosyayı görüyorsun:
>
> ```
> -rwxr-x--- 1 root devops 1832 Sep 15 09:12 deploy.sh
> ```
>
> 1. Kimler bu script'i **çalıştırabilir**? Kimler **okuyabilir**? Kimler **değiştirebilir**?
> 2. `devops` grubunda olmayan `alice` kullanıcısı bu dosyayla ne yapabilir? Ya bulunduğu dizinde
>    yazma izni varsa?
> 3. `alice`'in bu script'i çalıştırabilmesini istiyorsun ama koda dokunmasını istemiyorsun.
>    En dar çözüm nedir — grup mu, ACL mi, sudoers mı? Hangi durumda hangisi?
>
> Bu üç soruyu tereddütsüz yanıtlayabiliyorsan, erişim modeli oturmuş demektir.
>
> **Lab 2 fikri:** Kendi makinende (veya bir test instance'ında) `raporcu` grubu, `ayse`/`mehmet`
> test kullanıcıları oluştur; `/srv/raporlar`'ı Cevap 2.4'teki `3770` + setgid + sticky ile kur;
> `sudo -u ayse` ve `sudo -u mehmet` ile dosya oluşturup silmeyi dene ve tasarımın gerçekten
> beklediğin gibi davrandığını **gör.** Bitince `userdel -r` ve `groupdel` ile temizle.

---

> **Navigasyon:** [◀ Faz 1 — Shell ve Dosya Sistemi](Faz_1_Shell_ve_Dosya_Sistemi.md) · **Faz 2** · [Ara Sınav 1 ▶](Ara_Sinav_1.md)