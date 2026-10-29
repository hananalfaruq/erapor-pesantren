# PRD — Sistem Pengumpulan Nilai UAS Pesantren (SMP & SMA)

| Atribut | Isi |
|---|---|
| Versi dokumen | 1.1 |
| Tanggal | 29 Oktober 2026 (revisi setelah laporan orientasi Tahap 1) |
| Lembaga | Pondok Pesantren Darul Kholidin (SMP & SMA) |
| Tujuan dokumen | Pegangan teknis untuk membangun aplikasi (dibaca manusia dan agen AI di Antigravity IDE) |
| Stack | Laravel 13, PHP 8.3/8.4, PostgreSQL 16+, Blade + Tailwind CSS + Alpine.js, Vite |
| Deployment | Server lokal pesantren (LAN), akses jarak jauh sementara via `trycloudflare` |
| Bahasa UI | Indonesia |

---

## 0. Cara Memakai Dokumen Ini (untuk Agen AI)

1. Kerjakan **per milestone** (Bagian 14). Selesaikan satu milestone, jalankan migrasi dan test, baru lanjut.
2. Nama kode (tabel, kolom, class, route) memakai **bahasa Inggris**. Semua teks yang tampil ke pengguna memakai **bahasa Indonesia**.
3. **Jangan meng-hardcode nama mata pelajaran** di kode atau seeder. Mapel sepenuhnya data yang diisi Administrator.
4. Semua aset (CSS, JS, font) di-bundle lokal lewat Vite. **Dilarang memakai CDN** karena server sering tanpa internet.
5. Jangan membuat fitur di luar ruang lingkup (Bagian 2.2), terutama cetak rapor, docx/PDF, ekskul, atau sikap.
6. Bila ada aturan yang ambigu, pilih perilaku yang paling aman (menolak input, bukan menebak), lalu catat di `docs/decisions.md`.
7. Setiap fitur yang punya **Kriteria Penerimaan (KP)** wajib punya test otomatis.

---

## 1. Ringkasan & Tujuan

Aplikasi web untuk **mengumpulkan, memvalidasi, dan merekap nilai UAS mata pelajaran kepesantrenan** (Tauhid, Fiqh, Nahwu, Shorof, Tarekh, dan seterusnya) pada jenjang SMP dan SMA. Nilai formal Kemendikbud tidak ditangani aplikasi ini.

Alur singkat:
1. Administrator menyiapkan master data (periode, kelas, mapel, kurikulum per tingkat, guru, santri) dan plotting pengajar.
2. Guru mapel mengisi nilai UAS (web grid atau import Excel).
3. Wali kelas mengisi absensi dan memantau kelengkapan nilai di kelasnya.
4. Administrator mengunduh **Excel rekap nilai per kelas** sebagai output akhir (untuk arsip dan sumber Mail Merge Word di luar aplikasi).

**Tujuan terukur**
- Guru dapat mengisi nilai satu kelas (±35 santri) dalam kurang dari 3 menit lewat web grid dengan keyboard.
- Tidak ada nilai di luar 2–9 yang bisa masuk ke database, lewat jalur apa pun.
- Nilai semester lama tidak bisa berubah tanpa tindakan eksplisit Administrator.
- Administrator dapat mengekspor rekap satu kelas atau seluruh kelas dalam kurang dari 30 detik.

---

## 2. Ruang Lingkup

### 2.1 Dalam Lingkup
- Manajemen periode (tahun ajaran + semester), isolasi data per periode, dan clone Ganjil ke Genap.
- Master data: kelas (rombel), mapel, kurikulum per tingkat, guru, santri, penempatan santri.
- Plotting guru mapel dan wali kelas per rombel per periode.
- Input nilai UAS (web grid dan import Excel), input absensi, monitoring kelengkapan.
- Export Excel rekap nilai per kelas (khusus Admin).
- Backup database terjadwal.
- Kenaikan kelas akhir tahun (Milestone 6).

### 2.2 Di Luar Lingkup (jangan dibuat)
- Cetak rapor, template Word, konversi PDF, QR code, tanda tangan digital.
- Nilai ekstrakurikuler, sikap (kebersihan/kelakuan/kerajinan), poin pelanggaran.
- Nilai selain UAS (harian, tugas, UTS), rata-rata, jumlah nilai, ranking.
- Portal atau akun wali santri, notifikasi WhatsApp.
- Konversi angka ke huruf/angka Arab, terbilang Arab (ditangani Admin di Excel/Word sendiri).
- Sinkronisasi dengan Dapodik atau sistem lain (data santri berdiri sendiri).

---

## 3. Glosarium

| Istilah | Arti |
|---|---|
| Tingkat | Level kelas pesantren 1–6. Tingkat 1–3 = SMP, 4–6 = SMA |
| Rombel / Kelas | Kelas paralel dalam satu tingkat, contoh `1A`, `1B`, … `1I` |
| Kelas Dapodik | Nama kelas di data Dapodik (teks bebas, contoh `7A`). Berbeda dengan kelas pesantren |
| Periode | Kombinasi tahun ajaran + semester, contoh `2026/2027 – Genap` |
| Periode aktif | Satu-satunya periode yang boleh diubah datanya |
| Enrollment | Penempatan santri di sebuah rombel pada sebuah periode |
| Plotting | Penugasan guru ke mapel+rombel dan penunjukan wali kelas |
| UAS | Ulangan Akhir Semester, satu-satunya komponen nilai |

---

## 4. Pengguna & Hak Akses

Akun hanya punya dua nilai `role`: `admin` dan `teacher`. Peran **Wali Kelas bukan role**, melainkan status yang diturunkan dari tabel `homeroom_assignments` pada periode aktif. Satu guru bisa sekaligus pengajar mapel dan wali kelas.

