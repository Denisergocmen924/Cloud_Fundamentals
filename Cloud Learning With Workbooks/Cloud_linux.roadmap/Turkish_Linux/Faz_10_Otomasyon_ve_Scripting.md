# Faz 10 — Otomasyon ve Scripting: Elle Yapmayı Bırakmak

> **Navigasyon:** [◀ Ara Sınav 4](Ara_Sinav_4.md) · **Faz 10** · [Faz 11 — Gözlemlenebilirlik ve Troubleshooting ▶](Faz_11_Gozlemlenebilirlik_ve_Troubleshooting.md)

---

## Nereden geliyoruz

Faz 0'dan 9'a kadar hep **elle** yaptın: shell'de komut yazdın (Faz 1), izin düzelttin (Faz 2), process
başlattın (Faz 3), disk bağladın (Faz 6), servis yazdın (Faz 8), sistemi sertleştirdin (Faz 9). Her biri tek
bir makinede, tek seferlik, senin klavyende. Faz 10 bu alışkanlığı kırıyor. Çünkü bir cloud engineer'ın gerçek
işi tek makine değildir — onlarca, yüzlerce, otomatik oluşan ve yok olan makinedir. Elle yapılan hiçbir şey bu
ölçekte tekrarlanamaz, denetlenemez, güvenilemez.

Üç fazın dersi burada bir zihniyete dönüşüyor:

- **Faz 5 — cloud-init.** İlk boot'ta çalışan yapılandırmayı gördün. cloud-init aslında bir **script'tir**;
  Faz 10 o script'i doğru yazmayı öğretir.
- **Faz 8 — immutable / baked AMI.** "Makineyi yamama, yeniden inşa et" fikrini gördün. Faz 10 o "yeniden
  inşa"nın nasıl **kodla** yapıldığını verir.
- **Faz 9 — sertleştirme adımları.** Elle yaptığın her hardening adımı, aslında tekrarlanabilir bir kod satırı
  olmalıydı. Faz 10 bunu sistematize eder.

## Bu fazın sorusu

Faz 9'a kadar "bir makineyi nasıl kurar, çalıştırır ve savunurum" dedik. Faz 10 tam tersini sorar:

> *"Elle yaptığım her şeyi — kurulum, yapılandırma, sertleştirme — nasıl tekrarlanabilir, güvenilir, iki kez
> çalıştığında bozmayan bir koda çeviririm; ve bu, 'sunucu yamama'dan 'altyapıyı kod olarak üretme'ye giden
> yolun neresindedir?"*

Bu fazın merkezinde iki fikir vardır: **sağlamlık** (bir script sessizce yanlış iş yapmamalı — hatada erken
durmalı) ve **idempotency** (aynı script iki kez çalıştığında sistemi bozmamalı, aynı sonuca ulaşmalı). Bu iki
fikir seni Bash'ten IaC'ye (Infrastructure as Code — kod olarak altyapı) taşıyan köprüdür. Bir cloud engineer
elle sunucu yamamaz — **üretir**.

Bu fazın sonunda, tekrarlayan bir işi güvenilir bir Bash script'ine çevirebilecek; bir script'in neden
sessizce felaket getirebileceğini (tırnaksız değişken, kontrol edilmeyen exit code) anlatabilecek; ve
idempotency ile cloud-init/user-data → Ansible/Terraform köprüsünü kurabileceksin.

---

## Bu fazın sonunda

- **Bash scripting temelini** (değişken, koşul, döngü, fonksiyon, exit code `$?`) yazabilecek ve quoting'in
  (`"$var"`) neden hayat kurtardığını mekanizma düzeyinde açıklayabileceksin
- **Sağlam script** yazabileceksin: `set -euo pipefail` ile hatada erken durma, `trap` ile temizlik, loglama
- Tırnaksız değişken + boşluklu/boş yol felaketini (`$DIR` boşken `rm -rf $DIR/`) tanıyabilecek ve
  önleyebileceksin
- **Bash'in nerede bitip Python'un nerede başladığını** karar verebileceksin (~20 satırı geçen mantık,
  JSON/HTTP işleri → Python)
- **Idempotency**'i açıklayabilecek ve bir script'i idempotent yazabileceksin
- **Cloud:** user-data script'i / cloud-init'in bu fazla ilişkisini; buradan Ansible/Terraform'a giden yolu; ve
  "mutable sunucu yamama" yerine "immutable yeniden inşa" felsefesini anlatabileceksin

---

## Faz haritası

| Bölüm | Konu | Derinlik | Neden burada |
|---|---|---|---|
| 10.1 | Bash scripting temeli | `[uygulama]` | Otomasyonun alfabesi: değişken, koşul, döngü, exit code |
| 10.2 | Sağlam script yazımı | `[uygulama]` | `set -euo pipefail`, `trap`, quoting — sessiz felaketi önle |
| 10.3 | Bash nerede biter, Python nerede başlar | `[kavram]` | Doğru aracı seçmek: ~20 satır / JSON-HTTP eşiği |
| 10.4 | Idempotency ve IaC köprüsü | `[kavram]` | Aynı script iki kez → bozmamalı; cloud-init → Ansible/Terraform |
| 10.5 | Bu faz bozulunca | — | Otomasyon/script arıza imzaları |

