+++
title = "Bikin Knowledge Base Sendiri — Hermes + Flask + Markdown"
date = 2026-06-07
description = "Cerita gue bikin web UI note-taking pribadi pake Hermes API + Flask backend + markdown files, plus fitur graph view kaya Obsidian"
+++

Beberapa minggu lalu gue punya masalah: note gue berserakan. Ada di Hermes (konteks chat), ada di OCIS file server, ada di file `.md` lokal. Masing-masing terisolasi. Susah nyari catetan lama.

Yang gue butuhin sebenarnya simpel:
- Note di **markdown** (biar portable)
- Bisa **search** cepet
- Ada **web UI** (biar bisa akses dari HP)
- Dark theme (karena gue nocturnal)
- **Remote access** via Tailscale

Gue punya Hermes Agent yang udah bisa akses file lokal, jadi gue pikir: *kenapa gak sambungin aja semua?*

---

## Arsitektur yang Kepikiran

Gue gambar di kepala:

```
Hermes (Telegram) ↔ ~/notes/ (.md files) ↔ Flask API ↔ Web UI (responsive)
                                             ↕ 
                                        OCIS WebDAV (backup)
```

Jadi:
- **Source of truth:** folder `~/notes/` — semua file `.md` biasa
- **Hermes skill:** baca/tulis/search note via Telegram
- **Web UI:** Flask serve static HTML + API buat browse/edit
- **Backup:** OCIS WebDAV via rclone, biar aman

Nggak ada database. Nggak ada server berat. Cuma Flask + filesystem.

---

## Bikin Backend

Gue buka laptop, terminal, mulai nulis `backend.py`. Satu file doang — Flask biasa:

```python
@app.route('/api/notes')     # List semua notes + folder tree
@app.route('/api/note')      # Baca note + frontmatter + backlinks
@app.route('/api/search?q=') # Search isi notes
@app.route('/api/save')      # Create / update note
@app.route('/api/delete')    # Hapus note
```

Fitur penting yang gue masukin:
- **YAML frontmatter** — parsing tags, dates dari `---` block
- **[[Wikilink]] detection** — nyari cross-reference antar note
- **Backlinks** — note mana aja yang nge-link ke note ini
- **Markdown → HTML** — pake library `markdown` biar tampilannya rapi

Yang paling gue suka: fitur backlink. Bisa liat "oh note ini direferensi sama note A, B, C" — persis kaya Obsidian.

## Frontend: 3-Column Dark UI

Untuk web UI, gue buat **single HTML file** pake Tailwind CDN. Struktur layout:

```
[icon bar 56px] [sidebar (slide)] [main content]
```

**Desktop:**
- Icon bar kiri (w-14, sticky) — Browse, Graph, Daily, OCIS status
- Sidebar slide-in — file tree + search
- Konten utama — note yang lagi dibaca

**Mobile (<768px):**
- Hamburger menu → sidebar slide
- Full-width content
- Search toggle inline

Warna pake palet gelap gue sendiri:

| Token | Hex | Buat |
|-------|-----|------|
| Sidebar | `#0f1117` | Background icon bar |
| Surface | `#16181f` | Background utama |
| Accent | `#6c5ce7` | Tombol aktif |
| Text | `#e4e6ef` | Tulisan utama |

Gue pilih Tailwind CDN biar gak perlu build step. Edit HTML → refresh browser → liat hasil. Cepet banget buat prototyping.

### CSS Flex Gotcha

Yang bikin gue pusing sejam: **body harus `display: flex`**. Kedengeran sepele, tapi tanpa itu icon bar dan konten utama numpuk vertikal, bukan horizontal. Klasik.

```css
body { display: flex; min-height: 100vh; overflow-x: hidden; }
```

Gue tulis di sini biar gue sendiri gak lupa nanti.

## Graph View — Obsidian-like Canvas

