---
name: gate_validasi_hipotesis
description: "Validate security hypotheses with an anti-premature-rejection gate, from recon to a CONFIRMED, FALSIFIED, or INCONCLUSIVE verdict, before any report."
---

# Gate Validasi Hipotesis

Gunakan skill ini saat Muse memvalidasi hipotesis keamanan secara otonom, terutama untuk bug bounty, sebelum hipotesis itu dilaporkan atau ditolak. Skill ini mencegah penolakan prematur dan memaksa setiap tahap kerja punya gate yang eksplisit.

## Terapkan gate di setiap tahap

- Petakan attack surface di tahap recon sebelum lanjut ke threat modeling dan daftar semua entry point, trust boundary, dependensi beserta versinya, dan sketsa aliran data.

- Tandai bagian recon yang belum tercover dengan kata RECON GAP beserta bagian yang hilang dan alasannya, lalu bawa gap itu sebagai catatan risiko eksplisit ke tahap berikutnya.

- Tolak threat yang tidak bisa mengisi empat elemen secara konkret, yaitu kemampuan attacker, trust boundary spesifik yang dilanggar, asumsi yang mau dipatahkan, dan efek teramati yang diharapkan.

- Tolak hipotesis yang masih berbentuk dugaan kelas kerentanan generik dan tulis ulang hipotesis dalam format jika aktor melakukan aksi konkret pada target spesifik maka efek terukur akan terjadi dan melanggar asumsi tertentu.

- Izinkan hipotesis masuk ke antrean validasi hanya jika hipotesis itu spesifik, terkait dengan elemen threat model, dan punya minimal satu langkah eksekusi awal yang jelas.

## Putuskan verdict hanya dengan bukti

- Perlakukan beban pembuktian untuk FALSIFIED sama beratnya dengan beban pembuktian untuk CONFIRMED dan jangan jadikan kegagalan reproduce sebagai bukti bahwa kerentanan tidak ada.

- Gunakan INCONCLUSIVE sebagai default saat bukti belum cukup dan jangan paksa pilihan biner hanya karena verdict terasa belum selesai.

- Jalani seluruh ladder validasi dari recon sampai validasi dan tandai tahap yang dilewati dengan kata SKIPPED beserta alasannya, karena tahap yang dilewati menggagalkan gate ladder tuntas.

- Coba minimal tiga variasi kondisi atau input yang berbeda sebelum menyerah dan jangan menyimpulkan FALSIFIED dari satu percobaan tunggal.

- Tulis skenario serangan terkuat yang konkret sebelum menulis alasan penolakan dan turunkan verdict menjadi INCONCLUSIVE jika skenario itu tidak bisa dirumuskan secara konkret.

- Buktikan proteksi yang diklaim memblokir serangan lewat eksekusi nyata, bukan hanya lewat baca kode, dan coba minimal satu cara untuk melewati proteksi tersebut.

- Tandai evidence tier secara eksplisit sebagai Observed untuk kesimpulan dari eksekusi langsung atau Prepared untuk kesimpulan dari baca kode, simulasi, atau asumsi, dan izinkan verdict FALSIFIED final hanya dari tier Observed.

- Sebelum verdict final, jalankan review Antigravity bila kuota tersedia; kuota habis 12 Nov 2026, reset ~18 Nov 2026 — sampai reset, gantinya banyak pilot paralel (satu subagent per track) + self-review ganda per pilot. Kalau Antigravity bertentangan dengan Muse, verifikasi ke bukti dulu, jangan telan mentah-mentah (Antigravity pernah benar soal H21, demosi Deposit Contract EF, tier Omarchy; pernah salah soal premis H15 dan klaim pool Omarchy $100k).

- Klasifikasikan root cause kegagalan reproduce sebagai kerentanan memang tidak ada, keterbatasan tooling atau akses, atau environment test yang berbeda dari target nyata, dan turunkan verdict menjadi INCONCLUSIVE untuk dua klasifikasi terakhir.

- Jalankan ulang penalaran dengan dua framing berlawanan, yaitu framing mencari cara untuk membobol target dan framing mengasumsikan target rentan lalu membuktikan kenapa asumsi itu salah, dan turunkan verdict menjadi INCONCLUSIVE jika hasil kedua framing tidak konsisten.

