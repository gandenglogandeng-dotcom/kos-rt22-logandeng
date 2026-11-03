# Panduan pemasangan — Sistem Pendataan Kos (GitHub + Vercel)

Sistem ini **tidak lagi memakai Apps Script** untuk melayani halaman web. Semua halaman
(form daftar, poster QR, dashboard Pak RT) sekarang di-hosting langsung di domain
`vercel.app`, dan **tidak pernah mengarahkan pengunjung ke `script.google.com`** —
ini yang membuat masalah "tidak bisa dibuka" di sebagian HP Android Chrome tidak
akan muncul lagi, karena penyebabnya memang ada di domain Google itu sendiri.

Spreadsheet Anda **tetap jadi database**, tidak perlu dipindah, tidak perlu diubah
strukturnya. Bedanya, sekarang diakses lewat "Service Account" (akun robot milik
Google Cloud), bukan lewat akun pribadi Anda.

Satu paket kode ini dipasang **satu kali per RT** (sama seperti pola Apps Script
sebelumnya: satu deployment = satu RT = satu Spreadsheet).

---

## Bagian A — Siapkan akses ke Google (sekali per RT, ±15 menit)

### A.1 Buat project Google Cloud (boleh pakai 1 project untuk semua RT, atau pisah)

1. Buka **console.cloud.google.com**, login dengan akun Google yang akan jadi
   "pemilik teknis" sistem ini (idealnya akun resmi desa, bukan akun pribadi —
   lihat catatan di bagian Saran di akhir panduan ini).
2. Klik dropdown project di atas, **New Project**. Beri nama misalnya
   `kos-logandeng`. Klik **Create**.

### A.2 Aktifkan API yang dibutuhkan

1. Di kotak pencarian atas, ketik **"Google Sheets API"**, klik hasilnya, klik **Enable**.
2. Ulangi untuk **"Google Drive API"**.

### A.3 Buat Service Account

1. Buka menu **IAM & Admin > Service Accounts**.
2. Klik **Create Service Account**.
3. Nama: misalnya `kos-rt21`. Klik **Create and Continue**, lalu **Done**
   (langkah izin/role boleh dilewati, tidak diperlukan).
4. Klik service account yang baru dibuat, buka tab **Keys**.
5. Klik **Add Key > Create new key**, pilih **JSON**, klik **Create**.
6. Sebuah file `.json` otomatis terunduh ke komputer Anda. **Simpan baik-baik,
   file ini seperti kata sandi** — jangan diunggah ke tempat umum atau GitHub.
7. Buka file JSON itu dengan Notepad. Anda akan butuh dua nilai dari dalamnya:
   - `client_email` → nanti jadi `GOOGLE_SERVICE_ACCOUNT_EMAIL`
   - `private_key` → nanti jadi `GOOGLE_PRIVATE_KEY`

### A.4 Bagikan Spreadsheet ke Service Account

1. Buka Spreadsheet RT yang mau disambungkan (yang sekarang berisi sheet
   `Daftar_Kos` dan `Data_Penghuni`).
2. Klik **Share**, tempel `client_email` dari langkah A.3, beri akses **Editor**,
   klik **Send** (boleh un-check "Notify people").
3. Ambil **Spreadsheet ID** dari alamat browser:
   ```
   https://docs.google.com/spreadsheets/d/ISI_INI_SPREADSHEET_ID/edit
   ```

### A.5 Buat folder Drive untuk dokumen

1. Buka **drive.google.com** (login dengan akun Google **pribadi/resmi Anda**, akun
   manusia biasa — bukan akun robot Service Account).
2. Buat folder baru, misalnya `KTP Kos RT 21`.
3. Buka folder itu, ambil **Folder ID** dari alamat browser:
   ```
   https://drive.google.com/drive/folders/ISI_INI_FOLDER_ID
   ```

**Folder ini TIDAK perlu dibagikan ke Service Account.** Penjelasannya ada di A.6.

### A.6 Kenapa perlu langkah OAuth tambahan untuk Drive (baca dulu sebelum lanjut)

Service Account (akun robot) **tidak diberi jatah penyimpanan sama sekali oleh
Google** — ini aturan Google sendiri untuk semua akun Gmail pribadi, bukan
kesalahan pengaturan. Kalau tetap dipaksa mengunggah file, akan selalu muncul
error `storageQuotaExceeded`, walaupun foldernya sudah dibagikan sebagai Editor.

