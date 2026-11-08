# Redmi 12C (earth) 充电时触控失灵 —— 内核修复

## 一、根因（已在设备上实测确认）

触控 IC 是 Novatek NT36672 系列（PID 0x5970，FW 0x12）。内核里小米的 NVT 驱动在
**充电时**会给 IC 下发一条 host command `0x53`（charger mode / 抗干扰模式）。

这颗 IC 接受 `0x53` 之后**不再拉 INT 中断线**，于是：

- 芯片仍在扫描、SPI 仍能读到手指数据（`/proc/nvt_data_dump` 里能看到指纹信号，峰值 662）
- 但 `/proc/interrupts` 里 `NVT-ts` 计数**完全不增长**，`getevent` 收不到任何事件
- 上层 InputDispatcher 正常（`DispatchFrozen=false`，注入 Home 键有效）

所以现象是"能显示、完全没触控"。开关屏（`nvt_ts_resume` 重新初始化 IC）或插拔数据线
（notifier 再下一次命令）就能恢复 —— 与用户描述完全一致。

手动把 charger 模式置 1 必死、置 0 必活，因果已闭环。

## 二、修复做法

**恒不下发 charger mode，永远让 IC 处于正常模式（0x51）。**

补丁 `patches/0001-nvt-disable-charger-mode.patch` 改两处：

| 文件 | 改动 |
|---|---|
| `nt36xxx.c` (`nvt_ts_resume`) | 删掉 `mt_get_charger_type()` 分支，无条件 `nvt_set_charger_switch(0)` |
| `nt36xxx_ext_proc.c` (`/proc/nvt_charger_switch` write) | 任何想写 1 的请求都被忽略，强制 0 |

## 三、为什么用官方 `earth-s-oss` 源码，而不是本仓库的 `ksu-next` 树

设备上两个槽位实际运行的**都是小米官方内核**：

| 分区 | 内核版本 | 编译者 |
|---|---|---|
| `boot_a`（当前，配 AOSP GSI） | `4.19.191-perf-gc92376a393a7` | `builder@pangu-build-component-...` |
| `boot_b` | `4.19.191-perf-01037-g7ca6f47236e2` | `builder@non-pangu-pod-7...` |

`vendor/lib/modules/` 下 13 个 `.ko`（wlan、bt、gps、fm、wmt、met、fpsgo…）是按
`4.19.191-perf-gc92376a393a7` 编的。所以：

- 用本仓库 `ksu-next`（**4.19.325**）编出来的内核，vermagic 是
  `4.19.325-perf-...`，与 vendor 模块不匹配 → **WiFi/蓝牙/GPS/FM 全部加载失败**
- 用官方 `earth-s-oss`（**4.19.191**）源码编，vermagic 可以精确对齐

工作流里做了两件事保证能直接刷：

1. `CONFIG_LOCALVERSION="-perf-gc92376a393a7"` + 关掉 `LOCALVERSION_AUTO`，
   并在编译前用 `make kernelrelease` 断言必须精确等于
   `4.19.191-perf-gc92376a393a7`
2. 关掉 `CONFIG_MODVERSIONS`（官方源码里默认不开），这样内核不做符号 CRC 校验，
   只校验 vermagic 字符串 —— 避免与 Dec-2023 构建的 vendor 模块 CRC 不一致

## 四、产物

GitHub Actions 产出 `earth-kernel-touchfix`：

- `AnyKernel3-earth-touchfix.zip` —— 推荐，TWRP / Magisk 里刷，自动保留现有 ramdisk、自动处理 A/B 槽
- `Image.gz-dtb` —— 内核镜像（自备打包方式时用）
- `build.log`、`kernelrelease.txt`

## 五、刷入与回滚

```
# 刷入（任选其一）
TWRP: 安装 AnyKernel3-earth-touchfix.zip
# 或 fastboot 方式自行打包 boot.img 后
fastboot flash boot_a boot.img
```

回滚：原厂 boot_a 已备份在 `12c-backup/partition_backups/boot_a.img`，
`dd` 回去即可（注意该备份里是更早的 `01045` 内核，如需回到当前 `gc92376a` 版本，
应使用本次编译前从设备 dd 出来的 `kernel-check/cur_boot_a.img`）。

## 六、验证方法

刷入后：

```bash
uname -a                      # 应为 4.19.191-perf-gc92376a393a7
su -c 'cat /proc/nvt_charger_switch'   # 应为 0
su -c 'cat /proc/interrupts | grep NVT' # 插着充电器滑动，计数必须增长
```

插着充电器滑动屏幕，触控应当完全正常；dmesg 里会出现
`earth-fix: charger_switch forced 0 (charger mode disabled)`。
