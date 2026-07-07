# CachyOS Mac Pro 6,1 ISO Profile

Custom CachyOS live ISO profile for the Mac Pro 6,1 (Late 2013). It is the CachyOS/Arch live-media side of the workspace: it expects `linux-macpro61` packages from the sibling kernel project and layers Mac Pro 6,1 boot defaults, amdgpu Southern Islands parameters, reboot protection, and installer conveniences on top of a CachyOS desktop ISO.

> Historical upstream note: the original `wolffcatskyy/cachyos-macpro-iso` and `wolffcatskyy/linux-mac` projects were archived on March 10, 2026. Treat this repository (`ishad0w/linux-trash-can-cachyos-iso`), the sibling [`linux-trash-can`](https://github.com/ishad0w/linux-trash-can), and the docs in this tree as the maintained source of truth.

## What This Is

This repository is an `archiso` profile plus build/test helpers. It does not build the kernel itself. The expected input is a local pacman repository containing:

- `linux-macpro61-*.pkg.tar.zst`
- `linux-macpro61-headers-*.pkg.tar.zst`

Those packages come from the sibling kernel repository's [`packaging/arch`](../linux-trash-can/packaging/arch) path. The ISO then boots with `linux-macpro61` where the active boot loader path uses the Mac Pro entry.

## Hardware Target

| Area | Current profile behavior |
|------|------|
| CPU | Ivy Bridge-EP Xeon family used by Mac Pro 6,1 |
| GPU | AMD FirePro D300/D500/D700 via `amdgpu` with `amdgpu.si_support=1` and `amdgpu.dc=0` |
| Storage | Apple AHCI SSD plus common aftermarket NVMe upgrades |
| Networking | Broadcom ethernet, NetworkManager, installer networking |
| Desktop | CachyOS Plasma live desktop with Calamares |
| Cold boot | `reboot` alias points to `poweroff`; `reboot.target` is masked in the live root |

## Current Boot Story

The intended live-boot path is:

```text
local-repo/linux-macpro61*.pkg.tar.zst
    -> buildiso.sh generates local-repo/macpro.db
    -> build/archiso/pacman.conf [macpro]
    -> archiso/packages_desktop.x86_64
    -> archiso/grub/grub.cfg or archiso/syslinux/archiso_sys-linux.cfg
    -> live ISO boots vmlinuz-linux-macpro61
```

GRUB, BIOS syslinux, systemd-boot, loopback, and PXE now have explicit Mac Pro 6,1 primary entries. CachyOS LTS remains installed as an intentional fallback/safe path.

## Build

### 1. Build or obtain the kernel packages

From the sibling kernel repository:

```bash
cd ../linux-trash-can/packaging/arch
makepkg -s
```

Copy the generated kernel and headers packages into this repository:

```bash
cd ../../../linux-trash-can-cachyos-iso
mkdir -p local-repo
cp ../linux-trash-can/packaging/arch/linux-macpro61-*.pkg.tar.zst local-repo/
cp ../linux-trash-can/packaging/arch/linux-macpro61-headers-*.pkg.tar.zst local-repo/
```

`buildiso.sh` creates or refreshes `local-repo/macpro.db` automatically before `mkarchiso`. To create it manually, run:

```bash
cd local-repo
repo-add macpro.db.tar.gz *.pkg.tar.zst
cd ..
```

### 2. Point pacman at the local repo

By default, [`buildiso.sh`](buildiso.sh) uses `./local-repo` and rewrites the `[macpro]` `Server` line in the copied build profile at `build/archiso/pacman.conf`. The committed [`archiso/pacman.conf`](archiso/pacman.conf) contains a placeholder so direct `mkarchiso` calls do not accidentally depend on a developer-local path.

To use a different package repository, set `MACPRO_LOCAL_REPO`:

```bash
MACPRO_LOCAL_REPO=/absolute/path/to/local-repo sudo -E ./buildiso.sh -p desktop -v -w
```

The repo must contain both `linux-macpro61` and `linux-macpro61-headers` packages. If either package is missing, the build stops before `mkarchiso` with a focused error.

### 3. Prepare CachyOS trust on plain Arch

On a CachyOS build host this is normally already handled. On a plain Arch host, seed the CachyOS signing key the same way the workflow does:

```bash
sudo pacman-key --init
sudo pacman-key --recv-keys F3B607488DB35A47 --keyserver keyserver.ubuntu.com
sudo pacman-key --lsign-key F3B607488DB35A47
```

### 4. Build the ISO

On an Arch Linux or CachyOS build host:

```bash
sudo pacman -S --needed archiso mkinitcpio-archiso squashfs-tools grub git
sudo ./buildiso.sh -p desktop -v -w
```

Artifacts are written under `out/desktop/`.

## Boot

1. Write the ISO to USB.
2. Fully power off the Mac Pro 6,1.
3. Press the power button and immediately hold Option.
4. Select the USB drive.
5. In GRUB, choose `CachyOS (Mac Pro 6,1)`.

Always power off instead of warm rebooting when switching kernels. Apple EFI often leaves the FirePro GPUs uninitialized after warm reboot, which can produce a black screen.

## Documentation

- [`MAP.md`](MAP.md) - repository map, change surfaces, and file purposes
- [`AGENTS.md`](AGENTS.md) - working rules for humans and automation
- [`TECH-DEBT.md`](TECH-DEBT.md) - confirmed stale, partial, or risky areas
- [`docs/upstream-sync-2026-07-08.md`](docs/upstream-sync-2026-07-08.md) - latest upstream merge notes and retained fork behavior
- [`CHANGELOG.md`](CHANGELOG.md) - inherited upstream CachyOS changelog snapshot, not a Mac Pro specific release log
- [`../linux-trash-can/README.md`](../linux-trash-can/README.md) - sibling kernel project overview
- [`../linux-trash-can/MAP.md`](../linux-trash-can/MAP.md) - kernel project map

## Current Caveats

- `local-repo/` must still be populated before the ISO can build; `buildiso.sh` only validates it, refreshes `macpro.db`, and rewrites the build-time pacman path.
- CachyOS LTS remains present as a fallback; primary live boot validation must confirm that the selected default path is `linux-macpro61`.
- `ci.build.sh` is stale and references scripts that are not in this repository.
- The GitHub workflow now checks out `ishad0w/linux-trash-can` and builds `linux-macpro61` packages before the ISO step; this still needs a successful CI run before treating it as a proven release path.
- ISO checksums are always created after a successful build. Detached signatures are created only when a GPG secret key is available; CI artifacts can be unsigned unless signing secrets are configured.
- This repo carries inherited CachyOS live ISO helpers, NVIDIA cleanup code, VM guest services, and generic test flows that have not all been re-audited for the Mac Pro-only target.

Use [`MAP.md`](MAP.md) to find the right file before changing a boot path or package list. Use [`TECH-DEBT.md`](TECH-DEBT.md) before deleting inherited CachyOS pieces.
