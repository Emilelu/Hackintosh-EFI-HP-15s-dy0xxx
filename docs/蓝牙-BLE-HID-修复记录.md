> [English](./en/Bluetooth-BLE-HID-Fix.md)

# 蓝牙 BLE HID 修复记录：RTL8821CE 的 `LE_READY_WAIT_FOR_LENGTH`

> 返回 [主 README](../README.md)｜机型：HP 15s-dy0xxx / i7-8565U / RTL8821CE / macOS Tahoe 26.7.2

本文记录 Realtek RTL8821CE 组合卡在 macOS Tahoe 下 **BLE HID 键盘能连接但无法完成配对**的完整排查与修复。结论先行：

- 症状：`ready-state: LE_READY_WAIT_FOR_LENGTH`，随后 `reason 722` 断开，没有 passkey 提示
- 根因：`bluetoothd` 等待 `LE PHY Update Complete (0x0C)` 与 `LE Data Length Change (0x07)`，Realtek 控制器不发这两个事件
- 修复：`RealtekBTPatcher.kext` 在第一条真实 `LE Read Remote Features Complete (0x04)` 后注入两个合成事件
- 安全开关：`-ibtpwrap`（不加时只安装 hook、只计数，不改写任何 buffer）
- 实测：`IBTPInjected=1`、`IBTPHandle=0x0010`、`ready-state: LE_READY_IDLE`、`smpPairingCompleted status=0`

---

## 1. 环境

| 项目 | 值 |
|:---|:---|
| 机型 | HP 15s-dy0xxx |
| CPU | Intel Core i7-8565U |
| 无线 / 蓝牙组合卡 | Realtek RTL8821CE |
| Wi-Fi PCI ID | `10EC:C821`，subsystem `103C:831A` |
| 蓝牙 USB ID | `0BDA:B00A` |
| macOS | Tahoe 26.7.2（Darwin 25.6.0 / `25G332`） |
| 蓝牙协议栈 | `bluetoothd`（用户态） |
| 相关 kext | `RealtekBluetoothFirmware` 1.0.2、`BlueToolFixup` 2.7.3、`RealtekBTPatcher` |
| 复现设备 | BLE HID 键盘，static random address（`0xD3…`，top bits `11`） |

---

## 2. 症状与日志

### 2.1 连接后立即断开

键盘能在蓝牙设置里出现，点击连接后系统显示「已连接」不到一秒，随后断开；没有 passkey / 配对确认窗口。系统日志的关键片段：

```text
GattServer device "245E5F78-..." connected for session
encryptionStatusChangedCb handle:0x... status=4803 isPairing=0
TS_LE_EncryptionCompleteCb completed with error STATUS 4803
Remote Features: 3D 00 00 00 00 00 00 00 ... ready: wait-for-features, PHYReady:no
Le topology - {... ready-state: LE_READY_WAIT_FOR_LENGTH, enc-state: OFF}
LE Link disconnected ... reason 722 encryptionPending 0
Will not retry
```

> 上面的设备 UUID、handle 已做省略；`ready-state`、`reason`、`STATUS` 等字段保持原值。

### 2.2 关键状态机的含义

`bluetoothd` 的 LE 连接初始化不是「连上就配对」，而是按顺序等待控制器上报能力：

```text
LE Read Remote Features Complete (0x04)
    -> 读取对端 LE feature
    -> 等待 LE PHY Update Complete (0x0C)
    -> 等待 LE Data Length Change (0x07)
    -> ready-state: LE_READY_IDLE
    -> 开始 SMP pairing
```

本机卡在：

```text
ready-state: LE_READY_WAIT_FOR_LENGTH
```

这说明前一步 `0x04` 已经完成，但 `0x0C` / `0x07` 从未到达。`reason 722` 是断开结果，不是根因本身。

---

## 3. 先排除的「假线索」