> **Bu fazda nasıl çalışmalı:** Bu fazın komutları çoğunlukla 🟢/🟡'dır — bir script yazıp çalıştırmak. Ama
> bir script **kör bir güçtür**: yazdığın hata, elle yaptığından **çok daha hızlı ve çok daha geniş** zarar
> verir (bir `rm -rf` döngüsü saniyeler içinde tüm bir dizini siler). Bu yüzden bu fazın en kritik alışkanlığı:
> her yıkıcı komutu (🔴) önce `echo` ile "dry-run" yap (gerçekte silmeden ne sileceğini yazdır), sonra
> gerçeğini çalıştır. Script'lerini kendi test makinende, önemsiz dizinlerde dene. En öğretici deney:
> `set -euo pipefail`'siz bir script yazıp bir komutunu kasten hatalı yap — script'in hatayı **görmezden gelip
> devam ettiğini** gör; sonra `set -e` ekleyip aynı şeyi yap, farkı yaşa.

---
---

# 10.1 Bash Scripting Temeli

## 10.1.1 Komut dizisinden programa: değişken, koşul, döngü `[uygulama]`

Faz 1'de shell'e tek tek komut yazdın. Bir **script**, o komutları bir dosyaya koyup bir program gibi
çalıştırmaktır. Temel yapı taşları:

```bash
#!/usr/bin/env bash          # shebang: bunu hangi yorumlayıcı çalıştırsın
name="web-01"                # değişken (= etrafında BOŞLUK YOK)
count=3

if [ "$count" -gt 0 ]; then  # koşul
  echo "$name has $count items"
fi

for i in 1 2 3; do           # döngü
  echo "iteration $i"
done

backup() {                   # fonksiyon
  local src="$1"             # ilk argüman; local = fonksiyona özel
  echo "backing up $src"
}
backup /etc
```

İki başlangıç tuzağı: (1) atamada `=` etrafında boşluk olamaz (`name = "x"` hatadır); (2) değişkeni **kullanırken**
`$` ile çağırırsın (`$name`), atarken çağırmazsın.

## 10.1.2 Exit code: her komutun sessiz sinyali `[uygulama]`

Faz 3'te her process'in bir çıkış kodu (exit code) döndürdüğünü ima etmiştik. Script otomasyonunun **kalbi**
budur: her komut biter bitmez `0` (başarı) ya da sıfır-olmayan (hata) bir kod döndürür ve bunu `$?` ile
okursun.

```bash
$ ls /var/log > /dev/null
$ echo $?
0                            # başarı
$ ls /yok-boyle-bir-yer 2>/dev/null
$ echo $?
2                            # hata (sıfır değil)
```

Bu neden kritik? Çünkü bir script başarıyı/başarısızlığı **görmezden gelirse** sessizce yanlış iş yapar:
"yedek aldım" der ama komut hata vermiştir; "servisi durdurdum, şimdi dosyayı siliyorum" der ama durdurma
başarısız olmuştur. Exit code, script'in "gerçekten oldu mu?" sorusuna verdiği tek cevaptır — ve 10.2'de bunu
otomatik bir güvenlik ağına (`set -e`) çevireceğiz.

> **🔧 Makinende gör** 🟢 — exit code zincirini izle
>
> ```
> $ true;  echo $?          # 0
> $ false; echo $?          # 1
> $ grep -q root /etc/passwd && echo "var" || echo "yok"   # exit code ile dallanma
> ```
>
> `&&` (öncekiler başarılıysa çalıştır) ve `||` (öncekiler başarısızsa çalıştır) doğrudan exit code'a bakar.
> Script'lerin akışı bu iki operatör ve `if` üzerine kuruludur — hepsi `$?`'yi okur.

## 10.1.3 Quoting neden hayat kurtarır `[mekanizma]`

Bu, Bash'in en çok can yakan mekanizmasıdır. Bir değişkeni tırnaksız kullandığında (`$var`), Bash onun
değerini **önce kelimelere böler** (word splitting) ve **glob genişletir** (`*` gibi karakterleri dosya
adlarına çevirir). Tırnak içinde (`"$var"`) ise değer olduğu gibi, tek parça kalır.

```bash
file="my report.txt"        # içinde boşluk var
rm $file                    # ❌ Bash bunu "rm my report.txt" görür → İKİ dosya siler: 'my' ve 'report.txt'
rm "$file"                  # ✅ tek dosya: 'my report.txt'
```

