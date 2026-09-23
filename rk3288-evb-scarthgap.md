# RK3288 EVB Yocto Build Guide — Scarthgap (Poky 5.0)

Build 1: `core-image-minimal`
Build 2: `rk3288-terminal-image` (HDMI display)

March 2026. Kernel: Linux 6.1 | U-Boot: 2017.09 | Distro: Poky 5.0.15

Source document: `RK3288_Yocto_Build_Guide.docx` (kept in this repo).

## 1. Project overview

Complete process for building two Yocto images for the Rockchip RK3288 EVB board using the Poky reference distribution on the Scarthgap (5.0) release. Every step, the reason behind each decision, all bugs encountered, and exactly how they were resolved.

| Item | Value |
| ---- | ----- |
| Host OS | Ubuntu (x86_64) |
| Yocto release | Scarthgap (Poky 5.0.15) |
| Target board | Rockchip RK3288 EVB |
| Machine config | `rockchip-rk3288-evb` |
| Kernel | Linux 6.1 (`linux-rockchip_6.1.bb`) |
| U-Boot | 2017.09 |
| Init system | systemd |
| Build 1 image | `core-image-minimal` |
| Build 2 image | `rk3288-terminal-image` (HDMI terminal) |
| Build directory | `/nvme/yocto/poky/build` |

## 2. Layer setup and configuration

### 2.1 Layers used

| Layer | Purpose |
| ----- | ------- |
| `meta` | Poky core — base recipes, classes, toolchain |
| `meta-poky` | Poky distro configuration |
| `meta-yocto-bsp` | Reference BSP machines |
| `meta-arm` / `meta-arm-toolchain` | ARM architecture support and GCC toolchain |
| `meta-openembedded/meta-oe` | Extended package recipes (htop, i2c-tools, etc.) |
| `meta-rockchip` | Rockchip SoC BSP — kernel, U-Boot, Mali GPU, ISP drivers |

### 2.2 Why these layers are needed

`meta-rockchip` is the critical layer. Without it, Yocto has no knowledge of the RK3288 SoC, its memory map, boot sequence, device tree, or any Rockchip-specific drivers. It provides the machine configuration file (`rockchip-rk3288-evb.conf`) and the kernel recipe (`linux-rockchip_6.1.bb`).

`meta-openembedded/meta-oe` is required because many practical packages (SSH server, htop, i2c-tools) are not in the Poky core. `meta-arm` provides the ARM Cortex-A17 toolchain tuning used by the RK3288.

### 2.3 Machine selection

Three RK3288 machine configs exist in `meta-rockchip/conf/machine/`:

| Machine config file | Use case |
| ------------------- | -------- |
| `rockchip-rk3288-evb.conf` | Standard RK3288 EVB — used for this build |
| `rockchip-rk3288-evb-act8846.conf` | EVB variant with ACT8846 PMIC chip |
| `rockchip-rk3288w-evb.conf` | RK3288W low-power variant (tablet-focused) |

The standard `rockchip-rk3288-evb` was selected as it covers the majority of EVB boards:

```bitbake
MACHINE = "rockchip-rk3288-evb"
```

## 3. Build 1 — core-image-minimal

### 3.1 Goal

Produce the smallest possible bootable Linux image to validate the toolchain, BSP, kernel, and U-Boot for the RK3288 EVB. Standard practice when bringing up a new board: start minimal, confirm it boots, then add features.

### 3.2 local.conf settings

```bitbake
MACHINE = "rockchip-rk3288-evb"
DISTRO  = "poky"

# systemd as init manager (requires usrmerge in Scarthgap)
DISTRO_FEATURES:append = " systemd usrmerge"
DISTRO_FEATURES:remove  = "sysvinit"
VIRTUAL-RUNTIME_init_manager = "systemd"
VIRTUAL-RUNTIME_initscripts  = "systemd-compat-units"

# Parallel build tuning
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE     = "-j8"

EXTRA_IMAGE_FEATURES += "debug-tweaks ssh-server-openssh"
```

### 3.3 Build command

