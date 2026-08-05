# Kontrak Filter, Loading, Feedback, dan Kepadatan UI

**Pemilik kanonik:** file ini untuk perilaku filter/list, loading request, pilihan
input, feedback, dan mode tampilan pada ERP Laravel Blade/Bootstrap. Aturan
visual tabel tetap dirujuk dari `table-design.md`; prinsip umum loading tetap
dirujuk dari `erp-principles.md` dan batas durasi dari `performance.md`.

## Tujuan dan alasan

Filter ERP bukan sekadar kontrol visual. Filter mengubah query server, posisi
operator, dan keputusan yang ia lihat. Karena itu kontraknya harus:

- mudah dipakai tanpa tombol `Terapkan` berulang;
- tidak menghapus filter lain ketika satu field berubah;
- terlihat sedang memuat dan menjelaskan hasil yang sedang dipakai;
- tetap dapat di-bookmark, di-refresh, dan dibuka kembali melalui URL;
- tetap aman pada data besar: server authoritative, bukan mengunduh semua data;
- tetap berfungsi sebagai GET biasa ketika JavaScript tidak tersedia.

Rekomendasi ini adalah **HIGH** untuk daftar kerja harian. Live update tidak
berarti query boleh pindah ke browser atau mutation boleh terjadi tanpa server.

## Kontrak live filter

Untuk layar daftar yang memilih live update:

1. Input text memakai debounce 300–400 ms. Jangan request setiap keypress.
2. Select atau filter lain memakai event `change` dan boleh memakai debounce yang
   sama supaya beberapa perubahan cepat digabung menjadi satu request.
3. URL adalah sumber persistence: tulis field bernilai ke query string, hapus
   field yang kosong, dan kembalikan `page` ke 1 setelah filter berubah.
4. Saat membentuk URL, pertahankan parameter lain yang tidak dimiliki form.
   Inilah yang mencegah `status` hilang ketika `segment` berubah.
5. Dengan JavaScript, ambil HTML/JSON dari server lalu ganti region hasil saja
   (`#main-content` atau region tabel), kemudian `history.pushState`. Jangan
   melakukan refresh dokumen penuh untuk perubahan filter biasa.
6. Batalkan request sebelumnya ketika request baru dimulai. Response lama tidak
   boleh menggantikan hasil filter yang lebih baru.
7. Tanpa JavaScript, form tetap menjadi GET biasa dengan tombol submit yang
   aksesibel; tombol dapat visually hidden bila interaksi otomatis tersedia.
8. `Clear filters` adalah aksi eksplisit yang menghapus field filter dan page,
   tetapi tetap mempertahankan parameter konteks yang memang bukan filter.

### Status hasil wajib

Setiap hasil harus menjawab secara langsung:

- filter apa yang aktif (contoh: `Filter aktif: Segmen: Retail · Status: Aktif`);
- hasil berasal dari server/indeks mana;
- jumlah/rentang data yang sedang terlihat;
- apakah request sedang berjalan (`Memuat…`) atau sudah selesai.

Jangan menampilkan toast sukses untuk setiap ketikan. Itu membuat noise dan
mengganggu operator. Ringkasan inline adalah feedback sukses untuk filter;
toast dipakai untuk error request, aksi user, atau notifikasi yang perlu masuk
ke perhatian operator.

## Loading state yang dapat dipercaya

Loading harus dipasang sebelum request dimulai:

- form/filter region memiliki `aria-busy="true"`;
- field yang tidak boleh berubah selama request dinonaktifkan;
- tabel lama tetap terlihat dengan overlay tenang, bukan hilang mendadak;
- overlay menyebut `Memuat data…` dan konteks `Berdasarkan filter yang baru
  dipilih`;
- result summary berpindah ke state loading;
- setelah berhasil, overlay hilang dan ringkasan filter baru terlihat;
- setelah gagal, data lama tetap ada, operator mendapat alasan dan langkah retry,
  serta error request dikirim melalui toast/status yang sesuai.

