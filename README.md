# Toko-nusantara-Bigquery
## 1. Skema Dataset: TokoNusantara

Toko online fiktif dengan 4 tabel yang saling berelasi.

| Relasi | Keterangan |
| --- | --- |
| `customers` 1 — N `orders` | Satu pelanggan bisa punya banyak order |
| `orders` 1 — N `order_items` | Satu order berisi banyak baris produk |
| `products` 1 — N `order_items` | Satu produk bisa muncul di banyak order |
| `customers.referred_by` → `customers.customer_id` | Relasi ke tabel sendiri (pelanggan yang mengajak) |

**Aturan bisnis:** revenue = `quantity × unit_price` dan hanya dihitung dari order berstatus `completed`. Status lain: `cancelled`, `returned`.

### Tabel `customers` (8 baris, 5 pertama)

| customer\_id | customer\_name | city | region | signup\_date | referred\_by |
| --- | --- | --- | --- | --- | --- |
| 1 | Andi Pratama | Jakarta | Jawa | 2023-01-10 | NULL |
| 2 | Budi Santoso | Bandung | Jawa | 2023-02-15 | 1 |
| 3 | Citra Lestari | Surabaya | Jawa | 2023-03-05 | 1 |
| 4 | Dewi Anggraini | Medan | Sumatra | 2023-03-20 | 2 |
| 5 | Eko Wijaya | Makassar | Sulawesi | 2023-04-11 | NULL |

### Tabel `products` (8 baris, 5 pertama)

| product\_id | product\_name | category | unit\_price |
| --- | --- | --- | --- |
| 101 | Laptop Ultra 14 | Elektronik | 12000000 |
| 102 | Mouse Nirkabel | Elektronik | 150000 |
| 103 | Headset Gaming | Elektronik | 450000 |
| 104 | Kaos Polos | Fashion | 80000 |
| 105 | Jaket Hoodie | Fashion | 250000 |

### Tabel `orders` (15 baris, 5 pertama)

| order\_id | customer\_id | order\_date | status |
| --- | --- | --- | --- |
| 1001 | 1 | 2024-01-05 | completed |
| 1002 | 2 | 2024-01-12 | completed |
| 1003 | 3 | 2024-01-20 | cancelled |
| 1004 | 1 | 2024-02-03 | completed |
| 1005 | 4 | 2024-02-14 | completed |

### Tabel `order_items` (23 baris, 5 pertama)

| order\_item\_id | order\_id | product\_id | quantity |
| --- | --- | --- | --- |
| 1 | 1001 | 101 | 1 |
| 2 | 1001 | 102 | 2 |
| 3 | 1002 | 104 | 3 |
| 4 | 1002 | 105 | 1 |
| 5 | 1003 | 103 | 1 |

## 8. Soal Latihan Bertingkat

### Soal 1 (menengah):

Tim regional ingin mengetahui kota yang kontribusinya signifikan. Tampilkan, **hanya dari order completed**, untuk setiap kota: jumlah order, jumlah pelanggan unik, dan total revenue. Munculkan hanya kota dengan revenue **di atas Rp 1.000.000**, urutkan dari revenue terbesar.

Kolom: `city`, `total_order`, `total_pelanggan`, `revenue`.

### Soal 2 (menengah+): 

Manajer operasional meminta laporan bulanan status order. Untuk setiap bulan, tampilkan jumlah order total, order `completed`, `cancelled`, `returned`, persentase pembatalan (satu desimal), dan revenue dari order completed. Peringatan: setelah join ke `order_items`, sebuah order muncul di banyak baris.

Kolom: `order_month` (format YYYY-MM), `total_order`, `order_completed`, `order_cancelled`, `order_returned`, `cancel_rate_pct`, `revenue_completed`.

### Soal 3 (lanjut): 

Tim merchandising ingin peringkat produk di dalam kategorinya berdasarkan revenue (order completed). **Semua produk harus tampil, termasuk yang belum pernah terjual** (revenue 0). Setelah itu, ambil hanya **2 produk teratas per kategori**.

Kolom: `product_name`, `category`, `revenue`, `rank_in_category`.

### Soal 4 (lanjut): 

Buat laporan revenue bulanan (order completed) lengkap dengan revenue bulan sebelumnya, selisih, persentase pertumbuhan month-over-month (satu desimal), dan revenue bulan berikutnya. Bonus: tambahkan kolom `tren` bernilai `Naik`, `Turun`, atau `Awal` (bulan pertama).

Kolom: `order_month`, `revenue`, `revenue_bulan_lalu`, `selisih`, `growth_pct`, `revenue_bulan_depan`.

### Soal 5 (kompleks): 

Buat ringkasan 360° pelanggan untuk tim CRM. **Semua pelanggan harus muncul**, termasuk yang belum pernah membeli. Hitung dari order completed: jumlah order, total revenue, kontribusi (%) terhadap total revenue wilayahnya (satu desimal), dan peringkat di dalam wilayahnya (tanpa lompatan nomor). Tambahkan dua label dengan `CASE WHEN`:

| Label | Aturan |
| --- | --- |
| `segment` | `Platinum` jika revenue ≥ 10.000.000; `Gold` jika ≥ 1.000.000; `Silver` jika > 0; selain itu `Dormant` |
| `tipe_pembeli` | `repeat` jika order ≥ 2; `one-time` jika = 1; selain itu `never` |

Kolom: `customer_name`, `region`, `total_order`, `revenue`, `kontribusi_pct_wilayah`, `rank_wilayah`, `segment`, `tipe_pembeli`. Urutkan dari revenue terbesar.
