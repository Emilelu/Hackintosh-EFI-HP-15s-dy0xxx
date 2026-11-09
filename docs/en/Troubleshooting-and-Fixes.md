> [简体中文](../疑难问题与修复记录.md)

# Troubleshooting and Fix Notes

> Back to [main README](../../README.en.md) | Model: HP 15s-dy0xxx / i7-8565U / UHD 620 / macOS Tahoe 26.7.2

This document collects the problems most easily misdiagnosed or repeatedly encountered during Hackintosh debugging on this machine. Each entry is recorded as "Symptom → Root cause → Action → Verification", with reproducible criteria wherever possible. Some root causes come from reverse engineering and source comparison of WhateverGreen / AppleIntelFramebuffer / WebKit; links are provided in the text.

## Quick Reference

| # | Issue | One-line conclusion |
|:---:|:---|:---|
| 1 | `disable-agdc=1` causes a 20-minute boot | WEG's AGDC disabler changes `IOFBProbeOptions` (1 → 1025); do not use it |
| 2 | `CNConnectorAlwaysConnected` (flags `0x0008`) causes a black screen | Forces `port-number=0`, which conflicts with the internal LVDS port 0; never add it to an external connector |
| 3 | Wrong connector table bytes cause boot hang / black screen | `plutil -data` on the command line misaligns bytes; write with `plistlib` or `-xml`, and verify field by field |
| 4 | `framebuffer-patch-enable` disables WEG's automatic DP→HDMI patch | `hdmiAutopatch = !applyFramebufferPatch && ...`; either go fully automatic or fully manual |
| 5 | Diagnostic methodology | `-v` is mandatory; on macOS 26 use IORegistry properties instead of `IOLog` / `SYSLOG` |
| 6 | Other ruled-out traps | `igfxonln` does not change EDID, `-igfxmpc=600` is not parsed, override EDID only supports 128B, etc. |
| 7 | Safari still 60fps on a 120Hz external display | WebKit preference `PreferPageRenderingUpdatesNear60FPSEnabled` defaults to true |

---

## 1. `disable-agdc=1` Causes a 20-Minute Boot

### Symptom

- The boot progress bar is extremely slow, usually taking **about 20 minutes** to reach the desktop;
- The system itself does not panic, and the log shows no obvious errors;
- After removing this configuration, the same EFI boots normally again (in the tens of seconds).

### Root Cause

`disable-agdc` is an AGDC (Apple Graphics Device Control) disabling switch provided by WhateverGreen. It changes AppleIntelFramebuffer's `IOFBProbeOptions`: normally the value is `1`, but with the AGDC disabler enabled it becomes **`1025` (`0x401`)**. This extra bit sends framebuffer probing into an abnormal fallback / wait path, which on this machine manifests as a long hang during boot.

`IOFBProbeOptions` is the direct evidence for diagnosing this issue:

```bash
# Normally it should be close to 1; if you see 1025 / 0x401, check whether disable-agdc is enabled
ioreg -lw0 | grep -i IOFBProbeOptions
```

### Action

- **Do not use `disable-agdc`**: remove the key from `DeviceProperties`; if there is an equivalent AGDC-disabling argument in boot-args, remove it as well.
- Do not stack other framebuffer properties just to "make AGDC not work"; the HDMI fix is unrelated to AGDC.
- If it has already been written and you forgot to back up, restore `config.plist.bak-*` following the single-user mode procedure in section 5.

### Verification

- Time the reboot: normally the desktop should appear within tens of seconds;
- `IOFBProbeOptions` in `ioreg` returns to `1`;
- If it is still slow, check whether any `-igfxagdc`-style arguments or other framebuffer properties remain.

---

## 2. `CNConnectorAlwaysConnected` (connector flags bit `0x0008`) Causes a Black Screen

### Symptom

- The internal or external display is black;
- `IODisplayConnect=0`, and the system report shows zero displays;
- But `Display status=1`, which looks like "the display is online" while there is actually no output;
- In severe cases the internal display is also knocked out.

