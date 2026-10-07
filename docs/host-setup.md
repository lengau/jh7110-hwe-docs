# Host setup: ubhejane (10.13.1.3)

Host: Ubuntu 24.04.5, x86_64, 80 cores, 1 TiB RAM. Login: `ssh ubuntu@10.13.1.3` (key auth, passwordless sudo). Done 2026-10-06.

## Packages installed

`gcc-riscv64-linux-gnu` and `g++-riscv64-linux-gnu` (13.3.0), `device-tree-compiler` (1.7.0), `bison`, `flex`, `libssl-dev`, `libelf-dev`, `libncurses-dev`, `libgnutls28-dev`, `uuid-dev`, `bc`, `u-boot-tools`, `swig`, `python3-dev`, `python3-setuptools`, `python3-pyelftools`, `ccache` (4.9.1), `mmdebstrap` (1.4.3), `qemu-user-static` and `binfmt-support`, `debian-archive-keyring`, `ubuntu-keyring`, `parted`, `dosfstools`, `e2fsprogs`, `kpartx`, `tio` (2.7), `tftpd-hpa` and `nfs-kernel-server`.

`libgnutls28-dev` and `uuid-dev` are additions to the plan, needed for the U-Boot build.

## Services

`tftpd-hpa` and `nfs-kernel-server` are installed but **stopped and disabled**. Enable them when we pick a netboot method.

## Checks

- `qemu-riscv64` binfmt handler is registered (`/proc/sys/fs/binfmt_misc/qemu-riscv64`).
- ccache size limit set to 50 GB (`ccache -M 50G`). Use it with `CROSS_COMPILE="ccache riscv64-linux-gnu-"`.
- No serial adapter is attached to this host.

## Known cosmetic issue

apt and perl print locale warnings over SSH because the client sends `LC_*=en_CA.UTF-8`, which the host does not have. They are harmless.

## Layout

All work lives under `~/mars-gpu/`: `repos/` (bare clones), `worktrees/`, `build/`, `out/`, `patches/`, `logs/`, `scripts/`. See `plan.md` for what each directory is for and `sources.md` for what is checked out.

## Added for the QEMU phase (2026-10-07)

- Packages: `qemu-system-misc` (8.2.2, distro), `squashfs-tools`, `lz4`, `lzop`, `xz-utils`, `zstd`, `cpio`, `cloud-image-utils`, `u-boot-qemu` (2025.10, provides `/usr/lib/u-boot/qemu-riscv64_smode/uboot.elf`).
- QEMU build dependencies: `ninja-build`, `libglib2.0-dev`, `libpixman-1-dev`, `libslirp-dev`, `python3-venv`, `python3-pip`, `libfdt-dev`, `zlib1g-dev`, `libzstd-dev`, `libattr1-dev`, `libcap-ng-dev`.
- **QEMU 11.1.2** is built from upstream source and installed in `/opt/qemu-11` (binaries `qemu-system-riscv64` and `qemu-riscv64`). It is not on the default `PATH`; the scripts reference `/opt/qemu-11/bin/` explicitly. The distro 8.2.2 is untouched. No PPA carries QEMU 11 for 24.04 (checked 2026-10-07). Noble has 8.2.2, resolute (26.04) has 10.2.1, and 26.10 has 11.0.3.
  - Configure line: `--prefix=/opt/qemu-11 --target-list=riscv64-softmmu,riscv64-linux-user --enable-slirp --disable-docs --disable-gtk --disable-sdl --disable-opengl --disable-vnc --disable-spice --disable-guest-agent --disable-tools --disable-werror`. Logs: `~/mars-gpu/logs/qemu-{configure,build,install}.log`. The build takes about 35 seconds.
