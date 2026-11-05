# Jawaban: Toko nusantara (BigQuery)

## Soal 1

```sql
SELECT
  c.city,
  COUNT(DISTINCT o.order_id)      AS total_order,
  COUNT(DISTINCT c.customer_id)   AS total_pelanggan,
  SUM(oi.quantity * p.unit_price) AS revenue
FROM tokonusantara.orders AS o
JOIN tokonusantara.customers AS c   ON o.customer_id = c.customer_id
JOIN tokonusantara.order_items AS oi ON o.order_id = oi.order_id
JOIN tokonusantara.products AS p    ON oi.product_id = p.product_id
WHERE o.status = 'completed'
GROUP BY c.city
HAVING SUM(oi.quantity * p.unit_price) > 1000000
ORDER BY revenue DESC;
```

| city    | total_order | total_pelanggan | revenue  |
| ------- | ----------- | --------------- | -------- |
| Jakarta | 5           | 2               | 27020000 |
| Medan   | 2           | 1               | 1350000  |
| Bandung | 2           | 1               | 1150000  |

**Mengapa solusi ini:**

- Empat tabel dihubungkan dengan `INNER JOIN` karena yang dibutuhkan hanya kota yang memang punya order. Rantai join mengikuti relasi: orders → customers, orders → order_items → products.
- `WHERE o.status = 'completed'` menyaring **baris** sebelum pengelompokan sehingga order batal/retur tidak ikut dihitung.
- `COUNT(DISTINCT ...)` wajib: setelah join ke `order_items`, satu order muncul di banyak baris, jadi `COUNT(*)` akan menghitung berlebih.
- `HAVING` menyaring **kelompok** berdasarkan hasil agregat. Kota yang gugur: Makassar (750.000), Denpasar (680.000), Surabaya (600.000).

**Kesalahan umum:** menaruh kondisi revenue di `WHERE` (error, agregat tidak boleh di sana), atau memakai `COUNT(*)` sebagai jumlah order.

---

## Soal 2

```sql
SELECT
  FORMAT_DATE('%Y-%m', DATE_TRUNC(o.order_date, MONTH)) AS order_month,
  COUNT(DISTINCT o.order_id)                                   AS total_order,
  COUNT(DISTINCT IF(o.status = 'completed', o.order_id, NULL)) AS order_completed,
  COUNT(DISTINCT IF(o.status = 'cancelled', o.order_id, NULL)) AS order_cancelled,
  COUNT(DISTINCT IF(o.status = 'returned',  o.order_id, NULL)) AS order_returned,
  ROUND(SAFE_DIVIDE(
    COUNT(DISTINCT IF(o.status = 'cancelled', o.order_id, NULL)),
    COUNT(DISTINCT o.order_id)) * 100, 1)                      AS cancel_rate_pct,
  SUM(IF(o.status = 'completed', oi.quantity * p.unit_price, 0)) AS revenue_completed
FROM tokonusantara.orders AS o
JOIN tokonusantara.order_items AS oi ON o.order_id = oi.order_id
JOIN tokonusantara.products AS p ON oi.product_id = p.product_id
GROUP BY order_month
ORDER BY order_month;
```

| order_month | total_order | order_completed | order_cancelled | order_returned | cancel_rate_pct | revenue_completed |
| ----------- | ----------- | --------------- | --------------- | -------------- | --------------- | ----------------- |
| 2024-01     | 3           | 2               | 1               | 0              | 33.3            | 12790000          |
| 2024-02     | 4           | 3               | 0               | 1              | 0.0             | 2310000           |
| 2024-03     | 4           | 4               | 0               | 0              | 0.0             | 14180000          |
| 2024-04     | 4           | 3               | 1               | 0              | 25.0            | 2270000           |

**Mengapa solusi ini:**