```bash
bitbake core-image-minimal
```

### 3.4 Bug #1 — systemd missing usrmerge feature

```text
ERROR: Nothing RPROVIDES 'systemd'
systemd was skipped: missing required distro feature 'usrmerge' (not in DISTRO_FEATURES)
```

Root cause: in Scarthgap (Yocto 5.0), systemd mandates the `usrmerge` distro feature as a hard dependency. UsrMerge merges `/bin`, `/sbin`, and `/lib` into their `/usr` equivalents. Older releases (e.g. Kirkstone) did not enforce this. Because `DISTRO_FEATURES` only contained `systemd` but not `usrmerge`, the systemd recipe was skipped, causing a cascade: `packagegroup-core-boot` requires systemd, and `core-image-minimal` requires `packagegroup-core-boot`.

Fix applied in `local.conf`:

```bitbake
DISTRO_FEATURES:append = " systemd usrmerge"
DISTRO_FEATURES:remove  = "sysvinit"
VIRTUAL-RUNTIME_init_manager = "systemd"
VIRTUAL-RUNTIME_initscripts  = "systemd-compat-units"
```

### 3.5 Bug #2 — fiq_debugger kernel compile error

```text
ERROR: fiq_debugger_arm.c:241 — implicit declaration of function 'THREAD_INFO'
drivers/soc/rockchip/fiq_debugger/fiq_debugger_arm.c: error: implicit declaration of function 'THREAD_INFO' [-Werror=implicit-function-declaration]
```

Root cause: the Rockchip FIQ Debugger driver uses the `THREAD_INFO()` macro to retrieve `thread_info` from a stack pointer. This macro was removed from the Linux kernel in version 4.9+ when `thread_info` was moved out of the kernel stack into `task_struct`. The BSP driver was never updated. The kernel builds with `-Werror`, so this becomes a hard failure.

Fix: disable the FIQ Debugger via a kernel config fragment (a low-level UART crash debugger, not needed for normal EVB bring-up) rather than patching the driver source.

File 1 — `meta-rockchip/recipes-kernel/linux/linux-rockchip/disable-fiq-debugger.cfg`:

```text
CONFIG_FIQ_DEBUGGER=n
CONFIG_RK_CONSOLE_THREAD=n
```

File 2 — `meta-rockchip/recipes-kernel/linux/linux-rockchip_6.1.bbappend`:

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"
SRC_URI:append = " file://disable-fiq-debugger.cfg"
```

Rebuild after the fix:

```bash
bitbake linux-rockchip -c cleansstate
bitbake core-image-minimal
```

### 3.6 Build 1 result

Build 1 SUCCESS:

```text
NOTE: Tasks Summary: Attempted 4660 tasks of which 2968 didn't need to be rerun and all succeeded.
```

| Output file | Purpose |
| ----------- | ------- |
| `boot.img` | Kernel + DTB + initramfs combined boot image |
| `uboot.img` | U-Boot bootloader |
| `trust.img` | ARM Trusted Firmware (TF-A / OP-TEE) |
| `idblock.img` | DDR initialization + miniloader (first stage) |
| `loader.bin` | U-Boot TPL/SPL loader |
| `rk3288-evb-rk808-linux.dtb` | Device tree blob |
| `core-image-minimal-...tar.gz` | Root filesystem (17 MB) |
| `modules-...tgz` | Kernel modules |

## 4. Build 2 — rk3288-terminal-image (HDMI display)

### 4.1 Goal

Extend Build 1 to show a login terminal directly on the HDMI-connected display. Useful for development, demos, and where serial console access is not convenient. The HDMI output is driven by the Synopsys DesignWare HDMI controller integrated into the RK3288 SoC.

### 4.2 Changes made vs Build 1

local.conf additions:

```bitbake
# Display support
DISTRO_FEATURES:append = " x11"

# Extra packages installed into image
IMAGE_INSTALL:append = " kbd util-linux bash openssh openssh-sshd"

