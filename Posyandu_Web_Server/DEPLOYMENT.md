# Panduan Deployment Posyandu Web Server

Panduan ini menjelaskan persiapan dan urutan rilis berdasarkan konfigurasi yang tersedia di `Posyandu_Web_Server`. Panduan ini tidak berarti layanan production sudah diprovisikan atau deployment sudah dijalankan.

## Model deployment saat ini

Compose production menjalankan API dan PostgreSQL dalam satu stack Podman pada host yang sama. PostgreSQL dipublikasikan hanya ke loopback host; API mengaksesnya melalui hostname internal `postgres`. Port aplikasi di dalam container tetap `3000`, sedangkan `APP_HOST_PORT` mengatur port host.

Target production di Makefile mengambil secret dari Infisical environment `prod`. Migrasi dijalankan dari host menggunakan Bun dan `DATABASE_URL` host-reachable. Folder migrasi tidak disalin ke image production.

Konfigurasi ini belum menyediakan variasi Compose untuk managed PostgreSQL. Jangan mengganti `DATABASE_URL` saja untuk menggunakan managed database: Compose saat ini membentuk ulang URL internal dari `DATABASE_USER`, `DATABASE_PASSWORD`, dan `DATABASE_NAME`, serta mensyaratkan service `postgres` sehat.

## Prasyarat operasional

- Host Linux dengan Podman dan Podman Compose.
- Bun `1.4.4` dan dependensi project terpasang pada host untuk menjalankan migrasi dan backfill.
- Infisical CLI sudah login dan terhubung ke project server. Pastikan environment `prod` tersedia.
- Domain API dan frontend, DNS, reverse proxy dengan TLS, aturan firewall, dan akses WebSocket `/ws` melalui proxy.
- PostgreSQL production, prosedur backup/restore yang sudah diuji, serta kapasitas disk untuk volume dan backup.
- Provisioning layanan Cloudflare R2, Cloudflare KV, dan Turnstile; Google OAuth, Resend, serta Upstash bila digunakan.

## Secret dan konfigurasi production

Simpan nilai production di Infisical environment `prod`. Jangan menaruh secret production di `.env`, dokumentasi, Git, atau client. `.env.example` di repository server hanya untuk development lokal.

### Aplikasi, URL, dan keamanan

| Variabel               | Persiapan                                                                                                                                                                               |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CORS_ORIGIN`          | Origin frontend production dengan HTTPS, tanpa path.                                                                                                                                    |
| `TRUSTED_ORIGINS`      | Daftar origin frontend yang dipercaya, dipisahkan koma.                                                                                                                                 |
| `BETTER_AUTH_URL`      | URL publik API/backend dengan HTTPS. Samakan dengan URL yang dipakai client dan konfigurasi OAuth.                                                                                      |
| `BETTER_AUTH_SECRET`   | Secret acak unik, sedikitnya 32 karakter.                                                                                                                                               |
| `CSRF_SECRET`          | Secret acak unik sedikitnya 32 karakter; jangan gunakan nilai development default.                                                                                                      |
| `TURNSTILE_SECRET_KEY` | Secret Cloudflare Turnstile yang aktif. Nilai `skip` hanya untuk development.                                                                                                           |
| `CRON_SECRET`          | Secret acak untuk endpoint pekerjaan terjadwal.                                                                                                                                         |
| `DATA_ENCRYPTION_KEY`  | 32 byte acak dalam 64 karakter hex atau base64. Gunakan key yang sama dengan data terenkripsi yang sudah ada; kehilangan atau penggantian tanpa re-enkripsi membuat data tidak terbaca. |
| `NODE_ENV`, `PORT`     | Compose menetapkan `production` dan port internal `3000`. Jangan mengubah port internal tanpa memperbarui Compose dan health check.                                                     |
| `APP_HOST_PORT`        | Opsional. Port host untuk API; default `3000`.                                                                                                                                          |

### Database dan migrasi

Untuk PostgreSQL dalam Compose, sediakan `DATABASE_USER`, `DATABASE_PASSWORD`, `DATABASE_NAME`, dan opsional `DB_PORT` serta konfigurasi pool (`DB_POOL_MAX`, `DB_POOL_IDLE_TIMEOUT_MS`, `DB_POOL_CONNECTION_TIMEOUT_MS`). Gunakan password alfanumerik atau pastikan karakter khusus di-encode dengan benar karena Compose membentuk URL internal dari nilai tersebut.

Sediakan juga `DATABASE_URL` yang dapat dijangkau **dari host** untuk perintah migrasi. Jika PostgreSQL berjalan di Compose, bentuknya menggunakan `localhost` dan port `DB_PORT` yang dipetakan, misalnya `postgresql://<user>:<password>@localhost:5432/<database>`. API di dalam container tidak memakai URL host ini; Compose membuat URL internal dengan hostname `postgres`.

