> [English](./en/BIOS-Settings.md)

# BIOS 设置建议：HP 15s-dy0xxx

> 返回 [主 README](../README.md)｜适用：HP 15s-dy0xxx / i7-8565U / macOS Tahoe 26.7.2

本文给出本机型安装与运行 macOS Tahoe 所需的 BIOS 设置基线。HP 15s 系列不同批次、不同 BIOS 版本暴露的选项名称可能不同；**下表以「功能含义」为准，不保证你的 BIOS 中每一项都可见**。

> ⚠️ 修改 BIOS 前请记录原始值并准备 Windows / Linux 恢复盘。不要刷入来源不明的修改版 BIOS，也不要照抄其他机型的 `setup_var` 偏移。

---

## 1. 推荐设置总表

| 设置项（常见名称） | 推荐值 | 必要性 | 说明 |
|:---|:---|:---:|:---|
| Secure Boot | **Disabled** | 必需 | OpenCore 与多数第三方 kext 未签名；开启后无法引导 |
| TPM Device / PTT | **Hidden / Disabled** | 建议 | macOS 不需要 TPM；双系统 Windows 11 若依赖 TPM，可保留 Enabled，macOS 一般仍可启动，但建议安装阶段先关闭 |
| Intel SGX | **Disabled** | 建议 | macOS 无 SGX 驱动，关闭可减少无谓的 ACPI / 资源暴露 |
| Intel Virtualization Technology (VT-x) | **Enabled** | 建议 | 提升虚拟机与部分内核功能兼容性；不使用虚拟机也可开启 |
| VT-d | **Disabled** | 强烈建议 | macOS 对 IOMMU 支持有限；若 BIOS 无此选项，在 OpenCore 中启用 `DisableIoMapper` |
| CFG Lock | **Disabled** | 强烈建议 | 避免内核写 MSR 0xE2 失败；若隐藏，使用 `AppleXcpmCfgLock` 规避 |
| DVMT Pre-Allocated | **64MB** | 强烈建议 | UHD 620 framebuffer / HDMI 的 stolen memory 基线；低于 32MB 容易出现花屏、黑屏或 HDMI 不亮 |
| DVMT Total Gfx Mem | **256MB / Max** | 建议 | 若 BIOS 有此项，给核显更大的动态显存上限 |
| Graphics / Switchable Graphics | **保持出厂默认（Hybrid / Switchable）** | 建议 | 不要强制「Discrete Only」，否则核显被关闭，macOS 无法驱动 |
| Fast Boot | **Disabled** | 必需 | 否则可能跳过 USB / OpenCore 启动项 |
| USB Boot | **Enabled** | 必需 | 允许从安装 U 盘启动 |
| Boot Mode / Legacy Support | **UEFI Only / Legacy Disabled** | 必需 | macOS 只支持 UEFI 引导 |
| CSM（Compatibility Support Module） | **Disabled** | 必需 | CSM 会引入 Legacy 启动路径，可能导致 OpenCore 无法接管引导；与 UEFI Only 同时设置 |
| SATA Controller Mode | **AHCI** | 建议 | 若机器有 SATA 盘 / 光驱位，AHCI 是 macOS 可识别模式；NVMe 不受此项影响 |
| Above 4G Decoding | **Enabled**（若可见） | 可选 | 改善大 BAR 设备映射；若启用后出现启动问题，恢复 Disabled 并保留 `ResizeAppleGpuBars=-1` |
| Wake on LAN | **Disabled** | 可选 | 避免睡眠时被网络唤醒；对安装不关键 |
| BIOS Password | 记录好，不要遗忘 | 建议 | 部分 HP BIOS 设置密码后，BIOS 启动阶段会要求输入；忘记会导致无法改设置 |

---

## 2. 关键项详解

### 2.1 Secure Boot

