# CLAUDE.md

Firmware for **LibreSDR** (also sold as ZynqSDR), a Chinese clone of the ADI ADALM-PLUTO. It is a fork of ADI `plutosdr-fw` that adds a `libre` target. The user communicates in Russian.

## Repository layout

The superproject (`kushpet/ad_plutosdr-fw`) holds the Makefile, `scripts/libre.{mk,its}`, `overclock.sh` and the docs. Everything board-specific lives in four submodules. Each one is a personal fork whose working branch is also named `libresdr`:

| Submodule | Fork | LibreSDR-specific files |
| --- | --- | --- |
| `hdl` | `kushpet/ad_hdl` | `projects/libre/` (system_bd.tcl, system_constr.xdc, system_top.v) |
| `linux` | `kushpet/ad_linux` | `arch/arm/boot/dts/zynq-libre.dts{,i}`, `arch/arm/configs/zynq_libre_defconfig`, `drivers/mtd/spi-nor/core.c` (Winbond UID) |
| `buildroot` | `kushpet/ad_buildroot` | `board/libre/`, `configs/zynq_libre_defconfig` |
| `u-boot-xlnx` | `kushpet/ad_u-boot-xlnx` | `arch/arm/dts/zynq-libre-sdr.dts`, `configs/zynq_libre_defconfig`, `include/configs/zynq-common.h` |

The `upstream` remote in every repo points to `analogdevicesinc/*`. To change a submodule: commit on its `libresdr` branch, push it to the fork, then commit the new submodule pointer in the superproject. Never leave the superproject pointing at a commit that has not been pushed.

## History, branches and tags

Every repo uses the same layout:

```
libre-v0.37  (tag)  ADI v0.37 + day0wl LibreSDR patch + local fixes; tested on hardware
    │
libre-v0.38  (tag)  merge: parent 1 = libre-v0.37, parent 2 = ADI v0.38 commit.
    │               Functionally equal to libre-v0.37: CMOS, no overclock.
    │
libre-v0.38.1 (tag) LVDS, DDR 500 MHz, Realtek PHY driver, S22ethlink, LEDs fix
    │
libre-v0.38.2 (tag) + eth_mode (PHY powered down when unused). Flashed to the user's QSPI 2026-10-09.
libresdr tip        = libre-v0.38.2 (+ docs commits after it)
```

- ADI's own release tags `v0.37`/`v0.38` exist in the superproject; do not move them.
- `u-boot-xlnx` has no v0.38 commit, because ADI uses the same u-boot for both releases.
- `libre_v0.37` and `libre_v0.38` are the original import branches. They are frozen and kept for reference. `libre_v0.38` holds the unmodified hz12opensource port, including the overclock and kernel hacks that were rejected; see the `libre-v0.38` merge commit messages for why.

## Building

```bash
export VIVADO_SETTINGS=/home/user/Xilinx/Vitis/2022.2/settings64.sh   # Vitis settings: provides vivado AND xsct
export BR2_DL_DIR=/home/user/buildroot-dl                             # persistent buildroot download cache
make            # full firmware: build/libre.frm, libre.dfu, boot.frm, ...
make sdimg      # build_sdimg/: BOOT.bin, uImage, devicetree.dtb, uEnv.txt, uramdisk.image.gz
```

