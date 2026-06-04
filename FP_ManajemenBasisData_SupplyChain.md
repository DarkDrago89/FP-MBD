# Final Project — Manajemen Basis Data
## Sistem Manajemen Supply Chain & Inventory
**Departemen Teknik Informatika — Institut Teknologi Sepuluh Nopember**

---

## Deskripsi Topik

Perusahaan distribusi barang menghadapi tantangan besar dalam mengelola alur rantai pasok secara efisien, mulai dari pemesanan ke supplier, pengelolaan stok gudang, hingga pemenuhan pesanan pelanggan. Ketidakakuratan data stok, keterlambatan pengiriman, dan kurangnya visibilitas terhadap status pesanan menjadi hambatan utama yang menurunkan efisiensi operasional.

Sistem basis data ini dirancang untuk mendukung tiga modul utama:

- **A. Manajemen Pemesanan ke Supplier (Purchase Order Management):** Mengelola seluruh siklus pembelian dari pembuatan PO, persetujuan, hingga penerimaan barang.
- **B. Manajemen Inventori & Gudang (Inventory & Warehouse Management):** Mengelola data stok barang, pergerakan barang masuk/keluar, dan lokasi penyimpanan di gudang.
- **C. Manajemen Pesanan Pelanggan (Sales Order Management):** Menangani penerimaan pesanan dari pelanggan, alokasi stok, dan pengiriman barang.

---

## Daftar Business Rule

### Modul A — Pemesanan ke Supplier

1. Setiap Purchase Order (PO) harus terhubung ke satu supplier yang aktif.
2. Satu PO dapat memuat lebih dari satu item barang (PO detail).
3. PO harus melewati status: `DRAFT` → `APPROVED` → `SENT` → `PARTIALLY_RECEIVED` → `COMPLETED` atau `CANCELLED`.
4. Hanya PO berstatus `APPROVED` yang dapat dikirimkan ke supplier.
5. Kuantitas barang yang diterima tidak boleh melebihi kuantitas yang dipesan dalam PO.
6. Setiap penerimaan barang (goods receipt) wajib mencatat tanggal, jumlah, dan kondisi barang.
7. Jika semua item PO telah diterima penuh, status PO otomatis berubah menjadi `COMPLETED`.

### Modul B — Inventori & Gudang

1. Setiap barang memiliki satu SKU unik dan terhubung ke satu kategori produk.
2. Stok barang dicatat per lokasi gudang (warehouse location/bin).
3. Setiap pergerakan stok (stock movement) harus mencatat tipe: `IN`, `OUT`, atau `ADJUSTMENT`.
4. Stok tidak boleh bernilai negatif; sistem harus menolak transaksi pengeluaran yang melebihi stok tersedia.
5. Barang dengan stok di bawah reorder point harus ditandai untuk pembuatan PO otomatis.
6. Setiap perubahan stok wajib menyimpan referensi dokumen sumber (PO number atau SO number).

### Modul C — Pesanan Pelanggan

1. Setiap Sales Order (SO) terhubung ke satu pelanggan terdaftar.
2. Satu SO dapat memuat lebih dari satu item barang (SO detail).
3. SO melewati status: `PENDING` → `CONFIRMED` → `PICKING` → `SHIPPED` → `DELIVERED` atau `CANCELLED`.
4. Stok hanya dialokasikan (reserved) setelah SO berstatus `CONFIRMED`.
5. Pengiriman hanya dapat dibuat untuk SO yang berstatus `PICKING` atau lebih lanjut.
6. Jika stok tidak mencukupi saat konfirmasi SO, sistem harus mengembalikan pesan error dan membatalkan alokasi.
7. Harga jual pada SO detail mengacu pada harga saat SO dibuat, bukan harga saat pengiriman.

---

## Daftar Database Transaction

