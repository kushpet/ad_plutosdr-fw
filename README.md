# plutosdr-fw (LibreSDR fork)

Personal fork of ADI's [plutosdr-fw](https://github.com/analogdevicesinc/plutosdr-fw) with LibreSDR (ZynqSDR) board support. Clone it recursively and it builds the `libre` target directly; there is no `apply.sh` step.

Original PlutoSDR wiki instructions (upstream/Pluto only, for reference): [Building the image](https://wiki.analog.com/university/tools/pluto/building_the_image)

## Branches and tags

The main branch is `libresdr`. The same branch name is used in all four submodule forks (`kushpet/ad_hdl`, `ad_linux`, `ad_buildroot`, `ad_u-boot-xlnx`).

| Ref | Base | Toolchain | Notes |
| --- | --- | --- | --- |
| tag `libre-v0.37` | ADI v0.37 + [day0wl](https://github.com/day0wl/libresdr-fw) patch + local fixes | Vivado/Vitis **2021.2** | Tested on hardware. AD9363 in CMOS mode. |
| tag `libre-v0.38` | ADI v0.38 + the same LibreSDR support | Vivado **2022.2** | Functionally the same as `libre-v0.37` (CMOS mode, no overclock). |
| `libresdr` (tip) | `libre-v0.38` + switch to LVDS | Vivado **2022.2** | AD9363 data interface in LVDS mode (the board is routed for it). |

In each repository, the `libre-v0.38` commit is a merge: the first parent is `libre-v0.37` and the second is the ADI v0.38 commit. The ADI history therefore stays reachable through the branch.

The v0.38 port was cross-checked against [hz12opensource/libresdr](https://github.com/hz12opensource/libresdr). Its CPU overclock (750 MHz on an XC7Z020-1), out-of-spec DDR timings, `-O3`/Spectre kernel hacks and iiod realtime tweaks were deliberately left out. The old branches `libre_v0.37` and `libre_v0.38` are kept unchanged for reference.

Schematics: `docs/plutosdr_schematic_revd_0.1.pdf` (original ADALM-PLUTO) and `docs/zynqsdr_rev5.pdf` (LibreSDR rev5).

## Build Instructions

### Host setup (once)

```bash
sudo apt-get install git build-essential fakeroot libncurses5-dev libssl-dev ccache
sudo apt-get install dfu-util u-boot-tools device-tree-compiler mtools dosfstools parted
sudo apt-get install bc python3 cpio zip unzip rsync file wget

git clone --recursive https://github.com/kushpet/ad_plutosdr-fw.git -b libresdr
cd ad_plutosdr-fw
```

PetaLinux is **not** used. The build runs Vivado and Vitis (for the bitstream and FSBL) and buildroot (for the cross toolchain and the root filesystem).

### Environment variables

| Variable | Required | Value |
| --- | --- | --- |
| `VIVADO_SETTINGS` | yes | `settings64.sh` of **Vitis 2022.2**, e.g. `/opt/Xilinx/Vitis/2022.2/settings64.sh`. Use the Vitis one, not Vivado's: it puts both `vivado` and `xsct` on the PATH. `xsct` builds the FSBL. Without this variable the Makefile assumes `/opt/Xilinx/Vivado/2022.2/settings64.sh`. |
| `BR2_DL_DIR` | no | A persistent directory for buildroot source downloads, e.g. `$HOME/buildroot-dl`. The first build needs internet access; later builds work offline from this cache. |

Nothing else is needed. `TARGET` defaults to `libre`. `CROSS_COMPILE` is set by the Makefile, and the Linaro GCC 7.3 toolchain is installed by buildroot itself into `buildroot/output/host`.

```bash
export VIVADO_SETTINGS=/opt/Xilinx/Vitis/2022.2/settings64.sh
export BR2_DL_DIR=$HOME/buildroot-dl

make          # firmware for flashing:   build/boot.frm, build/libre.frm, *.dfu, zip
make sdimg    # SD card boot files:      build_sdimg/
```

- A full build takes about 1–1.5 h on 8 cores. Most of the time is spent in Vivado and buildroot.
- `make` starts with `clean-build`, which **deletes `build/` and `build_sdimg/`**. Copy any images you want to keep somewhere else first.
- Update the submodules after every `git pull` or checkout: `git submodule update --init --recursive`.
- When switching between revisions built with different Vivado versions (for example, `libre-v0.37` with 2021.2), run `git clean -fdX` in `hdl/` first. Otherwise stale IP builds are reused.

To build `libre-v0.37`: `git checkout libre-v0.37 && git submodule update`. Then use Vitis 2021.2 for `VIVADO_SETTINGS`, and also set `CROSS_COMPILE=arm-linux-gnueabihf-` and add `<Vitis 2021.2>/gnu/aarch32/lin/gcc-arm-linux-gnueabi/bin` to `PATH`.

Optional CPU/DDR overclock (out of spec, at your own risk): after `make`, run `make overclock OVERCLOCK_CPU_MULT=<n> OVERCLOCK_DDR_MULT=<n>`, then `make sdimg`. This only patches the PLL multipliers in the FSBL's `ps7_init.c`. The DDR timings stay at the 525 MHz preset.

## Booting from an SD card (flash is not touched)

The board picks the boot source in hardware. The SD card-detect signal drives Zynq `BOOT_MODE[2]` (MIO4, see `docs/zynqsdr_rev5.pdf`). With a card inserted, the BootROM boots from SD; without one, it boots from QSPI flash. Booting from SD therefore works whatever is in the flash, and it is also the recovery path if a flash update goes wrong.

**One FAT32 partition is enough.** u-boot loads all files from the first partition (`load mmc 0 ...`), and the root filesystem is a ramdisk (`uramdisk.image.gz`) unpacked into RAM. The two-partition layout (FAT `BOOT` + ext4 `rootfs`) that you may have seen elsewhere belongs to ADI Kuiper Linux and similar full Linux images that keep their root filesystem on the card. This firmware does not use a second partition. One will not hurt, but it is ignored.

```bash
lsblk                                   # find the card, e.g. /dev/sdX -- double-check, this erases it!
sudo umount /dev/sdX?*                  # unmount any auto-mounted partitions
sudo parted -s /dev/sdX mklabel msdos mkpart primary fat32 1MiB 100%
sudo mkfs.vfat -F 32 -n LIBRESDR /dev/sdX1
sudo mount /dev/sdX1 /mnt
sudo cp build_sdimg/{BOOT.bin,uImage,devicetree.dtb,uEnv.txt,uramdisk.image.gz} /mnt/
sync && sudo umount /mnt
```

Only these 5 files are needed. The other files in `build_sdimg/` (`fsbl.elf`, `system_top.bit`, `u-boot.elf`, `boot.bif`, `ramdisk.image.gz`) are intermediate files that `BOOT.bin` and `uramdisk.image.gz` were made from.

Insert the card, connect the board and power it on. Signs that it booted:
- `pl_led1` blinks a heartbeat.
- On the debug USB port (onboard FT2232HQ), the serial console `/dev/ttyUSB2` (115200 8N1) shows the u-boot and kernel log. u-boot prints `Copying Linux from SD to RAM...`.
- On the OTG USB port, the board appears like a PlutoSDR: an RNDIS network interface with the board at `192.168.2.1` (your PC gets `192.168.2.10` over DHCP), plus a USB drive named **PlutoSDR**.
- Gigabit Ethernet comes up with the static address `192.168.1.10`.
- Log in with `ssh root@192.168.2.1` (password `analog`), or test the radio with `iio_info -u ip:192.168.2.1`.

## Writing the firmware to the internal QSPI flash

This is done from a running board, booted either from SD or from the existing flash, through the **PlutoSDR** USB drive, the same way as on a stock Pluto. The board checks the md5 sum embedded in each `.frm` file before writing it.

1. Copy `build/boot.frm` (FSBL, bitstream, u-boot and the default u-boot environment) to the root of the PlutoSDR drive.
2. **Eject** the drive; do not unplug it. On Linux use `eject /dev/sdY`, where `sdY` is the PlutoSDR drive, or "Eject" in the file manager. Ejecting is what triggers the update. `pl_led1` blinks fast while the flash is being written. When it finishes, the board reboots. With the SD card still inserted, it boots from SD again, which is fine. When the drive reappears, it contains `BOOT_SUCCESS` on success, or `BOOT_FAILED` / `FAILED_*` on failure.
3. Copy `build/libre.frm` (kernel, device tree and rootfs) to the drive and eject it again. When it finishes, the drive contains `SUCCESS` (or `FAILED*`).
4. Power off, remove the SD card and power on. The board now boots from QSPI flash.

Do `boot.frm` first and `libre.frm` second. Writing `boot.frm` resets the u-boot environment, including `fit_size`, which `libre.frm` sets to the image size. The board still boots if the order is reversed, because u-boot falls back to reading the whole 30 MB partition, but booting is slower. For later updates, usually only `libre.frm` is needed. You can also copy the whole `libresdr-fw-*.zip` to the drive, and the board unpacks the `.frm` files from it itself.

If the board does not boot from flash afterwards, insert the SD card. It always takes priority, so you can repeat the procedure.

## Troubleshooting

If you receive an error similar to the following:
```
Starting SDK. This could take few seconds... timeout while establishing a connection with SDK
   while executing
"error "timeout while establishing a connection with SDK""
   (procedure "getsdkchan" line 108)
   invoked from within
"getsdkchan"
   (procedure "createhw" line 26)
   invoked from within
"createhw {*}$args"
   (procedure "::sdk::create_hw_project" line 3)
   invoked from within
"sdk create_hw_project -name hw_0 -hwspec build/system_top.hdf"
   (file "scripts/create_fsbl_project.tcl" line 5)
```
you may be able to work around it by preventing eclipse from using GTK3 for the Standard Widget Toolkit (SWT). Prior to running `make`, also set:
```bash
export SWT_GTK3=0
```
This problem seems to affect Ubuntu 16.04LTS only.

## Build Artifacts

```
build/
├── boot.bif
├── boot.bin
├── boot.dfu
├── boot.frm
├── libre.dfu
├── libre.frm
├── libre.itb
├── libresdr-fw-vX.XX.zip
├── rootfs.cpio.gz
├── sdk/
├── system_top.bit
├── system_top.xsa
├── u-boot.elf
├── uboot-env.bin
├── uboot-env.txt
├── zImage
└── zynq-libre.dtb

build_sdimg/            # produced by `make sdimg`, ready to copy to an SD card
├── BOOT.bin
├── boot.bif
├── devicetree.dtb
├── fsbl.elf
├── system_top.bit
├── u-boot.elf
├── uEnv.txt
├── uImage
├── uramdisk.image.gz
└── ramdisk.image.gz
```

### Main targets

| File | Comment |
| ------------- | ------------- |
| libre.frm | Main LibreSDR firmware file used with the USB Mass Storage Device |
| libre.dfu | Main LibreSDR firmware file used in DFU mode |
| boot.frm  | First and Second Stage Bootloader (u-boot + fsbl + uEnv) used with the USB Mass Storage Device |
| boot.dfu  | First and Second Stage Bootloader (u-boot + fsbl) used in DFU mode |
| uboot-env.dfu  | u-boot default environment used in DFU mode |
| libresdr-fw-vX.XX.zip  | ZIP archive containing all of the files above |

### `make sdimg` targets (SD card boot)

| File | Comment |
| ------------- | ------------- |
| BOOT.bin | FSBL + bitstream + u-boot boot image for the SD card |
| uImage | u-boot-wrapped Linux kernel image |
| devicetree.dtb | Device Tree Blob for LibreSDR (`zynq-libre.dtb`) |
| uEnv.txt | u-boot environment used to boot from the SD card |
| uramdisk.image.gz | u-boot-wrapped root filesystem ramdisk |

### Other intermediate targets

| File | Comment |
| ------------- | ------------- |
| boot.bif | Boot Image Format file used to generate the Boot Image |
| boot.bin | Final Boot Image |
| libre.itb | u-boot Flattened Image Tree |
| rootfs.cpio.gz | The Root Filesystem archive |
| sdk | Vivado/XSDK Build folder including the FSBL |
| system_top.bit | FPGA Bitstream (from HDF/XSA) |
| system_top.xsa | FPGA Hardware Description File exported by Vivado |
| u-boot.elf | u-boot ELF Binary |
| uboot-env.bin | u-boot default environment in binary format created from uboot-env.txt |
| uboot-env.txt | u-boot default environment in human readable text format |
| zImage | Compressed Linux Kernel Image |
| zynq-libre.dtb | Device Tree Blob for LibreSDR |

## Credits

- [analogdevicesinc/plutosdr-fw](https://github.com/analogdevicesinc/plutosdr-fw) — original PlutoSDR firmware
- [day0wl/libresdr-fw](https://github.com/day0wl/libresdr-fw) — LibreSDR (ZynqSDR) board-support patch for v0.37
- [hz12opensource/libresdr](https://github.com/hz12opensource/libresdr) — port of that patch to v0.38, source of the LVDS interface change
