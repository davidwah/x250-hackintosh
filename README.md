# EFI OpenCore — ThinkPad X250 · macOS Monterey 12.7.6

Backup EFI final yang sudah teruji jalan di ThinkPad X250 pada **27 Oktober 2026**.

---

## Hardware target

| Komponen | Detail |
| --- | --- |
| Laptop | Lenovo ThinkPad X250 |
| CPU | Intel Core i5-5300U (Broadwell) |
| GPU | Intel HD Graphics 5500 |
| RAM | 16 GB (1 keping) |
| SSD | 2,5" SATA |
| WiFi + Bluetooth | Intel AX200 |
| LAN | Intel I218-LM |
| Audio | Realtek ALC3232 (ALC292) |
| macOS | Monterey 12.7.6 (21H1320) |
| SMBIOS | MacBookPro12,1 |

## Status fitur

| Fitur | Status |
| --- | --- |
| Grafis (akselerasi penuh) | ✅ |
| WiFi | ✅ |
| Bluetooth | ✅ |
| Speaker, headphone, mikrofon | ✅ |
| Audio HDMI | ✅ |
| Brightness (slider + Fn+F5/F6) | ✅ |
| Volume (Fn) | ✅ |
| Baterai | ✅ |
| Keyboard, TrackPoint, trackpad | ✅ |
| LAN | ✅ |
| Webcam | ✅ |
| Sleep / wake | ✅ (dengan pengaturan `pmset` di bawah) |
| App Store, iCloud | ✅ |
| AirDrop / Handoff | ❌ butuh kartu Broadcom |
| Fingerprint, SD card reader | ❌ tidak didukung |

---

## Struktur folder

```
EFI/
├── BOOT/
│   ├── BOOTx64.efi            # bootstrap OpenCore
│   └── .contentVisibility     # menyembunyikan entri OpenCore dari menu boot
└── OC/
    ├── OpenCore.efi           # OpenCore 1.0.8 RELEASE
    ├── config.plist           # konfigurasi utama
    ├── ACPI/                  # SSDT (lihat tabel di bawah)
    ├── Drivers/               # driver UEFI
    ├── Kexts/                 # driver macOS
    ├── Resources/             # aset tampilan menu (tidak dipakai, PickerMode Builtin)
    └── Tools/
        └── OpenShell.efi      # UEFI Shell
```

## ACPI (`EFI/OC/ACPI`)

| File | Fungsi |
| --- | --- |
| `SSDT-PLUG-DRTNIA.aml` | Mengaktifkan power management CPU (XCPM) |
| `SSDT-EC-USBX-LAPTOP.aml` | Embedded Controller palsu + properti daya USB |
| `SSDT-PNLF.aml` | Kontrol backlight layar. Sudah di-patch dari `GFX0` ke `VID_`, karena kartu grafis ThinkPad bernama `VID` di DSDT |
| `SSDT-EHCI-OFF.aml` | Mematikan controller USB 2.0 lama (EHCI) di macOS, sehingga semua port pindah ke XHCI (dibutuhkan Bluetooth dan webcam) |
| `SSDT-IRQ.aml` | IRQ fix: HPET, RTC, timer, dan PIC baru dengan IRQ 0 dan 8. Tanpa ini, audio internal tidak terdeteksi |
| `SSDT-BKEY.aml` | Mengalihkan tombol Fn+F5/F6 ke keyboard PS/2 sebagai tombol brightness |

## ACPI Patch (di `config.plist`, berlaku hanya untuk DSDT)

| Patch | Find → Replace | Fungsi |
| --- | --- | --- |
| HPET `_STA` | `If (\WNTF && !\WXPF)` → `If (_OSI ("Darwin"))` | Mematikan HPET bawaan di macOS (pasangan `SSDT-IRQ`) |
| `_Q14` → `XQ14` | `5F513134` → `58513134` | Tombol brightness naik (pasangan `SSDT-BKEY`) |
| `_Q15` → `XQ15` | `5F513135` → `58513135` | Tombol brightness turun (pasangan `SSDT-BKEY`) |

## Drivers (`EFI/OC/Drivers`)

| File | Fungsi |
| --- | --- |
| `OpenRuntime.efi` | Wajib untuk OpenCore (memory map, NVRAM) |
| `HfsPlus.efi` | Membaca partisi HFS+ (installer / recovery). Dipakai sebagai pengganti `OpenHfsPlus` |
| `ResetNvramEntry.efi` | Entri **Reset NVRAM** di menu OpenCore |

## Kexts (`EFI/OC/Kexts`)