Kural basit ve mutlaktır: **her değişken genişletmesini çift tırnak içine al** (`"$var"`, `"$@"`,
`"${arr[@]}"`) — istisnai olarak bölmeyi bilerek istediğin durumlar dışında. Bu tek alışkanlık, script
felaketlerinin büyük kısmını daha doğmadan önler (10.2.3'te bunun `rm -rf` versiyonunu göreceğiz).

> **⚠️ Yaygın yanılgı: "Değişkenimde boşluk yok, tırnağa gerek yok."**
>
> Bugün yok — ama script'in yarın başka bir girdiyle, başka bir kullanıcının dosya adıyla, ya da boş bir
> değişkenle çalışacak. Quoting bir "şimdiki değere" göre değil, "her olası değere" göre yazılır. Boş bir
> değişken (`var=""` iken `$var`) tırnaksızsa **hiç argüman** olur ve komutun anlamını değiştirir; tırnaklıysa
> **boş bir argüman** olur. İkisi çok farklıdır ve fark tam da felaketlerin çıktığı yerdir. Refleks: istisnasız
> tırnakla.

> **🤔 Düşün 10.1** — Bir script'in şu satırı var: `cp $SRC $DST`. `SRC="/tmp/a b.txt"` (boşluklu) ve
> `DST="/backup/"`. (a) Bu satır tırnaksız çalışınca `cp` kaç argüman görür ve ne olur? (b) `SRC` boş bir
> değişkenken (`SRC=""`) tırnaksız `cp $SRC $DST` ne yapar — `cp "$SRC" "$DST"` ile farkı nedir? (c) Kuralı tek
> cümlede yaz.
>
> *(Cevap: fazın sonunda)*

---
---

# 10.2 Sağlam Script Yazımı

## 10.2.1 `set -euo pipefail`: hatada erken dur `[uygulama]`

Varsayılan olarak Bash **affedicidir** — bir komut hata verse bile script bir sonraki satıra devam eder. Bu,
otomasyonda felakettir: "servisi durdur → dosyayı sil" script'inde durdurma başarısız olsa bile silme çalışır.
Çözüm, her sağlam script'in ilk satırıdır:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

Üç ayrı koruma:

- **`-e`** (errexit): bir komut sıfır-olmayan kod döndürünce script **hemen durur**. Sessiz devam yok.
- **`-u`** (nounset): tanımsız bir değişken kullanılınca hata ver. `rm -rf "$DIR/"` içindeki `$DIR` yazım
  hatasıyla tanımsızsa, `-u` script'i durdurur — köke `rm -rf /` çalıştırmaz.
- **`-o pipefail`**: bir pipe'ın (`a | b`) herhangi bir aşaması hata verirse tüm pipe hata sayılır. Varsayılan
  olarak yalnızca **son** komutun kodu görülür; bu, `curl ... | tar ...`'da `curl` çöktüğü hâlde başarı sanmaya
  yol açar.

Bu tek satır, script'in "affedici"den "titiz"e dönüşmesidir — sessizce yanlış iş yapmak yerine, ilk hatada
durup sana haber verir.

## 10.2.2 `trap` ile temizlik ve loglama `[kavram]`

Bir script yarıda hata verirse (ya da kullanıcı Ctrl-C yaparsa) geride yarım iş bırakabilir: bir geçici dizin,
bir kilit dosyası, mount edilmiş bir disk. `trap`, "script hangi sebeple biterse bitsin şu temizliği yap"
demenin yoludur:

```bash
tmpdir=$(mktemp -d)
trap 'rm -rf "$tmpdir"' EXIT   # script nasıl biterse bitsin geçici dizini sil
```

`EXIT` sinyali script normal bittiğinde de, hata ile durduğunda da (`set -e`) tetiklenir — böylece temizlik
garanti olur. Loglama da sağlamlığın parçasıdır: script'in ne yaptığını `echo "[$(date)] adım X"` gibi
satırlarla veya `logger` ile journald'a (Faz 5!) yazmak, sonradan "ne oldu" sorusunu cevaplar.

## 10.2.3 Tırnaksız değişken + boşluklu yol = felaket `[uygulama]`

İşte bu fazın en pahalı arızası, gerçek dünyadan sayısız kez yaşanmıştır. Bir temizlik script'i:

```bash
DIR="/var/tmp/cache"
rm -rf $DIR/                  # ❌ tırnaksız
```

Üç ayrı felaket senaryosu:

1. **`DIR` boş veya tanımsız** (yazım hatası, atanmamış değişken): `rm -rf /` olur — **tüm sistemi siler.**
   `set -u` yalnızca *tanımsız* durumu yakalar (isimdeki yazım hatası); `""` atanmış değişken kaçar, bu
   yüzden `[ -n "$DIR" ]` (veya `${DIR:?}`) ekle. Tırnak + `set -u` + bu kontrol birlikte hayat kurtarır.
2. **`DIR` boşluk içeriyor** (`/var/tmp/my cache`): `rm -rf /var/tmp/my cache/` → Bash iki argüman görür:
   `/var/tmp/my` ve `cache/` — yanlış dizinleri siler.
3. **`DIR` glob içeriyor** ya da genişliyor: beklenmedik dosyalar eşleşir.

Doğru hâli her üçünü de kapatır:

```bash
set -euo pipefail
DIR="/var/tmp/cache"
[ -n "$DIR" ] || { echo "DIR boş, iptal"; exit 1; }   # ekstra güvenlik: boşsa dur
rm -rf "$DIR"/                # ✅ tırnaklı
```

> **🔧 Makinende gör** 🟡 — yıkıcı komuttan önce dry-run
>
> ```
> $ DIR="/var/tmp/test space"
> $ printf '[%s]\n' rm -rf $DIR/     # ❌ tırnaksız: argüman başına bir satır
> [rm]
> [-rf]
> [/var/tmp/test]
> [space/]
> $ printf '[%s]\n' rm -rf "$DIR"/   # ✅ tırnaklı: yol tek argüman kalır
> [rm]
> [-rf]
> [/var/tmp/test space/]
> ```
>
> Komutun argümanlarının önüne `printf '[%s]\n'` koymak (düz `echo` iki hâli de aynı basardı), hiçbir şey
> silmeden gerçekte **ne** geçirileceğini gösterir. Bu "dry-run" alışkanlığı, bu fazın tek en değerli
> güvenlik refleksidir. Tırnaksız versiyonun iki argümana bölündüğünü kendi gözünle gör.

> **🤔 Düşün 10.2** — Bir meslektaşın script'i `set -e` **olmadan** şöyle: `cd "$WORKDIR"` sonra
> `rm -rf ./*`. `WORKDIR` var olmayan bir dizine işaret ediyor (silinmiş). (a) `cd` başarısız olunca `set -e`
> yoksa script ne yapar, ve `rm -rf ./*` **hangi dizinde** çalışır? (b) Bu neden felakettir? (c) `set -euo
> pipefail` ve `cd "$WORKDIR" || exit 1` bunu nasıl önlerdi?
>
> *(Cevap: fazın sonunda)*

---
---

# 10.3 Bash Nerede Biter, Python Nerede Başlar

## 10.3.1 Doğru aracı seçmek `[kavram]`

Bash harika bir **yapıştırıcıdır**: komutları zincirlemek, dosya taşımak, servis başlatmak için mükemmel. Ama
bir eşikten sonra Bash bir **yük** hâline gelir. Genel kurallar:

- **~20 satırı geçen mantık.** Karmaşık koşullar, iç içe döngüler, veri yapıları gerektiğinde Bash okunmaz ve
  hataya açık olur. Python'un net sözdizimi ve gerçek veri yapıları (liste, sözlük) devreye girer.
- **JSON / yapılandırılmış veri.** Bash JSON'u doğru ayrıştıramaz (`grep`/`sed` ile "ayrıştırmak" kırılgandır);
  `jq` yardımcı olur ama karmaşık dönüşümde Python'un `json` modülü kıyaslanamaz.
- **HTTP / API işleri.** Birkaç `curl` çağrısı Bash'te olur; ama API sayfalama, yeniden deneme, hata yönetimi
  gerektiğinde Python'un `requests`'i doğru araçtır.
- **Test edilebilirlik.** Ciddi mantık test istiyorsa, Python'un test ekosistemi Bash'ten çok ilerdedir.

Refleks: "Bu bir dosya/komut yapıştırma işi mi, yoksa gerçek bir program mı?" İlkiyse Bash, ikinciyse Python.
Yanlış aracı zorlamak (500 satırlık Bash, ya da tek `mv` için Python) her iki yönde de hatadır.

> **💡 Cloud bağlantısı — nerede hangi dil:** Bir EC2 user-data'sında (Faz 5) genellikle kısa bir **Bash**
> script'i vardır: paket kur, servisi başlat, config indir. Ama uygulamanın asıl mantığı (bir Lambda, bir veri
> işleme adımı, bir deploy aracı) neredeyse her zaman **Python**'dur. İkisi bir arada yaşar: Bash makineyi
> ayağa kaldırır, Python işi yapar. Doğru sınırı çizmek, ölçeklenen ve bakımı yapılabilen otomasyonun
> temelidir.

