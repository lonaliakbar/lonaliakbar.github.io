# Dokumentasi Teknis: Kamus Bahsa Acèh (Web App)

**Versi dokumen:** 1.0
**Tanggal:** 31 Agustus 2026
**Jenis proyek:** Aplikasi web kamus digital komunitas (Bahasa Acèh ⇄ Bahasa Indonesia)

---

## 1. Ringkasan Proyek

Kamus Bahsa Acèh adalah aplikasi web satu-halaman (single-page app) untuk mencari, menambah, mengubah, dan menghapus entri kamus Bahasa Acèh–Indonesia secara kolaboratif. Aplikasi ini dibangun tanpa framework front-end (HTML/CSS/JavaScript murni) dan menggunakan **Google Apps Script (GAS)** sebagai backend, dengan **Google Sheets** sebagai basis data.

### 1.1 Tujuan
- Mendigitalkan dan mempublikasikan kamus Bahasa Acèh secara terbuka dan mudah dicari.
- Memungkinkan banyak kontributor menambah/mengedit kata secara kolaboratif (mirip wiki), dengan jejak riwayat perubahan.
- Menyediakan profil kontributor agar setiap orang bisa melihat kontribusinya sendiri.

### 1.2 Komponen Utama
| Komponen | Teknologi | Peran |
|---|---|---|
| Front-end | HTML + CSS + Vanilla JavaScript (1 file) | Antarmuka pengguna, semua interaksi |
| Back-end | Google Apps Script (`doGet`/`doPost`) | API, logika bisnis, otentikasi |
| Basis data | Google Sheets (3 sheet: `Kamus`, `Users`, `Riwayat`) | Penyimpanan data |
| Komunikasi | REST-like via `fetch()`, JSON | Front-end ⇄ Back-end |

---

## 2. Arsitektur Sistem

```
┌─────────────────────┐        HTTPS (fetch/JSON)        ┌──────────────────────────┐
│   Front-end (HTML)   │ ───────────────────────────────▶ │  Google Apps Script       │
│   - Tab Kamus         │ ◀─────────────────────────────── │  (doGet / doPost)         │
│   - Tab Tambah Kata   │                                   │                            │
│   - Tab Riwayat       │                                   │  ┌──────────────────────┐  │
│   - Tab Akun          │                                   │  │  Google Sheets        │  │
│   - Modal Edit        │                                   │  │  - Kamus              │  │
└─────────────────────┘                                   │  │  - Users              │  │
     localStorage                                           │  │  - Riwayat            │  │
     (menyimpan sesi login: username + password polos)      │  └──────────────────────┘  │
                                                              └──────────────────────────┘
```

Tidak ada server aplikasi terpisah — semua logika berjalan di Google Apps Script yang di-deploy sebagai **Web App** (`script.google.com/macros/s/.../exec`), diakses langsung dari browser klien.

---

## 3. Struktur Data (Google Sheets)

### 3.1 Sheet `Kamus`

Header yang **sebenarnya ada di data produksi saat ini** (55 kolom):

```
1  kata_aceh
2  tipe_kata
3  arti_indonesia
4  contoh1_aceh      5  contoh1_indo
6  contoh2_aceh      7  contoh2_indo
8  contoh3_aceh      9  contoh3_indo
10 contoh4_aceh     11  contoh4_indo
12 contoh5_aceh     13  contoh5_indo
14 contoh6_aceh     15  contoh6_indo
16 contoh7_aceh     17  contoh7_indo
18 rujukan
19 sumber
20 contoh9_aceh     21  contoh9_indo   ← contoh8 TIDAK ADA (lompat dari 7 ke 9)
22 contoh10_aceh … dst sampai …
52 contoh25_aceh    53  contoh25_indo
54 rujukan          55  sumber        ← DUPLIKAT kolom 18–19
```

