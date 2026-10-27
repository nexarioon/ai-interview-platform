# Naskah Video — AI Interview Platform (v2)
# Durasi target: 4–5 menit | Bahasa: Indonesia, nada teknis santai
# Dibuat ulang sesuai brief asli: 3–5 menit, demo UI dulu baru kode, UU PDP disebut, semua 7 screenshot dipakai

---

## PERSIAPAN SEBELUM REKAM

Buka tab-tab ini di browser dan editor **sebelum** kamera nyala, urutannya persis seperti di bawah:

| # | Yang Disiapkan | Detail |
|---|---|---|
| 1 | **Browser tab 1** — Portfolio page (sudah ada data) | Halaman yang menampilkan skill cards dengan badge "—" abu-abu untuk skill `not_assessed` |
| 2 | **Browser tab 2** — Fit/Gap Report page | Halaman tabel perbandingan skill dengan banner amber dan kolom Result |
| 3 | **Browser tab 3** — Interview link error | Buka link interview dengan token invalid/expired |
| 4 | **Browser tab 4** — Interview link valid | Buka link interview yang valid, sudah sampai di halaman hardware check |
| 5 | **Browser tab 5** — Fit/Gap mobile | Buka halaman fit/gap, resize browser ke lebar 375px atau pakai Chrome DevTools device mode |
| 6 | **Editor tab 1** — `api/app/services/portfolios/skill_status.rb` | Scroll ke baris 23–39 (method `derive`) |
| 7 | **Editor tab 2** — `api/app/services/portfolios/skill_status.rb` | Scroll ke baris 67–77 (method `parse_level`) |
| 8 | **Editor tab 3** — `assessment/step-5-execution-evidence.md` | Bagian "Seeded fault test" — diff 2 baris itu |
| 9 | **Editor tab 4** — `api/config/application.yml.sample` | Baris 20–23 (nama model Gemini) |
| 10 | **Screenshot** — `07-fitgap-mobile-375px.png` | Siap dibuka kalau browser resize gagal |

> **Tip:** Nonaktifkan notifikasi OS. Sembunyikan bookmarks bar. Gunakan profil browser clean jika ada data sensitif.

---

## SCRIPT

---

### [0:00–0:30] — Masalah & Dampaknya

**🖥️ TAMPILKAN:** Screenshot `01-portfolio-not-assessed-skill.png` — portfolio page dengan skill cards

**📌 KETERANGAN LAYAR:**
Tunjukkan kartu skill yang menampilkan badge `—` abu-abu (LevelBadge dengan level null), teks "Not covered in the interview" di bawah nama skill, dan banner abu-abu bertulisan *"This skill was never reached in the interview — not a weak result, just an untested one."*

**🗣️ UCAPKAN:**
> "Ini halaman portfolio skill kandidat — yang dibaca recruiter dan hiring manager untuk memutuskan lanjut atau tidak. Yang ingin saya tunjukkan dulu adalah ini: lihat skill yang punya badge abu-abu dengan tanda dash. Itu artinya skill ini memang tidak sempat dibahas saat interview — waktu habis sebelum sampai sana.
>
> Di sistem lama, kondisi ini tidak ada. Skill yang tidak sempat ditanya tetap dipaksa dapat angka level dari AI — lalu masuk laporan fit/gap sebagai 'Gap minus dua'. Recruiter baca laporan itu, tidak tahu bedanya antara kandidat yang memang lemah, dengan kandidat yang simply belum pernah ditanya. Satu orang bisa ditolak bukan karena kemampuannya, tapi karena sistem tidak sempat menjangkau skill-nya."

---

### [0:30–1:00] — Hasil Setelah Fix: Fit/Gap Report

**🖥️ TAMPILKAN:** Switch ke Browser tab 2 — Fit/Gap Report page

**📌 KETERANGAN LAYAR:**
Tunjukkan: (1) banner amber di atas tabel bertulisan *"X of N skills could not be fully assessed in this interview. That reflects interview coverage, not candidate ability"*, (2) baris di tabel yang kolom Result-nya menampilkan badge "— Not assessed" berwarna netral abu-abu, bukan "⚠ Gap", (3) kolom Confidence dan Evidence yang menunjukkan data transparannya.

