# 构建记录 · Kupfer Linux（一加6 / Plasma Mobile 完整版）

## 1. 构建环境

| 项目 | 内容 |
| --- | --- |
| 宿主 | Windows + WSL2 Arch Linux（`6.18.33.2-microsoft-standard-WSL2`，x86_64，16 vCPU，7GB RAM） |
| 交叉执行 | `qemu-user-static-bin` + `binfmt_misc`（aarch64 透明执行） |
| 构建窗口 | 2026-11-04 02:17:21 → 2026-11-04 02:36 |
| 构建目录 | WSL 内 `/build`（rootfs、cache、prebuilts、out、logs） |

## 2. 软件源

| 来源 | 说明 |
| --- | --- |
| Arch Linux ARM aarch64 rootfs | `ArchLinuxARM-aarch64-latest.tar.gz`（791M） |
| Kupfer 预编译包 | `gitlab.com/kupfer/packages/prebuilts`，branch `dev`，commit `d5f9613` |
| Kupfer 仓库 | `main`、`boot`、`device`、`firmware`、`linux`（本地 file:// 挂载，避免 GitLab 抖动） |
| ALARM 镜像 | 清华 TUNA（主）、中科大 USTC（备）：`core` / `extra` / `alarm` / `aur` |
| Kupfer 包定义参考 | `gitlab.com/kupfer/packages/pkgbuilds`（flavour-plasma-mobile 包清单） |

## 3. 构建步骤

| 步骤 | 脚本 | 内容 |
| --- | --- | --- |
| 1 | `01-bootstrap.sh` | 解压 ALARM rootfs；写 pacman 源；`pacman-key`；`pacman -Syu`；移除原生 `linux-aarch64`；安装 `linux-sdm845`、设备包、Plasma Mobile 完整版、中文字体、SSH、Qt/X11 组件 |
| 2 | `02-configure.sh` | `zh_CN` locale、时区、`arch/arch`、sudo、SSH 自启、`plasma-mobile.service`、`kwinrc` 虚拟键盘、fontconfig、NetworkManager/systemd-resolved、`mkinitcpio -P`（生成 initramfs 与 `/boot/aboot.img`） |
| 3 | `03-image.sh` | 生成 4K 扇区 MBR 镜像（128MiB boot + 其余 root）、`mkfs.ext2/ext4`、rsync、导出 `boot.img`、SHA256 |
| 4 | `04-finalize.sh` | 渲染文档、复制产物到桌面 `Archlinux` 文件夹、校验 |

## 4. 关键参数

### deviceinfo（Kupfer `device-sdm845-oneplus-enchilada`）

```ini
deviceinfo_codename="oneplus-enchilada"
deviceinfo_flash_method="fastboot"
deviceinfo_partitions_data="/dev/sda17"
deviceinfo_rootfs_image_sector_size="4096"
deviceinfo_kernel_cmdline="console=ttyMSM0,115200"
deviceinfo_append_dtb="true"
deviceinfo_flash_pagesize="4096"
deviceinfo_flash_offset_base="0x00000000"
deviceinfo_flash_offset_kernel="0x00008000"
deviceinfo_flash_offset_ramdisk="0x01000000"
deviceinfo_flash_offset_second="0x00f00000"
deviceinfo_flash_offset_tags="0x00000100"
```

### boot.img（Android boot header v0）

- kernel：`/boot/Image.gz` + 追加 `dtbs/qcom/sdm845-oneplus-enchilada.dtb`
- ramdisk：`/boot/initramfs-linux.img`（zstd 压缩）
- cmdline：`kupfer earlymodules= console=tty0 console=ttyMSM0,115200`

### rootfs 磁盘镜像

- 分区表：MBR，逻辑扇区 4096 字节（与设备 UFS 一致）
- `p1`：ext2，label `kupfer_boot`，128MiB
- `p2`：ext4，label `kupfer_root`，块大小 4096，`-O ^metadata_csum -m 1`（inode 按默认比例）
- 镜像大小：7.4 GB（7,395,606,528 字节；其中约 1.2 GB 为空闲空间，首次启动自动扩容到 userdata 全部容量）
- 首次启动：Kupfer `rootfsdetect` 在 userdata 内找到 `LABEL=kupfer_root` 子分区；`rootfsresize` 将其扩容到整块 userdata 并 `resize2fs`

## 5. 主要软件版本

| 软件包 | 版本 |
| --- | --- |
| `linux-sdm845` | 7.1.0-1 |
| `plasma-mobile` | 6.8.2-1 |
| `plasma-workspace` | 6.8.2-1 |
| `kwin` | 6.8.2-1 |
| `plasma-keyboard` | 6.8.2-1 |
| `plasma-settings` | 26.08.2-1 |
| `plasma-nm` | 6.8.2-1 |
| `plasma-pa` | 6.8.2-1 |
| `kscreen` | 6.8.2-1 |
| `kscreenlocker` | 6.8.2-1 |
| `powerdevil` | 6.8.2-1 |
| `kclock` | 26.08.2-2 |
| `kweather` | 26.08.2-2 |
| `angelfish` | 26.08.2-1 |
| `qmlkonsole` | 26.08.2-1 |
| `kdeconnect` | 26.08.2-3 |
| `qt6-base` | 6.12.0-3 |
| `qt6-wayland` | 6.12.0-1 |
| `qt6-virtualkeyboard` | 6.12.0-1 |
| `xorg-xwayland` | 24.1.14-1 |
| `pipewire` | 1:1.6.10-1 |
| `wireplumber` | 0.5.18-1 |
| `networkmanager` | 1.58.1-3 |
| `systemd` | 262.1-1 |
| `openssh` | 10.6p1-2 |
| `mesa` | 1:26.2.6-1 |
| `noto-fonts-cjk` | 20240730-1 |
| `adobe-source-han-sans-cn-fonts` | 2.005-2 |

