# Ara Sınav 1 — Faz 0–2: Model, Shell ve Erişim

> **Navigasyon:** [◀ Faz 2 — Kullanıcılar, İzinler ve Kimlik](Faz_2_Kullanicilar_ve_Izinler.md) · **Ara Sınav 1** · [Faz 3 — Process ve Kaynak ▶](Faz_3_Process_ve_Kaynak.md)

---

## Bu sınav neyi ölçüyor?

Faz sonundaki "Kendini sına" testleri tek bir fazı yoklar: *"Bu fazı anladın mı?"* Ara sınav
**farklı** bir şey ölçer: *"Bu üç fazı birbirine bağlayabiliyor musun?"*

Gerçek bir Linux sunucusunda hiçbir sorun tek bir fazın içinde kalmaz. "Permission denied"
gördüğünde aynı anda üç fazın bilgisini kullanırsın: sorunun **nerede** oluştuğunu (Faz 0 —
kernel/user space sınırı, syscall), hangi **araçla** baktığını (Faz 1 — shell, pipe, akışlar) ve
**kimin neye** erişmeye çalıştığını (Faz 2 — kimlik, izin sınıfı). Bu sınavın soruları çoğunlukla
tek bir fazdan cevaplanamaz — tam da bu yüzden buradalar.

**Nasıl çalışılır:**

- Kâğıt-kalemle çöz; cevabı yazmadan bir sonrakine geçme.
- Cevap anahtarı her sorunun **hangi fazların kesişiminde** durduğunu söyler — kaçırdığın soru
  bir fazı değil, iki faz arasındaki **köprüyü** işaret eder.
- 21 soru var: Bölüm A (bağlantı muhakemesi, 1–9), Bölüm B (senaryo, 10–15), Bölüm C (komut ve
  çıktı okuma, 16–21).
- Hedef süre: ~1 saat. Ama süre önemli değil; önemli olan her cevabın *neden* öyle olduğunu bir
  cümleyle gerekçelendirebilmek.

