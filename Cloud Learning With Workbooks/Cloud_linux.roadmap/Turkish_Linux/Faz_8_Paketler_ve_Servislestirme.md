# Faz 8 — Paketler, Yazılım ve Servisleştirme

> **Navigasyon:** [◀ Faz 7 — Ağ ve Bağlantı](Faz_7_Ag_ve_Baglanti.md) · **Faz 8** · [Faz 9 — Güvenlik ve Sertleştirme ▶](Faz_9_Guvenlik_ve_Sertlestirme.md)

---

## Nereden geliyoruz

Faz 7'nin sonunda sana bir soru bıraktık: bir uygulamayı `apt install` ile kurdun ama `systemctl status`
"not found" diyor — "kurulu olmak" ile "bir servis olarak çalışıyor olmak" arasında hangi adım eksik?
Cevap tam bu fazın konusu. Faz 5-7 boyunca hep **zaten var olan** servislerle çalıştın: SSH dinliyordu,
nginx ayaktaydı, systemd onları yönetiyordu. Ama o yazılımlar makineye **nasıl geldi**, ve senin kendi
uygulaman nasıl onlar gibi "systemd'nin yönettiği bir servis" olur? Faz 8 bunu anlatıyor.

Üç fazın bilgisi burada doğrudan birleşiyor:

- **Faz 5 — servisler unit'tir.** Bir `.service` unit dosyasının systemd tarafından nasıl başlatıldığını,
  `active (running)` durumunu, `journalctl -u` ile logunu gördün (5.3). Bu fazda o unit dosyasını **sen
  yazacaksın** — kendi uygulaman için.
- **Faz 3 — process ve onun ölümü.** `nohup python app.py &` ile arka plana attığın bir process'in neden
  kırılgan olduğunu (oturum kapanınca ölür, çökünce geri gelmez) Faz 3'ün process modeliyle anlayacaksın.
- **Faz 7 — servis ağa açılır.** "Servise bağlanamıyorum" olayının en dıştaki kapısı "servis kurulu ve
  çalışıyor mu"ydu. Bu faz o kapının arkasını — kurulum ve servisleştirme — dolduruyor.

## Bu fazın sorusu

Faz 7 servisi dış dünyaya bağladı. Bu faz servisi makineye getiriyor ve kalıcı kılıyor:

> *"Bir yazılım makineye nasıl girer, ve kendi uygulamamı makine yeniden başladığında bile kendiliğinden
> ayağa kalkan, çökünce kendini toparlayan gerçek bir servise nasıl dönüştürürüm?"*

Bir cloud engineer için bu iki soru iç içedir. Yazılımın makineye girişi neredeyse her zaman bir **paket
yöneticisinden** (`apt`) geçer — ve paket yöneticisi sadece "dosya kopyalayan" bir araç değil, sürümleri,
bağımlılıkları ve imzaları yöneten bir sistemdir; bozulduğunda ("kırık bağımlılık", "repo ulaşılamıyor")
tüm kurulumu durdurur. Kendi uygulamanı çalıştırmak ise bir dosyayı elle başlatmaktan ibaret değildir:
üretimde bir uygulama **kendiliğinden** başlamalı, çökünce **kendini yeniden başlatmalı**, ve loglarını
**merkezi bir yere** yazmalıdır. Bunu sağlayan şey, Faz 5'te tüketici olarak gördüğün systemd'yi şimdi
**üretici** olarak kullanmaktır: kendi `.service` unit'ini yazmak.

Bu fazın sonunda, "kurulu olmak ≠ servis olarak çalışıyor olmak" ayrımını kuracak; bir uygulamayı
`nohup ... &` gibi kırılgan bir yolla değil, systemd unit'i ile üretime uygun şekilde servisleştirecek;
ve bulutun "sunucuyu elle yamama, yeniden inşa et" (immutable) felsefesini anlayacaksın.

---

## Bu fazın sonunda

- `apt`'in (ve alttaki `dpkg`'nin) bir paketi nasıl kurduğunu; repository, GPG imzası ve sürüm sabitleme
  (pinning) kavramlarını açıklayabileceksin
- "Kırık bağımlılık", "repo ulaşılamıyor" ve "sürüm sürüklenmesi" arızalarını tanıyıp nereye bakacağını
  bileceksin
- Kendi Python/FastAPI uygulamanı bir systemd `.service` unit'i olarak yazabilecek; auto-restart ve
  merkezi log'u yapılandırabileceksin
- `nohup python app.py &`'in neden üretime uygun olmadığını (oturum bağımlılığı, restart yok, log
  dağınıklığı) somut olarak açıklayabileceksin
- Kaynaktan derlemenin (`./configure && make && make install`) ne olduğunu **kavram düzeyinde** bilecek;
  ne zaman gerektiğini (ve genelde neden gerekmediğini) ayırt edebileceksin
