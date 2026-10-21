# PS5 Linux Image Builder

Builds bootable Linux USB images for PlayStation 5 using Docker containers. Supports Ubuntu 26.04, Arch, CachyOS (Gamescope + Steam), Fedora (GNOME), individually or as a multi-distro image with kexec switching.

## About this fork

This is a fork of [ps5-linux/ps5-linux-image](https://github.com/ps5-linux/ps5-linux-image) that builds
the images on **kernel 7.3** with Wi-Fi and Bluetooth support for PS5 units that ship the
**MediaTek MT7921** chip. Changes from upstream:

- The kernel patches come from [airkingbr/ps5-linux-patches](https://github.com/airkingbr/ps5-linux-patches)
  (branch `meower`), a 7.3 patch set that enables the in-tree `mt7921e` Wi-Fi driver and adds two
  Bluetooth fixes for the MediaTek `0e8d:3603` chip (from
  [WizGery/ps5-mediatek-fixes](https://github.com/WizGery/ps5-mediatek-fixes)).
- The kernel tag is read from the patch set's `.config`: a `7.3.0-rc6` config is built on the `v7.3`
  tag. Upstream read an empty tag from it and failed to clone.
- The patch set's `firmware/` (built into the kernel through `CONFIG_EXTRA_FIRMWARE`) and `system/`
  are copied next to the kernel tree, since the kernel builder container only mounts that tree.
- `system/` is packaged with the kernel: a udev rule that disables the phantom second Bluetooth
  controller and USB autosuspend of the MediaTek chip, and the `mt7921e disable_aspm=1` and
  `btusb enable_autosuspend=0` module options. They are installed in every image.
- The out-of-tree NXP IW620 Wi-Fi driver (Marvell units) does not build against 7.3 yet. Instead of
  failing the build, the image is made without it. **Images from this fork have no Wi-Fi on IW620
  units**; MediaTek units are not affected.

Which chip a PS5 has:

```bash
lspci -nn | grep -i 14c3:7961   # MediaTek MT7921 Wi-Fi
lsusb | grep -i 0e8d:3603       # MediaTek Bluetooth
```

To build from the upstream patch set instead, set `PATCHES_REPO` and `--patches-ref`:

```bash
PATCHES_REPO=https://github.com/ps5-linux/ps5-linux-patches.git ./build_image.sh --patches-ref main
```

## Prerequisites

- Docker (with permission to run `--privileged` containers) — install as per your distro's instructions
- ~30GB free disk space for Ubuntu, Arch, or CachyOS

Once Docker is installed, add your user to the docker group and apply it without logging out:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Building on Windows with WSL

The build also runs in WSL 2 (tested with Ubuntu 26.04 in WSL, Docker installed inside it). Clone the
repository in the WSL home directory, not under `/mnt/c`, which is much slower:

```bash
git clone https://github.com/airkingbr/ps5-linux-image.git ~/ps5-linux-image
cd ~/ps5-linux-image
sudo ./build_image.sh --distro ubuntu2604
```

An Ubuntu image took about 27 minutes on a 24-thread machine. The image ends up in
`~/ps5-linux-image/output/`; copy it to a Windows drive (`cp output/*.img /mnt/d/`) and flash it
with [balenaEtcher](https://etcher.balena.io/) or [Rufus](https://rufus.ie/) in DD image mode.

## Quick Start

```bash
# Build a single Ubuntu 26.04 image
./build_image.sh --distro ubuntu2604

OR

# Build CachyOS (Arch-based, Gamescope + Steam Big Picture)
./build_image.sh --distro cachyos

OR

OR

# Build Fedora (GNOME desktop)
./build_image.sh --distro fedora

OR

# Build a multi-distro image (ubuntu2604 + arch + cachyos)
./build_image.sh --distro all
```

The script auto-clones the kernel source, applies PS5 patches, compiles, and builds the image. Subsequent runs reuse cached artifacts automatically. Press Ctrl+C at any time to abort cleanly.

## Flash to USB

```bash
sudo dd if=output/ps5-ubuntu2604.img of=/dev/sdX bs=4M status=progress
```

### Check Wi-Fi and Bluetooth on a MediaTek unit

After booting the image:

```bash
uname -r
sudo dmesg | grep -iE 'mt7921|PS5: using bulk OUT|Oops'
ip link | grep wl
ls /sys/class/bluetooth/
```

`dmesg` must show the MT7921 `ASIC revision` and `PS5: using bulk OUT 0x02 for ACL`, with no `Oops`.
There must be a `wl*` interface and a single `hci*` controller. Pairing a DualSense over Bluetooth
is described in [WizGery/ps5-mediatek-fixes](https://github.com/WizGery/ps5-mediatek-fixes#dualsense-over-bluetooth).

## Options

| Flag | Description | Default |
|------|-------------|---------|
| `--distro` | `ubuntu2604`, `arch`, `cachyos`, `fedora`, or `all` | `ubuntu2604` |
| `--kernel` | Path to kernel source directory | auto-clone version selected by PS5 patch set |
| `--img-size` | Disk image size in MB | `12000` (`32000` for `all`) |
| `--clean` | Remove all cached build artifacts and start fresh | off |
| `--kernel-only` | Build and package the kernel only, then exit | off |
| `--patches-ref` | Branch, tag, or commit SHA for patches | `meower` |

## Caching

The build automatically skips stages that have already completed:

- **Kernel source** — reused if `work/linux/` exists
- **Kernel packages** — reused if `.deb`/`.pkg.tar.zst` files exist in `linux-bin/`
- **Root filesystem** — reused if chroot directories are populated

Use `--clean` to wipe everything and rebuild from scratch. The build will also suggest `--clean` if a stage fails.

## Build Output

```
PS5 Linux Image Builder
=======================
  Distro:       all
                (ubuntu2604 arch cachyos)
  Image size:   32000MB
  Kernel src:   /path/to/work/linux

Stages:
  1. Kernel            cached
  2. Root filesystem   build
  3. Disk image        build

Logs: /path/to/build.log

  ✓ Kernel packages (cached)
  ✓ Build image builder image
  ⠹ Building arch rootfs
```

All verbose output goes to `build.log`. The terminal shows a spinner with live progress.

## Distributions

| Distro | Desktop | Kernel format | Init |
|--------|---------|---------------|------|
| Ubuntu 26.04 (Resolute) | GNOME | `.deb` | systemd |
| Arch | Sway | `.pkg.tar.zst` | systemd |
| CachyOS | Gamescope + Steam Big Picture (Arch + `[cachyos]` repo, no v3 migration in image build) | `.pkg.tar.zst` | systemd |

## Multi-distro Image

`--distro all` builds a 32GB image with 4 partitions (one EFI boot partition plus three root filesystems):

| Partition | Type | Label | Content |
|-----------|------|-------|---------|
| p1 | FAT32 | boot | Shared kernel, per-distro initrds, kexec scripts |
| p2 | ext4 | ubuntu2604 | Ubuntu 26.04 rootfs |
| p3 | ext4 | arch | Arch rootfs |
| p4 | ext4 | cachyos | CachyOS rootfs |

The boot partition contains kexec scripts to switch between distros at runtime. Ubuntu 26.04 is the default boot target.

## Building the Kernel Standalone

Use `--kernel-only` to compile the PS5 kernel and produce installable packages without building a full disk image.

```bash
./build_image.sh --kernel-only                                # .deb (default)
./build_image.sh --kernel-only --distro all                   # .deb + .pkg.tar.zst
./build_image.sh --kernel-only --patches-ref main             # fetch from specific branch/tag
./build_image.sh --kernel-only --clean                        # wipe and rebuild from scratch
```

Output packages are written to `linux-bin/`. Install on a running PS5 Linux system:

```bash
sudo dpkg -i linux-bin/linux-ps5_*.deb
```

## Directory Layout

```
build_image.sh                  # Image builder (also supports --kernel-only)
docker/
  kernel-builder/               # Kernel compilation container
  kernel-builder-arch/          # Repackages .deb kernel as .pkg.tar.zst
  image-builder/
    Dockerfile                  # Image building container (distrobuilder)
    entrypoint.sh               # Single-distro build logic
    entrypoint-multi.sh         # Multi-distro build logic
distros/
  ubuntu2604/                   # Ubuntu 26.04 (Resolute)
  arch/                         # Arch Linux
  cachyos/                      # CachyOS repos + Gamescope/Steam
  shared/                       # Kernel postinst hooks (single + multi)
boot/
  cmdline.txt                   # Kernel cmdline template (__DISTRO__ placeholder)
  vram.txt                      # VRAM allocation
  kexec-{ubuntu2604,arch,cachyos}.sh
work/                           # Build artifacts (auto-created)
linux-bin/                      # Compiled kernel packages
output/                         # Final .img files
```