| Kemampuan | Admin | Guru Mapel | Wali Kelas |
|---|---|---|---|
| Kelola periode, kelas, mapel, kurikulum, guru, santri | Ya | Tidak | Tidak |
| Plotting guru dan wali kelas | Ya | Tidak | Tidak |
| Input/ubah nilai | Semua kelas-mapel (periode aktif) | Hanya kelas-mapel yang diampu | Hanya kelas-mapel yang **diampunya sendiri** (lewat `teaching_assignments`). Status wali kelas **tidak** memberi hak input nilai mapel guru lain |
| Input absensi | Semua kelas | Tidak | Hanya kelas perwalian |
| Lihat monitoring kelengkapan | Semua kelas | Progres kelas-mapel miliknya | Kelas perwalian (semua mapel) |
| Unduh template import nilai | Semua | Yang diampu | Yang diampu |
| Export Excel rekap nilai | **Ya (satu-satunya)** | Tidak | Tidak |
| Lihat data periode lama | Ya, hanya baca | Tidak | Tidak |

Catatan:
- Wali kelas hanya melihat **status** kelengkapan nilai mapel lain (terisi/belum, jumlah), bukan nilainya.
- Akun admin boleh ditugaskan sebagai guru/wali kelas. Admin selalu melihat menu admin lengkap; bila ia punya penugasan mengajar atau perwalian, menu "Kelas Saya" (dan Absensi/Kelengkapan Kelas) juga muncul.

---

## 5. Aturan Bisnis

| ID | Aturan |
|---|---|
| BR-01 | Hanya ada **satu komponen nilai per mapel per santri per periode**: UAS. |
| BR-02 | Nilai wajib **bilangan bulat 2–9**. Nilai 0, 1, ≥10, desimal (7.5), huruf, atau teks ditolak di form web, import Excel, dan database (CHECK constraint). |
| BR-03 | Struktur: Tingkat 1–3 = jenjang SMP, Tingkat 4–6 = jenjang SMA. Rombel dibuat dinamis, jumlahnya berbeda tiap tingkat. |
| BR-04 | Kurikulum (daftar mapel) **berbeda per tingkat tetapi sama untuk semua rombel dalam tingkat yang sama**. Diatur lewat `grade_level_subjects`. |
| BR-05 | Guru hanya bisa menginput nilai untuk pasangan (rombel, mapel) yang tercatat di `teaching_assignments` periode aktif **dan** mapel itu ada di kurikulum tingkat rombel tersebut. |
| BR-06 | Satu (rombel, mapel) pada satu periode hanya punya **satu** guru pengampu. Satu rombel hanya punya **satu** wali kelas per periode. |
| BR-07 | Hanya **satu periode berstatus aktif** pada satu waktu. Seluruh operasi tulis ditolak jika periode yang dituju bukan periode aktif. |
| BR-08 | Setiap santri hanya punya **satu enrollment per periode**. |
| BR-09 | Absensi berupa tiga bilangan bulat ≥ 0: Sakit, Izin, Alpa (satuan hari, per semester). |
| BR-10 | Di export Excel: mapel **tidak ada di kurikulum tingkat** itu → sel berisi `-`. Mapel ada di kurikulum tetapi **belum dinilai** → sel **kosong**. Absensi belum diisi → sel kosong. |
| BR-11 | Clone Ganjil ke Genap menyalin enrollment (termasuk kelas Dapodik), wali kelas, dan plotting guru. **Tidak** menyalin nilai dan absensi. |
| BR-12 | NIS unik secara global, disimpan sebagai teks (angka nol di depan tidak boleh hilang). |
| BR-13 | Kelas Dapodik disimpan **per enrollment** (berubah tiap periode), bukan di profil santri. |
| BR-14 | Penghapusan data master yang sudah dipakai diblokir (pakai FK `RESTRICT`); gunakan penonaktifan (`is_active`). |

---

## 6. Tech Stack & Konfigurasi

### 6.1 Stack
- **Backend:** Laravel 13, PHP 8.3/8.4 (ikuti persyaratan minimum resmi Laravel 13 saat instalasi).
- **Database:** PostgreSQL 16+, database `erapor_pesantren`.
- **Frontend:** Blade + Tailwind CSS + Alpine.js, build dengan Vite, semua aset lokal.
- **Excel:** `maatwebsite/excel` v4.x (terverifikasi kompatibel dengan Laravel 13 dan PHP ^8.3 pada Tahap 1). Ini major versi baru: **baca dokumentasi v4 dan jangan menulis kode dari ingatan/tutorial v3**, karena sebagian API bisa berbeda. Bila ada fitur yang tidak tersedia (mis. data validation, proteksi sheet, sheet tersembunyi), gunakan PhpSpreadsheet langsung atau `spatie/simple-excel` dan catat di `docs/decisions.md`.
- **Auth:** session auth bawaan Laravel (login custom dengan `username`), tanpa registrasi publik.
- **Test:** Pest atau PHPUnit (feature test dengan database PostgreSQL terpisah `erapor_pesantren_test`).

### 6.2 `.env` utama
```
APP_NAME="E-Rapor Pesantren"
APP_ENV=production
APP_DEBUG=false
APP_TIMEZONE=Asia/Jakarta
APP_LOCALE=id
APP_URL=http://erapor.local

DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=erapor_pesantren
DB_USERNAME=...
DB_PASSWORD=...

SESSION_DRIVER=file
SESSION_LIFETIME=120
CACHE_STORE=file
QUEUE_CONNECTION=sync

ADMIN_USERNAME=admin
ADMIN_PASSWORD=          # diisi saat seeding awal
ADMIN_LAN_ONLY=false     # true = halaman admin ditolak bila diakses via tunnel (lihat SEC-07)
BACKUP_PATH=D:\backup\erapor
BACKUP_RETENTION_DAYS=30
```

### 6.3 Proxy Cloudflare (`trycloudflare`)
`bootstrap/app.php`:
```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->trustProxies(at: '*');
})
```
`AppServiceProvider::boot()` (aman dipakai bersama trustProxies):
```php
if (request()->header('X-Forwarded-Proto') === 'https') {
    \Illuminate\Support\Facades\URL::forceScheme('https');
}
```
Catatan: URL `trycloudflare` **berubah setiap tunnel dijalankan ulang** dan tanpa jaminan layanan. Admin perlu membagikan link baru setiap kali. Untuk link tetap, gunakan named tunnel dengan domain sendiri (di luar lingkup awal).

