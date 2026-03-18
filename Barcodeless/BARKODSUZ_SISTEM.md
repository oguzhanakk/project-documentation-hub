# 📦 Barkodsuz Eşleştirme Sistemi
### Kayıp & Barkod Hasarlı Paketler İçin Otomasyon Altyapısı

---

## 🔍 Problem: Neden Bu Sistem Var?

### Gerçek Hayatta Ne Oluyor?

Trendyol lojistik sürecinde her paket barkod ile takip edilir. Ancak bazı paketlerin barkodları:

- Fiziksel hasar nedeniyle **okunamaz** hale gelir
- Taşıma sırasında **etiket kaybolur**
- Depo kamerası veya el terminali **okuma yapamaz**

Bu noktada paket **"kayıp"** statüsüne düşer. Ürün fiziksel olarak depoda durmaktadır, ama sistem onu tanıyamaz.

---

### Mevcut Manuel Süreç (Otomasyon Öncesi)

```
📋 Saha ekibi rapor çıktısı alır
        │
        ▼
👀 Ürünü fiziksel olarak bakar, listeden benzerini arar
        │
        ▼
✍️  Manuel olarak eşleştirir (kişi başına günlerce sürer)
        │
        ▼
⚠️  Hata oranı yüksek, ölçeklenemiyor
```

**Sorun:** Yüz binlerce kayıp paketi birkaç kişiyle manuel eşleştirmek; hem zaman alıcı, hem hatalı, hem de ölçeklenemez.

---

### Bizim Çözümümüz: Tam Otomasyon

Saha ekibi artık ürünü fiziksel olarak inceleyip lojistik ekranına şunları giriyor:

| Alan | Örnek |
|------|-------|
| **kbarcode** | `K1763989382665` |
| **Marka** | `Zara` |
| **Ürün İçeriği** | `Kadın Siyah Deri Ceket` |
| **Beden** | `M` |
| **Depo Fotoğrafı** | 📸 Elde tutup çekilen fotoğraf |

Sistem bu bilgilerle **DWH'daki kayıp paketler** arasında otomatik eşleştirme yapıyor ve sonucu rapora yansıtıyor.

---

## 🏗️ Sistem Mimarisi — Tek Bakışta

```
┌─────────────────────────────────────────────────────────────────────┐
│                        VERİ KAYNAKLARI                              │
│                                                                     │
│  📊 Google BigQuery (DWH)                                           │
│  ┌──────────────────────┐    ┌──────────────────────────────────┐   │
│  │   warehouse_base_v1  │    │        package_base_v1           │   │
│  │  (Depo ürünleri —    │    │  (Müşteri paketleri —           │   │
│  │   barkod hasarlı)    │    │   kayıp/eşlenecek)              │   │
│  └──────────┬───────────┘    └──────────────┬───────────────────┘   │
└─────────────┼────────────────────────────────┼─────────────────────┘
              │                                │
              ▼                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     EŞLEŞTİRME PIPELINE                             │
│                                                                     │
│  Adım 1 ──▶  🔑 SKU Matching          (EN GÜVENİLİR)               │
│  Adım 2 ──▶  🔗 Link Matching         (Trendyol Ürün ID)            │
│  Adım 3 ──▶  📍 Exact Matching        (İçerik Bazlı)                │
│  Adım 4 ──▶  🎯 Fuzzy Matching        (Benzerlik Bazlı)             │
│  Adım 5 ──▶  🤖 AI Görsel Matching    (Görsel Analiz)               │
│  Adım 6 ──▶  🧹 Cleanup              (Hata Temizleme)               │
│  Adım 7 ──▶  🔄 Retry                (İkinci Şans)                  │
│  Adım 8 ──▶  🧠 AI Final Matching    (Son Çare)                     │
└─────────────────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         ÇIKTI                                        │
│                                                                     │
│  💾 PostgreSQL                                                       │
│  ├─ missing_packages_match          (Son run sonuçları)             │
│  └─ missing_packages_match_historic (Tüm geçmiş + versioning)      │
│                                                                     │
│  📊 Raporlama & Dashboard                                           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📊 Veri Kaynakları

### 🏭 `warehouse_base_v1` — Depodaki Kayıp Ürünler

Barkodu okunamayan, saha ekibi tarafından **elle kaydedilmiş** ürünler.

```
kbarcode              → K1763989382665      (Benzersiz depo barkodu)
sku                   → 8683161605318       (Tedarikçi/üretici kodu)
product_content_name  → Bağ Budama Makası   (Saha ekibi elle girdi)
brand_name            → aslan kara          (Saha ekibi elle girdi)
size                  → -                   (Bilinmiyorsa "-")
warehouse_image_url   → s3.trendyol.com/... (Saha ekibinin çektiği fotoğraf)
location              → KMBA1000000000000   (Depo rafı)
sorting_center_name   → ESENYURT_SC
```

> ⚠️ **Önemli:** Bu veriler saha ekibi tarafından **elle girildiği için** yazım hataları veya eksiklikler içerebilir.

---

### 📦 `package_base_v1` — Müşteri Paketleri (Kayıp)

Sistem tarafından kayıp olarak işaretlenmiş, barkodu hasarlı paketler.

```
integration_code      → 7330026227808172    (Paket kodu)
sku                   → 2130105007484       (Tedarikçi kodu)
product_content_name  → Made In Disintegration Antrasit Unisex Sweatshirt Hoodie
brand_name            → Trendiz
size                  → M
image_url             → cdn.dsmcdn.com/...  (Trendyol katalog görseli — stüdyo çekimi)
package_count         → 1
```

> ✅ **Önemli:** Bu veriler **sistem tarafından otomatik kaydedildiği için** güvenilirdir.

---

## 🎯 Eşleştirme Pipeline — Adım Adım

Sistem önce en güvenilir yöntemi dener, eşleşme bulamazsa bir sonrakine geçer. Her adım, önceki adımdan **elenen** kayıtlara uygulanır.

---

### Adım 1 — 🔑 SKU Matching *(En Güvenilir)*

**Ne yapar?** SKU (Stock Keeping Unit), tedarikçi/üretici tarafından atanan benzersiz koddur. Her iki tarafta da varsa, en güvenilir eşleştirme metodudur.

**3 farklı kural:**

| Kural | Karşılaştırma | Güvenilirlik |
|-------|--------------|:---:|
| SKU + Beden | `sku="12345678"` + `size="M"` | ⭐⭐⭐⭐⭐ 98% |
| SKU + Marka | `sku="12345678"` + `brand="Zara"` | ⭐⭐⭐⭐⭐ 98% |
| SKU (tek başına) | `sku="12345678"` | ⭐⭐⭐⭐⭐ 95% |

```
Warehouse: sku=12345678, size=M
Package:   sku=12345678, size=M
                ↕               ↕
           EŞİT            EŞİT
                    ✅ EŞLEŞME!
