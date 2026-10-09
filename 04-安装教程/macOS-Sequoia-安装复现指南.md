# macOS Sequoia 15.x 安装复现指南

> 目标：从一台裸机（或已装 Windows/Linux 的联想小新 13 / Pro 13）开始，装出一台和原机**功能完全一致**的 macOS Sequoia 15.8。
>
> 全程约 1.5 小时（不含下载时间）。建议先通读一遍再动手。

---

## 第 0 步：确认你的机器对得上

这套 EFI 是为下面这台机器调的，**硬件不一致请谨慎**（核显 / 网卡 / 声卡差异会导致功能缺失）：

| 项目 | 值 |
|---|---|
| 机型 | 联想小新 13 / Pro 13（Comet Lake） |
| CPU | Intel Core i7-10710U（6C/12T，Comet Lake-U） |
| 核显 | Intel UHD 620（Coffee Lake） |
| 声卡 | Realtek ALC257 + Intel cAVS `8086:02c8` |
| 无线 | Broadcom BCM4360（`14e4:43a0`） |
| 触控板 | I2C |
| 键盘 | PS/2 |
| 目标系统 | macOS 15 Sequoia（实测 15.8 / 24H23） |

同样是 Comet Lake 的其他联想机型，大概率能用，但触控板（I2C vs PS2）、声卡 layout-id 可能需要调整。

### 你需要准备

- 一个 **≥16GB 的 U 盘**（会被完全抹掉）
- 一台能联网的 Mac / 黑苹果（用来制作安装盘）或本机的现有系统
- 可选：ProperTree / OCAT（编辑 config.plist）、GenSMBIOS（生成序列号）
- 目标盘至少留 **40GB** 空闲

---

## 第 1 步：BIOS 设置

开机狂按 **F2**（联想机型，部分机型需配合 Fn）进 BIOS，改下面这些。
不同 BIOS 版本项名会略有出入，**找不到的项跳过即可**。

| 设置项 | 改成 | 为什么 |
|---|---|---|
| **Secure Boot** | `Disabled` | 否则 OC 无法加载 |
| **CFG Lock**（MSR 0xE2） | `Disabled` | 影响 CPU 原生电源管理。如果 BIOS 里没有此项，需要改用 OC 的 `AppleXcpmCfgLock = true` 绕过 |
| **VT-d** | `Disabled` | **内置麦克风的前提**（配合 `DisableIoMapper`）。BIOS 没有就靠 config 里的 `DisableIoMapper=true` 顶上 |
| **SATA Mode / Storage** | `AHCI` | macOS 不认 RAID/IRST |
| **DVMT Pre-Allocated** | `64MB`（至少 32MB） | 核显显存，太小会花屏/黑屏 |
| **Fast Boot / 快速启动** | `Disabled` | 避免跳过 USB 设备枚举 |
| **OS Optimized Defaults** | `Disabled` / `Other OS` | 联想机型特有项，开启会锁死一些设置 |
| **Secure Boot / Intel Platform Trust** | 关 | — |

改完 **F10 保存退出**。

---

## 第 2 步：下载 Sequoia 完整安装包

### 方法 A（推荐）：直接从 Apple CDN 拉完整包

App Store 常常只给一个几十 MB 的"存根"，容易失败。用直链最稳：

```bash
# 拉目录（关掉代理，避免被掐断）
curl --noproxy '*' -o sucatalog.gz \
  "https://swscan.apple.com/content/catalogs/others/index-15-14-13-12-10.16-10.15-10.14-10.13-10.12-10.11-10.10-10.9-mountainlion-lion-snowleopard-leopard-merged-1.sucatalog.gz"

# 解析出 InstallAssistant.pkg 的地址（找 com.apple.pkg.InstallAssistant.macOSSequoia，取最新的那条）
python3 - <<'PY'
import gzip, plistlib
d = plistlib.loads(gzip.open('sucatalog.gz','rb').read())
best = None
for p in d['Products'].values():
    ids = p.get('ExtendedMetaInfo',{}).get('InstallAssistantPackageIdentifiers',{})
    if ids.get('SharedSupport') == 'com.apple.pkg.InstallAssistant.macOSSequoia':
        if best is None or p['PostDate'] > best['PostDate']:
            best = p
for pkg in best['Packages']:
    if 'InstallAssistant' in pkg['URL']:
        print(pkg['URL']); break
PY

# 下载（支持断点续传，中断了重跑同一条命令即可）
curl -L -C - --retry 30 --retry-all-errors --noproxy '*' -o InstallAssistant.pkg "<上一步输出的 URL>"
```