# Developer-friendly image features
EXTRA_IMAGE_FEATURES += "debug-tweaks ssh-server-openssh tools-debug"
```

`x11` was added to `DISTRO_FEATURES` because the DRM/KMS display stack on Rockchip depends on it being present.

Kernel config fragment — the existing `disable-fiq-debugger.cfg` was updated to also enable HDMI output and framebuffer console:

```text
# Disable broken driver (from Build 1)
CONFIG_FIQ_DEBUGGER=n
CONFIG_RK_CONSOLE_THREAD=n

# DRM / Rockchip display engine
CONFIG_DRM=y
CONFIG_DRM_ROCKCHIP=y
CONFIG_FB=y
CONFIG_FRAMEBUFFER_CONSOLE=y
CONFIG_VT=y
CONFIG_VT_CONSOLE=y
CONFIG_DUMMY_CONSOLE=y

# Synopsys DesignWare HDMI controller (integrated in RK3288)
CONFIG_DRM_DW_HDMI=y
CONFIG_DRM_DW_HDMI_AHB_AUDIO=y
CONFIG_DRM_DW_HDMI_I2S_AUDIO=y
CONFIG_ROCKCHIP_DW_HDMI=y
```

| Config option | Why it is needed |
| ------------- | ---------------- |
| `CONFIG_DRM_ROCKCHIP` | Main Rockchip DRM/KMS display driver — controls the VOP (Video Output Processor) |
| `CONFIG_DRM_DW_HDMI` | Synopsys DesignWare HDMI IP driver — the HDMI encoder block inside the RK3288 |
| `CONFIG_ROCKCHIP_DW_HDMI` | Rockchip-specific glue between `DRM_ROCKCHIP` and `DRM_DW_HDMI` |
| `CONFIG_FRAMEBUFFER_CONSOLE` | Renders the Linux text console onto the framebuffer |
| `CONFIG_VT` / `CONFIG_VT_CONSOLE` | Virtual Terminal subsystem — required for getty and login prompt on tty1 |

Custom image recipe — `meta-rockchip/recipes-core/images/rk3288-terminal-image.bb`:

```bitbake
SUMMARY = "RK3288 image with HDMI terminal display"

require recipes-core/images/core-image-base.bb

IMAGE_INSTALL:append = " \
    kbd \
    util-linux \
    bash \
    openssh \
    openssh-sshd \
    i2c-tools \
    htop \
    e2fsprogs \
"

IMAGE_FEATURES += " \
    debug-tweaks \
    ssh-server-openssh \
    tools-debug \
"
```

`core-image-base` was used as the base (rather than `core-image-minimal`) because it includes a broader set of filesystem utilities and init scripts needed when a display and input subsystem is present.

systemd getty on tty1: by default, systemd only starts getty on `ttyFIQ0` (the Rockchip serial console). To show a login prompt on HDMI, getty needs to start on `tty1`.

File 1 — `meta-rockchip/recipes-core/systemd/systemd/tty1-getty.conf`:

```ini
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin root --noclear tty1 linux
```

File 2 — `meta-rockchip/recipes-core/systemd/systemd_%.bbappend`:

```bitbake
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

SRC_URI:append = " file://tty1-getty.conf"

do_install:append() {
    install -d ${D}${systemd_system_unitdir}/getty@tty1.service.d/
    install -m 0644 ${WORKDIR}/tty1-getty.conf \
        ${D}${systemd_system_unitdir}/getty@tty1.service.d/override.conf
    install -d ${D}${systemd_system_unitdir}/getty.target.wants/
    ln -sf ../getty@.service \
        ${D}${systemd_system_unitdir}/getty.target.wants/getty@tty1.service
}
```

`--autologin root` gives immediate shell access on screen after boot, appropriate for a development EVB.

### 4.3 Build command

```bash
bitbake linux-rockchip -c cleansstate
bitbake rk3288-terminal-image
```

## 5. Flashing the board

### 5.1 Output files

Both builds produce the same set of boot images; the rootfs tarball differs. Deploy directory:

```text
/nvme/yocto/poky/build/tmp/deploy/images/rockchip-rk3288-evb/
```

### 5.2 Why no .wic file?

Unlike PC-style boards, Rockchip platforms use a proprietary partition layout and boot ROM protocol. The `meta-rockchip` BSP produces individual partition images (idblock, uboot, trust, boot, rootfs) rather than a single `.wic` SD card image. Each partition must be written to a specific LBA offset.

### 5.3 Flashing via rkdeveloptool (Maskrom mode)

Put the board into Maskrom mode by holding the RECOVERY button while powering on, then connect the USB OTG port to the host PC.

```bash
cd /nvme/yocto/poky/build/tmp/deploy/images/rockchip-rk3288-evb/

