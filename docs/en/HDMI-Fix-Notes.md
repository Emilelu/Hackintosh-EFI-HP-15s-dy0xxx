> [简体中文](../HDMI-修复记录.md)

# HDMI Fix Notes: From VBT to `con1 busId = 1`

> Back to [main README](../../README.en.md) | Model: HP 15s-dy0xxx / i7-8565U / macOS Tahoe 26.7.2

This document records the complete process of bringing up an external HDMI display on this machine's Intel UHD Graphics 620 (native `0x3EA0`, spoofed as `0x3EA5`) under macOS Tahoe. Conclusions first:

- Physical interface: **HDMI = DDI B, native TMDS, no LSPCON**
- WhateverGreen connector: **`con1 busId = 1`**
- Must be used as a set: `igfxonln=1 igfxonlnfbs=0x0100 igfxfcms=1 igfxfcmsfbs=0x0100`
- Verified: **2560×1440 @ 120Hz**
- 4K60 is unavailable: this machine has no LSPCON, and the Tahoe native HDMI path has a 300MHz limit; 4K30 is also restricted by the mode table

---

## 1. Environment and Symptoms

| Item | Value |
|:---|:---|
| Model | HP 15s-dy0xxx |
| CPU | Intel Core i7-8565U (Whiskey Lake, 4C8T) |
| iGPU | Intel UHD Graphics 620, native `0x3EA0` |
| macOS | Tahoe 26.7.2 (Darwin 25.6.0 / `25G332`) |
| Bootloader | OpenCore 1.x (placeholder) |
| SMBIOS | `MacBookPro15,2` |
| Internal display | 1920×1080 @ 60Hz |
| External display | HDMI, verified 2560×1440 @ 120Hz |

**Symptoms**: The internal display works and iGPU acceleration works, but there is no signal when an HDMI monitor is connected. The system can detect hot-plug events, yet AppleIntelFramebuffer cannot read a valid EDID, and neither `fIsHDMI` nor `newOnline` becomes true.

The typical failure paths are:

1. Changing only con1's `type` to HDMI (`0x0800`) while keeping `busId=5` copied from a CFL template;
2. Or changing it to `busId=1` but not enabling force-online / force-modeset at the same time;
3. The system log shows no `fIsHDMI=1`, `newOnline=1`, or shows `getDisplayStatus=0`, so modeset is never triggered.

> This is not a case of "the HDMI port is broken" or "the UHD 620 does not support external displays"; it is a case where two conditions — **connector mapping + display online state** — are not satisfied at the same time.

---

## 2. First, Confirm the Physical Port: Decode `i915_vbt.bin`

### 2.1 Why the VBT Matters

macOS's AppleIntelFramebuffer does not tell us whether the HDMI port on the motherboard is wired to DDI B or DDI C. Laptops with the same generation of UHD 620 can have completely different VBTs: HDMI may be on DDI B, on DDI C, or routed through an LSPCON. Trying `busId` values blindly is guesswork.

The VBT (Video BIOS Table) is provided by the BIOS and contains, for each child device:

- `device_type`: HDMI / DP / eDP / LVDS
- `dvo_port`: DDI port (HDMIB, HDMIC, etc.)
- `lspcon`: whether the path goes through an LSPCON
- `ddc_pin`: DDC / GMBUS pin

The Linux i915 driver parses the VBT and exposes the raw binary through debugfs.

### 2.2 Extracting the VBT

In an Ubuntu Live session (or any Linux installation with a reasonably recent kernel), run:

```bash
sudo mount -t debugfs none /sys/kernel/debug 2>/dev/null || true

# The path may be /sys/kernel/debug/dri/0/i915_vbt,
# or a directory with a PCI address; use whichever actually exists.
sudo cp /sys/kernel/debug/dri/0000:00:02.0/i915_vbt /tmp/i915_vbt.bin
sudo chmod a+r /tmp/i915_vbt.bin
```

If the file is not found, first confirm that i915 is loaded:

```bash
ls /sys/kernel/debug/dri/
lspci -nnk | grep -A3 VGA
```

