# Semantic Search & LLM Application: Asisten FAQ Customer Service E-commerce

Prototype asisten AI yang menjawab pertanyaan pelanggan e-commerce berdasarkan knowledge base FAQ. Cara kerjanya:

1. Sistem mencari entri FAQ yang **maknanya** paling dekat dengan pertanyaan pelanggan (_semantic search_ dengan Gemini Embeddings + FAISS).
2. Entri yang ditemukan disisipkan sebagai context ke prompt.
3. Gemini menyusun jawaban yang natural **hanya** dari context tersebut.

Prototype ini menjadi fondasi sebelum dikembangkan menjadi sistem RAG (Retrieval-Augmented Generation) yang lengkap.

## 1. Problem Statement

### Objective

- **Bisnis:** Tim customer service e-commerce menerima ratusan pertanyaan berulang setiap hari, seputar pemesanan, pengiriman, pembayaran, dan pengembalian barang. Jawaban untuk sebagian besar pertanyaan itu sebenarnya sudah ada di FAQ. Tim ingin asisten AI yang bisa menjawab pertanyaan seperti ini secara otomatis, supaya:
  - pelanggan mendapat jawaban lebih cepat;
  - jawaban tetap konsisten dengan kebijakan resmi di FAQ;
  - agen manusia bisa fokus ke kasus yang memang butuh penanganan khusus.
- **Teknis:** Membangun pipeline end-to-end `ingest → embed → store → retrieve → generate`. Pipeline ini harus:
  1. **menemukan entri FAQ yang relevan walaupun pelanggan memakai kata yang berbeda**, misalnya "lupa kata sandi" untuk FAQ yang berbunyi "lupa password", atau "nyicil" untuk "cicilan";
  2. **menyusun jawaban yang natural** berdasarkan FAQ yang ditemukan;
  3. **mengakui dengan jujur** bila jawabannya tidak ada di FAQ, alih-alih mengarang.

|                   |                                                                                                                                                                            |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Input             | Pertanyaan bebas dari pelanggan (Bahasa Indonesia, formal/informal)                                                                                                        |
| Output            | Jawaban natural dalam Bahasa Indonesia + daftar FAQ sumbernya                                                                                                              |
| Sumber kebenaran  | Knowledge base FAQ (bukan pengetahuan umum LLM)                                                                                                                            |
| Kriteria berhasil | FAQ yang benar ter-retrieve pada sebagian besar query uji, termasuk query berparafrase. Jawaban LLM setia pada context, dan pertanyaan di luar FAQ dijawab dengan fallback |

### Kenapa semantic search, bukan keyword search?

Pelanggan jarang memakai kata yang persis sama dengan FAQ. Pencarian kata kunci (`str.contains`, SQL `LIKE`) gagal mencocokkan "kata sandi" dengan "password", atau "uang sudah kepotong" dengan "pembayaran berhasil dipotong". Semantic search mengubah teks menjadi vektor (_embedding_), sehingga kalimat yang **maknanya** mirip punya vektor yang berdekatan, walaupun kata-katanya berbeda.

## 2. Knowledge Base