### Integrasi aplikasi

| Layanan                     | Variabel / tindakan                                                                                                                                                                                                                                       |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Google OAuth                | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`; daftarkan callback URL sesuai konfigurasi Better Auth dan domain production.                                                                                                                                  |
| Email Resend                | `RESEND_API_KEY`, `SENDER_EMAIL`; verifikasi domain pengirim sebelum mengaktifkan email OTP.                                                                                                                                                              |
| Informasi kontak            | `CONTACT_NAME`, `CONTACT_WHATSAPP`, `CONTACT_EMAIL`.                                                                                                                                                                                                      |
| Cloudflare R2               | `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`; isi `R2_PUBLIC_URL` bila media disajikan melalui public domain. Batasi token pada bucket yang diperlukan.                                                                  |
| Cloudflare KV               | `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_KV_NAMESPACE_ID`. Ketiganya harus sesuai account, API token, dan namespace yang diprovisikan. Jika kosong atau tidak lengkap, cache jatuh ke memori proses dan tidak dibagi antar-instance.  |
| Upstash Redis               | `UPSTASH_REDIS_REST_URL` dan `UPSTASH_REDIS_REST_TOKEN` untuk Better Auth secondary storage dan distributed rate limiter. Mengisi `UPSTASH_REDIS_URL` saja tidak mengaktifkan adapter Redis yang saat ini digunakan.                                      |
| Web Push                    | Jika aktif, isi `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, dan `VAPID_SUBJECT` sebagai pasangan baru. Jika nonaktif, biarkan kedua key kosong. Pasangan yang sebelumnya pernah tertanam di source harus dianggap terekspos dan tidak boleh digunakan ulang. |
| Batas upload dan rate limit | Atur variabel `RATE_LIMIT_*`, `MEDIA_UPLOAD_MAX_SIZE_MB`, `MEDIA_UPLOAD_MAX_RAW_SIZE_MB`, dan `MEDIA_UPLOAD_ALLOWED_EXTENSIONS` sesuai kapasitas serta kebijakan layanan.                                                                                 |

Daftar nama dan validasi lengkap terdapat di `src/configs/env.ts` serta `docker/compose/base.yml` pada repository server. Secret Cloudflare KV bersifat opsional di schema agar development dapat berjalan tanpa layanan remote, tetapi harus diisi agar deployment production benar-benar menggunakan cache KV lintas proses. Bila Cloudflare KV tidak tersedia, aplikasi tetap dapat membaca database dan memakai fallback cache lokal.

## Urutan deployment

Jalankan dari folder repository `Posyandu_Web_Server`. Target `*-prod` mengambil secret dari Infisical environment `prod`.

1. Pastikan secret dan konfigurasi di atas sudah tersedia. Jangan menampilkan nilai secret dalam log atau output terminal yang dibagikan.
2. Buat backup PostgreSQL dan pastikan prosedur restore dapat digunakan. Pastikan backup key `DATA_ENCRYPTION_KEY` disimpan terpisah.
3. Jalankan database production dan tunggu health check PostgreSQL:

   ```bash
   make dc-prod-db-up
   ```

