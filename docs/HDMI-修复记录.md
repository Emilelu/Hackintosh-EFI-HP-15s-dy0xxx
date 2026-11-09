> [English](./en/HDMI-Fix-Notes.md)

# HDMI 修复记录：从 VBT 到 `con1 busId = 1`

> 返回 [主 README](../README.md)｜机型：HP 15s-dy0xxx / i7-8565U / macOS Tahoe 26.7.2

本文记录本机 Intel UHD Graphics 620（原生 `0x3EA0`，spoof `0x3EA5`）在 macOS Tahoe 下点亮 HDMI 外接显示器的完整过程。结论先行：

- 物理接口：**HDMI = DDI B，原生 TMDS，无 LSPCON**
- WhateverGreen 连接器：**`con1 busId = 1`**
- 必须成套使用：`igfxonln=1 igfxonlnfbs=0x0100 igfxfcms=1 igfxfcmsfbs=0x0100`
- 实测：**2560×1440 @ 120Hz**
- 4K60 不可用：本机无 LSPCON，且 Tahoe 原生 HDMI 分支存在 300MHz 限制；4K30 亦受模式表限制

---

## 1. 环境与症状

| 项目 | 值 |
|:---|:---|
| 机型 | HP 15s-dy0xxx |
| CPU | Intel Core i7-8565U（Whiskey Lake，4C8T） |
| 核显 | Intel UHD Graphics 620，原生 `0x3EA0` |
| macOS | Tahoe 26.7.2（Darwin 25.6.0 / `25G332`） |
| 引导 | OpenCore 1.x（占位） |
| SMBIOS | `MacBookPro15,2` |
| 内屏 | 1920×1080 @ 60Hz |
| 外屏 | HDMI，实测 2560×1440 @ 120Hz |

**症状**：内屏正常、核显加速正常，但 HDMI 接显示器无信号。系统能检测到热插拔，AppleIntelFramebuffer 却读不到有效 EDID，`fIsHDMI` 与 `newOnline` 均不成立。

典型的失败路径是：

1. 只把 con1 的 `type` 改成 HDMI（`0x0800`），保留从 CFL 模板抄来的 `busId=5`；
2. 或者虽然改成 `busId=1`，但没有同时启用 force-online / force-modeset；
3. 系统日志中没有 `fIsHDMI=1`、`newOnline=1`，或者 `getDisplayStatus=0`，最终不触发 modeset。

> 这不是「HDMI 口坏了」或「UHD 620 不支持外接」的问题，而是 **连接器映射 + 显示在线状态** 两个条件没有同时满足。

---

## 2. 先确认物理端口：解析 `i915_vbt.bin`

### 2.1 为什么必须看 VBT

macOS 的 AppleIntelFramebuffer 不会告诉我们「主板上的 HDMI 口接的是 DDI B 还是 DDI C」。同一代 UHD 620 笔记本的 VBT 可能完全不同：有的 HDMI 在 DDI B，有的在 DDI C，有的经过 LSPCON。只靠试 `busId` 是盲猜。

VBT（Video BIOS Table）由 BIOS 提供，包含每个 child device 的：

- `device_type`：HDMI / DP / eDP / LVDS
- `dvo_port`：DDI 端口（HDMIB、HDMIC 等）
- `lspcon`：是否经过 LSPCON
- `ddc_pin`：DDC / GMBUS pin

Linux i915 驱动会解析 VBT，并在 debugfs 中提供原始二进制。

### 2.2 提取 VBT

在 Ubuntu Live（或任意安装了较新内核的 Linux）中执行：

```bash
sudo mount -t debugfs none /sys/kernel/debug 2>/dev/null || true

# 路径可能是 /sys/kernel/debug/dri/0/i915_vbt，
# 也可能是带 PCI 地址的目录；以实际存在者为准。
sudo cp /sys/kernel/debug/dri/0000:00:02.0/i915_vbt /tmp/i915_vbt.bin
sudo chmod a+r /tmp/i915_vbt.bin
```

如果提示找不到文件，先确认 i915 已加载：

```bash
ls /sys/kernel/debug/dri/
lspci -nnk | grep -A3 VGA
```

### 2.3 解码 VBT

