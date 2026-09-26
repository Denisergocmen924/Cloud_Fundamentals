# Ara Sınav 4 — Faz 7–9: Ağ, Servisleştirme ve Güvenlik

> **Navigasyon:** [◀ Faz 9 — Güvenlik ve Sertleştirme](Faz_9_Guvenlik_ve_Sertlestirme.md) · **Ara Sınav 4** · [Faz 10 — Otomasyon ve Scripting ▶](Faz_10_Otomasyon_ve_Scripting.md)

---

## Bu sınav neyi ölçüyor?

Önceki ara sınavlar iki fazı birleştiriyordu. Bu sınav **üç** fazı birleştirir — çünkü Faz 7, 8 ve 9 aslında
tek bir hikâyenin üç yüzüdür: **sunucunun dış dünyayla ilişkisi ve savunması.**

Faz 7 servisi ağa **açtı** (erişilebilirlik: bind adresi, port, güvenlik grubu, firewall). Faz 8 servisi
makineye **getirdi ve yönetilebilir kıldı** (kurulum, systemd unit, `User=`, auto-restart). Faz 9 tüm bunları
bilinçli olarak **sınırladı** (en az yetki, MAC, sırlar, patlama yarıçapı). Üçü tek bir olayda kesişir:
"servise bağlanamıyorum" (Faz 7) çoğu zaman bir güvenlik kararının (Faz 9: SG/AppArmor) sonucudur; `nohup`
yerine unit yazmak (Faz 8) hem operasyonel hem güvenlik (`User=`, Faz 9) bir karardır; ve aynı `ss -tulpn`
komutu üç faza üç farklı gözle bakar. Bu sınav, bu üç fazı **tek bir refleks** olarak görüp göremediğini
ölçer.

**Nasıl çalışılır:**

- Kâğıt-kalemle çöz; cevabı yazmadan bir sonrakine geçme.
- Cevap anahtarı her sorunun **hangi fazların kesişiminde** durduğunu söyler — kaçırdığın soru bir fazı değil,
  fazlar arası **köprüyü** işaret eder.
- 21 soru var: Bölüm A (bağlantı muhakemesi, 1–9), Bölüm B (senaryo: "servis dışarıdan erişilemiyor, sonra
  ele geçiriliyor", 10–15), Bölüm C (komut ve çıktı okuma, 16–21).
- Hedef süre: ~1 saat. Ama süre önemli değil; önemli olan her cevabın *neden* öyle olduğunu bir cümleyle
  gerekçelendirebilmek.

> **🤔 Başlamadan önce:** Şu cümleyi kendi kelimelerinle tamamla: *"`systemctl status` yeşil ama servise
> dışarıdan hâlâ bağlanamıyorum — çünkü `systemctl` yalnızca ___ kapısını kontrol eder, geri kalan kapıları
> değil."* Bu tek cümle Faz 7 (kapılar) ile Faz 8'i (servis ayakta) birleştiriyor. Cevabını bir kenara yaz;
> Soru 2 ve Bölüm B'de göreceğiz.

---

# Bölüm A — Bağlantı muhakemesi (1–9)

Her soru en az iki fazın bilgisini birleştirmeni ister. Kısa ama gerekçeli cevap ver.

**1.** Faz 8'de bir servisi `nohup python app.py &` yerine bir systemd unit'i olarak yazmayı öğrendin. Faz
9'da `User=appuser` satırının bir güvenlik kararı olduğunu gördün. `nohup` ile çalıştırmanın, Faz 9'un en az
yetki ilkesini de neden ihlal ettiğini açıkla: `nohup` süreç hangi kimlikle/yetkiyle koşar, ve bu unit'e göre
patlama yarıçapını nasıl büyütür?

**2.** Faz 7'de "bir servise beş kapının hepsi açık olmalı" dedik (DNS, SG, host firewall, bind adresi,
süreç). Faz 8'de `systemctl status`'ın yeşil olmasının servisin ayakta olduğunu gösterdiğini gördük.
`systemctl status` bu beş kapıdan **hangisini** doğrular, ve neden "yeşil ama erişilemiyor" durumu Faz 7 ile
Faz 8'in tam kesişimidir?

