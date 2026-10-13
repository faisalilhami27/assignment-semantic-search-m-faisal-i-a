# Semantic Search & LLM-Powered FAQ Assistant

Repositori ini berisi implementasi prototype sistem **Semantic Search yang terintegrasi dengan Large Language Model (LLM)** untuk menjawab pertanyaan customer service e-commerce secara otomatis, akurat, dan grounded pada knowledge base FAQ.

Proyek ini merupakan bagian dari **Individual Assignment - AI Engineering Bootcamp: Semantic Search & LLM Application (Advanced Prompt Engineering & Context Management)**.

---

## 1. Problem Statement & Deskripsi Knowledge Base

### Problem Statement
Tim customer service e-commerce menerima ratusan pertanyaan repetitif setiap harinya seputar pemesanan, pengiriman, pembayaran, hingga pengembalian barang. Pencarian leksikal konvensional (berbasis pencocokan kata kunci eksak) sering gagal melayani pertanyaan pelanggan karena pelanggan cenderung menggunakan sinonim, bahasa sehari-hari, atau kalimat parafrase yang tidak persis sama dengan dokumen FAQ.

**Objective**: Membangun prototype awal asisten AI pintar yang mampu:
1. Menemukan entri FAQ paling relevan berdasarkan makna semantik (*semantic similarity*), bukan sekadar kemiripan kata kunci.
2. Menyusun jawaban yang alami, ramah, dan solutif menggunakan LLM yang dibatasi secara ketat (*grounded*) hanya pada konteks FAQ yang ditemukan.
3. Menolak menjawab secara sopan (*graceful fallback*) dan anti-halusinasi jika informasi tidak tersedia di knowledge base.

### Karakteristik Knowledge Base
- **File**: `data/ecommerce_faq_knowledge_base.csv`
- **Total Entri**: 59 entri FAQ
- **Struktur Kolom**:
  - `faq_id`: Identifikator unik entri FAQ (1–59).
  - `category`: Klasifikasi kategori layanan (`Pemesanan`, `Pengiriman`, `Pembayaran`, `Pengembalian`, `Akun`, `Promo`, `Produk`, `Customer Service`).
  - `question`: Pertanyaan standar pelanggan.
  - `answer`: Jawaban resmi dan kebijakan platform.

---

## 2. Arsitektur & Alur Sistem (End-to-End Pipeline)

Sistem bekerja melalui 5 tahapan utama pipeline Retrieval-Augmented Generation (RAG dasar):

```
[CSV Knowledge Base] 
         │
         ▼ (1. Ingest)
[Unit Retrievable: Kategori + Question + Answer]
         │
         ▼ (2. Embed)
[Gemini gemini-embedding-001 (3072 Dimensi)]
         │
         ▼ (3. Store)
[FAISS IndexFlatL2 Vector Database]
         │
         ▼ (4. Retrieve) ◄─── [Query Pengguna: "Kupon diskon tidak bisa dipakai"]
[Top-3 Entri FAQ Paling Relevan + Distance Score]
         │
         ▼ (5. Generate)
[Structured Prompt: Persona + Task + Context + Format]
         │
         ▼
[Google Gemini 3.5 Flash Lite LLM]
         │
         ▼
[Jawaban Final Grounded & Ramah]
```

1. **Ingest**: Membaca dataset CSV dengan Pandas dan menyusun unit teks representatif yang menggabungkan metadata kategori, pertanyaan, dan jawaban:
   $$\text{Content} = \text{"Kategori: "} + \text{category} + \text{" | Pertanyaan: "} + \text{question} + \text{" | Jawaban: "} + \text{answer}$$
2. **Embed**: Mengonversi setiap unit teks menjadi vektor dense embedding berdimensi 3072 menggunakan Google Gemini Embeddings API (`gemini-embedding-001`).
3. **Store**: Menyimpan seluruh vektor embedding ke dalam vector database FAISS menggunakan index Euclidean Distance (`faiss.IndexFlatL2`).
4. **Retrieve**: Menerima query teks pengguna, mengonversinya menjadi query embedding, dan melakukan pencarian $k$-nearest neighbors pada index FAISS untuk menghasilkan Top-$k$ entri FAQ beserta jarak Euclidean (*L2 distance*).
5. **Generate**: Merangkai hasil retrieval ke dalam prompt terstruktur (Persona, Task, Constraints, Context, Format) dan memanggil LLM (`gemini-3.5-flash-lite`) untuk menghasilkan respon alami tanpa halusinasi.

