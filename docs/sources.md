# Sources and pinned commits

All on ubhejane under `~/mars-gpu/`. Bare clones are in `repos/`, checkouts in `worktrees/`. Fetched 2026-10-06.

| Worktree | Repo (bare clone) | Ref | Commit |
|---|---|---|---|
| `linux-base` | `repos/linux.git` (torvalds/linux) | tag `v7.3-rc5`, detached | `72d3fcf802c45d00b300f25b848a93c3a2bd7c7e` |
| `linux-gpu` | `repos/linux.git` | branch `mars-gpu` | `54bf2745aa8eebabe8efed662f7cb334135a0bdb` |
| `reference` | `repos/reference.git` (domibel/visionfive2_gpu_bringup) | `main`, detached | `3735836dd451674eafb34e5ec3fb68ffc815a224` |
| `firmware` | `repos/firmware.git` (gitlab.freedesktop.org/imagination/linux-firmware) | detached | `8a58f81883f7be458daa34e418cc4079f995b279` |
| `u-boot` | `repos/u-boot.git` (u-boot/u-boot) | tag `v2026.07`, detached | `ece349ade2973e220f524ce59e59711cc919263f` |
| `qemu` | `repos/qemu.git` (gitlab.com/qemu-project/qemu) | tag `v11.1.2`, detached; built in `build/qemu`, installed to `/opt/qemu-11` | `4fc49f46dc95d4a27de2509e7fceb2931e91faeb` |
| `opensbi` | `repos/opensbi.git` (riscv-software-src/opensbi) | tag `v1.9`, detached | `cbf9f6734dd85a982c63e3cb5db7ffe09da839ca` |

`linux.git` has a second remote, `domibel` (`https://github.com/domibel/linux`), with these branches fetched:

| Branch | Commit |
|---|---|
| `domibel/powervr_on_jh7110_visionfive2_v7.3-rc5` | `77f50b34b3e3308abf520e993e12d65be7b07c5c` |
| `domibel/jh7110_dc8200_hdmi_v7.3-rc5` | `95e43ff558b73d9fbd27b3ecda6024e414eea9e6` |
| `domibel/powervr_on_jh7110_visionfive2_v7.2` | fallback, fetched |
| `domibel/jh7110_dc8200_hdmi_v7.2` | fallback, fetched |

## `mars-gpu` branch

Created from `v7.3-rc5` with two `--no-ff` merges, in this order. Both were clean (no conflicts):

1. `domibel/powervr_on_jh7110_visionfive2_v7.3-rc5` (merge `615b70ff4aef`)
2. `domibel/jh7110_dc8200_hdmi_v7.3-rc5` (merge `54bf2745aa8e`)

The merges use a local git identity set in `linux.git` (`Mars GPU bring-up <ubuntu@ubhejane>`).

## Notes

- Mainline `v7.3-rc5` already contains `jh7110-milkv-mars.dts` (and the `marscm` variants) in `arch/riscv/boot/dts/starfive/`.
- The firmware commit contains `powervr/rogue_36.50.54.182_v1.fw` (131072 bytes, sha256 `59127d3a02e48876310a520b7dc355f7430d3748d2940caaa33f39237d517abb`).
- The U-Boot tag is `v2026.07`, one of the two versions the reference repo says it tested. OpenSBI `v1.9` is the newest tag. U-Boot's doc requires OpenSBI v1.5 or newer (its example uses v1.7), so v1.9 should be fine; the build is in `boot.md`.
- Mesa is not cloned. We will do that only if the riscv64 distro's Mesa is not 26.2.x.

## Other downloads

| Item | Location on ubhejane | Source / checksum |
|---|---|---|
| Ubuntu 24.04.5 preinstalled server image, riscv64 (QEMU `virt`) | `~/mars-gpu/images/ubuntu-24.04.5-preinstalled-server-riscv64.img(.xz)` | `https://cdimage.ubuntu.com/releases/24.04/release/`; sha256 `1cbd4b187f33356107daa637a5588d8f8a1744c7a012189e9c5e6d0b177825c4` (verified) |

The same release directory has a `+jh7110` image for the Mars-class boards (`ubuntu-24.04.5-preinstalled-server-riscv64+jh7110.img.xz`, sha256 `f1e6ebe69948dabde72642b70b8f195f3844915288d78e0a600f8b5e3a5ffd6f`). It is not downloaded yet; it is the obvious candidate for the Phase 8 baseline boot.