---
---

# 10.4 Idempotency ve IaC Köprüsü

## 10.4.1 Idempotency: aynı script iki kez çalışınca bozmamalı `[kavram]`

**Idempotent** bir işlem, bir kez de çalışsa on kez de çalışsa sistemi **aynı** son duruma getirir — ve ikinci
çalıştırma bir şeyi bozmaz. Bu, otomasyonun en temel ama en çok atlanan özelliğidir. Örnek:

```bash
# ❌ idempotent DEĞİL: ikinci çalıştırma hata verir / satırı iki kez ekler
mkdir /opt/app                          # dizin varsa ikinci kez hata
echo "export PATH=..." >> ~/.bashrc     # her çalıştırmada satırı TEKRAR ekler

# ✅ idempotent: kaç kez çalışırsa çalışsın aynı sonuç
mkdir -p /opt/app                       # varsa sorun etmez
grep -qxF "export PATH=..." ~/.bashrc || echo "export PATH=..." >> ~/.bashrc  # yoksa ekle
```

Neden şart? Çünkü gerçek otomasyon **tekrar çalışır**: cloud-init bazen yeniden koşar, bir deploy iki kez
tetiklenir, bir yapılandırma aracı her dakika "istenen durumu" uygular. Idempotent değilse, her tekrar sistemi
biraz daha bozar — çift eklenmiş satırlar, hata veren adımlar, tutarsız durum. Idempotency, "istenen durumu
tarif et, oraya güvenle tekrar tekrar götür" demektir.

![Şekil 10.1 — Idempotency: aynı script iki kez çalıştığında ne olur. Solda idempotent-olmayan script (mkdir /opt/app; echo satırı >> dosya): ilk çalıştırma dizini oluşturur ve satırı ekler; ikinci çalıştırma "dizin var" hatası verir ve aynı satırı bir daha ekleyerek dosyayı bozar — her tekrar durumu kötüleştirir. Sağda idempotent script (mkdir -p; grep -qxF || echo): ilk çalıştırma istenen duruma getirir; ikinci çalıştırma hiçbir şeyi değiştirmez, aynı son durum korunur. Alttaki ders: idempotency "istenen durumu tarif et, oraya güvenle tekrar tekrar götür" demektir — IaC'nin temelidir.](../diagrams/png/lx-10-01-idempotency.png)

## 10.4.2 Bash'ten IaC'ye köprü `[uygulama]`

Idempotency fikri seni doğrudan **IaC**'ye (Infrastructure as Code) taşır. Yolculuk şöyle ilerler:

1. **Elle** yaparsın (Faz 0-9) — tek makine, tekrarlanamaz.
2. **Bash script'i** yazarsın — tekrarlanabilir ama kırılgan, idempotency'i sen sağlarsın.
3. **user-data / cloud-init** (Faz 5) — o script'i boot anında otomatik çalıştırırsın; makine kendi kendini
   kurar.
4. **Ansible** — idempotency'i **kendisi** garanti eden bir araç; "istenen durumu" tarif edersin (paket kurulu
   olsun, servis çalışsın), Ansible sistemi oraya götürür ve tekrar çalışınca bozmaz.
