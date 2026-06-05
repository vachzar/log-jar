+++
title = "Belajar Setup OCIS dengan HTTPS — Cerita dari Depan Layar"
date = 2026-06-05
description = "Gimana aku setup ownCloud Infinite Scale 8 dengan HTTPS, dibalikin dari Caddy ke nginx, dan pelajaran soal Cookie yang bikin pusing"
+++

Beberapa waktu lalu aku butuh akses file pribadi, **dari mana aja**. Keluarga juga kadang perlu upload/download dokumen, jadi aku pikir daripada pake Google Drive mending self-host aja. Pilihannya jatuh ke **ownCloud Infinite Scale (OCIS)** — katanya sih modern, cepat, gak butuh database.

Dan ternyata… banyak drama.

---

## Arsitektur yang Kebetulan Kompleks

Jadi begini setup-ku:

- **Server utama** — minipc di rumah (192.168.1.2), jalanin OCIS di Docker
- **VPS** — buat jembatan dari luar
- **AdGuard Home** — rewrite `awan.vachzar.com` ke IP lokal biar kalau akses dari rumah gak perlu keluar internet dulu

```
Dari luar → VPS (NPM) → Tailscale → OCIS ✅
Dari rumah → AdGuard rewrite → nginx lokal → OCIS ✅
```

Maksud sih simpel. Ternyata dari lokal aja udah rempong duluan.

---

## Kenapa Gak Pake Caddy Aja?

Awalnya aku pake **Caddy**. Instal gampang, SSL otomatis lewat Cloudflare DNS module. Tapi… pas udah nyala, OCIS ngasih **error 400**. Pesannya soal **XSRF token**. Intinya cookie `__Secure-KKBS` gak kebentuk sama sekali.

Caddy sih nge-forward request-nya pake TLS. Tapi entah kenapa header yang dikirim ke OCIS gak sesuai, dan OCIS sebagai IDP (Identity Provider) gak mau ngasih cookie itu kalau gak yakin koneksinya aman. Pokoknya **Caddy gagal total** buat setup ini.

Aku udah coba oprek config Caddy, tapi makin gali makin dalem. Akhirnya nyerah.

---

## Balik ke nginx — Seperti Old Friend

Pas install nginx, rasanya kayak balik ke rumah. Config filenya straightforward. Ini dia konfigurasi yang akhirnya work:

- **SSL** pake **ZeroSSL** via **acme.sh** — DNS-01 challenge pake Cloudflare API token
- **Cert disimpan** di `/etc/nginx/ssl/`
- **Nginx proxy_pass** OCIS via HTTPS, tapi dengan `proxy_ssl_verify off` — karena OCIS pake self-signed cert di internal

Yang paling penting: **nginx ngasih header yang bener**. Nggak tahu kenapa Caddy gagal di bagian ini, tapi nginx jalan mulus begitu config dipasang.

---

## Pelajaran yang Kudapat

1. **Gak semua reverse proxy cocok buat semua backend.** Caddy keren buat banyak hal, tapi OCIS 8.0.4 pilih kasih.
2. **Cookie itu ternyata penting banget.** XSRF token yang gak kebentuk bisa bikin semuanya berhenti.
3. **AdGuard rewrite is clutch.** Biar trafik lokal gak muter keluar internet dulu baru balik lagi — latency turun drastis.
4. **acme.sh + DNS challenge > HTTP challenge.** Gak perlu buka port 80, cert jalan terus.

---

## Status Sekarang

OCIS 8.0.4 jalan di Docker (port 8001), di belakang nginx. HTTPS works dari luar lewat VPS, juga dari lokal lewat AdGuard → nginx. Keluarga bisa upload/download dengan tenang.

File config nginx dan cert files aman disimpan, SSL renewal otomatis di-handle cron acme.sh tinggal reload nginx.

Kadang solusi paling sederhana emang yang terbaik. Nginx + acme.sh. Old but gold.
