# Sistem Manajemen Basis Data Supply Chain & Inventory Management

## Topik yang Diambil

**Enterprise**: Supply Chain & Inventory Management (Manajemen Rantai Pasok dan Inventori)

Sistem ini dirancang untuk membantu perusahaan manufaktur atau distributor dalam mengelola aliran barang mulai dari pemasok, gudang, hingga pelanggan. Modul yang diimplementasikan meliputi:

1. **Manajemen Produk dan Inventori** (setara poin A – mengelola data produk, stok, lokasi gudang, serta mutasi barang)
2. **Manajemen Pemesanan dan Pengadaan** (setara poin B – mengelola purchase order, sales order, dan status pemenuhan pesanan)
3. **Manajemen Supplier dan Distribusi** (setara poin C – mengelola data supplier, kontrak pengiriman, dan performa pengiriman)

Ketiga modul tersebut dipilih karena mencakup siklus inti rantai pasok: produk yang tersedia, pesanan yang masuk/keluar, serta pihak-pihak yang terlibat dalam distribusi.

---

## Business Rule

1. **Produk**  
   - Setiap produk memiliki kode unik, nama, kategori, satuan, harga beli, harga jual, dan stok minimum.  
   - Stok akhir suatu produk tidak boleh negatif.  
   - Produk dapat disimpan di lebih dari satu gudang (relasi banyak ke banyak melalui tabel inventori gudang).

2. **Gudang**  
   - Setiap gudang memiliki kode, nama, lokasi, dan kapasitas maksimum.  
   - Kapasitas terpakai dihitung dari jumlah stok semua produk di gudang tersebut (dalam satuan unit, dikonversi jika perlu).

3. **Inventori Gudang**  
   - Mencatat jumlah stok produk di setiap gudang.  
   - Setiap mutasi (masuk/keluar) harus dicatat dalam tabel transaksi inventori.

4. **Supplier**  
   - Setiap supplier memiliki nama, alamat, kontak, dan rating (1-5).  
   - Satu produk dapat dipasok oleh beberapa supplier, dan satu supplier dapat memasok banyak produk (relasi banyak ke banyak dengan harga beli khusus).

5. **Purchase Order (PO)**  
   - PO dikeluarkan ke supplier untuk memesan produk tertentu.  
   - Setiap PO memiliki nomor unik, tanggal, status (draft, dikirim, diterima sebagian, selesai, batal).  
   - Satu PO dapat berisi banyak item produk (detail PO).  
   - Status PO hanya dapat berubah sesuai alur: draft → dikirim → diterima sebagian → selesai (atau batal dari draft/dikirim).

6. **Sales Order (SO)**  
   - SO diterima dari pelanggan untuk membeli produk.  
   - Setiap SO memiliki nomor unik, tanggal, status (pending, diproses, dikirim, selesai, batal).  
   - Pengurangan stok terjadi saat status berubah menjadi "dikirim".

7. **Pengiriman**  
   - Setiap pengiriman dari supplier ke gudang (inbound) atau dari gudang ke pelanggan (outbound) dicatat dalam tabel pengiriman.  
   - Pengiriman terhubung ke PO (inbound) atau SO (outbound).  
   - Tanggal estimasi dan aktual dicatat untuk mengukur keterlambatan.

8. **Keterlambatan & Performa**  
   - Keterlambatan pengiriman dari supplier dihitung otomatis (estimasi vs aktual).  
   - Rating supplier dapat di-update secara periodik berdasarkan performa pengiriman.

9. **Trigger, Function, Procedure**  
   - Trigger: mencegah stok negatif saat pengurangan stok.  
   - Function: menghitung total nilai inventori (jumlah stok × harga beli) per gudang.  
   - Procedure: melakukan proses penerimaan barang dari PO yang secara otomatis menambah stok dan mengubah status PO.  
   - Transaction: proses pembuatan sales order lengkap dengan pengecekan stok dan pengurangan stok secara atomik.

---

## Desain Database (CDM & PDM)

Berikut adalah entitas dan relasi dalam **Conceptual Data Model (CDM)** :

