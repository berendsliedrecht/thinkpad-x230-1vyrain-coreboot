# ThinkPad X230 1vyrain + coreboot installation process (UEFI)

Flash coreboot on a Lenovo ThinkPad X230 with 1vyrain, without opening the laptop and without a CH341A programmer, Raspberry Pi or SOIC clip. The whole process is a BIOS downgrade, a coreboot build with the tianocore/edk2 (UEFI) payload, and a flash over the local network. 1vyrain needs BIOS 2.60 or lower, so the first part is the downgrade.

Last verified: 2026-09-28

> **Note:**
>
> - This may also work for other Ivy Bridge ThinkPads, but you will need a different BIOS to downgrade to and some tweaks along the way.
> - You need a second PC to build and host the ROM. The X230 must be connected to the same local network via ethernet.
> - Written for Linux.

Setup used:

- ThinkPad X230, i5-3380M, 12 MiB flash (8 MiB + 4 MiB chip)
- coreboot + tianocore/edk2 (MrChromebox)
- Build PC on Arch Linux

1. [Downgrade the BIOS to 2.60](#downgrade-the-bios-to-260)
2. [Build coreboot](#build-coreboot)
3. [Flash with 1vyrain](#flash-with-1vyrain)

## Downgrade the BIOS to 2.60

> **Note:** Based on the guide by [gch1p](https://github.com/gch1p/thinkpad-bios-software-flashing-guide). If something does not work, check there.

Check the BIOS version by pressing `F1` during startup. If it is **2.60 or lower**, skip this part.

### Get the boot image

Create the working directory, download the BIOS from Lenovo and extract the boot image.

```bash
mkdir x230
cd x230
curl -LO https://download.lenovo.com/pccbbs/mobiles/g2uj17us.iso
git clone https://github.com/rainer042/geteltorito
perl geteltorito/geteltorito.pl -o bios.img g2uj17us.iso
```

### Patch the boot image to use `dosflash.exe`

Mount the image and list the `FLASH` directory.

```bash
sudo mount -t vfat ./bios.img /mnt -o loop,offset=16384
ls /mnt/FLASH
```

This shows a directory like `G2ETA0WW`. List it as well.

```bash
ls /mnt/FLASH/G2ETA0WW
```

This shows a file ending in `.FL1`, like `$01D3000.FL1`.

> **Note:** If your directory or file name differ, replace `G2ETA0WW` and `$01D3000.FL1` in the `sed` command below.

```bash
sed -i 's|.*command\.com.*|dosflash.exe /sd /file G2ETA0WW\\$01D3000.FL1\r|I' /mnt/AUTOEXEC.BAT
sudo umount /mnt
```

### Flash the boot image to a USB stick

Find the USB stick with `lsblk` and replace `/dev/sdX` below.

> **Note:** This wipes the USB stick.

```bash
sudo dd if=./bios.img of=/dev/sdX bs=1M
sync
```

### Run the BIOS downgrade

> **Note:** Keep the battery in and the AC adapter connected during the whole process.

Press `F1` during startup, go to `Startup` and set the boot mode to `Legacy Only`, or `Both` with `Legacy First`. Save and restart.

Press `F12` during startup, select the USB stick and follow the on-screen instructions. The laptop turns off and on again during the flash, this is normal.

Press `F1` during startup, check that the version is now **2.60**, and set the boot mode back to `UEFI Only`, or `Both` with `UEFI First`. Save and restart.

## Build coreboot

> **Note:** Build on the second PC. Building on the X230 takes hours, and the ROM has to be hosted from the PC anyway.

This uses coreboot with the tianocore/edk2 payload from MrChromebox. For other payloads see the [coreboot docs](https://doc.coreboot.org/payloads.html).

### Install dependencies and clone coreboot

See the [coreboot docs](https://doc.coreboot.org/tutorial/part1.html) for other distros. On Arch:

```bash
sudo pacman -S --needed base-devel gcc-ada curl git ncurses zlib nasm python acpica
```

```bash
git clone https://review.coreboot.org/coreboot
git -C coreboot submodule update --init --checkout
make -C coreboot crossgcc-i386 CPUS=$(nproc)
```

### Configure

```bash
cat > x230.defconfig <<EOF
CONFIG_VENDOR_LENOVO=y
CONFIG_BOARD_LENOVO_X230=y
CONFIG_CBFS_SIZE=0x400000
CONFIG_PAYLOAD_EDK2=y
CONFIG_EDK2_REPO_MRCHROMEBOX=y
EOF
```

> **Note:** `CONFIG_CBFS_SIZE=0x400000` is required. 1vyrain only flashes the 4 MiB chip, so CBFS must fit in it. The default (0x700000) does not.

> **Note:** Older coreboot versions use `CONFIG_PAYLOAD_TIANOCORE` and `CONFIG_TIANOCORE_REPO_MRCHROMEBOX`. Check with `grep -i mrchromebox coreboot/payloads/external/Kconfig`.

```bash
make -C coreboot clean
make -C coreboot defconfig KBUILD_DEFCONFIG=$PWD/x230.defconfig
grep CBFS_SIZE coreboot/.config
```

The `grep` must print `CONFIG_CBFS_SIZE=0x400000`.

> **Note:** Run `make -C coreboot clean` again after changing the defconfig.

### Build and host

```bash
make -C coreboot -j$(nproc)
```

The first build also builds edk2 and takes a while. The result is `coreboot/build/coreboot.rom`. Host it:

```bash
python3 -m http.server 8000 --directory coreboot/build
```

Get the IP of the PC with `ip addr`. The ROM is now at `http://<ip>:8000/coreboot.rom`. Keep this running.

## Flash with 1vyrain

### Prepare

Connect the X230 with an ethernet cable to the same local network as the PC. Check from the PC that the ROM is reachable:

```bash
curl -sI http://<ip>:8000/coreboot.rom | head -1
```

Download the ISO from [1vyra.in](https://1vyra.in) and save it as `1vyrain.iso` in the working directory. Find the USB stick with `lsblk` and replace `/dev/sdX` below.

> **Note:** The `dd` flags matter. With only `bs=1M` the stick did not boot.

```bash
sudo dd if=./1vyrain.iso of=/dev/sdX bs=4M conv=fsync oflag=direct status=progress
```

### Flash

> **Note:** Keep the battery in and the AC adapter connected during the whole process.

Boot mode must be UEFI (set at the end of part 1). Press `F12` during startup and select the USB stick.

In the 1vyrain menu select **option 2** (flash custom ROM) and enter the URL:

```
http://<ip>:8000/coreboot.rom
```

Confirm. The laptop suspends and resumes on its own, this is the exploit. After that 1vyrain flashes the ROM and reports when it is done.

Remove the USB stick and power on. The edk2 boot menu shows instead of the Lenovo logo. `Esc` or `F2` opens the setup.

edk2 boots `\EFI\BOOT\BOOTX64.EFI` by default. My loader was already at that path, so it booted straight into the OS.