- **Cloud:** paketleri "baked-in" bir AMI ile önceden pişirmenin, sürüm pinning ile tekrarlanabilirliğin
  ve "sunucuyu elle yamama, yeniden inşa et" (immutable) felsefesinin neden bulut operasyonunun temeli
  olduğunu anlatabileceksin

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden burada |
|---|---|---|---|
| 8.1 | Paket yöneticileri | `[mekanizma]` | Yazılım makineye nasıl girer: `apt`, `dpkg`, repo, imza |
| 8.2 | Kendi uygulamanı servisleştirmek | `[uygulama]` | **Fazın kalbi** — `nohup &` değil, systemd unit |
| 8.3 | Kaynaktan derleme (farkındalık) | `[atla]` | `./configure && make` — ne zaman gerekir |
| 8.4 | Immutable yaklaşım | `[kavram]` | "Elle yamama, yeniden inşa et" — bulut felsefesi |
| 8.5 | Bu faz bozulunca | — | Paket ve servisleştirme arıza imzaları |

> **Bu fazda nasıl çalışmalı:** Bu fazın gözlem komutları 🟢'dir (`apt list --installed`, `apt-cache
> policy`, `systemctl status`, `journalctl -u`). Kurulum ve servisleştirme adımları 🟡/🔴 içerir:
> `apt install/remove` sistemi değiştirir (🟡, `apt` bunları geri alınabilir tutar ama bir paketi
> kaldırmak bağımlı servisleri durdurabilir 🔴). En öğretici deney: basit bir `hello.service` unit'i
> yazıp `systemctl` ile başlatmak, sonra kasten çökertip auto-restart'ın devreye girişini `journalctl` ile
> izlemek. Kendi test makinende çalış — paket kaldırma denemelerini üretim makinesinde yapma.

---
---

# 8.1 Paket Yöneticileri

## 8.1.1 apt ve dpkg: bir paket nasıl kurulur `[mekanizma]`

Linux'ta yazılım kurmanın standart yolu **paket yöneticisidir**. İki büyük aile vardır: Debian/Ubuntu'da
**`apt`** (altında **`dpkg`**), RHEL/Amazon Linux'ta **`dnf`/`yum`** (altında **`rpm`**). Bu fazda
Ubuntu'ya odaklanıyoruz, ama zihinsel model her ikisinde aynıdır — sadece komut isimleri değişir.

Katmanları ayırmak önemli: **`dpkg`** alt katmandır; tek bir `.deb` dosyasını açar, dosyalarını sisteme
(çoğu `/usr/bin`, `/etc`, `/lib` altına) yerleştirir ve kaydını tutar. Ama `dpkg` **bağımlılıkları
çözmez** — bir paket başka paketlere ihtiyaç duyuyorsa onları kendi bulup kurmaz. İşte **`apt`** üst
katmandır: hangi paketin nelere ihtiyaç duyduğunu (bağımlılık ağacını) çözer, gerekli her şeyi bir
**repository**'den indirir, ve `dpkg`'yi doğru sırayla çağırır. Yani `apt install nginx` dediğinde: apt
nginx'in bağımlılıklarını hesaplar, hepsini indirir, dpkg ile sırayla kurar.

> **🔧 Makinende gör** 🟢 — kurulu paketleri ve bir paketin durumunu oku
>
> ```
> $ apt list --installed | wc -l          # kaç paket kurulu
> 1847
> $ apt-cache policy nginx                # hangi sürüm kurulu, hangisi mevcut, nereden
>   Installed: 1.18.0-6ubuntu14.4
>   Candidate: 1.18.0-6ubuntu14.4
>   Version table: ...
> $ dpkg -L nginx | head                  # bu paket hangi dosyaları koydu
> /usr/sbin/nginx
> /etc/nginx/nginx.conf
> ...
> ```
>
> `apt-cache policy` sana üç kritik bilgiyi verir: **kurulu** sürüm, **aday** (kurulabilir en yeni) sürüm,
> ve hangi **repo**'dan geldiği. `dpkg -L` ise bir paketin diske tam olarak neleri koyduğunu gösterir —
> "bu komut hangi paketten geldi" veya "config dosyası nerede" sorusunun cevabı.

## 8.1.2 Repository, GPG imzası ve sürüm sabitleme `[kavram]`

Paketler nereden gelir? Bir **repository** (repo), paketlerin ve onların meta verisinin (sürüm,
bağımlılık) tutulduğu bir sunucudur. Makinen hangi repolara güveneceğini `/etc/apt/sources.list` ve
`/etc/apt/sources.list.d/` altındaki dosyalardan bilir. `apt update` bu repoların **paket listesini**
tazeler (paketleri değil — sadece "neler mevcut" bilgisini); `apt upgrade` ise kurulu paketleri yeni
sürümlerine yükseltir.

İki kavram güvenlik ve kararlılık için kritiktir:

- **GPG imzası.** Bir repodan gelen paketler kriptografik olarak imzalanır; apt, indirdiği paketin gerçekten
  o repodan geldiğini ve yolda değiştirilmediğini imzayla doğrular. "Repository is not signed" gibi hatalar
  bu güven zincirinin kopmasıdır — apt, doğrulayamadığı bir paketi kurmayı reddeder (doğru davranış).
- **Sürüm sabitleme (pinning).** Bir paketi belirli bir sürümde **kilitlemek**. Üretimde "her `apt upgrade`
  bir servisin sürümünü değiştirsin" istemezsin — beklenmedik bir sürüm bir uygulamayı bozabilir. Pinning
  ile bir paketi bilinçli olarak sabit tutarsın; tekrarlanabilirliğin (8.4) temelidir.

> **⚠️ Yaygın yanılgı: "`apt update` paketlerimi günceller."**
>
> Hayır — bu en sık karıştırılan ikilidir. **`apt update`** sadece repoların **paket listesini** tazeler:
> "hangi paketlerin hangi yeni sürümleri mevcut" bilgisini indirir, tek bir paketi bile güncellemez.
> Paketleri asıl yükselten **`apt upgrade`**'dir. Doğru sıra hep `apt update` (listeyi tazele) → sonra
> `apt install`/`apt upgrade` (kur/yükselt). "Paketi bulamıyor" hatasının sık sebebi, `apt update`
> yapmadan eski bir listeyle iş görmeye çalışmaktır.

> **🤔 Düşün 8.1** — Bir sunucuda `apt install yeni-arac` diyorsun ama "Unable to locate package
> yeni-arac" hatası alıyorsun — oysa bu aracın var olduğundan eminsin. Aynı sunucuda başka biri dün bu
> repoyu `sources.list.d`'ye yeni eklemiş. Hangi tek komutu **önce** çalıştırman gerekirdi, ve o komut tam
> olarak neyi tazeler (paketleri mi, yoksa başka bir şeyi mi)?
>
> *(Cevap: fazın sonunda)*

---
---

# 8.2 Kendi Uygulamanı Servisleştirmek

## 8.2.1 Neden `nohup python app.py &` üretime uygun değil `[uygulama]`

Kendi uygulamanı çalıştırmanın en hızlı yolu şu gibi görünür:

```
$ nohup python app.py &        # arka plana at, oturum kapansa da ölmesin
```

Bu bir test için idare eder ama üretimde **üç ölümcül eksiği** vardır — ve her biri önceki fazların
bilgisiyle açıklanır:

1. **Çökünce geri gelmez (Faz 3).** Process bir hata alıp ölürse (segfault, unhandled exception, OOM
   killer — Faz 4!), onu geri başlatan kimse yoktur. Uygulaman gecenin üçünde sessizce ölür, sabah kimse
   fark edene kadar kapalı kalır.
2. **Boot'ta başlamaz (Faz 5).** Makine yeniden başlarsa (kernel güncellemesi, bulutta instance replace)
   `nohup` ile başlattığın process geri gelmez. `systemctl enable` gibi bir "boot'ta başlat" mekanizması
   yoktur.
3. **Logu dağınıktır.** Çıktı `nohup.out` diye bir dosyaya birikir; döndürülmez (rotate), merkezi değildir,
   `journalctl -u` ile diğer servislerle birlikte okunamaz. Faz 5'teki tek pencereden log okuma refleksin
   burada işe yaramaz.

Kısacası: `nohup ... &` bir process **başlatır** ama onu **yönetmez**. Üretim, yönetilen bir servis ister —
ve onu yöneten şey systemd'dir. Aşağıdaki şekil iki yolu yan yana koyuyor: aynı uygulama, iki farklı akıbet.

![Şekil 8.1 — Bir betikten yönetilen bir servise: aynı uygulama (app.py) iki yolla çalıştırılabilir. Solda `nohup python app.py &` — bir process başlar ama kimse yönetmez: çökünce geri gelmez, boot'ta başlamaz, logu dağınıktır, oturum kapanınca ölür. Sağda `myapp.service` (systemd unit) — yönetilen bir servis: `Restart=on-failure` ile otomatik yeniden başlar, `systemctl enable` ile boot'ta başlar, journald'a merkezi log yazar, oturumdan bağımsızdır. Ders: diskte kurulu olmak, yönetilen bir servis olarak çalışmakla aynı şey değildir.](../diagrams/png/lx-8-01-script-to-service.png)

## 8.2.2 Bir systemd unit'i yazmak: uygulamanı gerçek bir servise dönüştür `[uygulama]`

Faz 5'te systemd'yi bir **tüketici** olarak gördün (başkasının yazdığı unit'leri okudun). Şimdi **üretici**
olacaksın: kendi uygulaman için bir `.service` unit dosyası yazacaksın. Dosya `/etc/systemd/system/`
altına konur; örn. `/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My FastAPI app
After=network.target