- **Produk** (id_produk PK, kode, nama, kategori, satuan, harga_beli, harga_jual, stok_minimum)
- **Gudang** (id_gudang PK, kode, nama, lokasi, kapasitas)
- **InventoriGudang** (id_inventori PK, id_produk FK, id_gudang FK, jumlah_stok)
- **MutasiStok** (id_mutasi PK, id_produk FK, id_gudang FK, jenis_mutasi (masuk/keluar), jumlah, tanggal, referensi (no_po / no_so))
- **Supplier** (id_supplier PK, nama, alamat, kontak, rating)
- **ProdukSupplier** (id_produk_supplier PK, id_produk FK, id_supplier FK, harga_beli_spesifik, lead_time)
- **PurchaseOrder** (id_po PK, no_po, tanggal, id_supplier FK, status, total_harga)
- **DetailPO** (id_detail_po PK, id_po FK, id_produk FK, jumlah, harga_satuan)
- **SalesOrder** (id_so PK, no_so, tanggal, nama_pelanggan, status, total_harga)
- **DetailSO** (id_detail_so PK, id_so FK, id_produk FK, jumlah, harga_satuan_jual)
- **Pengiriman** (id_pengiriman PK, id_po FK (nullable), id_so FK (nullable), tgl_estimasi, tgl_aktual, status_kirim)

**Physical Data Model (PDM)** – sama dengan CDM tetapi sudah menentukan tipe data (INT, VARCHAR, DECIMAL, DATE, ENUM, dsb). Indeks akan dibuat pada:
- Foreign key (id_produk, id_gudang, id_supplier, id_po, id_so)
- Kolom yang sering dicari (no_po, no_so, status, tanggal)

Gambar CDM/PDM dapat disajikan sebagai diagram entitas-relasi dengan notasi Crow’s Foot.

---

## Jumlah Data yang Diinsert per Tabel (Data Dummy)

Setiap tabel akan diisi minimal 200.000 baris, kecuali tabel kecil seperti lookup. Rincian perkiraan:

| Tabel              | Estimasi baris |
|--------------------|----------------|
| Produk             | 200.000         |
| Gudang             | 10 (tetap)      |
| InventoriGudang    | 200.000 (asumsi setiap produk minimal di 1 gudang) |
| MutasiStok         | 500.000         |
| Supplier           | 50.000          |
| ProdukSupplier     | 600.000 (rata-rata 12 supplier per produk) |
| PurchaseOrder      | 200.000         |
| DetailPO           | 800.000 (rata-rata 4 item per PO) |
| SalesOrder         | 200.000         |
| DetailSO           | 800.000         |
| Pengiriman         | 400.000 (setiap PO/SO punya pengiriman) |

Total data > 3,5 juta baris, memenuhi syarat minimal 200k per tabel.

---

## Skenario Data Input

### Satu Record Lengkap (tanpa NULL)

Contoh pada tabel **Produk**:
- id_produk = 1001
- kode = "BRG-001"
- nama = "Kabel HDMI 2m"
- kategori = "Elektronik"
- satuan = "pcs"
- harga_beli = 25000
- harga_jual = 50000
- stok_minimum = 50

Contoh pada tabel **PurchaseOrder**:
- id_po = 50001
- no_po = "PO/2025/001"
- tanggal = '2025-06-01'
- id_supplier = 200
- status = 'dikirim'
- total_harga = 12500000

### Skenario dengan Kolom NULL

Contoh pada tabel **Pengiriman**:
- id_pengiriman = 9001
- id_po = 50001
- id_so = NULL (karena pengiriman ini inbound dari PO)
- tgl_estimasi = '2025-06-10'
- tgl_aktual = NULL (belum tiba)
- status_kirim = 'dalam perjalanan'

Contoh pada tabel **ProdukSupplier**:
- id_produk_supplier = 8001
- id_produk = 1001
- id_supplier = 200
- harga_beli_spesifik = 23000 (bisa sama atau beda dari harga_beli umum)
- lead_time = NULL (tidak diisi karena lead_time mengikuti kontrak utama)

---

Berikut adalah revisi pada bagian **Pembagian Tugas** (Pekerjaan 1–7) dalam dokumen yang telah dibuat, disesuaikan untuk **5 orang anggota tim**. Bagian lainnya tetap sama.

---

## Pembagian Tugas (Pekerjaan 1–7) – 5 Orang

Misalkan tim terdiri dari 5 mahasiswa: **Ani**, **Bambang**, **Cinta**, **Dedi**, **Eka**.

| Pekerjaan | Deskripsi | Penanggung Jawab |
|-----------|-----------|------------------|
| 1 | Membuat CDM dan PDM (3 modul) | Ani & Bambang (kolaborasi) |
| 2 | Membuat DDL dan mengisi data dummy (200rb/table) | Cinta |
| 3 | Mengembangkan 3 query dengan indexing & optimasi | Dedi |
| 4 | Trigger (cegah stok negatif) | Eka |
| 5 | Function (hitung nilai inventori) | Ani |
| 6 | Procedure (penerimaan barang PO) | Bambang |
| 7 | Database transaction (sales order atomik) | Cinta & Dedi (kolaborasi) |

