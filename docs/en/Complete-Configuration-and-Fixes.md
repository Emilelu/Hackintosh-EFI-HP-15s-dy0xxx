> [简体中文](../完整配置清单与已修复项.md)

# Complete Configuration Checklist and Fixed Items

This document summarizes **all** configured/fixed items in the HP 15s-dy0xxx Hackintosh project that are easy to overlook, as a supplement to the [Troubleshooting and Fix Notes](./Troubleshooting-and-Fixes.md).

---

## A. Bluetooth System Layer (NVRAM, Most Critical)

| Item | Value | Notes |
|---|---|---|
| `bluetoothInternalControllerInfo` | `00` × 14 bytes (i.e. 28 hex `0` characters) | **Must be exactly 14 bytes**. Writing 13 bytes makes `bluetoothd` crash-loop with `EXC_GUARD namespc 18 reason 2`; writing 29 bytes (parsing 28 hex characters as ASCII) likewise reports `CFDataGetLength(internalControllerInfoRef) != 14` |
| `bluetoothExternalDongleFailed` | `00` | Required item for `bluetoothd` to accept third-party Bluetooth controllers |
| `csr-active-config` | `030a0000` | Partially disables SIP; required for kext injection/loading |
| `run-efi-updater` | `No` | Prevents firmware updates from writing to NVRAM |
| `SystemAudioVolume` | `46` | Boot-up volume level |

boot-args related:

| Argument | Purpose |
|---|---|
| `-amfipassbeta` | AMFIPass, bypasses AMFI validation to allow injecting unsigned kexts |
| `-btlfxa2dpcheck` | BlueToolFixup 2.7.3's AirPods A2DP "connected but no sound" fix |

### Why 14 Bytes Is So Critical

`bluetoothd` reads `bluetoothInternalControllerInfo` at startup to construct the internal controller object.
The length must be `14` (6-byte address + 4-byte class + 4 bytes of other data); otherwise it crashes directly with `EXC_GUARD`,
which manifests as the Bluetooth icon repeatedly flickering and `bluetoothd` restarting over and over.

> Write method: `NVRAM → Add → 7C436110-AB2A-4BBB-A880-FE41995C9F82 → bluetoothInternalControllerInfo`
> Use the `<data>` type, with 14 `0x00` bytes.

---

## B. Backlight and Boot Black Screen

**Symptom**: after boot, the internal display stays black for several minutes before lighting up (the system has actually finished booting; only the backlight is off).

**Fix combination (DeviceProperties + boot-arg, all required)**:

```
applbkl                        = 03000000
enable-backlight-registers-fix = 01000000
enable-hdmi-dividers-fix       = 01000000
boot-args += -igfxblr
```

