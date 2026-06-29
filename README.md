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

**Hedef:** F1 ≥ 0.92

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

**Hedef:** F1 ≥ 0.88

---

### M3 — Hallucination Rate (Halüsinasyon Oranı)

**Soru:** MCP, Figma tasarımında veya Ark dokümantasyonunda olmayan şeyleri uydurmuş mu?

**Yöntem:** Hibrit (otomatik + LLM-as-Judge)

**İki halüsinasyon kategorisi:**

#### A) Kod Halüsinasyonu (Otomatik)

| Tip | Ne demek | Nasıl tespit edilir |
|---|---|---|
| Fabricated Component | Ark'ta var olmayan bir component kullanılmış | Component isimlerini Ark export listesiyle karşılaştır |
| Fabricated Prop | Component var ama type definition'da tanımsız bir prop kullanılmış | `tsc --noEmit` type-check |
| Catalog Mismatch | Component Ark'ta var ama yanlış catalog'dan (efa/cfa/wsp) seçilmiş | Catalog eşleşme kontrolü |
| Phantom Component | Figma'da olmayan ama Ark'ta var olan bir component eklenmiş | M1'deki precision kontrolünden düşenler — Figma'da yok ama üretilmiş |
| Phantom Prop | Figma'da olmayan bir prop eklenmiş | M2'deki precision kontrolünden düşenler |

#### B) Görsel Halüsinasyon (LLM-as-Judge)

Figma screenshot ile üretilen render screenshot'ı karşılaştırılarak tespit edilir:

```
İki görsel veriyorum.
Birincisi orijinal Figma tasarımı, ikincisi üretilen kodun tarayıcı render'ı.

Şu kriterlere göre karşılaştır:
1. Figma'da olup çıktıda OLMAYAN elemanlar → listele (eksik eleman)
2. Figma'da olmayıp çıktıda OLAN elemanlar → listele (fazla eleman)
3. Var ama görsel olarak yanlış olan elemanlar → listele (renk, boyut, konum hatası)

Her madde için elemanın ne olduğunu ve nerede olduğunu belirt.
Yalnızca JSON formatında cevap ver.
```

**Formül:**

```
Kod Hallucination Rate = (Fabricated + Catalog Mismatch + Phantom öğeler)
                         / Toplam üretilen öğe sayısı × 100

Görsel Hallucination Rate = (Eksik + Fazla + Görsel yanlış eleman)
                            / Toplam eleman sayısı × 100

Toplam Hallucination Rate = (Kod × 0.4) + (Görsel × 0.6)
```

**Hedef:** Toplam < %25, Kod halüsinasyonu < %10

---

### M4 — Visual Fidelity (Görsel Sadakat)

**Soru:** Üretilen TSX tarayıcıda render edildiğinde Figma tasarımına ne kadar benziyor?

**Yöntem:** Hibrit (otomatik pixel karşılaştırma + LLM-as-Judge)

**Bu, pipeline'ın en kritik metriğidir.** Diğer metrikler component ve prop seviyesinde doğruluğu ölçerken, bu metrik "bütün olarak sonuç doğru görünüyor mu" sorusunu cevaplar.

**İki ölçüm paralel çalışır:**

#### A) Pixel-Level Karşılaştırma (Otomatik)

1. Üretilen TSX, Playwright ile headless browser'da render edilir
2. Sabit viewport boyutunda screenshot alınır
3. Figma screenshot ile karşılaştırılır

```
SSIM (Structural Similarity Index): 0 ile 1 arası
  0 = tamamen farklı
  1 = piksel piksel aynı
```

SSIM, basit pixel diff'ten daha anlamlıdır çünkü 1px kayma bile pixel diff'i şişirirken SSIM yapısal benzerliğe odaklanır.

Araçlar: Pixelmatch, Resemble.js veya Playwright built-in visual comparison.

**Not:** Figma screenshot ile tarayıcı render'ı arasında font rendering, anti-aliasing gibi doğal farklar olacaktır. SSIM = 1.0 beklemek gerçekçi değildir. Baseline olarak ilk golden set çalıştırmasında ortalamayı ölçüp hedefi buna göre kalibre edin.

