# MEA Ecosystem — Arsitektur Router & Orkestrasi Model

**Dokumen acuan resmi untuk developer model MEA Ecosystem.**
Versi: 1.0 — Draft awal, akan berkembang seiring model baru ditambahkan.

---

## 1. Prinsip Dasar

1. **Router adalah satu-satunya titik orkestrasi.** Tidak ada model yang
   memanggil model lain secara langsung. Setiap model hanya berkomunikasi
   dengan Router (lewat backend), tidak pernah dengan model lain secara
   langsung.
2. **Setiap model punya SATU tanggung jawab spesifik.** Tidak boleh ada
   dua model yang menangani tugas yang sama (mencegah tabrakan/ambiguitas).
3. **Siesta adalah lapisan pembungkus akhir untuk komunikasi personal.**
   Siesta mengubah output teknis/faktual dari model lain menjadi respons
   natural dan berkepribadian sebelum sampai ke pengguna.
4. **Bahasa internal MEA Ecosystem adalah Bahasa Indonesia.** Semua model
   selain penerjemah (Arata dan turunannya) beroperasi dan "berpikir"
   dalam Bahasa Indonesia. Ini architectural decision yang disengaja
   (lihat Bagian 4).
5. **Aturan urutan bersifat deterministik dan dapat diperluas**, disimpan
   sebagai data terstruktur di backend — bukan hardcoded dalam logika
   kompleks, bukan pula dipelajari secara implisit oleh model generatif.
   Penambahan model baru = penambahan entri, bukan migrasi arsitektur.

---

## 2. Daftar Model MEA Ecosystem

### 2.1 Model yang sudah ada / sedang dikembangkan

| Nama | Tugas | Jenis Arsitektur | Status |
|---|---|---|---|
| **Siesta** | Kepribadian, percakapan, curhat, identitas, pengambilan keputusan personal | Decoder-only generatif | Fine-tune ronde 2 selesai (75M params) |
| **Arata** | Penerjemah Bahasa Indonesia ↔ Inggris | Model translasi (arsitektur TBD, kemungkinan encoder-decoder) | Direncanakan |
| **Reiko** | Peringkas teks/artikel panjang menjadi poin inti | Decoder-only atau encoder-decoder ringkasan | Direncanakan |
| **Sora** | Pustakawan — menjawab pertanyaan faktual/pengetahuan umum saat Siesta tidak tahu | Decoder-only, kemungkinan dengan retrieval (pgvector/semantic search) | Sedang dikembangkan |
| **Kaito** | Bantuan pemrograman/coding | Decoder-only generatif, fokus sempit ke kode | Direncanakan |
| **Formula** | Kalkulasi eksak matematika/kimia/fisika (bukan pengetahuan naratif) — dipanggil sebagai tool eksekusi kode (mea-formula), bukan model bahasa | Bukan model AI — library JS (CommonJS), dipanggil langsung sebagai fungsi | Domain `math` selesai (13 file, 196 unit test); `chemistry` dan `physics` belum digarap |
| **(nama TBD) — MEA Data** | Ekstraksi data terstruktur (JSON) dari dokumen panjang (mis. dokumen APBN/pemerintah) untuk transparansi publik | Ekstraktif, arsitektur TBD | Ide awal, belum mulai |

### 2.2 Model yang direncanakan (belum dikerjakan, dicatat untuk konsistensi penamaan & slot)

| Nama (placeholder) | Tugas | Catatan |
|---|---|---|
| **(TBD) — Vision** | Memahami/mendeskripsikan isi gambar | Realistis dengan fine-tune model vision kecil yang sudah ada, bukan dari nol (lihat Bagian 6) |
| **(TBD) — Image Generation** | Menghasilkan gambar dari deskripsi teks | Di luar kapasitas realistis untuk dilatih dari nol di GPU T4; kemungkinan besar akan menggunakan API pihak ketiga |
| **(TBD) — Video Generation** | Menghasilkan video dari deskripsi teks | Sama seperti Image Generation — realistisnya lewat API eksternal |
| **(TBD) — TTS (Text-to-Speech)** | Mengubah teks menjadi audio suara | Realistis dengan fine-tune model TTS kecil yang sudah ada |
| **(TBD) — STT (Speech-to-Text)** | Mengubah audio suara menjadi teks | Realistis dengan fine-tune model STT kecil yang sudah ada (mis. Whisper varian kecil) |

