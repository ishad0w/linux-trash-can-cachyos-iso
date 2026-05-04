# Working on linux-trash-can-cachyos-iso

This repository is the CachyOS live ISO profile for Mac Pro 6,1. It is not the kernel source. The kernel source and Arch package recipe live in the sibling `linux-trash-can` repository.

Use this file as the operating guide. Use [`MAP.md`](MAP.md) as the file-by-file index and [`TECH-DEBT.md`](TECH-DEBT.md) before deleting inherited CachyOS paths.

## Read Order

1. [`README.md`](README.md)
2. [`MAP.md`](MAP.md)
3. [`TECH-DEBT.md`](TECH-DEBT.md)
4. [`archiso/pacman.conf`](archiso/pacman.conf)
5. [`archiso/packages_desktop.x86_64`](archiso/packages_desktop.x86_64)
6. [`archiso/grub/grub.cfg`](archiso/grub/grub.cfg)
7. [`buildiso.sh`](buildiso.sh) and [`util-iso.sh`](util-iso.sh)

## Source Of Truth

| Topic | Primary files | Notes |
|------|------|------|
| Project scope and workflow | [`README.md`](README.md) | Build and boot overview for the ISO profile. |
| File navigation | [`MAP.md`](MAP.md) | Directory map and change surfaces. |
| Confirmed debt | [`TECH-DEBT.md`](TECH-DEBT.md) | Start here before cleanup work. |
| Kernel package input | [`archiso/pacman.conf`](archiso/pacman.conf), [`local-repo/`](local-repo), [`util-iso.sh`](util-iso.sh) | The committed pacman file has a placeholder `file://` path. `buildiso.sh` rewrites the copied build profile from `MACPRO_LOCAL_REPO` or `./local-repo`. |
| Live package set | [`archiso/packages_desktop.x86_64`](archiso/packages_desktop.x86_64) | Contains `linux-macpro61`, headers, and generic CachyOS desktop packages. |
| GRUB live boot | [`archiso/grub/grub.cfg`](archiso/grub/grub.cfg) | Main UEFI boot menu with the Mac Pro kernel as the intended primary path. |
| BIOS/syslinux live boot | [`archiso/syslinux/archiso_sys-linux.cfg`](archiso/syslinux/archiso_sys-linux.cfg) | BIOS path with a Mac Pro kernel entry. |
| systemd-boot entries | [`archiso/efiboot/loader/entries/`](archiso/efiboot/loader/entries) | Currently inherited/generic and not aligned with the Mac Pro GRUB path. |
| Live root overlay | [`archiso/airootfs/`](archiso/airootfs) | Reboot protection, modprobe defaults, installer launchers, services, and cleanup scripts. |
| Build orchestration | [`buildiso.sh`](buildiso.sh), [`util-iso.sh`](util-iso.sh), [`util.sh`](util.sh), [`util-msg.sh`](util-msg.sh), [`util-iso-mount.sh`](util-iso-mount.sh) | CachyOS archiso build wrapper. |
| CI and tests | [`.github/workflows/build.yml`](.github/workflows/build.yml), [`testcases/`](testcases), [`machines/`](machines), [`testiso.sh`](testiso.sh) | The build job now creates Mac Pro package inputs from the sibling kernel repo. The test paths still need revalidation for this fork and Mac Pro behavior. |

## Hard Rules

- Keep repository documentation in English.
- Treat `ishad0w/linux-trash-can-cachyos-iso` and the sibling `ishad0w/linux-trash-can` kernel repo as the maintained source of truth. Historical `wolffcatskyy/*` links are archival only.
- Preserve Mac Pro 6,1 boot invariants: cold boot requirement, `amdgpu.si_support=1`, `amdgpu.dc=0`, `radeon.si_support=0` where radeon can load, and `acpi_mask_gpe=0x16` on the current packaged path.
- If you change the kernel package name, update the package list, bootloader entries, mkinitcpio presets, pacman hook, README, and MAP together.
- If you change a boot parameter, inspect GRUB, syslinux, systemd-boot, loopback, PXE, and modprobe copies before calling the change complete.
- Do not assume the committed `[macpro]` repo URL is directly usable. It is a placeholder; use `buildiso.sh` or set a real `file://` path when calling `mkarchiso` directly.
- Keep shell changes Bash-friendly and syntax-checkable with `bash -n`.
- Do not delete inherited CachyOS/NVIDIA/VM/test helpers just because they look generic; first confirm whether Calamares, CachyOS tooling, or live testing still depends on them.