#### B) Semantik Değerlendirme (LLM-as-Judge)

```
İki görsel veriyorum. Sol: orijinal Figma tasarımı. Sağ: üretilen kodun render'ı.

Aşağıdaki her boyutu 1-10 arası puanla:
1. Layout sadakati — Elemanların konumları, hiyerarşisi, akış yönü doğru mu?
2. Spacing/sizing — Padding, margin, gap, element boyutları uyumlu mu?
3. Tipografi — Font boyutu, ağırlığı, hizalama doğru mu?
4. Renk ve stil — Renkler, border, shadow, radius uyumlu mu?
5. İçerik bütünlüğü — Tüm metin, ikon ve görseller mevcut mu?

Her boyut için kısa açıklama yaz. Sonunda genel skor ver (1-10).
Yalnızca JSON formatında cevap ver.
```

**Bileşik skor:**

```
Visual Fidelity = (SSIM × 0.4) + (LLM-Judge Skoru / 10 × 0.6)
```

**Hedef:** ≥ 0.75

---

### M5 — Compilability (Derlenebilirlik)

**Soru:** Üretilen TSX dosyası hatasız derlenebiliyor mu?

**Yöntem:** Otomatik

**Nasıl çalışır:**

1. Üretilen TSX dosyası, Ark library'nin kurulu olduğu bir proje ortamına kopyalanır
2. `tsc --noEmit` komutu çalıştırılır
3. Sonuç: derlendi (1) veya derlenmedi (0)

```
Compilability Rate = Derlenen dosya sayısı / Toplam dosya sayısı × 100
```

**Derlenmeyenlerde hata tipi de kaydedilir:**

| Hata Tipi | Örnek |
|---|---|
| Import hatası | Component bulunamadı, yanlış path |
| Type hatası | Prop tipi uyumsuz, eksik zorunlu prop |
| Syntax hatası | JSX tag'i kapanmamış, parantez eksik |

Bu dağılım MCP'nin sistematik olarak hangi alanda zayıf olduğunu gösterir.

**Hedef:** ≥ %90

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

**Hedef:** ≥ %70

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

**Hedef:** ≥ 0.82

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

**Hedef:** P95 < 3000 ms

---

## Özet Tablo

| # | Metrik | Ne ölçer | Ground Truth | Yöntem | Hedef | Ağırlık |
|---|---|---|---|---|---|---|
| M1 | Component Detection F1 | Doğru component seçilmiş mi | Figma'daki component listesi | Otomatik (AST parse) | F1 ≥ 0.92 | %15 |
| M2 | Prop Accuracy F1 | Prop ve variant'lar doğru aktarılmış mı | Figma'daki prop/variant bilgileri | Otomatik (AST + tsc) | F1 ≥ 0.88 | %15 |
| M3 | Hallucination Rate | Uydurma veya fazla öğe var mı | Ark export listesi + Figma screenshot | Hibrit (AST + LLM-as-Judge) | < %25 | %15 |
| M4 | Visual Fidelity | Render Figma'ya benziyor mu | Figma screenshot | Hibrit (SSIM + LLM-as-Judge) | ≥ 0.75 | %25 |
| M5 | Compilability | Kod derleniyor mu | Ark type definitions | Otomatik (tsc --noEmit) | ≥ %90 | %10 |
| M6 | Code Quality | Kod prensiplerine uyuyor mu | Şirket kod prensipleri | LLM-as-Judge / human review | ≥ %70 | %10 |
| M7 | Consistency | Her seferinde benzer sonuç mu | Kendi çıktıları arası | Otomatik (diff + SSIM) | ≥ 0.82 | %5 |
| M8 | Latency | Ne kadar sürüyor | — | Otomatik (süre ölçümü) | P95 < 3s | %5 |

**Genel Performans Skoru (M0):**

