# beagleV-yocto-image
# BeagleV-Ahead Yocto Build and SD Boot 

This document records the work I did on the BeagleV-Ahead in the same
sequence I followed it, from preparing the Yocto environment to building
the image, flashing the SD card,and testing the boot process.

------------------------------------------------------------------------

For the BeagleV-Ahead build, I needed BitBake, OpenEmbedded-Core, and
the RISC-V BSP layer.

I cloned:

``` bash
git clone https://git.openembedded.org/bitbake
git clone https://git.openembedded.org/openembedded-core
git clone https://github.com/riscv/meta-riscv.git
```

At first, I had tried using a Yocto release branch together with a newer
`meta-riscv` branch. This caused parsing errors because the layers were
not compatible with each other.
The resulting workspace was:

``` text
~/yocto/
├── bitbake/
├── openembedded-core/
├── meta-riscv/
└── build-beaglev-ahead/
```

------------------------------------------------------------------------

## 2. Preparing Python

I used Python 3.10.14 through `pyenv`.

``` bash
pyenv shell 3.10.14
python3 --version
```

The output was:

``` text
Python 3.10.14
```

------------------------------------------------------------------------

## 3. Creating the BeagleV-Ahead build environment

From the Yocto workspace, I initialized a separate build directory for
the board:

``` bash
cd ~/yocto
source openembedded-core/oe-init-build-env build-beaglev-ahead
```

This created/entered:

``` text
~/yocto/build-beaglev-ahead
```

I then made sure that the build included the required layers:

``` text
/home/adn/yocto/openembedded-core/meta
/home/adn/yocto/meta-riscv
```

I checked them with:

``` bash
bitbake-layers show-layers
```

------------------------------------------------------------------------

## 4. Selecting the target board

I configured the target machine in:

``` text
~/yocto/build-beaglev-ahead/conf/local.conf
```

using:

``` conf
MACHINE = "beaglev-ahead"
```

I verified that BitBake was actually using this machine with:

``` bash
bitbake -e core-image-minimal | grep '^MACHINE='
```

which showed:

``` text
MACHINE="beaglev-ahead"
```

The target system was RISC-V 64-bit:

``` text
TARGET_SYS = riscv64-oe-linux
```

------------------------------------------------------------------------

## 5. Limiting the build to six parallel jobs
 I
limited the parallelism in `local.conf`:

``` conf
BB_NUMBER_THREADS = "6"
PARALLEL_MAKE = "-j 6"
```
------------------------------------------------------------------------

## 6. Checking the BeagleV-Ahead machine configuration

Before building, I inspected the board configuration in:

``` text
meta-riscv/conf/machine/beaglev-ahead.conf
```

The configuration selects:

``` conf
PREFERRED_PROVIDER_virtual/kernel ?= "linux-beaglev-dev"
PREFERRED_PROVIDER_virtual/bootloader ?= "u-boot-beaglev-ahead"
PREFERRED_PROVIDER_opensbi ?= "opensbi-revyos"
```

It also uses:

``` conf
WKS_FILE ?= "beaglev-ahead.wks"
```

and adds the BeagleV-Ahead firmware and bootloader dependencies.

A line in this file later became important during the SD boot
investigation:

 I continued with the normal Yocto image build.

------------------------------------------------------------------------

## 7. Building `core-image-minimal`

I started the image build with:

``` bash
bitbake core-image-minimal
```

The build completed successfully.

All tasks were executed successfully.

The important point here is that the Yocto build itself was successful.

------------------------------------------------------------------------

## 8. Checking the generated files

After the build completed, I went to:

``` bash
cd ~/yocto/build-beaglev-ahead/tmp/deploy/images/beaglev-ahead/
```

The file I chose for the SD card was the generated WIC image:

`core-image-minimal-beaglev-ahead.rootfs-20260901190201.wic.gz`

I chose the WIC image because I wanted to write a complete disk image to the
SD card rather than flash the generated components individually. A WIC image
contains the disk partition table and the partitions populated according to
the board's `.wks` file.

For BeagleV-Ahead, the build uses:

`meta-riscv/files/wic/beaglev-ahead.wks`

so the generated WIC contains the GPT ,guid partition table, partition layout, the boot partition,
and the root filesystem partition.

Therefore, at this stage I expected the `.wic` image to be the appropriate
artifact to write directly to the whole SD card using `dd`.

```

------------------------------------------------------------------------

## 9. Inspecting the WIC image definition

Before/while investigating the image, I checked the WKS file used to
construct it:

``` text
meta-riscv/files/wic/beaglev-ahead.wks
```

It contains:

``` text
# short-description: Create SD card image for BeagleV-ahead
bootloader --ptable gpt

part --label empty --part-name empty --align 4096 --size=2M