下完校验签名：

```bash
pkgutil --check-signature InstallAssistant.pkg
```

然后安装（会在「应用程序」里出现安装 App）：

```bash
sudo installer -pkg InstallAssistant.pkg -target /
```

> 下载时建议加 `caffeinate -i` 防睡眠中断；有线网优于 Wi-Fi。

### 方法 B：App Store / 软件更新（简单但易失败）

在 App Store 搜 "macOS Sequoia"，点获取。如果只下到几十 MB 的假安装包，回到方法 A。

---

## 第 3 步：制作安装 U 盘

```bash
# 1) 抹盘：GPT + Mac OS 扩展（日志式）。把 diskX 换成你的 U 盘（用 diskutil list 确认！）
diskutil list
diskutil eraseDisk JHFS+ "USB" GPT /dev/diskX

# 2) 写入安装器
sudo /Applications/Install\ macOS\ Sequoia.app/Contents/Resources/createinstallmedia \
     --volume /Volumes/USB --nointeraction

# 3) 挂载 U 盘的 EFI 分区，把本包的 EFI 拷进去
diskutil list            # 找到 U 盘的 EFI 分区，通常是 diskXs1
sudo diskutil mount diskXs1

cp -a "Hackintosh-EFI-Lenovo-Xiaoxin13-Sequoia/01-EFI/EFI" /Volumes/EFI/
# 如果提示已存在 EFI 目录，先删掉 U 盘上那个空的再拷

diskutil unmount diskXs1
```

**校验一下**拷进去的东西对不对：

```bash
ls /Volumes/EFI/EFI/          # 应该有 Boot 和 OC 两个目录
ls /Volumes/EFI/EFI/OC/       # 应该有 config.plist、OpenCore.efi、Kexts、ACPI、Drivers
```

---

## 第 4 步：从 U 盘启动并安装

1. 插上 U 盘，开机按 **F12**（联想的启动菜单键）选择 U 盘启动
2. 出现 OpenCore 引导菜单 → 选 **`Install macOS Sequoia`**
   - 如果看不到条目：检查 `Misc → Security → ScanPolicy`，本包已设为 `0`（扫描全部）
3. 进安装器后，先打开 **磁盘工具**：
   - 右上角"显示"→"显示所有设备"
   - 选中目标**磁盘**（不是卷宗），点"抹掉"
   - 名称：`Macintosh HD`，格式：**APFS**，方案：**GUID 分区图**
4. 退出磁盘工具 → 点"安装 macOS Sequoia" → 同意条款 → 选 `Macintosh HD`
5. **机器会自动重启若干次**，每次重启后可能仍需要手动选择 U 盘的 `macOS Installer` 条目
6. 最后进到"选择国家/地区"等设置向导 —— 到这一步系统就装好了

> 整个过程约 30~50 分钟，期间屏幕可能黑屏、进度条不动，耐心等，别断电。

---

## 第 5 步：装好后必须做的收尾

进到桌面后，这套 EFI **还只在 U 盘上**。拔了 U 盘就开不了机。按顺序完成：

### 5.1 把 EFI 落到硬盘上

```bash
diskutil list                              # 找硬盘的 EFI 分区，通常是 disk0s1
sudo diskutil mount disk0s1
cp -a "Hackintosh-EFI-Lenovo-Xiaoxin13-Sequoia/01-EFI/EFI" /Volumes/EFI/
diskutil unmount disk0s1
```

### 5.2 重新生成序列号（换机必做）

用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) 生成 `MacBookPro15,4` 的三件套，填进
硬盘 EFI 的 `EFI/OC/config.plist` → `PlatformInfo → Generic`：

- `SystemSerialNumber`
- `MLB`
- `SystemUUID`

**不改的话 iCloud / iMessage / App Store 会和原机冲突。**

### 5.3 安装内置麦克风 kext

