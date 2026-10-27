# Naskah Video — Laporan Audit AI Interview Platform

Target durasi: 5–7 menit (lebih panjang dari draft sebelumnya karena sekarang mencakup semua temuan + lokasi
kode persisnya, bukan cuma satu bug). Bahasa: Indonesia, nada ngobrol teknis ke sesama engineer, bukan baca
naskah formal. ID temuan (F-01, F-02, dst.) mengikuti penomoran di `case-study-report.docx` / Google Doc.

Cara pakai: buka VS Code (atau editor manapun) di repo ini sebagai layar utama, siapkan tab-tab file yang
disebut di bawah sebelum rekam supaya tidak bolak-balik cari file saat live. Screenshot ada di
`assessment/screenshots/`.

---

## 0:00–0:25 — Pembuka

**Layar:** Halaman pertama laporan (DOCX/Google Doc) — cover dengan judul "AI Interview Platform".

**Ucapkan:**
> "Halo, ini demo case study Fullstack Product Engineer dari Rakamin. Saya [Nama]. Repo ini, AI Interview
> Platform, menjalankan wawancara otomatis pakai AI, lalu AI itu yang menyusun portfolio skill dan laporan
> fit/gap kandidat — yang dipakai recruiter buat memutuskan lanjut atau tidak.
>
> Saya audit seluruh alurnya, benar-benar jalankan aplikasinya, bukan cuma baca kode. Ketemu beberapa bug
> dengan tingkat keparahan beda-beda — saya kelompokkan P0 sampai P3 di laporan. Video ini saya jalanin
> urut: temuan-temuan itu, baris kode persisnya, lalu satu bug yang saya dalami sampai tuntas."

---

## 0:25–1:10 — Ringkasan temuan P0 (3 bug, 2 sudah fix)

**Layar:** Tabel ringkasan temuan di laporan (Bagian 3), lalu buka `api/app/models/session.rb` baris 29–33.

**Ucapkan:**
> "Mulai dari yang paling parah, P0 — menghalangi produk jalan sama sekali.
>
> F-01: tautan undangan interview. Lihat di sini, `session.rb` baris 32 sampai 33 —"

**Tunjukkan kode** (`api/app/models/session.rb:32-33`):
```ruby
def invite_url
  base = ENV.fetch('WEB_BASE_URL', 'http://localhost:5173')
```

> "— sebelumnya method ini ambil base URL dari API-nya sendiri, padahal halaman interview itu ada di
> frontend, origin yang beda total di deployment nyata. Jadi setiap tautan yang dikirim ke kandidat pasti
> 404. Saya tambahkan env var `WEB_BASE_URL` terpisah — sudah saya cek juga di
> `api/config/application.yml.sample` dan `api/k8s/configmap.yaml`, dua-duanya saya update.
>
> F-02, bug terpisah di frontend — `InterviewPage.tsx` baris 56 —"

**Tunjukkan kode** (`web/src/pages/interview/InterviewPage.tsx:56`):
```ts
.catch(() => setInterviewState("link_error"));
```

> "— sebelumnya setiap kegagalan ambil data kandidat itu diperlakukan sama persis kayak interview yang
> sukses selesai. Sekarang ada state `link_error` sendiri. Dua-duanya sudah saya screenshot — nanti saya
> tunjukkan.
>
> F-03 dan F-04 — IDOR lintas tenant dan bug pemilihan tenant bebas saat login — saya temukan juga, tapi
> SENGAJA tidak saya kerjakan di submission ini. Nanti saya jelaskan kenapa di bagian akhir."

---

## 1:10–1:40 — Bukti visual F-01 & F-02

**Layar:** Screenshot **03-interview-link-error-state.png**, lalu **04-interview-link-working-hardware-check.png**.

**Ucapkan:**
> "Ini hasilnya. Tautan yang tidak valid sekarang nampilin error state yang jelas, bukan layar kosong atau
> macet. Dan ini tautan yang valid — kandidat langsung masuk ke tahap hardware check sebelum interview
> dimulai."

---

## 1:40–3:30 — F-05: bug utama yang saya dalami

**Layar:** Tabel ringkasan lagi, sorot baris F-05, lalu screenshot **02-fitgap-not-assessed-not-gap.png**.

**Ucapkan:**
> "Jadi untuk temuan yang saya kerjakan, yang pertama itu soal skill yang sebenarnya tidak sempat ditanya
> saat interview, tapi langsung dimasukkan jadi sebuah 'gap' — yang akibatnya, orang yang sebenarnya punya
> skill itu bisa tidak lolos kriteria, cuma karena skill-nya tidak sempat ter-data, bukan karena dia memang
> tidak punya kemampuannya.
>
> Root cause-nya itu karena dua hal yang nyambung satu sama lain. Pertama, sistem lama itu mewajibkan setiap
> skill dapat angka level — jadi walaupun interview-nya belum sampai bahas skill itu, AI tetap dipaksa
> nebak levelnya. Kedua, ada bug parsing: kalau AI jawab dalam bentuk teks kayak 'L3', kode lama itu salah
> baca jadi level 1, bukan level 3 atau malah dianggap gagal. Dua-duanya ketemu, hasilnya: skill yang
> seharusnya 'belum dinilai' malah keluar sebagai 'gap', dengan angka yang juga salah."

