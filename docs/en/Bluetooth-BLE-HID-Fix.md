> [简体中文](../蓝牙-BLE-HID-修复记录.md)

# Bluetooth BLE HID Fix Notes: `LE_READY_WAIT_FOR_LENGTH` on the RTL8821CE

> Back to [main README](../../README.en.md) | Model: HP 15s-dy0xxx / i7-8565U / RTL8821CE / macOS Tahoe 26.7.2

This document records the complete investigation and fix for the Realtek RTL8821CE combo card under macOS Tahoe, where a **BLE HID keyboard could connect but never complete pairing**. Conclusions first:

- Symptom: `ready-state: LE_READY_WAIT_FOR_LENGTH`, followed by a disconnect with `reason 722`, with no passkey prompt
- Root cause: `bluetoothd` waits for `LE PHY Update Complete (0x0C)` and `LE Data Length Change (0x07)`, but the Realtek controller does not send either event
- Fix: `RealtekBTPatcher.kext` injects two synthetic events after the first real `LE Read Remote Features Complete (0x04)`
- Safety switch: `-ibtpwrap` (without it, hooks are only installed and events counted; no buffer is rewritten)
- Verified: `IBTPInjected=1`, `IBTPHandle=0x0010`, `ready-state: LE_READY_IDLE`, `smpPairingCompleted status=0`

---

## 1. Environment

| Item | Value |
|:---|:---|
| Model | HP 15s-dy0xxx |
| CPU | Intel Core i7-8565U |
| Wireless / Bluetooth combo card | Realtek RTL8821CE |
| Wi-Fi PCI ID | `10EC:C821`, subsystem `103C:831A` |
| Bluetooth USB ID | `0BDA:B00A` |
| macOS | Tahoe 26.7.2 (Darwin 25.6.0 / `25G332`) |
| Bluetooth stack | `bluetoothd` (userspace) |
| Related kexts | `RealtekBluetoothFirmware` 1.0.2, `BlueToolFixup` 2.7.3, `RealtekBTPatcher` |
| Test device | BLE HID keyboard, static random address (`0xD3…`, top bits `11`) |

---

## 2. Symptoms and Logs

### 2.1 Immediate Disconnect After Connecting

The keyboard appears in Bluetooth settings. After clicking Connect, the system shows "Connected" for less than a second, then disconnects; there is no passkey / pairing confirmation dialog. Key excerpts from the system log:

```text
GattServer device "245E5F78-..." connected for session
encryptionStatusChangedCb handle:0x... status=4803 isPairing=0
TS_LE_EncryptionCompleteCb completed with error STATUS 4803
Remote Features: 3D 00 00 00 00 00 00 00 ... ready: wait-for-features, PHYReady:no
Le topology - {... ready-state: LE_READY_WAIT_FOR_LENGTH, enc-state: OFF}
LE Link disconnected ... reason 722 encryptionPending 0
Will not retry
```

> Device UUIDs and handles above have been truncated; fields such as `ready-state`, `reason`, and `STATUS` retain their original values.

### 2.2 What the State Machine Means

`bluetoothd`'s LE connection initialization is not "connect and pair immediately"; it waits for the controller to report capabilities in sequence:

```text
LE Read Remote Features Complete (0x04)
    -> read peer LE features
    -> wait for LE PHY Update Complete (0x0C)
    -> wait for LE Data Length Change (0x07)
    -> ready-state: LE_READY_IDLE
    -> start SMP pairing
```

This machine gets stuck at:

```text
ready-state: LE_READY_WAIT_FOR_LENGTH
```

This means the previous step, `0x04`, has completed, but `0x0C` / `0x07` never arrive. `reason 722` is the disconnect result, not the root cause itself.

---

## 3. "False Leads" Ruled Out First

