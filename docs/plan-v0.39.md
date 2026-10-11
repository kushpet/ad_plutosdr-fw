# Plan: merge ADI v0.39 into `libresdr` and build with Vivado 2025.2

Written 2026-10-11 on the old system (Ubuntu 20.04, Vivado/Vitis 2022.2), to be carried out on the user's other system (Ubuntu 22.04, Vivado 2025.2). Read `CLAUDE.md` first: it holds the board facts, the repository rules and the open items. Local Claude memory does not move between systems, so everything needed is in `CLAUDE.md`, `README.md` and this file.

Goal: tag `libre-v0.39` in all repositories, built with Vivado 2025.2, with every LibreSDR fix of `libre-v0.38.3` kept, and verified on the board.

## Facts checked on 2026-10-11

| | ADI v0.38 | ADI v0.39 | ADI master (future v0.40, untagged) |
| --- | --- | --- | --- |
| Date | 2023-11-17 | 2024-10-15 | last merge 2026-08-13 (`staging/next-v0p40`) |
| Vivado | 2022.2 | **2023.2** | 2025.1 |
| Linux | 5.15 | **6.1** (`f3da30df6004`) | newer |
| u-boot | `u-boot-xlnx` `90401ce9ce` | **same commit**: nothing to merge | mainline `u-boot` submodule |
| hdl | `1978df2985ce` | `065c8f186ef8` | `4840c81f2af1` |
| buildroot | `f70f4aff40bc` | `e783aadcceb` | `2e2994de5213` + new `br2-external` |

- No ADI release targets Vivado 2025.2. v0.39 expects 2023.2. Master (2025.1) is a big restructure; leave it until ADI tags v0.40.
- The version checks can be bypassed. For the superproject Makefile, set `export VIVADO_VERSION=2025.2`; it calls `$(error)` on a mismatch otherwise. For the HDL scripts, set `export ADI_IGNORE_VERSION_CHECK=1`, which `hdl/scripts/adi_env.tcl` honours.
- ADI's own HDL moves 2023.2 → 2024.2 → 2025.1 only changed the version string in `adi_env.tcl` (commits `a412d1eaf`, `4fa4bbe92`). So the HDL sources are expected to work with newer Vivado. Not yet verified with 2025.2.
- The libre block design uses `xlconcat`, `xlslice` and `util_vector_logic`. Xilinx deprecated these in 2024.2 in favour of "inline HDL" variants, and announced their removal for the 2026 Vivado releases (ADI commit `eadd7d695`). They should still exist in 2025.2; expect deprecation warnings. If they are missing, port as ADI did in `eadd7d695`.
- The FSBL is built with `xsct` (`scripts/create_fsbl_project.tcl`), and ADI master still does this with 2025.1. Check that the 2025.2 install provides `xsct` (and `bootgen`).
- The libre HDL project (`hdl/projects/libre`) was derived by day0wl from an older pluto design without the TDD block. ADI's pluto changes v0.38 → v0.39 are almost all about TDD and DMA sync (`SYNC_TRANSFER_START`). They do not apply to libre. `tx_upack/reset` is already connected in libre.
- **The Winbond EAR kernel fix is still needed in 6.1.** Xilinx `linux-xlnx` `xilinx-v2023.2`, `drivers/mtd/spi-nor/core.c`, `read_ear()` and `spi_nor_write_ear()` still list only AMD/ST/Macronix/PMC. In 6.1 the field is `nor->info->id[0]`, not `nor->jedec_id`. Without the fix, flashing breaks again (see `CLAUDE.md`, QSPI).
- **Rollback images:** `libre-v0.38.3` is on the old system's disk at `~/libresdr-images/libre-v0.38.3/` (fw zip with `.frm`/`.dfu`, jtag-bootstrap zip). You can rebuild them from the tag only with Vivado 2022.2.

## Phase 0: set up the new system

The user works with **the same repository on the old system's disk**, mounted in Ubuntu 22.04 (no new clone). The repository path is therefore different from `/home/user/GitHub/kushpet/ad_plutosdr-fw`. The same holds for `~/buildroot-dl` and `~/libresdr-images` on that disk. Paths in `CLAUDE.md` (Building) are those of the old system.

