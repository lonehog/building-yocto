# RPi 4B — core-image-minimal with bitbake-setup (Wrynose 6.0)

Warm-up build used to learn the new `bitbake-setup` wizard and fragments before touching the RK3288. Full index in `README.md`.

## Setup

The wizard introduced in Wrynose 6.0 sets up bitbake layers:

1. Template: Wrynose (latest LTS)
2. Bitbake config: `poky` (option 1, no sstate acceleration)
3. Target machine: `genericarm64`

```bash
$ ./bitbake/bin/bitbake-setup init
NOTE: Bitbake-setup is using /home/surya/bitbake-builds as top directory.

Available Configuration Templates:
1. oe-nodistro-master
2. oe-nodistro-wrynose  (release 6.0 'wrynose', supported until 2030-05-31)
3. poky-master
4. poky-wrynose         (release 6.0 'wrynose', supported until 2030-05-31)

Please select one of the above configurations by its number: 4

Available bitbake configurations:
1. poky
2. poky-with-sstate

Please select one of the above bitbake configurations by its number: 1

Target machines:
1. machine/qemux86-64
2. machine/qemuarm64
3. machine/qemuriscv64
4. machine/genericarm64   Arm64 SystemReady IR/ES platforms
5. machine/genericx86-64

Please select one of the above options by its number: 4
```

Source the build environment:

```bash
$ source build/init-build-env
```

Add meta-raspberrypi layers:

```bash
git clone -b wrynose https://git.yoctoproject.org/meta-raspberrypi
```

Fragments replace manual `local.conf` edits. Instead of appending `IMAGE_INSTALL`, `ENABLE_UART`, WiFi firmware, etc. by hand, toggle fragments and the config is composed for you.

Enable the fragment for the RPi machine:

```bash
$ bitbake-config-build enable-fragment machine/raspberrypi4-64
```

Accept the non-free firmware license (Pi boards need Broadcom/VideoCore firmware) in `build/conf/local.conf`:

```bitbake
LICENSE_FLAGS_ACCEPTED += "synaptics-killswitch"
```

Build:

```bash
$ bitbake core-image-minimal
```

## Issues encountered

### 1. AppArmor blocking Bitbake on Ubuntu

Building on Ubuntu bare metal, Bitbake's use of unprivileged user namespaces is blocked by AppArmor's restrictions.

Fix: add a permissive AppArmor profile for unprivileged user namespaces rather than disabling AppArmor globally.

### 2. Flashing to SD card — device path confusion

`bmaptool` with `/dev/sdb` failed when the card showed as `sda`:

```text
bmaptool: WARNING: "/dev/sdb" does not exist, creating a regular file "/dev/sdb"
bmaptool: ERROR: cannot open destination file '/dev/sdb': [Errno 13] Permission denied
```

Fix: use the stable by-id device path:

```bash
ls -la /dev/disk/by-id/ | grep -i usb
sudo bmaptool copy build/tmp/deploy/images/raspberrypi4-64/core-image-minimal-raspberrypi4-64.wic.bz2 /dev/disk/by-id/usb-<exact-id>
```

## Takeaway

Zero to booting `core-image-minimal` on RPi 4B took well under the time it took just to configure bblayers/local.conf by hand in 2024. The fragment model removes most copy-paste failure modes.

Booting to a login prompt on Poky 6.0.2:

![RPi 4B boot log](images/rpi4b-boot-log.jpg)
