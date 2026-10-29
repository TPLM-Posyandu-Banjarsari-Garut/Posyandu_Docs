# Dokumentasi Posyandu Web Server

Server menyediakan API untuk autentikasi dan operasional Posyandu Banjarsari. Proyek memakai Bun 1.4.4, Express 5, TypeScript, Drizzle ORM, PostgreSQL, Better Auth, dan WebSocket.

Daftar fungsi yang terlihat pada API dan client dirangkum pada [functional requirement baseline](./FUNCTIONAL_REQUIREMENTS.md).

## Struktur dan alur request

`src/server.ts` memuat konfigurasi, memulai listener HTTP, lalu memasang WebSocket pada server yang sama. `src/app.ts` menyiapkan middleware keamanan, logging, autentikasi, dokumentasi API, routing, dan penanganan error.

Sebagian besar fitur mengikuti susunan ini:

1. `src/routes/` memasang path dan middleware endpoint.
2. `src/middlewares/` memeriksa session, role, validasi request, rate limit, dan audit.
3. `src/controllers/` menangani request dan response HTTP.
4. `src/services/` menjalankan aturan bisnis.
5. `src/repositories/` membaca atau mengubah data melalui Drizzle.
6. `src/db/schemas/` dan `src/db/relations.ts` mendefinisikan skema PostgreSQL.

File migrasi SQL berada di `migrations/`. Spesifikasi endpoint OpenAPI berada di `src/docs/`.

## Prasyarat

- Bun 1.4.4. Skrip `dev`, `build`, dan `start` memeriksa versi ini.
- PostgreSQL yang dapat diakses dari server.
- Infisical CLI untuk menjalankan perintah melalui konfigurasi secret yang dipakai tim.
- Podman dan Podman Compose bila menjalankan PostgreSQL serta server dalam container.

## Instalasi dan menjalankan server

Dari folder `Posyandu_Web_Server`:

```bash
bun install --frozen-lockfile
```

Perintah `make` membungkus skrip Bun dan Infisical. Environment default Makefile dan `.infisical.json` saat ini adalah `dev`. Pastikan environment tersebut memakai database lokal sebelum menjalankan server atau migrasi. Untuk ringkasan deployment dan batas infrastruktur, lihat [catatan arsitektur dan infrastruktur](./ARSITEKTUR_DAN_INFRASTRUKTUR.md).

```bash
make dev INFISICAL_ENV=dev
```

Server lokal sebaiknya memakai `PORT=3000` agar sesuai dengan default proxy API client dan pemetaan port Compose. Untuk menjalankan container server beserta PostgreSQL:

```bash
make dc-up INFISICAL_ENV=dev
```

`make dc-down` menghentikan container. `make dc-reset` juga menghapus volume PostgreSQL lokal dan seluruh datanya.

Perintah proyek lainnya:

| Perintah | Kegunaan |
| --- | --- |
| `bun run build` atau `make build` | Mengompilasi TypeScript ke `dist/` |
| `bun run start` atau `make start` | Menjalankan hasil build; `make start` memuat secret Infisical |
| `bun run typecheck` atau `make typecheck` | Memeriksa tipe TypeScript |
| `bun run lint:check` atau `make lint` | Menjalankan ESLint |
| `make db-gen` | Membuat migrasi Drizzle dari perubahan skema |
| `make db-mig INFISICAL_ENV=dev` | Menjalankan migrasi pada database dari environment yang dipilih |
| `make db-studio INFISICAL_ENV=dev` | Membuka Drizzle Studio |
| `bun run db:seed` | Mengisi data awal vaksin dan vitamin |

Gunakan `make help` untuk melihat target Makefile yang tersedia.

## Konfigurasi environment

`src/configs/env.ts` memvalidasi konfigurasi pada saat proses server mulai. Nilai secret jangan ditulis ke dokumentasi, Git, atau client. Repo server memiliki panduan [enkripsi data sensitif](./Posyandu_Web_Server/SECURITY_ENCRYPTION.md) dan [README instalasi](../Posyandu_Web_Server/README.md) yang lebih rinci.

