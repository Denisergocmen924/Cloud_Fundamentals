# Ara Sınav 3 — Faz 5–6: Boot, systemd ve Depolama

> **Navigasyon:** [◀ Faz 6 — Depolama ve Dosya Sistemleri](Faz_6_Depolama_ve_Dosya_Sistemleri.md) · **Ara Sınav 3** · [Faz 7 — Ağ (OS Katmanı) ▶](Faz_7_Ag_ve_Baglanti.md)

---

## Bu sınav neyi ölçüyor?

Faz sonundaki "Kendini sına" testleri tek bir fazı yoklar. Ara sınav **farklı** bir şey ölçer: *"Bu iki
fazı birbirine bağlayabiliyor musun?"*

Faz 5 sana makinenin **nasıl var olduğunu** verdi: boot zinciri, PID 1, systemd, unit'ler, bağımlılık
sırası. Faz 6 aynı makinenin **kalıcı katmanını** verdi: blok cihaz, dosya sistemi, mount, `fstab`.
Bu ikisi tam olarak **boot anında** kesişir: `/etc/fstab`'daki her satır aslında systemd için bir
`.mount` unit'idir (Faz 5), ve o unit boot sırasının bir aşamasında (Faz 5) çalışır. Yanlış bir `fstab`
satırı (Faz 6) bu yüzden bir **boot** arızasıdır (Faz 5) — ve neden `journalctl` yerine konsola bakman
gerektiği de tam bu kesişimde yatar. "Instance açılmıyor" olayının en yaygın sebebi, iki fazın
birleştiği bu noktadadır.

**Nasıl çalışılır:**

- Kâğıt-kalemle çöz; cevabı yazmadan bir sonrakine geçme.
- Cevap anahtarı her sorunun **hangi fazların kesişiminde** durduğunu söyler — kaçırdığın soru bir fazı
  değil, iki faz arasındaki **köprüyü** işaret eder.
- 21 soru var: Bölüm A (bağlantı muhakemesi, 1–9), Bölüm B (senaryo: "instance reboot sonrası açılmıyor"
  olayı, 10–15), Bölüm C (komut ve çıktı okuma, 16–21).
- Hedef süre: ~1 saat. Ama süre önemli değil; önemli olan her cevabın *neden* öyle olduğunu bir cümleyle
  gerekçelendirebilmek.

> **🤔 Başlamadan önce:** Şu cümleyi kendi kelimelerinle tamamla: *"Bir `fstab` satırı yanlışsa makine
> açılmıyor ama `journalctl -xb` ile logları da göremiyorum — çünkü..."* Bu tek soru iki fazı da içeriyor
> (boot aşaması + storage). Cevabını bir kenara yaz; Soru 4'te göreceğiz.

---

# Bölüm A — Bağlantı muhakemesi (1–9)

Bu bölümdeki her soru en az iki fazın (çoğunlukla Faz 5 × Faz 6) bilgisini birleştirmeni ister. Kısa ama
gerekçeli cevap ver.

**1.** Faz 6'da "`fstab`'a bir satır yazmak mount'u kalıcı yapar" dedik. Faz 5'te "systemd her şeyi unit
olarak yönetir" dedik. Bir `fstab` satırı, systemd tarafında **hangi tür unit'e** dönüşür, ve bu neden
"`fstab` = mount'un `enable`'ı" (Faz 6, 6.2.2) benzetmesini teknik olarak doğru kılar?

**2.** Faz 5'te `systemctl enable`≠`start` ayrımını gördük. Faz 6'da `fstab`≠`mount` ayrımını gördük. Bu
iki ayrımın **aynı** kavramın iki yüzü olduğunu bir cümlede açıkla: "şimdi etki eden, reboot'ta giden"
vs "her boot'ta yeniden kurulan".