[Service]
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/venv/bin/uvicorn app:app --host 0.0.0.0 --port 8000
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Her satır 8.2.1'deki bir eksiği kapatır: `Restart=on-failure` → çökünce systemd otomatik yeniden başlatır
(eksik 1 çözüldü); `WantedBy=multi-user.target` + `systemctl enable` → boot'ta başlar (eksik 2 çözüldü);
stdout/stderr otomatik journald'a gider → `journalctl -u myapp` (eksik 3 çözüldü). `After=network.target`
Faz 7'ye bir selam: ağ hazır olmadan başlatma. `--host 0.0.0.0` ise Faz 7.4.2'nin dersi: dışarıdan
erişilebilir olsun.

> **🔧 Makinende gör** 🟡 — bir servisi tanıt, başlat, boot'a ekle
>
> ```
> $ sudo systemctl daemon-reload          # 🟡 yeni/değişen unit dosyasını systemd'ye okut
> $ sudo systemctl start myapp            # 🟡 şimdi başlat
> $ sudo systemctl enable myapp           # 🟡 boot'ta da başlasın
> $ systemctl status myapp                # 🟢 durumu gör (active running mı)
> $ journalctl -u myapp -f                # 🟢 logunu canlı izle
> ```
>
> Kritik refleks: bir unit dosyasını **her değiştirdiğinde** `sudo systemctl daemon-reload` yapmalısın —
> yoksa systemd hâlâ eski tanımı kullanır. "Unit'i düzelttim ama değişmedi" arızasının bir numaralı sebebi
> budur. `start` ile `enable` ayrı şeydir (Faz 5.3): `start` şimdi başlatır, `enable` boot'a ekler — ikisi
> de gerekir.

