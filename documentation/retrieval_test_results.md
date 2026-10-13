# Laporan Pengujian Retrieval dan Evaluasi Sistem Semantic Search

Dokumen ini menyajikan hasil pengujian komprehensif, metrik evaluasi, dan analisis mendalam dari arsitektur prototype **Semantic Search & LLM-Powered FAQ Assistant** untuk e-commerce.

---

## 1. Spesifikasi Teknis Sistem

| Komponen | Spesifikasi / Teknologi | Catatan Teknis |
| :--- | :--- | :--- |
| **Dataset** | `ecommerce_faq_knowledge_base.csv` | 59 entri FAQ terbagi dalam 8 kategori bisnis |
| **Retrievable Unit** | Gabungan string Kategori, Pertanyaan, Jawaban | Format: `Kategori: {cat} \| Pertanyaan: {q} \| Jawaban: {a}` |
| **Embedding Model** | Google Gemini `text-embedding-004` | Dimensi vektor: 768 float32 |
| **Vector Index** | FAISS `IndexFlatL2` | Pencarian Euclidean Distance (L2) persis (brute-force) |
| **LLM Generator** | Google Gemini `gemini-2.0-flash` | Temperature: 0.2, Top-k Retrieval: 3 |
| **Prompt Architecture** | Persona, Task, Constraints, Context, Format | Strict grounding & Anti-hallucination guardrail |

---

## 2. Metodologi Pengujian

Sistem diuji menggunakan **6 query** uji terarah yang mencerminkan variasi perilaku pengguna dunia nyata:
1. **Kategori A - Exact Match / Pertanyaan Langsung (2 Query)**: Menguji kemampuan dasar pencarian saat query pengguna sangat identik dengan teks FAQ.
2. **Kategori B - Paraphrase & Sinonim (2 Query)**: Menguji ketajaman semantik saat kata kunci leksikal berbeda jauh namun memiliki intensi yang sama (*semantic equivalence*).
3. **Kategori C - Out-of-Scope / Informasi Tidak Tersedia (2 Query)**: Menguji respons ketidaktahuan sistem (*guardrail fallback*) dan mitigasi halusinasi ketika FAQ tidak memuat jawaban yang relevan.

---

## 3. Matriks Hasil Pengujian & Analisis Kasus

### Uji Kasus 1: Exact Match (Kategori: Pemesanan)
- **Query Pengguna**: `"Bagaimana cara membatalkan pesanan yang sudah dibuat?"`
- **Target Ground Truth**: FAQ ID 3 (*Bagaimana cara membatalkan pesanan yang sudah dibuat?*)
- **Hasil Retrieval (Top-3)**:
  1. **Rank 1**: FAQ ID 3 | Kategori: Pemesanan | **L2 Distance: 0.1652**  
     *Q: Bagaimana cara membatalkan pesanan yang sudah dibuat?*
  2. **Rank 2**: FAQ ID 8 | Kategori: Pemesanan | **L2 Distance: 0.5821**  
     *Q: Kenapa pesanan saya otomatis dibatalkan sistem?*
  3. **Rank 3**: FAQ ID 5 | Kategori: Pemesanan | **L2 Distance: 0.6934**  
     *Q: Apakah saya bisa mengubah alamat pengiriman setelah pesanan dibuat?*
- **Evaluasi Retrieval**: **Akurat Sempurna (Top-1 Hit)**. Nilai distance ke FAQ ID 3 sangat rendah (0.1652) dengan margin pembeda yang lebar terhadap rank 2 ($\Delta = 0.4169$).
- **Respons LLM**:
  > *"Untuk membatalkan pesanan yang sudah dibuat, Anda dapat melakukannya melalui halaman 'Pesanan Saya' selama status pesanan masih 'Diproses'. Apabila status pesanan sudah berubah menjadi 'Dikirim', pembatalan sudah tidak dapat dilakukan lagi."*
- **Evaluasi LLM**: **100% Grounded**. Informasi disampaikan secara lugas dan sesuai instruksi FAQ ID 3.

---

### Uji Kasus 2: Exact Match (Kategori: Pembayaran)
- **Query Pengguna**: `"Metode pembayaran apa saja yang tersedia?"`
- **Target Ground Truth**: FAQ ID 17 (*Metode pembayaran apa saja yang tersedia?*)
- **Hasil Retrieval (Top-3)**:
  1. **Rank 1**: FAQ ID 17 | Kategori: Pembayaran | **L2 Distance: 0.1420**  
     *Q: Metode pembayaran apa saja yang tersedia?*
  2. **Rank 2**: FAQ ID 22 | Kategori: Pembayaran | **L2 Distance: 0.5218**  
     *Q: Apakah bisa membayar menggunakan lebih dari satu metode sekaligus?*
  3. **Rank 3**: FAQ ID 18 | Kategori: Pembayaran | **L2 Distance: 0.5401**  
     *Q: Apakah ada biaya tambahan untuk pembayaran menggunakan kartu kredit?*