---

## 7. Skema Database (PostgreSQL)

11 tabel inti + 1 tabel opsional (`activity_logs`). Tabel bawaan Laravel (`migrations`, `password_reset_tokens`, dll.) dipertahankan seperlunya. Semua tabel punya `created_at`/`updated_at`. FK memakai `RESTRICT` kecuali disebut lain.

### 7.1 `users`
| Kolom | Tipe | Null | Keterangan |
|---|---|---|---|
| id | bigint PK | | |
| name | varchar(150) | tidak | Nama + gelar |
| name_arabic | varchar(150) | ya | Nama Arab (dipakai di export) |
| username | varchar(50) | tidak | **UNIQUE**, huruf kecil |
| password | varchar | tidak | bcrypt |
| role | varchar(10) | tidak | CHECK `role IN ('admin','teacher')` |
| is_active | boolean | tidak | default true |
| must_change_password | boolean | tidak | default true |
| last_login_at | timestamp | ya | |
| remember_token | varchar(100) | ya | |

### 7.2 `academic_years` (periode)
| Kolom | Tipe | Null | Keterangan |
|---|---|---|---|
| id | bigint PK | | |
| year_name | varchar(9) | tidak | Format `2026/2027` |
| semester | varchar(6) | tidak | CHECK `IN ('ganjil','genap')` |
| is_active | boolean | tidak | default false |
| activated_at | timestamp | ya | |

Constraint: `UNIQUE (year_name, semester)`; **partial unique index** agar hanya satu aktif:
```sql
CREATE UNIQUE INDEX academic_years_single_active ON academic_years (is_active) WHERE is_active = true;
```

### 7.3 `classrooms` (master rombel, global lintas periode)
| Kolom | Tipe | Null | Keterangan |
|---|---|---|---|
| id | bigint PK | | |
| grade_level | smallint | tidak | 1–6 |
| level | varchar(3) | tidak | `SMP` / `SMA` |
| name | varchar(10) | tidak | Contoh `1A`, **UNIQUE**, wajib diawali digit `grade_level` (divalidasi di FormRequest **dan** CHECK database) |
| is_active | boolean | tidak | default true |

CHECK: `(grade_level BETWEEN 1 AND 3 AND level='SMP') OR (grade_level BETWEEN 4 AND 6 AND level='SMA')`. CHECK kedua: `left(name, 1) = grade_level::text`.

### 7.4 `subjects` (master mapel)
| Kolom | Tipe | Null | Keterangan |
|---|---|---|---|
| id | bigint PK | | |
| code | varchar(30) | tidak | **UNIQUE**, regex `^[a-z0-9_]+$`, dipakai sebagai header Excel mode Mail Merge |
| name_latin | varchar(100) | tidak | |
| name_arabic | varchar(100) | ya | |
| sort_order | integer | tidak | Urutan kolom di Excel dan layar |
| is_active | boolean | tidak | default true. Mapel nonaktif tidak muncul di pilihan baru (kurikulum, plotting), data lama tetap |

### 7.5 `grade_level_subjects` (kurikulum per tingkat)
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | bigint PK | |
| grade_level | smallint | CHECK 1–6 |
| subject_id | bigint FK → subjects | |

`UNIQUE (grade_level, subject_id)`.

### 7.6 `students`
| Kolom | Tipe | Null | Keterangan |
|---|---|---|---|
| id | bigint PK | | |
| nis | varchar(20) | tidak | **UNIQUE**, teks |
| name_latin | varchar(150) | tidak | |
| name_arabic | varchar(150) | ya | |
| gender | char(1) | tidak | CHECK `IN ('L','P')` |
| is_active | boolean | tidak | default true |

Index: `name_latin` (pencarian `ILIKE`).

### 7.7 `class_enrollments`
| Kolom | Tipe | Null | Keterangan |
|---|---|---|---|
| id | bigint PK | | |
| academic_year_id | bigint FK → academic_years | tidak | |
| classroom_id | bigint FK → classrooms | tidak | |
| student_id | bigint FK → students | tidak | |
| dapodik_class | varchar(10) | ya | Teks bebas |

`UNIQUE (academic_year_id, student_id)`; index `(academic_year_id, classroom_id)`.

### 7.8 `homeroom_assignments`
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | bigint PK | |
| academic_year_id | FK | |
| classroom_id | FK | |
| teacher_id | FK → users | |

`UNIQUE (academic_year_id, classroom_id)`; index `(academic_year_id, teacher_id)`.

### 7.9 `teaching_assignments`
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | bigint PK | |
| academic_year_id | FK | |
| classroom_id | FK | |
| subject_id | FK | |
| teacher_id | FK → users | |

`UNIQUE (academic_year_id, classroom_id, subject_id)`; index `(academic_year_id, teacher_id)`.

### 7.10 `subject_grades`
| Kolom | Tipe | Null | Keterangan |
|---|---|---|---|
| id | bigint PK | | |
| class_enrollment_id | bigint FK → class_enrollments **ON DELETE CASCADE** | tidak | |
| subject_id | bigint FK → subjects | tidak | |
| score | smallint | tidak | **CHECK (score BETWEEN 2 AND 9)** |
| updated_by | bigint FK → users | ya | |

`UNIQUE (class_enrollment_id, subject_id)`; index `(subject_id)`. **Baris hanya ada jika nilai sudah diisi** (tidak ada baris = belum dinilai).

### 7.11 `attendances`
| Kolom | Tipe | Keterangan |
|---|---|---|
| id | bigint PK | |
| class_enrollment_id | FK → class_enrollments **ON DELETE CASCADE**, UNIQUE | |
| sick | smallint default 0 | CHECK `>= 0 AND <= 999` |
| permission | smallint default 0 | CHECK sama |
| alpha | smallint default 0 | CHECK sama |
| updated_by | FK → users, null | |

Baris dibuat saat wali kelas pertama kali menyimpan absensi kelasnya.