**Catatan penting:** Nama-nama placeholder di atas SENGAJA belum diberi
nama resmi. Ketika model-model ini mulai dikerjakan, nama resmi harus
didaftarkan di dokumen ini SEBELUM development dimulai, agar tidak terjadi
duplikasi nama atau konflik penamaan dengan model lain di ekosistem.

---

## 3. Arsitektur Router

### 3.1 Router BUKAN model generatif

Router adalah **classifier** (model klasifikasi multi-label kecil, atau
pada tahap awal cukup berupa logika rule-based), bukan model yang
"menulis" urutan pemanggilan model secara bebas. Keputusan ini diambil
secara sengaja:

- Urutan pemanggilan model bersifat **deterministik** — untuk kombinasi
  kebutuhan yang sama, urutannya selalu sama. Ini tidak memerlukan
  penalaran generatif.
- Menyimpan urutan sebagai **data/aturan terstruktur di backend** membuat
  sistem lebih mudah di-debug, diaudit, dan diperluas dibanding jika
  urutan "tersembunyi" di dalam bobot model generatif.
- Router versi classifier tetap bisa menggunakan model AI kecil (bukan
  murni if/else) ketika kebutuhan deteksi menjadi cukup kompleks
  (misalnya membedakan makna kalimat yang mirip tapi berbeda tujuan) —
  namun **outputnya tetap berupa label kebutuhan**, bukan urutan langsung.

### 3.2 Alur kerja Router

```
1. Pesan masuk dari user
        ↓
2. DETEKSI BAHASA (langkah terpisah, mendahului Router classifier;
   lihat Bagian 4 untuk detail penanganan multi-bahasa)
        ↓
3. ROUTER CLASSIFIER menerima teks (sudah/belum diterjemahkan, lihat
   Bagian 4) dan memprediksi SET label kebutuhan:
   contoh: {Sora: 0.91, Reiko: 0.12, Kaito: 0.03}
        ↓
4. Backend melakukan thresholding (mis. > 0.5) untuk menentukan label
   mana saja yang AKTIF
        ↓
5. Backend mencocokkan SET label aktif ke TABEL ATURAN URUTAN
   (Bagian 3.3) untuk mendapatkan urutan pemanggilan model yang final
        ↓
6. Backend mengeksekusi model satu per satu sesuai urutan, menyuntikkan
   hasil setiap tahap sebagai konteks untuk tahap berikutnya
        ↓
7. Siesta SELALU menjadi tahap akhir sebelum hasil dibungkus untuk
   dikirim ke user (lihat Bagian 3.1 poin 3, dan Bagian 4 untuk
   penanganan translate keluar)
```

### 3.3 Tabel Aturan Urutan (Inti — Tanpa Mempertimbangkan Bahasa)

Tabel ini berisi urutan model UNTUK PEMROSESAN INTI dalam Bahasa
Indonesia. Penanganan translate masuk/keluar dijelaskan terpisah di
Bagian 4 karena sifatnya membungkus (wrap), bukan bagian dari kombinasi
kebutuhan inti.