Kelompok variabel yang diperlukan:

| Kelompok | Variabel |
| --- | --- |
| Aplikasi dan jaringan | `NODE_ENV`, `PORT`, `LOG_LEVEL`, `CORS_ORIGIN`, `TRUSTED_ORIGINS` |
| Database | `DATABASE_URL`; pool opsional: `DB_POOL_MAX`, `DB_POOL_IDLE_TIMEOUT_MS`, `DB_POOL_CONNECTION_TIMEOUT_MS` |
| Session dan login | `DATA_ENCRYPTION_KEY`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` |
| Email dan informasi kontak | `RESEND_API_KEY`, `SENDER_EMAIL`, `CONTACT_NAME`, `CONTACT_WHATSAPP`, `CONTACT_EMAIL` |
| Pembatasan request | `RATE_LIMIT_GLOBAL_*`, `RATE_LIMIT_AUTH_*`, `RATE_LIMIT_SIGNIN_*`, `RATE_LIMIT_SIGNUP_*`, `RATE_LIMIT_OTP_REQ_*`, `RATE_LIMIT_OTP_VERIFY_*`, `RATE_LIMIT_RESET_PWD_*`, `RATE_LIMIT_DELETE_ACC_*`, `RATE_LIMIT_CHANGE_PWD_*`, `RATE_LIMIT_MEDIA_UPLOAD_*`; setiap kelompok memakai akhiran `WINDOW_MINUTES` dan `MAX` |
| Penyimpanan media | `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `MEDIA_UPLOAD_MAX_SIZE_MB`, `MEDIA_UPLOAD_MAX_RAW_SIZE_MB`, `MEDIA_UPLOAD_ALLOWED_EXTENSIONS`; `R2_PUBLIC_URL` opsional |
| Cloudflare KV cache | `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_KV_NAMESPACE_ID` |
| Session dan pekerjaan terjadwal | `CRON_SECRET`, `SESSION_EXPIRES_IN`, `SESSION_UPDATE_AGE`, `SESSION_COOKIE_CACHE_ENABLED`, `SESSION_COOKIE_CACHE_MAX_AGE` |
| Integrasi opsional | Redis untuk Better Auth dan limiter: `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN`, `UPSTASH_REDIS_URL`; CAPTCHA: `TURNSTILE_SECRET_KEY`; CSRF: `CSRF_SECRET`; Web Push: `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` |

Pool PostgreSQL, Redis, CAPTCHA, CSRF, URL publik R2, dan Web Push memiliki nilai opsional atau default di kode. Untuk deployment, tetapkan nilai yang sesuai environment secara eksplisit. `DATA_ENCRYPTION_KEY` harus mewakili 32 byte dan diperlukan untuk membaca kolom terenkripsi. Simpan key ini di pengelola secret dan buat backup terpisah.

Untuk koneksi database dari host, gunakan hostname yang dapat dijangkau dari host, biasanya `localhost`. Dalam Compose, aplikasi memakai hostname service `postgres`; `DATABASE_URL` Compose dibentuk dari `DATABASE_USER`, `DATABASE_PASSWORD`, dan `DATABASE_NAME`. Pastikan environment yang dipilih memang menunjuk ke database lokal sebelum menjalankan migrasi.

## Modul API

Semua rute bisnis menggunakan prefix `/api`. Daftar prefix terpasang di `src/routes/index-routes.ts`.

