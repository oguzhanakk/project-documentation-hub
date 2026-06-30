# Notlar 2 — İlk İnceleme Sonrası Değerlendirmeler

İlk incelemelerim sonrasında aşağıdaki hususların değerlendirilmesinin faydalı olacağını düşünüyorum:

1. Ölçüm değerlerinin ısı haritası şeklinde gösteriminde, görselleştirmenin traversler yerine raylar üzerinde yapılmasının daha anlamlı ve kullanıcı açısından daha anlaşılır olacağını değerlendiriyorum.

2. Sistemin farklı zamanlardaki değişimleri nasıl yansıttığını daha iyi görebilmek adına, mümkünse 3 veya 4 farklı ölçüm datası yüklenerek sonuçları tekrar incelemek isterim. Bunun için size 1 ya da 2 ölçüm verisi daha içeren dosyaları sunabilirim.

3. Belirli kilometre aralıklarına ait verileri grafik olarak görüntüleyebilmek oldukça faydalı olacaktır. Örneğin, 281+000 – 282+000 arasındaki dever ölçüm sonuçlarını grafik üzerinde görebilirsek verilerin yorumlanması daha kolay olabilir. Bu grafiğin y ekseni bahse konu data, x ekseni de ölçüm alınan nokta olmalı, 281+000, 281+000,25, 281+000,50, 281+000,75 … şeklinde.

4. Arayüzde kullanılan "demir açıklığı" ifadesinin, demiryolu terminolojisinde daha yaygın kullanılan "ekartman" ifadesi ile değiştirilmesi gerekli.

5. Yapay zekâ destekli raporlama kısmını ayrıca bir toplantıda biraz daha detaylı konuşabilirsek çok faydalı olacağını düşünüyorum. Özellikle yapay zekâ tarafı yeterince güçlü olacaksa, filtreleme için ayrıca bir arayüze ihtiyaç duyup duymayacağımızı birlikte değerlendirebiliriz. Limit aşımı tespit edilen noktalar için otomatik raporlama yapılabilmesi mümkün olabilir mi? Örneğin, aşımın gerçekleştiği konumun kilometresi ve koordinatı, hangi parametrede (ekartman, dever, vb.) ve hangi seviyede (AL, IL veya IAL) limit aşımı olduğu ile birlikte, mümkünse ilgili bölgenin model üzerindeki görüntüsünün veya ekran çıktısının rapora eklenmesi oldukça faydalı olacaktır.

6. Araziyi üç boyutlu olarak görüntülemek mümkün olmayacaksa, uydu görüntüsü veya benzeri bir altlık veriyi sisteme entegre etme imkânımız olup olmadığını da bir toplantıda değerlendirebilir miyiz?

7. Ölçüm verilerinin değerlendirilmesinde kullanılan limit değerleri üç farklı seviyeden oluşmakta olup, bu değerler hattın işletme hızına göre değişiklik göstermektedir. İlgili limit tablolarını tarafınıza iletebilirim. Grafiklerde, ölçülen verilerin yanı sıra bu üç seviyeye ait alt ve üst limit değerlerinin de referans çizgileri olarak gösterilmesi mümkün olabilir mi? Böyle bir gösterimin verilerin yorumlanmasını ve kritik bölgelerin tespitini kolaylaştıracağını düşünüyorum.

---

## Ne Yapıldı? — Kısa Özet

**Not 1 — Heatmap: Traversler / Raylar Filtresi**
IoT Heatmaps paneline "Filtre" dropdown'ı eklendi. "Tüm Elemanlar ", "Traversler" (`Baluster-Track_Tile_ADP`) veya "Raylar" (`Component_A-0-0`) seçilebiliyor. Seçilmeyen eleman tipleri gri gösterilirken seçilen tipler üzerinde ısı haritası renklendirmesi uygulanıyor. Elemanlar BIM modelindeki `Kilometraj_KPM` özelliği üzerinden sensör verisine eşleniyor (önceki orantısal indekslemeden çok daha doğru).