Gue pengen ada visualisasi koneksi antar note. Seperti Obsidian Graph View. Tapi tanpa library berat kayak D3.js atau vis.js.

Solusi: **Canvas API murni.**

```javascript
function drawGraph() {
    const notes = allNotes.filter(n => n.wikilinks?.length);
    const centerX = canvas.width / 2;
    const centerY = canvas.height / 2;
    const radius = Math.min(centerX, centerY) * 0.6;

    notes.forEach((note, i) => {
        const angle = (2 * Math.PI * i) / notes.length;
        const r = radius * (0.7 + Math.random() * 0.3);
        // Gambar node di posisi (centerX + cos(angle)*r, centerY + sin(angle)*r)
        // Gambar garis ke linked notes
    });
}
```

Hasilnya? Kira-kira kaya spider web — semua note terhubung bikin lingkaran. Node yang aktif (note yang lagi dibaca) di-highlight ungu `#6c5ce7`. Sederhana, ringan, gak perlu load library 500KB.

## Integrasi Hermes

Bagian paling keren: **Hermes bisa akses notes yang sama dari Telegram.**

Gue bikin Hermes skill `vachzar-notes` yang bisa:
- `read note devops/hermes-agent-setup` — baca note + frontmatter + backlinks
- `search notes docker` — search isi semua file `.md`
- `list notes` — liat folder tree
- `bikin catatan bkd/resume.md — content` — bikin note baru

Jadi workflow gue sekarang:
**Di HP (Telegram)** → Hermes → `~/notes/` → **Web UI di laptop**
**Di laptop** → Web UI → `~/notes/` → **Hermes nanti bacanya sama**

Semua sinkron karena file-nya sama.

## Auto-start via systemd

Biar gak perlu nyalain manual tiap boot:

```bash
sudo tee /etc/systemd/system/vachzar-notes.service << 'EOF'
[Unit]
Description=Vachzar Notes Web UI
After=network.target

[Service]
Type=simple
User=byo
WorkingDirectory=/home/byo/vachzar-notes
ExecStart=/usr/bin/python3 backend.py
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable --now vachzar-notes
```

Akses via:
- `http://localhost:7777` — lokal
- `http://192.168.1.2:7777` — LAN
- `http://100.105.243.46:7777` — Tailscale (dari luar)

## OCIS Backup

Biar notes gak ilang kalo SSD jebol, gue sync ke OCIS WebDAV pake rclone:

```bash
rclone sync ~/notes/ ocis-notes:/ --backup-dir "backup-deleted/$(date +%F)"
```

Jalan tiap jam via systemd timer. File kehapus aman di folder tanggal.

---

## Kenapa Ini Cocok Buat Gue

1. **Gak ada vendor lock-in** — notes file `.md` biasa, bisa dibaca text editor mana pun
2. **Hermes integration** — bisa akses dari Telegram kapan aja
3. **Ringan** — Flask + filesystem, gak perlu database, gak perlu Node.js build step
4. **Mobile friendly** — Tailwind responsive, jalan di HP Redmi Note 12 gue
5. **Graph view** — Obsidian-like tanpa Obsidian

Yang paling pentung: **semua ada di satu folder `~/notes/`**. Gue bisa `grep`, `rsync`, `git commit`, apa aja. Ini yang bikin gue milih pendekatan file-based dibanding pake aplikasi note-taking yang udah jadi.

---

## Status Sekarang

Vachzar Notes berjalan di systemd, bisa diakses dari mana aja via Tailscale. Hermes skill udah terintegrasi — tinggal chat di Telegram, note kegenerate. Graph view-nya kadang gue buka cuma buat liat-lihat jaringannya aja.

Masih ada yang mau gue tambahin:
- **Daily notes template** — auto-create note harian pas dibuka
- **Reverse proxy** ke `notes.awan.vachzar.com`
- **Drag & drop** upload file ke notes

Tapi untuk MVP? Udah lebih dari cukup. Notes gue sekarang gak berserakan lagi.
