# README — Arsitektur Siesta v5

Dokumentasi teknis arsitektur model Siesta v5, bagian dari MEA Ecosystem. Dokumen ini mencakup spesifikasi model, tokenizer, corpus pretrain, proses training, dan keputusan desain beserta alasannya — disusun untuk keperluan dokumentasi proyek dan dapat menjadi bahan rujukan akademik (skripsi/riset).

---

## 1. Ringkasan Proyek

Siesta adalah model bahasa percakapan (conversational language model) berbasis arsitektur decoder-only Transformer, dikembangkan sebagai bagian dari MEA Ecosystem — sebuah sistem multi-model di mana setiap model memiliki spesialisasi domain masing-masing (Siesta untuk percakapan empatik, Sora untuk knowledge/pustakawan, Arata untuk penerjemahan, dan model lain yang akan bertambah).

Prinsip perancangan Siesta mengikuti filosofi **pertumbuhan bertahap seperti manusia**: setiap generasi model (diukur dari jumlah parameter) menambah kapasitas baru tanpa menghapus kapasitas generasi sebelumnya, dilatih di atas kombinasi seluruh skema data dari generasi awal hingga generasi terbaru.

---

## 2. Riwayat Generasi Model

| Generasi | Parameter | Skema data yang diperkenalkan | Karakteristik baru |
|---|---|---|---|
| v1 | ~8M | Skema 1 | Percakapan single-turn dasar |
| v2 | ~21M | Skema 2 | Percakapan multi-turn tanpa metadata |
| v3 | ~75M | Skema 3 | Percakapan multi-turn dengan metadata sesi |
| v4 | ~75M (fine-tune lanjutan) | — | Perluasan cakupan kategori topik (identitas, sekolah, keluarga, dll) |
| **v5** | **~303M** | **Skema 4–9** | **Proses berpikir eksplisit (persepsi, emosi, tujuan, keputusan) dan kemampuan memanggil bantuan eksternal (router)** |

Setiap generasi baru dilatih ulang dari nol pada tahap pretrain (karena perubahan arsitektur/ukuran parameter tidak memungkinkan transfer bobot langsung), tetapi tahap fine-tune selalu menggunakan gabungan seluruh skema data dari generasi sebelumnya ditambah skema baru — bukan menggantikannya.

---

## 3. Arsitektur Model (v5)

### 3.1 Spesifikasi

| Parameter | Nilai |
|---|---|
| Tipe arsitektur | Decoder-only Transformer (gaya GPT) |
| Total parameter | 302.651.392 (~303M) |
| Vocabulary size | 24.000 |
| Embedding dimension | 1024 |
| Jumlah layer | 20 |
| Jumlah attention head | 16 |
| Feed-forward dimension | 4096 |
| Context length | 1536 token |
| Dropout | 0.1 |
| Positional encoding | Learned positional embedding (`nn.Embedding`), bukan sinusoidal |
| Normalisasi | Pre-LayerNorm (LayerNorm sebelum sub-layer attention dan feed-forward) |
| Fungsi aktivasi | GELU |
| Attention masking | Causal mask (lower-triangular) via `torch.tril` |

### 3.2 Struktur Blok Transformer

Setiap `TransformerBlock` terdiri dari:
1. `LayerNorm` → `MultiHeadAttention` (dengan causal mask) → residual connection
2. `LayerNorm` → `FeedForward` (Linear → GELU → Linear) → residual connection

Implementasi menggunakan PyTorch native (`torch.nn`), tanpa dependensi library eksternal seperti HuggingFace Transformers — seluruh arsitektur ditulis dari nol (`model.py`).

### 3.3 Estimasi Parameter

Formula kasar yang digunakan untuk estimasi jumlah parameter sebelum implementasi:

```
total_params = (vocab_size × embed_dim) × 2          # token embedding + output head
              + (context_length × embed_dim)          # positional embedding
              + n_layers × (4 × embed_dim²             # attention (Q,K,V,output projections)
                             + 2 × embed_dim × ff_dim)  # feed-forward
```

### 3.4 Perbandingan dengan Generasi Sebelumnya

| | v4 (75M) | v5 (303M) |
|---|---|---|
| vocab_size | 12.000 | 24.000 |
| embed_dim | 640 | 1024 |
| n_layers | 12 | 20 |
| n_heads | 8 | 16 |
| ff_dim | 2560 | 4096 |
| context_length | 1024 | 1536 |

---

## 4. Tokenizer