| No | Nama Transaksi | Deskripsi | Tabel yang Terlibat |
|----|----------------|-----------|----------------------|
| T1 | Create Purchase Order | Membuat PO beserta detail item secara atomik | `purchase_order`, `po_detail` |
| T2 | Goods Receipt | Menerima barang dari supplier, update stok, dan update status PO | `goods_receipt`, `gr_detail`, `inventory`, `po_detail`, `purchase_order` |
| T3 | Confirm Sales Order | Konfirmasi SO dan alokasi stok secara atomik | `sales_order`, `so_detail`, `inventory` |
| T4 | Stock Adjustment | Penyesuaian stok manual dengan pencatatan alasan | `inventory`, `stock_movement` |
| T5 | Ship Sales Order | Proses pengiriman SO: kurangi stok, buat dokumen shipment | `sales_order`, `shipment`, `inventory`, `stock_movement` |
| T6 | Cancel Purchase Order | Pembatalan PO beserta pembalikan alokasi anggaran | `purchase_order`, `po_detail` |
| T7 | Transfer Stok Antar Gudang | Memindahkan stok dari satu lokasi ke lokasi lain secara atomik | `inventory`, `stock_movement` |

---

## Desain Database

### CDM (Conceptual Data Model) — Deskripsi Entitas & Relasi

```
[SUPPLIER] ──(1,N)── [PURCHASE_ORDER] ──(1,N)── [PO_DETAIL] ──(N,1)── [PRODUCT]
                             |
                         (1,N)
                             |
                      [GOODS_RECEIPT] ──(1,N)── [GR_DETAIL]

[PRODUCT] ──(N,1)── [PRODUCT_CATEGORY]
[PRODUCT] ──(1,N)── [INVENTORY] ──(N,1)── [WAREHOUSE_LOCATION]
[INVENTORY] ──(1,N)── [STOCK_MOVEMENT]

[CUSTOMER] ──(1,N)── [SALES_ORDER] ──(1,N)── [SO_DETAIL] ──(N,1)── [PRODUCT]
[SALES_ORDER] ──(1,1)── [SHIPMENT]
```

### PDM (Physical Data Model) — Skema Tabel

#### Tabel: `supplier`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| supplier_id | SERIAL | PK |
| supplier_code | VARCHAR(20) | UNIQUE NOT NULL |
| supplier_name | VARCHAR(100) | NOT NULL |
| contact_person | VARCHAR(100) | |
| phone | VARCHAR(20) | |
| email | VARCHAR(100) | |
| address | TEXT | |
| is_active | BOOLEAN | DEFAULT TRUE |
| created_at | TIMESTAMP | DEFAULT NOW() |

#### Tabel: `product_category`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| category_id | SERIAL | PK |
| category_name | VARCHAR(50) | NOT NULL |
| description | TEXT | |

#### Tabel: `product`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| product_id | SERIAL | PK |
| sku | VARCHAR(30) | UNIQUE NOT NULL |
| product_name | VARCHAR(150) | NOT NULL |
| category_id | INT | FK → product_category |
| unit | VARCHAR(20) | NOT NULL |
| unit_price | NUMERIC(15,2) | NOT NULL |
| reorder_point | INT | DEFAULT 0 |
| lead_time_days | INT | DEFAULT 7 |
| is_active | BOOLEAN | DEFAULT TRUE |

#### Tabel: `warehouse_location`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| location_id | SERIAL | PK |
| warehouse_code | VARCHAR(10) | NOT NULL |
| zone | VARCHAR(10) | |
| rack | VARCHAR(10) | |
| bin | VARCHAR(10) | |
| max_capacity | INT | |

#### Tabel: `inventory`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| inventory_id | SERIAL | PK |
| product_id | INT | FK → product |
| location_id | INT | FK → warehouse_location |
| qty_on_hand | INT | NOT NULL DEFAULT 0 |
| qty_reserved | INT | NOT NULL DEFAULT 0 |
| qty_available | INT | GENERATED ALWAYS AS (qty_on_hand - qty_reserved) STORED |
| last_updated | TIMESTAMP | DEFAULT NOW() |