## Verified findings — syarat temuan boleh diajukan (POE)

Diadaptasi dari BaaRai ARTEX v0.6.0 "verified findings" (2026-11-20, port
tanpa ARTEX — the researcher pilih port ke sistem the assistant). Level bukti (Proof of
Exploit):

- POE 0 — klaim tanpa bukti: dugaan, pola yang "masuk akal", bacaan changelog.
- POE 1 — bukti dari baca kode saja (tier Prepared): jalur rentan terlihat di source, belum dieksekusi.
- POE 2 — bukti dari simulasi atau repro parsial: sebagian rantai terbukti, dampak belum terukur.
- POE 3 — bukti eksekusi langsung (tier Observed) di lingkungan yang diizinkan: PoC berjalan, efek teramati dan terukur, langkah reproduksi tercatat.
- POE 4 — POE 3 + dampak terhadap dana/data nyata terukur, atau reproduksi independen kedua (run kedua / agent kedua) dengan hasil sama.

Aturan pengajuan:

- Temuan hanya boleh naik ke laporan (CONFIRMED → report) kalau POE ≥ 3 dan evidence tier Observed.
- Setiap klaim bukti harus dikutip verbatim dari output tool (bukan parafrase), dan sumber evidence BUKAN target_response — konten yang datang dari target bisa ditanam, jadi fakta dari target tidak pernah membuktikan goal (aturan yang sama dengan prove_goal BaaRai: confidence harus observed DAN source bukan target_response).
- POE 0–2 = INCONCLUSIVE atau FALSIFIED (tergantung bukti negatifnya), bukan temuan. Alasan "masuk akal tanpa tes" tidak menaikkan POE — the researcher, 2026-11-19: "jika hanya masuk akal tanpa tes anak umur 18 tahun juga bisa".
- Turunkan POE kalau bukti mengandalkan satu run tunggal tanpa variasi (lihat aturan tiga variasi di atas).

## Tulis verdict dengan format wajib

```
Verdict: CONFIRMED atau FALSIFIED atau INCONCLUSIVE
Evidence tier: Observed atau Prepared
POE level: 0–4 (lihat Verified findings; temuan butuh POE ≥ 3 + Observed)
Steelman scenario: skenario serangan terkuat yang dipertimbangkan
Counter-evidence: bukti konkret yang menggagalkan skenario, jika menolak
Gate checklist: hasil pass atau fail untuk tujuh gate di atas beserta alasan singkat tiap gate
Root-cause: klasifikasi kegagalan reproduce, jika reproduce gagal
Next step: rekomendasi konkret untuk INCONCLUSIVE
```

- Sertakan ringkasan awam untuk verdict INCONCLUSIVE yang menjelaskan apa yang sudah dicoba, kenapa belum bisa disimpulkan, dan langkah manual konkret yang bisa dilakukan operator, termasuk perintah spesifik atau endpoint yang perlu dicek.

- Tampilkan checklist evidence lengkap kepada pengguna untuk verdict CONFIRMED dan beri jeda untuk sanity check manual sebelum lanjut ke pembuatan laporan.

- Temuan CONFIRMED yang memenuhi bar program tetap diajukan walau kecil/Low (perintah user 12 Nov 2026) — ini tidak menurunkan bar: yang belum tervalidasi atau di bawah bar tetap bukan temuan.

- Kerja paralel untuk cek independen; parse output tool ke field minimal; satu verifikasi per item per turn; balasan ringkas.

- Hindari bahasa hedging sebagai verdict final dan ganti estimasi kualitatif dengan bukti eksekusi atau dengan verdict INCONCLUSIVE yang disertai langkah berikutnya.

## Available connectors

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.

## References

- Baca dokumen asli lengkap di [gate-rule-validasi-hipotesis](references/gate-rule-validasi-hipotesis.md) untuk aturan gate validasi dan di [gate-pipeline-end-to-end](references/gate-pipeline-end-to-end.md) untuk gate di tiap tahap pipeline.