# Step 1: Initialize DDR via loader
sudo rkdeveloptool db loader.bin

# Step 2: Flash partitions in order
sudo rkdeveloptool wl 0x0    idblock.img   # DDR init + miniloader
sudo rkdeveloptool wl 0x40   uboot.img     # U-Boot
sudo rkdeveloptool wl 0x1000 trust.img     # ARM Trusted Firmware
sudo rkdeveloptool wl 0x2000 boot.img      # Kernel + DTB

# Step 3: Reboot
sudo rkdeveloptool rd
```

### 5.4 Monitoring boot via serial console

```bash
# 115200 baud on ttyFIQ0 (as defined in rk3288.inc)
sudo picocom /dev/ttyUSB0 -b 115200
```

## 6. Bug summary

| Bug | Error | Root cause | Fix |
| --- | ----- | ---------- | --- |
| Bug #1, Build 1 | `Nothing RPROVIDES 'systemd'` | Scarthgap enforces `usrmerge` as a hard dependency for systemd. Not required in older releases like Kirkstone. | Added `usrmerge` to `DISTRO_FEATURES` alongside `systemd` in `local.conf` |
| Bug #2, Build 1 | `fiq_debugger_arm.c: implicit declaration of THREAD_INFO()` | `THREAD_INFO()` macro removed from Linux kernel 4.9+. BSP driver never updated. `-Werror` makes it a hard failure. | Kernel `.cfg` fragment with `CONFIG_FIQ_DEBUGGER=n`, injected via `bbappend` |

## 7. Files created / modified

| File path | Purpose |
| --------- | ------- |
| `build/conf/local.conf` | Machine, distro, systemd, display, and package settings |
| `meta-rockchip/recipes-kernel/linux/linux-rockchip_6.1.bbappend` | Injects kernel config fragment into linux-rockchip 6.1 build |
| `meta-rockchip/recipes-kernel/linux/linux-rockchip/disable-fiq-debugger.cfg` | Disables FIQ debugger; enables DRM, framebuffer, HDMI kernel options |
| `meta-rockchip/recipes-core/images/rk3288-terminal-image.bb` | Custom image recipe with SSH, kbd, htop, bash, and display packages |
| `meta-rockchip/recipes-core/systemd/systemd_%.bbappend` | Installs tty1 getty override so login prompt appears on HDMI display |
| `meta-rockchip/recipes-core/systemd/systemd/tty1-getty.conf` | systemd drop-in: starts agetty with autologin on tty1 (framebuffer) |

## 8. Key design decisions

| Decision | Reason |
| -------- | ------ |
| Used kernel 6.1 (not 4.4 or 5.10) | `rockchip-common.inc` sets `PREFERRED_VERSION_linux-rockchip = "6.1%"` for all Rockchip machines on Scarthgap. Maintained LTS kernel for this BSP. |
| Disabled FIQ Debugger (not patched) | Disabling is safer and lower maintenance than patching a driver that touches kernel-internal structures. |
| Used `core-image-base` for Build 2 | `core-image-minimal` lacks some filesystem and init infrastructure needed when a display subsystem is active. |
| Autologin on tty1 | Development EVB does not need login security. `--autologin root` gives immediate shell after boot. |
| Individual partition flashing (not `.wic`) | `meta-rockchip` BSP does not generate `.wic` images. Rockchip boot ROM requires specific LBA offsets handled by `rkdeveloptool`. |