**Not 3 — Km Aralığı Grafiği**
Viewer toolbar'ına yeni bir buton eklendi. Tıklandığında açılan panelden Km başlangıç/bitiş, kanal ve yıl seçilerek "Grafiği Çiz" butonuna basılıyor. Seçilen aralıktaki tüm ölçüm noktaları x ekseninde (281+66.00, 281+66.25 … formatında), y ekseninde ölçüm değeriyle çiziliyor. AL/IL/IAL limit çizgileri grafik üzerinde kesikli çizgi olarak gösteriliyor.

**Not 4 — "Demir Açıklığı" → "Ekartman"**
`services/googlesheets.js` içindeki gauge kanalının adı güncellendi. Arayüzde artık "Ekartman" görünüyor.

**Not 7 — 3 Seviyeli Limit (AL / IL / IAL)**
Her ölçüm kanalı için üç ayrı limit seviyesi tanımlandı. Heatmap'te renk artık tek kırmızı değil: AL aşımı sarı, IL aşımı turuncu, IAL aşımı kırmızı gösteriliyor. Sensör detay grafiklerinde de her seviye için kesikli yatay referans çizgileri eklendi. Limit sayısal değerleri şimdilik placeholder; gerçek tablo iletilince güncellenecek.

**Bekleyenler**
- Not 2 — Çoklu yıl: ek ölçüm dosyaları gelince
- Not 5 — AI raporlama: toplantı sonrası
- Not 6 — Uydu görüntüsü: toplantı sonrası
- Not 7 (limit değerleri) — Gerçek tablo iletilince güncellenecek

---

## Uygulama Durumu

| Madde | Durum | Tamamlanma |
|-------|-------|-----------|
| "Ekartman" terminoloji düzeltmesi | ✅ Tamamlandı | `services/googlesheets.js` satır 27 |
| 3 seviyeli limit yapısı (AL/IL/IAL) | ✅ Tamamlandı | `googlesheets.js` + `routes/iot.js` |
| Heatmap 3 renk kademesi (sarı/turuncu/kırmızı) | ✅ Tamamlandı | `main.js` — `_applyHeatmap` |
| SensorDetail grafiğine 3 limit referans çizgisi | ✅ Tamamlandı | `main.js` — `_createChart` |
| Km aralığı grafiği (yeni toolbar butonu + panel) | ✅ Tamamlandı | `main.js` + `routes/iot.js` `/kmrange` |
| Heatmap — Traversler / Raylar filtresi (`Kilometraj_KPM` eşleme) | ✅ Tamamlandı | `main.js` — `_buildKmElements`, `updateTypeFilter` |
| Çoklu dataset (3-4 yıl) | ⏳ Bekliyor | Ek ölçüm dosyaları gelince |
| 3 seviyeli limit gerçek değerleri | ⏳ Bekliyor | Limit tablosu gelince |
| AI otomatik raporlama | 📅 Toplantı sonrası | — |
| Uydu görüntüsü altlığı | 📅 Toplantı sonrası | — |

---

## Yapılan Değişikliklerin Detayı

### `services/googlesheets.js`
- `'Demir Açıklığı'` → `'Ekartman'` (gauge kanalı)
- `CHANNEL_META` yapısı 3 seviyeli limite güncellendi: her kanal için `limits: { AL, IL, IAL }` objesi eklendi
- Sensör objesine `km` ve `metre` alanları eklendi (Km aralığı sorgusu için gerekli)
- Channels API çıktısı `limits` objesini de döndürüyor; geriye dönük uyumluluk için `limitMin/limitMax` = AL sınırları

### `routes/iot.js`
- Yeni endpoint: `GET /iot/kmrange?kmFrom=&mFrom=&kmTo=&mTo=&channel=&year=`
  - Belirtilen Km+Metre aralığındaki tüm ölçüm noktalarını sıralı döndürür
  - Kanal meta bilgileri (3 seviyeli limitler dahil) de yanıtta yer alır

### `public/extensions/IoTExtension/contents/main.js` (1110 → 1404 satır)
- **Heatmap renk mantığı:** AL aşımı → sarı, IL aşımı → turuncu, IAL aşımı → kırmızı (önceden tek kırmızı vardı)
- **Heatmap legend:** AL/IL/IAL limit değerlerini renkli gösteriyor
- **SensorDetail grafikleri:** Her kanalda AL/IL/IAL için 3 çift kesikli yatay referans çizgisi eklendi (Chart.js dataset)
- **Yeni `KmRangeChartPanel`:** Km başlangıç/bitiş, kanal ve yıl seçimi; Chart.js ile ölçüm noktaları grafiği; limit referans çizgileri
- **Yeni `SensorRangeChartExtension`:** Toolbar'a yeni buton (sinyal dalgası + dikey kesik çizgi ikonu); `IoT.RangeChart` olarak kayıtlı
- **IoTExtension wrapper:** 4 → 5 alt extension (yeni `IoT.RangeChart` eklendi)