5. **Terraform** — makinenin **kendisini** (ve tüm bulut kaynaklarını) kod olarak tanımlarsın; altyapı artık
   bir metin dosyasıdır, sürüm kontrolündedir, gözden geçirilir.

Bu köprünün altındaki felsefe Faz 8'in immutable fikridir: **"mutable sunucu yamama" yerine "immutable yeniden
inşa".** Bir sunucuda bir şey bozulunca onu elle tamir etmezsin (mutable patching — durumu bilinmez hâle
getirir); onu **yok edip koddan yeniden inşa edersin** (immutable rebuild — durum her zaman koda eşittir). Bu
yüzden bu faz, tek bir script yazmayı değil, **tüm altyapıyı tekrarlanabilir kod olarak düşünmeyi** öğretir.

> **💡 Cloud bağlantısı — user-data'dan Terraform'a:** Bir EC2 instance'ının user-data'sı (Faz 5) çoğu zaman
> senin ilk "IaC" adımındır: boot'ta çalışan bir Bash script'i. Ama ölçek büyüdükçe bu yetmez — 50 makineyi
> tutarlı tutmak, değişiklikleri gözden geçirmek, geri almak istersin. İşte o zaman **Ansible** (makinenin
> içini yapılandırır) ve **Terraform** (makinenin kendisini ve ağını/SG'yi/IAM'i üretir) devreye girer. Ama
> hepsinin altında bu fazın iki dersi yatar: sağlamlık (sessizce bozma) ve idempotency (tekrar çalışınca aynı
> sonuç). Bir aracı öğrenmeden önce bu iki fikri anlamak, hangi aracı kullanırsan kullan işe yarar.

> **🤔 Düşün 10.3** — Bir cloud-init user-data script'in bir paket kuruyor ve `/etc/app.conf`'a bir yapılandırma
> satırı ekliyor. Bir gün instance yeniden boot ediyor ve cloud-init'in bir kısmı yeniden çalışıyor. (a)
> Script idempotent değilse (`>>` ile satır ekliyor) config dosyasına ne olur? (b) Bunu idempotent yapmak için
> hangi tek deseni kullanırsın? (c) Bu neden "immutable yeniden inşa" felsefesinin küçük bir örneğidir —
> idempotency ile immutability nasıl aynı hedefe (durum = kod) hizmet eder?
>
> *(Cevap: fazın sonunda)*

---
---

# 10.5 Bu Faz Bozulunca — Otomasyon/Script Arıza İmzaları

Bu fazın arızaları sinsidir: bir script "çalışıyor gibi görünür" ama sessizce yanlış iş yapar. Tablo belirtiyi
sebebe bağlar:

| Belirti | Muhtemel sebep | Bakılacak yer | İlgili bölüm |
|---|---|---|---|
| Script hata veren komuttan sonra devam etti, yanlış sonuç | `set -e` yok | İlk satıra `set -euo pipefail` ekle | 10.2.1 |
| `rm`/`cp` yanlış dosyaları işledi (boşluklu yol) | Tırnaksız değişken | Tüm `$var` → `"$var"`; `echo` ile dry-run | 10.1.3, 10.2.3 |
| `rm -rf /` benzeri felaket | Boş/tanımsız değişken tırnaksız | `set -u` + `[ -n "$DIR" ]` kontrolü | 10.2.3 |
| `curl ... \| tar ...` başarı sandı ama curl çöktü | `pipefail` yok | `set -o pipefail` | 10.2.1 |
| Script ikinci çalıştırmada hata verdi / bozdu | Idempotent değil | `mkdir -p`, `grep -qxF \|\|`, koşullu ekleme | 10.4.1 |
| Config satırı her boot'ta tekrar eklendi | cloud-init idempotent değil | Ekleme öncesi varlık kontrolü | 10.4.1 |
| Yarım kalan geçici dizin/kilit dosyası birikti | `trap` ile temizlik yok | `trap '...' EXIT` | 10.2.2 |
| 400 satırlık Bash bakımı imkânsız | Yanlış araç | JSON/HTTP/karmaşık mantık → Python | 10.3.1 |

> **Bu tablodan çıkan ders:** Bu fazın iki büyük fikri her satırın altında yatar. Birincisi **sessiz
> başarısızlık düşmandır**: bir script'in en tehlikeli hâli çökmesi değil, hata verip **devam etmesidir** —
> `set -euo pipefail` ve quoting tam bunu önler, script'i "affedici"den "titiz"e çevirir. İkincisi
> **tekrarlanabilirlik güvenilirliktir**: idempotent olmayan bir script bir kez işe yarayabilir ama ikinci
> çalıştırmada güvenilmezdir — ve gerçek otomasyon her zaman tekrar çalışır. Ve unutma: bir script kör bir
> güçtür; elle yaptığın bir hata bir dosyayı, script'te aynı hata bir filoyı vurur — bu yüzden yıkıcı komutları
> önce `echo` ile dene.

---
---

# Faz 10 — Düşün sorularının cevapları

## Cevap 10.1 — Tırnaksız değişken kelimelere bölünür; kural: her zaman tırnakla