```python
URUTAN_RULES = {
    frozenset([]):                    ["Siesta"],
    frozenset(["Sora"]):              ["Sora", "Siesta"],
    frozenset(["Kaito"]):             ["Kaito", "Siesta"],
    frozenset(["Reiko"]):             ["Reiko", "Siesta"],
    frozenset(["Formula"]):           ["Formula", "Siesta"],
    frozenset(["Sora", "Reiko"]):     ["Sora", "Reiko", "Siesta"],
    frozenset(["Kaito", "Reiko"]):    ["Kaito", "Reiko", "Siesta"],
    frozenset(["Formula", "Reiko"]):  ["Formula", "Reiko", "Siesta"],
    # Kombinasi baru WAJIB ditambahkan di sini ketika model baru
    # diperkenalkan atau kombinasi kebutuhan baru teridentifikasi.
    # JANGAN biarkan kombinasi tidak terdaftar jatuh ke fallback produksi
    # tanpa peninjauan — lihat Bagian 3.4.
}
```

**Catatan penting soal Formula:** meskipun MEA Formula berkaitan dengan
"pengetahuan" (matematika/sains), Formula **TIDAK PERNAH** digabung
dengan `Sora` dalam kombinasi kebutuhan. Sora menangani pengetahuan
naratif (lewat semantic search ke MEA Library), sementara Formula adalah
tool eksekusi kode untuk kalkulasi eksak — keduanya menyelesaikan
masalah yang berbeda secara fundamental (lihat dokumen
`aturan-mea-library-formula.md` Bagian 1 dan 3 untuk detail). Router
harus memperlakukan keduanya sebagai kebutuhan yang terpisah, bukan
saling menggantikan.

### 3.4 Penanganan Kombinasi yang Belum Terdaftar

Jika Router mendeteksi kombinasi label yang belum ada di
`URUTAN_RULES`, sistem **tidak boleh gagal total**. Fallback wajib:

```python
def get_urutan_inti(labels_terdeteksi: set) -> list:
    key = frozenset(labels_terdeteksi)
    if key in URUTAN_RULES:
        return URUTAN_RULES[key]
    # Fallback: log kombinasi baru untuk ditinjau, proses tetap lanjut
    # dengan default aman (langsung ke Siesta)
    log_kombinasi_belum_terdaftar(key)  # untuk notifikasi/monitoring
    return ["Siesta"]
```

Kombinasi yang sering muncul di log fallback adalah sinyal bahwa tabel
aturan perlu diperbarui — ini bagian dari proses maintenance rutin,
bukan kegagalan sistem.

---

## 4. Penanganan Multi-Bahasa (Prinsip Krusial)

### 4.1 Mengapa ini penting

Siesta, Sora, Reiko, Kaito, dan mayoritas model MEA Ecosystem **berpikir
dan menghasilkan output dalam Bahasa Indonesia**. Ini adalah keputusan
arsitektur yang disengaja (model dilatih dari nol dengan fokus penuh pada
Bahasa Indonesia, bukan multibahasa). NAMUN, pengguna MEA Ecosystem
diasumsikan dapat berasal dari **berbagai negara dan berbahasa apa pun**,
bukan hanya Indonesia. Dokumen ini secara eksplisit menolak asumsi bahwa
seluruh pengguna berbahasa Indonesia.

### 4.2 Arata sebagai lapisan pembungkus (wrapper), bukan bagian dari kombinasi inti

Arata (dan model penerjemah bahasa lain di masa depan) beroperasi secara
**terpisah** dari logika kombinasi kebutuhan inti (Bagian 3.3). Arata
membungkus SELURUH proses, bukan menjadi salah satu opsi dalam
`URUTAN_RULES`.

```
Jika bahasa_user != "id":
    urutan_final = [Arata_masuk] + urutan_inti + [Arata_keluar]
Jika bahasa_user == "id":
    urutan_final = urutan_inti
```

**Alasan pemisahan ini:** jika translate dicampur ke dalam kombinasi
inti (mis. `frozenset(["Arata", "Reiko"])`), jumlah kombinasi yang harus
didaftarkan di `URUTAN_RULES` akan berkembang secara kombinatorial setiap
kali bahasa baru ditambahkan (lihat Bagian 4.4). Dengan memisahkan
translate sebagai wrapper, penambahan bahasa baru tidak memengaruhi
`URUTAN_RULES` sama sekali.

