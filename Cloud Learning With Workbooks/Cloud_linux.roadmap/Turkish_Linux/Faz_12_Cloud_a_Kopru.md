# Faz 12 — Cloud'a Köprü: Boş Bir AMI'den Production'a

> **Navigasyon:** [◀ Faz 11 — Gözlemlenebilirlik ve Troubleshooting](Faz_11_Gozlemlenebilirlik_ve_Troubleshooting.md) · **Faz 12** · [Ek A — Komut Sözlüğü ▶](EK_A_Komut_Sozlugu.md)

---

## Nereden geliyoruz

Faz 0'dan Faz 11'e kadar her fazda **tek bir Linux temeli** öğrendin: zihinsel model, shell, izinler, process,
bellek, boot, depolama, ağ, paket, güvenlik, otomasyon, teşhis. Her biri kendi başına bir taş. Faz 12 bu
taşların hepsini **tek bir kemere** örer. Çünkü bir bulut mühendisi olarak işin, bu temellerin her birini
AWS'teki karşılığına oturtmak ve boş bir makinenin production'a dönüşen yolculuğunun her adımını **neden orada
olduğunu** Linux temellerine dayanarak anlatabilmektir.

Bu faz yeni bir konu öğretmez — öğrendiğin her şeyi **bir hikâyede birleştirir**. Bu yüzden bu fazın "nereden
geliyoruz"u tek bir faz değil, **tüm yolculuktur**:

- **Faz 0 + 8** → AMI (donmuş bir Linux)
- **Faz 5 + 10** → cloud-init / user-data (boot-time yapılandırma)
- **Faz 7** → SSH keypair / SSM (erişim)
- **Faz 9** → IAM instance role (diskte key tutmadan yetki)
- **Faz 6** → EBS + fstab/UUID (kalıcı depolama)
- **Faz 5 + 8** → systemd service (uygulamanın yaşam döngüsü)
- **Faz 4 + 11** → CloudWatch agent / Logs (uzaktan gözlemlenebilirlik)
- **Faz 3.6** → cgroup + namespace → ECS/EKS (kernel paylaşan Linux)
- **Faz 7 + 9** → Security Group vs host firewall / AppArmor (iki savunma katmanı)

## Bu fazın sorusu

Faz 0-11 boyunca hep "bu Linux temeli **nasıl** çalışır" dedik. Faz 12 son ve en birleştirici soruyu sorar:

> *"Boş bir Ubuntu AMI'yi alıp production'da çalışan bir servise dönüştürürken — cloud-init → kullanıcı/SSH →
> EBS mount → paket → systemd servis → log/monitoring → hardening — her adım **hangi Linux temeline** dayanıyor
> ve **neden** tam orada olmak zorunda? Bir AWS kavramını (AMI, user-data, IAM role, SG) gördüğümde, altında
> hangi Linux gerçeğinin yattığını söyleyebiliyor muyum?"*

Bu fazın merkezinde tek bir fikir var: **AWS soyutlamaları Linux'un üzerine kuruludur.** AMI donmuş bir Linux'tur,
Lambda görmediğin bir Linux'tur, ECS container kernel paylaşan bir Linux'tur. Bir bulut mühendisini junior'dan
ayıran şey, bu soyutlamanın **altını görebilmektir**: bir instance "healthy ama uygulama yok" dediğinde panik
yapmaz, çünkü altındaki systemd servisini (Faz 8), bind adresini (Faz 7), IAM rolünü (Faz 9) okuyabilir.

Bu fazın sonunda: boş bir AMI'nin boot'tan production'a yolculuğunun her adımını Linux temeline bağlayabilecek;
her büyük AWS kavramının altındaki Linux gerçeğini söyleyebilecek; ve bir sorun çıktığında hangi katmana
ineceğini (bu yolculuğun hangi halkası) bileceksin.

---

## Bu fazın sonunda

- Boş bir Ubuntu AMI'nin **boot'tan production'a yolculuğunun** yedi adımını (cloud-init → kullanıcı/SSH → EBS
  mount → paket → systemd → log → hardening) sırayla anlatabileceksin
- Her adımın **hangi Linux temeline** dayandığını ve **neden** orada olduğunu söyleyebileceksin
- Büyük **AWS kavramlarının** altındaki Linux gerçeğini açabileceksin: AMI, cloud-init, IAM role, EBS, systemd,
  CloudWatch, ECS/EKS, Lambda, SG
- **İki savunma katmanını** (Security Group vs host firewall/AppArmor) ayırt edebileceksin
- Bir sorun çıktığında bu yolculuğun **hangi halkasının koptuğunu** teşhis refleksinle (Faz 11) bulabileceksin

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden burada |
|---|---|---|---|
| 12.1 | AMI = donmuş bir Linux | `[uygulama]` | Faz 0 + 8: paket katmanı + zihinsel model AWS'e oturur |
| 12.2 | cloud-init / user-data ve IAM role | `[uygulama]` | Faz 5 + 10 + 9: boot-time yapılandırma + diskte key yok |
| 12.3 | EBS + fstab ve systemd servisi | `[uygulama]` | Faz 6 + 5 + 8: kalıcı depolama + uygulama yaşam döngüsü |
| 12.4 | İki savunma katmanı ve gözlemlenebilirlik | `[kavram]` | Faz 7 + 9 + 11: SG vs host, CloudWatch |
| 12.5 | Container ve Lambda — hâlâ Linux | `[kavram]` | Faz 3.6: cgroup/namespace → ECS/EKS; Lambda |
| 12.6 | Bu köprü bozulunca | — | Yolculuğun hangi halkası koparsa hangi belirti |

