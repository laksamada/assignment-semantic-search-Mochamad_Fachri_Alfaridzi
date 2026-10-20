# Semantic Search & LLM Application

Prototype asisten customer service e-commerce. Pertanyaan pelanggan dicocokkan dengan knowledge base FAQ lewat semantic search (Gemini embeddings + FAISS), lalu jawabannya disusun oleh Gemini berdasarkan FAQ yang ditemukan.

## Problem Statement

- **Objective**: menjawab pertanyaan repetitif pelanggan (pemesanan, pengiriman, pembayaran, pengembalian, dll.) secara otomatis dari FAQ yang sudah ada, termasuk saat pelanggan memakai kata-kata yang berbeda dari FAQ.
- **Knowledge base**: `data/ecommerce_faq_knowledge_base.csv` ([sumber](https://drive.google.com/file/d/1A1m685JChJQ3gvUDh-IBWdj-F2O7_mL_/view?usp=sharing)).
  - Isi: 59 entri, kolom `faq_id`, `category`, `question`, `answer`.
  - 8 kategori: Pemesanan 9, Pengiriman 9, Pembayaran 8, Pengembalian 8, Akun 7, Produk 7, Promo 6, Customer Service 5.
- **Batasan/asumsi**:
  - Sistem hanya menjawab dari FAQ. Jika informasi tidak ada, sistem menyatakan belum tersedia dan mengarahkan pelanggan ke customer service.
  - Jawaban FAQ pendek, jadi satu entri FAQ menjadi satu unit retrieval tanpa chunking.
  - Data asli tidak diubah.

## Alur Sistem

| Tahap | Implementasi |
|---|---|
| Ingest | Load CSV, gabungkan `Kategori + Pertanyaan + Jawaban` jadi satu teks per entri |
| Embed | `gemini-embedding-001`, `task_type=RETRIEVAL_DOCUMENT` untuk dokumen dan `RETRIEVAL_QUERY` untuk query; hasil berupa 59 × 3072 vektor yang dinormalisasi L2 |
| Store | FAISS `IndexFlatIP` (inner product dari vektor ternormalisasi = cosine similarity), disimpan ke `index/faq_index.faiss`, `ntotal = 59` |
| Retrieve | `semantic_search(query, top_k=3)` mengembalikan faq_id, kategori, pertanyaan, jawaban, dan skor similarity |
| Generate | `generate_answer(query)`: retrieval, lalu prompt (Persona, Task, Context, Format), lalu `gemini-3.5-flash-lite` dengan `temperature=0.2` |

Prompt mewajibkan LLM:
- menjawab hanya dari context,
- mengabaikan entri yang tidak relevan,
- tidak menambah kebijakan di luar FAQ,
- menyatakan "belum tersedia di FAQ" jika tidak ada jawaban,
- menutup jawaban dengan `Sumber: FAQ #id`.

## Hasil Pengujian Relevansi

| # | Tipe | Query | Top-1 (skor) | Retrieval | Jawaban LLM |
|---|---|---|---|---|---|
| 1 | direct | Bagaimana cara membatalkan pesanan yang sudah dibuat? | #3 (0.787) | Tepat | Sesuai FAQ #3, sumber dikutip |
| 2 | direct | Metode pembayaran apa saja yang tersedia? | #17 (0.796) | Tepat | Menyalin isi FAQ #17 tanpa tambahan |
| 3 | paraphrase | aku lupa kata sandi, gimana caranya bisa masuk lagi ke akunku? | #32 (0.775) | Tepat | Diubah jadi 3 langkah bernomor sesuai format |
| 4 | paraphrase | saldo m-banking udah kepotong tapi orderan masih nunggu dibayar, kok gitu? | #21 (0.769) | Tepat | Sesuai FAQ #21 (tunggu 15-30 menit / hubungi CS) |
| 5 | paraphrase | barang yang dateng pecah dan penyok, harus ngapain? | #16 (0.800) | Tepat | Sesuai FAQ #16 (foto/video, lapor 2x24 jam) |
| 6 | out-of-kb | Apakah ada toko offline di Surabaya untuk ambil barang langsung? | #58 (0.642) | Tidak relevan (memang tidak ada) | Menyatakan info belum tersedia, arahkan ke CS, `Sumber: -` |
| 7 | out-of-kb | Bisa kirim paket ke luar negeri, misalnya ke Singapura? | #15 (0.689) | Tidak relevan (memang tidak ada) | Menyatakan info belum tersedia, arahkan ke CS, `Sumber: -` |

Hit@1 dan Hit@3 untuk 5 query in-KB: 100%.

**Retrieval akurat.**
- Kelima query yang jawabannya ada di FAQ mendapat entri yang benar di peringkat 1.
- Query parafrase memakai kata yang tidak ada di FAQ sama sekali, tapi tetap ketemu karena embedding menangkap makna:
  - "kata sandi" / "masuk lagi" → "password" / "mengatur ulang"
  - "saldo m-banking kepotong" / "orderan nunggu dibayar" → "pembayaran dipotong" / "status belum terbayar"
  - "pecah dan penyok" → "kondisi rusak"
- Skor top-1 query parafrase (rata-rata 0.781) hampir sama dengan query langsung (0.791).
- Skor query langsung tidak mendekati 1.0, padahal teksnya identik dengan pertanyaan FAQ. Ini karena unit yang di-embed juga berisi kategori dan jawaban, bukan pertanyaan saja.

**Retrieval kurang relevan.**
- Dua query out-of-KB tetap mengembalikan 3 entri, karena FAISS selalu mengembalikan top-k. Entri-entri itu hanya mirip topik:
  - toko offline → FAQ reseller, same-day, dan stok
  - kirim ke luar negeri → FAQ jasa pengiriman dan asuransi
- Skor top-1-nya (0.642 dan 0.689) jelas lebih rendah daripada semua query in-KB (minimal 0.769).
- Margin antara peringkat 1 dan 3 juga sangat tipis (≤0.006), artinya tidak ada entri yang menonjol.
- Di peringkat 2-3 query in-KB masih ada entri yang hanya mirip topik, misalnya #24 pengembalian untuk query pembatalan. Inilah alasan prompt menyuruh LLM mengabaikan entri yang tidak menjawab.

**Kualitas jawaban LLM.**
- Pada 5 query in-KB, jawaban hanya mengambil dari FAQ peringkat 1, mengikuti format (langkah bernomor untuk prosedur), dan menyebut sumber dengan benar.
- Entri peringkat 2-3 yang tidak relevan tidak ikut dicampur ke jawaban.
- Pada 2 query out-of-KB, LLM tidak mengarang, misalnya tidak menyebut alamat toko atau tarif ongkir internasional. LLM menyatakan informasinya belum tersedia dan mengarahkan ke customer service, sesuai instruksi di bagian TASK.

**Keterbatasan.**
- **Tidak ada threshold skor.** Penolakan untuk query out-of-KB sepenuhnya bergantung pada LLM. Dari data uji ini, threshold sekitar 0.72 sudah memisahkan in-KB (≥0.769) dari out-of-KB (≤0.689) dan bisa menghemat panggilan LLM. Tapi angka ini baru dari 7 query, jadi perlu divalidasi dengan query yang lebih banyak.
- **Jumlah uji kecil.** Hanya 7 query, dan pertanyaan majemuk (dua topik sekaligus) belum diuji.
- **Ketergantungan pada API.** Embedding dan LLM tergantung kuota free tier Gemini. `gemini-3.8-flash` hanya 20 request/hari, sehingga dipakai `gemini-3.5-flash-lite`.
- **Knowledge base kecil.** Hanya 59 entri, banyak pertanyaan umum tidak tercakup (toko offline, pengiriman internasional).

Output lengkap (top-3, skor, dan jawaban LLM) ada di Section 8 notebook dan di `documentation/retrieval_test_results.csv`.

## Struktur Project

```
semantic-search-assignment/
├── data/ecommerce_faq_knowledge_base.csv
├── notebook/starter_notebook_semantic_search.ipynb
├── index/faq_index.faiss
├── documentation/
│   ├── retrieval_test_results.md
│   └── retrieval_test_results.csv
├── .env.example
├── README.md
└── requirements.txt
```

## Cara Menjalankan

```bash
pip install -r requirements.txt
cp .env.example .env    # isi GEMINI_API_KEY dari https://aistudio.google.com/apikey
cd notebook
jupyter notebook starter_notebook_semantic_search.ipynb
```

Jalankan semua cell dari atas. Notebook membuat ulang embedding dan index FAISS, lalu menjalankan 7 query uji (sekitar 20 request ke Gemini API).
