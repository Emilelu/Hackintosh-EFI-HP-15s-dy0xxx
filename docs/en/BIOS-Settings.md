> [简体中文](../BIOS-设置.md)

# BIOS Settings Recommendations: HP 15s-dy0xxx

> Back to [main README](../../README.en.md) | Applies to: HP 15s-dy0xxx / i7-8565U / macOS Tahoe 26.7.2

This document provides the BIOS settings baseline required to install and run macOS Tahoe on this model. Option names exposed by different batches and BIOS versions of the HP 15s series may differ; **the table below is organized by function/meaning, and not every item is guaranteed to be visible in your BIOS**.

> ⚠️ Before modifying the BIOS, record the original values and prepare a Windows / Linux recovery drive. Do not flash modified BIOS images of unknown origin, and do not copy `setup_var` offsets from other models.

---

## 1. Recommended Settings Summary

| Setting (common name) | Recommended value | Necessity | Notes |
|:---|:---|:---:|:---|
| Secure Boot | **Disabled** | Required | OpenCore and most third-party kexts are unsigned; booting fails when enabled |
| TPM Device / PTT | **Hidden / Disabled** | Recommended | macOS does not need TPM; if dual-boot Windows 11 relies on TPM, it may remain Enabled and macOS generally still boots, but it is recommended to disable it during installation |
| Intel SGX | **Disabled** | Recommended | macOS has no SGX driver; disabling reduces unnecessary ACPI / resource exposure |
| Intel Virtualization Technology (VT-x) | **Enabled** | Recommended | Improves VM and some kernel feature compatibility; may be left enabled even without VMs |
| VT-d | **Disabled** | Strongly recommended | macOS has limited IOMMU support; if the BIOS lacks this option, enable `DisableIoMapper` in OpenCore |
| CFG Lock | **Disabled** | Strongly recommended | Avoids kernel failures when writing MSR 0xE2; if hidden, use `AppleXcpmCfgLock` |
| DVMT Pre-Allocated | **64MB** | Strongly recommended | Baseline stolen memory for the UHD 620 framebuffer / HDMI; below 32MB is prone to garbled screens, black screens, or HDMI not lighting up |
| DVMT Total Gfx Mem | **256MB / Max** | Recommended | If the BIOS has this item, gives the iGPU a larger dynamic VRAM limit |
| Graphics / Switchable Graphics | **Keep factory default (Hybrid / Switchable)** | Recommended | Do not force "Discrete Only", otherwise the iGPU is disabled and macOS cannot drive it |
| Fast Boot | **Disabled** | Required | Otherwise USB / OpenCore boot entries may be skipped |
| USB Boot | **Enabled** | Required | Allows booting from the installation USB drive |
| Boot Mode / Legacy Support | **UEFI Only / Legacy Disabled** | Required | macOS only supports UEFI boot |
| CSM (Compatibility Support Module) | **Disabled** | Required | CSM introduces a Legacy boot path that may prevent OpenCore from taking over boot; set it together with UEFI Only |
| SATA Controller Mode | **AHCI** | Recommended | If the machine has a SATA disk / optical bay, AHCI is the mode macOS can recognize; NVMe is unaffected |
| Above 4G Decoding | **Enabled** (if visible) | Optional | Improves large-BAR device mapping; if boot problems appear after enabling, restore Disabled and keep `ResizeAppleGpuBars=-1` |
| Wake on LAN | **Disabled** | Optional | Prevents network wake during sleep; not critical for installation |
| BIOS Password | Record it; do not forget it | Recommended | Some HP BIOSes prompt for the password during BIOS boot after it is set; forgetting it means you cannot change settings |

---

## 2. Key Items Explained

### 2.1 Secure Boot

- Set to **Disabled**. OpenCore, WhateverGreen, AirPortRTW, RealtekBTPatcher, etc. are not Apple-signed components.
- If the BIOS has "Secure Boot Mode", choose **Setup Mode / Disabled**, not Standard / Custom.
- Do not re-enable it after installation; if Windows dual boot needs Secure Boot, use a separate boot entry and do not use OpenCore's Secure Boot chain.

### 2.2 CFG Lock

CFG Lock is the Intel platform's write protection for MSR `0xE2` (IA32_PERF_CTL / C-state related). macOS's XCPM power management needs to write this MSR; when it is not disabled, common symptoms are:

- boot hangs or panics;
- CPU power management does not work and frequencies are abnormal;
- sleep/wake fails.

Prefer to disable it in the BIOS. If the BIOS does not expose this option:

1. **Do not** directly use `setup_var 0x?? 0x0` offsets found online for the same model; offsets differ between BIOS versions, and a wrong write may brick the machine.
2. In OpenCore `Kernel → Quirks`, enable `AppleXcpmCfgLock` (evaluate `AppleCpuPmCfgLock` as well if necessary).
3. Use `ocvalidate` to confirm the quirk spelling matches the OpenCore version.

### 2.3 VT-d and `DisableIoMapper`

- Prefer to **Disable VT-d** in the BIOS.
- If the BIOS has no VT-d option, or VT-d must be retained for a Windows VM, enable in OpenCore:
  - `Kernel → Quirks → DisableIoMapper = true`
- `DisableIoMapper` makes macOS not use IOMMU mappings; this is not the same as disabling hardware virtualization, and VT-x may remain enabled.
- Do not enable both `DisableIoMapper` and `DisableIoMapperMapping` unless there is a clear troubleshooting need.

### 2.4 DVMT

The UHD 620's framebuffer needs a block of stolen memory. HP consumer BIOSes often hide the DVMT option. Recommendations:

- If visible in the BIOS: **DVMT Pre-Allocated = 64MB**, **DVMT Total Gfx Mem = 256MB / Max**.
- If not visible in the BIOS:
  1. Keep the default first, and verify that the internal display and HDMI work;
  2. If there are garbled screens / HDMI does not light up, use WhateverGreen's `framebuffer-stolenmem` / `framebuffer-fbmem` as a software fallback first;
  3. Do not blindly modify unknown UEFI variables for DVMT; if modification is truly needed, first confirm the BIOS version, variable GUID, and offset with `UEFITool` / `IFR Extractor`.

> The verification baseline for the HDMI fix on this machine is **64MB DVMT**. 32MB may work with a 1080p internal display, but it is not recommended as a starting point for HDMI debugging.

### 2.5 iGPU / dGPU

- Keep **Switchable Graphics / Hybrid Graphics** in the BIOS at the default (usually Enabled).
- Do not select **Discrete Graphics Only**: the MX250 has no driver under macOS, and this would also cut off iGPU output.
- The MX250 is disabled at the ACPI level by `SSDT-Disable_GPU_RP05.aml`; this does not conflict with the BIOS graphics switching setting.
- If the BIOS has **Graphics Memory / UMA Frame Buffer Size**, set it according to the DVMT recommendations in 2.4.

### 2.6 Boot and USB

- **Fast Boot = Disabled**: ensures that the USB keyboard, USB drive, and OpenCore picker are usable.
- **USB Boot = Enabled**: otherwise the BIOS may not show the USB boot entry.
- **Boot Mode = UEFI**, **Legacy Support = Disabled**: the macOS installer only accepts UEFI boot.
- During installation, move the USB drive to the first boot position; after installation, move the internal NVMe's `OpenCore` boot entry to the first position.

---

## 3. If the BIOS Hides DVMT / CFG Lock

This is the most common situation on the HP 15s series. Handle it in the following order:

### Option A: Verify That the Default Values Are Sufficient

1. Install macOS without changing DVMT;
2. Test the internal display at 1920×1080@60 and HDMI at 2560×1440@120;
3. If both work, do not modify hidden variables just to "look more standard".

### Option B: Software Fallback

- CFG Lock: OpenCore `AppleXcpmCfgLock = true`
- DVMT: WhateverGreen `framebuffer-stolenmem` / `framebuffer-fbmem` (calculate according to the WEG documentation and your own resolution; do not copy other people's values)
- These patches can only alleviate insufficient framebuffer memory; **they cannot replace VBT connector mapping**. HDMI's `busId` must still be correct.

### Option C: If UEFI Variables Really Must Be Modified

Proceed only when all of the following conditions are met:

- The machine's BIOS version and BIOS region (`BIOS region` dump) have been confirmed;
- The `Setup` module has been located with `UEFITool`, and the target option's VarStore / offset has been confirmed with `IFR Extractor`;
- The original BIOS has been backed up, or you have confirmed that HP BIOS Recovery is available;
- A hardware programmer or after-sales recovery option is available.

**This document does not provide any `setup_var` offsets**, because they are not portable across BIOS versions.

---

## 4. Post-Change Verification

In Windows, you can quickly confirm:

```powershell
# Confirm UEFI mode (should show UEFI)
msinfo32

# Confirm Secure Boot status (should return False)
Confirm-SecureBootUEFI
```

In macOS, confirm:

```bash
# iGPU and displays
system_profiler SPDisplaysDataType

# Boot arguments (should contain the boot-args from this document)
nvram boot-args
```

---

## 5. BIOS Update Notes

- HP BIOS updates may reset Secure Boot, VT-d, DVMT, boot order, and other settings; after an update, re-check every item.
- Do not update the BIOS while macOS is being installed / debugged; finish the system installation and keep a backup EFI first.
- If HDMI stops working after a BIOS update, first check whether `VT-d`, `DVMT`, and `Secure Boot` were reset, then check whether the VBT changed with the BIOS update (very few models change their VBT).
- If a BIOS update changes the VBT, `con1 busId` may no longer apply; re-extract `i915_vbt.bin` and recalculate according to the [HDMI fix notes](./HDMI-Fix-Notes.md).

---

## 6. Recovery and Emergency

1. Prepare a Windows installation USB drive or a Linux Live USB drive;
2. In the BIOS, perform **Load Setup Defaults**, then disable Secure Boot, VT-d, and CFG Lock again;
3. If you cannot enter the BIOS, try HP's BIOS Recovery (usually by holding `Win + B` or `Win + V` at power-on; refer to your model's manual for the exact key);
4. If OpenCore cannot boot, mount the ESP with another EFI or a Windows boot drive and replace / remove the problematic `config.plist` or kext.

> Do not replace `EFI/OC/Kexts` in bulk without a backup. Change only one variable at a time and record the result.

---

## 7. References

- [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/)
- [OpenCore Configuration (Dortania)](https://dortania.github.io/docs/latest/Configuration.html)
- [acidanthera/WhateverGreen](https://github.com/acidanthera/WhateverGreen)
- [acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg)
- This repository's [HDMI fix notes](./HDMI-Fix-Notes.md)
- This repository's [main README](../../README.en.md)

---

## 8. Items to Confirm on This Machine

The following information has not yet been recorded from this machine's BIOS; it is recommended to fill it in before release:

- [ ] BIOS vendor, version number, and release date
- [ ] Whether DVMT Pre-Allocated / CFG Lock are visible in the BIOS
- [ ] The actual DVMT value set (whether 64MB is the default or manually configured)
- [ ] Whether TPM is enabled (required for dual-boot Windows 11)
- [ ] The `Confirm-SecureBootUEFI` return value after Secure Boot is disabled
- [ ] Whether the VBT changes after a BIOS update
