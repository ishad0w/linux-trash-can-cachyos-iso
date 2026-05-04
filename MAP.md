# Repository Map

This file is the navigation index for `linux-trash-can-cachyos-iso`: what each area does, where the Mac Pro-specific behavior lives, and which inherited CachyOS paths are still only partially aligned.

## Read This First

| File | Use it for |
|------|------|
| [`README.md`](README.md) | Project overview, build inputs, boot flow, and current caveats. |
| [`AGENTS.md`](AGENTS.md) | Working rules, source-of-truth notes, and validation expectations. |
| [`TECH-DEBT.md`](TECH-DEBT.md) | Confirmed debt and cleanup priorities. |
| [`archiso/pacman.conf`](archiso/pacman.conf) | Pacman repositories used during ISO build, including the custom `[macpro]` repo. |
| [`archiso/packages_desktop.x86_64`](archiso/packages_desktop.x86_64) | Live ISO package list. |
| [`archiso/grub/grub.cfg`](archiso/grub/grub.cfg) | Main GRUB boot menu and Mac Pro kernel parameters. |
| [`buildiso.sh`](buildiso.sh) | Top-level ISO build wrapper. |

## How The Repository Fits Together

```text
../linux-trash-can/packaging/arch/*.pkg.tar.zst
    -> local-repo/
    -> buildiso.sh refreshes macpro.db
    -> build/archiso/pacman.conf [macpro]
    -> archiso/packages_desktop.x86_64
    -> buildiso.sh + util-iso.sh
    -> out/desktop/*.iso

archiso/grub/grub.cfg + archiso/syslinux/archiso_sys-linux.cfg
    -> intended Mac Pro live boot path

archiso/airootfs/*
    -> live environment defaults, installer scripts, services, hooks

testcases/* + machines/*
    -> quicktest/quickemu installer smoke tests
```

## Current Change Surfaces

| Surface | Main files | What is really changing here |
|------|------|------|
| Kernel package source | [`archiso/pacman.conf`](archiso/pacman.conf), [`local-repo/`](local-repo), [`archiso/packages_desktop.x86_64`](archiso/packages_desktop.x86_64), [`util-iso.sh`](util-iso.sh) | The ISO consumes prebuilt `linux-macpro61` and headers packages. `buildiso.sh`/`util-iso.sh` validate the repo, refresh `macpro.db`, and rewrite the copied build profile to the current local repo path. |
| Live boot entries | [`archiso/grub/grub.cfg`](archiso/grub/grub.cfg), [`archiso/syslinux/archiso_sys-linux.cfg`](archiso/syslinux/archiso_sys-linux.cfg), [`archiso/efiboot/loader/entries/`](archiso/efiboot/loader/entries), [`archiso/grub/loopback.cfg`](archiso/grub/loopback.cfg), [`archiso/syslinux/archiso_pxe-linux.cfg`](archiso/syslinux/archiso_pxe-linux.cfg) | GRUB and BIOS syslinux have Mac Pro entries. systemd-boot, loopback, and PXE are still mostly generic CachyOS LTS/stable paths. |
| Live root hardware defaults | [`archiso/airootfs/etc/modprobe.d/macpro-gpu.conf`](archiso/airootfs/etc/modprobe.d/macpro-gpu.conf), [`archiso/airootfs/etc/profile.d/no-reboot.sh`](archiso/airootfs/etc/profile.d/no-reboot.sh), [`archiso/airootfs/etc/systemd/system/reboot.target`](archiso/airootfs/etc/systemd/system/reboot.target) | Forces amdgpu SI support, disables warm reboot behavior, and masks reboot in the live root. |
| Installer launch | [`archiso/airootfs/usr/local/bin/calamares-online.sh`](archiso/airootfs/usr/local/bin/calamares-online.sh), [`archiso/airootfs/usr/local/bin/pkexec-wrapper`](archiso/airootfs/usr/local/bin/pkexec-wrapper), [`archiso/airootfs/usr/local/bin/prepare-live-desktop.sh`](archiso/airootfs/usr/local/bin/prepare-live-desktop.sh) | Keyring refresh, Calamares launch, desktop preparation, and live environment workarounds. |
| Inherited cleanup helpers | [`archiso/airootfs/usr/local/bin/remove-nvidia`](archiso/airootfs/usr/local/bin/remove-nvidia), [`removeun`](archiso/airootfs/usr/local/bin/removeun), [`removeun-online`](archiso/airootfs/usr/local/bin/removeun-online), [`nvidia-module-loader`](archiso/airootfs/usr/local/bin/nvidia-module-loader) | Generic CachyOS/NVIDIA/VM cleanup behavior that may or may not still belong in a Mac Pro-only ISO. |
| Build wrapper | [`buildiso.sh`](buildiso.sh), [`util-iso.sh`](util-iso.sh), [`util.sh`](util.sh), [`util-iso-mount.sh`](util-iso-mount.sh), [`util-msg.sh`](util-msg.sh) | Prepares the archiso profile, mutates generated files, calls `mkarchiso`, then creates checksums and signs outputs only when a GPG secret key is available. |
| CI/test flow | [`.github/workflows/build.yml`](.github/workflows/build.yml), [`ci.build.sh`](ci.build.sh), [`testiso.sh`](testiso.sh), [`testcases/`](testcases), [`machines/`](machines) | GitHub Actions now checks out the sibling kernel repo and builds `linux-macpro61` packages before the ISO step. The quicktest/local paths remain inherited from CachyOS and still need revalidation for Mac Pro-specific behavior. |

