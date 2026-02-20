# Barkodsuz - Kurallar, Filtreler ve Ozel Durumlar

> Eslestirme stratejilerinin detayi icin bkz: [MATCHING_HIERARCHY.md](./MATCHING_HIERARCHY.md)
> Bu dokumanda **preprocessing**, **kalite filtreleri**, **ozel durumlar**, **historic tablo yonetimi** ve stratejilerin teknik detaylari yer alir.

---

## Veri Kaynaklari

| Tablo | Icerik | Anahtar Alan |
|-------|--------|--------------|
| `warehouse_base_v2` | Depodaki fiziksel urunler | `kbarcode` |
| `package_base_v1` | Siparis paketleri | `integration_code` |

---

## Preprocessing (On Isleme)

Eslestirme baslamadan once her iki tabloya da su donusumler uygulanir:

| Alan | Donusum |
|------|---------|
| `brand_name` | `lowercase + strip` |
| `product_content_name` | `lowercase + strip` |
| `size` | BQ sorgusunda `LOWER(TRIM(size))` — http linkler, `-`, `null`, bos degerler → NULL |
| `brand_name` (ozel) | BQ sorgusunda: `Trendyolmilla` → `TRENDYOLMİLLA`, `bilinmiyor` → NULL, `diger` → `Diger`, `Pull*` → `Pull & Bear` |

### Blacklist

Google Sheets'ten dinamik okunan `integration_code` listesi. Bu kodlar package tablosundan **filtrelenir** ve hicbir stratejiyle eslestirmeye dahil edilmez. Service account ile `GOOGLE_APPLICATION_CREDENTIALS.json` uzerinden okunur.

---

## Strateji Teknik Detaylari

### Link Matching: URL Parsing

**URL formati:** `https://www.trendyol.com/.../urun-adi-p-12345?v=VARIANT`

**ID cikarma kurallari:**
- `-p-(\d+)` regex ile product ID alinir (ornek: `-p-12345` → `12345`)
- `?v=xxx` varsa variant bilgisi eklenir → `12345_v_xxx`
- Iki tarafta da ayni product ID varsa eslesme yapilir

**Tekillestirme:** Ayni kbarcode birden fazla paket ile eslesirse, `last_occurrence_datetime`'a gore en eski tarihli paket secilir.

### Fuzzy Matching: Aday Filtreleme ve Threshold'lar

Production'da binlerce warehouse x yuz binlerce package oldugu icin her paketi her warehouse ile karsilastirmak cok yavas olur. Bu yuzden once adaylar daraltilir:

**Aday Filtreleme:**
- Her paket icin, sadece **brand ilk 3 karakter** VEYA **SKU ilk 3 karakter** eslesen warehouse kayitlari aday olarak secilir
- Maksimum **200 aday** ile sinirlanir
- Aday bulunamazsa tum warehouse havuzu kullanilir
- `9999` ile baslayan SKU'lar icin sadece **brand-only filtreleme** uygulanir

**Threshold'lar:**

| Parametre | Deger | Aciklama |
|-----------|-------|----------|
| `brand_threshold` | 85 | Brand benzerlik skoru minimum %85 |
| `content_threshold` | 85 | Content benzerlik skoru minimum %85 |
| `sku_fuzzy_threshold` | 60 | SKU3+content icin content minimum %60 |
| `levenshtein_threshold` | 20 | SKU3+content icin brand Levenshtein mesafesi max 20 |

**Fuzzy Kurallar (oncelik sirasina gore):**

**Kural 1 — SKU Startswith:**

```
wh_sku == pkg_sku
VEYA wh_sku.startswith(pkg_sku)
VEYA pkg_sku.startswith(wh_sku)
```

`9999` ile baslayan SKU'lar icin bu kural **atlanir**.

**Kural 2 — Brand + Content Fuzzy:**

```
fuzz.ratio(wh_brand, pkg_brand) >= 85
VE fuzz.ratio(wh_content, pkg_content) >= 85
```

Birden fazla aday varsa en yuksek content skoru olan secilir.

**Kural 3 — Content-Only Fuzzy:**

```
fuzz.ratio(wh_content, pkg_content) > 85
```

Brand eslesmese bile content tek basina yeterince benzerse eslesme yapilir.

**Kural 4 — SKU3 + Content + Levenshtein:**

```
wh_sku[:3] == pkg_sku[:3]                   (SKU ilk 3 karakter ayni)
VE fuzz.ratio(wh_content, pkg_content) > 60  (content %60+ benzer)
VE Levenshtein(wh_brand, pkg_brand) <= 20    (brand mesafesi max 20)
```

`999` ile baslayan SKU'lar icin bu kural da **atlanir**.

### AI Matching: Kategori Bazli Similarity

ADIM 5'teki AI Matching, `ai_analysis_cache_temp` tablosundaki `enhanced_signature` verisini kullanarak kategori bazli benzerlik hesaplar.

**Similarity Formulu:**

```
overall_similarity = (
    critical_rate    x 0.60 +   -- Kritik alanlar (kategori, tip, renk, cinsiyet vb.)
    important_rate   x 0.25 +   -- Onemli alanlar (marka, malzeme, boyut vb.)
    optional_rate    x 0.10 +   -- Opsiyonel alanlar (desen, stil vb.)
    signature_sim    x 0.05     -- Enhanced signature benzerlik
)
```

**Kategori-Ozel Kritik Alanlar:**