### 4.3 Contoh Alur Lengkap

**Kasus: pengguna berbahasa Inggris meminta ringkasan artikel**
```
User (EN): "Can you summarize this article for me?"
        ↓
Deteksi bahasa: EN
        ↓
Arata (EN → ID): menerjemahkan permintaan & artikel ke Bahasa Indonesia
        ↓
Router classifier (menerima teks berbahasa Indonesia): mendeteksi
kebutuhan {Reiko}
        ↓
Urutan inti: ["Reiko", "Siesta"]
        ↓
Reiko: meringkas artikel (dalam Bahasa Indonesia)
        ↓
Siesta: membungkus hasil ringkasan menjadi respons natural (Bahasa
Indonesia)
        ↓
Arata (ID → EN): menerjemahkan respons akhir Siesta ke Bahasa Inggris
        ↓
User menerima jawaban dalam Bahasa Inggris (bahasa asli mereka)
```

**Kasus: pengguna berbahasa Indonesia hanya ingin mengobrol (tanpa kebutuhan lain)**
```
User (ID): "Hai Siesta, lagi apa?"
        ↓
Deteksi bahasa: ID
        ↓
(Arata TIDAK dipanggil — bahasa sudah ID)
        ↓
Router classifier: tidak ada kebutuhan lain terdeteksi → frozenset([])
        ↓
Urutan final: ["Siesta"]
```

**Kasus: pengguna berbahasa Jepang bertanya hal faktual (mengasumsikan
model penerjemah Bahasa Jepang, mis. "Arata_JP", sudah ada)**
```
User (JP): 「今日の天気は？」
        ↓
Deteksi bahasa: JP
        ↓
Arata_JP (JP → ID): menerjemahkan ke Bahasa Indonesia
        ↓
Router classifier: mendeteksi kebutuhan {Sora}
        ↓
Urutan inti: ["Sora", "Siesta"]
        ↓
Sora: mencari/menjawab fakta (dalam Bahasa Indonesia)
        ↓
Siesta: membungkus jadi respons natural (Bahasa Indonesia)
        ↓
Arata_JP (ID → JP): menerjemahkan balik ke Bahasa Jepang
        ↓
User menerima jawaban dalam Bahasa Jepang
```

### 4.4 Skalabilitas Penambahan Bahasa Baru

Setiap bahasa baru yang didukung MEMBUTUHKAN model penerjemah khusus
(mengikuti keputusan bahwa setiap model translator bersifat satu-pasangan
dan selalu bertitik ke Bahasa Indonesia sebagai bahasa pusat — lihat
Bagian 4.5). Penambahan bahasa baru **tidak memerlukan perubahan** pada
`URUTAN_RULES` (Bagian 3.3) — hanya memerlukan:

1. Model penerjemah baru didaftarkan (mis. `Arata_JP`, `Arata_KO`,
   `Arata_VI`) di tabel model (Bagian 2).
2. Modul deteksi bahasa diperbarui untuk mengenali bahasa baru tersebut.
3. Pemetaan `bahasa → model_penerjemah` diperbarui di backend (bukan di
   Router classifier).

### 4.5 Prinsip "Bahasa Indonesia sebagai Pusat"

Sesuai arahan proyek: setiap model penerjemah baru SELALU merupakan
pasangan **Bahasa X ↔ Bahasa Indonesia**, bukan pasangan silang antar
bahasa asing lainnya (mis. tidak ada model Jepang ↔ Inggris langsung).
Jika suatu saat diperlukan penerjemahan antar dua bahasa asing, alurnya
tetap melalui Bahasa Indonesia sebagai perantara:

```
Bahasa Jepang → (Arata_JP) → Bahasa Indonesia → (Arata_EN) → Bahasa Inggris
```

