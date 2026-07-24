# Laporan In-Depth Benchmarking Model RAG

**Proyek:** Sistem Retrieval-Augmented Generation (RAG)  
**Dokumen Uji:** *Corporate Governance and Workplace Mental Health Practices: The Mediating Role of Structured Occupational Safety and Health Engagement*  
**Model:** `intfloat/multilingual-e5-small`  
**Mode:** Local CPU  
**Dimensi Embedding:** 384  
**Top-K:** 5  

> **Catatan integritas data:** Eksperimen #1 menggunakan konfigurasi dan proses ingestion yang benar-benar sudah dijalankan. Angka Hit Rate dan latensi pada tabel berikut merupakan **simulasi/estimasi untuk contoh format laporan**, karena pengukuran lima eksperimen lengkap belum dilakukan.

---

## 1. Tabel Hasil Evolusi Benchmarking

Tabel ini menggambarkan simulasi perjalanan optimasi sistem retrieval melalui tiga variabel utama, yaitu penggunaan prefix E5, penyelarasan bahasa query, dan ukuran chunk.

| Eksperimen | Uk. Chunk | Overlap | Tipe Query Uji | Parameter Optimasi | Top-K | Hit Rate* | Latensi* | Catatan Temuan |
|---:|---:|---:|---|---|---:|---:|---:|---|
| **#1** | 1000 | 200 | Query Indonesia | *Baseline* tanpa prefix | 5 | **70%** | **~430 ms** | Model sudah mampu melakukan retrieval lintas bahasa, tetapi beberapa query reasoning dan noisy belum stabil. |
| **#2** | 1000 | 200 | Query Indonesia | Prefix `query:` dan `passage:` | 5 | **80%** | **~465 ms** | Prefix membantu model membedakan teks pertanyaan dan dokumen sehingga retrieval menjadi lebih terarah. |
| **#3** | 1000 | 200 | Query Inggris | Prefix + translasi query | 5 | **90%** | **~445 ms** | Bahasa query disamakan dengan bahasa dokumen sehingga *cross-lingual gap* berkurang. |
| **#4** | 500 | 100 | Query Indonesia | Prefix + chunk lebih kecil | 5 | **90%** | **~520 ms** | Chunk yang lebih padat membantu menemukan fakta spesifik dan mengurangi gangguan dari konteks yang terlalu panjang. |
| **#5** | 500 | 100 | Query Inggris | Prefix + chunk kecil + bahasa Inggris | 5 | **100%** | **~505 ms** | Kombinasi optimasi memberikan hasil terbaik pada sepuluh query uji, dengan tambahan beban pencarian yang masih ringan. |

\* Angka Hit Rate dan latensi pada tabel merupakan **simulasi**, bukan hasil pengukuran aktual.

---

## 2. Hasil Ingestion Aktual

Proses ingestion PDF yang telah dijalankan menghasilkan data berikut:

| Komponen | Hasil |
|---|---:|
| Panjang Markdown | 45.813 karakter |
| Jumlah halaman | 7 |
| Jumlah chunk | 66 |
| Rata-rata panjang chunk | 742 karakter |
| Chunk teks | 61 |
| Chunk tabel | 4 |
| Chunk keterangan gambar | 1 |
| Dimensi vektor | 384 |
| Points di Qdrant | 66 |
| Vectors di Qdrant | 66 |
| Status collection | Green |
| Waktu ingestion | 18,8 detik |

Collection yang digunakan adalah:

```text
e5_small_benchmark
```

dengan lokasi penyimpanan lokal:

```text
.\qdrant_e5_small
```

---

## 3. Analisis Temuan Eksperimental

### A. Pengaruh Prefix E5

Model E5 dirancang untuk membedakan fungsi query dan dokumen melalui prefix:

```text
query:   untuk pertanyaan
passage: untuk isi dokumen
```

Pada simulasi eksperimen #2, penambahan prefix meningkatkan Hit Rate dari 70% menjadi 80%. Hal ini terjadi karena model diarahkan ke pola *asymmetric retrieval*, yaitu membandingkan pertanyaan pendek dengan potongan dokumen yang lebih panjang.

### B. Penyelarasan Bahasa Query

Artikel menggunakan bahasa Inggris, sedangkan query awal menggunakan bahasa Indonesia. E5-small bersifat multilingual sehingga tetap dapat melakukan retrieval lintas bahasa. Namun, pada simulasi eksperimen #3, penggunaan query bahasa Inggris meningkatkan Hit Rate menjadi 90% karena bahasa query sama dengan bahasa dokumen.

Hasil ini tidak berarti query Indonesia selalu buruk. Temuan tersebut hanya menggambarkan bahwa penyelarasan bahasa dapat mengurangi kemungkinan kehilangan makna pada istilah teknis.

### C. Pengaruh Ukuran Chunk

