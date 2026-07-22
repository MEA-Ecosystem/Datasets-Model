# Aturan Arsitektur — MEA Library & MEA Formula

Catatan keputusan supaya tidak lupa alasan dan posisi kedua komponen ini
dalam MEA Ecosystem. Pelengkap dari `mea-ecosystem-router-architecture.md`.

---

## 1. Posisi dalam Arsitektur — Dua Jalur Berbeda

**MEA Library dan MEA Formula BUKAN model/tool yang sama, dan tidak
diakses lewat jalur yang sama:**

```
Router mendeteksi kebutuhan:
        ↓
"Butuh pengetahuan naratif" (sejarah/budaya/mitologi)
        → Sora (retrieval semantic search ke MEA Library / MEA Library 0.1)

"Butuh kalkulasi matematis/sains" (turunan/integral/statistik/dst)
        → LANGSUNG panggil MEA Formula sebagai tool eksekusi kode,
          TIDAK lewat Sora. Backend eksekusi fungsi JS-nya, hasil
          {result, error, meta} disuntik ke Siesta untuk dibungkus.
```

**Alasan pemisahan:** MEA Formula adalah kalkulasi eksak yang harus
dieksekusi sebagai kode (presisi, tidak boleh halusinasi angka), bukan
pengetahuan naratif yang perlu "dipahami maknanya" oleh model bahasa.
Menyerahkan kalkulasi ke Sora (model bahasa) berisiko sama seperti
masalah yang dihindari di MEA Data — model bisa salah/berhalusinasi
angka. MEA Formula harus dipanggil sebagai fungsi, bukan "dijelaskan"
oleh model.

**Konsekuensi untuk Router:** perlu label/kategori terpisah untuk MEA
Formula (di luar label `Sora`), karena ini bukan bagian dari tugas Sora.

---

## 2. MEA Library — Ada DUA Sumber, Keduanya Diakses Sora

| Sumber | Isi | Cara diperoleh |
|---|---|---|
| **MEA Library** | Sejarah, budaya, mitologi — kurasi manual | Ditulis manual (dibantu Claude), disusun per kategori/periode |
| **MEA Library 0.1** | Hasil crawl otomatis (dikelola oleh Hany) | Crawling, sudah punya embedding vector |

Keduanya menjadi **source module terpisah** di MEA Study (pola sama
seperti `hany.js`), sama-sama diakses Sora lewat semantic search. Sora
tidak perlu tahu detail internal masing-masing source — hanya perlu
kontrak yang sama: kirim query, terima hasil relevan.

---

## 3. Kapan Butuh Vector Embedding, Kapan Tidak

Ini prinsip inti yang membedakan MEA Library dan MEA Formula secara
fundamental — **jenis "pencarian" yang dibutuhkan berbeda total**:

| | MEA Library | MEA Formula |
|---|---|---|
| Masalah yang diselesaikan | Cari makna yang mirip (user bisa memakai kata berbeda dari yang ada di entry) | Cari fungsi yang tepat berdasarkan jenis operasi (kategori terbatas & terstruktur) |
| Solusi teknis | Vector embedding + cosine similarity | Classifier kecil (mirip Router) — TIDAK butuh vector database |
| Kenapa | Narasi bebas, banyak cara berbeda untuk "bilang hal yang sama" | Nama fungsi sudah unik/spesifik (`derivative`, `solveQuadratic`, dst), tidak ambigu secara makna |
| Analogi | Mencari buku di perpustakaan berdasarkan topik | Memilih tombol kalkulator yang tepat |

**Jangan generate embedding untuk MEA Formula** — itu pemborosan kerja
tanpa manfaat, karena masalahnya bukan soal makna yang mirip.

---

## 4. Pipeline Embedding untuk MEA Library (Kurasi Manual)

### 4.1 Tidak perlu generate ulang FILE JSON

Vector embedding **TIDAK disuntik ke dalam file JSON**. File JSON tetap
menjadi source of truth untuk konten (judul, ringkasan, konten, tags) —
tidak diubah strukturnya sama sekali.

**Alasan:**
- Vector embedding (ratusan-ribuan angka desimal) akan membengkakkan
  file yang seharusnya tetap ramping (5–15 KB sesuai aturan ukuran file
  yang sudah ditetapkan)
- Kalau embedding menempel di file, update teks (misal perbaiki typo)
  berisiko lupa sinkron ulang embedding-nya
- Melanggar prinsip yang sudah ditetapkan sendiri: repo harus fleksibel,
  tidak terikat satu cara akses

### 4.2 Solusi: Tabel terpisah, terhubung lewat `id`

```sql
CREATE TABLE mea_library_embeddings (
    id TEXT PRIMARY KEY,       -- sama persis dengan field "id" di JSON
    embedding VECTOR(384),     -- pgvector, dimensi sesuai model yang dipakai
    source_file TEXT,          -- opsional, untuk tracing
    updated_at TIMESTAMP DEFAULT now()
);
```

### 4.3 Alur kerja

1. File JSON tetap seperti sekarang, tidak diubah.
2. Script terpisah (mis. `generate_embeddings.js`) membaca semua file
   JSON, untuk tiap entry mengambil `judul + ringkasan + tags` sebagai
   teks sumber, generate vector, lalu insert/update ke tabel embedding.
