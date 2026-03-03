# 🐧 Linux Mint Guides

A collection of personal installation and configuration guides for **Linux Mint**,
**LMDE**, and related distros — focused on encrypted root (LUKS), Btrfs
filesystems, and secure setups.

> ⚠️ **Warning:** Most guides involve **wiping drives entirely**. Always back up
> your data before following any installation guide.

---

## 📚 Guides

### 🔐 [LMDE + LUKS + Btrfs + Subvolumes](guides/LMDE-BTRFS-LUKS.md)

Install **Linux Mint Debian Edition (LMDE)** with encrypted root (LUKS) and a
Btrfs filesystem using separate `@` and `@home` subvolumes.

- UEFI + GPT partitioning with `gdisk`
- LUKS container + Btrfs on top
- Encrypted swap via `/etc/crypttab`
- Notes for older MacBooks (`intel_iommu=off`)
- grml-rescueboot integration (bootable ISO in GRUB)

---

### 🔐 [Ubuntu & Derivatives — Auto-unlock Secondary LUKS Drive at Boot](guides/Ubuntu-Secondary-LUKS.md)

Set up automatic unlocking of a **secondary LUKS-encrypted hard drive** at boot
on Ubuntu, Linux Mint, and other Ubuntu-based derivatives, using `/etc/crypttab`
and a keyfile.

---

### 🔐 [Linux Mint & Ubuntu Derivatives — Btrfs Root + LUKS2](guides/LinuxMint-Zorin-Btrfs-Root-Luks2.md)

Install **Linux Mint** or other Ubuntu-based derivatives with LUKS2 encrypted
root on a Btrfs filesystem.

---

## ℹ️ About

These are personal reference guides collected while setting up encrypted Linux
systems. Shared in case they help others doing similar installs.