| Kext | Versi | Fungsi |
| --- | --- | --- |
| Lilu | 1.7.2 | Dasar untuk kext lain |
| VirtualSMC | 1.3.5 | Emulasi SMC |
| SMCBatteryManager | 1.3.5 | Status baterai |
| SMCProcessor | 1.3.5 | Sensor suhu CPU |
| ECEnabler | 1.0.6 | Baca baterai ThinkPad tanpa patch DSDT |
| WhateverGreen | 1.7.1 | Grafis HD 5500 |
| AppleALC | 1.9.8 | Audio (layout 15) |
| IntelMausi | 1.0.8 | LAN Intel I218-LM |
| VoodooPS2Controller | 2.3.4 | Keyboard, TrackPoint, trackpad |
| BrightnessKeys | 1.0.3 | Tombol brightness |
| AirportItlwm | 2.3.0 (build Monterey) | WiFi AX200. **Hanya untuk Monterey** |
| BlueToolFixup | 2.7.2 | Perbaikan Bluetooth Monterey |
| IntelBluetoothFirmware | 2.4.0 | Firmware Bluetooth Intel |
| IntelBTPatcher | 2.4.0 | Patch Bluetooth Intel |
| X250-USBMap | 1.0 (custom) | USB map 12 port (`HSP0–7`, `SSP0–3`). Membuka port internal yang disembunyikan BIOS (Bluetooth, webcam) |

---

## Pengaturan penting di `config.plist`

| Bagian | Nilai | Catatan |
| --- | --- | --- |
| `DeviceProperties` → iGPU `PciRoot(0x0)/Pci(0x2,0x0)` | `AAPL,ig-platform-id = 06002616`, `framebuffer-stolenmem = 00003001`, `framebuffer-fbmem = 00009000` | Patch framebuffer karena DVMT X250 terkunci 32 MB |
| `DeviceProperties` → audio `PciRoot(0x0)/Pci(0x1b,0x0)` | `layout-id = 1C000000` | Ditimpa oleh boot-arg `alcid=15` |
| `Kernel → Quirks` | `AppleCpuPmCfgLock`, `AppleXcpmCfgLock`, `DisableIoMapper`, `PowerTimeoutKernelPanic` = `true` | CFG Lock X250 tidak bisa dibuka dari BIOS |
| `Misc → Boot` | `ShowPicker = false`, `HideAuxiliary = false`, `Timeout = 5` | Tahan **Esc** saat boot untuk menampilkan menu OpenCore |
| `Misc → Security` | `SecureBootModel = Default` | Wajib `Default` agar AirportItlwm termuat |
| `Misc → Debug` | `Target = 3`, `SysReport = false` | Logging minimal |
| `NVRAM → boot-args` | `keepsyms=1 alcid=15 alcdelay=1000 igfxonln=1` | `igfxonln=1` mencegah freeze saat colok/cabut HDMI |
| `NVRAM → csr-active-config` | `00000000` | SIP aktif |
| `UEFI → Input` | `KeySupport = false` | Dengan `true`, tombol Enter tidak terbaca di menu OpenCore |
| `PlatformInfo` | `MacBookPro12,1` | Serial, MLB, UUID unik. ROM = MAC LAN `en0` |

## Setting BIOS yang dipakai

- Security → Secure Boot: **Disabled**
- Security → Virtualization → Intel VT-d: **Disabled**
- Security → Anti-Theft → Computrace: **Disabled**
- Config → Intel AMT: **Disabled**
- Config → Serial ATA: **AHCI**
- Config → USB → USB 3.0 Mode: **Enabled**
- Startup → UEFI/Legacy Boot: **UEFI Only**, CSM **No**

## Pengaturan macOS setelah install

```bash
sudo pmset -a hibernatemode 0 standby 0 autopoweroff 0 proximitywake 0 tcpkeepalive 0
```

---

## Cara restore

### A. EFI di SSD rusak atau hilang (boot dari flashdisk)

1. Format flashdisk: **GPT + FAT32**.
2. Ekstrak zip backup, lalu salin folder `EFI` ke root flashdisk.
3. Colok ke X250, tekan **F12**, lalu pilih flashdisk.
4. Setelah masuk macOS, salin EFI ke SSD:
   ```bash
   sudo mkdir -p /Volumes/SSDEFI
   sudo diskutil mount -mountPoint /Volumes/SSDEFI disk0s1
   sudo cp -R /Volumes/<flashdisk>/EFI /Volumes/SSDEFI/
   ```

### B. Pindah ke SSD baru

1. Clone macOS ke SSD baru (misalnya pakai Carbon Copy Cloner dengan Legacy Bootable Copy Assistant).
2. Cek identifier partisi EFI SSD baru dengan `diskutil list`, mount, lalu salin folder `EFI` dari backup ini.

## Catatan saat upgrade macOS

- **Update minor Monterey (12.7.x)**: aman. Update OpenCore dan kext dulu sebelum update macOS.
- **Ventura atau versi lebih baru**: butuh SMBIOS baru, AirportItlwm sesuai versi, dan **OpenCore Legacy Patcher** (root patch grafis HD 5500). Jangan pakai EFI ini apa adanya.

## Referensi

- [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/)
- [Dortania OpenCore Post-Install](https://dortania.github.io/OpenCore-Post-Install/)
- [zamkara/Lenovo-Thinkpad-X250-Hackintosh](https://github.com/zamkara/Lenovo-Thinkpad-X250-Hackintosh), sumber IRQ fix, EHCI-off, dan patch tombol brightness
- [acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg)
- [OpenIntelWireless](https://github.com/OpenIntelWireless)
