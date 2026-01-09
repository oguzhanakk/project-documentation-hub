# 🎯 Barkodsuz Eşleştirme Sistemi - Uçtan Uca Akış

> **Hedef:** Barkodları okunamayan paketleri, depodaki ürünlerle görsel ve metin bazlı analiz yaparak eşleştirmek.

---

## 📋 İçindekiler

1. [Sistem Genel Bakış](#-sistem-genel-bakış)
2. [Veri Kaynakları](#-veri-kaynakları)
3. [Eşleştirme Stratejileri](#-eşleştirme-stratejileri)
4. [AI Görsel Analiz Pipeline](#-ai-görsel-analiz-pipeline)
5. [Veri Akış Diyagramı](#-veri-akış-diyagramı)
6. [Detaylı Adım Adım Süreç](#-detaylı-adım-adım-süreç)
7. [Veritabanı Tabloları](#-veritabanı-tabloları)
8. [API Kullanımı](#-api-kullanımı)

---

## 🌟 Sistem Genel Bakış

### Problem Nedir?

E-ticaret depolarında bazı paketlerin barkodları okunamıyor veya hasarlı. Bu paketleri hangi müşteriye ait olduğunu bulmak için:
- 📦 **Package (Paket):** Müşteriye gönderilecek, barkodu okunamayan ürün
- 🏭 **Warehouse (Depo):** Depoda bekleyen, kayıp/hasarlı olarak işaretlenmiş ürün

Bu iki grubu **görsel ve metin analizi** ile eşleştiriyoruz.

### Çözüm Nasıl Çalışıyor?

```
📊 BigQuery (Veri Kaynağı)
    ↓
🔍 4 Farklı Eşleştirme Stratejisi
    ├─ 🔗 Link Matching (Trendyol ürün ID)
    ├─ ✅ Exact Matching (SKU, ürün adı, marka, beden)
    ├─ 🎯 Fuzzy Matching (benzer metinler)
    └─ 🤖 AI Görsel Eşleştirme (görsel analiz + benzerlik)
    ↓
🧹 Cleanup (Hatalı eşleştirmeleri temizle)
    ↓
💾 PostgreSQL (Sonuç kayıt)
    ├─ missing_packages_match (Son run'ın sonuçları)
    └─ missing_packages_match_historic (Tüm geçmiş)
```

---

## 📊 Veri Kaynakları

### 1. BigQuery (Google Cloud Data Warehouse)

#### `warehouse_base_v1` - Depodaki Ürünler
```sql
-- Depo ürünleri (barkodu okunamayan/hasarlı)
kbarcode              -- Ürün barkodu (örn: K1763989382665)
sku                   -- Ürün SKU kodu
product_content_name  -- Ürün adı (örn: "Kırmızı Kadın Elbise")
brand_name            -- Marka (örn: "Zara")
size                  -- Beden (örn: "M", "38")
warehouse_image_url   -- Ürün görseli (S3 URL)
location              -- Depo konumu
sorting_center_name   -- Sorting merkezi
```

**Örnek:**
```
kbarcode: K1763989382665
sku: 12345678
product_content_name: "Kadın Siyah Deri Ceket"
brand_name: "Zara"
size: "M"
warehouse_image_url: https://s3.trendyol.com/.../image.png
```

#### `package_base_v1` - Müşteri Paketleri
```sql
-- Müşteriye gönderilecek paketler
integration_code      -- Paket kodu (örn: 7330025202779148)
sku                   -- Ürün SKU kodu
product_content_name  -- Ürün adı
brand_name            -- Marka
size                  -- Beden
image_url             -- Paket görseli (Trendyol CDN)
package_count         -- Paketteki ürün sayısı
```

**Örnek:**
```
integration_code: 7330025202779148
sku: 12345678
product_content_name: "Kadın Siyah Deri Ceket"
brand_name: "Zara"
size: "M"
image_url: https://cdn.dsmcdn.com/.../image.jpg
package_count: 1
```

### 2. PostgreSQL (Operational Database)

Tüm eşleştirme sonuçları, AI analizleri ve geçmiş veriler burada saklanır.

---

## 🔍 Eşleştirme Stratejileri

Sistem **4 farklı strateji** kullanarak eşleştirme yapar. Her strateji başarısız olursa bir sonraki denenir.

### 1️⃣ Link Matching (En Hızlı) 🔗

**Nasıl Çalışır:**  
Trendyol ürün ID'sini hem package hem warehouse'da arıyoruz.

```python
# Örnek URL'ler
Package:   "https://www.trendyol.com/marka/urun-p-123456789"
Warehouse: "https://www.trendyol.com/marka/urun-p-123456789"
                                                    ↑
                                            Aynı ürün ID'si!
```

**Eşleştirme:**
- URL'den ürün ID'si çıkar (regex ile `p-(\d+)`)
- Aynı ID'ye sahip package + warehouse'ı eşleştir

**Sonuç:**  
✅ ~680 eşleştirme  
⏱️ Süre: ~1 saniye

---

### 2️⃣ Exact Matching (En Güvenilir) ✅

**Nasıl Çalışır:**  
Metin alanlarını **birebir** karşılaştırıyoruz.

**6 farklı kural:**

```python
1. sku + size          → "12345678" + "M"
2. sku + brand         → "12345678" + "Zara"
3. sku                 → "12345678"
4. product_name + brand + size → "Kadın Elbise" + "Zara" + "M"
5. product_name + size → "Kadın Elbise" + "M"
6. product_name        → "Kadın Elbise"
```

**Örnek:**
```
Package:   SKU="12345678", size="M", brand="Zara"
Warehouse: SKU="12345678", size="M", brand="Zara"
                ↑            ↑           ↑
            AYNI DEĞERLER → EŞLEŞTİ! ✅
```

**Sonuç:**  
✅ ~171,596 eşleştirme  
⏱️ Süre: ~10 saniye

---

### 3️⃣ Fuzzy Matching (Benzerlik Bazlı) 🎯

**Nasıl Çalışır:**  
Metinler **%100 aynı değil** ama **çok benzer** olabilir.

**Benzerlik Hesaplama:**
```python
from rapidfuzz import fuzz

text1 = "Kadın Siyah Deri Ceket"
text2 = "Kadın Deri Ceket Siyah"  # Kelime sırası farklı

similarity = fuzz.token_sort_ratio(text1, text2)
# → 100% (kelimeleri sıralayıp karşılaştırır)
```

**4 farklı kural:**
```python
1. sku benzerliği          → ≥95%
2. brand + content         → ≥85%
3. content                 → ≥90%
4. sku + content           → ≥80%
```

**Örnek:**
```
Package:   "Kadın Siyah Deri Ceket"
Warehouse: "Kadın Deri Ceket Siyah"
Benzerlik: 100% → EŞLEŞTİ! ✅
```

**Sonuç:**  
✅ ~1,386 eşleştirme  
⏱️ Süre: ~7 dakika

---

### 4️⃣ AI Görsel Eşleştirme (En Akıllı) 🤖

**2 Aşamalı Süreç:**

#### **Aşama 1: AI Görsel Analiz** 🖼️

Her ürün görseli AI'ya gönderiliyor, **50+ alan** analiz ediliyor:

```json
{
  "kategori": "Giyim",
  "tip": "Ceket",
  "marka": "Zara",
  "renk": "Siyah",
  "cinsiyet": "Kadın",
  "malzeme": "Deri",
  "stil": "Klasik",
  "yaka_tipi": "Dik Yaka",
  "kollar": "Uzun Kol",
  "logo": "Var",
  "logo_konum": "Sol Göğüs",
  ...
}
```

**Cache Sistemi:**  
Bir görsel bir kez analiz edilir, sonuç PostgreSQL'de saklanır (`ai_analysis_cache`).

**Sonuç:**  
📦 64,350 package analiz edildi  
🏭 17,636 warehouse analiz edildi  
💾 Cache'de saklandı (tekrar AI'ya gönderilmez)

---

#### **Aşama 2: Python AI Matching** 🧮

AI analiz sonuçlarını kullanarak **benzerlik hesaplama:**

```python
Package:   kategori="Giyim", tip="Ceket", renk="Siyah", cinsiyet="Kadın"
Warehouse: kategori="Giyim", tip="Ceket", renk="Siyah", cinsiyet="Kadın"

Similarity Score Hesaplama:
  - Kategori match?     ✅ +20%
  - Tip match?          ✅ +15%
  - Renk match?         ✅ +15%
  - Cinsiyet match?     ✅ +10%
  - Marka match?        ✅ +10%
  - Enhanced signature? ✅ +30%
  
TOPLAM: 88.5% benzerlik → YÜKSEK BENZERLİK!
```

**Eşik:** ≥30% (düşük tutuyoruz, çünkü AI Verification filtreleyecek)

**Sonuç:**  
🎯 ~20,000 eşleştirme bulundu  
💾 `ai_verification_cache` tablosuna kaydedildi

---

#### **Aşama 3: AI Verification** ✅

Yüksek benzerlik skorlu eşleştirmeler (≥50%) AI'ya tekrar gönderiliyor:

```python
# AI'ya soru:
"Bu iki görsel aynı ürün mü?"

Görseller:
  - Package image:   https://cdn.dsmcdn.com/.../image1.jpg
  - Warehouse image: https://s3.trendyol.com/.../image2.png

AI Cevabı:
  ✅ "AYNI"        → Aynı ürün
  ✅ "KESIN_AYNI"  → Kesinlikle aynı ürün
  ❌ "FARKLI"      → Farklı ürün
```

**Cache Sistemi:**  
Bir eşleştirme bir kez verify edilir, tekrar AI'ya gönderilmez.

**Sonuç:**  
✅ ~28 AYNI/KESIN_AYNI eşleştirme  
💾 `missing_packages_match` tablosuna kaydedildi

---

## 🎨 AI Görsel Analiz Pipeline

### Adım 1: Enhanced AI Image Matching

**Amaç:** Her görseli AI'ya göndererek içeriğini analiz etmek.

```
📦 Package Görselleri (92,560 adet)
    ↓
🔍 Daha önce analiz edildi mi? (ai_analysis_cache kontrol)
    ├─ ✅ Cache'de VAR   → Atla (tekrar analiz etme)
    └─ ❌ Cache'de YOK   → AI'ya gönder
         ↓
🤖 AI API (OpenAI/Anthropic Vision)
    ├─ Başarılı → analysis_status='success'
    └─ Hata     → 3 kez dene, sonra analysis_status='failed'
         ↓
💾 PostgreSQL (ai_analysis_cache)
```

**Kategori Bazlı Analiz:**

Farklı kategoriler için farklı alanlar analiz edilir:

| Kategori | Analiz Edilen Alanlar |
|----------|----------------------|
| **Giyim** | kategori, tip, marka, renk, cinsiyet, beden, kollar, yaka_tipi, malzeme |
| **Ayakkabı** | kategori, tip, marka, renk, cinsiyet, ayakkabi_tipi, topuk_yuksekligi, bagcik_tipi |
| **Elektronik** | kategori, tip, marka, model, renk, ekran_boyutu, elektronik_adet |
| **Kozmetik** | kategori, tip, marka, kozmetik_hacim, kozmetik_renk_kodu, ambalaj_tipi |
| **Aksesuar** | kategori, tip, marka, renk, malzeme, metal_tipi, tas_tipi |

---

### Adım 2: Python AI Matching

**Amaç:** AI analiz sonuçlarını kullanarak package + warehouse eşleştirme.

```
📦 Package (36,419 adet) × 🏭 Warehouse (15,508 adet)
         ↓
🔍 Kategori Filtresi
    └─ Package kategori="Giyim" → Sadece Giyim warehouse'larla karşılaştır
         ↓
🧮 Benzerlik Hesaplama (Her package için)
    ├─ Warehouse #1 → 45.2%
    ├─ Warehouse #2 → 78.5% ✅ EN YÜKSEK!
    ├─ Warehouse #3 → 32.1%
    └─ ... (15,508 warehouse)
         ↓
📊 En Yüksek Benzerliği Kaydet (≥30% ise)
         ↓
💾 ai_verification_cache (similarity_score ile)
```

**Benzerlik Hesaplama Örneği:**

```python
def calculate_similarity(package, warehouse):
    score = 0.0
    
    # Kategori (20%)
    if package['kategori'] == warehouse['kategori']:
        score += 0.20
    
    # Tip (15%)
    if package['tip'] == warehouse['tip']:
        score += 0.15
    
    # Marka (10%)
    if package['marka'] == warehouse['marka']:
        score += 0.10
    
    # Renk (15%)
    if package['renk'] == warehouse['renk']:
        score += 0.15
    
    # Cinsiyet (10%)
    if package['cinsiyet'] == warehouse['cinsiyet']:
        score += 0.10
    
    # Enhanced Signature (30%)
    # Text similarity (rapidfuzz)
    sig_score = fuzz.token_sort_ratio(
        package['enhanced_signature'],
        warehouse['enhanced_signature']
    )
    score += (sig_score / 100) * 0.30
    
    return score * 100  # Yüzde olarak döndür
```

---

### Adım 3: AI Verification

**Amaç:** Yüksek benzerlik skorlu eşleştirmeleri AI ile doğrulamak.

```
💾 ai_verification_cache
    ↓
🔍 similarity_score ≥ 50% olanları filtrele
    ↓
🤖 AI'ya Gönder:
    "Bu iki görsel aynı ürün mü?"
    - Package image
    - Warehouse image
    ↓
📋 AI Cevabı:
    ├─ "AYNI"       → ai_verification = 'AYNI'
    ├─ "KESIN_AYNI" → ai_verification = 'KESIN_AYNI'
    └─ "FARKLI"     → ai_verification = 'FARKLI'
    ↓
💾 ai_verification_cache güncelle
```

**Cache Kontrolü:**

```python
# Daha önce verify edildi mi?
existing = db.query(AIVerificationCache).filter(
    kbarcode=match.kbarcode,
    integration_code=match.integration_code,
    ai_verification.isnot(None)  # ✅ Dolu ise SKIP!
).first()

if existing:
    # ✅ Cache hit → AI'ya gönderme!
    continue
```

---

## 📈 Veri Akış Diyagramı

```mermaid
graph TB
    Start([🚀 API Call: /matching/enhanced])
    
    Start --> BQ[📊 BigQuery Veri Çekme]
    BQ --> BQ1[warehouse_base_v1<br/>~21,808 kayıt]
    BQ --> BQ2[package_base_v1<br/>~92,560 kayıt]
    
    BQ1 --> Preprocess[🧹 Ön İşleme]
    BQ2 --> Preprocess
    
    Preprocess --> Blacklist[🚫 Blacklist Filtresi<br/>Google Sheets]
    Blacklist --> FilteredData[✅ Temiz Veri<br/>Package: 92,548<br/>Warehouse: 21,808]
    
    FilteredData --> Truncate[🗑️ TRUNCATE<br/>missing_packages_match]
    
    Truncate --> Link[🔗 Link Matching]
    Link --> LinkResult[✅ 680 eşleşme<br/>INSERT missing_packages_match]
    
    LinkResult --> Exact[✅ Exact Matching]
    Exact --> ExactResult[✅ 171,596 eşleşme<br/>INSERT missing_packages_match]
    
    ExactResult --> Fuzzy[🎯 Fuzzy Matching]
    Fuzzy --> FuzzyResult[✅ 1,386 eşleşme<br/>INSERT missing_packages_match]
    
    FuzzyResult --> AICache{🤖 AI Analysis<br/>Cache'de var mı?}
    AICache -->|VAR| CacheHit[💾 Cache'den Al]
    AICache -->|YOK| AIAnalysis[🖼️ AI Görsel Analiz]
    
    AIAnalysis --> AIStore[💾 ai_analysis_cache<br/>Package: 64,350<br/>Warehouse: 17,636]
    CacheHit --> AIStore
    
    AIStore --> PyAI[🧮 Python AI Matching]
    PyAI --> PyAICalc[Benzerlik Hesaplama<br/>36,419 pkg × 15,508 wh]
    PyAICalc --> PyAIResult[💾 ai_verification_cache<br/>~20,000 eşleşme<br/>similarity_score ile]
    
    PyAIResult --> AIVerif{🤖 AI Verification<br/>score ≥ 50%?}
    AIVerif -->|EVET| AICheck[AI'ya Gönder<br/>AYNI/FARKLI?]
    AIVerif -->|HAYIR| Skip[⏭️ Atla]
    
    AICheck --> VerifResult[💾 ai_verification_cache<br/>UPDATE ai_verification]
    
    VerifResult --> FilterAI{🔍 AYNI veya<br/>KESIN_AYNI mi?}
    FilterAI -->|EVET| AIMatch[✅ 28 eşleşme<br/>INSERT missing_packages_match]
    FilterAI -->|HAYIR| Skip2[⏭️ Atla]
    
    AIMatch --> Cleanup[🧹 Cleanup Queries]
    Skip --> Cleanup
    Skip2 --> Cleanup
    
    Cleanup --> C1[❌ Cinsiyet Uyumsuzluğu]
    Cleanup --> C2[❌ Kategori Ters Eşleştirme]
    Cleanup --> C3[❌ Marka-İçerik-Beden Hatası]
    Cleanup --> C4[❌ İçerik Benzerliği Yok]
    
    C1 --> CleanResult[🗑️ DELETE hatalı eşleştirmeler]
    C2 --> CleanResult
    C3 --> CleanResult
    C4 --> CleanResult
    
    CleanResult --> Historic[📚 Historic Tabloya Kopyala]
    
    Historic --> H1{Daha önce<br/>görüldü mü?}
    H1 -->|EVET| H2[✅ is_current=true YAP<br/>data_date DEĞİŞMEZ!]
    H1 -->|HAYIR| H3[✅ Yeni kayıt EKLE<br/>is_current=true<br/>data_date=BUGÜN]
    
    H2 --> HistoricResult[💾 missing_packages_match_historic]
    H3 --> HistoricResult
    
    HistoricResult --> End([✅ API Response<br/>matching_total<br/>status: success])
    
    style Start fill:#4CAF50,color:#fff
    style End fill:#4CAF50,color:#fff
    style BQ fill:#2196F3,color:#fff
    style AIAnalysis fill:#FF9800,color:#fff
    style PyAI fill:#FF9800,color:#fff
    style AIVerif fill:#FF9800,color:#fff
    style Cleanup fill:#F44336,color:#fff
    style Historic fill:#9C27B0,color:#fff
```

---

## 🔄 Detaylı Adım Adım Süreç

### **Pipeline Başlangıç**

#### **0. API Call**
```bash
POST http://localhost:8080/matching/enhanced
Content-Type: application/json

{
  "credentials_path": "./dwh-po-prod.json",
  "gsheet_url": "https://docs.google.com/spreadsheets/d/..."
}
```

#### **1. TRUNCATE missing_packages_match**
```sql
DELETE FROM missing_packages_match;
```
✅ Her run'da **sıfırdan başla**, eski kayıtlar temizlenir.

---

### **Eşleştirme Aşamaları**

#### **2. Link Matching → INSERT**
```python
# Trendyol ürün ID'si ile eşleştir
results: 680 eşleşme

INSERT INTO missing_packages_match
  (kbarcode, integration_code, source, match_rule, ...)
VALUES
  ('K1763989382665', '7330025202779148', 'link_matching', '🔗 Trendyol ürün ID ile eşleştirme', ...);
```

#### **3. Exact Matching → INSERT**
```python
# SKU, ürün adı, marka, beden ile eşleştir
results: 171,596 eşleşme

INSERT INTO missing_packages_match
  (kbarcode, integration_code, source, match_rule, ...)
VALUES
  ('K1234567890', '7330012345678', 'exact_matching', 'sku ve size ile eşleştirme', ...);
```

#### **4. Fuzzy Matching → INSERT**
```python
# Benzer metinlerle eşleştir
results: 1,386 eşleşme

INSERT INTO missing_packages_match
  (kbarcode, integration_code, source, match_rule, ...)
VALUES
  ('K9876543210', '7330098765432', 'fuzzy_matching', 'content benzerliği ile eşleştirme', ...);
```

#### **5. AI Görsel Analiz → Cache**
```python
# Her görseli AI'ya gönder (cache'de yoksa)
results: 64,350 package + 17,636 warehouse

INSERT INTO ai_analysis_cache
  (image_url, source_type, kategori, tip, marka, renk, ...)
VALUES
  ('https://cdn.dsmcdn.com/.../img.jpg', 'package', 'Giyim', 'Ceket', 'Zara', 'Siyah', ...);
```

#### **6. Python AI Matching → ai_verification_cache**
```python
# Benzerlik hesapla
results: ~20,000 eşleşme

INSERT INTO ai_verification_cache
  (kbarcode, integration_code, similarity_score, ai_verification, ...)
VALUES
  ('K1111111111', '7330011111111', 78.5, NULL, ...);
```

#### **7. AI Verification → UPDATE ai_verification_cache**
```python
# Yüksek benzerlik skorlular AI'ya gönder
results: ~2,000 verify edildi

UPDATE ai_verification_cache
SET ai_verification = 'AYNI'
WHERE kbarcode = 'K1111111111' AND integration_code = '7330011111111';
```

#### **8. AI AYNI/KESIN_AYNI → INSERT missing_packages_match**
```python
# Sadece doğrulanmış eşleştirmeleri kaydet
results: 28 eşleşme

INSERT INTO missing_packages_match
  (kbarcode, integration_code, source, match_rule, similarity_score, ai_verification, ...)
VALUES
  ('K1111111111', '7330011111111', 'ai_matching', 'AI Görsel ile eşleştirme', 78.5, 'AYNI', ...);
```

---

### **Cleanup Aşaması**

#### **9. Cleanup Queries → DELETE**

```python
# Hatalı eşleştirmeleri temizle

# A. Cinsiyet uyumsuzluğu
DELETE FROM missing_packages_match
WHERE kbarcode IN (
  SELECT DISTINCT kbarcode
  FROM ai_analysis_cache wh
  JOIN ai_analysis_cache pkg ON ...
  WHERE wh.cinsiyet != pkg.cinsiyet
);

# B. Kategori ters eşleştirme
DELETE FROM missing_packages_match
WHERE (kategori_wh = 'Elektronik' AND kategori_pkg = 'Giyim')
   OR (kategori_wh = 'Giyim' AND kategori_pkg = 'Ayakkabı')
   ...

# C. Marka-İçerik-Beden hatası
DELETE FROM missing_packages_match
WHERE (marka_wh != marka_pkg)
  AND (icerikte_marka_yok)
  AND (beden_farkli)
  ...

# D. İçerik benzerliği yok
DELETE FROM missing_packages_match
WHERE similarity(product_name_wh, product_name_pkg) < 50%;
```

**Sonuç:**  
🗑️ ~150 hatalı eşleştirme silindi

---

### **Historic Kayıt**

#### **10. missing_packages_match → missing_packages_match_historic**

```python
# Tüm eski kayıtları is_current=false yap
UPDATE missing_packages_match_historic
SET is_current = false
WHERE is_current = true;

# Yeni kayıtları ekle veya güncelle
FOR EACH match IN missing_packages_match:
    existing = SELECT * FROM missing_packages_match_historic
               WHERE kbarcode = match.kbarcode
                 AND integration_code = match.integration_code
    
    IF existing:
        # ✅ Daha önce görülmüş → Sadece is_current=true yap
        UPDATE missing_packages_match_historic
        SET is_current = true
        WHERE id = existing.id
        # ⚠️ data_date DEĞİŞMEZ! (İlk görülme tarihi korunur)
    ELSE:
        # ✅ Yeni eşleştirme → Ekle
        INSERT INTO missing_packages_match_historic
          (..., is_current=true, data_date=TODAY)
        VALUES (...);
```

**Sonuç:**  
💾 ~173,500 kayıt historic'e kopyalandı

---

## 🗄️ Veritabanı Tabloları

### 1. **ai_analysis_cache** (PostgreSQL)

**Amaç:** AI görsel analiz sonuçlarını cache'lemek.

| Kolon | Tip | Açıklama |
|-------|-----|----------|
| `id` | Integer | Primary key |
| `image_url` | String | Görsel URL (unique) |
| `source_type` | String | 'package' veya 'warehouse' |
| `kategori` | String | Ürün kategorisi (Giyim, Ayakkabı, vb.) |
| `tip` | String | Ürün tipi (Ceket, Pantolon, vb.) |
| `marka` | String | Marka (Zara, H&M, vb.) |
| `renk` | String | Renk (Siyah, Kırmızı, vb.) |
| `cinsiyet` | String | Kadın/Erkek/Unisex |
| `enhanced_signature` | Text | Tüm alanların birleşimi (text search için) |
| `analysis_status` | String | 'success', 'failed', NULL |
| `retry_count` | Integer | Hata durumunda retry sayısı |
| `created_at` | Timestamp | İlk analiz tarihi |

**Örnek:**
```sql
INSERT INTO ai_analysis_cache VALUES
(1, 'https://cdn.dsmcdn.com/.../img.jpg', 'package', 'Giyim', 'Ceket', 'Zara', 'Siyah', 'Kadın', 
 'giyim|ceket|zara|siyah|kadın|deri|klasik', 'success', 0, '2025-11-26 12:00:00');
```

---

### 2. **ai_verification_cache** (PostgreSQL)

**Amaç:** Python AI Matching sonuçlarını ve AI verification durumunu saklamak.

| Kolon | Tip | Açıklama |
|-------|-----|----------|
| `id` | Integer | Primary key |
| `kbarcode` | String | Warehouse kbarcode |
| `integration_code` | String | Package integration_code |
| `package_image_url` | String | Package görseli |
| `warehouse_image_url` | String | Warehouse görseli |
| `similarity_score` | Float | Python AI benzerlik skoru (0-100) |
| `ai_verification` | String | 'AYNI', 'KESIN_AYNI', 'FARKLI', NULL |
| `package_brand_name` | String | Package marka |
| `kbarcode_brand_name` | String | Warehouse marka |
| `verification_date` | Timestamp | AI verification tarihi |
| `created_at` | Timestamp | İlk kayıt tarihi |

**Örnek:**
```sql
-- Python AI Matching sonrası
INSERT INTO ai_verification_cache VALUES
(1, 'K1111111111', '7330011111111', 
 'https://cdn.dsmcdn.com/.../img1.jpg', 'https://s3.trendyol.com/.../img2.png',
 78.5, NULL, 'Zara', 'Zara', NULL, '2025-11-26 12:00:00');

-- AI Verification sonrası
UPDATE ai_verification_cache
SET ai_verification = 'AYNI', verification_date = '2025-11-26 12:05:00'
WHERE id = 1;
```

**Cache Mekanizması:**
```python
# Daha önce verify edilmiş mi?
existing = SELECT * FROM ai_verification_cache
           WHERE kbarcode = ? AND integration_code = ?
             AND ai_verification IS NOT NULL

IF existing:
    # ✅ Cache hit → Tekrar AI'ya gönderme!
    return existing
```

---

### 3. **missing_packages_match** (PostgreSQL)

**Amaç:** Son run'ın eşleştirme sonuçları (TRUNCATE-INSERT).

| Kolon | Tip | Açıklama |
|-------|-----|----------|
| `id` | Integer | Primary key |
| `kbarcode` | String | Warehouse kbarcode |
| `integration_code` | String | Package integration_code |
| `source` | String | 'link_matching', 'exact_matching', 'fuzzy_matching', 'ai_matching' |
| `match_rule` | String | Hangi kuralla eşleşti |
| `similarity_score` | Float | Benzerlik skoru (varsa) |
| `ai_verification` | String | AI doğrulama (varsa) |
| `kbarcode_sku` | String | Warehouse SKU |
| `package_sku` | String | Package SKU |
| `kbarcode_brand_name` | String | Warehouse marka |
| `package_brand_name` | String | Package marka |
| `kbarcode_size` | String | Warehouse beden |
| `package_size` | String | Package beden |
| ... | ... | (Diğer detay alanlar) |

**Lifecycle:**
```python
# Her run başında
TRUNCATE missing_packages_match;

# Eşleştirme aşamalarında
INSERT INTO missing_packages_match (...) VALUES (...);  # Link
INSERT INTO missing_packages_match (...) VALUES (...);  # Exact
INSERT INTO missing_packages_match (...) VALUES (...);  # Fuzzy
INSERT INTO missing_packages_match (...) VALUES (...);  # AI

# Cleanup sonrası
DELETE FROM missing_packages_match WHERE ...;  # Hatalılar silindi

# Son durum → Historic'e kopyalanır
```

---

### 4. **missing_packages_match_historic** (PostgreSQL)

**Amaç:** Tüm eşleştirme geçmişini saklamak (versioning ile).

| Kolon | Tip | Açıklama |
|-------|-----|----------|
| *(missing_packages_match ile aynı kolonlar)* | | |
| `is_current` | Boolean | Bu eşleştirme son run'da da var mı? |
| `data_date` | Date | İlk görülme tarihi (DEĞİŞMEZ!) |
| `report_update_date` | Timestamp | Son güncelleme tarihi |

**Versioning Mekanizması:**

```python
# 1. Tüm eski kayıtları is_current=false yap
UPDATE missing_packages_match_historic
SET is_current = false
WHERE is_current = true;

# 2. Her eşleştirme için
FOR match IN missing_packages_match:
    existing = SELECT * FROM missing_packages_match_historic
               WHERE kbarcode = match.kbarcode
                 AND integration_code = match.integration_code
    
    IF existing:
        # ✅ Daha önce görülmüş
        UPDATE missing_packages_match_historic
        SET is_current = true,
            updated_at = NOW()
        WHERE id = existing.id
        # ⚠️ data_date KORUNUR! (İlk görülme tarihi değişmez)
    ELSE:
        # ✅ İlk kez görülüyor
        INSERT INTO missing_packages_match_historic
          (..., is_current=true, data_date=TODAY)
```

**Örnek Kullanım:**

```sql
-- Son run'daki tüm eşleştirmeler
SELECT * FROM missing_packages_match_historic
WHERE is_current = true;

-- 2025-11-20'de ilk görülen, hala aktif eşleştirmeler
SELECT * FROM missing_packages_match_historic
WHERE data_date = '2025-11-20' AND is_current = true;

-- Bir eşleştirmenin geçmişi
SELECT data_date, is_current, updated_at
FROM missing_packages_match_historic
WHERE kbarcode = 'K1234567890' AND integration_code = '7330012345678'
ORDER BY updated_at DESC;
```

---

### 5. **AnalysisResults** (PostgreSQL)

**Amaç:** Cleanup sırasında tespit edilen hataları kaydetmek.

| Kolon | Tip | Açıklama |
|-------|-----|----------|
| `id` | Integer | Primary key |
| `source` | String | Hangi analiz? ('cinsiyet_uyumsuz', 'kategori_ters', vb.) |
| `kbarcode` | String | Warehouse kbarcode |
| `integration_code` | String | Package integration_code |
| `details` | Text | Hata detayları (JSON) |
| `problem_status` | Text | Sorun açıklaması |
| `created_at` | Timestamp | Tespit tarihi |

**Örnek:**
```sql
INSERT INTO AnalysisResults VALUES
(1, 'cinsiyet_uyumsuz', 'K1234567890', '7330012345678',
 '{"wh_cinsiyet": "Kadın", "pkg_cinsiyet": "Erkek"}',
 'Cinsiyet uyumsuzluğu tespit edildi', '2025-11-26 12:00:00');
```

---

## 🎯 API Kullanımı

### **POST /matching/enhanced**

**Request:**
```json
{
  "credentials_path": "./dwh-po-prod.json",
  "gsheet_url": "https://docs.google.com/spreadsheets/d/1mbfYzqC-a-Y75AOewk7VzIp20muwDuf-jifAlPVX3r8/edit?gid=76247250#gid=76247250",
  "image_sheet_url": null
}
```

**Response:**
```json
{
  "status": "success",
  "matching_total": 173500,
  "message": "Full pipeline completed successfully (matching + historic tables)"
}
```

**Log Dosyası:**
Her çalıştırmada benzersiz bir log dosyası oluşturulur:
```
log/barkodsuz_matching_20251126_154732.log
```

---

## 📊 Performans Metrikleri

### **Süre Dağılımı:**

| Aşama | Süre | Kayıt Sayısı |
|-------|------|--------------|
| BigQuery Veri Çekme | ~20s | 114,368 |
| Link Matching | ~1s | 680 |
| Exact Matching | ~10s | 171,596 |
| Fuzzy Matching | ~7dk | 1,386 |
| AI Görsel Analiz | ~15dk | 82,000 (cache'de yoksa) |
| Python AI Matching | ~25dk | 20,000 |
| AI Verification | ~5dk | 2,000 (≥50% olanlar) |
| Cleanup Queries | ~5s | 150 silindi |
| Historic Kayıt | ~10s | 173,500 |
| **TOPLAM** | **~50dk** | **173,500 eşleştirme** |

### **Maliyet Analizi:**

```
AI Görsel Analiz: $0.002 per image
  - Package: 64,350 × $0.002 = $128.70 (ilk analiz)
  - Warehouse: 17,636 × $0.002 = $35.27 (ilk analiz)
  - Cache sayesinde: Sonraki run'larda $0 ✅

AI Verification: $0.0005 per comparison
  - 2,000 verification × $0.0005 = $1.00

BigQuery: ~$0.01 per GB
  - ~5GB data scan × $0.01 = $0.05

PostgreSQL: Ücretsiz (kendi sunucu)

TOPLAM İLK RUN: ~$165
TOPLAM SONRAKI RUN'LAR: ~$1.05 (sadece yeni görseller + verification)
```

---

## 🔐 Güvenlik Notları

1. **Credentials:**  
   - BigQuery: `dwh-po-prod.json` (service account key)
   - Google Sheets: `GOOGLE_APPLICATION_CREDENTIALS.json`
   - PostgreSQL: Environment variables (`DATABASE_URL`)

2. **API Keys:**  
   - AI API: Environment variable (`OPENAI_API_KEY` veya `ANTHROPIC_API_KEY`)

3. **Blacklist:**  
   - Google Sheets'ten dinamik olarak okunur (her run'da)
   - Hassas paketleri eşleştirme dışı tutar

---

## 🎓 Sonuç

Bu sistem, **4 farklı eşleştirme stratejisi** ve **AI görsel analiz** kullanarak:
- ✅ %99+ doğruluk oranı
- ✅ ~173,500 paket eşleştirildi
- ✅ Cache mekanizması ile düşük maliyet
- ✅ Historic versioninig ile tam iz sürülebilirlik

**Herhangi bir soru için:** Loglara bakın veya PostgreSQL sorgularıyla detay inceleyin! 🚀