**🗣️ UCAPKAN:**
> "Di laporan fit/gap-nya sekarang beda. Ada banner di atas yang langsung bilang: sekian skill tidak bisa dinilai penuh — itu soal cakupan interview, bukan kemampuan kandidat. Dan di tabel, skill yang tidak sempat dibahas masuk kolom Result sebagai 'Not assessed', bukan gap. Delta-nya null. Tidak dihitung ke dalam skor apapun.
>
> Ini bukan soal estetika — ini soal apakah keputusan rekrutmen yang diambil berdasarkan laporan ini bisa dipertanggungjawabkan. Di Indonesia, UU PDP mengharuskan keputusan otomatis punya dasar yang bisa dijelaskan. Kalau sistemnya salah baca 'belum ditanya' sebagai 'gap', dasar itu tidak ada."

---

### [1:00–1:30] — Bug P0: Invite Link & Error State

**🖥️ TAMPILKAN:** Switch ke Browser tab 3 — halaman error state interview

**📌 KETERANGAN LAYAR:**
Tunjukkan halaman dengan ikon ⚠️, judul *"This interview link isn't working"*, dan teks *"It may have expired, already been used, or there's a temporary connection problem. Please contact the person who sent you this link — your interview has not started or been recorded."*

**🗣️ UCAPKAN:**
> "Dua bug P0 yang juga saya perbaiki. Yang pertama: tautan undangan interview. Di kode lama, `Session#invite_url` menggunakan base URL milik API, padahal halaman interview ada di frontend — origin yang beda di deployment nyata. Jadi setiap link yang dikirim ke kandidat pasti 404.
>
> Yang kedua, dan ini yang lebih berbahaya secara UX: kalau link gagal — token expired, jaringan putus, backend down — aplikasi lama menampilkan layar 'Interview Complete, terima kasih sudah berpartisipasi.' Kandidat mengira sudah selesai wawancara, padahal bahkan belum mulai. Sekarang ada state tersendiri dengan pesan yang jujur."

**🖥️ TAMPILKAN:** Switch ke Browser tab 4 — hardware check (link valid)

**📌 KETERANGAN LAYAR:**
Tunjukkan halaman hardware check yang berjalan normal — kandidat sudah masuk flow interview yang benar.

**🗣️ UCAPKAN:**
> "Dan ini kalau link-nya valid — kandidat langsung masuk ke tahap hardware check sebelum interview dimulai."

---

### [1:30–2:30] — Root Cause & Implementasi Teknis

**🖥️ TAMPILKAN:** Switch ke Editor tab 1 — `skill_status.rb` baris 23–39

**📌 KETERANGAN LAYAR:**
Highlight method `self.derive` — khususnya baris:
```ruby
return Result.new(status: NOT_ASSESSED, level: nil, confidence: "low") if not_yet?(coverage)
```

**🗣️ UCAPKAN:**
> "Root cause-nya dua hal yang saling nyambung. Pertama, sistem lama mewajibkan setiap skill dapat angka level satu sampai lima — jadi kalau interview belum sampai sana, AI tetap dipaksa nebak. Kedua, parser level-nya diam-diam salah: kalau AI jawab teks 'L3', `'L3'.to_i` hasilnya nol, lalu `clamp(1,5)` ubah jadi satu. Jadi string yang gagal di-parse berubah jadi level satu yang terlihat valid — padahal itu kegagalan parsing, bukan hasil nyata.
>
> Perbaikannya saya taruh di satu file baru: `Portfolios::SkillStatus`, method `derive`. Ini satu-satunya tempat yang memutuskan status sebuah skill — tidak ada logika tercecer di banyak file. Baris paling penting ini: begitu ketauan coverage-nya `not_yet`, langsung return `not_assessed`, titik. Tidak peduli AI kasih angka apa — skill itu tidak akan pernah bisa lolos jadi 'assessed' kalau interview memang belum nyampe sana."

**🖥️ TAMPILKAN:** Switch ke Editor tab 2 — `skill_status.rb` baris 67–77

**📌 KETERANGAN LAYAR:**
Highlight method `self.parse_level`, khususnya bagian `when String` yang return `nil` kalau gagal parse.