Jalan keluarnya: khusus untuk **mengunggah KTP/Nikah/KK**, sistem memakai
**akun Google asli Anda** lewat OAuth2 (bukan Service Account), supaya file
tersimpan dan terhitung ke jatah 15 GB akun itu. Spreadsheet tetap memakai
Service Account seperti biasa — itu tidak bermasalah.

### A.7 Buat OAuth Client ID

1. Masih di **Google Cloud Console**, buka **APIs & Services > Credentials**.
2. Kalau belum pernah mengatur **OAuth consent screen**, klik dulu menu
   **OAuth consent screen** di sidebar, pilih **External**, isi nama aplikasi
   bebas (misalnya `Kos RT 21`), email Anda di kolom yang diminta, **Save and
   Continue** terus sampai selesai (boleh lewati bagian Scopes dan Test users).
3. Kembali ke **Credentials**, klik **+ Create Credentials > OAuth client ID**.
4. **Application type**: pilih **Desktop app**.
5. Nama bebas, misalnya `Kos RT 21 Desktop`. Klik **Create**.
6. Muncul jendela berisi **Client ID** dan **Client Secret** — salin keduanya,
   simpan di catatan sementara.

### A.8 Ambil Refresh Token (sekali per RT, pakai OAuth Playground)

1. Buka **developers.google.com/oauthplayground**.
2. Klik ikon ⚙️ (gear) di pojok kanan atas.
3. Centang **"Use your own OAuth credentials"**.
4. Tempel **Client ID** dan **Client Secret** dari langkah A.7 ke kotak yang muncul.
5. Tutup jendela pengaturan itu.
6. Di kolom kiri ("Step 1 — Select & authorize APIs"), ada kotak kosong di
   paling bawah bertuliskan *"Input your own scopes"*. Ketik:
   ```
   https://www.googleapis.com/auth/drive
   ```
   Klik **Authorize APIs**.
7. Anda akan diarahkan login Google — **pilih akun yang sama dengan pemilik
   folder di langkah A.5** (akun manusia asli Anda, bukan Service Account).
8. Klik **Allow** saat diminta izin akses Drive.
9. Anda kembali ke OAuth Playground, sekarang di **Step 2**, klik tombol
   **Exchange authorization code for tokens**.
10. Akan muncul **Refresh token** di sisi kanan — salin nilainya.

Simpan 6 nilai ini sebelum lanjut ke Bagian C:
`client_email`, `private_key`, `SPREADSHEET_ID`, `DRIVE_FOLDER_ID`,
`Client ID` + `Client Secret` (dari A.7), `Refresh token` (dari A.8).

---

## Bagian B — Unggah kode ke GitHub

### B.1 Kalau belum punya akun GitHub

Daftar gratis di **github.com**.

### B.2 Buat repository baru

1. Klik **+** di kanan atas > **New repository**.
2. Nama: misalnya `kos-rt21-logandeng`. Pilih **Private** (supaya kode tidak
   terlihat publik — walau tidak ada kredensial di dalam kode, lebih aman).
3. Klik **Create repository**.

### B.3 Unggah folder project ini

Cara termudah tanpa command line:

1. Di halaman repository yang baru, klik **uploading an existing file**.
2. Seret **semua isi folder** `kos-vercel` (bukan folder itu sendiri — isinya:
   `api/`, `lib/`, `public/`, `package.json`, `vercel.json`, `.gitignore`, dll.)
   ke kotak upload.
3. Scroll ke bawah, klik **Commit changes**.

---

## Bagian C — Deploy ke Vercel lewat GitHub

### C.1 Import repository

1. Buka **vercel.com/new**.
2. Klik **Import Git Repository**, hubungkan akun GitHub Anda kalau diminta.
3. Pilih repository `kos-rt21-logandeng` tadi, klik **Import**.

### C.2 Isi Environment Variables SEBELUM klik Deploy

Di halaman konfigurasi sebelum deploy, buka bagian **Environment Variables**,
isi satu per satu (Key lalu Value, klik **Add** tiap selesai satu):

