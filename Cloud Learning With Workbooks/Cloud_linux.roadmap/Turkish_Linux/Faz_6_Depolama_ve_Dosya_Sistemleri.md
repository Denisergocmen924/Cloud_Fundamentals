# Faz 6 — Depolama ve Dosya Sistemleri: Kalıcı Katman

> **Navigasyon:** [◀ Faz 5 — Boot, Init ve systemd](Faz_5_Boot_Init_systemd.md) · **Faz 6** · [Ara Sınav 3 ▶](Ara_Sinav_3.md)

---

## Nereden geliyoruz

Faz 5 sana makinenin nasıl **var olduğunu** verdi: boot zinciri, systemd, servislerin ayağa kalkması.
O fazın sonunda bir arıza türünden söz edip geçtik: **`/etc/fstab`'daki yanlış bir satır makineyi
boot'ta emergency moduna düşürür.** 5.7 tablosundaki "instance açılmıyor" satırlarının bir kısmı aslında
bir **disk/mount** sorunuydu. Bu faz tam olarak o kutuyu açıyor: mount nedir, `fstab` neden boot'u
kilitleyebilir, ve bir diskin ham hâlden kullanılabilir bir dosya sistemine nasıl geldiği.

Faz 5'ten iki şey burada doğrudan işe yarayacak:

- **`.mount` unit'leri ve systemd'nin bağımlılık modeli** (5.3.1, 5.3.4). Bir mount aslında systemd
  için bir unit'tir; `Requires`/`After` yarışının (Cevap 5.3) disk tarafını burada göreceksin.
- **Boot zincirinin nerede kırıldığı** (5.1, 5.7). `fstab` hatası boot'u init aşamasında kilitler —
  yani OS ayağa kalkmadan takılır, tam da 5.1.1'de "OS'tan önceki halka" dediğimiz sınıra yakın bir
  yer.

Ayrıca Faz 4'ün disk I/O sezgisi (`iostat`, `%util`, `await`) burada fiziksel zemine oturur: o
metrikleri üreten şey, bu fazda tanıyacağın blok cihaz ve dosya sistemidir.

## Bu fazın sorusu

Faz 5 "makine nasıl çalışır hâle geliyor" dedi. Bu faz veriye bakıyor:

> *"Veri diskte nasıl duruyor, nasıl erişilebilir hâle geliyor — ve bu katman nerede bozulunca makine
> açılmaz veya 'disk dolu' der?"*

Cloud'da en sık yapılan operasyonlardan biri bir instance'a yeni disk (EBS volume) eklemektir: ekle →
gör → biçimle → mount et → kalıcı yap. Bu beş adımlık akışın her adımının bir tuzağı var, ve en
tehlikelisi sonuncusu: `fstab`'a yanlış yazarsan makine bir daha **boot etmez**. Bu yüzden bu faz hem
"nasıl yapılır"ı hem "nasıl bozulur"u birlikte öğretir.

Bu fazın sonunda yeni bir EBS volume'ünü **boot'u kilitlemeyecek** şekilde (UUID ile, cihaz adıyla
değil — neden olduğunu bilerek) kalıcı mount etme adımlarını sırayla söyleyebileceksin. Ayrıca "disk
dolu değil ama 'no space left' diyor" gibi ilk bakışta imkânsız görünen bir arızayı (inode tükenmesi)
teşhis edebileceksin.

---

## Bu fazın sonunda

- Ham blok cihaz ile üstüne "biçimlenen" dosya sistemi arasındaki farkı; `/dev/sda` ve `/dev/nvme0n1`
  gibi cihaz adlarını; `lsblk` ile cihaz ağacını okumayı açıklayabileceksin
- Partition, `mkfs` (biçimleme), `mount`/`umount` ve mount point kavramlarını; bir diskin ham hâlden
  kullanılabilir hâle gelme akışını anlatabileceksin
- `/etc/fstab`'ın ne olduğunu, boot'ta nasıl okunduğunu ve **yanlış bir satırın makineyi neden kurtarma
  moduna düşürdüğünü** açıklayabileceksin
- ext4 ile xfs'i (cloud'da yaygın iki dosya sistemi) ana hatlarıyla ayırt edebilecek; journaling'in
  ne işe yaradığını (yarım kalan yazımdan kurtulma) bileceksin
- inode'un ne olduğunu; diskte yer varken **inode**'un nasıl tükenebileceğini; `df -h` ile `df -i`
  farkını açıklayabileceksin
- LVM'in ne için var olduğunu (disk büyütmeyi esnetmek) kavram düzeyinde bileceksin
- **Cloud:** bir EBS volume'ünü attach → `lsblk` → `mkfs` → `mount` → `fstab` (UUID ile) akışıyla kalıcı
  mount edebilecek; kök volume'ü `growpart`+`resize2fs` ile büyütebilecek; instance store'un neden
  ephemeral (durunca kaybolan) olduğunu anlatabileceksin

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden burada |
|---|---|---|---|
| 6.1 | Blok cihaz vs dosya sistemi | `[mekanizma]` | Diskin iki katmanı: ham cihaz ve üstündeki format |
| 6.2 | Bölme, biçimleme, mount ve fstab | `[mekanizma]` | **Fazın kalbi** — ve `fstab`'ın boot'u kilitleme tuzağı |
| 6.3 | Dosya sistemleri: ext4, xfs, journaling | `[kavram]` | Cloud'da hangi format, neden |
| 6.4 | Inode — gizli tuzak | `[mekanizma]` | "Yer var ama disk dolu" arızası |
| 6.5 | LVM ve genişletme | `[kavram]` | Disk büyütmenin esnek yolu |
| 6.6 | Cloud depolama akışı | `[uygulama]` | **Fazın çıktısı** — EBS'yi kalıcı ve güvenli mount et |
| 6.7 | Bu faz bozulunca | — | Depolama arızalarının imzaları |