### Root Cause

`CNConnectorAlwaysConnected` is connector flags bit `0x0008`, meaning "this connector is always considered connected", usually used for **internal displays / LVDS**. When it is added to an external HDMI connector (con1 on this machine), the system forcibly sets that connector's `port-number` to `0`; the internal LVDS/eDP display itself occupies port 0, so the two conflict and the display state machine enters an "online but no output" black-screen state.

This machine's `con0` (internal display) has `flags=0x0098`, which already contains `0x0008`; this is an internal-display-specific configuration, and **this value must not be copied to `con1`**. `con1` should keep the native `0x01C7`.

### Action

- **Never set `flags 0x0008` on an external HDMI connector**;
- Keep `con1` at `flags=0x01C7` and `con2` at `0x01C7`;
- If you need to "force online", use the boot-args `igfxonln=1` / `igfxfcms=1` instead of changing connector flags;
- After modifying, always decode and print `idx/bus/pipe/type/flags` to confirm that `0x01C7` was not written as `0x01CF` or some other value.

### Verification

- `system_profiler SPDisplaysDataType` shows both the internal and external displays;
- `IODisplayConnect` is non-zero;
- If the system report shows 0 displays but `Display status=1`, suspect a wrong flags value first.

---

## 3. Wrong Connector Table Bytes Cause Boot Hang / Black Screen

### Symptom

- After modifying `framebuffer-con0-alldata`, boot hangs, the screen is black, or the internal display works but HDMI disappears;
- Decoding reveals fields inconsistent with expectations, e.g. intending to write `00000800 / 01C7` but actually getting `03000800 / 0107`;
- Most likely when passing base64 from the command line with `plutil -data`.

### Root Cause

`plutil -data`'s command-line argument handling reinterprets the base64 string according to shell / plutil rules, causing byte misalignment or truncation. A connector record is a fixed 12-byte structure, so any 1-byte offset shifts `type` / `flags` as a whole. OpenCore does not validate the semantics of these fields, and the wrong values are used directly by WhateverGreen / AppleIntelFramebuffer, resulting in a boot hang or black screen.

### Action

**Prefer direct reads/writes with Python `plistlib`** (bypassing shell base64 conversion):

```python
import plistlib
from pathlib import Path

path = Path("config.plist")
with path.open("rb") as f:
    cfg = plistlib.load(f)

dev = cfg["DeviceProperties"]["Add"]["PciRoot(0x0)/Pci(0x2,0x0)"]
data = dev["framebuffer-con0-alldata"]
assert len(data) == 36, f"expect 36 bytes, got {len(data)}"

for i in range(3):
    rec = data[i * 12:(i + 1) * 12]
    idx = rec[0]
    bus = rec[1]
    pipe = rec[2]
    typ = int.from_bytes(rec[3:7], "little")
    flags = int.from_bytes(rec[7:11], "little")
    print(f"con{i}: idx={idx} bus=0x{bus:02X} pipe=0x{pipe:02X} "
          f"type=0x{typ:04X} flags=0x{flags:04X}")

# Verify first, then write back; back up before writing
with path.open("wb") as f:
    plistlib.dump(cfg, f, fmt=plistlib.FMT_XML, sort_keys=False)
```

You can also use plutil's XML value form to avoid command-line base64 parsing:

```bash
plutil -replace 'DeviceProperties.Add.PciRoot(0x0)/Pci(0x2,0x0).framebuffer-con0-alldata' \
  -xml '<data>AAAIAAIAAACYAAAAAQEJAAAIAADHAQAAAgQKAAAEAADHAQAA</data>' config.plist
```

**After every change, decode and print `idx / bus / pipe / type / flags` for field-by-field verification**, and compare against the table below:

| Connector | idx | busId | pipe | type | flags |
|:---|:---:|:---:|:---:|:---:|:---:|
| con0 | 0 | `0x00` | `0x08` | `0x0002` | `0x0098` |
| con1 | 1 | **`0x01`** | `0x09` | `0x0800` | `0x01C7` |
| con2 | 2 | `0x04` | `0x0A` | `0x0400` | `0x01C7` |