- Dengan conditional aggregation, **satu kali scan** menghasilkan semua kolom status (pivot). Alternatifnya join berulang atau beberapa query, jauh lebih boros.
- Karena tabel sudah di-join ke `order_items`, jumlah order harus dihitung dengan `COUNT(DISTINCT IF(...))`. `COUNTIF(status = 'cancelled')` akan menghitung **baris item**, bukan order, dan angkanya salah bila order berisi lebih dari satu item.
- Revenue memakai `SUM(IF(status = 'completed', ..., 0))` sehingga hanya order completed yang masuk, tanpa mengorbankan baris status lain untuk kolom jumlah.
- `SAFE_DIVIDE` mencegah error pembagian nol pada bulan tanpa order.

**Kesalahan umum:** memfilter `WHERE status = 'completed'` (semua kolom status lain menjadi 0 atau hilang), dan mencampur jumlah order dengan jumlah baris item.

---

## Soal 3

```sql
WITH product_revenue AS (
  SELECT
    p.product_id, p.product_name, p.category,
    COALESCE(SUM(IF(o.status = 'completed', oi.quantity * p.unit_price, NULL)), 0) AS revenue
  FROM tokonusantara.products AS p
  LEFT JOIN tokonusantara.order_items AS oi ON p.product_id = oi.product_id
  LEFT JOIN tokonusantara.orders AS o       ON oi.order_id = o.order_id
  GROUP BY p.product_id, p.product_name, p.category
),
ranked AS (
  SELECT
    product_name, category, revenue,
    RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS rank_in_category
  FROM product_revenue
)
SELECT product_name, category, revenue, rank_in_category
FROM ranked
WHERE rank_in_category <= 2
ORDER BY category, rank_in_category;
```

| product_name    | category     | revenue  | rank_in_category |
| --------------- | ------------ | -------- | ---------------- |
| Laptop Ultra 14 | Elektronik   | 24000000 | 1                |
| Mouse Nirkabel  | Elektronik   | 1200000  | 2                |
| Sepatu Lari     | Fashion      | 2400000  | 1                |
| Jaket Hoodie    | Fashion      | 1500000  | 2                |
| Blender Mini    | Rumah Tangga | 1200000  | 1                |
| Rice Cooker     | Rumah Tangga | 0        | 2                |

(Tanpa filter top 2, produk peringkat 3 adalah Headset Gaming 450.000 dan Kaos Polos 800.000.)

**Mengapa solusi ini:**

- `LEFT JOIN` **dari `products`** menjaga produk yang belum pernah terjual (Rice Cooker) agar tetap tampil. Bila dimulai dari `order_items`, produk itu hilang.
- Syarat status `completed` ditaruh **di dalam agregat** (`SUM(IF(...))`), bukan di `WHERE`. Bila ditaruh di `WHERE`, baris produk tanpa order ikut tersaring dan LEFT JOIN berubah menjadi INNER JOIN. `COALESCE(..., 0)` mengubah NULL menjadi 0.
- Window function bekerja **setelah** `GROUP BY`, sehingga `RANK()` memeringkat nilai revenue yang sudah diagregasi per produk. Itu sebabnya agregasi dilakukan di CTE terpisah.
- `PARTITION BY category` membuat peringkat dihitung terpisah untuk tiap kategori.
- Window function tidak boleh dipakai di `WHERE`, maka filter top 2 dilakukan di query luar atau memakai `QUALIFY`.
- `RANK()` dipilih agar produk dengan revenue sama mendapat peringkat sama. Gunakan `ROW_NUMBER()` jika Anda butuh tepat N baris dan tie boleh diputus acak.

**Alternatif dengan `QUALIFY`:**

```sql
SELECT product_name, category, revenue,
       RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS rank_in_category
FROM product_revenue
WHERE TRUE
QUALIFY RANK() OVER (PARTITION BY category ORDER BY revenue DESC) <= 2
ORDER BY category, rank_in_category;
```

---

## Soal 4