```
M0 = Σ (Metrik skoru × Ağırlık)
Hedef: M0 ≥ 0.75
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
| `tsc --noEmit` | Üretilen TSX + Ark type definitions | Derleme sonucu + hata listesi | M2 (type check), M5 |
| Ark export listesi + catalog karşılaştırma | Üretilen component isimleri | Fabricated/mismatch raporu | M3 (kod) |
| SSIM: Figma screenshot ↔ render screenshot | İki screenshot | Benzerlik skoru | M4 (pixel) |
| Diff: 5 TSX birbirleriyle | 5 üretilen TSX | Değişen satır oranı | M7 (kod) |
| SSIM: 5 render screenshot birbirleriyle | 5 screenshot | Tutarlılık skoru | M7 (görsel) |
| Süre istatistikleri | Çalıştırma süreleri | P50, P95, P99 | M8 |

### Adım 5 — LLM-as-Judge

| Değerlendirme | Girdi | Metrik |
|---|---|---|
| Görsel sadakat puanlama | Figma screenshot + render screenshot | M4 (semantik) |
| Eksik/fazla eleman tespiti | Figma screenshot + render screenshot | M3 (görsel) |
| Kod prensipleri kontrolü | Üretilen TSX + şirket kod prensipleri | M6 |

### Adım 6 — Human Review (örneklemin %20-30'u)

LLM-as-Judge'ın güvenilirliğini kalibre etmek için golden set'in %20-30'u insan tarafından da değerlendirilir:

- LLM-Judge skorlarıyla insan skorları karşılaştırılır
- Korelasyon < 0.7 ise LLM prompt'ları iyileştirilir
- Edge case'ler ve LLM'in kaçırdığı hatalar belirlenir

### Adım 7 — Raporlama

- Metrik bazında skorlar
- Zorluk seviyesine göre kırılım (basit/orta/karmaşık)
- Catalog bazında kırılım (efa/cfa/wsp)
- Hata kategorileri dağılımı
- Genel M0 skoru
- Önceki çalıştırmalarla karşılaştırma (regresyon kontrolü)

---

## LLM-as-Judge Güvenilirlik Kontrolleri

LLM-as-Judge kullanılan metriklerde (M3, M4, M6) sonuçların güvenilir olması için:

| Kontrol | Nasıl yapılır | Kabul kriteri |
|---|---|---|
| Tekrar edilebilirlik | Aynı örneği 3 kez ayrı LLM çağrılarıyla değerlendir | Skorlar arası std sapma < 1.5 |
| İnsan kalibrasyonu | İlk 10 örneği hem LLM hem insan değerlendirsin | Pearson korelasyonu ≥ 0.7 |
| Anchoring | Prompt'a "bu 3 puan, bu 8 puan" referans örnekleri ekle | Judge'ın tutarlılığı artar |

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

**Bu metrikler neden standart bir LLM eval framework'ünden alınmamış?**
Standart framework'ler soru-cevap formatındaki LLM çıktıları için tasarlanmış. Bizim çıktımız kod olduğu için domain'e özel metrikler gerekiyor. Ancak metriklerin temeli global ve kanıtlanmış yöntemler: F1 score, SSIM, precision/recall. Bunları kendi dönüşüm pipeline'ımıza uyarladık.

**Catalog (efa/cfa/wsp) kontrolü neden önemli?**
Bir component teknik olarak Ark'ta var olabilir ama sayfanın ait olduğu catalog'a ait olmayabilir. Yanlış catalog'dan component kullanmak, uygulamada tutarsız UI veya import hataları yaratır.

**Hedef değerler nasıl belirlendi?**
İlk golden set çalıştırmasında baseline ölçülecek. Hedefler "ulaşılabilir ama zorlayıcı" olacak şekilde kalibre edilecek. Pipeline olgunlaştıkça hedefler yukarı çekilebilir.

**Visual Fidelity neden en yüksek ağırlıklı?**
Figma→kod dönüşümünün temel vaadi "Figma'daki tasarımı koda çevirmek." Bir geliştirici render'a baktığında Figma'yı görmüyorsa, component ve prop'lar teknik olarak doğru olsa bile dönüşüm başarısız demektir.