## Change Matrix

| Change type | Minimum files to inspect | Usually also update |
|------|------|------|
| Kernel package input | [`archiso/pacman.conf`](archiso/pacman.conf), [`archiso/packages_desktop.x86_64`](archiso/packages_desktop.x86_64), [`local-repo/`](local-repo) | [`README.md`](README.md), [`MAP.md`](MAP.md), CI workflow |
| Boot menu or kernel cmdline | [`archiso/grub/grub.cfg`](archiso/grub/grub.cfg), [`archiso/syslinux/archiso_sys-linux.cfg`](archiso/syslinux/archiso_sys-linux.cfg), [`archiso/efiboot/loader/entries/`](archiso/efiboot/loader/entries), [`archiso/grub/loopback.cfg`](archiso/grub/loopback.cfg), [`archiso/syslinux/archiso_pxe-linux.cfg`](archiso/syslinux/archiso_pxe-linux.cfg) | [`README.md`](README.md), [`TECH-DEBT.md`](TECH-DEBT.md) |
| Live root defaults | [`archiso/airootfs/etc/modprobe.d/`](archiso/airootfs/etc/modprobe.d), [`archiso/airootfs/etc/profile.d/no-reboot.sh`](archiso/airootfs/etc/profile.d/no-reboot.sh), [`archiso/airootfs/etc/systemd/`](archiso/airootfs/etc/systemd) | [`README.md`](README.md), [`MAP.md`](MAP.md) |
| Build script changes | [`buildiso.sh`](buildiso.sh), [`util-iso.sh`](util-iso.sh), [`util.sh`](util.sh), [`util-iso-mount.sh`](util-iso-mount.sh), [`util-msg.sh`](util-msg.sh) | [`README.md`](README.md), CachyOS keyring/trust notes, CI workflow |
| Installer/test changes | [`archiso/airootfs/usr/local/bin/`](archiso/airootfs/usr/local/bin), [`testcases/`](testcases), [`machines/`](machines) | [`README.md`](README.md), [`TECH-DEBT.md`](TECH-DEBT.md) |
| Release process changes | [`.github/workflows/build.yml`](.github/workflows/build.yml), [`README.md`](README.md), [`CHANGELOG.md`](CHANGELOG.md) | Root workspace docs |

## Validation

- Run `bash -n` on every changed shell script.
- For docs-only changes, verify relative links from the edited files.
- On an Arch/CachyOS build host, run `sudo ./buildiso.sh -p desktop -v -w` after build-script or profile changes.
- If CI changes are touched, validate that the workflow still builds or otherwise provides the `linux-macpro61` and headers packages before expecting ISO builds to pass.
- Heavy ISO builds, live USB boot tests, and Mac Pro hardware tests are expensive. If they were not run, say so explicitly.

## Watchpoints

- [`buildiso.sh`](buildiso.sh) expects `./local-repo` by default, or `MACPRO_LOCAL_REPO` when a different package repo is needed. It refreshes `macpro.db` and rewrites the copied `[macpro]` URL before `mkarchiso`.
- [`archiso/efiboot/loader/entries/`](archiso/efiboot/loader/entries) still defaults to generic CachyOS LTS/stable entries rather than `linux-macpro61`.
- [`archiso/airootfs/etc/mkinitcpio.d/linux.preset`](archiso/airootfs/etc/mkinitcpio.d/linux.preset) references `linux-cachyos-lts`, while GRUB/syslinux expect `initramfs-linux-macpro61.img` for the primary Mac Pro path.
- [`ci.build.sh`](ci.build.sh) references `fix_permissions.sh` and `mkarchiso`, neither of which exists in this tree.
- [`util-iso.sh`](util-iso.sh) patches `/usr/bin/mkarchiso` in place on the host, which is risky and should be treated as build-host mutation.
- [`CHANGELOG.md`](CHANGELOG.md) is a generic upstream CachyOS changelog snapshot, not a maintained Mac Pro release log.