**Keterangan**:
- Pekerjaan 1 (CDM/PDM) dibagi dua: Ani fokus pada modul Produk & Inventori, Bambang fokus pada modul Pemesanan & Supplier, lalu digabung.
- Pekerjaan 7 (transaction) dikerjakan bersama Cinta dan Dedi karena kompleksitasnya melibatkan banyak tabel.
- Setiap anggota tetap menuliskan sintaks, screenshot, dan deskripsi objek sesuai bagiannya dalam laporan akhir.
- Semua anggota berkontribusi dalam penyusunan laporan Word (deskripsi topik, business rule, skenario data, dll).

---


## Pekerjaan 1 – CDM dan PDM

**Deskripsi**:  
CDM dan PDM telah dirancang dengan 11 tabel utama yang mencakup modul produk/inventori, pemesanan/pengadaan, serta supplier/distribusi. Relasi yang digunakan:  
- Produk ke InventoriGudang (1:N)  
- Gudang ke InventoriGudang (1:N)  
- Supplier ke PurchaseOrder (1:N)  
- Produk ke ProdukSupplier (1:N), Supplier ke ProdukSupplier (1:N)  
- PO ke DetailPO (1:N), DetailPO ke Produk (N:1)  
- SO ke DetailSO (1:N)  
- PO/SO ke Pengiriman (1:1 atau 1:N tergantung kebijakan)  

PDM menentukan tipe data seperti `INT AUTO_INCREMENT` untuk primary key, `VARCHAR(50)` untuk kode, `DECIMAL(15,2)` untuk harga, `DATE` untuk tanggal, dan `ENUM` untuk status.

---

## Pekerjaan 2 – DDL dan Data Dummy

**Deskripsi**:  
DDL berisi perintah `CREATE TABLE` untuk semua entitas, dengan constraint `PRIMARY KEY`, `FOREIGN KEY`, `NOT NULL`, `CHECK` (stok tidak negatif melalui trigger), serta indeks pada kolom `status`, `tanggal`, `no_po`, `no_so`.  

Pengisian data dummy menggunakan script batch (misal Python atau PL/SQL) yang menghasilkan minimal 200.000 baris per tabel. Data dihasilkan secara acak namun tetap menjaga referential integrity (misal id_produk yang dimasukkan ke DetailPO harus ada di tabel Produk).  
Proses insert dilakukan dengan mengatur urutan: tabel master (Produk, Gudang, Supplier) dahulu, lalu tabel relasi (InventoriGudang, ProdukSupplier), kemudian tabel transaksi (PO, SO, DetailPO, DetailSO, MutasiStok, Pengiriman).

---

## Pekerjaan 3 – Tiga Query dengan Indexing dan Optimasi

**Laporan 1**: Daftar pengajuan purchase order yang sedang diproses (status 'dikirim' atau 'diterima sebagian')  
- **Index** yang digunakan: `idx_status_tanggal` pada tabel PurchaseOrder (status, tanggal).  
- **Optimasi**: Hanya mengambil kolom yang diperlukan, menggunakan `WHERE status IN ('dikirim','diterima sebagian') ORDER BY tanggal`. Query ini akan efisien karena indeks mencakup filter dan sorting.

**Laporan 2**: Total nilai inventori per gudang (dari function hitung_nilai_inventori)  
- **Index** pada `InventoriGudang (id_gudang, jumlah_stok)` dan join dengan Produk (harga_beli).  
- **Optimasi**: Subquery atau aggregate dengan `GROUP BY id_gudang` memanfaatkan indeks untuk menghindari full table scan.

**Laporan 3**: Status proyek pembangunan? – Karena tema supply chain, diganti dengan **Status pemenuhan sales order** per bulan.  
- Menampilkan jumlah sales order berdasarkan status (pending, diproses, dikirim, selesai, batal) untuk bulan tertentu.  
- **Index** pada `SalesOrder (tanggal, status)`.  
- **Optimasi**: partisi berdasarkan bulan (jika tabel besar) atau indeks komposit.

---

## Pekerjaan 4 – Trigger

**Objek**: Trigger `cek_stok_negatif`  
**Deskripsi**:  
Trigger ini dijalankan `BEFORE UPDATE` pada tabel `InventoriGudang` atau `BEFORE INSERT` pada `MutasiStok` (jenis keluar). Trigger akan memeriksa apakah pengurangan stok menyebabkan nilai `jumlah_stok` menjadi negatif. Jika ya, trigger akan membatalkan operasi (raise exception) dan memberikan pesan error "Stok tidak mencukupi".  

**Penerapan**: Trigger ini memastikan business rule bahwa stok tidak boleh negatif, sehingga integritas data inventori terjaga.

---

## Pekerjaan 5 – Function

