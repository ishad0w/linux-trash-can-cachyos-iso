# Technical Debt

Current repo-level debt inventory for `linux-trash-can-cachyos-iso`.

Scope of this file:

- Tracks debt visible from the committed tree, docs, scripts, and boot profile
- Focuses on stale, partial, misleading, risky, or likely non-working paths
- Uses the current repository state as of 2026-05-04

This is not a full ISO build or Mac Pro hardware validation report. Some items are confirmed by tree inspection and path mismatch, not by live boot testing.

## Priority Legend

- `P1` - structural debt or likely breakage that can block a reproducible ISO
- `P2` - stale or partial paths that waste contributor time or weaken support claims
- `P3` - lower-risk cleanup and alignment work

## P1

### ISO-TD-001 - `[macpro]` package repo must be provided before local or CI builds

**State:** partially mitigated
**Area:** package input / reproducibility

Evidence:

- [`archiso/pacman.conf`](archiso/pacman.conf) now carries a placeholder `[macpro]` `file://` URL instead of a developer-local path
- [`util-iso.sh`](util-iso.sh) validates `MACPRO_LOCAL_REPO` or `./local-repo`, refreshes `macpro.db`, and rewrites the copied `build/archiso/pacman.conf` before `mkarchiso`
- CI and local builds still cannot succeed unless `linux-macpro61` and `linux-macpro61-headers` packages are present in the local repo

Why this matters:

- A fresh clone cannot build the ISO without the sibling kernel packages
- CI cannot prove the profile is buildable with the intended custom kernel

Exit criteria:

- Update CI to create, download, or otherwise provide the `linux-macpro61` package repo before `buildiso.sh`
- Document the chosen CI package source in README and release docs
- Keep direct `mkarchiso` usage documented as requiring a real `[macpro]` `file://` path

### ISO-TD-002 - Bootloader paths disagree on the default kernel

**State:** open
**Area:** boot profile / hardware support

Evidence:

- [`archiso/grub/grub.cfg`](archiso/grub/grub.cfg) has a primary `linux-macpro61` entry with Mac Pro parameters
- [`archiso/syslinux/archiso_sys-linux.cfg`](archiso/syslinux/archiso_sys-linux.cfg) has a BIOS `linux-macpro61` entry
- [`archiso/efiboot/loader/entries/01-archiso-linux.conf`](archiso/efiboot/loader/entries/01-archiso-linux.conf) defaults to `linux-cachyos-lts`
- [`archiso/efiboot/loader/entries/02-archiso-linux-cachyos.conf`](archiso/efiboot/loader/entries/02-archiso-linux-cachyos.conf) references `linux-cachyos`, which is not listed in [`archiso/packages_desktop.x86_64`](archiso/packages_desktop.x86_64)
- [`archiso/grub/loopback.cfg`](archiso/grub/loopback.cfg) and [`archiso/syslinux/archiso_pxe-linux.cfg`](archiso/syslinux/archiso_pxe-linux.cfg) still boot generic CachyOS LTS

Why this matters:

- UEFI users can accidentally boot a generic kernel path instead of the Mac Pro kernel depending on boot loader path
- Some entries can reference kernel artifacts that are not installed into the ISO
- Hardware support claims are only true for the Mac Pro-specific entries

Exit criteria:

- Decide which boot loaders are supported
- Make every supported boot loader default to `linux-macpro61` with the same kernel parameters
- Delete or clearly label unsupported loopback/PXE/systemd-boot entries

### ISO-TD-003 - Initramfs generation path is not aligned with the primary Mac Pro kernel

**State:** open
**Area:** initramfs / archiso profile

Evidence:

- GRUB/syslinux primary entries expect `initramfs-linux-macpro61.img`
- [`archiso/airootfs/etc/mkinitcpio.d/linux.preset`](archiso/airootfs/etc/mkinitcpio.d/linux.preset) references `vmlinuz-linux-cachyos-lts` and `initramfs-linux-cachyos-lts.img`
- The sibling kernel project documents a no-initramfs primary design but still tolerates initramfs as fallback

Why this matters:

- The ISO can expose a Mac Pro primary entry without a clearly generated matching initramfs
- Debugging boot failures becomes ambiguous: kernel package issue, preset issue, or bootloader issue

Exit criteria:

- Add/verify a `linux-macpro61` mkinitcpio preset in the ISO profile, or
- Change boot entries to the intended no-initramfs contract if supported by archiso, and
- Document the chosen contract in README and MAP

### ISO-TD-004 - CI is not proven to build this fork's real custom-kernel ISO

**State:** open
**Area:** CI / release

