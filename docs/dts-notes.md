# Mars devicetree analysis (Phase 2)

Done 2026-10-06 on ubhejane, in `~/mars-gpu/worktrees/linux-gpu` (branch `mars-gpu`, commit `54bf2745aa8e`).

## Conclusion

**The merged tree already enables the GPU, display and HDMI on the Milk-V Mars. No Mars-specific DTS patch is needed to enable these nodes.** Whether the Mars needs board-level fixes can only be shown on hardware (see the risks below).

## Why

The domibel branches change only two devicetree files:

- `jh7110.dtsi` (SoC level): adds the `gpu@18000000` node, the `vout_subsystem` with `dc8200`, the HDMI subsystem (`hdmi_controller` and `hdmi_phy`), `vout_syscon`, `voutcrg` (moved under the subsystem), and `xin24m`. It removes `hdmitx0_pixelclk`.
- `jh7110-common.dtsi` (shared by all JH7110 boards): adds the 512 MiB `linux,cma` default pool (`alloc-ranges = 0x70000000..0x90000000`), an `hdmi-connector` node, enables `&dc8200`, `&hdmi_controller` and `&voutcrg`, and adds the `hdmi_pins` pinmux (GPIO 0 and 1 for DDC SCL/SDA, GPIO 14 for CEC, GPIO 15 for HPD).

`jh7110-milkv-mars.dts` includes `jh7110-common.dtsi` directly, so it inherits all of this. Mars's own file only touches Ethernet, I2C0, eMMC/SD, PCIe, PHY, PWM, SPI0 and USB.

## Checks performed

- `make defconfig dtbs` for `ARCH=riscv` succeeds with no DTS warnings. `jh7110-milkv-mars.dtb` builds. Build dir: `~/mars-gpu/build/linux-gpu/`.
- I decompiled the Mars DTB and compared the `gpu@18000000`, `display-subsystem@29400000` (with `dc8200`, HDMI controller and PHY, `vout_syscon`, `voutcrg`) and `reserved-memory` nodes against the reference repo's `visionfive2-live.dts`. Apart from property ordering and phandle labels, they are identical: the same compatibles, regs, clocks, resets, power domains, interrupts and CMA pool.
- `gpu` has `status = "okay"` in `jh7110.dtsi`, as in the reference.
- Decompiled copies are kept in `~/mars-gpu/out/dts-analysis/` (`mars.dts`, `vf2.dts`, `jh7110-milkv-mars.dtb`).

## Risks and open items (to check on hardware or against the schematic)

1. **RAM size (resolved, pending confirmation).** The CMA pool is fixed at `0x70000000..0x90000000`. The source comment says it "fits in the memory every VisionFive 2 variant has" (2 GB or more). The Mars comes in 1, 2, 4 and 8 GB versions. A 1 GB Mars would not fit it: memory ends at `0x80000000`, so only 256 MiB of the range exists. The user believes this board has 8 GB, which is fine (memory spans `0x40000000..0x240000000`). Confirm with `free -h` or the U-Boot banner on first boot. No patch is needed unless that turns out to be wrong.
2. **HDMI pins.** The pinmux uses GPIO 0, 1, 14 and 15, put into `jh7110-common.dtsi`, which the Mars shares. We should confirm this against the Mars schematic. It is likely the same as the VF2 because both boards use the same common file, but we have not verified it.
3. **`cma=` on the command line.** The reference system boots with `cma=128M` in addition to the 512 MiB DT pool. The kernel prefers the command line for the default CMA area, so `cma=128M` may cause the DT pool to be ignored. Try with and without `cma=` on the Mars.
4. **Mars variants.** The `marscm` boards (CM, CM-Lite, CM-eMMC) are different boards with their own DTS files. We target the plain Mars (`jh7110-milkv-mars.dts`, `compatible = "milkv,mars"`).
5. **Dual PHY/second GMAC.** Not relevant to the GPU.

## Patch status

`~/mars-gpu/patches/` is empty. No Mars commit was made on `mars-gpu`. If we need a Mars patch later (pinout, or a RAM size other than 8 GB), it goes on `mars-gpu` and is exported with `git format-patch`.