#### Tabel: `stock_movement`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| movement_id | SERIAL | PK |
| product_id | INT | FK → product |
| location_id | INT | FK → warehouse_location |
| movement_type | VARCHAR(15) | CHECK IN ('IN','OUT','ADJUSTMENT','TRANSFER_IN','TRANSFER_OUT') |
| qty | INT | NOT NULL |
| reference_doc | VARCHAR(50) | |
| notes | TEXT | |
| created_by | VARCHAR(50) | |
| created_at | TIMESTAMP | DEFAULT NOW() |

#### Tabel: `purchase_order`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| po_id | SERIAL | PK |
| po_number | VARCHAR(30) | UNIQUE NOT NULL |
| supplier_id | INT | FK → supplier |
| order_date | DATE | NOT NULL |
| expected_date | DATE | |
| status | VARCHAR(25) | CHECK IN ('DRAFT','APPROVED','SENT','PARTIALLY_RECEIVED','COMPLETED','CANCELLED') |
| total_amount | NUMERIC(15,2) | |
| approved_by | VARCHAR(50) | |
| approved_at | TIMESTAMP | |
| notes | TEXT | |
| created_at | TIMESTAMP | DEFAULT NOW() |

#### Tabel: `po_detail`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| po_detail_id | SERIAL | PK |
| po_id | INT | FK → purchase_order |
| product_id | INT | FK → product |
| qty_ordered | INT | NOT NULL |
| qty_received | INT | DEFAULT 0 |
| unit_price | NUMERIC(15,2) | NOT NULL |

#### Tabel: `goods_receipt`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| gr_id | SERIAL | PK |
| gr_number | VARCHAR(30) | UNIQUE NOT NULL |
| po_id | INT | FK → purchase_order |
| receipt_date | DATE | NOT NULL |
| received_by | VARCHAR(50) | |
| notes | TEXT | |

#### Tabel: `gr_detail`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| gr_detail_id | SERIAL | PK |
| gr_id | INT | FK → goods_receipt |
| po_detail_id | INT | FK → po_detail |
| product_id | INT | FK → product |
| location_id | INT | FK → warehouse_location |
| qty_received | INT | NOT NULL |
| condition | VARCHAR(20) | CHECK IN ('GOOD','DAMAGED','PARTIAL') |

#### Tabel: `customer`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| customer_id | SERIAL | PK |
| customer_code | VARCHAR(20) | UNIQUE NOT NULL |
| customer_name | VARCHAR(100) | NOT NULL |
| contact_person | VARCHAR(100) | |
| phone | VARCHAR(20) | |
| email | VARCHAR(100) | |
| address | TEXT | |
| credit_limit | NUMERIC(15,2) | DEFAULT 0 |
| is_active | BOOLEAN | DEFAULT TRUE |

#### Tabel: `sales_order`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| so_id | SERIAL | PK |
| so_number | VARCHAR(30) | UNIQUE NOT NULL |
| customer_id | INT | FK → customer |
| order_date | DATE | NOT NULL |
| requested_date | DATE | |
| status | VARCHAR(20) | CHECK IN ('PENDING','CONFIRMED','PICKING','SHIPPED','DELIVERED','CANCELLED') |
| total_amount | NUMERIC(15,2) | |
| notes | TEXT | |
| created_at | TIMESTAMP | DEFAULT NOW() |

#### Tabel: `so_detail`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| so_detail_id | SERIAL | PK |
| so_id | INT | FK → sales_order |
| product_id | INT | FK → product |
| qty_ordered | INT | NOT NULL |
| qty_delivered | INT | DEFAULT 0 |
| unit_price | NUMERIC(15,2) | NOT NULL |

#### Tabel: `shipment`
| Kolom | Tipe | Constraint |
|-------|------|------------|
| shipment_id | SERIAL | PK |
| shipment_number | VARCHAR(30) | UNIQUE NOT NULL |
| so_id | INT | FK → sales_order |
| ship_date | DATE | |
| carrier | VARCHAR(50) | |
| tracking_number | VARCHAR(50) | NULL |
| status | VARCHAR(20) | CHECK IN ('PENDING','IN_TRANSIT','DELIVERED','RETURNED') |
| delivered_at | TIMESTAMP | NULL |