Evidence:

- [`.github/workflows/build.yml`](.github/workflows/build.yml) runs `./buildiso.sh` but does not build or fetch `linux-macpro61` packages into `local-repo/`
- [`ci.build.sh`](ci.build.sh) references `fix_permissions.sh` and `mkarchiso`, neither of which exists in this tree
- [`buildiso.sh`](buildiso.sh) now fails early if the Mac Pro packages are missing, but the workflow still does not provide them

Why this matters:

- CI can fail before testing the ISO profile, or pass only after hidden external state is supplied
- The release path is not self-documenting

Exit criteria:

- Remove or repair [`ci.build.sh`](ci.build.sh)
- Teach GitHub Actions to build/fetch `linux-macpro61` packages and create `local-repo`
- Upload ISO artifacts only after the Mac Pro boot artifacts are present and verified

## P2

### ISO-TD-005 - Build script mutates `/usr/bin/mkarchiso` on the host

**State:** open
**Area:** build host safety

Evidence:

- [`util-iso.sh`](util-iso.sh) patches `/usr/bin/mkarchiso` to remove `archlinux-keyring-wkd-sync.timer`
- Verbose mode also edits `/usr/bin/mkarchiso` to change `quiet="y"` to `quiet="n"`

Why this matters:

- Builds leave state behind on the host system
- Repeated builds can be hard to reason about because the system tool has been edited

Exit criteria:

- Patch a copied `mkarchiso` wrapper inside the build workspace, or
- Use supported archiso hooks/options instead of editing `/usr/bin/mkarchiso`

### ISO-TD-006 - Inherited NVIDIA and generic hardware paths are still present

**State:** open
**Area:** live root cleanup / target scope

Evidence:

- [`archiso/airootfs/etc/modprobe.d/nvidia-loader.conf`](archiso/airootfs/etc/modprobe.d/nvidia-loader.conf) intercepts NVIDIA module loads
- [`archiso/airootfs/usr/local/bin/nvidia-module-loader`](archiso/airootfs/usr/local/bin/nvidia-module-loader) dynamically chooses nouveau/NVIDIA drivers
- [`archiso/airootfs/usr/local/bin/removeun`](archiso/airootfs/usr/local/bin/removeun) still removes NVIDIA packages conditionally
- The Mac Pro 6,1 target uses AMD FirePro GPUs

Why this matters:

- The ISO reads as Mac Pro-specific but still carries generic GPU-management behavior
- Some inherited cleanup may still be useful for CachyOS installer flows, so blind deletion is risky

Exit criteria:

- Audit whether Calamares/chwd/live boot still depends on these helpers
- Remove the NVIDIA path if unused, or document it as inherited generic fallback

### ISO-TD-007 - Release and issue links were historically tied to archived upstream

**State:** open
**Area:** documentation / release process

Evidence:

- The previous README pointed users at `wolffcatskyy/cachyos-macpro-iso` releases and issues
- The current README now treats those links as archival, but this fork still lacks a documented active release process

Why this matters:

- Users need a clear answer on whether they should download a prebuilt ISO or build locally
- Maintainers need a release checklist for split artifacts, checksums, signatures, and hardware-test notes

Exit criteria:

- Create an active release process for this fork, or
- State explicitly that no maintained prebuilt ISO is currently published

### ISO-TD-008 - Test coverage is generic installer automation, not Mac Pro hardware validation

**State:** open
**Area:** tests / validation

Evidence:

- [`testcases/cachyos/dailylive/test_install_calamares`](testcases/cachyos/dailylive/test_install_calamares) exercises a generic Calamares install flow in QEMU
- [`testiso.sh`](testiso.sh) launches the ISO in VirtualBox
- Neither path validates FirePro initialization, Apple EFI cold-boot behavior, Thunderbolt display behavior, or the real Mac Pro boot picker path

Why this matters:

- Passing tests can still miss the failures that matter most on this hardware
- Release notes need to distinguish VM smoke tests from hardware validation

Exit criteria:

- Add a validation matrix that separates build, VM boot, installer, and real Mac Pro hardware tests
- Capture hardware test logs for each released ISO

### ISO-TD-011 - Local smoke-test helper hardcodes VirtualBox host state

**State:** open
**Area:** local testing / host assumptions

Evidence:

- [`testiso.sh`](testiso.sh) hardcodes the VM name `CachyOS`
- It writes under `~/VirtualBox VMs/CachyOS/`
- It creates a fixed `10GB` VDI and fixed VM XML with static CPU/RAM/display/network assumptions
- It reads locale from `/etc/locale.conf` and attaches the first matching ISO found under the requested output folder