---

## 3. Desain Prompt & Guardrail

Prompt dirancang mengikuti arsitektur **Persona - Task - Constraints - Context - Format** dengan prinsip *zero-hallucination guardrail*:

- **Persona**: Customer Service Virtual AI e-commerce yang ramah, sopan, dan profesional.
- **Task**: Menjawab pertanyaan pengguna secara ringkas dan solutif HANYA berdasarkan konteks FAQ.
- **Constraints**:
  1. *Grounded Knowledge*: Dilarang berasumsi atau menggunakan pengetahuan umum di luar konteks yang diinjeksikan.
  2. *Out-of-Scope Fallback*: Jika informasi tidak ada di konteks, model wajib menyatakan secara jujur ketidaktahuannya dan menyarankan menghubungi Customer Service resmi.
  3. *No Hallucination*: Dilarang mengarang prosedur, durasi hari, atau nomor kontak fiktif.
- **Context**: Injeksi terstruktur dari Top-3 hasil semantic search (mencakup ID, Kategori, Pertanyaan FAQ, dan Jawaban FAQ).
- **Format**: Jawaban langsung dalam Bahasa Indonesia santun.

---

## 4. Hasil Pengujian & Evaluasi Relevansi

Sistem diuji menggunakan 6 skenario query komprehensif:

| No | Tipe Query | Query Pengguna | Top-1 FAQ Retrieved | L2 Dist | Status Retrieval | Evaluasi Jawaban LLM |
| :---: | :--- | :--- | :--- | :---: | :---: | :--- |
| **1** | Exact Match | *"Bagaimana cara membatalkan pesanan yang sudah dibuat?"* | FAQ ID 3 (Pemesanan) | 0.1652 | Akurat | **100% Grounded**. Menyebutkan pembatalan lewat menu Pesanan Saya sebelum status 'Dikirim'. |
| **2** | Exact Match | *"Metode pembayaran apa saja yang tersedia?"* | FAQ ID 17 (Pembayaran) | 0.1420 | Akurat | **100% Grounded**. Menyebutkan transfer bank, kartu kredit/debit, e-wallet, dan minimarket. |
| **3** | Paraphrase | *"Kurir antar barang saya belum kunjung sampai padahal telah melewati perkiraan hari, solusinya bagaimana?"* | FAQ ID 12 (Pengiriman) | 0.4187 | Akurat | **Sangat Baik**. Berhasil menangkap sinonim keterlambatan paket tanpa kata kunci persis. Menyarankan hubungi CS jika > 3 hari. |
| **4** | Paraphrase | *"Kupon diskon belanja saya tidak dapat digunakan sewaktu checkout, apa alasannya?"* | FAQ ID 38 (Promo) | 0.3895 | Akurat | **Sangat Baik**. Menghubungkan "kupon diskon" dengan FAQ "kode voucher". Menjelaskan 3 alasan kedaluwarsa/kategori/kuota. |
| **5** | Out-of-Scope | *"Apakah ada toko offline fisik yang bisa saya kunjungi di Jakarta untuk mencoba pakaian langsung?"* | FAQ ID 46 (Produk - Size Chart) | 0.7812 | Irrelevan (Noise) | **Guardrail Berhasil**. LLM menolak halusinasi, menyatakan informasi toko offline tidak ada di FAQ, dan mengarahkan ke CS. |
| **6** | Out-of-Scope | *"Bagaimana cara melamar lowongan kerja sebagai kurir ekspedisi di platform ini?"* | FAQ ID 15 (Pengiriman) | 0.8250 | Irrelevan (Noise) | **Guardrail Berhasil**. LLM secara jujur menyatakan info rekrutmen tidak ada dalam FAQ bantuan pelanggan. |

> *Detail lengkap hasil pengujian, metrik jarak, dan analisis kegagalan dapat dilihat pada dokumen [documentation/retrieval_test_results.md](documentation/retrieval_test_results.md).*

---

## 5. Keterbatasan Sistem (System Limitations)

1. **Forced Nearest Neighbor pada Vector Search**:
   FAISS selalu mengembalikan top-$k$ dokumen terdekat bahkan ketika query benar-benar *out-of-scope*. Nilai L2 distance pada query out-of-scope melonjak tinggi (> 0.70), namun dokumen tetap dikirim ke prompt LLM.
