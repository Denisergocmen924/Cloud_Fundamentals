# Bilgisayarın Fiziksel Katmanı — Offline Çalışma Kitabı

> **Cloud Engineer Temel Yol Haritası — Donanım**
> Bu, [`Cloud_hardware_roadmap_temel_tr.md`](../../../Cloud%20Learning%20With%20Mentor/Turkish/Cloud_hardware_roadmap_temel_tr.md) yol
> haritasının **tek başına, YZ mentoru olmadan, internetsiz** baştan sona işlenebilir hâlidir.

> 🇬🇧 English version: **[README_en.md](../English_Hardware/README_en.md)**

---

## Bu kitap nedir, ana haritadan farkı ne?

Mentor klasöründeki yol haritası dosyası bir **iskelettir**: bir YZ mentoruna verilir, mentor
konuları sırayla açar. İskelet tek başına okunduğunda "ne öğrenmem gerektiğini" söyler ama
"onu öğretmez".

Bu kitap o iskeletin **eti**dir. Aynı fazlar, aynı alt maddeler, aynı derinlik etiketleri —
ama her madde burada:

- **anlatılır** (sezgi → mekanizma → rakam),
- **sorulur** (senin düşünmen için bırakılan sorular),
- **cevaplanır** (aynı dosyanın ilerleyen bölümünde),
- **bağlanır** (önceki maddeye ve AWS'teki karşılığına).

Hiçbir yerde "bunu mentoruna sor" demez. Dosyayı bir uçakta, internetsiz, tek başına
baştan sona işleyebilirsin.

---

## Dosyalar

| Dosya | Faz | Konu | Yaklaşık süre |
|---|---|---|---|
| [Faz_0_Sayisal_Temel.md](Faz_0_Sayisal_Temel.md) | 0 | Binary, transistör, mantık kapıları, flip-flop | 3–5 saat |
| [Faz_1_CPU_Mimarisi.md](Faz_1_CPU_Mimarisi.md) | 1 | ALU, fetch-decode-execute, clock, pipeline, core/vCPU, ISA | 5–7 saat |
| [Faz_2_Bellek_Hiyerarsisi.md](Faz_2_Bellek_Hiyerarsisi.md) | 2 | Cache, RAM, sanal bellek, swap, NUMA | 7–10 saat |
| [Faz_3_Depolama.md](Faz_3_Depolama.md) | 3 | HDD, SSD, NVMe, IOPS/throughput/latency, RAID | 5–7 saat |
| [Faz_4_Bus_ve_IO.md](Faz_4_Bus_ve_IO.md) | 4 | Bus, PCIe, DMA, interrupt, chipset | 4–5 saat |
| [Faz_5_Ag_Donanimi.md](Faz_5_Ag_Donanimi.md) | 5 | NIC, switch, bandwidth/latency, RDMA | 3–4 saat |
| [Faz_6_Sanallastirma_Donanimi.md](Faz_6_Sanallastirma_Donanimi.md) | 6 | Hypervisor, VT-x, EPT, vCPU scheduling, SR-IOV, container | 6–8 saat |
| [Faz_7_Cloud_Baglantisi.md](Faz_7_Cloud_Baglantisi.md) | 7 | Instance aileleri, Nitro, EBS seçimi, noisy neighbor, darboğaz avı | 5–7 saat |

**Ekler:**

| Dosya | Ne işe yarar |
|---|---|
| [EK_A_Referans_Tablolari.md](EK_A_Referans_Tablolari.md) | Gecikme merdiveni, birim tabloları, PCIe/DDR/EBS rakamları — masaüstünde açık tut |
| [EK_B_Terim_Sozlugu.md](EK_B_Terim_Sozlugu.md) | Tüm terimler, kısa tanım + geçtiği bölüm + kolay karıştırılan çiftler |

---

## Nasıl çalışılır

### 1. Sırayı bozma

Her faz bir öncekinin **kelimeleriyle** konuşur. Faz 2 "cache miss"i anlatırken Faz 0'daki
SRAM hücresini, Faz 1'deki pipeline stall'ı kullanır. Faz 4'ü Faz 2'siz okursan cümleleri
anlarsın ama **neden**ini anlamazsın.

Tek istisna: Faz 0'ın `[kavram]` etiketli kısımları (mantık kapıları, flip-flop) hızlı
geçilebilir — ama **atlanmamalı**, çünkü Faz 2'nin tamamı oraya dayanır.

### 2. Kutuları ciddiye al

Metinde dört tür kutu var. Hepsinin farklı bir işi var:

> **🤔 Düşün 2.3** — Buradaki soruyu **okur okumaz cevaplama.** Kalem al, 2 dakika düşün,
> tahminini yaz. Cevabı fazın sonundaki *"Düşün sorularının cevapları"* bölümünde.
> Tahminin yanlış çıkarsa bu iyi bir şey — yanlış tahmin, doğru cevabı kalıcı yapar.

**❓ Akla gelen soru:** Okurken doğal olarak oluşan soru. Cevabı **hemen altında**.
Bunlar beklemez, çünkü cevaplanmazsa okuma akışını kırar.

**⚠️ Yaygın yanılgı:** Çoğu insanın yanlış bildiği şey. Bunları okurken "ben de böyle
biliyordum" diyorsan, o cümleyi işaretle.

**🔧 Makinende gör:** İsteğe bağlı. Linux makinen varsa çalıştır, yoksa **beklenen çıktı**
zaten yazılı — okuyup geçebilirsin. Hiçbiri sistemini bozmaz (tamamı salt-okunur).

### 3. Derinlik etiketlerine uy

| Etiket | Ne bekleniyor | Nasıl test edersin |
|---|---|---|
| `[kavram]` | Sezgisel açıklayabilmek | "Bunu bir arkadaşıma 30 saniyede anlatabilir miyim?" |
| `[mekanizma]` | Adım adım anlatabilmek | "Kalem kâğıtla akışı çizebilir miyim?" |
| `[uygulama]` | Bir mimari kararla ilişkilendirebilmek | "Hangi instance / hangi disk — **neden**?" |
| `[atla]` | Sadece adını bilmek | Adını duyunca "o şu alandaydı" diyebilmek yeter |

`[uygulama]` bu haritada **"komut çalıştır" demek değildir.** Konuyu somut bir cloud
kararına bağlayabilmek demektir. Bir `[uygulama]` maddesini, "bu bilgi hangi seçimi
değiştirir?" sorusuna cevap veremeden geçme.

### 4. Faz sonunu atlama

Her faz şununla biter:

1. **Düşün sorularının cevapları** — kendi tahminlerinle karşılaştır
2. **Sık sorulan sorular** — fazın etrafındaki "peki ya şu?" soruları
3. **Kendini sına** — 18–22 soruluk test + tam cevap anahtarı
4. **Kapanış ve köprü** — bu fazdan ne kaldı, sonraki faz bunun neresine bağlanıyor

Testte %70'in altında kaldıysan **fazı tekrar etmek yerine**, yanlış yaptığın sorunun
işaret ettiği bölüme geri dön. Her cevap anahtarı hangi bölüme ait olduğunu söyler.

---

## İlerleme takibi

Kendi konumunu buradan takip et. Bir fazı bitirdiğinde kutuyu işaretle.

```
[ ] Faz 0 — Sayısal Temel
[ ] Faz 1 — CPU Mimarisi
[ ] Faz 2 — Bellek Hiyerarşisi          ← en kritik faz, acele etme
[ ] Faz 3 — Depolama
[ ] Faz 4 — Sistem Bus'ları ve I/O
[ ] Faz 5 — Ağ Donanımı
[ ] Faz 6 — Sanallaştırma Donanımı      ← "aha" anlarının fazı
[ ] Faz 7 — Cloud Bağlantısı            ← öğrenme değil, karar verme fazı
```

---

## Bu kitabın hedefi ve sınırı

**Hedef:** Bir cloud mimarisi kararının önüne geldiğinde — hangi instance ailesi, hangi EBS
tipi, neden yavaş, darboğaz nerede — cevabı **ezberden değil fizikten** türetebilmek.

**Sınır:** Bu kitap transistör tasarımcısı, çip mimarı veya kernel geliştiricisi yetiştirmez.
CMOS fiziği, VLSI, mikrokod tasarımı `[atla]` etiketlidir ve bilinçli olarak dışarıda
bırakılmıştır.

**Üç içgüdü sorusu.** Bu kitabın tamamı şu üç soruyu düşünmeden cevaplayabilmen için var:

1. **Bu iş yükü neyle sınırlı?** → CPU mu, bellek mi, disk mi, ağ mı (Faz 1, 2, 3, 5, 7)
2. **Bu instance'ın altında fiziksel olarak ne var?** → core, cache, NUMA, hypervisor (Faz 1, 2, 6)
3. **Bu yavaşlık nereden geliyor?** → cache miss, swap, IOPS tavanı, steal time, PCIe doygunluğu (Faz 2, 3, 6, 7)

---

## Ana haritayla ilişki

| Bu kitap | Ana harita |
|---|---|
| Offline, tek başına işlenir | YZ mentoruna verilir |
| Anlatım + soru + cevap | Konu listesi + mentor talimatı |
| Sabit içerik | Öğrencinin seviyesine göre mentor kalibre eder |
| Kendini sına testleri | Mentor sözlü yoklar |

İkisi **rakip değil**. Sıralı kullanım önerilir: önce bu kitapla fazı işle, sonra ana
haritayı bir YZ mentoruna verip aynı fazı **sözlü** tekrar et. Okuyarak öğrendiğini
anlatarak sağlamlaştırırsın.

---

*Bu offline sürüm, Denis Ergöçmen'in Cloud Engineer temel yol haritası serisine dayanır.*
