# Linux Mint Guides
A collection of personal installation and configuration guides for **Linux Mint**, **LMDE**, and related distros focused on encrypted root (LUKS), Btrfs filesystems, secure setups, and system configuration.
> Warning: Some guides involve **wiping drives entirely**. Always back up your data before following any installation guide.
---
## Guides
### [Linux Mint & Ubuntu Derivatives - Btrfs Root + LUKS2](guides/LinuxMint-Zorin-Btrfs-Root-Luks2.md)
Install **Linux Mint** or other Ubuntu-based derivatives with LUKS2 encrypted root on a Btrfs filesystem.
- UEFI + GPT partitioning with `parted`
- LUKS2 with Argon2id key derivation
- Btrfs with `@` and `@home` subvolumes
- zram swap (no swap partition)
- Timeshift snapshots + grml-rescueboot for recovery
---
### [LMDE + LUKS + Btrfs + Subvolumes](guides/LMDE-BTRFS-LUKS.md)
Install **Linux Mint Debian Edition (LMDE)** with encrypted root (LUKS) and a Btrfs filesystem using separate `@` and `@home` subvolumes.
- UEFI + GPT partitioning with `gdisk`
- LUKS container + Btrfs on top
- Notes for older MacBooks (`intel_iommu=off`)
- grml-rescueboot integration (bootable ISO in GRUB)
---
### [Ubuntu & Derivatives - Auto-unlock Secondary LUKS Drive at Boot](guides/Ubuntu-Secondary-LUKS.md)
Set up automatic unlocking of a **secondary LUKS-encrypted hard drive** at boot on Ubuntu, Linux Mint, and other Ubuntu-based derivatives, using `/etc/crypttab` and a keyfile.
- Works with both ext4 and btrfs filesystems on the secondary drive
- Keyfile stored on the encrypted root — unlocks automatically after main drive is decrypted
---
### [Ubuntu & Derivatives - Post-Install Guide](guides/ubuntu-post-install.md)
Everything to do right after a fresh Ubuntu, Linux Mint, or Ubuntu-based install.
- Kernel hardening and system tuning via `sysctl`
- Cloudflare DNS over TLS with `systemd-resolved`
- Random MAC address at every boot via NetworkManager
- UFW firewall setup (including QEMU/Virt-Manager support)
- Disable unnecessary services (CUPS, Bluetooth)
- Recommended software (KeePassXC, Zsh, Virt-Manager, and more)
- **Brave Browser** — install from the official apt repo + managed policy file to disable AI, telemetry, crypto/rewards bloat, and other extras; configurable DoH provider (Cloudflare, Quad9, AdGuard, or OS default)
- Cinnamon desktop tweaks (Linux Mint)
- Font configuration — rendering quality, metric-compatible MS font substitutions, and recommended system fonts (Inter, JetBrains Mono, Linux Libertine)
- LibreOffice font setup for maximum MS Office document compatibility
---
### [Encrypted USB Backup Drive Setup (LUKS2 + ext4)](guides/Encrypted-USB-Backup-LUKS2.md)
Format a USB drive with strong LUKS2 encryption using argon2id — consistent with a hardened Linux system drive setup.
- LUKS2 with Argon2id KDF (`--iter-time 4000`)
- ext4 filesystem inside the encrypted container
- Drive labelling and permission fixes for rsync/file managers
- Everyday mount/unmount workflow with GNOME Files
---
## About
These are personal reference guides collected while setting up and configuring Linux systems. Shared in case they help others doing similar work.
