# QEMU kernel smoke test (Phase 4)

Run 2026-10-07 on ubhejane against kernel `7.3.0-rc5-00040-g54bf2745aa8e` (the build with the GPU and containers fragments). **Result: 67 passed, 0 failed**, in about 12 seconds of wall time.

## Scope

QEMU has no JH7110 machine, so this uses the generic `virt` machine. It checks that the kernel boots and that the container and snap options work. It does **not** exercise the Mars devicetree, the GPU, HDMI, the DC8200 or any JH7110-specific driver. Those need the board.

## Setup (all on ubhejane)

- Installed: `qemu-system-misc` (QEMU 8.2.2), `squashfs-tools`, `lz4`, `lzop`, `xz-utils`, `zstd`, `cpio`.
- Guest userland: Ubuntu 24.04 (noble) riscv64, extracted with `mmdebstrap --variant=extract` (`busybox-static`, `kmod`, `nftables`, `util-linux`, `iproute2`). It is packed as an initramfs together with `out/modules` and five test squashfs images.

Scripts, in `~/mars-gpu/scripts/`:

| Script | Purpose |
|---|---|
| `qemu-initramfs.sh` | builds `out/qemu/initramfs.cpio.gz` (rootfs extract + modules + squashfs test images + init) |
| `qemu-init.sh` | the guest's `/init`; runs every check, prints `PASS:`/`FAIL:` lines and `SMOKE RESULT:`, powers off |
| `qemu-smoke.sh` | runs QEMU, writes `~/mars-gpu/logs/qemu-smoke.log`, exits non-zero on any failure or timeout (900 s) |

Rootfs tarball: `~/mars-gpu/out/qemu/rootfs.tar`. Rebuild the initramfs after every kernel build, because it embeds the modules.

Command line used by `qemu-smoke.sh`:

```
qemu-system-riscv64 -M virt -cpu rv64 -smp 4 -m 4G -nographic -no-reboot \
  -kernel out/Image -initrd out/qemu/initramfs.cpio.gz \
  -append "console=ttyS0 earlycon panic=-1 loglevel=6" \
  -netdev user,id=n0 -device virtio-net-pci,netdev=n0
```

## What passed

- **Boot:** reaches init, with no Oops, BUG, WARNING or panic in `dmesg`. `/proc/meminfo` shows `CmaTotal` 128 MiB (the `CMA_SIZE_MBYTES` default; the `virt` devicetree has no 512 MiB pool).
- **Modules:** `depmod` works, and these load with `modprobe`: `veth`, `bridge`, `tun`, `vxlan`, `macvlan`, `ipvlan`, `cuse`, `dummy`, `nf_conntrack`, `nf_tables`, `nf_nat`, `nft_ct`, `nft_nat`, `nft_masq`, `nft_compat`, `nf_conntrack_netlink`, `ip_tables`, `ip6_tables`, `iptable_nat`, `iptable_filter`, `xt_MASQUERADE`, `xt_conntrack`, `xt_comment`, `xt_mark`, `xt_multiport`, `xt_state`, `xt_physdev`, `br_netfilter`, `8021q`.
- **Namespaces:** user, pid, net, mount, uts, ipc, cgroup and time, plus a nested user+pid+net+mount combination.
- **Cgroup v2:** controllers present: `cpuset cpu io memory hugetlb pids rdma misc`.
- **Networking:** a veth pair moved into a netns; a bridge with a veth enslaved; a dummy interface; `/dev/net/tun`; the virtio-net device.
- **Filesystems:** overlayfs mounts; FUSE is present with `/dev/fuse`; squashfs images made with gzip, lzo, lz4, xz and zstd (each with a 3 MB incompressible file) loop-mount and read back with the right content and checksum.
- **Security:** AppArmor is in the LSM list and enabled, and `/sys/kernel/security/apparmor` exists. Seccomp and seccomp filters are reported by `/proc/self/status`.
- **nftables:** a `nat` table with a postrouting masquerade chain, and an `inet` filter table with a conntrack rule.

## Things found along the way

- The default module autoload path `/sbin/modprobe` does not exist in the guest, so nft's automatic loading failed until `/proc/sys/kernel/modprobe` was set. This is a guest-image detail, not a kernel problem. The real rootfs needs a working `/sbin/modprobe` (Ubuntu and Debian have one).
- busybox `ash` runs its own applets in preference to files in `PATH`, so the init script calls the real `unshare`, `ip` and `modprobe` by full path.

## Not covered

- **Active LSMs:** the running LSM list is only `capability,apparmor`. `defconfig` does not build Yama, Lockdown, Landlock or Integrity in (`SECURITY_YAMA` is off). Neither snapd nor LXD requires them, but Ubuntu normally has Yama (`ptrace_scope`), so we can add `CONFIG_SECURITY_YAMA=y` (and the others) later if we want parity.
- **AppArmor profile loading:** the interface exists, but we did not load profiles. That needs the `apparmor` userspace tools in the real rootfs.
- **Squashfs xattrs:** the images were built with xattrs, but the guest does not read them back (busybox lacks `getfattr`).
- **snapd and LXD themselves:** not run. They need the full rootfs (Phase 6). Re-run the kernel in QEMU against it during Phase 7.
- **Hardware:** everything JH7110-specific, including the devicetree, GPU, HDMI and cache-ops/errata alternatives.