### 7.12 `activity_logs` (opsional, Milestone 6)
`id`, `user_id`, `action` (varchar), `subject_type`, `subject_id`, `old_values` (jsonb), `new_values` (jsonb), `ip_address`, `created_at`. Mencatat: perubahan/penghapusan nilai, perubahan absensi, ganti periode aktif, clone semester, import, export.

### 7.13 Seeder
- `AdminSeeder` (wajib): membuat akun admin dari `ADMIN_USERNAME` dan `ADMIN_PASSWORD` dengan `must_change_password = true`. Bila `ADMIN_PASSWORD` kosong, seeder **berhenti dengan pesan error yang jelas** (tidak membuat akun dengan password kosong/bawaan). **Tidak ada seeder mapel.**
- `DemoSeeder` (hanya `APP_ENV=local`): 6 tingkat, 2–3 rombel per tingkat, 5 mapel dummy dengan kurikulum berbeda per tingkat, 10 guru, ±150 santri, plotting lengkap. Untuk uji coba dan screenshot.

---

## 8. Fitur & Kriteria Penerimaan (KP)

Prioritas: **M** = Must, **S** = Should, **C** = Could.

### 8.1 Autentikasi & Akun
| ID | Fitur | Prio |
|---|---|---|
| FR-AUTH-01 | Login dengan username + password | M |
| FR-AUTH-02 | Wajib ganti password saat login pertama (`must_change_password`) | M |
| FR-AUTH-03 | Admin mereset password guru, membuat/menonaktifkan akun | M |
| FR-AUTH-04 | Ganti password sendiri | M |

KP:
- Akun `is_active = false` tidak bisa login, pesan: "Akun dinonaktifkan. Hubungi Administrator."
- Login gagal 5 kali per menit (per username+IP) → diblokir sementara (throttle bawaan Laravel).
- Password minimal 8 karakter. Pesan error login tidak membocorkan apakah username ada.
- Reset password oleh admin membuat password sementara acak yang ditampilkan **sekali** dan mengaktifkan `must_change_password`.
- Setelah login, pengguna diarahkan ke dashboard sesuai role.

### 8.2 Manajemen Periode
| ID | Fitur | Prio |
|---|---|---|
| FR-PRD-01 | CRUD periode (tahun ajaran + semester) | M |
| FR-PRD-02 | Mengaktifkan periode (menonaktifkan yang lain dalam satu transaksi) | M |
| FR-PRD-03 | Clone Ganjil ke Genap dalam satu klik | M |
| FR-PRD-04 | Admin melihat periode lama (read-only) lewat pemilih periode | M |
| FR-PRD-05 | Banner periode di header semua halaman | M |

KP:
- Semua layar menampilkan badge periode, contoh "2026/2027 – Genap · Aktif".
- Aktivasi periode memakai `DB::transaction`; indeks parsial menjamin satu aktif. Mengaktifkan ulang periode lama diperbolehkan (Admin, dengan dialog konfirmasi).
- **Clone**: tombol "Buat Semester Genap & Salin Konfigurasi" aktif jika ada periode Ganjil pada tahun ajaran terpilih. Dalam satu transaksi: membuat periode Genap (atau memakai yang sudah ada dan kosong), menyalin `class_enrollments`, `homeroom_assignments`, `teaching_assignments`, lalu mengaktifkannya. Nilai dan absensi tidak disalin.
- Clone ditolak bila periode Genap tujuan sudah memiliki enrollment (pesan jelas), agar tidak terjadi duplikasi atau penimpaan.
- Guru dan wali kelas selalu bekerja di periode aktif. Jika belum ada periode aktif: pesan "Belum ada periode aktif. Hubungi Administrator."
- Semua endpoint tulis memvalidasi bahwa periode target = periode aktif (BR-07), termasuk saat dipanggil langsung lewat HTTP.

### 8.3 Master Data (Admin)
| ID | Fitur | Prio |
|---|---|---|
| FR-MST-01 | CRUD kelas/rombel (tingkat, nama) | M |
| FR-MST-02 | CRUD mapel (kode, nama Latin, nama Arab, urutan) | M |
| FR-MST-03 | Matriks kurikulum: baris = mapel, kolom = Tingkat 1–6, centang | M |
| FR-MST-04 | CRUD guru + Import Excel guru | M |
| FR-MST-05 | CRUD santri + Import Excel santri (+ penempatan kelas) | M |
| FR-MST-06 | Pencarian dan filter daftar santri (nama/NIS, kelas, status) | M |
| FR-MST-07 | Pindah rombel santri di periode aktif | M |

KP:
- `classrooms.name` harus diawali digit tingkatnya; `level` diisi otomatis dari tingkat.
- Kelas/mapel/guru/santri yang sudah dipakai tidak bisa dihapus (pesan: "Data sedang dipakai di …"); sediakan nonaktifkan (`is_active`). Kelas dan mapel nonaktif tidak muncul di pilihan baru (enrollment, kurikulum, plotting) tetapi data lama tetap utuh.
- Matriks kurikulum: mencabut centang mapel dari suatu tingkat **diblokir** bila sudah ada nilai mapel itu pada rombel tingkat tersebut di periode aktif.
- Mengubah `code` mapel menampilkan peringatan bahwa header Excel Mail Merge ikut berubah.
- Pindah rombel dalam **tingkat yang sama** mempertahankan nilai. Pindah **lintas tingkat** hanya diizinkan jika santri belum punya nilai/absensi di periode itu.
- Santri nonaktif tidak muncul di daftar penempatan baru, tetapi riwayat enrollment dan nilainya tetap ada.
- Import guru/santri: spesifikasi di Bagian 9.1 dan 9.2.

### 8.4 Penempatan & Plotting (Admin)
| ID | Fitur | Prio |
|---|---|---|
| FR-PLT-01 | Tetapkan wali kelas per rombel (periode aktif) | M |
| FR-PLT-02 | Matriks plotting guru: baris = rombel, kolom = mapel kurikulum tingkat itu, sel = dropdown guru | M |
| FR-PLT-03 | Isi massal: terapkan satu guru ke semua rombel dalam satu tingkat untuk satu mapel | S |

