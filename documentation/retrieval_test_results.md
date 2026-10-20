# Hasil Pengujian Relevansi

Model: `gemini-embedding-001` + FAISS IndexFlatIP (top-3), LLM `gemini-3.5-flash-lite`.

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