```

> 🛡️ SKU eşleştirmeleri **cleanup'a girmez** — doğrudan final tabloya yazılır.

---

### Adım 2 — 🔗 Link Matching

**Ne yapar?** Saha ekibi zaman zaman Trendyol ürün linkini kaydeder. URL'den ürün ID'si çıkarılır, her iki tarafta aynıysa eşleştirme yapılır.

```python
# URL'den ürün ID çıkar (regex)
"https://www.trendyol.com/marka/urun-p-123456789"
                                            ↑
                                     Ürün ID: 123456789

Warehouse: product_link → p-123456789
Package:   product_link → p-123456789
                              ↕
                         AYNI ID → ✅ EŞLEŞME! (%100 güven)
```

> 🛡️ Link eşleştirmeleri de **cleanup'a girmez.**

---

### Adım 3 — 📍 Exact Matching *(İçerik Bazlı)*

**Ne yapar?** Metin alanlarını **birebir** karşılaştırır. Normalize edilmiş (küçük harf, trim) değerler kullanılır.

**3 farklı kural:**

| Kural | Örnek | Güvenilirlik |
|-------|-------|:---:|
| Ürün Adı + Marka + Beden | `"Kadın Elbise"` + `"Zara"` + `"M"` | ⭐⭐⭐⭐ 90% |
| Ürün Adı + Beden | `"Kadın Elbise"` + `"M"` | ⭐⭐⭐⭐ 85% |
| Ürün Adı (tek) | `"Kadın Siyah Deri Ceket"` | ⭐⭐⭐ 75% |

> ⚠️ Exact eşleştirmeler **cleanup'a girer** — hata olasılığı var.

---

### Adım 4 — 🎯 Fuzzy Matching *(Benzerlik Bazlı)*

**Ne yapar?** Metinler birebir aynı olmasa da **yüksek benzerlik** varsa eşleştirir. Kelime sırası farkı, yazım yanlışı vb. tolere edilir.

```python
from rapidfuzz import fuzz

text1 = "Kadın Siyah Deri Ceket"
text2 = "Kadın Deri Ceket Siyah"   # ← Kelime sırası farklı

fuzz.token_sort_ratio(text1, text2)
# → %100 ✅  (kelimeleri sıralayıp karşılaştırır)
```

**4 farklı kural:**

| Kural | Eşik | Güvenilirlik |
|-------|------|:---:|
| SKU benzerliği | ≥ %95 | ⭐⭐⭐⭐ 85% |
| Marka + İçerik benzerliği | ≥ %85 | ⭐⭐⭐ 80% |
| İçerik benzerliği | ≥ %90 | ⭐⭐⭐ 75% |
| SKU + İçerik benzerliği | ≥ %80 | ⭐⭐⭐ 80% |

> ⚠️ Fuzzy eşleştirmeler **cleanup'a girer.**

---

### Adım 5 — 🤖 AI Görsel Eşleştirme *(3 Aşamalı)*

Metin bazlı eşleşemeyen ürünler için **görsel yapay zeka** devreye girer. Bu üç aşamadan oluşur:

---

#### 🔬 Aşama 5A — AI Görsel Analiz (Her Görsel İçin)

Her ürün görseli **Trendyol AI API**'ye gönderilir. AI görseli inceleyerek **50'den fazla alan** çıkarır:

```json
{
  "kategori":       "Giyim",
  "tip":            "Ceket",
  "marka":          "Zara",
  "renk":           "Siyah",
  "cinsiyet":       "Kadın",
  "malzeme":        "Deri",
  "stil":           "Klasik",
  "yaka_tipi":      "Dik Yaka",
  "kollar":         "Uzun Kol",
  "giyim_kesim":    "Regular Fit",
  "logo":           "Var",
  "logo_konum":     "Sol Göğüs",
  "signature":      "giyim|ceket|zara|siyah|deri|kadın|klasik",
  "enhanced_signature": "giyim|ceket|zara|siyah|kadın|deri|klasik|uzunkol|dikyaka"
  ...
}
```

**Bağlam Farkındalığı:** AI'ya görseli gönderirken ürün metadatası da eklenir:

```
📦 PAKET (Trendyol katalog — %100 güvenilir):
  • Marka: Zara
  • Ürün: Kadın Siyah Deri Ceket
  • Beden: M

🏭 DEPO (saha kaydı — tahmini, yazım hatası olabilir):
  • Marka: zara
  • Ürün: kadin deri ckt siyah
  • Beden: medium
```

Bu sayede AI, depodaki dağınık arka planı değil, **doğru ürüne odaklanarak** analiz yapar.

**Cache Mekanizması:**

```
Görsel daha önce analiz edildi mi?
        ├── ✅ EVET → PostgreSQL cache'den al (AI'ya gönderme, maliyet sıfır)
        └── ❌ HAYIR → AI API'ye gönder → sonucu cache'e kaydet
