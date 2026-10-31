# GSI Manager — implementation status

Branch: `feature/gsi-manager-erofs-ext4-20261031`  
Base: `android-12.1`  
Target device: Samsung Galaxy A35 (SM-A356E)

## Milestone 1: image preflight and guarded raw flashing

This milestone adds a GSI Manager entry to Advanced, a dedicated image picker, read-only format recognition, and a separately confirmed raw-image write path to the currently mapped active-slot System partition. Format recognition checks the EROFS/EXT4 signatures; it does not yet run a filesystem integrity checker on the source image.

- Android sparse image magic.
- Raw EROFS superblock magic at offset 1024.
- Raw EXT4 filesystem magic at offset 1080.

It reports the image size and recognized signature. The preflight shows the currently mapped System logical partition's active slot, filesystem type and size. The separate raw flash action accepts only raw EROFS and raw EXT4 images; it blocks Android sparse images, checks the active slot, mapper path, target size, and `/metadata` Virtual A/B snapshot state, then unmounts System before writing. It uses the existing SHRP raw block writer restricted to `/system`, and leaves LP metadata, the inactive slot, `/data`, userdata, vendor, product and odm untouched. It does not calculate free extents or resize the logical partition.

Sparse images are identified as sparse containers only; their contained filesystem is not yet inspected. Compressed images and unknown formats are reported as unsupported.

## Build prerequisites

The matching A35 feature tree enables `TW_INCLUDE_GSI_MANAGER := true`. The SHRP 12.1 manifest's original EROFS tools are older than the required recovery/xattr feature set, so the separate builder feature workflow pins `external/erofs-utils` to AOSP commit `7293c06aaf1a792ff4fd94b1baa5a2194406c5a5` (2025-01-19). That revision defines `fsck.erofs.recovery` and supports `--xattrs` export using `lsetxattr`. The feature workflow checks these source capabilities before building. This has not yet been compiled or tested on the A35.

The new build path is deliberately isolated in `feature/gsi-manager-build-20261031` and remains manual. None of these changes has been compiled or tested on the A35. Do not flash this feature branch to the phone yet.

## Planned milestones

1. **GSI image preflight and raw flashing** — implemented in source, not yet compiled/tested. Only raw EROFS/EXT4 images that fit the active System mapper are offered; sparse images and any state that cannot be checked are blocked. Source image filesystem integrity checking is still pending.
2. **EROFS extraction and EXT4 image creation** — the core has an opt-in build switch, `TW_INCLUDE_GSI_MANAGER=true`, which requests `fsck.erofs.recovery`, `mke2fs.recovery` and `e2fsdroid.recovery`; the base recovery already includes its `mke2fs.conf` module. The converter must require `fsck.erofs --extract=<staging-dir> --preserve --xattrs`, preserve ownership/permissions as root, and prove that every security xattr including `security.selinux` survived extraction. It must also validate the EXT4 image and its labels before any target write path exists.
3. **Dynamic partition planning** — read current LP metadata, active slot, group limits, free extents, snapshot/merge state and target partition size. No automatic resizing until all these conditions can be proven safe.
4. **GApps ZIP compatibility** — inspect the user-provided ZIP for Android API level, ARM64 support and package metadata; then pass a compatible package to the existing ZIP installer. No Magisk module installation or bundled GApps binaries.
5. **Write/flash stage** — a first raw-only path now has an explicit confirmation, active-slot check, mapped-System check, size check, OTA snapshot-state check and System unmount. It does not resize `super`; any size mismatch aborts. The conversion/GApps path still needs its own validation, backup/rollback strategy and testing before it can write.

## Important limitations

EROFS cannot be converted in place while preserving the current filesystem. The intended workflow is: inspect the EROFS image, extract its tree while preserving ownership, xattrs and SELinux labels, create a new EXT4 image, verify it, and then replace the logical system image. A GApps package must match the ROM's Android version/API and ARM64 ABI; Pico/Full/Stock are package variants, not universal compatibility guarantees.

**This is still experimental source, not a tested recovery build.** The raw-only write action is not ready for device use until it compiles and passes host/device tests. EROFS→EXT4 conversion, sparse expansion, and GApps ZIP inspection/installation are not implemented yet.