---

## Jumlah Data yang Diinsert per Tabel

| Tabel | Jumlah Record | Keterangan |
|-------|--------------|------------|
| supplier | 500 | Supplier aktif & non-aktif |
| product_category | 50 | Kategori produk |
| product | 5.000 | SKU aktif |
| warehouse_location | 2.000 | Lokasi bin per gudang |
| inventory | 200.000 | 1 record per product per lokasi |
| stock_movement | 500.000 | Riwayat pergerakan stok |
| purchase_order | 50.000 | PO selama 3 tahun |
| po_detail | 200.000 | Rata-rata 4 item per PO |
| goods_receipt | 45.000 | ~90% PO sudah diterima |
| gr_detail | 180.000 | Detail penerimaan |
| customer | 10.000 | Pelanggan aktif & non-aktif |
| sales_order | 80.000 | SO selama 3 tahun |
| so_detail | 240.000 | Rata-rata 3 item per SO |
| shipment | 75.000 | SO yang sudah dikirim |

**Total keseluruhan: ±1.587.550 record**

---

## Skenario Data yang Diinput

### Record Lengkap

**Purchase Order — PO-2024-00001**

```
po_id        : 1
po_number    : PO-2024-00001
supplier_id  : 12  (PT. Maju Jaya Elektronik)
order_date   : 2024-01-15
expected_date: 2024-01-29
status       : COMPLETED
total_amount : 45.750.000
approved_by  : admin_budiman
approved_at  : 2024-01-15 10:32:00
notes        : Pengadaan rutin Q1 2024
created_at   : 2024-01-15 09:00:00
```

**PO Detail**

```
po_detail_id : 1
po_id        : 1
product_id   : 88  (Kabel UTP Cat6, 305m)
qty_ordered  : 50
qty_received : 50
unit_price   : 450.000
```

```
po_detail_id : 2
po_id        : 1
product_id   : 91  (Switch 24-port)
qty_ordered  : 10
qty_received : 10
unit_price   : 3.075.000
```

### Record dengan Kolom NULL

**Sales Order — SO-2024-05892 (belum dikirim)**

```
so_id         : 5892
so_number     : SO-2024-05892
customer_id   : 203  (CV. Berkah Niaga)
order_date    : 2024-06-10
requested_date: 2024-06-17
status        : CONFIRMED
total_amount  : 12.500.000
notes         : NULL   ← tidak ada catatan khusus
created_at    : 2024-06-10 14:22:00
```

**Shipment terkait (belum dibuat)**

```
shipment_id      : NULL  ← belum ada shipment
tracking_number  : NULL  ← belum ada resi pengiriman
delivered_at     : NULL  ← belum terkirim
```

---

## Pembagian Tugas

| Anggota | Nama | Pekerjaan |
|---------|------|-----------|
| Anggota 1 | *(Nama)* | Pekerjaan 1 (CDM & PDM Modul A — Supplier & Purchase Order), Pekerjaan 2 (DDL + data dummy tabel Supplier, Product Category, Product) |
| Anggota 2 | *(Nama)* | Pekerjaan 1 (CDM & PDM Modul B — Inventory & Warehouse), Pekerjaan 2 (DDL + data dummy tabel Warehouse Location, Inventory, Stock Movement) |
| Anggota 3 | *(Nama)* | Pekerjaan 1 (CDM & PDM Modul C — Customer & Sales Order), Pekerjaan 2 (DDL + data dummy tabel Customer, Sales Order, SO Detail, Shipment), Pekerjaan 3 (Query 3 — Status Pengiriman Sales Order) |
| Anggota 4 | *(Nama)* | Pekerjaan 2 (DDL + data dummy tabel Purchase Order, PO Detail, Goods Receipt, GR Detail), Pekerjaan 3 (Query 1 — Daftar PO Sedang Diproses, Query 2 — Stok di Bawah Reorder Point), Pekerjaan 7 (Database Transaction — Goods Receipt) |
| Anggota 5 | *(Nama)* | Pekerjaan 4 (Trigger), Pekerjaan 5 (Function), Pekerjaan 6 (Procedure) |

