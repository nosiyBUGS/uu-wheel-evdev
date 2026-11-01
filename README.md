# UU Remote Wheel Fix (evdev / uinput)

在 Android 平板上修复 **UU远程（UU Remote）** 远程会话鼠标滚轮方向反转的问题。

系统级方案：抓取物理鼠标的 evdev 事件，通过 `/dev/uinput` 重新发一个虚拟鼠标。**不依赖 Frida，不需要改主机，开机自启。**

## 背景

在 Xiaomi Pad 5 Pro 12.4（Android 12 / MIUI，arm64-v8a，KernelSU/SukiSU）上，UU远程 远程桌面里的鼠标滚轮方向是反的；平板本地滚动正常。

Frida 钩 `MotionEvent.getAxisValue(AXIS_VSCROLL)` 取反可以临时修复，但需要主机 watcher，重启后容易失效。本项目把修复下沉到 Android 输入层。

## 原理

```
物理鼠标 (BT5.0 Mouse / Xiaomi Mouse)
        │
        │  EVIOCGRAB（独占）
        ▼
   uu_wheel_flip
        │
        │  write /dev/uinput
        ▼
虚拟鼠标 (UU Wheel Flip ...)
        │
        ▼
Android InputReader → UU远程 / 本地 App
```

关键点：**最终验证下来，虚拟鼠标直通（`invert=0`，不翻转）方向就是正确的**；在 evdev 层翻转反而会变成双重取反。因此默认配置是直通。

## 环境

- Android arm64-v8a
- 已 root（KernelSU / SukiSU；Magisk 理论上也可用）
- 内核开启 `CONFIG_INPUT_UINPUT`，存在 `/dev/uinput`
- 测试环境：Xiaomi Pad 5 Pro 12.4（`22081281AC`），Android 12 / SDK 31，SukiSU Ultra 3.2.0

## 构建

需要 [Zig](https://ziglang.org/) 0.13.x（交叉编译）和 `zip`：

```bash
ZIG=/path/to/zig ./build.sh
```

生成：

- `module/uu_wheel_flip`：ARM64 静态二进制
- `dist/uu-wheel-evdev.zip`：KernelSU/Magisk 模块包

手动编译：

```bash
zig cc -target aarch64-linux-musl -static -O2 -o uu_wheel_flip src/uu_wheel_flip.c
```

## 安装

```bash
adb push dist/uu-wheel-evdev.zip /data/local/tmp/
adb shell "su -c 'ksud module install /data/local/tmp/uu-wheel-evdev.zip'"
adb reboot
```

Magisk：

```bash
adb shell "su -c 'magisk --install-module /data/local/tmp/uu-wheel-evdev.zip'"
adb reboot
```

开机后模块会：

1. 等待约 15 秒
2. 启动 `uu_wheel_flip`
3. 抓取名字包含 `BT5.0 Mouse` / `Xiaomi Mouse` 的鼠标
4. 创建 `UU Wheel Flip ...` 虚拟鼠标

## 配置

控制文件：

```bash
adb shell "su -c 'echo 0 > /data/local/tmp/uu_wheel_flip.invert'"
```

- `0`：直通，不翻转（默认，当前验证正确）
- `1`：翻转 `REL_WHEEL` / `REL_HWHEEL`

修改后**立即生效**，不需要重启 daemon。日志：

```bash
adb shell "su -c 'cat /data/local/tmp/uu_wheel_flip.log'"
```

## 注意

- 目前只接管名字含 `BT5.0 Mouse` 或 `Xiaomi Mouse` 的设备。换鼠标需要改 `src/uu_wheel_flip.c` 里的 `is_target()`。
- 开机后 daemon 有约 15 秒启动延迟；如果刚开机就打开 UU远程，可能先绑到物理鼠标，重开一次 App 即可。
- **不要同时运行主机端的 `WheelFlip.exe`**。它会把 UU远程 注入的滚轮再翻一次，和本方案形成双重取反。本项目测试时已将其禁用。
- 如果之前装过 Frida 的 `uu-wheel-fix` 模块，请先卸载，避免残留的 frida-server 干扰。
- 不同 ROM/输入栈可能有差异；如果方向不对，先尝试切换 `invert`，再检查是否有其他软件在改滚轮方向。

## License

MIT
