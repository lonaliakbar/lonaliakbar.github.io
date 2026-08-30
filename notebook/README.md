# Delpher Scraper & Preservasi Internet Archive

Notebook Google Colab untuk mengunduh arsip buku digital dari [Delpher](https://www.delpher.nl) — pustaka digital nasional Belanda berisi buku, surat kabar, dan majalah kolonial bersejarah — lalu mengarsipkannya secara otomatis ke [Internet Archive](https://archive.org).

Dibuat untuk mendukung digitalisasi arsip sejarah kolonial Aceh dan Nusantara, sebagai bagian dari upaya preservasi digital [Basa Aceh Project](https://github.com).

## Fitur

- Pencarian otomatis buku di Delpher berdasarkan daftar kata kunci (mis. `atjeh, pidie, gajo`)
- Ekstraksi otomatis: judul, identifier, teks OCR (`.txt`), berkas PDF, dan gambar tiap halaman (`.jpg`)
- Dashboard real-time bergaya Swiss (hitam-putih) di dalam notebook: progres, statistik, log sistem
- Sistem resume otomatis — melanjutkan proses dari titik terakhir jika sesi terputus
- Anti-duplikasi — buku yang sudah diunduh tidak diproses ulang
- Upload otomatis ke Internet Archive setiap satu buku selesai diproses
- Penyimpanan hasil ke Google Drive dengan pemantauan kapasitas

## Struktur Repo

```
├── Delpher_Scraper_dan_Preservasi_Internet_Archive.ipynb   # Notebook utama
├── docs/
│   └── Dokumentasi Teknis - Delpher Scraper.pdf            # Dokumentasi lengkap
├── requirements.txt
├── LICENSE
└── README.md
```

## Cara Pakai

1. Buka notebook ini di [Google Colab](https://colab.research.google.com/).
2. **Siapkan kredensial Internet Archive di Colab Secrets** (ikon 🔑 di sidebar kiri Colab) dengan nama:
   - `IA_ACCESS_KEY`
   - `IA_SECRET_KEY`

   Dapatkan key dari https://archive.org/account/s3.php — **jangan pernah** menuliskan key ini langsung di kode.
3. Jalankan **Cell 1** sekali di awal sesi untuk memasang dependensi dan Google Chrome.
4. Jalankan **Cell 2**, masukkan kata kunci pencarian di kolom yang tersedia, lalu tekan **▶ MULAI SCRAPING**.
5. **Cell 3** bersifat opsional — hanya dijalankan bila ada folder lama yang belum sempat ter-upload ke Internet Archive (backfill).

Hasil unduhan tersimpan di Google Drive pada folder `MyDrive/COK DELPHER 3`, dengan satu subfolder per buku (`DLP-{nomor}-{judul}`) berisi PDF, teks OCR, gambar halaman, dan metadata.

Dokumentasi teknis lengkap (arsitektur, alur kerja, penanganan error, riwayat bug) ada di [`docs/Dokumentasi Teknis - Delpher Scraper.pdf`](docs/Dokumentasi%20Teknis%20-%20Delpher%20Scraper.pdf).

## Dependensi

Lihat [`requirements.txt`](requirements.txt). Notebook juga memerlukan Google Chrome (dipasang otomatis oleh Cell 1) untuk browsing via Selenium.

## Catatan Etika & Legalitas

- Periksa syarat & ketentuan (Terms of Service) Delpher terkait scraping otomatis dan distribusi ulang konten sebelum menggunakan notebook ini secara luas.
- Metadata upload di kode ini menandai lisensi CC-BY 4.0 secara default — verifikasi status hak cipta/domain publik setiap dokumen sebelum diunggah ke Internet Archive.
- Pertimbangkan menambahkan jeda (rate limiting) yang lebih manusiawi antar-request agar tidak membebani server Delpher.

## Lisensi

Kode dalam repo ini dirilis di bawah lisensi MIT (lihat [LICENSE](LICENSE)). Ini tidak berlaku untuk konten yang diunduh dari Delpher maupun diunggah ke Internet Archive, yang tunduk pada lisensi/hak ciptanya masing-masing.