Jangan memakai spinner tanpa label pada layar yang dipakai operator 40+. Jangan
menyembunyikan seluruh tabel sehingga operator mengira datanya hilang. Untuk
request yang lama, pertahankan data sebelumnya sebagai konteks dan tandai
bahwa itu data sebelum filter baru, atau gunakan skeleton jika struktur lama
menyesatkan.

## Pilihan input

| Kondisi | Default | Alasan |
|---|---|---|
| 2–3 opsi, tetap, semua mudah diingat | native `<select>` | paling stabil, keyboard dan Livewire rendah friction |
| Lebih dari 3 opsi tetap | searchable select | mengurangi scan panjang dan salah pilih |
| Opsi dinamis/remote | searchable select + endpoint server | tidak mengunduh seluruh master data |
| User boleh membuat nilai baru | combobox khusus dengan kontrak create | jangan mengaktifkan `create` secara default |

Untuk stack Blade + Bootstrap ini, Tom Select adalah capability opsional yang
di-lazy-load ketika marker searchable select ada. Ia tidak memerlukan jQuery,
mendukung throttle/load remote, dan memiliki lifecycle `destroy`. Pada project
yang sudah terikat kuat ke jQuery, Select2 boleh dipakai, tetapi catat biaya
jQuery, `wire:ignore`, re-inisialisasi, escaping, dan risiko DOM stale.

Aturan searchable select:

- `create: false` kecuali domain memang mengizinkan nilai baru;
- pilihan yang sudah dipilih harus tersedia pada HTML awal atau dimuat ulang;
- query remote harus server-side, diberi batas, authorization, dan test;
- gunakan `loadThrottle`/debounce; jangan membuat request setiap karakter tanpa
  batas minimum;
- dropdown tidak boleh terpotong modal, viewport, table footer, atau overflow;
- escape label/HTML dari server;
- untuk Livewire, bungkus integrasi third-party dengan `wire:ignore`, sinkronkan
  value secara eksplisit, dan destroy/re-init pada lifecycle/morph yang benar.

Dokumentasi yang menjadi evidence untuk keputusan ini:

- [Tom Select docs](https://tom-select.js.org/docs/)
- [Tom Select API](https://tom-select.js.org/docs/api/)
- [Select2 Ajax](https://select2.org/data-sources/ajax/)
- [Livewire wire:ignore](https://livewire.laravel.com/docs/3.x/wire-ignore)
- [Livewire wire:model](https://livewire.laravel.com/docs/3.x/wire-model)

## Alert versus toast

Ini adalah hard rule kantor:

| Situasi | Komponen | Contoh |
|---|---|---|
| informasi/peringatan yang harus tetap terlihat di halaman | Bootstrap alert/inline alert | batas interaksi, peringatan masa berlaku, instruksi pemulihan |
| feedback sementara dari request/aksi/notifikasi | toast atau SweetAlert2 | berhasil disimpan, request gagal, job selesai |
| konfirmasi destruktif | modal/SweetAlert2 | hapus, batalkan posting, irreversible action |

Jangan mengubah alert statis menjadi toast hanya karena ingin UI terlihat
modern. Jangan membuat toast untuk setiap perubahan filter live. Error request
filter boleh memakai toast **dan** ringkasan inline bila operator perlu tahu
bahwa data lama masih ditampilkan.

## Complete versus compact

Mode adalah cara mengurangi beban mental, bukan izin menghapus informasi.

Mode `complete` boleh menampilkan penjelasan, alasan desain, helper text, dan
konteks onboarding. Mode `compact` harus tetap mempertahankan:

- label field;
- filter aktif dan hasilnya;
- tag yang membantu klasifikasi;
- status text + icon;
- aksi utama dan keyboard path;
- error, loading, dan recovery instruction.

Perbedaan compact dibuat melalui spacing, jumlah prose panjang, grouping,
penekanan aksi berikutnya, dan ukuran metadata—bukan `display:none` pada tag,
status, alert penting, atau action. Control mode harus eksplisit, misalnya
segmented control `Lengkap | Ringkas`, bukan tombol yang hanya berkata
`Mode lain`.

## Keseragaman geometri

Operator melihat tabel sebagai pola vertikal. Label status yang panjangnya
berbeda tidak boleh membuat kolom melompat. Untuk himpunan status yang sudah
didefinisikan:

- tetapkan width/min-width yang sama;
- tetapkan min-height yang sama;
- pusatkan icon dan label secara konsisten;
- cegah wrapping bila semua label memang dapat muat pada lebar kontrak;
- jika label dinamis dapat lebih panjang, tetapkan aturan truncate/tooltip dan
  uji label terpanjang.

Aturan yang sama berlaku untuk action button, pagination, filter control, dan
cell yang memiliki nilai terkontrol. Ukuran harus berasal dari token, bukan
improvisasi per row.

## Acceptance test minimum

### Pest

- filter query menggabungkan seluruh field yang aktif;
- field invalid tidak mengubah query menjadi filter tak dikenal;
- response merender field terisi, summary filter, loading marker, dan clear;
- empty state menyebut konteks filter;
- query string bertahan saat membuka detail/back/refresh;
- `Terapkan` tidak muncul pada kontrak live filter;
- error server tidak menghapus data/summary lama secara diam-diam.

### Browser

- debounce menghasilkan satu request setelah jeda, bukan request per keypress;
- perubahan filter kedua mempertahankan filter pertama;
- filter memakai `fetch`/partial replacement dan tidak membuat document request
  penuh ketika capability SPA-like tersedia;
- overlay table dan `aria-busy` terlihat selama request yang diperlambat;
- searchable select dapat dipakai dengan keyboard dan tidak tertutup dropdown;
- complete/compact tetap mempertahankan tag/status/aksi;
- setelah browser refresh, query URL mengisi kembali semua field;
- lebar/tinggi status dan tombol yang seharusnya seragam benar-benar sama pada
  screenshot dan pengukuran DOM.

## Catatan kesalahan dan improvisasi slice 2026-08-05

Catatan ini sengaja dipelihara agar agent berikutnya tidak mengulang kesalahan:

1. CSS compact awal menyembunyikan helper, isi alert, dan tag registry. Itu
   melanggar tujuan compact; diperbaiki menjadi pengurangan prose/spacing tanpa
   menghapus informasi operasional.
2. Assertion browser awal mencari substring `Terapkan`, sehingga ikut membaca
   kata `diterapkan`. Assertion diganti menjadi exact text.
3. Harness Playwright awal memakai `route.url()`, padahal URL harus dibaca dari
   `route.request().url()`. Test diperbaiki dan kesalahan dicatat.
4. Implementasi awal memakai navigasi dokumen penuh. Karena user meminta minim
   refresh, kontrak diubah menjadi fetch + partial `#main-content` replacement,
   `pushState`, abort stale request, dan fallback GET tanpa JavaScript.
5. Browser test awal mengubah native select tersembunyi dengan `selectOption`;
   itu dapat membuat tampilan Tom Select tidak mencerminkan pilihan. Test akhir
   memilih option melalui `.ts-control`/dropdown seperti operator.
6. Link `Atur ulang` di registry sempat membungkus menjadi dua baris. Ditambah
   `white-space: nowrap` dan flex basis tetap agar alignment tidak rusak.
7. Status badge awal mengikuti panjang teks masing-masing. Kontrak geometri
   ditetapkan dan diukur ulang di browser: slice ini menghasilkan lebar 152px
   dan tinggi 43px untuk lima status fixture.

Jika perubahan berikutnya mengubah salah satu kontrak, tambahkan receipt dan
perbarui bagian ini sebelum menyatakan hasilnya stabil.
