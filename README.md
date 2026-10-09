# 联想小新 13 / Pro 13 黑苹果 EFI — macOS Sequoia 15.8

一套**实测可引导、可日常使用**的 OpenCore EFI，源自一台正在跑 `macOS 15.8 Sequoia (24H23)` 的真机。

EFI、麦克风修复、Wi-Fi 修复、安装教程全部打包在这里 —— 照着做就能复现同一台机器。

---

## 一、装完之后你能得到什么

| 部件 | 型号 / 标识 | 状态 | 依赖 |
|---|---|---|---|
| CPU | Intel Core i7-10710U（Comet Lake-U，6C/12T） | ✅ 原生变频 | CPUFriend + SSDT-PLUG |
| 核显 | Intel UHD 620（Coffee Lake） | ✅ 硬件加速 | WhateverGreen + framebuffer 注入 |
| HDMI / Type-C 视频输出 | LSPcon 转换芯片 | ✅ 已修复 | `framebuffer-con1-*` 系列注入 |
| 扬声器 / 耳机 | Realtek ALC257 | ✅ | AppleALC `layout-id=101` |
| **内置麦克风（DMIC）** | Intel cAVS `8086:02c8` | ✅ | **LatSOFAudio.kext**（须手动装，见 `02`） |
| **Wi-Fi** | Broadcom BCM4360（`14e4:43a0` / 子系统 `106b:117`，苹果原装卡） | ✅ | IO80211FamilyLegacy + IOSkywalkFamily + AirportBrcmFixup |
| 蓝牙 | BCM4360 内置蓝牙 | ✅ 原生免驱 | 无 |
| 触控板 | I2C | ✅ 多指手势 | VoodooI2C + VoodooI2CHID |
| 键盘 | PS/2 | ✅ | VoodooPS2Controller |
| 摄像头 | Chicony USB UVC | ✅ 原生免驱 | 无 |
| 电池 | — | ✅ 电量 / 充电状态 | SMCBatteryManager + SSDT-OCBAT1 |
| 亮度调节 | — | ✅ 亮度键 | BrightnessKeys + SSDT-PNLF-CFL |
| 睡眠 / 唤醒 | — | ✅ | HibernationFixup + SSDT-GPRW / DeepIdle |

> 唯一需要**额外动手**的是内置麦克风：它的 kext 装在系统盘而非 EFI 里，见 `02-麦克风修复/`。

---

## 二、目录说明

```
Hackintosh-EFI-Lenovo-Xiaoxin13-Sequoia/
├── README.md                    ← 你正在看的文件
├── 01-EFI/
│   └── EFI/
│       ├── Boot/BOOTx64.efi     ← UEFI 默认引导入口
│       └── OC/                  ← OpenCore 主体（config.plist + kext + SSDT + 驱动）
├── 02-麦克风修复/
│   ├── LatSOFAudio.kext         ← 编译好的内核扩展（v1.1.29）
│   ├── NHLT                     ← ACPI NHLT 表，移植到其他机型的依据
│   ├── 安装说明.md              ← 怎么装、怎么验
│   └── 源码文档/                ← 项目 README 与移植指南
├── 03-WiFi修复/
│   └── WiFi修复说明.md          ← Sequoia 上复活 BCM4360 的原理与配置
└── 04-安装教程/
    └── macOS-Sequoia-安装复现指南.md   ← 从零到能进桌面的完整步骤
```

---

## 三、最快上手（三种用法）

### 用法 A：全新安装 Sequoia
直接看 → [`04-安装教程/macOS-Sequoia-安装复现指南.md`](04-安装教程/macOS-Sequoia-安装复现指南.md)

### 用法 B：已有黑苹果，只想换 EFI
1. 挂载你硬盘的 EFI 分区：`sudo diskutil mount disk0s1`
2. 备份原有 EFI（**务必**）
3. 把 `01-EFI/EFI` 整个拷进去，覆盖同名目录
4. **改序列号**（见下一节，否则 iCloud / iMessage 会跟原机冲突）
5. 重启，开机按引导项选 OpenCore

### 用法 C：只看某个修复怎么做
- 麦克风 → [`02-麦克风修复/安装说明.md`](02-麦克风修复/安装说明.md)
- Wi-Fi → [`03-WiFi修复/WiFi修复说明.md`](03-WiFi修复/WiFi修复说明.md)

---

## 四、⚠️ 换机必读：序列号必须重新生成

本仓库的 `config.plist` 里，三件套已经被替换成**占位值**（真机序列号不公开）：

```
SystemProductName   = MacBookPro15,4
SystemSerialNumber  = A02XXXXXXXXX                        ← 占位，必须替换
MLB                 = A0200000000000000                   ← 占位，必须替换
SystemUUID          = 00000000-0000-0000-0000-000000000000 ← 占位，必须替换
```

**不替换就用，iCloud / iMessage / FaceTime / App Store 会不可用**，也可能和别人的机器撞车。
生成属于你自己的三件套是第一步，别跳过。

生成方法（推荐 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS)）：

```bash
git clone https://github.com/corpnewt/GenSMBIOS
cd GenSMBIOS && chmod +x GenSMBIOS.command && ./GenSMBIOS.command
# 选 3) Generate SMBIOS → 机型填 MacBookPro15,4 → 生成后写入 config.plist 的 PlatformInfo → Generic
```

用 ProperTree / OCAT 打开 config.plist，把新的 `SystemSerialNumber`、`MLB`、`SystemUUID` 填进
`PlatformInfo → Generic` 即可（`SystemProductName` 保持 `MacBookPro15,4` 不变）。

