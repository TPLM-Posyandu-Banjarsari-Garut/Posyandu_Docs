# Functional Requirement Baseline

Dokumen ini mencatat fungsi yang terlihat pada halaman client, service API, route server, dan skema validasinya. Ini adalah baseline hasil pembacaan kode, bukan spesifikasi bisnis yang sudah disahkan. Aturan yang tidak tampak di kode, seperti SLA, kebijakan klinis, atau kriteria kelulusan operasional, tidak diasumsikan di sini.

## Aktor

| Aktor | Cakupan yang terlihat |
| --- | --- |
| Pengunjung | Membuka halaman login dan alur pemulihan akun. |
| Orang tua (`parent`) | Mengelola profil keluarga dan anak, melihat informasi layanan, serta membuat dan memantau booking konsultasi. |
| Kader (`cadre`) | Mengelola data operasional Posyandu sesuai penugasan dan hak akses endpoint. |
| Bidan (`midwife`) | Mengelola data layanan kesehatan sesuai penugasan dan hak akses endpoint. |
| Pengelola Posyandu (`posyandu_admin`) dan desa (`village_admin`) | Mengelola akun, Posyandu, data, dan pemulihan data sesuai hak akses endpoint. |
| Pemanggil pekerjaan terjadwal | Memicu pekerjaan backend dengan bearer secret `CRON_SECRET`. |