(a) `cp $SRC $DST` tırnaksız ve `SRC="/tmp/a b.txt"` boşluklu olduğunda Bash word splitting yapar: `cp`
**üç** argüman görür — `/tmp/a`, `b.txt`, `/backup/`. `cp` bunu "iki kaynağı bir hedefe kopyala" sanır,
`/tmp/a` ve `b.txt` diye dosyalar arar (ikisi de yok), hata verir; ya da beklenmedik dosyaları kopyalar.
`cp "$SRC" "$DST"` ise iki argüman görür ve doğru dosyayı kopyalar. (b) `SRC=""` (boş) iken tırnaksız
`cp $SRC $DST`, `SRC` **hiç argüman üretmez** — komut `cp /backup/` olur, tek argümanlı `cp` hata verir (ya da
daha kötüsü, yanlış davranır). `cp "$SRC" "$DST"` ise boş bir argüman geçirir (`cp "" "/backup/"`) — yine hata
ama davranış öngörülebilir (değişken `""` değil de *tanımsız* olsaydı `set -u` yakalardı). (c) Kural: **her değişken genişletmesini istisnasız
çift tırnak içine al** (`"$var"`) — çünkü tırnak değeri "her olası girdide" tek parça tutar; bölmeyi tırnaksız
bırakmak yalnızca bilerek istediğinde yapılır.
**İlgili bölüm:** 10.1.3 · **Devamı:** 10.2.3 (`rm -rf` felaketi).

## Cevap 10.2 — `cd` başarısız olur, `rm -rf ./*` yanlış dizinde çalışır

(a) `set -e` yoksa `cd "$WORKDIR"` başarısız olur (dizin yok) ama script **durmaz, devam eder** — ve mevcut
çalışma dizini değişmediği için `rm -rf ./*` script'in başlatıldığı **o anki dizinde** (ör. kullanıcının
ev dizini ya da bir proje kökü) çalışır. (b) Bu felakettir çünkü hedeflenen dizin yerine tamamen alakasız,
belki kritik bir dizin silinir — `cd`'nin sessiz başarısızlığı yıkıcı komutu yanlış yere yönlendirir. (c)
`set -euo pipefail` ile `cd` başarısız olunca script **hemen durur**, `rm`'e hiç ulaşmaz; ayrıca açıkça
`cd "$WORKDIR" || exit 1` yazmak aynı garantiyi verir. İki katman (fail-fast + açık kontrol) sessiz
başarısızlığı yıkıcı komuttan önce keser.
**İlgili bölüm:** 10.2.1, 10.2.3 · **Devamı:** 10.5 arıza tablosu (satır 1, 3).

## Cevap 10.3 — Config dosyası şişer; koşullu ekleme; idempotency = durum kod'a eşit

(a) Script idempotent değilse ve `>>` ile satır ekliyorsa, cloud-init her yeniden çalıştığında **aynı satırı
bir daha** ekler — config dosyası zamanla aynı satırın kopyalarıyla şişer, hatta çelişkili/çift ayarlar
uygulamayı bozabilir. (b) İdempotent desen: eklemeden önce satırın var olup olmadığını kontrol et —
`grep -qxF "satır" /etc/app.conf || echo "satır" >> /etc/app.conf` (yoksa ekle). (c) Bu "immutable yeniden
inşa"nın küçük örneğidir çünkü ikisi de aynı hedefe hizmet eder: **sistemin durumu her zaman koda eşit
olmalı.** Idempotency, aynı kodu tekrar çalıştırmanın durumu değiştirmemesini sağlar (durum = kodun tarif
ettiği son hâl); immutability ise makineyi yamamak yerine koddan yeniden inşa ederek durumu koda sabitler. İkisi
birlikte "durum sürüklenmesini" (drift) önler — ister aynı script'i tekrar çalıştır, ister makineyi yeniden
kur, sonuç hep aynıdır.
**İlgili bölüm:** 10.4.1-10.4.2 · **Devamı:** 10.5 arıza tablosu (satır 6).

---
---

# Faz 10 — Sık sorulan sorular

**S1 — `set -euo pipefail` tek cümlede ne yapar?** Script'i "affedici"den "titiz"e çevirir: `-e` hatada durur,
`-u` tanımsız değişkende durur, `-o pipefail` pipe'ın herhangi bir aşaması çökünce hata sayar. Sessiz yanlış
işi önler (10.2.1).

**S2 — Neden her değişkeni tırnaklamalıyım?** Tırnaksız değişken word splitting + glob genişlemesine uğrar;
boşluklu, boş veya glob içeren değerlerde komut yanlış argüman görür. `"$var"` değeri her girdide tek parça
tutar (10.1.3).

**S3 — `$?` nedir?** Son çalışan komutun exit code'u: `0` başarı, sıfır-olmayan hata. Script'in "gerçekten oldu
mu" sorusuna cevabı; `&&`, `||`, `if` hepsi buna bakar (10.1.2).

**S4 — Ne zaman Bash'ten Python'a geçmeliyim?** ~20 satırı geçen mantık, JSON/yapılandırılmış veri, ciddi
HTTP/API işi veya test gereksinimi olduğunda. Bash yapıştırıcıdır, Python programdır (10.3.1).

**S5 — Idempotency neden önemli?** Gerçek otomasyon tekrar çalışır (cloud-init, deploy, config aracı). İdempotent
değilse her tekrar durumu bozar (çift satır, hata). İdempotent = kaç kez çalışırsa aynı son durum (10.4.1).

**S6 — `trap` ne işe yarar?** Script nasıl biterse bitsin (normal, hata, Ctrl-C) bir temizlik çalıştırır — ör.
geçici dizini silmek: `trap 'rm -rf "$tmpdir"' EXIT`. Yarım kalan iş bırakmaz (10.2.2).