KP:
- Sel plotting hanya muncul untuk mapel yang ada di kurikulum tingkat rombel tersebut (BR-05).
- Dropdown hanya memuat user aktif.
- Mengganti guru pengampu tidak menghapus nilai yang sudah ada.
- Satu rombel hanya satu wali kelas; menetapkan ulang menggantikan yang lama.

### 8.5 Input Nilai (Guru Mapel / Wali Kelas / Admin)
| ID | Fitur | Prio |
|---|---|---|
| FR-GRD-01 | Halaman "Kelas Saya": daftar kartu (rombel + mapel) dengan progres terisi/total | M |
| FR-GRD-02 | Web grid input nilai per (rombel, mapel) | M |
| FR-GRD-03 | Unduh template Excel dan import nilai | M |
| FR-GRD-04 | Hapus nilai (kosongkan sel lalu simpan) | M |

KP web grid:
- Kolom: No, NIS, Nama Santri, Nilai. Hanya santri yang ter-enroll di rombel pada periode aktif.
- Input satu karakter. **Hanya angka 2–9 yang diterima**; karakter lain tidak masuk ke kolom. Setelah angka valid diketik, fokus otomatis pindah ke baris berikutnya. Navigasi: Enter/↓/Tab = bawah, Shift+Tab/↑ = atas.
- Tombol "Simpan" mengirim semua baris sekaligus (bulk upsert dalam transaksi). Indikator "Ada perubahan belum disimpan" dan peringatan saat meninggalkan halaman.
- Sel yang dikosongkan lalu disimpan **menghapus** nilai tersebut.
- Validasi server tetap berlaku (nilai 2–9, enrollment milik rombel itu, hak akses). Pelanggaran → HTTP 422/403 dan pesan Indonesia, contoh: "Nilai harus berupa angka bulat 2 sampai 9."
- Layar berfungsi di HP (input `inputmode="numeric"`, tombol cukup besar).
- Guru yang tidak mengampu (rombel, mapel) tersebut mendapat 403.

KP import: spesifikasi di Bagian 9.3.

### 8.6 Absensi & Monitoring (Wali Kelas)
| ID | Fitur | Prio |
|---|---|---|
| FR-ABS-01 | Grid absensi: No, NIS, Nama, Sakit, Izin, Alpa | M |
| FR-ABS-02 | Dashboard kelengkapan kelas perwalian | M |
| FR-ABS-03 | Import absensi via Excel | C |

KP:
- Hanya bilangan bulat 0–999. Sel kosong saat simpan dianggap 0 dan baris absensi dibuat untuk semua santri kelas.
- Dashboard: matriks santri × mapel atau ringkasan per mapel. Tiap mapel berstatus **Belum diisi / Sebagian (x dari y) / Lengkap**, plus nama guru pengampu, dan status absensi.
- Wali kelas hanya mengakses rombel perwaliannya di periode aktif.

### 8.7 Monitoring Global & Export (Admin)
| ID | Fitur | Prio |
|---|---|---|
| FR-ADM-01 | Dashboard progres: matriks rombel × mapel (status warna), filter tingkat | M |
| FR-ADM-02 | Progres per guru (berapa kelas-mapel yang belum lengkap) | S |
| FR-ADM-03 | Export Excel rekap nilai per kelas | M |
| FR-ADM-04 | Admin dapat menginput/mengoreksi nilai dan absensi kelas mana pun (periode aktif) | M |
| FR-ADM-05 | Checklist onboarding di dashboard saat data kosong | S |

KP export: spesifikasi di Bagian 9.4.
- Checklist onboarding menampilkan urutan: Periode aktif → Kelas → Mapel → Kurikulum tingkat → Guru → Plotting → Santri, dengan status selesai/belum.

### 8.8 Kenaikan Kelas (Milestone 6)
| ID | Fitur | Prio |
|---|---|---|
| FR-PRM-01 | Buat periode Ganjil tahun ajaran baru dan promosikan santri | S (wajib sebelum tahun ajaran berikutnya) |

KP:
- Admin memetakan rombel asal → rombel tujuan (contoh `1A → 2A`, `3B → 4B`; Tingkat 6 = lulus, tidak dipindahkan), melihat **pratinjau** (jumlah santri per pemetaan), lalu menjalankan dalam satu transaksi.
- Kelas Dapodik tujuan dapat diisi lewat import atau diedit manual setelahnya.
- Wali kelas dan plotting guru **tidak** disalin otomatis (diatur ulang Admin).
- Santri Tingkat 6 ditandai lulus (`is_active = false`) lewat aksi terpisah dengan konfirmasi.

### 8.9 Keamanan, Log, & Backup
| ID | Fitur | Prio |
|---|---|---|
| SEC-01 | Role middleware + Policy pada semua route | M |
| SEC-02 | Tanpa registrasi publik; semua route di balik `auth` kecuali login | M |
| SEC-03 | Throttle login (5/menit) | M |
| SEC-04 | `APP_DEBUG=false` di produksi, CSRF aktif di semua form | M |
| SEC-05 | Activity log perubahan nilai/absensi | S |
| SEC-06 | Backup harian `pg_dump` terjadwal | M |
| SEC-07 | Opsi `ADMIN_LAN_ONLY`: bila true, route admin menolak request yang membawa header `Cf-Connecting-Ip` (berarti lewat tunnel) | S |

KP backup: Laravel scheduler menjalankan perintah `backup:database` harian (pukul 23:00) yang menghasilkan `erapor_YYYYMMDD_HHMM.sql.gz` ke `BACKUP_PATH`, menyimpan `BACKUP_RETENTION_DAYS` hari terakhir, dan mencatat kegagalan ke log. Server menjalankan `php artisan schedule:run` tiap menit (Task Scheduler di Windows / cron di Linux). Dokumentasikan prosedur restore di `docs/restore.md`.

---

## 9. Spesifikasi Import & Export Excel

