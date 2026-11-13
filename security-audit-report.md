# 🛡️ Laporan Audit Keamanan (Security Audit Report — Putaran ke-2)
**Aplikasi**: Sistem Informasi Pondok Pesantren Miftahul Huda Pusat  
**Target Audit**: Backend (Laravel 11 API), Frontend (Vue 3 / Vite), Apache Config (`.htaccess`), & Storage  
**Tanggal Audit**: 12 November 2026  
**Klasifikasi**: Internal Confidential / Defensive Security Review  

---

## 📌 Ringkasan Status Keamanan

Audit keamanan mendalam putaran ke-2 telah dilakukan terhadap keseluruhan basis kode per **12 November 2026**.  
Seluruh **11 temuan pada Audit Tahap 1** (11 Nov 2026) terverifikasi **tetap aman dan terproteksi** (pengecualian user/admin RBAC, ekspor finansial, import Excel, penghapusan `clear-cache.php`, BOLA `KomunikasiController`, `InternalApiMiddleware` fail-closed, rate limiter login/monitoring, dan `config/cors.php`).

Namun, inspeksi lebih dalam terhadap seluruh modul controller, file `.htaccess`, konfigurasi token Sanctum, serta bundel frontend menemukan **sejumlah kerentanan baru yang sangat penting untuk segera ditangani**:

| Tingkat Keparahan | Jumlah | Ringkasan Temuan Baru (12 Nov 2026) | Status |
|---|:---:|---|:---:|
| 🔴 **CRITICAL** | 3 | 1. **Hardcoded `X-API-KEY` di Frontend JS** membobol proteksi `internal.api` (ekspose PII santri)<br>2. **Arbitrary File Deletion** pada `InstansiLampiranController::delete`<br>3. **Unvalidated File Upload** pada `ModulAjarController` ke disk `public` | ⚠️ Perlu Perbaikan |
| 🟠 **HIGH** | 4 | 1. **Missing RBAC (`role:...`)** pada rute `settings`, `kesehatan`, `izin`, `keuangan` (mutasi/syahriah), `nilai/kelas`<br>2. **Sanctum Token Tidak Dicabut** saat user dinonaktifkan / reset password & token tanpa masa kedaluwarsa<br>3. **Override Wildcard CORS (`*`) + Credentials** di `backend/public/.htaccess`<br>4. **Kebocoran PII & Riwayat Pelanggaran Santri** pada endpoint publik `/api/dashboard` & `/api/monitoring/realtime` | ⚠️ Perlu Perbaikan |
| 🟡 **MEDIUM** | 2 | 1. **Error/Stack Detail Leakage** (`$e->getMessage()`, `getFile()`, `getLine()`) di JSON error response<br>2. **Filename menggunakan `getClientOriginalExtension()`** pada `AbsenGuruController` | ⚠️ Jadwalkan |
| 🔵 **LOW / PROD** | 2 | 1. `APP_DEBUG=true` & `APP_ENV=local` pada konfigurasi saat ini<br>2. Berkas duplikat `backend/public/(1).htaccess` | ℹ️ Pra-Produksi |

---

## 🔍 Rincian Temuan Audit Baru (12 November 2026)

---