| Prefix | Fitur |
| --- | --- |
| `/api/health` | Health check proses dan kesiapan database |
| `/api/users`, `/api/parents`, `/api/cadres`, `/api/midwifes` | Pengguna dan profil peran |
| `/api/posyandus` | Data Posyandu |
| `/api/childrens`, `/api/pregnancy-records` | Data anak dan catatan kehamilan |
| `/api/examinations`, `/api/examination-schedules`, `/api/examination-records` | Layanan dan jadwal pemeriksaan |
| `/api/vaccines`, `/api/immunization-records`, `/api/kipi-details` | Vaksin, imunisasi, dan KIPI |
| `/api/vitamins`, `/api/vitamin-records`, `/api/nutrition-records` | Vitamin, catatan pemberian, dan gizi |
| `/api/consultations` | Konsultasi dan antrean layanan |
| `/api/inventories` | Inventaris |
| `/api/education-categories`, `/api/educations` | Kategori dan materi edukasi |
| `/api/notifications`, `/api/push-subscriptions` | Notifikasi aplikasi dan langganan Web Push |
| `/api/medias` | Unggah dan pengelolaan media |
| `/api/trash` | Pengelolaan data terhapus |
| `/api/cron` | Pemicu pekerjaan terjadwal |

Endpoint pemeriksaan kesehatan:

- `GET /api/health` memberikan ringkasan status server.
- `GET /api/health/live` memeriksa apakah proses merespons.
- `GET /api/health/ready` memeriksa koneksi database dan mengembalikan `503` bila database belum siap.

Swagger UI tersedia di `/api/docs`, sedangkan dokumen OpenAPI JSON tersedia di `/api/docs/openapi.json`. Spesifikasi berasal dari `src/docs/`; gunakan itu untuk melihat method, parameter, skema, dan aturan autentikasi setiap endpoint.

## Autentikasi dan keamanan request

- Better Auth menangani session dan endpoint `/api/auth/*`. Endpoint `/api/auth/me` mengembalikan session pengguna aktif.
- API memeriksa session serta role pada middleware sebelum menjalankan operasi yang dibatasi.
- Browser menyertakan cookie session. CORS server mengaktifkan credentials dan membatasi origin ke `CORS_ORIGIN` serta `TRUSTED_ORIGINS`.
- Endpoint mutasi di luar `/api/auth/*` memerlukan token CSRF. Client mengambil token dari `GET /api/csrf-token` dan mengirimnya pada header `X-CSRF-Token`.
- Middleware server menerapkan rate limit, validasi request, audit, dan header keamanan.
- Beberapa identifier dan catatan sensitif dienkripsi server-side. Cakupan kolom, batas perlindungan, dan prosedur backfill dijelaskan di [panduan enkripsi](./Posyandu_Web_Server/SECURITY_ENCRYPTION.md).

Server juga menyediakan WebSocket pada `/ws` untuk notifikasi dan pembaruan slot konsultasi. Konfigurasi aplikasi client harus menunjuk ke host server yang sama.

## Perubahan skema dan data

1. Ubah definisi yang relevan di `src/db/schemas/` dan relasi di `src/db/relations.ts`.
2. Buat file migrasi dengan `make db-gen`.
3. Tinjau SQL hasil generate, lalu terapkan ke database pengembangan menggunakan `make db-mig INFISICAL_ENV=dev`.
4. Perbarui validasi, service, controller, dokumentasi OpenAPI, serta client bila kontrak API berubah.

Perubahan pada kolom terenkripsi perlu mengikuti prosedur di [panduan enkripsi](./Posyandu_Web_Server/SECURITY_ENCRYPTION.md). Jangan mengganti key enkripsi tanpa proses re-enkripsi yang direncanakan.

## Pemecahan masalah

- Server berhenti saat startup: periksa nama dan format variabel yang disebut oleh `src/configs/env.ts` serta environment Infisical yang dipilih.
- Koneksi PostgreSQL gagal: bedakan URL host (`localhost`) dari URL container (`postgres`) dan periksa apakah database siap.
- Client menerima error CORS atau cookie tidak terkirim: cocokkan origin client dengan `CORS_ORIGIN`/`TRUSTED_ORIGINS`, dan gunakan `withCredentials` pada request browser.
- Request mutasi mendapat `403`: pastikan alur mengambil `/api/csrf-token`, menyimpan cookie terkait, dan mengirim header `X-CSRF-Token`.
- Client tidak dapat tersambung: samakan `PORT` server, port proxy client, dan port host yang dipetakan Compose.
