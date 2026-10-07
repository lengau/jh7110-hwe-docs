# Root filesystem (Phase 6)

Status 2026-10-07: the Mesa build is done; the image assembly still has open items (below).

## OS choice: Ubuntu 24.04.5 (noble), riscv64

- **Ubuntu 26.04 (resolute) is not usable on the Mars.** Its riscv64 images are built with an RVA23 baseline, and the JH7110's U74 cores are not RVA23-compatible (no hypervisor, no zicond/zicboz and other RVA23 bits; see `kernel.md` and the plan risks). Any 26.04-era riscv64 binaries (and packages from the archive) are built to the newer baseline and can fail to run. Noble is the last release built for the baseline the JH7110 meets.
- Consequences accepted: noble's archive `mesa-vulkan-drivers` is 25.2.8 and has **no PowerVR driver** (`libvulkan_powervr_mesa.so` is absent). We build Mesa ourselves (below).
- The base image is the 24.04.5 preinstalled server image already in `sources.md`, assembled by `scripts/ubuntu-image.sh`.

## Mesa 26.2.3 (matches the reference)

- Upstream tag `mesa-26.2.3` (`31e9a6b2e95e30d84bf3177d1f497d063e59b6b2`), the same version the reference repo used. It contains `bxe-4-32` device info (`36.50.54.182`). Checked: 25.2.8 does not.

- Built inside a **riscv64 noble chroot** under `qemu-user` (`build/mesa-chroot`, made with `mmdebstrap --variant=buildd`, build deps installed inside). Script: `scripts/build-mesa.sh` with `configure` and `build` steps; logs `logs/mesa-configure.log`, `logs/mesa-build.log`.

- The build takes about 3 minutes to configure and 12 minutes to build, single-threaded from the host's point of view (emulated `ninja` runs all 80 jobs, but qemu-user makes CPU-heavy compiles slow; ccache helps on rebuilds).

- Configure line (from the script):

  ```
  --prefix=/usr/local --libdir=lib/riscv64-linux-gnu --buildtype=release -Db_ndebug=true
  -Dvulkan-drivers=imagination,swrast -Dgallium-drivers=zink,llvmpipe
  -Dplatforms=wayland,x11 -Degl=enabled -Dgbm=enabled -Dgles2=enabled -Dglx=dri
  -Dllvm=enabled -Dgallium-va=disabled -Dvalgrind=disabled -Dbuild-tests=false
  ```

  So we get: the PowerVR Vulkan driver, swrast (llvmpipe), the zink gallium driver (OpenGL ES through Vulkan), EGL/GBM for Wayland and X, with `libvulkan_lvp` (llvmpipe) as the CPU fallback for vulkaninfo.

- Output: `build/mesa-dest/usr/local/` (mounted as `out/mesa/` while building), containing:

  - `lib/riscv64-linux-gnu/libvulkan_powervr_mesa.so` (strings confirm `BXE-4-32`, `Mesa 26.2.3 (git-31e9a6b2e9)`)
  - `lib/riscv64-linux-gnu/libvulkan_lvp.so`
  - `share/vulkan/icd.d/powervr_mesa_icd.riscv64.json`
  - `lib/riscv64-linux-gnu/dri/` (zink, swrast, kms_swrast and the other DRI modules), plus EGL, GBM and GLX libraries

- It installs to `/usr/local`, so it will be used ahead of the distro Mesa once copied into the image.

### Chroot gotchas (what the build needed)

- `python3-mako` etc. from noble; **meson from pip** (noble's 1.3 is < 1.4, required by Mesa 26).
- The **PowerVR Vulkan driver requires CLC**: `LLVMSPIRVLib` (SPIRV-LLVM-Translator), so `libllvmspirvlib-18-dev` / `llvm-spirv-18`, and `clang-18`, `libclang-18-dev`, `libclang-cpp18-dev`, `rustc`, `cargo`, `bindgen`, `cbindgen` (imagination uses Rust for its offline compiler).
- Noble's SPIRV-Tools is 2023.6.1 but Mesa needs >= 2024.1, so **SPIRV-Tools 2024.4 (`v2024.4`) and its matching SPIRV-Headers** (revision from SPIRV-Tools' `DEPS`: `3f17b2af...`) are built from source into `/usr/local` inside the chroot. Without the matching headers the SPIRV-Tools build fails with missing `spv::Op` members.
- Also needed: `libx11-xcb-dev`, `libdisplay-info-dev`, `glslang-tools`, `spirv-tools`, the usual xcb/wayland -dev packages. Full package list is in `scripts/build-mesa.sh` history (`logs/mesa-chroot.log`).
- The chroot's build tree, source and install tree are bind mounts from `build/mesa`, `worktrees/mesa` and `build/mesa-dest`, so the outputs live outside the chroot and survive a chroot recreate.

### Open question: llvmpipe for llvmpipe?

`-Dgallium-drivers=zink,llvmpipe` was chosen so that `glmark2-es2-wayland` has a fallback if the PowerVR stack misbehaves, and so `es2_info`/glxgears-style checks work even before the GPU comes up. If disk space or build time matters it can be dropped to zink only.

## GPU firmware

- The reference pins `rogue_36.50.54.182_v1.fw` from `imagination/linux-firmware` at `8a58f818` (the `firmware` worktree; sha256 recorded in `sources.md`).
- Our kernel's `CONFIG_FW_LOADER` will look for it at `/lib/firmware/powervr/rogue_36.50.54.182_v1.fw`.
- Ubuntu noble ships a different PowerVR firmware (`rogue_36.53.104.796_v1.fw.zst` is present in the Ubuntu 26.04 image's `/usr/lib/firmware/powervr/`), i.e. the distros are moving on. **We install the pinned 36.50.54.182 firmware** (uncompressed, from the firmware worktree) because the reference results and Mesa's device info are matched to that version. If the firmware fails to load we revisit.

## Not done yet (left open for Phase 7 image assembly)

- Installing `out/mesa` into the image and pointing the Vulkan loader at the ICD JSONs (they are installed to the standard `/usr/local/share/vulkan/icd.d/`, which `Vulkan-Loader` finds automatically).
- Installing the pinned GPU firmware.
- The real rootfs image for the board: which media (SD vs the `+jh7110` Ubuntu image as a base), the `extlinux.conf` from `boot.md`, and whether `cma=` goes on the command line.
- Adding the benchmark and compositor packages (`vulkan-tools`, `vkmark`, `glmark2-es2-wayland`, `labwc`, `weston`, `sway`) from the archive — they are distro packages and install normally on noble.
- Re-running the QEMU test against this finished rootfs, plus `snapd` and `lxd` on it (Phase 7's software checks). Note: `snapd` works on this kernel already (see `qemu-smoke.md`); what remains is confirming the GPU-usable Mesa on top.

## Chroot maintenance

```
sudo mmdebstrap ... noble build/mesa-chroot http://ports.ubuntu.com/ubuntu-ports   # recreate (see log for the package list)
scripts/build-mesa.sh configure && scripts/build-mesa.sh build
```

The script bind-mounts `proc`/`dev`/`src`/`build`/`dest`/`out` and unmounts them on exit; if it dies hard, `sudo umount -R` the chroot manually.
