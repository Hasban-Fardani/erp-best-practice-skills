# Struktur Backend dan Quality Gate

**Pemilik kanonik:** file ini mengatur ukuran file backend Laravel, urutan
method, inheritance, dan bukti quality gate. Aturan domain tetap dimiliki oleh
reference yang lebih spesifik seperti `concurrency.md`, `forms.md`, atau
`money-and-data-integrity.md`.

## Tujuan dan alasan

Kode ERP dirawat selama bertahun-tahun oleh senior, junior, dan agent AI.
Aturan ini membatasi biaya pencarian dan mengurangi ruang bagi AI untuk
mengklaim refactor sudah aman padahal belum dibuktikan.

## Batas backend

- Setiap file PHP di `app/` maksimal **50 baris fisik**.
- Jika lebih, pecah berdasarkan satu tanggung jawab yang dapat diberi nama:
  query, validasi, rendering output, adapter framework, atau service boundary.
- Jangan memecah button sederhana ke banyak lapisan hanya demi mengejar angka.
  File pemanggil tetap boleh menjadi façade tipis agar alur utama mudah dibaca.
- `tools/quality/` adalah infrastructure untuk memeriksa aplikasi, bukan
  runtime backend. Ia boleh memiliki struktur berbeda, tetapi tidak boleh
  menjadi tempat menyembunyikan pelanggaran di `app/`.

## Urutan method

Dalam setiap class aplikasi, urutkan method dari API paling abstrak ke detail:

1. `public` — kontrak yang dipanggil route, command, job, atau class lain;
2. `protected` — extension point yang memang dibutuhkan subclass/framework;
3. `private` — detail implementasi terakhir.

Guard memeriksa urutan visibilitas ini secara struktural. Method framework
seperti `handle`, `rules`, atau `render` tetap ditempatkan pada kelompok
visibilitasnya; jangan mengubah visibilitas hanya untuk memuaskan urutan.

## Batas inheritance

- Maksimal dua tingkat parent yang didefinisikan aplikasi.
- Parent dari framework/library tidak dihitung sebagai tingkat aplikasi,
  tetapi tetap harus dipakai hanya jika boundary framework membutuhkannya.
- Utamakan komposisi: service kecil, collaborator, policy, query object, atau
  renderer yang memiliki satu alasan perubahan.
- Interface dan dependency injection lebih disukai daripada base class besar.
  Jangan membuat abstraction hanya untuk memindahkan satu baris yang masih
  jelas di tempat pemanggil.

## PHP Insights

PHP Insights adalah sinyal terukur, bukan pengganti review domain. Pada skala
yang dipakai package, skor 1–49 berarti merah, 50–79 kuning, dan 80–100 hijau.
Kontrak proyek:

- `quality`, `complexity`, `architecture`, dan `style` minimal **50**;
- kompleksitas hijau (`>=80`) adalah target, bukan alasan untuk membuat
  abstraction berlebihan;
- tidak boleh menyembunyikan pelanggaran aplikasi dengan mengecualikan
  `app/` dari scan;
- `tools/quality/` boleh dikecualikan dari PHP Insights karena ia diperiksa
  oleh guard tersendiri.

Perintah bukti:

```text
php tools/quality/backend-guard.php
composer test --no-interaction --no-progress
composer quality --no-interaction --no-progress
php artisan insights --no-interaction
```

`composer test` gagal bila guard gagal. `composer quality` gagal bila salah
  satu requirement PHP Insights di bawah 50. Warnings yang masih berada di
  zona kuning dicatat sebagai backlog dengan file, skor, dan alasan; jangan
  menyebutnya hijau.

## Akuntabilitas agent

Setiap agent yang mengubah backend harus melaporkan:

- file yang melewati 50 baris sebelum dan sesudah perubahan;
- hasil guard dan PHP Insights yang benar-benar dijalankan;
- hasil Pest untuk behavior yang berubah;
- bagian yang belum diperiksa sebagai `NOT RUN` atau `UNKNOWN`;
- kesalahan/improvisasi yang terjadi dan pencegahannya pada pekerjaan berikut.

Screenshot hanya membuktikan tampilan yang terlihat. Ia tidak membuktikan
query, authorization, concurrency, test, atau skor quality.