### Backup and Recovery

- Before modification: `cp config.plist config.plist.bak-$(date +%Y%m%d-%H%M%S)`;
- After modification: keep at least one bootable backup EFI;
- If it can no longer boot: restore the backup following the `Cmd+S` single-user procedure in section 5.

---

## 4. `framebuffer-patch-enable` Disables WEG's Automatic DP→HDMI Patch

### Symptom

- HDMI previously worked, but adding `framebuffer-patch-enable=1` or some manually written connector properties makes HDMI go black instead;
- Or changing only part of con1's fields breaks HDMI.

### Root Cause

WhateverGreen's automatic HDMI patching logic is:

```cpp
bool hdmiAutopatch = !applyFramebufferPatch && ...;
```

Once `framebuffer-patch-enable` is true, `applyFramebufferPatch` is true, and **WEG's automatic DP→HDMI patch is disabled entirely**. At that point the system relies completely on your manually written connector table; if only half the set is written (for example, only `con1-type` without `con1-busId` / `pipe` / `flags`), you get an incomplete or even self-contradictory connector configuration.

### Action

Choose one of the two, do not do half a job:

1. **Fully automatic**: delete `framebuffer-patch-enable` and all manually written `framebuffer-conX-*` properties, and let WEG detect and patch automatically;
2. **Fully manual**: keep `framebuffer-patch-enable=1` and write the complete 36-byte `framebuffer-con0-alldata` in one go (all three segments con0 / con1 / con2), rather than only the partial fields of con1.

This machine ultimately uses option 2: the complete 36-byte table + `framebuffer-patch-enable=1`. If you only want a minimal patch, use the complete combination `framebuffer-con1-enable` + `framebuffer-con1-busid` + `framebuffer-con1-type` + `framebuffer-con1-flags` instead, and do not mix it with `con0-alldata`.

### Verification

- Decode and verify all fields of all three connectors;
- If the display does not light up, temporarily remove `framebuffer-patch-enable` and the manually written table to check whether WEG's automatic path can light it up, which tells you whether the problem lies in the manual table.

---

## 5. Diagnostic Methodology

### 5.1 Boot Problems Require `-v`

For any "boot hang / black screen / garbled screen" problem, the first step is to add `-v debug=0x100 keepsyms=1` and capture the complete boot log. Without `-v`, OpenCore only shows the Apple logo, and it is impossible to tell which kext or which stage the boot is stuck at.

### 5.2 On macOS 26, Do Not Rely on Lilu Plugins' `IOLog` / `SYSLOG`

macOS 26 (Tahoe) imposes stricter limits on third-party kext log output; Lilu plugins' `IOLog` / `SYSLOG` often fail to appear in `log show`. Prefer **IORegistry properties** as the observation surface:

```bash
# Example: RealtekBTPatcher injection counters
ioreg -lw0 | grep -oE '"IBTP[A-Za-z]+" = <[^>]*>'

# Example: framebuffer probe options
ioreg -lw0 | grep -i IOFBProbeOptions
```

If you must see WEG's framebuffer debug logs, use the DEBUG build of WhateverGreen and add `-igfxfbdbg`; but note that **it must not be used together with `disable-agdc`**, because the combination panics (see 5.3).

### 5.3 WEG DEBUG and `disable-agdc` Must Not Be Used Together

- `-igfxi2cdbg` enables WEG's I2C / framebuffer debug output;
- But when it coexists with `disable-agdc`, this machine actually triggers a **kernel panic**;
- Troubleshooting order: remove `disable-agdc` first, then add `-igfxi2cdbg`; do not reverse it.

### 5.4 Recovery Procedure: `Cmd+S` Single-User → Mount ESP → Restore Backup

When a bad `config.plist` change prevents the system from booting:

1. Hold **`Cmd+S`** at boot to enter single-user mode (if FileVault is enabled, you may need to unlock first);
2. Mount the ESP:

   ```bash
   diskutil list
   diskutil mount disk0s1        # use the actual diskXsY of your ESP
   ```

