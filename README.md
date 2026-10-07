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

## Build Instructions

```bash
sudo apt-get install git build-essential fakeroot libncurses5-dev libssl-dev ccache
sudo apt-get install dfu-util u-boot-tools device-tree-compiler mtools
sudo apt-get install bc python3 cpio zip unzip rsync file wget

git clone --recursive https://github.com/kushpet/ad_plutosdr-fw.git -b libresdr
cd ad_plutosdr-fw

export VIVADO_SETTINGS=/opt/Xilinx/Vivado/2022.2/settings64.sh

make
make sdimg
```

`TARGET` already defaults to `libre`. Since v0.38, the cross compiler (Linaro GCC 7.3 `arm-linux-gnueabihf`) is installed by buildroot under `buildroot/output/host`, so there is no need to set `CROSS_COMPILE` or `PATH`. `make` aborts if the Vivado found through `VIVADO_SETTINGS` is not 2022.2.

To build `libre-v0.37`, check out that tag (`git checkout libre-v0.37 && git submodule update`) and use Vivado/Vitis 2021.2 as described in that revision's README.

Optional CPU/DDR overclock (out of spec for the XC7Z020-1, at your own risk): after `make`, run `make overclock OVERCLOCK_CPU_MULT=<n> OVERCLOCK_DDR_MULT=<n>`, then `make sdimg`. This only patches PLL multipliers in the FSBL's `ps7_init.c`. The DDR timings stay at the 525 MHz preset.

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
- [day0wl/libresdr-fw](https://github.com/day0wl/libresdr-fw) — LibreSDR (ZynqSDR) board-support patch for v0.37
- [hz12opensource/libresdr](https://github.com/hz12opensource/libresdr) — port of that patch to v0.38, source of the LVDS interface change
