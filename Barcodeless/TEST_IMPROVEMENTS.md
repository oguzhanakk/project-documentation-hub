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
Eşleşme tablosuna "doğrulanmış mı?" alanı eklendi. 1. testten sonra 910 doğru eşleşme "evet", 1300 hatalı eşleşme "hayır" olarak işaretlendi. Pipeline artık "evet" olanlara hiç bakmıyor, "hayır" olanlarda ise aynı yanlış eşleşmeyi bloke edip farklı bir eşleşme arıyor.

**Ne bekliyoruz:**
- Doğrulanmış eşleşmeler atlanıyor → daha hızlı çalışma
- Hatalı eşleşmeler tekrar deniyor ama aynı yanlış çift bloke → farklı bir eşleşme gelirse kaydediliyor

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