---

## Pekerjaan 1 — CDM & PDM

**Deskripsi:**
CDM dan PDM mencakup tiga modul utama: Purchase Order Management, Inventory & Warehouse Management, dan Sales Order Management. CDM menggambarkan entitas dan relasi antar entitas secara konseptual tanpa mempertimbangkan implementasi fisik. PDM menjabarkan seluruh tabel beserta tipe data, constraint, dan foreign key yang siap diimplementasikan di PostgreSQL.

Detail skema lengkap tersedia pada bagian Desain Database di atas.

**Pembagian Tugas:**

| Anggota | Tugas |
|---------|-------|
| Anggota 1 | CDM & PDM Modul A: entitas Supplier, Purchase Order, PO Detail, Goods Receipt, GR Detail |
| Anggota 2 | CDM & PDM Modul B: entitas Product Category, Product, Warehouse Location, Inventory, Stock Movement |
| Anggota 3 | CDM & PDM Modul C: entitas Customer, Sales Order, SO Detail, Shipment |
| Anggota 4 | Review & integrasi CDM/PDM lintas modul, memastikan konsistensi foreign key antar modul |
| Anggota 5 | Finalisasi diagram CDM & PDM (penggambaran visual menggunakan tools seperti DBDesigner / ERDPlus) |

---

## Pekerjaan 2 — DDL & Data Dummy

**Deskripsi:**
DDL mencakup pembuatan seluruh tabel, index awal, dan constraint. Data dummy dibangkitkan menggunakan script Python/pgbench/Faker untuk memenuhi minimal 200.000 record per tabel utama. Setiap tabel memiliki skenario data yang realistis sesuai alur bisnis supply chain.

**Pembagian Tugas:**

| Anggota | Tugas |
|---------|-------|
| Anggota 1 | DDL + generate data dummy: tabel `supplier`, `product_category`, `product` |
| Anggota 2 | DDL + generate data dummy: tabel `warehouse_location`, `inventory`, `stock_movement` |
| Anggota 3 | DDL + generate data dummy: tabel `customer`, `sales_order`, `so_detail`, `shipment` |
| Anggota 4 | DDL + generate data dummy: tabel `purchase_order`, `po_detail`, `goods_receipt`, `gr_detail` |
| Anggota 5 | Validasi data dummy lintas tabel (referential integrity check), dokumentasi jumlah record per tabel |

---

## Pekerjaan 3 — Query dengan Indexing & Optimasi

### Query 1 — Daftar PO yang Sedang Diproses (Modul A)

**Deskripsi:**
Menampilkan seluruh Purchase Order berstatus `SENT` atau `PARTIALLY_RECEIVED` beserta nama supplier, total item, dan persentase penerimaan barang. Query ini digunakan tim purchasing untuk memonitor PO yang belum selesai.

**Index yang dibuat:** pada kolom `status` dan `supplier_id` di tabel `purchase_order`, serta kolom `po_id` di tabel `po_detail`.

**Penanggungjawab: Anggota 4**

---

### Query 2 — Laporan Stok di Bawah Reorder Point (Modul B)

**Deskripsi:**
Menampilkan semua produk yang stok tersedianya berada di bawah atau sama dengan reorder point, dikelompokkan per kategori. Digunakan tim warehouse untuk memicu pembuatan PO baru.

**Index yang dibuat:** pada kolom `product_id` dan `location_id` di tabel `inventory`, serta kolom `category_id` dan `reorder_point` di tabel `product`.

**Penanggungjawab: Anggota 4**

---

### Query 3 — Status Pengiriman Sales Order (Modul C)

**Deskripsi:**
Laporan status pengiriman SO dalam periode tertentu: menampilkan detail SO, data pelanggan, status shipment, dan keterlambatan pengiriman dibandingkan requested date. Digunakan tim sales dan logistik.