Portal UI menggunakan awalan `/orangtua`, `/kader`, `/bidan`, dan `/admin`. Hak API ditentukan per route, bukan hanya dari nama portal. Nilai role `admin` juga ada di enum akun dan dikenali middleware client; namun union role pada middleware otorisasi route server tidak mencakup `admin`. Lihat [catatan perbedaan implementasi](#hal-yang-perlu-dikonfirmasi).

## Persyaratan fungsional

Label cakupan: **UI + API** berarti halaman/service client dan endpoint server tersedia; **API/service** berarti kemampuan terlihat di backend atau service client tetapi belum ditemukan halaman pengguna khusus; **UI lokal** berarti halaman ada, tetapi alur yang diperiksa belum tersambung ke API.

| ID | Persyaratan fungsional | Aktor utama | Cakupan dan bukti di repo |
| --- | --- | --- | --- |
| FR-AUTH-01 | Sistem menyediakan login dan logout menggunakan email/password serta session berbasis cookie. | Semua pengguna | UI + API: halaman login portal; Better Auth pada `/api/auth/*`. |
| FR-AUTH-02 | Sistem menyediakan verifikasi email dengan OTP dan pemulihan password menggunakan OTP/email. | Pengunjung, pengguna | UI + API: halaman OTP/lupa password pada tiap portal; service auth dan plugin Better Auth. |
| FR-AUTH-03 | Sistem menyediakan pendaftaran orang tua dan login Google melalui service autentikasi. | Orang tua | API/service: `orangTuaAuthService.ts` dan endpoint Better Auth. Keberadaan halaman pendaftaran terpisah tidak teridentifikasi dari daftar route saat ini. |
| FR-AUTH-04 | Sistem membatasi portal berdasarkan role dan menolak request API yang tidak memenuhi session, status akun, verifikasi email, atau pemeriksaan role terkait. | Semua pengguna | UI + API: `middleware.ts`, `verify-auth.ts`, dan `authorize-role.ts`. |
| FR-PROFILE-01 | Pengguna dapat melihat dan memperbarui informasi profil serta mengganti password melalui layanan autentikasi. | Pengguna aktif | UI + API/service: halaman profil portal dan service `authService.ts`. |
| FR-USER-01 | Pengelola yang berwenang dapat membuat, melihat, memperbarui, menghapus, dan memulihkan akun pengguna. | Pengelola | UI + API: `/admin/kelola-buat-akun`, `/api/users`, dan endpoint restore. |
| FR-MASTER-01 | Pengelola yang berwenang dapat membuat, melihat, memperbarui, menghapus, dan memulihkan data Posyandu. | Pengelola | UI + API: `/admin/kelola-posyandu`, `/api/posyandus`. |
| FR-STAFF-01 | Sistem menyediakan pengelolaan profil orang tua, kader, dan bidan melalui API sesuai role dan cakupan penugasan. | Pengelola, kader, bidan, orang tua | API/service: `/api/parents`, `/api/cadres`, `/api/midwifes`. |
| FR-CHILD-01 | Orang tua dapat mendaftarkan dan melihat anak yang terhubung dengan profilnya. Petugas dapat mencari, melihat, memperbarui, menghapus, dan memulihkan data anak sesuai hak akses. | Orang tua, kader, bidan, pengelola | UI + API: halaman data/pendaftaran/detail anak; `/api/childrens`; pemeriksaan relasi dan penugasan pada `AuthorizationService`. |
| FR-REPORT-01 | Pengguna berwenang dapat melihat laporan bayi baru lahir dengan filter bulan, tahun, Posyandu, pencarian, halaman, dan ringkasan. | Pengelola, kader, bidan | UI + API: `/admin/laporan-bayi-baru-lahir`; `GET /api/childrens/reports/newborns`. |
| FR-PREG-01 | Sistem menyediakan pembuatan, pembacaan, pembaruan, penghapusan, dan pemulihan catatan kehamilan. | Kader, bidan, pengelola; orang tua untuk akses baca sesuai otorisasi | API/service: `/api/pregnancy-records`; halaman khusus belum teridentifikasi pada client. |
| FR-EXAM-01 | Petugas dapat mengelola jenis pemeriksaan dan jadwal pemeriksaan; pengguna yang berhak dapat melihat jadwal. | Kader, bidan, pengelola, orang tua | UI + API: halaman jadwal Posyandu; `/api/examinations` dan `/api/examination-schedules`. |
| FR-EXAM-02 | Petugas dapat mencatat dan memperbarui hasil pemeriksaan anak, serta membaca riwayat sesuai hak akses. | Kader, bidan, pengelola; orang tua untuk akses baca sesuai otorisasi | API/service: `/api/examination-records`; UI khusus bergantung pada halaman detail bayi. |
| FR-HEALTH-01 | Petugas dapat mengelola catatan gizi dan pengukuran anak; orang tua dapat melihat catatan terkait anak yang dapat diaksesnya. | Kader, bidan, pengelola, orang tua | UI + API/service: service nutrisi dan halaman detail anak; `/api/nutrition-records`. |
| FR-IMM-01 | Petugas dapat mengelola katalog vaksin dan catatan imunisasi; orang tua dapat melihat riwayat imunisasi anak. | Kader, bidan, pengelola, orang tua | UI + API: daftar/riwayat imunisasi; `/api/vaccines` dan `/api/immunization-records`. |
| FR-IMM-02 | Sistem menyediakan pencatatan dan pengelolaan detail KIPI yang terkait dengan imunisasi. | Kader, bidan, pengelola | API/service: `/api/kipi-details`; halaman khusus tidak teridentifikasi dari daftar route. |
| FR-VIT-01 | Petugas dapat mengelola katalog vitamin dan catatan pemberian vitamin; pengguna yang berhak dapat melihat catatan terkait anak. | Kader, bidan, pengelola, orang tua untuk akses baca sesuai otorisasi | API/service: `/api/vitamins` dan `/api/vitamin-records`; service tersedia pada client. |
| FR-CONS-01 | Orang tua dapat melihat slot yang tersedia dan membuat booking untuk layanan kehamilan, tumbuh kembang anak, atau layanan umum. Booking dapat mengaitkan Posyandu, bidan, anak atau catatan kehamilan, serta waktu layanan. | Orang tua | UI + API: alur booking bertahap pada `/orangtua/booking-layanan`; `/api/consultations/slots/available` dan `POST /api/consultations`. |
| FR-CONS-02 | Sistem memberi nomor antrean untuk booking dan memungkinkan pengguna melihat riwayat/status booking. Petugas yang berwenang dapat mengelola status dan mengirim pemberitahuan booking. | Orang tua, kader, bidan, pengelola | UI + API: `/orangtua/lihat-antrean`, `/kader/jadwal-konsultasi`, `/bidan/jadwal-konsultasi`; `/api/consultations`. |
| FR-EDU-01 | Pengguna dapat melihat kategori dan materi edukasi. Petugas yang berwenang dapat membuat, memperbarui, menghapus, dan memulihkan materi serta kategori. | Orang tua, kader, bidan, pengelola | UI + API: daftar/baca/tulis edukasi; `/api/education-categories` dan `/api/educations`. |
| FR-INV-01 | Petugas dapat membuat, melihat, memperbarui, menghapus, dan memulihkan data inventaris. | Kader, bidan, pengelola | UI + API: halaman inventaris; `/api/inventories`. |
| FR-NOTIF-01 | Pengguna dapat melihat notifikasi, menghitung notifikasi belum dibaca, menandai satu notifikasi dibaca, atau menandai semuanya dibaca. | Semua pengguna terautentikasi | UI + API: provider/hook notifikasi; `/api/notifications`. |
| FR-NOTIF-02 | Sistem menyediakan langganan dan penghentian Web Push serta pengiriman notifikasi real-time melalui WebSocket. | Pengguna terautentikasi | UI + API: provider notifikasi; `/api/push-subscriptions`; WebSocket `/ws`. |
| FR-MEDIA-01 | Sistem dapat mengunggah sampai lima file per request, memeriksa batas ukuran/ekstensi konfigurasi, menampilkan metadata, dan menghapus media. | Pengguna terautentikasi | API/service: `/api/medias`; file disimpan di R2 dan metadatanya di PostgreSQL. |
| FR-TRASH-01 | Pengelola dapat melihat daftar data terhapus dan memulihkan item berdasarkan jenis serta ID. | Pengelola | API/service: `/api/trash`. |
| FR-JOB-01 | Pemanggil terjadwal dengan bearer `CRON_SECRET` dapat memicu pekerjaan backend, termasuk pengingat booking, tindak lanjut, kedaluwarsa booking, dan pembersihan harian. | Pemanggil pekerjaan terjadwal | API/service: `POST /api/cron/trigger` dan `CronService`. |
| FR-OPS-01 | Sistem menyediakan endpoint health, liveness, dan readiness; readiness memeriksa koneksi database. | Operator dan monitoring | API: `GET /api/health`, `/api/health/live`, `/api/health/ready`. |

## Ringkasan portal client

| Portal | Fungsi yang tampak di halaman |
| --- | --- |
| Admin | Login/OTP/pemulihan akun, pengelolaan akun, pengelolaan Posyandu, laporan bayi baru lahir. |
| Bidan | Data anak, edukasi, imunisasi, inventaris, jadwal konsultasi, jadwal Posyandu, profil. |
| Kader | Data anak, edukasi, imunisasi, inventaris, jadwal konsultasi, jadwal Posyandu, profil. |
| Orang tua | Data dan riwayat anak, booking/antrean konsultasi, edukasi, riwayat imunisasi, jadwal Posyandu, konsultasi, profil. |

## Hal yang perlu dikonfirmasi

- **Role `admin`:** enum akun dan middleware client mengenali `admin`, tetapi `authorizeRoles` pada server hanya menerima `posyandu_admin`, `village_admin`, `midwife`, `cadre`, dan `parent`. Tetapkan apakah `admin` merupakan alias, role tersendiri dengan akses API, atau role lama.
- **Kelola Posyandu orang tua:** halaman `/orangtua/kelola-posyandu` memakai state/data lokal pada komponen dan belum terlihat menggunakan service API. Jangan anggap perubahan di halaman tersebut tersimpan ke server.
- **Imunisasi orang tua:** route `/orangtua/buat-data-imunisasi` saat ini membaca data anak, vaksin, catatan imunisasi, dan nutrisi untuk menampilkan riwayat. Endpoint tulis `/api/immunization-records` membatasi operasi tulis ke petugas/pengelola.
- **Fitur API tanpa halaman khusus:** catatan kehamilan, detail KIPI, pengelolaan media, trash, dan pemicu cron tersedia di backend, tetapi halaman pengguna khusus tidak terlihat pada client.
- **Aturan detail:** daftar method, schema, filter, dan otorisasi endpoint dapat berubah. Gunakan Swagger `/api/docs`, route server, dan validasi sebagai sumber teknis untuk kriteria penerimaan per endpoint.

Untuk menjadikan baseline ini spesifikasi yang disetujui, pemilik produk perlu menetapkan aturan bisnis dan acceptance criteria yang belum dinyatakan oleh kode.
