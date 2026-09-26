# Linux — Cloud Engineering Öğrenme Yol Haritası

---

## Mentor Başlangıç Talimatı — bu dosyayı okuyan AI için

Bu, Linux yol haritasıdır ve sen bu haritayı işleyen öğrencinin **Socratic mentoru**sun. Bu dosya sana verildiyse aşağıdaki kuralları benimse ve öğrencinin yön vermesini bekle. Öğrenci "**Faz X'teyiz**" (fazın başından) ya da "**Faz X.Y'deyiz**" (alt-maddeden) dediğinde, o noktadan **kesintisiz** devam et — baştan özet çıkarma, izin isteme, dağılma.

**Öğrenme ritmi — en önemli kural: yapıcı (constructive) Socratic yöntem.** Bu ne saf soru-cevap ne de düz anlatımdır; ikisinin birleşimidir. Her kavramda sıra şudur: **(1) yönlendirici bir soru sor** — "sence X'in olması için ne gerekiyor olabilir?" gibi, öğrencinin **elindeki mevcut bilgiyle** tahmin yürütebileceği bir soru → **(2) öğrenci tahmin eder** ("şu şu olabilir mi?") → **(3) doğruysa hemen onayla ve terimi ver:** "evet, tam olarak — bunun adı **X**'tir, şu anlama gelir: …" → **(4) yanlışsa dolambaçlı soruyla uğraşma, direkt düzelt** → **(5) terim konduktan sonra üstüne birlikte kavram inşa edin.**

Kritik kural: **Öğrencinin elinde hiç olmayan bir kavramı tahmin ettirmeye çalışma.** Soru, keşif ettirmek için değil, bildiğinden bilmediğine **köprü kurmak** içindir. Cevap gelir gelmez adını koy, tanımını ver, kavramı havada bırakma. Bir kavram tamamlanmadan sonrakine geçilmez.

**Öğrencinin seviyesine göre kalibrasyon.** Bu harita, Linux'a tümüyle yabancı olmayan bir öğrenci varsayar (navigasyon, izinler, dosya işlemleri, temel kullanım bilinir kabul edilir); ama seviyeyi baştan bilemezsin. Bu yüzden temel fazlarda `ls`/`cd` gibi bilinme ihtimali yüksek konularda oyalanma — bildiğini hızla teyit et, **derinliğe ve "neden"e** geç. Öğrenci bir hardening (UFW, AppArmor, auditd, rkhunter, SSH sıkılaştırma) ya da troubleshooting geçmişi olduğunu belirtirse ilgili kısımları hızlı geç. Emin değilsen "bunu biliyor musun, yoksa üstünden geçelim mi?" diye sor; varsayma.

**Faz açılışı.** Bir faza girerken önce ~2 satır oryantasyon ver: bu faz neyi kapsıyor + önceki fazdan neyi bildiğini varsaydığın + genel bağlam. Ondan sonra ilk kavramın kısa anlatımına geç. (Her faz bir öncekinin kelimelerini kullanır; sırayı bozma.)

**Dil ve stil.**
- Türkçe. İngilizce teknik terim ilk geçişte parantez içinde Türkçe telaffuz + kısa tanım (ör. "syscall (sis-kol = kullanıcı programının çekirdekten iş istemesi)").
- Derinlik etiketlerine uy: `[kavram]` sezgisel açıklayabilecek kadar, `[mekanizma]` adım adım anlat, `[uygulama]` **kendi Ubuntu makinende elle çalıştır ve gör**, `[atla]` sadece adını bil (Cloud Engineer seviyesinde derine inme).
- Cloud örnekleriyle bağla; her konunun **"Bozulunca"** açısını kullan — arıza içgüdüsü bu haritanın kalbidir. Linux'un işi cloud'da %80 "bu neden bozuk?" sorusudur; bu yüzden failure-mode her fazın merkezinde.

**Düzeltme — direkt olsun.** Öğrenci yanlış cevap verdiğinde onu doğruya götürecek ikinci bir soru arama; **doğrusunu net biçimde söyle**, ardından kısa gerekçesini ver. Dolaylı yönlendirme yavaşlatır ve yorar. Öğrenci "direkt söyle" dediğinde tartışmasız direkt anlat.

**Lab.** İlgili `[uygulama]` kavramından **hemen sonra** o faza ait Lab'ı öner (dosyadaki komutlarla). Bu bir **öneridir, zorlama değildir.** Öğrenci komutu çalıştırıp çıktıyı getirirse birlikte okursunuz. "Şu an makinede değilim" veya "sonra" derse **zorlama, ısrar etme, akışı bozma** — kısa bir not düş ve devam et. Karar öğrencinindir; sen kararına uyarsın. Uyarı: bazı Lab'lar sistemi değiştirir (mount, fstab, servis); yıkıcı olabilecek adımlarda önce "bu kalıcı, dikkat" notu ver.

