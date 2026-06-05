# 最小可运行 NSH 系统 defconfig 参考

\[ [English](../../../../en/contest_2026/hardware_porting/defconfig_reference/minimum_nsh_baseline.md) | 简体中文 \]

> 本文档面向「新硬件适配赛道」参赛者，从 openvela 已经适配的 **goldfish-x86_64-ap** 模拟器板出发，逐项讲解一个能在新硬件上"启动到 NSH 提示符 + 联网 + 显示 + 传感"所需的 CONFIG 选项。
>
> 你在为新板子写 defconfig 时，可以把本文当作"最小可工作"基线对照表：先确认必备项是否都开了，再按需裁剪或扩展。

## 一、为什么用 goldfish 作为基线

openvela 已适配 [Android Goldfish](https://source.android.com/docs/core/runtime/instr) 模拟器，它是一块**纯虚拟硬件**：CPU、外设、传感、网络、显示全部由 QEMU 提供。这意味着：

1. **零硬件门槛**：参赛者无需任何实物板就能跑起来，是验证 openvela 系统行为最快的入口。
2. **子系统覆盖最完整**：goldfish 单一 defconfig 已经一次性把 NSH shell、网络栈（TCP/UDP/PING/Telnet/iperf）、图形（LVGL + framebuffer + 触摸 + 摄像头）、传感（uORB + GNSS + 电池）、调试（DEBUG_FEATURES + ALLSYMS）、ADBD 等子系统全部启用。
3. **三种架构同源**：仓库中提供了 `goldfish-x86_64-ap`、`goldfish-arm64-v8a-ap`、`goldfish-armeabi-v7a-ap` 三套同构 defconfig，参赛者把"非架构相关"配置原样照搬到新板子，再替换架构标识即可。

> **完整 defconfig 路径**：`vendor/openvela/boards/vela/configs/goldfish-x86_64-ap/defconfig`（共 253 行，本文档挑选其中 ~80 个关键项分组讲解，其余项在原文件中可直接查阅）。

## 二、基线全景：8 大功能桶

`goldfish-x86_64-ap` 共 246 个有效 CONFIG 项，按功能可归为 8 大类：

| 桶 | 项数 | 是否新硬件必需 | 说明 |
| ---- | ---- | ---- | ---- |
| 1. 架构与板级标识 | 37 | ✅ 必需（替换为目标架构） | 决定编译器、入口、内存布局 |
| 2. NSH Shell 与控制台 | 13 | ✅ 必需 | 没有它就只剩裸内核，无法交互 |
| 3. 任务调度与 C/C++ 运行时 | 32 | ✅ 必需 | TLS、堆栈、worker 线程、libc/libcxx |
| 4. 文件系统与 procfs | 10 | ✅ 强烈建议 | 至少要 PROCFS，方便调试 |
| 5. 网络栈 | 51 | ⭕ 可选 | 不需要联网可全部关掉 |
| 6. 图形与输入 | 27 | ⭕ 可选 | 不需要显示可全部关掉 |
| 7. 传感器与 uORB | 16 | ⭕ 可选 | 取决于板子是否带传感器 |
| 8. 调试与诊断 | 14 | ✅ 强烈建议 | 移植阶段务必打开 |

下面按桶展开。**带 `+` 前缀的 CONFIG 表示"在已有 defconfig 上需要新增的依赖项"**（参考自 openvela 社区惯例）。

---

## 三、桶 1：架构与板级标识（必需）

这是整个 defconfig 中**唯一需要按目标硬件替换**的部分。其他七个桶在不同架构下基本不变。

### 关键 CONFIG

```
CONFIG_ARCH="x86_64"
CONFIG_ARCH_X86_64=y
CONFIG_ARCH_CHIP="goldfish"
CONFIG_ARCH_CHIP_GOLDFISH_X86_64=y
CONFIG_ARCH_BOARD_CUSTOM=y
CONFIG_ARCH_BOARD_CUSTOM_DIR="../vendor/openvela/boards/vela/"
CONFIG_ARCH_BOARD_CUSTOM_DIR_RELPATH=y
CONFIG_ARCH_BOARD_CUSTOM_NAME="vela"
CONFIG_ARCH_INTEL64_CORE_FREQ_KHZ=2600000
CONFIG_ARCH_MULTIBOOT1=y
CONFIG_ARCH_SETJMP_H=y
CONFIG_ARCH_SIZET_LONG=y
CONFIG_BOARD_LATE_INITIALIZE=y
CONFIG_BOARD_LOOPSPERMSEC=999
CONFIG_BOOT_RUNFROMEXTSRAM=y
CONFIG_RAM_SIZE=268435456
```

### 串口控制台（goldfish 用 16550 UART）

```
CONFIG_16550_UART=y
CONFIG_16550_UART0=y
CONFIG_16550_UART0_BASE=0x3f8
CONFIG_16550_UART0_CLOCK=1843200
CONFIG_16550_UART0_IRQ=36
CONFIG_16550_UART0_SERIAL_CONSOLE=y
CONFIG_16550_ADDRWIDTH=16
CONFIG_CONSOLE_SYSLOG=y
```

### 适配到新硬件时如何改

| 项 | 替换原则 |
| ---- | ---- |
| `ARCH=` / `ARCH_X86_64=y` | 改成目标 CPU 架构（`arm`/`arm64`/`risc-v` 等） |
| `ARCH_CHIP=` / `ARCH_CHIP_<NAME>=y` | 改成目标 SoC，对应 `nuttx/arch/<arch>/src/<chip>/` |
| `ARCH_BOARD_CUSTOM_*` | 指向你新建的 vendor 目录路径 |
| `16550_UART*` | 替换成目标芯片的 UART 驱动配置（如 STM32 用 `STM32_USART1=y`） |
| `RAM_SIZE` | 替换成目标板实际 RAM 容量 |
| `BOARD_LOOPSPERMSEC` | 与 CPU 频率挂钩，可先填 999 后续校准 |

> 完整移植流程参见 [openvela 芯片移植指南](../../../chip_porting/porting_guide.md)。

---

## 四、桶 2：NSH Shell 与控制台（必需）

NSH（NuttShell）是 openvela 的命令行入口，**没有它系统启动后只能干瞪眼**。

```
CONFIG_SYSTEM_NSH=y
CONFIG_SYSTEM_NSH_STACKSIZE=8192
CONFIG_NSH_ARCHINIT=y
CONFIG_NSH_BUILTIN_APPS=y
CONFIG_NSH_DISABLE_IFUPDOWN=y
CONFIG_NSH_LINELEN=256
CONFIG_NSH_PROMPT_MAX=32
CONFIG_NSH_PROMPT_STRING="openvela-ap> "
CONFIG_INIT_ENTRYPOINT="nsh_main"
CONFIG_BUILTIN=y
CONFIG_SYSTEM_CLE_CMD_HISTORY=y
CONFIG_SYSTEM_CLE_CMD_HISTORY_LINELEN=256
CONFIG_TTY_SIGINT=y
CONFIG_TTY_SIGTSTP=y
```

### 这些 CONFIG 各自的作用

- **`SYSTEM_NSH` / `INIT_ENTRYPOINT="nsh_main"`**：把 NSH 注册为系统第一个用户态任务，开机即进入提示符。
- **`NSH_ARCHINIT`**：让 NSH 启动前调用 `board_late_initialize()`，方便板级驱动注册。
- **`NSH_BUILTIN_APPS`**：允许 `apps/` 下的应用以"内置命令"形式被 NSH 调用（如 `hello`、`iperf`、`ping`）。
- **`NSH_PROMPT_STRING`**：提示符字符串，随板子修改即可（如 `"openvela-mynew> "`）。
- **`SYSTEM_CLE_CMD_HISTORY`**：开启上下方向键历史命令回滚，开发体验提升明显。
- **`TTY_SIGINT` / `TTY_SIGTSTP`**：让 Ctrl-C / Ctrl-Z 真正能中断/挂起前台任务。

### 最小可运行子集

如果你只想验证"NSH 能起来"，桶 2 的 14 项中 **`SYSTEM_NSH`、`INIT_ENTRYPOINT`、`NSH_ARCHINIT`、`BUILTIN`** 四项是真正不可少的。

---

## 五、桶 3：任务调度与 C/C++ 运行时（必需）

```
CONFIG_DEFAULT_TASK_STACKSIZE=4194304
CONFIG_IDLETHREAD_STACKSIZE=4194304
CONFIG_PTHREAD_STACK_MIN=8192
CONFIG_POSIX_SPAWN_DEFAULT_STACKSIZE=2048
CONFIG_SCHED_TICKLESS=y
CONFIG_SCHED_TICKLESS_ALARM=y
CONFIG_SCHED_TICKLESS_LIMIT_MAX_SLEEP=y
CONFIG_USEC_PER_TICK=1
CONFIG_SCHED_HPWORK=y
CONFIG_SCHED_HPWORKPRIORITY=192
CONFIG_SCHED_HPWORKSTACKSIZE=8192
CONFIG_SCHED_LPWORK=y
CONFIG_SCHED_LPWORKSTACKSIZE=8192
CONFIG_SCHED_BACKTRACE=y
CONFIG_SCHED_HAVE_PARENT=y
CONFIG_SCHED_CHILD_STATUS=y
CONFIG_PREALLOC_CHILDSTATUS=16
CONFIG_PRIORITY_INHERITANCE=y
CONFIG_SIG_DEFAULT=y
CONFIG_STACK_COLORATION=y
CONFIG_TLS_NELEM=16
CONFIG_TLS_TASK_NELEM=16
CONFIG_TLS_NCLEANUP=2
CONFIG_LIBC_ENVPATH=y
CONFIG_LIBC_EXECFUNCS=y
CONFIG_LIBC_LOCALTIME=y
CONFIG_LIBM=y
CONFIG_LIBCXX=y
CONFIG_CXX_EXCEPTION=y
CONFIG_CXX_RTTI=y
CONFIG_CXX_WCHAR=y
CONFIG_LIBUV=y
CONFIG_MM_PGALLOC=y
```

### 注意事项

- **`DEFAULT_TASK_STACKSIZE=4 MiB`** 与 **`IDLETHREAD_STACKSIZE=4 MiB`** 是 goldfish 模拟器（256 MiB RAM）的设置。**移植到 RAM 紧张的真实硬件时请大幅调小**（典型值 `4096 ~ 16384`）。
- **`SCHED_TICKLESS=y`** 启用无滴答调度，配合 `USEC_PER_TICK=1` 提供微秒级时间精度，但要求底层有高精度 alarm 定时器；不具备时改回传统 ticked 模式（删掉这三项）。
- **`SCHED_HPWORK` / `SCHED_LPWORK`**：高/低优先级 worker 线程，被驱动框架（输入、传感器、网络）广泛使用，**不开就有大量驱动跑不起来**。
- **`LIBCXX` + `CXX_EXCEPTION/RTTI/WCHAR`**：完整 C++ ABI 支持，openvela 的 LVGL、Camera 等模块依赖。如目标固件极度精简可关闭，但要承担库依赖崩塌的风险。

---

## 六、桶 4：文件系统与 procfs（强烈建议）

```
CONFIG_FS_FAT=y
CONFIG_FS_PROCFS=y
CONFIG_FS_SHMFS=y
CONFIG_FS_V9FS=y
CONFIG_V9FS_VIRTIO_9P=y
CONFIG_FS_LOCK_BUCKET_SIZE=8
CONFIG_FS_NAMED_SEMAPHORES=y
CONFIG_PSEUDOFS_FILE=y
CONFIG_PSEUDOFS_SOFTLINKS=y
CONFIG_NAME_MAX=64
```

### 强烈建议保留的子项

- **`FS_PROCFS=y`**：挂载 `/proc` 后可以 `cat /proc/meminfo`、`cat /proc/<pid>/status` 排查问题，移植阶段是救命稻草。
- **`PSEUDOFS_FILE` + `PSEUDOFS_SOFTLINKS`**：支持伪文件系统中的虚拟文件和软链接（`/dev/console` 等）。
- **`FS_FAT=y`**：通用 FAT 文件系统，用于挂载 SD 卡等存储介质。

### goldfish 专用、其他板可不要的

- **`FS_V9FS` + `V9FS_VIRTIO_9P`**：通过 virtio-9p 把宿主机目录挂到虚拟机里，**仅 QEMU 场景需要**，真实硬件可关。

---

## 七、桶 5：网络栈（按需启用）

goldfish 默认带完整 TCP/IP 栈，含 telnet 服务、iperf、tftp、webclient。**如果你的目标板没有网卡或第一阶段不联网，这 51 项可以全部关掉，能显著缩小固件体积。**

### 网络核心

```
CONFIG_NET=y
CONFIG_NET_TCP=y
CONFIG_NET_UDP=y
CONFIG_NET_ICMP_SOCKET=y
CONFIG_NET_IGMP=y
CONFIG_NET_LOCAL=y
CONFIG_NET_PKT=y
CONFIG_NET_NETLINK=y
CONFIG_NET_BROADCAST=y
CONFIG_NET_STATISTICS=y
CONFIG_NET_ETH_PKTSIZE=1514
```

### TCP 写缓冲与可靠性增强

```
CONFIG_NET_TCP_WRITE_BUFFERS=y
CONFIG_NET_TCP_NWRBCHAINS=128
CONFIG_NET_TCP_DELAYED_ACK=y
CONFIG_NET_TCP_SELECTIVE_ACK=y
CONFIG_NET_TCP_OUT_OF_ORDER_BUFSIZE=32768
CONFIG_NET_RECV_BUFSIZE=32768
CONFIG_NET_SEND_BUFSIZE=32768
```

### IP 地址（goldfish 固定地址）

```
CONFIG_NETINIT_THREAD=y
CONFIG_NETINIT_IPADDR=0x0a00020f          # 10.0.2.15
CONFIG_NETINIT_DRIPADDR=0x0a000202        # 10.0.2.2  网关
CONFIG_NETINIT_DNSIPADDR=0x0a000203       # 10.0.2.3  DNS
CONFIG_NETINIT_DNS=y
CONFIG_NETDB_DNSCLIENT=y
CONFIG_NETDB_DNSCLIENT_ENTRIES=4
```

### 网卡驱动 + IOB 缓冲池

```
CONFIG_NET_E1000=y
CONFIG_DRIVERS_VIRTIO_NET=y
CONFIG_IOB_BUFSIZE=2048
CONFIG_IOB_NBUFFERS=8192
CONFIG_IOB_ALIGNMENT=64
CONFIG_IOB_NCHAINS=64
```

### 应用层工具

```
CONFIG_SYSTEM_PING=y
CONFIG_SYSTEM_PING_STACKSIZE=8192
CONFIG_NETUTILS_TELNETD=y
CONFIG_SYSTEM_TELNETD_STACKSIZE=8192
CONFIG_NETUTILS_IPERF=y
CONFIG_NETUTILS_IPERF_STACKSIZE=8192
CONFIG_UTILS_IPERF2=y
CONFIG_NETUTILS_TFTPC=y
CONFIG_NETUTILS_WEBCLIENT=y
CONFIG_NETUTILS_CODECS=y
```

> **适配建议**：移植到真实网卡时，把 `NET_E1000` 和 `DRIVERS_VIRTIO_NET` 换成你的 PHY/MAC 驱动（如 `STM32_ETHMAC=y`、`ESP32_WIRELESS=y`），其他网络配置可基本照搬。

---

## 八、桶 6：图形与输入（按需启用）

goldfish 带完整 LVGL + framebuffer + 触摸 + 摄像头链路。如目标板无显示，全部关闭。

### 显示与 framebuffer

```
CONFIG_VIDEO=y
CONFIG_VIDEO_FB=y
CONFIG_VIDEO_STREAM=y
CONFIG_DRIVERS_VIDEO=y
CONFIG_GOLDFISH_GPU_FB=y
CONFIG_GOLDFISH_GPU_FB_PRIORITY=101
CONFIG_GOLDFISH_PIPE=y
```

### LVGL 图形库

```
CONFIG_GRAPHICS_LVGL=y
CONFIG_LV_USE_NUTTX=y
CONFIG_LV_USE_NUTTX_LIBUV=y
CONFIG_LV_USE_NUTTX_TOUCHSCREEN=y
CONFIG_LV_USE_FREETYPE=y
CONFIG_LV_USE_LOG=y
CONFIG_LV_USE_DEMO_WIDGETS=y
CONFIG_LV_USE_CLIB_MALLOC=y
CONFIG_LV_USE_CLIB_SPRINTF=y
CONFIG_LV_USE_CLIB_STRING=y
CONFIG_LV_BIN_DECODER_RAM_LOAD=y
CONFIG_LV_DEF_REFR_PERIOD=16
CONFIG_LV_CACHE_DEF_SIZE=8388608
CONFIG_LV_IMAGE_HEADER_CACHE_DEF_CNT=128
CONFIG_LIB_FREETYPE=y
CONFIG_LIBYUV=y
```

### 输入与摄像头

```
CONFIG_INPUT=y
CONFIG_INPUT_GOLDFISH_EVENTS=y
CONFIG_UINPUT_TOUCH=y
CONFIG_GOLDFISH_CAMERA=y
CONFIG_SYSTEM_NXCAMERA=y
```

### Demo

```
CONFIG_EXAMPLES_FB=y
CONFIG_EXAMPLES_LVGLDEMO=y
```

> **适配建议**：把 `GOLDFISH_GPU_FB` / `INPUT_GOLDFISH_EVENTS` / `GOLDFISH_CAMERA` 替换成你的真实 LCD/触摸/摄像头驱动。LVGL 子树本身可保留，是上层 UI 框架。

---

## 九、桶 7：传感器与 uORB（按需启用）

```
CONFIG_SENSORS=y
CONFIG_UORB=y
CONFIG_USENSOR=y
CONFIG_UORB_LISTENER=y
CONFIG_UORB_ALERT=y
CONFIG_UORB_ERROR=y
CONFIG_UORB_WARN=y
CONFIG_UORB_INFO=y
CONFIG_SENSORS_GOLDFISH_SENSOR=y
CONFIG_SENSORS_GNSS=y
CONFIG_SENSORS_GOLDFISH_GNSS=y
CONFIG_SENSORS_GNSS_RECV_BUFFERSIZE=4096
CONFIG_BATTERY_CHARGER=y
CONFIG_BATTERY_GAUGE=y
CONFIG_BATTERY_MONITOR=y
CONFIG_GOLDFISH_BATTERY=y
CONFIG_SYSTEM_BATTERYDUMP=y
```

### 关键 CONFIG 说明

- **`SENSORS=y`**：传感器驱动框架总开关。
- **`UORB=y`**：uORB（Micro Object Request Broker）订阅/发布框架，是 openvela 传感器数据的标准上行通道。
- **`USENSOR=y`**：用户态传感器接口。
- **`SENSORS_GOLDFISH_SENSOR/GNSS/BATTERY`**：goldfish 模拟传感器，移植时替换为目标板真实传感器驱动（如 `SENSORS_BMI160=y`、`SENSORS_LIS2DH=y`）。

---

## 十、桶 8：调试与诊断（强烈建议）

**移植阶段必开**。这些 CONFIG 让符号、错误信息、子系统日志全部可见，能让你在板子起不来时快速定位到具体子系统。

```
CONFIG_DEBUG_FEATURES=y
CONFIG_DEBUG_SYMBOLS=y
CONFIG_DEBUG_CUSTOMOPT=y
CONFIG_DEBUG_OPTLEVEL="-O3"
CONFIG_ALLSYMS=y
CONFIG_DEBUG_NET=y
CONFIG_DEBUG_NET_ERROR=y
CONFIG_DEBUG_PCI=y
CONFIG_DEBUG_PCI_ERROR=y
CONFIG_DEBUG_PCI_INFO=y
CONFIG_DEBUG_PCI_WARN=y
CONFIG_DEBUG_UORB=y
CONFIG_DEBUG_VIRTIO=y
CONFIG_DEBUG_VIRTIO_ERROR=y
```

### 调试功能补充建议

- **板子不起来 / 卡在某处**：建议加上 `CONFIG_SCHED_BACKTRACE=y`（已在桶 3 启用），出现 panic 时能打出函数调用栈。
- **进一步排查**：可以添加 `CONFIG_DEBUG_SCHED=y`、`CONFIG_DEBUG_MM=y`、`CONFIG_DEBUG_FS=y` 等子系统级调试开关。
- **release 版本**：移植稳定后再删掉 `DEBUG_*=y` 系列以缩小固件、提升性能。

---

## 十一、其他实用项（adbd、PCI、虚拟化）

```
CONFIG_SYSTEM_ADBD=y
CONFIG_ADBD_QEMU_SERVER=y
CONFIG_ADBD_FILE_SERVICE=y
CONFIG_PCI=y
CONFIG_PCI_MSIX=y
CONFIG_PCI_QEMU_EDU=y
CONFIG_PCI_QEMU_TEST=y
CONFIG_DRIVERS_VIRTIO_PCI=y
CONFIG_DRIVERS_VIRTIO_BLK=y
CONFIG_DRIVERS_VIRTIO_RNG=y
CONFIG_DEV_URANDOM=y
CONFIG_EVENT_FD=y
CONFIG_TIMER_FD=y
CONFIG_SYSTEM_TIME64=y
CONFIG_SYSTEM_SYSTEM=y
CONFIG_SYSTEM_SYSTEM_STACKSIZE=8192
CONFIG_ALLOW_BSD_COMPONENTS=y
CONFIG_GNSSUTILS_MINMEA_LIB=y
```

- **`SYSTEM_ADBD` + `ADBD_QEMU_SERVER`**：goldfish 通过 adbd 提供 `adb shell` 接入，**真实硬件常用 USB Gadget ADB（`SYSTEM_ADB=y`、`USBADB=y`）替代**。
- **`PCI` / `DRIVERS_VIRTIO_*`**：QEMU 虚拟化总线，真实硬件无需。
- **`SYSTEM_TIME64=y`**：64 位时间戳，**强烈建议保留**以避免 2038 问题。

---

## 十二、新硬件适配快速 Checklist

参赛者完成 BSP 移植后，建议对照以下 Checklist 检查 defconfig：

- [ ] **架构与板级标识**已替换为目标硬件（桶 1）
- [ ] **串口控制台驱动**已切换到目标芯片对应驱动（桶 1）
- [ ] **NSH shell 入口**`SYSTEM_NSH=y` 与 `INIT_ENTRYPOINT="nsh_main"` 已开启（桶 2）
- [ ] **任务栈与 idle 栈**已根据真实 RAM 容量调整（桶 3，goldfish 4 MiB 仅供参考）
- [ ] **`FS_PROCFS=y`** 已开启，便于 runtime 调试（桶 4）
- [ ] **网络/图形/传感**按硬件能力按需启用（桶 5/6/7），不需要的整桶关掉
- [ ] **`DEBUG_FEATURES=y` + `ALLSYMS=y` + `DEBUG_SYMBOLS=y`** 已开启（桶 8）
- [ ] **`SYSTEM_TIME64=y`** 保留以避免 2038 问题
- [ ] 移除所有以 `GOLDFISH_*` / `VIRTIO_*` / `PCI_*` / `MULTIBOOT*` / `X86_64_*` 为前缀、与目标硬件无关的项

通过 Checklist 后即可执行 `./build.sh <vendor>:<config> -j` 编译，串口接通后看到 `openvela-ap> ` 提示符即代表 L0 基线启用成功。

---

## 十三、参考资料

| 资源 | 说明 |
| ---- | ---- |
| [完整 goldfish-x86_64-ap defconfig](https://github.com/open-vela/manifests/blob/dev-ai-contest-2026/) | 在 manifests 仓库 sync 后位于 `vendor/openvela/boards/vela/configs/goldfish-x86_64-ap/defconfig` |
| [goldfish-arm64-v8a-ap defconfig](https://github.com/open-vela/manifests/blob/dev-ai-contest-2026/) | arm64 架构同构变体（`vendor/openvela/boards/vela/configs/goldfish-arm64-v8a-ap/defconfig`） |
| [goldfish-armeabi-v7a-ap defconfig](https://github.com/open-vela/manifests/blob/dev-ai-contest-2026/) | armv7-a 架构变体，**额外启用了 BLE / Wi-Fi / Audio 等手机栈**，可作为后续子系统 CONFIG 参考 |
| [openvela 芯片移植指南](../../../chip_porting/porting_guide.md) | 从零开始的 BSP 移植完整流程 |
| [新硬件适配赛道详细指引](../hardware_porting_track_guide.md) | 赛道说明、评分维度与参考资源 |
| [Kconfig 使用指南](../../../device_dev_guide/build/Kconfig.md) | menuconfig / defconfig / .config 的关系详解 |

> **后续文档计划**：基于本 L0 基线，后续会陆续补充以下子系统的增量 CONFIG 文档：**Bluetooth/Wi-Fi**、**Audio**、**Power Management**、**MTD/MMC 存储**。这四块在 `goldfish-x86_64-ap` 中**未启用**，但 `goldfish-armeabi-v7a-ap` 已包含完整范例可供参考。
