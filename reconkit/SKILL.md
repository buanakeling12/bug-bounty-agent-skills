---
name: reconkit
description: "Authorized bug bounty recon with a scope gate, sequential recon modules, and canary-based safe validation from C0 to C4."
---

# ReconKit Bug Bounty Recon

Gunakan skill ini saat Muse melakukan recon untuk bug bounty pada target yang sudah diotorisasi, sebelum masuk ke hunt per kelas kerentanan. Skill ini mengadaptasi metodologi ReconKit untuk Muse dan mengutamakan scope gate, deteksi, dan validasi aman dengan canary.

## Terapkan scope gate dulu

- Minta pengguna menyebutkan target dan bukti otorisasi sebelum menjalankan recon aktif, yaitu program bug bounty yang sedang berlaku, aturan engagement, atau kepemilikan aset sendiri.

- Catat target yang diotorisasi dalam satu daftar scope dan tolak menjalankan modul apa pun terhadap host yang tidak ada di daftar itu.

- Gunakan hanya recon pasif untuk target yang belum terkonfirmasi otorisasinya dan jangan anggap domain yang mirip atau subdomain yang ditemukan sebagai bukti bahwa domain itu masuk scope.

- Simpan secret dan kunci API di luar repo dan di luar laporan dan jangan pernah menulis secret ke file output yang dibagikan.

## Jalankan modul recon berurutan

- Enumerasi subdomain secara pasif dari certificate transparency, DNS, dan sumber publik, lalu gabungkan dan hilangkan duplikat sebelum probing.

- Probing host yang hidup untuk status, judul, dan teknologi, lalu petakan TLS dan sertifikat untuk menemukan host yang salah konfigurasi atau sudah kedaluwarsa.

- Crawl URL dari host yang hidup dan kumpulkan endpoint, parameter, file JavaScript, dan referensi cloud secara read only.

- Cari parameter tersembunyi, path sensitif, dan endpoint API dari hasil crawl dan dari file JavaScript sebelum memilih kelas kerentanan yang mau diuji.

- Jalankan template nuclei untuk CVE, takeover, panel, dan misconfig hanya pada host yang hidup dan sudah masuk scope, dan catat template yang gagal atau tool yang hilang sebagai gap, bukan sebagai hasil bersih.

- Buat peringkat attack surface dari hasil recon dengan bobot program, yaitu host yang paling bernilai, endpoint yang paling sensitif, dan temuan yang paling bisa dibuktikan dampaknya.

- Cek independen jalan paralel; verifikasi on-chain via RPC langsung dulu sebelum browser (browser paling lambat); parse output tool ke field minimal; satu verifikasi per item per turn.

## Validasi aman dengan prove dan confidence

- Beri setiap kandidat temuan level confidence dari C0 sampai C4, yaitu C0 untuk noise, C1 untuk scanner hit, C2 untuk canary yang terkonfirmasi, C3 untuk dampak yang sudah dinilai manusia, dan C4 untuk report ready.

- Validasi kandidat hanya dengan marker atau canary yang aman, misalnya marker refleksi untuk XSS, canary matematika untuk SSTI, fingerprint takeover untuk DNS, dan recheck nuclei yang aman.

- Matikan secara default teknik yang merusak atau yang mengambil data, termasuk sqlmap, reverse shell, credential stuffing, dan dump data, kecuali pengguna memberi otorisasi eksplisit untuk teknik itu pada target itu.

- Buat attack graph sederhana dari kandidat yang terkonfirmasi, yaitu aset mana yang mengarah ke aset lain dan rantai mana yang paling mungkin menghasilkan dampak nyata.

- Serahkan kandidat C2 ke atas ke skill gate_validasi_hipotesis untuk verdict final dan jangan langsung menulis laporan dari scanner hit C1.

## Available connectors

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.

## Websites

- [ReconKit](https://thevillagehacker.com/projects/reconkit.html) for modul recon, scope gate, dan prove yang aman. Bawa nama modul dan kebijakan prove dari halaman itu ke tugas dan verifikasi bahwa target sudah masuk scope sebelum menjalankan modul apa pun.

- [ReconKit GitHub](https://github.com/thevillagehacker/Reconkit) for struktur repo, perintah CLI, dan layout output. Bawa struktur output itu sebagai acuan penamaan file dan verifikasi bahwa tool yang dibutuhkan tersedia sebelum mengklaim satu modul sudah selesai.