```sql
WITH monthly_revenue AS (
  SELECT
    DATE_TRUNC(o.order_date, MONTH) AS order_month,
    SUM(oi.quantity * p.unit_price) AS revenue
  FROM tokonusantara.orders AS o
  JOIN tokonusantara.order_items AS oi ON o.order_id = oi.order_id
  JOIN tokonusantara.products AS p ON oi.product_id = p.product_id
  WHERE o.status = 'completed'
  GROUP BY order_month
),
with_neighbors AS (
  SELECT
    order_month, revenue,
    LAG(revenue)  OVER (ORDER BY order_month) AS revenue_bulan_lalu,
    LEAD(revenue) OVER (ORDER BY order_month) AS revenue_bulan_depan
  FROM monthly_revenue
)
SELECT
  order_month,
  revenue,
  revenue_bulan_lalu,
  revenue - revenue_bulan_lalu AS selisih,
  ROUND(SAFE_DIVIDE(revenue - revenue_bulan_lalu, revenue_bulan_lalu) * 100, 1) AS growth_pct,
  revenue_bulan_depan,
  CASE
    WHEN revenue_bulan_lalu IS NULL THEN 'Awal'
    WHEN revenue > revenue_bulan_lalu THEN 'Naik'
    ELSE 'Turun'
  END AS tren
FROM with_neighbors
ORDER BY order_month;
```

| order_month | revenue  | revenue_bulan_lalu | selisih   | growth_pct | revenue_bulan_depan | tren  |
| ----------- | -------- | ------------------ | --------- | ---------- | ------------------- | ----- |
| 2024-01-01  | 12790000 | NULL               | NULL      | NULL       | 2310000             | Awal  |
| 2024-02-01  | 2310000  | 12790000           | -10480000 | -81.9      | 14180000            | Turun |
| 2024-03-01  | 14180000 | 2310000            | 11870000  | 513.9      | 2270000             | Naik  |
| 2024-04-01  | 2270000  | 14180000           | -11910000 | -84.0      | NULL                | Turun |

**Mengapa solusi ini:**

- Window function harus bekerja pada tabel dengan **grain satu baris per bulan**, maka revenue diagregasi lebih dulu di CTE `monthly_revenue`. Memakai `LAG` langsung pada data order akan membandingkan order dengan order, bukan bulan dengan bulan.
- `ORDER BY order_month` di dalam `OVER` menentukan arah waktu: `LAG` mengambil bulan sebelumnya, `LEAD` bulan berikutnya. Baris pertama/terakhir menghasilkan NULL karena tidak punya tetangga.
- `LAG` dan `LEAD` dihitung sekali di CTE kedua, lalu selisih dan persentase dihitung di query luar. Ini lebih rapi dan lebih mudah di-debug daripada mengulang `LAG(...) OVER (...)` berkali-kali.
- `SAFE_DIVIDE` menghindari error saat bulan sebelumnya 0; kolom `tren` memakai `CASE WHEN` yang mengevaluasi dari atas, jadi NULL (bulan pertama) ditangani lebih dulu.

**Catatan penting:** `LAG` mengambil **baris sebelumnya**, bukan **bulan kalender sebelumnya**. Bila ada bulan tanpa transaksi, bulan itu tidak muncul dan perbandingan menjadi menyesatkan. Solusi produksi: buat kerangka bulan lengkap dengan `GENERATE_DATE_ARRAY` lalu `LEFT JOIN` data revenue.

---

## Soal 5

