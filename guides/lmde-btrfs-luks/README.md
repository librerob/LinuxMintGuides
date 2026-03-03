# Installing **Linux Mint Debian Edition (LMDE)**

## with **Full Disk Encryption (LUKS)** + **Btrfs + Subvolumes**

This guide walks through setting up LMDE with LUKS full‑disk encryption and a Btrfs filesystem with separate subvolumes for `/` and `/home`.
It’s inspired by *Sindastra’s guide (April 2024)*. 

---

## 📌 Prerequisites

* A USB stick with **Linux Mint Debian Edition** live image written to it
* Bootable on your machine (UEFI)
* A **backup of any important data** — this process *wipes the target drive* 

---

## 🚀 1) Boot & Start Expert Installer

1. Boot into the LMDE live environment.

2. Open a terminal and run:

   ```bash
   sudo live-installer-expert-mode
   ```

3. Proceed through the GUI setup until **Installation Type**

4. Select **Manual Partitioning**

5. Enable **Expert Mode**

6. Open another terminal tab:

   ```bash
   sudo -i
   lsblk -o +FSTYPE
   ```

---

## 🖥 2) Note for Old MacBooks (if needed)

If the internal drive doesn’t show up on older Macs, at the boot menu press **e** and replace:

```
quiet splash
```

with:

```
intel_iommu=off
```

Then press **F10** to continue. ([Sindastra's info dump][1])

---

## 💥 3) Wiping & Creating Partitions

Assuming your install target is **/dev/sda**, wipe and create partitions:

```bash
gdisk /dev/sda
```

* Press `o` (create new empty GPT)
* Create partitions:

  * **EFI (ESP)** — +200M, hex code `ef00`
  * **/boot** — +8G, hex code `8300`
  * **Swap (optional)** — +5G, hex code `8300`
  * **LUKS container** — rest of disk, hex code `8309`
* Write with `w` and confirm

Then wipe old filesystem metadata:

```bash
wipefs -a /dev/sda1
wipefs -a /dev/sda2
wipefs -a /dev/sda3
wipefs -a /dev/sda4
```

> 🚨 This *permanently destroys* existing data on `/dev/sda`. Make sure this is the correct target drive. ([Sindastra's info dump][1])

---

## 📀 4) Format Partitions

### Create EFI and Boot

```bash
mkfs.vfat -c -F32 -n ESP /dev/sda1
mkfs.ext4 -L boot /dev/sda2
```

### Dummy swap filesystem (for later UUID use)

```bash
mkfs.ext2 -L cryptswap /dev/sda3 1M
```

### Format LUKS & Btrfs

```bash
cryptsetup luksFormat /dev/sda4
# Enter passphrase

cryptsetup open /dev/sda4 btrfs
mkfs.btrfs /dev/mapper/btrfs
```

---

## 🧱 5) Btrfs Subvolumes & Mounting

Mount and create subvolumes:

```bash
mount /dev/mapper/btrfs /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
umount /mnt
```

Then mount them under the install target:

```bash
mkdir -p /target
mount /dev/mapper/btrfs -o subvol=@,defaults,compress=zstd:1 /target
mkdir -p /target/home
mount /dev/mapper/btrfs -o subvol=@home,defaults,compress=zstd:1 /target/home
mkdir -p /target/boot
mount /dev/sda2 /target/boot
mkdir -p /target/boot/efi
mount /dev/sda1 /target/boot/efi
```

---

## 🛠 6) Continue Installation

Back in the installer:

* Click **Next**
* Select to install **GRUB** on `/dev/sda`
* Confirm everything looks good
* Proceed with installation

Enjoy a snack — this part takes a moment. ([Sindastra's info dump][1])

---

## 📦 7) Post‑Install Steps (After "Installation Paused")

Once the installer pauses:

### Update & install scripts

```bash
apt update
apt install arch-install-scripts
```

Generate an fstab:

```bash
genfstab -U /target > /target/etc/fstab
```

---

## 🔐 8) Enable Encrypted Mounts

Find the UUID for your LUKS device:

```bash
lsblk -o +label,fstype,UUID
```

Edit the crypttab:

```bash
nano /target/etc/crypttab
```

Example:

```
btrfs UUID=YOUROWNUUID none
swap   LABEL=cryptswap /dev/urandom swap,offset=2048,cipher=aes-xts-plain64,size=512
```

Save and exit.

Then edit `fstab` to add swap:

```
/dev/mapper/swap none swap defaults 0 0
```

---

## 🔁 9) Finish Installer

Return to the GUI:

* Hit **Next**
* Installer will finish locale, bootloader, etc.
* Reboot

If all was done correctly, the system should boot into the new encrypted LMDE install. ([Sindastra's info dump][1])

---

## 🖥 10) Fix Permanent MacBook Boot Quirk (if needed)

If you used `intel_iommu=off`, permanently add it:

```bash
sudo nano /etc/default/grub
```

Append `intel_iommu=off` inside:

```
GRUB_CMDLINE_LINUX_DEFAULT="… intel_iommu=off"
```

Update grub:

```bash
sudo update-grub
sudo update-grub2
```

---

## ✨ Final Notes

* You now have a lightweight, encrypted LMDE desktop with Btrfs and separate subvolumes. ([Sindastra's info dump][1])
* Optional aesthetic tweaks:

  * Install Libertinus fonts
  * Install grml-rescueboot (reason why boot parition is so big, so we can move LMDE live iso in there for live boot in grub menu to fix system if needed)

---
