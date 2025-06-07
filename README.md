# OnePlus SukiSU Ultra Kernel Build Workflow (Ace2Pro Default)

This GitHub Actions workflow is designed to automatically compile a OnePlus kernel with SukiSU Ultra (KernelSU) integration and optional features like SUSFS, VFS hooks, KPM, and ZRAM enhancements. The defaults are pre-configured for the OnePlus Ace 2 Pro (Snapdragon 8 Gen 2, codename *kalama*), and you can override inputs to build for other devices. The workflow performs the following high-level steps:

* Initializes the OnePlus kernel repository (using the specified branch and device manifest) and removes any `-dirty` suffix from version info.
* Applies the KernelSU (SukiSU Ultra) setup and relevant patches (SUSFS, hide-stuff patch, VFS hooks, optional ZRAM/LZ4KD patches).
* Adjusts the kernel configuration (e.g. enabling `CONFIG_KSU`, `CONFIG_KPM`, SUSFS options, and optional ZRAM/LZ4).
* Builds the kernel using either the GKI method or a fallback Oplus build script, depending on the CPU.
* Prepares an AnyKernel3 directory, copies the built `Image`, and optionally patches it with `patch_linux` if KPM is enabled.
* Downloads the latest SUSFS module (from CI or Release) and packages the results.

## Workflow Configuration

Below is the modified `build.yml` workflow configuration. It defaults to the OnePlus Ace 2 Pro settings (CPU `sm8550`, manifest `oneplus_ace2pro_v`, codename `kalama`, Android 13, kernel 5.15, GKI build, SUSFS CI download enabled, VFS and KPM enabled, ZRAM disabled). You can customize the `workflow_dispatch` inputs as needed.