| Kategori | Kritik Alanlar |
|----------|---------------|
| Giyim | kategori, tip, renk, cinsiyet, giyim_uzunluk, giyim_kesim |
| Kozmetik | kategori, tip, kozmetik_adet, uygulama_alani |
| Elektronik | kategori, tip, elektronik_adet, model_numarasi |
| Ayakkabi | kategori, tip, cinsiyet, ayakkabi_tipi, topuk_yuksekligi |
| Gida | kategori, tip, gida_adet, gida_tipi |
| Kitap | kategori, tip, kitap_isim, kitab_yazar |

**Universal kritik alanlar (tum kategoriler):** `hacim`, `agirlik`, `adet_sayisi` — bunlar eslesmedigi durumda direkt **red**.

**Minimum threshold:** `overall_similarity >= 30%` olan eslesmeler `ai_verification_cache` tablosuna yazilir.

---

## Kalite Filtreleri (Post-Processing)

ADIM 1-4'un birlestirilen sonuclarina 3 kalite filtresi sirayla uygulanir. Bu filtreler **ADIM 5-6 ve Trendyol link eslestirmelerinden muaftir**.

Filtreler `services/matching/post_processing.py` icindeki `apply_quality_filters()` fonksiyonunda uygulanir.

### Filtre 1: Tam Paket Kontrolu

```
matched_count == package_count
```

Bir pakette N urun varsa (`package_count=N`), o `integration_code`'a tam N tane farkli `kbarcode` eslesmiş olmalidir.

**Ornek:** Pakette 3 urun var ama sadece 2'si eslesti → bu paket **elenir**.

**Tersi de gecerli:** Pakette 1 urun var ama 5 farkli kbarcode eslesti → tutarsiz, **elenir**.

### Filtre 2: Ayni Lokasyon Kontrolu

```
location_count == 1
```

Bir `integration_code`'a eslesen tum `kbarcode`'lar ayni depo lokasyonunda (`location_wh`) olmalidir. Farkli lokasyonlardan gelen eslesmeler **elenir**.

**Ornek:** IC-001'e eslesen 3 kbarcode var, 2'si A lokasyonunda, 1'i B lokasyonunda → tum IC-001 eslesmesi **elenir**.

### Filtre 3: Tekil Eslestirme

```
1 kbarcode → 1 integration_code
```

Her `kbarcode` sadece 1 `integration_code` ile eslesebilir. Birden fazla eslesme varsa, `match_rule`'a gore siralanir ve en yuksek oncelikli kalan kalir (duplikatsizlestirme).

### Filtre Akisi

```
Tum eslestirme sonuclari (SKU + Link + Exact + Fuzzy)
    |
    +-- Trendyol link eslestirmeleri → MUAF (direkt final sonuca)
    |
    +-- Diger eslestirmeler
         |
         +-- Filtre 1: Tam Paket (matched_count == package_count) → elenenler cikar
         |
         +-- Filtre 2: Ayni Lokasyon (tek lokasyon) → elenenler cikar
         |
         +-- Filtre 3: Tekil Eslestirme (1 kbarcode → 1 IC) → duplikatsizlastir
              |
              +-- Final sonuc → missing_packages_match'e kaydet
```

---

## Ozel Durumlar ve Kurallar

### 9999 SKU Kurali

`9999` ile baslayan SKU'lar marketplace urunleridir. Bu urunler icin:
- Fuzzy ADIM 4'te **SKU startswith** kurali atlanir
- Fuzzy ADIM 4'te **SKU3 + content** kurali atlanir
- Aday filtrelemede sadece **brand-only filtreleme** uygulanir (brand ilk 3 karakter)

### Trendyol Link Muafiyeti

Trendyol link eslestirmeleri (`match_rule = "Trendyol urun ID ile eslestirme"`):
- Kalite filtrelerinden **muaf** tutulur
- Product ID bazli oldugu icin %100 guvenilir kabul edilir

---

## Historic Tablo Yonetimi

Eslestirme sonuclari uc tabloda yonetilir:

### 1. `missing_packages_match` (Truncate-Insert)

Her calistirmada **truncate** edilir ve o calistirmanin sonuclari yazilir. Source degerleri:

| Source | Kaynak |
|--------|--------|
| `sku_matching` | ADIM 1 |
| `link_matching` | ADIM 2 |
| `exact_matching` | ADIM 3 |
| `fuzzy_matching` | ADIM 4 |
| `ai_matching` | ADIM 5+6 (AYNI/KESIN_AYNI olanlar) |

### 2. `missing_packages_match_historic` (Versiyon Kontrolu)

Gecmis eslestirmeleri `is_current` flag ile yonetir:

1. Yeni calistirma basladiginda → Tum `is_current=true` kayitlari `is_current=false` yapilir
2. Her eslesme icin:
   - Daha once **gorulmus** → Sadece `is_current=true` yapilir (`data_date` **degismez** — ilk gorulme tarihi korunur)
   - **Gorulmemis** → Yeni kayit eklenir (`is_current=true`, bugunun tarihi)

### 3. `ai_verification_cache` (AI Dogrulama)

AI eslestirme ve dogrulama sonuclarini tutar:

1. Ilk eslestirme → `similarity_score` hesaplanir
2. Score >= 50 olanlar → AI verification icin Trendyol AI'a gonderilir
3. Sonuclar (AYNI, KESIN_AYNI, FARKLI) guncellenir
4. AYNI veya KESIN_AYNI olanlar → `missing_packages_match`'e transfer edilir