1. Packages: the `apt-get install` lines in `README.md` → Host setup. Note any that are missing on 22.04.
2. Git:
   - If the user's UID differs from 1000 on the new system, git refuses the repositories ("dubious ownership"). Fix that with `git config --global --add safe.directory '*'`, or align the UID.
   - Author identity is stored in each repo's local config (set 2026-10-11): name `Dmitry Kushpet`, superproject email `kushpet@users.noreply.github.com`, submodules `kushpet@gmail.com`. Check it with `git config user.email` in each repo.
   - Pushing needs a GitHub SSH key on the new system. Copy `id_ed25519*` from the old disk's `home/user/.ssh/` to `~/.ssh/` (mode 600), or add a new key to GitHub. Test with `ssh -T git@github.com`.
   - The `upstream` remotes already exist in all repositories. Submodules are on their `libresdr` branches; check with `git -C <repo> branch --show-current` before committing.
3. **Clean the old build trees before the first build.** They hold absolute paths from the old system and 2022.2 Vivado state:
   - `rm -rf buildroot/output` (3.8 GB; the toolchain is rebuilt);
   - `git clean -fdX` in `hdl/` and `u-boot-xlnx/`;
   - `make -C linux mrproper`.
   After this, the old system can no longer build incrementally. It does not need to: the v0.38.3 images are archived.
4. Vivado 2025.2: find the settings script that puts **both** `vivado` and `xsct` on the PATH. With 2025.x it may be `<install>/2025.2/Vivado/settings64.sh` or the Vitis one. Check with `vivado -version`, `which xsct bootgen`.
5. `BR2_DL_DIR`: point it at the old disk's `home/user/buildroot-dl` (download cache, saves time).
6. Rollback images: `home/user/libresdr-images/libre-v0.38.3/` on the old disk.
7. Board access from the new system:
   - the console is FTDI interface 02 (`/dev/ttyUSBn`, 115200), and the user must be in the `dialout` group;
   - USB RNDIS: the board is at 192.168.2.1; NetworkManager creates the PC side by itself;
   - `dfu-util` needs a udev rule for `0456:b673` and `0456:b674` (the old system had `53-adi-plutosdr-usb.rules`), or root.

## Phase 1: does our current HDL build with 2025.2? (no merge yet)

This isolates "new Vivado" from "new ADI code". It takes about 30–40 minutes and needs no buildroot.

```bash
git checkout libre-v0.38.3 && git submodule update     # or stay on the libresdr tip
source <2025.2 settings64.sh>
export ADI_IGNORE_VERSION_CHECK=1
make -C hdl/projects/libre
```

Check the result: the build ends without errors, timing is met (`hdl/projects/libre/timing_impl.log`), and the IP upgrade and deprecation messages are as expected. If it fails, fix that first (IP versions, deprecated cores), on `libresdr` in `hdl`. Run `git clean -fdX` in `hdl/` before the first 2025.2 build (stale 2022.2 IP builds).

## Phase 2: merge v0.39, repository by repository

Use the same scheme as `libre-v0.38`: on each `libresdr` branch, a merge commit whose second parent is the ADI v0.39 commit. The merge commit message records what was resolved and why. Fetch first: `git fetch upstream` (the superproject also gets tag `v0.39`).

1. **hdl**: `git merge 065c8f186ef8`. Conflicts are expected only where LibreSDR touched shared files. `projects/libre` is ours, so check it against the merged library (IP parameter changes in `axi_ad9361`, `axi_dmac`, `util_cpack2/upack2`). Decide whether to leave the ADI version string at 2023.2 and use `ADI_IGNORE_VERSION_CHECK`, or set it to 2025.2 for libre.
2. **linux** (5.15 → 6.1, the biggest step): `git merge f3da30df6004`. LibreSDR changes to carry over; list them with `git diff libre-v0.38 libre-v0.38.3` and the day0wl patch in `libre-v0.37`:
   - `arch/arm/boot/dts/zynq-libre.dts{,i}`: compare ADI's `zynq-pluto-sdr*.dts{,i}` v0.38 → v0.39 and port relevant changes. Keep LVDS (`adi,lvds-mode-enable`), `winbond,w25q256`, LEDs, gpio-keys on MIO12, the Realtek PHY. In 6.1 the dts files may live under a different path; check.
   - `arch/arm/configs/zynq_libre_defconfig`: rebuild from the v0.39 `zynq_pluto_defconfig` plus the libre deltas (Realtek PHY driver, GCC plugins off, ...). Diff the old libre and pluto defconfigs to get the delta list.
   - `drivers/mtd/spi-nor/core.c`: the Winbond Unique ID print (`SPI-NOR-UniqueID`, which `S23udc` parses for hw_serial) and the **EAR fix** (`CFI_MFR_WINBND` in `read_ear` and `spi_nor_write_ear`).