```yaml
name: Build OnePlus_SukiSU Ultra All
on:
  workflow_dispatch:
    inputs:
      CPU:
        type: choice
        description: "Branch (CPU model, e.g., sm8550 for Ace2Pro)"
        required: true
        default: sm8550
        options:
          - sm7550
          - sm7675
          - sm8450
          - sm8475
          - sm8550
          - sm8650
          - sm8750
      FEIL:
        type: choice
        description: "Device configuration file (profile)"
        required: true
        default: oneplus_ace2pro_v
        options:
          - oneplus_nord_ce4_v
          - oneplus_ace_3v_v
          - oneplus_nord_4_v
          - oneplus_10_pro_v
          - oneplus_10t_v
          - oneplus_11r_v
          - oneplus_ace2_v
          - oneplus_ace_pro_v
          - oneplus_11_v
          - oneplus_12r_v
          - oneplus_ace2pro_v
          - oneplus_ace3_v
          - oneplus_open_v
          - oneplus12_v
          - oneplus_13r
          - oneplus_ace3_pro_v
          - oneplus_ace5
          - oneplus_pad2_v
          - oneplus_13
          - oneplus_13t
          - oneplus_ace5_pro
      CPUD:
        type: choice
        description: "Processor codename (use 'kalama' for Ace2Pro)"
        required: true
        default: kalama
        options:
          - crow
          - waipio
          - kalama
          - pineapple
          - sun
      ANDROID_VERSION:
        type: choice
        description: "Android base version"
        required: true
        default: android13
        options:
          - android12
          - android13
          - android14
          - android15
      KERNEL_VERSION:
        type: choice
        description: "Kernel major version"
        required: true
        default: "5.15"
        options:
         - "5.10"
         - "5.15"
         - "6.1"
         - "6.6"
      BUILD_METHOD:
        type: choice
        description: "Build method (GKI or PERF)"
        required: true
        default: gki
        options:
          - gki
          - perf
      SUSFS_CI:
        type: boolean
        description: "Download SUSFS module using CI?"
        required: true
        default: true
      VFS:
        type: boolean
        description: "Enable manual hooks (VFS)?"
        required: true
        default: true
      KPM:
        type: boolean
        description: "Enable KPM?"
        required: true
        default: true
      ZRAM:
        type: boolean
        description: "Enable additional ZRAM algorithms?"
        required: true
        default: false

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Maximize build space
        uses: easimon/maximize-build-space@master
        with:
          root-reserve-mb: 8192
          temp-reserve-mb: 2048
          swap-size-mb: 8192
          remove-dotnet: 'true'
          remove-android: 'true'
          remove-haskell: 'true'
          remove-codeql: 'true'

      - name: Configure Git
        run: |
          git config --global user.name "Numbersf"
          git config --global user.email "263623064@qq.com"

      - name: Install dependencies
        run: |
          sudo apt update && sudo apt upgrade -y
          sudo apt install -y python3 git curl jq

      - name: Show selected inputs (debug)
        run: |
          echo "Selected CPU: ${{ github.event.inputs.CPU }}"
          echo "Selected FEIL: ${{ github.event.inputs.FEIL }}"
          echo "Selected CPUD: ${{ github.event.inputs.CPUD }}"
          echo "Selected ANDROID_VERSION: ${{ github.event.inputs.ANDROID_VERSION }}"
          echo "Selected KERNEL_VERSION: ${{ github.event.inputs.KERNEL_VERSION }}"
          echo "Selected BUILD_METHOD: ${{ github.event.inputs.BUILD_METHOD }}"
          echo "Selected SUSFS_CI: ${{ github.event.inputs.SUSFS_CI }}"
          echo "Selected ZRAM: ${{ github.event.inputs.ZRAM }}"
          echo "Selected VFS: ${{ github.event.inputs.VFS }}"
          echo "Selected KPM: ${{ github.event.inputs.KPM }}"

      - name: Install repo tool
        run: |
          curl https://storage.googleapis.com/git-repo-downloads/repo > ~/repo
          chmod a+x ~/repo
          sudo mv ~/repo /usr/local/bin/repo

      - name: Initialize repo and sync
        run: |
          mkdir kernel_workspace && cd kernel_workspace
          repo init -u https://github.com/OnePlusOSS/kernel_manifest.git -b refs/heads/oneplus/${{ github.event.inputs.CPU }} -m ${{ github.event.inputs.FEIL }}.xml --depth=1
          repo sync
          if [  -e kernel_platform/common/BUILD.bazel ]; then
            sed -i '/^[[:space:]]*"protected_exports_list"[[:space:]]*:[[:space:]]*"android\\/abi_gki_protected_exports_aarch64",$/d' kernel_platform/common/BUILD.bazel
          fi
          if [  -e kernel_platform/msm-kernel/BUILD.bazel ]; then
            sed -i '/^[[:space:]]*"protected_exports_list"[[:space:]]*:[[:space:]]*"android\\/abi_gki_protected_exports_aarch64",$/d' kernel_platform/msm-kernel/BUILD.bazel
          fi
          rm kernel_platform/common/android/abi_gki_protected_exports_* || echo "No protected exports!"
          rm kernel_platform/msm-kernel/android/abi_gki_protected_exports_* || echo "No protected exports!"

      - name: Force remove -dirty suffix
        run: |
          cd kernel_workspace/kernel_platform
          sed -i 's/ -dirty//g' common/scripts/setlocalversion
          sed -i 's/ -dirty//g' msm-kernel/scripts/setlocalversion
          sed -i 's/ -dirty//g' external/dtc/scripts/setlocalversion
          sed -i '$i res=$(echo "$res" | sed \\'s/-dirty//g' common/scripts/setlocalversion
          git add -A
          git commit -m "Force remove -dirty suffix from kernel version"

      - name: Add KernelSU-SukiSU Ultra
        run: |
          cd kernel_workspace/kernel_platform
          curl -LSs "https://raw.githubusercontent.com/SukiSU-Ultra/SukiSU-Ultra/main/kernel/setup.sh" | bash -s susfs-dev
          cd ./KernelSU
          KSU_VERSION=$(expr $(/usr/bin/git rev-list --count main) "+" 10606)
          echo "KSUVER=$KSU_VERSION" >> $GITHUB_ENV
          export KSU_VERSION=$KSU_VERSION
          sed -i "s/DKSU_VERSION=12800/DKSU_VERSION=${KSU_VERSION}/" kernel/Makefile

      - name: Apply SukiSU Ultra Patches
        run: |
          cd kernel_workspace
          git clone https://gitlab.com/simonpunk/susfs4ksu.git -b gki-${{ github.event.inputs.ANDROID_VERSION }}-${{ github.event.inputs.KERNEL_VERSION }}
          git clone https://github.com/ShirkNeko/SukiSU_patch.git
          cd kernel_platform/common
          cp ../../susfs4ksu/kernel_patches/50_add_susfs_in_gki-${{ github.event.inputs.ANDROID_VERSION }}-${{ github.event.inputs.KERNEL_VERSION }}.patch ./
          cp ../../susfs4ksu/kernel_patches/fs/* ./
          cp ../../susfs4ksu/kernel_patches/include/linux/* ./
          patch -p1 < 50_add_susfs_in_gki-${{ github.event.inputs.ANDROID_VERSION }}-${{ github.event.inputs.KERNEL_VERSION }}.patch || true
          if [ "${{ github.event.inputs.ZRAM }}" = "true" ]; then
            cp -r ../SukiSU_patch/other/zram/lz4k/include/linux/* ./
            cp -r ../SukiSU_patch/other/zram/lz4k/lib/* ./
            cp -r ../SukiSU_patch/other/zram/lz4k/crypto/* ./
            cp -r ../SukiSU_patch/other/zram/lz4k_oplus ./
          fi

      - name: Apply Hide-stuff Patch
        run: |
          cd kernel_workspace/kernel_platform/common
          cp ../../SukiSU_patch/69_hide_stuff.patch ./
          patch -p1 -F3 < 69_hide_stuff.patch

      - name: Apply VFS (hooks) Patch
        run: |
          cd kernel_workspace/kernel_platform/common
          if [ "${{ github.event.inputs.VFS }}" = "true" ]; then
            cp ../../SukiSU_patch/hooks/syscall_hooks.patch ./
            patch -p1 -F3 < syscall_hooks.patch
          fi

      - name: Apply LZ4KD Patch
        run: |
          cd kernel_workspace/kernel_platform/common
          if [ "${{ github.event.inputs.ZRAM }}" = "true" ]; then
            cp ../../SukiSU_patch/other/zram/zram_patch/${{ github.event.inputs.KERNEL_VERSION }}/lz4kd.patch ./
            patch -p1 -F3 < lz4kd.patch || true
          fi

      - name: Add kernel configuration options
        run: |
          cd kernel_workspace/kernel_platform
          CONFIG_FILE=./common/arch/arm64/configs/gki_defconfig
          echo "CONFIG_KSU=y" >> "$CONFIG_FILE"
          echo "CONFIG_KPM=y" >> "$CONFIG_FILE"
          if [ "${{ github.event.inputs.VFS }}" = "false" ]; then
            echo "CONFIG_KPROBES=y" >> "$CONFIG_FILE"
          fi
          if [ "${{ github.event.inputs.VFS }}" = "true" ]; then
            echo "CONFIG_KSU_SUSFS_SUS_SU=n" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_MANUAL_HOOK=y" >> "$CONFIG_FILE"
          fi
          if [ "${{ github.event.inputs.VFS }}" = "false" ]; then
            echo "CONFIG_KSU_SUSFS_SUS_SU=y" >> "$CONFIG_FILE"
          fi
          if [ "${{ github.event.inputs.VFS }}" = "true" ]; then
            echo "CONFIG_KSU_SUSFS=y" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_HAS_MAGIC_MOUNT=y" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_SUS_PATH=y" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_SUS_MOUNT=y" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_AUTO_ADD_SUS_KSU_DEFAULT_MOUNT=y" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_AUTO_ADD_SUS_BIND_MOUNT=y" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_SUS_KSTAT=y" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_SUS_OVERLAYFS=n" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_TRY_UMOUNT=y" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_AUTO_ADD_TRY_UMOUNT_FOR_BIND_MOUNT=y" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_SPOOF_UNAME=y" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_ENABLE_LOG=y" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_HIDE_KSU_SUSFS_SYMBOLS=y" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_SPOOF_CMDLINE_OR_BOOTCONFIG=y" >> "$CONFIG_FILE"
            echo "CONFIG_KSU_SUSFS_OPEN_REDIRECT=y" >> "$CONFIG_FILE"
          fi
          sed -i 's/check_defconfig//' ./common/build.config.gki

      - name: LZ4KD Configuration (ZRAM support)
        run: |
          cd kernel_workspace/kernel_platform
          if [ "${{ github.event.inputs.ZRAM }}" = "true" ]; then
            CONFIG_FILE=./common/arch/arm64/configs/gki_defconfig
            if [ "${{ github.event.inputs.KERNEL_VERSION }}" = "5.10" ]; then
              echo "CONFIG_ZSMALLOC=y" >> "$CONFIG_FILE"
              echo "CONFIG_ZRAM=y" >> "$CONFIG_FILE"
              echo "CONFIG_MODULE_SIG=n" >> "$CONFIG_FILE"
              echo "CONFIG_CRYPTO_LZO=y" >> "$CONFIG_FILE"
              echo "CONFIG_ZRAM_DEF_COMP_LZ4KD=y" >> "$CONFIG_FILE"
            fi
            if [ "${{ github.event.inputs.KERNEL_VERSION }}" != "6.6" ] && [ "${{ github.event.inputs.KERNEL_VERSION }}" != "5.10" ]; then
              sed -i 's/CONFIG_ZSMALLOC=m/CONFIG_ZSMALLOC=y/g' "$CONFIG_FILE"
              sed -i 's/CONFIG_ZRAM=m/CONFIG_ZRAM=y/g' "$CONFIG_FILE"
            fi
            if [ "${{ github.event.inputs.KERNEL_VERSION }}" = "6.6" ]; then
              echo "CONFIG_ZSMALLOC=y" >> "$CONFIG_FILE"
              sed -i 's/CONFIG_ZRAM=m/CONFIG_ZRAM=y/g' "$CONFIG_FILE"
            fi
            if [ "${{ github.event.inputs.ANDROID_VERSION }}" = "android14" ] || [ "${{ github.event.inputs.ANDROID_VERSION }}" = "android15" ]; then
              if [ -e ./common/modules.bzl ]; then
                sed -i 's/"drivers\\/block\\/zram\\/zram\\.ko",//g; s/"mm\\/zsmalloc\\.ko",//g' "./common/modules.bzl"
              fi
              if [ -e ./msm-kernel/modules.bzl ]; then
                sed -i 's/"drivers\\/block\\/zram\\/zram\\.ko",//g; s/"mm\\/zsmalloc\\.ko",//g' "./msm-kernel/modules.bzl"
                echo "CONFIG_ZSMALLOC=y" >> "msm-kernel/arch/arm64/configs/${{ github.event.inputs.CPUD }}-GKI.config"
                echo "CONFIG_ZRAM=y" >> "msm-kernel/arch/arm64/configs/${{ github.event.inputs.CPUD }}-GKI.config"
              fi
              echo "CONFIG_MODULE_SIG_FORCE=n" >> "$CONFIG_FILE"
            fi
            if grep -q "CONFIG_ZSMALLOC=y" "$CONFIG_FILE" && grep -q "CONFIG_ZRAM=y" "$CONFIG_FILE"; then
              echo "CONFIG_CRYPTO_LZ4HC=y" >> "$CONFIG_FILE"
              echo "CONFIG_CRYPTO_LZ4K=y" >> "$CONFIG_FILE"
              echo "CONFIG_CRYPTO_LZ4KD=y" >> "$CONFIG_FILE"
              echo "CONFIG_CRYPTO_842=y" >> "$CONFIG_FILE"
              echo "CONFIG_ZRAM_WRITEBACK=y" >> "$CONFIG_FILE"
            fi
          fi

      - name: Build kernel (GKI)
        if: ${{ github.event.inputs.CPU == 'sm8650' || github.event.inputs.CPU == 'sm7675' }}
        run: |
          cd kernel_workspace
          ./kernel_platform/build_with_bazel.py -t ${{ github.event.inputs.CPUD }} ${{ github.event.inputs.BUILD_METHOD }}

      - name: Fallback build kernel
        if: ${{ github.event.inputs.CPU != 'sm8650' && github.event.inputs.CPU != 'sm7675' }}
        run: |
          cd kernel_workspace
          LTO=thin SYSTEM_DLKM_RE_SIGN=0 BUILD_SYSTEM_DLKM=0 KMI_SYMBOL_LIST_STRICT_MODE=0 ./kernel_platform/oplus/build/oplus_build_kernel.sh ${{ github.event.inputs.CPUD }} ${{ github.event.inputs.BUILD_METHOD }}

      - name: Prepare AnyKernel3 directory
        run: |
          git clone https://github.com/Numbersf/AnyKernel3 --depth=1
          rm -rf ./AnyKernel3/.git

      - name: Copy compiled Image to AnyKernel3
        run: |
          dir1="kernel_workspace/kernel_platform/out/msm-kernel-${{ github.event.inputs.CPUD }}-${{ github.event.inputs.BUILD_METHOD }}/dist/"
          dir2="kernel_workspace/kernel_platform/bazel-out/k8-fastbuild/bin/msm-kernel/${{ github.event.inputs.CPUD }}_gki_kbuild_mixed_tree/"
          dir3="kernel_workspace/kernel_platform/out/msm-${{ github.event.inputs.CPUD }}-${{ github.event.inputs.CPUD }}-${{ github.event.inputs.BUILD_METHOD }}/dist/"
          dir4="kernel_workspace/kernel_platform/out/msm-kernel-${{ github.event.inputs.CPUD }}-${{ github.event.inputs.BUILD_METHOD }}/gki_kernel/common/arch/arm64/boot/"
          dir5="kernel_workspace/kernel_platform/out/msm-${{ github.event.inputs.CPUD }}-${{ github.event.inputs.CPUD }}-${{ github.event.inputs.BUILD_METHOD }}/gki_kernel/common/arch/arm64/boot/"
          target="./AnyKernel3/"
          found=$(find "$dir1" -name "Image" -o -name "Image.gz")
          if [ -n "$found" ]; then
            cp "$found" ./AnyKernel3/Image
          else
            echo "Image not found!"
            exit 1
          fi
```