> **Bu fazda nasıl çalışmalı:** Bu faz Faz 5'ten bir adım daha "tehlikeli"dir çünkü kalıcı veri ve boot
> ile oynar. Gözlem komutları 🟢 (`lsblk`, `df -h`, `df -i`, `blkid`, `cat /etc/fstab` — sadece okur).
> Ama `mkfs` (biçimleme) bir diski **geri dönüşsüz siler** 🔴 — asla yanlış cihaza çalıştırma. `mount`
> geçicidir 🟡 (reboot'ta gider). `/etc/fstab`'ı düzenlemek 🔴'dır çünkü yanlış satır **boot'u kilitler**
> — her `fstab` düzenlemesi bir doğrulama (`mount -a` veya `findmnt --verify`) ile bitmeli. **Bu fazın
> `fstab`/`mkfs` deneylerini yalnızca atılabilir bir test instance'ında yap.** En aydınlatıcı an: boş
> bir EBS volume ekleyip `lsblk`'te belirmesini, sonra biçimleyip mount ettikten sonra ağaçta mount
> point'iyle görünmesini izlemek.

---
---

# 6.1 Blok Cihaz vs Dosya Sistemi

## 6.1.1 İki ayrı katman: ham cihaz ve üstündeki format `[mekanizma]`

Faz 0'da "her şey dosyadır" dedik; diskler de bu kuralın içindedir ama özel bir alt tür olarak: **blok
cihaz**. Şimdi kurmamız gereken model, çoğu insanın hiç ayırmadığı iki katmanı birbirinden ayırmaktır:

1. **Blok cihaz (ham disk).** Diskin kendisi, çekirdeğe göre sadece **numaralandırılmış blokların** (sabit
   boyutlu, örneğin 512 bayt veya 4 KB'lık parçaların) uzun bir dizisidir. Üstünde henüz hiçbir yapı,
   dosya, dizin yoktur — sadece "blok 0, blok 1, blok 2..." diye giden ham depolama. Linux bunu
   `/dev/` altında bir dosya olarak temsil eder: `/dev/sda` (klasik SATA/SCSI diski), `/dev/nvme0n1`
   (NVMe SSD — cloud'da yaygın), `/dev/xvda` (Xen sanal diski).
2. **Dosya sistemi (format).** Bu ham blokların üstüne "biçimlenen" (yazılan) bir organizasyon
   şemasıdır: dosyalar, dizinler, izinler (Faz 2!), zaman damgaları, ve hangi verinin hangi blokta
   olduğunu tutan defterler. `mkfs` komutu tam olarak bu şemayı ham cihazın üstüne kurar. Dosya sistemi
   olmadan bir blok cihaza dosya yazamazsın — sadece ham bloklara erişebilirsin.

Analoji: blok cihaz **boş, çizgisiz bir defter**tir (sadece sayfalar var). Dosya sistemi, o deftere
çizilen **satırlar, içindekiler tablosu ve sayfa numaralarıdır** — hangi bilginin nerede olduğunu
bulmanı sağlayan düzen. Aynı boş defter (blok cihaz) farklı düzenlerle (ext4, xfs) biçimlenebilir; ama
her seferinde eskisinin üstüne yazılır.

> **⚠️ Yaygın yanılgı: "Disk = dosya sistemi, ikisi aynı şey."**
>
> Değil. Bir diskin (blok cihaz) **var olması**, üstünde bir dosya sistemi olduğu anlamına gelmez. Yeni
> eklenen bir EBS volume'ü `lsblk`'te görünür (blok cihaz var) ama `mkfs` yapılmadan mount edilemez
> (dosya sistemi yok). "Diski ekledim ama mount edemiyorum" arızasının çok yaygın sebebi budur: cihaz
> var, format yok. İki katmanı ayırmak bu arızayı bir bakışta çözer.

## 6.1.2 `lsblk` ile cihaz ağacını okumak `[uygulama]`

Blok cihazları ve üstlerindeki yapıyı görmenin standart aracı `lsblk`'tir (*list block devices*). Bir
ağaç çizer: diskler, altlarındaki partition'lar, ve her partition'ın nereye mount edildiği.

> **🔧 Makinende gör** 🟢 — blok cihaz ağacını oku
>
> ```
> $ lsblk
> NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
> nvme0n1     259:0    0   20G  0 disk
> ├─nvme0n1p1 259:1    0 19.9G  0 part /
> └─nvme0n1p15 259:2   0  99M  0 part /boot/efi
> nvme1n1     259:3    0   50G  0 disk
> ```
>
> `nvme0n1` = kök disk (20G); altındaki `nvme0n1p1` bir partition'dır ve `/` köküne mount edilmiş.
> `nvme1n1` = yeni eklenmiş 50G'lık bir EBS volume — dikkat: `TYPE disk` ama `MOUNTPOINTS` **boş** ve
> altında partition yok. Yani cihaz var, ama henüz ne biçimlenmiş ne mount edilmiş. 6.6'da tam olarak
> bunu kullanılabilir hâle getireceğiz.

Sütunların anlamı: `NAME` cihaz/partition adı; `SIZE` boyut; `TYPE` (disk = tüm cihaz, part =
partition, lvm = LVM birimi); `MOUNTPOINTS` nereye bağlı (boşsa mount edilmemiş). Bu tek komut, "hangi
diskler var, hangileri kullanımda, yenisi belirdi mi" sorularının hepsini cevaplar.

> **💡 Cloud bağlantısı — attach ettiğin volume neden hemen görünür:** Bir EBS volume'ünü konsoldan bir
> instance'a attach ettiğinde, çekirdek onu anında yeni bir blok cihaz (`/dev/nvme1n1`) olarak görür ve
> `lsblk`'te belirir — çünkü attach, fiziksel bir diski takmakla aynı şeydir (Nitro üstünde NVMe olarak
> sunulur). Ama görünmesi kullanılabilir olması demek değildir: sıradaki adımlar (biçimle, mount)
> senin işin. "Volume'ü attach ettim ama `df`'te yok" şaşkınlığının cevabı budur: `df` mount edilmiş
> dosya sistemlerini gösterir; `lsblk` ham cihazları. Yeni volume önce `lsblk`'te görünür, `df`'e
> ancak mount'tan sonra girer.

> **🤔 Düşün 6.1** — `lsblk` çıktısında `nvme1n1` cihazını `TYPE disk`, `SIZE 50G`, `MOUNTPOINTS` boş
> görüyorsun. Bir meslektaşın "disk bozuk, `df -h`'te hiç görünmüyor" diyor. Faz 6.1.1'deki iki-katman
> modelini ve yukarıdaki cloud kutusunu kullanarak, diskin bozuk **olmadığını** ve `df`'te görünmemesinin
> neden beklenen bir durum olduğunu açıkla. Diski `df`'e sokmak için hangi iki adım gerekir?
>
> *(Cevap: fazın sonunda)*

---
---

# 6.2 Bölme, Biçimleme, Mount ve fstab

## 6.2.1 Ham diskten kullanılabilir dosya sistemine: dört adım `[mekanizma]`

Bir blok cihazı kullanılabilir hâle getirmek bir sıra iştir. Her adım bir öncekinin üstüne bir katman
ekler:

1. **(İsteğe bağlı) Bölme (partition).** Bir diski mantıksal parçalara bölmek. Cloud'da tek amaçlı bir
   veri diskini genellikle **bölmeden**, tüm cihazı tek dosya sistemi yaparsın — ama kök disk
   genellikle bölünmüştür (`nvme0n1p1` = kök, `p15` = EFI). Bölme aracı: `parted`, `fdisk`.
2. **Biçimleme (`mkfs`).** Ham cihaza (veya partition'a) bir dosya sistemi yazmak: `mkfs.ext4
   /dev/nvme1n1`. Bu adım cihazın üstüne dosya/dizin defterini kurar — ve **var olan her şeyi siler**.
3. **Mount (`mount`).** Dosya sistemini, dizin ağacının bir noktasına (**mount point**) bağlamak:
   `mount /dev/nvme1n1 /data`. Bundan sonra `/data` altına yazdığın her şey o diske gider. Mount point,
   var olan bir boş dizindir; mount, o dizini diskin "kapısı" yapar.
4. **Kalıcı yapma (`/etc/fstab`).** `mount` komutu geçicidir — reboot'ta kaybolur. Kalıcı olması için
   mount'u `/etc/fstab`'a bir satır olarak yazarsın; böylece her boot'ta systemd onu otomatik mount
   eder (6.2.3).

> **🔧 Makinende gör** 🟡🔴 — dört adımı sırayla (test instance'ında)
>
> ```
> $ sudo mkfs.ext4 /dev/nvme1n1          # 🔴 GERİ DÖNÜŞSÜZ — cihazı biçimler
> $ sudo mkdir -p /data                  # 🟢 mount point (boş dizin)
> $ sudo mount /dev/nvme1n1 /data        # 🟡 şimdi bağla (reboot'ta gider)
> $ df -h /data
> Filesystem      Size  Used Avail Use% Mounted on
> /dev/nvme1n1     49G   24K   47G   1% /data
> ```
>
> **Geri alma:** `mkfs` geri alınamaz (diskteki veri gider) — bu yüzden **doğru cihaz adını üç kez
> kontrol et** (`lsblk` ile). `mount`'u geri almak: `sudo umount /data`. Henüz `fstab`'a bir şey
> yazmadığımız için reboot zaten mount'u kaldırır; kalıcı hâle 6.2.3'te geçeceğiz.

## 6.2.2 Mount point kavramı: dizin ağacına bağlanmak `[kavram]`

Windows'tan gelenlerin en çok takıldığı yer burasıdır: Linux'ta diskler **harf almaz** (C:, D: yok).
Bunun yerine her dosya sistemi, tek bir birleşik dizin ağacının (`/`'ten başlayan) bir **noktasına**
bağlanır. `/data` bir mount point ise, `/data` altındaki dosyalar aslında `nvme1n1` diskinde yaşar;
`/home` başka bir diske mount edilmişse oradaki dosyalar o diskte. Kullanıcı için hepsi tek, kesintisiz
bir ağaç gibi görünür — hangi dosyanın hangi fiziksel diskte olduğunu bilmesi gerekmez.

Bunun iki önemli sonucu var:

- **Mount, altındaki dizini gizler.** `/data` içinde önceden dosyalar varken oraya bir disk mount
  edersen, o eski dosyalar "kaybolmaz" ama artık **görünmez** (mount kaldırılınca geri gelirler). Bu
  yüzden mount point olarak boş bir dizin seçilir.
- **Aynı disk farklı yerlere görünebilir**, ve mount hiyerarşisi iç içe geçebilir (`/data` bir disk,
  `/data/logs` başka bir disk).

> **❓ Akla gelen soru: "`mount` ile bağladım, çalışıyor. `fstab`'a yazmama neden gerek var, fazladan
> iş değil mi?"**
>
> Çünkü `mount` komutunun etkisi **çekirdeğin o anki durumundadır** ve reboot'ta sıfırlanır — tıpkı
> Faz 5'teki `systemctl start`ın reboot'ta kaybolması gibi (5.3.3). `fstab`, mount'un "enable"ıdır:
> "her boot'ta bunu otomatik bağla" kaydı. `fstab`'a yazmazsan, bir reboot sonrası `/data` boş bir
> dizine döner ve oraya yazan uygulaman aniden kök diske (veya hataya) yazmaya başlar. "Disk mount'luydu,
> reboot sonrası gitti" arızası, `fstab`'sız mount'un ta kendisidir.

## 6.2.3 `/etc/fstab`: kalıcı mount ve boot'u kilitleme tuzağı `[mekanizma]`

`/etc/fstab` (*filesystem table*), boot'ta hangi dosya sistemlerinin nereye, nasıl mount edileceğini
tanımlayan tablodur. Her satır bir mount'u tarif eder ve altı alandan oluşur:

```
# <cihaz/UUID>                              <mount point> <tip>  <seçenekler>     <dump> <pass>
UUID=8f3b...c2                              /data         ext4   defaults          0      2
```

- **cihaz/UUID:** hangi disk. **Cihaz adı (`/dev/nvme1n1`) yerine UUID kullanılır** — nedeni birazdan.
- **mount point:** nereye bağlanacak (`/data`).
- **tip:** dosya sistemi türü (`ext4`, `xfs`).
- **seçenekler:** `defaults`, `noatime`, `nofail`... mount davranışını ayarlar.
- **dump/pass:** yedekleme ve boot'ta `fsck` (dosya sistemi kontrolü) sırası; genelde `0 2` (kök için
  `0 1`).

**Neden UUID, cihaz adı değil?** Çünkü cihaz adları (`/dev/nvme1n1`, `/dev/nvme0n1`) **sabit
değildir**: bir instance'a ikinci bir disk eklersen veya diskleri farklı sırada attach edersen,
çekirdeğin verdiği isimler yer değiştirebilir — dünkü `nvme1n1` bugün `nvme2n1` olabilir. `fstab`'da
cihaz adı yazdıysan ve isim kaydıysa, boot'ta systemd o adı bulamaz, mount başarısız olur, ve
(seçeneklere göre) makine **emergency moduna** düşer. UUID (*Universally Unique Identifier*) ise dosya
sisteminin kendisine yazılı, diske özgü ve **değişmeyen** bir kimliktir — disk hangi cihaz adını alırsa
alsın UUID aynı kalır. `blkid` ile öğrenilir.

> **🔧 Makinende gör** 🔴 — kalıcı mount'u güvenli kur
>
> ```
> $ sudo blkid /dev/nvme1n1
> /dev/nvme1n1: UUID="8f3b-...-c2" TYPE="ext4"
> # /etc/fstab'a ekle:
> UUID=8f3b-...-c2  /data  ext4  defaults,nofail  0  2
> $ sudo mount -a          # fstab'daki TÜM satırları dene — HATA VERMEDEN boot doğrulaması
> $ findmnt --verify       # fstab'ı sözdizimi/tutarlılık açısından denetle
> ```
>
> **Geri alma:** `fstab`'a satır ekledikten sonra **asla doğrulamadan reboot etme.** `sudo mount -a`
> tüm fstab satırlarını dener; hata verirse satırın yanlıştır ve **düzeltmeden reboot edersen makine
> boot etmez.** Satırı geri almak: eklediğin satırı `fstab`'dan sil ve `sudo umount /data`. İpucu:
> `nofail` seçeneği, o disk bulunamazsa boot'un durmak yerine devam etmesini sağlar — cloud'da veri
> diskleri için hayat kurtarır.

> **🤔 Düşün 6.2** — Bir mühendis yeni bir veri diskini `fstab`'a `/dev/nvme1n1  /data  ext4  defaults
> 0 2` diye ekledi, `mount -a` denemeden reboot etti. Makine artık SSH kabul etmiyor, konsol "Cannot
> open access to console, the root account is locked... Give root password for maintenance" diyor.
> Faz 6.2.3 (UUID vs cihaz adı, `nofail`) ve Faz 5.1 (boot zinciri) bilgini birleştirerek: (a) büyük
> olasılıkla ne oldu, (b) bu satırın hangi iki ayrı seçimi bu felaketi önlerdi?
>
> *(Cevap: fazın sonunda)*

---
---

# 6.3 Dosya Sistemleri: ext4, xfs ve Journaling

## 6.3.1 Cloud'da iki yaygın format: ext4 ve xfs `[kavram]`

`mkfs` yaparken bir dosya sistemi **tipi** seçersin. Linux dünyasında düzinelerce vardır ama cloud'da
pratikte iki tanesiyle karşılaşırsın:

- **ext4** — Linux'un uzun yıllardır standart, "güvenli varsayılan" dosya sistemi. Olgun, çok iyi test
  edilmiş, her yerde çalışır. Ubuntu'nun kök diski genelde ext4'tür. Küçültme (shrink) dâhil geniş araç
  desteği vardır.
- **xfs** — büyük dosyalar ve yüksek paralel I/O için tasarlanmış, yüksek performanslı bir dosya
  sistemi. Amazon Linux ve RHEL/CentOS kök diskinde varsayılandır. Büyük veri diskleri, veritabanları
  ve çok çekirdekli yazma yükleri için sık seçilir. Önemli bir kısıt: xfs **büyütülebilir ama
  küçültülemez** (shrink yok).

Pratik seçim kuralı bu faz için: bir dağıtımın varsayılanını (Ubuntu→ext4, Amazon Linux→xfs) bozmak
için özel bir sebebin yoksa onu kullan. İkisi de günlük kullanımda "dosya sistemi" gibi hisseder; fark
uç yüklerde ve yönetim araçlarında ortaya çıkar. Bu fazın hedefi ikisini derinlemesine kıyaslamak değil,
`fstab`'da `ext4` mi `xfs` mi yazacağını bilmek ve `mkfs.xfs`/`mkfs.ext4` ayrımını tanımaktır.

> **⚠️ Yaygın yanılgı: "Dosya sistemi tipi sadece bir tercih, mount ederken istediğimi seçerim."**
>
> Hayır — tip diske **`mkfs` sırasında yazılır** ve o diskin kalıcı bir özelliğidir. `mkfs.xfs` ile
> biçimlenmiş bir diski `fstab`'da `ext4` diye mount etmeye çalışırsan mount başarısız olur. `fstab`'daki
> tip alanı diskteki gerçek formatla eşleşmek zorundadır; `blkid` sana o diskin gerçek `TYPE`'ını söyler.
> Tipi değiştirmenin tek yolu yeniden biçimlemektir (ve bu her şeyi siler).

## 6.3.2 Journaling: yarım kalan yazımdan kurtulmak `[kavram]`

Hem ext4 hem xfs **journaling** (günlükleme) yapan dosya sistemleridir. Sorun şu: bir dosya yazmak
aslında birden çok ayrı disk işlemidir (veriyi yaz, defteri güncelle, boş blok listesini güncelle...).
Tam bu işlemlerin ortasında elektrik giderse veya instance çökerse, dosya sistemi **tutarsız** bir
hâlde kalır: defter bir şey der, gerçek bloklar başka. Journaling'siz eski dosya sistemlerinde bu, boot'ta
saatlerce süren tam bir `fsck` taraması ve bazen veri kaybı demekti.

Journaling bunu şöyle çözer: dosya sistemi asıl değişikliği yapmadan **önce** ne yapacağını küçük bir
"günlüğe" (journal) yazar. Yazım yarıda kesilirse, boot'ta dosya sistemi günlüğe bakar ve ya işlemi
tamamlar ya da geri alır — böylece hızlıca **tutarlı** bir duruma döner. Bu, "instance beklenmedik
şekilde reboot oldu ama dosya sistemi sağlam açıldı" deneyiminin arkasındaki mekanizmadır.

> **💡 Cloud bağlantısı — journaling neden cloud'da daha da önemli:** Cloud instance'ları senin
> kontrolün dışında durabilir/yeniden başlayabilir: spot instance geri alınır, alttaki donanım arızası
> bir "stop/start" tetikler, veya sen yanlışlıkla force-reboot yaparsın. Bu ani kesintiler journaling'in
> tam olarak koruduğu senaryodur. Bu yüzden cloud'daki neredeyse tüm modern dosya sistemleri
> journaling'lidir — ve bu yüzden bir instance beklenmedik kesintiden sonra genelde veri kaybı olmadan
> geri gelir. Yine de journaling **uygulama düzeyi** tutarlılığı garanti etmez (yarım yazılmış bir
> uygulama dosyası yine bozuk olabilir); sadece dosya sisteminin kendi yapısını korur.

> **🤔 Düşün 6.3** — Bir ekip arkadaşın yeni bir veri diskini `mkfs.xfs` ile biçimlendirdi ama `fstab`'a
> "Ubuntu kök diskimiz ext4" diyerek tür olarak `ext4` yazdı. (a) `sudo mount -a` çalıştırınca ne olur ve
> neden (6.3.1)? (b) Diskin gerçek türünü hangi komut gösterir; çözüm yeniden biçimlendirmek mi, `fstab`'ı
> düzeltmek mi? (c) Aylar sonra ekip maliyeti düşürmek için bu volume'ü küçültmek istiyor. 6.3.1 xfs
> hakkında ne diyor ve pratik alternatif nedir?
>
> *(Cevap: fazın sonunda)*

---
---

# 6.4 Inode — Gizli Tuzak

## 6.4.1 Inode nedir: dosyanın kimlik kartı `[mekanizma]`

Faz 2'de izinleri, sahipliği, zaman damgalarını öğrendin. Şimdi soru: dosya sistemi bütün bu bilgiyi
**nerede** tutuyor? Cevap: **inode**'da. Her dosya (ve her dizin) için dosya sistemi bir inode tutar —
o dosyanın **metadata kimlik kartı**:

- dosyanın boyutu, sahibi (UID), grubu (GID), izin bitleri (Faz 2!)
- zaman damgaları (oluşturma, değiştirme, erişim)
- ve en önemlisi: dosyanın **verisinin hangi disk bloklarında** olduğunun listesi

Dikkat: inode dosyanın **adını** tutmaz. Ad, dizinde (o da bir dosyadır) "ad → inode numarası"
eşlemesi olarak yaşar. Bu yüzden bir dosyanın birden çok adı olabilir (hard link): aynı inode'a işaret
eden farklı adlar. `ls -i` her dosyanın inode numarasını gösterir.

Kritik nokta şu: bir dosya sistemi biçimlenirken (`mkfs`) **sabit sayıda inode** ayrılır. Yani diskin
iki ayrı "bütçesi" vardır:

1. **Veri blokları** (asıl dosya içeriği için yer) — `df -h`'in gösterdiği.
2. **Inode'lar** (dosya başına bir kimlik kartı için yer) — `df -i`'nin gösterdiği.

Bu iki bütçe **birbirinden bağımsızdır** ve ayrı ayrı tükenebilir.

## 6.4.2 "Yer var ama disk dolu": inode tükenmesi `[mekanizma]`

Buradan çok kafa karıştıran bir arıza doğar. Diyelim ki bir dizin milyonlarca **çok küçük** dosyayla
doluyor (tipik suçlular: bir cache dizini, e-posta kuyruğu, ya da her isteği ayrı dosya olarak yazan
kötü yapılandırılmış bir uygulama). Her dosya bir inode harcar ama neredeyse hiç veri bloğu harcamaz.
Sonuç: **inode bütçesi biter ama veri bütçesi hâlâ yarı boştur.** Sistem artık yeni dosya oluşturamaz
ve "No space left on device" hatası verir — ama `df -h` diskin %50 boş olduğunu gösterir. İşte bu ilk
bakışta imkânsız görünen arıza budur.

> **🔧 Makinende gör** 🟢 — iki bütçeyi ayrı ayrı oku
>
> ```
> $ df -h /data                     # VERİ blokları bütçesi
> Filesystem      Size  Used Avail Use% Mounted on
> /dev/nvme1n1     49G   22G   25G  48% /data
> $ df -i /data                     # INODE bütçesi
> Filesystem      Inodes   IUsed  IFree IUse% Mounted on
> /dev/nvme1n1  3276800  3276800      0  100% /data
> ```
>
> `df -h` %48 doluluk der (bol yer var). Ama `df -i` inode'ların **%100** dolu olduğunu gösterir — yeni
> dosya oluşturulamaz. Teşhis buradadır: "No space" hatası + `df -h` boş görünüyorsa, **her zaman
> `df -i`'ye bak.** Çözüm: gereksiz küçük dosyaları silmek (inode'ları serbest bırakır); kalıcı çözüm
> genelde uygulamanın neden milyonlarca dosya ürettiğini düzeltmektir.

> **❓ Akla gelen soru: "O zaman `mkfs` yaparken çok fazla inode ayırsam bu sorun hiç olmaz mı?"**
>
> Ayarlanabilir (`mkfs.ext4 -N <sayı>` veya `-i <bayt/inode>`), ama bedava değil: her inode disk
> alanı kaplar, yani inode sayısını çok artırmak veri için kullanılabilir alanı azaltır. Varsayılanlar
> (genelde her ~16 KB veri için bir inode) tipik iş yükleri için dengelidir. "Milyonlarca küçük dosya"
> senaryosu istisnadır — ve orada bile gerçek çözüm genelde tasarımı düzeltmektir (dosyaları
> birleştirmek, bir veritabanı veya nesne deposu kullanmak), inode sayısını şişirmek değil. xfs bu
> konuda daha esnektir çünkü inode'ları dinamik ayırır, ama o da sonsuz değildir.

> **🤔 Düşün 6.4** — Bir uygulama sunucusu "cannot write file: No space left on device" hatasıyla
> istekleri reddediyor. Panikleyen ekip diski büyütmek (Faz 6.5/6.6) için hazırlanıyor. `df -h /var`
> çıktısı `Use% 61%` diyor. Faz 6.4.2'yi kullanarak: (a) diski büyütmenin bu sorunu büyük olasılıkla
> **çözmeyeceğini** neden söyleyebilirsin, (b) teşhisi kesinleştirmek için hangi tek komutu
> çalıştırırsın, (c) çıktı beklediğin gibi çıkarsa gerçek kök sebep sınıfı nedir?
>
> *(Cevap: fazın sonunda)*

---
---

# 6.5 LVM ve Genişletme

## 6.5.1 LVM neden var: diski dosya sisteminden ayırmak `[kavram]`

Şimdiye kadarki modelde bir dosya sistemi doğrudan bir blok cihazın (veya partition'ın) üstünde yaşadı.
Bunun bir sıkıntısı var: dosya sistemi o tek cihazın boyutuna hapsolur. Diski büyütmek istersen ya
volume'ü büyütüp partition'ı uzatman gerekir (yapılabilir ama esnek değil) ya da ikinci bir diski
tamamen ayrı bir mount point'e bağlaman gerekir — ikisini tek bir `/data` gibi göstermek mümkün olmaz.

**LVM (Logical Volume Manager)**, blok cihaz ile dosya sistemi arasına bir esneklik katmanı koyar. Üç
kavramla çalışır:

1. **PV (Physical Volume):** LVM'e verdiğin ham blok cihazlar (`/dev/nvme1n1`, `/dev/nvme2n1`...).
2. **VG (Volume Group):** bir veya daha çok PV'nin birleştiği tek bir "havuz". Havuzun toplam boyutu
   PV'lerin toplamıdır.
3. **LV (Logical Volume):** havuzdan kestiğin, üstüne dosya sistemi kurup mount ettiğin sanal disk.

Kilit fayda: bir LV'yi, havuzda yer olduğu sürece (veya havuza yeni bir disk ekleyerek) **online
büyütebilirsin** — mount'u kaldırmadan, çoğu zaman kesintisiz. `/data` doluyorsa, yeni bir EBS volume'ü
attach edip PV yaparsın, VG'ye eklersin, LV'yi büyütürsün, dosya sistemini genişletirsin — hepsi
çalışırken. Doğrudan-cihaz modelinde bu kadar esnek değildi.

> **⚠️ Yaygın yanılgı: "LVM disk büyütür / performans katar."**
>
> LVM sihirli bir şekilde yer eklemez ve genelde performans için değildir. Sadece var olan blok
> cihazları esnek şekilde **birleştirip bölmene** izin veren bir katmandır. Havuzda (VG'de) boş yer
> yoksa, LV'yi büyütmek için önce havuza gerçek bir disk (PV) eklemen gerekir — LVM o diski sağlamaz,
> sen (veya cloud'da EBS) sağlarsın. Faydası kapasiteyi **artırmak** değil, kapasiteyi **yönetmeyi
> esnetmektir.**

Bu faz için LVM'i kavram düzeyinde bilmen yeter: "diski dosya sisteminden ayıran, online büyütmeyi
kolaylaştıran bir katman." Cloud'da birçok dağıtım kök diski LVM üstüne kurar (özellikle RHEL ailesi),
bu yüzden `lsblk` çıktısında `lvm` tipinde satırlar görürsen şaşırma.

> **🤔 Düşün 6.5** — `/data` dosya sistemi, tek bir 100 GB diskten (PV) oluşan bir VG içindeki LV üzerinde
> duruyor ve %95 dolu. Bir arkadaşın diyor ki: "LVM büyütmeyi kolaylaştırır — LV'yi büyüt yeter." (a) Bu
> şu anda neden başarısız olabilir (6.5.1)? (b) Bulutta önce ne yapmalısın ve LVM kavramları hangi sırayla
> gelir? (c) LV büyüdükten sonra `df -h /data` hâlâ eski boyutu gösteriyor. Hangi katman geride kaldı ve
> bu, 6.6.2'nin hangi kuralını tekrar ediyor?
>
> *(Cevap: fazın sonunda)*

---
---

# 6.6 Cloud Depolama Akışı — Fazın Çıktısı

## 6.6.1 EBS'yi kalıcı ve güvenli mount etmek: uçtan uca `[uygulama]`

Şimdi bu fazın tüm parçalarını, cloud'da en sık yapılan depolama operasyonunda birleştiriyoruz: bir
instance'a yeni bir kalıcı disk (EBS volume) ekleyip, boot'u kilitlemeden kalıcı mount etmek. Sıra
tam olarak 6.2.1'deki dört adımın cloud versiyonudur:

> **🔧 Makinende gör** 🟢🔴 — EBS attach → kalıcı mount (test instance'ında)
>
> ```
> # 0) Konsoldan EBS volume oluştur + instance'a attach et (AWS tarafı)
> $ lsblk                                # 🟢 1) yeni cihaz belirdi mi?
> nvme1n1  259:3  0  50G  0 disk         #    → var, mount point boş, format yok
> $ sudo mkfs.ext4 /dev/nvme1n1          # 🔴 2) biçimle (GERİ DÖNÜŞSÜZ)
> $ sudo mkdir -p /data                  # 🟢 3) mount point
> $ sudo mount /dev/nvme1n1 /data        # 🟡    şimdi geçici mount, test et
> $ sudo blkid /dev/nvme1n1              # 🟢 4) UUID'yi öğren
> /dev/nvme1n1: UUID="8f3b-...-c2" TYPE="ext4"
> #    /etc/fstab'a ekle:  UUID=8f3b-...-c2  /data  ext4  defaults,nofail  0  2
> $ sudo mount -a                        # 🟢    fstab'ı DOĞRULA (hata vermemeli)
> $ findmnt --verify                     # 🟢    ekstra tutarlılık denetimi
> ```
>
> **Geri alma:** Bu akıştaki tek geri-dönüşsüz adım `mkfs`'tir — cihaz adını `lsblk` ile üç kez kontrol
> et. `fstab` satırını `mount -a` ile doğrulamadan **asla reboot etme** (6.2.3'teki felaket). `nofail`
> seçeneği burada kasıtlı: disk bir gün bulunamazsa boot durmaz, sadece o mount atlanır. Tüm mount'u
> geri almak: `fstab` satırını sil, `sudo umount /data`.

![Şekil 6.1 — Cloud depolama akışı: bir EBS blok cihazının mkfs → mount → fstab (UUID) ile kalıcı bir mount'a dönüşmesi; lsblk cihazı hemen görür ama df ancak mount'tan sonra görür; yanlış bir fstab satırı boot'u kilitler.](../diagrams/png/lx-6-01-storage-stack.png)

Bu kısa akış, "instance'a disk ekleme" görevinin tamamıdır ve bu fazın çıktısıdır. Ezberleme —
her adımın **neden** orada olduğunu bil: `lsblk` (blok cihaz var mı, 6.1), `mkfs` (dosya sistemi kur,
6.2/6.3), `mount` (ağaca bağla, 6.2), `blkid`+UUID+`nofail` (boot'u kilitleme, 6.2.3), `mount -a`
(doğrula).

## 6.6.2 Kök diski büyütmek: growpart + resize2fs `[uygulama]`

İkinci çok yaygın operasyon: mevcut kök diskin doluyor ve büyütmek istiyorsun. Cloud'da bu iki katmanlı
bir iştir, ve sırası önemlidir:

1. **Cloud tarafı:** EBS volume'ünün boyutunu konsoldan artırırsın (örneğin 20G → 40G). Ama bu tek
   başına dosya sistemini büyütmez — instance içeride hâlâ eski boyutu görür. Blok cihaz büyüdü, üstteki
   katmanlar büyümedi.
2. **OS tarafı, iki adım:**
   - `growpart` — partition'ı diskin yeni sınırına kadar uzatır (`sudo growpart /dev/nvme0n1 1`).
   - `resize2fs` (ext4) veya `xfs_growfs` (xfs) — dosya sistemini partition'ın yeni sınırına kadar
     genişletir (`sudo resize2fs /dev/nvme0n1p1`).

Sıra, 6.1'deki katman modelinin aynısıdır: önce blok cihaz (EBS) büyür, sonra partition (`growpart`),
sonra dosya sistemi (`resize2fs`). Alttaki katmanı büyütmeden üsttekini büyütemezsin.

> **💡 Cloud bağlantısı — instance store neden kalıcı değil (ephemeral):** EBS kalıcıdır: instance'ı
> stop/start etsen de, terminate etmediğin sürece veri diskte kalır. Ama bazı instance tipleri bir de
> **instance store** (yerel NVMe) sunar — bu diskler fiziksel olarak host makineye bağlıdır ve **çok
> hızlıdır**, ama instance stop/terminate olduğunda (veya host değiştiğinde) **içindeki her şey silinir.**
> `lsblk`'te EBS ile aynı görünürler; farkı bilmek hayatidir. Kural: kalıcı veri (veritabanı, kullanıcı
> dosyaları) → EBS; sadece geçici/yeniden üretilebilir veri (cache, scratch, geçici derleme) → instance
> store. "Instance'ı stop/start ettim, `/mnt`'teki her şey gitti" arızasının cevabı budur: orası büyük
> ihtimalle ephemeral instance store'du.

> **🤔 Düşün 6.6** — EBS kök volume'ünü konsoldan 20G'den 60G'ye çıkardın. Birkaç dakika bekledin ama
> `df -h /` hâlâ `20G` gösteriyor, `lsblk` ise `nvme0n1` diskini `60G`, altındaki `nvme0n1p1`
> partition'ını hâlâ `19.9G` gösteriyor. Faz 6.6.2'nin katman modelini kullanarak: (a) hangi katman(lar)
> büyüdü, hangileri büyümedi, (b) hangi iki komutu hangi sırayla çalıştırırsın, (c) `df -h` çıktısı ne
> zaman `60G`'yi gösterir?
>
> *(Cevap: fazın sonunda)*

---
---

# 6.7 Bu Faz Bozulunca — Depolama Arıza İmzaları

Bu fazın komutları kalıcı veri ve boot ile oynadığı için, arızaları da en pahalı olanlardandır. Aşağıdaki
tablo, bir depolama sorununu **belirtisinden** hangi bölüme ait olduğuna hızlı götürür:

| Belirti | Muhtemel sebep | Bakılacak yer | İlgili bölüm |
|---|---|---|---|
| Instance boot etmiyor, konsol "maintenance / root password" istiyor | `fstab`'da hatalı satır (yanlış UUID/cihaz, `nofail` yok) | Konsol çıktısı; kurtarma modunda `fstab` | 6.2.3 |
| "No space left on device" ama `df -h` boş gösteriyor | Inode tükenmesi (çok sayıda küçük dosya) | `df -i` | 6.4.2 |
| Attach ettiğim disk `df`'te yok | Cihaz var ama biçimlenmemiş/mount edilmemiş | `lsblk` (cihaz var mı, mount boş mu) | 6.1.2, 6.2.1 |
| Mount ediyorum ama "unknown filesystem type" | `fstab`'daki tip diskteki gerçek formatla eşleşmiyor | `blkid` (gerçek TYPE) | 6.3.1 |
| Reboot sonrası mount kayboldu | `mount` yapılmış ama `fstab`'a yazılmamış | `cat /etc/fstab` | 6.2.2 |
| EBS'yi büyüttüm, `df` hâlâ eski boyutu gösteriyor | Partition/dosya sistemi büyütülmemiş | `lsblk` (katman boyutları) | 6.6.2 |
| Stop/start sonrası bir dizindeki veri gitti | O disk ephemeral instance store'du | Instance tipi; EBS mı instance store mu | 6.6.2 |

> **Bu tablodan çıkan ders:** Depolama arızalarının neredeyse tamamı **katman karışıklığından** doğar:
> blok cihaz mı yok, dosya sistemi mi yok, mount mu yok, kalıcılık (`fstab`) mı yok. Bir belirtiyle
> karşılaştığında panik yapıp "disk bozuk" demeden önce, 6.1'in katman modelini yukarıdan aşağı yürü:
> `lsblk` (cihaz var mı?) → `blkid` (format var mı?) → `df`/`mount` (bağlı mı?) → `cat /etc/fstab`
> (kalıcı mı?). Yedi arızanın altısı bu dört komutla teşhis edilir. En pahalısı olan `fstab`-boot
> kilidi ise hiç yaşanmaz eğer tek bir alışkanlığın varsa: **her `fstab` düzenlemesinden sonra, reboot
> etmeden önce `sudo mount -a`.**

---
---

# Faz 6 — Düşün sorularının cevapları

## Cevap 6.1 — Disk "bozuk değil", sadece ham

Diski `df`'in göstermemesi tamamen beklenen bir durum, çünkü `df` **mount edilmiş dosya sistemlerini**
listeler; ham blok cihazları değil. `lsblk` çıktısı diskin sağlıklı olduğunu zaten kanıtlıyor: cihaz
görünüyor (`TYPE disk`, `SIZE 50G`), yani çekirdek onu tanıyor ve attach başarılı. Görünmemesinin sebebi
6.1.1'deki iki-katman modeli: cihaz (blok katmanı) var, ama üstünde henüz **dosya sistemi yok** ve
hiçbir yere **mount edilmemiş**. `df` sadece ikinci katmanı bilir. Diski `df`'e sokmak için iki adım
gerekir: (1) biçimle (`mkfs.ext4 /dev/nvme1n1`) — üstüne bir dosya sistemi kur; (2) mount et
(`mount /dev/nvme1n1 /data`) — dizin ağacına bağla. Bundan sonra `df` onu gösterir.
**İlgili bölüm:** 6.1.1-6.1.2 · **Devamı:** 6.6.1 tam akış.

## Cevap 6.2 — `fstab` cihaz adıyla yazıldı ve isim kaydı; `nofail` de yoktu

(a) Neredeyse kesin olarak şu oldu: `fstab`'a **cihaz adıyla** (`/dev/nvme1n1`) yazıldı. Reboot'ta
çekirdeğin disklere verdiği isimler yer değiştirdi (belki başka bir volume de attach'lıydı), böylece
boot anında `/dev/nvme1n1` ya yoktu ya başka bir diske işaret ediyordu. `nofail` seçeneği de olmadığı
için, mount başarısız olunca systemd boot'u durdurdu ve kurtarma/maintenance moduna düştü — SSH
başlamadı çünkü sistem çok-kullanıcılı hedefe (Faz 5.3.4) hiç ulaşamadı. (b) İki ayrı seçim bu felaketi
önlerdi: **(1) cihaz adı yerine UUID kullanmak** — UUID diske özgü ve değişmez, isim kayması onu
etkilemez (6.2.3); **(2) `nofail` seçeneğini eklemek** — böylece disk bir sebeple bulunmasa bile boot
durmaz, sadece o mount atlanır ve makine erişilebilir kalır. İkisi birlikte cloud'da standart pratiktir.
**İlgili bölüm:** 6.2.3 · **Devamı:** 6.6.1'deki `mount -a` doğrulama alışkanlığı.

## Cevap 6.3 — Tür `mkfs`'te yazılır; yanlış `fstab` türü mount'u düşürür; xfs küçültülemez

(a) Mount, yanlış dosya sistemi türü hatasıyla **başarısız olur**: tür, `mkfs` sırasında **diske
yazılır** ve diskin kalıcı bir özelliğidir (6.3.1); `fstab`'taki tür alanı buna uymak zorundadır. Olduğu
gibi bırakılırsa aynı satır boot'ta da düşer — `nofail` yoksa bu boot'u kilitleyebilir (6.2.3). (b)
`blkid` diskin gerçek `TYPE` değerini gösterir — burada `xfs`. Çözüm **`fstab` satırını `xfs` olarak
düzeltmektir**; diskte bir sorun yok, yeniden biçimlendirmek verisini silerdi. Ardından `sudo mount -a`
ile doğrula (6.6.1). (c) xfs **büyütülebilir ama küçültülemez** (6.3.1). Pratik yol: yeni, daha küçük
bir volume oluştur, veriyi kopyala, mount'u ona geçir ve eskisini emekli et — küçültme ileride sık
gerekecekse `mkfs` aşamasında ext4 seç.
**İlgili bölüm:** 6.3.1 · **Devamı:** 6.2.3 (`fstab` boot tuzağı).

## Cevap 6.4 — Diski büyütmek yanlış katmanı hedefler; suçlu inode

(a) Diski büyütmek bu sorunu büyük olasılıkla çözmez çünkü `df -h /var` **%61** diyor — yani **veri
bloğu** bütçesi dolu değil, bol yer var. "No space" hatasıyla çelişen tek şey ikinci, gizli bütçedir:
inode'lar (6.4.1). Diski büyütmek veri bloğu bütçesini artırır ama inode tükenmesini çözmez (klasik
ext4'te inode sayısı `mkfs`'te sabitlenir). (b) Teşhisi kesinleştiren tek komut: `df -i /var` — inode
kullanımını gösterir. (c) Beklenti: `IUse%` **%100** çıkar. O zaman gerçek kök sebep sınıfı **inode
tükenmesidir**: bir yerde çok sayıda çok küçük dosya birikmiş (cache, kuyruk, log parçaları). Çözüm
diski büyütmek değil, o küçük dosyaları temizlemek ve onları üreten davranışı düzeltmektir.
**İlgili bölüm:** 6.4.2 · **Devamı:** 6.7 arıza tablosu (satır 2).

## Cevap 6.5 — LVM alan üretmez; önce PV ekle, sonra LV'yi ve dosya sistemini büyüt

(a) LVM alan yaratmaz, havuzda olanı yönetir: VG'de boş yer yoksa LV'ye eklenecek boyutun alınacağı bir
yer yoktur (6.5.1, yaygın yanılgı kutusu). (b) Önce **yeni bir EBS volume attach et** — LVM'in
sağlayamadığı gerçek disk bu — sonra onu **PV** yap, **VG'ye ekle** ve ancak ondan sonra **LV'yi
büyüt**: aşağıdan yukarıya, PV → VG → LV. (c) LV'nin üstündeki dosya sistemi ayrı bir katmandır ve
kendiliğinden büyümez; onu genişletene kadar (`resize2fs` ext4 için, `xfs_growfs` xfs için) `df -h
/data` değişmez. Bu, 6.6.2'nin kuralının aynısı: alt katman büyür, üsttekiler otomatik takip etmez.
**İlgili bölüm:** 6.5.1 · **Devamı:** 6.6.2 (kök diski büyütme).

## Cevap 6.6 — Sadece alt katman (EBS) büyüdü; partition ve dosya sistemi geride kaldı

(a) `lsblk` cevabı veriyor: `nvme0n1` (blok cihaz) `60G` — yani **EBS katmanı büyüdü**. Ama
`nvme0n1p1` (partition) hâlâ `19.9G` ve dosya sistemi de onun içinde sıkışmış — yani **partition ve
dosya sistemi katmanları büyümedi**. 6.6.2'deki kural: alttaki katman büyür, üsttekiler otomatik
büyümez. (b) İki komut, bu sırayla: önce `sudo growpart /dev/nvme0n1 1` (partition'ı diskin yeni
sınırına uzat), sonra `sudo resize2fs /dev/nvme0n1p1` (ext4 dosya sistemini partition'ın yeni sınırına
genişlet; xfs olsaydı `xfs_growfs`). (c) `df -h /` ancak `resize2fs` bittikten sonra `60G` gösterir —
çünkü `df` en üstteki katmanı (dosya sistemi) okur, ve o katman en son büyüyendir.
**İlgili bölüm:** 6.6.2 · **Devamı:** 6.7 arıza tablosu (satır 6).

---
---

# Faz 6 — Sık sorulan sorular

**S1 — Bir veri diskini partition'lamalı mıyım, yoksa tüm cihazı doğrudan biçimleyebilir miyim?**
Cloud'da tek amaçlı bir veri diski için genelde **partition'sız**, tüm cihazı doğrudan biçimlemek
(`mkfs.ext4 /dev/nvme1n1`) yaygın ve tamamen geçerlidir — bir katman az, bir tuzak az. Partition,
tek diski birden çok bölüme ayırman gerektiğinde veya boot/EFI düzeni için gerekir. Kök disk neredeyse
her zaman partition'lıdır çünkü EFI ve boot ayrı bölümler ister.

**S2 — `mount` ile `fstab` arasındaki fark tam olarak ne?** `mount` **şimdi, bir kez** bağlar ve
reboot'ta kaybolur (Faz 5'teki `systemctl start` gibi geçici). `fstab` **her boot'ta otomatik** bağlar
(`systemctl enable` gibi kalıcı). İkisi bağımsızdır: `fstab`'a yazmak diski hemen mount etmez (`mount -a`
gerekir); `mount` yapmak da onu kalıcı yapmaz. Doğru kalıcı kurulum ikisini de içerir.

**S3 — `nofail` seçeneğini her zaman koymalı mıyım?** Veri diskleri için evet, güçlü bir pratik: disk
bir gün bulunamazsa (yanlış attach, silinmiş volume) boot'un durmasını değil, o mount'un atlanmasını
istersin. Kök disk için `nofail` **konulmaz** — kök yoksa zaten çalışacak bir sistem yoktur, orada
başarısızlığın görünür olması gerekir.

**S4 — ext4 mı xfs mi seçmeliyim?** Özel bir sebebin yoksa dağıtımın varsayılanını kullan
(Ubuntu→ext4, Amazon Linux→xfs). Çok büyük dosyalar / yüksek paralel yazma → xfs biraz avantajlı. Diski
ileride **küçültme** ihtimalin varsa → ext4 (xfs küçülemez). Günlük işte fark çoğu zaman fark edilmez.

**S5 — `df -h` disk dolu diyor ama büyük dosya bulamıyorum, ne yapmalıyım?** İki klasik sebep: (1)
**silinen ama hâlâ açık dosya** — bir süreç büyük bir logu tutmaya devam ederken dosya silinmiş; alan
süreç ölene kadar geri gelmez (`lsof | grep deleted`). (2) Bir **mount point'in altına saklanmış**
dosyalar — bir dizine mount edilmeden önce oraya yazılmış veri. `df -i` ile inode'u da ele; bu Faz
6.4'ün senaryosu.

**S6 — Bir diski güvenle nasıl çıkarırım (detach)?** Sıra tersine döner: önce `sudo umount /data`
(dosya sistemini ağaçtan ayır), **sonra** konsoldan detach. Mount'luyken detach etmek veri kaybı ve
asılı I/O riski taşır. `umount` "target is busy" derse, bir süreç hâlâ o dizini kullanıyordur
(`lsof /data` ile bul). Kalıcı mount'sa `fstab` satırını da sil, yoksa bir sonraki boot onu arar.

**S7 — LVM'i cloud'da kullanmalı mıyım?** Zorunlu değil. Basit "bir instance, bir-iki veri diski"
senaryosunda doğrudan-cihaz modeli daha az karmaşıktır. LVM, birden çok diski tek mantıksal birim
yapman veya kesintisiz online büyütme esnekliği istediğin daha ileri durumlarda kazandırır. Birçok
dağıtım kök diski yine de LVM üstüne kurar; `lsblk`'te `lvm` satırları görürsen bu yüzdendir.

---
---

# Faz 6 — Kendini sına

Cevaplarını bir kâğıda yaz, sonra cevap anahtarıyla karşılaştır. Hedef: 18 sorudan 14+.

## Bölüm A — Tanım ve mekanizma (kavramı biliyor musun)

1. Blok cihaz ile dosya sistemini bir cümleyle ayırt et. Hangisini `mkfs` oluşturur?
2. `lsblk` ile `df -h` her ikisi de "diskleri" gösterir gibi durur. Aralarındaki temel fark nedir —
   hangisi ham cihazı, hangisi mount edilmiş dosya sistemini listeler?
3. Bir diski kullanılabilir hâle getirmenin dört adımını sırayla say.
4. Mount point nedir? "Mount, altındaki dizini gizler" ne demek?
5. `/etc/fstab`'da cihaz adı yerine neden UUID kullanılır? UUID nereden öğrenilir?
6. Journaling hangi problemi çözer? Neden cloud'da özellikle önemlidir?
7. Inode nedir ve neyi **tutmaz**? (İpucu: dosyanın adı nerede yaşar?)
8. `df -h` ile `df -i` neyi ayrı ayrı ölçer?

## Bölüm B — Uygula ve teşhis et (senaryoda kullanabiliyor musun)

9. Yeni attach edilmiş bir EBS volume'ünü kalıcı mount etmek için komutları doğru sırayla yaz
   (`lsblk`'ten `mount -a`'ya kadar). Her adımın yanına 🟢/🟡/🔴 riski koy.
10. Bir instance reboot sonrası açılmıyor, konsol "maintenance mode / root password" istiyor. İlk
    şüphelenmen gereken dosya hangisi ve neden?
11. "No space left on device" hatası alıyorsun ama `df -h` diskin %55 boş olduğunu gösteriyor. Hangi
    tek komutu çalıştırırsın ve ne görmeyi beklersin?
12. EBS volume'ünü 30G'den 80G'ye çıkardın ama `df -h /` hâlâ 30G gösteriyor. Hangi iki komutu hangi
    sırayla çalıştırırsın?
13. `fstab`'a yeni bir satır ekledin. Reboot etmeden önce hangi tek komutla onu doğrularsın, ve neden
    bu adım pazarlık konusu değil?
14. Bir dizindeki veri, instance'ı stop/start ettikten sonra kayboldu. Muhtemel sebep nedir ve o disk
    ne tür bir depolamaydı?

## Bölüm C — Muhakeme ve bağlantı (neden'i açıklayabiliyor musun)

15. `nofail` seçeneği bir veri diskinde neden iyi bir fikir ama kök diskte neden **kötü** bir fikirdir?
16. Faz 5'teki `.mount` unit'leri ve `systemctl enable`≠`start` ayrımı, bu fazın `fstab`≠`mount`
    ayrımıyla nasıl aynı kavramın iki yüzüdür?
17. Bir meslektaşın "inode sorununu çözmek için diski büyütelim" diyor. Bunun neden yanlış katmanı
    hedeflediğini bir cümleyle açıkla.
18. `mkfs`'i neden bu fazın en tehlikeli komutu sayıyoruz? Onu güvenli kullanmanın tek altın kuralı ne?

---

## Cevap anahtarı

1. Blok cihaz = numaralandırılmış ham blokların dizisi (yapı yok); dosya sistemi = üstüne yazılan
   dosya/dizin organizasyonu. `mkfs` **dosya sistemini** oluşturur (6.1.1). — 2. `lsblk` **ham blok
   cihazları** ve ağacı listeler (mount olmasa da); `df -h` yalnızca **mount edilmiş dosya sistemlerini**
   ve doluluğunu (6.1.2). — 3. Bölme (partition, isteğe bağlı) → biçimleme (`mkfs`) → mount → kalıcı
   yapma (`fstab`) (6.2.1). — 4. Mount point, bir dosya sisteminin dizin ağacına bağlandığı dizindir;
   oraya bir disk mount edilince altında önceden var olan dosyalar görünmez olur (mount kalkınca geri
   gelir) (6.2.2). — 5. Cihaz adları reboot/attach sırasında değişebilir; UUID diske özgü ve sabittir,
   böylece yanlış diski mount etme / boot kilitlenmesi önlenir. `blkid` ile öğrenilir (6.2.3). — 6.
   Yarım kalan yazımdan sonra dosya sistemini hızlıca **tutarlı** hâle döndürür; cloud'da ani stop/spot
   geri alma/host arızası sık olduğu için kritiktir (6.3.2). — 7. Inode, bir dosyanın metadata kimlik
   kartıdır (boyut, sahip, izinler, zaman, veri bloklarının listesi); dosyanın **adını tutmaz** — ad
   dizinde "ad→inode" eşlemesi olarak yaşar (6.4.1). — 8. `df -h` **veri bloğu** bütçesini; `df -i`
   **inode** bütçesini ayrı ayrı ölçer; ikisi bağımsızdır (6.4.1-6.4.2).

9. `lsblk` 🟢 → `mkfs.ext4 /dev/nvme1n1` 🔴 → `mkdir /data` 🟢 → `mount /dev/nvme1n1 /data` 🟡 →
   `blkid` 🟢 → `fstab`'a UUID satırı ekle 🔴 → `mount -a` 🟢 (6.6.1). — 10. `/etc/fstab` — hatalı bir
   satır (yanlış cihaz/UUID, `nofail` yok) boot'ta mount'u başarısız kılıp maintenance moduna düşürür
   (6.2.3). — 11. `df -i` — inode kullanımının **%100** olmasını beklersin (inode tükenmesi) (6.4.2). —
   12. `sudo growpart /dev/nvme0n1 1` sonra `sudo resize2fs /dev/nvme0n1p1` (6.6.2). — 13. `sudo
   mount -a` (veya `findmnt --verify`); çünkü doğrulanmamış hatalı bir `fstab` satırı bir sonraki boot'u
   kilitler ve makineyi erişilemez yapar (6.2.3, 6.6.1). — 14. O disk büyük olasılıkla **ephemeral
   instance store**'du; stop/start'ta içeriği silinir — kalıcı veri için EBS gerekir (6.6.2).

15. Veri diskinde `nofail`: disk bulunamazsa boot durmasın, sadece o mount atlansın (makine erişilebilir
    kalır). Kök diskte `nofail` kötüdür çünkü kök yoksa çalışan bir sistem yoktur; başarısızlığın
    gizlenmesi değil görünür olması gerekir (6.2.3, S3). — 16. İkisi de "geçici çalışan durum" ile
    "kalıcı, boot'ta tekrar kurulan yapılandırma" ayrımıdır: `start`/`mount` şimdi etkir, reboot'ta
    gider; `enable`/`fstab` ise her boot'ta otomatik tekrar kurar (5.3.3, 6.2.2). — 17. Diski büyütmek
    **veri bloğu** bütçesini artırır; inode tükenmesi ayrı bir bütçedir (`df -i`), o yüzden büyütmek
    sorunu çözmez — çözüm küçük dosyaları temizlemektir (6.4.2). — 18. `mkfs` bir diski **geri
    dönüşsüz** biçimler (var olan her şeyi siler); altın kural: çalıştırmadan önce cihaz adını `lsblk`
    ile üç kez doğrula, asla tahminle yazma (6.2.1, 6.6.1).

## Puanlama

| Doğru sayısı | Ne anlama geliyor |
|---|---|
| 16-18 | Depolama katmanlarını ve `fstab` tuzağını sağlam kavradın. Ara Sınav 3'e hazırsın. |
| 13-15 | İyi. Kaçırdığın soruların bölümlerini bir kez daha oku (özellikle 6.2.3 ve 6.4). |
| 9-12 | Temel var ama kırılgan. 6.2 (mount/fstab) ve 6.6'yı (cloud akışı) tekrar çalış. |
| 0-8 | Fazı yeniden gez; `lsblk`/`df -h`/`df -i`/`blkid` komutlarını bir test diskinde bizzat çalıştır. |

Kaçırdığın soru → dönmen gereken bölüm:

| Soru | Bölüm |
|---|---|
| 1, 2 | 6.1 Blok cihaz vs dosya sistemi |
| 3, 4, 9 | 6.2.1-6.2.2 Adımlar ve mount point |
| 5, 10, 13 | 6.2.3 fstab ve UUID |
| 6 | 6.3.2 Journaling |
| 7, 8, 11, 17 | 6.4 Inode |
| 12, 14, 18 | 6.6 Cloud akışı ve büyütme |
| 15, 16 | 6.2.3 + Faz 5 bağlantısı |

---
---

# Faz 6 — Kapanış ve Faz 7'ye Köprü

## Bu fazdan ne taşıyorsun

Faz 6 sana **kalıcı katmanı** verdi. Artık bir diski katman katman görüyorsun: ham blok cihaz
(`lsblk`) → dosya sistemi (`mkfs`, `blkid`) → mount (`mount`, mount point) → kalıcılık (`/etc/fstab`,
UUID). Bu model üç ayrı arızayı bir bakışta ayırmanı sağlıyor: cihaz mı yok, format mı yok, mount mu
yok, kalıcılık mı yok. En pahalı arızayı — `fstab`'ın boot'u kilitlemesini — önleyen alışkanlığı da
öğrendin: **her `fstab` düzenlemesinden sonra `mount -a`.** Ve "yer var ama disk dolu" gibi imkânsız
görünen inode arızasını `df -i` ile teşhis edebiliyorsun.

## Faz 7 bunun neresine bağlanıyor

Şimdiye kadar tek bir makinenin içindeydik: process (Faz 3), bellek (Faz 4), boot (Faz 5), disk (Faz
6). Faz 7 makinenin **dış dünyayla konuşmasına** geçiyor: ağ. Ve depolama ile ağ, cloud'da aynı
"instance'a bir şey ekle" hikâyesinin iki yüzüdür — Faz 6'da bir diski attach edip mount ettin; Faz
7'de bir ağ arayüzünü, IP'yi, portu ve güvenlik kurallarını tanıyacaksın. İki fazın ortak dersi şu
olacak: cloud'da bir kaynağın **var olması** (disk attach'lı / arayüz ayakta) ile **kullanılabilir
olması** (mount'lu / port dinleniyor ve güvenlik grubu izin veriyor) ayrı şeylerdir — ve arızaların
çoğu tam bu boşlukta yaşar.

> **🤔 Faz çıktısı — kendine sor:** Bir instance'a hem yeni bir EBS volume attach ettin (Faz 6) hem de
> üstünde bir web servisi çalıştırmak istiyorsun (Faz 7). Faz 6'da "disk var ama mount yok → `df`'te
> yok" arızasını gördün. Faz 7'ye geçmeden tahmin et: bir servis "port'u dinliyor" ama dışarıdan
> erişilemiyorsa, bunun disk örneğindeki **hangi** boşluğa benzediğini düşün — "var olmak" ile
> "erişilebilir olmak" arasındaki hangi katman eksik olabilir?
>
> **🧪 Lab 6 fikri (kendi test instance'ında):** (1) Bir EBS volume oluştur, attach et, `lsblk`'te
> belirmesini izle. (2) `mkfs.ext4` ile biçimle, `/data`'ya mount et, `df -h`'te gör. (3) `blkid` ile
> UUID'yi al, `fstab`'a `nofail` ile ekle, `sudo mount -a` ile **doğrula**. (4) Bilerek `fstab`'a
> `nofail` olmadan **yanlış** bir UUID yaz, `mount -a`'nın nasıl hata verdiğini gör (ama reboot ETME) —
> sonra düzelt. (5) Volume'ü konsoldan büyüt, `growpart`+`resize2fs` ile `df -h`'in yeni boyutu
> göstermesini izle. Bu beş adım, bu fazın tüm mekanizmasını elinde toplar.

---

> **Navigasyon:** [◀ Faz 5 — Boot, Init ve systemd](Faz_5_Boot_Init_systemd.md) · **Faz 6** · [Ara Sınav 3 ▶](Ara_Sinav_3.md)