**S7 — user-data, Ansible ve Terraform arasındaki fark?** user-data boot'ta bir script çalıştırır (makinenin
içi, bir kez); Ansible istenen durumu idempotent uygular (makinenin içi, tekrar tekrar); Terraform makinenin ve
bulut kaynaklarının **kendisini** kod olarak üretir (altyapının tamamı) (10.4.2).

---
---

# Faz 10 — Kendini sına

Cevaplarını bir kâğıda yaz, sonra cevap anahtarıyla karşılaştır. Hedef: 18 sorudan 14+.

## Bölüm A — Tanım ve mekanizma

1. Exit code (`$?`) nedir ve script otomasyonunda neden kalptir? `0` ne demek?
2. Quoting (`"$var"`) tırnaksıza göre ne yapar? Word splitting'i bir örnekle açıkla.
3. `set -e`, `set -u`, `set -o pipefail` ayrı ayrı ne yapar?
4. Idempotency nedir? İdempotent olmayan bir komut ile idempotent karşılığını örnekle.
5. `trap '...' EXIT` ne zaman ve neden çalışır?
6. Bash nerede biter, Python nerede başlar — üç eşik kriteri yaz.
7. "Mutable sunucu yamama" ile "immutable yeniden inşa" arasındaki farkı açıkla.
8. `&&` ve `||` operatörleri exit code'u nasıl kullanır?

## Bölüm B — Uygula ve teşhis et

9. Bir script hata veren bir komuttan sonra devam edip yanlış sonuç üretiyor. İlk satıra ne eklersin?
10. `rm -rf $DIR/` satırındaki iki ayrı felaket senaryosunu (boş `DIR`, boşluklu `DIR`) ve düzeltmesini yaz.
11. Bir cloud-init script'i her boot'ta config dosyasına aynı satırı ekliyor. Sebep ve idempotent düzeltme?
12. Bir `curl ... | tar ...` boru hattı, `curl` çöktüğü hâlde script'i başarılı sanıyor. Hangi ayar eksik?
13. Bir yıkıcı `rm` komutunu çalıştırmadan önce ne çalışacağını nasıl güvenle görürsün?
14. 500 satırlık, JSON ayrıştıran, API çağıran bir Bash script'i devraldın. Ne önerirsin ve neden?

## Bölüm C — Muhakeme ve bağlantı

15. Faz 5'teki cloud-init, bu fazın hangi iki dersini (sağlamlık, idempotency) doğrudan gerektirir? Neden?
16. Faz 8'in immutable/baked AMI felsefesi, bu fazın idempotency fikriyle nasıl aynı hedefe (durum = kod)
    hizmet eder?
17. Faz 9'da elle yaptığın sertleştirme adımları neden aslında kod olmalıydı? IaC bunu nasıl çözer?
18. `set -euo pipefail` + quoting + idempotency üçlüsünü tek cümlede birleştir: güvenilir otomasyonun özü nedir?

---

## Cevap anahtarı

1. Son komutun başarı/başarısızlık sinyali; script "gerçekten oldu mu" sorusunu buna göre yanıtlar; `0` =
   başarı, sıfır-olmayan = hata (10.1.2). — 2. Tırnaksız değişken kelimelere bölünür + glob genişler; `rm $f`
   `f="a b"` iken → iki dosya; `"$var"` değeri tek parça tutar (10.1.3). — 3. `-e` hatada durur, `-u` tanımsız
   değişkende durur, `-o pipefail` pipe'ın herhangi bir aşaması çökünce tüm pipe'ı hata sayar (10.2.1). — 4. Aynı
   işlem kaç kez çalışırsa aynı son durum; `mkdir /x` (ikinci kez hata) vs `mkdir -p /x` (idempotent) (10.4.1).
   — 5. Script nasıl biterse bitsin (normal/hata/sinyal) tetiklenir; garanti temizlik için (geçici dizin/kilit)
   (10.2.2). — 6. ~20 satırı geçen mantık; JSON/yapılandırılmış veri; ciddi HTTP/API veya test gereksinimi →
   Python (10.3.1). — 7. Mutable: bozuk sunucuyu elle tamir (durum bilinmez olur); immutable: yok edip koddan
   yeniden inşa (durum = kod) (10.4.2). — 8. `&&` önceki başarılıysa (exit 0) çalışır, `||` önceki başarısızsa
   çalışır — ikisi de `$?`'ye bakar (10.1.2).

9. `set -euo pipefail` (10.2.1). — 10. Boş `DIR` → `rm -rf /` (tüm sistem); boşluklu `DIR` → yanlış dizinler;
   düzeltme: `set -u` + `[ -n "$DIR" ]` + `rm -rf "$DIR"/` (10.2.3). — 11. İdempotent değil (`>>` koşulsuz
   ekliyor); düzeltme: `grep -qxF "satır" dosya || echo "satır" >> dosya` (10.4.1). — 12. `set -o pipefail`
   eksik; onsuz yalnızca son komutun (tar) kodu görülür (10.2.1). — 13. Komutun önüne `echo` koy (dry-run) —
   hiçbir şey silmeden gerçekte ne çalışacağını gösterir (10.2.3). — 14. Python'a taşımayı öner: JSON/HTTP/karmaşık
   mantık + test edilebilirlik Bash'in eşiğini aşıyor; Bash yapıştırıcı, bu bir program (10.3.1).

