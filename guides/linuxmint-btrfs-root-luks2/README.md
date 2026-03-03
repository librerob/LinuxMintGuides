# BTRFS + LUKS2 Installation Guide (Root-Only Encryption)
### Linux Mint / Zorin OS — Ubiquity installer — Timeshift snapshots + grml-rescueboot

> **Based on** [mutschler.dev's Ubuntu 20.04 btrfs guide](https://mutschler.dev/linux/ubuntu-btrfs-20-04/), adapted for:
> - **LUKS2** with Argon2id key derivation instead of LUKS1
> - **Root-only encryption** — `/boot` is a separate unencrypted partition, which is what makes LUKS2 possible
> - **6 GiB `/boot` partition** — sized to accommodate a live ISO for [grml-rescueboot](https://github.com/grml/grml-rescueboot)
> - **Mount options:** `defaults,compress=zstd:1`

---

## Overview

This guide installs Linux Mint or Zorin OS with the following disk structure:

- An **unencrypted 512 MiB FAT32 EFI partition** for the GRUB bootloader
- An **unencrypted 6 GiB ext4 `/boot` partition** — kept large enough to store a live ISO for grml-rescueboot, which lets you boot into a live environment directly from GRUB and use it to restore Timeshift snapshots on an encrypted system
- A **4 GiB swap partition** (encrypted at runtime via crypttab with a random key)
- A **LUKS2 encrypted partition** for the root btrfs filesystem, containing:
  - Subvolume `@` mounted at `/`
  - Subvolume `@home` mounted at `/home`
- Automatic system snapshots using [Timeshift](https://github.com/linuxmint/timeshift), with rollback done by booting the live ISO via grml-rescueboot

### Why LUKS2 instead of LUKS1?

The GRUB bootloader shipped with Linux Mint and Zorin OS cannot decrypt LUKS2 partitions. The original mutschler.dev guide works around this by using LUKS1, which GRUB does understand. This guide takes a different approach: by placing `/boot` on its own unencrypted partition, GRUB never needs to touch LUKS at all. It reads the kernel and initramfs directly from the unencrypted `/boot`, and then the initramfs handles decrypting the LUKS2 root partition before handing control to the OS. This means:

- LUKS2 is fully supported with no GRUB limitations to work around
- Argon2id key derivation is used instead of LUKS1's PBKDF2, which is far more resistant to brute-force attacks
- GRUB configuration is simpler — no `GRUB_ENABLE_CRYPTODISK` required

The only tradeoff is that the contents of `/boot` (your kernel and initramfs) are readable by someone with physical access to the drive. For the vast majority of users this is an acceptable tradeoff, and the stronger LUKS2 encryption on root more than compensates.

### Why a 6 GiB `/boot`?

A typical `/boot` partition only needs a few hundred megabytes for kernels and initramfs files. We size it at 6 GiB here to leave room for a live rescue ISO (such as a Mint or Grml ISO) used by [grml-rescueboot](https://github.com/grml/grml-rescueboot). grml-rescueboot adds a GRUB entry that boots directly from an ISO file placed in `/boot/grml/`, giving you a fully working live environment from which you can unlock your LUKS2 partition, mount your btrfs subvolumes, and restore a Timeshift snapshot — without needing a separate USB drive.

### Compatibility

This guide uses the **Ubiquity installer**, which is still the default installer in:

- Linux Mint 21.x and 22.x
- Zorin OS 16 and 17

---

## Step 0: General Remarks

**Strongly recommended: test this entire procedure in a virtual machine before running it on real hardware.**

You can use any VM software (VirtualBox, GNOME Boxes, virt-manager, etc.) with a 64 GB virtual disk and 4+ GB RAM to follow along safely.

Boot your installation USB in **UEFI mode**. This guide covers UEFI only. If you see a screen asking you to choose between UEFI and Legacy/BIOS boot, choose UEFI.

---

## Step 1: Boot the Install, Check UEFI Mode, and Open a Root Shell

Boot your installation USB, choose your language, and click **Try** (not "Install"). Once the live desktop has loaded, open a terminal with `Ctrl+Alt+T`.

If you have a non-English keyboard layout, set it first: go to **Settings → Region & Language** and configure your keyboard before continuing.

Verify you are booted in UEFI mode:

```bash
mount | grep efivars
# efivarfs on /sys/firmware/efi/efivars type efivarfs (rw,nosuid,nodev,noexec,relatime)
```

If you see output similar to the above, you are in UEFI mode and can proceed. Now switch to an interactive root session:

```bash
sudo -i
```

Keep this terminal open for the entire installation. Do not close it until we are completely finished.

---

## Step 2: Prepare Partitions Manually

### Identify Your Target Disk

```bash
lsblk
# NAME  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
# loop0   7:0    0   1.9G  1 loop /rofs
# loop1   7:1    0  27.1M  1 loop /snap/snapd/7264
# ...
# vda   252:0    0    64G  0 disk
```

You can also open GParted or look in `/dev` to confirm. Common disk names are `sda` for SATA SSDs and HDDs, and `nvme0n1` for NVMe drives.

**Replace `vda` with your actual disk name in every command throughout this guide.**

### Create Partition Table and Layout

We will create 4 partitions:

| # | Size | Role | Filesystem |
|---|------|------|------------|
| vda1 | 512 MiB | EFI System Partition | FAT32 |
| vda2 | 6 GiB | `/boot` (unencrypted, holds rescue ISO) | ext4 |
| vda3 | 4 GiB | swap | encrypted at runtime |
| vda4 | remainder | LUKS2 encrypted root | btrfs inside LUKS2 |

```bash
parted /dev/vda
  mklabel gpt
  mkpart primary 1MiB 513MiB
  mkpart primary 513MiB 6657MiB
  mkpart primary 6657MiB 10753MiB
  mkpart primary 10753MiB 100%
  print
  quit
```

Your output should look something like:

```
Number  Start     End       Size    File system  Name     Flags
 1      1049kB    538MB     537MB                primary
 2      538MB     6980MB    6442MB               primary
 3      6980MB    11.3GB    4295MB               primary
 4      11.3GB    68.7GB    57.4GB               primary
```

> **Do not set partition names or flags.** The Ubiquity installer has known issues with named GPT partitions and may fail or behave unexpectedly if names or flags are set.

### Create the LUKS2 Encrypted Partition

```bash
cryptsetup luksFormat --type=luks2 \
  --pbkdf argon2id \
  --iter-time 4000 \
  /dev/vda4
# WARNING!
# ========
# This will overwrite data on /dev/vda4 irrevocably.
# Are you sure? (Type uppercase yes): YES
# Enter passphrase for /dev/vda4:
# Verify passphrase:
```

Use a strong passphrase. The flags explained:

- `--type=luks2` — use LUKS version 2
- `--pbkdf argon2id` — use Argon2id key derivation, which is memory-hard and far more resistant to GPU-based brute-force than LUKS1's PBKDF2
- `--iter-time 4000` — spend ~4 seconds deriving the key (default is ~2 seconds), making offline brute-force attacks twice as expensive. Since you only type this passphrase once at boot, the extra 2 seconds is not noticeable in practice.

Now open (unlock) the encrypted partition and map it to a name we can use:

```bash
cryptsetup luksOpen /dev/vda4 cryptdata
# Enter passphrase for /dev/vda4:

ls /dev/mapper/
# control  cryptdata
```

### Create Filesystems

```bash
# EFI partition — FAT32
mkfs.fat -F32 /dev/vda1

# /boot partition — unencrypted ext4
mkfs.ext4 /dev/vda2

# Root btrfs filesystem inside the LUKS2 container
mkfs.btrfs /dev/mapper/cryptdata
# btrfs-progs v5.x.x
# See http://btrfs.wiki.kernel.org for more information.
# Label:              (null)
# UUID:               xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
# Node size:          16384
# Sector size:        4096
# Filesystem size:    xx.xxGiB
# ...
```

We pre-format `cryptdata` as btrfs now because the Ubiquity installer can complain about unmounted devices otherwise.

---

## Step 3 (Optional): Optimize btrfs Mount Options Before Installing

The Ubiquity installer does not write optimal btrfs mount options. We pre-edit two files in the live environment so the correct options end up in the installed system's `/etc/fstab`.

We use `defaults,compress=zstd:1` as our mount options. `zstd:1` means zstd compression at level 1 — fast, effective, and low on CPU overhead.

Edit the first file:

```bash
nano /usr/lib/partman/mount.d/70btrfs
```

Find and update these two lines (around lines 24 and 31):

```bash
# line 24:
options="${options:+$options,}subvol=@,defaults,compress=zstd:1"

# line 31:
options="${options:+$options,}subvol=@home,defaults,compress=zstd:1"
```

Edit the second file:

```bash
nano /usr/lib/partman/fstab.d/btrfs
```

Update these lines:

```bash
# line 30: pass=0
# line 31: home_options="${options:+$options,}subvol=@home,defaults,compress=zstd:1"
# line 32: options="${options:+$options,}subvol=@,defaults,compress=zstd:1"
# line 36: pass=0
# line 37: options="${options:+$options,}subvol=@home,defaults,compress=zstd:1"
# line 40: pass=0
# line 56: echo "$home_path" "$home_mp" btrfs "$home_options" 0 0
```

> **OBS:** There is a known Ubuntu/Mint bug where modifying the partman scripts can sometimes cause Ubiquity to fail mid-install. It is not common, but it does happen. The safe fallback is to skip Step 3 entirely, let the installer finish, and then correct the mount options manually in `/etc/fstab` during Step 5 in the chroot. It makes no practical difference to the end result since no user data gets written during the install anyway.

---

## Step 4: Install Using the Ubiquity Installer (Without Bootloader)

Run the installer with the `--no-bootloader` flag. We skip the bootloader here because we will install and configure GRUB manually in the chroot after the install completes:

```bash
ubiquity --no-bootloader
```

Work through the language, keyboard, and installation type screens as normal. When you reach **Installation Type**, choose **Something Else** to open the manual partitioner.

Assign partitions as follows:

- Select **`/dev/vda1`** → press **Change** → set **Use as**: `EFI System Partition`
- Select **`/dev/vda2`** → press **Change** → set **Use as**: `ext4 journaling filesystem`, check **Format the partition**, set **Mount point**: `/boot`
- Select **`/dev/vda3`** → press **Change** → set **Use as**: `swap area`
- Select **`/dev/mapper/cryptdata`** (shown as type btrfs) → press **Change** → set **Use as**: `btrfs journaling filesystem`, check **Format the partition**, set **Mount point**: `/`

> If you see other existing EFI partitions on the disk (e.g. from a previous OS), deactivate or leave them unassigned so the installer does not interfere with them.

> If you do not assign a swap partition, Ubiquity will create a swapfile. On btrfs, swapfiles must live in a dedicated subvolume or they will break snapshots. See **Option B** in the Encrypted Swap section below if this applies to you.

Double-check your partition assignments, press **Install Now**, and confirm the changes. Fill in your timezone, username, and password. When the installer finishes, choose **Continue Testing** — **do not reboot yet**.

---

## Step 5: Post-Installation Steps

### Create a chroot Environment and Enter Your System

Return to your root terminal and mount the newly installed system:

```bash
mount -o subvol=@,defaults,compress=zstd:1 /dev/mapper/cryptdata /mnt
```

Bind-mount the virtual filesystems:

```bash
for i in /dev /dev/pts /proc /sys /run; do sudo mount -B $i /mnt$i; done
```

Copy DNS configuration so the chroot has network access:

```bash
sudo cp /etc/resolv.conf /mnt/etc/
```

Enter the chroot:

```bash
sudo chroot /mnt
```

You are now operating inside your newly installed system. Mount all remaining partitions and verify:

```bash
mount -av
# /                        : ignored
# /boot                    : successfully mounted
# /boot/efi                : successfully mounted
# /home                    : successfully mounted
# none                     : ignored
```

Check the btrfs subvolumes:

```bash
btrfs subvolume list /
# ID 256 gen 164 top level 5 path @
# ID 258 gen 30 top level 5 path @home
```

The subvolume `@` is mounted at `/` and `@home` at `/home`. This is correct.

### Verify /etc/fstab

```bash
cat /etc/fstab
```

You should see entries along these lines:

```
/dev/mapper/cryptdata  /         btrfs  defaults,subvol=@,compress=zstd:1      0  0
/dev/mapper/cryptdata  /home     btrfs  defaults,subvol=@home,compress=zstd:1  0  0
UUID=xxxx-xxxx         /boot     ext4   defaults                                0  2
UUID=xxxx-xxxx         /boot/efi vfat   umask=0077                              0  1
/dev/mapper/cryptswap  none      swap   sw                                      0  0
```

If the mount options are wrong, correct them now:

```bash
nano /etc/fstab
```

### Create crypttab

The `crypttab` file tells the initramfs which LUKS partitions to unlock at boot:

```bash
export UUIDVDA4=$(blkid -s UUID -o value /dev/vda4)
echo "cryptdata UUID=${UUIDVDA4} none luks" >> /etc/crypttab

cat /etc/crypttab
# cryptdata UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx none luks
```

> **Important:** The UUID must be from `/dev/vda4` (the LUKS container), not from `/dev/mapper/cryptdata` (the unlocked device inside it). You can list all UUIDs with `blkid`. The `luks` option works correctly with both LUKS1 and LUKS2.

### Encrypted Swap

#### Option A: Encrypted Swap Partition

Since we don't need hibernation, we can encrypt the swap partition with a fresh random key on every boot:

```bash
export SWAPUUID=$(blkid -s UUID -o value /dev/vda3)
echo "cryptswap UUID=${SWAPUUID} /dev/urandom swap,offset=1024,cipher=aes-xts-plain64,size=512" >> /etc/crypttab

cat /etc/crypttab
# cryptdata  UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx none luks
# cryptswap  UUID=yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy /dev/urandom swap,offset=1024,cipher=aes-xts-plain64,size=512
```

Update fstab so the swap entry points to the encrypted device:

```bash
sed -i "s|UUID=${SWAPUUID}|/dev/mapper/cryptswap|" /etc/fstab

cat /etc/fstab
# ...
# /dev/mapper/cryptswap  none  swap  sw  0  0
```

#### Option B: Swapfile in a Dedicated btrfs Subvolume

If you skipped the swap partition, Ubiquity will have created a swapfile. We need to move it into its own dedicated btrfs subvolume because a swapfile inside the `@` subvolume will prevent snapshots from working correctly.

Remove the existing swapfile:

```bash
swapoff /swapfile
rm /swapfile
sed -i '\|^/swapfile|d' /etc/fstab
```

Mount the top-level btrfs filesystem (subvolume ID 5, always present):

```bash
mkdir /btrfs_pool
mount -o subvolid=5 /dev/mapper/cryptdata /btrfs_pool
ls /btrfs_pool
# @  @home
```

Create a dedicated `@swap` subvolume:

```bash
btrfs subvolume create /btrfs_pool/@swap
ls /btrfs_pool
# @  @home  @swap
```

Create a 4 GiB swapfile with the correct btrfs properties (adjust size to your needs):

```bash
truncate -s 0 /btrfs_pool/@swap/swapfile
chattr +C /btrfs_pool/@swap/swapfile
btrfs property set /btrfs_pool/@swap/swapfile compression none
fallocate -l 4G /btrfs_pool/@swap/swapfile
chmod 600 /btrfs_pool/@swap/swapfile
mkswap /btrfs_pool/@swap/swapfile
# Setting up swapspace version 1, size = 4 GiB (4294963200 bytes)
# no label, UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Create the mount point inside `@`:

```bash
mkdir /btrfs_pool/@/swap
```

Add entries to fstab:

```bash
echo "UUID=$(blkid -s UUID -o value /dev/mapper/cryptdata)  /swap  btrfs  defaults,subvol=@swap,compress=zstd:1  0  0" >> /etc/fstab
echo "/swap/swapfile  none  swap  sw  0  0" >> /etc/fstab
```

Unmount the pool:

```bash
umount /btrfs_pool
```

### Install the EFI Bootloader

Because `/boot` is unencrypted, this is a completely standard GRUB installation — no special crypto flags are needed:

```bash
grub-install --target=x86_64-efi \
  --efi-directory=/boot/efi \
  --bootloader-id=ubuntu \
  --recheck
```

> **Note:** You do **not** need `GRUB_ENABLE_CRYPTODISK=y` in `/etc/default/grub`. That setting is only required when `/boot` itself lives inside a LUKS container. Here, GRUB reads `/boot` from the plain ext4 partition and never interacts with LUKS at all.

Update GRUB:

```bash
update-grub
```

Rebuild the initramfs so it picks up the crypttab configuration:

```bash
update-initramfs -u -k all
```

---

## Step 6: Reboot, Checks, and Update

Exit the chroot:

```bash
exit
```

Unmount everything cleanly:

```bash
for i in /dev/pts /dev /proc /sys /run; do sudo umount /mnt$i; done
sudo umount /mnt/boot/efi
sudo umount /mnt/boot
sudo umount /mnt/home
sudo umount /mnt
```

Reboot:

```bash
reboot
```

At boot you will be prompted once by the initramfs to enter your LUKS2 passphrase. After entering it, the system boots normally into your desktop.

### Verify the Setup

Once logged in, open a terminal and run these checks:

```bash
# Confirm LUKS2 is in use
sudo cryptsetup status cryptdata
# type:    LUKS2   ← should say LUKS2

# Confirm Argon2id key derivation
sudo cryptsetup luksDump /dev/vda4 | grep -E "Version|PBKDF"
# Version:        2
# PBKDF:          argon2id

# Check btrfs subvolumes
sudo btrfs subvolume list /
# ID 256 gen ... path @
# ID 258 gen ... path @home

# Check mount options
findmnt -t btrfs

# Confirm partition layout
lsblk -f
# vda2   ext4         /boot   ← unencrypted ext4
# vda4   crypto_LUKS          ← LUKS2 container
```

### Update the System

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Step 7: Install Timeshift and grml-rescueboot

### Install Timeshift

```bash
sudo apt install timeshift
```

Open Timeshift from the applications menu and configure it:

- **Snapshot type**: BTRFS
- **Snapshot location**: select your root btrfs partition (`/dev/mapper/cryptdata`)
- Configure a snapshot schedule to your liking (daily, weekly, etc.)
- Click **Create** to take your first manual snapshot and verify everything works

### Install grml-rescueboot

[grml-rescueboot](https://github.com/grml/grml-rescueboot) adds a GRUB menu entry that boots directly from a live ISO stored in `/boot/grml/`. This gives you a fully working live rescue environment accessible from GRUB itself — no USB stick needed. From the live environment you can unlock your LUKS2 partition, mount your btrfs subvolumes, and restore any Timeshift snapshot. This is why we sized `/boot` at 6 GiB — enough to comfortably hold a live ISO alongside your kernels.

```bash
sudo apt install grml-rescueboot
```

Download a live ISO of your choice (a Mint ISO, a Grml ISO, etc.) and place it in `/boot/grml/`:

```bash
sudo mkdir -p /boot/grml
# Copy or download your ISO into /boot/grml/
# Example:
sudo cp /path/to/your.iso /boot/grml/rescue.iso
```

Update GRUB to add the rescue boot entry:

```bash
sudo update-grub
```

Reboot and check your GRUB menu — you should now see a rescue/live entry alongside your normal boot options. Selecting it boots directly from the ISO without any USB drive.

### Restoring a Timeshift Snapshot from the Live Environment

If your system breaks and you need to restore a snapshot:

1. Reboot and as soon as the machine starts, **hold or tap `Shift`** to bring up the GRUB menu
2. Select the live ISO entry added by grml-rescueboot
3. Once the live environment has loaded, install Timeshift if it is not already present — if you are using a Linux Mint live ISO, [Timeshift](https://github.com/linuxmint/timeshift) is already included
4. Open Timeshift, and when prompted to select a snapshot location, choose your encrypted root disk
5. Enter your LUKS2 passphrase when asked — Timeshift will unlock the partition to read the snapshots on it
6. Select the snapshot you want to restore and click **Restore**
7. Once the restore completes, reboot back into your fully working system

---

## Summary of Changes vs. the Original mutschler.dev Guide

| | Original guide | This guide |
|---|---|---|
| LUKS version | LUKS1 | **LUKS2** |
| Key derivation | PBKDF2 | **Argon2id** (`--iter-time 4000`) |
| `/boot` location | Inside LUKS1 (encrypted) | **Separate unencrypted ext4 partition** |
| `/boot` size | Part of root partition | **6 GiB** (room for rescue ISO) |
| Number of partitions | 3 | **4** |
| GRUB crypto module | Required | **Not needed** |
| `GRUB_ENABLE_CRYPTODISK` | Required | **Not needed** |
| Who decrypts LUKS | GRUB (boot time) | **initramfs** (after GRUB hands off) |
| Mount options | `ssd,noatime,space_cache,commit=120,compress=zstd` | **`defaults,compress=zstd:1`** |
| Rescue / rollback method | grub-btrfs snapshot entries | **grml-rescueboot live ISO in GRUB** |
| Compatible distros | Ubuntu 20.04, Mint, Zorin | **Linux Mint 21/22, Zorin OS 16/17** |