安装 [intel-gpu-tools](https://gitlab.freedesktop.org/drm/igt-gpu-tools)：

```bash
sudo apt update
sudo apt install intel-gpu-tools
```

查看 Block 2（General Definitions）的完整内容：

```bash
intel_vbt_decode --file=/tmp/i915_vbt.bin --block=2 --hexdump
```

也可以先整体解码，确认 BDB 版本、child 数量与端口摘要：

```bash
intel_vbt_decode --file=/tmp/i915_vbt.bin --hexdump
```

> `intel_vbt_decode` 的官方用法是 `--file=FILE`；部分旧版本也接受位置参数，若命令报错请以 `intel_vbt_decode --help` 为准。

### 2.4 `struct bdb_general_definitions` 的 5 字节头

Linux 的 [`intel_vbt_defs.h`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_vbt_defs.h) 定义了：

```c
struct bdb_general_definitions {
    u8 crt_ddc_gmbus_pin;   /* 1 byte */
    u8 dpms_non_acpi:1;     /* 1 byte 的 bitfield */
    u8 skip_boot_crt_detect:1;
    u8 dpms_aim:1;
    u8 rsvd1:5;
    u8 boot_display[2];     /* 2 bytes */
    u8 child_dev_size;      /* 1 byte */
    u8 devices[];           /* child_device_config 数组 */
} __packed;
```

因此头部固定为 **5 字节**：

```text
offset 0x00: crt_ddc_gmbus_pin     (1 B)
offset 0x01: DPMS / reserved       (1 B)
offset 0x02: boot_display[0..1]    (2 B)
offset 0x04: child_dev_size        (1 B)
offset 0x05: child[0] 开始
```

Block 2 的 `block_size` 不包含 3 字节 BDB block header（block id + 2 字节 size）。所以：

```text
child 数量 = (block_size - 5) / child_dev_size
```

本机信息：

| 字段 | 值 | 含义 |
|:---|:---|:---|
| BDB version | **212** | 属于 196–215 区间 |
| `child_dev_size` | **38**（`0x26`） | Linux `child_device_expected_size()` 对 BDB 196–215 返回 38 |
| 头部大小 | 5 字节 | `sizeof(struct bdb_general_definitions)` |
| Block 2 `block_size` | **309** | 本机 VBT 实测 |
| child 数 | **8** | `(309 - 5) / 38 = 8`，余数 0 |

> 注意：数组中可能存在 `device_type == 0` 的空条目；Linux `parse_general_definitions()` 会跳过它们，但计算 child 总数时仍然计入。不要用「看到几个非空条目」反推 block_size。

### 2.5 本机 `child[1]` 的关键字段

本机 VBT 的 HDMI 位于 **`child[1]`**，字段如下：

| 字段 | VBT 值 | Linux 定义 / 解释 |
|:---|:---:|:---|
| `device_type` | `0x60D2` | `DEVICE_TYPE_HDMI`，即 HDMI/DVI TMDS 输出 |
| `dvo_port` | `0x01` | `DVO_PORT_HDMIB` → **DDI B** |
| `lspcon` | `0` | **无 LSPCON**，原生 TMDS |
| `ddc_pin` | `0x01` | `DDC_BUS_DDI_B` → GMBUS pin 1 |
| `i2c_pin` | 与 DDC 配合 | 不作为主判断依据 |

对应的 Linux 常量来自 [`intel_vbt_defs.h`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_vbt_defs.h)：

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

**结论：物理 HDMI = DDI B + 原生 TMDS + DDC pin 1。**

---

## 3. VBT 如何映射到 WhateverGreen

### 3.1 `ConnectorInfo` 的字段

WhateverGreen 的 [`kern_fb.hpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_fb.hpp) 定义了：

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

`busId` 的注释明确说明它是 **GMBUS ID**，不是 DDI 字母，也不是 macOS 的 framebuffer index。某些老平台/老教程会把 `GMBUS_PIN_DPB` 写成 `5`，但本机是 CNP 世代，映射不同。

### 3.2 CNP 世代的 DDC 映射

Linux [`intel_hdmi.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_hdmi.c) 中的 `cnp_encoder_to_ddc_pin()`：

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

[`intel_gmbus.h`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_gmbus.h) 中：

```c
#define GMBUS_PIN_1   1
#define GMBUS_PIN_2   2
#define GMBUS_PIN_4   4
```

因此，本机：

```text
VBT ddc_pin = 0x01
    -> DDC_BUS_DDI_B
    -> PORT_B
    -> CNP: GMBUS_PIN_1
    -> WhateverGreen busId = 1
```

### 3.3 三重证据

| 证据 | 内容 | 说明 |
|:---|:---|:---|
| 1. VBT 实测 | `ddc_pin=0x01`，`dvo_port=0x01`，`lspcon=0` | 物理端口为 DDI B，DDC 为 pin 1 |
| 2. 同系列 EFI | [shammu2bd/HP-15s-du1xxx-Hackintosh](https://github.com/shammu2bd/HP-15s-du1xxx-Hackintosh) 的 [`config.plist`](https://github.com/shammu2bd/HP-15s-du1xxx-Hackintosh/blob/main/EFI/OC/config.plist) 使用 `framebuffer-con1-busid=1` | HP 15s 同系列、同代 UHD 620 的独立实现 |
| 3. 上游讨论 | [acidanthera/bugtracker #420](https://github.com/acidanthera/bugtracker/issues/420#issuecomment-511499674) 评论明确写「Changing the `bus-id` of the HDMI connector to one of the allowed `bus-id`s for HDMI (`1` works for me)」 | 说明 `1` 是 HDMI 的合法 busId 之一 |

补充：真正让这套映射成立的机制是 CNP 的 `PORT_B -> GMBUS_PIN_1`，见 [Linux `intel_hdmi.c`](https://github.com/torvalds/linux/blob/master/drivers/gpu/drm/i915/display/intel_hdmi.c)。`busId=1` 不是玄学，也不是「DDI B 的编号」。

---

## 4. 最终配置

### 4.1 三个连接器一次写入

本机最终使用一个 36 字节的 `framebuffer-con0-alldata`，从 con0 开始写满三个连接器：

| 连接器 | index | busId | pipe | pad | type | flags | 目的 |
|:---:|---:|---:|---:|---:|:---:|:---:|:---|
| con0 | 0 | 0 | 8 | 0 | `0x0002` (LVDS) | `0x0098` | eDP 内屏 |
| con1 | 1 | **1** | 9 | 0 | `0x0800` (HDMI) | `0x01C7` | HDMI / DDI B |
| con2 | 2 | 4 | 10 | 0 | `0x0400` (DP) | `0x01C7` | DP @ DDI C |

十六进制原文：

```text
00 00 08 00 02 00 00 00 98 00 00 00
01 01 09 00 00 08 00 00 C7 01 00 00
02 04 0A 00 00 04 00 00 C7 01 00 00
```

Base64：

```text
AAAIAAIAAACYAAAAAQEJAAAIAADHAQAAAgQKAAAEAADHAQAA
```

可直接放入 OpenCore：

```xml
<key>framebuffer-con0-alldata</key>
<data>AAAIAAIAAACYAAAAAQEJAAAIAADHAQAAAgQKAAAEAADHAQAA</data>
```

> `framebuffer-con0-alldata` 是从 con0 起的整张连接器表，不要把它误当成「只改 con0」。`con1` 的 busId 在偏移 `0x0D`（第 14 字节）处，值为 `0x01`。

### 4.2 如果只想做最小化 patch

等价地，也可以单独写 con1：

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

> 两种写法二选一，不要同时混用同一字段。本机最终使用 `con0-alldata`，便于把三个连接器的 pipe / type 固定在一张表里。

### 4.3 必须成套的 boot-args

```text
igfxonln=1 igfxonlnfbs=0x0100 igfxfcms=1 igfxfcmsfbs=0x0100
```

| 参数 | 作用 | 缺少时的现象 |
|:---|:---|:---|
| `igfxonln=1` | 强制显示在线 | 显示器被判定离线，`newOnline=0` |
| `igfxonlnfbs=0x0100` | 将 force-online 限定到 FB1（`0x0100` 是 FB1 位掩码） | FB1 不生效 |
| `igfxfcms=1` | 强制完成 modeset | 即使在线也不点亮 |
| `igfxfcmsfbs=0x0100` | 将 force-modeset 限定到 FB1 | FB1 不触发 modeset |

这四项不是「可选优化」，而是本机 HDMI 点亮的必要组成。缺少该组时，`getDisplayStatus` 返回 `0`，后续不会进入 modeset。

### 4.4 EDID override

```xml
<key>AAPL01,override-no-connect</key>
<data>（此处填入显示器真实 128 字节 EDID 的 Base64）</data>
```

- 键名必须是 **`AAPL01`**：本机 HDMI 对应 FB1；`AAPL00` 会作用于内屏 FB0。
- 只支持 **128 字节**：见 [acidanthera/bugtracker #2374](https://github.com/acidanthera/bugtracker/issues/2374)。256 字节 EDID 或 DisplayID 扩展块不会被读取。
- 本机 EDID 的首选 DTD：**3840×2160 @ 30Hz，297MHz**。
- EDID 含显示器序列号等隐私信息，**本文不公开原始字节**；请从自己的显示器提取，不要复制他人的 EDID。

提取 128 字节 EDID 的示例（Linux）：

```bash
# HDMI 接口名可能是 card0-HDMI-A-1 / card1-HDMI-A-1，以实际为准
cat /sys/class/drm/card0-HDMI-A-1/edid > /tmp/hdmi-edid.bin
ls -l /tmp/hdmi-edid.bin          # 通常是 128 或 256 字节
edid-decode /tmp/hdmi-edid.bin    # 确认首选 DTD 与校验和
head -c 128 /tmp/hdmi-edid.bin | base64 -w0
```

> 如果 `edid-decode` 显示 256 字节 / 含 Extension block，只取前 128 字节。多出来的部分 WEG 不会读取，不能靠它解决 4K 模式问题。

---

## 5. 日志证据

### 5.1 成功点亮时的关键行

`busId=1` + 四个 force 参数 + EDID override 同时生效后，首次出现：

```text
[Getonline_info] fIsHDMI=1, fEnableAudio=1, newOnline=1, newPortType=1
```

字段含义：

| 字段 | 值 | 含义 |
|:---|:---:|:---|
| `fIsHDMI` | 1 | 驱动将该连接器识别为 HDMI/TMDS |
| `fEnableAudio` | 1 | HDMI 音频路径被标记为启用 |
| `newOnline` | 1 | 显示器进入在线状态 |
| `newPortType` | 1 | 端口类型完成切换 |

### 5.2 缺少 force 参数时

如果只改连接器、没有成套的 `igfxonln` / `igfxfcms`，会出现：

```text
getDisplayStatus=0
```

`getDisplayStatus` 为 0 时，后续 `setDisplayMode` 不会被触发，HDMI 保持黑屏。这不是 EDID 的问题，而是显示状态机没有进入可 modeset 的状态。

### 5.3 如何采集这些日志

WhateverGreen 的 `kern_igfx_debug.cpp` 提供了 `getOnlineInfo`、`getDisplayStatus`、`connectionProbe` 等 DEBUG 追踪点：

- 使用 **DEBUG 版 WhateverGreen**；
- 临时加入 `-igfxfbdbg`；
- 用 `log show` 或 `dmesg` 过滤 `AppleIntelFramebuffer` / `igfx`。

```bash
log show --last 5m --style compact | grep -E 'Getonline_info|getDisplayStatus|fIsHDMI|igfx'
```

> `kern_igfx_debug.cpp` 是 DEBUG-only 的 framebuffer 追踪代码；RELEASE 版 WhateverGreen 不包含这些日志。

---

## 6. 失败尝试清单

下表是本机排查过程中实际遇到或验证过的失败路径。它们能解释「为什么看起来都改对了，HDMI 还是不亮」。

| # | 尝试 | 结果 | 根因 / 结论 |
|:---:|:---|:---|:---|
| 1 | con1 `type=0x0800`，沿用 `busId=5` | 无信号；无 `fIsHDMI=1` | `5` 是部分老平台 `GMBUS_PIN_DPB` 的编号，本机 CNP 的 DDI B 是 `GMBUS_PIN_1` |
| 2 | 改 `busId=1`，但没有 force-online / force-modeset | 连接器状态不完整，`getDisplayStatus=0` | 显示状态机未进入 modeset |
| 3 | 只加 `igfxonln`，缺少 `igfxfcms` | 仍无法稳定点亮 | force online 与 force modeset 是两条独立路径，必须成套 |
| 4 | 给 HDMI con1 加 `flags=0x0008`（`CNConnectorAlwaysConnected`） | 黑屏，内屏可能被顶掉 | `CNConnectorAlwaysConnected` 会把连接器当作内屏/LVDS，`port-number=0` 与内屏冲突 |
| 5 | 使用 `-igfxmpc=600` | 参数值被忽略，4K 仍未出现 | `-igfxmpc` 是裸标志，不解析 `=600`；内部默认 675MHz |
| 6 | 注入 256 字节 EDID / DisplayID 扩展 | 只使用前 128 字节 | `override-no-connect` 只支持 128 字节，见 #2374 |
| 7 | 试图通过 LSPCON 路径点亮 4K60 | 本机无 LSPCON，无法启用 | VBT `lspcon=0`，HDMI 是原生 TMDS |
| 8 | 将 con1 `flags` 改为 `0x1CF` | 黑屏 / 无信号 | `flags` 不是“位越多越好”；`0x1CF` 破坏原生 `0x01C7` 的语义，保持 `0x01C7` |

### 6.1 EDID 变体实验（7 种，全部失败）

排查过程中累计构造并测试了 **7 种 128 字节 EDID 变体**，覆盖不同的首选 DTD（4K30 / 4K25 / 1440p120）、CEA 扩展块 / SVD 裁剪、厂商与产品标识替换、校验和修正等组合。结果一致：

- 系统能把显示器识别为有效 EDID，但 `AppleIntelFramebuffer` 的模式列表没有新增 3840×2160；
- 部分变体因 DTD 不合规或与真实显示器能力不匹配，直接无信号；
- 由此确认：**override EDID 不是 4K 模式表的来源**，继续堆 EDID 变体没有收益。

> 7 种变体的具体字节不再逐一保留，避免被误当成“可用的 4K EDID”复制。EDID 必须来自你自己的显示器。

### 不要做的事

1. **不要给 HDMI 连接器加 `CNConnectorAlwaysConnected (flags 0x0008)`**。该标志专用于内屏/LVDS；external 连接器加上后，port-number 计算会与内屏冲突，直接黑屏。`con0` 的 `flags=0x0098` 含这个位，是内屏专用，不要复制到 con1。
2. **不要写 `-igfxmpc=600`**。WhateverGreen 只检查 `-igfxmpc` 是否存在，不解析等号后的值。需要改变上限时，只能用 WEG 支持的属性/代码路径，且 675MHz 也不等于 4K60。
3. **不要做 busId 盲扫**。先解 VBT，确认 `ddc_pin` 与 DDI 映射，再定 `busId`。盲试 0–6 可能把内屏顶掉或破坏音频路径。
4. **不要复制他人的 EDID**。EDID 含厂商、型号、序列号与 DTD；错误 EDID 会让 macOS 选择错误模式。
5. **不要同时混用 `framebuffer-con0-alldata` 与 `framebuffer-con1-*`**。同一连接器被两套键重复覆盖时，最终值取决于 WEG 处理顺序，极难排查。
6. **不要把 `busId` 当作 DDI 字母**。`DDI B` 不代表 `busId=2`，`DDI C` 也不代表 `busId=3`；它们是完全不同的编号体系。

---

## 7. 复现步骤（检查清单）

- [ ] BIOS：Secure Boot 关闭、DVMT ≥ 64MB、CFG Lock 关闭（或 quirk 规避）
- [ ] 从 Ubuntu Live 提取 `i915_vbt.bin`
- [ ] `intel_vbt_decode --file=i915_vbt.bin --block=2 --hexdump`
- [ ] 确认 `child_dev_size=38`，按 `(block_size-5)/38` 计算 child 数
- [ ] 确认 HDMI child 的 `device_type=0x60D2`、`dvo_port=1`、`lspcon=0`、`ddc_pin=1`
- [ ] 在 `config.plist` 写入 36 字节 `framebuffer-con0-alldata`（或等价 con1 patch）
- [ ] 加入 `igfxonln=1 igfxonlnfbs=0x0100 igfxfcms=1 igfxfcmsfbs=0x0100`
- [ ] 提取显示器自己的 128 字节 EDID，写入 `AAPL01,override-no-connect`
- [ ] 重启，确认日志出现 `[Getonline_info] fIsHDMI=1, fEnableAudio=1, newOnline=1, newPortType=1`
- [ ] 在系统设置中确认 2560×1440 @ 120Hz
- [ ] 保留可启动的备用 EFI 后再调整其他参数

---

## 8. 为什么 4K 仍然不可用

这是本文最重要的「已知限制」部分。**不要因为 2560×1440@120 成功，就认为 4K60 只差一个 busId。**

### 8.1 `fMaxPixelClock` 的 300MHz 硬编码

在 macOS Tahoe 的 `AppleIntelFramebuffer::getOnlineInfo` 的 native HDMI 分支中，存在如下赋值：

```asm
movq $0x11E1A300, 0x4368(%rbx)
```

`0x11E1A300 = 300,000,000`，即 **300MHz**。

[b00t0x/WhateverXe issue #41](https://github.com/b00t0x/WhateverXe/issues/41) 在 ADL-S / UHD 770 / Tahoe 25G229 上给出了逐字节分析：该立即数把 `fMaxPixelClock` 限制在 300MHz；本机日志的同一组 `Getonline_info` 字段也给出 `fMaxPixelClock=300000000`。4K60 所需的约 594MHz 被排除，4K30（297MHz）虽然低于限制，但模式表是否能列出 4K30 还取决于 AppleIntelFramebuffer 提供的模式列表。

### 8.2 `-igfxmpc` 做了什么、没做什么

WhateverGreen 的 [`kern_igfx_clock.cpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_igfx_clock.cpp) 中 `MaxPixelClockOverride` 会 hook `AppleIntelFramebuffer::connectionProbe`，改写 `IOFBTimingRange` 的 `maxPixelClock`。其默认值在 [`kern_igfx.hpp`](https://github.com/acidanthera/WhateverGreen/blob/master/WhateverGreen/kern_igfx.hpp) 中为：

```cpp
uint64_t maxPixelClockFrequency = 675000000; // 675MHz
```

但：

- `-igfxmpc` 只是**启用开关**，不解析 `=600`；
- 675MHz 只是 WEG 的 `IODisplayTimingRangeV1.maxPixelClock` 上限；`-igfxmpc` **只改这个上限，不会向 `AppleIntelFramebuffer` 的模式表添加任何模式**；
- Tahoe 的 native HDMI 分支仍可能把 `fMaxPixelClock` 写回 300MHz；
- `FBBoostPixelFrequencyLimit(0x100)` 在 CFL 上**无效**：它不会改变 CFL 原生 HDMI 的 300MHz 限制，也不会改变模式表；
- [acidanthera/bugtracker #1377](https://github.com/acidanthera/bugtracker/issues/1377) 也记录了「设置 `enable-max-pixel-clock-override` 后，4K 面板仍只能 4K@48」的案例，说明单靠 max pixel clock 不够。

这也解释了一个容易困惑的现象：本机 2560×1440@120 的 pixel clock 约为 **483.4MHz**，仍能工作，因为 `-igfxmpc` 把 `IOFBTimingRange.maxPixelClock` 提高到 **675MHz**；但 `fMaxPixelClock` 字段仍是 300MHz，且 4K 模式没有进入模式表。两者并不矛盾。

### 8.3 模式表不读取 override EDID 的 DTD

`AAPL01,override-no-connect` 的作用是**替换 EDID 数据**，不是向 AppleIntelFramebuffer 注入新 mode。它的行为是：

1. 驱动读取连接的显示器 EDID 时，用 override 数据代替；
2. 但最终可选模式列表仍由 `AppleIntelFramebuffer` 的内部表/驱动逻辑提供；
3. 因此 EDID 中的 4K30 DTD **不会自动变成系统模式列表中的 4K30 选项**。

[b00t0x/WhateverXe issue #41](https://github.com/b00t0x/WhateverXe/issues/41) 也明确指出：CoreGraphics 模式枚举包含真正的 1080p60、720p60，但 HiDPI 4K 使用的 3840×2160 raster 仍会被 300MHz 限制拦住。这进一步说明「EDID 有 4K DTD」不等于「系统会提供 4K 模式」。

### 8.4 本机没有 LSPCON

VBT 明确给出：

```text
lspcon = 0
```

LSPCON 的作用是把 DisplayPort 转成 HDMI 2.0，以突破原生 TMDS 的 300MHz 限制。本机 HDMI 是 **原生 TMDS**，没有 LSPCON 芯片，因此：

- 4K60（约 594MHz）不可行；
- 4K30（297MHz）从时钟角度理论可行，但受 8.3 的模式表限制；
- 通过修改 EDID、加 `-igfxmpc`、换 `busId` 都无法变出 LSPCON。

### 8.5 结论

```text
2560×1440 @ 120Hz  -> 已验证可用
3840×2160 @ 30Hz   -> 理论可行，但系统模式表未提供；未实现
3840×2160 @ 60Hz   -> 本机无 LSPCON + 300MHz 限制；不可实现
```

如果你的显示器支持 2560×1440@120，这是本机目前最合适的 HDMI 工作模式。不要为了追求 4K 去改 `CNConnectorAlwaysConnected` 或扫 `busId`，那些路径不会绕过 300MHz 限制，反而会引入黑屏。

### 8.6 待验证的 4K30 方案

4K30 的 297MHz 低于 300MHz 限制，**从时钟角度理论可行**；当前障碍主要是模式表与 stolen memory 配置。以下方案参考 Xiaomi Pro（UHD 620）的 4K30 成功案例，属于 **待验证**，尚未在本机确认成功：

1. **BIOS**：`DVMT Pre-Allocated = 64MB`（若 BIOS 可选更高，96MB / 128MB 更稳）。
2. **删除 `framebuffer-fbmem` 与 `framebuffer-stolenmem`**：这两个显存 patch 会把可用 stolen memory 压得过低，导致 AppleIntelFramebuffer 不列出 4K 模式。若还存在 `framebuffer-unifiedmem`，一并评估是否删除。
3. **boot-args 增加 `-igfxhdmidivs`**：WhateverGreen 的 HDMI divider 修复，官方说明对 HDMI 1.4 接口上的 2K / 4K 模式有帮助。
4. **EDID override 做 A/B 测试**：保留正确的 128 字节 EDID 或暂时移除，观察 AppleIntelFramebuffer 是否自行列出 4K30 模式；关键不是 EDID 里有没有 4K DTD，而是驱动模式表是否生成 4K30。
5. **验证**：重启后在「系统设置 → 显示器」中检查是否有 `3840×2160 @ 30Hz`；同时用日志确认 `[Modeset] ... 297000000 Hz` 或接近值。

风险与注意事项：

- 若实际 DVMT 只有 32MB，删除 `framebuffer-stolenmem` 可能导致开机黑屏 / panic；请先确认 BIOS 已设为 64MB，或只加回一个 32MB 的 `framebuffer-stolenmem` 兜底，不要同时加 `framebuffer-fbmem` / `framebuffer-unifiedmem`。
- `-igfxhdmidivs` 只影响 modeset 阶段的时钟分频，**不会生成模式**；没有 4K30 模式时加它也不会“变出”4K。
- 该方案与现有 1080p / 1440p 配置存在取舍，务必保留可启动的备用 EFI，一次只改一项并记录结果。
- 参考案例：[daliansky/XiaoMi-Pro-Hackintosh issue #117](https://github.com/daliansky/XiaoMi-Pro-Hackintosh/issues/117)。本机是否适用取决于 BIOS / DVMT 与 VBT，**不保证成功**。

---

## 9. 附录：按字节解码 `framebuffer-con0-alldata`

```text
offset  bytes                       field
0x00    00                          con0.index = 0
0x01    00                          con0.busId = 0
0x02    08                          con0.pipe = 8
0x03    00                          con0.pad = 0
0x04    02 00 00 00                 con0.type = 0x0002 (LVDS)
0x08    98 00 00 00                 con0.flags = 0x0098
0x0C    01                          con1.index = 1
0x0D    01                          con1.busId = 1  <-- 关键
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

所有多字节字段均为 little-endian。

---

## 10. 参考资料

完整致谢与全部外部链接见 [主 README 的 Credits 章节](../README.md#credits--致谢)。与本篇直接相关的核心资料：

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
