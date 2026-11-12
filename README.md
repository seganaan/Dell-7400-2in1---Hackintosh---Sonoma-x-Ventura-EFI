# Dell Latitude 7400 2-in-1 — Hackintosh OpenCore EFI (Ventura & Sonoma)

An OpenCore EFI built from scratch for the **Dell Latitude 7400 2-in-1** (Intel Core i7, Intel UHD 620 graphics). Confirmed working on **macOS Ventura (13.x)** and **macOS Sonoma (14.x)**.

> ⚠️ **Before making this repo public:** `config.plist` contains `PlatformInfo → Generic` values (serial number, MLB, ROM, SmUUID). Generate your own with GenSMBIOS / OCAT and never publish the ones you actually use.

---

## Repository layout

| Path | Purpose |
|------|---------|
| `BOOT/` | `BOOTx64.efi` — the OpenCore bootstrap loaded by the firmware |
| `OC/` | OpenCore itself: `config.plist`, ACPI SSDTs, Drivers, Kexts, OpenCore.efi |
| `Dell/logs/` | Debug logs captured while building/verifying the EFI |
| `.DS_Store`, `.gitattributes` | Housekeeping files |

---

## System summary (what `config.plist` is set up for)

| Item | Value |
|------|-------|
| Emulated Mac | `MacBookPro15,2` (Coffee/Whiskey Lake-class laptop SMBIOS) |
| iGPU | Intel UHD 620, spoofed as UHD 630 mobile (`AAPL,ig-platform-id` = `0x3E9B0000`, `device-id` = `0x3E9B`) |
| Internal panel link rate | `dpcd-max-link-rate` = `0x0A` (2.7 Gbps / HBR) |
| Boot picker | OpenCore built-in picker, shown with a 5 s timeout (no OpenCanopy) |
| SIP | Enabled (`csr-active-config` = `00000000`) |
| Secure Boot Model | Disabled |
| OpenCore profile | Debug-style (`AppleDebug`, `ApplePanic`, `Target = 67`) — logs are written for troubleshooting |

---

## Working components

Everything below is enabled in the config; the kext responsible is in brackets.

| Component | Status | How it's handled |
|-----------|:------:|------------------|
| **Graphics acceleration** (UHD 620) | ✅ | WhateverGreen + device-id / platform-id spoof |
| **Internal display + brightness control** | ✅ | `SSDT-PNLF` + WhateverGreen backlight fix (`-igfxblt`) |
| **Wi-Fi** | ✅ | `AirportItlwm` (Intel Wi-Fi, native Wi-Fi menu) |
| **Bluetooth** | ✅ | `IntelBluetoothFirmware` + `IntelBTPatcher` + `BlueToolFixup` (`-btlfxallowanyaddr`) |
| **Audio** (speakers / headphones / mic) | ✅ | `AppleALC` with `alcid=77` |
| **Keyboard** | ✅ | `VoodooPS2Controller` + `VoodooPS2Keyboard` |
| **Trackpad / touch input** | ✅ | `VoodooI2C`, `VoodooI2CHID`, `VoodooGPIO`, `VoodooI2CServices`, `VoodooInput` (I²C-HID) |
| **PS/2 mouse & trackpad fallback** | ✅ | `VoodooPS2Mouse`, `VoodooPS2Trackpad` |
| **Battery status** | ✅ | `VirtualSMC` + `SMCBatteryManager` |
| **CPU sensors / temperatures** | ✅ | `SMCProcessor` |
| **CPU power management** (turbo, speed stepping) | ✅ | `SSDT-PLUG-DRTNIA` + `CPUFriend` + `CPUFriendDataProvider` |
| **NVMe SSD power / performance** | ✅ | `NVMeFix` |
| **Sleep / wake** | ✅ | `SSDT-GPRW` (+ `GPRW → XPRW` ACPI patch), `HibernationFixup`, `SSDT-EC-USBX-LAPTOP` (USB power properties) |
| **Embedded controller** | ✅ | `SSDT-EC-USBX-LAPTOP` |

> Not part of this EFI: USB port map kext, Thunderbolt, fingerprint reader, SD card reader, webcam-specific fixes. Add them yourself if needed.

---

## What each section of `config.plist` does

**ACPI**
- `SSDT-EC-USBX-LAPTOP` – fake embedded controller (macOS requires one) and USB power properties.
- `SSDT-PLUG-DRTNIA` – enables native Intel CPU power management (XCPM).
- `SSDT-PNLF` – creates the backlight device so brightness keys/slider work.
- `SSDT-GPRW` + rename patch `GPRW → XPRW` – prevents instant wake / wake-from-USB problems.
- `ResetLogoStatus` quirk is on; the other ACPI quirks and the sample patches are off.

**Booter**
- Standard Coffee/Whiskey Lake laptop quirk set: `AvoidRuntimeDefrag`, `EnableSafeModeSlide`, `ProvideCustomSlide`, `RebuildAppleMemoryMap`, `SetupVirtualMap`, `SyncRuntimePermissions`, `FixupAppleEfiImages`.

