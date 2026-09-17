# yokla

**Teknik mülakat soruları ve puanlanan mülakat simülasyonu — 11 alan, 50 uzmanlık, 29.400 soru, TR + EN.**

🔗 **[yokla.dev](https://yokla.dev?utm_source=github&utm_medium=readme&utm_campaign=yokla-dev)** · Ücretsiz, kayıt gerekmiyor.

---

## Ne yapar

Alan, kıdem ve uzmanlık seçersin (ör. *backend · mid · Go*), 20 soruluk puanlanan bir mülakata
girersin. Sonunda skorunu, konu bazlı dağılımını, her sorunun açıklamalı çözümünü ve aynı
konfigürasyonda mülakata girmiş diğer adaylara göre yüzdelik dilimini görürsün.

Elindeki iş ilanının metnini yapıştırıp mülakatı ilana göre kurdurabilirsin de.

| | |
|---|---|
| **Alan** | backend · frontend · database · devops-cloud · mobile · security · qa-test-automation · data-engineer · game-dev · ai-engineer · ml-engineer (+ fullstack karışımı) |
| **Uzmanlık** | 50 (Go, Python, Java, C#/.NET, Node.js, PHP, Rust, React, Vue, Angular, TypeScript, PostgreSQL, MySQL, MongoDB, Kubernetes, AWS, Terraform, Kafka, Spark, Airflow, LLM/RAG, classical-ml, deep-learning, …) |
| **Kıdem** | junior · mid · senior |
| **Soru** | 29.400 (1176 hücre × 25), tamamı TR + EN |
| **Fiyat** | ücretsiz |

## Örnek mülakat soruları

Mülakata girmeden soruları cevapları ve açıklamalarıyla inceleyebilirsin:

- [Tüm alanlar](https://yokla.dev/mulakat-sorulari?utm_source=github&utm_medium=readme&utm_campaign=yokla-dev)
- [Backend mülakat soruları](https://yokla.dev/mulakat-sorulari/backend) ·
  [Frontend](https://yokla.dev/mulakat-sorulari/frontend) ·
  [Veritabanı](https://yokla.dev/mulakat-sorulari/database) ·
  [DevOps & Cloud](https://yokla.dev/mulakat-sorulari/devops-cloud) ·
  [Mobil](https://yokla.dev/mulakat-sorulari/mobile) ·
  [Güvenlik](https://yokla.dev/mulakat-sorulari/security) ·
  [QA & Test Otomasyonu](https://yokla.dev/mulakat-sorulari/qa-test-automation) ·
  [Data Engineering](https://yokla.dev/mulakat-sorulari/data-engineer) ·
  [Oyun Geliştirme](https://yokla.dev/mulakat-sorulari/game-dev) ·
  [AI Engineering](https://yokla.dev/mulakat-sorulari/ai-engineer) ·
  [ML Engineering](https://yokla.dev/mulakat-sorulari/ml-engineer)

---

## Teknik taraf

Ürün kodu kapalı (soru bankası imaja gömülü). Ama mühendislik kararlarının çoğu ve
yakalanan bug'lar açık yazıldı — aşağıdakiler ölçülmüş, hatırlanmış değil.

### Stack
**Backend** Go 1.25 · Echo v5 · raw **pgx** (ORM yok) · PostgreSQL 18 · goose migration ·
Prometheus
**Frontend** Nuxt 4 (SSR) · Vue 3 · Pinia · Tailwind v4 · @nuxtjs/i18n (TR `/`, EN `/en`)
**Altyapı** Docker Swarm · GHCR · GitHub Actions (test gate'li CI, tag → deploy) · nginx ·
Cloudflare

Grafik kütüphanesi yok — skor trendi ve dağılım grafikleri bağımlılıksız inline SVG.
Paylaşım kartı (1200×630 PNG) headless browser olmadan, pure-Go üretiliyor.

### Ölçerek verilen "hayır" kararları

**pgvector reddedildi.** 29.400 soruluk korpusta benzerlik taraması `float4[]` kolonu +
uygulama tarafı kosinüs ile **0.5 saniye** sürüyor. Bir extension, bir index tipi ve bir
operatör sınıfı eklemek için gereken bakım maliyeti bu sayının karşılığı değildi.

**Redis reddedildi.** İhtiyaç duyulan tek şey oturum/limit sayaçlarıydı; Postgres zaten
oradaydı ve yükü taşıdı. Mail kuyruğu da broker yerine bir Postgres tablosu
(`FOR UPDATE SKIP LOCKED` + exponential backoff).

### Yakalanmış gerçek bug'lar

**Tek kullanıcıyla asla göremeyeceğin deadlock.** Mülakat tamamlama akışı, istatistik
sayaçlarını join sırasına göre kilitliyordu. 100 eşzamanlı tamamlama = **55 × HTTP 500,
p50 34 saniye**. Sıralı `FOR NO KEY UPDATE` CTE'ye çevrildikten sonra: **0 hata, p50 91 ms.**
10 eşzamanlı kullanıcı bile bir hata veriyordu — yani bu duyuru günü patlayacak bir bug'dı.

**Kişiselleştirilmiş sayfa nasıl cache'lenir.** Ana sayfa SSR tavanı ~31 istek/sn ölçüldü.
`cache: { swr: true, maxAge: 60, varies: ['cookie'] }` ile cookie'siz gelen herkes tek
render'ı paylaşıyor, oturumlu istek asla başkasının sayfasını almıyor: **189 → 2236 istek/sn.**

**Deploy penceresi.** İki replikalı rollout 30-50 saniyelik kesinti veriyordu. `--detach=false`
ile gerçek bekleme + convergence kontrolü + ana sayfa SWR cache'i birleşince, 271 örneklik
probda **en uzun kesinti 0 saniye**.

### Soru bankası nasıl üretildi

29.400 sorunun tamamı AI ile üretildi, ama asıl iş doğrulamada. Her uzmanlık hücresi
(6 topic × 3 kıdem × 25 = 450 soru) **dört fazdan** geçiyor:

1. **Üretim** — her topic ayrı bir ajanda, her sayısal iddia *fiilen çalıştırılıp ölçülerek*
2. **Bağımsız denetim** — başka bir ajan her iddiayı sıfırdan yeniden ölçer
3. **Zorunlu ikinci göz** — hiçbir şeyden haberi olmayan taze bir denetçi (salt-okunur)
4. **Hedefli düzeltme** — 3. fazın bulgularını *doğrulayarak* uygular; doğrulanamayan bulgu uygulanmaz

Kodu gerçekten çalıştırmak fark yaratıyor. Son üç hücrede üretici ajanların **kendi taslaklarında**
buldukları yanlış iddia sayısı: **29 → 37 → 144**. Çoğu yaygın klişenin tersi çıktı:

- *"Ağaç modelleri etkileşimi zaten öğrenir"* — derinlik-1'de **R² = −0.024**; açık çarpım
  sütunuyla 0.975
- *"Adam'da bias correction olmasa ilk adımlar küçük kalır"* — tersi, **3.16 kat büyük**
- *"Binning bilgi kaybıdır"* — doğrusal modelde 0.1 → **0.85**
- *"log dönüşümü çarpıklığı azaltır"* — sol çarpık veride −4.35 → **−14.95**

### Ölçülmeyen eksende felaket: "kör strateji"

Şıkların uzunluğu bir ipucu verirse aday soruyu okumadan doğru cevabı bulur. Araç bunu
sayıyordu, üretim turları da sayıyı aşağı çekiyordu.

Bir ajan bu işi **fazlasıyla iyi** yaptı: kendisine verilen iki metriği ("doğru şık en uzun",
"en kısa") **ikisini birden %0'a** indirdi ve raporunda başarı olarak bildirdi. Diskten ölçünce
doğru cevabın 25 sorunun **21'inde tam olarak 2. en uzun şık** olduğu çıktı — aday soruyu hiç
okumadan "uzunluğa göre sırala, ikinciyi seç" diyerek **%84** alıyordu.

Doğru metrik bir uç değil, dağılımın kendisi: **kör strateji = aday soruyu okumadan en iyi sabit
stratejiyle kaç alır.** Rastgele %25. Araç soru bankasının tamamına çevrilince 2208 dosya-dil ölçümünün
**1616'sı eşiğin üstünde** çıktı — yani metrik yazılana kadar biriken, kimsenin görmediği bir borç.

Sonraki hücrede aynı desenin **ikinci hali** bulundu: gerekçe bağlacı (`because` / `çünkü`)
**tek bir şıkta** geçtiğinde o şık neredeyse hiç doğru değildi (bir dosyada 18 tetikleme /
0 isabet). Aday soruyu okumadan o şıkkı *eleyerek* 4 seçenekten 3'e iniyordu. Sebep masum:
üreticiler distractor'ları gerekçelendiriyor, doğru şıkları düz bırakıyor.

---

## Bağlantılar

- 🌐 **[yokla.dev](https://yokla.dev?utm_source=github&utm_medium=readme&utm_campaign=yokla-dev)** — ürün
- 📄 [Hakkında](https://yokla.dev/hakkinda) · [Gizlilik](https://yokla.dev/gizlilik)
- 📚 [Mülakat soruları kataloğu](https://yokla.dev/mulakat-sorulari?utm_source=github&utm_medium=readme&utm_campaign=yokla-dev)

Soru, hata bildirimi veya geri bildirim için **Issues** açabilirsin.

## Lisans

Bu repo (dokümantasyon) MIT. Ürün kodu ve soru bankası kapalıdır.