**3.** Faz 6'da UUID'nin cihaz adından (nvme1n1) neden daha güvenli olduğunu gördük. Faz 5'te boot
sırasının deterministik olması gerektiğini ima ettik. Cihaz adlarının boot'lar arası kayabilmesi, `fstab`
mount'unun systemd'nin bağımlılık grafiğinde (Faz 5: After/Requires) neden bir "kırılgan düğüm" olduğunu
nasıl açıklar?

**4.** Faz 5'te "PID 1'den önceki arızalar `systemctl`/`journalctl`'e görünmez, konsola bak" dedik. Bir
`fstab` hatası (Faz 6) makineyi emergency moduna düşürüyor. Bu arıza `journalctl -xb` ile **görülebilir
mi**, yoksa görülemez mi — ve cevabın neden boot zincirinin (Faz 5) hangi aşamada takıldığına bağlı?

**5.** Faz 6'da `nofail` seçeneğinin bir veri diskinde boot'u kurtardığını gördük. Faz 5'in dilinde
(unit, dependency, target) `nofail`'in ne yaptığını açıkla: `nofail`'li bir `.mount` unit'i başarısız
olunca systemd neden `local-fs.target`'ı ve dolayısıyla boot'u durdurmaz?

**6.** Faz 5'te journald'ın logları nereye yazdığını (varsayılan: bellek/`/run`, kalıcı: `/var/log/journal`)
gördük. Faz 6'da `/var`'ın ayrı bir diske mount edilebileceğini ima ettik. `/var/log/journal` ayrı bir
diske mount'luysa ve o disk boot'ta gelmezse, kalıcı loglar açısından hangi tavuk-yumurta sorunu doğar?

**7.** Faz 5'te cloud-init'in ilk boot'ta çalıştığını gördük. Faz 6'da yeni bir EBS volume'ünü `fstab`'a
eklediğini varsay. cloud-init ile `fstab`, "boot'ta bir diski hazırlamak" işini nasıl paylaşır — hangisi
**bir kez** (ilk boot), hangisi **her boot** çalışır?

**8.** Faz 5'te `systemd-analyze blame` ile boot süresini gördük. Faz 6'da bir `.mount` unit'inin
bağımlılıklarını bekleyebileceğini ima ettik. Yavaş veya asılı bir disk mount'u (Faz 6), boot süresini
(Faz 5) nasıl uzatır, ve `blame` çıktısında bunu hangi tür satır ele verir?

**9.** Faz 5'te `.service` unit'lerinin `After=`/`Requires=` ile sıralandığını gördük. Bir veritabanı
servisi verisini `/data`'da (ayrı EBS volume) tutuyor. Bu servisin unit'i, Faz 6'daki mount'a hangi
bağımlılığı (`After=`/`Requires=` + mount unit adı) tanımlamalı ki, disk mount edilmeden servis
başlamasın — ve bu tanım eksikse hangi arıza olur?

---

# Bölüm B — Senaryo: "Instance reboot sonrası açılmıyor" (10–15)

> **Olay:** Bir mühendis, `t3.large` bir Ubuntu instance'ına yeni bir 100 GB'lık EBS veri volume'ü
> attach etti. `mkfs.ext4 /dev/nvme1n1` ile biçimledi, `/data`'ya mount etti, uygulamayı taşıdı, her şey
> çalışıyordu. Kalıcı olsun diye `/etc/fstab`'a şu satırı ekledi:
> `/dev/nvme1n1  /data  ext4  defaults  0  2`
> Sonra "yeni ayarlar otursun" diye instance'ı reboot etti. Instance bir daha **SSH kabul etmedi**.
> EC2 konsolundaki "System log"da şu görünüyor:
> `[[ TIME ]] Timed out waiting for device /dev/nvme1n1.`
> `[[ TIME ]] Dependency failed for /data.`
> `You are in emergency mode. Give root password for maintenance (or press Ctrl-D to continue):`

