# Enkripsi data sensitif

## Cakupan tahap pertama

Kolom berikut dienkripsi di server dengan AES-256-GCM sebelum ditulis ke PostgreSQL:

- NIK anak, orang tua, kader, dan bidan.
- Nomor telepon dan alamat orang tua.
- Nomor pegawai dan STR bidan.
- Catatan naratif kehamilan, gizi, pemeriksaan, imunisasi, KIPI, vitamin, dan konsultasi.

Nama, tanggal lahir, tanggal layanan, status, dan angka pengukuran belum dienkripsi pada tahap ini. Aplikasi masih menggunakannya untuk pencarian, laporan, pengingat, dan perhitungan. Kolom tersebut tetap membutuhkan pembatasan akses, backup yang terlindungi, dan enkripsi disk pada host PostgreSQL. Lampiran/media perlu ditinjau terpisah.

Pencarian NIK, STR, telepon, dan alamat dilakukan dengan HMAC blind index yang disimpan bersama ciphertext. Pencarian pada kolom-kolom itu menjadi pencarian nilai penuh, bukan potongan teks. Catatan klinis tidak lagi ikut dalam pencarian bebas konsultasi; pengguna yang berwenang tetap dapat membukanya dari detail rekamannya. Blind index tetap mengungkapkan apakah nilai yang sama muncul di kolom yang sama, tetapi tidak menyimpan nilainya secara langsung.

## Akses data terbaca

Drizzle mendekripsi kolom saat server membaca database. API tetap mengirim nilai terbaca setelah pemeriksaan otorisasi yang sudah berlaku, sehingga UI maker/admin tidak perlu diubah dan client tidak pernah menerima kunci enkripsi. Maker mengakses data melalui akun admin yang memang diberi otorisasi; kepemilikan proyek tidak menambahkan role atau bypass akses baru. `posyandu_admin` tetap mengikuti cakupan yang sudah berlaku, termasuk monitoring semua Posyandu.

Enkripsi ini melindungi salinan database dari pembaca yang hanya memperoleh akses ke PostgreSQL atau file dump. Server aplikasi dan pihak yang dapat mengambil secret dari Infisical tetap dapat mendekripsi data. Batasi akses Infisical dan server produksi sesuai kebutuhan, serta audit akses admin.

Target Podman Compose lokal mengambil secret langsung dari proses Infisical, tidak menulis salinan `.env`, dan hanya mengirim kredensial database ke container PostgreSQL. Pengelola Podman pada host tetap dapat membaca environment container aplikasi; batasi akses host tersebut.

Port PostgreSQL Compose hanya terikat ke loopback host. Container aplikasi tetap mengakses database melalui network internal Compose. Bila alur lama sudah pernah membuat `.env` berisi secret, periksa dan hapus file lama secara aman setelah dipastikan tidak dipakai lagi.

## Kunci Infisical

Sediakan secret server-side bernama `DATA_ENCRYPTION_KEY`. Isinya harus 32 byte acak yang di-encode sebagai 64 karakter hex atau base64. Contoh membuat key hex:

```bash
openssl rand -hex 32
```

Simpan hasilnya langsung di Infisical; jangan commit ke repository, kirim lewat chat, atau menaruhnya di client. Gunakan key berbeda untuk development, staging, dan production. Simpan backup/escrow key dengan kontrol akses terpisah. Kehilangan key membuat ciphertext tidak dapat dibaca. Jangan mengganti key di Infisical tanpa proses re-enkripsi; format ini belum menyediakan rotasi otomatis.

## Penerapan pada database yang sudah berisi data

Lakukan pada database aktif yang saat ini dipakai aplikasi. Migrasi PostgreSQL ini juga tetap berlaku ketika database nanti dipindah ke PostgreSQL lokal Podman.

1. Pastikan `DATA_ENCRYPTION_KEY` sudah tersedia untuk proses server dan CLI melalui Infisical.
2. Buat dan verifikasi backup database serta simpan backup key secara terpisah.
3. Hentikan instance aplikasi versi lama agar tidak ada penulisan plaintext baru.
4. Jalankan migrasi skema dengan `make db-mig`, lalu deploy versi server ini.
5. Dengan server versi baru berjalan dan key yang sama, lihat jumlah data lama yang belum terenkripsi:

   ```bash
   make data-encrypt-preview
   ```

6. Setelah backup dipastikan tersedia, jalankan backfill:

   ```bash
   make data-encrypt-apply
   ```

7. Jalankan `make data-encrypt-preview` sekali lagi. Setiap kolom harus melaporkan `0 rows need encryption`. Simpan log jumlah saja; script tidak mencetak nilai data.

Backfill bisa dijalankan ulang. Script mengenkripsi dalam batch dan hanya memperbarui baris jika nilainya tidak berubah sejak dibaca. Backup lama, ekspor, atau dump yang dibuat sebelum backfill masih dapat berisi plaintext; atur retensi dan penghapusannya secara terpisah.

Jangan jalankan backfill terhadap production sebelum key Infisical, backup, dan urutan rollout sudah dikonfirmasi. Jangan mengirim nilai key atau data hasil dump untuk proses validasi.