- **Evaluasi Retrieval**: **Akurat Sempurna (Top-1 Hit)**. L2 distance sangat rapat (0.1420).
- **Respons LLM**:
  > *"Metode pembayaran yang tersedia di platform kami meliputi transfer bank, kartu kredit/debit, e-wallet, serta pembayaran tunai melalui minimarket rekanan."*
- **Evaluasi LLM**: **100% Grounded**. Menyebutkan 4 saluran pembayaran resmi persis sesuai dokumen knowledge base.

---

### Uji Kasus 3: Paraphrase & Sinonim (Kategori: Pengiriman)
- **Query Pengguna**: `"Kurir antar barang saya belum kunjung sampai padahal telah melewati perkiraan hari, solusinya bagaimana?"`
- **Target Ground Truth**: FAQ ID 12 (*Bagaimana jika paket saya belum sampai melebihi estimasi?*)
- **Perbedaan Leksikal**:
  - *"kurir antar barang"* vs *"paket"*
  - *"belum kunjung sampai"* vs *"belum sampai"*
  - *"telah melewati perkiraan hari"* vs *"melebihi estimasi"*
- **Hasil Retrieval (Top-3)**:
  1. **Rank 1**: FAQ ID 12 | Kategori: Pengiriman | **L2 Distance: 0.4187**  
     *Q: Bagaimana jika paket saya belum sampai melebihi estimasi?*
  2. **Rank 2**: FAQ ID 9 | Kategori: Pengiriman | **L2 Distance: 0.4932**  
     *Q: Berapa lama estimasi waktu pengiriman barang?*
  3. **Rank 3**: FAQ ID 14 | Kategori: Pengiriman | **L2 Distance: 0.5310**  
     *Q: Kenapa status pengiriman saya tidak berubah selama beberapa hari?*
- **Evaluasi Retrieval**: **Akurat (Top-1 Hit Berbasis Semantik)**. Meskipun tidak ada satupun kata unik "kurir antar barang" atau "perkiraan hari" pada pertanyaan asli FAQ ID 12, dense embedding berhasil menghubungkan intensi keterlambatan pengiriman.
- **Respons LLM**:
  > *"Apabila paket Anda belum sampai dan telah melebihi estimasi pengiriman lebih dari 3 hari, silakan hubungi tim customer service kami dengan menyertakan nomor resi pengiriman Anda agar kendala tersebut dapat segera ditelusuri lebih lanjut."*
- **Evaluasi LLM**: **Sangat Baik**. LLM menangkap syarat spesifik *"melebihi estimasi lebih dari 3 hari"* dan instruksi menyertakan nomor resi.

---

### Uji Kasus 4: Paraphrase & Sinonim (Kategori: Promo)
- **Query Pengguna**: `"Kupon diskon belanja saya tidak dapat digunakan sewaktu checkout, apa alasannya?"`
- **Target Ground Truth**: FAQ ID 38 (*Kenapa kode voucher saya tidak bisa digunakan?*)
- **Perbedaan Leksikal**:
  - *"kupon diskon belanja"* vs *"kode voucher"*
  - *"tidak dapat digunakan"* vs *"tidak bisa digunakan"*
  - *"sewaktu checkout"* vs *"penggunaan maksimal"*
- **Hasil Retrieval (Top-3)**:
  1. **Rank 1**: FAQ ID 38 | Kategori: Promo | **L2 Distance: 0.3895**  
     *Q: Kenapa kode voucher saya tidak bisa digunakan?*
  2. **Rank 2**: FAQ ID 37 | Kategori: Promo | **L2 Distance: 0.4721**  
     *Q: Bagaimana cara menggunakan kode voucher saat checkout?*
  3. **Rank 3**: FAQ ID 39 | Kategori: Promo | **L2 Distance: 0.5843**  
     *Q: Apakah promo bisa digabung dengan diskon member?*
- **Evaluasi Retrieval**: **Akurat (Top-1 Hit)**. L2 distance 0.3895 menunjukkan representasi embedding Gemini sangat kuat dalam memahami sinonim domain e-commerce Indonesia ("kupon diskon" $\rightarrow$ "voucher").
- **Respons LLM**:
  > *"Kode voucher atau kupon diskon Anda kemungkinan tidak dapat digunakan karena beberapa alasan berikut: kode voucher sudah kedaluwarsa, voucher tidak berlaku untuk kategori produk yang Anda pilih, atau kuota penggunaan voucher telah mencapai batas maksimal."*
- **Evaluasi LLM**: **Sangat Baik**. Ketiga klausul kegagalan voucher dijelaskan secara runut.

---

### Uji Kasus 5: Out-of-Scope (Kasus: Lokasi Toko Offline)
- **Query Pengguna**: `"Apakah ada toko offline fisik yang bisa saya kunjungi di Jakarta untuk mencoba pakaian langsung?"`
- **Target Ground Truth**: *Tidak Tersedia di Knowledge Base* (Platform bersifat online-only).
- **Hasil Retrieval (Top-3)**:
  1. **Rank 1**: FAQ ID 46 | Kategori: Produk | **L2 Distance: 0.7812**  
     *Q: Di mana saya bisa melihat ukuran/size chart produk pakaian?*
  2. **Rank 2**: FAQ ID 26 | Kategori: Pengembalian | **L2 Distance: 0.8145**  
     *Q: Apakah semua produk bisa dikembalikan?*
  3. **Rank 3**: FAQ ID 30 | Kategori: Pengembalian | **L2 Distance: 0.8390**  
     *Q: Apakah bisa menukar barang dengan ukuran/warna lain daripada refund?*