> ⚠️ Susunan ini **berbeda** dari susunan default yang dibuat otomatis oleh fungsi `getSheet()` di backend saat sheet belum ada (lihat §6.1). Ini adalah sumber utama dari sejumlah bug — lihat §8.

Setiap baris = satu entri kata. Kolom `contohN_aceh` / `contohN_indo` bersifat dinamis (bisa sampai 25 pasang), ditambahkan otomatis ke akhir sheet saat dibutuhkan.

### 3.2 Sheet `Users`

| Kolom | Isi |
|---|---|
| `username` | Nama pengguna (unik, tidak case-sensitive saat dicek) |
| `password` | **Disimpan sebagai teks polos (plaintext), tidak di-hash** |
| `created_at` | Timestamp ISO saat registrasi |

Data pengguna contoh saat ini: `admin`, `teuku`, `cut`, `editor`, `guru`.

### 3.3 Sheet `Riwayat`

| Kolom | Isi |
|---|---|
| `timestamp` | Waktu aksi (ISO string) |
| `username` | Siapa yang melakukan aksi |
| `aksi` | `TAMBAH` / `UBAH` / `HAPUS` |
| `kata_aceh` | Kata yang terpengaruh |
| `detail` | Kolom ke-5, ada di data tapi **tidak pernah diisi oleh backend saat ini** (`catatRiwayat` hanya menulis 4 kolom) |

---

## 4. Front-end: Struktur Antarmuka

Antarmuka terbagi menjadi 4 tab utama + 1 modal:

### 4.1 Tab "Kamus" (default aktif)
- **List View**: kotak pencarian (`#searchInput`) dengan debounce 300ms → memanggil `loadKamus(query)`.
  - Pencarian mencocokkan substring pada `kata_aceh` ATAU `arti_indonesia` (case-insensitive), dilakukan **di sisi server** (GAS).
  - Hasil ditampilkan sebagai kartu (`entry-card`): kata, jenis kata, arti singkat.
- **Detail View**: muncul saat kartu diklik (`showDetail(index)`).
  - Menampilkan kata, jenis kata, arti lengkap, daftar contoh kalimat (loop `contoh1`, `contoh2`, … berhenti di nomor pertama yang kosong — lihat bug §8.3), rujukan (sebagai "chip" yang bisa diklik untuk mencari kata rujukan), dan sumber.
  - Tombol aksi: "📜 Riwayat Kata" (selalu tampil), "✏️ Ubah" dan "🗑️ Hapus" (hanya jika sudah login).

### 4.2 Tab "+ Tambah Kata"
Form dengan field:
- Kata Acèh (wajib)
- Jenis Kata: dropdown (`n`, `v`, `adj`, `adv`, `lainnya`)
- Arti Indonesia (wajib, textarea)
- Contoh Kalimat & Terjemahan: baris dinamis (tombol "+ Tambah Contoh Kalimat"), tiap baris punya input Acèh + Indonesia + tombol hapus baris
- Rujukan Kata Lain (opsional, dipisah koma)
- Sumber/Referensi: dropdown pilihan (Pusat Bahasa / A.G. Abdul Gani / Tutur Lisan / Lainnya dengan input bebas)

Submit memanggil `handleTambahKata()` → menolak jika belum login → mengirim `action: "add"` ke API beserta `auth_user`/`auth_pass`.

### 4.3 Tab "Riwayat"
Menampilkan **seluruh** riwayat perubahan (semua pengguna, semua kata) dalam bentuk tabel: Waktu, Pengguna, Aksi, Kata. Diambil lewat `action: "get_riwayat"`, diurutkan dari yang terbaru (dibalik di sisi klien).

### 4.4 Tab "Akun"
Dua kondisi:
- **Belum login**: toggle antara form Masuk dan form Daftar.
- **Sudah login**: profil kontributor —
  - 2 kartu statistik: jumlah kata ditambah, jumlah kali mengedit
  - Tabel "Kata Apa Saja yang Anda Tambahkan"
  - Tabel "Aktivitas Terakhir Anda" (dengan badge warna: hijau=TAMBAH, kuning=UBAH, merah=HAPUS)

