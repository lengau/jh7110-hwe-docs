# Plan: PowerVR GPU bring-up on Milk-V Mars

Goal: reproduce https://github.com/domibel/visionfive2_gpu_bringup (PowerVR BXE-4-32 + DC8200/HDMI on JH7110) on a Milk-V Mars, and verify with `vulkaninfo`, `vkmark` and `glmark2`.

Strategy: do everything that does not need the board up-front on **ubhejane (10.13.1.3)**, so that the board is only needed for the final boot-and-test phase. This repo holds documentation only (`docs/`). Old material is in `holding/` and is not part of this effort.

## Context

- Mars and VisionFive 2 both use the JH7110 SoC, so the GPU (BXE-4-32, BVNC 36.50.54.182) and the display pipeline (DC8200 + Inno HDMI) are the same. The differences are board-level: devicetree, regulators, HDMI connector wiring, boot firmware and storage.
- ubhejane (surveyed 2026-10-06): x86_64, Ubuntu 24.04.5, 80 cores, 1 TiB RAM, 1.6 TiB free, user `ubuntu` with passwordless sudo. It has `git`, `gcc` and `make`. It lacks the riscv64 cross toolchain, `dtc`, `bison`, `flex`, `mmdebstrap`, `qemu-user-static`, `tio` and any TFTP/NFS server. No serial adapter is attached to it. The apt candidates for `gcc-riscv64-linux-gnu`, `mmdebstrap`, `qemu-user-static` and `tio` all exist.
- Upstream refs verified on 2026-10-06:
  - Mainline tag `v7.3-rc5` exists (`5a956dde`). `v7.3-rc6` also exists.
  - `domibel/linux` has `jh7110_dc8200_hdmi_v7.3-rc5` (`95e43ff5`) and `powervr_on_jh7110_visionfive2_v7.3-rc5` (`77f50b34`). It also has the same pair for `v7.3-rc4`, `v7.2`, `v7.1-rc5` and `v7.0.3`, which are fallbacks.
  - The reference repo's HEAD is `3735836d`.

## Upstream recipe (from the reference repo)

- Kernel: `v7.3-rc5` plus the two `domibel/linux` branches above, merged in that order (powervr first, then dc8200_hdmi).
- Config additions: `DRM_POWERVR=m`, `DRM_VERISILICON_DC=y`, `CMA=y`, `ERRATA_SIFIVE=y`, `ERRATA_SIFIVE_XPBMTUC=y`, `STARFIVE_STARLINK_CACHE=y`, `RISCV_NONSTANDARD_CACHE_OPS=y`, `DRM_STARFIVE_JH7110_INNO_HDMI=y`, `PHY_STARFIVE_JH7110_INNO_HDMI=y`, `SOC_STARFIVE_JH7110_VOUT_SUBSYSTEM=y`, `SOC_STARFIVE_JH7110_HDMI_SUBSYSTEM=y`.
- Kernel cmdline: `cma=128M` or more (64M fails).
- Firmware: `rogue_36.50.54.182_v1.fw` from `imagination/linux-firmware` on gitlab.freedesktop.org, pinned to commit `8a58f818`, installed in `/lib/firmware/powervr/`.
- Userspace: Mesa 26.2.x with `PVR_I_WANT_A_BROKEN_VULKAN_DRIVER=1` and `MESA_VK_DEVICE_SELECT=1010:36054182`. Tools: `vulkan-tools`, `vkmark`, `glmark2-es2-wayland`, `labwc`, `weston`, `sway`.
- Bootloader: U-Boot 2025.01 or 2026.07, with the DTB loaded from the ESP.
- Reference numbers (README of the reference repo): for example, vkmark KMS 1920x1080: clear 485 fps, cube 140 fps.

## Directory layout on ubhejane

Everything lives under `~/mars-gpu/`:

```
~/mars-gpu/
  repos/                bare/mirror clones (shared object stores)
    linux.git           torvalds/linux, plus remote "domibel"
    reference.git       domibel/visionfive2_gpu_bringup
    firmware.git        imagination/linux-firmware
    mesa.git            mesa/mesa (only if we need a newer Mesa than the distro's)
    u-boot.git          u-boot/u-boot
    opensbi.git         riscv-software-src/opensbi
  worktrees/            one worktree per purpose
    linux-base          v7.3-rc5, untouched
    linux-gpu           merged branch "mars-gpu" (the one we build)
    reference           checkout of the reference repo (README, .config, live DTS, scripts)
    firmware            pinned firmware commit
    u-boot              pinned release tag
    opensbi             pinned release tag
  build/                out-of-tree build dirs (linux-gpu/, u-boot/, ...)
  out/                  deliverables: Image, modules, dtbs, rootfs, firmware, boot files
  patches/              our Mars-specific patch series (git format-patch output)
  logs/
  scripts/              all scripted steps below
```

## Phases (all on ubhejane)

### Phase 0: Host setup

- `apt install`: `gcc-riscv64-linux-gnu`, `g++-riscv64-linux-gnu`, `device-tree-compiler`, `bison`, `flex`, `libssl-dev`, `libelf-dev`, `libncurses-dev`, `bc`, `u-boot-tools`, `swig`, `python3-dev`, `python3-setuptools`, `python3-pyelftools`, `ccache`, `mmdebstrap`, `qemu-user-static`, `binfmt-support`, `debian-archive-keyring`, `ubuntu-keyring`, `parted`, `dosfstools`, `e2fsprogs`, `kpartx`, `tio`, and `tftpd-hpa` plus `nfs-kernel-server` (installed, but left disabled until we pick a boot method).
- Check that `qemu-riscv64` binfmt is registered after install (`/proc/sys/fs/binfmt_misc/qemu-riscv64`).
- Create the directory layout above. Install `ccache` and set it up for the cross compiler.
- Write `docs/host-setup.md`.

### Phase 1: Clone repositories and set up worktrees

- Clone `torvalds/linux` as a bare repo in `repos/linux.git`. Add the `domibel` remote and fetch the tags and the two `v7.3-rc5` branches (plus the `v7.2` pair as a fallback).
- Clone the other repos (reference, firmware, u-boot, opensbi, and Mesa only if needed) as bare repos.
- Create worktrees:
  - `linux-base`: detached at `v7.3-rc5`.
  - `linux-gpu`: new branch `mars-gpu` from `v7.3-rc5`, then merge `domibel/powervr_on_jh7110_visionfive2_v7.3-rc5` and `domibel/jh7110_dc8200_hdmi_v7.3-rc5`. Record any conflicts and how they were resolved.
  - `reference`: the reference repo at its current HEAD.
  - `firmware`: pinned to `8a58f818`.
  - `u-boot` and `opensbi`: the latest release tags that the reference (2025.01 / 2026.07) points to.
- Record every pinned commit hash in `docs/sources.md`, so that moving branches cannot break reproducibility.

### Phase 2: Inspect the Mars devicetree situation (no hardware needed)

- In `linux-gpu`, check whether `arch/riscv/boot/dts/starfive/jh7110-milkv-mars.dts` exists and what it includes. Build it with `make dtbs` and decompile with `dtc`.
- Diff it against `jh7110-starfive-visionfive-2-v1.3b.dts` and against the reference repo's `visionfive2-live.dts`. List which GPU, DC8200, HDMI, power-domain, clock and reset nodes are enabled for the VF2 but not for the Mars, and the board-level differences (HDMI pins, hotplug GPIO, regulators, PHY).
- Write a minimal Mars patch (`patches/0001-...`) that enables the same nodes. Commit it on the `mars-gpu` branch. Document it in `docs/dts-notes.md`.

### Phase 3: Build the kernel

