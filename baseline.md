# Gün 1 — Baseline (indekssiz)

**Tablo:** `orders`, 5.000.000 satır
**Ortam:** PostgreSQL 18.4, WSL2 Ubuntu 26.04
**Tablo boyutu:** ~54.450 buffer × 8 KB ≈ **425 MB**
**shared_buffers:** `SHOW shared_buffers;` → **128 MB**

Not: `Workers Launched: 2` + lider süreç = **3 paralel süreç**. `loops=3` olan düğümlerde `rows` ve `Rows Removed` değerleri **worker başına**; toplam için ×3.

---

## Özet tablo

| # | Sorgu | Plan düğümü | Tahmini (toplam) | Gerçek (toplam) | Süre (ms) | shared hit / read | Filtrede elenen |
|---|---|---|---|---|---|---|---|
| 1 | son 30 gün paid, LIMIT 50 | Parallel Seq Scan → Sort (top-N heapsort) → Gather Merge | 171.159 | 136.242 | 1370 | 15.859 / 38.683 | 4.86M |
| 2 | user_id = 42 | Parallel Seq Scan → Gather | 26 | 25 | 174 / 1401 | 15.794 / 38.672 | 5.0M |
| 3 | status bazında COUNT + SUM | Parallel Seq Scan → Partial HashAggregate → Finalize GroupAggregate | 6.250.000 | 5.000.000 | 627 | 15.832 / 38.650 | 0 |
| 4 | amount > 950 sayısı | Parallel Seq Scan → Partial Aggregate | 320.607 | 250.681 | 245 | 15.822 / 38.644 | 4.75M |
| 5 | email LIKE '%gmail.com' sayısı | Parallel Seq Scan → Partial Aggregate | 1.325.757 | 1.250.488 | 263 | 15.843 / 38.623 | 3.75M |

**Beşinde de ortak:** `Parallel Seq Scan on orders`, `read ≈ 38.650`. Her sorgu tablonun tamamını okuyor.

---

## Sorgu 1 — son 30 günün paid siparişleri

```sql
SELECT * FROM orders
WHERE status = 'paid' AND created_at > now() - interval '30 days'
ORDER BY created_at DESC
LIMIT 50;
```

- `Filter: ((status = 'paid') AND (created_at > (now() - '30 days')))`
- `Rows Removed by Filter: 1.621.253` (worker başına)
- `Sort Method: top-N heapsort  Memory: 36kB`
- Sort düğümü: `actual time=1059.045..1059.054` — altındaki tarama: `0.518..1052.514`

**Gözlem:** Sıralama pratikte bedava (~7 ms). `LIMIT 50` sayesinde 136 bin satır tam sıralanmadı, sadece en iyi 50'yi tutan yığın kullanıldı. Süre tamamen taramada.

**Notum:**
> İndeks okunacak veri miktarını 54.450 sayfadan ~55 sayfaya düşürürdü, ayrıca sıralama adımını tamamen ortadan kaldırır

---

## Sorgu 2 — user_id = 42

```sql
SELECT * FROM orders WHERE user_id = 42;
```

- Tahmin `rows=26`, gerçek `rows=25` → **tahmin neredeyse kusursuz**
- `Rows Removed by Filter: 1.666.658` (worker başına) = toplam 5M
- İki çalıştırma: 174 ms ve 1401 ms

**Gözlem:** Planner sadece ~26 satır döneceğini biliyordu ve yine de 425 MB okudu. Bilgi eksikliği değil, **erişim yolu** eksikliği: `user_id` üzerinde indeks yok, dolayısıyla tek seçenek tam tarama.

Tahminin nereden geldiği `pg_stats`'ta görünüyor: `user_id` için `n_distinct = 188.806`. 5.000.000 / 188.806 ≈ **26,5**. Planner tam olarak bu bölmeyi yapmış.

**İki çalıştırma karşılaştırması (hipotez testi):** İkinci çalıştırma daha *yavaş* çıktı ve `hit` değeri artmadı (15.794 → 15.853). Yani "ikinci sefer cache'ten gelir, hızlanır" beklentisi burada gerçekleşmedi.

**Notum:**
> shared buffer 128 mb, buna 425 mblik tablo sığdırılamaz. bu sorguda zaten neredeyse tüm tabloyu döndüğünden cacheye aldığı alan sınırlıdır sorgu 2. kezde hızlanmaz.