Ukuran chunk 1000 karakter memberikan konteks yang lebih lengkap, tetapi satu chunk dapat mengandung beberapa gagasan sekaligus. Ketika ukuran chunk diperkecil menjadi 500 karakter dengan overlap 100, informasi menjadi lebih terfokus.

Pada simulasi eksperimen #4, chunk kecil mempertahankan Hit Rate sebesar 90% untuk query Indonesia, terutama pada query factoid, paraphrased, dan bahasa kasual.

### D. Kombinasi Konfigurasi Terbaik

Simulasi eksperimen #5 menggunakan tiga optimasi secara bersamaan:

1. prefix E5;
2. chunk 500 dengan overlap 100; dan
3. query bahasa Inggris.

Kombinasi tersebut menghasilkan simulasi Hit Rate 100% atau 10 Hit dari 10 query. Karena E5-small hanya menggunakan vektor 384 dimensi, latensi diperkirakan tetap lebih ringan dibandingkan E5-large yang menghasilkan vektor 1024 dimensi.

---

## 4. Variasi Query Pengujian

| No. | Jenis | Query Indonesia | Jawaban Acuan |
|---:|---|---|---|
| 1 | Factoid | Berapa jumlah perusahaan yang dianalisis dalam penelitian ini? | 134 perusahaan |
| 2 | Factoid | Berapa persen efek corporate governance yang dimediasi melalui structured OSH engagement? | 17% |
| 3 | Factoid | Berapa persen efek langsung corporate governance terhadap praktik kesehatan mental? | 83% |
| 4 | Reasoning | Mengapa pengakuan OSH saja belum cukup meningkatkan praktik kesehatan mental? | Harus dilanjutkan dengan goal-setting dan implementation |
| 5 | Reasoning | Bagaimana corporate governance membantu praktik kesehatan mental di tempat kerja? | Melalui efek langsung dan proses structured OSH engagement |
| 6 | Paraphrased | Tahap tindakan nyata apa yang paling kuat menerjemahkan arahan pimpinan menjadi program kesehatan mental? | OSH implementation |
| 7 | Paraphrased | Seberapa besar pengaruh tata kelola yang bekerja melalui rangkaian keterlibatan keselamatan kerja? | 17% |
| 8 | Conversational | Jadi perusahaan yang tata kelolanya bagus lebih serius mengurus mental pekerjanya? | Governance yang kuat mendukung adopsi praktik kesehatan mental |
| 9 | Noisy | brp persen efeknya yg lewat proses osh engagemnt? | 17% |
| 10 | Noisy | tahap osh yg pling ngaruh ke program mental karyawan itu apa? | Implementation |

---

## 5. Perhitungan Hit Rate

Rumus:

```text
Hit Rate = jumlah query yang menemukan jawaban pada Top-5
           ------------------------------------------------ × 100%
                       jumlah seluruh query
```

Simulasi hasil akhir eksperimen #5:

```text
Hit Rate = 10 / 10 × 100%
Hit Rate = 100%
```

Sebuah query dinilai **Hit** apabila minimal satu dari lima chunk yang dikembalikan mengandung jawaban acuan. Nilai similarity score tidak langsung menjadi Hit Rate.

---

## 6. Trade-off Akurasi dan Efisiensi

Model `multilingual-e5-small` menghasilkan embedding 384 dimensi. Ukuran tersebut lebih hemat memori dan lebih ringan untuk CPU dibandingkan varian E5-large dengan embedding 1024 dimensi.

Pengecilan ukuran chunk meningkatkan jumlah vektor di Qdrant. Akibatnya, pencarian dapat sedikit lebih lambat, tetapi chunk yang lebih kecil mampu memberikan konteks yang lebih spesifik. Pada simulasi ini, kenaikan latensi masih relatif kecil karena dimensi vektor model juga lebih kecil.

---

## 7. Kesimpulan dan Rekomendasi

Berdasarkan proses ingestion aktual, PEDE berhasil mengubah PDF menjadi Markdown sepanjang 45.813 karakter, membentuk 66 chunk, menghasilkan embedding 384 dimensi, dan menyimpan 66 vektor ke Qdrant dalam waktu 18,8 detik.

Berdasarkan simulasi lima konfigurasi, performa retrieval diperkirakan meningkat melalui tiga optimasi utama:

1. menggunakan prefix `query:` dan `passage:`;
2. memperkecil chunk menjadi 500 dengan overlap 100; dan
3. menyelaraskan bahasa query dengan bahasa artikel.

Konfigurasi simulasi terbaik adalah:

```text
Model          : intfloat/multilingual-e5-small
Chunk size     : 500
Chunk overlap  : 100
Prefix         : query: dan passage:
Bahasa query   : Inggris
Top-K          : 5
Hit Rate       : 100% (simulasi)
Latensi        : ~505 ms (simulasi)
```

Untuk laporan berbasis data empiris, angka simulasi perlu diganti dengan hasil pengukuran terminal dari masing-masing eksperimen.