**DeviceProperties** (iGPU at `PciRoot(0x0)/Pci(0x2,0x0)`)
- `AAPL,ig-platform-id`, `device-id` spoof and `dpcd-max-link-rate` for the built-in eDP panel.

**Kernel**
- Kext load order: Lilu → VirtualSMC → VoodooPS2 → WhateverGreen → VoodooI2C stack → AppleALC → SMC plugins → AirportItlwm → NVMeFix → Bluetooth stack → CPUFriend → HibernationFixup.
- Quirks enabled: `AppleXcpmCfgLock` (CFG-Lock is *not* disabled in BIOS, so this works around it), `DisableIoMapper`, `DisableLinkeditJettison`, `PanicNoKextDump`, `PowerTimeoutKernelPanic`.
- A `IOSkywalkFamily` block entry and a few other sample patches exist but are **disabled**.

**Misc**
- Picker is visible (`ShowPicker`), auxiliary entries hidden, `HibernateMode = Auto`, `AllowSetDefault` on, `DmgLoading = Signed`, `Vault = Optional`, `BlacklistAppleUpdate` on, `SecureBootModel = Disabled`.

**NVRAM**
- Contains the `boot-args` described below, `run-efi-updater = No`, SIP enabled, and the standard OpenCore keyboard-language/audio-volume variables.

**PlatformInfo**
- Automatic SMBIOS generation as `MacBookPro15,2`, `UpdateSMBIOSMode = Create`, `SpoofVendor` on.

**UEFI**
- Drivers: `HFSPlus.efi`, `OpenRuntime.efi`, `ResetNvramEntry.efi`.
- `ConnectDrivers`, `ReleaseUsbOwnership`, `RequestBootVarRouting`, `EnableVectorAcceleration` on, keyboard support on.
- One `ReservedMemory` region (4 KB, `RuntimeCode`) is enabled as a black-screen-after-hibernation workaround.

---

## Boot arguments

```
keepsyms=1 debug=0x100 igfxonln=1 -igfxmlr -igfxblt -btlfxallowanyaddr alcid=77
```

| Argument | Meaning |
|----------|---------|
| `keepsyms=1` | Keep kernel symbols in panic logs (easier debugging) |
| `debug=0x100` | Don't auto-reboot on a kernel panic, so you can read it |
| `igfxonln=1` | WhateverGreen: force all displays to report as online |
| `-igfxmlr` | WhateverGreen: DPCD maximum link rate fix (pairs with `dpcd-max-link-rate`) |
| `-igfxblt` | WhateverGreen: alternative backlight registers fix (see troubleshooting) |
| `-btlfxallowanyaddr` | BlueToolFixup: allow Bluetooth to work with any controller address |
| `alcid=77` | AppleALC audio layout ID 77 |

---

## Troubleshooting

### Screen stays dark for ~3 minutes after the Apple logo / boot
On Coffee/Whiskey Lake laptops running **macOS 13.4 or newer**, the classic WhateverGreen backlight fix (`-igfxblr`) can leave the panel dark for around three minutes after the Apple logo before the backlight finally comes on.

**Fix:** in `OC/config.plist` → `NVRAM → Add → 7C436110-AB2A-4BBB-A880-FE41995C9F82 → boot-args`, replace `-igfxblr` with **`-igfxblt`**.

```diff
- keepsyms=1 debug=0x100 igfxonln=1 -igfxmlr -igfxblr -btlfxallowanyaddr alcid=77
+ keepsyms=1 debug=0x100 igfxonln=1 -igfxmlr -igfxblt -btlfxallowanyaddr alcid=77
```

Notes:
- `-igfxblt` is the *alternative* backlight registers fix introduced in WhateverGreen **1.6.5**; make sure your WhateverGreen is that version or newer.
- Use **one or the other**, not both, and don't combine with other backlight experiments.
- This repo already ships with `-igfxblt`.
- After editing, reset NVRAM from the OpenCore picker (`ResetNvramEntry`) so the new `boot-args` take effect.

### Other quick checks
- **No Wi-Fi after updating macOS:** update `AirportItlwm` to the build matching your macOS version.
- **Bluetooth missing:** make sure `BlueToolFixup`, `IntelBluetoothFirmware` and `IntelBTPatcher` are all present and enabled, and that you're on Ventura or newer.
- **Boot loop / panic:** the config is debug-flavoured; check `Dell/logs/` and the OpenCore log written to the EFI partition.

---

## Credits

[Acidanthera](https://github.com/acidanthera) (OpenCore, Lilu, WhateverGreen, VirtualSMC, AppleALC, NVMeFix, CPUFriend, HibernationFixup, BlueToolFixup), [OpenIntelWireless](https://github.com/OpenIntelWireless) (AirportItlwm, IntelBluetoothFirmware, IntelBTPatcher), the VoodooI2C and VoodooPS2 maintainers, and the Dortania OpenCore guide.

## Disclaimer

Provided as-is for the specific hardware above. Back up your own EFI before using it, and use your own SMBIOS data.