2. **Ketergantungan Tunggal pada Dense Retrieval**:
   Model embedding sangat kuat menangkap makna semantik, namun rentan pada kata kunci spesifik tingkat tinggi (misal: kode promo alfanumerik spesifik atau nomor resi unik) yang lebih optimal ditangani oleh pencarian leksikal (BM25).
3. **Belum Menggunakan Re-ranking**:
   Sistem bi-encoder retrieval menghitung kemiripan secara global tanpa cross-attention antara query dan dokumen individual, sehingga urutan top-$k$ masih bisa dioptimalkan lebih lanjut.

---

## 6. Rekomendasi Peningkatan Produksi

- **Distance Threshold Gate**: Pasang ambang batas L2 distance ($\tau \approx 0.65$). Jika $\text{distance} > \tau$, sistem langsung mengembalikan pesan fallback standar tanpa memanggil API LLM (mengurangi latensi hingga 80% dan memangkas biaya token).
- **Hybrid Search**: Gabungkan BM25 leksikal dengan FAISS dense vector menggunakan algoritma *Reciprocal Rank Fusion* (RRF).
- **Index Normalization**: Lakukan normalisasi L2 pada vektor dan gunakan Cosine Similarity (`faiss.IndexFlatIP`) agar rentang skor kemiripan berada pada skala terstandardisasi 0 hingga 1.

---

## 7. Struktur Folder Proyek

```
semantic-search-assignment/
├── data/
│   └── ecommerce_faq_knowledge_base.csv       # Dataset knowledge base 59 entri FAQ
├── notebook/
│   └── starter_notebook_semantic_search.ipynb # Notebook implementasi end-to-end
├── documentation/
│   └── retrieval_test_results.md              # Laporan analisis pengujian retrieval
├── README.md                                  # Dokumentasi utama proyek
└── requirements.txt                           # Dependensi library Python
```

---

## 8. Panduan Menjalankan Proyek

### Prasyarat
- Python 3.10+
- Google GenAI API Key (`GEMINI_API_KEY`)

### Langkah Instalasi

1. **Clone repository dan masuk ke direktori proyek**:
   ```bash
   git clone <url-repository-anda>
   cd semantic-search-assignment
   ```

2. **Buat dan aktifkan virtual environment**:
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate  # Untuk Linux/macOS
   # .venv\Scripts\activate   # Untuk Windows
   ```

3. **Install dependensi library**:
   ```bash
   pip install -r requirements.txt
   ```

4. **Konfigurasi Environment Variable API Key**:
   Buat file `.env` di root direktori:
   ```env
   GEMINI_API_KEY=AIzaSy...isi_api_key_anda_disini...
   ```

5. **Jalankan Jupyter Notebook**:
   ```bash
   jupyter notebook notebook/starter_notebook_semantic_search.ipynb
   ```
   Atau buka file `notebook/starter_notebook_semantic_search.ipynb` langsung di editor (VS Code / Cursor) dan jalankan seluruh cell dari atas ke bawah secara berurutan.

---

## 9. Checklist Pemenuhan Komponen Tugas

| No | Komponen Output Sesuai Rubrik | Lokasi di Repositori | Status |
|:---:|:---|:---|:---:|
| 1 | Document / knowledge base processing code | `notebook/starter_notebook_semantic_search.ipynb` (Bagian 1 & 2) | Selesai |
| 2 | Embedding pipeline code | `notebook/starter_notebook_semantic_search.ipynb` (Bagian 3) | Selesai |
| 3 | Vector database (FAISS index) beserta script pembuatannya | `notebook/starter_notebook_semantic_search.ipynb` (Bagian 4) | Selesai |
| 4 | Semantic search function | `notebook/starter_notebook_semantic_search.ipynb` (Bagian 5) | Selesai |
| 5 | LLM-powered response function (prompt design + integrasi LLM) | `notebook/starter_notebook_semantic_search.ipynb` (Bagian 6 & 7) | Selesai |
| 6 | Hasil pengujian relevansi (minimal 5 query) beserta analisis | `notebook/starter_notebook_semantic_search.ipynb` (Bagian 8 - 6 skenario pengujian)<br>`documentation/retrieval_test_results.md` | Selesai |
| 7 | File `requirements.txt` | `requirements.txt` di root repository | Selesai |