| Key | Value |
|---|---|
| `GOOGLE_SERVICE_ACCOUNT_EMAIL` | dari file JSON, field `client_email` |
| `GOOGLE_PRIVATE_KEY` | dari file JSON, field `private_key` (salin **apa adanya**, termasuk `-----BEGIN PRIVATE KEY-----` dan `\n` di dalamnya) |
| `SPREADSHEET_ID` | dari langkah A.4 |
| `GOOGLE_OAUTH_CLIENT_ID` | dari langkah A.7 |
| `GOOGLE_OAUTH_CLIENT_SECRET` | dari langkah A.7 |
| `GOOGLE_OAUTH_REFRESH_TOKEN` | dari langkah A.8 |
| `DRIVE_FOLDER_ID` | dari langkah A.5 |
| `RT_TOKEN` | buat sendiri: ketik bebas minimal 24 karakter acak, atau pakai generator kata sandi online |
| `NAMA_DESA` | contoh: `Desa Logandeng RT 21` |

### C.3 Deploy

Klik **Deploy**, tunggu sekitar 30-60 detik.

Setelah selesai, Anda dapat alamat seperti:
```
https://kos-rt21-logandeng.vercel.app
```

---

## Bagian D — Isi data kos & uji coba

### D.1 Isi sheet Daftar_Kos

Kolom yang dibaca sistem baru ini **hanya A sampai F**:
`Kode Kos | Nama Kos | Nama Pengelola | No. WA | Alamat | Jumlah Kamar`

Kolom G/H/I (Link QR, Link Poster, Link Pendek) dari versi Apps Script lama
**tidak lagi dipakai** dan boleh dibiarkan/dihapus — poster sekarang dibuat
otomatis oleh halaman `poster.html`, bukan dari rumus di sheet.

### D.2 Tiga link penting Anda

Ganti `kos-rt21-logandeng.vercel.app` dan `KOS01` sesuai milik Anda:

- **Form penghuni (untuk QR):** `https://kos-rt21-logandeng.vercel.app/?kos=KOS01`
- **Poster siap cetak:** `https://kos-rt21-logandeng.vercel.app/poster.html?kos=KOS01`
- **Dashboard Pak RT:** `https://kos-rt21-logandeng.vercel.app/rt.html?k=ISI_RT_TOKEN_ANDA`

### D.3 Uji coba

1. Buka link **poster**, pastikan QR muncul dan tidak ada pesan error merah.
2. Cetak atau buka link **form** langsung, coba daftar dengan data tes.
3. Buka **dashboard Pak RT**, pastikan data tes tadi muncul, coba tombol **Lihat KTP**.
4. Hapus data tes dari sheet `Data_Penghuni` dan file tesnya dari folder Drive.
5. **Coba khusus di HP yang tadinya selalu gagal** — ini pengujian paling penting.

---

## Perbedaan dari versi Apps Script (ringkas)

- **Ukuran file maksimal turun dari 3 MB menjadi 2 MB** per dokumen. Ini batasan
  teknis dari Vercel (ukuran maksimal satu request), bukan pilihan bebas — kalau
  dilanggar, request akan ditolak Vercel sebelum sempat diproses kode ini.
- **Tidak perlu lagi Bit.ly maupun pengaturan Vercel-redirect terpisah** — seluruh
  situs memang sudah di Vercel sejak awal, jadi masalah yang coba diperbaiki kedua
  cara itu sudah tidak ada sama sekali di arsitektur baru ini.
- **Satu set kode untuk semua halaman** (bukan 4 file terpisah yang harus selalu
  diperbarui bersamaan seperti `Code.gs`/`Index`/`RT`/`Poster` dulu) — jadi risiko
  "lupa update salah satu file" yang kemarin menyebabkan RT 22 error, sekarang
  tidak mungkin terjadi lagi karena memang hanya ada satu sumber kode per RT,
  dan GitHub+Vercel men-deploy semuanya sekaligus setiap kali ada perubahan.
- **Dashboard Pak RT boleh dibuka tanpa harus tahu token berulang kali** — token
  cukup ada di link yang dikirimkan sekali ke Pak RT.

## Transisi aman dari sistem lama

QR dan link Apps Script lama (`script.google.com/.../exec?kos=...`) **tetap
berfungsi seperti biasa** selama Anda belum menggantinya. Jadi Anda bisa:

1. Pasang sistem baru ini untuk satu RT dulu sebagai percobaan.
2. Cetak ulang poster barunya, tempel di satu-dua kos dulu.
3. Pastikan berjalan lancar beberapa hari.
4. Baru ganti semua poster di RT itu, lalu lanjut ke RT berikutnya.

Kedua sistem membaca Spreadsheet yang sama, jadi data tidak akan pecah dua tempat
selama Anda memakai `SPREADSHEET_ID` yang sama persis dengan Spreadsheet RT
yang sudah berjalan.
