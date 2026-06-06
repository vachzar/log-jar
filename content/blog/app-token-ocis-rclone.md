+++
title = "Berburu App Token di ownCloud Infinite Scale — Petualangan yang Tak Terduga"
date = 2026-06-05
description = "Cerita gimana aku nyari fitur App Token di oCIS 8.0.4 yang ternyata gak kelihatan di mana-mana, sampe akhirnya nemu jawaban di tempat yang gak kusangka"
+++

Ceritaku soal OCIS belum selesai. Setelah berhasil bikin HTTPS jalan pake nginx, tantangan berikutnya muncul: **gimana caranya backup isi OCIS ke server lokal?**

Biasanya sih pake **rclone** — tinggal sambungin WebDAV, sync, beres. Tapi kali ini gak semulus itu.

Aku gak mau pake password admin langsung buat koneksi rclone. Udah sewajarnya pake **App Token** — token khusus yang lebih aman, bisa dibatesin masa berlakunya. Masalahnya… OCIS ini **gak punya menu App Token di Web UI-nya**.

---

## "Kok Gak Ada, Ya?"

Biasanya di ownCloud/Nextcloud ada menu *Security* atau *App Password* atau *App Token*. Di OCIS? Nihil. Sama sekali gak ada.

Aku coba akses endpoint WebDAV langsung:

```
curl -I https://awan.vachzar.com/webdav
```

Balasannya `HTTP 401` dengan `WWW-Authenticate: Bearer`. OK, WebDAV-nya hidup. Tapi gimana cara dapet token?

Balik ke Docker Compose. Cek environment variable:

```
docker exec -it ocis-ocis-1 env | grep APP
```

`PROXY_ENABLE_APP_AUTH=true` udah aktif. Tapi pas check service yang jalan:

```
docker exec -it ocis-ocis-1 ocis list | grep auth
```

Cuma ada: `auth-basic`, `auth-machine`, `auth-service`. **Gak ada `auth-app`**!

---

## Solusi Sementara (Yang Sebenarnya Jalan Pintas)

Karena penasaran, aku coba paksa pake **Basic Auth** dulu — tinggal set `PROXY_ENABLE_BASIC_AUTH: "true"` terus coba login pake username/password admin.

Dan… berhasil. WebDAV jalan, rclone connect. Tapi rasanya **jorok** banget. Password admin kepajang di config rclone, gak ada kontrol expiry, gak ada isolasi akses. Harus ada cara yang lebih bener.

---

## "Oh, Ternyata Perlu Ada Service-nya"

Setelah gali-gali dokumentasi OCIS dan forum, nemu satu variable yang selama ini terlewat:

**`OCIS_ADD_RUN_SERVICES: "auth-app"`**

Ternyata `auth-app` ini service opsional. Walaupun `PROXY_ENABLE_APP_AUTH` udah `true`, service-nya sendiri **gak otomatis jalan**. Perlu ditambahin manual.

Aneh sih. Tapi ya sudahlah — ini kan OCIS Community, mungkin emang designnya gitu.

Tambahkan ke Docker Compose:

```yaml
environment:
  PROXY_ENABLE_BASIC_AUTH: "true"
  PROXY_ENABLE_APP_AUTH: "true"
  OCIS_ADD_RUN_SERVICES: "auth-app"
```

Restart container — `docker compose down && docker compose up -d --force-recreate`. Doa sebentar…

Cek lagi:

```
docker exec -it ocis-ocis-1 ocis list | grep auth
```

**`auth-app` muncul!** 🎉

---

## Bikin Token

Setelah service-nya jalan, bikin token gampang banget:

```bash
docker exec -it ocis-ocis-1 ocis auth-app create --user-name admin
```

Keluarlah一串 token. Langsung test:

```bash
curl -u admin:<TOKEN> -X PROPFIND https://awan.vachzar.com/webdav
```

```xml
<d:multistatus>
```

Beres. WebDAV + App Token = match made in heaven.

---

## Rclone + OCIS + Backup Aman

Config rclone-nya:

```
Type: webdav
Vendor: owncloud
URL: https://awan.vachzar.com/webdav
User: admin
Password: <APP_TOKEN>
```