- v0.38 and later need **Vivado/Vitis 2022.2**. `libre-v0.37` needs 2021.2, which is also installed in `~/Xilinx`. The Makefile aborts on a version mismatch.
- The cross toolchain (Linaro GCC 7.3) is built and installed by buildroot. Do not set `CROSS_COMPILE` or `PATH`.
- `make` always runs `clean-build`, which **deletes `build/` and `build_sdimg/`**. Copy out any images you want to keep first.
- When switching between tags built with different Vivado versions, run `git clean -fdX` in `hdl/`. Otherwise stale IP builds (`library/*/*.xpr`, `ipcache/`) are reused.
- After checking out another superproject revision, always run `git submodule update`.
- `~/petalinux-cache` holds Yocto/PetaLinux downloads and sstate. It is **not** used by this build, which uses Vivado + buildroot, not PetaLinux.
- `make overclock OVERCLOCK_CPU_MULT=n OVERCLOCK_DDR_MULT=n` is opt-in and out of spec. Do not enable overclocking by default.
- GCC plugins are explicitly disabled in `zynq_libre_defconfig`. With the Linaro toolchain they would otherwise be auto-enabled, and the build would then need host `libmpc-dev`.
- The `bootgen` warning "fsbl.elf.0 range is overlapped with partition system_top.bit.0" is expected and harmless.
- Reference build (2026-10-07, 8 cores): tag `libre-v0.38` and the LVDS tip both built cleanly with Vivado 2022.2 and met timing.

## Hardware (see `docs/zynqsdr_rev5.pdf`; original Pluto: `docs/plutosdr_schematic_revd_0.1.pdf`)

- SoC **XC7Z020-1CLG400I** (speed grade −1: CPU max 667 MHz). 50 MHz PS_CLK.
  - The HDL project (inherited from day0wl) targets `xc7z020clg400-2`, so timing is analysed for −2, which is optimistic for this board.
  - With a 50 MHz PS_CLK, the requested 666.67 MHz becomes an actual **675 MHz** APU clock. This was the same in libre-v0.37, which runs stably.
  - The chip on the user's board is marked only `XC7Z020 CLG400ABX2209` and has no speed-grade line. The grade cannot be read over JTAG either, so treat it as −1, as the schematic does.
- RF: **AD9363**, 40 MHz reference (CLK-40M from a VCTCXO tuned by a DAC5311). Two RX and two TX on SMA. MMCX inputs for PPS and a 10 MHz external reference.
- RAM: 2× **MT41K256M16TW** (DDR3L 1.35 V, 32-bit bus, 1 GiB). HDL uses the MT41J256M16 RE-125 preset with **DDR at 500 MHz** (`PCW_UIPARAM_DDR_FREQ_MHZ 500`, PLL 1000/2).
  - The old default request was 533.33 MHz, which the 50 MHz crystal turns into 525 MHz. Its 3rd harmonic (1575.02 MHz) put a +36 dB spur 0.4 MHz from GPS L1. Do not go back to 525/533.
  - `PCW_UIPARAM_ACT_DDR_FREQ_MHZ` is only a display value; setting it changes nothing.
- QSPI: **W25Q256JV** (Winbond, 32 MiB). DT compatible `winbond,w25q256`. Partitions: fsbl+uboot 1M, uboot-env 128k, nvmfs, linux 30M.
- Ethernet: **RTL8211E-VB** RGMII PHY on MIO16–27 (the DT comment "Marvell 88e1512" is a leftover from Pluto), MDIO MIO52–53, PHY reset MIO46, default IP 192.168.1.10.
- USB0 OTG via USB3320 ULPI, reset MIO47; USB gadget with RNDIS 192.168.2.1. UART0 on MIO14–15 is the console, reached through an onboard FT2232HQ, which also provides JTAG (`/dev/ttyUSB2`, 115200).
- **AD9363 data bus is routed as LVDS pairs into bank 34 (VCCO 2.5 V)**: DATA_CLK N20/P20, FB_CLK N18/P19, RX_FRAME U18/U19, TX_FRAME Y16/Y17, RX_D0..5 and TX_D0..5 as in `projects/libre/system_constr.xdc`. Bank 35 is 3.3 V (LEDs, PL SPI).
- PL LEDs: `pl_led0` on J20 and `pl_led1` on H20 (LVCMOS33), driven by EMIO GPIO 17/18, which is Linux `gpio0` 71/72. `pl_led1` carries the heartbeat trigger. The `board/libre` scripts (update.sh, update_frm.sh, automounter.sh, udc_handle_suspend.sh) use `pl_led1:blue` for flash and status indication. The old day0wl `led0` on MIO15 was a mistake, because that pin is UART0.
- hw_serial: the kernel prints `SPI-NOR-UniqueID <hex>` for the Winbond flash (command 0x4B with 4 dummy bytes), and `board/libre/S23udc` parses it from dmesg.