- Save the config additions as `scripts/gpu.fragment`.
- In `build/linux-gpu/`: `make ARCH=riscv CROSS_COMPILE="ccache riscv64-linux-gnu-" -C worktrees/linux-gpu O=... defconfig`, then merge the fragment with `scripts/kconfig/merge_config.sh` and run `olddefconfig`. Check that every config symbol from the fragment survived `olddefconfig`.
- Build `Image`, `modules` and `dtbs` with `-j80`. Install the modules into `out/modules/` and copy `Image`, `jh7110-milkv-mars.dtb` and `.config` into `out/`.
- Also build the VF2 DTB, so we can compare the two.
- Put the commands in `scripts/build-kernel.sh`. Write `docs/kernel.md`.

### Phase 4: Kernel smoke test in QEMU (on ubhejane)

**Status: done 2026-10-07, 67 of 67 checks pass. See `qemu-smoke.md` for the results, the scripts and what is not covered.** A follow-up boots the same kernel inside a stock Ubuntu 24.04.5 riscv64 image under QEMU 11.1.2 and runs the real snapd and LXD; that also works (`qemu-smoke.md`, second part). The text below is the original plan; the implemented checks are slightly different in detail (the initramfs is an Ubuntu noble riscv64 extract rather than plain busybox).

Goal: catch gross kernel problems, and check the container/snap config options, before any hardware is involved. QEMU has no JH7110 machine, so this uses the generic `virt` machine. The Mars devicetree, the GPU, HDMI and the JH7110-specific drivers are **not** exercised. Those wait for the board.

Setup:

