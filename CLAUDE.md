# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`vps2arch` is a single POSIX shell script (`vps2arch`) that converts a running Linux VPS to Arch Linux in-place, without needing a reboot into an installer. It works by:

1. Downloading the Arch Linux bootstrap tarball and extracting it to `/root.<arch>/`
2. Mounting proc/sys/dev/the current root into the chroot
3. Using `ld.so` from the bootstrap chroot to invoke `chroot` itself — the only way to run commands after the host filesystem is wiped
4. Wiping all host files (except `/dev`, `/proc`, `/sys`, the bootstrap dir)
5. Installing Arch Linux packages into the original `/` via `pacstrap -M /mnt`
6. Configuring bootloader (grub/syslinux), network (systemd-networkd/netctl), and restoring the root password

## Testing

Tests run inside GitLab CI using QEMU. A Debian 10 cloud image is customized with `virt-customize`, booted, and the script is SCP'd in and executed. Success is checked by verifying `/etc/arch-release` exists after the VM reboots.

There is **no local test runner**. To manually test, you need a disposable VM or VPS. The CI pipeline (`.gitlab-ci.yml`) defines the test matrix:

- `test_default` — default options (grub + systemd-networkd)
- `test_netctl` — `-n netctl`
- `test_syslinux` — `-b syslinux`
- `test_default_uefi` — UEFI firmware (OVMF)

## Script Options

```
./vps2arch [-b grub|syslinux|none] [-n systemd-networkd|netctl|none] [-m mirror]...
```

Defaults: bootloader=`grub`, network=`systemd-networkd`. OpenVZ forces `bootloader=none` and `network=netctl`. LXC forces `bootloader=none`.

## Shell Style

- POSIX `sh` (shebang `#!/bin/sh`), not bash
- `set -e` at top; functions use `local` for variables
- Mirror selection uses the Arch mirrorlist API with country auto-detection
- The "black magic" line at `install_packages` uses the bootstrap's `ld-*.so.2` to invoke `chroot` after the host system is deleted