| Lead | Why it is not the root cause | How it was verified |
|:---|:---|:---|
| `STATUS 4803` / `isPairing=0` | Also appears in successful pairing logs; it is an encryption state callback, not the sole indicator of pairing failure | Compare with the success log in [soyeoning94/hackintosh-logitech-k251-bluetooth-pairing-fix](https://github.com/soyeoning94/hackintosh-logitech-k251-bluetooth-pairing-fix) |
| RPA / LL Privacy | The keyboard uses a **static random address** (`0xD3…`, top bits `11`), not a resolvable private address | Address type in the `bluetoothd` log |
| Stale bond (leftover pairing) | Checked `com.apple.bluetoothd.plist`; no leftover `SMPDistributionKeys`; re-pairing after removing the device still reproduces | Check the pairing database and remove the device, then retry |
| `rtl8821c_config.bin` missing features | This file is provided by the `RealtekBluetoothFirmware` firmware loader; it is a firmware configuration, not an HCI feature bitmap | Check the kext resources and load log |
| USB enumeration / firmware upload failure | The Bluetooth controller enumerates normally, can establish an LE connection, and can return `LE Read Remote Features Complete` | `ioreg` / USB device tree |

---

## 4. Root Cause: Two Missing LE Meta Events

### 4.1 The Two Events `bluetoothd` Waits For

| Event | LE Meta subevent | Purpose | Realtek RTL8821CE |
|:---|:---:|:---|:---:|
| `LE PHY Update Complete` | `0x0C` | Indicates that the PHY has switched to 2M / Coded | ❌ Not sent |
| `LE Data Length Change` | `0x07` | Indicates that TX/RX octets and time have been updated | ❌ Not sent |

`bluetoothd`'s `LE_READY_WAIT_FOR_LENGTH` state requires these two events to complete at least once. If the controller does not send them, the state machine never reaches `LE_READY_IDLE`; SMP never starts, so there is no passkey, and the link eventually disconnects with `reason 722`.

### 4.2 Why Intel's Patch Cannot Be Reused Directly

`IntelBTPatcher` from [OpenIntelWireless/IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware) targets Intel controllers and is built around `IOUSBHostDevice::deviceRequest` / `IOUSBHostPipe`. Its [#514](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/514) analysis and [PR #515](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/pull/515)'s three-path `IOUSBHostIOSource::io` implementation are important references, but the event injection strategy targets Intel's duplicate `0x04` behavior.

Realtek's characteristics:

- It returns **only one** `LE Read Remote Features Complete (0x04)`, so no deduplication is needed;
- Two synthetic events must be appended **after that same real event**;
- Tahoe's pipe/stream I/O tail-jumps into `IOSource::io`, so all three `IOUSBHostIOSource::io` overloads must be hooked; hooking only `IOUSBHostPipe::io` is not enough.

---

## 5. Fix Implementation: `RealtekBTPatcher.kext`

### 5.1 Hook Paths

`RealtekBTPatcher` is a Lilu plugin. Its core hooks:

1. **`IOUSBHostDevice::deviceRequest`** — the HCI command path, used to observe `LE Read Remote Features` commands.
2. **Three `IOUSBHostIOSource::io` overloads** — all must be covered on Tahoe:
   ```text
   __ZN17IOUSBHostIOSource2ioEP18IOMemoryDescriptorjP19IOUSBHostCompletionj
   __ZN17IOUSBHostIOSource2ioEP18IOMemoryDescriptorjRjj
   __ZN17IOUSBHostIOSource2ioEP18IOMemoryDescriptorjP26IOUSBHostBundledCompletionj
   ```
3. **`IOUSBHostPipe::init` / `initWithDescriptors`** — captures the interrupt-IN pipe.
4. Filtering by vendor ID: Realtek `0x0BDA`, Intel `0x8087`.

> The three `IOSource::io` hooks follow the Tahoe routing and deduplication approach from [if-lensman/Hackintosh-MacPro7-1-EFI](https://github.com/if-lensman/Hackintosh-MacPro7-1-EFI); hooking only one path will cause the actual data flow to bypass the patch.

### 5.2 Injection Strategy

After the **first real** `LE Read Remote Features Complete (0x04)`, append two events to the same transfer:

```text
LE PHY Update Complete : 3E 06 0C 00 <h_lo> <h_hi> 02 02                    (8 bytes)
LE Data Length Change  : 3E 0B 07 <h_lo> <h_hi> 1B 00 48 01 1B 00 48 01     (13 bytes)
```

Field descriptions:

| Byte | Meaning |
|:---|:---|
| `3E` | HCI Event Code: LE Meta Event |
| `06` / `0B` | Parameter length |
| `0C` / `07` | LE subevent: PHY Update Complete / Data Length Change |
| `00` | Status: Success |
| `<h_lo> <h_hi>` | Connection handle taken from the real `0x04` event's `data[2]` / `data[3]` (little-endian) |
| `02 02` | TX / RX PHY = LE 2M |
| `1B 00` | Max TX octets = 27 |
| `48 01` | Max TX time = 328 |
| `1B 00` / `48 01` | Max RX octets / time, same as above |

> `0x0010` is exactly the `IBTPHandle` in this machine's success log; the handle must be taken from the real event and must not be hardcoded.

### 5.3 Fallback: Clear the EDL Bit

If the injection path does not hit, the code also attempts to clear the **LE Data Packet Length Extension** bit (bit 5) of `LE Read Local Supported Features`, so that `bluetoothd` skips the length gatekeeping. This path is a fallback, not the primary path that succeeded on this machine.

### 5.4 Differences from the Intel Approach

| Item | Intel `IntelBTPatcher` | Realtek `RealtekBTPatcher` |
|:---|:---|:---|
| Number of `0x04` events | May be duplicated | Only one |
| Deduplication | Required | Not required |
| `IOUSBHostIOSource::io` | Three paths needed on Tahoe | Three paths needed on Tahoe |
| Event injection | Targets the Intel event sequence | Appends `0x0C` + `0x07` |
| Alternative | — | Clear EDL bit 5 (same idea as `0x2022`) |

[flxxxxddd/SimpleMtkBtFw](https://github.com/flxxxxddd/SimpleMtkBtFw)'s [PR #3](https://github.com/flxxxxddd/SimpleMtkBtFw/pull/3) provides an alternative for the MT7922 using `HCI_LE_Set_Data_Length (0x2022)`; this machine does not use that path because the Realtek problem is "events are not reported", not "length is not set".

---

## 6. Installation and boot-args

### 6.1 Recommended Kernel → Add Order

```text
Lilu.kext
RealtekBluetoothFirmware.kext      (1.0.2, official release binary)
BlueToolFixup.kext                 (2.7.3)
RealtekBTPatcher.kext
```

> This is the order used on this machine: Lilu loads first, the firmware loader loads before the stack patches, and `BlueToolFixup` loads before the Realtek-specific patcher. In the actual config, `BundlePath` and dependency relationships take precedence.

### 6.2 boot-args

| Argument | Purpose |
|:---|:---|
| `-ibtpwrap` | **Enables event rewriting**; without it, hooks are only installed and counters published, and no buffer is modified |
| `-ibtcompatdbg` | Outputs `RealtekBTPatcher` diagnostics; can be removed once stable |
| `-ibtcompatoff` | Completely disables the plugin (for troubleshooting / rollback) |
| `-ibtcompatbeta` | Allows loading on beta systems (per the upstream README) |
| `-btlfxa2dpcheck` | `BlueToolFixup`'s A2DP check path, preventing the Bluetooth audio channel from dropping too early; separate from the BLE HID fix |

The relevant fragment finally used on this machine:

```text
... -btlfxa2dpcheck -ibtcompatdbg -ibtpwrap
```

> Do not use `-ibtpwrap` and `-ibtcompatoff` together; the latter has higher priority and disables the plugin.

### 6.3 Safe Rollback

1. Keep a backup EFI that does not contain `RealtekBTPatcher.kext` or `-ibtpwrap`;
2. Boot with `-ibtcompatoff` to verify whether the original behavior returns;
3. If a kernel panic occurs, remove `RealtekBTPatcher.kext` and reset NVRAM.

---

## 7. Verification and Observation

### 7.1 IORegistry Properties

`RealtekBTPatcher` publishes the following properties on the Bluetooth USB device:

| Property | Meaning | Success value on this machine |
|:---|:---|:---|
| `IBTPRouteErr` | `routeMultiple` error code for each symbol; 0 = OK | `dev=0 async=0 iosC=0 iosR=0 iosB=0 initCfg=0 initPipe=0` |
| `IBTPPipeCaptured` | Whether the interrupt pipe was captured | `Y` |
| `IBTPPipeAddr` | Kernel address of the interrupt pipe | — |
| `IBTPIOSourceHits` | Hit count of the three `IOSource::io` paths | `1374` |
| `IBTPWrapEnabled` | Whether `-ibtpwrap` is in effect | `Y` |
| `IBTPReqLRF` | Number of `LE Read Remote Features` commands observed | `1` |
| `IBTPInjected` | Number of synthetic event batches appended | `1` |
| `IBTPHandle` | Connection handle used for the last injection | `0x0010` |
| `IBTPEDLClear` | Number of times the EDL bit was cleared | — (use the actual IORegistry value) |

Collection command:

```bash
ioreg -lw0 | grep -oE '"IBTP[A-Za-z]+" = <[^>]*>'
```

### 7.2 System Logs

```bash
log show --last 5m --style compact | grep -E 'IBTP|ready-state|smpPairingCompleted|LE_READY'
```

Key excerpt on success (abridged):

```text
IBTPRouteErr     = dev=0 async=0 iosC=0 iosR=0 iosB=0 initCfg=0 initPipe=0
IBTPPipeCaptured = Y
IBTPIOSourceHits = 1374
IBTPReqLRF       = 1
IBTPInjected     = 1
IBTPHandle       = 0x0010
IBTPWrapEnabled  = Y

Remote Features ... ready: wait-for-features, PHYReady:no
Data Length Changed for central lmhandle 00000010: Tx [27 -> 27, 328], Rx [27 -> 27, 328],
                    ready-state: wait-for-length, phy-ready 1
ready-state: LE_READY_IDLE
Accepting pairing request from device "245E5F78-..."
smpPairingCompleted address=Random D3:0B:... status=0
```

### 7.3 Pairing Result

- The keyboard pairs successfully and can type normally;
- There is no passkey dialog (BLE HID Just Works / bonding follows the device's capabilities);
- `smpPairingCompleted status=0`;
- First pairing and typing have been verified on this machine; reconnection and re-pairing after deleting the pairing have not been separately recorded and remain to be verified.

---

## 8. Troubleshooting Table

| Symptom | Possible cause | Action |
|:---|:---|:---|
| `IBTPWrapEnabled=N` | `-ibtpwrap` is not present | Add it to boot-args and reboot |
| `IBTPRouteErr` is non-zero | Symbol resolution failed due to a macOS version change | Check whether the Lilu version and `RealtekBTPatcher` match the current system |
| `IBTPPipeCaptured=N` | Interrupt pipe not captured | Confirm USB ID `0BDA:B00A`; check whether the `IOUSBHostPipe::init` hook hits |
| `IBTPIOSourceHits=0` | None of the three `IOSource::io` paths hit | Check whether the system is Tahoe; confirm that no other Bluetooth patch is intercepting |
| `IBTPReqLRF=0` | No `LE Read Remote Features` observed | The keyboard may not have initiated a connection; remove the device and pair again |
| `IBTPInjected=0` with `IBTPWrapEnabled=Y` | The first `0x04` was not captured | Check the filter, pipe, and system log; confirm the device is not Classic BR/EDR |
| `ready-state` remains `LE_READY_WAIT_FOR_LENGTH` | Injected events not accepted by the stack | Check whether the handle is correct; try `-ibtcompatdbg` to observe |
| Connects then disconnects, `reason 722` | Events still missing / state machine timeout | Confirm `IBTPInjected=1`; check whether `-ibtcompatoff` was used by mistake |
| Bluetooth audio issues | A2DP clock / firmware issue, different from BLE HID | Confirm `-btlfxa2dpcheck` and `BlueToolFixup` 2.7.3; check audio logs |

---

## 9. Known Limitations

- This fix has only been verified on **macOS Tahoe 26.x + RTL8821CE + this machine's USB `0BDA:B00A`**; other Realtek models (RTL8822CE, RTL8852, etc.) are not guaranteed to work.
- The primary verification target is a BLE HID keyboard; BLE mice, headphones, multiple simultaneous BLE devices, and reconnection after sleep/wake have not been fully verified.
- The Xbox controller uses Classic BR/EDR and does not go through the LE event path, so it does not need this patch; this does not prove that other Classic devices need no patch.
- The synthetic events are a compatibility layer for missing controller behavior, not a Realtek firmware fix; system or firmware updates may change the event sequence and break the patch.
- `-ibtpwrap` rewrites HCI event buffers; remove the argument when it is not needed, since the default mode only observes and does not rewrite.
- If `BlueToolFixup` or `RealtekBluetoothFirmware` is upgraded later, the version combination must be re-verified; this machine is pinned to `2.7.3` + `1.0.2`.

---

## 10. References

See the [Credits section of the main README](../../README.en.md#credits) for the full acknowledgements and all external links. Core references directly related to this document:

- [OpenIntelWireless/IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware) (zxystd's original `IntelBTPatcher`)
  - [issue #514](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/514)
  - [PR #515](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/pull/515)
  - [issue #497](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/497)
- [if-lensman/Hackintosh-MacPro7-1-EFI](https://github.com/if-lensman/Hackintosh-MacPro7-1-EFI)
- [thegwchr/RealtekBluetoothFirmware](https://github.com/thegwchr/RealtekBluetoothFirmware)
- [acidanthera/BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM) (`BlueToolFixup`)
- [soyeoning94/hackintosh-logitech-k251-bluetooth-pairing-fix](https://github.com/soyeoning94/hackintosh-logitech-k251-bluetooth-pairing-fix)
- [flxxxxddd/SimpleMtkBtFw](https://github.com/flxxxxddd/SimpleMtkBtFw) / [PR #3](https://github.com/flxxxxddd/SimpleMtkBtFw/pull/3)
- [watchthelight-labs/WTLLegionBtFix](https://github.com/watchthelight-labs/WTLLegionBtFix)
- [Emilelu/RealtekBTPatcher](https://github.com/Emilelu/RealtekBTPatcher)
- [Emilelu/BlueToolFixup-Fix26](https://github.com/Emilelu/BlueToolFixup-Fix26)


---

## 9. Bluetooth keyboard broken after macOS 26.7.3 update (2026-11-10)

**Symptom**: after updating from 26.7.2 to 26.7.3, the Bluetooth keyboard is discovered but never
connects, and Bluetooth cycles on/off.

### Two independent problems

**Problem A: `bluetoothd` crashes in a loop (EXC_GUARD namespc 18)**

Log:
```
bluetoothd ... Full corpse enqueued for bluetoothd
bluetoothd | exception: EXC_GUARD, subtype GUARD_TYPE_USER, namespc 18, reason 2
```
The crash address corresponds to the **length check of `bluetoothInternalControllerInfo`**
(it must be exactly 14 bytes).

**Cause**: after the upgrade, NVRAM `bluetoothExternalDongleFailed` was set to `%01`
(third-party dongle failed to initialise), so `bluetoothd` refused the controller and crashed.

**Fix**: add both keys to `config.plist → NVRAM → Add` and `Delete`:
```
bluetoothExternalDongleFailed   = 00        (1 byte)
bluetoothInternalControllerInfo = 00 x 14   (14 bytes)
```
The `Delete` list forces a reset at every boot, even if the system rewrites the value.
After the fix the controller returns to `THIRD_PARTY_DONGLE`, `Firmware v61592`,
`Address DC:F5:05:FF:E4:02`.

**Problem B: keyboard still won't connect, stuck at `LE_READY_WAIT_FOR_LENGTH`**

Even with Bluetooth stable, a pure-BLE keyboard connects then stops at `LE_READY_WAIT_FOR_LENGTH`,
then disconnects after 30 s with `reason 722`.

**Fix**: update `IntelBTPatcher.kext` (commit `b16fe69`). Key logic:

| Feature | Trigger |
|---|---|
| Hide LE Data Packet Length Extension bit (bit5) | **unconditional** (so bluetoothd can initiate a connection) |
| Inject LE Data Length Change event (0x07) | gated by `-ibtpwrap` |
| Inject LE PHY Update Complete event (0x0C) | **removed** (causes Bluetooth soft-reset on 26.7.3) |
| Forward original HCI events | **always kept** (avoids panic) |

> ⚠️ **Earlier bug**: a `if(!g_wrapEnabled) return;` at the top of `rewriteHciEvent`
> stopped the original forwarding and caused a kernel panic. Removed; only the rewrite/inject
> paths are gated.

**Verified** (real pairing success):
```
Data Length Changed ... ready-state: wait-for-length, phy-ready 0
Le topology ... ready-state: LE_READY_IDLE
Classic Pairing Agent successfully created
Accepting pairing request from "82F8798D"
Encryption is now enabled
smpPairingCompleted ... status=0
```
`IBTPInjected=31`, `IBTPHandle=0x0010` confirm the injection used the real connection handle.

### New `bm3_usb` module in 26.7.3

26.7.3's `bluetoothd` adds a `bm3_usb` module (USB dongle probe) that natively supports 0BDA:B00A:
```
[bm3_usb][Found_USB_Dongle] -- FoundBluetoothInterfaces() for deviceClass (224),
   idVendor = 0x0BDA, idProduct = 0xB00A returned true
[bm3_usb] setting gUseInternalController to FALSE
```
So the Realtek controller is correctly recognised as `THIRD_PARTY_DONGLE`. This module is not the problem.

### Fixed & committed after this

- `config.plist` (sanitised), boot-args include `-ibtpwrap`
- `config.plist.KNOWNGOOD-BT-BLEHID-FIXED` (verified working snapshot)
- `Kexts/IntelBTPatcher.kext` (`b16fe69` build, sha256 `89f9b0c8...`)

### Key diagnostic commands

```bash
# controller state
system_profiler SPBluetoothDataType

# is bluetoothd crashing?
pgrep -x bluetoothd
log show --last 5m --predicate 'process == "kernel"' --style compact | grep -c 'corpse enqueued for bluetoothd'

# patcher counters (IORegistry)
ioreg -lw0 | grep -oE '"IBTP(Injected|Handle|EDLClear|WrapEnabled)" = <[^>]*>'

# LE connection state
log show --last 2m --style compact --predicate 'subsystem == "com.apple.bluetooth"' | grep -E 'Le topology|ready-state'
```

### Troubleshooting checklist
1. First confirm the NVRAM Bluetooth variables are not corrupted (Problem A)
2. If Bluetooth is stable but the keyboard still won't connect, watch whether `IBTPInjected`
   increments on connect (whether the Problem-B patch fired)
3. If the system connects but is stuck at `LE_READY_WAIT_FOR_LENGTH`, confirm the
   Data Length Change injection is executing

---

## 11. IntelBTPatcher 2.6.0 / 2.6.1: wake-from-sleep panic fix and boot guard

### 2.6.0 - fixing a kernel UAF panic after wake

**Symptom**: about 0.8 s after waking from clamshell sleep, a kernel panic with this stack:

```
Kernel trap type = 14=page fault, Fault CR2: 0x268
com.zxystd.IntelBTPatcher : CIntelBTPatcher::newHostDeviceRequest(...) + 0x160
 <- IOUSBHostDevice::deviceRequest
 <- IOUSBHostDevice::getDescriptorGated <- getDescriptor <- getDeviceDescriptor
 <- CIntelBTPatcher::captureIntelInterruptPipe + 0x40
 <- CIntelBTPatcher::newInitPipeConfig + 0x40
 <- AppleUSBXHCIPipe::withDescriptorsAndOwners <- AppleUSBXHCIDevice::createPipe
 <- IOUSBHostDevice::start (_block_invoke)
```

**Root cause**: the old code kept a bare `IOUSBHostDevice *` in `g_btDevice` without retaining it.
On wake the USB stack re-enumerates and releases the old device object; when the new device's
`start()` runs, the hook writes IORegistry properties through the stale pointer, so
`g_btDevice->setProperty()` dispatches through a dead vtable (`callq *0x268(%rcx)` with `%rcx == 0`)
and page-faults.

**Fix** (commit `98b3228`):
- `setBtDevice()` retains the new device before swapping and releases the old one.
- `btDeviceUsable()` checks the pointer is non-null **and** its first field (the vtable slot) is non-null.
- `setPropData()` / `setDeviceData()` explicitly release the `OSData::withBytes` object, fixing a
  kernel-object leak on every HCI device request.

### 2.6.1 - boot guard (off by default)

`-ibtpguard` makes the rewriting logic observe-only for the first 60 s of uptime
(`IBTPArmed=N` then `Y`). It exists to rule out the boot-time controller race: the four errors seen at
every boot (`OI_HCICMD_ReadLocalVersionInformation failed: STATUS 408`,
`Failed to read local Fast LE Connection data ... result:11`,
`LE_InitMultipleAdvertising failed: STATUS 152`, `Server.LE.Scan Failed to add rule`)
are **not** caused by this patch - they also appear in logs where the keyboard connected fine, and
in logs from 2026-11-10.

> Default behaviour is identical to 2.6.0 unless `-ibtpguard` is passed.

### Building without Xcode

The macOS 26 `ld` no longer synthesises `_kmod_info`, so a hand-linked kext fails with
`kexts must have a _kmod_info symbol` and OpenCore skips it silently. The repo script handles this:

```sh
zsh scripts/build-local.sh 2.6.1
```

It generates `kmod_info.cpp`, fills in the `Info.plist` platform keys, and asserts both `_kmod_info`
and `CFBundleSupportedPlatforms` are present. Two gotchas: `TARGET_OS_OSX` in
`MacKernelSDK/Headers/TargetConditionals.h` must be 0, and `-D` values must not contain spaces.

### Current pinned version

- `Kexts/IntelBTPatcher.kext`: 2.6.1, sha256 `c3967d6ed13d6e65a7a21775639c28413b76a63d1a7aa3eed9ef486b39667512`
- boot-args still `-ibtpwrap` (no `-ibtpguard`, i.e. 2.6.0 behaviour)

## 12. Important: `codesign` failing on `bluetoothd` is normal here

This is easy to trip over while debugging Bluetooth, so it is written down.

### Symptom

```sh
codesign -v /usr/sbin/bluetoothd
# /usr/sbin/bluetoothd: invalid signature (code or signature have been modified)
```

Its sha256 also differs from Apple's, in just 11 bytes across three spots:

```
0x0edd1e   1 byte
0x1e7bb7   8 consecutive 0x90 bytes (a NOP sled)
0x45c13e   45 31 f6 eb 25   (original: 45 85 f6 74 25)
```

### Cause

All three are written by **BlueToolFixup itself** when `-btlfxa2dpcheck` is active.
It routes `_cs_validate_page` and rewrites code pages of `/usr/sbin/bluetoothd`
in place, before the kernel validates them:

- `0x45c13e` is the A2DP gate: `test r14d,r14d; je` becomes `xor r14d,r14d; jmp`,
  making bluetoothd unconditionally take the audio-publish path (this is what stops
  AirPods audio from breaking up);
- `0x1e7bb7` is the NOP sled `searchAndPatch` leaves behind.

Both are valid x86, not bit rot, and not disk damage.

### The decisive test (stop suspecting the disk)

Toggle only the daemon; touch no storage:

```sh
sudo launchctl bootout system/com.apple.bluetoothd
sudo purge
shasum -a 256 /usr/sbin/bluetoothd     # -> GOOD 579bb5f5… (Apple original)
sudo launchctl kickstart -k system/com.apple.bluetoothd
shasum -a 256 /usr/sbin/bluetoothd     # -> BAD  c31f8ff4…
```

**The bytes follow the process, so the process is the author.** The file's other
vnode (`/System/Volumes/Update/mnt1/usr/sbin/bluetoothd`) has its own page-cache
entry that the patcher never touches, which is why it always hashes clean — that
once got misread as "the boot snapshot is corrupted".

### Three consequences

1. **Do not remove `-btlfxa2dpcheck`.** Without it AirPods audio breaks up.
2. **`btpurge-watch.sh` must never use the file hash as a health criterion.** While
   bluetoothd runs, the file always looks "bad", so a hash check would make the
   watchdog restart it every 60 seconds forever. v3 judges controller state only
   (`State` / `Firmware` / `Address`).
3. **The occasional `EXC_BAD_ACCESS (SIGBUS)` was the inherent cost of this patch**,
   not a hardware fault. bluetoothd is an APFS-compressed binary; once a page has
   been rewritten, a later fault must re-decompress it from the compressed stream,
   the result no longer matches the recorded hash, the fault fails and the process
   takes a `SIGBUS`. **This is now fixed** — see §12.1 below.

### 12.1 The SIGBUS fix: drop the compression flag (2026-11-15)

The crash came from the *hash-verified decompression path*, not from the rewritten
bytes. So remove the compression flag: a plain file has no hash to mismatch.

```sh
sudo umount "/Volumes/Macintosh HD 1"
sudo mount -o nobrowse -t apfs /dev/disk1s4 /System/Volumes/Update/mnt1
sudo mount -uw /System/Volumes/Update/mnt1
sudo chflags nocompressed /System/Volumes/Update/mnt1/usr/sbin/bluetoothd
sudo bless --mount /System/Volumes/Update/mnt1 --bootefi --create-snapshot
sudo reboot
```

(Requires `csrutil authenticated-root` to be disabled; `bless --create-snapshot`
needs `--bootefi` and must not be given a boot.efi path argument.)

Measured 58 minutes after booting the new snapshot:

| | before | after |
| --- | --- | --- |
| `ls -lO` flags | `restricted,compressed` | `restricted` |
| bluetoothd SIGBUS crashes | 3 on 11-14 evening | **0** |
| real `apfs_announce_hash_mismatch` | 10 on 11-14, 32 on 11-10 | **0** |
| controller | `attrib_on / v61592 c30136` | `attrib_on / v61592 c30136` |
| encrypted connections | working | 2, still working |

The patched bytes remain and `codesign -v` still fails — expected and harmless.
What disappeared is the hash-checked decompression path.

**A macOS update can re-compress or replace the file and silently undo this.**
Hence the `com.local.btdecomp.track` daemon (`/usr/local/bin/bt-decomp-track.py`,
samples every 15 minutes into `/var/log/bt-decomp-fix.log`):

```
2026-11-15 00:24:57 decomp=yes crashes=63 newest=… mismatch=0 bt=attrib_on|v61592 c30136|DC:F5:05:FF:E4:02
```

`decomp=NO` or `mismatch>0` means an update undid the fix; rerun the five commands.

### Also: `-25291` is not why the keyboard fails

`bluetoothd` logs

```
Failed to query keychain for device Random D3:0C:D1:FA:CA:CA with result -25291
```

`-25291` is `errSecNotAvailable`. It looks like "the patch invalidated the signature,
so bluetoothd lost its keychain entitlement, so it cannot read the bond, so HID fails",
but the numbers refuse that story:

- one single boot: **523** occurrences of `-25291` **and** 7 successful
  `Encryption is now enabled`;
- 2026-11-13, 11:00 hour: 342 `-25291` **and** a successful encrypted keyboard connection;
- 2026-11-14 23:32:40: `Outgoing connection … successful` →
  `Encryption is now enabled`, with `-btlfxa2dpcheck` active.

It is background noise on this Realtek/Tahoe combination. The keyboard's real
intermittent failure is that **a BLE keyboard does not advertise while idle** (only
on a keypress) plus a boot-time race, covered earlier in this document.