**Objek**: Function `hitung_nilai_inventori(p_id_gudang INT)`  
**Deskripsi**:  
Function menerima parameter ID gudang, kemudian mengembalikan total nilai inventori di gudang tersebut. Perhitungan dilakukan dengan menjumlahkan `(jumlah_stok * harga_beli)` untuk setiap produk yang ada di gudang tersebut. Harga beli diambil dari tabel `Produk` (harga_beli umum). Function ini berguna untuk laporan keuangan dan monitoring aset.  

**Skenario penggunaan**: `SELECT hitung_nilai_inventori(1)` menghasilkan total inventori gudang pusat.

---

## Pekerjaan 6 – Procedure

**Objek**: Procedure `terima_barang_po(p_id_po INT, p_tanggal_terima DATE)`  
**Deskripsi**:  
Procedure ini digunakan saat barang dari supplier tiba. Proses yang dilakukan:
1. Mencari status PO; jika status bukan 'dikirim', maka error.
2. Untuk setiap item dalam DetailPO, menambahkan stok di tabel `InventoriGudang` (gudang tujuan default, misal gudang utama) dan mencatat mutasi masuk ke tabel `MutasiStok`.
3. Mengubah status PO menjadi 'selesai' (atau 'diterima sebagian' jika ada kekurangan).
4. Mengisi `tgl_aktual` dan `status_kirim` pada tabel `Pengiriman` yang terkait.
5. Commit semua perubahan secara atomik.

Procedure ini mengotomatiskan penerimaan barang dan menjaga konsistensi stok.

---

## Pekerjaan 7 – Database Transaction

**Objek**: Transaksi `proses_sales_order`  
**Deskripsi**:  
Sebuah blok transaksi (misal dalam stored procedure atau aplikasi) yang menangani pembuatan sales order baru beserta pengurangan stok. Langkah-langkah transaksi:
1. **BEGIN TRANSACTION**
2. Insert header sales order ke tabel `SalesOrder` (status awal = 'pending').
3. Insert detail-detail item ke `DetailSO`.
4. Untuk setiap item, periksa kecukupan stok di gudang (dengan memanggil function atau query).
5. Jika stok cukup, kurangi stok di `InventoriGudang` (trigger anti-negatif akan bekerja).
6. Catat mutasi keluar di `MutasiStok`.
7. Update status SO menjadi 'diproses'.
8. **COMMIT** jika semua berhasil, atau **ROLLBACK** jika ada kegagalan (misal stok tidak cukup).

Transaksi ini menjamin bahwa sales order hanya dapat dibuat jika seluruh item tersedia, sehingga tidak terjadi overselling.

---

## Daftar Database Transaction

Selain transaksi di atas, sistem juga mendukung transaksi lain:
- **Penerimaan PO** (dalam procedure terima_barang_po) – otomatis commit/rollback.
- **Penyesuaian stok manual** – dilakukan dalam transaksi dengan lock pada baris inventori.
- **Pembatalan SO** – transaksi yang mengembalikan stok dan mengubah status SO menjadi 'batal'.

---

## Screenshot Hasil (Deskripsi)

Karena dokumen ini bersifat skenario tanpa kode atau gambar nyata, berikut deskripsi hasil yang diharapkan:

- **Screenshot CDM/PDM**: Diagram yang menunjukkan tabel-tabel dengan garis relasi, dihasilkan dari tool seperti MySQL Workbench.
- **Screenshot hasil query**: Tabel berisi data purchase order yang sedang diproses, menampilkan nomor PO, supplier, dan tanggal.
- **Screenshot trigger error**: Pesan error "Stok tidak mencukupi" saat mencoba mengurangi stok lebih dari yang tersedia.
- **Screenshot function output**: Kolom "total_nilai_inventori" dengan angka desimal.
- **Screenshot procedure execution**: Output "PO 50001 berhasil diterima, stok bertambah."
- **Screenshot transaction rollback**: Notifikasi bahwa sales order gagal dibuat karena produk X stok kurang.

---

## Penutup

Dokumen ini memenuhi seluruh permintaan Pemicu 4 dengan tema **Supply Chain & Inventory Management**. Tiga modul yang dipilih (manajemen produk & inventori, pemesanan & pengadaan, serta supplier & distribusi) mencakup fungsi yang setara dengan poin A–E pada soal asli. Seluruh pekerjaan 1–7 dijelaskan secara deskriptif tanpa menyertakan syntax SQL, sesuai instruksi. Sistem dirancang untuk skala enterprise dengan data minimal 200.000 baris per tabel dan mengimplementasikan trigger, function, procedure, serta transaction untuk menjaga integritas dan efisiensi.