| 线索 | 为什么不是根因 | 验证方式 |
|:---|:---|:---|
| `STATUS 4803` / `isPairing=0` | 成功配对日志中同样会出现；它是加密状态回调，不是配对失败的唯一标志 | 对照 [soyeoning94/hackintosh-logitech-k251-bluetooth-pairing-fix](https://github.com/soyeoning94/hackintosh-logitech-k251-bluetooth-pairing-fix) 的成功日志 |
| RPA / LL Privacy | 键盘使用 **static random address**（`0xD3…`，top bits `11`），不是可解析私有地址 | `bluetoothd` 日志中的 address type |
| Stale bond（旧配对残留） | 检查 `com.apple.bluetoothd.plist`，没有 `SMPDistributionKeys` 残留；移除设备后重新配对仍复现 | 检查配对数据库并移除设备重试 |
| `rtl8821c_config.bin` 缺少 feature | 该文件由 `RealtekBluetoothFirmware` 固件加载器提供，是固件配置，不是 HCI feature bitmap | 检查 kext 资源与加载日志 |
| USB 枚举 / 固件上传失败 | 蓝牙控制器能正常枚举、能建立 LE 连接、能返回 `LE Read Remote Features Complete` | `ioreg` / USB 设备树 |

---

## 4. 根因：两个缺失的 LE Meta 事件

### 4.1 `bluetoothd` 等待的两个事件

| 事件 | LE Meta subevent | 作用 | Realtek RTL8821CE |
|:---|:---:|:---|:---:|
| `LE PHY Update Complete` | `0x0C` | 告知 PHY 已切换到 2M / Coded | ❌ 不发 |
| `LE Data Length Change` | `0x07` | 告知 TX/RX octets 与 time 已更新 | ❌ 不发 |

`bluetoothd` 的 `LE_READY_WAIT_FOR_LENGTH` 状态要求这两个事件至少完成一次。控制器不发，状态机就永远不进入 `LE_READY_IDLE`；SMP 不会开始，所以没有 passkey，链路最终以 `reason 722` 断开。

### 4.2 为什么 Intel 的补丁不能直接复用

[OpenIntelWireless/IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware) 的 `IntelBTPatcher` 针对 Intel 控制器，围绕 `IOUSBHostDevice::deviceRequest` / `IOUSBHostPipe` 构建；其 [#514](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/514) 分析与 [PR #515](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/pull/515) 的三路 `IOUSBHostIOSource::io` 实现是重要参考，但事件注入策略针对 Intel 的重复 `0x04` 行为。

Realtek 的特点：

- **只返回一条** `LE Read Remote Features Complete (0x04)`，不需要去重；
- 需要在**同一条真实事件之后**追加两个合成事件；
- Tahoe 的 pipe/stream I/O 会 tail-jump 到 `IOSource::io`，必须 hook 三条 `IOUSBHostIOSource::io` 重载，不能只 hook `IOUSBHostPipe::io`。

---

## 5. 修复实现：`RealtekBTPatcher.kext`

### 5.1 Hook 路径

`RealtekBTPatcher` 是 Lilu 插件，核心 hook：

1. **`IOUSBHostDevice::deviceRequest`** —— HCI command 路径，用于观察 `LE Read Remote Features` 命令。
2. **三条 `IOUSBHostIOSource::io` 重载** —— Tahoe 必须全部覆盖：
   ```text
   __ZN17IOUSBHostIOSource2ioEP18IOMemoryDescriptorjP19IOUSBHostCompletionj
   __ZN17IOUSBHostIOSource2ioEP18IOMemoryDescriptorjRjj
   __ZN17IOUSBHostIOSource2ioEP18IOMemoryDescriptorjP26IOUSBHostBundledCompletionj
   ```
3. **`IOUSBHostPipe::init` / `initWithDescriptors`** —— 捕获 interrupt-IN pipe。
4. 通过 vendor ID 过滤：Realtek `0x0BDA`、Intel `0x8087`。

> 三路 `IOSource::io` hook 来自 [if-lensman/Hackintosh-MacPro7-1-EFI](https://github.com/if-lensman/Hackintosh-MacPro7-1-EFI) 的 Tahoe 路由与去重思路；只 hook 一路会导致实际数据流绕过 patch。

### 5.2 注入策略

在**第一条真实的** `LE Read Remote Features Complete (0x04)` 之后，向同一个 transfer 追加两个事件：

```text
LE PHY Update Complete : 3E 06 0C 00 <h_lo> <h_hi> 02 02                    (8 bytes)
LE Data Length Change  : 3E 0B 07 <h_lo> <h_hi> 1B 00 48 01 1B 00 48 01     (13 bytes)
```

字段说明：

| 字节 | 含义 |
|:---|:---|
| `3E` | HCI Event Code：LE Meta Event |
| `06` / `0B` | 参数长度 |
| `0C` / `07` | LE subevent：PHY Update Complete / Data Length Change |
| `00` | 状态：Success |
| `<h_lo> <h_hi>` | 从真实 `0x04` 事件的 `data[2]` / `data[3]` 取得的 connection handle（小端） |
| `02 02` | TX / RX PHY = LE 2M |
| `1B 00` | Max TX octets = 27 |
| `48 01` | Max TX time = 328 |
| `1B 00` / `48 01` | Max RX octets / time，同上 |

> `0x0010` 就是本机成功日志中的 `IBTPHandle`；handle 必须从真实事件取，不能硬编码。

### 5.3 备用路径：清 EDL bit

如果注入路径没有命中，代码还会尝试清除 `LE Read Local Supported Features` 的 **LE Data Packet Length Extension** bit（bit 5），让 `bluetoothd` 跳过长度门控。这条路径是 fallback，不是本机成功的主路径。

### 5.4 与 Intel 方案的差异

| 项目 | Intel `IntelBTPatcher` | Realtek `RealtekBTPatcher` |
|:---|:---|:---|
| `0x04` 事件数量 | 可能重复 | 只有一条 |
| 去重 | 需要 | 不需要 |
| `IOUSBHostIOSource::io` | Tahoe 需三路 | Tahoe 需三路 |
| 事件注入 | 针对 Intel 事件序列 | 追加 `0x0C` + `0x07` |
| 备选方案 | — | 清 EDL bit 5（`0x2022` 同类思路） |

[flxxxxddd/SimpleMtkBtFw](https://github.com/flxxxxddd/SimpleMtkBtFw) 的 [PR #3](https://github.com/flxxxxddd/SimpleMtkBtFw/pull/3) 提供了 MT7922 使用 `HCI_LE_Set_Data_Length (0x2022)` 的替代方案；本机没有采用该路径，因为 Realtek 的问题在「事件不上报」而不是「长度未设置」。

---

## 6. 安装与 boot-args

### 6.1 建议的 Kernel → Add 顺序

```text
Lilu.kext
RealtekBluetoothFirmware.kext      (1.0.2，官方 release 二进制)
BlueToolFixup.kext                 (2.7.3)
RealtekBTPatcher.kext
```

> 这是本机使用的顺序，目的是让 Lilu 先加载、固件加载器先于协议栈补丁、`BlueToolFixup` 先于 Realtek 专用 patcher。实际 config 中以 `BundlePath` 与依赖关系为准。

### 6.2 boot-args

| 参数 | 作用 |
|:---|:---|
| `-ibtpwrap` | **启用事件改写**；不加时只安装 hook、发布计数器，不修改任何 buffer |
| `-ibtcompatdbg` | 输出 `RealtekBTPatcher` 诊断信息；稳定后可移除 |
| `-ibtcompatoff` | 完全禁用插件（排障 / 回滚用） |
| `-ibtcompatbeta` | 允许在 beta 系统上加载（按上游 README） |
| `-btlfxa2dpcheck` | `BlueToolFixup` 的 A2DP 检查路径，避免蓝牙音频通道过早断开；与 BLE HID 修复分开 |

本机最终使用的相关片段：

```text
... -btlfxa2dpcheck -ibtcompatdbg -ibtpwrap
```

> 不要把 `-ibtpwrap` 与 `-ibtcompatoff` 同时使用；后者优先级更高，插件会被禁用。

### 6.3 安全的回滚方式

1. 保留一个不含 `RealtekBTPatcher.kext` 或 `-ibtpwrap` 的备用 EFI；
2. 启动时加 `-ibtcompatoff` 验证是否恢复原状；
3. 若出现 kernel panic，移除 `RealtekBTPatcher.kext` 并重置 NVRAM。

---

## 7. 验证与观测

### 7.1 IORegistry 属性

`RealtekBTPatcher` 在蓝牙 USB 设备上发布以下属性：

| 属性 | 含义 | 本机成功值 |
|:---|:---|:---|
| `IBTPRouteErr` | 每个符号的 `routeMultiple` 错误码，0 = OK | `dev=0 async=0 iosC=0 iosR=0 iosB=0 initCfg=0 initPipe=0` |
| `IBTPPipeCaptured` | 是否捕获 interrupt pipe | `Y` |
| `IBTPPipeAddr` | interrupt pipe 内核地址 | — |
| `IBTPIOSourceHits` | 三路 `IOSource::io` 命中次数 | `1374` |
| `IBTPWrapEnabled` | `-ibtpwrap` 是否生效 | `Y` |
| `IBTPReqLRF` | 见到的 `LE Read Remote Features` 命令数 | `1` |
| `IBTPInjected` | 合成事件批次追加次数 | `1` |
| `IBTPHandle` | 最后一次注入使用的 connection handle | `0x0010` |
| `IBTPEDLClear` | 清除 EDL bit 的次数 | —（以 IORegistry 实际值为准） |

采集命令：

```bash
ioreg -lw0 | grep -oE '"IBTP[A-Za-z]+" = <[^>]*>'
```

### 7.2 系统日志

```bash
log show --last 5m --style compact | grep -E 'IBTP|ready-state|smpPairingCompleted|LE_READY'
```

成功时的关键片段（节选）：

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

### 7.3 配对结果

- 键盘配对成功，可正常输入；
- 没有 passkey 弹窗（BLE HID 的 Just Works / 绑定流程按设备能力执行）；
- `smpPairingCompleted status=0`；
- 本机已验证首次配对与输入；重新连接、删除配对后重配尚未单独记录，待验证。

---

## 8. 故障排查表

| 现象 | 可能原因 | 处理 |
|:---|:---|:---|
| `IBTPWrapEnabled=N` | 没有加 `-ibtpwrap` | 在 boot-args 加入后重启 |
| `IBTPRouteErr` 非 0 | macOS 版本变化导致符号解析失败 | 检查 Lilu 版本与 `RealtekBTPatcher` 是否匹配当前系统 |
| `IBTPPipeCaptured=N` | interrupt pipe 未捕获 | 确认 USB ID `0BDA:B00A`；检查 `IOUSBHostPipe::init` hook 是否命中 |
| `IBTPIOSourceHits=0` | 三路 `IOSource::io` 未命中 | 检查是否为 Tahoe；确认没有其他蓝牙 patch 抢占 |
| `IBTPReqLRF=0` | 没有观察到 `LE Read Remote Features` | 键盘可能未发起连接；先移除设备重新配对 |
| `IBTPInjected=0` 且 `IBTPWrapEnabled=Y` | 未捕获到第一条 `0x04` | 检查 filter、pipe 与系统日志；确认设备不是 Classic BR/EDR |
| `ready-state` 仍为 `LE_READY_WAIT_FOR_LENGTH` | 注入事件未被协议栈接受 | 检查 handle 是否正确；尝试 `-ibtcompatdbg` 观察 |
| 连接后又断开，`reason 722` | 事件仍缺失 / 状态机超时 | 确认 `IBTPInjected=1`；检查是否误用 `-ibtcompatoff` |
| 蓝牙音频异常 | A2DP 时钟 / 固件问题，与 BLE HID 不同 | 确认 `-btlfxa2dpcheck` 与 `BlueToolFixup` 2.7.3；查看音频日志 |

---

## 9. 已知限制

- 本修复只在 **macOS Tahoe 26.x + RTL8821CE + 本机 USB `0BDA:B00A`** 上验证；其他 Realtek 型号（RTL8822CE、RTL8852 等）不保证适用。
- 主要验证对象是 BLE HID 键盘；BLE 鼠标、耳机、多个 BLE 设备同时连接、睡眠唤醒后的重连尚未完整验证。
- Xbox 手柄使用 Classic BR/EDR，不经过 LE 事件路径，因此不需要本补丁；这不能证明其他 Classic 设备也无需补丁。
- 合成事件是对控制器缺失行为的兼容层，不是 Realtek 固件修复；系统或固件更新可能改变事件序列，导致 patch 失效。
- `-ibtpwrap` 会改写 HCI 事件 buffer；不使用时应移除该参数，默认模式只观测不改写。
- 若后续 `BlueToolFixup` 或 `RealtekBluetoothFirmware` 升级，需重新验证版本组合；本机固定为 `2.7.3` + `1.0.2`。

---

## 10. 参考资料

完整致谢与所有外部链接见 [主 README 的 Credits 章节](../README.md#credits--致谢)。与本篇直接相关的核心资料：

- [OpenIntelWireless/IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware)（zxystd 原版 `IntelBTPatcher`）
  - [issue #514](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/514)
  - [PR #515](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/pull/515)
  - [issue #497](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/497)
- [if-lensman/Hackintosh-MacPro7-1-EFI](https://github.com/if-lensman/Hackintosh-MacPro7-1-EFI)
- [thegwchr/RealtekBluetoothFirmware](https://github.com/thegwchr/RealtekBluetoothFirmware)
- [acidanthera/BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM)（`BlueToolFixup`）
- [soyeoning94/hackintosh-logitech-k251-bluetooth-pairing-fix](https://github.com/soyeoning94/hackintosh-logitech-k251-bluetooth-pairing-fix)
- [flxxxxddd/SimpleMtkBtFw](https://github.com/flxxxxddd/SimpleMtkBtFw) / [PR #3](https://github.com/flxxxxddd/SimpleMtkBtFw/pull/3)
- [watchthelight-labs/WTLLegionBtFix](https://github.com/watchthelight-labs/WTLLegionBtFix)
- [Emilelu/RealtekBTPatcher](https://github.com/Emilelu/RealtekBTPatcher)
- [Emilelu/BlueToolFixup-Fix26](https://github.com/Emilelu/BlueToolFixup-Fix26)


---

## 9. macOS 26.7.3 升级后蓝牙键盘失效（2026-11-10）

**症状**：从 26.7.2 升级到 26.7.3 后，蓝牙键盘扫描得到但连不上，蓝牙反复开关。

### 两个独立问题

**问题 A：`bluetoothd` 反复崩溃（EXC_GUARD namespc 18）**

日志：
```
bluetoothd ... Full corpse enqueued for bluetoothd
bluetoothd | exception: EXC_GUARD, subtype GUARD_TYPE_USER, namespc 18, reason 2
```
崩溃地址对应 `bluetoothInternalControllerInfo` 的**长度校验**（必须恰好 14 字节）。

**原因**：升级后 NVRAM 里的 `bluetoothExternalDongleFailed` 被写成 `%01`（第三方 dongle 初始化失败），
导致 `bluetoothd` 拒绝使用控制器后反复崩溃。

**修复**：在 `config.plist → NVRAM → Add` 和 `Delete` 里都加入：
```
bluetoothExternalDongleFailed   = 00        (1 字节)
bluetoothInternalControllerInfo = 00 × 14   (14 字节)
```
`Delete` 列表保证每次开机强制重置，即使系统再次写坏也不怕。
修复后控制器恢复 `THIRD_PARTY_DONGLE`、`Firmware v61592`、`Address DC:F5:05:FF:E4:02`。

**问题 B：键盘仍连不上，卡 `LE_READY_WAIT_FOR_LENGTH`**

即使蓝牙稳定，纯 BLE 键盘连接后仍停在 `LE_READY_WAIT_FOR_LENGTH`，30 秒后 `reason 722` 断开。

**修复**：更新 `IntelBTPatcher.kext`（提交 `b16fe69`），关键逻辑调整为：
| 功能 | 触发条件 |
|---|---|
| 隐藏 LE Data Packet Length Extension 位 (bit5) | **无条件**（让 bluetoothd 能发起连接） |
| 注入 LE Data Length Change 事件 (0x07) | `-ibtpwrap` 控制 |
| 注入 LE PHY Update Complete 事件 (0x0C) | **彻底移除**（26.7.3 上会引起蓝牙软复位） |
| 原始 HCI 事件转发 | **始终保留**（避免 panic） |

> ⚠️ **之前的 bug**：曾在 `rewriteHciEvent` 开头写 `if(!g_wrapEnabled) return;`，
> 这会导致原始事件转发被中断、触发 kernel panic。已移除，只把「改写/注入」包进门控。

**验证**（实测配对成功）：
```
Data Length Changed ... ready-state: wait-for-length, phy-ready 0
Le topology ... ready-state: LE_READY_IDLE
Classic Pairing Agent successfully created
Accepting pairing request from "82F8798D"
Encryption is now enabled
smpPairingCompleted ... status=0
```
`IBTPInjected=31`、`IBTPHandle=0x0010` 确认注入使用真实连接句柄。

### 26.7.3 新增的 `bm3_usb` 模块

26.7.3 的 `bluetoothd` 新增 `bm3_usb` 模块（USB dongle 探测），它原生支持 0BDA:B00A：
```
[bm3_usb][Found_USB_Dongle] -- FoundBluetoothInterfaces() for deviceClass (224),
   idVendor = 0x0BDA, idProduct = 0xB00A returned true
[bm3_usb] setting gUseInternalController to FALSE
```
因此可以正常识别 Realtek 控制器为 `THIRD_PARTY_DONGLE`。这个模块本身不是问题。

### 修复后固化的配置

- `config.plist`（已脱敏），boot-args 含 `-ibtpwrap`
- `config.plist.KNOWNGOOD-BT-BLEHID-FIXED`（已验证可用快照）
- `Kexts/IntelBTPatcher.kext`（`b16fe69` 构建，sha256 `89f9b0c8...`）

### 关键诊断命令

```bash
# 蓝牙控制器状态
system_profiler SPBluetoothDataType

# 检查 bluetoothd 是否崩溃
pgrep -x bluetoothd
log show --last 5m --predicate 'process == "kernel"' --style compact | grep -c 'corpse enqueued for bluetoothd'

# 补丁注入计数（IORegistry）
ioreg -lw0 | grep -oE '"IBTP(Injected|Handle|EDLClear|WrapEnabled)" = <[^>]*>'

# LE 连接状态
log show --last 2m --style compact --predicate 'subsystem == "com.apple.bluetooth"' | grep -E 'Le topology|ready-state'
```

### 排查要点
1. 第一步确认 NVRAM 蓝牙变量是否被写坏（问题 A）
2. 蓝牙稳定但仍连不上 → 看 `IBTPInjected` 是否在连接时增长（问题 B 的补丁是否触发）
3. 系统能连接但卡 `LE_READY_WAIT_FOR_LENGTH` → 确认注入 Data Length Change 是否执行
