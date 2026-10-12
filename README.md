# Semantic Search & LLM Application
**Individual Assignment — AI Engineering Bootcamp**

---

## Problem Statement

Tim customer service sebuah platform e-commerce menerima ratusan pertanyaan repetitif dari pelanggan setiap harinya — seputar pemesanan, pengiriman, pembayaran, hingga pengembalian barang. Tujuan assignment ini adalah membangun **prototype asisten AI** yang dapat menjawab pertanyaan pelanggan secara otomatis berdasarkan knowledge base FAQ yang sudah ada.

**Objective utama:**
- Mencari entri FAQ paling relevan terhadap pertanyaan pelanggan, meskipun pelanggan tidak menggunakan kata-kata yang persis sama dengan FAQ (semantic search, bukan keyword search).
- Menyusun jawaban yang natural dan akurat menggunakan LLM berdasarkan FAQ yang ditemukan.

**Batasan dan Asumsi:**
- Sistem hanya menjawab berdasarkan FAQ yang tersedia di knowledge base. Jika informasi tidak ditemukan, sistem akan menyatakan ketidaktahuannya dan menyarankan pengguna menghubungi customer service.
- Tidak ada mekanisme update knowledge base secara real-time — knowledge base bersifat statis.
- Sistem belum menangani multi-turn conversation (setiap query diproses secara independen).

---

## Knowledge Base

| Atribut | Detail |
|---|---|
| File | `data/ecommerce_faq_knowledge_base.csv` |
| Jumlah entri | 59 FAQ |
| Kolom | `faq_id`, `category`, `question`, `answer` |
| Kategori | Pemesanan, Pengiriman, Pembayaran, Pengembalian, Akun, Promo, Produk, Customer Service |

Data tidak diubah secara langsung — hanya dibaca dan diproses sebagai unit retrievable (gabungan `question` + `answer`).

---

## Alur Sistem

```
Knowledge Base (CSV)
        │
        ▼
1. INGEST — Muat CSV dengan Pandas, gabungkan question + answer
        │       menjadi satu unit teks per entri
        ▼
2. EMBED — Ubah setiap unit teks menjadi vektor (768 dimensi)
        │   menggunakan Gemini Embeddings API (text-embedding-004)
        ▼
3. STORE — Simpan seluruh vektor ke FAISS IndexFlatL2
        │
        ▼
4. RETRIEVE — Saat ada query masuk, embed query → cari top-K
        │     vektor terdekat (L2 distance) di FAISS index
        ▼
5. GENERATE — Susun prompt terstruktur (Persona/Task/Context/Format)
              yang menyisipkan hasil retrieval → kirim ke Gemini AI
              → hasilkan jawaban natural dalam Bahasa Indonesia
```

---

## Struktur Project

```
semantic-search-assignment/
├── data/
│   └── ecommerce_faq_knowledge_base.csv
├── notebook/
│   └── starter_notebook_semantic_search.ipynb
├── documentation/
│   └── retrieval_test_results.md   (opsional)
├── README.md
└── requirements.txt
```

---

## Cara Menjalankan

### 1. Siapkan environment

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Set API Key

```bash
export GEMINI_API_KEY="your_gemini_api_key_here"
# Windows (PowerShell):
# $env:GEMINI_API_KEY="your_gemini_api_key_here"
```

Dapatkan API key di: https://aistudio.google.com/app/apikey

### 3. Jalankan notebook

```bash
cd notebook
jupyter notebook starter_notebook_semantic_search.ipynb
```

Jalankan semua cell secara berurutan (Run All).

---

## Hasil Pengujian Query

Sistem diuji dengan 5 query yang mencakup skenario berbeda:

| # | Query | Tipe | Relevansi Retrieval | Kualitas Jawaban LLM |
|---|---|---|---|---|
| 1 | "Bagaimana cara melacak paket yang sedang dikirim?" | Langsung / kata kunci eksak | ✅ Sangat relevan | ✅ Akurat |
| 2 | "Bagaimana cara membatalkan pesanan saya?" | Langsung / kata kunci eksak | ✅ Sangat relevan | ✅ Akurat |
| 3 | "Barang saya sudah lama belum nyampe, sudah lewat dari jadwal estimasi" | Parafrase / bahasa informal | ✅ Relevan | ✅ Akurat |
| 4 | "Kalau saya kembalikan produk yang rusak, kapan uang saya bisa balik?" | Sinonim ('uang balik' = refund) | ✅ Relevan | ✅ Akurat |
| 5 | "Apakah ada fitur wishlist atau simpan produk favorit di platform ini?" | Di luar knowledge base | ⚠️ Tidak relevan (topik tidak ada) | ✅ Menolak mengarang |

### Analisis

- **Query 1 & 2** (langsung): Retrieval sempurna karena kalimat query sangat mirip dengan teks FAQ. Distance L2 kecil.
- **Query 3** (parafrase informal): Semantic search berhasil menangkap makna "belum nyampe / lewat jadwal" → memetakan ke FAQ tentang paket melebihi estimasi. Ini keunggulan utama semantic search vs keyword search.
- **Query 4** (sinonim): "Uang saya bisa balik" berhasil dipetakan ke konsep refund/pengembalian dana. Retrieval mengembalikan FAQ tentang durasi refund dan tujuan dana refund — keduanya relevan.
- **Query 5** (di luar KB): Karena tidak ada FAQ tentang wishlist, retrieval menghasilkan entri yang kurang relevan. Namun **LLM berperilaku sesuai instruksi prompt** — menyatakan informasi tidak tersedia dan menyarankan menghubungi customer service. Guardrail berjalan baik.

---

## Keterbatasan Sistem

1. **Knowledge base statis** — Sistem tidak dapat menjawab pertanyaan di luar 59 FAQ yang tersedia. Perlu update manual jika ada FAQ baru.
2. **Tidak ada threshold relevansi** — Sistem selalu mengembalikan top-K hasil meski distance-nya besar (query sangat tidak relevan). Idealnya ada filter distance minimum.
3. **Single-turn only** — Tidak mendukung percakapan multi-turn; konteks percakapan sebelumnya tidak diingat.
4. **Bahasa** — Performa optimal untuk Bahasa Indonesia. Query dalam bahasa lain belum diuji.
5. **FAISS in-memory** — Index FAISS tidak disimpan ke disk, sehingga harus dibuat ulang setiap kali notebook dijalankan. Untuk produksi, perlu `faiss.write_index()`.