**Index yang dibuat:** pada kolom `status` dan `order_date` di tabel `sales_order`, serta kolom `so_id` dan `status` di tabel `shipment`.

**Penanggungjawab: Anggota 3**

---

**Pembagian Tugas Pekerjaan 3:**

| Anggota | Tugas |
|---------|-------|
| Anggota 1 | Review query & validasi hasil output Query 1 |
| Anggota 2 | Review query & validasi hasil output Query 2 |
| Anggota 3 | Membuat dan menguji Query 3 beserta indexing tabel Sales Order & Shipment |
| Anggota 4 | Membuat dan menguji Query 1 & Query 2 beserta indexing tabel PO & Inventory |
| Anggota 5 | Dokumentasi hasil EXPLAIN ANALYZE ketiga query, analisis perbandingan sebelum dan sesudah indexing |

---

## Pekerjaan 4 — Trigger

### Trigger: `trg_update_po_status_on_receipt`

**Deskripsi:**
Trigger ini berjalan otomatis setiap kali ada insert pada tabel `gr_detail` (detail penerimaan barang). Fungsinya adalah memperbarui kolom `qty_received` pada `po_detail` sesuai kuantitas yang baru diterima, lalu memeriksa apakah seluruh item dalam PO terkait sudah diterima penuh.

Jika semua item sudah terpenuhi, status PO otomatis diubah menjadi `COMPLETED`. Jika baru sebagian, statusnya diubah menjadi `PARTIALLY_RECEIVED`. Trigger ini memastikan konsistensi status PO tanpa perlu update manual dari sisi aplikasi.

**Tabel yang terlibat:** `gr_detail`, `po_detail`, `purchase_order`

**Event:** `AFTER INSERT ON gr_detail FOR EACH ROW`

**Pembagian Tugas:**

| Anggota | Tugas |
|---------|-------|
| Anggota 1 | Review logika trigger dan validasi edge case (penerimaan berlebih, PO sudah cancelled) |
| Anggota 2 | Pengujian trigger dengan skenario penerimaan parsial dan penerimaan penuh |
| Anggota 3 | Dokumentasi deskripsi trigger dan screenshot hasil sebelum/sesudah eksekusi |
| Anggota 4 | Review integrasi trigger dengan transaksi Goods Receipt (Pekerjaan 7) |
| Anggota 5 | Membuat syntax trigger dan fungsi trigger, memastikan kompatibilitas dengan DDL |

---

## Pekerjaan 5 — Function

### Function: `fn_get_stock_summary`

**Deskripsi:**
Function ini menerima `product_id` sebagai parameter dan mengembalikan ringkasan stok produk tersebut secara agregat dari semua lokasi gudang. Informasi yang dikembalikan meliputi: total on-hand, total reserved, total available, status reorder (apakah stok sudah di bawah reorder point), dan rekomendasi jumlah yang perlu dipesan ke supplier.

Function ini dapat dipanggil langsung dari query SQL maupun dari lapisan aplikasi untuk menampilkan informasi stok secara real-time tanpa perlu menulis query agregat berulang.

**Tipe return:** composite type `stock_summary_type` berisi SKU, nama produk, qty on-hand, qty reserved, qty available, reorder point, flag need_reorder, dan suggested_order qty.

**Tabel yang terlibat:** `product`, `inventory`

**Pembagian Tugas:**

| Anggota | Tugas |
|---------|-------|
| Anggota 1 | Review logika kalkulasi suggested_order dan validasi output |
| Anggota 2 | Pengujian function dengan berbagai skenario: stok normal, stok kritis, produk tidak ada |
| Anggota 3 | Dokumentasi deskripsi function dan screenshot hasil pemanggilan |
| Anggota 4 | Pengujian integrasi: memanggil function dalam konteks Query 2 (reorder report) |
| Anggota 5 | Membuat syntax function dan definisi composite type, memastikan function bersifat STABLE |

---

## Pekerjaan 6 — Procedure

### Procedure: `sp_confirm_sales_order`