> **💡 Cloud bağlantısı — cloud-init ve servisin ilk doğuşu:** Bulutta bir instance ilk açıldığında
> **cloud-init** çalışır (Faz 5): önyükleme sırasında paketleri kurabilir, dosyaları yerleştirebilir, ve
> senin `myapp.service`'ini kurup `enable` edebilir — hepsi elle bağlanmadan. Yani "instance'ı başlat, 30
> saniye sonra uygulaman `0.0.0.0:8000`'de canlı olsun" akışının tamamı otomatiktir. Bu, bir sonraki
> bölümün (8.4 immutable) temelidir: sunucuya elle girip uygulama kurmak yerine, kurulumu bir tarife
> (cloud-init/AMI) yazarsın ve makine kendini kurar.

> **🤔 Düşün 8.2** — Bir uygulamayı `myapp.service` olarak yazdın, `systemctl start myapp` dedin, çalıştı.
> Sonra unit dosyasında `ExecStart` satırını düzelttin ve tekrar `systemctl restart myapp` dedin ama
> değişiklik **etkili olmadı** — servis hâlâ eski komutla çalışıyor. (a) Hangi tek komutu atladın? (b) Bu
> arıza, Faz 7'deki hangi "çalışan durum vs kalıcı tanım" ayrımının (örn. `ip addr` vs netplan, `mount` vs
> fstab) bir kardeşidir?
>
> *(Cevap: fazın sonunda)*

---
---

# 8.3 Kaynaktan Derleme (Farkındalık)

## 8.3.1 `./configure && make && make install`: ne zaman gerekir `[atla]`

Bazen bir yazılım paket deposunda yoktur, ya da sana çok eski bir sürüm sunulur; o zaman **kaynaktan
derleme** gündeme gelir. Klasik üçlü:

```
$ ./configure        # sistemini incele, derleme ayarlarını hazırla
$ make               # kaynağı derle (binary üret)
$ sudo make install  # üretilen dosyaları sisteme yerleştir
```

Bunu kavram olarak bilmen yeter — bu fazda derinlemesine girmiyoruz (`[atla]`). Kritik olan **ne zaman
gerektiğini ve neden genelde gerekmediğini** ayırt etmek. Kaynaktan derleme şu bedelleri getirir: paket
yöneticisinin **dışında** kaldığı için `apt`'nin kaydında görünmez (`dpkg -L` bulmaz), otomatik
güncellenmez, ve `make install` ile diskin her yerine dağılan dosyaları temiz kaldırmak zordur. Bu yüzden
modern pratikte tercih sırası: **önce paket** (`apt`), yoksa **resmi bir repo ekle**, o da yoksa **hazır
binary** veya **konteyner**, en son çare **kaynaktan derleme**.

> **❓ Akla gelen soru: "Madem kaynaktan derleyebiliyorum, neden hep paket kullanayım?"**
>
> Çünkü paket yöneticisi sana sadece "kurulum" değil, bir **yaşam döngüsü** verir: sürüm takibi, bağımlılık
> çözümü, güvenlik güncellemeleri, temiz kaldırma, ve imzayla doğrulanmış güven. Kaynaktan derleme bunların
> hepsini elle sırtlanman anlamına gelir — bir sunucuda onlarca elle derlenmiş araç, birkaç ay sonra kimin
> hangi sürümü nereye koyduğu bilinmeyen bir çöplüğe döner. Paket yöneticisi disiplindir; kaynaktan
> derleme, sadece gerçekten gerektiğinde başvurulan bir istisnadır.

---
---

# 8.4 Immutable Yaklaşım

## 8.4.1 "Sunucuyu elle yamama, yeniden inşa et" `[kavram]`

Şimdiye kadarki her şey (paket kur, unit yaz, config düzenle) bir sunucuya **elle** dokunmak üzerineydi.
Bulut operasyonunun olgun hâli bu alışkanlığı tersine çevirir: sunucuya elle dokunmayı bir **arıza kaynağı**
olarak görür ve onun yerine **tekrarlanabilirliği** koyar. Buna **immutable (değişmez) altyapı** denir.

Fikir şudur: bir sunucuyu bir kez elle kurup sonra aylarca üstünde ayar yapmak yerine, kurulumun tamamını
bir **tarife** yazarsın — hangi paketler, hangi sürümler (pinning!), hangi unit dosyaları, hangi config —
ve bu tarifeden önceden pişmiş bir **imaj** (AMI) üretirsin. Yeni bir makine gerektiğinde bu imajdan
başlatırsın; makine daha ilk saniyeden doğru yapılandırılmış gelir. Bir şey bozulursa, o makineyi **elle
tamir etmezsin** — atarsın ve imajdan yenisini başlatırsın.