Why this matters:

- Running the helper can collide with an existing user VM named `CachyOS`
- The test is host-specific, not isolated, and not clearly tied to a particular ISO artifact
- Contributors can mistake this for a supported validation path even though it does not validate Mac Pro hardware

Exit criteria:

- Parameterize VM name, VM directory, disk size, memory, and ISO path
- Prefer disposable per-run VM names or quicktest/QEMU for automation
- Document `testiso.sh` as a local convenience helper only, or replace it with the supported test flow

### ISO-TD-012 - Build, test, and installer paths depend on moving external inputs

**State:** open
**Area:** reproducibility / external dependencies

Evidence:

- [`util-iso.sh`](util-iso.sh) fetches the CachyOS mirrorlist from the `CachyOS-PKGBUILDS` `master` branch during build
- [`.github/workflows/build.yml`](.github/workflows/build.yml) uses the floating `archlinux:base-devel` container tag and installs latest packages at workflow runtime
- The quicktest job builds `quickemu` from the AUR at current HEAD
- [`archiso/airootfs/usr/local/bin/calamares-online.sh`](archiso/airootfs/usr/local/bin/calamares-online.sh) installs `cachyos-calamares-next` at runtime before launching the installer

Why this matters:

- A CI/build/test result can change without any commit in this repository
- Reproducing an ISO after a package repo or AUR change becomes difficult

Exit criteria:

- Pin container images, AUR inputs, and downloaded helper files to reviewed versions where possible
- Record package database timestamps or artifact manifests for releases
- Decide whether runtime installer package refresh is required or should be replaced with packages already present in the ISO

### ISO-TD-013 - Inherited automated boot script execution needs a fork policy

**State:** open
**Area:** live ISO behavior / inherited archiso hooks

Evidence:

- [`archiso/airootfs/root/.automated_script.sh`](archiso/airootfs/root/.automated_script.sh) reads a `script=` kernel command-line parameter
- If the value is an HTTP/HTTPS/FTP/TFTP URL, it downloads it to `/tmp/startup_script` and executes it
- If the value is a local path, it copies and executes that path

Why this matters:

- This can be a useful archiso automation feature, but it is not documented as part of the Mac Pro profile contract
- Public rescue/live media should make boot-time remote code execution behavior explicit

Exit criteria:

- Decide whether the fork supports `script=` automation
- If yes, document the feature and its trust boundary
- If no, remove or disable the hook in the Mac Pro profile

## P3

### ISO-TD-009 - `CHANGELOG.md` is an inherited upstream CachyOS changelog

**State:** open
**Area:** documentation / release notes

Evidence:

- [`CHANGELOG.md`](CHANGELOG.md) tracks upstream CachyOS release notes
- It is not scoped to this Mac Pro fork or to the `linux-macpro61` profile changes

Why this matters:

- Readers can mistake upstream CachyOS changes for fork-specific release notes
- Real profile changes are hard to audit over time

Exit criteria:

- Rename or label the file as upstream reference, and
- Add a fork-specific `RELEASES.md` or release-notes section when actual ISO releases resume

### ISO-TD-010 - `.gitignore` has duplicated artifact patterns

**State:** open
**Area:** cleanup

Evidence:

- [`.gitignore`](.gitignore) repeats `build/`, `out/`, `*.iso`, and checksum/artifact-related patterns

Why this matters:

- Low impact, but it signals the file has accumulated without cleanup

Exit criteria:

- Deduplicate `.gitignore` while preserving all generated artifact protections

## Suggested Order

1. Resolve `ISO-TD-001`, `ISO-TD-002`, and `ISO-TD-003` first. They determine whether the ISO boots the intended kernel reproducibly.
2. Repair CI, host mutation, and moving inputs next: `ISO-TD-004`, `ISO-TD-005`, and `ISO-TD-012`.
3. Then audit inherited CachyOS/NVIDIA/test/automation paths: `ISO-TD-006`, `ISO-TD-008`, `ISO-TD-011`, and `ISO-TD-013`.
4. Finish with release documentation and cleanup: `ISO-TD-007`, `ISO-TD-009`, and `ISO-TD-010`.

## Explicitly Not Listed Here

These are constraints, not debt by themselves:

- Cold-boot requirement on Apple EFI
- Need for external `linux-macpro61` packages
- Keeping `linux-cachyos-lts` as an explicit fallback, if documented and bootable
- Using Calamares and CachyOS desktop packages as the live ISO base
- Standard Linux/archiso paths under `/etc`, `/usr`, `/boot`, and `/home/liveuser` when they are part of the live profile contract
