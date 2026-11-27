---
name: bug_bounty_hunt
description: "Test vulnerability classes on an authorized, recon-mapped target before gate validation and reporting. Adapts public bug-bounty hunt patterns per class; every hunt ends at the validation gate."
---

# Bug Bounty Hunt

Gunakan skill ini saat Muse menguji kelas kerentanan pada target yang sudah diotorisasi dan sudah dipetakan lewat recon, sebelum temuan itu divalidasi dan dilaporkan. Skill ini mengadaptasi pola hunt per kelas dari bundle bug bounty publik untuk Muse dan selalu mengakhiri hunt di gate validasi.

## Mulai dari scope dan permukaan serangan

- Konfirmasi target, scope program, akun uji yang boleh dipakai, dan batas yang tidak boleh disentuh sebelum menguji kelas kerentanan apa pun.

- Pilih kelas kerentanan dari permukaan serangan yang benar-benar terlihat di recon, bukan dari daftar generik, dan catat endpoint, parameter, dan peran akun yang dipakai untuk setiap pengujian.

- Uji satu hipotesis dalam satu waktu dan catat input yang dikirim, respons yang diterima, dan perubahan state yang teramati, supaya temuan bisa diulang oleh orang lain.

- Kerja paralel untuk cek independen; verifikasi on-chain via RPC langsung dulu sebelum browser (browser paling lambat); parse output tool ke field minimal; satu verifikasi per item per turn; balasan ringkas, hash bentuk pendek.

## Uji kelas injection dan server side

- Uji XSS dengan membedakan konteks refleksi, yaitu konteks HTML, atribut, JavaScript, dan URL, dan buktikan eksekusi dengan marker unik sebelum mengklaim dampak.

- Uji SQL injection dengan canary boolean dan error yang tidak merusak dan jangan menjalankan dump data atau perintah destruktif untuk membuktikan temuan.

- Uji SSRF dengan canary out of band dan peta jalur dari parameter ke request server side, termasuk metadata cloud dan layanan internal yang bisa dijangkau.

- Uji SSTI dengan canary matematika dan bedakan template engine yang dipakai sebelum menyimpulkan eksekusi template.

- Uji XXE, HTTP request smuggling, dan cache poisoning hanya pada endpoint yang menunjukkan parser, header, atau cache yang relevan di recon.

## Uji otorisasi dan identitas

- Uji IDOR dan broken object level authorization dengan membandingkan objek milik akun sendiri dan objek milik akun uji lain yang diotorisasi untuk pengujian, dan catat perbedaan akses yang teramati.

- Uji auth bypass dengan memvariasikan peran, token, dan kondisi session, dan jangan memakai kredensial pengguna nyata atau kredensial yang bocor di luar pengujian.

- Akun uji: jangan simpan login akun kedua ke Secure Vault untuk site yang sama — Vault hanya simpan satu login per site dan yang kedua menimpa yang pertama. Waspada autofill browser yang menimpa field sign-in dengan login tersimpan.

- Uji JWT dengan memeriksa algoritma, verifikasi signature, masa berlaku, dan klaim yang dipercaya server, dan coba variasi yang relevan tanpa menebak secret secara brute force massal.

- Uji OAuth dan SAML dengan memetakan redirect, state, dan binding antara identitas dan session, dan buktikan pengambilalihan akun hanya lewat akun uji yang diotorisasi.

- Uji MFA bypass, CSRF, dan race condition dengan skenario yang bisa diulang dan dengan dampak yang terukur, bukan hanya dengan anomali respons sekali jalan.

## Uji API dan logika bisnis

- Uji API dengan menginventarisasi endpoint dari dokumentasi dan dari file JavaScript, lalu uji mass assignment, otorisasi per objek, rate limit, dan validasi input per endpoint.

- Uji GraphQL dengan menginventarisasi query, mutation, dan skema yang terekspos, lalu uji otorisasi dan introspeksi hanya pada endpoint yang masuk scope.

- Uji file upload dengan memeriksa tipe file, ekstensi, dan jalur penyimpanan yang diizinkan server, dan buktikan eksekusi atau akses file hanya dengan file uji yang aman.

- Uji logika bisnis dengan memetakan alur normal, yaitu harga, kuantitas, status order, dan hak akses, lalu cari langkah yang bisa dilewati, diulang, atau dibalik urutannya.

- Rangkai temuan menjadi chain hanya jika setiap tautan rantai sudah terbukti sendiri, misalnya IDOR yang mengarah ke auth bypass, SSRF yang mengarah ke metadata cloud, atau XSS yang mengarah ke pengambilalihan akun lewat akun uji.

## Serahkan ke gate validasi sebelum laporan

- Serahkan setiap kandidat ke skill gate_validasi_hipotesis dan terapkan verdict CONFIRMED, FALSIFIED, atau INCONCLUSIVE beserta evidence tier dan gate checklist.

- Buang kandidat yang hanya berupa scanner hit tanpa bukti eksekusi dan jangan naikkan severity hanya karena kelas kerentanannya terdengar berat.

- Sebelum verdict final, jalankan review Antigravity bila kuota tersedia; kuota habis 12 Nov 2026, reset ~18 Nov 2026 — sampai reset, gantinya banyak pilot paralel (satu subagent per track) + self-review ganda per pilot. Kalau Antigravity bertentangan dengan Muse, verifikasi ke bukti dulu, jangan telan mentah-mentah.

- Temuan yang memenuhi bar program tetap diajukan walau kecil/Low (perintah user 12 Nov 2026) — ini tidak menurunkan bar: yang belum tervalidasi atau di bawah bar tetap bukan temuan.

- Tulis laporan dengan dampak yang terukur, langkah reproduce yang lengkap, dan bukti yang sudah diredaksi dari cookie dan data pribadi yang tidak relevan.

## Available connectors

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.

## Websites

- [Claude-BugHunter](https://github.com/xiaolai/Claude-BugHunter) for peta kemampuan hunt per kelas, engagement flow, dan pola dari laporan yang sudah dipublikasikan. Bawa kelas kerentanan dan rantai yang relevan ke tugas dan verifikasi bahwa kelas itu cocok dengan permukaan serangan target sebelum mengujinya.