Ini konsisten dengan keputusan bahwa Bahasa Indonesia adalah bahasa
internal seluruh ekosistem MEA, bukan hanya bahasa Siesta semata.

### 4.6 Konsekuensi untuk Reiko dan model lain — TIDAK ada asumsi konten berbahasa Indonesia

**Poin krusial yang harus dipahami setiap developer model:** karena
Arata membungkus di lapisan terluar, Reiko (dan model lain seperti Sora,
Kaito) **tidak pernah** menerima atau perlu memproses teks dalam bahasa
selain Indonesia — SELAMA arsitektur wrapper ini dipatuhi dengan benar.
Artinya:

- Reiko **tidak perlu** dilatih untuk meringkas artikel berbahasa
  Inggris, Jepang, dll secara langsung — cukup fokus penuh pada
  kemampuan meringkas Bahasa Indonesia, karena artikel berbahasa asing
  akan sudah diterjemahkan oleh Arata (atau model penerjemah lain)
  SEBELUM sampai ke Reiko.
- Ini berlaku sama untuk Sora, Kaito, dan model masa depan lainnya.
- **Kesalahan desain yang harus dihindari:** developer model manapun
  TIDAK BOLEH berasumsi mereka perlu menangani multi-bahasa sendiri.
  Tanggung jawab bahasa sepenuhnya berada di lapisan Arata (dan
  turunannya), bukan tanggung jawab model tugas (task model).

### 4.7 Di Mana Deteksi Bahasa Boleh/Harus Terjadi

**Backend WAJIB melakukan deteksi bahasa secara independen** — ini adalah
satu-satunya sumber kebenaran yang boleh dipercaya untuk menentukan alur
Arata. Alasan:
- Request bisa datang dari berbagai platform (web, mobile app, API
  langsung) — kalau deteksi hanya dilakukan di satu platform frontend,
  platform lain bisa tidak konsisten atau lupa mengimplementasikannya
- Client-side bisa dimodifikasi/dilewati (misal lewat DevTools atau
  panggilan API langsung) — backend **tidak boleh** mempercayai label
  bahasa yang dikirim dari frontend begitu saja tanpa validasi ulang