```

> 💡 İlk analiz maliyetlidir, sonrasında aynı görsel için **sıfır maliyet.**

---

#### 🧮 Aşama 5B — Python AI Matching (Benzerlik Skoru)

AI analiz sonuçları kullanılarak her warehouse ↔ package çifti için benzerlik skoru hesaplanır:

```
Alan              Ağırlık    Örnek
─────────────────────────────────────────────
product_content    %70       "Kadın Ceket" ~ "Kadın Deri Ceket"   → 85%
enhanced_signature %10       imza benzerligi
kategori           %5        "Giyim" == "Giyim"                    → tam
tip                %5        "Ceket" == "Ceket"                    → tam
renk               %5        "Siyah" == "Siyah"                    → tam
diğer alanlar      %5        ...
─────────────────────────────────────────────
TOPLAM SKOR                                   → 88.5%
```

> 🔍 Eşik değeri **%30** — düşük tutulur çünkü AI Verification bir sonraki aşamada filtreleyecek.

Sonuçlar `ai_verification_cache` tablosuna yazılır.

---

#### ✅ Aşama 5C — AI Verification (Görsel Doğrulama)

Benzerlik skoru ≥ %50 olan çiftler AI'ya **ikinci kez** gönderilir. Bu sefer iki görsel yan yana gösterilerek sorulur: *"Aynı ürün mü?"*

```
┌────────────────────────────────────────────────────────┐
│  image1 = Depo fotoğrafı (saha çekimi, arka plan var)  │
│  image2 = Trendyol katalog görseli (stüdyo, temiz)     │
│                                                        │
│  AI'ya sağlanan ek bilgiler:                           │
│  📦 Paket: Zara | Kadın Siyah Deri Ceket | M          │
│  🏭 Depo:  Zara | kadin deri cekit siayh | medium     │
│                                                        │
│  AI'ya verilen özel talimat:                           │
│  "Paket görseli 3'lü set olabilir, depoda 1 tane      │
│   görünüyorsa bu NORMAL — yine AYNI diyebilirsin"     │
└────────────────────────────────────────────────────────┘
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
      KESIN_AYNI | AYNI        FARKLI
      (+ güven skoru 0-100)   (eşleşme yok)
             │
             ▼
   missing_packages_match'e yaz
```

**Yanıt Formatı:** `KESIN_AYNI|95` veya `AYNI|72` veya `FARKLI|15`

AI hem kategori hem de 0-100 arası **güven skoru** döndürür. Her ikisi de `ai_verification_cache` tablosuna kaydedilir.

---

### Adım 5.5 — 🔬 Kalite Filtreleri *(Eşleştirme Sonrası, Cleanup'tan Önce)*

Her eşleştirme turunun sonunda, sonuçlar cleanup'a gönderilmeden önce **3 kalite filtresi** uygulanır. Bu filtreler mantıksal tutarsızlıkları erken yakalamak içindir.

> 🛡️ Trendyol Link eşleştirmeleri (%100 güven) bu filtrelerden de **muaftır**.

---

#### Filtre 1 — 📍 Lokasyon Filtresi

**Mantık:** Aynı pakete (`integration_code`) birden fazla kbarcode eşleştiyse, bu kbarcode'ların hepsi **aynı depo lokasyonunda** olmalıdır. Farklı raftan/merkezden gelen karışık eşleştirmeler güvenilir değildir.

```
integration_code: 7330026227808172

Eşleşen kbarcode'lar:
  K111 → lokasyon: KMBA1000   ✅
  K222 → lokasyon: KMBA1000   ✅
  K333 → lokasyon: ESENYURT_B ❌ ← Farklı merkez!

Sonuç: K333 → FILTER_LOCATION olarak analysis_results'a yazılır ve elenır
```

---

#### Filtre 2 — 🎯 Akıllı Tekil Eşleştirme

**Mantık:** Bir kbarcode yalnızca **bir** pakete eşleşebilir (1 depo ürünü = 1 müşteri paketi). Aynı kbarcode birden fazla eşleştirme adayıyla gelirse, en iyi olanı seçilir.

**Seçim önceliği (sırayla):**

```
1. %50 product_count şartını sağlayan mı? (aşağıda açıklanıyor)
2. Paketin ürün sayısı fazla mı? (product_count)
3. Eşleştirme kuralı önceliği mi?
   (Link > SKU+Beden > SKU+Marka > SKU > İçerik+Marka+Beden > ...)