Aturan umum import:
- Format `.xlsx`, maksimum 2 MB.
- Header dibaca berdasarkan **nama kolom** (huruf besar/kecil tidak berpengaruh, spasi ujung di-trim), bukan posisi.
- Proses **dua tahap**: (1) validasi seluruh baris tanpa menulis ke database, (2) bila **nol error**, simpan dalam satu transaksi (all-or-nothing). Bila ada error, tampilkan tabel error (nomor baris, kolom, pesan) dan sediakan unduhan `.xlsx` berisi daftar error.
- Teks Excel dipotong spasi ujung; NIS selalu dibaca sebagai teks.
- Hasil sukses menampilkan ringkasan: jumlah baru, diperbarui, tidak berubah, dilewati.

### 9.1 Import Guru
| Kolom | Wajib | Aturan |
|---|---|---|
| nama | ya | Maks 150 karakter |
| nama_arab | tidak | |
| username | ya | Unik, `[a-z0-9._]`, huruf kecil |
| password_awal | tidak | Bila kosong, dibuat acak dan ditampilkan sekali di hasil import |

Semua akun hasil import: `role = teacher`, `must_change_password = true`. Username yang sudah ada → baris diperbarui (nama saja, tidak mengubah password).

### 9.2 Import Santri (+ penempatan)
| Kolom | Wajib | Aturan |
|---|---|---|
| nis | ya | Teks, unik di dalam file |
| nama | ya | |
| nama_arab | tidak | |
| jk | ya | `L` atau `P` (tidak peka huruf besar/kecil) |
| kelas | ya | Harus sudah ada di master kelas (contoh `1A`) |
| kelas_dapodik | tidak | Teks bebas, maks 10 karakter |

- NIS sudah ada → data santri diperbarui (upsert). Enrollment periode aktif dibuat/diubah ke `kelas` tersebut dengan aturan pindah rombel (FR-MST-07).
- Import ditolak seluruhnya bila tidak ada periode aktif.

### 9.3 Template & Import Nilai
**Unduh template** untuk satu (rombel, mapel) pada periode aktif. File berisi:
- Sheet `Nilai`: kolom `nis`, `nama`, `nilai`. Baris = santri yang ter-enroll, terurut nama. Nilai yang sudah terisi ikut tampil. Sel `nilai` diberi **data validation** (bilangan bulat 2–9). Kolom `nis` dan `nama` dikunci (proteksi sheet tanpa password).
- Sheet `_meta` (tersembunyi): `period_id`, `classroom_id`, `subject_id`, `generated_at`.
- Judul di baris atas atau nama file memuat rombel dan mapel agar mudah dikenali, contoh `nilai_1A_tauhid.xlsx`.

**Aturan import:**
- Sistem mencocokkan `_meta` dengan halaman tujuan. Beda rombel/mapel/periode → ditolak: "File ini untuk kelas X mapel Y, bukan kelas ini."
- Baris dicocokkan lewat **NIS** (bukan urutan baris). NIS yang tidak ter-enroll di rombel itu → error. NIS ganda dalam file → error.
- `nilai` kosong → **dilewati** (tidak menghapus nilai yang sudah ada). Penghapusan hanya lewat web grid.
- `nilai` valid: angka bulat 2–9, termasuk `7` atau `7.0` dari Excel. `7.5`, `1`, `10`, `abc` → error dengan nomor baris dan isi sel.
- Santri yang tidak ada di file dibiarkan apa adanya.
- Pengguna wajib berhak atas (rombel, mapel) tersebut.

### 9.4 Export Rekap Nilai per Kelas (Admin)
**Pilihan di layar:**
- Periode (default aktif; periode lama boleh, read-only).
- Cakupan: satu rombel / semua rombel satu tingkat / semua rombel.
- Gaya header mapel: **Nama Mapel** (untuk arsip) atau **Kode Mapel** (untuk Mail Merge Word).
- Sebelum mengunduh, tampilkan ringkasan kelengkapan (sel nilai kosong per rombel) sebagai peringatan, bukan penghalang.

**Struktur file `.xlsx`:**
- Satu sheet per rombel (nama sheet = nama rombel, contoh `1A`). Cakupan banyak rombel = banyak sheet dalam satu file. Nama file contoh `rekap_nilai_2026-2027_genap_1A.xlsx`.
- **Baris 1 = header** (tanpa judul/merge cell di atas, agar langsung bisa dipakai sebagai sumber Mail Merge). Data mulai baris 2.
- Kolom, berurutan:

| Kelompok | Kolom |
|---|---|
| Identitas | No, NIS (teks), Nama Lengkap, Nama Arab, L/P, Kelas, Kelas Dapodik, Wali Kelas, Wali Kelas (Arab) |
| Mapel | Seluruh mapel **aktif** di master, ditambah mapel nonaktif yang memiliki nilai pada periode yang diexport; urut `sort_order`; daftar kolom sama di setiap sheet dalam satu file |
| Absensi | Sakit, Izin, Alpa |

- Isi sel mapel: angka `2`–`9` (tipe numerik) bila dinilai; `-` bila mapel tidak ada di kurikulum tingkat rombel itu; **kosong** bila ada di kurikulum tetapi belum dinilai (BR-10).
- Isi absensi: angka; kosong bila belum diisi wali kelas.
- Santri terurut nama (A–Z). Hanya santri ter-enroll di rombel pada periode tersebut.
- Wali Kelas/Wali Kelas (Arab) kosong bila belum ditetapkan.

Contoh (Tingkat 1, mapel `nahwu` tidak diajarkan):

| No | NIS | Nama Lengkap | Nama Arab | L/P | Kelas | Kelas Dapodik | Wali Kelas | Wali Kelas (Arab) | tauhid | fiqh | nahwu | Sakit | Izin | Alpa |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 252607045 | Dafa Ghaitsa Budiman | دفا غيثا بوديمان | L | 1A | 7A | Ust. Abdul Mughni | عبد المغني | 8 | 7 | - | 2 | 0 | 0 |

---

## 10. Daftar Layar & Navigasi

**Umum:** Login, Ganti Password, header berisi badge periode + nama pengguna + keluar.

**Admin** (sidebar):
- Dashboard (checklist onboarding, progres)
- Periode (daftar, aktivasi, clone, kenaikan kelas)
- Master → Kelas, Mapel, Kurikulum Tingkat, Guru, Santri (+ Import)
- Penempatan → Wali Kelas, Plotting Guru
- Input Nilai & Absensi (memilih kelas lalu grid yang sama dengan guru)
- Monitoring (matriks rombel × mapel)
- Export Rekap
- Log Aktivitas (Milestone 6)

