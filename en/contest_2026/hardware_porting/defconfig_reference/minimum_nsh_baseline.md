# Minimum Bootable NSH System defconfig Reference

\[ English | [简体中文](../../../../zh-cn/contest_2026/hardware_porting/defconfig_reference/minimum_nsh_baseline.md) \]

> This document is written for participants in the **Hardware Porting Track**. Starting from the already-supported **goldfish-x86_64-ap** emulator board, it walks through the CONFIG options required to "boot to an NSH prompt + network + display + sensors" on a new piece of hardware.
>
> When you write a defconfig for a new board, treat this document as a "minimum-working" baseline checklist: first confirm all mandatory items are enabled, then trim or extend on demand.

## 1. Why goldfish as the baseline

openvela has been ported to the [Android Goldfish](https://source.android.com/docs/core/runtime/instr) emulator, a **purely virtual hardware platform**: CPU, peripherals, sensors, networking and display are all provided by QEMU. This means:

1. **Zero hardware barrier**: contestants can boot it up without any physical board, making it the fastest entry point for verifying openvela system behavior.
2. **Most complete subsystem coverage**: a single goldfish defconfig already enables the NSH shell, networking stack (TCP/UDP/PING/Telnet/iperf), graphics (LVGL + framebuffer + touch + camera), sensors (uORB + GNSS + battery), debugging (DEBUG_FEATURES + ALLSYMS), ADBD, and more — all in one shot.
3. **Three architectures, same skeleton**: the repository ships three structurally identical defconfigs — `goldfish-x86_64-ap`, `goldfish-arm64-v8a-ap`, `goldfish-armeabi-v7a-ap`. Contestants can copy the "non-architecture-specific" options verbatim to a new board and only swap out the architecture identifiers.

> **Full defconfig path**: `vendor/openvela/boards/vela/configs/goldfish-x86_64-ap/defconfig` (253 lines total). This document selects ~80 key items and explains them in groups; the rest can be looked up directly in the source file.

## 2. Baseline overview: 8 functional buckets

`goldfish-x86_64-ap` contains 246 effective CONFIG entries, which fall into 8 functional categories:

| Bucket | Count | Required for new HW? | Description |
| ---- | ---- | ---- | ---- |
| 1. Architecture & board identity | 37 | ✅ Required (replace per target arch) | Determines toolchain, entry point, memory layout |
| 2. NSH shell & console | 13 | ✅ Required | Without it, a booted kernel has no interactive UI |
| 3. Scheduling & C/C++ runtime | 32 | ✅ Required | TLS, stack, worker threads, libc/libcxx |
| 4. Filesystem & procfs | 10 | ✅ Strongly recommended | At least PROCFS for debugging |
| 5. Networking | 51 | ⭕ Optional | Disable all if no networking is needed |
| 6. Graphics & input | 27 | ⭕ Optional | Disable all if no display is present |
| 7. Sensors & uORB | 16 | ⭕ Optional | Depends on whether the board has sensors |
| 8. Debug & diagnostics | 14 | ✅ Strongly recommended | Always enable during porting |

Each bucket is detailed below. **A `+` prefix in front of a CONFIG indicates "an extra dependency that must be added on top of the existing defconfig"** (a community convention).

---

## 3. Bucket 1: Architecture & board identity (Required)

This is the **only** part of the entire defconfig that **must be replaced for the target hardware**. The other seven buckets are largely architecture-agnostic.

### Key CONFIGs

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

### Serial console (goldfish uses 16550 UART)

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

### How to adapt to new hardware

| Item | Replacement principle |
| ---- | ---- |
| `ARCH=` / `ARCH_X86_64=y` | Change to the target CPU arch (`arm` / `arm64` / `risc-v` etc.) |
| `ARCH_CHIP=` / `ARCH_CHIP_<NAME>=y` | Change to the target SoC, matching `nuttx/arch/<arch>/src/<chip>/` |
| `ARCH_BOARD_CUSTOM_*` | Point to your new vendor directory |
| `16550_UART*` | Replace with the target chip's UART driver (e.g. `STM32_USART1=y` for STM32) |
| `RAM_SIZE` | Replace with the actual RAM size of the target board |
| `BOARD_LOOPSPERMSEC` | Tied to CPU frequency — fill 999 first, calibrate later |

> See [openvela Chip Porting Guide](../../../chip_porting/porting_guide.md) for the complete porting workflow.

---

## 4. Bucket 2: NSH shell & console (Required)

NSH (NuttShell) is openvela's command-line entry point. **Without it the system has no usable UI after boot.**

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

### What each CONFIG does

- **`SYSTEM_NSH` / `INIT_ENTRYPOINT="nsh_main"`**: registers NSH as the first userland task — the prompt appears immediately after boot.
- **`NSH_ARCHINIT`**: lets NSH invoke `board_late_initialize()` before showing the prompt, useful for board-driver registration.
- **`NSH_BUILTIN_APPS`**: allows applications under `apps/` to be invoked as built-in NSH commands (`hello`, `iperf`, `ping`, etc.).
- **`NSH_PROMPT_STRING`**: the prompt string — change it per board (e.g. `"openvela-mynew> "`).
- **`SYSTEM_CLE_CMD_HISTORY`**: enables Up/Down arrow command history — a major dev-experience boost.
- **`TTY_SIGINT` / `TTY_SIGTSTP`**: makes Ctrl-C / Ctrl-Z actually interrupt/suspend the foreground task.

### Truly minimal subset

If you only want to verify "NSH starts", out of bucket 2's 14 items, only **`SYSTEM_NSH`, `INIT_ENTRYPOINT`, `NSH_ARCHINIT`, `BUILTIN`** are strictly indispensable.

---

## 5. Bucket 3: Scheduling & C/C++ runtime (Required)

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

### Notes

- **`DEFAULT_TASK_STACKSIZE=4 MiB`** and **`IDLETHREAD_STACKSIZE=4 MiB`** are tuned for the goldfish emulator (256 MiB RAM). **Drop these drastically when porting to RAM-constrained hardware** — typical values are `4096 ~ 16384`.
- **`SCHED_TICKLESS=y`** enables tickless scheduling, paired with `USEC_PER_TICK=1` for microsecond time precision — but requires the chip to provide a high-resolution alarm timer. If unavailable, drop these three to fall back to the legacy ticked mode.
- **`SCHED_HPWORK` / `SCHED_LPWORK`**: high/low-priority worker threads, widely used by the driver framework (input, sensors, networking). **Disabling them breaks many drivers**.
- **`LIBCXX` + `CXX_EXCEPTION/RTTI/WCHAR`**: full C++ ABI support — required by openvela LVGL, Camera, etc. May be disabled on extremely lean firmware, but you risk dependency breakage.

---

## 6. Bucket 4: Filesystem & procfs (Strongly recommended)

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

### Strongly recommended to keep

- **`FS_PROCFS=y`**: mounting `/proc` lets you `cat /proc/meminfo`, `cat /proc/<pid>/status` to diagnose issues — a lifesaver during porting.
- **`PSEUDOFS_FILE` + `PSEUDOFS_SOFTLINKS`**: support virtual files and soft links in pseudo filesystems (`/dev/console`, etc.).
- **`FS_FAT=y`**: generic FAT filesystem, used for SD cards and similar storage.

### goldfish-specific, can be dropped on other boards

- **`FS_V9FS` + `V9FS_VIRTIO_9P`**: mounts host directories into the VM via virtio-9p — **only useful in QEMU**, drop on real hardware.

---

## 7. Bucket 5: Networking stack (Optional)

goldfish ships a complete TCP/IP stack with telnetd, iperf, tftp, webclient. **If your target board has no NIC or networking is not in scope for stage 1, all 51 items can be disabled — significantly shrinking firmware size.**

### Networking core

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

### TCP write buffers and reliability tuning

```
CONFIG_NET_TCP_WRITE_BUFFERS=y
CONFIG_NET_TCP_NWRBCHAINS=128
CONFIG_NET_TCP_DELAYED_ACK=y
CONFIG_NET_TCP_SELECTIVE_ACK=y
CONFIG_NET_TCP_OUT_OF_ORDER_BUFSIZE=32768
CONFIG_NET_RECV_BUFSIZE=32768
CONFIG_NET_SEND_BUFSIZE=32768
```

### IP address (goldfish fixed addresses)

```
CONFIG_NETINIT_THREAD=y
CONFIG_NETINIT_IPADDR=0x0a00020f          # 10.0.2.15
CONFIG_NETINIT_DRIPADDR=0x0a000202        # 10.0.2.2  gateway
CONFIG_NETINIT_DNSIPADDR=0x0a000203       # 10.0.2.3  DNS
CONFIG_NETINIT_DNS=y
CONFIG_NETDB_DNSCLIENT=y
CONFIG_NETDB_DNSCLIENT_ENTRIES=4
```

### NIC driver + IOB buffer pool

```
CONFIG_NET_E1000=y
CONFIG_DRIVERS_VIRTIO_NET=y
CONFIG_IOB_BUFSIZE=2048
CONFIG_IOB_NBUFFERS=8192
CONFIG_IOB_ALIGNMENT=64
CONFIG_IOB_NCHAINS=64
```

### Application-level utilities

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

> **Porting tip**: when moving to a real NIC, replace `NET_E1000` and `DRIVERS_VIRTIO_NET` with your PHY/MAC driver (`STM32_ETHMAC=y`, `ESP32_WIRELESS=y`, etc.); the rest of the networking config can be reused as-is.

---

## 8. Bucket 6: Graphics & input (Optional)

goldfish ships a complete LVGL + framebuffer + touch + camera pipeline. Disable all if the target board has no display.

### Display & framebuffer

```
CONFIG_VIDEO=y
CONFIG_VIDEO_FB=y
CONFIG_VIDEO_STREAM=y
CONFIG_DRIVERS_VIDEO=y
CONFIG_GOLDFISH_GPU_FB=y
CONFIG_GOLDFISH_GPU_FB_PRIORITY=101
CONFIG_GOLDFISH_PIPE=y
```

### LVGL graphics library

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

### Input & camera

```
CONFIG_INPUT=y
CONFIG_INPUT_GOLDFISH_EVENTS=y
CONFIG_UINPUT_TOUCH=y
CONFIG_GOLDFISH_CAMERA=y
CONFIG_SYSTEM_NXCAMERA=y
```

### Demos

```
CONFIG_EXAMPLES_FB=y
CONFIG_EXAMPLES_LVGLDEMO=y
```

> **Porting tip**: replace `GOLDFISH_GPU_FB` / `INPUT_GOLDFISH_EVENTS` / `GOLDFISH_CAMERA` with your real LCD/touch/camera drivers. The LVGL subtree itself can be retained as the upper UI framework.

---

## 9. Bucket 7: Sensors & uORB (Optional)

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

### Key CONFIG explanations

- **`SENSORS=y`**: master switch for the sensor driver framework.
- **`UORB=y`**: the uORB (Micro Object Request Broker) pub/sub framework — openvela's standard upstream channel for sensor data.
- **`USENSOR=y`**: userspace sensor interface.
- **`SENSORS_GOLDFISH_SENSOR/GNSS/BATTERY`**: goldfish virtual sensors — replace with your target board's real sensor drivers when porting (`SENSORS_BMI160=y`, `SENSORS_LIS2DH=y`, etc.).

---

## 10. Bucket 8: Debug & diagnostics (Strongly recommended)

**Always enable during porting.** These CONFIGs make symbols, error messages and per-subsystem logs fully visible — when the board fails to boot, they let you pinpoint the broken subsystem fast.

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

### Additional debugging suggestions

- **Board fails to boot / hangs**: add `CONFIG_SCHED_BACKTRACE=y` (already enabled in bucket 3) — prints the call stack on panic.
- **Deeper investigation**: add `CONFIG_DEBUG_SCHED=y`, `CONFIG_DEBUG_MM=y`, `CONFIG_DEBUG_FS=y` and other subsystem-level switches.
- **Release builds**: once porting stabilizes, drop `DEBUG_*=y` to shrink firmware and improve performance.

---

## 11. Other useful items (adbd, PCI, virtualization)

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

- **`SYSTEM_ADBD` + `ADBD_QEMU_SERVER`**: goldfish offers `adb shell` access via adbd. **Real hardware typically uses USB Gadget ADB instead (`SYSTEM_ADB=y`, `USBADB=y`).**
- **`PCI` / `DRIVERS_VIRTIO_*`**: QEMU virtualization buses — not needed on real hardware.
- **`SYSTEM_TIME64=y`**: 64-bit timestamps. **Strongly recommended to keep** to avoid the 2038 problem.

---

## 12. New-hardware adaptation Checklist

Once you've completed BSP porting, walk through this checklist to validate your defconfig:

- [ ] **Architecture & board identity** replaced with target hardware (bucket 1)
- [ ] **Serial console driver** swapped to the target chip's UART driver (bucket 1)
- [ ] **NSH shell entry** `SYSTEM_NSH=y` and `INIT_ENTRYPOINT="nsh_main"` enabled (bucket 2)
- [ ] **Task and idle stacks** sized to actual RAM (bucket 3 — goldfish 4 MiB is reference only)
- [ ] **`FS_PROCFS=y`** enabled for runtime debugging (bucket 4)
- [ ] **Networking / graphics / sensors** enabled per hardware capability (buckets 5/6/7); disable whole buckets if not applicable
- [ ] **`DEBUG_FEATURES=y` + `ALLSYMS=y` + `DEBUG_SYMBOLS=y`** enabled (bucket 8)
- [ ] **`SYSTEM_TIME64=y`** retained to avoid the 2038 problem
- [ ] All entries prefixed with `GOLDFISH_*` / `VIRTIO_*` / `PCI_*` / `MULTIBOOT*` / `X86_64_*` and unrelated to the target hardware have been removed

After passing the checklist, run `./build.sh <vendor>:<config> -j` to build. If the serial console shows the `openvela-ap> ` prompt, the L0 baseline is working.

---

## 13. References

| Resource | Notes |
| ---- | ---- |
| [Full goldfish-x86_64-ap defconfig](https://github.com/open-vela/manifests/blob/dev-ai-contest-2026/) | After `repo sync` it lives at `vendor/openvela/boards/vela/configs/goldfish-x86_64-ap/defconfig` |
| [goldfish-arm64-v8a-ap defconfig](https://github.com/open-vela/manifests/blob/dev-ai-contest-2026/) | Structurally identical arm64 variant (`vendor/openvela/boards/vela/configs/goldfish-arm64-v8a-ap/defconfig`) |
| [goldfish-armeabi-v7a-ap defconfig](https://github.com/open-vela/manifests/blob/dev-ai-contest-2026/) | armv7-a variant — **adds full BLE / Wi-Fi / Audio (phone-class) stack**, useful as a reference for follow-up subsystem CONFIG docs |
| [openvela Chip Porting Guide](../../../chip_porting/porting_guide.md) | End-to-end BSP porting workflow |
| [Hardware Porting Track Guide (zh-cn)](../../../../zh-cn/contest_2026/hardware_porting/hardware_porting_track_guide.md) | Track description, scoring criteria and reference resources (Chinese only at this stage) |
| [Kconfig Usage Guide](../../../device_dev_guide/build/Kconfig.md) | menuconfig / defconfig / .config relationship explained |

> **Upcoming docs**: building on this L0 baseline, follow-up documents will cover incremental CONFIG sets for these subsystems: **Bluetooth/Wi-Fi**, **Audio**, **Power Management**, **MTD/MMC storage**. These four are **not enabled** in `goldfish-x86_64-ap`, but `goldfish-armeabi-v7a-ap` already contains complete reference examples.