## 6. 产物校验

```text
1cd5f2f7fa26410161be178d2ffae0ea58d89f4265f320f309cd041a6789629d  boot.img
140ac760aaba7f18682dc0a0c2244ef1599ed475fd8660e3e1ca6b827fcbc945  kupfer-enchilada-plasma-mobile.img
```

## 7. 重新构建

1. 准备 WSL 环境：`pacman -S base-devel arch-install-scripts qemu-user-static-bin rsync e2fsprogs parted dosfstools`
2. 准备 `/build/cache/ArchLinuxARM-aarch64-latest.tar.gz`、包缓存与 `/build/prebuilts`（Kupfer prebuilts 克隆）
3. 依次执行 `01-bootstrap.sh`（可 `REUSE_ROOTFS=1` 续跑）、`02-configure.sh`、`03-image.sh`、`04-finalize.sh`

## 8. 已知限制

- 已于 2026-11-06 在 OnePlus 6 真机刷写并成功启动（见 §9）。
- 镜像未做 AVB 签名，必须解锁 bootloader。
- 首次启动会生成 machine-id 与 SSH host key。
- 调制解调器/摄像头/传感器支持取决于 mainline sdm845 内核与固件。
- `plasma-dialer`、`plasma-phonebook`、`hfd-service`、`spacebar`、`plasma-mobile-sounds` 不在 ALARM 官方仓库，构建时被跳过（构建日志已清理）。
- ext4 初始用 `-N 100000` 时因文件数超出导致 ENOSPC，最终改为默认 inode 比例。

## 9. 实机刷写与验证记录（2026-11-06）

### 设备状态

| 项目 | 值 |
| --- | --- |
| 序列号 | `f6721003` |
| `getvar product` | `sdm845` |
| bootloader 解锁 | `unlocked: yes` |
| `current-slot` | `a`（A/B 设备） |
| `partition-size:boot` | `0x4000000`（64MiB） |
| `partition-size:userdata` | `0x1B800BB000`（约 110GiB） |

### 实际刷写命令

```text
fastboot flash boot_a boot.img        # 44,548 KB，1.97s
fastboot flash boot_b boot.img        # 44,548 KB，1.68s
fastboot -S 100M flash userdata kupfer-enchilada-plasma-mobile.img
fastboot reboot
```

- fastboot 版本：**platform-tools 33.0.0-8141338**（Windows）
- userdata 稀疏分块：**55/55 块**，每块 ~100MB，总耗时 **315 秒**
- 写入大小：7,395,606,528 字节镜像（其中约 1.2GB 为零/空闲，因此稀疏后约 5.5GB 实际传输）

### 启动验证（无屏幕/串口条件下的证据）

重启后设备离开 fastboot；Windows 枚举到 USB 复合设备：

```text
USB\VID_1D6B&PID_0104\KUPFER        product="Arch Linux ARM", manufacturer="kupfer", serial="kupfer"
USB\VID_1D6B&PID_0104&MI_00         RNDIS 接口
```

该 gadget 由镜像内 `/usr/bin/usb-tethering`（kupfer `usb-tethering.service`）在启动时通过 configfs 创建，
因此可确认：**内核已启动 → initramfs 找到并挂载 `kupfer_root` → systemd 已启动 → usb-tethering 服务已运行**。

### 刷写过程中遇到的问题与解决

1. **platform-tools 37.0.0 在 Windows 刷 7.4GB 镜像时报 `std::bad_alloc`**（带 `-S 100M` 和不带均失败，日志仅一行 `libc++abi: terminating due to uncaught exception of type std::bad_alloc`）。
   → 换用 **33.0.0** 的 `fastboot.exe` 后正常；脚本已注明该限制。
2. **一加6 boot 为 A/B 双槽**：`fastboot flash boot` 只写当前槽（`boot_a`），已补刷 `boot_b`；脚本已改为双槽刷写。
3. **Windows 11 无 RNDIS 驱动**（设备管理器代码 28）：`netrndis.inf` 只匹配 `ms_rndisusb`，`rndiscmp.inf` 只匹配 `USB\Class_EF&SubClass_04&Prot_01`，而 Linux gadget 暴露的是 `Class_02/06/00`，因此 Windows 无法绑定驱动，USB 网络 SSH 不可用。Linux 主机可原生识别（`usb0` = 172.16.42.1/24）。
   → 本次通过 USB gadget 枚举确认启动成功；日常使用建议在手机 UI 里连 Wi-Fi 后 SSH。后续如有需要，可在镜像中增加 CDC-ACM 串口控制台（Windows 自带 `usbser.sys` 驱动），即可在 Windows 上直接获得串口 shell。