## File Map

### Root

| Path | Purpose | Notes |
|------|------|------|
| [`README.md`](README.md) | Top-level project description, build workflow, and docs hub. | Updated to treat historical upstream links as archival and the local fork as current. |
| [`AGENTS.md`](AGENTS.md) | Working rules for humans and automation. | Added so future work starts with known invariants and drift points. |
| [`MAP.md`](MAP.md) | File-by-file repository map. | This file. |
| [`TECH-DEBT.md`](TECH-DEBT.md) | Debt register for stale, partial, risky, or misleading paths. | Start here before cleanup. |
| [`CHANGELOG.md`](CHANGELOG.md) | Upstream CachyOS changelog snapshot. | Not a maintained Mac Pro-specific changelog. |
| [`LICENSE`](LICENSE) | GPL license text. | Inherited from the CachyOS live ISO profile. |
| [`.gitignore`](.gitignore) | Build artifact ignore rules. | Includes duplicated build/out/ISO patterns and local repo package outputs. |
| [`.github/workflows/build.yml`](.github/workflows/build.yml) | GitHub Actions build + quicktest workflow. | Checks out `ishad0w/linux-trash-can`, builds `packaging/arch`, stages packages into `local-repo/`, then runs `buildiso.sh`. Still needs a successful CI run before it is a proven release path. |

### Build Scripts

| Path | Purpose | Notes |
|------|------|------|
| [`buildiso.sh`](buildiso.sh) | Top-level build wrapper. | Parses `-p`, `-c`, `-r`, `-w`, `-v`; imports helper scripts; calls `run_build`. Root escalation is currently commented out. |
| [`util-iso.sh`](util-iso.sh) | Main archiso orchestration. | Generates MOTD/version tags, prepares profile, validates the Mac Pro package repo, refreshes `macpro.db`, rewrites the copied `[macpro]` pacman path, mutates `/usr/bin/mkarchiso`, runs `mkarchiso`, creates checksums, and delegates optional ISO signing. |
| [`util.sh`](util.sh) | Generic helpers. | Timers, root escalation, archiso dependency check, checksums, and optional GPG signing when a secret key is available. |
| [`util-iso-mount.sh`](util-iso-mount.sh) | Mount cleanup helpers. | Unmounts active image and filesystem mounts under the work directory. |
| [`util-msg.sh`](util-msg.sh) | Terminal output helpers. | Colorized `msg`, `info`, `warning`, `error`, and `import`. |
| [`ci.build.sh`](ci.build.sh) | Stale CI helper. | References missing `fix_permissions.sh` and `mkarchiso`; not the current top-level build path. |
| [`testiso.sh`](testiso.sh) | Local VirtualBox launch helper. | Attaches the latest built ISO to a `CachyOS` VM. Not Mac Pro hardware validation. |
| [`lddd.sh`](lddd.sh) | Missing-library scanner. | Inherited helper that scans ELF files and reports possible rebuilds. |