**Layar:** Buka `api/app/services/portfolios/skill_status.rb` — file baru, tunjukkan baris 23–39.

**Ucapkan:**
> "Perbaikannya saya taruh di satu file baru, `Portfolios::SkillStatus`, method `derive` di baris 23 — ini
> yang jadi satu-satunya tempat yang memutuskan status sebuah skill, supaya nggak ada lagi logika yang
> tercecer di banyak file—"

**Tunjukkan kode** (`api/app/services/portfolios/skill_status.rb:23-39`):
```ruby
def self.derive(coverage:, raw_level:, evidence_count:)
  level = parse_level(raw_level)
  return Result.new(status: NOT_ASSESSED, level: nil, confidence: "low") if not_yet?(coverage)
  confidence = derive_confidence(coverage: coverage, evidence_count: evidence_count)
  if level.nil? || confidence == "low" || evidence_count.zero?
    Result.new(status: INSUFFICIENT_EVIDENCE, level: level, confidence: confidence)
  else
    Result.new(status: ASSESSED, level: level, confidence: confidence)
  end
end
```

> "— yang paling penting itu baris ketiga ini: begitu ketauan coverage-nya `not_yet`, artinya memang belum
> dibahas, langsung dikembalikan `not_assessed`, titik, nggak peduli AI mau bilang apa soal skill itu. Jadi
> walaupun AI kebetulan ngarang angka yang keliatannya valid, skill itu tetap nggak akan bisa 'lolos' jadi
> dinilai kalau interview-nya memang belum nyampe situ.
>
> Terus ada bug lain yang nyambung ke ini — parser level-nya, baris 67 di file yang sama —"

**Tunjukkan kode** (`api/app/services/portfolios/skill_status.rb:67-77`):
```ruby
def self.parse_level(raw_level)
  case raw_level
  when Integer
    raw_level.between?(1, 5) ? raw_level : nil
  when Float
    raw_level == raw_level.to_i ? parse_level(raw_level.to_i) : nil
  when String
    match = raw_level.strip.match(/\A[Ll]?(\d+)\z/)
    match ? parse_level(match[1].to_i) : nil
  end
end
```

> "— kode lama pakai `.to_i.clamp(1,5)`. Masalahnya, `\"L3\".to_i` itu hasilnya nol, bukan error — dan nol
> di-clamp jadi satu. Jadi kalau AI jawab 'L3', diam-diam kesimpen sebagai level 1. Parser baru ini sengaja
> return `nil` kalau gagal parse, bukan maksa jadi angka valid.
>
> Dua method ini dipanggil dari `Portfolios::Generator`, file `api/app/services/portfolios/generator.rb`,
> method `save_skills` baris 155, pemanggilannya di baris 218 —"

**Tunjukkan kode** (`api/app/services/portfolios/generator.rb:218`):
```ruby
status = Portfolios::SkillStatus.derive(
```

> "— dan di sinilah juga saya temukan bug kedua yang nggak ada hubungannya sama bug utama, murni ketemu pas
> nulis test: ada savepoint transaction di baris 173 yang kalau di-rollback, Rails diam-diam nyimpen ulang
> record yang seharusnya batal — saya fix dengan `portfolio.portfolio_skills.reset` di baris 212. Ini saya
> temukan lewat skrip reproduksi minimal sebelum saya tulis satu baris perbaikan pun — bukan tebakan."

---

## 3:30–4:10 — Kontrak API dan root cause lain

**Layar:** `web/src/types/index.ts` baris 19–24, lalu `web/src/components/assessment/SkillPicker.tsx` baris 41–47.

**Ucapkan:**
> "Masih bagian dari root cause yang sama. Di `types/index.ts` baris 19 sampai 24 — tipe `skill_id` sebelumnya
> `number`, padahal kolom aslinya di database itu `varchar`, string kayak 'SK-ENG-001'. Ini bug tipe data
> yang nyambung ke `SkillPicker.tsx` baris 47 —"

**Tunjukkan kode** (`web/src/components/assessment/SkillPicker.tsx:47`):
```ts
skill_id: s.skill_id,
```

> "— sebelumnya baris ini kirim `undefined` terus, jadi semua pencocokan skill jatuh ke pencocokan nama
> teks, bukan ID — root cause kenapa skill dari taxonomy malah dilabeli 'custom' di prompt AI.
>
> Dan satu lagi, `ComparisonTable.tsx` baris 98 —"

**Tunjukkan kode** (`web/src/components/fitgap/ComparisonTable.tsx:98`):
```tsx
{LEVEL_LABELS[c.expected_level]}
```

> "— sebelumnya frontend baca field `required_level`, padahal backend kirim `expected_level`. Jadi kolom
> 'Required' di tabel fit/gap selalu kosong. Ini bug kecil tapi dampaknya langsung kelihatan di layar."

---

## 4:10–4:35 — Migration database