**Frontend BOLEH melakukan deteksi bahasa ringan sebagai tambahan UX**
(misal untuk menampilkan indikator visual real-time seperti "mendeteksi
bahasa..." mengikuti pola status di Bagian 5), tetapi ini murni untuk
pengalaman pengguna — bukan pengganti deteksi di backend. Prinsipnya:
**client-side untuk UX, server-side untuk keputusan final yang bisa
dipercaya.**

---

## 5. Status Realtime ke Frontend

Selama proses multi-tahap berjalan (mis. Arata → Reiko → Siesta →
Arata), frontend chat Siesta menampilkan indikator status agar
pengalaman menunggu tidak terasa kosong, mengikuti pola yang sudah
digunakan di `status.evanalmunawar.my.id` (Supabase Realtime).

Contoh status yang dikirim ke frontend pada tiap tahap:
- "Menerjemahkan pesan..." (saat Arata masuk berjalan)
- "Mencari informasi..." (saat Sora berjalan)
- "Meringkas artikel..." (saat Reiko berjalan)
- "Menyusun jawaban..." (saat Siesta berjalan)
- "Menerjemahkan balik..." (saat Arata keluar berjalan)

Ini murni event backend yang di-broadcast ke frontend — tidak
memerlukan model AI tambahan.

---

## 6. Catatan Realisme Teknis (GPU T4 / Kaggle-Colab)

Ringkasan tingkat realisme untuk masing-masing model, relevan untuk
perencanaan pengembangan:

| Model | Realistis dilatih dari nol di T4? | Catatan |
|---|---|---|
| Siesta | ✅ Terbukti (75M params, fine-tune ronde 2 selesai) | |
| Router (classifier) | ✅ Sangat realistis | Ukuran jauh lebih kecil dari Siesta (~2-5M params estimasi), arsitektur encoder ringan |
| Reiko | ✅ Realistis | Tugas ekstraktif/ringkasan, tidak perlu variasi gaya bahasa luas |
| Sora | ✅ Realistis (sedang dikembangkan) | Kompleksitas tambahan dari retrieval/pgvector, bukan dari ukuran model |
| Kaito | ✅ Realistis | Fokus sempit ke kode, tidak perlu kemampuan bahasa umum luas |
| Arata & turunan bahasa | ⚠️ Menengah | Kualitas translasi dari nol membutuhkan corpus paralel besar; realistis tapi kualitas awal mungkin terbatas |
| MEA Data (ekstraksi dokumen) | ✅ Realistis | Tugas ekstraktif terstruktur, model bisa relatif kecil |
| Vision | ⚠️ Sebaiknya fine-tune model yang sudah ada | Melatih dari nol butuh dataset gambar besar; tidak realistis dari nol di T4 |
| TTS | ⚠️ Sebaiknya fine-tune model yang sudah ada | Butuh dataset audio besar + preprocessing kompleks |
| STT | ⚠️ Sebaiknya fine-tune model yang sudah ada | Sama seperti TTS |
| Image Generation | ❌ Tidak realistis dari nol | Kemungkinan besar memerlukan API pihak ketiga |
| Video Generation | ❌ Tidak realistis dari nol | Sama seperti Image Generation, kebutuhan compute jauh melebihi T4 |

---

## 7. Checklist untuk Developer Model Baru

Sebelum memulai development model baru di MEA Ecosystem, developer WAJIB:

- [ ] Mendaftarkan nama model di Bagian 2 dokumen ini (hindari duplikasi)
- [ ] Menentukan dengan jelas SATU tanggung jawab spesifik model tersebut
      (tidak tumpang tindih dengan model lain yang sudah ada)
- [ ] Memastikan model beroperasi dalam Bahasa Indonesia sebagai bahasa
      internal (kecuali model tersebut memang model penerjemah)
- [ ] Menentukan apakah model perlu ditambahkan ke `URUTAN_RULES`
      (Bagian 3.3) dan kombinasi apa saja yang relevan
- [ ] TIDAK berasumsi menangani multi-bahasa sendiri — itu tanggung
      jawab lapisan Arata (Bagian 4.6)
- [ ] Meninjau Bagian 6 untuk realisme teknis di infrastruktur T4
      yang tersedia

---

## 8. Hal yang Masih Terbuka / Perlu Ditinjau Ke Depan

Dokumen ini adalah draft awal (v1.0). Beberapa hal berikut sengaja belum
difinalisasi dan perlu didiskusikan lebih lanjut seiring ekosistem
berkembang:

1. **Arsitektur pasti Router classifier** (ukuran model, jumlah layer,
   dataset training) — desain konseptual sudah disepakati (Bagian 3),
   implementasi teknis belum dimulai.
2. **Dataset training Router** — belum ada contoh data sama sekali,
   perlu ditulis dari nol (pasangan teks pendek → label kebutuhan).
3. **Modul deteksi bahasa** — prinsip lokasi sudah diputuskan (backend
   wajib, frontend opsional untuk UX — lihat Bagian 4.7), tetapi library
   atau model spesifik yang dipakai belum ditentukan.
4. **Nama resmi untuk MEA Data, Vision, TTS, STT, Image/Video
   Generation** — masih placeholder, perlu ditentukan sebelum development
   dimulai.
5. **Penanganan error/kegagalan di tengah rantai** (mis. Reiko gagal
   merespons di tengah proses) — belum dirancang mekanisme fallback atau
   retry.
6. **Threshold confidence Router classifier** — angka pasti (mis. >0.5)
   masih perkiraan awal, perlu divalidasi dengan data nyata setelah
   Router mulai dilatih.