**Ara sınavlar.** ✅ ile işaretli checkpoint'lere gelince ara sınavı **otomatik öner.** Öğrenci ertelemek isterse **ertele**, dayatma.

**İletişim.** Tetik/komut sistemi yok — doğal konuş. **Öğrenciyi anlamadığın anda anladığını varsayma; açıkça "seni tam anlamadım" de** ve netleştirici soru sor.

**Sınırlar.** Aynı anda tek kavram; konu atlama yok; istenmemiş roadmap sapması yok. Bu harita **kernel geliştiricisi** yetiştirmez — amaç, sistemi kararları gerekçelendirecek ve arızayı avlayacak kadar derinden görmek. Faz-içi ilerlemeyi **sen takip etme** — öğrenci kendi yönetir ve "Faz X.Y'deyiz" diyerek konumu sabitler. Geri bildirim dürüst ve direkt olur.

**Bağımsızlık notu.** Bu harita network ve hardware haritalarından **bağımsızdır** — onlara referans vermeden tek başına yürür. Ağ *protokol teorisi* gerektiren yerlerde (Faz 7) teoriyi başka bir alana bırak, burada sadece **OS tarafındaki config ve araçlara** odaklan.

---

> **Kuzey yıldızı — üç içgüdü sorusu.**
> Bu haritanın tek amacı, bir gün bir Linux sunucusuna baktığında bu üç soruyu düşünmeden cevaplayabilmen:
> 1. **Sistem şu an ne yapıyor, kaynağı ne yiyor?** → process durumu, load, CPU/RAM/IO tüketicisi (Faz 3, 4, 11)
> 2. **Neden erişemiyorum / neden "permission denied"?** → izin modeli, ownership, MAC, capabilities (Faz 2, 9)
> 3. **Bu makine boot'tan çalışan servise nasıl geliyor?** → boot → systemd → cloud-init → servis (Faz 5, 8, 12)
>
> Tasarım prensibi: troubleshooting sona bırakılmadı. Her fazda
> **"bu bilgi bozulunca nasıl görünür?"** açısı var. İçgüdü ancak böyle oluşur. Faz 11 bu üç sorunun "bozulunca" hâlini tek bir sistematik reflekse dönüştürür.

## Derinlik etiketleri (diğer haritalarla aynı sistem)

- `[kavram]` — sadece ne olduğunu bil, sezgisel açıklayabil.
- `[mekanizma]` — adım adım nasıl çalıştığını anlatabil.
- `[uygulama]` — kendi Ubuntu makinende elle çalıştır, gör.
- `[atla]` — farkındalık yeter, derine inme.
- **Cloud bağlantısı** — konunun AWS'te nereye map olduğu.
- **Bozulunca** — bu konu arızalanınca sistemde nasıl görünür (içgüdü tetikleyici).

---

## Faz 0 — Zihinsel Model: Linux Neden ve Nasıl?

Bu fazın amacı: Linux'u komut ezberi olarak değil, **çekirdek + kullanıcı alanı + "her şey dosyadır"** üçlüsüyle tutarlı bir sistem olarak kurmak. Cloud'daki her sunucu bir Linux kutusudur; bu modeli oturtmadan sonraki her şey havada kalır.

### Konular

**0.1 Bir işletim sisteminin işi**
- Kernel (körnıl = çekirdek): donanımı yöneten, kaynağı paylaştıran katman `[kavram]`
- Kullanıcı programlarının donanıma doğrudan değil, çekirdek üzerinden erişmesi `[kavram]`

**0.2 Kernel space vs User space**
- İki ayrı ayrıcalık dünyası: neden uygulaman RAM'e/diske doğrudan dokunamaz `[mekanizma]`
- Syscall (sis-kol = kullanıcı programının çekirdekten iş istemesi): iki dünya arasındaki tek kapı `[mekanizma]`
- **Bozulunca:** bir process çekirdekte takılırsa (D state) neden `kill -9` bile onu öldüremez

**0.3 "Her şey dosyadır" felsefesi**
- Cihazlar, process'ler, ağ soketleri — hepsi dosya arayüzüyle görünür `[kavram]`
- `/proc` ve `/sys`: çekirdeğin canlı durumunu dosya gibi okumak `[kavram]`

**0.4 Distro manzarası (farkındalık)**
- Debian/Ubuntu ailesi vs RHEL/Fedora/Amazon Linux ailesi — paket yöneticisi ve varsayılan farkları `[kavram]`
- **Cloud bağlantısı:** AMI (ey-em-ay = donmuş bir Linux imajı) — Amazon Linux 2023 vs Ubuntu tercihi neyi değiştirir `[kavram]`

> **Faz 0 çıktısı:** "Bir uygulama diske yazmak istediğinde ne oluyor" — user space'ten syscall ile kernel'e geçişi kendi cümlelerinle anlatabilmelisin.

---

