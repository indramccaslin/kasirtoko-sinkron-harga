# KasirToko — Sinkron Harga Berbasis Nama Produk

Catatan implementasi untuk KasirToko Portable (`com.gentatekno.app.portable.kasirtoko`).

Aturan aktif: **dua produk dengan `product_name` yang persis sama harus selalu bergerak bersama**. Edit harga dari mana pun — termasuk dari layar pembelian — semua varian dengan nama itu ikut berubah memakai rasio yang sama, sehingga perbandingan antar satuan tetap terjaga.

---

## Prompt (siap copy)

```text
Di KasirToko Portable (package com.gentatekno.app.portable.kasirtoko), database ada di /storage/emulated/0/KasirToko/database/ (product.db dan price.db).

Pasang sinkronisasi harga berbasis NAMA PRODUK:
- Saat harga produk diedit, semua produk lain yang product_name-nya PERSIS SAMA harus ikut berubah.
- Berlaku saat edit di Data Produk (Harga Dasar / Harga Jual) maupun di pembelian → pilih produk → Harga Jual / Unit.
- Perubahan memakai rasio, bukan menyamakan harga, supaya perbandingan antar satuan tetap masuk akal.
  Contoh: 1 Bal = 50 Kg. Kalau Bal 730000 diubah jadi 800000, maka Kg 14600 otomatis jadi 16000.
- Harga Beli / Unit di pembelian tetap hanya memengaruhi baris transaksi/keranjang.
- price.db / Level Harga (Grosir dan Eceran) ikut tersinkron saat edit lewat Data Produk.

Aturan:
- Jangan pernah tekan tombol Reset Data.
- product_uxid dan price_uxid jangan diubah.
- product_type tetap BARANG.
- Eceran = Grosir + 10%, kecuali konvensi lama yang sudah ada.
- Backup dulu sebelum mengubah database.
- Matikan aplikasi saat edit database, lalu hidupkan lagi.
- Set PRAGMA user_version=1.

Yang harus dilakukan:
1. Backup product.db dan price.db ke folder tersendiri.
2. Hentikan aplikasi.
3. Hapus trigger lama berbasis mapping (trg_harga_from_induk_product dan trg_harga_from_induk_price) supaya tidak ada update ganda.
4. Di product.db pasang trigger AFTER UPDATE OF product_base_price, product_sale_price:
   untuk setiap baris lain dengan product_name = NEW.product_name, kalikan harga lama dengan
   rasio NEW/OLD (abaikan kalau harga lama 0), lalu bulatkan ke 2 desimal.
5. Di price.db pasang trigger AFTER UPDATE OF price_product_base_price, price_product_sale_price:
   lakukan hal yang sama untuk semua baris dengan price_product_name = NEW.price_product_name,
   dan hitung ulang price_value (Grosir memakai Harga Dasar, Eceran memakai Harga Jual).
   Jangan lupa sinkronkan price_value pada baris yang sedang diedit.
6. Hidupkan lagi aplikasi.
7. Validasi: price.db vs product.db tidak ada selisih, tidak ada kode duplikat,
   semua produk punya baris Level Harga, user_version=1, aplikasi berjalan normal.
8. Laporkan path backup, nama trigger yang dipasang, jumlah nama produk yang punya lebih dari
   satu varian, dan hasil validasi.
```

---

## Kenapa berbasis nama

Pendekatan sebelumnya memakai tabel `sync_rcg` / `sync_eceran` (konversi `Dus → Rcg → Pcs`, `Bal → Kg`, dll.). Itu bekerja, tapi hanya untuk produk yang terdaftar — masih ada produk dengan varian nama sama yang belum terpetakan (misalnya `AOKARG`, `BS2000`).

Dengan aturan berbasis nama, semua varian otomatis tercakup tanpa perlu audit satu per satu. Trigger lama dihapus agar tidak ada dua jalur update yang saling menimpa.

Mapping konversi (`sync_rcg`, `sync_eceran`, `sync_harga`) sengaja **dibiarkan** di database karena masih dipakai trigger sinkronisasi stok.

---

## Trigger yang dipasang

| Database | Trigger | Fungsi |
| --- | --- | --- |
| `product.db` | `trg_harga_nama_product` | Menyebarkan perubahan Harga Dasar / Harga Jual ke semua produk dengan nama sama |
| `price.db` | `trg_harga_nama_price` | Menyebarkan perubahan ke Level Harga dan menghitung ulang `price_value` |

Karena `updateSalePrice()` di aplikasi menulis ke `product.db`, edited harga dari layar pembelian ikut memicu `trg_harga_nama_product`.

---

## Contoh perilaku

| Aksi | Hasil |
| --- | --- |
| `BERAS12500` (Bal) 730.000 → 800.000 | `BERAS12500-KG` → 16.000, `BERAS12500-LTR` → 12.307,69 |
| `BERAS12500-KG` Eceran → 20.000 (edit dari pembelian) | Bal → 1.000.000, Ltr → 15.384,61 |
| `AOKARG` (dulu belum terpetakan) Eceran → 15.000 | `AOKARG-PAK` → 1.285,71, `AOKARG-PCS` → 142,86 |

---

## Batasan

- `Harga Beli / Unit` di pembelian tidak mengubah harga master produk — hanya baris keranjang. Ini desain aplikasinya.
- Edit dari pembelian hanya menyentuh `product.db`. Baris Level Harga di `price.db` baru ikut berubah saat ada edit lewat **Data Produk**, karena aplikasi tidak menulis `price.db` dari layar pembelian.

---

## Validasi terakhir

| Pemeriksaan | Hasil |
| --- | ---: |
| Total produk | 766 |
| Nama produk dengan lebih dari satu varian | 256 |
| Produk tanpa Level Harga | 0 |
| Kode duplikat | 0 |
| Selisih `price.db` vs `product.db` | 0 |
| Selisih `price_value` Grosir / Eceran | 0 |
| `PRAGMA user_version` | 1 (kedua DB) |