| Item | Purpose |
|---|---|
| `applbkl` = `0x03000000` | Declares the backlight control method as PCH PWM |
| `enable-backlight-registers-fix` | Fixes the backlight register write path (WEG's `-igfxblr` equivalent switch) |
| `-igfxblr` | Enables the fix above |
| `enable-hdmi-dividers-fix` | Fixes the divider deadlock at high HDMI pixel clocks |

---

## C. Graphics and Display

| Item | Value | Notes |
|---|---|---|
| `SSDT-Disable_GPU_RP05.aml` | ACPI | Disables the MX250 dGPU to avoid fighting for display resources / heat |
| `AAPL,ig-platform-id` | `0900A53E` (0x3EA50009) | Whiskey Lake three-connector platform |
| `device-id` | `A53E0000` (0x3EA5) | Lets the Apple driver match the UHD 620 |
| `framebuffer-con0-alldata` | Complete 36-byte table | **Prefer the complete table** to avoid `conX-type/busid/flags` scattered properties being affected by WEG's override order |
| `enable-hdmi-dividers-fix` | `01000000` | HDMI dividers |
| `-igfxmpc` | **Bare flag**, no value | The value in `-igfxmpc=600` is not parsed; WEG only does `checkKernelArgument("-igfxmpc")` and is internally fixed at 675MHz |
| `-igfxhdmidivs` | boot-arg | Same as `enable-hdmi-dividers-fix` |

### Items Never to Use

| Item | Consequence |
|---|---|
| `disable-agdc` | Boot becomes 20 minutes (`IOFBProbeOptions` changes from 1 to 1025) |
| `CNConnectorAlwaysConnected` (connector flags bit `0x0008`) | Forces `port-number = 0`, conflicting with the internal LVDS port 0 → black screen, `IODisplayConnect = 0` |
| `enable-hdmi20` / `-cdfon` | Not implemented on macOS 11+, no effect on Tahoe |

---

## D. OpenCore Environment

### ACPI (9 SSDTs)

| SSDT | Purpose |
|---|---|
| `SSDT-ALS0.aml` | Ambient light sensor placeholder |
| `SSDT-SBUS.aml` | SMBus compatibility |
| `SSDT-Disable_GPU_RP05.aml` | Disables the MX250 |
| `SSDT-EC.aml` | Embedded controller (avoids AppleACPIEC conflicts) |
| `SSDT-MCHC.aml` | Memory controller |
| `SSDT-PMC.aml` | Power management controller (native NVRAM support) |
| `SSDT-PLUG.aml` | CPU power management (XCPM) |
| `SSDT-PNLF.aml` | Backlight (works with `applbkl`) |
| `SSDT-RTCAWAC.aml` | RTC fix |

### Booter → Quirks

```
RequestBootVarRouting   = True
AvoidRuntimeDefrag      = True
EnableWriteUnprotector  = True
ProtectMemoryRegions    = False
SetupVirtualMap         = True
SyncRuntimePermissions  = False
```

### Misc → Security

```
ScanPolicy           = 769
SecureBootModel      = Disabled
Vault                = Optional
BlacklistAppleUpdate = True
DmgLoading           = Signed
AllowSetDefault      = True
ExposeSensitiveData  = 6
```

### Misc → Boot

```
ShowPicker        = True
HideAuxiliary     = False
Timeout           = 5
LauncherOption    = Full
LauncherPath      = Default
PickerMode        = External
PickerVariant     = Auto
PickerAttributes  = 17
HibernateMode     = None
```

---

## E. Dual Boot (Ubuntu)

| Item | Notes |
|---|---|
| APFS container resize | 397.3 GB → 287 GB, freeing about 100 GB |
| Ubuntu 24.04 | Installed to the `UBUNTUIMG` partition (8 GB FAT32, `disk0s6`, UUID `792E161C-CD0A-3F3F-943E-041EA60558EE`) |
| `EFI/BOOT/bootx64.efi` | Was originally renamed to `.disabled` and needs to be restored |
| OpenCore boot entry | Ubuntu entry added with `05-opencore-ubuntu-entry.sh` |

---

## F. Incident Lessons and Diagnostic Methodology

### F1. `plutil -data` Command-Line Base64 Causes Byte Misalignment

**Measured multiple times**: writing `00000800` / `01C7` as `03000800` / `0107`, causing boot hang or black screen.

**Correct approach**:
- Read/write directly with Python `plistlib`, or
- `plutil -replace <key> -xml '<data>BASE64</data>'`
- **After every change, decode and print `idx / bus / pipe / type / flags` field by field for verification**

### F2. `killall -HUP WindowServer` Breaks Display Enumeration

Symptom: `IODisplayConnect = 0`, `system_profiler` shows 0 displays, and only a reboot recovers.
**Do not use signals to trigger display re-probing**.

### F3. boot-args Was Once Written Empty

Cause: the script read a stale `/tmp/cfg.xml`.
**Every configuration change must re-read the current value from the target file**.

### F4. `test -s` Misjudges sysfs EDID Files in Ubuntu Live

The `stat` size of `/sys/class/drm/*/edid` shows 0, but the file is actually readable as 128/256 bytes.
Use `cat <file> | wc -c` to determine the size.

### F5. Lilu Plugin Logs Invisible on macOS 26 (Important Correction)

**Root cause (two compounding factors)**:

1. **Both Lilu and the plugin must be compiled as DEBUG builds for `DBGLOG` to be retained.**
   The official release binary of `Lilu.kext 1.7.3` is about 531 KB, and **the number of `DBG` strings in the binary is 0**
   → This is a RELEASE build, and `DBGLOG` has been compiled out entirely.
   Therefore **`-liludbg` / `-liludbgall` have no effect on the RELEASE build** (there is no code to output).
2. **Early-boot kext `IOLog` output is flushed out by the kernel log ring buffer.**
   Measured: 365 later AirPortRTW log entries remain, while early plugin `IOLog` entries do not appear.

**Correct approach**:
- To see Lilu/WEG debug logs: **Lilu DEBUG build + plugin DEBUG build + `-liludbg` (or `-liludbgall`)**
- Or (adopted by this project): **publish state as IORegistry properties**, independent of build type and log buffer,
  read with `ioreg -lw0 | grep -oE '"IBTP[A-Za-z]+" = <[^>]*>'`

### F6. WEG DEBUG's `-igfxi2cdbg` Must Not Be Used with `disable-agdc`

`kern_igfx_debug.cpp` L333-334 contains `PANIC("igfx", "igfxagdc=0 is not compatible with framebuffer debugging")`.
When `-igfxfbdbg` is needed, `disable-agdc` must be removed first.

### F7. Boot Problems Require `-v`

When the screen is black, it is impossible to tell where the boot is stuck. `debug=0x100 keepsyms=1 -v` is essential.

### F8. Recovery Methods

```
Cmd+S single-user mode
mount -uw /
mkdir -p /Volumes/ESP
mount_msdos /dev/disk0s1 /Volumes/ESP
cd /Volumes/ESP/EFI/OC
cp config.plist.bak-xxxx config.plist
reboot
```

---

## G. Wi-Fi

| Item | Notes |
|---|---|
| `AirPortRTW 1.2.0` | Supports **WPA3-Personal** |
| `RealtekAirPortRTWFix` | Restores the coexistence handshake with Bluetooth |
| Driver conflict | **Must not load** `AirPort_RTW88.kext` and `AirPortRTW.kext` at the same time |
| Bluetooth firmware | `RealtekBluetoothFirmware` **must use the official release binary**; a self-compiled build completely breaks Bluetooth RX (zero advertising, 0 results for classic inquiries) |

---

## H. Other Ruled-Out Traps

| Item | Conclusion |
|---|---|
| `igfxonln` / `force-online` | Only changes `getDisplayStatus`, not `GetOnlineInfo`/EDID; ineffective on its own |
| `AAPL0X,override-no-connect` | Only supports **128 bytes** (only the base block is used; extension blocks are ignored) |
| `STATUS 4803` | Also appears in successful Bluetooth pairing logs; **it is not a fault** |
| `rtl8821c_config.bin` | Not an LE feature switch (just one register write, offset 0xD9 = 0x0F) |
| `FBBoostPixelFrequencyLimit (0x100)` | Only effective on Haswell; ineffective on CFL |
| `-igfxmpc=600` | The value is not parsed; only the bare flag is recognized |