İki kavram bunu mümkün kılar ve ikisi de bu fazdan gelir:

- **Sürüm sabitleme (8.1.2).** Tarifedeki her paket sabit bir sürüme kilitliyse, aynı tariften bugün ve üç
  ay sonra **birebir aynı** makine çıkar. Pinning olmadan immutable olmaz — "aynı tarif" her seferinde
  farklı sürümler çekerse tekrarlanabilirlik yalandır.
- **Servisleştirme (8.2).** Uygulaman bir systemd unit'i olarak tarifede yazılıysa, imajdan doğan makine
  onu kendiliğinden `enable` edip başlatır — elle `start` gerekmez.

> **💡 Cloud bağlantısı — baked AMI ve "cattle, not pets":** Bu felsefenin bulut karşılığı **baked-in
> AMI**'dır: uygulaman, bağımlılıkların ve config'in önceden bir makine imajına "pişirilir". Bir Auto
> Scaling grubu bu AMI'den istediği kadar birebir aynı instance başlatır. Sektörün sloganı: sunucular
> **"pets" değil "cattle"** olmalı — isim verip elle beslediğin, hastalanınca tedavi ettiğin evcil
> hayvanlar değil; numaralı, birbirinin aynı, biri bozulunca tereddütsüz değiştirdiğin sürü hayvanları.
> "Bu sunucuda özel bir ayar var, sakın silme" cümlesi bir immutable altyapıda duyulmamalıdır — her ayar
> tarifededir, her makine atılabilir. Faz 12'de bu felsefeyi bulut mimarisiyle tam olarak birleştireceğiz.

---
---

# 8.5 Bu Faz Bozulunca — Paket ve Servisleştirme Arıza İmzaları

Bu fazın arızaları iki gruba ayrılır: **paket** (yazılım makineye giremiyor) ve **servisleştirme** (yazılım
girdi ama düzgün çalışmıyor). Aşağıdaki tablo belirtiyi doğru katmana götürür:

| Belirti | Muhtemel sebep | Bakılacak yer | İlgili bölüm |
|---|---|---|---|
| `apt install X` → "Unable to locate package" | `apt update` yapılmadı / repo yok | `apt update`, `sources.list` | 8.1.2 |
| `apt install` → "unmet dependencies" / kırık bağımlılık | Bağımlılık çakışması / yarım kalmış kurulum | `apt -f install`, `dpkg --configure -a` | 8.1.1 |
| `apt update` → "repository is not signed" | GPG anahtarı eksik/geçersiz | Repo GPG anahtarını ekle | 8.1.2 |
| Paket sürümü beklenmedik şekilde değişti | Pinning yok, `apt upgrade` yükseltti | `apt-cache policy X`, pin ekle | 8.1.2 |
| `apt install` ettim ama `systemctl status` "not found" | Paket bir systemd servisi tanımlamıyor | `dpkg -L X` (unit dosyası var mı) | 8.2.2 |
| Unit'i düzelttim ama değişiklik etkisiz | `daemon-reload` yapılmadı | `sudo systemctl daemon-reload` | 8.2.2 |
| Uygulama çökünce geri gelmiyor | `Restart=` yok / `nohup &` ile başlatılmış | Unit'e `Restart=on-failure` ekle | 8.2.1 |
| Uygulama boot'ta başlamıyor | `systemctl enable` yapılmadı | `systemctl enable X`, `is-enabled` | 8.2.2 |

> **Bu tablodan çıkan ders:** Bu fazın iki büyük ayrımı her arızanın altında yatar. Birincisi
> **`apt update` ≠ `apt upgrade`**: biri listeyi tazeler, diğeri paketleri yükseltir — karıştırmak "paketi
> bulamıyor" ve "beklenmedik sürüm" arızalarının kökenidir. İkincisi, Faz 7'den taşınan çizginin bir üst
> basamağı: **kurulu olmak ≠ servis olarak çalışıyor olmak**. Bir paketi kurmak dosyaları diske koyar; onu
> systemd'nin yönettiği, çökünce toparlanan, boot'ta başlayan bir servise çeviren şey, senin yazdığın unit
> ve `enable`'dır. Ve unutma: bir unit'i her değiştirdiğinde `daemon-reload` — bu, Faz 7'deki `mount` vs
> fstab, `ip addr` vs netplan ayrımının aynısıdır: **çalışan durum** ile **kalıcı tanım** ayrı katmanlardır.

---
---

# Faz 8 — Düşün sorularının cevapları

## Cevap 8.1 — Önce `apt update`; o, paket listesini tazeler (paketleri değil)

Önce **`sudo apt update`** çalıştırman gerekirdi. Yeni eklenen repo `sources.list.d`'ye yazıldı ama makinen
o repodaki paketlerden henüz **habersiz** — çünkü elindeki paket listesi eski. `apt update` tam olarak bunu
tazeler: yapılandırılmış tüm repoların **paket meta verisini** (hangi paket, hangi sürüm, hangi bağımlılık
mevcut) yeniden indirir. Tek bir paketi bile güncellemez/kurmaz — sadece "neler mevcut" bilgisini yeniler.
`update` sonrası `apt install yeni-arac` artık paketi bulur. Doğru sıra her zaman: **`apt update` → sonra
`install`/`upgrade`**.
**İlgili bölüm:** 8.1.2 · **Devamı:** 8.5 arıza tablosu (satır 1).

