# Cloud Engineer — Temel Yol Haritaları (TR)

Bu repo, **Cloud Engineer** olmak isteyen biri için hazırlanmış, Türkçe ve
**Socratic (yapıcı soru-cevap) mentorluk** temelli üç **temel** öğrenme yol
haritasından oluşur. Amaç komut ezberletmek değil; bulutun **altındaki sistemi**
(donanım, işletim sistemi, ağ) bir mimari kararı gerekçelendirecek ve arızayı
avlayacak kadar derinden görmektir.

> Bu üçlü, kariyerin **temel (foundation) katmanıdır** — tüm yolculuğun kabaca
> ilk üçte biri. Bittiğinde AWS servisleri, IaC (Terraform), container
> orkestrasyon, CI/CD ve mimari desenler gibi **uygulama katmanı** haritalarıyla
> devam edilmelidir (bkz. "Sonraki adım").

---

## Dosyalar

| Dosya | Konu | Faz | Ne kazandırır |
|---|---|---|---|
| `Cloud_hardware_roadmap_temel_tr.md` | Bilgisayarın fiziksel katmanı (CPU, bellek, depolama, bus, ağ donanımı, sanallaştırma) | 0–7 | "Instance seçimini ve darboğazı **fizikle** gerekçelendir" |
| `Cloud_linux_roadmap_temel_tr.md` | Linux / işletim sistemi (shell, izinler, process, bellek, boot/systemd, depolama, ağ-OS, hardening, otomasyon, troubleshooting) | 0–12 | "Çalışan bir sunucuyu **oku** ve arızayı **avla**" |
| `Cloud_network_roadmap_temel_tr.md` | Ağ temelleri (OSI/encapsulation, adresleme, subnet/CIDR, L2/L3, transport, DNS, NAT, HTTP/TLS, firewall) | 0–11 | "Veri nasıl akıyor, **paket neden düştü**" |

Üç dosya birbirini akıllıca sınırlar (örtüşme minimaldir): donanım ağ
*donanımını*, network ağ *protokolünü* alır; Linux protokol teorisini network
haritasına devreder.

---

## Nasıl kullanılır?

Bu dosyalar **kendi kendine okunan makaleler değil**, bir **YZ mentoruna
verilen talimat dosyalarıdır**. Her dosyanın başında **"Mentor Başlangıç
Talimatı — bu dosyayı okuyan AI için"** bölümü vardır; mentor bu kuralları
benimser, sen (öğrenci) yönü verirsin.

### 1. Dosyayı bir YZ mentoruna ver
Bir dosyanın tamamını Claude gibi bir asistana yükle/yapıştır ve "bu benim yol
haritam, mentorluk kurallarını benimse" de. Aynı anda **tek bir** haritayla
çalış.

### 2. Konumunu sen sabitle
İlerlemeyi mentor değil **sen** yönetirsin:
- **"Faz X'teyiz"** → o fazın başından başla
- **"Faz X.Y'deyiz"** → o alt-maddeden kesintisiz devam et

### 3. Socratic ritme uy
Her kavramda akış: mentor **yönlendirici bir soru sorar** → sen **tahmin
edersin** → doğruysa mentor **terimi koyar ve tanımlar** → yanlışsa **direkt
düzeltir** → üstüne birlikte inşa edersiniz. Bir kavram bitmeden diğerine
geçilmez.

### 4. Derinlik etiketlerini oku
Her başlığın yanındaki etiket, o konuyu **ne kadar** öğrenmen gerektiğini söyler:

| Etiket | Anlamı |
|---|---|
| `[kavram]` | Sezgisel açıklayabil / tanımını verebil yeter |
| `[mekanizma]` | Adım adım nasıl çalıştığını anlatabilmelisin |
| `[uygulama]` | Kendi makinende elle çalıştır & gör / mimari kararla ilişkilendir |
| `[atla]` | Sadece adını bil, derine inme |

### 5. Lab ve ara sınavları kullan (opsiyonel)
- **Lab / komut önerileri**: `[uygulama]` konularından sonra mentor ilgili
  komutları önerir. Çalıştırıp çıktıyı getirirsen birlikte okursunuz.
  İstemezsen zorlanmaz — karar sende.
- **✅ Ara sınavlar / faz kapanışları**: Checkpoint'lere gelince mentor sınavı
  otomatik önerir. Ertelemek istersen ertelenir.
- ⚠️ Bazı Lab'lar sistemi kalıcı değiştirir (mount, fstab, servis). Yıkıcı
  adımlarda mentor önce uyarır.

### 6. Faydalı komutlar
- **"direkt söyle"** → Socratic soruyu bırak, doğrudan anlat.
- **"Bozulunca"** ekseni her fazda vardır: "bu bilgi arızalanınca sistemde nasıl
  görünür?" — arıza içgüdüsünü buradan kur.

---

## Önerilen sıra

Haritalar bağımsızdır, ama Cloud Engineer hedefi için önerilen akış:

1. **Network Faz 0–2** (subnet/CIDR) + **Linux Faz 0–4** → paralel başla; bunlar
   günlük iş becerileridir.
2. **Hardware'i "derin neden" hattı** olarak araya serpiştir. Hardware Faz 0
   (mantık kapıları, flip-flop) `[kavram]` düzeyinde hızlı geçilebilir.
3. Kalan fazları ilerlet ve üç haritanın da **son fazını (Cloud'a Köprü)** birlikte
   yaparak AWS'e geçişi oturt.

---

## Bu haritalar neyi kapsar / kapsamaz

**Kapsar (derin):** donanım/compute fiziği, Linux/OS, ağ temelleri (L1–L7),
Bash scripting, troubleshooting metodolojisi, sanallaştırma/container *kavramı*.

**Kapsamaz (bilinçli olarak — sonraki katman):**
- AWS çekirdek servisleri operasyonel derinlik (EC2/VPC/S3/IAM/RDS…)
- IaC — Terraform / CloudFormation / CDK
- Container orkestrasyon — Docker derinlik, Kubernetes/ECS/EKS
- CI/CD, Git, Python (boto3) otomasyon
- Veritabanları, ölçekli gözlemlenebilirlik, mimari desenler, maliyet/FinOps

---

## Sonraki adım (öneri)

Aynı Socratic + "Bozulunca" + cloud-köprüsü formatıyla eksik uygulama katmanını
haritalamak:

```
4. AWS Core Services (SAA odaklı)     ← en acil
5. IaC / Terraform
6. Docker → Kubernetes / ECS-EKS
7. CI/CD + Git + Python otomasyon
8. Mimari desenler + gözlemlenebilirlik + FinOps  ← capstone
```

---

## Notlar

- Tüm içerik Türkçedir; teknik terimler İngilizce kalır, ilk geçişte parantez
  içinde Türkçe telaffuz + kısa tanım verilir.
- Cloud örnekleri **AWS-merkezlidir**; GCP/Azure'da terminoloji farklıdır.
- Dosyalar kişiye özel değildir; öğrenci genel olarak ele alınmıştır ve herkes
  tarafından işlenebilir.

---

*Hazırlayan: Denis Ergöçmen, Haziran 2026 — genel kullanım için nötrleştirilmiş sürüm*
