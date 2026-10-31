# GSI Manager — implementation status

Branch: `feature/gsi-manager-erofs-ext4-20261031`  
Base: `android-12.1`  
Target device: Samsung Galaxy A35 (SM-A356E)

## Milestone 1: read-only image preflight

This milestone adds a GSI Manager entry to Advanced and a dedicated image picker. The analysis action only opens the selected image read-only and checks known signatures:

- Android sparse image magic.
- Raw EROFS superblock magic at offset 1024.
- Raw EXT4 filesystem magic at offset 1080.

It reports the image size and recognized signature. It also shows the currently mapped System logical partition's active slot, filesystem type and size as a read-only target preflight. It does not yet calculate free extents in the LP group. The action does not flash, mount, format, extract, resize, rewrite, or delete anything. It never writes to or formats `/data`, and it does not modify `super` metadata.

Sparse images are identified as sparse containers only; their contained filesystem is not yet inspected. Compressed images and unknown formats are reported as unsupported.

## Build prerequisites

The matching A35 feature tree enables `TW_INCLUDE_GSI_MANAGER := true`. The SHRP 12.1 manifest's original EROFS tools are older than the required recovery/xattr feature set, so the separate builder feature workflow pins `external/erofs-utils` to AOSP commit `7293c06aaf1a792ff4fd94b1baa5a2194406c5a5` (2025-01-19). That revision defines `fsck.erofs.recovery` and supports `--xattrs` export using `lsetxattr`. The feature workflow checks these source capabilities before building. This has not yet been compiled or tested on the A35.

The new build path is deliberately isolated in `feature/gsi-manager-build-20261031` and remains manual until the complete installer/converter is ready. Do not select this diagnostic/preflight build for normal use.

## Planned milestones

1. **GSI image preflight** — current milestone, read-only only. Shows the active System target, but does not yet determine LP group free space.
2. **EROFS extraction and EXT4 image creation** — the core has an opt-in build switch, `TW_INCLUDE_GSI_MANAGER=true`, which requests `fsck.erofs.recovery`, `mke2fs.recovery` and `e2fsdroid.recovery`; the base recovery already includes its `mke2fs.conf` module. The converter must require `fsck.erofs --extract=<staging-dir> --preserve --xattrs`, preserve ownership/permissions as root, and prove that every security xattr including `security.selinux` survived extraction. It must also validate the EXT4 image and its labels before any target write path exists.
3. **Dynamic partition planning** — read current LP metadata, active slot, group limits, free extents, snapshot/merge state and target partition size. No automatic resizing until all these conditions can be proven safe.
4. **GApps ZIP compatibility** — inspect the user-provided ZIP for Android API level, ARM64 support and package metadata; then pass a compatible package to the existing ZIP installer. No Magisk module installation or bundled GApps binaries.
5. **Write/flash stage** — separate explicit confirmation, verified target mapping and size checks. System replacement may overwrite the existing logical `system` contents. It must never format or wipe `/data`, userdata, metadata, vendor, or other personal partitions. If metadata, snapshot state, free space, or filesystem metadata cannot be validated, the action must abort without writing.

## Important limitations

EROFS cannot be converted in place while preserving the current filesystem. The intended workflow is: inspect the EROFS image, extract its tree while preserving ownership, xattrs and SELinux labels, create a new EXT4 image, verify it, and then replace the logical system image. A GApps package must match the ROM's Android version/API and ARM64 ABI; Pico/Full/Stock are package variants, not universal compatibility guarantees.

**This milestone is not a GSI flasher or EROFS converter yet.** The UI and read-only identification are a safe foundation only. Do not use it to install a ROM.
