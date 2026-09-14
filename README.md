# Cloud Engineer — Foundation Learning Paths

> **Two ways to learn the same three subjects — with a mentor, or on your own.**
> Every path exists in **English and Turkish**.

🇬🇧 **English** (below) · 🇹🇷 **[Türkçe](#tr)**

---

## What this repository is

This repository is a **foundation curriculum** for someone becoming a Cloud Engineer. It
does not teach cloud services first — it teaches what sits **underneath** them: the physical
machine, the operating system, and the network. The goal is to be able to justify an
architectural decision and hunt down a fault from first principles, instead of reciting
commands.

It is deliberately **not** a single document. The same material is delivered in two
different shapes, because two different kinds of study session need two different things.

---

## The two learning paths

| Path | What it is | Use it when |
|---|---|---|
| **[Cloud Learning With Mentor](Cloud%20Learning%20With%20Mentor/)** | A **roadmap** — a structured syllabus written to be handed to an AI mentor, which then opens the topics in order, asks questions, and calibrates to your level. | You want a dialogue: to be questioned, corrected, and pushed. Requires an AI assistant. |
| **[Cloud Learning With Workbooks](Cloud%20Learning%20With%20Workbooks/)** | A **workbook** — the same syllabus written out in full: explanation, thinking exercises, answers, self-tests. | You want to study alone, offline, at your own pace. Requires nothing but the file. |

They are **not rivals.** The recommended pattern is sequential: work through a phase in the
workbook, then hand the matching roadmap to a mentor and go through the same phase **out
loud.** Explaining what you read is what makes it stick.

---

## The three subjects

Both paths cover the same three subjects, in this order:

| Subject | The question it answers |
|---|---|
| **Hardware** | What is physically underneath this machine, and what is this workload limited by? |
| **Linux** | How do I read a running server, and how do I hunt a fault on it? |
| **Network** | How does the data actually flow, and why was that packet dropped? |

They are scoped to overlap as little as possible: hardware takes the network *hardware*,
network takes the network *protocols*, and Linux hands protocol theory over to the network
subject.

---

## Languages

Everything exists in **English** and **Turkish** — both paths, all three subjects. The two
versions are kept in step with each other in content and structure; neither is a summary of
the other.

Pick one language and stay in it. Switching mid-subject costs you the vocabulary you have
just built.

> **One exception:** the material is supported by **diagrams**, and the terminology inside
> every diagram is **in English** regardless of which language version you are reading. This
> is deliberate — the technical terms are English in practice, in documentation, and in every
> job interview, so learning them in English from the start is part of the point.

---

## Repository layout

```
.
├── Cloud Learning With Mentor/         ← roadmaps, to hand to an AI mentor
│   ├── English/
│   └── Turkish/
│
└── Cloud Learning With Workbooks/      ← offline workbooks, to study alone
    ├── Cloud_hardware.roadmap/
    ├── Cloud_linux.roadmap/
    └── Cloud_network.roadmap/
```

The logic is the same at every level:

1. **Top level — how you learn.** Mentor or workbook.
2. **Second level — what you learn.** Hardware, Linux, network.
3. **Inside that — which language.** English and Turkish sit side by side.

Each folder carries its own README explaining how that specific path is worked through, and
each subject is split into **phases** that are meant to be taken in order. Start from the
README of the path you chose, not from the middle of a file.

---

## Where to start

1. **Choose a path** — mentor if you want to be questioned, workbook if you want to work alone.
2. **Choose a language** — and stay in it.
3. **Start with hardware.** Linux and network both lean on it; the reverse is not true.
4. **Open that path's own README** and follow it from there.

---

## Status

| Subject | Mentor (EN / TR) | Workbook (EN / TR) |
|---|---|---|
| Hardware | ✅ / ✅ | ✅ / ✅ |
| Linux | ✅ / ✅ | in progress |
| Network | ✅ / ✅ | planned |

Diagrams are being produced and added to the workbooks phase by phase.

---
---

<a name="tr"></a>

# Cloud Engineer — Temel Öğrenme Yolları

> **Aynı üç konuyu öğrenmenin iki yolu — mentorla ya da tek başına.**
> Her yolun **İngilizce ve Türkçe** sürümü var.

🇹🇷 **Türkçe** (aşağıda) · 🇬🇧 **[English](#cloud-engineer--foundation-learning-paths)**

---

## Bu repo nedir

Bu repo, Cloud Engineer olmak isteyen biri için hazırlanmış bir **temel müfredattır.** Önce
cloud servislerini öğretmez — onların **altında** ne olduğunu öğretir: fiziksel makine,
işletim sistemi ve ağ. Amaç komut ezberletmek değil; bir mimari kararı gerekçelendirebilmek
ve bir arızayı ilk prensiplerden avlayabilmektir.

Bilinçli olarak **tek bir belge değildir.** Aynı içerik iki farklı biçimde sunulur, çünkü iki
farklı çalışma biçimi iki farklı şey ister.

---

## İki öğrenme yolu

| Yol | Nedir | Ne zaman kullanılır |
|---|---|---|
| **[Cloud Learning With Mentor](Cloud%20Learning%20With%20Mentor/)** | Bir **yol haritası** — bir YZ mentoruna verilmek üzere yazılmış yapılandırılmış müfredat. Mentor konuları sırayla açar, soru sorar, seviyene göre kalibre eder. | Karşılıklı konuşmak istiyorsan: sorgulanmak, düzeltilmek, zorlanmak. Bir YZ asistanı gerekir. |
| **[Cloud Learning With Workbooks](Cloud%20Learning%20With%20Workbooks/)** | Bir **çalışma kitabı** — aynı müfredatın baştan sona yazılmış hâli: anlatım, düşünme egzersizleri, cevaplar, kendini sınama testleri. | Tek başına, internetsiz, kendi hızında çalışmak istiyorsan. Dosyadan başka hiçbir şey gerekmez. |

Bu ikisi **rakip değil.** Önerilen kullanım sıralıdır: bir fazı çalışma kitabıyla işle, sonra
aynı fazın yol haritasını bir mentora verip konuyu **sözlü** tekrar et. Okuduğunu anlatmak,
öğrendiğini kalıcı yapan şeydir.

---

## Üç konu

Her iki yol da aynı üç konuyu, bu sırayla kapsar:

| Konu | Cevapladığı soru |
|---|---|
| **Hardware (Donanım)** | Bu makinenin altında fiziksel olarak ne var, bu iş yükü neyle sınırlı? |
| **Linux** | Çalışan bir sunucuyu nasıl okurum, üzerindeki arızayı nasıl avlarım? |
| **Network (Ağ)** | Veri gerçekte nasıl akıyor, o paket neden düştü? |

Üçü birbirini olabildiğince az örtecek şekilde sınırlanmıştır: donanım ağın *donanımını*,
network ağın *protokolünü* alır; Linux protokol teorisini network konusuna devreder.

---

## Diller

Her şeyin **İngilizce** ve **Türkçe** sürümü var — her iki yol, üç konunun tamamı. İki sürüm
içerik ve yapı olarak birbiriyle eşit tutulur; hiçbiri diğerinin özeti değildir.

Bir dil seç ve onda kal. Konunun ortasında dil değiştirmek, o ana kadar kurduğun terim
dağarcığını kaybettirir.

> **Tek istisna:** içerik **diyagramlarla** desteklenir ve her diyagramın içindeki terminoloji,
> hangi dil sürümünü okuduğundan bağımsız olarak **İngilizcedir.** Bu bilinçli bir tercihtir —
> teknik terimler sahada, dokümantasyonda ve her iş görüşmesinde zaten İngilizcedir; onları
> baştan İngilizce öğrenmek işin bir parçasıdır.

---

## Repo yapısı

```
.
├── Cloud Learning With Mentor/         ← YZ mentoruna verilecek yol haritaları
│   ├── English/
│   └── Turkish/
│
└── Cloud Learning With Workbooks/      ← tek başına işlenecek offline çalışma kitapları
    ├── Cloud_hardware.roadmap/
    ├── Cloud_linux.roadmap/
    └── Cloud_network.roadmap/
```

Mantık her seviyede aynı:

1. **Üst seviye — nasıl öğrendiğin.** Mentor mu, çalışma kitabı mı.
2. **İkinci seviye — ne öğrendiğin.** Donanım, Linux, ağ.
3. **Onun içinde — hangi dil.** İngilizce ve Türkçe yan yana durur.

Her klasörün kendi README'si vardır ve o yolun nasıl işleneceğini anlatır. Her konu, sırayla
alınması gereken **fazlara** bölünmüştür. Bir dosyanın ortasından değil, seçtiğin yolun
README'sinden başla.

---

## Nereden başlamalı

1. **Bir yol seç** — sorgulanmak istiyorsan mentor, tek başına çalışacaksan çalışma kitabı.
2. **Bir dil seç** — ve onda kal.
3. **Donanımla başla.** Linux ve ağ ona yaslanır; tersi geçerli değildir.
4. **Seçtiğin yolun kendi README'sini aç** ve oradan devam et.

---

## Durum

| Konu | Mentor (EN / TR) | Çalışma kitabı (EN / TR) |
|---|---|---|
| Hardware | ✅ / ✅ | ✅ / ✅ |
| Linux | ✅ / ✅ | devam ediyor |
| Network | ✅ / ✅ | planlandı |

Diyagramlar üretilip çalışma kitaplarına faz faz ekleniyor.