**Deskripsi:**
Procedure ini menangani proses konfirmasi Sales Order secara menyeluruh dalam satu transaksi atomik. Langkah-langkah yang dijalankan oleh procedure ini adalah:

1. Validasi bahwa SO yang diminta berstatus `PENDING`.
2. Untuk setiap item dalam SO, memeriksa apakah stok tersedia mencukupi kuantitas yang dipesan.
3. Jika semua item tersedia, menaikkan `qty_reserved` di tabel `inventory` untuk setiap item lalu mengubah status SO menjadi `CONFIRMED`.
4. Jika salah satu item stok tidak mencukupi, me-rollback seluruh alokasi dan mengembalikan pesan error melalui parameter output.

Procedure menerima `so_id` sebagai input dan mengembalikan dua parameter output: `p_success` (boolean) dan `p_message` (text) yang dapat dibaca oleh aplikasi pemanggil.

**Tabel yang terlibat:** `sales_order`, `so_detail`, `inventory`

**Parameter:** `IN p_so_id INT`, `OUT p_success BOOLEAN`, `OUT p_message TEXT`

**Pembagian Tugas:**

| Anggota | Tugas |
|---------|-------|
| Anggota 1 | Review logika alokasi stok dan pengecekan concurrent access |
| Anggota 2 | Pengujian skenario: stok cukup, stok kurang di salah satu item, SO sudah berstatus CONFIRMED |
| Anggota 3 | Dokumentasi deskripsi procedure dan screenshot hasil eksekusi sukses & gagal |
| Anggota 4 | Pengujian integrasi procedure dengan transaksi Ship Sales Order (T5) |
| Anggota 5 | Membuat syntax procedure lengkap termasuk logika rollback dan output parameter |

---

## Pekerjaan 7 — Database Transaction

### Transaksi: Goods Receipt (T2) — Penerimaan Barang dari Supplier

**Deskripsi:**
Transaksi ini adalah skenario paling kritis dalam modul supply chain: ketika barang dari supplier tiba di gudang. Seluruh operasi berikut harus berhasil sepenuhnya atau gagal semuanya (prinsip ACID):

1. Insert record baru ke tabel `goods_receipt` sebagai dokumen penerimaan.
2. Insert setiap baris item ke tabel `gr_detail`.
3. Trigger `trg_update_po_status_on_receipt` berjalan otomatis untuk memperbarui `qty_received` di `po_detail` dan status `purchase_order`.
4. Update `qty_on_hand` di tabel `inventory` untuk setiap produk yang diterima.
5. Insert record ke tabel `stock_movement` bertipe `IN` sebagai audit trail pergerakan stok.

Jika salah satu langkah gagal (misalnya referensi PO tidak valid, atau stok melebihi kapasitas lokasi), seluruh transaksi di-rollback menggunakan `SAVEPOINT` sehingga tidak ada perubahan parsial yang tersimpan di database.

**Tabel yang terlibat:** `goods_receipt`, `gr_detail`, `po_detail`, `purchase_order`, `inventory`, `stock_movement`

**Mekanisme:** `BEGIN` → `SAVEPOINT` → operasi DML → `COMMIT` atau `ROLLBACK TO SAVEPOINT`

**Pembagian Tugas:**

| Anggota | Tugas |
|---------|-------|
| Anggota 1 | Review alur transaksi dan validasi konsistensi dengan business rule Modul A |
| Anggota 2 | Pengujian skenario rollback: PO tidak valid, qty diterima melebihi qty dipesan |
| Anggota 3 | Dokumentasi hasil verifikasi data sebelum dan sesudah transaksi (screenshot) |
| Anggota 4 | Membuat syntax transaksi lengkap dengan SAVEPOINT, verifikasi pasca-commit |
| Anggota 5 | Review integrasi transaksi dengan trigger (Pekerjaan 4) dan memastikan urutan eksekusi benar |

---

*Dokumen ini dibuat sebagai Final Project Mata Kuliah Manajemen Basis Data*
*Departemen Teknik Informatika — ITS*
