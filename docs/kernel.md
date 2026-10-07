# Kernel build (Phase 3)

Built 2026-10-07 on ubhejane (rebuilt clean the same day with the containers fragment). Source: `~/mars-gpu/worktrees/linux-gpu`, branch `mars-gpu`, commit `54bf2745aa8e` (`v7.3-rc5` plus the two domibel merges; see `sources.md`). Kernel release string: `7.3.0-rc5-00040-g54bf2745aa8e`.

## Build

```
~/mars-gpu/scripts/build-kernel.sh      # JOBS=80 by default
CLEAN=1 ~/mars-gpu/scripts/build-kernel.sh   # wipe the build dir first
CLEAN=1 CONFIG_ONLY=1 ~/mars-gpu/scripts/build-kernel.sh   # configure and verify only
```

Log: `~/mars-gpu/logs/build-kernel.log`. A build takes about 3 to 11 minutes with 80 jobs. There are no compiler errors or warnings; the only warnings are perl locale noise.

The script:

1. Runs `defconfig` in `~/mars-gpu/build/linux-gpu/` (out-of-tree).
1. Merges every `~/mars-gpu/scripts/*.fragment` (`10-gpu.fragment`, `20-containers.fragment`) with `scripts/kconfig/merge_config.sh`, then `olddefconfig`.
1. **Fails if any line of the fragment did not survive `olddefconfig`.** This is how we caught the problems below.
1. Builds `Image modules dtbs` with `ARCH=riscv CROSS_COMPILE="ccache riscv64-linux-gnu-"`.
1. Installs into `~/mars-gpu/out/`.

## GPU/display fragment (`scripts/10-gpu.fragment`)

The reference repo's 11 options:

```
CONFIG_DRM_POWERVR=m
CONFIG_DRM_VERISILICON_DC=y
CONFIG_CMA=y
CONFIG_ERRATA_SIFIVE=y
CONFIG_ERRATA_SIFIVE_XPBMTUC=y
CONFIG_STARFIVE_STARLINK_CACHE=y
CONFIG_RISCV_NONSTANDARD_CACHE_OPS=y
CONFIG_DRM_STARFIVE_JH7110_INNO_HDMI=y
CONFIG_PHY_STARFIVE_JH7110_INNO_HDMI=y
CONFIG_SOC_STARFIVE_JH7110_VOUT_SUBSYSTEM=y
CONFIG_SOC_STARFIVE_JH7110_HDMI_SUBSYSTEM=y
```

Our additions. The reference repo starts from a Debian config, which already has these. Our start is the upstream `defconfig`, which does not:

```
CONFIG_DRM=y
CONFIG_DRM_DISPLAY_CONNECTOR=y
CONFIG_CLK_STARFIVE_JH7110_VOUT=y
CONFIG_DRM_FBDEV_EMULATION=y
CONFIG_FB=y
CONFIG_FRAMEBUFFER_CONSOLE=y
CONFIG_DMA_CMA=y
CONFIG_CMA_SIZE_MBYTES=128
```

Why each is needed:

- `DRM=y`: upstream `defconfig` has `DRM=m`, and Kconfig silently reduced `DRM_VERISILICON_DC=y` and `DRM_STARFIVE_JH7110_INNO_HDMI=y` to `m` (the first run of the script reported both as not applied). The display drivers are meant to be built in so the framebuffer console appears early, so `DRM` must be built in too.
- `DRM_DISPLAY_CONNECTOR=y`: the devicetree has an `hdmi-connector` node, and this driver binds it. It was not set in `defconfig`.
- `CLK_STARFIVE_JH7110_VOUT=y`: the display clock/reset controller (`voutcrg`). It has to be built in for the built-in display drivers to probe early.
- `DMA_CMA=y`, `CMA_SIZE_MBYTES=128`: `defconfig` had `CMA=y` but not `DMA_CMA`, which the devicetree's `linux,cma` pool needs in order to be used at all. The 128 MiB default matches the reference's Debian config.
- `DRM_FBDEV_EMULATION`, `FB`, `FRAMEBUFFER_CONSOLE`: console on HDMI, as in the reference.

This is our best reading of the reference setup. It is not yet tested on hardware. Update this list if boot testing shows something else is needed.

## Containers and snap fragment (`scripts/20-containers.fragment`)

Added for running LXD system containers and snapd. A clean rebuild with it succeeded (2026-10-07). The script confirmed every line landed in the final `.config`. Contents, by purpose:

- **Squashfs for snaps:** `SQUASHFS=y` with `XATTR`, `FILE_DIRECT`, per-CPU parallel decompression (`SQUASHFS_COMPILE_DECOMP_MULTI_PERCPU`), and every compressor (`ZLIB`, `LZ4`, `LZO`, `XZ`, `ZSTD`); `BLK_DEV_LOOP=y`, `4K_DEVBLK_SIZE`.
- **Filesystems and devices:** `FUSE_FS=y`, `CUSE=m`, `OVERLAY_FS=y` (was `m`), `FANOTIFY=y`, `TUN=m`, `BINFMT_MISC=m`.
- **Cgroups and misc:** `PSI`, `CGROUP_MISC`, `CGROUP_RDMA`, `CGROUP_NET_PRIO`, `USERFAULTFD`, `BPF_JIT`. The `defconfig` already had namespaces (user, pid, net, uts, ipc), cgroups (memory, pids, freezer, device, cpuset, bpf, blkio, CFS bandwidth), seccomp filter, audit, keys, veth, bridge, macvlan, ipvlan, vxlan, POSIX mqueue, `FHANDLE` and checkpoint/restore.
- **AppArmor:** `SECURITY_APPARMOR` was already `y`, but `defconfig`'s `CONFIG_LSM` string did not include it, so it would never have been active. Now `CONFIG_LSM="landlock,lockdown,yama,integrity,apparmor,bpf"` and `DEFAULT_SECURITY_APPARMOR=y`.
- **Networking for LXD bridges:** nftables (`NF_TABLES`, `NF_NAT`, `NFT_CT`, `NFT_NAT`, `NFT_MASQ`, `NFT_REDIR`, `NFT_REJECT`, `NFT_LOG`, `NFT_LIMIT`, `NFT_COMPAT`, `NF_TABLES_INET`, `NF_TABLES_BRIDGE`), `NF_CT_NETLINK`, `IP_MULTIPLE_TABLES`, `IPV6_MULTIPLE_TABLES`, and the xtables targets and matches that `iptables-nft` needs (`NAT`, `MASQUERADE`, `CHECKSUM`, `CT`, `comment`, `mark`, `multiport`, `state`, `physdev`). Also the legacy iptables tables (`NETFILTER_XTABLES_LEGACY`, `IP_NF_IPTABLES_LEGACY`, `IP6_NF_IPTABLES_LEGACY` and the `filter`, `nat`, `mangle` and masquerade tables for IPv4 and IPv6), so legacy `iptables` also works.

Three fragment lines needed correcting during the first attempt, because the symbol names differ in 7.3: `NF_CONNTRACK_NETLINK` is `NF_CT_NETLINK`, `NFT_COUNTER` no longer exists (it is part of the nf_tables core), and the squashfs decompressor choice is `SQUASHFS_COMPILE_DECOMP_MULTI_PERCPU`.

Not enabled: vsock and virtio-fs, `KVM` guests (the JH7110's U74 cores have no hypervisor extension, so LXD virtual machines cannot work; system containers can), and `RT_GROUP_SCHED` (it would break cgroup v2 CPU control).

## Outputs (`~/mars-gpu/out/`)

| Path | Contents |
|---|---|
| `Image` | kernel, 29 MB |
| `modules/lib/modules/7.3.0-rc5-00040-g54bf2745aa8e/` | modules (29 MB, stripped), including `kernel/drivers/gpu/drm/imagination/powervr.ko` |
| `dtbs/starfive/jh7110-milkv-mars.dtb` | Mars devicetree (also the VF2 and other JH7110 DTBs) |
| `kernel.config` | final `.config` |
| `kernel.release` | release string |

The outputs above are from the latest clean build, which includes the containers fragment.

Built in the config: `DRM`, `DRM_VERISILICON_DC`, `DRM_DISPLAY_CONNECTOR`, `CLK_STARFIVE_JH7110_VOUT`, `DMA_CMA`. Only `powervr` is a module.

## Notes

- The reference config also sets `CMA_SIZE_MBYTES=128` while the devicetree reserves 512 MiB. See `dts-notes.md` (open item 3) on what `cma=` does.
- The build uses upstream `defconfig` as a base, not a distro config. Checked in `kernel.config`: the SD/eMMC controller (`MMC_DW_STARFIVE`), `EXT4_FS` and `VFAT_FS` are built in, so booting from SD or eMMC needs no initramfs. The Ethernet driver (`DWMAC_STARFIVE`, `STMMAC_ETH`) is a module, so an NFS root would need an initramfs, or those drivers switched to `=y`. Decide this in Phase 5/6 if we choose network boot.
- Not done in this phase: firmware and rootfs (Phase 6), bootloader (Phase 5), QEMU test (Phase 4).