- **File:** `data/ecommerce_faq_knowledge_base.csv` (isi asli, tidak diubah)
- **Sumber:** [Google Drive dari brief assignment](https://drive.google.com/file/d/1A1m685JChJQ3gvUDh-IBWdj-F2O7_mL_/view?usp=sharing)
- **Ukuran:** 59 entri dan 4 kolom, tanpa missing value. `faq_id` unik dan tidak ada pertanyaan duplikat.

| Kolom      | Isi                         |
| ---------- | --------------------------- |
| `faq_id`   | ID unik entri (1–59)        |
| `category` | Kategori topik (8 kategori) |
| `question` | Pertanyaan FAQ              |
| `answer`   | Jawaban resmi FAQ           |

| Kategori         | Jumlah |
| ---------------- | -----: |
| Pemesanan        |      9 |
| Pengiriman       |      9 |
| Pembayaran       |      8 |
| Pengembalian     |      8 |
| Akun             |      7 |
| Produk           |      7 |
| Promo            |      6 |
| Customer Service |      5 |

**Karakteristik yang memengaruhi desain sistem:**

1. **Teks pendek.** Gabungan question + answer hanya 144–240 karakter, dengan rata-rata sekitar 25 kata per entri. Karena itu **satu entri FAQ = satu unit retrievable**, tanpa chunking.
2. **Ada entri yang topiknya berdekatan.** Contohnya:
   - #10 "cara melacak paket" dan #52 "resi belum bisa dilacak"
   - #12 "paket belum sampai melebihi estimasi" dan #14 "status pengiriman tidak berubah"
   - #24 "pengembalian barang tidak sesuai" dan #54 "menerima barang yang salah"
   - #3 "membatalkan pesanan" dan #8 "pesanan otomatis dibatalkan"

   Pasangan seperti ini berpotensi saling tertukar saat retrieval. Inilah salah satu alasan sistem mengambil **top-k** entri, bukan hanya satu.

3. **Jawabannya umum, bukan spesifik.** Banyak jawaban memakai kata "umumnya", "biasanya", atau "tergantung". FAQ juga tidak memuat detail konkret seperti nomor telepon CS, nama bank, daftar kota same-day, atau syarat cicilan per bank. LLM harus mempertahankan nuansa ini dan tidak boleh "melengkapi" detail yang tidak ada.
4. **Distribusi kategori tidak merata.** Jumlahnya berkisar antara 5 entri (Customer Service) dan 9 entri (Pemesanan, Pengiriman). Ini cukup untuk prototype, tetapi cakupan tiap topik masih tipis.

## 3. Batasan & Asumsi

1. **Closed-domain: hanya menjawab dari FAQ.** LLM diarahkan untuk tidak memakai pengetahuan umumnya. Pertanyaan yang jawabannya tidak ada di FAQ, misalnya COD atau pengiriman ke luar negeri, dijawab dengan pesan bahwa informasinya belum tersedia, lalu pelanggan diarahkan ke customer service.
2. **Tidak terhubung ke data transaksi.** FAQ hanya berisi kebijakan umum. Sistem **tidak bisa** menjawab pertanyaan yang butuh data pribadi pelanggan, misalnya "pesanan saya nomor 123 sudah sampai mana?" atau "berapa saldo refund saya?".
3. **Knowledge base dianggap benar dan terkini.** Isi FAQ tidak divalidasi ulang dan tidak diubah (sesuai brief). Kalau FAQ diperbarui, embedding dan index FAISS harus dibuat ulang.
4. **Satu entri = satu unit retrievable, tanpa chunking.** Teksnya sudah pendek, dan chunking juga di luar cakupan sesi ini.
5. **Retriever selalu mengembalikan top-k.** FAISS tetap mengembalikan k entri terdekat walaupun tidak ada yang benar-benar relevan. Keputusan "informasinya tidak ada" diserahkan ke LLM melalui instruksi di prompt.
6. **Single-turn.** Setiap pertanyaan diproses sendiri-sendiri, tanpa memori percakapan sebelumnya.
7. **Bahasa Indonesia.** Knowledge base dan query uji berbahasa Indonesia, baik formal maupun informal. Bahasa lain dan bahasa campuran tidak diuji.
8. **Bergantung pada Gemini API.**
   - Hasil bisa berubah mengikuti versi model.
   - Output LLM tidak sepenuhnya deterministik, walaupun temperature diset rendah.
   - Pemakaian dibatasi rate limit dan kuota.
   - Butuh koneksi internet, dan setiap query dikirim ke server Google.
9. **Prototype, bukan sistem produksi.**
   - Evaluasi dilakukan manual pada sekumpulan kecil query uji, bukan benchmark berskala besar.
   - Belum ada UI, monitoring, maupun guardrail produksi.