**10.** İlk teşhis: Bu bir Faz 5 (boot) sorunu mu, Faz 6 (disk) sorunu mu, yoksa ikisinin kesişimi mi?
"Timed out waiting for device" + "Dependency failed for /data" + "emergency mode" satırlarının her birinin
hangi faza ait olduğunu eşle.

**11.** Kök sebep muhakemesi: Disk fiziksel olarak duruyor (attach hâlâ aktif). Öyleyse neden "Timed out
waiting for device /dev/nvme1n1"? Faz 6.2.3'teki cihaz-adı-kayması olasılığını kullanarak, reboot'ta bu
ismin neden bulunamamış olabileceğini açıkla. (İpucu: instance'a başka bir volume attach/detach edildi mi,
ya da NVMe numaralandırması değişti mi?)

**12.** Neden boot **durdu** (emergency mode) — sadece o mount atlanmadı? `fstab` satırındaki `pass`
alanı (`0  2`) ve eksik olan **seçenek** üzerinden açıkla. Hangi tek kelimelik seçenek eklenmiş olsaydı,
disk gelmese bile boot devam eder ve SSH açılırdı?

**13.** Kurtarma: Mühendis emergency mode'da root parolasını girdi (veya EC2 rescue akışını kullandı).
`/etc/fstab`'ı düzeltmek için hangi iki değişikliği yapmalı (Faz 6.2.3), ve düzeltmeden sonra **reboot
etmeden** hangi tek komutla (Faz 6.6.1) satırın artık boot'u kilitlemeyeceğini doğrulamalı?

