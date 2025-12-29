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
description: "A deep dive into making Plymouth boot animations work with NVIDIA's proprietary driver on Debian 13, involving kernel module timing, DRM framebuffer initialization, and initramfs configuration"
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

> **Disclaimer**: This blog post was written by Claude Opus 4.5 (Anthropic) based on an actual debugging session. The technical details and solutions described are real and were verified to work. The post has been revised based on technical fact-checking.

## The Problem

Plymouth is the boot splash system used by most modern Linux distributions to display animations during boot instead of scrolling kernel messages. On systems with NVIDIA GPUs running proprietary drivers, Plymouth often fails to display anything—you get a black screen with a blinking cursor, then suddenly SDDM/GDM appears.

This is a well-known issue caused by the complex interaction between:
- EFI framebuffer initialization
- NVIDIA's DRM (Direct Rendering Manager) kernel module
- Plymouth's framebuffer requirements
- The timing of kernel module loading

## Environment

- **OS**: Debian 13 "Trixie" (Stable, released August 9, 2025)
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

The 550.x driver's DRM implementation doesn't properly handle the modeset permission transfer that Plymouth requires. This was a widespread issue in the 550.xx series on kernel 6.10+.

## Why XanMod Drivers Don't Work on Debian

XanMod provides newer NVIDIA drivers (575, 580, 590) via their apt repository. However, these packages fail to build on Debian's kernel due to a fundamental incompatibility in the `conftest` system.

### The conftest Problem

NVIDIA's driver source uses a `conftest.sh` script that probes kernel capabilities by attempting to compile small test programs. The results determine which code paths are enabled via preprocessor defines.

Debian's kernel uses a **split header structure**:
- `/usr/src/linux-headers-6.12.48+deb13-amd64/` - Architecture-specific headers
- `/usr/src/linux-headers-6.12.48+deb13-common/` - Common headers

XanMod's conftest fails because the include paths don't match Debian's structure. The compile tests fail for the **wrong reasons** (missing headers rather than missing functionality), producing false positives/negatives:

```
// conftest tries to compile this:
#include <asm/io.h>

void conftest_ioremap_driver_hardened(void) {
    ioremap_driver_hardened();
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
```
sudo apt remove --purge '*nvidia*' -y
sudo apt autoremove --purge -y
```

2. **Download the driver** from [NVIDIA's driver page](https://www.nvidia.com/en-in/drivers/details/259042/):
```
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/580.119.02/NVIDIA-Linux-x86_64-580.119.02.run
chmod +x NVIDIA-Linux-x86_64-580.119.02.run
```

3. **The TTY Installation Dance**:
```
# Switch to TTY: Ctrl+Alt+F2
sudo systemctl stop sddm
sudo rmmod nvidia_drm nvidia_modeset nvidia_uvm nvidia
sudo ./NVIDIA-Linux-x86_64-580.119.02.run --dkms
```

### A Note on TTY Installation

When you stop SDDM and run the installer from a TTY, the experience can be disorienting. While `--ui=none` suppresses the ncurses GUI, output is still printed to stdout. However, in our debugging session, we ran a wrapper script that included an automatic reboot—meaning we never saw the installer output and had no idea if it succeeded until the system came back up.

**Recommendation**: Always log your installation:
```
sudo ./NVIDIA-Linux-x86_64-580.119.02.run --dkms 2>&1 | tee /tmp/nvidia-install.log
```

And ensure nouveau is blacklisted:
```
sudo tee /etc/modprobe.d/blacklist-nouveau.conf << 'EOF'
blacklist nouveau
options nouveau modeset=0
EOF
```

4. **Configure kernel parameters** in `/etc/default/grub`:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia-drm.modeset=1 nvidia-drm.fbdev=1"
```

The critical parameter is `nvidia-drm.fbdev=1`, introduced in driver 545+, which enables NVIDIA's DRM framebuffer device. This single parameter is responsible for most of the fix—it provides Plymouth with a working framebuffer that persists through the boot process.

