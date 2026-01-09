# 🧹 Cleanup API Endpoints

Enhanced Cleanup Services için REST API endpoints.

## 📋 Endpoints

### 1. 🚀 Enhanced Cleanup Pipeline Çalıştır

```http
POST /cleanup/run-enhanced-cleanup?limit=1000&dry_run=true
```

**Parametreler:**
- `limit` (int): Her analiz için maksimum kayıt sayısı (default: 1000)
- `dry_run` (bool): Sadece analiz yap, gerçek silme yapma (default: true)

**Response:**
```json
{
  "success": true,
  "message": "Enhanced Cleanup pipeline başlatıldı (background task)",
  "initial_count": 5432,
  "limit": 1000,
  "dry_run": true,
  "status": "processing"
}
```

**İşlem Akışı:**
1. `missing_packages_match` tablosundaki verileri analiz eder
2. 6 katmanlı cleanup işlemlerini çalıştırır:
   - Q1: AI Mantık Kontrolü
   - Q2: Size/Content Analizi
   - Q3: Kategori Kontrolü
   - Q4: Enhanced Similarity Analysis
   - Q5: Enhanced Category Analysis
   - Q6: Enhanced Brand Analysis
3. `dry_run=false` ise temizlenen verileri `missing_packages_match_historic`'e taşır

---

### 2. 🔍 Cleanup Durumu

```http
GET /cleanup/status
```

**Response:**
```json
{
  "success": true,
  "current_table": {
    "name": "missing_packages_match",
    "count": 1234
  },
  "historic_table": {
    "name": "missing_packages_match_historic",
    "count": 45678
  },
  "ai_verification_stats": [
    {"verification": "KESIN_AYNI", "count": 123},
    {"verification": "AYNI", "count": 456},
    {"verification": "BENZER", "count": 78},
    {"verification": "FARKLI", "count": 90},
    {"verification": "NULL", "count": 487}
  ],
  "match_rule_stats": [
    {"rule": "AI Görsel ile eşleştirme", "count": 234},
    {"rule": "exact matching", "count": 567},
    {"rule": "fuzzy matching", "count": 123}
  ],
  "timestamp": "2025-12-03T09:45:00.123456"
}
```

---

### 3. 📦 Manuel Historic'e Taşıma

```http
POST /cleanup/manual-move-to-historic
```

**Response:**
```json
{
  "success": true,
  "message": "1234 kayıt historic tabloya taşınıyor (background task)",
  "current_count": 1234,
  "status": "processing"
}
```

**Not:** Bu endpoint sadece veri taşıma yapar, cleanup yapmaz.

---

### 4. 🗑️ Current Tablo Temizleme

```http
DELETE /cleanup/truncate-current?confirm=true
```

**Parametreler:**
- `confirm` (bool): **ZORUNLU** - True olmalı, aksi halde işlem yapılmaz

**Response:**
```json
{
  "success": true,
  "message": "missing_packages_match tablosu temizlendi",
  "deleted_count": 1234
}
```

**⚠️ DİKKAT:** Bu işlem geri alınamaz!

---

### 5. 🧪 Enhanced Analyzer Test

```http
GET /cleanup/test-enhanced-analyzers?limit=10
```

**Parametreler:**
- `limit` (int): Test için kayıt sayısı (default: 10)

**Response:**
```json
{
  "success": true,
  "message": "Enhanced Analyzer test tamamlandı",
  "test_limit": 10,
  "dry_run": true,
  "results": {
    "q1_ai_logic": 2,
    "q2_size_content": 1,
    "q3_category": 0,
    "q4_enhanced_similarity": 3,
    "q5_enhanced_category": 1,
    "q6_enhanced_brand": 2,
    "total_deleted": 9
  },
  "note": "Bu test hiçbir veriyi silmez, sadece analiz yapar"
}
```

---

## 🔄 Kullanım Senaryoları

### Senaryo 1: Test Çalıştırması
```bash
# 1. Önce test et
curl -X GET "http://localhost:8000/cleanup/test-enhanced-analyzers?limit=50"

# 2. Durumu kontrol et
curl -X GET "http://localhost:8000/cleanup/status"

# 3. Dry run ile gerçek veri üzerinde test
curl -X POST "http://localhost:8000/cleanup/run-enhanced-cleanup?limit=100&dry_run=true"
```

### Senaryo 2: Production Cleanup
```bash
# 1. Durumu kontrol et
curl -X GET "http://localhost:8000/cleanup/status"

# 2. Gerçek cleanup çalıştır
curl -X POST "http://localhost:8000/cleanup/run-enhanced-cleanup?limit=5000&dry_run=false"

# 3. Sonuçları kontrol et
curl -X GET "http://localhost:8000/cleanup/status"
```

### Senaryo 3: Manuel Veri Yönetimi
```bash
# 1. Manuel historic'e taşı
curl -X POST "http://localhost:8000/cleanup/manual-move-to-historic"

# 2. Current tabloyu temizle (DİKKAT!)
curl -X DELETE "http://localhost:8000/cleanup/truncate-current?confirm=true"
```

---

## 🛡️ Güvenlik Önlemleri

- ✅ **Default DRY RUN:** Tüm cleanup işlemleri varsayılan olarak `dry_run=true`
- ✅ **Background Tasks:** Uzun işlemler background'da çalışır
- ✅ **Confirmation Required:** Tehlikeli işlemler için `confirm=true` gerekli
- ✅ **Detailed Logging:** Tüm işlemler detaylı loglanır
- ✅ **Status Monitoring:** Real-time durum takibi
- ✅ **Error Handling:** Comprehensive error responses

---

## 📊 Cleanup Pipeline Detayları

### 6 Katmanlı Analiz:

1. **Q1: AI Mantık Kontrolü**
   - AI eşleştirmelerinde kategori/renk/cinsiyet uyumsuzluğu
   - Cache'de eksik görsel analizi tespiti

2. **Q2: Size/Content Analizi**
   - Akıllı size kontrolü (S-M ↔ M uyumlu)
   - Product content benzerlik analizi

3. **Q3: Kategori Kontrolü**
   - AI cache'deki kategori bilgilerini karşılaştırma
   - Warehouse vs Package kategori uyumsuzluğu

4. **Q4: Enhanced Similarity Analysis**
   - Levenshtein distance, Jaccard similarity
   - Fuzzy string matching (%80 threshold)
   - Türkçe karakter normalizasyonu

5. **Q5: Enhanced Category Analysis**
   - Kontekst bazlı kategori tespiti
   - Renk uyumluluk kontrolü
   - Cinsiyet uyumsuzluk analizi

6. **Q6: Enhanced Brand Analysis**
   - Brand normalizasyon (Nike, Samsung, H&M vb.)
   - Akıllı size kontrolü
   - Content similarity analizi

---

## 🔗 Entegrasyon

Bu API, ana Barkodsuz matching pipeline'ına entegre edilmiştir:

1. **Matching Pipeline** → Eşleştirmeler `missing_packages_match`'e yazılır
2. **Enhanced Cleanup** → Sıkıntılı eşleştirmeler temizlenir
3. **Historic Transfer** → Temiz veriler `missing_packages_match_historic`'e taşınır

**Ana endpoint:** `POST /matching/enhanced` → Otomatik cleanup dahil