part /boot --source bootimg-partition --fstype=ext4 \
     --label boot --part-name boot --align 4096 --size=100M

part / --source rootfs --fstype=ext4 \
     --label root --part-name root --align 4096 --size 1G
```

So the WIC image creates a GPT disk image containing an empty partition,
a boot partition, and a root filesystem partition.

------------------------------------------------------------------------

## 10. Connecting the SD card

I inserted a 32 GB SD card into the laptop using an SD card reader.

I checked the available block devices before writing anything:

``` bash
lsblk -o NAME,SIZE,MODEL,TRAN,FSTYPE,MOUNTPOINTS
```

The SD card appeared as:

``` text
/dev/sda
```

My laptop's internal disk was:

``` text
/dev/nvme0n1
```

------------------------------------------------------------------------

## 11. Unmounting the SD card

Before writing the disk image, I unmounted the existing SD partitions:

``` bash
sudo umount /dev/sda1 2>/dev/null
sudo umount /dev/sda2 2>/dev/null
```

------------------------------------------------------------------------

## 12. Flashing the Yocto WIC image to the SD card

I decompressed the generated `.wic.gz` image to obtain the `.wic` disk
image.

The resulting image was used as:

``` text
beaglev-ahead.wic
```

I then wrote the complete WIC disk image directly to the complete SD
card:

``` bash
sudo dd if=beaglev-ahead.wic of=/dev/sda bs=4M status=progress conv=fsync
```

Here, `dd` performs a raw block-level copy.

This means I did not simply copy `beaglev-ahead.wic` as a normal file
into the SD card.

Instead, the operation was:

``` text
beaglev-ahead.wic
        |
        | dd
        v
     /dev/sda
  entire SD card
```

The operation wrote approximately bytes to the SD card.

------------------------------------------------------------------------

## 13. Checking the SD card after flashing

After `dd` completed, I checked the SD card again:

``` bash
lsblk /dev/sda
```

It showed:

``` text
sda
├─sda1    2M
├─sda2  130M
└─sda3  1.3G
```

I also inspected the partition table with:

``` bash
sudo fdisk -l /dev/sda
```

The three partitions were visible.

`fdisk` also printed GPT warnings because the generated WIC image was
around 1.44 GiB while the physical SD card was around 29.1 GiB:

``` text
GPT PMBR size mismatch
The backup GPT table is not on the end of the device
```

I did not assume that these warnings were the reason for the boot
failure; they only showed that the disk image was smaller than the
physical card it had been written to.

------------------------------------------------------------------------

## 14. Connecting UART to observe the board boot

To see exactly what the board was doing during boot, I connected an FTDI
UART adapter:

On Ubuntu it appeared as:

``` text
/dev/ttyUSB0
```

I opened the serial terminal using:

``` bash
sudo minicom -D /dev/ttyUSB0 -b 115200
```

------------------------------------------------------------------------

## 15. Testing the board with a normal boot

First, I powered the board normally without forcing SD boot.

The board booted from its existing onboard eMMC installation.

From UART I could see U-Boot SPL, U-Boot, the kernel loading, and
eventually Linux reaching the login stage.

The sequence visible from the existing eMMC installation was
approximately:

``` text
Power / Reset
      ↓
BootROM
      ↓
U-Boot SPL
      ↓
U-Boot
      ↓
extlinux
      ↓
Linux Image + Device Tree
      ↓
Starting kernel ...
      ↓
Linux
      ↓
login
```

U-Boot reported the MMC devices:

``` text
sdhci@ffe7080000: 0
sd@ffe7090000: 1
```

The normal boot selected:

``` text
mmc0
```

which was the eMMC.

It found:

``` text
/extlinux/extlinux.conf
```

and loaded the existing Linux kernel and Device Tree successfully.

This confirmed that the board itself was working and that the existing
eMMC installation provided a known-good boot path.

------------------------------------------------------------------------

## 16. Trying to boot from the SD card

I then checked the BeagleV-Ahead boot controls.

The board documentation shows that SD boot is selected by holding the
**SD BOOT** button while resetting or power-cycling the board.

I inserted the flashed SD card, held the SD BOOT button, and
powered/reset the board while watching the UART output.

Instead of reaching SPL or U-Boot from the SD card, the UART showed:

``` text
brom_ver 8
[APP][E] protocol_connect failed, exit.
[APP][E] mtdt read e.
[APP][E] mtdt read e.
[APP][E] sd card boot error.
```

The messages repeated while I continued holding the SD BOOT button.

The sequence I observed was therefore:

``` text
Power / Reset
      ↓
BootROM
      ↓
SD boot selected/attempted
      ↓