- 设为 **Disabled**。OpenCore、WhateverGreen、AirPortRTW、RealtekBTPatcher 等均不是 Apple 签名组件。
- 若 BIOS 中有「Secure Boot Mode」，选择 **Setup Mode / Disabled**，不要选 Standard / Custom。
- 安装完成后也不要重新开启；Windows 双系统如需 Secure Boot，请使用单独启动项，不要用 OpenCore 的 Secure Boot 链。

### 2.2 CFG Lock

CFG Lock 是 Intel 平台对 MSR `0xE2`（IA32_PERF_CTL / C-state 相关）的写保护。macOS 的 XCPM 电源管理需要写该 MSR；未关闭时常见症状是：

- 启动卡住或 panic；
- CPU 电源管理不工作、频率异常；
- 睡眠唤醒失败。

优先在 BIOS 中关闭。若 BIOS 没有暴露该选项：

1. **不要**直接使用网上同型号的 `setup_var 0x?? 0x0` 偏移；不同 BIOS 版本偏移不同，写错可能变砖。
2. 在 OpenCore `Kernel → Quirks` 中启用 `AppleXcpmCfgLock`（必要时再评估 `AppleCpuPmCfgLock`）。
3. 用 `ocvalidate` 确认 quirk 拼写与 OpenCore 版本匹配。

### 2.3 VT-d 与 `DisableIoMapper`

- 优先在 BIOS 中 **Disable VT-d**。
- 如果 BIOS 没有 VT-d 选项，或必须为 Windows 虚拟机保留 VT-d，则在 OpenCore 中启用：
  - `Kernel → Quirks → DisableIoMapper = true`
- `DisableIoMapper` 会让 macOS 不使用 IOMMU 映射；这不等于关闭硬件虚拟化，VT-x 仍可开启。
- 不要同时开启 `DisableIoMapper` 与 `DisableIoMapperMapping`，除非有明确的排障需求。

### 2.4 DVMT

UHD 620 的 framebuffer 需要一块 stolen memory。HP 消费级 BIOS 经常隐藏 DVMT 选项。建议：

- 若 BIOS 可见：**DVMT Pre-Allocated = 64MB**，**DVMT Total Gfx Mem = 256MB / Max**。
- 若 BIOS 不可见：
  1. 先保持默认，验证内屏与 HDMI 是否正常；
  2. 若有花屏 / HDMI 不亮，可先用 WhateverGreen 的 `framebuffer-stolenmem` / `framebuffer-fbmem` 做软件兜底；
  3. 不要为了 DVMT 盲目修改未知 UEFI 变量；确需修改时，必须先用 `UEFITool` / `IFR Extractor` 确认 BIOS 版本、变量 GUID 与偏移。

> 本机 HDMI 修复的验证基线是 **64MB DVMT**。32MB 在 1080p 内屏下可能可用，但不建议作为 HDMI 调试起点。

### 2.5 核显 / 独显

- BIOS 中的 **Switchable Graphics / Hybrid Graphics** 保持默认（通常为 Enabled）。
- 不要选 **Discrete Graphics Only**：MX250 在 macOS 下无驱动，且会切断核显输出。
- MX250 由 `SSDT-Disable_GPU_RP05.aml` 在 ACPI 层禁用；这与 BIOS 的显卡切换不冲突。
- 若 BIOS 有 **Graphics Memory / UMA Frame Buffer Size**，按 2.4 的 DVMT 建议设置。

### 2.6 启动与 USB

- **Fast Boot = Disabled**：确保 USB 键盘、U 盘与 OpenCore 选择器可用。
- **USB Boot = Enabled**：否则 BIOS 可能不显示 U 盘启动项。
- **Boot Mode = UEFI**、**Legacy Support = Disabled**：macOS 安装器只接受 UEFI 启动。
- 安装时把 U 盘调到启动顺序第一位；安装完成后把内置 NVMe 的 `OpenCore` 启动项调到第一位。

---

## 3. 如果 BIOS 隐藏了 DVMT / CFG Lock

这是 HP 15s 系列最常见的情况。建议按以下顺序处理：

### 方案 A：先验证默认值是否足够

