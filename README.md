# Semantic Search & LLM Application: Asisten FAQ Customer Service E-commerce

Prototype asisten AI yang menjawab pertanyaan pelanggan e-commerce berdasarkan knowledge base FAQ. Cara kerjanya:

1. Sistem mencari entri FAQ yang **maknanya** paling dekat dengan pertanyaan pelanggan (_semantic search_ dengan Gemini Embeddings + FAISS).
2. Entri yang ditemukan disisipkan sebagai context ke prompt.
3. Gemini menyusun jawaban yang natural **hanya** dari context tersebut.

Prototype ini menjadi fondasi sebelum dikembangkan menjadi sistem RAG (Retrieval-Augmented Generation) yang lengkap.

**Hasil singkat** (9 query uji):

- Dari 7 query yang punya jawaban, FAQ yang benar berada di **top-1 pada 6 query** dan di **top-3 pada ketujuhnya**, termasuk query parafrase dan informal.
- LLM tidak mengarang informasi.
- 2 query yang tidak ada di FAQ dijawab dengan fallback.

Detailnya ada di [bagian 7](#7-hasil-pengujian--analisis).

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

## 4. Alur Sistem

```
OFFLINE (dijalankan sekali, hasilnya disimpan)
  CSV FAQ ──► Ingest ──────────► Embed ─────────────────► Store
              pandas              gemini-embedding-001     FAISS IndexFlatL2
              'Q: ... A: ...'     59 × 3072 dimensi        → vector_store/faq.index

ONLINE (setiap pertanyaan pelanggan)
  Query ──► Embed query ──► Retrieve top-3 ──► Prompt (P-T-C-F) ──► Gemini ──► Jawaban + sumber FAQ
```

| Tahap        | Notebook section | Yang dilakukan                                                                                            | Fungsi / objek                        |
| ------------ | ---------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| **Ingest**   | 1–2              | Memuat CSV, memeriksa struktur data, lalu menggabungkan `question` + `answer` menjadi satu teks per entri | `df`, `documents`                     |
| **Embed**    | 3                | Mengubah setiap teks menjadi vektor 3072 dimensi                                                          | `get_embedding()`, `embeddings`       |
| **Store**    | 4                | Memasukkan 59 vektor ke index FAISS, menyimpannya ke file, lalu memverifikasi jumlah vektornya            | `index`, `vector_store/faq.index`     |
| **Retrieve** | 5                | Meng-embed query, mencari 3 FAQ terdekat, dan mengembalikannya beserta `distance`                         | `semantic_search()`                   |
| **Generate** | 6–7              | Menyisipkan hasil retrieval sebagai context ke prompt, lalu mengirim prompt itu ke Gemini                 | `build_prompt()`, `generate_answer()` |

## 5. Struktur Project

```
├── data/
│   └── ecommerce_faq_knowledge_base.csv        # knowledge base asli (tidak diubah)
├── notebook/
│   └── starter_notebook_semantic_search.ipynb  # seluruh pipeline + pengujian + analisis
├── vector_store/
│   └── faq.index                               # FAISS index (dibuat oleh notebook section 4)
├── documentation/
│   └── retrieval_test_results.md               # detail hasil uji per query
├── README.md
├── requirements.txt
├── .env.example                                # template API key; salin menjadi .env
└── .gitignore
```

## 6. Pilihan Teknis

| Komponen         | Pilihan                                                       | Alasan                                                                                                                                                                                    |
| ---------------- | ------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Unit retrievable | `'Q: <question> A: <answer>'` per entri (bawaan starter)      | Pertanyaan membantu mencocokkan query dengan bentuk serupa. Jawaban membawa informasi yang dicari. Teksnya pendek, jadi tidak perlu chunking                                              |
| Model embedding  | `gemini-embedding-001` (3072 dimensi)                         | `text-embedding-004` dari starter sudah tidak tersedia di API                                                                                                                             |
| Vector index     | `faiss.IndexFlatL2` (bawaan starter), disimpan ke `faq.index` | Exact search sudah cukup untuk 59 vektor. Vektor Gemini sudah ternormalisasi (panjang 1), sehingga `distance` L2² = `2 − 2·cosine`. Artinya urutan hasilnya sama dengan cosine similarity |
| `top_k`          | 3                                                             | Memberi cadangan kalau FAQ yang benar tidak berada di posisi pertama (lihat bagian hasil pengujian)                                                                                       |
| Prompt           | Persona, Task, Context bawaan starter + **Format**            | Brief mewajibkan struktur P-T-C-F. Bagian Format mengatur gaya jawaban, melarang menambah atau menyimpulkan detail di luar FAQ, dan mematok kalimat fallback                              |
| Model LLM        | `gemini-3.1-flash-lite`, `temperature=0.3` (bawaan)           | `gemini-2.0-flash` dari starter sudah tidak tersedia. Varian flash-lite cukup untuk menyusun ulang jawaban FAQ. Temperature rendah menjaga jawaban konsisten                              |
| Retry API        | `HttpRetryOptions()` di `genai.Client`                        | Secara default SDK tidak melakukan retry. Selama pengujian, model sempat mengembalikan `503 high demand`. Dengan retry, Run All tetap selesai tanpa error                                 |

## 7. Hasil Pengujian & Analisis

Sistem diuji dengan 9 query yang mencakup empat tipe. FAQ yang diharapkan untuk setiap query ditentukan **sebelum** sistem dijalankan.

| #   | Tipe      | Query                                                     | Top-1 (distance) | Rank FAQ benar | Retrieval | Jawaban LLM                           |
| --- | --------- | --------------------------------------------------------- | ---------------- | :------------: | :-------: | ------------------------------------- |
| 1   | Jelas     | Metode pembayaran apa saja yang tersedia?                 | #17 (0.314)      |       1        |    ✅     | ✅ Sesuai FAQ                         |
| 2   | Jelas     | Bagaimana cara melacak paket yang sedang dikirim?         | #10 (0.260)      |       1        |    ✅     | ✅ Sesuai FAQ                         |
| 3   | Parafrase | Saldo saya sudah terpotong tapi pesanan masih belum lunas | #21 (0.319)      |       1        |    ✅     | ✅ Sesuai FAQ                         |
| 4   | Parafrase | Barang yang datang pecah, harus gimana?                   | #16 (0.336)      |       1        |    ✅     | ✅ Sesuai FAQ                         |
| 5   | Parafrase | Duit refund saya balik ke mana ya?                        | #27 (0.238)      |       1        |    ✅     | ✅ Sesuai FAQ                         |
| 6   | Parafrase | bisa tuker size?                                          | #46 (0.681)      |       2        |    ⚠️     | ✅ Dijawab dari #30 (rank 2)          |
| 7   | Sebagian  | Berapa nomor telepon customer service?                    | #48 (0.272)      |       1        |    ✅     | ⚠️ Tidak mengarang, tapi kontradiktif |
| 8   | Tidak ada | Bisa bayar COD?                                           | #17 (0.612)      |       –        |     –     | ✅ Fallback                           |
| 9   | Tidak ada | Bisa kirim ke luar negeri?                                | #9 (0.585)       |       –        |     –     | ✅ Fallback                           |

**Skor retrieval:** dari 7 query yang punya jawaban, FAQ yang benar berada di top-1 pada **6 query** dan di top-3 pada **ketujuhnya**.

Detail lengkap (top-3, distance, dan jawaban LLM apa adanya) ada di [`documentation/retrieval_test_results.md`](documentation/retrieval_test_results.md). Analisis lengkapnya ada di notebook, cell "Catatan Analisis Relevansi".

### Temuan utama

1. **Semantic search menangkap makna, bukan kata kunci.**
   - Contoh yang berhasil: "terpotong / belum lunas" → "dipotong / belum terbayar" (#21), "pecah" → "rusak" (#16), "duit refund balik ke mana" → #27.
   - Distance query parafrase (0.24–0.34) setara dengan query yang hampir identik dengan FAQ (0.26–0.31). Bahasa informal tidak menurunkan kualitas retrieval.
2. **Titik lemah: query sangat pendek dan ambigu.**
   - "bisa tuker size?" terseret ke #46 (size chart), karena hanya FAQ itu yang memuat kata "size". FAQ yang benar (#30, menukar ukuran) berada di rank 2, dengan selisih distance hanya 0.015.
   - `top_k=3` menyelamatkan kasus ini: LLM tetap menjawab dari #30.
   - Loop bawaan hanya mencetak "Sumber teratas" (#46), sehingga bisa menyesatkan saat debugging.
3. **Distance saja tidak bisa dipakai sebagai threshold penolakan.**
   - Query di luar FAQ punya distance top-1 0.585–0.612, dengan top-3 yang nyaris datar (selisih 0.004–0.029).
   - Tapi query valid "bisa tuker size?" punya distance lebih besar (0.681). Threshold akan menolak query valid.
   - Karena itu, keputusan "informasi tidak tersedia" tepat diserahkan ke LLM. Distance lebih berguna sebagai sinyal ambiguitas.
4. **LLM setia pada context.**
   - Tidak ada nomor, nama bank, atau angka karangan. Nuansa "biasanya" dan angka di FAQ dipertahankan.
   - Untuk "luar negeri", LLM tidak keliru menyamakannya dengan "luar pulau" (FAQ #9). Instruksi "jangan menyimpulkan detail" di bagian Format bekerja.
5. **Fallback aman, tapi kaku.**
   - **COD:** dijawab dengan fallback. Pilihan ini aman, karena FAQ tidak menyatakan COD tidak tersedia. Tapi jawabannya kurang membantu, karena LLM tidak menyebut metode yang tersedia.
   - **Nomor telepon CS:** LLM menjawab sebagian dengan benar ("tertera di halaman bantuan"), lalu menempelkan "Maaf, saya belum memiliki informasi...". Hasilnya kontradiktif.

## 8. Keterbatasan Sistem

Keterbatasan berikut **ditemukan saat pengujian**, di luar batasan desain di bagian 3:

1. **Query sangat pendek dan ambigu rawan salah di top-1.** Kemiripan kata (misalnya "size") bisa mengalahkan maksud pertanyaan.
2. **Tidak ada mekanisme "tidak relevan" di sisi retriever.** FAISS selalu mengembalikan 3 hasil, dan distance tidak memisahkan query relevan dari yang tidak relevan dengan bersih.
3. **Template fallback terlalu kaku untuk jawaban parsial.** Akibatnya jawaban bisa kontradiktif, atau kurang membantu padahal context memuat informasi yang relevan.
4. **Bergantung pada ketersediaan dan kuota API.**
   - Selama pengujian, `gemini-3.1-flash-lite` beberapa kali mengembalikan `503 high demand`. Ini diatasi dengan `HttpRetryOptions`.
   - Kuota free tier embedding adalah 100 teks per menit. Satu kali Run All memakai sekitar 80 teks, karena 59 FAQ di-embed ulang. Menjalankan ulang dalam waktu kurang dari 1 menit bisa memicu `429`.
5. **Output LLM tidak sepenuhnya deterministik.** Teks jawaban bisa sedikit berbeda antar-run, sedangkan hasil retrieval selalu sama.
6. **Index bergantung pada urutan CSV.** FAISS hanya menyimpan vektor. Hubungan `idx` ke `df.iloc[idx]` hanya benar selama urutan data tidak berubah.
7. **Skala evaluasi kecil.** Hasil dari 9 query hanya indikasi, bukan bukti statistik.

## 9. Cara Menjalankan

**Prasyarat:** Python 3.12 dan Gemini API key ([Google AI Studio](https://aistudio.google.com/apikey)).

```bash
git clone https://github.com/rework-ai-engineer/assignment-semantic-search-iqbal-rahmatullah.git
cd assignment-semantic-search-iqbal-rahmatullah

python3.12 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env             # lalu isi GEMINI_API_KEY di file .env
```

Setelah itu:

1. Buka `notebook/starter_notebook_semantic_search.ipynb` di VS Code atau Jupyter.
2. Pilih kernel **`.venv`**.
3. Klik **Run All**. Prosesnya sekitar 3–4 menit, dan sebagian besar waktunya habis untuk embedding 59 FAQ dan 10 panggilan LLM.

**Catatan:**

- Section 4 akan menulis ulang `vector_store/faq.index`.
- Kalau muncul `429 RESOURCE_EXHAUSTED`, kuota embedding per menit sedang habis. Tunggu sekitar 1 menit, lalu jalankan lagi.
