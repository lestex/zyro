# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

`omarchy-iso` builds a bootable Arch Linux ISO that automates the installation of [Omarchy](https://github.com/basecamp/omarchy). The ISO runs an interactive TUI configurator (keyboard, user, disk), then hands off to `archinstall` for the actual installation.

## Key Commands

```bash
# Build the ISO (outputs to ./release/)
./bin/omarchy-iso-make

# Build without package cache (fresh download)
./bin/omarchy-iso-make --no-cache

# Build against local Omarchy source ($OMARCHY_PATH must be set)
./bin/omarchy-iso-make --local-source

# Build using the dev branch of the Omarchy installer
./bin/omarchy-iso-make --dev

# Build using the rc branch
./bin/omarchy-iso-make --rc

# Boot a built ISO in QEMU (UEFI, KVM, virtio-vga)
./bin/omarchy-iso-boot [release/omarchy.iso]

# Save/restore/list QEMU VM snapshots
./bin/omarchy-vm save [name]
./bin/omarchy-vm boot [name]
./bin/omarchy-vm list

# Full release: build, sign, upload
./bin/omarchy-iso-release [version]

# Individual release steps
./bin/omarchy-iso-sign [release/omarchy.iso]
./bin/omarchy-iso-upload [release/omarchy.iso]
```

### Environment Variable Overrides

```bash
OMARCHY_INSTALLER_REPO="myuser/omarchy-fork" OMARCHY_INSTALLER_REF="some-feature" ./bin/omarchy-iso-make
```

## Architecture

### Build Pipeline

1. **`bin/omarchy-iso-make`** — Entry point on the developer's machine. Runs a Docker container (`archlinux/archlinux:latest`) with the repo directories bind-mounted as read-only. Assembles the ISO inside the container and outputs to `./release/`.

2. **`builder/build-iso.sh`** — Runs inside the Docker container. Steps:
   - Installs build tools (archiso, git, grub, etc.) inside the container
   - Copies the upstream Arch `releng` profile from the `archiso` submodule as the base
   - Overlays `configs/` on top of the base profile
   - Clones (or mounts) the Omarchy installer into `airootfs/root/omarchy`
   - Downloads and verifies the latest Node.js binary for offline use
   - Downloads all required packages (ISO packages + Omarchy installer packages + archinstall packages) into an offline pacman repo inside the ISO (`/var/cache/omarchy/mirror/offline/`)
   - Calls `mkarchiso` to assemble the final ISO

3. **`archiso/`** — Git submodule tracking the upstream Arch Linux `archiso` tool. Provides the `releng` base profile and the `mkarchiso` tool.

### Configs Overlay (`configs/`)

Everything in `configs/` is overlaid onto the `releng` base profile before `mkarchiso` runs:

- **`profiledef.sh`** — ISO metadata (name, label, bootmodes, file permissions). Declares the `[/root/configurator]` script executable.
- **`pacman-offline.conf`** — pacman config used by the live ISO environment; points only to the bundled offline mirror.
- **`pacman-online-{stable,edge,rc}.conf`** — pacman configs used during build to download packages; point to Omarchy's CDN mirrors and the `[omarchy]` custom repo.
- **`airootfs/root/configurator`** — The Omarchy Configurator TUI script. This is what users interact with when they boot the ISO. It collects keyboard layout, username/password, hostname, timezone, and target disk, then generates `user_configuration.json` and `user_credentials.json` for `archinstall`.
- **`airootfs/`** — Files overlaid into the live ISO filesystem. Plymouth theme setup, mkinitcpio configs, etc.
- **`builder/archinstall.packages`** — Additional packages downloaded into the offline mirror for use by `archinstall` during installation.

### Configurator Flow

The `configs/airootfs/root/configurator` bash script is the interactive installer frontend:
1. Keyboard layout selection (`loadkeys`)
2. Username, password (hashed with yescrypt via `openssl passwd -6`), full name, email, hostname, timezone
3. Disk selection (excludes the boot media; lists partitions for context)
4. Detects T2 Macs via PCI ID `106b:1801/1802` and selects `linux-t2` kernel instead of `linux`
5. Generates `user_configuration.json` (archinstall config with LUKS encryption, Btrfs with Snapper, Limine bootloader) and `user_credentials.json`
6. The `.automated_script.sh` (from the `releng` base) picks these up and calls `archinstall`

### Package Mirrors

Three mirror tiers controlled by `$OMARCHY_MIRROR` (stable/edge/rc):
- **stable** — `stable-mirror.omarchy.org`
- **edge** — `edge-mirror.omarchy.org`
- **rc** — `rc-mirror.omarchy.org`
- Custom `[omarchy]` repo at `pkgs.omarchy.org`
- T2 Mac packages from `arch-mact2` GitHub releases mirror

### Caching

`omarchy-iso-make` mounts a dated host directory (`~/.cache/omarchy/iso_YYYY-MM-DD/`) into the Docker container as the package cache. This avoids re-downloading packages on same-day rebuilds. Use `--no-cache` to skip it. The host's `/var/cache/pacman/pkg` is also mounted if present.

### VM Testing

`bin/omarchy-iso-boot` runs QEMU with KVM, UEFI (edk2/OVMF), virtio-vga-gl, and a 40 GB qcow2 disk at `/tmp/omarchy-iso-boot.qcow2`. SSH is available at `localhost:2222`. `bin/omarchy-vm` wraps save/restore of the `/tmp` VM state to `./vm-saves/`.

### Upload

Uploading uses `rclone` to an R2 bucket (`Omarchy:omarchy/`). Configure with `bin/omarchy-iso-rclone-config`. Signing creates a detached GPG sig alongside the ISO.