1. 不改 DVMT，直接安装 macOS；
2. 测试内屏 1920×1080@60、HDMI 2560×1440@120；
3. 若都正常，不要为了「看起来更标准」去改隐藏变量。

### 方案 B：软件兜底

- CFG Lock：OpenCore `AppleXcpmCfgLock = true`
- DVMT：WhateverGreen `framebuffer-stolenmem` / `framebuffer-fbmem`（按 WEG 文档与自身分辨率计算，不要照抄他人值）
- 这些补丁只能缓解 framebuffer 内存不足，**不能替代 VBT 连接器映射**；HDMI 的 `busId` 仍必须正确。

### 方案 C：确需修改 UEFI 变量

仅在以下条件全部满足时进行：

- 已确认本机 BIOS 版本与 BIOS 区域（`BIOS region` dump）；
- 已用 `UEFITool` 找到 `Setup` 模块，并通过 `IFR Extractor` 确认目标选项的 VarStore / offset；
- 已备份原始 BIOS 或确认可以使用 HP BIOS Recovery；
- 有可用的硬件编程器或售后恢复方案。

**本文不提供任何 `setup_var` 偏移**，因为不同 BIOS 版本不可通用。

---

## 4. 修改后的验证

在 Windows 中可快速确认：

```powershell
# 确认 UEFI 模式（应显示 UEFI）
msinfo32

# 确认 Secure Boot 状态（应返回 False）
Confirm-SecureBootUEFI
```

在 macOS 中确认：

```bash
# 核显与显示
system_profiler SPDisplaysDataType

# 启动参数（应包含本文的 boot-args）
nvram boot-args
```

---

## 5. BIOS 更新注意事项

- HP BIOS 更新可能重置 Secure Boot、VT-d、DVMT、启动顺序等设置；更新后必须逐项复查。
- 不要在 macOS 安装 / 调试期间更新 BIOS；先完成系统安装并保留备用 EFI。
- 如果 BIOS 更新后 HDMI 失效，先检查 `VT-d`、`DVMT`、`Secure Boot` 是否被重置，再检查 VBT 是否随 BIOS 更新变化（极少数机型会改变 VBT）。
- 若 BIOS 更新改变了 VBT，`con1 busId` 可能不再适用；需要重新提取 `i915_vbt.bin` 并按 [HDMI 修复记录](./HDMI-修复记录.md) 重算。

---

## 6. 恢复与应急

1. 准备 Windows 安装 U 盘或 Linux Live U 盘；
2. 在 BIOS 中执行 **Load Setup Defaults**，再重新关闭 Secure Boot、VT-d、CFG Lock；
3. 若无法进入 BIOS，尝试 HP 的 BIOS Recovery（通常为开机时按 `Win + B` 或 `Win + V`，具体以本机手册为准）；
4. 若 OpenCore 无法启动，先用另一块 EFI 或 Windows 启动盘挂载 ESP，替换 / 删除有问题的 `config.plist` 或 kext。

> 不要在没有备份的情况下批量替换 `EFI/OC/Kexts`。每次只改一个变量，并记录结果。

---

## 7. 参考资料

- [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/)
- [OpenCore Configuration（Dortania）](https://dortania.github.io/docs/latest/Configuration.html)
- [acidanthera/WhateverGreen](https://github.com/acidanthera/WhateverGreen)
- [acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg)
- 本仓库 [HDMI 修复记录](./HDMI-修复记录.md)
- 本仓库 [主 README](../README.md)

---

## 8. 本机待核对项

以下信息尚未从本机 BIOS 中记录，发布前建议补齐：

- [ ] BIOS 厂商、版本号与发布日期
- [ ] BIOS 中是否可见 DVMT Pre-Allocated / CFG Lock
- [ ] 实际设置的 DVMT 值（64MB 是否为默认/手动）
- [ ] TPM 是否启用（双系统 Windows 11 需要）
- [ ] Secure Boot 关闭后的 `Confirm-SecureBootUEFI` 返回值
- [ ] BIOS 更新后 VBT 是否变化