## Cevap 8.2 — `daemon-reload`'u atladın; bu, "çalışan durum vs kalıcı tanım" ailesindendir

(a) **`sudo systemctl daemon-reload`**'u atladın. Unit dosyasını diskte değiştirdin ama systemd hâlâ
belleğe önceden okuduğu **eski tanımı** kullanıyor; `restart` o eski tanımla yeniden başlattı. `daemon-reload`
systemd'ye "unit dosyalarını diskten yeniden oku" der; ondan sonra `restart` yeni `ExecStart`'ı kullanır.
Doğru sıra: dosyayı düzelt → `daemon-reload` → `restart`. (b) Bu, Faz 7'deki (ve Faz 6'daki) **çalışan durum
vs kalıcı tanım** ayrımının tam kardeşidir: systemd'nin bellekteki aktif tanımı = "çalışan durum"; diskteki
`.service` dosyası = "kalıcı tanım". Tıpkı `ip addr` (çalışan) vs netplan (kalıcı), veya `mount` (çalışan)
vs fstab (kalıcı) gibi — diski değiştirmek, çalışan durumu otomatik güncellemez; arada bir "yeniden oku"
adımı (`daemon-reload` / `mount -a` / netplan apply) vardır.
**İlgili bölüm:** 8.2.2 · **Devamı:** 8.5 arıza tablosu (satır 6).

---
---

# Faz 8 — Sık sorulan sorular

**S1 — `apt update` ile `apt upgrade` farkı nedir?** `apt update` repoların **paket listesini** tazeler
(hiçbir paketi değiştirmez); `apt upgrade` kurulu paketleri yeni sürümlerine **yükseltir**. Doğru sıra hep
`update` → sonra `install`/`upgrade` (8.1.2).

**S2 — `apt` mı `dpkg` mı kullanmalıyım?** Neredeyse her zaman `apt`. `apt` bağımlılıkları çözer ve
repolardan indirir; `dpkg` sadece tek bir `.deb` dosyasını kurar ve bağımlılık çözmez. `dpkg`'yi genelde
sadece sorgulamak için kullanırsın (`dpkg -L`, `dpkg -l`) (8.1.1).

**S3 — Bir komutun hangi paketten geldiğini nasıl bulurum?** Kuruluysa `dpkg -S $(which komut)`; henüz
kurulu değilse `apt-file search komut` (8.1.1).

**S4 — `nohup python app.py &` neden yeterli değil?** Üç eksik: çökünce geri gelmez, boot'ta başlamaz, logu
dağınıktır. systemd unit'i üçünü de çözer (`Restart=`, `enable`, journald) (8.2.1).

**S5 — Unit dosyasını değiştirdim ama etkisiz, neden?** `sudo systemctl daemon-reload` yapmadın. systemd
diskteki değişikliği otomatik görmez; her unit değişikliğinden sonra `daemon-reload` şarttır (8.2.2).

**S6 — `systemctl start` ile `enable` arasındaki fark?** `start` servisi **şimdi** başlatır (reboot'ta
gitmez geri gelmez); `enable` onu **boot'ta** başlayacak şekilde işaretler. Kalıcı bir servis için ikisi de
gerekir (8.2.2, Faz 5.3).