| Parameter | Nilai |
|---|---|
| Algoritma | Byte-Pair Encoding (BPE) |
| Vocabulary size | 24.000 |
| Library | HuggingFace `tokenizers` |
| Pre-tokenizer | Whitespace |
| Minimum frequency | 2 |

Tokenizer v5 dilatih ulang dari nol (tidak mewarisi vocabulary dari v4) menggunakan corpus gabungan (lihat bagian 5). Token spesial yang didefinisikan:

```
[PAD], [UNK],
<sesi>, <topik>, <waktu>, <ringkasan>, <tanya>, <jawab>, <selesai>,
<persepsi>, <emosi>, <tujuan>, <keputusan>, <butuh_bantuan>,
<jawab_awal>, <jawab_akhir>
```

Seluruh token spesial di atas terverifikasi ter-tokenisasi sebagai satu unit utuh (tidak terpecah menjadi subword) berdasarkan pengujian encoding pada tahap pengembangan.

---

## 5. Corpus Pretrain

Corpus pretrain disusun dari tiga sumber untuk mencakup bahasa Indonesia formal, informal, dan kontekstual:

| Sumber | Jumlah entri | Kontribusi |
|---|---|---|
| Wikipedia Bahasa Indonesia (dibersihkan) | 638.797 artikel | Konteks kalimat natural, topik luas, struktur bahasa formal |
| KBBI-V (Kamus Besar Bahasa Indonesia edisi V) | 146.278 entri | Kosakata baku dan definisi resmi |
| Kamus slang & bahasa gaul Indonesia | 2.027 entri | Bahasa informal/sehari-hari |

**Format penggabungan:**
- Wikipedia: `{judul artikel}. {isi artikel}`
- KBBI: `{kata}: {definisi}`
- Slang: `{kata slang}: {kata baku}`

**Statistik corpus akhir:**
- Ukuran file: 986,51 MB
- Total baris: 787.102
- Total token (setelah tokenisasi): 249.937.721
- Total chunk training (sliding window sepanjang context_length): 162.719

---

## 6. Proses Pretraining

### 6.1 Konfigurasi Training

| Parameter | Nilai |
|---|---|
| Optimizer | AdamW (weight_decay=0.01) |
| Learning rate | 5×10⁻⁴ |
| LR scheduler | Warmup linear (1000 step) → cosine decay |
| Batch size | 1 |
| Gradient accumulation steps | 8 (effective batch size = 8) |
| Mixed precision | Automatic Mixed Precision (AMP), `torch.amp.autocast` |
| Gradient clipping | max_norm = 1.0 |
| Epoch (maksimum) | 10 (162.719 step/epoch, total 1.627.190 step) |
| Early stopping | Patience 6 kali cek berturut-turut tanpa perbaikan, min_delta 0.01 |
| Checkpoint interval | Setiap 2000 step |

### 6.2 Fungsi Loss

Cross-entropy loss dihitung pada **seluruh token** dalam sequence (tanpa masking), karena tahap pretrain bertujuan mempelajari distribusi bahasa umum, bukan format tanya-jawab. Ini berbeda dengan tahap fine-tune, di mana loss di-masking hanya pada bagian `<jawab>...<selesai>`.

### 6.3 Infrastruktur

Training dilakukan pada GPU NVIDIA T4 (16 GB VRAM) melalui platform Kaggle Notebooks, dengan kuota mingguan terbatas (~30 jam GPU/minggu). Karena batasan ini, training dilakukan secara bertahap lintas sesi (checkpoint-resume) selama beberapa minggu.

**Kendala teknis yang ditemui dan solusinya:**
- **Out-of-memory (OOM)** pada context_length=2048 dengan batch_size=8 → context_length diturunkan ke 1536, batch_size diturunkan ke 1 dengan gradient accumulation.
- **Instabilitas saat resume training** (loss melonjak setelah checkpoint dimuat ulang dengan reset optimizer state) → ditemukan bahwa training paling stabil ketika dilakukan secara linear tanpa intervensi berulang (reset LR, reset optimizer) di tengah proses; strategi akhir yang digunakan adalah menjaga learning rate konstan (5×10⁻⁴) sepanjang training dan hanya melakukan resume standar (`resume_optimizer=True`) antar sesi.
- **Bug pada penghitungan loss saat resume** — variabel akumulator loss tidak di-reset dengan benar relatif terhadap titik resume, menyebabkan nilai loss yang ditampilkan pada log tidak akurat pada sesi-sesi awal resume. Bug ini telah diperbaiki dengan menambahkan penghitung independen (`steps_processed_this_epoch`) yang selalu dimulai dari nol pada setiap sesi training, terlepas dari step absolut saat resume.