> **Bu faz nasıl çalışılır:** Bu faz bir **sentez** fazıdır — yeni komut ezberlemek yerine, öğrendiğin her şeyi
> bağlamak için çalışılır. Bu fazın en değerli alışkanlığı şudur: her AWS kavramını gördüğünde kendine sor
> "**bunun altındaki Linux gerçeği ne?**" AMI mı? → donmuş bir kök dosya sistemi (Faz 6) + kurulu paketler (Faz
> 8). Eğer bir AWS hesabına erişimin varsa (kendi test hesabın), bu fazı en iyi şekilde **gerçekten bir t2.micro
> başlatarak** çalışırsın: boş bir Ubuntu instance aç, içine SSH ile gir ve bu fazın yedi adımını sırayla elle
> yap — her adımda "bu hangi fazdı" diye kendine sorarak. Bu, tüm workbook'u tek bir pratikte toplar. Erişimin
> yoksa bile, yolculuğu kâğıt üzerinde adım adım anlatarak aynı sentezi yaparsın.

Bu fazın tamamı tek bir resimde: boş bir AMI'nin yedi adımda production'a dönüşen yolculuğu ve her adımın hangi
faza dayandığı.

![Şekil 12.1 — Boş bir Ubuntu AMI'den production'a yolculuk. Yedi aşamalı bir boru hattı, her aşama bir AWS adımı ve altında dayandığı Linux fazı: (1) cloud-init / user-data (Faz 5+10), (2) kullanıcı/SSH erişimi (Faz 7), (3) EBS mount + fstab (Faz 6), (4) paket kurulumu (Faz 8), (5) systemd servisi (Faz 5+8), (6) log/monitoring — CloudWatch (Faz 4+11), (7) hardening — SG + host firewall + IAM role + AppArmor (Faz 7+9) → production servis. Altta ders: her aşama AWS adı takmış bir Linux temelidir; bir aşama bozulunca "AWS"i değil, altındaki Linux katmanını debug eder, kanıtı orada bulursun (Faz 11).](../diagrams/png/lx-12-01-ami-to-production.png)

---
---

# 12.1 AMI = Donmuş Bir Linux

## 12.1.1 Bir AMI'nin altında ne var `[uygulama]`

Bir **AMI** (Amazon Machine Image — Amazon Makine İmajı) AWS'te bir instance başlatmak için kullandığın
şablondur. Ama altında yeni bir şey yoktur: AMI, **donmuş bir Linux kök dosya sistemidir.** İçinde ne var?

- Bir **kök dosya sistemi** (Faz 6) — `/`, `/etc`, `/usr`, `/var` dizin ağacı (Faz 1'in FHS'i)
- **Kurulu paketler** (Faz 8) — kernel, systemd, apt/dpkg veritabanı, önceden kurulmuş servisler
- Bir **kullanıcı/izin yapısı** (Faz 2) — `ubuntu` kullanıcısı, sudoers, SSH yapılandırması
- **Boot yapılandırması** (Faz 5) — GRUB, initramfs, systemd hedefleri

AMI'yi "başlattığında" AWS aslında bu donmuş dosya sistemini bir EBS diskine kopyalar (Faz 6) ve o diskten boot
eden bir sanal makine oluşturur. Yani AMI, Faz 8'in "immutable / baked image" fikrinin (donmuş, tekrar
üretilebilir bir sistem) AWS'teki tam karşılığıdır: sunucuyu elle yamamak yerine, her şeyi içine "pişirilmiş"
bir imaj alır ve ondan yeni makineler üretirsin.

> **💡 Cloud bağlantısı — kendi AMI'ni pişirmek:** Boş bir Ubuntu AMI ile başlarsın; ama production'da genellikle
> **kendi AMI'ni** üretirsin (Packer gibi araçlarla): boş AMI'yi başlat, üzerine paketlerini kur, hardening yap
> (Faz 9), sonra o hâlini bir AMI olarak "dondur." Bir sonraki instance bu donmuş hâlden **saniyeler içinde**
> ayağa kalkar — Faz 10'un "mutable yamama vs immutable yeniden inşa" felsefesi tam olarak budur. AMI, "bir kez
> doğru kur, sonra tekrar üret" fikrinin somut aracıdır.


> **🤔 Düşün 12.1** — X takımı boş bir Ubuntu AMI başlatıp paketlerini her seferinde SSH ile elle kuruyor. Y takımı bir kez kurup sertleştiriyor, sonra kendi AMI'sini pişiriyor. Autoscaling'in bir dakika içinde 10 instance eklemesi gerekiyor ve az önce kritik bir güvenlik yaması çıktı. (a) Hangi takım daha hızlı ölçeklenir ve neden? (b) Her takım yamayı nasıl uygular? (c) Bu, önceki fazlardaki hangi fikirdir?
>
> *(Cevap: fazın sonunda)*
---
---

# 12.2 cloud-init / user-data ve IAM Role

## 12.2.1 cloud-init: boot'ta kendini yapılandıran makine `[uygulama]`

Donmuş bir AMI her instance'da aynıdır — ama her instance biraz farklı olmalı (farklı hostname, farklı
uygulama config'i, farklı SSH anahtarı). Bu farkı **cloud-init** kapatır. Faz 5'te gördüğün gibi cloud-init,
sistemin ilk boot'unda çalışan bir systemd bileşenidir; **user-data** olarak verdiğin script'i çalıştırır:

```bash
#!/bin/bash
# EC2 user-data — ilk boot'ta çalışır (Faz 10 — bir Bash script'i!)
apt-get update && apt-get install -y nginx    # paket kur (Faz 8)
systemctl enable --now nginx                   # servisi kalıcı tanıma ekle + başlat (Faz 5)
```

