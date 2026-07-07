# Upstream Sync Notes, 2026-07-08

This fork was synced with `CachyOS/CachyOS-Live-ISO` `master` on 2026-07-08.

## Accepted Upstream Changes

- `cachyos-calamares` replaces the older `cachyos-calamares-next` package path.
- `inotify-tools` and `kxkb2locale1` were added to the live desktop package list.
- CachyOS live KDE keyboard sync hook was added.
- Plasma welcome/live marker was added to the default skeleton config.
- Quicktest Calamares launch/failure handling updates were accepted.
- Upstream changelog entries were merged.

## Fork-Specific Changes Preserved

- `linux-macpro61` and `linux-macpro61-headers` remain in the desktop package list.
- The `[macpro]` local repo flow remains required for kernel packages.
- The ISO workflow still builds the sibling `ishad0w/linux-trash-can` package before building the ISO.
- `cachyos-macpro` profile naming and Mac Pro boot parameters remain the fork contract.
- ISO signing gracefully skips when CI has no GPG secret key; `SKIP_ISO_SIGNING=true` is set in the build job.

## Boot Path Alignment

Primary entries now point at `linux-macpro61` in:

- GRUB
- BIOS syslinux
- systemd-boot entry `01-archiso-linux.conf`
- GRUB loopback
- PXE syslinux

`linux-cachyos-lts` remains installed and exposed as an explicit fallback/safe path. It is not the primary Mac Pro boot contract.

## Remaining Validation

- Build the ISO with a freshly built `linux-macpro61 7.1.3` package.
- Verify `initramfs-linux-macpro61.img` is present in the ISO.
- Run quicktest/Calamares matrix.
- Boot on real Mac Pro 6,1 hardware after full poweroff.