### 6.4 Ketahanan Sistem (Checkpoint & Notifikasi)

- Checkpoint disimpan secara lokal dan otomatis diunggah (auto-push) ke dataset penyimpanan setiap 2000 step, termasuk bobot model, state optimizer, dan state scheduler — memungkinkan pemulihan penuh (full resume) antar sesi.
- Checkpoint dengan loss terbaik disimpan terpisah (`_BEST.pt`) untuk memastikan model dengan performa tertinggi selalu tersedia, terlepas dari checkpoint step terakhir.
- Notifikasi progres (step, loss, estimasi waktu selesai) dikirim otomatis ke Telegram pada setiap checkpoint, memungkinkan pemantauan tanpa harus mengawasi sesi secara langsung.

---

## 7. Skema Data Fine-tune

Fine-tuning menggunakan sembilan skema data yang dirancang untuk merepresentasikan kondisi nyata percakapan dalam sistem produksi. Dokumentasi lengkap struktur dan contoh setiap skema tersedia di dokumen terpisah: **README_Skema_Siesta_v5.md**.

Ringkasan konsep utama:

1. **Prinsip aditif** — skema baru (4–9) tidak menggantikan skema lama (1–3); keduanya digabungkan dalam satu sesi fine-tuning.
2. **Proses berpikir eksplisit** — skema 4 ke atas memperkenalkan tahap `<persepsi>`, `<emosi>`, `<tujuan>`, `<keputusan>` sebelum `<jawab>`, memungkinkan model menghasilkan representasi "alasan" yang dapat diperiksa (interpretable) sebelum menghasilkan jawaban akhir — dapat disembunyikan dari antarmuka pengguna namun tetap tercatat untuk keperluan debug/analisis.
3. **Delegasi tugas (`<butuh_bantuan>`)** — model tidak memanggil sub-sistem secara langsung, melainkan mendelegasikan ke komponen terpisah bernama **Router**, yang bertanggung jawab menentukan apakah permintaan diteruskan ke model internal MEA Ecosystem atau ke API eksternal.
4. **Redundansi terancang** — skema tanpa riwayat percakapan (4, 7) sengaja dipertahankan sebagai mekanisme fallback bila backend gagal menyuntikkan riwayat atau metadata sesi.

---

## 8. Keputusan Desain dan Justifikasi

| Keputusan | Alasan |
|---|---|
| Decoder-only, bukan encoder-decoder | Tugas utama Siesta adalah generasi teks percakapan (bukan transformasi teks seperti penerjemahan), sehingga arsitektur decoder-only konsisten dengan pendekatan model bahasa generatif seperti keluarga GPT |
| Label emosi dalam Bahasa Indonesia | Label berbahasa Inggris (mis. "sadness") ditemukan terpecah menjadi beberapa subword oleh tokenizer, mengurangi efisiensi representasi; label Bahasa Indonesia konsisten dengan keseluruhan corpus |
| Proses berpikir sebagai token yang di-generate (visible reasoning), bukan representasi tersembunyi | Memungkinkan proses debug dan analisis tanpa mengubah arsitektur inti (tetap decoder-only murni); dapat disembunyikan dari pengguna akhir pada lapisan aplikasi |
| Delegasi via Router, bukan pemanggilan model spesifik langsung | Menjaga Siesta tetap sederhana secara arsitektural dan tidak memerlukan retraining setiap kali ekosistem menambah model/API baru |
| Model spesialis (bukan general-purpose) | Cakupan tugas yang lebih sempit (percakapan empatik Bahasa Indonesia) memungkinkan parameter yang tersedia dialokasikan lebih efisien dibanding model general-purpose yang harus mencakup domain pengetahuan luas |

---

## 9. Status Pengembangan (per dokumen ini ditulis)

- Pretraining v5: sedang berlangsung, loss telah turun dari ~10,2 (awal) ke kisaran ~3,3 pada step ~76.000 dari maksimum 1.627.190 step, menunjukkan tanda-tanda mendekati titik konvergensi (plateau).
- Dataset fine-tune Skema 1–3: 17.437 sample, telah digunakan pada generasi v1–v4.
- Dataset fine-tune Skema 4–9: baru tersedia dalam bentuk contoh acuan (3 sample per skema), penulisan dataset skala penuh belum dimulai.
- Rencana pengembangan lanjutan (generasi v6): eksplorasi arsitektur berbasis JAX dengan optimizer Adafactor untuk pemanfaatan TPU, ditargetkan pada skala parameter lebih besar (>1B), dijalankan secara paralel dengan siklus fine-tuning GPU pada model produksi yang sudah ada.