> **🤔 Başlamadan önce:** Şu cümleyi kendi kelimelerinle tamamla: *"`sudo echo merhaba >
> /etc/deneme.txt` komutu neden `Permission denied` verir, oysa dosyayı yazacak olan root'tur?"*
> Bu tek soru üç fazı da içeriyor. Cevabını bir kenara yaz; Soru 4'te göreceğiz.

---

# Bölüm A — Bağlantı muhakemesi (1–9)

Bu bölümdeki her soru en az iki fazın bilgisini birleştirmeni ister. Kısa ama gerekçeli cevap ver.

**1.** "Her şey dosyadır" (Faz 0) ilkesi ile izin modeli (Faz 2) `/proc/1/environ` dosyasında nasıl
buluşur? Bu dosyayı neden sadece root okuyabilir, oysa `/proc/1/` altındaki `status` çoğu kullanıcı
tarafından okunabilir?

**2.** Bir programın `open("/etc/shadow", O_RDONLY)` çağrısı `EACCES` ile döner. Bu reddi **kim**
veriyor — shell mi, program mı, kernel mi? Cevabını Faz 0'daki user space / kernel space sınırıyla
ilişkilendir.

**3.** `cat /var/log/syslog | grep error` boru hattında `cat` `Permission denied` verirse `grep`
ne görür? İki komut aynı kimlikle mi çalışıyor, ve `sudo cat ... | grep ...` yazsaydın hangi taraf
root olurdu?

**4.** `sudo echo merhaba > /etc/deneme.txt` neden `Permission denied` verir? Yönlendirmeyi (`>`)
kim yapıyor ve hangi kimlikle? Aynı işi doğru yapan **iki** farklı komut yaz.

**5.** Faz 1'de `which`, PATH'te ilk eşleşen çalıştırılabilir dosyayı bulur. Bir saldırgan senin
`~/bin` dizinini PATH'in **başına** ekletip oraya sahte bir `ls` koyabilse ne olurdu? Bu neden aynı
zamanda bir Faz 2 (kimlik/izin) sorusudur?

**6.** `/usr/bin/passwd` dosyası neden hem FHS'e göre `/usr/bin`'de (Faz 1) hem de setuid bitli
(Faz 2)? Bu iki gerçeği tek bir cümlede bağla: "sıradan bir kullanıcı, kendisinin okuyamadığı bir
dosyayı nasıl güncelliyor?"

**7.** `ls -l /etc/shadow` çıktısında ilk karakter `-`, izinler `rw-r-----`, grup `shadow`. Bu tek
satır Faz 0'ın hangi ilkesini, Faz 1'in hangi aracını ve Faz 2'nin hangi üç kavramını aynı anda
gösteriyor?

**8.** `command 2>/dev/null` yazımı Faz 1'de stderr'i yutmak için kullanılır. Bir izin sorununu
ararken (`Permission denied` bir stderr mesajıdır) bu alışkanlık neden tehlikeli olabilir? Faz 2'nin
karar ağacıyla ilişkilendir.

**9.** Ubuntu'da `adduser`, Amazon Linux'ta `useradd` (Faz 2) — bu fark aslında Faz 0'ın hangi
kavramının (distro ailesi) doğrudan sonucudur? `apt` vs `dnf` farkıyla aynı kökten mi geliyor?

---

# Bölüm B — Senaryo: yeni sunucuya ilk giriş (10–15)

> Yeni oluşturulmuş bir Ubuntu 24.04 EC2 instance'ına `ubuntu` kullanıcısı olarak SSH ile girdin.
> Üstünde bir uygulama var: `/opt/app/run.sh` ve logları `/var/log/app/` altına yazıyor. Aşağıdaki
> altı soru bu sunucuda sırayla başına gelenler.

**10.** `ssh ubuntu@<ip>` çalıştırdın ama `Permission denied (publickey)` aldın. Bu mesaj Faz 2'nin
hangi mekanizmasına işaret ediyor, ve `~/.ssh/authorized_keys` dosyasının izni bununla nasıl
ilişkili? (İpucu: StrictModes.)

**11.** İçeri girdin. `docker ps` dedin, `permission denied while trying to connect to the Docker
daemon socket` aldın. `sudo usermod -aG docker ubuntu` çalıştırdın ama aynı terminalde hâlâ
reddediyor. Neden, ve çıkış-giriş yapmadan çözümün nedir? Bu hangi Faz 2 ilkesinin (kimlik ne zaman
sabitlenir) sonucu?

**12.** `cat /var/log/app/app.log` dedin, `Permission denied`. Dosya `ls -l` ile bakınca
`-rw-r----- 1 root adm`. Sen `adm` grubunda değilsin. Bu reddi çekirdek hangi izin sınıfına bakarak
verdi — sahip mi, grup mu, diğer mi? İki farklı çözüm yaz (biri kalıcı erişim, biri tek seferlik).

**13.** `/opt/app/run.sh` dosyası `755` ve sensin sahibi değil ama okuma iznin var. `./run.sh`
dedin, `Permission denied` aldın; `bash /opt/app/run.sh` çalıştı. `/opt` ayrı bir diskte mount'lu.
Muhtemel sebep ne, ve hangi tek komutla doğrularsın? (Bu soru Faz 1'in dosya sistemi + Faz 2'nin
çalıştırma bitini birleştirir.)

**14.** Uygulamayı düzeltmek için bir arkadaşın `sudo chmod -R 777 /opt/app` öneriyor. Bu komut hem
Faz 2 (izin modeli) hem Faz 0 (her process bu makinede) açısından neden kötü bir fikir? En az iki
somut zarar say.

**15.** Sorunun logunu bulmak için `sudo journalctl -u app | tail -50` dedin ve çıktı bir pager'da
(`less`) açıldı. Bu, sınırlı `sudo` yetkisi verilmiş bir kullanıcı için neden bir güvenlik sorunu
olabilir? (Faz 2'deki kabuğa kaçış fikriyle bağla.)

---

# Bölüm C — Komut ve çıktı okuma (16–21)

Aşağıdaki çıktıların her birinde **tek bir satırı** okuyup soruyu cevapla.

**16.** Bu iki çıktı arasındaki fark neyi kanıtlıyor?

```
$ id
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),27(sudo)
$ id ubuntu
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),27(sudo),988(docker)
```

**17.** `ls -l` çıktısında bir satır şöyle: `lrwxrwxrwx 1 root root 7 ... /bin -> usr/bin`. Bu satır
Faz 0'ın hangi kavramını (dosya türleri) ve Faz 1'in hangi gerçeğini (modern FHS'te `/bin` nedir)
aynı anda gösteriyor? İzinlerin (`rwxrwxrwx`) neden burada aldatıcı olduğunu söyle.

**18.** `stat -c '%A %U:%G %n' /tmp` çıktısı: `drwxrwxrwt root:root /tmp`. Sonundaki `t` nedir, hangi
somut davranışı sağlar, ve `/tmp` gibi herkese açık bir dizinde neden şart?

**19.** `findmnt -T /opt/app/run.sh` çıktısı:

```
TARGET SOURCE       FSTYPE OPTIONS
/opt   /dev/nvme1n1 ext4   rw,nosuid,nodev,noexec,relatime
```

13. sorudaki `./run.sh` reddini bu çıktının hangi kelimesi açıklıyor? `bash run.sh`'in neden yine de
çalıştığını bir cümleyle söyle.

**20.** `namei -l /home/ubuntu/site/index.html` çıktısı:

```
f: /home/ubuntu/site/index.html
drwxr-xr-x root   root   /
drwxr-xr-x root   root   home
drwxr-x--- ubuntu ubuntu ubuntu
drwxrwxr-x ubuntu ubuntu site
-rw-r--r-- ubuntu ubuntu index.html
```

Nginx (`www-data` kullanıcısı) bu dosyayı okuyamıyor. Hangi **satır** engeli koyuyor, ve neden
dosyanın kendi izni (`-rw-r--r--`, herkese okunur) bu durumu kurtarmıyor?

**21.** Aşağıdaki komut zinciri ne yapıyor ve her parçası hangi faza ait?

```
$ ps aux | grep nginx | awk '{print $1}' | sort -u
```

Çıktının `root` ve `www-data` içermesi Faz 2 açısından ne anlatır (neden nginx iki farklı kimlikle
process çalıştırır)?

---

## Cevap anahtarı

Her cevabın sonunda o sorunun **hangi fazların kesişiminde** durduğu belirtilmiştir.

**1.** İkisi de "her şey dosyadır"ın sonucu: `/proc/<pid>/environ` ve `status`, bellekteki process
bilgisini **dosya gibi** sunar (Faz 0). Ama bu sanal dosyaların da izinleri vardır (Faz 2):
`environ` bir process'in ortam değişkenlerini (parolalar, tokenlar olabilir) taşıdığı için `0400`
ve sahibi process'in sahibidir — PID 1 root'a ait olduğundan sadece root okur. `status` daha zararsız
meta veri olduğu için herkese açıktır. · *Faz 0 × Faz 2*

**2.** Reddi **kernel** verir. `open()` bir syscall'dır (Faz 0); kullanıcı programı isteği yapar ama
izin kontrolü user space'te değil, çağrının geçtiği kernel space sınırında yapılır. Shell zaten
işin içinde değil — programı başlatıp kenara çekilmiştir. Program da sadece `EACCES` errno'sunu
alır, kararı vermez. · *Faz 0 × Faz 2*

**3.** `grep` **boş girdi** görür (ya da hiçbir şey) ve eşleşme bulamaz; `cat`'in `Permission denied`
mesajı stderr'e gider, boruya değil — yani `grep`'e ulaşmaz, terminale düşer. Boru sadece stdout'u
taşır (Faz 1). Her iki komut da **senin** kimliğinle çalışır (Faz 2). `sudo cat ... | grep ...`
yazsaydın **sadece `cat`** root olurdu; `grep` yine sen olurdun — çünkü `sudo` yalnızca kendi
argümanı olan komutu yükseltir. · *Faz 1 × Faz 2*

**4.** Yönlendirmeyi (`>`) **shell** yapar, komut değil — ve shell **senin** kimliğinle çalışır
(Faz 1 + Faz 2). `sudo` sadece `echo`'yu root yapar; ama dosyayı açan `>` operatörü hâlâ sensin,
`/etc`'ye yazma iznin yok. Doğru yollar:
(a) `echo merhaba | sudo tee /etc/deneme.txt` — `tee` root olarak çalışıp dosyayı açar.
(b) `sudo sh -c 'echo merhaba > /etc/deneme.txt'` — yönlendirmeyi root'un çalıştırdığı bir kabuğun
içine koyar. · *Faz 1 × Faz 2*

**5.** Sen bir komut yazdığında (`ls`) shell PATH'i baştan tarar; `~/bin` baştaysa sahte `ls`
çalışır — bu bir **PATH ele geçirme** (Faz 1). Faz 2 sorusu olmasının sebebi: o sahte `ls` **senin
kimliğinle** çalışır, yani senin okuyabildiğin her şeyi okuyabilir, sen `sudo ls` dersen root olur.
Yetki, çalışan process'in kimliğine bağlıdır — dosyanın nereden geldiğine değil. · *Faz 1 × Faz 2*

**6.** `/usr/bin`, FHS'te "tüm kullanıcılar için ikincil komutlar"ın yeridir (Faz 1); `passwd`
oraya konur çünkü herkesin çalıştırması gereken bir araçtır. setuid biti (Faz 2) ise şunu çözer:
`passwd` çalıştığında **efektif UID'si root'a yükselir**, böylece sıradan kullanıcı `/etc/shadow`'u
kendisi okuyamazken `passwd` aracılığıyla kendi satırını güncelleyebilir. Tek cümle: *kullanıcı
dosyaya değil, dosyayı root yetkisiyle açan güvenilir bir programa erişir.* · *Faz 1 × Faz 2*

**7.** İlk karakter `-`: sıradan bir dosya — Faz 0'ın "her şey dosyadır" ve dosya türü kavramı.
`ls -l`: Faz 1'in temel gözlem aracı. `rw-r-----` + `root` sahip + `shadow` grup: Faz 2'nin üç
kavramı — izin bitleri (rwx), sahiplik (kullanıcı) ve grup sahipliği. Tek satır, üç fazın
kesişimidir. · *Faz 0 × Faz 1 × Faz 2*

**8.** `2>/dev/null` stderr'i çöpe atar (Faz 1). Bir izin sorununu ararken tehlikelidir çünkü
`Permission denied` mesajları **stderr'e** yazılır — onları susturursan Faz 2 karar ağacının en
önemli ipucunu (hangi yolda, hangi errno ile reddedildi) kaybedersin. `find / -name x 2>/dev/null`
gürültüyü azaltmak için yaygındır ama teşhis yaparken stderr'e **bakmak** gerekir. · *Faz 1 × Faz 2*

**9.** Evet, aynı kökten: distro **ailesi** (Faz 0). Ubuntu Debian ailesindendir → `apt` + `adduser`;
Amazon Linux/RHEL Red Hat ailesindendir → `dnf` + `useradd`. Paket yöneticisi ve kullanıcı
araçlarının farkı, aynı çekirdeğin (Linux) üstüne farklı ekosistemlerin kurulmasının sonucudur;
"hangi distro ailesi" sorusu her ikisini de aynı anda cevaplar. · *Faz 0 × Faz 2*

**10.** `Permission denied (publickey)`: sunucu, senin özel anahtarınla eşleşen bir açık anahtarı
`~/.ssh/authorized_keys`'te bulamadı ya da bulmasına rağmen **StrictModes** kontrolünde takıldı
(Faz 2, 2.5.1). sshd, `.ssh` dizininin `700` ve `authorized_keys`'in `600` (ve ev dizininin
başkasına yazılabilir olmaması) şartını arar; bunlar gevşekse anahtar geçerli olsa bile reddeder.
Yani izin **fazlalığı** da giriş engeller. · *Faz 2 (× cloud-init, Faz 5'e köprü)*

**11.** Grup listesi oturum açılışında sabitlenir ve `fork` ile miras kalır (Faz 2, 2.1.1); çalışan
kabuğun `usermod`'dan sonra yazılan yeni `docker` grubundan haberi yok. Çözüm: `newgrp docker`
(mevcut kabukta yeni grup bağlamı) ya da tam çıkış-giriş. `id` eski listeyi, `id ubuntu` yeni
listeyi gösterir — fark tam da sorunun kaynağıdır. · *Faz 2, Cevap 2.1*

**12.** Çekirdek **grup** sınıfına baktı ve reddetti: sen sahip değilsin (`root` sahip), `adm`
grubunda da değilsin, dolayısıyla "diğer" sınıfına düştün ve diğer için hiçbir izin yok (`----`).
Çözümler: (a) kalıcı — `sudo usermod -aG adm ubuntu` (sonra yeniden giriş) ile `adm` grubuna gir;
(b) tek seferlik — `sudo cat /var/log/app/app.log` ya da `sudo less ...`. · *Faz 2, 2.2.1–2.2.3*

**13.** `/opt` mount'u muhtemelen `noexec`'tir; `./run.sh` bir `execve` çağırır ve dosya bitleri
`755` olsa bile `noexec` mount'ta çalıştırma reddedilir (Faz 2, 2.5.3). `bash run.sh` çalışır çünkü
çalıştırılan şey `bash`'tir (exec izinli bir mount'ta); `run.sh` sadece **okunur**. Doğrulama:
`findmnt -T /opt/app/run.sh`. · *Faz 1 (mount) × Faz 2 (execve)*

**14.** (a) `777` dizindeki her dosyayı **makinedeki her process'e** yazılabilir yapar — ele
geçirilmiş bir servis bile uygulama kodunu değiştirebilir (Faz 0: her process bir kimlikle çalışır,
Faz 2: 2.2.2). (b) `-R` ile içindeki gizli anahtar/config dosyaları da herkese açılır. (c) Kimin
neden reddedildiğini hiç öğrenmeden sorunu "çözer" — yani asıl mekanizmayı gizler. Doğrusu: kim,
neye, hangi izinle erişmeli sorusunu dar biçimde cevaplamak. · *Faz 0 × Faz 2*

**15.** `less` (pager) içinden `!sh` yazarak bir kabuk açılabilir. `sudo journalctl` root olarak
çalışıyorsa ve çıktı root'un başlattığı bir pager'da açılıyorsa, o `!sh` **root kabuğu** verir —
sınırlı sanılan `sudo journalctl` yetkisi tam root'a dönüşür (Faz 2, 2.4.1 kabuğa kaçış). Bu yüzden
`sudo` verilen komutların içinden başka komut çalıştırılabiliyor mu diye bakılır (`SYSTEMD_PAGER`,
`sudoedit` mantığı). · *Faz 2, 2.4*

**16.** `id` çalışan kabuğun **anlık** kimliğini gösterir (docker YOK); `id ubuntu` `/etc/group`'tan
**taze** okur (docker VAR). Fark, kimliğin oturumda sabitlenip dosyanın ise güncel olmasıdır — yani
"gruba eklendim ama çalışmıyor" durumunun kanıtı. · *Faz 2, 2.1.1*

**17.** `l` ile başlaması: bir **sembolik link** (Faz 0, dosya türleri). `/bin -> usr/bin`: modern
FHS'te `/bin` artık ayrı bir dizin değil, `/usr/bin`'e bir linktir (Faz 1, "usr birleşmesi").
İzinler `rwxrwxrwx` aldatıcıdır çünkü **linkin kendi izinleri kullanılmaz** — erişim, işaret ettiği
**hedefin** izinlerine göre yapılır. · *Faz 0 × Faz 1*

**18.** Sondaki `t` **sticky bit**'tir (Faz 2, 2.3.1). Sağladığı davranış: dizinde herkes dosya
oluşturabilir ama bir dosyayı **yalnızca sahibi** (ya da root) silebilir. `/tmp` herkese yazılabilir
(`rwxrwxrwx`) olduğu için şarttır — sticky olmasa herhangi bir kullanıcı başkasının geçici
dosyalarını silebilirdi. · *Faz 2, 2.3.1*

**19.** `noexec`. `./run.sh` dosyayı **çalıştırmak** ister (execve), `noexec` mount bunu bit'lerden
bağımsız reddeder. `bash run.sh` çalışır çünkü orada çalıştırılan `bash`'tir; `run.sh` sadece
okunan bir veri dosyasıdır. · *Faz 2, 2.5.3 × Faz 6 köprüsü*

**20.** Engeli koyan satır: `drwxr-x--- ubuntu ubuntu ubuntu` — yani `/home/ubuntu` dizini. `www-data`
ne sahip (`ubuntu`) ne de grupta, "diğer" sınıfına düşer ve orada `---` var; yani bu dizinden
**geçme (traverse) izni yok**. Yolun bir dizininde `x` eksikse, alttaki dosyanın izni ne olursa
olsun çekirdek daha o dosyaya ulaşamadan reddeder. Dosyanın `-rw-r--r--` olması kurtarmaz çünkü
sorun okuma değil, **yola girme**. · *Faz 2, 2.2.3*

**21.** `ps aux` çalışan process'leri listeler (Faz 3'ün ön tadımı ama burada Faz 1 aracı olarak),
`grep nginx` nginx satırlarını süzer, `awk '{print $1}'` ilk sütunu (kullanıcı adını) alır, `sort -u`
tekilleştirir — hepsi Faz 1'in akış/pipe/metin işleme cephanesi. `root` ve `www-data` çıkması: nginx
**master** process'i root olarak başlar (80 gibi ayrıcalıklı porta bağlanmak için), sonra **worker**
process'lerini yetkisiz `www-data` kimliğiyle çalıştırır — en az yetki ilkesi (Faz 2, 2.4.2). ·
*Faz 1 × Faz 2 (× Faz 3 köprüsü)*

---

## Puanlama

| Doğru sayısı | Değerlendirme |
|---|---|
| 19–21 | Üç fazı tek bir modele bağlamışsın. Faz 3'e güvenle geç. |
| 15–18 | Sağlam. Kaçırdığın soruların işaret ettiği **köprüye** (tek faza değil) bir tur dön. |
| 10–14 | Fazları ayrı ayrı biliyorsun ama arasındaki bağ zayıf. Aşağıdaki tabloyu kullan. |
| 0–9 | İlgili fazları, özellikle "Bozulunca" ve "Kapanış" bölümlerini tekrar et; ara sınav bu köprüler oturmadan geçilmez. |

**Hangi soruyu kaçırırsan nereye dön:**

| Kaçırdığın soru | Dön — bu köprü zayıf |
|---|---|
| 1, 2, 7, 17 | Faz 0 × Faz 2 — "her şey dosyadır" + izin/dosya türü |
| 3, 4, 5, 8 | Faz 1 × Faz 2 — shell/pipe/yönlendirme **senin kimliğinle** çalışır |
| 6, 9 | Faz 0/1 × Faz 2 — FHS + setuid, distro ailesi + araçlar |
| 10, 11, 16 | Faz 2 (2.1, 2.5) — kimlik ne zaman sabitlenir, SSH/StrictModes |
| 12, 20 | Faz 2 (2.2.1–2.2.3) — izin sınıfı seçimi ve yol traverse |
| 13, 19 | Faz 1/6 × Faz 2 — mount seçenekleri (noexec) + execve |
| 14, 15, 18, 21 | Faz 2 (2.3, 2.4) — özel bitler, en az yetki, kabuğa kaçış |

---

## Kapanış — buradan Faz 3'e

Bu üç faz sana bir sunucuya girip **statik** bir resmini okumayı öğretti: hangi dosya, kimin,
hangi izinle; hangi komut, hangi kimlikle. Ama henüz bir şeyi soramadın: **"bu makine şu an ne
yapıyor?"** Bir process'in kimliği var (Faz 2) ama process'in kendisini — PID'sini, ebeveynini,
durumunu, tükettiği kaynağı — daha görmedin.

Faz 3 tam buraya bağlanır: Faz 2'nin "her process bir UID/GID taşır" cümlesini alıp üstüne "her
process bir PID, bir ebeveyn ve bir durum taşır"ı koyar; `sudo`'nun "farklı kimlikle process
başlatmak" olduğunu gördün, şimdi o process'lerin nasıl doğduğunu (`fork`/`exec`), nasıl
öldüğünü (sinyaller, zombie) ve kaynaklarının nasıl sınırlandığını (cgroup — container'ların
altındaki gerçek) göreceksin.

> **Devam etmeden önce:** Yukarıdaki 21. sorunun cevabını tereddütsüz verebiliyorsan — neden nginx
> aynı anda `root` ve `www-data` kimlikleriyle process çalıştırır — Faz 3'e hazırsın. Veremiyorsan,
> Faz 2'nin 2.4.2 (en az yetki) bölümüne bir dakika dön; Faz 3 o cümlenin üstüne kurulacak.

---

> **Navigasyon:** [◀ Faz 2 — Kullanıcılar, İzinler ve Kimlik](Faz_2_Kullanicilar_ve_Izinler.md) · **Ara Sınav 1** · [Faz 3 — Process ve Kaynak ▶](Faz_3_Process_ve_Kaynak.md)
