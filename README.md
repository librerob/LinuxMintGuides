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
- Disable unnecessary services (CUPS, Bluetooth, OpenVPN, ModemManager, kerneloops, casper-md5check)
- Remove unwanted packages (warpinator, Samba, Rhythmbox, Hypnotix, and more)
- Recommended software (Brave, KeePassXC, Zsh, Virt-Manager, and more)
- KVM group setup and Windows 10 VM XML reference
- Cinnamon desktop tweaks (Linux Mint)

---

### [Linux - Set Default Fonts and Font Aliases with Fontconfig](guides/linux-fontconfig)

Configure system and web fonts on Linux using **fontconfig** so preferred fonts are used consistently across all applications and websites.

- Install metric-compatible replacements for Arial, Times New Roman, Calibri, and more
- Alias common Microsoft font names to open font equivalents (`fonts-croscore`, `fonts-crosextra-*`)
- Add Noto fallbacks for full Unicode and emoji support
- Verify font substitution with `fc-match`
- Per-user (`~/.config/fontconfig/fonts.conf`) and system-wide options

---

### [Flat Remix GTK Theme — Cinnamon Patch](guides/gtk-theme/)

A patched version of the [Flat Remix GTK theme](https://www.gnome-look.org/p/1214931/) customised for **Linux Mint Cinnamon** — Blue highlights, Darkest Solid (AMOLED pitch black).

- Fixes unstyled and broken dialogs introduced in newer Cinnamon versions
- GTK4 app coverage via `~/.config/gtk-4.0` symlink
- Flatpak support
- Pairs with Papirus Dark or the matching Flat Remix icon theme

---

## About

These are personal reference guides collected while setting up and configuring Linux systems. Shared in case they help others doing similar work.
