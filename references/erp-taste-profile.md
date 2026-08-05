# Profil Taste yang Diadaptasi untuk ERP

**Status:** profil internal untuk boilerplate dan registry; bukan salinan
aturan Taste Skill dan bukan pengganti review domain.

## Mengapa adaptasi diperlukan

Taste Skill v2 sendiri menyatakan fokusnya pada landing page, portfolio, dan
redesign, serta tidak menargetkan dashboard, data table, admin panel, atau
multi-step product UI. ERP kita justru hidup di area tersebut. Karena itu kita
mengambil disiplin brief, dials, konsistensi sistem, state lengkap, dan
anti-AI-slop; heuristik marketing yang bertentangan dengan throughput ERP
ditolak.

Sumber yang dibaca pada 2026-08-05:

- [Panduan Taste Skill](https://www.tasteskill.dev/guide)
- [Source SKILL.md Taste Skill](https://raw.githubusercontent.com/Leonxlnx/taste-skill/main/skills/taste-skill/SKILL.md)
- [Changelog Taste Skill](https://raw.githubusercontent.com/Leonxlnx/taste-skill/main/CHANGELOG.md)

## Brief sebelum membuat layar

Tulis satu kalimat **Design Read** dalam bahasa Indonesia sebelum menyentuh
komponen: siapa operatornya, pekerjaan yang ingin diselesaikan, data apa yang
paling penting, risiko salah baca, dan mengapa mode/tata letak yang dipilih
membantu pekerjaan itu. Jika requirement ambigu, tandai `UNKNOWN` dan tanyakan
satu hal yang benar-benar mengubah keputusan.

Lalu isi tiga dials berikut pada receipt atau dokumentasi layar:

| Dial | Default ERP | Arti praktis |
|---|---:|---|
| `DESIGN_VARIANCE` | 2–4 | Konsistensi tinggi; variasi hanya untuk prioritas kerja, bukan dekorasi. |
| `MOTION_INTENSITY` | 1–2 | Transisi ringan untuk state; tidak ada scroll hijack atau animasi yang menghambat input. |
| `VISUAL_DENSITY` | 5–8 | Form memakai 5–6; tabel operasional memakai 7–8; registry docs memakai 4–6. |

Nilai di atas adalah baseline yang dapat diubah berdasarkan bukti usability.
`compact` menaikkan density dan mengurangi prose/ruang, sedangkan `complete`
menambah bantuan dan pemisah visual. Keduanya tetap memuat label, tag, status,
aksi utama, state loading/error, dan jalur keyboard.

## Yang dipakai

- Satu design system yang jelas per project. Di dua project ini sumber warna,
  radius, typography, focus, border, dan komponen tetap Bootstrap Sass + token
  ERP; jangan membuat utilitas CSS kedua yang bersaing.
- Satu keluarga icon melalui komponen bersama. Tidak ada SVG ad-hoc. Lucide
  yang sudah menjadi dependency project dipertahankan sampai ada bukti bahwa
  perpindahan icon memberi manfaat nyata.
- State lengkap: loading mengikuti bentuk akhir, empty state menjelaskan
  langkah berikutnya, error menunjukkan pemulihan, dan feedback request masuk
  toast/SweetAlert sesuai aturan project.
- Label berada di atas input, contrast input cukup jelas terhadap canvas,
  focus terlihat, target sentuh tidak dipadatkan hanya demi estetika.
- Satu skala radius dan border yang konsisten; shadow dekoratif tetap dilarang
  karena ERP memprioritaskan keterbacaan jangka panjang.
- Data contoh harus realistis dan berbahasa Indonesia. Jangan memakai data
  filler yang membuat operator salah memahami panjang atau status sebenarnya.
- Komponen/block harus memiliki source ownership, boundary penggunaan,
  `not_for`, state owner, dan contoh yang dapat dijalankan.

## Yang ditolak atau diubah maknanya

| Heuristik Taste | Perlakuan ERP | Alasan |
|---|---|---|
| Hero besar, CTA above-the-fold, delapan section | Ditolak untuk layar kerja | Operator datang untuk menyelesaikan tugas, bukan membaca marketing page. |
| Bento/card grid dekoratif | Hanya dipakai jika grouping-nya punya keputusan operasional | Card identik menambah beban pencarian. |
| Hindari data table/list | Ditolak untuk data operasional | Table adalah alat utama scanning, audit, dan sorting ERP. |
| Motion showcase, scroll hijack, 3D | Ditolak | Mengganggu keyboard, usia lanjut, perangkat rendah, dan pemulihan. |
| Gradient, glow, neon, pure black, dekorasi dots | Ditolak kecuali warna/status memiliki makna bisnis | Mengurangi signal-to-noise dan berisiko terlihat seperti state penting. |
| Pilihan icon library baru | Tidak otomatis diikuti | Konsistensi dependency dan bundle lebih penting daripada tren. |
| Copy berbahasa Inggris atau slogan filler | Diubah menjadi copy Indonesia yang spesifik | Bahasa operator adalah bagian dari safety dan onboarding. |

## Profil layar ERP

### Shell, tabel, dan form

- `DESIGN_VARIANCE=2–3`, `MOTION_INTENSITY=1`, `VISUAL_DENSITY=6–8`.
- Prioritaskan alignment kolom, status dengan ukuran token yang sama, border
  yang terlihat, dan background control yang berbeda dari canvas.
- Aksi utama baris selalu terlihat. Dropdown di baris terakhir harus membuka
  ruang yang aman dan tidak tertutup footer; bila perlu gunakan placement ke
  atas atau menu yang keluar dari clipping container tanpa menutup aksi utama.
- Loading filter mempertahankan data lama sebagai konteks, memberi `aria-busy`,
  indikator proses, dan ringkasan filter aktif. Hindari refresh penuh.

### Registry documentation dan preview

- `DESIGN_VARIANCE=3–5`, `MOTION_INTENSITY=1–2`, `VISUAL_DENSITY=4–6`.
- Halaman item adalah satu decision page: preview hidup, contoh penggunaan,
  props/configuration, dependency, state, test contract, provenance, dan
  batas `not_for` berada dalam alur yang sama.
- Search global tetap satu. Hasil lexical dan meaning harus diberi label sesuai
  mesin yang benar-benar dipakai; fallback lexical tidak boleh disebut semantic.

## Preflight wajib

Sebelum handoff, cek dua kali dengan dua sudut berbeda:

1. **Kontrak:** Design Read, dials, owner source, mode, label Indonesia,
   dependency, state loading/empty/error, accessibility, dan boundary prod ada.
2. **Persepsi:** screenshot pada ukuran target, alignment status/button/input,
   scan tabel, keyboard/focus, wrapping teks panjang, responsive, dan bundle
   capability yang tidak diperlukan tidak ikut dimuat.

Jika ada kegagalan atau improvisasi, catat file, keputusan, bukti yang belum
ada, dan pencegahan. Jangan mengubah “terlihat baik” menjadi klaim “aman untuk
production”.