**S7 — Immutable altyapı ne demek, kısaca?** Sunucuyu elle yamamak yerine, kurulumu bir tarife (pinning'li
paketler + unit'ler) yazıp önceden pişmiş bir imajdan (AMI) başlatmak; bozulanı tamir etmek yerine atıp
yeniden inşa etmek. "Cattle, not pets" (8.4.1).

---
---

# Faz 8 — Kendini sına

Cevaplarını bir kâğıda yaz, sonra cevap anahtarıyla karşılaştır. Hedef: 18 sorudan 14+.

## Bölüm A — Tanım ve mekanizma

1. `apt` ile `dpkg` arasındaki katman farkını açıkla: hangisi bağımlılık çözer, hangisi tek `.deb` kurar?
2. `apt update` ile `apt upgrade` tam olarak ne yapar? Doğru sıra nedir?
3. Bir repository nedir, ve GPG imzası neyi garanti eder?
4. Sürüm sabitleme (pinning) nedir ve neden üretimde önemlidir?
5. `nohup python app.py &`'in üç üretim eksiğini say; her birini önceki bir faza bağla.
6. Bir systemd unit'inde `Restart=on-failure`, `WantedBy=multi-user.target` ve `ExecStart` satırları ne işe
   yarar?
7. `systemctl start` ile `systemctl enable` arasındaki fark nedir?
8. Immutable altyapının temel fikri nedir? "Cattle, not pets" ne anlatır?

## Bölüm B — Uygula ve teşhis et

9. `apt install aracadı` → "Unable to locate package". İlk çalıştıracağın komut nedir ve neden?
10. Bir uygulamayı `apt install` ettin ama `systemctl status uygulama` "not found" diyor. Bu ne anlama
    gelir, ve nasıl doğrularsın?
11. Unit dosyasındaki `ExecStart`'ı düzelttin, `systemctl restart` yaptın ama değişiklik etkisiz. Hangi
    komutu atladın?
12. Uygulaman gece çökmüş ve sabaha kadar kapalı kalmış. Unit'e hangi tek satırı ekleyerek bunu önlersin?
13. Makine reboot sonrası uygulaman gelmiyor. Durumu hangi komutla kontrol eder, hangi komutla düzeltirsin?
14. Bir komutun hangi paketten geldiğini ve o paketin diske koyduğu dosyaları nasıl bulursun?

## Bölüm C — Muhakeme ve bağlantı

15. "Kurulu olmak ≠ servis olarak çalışıyor olmak" ayrımını bir örnekle açıkla. Bu, Faz 7'deki hangi
    ayrımın bir üst basamağıdır?
16. `daemon-reload` gereği, Faz 6'daki `mount` vs fstab ve Faz 7'deki `ip addr` vs netplan ayrımlarıyla
    aynı aileye nasıl düşer? Ortak ilkeyi bir cümlede yaz.
17. Immutable altyapı, bu fazın hangi iki kavramına (8.1 ve 8.2'den) dayanır ve neden ikisi de şart?
18. Bir cloud-init tarifesi, bir instance ilk açıldığında bu fazın hangi adımlarını (paket + servis)
    otomatikleştirir?

---

## Cevap anahtarı

1. `apt` üst katman: bağımlılıkları çözer, repolardan indirir; `dpkg` alt katman: tek bir `.deb` kurar,
   bağımlılık çözmez (8.1.1). — 2. `apt update` repoların paket listesini tazeler (paket değiştirmez);
   `apt upgrade` kurulu paketleri yükseltir; sıra: `update` → `upgrade`/`install` (8.1.2). — 3. Repo,
   paketlerin ve meta verisinin tutulduğu sunucu; GPG imzası paketin gerçekten o repodan geldiğini ve yolda
   değiştirilmediğini garanti eder (8.1.2). — 4. Bir paketi belirli sürümde kilitlemek; üretimde beklenmedik
   sürüm değişikliklerini (ve bozulmaları) önler, tekrarlanabilirliğin temeli (8.1.2). — 5. (1) Çökünce geri
   gelmez (Faz 3/4), (2) boot'ta başlamaz (Faz 5), (3) logu dağınık (Faz 5 journald); systemd üçünü çözer
   (8.2.1). — 6. `Restart=on-failure` çökünce otomatik yeniden başlatır; `WantedBy=multi-user.target`
   (+enable) boot'ta başlatır; `ExecStart` çalıştırılacak komuttur (8.2.2). — 7. `start` şimdi başlatır
   (kalıcı değil); `enable` boot'a ekler; kalıcı servis için ikisi de gerekir (8.2.2). — 8. Sunucuyu elle
   yamama, tarifeden önceden pişmiş imajdan başlat, bozulanı at-yeniden inşa et; "cattle, not pets" =
   sunucular isimli evcil değil, atılabilir sürü (8.4.1).

9. `sudo apt update` — makinenin paket listesi eski; yeni eklenen repoyu/yeni sürümleri görmesi için önce
   listeyi tazelemek gerekir (8.1.2). — 10. Paket bir systemd servisi **tanımlamamış** (her paket
   tanımlamaz); `dpkg -L uygulama | grep '\.service'` ile unit dosyası koyup koymadığına bakarsın; yoksa
   kendi unit'ini yazarsın (8.2.2). — 11. `sudo systemctl daemon-reload` (8.2.2). — 12. `Restart=on-failure`
   (istersen `RestartSec=` ile) (8.2.1). — 13. Kontrol: `systemctl is-enabled uygulama` / `systemctl status`;
   düzeltme: `sudo systemctl enable uygulama` (8.2.2). — 14. `dpkg -S $(which komut)` (hangi paket);
   `dpkg -L paket` (koyduğu dosyalar) (8.1.1).

15. Örn.: `apt install postgresql` dosyaları diske koyar (kurulu) ama servisi kendin başlatıp `enable`
    etmeden ya da kendi uygulaman için unit yazmadan "çalışıyor" olmaz. Bu, Faz 7'deki "var olmak ≠
    erişilebilir olmak" (process dinliyor ama `0.0.0.0`'a bind değil) ayrımının bir üst basamağıdır: burada
    "diskte var olmak ≠ yönetilen bir servis olarak çalışmak" (8.5, Faz 7.4.2). — 16. Hepsinde **diskteki
    kalıcı tanımı** değiştirmek, **bellekteki/çalışan durumu** otomatik güncellemez; arada bir "yeniden oku"
    adımı gerekir: `daemon-reload` / `mount -a` / netplan apply. Ortak ilke: çalışan durum ile kalıcı config
    ayrı katmanlardır (8.5). — 17. Sürüm pinning (8.1.2) → aynı tariften birebir aynı makine çıkar; ve
    servisleştirme (8.2) → uygulama unit olarak tarifede olduğu için imajdan doğan makine onu kendiliğinden
    başlatır. Pinning olmadan tekrarlanabilirlik yalan, unit olmadan uygulama otomatik ayağa kalkmaz (8.4.1).
    — 18. Paket tarafı: `apt update` + `apt install` (pinning'li) ile paketleri kurar; servis tarafı: unit
    dosyasını yerleştirir, `daemon-reload` + `enable` + `start` ile uygulamayı ayağa kaldırır — hepsi elle
    bağlanmadan (8.2.2, 8.4.1).

## Puanlama

| Doğru sayısı | Ne anlama geliyor |
|---|---|
| 16-18 | Yazılımı makineye getirmeyi ve servisleştirmeyi kavradın. Faz 9'a hazırsın. |
| 13-15 | İyi. Kaçırdığın soruların bölümlerini tekrar oku (özellikle 8.1.2 ve 8.2.2). |
| 9-12 | Temel var ama kırılgan. `apt update`/`upgrade` ayrımı ve unit yazımı üzerine çalış. |
| 0-8 | Fazı yeniden gez; bir test makinesinde basit bir `hello.service` yazıp başlat, çökert, izle. |

Kaçırdığın soru → dönmen gereken bölüm:

| Soru | Bölüm |
|---|---|
| 1, 2, 3 | 8.1 Paket yöneticileri |
| 4, 17 | 8.1.2 Pinning + 8.4 Immutable |
| 5, 12 | 8.2.1 Neden nohup değil |
| 6, 7, 11, 13 | 8.2.2 Unit yazmak |
| 8, 18 | 8.4 Immutable / cloud-init |
| 9, 14 | 8.1.1 apt/dpkg |
| 10, 15 | Kurulu ≠ servis (8.2.2, 8.5) |
| 16 | Çalışan durum vs kalıcı tanım (8.5) |

---
---

# Faz 8 — Kapanış ve Faz 9'a Köprü

## Bu fazdan ne taşıyorsun

Faz 8 sana yazılımın **yaşam döngüsünü** verdi: makineye nasıl girer (`apt`/`dpkg`, repo, imza, pinning),
ve senin uygulaman nasıl gerçek bir servise dönüşür (systemd unit, `Restart=`, `enable`, journald). İki
kalıcı ayrım edindin: **`apt update` ≠ `apt upgrade`** (liste vs yükseltme) ve **kurulu olmak ≠ servis
olarak çalışıyor olmak**. Ve `daemon-reload` refleksiyle "çalışan durum vs kalıcı tanım" ilkesini üçüncü
kez gördün (Faz 6 fstab, Faz 7 netplan, Faz 8 unit) — artık bu bir kalıp. Son olarak immutable felsefeyi
tanıdın: elle yamama, tarifeden inşa et, "cattle, not pets".

## Faz 9 bunun neresine bağlanıyor

Faz 8 boyunca hep **daha fazla** yaptık: paket kur, servis aç, uygulamayı ayağa kaldır. Faz 9 tam tersini
sorar: **neyi kısıtlamalıyım?** Kurduğun her paket, açtığın her port, verdiğin her yetki bir **saldırı
yüzeyidir**. Faz 9 (Güvenlik ve Sertleştirme) bu yüzeyi bilinçli olarak daraltmayı öğretir: en az yetki
ilkesi, SSH/ağ sıkılaştırma (Faz 7'nin devamı), AppArmor/SELinux (DAC izinlerinin üstünde ikinci bir
kilit — "izinler doğru ama yine engelleniyor" arızası), ve sırların diskte değil IAM/Secrets Manager'da
yönetimi. Faz 8'de bir servisi `0.0.0.0`'a bind ettin; Faz 9 "peki bu servisi kim, hangi yetkiyle
çalıştırıyor, ve ele geçirilirse ne kadar zarar verebilir?" diye sorar. Kurmak ile korumak, aynı madalyonun
iki yüzüdür.

> **🤔 Faz çıktısı — kendine sor:** Faz 8'de unit dosyasına `User=appuser` yazdın — uygulamayı root
> yerine sınırlı bir kullanıcıyla çalıştırmak için. Faz 9'a geçmeden düşün: eğer bu uygulama ele geçirilirse,
> `User=root` ile çalışsaydı saldırgan ne yapabilirdi, `User=appuser` ile ne yapamaz? Bu, Faz 9'un
> "en az yetki ilkesi"nin ta kendisidir — sen aslında Faz 8'de zaten bir hardening kararı verdin.
>
> **🧪 Lab 8 fikri (kendi test instance'ında):** (1) `apt list --installed | wc -l` ile kaç paket kurulu
> gör; `apt-cache policy nginx` ile sürüm/repo bilgisini oku. (2) Basit bir betik yaz (örn. her 5 saniyede
> tarih basan bir döngü), onu bir `hello.service` unit'i olarak `/etc/systemd/system/`'e koy. (3)
> `daemon-reload` → `start` → `enable` → `status` → `journalctl -u hello -f` zincirini uygula. (4) Betiği
> kasten çökert (`exit 1`), `Restart=on-failure` ile systemd'nin onu geri getirişini `journalctl`'de izle.
> (5) `nohup ./betik &` ile aynısını başlat, oturumu kapat, geri gel — farkı bizzat gör. Bu adımlar bu
> fazın "kurulu ≠ servis" refleksini elinde toplar.

---

> **Navigasyon:** [◀ Faz 7 — Ağ ve Bağlantı](Faz_7_Ag_ve_Baglanti.md) · **Faz 8** · [Faz 9 — Güvenlik ve Sertleştirme ▶](Faz_9_Guvenlik_ve_Sertlestirme.md)