### 2.3 Decoding the VBT

Install [intel-gpu-tools](https://gitlab.freedesktop.org/drm/igt-gpu-tools):

```bash
sudo apt update
sudo apt install intel-gpu-tools
```

View the full contents of Block 2 (General Definitions):

```bash
intel_vbt_decode --file=/tmp/i915_vbt.bin --block=2 --hexdump
```

You can also decode the whole file first to confirm the BDB version, child count, and port summary:

```bash
intel_vbt_decode --file=/tmp/i915_vbt.bin --hexdump
```

> The official usage of `intel_vbt_decode` is `--file=FILE`; some older versions also accept a positional argument. If the command errors out, refer to `intel_vbt_decode --help`.

### 2.4 The 5-Byte Header of `struct bdb_general_definitions`

Linux's [`intel_vbt_defs.h`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_vbt_defs.h) defines:

```c
struct bdb_general_definitions {
    u8 crt_ddc_gmbus_pin;   /* 1 byte */
    u8 dpms_non_acpi:1;     /* bitfield in 1 byte */
    u8 skip_boot_crt_detect:1;
    u8 dpms_aim:1;
    u8 rsvd1:5;
    u8 boot_display[2];     /* 2 bytes */
    u8 child_dev_size;      /* 1 byte */
    u8 devices[];           /* array of child_device_config */
} __packed;
```

Therefore the header is fixed at **5 bytes**:

```text
offset 0x00: crt_ddc_gmbus_pin     (1 B)
offset 0x01: DPMS / reserved       (1 B)
offset 0x02: boot_display[0..1]    (2 B)
offset 0x04: child_dev_size        (1 B)
offset 0x05: start of child[0]
```

Block 2's `block_size` does not include the 3-byte BDB block header (block id + 2-byte size). Therefore:

```text
child count = (block_size - 5) / child_dev_size
```

Information for this machine:

| Field | Value | Meaning |
|:---|:---|:---|
| BDB version | **212** | Falls in the 196–215 range |
| `child_dev_size` | **38** (`0x26`) | Linux `child_device_expected_size()` returns 38 for BDB 196–215 |
| Header size | 5 bytes | `sizeof(struct bdb_general_definitions)` |
| Block 2 `block_size` | **309** | Based on the actual VBT |
| Child count | **8** | `(309 - 5) / 38 = 8`, remainder 0 |

> Note: the array may contain empty entries with `device_type == 0`; Linux `parse_general_definitions()` skips them, but they are still counted when calculating the total number of children. Do not infer `block_size` from "how many non-empty entries I see".

### 2.5 Key Fields of `child[1]` on This Machine

The HDMI port in this machine's VBT is at **`child[1]`**, with the following fields:

| Field | VBT value | Linux definition / interpretation |
|:---|:---:|:---|
| `device_type` | `0x60D2` | `DEVICE_TYPE_HDMI`, i.e. an HDMI/DVI TMDS output |
| `dvo_port` | `0x01` | `DVO_PORT_HDMIB` → **DDI B** |
| `lspcon` | `0` | **No LSPCON**, native TMDS |
| `ddc_pin` | `0x01` | `DDC_BUS_DDI_B` → GMBUS pin 1 |
| `i2c_pin` | Used together with DDC | Not used as the primary criterion |

The corresponding Linux constants come from [`intel_vbt_defs.h`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_vbt_defs.h):

```c
#define DEVICE_TYPE_HDMI     0x60D2
#define DVO_PORT_HDMIB       1
enum vbt_gmbus_ddi {
    DDC_BUS_DDI_B = 0x1,
    DDC_BUS_DDI_C,
    DDC_BUS_DDI_D,
    ...
};
```

**Conclusion: physical HDMI = DDI B + native TMDS + DDC pin 1.**

---

## 3. How VBT Maps to WhateverGreen

### 3.1 `ConnectorInfo` Fields

WhateverGreen's [`kern_fb.hpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_fb.hpp) defines:

```cpp
struct PACKED ConnectorInfo {
    int8_t index;
    uint8_t busId;      // GMBUS pin
    uint8_t pipe;
    uint8_t pad;
    ConnectorType type;
    ConnectorFlags flags;
};
```

The comment on `busId` explicitly states that it is a **GMBUS ID**, not the DDI letter and not the macOS framebuffer index. Some older platforms/tutorials write `GMBUS_PIN_DPB` as `5`, but this machine is the CNP generation, whose mapping differs.

### 3.2 DDC Mapping on the CNP Generation

`cnp_encoder_to_ddc_pin()` in Linux [`intel_hdmi.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_hdmi.c):

```c
static u8 cnp_encoder_to_ddc_pin(struct intel_encoder *encoder)
{
    switch (encoder->port) {
    case PORT_B:
        ddc_pin = GMBUS_PIN_1;   // == 1
        break;
    case PORT_C:
        ddc_pin = GMBUS_PIN_2;
        break;
    case PORT_D:
        ddc_pin = GMBUS_PIN_4;
        break;
    ...
    }
}
```

From [`intel_gmbus.h`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_gmbus.h):

```c
#define GMBUS_PIN_1   1
#define GMBUS_PIN_2   2
#define GMBUS_PIN_4   4
```

Therefore, on this machine:

```text
VBT ddc_pin = 0x01
    -> DDC_BUS_DDI_B
    -> PORT_B
    -> CNP: GMBUS_PIN_1
    -> WhateverGreen busId = 1
```

### 3.3 Triple Evidence

| Evidence | Content | Notes |
|:---|:---|:---|
| 1. VBT measurement | `ddc_pin=0x01`, `dvo_port=0x01`, `lspcon=0` | Physical port is DDI B, DDC is pin 1 |
| 2. Same-series EFI | [shammu2bd/HP-15s-du1xxx-Hackintosh](https://github.com/shammu2bd/HP-15s-du1xxx-Hackintosh)'s [`config.plist`](https://github.com/shammu2bd/HP-15s-du1xxx-Hackintosh/blob/main/EFI/OC/config.plist) uses `framebuffer-con1-busid=1` | An independent implementation for the same HP 15s series and same-generation UHD 620 |
| 3. Upstream discussion | The comment in [acidanthera/bugtracker #420](https://github.com/acidanthera/bugtracker/issues/420#issuecomment-511499674) explicitly says "Changing the `bus-id` of the HDMI connector to one of the allowed `bus-id`s for HDMI (`1` works for me)" | Shows that `1` is one of the valid busIds for HDMI |

Supplement: the mechanism that actually makes this mapping correct is CNP's `PORT_B -> GMBUS_PIN_1`, see [Linux `intel_hdmi.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_hdmi.c). `busId=1` is neither magic nor "the number of DDI B".

---

## 4. Final Configuration

### 4.1 Writing All Three Connectors at Once

The final configuration on this machine uses a single 36-byte `framebuffer-con0-alldata` that fills all three connectors starting from con0:

| Connector | index | busId | pipe | pad | type | flags | Purpose |
|:---:|---:|---:|---:|---:|:---:|:---:|:---|
| con0 | 0 | 0 | 8 | 0 | `0x0002` (LVDS) | `0x0098` | eDP internal display |
| con1 | 1 | **1** | 9 | 0 | `0x0800` (HDMI) | `0x01C7` | HDMI / DDI B |
| con2 | 2 | 4 | 10 | 0 | `0x0400` (DP) | `0x01C7` | DP @ DDI C |

Raw hex:

```text
00 00 08 00 02 00 00 00 98 00 00 00
01 01 09 00 00 08 00 00 C7 01 00 00
02 04 0A 00 00 04 00 00 C7 01 00 00
```

Base64:

```text
AAAIAAIAAACYAAAAAQEJAAAIAADHAQAAAgQKAAAEAADHAQAA
```

Can be placed directly into OpenCore:

```xml
<key>framebuffer-con0-alldata</key>
<data>AAAIAAIAAACYAAAAAQEJAAAIAADHAQAAAgQKAAAEAADHAQAA</data>
```

> `framebuffer-con0-alldata` is the entire connector table starting at con0; do not mistake it for "only modifying con0". `con1`'s busId is at offset `0x0D` (the 14th byte), with value `0x01`.

### 4.2 If You Only Want a Minimal Patch

Equivalently, you can write con1 separately:

```xml
<key>framebuffer-con1-enable</key>
<data>AQAAAA==</data>
<key>framebuffer-con1-busid</key>
<data>AQAAAA==</data>
<key>framebuffer-con1-type</key>
<data>AAgAAA==</data>
<key>framebuffer-con1-flags</key>
<data>hwEAAA==</data>
```

> Choose one of the two approaches; do not mix both for the same field. This machine ultimately uses `con0-alldata`, which keeps the pipe / type of all three connectors fixed in a single table.

### 4.3 The boot-args That Must Be Used as a Set

```text
igfxonln=1 igfxonlnfbs=0x0100 igfxfcms=1 igfxfcmsfbs=0x0100
```

| Argument | Purpose | Symptom when missing |
|:---|:---|:---|
| `igfxonln=1` | Forces the display online | Display is judged offline, `newOnline=0` |
| `igfxonlnfbs=0x0100` | Restricts force-online to FB1 (`0x0100` is the FB1 bitmask) | No effect on FB1 |
| `igfxfcms=1` | Forces modeset completion | Even if online, the display does not light up |
| `igfxfcmsfbs=0x0100` | Restricts force-modeset to FB1 | FB1 does not trigger modeset |

These four are not "optional optimizations" but a necessary part of getting HDMI to light up on this machine. Without this group, `getDisplayStatus` returns `0`, and modeset is never entered.

### 4.4 EDID Override

```xml
<key>AAPL01,override-no-connect</key>
<data>(insert the Base64 of the display's real 128-byte EDID here)</data>
```

- The key name must be **`AAPL01`**: HDMI on this machine corresponds to FB1; `AAPL00` would target the internal display FB0.
- Only **128 bytes** are supported: see [acidanthera/bugtracker #2374](https://github.com/acidanthera/bugtracker/issues/2374). 256-byte EDID or DisplayID extension blocks are not read.
- The preferred DTD in this machine's EDID: **3840×2160 @ 30Hz, 297MHz**.
- EDID contains privacy information such as the display serial number, so **the raw bytes are not published in this document**; extract it from your own display and do not copy someone else's EDID.

Example of extracting a 128-byte EDID (Linux):

```bash
# The HDMI interface name may be card0-HDMI-A-1 / card1-HDMI-A-1; use the actual one
cat /sys/class/drm/card0-HDMI-A-1/edid > /tmp/hdmi-edid.bin
ls -l /tmp/hdmi-edid.bin          # usually 128 or 256 bytes
edid-decode /tmp/hdmi-edid.bin    # confirm the preferred DTD and checksum
head -c 128 /tmp/hdmi-edid.bin | base64 -w0
```

> If `edid-decode` reports 256 bytes / an Extension block, take only the first 128 bytes. WEG will not read the extra part; it cannot be used to solve the 4K mode problem.

---

## 5. Log Evidence

### 5.1 Key Lines When Output Works

After `busId=1` + the four force arguments + the EDID override all take effect, the following appears for the first time:

```text
[Getonline_info] fIsHDMI=1, fEnableAudio=1, newOnline=1, newPortType=1
```

Field meanings:

| Field | Value | Meaning |
|:---|:---:|:---|
| `fIsHDMI` | 1 | The driver identifies this connector as HDMI/TMDS |
| `fEnableAudio` | 1 | The HDMI audio path is marked as enabled |
| `newOnline` | 1 | The display enters the online state |
| `newPortType` | 1 | Port type switching completed |

### 5.2 When the Force Arguments Are Missing

If only the connector is changed without the full set of `igfxonln` / `igfxfcms`, you will see:

```text
getDisplayStatus=0
```

When `getDisplayStatus` is 0, the subsequent `setDisplayMode` is not triggered and HDMI stays black. This is not an EDID problem; the display state machine never reaches a state where modeset can happen.

### 5.3 How to Collect These Logs

WhateverGreen's `kern_igfx_debug.cpp` provides DEBUG trace points such as `getOnlineInfo`, `getDisplayStatus`, and `connectionProbe`:

- Use the **DEBUG build of WhateverGreen**;
- Temporarily add `-igfxfbdbg`;
- Filter `AppleIntelFramebuffer` / `igfx` with `log show` or `dmesg`.

```bash
log show --last 5m --style compact | grep -E 'Getonline_info|getDisplayStatus|fIsHDMI|igfx'
```

> `kern_igfx_debug.cpp` is DEBUG-only framebuffer tracing code; the RELEASE build of WhateverGreen does not include these logs.

---

## 6. Failed Attempts

The table below lists the failure paths actually encountered or verified during troubleshooting on this machine. They explain "why HDMI still does not light up even though everything looks correct".

| # | Attempt | Result | Root cause / conclusion |
|:---:|:---|:---|:---|
| 1 | con1 `type=0x0800`, keeping `busId=5` | No signal; no `fIsHDMI=1` | `5` is the `GMBUS_PIN_DPB` number on some older platforms; DDI B on this machine's CNP is `GMBUS_PIN_1` |
| 2 | Change `busId=1` without force-online / force-modeset | Incomplete connector state, `getDisplayStatus=0` | The display state machine never enters modeset |
| 3 | Add only `igfxonln`, missing `igfxfcms` | Still cannot light up reliably | force online and force modeset are two independent paths and must be used together |
| 4 | Add `flags=0x0008` (`CNConnectorAlwaysConnected`) to HDMI con1 | Black screen; the internal display may be knocked out | `CNConnectorAlwaysConnected` treats the connector as an internal display/LVDS; `port-number=0` conflicts with the internal display |
| 5 | Use `-igfxmpc=600` | Argument value ignored; 4K still absent | `-igfxmpc` is a bare flag and does not parse `=600`; the internal default is 675MHz |
| 6 | Inject a 256-byte EDID / DisplayID extension | Only the first 128 bytes are used | `override-no-connect` only supports 128 bytes, see #2374 |
| 7 | Try to light up 4K60 through the LSPCON path | This machine has no LSPCON and cannot enable it | VBT `lspcon=0`; HDMI is native TMDS |
| 8 | Change con1 `flags` to `0x1CF` | Black screen / no signal | `flags` is not "the more bits the better"; `0x1CF` breaks the semantics of native `0x01C7`; keep `0x01C7` |

### 6.1 EDID Variant Experiment (7 Variants, All Failed)

During troubleshooting, **7 different 128-byte EDID variants** were constructed and tested, covering different preferred DTDs (4K30 / 4K25 / 1440p120), CEA extension block / SVD trimming, vendor and product ID substitution, checksum fixes, and other combinations. The results were consistent:

- The system recognizes the display as having a valid EDID, but the `AppleIntelFramebuffer` mode list does not gain 3840×2160;
- Some variants produced no signal at all because the DTD was non-compliant or did not match the real display's capabilities;
- This confirms that **override EDID is not the source of the 4K mode table**, and piling up more EDID variants yields no benefit.

> The specific bytes of the 7 variants are not retained one by one, to avoid being copied as a "usable 4K EDID". EDID must come from your own display.

### Things Not to Do

1. **Do not add `CNConnectorAlwaysConnected (flags 0x0008)` to an HDMI connector.** This flag is intended for internal displays/LVDS; when added to an external connector, the port-number calculation conflicts with the internal display and causes an immediate black screen. `con0`'s `flags=0x0098` contains this bit and is specific to the internal display — do not copy it to con1.
2. **Do not write `-igfxmpc=600`.** WhateverGreen only checks whether `-igfxmpc` is present and does not parse the value after the equals sign. To change the limit, only use properties/code paths supported by WEG, and 675MHz still does not equal 4K60.
3. **Do not brute-force scan busId values.** Decode the VBT first, confirm the `ddc_pin` and DDI mapping, and then decide the `busId`. Blindly trying 0–6 may knock out the internal display or break the audio path.
4. **Do not copy someone else's EDID.** EDID contains vendor, model, serial number, and DTD; a wrong EDID will cause macOS to select the wrong mode.
5. **Do not mix `framebuffer-con0-alldata` with `framebuffer-con1-*` at the same time.** When the same connector is overwritten by two sets of keys, the final value depends on WEG's processing order and is extremely difficult to debug.
6. **Do not treat `busId` as the DDI letter.** `DDI B` does not imply `busId=2`, and `DDI C` does not imply `busId=3`; they are completely different numbering systems.

---

## 7. Reproduction Steps (Checklist)

- [ ] BIOS: Secure Boot disabled, DVMT ≥ 64MB, CFG Lock disabled (or bypassed with a quirk)
- [ ] Extract `i915_vbt.bin` from Ubuntu Live
- [ ] `intel_vbt_decode --file=i915_vbt.bin --block=2 --hexdump`
- [ ] Confirm `child_dev_size=38` and compute the child count as `(block_size-5)/38`
- [ ] Confirm the HDMI child has `device_type=0x60D2`, `dvo_port=1`, `lspcon=0`, `ddc_pin=1`
- [ ] Write the 36-byte `framebuffer-con0-alldata` (or the equivalent con1 patch) into `config.plist`
- [ ] Add `igfxonln=1 igfxonlnfbs=0x0100 igfxfcms=1 igfxfcmsfbs=0x0100`
- [ ] Extract the display's own 128-byte EDID and write it to `AAPL01,override-no-connect`
- [ ] Reboot and confirm the log shows `[Getonline_info] fIsHDMI=1, fEnableAudio=1, newOnline=1, newPortType=1`
- [ ] Confirm 2560×1440 @ 120Hz in System Settings
- [ ] Keep a bootable backup EFI before adjusting any other parameters

---

## 8. Why 4K Is Still Unavailable

This is the most important "known limitations" section of this document. **Do not assume that 4K60 is only one busId away just because 2560×1440@120 works.**

### 8.1 The 300MHz Hardcode in `fMaxPixelClock`

In the native HDMI branch of `AppleIntelFramebuffer::getOnlineInfo` in macOS Tahoe, there is the following assignment:

```asm
movq $0x11E1A300, 0x4368(%rbx)
```

`0x11E1A300 = 300,000,000`, i.e. **300MHz**.

[b00t0x/WhateverXe issue #41](https://github.com/b00t0x/WhateverXe/issues/41) provides a byte-by-byte analysis on ADL-S / UHD 770 / Tahoe 25G229: this immediate value limits `fMaxPixelClock` to 300MHz; the same `Getonline_info` field group in this machine's log also reports `fMaxPixelClock=300000000`. The ~594MHz required for 4K60 is excluded. Although 4K30 (297MHz) is below the limit, whether the mode table can list 4K30 still depends on the mode list provided by AppleIntelFramebuffer.

### 8.2 What `-igfxmpc` Does and Does Not Do

`MaxPixelClockOverride` in WhateverGreen's [`kern_igfx_clock.cpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_igfx_clock.cpp) hooks `AppleIntelFramebuffer::connectionProbe` and rewrites `maxPixelClock` in `IOFBTimingRange`. Its default value in [`kern_igfx.hpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_igfx.hpp) is:

```cpp
uint64_t maxPixelClockFrequency = 675000000; // 675MHz
```

However:

- `-igfxmpc` is only an **enable switch** and does not parse `=600`;
- 675MHz is only the upper bound of WEG's `IODisplayTimingRangeV1.maxPixelClock`; `-igfxmpc` **only changes this upper bound and does not add any mode to `AppleIntelFramebuffer`'s mode table**;
- Tahoe's native HDMI branch may still write `fMaxPixelClock` back to 300MHz;
- `FBBoostPixelFrequencyLimit(0x100)` has **no effect** on CFL: it does not change the 300MHz limit of CFL native HDMI, nor the mode table;
- [acidanthera/bugtracker #1377](https://github.com/acidanthera/bugtracker/issues/1377) also records a case where "after setting `enable-max-pixel-clock-override`, a 4K panel can still only do 4K@48", showing that max pixel clock alone is not enough.

This also explains an easily confusing observation: this machine's 2560×1440@120 pixel clock is about **483.4MHz** and still works because `-igfxmpc` raises `IOFBTimingRange.maxPixelClock` to **675MHz**; however, the `fMaxPixelClock` field is still 300MHz, and the 4K modes never enter the mode table. The two facts are not contradictory.

### 8.3 The Mode Table Does Not Read the Override EDID's DTD

The role of `AAPL01,override-no-connect` is to **replace the EDID data**, not to inject a new mode into AppleIntelFramebuffer. Its behavior is:

1. When the driver reads the EDID of the connected display, the override data is used instead;
2. But the final list of available modes is still provided by `AppleIntelFramebuffer`'s internal table / driver logic;
3. Therefore, a 4K30 DTD in the EDID **does not automatically become a 4K30 option in the system mode list**.

[b00t0x/WhateverXe issue #41](https://github.com/b00t0x/WhateverXe/issues/41) also points out explicitly that CoreGraphics' mode enumeration includes true 1080p60 and 720p60, but the 3840×2160 raster used by HiDPI 4K is still blocked by the 300MHz limit. This further shows that "the EDID has a 4K DTD" does not mean "the system will offer a 4K mode".

### 8.4 This Machine Has No LSPCON

The VBT clearly states:

```text
lspcon = 0
```

The role of LSPCON is to convert DisplayPort to HDMI 2.0 in order to break through the native TMDS 300MHz limit. This machine's HDMI is **native TMDS** with no LSPCON chip, therefore:

- 4K60 (~594MHz) is not feasible;
- 4K30 (297MHz) is theoretically feasible from a clock perspective, but is limited by the mode table as described in 8.3;
- Modifying the EDID, adding `-igfxmpc`, or changing `busId` cannot conjure an LSPCON.

### 8.5 Conclusion

```text
2560×1440 @ 120Hz  -> verified working
3840×2160 @ 30Hz   -> theoretically feasible, but not provided by the system mode table; not implemented
3840×2160 @ 60Hz   -> no LSPCON on this machine + 300MHz limit; not achievable
```

If your display supports 2560×1440@120, that is currently the most suitable HDMI working mode for this machine. Do not change `CNConnectorAlwaysConnected` or scan `busId` values in pursuit of 4K — those paths do not bypass the 300MHz limit and will instead introduce black screens.

### 8.6 Proposed 4K30 Configuration (To Be Verified)

The 297MHz of 4K30 is below the 300MHz limit, so it is **theoretically feasible from a clock perspective**; the current obstacles are mainly the mode table and stolen memory configuration. The following plan references the successful 4K30 case on the Xiaomi Pro (UHD 620) and is **to be verified**; success has not been confirmed on this machine:

1. **BIOS**: `DVMT Pre-Allocated = 64MB` (if the BIOS offers higher values, 96MB / 128MB is safer).
2. **Remove `framebuffer-fbmem` and `framebuffer-stolenmem`**: these two VRAM patches squeeze the available stolen memory too low, causing AppleIntelFramebuffer not to list 4K modes. If `framebuffer-unifiedmem` is also present, evaluate removing it as well.
3. **Add `-igfxhdmidivs` to boot-args**: WhateverGreen's HDMI divider fix; the official description says it helps with 2K / 4K modes on HDMI 1.4 ports.
4. **A/B test the EDID override**: keep the correct 128-byte EDID or temporarily remove it, and observe whether AppleIntelFramebuffer lists a 4K30 mode by itself; the key is not whether the EDID contains a 4K DTD, but whether the driver mode table generates 4K30.
5. **Verification**: after rebooting, check whether `3840×2160 @ 30Hz` is available in "System Settings → Displays"; also confirm `[Modeset] ... 297000000 Hz` or a close value in the log.

Risks and caveats:

- If the actual DVMT is only 32MB, removing `framebuffer-stolenmem` may cause a black screen / panic at boot; confirm that the BIOS is set to 64MB first, or add back only a 32MB `framebuffer-stolenmem` as a fallback, and do not add `framebuffer-fbmem` / `framebuffer-unifiedmem` at the same time.
- `-igfxhdmidivs` only affects the clock divider during the modeset stage and **does not generate modes**; adding it when no 4K30 mode exists will not "produce" 4K.
- This plan involves trade-offs against the existing 1080p / 1440p configuration; always keep a bootable backup EFI, change one item at a time, and record the result.
- Reference case: [daliansky/XiaoMi-Pro-Hackintosh issue #117](https://github.com/daliansky/XiaoMi-Pro-Hackintosh/issues/117). Whether it applies to this machine depends on BIOS / DVMT and VBT; **success is not guaranteed**.

---

## 9. Appendix: Decoding `framebuffer-con0-alldata` Byte by Byte

```text
offset  bytes                       field
0x00    00                          con0.index = 0
0x01    00                          con0.busId = 0
0x02    08                          con0.pipe = 8
0x03    00                          con0.pad = 0
0x04    02 00 00 00                 con0.type = 0x0002 (LVDS)
0x08    98 00 00 00                 con0.flags = 0x0098
0x0C    01                          con1.index = 1
0x0D    01                          con1.busId = 1  <-- key
0x0E    09                          con1.pipe = 9
0x0F    00                          con1.pad = 0
0x10    00 08 00 00                 con1.type = 0x0800 (HDMI)
0x14    C7 01 00 00                 con1.flags = 0x01C7
0x18    02                          con2.index = 2
0x19    04                          con2.busId = 4
0x1A    0A                          con2.pipe = 10
0x1B    00                          con2.pad = 0
0x1C    00 04 00 00                 con2.type = 0x0400 (DP)
0x20    C7 01 00 00                 con2.flags = 0x01C7
```

All multi-byte fields are little-endian.

---

## 10. References

See the [Credits section of the main README](../../README.en.md#credits) for the full acknowledgements and all external links. Core references directly related to this document:

- [acidanthera/WhateverGreen `kern_fb.hpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_fb.hpp)
- [acidanthera/WhateverGreen `kern_igfx_clock.cpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_igfx_clock.cpp)
- [acidanthera/WhateverGreen `kern_igfx_debug.cpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_igfx_debug.cpp)
- [acidanthera/Lilu](https://github.com/acidanthera/Lilu)
- [acidanthera/bugtracker #420](https://github.com/acidanthera/bugtracker/issues/420#issuecomment-511499674)
- [acidanthera/bugtracker #1377](https://github.com/acidanthera/bugtracker/issues/1377)
- [acidanthera/bugtracker #2374](https://github.com/acidanthera/bugtracker/issues/2374)
- [b00t0x/WhateverXe issue #41](https://github.com/b00t0x/WhateverXe/issues/41)
- [z1000biker/HP-Desktop-Pro-G2-Tahoe-UHD630-HDMI-Fix](https://github.com/z1000biker/HP-Desktop-Pro-G2-Tahoe-UHD630-HDMI-Fix)
- [shammu2bd/HP-15s-du1xxx-Hackintosh](https://github.com/shammu2bd/HP-15s-du1xxx-Hackintosh)
- [dreAd1811/HACKINTOSH-EFI-HP-15s-du0xxx](https://github.com/dreAd1811/HACKINTOSH-EFI-HP-15s-du0xxx)
- [daliansky/XiaoMi-Pro-Hackintosh issue #117](https://github.com/daliansky/XiaoMi-Pro-Hackintosh/issues/117)
- [torvalds/linux `intel_vbt_defs.h`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_vbt_defs.h)
- [torvalds/linux `intel_bios.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_bios.c)
- [torvalds/linux `intel_hdmi.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_hdmi.c)
- [torvalds/linux `intel_gmbus.h`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_gmbus.h)
- [intel-gpu-tools / `intel_vbt_decode`](https://gitlab.freedesktop.org/drm/igt-gpu-tools/-/blob/master/tools/intel_vbt_decode.c)