**3.** Faz 7'de bir servisin `0.0.0.0` yerine `127.0.0.1`'e bind edilmesinin onu dışarıya kapattığını gördük.
Faz 9'da en az yetki/yüzey daraltmayı gördük. Bir veritabanını yalnızca yerel uygulamanın kullandığı bir
sunucuda, DB'yi `127.0.0.1`'e bind etmek Faz 9'un hangi ilkesinin somut bir uygulamasıdır, ve bu SG'de portu
kapatmaktan farklı olarak hangi ek savunma katmanını (defense in depth) sağlar?

**4.** Faz 7'de Security Group'un "işletim sistemine görünmez" olduğunu (SG paketi OS'a ulaşmadan düşürür)
gördük. Faz 9'da katmanlı savunmayı gördük. Bir saldırgan host firewall'u (ufw) bir şekilde aşsa bile SG hâlâ
neden koruyabilir — bu iki firewall'un **bağımsız katmanlar** olması defense in depth'i nasıl örnekler?

**5.** Faz 8'de `daemon-reload`'un unit dosyasındaki değişikliği systemd'ye okuttuğunu (çalışan durum vs kalıcı
tanım) gördük. Faz 9'da bir servisin `User=` veya `AmbientCapabilities=` satırını değiştirdiğini varsay. Bu
güvenlik değişikliğinin **etki etmesi** için hangi iki adımı (Faz 8) sırayla yapman gerekir, ve sadece dosyayı
düzenleyip bırakırsan ne olur?

**6.** Faz 7'de SSH'ı key-only + root kapalı yaptık. Faz 9'da bunu bir "erişim katmanı" olarak konumlandırdık.
Faz 9'daki IAM instance role fikri (sırrı diske yazmadan kimlikle erişim), Faz 7.3'teki hangi SSH fikrinin
(anahtar dağıtma yerine SSM/kimlik) veri erişimi versiyonudur — ortak prensibi tek cümlede yaz.

**7.** Faz 8'de immutable/baked AMI'yi (yazılımı imaja pişir) gördük. Faz 9'da CIS-hardened AMI'yi gördük. Bu
iki fikir "her makinede elle X yapmak" yerine "X'i imaja bir kez koymak" biçiminde nasıl birleşir — ve
"güvenlik bir kurulum adımı değil, imajın bir özelliğidir" cümlesini bu birleşimle açıkla.

**8.** Faz 7'de `ss -tulpn` ile "servis doğru adreste dinliyor mu" diye baktık. Faz 9'da aynı komutla "kaç
gereksiz port dışa açık" diye baktık. Tek bir `ss -tulpn` çıktısına bakan bir cloud engineer'ın **aynı anda**
sorması gereken üç soruyu (Faz 7 erişilebilirlik, Faz 8 operasyon, Faz 9 saldırı yüzeyi) yaz.

**9.** Faz 9'da "izinler doğru ama Permission denied" arızasının MAC (AppArmor/SELinux) katmanında olduğunu
gördük. Bir servis bir porta bağlanamıyor: `User=appuser` (root değil) ve port 80. Bu arıza bir **MAC**
sorunu mu, yoksa bir **capability** (Faz 9.5) sorunu mu — ikisini nasıl ayırt edersin, ve doğru çözüm nedir?

---

# Bölüm B — Senaryo: "Servis dışarıdan erişilemiyor, sonra ele geçiriliyor" (10–15)

> **Olay:** Bir mühendis, `t3.medium` bir Ubuntu instance'ında bir FastAPI uygulamasını 8000 portunda
> çalıştırdı. `curl localhost:8000` makinenin **içinden** çalışıyor, ama tarayıcıdan (instance'ın public IP'si)
> **erişilemiyor**. Mühendis "hızlı çözeyim" diye şunları yaptı: (1) uygulamayı `sudo nohup python app.py &`
> ile **root** olarak yeniden başlattı; (2) Security Group'a `0.0.0.0/0` için tüm portları açan bir kural
> ekledi; (3) uygulamanın bind adresini kontrol etmeden bıraktı. Sonra erişim çalıştı, mühendis konuyu kapattı.
> **Üç hafta sonra:** uygulamada bilinen bir açık üzerinden sunucu ele geçirildi; saldırgan makinede root oldu,
> başka servislerin verilerini okudu ve makineyi bir botnet'e kattı. Log'da (sonradan) `app.py`'ın
> `0.0.0.0:8000` yerine önce `127.0.0.1:8000`'e bind ettiği, ama mühendisin `--host 0.0.0.0` eklediği görülüyor.