**Layar:** `api/db/migrate/20261024063750_add_assessment_status_to_portfolio_skills.rb`, scroll ke baris 22–49 dan 93.

**Ucapkan:**
> "Semua status ini disimpan, bukan dihitung ulang tiap kali dibaca — saya buat migration baru. Baris 22,
> bikin enum `assessment_status` tiga nilai. Baris 44 sampai 49, backfill data lama berdasarkan coverage yang
> sudah ada. Dan baris 93, ada CHECK constraint di database level —"

**Tunjukkan kode** (konsep, baris 93 di migration):
```sql
CHECK ((assessment_status <> 'not_assessed') OR (ai_level IS NULL))
```

> "— supaya skill yang belum dinilai itu *tidak bisa* punya angka level tersimpan, sekalipun ada bug di kode
> aplikasi nanti. Constraint-nya yang jaga integritas datanya, bukan cuma disiplin programmer."

---

## 4:35–5:10 — Bukti pengujian: seeded fault test

**Layar:** Terminal, atau `assessment/step-5-execution-evidence.md` bagian seeded fault test.

**Ucapkan:**
> "Supaya saya nggak cuma klaim 'ada test', saya buktikan test-nya beneran mendeteksi regresi. Saya sengaja
> rusak satu baris di `skill_status.rb` baris 25 — persis baris yang saya tunjukkan tadi —"

**Tunjukkan kode (diff seeded fault):**
```diff
- return Result.new(status: NOT_ASSESSED, ...) if not_yet?(coverage)
+ return Result.new(status: NOT_ASSESSED, ...) if not_yet?(coverage) && level.nil?
```

> "— jalankan test suite, 3 dari 42 test gagal, persis di titik yang seharusnya gagal. Saya revert, suite
> balik hijau 42 dari 42. Totalnya 49 test — 42 RSpec, 7 Vitest — saya bangun dari nol karena repo ini
> sebelumnya nggak punya test runner sama sekali, baik di backend maupun frontend."

---

## 5:10–5:35 — Verifikasi AI sungguhan

**Layar:** `api/config/application.yml.sample` baris 20–23.

**Ucapkan:**
> "Dua hal lagi yang cuma ketahuan dengan beneran manggil API-nya. Pertama, model Gemini yang dikonfigurasi
> di repo — `gemini-2.0-flash-001` — sudah mati, saya buktikan dengan panggilan asli yang balasannya
> 'no longer available'. Saya ganti ke `gemini-3.8-flash` setelah verifikasi respons 200 sungguhan.
>
> Kedua, timeout yang sudah ada di kode, 180 detik, ternyata nggak cukup buat model reasoning yang baru —
> saya ukur langsung: gagal di 180 detik, gagal lagi di 120, baru berhasil di 300 detik. Nilai 240 detik yang
> saya pakai sekarang itu dari hasil pengukuran, bukan tebakan."

---

## 5:35–6:00 — Yang sengaja TIDAK dikerjakan

**Layar:** Bagian 8 di laporan ("Di Luar Cakupan & Penutup").

**Ucapkan:**
> "Saya temukan total 15 bug dengan keparahan berbeda-beda, semua saya daftar di laporan. Tapi saya sengaja
> cuma dalami satu — F-05 tadi — sampai tuntas. Sisanya saya dokumentasikan jujur dengan alasan kenapa
> ditunda, termasuk F-03 dan F-04 tadi, IDOR lintas tenant, yang butuh keputusan desain lintas tim, bukan
> tambalan cepat di submission yang sudah bawa satu migration data.
>
> Ini trade-off yang memang diminta brief-nya: kedalaman di satu irisan yang koheren, bukan tambal sulam
> tipis di banyak tempat."

---

## 6:00–6:20 — Penutup

**Layar:** Halaman akhir laporan / link PR.

**Ucapkan:**
> "Itu ringkasan case study ini — tiga bug P0 yang saya temukan dan dua sudah diperbaiki, satu bug P1 yang
> saya dalami sampai akar penyebabnya dengan bukti test dan verifikasi AI sungguhan, dan sisanya
> didokumentasikan jujur. Detail lengkap, termasuk semua kriteria penerimaan dan trade-off, ada di laporan
> yang saya lampirkan. Terima kasih."

---

## Catatan produksi

- Ganti `[Nama]` sebelum rekam.
- Semua nomor baris di atas sudah dicek langsung ke file repo saat naskah ini dibuat — kalau ada perubahan
  kode setelahnya, cek ulang sebelum rekam (`grep -n` nama method/variable yang disebut).
- Kalau waktu mepet (target 3–5 menit), potong bagian **3:30–4:10** (kontrak API) dan **4:10–4:35**
  (migration) — cukup disebut satu kalimat tanpa buka kodenya.
- Bagian yang TIDAK boleh dipotong: **1:40–3:30** (F-05, bug utama) — ini inti yang dinilai, dan
  **5:35–6:00** (yang sengaja tidak dikerjakan) — brief eksplisit minta trade-off dinyatakan jelas.
- Siapkan semua tab file di editor SEBELUM mulai rekam, urutannya persis seperti di naskah ini, supaya tidak
  ada jeda cari file saat rekam jalan.