```

Seçilmeyen diğer adaylar → `FILTER_TEKIL` olarak elenır.

---

#### Filtre 3 — 📊 %50 Ürün Sayısı Filtresi

**Mantık:** Bir paket birden fazla ürün içerebilir (örn. `package_count=3`). Eğer pakete bağlı eşleşen kbarcode sayısı, paketin içerdiği ürün sayısının **%50'sinden azsa**, bu eşleştirme güvenilir sayılmaz ve elenır.

```
integration_code: 7330026227808172
package_count (BQ'dan çekilir): 4 ürün

Eşleşen kbarcode sayısı: 1
1 / 4 = %25 < %50 → ❌ FILTER_PRODUCT_COUNT — elenır

Eşleşen kbarcode sayısı: 3
3 / 4 = %75 ≥ %50 → ✅ Geçer
```

> 💡 `product_count` BigQuery'deki `tex_iade_kargo_detay_urun_icerigi` tablosundan çekilir. Bu veri yoksa filtre atlanır.

**Filtre Özeti:**

```
┌──────────────────────────────────────────────────────┐
│  Filtre 1: Lokasyon     → Farklı merkezden geliyorsa el│
│  Filtre 2: Tekil        → 1 kbarcode = 1 paket         │
│  Filtre 3: %50 Sayı     → Çok az eşleşme varsa el       │
│                                                        │
│  Elenenler → analysis_results'a yazılır               │
│  (Retry adımında bu kbarcode'lar tekrar denenir)       │
└──────────────────────────────────────────────────────┘
```

---

### Adım 6 — 🧹 Cleanup *(Kalite Kontrol)*

Exact ve Fuzzy eşleştirmelerin hatalı olanları temizlenir. SKU, Link ve AI (AYNI/KESIN_AYNI) eşleştirmeleri bu adımdan **muaftır**.

```
┌─────────────────────────────────────────────────────────────────┐
│  A. Cinsiyet Uyumsuzluğu                    🔴 ÇOK KRİTİK      │
│     Warehouse: Kadın ceket                                       │
│     Package:   Erkek ceket    → ❌ SİL                          │
├─────────────────────────────────────────────────────────────────┤
│  B. Kategori Ters Eşleştirme                🔴 ÇOK KRİTİK      │
│     Warehouse: Elektronik                                        │
│     Package:   Giyim          → ❌ SİL                          │
├─────────────────────────────────────────────────────────────────┤
│  C. Marka + Beden Hatası                    🟠 YÜKSEK           │
│     Warehouse: Zara, beden M                                     │
│     Package:   H&M, beden XL  → ❌ SİL                          │
├─────────────────────────────────────────────────────────────────┤
│  D. İçerik Benzerliği Çok Düşük             🟡 ORTA             │
│     Ürün adı benzerliği < %50 → ❌ SİL                          │
└─────────────────────────────────────────────────────────────────┘
```

Silinen kayıtların nedeni `analysis_results` tablosuna yazılır (izlenebilirlik için).

**Örnek Cleanup Etkisi:**

```
Cinsiyet Uyumsuzluğu:      −450 kayıt  (−0.26%)
Kategori Ters Eşleştirme:  −85  kayıt  (−0.05%)
Marka-İçerik-Beden Hatası: −52  kayıt  (−0.03%)
İçerik Benzerliği Yok:     −30  kayıt  (−0.02%)
─────────────────────────────────────────────────
TOPLAM:                    −617 kayıt  (−0.36%)
```

---

### Adım 7 — 🔄 Retry *(Elenenler İçin İkinci Şans)*

Kalite filtreleri veya cleanup tarafından **elenen kbarcode'lar** tekrar pipeline'a sokulur.

```
analysis_results tablosundan elenen kbarcode'ları al
        │
        ▼
Hâlâ aktif eşleşmesi olanları çıkar
        │
        ▼
Daha önce denenen yanlış çiftleri engelle (rejected pairs)
        │
        ▼
SKU → Link → Exact → Fuzzy (aynı pipeline, farklı adaylarla)
        │
        ▼
Yeni eşleşmeler → historic'e incremental ekle
(mevcut is_current=true kayıtlara dokunmaz)
```

> 💡 Retry, ana pipeline ile aynı 3 kalite filtresini uygular. Farkı: başarılı retry olan kbarcode'ların eski filtre kayıtlarını da temizler.

---

### Adım 8 — 🧠 AI Final Matching *(Son Çare)*

Retry'dan sonra hâlâ eşleşemeyen kbarcode'lar için AI cache'indeki **50+ analiz alanı** field-weighted skorlama ile karşılaştırılır.

| Alan | Ağırlık |
|------|---------|
| `kategori` | %30 |
| `tip` | %25 |
| `renk` | %15 |
| `boyut / adet` | %10 |
| `enhanced_signature` | %20 |

> 🔍 **Marka kullanılmaz** — saha ekibi farklı marka girebilir.  
> 🎯 Eşik: **%40** — son şans adımı olduğu için düşük tutulur.

---

## 🔁 Manuel Doğrulama & Geri Bildirim Döngüsü

Sistem tamamen otomatik çalışır, ancak operatörler her eşleştirmeyi **onaylayabilir veya reddedebilir**. Bu geri bildirim bir sonraki pipeline çalıştırmasına doğrudan etki eder.

### `match_verified` Alanı

`missing_packages_match_historic` tablosundaki her eşleştirme için operatör şu değerlerden birini seçebilir:

| Değer | Anlam | Bir Sonraki Run'a Etkisi |
|-------|-------|--------------------------|
| `NULL` | Henüz incelenmedi (default) | Normal işlem görür |
| `'yes'` | Operatör: "Bu eşleştirme doğru" | kbarcode **tamamen atlanır** — zaten doğrulandı, yeniden işlenmez |
| `'no'` | Operatör: "Bu eşleştirme yanlış" | O (kbarcode, integration_code) çifti **kalıcı olarak engellenir** — aynı hata tekrar üretilmez |

### Akış

```
Operatör raporu inceler
        │
        ├── "Bu doğru" → match_verified = 'yes'
        │       └── Sonraki run: Bu kbarcode warehouse listesinden çıkarılır
        │           Pipeline onu görmez bile, zaman kazanılır
        │
        └── "Bu yanlış" → match_verified = 'no'
                └── Sonraki run: Bu (kbarcode, IC) çifti rejected_pairs'e eklenir
                    Pipeline bu çifti hiç üretmez
                    Kbarcode serbest kalır, başka bir pakete eşlenebilir
```

### Neden Önemli?

```
Run 1:  K111 → integration_code: 7330001  ← Yanlış eşleştirme
Operatör: match_verified = 'no'

Run 2:  K111 için 7330001 artık aday değil
        K111 → integration_code: 7330002  ← Doğru eşleştirme bulundu ✅
```

Sistem her çalıştırmada bu geri bildirimlerden **öğrenir** ve aynı hatayı tekrar yapmaz.

---

## 🤖 AI Kullanımı — Nereden Nereye?

Sistemde AI **iki farklı noktada, iki farklı amaçla** kullanılır:

```
┌──────────────────────────────────────────────────────────────┐
│  AI KULLANIM 1: Görsel Analiz (Her Görsel — Tek Seferlik)    │
│                                                              │
│  Endpoint: POST /ai/analyze-images                           │
│  Model: Trendyol AI API (Vision)                             │
│                                                              │
│  Amaç: Görselden ürün özelliklerini çıkar                    │
│  Girdi: Görsel URL + ürün metadatası (context)               │
│  Çıktı: 50+ alan (kategori, renk, beden, marka...)           │
│  Cache: PostgreSQL (ai_analysis_cache_temp)                  │
│  Maliyet: ~$0.002 / görsel (sadece ilk analiz)               │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  AI KULLANIM 2: Görsel Doğrulama (Çift Karşılaştırma)        │
│                                                              │
│  Endpoint: POST /ai-verification/fill-missing-verifications  │
│  Model: Trendyol AI API (Vision — çift görsel)               │
│                                                              │
│  Amaç: "Bu iki görsel aynı ürün mü?" sorusunu yanıtla        │
│  Girdi: Depo fotoğrafı + Katalog görseli + ürün bilgileri    │
│  Çıktı: KESIN_AYNI|95 / AYNI|72 / FARKLI|10                 │
│  Cache: PostgreSQL (ai_verification_cache)                   │
│  Maliyet: ~$0.0005 / karşılaştırma                           │
└──────────────────────────────────────────────────────────────┘
```

### AI'ın Zorluğu: Depo vs. Stüdyo

```
image1 (Depo Fotoğrafı)          image2 (Trendyol Katalog)
─────────────────────────        ──────────────────────────
📸 Elde tutulmuş                 📸 Profesyonel stüdyo
🏭 Arka plan: raflar, kutular    ⬜ Arka plan: beyaz/düz
💡 Depo aydınlatması             💡 Mükemmel ışık
📦 Ambalaj buruşuk olabilir      📦 Ürün kusursuz
🔄 Farklı açı                    🔄 Standart açı
```

AI'ya şu talimat verilir: *"Arka plan, ışık ve açı farklarını görmezden gel. Ürünün markasına, modeline, rengine ve bedenine odaklan."*

**Özel Senaryo — Çok Adetli Paket:**  
Katalog görseli 3'lü seti gösterirken depo fotoğrafı sadece 1 adet içerebilir. Bu durumda AI'ya özellikle belirtilir: *"Paket görseli set olabilir, depoda 1 tane görünüyorsa NORMAL — yine AYNI diyebilirsin."*

---

## 🗄️ Veritabanı Tabloları

```
PostgreSQL
├─ ai_analysis_cache_temp      → Görsel analiz sonuçları (cache)
├─ ai_verification_cache       → Eşleştirme adayları + doğrulama
├─ missing_packages_match      → Son run'ın sonuçları (TRUNCATE-INSERT)
├─ missing_packages_match_historic → Tüm geçmiş (versioning)
└─ analysis_results            → Eleme/hata kayıtları (izlenebilirlik)
```

### 📋 `ai_analysis_cache_temp`

Her görsel bir kez analiz edilir, sonuç burada saklanır.

| Kolon | Açıklama |
|-------|----------|
| `image_url` | Görsel URL (unique anahtar) |
| `kategori`, `tip`, `marka`, `renk` | AI çıktı alanları |
| `cinsiyet`, `malzeme`, `beden` | AI çıktı alanları |
| `enhanced_signature` | Tüm alanların birleşik özeti |
| `analysis_status` | `success` / `failed` |
| `source_type` | `package` / `warehouse` |

---

### 📋 `ai_verification_cache`

Python AI Matching sonuçları ve doğrulama durumu.

| Kolon | Açıklama |
|-------|----------|
| `kbarcode` | Warehouse barkodu |
| `integration_code` | Package kodu |
| `similarity_score` | Python AI benzerlik skoru (0–100) |
| `ai_verification` | `KESIN_AYNI` / `AYNI` / `FARKLI` / NULL |
| `ai_confidence` | AI'ın verdiği güven skoru (0–100) |
| `package_brand_name`, `package_product_content_name`, `package_size` | Paket metadatası |
| `kbarcode_brand_name`, `kbarcode_product_content_name`, `kbarcode_size` | Depo metadatası |

---

### 📋 `missing_packages_match`

Her çalıştırmada sıfırlanır (TRUNCATE), pipeline çıktısını tutar.

| Kolon | Açıklama |
|-------|----------|
| `kbarcode` | Eşleşen depo ürünü |
| `integration_code` | Eşleşen paket |
| `source` | `sku_matching` / `link_matching` / `exact_matching` / `fuzzy_matching` / `ai_matching` |
| `match_rule` | Hangi kuralın eşleştirdiği |
| `similarity_score` | Benzerlik skoru (varsa) |
| `ai_verification` | AI doğrulama sonucu (varsa) |

---

### 📋 `missing_packages_match_historic`

Tüm geçmiş eşleştirmeler, versioning ile.

| Kolon | Açıklama |
|-------|----------|
| *(missing_packages_match kolonları)* | |
| `is_current` | Bu çift son run'da da eşleşti mi? |
| `data_date` | İlk görülme tarihi — **değişmez** |
| `report_update_date` | Son güncelleme |

```sql
-- Son run'daki tüm eşleştirmeler
SELECT * FROM missing_packages_match_historic
WHERE is_current = true;

-- Bir kbarcode'un geçmişi
SELECT data_date, is_current, match_rule, source
FROM missing_packages_match_historic
WHERE kbarcode = 'K1763989382665'
ORDER BY updated_at DESC;
```

---

## 📈 Güvenilirlik Hiyerarşisi

Eşleştirme kuralları güvenilirliğe göre sıralanmış:

```
En Güvenilir                                        En Az Güvenilir
     │                                                      │
     ▼                                                      ▼
  🔑 SKU+Beden   🔗 Link   📍 İçerik+Marka+Beden   🎯 Fuzzy   🤖 AI
     %98            %100         %90                  %75-85     %90-100
     │              │            │                    │           │
     │              │            │                    │           │
  MUAF           MUAF        Cleanup'a             Cleanup'a   MUAF
  (cleanup yok)              girer                 girer
```

**Özet Tablo:**

| Eşleştirme | Güvenilirlik | Cleanup Muaf? |
|------------|:---:|:---:|
| 🔑 SKU + Beden | ⭐⭐⭐⭐⭐ %98 | ✅ Muaf |
| 🔑 SKU + Marka | ⭐⭐⭐⭐⭐ %98 | ✅ Muaf |
| 🔑 SKU (tek) | ⭐⭐⭐⭐⭐ %95 | ✅ Muaf |
| 🔗 Link (Trendyol ID) | ⭐⭐⭐⭐⭐ %100 | ✅ Muaf |
| 📍 Ad + Marka + Beden | ⭐⭐⭐⭐ %90 | ❌ Cleanup'a girer |
| 📍 Ad + Beden | ⭐⭐⭐⭐ %85 | ❌ Cleanup'a girer |
| 📍 Ad (tek) | ⭐⭐⭐ %75 | ❌ Cleanup'a girer |
| 🎯 Fuzzy (≥%95) | ⭐⭐⭐⭐ %85 | ❌ Cleanup'a girer |
| 🎯 Fuzzy (≥%80) | ⭐⭐⭐ %75 | ❌ Cleanup'a girer |
| 🤖 AI (KESIN_AYNI) | ⭐⭐⭐⭐⭐ %100 | ✅ Muaf |
| 🤖 AI (AYNI) | ⭐⭐⭐⭐ %90 | ✅ Muaf |

---

## 🚀 Production Pipeline — Tam Akış

```
ADIM 1: AI Görsel Analiz Cache (bağımsız çalışır)
────────────────────────────────────────────────────
  POST /ai/analyze-images
  • Warehouse + Package görsellerini AI'a gönder
  • Sonuçları ai_analysis_cache_temp'e yaz
  • Bir kez yapılır, sonrası cache'den gelir

          ↓ (tamamlandıktan sonra)

ADIM 2: Ana Eşleştirme Pipeline
────────────────────────────────────────────────────
  POST /matching/enhanced
  a) missing_packages_match TRUNCATE
  b) match_verified='yes' kbarcode'ları atla
  c) match_verified='no' çiftlerini engelle
  d) SKU → Link → Exact → Fuzzy eşleştirme
  e) Kalite filtreleri: Lokasyon, Tekil, %50 ürün sayısı
  f) Elenenleri analysis_results'a yaz
  g) missing_packages_match'e kaydet
  h) Historic tabloya kopyala

          ↓