15. Sağlamlık (boot'ta sessizce hata verirse makine yanlış kurulur, kimse görmez) ve idempotency (cloud-init
    yeniden çalışabilir; idempotent değilse her tekrar bozar) — ikisi de otomatik, gözetimsiz çalıştığı için
    kritiktir (10.4.2, Faz 5). — 16. İkisi de "durum = kod" hedefine hizmet eder: idempotency aynı kodu tekrar
    çalıştırmanın durumu değiştirmemesini sağlar; immutability makineyi koddan yeniden inşa ederek durumu koda
    sabitler; birlikte drift'i önler (10.4.2, Faz 8.4). — 17. Elle yapılan hardening tekrarlanamaz, denetlenemez,
    50 makinede tutarsızdır; kod olarak (Ansible/Terraform) yazılınca tekrarlanabilir, gözden geçirilebilir,
    sürüm kontrollü ve idempotent olur (10.4.2, Faz 9). — 18. Güvenilir otomasyonun özü: sessizce başarısız olma
    (`set -euo pipefail` + quoting) ve tekrar çalışınca bozma (idempotency) — yani her koşulda öngörülebilir,
    tekrarlanabilir davranış (10.2, 10.4).

## Puanlama

| Doğru sayısı | Ne anlama geliyor |
|---|---|
| 16-18 | Sağlam ve idempotent otomasyon yazabilirsin. Faz 11'e (troubleshooting) hazırsın. |
| 13-15 | İyi. Kaçırdığın soruların bölümlerini tekrar oku (özellikle 10.2 ve 10.4). |
| 9-12 | Temel var ama kırılgan. `set -euo pipefail`, quoting ve idempotency üzerine çalış. |
| 0-8 | Fazı yeniden gez; kendi test makinende `echo` ile dry-run yaparak küçük script'ler yaz. |

Kaçırdığın soru → dönmen gereken bölüm:

| Soru | Bölüm |
|---|---|
| 1, 8 | 10.1.2 Exit code |
| 2, 10 | 10.1.3 / 10.2.3 Quoting |
| 3, 9, 12 | 10.2.1 `set -euo pipefail` |
| 5 | 10.2.2 `trap` |
| 4, 11, 16 | 10.4.1 Idempotency |
| 6, 14 | 10.3.1 Bash vs Python |
| 7, 15, 17 | 10.4.2 IaC köprüsü |
| 13 | 10.2.3 dry-run |
| 18 | 10.2 + 10.4 birleşim |

---
---

# Faz 10 — Kapanış ve Faz 11'e Köprü

## Bu fazdan ne taşıyorsun

Faz 10 seni "elle yapan"dan "üreten"e dönüştürdü. İki kalıcı fikir edindin: **sağlamlık** (bir script sessizce
yanlış iş yapmamalı — `set -euo pipefail`, quoting, `trap`, exit code kontrolü) ve **idempotency** (aynı kod
tekrar çalışınca aynı son durum). Bu iki fikrin, tek bir script'ten tüm altyapıyı kod olarak yöneten IaC'ye
(cloud-init → Ansible → Terraform) giden köprü olduğunu gördün — ve bunun altında Faz 8'in immutable
felsefesinin yattığını: "sunucu yamama, yeniden inşa et." Artık elle yaptığın her tekrarlayan işi, tekrarlanabilir
ve güvenilir bir koda çevirebilirsin.

## Faz 11 bunun neresine bağlanıyor

Faz 10 "işi otomatikleştirmeyi" verdi; Faz 11 "iş bozulunca kanıtı bulmayı" verir. İkisi bir madalyonun iki
yüzüdür: otomasyon sistemleri **kurar**, troubleshooting sistemleri **onarır**. Ve tam da otomasyon yüzünden
troubleshooting daha da kritik olur — bir script sessizce yanlış iş yaptığında (10.2), onu ancak sistematik bir
adli refleksle (log → durum → kaynak → ağ) yakalarsın. Faz 11, önceki tüm fazların "bu faz bozulunca"
notlarını tek bir metodolojiye toplar: bir şey bozulunca panik değil, katman katman kanıt.

> **🤔 Faz çıktısı — kendine sor:** Bu fazda gördüğün "sessiz başarısızlık" (script hata verip devam etti) ile
> Faz 11'in "kanıt nerede" sorusu nasıl bağlanır? Bir cloud-init script'in bir makinede sessizce yarım kaldı
> ve makine "sağlıklı görünüyor" ama uygulama çalışmıyor. Faz 11'e girmeden düşün: bu sessiz arızanın kanıtını
> hangi sırayla (hangi log, hangi durum komutu) arardın — ve bu neden tam olarak "instance sağlıklı ama uygulama
> değil" ayrımıdır?
>
> **🧪 Lab 10 fikri (kendi test instance'ında):** Kısa bir yedekleme/temizlik script'i yaz: (1) `set -euo
> pipefail` ile başlat. (2) Bir dizindeki 7 günden eski logları bul (`find /var/log/test -type f -mtime +7`) —
> önce sadece `-print` ile listele (dry-run!), sonra `-delete` ekle. (3) Her yıkıcı komuttan önce `echo` ile ne
> silineceğini gör. (4) Exit code'u kontrol et (`echo $?`). (5) Script'i **iki kez** çalıştır — ikinci
> çalıştırma hata veriyor mu, yoksa idempotent mi? İdempotent değilse düzelt. Bu lab, bu fazın sağlamlık +
> idempotency refleksini elinde toplar.

---

> **Navigasyon:** [◀ Ara Sınav 4](Ara_Sinav_4.md) · **Faz 10** · [Faz 11 — Gözlemlenebilirlik ve Troubleshooting ▶](Faz_11_Gozlemlenebilirlik_ve_Troubleshooting.md)