3. Enter the OpenCore directory and restore the backup:

   ```bash
   cd /Volumes/EFI/EFI/OC
   ls -l config.plist.bak-*
   cp config.plist.bak-20260101-120000 config.plist   # replace with your backup filename
   ```

4. Confirm the file size / checksum is correct, then `reboot`;
5. If even single-user mode cannot be entered, mount the ESP with another EFI or a Windows / Linux boot drive and replace the file.

> Always create a timestamped backup before every change to `config.plist`; do not keep only one "final version".

---

## 6. Other Ruled-Out Traps

| Trap | Reality | Correct approach |
|:---|:---|:---|
| `igfxonln` / `force-online` can fix EDID | It only changes the online state of `getDisplayStatus`, not the EDID contents | For EDID issues use `AAPL01,override-no-connect` (128 bytes) |
| `-igfxmpc=600` can raise the pixel clock | WEG only recognizes the bare flag `-igfxmpc` and does not parse `=600`; internally fixed at 675MHz | Write `-igfxmpc` without a value |
| `AAPL0X,override-no-connect` can write 256 bytes | Only **128 bytes** supported; 256-byte / DisplayID extensions are not read | Inject only the first 128 bytes; trim extension blocks if necessary |
| `enable-hdmi20` / `-cdfon` can enable HDMI 2.0 on newer systems | macOS 11+ no longer implements this path | Do not pin 4K60 hopes on this argument; this machine has no LSPCON and 4K60 is not feasible |
| `STATUS 4803` definitely indicates a Bluetooth failure | `STATUS 4803` / `isPairing=0` also appear in successful pairing logs | Rely on `ready-state` and `smpPairingCompleted status` |
| `rtl8821c_config.bin` is an LE feature switch | It is a Realtek firmware configuration / RF parameter file, not an HCI feature bitmap | Do not replace this file to "enable LE" |
| `disable-agdc` can solve display problems | Changes `IOFBProbeOptions` from 1 to 1025, causing a 20-minute boot | Do not use it; solve HDMI with connector + `igfxonln` / `igfxfcms` |

---

## 7. Safari Still 60fps on a 120Hz External Display

> ✅ **Status: RESOLVED (verified on real hardware, 2026-11-10)** — after disabling the Safari feature flag "Prefer Page Rendering Updates near 60fps", testufo reports 120fps again.


### Symptom

- The external display is confirmed at **120Hz** in System Information / Display settings;
- Desktop, Chrome, and Firefox may be fine, but **Safari page animations, scrolling, and testufo still show 60fps**;
- Changing to a 120Hz display, changing cables, or changing resolutions all have no effect.

### Root Cause

WebKit has a hidden preference, **`PreferPageRenderingUpdatesNear60FPSEnabled`**, whose default value is `true`. It caps the page rendering update rate at close to 60fps even though `DisplayLinkMac` has already obtained a 120Hz refresh rate from the display. The relevant implementation is in:

- [`DisplayLinkMac.cpp`](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/UIProcess/mac/DisplayLinkMac.cpp): display link and refresh-rate synchronization;
- [`Page.cpp`](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/page/Page.cpp): page rendering update scheduling;
- [`AnimationFrameRate.cpp`](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/platform/graphics/AnimationFrameRate.cpp): animation frame rate and the 60fps preference;
- [`UnifiedWebPreferences.yaml`](https://github.com/WebKit/WebKit/blob/main/Source/WTF/Scripts/Preferences/UnifiedWebPreferences.yaml): definition of `PreferPageRenderingUpdatesNear60FPSEnabled`.

### Fix (GUI)

1. Open **Safari → Settings → Advanced**;
2. Check **"Show features for web developers"**;
3. A **"Develop"** menu appears in the menu bar; go to **Develop → Feature Flags**;
4. Search for **"Prefer Page Rendering Updates near 60fps"**;
5. **Uncheck** it;
6. **`Cmd+Q` to quit Safari completely**, then reopen it (closing windows is not enough).

### Fix (Command Line, Fallback)

First quit Safari completely, then run:

```bash
defaults write com.apple.Safari WebKitPreferPageRenderingUpdatesNear60FPSEnabled -bool false
```

Then reopen Safari. If your system settings have "restore / sync", confirm that the preference is not overridden by iCloud or a configuration profile.

### Verification

- Open <https://www.testufo.com/refreshrate> and confirm that it shows 120fps (or close to the display's refresh rate);
- Open a high-frame-rate animation page in Safari and compare it with Chrome / Firefox;
- If **Chrome / Firefox are also limited to 60fps**, the limit is at the system level (display link, WindowServer composition, or cable / interface bandwidth), not a WebKit preference; in that case go back to display settings and HDMI link troubleshooting;
- If only Safari is at 60fps, apply the preference change above.

---

## 8. Pre-Change Checklist

- [ ] `config.plist` and the entire `EFI` folder have been backed up;
- [ ] The current `config.plist` has been confirmed to pass `ocvalidate`;
- [ ] Only one variable is changed at a time (one of: connector, boot-arg, kext);
- [ ] After changing a connector, it has been decoded with `plistlib` / `plutil` and verified field by field;
- [ ] A bootable backup EFI (USB drive or another partition) is available;
- [ ] When a black screen / boot hang occurs, add `-v` first and record the last log line, then recover according to section 5.

## 9. References

- [acidanthera/WhateverGreen](https://github.com/acidanthera/WhateverGreen) — framebuffer / AGDC / `igfxonln` / `igfxfcms` / `-igfxmpc` implementation.
- [acidanthera/Lilu](https://github.com/acidanthera/Lilu) — plugin loading and patching framework.
- [acidanthera/bugtracker #420](https://github.com/acidanthera/bugtracker/issues/420) — community evidence for `con1 busId=1`.
- [acidanthera/bugtracker #2374](https://github.com/acidanthera/bugtracker/issues/2374) — override EDID only supports 128 bytes.
- [WebKit/WebKit `DisplayLinkMac.cpp`](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/UIProcess/mac/DisplayLinkMac.cpp)
- [WebKit/WebKit `Page.cpp`](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/page/Page.cpp)
- [WebKit/WebKit `AnimationFrameRate.cpp`](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/platform/graphics/AnimationFrameRate.cpp)
- [WebKit/WebKit `UnifiedWebPreferences.yaml`](https://github.com/WebKit/WebKit/blob/main/Source/WTF/Scripts/Preferences/UnifiedWebPreferences.yaml)
- [WebKit bug 173434](https://bugs.webkit.org/show_bug.cgi?id=173434)
- [kevmoo/flutter-wasm-compare#40](https://github.com/kevmoo/flutter-wasm-compare/issues/40)
- [wailsapp/wails#6056](https://github.com/wailsapp/wails/issues/6056)

> See the [Credits section of the main README](../../README.en.md#credits) for the full acknowledgements and all external links. For the detailed derivation of the HDMI connector, see the [HDMI fix notes](./HDMI-Fix-Notes.md); for Bluetooth event injection, see the [Bluetooth BLE HID fix notes](./Bluetooth-BLE-HID-Fix.md).


---

## 9. Internal display brightness resets after every reboot

**Symptom**: brightness returns to default after each reboot.

**Diagnosis**:
- NVRAM `backlight-level` exists and updates when the slider moves (the value **is** saved)
- OpenCore `NVRAM → Delete` does not remove it
- The culprit is **`applbkl = 03000000`**: this custom backlight data prevents macOS from applying the saved `backlight-level` at boot

**Fix**: **remove `applbkl`** from `DeviceProperties → PciRoot(0x0)/Pci(0x2,0x0)`, keep:
- `enable-backlight-registers-fix = 01000000`
- boot-arg `-igfxblr` (these two are what actually fix the "black screen for minutes at boot" issue)

**Verified**: 2026-11-10, brightness persists across reboots after removing `applbkl`.