### `archiso/`

| Path | Purpose | Notes |
|------|------|------|
| [`archiso/profiledef.sh`](archiso/profiledef.sh) | archiso profile metadata. | Sets `iso_name=cachyos-macpro`, GRUB/syslinux boot modes, squashfs options, and file permissions. |
| [`archiso/pacman.conf`](archiso/pacman.conf) | Build-time pacman config template. | Contains CachyOS, Arch, multilib, and `[macpro]` repos. The committed `[macpro]` URL is a placeholder; `buildiso.sh` rewrites the copied config in `build/archiso/`. |
| [`archiso/packages_desktop.x86_64`](archiso/packages_desktop.x86_64) | Desktop live ISO package list. | Includes 194 uncommented packages, including `linux-macpro61`, headers, and `linux-cachyos-lts` fallback. |
| [`archiso/bootstrap_packages.x86_64`](archiso/bootstrap_packages.x86_64) | Minimal bootstrap package list. | Currently only `arch-install-scripts` and `base`. |

### Boot Loaders

| Path | Purpose | Notes |
|------|------|------|
| [`archiso/grub/grub.cfg`](archiso/grub/grub.cfg) | Main GRUB boot menu. | Primary entry boots `vmlinuz-linux-macpro61` with `amdgpu.si_support=1`, `amdgpu.dc=0`, `acpi_mask_gpe=0x16`, and NVMe loading. |
| [`archiso/grub/loopback.cfg`](archiso/grub/loopback.cfg) | GRUB loopback ISO boot. | Still generic CachyOS LTS and `i915/amdgpu.modeset` oriented, not Mac Pro-specific. |
| [`archiso/syslinux/archiso_sys-linux.cfg`](archiso/syslinux/archiso_sys-linux.cfg) | BIOS/syslinux live boot entries. | Has a Mac Pro primary entry, LTS fallback, and nomodeset safe mode. |
| [`archiso/syslinux/archiso_pxe-linux.cfg`](archiso/syslinux/archiso_pxe-linux.cfg) | PXE boot entries. | Still generic CachyOS LTS. |
| [`archiso/efiboot/loader/entries/`](archiso/efiboot/loader/entries) | systemd-boot entries. | Still generic CachyOS LTS/stable and does not expose `linux-macpro61` as the default. |
| [`archiso/efiboot/loader/loader.conf`](archiso/efiboot/loader/loader.conf) | systemd-boot default selector. | Defaults to `01-archiso-linux.conf`, currently LTS. |

### `archiso/airootfs/`