---

# Ubuntu 24.04 image under QEMU 11, with our kernel (Phase 4 follow-up)

Done 2026-10-07. The minimal initramfs test above shows the kernel boots and the options work. This second test boots the same kernel inside a stock **Ubuntu 24.04.5 riscv64** image, and runs the real snapd and LXD. **Result: it works.**

## Tooling

- **QEMU 11.1.2**, built from source into `/opt/qemu-11` (see `host-setup.md`). The 67-check initramfs test passes on it too (`QEMU=/opt/qemu-11/bin/qemu-system-riscv64 scripts/qemu-smoke.sh`).
- Base image: `ubuntu-24.04.5-preinstalled-server-riscv64.img` (see `sources.md`). It boots UEFI: U-Boot, then GRUB, then the kernel. Its stock kernel is `7.0.0-31-generic`.

Scripts, in `~/mars-gpu/scripts/`:

| Script | Purpose |
|---|---|
| `ubuntu-image.sh` | copies the pristine base image to `images/work.img`, grows it to 24 GB, installs our kernel (`/boot/vmlinuz-REL`, `config-REL`, `System.map-REL`, modules under `/usr/lib/modules/REL`), runs `depmod`, `update-initramfs` and `update-grub` in a chroot, and writes a cloud-init seed to the `CIDATA` partition. Re-run it after every kernel build. Always starts from the pristine copy. |
| `ubuntu-qemu.sh` | boots `work.img` under QEMU 11: `-M virt -cpu rv64 -smp 8 -m 8G`, `uboot.elf` as the firmware, virtio disk, user-mode networking with `localhost:2222` forwarded to the guest's ssh, virtio-rng, serial to `logs/ubuntu-qemu.log`, QEMU monitor on `images/monitor.sock`, daemonized. |
| `vmssh.sh` | ssh into the running guest as `ubuntu` with the generated key (`images/qemu_key`). |

The cloud-init seed sets hostname `mars-qemu`, user `ubuntu` with password `ubuntu`, and the key from `images/qemu_key.pub`. GRUB picks the newest kernel (ours, `7.3.0-rc5-...`) by default. The stock 7.0.0-31 entry is still there.

Stop the guest with `scripts/vmssh.sh "sudo shutdown -h now"`. It powers off cleanly (`reboot: Power down`, and the QEMU process exits). **The VM is currently stopped.**

## Results

| Check | Result |
|---|---|
| Boots our kernel via U-Boot, then GRUB, with an initramfs from `update-initramfs` | works. `uname -r` is `7.3.0-rc5-00040-g54bf2745aa8e`, and cloud-init reports `done` |
| ssh and networking | works. Slirp gives HTTPS and DNS; ICMP ping does not work in slirp, as usual |
| Active LSMs | `capability,apparmor`. `aa-status`: 117 profiles loaded, 23 in enforce mode |
| `snapd` 2.76.3 | works. It refreshed itself to 2.77.1 and installed `core24` |
| `snap install lxd` | works: 5.21.8 LTS (`5.21/stable`), taking about 3 minutes under emulation. Squashfs snap mounts work |
| `snap install hello-world` | fails with "cannot install snap base core". That snap uses the `core` (16) base, which has no riscv64 build. It is not a kernel problem. |
| `lxd init --auto` | works: `dir` storage pool, `lxdbr0` bridge (IPv4 and IPv6, using nftables) |
| `lxc launch ubuntu:24.04 c1` | works. It picked the riscv64 image automatically, took about 90 seconds, and the container reached `running` and got an IPv4 and IPv6 address on `lxdbr0`. Inside it: Ubuntu 24.04.5, the host kernel, DNS, HTTPS to the Ubuntu archive, `apt-get update`, systemd `running`, and a 1,000,000,000-ID user namespace map. Seccomp is on, in filter mode. |

Cleanup: the test container `c1` was deleted. `snapd`, `lxd` and `core24` are installed in `work.img`. Re-running `ubuntu-image.sh` gives a fresh image.

## Caveats

- `dmesg` has a handful of AppArmor `DENIED` audit lines: `snap-confine` mount with `rw, rbind`, and the `snap.lxd.lxd` and `snap.lxd.lxc` profiles being denied `dac_override` and `dac_read_search`. Everything we tried worked, so I think they are the usual snap profile noise. We have not compared against the stock `7.0.0-31` kernel. Easy to do: pick it in GRUB and repeat the snap and LXD steps.
- The kernel log has no BUG, WARNING or Oops lines.
- Not covered: LXD virtual machines (not possible on the JH7110, see `kernel.md`), the GPU and HDMI stack, and anything specific to the Mars hardware.
- The Ubuntu image is Ubuntu. For Phase 6 this settles that Ubuntu riscv64 is a workable root filesystem for snapd and LXD with this kernel.
