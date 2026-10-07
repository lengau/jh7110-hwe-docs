# Boot firmware and bootloader (Phase 5)

Built 2026-10-07 on ubhejane. **Build only: nothing has been flashed or written to any board.**

## What the Mars needs

- **No separate Mars U-Boot target.** U-Boot's `milk-v_mars.rst` says: *"U-Boot for the Milk-V Mars uses the same U-Boot binaries as the VisionFive 2 board. In U-Boot SPL the actual board is detected and the device-tree patched accordingly."* The build uses `starfive_visionfive2_defconfig`. The board is identified from the I2C EEPROM product ID (`MARS...`), and U-Boot then sets `fdtfile=starfive/jh7110-milkv-mars.dtb`. The FIT image contains the Mars DTB, among others (`board/starfive/visionfive2/spl.c`, `starfive_visionfive2.c`).
- OpenSBI must be v1.5 or newer, generic platform, `fw_dynamic.bin`. U-Boot's doc example uses v1.7. We use **v1.9** (newest tag). If it misbehaves on the board, try v1.7 (the tag the doc names).
- Boot chain: BootROM, then U-Boot SPL (initializes DRAM and PLLs), then OpenSBI `fw_dynamic` plus U-Boot proper (one FIT image, `u-boot.itb`), then the kernel via `bootstd`. The enabled boot methods include `extlinux`, EFI loader and EFI boot manager, so both an `extlinux.conf` and a GRUB-EFI install can be booted.

## Build

```
~/mars-gpu/scripts/build-uboot.sh          # CLEAN=1 to wipe the build dirs; JOBS=80 by default
```

Log: `~/mars-gpu/logs/build-uboot.log`. Takes under a minute. Sources: `worktrees/opensbi` (`v1.9`) and `worktrees/u-boot` (`v2026.07`); see `sources.md`. The script:

1. Builds OpenSBI: `make PLATFORM=generic` into `build/opensbi`, giving `platform/generic/firmware/fw_dynamic.bin`.
2. Builds U-Boot with `starfive_visionfive2_defconfig` and `OPENSBI=<that fw_dynamic.bin>` into `build/u-boot`.
3. Builds a second, **debug-UART** variant in `build/u-boot-debuguart`, with the config changes from U-Boot's doc (`DEBUG_UART_NS16550`, base `0x10000000`, clock 24 MHz, shift 2, SBI debug console off, and the same for SPL). SPL is silent without these, because U-Boot proper uses the SBI console. This variant gives output from SPL on the first bring-up.

Both builds use a cross-compiler `riscv64-linux-gnu-` (GCC 13.3).

## Artifacts (`~/mars-gpu/out/boot/`)

| File | Size | Purpose |
|---|---|---|
| `default/u-boot-spl.bin.normal.out` | 152,105 B | SPL with the StarFive header (what the BootROM loads) |
| `default/u-boot.itb` | 1,631,245 B | FIT: U-Boot main (load `0x40200000`) + OpenSBI (`0x40000000`) + the board DTBs |
| `debuguart/u-boot-spl.bin.normal.out`, `debuguart/u-boot.itb` | 152,417 B and 1,631,357 B | same, with an early debug UART |
| `default/.config`, `debuguart/.config` | | final U-Boot configs |
| `fw_dynamic.bin` | | OpenSBI payload used |
| `SHA256SUMS` | | checksums of all of the above |

U-Boot reports `U-Boot 2026.07` in the binary. The FIT contains DTBs for the Mars, `marscm-emmc`, `marscm-lite`, VisionFive 2 (v1.2a, v1.3b, lite), Star64, Orange Pi RV and DeepComputing FML13V01.

## Flash layout and boot modes (from `jh7110_common.rst`)

Boot-source selection uses the RGPIO1:RGPIO0 strap pins, which the board exposes as boot-mode switches or jumpers. **Check how the Mars exposes them before touching anything (Phase 8).** For SPL:

| RGPIO1:RGPIO0 | U-Boot SPL loads U-Boot from |
|---|---|
| 0:0 | QSPI NOR, offset `0x100000` |
| 0:1 | SD (MMC2), a raw or partition location |
| 1:0 | eMMC (MMC1), a raw or partition location |
| 1:1 | UART (YMODEM), the recovery mode |