## README (Usage)

This section explains how to use the workflow and its inputs.

* **Trigger:** Manual dispatch via GitHub Actions (`workflow_dispatch`). You must run this workflow manually from the GitHub Actions tab.

* **Inputs:** Configure the following parameters (defaults shown for OnePlus Ace 2 Pro):

  * **CPU (Branch):** The kernel branch matching the Snapdragon model. *Default: `sm8550`* (Snapdragon 8 Gen 2).
  * **FEIL (Device Profile):** The device XML manifest name. *Default: `oneplus_ace2pro_v`*.
  * **CPUD (Codename):** Device codename. *Default: `kalama`* (OnePlus Ace 2 Pro).
  * **ANDROID\_VERSION:** Base Android (e.g., `android13`). *Default: `android13`*.
  * **KERNEL\_VERSION:** Major kernel version. *Default: `5.15`*.
  * **BUILD\_METHOD:** Build method, either `gki` (default) or `perf`.
  * **SUSFS\_CI:** If `true`, download the SUSFS kernel module from a CI artifact; if `false`, from the latest GitHub Release. *Default: `true`*.
  * **VFS:** Enable manual VFS hooks (SUSFS). *Default: `true`*.
  * **KPM:** Enable KPM support in KernelSU. *Default: `true`*.
  * **ZRAM:** Enable extra ZRAM algorithms (LZ4KD support). *Default: `false`*.

