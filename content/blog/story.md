+++
title = "🔋 Perjalanan Membangun Bluetooth Battery Monitor"
date = 2026-06-19
description = "Sebuah cerita tentang eksperimen coding, debugging, dan akhirnya berhasil membaca baterai speaker Bluetooth di Windows"
+++

# 🔋 Perjalanan Membangun Bluetooth Battery Monitor

> Sebuah cerita tentang eksperimen coding, debugging, dan akhirnya berhasil membaca baterai speaker Bluetooth di Windows.

**Author:** JARxAI  
**Date:** June 2026  
**Tools:** Python, PowerShell, C#, Windows API

---

## 📖 Mulai dari Masalah Sederhana

> "Speaker bluetooth aku konek di laptop, bisa nggak sih bikin aplikasi kecil yang nampilin device yang konek dan bar baterainya?"

Itu pertanyaan awal yang memulai semuanya. Kedengarannya sederhana kan? Ternyata... **sangat tidak sederhana.**

---

## 🔄 Fase 1: Python + PowerShell (Awal)

Mulai dengan script Python pakai `pystray` untuk system tray icon. Rencananya:
- Scan device Bluetooth via PowerShell
- Baca baterai via WMI
- Tampilkan di tray

```python
# Versi pertama - looks simple right?
def get_battery():
    ps = "Get-WmiObject Win32_Battery | Select EstimatedChargeRemaining"
    return run_ps(ps)
```

**Hasil:** Tampil 79% ❌

**Masalah:** `Win32_Battery` itu baca baterai **LAPTOP**, bukan speaker Bluetooth! 🤦

---

## 🔍 Fase 2: Mencari Cara Baca Baterai Speaker

Mulai research: "How to read Bluetooth speaker battery on Windows?"

### Percobaan 1: PnP Device Properties
```powershell
Get-PnpDevice -FriendlyName '*Play 2*' | Get-PnpDeviceProperty
```

**Hasil:** Ada property `{104EA319-6EE2-4701-BD47-8DDBF425BBE5} 2` yang return battery level!

**Masalah:** Device yang benar ada di "Hands-Free AG", bukan di "Avrcp Transport" yang terdeteksi.

### Percobaan 2: WinRT API (Python)
```python
from winrt.windows.devices.bluetooth import BluetoothAdapter
```

**Hasil:** Gagal. Device adalah Classic Bluetooth (A2DP), bukan BLE. WinRT hanya support BLE devices.

### Percobaan 3: C# dengan Win32 Bluetooth API
Buat C# project untuk akses `BluetoothFindFirstDevice` API.

**Hasil:** Device terdeteksi, tapi tetap tidak bisa baca battery. Windows Settings pakai API internal yang tidak tersedia untuk third-party.

---

## 💡 Fase 3: StackOverflow ke Rescue!

Nemukan [thread ini](https://stackoverflow.com/questions/71736070) yang menjelaskan:

> "Battery info ada di property `{104EA319-6EE2-4701-BD47-8DDBF425BBE5} 2` pada device yang tepat."

**Key insight:** Harus search semua device yang match nama speaker, bukan hanya device pertama!

```powershell
# Yang benar
Get-PnpDevice -FriendlyName '*Play 2*' | ForEach-Object {
    $test = $_ | Get-PnpDeviceProperty -KeyName '{104EA319-...} 2'
    if ($test) { $test.Data }
}
```

**Hasil:** 70% ✅ (bukan 79% yang salah tadi!)

---

## 🐛 Fase 4: Bug Hunting Maraton

### Bug 1: ERAZER muncul meskipun disconnected
**Masalah:** Script tampilkan semua device yang punya battery, termasuk yang sudah mati.

**Penyebab:** Windows cache battery data! Device yang pernah connected tetap punya battery value meskipun sudah disconnected.

### Bug 2: Options tidak work
**Masalah:** Pilih device di Options, tapi tetap tampilkan semua.

**Penyebab:** Logic filtering tidak ada di code!

### Bug 3: Script tidak muncul sama sekali
**Masalah:** Background scan belum selesai saat icon dibuat.

**Penyebab:** Race condition antara icon creation dan data loading.

---

## 🔑 Fase 5: The Holy Grail - Connection Property!

Nemukan property ajaib:

```
{83DA6326-97A6-4088-9453-A1923F573B29} 15
```

Property ini:
- Return `True` = Device **BENAR-BENAR** connected
- Return `False` = Device disconnected (tapi mungkin masih punya cached battery)

**Test result:**
```
ERAZER X8: Connected=False, Battery=100% (cached!)
Play 2:    Connected=True,  Battery=50%  (actual!)
RDM:       Connected=False, Battery=None
```

**Akhirnya bisa bedakan device yang aktif vs yang cuma paired!**

---

## 📊 Teknologi yang Digunakan

| Tool | Fungsi |
|------|--------|
| **Python** | Main language, system tray |
| **pystray** | System tray icon |
| **Pillow** | Icon image generation |
| **PowerShell** | Windows API queries |
| **WMI/PnP** | Device detection |
| **C#** | Prototyping (discarded) |
| **PyInstaller** | Build ke EXE |

---

## 📈 Statistik Eksperimen

| Metric | Value |
|--------|-------|
| **Total coding sessions** | ~15 |
| **Lines of code written** | ~2000+ |
| **Lines of code final** | ~150 |
| **PowerShell commands tested** | 50+ |
| **Properties discovered** | 100+ |
| **Bugs fixed** | 15+ |
| **Cups of coffee** | ☕☕☕☕☕ |

---

## 🎯 Lessons Learned

### 1. Windows Bluetooth API ≠ Simple
> "Baca baterai speaker Bluetooth di Windows itu TIDAK SEMUDAH yang dibayangkan."

Windows Settings pakai API internal yang tidak tersedia untuk third party apps.

### 2. Cached Data is Tricky
> "Device yang sudah mati masih punya battery data di Windows."

Harus cek connection status DULU baru baca battery.

### 3. Property Names are Cryptic
> `"{83DA6326-97A6-4088-9453-A1923F573B29} 15"`

Siapa yang sangka property dengan UUID acak itu adalah kunci untuk mengecek koneksi Bluetooth?

### 4. StackOverflow is Gold
> Tanpa thread di StackOverflow, kita tidak akan menemukan property battery yang benar.

### 5. Start Simple, Iterate Fast
> Mulai dari 400 baris code, akhirnya jadi 150 baris yang lebih powerful.

---

## 🔮 Future Improvements

- [ ] Multi-device display (tampilkan beberapa device sekaligus)
- [ ] Notification saat baterai rendah
- [ ] Auto-connect audio output
- [ ] Support lebih banyak device types
- [ ] Custom icon themes
- [ ] Export battery history

---

## 💬 Penutup

> "Yang kedengarannya sederhana - 'baca baterai speaker' - ternyata butuh 15+ iterasi debugging, 50+ PowerShell commands, dan discovery property Windows yang sangat cryptic."

Tapi pada akhirnya... **berhasil juga!** 🎉

Dan yang paling penting: **Play 2 sekarang tampil dengan battery 50% di taskbar!** 🔋

---

**Made with ❤️ by JARxAI**  
**June 2026**