**Guru / Wali Kelas** (sidebar):
- Kelas Saya (kartu rombel + mapel)
- Input Nilai (grid) dan Import Nilai
- Absensi *(hanya muncul bila wali kelas)*
- Kelengkapan Kelas *(hanya muncul bila wali kelas)*

Prinsip UI: tombol aksi utama jelas ("Simpan"), konfirmasi untuk aksi berisiko (aktivasi periode, clone, hapus), pesan sukses/gagal berbahasa Indonesia, tabel besar mendukung pencarian dan paginasi 50 baris (kecuali grid input yang menampilkan satu kelas penuh).

---

## 11. Kebutuhan Non-Fungsional

**Skala:** ±1.000+ santri, 40–50 rombel, 30+ guru, belasan hingga puluhan mapel, penginputan bersamaan saat musim ujian.

**Performa (jaringan lokal):**
- Halaman umum < 2 detik; grid input satu kelas < 1 detik; simpan satu kelas < 1 detik.
- Import satu kelas < 5 detik; export seluruh kelas < 30 detik (sinkron, tanpa antrean; gunakan chunk/eager loading, hindari N+1).

**Integritas data:** CHECK dan UNIQUE constraint di database; semua operasi massal dalam `DB::transaction`; simpan nilai memakai `upsert` pada kunci unik agar aman saat penginputan bersamaan (last write wins per sel).

**Keamanan:** lihat 8.9. Tambahan: password di-hash bcrypt; session `file` dengan lifetime 120 menit; cookie `secure` otomatis di HTTPS; tidak ada endpoint publik selain login; database tidak boleh terbuka ke luar server.

**Deployment lokal:**
- PC server dengan IP statis di LAN; Windows (Laragon/Herd) atau Linux (Nginx + PHP-FPM). Port database tidak dibuka ke jaringan.
- Build produksi: `composer install --no-dev -o`, `npm run build`, `php artisan migrate --force`, `config:cache`, `route:cache`, `view:cache`.
- Akses jarak jauh sementara: `cloudflared tunnel --url http://localhost:<port>` (lihat 6.3).
- Font Arab untuk tampilan nama santri/guru: gunakan Noto Naskh Arabic (atau Noto Sans Arabic) lewat paket npm `@fontsource`, di-bundle oleh Vite. Tanpa CDN dan tanpa Google Fonts.

**Kompatibilitas browser:** Chrome/Edge/Firefox terbaru, termasuk di HP Android.

**Locale:** zona waktu `Asia/Jakarta`, locale `id`, pesan validasi Indonesia (`lang/id`).

---

## 12. Struktur Kode yang Disarankan

```
app/
  Http/
    Controllers/Admin/...        (Period, Classroom, Subject, Curriculum, Teacher, Student, Plotting, Recap)
    Controllers/Teacher/...      (MyClasses, GradeEntry, GradeImport, Attendance, HomeroomStatus)
    Middleware/                  (EnsureRole, EnsurePeriodIsActive, ForcePasswordChange, AdminLanOnly)
    Requests/                    (FormRequest per aksi tulis)
  Models/
  Policies/                      (GradePolicy, AttendancePolicy)
  Services/
    CurrentPeriod.php            (periode aktif; override read-only untuk admin lewat session)
    PeriodService.php            (aktivasi, clone, kenaikan kelas)
    GradeService.php             (bulk upsert nilai, validasi akses)
    CompletenessService.php      (hitung status kelengkapan)
  Imports/                       (TeacherImport, StudentImport, GradeImport)
  Exports/                       (GradeTemplateExport, ClassRecapExport, ImportErrorExport)
  Console/Commands/BackupDatabase.php
database/migrations, database/seeders
resources/views (Blade), resources/js (Alpine), resources/css (Tailwind)
tests/Feature, tests/Unit
docs/decisions.md, docs/restore.md
```

Konvensi penting:
- **Jangan memakai global scope otomatis** untuk periode. Gunakan `CurrentPeriod::id()` secara eksplisit dan scope `forPeriod($id)` supaya mudah diuji dan tidak menimbulkan kebocoran data lintas periode.
- Middleware `EnsurePeriodIsActive` + pengecekan di Service menjadi lapis ganda untuk BR-07.
- Hak akses nilai/absensi diperiksa lewat Policy, bukan hanya menyembunyikan menu.

---

## 13. Strategi Pengujian

Feature test minimum (tiap butir = minimal satu test):

1. Nilai: 2 dan 9 diterima; 1, 10, 0, 7.5, "a" ditolak di form dan import; database menolak insert langsung di luar 2–9.
2. Hak akses: guru lain mendapat 403 pada (rombel, mapel) yang bukan miliknya; guru mapel tidak bisa export; wali kelas hanya mengakses rombel perwaliannya.
3. Periode: hanya satu aktif; operasi tulis ke periode non-aktif ditolak; clone menyalin tiga tabel yang benar, tidak menyalin nilai/absensi, menolak duplikasi.
4. Kurikulum: mapel di luar kurikulum tingkat tidak bisa di-plot, tidak bisa dinilai, dan muncul `-` di export.
5. Import nilai: cocok NIS, abaikan sel kosong, tolak `_meta` yang salah, all-or-nothing saat ada error, laporan error benar.
6. Import santri/guru: upsert, validasi kelas, NIS duplikat, NIS dengan nol di depan terjaga.
7. Export: struktur kolom, header Nama vs Kode, `-` vs kosong, urutan mapel, satu sheet per rombel, Wali Kelas terisi.
8. Absensi: hanya bilangan bulat ≥ 0; sel kosong menjadi 0.
9. Autentikasi: akun nonaktif ditolak, throttle login, paksa ganti password.
10. Pindah rombel: dalam tingkat sama mempertahankan nilai; lintas tingkat diblokir bila sudah ada nilai.

---

## 14. Milestone & Definition of Done