3. Saat Sora butuh mencari: generate embedding dari query user → cari
   kemiripan (cosine similarity) di tabel embedding → dapat daftar `id`
   yang relevan → ambil isi lengkap dari file JSON (atau
   `index/master-index.json`) berdasarkan `id` tersebut.

### 4.4 Kapan generate ulang embedding dibutuhkan

**Hanya saat:**
- Ada entry baru ditambahkan → generate embedding untuk entry itu saja
- Entry lama diedit (judul/ringkasan/tags berubah) → generate ulang
  embedding entry itu saja, update baris di tabel

**Tidak pernah perlu generate ulang SEMUA entry**, kecuali model
embedding yang dipakai diganti/upgrade di masa depan.

### 4.5 Model embedding tidak perlu dilatih dari nol

Berbeda dari Siesta/Router yang dibangun dari nol, model embedding untuk
retrieval ini sebaiknya memakai model open-source yang sudah ada (yang
mendukung Bahasa Indonesia) — "membuat representasi vektor dari teks"
adalah tugas general yang tidak membutuhkan spesialisasi khusus
MEA Ecosystem.

---

## 5. MEA Formula — Dua Tantangan Berbeda (Bukan Satu)

Begitu MEA Formula dipanggil, ada **dua** langkah yang perlu diselesaikan
secara terpisah — jangan dianggap sebagai satu masalah:

### 5.1 Klasifikasi kategori/fungsi

Menentukan fungsi mana dari puluhan fungsi (`derivative`, `solveQuadratic`,
`mean`, dst di seluruh modul `math/`) yang cocok dengan permintaan user.
Ini bisa didekati dengan classifier kecil (mirip arsitektur Router yang
sudah dibangun), atau rule-based sederhana pada tahap awal.

### 5.2 Ekstraksi parameter

Setelah tahu fungsi yang tepat, masih perlu mengekstrak nilai/parameter
dari kalimat natural user (misal dari "turunan dari x kuadrat di titik
3" perlu diekstrak: `fn = x => x*x`, `x = 3`). Ini adalah tantangan
**ekstraksi terstruktur**, mirip dengan pendekatan yang direncanakan
untuk MEA Data (ekstraksi JSON dari dokumen panjang) — bukan sekadar
klasifikasi kategori.

**Kedua langkah ini belum didesain detail** — dicatat sebagai pekerjaan
lanjutan, bukan diselesaikan sekarang.

---

## 6. Status MEA Formula per Domain

**Catatan penting:** README asli `mea-formula` saat ini mencantumkan 3
domain (math, chemistry, physics), namun cakupan sebenarnya yang
direncanakan Kudo lebih luas dari itu — README perlu diperbarui untuk
mencerminkan domain tambahan di luar 3 itu (belum dirinci detailnya,
dicatat di Bagian 7 sebagai hal yang belum diputuskan).

| Domain | Status |
|---|---|
| `math` | Selesai — 13 file, 196 unit test pass |
| `chemistry` | **Sedang dikerjakan** (per 23 Juli 2026) |
| `physics` | Belum digarap |
| *(domain lain di luar 3 ini)* | Direncanakan, README belum mencerminkan cakupan penuh |

**Implikasi untuk dataset Router:** dataset `router_dataset_formula.jsonl`
saat ini hanya berisi kalkulasi matematika, karena itu satu-satunya
domain yang sudah selesai — ini sudah tepat untuk kondisi sekarang.
**Namun begitu domain baru (chemistry, physics, dst) selesai
diimplementasikan di `mea-formula`, dataset Router WAJIB ditambah
contoh baru untuk domain tersebut** (mis. "hitung molaritas larutan
ini" → label `Formula`, bukan `Sora`), lalu training ulang. Tanpa
pembaruan ini, Router akan terus salah mengarahkan permintaan domain
baru itu ke Sora karena itulah pola yang dipelajarinya dari dataset
lama.

Semua fungsi publik mengikuti format return standar
`{ result, error, meta }` — dirancang secara sengaja agar predictable
untuk tool-calling model (model bisa selalu cek `error` dulu sebelum
memakai `result`, tanpa menebak-nebak makna `NaN`/`undefined`).

---

## 7. Hal yang Belum Diputuskan (dari sumber README asli, dicatat ulang)

- Mekanisme konkret Sora/MEA Study fetch dari repo MEA Library (GitHub
  raw langsung vs sync ke Supabase/R2)
- Detail pipeline generate embedding MEA Library — siapa yang generate,
  disimpan di mana, model embedding spesifik yang dipakai
- Desain classifier + parser ekstraksi parameter untuk MEA Formula
  (Bagian 5) — baru sebatas identifikasi kebutuhan, belum ada rancangan
  teknis
- Label resmi di `URUTAN_RULES` Router untuk MEA Formula sudah
  ditentukan: **"Formula"** (lihat `mea-ecosystem-router-architecture.md`
  Bagian 2.1 dan 3.3, sudah diimplementasikan di dataset Router v3)
- **README `mea-formula` perlu diperbarui** — cakupan domain yang
  direncanakan lebih luas dari 3 domain (math/chemistry/physics) yang
  tercantum saat ini; domain tambahan belum dirinci
- **Rutinitas sinkronisasi dataset Router per domain baru MEA Formula**
  belum menjadi proses formal — perlu jadi kebiasaan: setiap domain baru
  selesai (chemistry, physics, dst), dataset
  `router_dataset_formula.jsonl` harus ditambah contoh baru untuk domain
  itu sebelum training ulang Router