ADIM 3: AI Verification
────────────────────────────────────────────────────
  POST /ai-verification/fill-missing-verifications
  • ai_verification_cache'deki NULL kayıtları bul
  • İki görseli yan yana AI'a gönder
  • KESIN_AYNI / AYNI / FARKLI sonucunu yaz

          ↓

ADIM 4: Cleanup
────────────────────────────────────────────────────
  POST /simple-cleanup/clean-historic
  • Cinsiyet uyumsuzluğu → SİL
  • Kategori ters eşleştirme → SİL
  • Marka-İçerik-Beden hatası → SİL
  • İçerik benzerliği düşük → SİL
  • SKU/Link/AI(AYNI,KESIN_AYNI) → MUAF

          ↓

ADIM 5: Retry
────────────────────────────────────────────────────
  POST /matching/retry
  • analysis_results'tan elenen kbarcode'ları al
  • Eski yanlış çiftleri engelle
  • SKU → Link → Exact → Fuzzy (yeni adaylarla)
  • Başarılı olanları historic'e incremental ekle

          ↓

ADIM 6: AI Final Matching
────────────────────────────────────────────────────
  POST /matching/final
  • Hâlâ eşleşemeyenleri AI cache verisiyle karşılaştır
  • Field-weighted skorlama (%70 içerik, %20 imza, ...)
  • Eşik: %40 (son şans, düşük)
  • Sonuçları historic'e incremental ekle
