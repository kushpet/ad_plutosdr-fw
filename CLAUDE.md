# CLAUDE.md

Firmware for **LibreSDR** (also sold as ZynqSDR), a Chinese clone of the ADI ADALM-PLUTO. It is a fork of ADI `plutosdr-fw` that adds a `libre` target. The user communicates in Russian.

## Repository layout

The superproject (`kushpet/ad_plutosdr-fw`) holds the Makefile, `scripts/libre.{mk,its}` and the docs. Everything board-specific lives in four submodules. Each one is a personal fork whose working branch is also named `libresdr`:

| Submodule | Fork | LibreSDR-specific files |
| --- | --- | --- |
| `hdl` | `kushpet/ad_hdl` | `projects/libre/` (system_bd.tcl, system_constr.xdc, system_top.v) |
| `linux` | `kushpet/ad_linux` | `arch/arm/boot/dts/zynq-libre.dts{,i}`, `arch/arm/configs/zynq_libre_defconfig`, `drivers/mtd/spi-nor/core.c` (Winbond EAR + UID) |
| `buildroot` | `kushpet/ad_buildroot` | `board/libre/`, `configs/zynq_libre_defconfig` |
| `u-boot-xlnx` | `kushpet/ad_u-boot-xlnx` | `arch/arm/dts/zynq-libre-sdr.dts`, `configs/zynq_libre_defconfig`, `include/configs/zynq-common.h` |

The `upstream` remote in every repo points to `analogdevicesinc/*`. To change a submodule: commit on its `libresdr` branch, push it to the fork, then commit the new submodule pointer in the superproject. Never leave the superproject pointing at a commit that has not been pushed. Push only to the user's forks (`origin`); nothing is ever sent to ADI.

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
libre-v0.38.2 (tag) + eth_mode (PHY powered down when unused).
    │
libre-v0.38.3 (tag) + Winbond EAR flash fix (kernel), DFU button on MIO12, u-boot
    │               console on UART0, maxcpus=2 default + config.txt key.
    │               Built with Vivado 2022.2 (last tag that needs it).
    │
libre-v0.39  (tag)  merge: parent 1 = libre-v0.38.3 line, parent 2 = ADI v0.39 commit
    │               (Linux 6.1). Built with Vivado 2025.2. + UART1 disabled / serial0 =
    │               UART0 in the DT, info.html compares the ADI base version.
    │               Flashed to the user's QSPI 2026-10-11 (md5 verified).
    │
libre-v0.39.1 (tag) eth_mode/S22ethlink removed (USB drops were PC-power related),
    │               VCTCXO DAC5311 to mid-scale at boot, HDL for speed grade −1, PL I2C +
    │               EEPROM, day0wl leftovers removed (gateway, kernel options, overclock,
    │               u-boot BOOTDELAY/gem0/gpio-keys), ADI S23udc/device_persistent_keys.
