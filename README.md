# Building Yocto

A guide to building Yocto for Raspberry Pi 4B and Rockchip RK3288.

- `rpi4b.md` — RPi 4B `core-image-minimal` on Wrynose 6.0 with `bitbake-setup` fragments
- `rk3288-evb-scarthgap.md` — RK3288 EVB on Scarthgap Poky 5.0.15, converted from `RK3288_Yocto_Build_Guide.docx`: `core-image-minimal` + `rk3288-terminal-image` (HDMI), kernel 6.1, `rkdeveloptool` flashing
- `rk3288-generic.md` — generic reference-design RK3288 board notes (ACT8846-class PMIC), boot evidence from Firefly RK3288 on 6.18
- `images/` — renamed board, PMIC, boot-log, btop, and fastfetch photos

## Overview

The last time I used Yocto was in 2024, and I had a steep learning curve setting up the project and building custom images. Since then, much has changed. As of July 2026, setting up projects and building images has become much better organized, simpler, and more intuitive thanks to the new bitbake-setup wizard and fragments.

Previously, I was ambitious and dove directly into building an image for the RK3288 eval board. Mind you, this is not a board from Radxa or Firefly, but a generic RK3288 board based on Rockchip's reference design with only a basic datasheet. Many issues arose during my last attempt, especially with the PMIC, and I was unable to complete the build. It took a lot of time.

This time, I'm taking a simpler approach:
1. Understand the new wizard and fragmentation by building a core-image-minimal for Raspberry Pi 4B
2. Then take the existing configuration for RK3288 and port it, fixing issues as they arise step by step.


## Part 1: core-image-minimal for RPi 4B with bitbake-setup

### Setup

The new wizard introduced in Wrynose 6.0 is intuitive. Run the setup command and you'll get the wizard below, which sets up your bitbake layers:
1. Choose your template—I chose Wrynose as it is the latest and LTS
2. Choose bitbake config—I believe with sstate your system will download build caches. I chose option 1 (poky)
3. Choose target machine—I am building for RPi 4B, so I selected generic ARM64
4. Continue with the setup

```bash
$ ./bitbake/bin/bitbake-setup init
NOTE: Bitbake-setup is using /home/surya/bitbake-builds as top directory.
NOTE: A site.conf file already exists. Please remove it if you would like to replace it with a default one

Available Configuration Templates:
1. oe-nodistro-master   OpenEmbedded - 'nodistro' basic configuration
2. oe-nodistro-wrynose  OpenEmbedded - 'nodistro' basic configuration, release 6.0 'wrynose' (supported until 2030-05-31)
3. poky-master          Poky - The Yocto Project testing distribution configurations and hardware test platforms
4. poky-wrynose         Poky - The Yocto Project testing distribution configurations and hardware test platforms, release 6.0 'wrynose' (supported until 2030-05-31)

Please select one of the above configurations by its number: 4

Available bitbake configurations:
1. poky              Poky - The Yocto Project testing distribution
2. poky-with-sstate  Poky - The Yocto Project testing distribution with internet sstate acceleration. Use with caution as it requires a completely robust local network with sufficient bandwidth.

Please select one of the above bitbake configurations by its number: 1

Target machines:
1. machine/qemux86-64     x86-64 system on QEMU
2. machine/qemuarm64      ARMv8 system on QEMU
3. machine/qemuriscv64    RISC-V system on QEMU
4. machine/genericarm64   Arm64 SystemReady IR/ES platforms
5. machine/genericx86-64  x86_64 (64-bit) PCs and servers

Please select one of the above options by its number: 4

```
2. Source your build environment:

```bash
$ source build/init-build-env
```

3. Add meta-raspberrypi layers

Go to the layers directory and clone the meta-raspberrypi Wrynose branch:

```bash
git clone -b wrynose https://git.yoctoproject.org/meta-raspberrypi
```

Fragments are the nice part now. Instead of manually appending `IMAGE_INSTALL`, `ENABLE_UART`, WiFi firmware, etc. into `local.conf`, you toggle fragments (wifi, bluetooth, debug-tweaks, etc.) and it composes the config for you. This greatly reduces the chance of silently breaking `local.conf` syntax.

4. Enable the fragment for the RPi machine:

```bash
$ bitbake-config-build enable-fragment machine/raspberrypi4-64
```
5. Accept the non-free firmware license (Pi boards need Broadcom/VideoCore firmware). Add to `build/conf/local.conf`:

```bash
LICENSE_FLAGS_ACCEPTED += "synaptics-killswitch"
```

6. Finally, build the image:

```bash
$ bitbake core-image-minimal
```

### Issues Encountered

**1. AppArmor blocking Bitbake on Ubuntu**

I am building on Ubuntu bare metal, and Bitbake's use of unprivileged user namespaces is blocked by AppArmor's unprivileged user namespace restrictions.

**Fix:** Add a permissive AppArmor profile for unprivileged user namespaces rather than disabling AppArmor globally. 



**2. Flashing to SD card — device path confusion**

When using bmaptool to write to my SD card (which showed as sda), the tool would sometimes write to the wrong device. 

```bash

bmaptool copy core-image-minimal-raspberrypi4-64.rootfs.wic.bz2 /dev/sdbbmaptool: info: discovered bmap file 'core-image-minimal-raspberrypi4-64.rootfs.wic.bmap'
bmaptool: WARNING: "/dev/sdb" does not exist, creating a regular file "/dev/sdb"
bmaptool: ERROR: An error occurred, here is the traceback:
Traceback (most recent call last):
  File "/usr/lib/python3/dist-packages/bmaptool/CLI.py", line 591, in open_files
    dest_obj = open(args.dest, "wb+")
bmaptool: ERROR: cannot open destination file '/dev/sdb':
[Errno 13] Permission denied: '/dev/sdb'
surya@dell:~/bitbake-builds/rpi4b/build/tmp/deploy/images/raspberrypi4-64$ lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda                         8:0    1  29.7G  0 disk
└─sda1                      8:1    1  29.7G  0 part
nvme0n1                   259:0    0 931.5G  0 disk
├─nvme0n1p1               259:1    0     1G  0 part /boot/efi
├─nvme0n1p2               259:2    0     2G  0 part /boot
└─nvme0n1p3               259:3    0 928.5G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0 928.5G  0 lvm  /
```



**Fix:** Use the stable by-id device path instead:

```bash
ls -la /dev/disk/by-id/ | grep -i usb
sudo bmaptool copy build/tmp/deploy/images/raspberrypi4-64/core-image-minimal-raspberrypi4-64.wic.bz2 /dev/disk/by-id/usb-<exact-id>
```



### Takeaway

Total time from zero to a booting `core-image-minimal` on RPi 4B: well under the time it took just to configure bblayers/local.conf correctly by hand in 2024. The fragment model eliminates most of the copy-paste-from-forum-post failure modes.

See `rpi4b.md` for the full RPi 4B steps.

## Part 2: RK3288

Two threads, kept separate on purpose:

- EVB on Scarthgap: see `rk3288-evb-scarthgap.md` (Build 1 minimal + Build 2 HDMI terminal, `usrmerge` and `fiq_debugger` fixes, `rkdeveloptool` flashing).
- Generic reference-design board: see `rk3288-generic.md` (PMIC close-up, board photo, Firefly 6.18 boot logs). The port from the EVB config is still in progress.



