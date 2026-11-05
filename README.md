# Kupfer Linux（Arch Linux ARM）· 一加6 · Plasma Mobile 完整版

> 可刷写镜像与说明 · 构建日期 2026-11-04 02:36 CST · 构建主机 Arch Linux WSL2（x86_64 交叉构建 aarch64）

## 1. 系统概览

| 项目 | 内容 |
| --- | --- |
| 设备 | OnePlus 6（enchilada，2018，SDM845，1080×2280） |
| 发行版 | Kupfer Linux = Arch Linux ARM + Kupfer 设备包/内核包 |
| 内核 | `linux-sdm845 7.1.0-1`（设备树 `qcom/sdm845-oneplus-enchilada`） |
| 桌面 | Plasma Mobile 6.8.2-1 **完整版**（Kupfer `flavour-plasma-mobile` 包全集） |
| 图形栈 | KWin Wayland + XWayland（`xorg-xwayland`）+ Qt6 6.12.0-3 |
| 语言 | 简体中文 `zh_CN.UTF-8`，KDE/Plasma/Qt 中文翻译 |
| 字体 | Noto Sans/Serif CJK、思源黑体/思源宋体 CN、文泉驿正黑、Noto Color Emoji |
| 输入法 | **未安装 fcitx5**；Plasma 自带键盘 `plasma-keyboard`（内置 Qt Virtual Keyboard 拼音） |
| 默认账户 | `arch` / `arch`（wheel 组、sudo 可用） |
| root | 密码 `arch`，SSH 已禁止 root 密码登录 |
| SSH | `openssh` 已安装，`sshd.service` 开机自启（自动拉起 `sshdgenkeys.service` 生成主机密钥） |
| 时区 / 主机名 | `Asia/Shanghai` / `enchilada` |
| 开机流程 | systemd 直接启动 `plasma-mobile.service`（别名 display-manager，tty1） |

## 2. 文件清单

| 文件 | 说明 |
| --- | --- |
| `boot.img`（44M） | Android boot 镜像：内核 `Image.gz` + 追加 DTB + initramfs（zstd） |
| `kupfer-enchilada-plasma-mobile.img`（7.4 GB） | 整盘镜像：MBR / 4K 扇区，`p1=kupfer_boot`(ext2 128MiB) + `p2=kupfer_root`(ext4) |
| `SHA256SUMS` | 镜像校验和 |
| `flash-linux.sh` / `flash-windows.bat` | 一键刷机脚本（Linux/macOS 与 Windows） |
| `README.md` | 本说明 |
| `BUILD.md` | 构建记录、版本清单与刷写验证记录 |

## 3. 刷机前准备

1. **解锁 bootloader**（会清空手机全部数据）：设置 → 关于手机 → 连点“版本号” → 开发者选项 → 打开“OEM 解锁”和“USB 调试” → 关机后按住 **音量上 + 电源** 进入 fastboot → `fastboot oem unlock`（屏幕确认）。
2. 电脑安装 platform-tools（`fastboot`/`adb`）。
3. **备份数据**：刷写 `userdata` 分区会清除原 Android 数据。
4. 电量建议 50% 以上，使用质量可靠的 USB 数据线。

## 4. 刷入

### 一键脚本

- Linux/macOS：`chmod +x flash-linux.sh && ./flash-linux.sh`
- Windows：双击 `flash-windows.bat`（需 fastboot.exe 在 PATH 中）

### 手动命令（一加6 的 boot 是 A/B 双槽）

```bash
fastboot devices
fastboot flash boot_a boot.img
fastboot flash boot_b boot.img
fastboot -S 100M flash userdata kupfer-enchilada-plasma-mobile.img
fastboot reboot
```

> - 一加6 的 **boot 是 A/B 双槽**，两个槽都要刷，否则可能从旧槽启动。
> - `-S 100M` 让 fastboot 按 100MB 分块稀疏刷写（本镜像 7.4GB，实际发送 55 块，约 5 分钟），**必须保留**。
> - 只刷 `boot_a`/`boot_b` 与 `userdata`；**绝对不要**刷 `aboot` 分区（会变砖）。
> - **Windows 实测**：platform-tools **35+ 刷 7.4GB 大镜像会报 `std::bad_alloc`**，请使用 **33.0.0 / 34.x** 的 `fastboot.exe`（本次即用 33.0.0 成功）；Linux 下一般无此问题。

## 5. 首次启动

> 本镜像已在一加6 真机完成刷写并成功启动（2026-11-06，记录见 [BUILD.md](BUILD.md) §9）。