- Install `qemu-system-misc` (provides `qemu-system-riscv64`; the default OpenSBI firmware comes with `qemu-system-data`).
- Build a small initramfs (busybox, riscv64, via `mmdebstrap` or the distro's `busybox-static`) with an init script that runs the checks below and powers off. Keep it in `~/mars-gpu/out/qemu/`.
- Script `scripts/qemu-smoke.sh`: `qemu-system-riscv64 -M virt -cpu rv64 -smp 4 -m 4G -nographic -kernel out/Image -initrd out/qemu/initramfs.cpio.gz -append "console=ttyS0 ..." -netdev user,id=n0 -device virtio-net-pci,netdev=n0`. It logs to `~/mars-gpu/logs/qemu-smoke.log` and fails on a timeout, a panic, or a failed check.

Checks:

1. The kernel boots to the init script, with no panic or oops. Record the `dmesg` warnings.
2. The modules install tree is usable: copy `out/modules` in (as a disk image or 9p share), run `depmod`, and `modprobe` the container-related modules (`veth`, `bridge`, `nf_tables`, `nft_nat`, `nft_masq`, `tun`, `vxlan`, `macvlan`, `cuse`).
3. Containers: all namespace types work (`unshare` for user, pid, net, mount, uts, ipc, cgroup, time); cgroup v2 mounts and has the controllers (`cpu`, `memory`, `pids`, `io`, `cpuset`); a veth pair can be created and moved into a netns; `overlayfs` mounts; `fuse` is present (`/dev/fuse`).
4. Snap: build a tiny squashfs on the host with each compressor (xz, zstd, lz4, lzo, gzip), loop-mount each in the guest, and read a file back. Check xattrs on squashfs.
5. Security: AppArmor is enabled (`/sys/kernel/security/apparmor`, `/sys/module/apparmor/parameters/enabled` is `Y`) and `cat /sys/kernel/security/lsm` lists `apparmor`. Seccomp filter mode is available (`/proc/self/status` shows `Seccomp`).
6. `nft` can create a table with a NAT/masquerade chain, if the `nft` binary is in the initramfs.
7. Save the results in `docs/qemu-smoke.md`, together with the exact command line.

Later reuse: once the full rootfs exists (Phase 6), boot that same kernel in QEMU against the real root filesystem, and try `snapd` and `lxd` there as the final software-side check. The rootfs should be Ubuntu riscv64 if we want those two to be the stock packages.

Known limits: TCG emulation on x86 is slow but fine for a smoke test. KVM on the Mars is not possible (JH7110's U74 cores have no hypervisor extension), so LXD virtual machines are out; only system containers apply.

### Phase 5: Boot firmware and bootloader (build only)

**Status: done 2026-10-07. See `boot.md`.** Findings: the Mars uses the VisionFive 2 U-Boot build (`starfive_visionfive2_defconfig`) and U-Boot identifies the Mars from its EEPROM; there is no separate Mars target. Built OpenSBI v1.9 and U-Boot v2026.07 (normal and debug-UART variants); nothing flashed. The text below is the original plan.

- Check whether the Mars needs a custom SPL/U-Boot/OpenSBI build, or whether the vendor firmware already on the board is enough. Build `u-boot-spl.bin.normal.out` and `u-boot.itb` for the Mars (`starfive_visionfive2_defconfig` with the Mars DT, or the Mars target if the pinned U-Boot has one) with OpenSBI as the `fw_dynamic` payload. Keep the artifacts in `out/` but do not flash anything until the board phase.
- Write the `extlinux.conf` / boot script with `cma=128M` and the Mars DTB path.
- Write `docs/boot.md`.

### Phase 6: Root filesystem

**Status: mostly done 2026-10-07. See `rootfs.md`.** Ubuntu 24.04.5 chosen (26.04 is RVA23-only and unusable on the JH7110), Mesa 26.2.3 built for riscv64 with the PowerVR Vulkan driver + zink + llvmpipe into `/usr/local`, pinned GPU firmware identified. Image assembly (Mesa install, firmware, benchmark packages, extlinux) still open for Phase 7. The text below is the original plan.

- Build a riscv64 rootfs with `mmdebstrap --arch=riscv64` into `out/rootfs/` (Debian sid or trixie, whichever has Mesa 26.2.x; check the version first), running under qemu-user.
- Packages: `mesa-vulkan-drivers`, `vulkan-tools`, `vkmark`, `glmark2-es2-wayland`, `labwc`, `weston`, `sway`, `openssh-server`, `kmod`, `linux-base`, `systemd-sysv`, `network-manager` or `systemd-networkd`, `sudo`, `libdrm-tests`, and `mesa-utils`.
- Install the pinned GPU firmware into `/lib/firmware/powervr/`, the kernel modules from `out/modules/`, and the test scripts from the reference repo. Set up users, the `render` and `video` groups, the hostname, an SSH key and a serial getty on `ttyS0` at 115200.
- Package it as an image (`out/mars-gpu.img`: an ESP or boot partition plus an ext4 root) and, optionally, as a tarball for NFS root.
- Write `docs/rootfs.md`.

### Phase 7: Pre-hardware validation on ubhejane

- Check the rootfs under `qemu-user` and `chroot`: `vulkaninfo`, `vkmark` and `glmark2` binaries run and their libraries resolve (no GPU, so expect only llvmpipe).
- Boot the kernel under `qemu-system-riscv64 -M virt` with the real rootfs, reusing the Phase 4 script, to catch gross kernel and rootfs problems (the GPU and HDMI will not be present). Try `snapd` and `lxd` here if they are installed.
- Check the image: partition table, the files `extlinux.conf` points to, that the DTB is in place, module dependencies (`depmod`), and the firmware file's checksum.
- Optional: stand up the TFTP/NFS tree from `out/` so that a network boot is a matter of enabling the services.

### Phase 8: Baseline boot from a premade image (needs the Mars)

First contact with the hardware, before any of our own builds. It uses a known-good premade image (for example, a Debian or Ubuntu riscv64 image for the Mars or JH7110) and changes nothing about our kernel.

Prerequisites: a serial console (a USB-UART on the 3-pin header, ideally attached to ubhejane so that `tio` can log it), a microSD card or other boot media, and a way to reach the board over the network.

Steps:

1. Write the premade image to the boot media (the image is downloaded and checked on ubhejane first).
2. Boot with the serial log captured to `~/mars-gpu/logs/` (`tio -L`). Record the U-Boot banner, board revision, kernel version and boot media.
3. Collect: `free -h` and `dmesg | grep -i -E "memory|cma"` (**confirms the 8 GB RAM**), `dtc -I fs -O dts /sys/firmware/devicetree/base` (live devicetree), `/proc/cmdline`, `lsmod`, and `dmesg` lines for the DRM, HDMI and GPU drivers if there are any.
4. Check whether the vendor kernel shows HDMI working, and note which pins and clocks it uses. This is a sanity check on the board only; it does not prove our devicetree pins are right.
5. Diff the live devicetree against `~/mars-gpu/out/dts-analysis/mars.dts` and update `docs/dts-notes.md`.
6. Decide the boot method for our own image (SD, eMMC or network) and whether the board can be power-cycled remotely.

Answers this phase gives us: RAM size, board revision, bootloader version, boot media, and how the serial and network connections work. It cannot answer the HDMI pin question or the `cma=` question; those need our kernel (Phase 9).

Write the results in `docs/baseline.md`.

### Phase 9: Our image on the board (needs the Mars)

Prerequisites: everything from Phase 8. Anything Phase 8 did not settle still needs an answer.

1. Write the image or set up a network boot, then boot and watch the serial log. Check that `dmesg` shows no `fbdev: Failed to setup emulation (ret=-12)` and that the HDMI console works.
2. `modprobe powervr`. Check `/dev/dri/renderD128` and the firmware load in `dmesg`.
3. `PVR_I_WANT_A_BROKEN_VULKAN_DRIVER=1 vulkaninfo --summary` lists `PowerVR B-Series BXE-4-32 MC1`.
4. `vkmark --winsys headless`, then `--winsys kms`. Then labwc and weston with `glmark2-es2-wayland -b jellyfish`.
5. Compare against the reference numbers. Save everything under `docs/results/`.
6. Boot with and without `cma=128M` and compare `dmesg` and `/proc/meminfo` (CmaTotal), to settle whether the command line overrides the 512 MiB pool in the devicetree. The HDMI output working (EDID read, hotplug detected) also confirms the HDMI pins.
7. Iterate on failures (CMA too small, firmware mismatch, a missing power domain or clock in the DTS, missing cache-ops or errata config, `ErrorOutOfDeviceMemory`). Record each symptom, cause and fix in `docs/troubleshooting.md`.

## Docs layout (this repo)

```
docs/
  plan.md             this file
  host-setup.md       ubhejane packages, directories, access
  sources.md          every repo, branch and pinned commit
  kernel.md           merge log, config fragment, build commands
  dts-notes.md        Mars DT analysis and changes
  boot.md             U-Boot/OpenSBI/extlinux details
  rootfs.md           rootfs build and contents
  qemu-smoke.md       Phase 4: QEMU smoke-test results
  baseline.md         Phase 8: premade-image boot results and live DT
  troubleshooting.md  symptoms, causes and fixes
  results/            logs, vulkaninfo and benchmark output
```

## Risks

- The `domibel` branches are rolling and can be force-pushed. We pin commit hashes in `docs/sources.md` as soon as we fetch them.
- The Mars DTS may need real work beyond enabling nodes. This is the biggest unknown, and we find out in Phase 2, before touching hardware.
- The Vulkan driver is experimental, and the reference results depend on the exact Mesa (26.2.x) and firmware versions. The riscv64 distro may not have that Mesa yet; if not, we build Mesa from `mesa.git` in the rootfs.
- Without a serial console, early-boot failures are very hard to debug. We need to settle that before Phase 8.

## Next step

Approve this plan, then start with Phase 0 and Phase 1 on ubhejane.
