---
title: "Getting Plymouth Boot Splash Working with NVIDIA 580 on Debian 13 Trixie"
date: 2025-12-29T17:00:00+01:00
tags: ["linux", "nvidia", "debian", "plymouth", "kernel"]
author: "Rick Lamers"
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
description: "A deep dive into making Plymouth boot animations work with NVIDIA's proprietary driver on Debian 13, involving kernel module timing, DRM framebuffer initialization, and initramfs surgery"
disableHLJS: false
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

> **Disclaimer**: This blog post was written by Claude Opus 4.5 (Anthropic) based on an actual debugging session. The technical details and solutions described are real and were verified to work.

## The Problem

Plymouth is the boot splash system used by most modern Linux distributions to display animations during boot instead of scrolling kernel messages. On systems with NVIDIA GPUs running proprietary drivers, Plymouth often fails to display anything—you get a black screen with a blinking cursor, then suddenly SDDM/GDM appears.

This is a well-known issue caused by the complex interaction between:
- EFI framebuffer initialization
- NVIDIA's DRM (Direct Rendering Manager) kernel module
- Plymouth's framebuffer requirements
- The timing of kernel module loading

## Environment

- **OS**: Debian 13 "Trixie" (Testing)
- **Kernel**: 6.12.48+deb13-amd64
- **GPU**: NVIDIA GeForce RTX 3090
- **Driver**: NVIDIA 580.119.02 (official `.run` installer)
- **Plymouth Theme**: softwaves

## Why NVIDIA 550.x Fails

Debian 13's repositories ship NVIDIA driver 550.163.01. This driver has a known bug involving `nv_drm_revoke_modeset_permission` that causes Plymouth to fail when attempting DRM master handoff. The kernel logs show:

```
WARNING: CPU: 4 PID: 380 at drivers/gpu/drm/drm_auth.c:320 drm_master_release+0x1e4/0x200 [drm]
nv_drm_revoke_modeset_permission: Failed to revoke DRM modeset permission
```

The 550.x driver's DRM implementation doesn't properly handle the modeset permission transfer that Plymouth requires.

## Why XanMod Drivers Don't Work on Debian

XanMod provides newer NVIDIA drivers (575, 580, 590) via their apt repository. However, these packages fail to build on Debian's kernel due to a fundamental incompatibility in the `conftest` system.

### The conftest Problem

NVIDIA's driver source uses a `conftest.sh` script that probes kernel capabilities by attempting to compile small test programs. The results determine which code paths are enabled via preprocessor defines.

Debian's kernel uses a **split header structure**:
- `/usr/src/linux-headers-6.12.48+deb13-amd64/` - Architecture-specific headers
- `/usr/src/linux-headers-6.12.48+deb13-common/` - Common headers

XanMod's conftest fails because the include paths don't match Debian's structure. The compile tests fail for the **wrong reasons** (missing headers rather than missing functionality), producing false positives/negatives:

```c
// conftest tries to compile:
#include <asm/io.h>
void conftest_ioremap_driver_hardened(void) {
    ioremap_driver_hardened();  // Debian-specific hardening function
}
```

When this fails to compile due to missing `asm/rwonce.h` (an include path issue), conftest incorrectly reports the function as "present," causing the driver to emit calls to a non-existent function:

```
error: implicit declaration of function 'ioremap_driver_hardened'
```

Other false detections include:
- `NV_MM_HAS_MMAP_LOCK` → undef (wrong: 6.12 uses `mmap_lock`, not `mmap_sem`)
- `NV_VM_FAULT_T_IS_PRESENT` → undef (wrong: 6.12 has `vm_fault_t`)
- `NV_PROC_OPS_PRESENT` → undef (wrong: 6.12 has `proc_ops`)

The result: every XanMod NVIDIA package fails with hundreds of kernel API mismatch errors.

## The Solution: Official NVIDIA .run Installer

NVIDIA's official `.run` installer includes its own conftest implementation that correctly probes kernel capabilities regardless of header layout.

### Installation Steps

1. **Remove all existing NVIDIA packages**:
```bash
sudo apt remove --purge '*nvidia*' -y
sudo apt autoremove --purge -y
```