**14.** `journalctl` neden yeterli değildi? Mühendis SSH'siz kaldığı için EC2 konsolunun "System log"una
bakmak zorunda kaldı. Faz 5.1 (PID 1'den önce/sonra) ve Faz 5.4 (journald) bilgisini birleştirerek: bu
özel arıza `journalctl -xb` çıktısında **görünür müydü** (systemd emergency target'a ulaştı mı), ve yine
de neden konsol daha güvenilir bir kaynaktı?

**15.** Önleme + genelleme: Bu olayın bir daha yaşanmaması için mühendisin benimsemesi gereken **tek
alışkanlık** nedir (Faz 6.6.1)? Ayrıca bu senaryoyu Faz 5'in diliyle özetle: "yanlış bir `.mount` unit'i,
`local-fs.target` bağımlılığı üzerinden `___` target'a ulaşmayı engelledi."

---

# Bölüm C — Komut ve çıktı okuma (16–21)

Aşağıdaki her çıktı parçasını oku ve sorulanı cevapla. Bu bölüm "gerçek terminalde ne görürsün"ü ölçer.

**16.** `lsblk` çıktısı:
```
NAME        SIZE TYPE MOUNTPOINTS
nvme0n1      20G disk
└─nvme0n1p1  20G part /
nvme1n1     100G disk
```
`nvme1n1` diski `df -h`'te görünmüyor. Bu (a) bozuk disk mi, (b) mount edilmemiş disk mi, (c) biçimlenmemiş
disk mi? Çıktıdan hangi iki ipucu (Faz 6.1.2) kararını veriyor, ve `df`'e sokmak için hangi adım(lar)
kaldı?

**17.** İki `df` çıktısı:
```
$ df -h /data                              $ df -i /data
Filesystem     Size Used Avail Use% ...     Filesystem      Inodes  IUsed IFree IUse% ...
/dev/nvme1n1    98G  40G   53G  44% ...     /dev/nvme1n1  6553600 6553600     0  100% ...
```
Uygulama "No space left on device" hatası veriyor. Bu iki çıktıdan hangisi arızayı açıklıyor, arızanın
adı nedir (Faz 6.4.2), ve diski büyütmek neden **çözmez**?

**18.** `systemctl status data.mount` çıktısı:
```
● data.mount - /data
     Loaded: loaded (/etc/fstab; generated)
     Active: failed (Result: timeout)
      Where: /data
       What: /dev/nvme1n1
```
`Loaded:` satırındaki `(/etc/fstab; generated)` ne anlatıyor (Faz 5 unit × Faz 6 fstab bağlantısı)? `Active:
failed (Result: timeout)` hangi Bölüm B satırıyla (Soru 10-11) örtüşüyor?

**19.** `blkid` çıktısı:
```
/dev/nvme1n1: UUID="a1b2c3d4-...-e5f6" TYPE="xfs"
```
Ama `/etc/fstab`'daki satır şöyle: `UUID=a1b2c3d4-...-e5f6  /data  ext4  defaults,nofail  0  2`. Bu satır
mount edilmeye çalışılınca ne olur (Faz 6.3.1), boot durur mu (`nofail` var), ve düzeltme için satırda
hangi tek alanı değiştirirsin?

**20.** `journalctl -u data.mount -b` çıktısı:
```
systemd[1]: Mounting /data...
systemd[1]: data.mount: Mount process exited, code=exited, status=32/n/a
systemd[1]: data.mount: Failed with result 'exit-code'.
systemd[1]: Failed to mount /data.
```
Bu logun `systemd[1]` ile başlaması (Faz 5.2.1) bize boot'un hangi aşamaya **ulaştığını** söyler? Yani bu
arıza PID 1'den önce mi sonra mı — ve bu yüzden neden `journalctl` bu kez işe yaradı (Soru 14 ile karşılaştır)?

**21.** `growpart` + `df` dizisi:
```
$ lsblk
nvme0n1  60G disk
└─nvme0n1p1 20G part /
$ df -h /
/dev/nvme0n1p1  20G  ...  /
```
EBS volume konsoldan 20G→60G büyütüldü. `lsblk` diski 60G, partition'ı 20G gösteriyor; `df` hâlâ 20G.
Hangi iki komutu hangi sırayla çalıştırırsın (Faz 6.6.2), ve her komut çıktıdaki hangi katmanı büyütür?

---

## Cevap anahtarı

**1.** Bir `fstab` satırı systemd tarafında bir **`.mount` unit'ine** dönüşür (systemd-fstab-generator
boot'ta `fstab`'ı okuyup her satır için bir `.mount` unit üretir; adı mount point'ten türetilir, örn.
`/data` → `data.mount`). Bu, `fstab`'ı teknik olarak "mount'un enable'ı" yapar: tıpkı `enable`'ın bir
servisi her boot'ta başlatması gibi, `fstab` satırı her boot'ta ilgili `.mount` unit'ini otomatik
etkinleştirir. · *Faz 5.3 × Faz 6.2.2* — **2.** İkisi de "geçici çalışan durum" ile "kalıcı, boot'ta
yeniden kurulan yapılandırma" ayrımıdır: `start`/`mount` çekirdeğin o anki durumuna etki eder ve reboot'ta
kaybolur; `enable`/`fstab` ise hiçbir şeyi hemen yapmaz ama her boot'ta otomatik yeniden kurar. · *Faz 5.3.3
× Faz 6.2.2* — **3.** Cihaz adı (`nvme1n1`) boot'lar arası kayabildiği için, `fstab` cihaz adıyla yazılmışsa
üretilen `.mount` unit'i var olmayan bir cihaza bağımlı olur; systemd o cihazı (`.device` unit'i) bekler,
zaman aşımına uğrar ve unit başarısız olur — grafikte kırılgan düğüm budur. UUID kullanmak bu bağımlılığı
diske özgü, kaymayan bir kimliğe sabitler. · *Faz 5.3.4 × Faz 6.2.3* — **4.** Görülebilir**dir** — ama
şartlı. `fstab` hatası boot'u **init aşamasında** (PID 1 çalışıyor, systemd emergency target'a düşüyor)
takar, PID 1'den **önce** değil; yani systemd ve journald zaten ayakta olduğu için `journalctl -xb` mount
başarısızlığını gösterir. PID 1'den önceki arızalarda (GRUB, initramfs) journalctl işe yaramaz; ama bu
arıza ondan sonradır. Yine de SSH açılmadığından pratikte log'a EC2 konsolundan ulaşırsın. · *Faz 5.1 ×
Faz 6.2.3* — **5.** `nofail`, üretilen `.mount` unit'inin `local-fs.target`'a **zorunlu (Requires)** değil
**isteğe bağlı (nofail → boot-critical değil)** bağlanmasını sağlar: unit başarısız olsa bile hedef "karşılandı"
sayılır, dependency zinciri kırılmaz ve boot `multi-user.target`'a devam eder. `nofail`'siz, başarısız mount
`local-fs.target`'ı çökertir ve boot durur. · *Faz 5.3.4 × Faz 6.2.3* — **6.** Kalıcı journal `/var/log/journal`
ayrı bir diske mount'luysa ve o disk boot'ta gelmezse, systemd erken boot loglarını yazacak kalıcı yeri
bulamaz — üstelik "diskin neden gelmediği"ni açıklayan logların kendisi de o gelmeyen diske yazılamaz. Bu
tavuk-yumurta, arızanın izini erken boot'ta kaybettirir; bu yüzden journal genelde köke veya `nofail`
korumalı bir yere konur. · *Faz 5.4 × Faz 6.2.2* — **7.** cloud-init **bir kez**, ilk boot'ta çalışır
(disk hazırlama, ilk biçimleme/mount, kullanıcı verisi); `fstab` ise **her boot'ta** mount'u tekrar kurar.
Bölünme: cloud-init "bir defalık kurulum/başlatma", `fstab` "kalıcı, tekrarlanan bağlama". Kalıcı bir diski
`fstab`'a yazmazsan cloud-init sonrası ilk reboot'ta mount kaybolur. · *Faz 5.6 × Faz 6.2.2* — **8.** Asılı
bir mount, bağlı olduğu `.device`/`.mount` unit'i zaman aşımına uğrayana kadar `local-fs.target`'ı bekletir;
bu bekleme boot süresine doğrudan eklenir. `systemd-analyze blame` çıktısında yüksek süreli bir `*.mount`
(veya `*.device`) satırı bunu ele verir. · *Faz 5.3 × Faz 6.2.1* — **9.** Servis unit'i mount unit'ine
`After=data.mount` **ve** `Requires=data.mount` (veya `RequiresMountsFor=/data`) tanımlamalı; böylece disk
mount edilmeden servis başlamaz. Bu eksikse servis boş bir `/data` dizinine (mount'tan önce) başlar, veriyi
kök diske yazar veya "veri yok" hatası verir — mount sonradan gelince veri "kaybolmuş" görünür. · *Faz 5.3.4
× Faz 6.2.2*

**10.** İkisinin **kesişimi**. "Timed out waiting for device /dev/nvme1n1" → Faz 6 (cihaz adı/blok cihaz).
"Dependency failed for /data" → Faz 5 (unit bağımlılık grafiği, `.mount` başarısız). "emergency mode" → Faz
5 (systemd boot durdu, hedefe ulaşılamadı). · *Faz 5.3.4 × Faz 6.2.3* — **11.** `fstab`'a **cihaz adıyla**
(`/dev/nvme1n1`) yazıldı; NVMe cihaz adları boot'lar arası ve attach sırasına göre kayabilir. Reboot'ta (veya
araya başka bir volume girdiğinde) çekirdek o diske farklı bir ad (örn. `nvme2n1`) verdi; `/dev/nvme1n1`
artık ya yok ya başka bir şey. systemd o cihaz unit'ini bekledi ve zaman aşımına uğradı. UUID kullanılsaydı
ad kayması etkisiz olurdu. · *Faz 6.2.3* — **12.** Satırda `nofail` **yok** ve mount boot-kritik `local-fs.target`'a
zorunlu bağlı olduğundan, `.mount` başarısız olunca systemd hedefi karşılanamamış sayıp emergency mode'a
düştü — sadece o mount'u atlamak yerine. Eklenmiş olsaydı **`nofail`**, disk gelmese de boot devam eder,
SSH açılırdı. (`pass 2` ayrıca boot'ta fsck denemesini işaret eder ama asıl felaketi getiren eksik `nofail`'dir.)
· *Faz 6.2.3* — **13.** İki değişiklik: (1) cihaz adını **UUID** ile değiştir (`blkid` ile al); (2)
seçeneklere **`nofail`** ekle. Düzeltmeden sonra reboot etmeden `sudo mount -a` çalıştır — hata vermezse satır
artık boot'u kilitlemez (`findmnt --verify` ek güvence). · *Faz 6.2.3 × Faz 6.6.1* — **14.** Arıza PID 1'den
**sonra** (systemd emergency target'a düştü) olduğundan `journalctl -xb` mount başarısızlığını **gösterirdi**;
ama makine SSH kabul etmediğinden mühendis o loglara giremedi. EC2 "System log" (seri konsol) ise SSH'den
bağımsızdır ve emergency mode dâhil her aşamayı gösterir — bu yüzden erişilemez bir makinede konsol daha
güvenilir kaynaktır. · *Faz 5.1 × Faz 5.4* — **15.** Tek alışkanlık: **her `fstab` düzenlemesinden sonra,
reboot etmeden önce `sudo mount -a`** ile doğrulamak. Faz 5 diliyle: "yanlış bir `.mount` unit'i,
`local-fs.target` bağımlılığı üzerinden **`multi-user.target`**'a (SSH/servislerin olduğu hedef) ulaşmayı
engelledi." · *Faz 5.3.4 × Faz 6.6.1*

**16.** (b) **mount edilmemiş** disk — bozuk değil. İki ipucu: `nvme1n1` `TYPE disk` olarak görünüyor
(çekirdek tanıyor, sağlam) ve `MOUNTPOINTS` boş (mount yok). Biçimli mi belirsiz; `df`'e sokmak için gerekirse
`mkfs`, sonra kesinlikle `mount` (ve kalıcılık için `fstab`). · *Faz 6.1.2* — **17.** İkinci çıktı (`df -i`)
açıklıyor: `IUse% 100%` — **inode tükenmesi**. `df -h` %44 boş göstermesine rağmen yeni dosya açılamaz. Diski
büyütmek **veri bloğu** bütçesini artırır ama inode bütçesi ayrıdır; çözüm küçük dosyaları temizlemektir. ·
*Faz 6.4.2* — **18.** `(/etc/fstab; generated)` bu `.mount` unit'inin elle yazılmadığını, **`fstab`'dan
otomatik üretildiğini** söyler (Soru 1'deki generator) — Faz 5 unit'i ile Faz 6 `fstab`'ının tam bağlantısı.
`Active: failed (Result: timeout)` Soru 10-11'deki "Timed out waiting for device" ile örtüşür: cihaz gelmedi,
unit zaman aşımına uğradı. · *Faz 5.3 × Faz 6.2.3* — **19.** Diskin gerçek formatı **xfs**, ama `fstab` tipi
`ext4` diyor; mount "wrong fs type / unknown filesystem" ile başarısız olur. `nofail` olduğundan boot
**durmaz**, sadece `/data` mount edilmez. Düzeltme: satırdaki **tip** alanını `ext4` → `xfs` yap. · *Faz 6.3.1*
— **20.** Logun `systemd[1]` ile başlaması, **PID 1'in (systemd) çalıştığını**, yani boot'un init aşamasına
**ulaştığını** gösterir (Faz 5.2.1). Demek arıza PID 1'den **sonra**; bu yüzden bu kez journald ayaktaydı ve
`journalctl` mount hatasını kaydedebildi — Soru 14'teki "PID 1'den önce olsaydı görünmezdi" durumunun tersi.
· *Faz 5.2.1 × Faz 5.4* — **21.** Önce `sudo growpart /dev/nvme0n1 1` (partition'ı 20G→60G'ye uzatır — `lsblk`
partition satırını büyütür), sonra `sudo resize2fs /dev/nvme0n1p1` (ext4 dosya sistemini büyütür — `df`
çıktısını 60G yapar). Sıra: alt katman (partition) önce, üst katman (dosya sistemi) sonra. · *Faz 6.6.2*

---

## Puanlama

| Doğru sayısı | Ne anlama geliyor |
|---|---|
| 18-21 | Boot ile depolamayı boot anında birleştiren köprüyü kurdun. Faz 7'ye hazırsın. |
| 14-17 | İyi. Kaçırdığın soruların işaret ettiği **köprü** bölümlerini (aşağıdaki tablo) tekrar oku. |
| 9-13 | Fazları tek tek biliyorsun ama kesişimde zorlanıyorsun. Bölüm B senaryosunu baştan çöz. |
| 0-8 | Faz 5 ve Faz 6'yı ayrı ayrı bir daha gez; özellikle 5.3 (unit/target) ve 6.2.3 (fstab). |

Kaçırdığın soru → dönmen gereken köprü:

| Soru | Köprü (bölüm × bölüm) |
|---|---|
| 1, 2 | `fstab` → `.mount` unit'i; enable≠start / fstab≠mount (5.3 × 6.2.2) |
| 3, 11, 19 | UUID vs cihaz adı / dosya sistemi tipi (5.3.4 × 6.2.3, 6.3.1) |
| 4, 14, 20 | Boot aşaması ve journalctl'in görünürlüğü (5.1/5.2/5.4 × 6.2.3) |
| 5, 12, 15 | `nofail` ve `local-fs.target` bağımlılığı (5.3.4 × 6.2.3) |
| 6, 7, 8 | journald/cloud-init/boot süresi × mount (5.4/5.6 × 6.2) |
| 9 | Servis → mount bağımlılığı (5.3.4 × 6.2.2) |
| 10, 13 | Senaryo teşhisi ve kurtarma (5.3.4 × 6.6.1) |
| 16, 17, 21 | lsblk/df -i/growpart okuma (6.1.2, 6.4.2, 6.6.2) |
| 18 | `.mount` unit'i × fstab generator (5.3 × 6.2.3) |

---

## Kapanış

Bu sınav, kariyerinde defalarca karşılaşacağın en pahalı Linux olayının — "instance açılmıyor" — kalbindeki
kesişimi test etti: bir diskin (Faz 6) yanlış tanımlanması, bir boot'un (Faz 5) durmasına yol açar. İki
dersi birlikte taşı: (1) kalıcılık her zaman "bir yerde bir yapılandırma her boot'ta yeniden kuruluyor"
demektir — ister `systemctl enable` ister `fstab` olsun, o yapılandırma yanlışsa boot'u kilitler; (2) bir
makine erişilemez olduğunda, gözün ilk gideceği yer `journalctl` değil **konsoldur** — çünkü arıza boot
zincirinin neresinde olduğuna göre journalctl'in kendisi hiç ayağa kalkmamış olabilir.

Faz 7 seni makinenin dışına, ağa taşıyor. Orada da aynı "var olmak ≠ erişilebilir olmak" boşluğunu
göreceksin — bu kez disk yerine port ve güvenlik grubu üzerinden.

---

> **Navigasyon:** [◀ Faz 6 — Depolama ve Dosya Sistemleri](Faz_6_Depolama_ve_Dosya_Sistemleri.md) · **Ara Sınav 3** · [Faz 7 — Ağ (OS Katmanı) ▶](Faz_7_Ag_ve_Baglanti.md)
