# JH7110 HWE Stack Planning & Build Guide

This guide outlines the process for creating a Hardware Enablement (HWE) stack for JH7110-based boards (e.g., VisionFive 2, Milk-V Mars) targeting Ubuntu 24.04 (Noble).

## Goals

Provide a rolling set of updated kernels and graphics components to enable experimental GPU and display features on Noble without requiring a full OS upgrade.

## PPA Structure

- **Development:** `ppa:lengau/jh7110-devel` (Daily/experimental builds, testing)
- **Stable:** `ppa:lengau/jh7110` (Verified releases)

## Stack Components

### 1. Kernel & DTB

- **Base:** Ubuntu 24.04 Noble.
- **Source:** Mainline kernel with patches from `domibel/linux` (specifically `powervr_on_jh7110_visionfive2` and `jh7110_dc8200_hdmi` branches).
- **Configuration:** Must include `DRM_POWERVR=m`, `DRM_VERISILICON_DC=y`, and sufficient `CMA` (128MB+).
- **Packaging:** Upload as a source package to the PPA. Enable `riscv64` builds in PPA settings.

### 2. Firmware

- **Required:** `powervr/rogue_36.50.54.182_v1.fw`.
- **Packaging:** Create a small firmware package (or a custom `linux-firmware` extension) to ensure the blob is placed in `/lib/firmware/powervr/`.

### 3. Graphics Stack (Mesa)

- **Requirement:** Mesa with PowerVR Vulkan support (e.g., 26.2.x).
- **Build:** Since stock Noble Mesa lacks the PowerVR ICD, this must be built from source and packaged.
- **Dependencies:** Ensure `libdrm` is updated if required by the newer Mesa version.

### 4. Rolling Metapackages

To make the stack "HWE", use metapackages that depend on the current stable versions of the above:

- `linux-generic-hwe-jh7110` $\\rightarrow$ depends on `linux-image-unsigned-[version]-generic`
- `mesa-hwe-jh7110` $\\rightarrow$ depends on `mesa-vulkan-drivers-[version]`

## Workflow

1. **Build & Test (Devel PPA):**
   - Upload source packages to `jh7110-devel`.
   - Verify on hardware: Boot $\\rightarrow$ `modprobe powervr` $\\rightarrow$ `vulkaninfo` $\\rightarrow$ `vkmark`.
1. **Release (Stable PPA):**
   - Once verified, copy the packages (or re-upload) to `jh7110`.
   - Update the rolling metapackage to point to the new versions.
1. **Verification:**
   - Ensure the metapackage allows users to stay on the HWE track via `apt upgrade`.