**10.** İlk teşhis (erişilemezlik): `curl localhost:8000` çalışıyor ama dışarıdan erişilemiyor. Ele geçirmeden
**önceki** bu ilk arıza, Faz 7'nin beş kapısından hangi **ikisiyle** açıklanabilir (bind adresi ve SG)? Her
birini bir cümleyle bağla.

**11.** Mühendisin (1). "çözümü" (`sudo nohup python app.py &`, root) hangi **iki** fazın dersini birden ihlal
etti? (Faz 8: neden `nohup` production değil; Faz 9: neden root felaket.) İkisini ayrı ayrı yaz.

**12.** Mühendisin (2). "çözümü" (SG'de `0.0.0.0/0` tüm portlar) neden erişilebilirliği çözerken saldırı
yüzeyini felaket boyutta büyüttü? Faz 7 (SG'nin işlevi) ve Faz 9 (yüzey daraltma) bilgisini birleştirerek,
"doğru" çözümün ne olması gerektiğini yaz (hangi tek portu, hangi kaynağa?).

**13.** Kök sebep muhakemesi (gerçek arıza neydi?): Log'a göre app aslında `127.0.0.1`'e bind ediyordu.
Mühendisin `--host 0.0.0.0` eklemesi doğru düzeltmeydi — ama o **bunu bilmeden** SG'yi ardına kadar açtı.
Bind adresi düzeltilince SG'yi neden yine de sadece 8000/gerekli kaynağa daraltmak gerekirdi — "erişim çalıştı"
neden yanıltıcı bir başarı sinyalidir?

**14.** Patlama yarıçapı: Uygulama `User=appuser` ile (root değil) çalışıyor olsaydı, ve bir AppArmor profili
olsaydı, ele geçirme sonrası hasar nasıl **farklı** olurdu? Faz 9'un iki katmanını (en az yetki + MAC) tek tek
saldırganın neyini engelleyeceğiyle eşle.

**15.** Önleme + genelleme: Bu olayın bir daha yaşanmaması için mühendisin benimsemesi gereken **üç alışkanlık**
nedir (her biri bir faza bağlı: Faz 7 teşhis yönü, Faz 8 servisleştirme, Faz 9 en az yetki)? Ayrıca olayı tek
cümlede özetle: "erişilebilirliği yanlış katmanda (___) çözmek, güvenliği (___) feda etti."

---

# Bölüm C — Komut ve çıktı okuma (16–21)

Aşağıdaki her çıktı parçasını oku ve sorulanı cevapla.

**16.** `ss -tulpn` çıktısı:
```
Netid State  Local Address:Port   Process
tcp   LISTEN 127.0.0.1:8000       python
tcp   LISTEN 0.0.0.0:22           sshd
```
Uygulamaya dışarıdan erişilemiyor ama SSH çalışıyor. Çıktıdaki hangi tek fark (Faz 7.4.2) arızayı açıklıyor,
ve app'i erişilebilir yapmak için bind adresini neye çevirmelisin?

**17.** `systemctl status myapp` çıktısı:
```
● myapp.service - My FastAPI app
     Loaded: loaded (/etc/systemd/system/myapp.service; enabled)
     Active: active (running) since ...
       Main PID: 1234 (python)
```
`Active: active (running)` **yeşil**, ama kullanıcı "siteye erişemiyorum" diyor. `systemctl status`'ın yeşil
olması hangi kapıyı (Faz 7) doğrular, hangilerini **doğrulamaz** — bir sonraki bakılacak komut nedir?

**18.** `sudo ufw status verbose` çıktısı:
```
Status: active
Default: deny (incoming), allow (outgoing)
To          Action  From
22/tcp      ALLOW   Anywhere
```
Uygulama 8000 portunda ama listede yok. Host firewall açısından ne oluyor (Faz 7.4.4), ve bu SG'den ayrı bir
katman olduğu için (Faz 9) neden **her ikisini de** kontrol etmen gerekir?

**19.** `sudo journalctl -k | grep apparmor` çıktısı:
```
apparmor="DENIED" operation="open" profile="myapp" name="/var/www/data/x.txt" ...
```
Servisin dosya izinleri (`ls -l`) kusursuz ama dosyayı okuyamıyor. Bu çıktı (Faz 9.3.2) arızanın hangi
katmanda olduğunu kanıtlıyor, ve doğru çözüm neden profili **kapatmak** değildir?

**20.** `getcap` + `ss` dizisi:
```
$ getcap /opt/myapp/server
(boş çıktı)
$ sudo -u appuser /opt/myapp/server --port 80
Error: Permission denied (bind :80)
```
Servis root değil (`appuser`) ve 80 portuna bağlanamıyor. Bu bir izin/MAC sorunu değil; hangi eksik (Faz 9.5)
buna yol açıyor, ve tüm servisi root yapmadan bunu hangi tek komutla çözersin?

**21.** İki durum karşılaştırması:
```
A:  User=root       (unit'te)      →  ele geçirilince: saldırgan tüm makinede root
B:  User=appuser    (unit'te)      →  ele geçirilince: saldırgan sadece appuser
    + AppArmor profile "myapp"                       + profilin izin verdiği yollar
```
Aynı uygulama açığı ikisinde de sömürülüyor. Faz 9'un diliyle: hangi kavram A ile B arasındaki farkı
adlandırır, ve neden "açığı önlemek" değil "hasarı sınırlamak" güvenliğin asıl işidir?

---

## Cevap anahtarı

**1.** `nohup python app.py &` süreci onu başlatan kullanıcının kimliğiyle koşar; genellikle "hızlı olsun" diye
`sudo` ile, yani **root** çalıştırılır. Root çalışan bir süreç ele geçirilirse saldırgan anında tüm makineyi
alır — patlama yarıçapı maksimumdur. systemd unit'i ise `User=appuser` ile süreci sınırlı bir kimliğe hapseder;
`nohup` bu kontrolü hiç sunmaz, dolayısıyla en az yetki ilkesini yapısal olarak ihlal eder. · *Faz 8.2 × Faz
9.1* — **2.** `systemctl status` yalnızca **beşinci kapıyı** (süreç ayakta mı, doğru çalışıyor mu) doğrular;
DNS, SG, host firewall ve bind adresini **hiç görmez**. Bu yüzden "yeşil ama erişilemiyor" tam kesişimdir:
servis (Faz 8) sağlam ama önündeki dört ağ kapısından (Faz 7) biri kapalıdır. · *Faz 7.4 × Faz 8.2* — **3.**
DB'yi `127.0.0.1`'e bind etmek **en az yetki / yüzey daraltmanın** somut uygulamasıdır — servis yalnızca gereken
kadar erişilebilir. SG'de portu kapatmaktan farkı: bu ayrı, bağımsız bir katmandır (defense in depth). SG
yanlışlıkla açılsa bile DB dış arayüzü hiç dinlemediği için erişilemez kalır — iki bağımsız kilit. · *Faz 7.4.2
× Faz 9.1/9.2* — **4.** SG ve host firewall bağımsız katmanlardır: SG bulut ağ seviyesinde (OS'a ulaşmadan),
ufw OS seviyesinde çalışır. Saldırgan ufw'yi (ör. bir yanlış yapılandırmayla) aşsa bile SG paketi daha OS'a
gelmeden düşürebilir; tersi de geçerli. Biri aşılınca diğeri hâlâ koruduğu için bu tam bir defense in depth
örneğidir. · *Faz 7.4 × Faz 9.1.2* — **5.** İki adım: (1) `sudo systemctl daemon-reload` (systemd'nin değişen
unit tanımını okuması), (2) `sudo systemctl restart myapp` (yeni tanımla süreci yeniden başlatması). Sadece
dosyayı düzenleyip bırakırsan çalışan süreç **eski** `User=`/capability ile koşmaya devam eder — güvenlik
değişikliği "kalıcı tanımda" vardır ama "çalışan durumda" yoktur. · *Faz 8.2.2 × Faz 9* — **6.** Faz 7.3'teki
"SSH anahtarını makinelere dağıtma; SSM/kimlik ile eriş" fikrinin veri erişimi versiyonudur. Ortak prensip:
**kalıcı bir sır (anahtar) dağıtmak yerine bir kimliği doğrulamak** — dağıtılan/saklanan kalıcı sır olmayınca
sızacak sır da olmaz. · *Faz 7.3.4 × Faz 9.6.2* — **7.** İkisi de "her makinede elle X yapmak" yerine "X'i
imaja bir kez pişirmek"tir: baked AMI yazılımı, hardened AMI güvenliği (profiller, kapalı servisler, sıkı SSH)
imaja koyar. Birleşince her yeni instance güvenli **doğar**, kimse elle sertleştirmez → "güvenlik bir kurulum
adımı değil, imajın bir özelliği"dir; immutable felsefe güvenliği tekrarlanabilir kılar. · *Faz 8.4 × Faz 9.3.2*
— **8.** (i) *Erişilebilirlik (Faz 7):* servis doğru adreste mi dinliyor (`0.0.0.0` vs `127.0.0.1`)? (ii)
*Operasyon (Faz 8):* beklediğim servis(ler) gerçekten ayakta ve doğru portta mı? (iii) *Saldırı yüzeyi (Faz 9):*
dışa dinleyen bu portlardan kaçı **gereksiz** ve kapatılmalı? · *Faz 7.4 × Faz 8.2 × Faz 9.2* — **9.** Bu bir
**capability** sorunudur, MAC değil. Ayırt etme: 80 gibi 1024-altı bir porta root-olmayan bir sürecin bağlanamaması
klasik ayrıcalıklı-port sorunudur; MAC olsaydı `journalctl -k | grep apparmor`'da bir DENIED satırı görürdün, o
yok. Çözüm: tüm servisi root yapmadan `CAP_NET_BIND_SERVICE` ver (`setcap` veya unit'te `AmbientCapabilities=`).
· *Faz 9.5 × Faz 9.3*

**10.** İki kapı: (i) **bind adresi** — app `127.0.0.1`'e bind ediyorsa yalnızca makinenin içinden erişilir
(`curl localhost` çalışır), dışarıdan asla; (ii) **Security Group** — 8000 portu SG'de dışarıya açık değilse
paket OS'a hiç ulaşmaz. İkisinden biri bile kapalıysa dışarıdan erişilemez. · *Faz 7.4.2 × Faz 7.4.3* — **11.**
(Faz 8) `nohup` production değildir: çökerse yeniden başlamaz, reboot'ta gitmez, logları dağınık, oturum
kapanınca ölebilir — yönetilen bir servis değil, başıboş bir süreçtir. (Faz 9) root çalıştırmak patlama
yarıçapını maksimuma çıkarır: tek bir açık = tüm makine. Mühendis tek bir "hızlı" adımda hem operasyonu hem
güvenliği feda etti. · *Faz 8.2.1 × Faz 9.1.1* — **12.** `0.0.0.0/0` için **tüm portları** açmak, SG'nin tek
işlevini (yüzeyi daraltmak) tersine çevirir: artık makinedeki her dinleyen port (SSH, DB, iç servisler) tüm
internete açıktır — devasa bir saldırı yüzeyi. Doğru çözüm: yalnızca **8000 portunu**, ve mümkünse yalnızca
gereken kaynağa (ör. bir load balancer/CloudFront ya da ofis IP'si), açmaktı — beyaz liste, kara liste değil.
· *Faz 7.4.3 × Faz 9.2.1* — **13.** Çünkü asıl arıza bind adresiydi (`127.0.0.1`); `--host 0.0.0.0` eklenince app
zaten dışarıya dinlemeye başlar ve **dar** bir SG (sadece 8000) ile erişim çalışırdı. SG'yi ardına kadar açmak
gereksiz ve tehlikeliydi. "Erişim çalıştı" yanıltıcıdır çünkü **birden çok değişikliği aynı anda** yaptı;
hangisinin gerçekten gerektiğini test etmedi — yüzeyi daraltma disiplini (Faz 9) tam da "işe yarayan en dar
yapılandırmayı bul"maktır. · *Faz 7.4.2 × Faz 9.2.1* — **14.** (en az yetki) `User=appuser` olsaydı saldırgan
root değil yalnızca `appuser` olurdu: sistem dosyalarını değiştiremez, başka kullanıcıların/servislerin
verisini okuyamaz, kalıcılık kurması zorlaşırdı. (MAC) AppArmor profili o sürecin erişebileceği dosya/işlem/ağ
kümesini zaten sınırladığından, saldırgan profilin dışına çıkamaz (ör. `/etc/shadow` okuma, keyfi ağ bağlantısı
engellenir). İki katman birlikte hasarı "tüm makine"den "tek sürecin dar kutusu"na indirir. · *Faz 9.1.1 × Faz
9.3* — **15.** Üç alışkanlık: (i) **Faz 7 — teşhisi dıştan içe yap:** erişilemezlikte önce bind adresi + SG'yi
kontrol et, kör kör SG açma. (ii) **Faz 8 — `nohup` değil unit yaz:** servisi `User=appuser` ile yönetilen bir
systemd unit'i yap. (iii) **Faz 9 — en az yetki:** root çalıştırma, SG'yi en dar kaynağa aç, gereksiz portları
kapat. Tek cümle: "erişilebilirliği yanlış katmanda (**SG'yi ardına kadar açarak**) çözmek, güvenliği (**patlama
yarıçapını maksimuma çıkararak / root + açık yüzey**) feda etti." · *Faz 7.4 × Faz 8.2 × Faz 9.1*

**16.** Tek fark **bind adresi**: app `127.0.0.1:8000`'e (yalnız yerel), sshd ise `0.0.0.0:22`'ye (tüm
arayüzler) dinliyor. Bu yüzden SSH dışarıdan çalışır, app çalışmaz. Çözüm: app'in bind adresini `0.0.0.0:8000`
yap (uygulama `--host 0.0.0.0` veya config). · *Faz 7.4.2* — **17.** `systemctl status`'ın yeşil olması yalnızca
**süreç kapısını** (Faz 7'nin 5. kapısı: servis ayakta ve çalışıyor) doğrular; DNS, SG, host firewall ve bind
adresini doğrulamaz. Bir sonraki komut: `sudo ss -tulpn` — servisin hangi adres:port'ta dinlediğini görmek
(bind adresi kapısı), ardından SG ve ufw kontrolü. · *Faz 8.2 × Faz 7.4* — **18.** Host firewall varsayılan
`deny (incoming)` ve listede yalnızca 22 var; 8000 **açık değil**, yani ufw gelen 8000 trafiğini düşürür. SG
ayrı, bulut seviyesi bir katman olduğundan (Faz 9 defense in depth), 8000 hem SG'de hem ufw'de açık olmalıdır —
biri bile kapalıysa erişilemez. Bu yüzden "erişilemiyor"da **her iki** firewall'u da kontrol et. · *Faz 7.4.4 ×
Faz 9.1.2* — **19.** `apparmor="DENIED"` satırı arızanın **MAC katmanında** (AppArmor) olduğunu kanıtlar — DAC
izinleri (`ls -l`) doğru olsa bile profil bu yola erişimi reddediyor. Doğru çözüm profili kapatmak (`aa-disable`)
değildir, çünkü o zaman o servisin ikinci kilidini tümüyle kaybedersin; doğrusu profile `/var/www/data/` yolunu
izinli eklemektir (en az yetki: sadece gereken yolu aç). · *Faz 9.3.2* — **20.** Eksik olan bir **capability**:
80 gibi ayrıcalıklı bir porta bağlanmak `CAP_NET_BIND_SERVICE` ister; root-olmayan `appuser` bu yetki olmadan
bind edemez. Tüm servisi root yapmadan çözüm: `sudo setcap 'cap_net_bind_service=+ep' /opt/myapp/server` (veya
unit'te `AmbientCapabilities=CAP_NET_BIND_SERVICE`). · *Faz 9.5.1* — **21.** Farkı adlandıran kavram **patlama
yarıçapı** (blast radius): aynı açık A'da tüm makineye (root) yayılırken B'de tek sürecin dar kutusuna
(`appuser` + AppArmor) hapsolur. Güvenliğin asıl işi "hasarı sınırlamak"tır çünkü hiçbir sistem sonsuza dek
açıksız kalmaz — er ya da geç bir açık sömürülür; katmanlı savunma + en az yetki, o an geldiğinde kaybın ne
kadar yayılacağını belirler. · *Faz 9.1.1 × Faz 9.3*

---

## Puanlama

| Doğru sayısı | Ne anlama geliyor |
|---|---|
| 18-21 | Ağ, servisleştirme ve güvenliği tek bir refleks olarak birleştirdin. Faz 10'a hazırsın. |
| 14-17 | İyi. Kaçırdığın soruların işaret ettiği **köprü** bölümlerini (aşağıdaki tablo) tekrar oku. |
| 9-13 | Fazları tek tek biliyorsun ama kesişimde zorlanıyorsun. Bölüm B senaryosunu baştan çöz. |
| 0-8 | Faz 7, 8 ve 9'u ayrı ayrı bir daha gez; özellikle 7.4 (kapılar), 8.2 (unit) ve 9.1 (en az yetki). |

Kaçırdığın soru → dönmen gereken köprü:

| Soru | Köprü (bölüm × bölüm) |
|---|---|
| 1, 11 | `nohup` vs unit + root patlama yarıçapı (8.2 × 9.1) |
| 2, 17 | Beş kapı × `systemctl status` yeşil (7.4 × 8.2) |
| 3, 16 | Bind adresi × yüzey daraltma (7.4.2 × 9.2) |
| 4, 18 | SG vs ufw bağımsız katmanlar (7.4.4 × 9.1.2) |
| 5 | daemon-reload × güvenlik değişikliği (8.2.2 × 9) |
| 6 | IAM role × SSH/SSM kimlik (7.3.4 × 9.6.2) |
| 7 | baked AMI × hardened AMI (8.4 × 9.3.2) |
| 8 | `ss -tulpn` üç göz (7.4 × 8.2 × 9.2) |
| 9, 20 | Capability vs MAC ayrımı (9.5 × 9.3) |
| 10, 12, 13, 15 | Senaryo: erişilebilirlik × yüzey (7.4 × 9.2) |
| 14, 21 | Patlama yarıçapı × en az yetki + MAC (9.1 × 9.3) |
| 19 | "İzin doğru ama engelli" = MAC (9.3.2) |

---

## Kapanış

Bu sınav, üç fazın tek bir mühendislik olayında nasıl kesiştiğini test etti: bir servisi ağa açmak (Faz 7),
onu yönetilebilir kılmak (Faz 8) ve bilinçli olarak sınırlamak (Faz 9) — aynı madalyonun üç yüzü. Senaryonun
dersi tektir ve pahalıdır: **erişilebilirliği yanlış katmanda çözmek güvenliği feda eder.** Mühendis "site
açılsın" diye SG'yi ardına kadar açtı; asıl arıza tek bir bind adresiydi. Üç dersi birlikte taşı: (1) teşhisi
her zaman dıştan içe yap, kör kör kapı açma; (2) bir süreci başlatmak onu yönetmek değildir — `nohup` değil,
`User=appuser` ile bir unit yaz; (3) güvenlik açığı önlemek değil, açık sömürüldüğünde patlama yarıçapını
daraltmaktır — en az yetki ve katmanlı savunma tam bunun içindir.

Faz 10 seni bu üç fazın manuel adımlarını **otomatikleştirmeye** taşıyor: elle kurup açıp sertleştirdiğin her
şeyi tekrarlanabilir betik ve yapılandırmaya çevirmek. "Bir kez yaptığın her manuel iş, ikinci kez bir betik
olmalı."

---

> **Navigasyon:** [◀ Faz 9 — Güvenlik ve Sertleştirme](Faz_9_Guvenlik_ve_Sertlestirme.md) · **Ara Sınav 4** · [Faz 10 — Otomasyon ve Scripting ▶](Faz_10_Otomasyon_ve_Scripting.md)