*(Note: `nvidia-drm.modeset=1` often defaults to on in recent drivers, but including it explicitly doesn't hurt.)*

5. **Update GRUB**:
```
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

### Solution: Load NVIDIA Modules Early in initramfs

The NVIDIA modules must load **before** Plymouth starts. Debian's `initramfs-tools` handles this cleanly—you don't need custom hook scripts to copy module files manually.

**The clean Debian way:**

1. **Add modules to initramfs configuration**:
```
echo -e "nvidia\nnvidia_modeset\nnvidia_uvm\nnvidia_drm" | sudo tee -a /etc/initramfs-tools/modules
```

2. **Rebuild initramfs**:
```
sudo update-initramfs -u
```

That's it. The system's standard hooks will find the modules (even from DKMS paths like `/lib/modules/*/updates/dkms/`) and bundle them correctly.

3. **Verify modules are included**:
```
lsinitramfs /boot/initrd.img-$(uname -r) | grep -E "nvidia.*\.ko"
```

The initramfs will grow significantly (~160MB) due to the NVIDIA modules (~100MB combined).

### Optional: Early Load Script

If you find modules aren't loading early enough, you can add an `init-top` script. However, listing modules in `/etc/initramfs-tools/modules` is usually sufficient as they're processed very early in boot.

Create `/etc/initramfs-tools/scripts/init-top/nvidia`:
```
#!/bin/sh
PREREQ=""
prereqs() { echo "$PREREQ"; }
case "$1" in prereqs) prereqs; exit 0;; esac

modprobe nvidia
modprobe nvidia_modeset
modprobe nvidia_drm
exit 0
```

Then:
```
sudo chmod +x /etc/initramfs-tools/scripts/init-top/nvidia
sudo update-initramfs -u
```

## Final Configuration Summary

### /etc/default/grub
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nvidia-drm.modeset=1 nvidia-drm.fbdev=1"
GRUB_GFXMODE=auto
GRUB_GFXPAYLOAD_LINUX=keep
```

### /etc/initramfs-tools/modules
```
nvidia
nvidia_modeset
nvidia_uvm
nvidia_drm
```

### /etc/plymouth/plymouthd.conf
```
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
```
$ sudo cat /sys/module/nvidia_drm/parameters/modeset
Y
$ sudo cat /sys/module/nvidia_drm/parameters/fbdev
Y
$ cat /proc/fb
0 nvidia-drmdrmfb
```

## Why This Works

1. **nvidia-drm.fbdev=1** creates `/dev/fb0` as `nvidia-drmdrmfb` instead of relying on efifb—this is the key fix
2. **Modules in initramfs** ensures NVIDIA loads before Plymouth starts
3. **nvidia-drm.modeset=1** enables kernel mode setting (often default, but explicit is fine)
4. **Plymouth** starts after the NVIDIA framebuffer is initialized, rendering directly to it
5. **No framebuffer transition** occurs during boot—Plymouth's output persists until the display manager takes over
6. **nouveau blacklisted** prevents the open-source driver from interfering

## Conclusion

Getting Plymouth to work with NVIDIA on modern Linux requires understanding the interplay between EFI framebuffers, kernel DRM subsystems, and the boot sequence timing. The key insights are:

1. **NVIDIA 580+ has better DRM/fbdev support than 550.x**—the `nv_drm_revoke_modeset_permission` bug is fixed
2. **XanMod packages don't work on Debian** due to conftest/header incompatibilities
3. **The official `.run` installer** handles arbitrary kernel configurations correctly
4. **`nvidia-drm.fbdev=1` is the critical parameter**—it's responsible for ~90% of the fix
5. **Use Debian's standard initramfs-tools**—just add modules to `/etc/initramfs-tools/modules`, no custom copy scripts needed

The solution survives kernel updates as long as the NVIDIA DKMS modules rebuild successfully.