```

---

## 📊 Gerçek Veri Metrikleri (Örnek Run)

### Eşleştirme Dağılımı

```
Kaynak               Kural                                  Adet     Oran
───────────────────────────────────────────────────────────────────────────
sku_matching         SKU + Beden                           19,318   11.1%
sku_matching         SKU + Marka                              438    0.3%
sku_matching         SKU (tek)                                 17    0.0%
link_matching        Trendyol ürün ID                         680    0.4%
exact_matching       Ad + Marka + Beden                       317    0.2%
exact_matching       Ad + Beden                           150,053   86.3%
exact_matching       Ad (tek)                               1,453    0.8%
fuzzy_matching       SKU benzerliği                           234    0.1%
fuzzy_matching       Marka + İçerik                           456    0.3%
fuzzy_matching       İçerik benzerliği                        512    0.3%
fuzzy_matching       SKU + İçerik                             184    0.1%
ai_matching          AI (KESIN_AYNI)                            8    0.0%
ai_matching          AI (AYNI)                                 20    0.0%
───────────────────────────────────────────────────────────────────────────
TOPLAM                                                    173,690  100%
Cleanup sonrası                                           173,073  (-617)
```

### Pipeline Performansı

| Aşama | Süre | Kayıt |
|-------|------|-------|
| BigQuery veri çekme | ~20 sn | 114,368 |
| SKU + Link Matching | ~2 sn | ~20,000 |
| Exact Matching | ~10 sn | ~152,000 |
| Fuzzy Matching | ~7 dk | ~1,386 |
| AI Görsel Analiz | ~15 dk | ~82,000 (cache'de yoksa) |
| Python AI Matching | ~25 dk | ~20,000 |
| AI Verification | ~5 dk | ~2,000 |
| Cleanup | ~5 sn | −617 |
| Historic Kayıt | ~10 sn | ~173,500 |
| **TOPLAM** | **~50 dk** | **173,073** |

### AI Maliyet Analizi

```
İlk Çalıştırma:
  Görsel Analiz:   ~82,000 görsel × $0.002  = $164
  AI Verification: ~2,000 çift   × $0.0005  = $1
  BigQuery:        ~5 GB         × $0.01    = $0.05
  ─────────────────────────────────────────────────
  TOPLAM:                                  ~$165