3. **buildroot**: `git merge e783aadcceb`. Port the ADI `board/pluto` changes v0.38 → v0.39 into `board/libre` (rule in `CLAUDE.md`), and keep the libre additions (`S22ethlink`, `eth_mode`, `maxcpus`, `pl_led1`, `update.sh`). Compare `configs/zynq_pluto_defconfig` v0.38 → v0.39 with `configs/zynq_libre_defconfig`. Check what toolchain v0.39 uses: the old one was Linaro GCC 7.3, and the superproject Makefile's `CROSS_COMPILE` must match.
4. **u-boot-xlnx**: nothing to merge (same ADI commit). Keep `libresdr`.
5. **superproject**: `git merge v0.39`. Keep `scripts/libre.mk`, `scripts/libre.its`, `TARGET ?= libre`, the libre bits in the Makefile and the docs. Set `VIVADO_VERSION` (see above). Commit the new submodule pointers only after each submodule is pushed.

## Phase 3: build

```bash
export VIVADO_SETTINGS=<2025.2 settings64.sh>
export VIVADO_VERSION=2025.2
export ADI_IGNORE_VERSION_CHECK=1
export BR2_DL_DIR=$HOME/buildroot-dl
make && make sdimg
```

New host issues on 22.04 (buildroot host tools, Python versions) get fixed and noted in `CLAUDE.md`.

## Phase 4: test on the board

The board runs `libre-v0.38.3`, whose kernel flashes correctly. Order: first without touching flash, then flash.

1. **RAM DFU first**, with no flash write: on the board console run `device_reboot ram`, then `dfu-util -D build/libre.dfu -a firmware.dfu && dfu-util -a firmware.dfu -e`. This tests the new kernel and rootfs with the old u-boot, and this release does not change u-boot anyway.
2. Checklist on the RAM-booted image:
   - dmesg has no `failed to read ear reg`, and `SPI-NOR-UniqueID` is printed;
   - `/dev/mtd3` at offsets 0 and 16 MiB matches what u-boot reads at 0x200000 and 0x1200000 (method in `CLAUDE.md`);
   - ad9361 probes, the LVDS interface tuning passes, `iio_info` works, the RX BIST tone is clean (`bist_tone "2 0 0 0"`, 2R2T at 30.72 Msps; do not go beyond the 125 MHz rx_clk constraint);
   - FM reception in SDRangel;
   - `nproc` = 2, `eth_mode` and `maxcpus` appear in config.txt and take effect;
   - the USB gadget stays up for several minutes, and Ethernet works if the new NIC has arrived;
   - the Button IRQ (`/proc/interrupts`) does not count console input.
3. If everything passes, flash through the PlutoSDR drive: `boot.frm` first, then `libre.frm`. Verify md5 (method in `CLAUDE.md`), set `fit_size` if needed, power-cycle, and repeat the key checks from flash. Then check the DFU button at power-up.
4. Rollback at any point: flash the `libre-v0.38.3` images through the drive or DFU, or boot from an SD card.

## Phase 5: finish

- Tag `libre-v0.39` (annotated) in all five repositories and push it. Build again from the tag so the version strings are clean (no `-dirty`). Commit `CLAUDE.md` changes only after tagging.
- Update `README.md` (tags table, Vivado version, host setup for 22.04) and `CLAUDE.md` (history, build section, verified facts). Keep the history diagram style.
- The user then tries SDRangel and ADI IIO Oscilloscope on Ubuntu 22.04. That was the reason for the move.