sd card boot error
```

When I released the SD BOOT button, the board returned to its normal
eMMC boot path.

So the eMMC boot was not automatically occurring while I continued
holding the SD button. The BootROM kept trying the SD path until I
released it.

Official board documentation:

https://docs.beagleboard.org/boards/beaglev/ahead/01-introduction.html

https://docs.beagleboard.org/beaglev-ahead.pdf

------------------------------------------------------------------------

## 17. Checking whether Linux could see the SD card

Because the BootROM reported an SD boot error, I wanted to distinguish
between an unreadable/broken SD card and an early boot-layout problem.

I allowed the board to boot normally from its existing eMMC system.

Once Linux started, it detected the SD card as another MMC device.

The SD appeared as:

``` text
mmcblk1
```

and Linux could see:

``` text
mmcblk1p1
mmcblk1p2
mmcblk1p3
```

This matched the partitions written by the WIC image.

Therefore, the SD card and its partition table could be detected by
Linux after the system had already booted.

The observed failure was specifically happening much earlier, during the
BootROM SD boot attempt.

------------------------------------------------------------------------

## 18. Going back to the Yocto BSP to understand the failure

After seeing:

``` text
sd card boot error
```

I went back to the BeagleV-Ahead machine configuration.

The important comment was:

``` conf
# Note: currently just a fastboot deployment is supported
# for which a separate boot.ext4 is generated
```

This is in:

``` text
meta-riscv/conf/machine/beaglev-ahead.conf
```

This changed the way I interpreted the result.

The board hardware supports selecting SD as a boot source, but the
current upstream Yocto BSP explicitly says that the currently supported
deployment method is **fastboot**.

Therefore, I did not conclude that the hardware cannot boot from SD.

Instead, I concluded that directly writing the WIC generated by this BSP
to an SD card with `dd` was not producing a working SD boot in this
setup.

------------------------------------------------------------------------

## 19. Checking `boot.ext4`

I also inspected the current BeagleV-Ahead kernel recipe:

``` text
meta-riscv/recipes-kernel/linux/linux-beaglev-dev.bb
```

The recipe specifically creates a separate:

``` text
boot.ext4
```

for the boot partition and comments that it can be flashed through
fastboot.

This agreed with what I had already found in `beaglev-ahead.conf`.

Reference:

https://github.com/riscv/meta-riscv/blob/master/recipes-kernel/linux/linux-beaglev-dev.bb

------------------------------------------------------------------------

## 20. Understanding fastboot and eMMC

At this point I separated two concepts:

``` text
eMMC     = onboard storage on the BeagleV-Ahead
fastboot = protocol/tool used to flash images to the board over USB
```

So the supported deployment path described by the BSP is conceptually:

``` text
Laptop
   |
   | USB / fastboot
   v
BeagleV-Ahead
   |
   v
eMMC
```


------------------------------------------------------------------------

## 21. Fastboot check

I installed/used the fastboot tool and at one point issued:

``` bash
sudo fastboot flash ram u-boot-with-spl.bin
```

The command only displayed:

``` text
< waiting for any device >
```

I cancelled it.

Since no fastboot device had been detected, nothing was flashed by that
attempt.

The existing eMMC installation remained unchanged.

------------------------------------------------------------------------


References:

https://lists.buildroot.org/pipermail/buildroot/2025-January/769874.html

https://lists.buildroot.org/pipermail/buildroot/2023-August/735639.html

https://github.com/pengutronix/genimage/blob/master/README.rst

I treated this only as useful comparison evidence. I did not treat it as
official TH1520 BootROM documentation, and I did not claim an exact
BootROM SD offset from it.

-----------------------------------------------------------------------
## References

BeagleV-Ahead documentation:

https://docs.beagleboard.org/boards/beaglev/ahead/01-introduction.html

https://docs.beagleboard.org/beaglev-ahead.pdf

BeagleV-Ahead Yocto machine configuration:

https://github.com/riscv/meta-riscv/blob/master/conf/machine/beaglev-ahead.conf

BeagleV-Ahead kernel recipe:

https://github.com/riscv/meta-riscv/blob/master/recipes-kernel/linux/linux-beaglev-dev.bb

meta-riscv:

https://github.com/riscv/meta-riscv

OpenEmbedded-Core:

https://git.openembedded.org/openembedded-core

BitBake:

https://git.openembedded.org/bitbake

U-Boot SPL documentation:

https://docs.u-boot-project.org/en/latest/usage/spl_boot.html

OpenSBI firmware documentation:

https://github.com/riscv-software-src/opensbi/blob/master/docs/firmware/fw.md

Buildroot BeagleV-Ahead SD image discussions:

https://lists.buildroot.org/pipermail/buildroot/2025-January/769874.html

https://lists.buildroot.org/pipermail/buildroot/2023-August/735639.html

genimage documentation:

https://github.com/pengutronix/genimage/blob/master/README.rst