> 机型 `MacBookPro15,4` 是这套配置的天花板：**最高支持到 macOS 15 Sequoia**，且走正规 OTA / App Store 路径，无需 OCLP。
> 想上 macOS 26 Tahoe 得换 `iMac20,1` + OpenCore 1.0.x 并重做核显补丁 —— 那是另一套工程，本包不含。

---

## 五、这套 EFI 的关键配置（改配置前先看这里）

`boot-args`：
```
amfi=0x80  igfxrpsc=1  -igfxblr  brcmfx-country=US  -lilubetaall
```
| 参数 | 作用 |
|---|---|
| `amfi=0x80` | 放宽 AMFI，允许加载未公证的 kext（LatSOFAudio 需要） |
| `-lilubetaall` | 允许 Lilu 在非正式支持的系统版本上工作（Sequoia 必需） |
| `igfxrpsc=1` / `-igfxblr` | CFL 核显的电源与背光相关修复 |
| `brcmfx-country=US` | 配合 AirportBrcmFixup 修正国家码 |

必须在位的 Quirks：

| 位置 | 项 | 值 | 为什么 |
|---|---|---|---|
| Kernel → Quirks | `DisableIoMapper` | **true** | **麦克风的前提**。VT-d 会静默吞掉 DSP 侧 DMA，DMIC 永远出不了声 |
| Kernel → Quirks | `CustomSMBIOSGuid` | true | 配合自定义 SMBIOS |
| Misc → Security | `SecureBootModel` | **Disabled** | 黑苹果必关，否则升级/启动卡死 |
| Booter → Quirks | `SetupVirtualMap` | true | 内存映射修复 |

---

## 六、组件版本清单（已随包归档）

| Kext | 版本 | 说明 |
|---|---|---|
| Lilu | 1.7.2 | 补丁引擎，所有 acidanthera kext 的依赖 |
| VirtualSMC 全家 | 1.3.8 | VirtualSMC / SMCBatteryManager / SMCProcessor / SMCSuperIO / SMCLightSensor |
| WhateverGreen | 1.7.1 | 核显 |
| AppleALC | 1.9.8 | 声卡（扬声器 / 耳机） |
| AirportBrcmFixup | 2.2.1 | Wi-Fi 国家码修复 |
| IO80211FamilyLegacy | 1200.12.2b1 | Wi-Fi 复活核心（含 AirPortBrcmNIC 1400.1.1 插件） |
| IOSkywalkFamily | 1.0 | Wi-Fi 复活核心，必须**排在 IO80211FamilyLegacy 之前** |
| VoodooI2C | 2.9.1 | I2C 触控板（含 VoodooGPIO / VoodooInput / VoodooI2CServices） |
| VoodooI2CHID | 1 | I2C HID 设备 |
| VoodooPS2Controller | 2.3.7 | PS/2 键盘 |
| HibernationFixup | 1.5.4 | 睡眠 |
| BrightnessKeys | 1.0.3 | 亮度键 |
| CPUFriend | 1.2.7 | 变频（**不要升级**，配套 CPUFriendDataProvider 是本机制定的数据） |
| CPUFriendDataProvider | 1.0.1 | 本机制定的变频数据，**换机需重新生成** |
| NVMeFix | 1.1.1 | **默认禁用**，需要 NVMe 电源管理再开 |
| RtWlanU / RtWlanU1827 | — | Realtek **USB** 无线网卡驱动。本机用不上（内置 BCM 已工作），不用 USB 网卡可在 config 里禁用 |

- OpenCore 本体：**0.9 系列**（2023-06 构建）— 已实测稳定引导 Sequoia 15.8
- ACPI：启用 15 个 SSDT（EC / AWAC / PLUG / PNLF / SBUS / PMCR / 电池 / 睡眠 / I2C / GPRW / PS2Map / LPCB-Wake 等）
- UEFI 驱动启用 6 个：HfsPlus、OpenRuntime、OpenCanopy、AudioDxe、CrScreenshotDxe、ToggleSipEntry

---

## 七、已知问题与边界（诚实交代）

1. **Wi-Fi 重启后偶尔要手动点一次连接**
   airportd 的自动加入会空转（`Known network is found nearby but not joined`），手动点一下秒连。
   功能本身正常，属 macOS 侧已保存网络档案的老问题，不影响使用。只需要将自己我wifi改为wap2即可。
2. **USB 端口未做定制映射**（`XhciPortLimit=false`，靠原生端口数未超限硬跑）。
   如果你发现某个 USB 口不认设备，需要自行生成 `USBPorts.kext`（用 USBMap / Hackintool）。
3. **内置麦克风依赖 `amfi=0x80` + 关闭内核扩展用户批准**，系统更新后可能需要重装一次 kext（见 `02`）。
4. **macOS 26 Tahoe 不在本包支持范围** —— 需要换 SMBIOS 与 OpenCore 版本，见第四节末尾说明。

---

## 八、排障速查

| 现象 | 先看 |
|---|---|
| 卡在 `EndRandomSeed` / 进不去安装器 | BIOS 里关 CFG Lock 与 VT-d；检查 `SecureBootModel=Disabled` |
| 进系统后无 Wi-Fi | `03-WiFi修复/` —— 多半是 IOSkywalkFamily 顺序或 kext 没注入 |
| 有声音但麦克风没输入 | `02-麦克风修复/` —— 检查 `DisableIoMapper` 与 TCC 麦克风权限 |
| 五国 / 卡 PCI 配置 | 删掉 `boot-args` 里 `-lilubetaall` 以外的项逐个二分排查 |
| 电池不显示 | 确认 SSDT-OCBAT1 与 SMCBatteryManager 都启用 |
