# Zoom on Niri (NVIDIA)

This branch fixes Zoom screen sharing on niri with NVIDIA GPUs. Without it, screen sharing shows a black screen.

There is also a separate, non-niri fix needed to prevent Zoom UI lag on niri.

## Screen sharing fix (this branch)

Niri's PipeWire screencast producer only offers DMA-BUF buffers by default. Zoom's PipeWire client can't import them, causing screen sharing to fail. Worse, Zoom caches DMA-BUF modifiers from the first share and re-advertises them as supported for subsequent shares, causing a black screen even when SHM fallback is available.

This branch:
1. Adds SHM fallback to niri's PipeWire screencast (upstream PR [#1791](https://github.com/YaLTeR/niri/pull/1791))
2. Removes DMA-BUF from the screencast params entirely, offering only SHM

Trade-off: all screencast consumers (OBS, etc.) receive SHM instead of zero-copy DMA-BUF, adding a GPU-CPU-GPU copy. For typical screencasting this is negligible.

### Build and install

```sh
git clone -b zoom-shm-fix https://github.com/slynchDev/niri.git
cd niri
cargo build --release
cargo deb --no-build
sudo dpkg -i target/debian/niri_*.deb
```

Log out and back in to restart niri.

### Staying up to date

```sh
git fetch origin
git rebase origin/main
cargo build --release
cargo deb --no-build
sudo dpkg -i target/debian/niri_*.deb
```

## Zoom UI lag fix (separate from this branch)

Niri sets `QT_QPA_PLATFORM="wayland"` globally in its environment block, which forces Zoom into its broken native Wayland mode. This causes severe UI and video lag, especially on NVIDIA.

Fix: create a local desktop entry that forces Zoom to run through XWayland:

```sh
mkdir -p ~/.local/share/applications
cat > ~/.local/share/applications/Zoom.desktop << 'EOF'
[Desktop Entry]
Name=Zoom
Comment=Zoom Video Conference
Exec=env QT_QPA_PLATFORM=xcb /usr/bin/zoom %U
Icon=Zoom
Terminal=false
Type=Application
Categories=Network;
MimeType=x-scheme-handler/zoommtg;x-scheme-handler/zoomus;x-scheme-handler/tel;x-scheme-handler/callto;x-scheme-handler/zoomphonecall;x-scheme-handler/zoomrc;x-scheme-handler/zoomjoinevent;
EOF
```

This overrides `/usr/share/applications/Zoom.desktop` per XDG spec. It only affects Zoom -- other Qt apps still get native Wayland.

To launch manually with the fix:

```sh
QT_QPA_PLATFORM=xcb zoom
```

## Tested with

- niri v25.11
- Zoom 6.7.5 (6891)
- PipeWire 1.4.2
- NVIDIA RTX 4090, driver 550.163.01
- Debian (trixie)
