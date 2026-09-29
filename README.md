# plutosdr-fw (LibreSDR fork)

This is a personal fork of ADI's [plutosdr-fw](https://github.com/analogdevicesinc/plutosdr-fw), with the [LibreSDR (ZynqSDR)](https://github.com/day0wl/libresdr-fw) board-support patch from [day0wl](https://github.com/day0wl) already applied and committed (branch `libre_v0.37`, based on the upstream v0.37 release). Unlike day0wl's original repo, there is no `apply.sh` step here — clone this fork recursively and it already builds the `libre` target.

Confirmed working: built with Vivado/Vitis **2021.2** and tested on real LibreSDR hardware.

Original PlutoSDR wiki instructions (for reference, upstream/Pluto only): [Building the image](https://wiki.analog.com/university/tools/pluto/building_the_image)

## Build Instructions

```bash
sudo apt-get install git build-essential fakeroot libncurses5-dev libssl-dev ccache
sudo apt-get install dfu-util u-boot-tools device-tree-compiler libssl1.0-dev mtools
sudo apt-get install bc python cpio zip unzip rsync file wget

git clone --recursive https://github.com/kushpet/ad_plutosdr-fw.git -b libre_v0.37
cd ad_plutosdr-fw

export CROSS_COMPILE=arm-linux-gnueabihf-
export PATH=$PATH:/opt/Xilinx/Vitis/2021.2/gnu/aarch32/lin/gcc-arm-linux-gnueabi/bin
export VIVADO_SETTINGS=/opt/Xilinx/Vivado/2021.2/settings64.sh

make
make sdimg
```

`TARGET` already defaults to `libre` in this fork's `Makefile`, so there is no need to set it manually. As with the upstream day0wl patch, **only Vivado/Vitis 2021.2 is supported** — newer Vivado versions are known not to work because of HDL design dependencies, so don't try to "upgrade" the toolchain here.

If you need to update submodules to their pinned commits later:
```bash
git pull
git submodule update --init --recursive
```

## First boot / deployment (SD card)

LibreSDR boards typically ship with empty flash, so the first boot has to happen from an SD card:

1. Format a small SD card as **FAT32**.
2. Copy everything from `build_sdimg/` (produced by `make sdimg`) onto the card: `BOOT.bin`, `uImage`, `devicetree.dtb`, `uEnv.txt`, `uramdisk.image.gz`.
3. Insert the card and power on the board.
4. Once it's running from the SD card, it can flash itself the same way a stock PlutoSDR does over USB mass storage/DFU, and will eventually boot from onboard flash without the SD card.

**Connectivity on first boot:**
- Gigabit Ethernet, default IP `192.168.1.10`
- Serial debug console: `/dev/ttyUSB2`, 115200 8N1

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
- [day0wl/libresdr-fw](https://github.com/day0wl/libresdr-fw) — LibreSDR (ZynqSDR) board-support patch this branch is based on
