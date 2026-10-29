# Dokumentasi Posyandu Web

Dokumentasi ini merangkum cara kerja dan pengembangan dua repositori aplikasi:

- [Posyandu Web Server](./Posyandu_Web_Server.md)
- [Posyandu Web Client](./Posyandu_Web_Client.md)
- [Catatan arsitektur dan infrastruktur](./ARSITEKTUR_DAN_INFRASTRUKTUR.md)
- [Functional requirement baseline](./FUNCTIONAL_REQUIREMENTS.md)

## Dokumen yang dipindahkan dari repo

- [Panduan enkripsi server](./Posyandu_Web_Server/SECURITY_ENCRYPTION.md)
- [Panduan agent client](./Posyandu_Web_Client/AGENTS.md)
- [Rujukan Claude client](./Posyandu_Web_Client/CLAUDE.md)

`AGENTS.md` dan `CLAUDE.md` disimpan di sini sesuai pemindahan dokumentasi. Karena keduanya tidak lagi berada di root repo client, tool coding tidak otomatis memuat instruksi tersebut.

Kode server berada di `../Posyandu_Web_Server`, sedangkan kode client berada di `../Posyandu_Web_Client`. Perintah instalasi dan pengembangan harus dijalankan dari folder proyek masing-masing.

## Menjalankan aplikasi lokal

1. Siapkan PostgreSQL dan konfigurasi environment server. Server memvalidasi konfigurasi saat mulai, jadi isi secret melalui Infisical atau environment lokal sesuai [panduan server](./Posyandu_Web_Server.md#konfigurasi-environment).
2. Jalankan server pada port `3000` agar cocok dengan alamat backend default yang dipakai client saat development. Bila memilih port lain, atur `PORT` server dan `NEXT_PUBLIC_API_URL` client ke port yang sama.
3. Jalankan client pada port `3001` dan set `NEXT_PUBLIC_API_URL` ke origin server, misalnya `http://localhost:3000`.

Client meneruskan request `/api/*` ke server melalui middleware Next.js. Server mengizinkan origin `http://localhost:3001` saat `NODE_ENV=development`; untuk origin lain, sesuaikan `CORS_ORIGIN` atau `TRUSTED_ORIGINS`.

## Peta integrasi

| Bagian | Server | Client |
| --- | --- | --- |
| API HTTP | `/api/*`, didaftarkan di `src/routes/index-routes.ts` | Request melalui `service/` dan proxy di `middleware.ts` |
| Autentikasi | Better Auth, session cookie, endpoint `/api/auth/*` | Halaman login per role, pemeriksaan session, logout |
| Perubahan data | Token CSRF pada endpoint `/api/csrf-token` | Axios menambahkan `X-CSRF-Token` untuk request mutasi |
| Notifikasi | WebSocket `/ws` dan Web Push | Provider notifikasi, hook WebSocket, dan service worker PWA |
| Kontrak API | Swagger UI pada `/api/docs`, OpenAPI JSON pada `/api/docs/openapi.json` | Tipe data pada `interfaces/` dan service API per modul |

Dokumentasi API yang dihasilkan server menjadi rujukan untuk path, parameter, dan bentuk request/response. Dokumen ini menjelaskan susunan proyek dan alur kerja lokal.