libresdr tip        = libre-v0.39.1 (+ docs commits after it)
```

- ADI's own release tags `v0.37`/`v0.38`/`v0.39` exist in the superproject; do not move them.
- `u-boot-xlnx` has no v0.38/v0.39 commit, because ADI uses the same u-boot for v0.37–v0.39.
- **ADI cuts releases from release branches.** In `hdl` and `linux`, the ADI v0.38 commit is not an ancestor of the v0.39 commit, so `git merge` conflicts in dozens of files LibreSDR never touched. The v0.39 merges were resolved as "ADI v0.39 tree + LibreSDR files" (`git checkout <adi> -- .`, drop files ADI removed, restore ours); the merge messages list what was carried over. Expect the same for v0.40. buildroot and the superproject merged cleanly.
- `libre_v0.37` and `libre_v0.38` are the original import branches. They are frozen and kept for reference. `libre_v0.38` holds the unmodified hz12opensource port, including the overclock and kernel hacks that were rejected; see the `libre-v0.38` merge commit messages for why.

## Building

Since 2026-10-11 the work happens on the user's Ubuntu 22.04 system. The repository stays on the old Ubuntu 20.04 disk, permanently mounted at `/mnt/ubuntu-20.04`:

```bash
cd /mnt/ubuntu-20.04/home/user/GitHub/kushpet/ad_plutosdr-fw
export VIVADO_SETTINGS=/home/user/Xilinx/2025.2/Vitis/settings64.sh   # provides vivado, xsct, bootgen
export BR2_DL_DIR=/mnt/ubuntu-20.04/home/user/buildroot-dl            # persistent buildroot download cache
make            # full firmware: build/libre.frm, libre.dfu, boot.frm, ...
make sdimg      # build_sdimg/: BOOT.bin, uImage, devicetree.dtb, uEnv.txt, uramdisk.image.gz
```

- `libre-v0.39` and later need **Vivado/Vitis 2025.2** (ADI v0.39 itself targets 2023.2). The Makefile sets `VIVADO_VERSION ?= 2025.2` and passes it to the HDL build as `REQUIRED_VIVADO_VERSION`, which `hdl/scripts/adi_env.tcl` honours, so no ADI file is patched and `ADI_IGNORE_VERSION_CHECK` is not needed. For a standalone `make -C hdl/projects/libre`, export `REQUIRED_VIVADO_VERSION=2025.2`.
- `libre-v0.38.x` tags need Vivado 2022.2, which exists only on the old system. Its build trees were cleaned on 2026-10-11, and the v0.38.3 images are archived in `/mnt/ubuntu-20.04/home/user/libresdr-images/`. The v0.39 images (`libre-v0.39/`) are archived there too.
- The cross toolchain (Linaro GCC 7.3) is installed by buildroot. Do not set `CROSS_COMPILE` or `PATH`.
- `make` always runs `clean-build`, which **deletes `build/` and `build_sdimg/`**. Copy out any images you want to keep first.
- Version strings come from `git describe`. Build from a clean tree at the tag (no `-dirty`). Before the tag exists, the firmware reports e.g. `libre-v0.38.3-10-gdb23`.
- When switching between tags built with different Vivado versions, run `git clean -fdX` in `hdl/`. Otherwise stale IP builds (`library/*/*.xpr`, `ipcache/`) are reused. To clean the kernel, use `make -C linux ARCH=arm mrproper` (without `ARCH=arm` it leaves `arch/arm` artifacts).
- After checking out another superproject revision, always run `git submodule update`.
- Timings on 8 cores with 2025.2: HDL from scratch about 15 min (all IP), a full `make` with the HDL and buildroot already built 3–10 min.
- `overclock.sh`/`make overclock` (day0wl, out of spec) were removed after v0.39. Do not overclock.
- GCC plugins are explicitly disabled in `zynq_libre_defconfig`. With the Linaro toolchain they would otherwise be auto-enabled, and the build would then need host `libmpc-dev`.
- Expected, harmless messages: `bootgen` "fsbl.elf.0 range is overlapped with partition system_top.bit.0"; XSCT 2025.2 "XSCT is deprecated and will be removed in future releases" (it still builds the FSBL; ADI master uses it with 2025.1 too); the dtc `unit_address_vs_reg` warnings for `scripts/libre.its`.
- HDL with 2025.2 (2026-10-11): `libre-v0.38.3` and the v0.39 merge both build without critical warnings and meet timing (WNS about 1.0 ns, analysed for −2). The `xlconcat`/`xlslice`/`util_vector_logic` IPs only get a "migrate to inline HDL" hint; Xilinx announced their removal for the 2026 releases (port as ADI did in hdl `eadd7d695` when needed). Pre-synthesis "No pins/cells matched" warnings for `system_constr.xdc` and `adi_fir_filter_constr.xdc` are from reading XDC while the IPs are black boxes; implementation applies them (checked `ASYNC_REG` on the routed design). These constraints are identical to ADI pluto.

## Hardware (see `docs/zynqsdr_rev5.pdf`; original Pluto: `docs/plutosdr_schematic_revd_0.1.pdf`)

- SoC **XC7Z020-1CLG400I** (speed grade −1: CPU max 667 MHz). 50 MHz PS_CLK.
  - Since libre-v0.39.1 the HDL targets `xc7z020clg400-1` (day0wl had −2). With 2025.2 it meets timing for −1: WNS 0.832 ns, WHS 0.015 ns, no critical warnings.
  - With a 50 MHz PS_CLK, the requested 666.67 MHz becomes an actual **675 MHz** APU clock. This was the same in libre-v0.37, which runs stably.
  - The chip on the user's board is marked only `XC7Z020 CLG400ABX2209` and has no speed-grade line. The grade cannot be read over JTAG either, so treat it as −1, as the schematic does.
- RF: **AD9363**, 40 MHz reference (CLK-40M from VCTCXO Y2). Two RX and two TX on SMA.
  - Y2's control voltage `VCTCXO_VFC` comes from a **DAC5311** (U12, 8 bit, VCC/Vref = +3.3V_CLK, 49.9 Ω in series). Its pins CLK_DAC_CS#/SCLK/SDI are PL H18/F19/F20. Until v0.39 they were not in the HDL, so the DAC sat at its power-on 0 V. Since v0.39.1 they are EMIO GPIO 19/20/21 (Linux gpio0 73/74/75), driven by `spi-gpio` + `ti-dac7311` (IIO device `dac5311`, attribute `out_voltage_raw`, no index); the Zynq SPI cannot send the 16-bit mode-1 words. `board/libre/S22vctcxo` writes 128 at boot.
  - Measured 2026-10-18 on the user's board against GSM FCCH (BTS ±0.05 ppm), room temperature: reference error −20.7 ppm at code 0, −2.2 ppm at 128, +13.3 ppm at 255 (0.14 ppm/LSB below mid, 0.12 above). DAC 128 + `xo_correction=39999912` gave 0.000 ± 0.003 ppm. Broadcast FM carriers (104.7, 90.3 MHz) read about 18 ppm off the GSM result, so FM is useless as an absolute reference here; relative DAC steps agreed.
  - The 40 MHz also reaches the PL on H16 (CLK-40M-FPGA, through an SN74AHC1G14).
  - MMCX inputs, both into the PL through ESD diodes, BAT54 clamps and SN74AHC1G14 Schmitt buffers: **PPS_IN** on K18 (DC-coupled via 1k), **CLKIN_10MHz** on K17 (49.9 Ω termination, AC-coupled 2×1000 pF, biased). Neither is used by the HDL yet. The user has no PPS source; a HackRF One with a better oscillator can provide 10 MHz on its CLKOUT (`hackrf_clock -o 1`).
- PL I2C `axi_iic_main` (0x41600000, as Pluto) on EXP_SCL/EXP_SDA = M15/K16: AT24C02D EEPROM U32 at 0x50 (blank, all 0xFF; DT read-only since v0.39.1). The AD7291 rail monitor U29 from the schematic does not answer at any address on the user's board (probably not fitted), so it is not in the DT. The xiic controller acknowledges writes to absent addresses (i2cdetect quick-write ranges show every address); use `i2cdetect -r`.
- RAM: 2× **MT41K256M16TW** (DDR3L 1.35 V, 32-bit bus, 1 GiB). HDL uses the MT41J256M16 RE-125 preset with **DDR at 500 MHz** (`PCW_UIPARAM_DDR_FREQ_MHZ 500`, PLL 1000/2).
  - The old default request was 533.33 MHz, which the 50 MHz crystal turns into 525 MHz. Its 3rd harmonic (1575.02 MHz) put a +36 dB spur 0.4 MHz from GPS L1. Do not go back to 525/533.
  - `PCW_UIPARAM_ACT_DDR_FREQ_MHZ` is only a display value; setting it changes nothing.
- QSPI: **W25Q256JV** (Winbond, 32 MiB). DT compatible `winbond,w25q256`. Partitions: fsbl+uboot 1M, uboot-env 128k, nvmfs, linux 30M.
  - Zynq QSPI uses 3-byte addresses; the upper 16 MiB is selected by the flash's Extended Address Register (EAR). EAR survives a soft reset (only a power cycle clears it), and the BootROM reads with whatever EAR is set. After reading above 16 MiB in u-boot, read an address below 16 MiB before `reset`.
  - The Xilinx spi-nor code (linux `drivers/mtd/spi-nor/core.c`, `read_ear`/`spi_nor_write_ear`) does not know Winbond: dmesg `failed to read ear reg`, Linux assumed bank 0 and could not change EAR. When u-boot left EAR=1, every Linux mtd access went to +16 MiB: update.sh wrote .frm files to the upper half, md5 checks via /dev/mtd* passed (same wrong bank), the old image kept booting, and soft reboots ran a stale Chinese u-boot (Oct 2023) that sat at 0x1000000. Fixed in libre-v0.38.3 (Winbond: RDEAR 0xC8, WREN + WREAR 0xC5); `spi_nor_shutdown` really clears EAR before reboot. Same bug: hz12opensource/libresdr#28. Xilinx 6.1 (v0.39) still lacks Winbond; the patch was re-applied on 6.1, where the check is `nor->info->id[0]` (verified 2026-10-11: Linux and u-boot read identical bytes at 0x200000, 0x1200000 and 0x1E00000).
  - The stale upper-half copies (Chinese boot image at 0x1000000, env at 0x1100000, an image at 0x1200000) and the tail of mtd0 after boot.bin were **erased on 2026-10-11** (`flash_erase /dev/mtd3 0xDB0000 0`, `flash_erase /dev/mtd0 0x90000 0`). nvmfs was left alone.
  - To check what is physically in flash independently of Linux, use u-boot: `sf probe 0:0 50000000 0; sf read 0x3000000 <addr> 0x100; md.b 0x3000000 0x30`.
- DFU button **SW1 `DFU_KEY` is on MIO12** (10k pull-up to 3.3 V, pressed = 0). Pluto code used MIO14, which on LibreSDR is UART0 RX: u-boot sometimes saw a false "Button pressed", loaded its default env (ignoring config.txt) and the button never worked; Linux gpio-keys fired BTN_0 (automounter remove_all) on console input. Fixed in libre-v0.38.3 in u-boot (misc_init_r, qspiboot) and the Linux DT. u-boot messages go to UART0 (serial@e0000000) instead of Pluto's UART1.
- UARTs: only **UART0** (MIO14–15) is routed and enabled in the PS7 config. `zynq.dtsi` (shared with Pluto) enables UART1 and sets `serial0 = &uart1`; on 6.1 that made UART1 fail with `xuartps e0001000.serial: uart_add_one_port() failed; err=-22`. Since v0.39 `zynq-libre.dtsi` sets `serial0 = &uart0` and disables UART1.
- Ethernet: **RTL8211E-VB** RGMII PHY on MIO16–27, MDIO MIO52–53, PHY reset MIO46, default IP 192.168.1.10 (static; Pluto defaults to DHCP). CKXTAL1 comes from the 50 MHz oscillator Y1 (net `ETH_CLK`). Linux uses a random MAC on every boot (`invalid hw address, using random`); u-boot never passed one.
- USB0 OTG via USB3320 ULPI, reset MIO47, REFCLK from a 12 MHz oscillator (Y3) shared with the FT4232. USB gadget with RNDIS 192.168.2.1. UART0 on MIO14–15 is the console, reached through an onboard FTDI (enumerates as FT4232H), which also provides JTAG (`/dev/ttyUSB2`, 115200).
  - Linux 6.1 (v0.39): `CONFIG_USB_ULPI` now depends on `USB_PHY`, which nothing selects, so ADI dropped it. ChipIdea then neither creates the ULPI PHY nor pulses `xlnx,phy-reset-gpio` (no "Found SMSC USB3320 ULPI transceiver" in dmesg); the USB3320 runs as u-boot set it up. This is how ADI Pluto v0.39 works, and the gadget works the same as on 5.15. `CONFIG_NOP_USB_XCEIV=y` would bring `USB_PHY` back, but then the new Xilinx `ulpi-phy` platform driver binds, not the 5.15 code path.
  - MIO16–39 (RGMII and ULPI) are LVCMOS25, slow slew, pull-up (SLCR `MIO_PIN_nn` = 0x14xx), the same slew as ADI Pluto.
- **AD9363 data bus is routed as LVDS pairs into bank 34 (VCCO 2.5 V)**: DATA_CLK N20/P20, FB_CLK N18/P19, RX_FRAME U18/U19, TX_FRAME Y16/Y17, RX_D0..5 and TX_D0..5 as in `projects/libre/system_constr.xdc`. Bank 35 is 3.3 V (LEDs, PL SPI).
- PL LEDs: `pl_led0` on J20 and `pl_led1` on H20 (LVCMOS33), driven by EMIO GPIO 17/18, which is Linux `gpio0` 71/72. `pl_led1` carries the heartbeat trigger. The `board/libre` scripts (update.sh, update_frm.sh, automounter.sh, udc_handle_suspend.sh) use `pl_led1:blue` for flash and status indication. The old day0wl `led0` on MIO15 was a mistake, because that pin is UART0.
- hw_serial: the kernel prints `SPI-NOR-UniqueID <hex>` for the Winbond flash (command 0x4B with 4 dummy bytes), and `board/libre/S23udc` parses it from dmesg.

## Rules that are easy to break

- **Hardware test status:**
  - 2026-10-08: libre-v0.38 (CMOS) and the LVDS tip receive FM in SDRangel on Windows.
  - 2026-10-11, **libre-v0.39** (RAM DFU first, then flashed): Linux 6.1 on 2 cores; no `failed to read ear reg`, UniqueID printed; AD9363 probes and the LVDS tuning passes; RX BIST tone (`bist_tone "2 0 0 0"` in `/sys/kernel/debug/iio/iio:device0/`; mode 2 = RX, mode 1 is TX) clean in 2R2T at 30.72 Msps (262144 samples, no bad sample, spurs ≤ −80 dBc); FM stations visible in SDRangel 7.27.2 on Ubuntu 22.04 (weak signal, indoor behind window bars); `eth_mode=auto` powered the PHY down without a cable; Button IRQ stays 0 on console input; heartbeat on pl_led1; DFU button at power-up enters DFU (`0456:b674`). Ethernet data was not tested (no second NIC yet).
  - 2026-10-18, **libre-v0.39.1 candidate in RAM (DFU)**, charger in Debug, OTG in the PC: eth0 up at boot with a cable (1G, no USB trouble), boot without a cable then plug in (link and address come up), unplug/replug at runtime; UDC "configured" throughout; `dac5311` probed and set to 128 by S22vctcxo; EEPROM read. GSM/FM measurements as under Hardware. Measurement scripts (`gsmfcch.py`: GSM FCCH reference error; `fmoffset.py`: FM carrier offset) are kept in `/mnt/ubuntu-20.04/home/user/libresdr-images/tools/`.
  - 2026-10-18, **libre-v0.39.1 flashed** (boot.frm then libre.frm via the drive, triggered over ssh; md5 of mtd0/mtd3 matched; env restored: ipaddr_eth, netmask_eth, hostname). Flash boot: all components report libre-v0.39.1, u-boot BOOTDELAY 0 (reboot to Ethernet up about 15 s), no `failed to read ear reg`, AD9363 probes, DAC 128, reference −2.21 ppm vs GSM.
  - 2026-10-18, **1 A charger only** (Debug port, OTG unplugged), 10.5 min: `mode=2r2t`, both TX on at −89.75 dB with DDS tones, all 4 RX channels streamed over Ethernet at 6 Msps = 48.0 MB/s average without loss, CPU 100% (iiod + busy loop). No reboot, no Ethernet drop; VCCINT 0.966–0.978 V, VCCAUX 1.775–1.787 V, Zynq 52–54 °C. `mode` set back to 1r1t afterwards. Unplugging the OTG while running did not reset the board.
  - **2R2T at 61.44 Msps (DATA_CLK 245.76 MHz, above the 125 MHz rx_clk constraint) killed the USB gadget, and the board needed a power cycle.** Do not test beyond the constraint without asking.
- **USB drops come from powering the board from the PC's USB, not from Ethernet as such** (tests 2026-10-18; both USB-C inputs, Debug/FT4232 and OTG, are diode-ORed into +5V, there is no other power input):
  - Board powered from two PC USB-3 ports, OTG in rear USB-3 port 3-1, PHY up with a 1G link, PC idle: the host dropped the gadget after 5m40s, and again 3.5 min after a UDC re-bind.
  - Samsung 5 V **1 A** charger in the Debug USB-C, OTG still in the PC: no drop in 63 min with the PHY up at 1G (30 min idle, 15 min USB 18 MB/s + Ethernet IQ 25 MB/s, 15 min more with all 8 PC cores busy); 10/10 "bind the UDC and `ifup eth0` at the same moment" cycles enumerated; 15 min with the PHY up and no cable were clean.
  - So the old findings (2026-10-08/09: drops within 10-60 s with the PHY powered, enumeration killed by link negotiation; 2026-10-11: drops depend on host port and PC load) were all with PC-only power. The PHY's extra current is what tips it. Whether VBUS droop or ground offset in the USB cable is the mechanism was not separated.
  - `eth_mode`/`S22ethlink` (v0.38.2–v0.39) were removed again after v0.39: eth0 comes up from S40network as on Pluto, and a cable may be plugged in after boot. **Practice: power the board from a 5 V supply (Debug USB-C) whenever Ethernet is used; with PC-only power, USB drops while the PHY is powered** (`ip link set eth0 down` by hand avoids it until reboot).
  - Pattern of every drop: the host resets the device after a transfer error (`unregister rndis_host`, `reset high-speed USB device`, `-71`), then gives up (port `not attached`). The board never sees the bus reset: PORTSC1 (0xE0002184) stays 0x88000A05 (line J), ULPI viewport 0xE0002170 reads 0x08000000 (ULPISS=1, no transaction pending), USBSTS 0x80, FRINDEX and the USB IRQ count freeze, UDC "configured". Reading 0xE0002170 is safe; starting a viewport transaction is not (a stuck read made the next re-bind fail). Re-binding the UDC (`echo "" > .../UDC; sleep 3; echo ci_hdrc.0 > .../UDC`) restores it.
  - GPIO `DATA_RO` (0xE000A060/64) does not show MIO pins owned by other peripherals (UART0 RX and ULPI CLK read a constant 0), so it cannot probe the ULPI lines.
  - Host side is not involved: autosuspend is off (`power/control=on`), bcdUSB 2.00 (no LPM). The FT4232 console on the same xHCI never dropped.
- **Ethernet (verified 2026-10-09):** 1000BASE-T to a PC on-board NIC (r8169) works.
  - iperf3 board->PC: 481 Mbit/s, 0 errors.
  - libiio, 1 channel: 10 Msps continuous without loss. The ceiling is about 12.5 Msps (~50 MB/s, board CPU / iiod bound). USB gives about 6 Msps.
  - The Realtek PHY driver is enabled; the RTL8211E binds to it.
  - **Do not use the user's AX88179 USB-Ethernet dongle for the SDR.** Under load it sends PAUSE frames, then gets stuck reporting every frame as `rx_errors` until replugged.
  - The schematic feeds RTL8211E CKXTAL1 from the 50 MHz oscillator Y1 (datasheet: 25 MHz), yet the link works. This is unexplained and has not been investigated further.
- After the board is power-cycled, the FT4232 console may re-enumerate under a different /dev/ttyUSBn. The console is FTDI interface 02: check `/sys/class/tty/ttyUSB*/device/../bInterfaceNumber`.
- **Driving the board console from scripts:** the console may sit at `login:`, `Password:` or a half-typed line, and a command sent blindly becomes a user name. Send Ctrl-C, check for the prompt, log in (root/analog) and confirm with an echo marker before sending anything like `device_reboot`. Long outputs garble on the console; write them to `/www/<file>` on the board and fetch `http://192.168.2.1/<file>` (busybox httpd). The host has no `curl`; use python `urllib` or `wget`.
- **Without a console** (charger in the Debug port), use ssh with the default password through `SSH_ASKPASS` (`libresdr-images/tools/bsh.sh`; `BOARD=192.168.2.1` for USB, default 192.168.10.10 over Ethernet). Do not install ssh keys on the board (the user's auto-mode rules treat that as persistence).
- **RX spurs near GPS L1** (measured 2026-10-09, 50 ohm on RX1, gain 71 dB, 1542-1602 MHz):
  - with DDR at 500 MHz, the remaining spurs are 1560.000 MHz (39×40 MHz, about +36 dB), 1600.000 MHz (40×40 MHz, about +39 dB) and 1500.02 MHz (3×DDR);
  - there is nothing in the L1 band;
  - Ethernet on/off/100M/1G and the USB gadget bound/unbound made no measurable difference.
  - `spurscan.py`-style method: 4 LOs at 20 Msps, ±7.5 MHz kept per LO, 64k FFT, 32 averages.
- SD boot (`sdboot`) neither runs `adi_loadvals` nor adds `uboot=` to bootargs. As a result, `config.txt` settings such as attr_val/mode are not applied to the DT, and info.html shows no u-boot version. This is stock ADI behaviour.
- **CPU cores:** `qspiboot` builds bootargs with `maxcpus=${maxcpus}`. Since libre-v0.38.3 the u-boot default is `maxcpus=2`, and config.txt [SYSTEM] has a `maxcpus` key (1|2; update.sh ignores other values). `sdboot` sets no bootargs, so CONFIG_CMDLINE applies and both cores run.
- **Flashing via the PlutoSDR drive from this PC:**
  - `gio mount -e` fails (no permission on /dev/sdX), and `udisksctl power-off` only powers the port off; neither triggers `update.sh`. Ejecting from the file manager should work.
  - Method used on 2026-10-11: copy the .frm file, `sync`, `udisksctl unmount -b /dev/sdX1`, then on the board console run `echo "" > /sys/kernel/config/usb_gadget/composite_gadget/functions/mass_storage.0/lun.0/file`. That is what update.sh waits for.
  - **Flash boot.frm and libre.frm in two separate ejects, boot.frm first.** In one eject, update.sh writes libre.frm first (setting `fit_size`) and boot.frm second, whose env overwrites `fit_size` (u-boot then falls back to a 30 MiB read).
  - Flashing boot.frm resets the whole u-boot env. Save `fw_printenv` first and restore the non-defaults afterwards. On the user's board these are `ipaddr_eth=192.168.10.10`, `netmask_eth=255.255.255.0`, `hostname=libre` (`eth_mode` is obsolete since the workaround was removed; the copy from before the v0.39 flash is in `libresdr-images/libre-v0.39-test2/uboot-env-before-flash.txt`).
  - Verify with `head -c <size> /dev/mtd0|mtd3 | md5sum` against the .frm minus its 33-byte md5 trailer (boot.frm also minus 1024 + 131072 bytes). This is only trustworthy on a kernel with the Winbond EAR fix (no `failed to read ear reg` in dmesg).
  - DFU: `device_reboot ram` (image to RAM only, no flash write: `dfu-util -D build/libre.dfu -a firmware.dfu; dfu-util -a firmware.dfu -e`) or `device_reboot sf` (flash). Useful to test a kernel without touching flash.
- **The CMOS/LVDS mode must match on both sides.** Both `hdl/projects/libre` (`CMOS_OR_LVDS_N`, IO standards, port names) and `linux/.../zynq-libre.dtsi` (`adi,lvds-mode-enable` vs `adi,full-port-enable`/`adi,swap-ports-enable`) have to agree. A mismatch boots, but the AD9363 interface tuning fails.
- `board/libre` is a copy of `board/pluto` with LibreSDR edits. After updating to a new ADI release, diff `board/pluto` old→new and port the changes (v0.38→v0.39: no changes). `update_from_github.sh` is intentionally absent, because it would fetch Pluto firmware. `msd/img/version.js` (info.html) compares the ADI base version (`libre-vX.Y…` → `vX.Y`) with ADI's latest plutosdr-fw release and links to ADI's release page, never to the Pluto firmware binary.
- `zynq_libre_defconfig` = ADI's `zynq_pluto_defconfig` of the same release + the LibreSDR deltas (since v0.39.1: SD card, MACB + REALTEK_PHY, EEPROM_AT24, SPI_GPIO + TI_DAC7311, GCC plugins off, DEBUG_ZYNQ_UART0). Rebuild it the same way for the next release. `diff zynq_pluto_defconfig zynq_libre_defconfig` must show only these.
- **buildroot reuses `output/target`.** A file that `post-build.sh` stops installing stays in the rootfs (after the S22ethlink removal it was still in `/etc/init.d`). After removing a board file, delete it from `buildroot/output/target` too (or clean buildroot) and check `ls buildroot/output/target/etc/init.d`.
- Audit 2026-10-18 (v0.39 vs ADI v0.39): hdl only adds `projects/libre`; linux touches only the libre DT/defconfig and `spi-nor/core.c`; u-boot touches `zynq-common.h` and `board.c`. Kept on purpose: AD9361 compatible `adi,ad9361` (70 MHz–6 GHz, ADI uses `ad9363a`), maxcpus=2, static Ethernet default, SD-card boot, ethtool/iperf3/htop and busybox taskset, branding, `ADC_INIT_DELAY 30` (tested). Not ported: ADI's axi_tdd/phaser block (Pluto-only connector). Left alone: ADI's `spi-nor-locking-disable` (day0wl dropped it; restoring it changes how boot.frm is written over u-boot's `sf protect lock`, needs a flash test).
- The old branch `libre_v0.38` has an `ADC_INIT_DELAY`, DDR timings and a 750 MHz APU setting. Do not copy them back in without a hardware test plan.
- Verify any pin change against `docs/zynqsdr_rev5.pdf`. `pdftotext -layout` extracts net names and ball numbers well enough to grep.
- Changes must be verified on the real board by the user. Say clearly what was only build-tested.

## Host (Ubuntu 22.04, set up 2026-10-11)

- User `user`, UID 1000 (same as the old system, so no git `safe.directory`), in `dialout` and `plugdev`. SSH key `~/.ssh/id_ed25519` is the same as on the old disk and authenticates to GitHub as `kushpet`.
- `/etc/udev/rules.d/53-adi-plutosdr-usb.rules` (copied from the old disk) gives access to `0456:b673/b674` and sets `ID_MM_DEVICE_IGNORE` for ModemManager.
- SDRangel 7.27.2 from the official `sdrangel_7.27.2_ubuntu-22.04_amd64.deb`; it needs `libuhd4.10.0` from `ppa:ettusresearch/uhd`. `libiio-utils` 0.23 is installed (`iio_info -s` lists the board over USB and `ip:192.168.2.1`). SDRangel's REST API is on `http://127.0.0.1:8091/sdrangel`.
- The internet NIC is `enp3s0` (onboard RTL8168g, NM profile `Wired connection 1`), cabled to a 100 Mbit/s switch, so its 100 Mbit/s link is expected. `amn0` is the user's VPN.
- **The board's Ethernet has its own NIC (permanent setup since 2026-10-18):** `enp4s0`, a PCIe RTL8125B 2.5GbE (PCI 04:00.0, in-kernel `r8169`, firmware `rtl8125b-2.fw`), cabled directly to the LibreSDR. NM profile `LibreSDR`: static `192.168.10.1/24`, no gateway (`never-default`), IPv6 off, autoconnect. The board side is `ipaddr_eth=192.168.10.10`. The VPN's `0.0.0.0/1` routes do not catch 192.168.10.0/24 (the connected route is more specific). Without carrier NM shows `enp4s0` as `unavailable`. `ethtool` is not installed.

## Open items / handoff (as of 2026-10-18)

1. **Done: libre-v0.39.1**, tagged and pushed in all five repos, flashed and verified on hardware (see Rules → Hardware test status); images in `/mnt/ubuntu-20.04/home/user/libresdr-images/libre-v0.39.1/`.
2. **USB drops**: explained by PC USB power (see Rules). Practice: 5 V charger in the Debug port whenever Ethernet is used.
3. **Next step: discipline the VCTCXO to an external 10 MHz** (HackRF One CLKOUT into the 10M MMCX): count CLK-40M-FPGA (H16) cycles against CLKIN_10MHz (K17) in the PL, read the count from Linux, steer `dac5311` from a small daemon, and use `xo_correction` for the residual below one DAC step (~0.13 ppm). PPS (K18) could gate the counter the same way, but the user has no PPS source.
4. **SDR over Ethernet:** re-measure the streaming ceiling on flash boot (2 cores).
5. Possibly investigate the 1560 MHz spur (39×40 MHz). It rose from +24 to +36 dB after the DDR-500 HDL rebuild, presumably due to placement.
6. Possibly a stable Ethernet MAC (e.g. derived from the flash UniqueID like the USB one).
7. **Next ADI release: v0.40** (master, Vivado 2025.1, big restructure, mainline u-boot, `br2-external`) waits for its tag. Expect release-branch merges as described under History.
8. The user prefers config via config.txt, concrete verified steps, and no speculative ideas.