## Faz 1 — Shell ve Dosya Sistemi: Operatörün Eli

Bu fazın amacı: shell'i cloud'daki **birincil arayüzün** olarak ustalaşmak ve dosya sistemi hiyerarşisini bir harita gibi ezber değil mantıkla okumak. Bir cloud engineer sunucuya çoğu zaman SSH'la, yani shell'le girer.

### Konular

**1.1 Shell nedir**
- Shell (şel = komutları yorumlayan program, bash) bir REPL: oku-çalıştır-yazdır döngüsü `[kavram]`
- Komut = program çağrısı; PATH ve komutun nasıl bulunduğu `[mekanizma]`

**1.2 Filesystem Hierarchy (FHS) — neyin nerede olduğu**
- `/etc` (config), `/var/log` (loglar), `/proc` & `/sys` (canlı durum), `/dev` (cihazlar), `/tmp`, `/home`, `/usr` `[mekanizma]`
- Bir cloud engineer'in en çok gittiği üç yer: `/etc`, `/var/log`, `/proc` — neden `[uygulama]`

**1.3 Navigasyon ve dosya işlemleri (hızlı teyit)**
- `cd`, `ls`, `cp`, `mv`, `rm`, `find` — biliniyorsa üstünden geçme `[uygulama]`
- `find` ile arıza avı: değişen dosyaları, büyük dosyaları bulmak `[uygulama]`

**1.4 Standart akışlar ve yönlendirme**
- stdin / stdout / stderr — üç ayrı akış, neden `2>` ayrı `[mekanizma]`
- Redirection (`>`, `>>`, `2>&1`) ve pipe (payp = `|`, bir programın çıktısını diğerine akıtmak) `[mekanizma]`
- **Unix felsefesi:** küçük araçları birbirine bağlamak — bu haritanın en önemli zihinsel modeli `[kavram]`

**1.5 Metin işleme cephanesi**
- `grep` (grep = satır süzme), `cut`, `sort`, `uniq`, `wc`, `head`/`tail` `[uygulama]`
- `sed` ve `awk` — akış üzerinde düzenleme ve alan işleme (kavram + temel kullanım) `[kavram]`
- **Cloud bağlantısı:** SSH üzerinden log tail'lemek, bir config'i grep'lemek — günlük iş
- **Bozulunca:** "logda hatayı bulamıyorum" → `grep -i error | tail` refleksi

> **Lab 1:** `journalctl -p err -n 50` çıktısını `grep`/`awk` ile süz. `find /var/log -size +50M` ile şişmiş logları bul. `df -h /var` ile `/var` doluluğunu gör.

---

## Faz 2 — Kullanıcılar, İzinler ve Kimlik: Erişim Modeli

Bu fazın amacı: "**neden erişemiyorum**" sorusunun kökü. Linux izin modeli, cloud'daki güvenlik kararlarının OS tarafındaki temelidir ve IAM'den ayrı bir katmandır.

### Konular

**2.1 Kullanıcı ve grup**
- `/etc/passwd`, `/etc/shadow`, `/etc/group` — kimlik nerede tutulur `[mekanizma]`
- UID/GID, root (UID 0) vs normal kullanıcı `[kavram]`

**2.2 İzin bitleri**
- rwx üçlüsü × (owner, group, other); octal gösterim (755, 644) `[mekanizma]`
- `chmod`, `chown`, `chgrp` `[uygulama]`
- Dizinde `x` biti ne demek (içine girebilme) — sık karıştırılan nokta `[mekanizma]`

**2.3 Özel bitler**
- setuid/setgid (set-yu-ay-di = programı sahibinin yetkisiyle çalıştırma) ve sticky bit `[kavram]`
- **Bozulunca:** yanlış setuid → yetki yükseltme açığı

**2.4 Privilege escalation: sudo**
- `sudo` ve `/etc/sudoers` — kim neyi root olarak çalıştırabilir `[mekanizma]`
- Neden production'da uygulamayı root çalıştırmak kötü fikir `[uygulama]`

**2.5 OS kimliği vs Cloud kimliği**
- **Cloud bağlantısı:** EC2'de `ubuntu`/`ec2-user` default kullanıcısı, SSH public key'in boot'ta enjekte edilmesi; OS izinleri ≠ IAM — ikisi ayrı katman
- **Bozulunca:** "permission denied" karar ağacı — kullanıcı mı, izin biti mi, ownership mı, mount mu (noexec/ro), MAC (Faz 9) mu

> **Faz 2 çıktısı:** Sana `-rwxr-x---  root  devops  deploy.sh` versem — kim çalıştırabilir, kim okuyabilir, `alice` (grup: devops değil) ne yapabilir — anında söyleyebilmelisin.
>
> **Lab 2:** `id`, `groups` ile kendini tanı. `ls -l /etc/shadow` ile neden okunamadığına bak. `stat dosya` ile tüm izin/ownership/zaman bilgisini oku.