Sonraki Çalıştırmalar (cache sayesinde):
  Yeni görseller:  ~500 görsel   × $0.002   = $1
  AI Verification: ~200 çift     × $0.0005  = $0.1
  ─────────────────────────────────────────────────
  TOPLAM:                                  ~$1.1
```

---

## 🔌 API Referansı

### AI Görsel Analiz

```bash
# Package tablosu için (context ile)
curl -X POST http://localhost:8087/ai/analyze-images \
  -H "Content-Type: application/json" \
  -d '{
    "custom_table": "dsm-data.data_acceptance_test.package_base_v1",
    "image_column": "image_url",
    "source_type": "package",
    "max_workers": 200,
    "context_columns": ["brand_name", "product_content_name", "size"],
    "context_reliable": true
  }'

# Warehouse tablosu için
curl -X POST http://localhost:8087/ai/analyze-images \
  -H "Content-Type: application/json" \
  -d '{
    "custom_table": "dsm-data.data_acceptance_test.warehouse_base_v1",
    "image_column": "warehouse_image_url",
    "source_type": "warehouse",
    "max_workers": 200,
    "context_columns": ["brand_name", "product_content_name", "size"],
    "context_reliable": false
  }'
```

### Ana Eşleştirme

```bash
curl -X POST http://localhost:8087/matching/enhanced \
  -H "Content-Type: application/json" \
  -d '{
    "credentials_path": "./dwh-po-prod.json"
  }'
```

### AI Verification

```bash
curl -X POST "http://localhost:8087/ai-verification/fill-missing-verifications?limit=30000"
```

### Cleanup

```bash
curl -X POST http://localhost:8087/simple-cleanup/clean-historic
```

### Retry

```bash
curl -X POST http://localhost:8087/matching/retry
```

### AI Final Matching

```bash
curl -X POST http://localhost:8087/matching/final
```

---

## 📊 Raporlama & Çıktı

### Looker Studio Dashboard

Tüm pipeline tamamlandıktan sonra sonuçlar **Looker Studio**'daki canlı rapora otomatik yansır. Rapor doğrudan `missing_packages_match_historic` tablosunu kaynak olarak kullanır.

```
PostgreSQL
  missing_packages_match_historic
          │
          │  (canlı bağlantı)
          ▼
  📊 Looker Studio Dashboard
          │
          ├── Günlük eşleştirme sayısı
          ├── Kaynak bazlı dağılım (SKU / Exact / Fuzzy / AI)
          ├── Lokasyon & merkez bazlı kırılım
          ├── is_current = true filtresi (sadece aktif eşleştirmeler)
          └── Export → Excel/CSV → Saha ekibi kullanımı
```

### Saha Ekibi Ne Görür?

| Alan | Örnek | Açıklama |
|------|-------|----------|
| `kbarcode` | K1763989382665 | Ürünün depo barkodu |
| `integration_code` | 7330026227808172 | Eşleşen müşteri paketi |
| `match_rule` | SKU + Beden | Hangi yöntemle eşleştirildi |
| `kbarcode_brand_name` | Zara | Depodaki ürünün markası |
| `package_brand_name` | Zara | Paketin markası |
| `sorting_center_name` | ESENYURT_SC | Hangi merkezde |
| `data_date` | 2025-11-26 | İlk görülme tarihi |
| `is_current` | true | Bu run'da da aktif mi? |

### Export & Aksiyon

```
Rapor → Export al (Excel)
          │
          ▼
Saha ekibi listeyi alır
          │
          ├── Eşleşen ürünler → müşteri paketine yönlendirilir
          └── Eşleşmeyen ürünler → bir sonraki run'a bırakılır
                                   veya manuel inceleme
```

> 🔗 Dashboard: [Looker Studio — Barkodsuz Rapor](https://lookerstudio.google.com/reporting/90e851d4-ed88-4c8c-a312-9a6a943f6805)

---

## 🎯 Özet: Neden Bu Kadar Katman?

Her katman, öncekinin çözemediği bir problemi çözer:

```
🔑 SKU Matching     → Aynı tedarikçi kodu? En kesin, en hızlı.
🔗 Link Matching    → Trendyol ID tuttu mu? %100 güven.
📍 Exact Matching   → Metin birebir mi? Hızlı, yüksek hacim.
🎯 Fuzzy Matching   → Yazım hatası / kelime sırası farkı var mı? Tolere et.
🤖 AI Matching      → Metin de işe yaramadı mı? Görsele bak.
🧹 Cleanup          → Kalan hataları çıkar.
🔄 Retry            → Elenenler için ikinci şans.
🧠 AI Final         → Son şans — AI cache + field scoring.
```

**Sonuç:** Manuel olarak günler alan süreç, ~50 dakikada ~173,000 eşleştirme ile tamamlanır.

---

## 🖥️ Production Pipeline — Manuel Çalıştırma Sırası

Pipeline adımları bağımsız endpoint'ler olarak ayrılmıştır. Her adım tamamlandıktan sonra sıradaki tetiklenir.

> **Neden ayrı?** Cleanup adımı (Adım 3), Adım 8 (AI Final Matching) öncesinde çalışmalıdır. Eğer otomatik birleştirilseydi hatalı eşleştirmeler AI Final'e girdi olurdu.

---

### Adım 1 — Matching Pipeline (Adım 1–7)

SKU → Link → Exact → Fuzzy → AI Matching → AI Verification → Bulk Matching + Historic Sync

```bash
curl -X POST http://localhost:8087/matching/enhanced \
  -H "Content-Type: application/json" \
  -d '{
    "credentials_path": "./dwh-po-prod.json"
  }'
