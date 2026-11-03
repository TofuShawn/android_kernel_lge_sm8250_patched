# LineageOS 23.2 kernel for LG V60 ThinQ (`timelm`) with BakaSU

`msm-4.19` kernel for the LG V60 ThinQ (SM8250 / `kona`, LineageOS codename `timelm`) with
[BakaSU](https://github.com/Baka-SU/BakaSU) root integrated at source level through the
**manual hook** backend. Built and then verified by booting the result on a physical LM-V600.
The reasoning behind each patch is in the commit messages.

| | |
|---|---|
| Base | `lineage-23.2` @ `29902cf733dc` — the commit the latest official 23.2 `boot.img` was built from |
| Kernel | 4.19.325 (`-cip133-st17`) |
| Root | BakaSU, `CONFIG_KSU_MANUAL_HOOK=y` (the tracepoint backend needs GKI2/5.10+) |
| Toolchain | clang-r416183b (clang 12.0.5) + `ld.lld`, as pinned by `build.config.common` |
| Config pair | `vendor/kona-perf_defconfig` + `vendor/timelm.config` fragment (there is no `timelm_defconfig`) |
| Branch here | `lineage-23.2-bakasu`; upstream `lineage-22.2` untouched |

## Build

```bash
# BakaSU itself is not vendored; setup.sh clones it into KernelSU/ and creates the
# drivers/kernelsu symlink (both gitignored).
curl -sSL https://raw.githubusercontent.com/Baka-SU/BakaSU/refs/heads/main/kernel/setup.sh | bash -s main

make O=out ARCH=arm64 CC=clang LD=ld.lld LLVM=1 LLVM_IAS=1 \
     CLANG_TRIPLE=aarch64-linux-gnu- CROSS_COMPILE=aarch64-linux-gnu- \
     CROSS_COMPILE_COMPAT=arm-linux-gnueabi- vendor/kona-perf_defconfig
./scripts/kconfig/merge_config.sh -O out/ -m out/.config arch/arm64/configs/vendor/timelm.config
make O=out ARCH=arm64 ... olddefconfig
make -j"$(nproc)" O=out ARCH=arm64 CC=clang LD=ld.lld LLVM=1 LLVM_IAS=1 \
     CLANG_TRIPLE=aarch64-linux-gnu- CROSS_COMPILE=aarch64-linux-gnu- \
     CROSS_COMPILE_COMPAT=arm-linux-gnueabi- DTC_FLAGS="-@ -f" all Image.gz dtbs modules
```

`O=out` is mandatory (an in-tree `.config` breaks the next build), and **`Image.gz` must be
named explicitly**: on this tree `all` expands to `dtbs` + `Image`, not `Image.gz`.
`DTC_FLAGS="-f"` is needed because in-tree dtc 1.4.6 treats LG's duplicate sound labels as
errors — but the `.dtbo` it produces are broken and must **not** be flashed; keep the ROM's.

## Two ways to brick the flash

* **The boot partition is bigger than its header says.** After the header-declared sections
  sit three appended FDT blobs that `aboot` selects between via the `androidboot.dtb_idx`
  cmdline token. A `boot.img` truncated at the declared end therefore has **no DTB**: the
  kernel dies before USB enumerates and the device drops into
  `BSP-Kernel / LGE Crash Handler : UEFI Crash!` (Sahara mode — exit with Volume Down ~1 s).
  Any repack must carry that tail to its new page-aligned end, and rewrite only
  `kernel_size` / `ramdisk_size`.
* **Never `fastboot flash boot_a <full partition dump>`.** LG's edk2 fastboot asserts while
  handling it (`SafeString.c (651): DestLength > SourceLength`) and lands in the same Sahara
  mode. Flash only files trimmed to real content (~24 MB).

Back up your own boot before flashing: dump `/dev/block/by-name/boot_a`, verify the copy,
and restore from that. Do not assume the other slot is a rollback — check what it actually
holds first.

```bash
fastboot flash boot_a <your-boot.img>
```

## Kernel modules

The ROM's 40 DLKMs in `/vendor/lib/modules` load against this kernel unmodified, so a
flashable zip only needs the kernel. The `-dirty` suffix in `uname -r` is irrelevant to
them, and the details of why are in the commit messages.

CI: built by `TofuShawn/NonGKI_Kernel_Build_timelm` (branch `Kpatch-1`).
