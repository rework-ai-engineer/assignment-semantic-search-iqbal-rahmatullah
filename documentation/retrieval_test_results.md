# Hasil Pengujian Retrieval & Response AI

Dokumen ini berisi detail hasil uji dari notebook section 8 (`notebook/starter_notebook_semantic_search.ipynb`). Ringkasan dan analisisnya ada di [README](../README.md#7-hasil-pengujian--analisis) dan di cell "Catatan Analisis Relevansi" di notebook.

**Setup pengujian:**

| Komponen | Nilai |
|---|---|
| Embedding | `gemini-embedding-001` (3072 dimensi) |
| Index | FAISS `IndexFlatL2` |
| Retrieval | `top_k=3` |
| LLM | `gemini-3.1-flash-lite`, `temperature=0.3` |
| Tanggal uji | 21 Oktober 2026 |

**Cara membaca `distance`:**
- `distance` adalah jarak L2 kuadrat. **Makin kecil, makin mirip.**
- Karena vektornya sudah ternormalisasi, nilai ini bisa dikonversi menjadi cosine similarity dengan rumus `cosine = 1 − distance/2`.
- FAQ yang diharapkan ditentukan **sebelum** sistem dijalankan.

## Ringkasan

| # | Tipe | Query | Harapan | Top-3 (distance) | Rank harapan | Retrieval | Jawaban LLM |
|---|---|---|---|---|---|---|---|
| 1 | Jelas | Metode pembayaran apa saja yang tersedia? | #17 | #17 (0.314), #22 (0.507), #1 (0.589) | 1 | ✅ Tepat, pemenang jelas | ✅ Lengkap dan sesuai #17 |
| 2 | Jelas | Bagaimana cara melacak paket yang sedang dikirim? | #10 | #10 (0.260), #52 (0.431), #12 (0.476) | 1 | ✅ Tepat, pemenang jelas | ✅ Sesuai #10 |
| 3 | Parafrase | Saldo saya sudah terpotong tapi pesanan masih belum lunas | #21 | #21 (0.319), #8 (0.479), #44 (0.516) | 1 | ✅ Tepat ("terpotong/lunas" → "dipotong/terbayar") | ✅ Sesuai #21, nuansa "biasanya" dan angka 15–30 menit dipertahankan |
| 4 | Parafrase | Barang yang datang pecah, harus gimana? | #16 | #16 (0.336), #54 (0.479), #24 (0.504) | 1 | ✅ Tepat ("pecah" → "rusak") | ✅ Sesuai #16 (foto/video, 2x24 jam) |
| 5 | Parafrase | Duit refund saya balik ke mana ya? | #27 | #27 (0.238), #25 (0.410), #24 (0.579) | 1 | ✅ Tepat, distance terkecil di seluruh uji | ✅ Sesuai #27 |
| 6 | Parafrase | bisa tuker size? | #30 | #46 (0.681), #30 (0.696), #58 (0.800) | 2 | ⚠️ Kurang tepat: top-1 #46 (size chart), #30 di rank 2, selisih hanya 0.015 | ✅ Tetap dijawab dari #30, tertolong top_k=3 |
| 7 | Sebagian | Berapa nomor telepon customer service? | #48 | #48 (0.272), #49 (0.462), #59 (0.496) | 1 | ✅ Tepat | ⚠️ Tidak mengarang nomor, tetapi kontradiktif: jawaban parsial + kalimat fallback |
| 8 | Tidak ada | Bisa bayar COD? | – | #17 (0.612), #22 (0.616), #37 (0.617) | – | – Wajar: tidak ada FAQ COD, top-3 datar (FAQ satu topik) | ✅ Fallback. Aman tapi kurang membantu (tidak menyebut metode yang tersedia) |
| 9 | Tidak ada | Bisa kirim ke luar negeri? | – | #9 (0.585), #15 (0.614), #13 (0.615) | – | – Wajar: tidak ada FAQ terkait, top-3 datar | ✅ Fallback, tidak terjebak "luar pulau" di #9 |

**Skor retrieval:** dari 7 query yang punya jawaban, FAQ yang benar berada di top-1 pada **6 query** dan di top-3 pada **ketujuhnya**.

## Detail per Query

### 1. Metode pembayaran apa saja yang tersedia?

- **Tipe:** Jelas
- **FAQ yang diharapkan:** #17

| Rank | faq_id | Kategori | Pertanyaan FAQ | Distance |
|---|---|---|---|---|
| 1 | #17 | Pembayaran | Metode pembayaran apa saja yang tersedia? | 0.314 |
| 2 | #22 | Pembayaran | Apakah bisa membayar menggunakan lebih dari satu metode sekaligus? | 0.507 |
| 3 | #1 | Pemesanan | Bagaimana cara melakukan pemesanan di platform ini? | 0.589 |

**Jawaban LLM:**

> Halo! Saat ini kami mendukung metode pembayaran melalui transfer bank, kartu kredit/debit, e-wallet, dan pembayaran melalui minimarket rekanan. Semoga membantu!

- **Penilaian retrieval:** ✅ Tepat, pemenang jelas
- **Penilaian jawaban LLM:** ✅ Lengkap dan sesuai #17

### 2. Bagaimana cara melacak paket yang sedang dikirim?

- **Tipe:** Jelas
- **FAQ yang diharapkan:** #10

| Rank | faq_id | Kategori | Pertanyaan FAQ | Distance |
|---|---|---|---|---|
| 1 | #10 | Pengiriman | Bagaimana cara melacak paket yang sedang dikirim? | 0.260 |
| 2 | #52 | Pengiriman | Kenapa nomor resi saya belum bisa dilacak di website kurir? | 0.431 |
| 3 | #12 | Pengiriman | Bagaimana jika paket saya belum sampai melebihi estimasi? | 0.476 |

**Jawaban LLM:**

> Anda dapat melacak paket dengan menggunakan nomor resi yang muncul di halaman 'Pesanan Saya' setelah barang dikirim, lalu gunakan nomor tersebut untuk melacak posisi paket di website kurir terkait.

- **Penilaian retrieval:** ✅ Tepat, pemenang jelas
- **Penilaian jawaban LLM:** ✅ Sesuai #10

### 3. Saldo saya sudah terpotong tapi pesanan masih belum lunas

- **Tipe:** Parafrase
- **FAQ yang diharapkan:** #21

| Rank | faq_id | Kategori | Pertanyaan FAQ | Distance |
|---|---|---|---|---|
| 1 | #21 | Pembayaran | Kenapa pembayaran saya berhasil dipotong tapi status pesanan masih belum terbayar? | 0.319 |
| 2 | #8 | Pemesanan | Kenapa pesanan saya otomatis dibatalkan sistem? | 0.479 |
| 3 | #44 | Produk | Bagaimana jika produk yang saya pesan ternyata kehabisan stok? | 0.516 |

**Jawaban LLM:**

> Halo! Hal tersebut biasanya terjadi karena adanya keterlambatan sinkronisasi pada sistem pembayaran. Silakan tunggu selama 15-30 menit, dan jika status pesanan Anda belum juga berubah, silakan hubungi customer service kami untuk bantuan lebih lanjut.

- **Penilaian retrieval:** ✅ Tepat ("terpotong/lunas" → "dipotong/terbayar")
- **Penilaian jawaban LLM:** ✅ Sesuai #21, nuansa "biasanya" dan angka 15–30 menit dipertahankan

### 4. Barang yang datang pecah, harus gimana?

- **Tipe:** Parafrase
- **FAQ yang diharapkan:** #16

| Rank | faq_id | Kategori | Pertanyaan FAQ | Distance |
|---|---|---|---|---|
| 1 | #16 | Pengiriman | Apa yang harus dilakukan jika paket diterima dalam kondisi rusak? | 0.336 |
| 2 | #54 | Pengembalian | Saya menerima barang yang salah, apa yang harus saya lakukan? | 0.479 |
| 3 | #24 | Pengembalian | Bagaimana prosedur pengembalian barang yang tidak sesuai? | 0.504 |

**Jawaban LLM:**

> Segera ambil foto atau video kondisi paket saat diterima dan laporkan melalui menu komplain dalam waktu 2x24 jam sejak barang diterima.

- **Penilaian retrieval:** ✅ Tepat ("pecah" → "rusak")
- **Penilaian jawaban LLM:** ✅ Sesuai #16 (foto/video, 2x24 jam)

### 5. Duit refund saya balik ke mana ya?

- **Tipe:** Parafrase
- **FAQ yang diharapkan:** #27

| Rank | faq_id | Kategori | Pertanyaan FAQ | Distance |
|---|---|---|---|---|
| 1 | #27 | Pengembalian | Ke mana dana refund akan dikembalikan? | 0.238 |
| 2 | #25 | Pengembalian | Berapa lama proses refund setelah pengembalian disetujui? | 0.410 |
| 3 | #24 | Pengembalian | Bagaimana prosedur pengembalian barang yang tidak sesuai? | 0.579 |

**Jawaban LLM:**

> Dana refund akan dikembalikan ke metode pembayaran awal, atau ke saldo akun apabila Anda memilih opsi tersebut saat pengajuan.

- **Penilaian retrieval:** ✅ Tepat, distance terkecil di seluruh uji
- **Penilaian jawaban LLM:** ✅ Sesuai #27

### 6. bisa tuker size?

- **Tipe:** Parafrase
- **FAQ yang diharapkan:** #30

| Rank | faq_id | Kategori | Pertanyaan FAQ | Distance |
|---|---|---|---|---|
| 1 | #46 | Produk | Di mana saya bisa melihat ukuran/size chart produk pakaian? | 0.681 |
| 2 | #30 | Pengembalian | Apakah bisa menukar barang dengan ukuran/warna lain daripada refund? | 0.696 |
| 3 | #58 | Pemesanan | Apakah bisa memesan dalam jumlah besar untuk kebutuhan reseller? | 0.800 |

**Jawaban LLM:**

> Penukaran ukuran tergantung pada kebijakan masing-masing penjual, jadi silakan cek ketersediaan stok terlebih dahulu sebelum mengajukan penukaran.

- **Penilaian retrieval:** ⚠️ Kurang tepat: top-1 #46 (size chart), #30 di rank 2, selisih hanya 0.015
- **Penilaian jawaban LLM:** ✅ Tetap dijawab dari #30, tertolong top_k=3

### 7. Berapa nomor telepon customer service?

- **Tipe:** Sebagian
- **FAQ yang diharapkan:** #48

| Rank | faq_id | Kategori | Pertanyaan FAQ | Distance |
|---|---|---|---|---|
| 1 | #48 | Customer Service | Bagaimana cara menghubungi customer service? | 0.272 |
| 2 | #49 | Customer Service | Berapa lama waktu respon customer service terhadap keluhan? | 0.462 |
| 3 | #59 | Customer Service | Bagaimana cara eskalasi keluhan jika belum ditangani customer service? | 0.496 |

**Jawaban LLM:**

> Nomor telepon customer service tertera di halaman bantuan pada aplikasi kami.
>
> Maaf, saya belum memiliki informasi spesifik mengenai nomor tersebut. Silakan hubungi customer service kami melalui fitur live chat atau email resmi untuk bantuan lebih lanjut.

- **Penilaian retrieval:** ✅ Tepat
- **Penilaian jawaban LLM:** ⚠️ Tidak mengarang nomor, tetapi kontradiktif: jawaban parsial + kalimat fallback

### 8. Bisa bayar COD?

- **Tipe:** Tidak ada
- **FAQ yang diharapkan:** tidak ada (harus fallback)

| Rank | faq_id | Kategori | Pertanyaan FAQ | Distance |
|---|---|---|---|---|
| 1 | #17 | Pembayaran | Metode pembayaran apa saja yang tersedia? | 0.612 |
| 2 | #22 | Pembayaran | Apakah bisa membayar menggunakan lebih dari satu metode sekaligus? | 0.616 |
| 3 | #37 | Promo | Bagaimana cara menggunakan kode voucher saat checkout? | 0.617 |

**Jawaban LLM:**

> Maaf, saya belum memiliki informasi tersebut. Silakan hubungi customer service kami untuk bantuan lebih lanjut.

- **Penilaian retrieval:** – Wajar: tidak ada FAQ COD, top-3 datar (FAQ satu topik)
- **Penilaian jawaban LLM:** ✅ Fallback. Aman tapi kurang membantu (tidak menyebut metode yang tersedia)

### 9. Bisa kirim ke luar negeri?

- **Tipe:** Tidak ada
- **FAQ yang diharapkan:** tidak ada (harus fallback)

| Rank | faq_id | Kategori | Pertanyaan FAQ | Distance |
|---|---|---|---|---|
| 1 | #9 | Pengiriman | Berapa lama estimasi waktu pengiriman barang? | 0.585 |
| 2 | #15 | Pengiriman | Apakah bisa memilih jasa pengiriman sendiri saat checkout? | 0.614 |
| 3 | #13 | Pengiriman | Apakah ongkos kirim sudah termasuk asuransi pengiriman? | 0.615 |

**Jawaban LLM:**

> Maaf, saya belum memiliki informasi tersebut. Silakan hubungi customer service kami untuk bantuan lebih lanjut.

- **Penilaian retrieval:** – Wajar: tidak ada FAQ terkait, top-3 datar
- **Penilaian jawaban LLM:** ✅ Fallback, tidak terjebak "luar pulau" di #9

_Teks jawaban LLM bisa sedikit berbeda di setiap run karena `temperature=0.3`. Hasil retrieval (top-3 dan distance) deterministik._
