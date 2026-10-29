# Dokumentasi Posyandu Web Client

Client adalah aplikasi web Next.js untuk admin, bidan, kader, dan orang tua. Proyek memakai Next.js 16.4.3, React 19.3.1, TypeScript, Bun 1.4.4, TanStack Query, Axios, dan konfigurasi Progressive Web App.

Daftar fungsi per portal dan integrasinya dengan backend dirangkum pada [functional requirement baseline](./FUNCTIONAL_REQUIREMENTS.md).

Untuk gambaran alur lintas repo, setup container, integrasi eksternal, dan batas deployment saat ini, lihat [catatan arsitektur dan infrastruktur](./ARSITEKTUR_DAN_INFRASTRUKTUR.md).

## Struktur proyek

| Folder/file | Isi |
| --- | --- |
| `app/` | Halaman dan layout berdasarkan App Router Next.js |
| `components/` | Komponen UI bersama dan komponen fitur |
| `hooks/` | Hook React, termasuk pengambilan data dan notifikasi |
| `service/` | Request API yang dikelompokkan berdasarkan fitur |
| `interfaces/` | Tipe TypeScript untuk request dan response |
| `providers/` | TanStack Query, dialog konfirmasi, dan notifikasi |
| `middleware.ts` | Pemeriksaan session/role dan proxy request API |
| `public/` | Aset statis dan ikon; plugin PWA menyiapkan service worker |
| `next.config.ts` | Header keamanan, PWA, dan kebijakan koneksi browser |

Folder route group seperti `(auth)`, `(pages)`, dan `(bayi)` mengelompokkan file tanpa menambah segmen URL. Parameter dinamis seperti `[id]` menjadi bagian URL.

## Portal dan fitur

| Portal | Halaman dan fitur utama |
| --- | --- |
| Admin (`/admin`) | Login dan OTP, kelola akun, kelola Posyandu, laporan bayi baru lahir |
| Bidan (`/bidan`) | Data dan pendaftaran bayi, edukasi, imunisasi, inventaris, jadwal konsultasi, jadwal Posyandu, profil |
| Kader (`/kader`) | Data dan pendaftaran bayi, edukasi, imunisasi, inventaris, jadwal konsultasi, jadwal Posyandu, profil |
| Orang tua (`/orangtua`) | Data anak, booking layanan dan antrean, edukasi, imunisasi, jadwal Posyandu, konsultasi, profil |

Setiap portal memiliki halaman login, verifikasi OTP, dan lupa password. Route aktual dapat dilihat langsung pada file `page.tsx` di `app/`.

## Menjalankan aplikasi

Dari folder `Posyandu_Web_Client`:

```bash
bun install --frozen-lockfile
bun run dev
```

Development server berjalan di `http://localhost:3001` dan skrip akan menolak runtime selain Bun 1.4.4.

Perintah lain:

```bash
bun run build
bun run start
bun run lint
```

`build` menghasilkan bundle produksi; `start` menjalankan hasil build. Jalankan keduanya dari folder client.

## Environment dan koneksi API

| Variabel | Kegunaan |
| --- | --- |
| `NEXT_PUBLIC_API_URL` | Origin server API, tanpa suffix `/api`. Saat development, middleware memakai `http://localhost:3000` jika nilai ini tidak diatur. |
| `NEXT_PUBLIC_TURNSTILE_SITE_KEY` | Site key Cloudflare Turnstile untuk widget login/registrasi; widget memberi pesan belum dikonfigurasi bila kosong. |

Untuk pengembangan lokal dengan server pada port `3000`, buat environment client yang memuat:

```dotenv
NEXT_PUBLIC_API_URL=http://localhost:3000
```

Middleware Next.js meneruskan request `/api/*` ke origin tersebut. Karena itu, jalankan server pada port yang sama atau ubah nilai `NEXT_PUBLIC_API_URL`. Origin server API juga perlu mengizinkan `http://localhost:3001` saat development.

Hook WebSocket memakai `NEXT_PUBLIC_API_URL` untuk membentuk alamat `/ws`. Pastikan server dan client memakai host serta skema yang sesuai. Gunakan HTTPS/WSS saat deployment melalui HTTPS.

## Alur data dan autentikasi

- `service/` memuat operasi API per fitur, misalnya auth, anak, imunisasi, jadwal, edukasi, dan inventaris.
- TanStack Query menyimpan hasil request di cache dan menyegarkan data saat koneksi pulih. Hook pada `hooks/query/` membungkus query dan mutation tiap fitur.
- `interfaces/` menampung bentuk data TypeScript yang dipakai komponen dan service.
- Axios mengirim cookie session dengan `withCredentials`. Untuk mutasi, interceptor mengambil token CSRF dan menambahkan `X-CSRF-Token`; bila token ditolak, client mencoba mengambil token baru dan mengulang request sekali.
- `middleware.ts` memvalidasi session untuk route portal, mengarahkan pengguna ke portal sesuai role, dan meneruskan request API ke server.
- `NotificationWebSocketProvider` mengaktifkan listener WebSocket dan langganan Web Push pada layout yang menggunakannya.

Endpoint backend dan skema request/response dapat dilihat pada Swagger server di `/api/docs`. Jika mengubah kontrak API, periksa service dan tipe terkait di client.

## PWA dan notifikasi

`app/manifest.ts` mendefinisikan nama aplikasi, mode tampilan standalone, warna tema, dan ikon PWA. Konfigurasi PWA berada pada `next.config.ts`; notifikasi browser memakai WebSocket, Web Push, dan izin notifikasi browser.

Untuk menguji notifikasi, browser harus mendukung service worker dan pengguna perlu memberi izin notifikasi. Nilai VAPID serta konfigurasi push berada di server; client tidak boleh menyimpan private key.

## Pemeliharaan

- Tambahkan halaman pada `app/` sesuai portal dan kelompok route yang sudah ada.
- Pisahkan request API dari komponen ke `service/`, lalu letakkan tipe terkait di `interfaces/`.
- Gunakan TanStack Query melalui hook yang sesuai di `hooks/query/` agar cache dan invalidasi tetap konsisten.
- Periksa middleware bila menambah role atau mengubah akses portal.
- Atur domain backend melalui `NEXT_PUBLIC_API_URL`; periksa juga policy koneksi CSP di `next.config.ts` bila domain berubah.