## Rules that are easy to break

- **Hardware test status (2026-10-08, user's board):** both libre-v0.38 (CMOS) and the LVDS tip receive FM correctly in SDRangel on Windows.
  - LVDS passes the AD9363 RX BIST tone (`bist_tone "2 0 0 0"`; mode 2 = RX, mode 1 is TX) cleanly in 2R2T at 30.72 Msps.
  - **2R2T at 61.44 Msps (DATA_CLK 245.76 MHz, above the 125 MHz rx_clk constraint) killed the USB gadget, and the board needed a power cycle.** Do not test beyond the constraint without asking.
- **USB vs Ethernet: the core board problem, only worked around.** Facts from 2026-10-08/09:
  - Whenever the Ethernet PHY is powered (link up, or even with no cable), the host drops the board's USB gadget within about 10-60 s (`device descriptor read/64, error -71`). Board-side Linux sees nothing and keeps the UDC "configured".
  - With `ip link set eth0 down` (PHY powered down), USB stays up (2-3 min tests).
  - The PHY negotiating a link while the host enumerates also kills enumeration.
  - Likely hardware: RGMII (MIO16-27) and ULPI (MIO28-39) share MIO bank 501, and fast slew on RGMII TX dropped USB instantly. Unproven.
  - Workaround in v0.38.2: `board/libre/S22ethlink` + `eth_mode` (u-boot env, `config.txt` [USB_ETHERNET]: `auto` default = keep the PHY up only if a cable has carrier within 5 s at boot, otherwise power it down; `on`; `off`). S40network skips `auto eth0` when `/var/run/eth0.off` exists.
  - Result: USB-only use is stable. **USB and Ethernet together are not reliable.** Use one of them for data. While Ethernet is in use, expect the USB drive and config.txt to drop out.
  - One USB drop at boot was also seen with the PHY powered down (2026-10-09 13:45). The cause is unknown; a different USB cable has not been tried yet.
- **Ethernet (verified 2026-10-09):** 1000BASE-T to a PC on-board NIC (r8169) works.
  - iperf3 board->PC: 481 Mbit/s, 0 errors.
  - libiio, 1 channel: 10 Msps continuous without loss. The ceiling is about 12.5 Msps (~50 MB/s, board CPU / iiod bound). USB gives about 6 Msps.
  - The Realtek PHY driver is enabled; the RTL8211E binds to it.
  - **Do not use the user's AX88179 USB-Ethernet dongle for the SDR.** Under load it sends PAUSE frames, then gets stuck reporting every frame as `rx_errors` until replugged. Every "gigabit is broken" symptom seen on 2026-10-08/09 went through that dongle. The dongle has since been removed. The motherboard NIC carries the internet again (NM profile `internet-mb`); see the open items. The board's `ipaddr_eth` is still 192.168.10.10.
  - The schematic feeds RTL8211E CKXTAL1 from the 50 MHz PS oscillator (datasheet: 25 MHz), yet the link works. This is unexplained and has not been investigated further.
- After the board is power-cycled, the FT4232 console may re-enumerate under a different /dev/ttyUSBn. The console is FTDI interface 02: check `/sys/class/tty/ttyUSB*/device/../bInterfaceNumber`.
- **RX spurs near GPS L1** (measured 2026-10-09, 50 ohm on RX1, gain 71 dB, 1542-1602 MHz):
  - with DDR at 500 MHz, the remaining spurs are 1560.000 MHz (39×40 MHz, about +36 dB), 1600.000 MHz (40×40 MHz, about +39 dB) and 1500.02 MHz (3×DDR);
  - there is nothing in the L1 band;
  - Ethernet on/off/100M/1G and the USB gadget bound/unbound made no measurable difference.
  - `spurscan.py`-style method: 4 LOs at 20 Msps, ±7.5 MHz kept per LO, 64k FFT, 32 averages.
- SD boot (`sdboot`) neither runs `adi_loadvals` nor adds `uboot=` to bootargs. As a result, `config.txt` settings such as attr_val/mode are not applied to the DT, and info.html shows no u-boot version. This is stock ADI behaviour.
- **Only one CPU core when booting from QSPI:**
  - `qspiboot` builds bootargs with `maxcpus=${maxcpus}`, and ADI's default env (u-boot `include/configs/zynq-common.h`) has `maxcpus=1`.
  - `sdboot` sets no bootargs, so the kernel's CONFIG_CMDLINE applies and both cores run.
  - All throughput numbers above were measured on SD boot (2 cores). Libre-v0.37 on flash was single-core as well.
- **Flashing via the PlutoSDR drive from this PC:**
  - `gio mount -e` fails (no permission on /dev/sdX), and `udisksctl power-off` only powers the port off; neither triggers `update.sh`. Ejecting from the file manager should work.
  - Fallback used on 2026-10-09: copy the .frm file, `sync`, unmount, then on the board console run `echo "" > /sys/kernel/config/usb_gadget/composite_gadget/functions/mass_storage.0/lun.0/file`. That is what update.sh waits for.
  - Flash boot.frm before libre.frm.
  - Verify with `head -c <size> /dev/mtd0|mtd3 | md5sum` against the .frm minus its 33-byte md5 trailer (boot.frm also minus 1024 + 131072 bytes).
- **The CMOS/LVDS mode must match on both sides.** Both `hdl/projects/libre` (`CMOS_OR_LVDS_N`, IO standards, port names) and `linux/.../zynq-libre.dtsi` (`adi,lvds-mode-enable` vs `adi,full-port-enable`/`adi,swap-ports-enable`) have to agree. A mismatch boots, but the AD9363 interface tuning fails.
- `board/libre` is a copy of `board/pluto` with LibreSDR edits. After updating to a new ADI release, diff `board/pluto` old→new and port the changes. `update_from_github.sh` is intentionally absent, because it would fetch Pluto firmware.
- The old branch `libre_v0.38` has an `ADC_INIT_DELAY`, DDR timings and a 750 MHz APU setting. Do not copy them back in without a hardware test plan.
- Verify any pin change against `docs/zynqsdr_rev5.pdf`. `pdftotext -layout` extracts net names and ball numbers well enough to grep.
- Changes must be verified on the real board by the user. Say clearly what was only build-tested.

## Open items / handoff (as of 2026-10-09)

1. **Two CPU cores everywhere.** Set `maxcpus=2` for LibreSDR: either change the default in u-boot `zynq-common.h` (shared with Pluto, so guard it for libre), or add a `maxcpus` key to config.txt ([SYSTEM]) the same way `eth_mode` was done. Then rebuild, tag (`libre-v0.38.3`) and reflash boot.frm (the u-boot env lives in boot.frm). Re-measure the Ethernet streaming ceiling on flash boot.
2. **USB drop at boot with the PHY down**: try another USB cable or port first. If drops persist, consider binding the UDC late (after boot completes); re-binding by hand after boot has always been stable.
3. **SDR over Ethernet** needs a second gigabit NIC (the PC's only NIC carries the internet, the AX88179 dongle is unusable, and the user's switch is 100M). The NM profile `internet-dongle` exists with autoconnect off; `internet-mb` (enp3s0) is the internet now. The old `sdr-libresdr` profile was renamed to `internet-mb`.
4. README: add the libre-v0.38.1 / v0.38.2 tags to the branches/tags table, and document `eth_mode` and the USB/Ethernet limitation.
5. Possibly investigate the 1560 MHz spur (39×40 MHz). It rose from +24 to +36 dB after the DDR-500 HDL rebuild, presumably due to placement.
6. The user's next big plan: Ubuntu 22.04 + newer Vivado on another disk, then follow ADI v0.39/v0.40 (see project memory). The user prefers config via config.txt, concrete verified steps, and no speculative ideas.