### Placeholder Limit Değerleri (güncellenecek)
```
Ekartman (Gauge): AL ±3mm | IL ±6mm | IAL ±9mm
Alignment:        AL ±3mm | IL ±5mm | IAL ±7mm
Long. Level:      AL ±3mm | IL ±5mm | IAL ±7mm
Cant (Dever):     AL ±15mm| IL ±20mm| IAL ±25mm
Twist:            AL ±3mm/m| IL ±5mm/m| IAL ±7mm/m
```
> Gerçek limit tablosu iletildiğinde `services/googlesheets.js` → `CHANNEL_META` içindeki `limits` değerleri güncellenecek.

---

## Notlara Göre Durum

### Not 1 — Heatmap: Traversler / Raylar Filtresi ✅

IoT Heatmaps panelinde "Filtre" dropdown'ı ile "Tüm Elemanlar", "Traversler" veya "Raylar" seçilebiliyor. Seçilmeyen tipler gri görünüyor, seçilen tipler üzerinde ısı haritası uygulanıyor. `Kilometraj_KPM` BIM özelliği üzerinden sensör↔eleman eşlemesi yapılıyor.

Model'deki eleman tipleri: `Baluster-Track_Tile_ADP` (1667 travers) · `Component_A-0-0` / `Component_A-0-0 (0)` (3 ray segmenti)

---

### Not 2 — Çoklu Ölçüm Tarihi (3-4 Dataset) ⏳

Altyapı hazır — `googlesheets.js`'deki `yearSheets` dizisine yeni yıl eklenince otomatik çalışır. Zaman slider'ı da eklenebilir. **Ek ölçüm dosyaları geldikçe uygulanacak.**

---

### Not 3 — Km Aralığı Grafiği ✅

Viewer toolbar'ına yeni buton eklendi. Km başlangıç/bitiş, kanal ve yıl seçilerek grafik çiziliyor. X ekseninde `281+66.00, 281+66.25…` formatı, y ekseninde ölçüm değeri. AL/IL/IAL limit çizgileri grafik üzerinde gösteriliyor.

---

### Not 4 — "Ekartman" Terminoloji ✅

`services/googlesheets.js` güncellendi, arayüzde artık "Ekartman" görünüyor.

---

### Not 5 — Yapay Zekâ Destekli Otomatik Raporlama 📅

Gereksinimler toplantıda netleştirilecek. Teknik olarak mümkün olan özellikler:
- Limit aşan noktaların listesi (Km, parametre, seviye)
- Excel/PDF rapor çıktısı
- Model ekran görüntüsü (`getScreenShot()` API)

---

### Not 6 — Uydu Görüntüsü / Altlık Veri 📅

Projede `GoogleMapsLocator` extension mevcut. Modelin WGS84 koordinatları bilinirse Leaflet.js veya Google Maps entegrasyonu yapılabilir. Toplantıda netleştirilecek.

---

### Not 7 — Üç Seviyeli Limit (AL / IL / IAL) ✅ / ⏳

Yapı tamamlandı: her kanal için `limits: { AL, IL, IAL }` tanımlı. Heatmap'te sarı/turuncu/kırmızı, grafiklerde kesikli referans çizgileri aktif. **Gerçek limit sayısal değerleri tablo gelince güncellenecek.**

Şu anki placeholder değerleri:
```
Ekartman (Gauge): AL ±3mm  | IL ±6mm  | IAL ±9mm
Alignment:        AL ±3mm  | IL ±5mm  | IAL ±7mm
Long. Level:      AL ±3mm  | IL ±5mm  | IAL ±7mm
Cant (Dever):     AL ±15mm | IL ±20mm | IAL ±25mm
Twist:            AL ±3mm/m| IL ±5mm/m| IAL ±7mm/m
```
