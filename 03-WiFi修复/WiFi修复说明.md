# Wi-Fi 修复 — 在 Sequoia 上复活 Broadcom BCM4360

> 本机无线网卡：**Broadcom BCM4360**（`14e4:43a0`，子系统 `106b:117` —— 苹果原装卡的标识，PCIe 接口，ACPI 路径 `ARPT`）。
> 这张卡在 Ventura 之前是**免驱**的，但从 macOS Sonoma/Sequoia 起苹果删掉了原生驱动。本包用 OpenCore Legacy Patcher（OCLP）体系的补丁把它救了回来。
>
> **好消息：修复所需的 kext 已经全部放进 `01-EFI/EFI/OC/Kexts/`，并且在 config.plist 里配好了。** 这个文件主要解释原理和排障 —— 如果你想照搬到别的机器，往下看。

---

## 一、问题：苹果删掉了什么

macOS 14/15 移除了对老 Broadcom 网卡的原生支持，原因是底层网络栈重构：

- `IOSkywalkFamily` 被重写，新的 Skywalk 栈不再兼容老的 `IO80211Family`
- `AirPort.BrcmNIC` 依赖的老 `IO80211Family` 接口整个消失

表现：进系统后**根本没有 Wi-Fi 图标**，网络偏好设置里只有一个"Wi-Fi（未安装）"或干脆没有。

---

## 二、方案：三件套 + 一条屏蔽规则

### 需要的 kext（本包已包含）

| Kext | 版本 | 作用 |
|---|---|---|
| `IOSkywalkFamily.kext` | 1.0 | **回退版** Skywalk 栈，替换系统自带的新版 |
| `IO80211FamilyLegacy.kext` | 1200.12.2b1 | 老版 `IO80211Family`，提供 Broadcom 驱动所需的接口 |
| ↳ `PlugIns/AirPortBrcmNIC.kext` | 1400.1.1 | BCM4360 的实际驱动 |
| `AirportBrcmFixup.kext` | 2.2.1 | 国家码 / 唤醒等辅助修复 |

### 最关键的一步：屏蔽系统自带的 Skywalk

只有注入旧 kext 是不够的 —— 系统自带的 `com.apple.iokit.IOSkywalkFamily` 会抢先加载。
必须在 `Kernel → Block` 里把它排除掉：

```
Identifier : com.apple.iokit.IOSkywalkFamily
Strategy   : Exclude
Enabled    : true
MinKernel  : 23.0.0
MaxKernel  : 24.99.99
```

> **没有这条 Block，Wi-Fi 一定出不来。** 这是整个修复里最容易漏的一步。

### 注入顺序（必须严格遵守）

`Kernel → Add` 里的顺序即为加载顺序，颠倒会失败：

```
[20] IOSkywalkFamily.kext                                   ← 先
[21] IO80211FamilyLegacy.kext                               ← 后
[22] IO80211FamilyLegacy.kext/.../AirPortBrcmNIC.kext        ← 最后
[23] AirportBrcmFixup.kext
```

### 版本范围（MinKernel / MaxKernel）

| Kext | MinKernel | MaxKernel | 含义 |
|---|---|---|---|
| IOSkywalkFamily | 23.0.0 | 24.99.99 | 只在 Sonoma(23) ~ Sequoia(24) 生效 |
| IO80211FamilyLegacy | 23.0.0 | 24.99.99 | 同上 |
| AirPortBrcmNIC 插件 | 23.0.0 | 24.99.99 | 同上 |
| AirportBrcmFixup | 23.0.0 | （无上限） | 全版本生效 |
| **Block IOSkywalkFamily** | 23.0.0 | 24.99.99 | 同上 |

> ⚠️ **边界警告**：升级到 **macOS 26 Tahoe（Darwin 25）** 后，这套补丁会**整体失效**（版本范围封顶在 24.99.99），Wi-Fi 会回到不可用状态。届时要用新版 OCLP 重新生成补丁，别指望直接升级。

### boot-args

```
brcmfx-country=US
```
配合 AirportBrcmFixup 修正国家码。若你在非美国地区遇到 5GHz 信道少 / 连不上某些 AP，可改为 `brcmfx-country=#a`（让驱动跟随 AP 广播的国家码）。

---

## 三、验证

```bash
# 1) 老版 Skywalk 是否被成功替换（版本应是 1.0，不是系统自带的 16.x）
kmutil showloaded --list-only | grep -i skywalk

# 2) Broadcom 驱动是否加载
kmutil showloaded --list-only | grep -iE "IO80211|AirPort.BrcmNIC"

# 3) 网卡硬件是否在
ioreg -rc IOPCIDevice -n ARPT | grep IOName
# 期望：pci14e4,43a0
```

期望输出：

```
com.apple.iokit.IOSkywalkFamily (1.0)              ← 注意是 1.0
com.apple.iokit.IO80211FamilyLegacy (1200.12.2b1)
com.apple.driver.AirPort.BrcmNIC (1400.1.1)
```

菜单栏出现 Wi-Fi 扇形图标，能搜到周围热点即成功。

---

## 四、排障

| 现象 | 原因 / 处理 |
|---|---|
| 完全没有 Wi-Fi 图标 | 90% 是 `Kernel → Block` 里漏了 `IOSkywalkFamily` 的 Exclude 规则 |
| 有图标但一直"正在查找网络" | 检查注入顺序：IOSkywalk 必须在 IO80211FamilyLegacy **之前** |
| `IOSkywalkFamily` 显示 16.x 而非 1.0 | Block 没生效。检查 MinKernel/MaxKernel 是否覆盖当前 Darwin 版本（`uname -r` 看） |
| 能连 2.4G 连不上 5G | 国家码问题，改 `brcmfx-country=#a` |
| 睡眠后 Wi-Fi 掉线 | 这是 BCM4360 + 老 Skywalk 的已知毛病，手动点一下即可恢复 |
| 升级 macOS 后失效 | 版本号超出 24.99.99 范围，见第二节的边界警告 |

---

## 五、已知问题（诚实交代）

**重启后偶尔需要手动点一次连接**

airportd 的自动加入会空转，日志表现为：

```
[corewifi] AUTO-JOIN: Known network is found nearby but not joined
[corewifi] AUTO-JOIN COMPLETED (result=0)
```

手动点一下网络名就秒连，Wi-Fi 功能本身完全正常。
这属于 macOS 侧"已保存网络档案"的老问题，与驱动补丁无关。

排查命令（供参考）：

```bash
/usr/bin/log show --last 10m --predicate 'process == "airportd"' --info --debug | grep AUTO-JOIN
```

> 注：zsh 有内置 `log`，必须写全路径 `/usr/bin/log`。

---

## 六、想用在别的机器上？

这套补丁适用于**所有被 Sequoia 剔除的 Broadcom 网卡**，不止 BCM4360。但要注意：

1. 确认你的网卡型号（`ioreg -rc IOPCIDevice -n ARPT | grep IOName`）
2. `AirPortBrcmNIC` 覆盖的是 BCM43xx 系列；更老的 BCM4321/4322 需要 `AirPortBrcm43224` 或 `AirPortBrcm4360`（由 AirportBrcmFixup 提供插件）
3. `MinKernel/MaxKernel` 要匹配你的 macOS 版本
4. 最省事的做法：直接用 [OpenCore Legacy Patcher](https://github.com/dortania/OpenCore-Legacy-Patcher) 跑一遍网络补丁，再把生成的 kext 导入本 EFI