Selesai deh. Sekarang backup jalan dari OCIS ke server lokal pake script simpel:

```bash
rclone sync ocis-source:/ /home/fajar/backup/ \
  --backup-dir "/home/fajar/backup-deleted/$(date +%F)" \
  --fast-list --progress
```

Yang keren: file yang kehapus dari OCIS **gak ilang** dari backup — dipindahin ke folder tanggal. Safety net yang bikin tidur lebih nyenyak.

---

## Biar Otomatis: Systemd Timer > Cron

Awalnya mau pake cron, tapi mending **systemd timer**. Kenapa?

- **Lebih gampang dipantau** — `systemctl list-timers`, `journalctl -u service-name`
- **Otomatis jalan setelah reboot** tanpa mikir
- **Persistent** — kalo mati pas jam schedule, jalan pas nyala lagi

### 1. Bikin Script Backup

```bash
mkdir -p /home/fajar/scripts
```

Bikin file `/home/fajar/scripts/rclone-ocis-sync.sh`:

```bash
#!/bin/bash

LOGDIR="/home/fajar/logs"
mkdir -p "$LOGDIR"

rclone sync \
    ocis-source:/ \
    /home/fajar/backup/ \
    --backup-dir "/home/fajar/backup-deleted/$(date +%F)" \
    --fast-list \
    --progress \
    --log-file="$LOGDIR/rclone-sync.log" \
    --log-level INFO
```

Beri izin:

```bash
chmod +x /home/fajar/scripts/rclone-ocis-sync.sh
```

### 2. Bikin Systemd Service

`/etc/systemd/system/rclone-ocis-sync.service`:

```ini
[Unit]
Description=Sync OCIS to Local Backup

[Service]
Type=oneshot
User=fajar
ExecStart=/home/fajar/scripts/rclone-ocis-sync.sh
```

### 3. Bikin Timer

Misalnya **setiap 1 jam**:

`/etc/systemd/system/rclone-ocis-sync.timer`:

```ini
[Unit]
Description=Run OCIS Sync Every Hour

[Timer]
OnBootSec=5min
OnUnitActiveSec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

### 4. Aktifkan

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now rclone-ocis-sync.timer
```

Cek:

```bash
systemctl list-timers | grep rclone
```

### 5. Tes Jalan Manual

```bash
sudo systemctl start rclone-ocis-sync.service
tail -f /home/fajar/logs/rclone-sync.log
# atau
journalctl -u rclone-ocis-sync.service -f
```

### Variasi Jadwal

| Kecepatan | Config Timer |
|-----------|-------------|
| **15 menit** | `OnUnitActiveSec=15min` |
| **Setiap jam** | `OnUnitActiveSec=1h` |
| **Setiap hari jam 01:00** | `OnCalendar=*-*-* 01:00:00` |

### Penting: Sync vs Copy

Pake `rclone sync` berarti file yang dihapus dari OCIS **ikut kehapus** di backup. Kalo lo mau aman, tambah `--backup-dir` (udah di script di atas) atau ganti pake `rclone copy` biar file backup gak pernah ilang.

---

## Pelajaran dari Petualangan Ini

1. **Jangan percaya config variable doang.** `PROXY_ENABLE_APP_AUTH=true` gak berarti service-nya jalan. Kadang perlu explicit `OCIS_ADD_RUN_SERVICES`.
2. **OCIS Community itu… interesting.** Fitur opsi, dokumentasi gak selalu lengkap, beberapa service perlu dinyalakan manual.
3. **Basic Auth itu jalan pintas, bukan solusi.** Bisa sih dipake, tapi App Token jauh lebih proper.
4. **`--backup-dir` di rclone itu lifesaver.** Udah pernah kejadian file kehapus dan baru sadar seminggu kemudian? Ini solusinya.
5. **Systemd timer > cron.** Lebih terintegrasi, gampang di-monitor, otomatis persistent.

---

## Status Sekarang

OCIS 8.0.4 + App Token + rclone sync berjalan dengan baik. Backup otomatis tiap jam via systemd timer, file yang terhapus aman di folder backup-deleted. Tinggal lupa urusan storage.

Satu lagi centang di list self-host.