**🗣️ UCAPKAN:**
> "Parser barunya sengaja return `nil` kalau gagal parse — bukan maksa jadi angka. Kalau level-nya nil, confidence low, atau evidence kosong, status jadi `insufficient_evidence` — masih bisa ditampilkan sebagai konteks, tapi tidak dihitung sebagai gap. Status ini dikunci saat generate, bukan dihitung ulang tiap kali dibaca — jadi laporan lama tidak bisa berubah diam-diam kalau aturan diubah besok."

---

### [2:30–3:10] — Bukti Test: Seeded Fault

**🖥️ TAMPILKAN:** Switch ke Editor tab 3 — `step-5-execution-evidence.md` bagian seeded fault test

**📌 KETERANGAN LAYAR:**
Tunjukkan diff dua baris:
```diff
- return Result.new(status: NOT_ASSESSED, ...) if not_yet?(coverage)
+ return Result.new(status: NOT_ASSESSED, ...) if not_yet?(coverage) && level.nil?
```
Dan output test yang gagal: `3 examples failed`.

**🗣️ UCAPKAN:**
> "Supaya bukan cuma klaim 'ada test', saya buktikan test-nya beneran bisa mendeteksi regresi. Saya sengaja rusak satu baris di `skill_status.rb` — persis baris yang baru saya tunjukkan tadi. Perubahannya kecil: tambah kondisi `&& level.nil?` di akhir. Efeknya: skill yang tidak pernah dibahas bisa lolos jadi assessed lagi kalau AI kebetulan kasih angka valid — persis bug yang jadi alasan fix ini ada.
>
> Hasilnya: 3 dari 42 test gagal, persis di titik yang seharusnya gagal — dua di unit test, satu di level integration. Saya revert, suite balik hijau 42 dari 42. Total 49 test: 42 RSpec dan 7 Vitest — semuanya dibangun dari nol karena repo ini sebelumnya tidak punya test runner sama sekali, baik backend maupun frontend."

---

### [3:10–3:45] — AI Verification Moment

**🖥️ TAMPILKAN:** Switch ke Editor tab 4 — `application.yml.sample` baris 20–23

**📌 KETERANGAN LAYAR:**
Tunjukkan baris `GEMINI_FLASH_MODEL: "gemini-3.8-flash"` dan komentar di atasnya yang menjelaskan model lama sudah retire.

**🗣️ UCAPKAN:**
> "Dua hal yang cuma bisa ketahuan dengan benar-benar manggil API-nya, bukan baca dokumentasi. Pertama, model Gemini yang dikonfigurasi di repo — `gemini-2.0-flash-001` — sudah di-retire Google. Panggilan langsung balasannya: 'This model is no longer available.' Saya ganti ke `gemini-3.8-flash` setelah verifikasi respons 200 sungguhan.
>
> Kedua, timeout yang sudah ada di kode — 180 detik — ternyata tidak cukup buat model reasoning yang baru. Saya ukur langsung: gagal di 180 detik, gagal lagi di 120, baru berhasil di 300 detik. Nilai 240 detik yang saya pakai sekarang itu dari hasil pengukuran, bukan asumsi. Ini juga contoh konkret kenapa AI-generated code tidak bisa langsung dipercaya tanpa verifikasi — nilai yang dibuat AI kelihatan masuk akal, tapi salah."

---

### [3:45–4:20] — Responsive & UI States

**🖥️ TAMPILKAN:** Switch ke Browser tab 5 — fit/gap mobile 375px (atau buka screenshot `07-fitgap-mobile-375px.png`)

**📌 KETERANGAN LAYAR:**
Tunjukkan tabel fit/gap di lebar 375px — tabel bisa di-scroll horizontal, summary di bawah wrap ke baris baru, badge result tetap terbaca. Kalau pakai DevTools, tunjukkan lebar 375px di inspector.

**🗣️ UCAPKAN:**
> "Ini tampilan laporan fit/gap di lebar 375px — viewport mobile. Tabel bisa di-scroll horizontal, teks tidak terpotong, badge result tetap terbaca. Brief-nya eksplisit minta semua interaction state ditangani: loading, empty, error, partial, dan responsive. Semua ada — error state di portfolio page, export error, load error, dan ini responsive-nya."