The U-Boot doc calls the SD and eMMC BootROM modes *deprecated* (the BootROM driver does not work with some SD cards), and recommends QSPI. QSPI NOR layout (16 MB):

| Offset | Length | Content |
|---|---|---|
| `0x000000` | `0x0f0000` | `u-boot-spl.bin.normal.out` |
| `0x0f0000` | `0x010000` | U-Boot environment |
| `0x100000` | `0xf00000` | `u-boot.itb` |

For SD cards the BootROM reads the GPT header from sector 1 and loads SPL from the partition whose type GUID is `2E54B353-1271-4842-806F-E436D6AF6985` (from the doc; eMMC loads SPL from sector 0 instead).

Updating QSPI from the U-Boot console (`sf probe`, `loady && sf update $loadaddr 0 $filesize`, `env erase`, then `sf update ... 100000`) or from Linux (`flashcp` on `/dev/mtd0` to `mtd2`) is described in the U-Boot doc. **Not done, and not needed for the first tests.**

### Recommendation for the first boots

- Don't overwrite the board's QSPI at first. Keep the vendor/Ubuntu U-Boot there as the known-good fallback. If a flash goes wrong, the recovery path is UART mode (1:1) plus a YMODEM load, which needs the serial console.
- To test our U-Boot build, use the SD (or UART) boot mode with our SPL/itb on the card, if the Mars switches allow it. Decide this in Phase 8 once we have seen what the board does and what boot media it has.
- Use the `debuguart` variant for the first bring-up.

## Booting our kernel from U-Boot

U-Boot's `bootstd` scans for `extlinux/extlinux.conf` and EFI binaries on the boot devices (NVMe and USB are scanned in `preboot`, SD/eMMC as usual). The kernel needs **our** DTB, not U-Boot's own (which comes from upstream Linux 7.x and does not have the GPU and HDMI nodes). Options:

1. **extlinux** (simplest, deterministic). `/boot/extlinux/extlinux.conf` on the rootfs or boot partition, using `fdtdir` so that U-Boot picks `$fdtdir/$fdtfile` (`starfive/jh7110-milkv-mars.dtb` on a Mars). Draft, to be finalized in Phase 7 with the real image layout:

   ```
   default mars-gpu
   timeout 30
   label mars-gpu
       menu label Mars GPU kernel (7.3.0-rc5-00040-g54bf2745aa8e)
       kernel /boot/vmlinuz-7.3.0-rc5-00040-g54bf2745aa8e
       initrd /boot/initrd.img-7.3.0-rc5-00040-g54bf2745aa8e
       fdtdir /boot/dtbs/7.3.0-rc5-00040-g54bf2745aa8e/
       append root=LABEL=cloudimg-rootfs rw rootwait console=ttyS0,115200 earlycon=sbi
   ```

   The DTBs are at `~/mars-gpu/out/dtbs/starfive/`, and `fdtdir` expects them below `/boot/dtbs/<release>/starfive/`.
2. **GRUB EFI** as in the Ubuntu image: Ubuntu's GRUB can load `/boot/dtb-<release>` if present, but it must then be the Mars DTB. Ubuntu's own image uses this route, so check how its `+jh7110` image handles the DTB in Phase 8 before choosing.

Kernel command line notes:

- `console=ttyS0,115200 earlycon=sbi` matches U-Boot's own default (`console=ttyS0,115200 debug rootwait earlycon=sbi`).
- Whether to add `cma=128M` is an open item (`dts-notes.md`, item 3). The first Mars boot should be without it, then with it.

## Not done / still to decide

- Whether to ship the QEMU-style Ubuntu image's GRUB or use extlinux for the board image (Phase 7).
- Verifying that OpenSBI 1.9 works on the board (fallback: v1.7).
- The Mars boot-mode switch positions, and what the board boots from now (Phase 8).
- Nothing has been run on hardware. U-Boot can't be tested in QEMU for this board, because QEMU has no JH7110 machine.