### 4.5 Modal "Ubah Kata"
Terbuka dari tombol "✏️ Ubah" di detail view (`openEditModal(index)`). Field sama seperti form tambah, tapi terisi otomatis dari data kata yang dipilih, termasuk logika pencocokan `tipe_kata` dan `sumber` ke opsi dropdown yang sesuai (fallback ke "Lainnya" jika tidak cocok).

---

## 5. Manajemen State & Sesi

- State disimpan di objek JS global `state = { auth, currentData, searchTimer }`.
- **Sesi login disimpan di `localStorage`** key `kamus_auth`, berisi `{ username, password }` — **password ikut tersimpan di penyimpanan browser dalam bentuk polos.**
- `state.currentData` menyimpan hasil pencarian terakhir agar detail view bisa diakses tanpa fetch ulang (diakses via index array).
- Tidak ada token sesi/JWT — setiap aksi yang butuh otentikasi (tambah/ubah/hapus) mengirim ulang `username` + `password` mentah ke server pada tiap request.

---

## 6. Backend: Google Apps Script

### 6.1 Routing

```
doGet(e)
 ├─ action=get_riwayat        → getRiwayatData()
 ├─ action=get_user_profile   → getUserProfileData(username)
 └─ (default) q=<query>       → getKamusData(query)

doPost(e)   [body JSON]
 ├─ action=login               → handleLogin()          [tanpa auth]
 ├─ action=register            → handleRegister()        [tanpa auth]
 ├─ (lainnya) → authenticateUser() dulu, baru:
 │    ├─ action=add            → handleAddKata()
 │    ├─ action=edit           → handleEditKata()
 │    └─ action=delete         → handleDeleteKata()
```

### 6.2 Auto-provisioning Sheet
Fungsi `getSheet(sheetName)` akan **membuat sheet baru otomatis** kalau belum ada, dengan header default:
- `Kamus`: `kata_aceh, tipe_kata, arti_indonesia, rujukan, sumber, pembuat, contoh1_aceh, contoh1_indo` (8 kolom)
- `Users`: `username, password, created_at`
- `Riwayat`: `timestamp, username, aksi, kata_aceh`

> Header default ini **tidak sama** dengan header 55-kolom yang sekarang benar-benar dipakai di data produksi (lihat §3.1) — sheet `Kamus` production kemungkinan pernah direstrukturisasi manual setelah dibuat, tanpa backend disesuaikan.

### 6.3 Fungsi Utama

| Fungsi | Tugas |
|---|---|
| `getKamusData(query)` | Cari & kembalikan entri kamus yang cocok, sebagai array of objects (key = header kolom) |
| `getRiwayatData()` | Kembalikan seluruh baris `Riwayat`, urutan terbalik (terbaru dulu) |
| `getUserProfileData(username)` | Hitung `total_added`/`total_edited` dari sheet `Riwayat`, plus fallback membaca sheet `Kamus` kolom `pembuat` (kolom index 5, hardcoded) untuk kata lama sebelum sistem riwayat ada |
| `authenticateUser(username, password)` | Cocokkan username+password (case-insensitive username, case-sensitive password) secara plaintext |
| `handleLogin` / `handleRegister` | Wrapper autentikasi/pendaftaran |
| `handleAddKata(data)` | Tulis baris baru; `rowData[3]`=rujukan, `rowData[4]`=sumber, `rowData[5]`=pembuat (**hardcoded index, lihat §8**); kolom `contohN_*` dicocokkan via `headers.indexOf`, kolom baru dibuat otomatis jika perlu |
| `handleEditKata(data)` | Cari baris berdasar `original_kata` (exact match, case-insensitive), timpa kolom 1–5 secara hardcoded index, kosongkan semua kolom `contoh*` lalu isi ulang dari data baru |
| `handleDeleteKata(data)` | Hapus baris pertama yang `kata_aceh`-nya cocok (exact match) |
| `catatRiwayat(username, aksi, kataAceh)` | Tambah baris log ke `Riwayat` (4 kolom saja) |
| `createJsonResponse(data)` | Bungkus response jadi `ContentService` JSON |