* **Workflow Steps (summary):**

  1. Sync OnePlus kernel source for the selected branch/profile.
  2. Remove any `-dirty` tag from version scripts.
  3. Install KernelSU (SukiSU Ultra) setup and apply patches (SUSFS, hide-stuff, VFS hooks, optional ZRAM).
  4. Enable KernelSU (`CONFIG_KSU`), KPM, and other options in `gki_defconfig`.
  5. Build the kernel. For most devices, it uses the OPLUS build script; for `sm8650`/`sm7675`, it uses a Bazel GKI build.
  6. Prepare an AnyKernel3 folder and copy the resulting `Image`. If KPM is enabled, it patches the image with `patch_linux`.
  7. Download the SUSFS kernel module ZIP (from CI or release) and include it.
  8. Upload the AnyKernel3 folder as an artifact, named `AnyKernel3_SukiSUUltra_<KSUVER>_<Device><Suffix>`.

* **Outputs:** The workflow produces an AnyKernel3 package containing the compiled kernel image (and optional dtbo/vendor images for certain devices). The artifact name includes the KernelSU version (`KSUVER`), device name (FEIL without `_v` suffix), and suffixes indicating enabled features (e.g. `_KPM`, `_VFS`, `_LZ4KD`).

### Example

To build the SukiSU Ultra kernel for the OnePlus Ace 2 Pro on Android 13 with default options (GKI build, KPM and VFS enabled, no extra ZRAM), use the following inputs:

```yaml
CPU: sm8550
FEIL: oneplus_ace2pro_v
CPUD: kalama
ANDROID_VERSION: android13
KERNEL_VERSION: 5.15
BUILD_METHOD: gki
SUSFS_CI: true
VFS: true
KPM: true
ZRAM: false
```

After running, check the **Actions** page for the uploaded artifact (AnyKernel3 ZIP). You can then flash this package on your device using a custom recovery.