1. 首次开机约 3–10 分钟：initramfs 会在 userdata 内找到 `LABEL=kupfer_root` 的分区，并自动把分区与文件系统扩容到整块 userdata（Kupfer `rootfsresize` hook）。
2. 进入 Plasma Mobile 中文界面，按向导连接 Wi-Fi、设置时间等。
3. 默认账户 `arch`，密码 `arch`；root 密码同为 `arch`（仅本地/串口维护用）。

## 6. 中文与输入法（重点）

- 系统 locale 为 `zh_CN.UTF-8`，Plasma/KDE/Qt 程序自动显示中文翻译。
- 虚拟键盘使用 Plasma 自带 **plasma-keyboard**（非 fcitx5，非 Maliit）：
  - `/etc/xdg/kwinrc` 中 `[Wayland] InputMethod` 指向 `org.kde.plasma.keyboard.desktop`，`VirtualKeyboardEnabled=true`，由 KWin 通过 input-method 协议启动；
  - 中文布局：`/usr/share/plasma/keyboard/layouts/zh_CN/main.qml`，使用 Qt Virtual Keyboard 的 `PinyinInputMethod`（拼音候选栏）。
- **Qt 程序**：Wayland 原生 Qt 通过 `text-input-v3` 接收提交文本，可直接输入中文。
- **X11 程序（XWayland）**：X11 客户端没有 text-input 协议，KWin 6.8.2-1 会把输入法提交的文本转换为 keysym，通过“临时键位映射 + 假输入”注入焦点窗口，因此 X11 程序同样可以输入中文（已安装 `xorg-xwayland`）。
- 未安装 fcitx5，也未设置 `QT_IM_MODULE`/`GTK_IM_MODULE`，避免与 Plasma 键盘冲突。
- 切换布局：设置 → 键盘 → 布局（或修改 `~/.config/plasmakeyboardrc` 的 `enabledLocales`）。

## 7. SSH / 远程登录

- **推荐：在手机桌面里连接 Wi-Fi**，然后 `ssh arch@<手机IP>`（密码 `arch`；可用 `ip a` 或路由器后台查 IP）。
- **USB 网络共享（Linux/macOS 主机）**：设备 `usb0` 固定为 `172.16.42.1/24`，插线后主机侧执行
  `sudo ip addr add 172.16.42.2/24 dev usb0 && sudo ip link set usb0 up`，然后 `ssh arch@172.16.42.1`。
- **Windows 11 注意**：系统默认没有 RNDIS 驱动，设备管理器里 `RNDIS` 会显示黄色感叹号（代码 28），
  因此 Windows 下 USB 网络不可直接用；请在手机里连 Wi-Fi，或用管理员安装 RNDIS 驱动，或改用 Linux 主机。
- `PermitRootLogin no`，root 不能通过 SSH 密码登录；需要时在设备上 `su -` 或 `sudo -i`。

## 8. 已知情况与排错

- 镜像约 7.4 GB（其中含约 1.2 GB 空闲空间）；首次启动后 `rootfsresize` 会把分区扩到 userdata 的全部容量。
- Kupfer 移动应用 `plasma-dialer`、`plasma-phonebook`、`hfd-service`（触感/振动反馈）、`spacebar`、`plasma-mobile-sounds` 在 Arch Linux ARM 官方仓库（core/extra/alarm/aur）中不可得，故未安装；Plasma Mobile 桌面本体与其余应用（kclock、kweather、angelfish、kalk、krecorder、qmlkonsole、index-fm、kdeconnect、discover、koko、elisa 等）均已安装。
- 本镜像已在 OnePlus 6 真机刷写并启动（2026-11-06）：设备枚举为 USB 复合设备 `1d6b:0104`（序列号 `KUPFER`、RNDIS 功能），说明内核、initramfs（rootfsdetect/rootfsresize）、rootfs、systemd 与 usb-tethering 均已正常启动。刷写细节见 [BUILD.md](BUILD.md) §9。
- 首次启动需要扩容 rootfs，期间（约 1–3 分钟）屏幕可能停留在启动画面，属正常。
- 黑屏/卡启动：确认 bootloader 已解锁；用 `fastboot boot boot.img` 排查；串口日志为 `ttyMSM0 115200`。
- 调制解调器、摄像头、传感器等功能取决于 mainline sdm845 内核与固件成熟度。
- `waydroid` 已安装但未初始化 Android 镜像，需要时执行 `sudo waydroid init`（需网络）。
- 回退 Android：使用一加官方 OxygenOS 全量包线刷恢复。

## 9. 校验

```bash
sha256sum -c SHA256SUMS
```
