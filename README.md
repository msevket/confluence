# Figma → Ark-React MCP: Değerlendirme Metrikleri

**Sahip:** [İsim]
**Son güncelleme:** [Tarih]
**Durum:** Aktif

---

## Bu Sayfa Ne İçin Var?

Figma tasarımlarını Ark-React koduna dönüştüren MCP pipeline'ımızın çıktı kalitesini nasıl ölçtüğümüzü anlatır. Buradaki bilgilerle herhangi biri eval sürecini baştan sonra çalıştırabilir, sonuçları yorumlayabilir ve yeni golden set örnekleri ekleyebilir.

---

## Pipeline'ın Çalışma Şekli

MCP'ye üç girdi girer, bir çıktı üretir:

| Girdi | Açıklama |
|---|---|
| Figma tasarım verisi | Figma'dan çekilen, LLM'in işleyebileceği şekilde sadeleştirilmiş yapısal bilgi (component isimleri, prop'lar, variant'lar, hiyerarşi, spacing vb.) |
| Figma screenshot | Sayfanın piksel-bazlı görsel referansı |
| Ark component dokümantasyonu + kod prensipleri | Şirketin Ark component library dokümantasyonu, kullanım kuralları ve system prompt |

| Çıktı | Açıklama |
|---|---|
| TSX dosyası | Ark library kullanılarak yazılmış, tek bir sayfanın React kodu |

Bu çıktının iki eksende doğru olması gerekiyor: **kod olarak doğru** (derlenebilir, doğru component ve prop kullanımı) ve **görsel olarak sadık** (tarayıcıda render edildiğinde Figma tasarımına benzer).

---

## Ground Truth: Figma'nın Kendisi

Değerlendirme için ayrı bir referans TSX dosyasına ihtiyacımız yok. **Figma tasarımı ground truth'tur** çünkü Figma'da şu bilgiler zaten mevcuttur:

- Hangi component'ların kullanıldığı (isimleriyle birlikte)
- Her component'ın prop'ları ve variant seçimleri
- Component hiyerarşisi (hangi element hangisinin içinde)
- Spacing, sizing, renk, tipografi değerleri

Bu sayede Figma'daki component/prop listesini üretilen TSX'teki component/prop listesiyle doğrudan karşılaştırıp F1 score hesaplayabiliyoruz.

---

## Golden Set

Golden set, tüm metriklerin üzerinde ölçüldüğü referans veri setidir.

### Bir Golden Set Örneğinin Yapısı

Her örnek şu 3 parçadan oluşur:

| # | Parça | Açıklama |
|---|---|---|
| 1 | Figma linki | İlgili sayfanın/frame'in Figma URL'i (component isimleri, prop'lar, hiyerarşi buradan çekilir) |
| 2 | Catalog bilgisi | Component'ın ait olduğu catalog: **efa**, **cfa** veya **wsp** |
| 3 | Ekran görüntüsü | Figma'dan alınmış sayfanın screenshot'ı |

### Zorluk Dağılımı

Golden set'te 20–30 arası sayfa bulunmalı, 3 zorluk seviyesine dağılmış:

| Seviye | Adet | Örnek Sayfalar |
|---|---|---|
| Basit | 8–10 | Tek bölümlü sayfalar: buton grubu, basit form, tek card |
| Orta | 8–10 | Çok bölümlü sayfalar: form + card grid, liste + filtre, tab yapısı |
| Karmaşık | 6–8 | Katmanlı sayfalar: tablo + modal, dashboard layout, nested componentlar |

### Golden Set'e Yeni Örnek Ekleme

1. Figma'dan frame'i belirle, linkini kaydet
2. Hangi catalog'a (efa/cfa/wsp) ait olduğunu belirle
3. Figma'dan screenshot al
4. 3 bilgiyi golden set'e ekle, zorluk seviyesini etiketle

---

## Metrikler

Toplamda 8 metrik ölçüyoruz.

---

### M1 — Component Detection F1 (Component Tespiti)

**Soru:** Figma'daki her component için Ark'ın doğru karşılığı seçilmiş mi?

**Yöntem:** Otomatik (Figma verisi + AST parse)

**Nasıl çalışır:**

1. Figma'dan sayfadaki tüm component isimleri çıkarılır → **Beklenen Set**
2. Üretilen TSX'teki tüm Ark component isimleri AST parser ile çıkarılır → **Üretilen Set**
3. İkisi karşılaştırılır

**Formüller:**

```
Precision = |Beklenen ∩ Üretilen| / |Üretilen|
  → "Ürettiklerinin ne kadarı Figma'da gerçekten var?"

Recall = |Beklenen ∩ Üretilen| / |Beklenen|
  → "Figma'daki componentların ne kadarını üretmiş?"

F1 = 2 × Precision × Recall / (Precision + Recall)
```

**Somut örnek:**

Figma'da 5 component var: `Button, Input, Select, Card, Modal`
MCP'nin ürettiği TSX'te 5 component var: `Button, Input, Select, Card, Dropdown`

- `Dropdown` Figma'da yok → Precision = 4/5 = 0.80
- `Modal` üretilmemiş → Recall = 4/5 = 0.80
- F1 = 2 × 0.80 × 0.80 / (0.80 + 0.80) = **0.80**

**Ek kontrol — Catalog doğruluğu:**

Eşleşen her component için, golden set'te belirtilen catalog'a (efa/cfa/wsp) ait olup olmadığı da kontrol edilir. Doğru component ama yanlış catalog = yanlış sayılır.

**Teknik uygulama:** Babel veya TypeScript AST parser ile üretilen TSX parse edilir, JSX element isimleri otomatik çıkarılır. Figma API veya sadeleştirilmiş Figma verisinden beklenen component listesi elde edilir.

**Hedef:** Baseline ölçümünden sonra belirlenecek

---

### M2 — Prop Accuracy F1 (Prop Doğruluğu)

**Soru:** Figma'da tanımlı prop ve variant seçimleri üretilen koda doğru yansımış mı?

**Yöntem:** Otomatik (Figma verisi + AST parse + TypeScript type-check)

**Nasıl çalışır:**

1. Figma'daki her component'ın prop/variant bilgileri çıkarılır → **Beklenen Prop Seti**
   (Örn: Button → variant=primary, size=large, disabled=false)
2. Üretilen TSX'teki eşleşen component'ın prop'ları AST ile çıkarılır → **Üretilen Prop Seti**
3. İkisi karşılaştırılır

**Üç alt boyut:**

| Alt Boyut | Kontrol | Hata Örneği |
|---|---|---|
| Prop ismi | Figma'daki prop/variant üretilmiş mi, adı doğru mu? | Figma'da `variant=primary` var, TSX'te `type="primary"` yazılmış |
| Prop değeri | Değer Figma'daki seçimle eşleşiyor mu? | Figma'da `size=large`, TSX'te `size="medium"` |
| Fazla prop | Figma'da olmayan bir prop eklenmiş mi? | Figma'da `loading` yok ama TSX'te `loading={true}` var |

**Formüller:**

```
Prop Precision = Doğru prop sayısı / Üretilen toplam prop sayısı
  → "Yazdığı prop'ların ne kadarı Figma ile uyumlu?"

Prop Recall = Doğru prop sayısı / Figma'daki toplam prop sayısı
  → "Figma'daki prop'ların ne kadarını doğru aktarmış?"

Prop F1 = 2 × Precision × Recall / (Precision + Recall)
```

**Ek kontrol — Type geçerliliği:**

Figma karşılaştırmasından bağımsız olarak, kullanılan her prop'un Ark'ın TypeScript type definition'ına uygun olup olmadığı da `tsc --noEmit` ile kontrol edilir. Figma ile eşleşse bile Ark'ta geçersiz bir prop = type hatası olarak kaydedilir.

**Hedef:** Baseline ölçümünden sonra belirlenecek

---

### M3 — Hallucination Rate (Halüsinasyon Oranı)

**Soru:** MCP, Figma tasarımında olmayan şeyleri ne oranda uydurmuş?

**Yöntem:** M1 ve M2'den türetilen bileşik metrik (ayrı bir test yapılmaz)

**M3 neden ayrı bir ölçüm değil?**

Halüsinasyon = "olmayan bir şeyi üretmek" demektir. Bu tam olarak M1 ve M2'deki **precision** metriğinin yakaladığı şeydir:

- M1 Precision düşükse → Figma'da olmayan componentlar üretilmiş (component halüsinasyonu)
- M2 Precision düşükse → Figma'da olmayan prop'lar eklenmiş (prop halüsinasyonu)

Dolayısıyla M3 için ayrı bir test çalıştırmıyoruz. M1 ve M2'nin precision değerlerinden türetiyoruz.

**Önemli ayrım — Precision vs Recall:**

| | Ne ölçer | Örnek |
|---|---|---|
| **Precision hatası** | Üretilen ama olmaması gereken şey → **halüsinasyon** | Figma'da `Modal` yok ama TSX'te `Modal` var |
| **Recall hatası** | Olması gereken ama üretilmemiş şey → **eksiklik** | Figma'da `Modal` var ama TSX'te yok |

Halüsinasyon sadece precision tarafıdır. Bir component'ı atlamak "uydurma" değil, "unutma"dır. Bu yüzden formülde F1 değil, precision kullanılır.

**Formül:**

```
M3 Hallucination Rate = 1 - (M1 Precision + M2 Precision) / 2
```

**Somut örnek:**

```
M1 Component Precision = 0.90 → ürettiklerinin %10'u Figma'da yok
M2 Prop Precision      = 0.85 → yazdığı prop'ların %15'i Figma'da yok
M3 = 1 - (0.90 + 0.85) / 2 = 1 - 0.875 = 0.125 → %12.5 halüsinasyon
```

**Hedef:** Baseline ölçümünden sonra belirlenecek

---

### M4 — Visual Fidelity (Görsel Sadakat)

**Soru:** Üretilen TSX tarayıcıda render edildiğinde Figma tasarımına ne kadar benziyor?

**Yöntem:** İki bağımsız değerlendirme, iki ayrı skor

**Bu, pipeline'ın en kritik metriğidir.** Diğer metrikler component ve prop seviyesinde doğruluğu ölçerken, bu metrik "bütün olarak sonuç doğru görünüyor mu" sorusunu cevaplar.

**Neden pixel karşılaştırma (SSIM) kullanmıyoruz:** Figma screenshot ile tarayıcı render'ı arasında font rendering, anti-aliasing, sub-pixel farklar gibi doğal farklılıklar var. Bu farklar görsel olarak anlamsız olmasına rağmen SSIM skorunu düşürür ve gürültülü sonuçlar üretir. "Benziyor mu" sorusunu insan gözü ve LLM-as-Judge çok daha anlamlı cevaplar.

**Değerlendirme sorusu (hem insan hem LLM için aynı):**

> "Bu render Figma tasarımına ne kadar benziyor? 1-10 arası puanla."

Alt boyut veya kriter listesi yok. Tek bir genel skor. Detaylı kırılımı (hangi component yanlış, hangi prop eksik) zaten M1 ve M2 yapıyor. M4'ün işi sadece görsel bütünlüğü ölçmek.

#### M4-Human (İnsan Değerlendirmesi)

Değerlendirici Figma screenshot ile render screenshot'ını yan yana görür ve 1-10 arası tek bir puan verir.

```
M4-Human = İnsan skoru / 10
```

#### M4-LLM (LLM-as-Judge Değerlendirmesi)

Aynı iki görsel vision destekli bir LLM'e verilir:

```
İki görsel veriyorum.
Birincisi orijinal Figma tasarımı, ikincisi üretilen kodun tarayıcı render'ı.
Bu render Figma tasarımına ne kadar benziyor? 1-10 arası puanla.
Kısa bir gerekçe yaz.
Yalnızca JSON formatında cevap ver: {"score": X, "reason": "..."}
```

```
M4-LLM = LLM skoru / 10
```

#### İki Skor Arasındaki Fark

İki skor ayrı ayrı raporlanır, birleştirilmez. Aradaki fark kendi başına anlamlı bir sinyaldir:

| Durum | Ne anlama gelir |
|---|---|
| İki skor yakın (fark ≤ 1 puan) | LLM ve insan aynı şeyi görüyor, değerlendirme güvenilir |
| İnsan düşük, LLM yüksek | LLM'in kaçırdığı görsel sorunlar var — LLM prompt'u iyileştirilmeli |
| İnsan yüksek, LLM düşük | LLM gereksiz yere cezalandırıyor — LLM prompt'u iyileştirilmeli |

Bu fark analizi, hem LLM prompt'unu geliştirmek hem de MCP pipeline'ındaki görsel sorunları anlamak için doğrudan feedback sağlar.

**Hedef:** Baseline ölçümünden sonra belirlenecek

---

### M5 — Compilability (Derlenebilirlik)

**Soru:** MCP'nin tek seferde ürettiği TSX dosyası hiçbir düzeltme yapılmadan derlenebiliyor mu?

**Yöntem:** Otomatik

**Önemli:** Bu metrik MCP'nin **ilk çıktısını** ölçer. Hata alınıp 1-2 düzeltme yapılırsa muhtemelen her kod derlenecektir — ama asıl soru MCP'nin ilk seferde çalışır kod üretip üretemediğidir.

**Nasıl çalışır:**

1. MCP'nin ürettiği TSX dosyası, Ark library'nin kurulu olduğu projeye olduğu gibi kopyalanır (hiçbir düzeltme yapılmaz)
2. `npm start` çalıştırılır
3. Sonuç: **derlendi** veya **derlenmedi**

**Hata alanlarda hata tipi de kaydedilir:**

| Hata Tipi | Örnek |
|---|---|
| Import hatası | Component bulunamadı, yanlış path |
| Type hatası | Prop tipi uyumsuz, eksik zorunlu prop |
| Syntax hatası | JSX tag'i kapanmamış, parantez eksik |

Bu dağılım MCP'nin sistematik olarak hangi alanda zayıf olduğunu gösterir.

**Hedef:** Baseline ölçümünden sonra belirlenecek

---

### M6 — Code Quality (Kod Kalitesi)

**Soru:** Üretilen kod şirketin kod prensiplerine uyuyor mu?

**Yöntem:** LLM-as-Judge veya human review

**5 kriter, her biri 0-2 puan:**

| Kriter | 0 puan | 1 puan | 2 puan |
|---|---|---|---|
| Naming convention | Standart dışı, tutarsız | Kısmen uyumlu | Prensiplere tam uyumlu |
| Styling yaklaşımı | Inline style, Ark token'ları yok | Kısmen token kullanımı | Ark'ın önerdiği yöntem tam uygulanmış |
| Component composition | Gereksiz deep nesting, anlamsız wrapperlar | Kısmen mantıklı bölünmüş | Temiz, okunabilir hiyerarşi |
| Import düzeni | Dağınık, gereksiz importlar | Kısmen düzenli | Gruplu, temiz, sadece gerekli importlar |
| Gereksiz kod | Kullanılmayan değişkenler, boş handlerlar var | Birkaç gereksiz parça | Temiz, fazlalık yok |

```
Code Quality = Toplam puan / 10 × 100
```

**Hedef:** Baseline ölçümünden sonra belirlenecek

---

### M7 — Consistency (Tutarlılık)

**Soru:** Aynı Figma frame'i her dönüştürüldüğünde benzer sonuç veriyor mu?

**Yöntem:** Otomatik (diff + SSIM)

**Nasıl çalışır:**

1. Aynı golden set örneği **5 kez** MCP'den geçirilir
2. Kod tutarlılığı: 5 TSX dosyası birbirleriyle diff edilir, değişen satır oranı hesaplanır
3. Görsel tutarlılık: 5 render screenshot'ı birbirleriyle SSIM karşılaştırılır

```
Kod Tutarlılığı    = 1 - (Ortalama değişen satır oranı)
Görsel Tutarlılık  = 5 render arasındaki ortalama SSIM
Consistency Score  = (Kod Tutarlılığı × 0.5) + (Görsel Tutarlılık × 0.5)
```

**Hedef:** Baseline ölçümünden sonra belirlenecek

---

### M8 — Latency (Gecikme)

**Soru:** MCP'nin bir Figma frame'ini TSX'e dönüştürmesi ne kadar sürüyor?

**Yöntem:** Otomatik (süre ölçümü)

**Nasıl çalışır:**

1. Her golden set örneği 10 kez çalıştırılır
2. Her çalıştırmada: istek gönderilme anı → TSX yanıtı gelme anı arası ölçülür (ms)
3. Sonuçlar frame zorluk seviyesine göre segmente edilir

```
P50  = Medyan (çalıştırmaların yarısı bunun altında)
P95  = Çalıştırmaların %95'i bunun altında
P99  = Çalıştırmaların %99'u bunun altında
```

**Hedef:** Baseline ölçümünden sonra belirlenecek

---

## Özet Tablo

| # | Metrik | Ne ölçer | Ground Truth | Yöntem | Hedef | Ağırlık |
|---|---|---|---|---|---|---|
| M1 | Component Detection F1 | Doğru component seçilmiş mi | Figma'daki component listesi | Otomatik (AST parse) | Baseline sonrası | %15 |
| M2 | Prop Accuracy F1 | Prop ve variant'lar doğru aktarılmış mı | Figma'daki prop/variant bilgileri | Otomatik (AST + tsc) | Baseline sonrası | %15 |
| M3 | Hallucination Rate | Uydurma component/prop oranı | M1 ve M2 precision değerleri | Türetilmiş (ayrı test yok) | Baseline sonrası | %15 |
| M4-Human | Visual Fidelity (İnsan) | Render Figma'ya benziyor mu | Figma screenshot | Human review | Baseline sonrası | %15 |
| M4-LLM | Visual Fidelity (LLM) | Render Figma'ya benziyor mu | Figma screenshot | LLM-as-Judge | Baseline sonrası | %10 |
| M5 | Compilability | İlk seferde derleniyor mu | Ark projesi | Otomatik (npm start) | Baseline sonrası | %10 |
| M6 | Code Quality | Kod prensiplerine uyuyor mu | Şirket kod prensipleri | LLM-as-Judge / human review | Baseline sonrası | %10 |
| M7 | Consistency | Her seferinde benzer sonuç mu | Kendi çıktıları arası | Otomatik (diff + SSIM) | Baseline sonrası | %5 |
| M8 | Latency | Ne kadar sürüyor | — | Otomatik (süre ölçümü) | Baseline sonrası | %5 |

**Genel Performans Skoru (M0):**

```
M0 = Σ (Metrik skoru × Ağırlık)
Hedef: Baseline ölçümünden sonra belirlenecek
```

---

## Eval Pipeline (Test Nasıl Çalıştırılır)

### Adım 1 — Hazırlık

- Golden set'ten test edilecek örnekleri belirle
- Her örneğin Figma linkine gidip component ve prop listesini çıkar (veya otomatik Figma API ile çek)
- Ark type definition'larının ve catalog bazlı component listelerinin (efa/cfa/wsp) güncel olduğunu doğrula
- Playwright'ın çalıştığını doğrula

### Adım 2 — MCP Çalıştırma

- Her golden set frame'i için Figma verisini ve screenshot'ı MCP'ye gönder
- Tutarlılık testi için her frame'i **5 kez** çalıştır
- Her çalıştırmanın süresini kaydet
- Üretilen TSX dosyalarını sakla

### Adım 3 — Render

- Her üretilen TSX'i Playwright ile headless browser'da render et
- Sabit viewport boyutunda screenshot al

### Adım 4 — Otomatik Kontroller

| Kontrol | Girdi | Çıktı | Metrik |
|---|---|---|---|
| AST parse → üretilen component listesi ↔ Figma component listesi | Figma verisi + üretilen TSX | F1 skoru + eşleşme raporu | M1 |
| AST parse → üretilen prop listesi ↔ Figma prop listesi | Figma verisi + üretilen TSX | Prop F1 skoru | M2 |
| `tsc --noEmit` | Üretilen TSX + Ark type definitions | Type hata listesi | M2 (type check) |
| `npm start` (düzeltme yapmadan) | Üretilen TSX + Ark projesi | Derlendi / derlenmedi + hata tipi | M5 |
| Diff: 5 TSX birbirleriyle | 5 üretilen TSX | Değişen satır oranı | M7 (kod) |
| SSIM: 5 render screenshot birbirleriyle | 5 screenshot | Tutarlılık skoru | M7 (görsel) |
| Süre istatistikleri | Çalıştırma süreleri | P50, P95, P99 | M8 |

**M3 (Hallucination Rate)** bu adımda ayrıca hesaplanmaz. M1 ve M2'nin precision değerleri hesaplandıktan sonra `M3 = 1 - (M1 Precision + M2 Precision) / 2` formülüyle türetilir.

### Adım 5 — LLM-as-Judge + Human Review

| Değerlendirme | Girdi | Metrik |
|---|---|---|
| Görsel sadakat puanlama (İnsan) | Figma screenshot + render screenshot | M4-Human |
| Görsel sadakat puanlama (LLM) | Figma screenshot + render screenshot | M4-LLM |
| Kod prensipleri kontrolü (LLM) | Üretilen TSX + şirket kod prensipleri | M6 |

İnsan ve LLM aynı görselleri aynı kriterlerle bağımsız olarak puanlar. İki skor ayrı ayrı raporlanır. Aradaki fark varsa neden farklı değerlendirdikleri incelenir — bu hem LLM prompt'unu hem MCP pipeline'ını geliştirmek için feedback sağlar.

### Adım 6 — Raporlama

- Metrik bazında skorlar
- M4-Human ve M4-LLM fark analizi
- Zorluk seviyesine göre kırılım (basit/orta/karmaşık)
- Catalog bazında kırılım (efa/cfa/wsp)
- Hata kategorileri dağılımı
- Genel M0 skoru
- Önceki çalıştırmalarla karşılaştırma (regresyon kontrolü)

---

## LLM-as-Judge Güvenilirlik Kontrolleri

LLM-as-Judge kullanılan metriklerde (M4-LLM, M6) tutarlılık için:

| Kontrol | Nasıl yapılır | Kabul kriteri |
|---|---|---|
| Tekrar edilebilirlik | Aynı örneği 3 kez ayrı LLM çağrılarıyla değerlendir | Skorlar arası std sapma < 1.5 |
| Anchoring | Prompt'a "bu 3 puan, bu 8 puan" referans örnekleri ekle | Judge'ın tutarlılığı artar |

M4-Human ile M4-LLM arasındaki fark analizi ayrıca LLM prompt'unun kalitesine dair doğal bir feedback mekanizması sağlar (detaylar M4 bölümünde).

---

## Ne Zaman Tekrar Çalıştırılmalı

Bu eval pipeline'ı şu durumlarda yeniden çalıştırılmalıdır:

- MCP'nin system prompt'u değiştiğinde
- Ark library'ye yeni component eklendiğinde veya mevcut component güncellendiğinde
- Catalog yapısında değişiklik olduğunda (yeni catalog, component taşıma)
- Underlying LLM modeli değiştiğinde
- Figma veri çekme mantığı değiştiğinde

Herhangi bir metrikte önceki çalıştırmaya göre **> %5 düşüş** tespit edilirse regresyon alarmı tetiklenir.

---

## Sık Sorulan Sorular

**Referans TSX olmadan F1 score'u nasıl hesaplıyoruz?**
Figma tasarımının kendisi ground truth görevi görüyor. Figma'daki component isimleri ve prop/variant bilgileri "beklenen set"i oluşturuyor, üretilen TSX'teki component ve prop'lar "üretilen set"i oluşturuyor. Bu iki set karşılaştırılarak precision, recall ve F1 hesaplanıyor.

**M3 (Hallucination Rate) neden ayrı bir test değil?**
Halüsinasyon "olmayan bir şeyi üretmek" demektir. Bu zaten M1 ve M2'deki precision metriğinin yakaladığı şeydir: Figma'da olmayan bir component veya prop üretilmişse precision düşer. M3'ü ayrı ölçmek aynı şeyi iki kez saymak olurdu. Bunun yerine M1 ve M2'nin precision değerlerinden türetiyoruz. Böylece raporda "Hallucination Rate" satırı var ama arkasında mükerrer bir test yok.

**Bu metrikler neden standart bir LLM eval framework'ünden alınmamış?**
Standart framework'ler soru-cevap formatındaki LLM çıktıları için tasarlanmış. Bizim çıktımız kod olduğu için domain'e özel metrikler gerekiyor. Ancak metriklerin temeli global ve kanıtlanmış yöntemler: F1 score, SSIM, precision/recall. Bunları kendi dönüşüm pipeline'ımıza uyarladık.

**Catalog (efa/cfa/wsp) kontrolü neden önemli?**
Bir component teknik olarak Ark'ta var olabilir ama sayfanın ait olduğu catalog'a ait olmayabilir. Yanlış catalog'dan component kullanmak, uygulamada tutarsız UI veya import hataları yaratır.

**Hedef değerler nasıl belirlenecek?**
Hedefler önceden sabitlenmedi. İlk golden set çalıştırmasında her metriğin baseline değeri ölçülecek, bu değer referans alınarak "ulaşılabilir ama zorlayıcı" hedefler belirlenecek. Pipeline olgunlaştıkça hedefler yukarı çekilebilir.

**Visual Fidelity neden en yüksek ağırlıklı?**
Figma→kod dönüşümünün temel vaadi "Figma'daki tasarımı koda çevirmek." Bir geliştirici render'a baktığında Figma'yı görmüyorsa, component ve prop'lar teknik olarak doğru olsa bile dönüşüm başarısız demektir.
