# Fork 说明：在 Ubuntu 20.04 (focal) 上构建

上游：<https://github.com/VallierDev/uu-remote-linux>（GPL-3.0，本 fork 沿用同一许可证）

这个 fork 只为一个目标服务：**让 X11 后端能在 glibc 2.31 的老发行版上从源码构建并运行**。
上游主线参考环境是 Ubuntu 24.04，预编译包要求 glibc ≥ 2.34，在 20.04 上跑不起来；源码构建则只差一个
GLib 兼容符号。改动面刻意保持最小，方便随时 rebase 上游。

## 相对上游的改动

| 文件 | 改动 |
|---|---|
| `c_engine/assist_window.c` | 加 13 行 `g_memdup2` 兼容实现（GLib < 2.68 时生效） |
| `FORK-NOTES.md` | 本文件 |

`g_memdup2` 是 GLib 2.68 引入的，Ubuntu 20.04 带的是 2.64，链接 GTK 登录窗口时会报
`undefined symbol: g_memdup2`。补丁按 GLib 官方实现补一个 static 版本，行为一致
（`byte_size == 0` 或 `mem == NULL` 时返回 `NULL` 且不分配）。

不想改代码的等价做法：`UU_NO_GTK=1 cargo build --release`，登录改成终端里扫码。

## 构建

```bash
sudo apt install -y build-essential pkg-config \
    libx11-dev libxext-dev libxtst-dev libxfixes-dev libgtk-3-dev \
    libgbm-dev libx11-xcb-dev libxcb-dri3-dev libxcb-damage0-dev libva-dev

cargo build --release --features inproc-av
```

`inproc-av` 在老环境上是**必需**的，不是可选优化：

- Ubuntu 20.04 的 apt FFmpeg 是 4.2.7，认不出 `h264_nvenc` 的 `-preset p4` 和 `-pix_fmt nvenc`
  （两者都是 FFmpeg 5.0+ 才有的），NVENC 探测必然失败并回退 CPU。
- 反过来，2025 年之后的 FFmpeg nightly 静态包要求 NVENC API 13.1，即驱动 ≥ 610；
  老驱动（如 580.xx）会被 `Driver does not support the required nvenc API version` 拒绝。

`inproc-av` 直接 `dlopen` 驱动里的 `libnvidia-encode.so.1`，绕开 FFmpeg 的 NVENC 封装，
只要求 NVENC API ≥ 12.1（驱动 ≥ 530）；截屏与编码尺寸一致时编码器直接读 XShm 缓冲，
省掉一次 host-to-device 拷贝。验证：

```
Desktop encoder: ... encoder=h264_nvenc pipeline=in-process
Native NVENC reads the registered XShm buffer directly (no host-to-device copy)
Encoded capture probe: frames=5 h264_bytes=166994 first_frame_idr=true
```

Wayland / portal 后端在 20.04 上仍然**不可用**（需要 `libei >= 1.1`、GNOME 45+ 的动态显示 API 与
新版 portal 的 RestoreToken），请不要开 `--features wayland,portal`。

## 装成用户服务

```bash
install -Dm755 target/release/uuremote-server ~/.local/bin/uuremote-server
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/uuremote-native.service <<'EOF'
[Unit]
Description=UU Remote Linux host (unofficial)
After=graphical-session.target
PartOf=graphical-session.target

[Service]
Environment=RUST_LOG=info
ExecStart=%h/.local/bin/uuremote-server
Restart=always
RestartSec=5

[Install]
WantedBy=graphical-session.target
EOF
systemctl --user daemon-reload
systemctl --user enable --now uuremote-native.service
uuremote-server doctor
```

GNOME / KDE 会把 `DISPLAY`、`XAUTHORITY` 注入用户 systemd 环境；Xfce / i3 需要把 `WantedBy` 改成
`default.target` 并加 `Environment=DISPLAY=:0`。

## 实测环境

Ubuntu 20.04.6 LTS（glibc 2.31）· GNOME Shell 3.36 · **Xorg 会话** · RTX 3090 Ti，驱动 580.126.09
· 2560x1080 单屏。`doctor` 结果：`selected backend X11`、`encoder Nvenc`、`signed in true`、`service active`。

## 同步上游

```bash
git fetch upstream && git rebase upstream/main
```

补丁只碰一个文件，冲突概率很低；上游若哪天改用 `g_memdup` 或抬高 GLib 下限，直接丢弃该 commit 即可。
