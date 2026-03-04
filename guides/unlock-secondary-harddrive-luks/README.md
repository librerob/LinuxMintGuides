# 🔐 Auto-Unlock a Secondary LUKS-Encrypted Drive at Boot

> **Applies to:** Ubuntu, Linux Mint, and most Ubuntu-based distributions.

---

## Overview

This guide covers how to configure a secondary LUKS-encrypted drive to unlock automatically at boot using a keyfile — no extra password prompt required.

Your drive setup looks like this:

```
/dev/nvme0n1p1   →   crypto_LUKS (encrypted container)
        ↓ unlock
BackupSSD512GB   →   /dev/mapper/BackupSSD512GB (decrypted mapper device)
        ↓ mount
filesystem       →   /mnt/data
```

The filesystem inside your LUKS container can be either **ext4** or **btrfs** — both are covered in this guide.

---

## ⚠️ The Most Common Mistake

There are **two different UUIDs** involved, and mixing them up is the most frequent source of errors.

| Layer | Device | UUID type | Goes in |
|---|---|---|---|
| Physical (locked) | `/dev/nvme0n1p1` | `crypto_LUKS` UUID | `/etc/crypttab` |
| Decrypted (unlocked) | `/dev/mapper/BackupSSD512GB` | filesystem UUID | `/etc/fstab` |

### 🧠 The Golden Rule

> **`/etc/crypttab` → LUKS UUID**
> **`/etc/fstab` → filesystem (ext4 or btrfs) UUID**
>
> Never mix them.

If you try to mount `/dev/nvme0n1p1` directly as a filesystem, you'll see an error like:

```
EXT4-fs (nvme0n1p1): VFS: Can't find ext4 filesystem
```

That's because the partition is an encrypted container — the filesystem only exists *inside* the unlocked mapper device.

---

## Step 1 — Create a Keyfile

The keyfile is what allows the system to unlock your drive automatically at boot. It is stored on your (already-encrypted) root drive, so it is only readable after your main drive password is entered.

```bash
sudo mkdir -p /etc/luks-keys
sudo dd if=/dev/urandom of=/etc/luks-keys/backupssd.key bs=512 count=4
sudo chmod 400 /etc/luks-keys/backupssd.key
```

Then register the keyfile with your LUKS container:

```bash
sudo cryptsetup luksAddKey /dev/nvme0n1p1 /etc/luks-keys/backupssd.key
```

You will be prompted for your existing LUKS passphrase to authorize adding the key.

---

## Step 2 — Get Both UUIDs

### 1. LUKS UUID (the locked/physical layer)

```bash
sudo blkid /dev/nvme0n1p1
```

Look for the line containing `TYPE="crypto_LUKS"`. Note the `UUID` value — this goes into `/etc/crypttab`.

---

### 2. Filesystem UUID (the unlocked layer)

First, manually unlock the drive once:

```bash
sudo cryptsetup luksOpen /dev/nvme0n1p1 BackupSSD512GB
```

Then query the unlocked mapper device:

```bash
sudo blkid /dev/mapper/BackupSSD512GB
```

Look for `TYPE="ext4"` or `TYPE="btrfs"`. Note the `UUID` value — this goes into `/etc/fstab`.

When you're done, close it again:

```bash
sudo cryptsetup close BackupSSD512GB
```

---

## Step 3 — Configure `/etc/crypttab`

This file tells the system which encrypted partitions to unlock at boot and how.

```bash
sudo nano /etc/crypttab
```

Add the following line (replace `AAAA-BBBB-CCCC` with your actual LUKS UUID):

```
BackupSSD512GB  UUID=AAAA-BBBB-CCCC  /etc/luks-keys/backupssd.key  luks
```

| Field | Description |
|---|---|
| `BackupSSD512GB` | The name for the mapper device (your choice) |
| `UUID=AAAA-BBBB-CCCC` | The **LUKS UUID** from Step 2 |
| `/etc/luks-keys/backupssd.key` | Path to the keyfile |
| `luks` | Unlock as a standard LUKS volume |

This is the same regardless of whether your filesystem inside is ext4 or btrfs.

---

## Step 4 — Configure `/etc/fstab`

This file tells the system where to mount the unlocked filesystem.

First, create a mount point:

```bash
sudo mkdir -p /mnt/data
```

Then open fstab:

```bash
sudo nano /etc/fstab
```

Choose the line that matches your filesystem type and add it (replace `XXXX-YYYY-ZZZZ` with your actual filesystem UUID):

### ext4

```
# Backup SSD 512GB
UUID=XXXX-YYYY-ZZZZ  /mnt/data  ext4  defaults,nofail,x-gvfs-show,x-gvfs-name=BackupSSD512GB  0  2
```

### btrfs

```
# Backup SSD 512GB
UUID=XXXX-YYYY-ZZZZ  /mnt/data  btrfs  defaults,compress=zstd:1,nofail,x-gvfs-show,x-gvfs-name=BackupSSD512GB  0  2
```

> `compress=zstd:1` enables transparent zstd compression at level 1 — fast, effective, and low on CPU. This is the recommended default for btrfs on SSDs.

### Mount options explained

| Option | Purpose |
|---|---|
| `defaults` | Standard mount defaults |
| `compress=zstd:1` | btrfs only — enables zstd compression at level 1 |
| `nofail` | Boot continues normally even if the drive is missing or fails |
| `x-gvfs-show` | Makes the drive visible in Nautilus/file manager sidebar |
| `x-gvfs-name=BackupSSD512GB` | Sets a friendly display name in the file manager |

> **Important:** Use the **filesystem UUID** (ext4 or btrfs) here — not the LUKS UUID, and not `/dev/nvme0n1p1`.

---

## Step 5 — Apply Changes

Reload systemd and regenerate the initramfs so the new configuration takes effect at boot:

```bash
sudo systemctl daemon-reload
sudo update-initramfs -u
```

---

## Step 6 — Test Before Rebooting

Always test the configuration without rebooting first. This simulates exactly what will happen at boot.

```bash
# Close and unmount if currently open
sudo umount /mnt/data 2>/dev/null
sudo cryptsetup close BackupSSD512GB 2>/dev/null

# Reload and trigger the unlock service
sudo systemctl daemon-reload
sudo systemctl start systemd-cryptsetup@BackupSSD512GB.service

# Mount all fstab entries
sudo mount -a
```

Then verify the result:

```bash
lsblk -f
```

For ext4 you should see:

```
NAME                  FSTYPE      MOUNTPOINT
nvme0n1p1             crypto_LUKS
└─BackupSSD512GB      ext4        /mnt/data
```

For btrfs you should see:

```
NAME                  FSTYPE      MOUNTPOINT
nvme0n1p1             crypto_LUKS
└─BackupSSD512GB      btrfs       /mnt/data
```

If the drive mounts with no password prompt — everything is working correctly.

---

## ✅ Why This Is Secure

Your keyfile lives on your encrypted root drive. The security chain works like this:

1. You enter your **main drive password** at boot
2. The root filesystem unlocks
3. The **keyfile becomes readable**
4. The second drive unlocks automatically using the keyfile

Without unlocking your main drive first, the keyfile is completely inaccessible — so the secondary drive remains protected.

---

## ❌ Common Mistakes to Avoid

- Mounting `/dev/nvme0n1p1` directly as ext4 or btrfs
- Putting the LUKS UUID in `/etc/fstab`
- Putting the filesystem UUID in `/etc/crypttab`
- Skipping `sudo update-initramfs -u` after making changes
- Rebooting without testing first