---

## 7. Alur Data End-to-End (Contoh: Menambah Kata)

1. Pengguna login → `handleLogin()` di front-end → POST `{action:"login", username, password}` → backend `authenticateUser()` cek ke sheet `Users` → sukses → front-end simpan `{username,password}` ke `state.auth` & `localStorage`.
2. Pengguna isi form "Tambah Kata" → submit → `handleTambahKata()` mengumpulkan field + `auth_user`/`auth_pass` + hingga 25 pasang `contohN_aceh`/`contohN_indo`.
3. POST ke `API_URL` dengan `action:"add"`.
4. Backend `doPost` → `authenticateUser()` ulang (server tidak percaya token, selalu verifikasi ulang tiap request) → jika sukses → `handleAddKata(data)`.
5. `handleAddKata` menulis baris baru ke sheet `Kamus`, lalu memanggil `catatRiwayat(user,"TAMBAH",kata)` → baris baru masuk ke sheet `Riwayat`.
6. Response `{status:"success"}` dikirim balik → front-end reset form, reload daftar kamus (`loadKamus("")`).

---

## 8. Masalah / Bug yang Ditemukan

### 8.1 🔴 Kritis — Password disimpan & dibandingkan plaintext
- Sheet `Users` menyimpan password tanpa hashing.
- `authenticateUser()` membandingkan string password mentah.
- Password juga ikut disimpan mentah di `localStorage` klien dan dikirim ulang di **setiap** request tambah/ubah/hapus (bukan hanya saat login).
- **Dampak:** siapa pun dengan akses ke Sheet, ke log Apps Script, atau ke `localStorage` korban bisa melihat password asli semua pengguna.

### 8.2 🔴 Kritis — Ketidakcocokan posisi kolom (hardcoded index vs header aktual)
Backend menulis `rujukan`/`sumber`/`pembuat` ke **posisi kolom tetap** (`getRange(row, 4)`, `(row, 5)`, `rowData[5]`), dengan asumsi urutan header lama (`rujukan` di kolom 4, `sumber` di kolom 5, `pembuat` di kolom 6). Namun header sheet produksi saat ini punya `contoh1_aceh`/`contoh1_indo` di kolom 4–5, dan tidak punya kolom `pembuat` sama sekali (kolom 6 aktualnya `contoh2_aceh`).

**Akibat konkret:**
- Setiap **edit kata**, nilai `rujukan` dan `sumber` yang diinput pengguna akan **menimpa isi `contoh1_aceh` dan `contoh1_indo`** kata tersebut — contoh kalimat pertama hilang tanpa peringatan.
- Setiap **tambah kata baru**, nama pengguna (`auth_user`) tertulis ke kolom `contoh2_aceh`, bukan ke kolom "pembuat" — sehingga fitur "kata yang saya tambahkan" (fallback dari sheet `Kamus`) tidak pernah bekerja dengan benar.
- Nilai `rujukan`/`sumber` yang benar-benar dimaksud pengguna sebenarnya **tidak pernah tersimpan** di kolom 18–19 (posisi header `rujukan`/`sumber` yang asli) — kolom itu jadi mati/tidak terisi oleh alur normal.

### 8.3 🟠 Sedang — Kolom `contoh8` hilang memutus rantai tampilan contoh kalimat
Header sheet melompat dari `contoh7_*` langsung ke `rujukan`/`sumber` lalu `contoh9_*`. Karena front-end (`showDetail`) menampilkan contoh kalimat dengan **loop berurutan yang berhenti begitu satu nomor tidak ditemukan** (`while (item["contoh"+exIndex+"_aceh"])`), maka begitu `contoh8_aceh` tidak ada, loop berhenti — **semua contoh kalimat nomor 9 sampai 25 tidak akan pernah ditampilkan**, walaupun datanya mungkin ada di sheet.