```

> Tablo belirtilmezse `warehouse_base_v1` ve `package_base_v1` default olarak kullanılır.

---

### Adım 2 — Eksik AI Verification'ları Doldur

`ai_verification_cache`'deki NULL sonuçlarını AI ile doldurur (AYNI / FARKLI / KESIN_AYNI).

```bash
curl -X POST "http://localhost:8087/ai-verification/fill-missing-verifications?limit=30000&batch_size=50"
```

---

### Adım 3 — Cleanup (Hatalı Eşleştirmeleri Temizle)

Cinsiyet uyumsuzluğu, kategori çakışması gibi kural tabanlı hataları `missing_packages_match_historic`'ten siler.

```bash
curl -X POST http://localhost:8087/simple-cleanup/clean-historic
```

---

### Adım 4 — Retry (Elenen Kayıtları Yeniden Dene)

Cleanup veya kalite filtreleri tarafından elenen kbarcode'lar için Adım 1–7 mantığını tekrar çalıştırır.

```bash
curl -X POST http://localhost:8087/matching/retry \
  -H "Content-Type: application/json" \
  -d '{
    "credentials_path": "./dwh-po-prod.json"
  }'
```

---

### Adım 5 — AI Final Matching (Adım 8 — Tur 1)

Hâlâ eşleşmemiş kayıtlar için `product_content_name` metin skoru + AI cache field skoru + marka bonusu ile son çare eşleştirme.

```bash
curl -X POST http://localhost:8087/matching/final \
  -H "Content-Type: application/json" \
  -d '{
    "credentials_path": "./dwh-po-prod.json"
  }'
```

---

### Adım 6 — AI Final Match Review — Tur 1 (Sanity Check)

Adım 8 çıktılarını AI'a tekrar gösterir. `ALAKASIZ` bulunanları siler, `analysis_results`'a yazar.

```bash
curl -X POST "http://localhost:8087/ai-verification/final-match-review?round_num=1&max_workers=30"
```

---

### Adım 7 — Retry (ALAKASIZ Silinenleri Yeniden Dene)

Review'da `ALAKASIZ` çıkıp silinen kbarcode'lar için tekrar Adım 1–7 mantığı.

```bash
curl -X POST http://localhost:8087/matching/retry \
  -H "Content-Type: application/json" \
  -d '{
    "credentials_path": "./dwh-po-prod.json"
  }'
```

---

### Adım 8 — AI Final Matching (Adım 8 — Tur 2)

Retry sonrası hâlâ eşleşmeyenler için ikinci AI Final Matching turu.

```bash
curl -X POST http://localhost:8087/matching/final \
  -H "Content-Type: application/json" \
  -d '{
    "credentials_path": "./dwh-po-prod.json"
  }'
```

---

### Adım 9 — AI Final Match Review — Tur 2

```bash
curl -X POST "http://localhost:8087/ai-verification/final-match-review?round_num=2&max_workers=30"
```

---

### Adım 10 — AI Final Matching Sonuçlarını Historic'e Ekle

`MANTIKLI` olarak onaylanan Adım 8 eşleştirmelerini `missing_packages_match_historic`'e yaz.

```bash
curl -X POST http://localhost:8087/ai-verification/final-historic-sync
```

---

### Adım 11 — BigQuery Sync

PostgreSQL'deki final tabloyu BigQuery'e kopyalar, Looker Studio raporu güncellenir.

```bash
curl -X POST http://localhost:8087/sync/postgres-to-bigquery \
  -H "Content-Type: application/json" \
  -d '{
    "credentials_path": "./dwh-po-prod.json"
  }'
```

---

### 🔧 Recovery: Pipeline Ortasında Patladıysa

`missing_packages_match` dolu ama historic kopyalanamadıysa (örn. kolon eksikliği), sadece sync adımını çalıştır:

```bash
curl -X POST http://localhost:8087/matching/sync-historic \
  -H "Content-Type: application/json" \
  -d '{
    "credentials_path": "./dwh-po-prod.json"
  }'
```

Sonrasında normal akışa (Adım 2'den) devam et.

---

### 📋 Tam Akış — Özet Tablo

| # | Endpoint | Açıklama |
|---|----------|----------|
| 1 | `POST /matching/enhanced` | Adım 1–7 + Historic Sync |
| 2 | `POST /ai-verification/fill-missing-verifications` | Eksik AI verification doldur |
| 3 | `POST /simple-cleanup/clean-historic` | Hatalı eşleştirmeleri sil |
| 4 | `POST /matching/retry` | Elenen kayıtları yeniden dene |
| 5 | `POST /matching/final` | AI Final Matching — Tur 1 |
| 6 | `POST /ai-verification/final-match-review?round_num=1` | Tur 1 sanity check |
| 7 | `POST /matching/retry` | ALAKASIZ silinenleri yeniden dene |
| 8 | `POST /matching/final` | AI Final Matching — Tur 2 |
| 9 | `POST /ai-verification/final-match-review?round_num=2` | Tur 2 sanity check |
| 10 | `POST /ai-verification/final-historic-sync` | Adım 8 sonuçlarını historic'e ekle |
| 11 | `POST /sync/postgres-to-bigquery` | BigQuery & Looker güncelle |
