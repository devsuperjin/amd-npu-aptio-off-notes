# NPU "off" 在 AMI Aptio BIOS 里的实现位置

> coreboot 用 devicetree 的 `npu off` 在固件枚举阶段关设备；AMI Aptio 没有一张对应的配置树，等价物拆在三处。以 AMD Strix Halo 平台的 NPU 为例。

## 1. 枚举本身：PciBusDxe

UEFI 标准 PCI 枚举驱动。只负责扫总线、分配资源，本身不做设备开关。

## 2. 策略钩子：EFI_PCI_PLATFORM_PROTOCOL

结构上最像 devicetree off 的位置。AMI 的 `PciPlatformDxe` / 平台 DXE 模块在枚举时被回调，可以按设备返回"禁用 / 隐藏 / 不分配资源"的策略。

```
Setup 菜单选项 → 写进 NVRAM 的 Setup 变量 → PciPlatform 回调消费
```

这是"配置决定某台设备是否启用"的标准 UEFI 通道。

## 3. SoC 内部 IP 的真实门控：不在 AMI 代码里，在 blob 里

NPU 这类 SoC 内部设备的"关"，实际路径：

```
CBS setup 选项（AMD 的设置树）→ Setup 变量 → AGESA DXE 模块（AmdInitEnv/Mid/Late 系列）
→ 写 Data Fabric 寄存器 / 发 SMU mailbox → SMU/PSP 固件做电源门控
```

真正动手的是 AGESA 二进制和 SMU 固件——AMI 代码只是把用户选项翻译成 AGESA 的输入参数。

## 在自己的 ROM 里找对应物

用 UEFITool 翻 BIOS 镜像，找这些模块名：

- `PciPlatformDxe`
- `AmdAgesa*`
- `AmdCbs*`
- `FchInit*`

Setup 选项本体在 NVRAM region 的变量区。这套手术工作流和 PSP region 手术同款——只是这次动的不是 PSP 目录，是 DXE FV 里的模块或 NVRAM 变量。

## 附注：为什么 ACPI patch 只算"软关"

- DSDT 里给 `Device(IPU)` 写 `_STA=0`：Windows 尊重它（设备管理器消失、不绑驱动）；Linux 忽略 PCI 设备的 `_STA`——照常枚举、驱动照绑，需再 blacklist。
- `_STA` 管不了总线存在性：POST 阶段固件已把设备挂上 PCI 总线，`lspci` 仍可见，电源门控归 SMU/驱动运行时。
- 真·off（总线不可见/断电）只能发生在 OS 之前的固件阶段——即本文第 2、3 节的位置。

## 附注

先查 BIOS 的 CBS 菜单有没有现成的 NPU 开关（Strix Halo 平台不少机器藏着），有就不必 patch。
