# Linux USB / PCIe 设备枚举入门（第 1～6 节）

> **适用读者**：已会用 `ls`、`dmesg`，想理解「U 盘插上后内核里发生了什么」的学习者  
> **目标**：建立从 **PCI 主桥扫描** 到 **USB 设备出现在 `/sys/bus/usb`** 的完整心智模型，并能用 sysfs / 用户态工具验证  
> **建议环境**：实体 Linux、虚拟机或 WSL2（需能访问 `/sys/bus/pci`；USB 设备可选）  
> **阅读方式**：按节顺序学习；每节有「动手实验」与「自测题」  
> **与官方文档关系**：本导读补充 `Documentation/translations/zh_CN/PCI/acpi-info.rst` 等现有译文，**不替代**《如何写 PCI/USB 驱动》

---

## 目录

1. [第 1 节：什么叫「枚举」？](#第-1-节什么叫枚举)
2. [第 2 节：PCIe 与 USB 的术语、拓扑](#第-2-节pcie-与-usb-的术语拓扑)
3. [第 3 节：整条链路总览与时间线](#第-3-节整条链路总览与时间线)
4. [第 4 节：PCI 主桥从哪里来？配置空间扫描](#第-4-节pci-主桥从哪里来配置空间扫描)
5. [第 5 节：PCI 上的 USB 主机控制器（以 xHCI 为例）](#第-5-节pci-上的-usb-主机控制器以-xhci-为例)
6. [第 6 节：USB hub 与下游设备枚举](#第-6-节usb-hub-与下游设备枚举)
7. [附录：观测命令、内核符号速查、延伸阅读](#附录观测命令内核符号速查延伸阅读)

---

## 第 1 节：什么叫「枚举」？

### 1.1 两个层次，不要混为一谈

日常说「USB 枚举」，往往把两件事叠在一起：

| 层次 | 英文常称 | 在问什么 | 典型结果 |
|------|----------|----------|----------|
| **总线 / 控制器层** | PCI enumeration、bus scan | 主板上/PCIe 插槽上的 **控制器芯片** 是否存在？ | `/sys/bus/pci/devices/0000:00:14.0` |
| **USB 协议层** | USB enumeration | 某条 **USB 总线** 上挂的 **设备 / hub** 是谁？地址、描述符？ | `/sys/bus/usb/devices/1-1`、`lsusb` 一行 |

**PCIe 枚举**（在 Linux 里更常说 **PCI 配置空间扫描**）回答：  
「这条 PCI 总线上，每个 **devfn**（设备号+功能号）读配置头，是不是合法设备？」

**USB 枚举**回答：  
「在这个 **root hub / 外部 hub** 的某个 **端口** 上，有没有设备？给它分配 **地址**，读 **设备/配置/接口描述符**。」

本教程的标题 **USB / PCIe 枚举**，指的是典型 PC 上的 **串联关系**：

```text
ACPI/DT 发现 PCI 主桥
    → 扫描 PCI 总线，发现 xHCI 控制器（PCI 功能）
        → xHCI 驱动注册 USB 主机控制器 + 根 hub
            → hub 驱动在端口上发现 U 盘、键盘等
```

---

### 1.2 用户态你「先看到」谁？

| 工具 | 主要反映哪一层 |
|------|----------------|
| `lspci` | PCI 设备（含 xHCI、USB3 xHC 等） |
| `lsusb` | USB 设备树（含 root hub 字符串 `usb1` 等） |
| `dmesg` | 两层日志交错：ACPI PCI Root Bridge、`xhci`、`usb 1-1: new USB device` |

**口诀**：**先有 PCI 上的主机控制器，后有 USB 总线上的设备。**

---

### 1.3 第一节必会名词（预告）

| 名词 | 一句话 |
|------|--------|
| **Host Bridge / 主桥** | CPU 侧与 PCI/PCIe 域之间的桥梁；其 **下游** 才开始 config 扫描 |
| **配置空间（config space）** | PCI 设备上的标准寄存器窗口；读 vendor/device ID 即「发现设备」 |
| **HCD（Host Controller Driver）** | USB 主机控制器驱动，把硬件包装成 `struct usb_hcd` |
| **Root Hub / 根 hub** | 每个 USB 主机控制器在软件里 **必有一个** 逻辑 hub（不是物理小盒子） |

---

### 1.4 动手实验（约 5 分钟）

```bash
# PCI 层：本机有多少 PCI 设备节点（云主机可能只有少量 virtio/根桥）
ls /sys/bus/pci/devices | wc -l

# USB 层：没有插设备时，通常至少有 root hub（本环境若无 USB 控制器则可能为 0）
ls /sys/bus/usb/devices 2>/dev/null | wc -l

# 若存在 USB，对照 PCI 里的 xHCI
lspci 2>/dev/null | grep -iE 'usb|xhci' || echo "本机 lspci 无 USB 类设备（虚拟机常见）"
lsusb 2>/dev/null || echo "未安装 lsusb 或无 usb 总线"
```

**观察目标**：理解「两个 `ls` 数不一样」是正常的——它们是不同的总线层级。

---

### 1.5 自测题

1. 插在主板 **USB 口** 上的 U 盘，会先出现在 `lspci` 还是 `lsusb`？  
2. 「枚举」和「加载驱动 `insmod`」是不是同一件事？（提示：枚举是 **发现**；驱动 **绑定** 可以稍后进行。）

<details>
<summary>参考答案</summary>

1. **`lsusb`**（U 盘是 USB 设备；`lspci` 里对应的是 **USB 主机控制器**，不是 U 盘本身）。  
2. **不是**。例如 PCI 扫描可先创建设备节点，**匹配驱动** 在 `pci_bus_add_devices` / `driver probe` 阶段；USB 侧 `usb_new_device` 之后接口驱动才 probe。

</details>

---

## 第 2 节：PCIe 与 USB 的术语、拓扑

### 2.1 PCI 命名：domain、bus、dev、function

 sysfs 路径示例：

```text
/sys/bus/pci/devices/0000:00:14.0
                      │    │  │ └─ function (0–7)
                      │    │  └─── device (0–31)
                      │    └────── bus (0–255)
                      └─────────── segment/domain
```

- **BDF** = Bus : Device . Function，如 `00:14.0`  
- **配置空间**：对每个 BDF，内核通过 ECAM/MMCONF 或 I/O 端口访问前 256 字节（PCIe 还有扩展空间）

读 **Vendor ID / Device ID** 非 `0xffff` 即认为槽位上有设备——这是 **硬件扫描** 的核心。

---

### 2.2 PCIe 与「纯 PCI」在教程里的关系

教程标题写 **PCIe**，是因为近十年 PC 的 **Root Bridge** 多为 **PCI Express**（ACPI HID 常为 `PNP0A08`）。  
内核 **扫描逻辑** 仍大量沿用 `drivers/pci/probe.c` 的 **PCI 模型**（bridge、subordinate bus、递归 scan）。

不必在入门阶段区分每条链路是 Gen3 还是 Gen4；**先掌握 scan + probe 顺序** 即可。

---

### 2.3 USB 拓扑：一条总线、一棵树

```text
                USB Host Controller (xHCI)
                         │
                    Root Hub (usbN)
                    /    |    \
              Port1   Port2   Port3
                │              │
             键盘           外部 Hub
                            /     \
                          U 盘    手机
```

- **Tier（层）**：根 hub 为 tier 1，其端口上的设备为 tier 2，以此类推（规范限制层数）。  
- **地址**：上电后设备先 **默认地址 0**，枚举时由主机分配 **1–127**。  
- **端点（endpoint）**：真正传数据的通道；枚举阶段大量走 **默认控制端点 EP0**。

---

### 2.4 一张对照表

| PCI 世界 | USB 世界 |
|----------|----------|
| Root Bridge | （无直接对应；是 CPU 侧） |
| PCI 设备 `00:14.0` xHCI | USB Host Controller |
| — | Root Hub `usb1` |
| — | `1-2` 表示 bus1 端口 2 上的设备 |
| `pci_driver.probe` | `usb_interface.driver.probe`（在 `usb_new_device` 之后） |

---

### 2.5 动手实验

```bash
# 任选一个 PCI 设备，看 vendor/device
for d in /sys/bus/pci/devices/*; do
  echo "$(basename "$d"): $(cat "$d/vendor" 2>/dev/null) $(cat "$d/device" 2>/dev/null)"
done | head -5

# USB 设备命名规则（若有）
ls /sys/bus/usb/devices/ 2>/dev/null
# 常见：usb1（root hub）、1-0:1.0（接口）、1-1（端口设备）
```

---

### 2.6 自测题

「`0000:00:14.0` 是 U 盘吗？」为什么？

<details>
<summary>参考答案</summary>

不是。它是 **PCI 功能**（通常是 **USB 主机控制器**）。U 盘会出现在 **USB 总线** 命名空间，如 `1-1`，由 hub 端口连接后枚举得到。

</details>

---

## 第 3 节：整条链路总览与时间线

### 3.1 逻辑流程图

```text
┌──────────────────────────────────────────────────────────────────┐
│ 固件/ACPI：描述 PCI Root Bridge（_CRS 资源、MCFG 等）            │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ acpi_pci_root_add → pci_acpi_scan_root → pci_scan_child_bus      │
│ 发现 xHCI 等设备的 pci_dev                                        │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ pci_bus_add_devices → xhci_pci_probe → usb_hcd_pci_probe         │
│ → usb_add_hcd → register_root_hub → usb_new_device(根 hub)       │
└────────────────────────────┬─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ hub 驱动：hub_event → hub_port_connect → usb_new_device(下游)   │
│ → usb_enumerate_device → 接口驱动 probe                          │
└──────────────────────────────────────────────────────────────────┘
```

非 ACPI 平台（大量 ARM 嵌入式）会把 **第一层** 换成 Device Tree + `pci_host_probe`，但 **扫描 → 添加设备 → PCI driver probe → USB core** 的形状不变。

---

### 3.2 内核里「谁先谁后」（简化）

| 阶段 | 典型代码位置 | 你 dmesg 里可能看到 |
|------|----------------|---------------------|
| ACPI 注册 PCI root handler | `drivers/acpi/pci_root.c` | `ACPI: PCI Root Bridge` |
| 扫描 child bus | `drivers/pci/probe.c` | `pci_bus 0000:00: ...` |
| 绑定 xHCI | `drivers/usb/host/xhci-pci.c` | `xhci_hcd ...` |
| 注册 HCD / 根 hub | `drivers/usb/core/hcd.c` | `usb usb1: ...` |
| 端口设备 | `drivers/usb/core/hub.c` | `new USB device found` |

**initcall 顺序** 细节因配置而异（内置 vs 模块）；入门阶段记住 **PCI 命名空间必须先于 xHCI probe** 即可。  
ACPI 路径在绑定 root 设备 **之前** 必须 scan（注释写得很直白）：

```716:723:drivers/acpi/pci_root.c
	/*
	 * Scan the Root Bridge
	 * --------------------
	 * Must do this prior to any attempt to bind the root device, as the
	 * PCI namespace does not get created until this call is made (and
	 * thus the root bridge's pci_dev does not exist).
	 */
	root->bus = pci_acpi_scan_root(root);
```

---

### 3.3 与写驱动文档的边界

| 文档 | 关注点 |
|------|--------|
| 本导读 | **发现设备** 的路径、观测方法 |
| `Documentation/translations/zh_CN/PCI/pci.rst` | 如何写 **PCI 驱动**（probe/remove、资源） |
| `Documentation/driver-api/usb/usb.rst` | USB core、URB、HCD 分工 |

---

### 3.4 动手实验：用 dmesg 粗筛

```bash
dmesg 2>/dev/null | grep -iE 'pci root|xhci|usb.*new' | tail -20
# 权限不足时：sudo dmesg ...
```

若无 USB 控制器，可能只有 PCI 相关行——仍可用于核对 **第 4 节** 的 ACPI/scan 日志。

---

### 3.5 自测题

为什么 xHCI 的 `probe` 一定发生在「PCI 扫描完成之后」？

<details>
<summary>参考答案</summary>

`xhci_pci_probe` 的入参是 **`struct pci_dev *`**。该结构体只有在 **pci_scan_device → pci_device_add** 创建 PCI 设备列表后才会存在；否则驱动无法匹配、无法 `pci_enable_device` 映射 BAR。

</details>

---

## 第 4 节：PCI 主桥从哪里来？配置空间扫描

### 4.1 为什么需要 ACPI（或 DT）？

**主桥本身** 不在「另一条 PCI 总线」上等你去扫——平台固件必须告诉内核：  
有哪些 **segment/bus 范围**、内存/I/O 窗口、中断路由等。

中文译文 **强烈推荐并行阅读**：  
`Documentation/translations/zh_CN/PCI/acpi-info.rst`（说明主桥与下游设备职责划分）。

x86/ACPI PC 上，常见入口：

- `drivers/acpi/pci_root.c`：`acpi_pci_root_add()` 处理 `PNP0A03` / `PNP0A08`  
- `drivers/pci/pci-acpi.c`：`pci_acpi_scan_root()` 创建 root bus 并触发扫描

---

### 4.2 配置空间扫描在做什么？

对每条 PCI bus，内核对 **dev 0–31** 的 **function 0** 先扫一遍（多功能设备再扫其他 function）：

```3092:3096:drivers/pci/probe.c
	dev_dbg(&bus->dev, "scanning bus\n");

	/* Go find them, Rover! */
	for (devnr = 0; devnr < PCI_MAX_NR_DEVS; devnr++)
		pci_scan_slot(bus, PCI_DEVFN(devnr, 0));
```

核心函数链（记名字即可，便于 grep 源码）：

```text
pci_scan_child_bus()
  → pci_scan_slot()
    → pci_scan_device()
      → pci_setup_device() / pci_device_add()
```

若设备是 **PCI-PCI bridge**，会 **递归** 扫描 **secondary bus**（下游总线）。

---

### 4.3 扫描完成之后：把设备交给驱动

`pci_bus_add_devices()`（`drivers/pci/bus.c`）遍历总线上的 `pci_dev`，调用 `device_attach`，最终进入 **`pci_device_probe`**（`drivers/pci/pci-driver.c`），匹配到 `xhci_pci_driver` 等。

**区分两个动词**：

- **scan**：建立 `pci_dev`、填资源、挂 sysfs  
- **probe**：某个 **pci_driver** 开始占用设备、映射 BAR、使能设备

---

### 4.4 用户态如何验证 PCI 枚举结果

```bash
# 树状查看
lspci -t 2>/dev/null | head -30

# 某个设备详情（将 BDF 换成你的）
lspci -s 00:00.0 -k 2>/dev/null

# sysfs 驱动绑定
readlink -f /sys/bus/pci/devices/0000:00:00.0/driver 2>/dev/null
```

---

### 4.5 热插拔与 rescan（了解）

PCIe 热插拔、用户触发 rescan 会再次调用扫描路径（如 `pci_rescan_bus`）。  
USB 侧热插拔则主要体现在 **hub 端口状态变化** 与 `hub_event` 工作队列。

---

### 4.6 动手实验

```bash
# 列出带 driver 绑定的 PCI 设备
for d in /sys/bus/pci/devices/*; do
  drv=$(basename "$(readlink -f "$d/driver" 2>/dev/null)" 2>/dev/null)
  echo "$(basename "$d"): ${drv:-<无驱动>}"
done
```

---

### 4.7 自测题

「`lspci` 看不到某块网卡」可能有哪些 **PCI 层** 原因？（列两点）

<details>
<summary>参考答案（示例）</summary>

1. 扫描阶段未识别（设备未上电、链路 training 失败、被 hide 在 BIOS 里）。  
2. 设备存在但 **class ID 被过滤** 或内核禁用（如 `pci=disable` 类启动参数）——需结合 dmesg 与 sysfs 是否存在节点判断。

</details>

---

## 第 5 节：PCI 上的 USB 主机控制器（以 xHCI 为例）

### 5.1 从 PCI 设备到 USB 总线

Intel/AMD 平台上，USB3 控制器常见为 **PCIe 设备**，驱动入口：

- `drivers/usb/host/xhci-pci.c`：`xhci_pci_probe()` → `xhci_pci_common_probe()`  
- 通用 PCI 框架：`drivers/usb/core/hcd-pci.c` 的 **`usb_hcd_pci_probe()`**

`usb_hcd_pci_probe` 典型工作：

1. `pci_enable_device()` / 设置 DMA mask  
2. `pci_iomap()` 映射 MMIO BAR  
3. `usb_create_hcd()` 创建 HCD 对象  
4. 调用具体 HC 的 `hc_driver->reset` / `start`  
5. **`usb_add_hcd()`** 向 usbcore 注册

---

### 5.2 `usb_add_hcd` 与根 hub

`usb_add_hcd()`（`drivers/usb/core/hcd.c`）会：

- 创建 `/sys/bus/usb/devices/usbN`  
- 调用 **`register_root_hub()`**，进而对根 hub 调用 **`usb_new_device()`**

因此：**USB 枚举的第一「个」设备永远是 root hub**——由 hub 驱动管理，端口对应物理口。

xHCI 还可能注册 **第二个** shared roothub（USB3），入门阶段知道「同一芯片可能对应两个 `usbN`」即可，细节见 `xhci_pci_common_probe()` 里对 `shared_hcd` 的处理。

---

### 5.3 对照实验（有 xHCI 的机器）

```bash
# PCI 侧
lspci -nn | grep -i xhci

# USB 侧 root hub（示例输出因机器而异）
ls -l /sys/bus/usb/devices/usb*/ 2>/dev/null | head
grep PRODUCT /sys/bus/usb/devices/usb?/uevent 2>/dev/null
```

期望建立的联系：**同一个物理芯片** → PCI 上有一个 BDF → USB 上出现 `usb1` / `usb2`。

---

### 5.4 其它 PCI USB 主机控制器（知道名字即可）

| 驱动文件 | 时代/场景 |
|----------|-----------|
| `ehci-pci.c` | USB 2.0 |
| `ohci-pci.c` / `uhci-pci.c` | 更老平台 |
| SoC 内置 USB | 可能 **platform device** 而非 PCI——跳过 PCI 层，直接 `usb_add_hcd` |

---

### 5.5 自测题

「模块 `modprobe xhci_pci` 失败」与「U 盘不识别」是否一定是同一问题？

<details>
<summary>参考答案</summary>

**不一定**。`xhci_pci` 失败是 **主机控制器 / PCI 层** 问题；U 盘识别失败可能是 **下游端口、hub 供电、USB 授权、设备描述符异常** 等。应分层排查：先 `lsusb` 能否看到设备，再看 dmesg 是 xHCI 还是 hub 报错。

</details>

---

## 第 6 节：USB hub 与下游设备枚举

### 6.1 谁负责「发现新 USB 设备」？

**核心文件**：`drivers/usb/core/hub.c`

| 函数 / 机制 | 作用 |
|-------------|------|
| `hub_probe()` | hub 接口被绑定（含 root hub） |
| `hub_event()` | 工作队列，处理连接/断开/复位 |
| `hub_port_connect()` | 某端口检测到设备，分配地址 |
| **`usb_new_device()`** | 对 **新地址** 的设备做 core 层 setup |
| **`usb_enumerate_device()`** | 读描述符、选配置、创建设备节点 |

注释说明 **`usb_new_device`** 的调用者限制（仅 hub / root hub 注册路径）：

```text
drivers/usb/core/hub.c — usb_new_device()
  仅 hub 驱动或 root-hub 注册逻辑应调用；设备处于 ADDRESS 状态，尚未完整配置。
```

---

### 6.2 枚举协议层（简化步骤）

1. 端口 **复位**，检测 **速度**（Low / Full / High / Super）  
2. **SET_ADDRESS**（控制传输，EP0）  
3. 读 **DEVICE descriptor**（至少 8 字节再读全长）  
4. 读 **CONFIGURATION / STRING** 等  
5. **SET_CONFIGURATION**  
6. 对每个 **interface** 调用 `device_add` → 匹配 **usb_driver**

外部 **USB Hub** 被识别后，会在其 **下行端口** 重复上述过程——因此会出现 **级联**。

---

### 6.3 与 «PCIe 枚举» 标题的关系（收束）

| 步骤 | 总线 | 关键结果 |
|------|------|----------|
| Root bridge + scan | PCI/PCIe | 发现 xHCI `pci_dev` |
| xHCI probe | PCI + USB core | 创建 `usb1` root hub |
| 插 U 盘 | USB | `1-1: new high-speed USB device` |

**标题里的 PCIe** 强调：现代 PC 上 USB 主机控制器 **常是 PCIe 设备**；  
**USB 枚举** 强调：真正读写 U 盘走的是 **USB 事务**（URB），与 PCI config 空间已无直接关系。

---

### 6.4 观测与调试

```bash
# 设备概要
lsusb -t 2>/dev/null

# 单设备描述符（需 usbutils）
lsusb -v -d 0781:5567 2>/dev/null | head -40   # 示例 VID:PID，请换成你的

# 内核日志
dmesg | tail -50 | grep -i usb
```

进阶：**usbmon**（`Documentation/translations/zh_CN/usb/usbmon.rst`）可抓控制传输，直接看到 GET_DESCRIPTOR 等。

**USB 设备授权**（部分发行版）：未授权前接口驱动不会 probe，表现像「枚举了但不可用」——见 `Documentation/translations/zh_CN/usb/authorization.rst`。

---

### 6.5 动手实验（无实体 U 盘时）

若仅有 root hub：

```bash
lsusb 2>/dev/null
# 预期至少一行 Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

若有可移动存储：

```bash
udevadm monitor --udev --subsystem-match=usb
# 另开终端插拔 U 盘，观察 ADD/REMOVE 事件顺序
```

---

### 6.6 自测题

root hub 的 **PRODUCT** 字符串为什么常显示 «Linux Foundation»？

<details>
<summary>参考答案</summary>

根 hub 由 **内核 usbcore + HCD 软件实现**，并非独立厂商 USB 芯片；描述符中的 idVendor/idProduct 使用 Linux 维护的 **占位 ID**（如 1d6b:0002），表示 **2.0 root hub**。

</details>

---

## 附录：观测命令、内核符号速查、延伸阅读

### A.1 命令速查

| 目的 | 命令 |
|------|------|
| PCI 树 | `lspci -t`, `lspci -nn` |
| PCI 驱动 | `lspci -k`, `/sys/.../driver` |
| USB 树 | `lsusb -t` |
| 实时 udev | `udevadm monitor` |
| 日志 | `dmesg -w`（需权限） |

### A.2 内核符号速查（建议 `grep` 源码）

| 符号 | 文件 | 一句话 |
|------|------|--------|
| `acpi_pci_root_add` | `drivers/acpi/pci_root.c` | ACPI 发现 root bridge 并 scan |
| `pci_scan_child_bus` | `drivers/pci/probe.c` | 扫描一条 PCI bus |
| `pci_bus_add_devices` | `drivers/pci/bus.c` | 触发 PCI 驱动 probe |
| `xhci_pci_probe` | `drivers/usb/host/xhci-pci.c` | xHCI 作为 PCI 驱动入口 |
| `usb_hcd_pci_probe` | `drivers/usb/core/hcd-pci.c` | PCI USB HCD 公共 probe |
| `usb_add_hcd` | `drivers/usb/core/hcd.c` | 注册 USB 总线与 root hub |
| `usb_new_device` | `drivers/usb/core/hub.c` | 新 USB 设备 core setup |
| `hub_port_connect` | `drivers/usb/core/hub.c` | 端口连接处理 |

### A.3 延伸阅读（仓库内）

| 路径 | 说明 |
|------|------|
| `Documentation/translations/zh_CN/PCI/acpi-info.rst` | PCI 主桥与 ACPI（中文） |
| `Documentation/translations/zh_CN/PCI/pci.rst` | 写 PCI 驱动（中文） |
| `Documentation/PCI/acpi-info.rst` | 同上（英文） |
| `Documentation/driver-api/usb/usb.rst` | USB 子系统架构（英文） |
| `Documentation/translations/zh_CN/usb/usbmon.rst` | 抓 USB 包（中文） |

### A.4 常见误区

1. **把 PCI 扫描与 USB 枚举当成一步** —— 实际是 **前后两段**，中间隔着 HCD + root hub。  
2. **以为 root hub 是可选** —— 每个 HCD **必有** root hub。  
3. **在 sysfs 里找 «U 盘对应的 PCI BDF»** —— U 盘通常 **没有** 独立 PCI 功能；应查 USB 路径与块设备 `sdX` 的关联（存储栈另论）。

### A.5 后续章节（未写入本文档）

若继续写第 7 节及以后，建议主题：**USB 存储栈（`usb-storage` → SCSI → block）**、**DMA 与缓冲**、**Type-C / role switch**、**VFIO 与 USB 直通** 等。

---

*文档版本：sections 1–6 初稿，与 Linux 内核源码树 `drivers/pci` / `drivers/usb` 对照编写。*