| Path | Purpose | Notes |
|------|------|------|
| [`archiso/airootfs/etc/modprobe.d/macpro-gpu.conf`](archiso/airootfs/etc/modprobe.d/macpro-gpu.conf) | Mac Pro GPU module defaults. | Forces amdgpu SI and disables radeon SI. |
| [`archiso/airootfs/etc/profile.d/no-reboot.sh`](archiso/airootfs/etc/profile.d/no-reboot.sh) | Live shell reboot guard. | Aliases `reboot` to a message plus `sudo poweroff`. |
| [`archiso/airootfs/etc/systemd/system/reboot.target`](archiso/airootfs/etc/systemd/system/reboot.target) | Reboot target mask. | Symlink to `/dev/null`. |
| [`archiso/airootfs/etc/pacman.d/hooks/99-esp-kernel-sync.hook`](archiso/airootfs/etc/pacman.d/hooks/99-esp-kernel-sync.hook) | Kernel-to-ESP sync hook. | Copies `vmlinuz-linux-macpro61` and initramfs to `/boot/efi` if mounted. |
| [`archiso/airootfs/etc/mkinitcpio.d/linux.preset`](archiso/airootfs/etc/mkinitcpio.d/linux.preset) | Initramfs preset. | Still points at `linux-cachyos-lts`, not `linux-macpro61`. |
| [`archiso/airootfs/usr/local/bin/calamares-online.sh`](archiso/airootfs/usr/local/bin/calamares-online.sh) | Online Calamares launcher. | Refreshes keyrings, syncs time, captures `inxi`, installs Calamares, then launches with `pkexec-wrapper`. |
| [`archiso/airootfs/usr/local/bin/prepare-live-desktop.sh`](archiso/airootfs/usr/local/bin/prepare-live-desktop.sh) | Desktop preparation helper. | Creates live trash directory and marks launchers trusted for XFCE. |
| [`archiso/airootfs/usr/local/bin/remove-nvidia`](archiso/airootfs/usr/local/bin/remove-nvidia) | VM guest package cleanup. | Removes VirtualBox/VMware/QEMU guest packages when not running in a VM. |
| [`archiso/airootfs/usr/local/bin/removeun`](archiso/airootfs/usr/local/bin/removeun) | Offline cleanup helper. | Inherited package and file cleanup logic; still mentions NVIDIA package removal. |
| [`archiso/airootfs/usr/local/bin/removeun-online`](archiso/airootfs/usr/local/bin/removeun-online) | Online cleanup helper. | Removes most non-base packages, switches to v3 repos when supported, repopulates CachyOS keys. |
| [`archiso/airootfs/usr/local/bin/nvidia-module-loader`](archiso/airootfs/usr/local/bin/nvidia-module-loader) | Smart NVIDIA/ nouveau loader. | Generic CachyOS behavior, questionable for a Mac Pro-only ISO. |

### Tests

| Path | Purpose | Notes |
|------|------|------|
| [`machines/cachyos-dailylive.conf`](machines/cachyos-dailylive.conf) | quickemu machine definition. | Finds the first ISO in the repository. |
| [`testcases/cachyos/dailylive/test_install_calamares`](testcases/cachyos/dailylive/test_install_calamares) | quicktest Calamares install flow. | Drives GRUB, launches Calamares, chooses bootloader/filesystem from environment variables, then powers off VM. |
| [`testcases/cachyos/dailylive/test_no_test_gather_screenshots_and_text_only`](testcases/cachyos/dailylive/test_no_test_gather_screenshots_and_text_only) | Manual screenshot/OCR gathering loop. | Useful for updating quicktest text anchors. |
| [`testcases/cachyos/dailylive/i18n/en_US`](testcases/cachyos/dailylive/i18n/en_US) | OCR anchor text. | English strings used by quicktest. |
| [`testcases/keymaps/en_US`](testcases/keymaps/en_US) | Key mapping for quicktest input. | Maps characters to QEMU sendkey names. |

## Generated And Runtime-Only Files

These are mentioned in docs or scripts but should not be committed:

- `build/`
- `out/`
- `work/`
- `local-repo/*.pkg.tar.zst`
- `local-repo/*.db*`
- `local-repo/*.files*`
- `archiso/packages.x86_64`
- `archiso/airootfs/etc/motd`
- `archiso/airootfs/etc/environment`
- `archiso/airootfs/etc/version-tag`
- `archiso/airootfs/etc/edition-tag`
- `*.iso`, `*.img`, `*.qcow2`, checksum/signature outputs

## Current Nuances To Remember

- This repo does not build the kernel. It consumes packages from the sibling kernel project.
- The committed `[macpro]` pacman repo path is a placeholder; direct `mkarchiso` calls still need a real `file://` repo path or should go through `buildiso.sh`.
- GRUB and BIOS syslinux are the most Mac Pro-specific boot paths today.
- systemd-boot, loopback GRUB, and PXE are inherited/generic and should not be treated as equivalent to the primary Mac Pro GRUB path.
- The package list includes `linux-cachyos-lts` as fallback but the `02-archiso-linux-cachyos.conf` entry references `linux-cachyos`, which is not in the package list.
- The live root masks warm reboot, but the GRUB menu still exposes a restart entry with a warning.
- The inherited upstream CachyOS changelog is not a release log for this fork.

Start with [`AGENTS.md`](AGENTS.md) when planning work. Start with this file when you need to know where that work actually lives.