### 8.4 🟠 Sedang — Kolom `rujukan`/`sumber` duplikat
Header muncul dua kali: kolom 18–19 dan kolom 54–55. Karena kode backend mencari kolom via `headers.indexOf("...")` untuk field `contoh*` (dinamis) tapi **tidak** untuk `rujukan`/`sumber` (hardcoded index tetap seperti di §8.2), duplikasi ini murni menjadi sampah data — tidak ada bagian kode yang secara sengaja menulis ke kolom 54–55.

### 8.5 🟡 Rendah — Penghapusan kata berdasarkan nama, bukan ID unik
`handleDeleteKata` mencocokkan `kata_aceh` secara exact-match dan menghapus **baris pertama** yang cocok. Untuk homonim (kata sama, arti beda — pola yang disebutkan dipakai di proses digitalisasi kamus ini) ini berisiko menghapus entri yang salah.

### 8.6 🟡 Rendah — Tidak ada validasi input di sisi server
Endpoint API bisa dipanggil langsung (di luar form HTML) dengan field kosong; server tidak menolaknya.

### 8.7 🟡 Rendah — Kolom `detail` di sheet `Riwayat` tidak pernah diisi
Sheet punya kolom ke-5 (`detail`) yang berisi teks deskriptif di data historis (mis. "Menambahkan kata baru"), tapi fungsi `catatRiwayat()` di backend saat ini hanya menulis 4 kolom — kolom `detail` untuk entri baru akan selalu kosong.

### 8.8 🟢 Catatan performa
`getUserProfileData` membaca ulang **seluruh** sheet `Riwayat` dan `Kamus` setiap kali profil dibuka — cukup untuk skala saat ini, tapi bisa melambat jika data tumbuh besar (ribuan baris).

---

## 9. Ringkasan Rekomendasi Perbaikan

| Prioritas | Masalah | Rekomendasi |
|---|---|---|
| Tinggi | Password plaintext | Hash password (mis. SHA-256 + salt) sebelum simpan/bandingkan; jangan kirim ulang password di setiap request — gunakan token sesi sederhana |
| Tinggi | Kolom hardcoded vs header aktual | Ubah semua akses kolom (`rujukan`, `sumber`, `pembuat`) memakai `headers.indexOf("nama_kolom")`, sama seperti perlakuan `contoh*` |
| Tinggi | `contoh8` hilang | Tambahkan kolom `contoh8_aceh`/`contoh8_indo` yang hilang, atau ubah logika tampilan agar tidak berhenti di nomor kosong pertama (loop sampai `contoh25` selalu) |
| Sedang | Kolom duplikat `rujukan`/`sumber` | Hapus kolom duplikat di kolom 54–55 secara manual setelah backend diperbaiki |
| Sedang | Hapus berdasar nama saja | Gunakan ID baris/ID unik untuk operasi edit & hapus, bukan pencocokan `kata_aceh` |
| Rendah | Validasi input server | Tambahkan pengecekan field wajib di backend, bukan hanya `required` di HTML |

---

## 10. Ringkasan Teknologi

- **Front-end:** HTML5, CSS3 (custom properties/variabel warna), JavaScript vanilla (tanpa framework/library eksternal)
- **Back-end:** Google Apps Script (`doGet`, `doPost`, `SpreadsheetApp`, `ContentService`)
- **Basis data:** Google Sheets (3 sheet kerja)
- **Autentikasi:** Username/password sederhana, tanpa hashing, tanpa token sesi (dikirim ulang tiap request)
- **Penyimpanan sesi klien:** `localStorage`
- **Pola komunikasi:** REST-like via `fetch()`, format JSON, `Content-Type: text/plain` untuk POST (workaround umum agar tidak kena preflight CORS di Apps Script)