---

## Sorgu 3 — status bazında COUNT + SUM

```sql
SELECT status, COUNT(*), SUM(amount) FROM orders GROUP BY status;
```

- `Filter` satırı **yok**, `Rows Removed by Filter` **yok**
- `Partial HashAggregate` → `Batches: 1  Memory Usage: 32kB`
- Tarama 172 ms'de bitiyor, HashAggregate 598 ms'de

**Gözlem:** Tek "dürüst" tam tarama. 5M satırın hepsi gerçekten gerekli, çünkü hepsi toplanacak. Diğer dördünde tarama israftı; burada değil.

`status` için `n_distinct = 4` ve `most_common_vals = {paid, shipped, pending, cancelled}` — planner 4 grup çıkacağını önceden biliyor, `rows=4` tahmini bu yüzden isabetli.

**Notum:**
> (status, amount) üzerine indeksleme yapsaydık tüm kolonlar okunmak yerine sadece bu 2 kolona odaklanır ve daha hızlı olabilirdi

---

## Sorgu 4 — amount > 950 sayısı

```sql
SELECT COUNT(*) FROM orders WHERE amount > 950;
```

- Tahmin 106.869/worker → toplam ~320.607
- Gerçek 83.560/worker → toplam 250.681
- **Sapma: %28 fazla tahmin**

**Gözlem:** `amount` için `n_distinct = 96.873`, `most_common_vals` boş — yani baskın bir değer yok, dağılım düzgün. Planner bu durumda histogram üzerinden kestirim yapıyor ve kestirim yaklaşık. Bu sorguda zararsız: %28 sapma plan seçimini değiştirmiyor. Sapma 100 kat olsaydı yanlış strateji seçilebilirdi.

**Notum:**
> istatistikleri ANALYZE toplar. 1M satır daha ekleseydik tablonun yaklaşık %10u değiştiği için ANALYZE otomatik olarak histogram istatistiklerini toplar ama hemen olmaz. toplu veri girişlerinde ANALYZE elle tetiklenir. " SELECT last_analyze, last_autoanalyze FROM pg_stat_user_tables WHERE relname='orders'; " 

---

## Sorgu 5 — email LIKE '%gmail.com' sayısı

```sql
SELECT COUNT(*) FROM orders WHERE email LIKE '%gmail.com';
```

- `Filter: (email ~~ '%gmail.com'::text)` — `~~` operatörü `LIKE`'ın iç gösterimi
- Tahmin 441.919/worker, gerçek 416.829/worker → **iyi tahmin**
- `email` için `n_distinct = 491.033`

**Gözlem:** Desenin **başında** `%` var. Bu, sıradan bir B-tree indeksinin işe yaramayacağı anlamına gelir — nedeni Gün 2'de test edilecek.

**Notum:**
> gmail.com un zaten sonu sabittir btree değerleri baştan sıralı tutar indexten faydalanamaz. eğer gmail% şeklinde yazsaydık indeksten faydalanabilirdi

---

## pg_stats çıktısı

```
  attname   | n_distinct |         most_common_vals
------------+------------+----------------------------------
 id         |         -1 |
 user_id    |     188806 |
 email      |     491033 |
 status     |          4 | {paid,shipped,pending,cancelled}
 amount     |      96873 |
 created_at |         -1 |
```

`n_distinct = -1` → her satırda farklı değer (benzersiz). `id` primary key olduğu için beklenen; `created_at` de rastgele üretildiği için pratikte benzersiz.

---

## Gün 1 sonucu — kendi cümlelerimle

> **Soru:** 5M satırlık tabloda 25 satır dönen sorgu bile neden tüm tabloyu okuyor? Planner "tahmini satır" sayısını nereden buluyor, gerçek satırdan saptığında ne olur?
>
> herhangi bir indeks olmadığı için bulmak için tüm tabloyu gezmek zorunda kalıyor. planner histogram dağılımı kullanarak tahmin eder, gerçek satırdan eğer çok fazla saparsa planner yanlış bir yöntem seçip sorguyu uzatıp karmaşıklaştırabilir.

## Gün 2 hedefi

Yukarıdaki 5 satırdaki `Parallel Seq Scan`'lerin en az 4'ünü `Index Scan`'e çevirip süreleri **50 ms altına** indirmek. Bu tablo "önce / sonra / kaç kat" haline gelecek.