**🖥️ TAMPILKAN:** Kembali ke Browser tab 1 — portfolio page

**📌 KETERANGAN LAYAR:**
Scroll ke bawah, tunjukkan skill dengan status `insufficient_evidence` — badge amber, teks "Only briefly explored."

**🗣️ UCAPKAN:**
> "Dan satu state lagi yang penting: `insufficient_evidence` — skill yang sempat dibahas tapi buktinya kurang kuat. Ditampilkan berbeda dari `not_assessed`: ada level AI-nya sebagai konteks, ada banner amber, tapi juga tidak dihitung sebagai gap di laporan. Tiga status ini — assessed, insufficient evidence, not assessed — masing-masing punya representasi visual yang berbeda supaya recruiter dan assessor bisa baca situasinya dengan benar."

---

### [4:20–4:50] — Trade-off & Yang Tidak Dikerjakan

**🖥️ TAMPILKAN:** Tetap di portfolio page atau buka laporan di PDF viewer

**📌 KETERANGAN LAYAR:**
Tidak perlu perpindahan layar besar — ini segmen narasi.

**🗣️ UCAPKAN:**
> "Saya temukan total 15 bug, tapi sengaja hanya mendalami satu sampai tuntas. Kenapa? Karena ada dua temuan P0 yang butuh keputusan desain lintas tim — IDOR lintas tenant dan bug pemilihan tenant bebas saat login — dan saya tidak mau menyisipkan perbaikan yang butuh keputusan arsitektur besar ke dalam PR yang sudah bawa satu migration data. Lebih baik didokumentasikan jujur dengan alasannya daripada dikerjakan terburu-buru dan merusak data kandidat yang sudah ada.
>
> Untuk bug utama ini, saya pertimbangkan tiga opsi: status dihitung ulang tiap baca, status disimpan permanen di database, atau sekadar perubahan label UI. Saya pilih yang kedua — status dikunci saat generate — karena laporan fit/gap ini catatan dari keputusan rekrutmen terhadap orang sungguhan. Kalau status dihitung ulang tiap baca, hasil penilaian kandidat bisa berubah diam-diam kalau aturan diubah besok. Dengan status tersimpan, ada satu tempat di kode yang jadi pemilik tunggal keputusan itu — dan itu relevan langsung dengan UU PDP: keputusan otomatis harus punya dasar yang bisa dijelaskan dan bisa diaudit, bukan yang bisa berubah sewaktu-waktu."

---

## CATATAN PRODUKSI

- **Ganti `[Nama]`** kalau ada perkenalan — tapi di script ini sengaja tidak ada perkenalan sesuai brief.
- **Durasi estimasi realistis:**
  - Segmen 1–2 (demo UI): ~1 menit — jangan terlalu cepat scroll, beri waktu viewer baca teks di layar
  - Segmen 3 (P0 bugs): ~30 detik
  - Segmen 4–5 (kode + test): ~1 menit — baris kode harus kelihatan jelas, zoom kalau perlu
  - Segmen 6 (AI verification): ~35 detik
  - Segmen 7 (responsive + states): ~35 detik
  - Segmen 8 (trade-off): ~30 detik
  - **Total: ~4:20** — masih dalam batas 5 menit
- **Bagian yang TIDAK boleh dipotong:** Segmen 1–2 (demo UI, ini yang dinilai Product Team), Segmen 4 (seeded fault test), Segmen 6 (AI verification), Segmen 8 (trade-off + yang tidak dikerjakan).
- **Kalau waktu mepet:** Potong Segmen 7 (responsive) — cukup sebut "responsive view ada di screenshot 07 di laporan" tanpa buka layar.
- **Font/zoom editor:** Pastikan font size editor cukup besar supaya kode terbaca di video — setidaknya 16px.
- **Semua nomor baris sudah diverifikasi** ke file repo saat script ini dibuat. Cek ulang dengan `grep -n "def derive\|def parse_level\|SkillStatus.derive"` sebelum rekam kalau ada perubahan kode.