| Milestone | Isi | DoD |
|---|---|---|
| **M1 — Fondasi** | Proyek Laravel 13, koneksi PostgreSQL, Vite/Tailwind/Alpine lokal, konfigurasi proxy Cloudflare, 11 migration + constraint, model + relasi, `AdminSeeder`, layout dasar, login + paksa ganti password, role middleware | `migrate:fresh --seed` sukses; admin bisa login dan ganti password; test constraint database lulus |
| **M2 — Master & Periode** | FR-PRD-*, FR-MST-*, FR-PLT-*, `CurrentPeriod`, `PeriodService` (aktivasi + clone), import guru/santri, checklist onboarding | Admin bisa menyiapkan seluruh data dari nol sampai plotting; clone berjalan; test periode dan import lulus |
| **M3 — Guru Mapel** | FR-GRD-* (Kelas Saya, web grid, template + import) | Guru mengisi nilai satu kelas lewat grid dan Excel; semua validasi 2–9 teruji |
| **M4 — Wali Kelas** | FR-ABS-*, dashboard kelengkapan, FR-ADM-01/02/04 | Wali kelas mengisi absensi dan melihat status; admin melihat matriks progres |
| **M5 — Export** | FR-ADM-03 sesuai Bagian 9.4 | File Excel terbuka benar di Excel dan bisa dipakai sebagai sumber Mail Merge Word |
| **M6 — Penyempurnaan** | Backup terjadwal, activity log, kenaikan kelas, `DemoSeeder`, `ADMIN_LAN_ONLY`, `docs/restore.md`, FR-ABS-03 | Backup harian terbukti jalan dan bisa di-restore; kenaikan kelas teruji |

Prompt pembuka yang disarankan di Antigravity:
> Baca `PRD.md` seluruhnya. Kerjakan **Milestone 1** saja sesuai Bagian 14. Ikuti aturan di Bagian 0. Setelah selesai, jalankan `migrate:fresh --seed` dan test, lalu laporkan hasilnya sebelum melanjutkan.

---

## 15. Asumsi, Keputusan Desain, & Risiko

Keputusan berikut diambil agar PRD tuntas. Ubah bila tidak sesuai.

| # | Keputusan | Alasan |
|---|---|---|
| A1 | Nilai **belum diisi = sel kosong**, mapel **tidak ada di tingkat = `-`** | Membedakan "belum dinilai" dari "memang tidak diajarkan", sehingga kekurangan data terlihat saat export |
| A2 | Kurikulum per tingkat bersifat **global** (tidak per periode) | Sesuai penjelasan bahwa kurikulum seragam per tingkat. Risiko: mengubah kurikulum juga mengubah tampilan export periode lama |
| A3 | Import **all-or-nothing** | Mencegah data setengah masuk dan sulit dilacak |
| A4 | Admin boleh menginput/mengoreksi nilai semua kelas (periode aktif) | Admin berhak penuh dan perlu koreksi cepat saat guru berhalangan |
| A5 | Wali kelas melihat **status** kelengkapan saja, bukan nilai mapel lain | Sesuai kebutuhan "memantau kelengkapan" |
| A6 | Export menyertakan kolom Nama Arab santri dan Wali Kelas (Arab) | Disiapkan untuk Mail Merge Word. Hapus bila tidak dibutuhkan |
| A7 | Kenaikan kelas tahunan dikerjakan di Milestone 6 | Dibutuhkan sebelum tahun ajaran berikutnya, tidak menghambat penggunaan semester ini |
| A8 | Mengosongkan sel di web grid menghapus nilai, sedangkan sel kosong di import dilewati | Penghapusan hanya lewat jalur yang disengaja |
| A9 | Kelas Dapodik berupa teks bebas tanpa validasi format | Contoh data bervariasi (`1A`, `7A`) |
| A10 | `classrooms` dan `subjects` punya `is_active` | Diperlukan agar BR-14 (penonaktifan) konsisten dengan skema |
| A11 | Export memuat mapel aktif + mapel nonaktif yang punya nilai di periode itu | Mapel yang sudah dinonaktifkan tidak boleh membuat nilai periode lama hilang dari export |
| A12 | Hak input nilai wali kelas = hanya mapel yang ia ampu | Selaras BR-05; wali kelas hanya mengisi absensi dan memantau |
| A13 | Nama kelas wajib diawali digit tingkat, dijaga FormRequest dan CHECK `left(name,1) = grade_level::text` | Mencegah kelas salah tingkat walau data masuk lewat jalur lain |

**Risiko**
- Kompatibilitas `maatwebsite/excel` dengan Laravel 13 belum terverifikasi: cek saat instalasi, siapkan fallback (Bagian 6.1).
- `trycloudflare` bersifat sementara (URL berubah, tanpa SLA) dan membuka aplikasi ke internet: wajib throttle login, password kuat, dan pertimbangkan `ADMIN_LAN_ONLY`.
- Server lokal tunggal: backup harus disalin ke media kedua (flashdisk/disk lain) secara rutin, bukan hanya satu folder di PC yang sama.
- Pergantian guru atau mapel di tengah semester: plotting dapat diubah tanpa kehilangan nilai, namun periksa monitoring setelahnya.

---

## 16. Lampiran: Alur Setup Awal (Admin)

1. Login → ganti password.
2. Buat periode (contoh `2026/2027 – Ganjil`) → aktifkan.
3. Buat kelas/rombel Tingkat 1–6 (contoh `1A`–`1I`, `4A`–`4F`).
4. Buat master mapel (kode, nama Latin, nama Arab, urutan).
5. Atur matriks kurikulum: centang mapel yang berlaku di tiap tingkat.
6. Import guru (atau input manual) → bagikan username dan password awal.
7. Plotting: tetapkan wali kelas per rombel dan guru per (rombel, mapel).
8. Import santri (NIS, nama, nama Arab, L/P, kelas, kelas Dapodik).
9. Cek dashboard onboarding: semua langkah hijau → beri tahu guru untuk mulai input nilai.
10. Akhir semester: pantau matriks kelengkapan → Export Rekap → buat semester Genap lewat tombol Clone.