### 1. 🔴 [CRITICAL] Hardcoded `X-API-KEY` di Kode Frontend Publik (Bypass Autentikasi Santri)
- **Berkas Terkait**:
  - [`frontend/src/services/api.js`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/frontend/src/services/api.js#L99)
  - [`frontend/src/services/transaksi.js`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/frontend/src/services/transaksi.js#L18)
  - [`backend/routes/api.php`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/routes/api.php#L528-L531)
- **Mekanisme Kerentanan**:
  Kunci rahasia API internal ditanam (*hardcoded*) langsung di kode klien JavaScript:
  ```js
  config.headers["X-API-KEY"] = `mh_yadi_xxx`;
  ```
  Karena kode Vite/Vue dikirimkan ke browser setiap pengguna, siapa pun dapat mengekstrak kunci ini dari hasil build JS dan langsung memanggil endpoint:
  - `GET /api/santri-extern` (dipetakan ke `SantriController::index` yang mengembalikan daftar santri beserta data pribadi orang tua, NIK, nomor telepon, alamat)
  - `GET /api/santri-extern/{id}` & `GET /api/santri/{nis}/nis`
  tanpa perlu login akun apa pun!
- **Rekomendasi Remediasi**:
  1. **Hapus** header `X-API-KEY` statis dari `frontend/src/services/api.js` dan `transaksi.js` (frontend sudah menggunakan Bearer Token `Authorization: Bearer ...` milik Sanctum; `X-API-KEY` seharusnya hanya dipakai antar-server / backend-to-backend).
  2. **Rotasi** nilai `API_KEY` di `.env` server dengan string acak baru yang hanya disimpan di sisi server.
  3. Batasi endpoint `santri-extern` hanya pada metode *read* yang dibutuhkan integrasi eksternal (bukan `apiResource` penuh) serta saring field sensitif jika perlu.

---

### 2. 🔴 [CRITICAL] Arbitrary File Deletion pada `InstansiLampiranController::delete`
- **Berkas Terkait**: [`InstansiLampiranController.php`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/app/Http/Controllers/Api/InstansiLampiranController.php#L43-L56) & [`backend/routes/api.php`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/routes/api.php#L174)
- **Mekanisme Kerentanan**:
  ```php
  $validated = $request->validate([
      'path' => 'required|string',
  ]);
  if (Storage::disk('public')->exists($validated['path'])) {
      Storage::disk('public')->delete($validated['path']);
  }
  ```
  Rute `DELETE /api/instansi/lampiran/delete` tidak memeriksa awalan direktori maupun path traversal (`..`). User login mana pun dapat mengirim `path: "santri/foto_xyz.jpg"` atau `"syahriah/bukti/..."` dan menghapus foto santri, bukti pembayaran syahriah, atau berkas SIM HP milik unit lain.
- **Rekomendasi Remediasi**:
  Validasi agar `$path` tidak mengandung `..`, dinormalisasi, wajib berada di bawah prefix `instansi/`, dan hanya boleh dijalankan oleh peran yang berwenang.

---

### 3. 🔴 [CRITICAL] Upload File Tanpa Validasi Tipe/Ukuran pada `ModulAjarController`
- **Berkas Terkait**: [`ModulAjarController.php`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/app/Http/Controllers/Api/ModulAjarController.php#L22-L76)
- **Mekanisme Kerentanan**:
  Pada metode `store()` dan `update()`:
  ```php
  $validated = $request->validate([
      'nama' => 'required',
      'deskripsi' => 'nullable',
      'file' => 'nullable',
  ]);
  ...
  $validated['file'] = $request->file('file')->store('modul-ajar', 'public');
  ```
  Aturan `'file' => 'nullable'` tidak memiliki `file`, `max`, maupun `mimes`. User login dapat mengunggah berkas `.html`, `.svg` ber-skrip JavaScript (*Stored XSS*), atau skrip berbahaya langsung ke folder `public/storage/modul-ajar/`. Selain itu `$validated['user'] = json_encode($request->user());` berpotensi menyimpan metadata user mentah.
- **Rekomendasi Remediasi**:
  1. Terapkan `'file' => 'nullable|file|max:20480|mimes:pdf,doc,docx,ppt,pptx,xls,xlsx,jpg,jpeg,png,txt,zip'`.
  2. Gunakan `Storage::disk('public')->delete(...)` (bukan default local disk `Storage::delete(...)` yang tidak sinkron dengan disk `'public'`).

---

### 4. 🟠 [HIGH] Ketiadaan Pembatasan Peran (*Missing RBAC*) pada Rute Modul Sensitif
- **Berkas Terkait**: [`backend/routes/api.php`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/routes/api.php)
- **Mekanisme Kerentanan**:
  Seluruh rute di dalam blok `Route::middleware(['auth:sanctum', 'record.activity'])` saat ini dapat diakses oleh **semua akun yang login**, tanpa membedakan jabatan/role:
  - **Pengaturan Sistem** (`lines 157-164`): `POST /api/settings/update`, `POST /api/settings/location`, `PUT /api/settings/{key}` — sembarang akun login dapat mengubah koordinat GPS absensi guru dan konfigurasi sistem.
  - **Keuangan & Mutasi** (`lines 105-108, 332-363`): `/api/pengeluaran`, `/api/pemasukan`, `/api/mutasi-kas`, `/api/mutasi-lazis`, `/api/syahriah`, `POST /api/shu/calculate` — sembarang akun (misal guru/keamanan) dapat memodifikasi atau menghapus catatan kas & SPP.
  - **Kesehatan / Rekam Medis** (`lines 484-511`): `/api/kesehatan/*` — data diagnosis medis dan stok obat dapat diakses & diubah oleh akun non-kesehatan.
  - **Perizinan Keamanan** (`lines 321-328`): `POST/PUT/DELETE /api/izin`, `POST/PUT/DELETE /api/jenis-izin`, `POST/DELETE /api/santri-masuk` — tidak dibatasi pada `role:admin,kamtib,keamanan,pengurus`.
  - **Akademik & Nilai** (`lines 270-297`): `/api/nilai`, `/api/nilai/simpan-per-kelas`, `/api/absen/otomatisasi-harian` — tidak dibatasi pada peran akademik/guru/admin.
- **Rekomendasi Remediasi**:
  Kelompokkan setiap modul rute di `routes/api.php` dengan middleware `role:...` sesuai unit kerja masing-masing.

---

### 5. 🟠 [HIGH] Sesi Token Sanctum Tidak Dicabut saat Penonaktifan / Reset Password & Tanpa Kedaluwarsa
- **Berkas Terkait**: [`UserController.php`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/app/Http/Controllers/Api/UserController.php#L113-L142) & [`config/sanctum.php`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/config/sanctum.php#L50)
- **Mekanisme Kerentanan**:
  1. Ketika admin menonaktifkan user (`UserController::destroy`) atau mereset password (`UserController::resetPassword`), `$user->tokens()->delete()` tidak dipanggil. Token Bearer yang sudah terbit tetap sah digunakan untuk mengakses API.
  2. Nilai `'expiration' => null` di `config/sanctum.php` membuat token berlaku selamanya tanpa batas waktu (*infinite session*).
- **Rekomendasi Remediasi**:
  1. Tambahkan `$user->tokens()->delete();` di `UserController::destroy()` dan `UserController::resetPassword()`.
  2. Set `'expiration' => env('SANCTUM_TOKEN_EXPIRATION', 1440)` (misal 24 jam) dan tambahkan pengecekan `$request->user()->is_active` pada middleware auth API.

---

### 6. 🟠 [HIGH] Konflik Header CORS Wildcard (`*`) pada `backend/public/.htaccess`
- **Berkas Terkait**: [`backend/public/.htaccess`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/public/.htaccess#L43-L47)
- **Mekanisme Kerentanan**:
  Meskipun `backend/config/cors.php` telah dikunci ke domain resmi, berkas Apache `.htaccess` menimpa header HTTP secara langsung:
  ```apache
  Header always set Access-Control-Allow-Origin "*"
  Header always set Access-Control-Allow-Credentials "true"
  ```
  Hal ini membatalkan isolasi CORS di level web server dan melanggar spesifikasi W3C CORS (kombinasi `*` + `Credentials: true`). Selain itu terdapat berkas duplikat `backend/public/(1).htaccess`.
- **Rekomendasi Remediasi**:
  Hapus blok header `Access-Control-*` manual dari `backend/public/.htaccess` (biarkan Laravel `HandleCors` + `config/cors.php` menanganinya secara dinamis dan aman), serta hapus berkas duplikat `(1).htaccess`.

---

### 7. 🟠 [HIGH] Eksposur Identitas Santri (Nama, NIS, Pelanggaran, Syahriah) di Endpoint Publik TV Monitoring
- **Berkas Terkait**: [`DashboardController.php`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/app/Http/Controllers/Api/DashboardController.php#L294-L310) (`lines 402-445`) & [`routes/api.php`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/routes/api.php#L94-L98)
- **Mekanisme Kerentanan**:
  Endpoint `GET /api/monitoring/realtime` dan `GET /api/dashboard` sengaja dibuka tanpa autentikasi untuk kebutuhan TV Display. Namun saat ini respons JSON mengembalikan data individual santri:
  - `santri_sedang_izin`: nama lengkap, **NIS**, alasan keluar
  - `syahriah.transaksi_terbaru`: nama lengkap, **NIS**, `penyetor`, `penerima`
  - `riwayat_pelanggaran_terbaru`: nama lengkap, **NIS**, jenis pelanggaran, poin, dan sanksi
  Siapa pun di internet yang memanggil `/api/monitoring/realtime` dapat melihat daftar santri yang sedang di luar pesantren atau memiliki catatan pelanggaran.
- **Rekomendasi Remediasi**:
  Masking NIS / nama penyetor pada endpoint publik atau lindungi akses detail non-agregat menggunakan display token khusus TV Wallboard (misal query/header token TV read-only).

---

### 8. 🟡 [MEDIUM] Kebocoran Pesan Exception Mentah pada JSON Error Response
- **Berkas Terkait**: [`SantriController.php`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/app/Http/Controllers/Api/SantriController.php#L788), [`ImportSantriController.php`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/app/Http/Controllers/Api/ImportSantriController.php#L61-L63), [`RuangKelasController.php`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/app/Http/Controllers/Api/RuangKelasController.php#L119), `ShuController.php`, `SavingController.php`, `SyahriahController.php`, `LogNfcController.php`
- **Mekanisme Kerentanan**:
  Mengembalikan `$e->getMessage()`, `$e->getFile()`, dan `$e->getLine()` pada respons HTTP 500 membocorkan struktur tabel SQL dan path direktori internal server kepada penyerang.
- **Rekomendasi Remediasi**:
  Log detail error ke `Log::error(...)` di server, dan kembalikan pesan generik yang aman ke klien (kecuali jika `config('app.debug')` aktif di local).

---

### 9. 🟡 [MEDIUM] Penggunaan `getClientOriginalExtension()` pada Upload File
- **Berkas Terkait**: [`AbsenGuruController.php`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/app/Http/Controllers/Api/AbsenGuruController.php#L50) (`line 117`)
- **Mekanisme Kerentanan**:
  Penamaan file memakai ekstensi kiriman klien (`$file->getClientOriginalExtension()`). Meski sudah divalidasi `mimes:jpeg,png,jpg`, praktik aman standar adalah menggunakan `$file->store('absen-guru', 'public')` atau `$file->extension()` yang menentukan ekstensi berdasarkan konten MIME asli.

---

### 10. 🔵 [LOW / PRA-PRODUKSI] Konfigurasi Debug & Berkas Duplikat Webroot
- [`backend/.env`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/.env#L2-L4): Pastikan `APP_ENV=production` dan `APP_DEBUG=false` sebelum deployment live.
- [`backend/public/(1).htaccess`](file:///c:/Users/Mr-A%202025/Desktop/PROJEK%20YADI/WEB/freebuff/backend/public/%281%29.htaccess): Hapus berkas duplikat yang tidak terpakai.

---

## 📋 Rekomendasi Prioritas Tindakan
1. **Segera (P0)**: Hapus hardcoded `X-API-KEY` dari JS frontend & rotasi key, perbaiki Arbitrary File Deletion di `InstansiLampiranController`, dan amankan validasi upload di `ModulAjarController`.
2. **Tinggi (P1)**: Terapkan `role:...` pada seluruh grup rute sensitif di `routes/api.php`, tambahkan pencabutan token Sanctum di `UserController`, dan bersihkan CORS wildcard di `public/.htaccess`.
3. **Menengah (P2)**: Sembunyikan `$e->getMessage()`/`getFile()` dari respons 500, masking/amankan detail PII di TV Monitoring publik, dan perbaiki `AbsenGuruController`.
