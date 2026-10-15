# Hackintosh – Intel i7-14700K + NVIDIA RTX 4070 Ti

OpenCore EFI running **macOS 26 Tahoe** (also tested on macOS 15.8.1 Sequoia) on a Raptor Lake desktop
with an **RTX 4070 Ti**, using the community **NullMoth** Metal driver.

> 🇻🇳 Bản tóm tắt tiếng Việt ở cuối trang: [Tiếng Việt](#tiếng-việt--tóm-tắt-nhanh)

---

## Hardware

| Part | Model |
|---|---|
| CPU | Intel Core i7-14700K (Raptor Lake, 8P + 12E) |
| Motherboard | ASUS TUF GAMING B760M-E D4 |
| GPU | NVIDIA GeForce RTX 4070 Ti 12 GB (AD104) |
| Ethernet | Realtek (RTL812xLucy.kext) |
| Display | 4K 144 Hz over DisplayPort |
| Dual boot | Windows on a separate NVMe (own EFI partition) |

## Software

| | |
|---|---|
| Bootloader | OpenCore 1.0.9 (DEBUG build) |
| SMBIOS | **MacPro7,1** (iMacPro1,1 also works on Sequoia, but cannot go to Tahoe) |
| NVIDIA driver | [NullMoth](https://github.com/nullmoth/nvidia-macos-driver) 2.0.0, installed with the **1401** app |
| SIP | `csr-active-config = 0x0A43` (required by NullMoth) |
| Secure Boot | `SecureBootModel = Disabled` |

### Kexts
Lilu, AMFIPass, AppleALC, CpuTopologyRebuild, NVMeFix, RTL812xLucy, RestrictEvents, VirtualSMC,
SMCProcessor, SMCSuperIO, USBToolBox + UTBDefault, WhateverGreen, XHCI-unsupported.

### ACPI
SSDT-EC, SSDT-PLUG-ALT, SSDT-RTCAWAC, SSDT-SBUS, SSDT-USB-Reset, SSDT-USBX.

### Notable config settings
| Setting | Value | Why |
|---|---|---|
| `Kernel > Emulate > Cpuid1Data` | `55060A00…` (Comet Lake spoof) | Raptor Lake is not natively known by macOS |
| `Kernel > Quirks > ProvideCurrentCpuInfo` | `true` | Hybrid P/E core CPU |
| boot-arg `ctrsmt=full` | with CpuTopologyRebuild | P/E core topology |
| `amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0` | boot-args | Needed for the NullMoth driver |
| `nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth` | boot-args | NullMoth driver options |
| `revpatch=sbvmm,cpuname` | RestrictEvents | Lets Software Update offer macOS updates |
| `Booter > Quirks > ResizeAppleGpuBars` | `0` | Small BAR for macOS early boot |
| `UEFI > Quirks > ResizeGpuBars` | `-1` | Leave BAR to firmware |

---

## BIOS settings ⚠️ (read this first)

| Setting | Value | Notes |
|---|---|---|
| **Intel(R) Speed Shift Technology** | **Enabled** | **Mandatory for MacPro7,1.** If disabled → kernel panic or freeze (see Troubleshooting #1) |
| Above 4G Decoding | Enabled | |
| Resizable BAR | **Enabled** when the NullMoth driver is installed / **Disabled** for the macOS installer and for booting without the driver | NullMoth needs a different BAR setup for installer vs driver |
| CFG Lock | Disabled if available (otherwise keep `AppleXcpmCfgLock = true`) | |
| CSM | Disabled | |
| Secure Boot | Disabled | |
| Serial Port (Super I/O) | doesn't matter | was tested, not related to any issue |

ASUS path: `Advanced → CPU Configuration → CPU Power Management Control → Intel(R) Speed Shift Technology`.

---

## How to use this EFI

1. **Generate your own SMBIOS** for `MacPro7,1` with [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) and fill
   `PlatformInfo > Generic`: `SystemSerialNumber`, `MLB`, `SystemUUID`, and set `ROM` to your NIC MAC address.
   These fields are intentionally **empty** in this repo.
2. Your USB map (`UTBDefault.kext`) and SSDTs are specific to this board – remap for yours.
3. Install macOS with **Resizable BAR disabled**.
4. Install the NVIDIA driver with the **1401** app (from the NullMoth project). It adds its own OpenCore tool /
   boot entries (`1401: Start once without NVIDIA driver`, `1401: Remove NVIDIA driver`) – those files are **not**
   included here (NullMoth license: PolyForm Noncommercial).
5. Enable **Resizable BAR**, **Reset NVRAM** once, boot macOS.
6. Check: `sysctl hw.model` → `MacPro7,1`; the 1401 app shows the driver as **running**.

---

## Troubleshooting – what we hit and what actually fixed it

### 1. Switching SMBIOS iMacPro1,1 → MacPro7,1: hang at Apple logo / verbose, or kernel panic
**Symptoms**
- Verbose stops mid-line (e.g. `apfs_trim_to_prev_fsize:180: disk█`), keyboard LEDs don't come back after replug,
  no ping on LAN – a real hard hang. Recovery hangs too.
- Or a panic like:
  ```
  _xcpm_hwp_enable ← _xcpm_cpu_control
  com.apple.driver.X86PlatformShim : sendPStateStepperTableXCPM / initializePMInfo
  System model name: MacPro7,1 (Mac-27AD2F918AE68F61)
  ```

**Cause:** with MacPro7,1, macOS power management enables **HWP (Intel Speed Shift)**. If Speed Shift is disabled in
BIOS, the CPU rejects it → panic/hang. iMacPro1,1 does not hit this path, which is why the same EFI "works" with iMacPro1,1.

**Fix:** BIOS → **Intel Speed Shift Technology = Enabled**. Then Reset NVRAM. Nothing else in the EFI needs to change.

**Things we tried that were NOT the cause** (save yourself the time):
`agdpmod=pikera`, Resizable BAR on/off (alone), disabling the NVIDIA driver, booting Recovery,
disabling the serial port, disabling USB kexts / AppleALC / CpuTopologyRebuild, Reset NVRAM alone.

**Harmless verbose lines** (appear on working boots too – don't chase them):
`handle_revert_to_snapshot … com.apple.os.update-…`, `Volume MacOS is unmounting`, `AppleKeyStore operation failed`,
`apfs_keybag_init … err = 2`, `AMFI: [non-fatal] unable to accelerate context`,
`vm_shared_region_start_address() failed`, `IOUserDockChannelSerial waiting for server`.

### 2. Black screen / freeze when booting **without** the NVIDIA driver (installer, Recovery, "Start once without NVIDIA driver")
Disable **Resizable BAR** in BIOS. NullMoth's installer path needs the small BAR; the driver path wants the large BAR.

### 3. Feels like 60 Hz on a 144 Hz monitor
Cursor is smooth (hardware cursor) but window drag/scroll ~60 fps. This is the Metal → Vulkan translation layer of
the driver, not a setting. Check `system_profiler SPDisplaysDataType` → `UI Looks like: … @ 144.00Hz` to confirm the
panel is at 144 Hz. Log also shows on RTX 40:
`nvrm: copy-engine queue could not be built on this GPU (VkResult -13)`.

### 4. HiDPI scaling (125 % / 150 %)
With NullMoth 2.0.0 more modes are listed, but scaled modes like 2560×1440 don't apply on this card.
**1920×1080 HiDPI @ 144 Hz** works. BetterDisplay "flexible scaling" did not help with this driver.

### 5. Chrome / Edge / Electron apps (VS Code, Discord, Claude…) feel slow
NullMoth 2.0.0 makes them render on the CPU by default. Set `NVMTL_ANGLE_METAL=1` to try their GPU path.

### 6. Clock wrong after switching between Windows and macOS
macOS stores UTC in the RTC, Windows stores local time. In Windows (admin CMD):
```cmd
reg add "HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /t REG_DWORD /d 1 /f
```
Also check the macOS time zone (`System Settings → General → Date & Time`) – a fresh install may default to Cupertino.

### 7. Upgrading Sequoia → Tahoe
1. Back up (Time Machine).
2. Update the driver in 1401 to a version that supports macOS 26 (2.0.0+ ships the Tahoe accelerator).
3. Software Update → Tahoe. During the installer reboots use **`1401: Start once without NVIDIA driver`** and
   **Resizable BAR disabled**. Keep **Speed Shift enabled**.
4. After install: Resizable BAR on, reinstall the driver for Tahoe in 1401, restart.

### 8. Dual boot safety
Windows and macOS are on different disks, each with its own EFI partition. Removing macOS later does not break
Windows – just put **Windows Boot Manager** first in the BIOS boot order.

---

## Credits
Acidanthera (OpenCore, Lilu, WhateverGreen, AppleALC, VirtualSMC, RestrictEvents, NVMeFix), USBToolBox,
CpuTopologyRebuild, AMFIPass, RTL812xLucy, corpnewt (GenSMBIOS), Dortania guide,
**NullMoth** (NVIDIA Metal driver – not redistributed here).

---

## Tiếng Việt – tóm tắt nhanh

**Cấu hình:** i7-14700K · ASUS TUF B760M-E D4 · RTX 4070 Ti · SMBIOS **MacPro7,1** · driver NullMoth 2.0.0 · macOS Tahoe.

**BIOS bắt buộc:**
- **Intel Speed Shift Technology: Enabled.** Nếu tắt, dùng MacPro7,1 sẽ treo ở logo Apple hoặc kernel panic ở `X86PlatformShim` / `xcpm_hwp_enable`. iMacPro1,1 không bị lỗi này, nên dễ tưởng là do SMBIOS hoặc do driver.
- Above 4G Decoding: Enabled.
- Resizable BAR: **tắt** khi cài macOS hoặc khi boot không có driver NVIDIA. **Bật** khi đã cài driver.

**Trước khi dùng:**
1. Tự tạo Serial, MLB, UUID cho MacPro7,1 bằng GenSMBIOS. Trong repo các mục này đang để trống.
2. USB map và SSDT là cấu hình riêng của main này, cần map lại nếu bạn dùng main khác.
3. Cài driver bằng app 1401. File của NullMoth không được đưa vào repo do giấy phép.
4. Sau khi đổi SMBIOS hoặc thay đổi BIOS: **Reset NVRAM**.

**Đã thử nhưng KHÔNG phải nguyên nhân treo:** `agdpmod=pikera`, chỉ tắt hoặc bật ReBAR, tắt driver NVIDIA, tắt cổng Serial, tắt kext USB/âm thanh/CpuTopologyRebuild.

**Lỗi còn tồn tại:**
- Kéo và cuộn cửa sổ chỉ khoảng 60 fps, trong khi con trỏ chuột mượt 144 Hz. Nguyên nhân là lớp dịch của driver.
- Chưa dùng được các mức scale 2560×1440. Dùng tốt nhất là 1920×1080 HiDPI @ 144 Hz.

**Lệch giờ khi dual boot với Windows:** xem mục Troubleshooting #6 ở trên.