2. **Download the driver** from [NVIDIA's driver page](https://www.nvidia.com/en-in/drivers/details/259042/):
```bash
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/580.119.02/NVIDIA-Linux-x86_64-580.119.02.run
chmod +x NVIDIA-Linux-x86_64-580.119.02.run
```

3. **The TTY Installation Dance** (see caveat below):
```bash
# Switch to TTY: Ctrl+Alt+F2
sudo systemctl stop sddm
sudo rmmod nvidia_drm nvidia_modeset nvidia_uvm nvidia
sudo ./NVIDIA-Linux-x86_64-580.119.02.run --dkms --no-questions --ui=none
```

### The Blind Installation Problem

Here's a reality check: when you stop SDDM and run the installer from a TTY with `--ui=none`, **you're flying blind**. The installer produces no visible output. You sit there staring at a black screen, hoping it's working. In our case, the script included an automatic reboot at the end—which was the only indication that anything had happened at all.

This is suboptimal for several reasons:
- No progress indication
- No error visibility
- If the installer fails silently, you won't know until you try to boot into a broken system
- The script may miss important steps (like blacklisting nouveau)

A more robust approach would be:
```bash
# Run with visible output
sudo ./NVIDIA-Linux-x86_64-580.119.02.run --dkms

# Or log everything
sudo ./NVIDIA-Linux-x86_64-580.119.02.run --dkms 2>&1 | tee /tmp/nvidia-install.log
```

And ensure nouveau is blacklisted:
```bash
cat > /etc/modprobe.d/blacklist-nouveau.conf << 'EOF'
blacklist nouveau
options nouveau modeset=0
EOF
```

4. **Configure kernel parameters** in `/etc/default/grub`:
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia-drm.modeset=1 nvidia-drm.fbdev=1"
```

The critical parameter is `nvidia-drm.fbdev=1`, introduced in driver 545+, which enables NVIDIA's DRM framebuffer device. This provides Plymouth with a working framebuffer that persists through the boot process.

5. **Update GRUB**:
```bash
sudo update-grub
```

## The Timing Problem

Even with the correct driver, Plymouth may still show a black screen. Examining `journalctl -b`:

```
16:44:04 kernel: fb0: EFI VGA frame buffer device
16:44:04 systemd[1]: Started plymouth-start.service
16:44:05 kernel: nvidia: loading out-of-tree module taints kernel
16:44:07 kernel: fbcon: nvidia-drmdrmfb (fb0) is primary device
```

Plymouth starts at `16:44:04` on the EFI framebuffer. NVIDIA takes over at `16:44:07`—a 3-second gap where Plymouth is rendering to a framebuffer that's about to be replaced.

### Solution: Load NVIDIA Modules in initramfs

The NVIDIA modules must load **before** Plymouth starts. This requires embedding them in the initramfs.

1. **Create a hook to copy modules** at `/etc/initramfs-tools/hooks/nvidia-force`:
```bash
#!/bin/sh
set -e
PREREQ=""
prereqs() { echo "$PREREQ"; }
case "$1" in prereqs) prereqs; exit 0;; esac
. /usr/share/initramfs-tools/hook-functions

KERNEL_VERSION="${version}"
MODULES_DIR="/lib/modules/${KERNEL_VERSION}/updates/dkms"

if [ -d "$MODULES_DIR" ]; then
    mkdir -p "${DESTDIR}/lib/modules/${KERNEL_VERSION}/updates/dkms"
    for mod in nvidia.ko nvidia-modeset.ko nvidia-drm.ko nvidia-uvm.ko; do
        if [ -f "${MODULES_DIR}/${mod}" ]; then
            cp "${MODULES_DIR}/${mod}" "${DESTDIR}/lib/modules/${KERNEL_VERSION}/updates/dkms/"
        fi
    done
fi
exit 0
```

2. **Create an early-load script** at `/etc/initramfs-tools/scripts/init-top/nvidia`:
```bash
#!/bin/sh
PREREQ=""
prereqs() { echo "$PREREQ"; }
case "$1" in prereqs) prereqs; exit 0;; esac

modprobe nvidia
modprobe nvidia_modeset
modprobe nvidia_drm
exit 0
```

3. **Make scripts executable and rebuild**:
```bash
chmod +x /etc/initramfs-tools/hooks/nvidia-force
chmod +x /etc/initramfs-tools/scripts/init-top/nvidia
sudo update-initramfs -u
```

4. **Verify modules are included**:
```bash
lsinitramfs /boot/initrd.img-$(uname -r) | grep nvidia
```

Expected output:
```
usr/lib/modules/6.12.48+deb13-amd64/updates/dkms/nvidia.ko
usr/lib/modules/6.12.48+deb13-amd64/updates/dkms/nvidia-modeset.ko
usr/lib/modules/6.12.48+deb13-amd64/updates/dkms/nvidia-drm.ko
usr/lib/modules/6.12.48+deb13-amd64/updates/dkms/nvidia-uvm.ko
scripts/init-top/nvidia
```

The initramfs will grow significantly (~160MB) due to the NVIDIA modules (~100MB combined).

## Final Configuration Summary

### /etc/default/grub
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia-drm.modeset=1 nvidia-drm.fbdev=1"
GRUB_GFXMODE=auto
GRUB_GFXPAYLOAD_LINUX=keep
```

### /etc/plymouth/plymouthd.conf
```ini
[Daemon]
Theme=softwaves
ShowDelay=0
```

### /etc/modprobe.d/nvidia-drm.conf
```
options nvidia-drm modeset=1 fbdev=1
```

### /etc/modprobe.d/blacklist-nouveau.conf
```
blacklist nouveau
options nouveau modeset=0
```

### Kernel Module Parameters (verify at runtime)
```bash
$ sudo cat /sys/module/nvidia_drm/parameters/modeset
Y
$ sudo cat /sys/module/nvidia_drm/parameters/fbdev
Y
$ cat /proc/fb
0 nvidia-drmdrmfb
```

## Why This Works

1. **init-top script** loads NVIDIA modules during initramfs, before the root filesystem is mounted
2. **nvidia-drm.modeset=1** enables kernel mode setting, allowing the kernel to manage display modes
3. **nvidia-drm.fbdev=1** creates `/dev/fb0` as `nvidia-drmdrmfb` instead of relying on efifb
4. **Plymouth** starts after the NVIDIA framebuffer is initialized, rendering directly to it
5. **No framebuffer transition** occurs during boot—Plymouth's output persists until the display manager takes over
6. **nouveau blacklisted** prevents the open-source driver from interfering

## Conclusion

Getting Plymouth to work with NVIDIA on modern Linux requires understanding the interplay between EFI framebuffers, kernel DRM subsystems, and the boot sequence timing. The key insights are:

1. NVIDIA 580+ has better DRM/fbdev support than 550.x
2. XanMod packages don't work on Debian due to conftest/header incompatibilities
3. The official `.run` installer's conftest handles arbitrary kernel configurations
4. Modules must load in initramfs, before Plymouth, to avoid framebuffer transitions
5. TTY installation is a leap of faith—log your output!

The solution is robust but requires manual initramfs configuration that won't survive kernel updates without a proper hook infrastructure—which is exactly what we've built here.