Bu, iki fazın buluşmasıdır: **Faz 5** (cloud-init boot zincirinin bir parçası) + **Faz 10** (user-data bir Bash
script'idir, o yüzden idempotent ve `set -euo pipefail` ile sağlam olmalı). "Donmuş imaj + boot-time
yapılandırma" ikilisi, tekrar üretilebilir ama kişiselleştirilebilir makineler üretmenin AWS yoludur.

## 12.2.2 IAM instance role: diskte key tutmadan yetki `[uygulama]`

Faz 9'da sırları (secret) diskte tutmanın tehlikesini gördün: bir makine ele geçirilirse, diskteki AWS anahtarı
da ele geçirilir (patlama yarıçapı). AWS'in çözümü **IAM instance role**'dür: makineye kalıcı bir anahtar
gömmezsin; onun yerine instance'a bir **rol** atarsın, ve makine geçici, otomatik dönen kimlik bilgilerini
metadata servisinden alır.

Bu doğrudan Faz 9'un iki fikrini uygular: **en az yetki** (rol sadece gereken izinleri taşır — S3'e yaz, başka
hiçbir şey değil) ve **patlama yarıçapını daraltmak** (makine ele geçirilse bile diskte kalıcı bir anahtar yok;
kimlik bilgileri geçici ve dönüyor). Faz 9'un "Secrets Manager / IAM role" cloud kutusu tam olarak buydu: sır
diskte durmaz, kimlik makinenin kimliğinden gelir.

> **🤔 Düşün 12.2** — Bir EC2 instance'ının S3'e dosya yazması gerekiyor. İki yol var: (a) AWS erişim anahtarını
> user-data ile diske yazmak, (b) instance'a bir IAM role atamak. (a) Her iki yolda da makine S3'e yazabilir —
> peki güvenlik açısından fark ne, "patlama yarıçapı" (Faz 9) hangisinde daha küçük ve neden? (b) IAM role
> kullanınca kimlik bilgileri **nerede** durur, disk ele geçirilince ne kaybedilir? (c) Bu, Faz 9'un "sır diskte
> durmasın" dersine nasıl bağlanıyor?
>
> *(Cevap: fazın sonunda)*

---
---

# 12.3 EBS + fstab ve systemd Servisi

## 12.3.1 EBS: kalıcı depolama ve fstab akışı `[uygulama]`

Bir EC2 instance'ının kök diski geçicidir; instance sonlandırılınca (bazı durumlarda) veri gider. Kalıcı veri
için **EBS** (Elastic Block Store) volume'ları eklersin — bunlar Faz 6'nın "blok cihazı" kavramının AWS
karşılığıdır. Bir EBS volume ekleyince yolculuk tam Faz 6'daki gibidir:

1. EBS volume instance'a bir blok cihaz olarak görünür (`/dev/nvme1n1` — Faz 6)
2. Üstüne bir dosya sistemi oluşturursun (`mkfs.ext4` — Faz 6)
3. Bir mount noktasına bağlarsın (`mount /dev/nvme1n1 /data` — Faz 6)
4. **`/etc/fstab`**'a UUID ile kalıcı tanım eklersin (Faz 6!) — yoksa reboot'ta mount kaybolur

Faz 6'nın "var olmak ≠ erişilebilir olmak" ve "çalışan mount vs kalıcı tanım (fstab)" dersleri burada
production kritikliği kazanır: fstab'a eklemezsen instance reboot ettiğinde `/data` boş gelir, uygulama
verisini bulamaz. UUID kullanma nedeni de Faz 6'daydı: cihaz adları (`/dev/nvme1n1`) değişebilir, UUID
değişmez.

## 12.3.2 systemd servisi: uygulamanın yaşam döngüsü `[uygulama]`

Uygulamanı bir kere elle başlatmak yeterli değildir (Faz 8). Production'da uygulaman bir **systemd servisi**
olmalı — ki boot'ta otomatik başlasın, çökünce yeniden başlasın, logu journald'a düşsün. Bu, Faz 5 + Faz 8'in
buluşmasıdır:

```ini
# /etc/systemd/system/myapp.service
[Service]
ExecStart=/opt/myapp/venv/bin/python /opt/myapp/app.py
Restart=on-failure          # çökünce yeniden başlat (Faz 8)
User=myapp                  # root olarak değil! (Faz 9 — en az yetki)

[Install]
WantedBy=multi-user.target  # boot'ta başla (Faz 5 — kalıcı tanım)
```

Faz 8'in "kurulu olmak ≠ servis olarak çalışıyor olmak" ve Faz 5'in "çalışan durum vs kalıcı tanım" dersleri
production'ın kalbidir: `systemctl enable --now myapp` ile hem şimdi başlatır (çalışan durum) hem boot'a
yazarsın (kalıcı tanım). Ve `User=myapp` ile Faz 9'un "root olarak çalıştırma" dersini uygularsın.

> **🤔 Düşün 12.3** — Bir EBS volume'u `/data`'ya mount ettin ve uygulamanı çalıştırdın, her şey iyi. Ama
> `/etc/fstab`'a eklemeyi unuttun. Ertesi hafta instance reboot etti. (a) Reboot sonrası `/data` ne durumda,
> uygulaman ne görür? (b) Bu, Faz 6'nın hangi dersinin (iki fikir) production'daki tam karşılığıdır? (c)
> Uygulamanı systemd servisi olarak çalıştırıyor olman bu durumu nasıl daha kötü ya da daha görünür yapar —
> servis boot'ta başlayıp boş `/data`'yı görürse ne olur, ve bunu Faz 11 refleksinle nasıl teşhis edersin?
>
> *(Cevap: fazın sonunda)*

---
---

# 12.4 İki Savunma Katmanı ve Gözlemlenebilirlik

## 12.4.1 Security Group vs host firewall / AppArmor: iki ayrı katman `[kavram]`

Cloud'da en çok karıştırılan şeylerden biri, savunmanın **iki ayrı katmanda** olmasıdır ve ikisi de Faz 7 + Faz
9'a dayanır:

- **Security Group (SG)** — AWS seviyesinde, ağ katmanında bir güvenlik duvarı. Instance'a **hiç ulaşmadan
  önce** trafiği eler (Faz 7 — ağ erişimi). "Port 443'e 0.0.0.0/0'dan izin ver" gibi kurallar. Bu, makinenin
  **dışındaki** duvardır.