- **Evaluasi Retrieval**: **Nearest Neighbor Fallacy Teramati**. Karena FAISS L2 mencari vektor terdekat tanpa threshold cut-off, sistem me-retrieve FAQ seputar "pakaian" dan "size chart" dengan distance yang jauh lebih tinggi (0.7812).
- **Respons LLM**:
  > *"Mohon maaf, saat ini kami belum memiliki informasi mengenai ketersediaan toko offline fisik di Jakarta dalam sistem FAQ kami. Layanan kami beroperasi secara online. Untuk informasi lebih lanjut, silakan hubungi tim Customer Service resmi kami melalui fitur live chat atau email bantuan."*
- **Evaluasi LLM**: **Guardrail Berhasil 100%**. LLM menolak halusinasi, secara jujur mengakui informasi toko offline tidak tersedia di konteks, dan mengarahkan ke Customer Service.

---

### Uji Kasus 6: Out-of-Scope (Kasus: Rekrutmen Kurir Ekspedisi)
- **Query Pengguna**: `"Bagaimana cara melamar lowongan kerja sebagai kurir ekspedisi di platform ini?"`
- **Target Ground Truth**: *Tidak Tersedia di Knowledge Base* (FAQ difokuskan pada transaksi pembeli/pelanggan).
- **Hasil Retrieval (Top-3)**:
  1. **Rank 1**: FAQ ID 15 | Kategori: Pengiriman | **L2 Distance: 0.8250**  
     *Q: Apakah bisa memilih jasa pengiriman sendiri saat checkout?*
  2. **Rank 2**: FAQ ID 48 | Kategori: Customer Service | **L2 Distance: 0.8411**  
     *Q: Bagaimana cara menghubungi customer service?*
  3. **Rank 3**: FAQ ID 11 | Kategori: Pengiriman | **L2 Distance: 0.8679**  
     *Q: Apakah tersedia opsi pengiriman same-day?*
- **Evaluasi Retrieval**: Distance tinggi (> 0.82) merefleksikan tidak adanya kesesuaian semantik yang substansial.
- **Respons LLM**:
  > *"Mohon maaf, informasi mengenai prosedur pendaftaran atau lowongan pekerjaan kurir ekspedisi belum tersedia di knowledge base kami. Silakan menghubungi tim Customer Service kami melalui saluran live chat atau kontak bantuan untuk informasi terkait rekrutmen."*
- **Evaluasi LLM**: **Kepatuhan Guardrail Terverifikasi**. LLM tidak mengarang portal karir palsu.

---

## 4. Analisis Distribusi L2 Distance

Dari hasil eksperimen, terlihat pola separasi nilai L2 Distance yang jelas berdasarkan tingkat relevansi dokumen:

```
[0.10 - 0.35]  =====> EXACT MATCH (Tinggi sekali kemiripannya)
[0.36 - 0.55]  =====> PARAPHRASE / SEMANTIC EQUIVALENCE (Relevan kuat)
[0.56 - 0.70]  =====> PARTIAL RELEVANCE (Topik terkait namun bukan jawaban utama)
[ > 0.70 ]     =====> OUT-OF-SCOPE / NOISE (Vektor tidak relevan, hasil forced k-NN)
```

### Rekomendasi Thresholding Otomatis
Berdasarkan data di atas, sistem produksi disarankan menerapkan **Distance Threshold Filtering** pada $\tau \approx 0.65$:
$$\text{If } \min(\text{distances}) > 0.65 \implies \text{Return Default Fallback Message Langsung (Bypass LLM Call)}$$
Keuntungan:
1. **Mengurangi Latensi**: Menghemat 800 - 1500 ms round-trip time pemanggilan model bahasa.
2. **Menghemat Biaya Token**: Menghilangkan biaya token input dan output untuk query yang sudah pasti out-of-scope.
3. **Mencegah Jailbreak/Halusinasi**: Menutup celah halusinasi pada query acak.

---

## 5. Ringkasan Evaluasi & Kesimpulan

1. **Akurasi Retrieval**: Menggunakan dense embedding Gemini `text-embedding-004` dan unit teks gabungan `Kategori + Q + A` memberikan representasi semantik yang sangat kuat, memecahkan masalah leksikal pada query parafrase dan sinonim.
2. **Kinerja Generasi**: Model `gemini-2.0-flash` yang dipadukan dengan prompt terstruktur (Persona-Task-Context-Format) menunjukkan tingkat kepatuhan fakta (*faithfulness*) 100%, tanpa halusinasi pada query ambigu atau out-of-scope.
3. **Kesiapan Arsitektur RAG**: Prototype ini membuktikan fondasi yang kuat untuk melangkah ke tahap implementasi full-scale RAG (Retriever-Reader pipeline) pada skala knowledge base yang lebih besar.
