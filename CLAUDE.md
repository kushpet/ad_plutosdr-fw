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
libresdr tip        + LVDS switch (hdl + linux only), + docs
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
- RAM: 2× **MT41K256M16TW** (DDR3L 1.35 V, 32-bit bus, 1 GiB). HDL uses the MT41J256M16 RE-125 preset at 525 MHz.
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
- **USB vs Ethernet at boot:** if the gigabit PHY negotiated its link at the same moment the host enumerated the USB gadget, the host dropped the board (`device descriptor read/64, error -71`). `board/libre/S22ethlink` brings eth0 up and waits up to 5 s for carrier before `S23udc` binds the UDC. Verified on hardware 2026-10-08. Steady-state USB + 1G Ethernet works fine.
  - While debugging this, the user's AX88179 USB-Ethernet dongle got stuck after a board crash and reported every frame as `rx_errors`. Replugging the dongle fixed it, so check the PC side before blaming the board.
- **Ethernet (verified 2026-10-09):** 1000BASE-T to a PC on-board NIC (r8169) works.
  - iperf3 board->PC: 481 Mbit/s, 0 errors.
  - libiio, 1 channel: 10 Msps continuous without loss. The ceiling is about 12.5 Msps (~50 MB/s, board CPU / iiod bound). USB gives about 6 Msps.
  - The Realtek PHY driver is enabled; the RTL8211E binds to it.
  - **Do not use the user's AX88179 USB-Ethernet dongle for the SDR.** Under load it sends PAUSE frames, then gets stuck reporting every frame as `rx_errors` until replugged. Every "gigabit is broken" symptom seen on 2026-10-08/09 went through that dongle. The user now runs internet on the dongle and the SDR on the motherboard NIC: NM profile `sdr-libresdr` on enp3s0, 192.168.10.120/24, never-default; the board's `ipaddr_eth` is 192.168.10.10, set via config.txt.
  - **Board-side issue that remains:** heavy Ethernet traffic (even at 100M), or bringing the 1G link up at boot, can drop the board's USB gadget (host: `device descriptor read/64, error -71`). The kernel and Ethernet keep running. Setting fast slew on the RGMII TX MIO pins dropped USB immediately; the ULPI pins MIO28-39 share bank 501 with RGMII MIO16-27. Use USB for configuration (PlutoSDR drive, config.txt) and Ethernet for streaming; do not stream over both. Keep the USB gadget enabled, since config.txt needs it.
  - The schematic feeds RTL8211E CKXTAL1 from the 50 MHz PS oscillator (datasheet: 25 MHz), yet the link works. This is unexplained and has not been investigated further.
- After the board is power-cycled, the FT4232 console may re-enumerate under a different /dev/ttyUSBn. The console is FTDI interface 02: check `/sys/class/tty/ttyUSB*/device/../bInterfaceNumber`.
- SD boot (`sdboot`) neither runs `adi_loadvals` nor adds `uboot=` to bootargs. As a result, `config.txt` settings such as attr_val/mode are not applied to the DT, and info.html shows no u-boot version. This is stock ADI behaviour.
- **The CMOS/LVDS mode must match on both sides.** Both `hdl/projects/libre` (`CMOS_OR_LVDS_N`, IO standards, port names) and `linux/.../zynq-libre.dtsi` (`adi,lvds-mode-enable` vs `adi,full-port-enable`/`adi,swap-ports-enable`) have to agree. A mismatch boots, but the AD9363 interface tuning fails.
- `board/libre` is a copy of `board/pluto` with LibreSDR edits. After updating to a new ADI release, diff `board/pluto` old→new and port the changes. `update_from_github.sh` is intentionally absent, because it would fetch Pluto firmware.
- The old branch `libre_v0.38` has an `ADC_INIT_DELAY`, DDR timings and a 750 MHz APU setting. Do not copy them back in without a hardware test plan.
- Verify any pin change against `docs/zynqsdr_rev5.pdf`. `pdftotext -layout` extracts net names and ball numbers well enough to grep.
- Changes must be verified on the real board by the user. Say clearly what was only build-tested.