- **Host firewall (ufw/iptables) + AppArmor/SELinux** — makinenin **içinde**, işletim sistemi seviyesinde.
  Host firewall Faz 7'nin OS-seviyesi güvenlik duvarıdır; AppArmor/SELinux ise Faz 9'un MAC katmanıdır ("izinler
  doğru ama engelli").

Bu iki katman Faz 9'un **katmanlı savunma** (defense in depth) fikrinin cloud hâlidir: SG dış kapıysa,
host firewall + MAC iç kapılardır. Biri diğerinin yerine geçmez — bir saldırgan SG'yi aşsa bile (yanlış
yapılandırma), iç katmanlar hâlâ durur. Faz 9'un iki-kilit modeli (DAC + MAC) burada üç-kilit olur: SG (ağ) +
host firewall (OS ağ) + AppArmor (OS zorunlu).

## 12.4.2 CloudWatch: uzaktan gözlemlenebilirlik `[kavram]`

Faz 11'de "kanıtı makinenin dışına taşı" dedik. **CloudWatch agent / Logs** bunun AWS aracıdır: makinenin
logları (journald, /var/log, uygulama logu — Faz 5, 11) ve metrikleri (CPU, RAM, disk — Faz 4) AWS'e akar.
Böylece instance'a girmeden, hatta instance çökmüş olsa bile kanıtı okursun. Bu, Faz 4 (kaynak metrikleri) +
Faz 11 (log ve teşhis) buluşmasıdır. Ve Faz 11'in dersi burada kritiktir: CloudWatch agent'ı **AMI'ye ya da
user-data'ya baştan** koyarsın — gözlemlenebilirlik sonradan eklenen değil, baştan tasarlanan bir şeydir.

> **💡 Cloud bağlantısı — üç katman, tek refleks:** Bir instance'a "erişilemiyor" dendiğinde, artık üç katmanı da
> okuyabiliyorsun: (1) SG mü engelliyor (AWS konsolu / `aws ec2 describe-security-groups`), (2) host firewall mü
> (`ufw status` — Faz 7), (3) uygulama doğru adrese mi bind (`ss -tulpn` — Faz 7). Faz 7'nin "üç mercek"i cloud'da
> "SG + host + bind" olur. Junior mühendis sadece SG'ye bakar; usta üç katmanı da eler. İşte bu workbook'un tüm
> amacı buydu: soyutlamanın altını görmek.


> **🤔 Düşün 12.4** — Kullanıcılar web uygulamana 443'ten ulaşamıyor. Security group 0.0.0.0/0'dan 443'e izin veriyor ve yeni bir meslektaşın "SG doğru, demek ki sorun ağda değil" sonucuna varıyor. (a) Trafiği yine de hangi iki başka katman engelleyebilir? (b) Her katmanı hangi komut inceler? (c) SG neden bu ikisini göremez?
>
> *(Cevap: fazın sonunda)*
---
---

# 12.5 Container ve Lambda — Hâlâ Linux

## 12.5.1 ECS/EKS container: kernel paylaşan Linux `[kavram]`

Faz 3.6'da container'ın anatomisini görmüştün: bir container ayrı bir makine değil, **kernel'i host ile paylaşan
ama namespace (izolasyon) ve cgroup (kaynak sınırı) ile ayrılmış bir process grubudur.** AWS'in **ECS/EKS**'i
(container orkestrasyonu) bu Linux gerçeğinin üzerine kuruludur:

- **namespace** (Faz 3.6) → container'ın kendi dosya sistemi, ağı, process ağacını görmesini sağlar
- **cgroup** (Faz 3.6) → container'ın CPU/RAM sınırını uygular (Faz 4)
- Container imajı → aslında Faz 8'in "katmanlı dosya sistemi" fikridir (bir AMI'nin küçük, taşınabilir hâli)

Yani bir container "hafif bir VM" değildir; kernel paylaşan izole bir Linux process'idir. Bir container "OOM
killed" olduğunda (Faz 4), bir container'ın izni yanlış olduğunda (Faz 2), hepsi hâlâ Linux gerçekleridir — sadece
namespace/cgroup ile paketlenmiştir.

## 12.5.2 Lambda: görmediğin ama yine Linux olan runtime `[kavram]`

**Lambda** (sunucusuz/serverless) sana hiç makine göstermez — sadece bir fonksiyon yazarsın, AWS çalıştırır. Ama
altında yine bir Linux vardır: fonksiyonun bir Linux process'i olarak, bir mikro-VM içinde (Firecracker), bir
dosya sistemine (`/tmp`), bir bellek sınırına (cgroup — Faz 4), bir çalışma zamanına (Python/Node runtime)
sahip olarak koşar. "Sunucusuz" demek "Linux yok" demek değil — "Linux'u **sen yönetmiyorsun**" demektir. Soyutlama
en yükseğe çıktığında bile, altında bu workbook'un tüm temelleri (process, bellek, dosya sistemi, izin) hâlâ
oradadır.

> **🤔 Düşün 12.5** — Bir arkadaşın "Lambda kullanıyorum, artık Linux öğrenmeme gerek yok" diyor. (a) Lambda'nın
> altında hangi Linux gerçekleri hâlâ çalışıyor (en az üç tane say)? (b) Lambda fonksiyonun "out of memory"
> hatası verirse ya da `/tmp` dolu derse, bu workbook'un hangi fazlarının bilgisi işine yarar? (c) "Soyutlama
> yükseldikçe Linux bilgisi gereksizleşir" iddiasına, bu fazın merkez fikriyle ("AWS soyutlamaları Linux'un
> üzerine kuruludur") nasıl cevap verirsin?
>
> *(Cevap: fazın sonunda)*

---
---

# 12.6 Bu Köprü Bozulunca — Yolculuğun Hangi Halkası Koparsa

Bu fazın "bozulması", boot'tan production'a yolculuğun bir halkasının kopmasıdır. Her belirti bir Linux temeline
(dolayısıyla bir faza) işaret eder:

| Belirti (AWS'te) | Kopan halka | Altındaki Linux gerçeği | Faz |
|---|---|---|---|
| Instance "healthy" ama uygulama cevap vermiyor | systemd servisi failed | Kurulu ≠ çalışıyor; `systemctl status` | 8, 11 |
| user-data çalışmadı, makine yarım yapılandırıldı | cloud-init script'i hata verdi | Sessiz başarısızlık; `set -euo pipefail`, cloud-init logu | 10 |
| Reboot sonrası `/data` boş | fstab'a eklenmedi | Çalışan mount ≠ kalıcı tanım | 6 |
| Uygulama S3'e yazamıyor ama SG açık | IAM role eksik/yanlış | Yetki ağ değil kimlik katmanında | 9 |
| Dışarıdan erişilemiyor ama SG 0.0.0.0/0 | Uygulama 127.0.0.1'e bind | Dinleme ≠ doğru adrese bind (üç mercek) | 7 |
| Instance çöktü, hiç log yok | CloudWatch agent baştan konmamış | Kanıtı önceden dışarı taşı | 11 |
| Container sürekli restart / OOM killed | cgroup bellek sınırı aşıldı | Container = kaynak sınırlı Linux process | 3.6, 4 |
| "İzinler doğru ama uygulama engelli" | AppArmor/SELinux (MAC) | DAC doğru, MAC engelliyor | 9 |

> **Bu tablodan çıkan ders — ve tüm workbook'un özeti:** Bu fazın tek büyük fikri her satırın altında yatıyor:
> **her AWS belirtisinin altında bir Linux gerçeği vardır.** "Instance healthy ama uygulama yok" bir AWS
> gizemi değil, bir systemd servisinin failed olmasıdır (Faz 8). "Erişilemiyor ama SG açık" bir AWS bug'ı değil,
> uygulamanın 127.0.0.1'e bind olmasıdır (Faz 7). Bir bulut mühendisini junior'dan ayıran tam olarak budur:
> AWS'in soyutlamasında kaybolmak yerine, altındaki Linux katmanına inip kanıtı bulmak (Faz 11 refleksi). Bu
> workbook'un ilk fazından beri kurduğumuz her taş, bu son fazda tek bir yetenekte birleşiyor: **soyutlamanın
> altını görmek.**

---
---

# Faz 12 — Düşün sorularının cevapları

## Cevap 12.1 — Pişirilmiş imaj: yeni makine dondurulmuş ve hazır başlar, yamalar tarif üzerinden geçer

(a) Y takımı: instance'ları donmuş durumdan saniyeler içinde kalkar; X ise her makinede kurulumu beklemek zorundadır ve her seferinde farklı bir sonuç riski (drift, insan hatası) vardır. (b) X makineleri tek tek SSH ile yamar (mutable — filo birbirinden ayrışır). Y tarifi günceller, yeni bir AMI pişirir ve eski instance'ları değiştirir (immutable — her makine aynıdır). (c) Faz 8/10'un "mutable yamalama vs immutable yeniden inşa"sı: bir kez doğru kur, sonra çoğalt.
**İlgili bölüm:** 12.1.1 · **Devamı:** 12.2.1 (cloud-init user-data)

## Cevap 12.2 — IAM role vs diskteki anahtar; patlama yarıçapı; sır diskte durmasın

(a) Her iki yolda da makine S3'e yazabilir, ama güvenlik farkı büyüktür: diske yazılan AWS anahtarı **kalıcı** ve
**statik**tir — makine ele geçirilirse (Faz 9), saldırgan bu anahtarı okur ve onunla dilediği yerden, dilediği
kadar S3'e erişir; anahtar sızarsa elle iptal edip döndürmen gerekir. IAM role'de ise "patlama yarıçapı" çok daha
küçüktür çünkü kimlik bilgileri **geçici** (kısa ömürlü) ve **otomatik dönen**dir. (b) IAM role kullanınca kimlik
bilgileri diskte **hiç durmaz** — makine onları anlık olarak instance metadata servisinden alır ve bellekte tutar;
disk ele geçirilse bile kalıcı bir anahtar bulunmaz, sadece dakikalar içinde geçersizleşecek geçici bir token
olabilir. (c) Bu, Faz 9'un "sır diskte durmasın" dersinin AWS'teki doğrudan uygulamasıdır: sır (uzun ömürlü
anahtar) hiç diske yazılmaz, kimlik makinenin **kendi kimliğinden** (IAM role) türetilir — böylece en az yetki +
küçük patlama yarıçapı birlikte sağlanır.
**İlgili bölüm:** 12.2.2 · **Bağlanır:** Faz 9 (sırlar, patlama yarıçapı).

## Cevap 12.3 — fstab'a eklenmeyen mount; çalışan vs kalıcı; systemd ile daha görünür

(a) Reboot sonrası `/data` **boştur** — çünkü mount sadece "çalışan durum"du, kalıcı tanıma (fstab) yazılmadı;
EBS volume hâlâ orada ama `/data`'ya bağlı değil. Uygulaman `/data`'ya baktığında ya boş bir dizin ya da (mount
noktası altındaki eski kök disk içeriğini) görür — verisini bulamaz. (b) Bu, Faz 6'nın **iki dersinin** tam
production karşılığıdır: "var olmak ≠ erişilebilir olmak" (volume var ama erişilebilir değil) ve "çalışan mount ≠
kalıcı tanım (fstab)" (mount komutu geçiciydi, fstab kalıcı olurdu). (c) systemd servisi bunu hem daha kötü hem
daha görünür yapar: servis boot'ta otomatik başlar (Faz 5 — kalıcı tanım) ve boş `/data`'yı görünce ya çöker
(veri yok) ya da boş dizine yazmaya başlar (daha kötü — veriyi yanlış yere yazar). Ama görünürlük de artar: Faz
11 refleksiyle `journalctl -u myapp` uygulamanın "dosya bulunamadı" hatasını gösterir, `df -h`/`mount` ise
`/data`'nın mount edilmediğini kanıtlar — kanıt zinciri seni fstab eksikliğine götürür.
**İlgili bölüm:** 12.3.1-12.3.2 · **Bağlanır:** Faz 6, Faz 5, Faz 11.

## Cevap 12.4 — SG + host firewall + bind adresi: üç katmanı da ele

(a) Host firewall (makinenin içindeki ufw/iptables) ve uygulamanın bind adresi (yalnızca `127.0.0.1`'i dinleyen servis, SG ne derse desin dışarıdan erişilemez); AppArmor/SELinux ise sürecin neye erişebileceğini sınırlayan ek bir iç kilittir. (b) SG: `aws ec2 describe-security-groups` (ya da konsol); host firewall: `sudo ufw status`; bind: `ss -tulpn`. (c) SG trafiği instance'a **ulaşmadan önce** süzer; OS'in içinde olanlar — host firewall kuralları ve sürecin hangi adresi dinlediği — onun görüş alanı dışındadır. Junior yalnızca SG'ye bakar; usta üç katmanı da eler.
**İlgili bölüm:** 12.4.1 · **Devamı:** 11.4.2 ("Neden erişilemiyor?")

## Cevap 12.5 — Lambda'nın altındaki Linux; OOM/tmp; soyutlama yükseldikçe

(a) Lambda'nın altında hâlâ çalışan Linux gerçekleri (en az üç): fonksiyon bir **Linux process'i** olarak koşar
(Faz 3); bir **bellek sınırı** vardır ve cgroup ile uygulanır (Faz 4 + 3.6); bir **dosya sistemi** vardır
(`/tmp`, kök dosya sistemi — Faz 6); bir çalışma zamanı (runtime) ve onun süreç modeli (Faz 3), izin yapısı (Faz
2) vardır. (b) "Out of memory" hatası → Faz 4 (bellek, OOM killer) ve Faz 3.6 (cgroup sınırı) bilgisi; `/tmp`
dolu → Faz 6 (dosya sistemi, disk doluluğu) bilgisi işine yarar — çünkü Lambda'nın bu limitleri Linux
mekanizmalarının ta kendisidir. (c) "Soyutlama yükseldikçe Linux gereksizleşir" iddiasına cevap: soyutlama
Linux'u **gizler ama yok etmez.** Lambda'da Linux'u sen yönetmezsin, ama bir sorun (OOM, timeout, /tmp dolu, izin)
çıktığında onu ancak altındaki Linux gerçeğini bilerek teşhis edebilirsin. Bu fazın merkez fikri tam da budur:
AWS soyutlamaları Linux'un üzerine kuruludur — soyutlama ne kadar yükselirse yükselsin, bir şey bozulduğunda
inebileceğin kat hâlâ Linux'tur. Bu yüzden Linux bilgisi soyutlamayla **gereksizleşmez, tam tersine ayırt edici
hâle gelir.**
**İlgili bölüm:** 12.5.1-12.5.2 · **Bağlanır:** Faz 3.6, Faz 4, tüm workbook.

---
---

# Faz 12 — Sık sorulan sorular

**S1 — AMI tam olarak nedir?** Donmuş bir Linux kök dosya sistemi: kurulu paketler (Faz 8), dizin ağacı (Faz 1,
6), kullanıcı/izin yapısı (Faz 2), boot yapılandırması (Faz 5). Instance başlatınca EBS'e kopyalanıp boot eder
(12.1).

**S2 — user-data ile cloud-init aynı şey mi?** cloud-init boot'ta çalışan systemd bileşenidir (Faz 5); user-data
ona verdiğin script'tir (Faz 10 — bir Bash script'i). cloud-init user-data'yı ilk boot'ta çalıştırır (12.2.1).

**S3 — Neden diske AWS anahtarı yazmak yerine IAM role?** Diskteki anahtar kalıcı ve statiktir; makine ele
geçirilirse sızar (büyük patlama yarıçapı). IAM role geçici, dönen kimlik verir; disk ele geçirilse bile kalıcı
sır yok (Faz 9) (12.2.2).

**S4 — EBS mount edince fstab'a neden eklemeliyim?** `mount` komutu sadece "çalışan durum"dur; reboot'ta kaybolur.
fstab kalıcı tanımdır — yoksa reboot sonrası `/data` boş gelir (Faz 6). UUID kullan, cihaz adı değişebilir
(12.3.1).

**S5 — Security Group ile host firewall arasındaki fark?** SG makinenin **dışında**, AWS ağ katmanında (Faz 7);
host firewall + AppArmor makinenin **içinde**, OS katmanında (Faz 7 + 9). İkisi katmanlı savunmanın iki ayrı
kapısıdır, biri diğerinin yerine geçmez (12.4.1).

**S6 — Container gerçek bir makine mi?** Hayır — kernel'i host ile paylaşan, namespace (izolasyon) ve cgroup
(kaynak sınırı) ile ayrılmış bir Linux process grubudur (Faz 3.6). "Hafif VM" değil, izole process (12.5.1).

**S7 — Lambda kullanınca Linux öğrenmeye gerek var mı?** Evet. Lambda Linux'u gizler ama altında hâlâ Linux
process'i, bellek sınırı (cgroup), dosya sistemi (/tmp) vardır. Bir sorun çıkınca onu ancak Linux temelleriyle
teşhis edersin (12.5.2).

---
---

# Faz 12 — Kendini sına

Bu son sınav bir sentez sınavıdır: soruların çoğu "hangi AWS kavramı hangi Linux temeline dayanır" bağlantısını
test eder. Cevaplarını kâğıda yaz. Hedef: 18 üzerinden 14+.

## Bölüm A — Kavram ve karşılık

1. Bir AMI'nin altında hangi dört Linux parçası vardır (fazlarıyla)?
2. cloud-init ile user-data'nın ilişkisi ve hangi iki faza dayandığı?
3. IAM instance role neden diske anahtar yazmaktan güvenlidir (Faz 9 iki fikir)?
4. EBS mount yolculuğunun dört adımı ve fstab'ın rolü (Faz 6)?
5. systemd servis biriminde `Restart=on-failure`, `User=myapp`, `WantedBy=multi-user.target` her biri hangi faza
   bağlanır?
6. Security Group ile host firewall/AppArmor arasındaki katman farkı?
7. Bir container'ı "hafif VM"den ayıran iki Linux mekanizması nedir (Faz 3.6)?
8. Lambda'nın altında hâlâ çalışan üç Linux gerçeği?

## Bölüm B — Uygulama ve teşhis

9. Instance "healthy" ama uygulama cevap vermiyor. Hangi halka koptu, hangi komutla doğrularsın?
10. user-data çalışmadı, makine yarım yapılandırıldı. Muhtemel neden ve hangi Faz 10 alışkanlığı önlerdi?
11. Reboot sonrası `/data` boş. Kopan halka ne, çözüm ne, hangi Faz 6 dersi?
12. Uygulama S3'e yazamıyor ama SG açık. Sorun hangi katmanda, çözüm ne?
13. Dışarıdan erişilemiyor ama SG 0.0.0.0/0. Hangi Linux gerçeği, hangi komutla kanıtlarsın?
14. Instance çöktü ve hiç log yok. Ne yapmalıydın, bu Faz 11'in hangi dersi?

## Bölüm C — Muhakeme ve sentez

15. "AWS soyutlamaları Linux'un üzerine kuruludur" fikrini üç örnekle (AMI, IAM role, container) açıkla.
16. Bir bulut mühendisini junior'dan ayıran "soyutlamanın altını görmek" ne demek? "Erişilemiyor ama SG açık"
    örneğiyle anlat.
17. Boş bir AMI'den production'a yolculuğun yedi adımını sırayla say ve her birini bir faza bağla.
18. Bu workbook'un ilk fazından (zihinsel model) bu son faza (cloud köprüsü) tek bir cümlelik ana fikir yaz:
    Linux öğrenmek cloud için neden temeldir?

---

## Cevap anahtarı

1. Kök dosya sistemi (Faz 6), kurulu paketler (Faz 8), kullanıcı/izin yapısı (Faz 2), boot yapılandırması (Faz
   5) (12.1.1). — 2. cloud-init boot'taki systemd bileşeni (Faz 5), user-data ona verilen Bash script'i (Faz 10);
   cloud-init user-data'yı ilk boot'ta çalıştırır (12.2.1). — 3. En az yetki (rol sadece gereken izni taşır) +
   küçük patlama yarıçapı (diskte kalıcı anahtar yok, kimlik geçici/dönen) (12.2.2). — 4. Blok cihaz görünür →
   mkfs → mount → fstab'a UUID ile kalıcı tanım; fstab olmadan reboot'ta mount kaybolur (12.3.1). —
   5. `Restart=on-failure` → Faz 8 (çökünce yeniden başla), `User=myapp` → Faz 9 (root değil, en az yetki),
   `WantedBy=multi-user.target` → Faz 5 (boot'ta başla, kalıcı tanım) (12.3.2). — 6. SG makinenin dışında AWS ağ
   katmanında (Faz 7), host firewall/AppArmor makinenin içinde OS katmanında (Faz 7+9); katmanlı savunmanın iki
   kapısı (12.4.1). — 7. namespace (izolasyon) + cgroup (kaynak sınırı); container kernel paylaşan izole process
   (12.5.1). — 8. Linux process (Faz 3), bellek sınırı/cgroup (Faz 4+3.6), dosya sistemi /tmp (Faz 6) — ve
   runtime/izin (Faz 2,3) (12.5.2).

9. systemd servisi failed; `systemctl status myapp` + `journalctl -u myapp` (status check ≠ uygulama) (12.6, Faz
   8+11). — 10. cloud-init script'i sessizce hata verip devam etti; `set -euo pipefail` (Faz 10) önlerdi;
   cloud-init logunu (`/var/log/cloud-init-output.log`) oku (12.6). — 11. Mount fstab'a eklenmedi; `/etc/fstab`'a
   UUID ile ekle; "çalışan mount ≠ kalıcı tanım" (Faz 6) (12.3.1, Cevap 12.3). — 12. Yetki ağ değil **kimlik**
   katmanında; IAM role eksik/yanlış — doğru izinli bir role ata (Faz 9) (12.6). — 13. Uygulama 127.0.0.1'e bind
   olmuş; `ss -tulpn | grep <port>` ile bind adresini gör (dinleme ≠ doğru adres, Faz 7 üç mercek) (12.6). —
   14. CloudWatch agent'ı AMI/user-data'ya baştan koymalıydın; "kanıtı önceden dışarı taşı" (Faz 11) (12.4.2). —
   15. AMI = donmuş Linux dosya sistemi; IAM role = Linux'un sır/kimlik yönetiminin AWS soyutlaması; container =
   namespace/cgroup ile paketlenmiş Linux process — hepsinin altında Linux mekanizması var (12, tüm faz). —
   16. Soyutlamanın altını görmek: AWS belirtisini bir Linux gerçeğine indirgemek. "Erişilemiyor ama SG açık" →
   junior sadece SG'ye bakar; usta uygulamanın bind adresini (127.0.0.1) `ss -tulpn` ile kontrol eder (Faz 7)
   (12.4.2, 12.6). — 17. cloud-init/user-data (Faz 5+10) → kullanıcı/SSH erişimi (Faz 7) → EBS mount + fstab (Faz
   6) → paket kurulumu (Faz 8) → systemd servis (Faz 5+8) → log/monitoring/CloudWatch (Faz 4+11) → hardening: SG
   + host firewall + IAM role + MAC (Faz 7+9) (12, "Final çıktısı"). — 18. Örnek: "Cloud, Linux'un üzerine
   kurulu soyutlamalardan ibarettir; bu yüzden bir bulut mühendisi soyutlama bozulduğunda inebileceği tek sağlam
   kata — Linux temellerine — hâkim olmak zorundadır" (tüm workbook).

## Puanlama

| Doğru | Anlamı |
|---|---|
| 16-18 | Tebrikler — tüm workbook'u sentezledin. Her AWS kavramının altındaki Linux'u görebiliyorsun. Cloud yolculuğuna hazırsın. |
| 13-15 | Çok iyi. Kaçırdığın bağlantıların fazlarına dön (özellikle 12.3 ve 12.6 tablosu). |
| 9-12 | Temeller var ama köprüler zayıf. Faz haritasını ve 12.6 tablosunu tekrar çalış — her satırı bir faza bağla. |
| 0-8 | Bu bir sentez fazı; önce ilgili temel fazları (6, 7, 8, 9, 11) tazele, sonra buraya dön. |

Kaçırdığın soru → dönmen gereken bölüm:

| Soru | Bölüm |
|---|---|
| 1, 15 | 12.1 AMI |
| 2, 10 | 12.2.1 cloud-init |
| 3, 12 | 12.2.2 IAM role |
| 4, 11 | 12.3.1 EBS + fstab |
| 5, 9 | 12.3.2 systemd servisi |
| 6, 13 | 12.4.1 İki savunma katmanı |
| 14 | 12.4.2 CloudWatch |
| 7, 8 | 12.5 Container ve Lambda |
| 16, 17, 18 | 12.6 + Final çıktısı |

---
---

# Faz 12 — Kapanış: Workbook'un Sonu ve Cloud Yolculuğunun Başı

## Bu fazdan — ve tüm workbook'tan — ne taşıyorsun

Faz 12, Faz 0'dan beri kurduğun her taşı tek bir kemerde birleştirdi: boş bir Ubuntu AMI'nin boot'tan
production'a yolculuğu. Artık her AWS kavramının altındaki Linux gerçeğini görebiliyorsun — AMI donmuş bir Linux,
cloud-init bir boot script'i, IAM role Faz 9'un sır dersi, EBS+fstab Faz 6'nın depolama akışı, systemd servis Faz
5+8'in yaşam döngüsü, container Faz 3.6'nın namespace/cgroup'u, Lambda görmediğin bir Linux. Ve en önemlisi:
bir şey bozulduğunda (instance healthy ama uygulama yok, erişilemiyor ama SG açık, reboot sonrası /data boş)
panik yapmadan, o belirtinin altındaki Linux katmanına inip kanıtı buluyorsun (Faz 11 refleksi).

Bu workbook'un ilk fazından son fazına kadar tek bir fikir taşındı: **Linux öğrenmek, cloud için bir ön koşul
değil, cloud'un ta kendisidir.** AWS, GCP, Azure — hepsi Linux'un üzerine kurulu soyutlamalardır. Soyutlama ne
kadar yükselirse yükselsin, bir şey bozulduğunda inebileceğin son sağlam kat hep Linux'tur. Bu yüzden bir bulut
mühendisini junior'dan ayıran şey en yeni AWS servisini bilmek değil, **her soyutlamanın altındaki Linux
gerçeğini okuyabilmektir.** Bu yeteneği artık kazandın.

## Bundan sonra nereye

Bu workbook'un sonu, gerçek cloud yolculuğunun başıdır. İki köprü seni ileri taşır:

> **🤔 Son çıktı — kendine sor:** Bu workbook boyunca öğrendiğin "soyutlamanın altını görmek" refleksini, hiç
> görmediğin bir AWS servisine (örneğin RDS, ECS, EKS) nasıl uygularsın? Yeni bir servis gördüğünde ilk soruların
> ne olmalı: "bunun altında hangi Linux mekanizması var, bir şey bozulunca hangi katmana inerim?" Bu tek soru,
> bu workbook'un sana kazandırdığı en kalıcı alışkanlıktır.
>
> **🧪 Bitirme Lab'ı (kendi AWS test hesabında) — tüm workbook'un pratiği:** Boş bir Ubuntu t2.micro instance
> başlat ve bu fazın yolculuğunu **baştan sona elle** yap: (1) user-data ile boot'ta nginx kur (Faz 5+10). (2)
> SSH ile gir, `ubuntu` kullanıcısını ve izinleri incele (Faz 2). (3) Bir EBS volume ekle, formatla, mount et,
> **fstab'a UUID ile ekle** (Faz 6). (4) Basit bir uygulamayı systemd servisi olarak, `User=` ile non-root yaz
> ve enable et (Faz 5+8+9). (5) Bir IAM role ata, diske hiç anahtar yazmadan S3'e eriş (Faz 9). (6) `ss -tulpn`
> ile bind adresini, `ufw` ile host firewall'u, SG'yi kontrol et — üç savunma katmanı (Faz 7+9). (7) CloudWatch
> agent kur, logları dışarı akıt (Faz 4+11). Şimdi **kasıtlı bir arıza kur** (uygulamayı 127.0.0.1'e bind et) ve
> Faz 11 refleksiyle teşhis et. Bu tek lab, bu workbook'un on üç fazını tek bir gerçek makinede canlandırır. (8)
> Bitince instance'ı **sonlandır** (terminate) — maliyet oluşmasın; bu da Faz 10'un "immutable, kullan-at
> altyapı" dersidir.

Bu workbook'u bitirdin. Artık her komutun **neden** orada olduğunu, her soyutlamanın **altında** ne olduğunu ve
bir şey bozulunca **hangi katmana** ineceğini biliyorsun. Cloud yolculuğun burada başlıyor.

---

> **Ekler:** [Ek A — Komut Sözlüğü](EK_A_Komut_Sozlugu.md) · [Ek B — Dosya ve Dizin Haritası](EK_B_Dosya_Dizin_Haritasi.md) · [Ek C — "Bozulunca" Hızlı Başvuru](EK_C_Bozulunca_Hizli_Basvuru.md)

> **Navigasyon:** [◀ Faz 11 — Gözlemlenebilirlik ve Troubleshooting](Faz_11_Gozlemlenebilirlik_ve_Troubleshooting.md) · **Faz 12** · [Ek A — Komut Sözlüğü ▶](EK_A_Komut_Sozlugu.md)