```sql
WITH customer_revenue AS (
  SELECT
    c.customer_id,
    c.customer_name,
    c.region,
    COUNT(DISTINCT o.order_id)                   AS total_order,
    COALESCE(SUM(oi.quantity * p.unit_price), 0) AS revenue
  FROM tokonusantara.customers AS c
  LEFT JOIN tokonusantara.orders AS o
    ON c.customer_id = o.customer_id AND o.status = 'completed'
  LEFT JOIN tokonusantara.order_items AS oi ON o.order_id = oi.order_id
  LEFT JOIN tokonusantara.products AS p     ON oi.product_id = p.product_id
  GROUP BY c.customer_id, c.customer_name, c.region
)
SELECT
  customer_name,
  region,
  total_order,
  revenue,
  ROUND(SAFE_DIVIDE(revenue, SUM(revenue) OVER (PARTITION BY region)) * 100, 1) AS kontribusi_pct_wilayah,
  DENSE_RANK() OVER (PARTITION BY region ORDER BY revenue DESC)                 AS rank_wilayah,
  CASE
    WHEN revenue >= 10000000 THEN 'Platinum'
    WHEN revenue >= 1000000  THEN 'Gold'
    WHEN revenue > 0         THEN 'Silver'
    ELSE 'Dormant'
  END AS segment,
  CASE
    WHEN total_order >= 2 THEN 'repeat'
    WHEN total_order = 1  THEN 'one-time'
    ELSE 'never'
  END AS tipe_pembeli
FROM customer_revenue
ORDER BY revenue DESC;
```

| customer_name   | region     | total_order | revenue  | kontribusi_pct_wilayah | rank_wilayah | segment  | tipe_pembeli |
| --------------- | ---------- | ----------- | -------- | ---------------------- | ------------ | -------- | ------------ |
| Andi Pratama    | Jawa       | 4           | 26120000 | 90.8                   | 1            | Platinum | repeat       |
| Dewi Anggraini  | Sumatra    | 2           | 1350000  | 100.0                  | 1            | Gold     | repeat       |
| Budi Santoso    | Jawa       | 2           | 1150000  | 4.0                    | 2            | Gold     | repeat       |
| Gilang Ramadhan | Jawa       | 1           | 900000   | 3.1                    | 3            | Silver   | one-time     |
| Eko Wijaya      | Sulawesi   | 1           | 750000   | 100.0                  | 1            | Silver   | one-time     |
| Fitri Handayani | Bali-Nusra | 1           | 680000   | 100.0                  | 1            | Silver   | one-time     |
| Citra Lestari   | Jawa       | 1           | 600000   | 2.1                    | 4            | Silver   | one-time     |
| Hana Putri      | Sumatra    | 0           | 0        | 0.0                    | 2            | Dormant  | never        |

**Mengapa solusi ini:**

- **CTE** memisahkan dua pekerjaan: (1) agregasi ke grain **satu baris per pelanggan**, (2) pelabelan dan perbandingan antar pelanggan. Window function harus dijalankan setelah grain pelanggan terbentuk.
- **`LEFT JOIN` dari `customers`** dengan syarat status di klausa `ON` menjaga pelanggan tanpa order completed (Hana Putri). Bila syarat itu dipindah ke `WHERE`, Hana lenyap dan label `Dormant` tidak pernah muncul. `COALESCE` mengubah revenue NULL menjadi 0 agar `CASE` dan window bekerja benar.
- **`SUM(revenue) OVER (PARTITION BY region)`** menghitung total wilayah **tanpa menghilangkan baris pelanggan**, sehingga kontribusi persen bisa dihitung per baris. `GROUP BY` tidak bisa melakukan ini dalam satu langkah.
- **`DENSE_RANK`** dipilih karena soal meminta peringkat tanpa lompatan nomor.
- **`CASE WHEN`** dievaluasi dari atas ke bawah dan berhenti pada kondisi pertama yang cocok, sehingga urutan `>= 10.000.000` lebih dulu, lalu `>= 1.000.000`, lalu `> 0` menghasilkan segmentasi yang benar. `ELSE` menangkap revenue 0.
- Citra Lestari punya order batal (1003) dan order completed (1010); hanya yang completed dihitung, maka total order-nya 1.

**Kesalahan umum:** memfilter status di `WHERE` (pelanggan tanpa order hilang), memakai `RANK` (nomor melompat bila ada nilai kembar), dan menghitung window sebelum agregasi per pelanggan selesai.

---