4. Terapkan migrasi schema dari host:

   ```bash
   make db-mig-prod
   ```

5. Jika database lama masih memiliki kolom sensitif plaintext, ikuti prosedur [enkripsi data sensitif](./SECURITY_ENCRYPTION.md). Jalankan preview, tinjau hasil agregat, pastikan backup valid, baru jalankan backfill:

   ```bash
   make data-encrypt-preview-prod
   make data-encrypt-apply-prod
   make data-encrypt-preview-prod
   ```

   Backfill hanya diperlukan bila ada data lama yang belum terenkripsi. Jangan mengirim nilai data atau key ke log/chat.

6. Build image dan jalankan stack di background; perintah menunggu health check service:

   ```bash
   make dc-prod-bg
   ```

   Untuk menjalankan di foreground, gunakan `make dc-prod-up`.

7. Periksa health endpoint dari domain API melalui reverse proxy:

   ```bash
   curl --fail https://<domain-api>/api/health/live
   curl --fail https://<domain-api>/api/health/ready
   ```

   Readiness harus berhasil setelah database tersedia. Lanjutkan dengan smoke test login/OAuth, pengiriman email, akses media R2, cache KV, upload yang diizinkan, CSRF pada mutasi, dan koneksi WebSocket dari client.

Compose tidak menjalankan migrasi otomatis. Selalu terapkan migrasi sebelum mengarahkan traffic ke image baru. Untuk rollback, siapkan image aplikasi sebelumnya dan rencana restore database; rollback kode saja belum tentu kompatibel dengan perubahan schema.

## Cache dan skalabilitas

Server memakai Cloudflare KV untuk cache-aside data yang sering dibaca. Cache memori lokal berfungsi sebagai L1 singkat dan fallback saat credential atau layanan KV tidak tersedia. KV bersifat eventually consistent; invalidasi dapat terlambat terlihat di instance/lokasi lain.

Upstash tetap terpisah dari cache aplikasi: Redis digunakan untuk secondary storage Better Auth dan rate limit lintas proses. Tanpa Redis, secondary storage Redis dinonaktifkan dan rate limiter memakai store memori proses; data utama Better Auth tetap disimpan di PostgreSQL. Untuk deployment beberapa API instance, konfigurasi Upstash. Koneksi WebSocket dan beberapa state session/cache masih berada di memori proses; notifikasi yang dibuat oleh instance berbeda tidak otomatis diteruskan ke instance pemegang koneksi. Pertahankan satu API instance sampai strategi fan-out WebSocket dan state bersama disiapkan.

## Pemeriksaan kesiapan

- [ ] Infisical environment `prod` terisi dan hanya dapat diakses operator yang berwenang.
- [ ] `DATA_ENCRYPTION_KEY` cocok dengan data terenkripsi dan backup key tersedia.
- [ ] Domain, HTTPS, CORS, trusted origin, OAuth callback, dan WebSocket proxy cocok.
- [ ] PostgreSQL, volume persisten, backup, dan restore telah diuji.
- [ ] R2 dan Cloudflare KV sudah diprovisikan; variabel KV lengkap agar cache tidak diam-diam hanya memakai memori.
- [ ] Email Resend dan Turnstile sudah diuji pada domain production.
- [ ] Redis Upstash aktif bila rate limit dan Better Auth perlu state lintas instance.
- [ ] Migrasi berhasil; backfill sudah diverifikasi bila diperlukan.
- [ ] Health checks dan smoke test berhasil setelah rilis.
- [ ] Log, pemantauan, pemilik on-call, dan prosedur rollback telah ditentukan.

## CI dan deployment otomatis

GitHub Actions saat ini hanya menjalankan unit test pada workflow CI. Hook lokal pre-commit menjalankan Prettier, ESLint, dan typecheck; pre-push menjalankan build serta unit test. Tidak ada build image atau deployment production otomatis di workflow.