---

## ✅ Ara Sınav 1 (Faz 0–2)

Zihinsel model + shell + izinler. Bu üçü olmadan çalışan bir sistemi okumak anlamsız.

---

## Faz 3 — Process ve Kaynak Yönetimi: Çalışan Sistem

Bu fazın amacı: "**sistem şu an ne yapıyor**" sorusunun çekirdeği. Ve container'ların altındaki gerçeği (cgroup + namespace) burada gömüyoruz — bu, ileride EC2/EKS kararlarının temeli.

### Konular

**3.1 Process anatomisi**
- PID (pi-ay-di = process kimliği), PPID (ebeveyn), init'in PID 1 olması `[mekanizma]`
- `fork` + `exec`: bir process başka bir process'i nasıl doğurur `[mekanizma]`

**3.2 Process durumları**
- Running, Sleeping, Stopped, Zombie, Uninterruptible (D — I/O'da kilitli) `[mekanizma]`
- **Bozulunca:** zombie birikimi (ebeveyn `wait` etmiyor) ve D-state (disk/NFS takılması) — neden `kill -9` D-state'i öldüremez

**3.3 Thread vs Process**
- Aynı adres uzayını paylaşan iş parçacıkları vs izole process'ler `[kavram]`
- Cloud bağlantısı: çok-thread'li uygulamada vCPU sayısının etkisi `[kavram]`

**3.4 Sinyaller**
- SIGTERM (nazik dur) vs SIGKILL (zorla öldür) vs SIGHUP (yeniden yükle) `[mekanizma]`
- **Cloud bağlantısı:** container/instance kapanırken uygulamaya önce SIGTERM gelir — graceful shutdown neden önemli `[uygulama]`

**3.5 Job control ve öncelik**
- Foreground/background, `&`, `nohup`, `jobs` `[kavram]`
- `nice`/`renice` — CPU önceliği `[kavram]`

**3.6 Kaynak sınırları: ulimit ve cgroups**
- `ulimit` — process başına dosya/bellek limitleri `[kavram]`
- cgroup (si-grup = control group = bir process grubunun CPU/RAM/IO kotası) `[mekanizma]`
- namespace — process'in gördüğü dünyayı izole etme (PID, mount, net) `[kavram]`
- **Cloud bağlantısı:** cgroup + namespace = container'ın kendisi; "container hafif VM değildir" burada anlaşılır `[uygulama]`

**3.7 Gözlem araçları**
- `ps aux`, `top`/`htop`, `pgrep`/`pkill`, `/proc/<pid>/` `[uygulama]`
- **Bozulunca:** tek bir process CPU'yu %100 yiyor → `top` → PID → `/proc/<pid>/` ile ne yaptığını görmek

> **Lab 3:** `ps -eLf` ile thread'leri gör. `sleep 300 &` ile arka plana at, `jobs`/`kill` ile yönet. `cat /proc/<pid>/status` ile bir process'in durumunu ve bellek kullanımını oku.

---

## Faz 4 — Bellek, I/O ve Performans Sezgisi

Bu fazın amacı: "kaynağı ne yiyor" sorusunun bellek/IO tarafı. Cloud'un en çok yanlış okunan metrikleri buradadır — özellikle "RAM dolu görünüyor ama değil" klasiği.

### Konular

**4.1 Sanal bellek ve process belleği**
- RSS (gerçekte kullanılan fiziksel RAM) vs VSZ (ayrılan sanal alan) farkı `[mekanizma]`
- Shared memory neden toplamı yanıltır `[kavram]`

**4.2 Page cache — "boş RAM" yanılgısı**
- Linux boş RAM'i disk cache olarak kullanır; `free`'deki buff/cache aslında müsait `[mekanizma]`
- **Bozulunca:** "RAM %95 dolu, panik" → aslında page cache; gerçek baskı `available` sütununda `[uygulama]`

**4.3 Swap ve OOM**
- Swap (svap = RAM taşınca diske sarkıtılan alan) neden yavaş `[mekanizma]`
- `swappiness` ve OOM Killer (o-o-em = RAM tükenince çekirdeğin process öldürmesi) `[mekanizma]`
- **Cloud bağlantısı:** üretimde swap neden tehlikeli; memory leak senaryosu; hangi process'in öldürüleceği (oom_score) `[uygulama]`

**4.4 Load average'ı doğru okumak**
- Load = CPU değil, **run queue** uzunluğu (çalışmayı bekleyen + D-state) `[mekanizma]`
- **Bozulunca:** yüksek load + düşük CPU → neredeyse her zaman I/O wait `[uygulama]`

**4.5 I/O sezgisi**
- Blocking vs non-blocking I/O; `iowait` ne anlatır `[kavram]`
- **Bozulunca:** disk doygun → tüm sistem "yavaş" ama CPU boşta

**4.6 Darboğaz triyajı (OS gözüyle)**
- CPU-bound mu, memory-bound mu, I/O-bound mu, network-bound mu — hangi araç ele verir `[uygulama]`
- Araçlar: `free`, `vmstat`, `iostat`, `top`, `/proc/meminfo`
- **Cloud bağlantısı:** bu triyaj right-sizing kararının OS tarafı; CloudWatch metrikleri bunun uzaktan hâli

> **Faz 4 çıktısı:** "Uygulamam yavaş" dendiğinde, hangi tek komutla başlayıp darboğazın dört türünden hangisi olduğunu 60 saniyede daraltacağını söyleyebilmelisin.
>
> **Lab 4:** `free -h` çıktısındaki `available` ile `free` farkını yorumla. `vmstat 1` ile `wa` (io wait) ve `si/so` (swap) sütunlarını canlı izle. `uptime` load'unu core sayınla kıyasla.

---

## ✅ Ara Sınav 2 (Faz 3–4)

Process + bellek + IO. "Sistem ne yapıyor / kaynağı ne yiyor" sorusunun tamamı burada.

---

## Faz 5 — Boot, Init ve systemd: Makinenin Yaşam Döngüsü

Bu fazın amacı: "**boot'tan servise makine nasıl geliyor**" sorusunun çekirdeği. Cloud'da bir instance'ın AMI'den çalışan sunucuya dönüşmesinin tamamı buraya dayanır.

### Konular

**5.1 Boot zinciri**
- Firmware → bootloader (GRUB = grab) → kernel → initramfs → init `[mekanizma]`
- initramfs neden var (kök disk mount edilmeden önceki geçici kök) `[kavram]`

**5.2 init sistemleri ve systemd**
- init = ilk process (PID 1), her şeyin atası `[kavram]`
- systemd (sistem-di = modern init + servis yöneticisi) neden standart oldu `[kavram]`

**5.3 systemd unit'leri**
- service, socket, timer, target, mount — unit tipleri `[mekanizma]`
- `systemctl start/stop/enable/status/restart` — `enable` ≠ `start` ayrımı `[uygulama]`
- Bağımlılık ve sıralama (After/Requires/Wants) `[kavram]`

**5.4 journald ve loglar**
- `journalctl` — systemd'nin log arayüzü; `-u servis`, `-p err`, `-b` (bu boot) `[uygulama]`

**5.5 Zamanlanmış işler**
- cron (kron = klasik zamanlayıcı) vs systemd timer — farkları `[kavram]`

**5.6 Cloud-init: bulutun boot-time sihri**
- **Cloud bağlantısı:** cloud-init + user-data — EC2'nin AMI'den çıkıp boot anında kendini yapılandırması (paket kurma, kullanıcı, servis başlatma) `[uygulama]`
- **Bozulunca:** "servis enabled ama başlamadı" (dependency), user-data script'i sessizce patladı (`/var/log/cloud-init-output.log`), boot bir unit'i beklerken asıldı

> **Lab 5:** `systemctl list-units --type=service --state=running` ile çalışan servisleri gör. `systemd-analyze blame` ile boot'u kimin yavaşlattığını bul. `journalctl -u ssh -b` ile SSH servisinin bu boot'taki loglarını oku.

---

## Faz 6 — Depolama ve Dosya Sistemleri: Kalıcı Katman

Bu fazın amacı: verinin diskte nasıl durduğu ve mount edildiği. EBS volume'ü ekleme → bölme → mount → kalıcı yapma akışı ve bu akışın **boot'u kilitleyebilen** tuzakları buraya dayanır.

### Konular

**6.1 Blok cihaz vs dosya sistemi**
- `/dev/sda`, `/dev/nvme0n1` — ham blok cihaz; üstüne dosya sistemi "biçimlenir" `[mekanizma]`
- `lsblk` ile cihaz ağacını okumak `[uygulama]`

**6.2 Bölme, biçimleme, mount**
- Partition, `mkfs`, `mount`/`umount`, mount point kavramı `[mekanizma]`
- `/etc/fstab` — kalıcı mount tanımı `[mekanizma]`
- **Bozulunca:** `fstab`'da yanlış satır → makine boot'ta kurtarma moduna düşer (en klasik "instance açılmıyor" nedeni) `[uygulama]`

**6.3 Dosya sistemleri**
- ext4 vs xfs — cloud'da yaygın olanlar `[kavram]`
- Journaling fikri (yarım kalan yazımdan kurtulma) `[kavram]`

**6.4 Inode — gizli tuzak**
- inode (ay-nod = dosyanın metadata kaydı); disk dolmadan **inode** tükenebilir `[mekanizma]`
- **Bozulunca:** `df -h` yer var diyor ama "no space left on device" → `df -i` inode doluluğu `[uygulama]`

**6.5 LVM ve genişletme (kavram)**
- LVM (el-vi-em = mantıksal disk yönetimi) — disk büyütmeyi esnetir `[kavram]`

**6.6 Cloud depolama akışı**
- **Cloud bağlantısı:** EBS volume attach → `lsblk` ile gör → mount → `fstab` (UUID ile, cihaz adı ile değil — neden); kök volume'ü büyütme (`growpart` + `resize2fs`); instance store'un ephemeral olması (host'a bağlı, durunca gider) `[uygulama]`

> **Faz 6 çıktısı:** Yeni bir EBS volume ekledim — onu kalıcı, boot'u kilitlemeyecek şekilde mount etme adımlarını sırayla söyleyebilmelisin (ve neden UUID kullandığını).
>
> **Lab 6:** `lsblk` ve `df -hT` ile mevcut disk düzenini oku. `df -i` ile inode kullanımına bak. `cat /etc/fstab` ile hangi mount'ların UUID ile tanımlı olduğunu incele (**fstab'ı düzenleme — sadece oku**).

---

## ✅ Ara Sınav 3 (Faz 5–6)

Yaşam döngüsü + depolama. "Makine boot'tan çalışan servise nasıl geliyor" + "veri nerede duruyor" burada birleşir.

---

## Faz 7 — Ağ (OS Katmanı): Sunucunun Dış Dünyası

Bu fazın amacı: **protokol teorisi değil**, sunucunun ağ tarafının OS'teki config ve araçları. SSH burada merkezdedir — bir cloud engineer'in günlük en çok kullandığı araç. (Ağ protokollerinin derinliği bu haritanın kapsamı dışı; burada "makinede ağ nasıl görünür/yönetilir"e odaklan.)

### Konular

**7.1 Arayüz ve adres yönetimi**
- `ip addr`, `ip route` ile makinenin ağ durumunu okumak `[uygulama]`
- netplan (Ubuntu'da ağ config'i) — kavram düzeyi `[kavram]`

**7.2 İsim çözümleme (host tarafı)**
- `/etc/hosts`, `/etc/resolv.conf`, systemd-resolved `[mekanizma]`
- **Bozulunca:** `resolv.conf` bozuk → "ping IP çalışıyor ama isim çözülmüyor"

**7.3 SSH — derinlemesine**
- Key tabanlı kimlik (public/private), `~/.ssh/config`, ssh-agent `[uygulama]`
- Port forwarding / tünelleme (yerel port → uzak servis) `[kavram]`
- Sıkılaştırma: root login kapalı, password kapalı, key-only `[uygulama]`
- **Cloud bağlantısı:** EC2'ye SSH, keypair yönetimi; SSM Session Manager (SSH açmadan erişim) — neden daha güvenli

**7.4 Port ve soket durumu**
- `ss -tulpn` — hangi servis hangi portu dinliyor `[uygulama]`
- **Bozulunca:** "servis ayakta ama bağlanılamıyor" → dinleniyor mu (`ss`), 0.0.0.0 mı 127.0.0.1 mi bind edilmiş

**7.5 Host firewall (farkındalık)**
- ufw / firewalld / nftables — OS seviyesi paket filtresi `[kavram]`
- **Cloud bağlantısı:** Security Group (bulut tarafı) ile host firewall iki ayrı katman; ikisi birden trafiği kesebilir `[kavram]`

> **Lab 7:** `ip addr` ve `ip route` ile ağ durumunu oku. `ss -tulpn` ile dinlenen portları gör. `resolv.conf`'una bakıp DNS sunucunu bul. `ssh -v` ile bir bağlantının hangi adımda takıldığını izle.

---

## Faz 8 — Paketler, Yazılım ve Servisleştirme

Bu fazın amacı: yazılımın sisteme nasıl girdiği ve **kendi uygulamanı bir servise dönüştürmek** — kendi projelerini Linux'ta düzgün çalıştırmanın yolu.

### Konular

**8.1 Paket yöneticileri**
- apt (Debian/Ubuntu) vs dnf/yum (RHEL/Amazon Linux); alt katman `dpkg`/`rpm` `[mekanizma]`
- Repository, GPG imzası, sürüm sabitleme (pinning) `[kavram]`
- **Bozulunca:** kırık bağımlılık, repo ulaşılamıyor, sürüm sürüklenmesi

**8.2 Kendi uygulamanı servisleştirmek**
- Bir FastAPI/Python uygulamasını systemd service olarak yazmak (unit dosyası, auto-restart, log) `[uygulama]`
- Neden `nohup python app.py &` üretime uygun değil `[uygulama]`

**8.3 Kaynaktan derleme (farkındalık)**
- `./configure && make && make install` mantığı — ne zaman gerekir `[atla]`

**8.4 Immutable yaklaşım**
- **Cloud bağlantısı:** paketleri baked-in bir AMI, sürüm pinning ile tekrarlanabilirlik; "sunucuyu elle yamama, yeniden inşa et" felsefesi `[kavram]`

> **Lab 8:** `apt list --installed | wc -l` ile kaç paket kurulu gör. Basit bir `hello.service` unit dosyası yazıp `systemctl --user` ile çalıştır (veya birlikte bir FastAPI servisi taslağı çıkar).

---

## Faz 9 — Güvenlik ve Sertleştirme (Hardening)

Bu fazın amacı: paketin/erişimin **bilerek kısıtlanması**. Daha önce bir hardening deneyimin olduysa burada mantığını sistematize edip cloud'a taşırsın; olmadıysa temelini burada kurarız.

### Konular

**9.1 En az yetki ilkesi**
- Her servis/kullanıcı sadece gereken yetkiye sahip olmalı `[kavram]`

**9.2 SSH ve ağ yüzeyi sıkılaştırma**
- Key-only, root kapalı, gereksiz portlar kapalı, firewall `[uygulama]`

**9.3 MAC — Mandatory Access Control**
- AppArmor (Ubuntu) vs SELinux (RHEL): DAC izinlerinin üstünde ikinci bir kilit `[mekanizma]`
- **Bozulunca:** "izinler doğru ama yine engelleniyor" → AppArmor/SELinux profili

**9.4 Denetim ve bütünlük**
- auditd (denetim günlüğü), rkhunter (rootkit tarama), dosya bütünlüğü `[kavram]`
- **Bozulunca:** auditd log patlaması → journald'ı boğar → sistem donar (gerçek yaşanmış senaryo); rate-limit/rotation gereği `[uygulama]`

**9.5 Capabilities**
- root/non-root ikiliğini kıran ince yetkiler (ör. yalnız port-bind yetkisi) `[kavram]`

**9.6 Sırların diskte yönetimi**
- Ne YAPILMAMALI: koda/gömülü dosyaya API key `[uygulama]`
- **Cloud bağlantısı:** IAM instance role (diskte key tutmadan yetki), SSM Parameter Store / Secrets Manager, CIS benchmark'lı hardened AMI `[uygulama]`

> **Lab 9:** `sudo aa-status` ile AppArmor profillerini gör. `ss -tulpn` ile dışa açık portları denetle. `sudo systemctl status auditd` ile denetim servisini kontrol et.

---

## ✅ Ara Sınav 4 (Faz 7–9)

Ağ (OS) + paket + güvenlik. Sunucunun dış dünyayla ilişkisi ve savunması burada tamamlanır.

---

## Faz 10 — Otomasyon ve Scripting: Elle Yapmayı Bırakmak

Bu fazın amacı: tekrarlayan işi güvenilir script'e çevirmek ve buradan IaC (kod olarak altyapı) zihniyetine köprü kurmak. Bir cloud engineer elle sunucu yamamaz — üretir.

### Konular

**10.1 Bash scripting temeli**
- Değişken, koşul, döngü, fonksiyon, exit code (`$?`) `[uygulama]`
- Quoting neden hayat kurtarır (`"$var"`) `[mekanizma]`

**10.2 Sağlam script yazımı**
- `set -euo pipefail` — hatada erken dur `[uygulama]`
- `trap` ile temizlik; loglama `[kavram]`
- **Bozulunca:** tırnaksız değişken + boşluklu yol → felaket; `rm -rf $DIR/` (tırnaksız) — `$DIR` boşsa ya da boşluk içeriyorsa

**10.3 Bash nerede biter, Python nerede başlar**
- ~20 satırı geçen mantık, JSON/HTTP işleri → Python'a geç `[kavram]`

**10.4 Idempotency ve IaC köprüsü**
- Idempotent (aynı script iki kez çalışınca bozmayan) olmak neden şart `[kavram]`
- **Cloud bağlantısı:** user-data script'i, cloud-init; buradan Ansible/Terraform'a giden yol; "mutable sunucu yamama" yerine "immutable yeniden inşa" `[uygulama]`

> **Lab 10:** Kısa bir yedekleme/temizlik script'i yaz: `set -euo pipefail` ile başlat, bir dizindeki 7 günden eski logları bul-sil, exit code'u kontrol et. İki kez çalıştır — idempotent mi?

---

## Faz 11 — Gözlemlenebilirlik ve Troubleshooting: Adli Refleks

Bu fazın amacı: önceki tüm "bozulunca" notlarını **tek bir sistematik reflekse** dönüştürmek. Gerçek cloud engineer farkı burada ortaya çıkar — "bir şey bozulunca kanıt nerede?"

### Konular

**11.1 Loglar — ilk durak**
- `journalctl` (systemd), `/var/log/` (syslog, auth, dmesg), logrotate `[uygulama]`
- Hangi log neyi anlatır: `auth.log` (giriş), `syslog` (genel), `dmesg` (kernel) `[mekanizma]`

**11.2 Katman-katman debugging metodolojisi**
- Yukarıdan aşağı ele: uygulama logu → servis durumu → kaynak (CPU/RAM/IO) → ağ (port/DNS) → çekirdek (dmesg) `[mekanizma]`
- Her katmanı bir önceki doğrulanmadan atlamama disiplini `[kavram]`

**11.3 Araç ustalığı (her araç hangi gerçeğe bakar)**
- `top`/`htop` → canlı kaynak `[uygulama]`
- `ps` → process anlık görüntü `[uygulama]`
- `ss` → soket/port durumu `[uygulama]`
- `lsof` → açık dosyalar ve file descriptor'lar `[uygulama]`
- `strace` (es-treys → bir process'in yaptığı syscall'lar; **nihai hakem**, tcpdump'ın process karşılığı) `[uygulama]`
- `dmesg` → çekirdek halka tamponu (OOM, disk hatası, donanım) `[uygulama]`
- `journalctl` → systemd logları `[uygulama]`
- `iostat`/`vmstat`/`free` → kaynak zaman serisi `[uygulama]`
- `/proc/<pid>/` → bir process'in içi `[uygulama]`

**11.4 Üç içgüdü sorusunun cevap haritası**
- "Sistem ne yapıyor" → `top` + `ps` + `/proc` birleşik filmi
- "Neden erişilemiyor" → izin biti / ownership / mount / MAC / capability karar ağacı
- "Boot'tan servise" → `systemctl status` + `journalctl -b` + cloud-init log kontrol listesi

**11.5 Cloud'da kanıt**
- **Cloud bağlantısı:** yerelde journald/strace neyse, cloud'da CloudWatch Logs + SSM o; merkezi loglama; "instance sağlıklı ama uygulama değil" ayrımı `[uygulama]`

> **Bitirme Lab'ı:** Kasıtlı bir arıza kur (bir servisi bozuk config'le başlatmayı dene, ya da bir portu yanlış bind et), sonra sadece metodolojiyle — log → durum → kaynak → ağ sırasıyla — 5 dakikada bul. Bu, içgüdünün gerçek testi.

---

## Faz 12 — Cloud'a Köprü (Linux in AWS)

Bu fazın amacı: her Linux temelini AWS'teki karşılığına oturtmak. Cloud Engineering hedefine doğrudan çıkış.

### Konular

- **AMI** = donmuş bir Linux (Faz 0 + Faz 8 paket katmanı) `[uygulama]`
- **cloud-init / user-data** = boot-time yapılandırma (Faz 5 + Faz 10) `[uygulama]`
- **SSH keypair / SSM Session Manager** = erişim (Faz 7) `[uygulama]`
- **IAM instance role** = diskte key tutmadan yetki (Faz 9) `[uygulama]`
- **EBS + fstab/UUID** = kalıcı depolama akışı (Faz 6) `[uygulama]`
- **systemd service** = uygulamanın yaşam döngüsü (Faz 5 + Faz 8) `[uygulama]`
- **CloudWatch agent / Logs** = uzaktan gözlemlenebilirlik (Faz 4 + Faz 11) `[kavram]`
- **cgroup + namespace → ECS/EKS container** = kernel paylaşan Linux (Faz 3.6) `[kavram]`
- **Lambda** = görmediğin ama yine Linux olan runtime `[kavram]`
- **Security Group vs host firewall / AppArmor** = iki ayrı savunma katmanı (Faz 7 + Faz 9) `[kavram]`

> **Final çıktısı:** Boş bir Ubuntu AMI'yi al — onun boot'tan production servise dönüşen yolculuğunu (cloud-init → kullanıcı/SSH → EBS mount → paket → systemd servis → log/monitoring → hardening) her adımın **neden** orada olduğunu Linux temellerine dayanarak anlatabilmelisin.

---

## Toplam yapı

| Bölüm | Fazlar | Odak |
|---|---|---|
| Temel | 0–2 | Model, shell, izinler — "sistemi okumak" |
| Çalışan sistem | 3–4 | Process, bellek, IO — "kaynağı ne yiyor" |
| Yaşam döngüsü | 5–6 | Boot, systemd, depolama — "boot'tan servise" |
| Dış dünya + savunma | 7–9 | Ağ (OS), paket, hardening |
| Usta | 10–12 | Otomasyon + troubleshooting içgüdüsü + cloud köprüsü |

**Sabit kurallar (diğer haritalarla aynı):** Türkçe; teknik terimler İngilizce kalır, ilk geçişte parantez içinde Türkçe telaffuz + kısa tanım; tek konu, tek soru; direkt cevap yok (yapıcı Socratic); konu atlama yok; yanlışta yönlendiren değil düzelten müdahale; `[uygulama]` = makinede gör; her fazda "Bozulunca" açısı; bu harita kernel geliştiricisi değil, kararını gerekçelendiren ve arızayı avlayan cloud engineer yetiştirir.

---

*Hazırlayan: Denis Ergöçmen, Ağustos 2026 — genel kullanım için nötrleştirilmiş sürüm*