```bash
cd "Hackintosh-EFI-Lenovo-Xiaoxin13-Sequoia/02-麦克风修复"
sudo kmutil install --volume / --path-with-history ./LatSOFAudio.kext
sudo kmutil create -z
sudo reboot
```

详细排障见 `02-麦克风修复/安装说明.md`。

### 5.4 重建缓存 / 修复权限（可选但推荐）

```bash
sudo kmutil install --volume / --update-all
sudo kcditto
```

---

## 第 6 步：验收清单

逐项确认，全绿才算复现成功：

```
[ ] 拔掉 U 盘能正常开机（说明 EFI 已落到硬盘）
[ ] 关于本机显示 MacBookPro15,4 / macOS 15.x
[ ] 显示器分辨率正确，可外接 HDMI / Type-C
[ ] 系统报告 → 图形卡/显示器：Intel UHD 620，显存正常，显示"内建显示器"
[ ] 扬声器出声，插耳机自动切换
[ ] 语音备忘录能录到人声                    ← 麦克风
[ ] Wi-Fi 能搜到并连上网络                  ← Wi-Fi
[ ] 蓝牙能搜到设备
[ ] 触控板多指手势可用
[ ] 键盘所有按键正常（含亮度/音量功能键）
[ ] 摄像头能出画面（Photo Booth）
[ ] 电量百分比正常显示
[ ] 睡眠后能唤醒，Wi-Fi 自动重连
[ ] App Store / iCloud 能登录               ← 说明序列号是干净的
```

### 快速验证命令

```bash
# 核显
ioreg -l | grep -i "AppleIntelCFLGraphics" | head -2

# 麦克风 kext
kmutil showloaded --list-only | grep -i latsof
log show --last 2m --predicate 'eventMessage CONTAINS "LatSOF"'

# Wi-Fi（Skywalk 必须是 1.0）
kmutil showloaded --list-only | grep -iE "skywalk|IO80211|BrcmNIC"

# 网卡硬件
ioreg -rc IOPCIDevice -n ARPT | grep IOName     # 期望 pci14e4,43a0

# 声卡
ioreg -rc AppleHDA | grep -i layout             # 期望 layout-id 101
```

---

## 第 7 步：常见翻车点

| 现象 | 处理 |
|---|---|
| 开机直接进 Windows / 找不到 U 盘 | BIOS 里关 Secure Boot；用 F12 手动选启动项 |
| OpenCore 菜单空白或看不到安装器 | `ScanPolicy` 应为 `0`；确认 U 盘 EFI 里 `EFI/OC/config.plist` 存在 |
| 卡在 `EndRandomSeed` | 通常是 CFG Lock 或 VT-d 没关，回第 1 步 |
| 卡在 `OpenCore is locked` | 检查 BIOS 时间是否正确 |
| 安装到一半报错"无法验证安装器" | 调整 BIOS 时间 / 断网重装 |
| 进系统后四叶草/五国 | 二分法排查：先禁用所有 kext 只留 Lilu，逐个加回来 |
| 无 Wi-Fi | `03-WiFi修复/` —— 检查 `Kernel → Block` 里的 IOSkywalkFamily 规则 |
| 麦克风没声音 | `02-麦克风修复/` —— 检查 `DisableIoMapper` 和 TCC 权限 |
| 触控板不动 | 触控板是 I2C 还是 PS2？本包两张驱动都装了，若你的机器是 PS2 触控板需另配 VoodooPS2Mouse |

---

## 附：以后怎么升级

1. **小版本升级**（15.8 → 15.9）：系统设置里直接 OTA 即可，EFI 不用动
2. **大版本升级前**：先备份 EFI
   ```bash
   sudo diskutil mount disk0s1
   cp -a /Volumes/EFI/EFI ~/Desktop/EFI-Backup-$(date +%F)
   ```
3. **升 macOS 26 Tahoe**：⚠️ 本套 EFI **不适用**。需要换 SMBIOS 为 `iMac20,1`、OpenCore 升到 1.0.x、重做核显补丁，且 **Wi-Fi 补丁会失效**（版本范围封顶 24.99.99）。这属于另一套工程。
4. **麦克风 kext 在大版本更新后需要重装**：重跑 5.3 的两条命令即可。
