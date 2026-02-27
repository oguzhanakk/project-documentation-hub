# 1. Test Sonrası Geliştirmeler (25.02.2026)

> **Kaynak:** 1. full test sonuçları ve manuel eşleştirme ekibiyle karşılaştırma
> **Hedef:** [Hatalı eşleşme sheeti](https://docs.google.com/spreadsheets/d/1WSMVnzbCXhUE-LRoRJcV0zgDbSI0caQEaKJZw_x4Of4/edit?gid=618115467#gid=618115467)'ndeki 235 hatalı/eksik eşleşmeyi düzeltmek
> **Referans:** [Manuel eşleştirme detay sheeti](https://docs.google.com/spreadsheets/d/11c2YolY6rJBMhU-KrUtPP1rJsehSN_zRAsJfLY4BYlc/edit?gid=1818166144#gid=1818166144)

### Geliştirmelerin Kaynağı

| Madde | Tespit Kaynağı |
|-------|---------------|
| 1. Çoklu ürün paket kaybı | Manuel eşleştirme sheeti ile karşılaştırma (doğru eşleşmeler arasında tespit) |
| 2. AI hatalı koruma | Yanlış eşleştirme sheetinden tespit |
| 3. Doğrulanmış eşleşme yönetimi | 1. test → 2. test geçiş ihtiyacı |
| 4. Ürün sayısı yanlış tablo | Yanlış eşleştirme sheetinden tespit, saha ekibiyle (Burak) doğrulama |
| 5. Yasaklı Statü + 24 Saat | Saha ekibi (Burak) bildirimi |
| 6. Threshold problemi | Manuel eşleştirme detay sheeti analizi |
| 7. AI limitasyonları | AI cache analizi |

> **Not:** Manuel eşleştirmeler her zaman sağlıklı değil — detaya bakıldığında alakasız örnekler de mevcut (markalar farklı, içerikler alakasız). Ancak sağlıklı olanlar üzerinden yapılan karşılaştırma, yukarıdaki geliştirmelerin tespitinde kritik rol oynadı.

---

## 1. Çoklu Ürün İçeren Paketlerde Eşleşme Kaybı

**Ne oldu:**
Bir gönderide birden fazla ürün olduğunda (örn: 3 masa örtüsü tek pakette), sistem ilk ürünü eşleştirdikten sonra o gönderiyi havuzdan çıkarıyordu. Geri kalan 2 ürün eşleşecek gönderi bulamıyordu.

**Neden yapıldı:**
Manuel eşleştirme ekibi aynı gönderiyle 3 farklı kbarcode'u eşleştirmişti, ama sistem sadece 1 tanesini buluyordu. Çoklu ürün içeren paketlerde eşleşme kaybı yaşanıyordu.

**Ne yapıldı:**
Eşleştirme pipeline'ında bir gönderi (integration_code) eşleştikten sonra havuzdan çıkarılıyordu. Bu mantık değiştirildi: artık sadece depo ürünü (kbarcode) havuzdan çıkıyor, gönderi havuzda kalıyor. Böylece aynı gönderiyle birden fazla depo ürünü eşleşebiliyor. Kural: 1 depo ürünü → 1 gönderi, ama 1 gönderi → N depo ürünü.

**Ne bekliyoruz:**
Özellikle çoklu ürün siparişlerinde eşleşme sayısı artacak. Manuel eşleştirme ekibinin bulduğu çoklu eşleşmeler artık sistem tarafından da yakalanacak.

---

## 2. AI Eşleştirmelerinde Hatalı Koruma

**Ne oldu:**
Sistem, AI'ın "aynı ürün" dediği eşleşmeleri kalite kontrolünden (cleanup) muaf tutuyordu. Ancak AI eşleştirme zaten sadece "aynı" dediği ürünleri eşleştirdiği için, tüm AI eşleşmeleri otomatik olarak korunuyordu — yanlış olanlar dahil.

**Neden yapıldı:**
AI eşleştirmesinden gelen bazı kayıtlarda marka/içerik uyumsuzluğu vardı ama bunlar "korunan" statüsünde olduğu için temizlenemiyordu.

**Ne yapıldı:**
Kalite kontrol (cleanup) muafiyet kuralı güncellendi. Eskiden "AI aynı dediyse dokunma" deniyordu. Artık "AI eşleştirmesiyle gelmiş ama marka/beden/kategori uyumsuzluğu varsa temizle" deniyor. SKU ve Link eşleştirmeleri hala korunan statüde — bunlar en güvenilir kaynaklar.

**Ne bekliyoruz:**
AI eşleştirmeleri artık kalite kontrolüne giriyor. Marka/beden/kategori uyumsuzluğu varsa tespit edilip temizlenebiliyor.

---

## 3. Doğrulanmış Eşleşmeleri Tekrar İşlememe

**Ne oldu:**
Pipeline her çalıştığında tüm dataları baştan işliyordu. 1. testte doğru bulunan 910 eşleşme tekrar işleniyor, 1300 hatalı eşleşme de aynı yanlış sonucu tekrar getiriyordu.

**Neden yapıldı:**
2. testi çalıştırırken:
- Doğru olan 910 eşleşmeyi boşuna tekrar işlememek
- Hatalı olan 1300 eşleşmede aynı yanlış sonucu değil, farklı/doğru bir eşleşme araması

**Ne yapıldı:**
Eşleşme tablosuna `match_verified` alanı eklendi. Saha ekibinin kontrol sonuçlarına göre:
- 1. testten sonra 910 doğru eşleşme → `match_verified='yes'`
- 1. testten sonra 1300 hatalı eşleşme → `match_verified='no'`
- 3. testte ek olarak 225 doğru + 153 hatalı eşleşme daha işaretlendi (BigQuery `hatali_eslestirmeler` tablosundan, %50 kuralına takılanlar hariç — çünkü %50 elenmeleri eşleştirmenin kendisiyle değil paket büyüklüğüyle ilgili)

Pipeline davranışı:

| match_verified | Pipeline davranışı |
|---|---|
| `yes` | Kbarcode warehouse'dan çıkarılır, **hiç işlenmez** — mevcut eşleşme korunur |
| `no` | Kbarcode işlenir ama aynı (kbarcode, IC) çifti **engellenir** — farklı eşleşme aranır |
| `NULL` | Normal işlenir |

**Ne bekliyoruz:**
- Doğrulanmış eşleşmeler atlanıyor → daha hızlı çalışma
- Hatalı eşleşmeler tekrar deniyor ama aynı yanlış çift bloke → farklı bir eşleşme gelirse kaydediliyor
- Her test döngüsünde saha ekibinin geri bildirimi sisteme giriyor → kümülatif iyileşme

---

## 4. Paket İçi Ürün Sayısı Kontrolü (Yanlış Tablo)

**Ne oldu:**
Bir pakette kaç ürün olduğunu kontrol ederken yanlış tablodan bakıyorduk (`ods_pbl_tex_sales`). Bu tablodaki sayılar, saha ekibinin (Burak) baktığı [Looker raporuyla](https://looker.trendyol.com/dashboards/19337?parcel_unique_id=7330030580933105) uyuşmuyordu. Ekip `tex_iade_kargo_detay_urun_icerigi` tablosuna bakıyor.

**Neden yapıldı:**
Yanlış ürün sayısı → %50 kuralı hatalı uygulanıyor → olması gereken eşleşmeler eleniyordu veya olmaması gerekenler geçiyordu.

**Ne yapıldı:**
Ürün sayısı kaynağı, ekibin baktığı Looker raporunun arkasındaki tablo (`tex_iade_kargo_detay_urun_icerigi`) ile değiştirildi. Artık ekiple aynı kaynaktan aynı sayıya bakıyoruz.

**Ne bekliyoruz:**
%50 kuralı doğru çalışacak. Ekiple aynı rakamları görüyoruz, tutarsızlık kalkıyor.

---

## 5. Yasaklı Statü (170 kayıt) ve 24 Saat Kuralı (433 kayıt)

**Ne oldu:**
Saha ekibi (Burak) lojistik detaylarına bakarak bazı gönderileri eşleştirme dışı bırakıyor:

- **Yasaklı Statü (170 kayıt):** Gönderinin son lojistik hareketine bakılıyor. `sağlık_durumu`, `operasyon_durumu`, `işlem_açıklaması` alanlarına göre gönderinin durumu "yasaklı" ise eşleştirme yapılmıyor.

- **24 Saat Kuralı (433 kayıt):** Son fiziksel hareket tarihi 24 saatin altındaysa gönderinin henüz hareket halinde olduğu kabul ediliyor ve eşleştirme yapılmıyor. 24 saati geçtiyse eşleşebilir.

Kontrol sheeti: [Lojistik kontrol](https://docs.google.com/spreadsheets/d/1CIWINX8KH8JKmvu1O5ALEwSx1Pcs_3nEc6ge7ghIp3c/edit?gid=0#gid=0)

**Neden henüz yapılmadı:**
Bu kontroller lojistik sistemindeki detaylı alanlara bağlı (`sağlık_durumu`, `operasyon_durumu`, `işlem_açıklaması`). Bizim erişebildiğimiz `delivery_history` tablosunda bu alanlar yok — sadece `event` ve `event_desc` var ama:

- `delivery_history`'deki event'ler saha ekibinin kullandığı `işlem_açıklaması` ile birebir örtüşmüyor
- İngilizce/Türkçe farklılıkları var (event isimleri İngilizce, ekibin baktığı alanlar Türkçe)
- Bazı event'ler `delivery_history`'de hiç yer almıyor
- Bu alanları maplemek için tüm kaynak ekipleri (lojistik, veri, operasyon) bir araya gelip event-to-açıklama eşleştirmesini çıkarması ve case'leri belirlemesi gerekiyor — bu başlı başına büyük bir iş

**Ne bekliyoruz:**
Bu alanların kaynak tabloları netleştirilir ve erişim sağlanırsa, saha ekibinin manuel yaptığı aynı kontrolleri otomatik olarak pipeline'a ekleyebiliriz. Bu durumda 170 + 433 = **603 kayıttaki** hata **0'a** düşer.

---

## 6. Eşleşmeyen Ama Eşleşmesi Gereken Kayıtlar (Threshold Problemi)

**Ne oldu:**
Manuel eşleştirme detayına bakıldığında iki farklı durum var:

**a) Sağlıksız manuel eşleştirmeler:** Bazı manuel eşleştirmelerde markalar ve içerikler tamamen farklı. Bunlar zaten eşleşmemeli — burada sorun bizde değil, manuel eşleştirme tarafında. Örnek: warehouse'da "Galeriyo - Makyaj Organizeri" var, package'da "Blackstar - Makyaj Organizeri" var — ürün adı aynı ama markalar farklı, gerçekten aynı ürün olup olmadığı belli değil.

**b) Benzer ama eşleşmeyen kayıtlar:** Markalar aynı, içerikler benzer ama sistem eşleştirmemiş.

Sebebi: İçerik benzerliği threshold'umuz %85. Marka %100 geliyor ama içerikler %65 bandında kaldığı için eşleşme oluşmuyor.

**Örnek 1:** Flying Tiger Copenhagen → "Gold Telefon Tutacağı" vs "gold akıllı telefon tutucu"
- Marka: %100 aynı
- İçerik benzerliği: ~%65-70
- Threshold: %85
- Sonuç: eşleşmedi (içerik threshold altında)

**Örnek 2:** Aynı ürünün farklı satıcılar tarafından farklı isimlendirilmesi: "Kadın Kare Desenli Triko Kazak" vs "Bayan Geometrik Desen Örgü Kazak" — aynı ürün ama kelimeler tamamen farklı.

**Ne yapılabilir:**
- Marka %100 eşleşiyorsa content threshold'unu düşürmek bir seçenek ama riskli — false positive artabilir
- Daha akıllı bir yaklaşım: marka aynıysa + kategori aynıysa → threshold'u %70'e indirmek gibi kademeli kurallar

**Durum:**
Henüz bir geliştirme yapılmadı. Threshold'u düşürmenin getireceği yanlış eşleşme riskini değerlendirmek gerekiyor.

---

## 7. AI Eşleştirme ve Görsel Analiz Limitasyonları

**Mevcut yapıda iki farklı AI kullanımı var:**

### a) AI Signature Matching (İmza Karşılaştırma)
Her görsel bağımsız olarak AI'a gönderilip analiz ediliyor (kategori, renk, marka, malzeme vb. çıkarılıyor). Sonra iki görselin imzaları karşılaştırılıyor.

**Problem:** Aynı ürün farklı ortamlarda çekildiğinde AI farklı yorumluyor:

| | Katalog Görseli (Trendyol) | Depo Görseli (Kamera) |
|---|---|---|
| **Ortam** | Stüdyo, beyaz fon, profesyonel ışık | Depo, el ile tutulmuş, streç sarılmış |
| **kategori** | "ev" | "aksesuar" |
| **tip** | "düzenleyici" | "makyaj organizeri" |
| **renk** | "pembe şeffaf gri" | "pembe kahverengi" |

Aynı pembe makyaj organizeri ama imzalar tamamen farklı çıkıyor → eşleşme kurulamıyor.

**Neden oluyor:** AI her görseli bağımsız yorumluyor. Depo fotoğrafında ürün streç ile sarılmış, karanlık, farklı açıdan çekilmiş — AI bunu gördüğünde farklı kategorize ediyor.

### b) AI Verification (Direkt Görsel Karşılaştırma)
İki görseli yan yana AI'a gönderip "aynı ürün mü?" diye soruyoruz. Bu daha güvenilir çünkü AI doğrudan karşılaştırma yapıyor.

**Problem:** API maliyeti var ve her eşleşme adayı için çağrı yapılması gerekiyor.

### Nasıl Daha İyi Kullanılabilir?

**1. Hybrid yaklaşım:** Normal eşleştirme (SKU/Link/Exact/Fuzzy) ile bulunamayan ama potansiyel olan çiftlerde AI Verification'ı devreye sokmak. Örneğin: marka aynı ama içerik %65-85 arası → doğrudan AI'a sor.

**2. AI Signature'ı iyileştirme:** Analiz promptunu standartlaştırarak "düzenleyici" vs "organizeri" gibi farklılıkları azaltmak. Sabit bir kategori listesi vererek AI'ın tutarlı sonuç üretmesini sağlamak.

**3. Normal data + AI harmanlaması:** Fuzzy matching'de marka eşleşiyorsa, AI Verification'dan da "AYNI" veya "KESIN_AYNI" geliyorsa → content threshold'u düşük olsa bile eşleştirmeyi kabul etmek. Birden fazla sinyalin birlikte değerlendirilmesi.

Örnek senaryo:
- Marka: %100 aynı (Flying Tiger) ✅
- İçerik: %65 benzer ("Gold Telefon Tutacağı" vs "gold akıllı telefon tutucu") ❌ (threshold altı)
- AI Verification: "AYNI" ✅
- Sonuç: 2/3 sinyal pozitif → eşleştir

**Durum:**
Henüz geliştirme yapılmadı. Bu alan en karmaşık kısım — false positive riski ile eşleşme kaybı arasında denge kurulması gerekiyor. Ek koşullar ve kurallar üzerinde çalışılması gerekiyor.

---
---

# 2. Test 2. Gün Sonrası Geliştirmeler (26.02.2026)

> **Kaynak:** 2. test çalıştırması sonrası saha ekibi (Burak) tarafından iletilen "%50 kuralına uymuyor" geri bildirimi
> **Tespit:** 63 gönderide %50 kuralına uymasına rağmen kayıtlar historic tabloya yazılmıştı

---

## 8. %50 Filtre Sırası Hatası (Kritik Bug Fix)

**Ne oldu:**
2. test sonuçlarında 63 gönderi "%50 kuralına uymuyor" olarak işaretlendi. İncelediğimizde bu kayıtların pipeline tarafından elenmesi gerektiği halde DB'ye yazıldığını gördük.

**Neden oldu:**
Kalite filtrelerinin uygulama sırası yanlıştı. %50 filtresi **Tekil Eşleştirme filtresinden ÖNCE** çalışıyordu. Bu da `matched_count` (eşleşen kbarcode sayısı) hesaplamasını şişiriyordu.

**Somut örnek:**
Gönderi `7330025197779000` — Siveno markasının 5 farklı ürün içeren bir paketi:

| Adım | Açıklama |
|------|----------|
| 1. SKU Matching | Paketin 4 farklı SKU'su var. Warehouse'da bu SKU'lardan 3 farklı kbarcode eşleşti |
| 2. matched_count = 3 | %50 filtresi: 3 ≥ 5×0.5 = 2.5 → **GEÇTİ** |
| 3. Tekil Eşleştirme | O 3 kbarcode'dan 2'si başka gönderilere atandı → geriye **1 kbarcode** kaldı |
| 4. Sonuç | DB'de 1 kbarcode ile kayıt var ama gerçek oran %20 (1/5) — elenmesi gerekirdi |

**Problem:** `matched_count` Tekil Eşleştirme'den önce hesaplandığı için şişik kalıyordu. Aynı SKU'ya sahip birden fazla kbarcode pipeline'da ilk etapta eşleşiyor ama sonra Tekil Eşleştirme sırasında başka gönderilere dağılıyordu. %50 filtresi bu dağılımdan habersiz eski (şişik) sayıyla karar veriyordu.

**Ne yapıldı:**
Kalite filtrelerinin uygulama sırası değiştirildi:

| Eski Sıra | Yeni Sıra |
|-----------|-----------|
| 1. %50 product_count filtresi | 1. Lokasyon filtresi (aynı lokasyonda mı?) |
| 2. Lokasyon filtresi | 2. Tekil Eşleştirme (1 kbarcode → 1 gönderi) |
| 3. Tekil Eşleştirme | 3. %50 product_count filtresi **(Tekil sonrası gerçek sayıyla)** |

Artık `matched_count` Tekil Eşleştirme **sonrası** hesaplanıyor. Bu sayede her gönderiye gerçekten atanan kbarcode sayısı üzerinden %50 kontrolü yapılıyor.

**Aynı örnek yeni sırayla:**

| Adım | Açıklama |
|------|----------|
| 1. Lokasyon filtresi | Geçti |
| 2. Tekil Eşleştirme | 3 kbarcode'dan 2'si başka gönderilere atandı → bu gönderi için **1 kbarcode** |
| 3. matched_count = 1 | %50 filtresi: 1 < 2.5 → **ELENDİ** ✅ |

**Ne bekliyoruz:**
- Daha önce yanlışlıkla geçen 63 benzeri kayıt artık doğru şekilde elenecek
- %50 kuralı gerçek eşleşme sayısını yansıtacak, şişik sayılarla karar verilmeyecek
- Hem production pipeline hem de diagnose endpoint aynı doğru sırayla çalışıyor

---

## 9. Akıllı Tekil Eşleştirme (Eşleşme Oranını Artırma)

**Ne oldu:**
Bir kbarcode birden fazla gönderiyle (integration_code) eşleştiğinde, sistem hangi gönderiyi seçeceğine `match_rule` alanının **alfabetik sıralaması** ile karar veriyordu. Bu tamamen rastgele bir seçimdi — ürün içeriğine göre eşleştirme, SKU eşleştirmesinden önce geliyordu çünkü "p" harfi "🔑" emojisinden önce sıralanıyordu. Eşleştirme güvenilirliği ve paket büyüklüğü hiç dikkate alınmıyordu.

**Neden yapıldı:**
Aynı kbarcode hem 3 ürünlü küçük bir paketle hem 15 ürünlü büyük bir paketle eşleşebiliyor. Eski mantıkta rastgele biri seçiliyordu. Oysa büyük pakete yönlendirmek, toplam eşleşme sayısını artırabilir — çünkü büyük paketin %50 kuralını geçmesi için daha çok kbarcode'a ihtiyacı var ve her ek kbarcode daha fazla toplam eşleşme koruyor.

**Somut örnek:**

Kbarcode `K177xxx` hem IC-A (3 ürünlü paket, 1 eşleşme) hem IC-B (15 ürünlü paket, 9 eşleşme) ile eşleşiyor:

| Senaryo | IC-A (3 ürün) | IC-B (15 ürün) | Toplam korunan |
|---------|---------------|----------------|----------------|
| **Eski:** Rastgele → IC-A | 2/3=%67 GEÇTİ | 9/15=%60 GEÇTİ | 11 kayıt |
| **Yeni:** Büyük paket → IC-B | 1/3=%33 ELENDİ | 10/15=%67 GEÇTİ | 10 kayıt |

Bu örnekte fark küçük, ama asıl kazanç şu senaryoda:

| Senaryo | IC-A (2 ürün) | IC-B (20 ürün, 9 eşleşme) | Toplam korunan |
|---------|---------------|---------------------------|----------------|
| **Eski:** Rastgele → IC-A | 1/2=%50 GEÇTİ (1 kayıt) | 9/20=%45 ELENDİ (0 kayıt) | **1 kayıt** |
| **Yeni:** Büyük paket → IC-B | 0/2=%0 ELENDİ (0 kayıt) | 10/20=%50 GEÇTİ (10 kayıt) | **10 kayıt** |

Kbarcode IC-B'ye giderek 10 eşleşme kurtarılıyor, IC-A'ya gitseydi sadece 1 eşleşme korunacaktı.

**Ne yapıldı:**
Tekil Eşleştirme mantığı tamamen değiştirildi. Artık 1 kbarcode birden fazla IC ile eşleştiğinde 3 aşamalı akıllı seçim yapılıyor:

| Öncelik | Kriter | Açıklama |
|---------|--------|----------|
| 1 | **%50 uygunluğu** | %50 kuralına uyan IC tercih edilir |
| 2 | **product_count (büyük paket)** | Daha fazla ürün içeren pakete yönlendirilir |
| 3 | **match_rule güvenilirliği** | Link > SKU > Exact > Fuzzy sıralaması |

Ayrıca `match_rule` öncelik sıralaması da düzeltildi — eskiden alfabetik sıralama kullanılıyordu (yanlış), artık mantıksal güvenilirlik sıralaması kullanılıyor:

| Öncelik | Eşleştirme Yöntemi |
|---------|--------------------|
| 0 (en güvenilir) | Trendyol Link |
| 1-3 | SKU eşleştirmeleri |
| 4-6 | İçerik bazlı (Exact) eşleştirmeler |
| 99 | Fuzzy eşleştirmeler |

**Ne bekliyoruz:**
- Kbarcode'lar en verimli pakete yönlendirilecek → toplam eşleşme sayısı artacak
- Büyük paketlerin %50 kuralını geçme şansı artacak (her ek kbarcode orada daha değerli)
- Eşleştirme güvenilirlik sıralaması artık doğru (SKU > Exact > Fuzzy, alfabetik değil)

---

## 10. Kalite Filtresi Elenme Kaydı + Retry Mekanizması

**Ne oldu:**
Kalite filtrelerinde (%50 kuralı, lokasyon, tekil eşleştirme) elenen kbarcode'lar tamamen kaybediliyordu. Cleanup'ta silinen eşleştirmeler `analysis_results` tablosuna yazılıyordu ama filtre elenmeleri hiçbir yere kaydedilmiyordu. Bu durumda:

1. Bir kbarcode bir gönderiyle eşleşti ama %50 kuralına takıldı → elendi ve kayboldu
2. Belki başka bir gönderiyle eşleşebilirdi ama tekrar denenme şansı yoktu
3. Cleanup'ta silinen bir eşleştirme, o kbarcode'un başka bir gönderiyle eşleşme olasılığını da yok ediyordu

**Neden yapıldı:**
Eşleştirme pipeline'ı sıralı çalışıyor: önce eşleştir → sonra filtrele → sonra cleanup yap. Bu süreçte her eleme "nihai" kabul ediliyordu. Oysa elenen bir kbarcode'un başka bir gönderiyle eşleşme ihtimali var — ama bu ihtimal hiç denenemiyor, çünkü kbarcode kaybolmuş oluyor.

**Ne yapıldı:**

**a) Filtre elenme kaydı:**
Kalite filtrelerinde elenen her `(kbarcode, integration_code)` çifti artık `analysis_results` tablosuna yazılıyor:

| problem_status | Açıklama |
|---------------|----------|
| `FILTER_LOCATION` | Farklı lokasyondan eşleşme (aynı gönderinin kbarcode'ları farklı lokasyonlarda) |
| `FILTER_TEKIL` | Tekil eşleştirmede başka gönderi tercih edildi |
| `FILTER_50_PERCENT` | %50 ürün sayısı kuralına uymuyor |

**b) Retry endpoint (`/matching/retry`):**
Yeni bir endpoint oluşturuldu. Bu endpoint:

1. `analysis_results`'tan tüm elenen/silinen kbarcode'ları toplar (hem filtre elenmeleri hem cleanup silmeleri)
2. Hâlâ `is_current=true` eşleşmesi olanları çıkarır (zaten eşleşmiş, tekrar gerek yok)
3. Eski yanlış eşleştirmeleri engeller — aynı `(kbarcode, integration_code)` çifti tekrar gelmez
4. `match_verified='no'` çiftlerini de engeller
5. Kalan kbarcode'lar için tam eşleştirme pipeline'ını çalıştırır (SKU → Link → Exact → Fuzzy)
6. Aynı kalite filtrelerini uygular (lokasyon, tekil, %50)
7. Yeni eşleştirmeleri hem `missing_packages_match`'e hem `historic`'e kaydeder (incremental — mevcut kayıtlara dokunmadan)
8. Başarılı retry'ların eski filtre kayıtlarını `analysis_results`'tan temizler

**Somut örnek:**

| Adım | Açıklama |
|------|----------|
| 1. Enhanced pipeline | K177xxx → IC-A ile eşleşti (SKU matching) |
| 2. %50 filtresi | IC-A'nın 10 ürünü var, sadece 2 eşleşme → %20 < %50 → **ELENDİ** |
| 3. analysis_results'a yazıldı | `problem_status='FILTER_50_PERCENT'`, K177xxx + IC-A kaydedildi |
| 4. Retry çalıştı | K177xxx tekrar pipeline'a sokuldu, IC-A engellendi |
| 5. Yeni eşleşme | K177xxx → IC-B ile eşleşti (Exact matching, 3 ürünlü paket, 2 eşleşme → %67 GEÇTİ) |
| 6. Sonuç | K177xxx artık IC-B ile historic'te `is_current=true` |

**Ne bekliyoruz:**
- Filtrelerde ve cleanup'ta kaybedilen kbarcode'lar ikinci bir şans alıyor
- Aynı yanlış eşleştirme tekrarlanmıyor (rejected pairs mekanizması)
- Toplam eşleşme sayısı artacak — özellikle %50 kuralına takılan ama başka gönderilerle eşleşebilecek kbarcode'lar için
- Pipeline'ın "tek geçişlik" sınırı aşılıyor, daha kapsamlı eşleştirme sağlanıyor

---
---

# 3. Son Durum ve Açık Kaygılar (27.02.2026)

> **Kaynak:** 3. test çalıştırması sonrası pipeline analizi ve [manuel eşleştirme karşılaştırma sheeti](https://docs.google.com/spreadsheets/d/17iK4JHgf9sGgFpSyJNom78hJledZ1pwcMlIfRd1jNdc/edit?gid=1031814373#gid=1031814373)
> **Durum:** Pipeline çalışıyor, filtreler doğru çalışıyor, retry mekanizması aktif. Ancak yapısal sınırlar ve kaynak veri kalitesi nedeniyle çözülemeyen durumlar var.

---

## 11. Tekli Ürün → Çoklu Paket Eşleştirme Sorunu (1→10 Problemi)

**Ne oldu:**
Manuel eşleştirme ekibi, aynı gönderiyle (integration_code) 10 ayrı tekli kbarcode'u eşleştirmiş. Ama bu 10 kbarcode warehouse'da bağımsız tekli ürünler olarak duruyor — aralarında bir "grup" ilişkisi yok.

**Somut örnek:**
Gönderi `7330028521871791` — Fureya markasının "Karışık Renkli 10 Adet Flamlı Pamuk Eşarp" paketi:

| Kbarcode | Warehouse İçeriği | Paket İçeriği |
|----------|-------------------|---------------|
| K1771611813617 | Selin - Düz Renk Tülbent Yazma | Fureya - Karışık Renkli 10 Adet Flamlı Pamuk Eşarp |
| K1771611813787 | Selin - Düz Renk Tülbent Yazma | Fureya - Karışık Renkli 10 Adet Flamlı Pamuk Eşarp |
| K1771611813937 | Selin - Düz Renk Tülbent Yazma | Fureya - Karışık Renkli 10 Adet Flamlı Pamuk Eşarp |
| ... (toplam 10 adet) | ... | ... |

**Neden eşleştiremiyoruz:**
- Sistem 1→1 bakış açısıyla çalışıyor: her kbarcode'u bağımsız olarak bir gönderiyle eşleştirmeye çalışıyor
- Warehouse'daki "Selin - Düz Renk Tülbent Yazma" ile paketin "Fureya - Karışık Renkli 10 Adet Flamlı Pamuk Eşarp" içeriği tamamen farklı
- Marka farklı (Selin vs Fureya), ürün adı farklı, SKU farklı — hiçbir eşleştirme stratejisi bunu yakalayamaz
- Manuel ekip bunu "fiziksel olarak yanyana duruyorlar" bilgisiyle eşleştiriyor, ama bu bilgi dijital veride yok

**Ne yapılabilir:**

1. **Kaynak veri tarafı:** Girilen ekranda "toplu girme" seçeneği gelebilir. Kullanıcı 10 tekli kbarcode'u bir grup olarak işaretleyebilir. Kaynak datada bu kbarcode'ların aynı gönderiyle ilişkili olduğu bir flag/ID ile belirtilebilir (örn: aynı `group_id` altında toplanması).

2. **Pipeline tarafı:** Kaynak veride grup bilgisi varsa, pipeline bu grubu tek bir birim olarak ele alabilir. Gruptaki kbarcode'ların toplam sayısı ile paketin ürün sayısı karşılaştırılır ve toplu eşleştirme yapılır.

3. **Şu anki durum:** Kaynak veride bu grup bilgisi olmadığı sürece, pipeline bu tür eşleştirmeleri yapamaz. Bu bir kod limiti değil, veri limiti.

---

## 12. Manuel Eşleştirmelerde Alternatif/Yanlış Marka Girişi

**Ne oldu:**
Manuel eşleştirme yapan kullanıcılar, ürünün gerçek markasını bulamadıklarında alternatif veya yakın marka giriyorlar. Bu, kaynak veriyi manipüle ediyor ve otomatik eşleştirmeyi imkansız hale getiriyor.

**Somut örnekler:**

| Kbarcode | Warehouse Marka | Warehouse İçerik | Paket Marka | Paket İçerik |
|----------|----------------|------------------|-------------|--------------|
| K1771692216922 | ENQ | 6lı Saç Tebeşiri Saç Boyası Seti | Royal paris | Temporary Hair Chalk - Saç Tebeşiri |
| K1771618067296 | KTS | 4 Adet 40w Floresan Yatay Led Bant Armatür | Gold ROYAL | 40 Watt 120 Cm Led Bant Armatür |

**Neden eşleştiremiyoruz:**
- Marka tamamen farklı (ENQ vs Royal paris, KTS vs Gold ROYAL)
- Ürün içeriği kısmen benzer ama dil/format farklılıkları var (Türkçe vs İngilizce, "6lı Saç Tebeşiri" vs "Temporary Hair Chalk")
- Size bilgisi yok
- Görseller de çok benzemeyince AI verification da yakalayamıyor
- Tüm sinyaller (marka, içerik, size, görsel) negatif veya belirsiz → eşleşme oranı imkansıza yaklaşıyor

**Ne yapılabilir:**

1. **Kaynak veri tarafı:** Manuel giriş ekranında "marka bulunamadı" seçeneği veya alternatif marka alanı eklenmeli. Böylece sistem, girilen markanın kesin mi yoksa tahmini mi olduğunu bilir.

2. **Pipeline tarafı:** Eğer marka "tahmini" olarak işaretlenmişse, marka eşleştirme ağırlığı düşürülebilir ve diğer sinyallere (içerik, görsel, kategori) daha fazla ağırlık verilebilir.

3. **Şu anki durum:** Kaynak veride marka güvenilirlik bilgisi olmadığı sürece, pipeline markayı güvenilir kabul ediyor ve farklı marka = farklı ürün olarak değerlendiriyor. Bu doğru bir davranış — aksi takdirde false positive patlar.

---

## 13. Hermes/Lojistik Statü Kaynaklı Yanlış Pozitifler

**Ne oldu:**
Bu konu Madde 5'te (Yasaklı Statü + 24 Saat Kuralı) detaylıca açıklandı. Toplam yanlış eşleştirmelerin **%50'den fazlası** bu kategoriden kaynaklanıyor.

Saha ekibi (Burak), lojistik sistemindeki `sağlık_durumu`, `operasyon_durumu`, `işlem_açıklaması` alanlarına bakarak bazı gönderileri eşleştirme dışı bırakıyor. Ancak bu alanlar bizim erişebildiğimiz tablolarda (`delivery_history`) mevcut değil.

**Etki:**
Bu alan çözülmeden yanlış eşleştirme oranı önemli ölçüde düşürülemez. 170 yasaklı statü + 433 adet 24 saat kuralı = **603 kayıt** potansiyel yanlış pozitif kaynağı.

**Durum:**
Kaynak tablo erişimi ve event-to-açıklama eşleştirmesi bekliyor. Detaylar Madde 5'te.

---

## 14. Manuel Eşleştirmelerle Başarı Oranı Doğrulaması

**Soru:** Manuel verilen eşleştirmelerle sonucu karşılaştırıp doğru oranlara gidebilir miyiz?

**Mevcut durum:**
- Manuel eşleştirme ekibinin verileri karşılaştırma kaynağı olarak kullanılıyor
- Ancak Madde 11 ve 12'de görüldüğü gibi, manuel eşleştirmelerin kendisi de her zaman sağlıklı değil (yanlış marka girişi, fiziksel gözleme dayalı toplu eşleştirme)
- Manuel ekibin erişebildiği bilgi (fiziksel olarak yanyana durma, lojistik detayları) ile sistemin erişebildiği bilgi (dijital veri, görseller) arasında büyük fark var

**Yapılabilecekler:**

1. **Karşılaştırma metriği tanımı:** "Doğru eşleştirme" tanımının netleştirilmesi gerekiyor. Manuel ekibin fiziksel gözleme dayalı eşleştirmesi ile dijital veri bazlı otomatik eşleştirme aynı kriter seti ile değerlendirilmeli.

2. **Ortak alt küme:** Her iki tarafın da eşleştirebildiği kayıtlar (marka, içerik, SKU tutarlı olan) üzerinden başarı oranı hesaplanabilir. Tutarsız kaynak verili (yanlış marka, toplu eşleştirme) kayıtlar bu hesaplamadan ayrılmalı.

3. **Geri bildirim döngüsü:** `match_verified` mekanizması zaten bunu yapıyor — her test döngüsünde saha ekibinin `yes`/`no` geri bildirimi sisteme giriyor. Kümülatif olarak doğruluk oranı artıyor.

---

## 15. %50 Kuralı ve Toplu Paket Değerlendirme Problemi

**Ne oldu:**
Bir pakette 5 ürün varsa ve 3 tanesi eşleştirilmişse, 2'si doğru 1'i yanlış olsa bile tüm paket "yanlış" olarak değerlendiriliyor. Bu, başarı oranlarını yapay olarak düşürüyor.

**Somut örnek:**

| Paket (5 ürün) | Eşleşme | Durum |
|----------------|---------|-------|
| Kbarcode-1 → IC-A | Doğru | ✅ |
| Kbarcode-2 → IC-A | Doğru | ✅ |
| Kbarcode-3 → IC-A | Yanlış | ❌ |
| Kbarcode-4 | Eşleşmedi | — |
| Kbarcode-5 | Eşleşmedi | — |

Mevcut değerlendirme: 3 eşleşmeden 1'i yanlış → paket "hatalı" → **5 kayıt yanlış sayılıyor**
Gerçek durum: 5 kayıttan 2'si doğru, 1'i yanlış, 2'si eşleşmemiş

**Neden sorun:**
- Başarı oranları olduğundan çok daha kötü görünüyor
- 1 yanlış eşleşme 5 yanlışa dönüşüyor (5x çarpan etkisi)
- Büyük paketlerde (10-15 ürün) bu çarpan etkisi daha da artıyor
- Toplu bakış açısına geçilmediği sürece metrikler yanıltıcı olacak

**Ne yapılabilir:**

1. **Kayıt bazlı değerlendirme:** Her kbarcode bağımsız değerlendirilmeli — doğru eşleşen ayrı, yanlış eşleşen ayrı, eşleşmeyen ayrı sayılmalı. Paket bazlı "hepsi doğru veya hepsi yanlış" mantığından çıkılmalı.

2. **Kısmi başarı metriği:** Bir paket için "3/5 eşleşti, 2/3 doğru" gibi kısmi başarı oranı hesaplanabilir. Bu daha gerçekçi bir resim verir.

3. **Toplu paket değerlendirme:** Madde 11'deki toplu eşleştirme altyapısı geldiğinde, paket bazlı değerlendirme de anlamlı hale gelir. O zamana kadar kayıt bazlı değerlendirme daha doğru sonuç verir.

**Şu anki durum:**
Bu bir hesaplama/raporlama problemi — pipeline'ın eşleştirme kalitesinden bağımsız. Değerlendirme metriklerinin güncellenmesi gerekiyor.

---

## 16. AI Görsel Analiz Performansı ve Veri Harmanlaması (TO-DO)

> **Öncelik notu:** Bu madde, Madde 11 (toplu paket) ve Madde 12 (kaynak veri kalitesi) çözüldükten **sonra** ele alınmalı. Kaynak veri sorunları düzelmeden AI tarafına yatırım yapmak verimsiz olur.

**Mevcut durum:**
`ai_analysis_cache_temp` tablosunda her görselin detaylı AI analizi tutuluyor: kategori, tip, alt_tip, marka, renk, desen, malzeme, stil, cinsiyet, boyut, hacim, adet, logo, baskı, doku ve onlarca kategori-özel alan. Bu veriler şu anda sadece AI signature matching (imza karşılaştırma) için kullanılıyor ama pipeline'daki diğer eşleştirme adımlarıyla harmanlanmıyor.

**Sorun:**
AI aynı ürünü farklı ortamlarda farklı yorumluyor. Katalog görseli ile depo görseli arasında tutarsızlıklar oluşuyor:

**Örnek 1 — Altınyıldız Classics parfüm (aynı ürün, 2 farklı görsel):**

- Katalog: https://cdn.dsmcdn.com//ty1000042/product/media/images/prod/PIM/20251215/14/15127cec-884f-4ac9-aa04-aadc49d116c3/1_org.jpg
- Depo: https://s3.trendyol.com/hs-return-packages/2026/2/23/b3c61e41-7501-4dd5-aeca-0e07842839b5.png

| Alan | Katalog Görseli (Trendyol) | Depo Görseli (Kamera) |
|------|---------------------------|----------------------|
| renk | koyu mavi, açık mavi, altın | siyah |
| malzeme | cam, metal, karton | karton |
| logo | yazılı marka + geometrik logo | yazılı marka + yıldız logosu |
| ambalaj | şişe | kutu |

Marka, kategori, tip, hacim aynı — ama renk, malzeme, ambalaj farklı çıkıyor. İmza karşılaştırması bu farklardan dolayı düşük skor veriyor.

**Örnek 2 — Mavi bluz (aynı ürün, katalog vs depo):**

- Katalog: https://cdn.dsmcdn.com//ty1826/prod/QC_PREP/20260211/14/e71bfcb7-8d67-324e-b5ce-511123d01414/1_org.jpg
- Depo: https://s3.trendyol.com/hs-return-packages/2026/2/21/287a42bf-ed4b-4348-bedc-eaa2c60625ad.png

| Alan | Katalog Görseli | Depo Görseli |
|------|----------------|-------------|
| tip | bluz | üst |
| alt_tip | kolsuz bluz | (yok) |
| cinsiyet | kadın | (belirsiz) |
| boyut | (yok) | M |
| yuz_doku | tamamen düz | dalgalı |
| dekor | drape | sade |
| giyim_kesim | dar | (yok) |
| yaka_tipi | boğazlı | (yok) |

Katalog görselinde model giyer halde, arka plan temiz, kıyafet net görünüyor — AI detaylı analiz yapabiliyor (kolsuz, boğazlı, dar kesim). Depo görselinde ürün elle tutulmuş, buruşuk, etiket asılı, arka planda depo rafları var — AI sadece "mavi üst giyim" diyebiliyor. Aynı bluz ama AI çıktıları çok farklı.

> **Bu örneklerin gösterdiği:** Sadece görselden eşleştirme yapmaya çalışmak çok zor — depo ortamı görselleri bozuyor, AI tutarsız sonuçlar veriyor. **Var olan data ile (SKU, marka, içerik, size) AI verisini harmanlamak** çok daha sağlıklı sonuç verir. Görseli tek başına değil, diğer sinyallerle birlikte destekleyici olarak kullanmak gerekiyor. Bu alan üzerinde çalışılması gerekiyor.

**Örnek 2 — Depo görseli birden fazla ürün içeriyor:**
Tek bir depo fotoğrafında hem mavi bluz hem koyu mavi kot pantolon var. AI ikisini ayrı analiz edemiyor, sadece birini (bluz) algılıyor ve sonucu ona göre veriyor. Diğer ürün (pantolon) tamamen kayıp.

**Yapılabilecekler (TO-DO):**

### a) AI cache verisini pipeline'a entegre etme
Mevcut eşleştirme adımlarında (Exact, Fuzzy) sadece text bazlı alanlar (product_content_name, brand_name, size) kullanılıyor. AI cache'teki ek alanlar (kategori, tip, cinsiyet, malzeme, renk) eşleştirme skorlamasına dahil edilebilir:
- Fuzzy matching'de marka eşleşmezse ama AI cache'te her iki görselin `kategori + tip + cinsiyet` aynıysa → ek sinyal olarak kullanılabilir
- İçerik threshold'un altında kalan eşleşmelerde AI verileri tiebreaker olabilir

### b) AI signature tutarlılığı
Farklı ortamlardan gelen görsellerde tutarsız sonuçlar azaltılabilir:
- Prompt'a "ürünün kendisine odaklan, ambalaj/ortam bilgisini ayrı tut" gibi yönlendirmeler
- Renk ve malzeme gibi ortamdan çok etkilenen alanların karşılaştırma ağırlığını düşürme
- Marka, kategori, tip gibi ortamdan bağımsız alanların ağırlığını artırma

### c) Multi-signal karar mekanizması
Tek bir eşleştirme stratejisine güvenmek yerine, birden fazla sinyali birlikte değerlendirme:

| Sinyal | Kaynak | Güvenilirlik |
|--------|--------|-------------|
| SKU eşleşmesi | Veri tabanı | Çok yüksek |
| Marka eşleşmesi | Text | Yüksek |
| İçerik benzerliği | Text (fuzzy) | Orta |
| AI kategori/tip | Görsel analiz | Orta |
| AI verification | Görsel karşılaştırma | Yüksek (ama maliyetli) |

Birden fazla sinyal pozitifse (ör: marka aynı + AI kategori aynı + içerik %65+) → threshold altında kalsa bile eşleştirme kabul edilebilir.

### d) Çoklu ürün tespiti
Depo görsellerinde birden fazla ürün olduğunda AI sadece birini algılıyor. Bu durumda:
- Görselde kaç ürün olduğunu tespit etme (object detection)
- Her ürün için ayrı analiz yapma
- Bu, Madde 11'deki toplu paket problemiyle de bağlantılı

**Şu anki durum:**
AI altyapısı çalışıyor, cache tablosu dolu, veriler mevcut. Ancak **önce kaynak veri sorunları (Madde 11: toplu paket girişi, Madde 12: marka güvenilirliği) çözülmeli.** Kaynak veri temiz olduğunda, AI verisini pipeline'a entegre etmek çok daha anlamlı ve ölçülebilir sonuçlar verir. Aksi takdirde AI iyileştirmeleri kirli verinin üstüne yapılmış olur ve gerçek etkisi ölçülemez.
