# Ubuntu Post-Install Guide

> Applies to: Ubuntu 22.04/24.04 and derivatives (Linux Mint, Pop!_OS, Zorin OS, etc.)

This guide covers everything to do right after a fresh Ubuntu install — system tuning, security configuration, disabling unnecessary services, setting up your preferred software, and configuring the Cinnamon desktop. Each section explains **what the setting does**, **where the file goes**, and **which service to restart**.

---

## Table of Contents

1. [Kernel & System Tuning (`99-hardening.conf`)](#1-kernel--system-tuning)
2. [Tune Swappiness (`99-swappiness.conf`)](#2-tune-swappiness)
3. [Cloudflare DNS over TLS (`resolved.conf`)](#3-cloudflare-dns-over-tls)
4. [Random MAC Address at Every Boot (`NetworkManager.conf`)](#4-random-mac-address-at-every-boot)
5. [UFW Firewall](#5-ufw-firewall)
6. [UFW + QEMU/Virt-Manager (Optional)](#6-ufw--qemuvirt-manager-optional)
7. [Disable Unnecessary Services](#7-disable-unnecessary-services)
8. [Remove Unwanted Packages](#8-remove-unwanted-packages)
9. [Software to Install](#9-software-to-install)
10. [Cinnamon Desktop Tweaks](#10-cinnamon-desktop-tweaks)
11. [Applying Everything & Verifying](#11-applying-everything--verifying)
12. [Quick Reference Cheatsheet](#12-quick-reference-cheatsheet)

---

## 1. Kernel & System Tuning

### What this does

This file tunes kernel parameters at boot via `sysctl`. It restricts information leaks, enables network attack mitigations, and hardens the filesystem.

**Parameter breakdown:**

| Parameter | Value | Effect |
|---|---|---|
| `kernel.kptr_restrict` | `2` | Hides kernel pointer addresses from all users (prevents info leaks) |
| `kernel.dmesg_restrict` | `1` | Restricts `dmesg` output to root only |
| `net.ipv4.tcp_syncookies` | `1` | Protects against SYN flood DoS attacks |
| `net.ipv4.tcp_rfc1337` | `1` | Protects against TIME_WAIT assassination attacks |
| `net.ipv4.conf.all.rp_filter` | `1` | Enables reverse path filtering (blocks IP spoofing) |
| `net.ipv4.conf.default.rp_filter` | `1` | Same as above, applied to new interfaces |
| `fs.protected_symlinks` | `1` | Prevents symlink-based privilege escalation attacks |
| `fs.protected_hardlinks` | `1` | Prevents hardlink-based privilege escalation attacks |
| `vm.mmap_rnd_bits` | `32` | Maximises ASLR entropy for 64-bit processes |
| `vm.mmap_rnd_compat_bits` | `16` | Maximises ASLR entropy for 32-bit (compat) processes |

### Where to put it

All custom `sysctl` drop-in files live in:

```
/etc/sysctl.d/
```

Files in this directory are loaded in **alphabetical order** at boot. The `99-` prefix ensures your settings load last and are not overridden by earlier files (e.g. `10-network-security.conf` that ships with Ubuntu).

### Create the file

```bash
sudo nano /etc/sysctl.d/99-hardening.conf
```

Paste the following content:

```ini
# kernel pointer/info leak hardening
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1

# network protections
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_rfc1337 = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# file system protections
fs.protected_symlinks = 1
fs.protected_hardlinks = 1

# ASLR randomness
vm.mmap_rnd_bits = 32
vm.mmap_rnd_compat_bits = 16
```

Save and close (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

### Apply without rebooting

```bash
sudo sysctl --system
```

This reloads all files in `/etc/sysctl.d/` and `/etc/sysctl.conf`. You should see each parameter echoed as it is applied.

### Verify

```bash
sysctl kernel.kptr_restrict
sysctl net.ipv4.tcp_syncookies
sysctl vm.mmap_rnd_bits
```

Each should return the value you set.

---

## 2. Tune Swappiness

### What this does

`vm.swappiness` controls how aggressively the kernel moves data from RAM to swap space. The default is `60`, meaning the kernel starts swapping relatively early. Setting it to `10` tells the kernel to strongly prefer keeping data in RAM and only use swap when necessary — this results in a more responsive desktop.

> **Note:** Setting it to `0` disables swap entirely (not recommended unless you have a lot of RAM), and values closer to `100` make the system swap very aggressively. `10` is a well-regarded sweet spot for desktop use.

### Where to put it

Same directory as above — `/etc/sysctl.d/`. Using a separate file keeps concerns separated and makes it easy to change later.

### Create the file

```bash
sudo nano /etc/sysctl.d/99-swappiness.conf
```

Paste:

```ini
vm.swappiness = 10
```

Save and close.

### Apply without rebooting

```bash
sudo sysctl --system
```

Or apply only this one parameter immediately:

```bash
sudo sysctl -w vm.swappiness=10
```

### Verify

```bash
cat /proc/sys/vm/swappiness
```

Should output `10`.

---

## 3. Cloudflare DNS over TLS

### What this does

By default, DNS queries are sent in plaintext — anyone on your network (your ISP, a coffee shop router, etc.) can read or tamper with them. This configuration:

- Routes all DNS through **Cloudflare's 1.1.1.1** (IPv4 and IPv6)
- Uses **DNS over TLS (DoT)** to encrypt queries
- Uses `systemd-resolved` as the local resolver stub

**Parameter breakdown:**

| Parameter | Value | Effect |
|---|---|---|
| `DNS=` | Cloudflare IPs | The upstream resolvers to use |
| `DNSOverTLS=yes` | — | Enforces TLS encryption for all DNS queries |
| `DNSStubListener=yes` | — | Keeps the local stub at `127.0.0.53` active (required by many apps) |
| `Domains=~.` | — | Makes this the default resolver for all domains |

### Where to put it

`systemd-resolved` reads its configuration from:

```
/etc/systemd/resolved.conf
```

This file already exists on Ubuntu — you will **edit** it rather than create it from scratch.

### Edit the file

```bash
sudo nano /etc/systemd/resolved.conf
```

Replace the contents (or update the `[Resolve]` section) to match:

```ini
[Resolve]
DNS=1.1.1.1 1.0.0.1 2606:4700:4700::1111 2606:4700:4700::1001
DNSOverTLS=yes
DNSStubListener=yes
Domains=~.
```

Save and close.

### Ensure the symlink is correct

Ubuntu should already have this, but verify that `/etc/resolv.conf` points to the systemd-resolved stub:

```bash
ls -la /etc/resolv.conf
```

It should show something like:
```
/etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
```

If it does **not** point there, fix it:

```bash
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

### Restart the service

```bash
sudo systemctl restart systemd-resolved
```

### Verify

```bash
resolvectl status
```

Look for your DNS servers (`1.1.1.1`, `1.0.0.1`) and confirm `DNS over TLS` shows as `yes`.

You can also test resolution:

```bash
resolvectl query cloudflare.com
```

---

## 4. Random MAC Address at Every Boot

### What this does

Your device's MAC address is a hardware identifier broadcasted over Wi-Fi and Ethernet. By default it is static, which makes it trivial for networks, hotspots, and passive observers to track your device across locations and over time.

This configuration tells **NetworkManager** to generate a new random MAC address for both Wi-Fi and Ethernet at every connection, and also randomises the MAC used during Wi-Fi scanning (before you even connect).

**Parameter breakdown:**

| Section | Parameter | Value | Effect |
|---|---|---|---|
| `[main]` | `dns=systemd-resolved` | — | Tells NetworkManager to hand DNS off to systemd-resolved |
| `[device]` | `wifi.scan-rand-mac-address=yes` | — | Randomises MAC during Wi-Fi scanning (probe requests) |
| `[connection]` | `wifi.cloned-mac-address=random` | `random` | New random MAC for each Wi-Fi connection |
| `[connection]` | `ethernet.cloned-mac-address=random` | `random` | New random MAC for each Ethernet connection |

> **Note on `stable` vs `random`:** Using `random` generates a completely new address every time. If you prefer a MAC that is random but *consistent per network* (same MAC each time you connect to your home Wi-Fi, but different for every other network), use `stable` instead of `random`.

### Where to put it

NetworkManager's main config file is:

```
/etc/NetworkManager/NetworkManager.conf
```

This file already exists — you will **edit** it.

### Edit the file

```bash
sudo nano /etc/NetworkManager/NetworkManager.conf
```

Replace its contents with:

```ini
[main]
plugins=ifupdown,keyfile
dns=systemd-resolved

[ifupdown]
managed=false

[device]
wifi.scan-rand-mac-address=yes

[connection]
wifi.cloned-mac-address=random
ethernet.cloned-mac-address=random
```

Save and close.

### Restart the service

```bash
sudo systemctl restart NetworkManager
```

> ⚠️ This will briefly disconnect your network. On a desktop this is usually fine; on a remote SSH session, reconnect immediately after.

### Verify

Check your current MAC address before and after reconnecting:

```bash
ip link show
```

Each time you disconnect and reconnect to a network, the MAC shown for your interface (`wlan0`, `eth0`, etc.) should change.

---

## 5. UFW Firewall

### What this does

UFW (Uncomplicated Firewall) is a frontend for `iptables`/`nftables` that makes managing firewall rules straightforward. On a fresh Ubuntu install, UFW is installed but **disabled** by default. Enabling it with a sensible default policy blocks all unsolicited incoming connections while allowing all outgoing traffic.

### Install UFW

UFW is almost always pre-installed, but to be sure:

```bash
sudo apt update && sudo apt install ufw -y
```

### Set default policies

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

This denies all inbound connections unless you explicitly allow them, and permits all outbound connections.

### Allow SSH (do this before enabling if you use SSH)

> ⚠️ If you connect to this machine via SSH, **add this rule before enabling UFW** or you will lock yourself out.

```bash
sudo ufw allow ssh
```

This allows port 22. If your SSH runs on a non-standard port (e.g. 2222), use `sudo ufw allow 2222/tcp` instead.

### Allow other common services (optional)

Only add rules for services you actually use:

```bash
sudo ufw allow 80/tcp       # HTTP
sudo ufw allow 443/tcp      # HTTPS
sudo ufw allow 53           # DNS (if hosting a local resolver)
```

### Enable UFW

```bash
sudo ufw enable
```

You will be prompted to confirm. Type `y` and press Enter. UFW will now start automatically at every boot.

### Check status

```bash
sudo ufw status verbose
```

You should see your rules and `Status: active`.

---

## 6. UFW + QEMU/Virt-Manager (Optional)

> **Skip this section if you do not use QEMU, KVM, or Virt-Manager.**

### The problem

Both UFW and Libvirt (the backend for Virt-Manager) manage `iptables`/`nftables` rules. When UFW is enabled with a `DROP` forward policy, it blocks the DHCP and DNS traffic that your virtual machines need to communicate with the host and the internet. This section fixes that without lowering your overall firewall protection.

### Step 1 — Allow traffic on the virtual bridge

Libvirt creates a virtual bridge interface, usually named `virbr0`, to route VM traffic. Tell UFW to permit forwarded traffic through it in both directions:

```bash
sudo ufw route allow in on virbr0 out on any
sudo ufw route allow out on virbr0 in on any
```

### Step 2 — Open DHCP and DNS ports on the bridge

Even with routing allowed, the host firewall may still drop DHCP and DNS requests from VMs. Open the required ports on `virbr0`:

```bash
sudo ufw allow in on virbr0 to any port 53        # DNS (TCP + UDP)
sudo ufw allow in on virbr0 to any port 67 proto udp  # DHCP server
sudo ufw allow in on virbr0 to any port 68 proto udp  # DHCP client
```

**Port reference:**

| Port | Protocol | Service |
|---|---|---|
| 53 | TCP/UDP | DNS — lets VMs resolve domain names |
| 67 | UDP | DHCP server — assigns IP addresses to VMs |
| 68 | UDP | DHCP client — used by VMs to request an IP |

### Step 3 — Fix the forward policy

UFW defaults to `DROP` for all forwarded traffic, which kills VM internet access even after the above rules. Change this in the UFW configuration file:

```bash
sudo nano /etc/default/ufw
```

Find the line:

```
DEFAULT_FORWARD_POLICY="DROP"
```

Change it to:

```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

Save and close (`Ctrl+O`, `Enter`, `Ctrl+X`).

### Step 4 — Reload UFW

```bash
sudo ufw reload
```

### Verify VMs are working

After starting a VM in Virt-Manager, check inside the VM that it has received an IP address and can reach the internet:

```bash
ip addr show          # should show an IP in the 192.168.122.x range
ping -c 3 8.8.8.8    # basic internet connectivity
ping -c 3 google.com # DNS resolution
```

### Troubleshooting

| Symptom | Fix |
|---|---|
| VM has no IP address | Ensure UDP ports 67 and 68 are open on `virbr0` |
| VM can ping `8.8.8.8` but not domain names | Ensure port 53 is open on `virbr0` |
| VM has no internet at all | Check that `DEFAULT_FORWARD_POLICY` is set to `ACCEPT` |
| Rules applied but still broken | Run `sudo ufw reload` and restart the VM |

> **Note:** If you renamed the default network in Virt-Manager or are using a custom bridge, replace `virbr0` with your actual bridge name. Run `ip addr` to list all interfaces and find yours.

---

## 7. Disable Unnecessary Services

### What this does

Several services are enabled by default on Ubuntu that many desktop users never need. Disabling them reduces the attack surface, frees up memory, and cuts down on background activity.

| Service | What it does | Why disable |
|---|---|---|
| `cups.service` | Common Unix Printing System | Only needed if you print |
| `cups-browsed.service` | Auto-discovers network printers | Past source of serious vulnerabilities |
| `bluetooth.service` | Bluetooth stack | Disable if you don't use Bluetooth |
| `openvpn.service` | OpenVPN daemon | Not needed unless you run an OpenVPN server or always-on client |
| `kerneloops.service` | Collects and submits kernel crash reports | Sends data to external servers; not useful on a desktop |
| `ModemManager.service` | Manages mobile broadband modems | Not needed on desktops without a mobile data modem |
| `casper-md5check.service` | Verifies live USB integrity at boot | Only relevant on live USBs — does nothing on an installed system |

> **Note:** These commands `stop` the service immediately (for the current session) and `disable` it so it does not start at the next boot. If you ever need to re-enable one, replace `stop` with `start` and `disable` with `enable`. If a service is not installed, systemctl will simply report that the unit was not found — that is fine.

### Disable CUPS (printing)

```bash
sudo systemctl stop cups.service cups-browsed.service
sudo systemctl disable cups.service cups-browsed.service
```

### Disable Bluetooth

```bash
sudo systemctl stop bluetooth.service
sudo systemctl disable bluetooth.service
```

### Disable OpenVPN

```bash
sudo systemctl stop openvpn.service
sudo systemctl disable openvpn.service
```

### Disable kerneloops

```bash
sudo systemctl stop kerneloops.service
sudo systemctl disable kerneloops.service
```

### Disable ModemManager

```bash
sudo systemctl stop ModemManager.service
sudo systemctl disable ModemManager.service
```

### Disable casper-md5check

```bash
sudo systemctl stop casper-md5check.service
sudo systemctl disable casper-md5check.service
```

### Verify they are all disabled

```bash
systemctl is-enabled \
  cups.service \
  cups-browsed.service \
  bluetooth.service \
  openvpn.service \
  kerneloops.service \
  ModemManager.service \
  casper-md5check.service
```

Each should return `disabled` or `not-found`. If any return `enabled`, re-run the disable command for that service.

> **Re-enabling later:** If you buy a printer, Bluetooth headset, or need any of these back:
> ```bash
> sudo systemctl enable --now cups.service
> sudo systemctl enable --now bluetooth.service
> ```

---

## 8. Remove Unwanted Packages

Linux Mint and Ubuntu ship several packages that many users never need. Removing them frees up disk space, reduces background activity, and removes services that may phone home or open unnecessary network ports.

| Package | What it is | Why remove |
|---|---|---|
| `warpinator` | LAN file sharing tool (Mint) | Niche tool most users never use; opens a network service |
| `sticky` | Sticky notes applet (Mint) | Desktop note app — remove if you don't use it |
| `samba-common` `samba-common-bin` | Samba Windows file sharing components | Not needed unless you share files with Windows machines on a LAN |
| `webapp-manager` | Creates web app launchers (Mint) | Rarely used; easy to reinstall if needed |
| `rhythmbox` `rhythmbox-data` `rhythmbox-plugins` `rhythmbox-plugin-tray-icon` | GNOME music player | Bulky; replace with a lighter player like Celluloid or just use a browser |
| `hypnotix` | IPTV player (Mint) | Niche app most users don't need |

```bash
sudo apt purge \
  warpinator \
  sticky \
  samba-common samba-common-bin \
  webapp-manager \
  rhythmbox rhythmbox-data rhythmbox-plugins rhythmbox-plugin-tray-icon \
  hypnotix
```

Clean up leftover dependencies afterwards:

```bash
sudo apt autoremove --purge
```

> **Note:** `apt purge` removes both the package and its configuration files, which is what you want for a clean removal. `apt remove` would leave config files behind. If any package is not installed, apt will simply skip it.

---

## 9. Software to Install

### From the official repositories (apt)

These packages are all available directly from Ubuntu's repos. Install them in one go:

```bash
sudo apt update && sudo apt install -y \
  geary \
  shortwave \
  grsync \
  keepassxc \
  gnome-feeds \
  gimp \
  bleachbit \
  libreoffice \
  zsh \
  celluloid \
  ffmpegthumbnailer \
  ffmpeg \
  synaptic \
  transmission-gtk \
  grml-rescueboot \
  bridge-utils \
  virt-manager
```

**What each package does:**

| Package | Description |
|---|---|
| `geary` | Clean, modern GNOME email client |
| `shortwave` | Internet radio player for GNOME |
| `grsync` | GUI frontend for rsync — great for backups |
| `keepassxc` | Offline, open-source password manager |
| `gnome-feeds` | RSS/Atom feed reader for GNOME |
| `gimp` | Full-featured image editor |
| `bleachbit` | System cleaner — frees disk space, clears caches and logs |
| `libreoffice` | Full office suite (Writer, Calc, Impress, etc.) |
| `zsh` | Z shell — a powerful, highly customisable shell |
| `celluloid` | Lightweight video player (GTK frontend for mpv) |
| `ffmpegthumbnailer` | Generates video thumbnails in file managers |
| `ffmpeg` | CLI tool for video/audio conversion and processing |
| `synaptic` | Graphical package manager — useful for browsing and pinning packages |
| `transmission-gtk` | Lightweight, clean BitTorrent client |
| `grml-rescueboot` | Adds a GRML rescue system entry to your GRUB boot menu |
| `bridge-utils` | Tools for managing network bridges — required for VM networking |
| `virt-manager` | GUI for managing KVM/QEMU virtual machines (also pulls in libvirt) |

### Add your user to the KVM group

After installing virt-manager, add your user to the `kvm` group so you can manage VMs without sudo:

```bash
sudo usermod -aG kvm $USER
```

**Reboot** for the group membership to take effect. You can verify it worked after rebooting:

```bash
groups | grep kvm
```

### Personal Windows 10 VM — XML Reference

> **This is a personal reference config**, not a universal template. Hardware values (vCPU count, RAM, disk path, model choices) are specific to this machine and should be adjusted to match your own setup. To use it, open Virt-Manager → **Edit → Preferences → Enable XML editing**, then open your VM → **Details → XML** and paste/edit accordingly. Alternatively import it with `virsh define win10.xml`.

#### What each section does

**`<memory>`** — Sets the RAM allocated to the VM. `12582912 KiB` = 12 GiB. Both `memory` (maximum) and `currentMemory` (currently used) are set to the same value, meaning no memory ballooning — the VM always gets the full 12 GiB.

**`<vcpu>`** — Number of virtual CPUs exposed to the guest. Set to `16` here.

**`<os firmware="efi">`** — Tells QEMU to boot using UEFI firmware (OVMF) instead of the old SeaBIOS. Required for Secure Boot and modern Windows installs.

**`<loader>` and `<nvram>`** — Point to the OVMF firmware files. The `_4M.ms.fd` variants include Microsoft's certificate store, which is needed for Secure Boot with Windows. `nvram` is where the VM stores its own UEFI variables (boot order, Secure Boot state, etc.) between sessions.

**`<firmware><feature>`** — `enrolled-keys` loads the Microsoft certificates into the UEFI firmware, and `secure-boot` enables Secure Boot enforcement. Both are needed for Windows 11 and recommended for Windows 10.

**`<hyperv>`** — Hyper-V enlightenments are paravirtualisation hints that tell Windows it is running inside a hypervisor, allowing it to use more efficient code paths instead of fighting the virtualisation layer. Each one:

| Enlightenment | What it does |
|---|---|
| `relaxed` | Disables strict timer checking — reduces overhead |
| `vapic` | Virtual APIC — speeds up interrupt handling |
| `spinlocks` | Tells Windows to use hypercalls instead of busy-waiting on locks |
| `vpindex` | Gives each vCPU a unique index — improves SMP scheduling |
| `runtime` | Exposes runtime counter to the guest |
| `synic` | Synthetic interrupt controller — needed for some Hyper-V features |
| `stimer` | Synthetic timers — more accurate than emulated PIT/RTC |
| `frequencies` | Exposes TSC frequency — improves timekeeping accuracy |

**`<smm state="on">`** — System Management Mode, required for Secure Boot to work correctly.

**`<cpu mode="host-passthrough">`** — Passes the host CPU's exact feature flags through to the guest instead of presenting a generic virtualised CPU. This gives the best performance and compatibility but means the VM can only be migrated to an identical CPU. `<topology sockets="1" dies="1" cores="8" threads="2"/>` maps the 16 vCPUs as 1 socket × 8 cores × 2 threads (matching a typical modern CPU's layout).

**`<clock offset="localtime">`** — Sets the VM's hardware clock to localtime rather than UTC. Windows expects the hardware clock to be in local time (unlike Linux which expects UTC), so this prevents Windows from showing the wrong time. The timer settings:

| Timer | Setting | Effect |
|---|---|---|
| `rtc tickpolicy="catchup"` | Catches up missed RTC ticks — prevents clock drift |
| `pit tickpolicy="delay"` | Delays missed PIT ticks — reduces CPU overhead |
| `hpet present="no"` | Disables HPET — not needed and can cause issues in VMs |
| `hypervclock present="yes"` | Uses Hyper-V synthetic clock — most accurate option for Windows guests |

**`<on_poweroff>`, `<on_reboot>`, `<on_crash>`** — Define what QEMU does when Windows shuts down, reboots, or crashes. `destroy` means the VM process is killed immediately on poweroff/crash; `restart` means it restarts on reboot.

**`<suspend-to-mem/disk enabled="no">`** — Disables VM suspend/hibernate. Keeping these off avoids issues with QEMU state files and is simpler for a desktop VM.

**`<disk type="file" device="disk">`** — The main Windows disk. `type="raw"` means a plain image file with no overhead (as opposed to qcow2 which adds copy-on-write features but with some performance cost). The `source file` path is where your `.img` file lives.

**`<disk type="file" device="cdrom">`** — The VirtIO drivers ISO, attached as a virtual CD-ROM. Windows needs these drivers for better paravirtualised disk and network performance — you download them from the Fedora VirtIO project.

**`<controller type="usb" model="qemu-xhci">`** — USB 3.0 controller (xHCI). Gives the VM USB 3.0 support for passing through USB devices.

**`<interface type="network" model="e1000e">`** — Virtual network card using the Intel e1000e model. This is a well-supported emulated NIC that works out of the box in Windows without needing extra drivers. For better performance you could switch to `virtio` after installing VirtIO drivers.

**`<graphics type="spice">`** — SPICE protocol for the display. SPICE is the default display protocol in Virt-Manager and works well for desktop VMs. `compression="off"` disables image compression for a sharper display. Clipboard and file transfer are disabled here for security — enable them if you need them.

**`<video model="virtio">`** — VirtIO GPU, which performs better than the default VGA or QXL models in Virt-Manager.

**`<watchdog model="itco" action="reset">`** — A virtual hardware watchdog. If the VM hangs completely, the watchdog fires and resets the VM automatically.

**`<memballoon model="virtio">`** — Memory balloon driver. Allows the hypervisor to reclaim unused RAM from the guest dynamically. With VirtIO drivers installed in Windows this works automatically in the background.

```xml
<domain type="kvm">
  <name>win10</name>
  <metadata>
    <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
      <libosinfo:os id="http://microsoft.com/win/10"/>
    </libosinfo:libosinfo>
  </metadata>
  <memory unit="KiB">12582912</memory>
  <currentMemory unit="KiB">12582912</currentMemory>
  <vcpu placement="static">16</vcpu>
  <os firmware="efi">
    <type arch="x86_64" machine="pc-q35-noble">hvm</type>
    <firmware>
      <feature enabled="yes" name="enrolled-keys"/>
      <feature enabled="yes" name="secure-boot"/>
    </firmware>
    <loader readonly="yes" secure="yes" type="pflash">/usr/share/OVMF/OVMF_CODE_4M.ms.fd</loader>
    <nvram template="/usr/share/OVMF/OVMF_VARS_4M.ms.fd">/var/lib/libvirt/qemu/nvram/win10_VARS.fd</nvram>
    <boot dev="hd"/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <hyperv mode="custom">
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vpindex state="on"/>
      <runtime state="on"/>
      <synic state="on"/>
      <stimer state="on"/>
      <frequencies state="on"/>
    </hyperv>
    <vmport state="off"/>
    <smm state="on"/>
  </features>
  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" cores="8" threads="2"/>
  </cpu>
  <clock offset="localtime">
    <timer name="rtc" tickpolicy="catchup"/>
    <timer name="pit" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
    <timer name="hypervclock" present="yes"/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled="no"/>
    <suspend-to-disk enabled="no"/>
  </pm>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type="file" device="disk">
      <driver name="qemu" type="raw"/>
      <source file="/path/to/your/win10.img"/>
      <target dev="sda" bus="sata"/>
      <address type="drive" controller="0" bus="0" target="0" unit="0"/>
    </disk>
    <disk type="file" device="cdrom">
      <driver name="qemu" type="raw"/>
      <source file="/path/to/virtio-win.iso"/>
      <target dev="sdb" bus="sata"/>
      <readonly/>
      <address type="drive" controller="0" bus="0" target="0" unit="1"/>
    </disk>
    <controller type="usb" index="0" model="qemu-xhci" ports="15">
      <address type="pci" domain="0x0000" bus="0x02" slot="0x00" function="0x0"/>
    </controller>
    <controller type="pci" index="0" model="pcie-root"/>
    <controller type="pci" index="1" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="1" port="0x10"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x0" multifunction="on"/>
    </controller>
    <controller type="sata" index="0">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1f" function="0x2"/>
    </controller>
    <controller type="virtio-serial" index="0">
      <address type="pci" domain="0x0000" bus="0x03" slot="0x00" function="0x0"/>
    </controller>
    <interface type="network">
      <source network="default"/>
      <model type="e1000e"/>
      <address type="pci" domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
    </interface>
    <serial type="pty">
      <target type="isa-serial" port="0">
        <model name="isa-serial"/>
      </target>
    </serial>
    <console type="pty">
      <target type="serial" port="0"/>
    </console>
    <input type="tablet" bus="usb">
      <address type="usb" bus="0" port="1"/>
    </input>
    <input type="mouse" bus="ps2"/>
    <input type="keyboard" bus="ps2"/>
    <graphics type="spice" autoport="yes">
      <listen type="address"/>
      <image compression="off"/>
      <clipboard copypaste="no"/>
      <filetransfer enable="no"/>
    </graphics>
    <sound model="ich9">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1b" function="0x0"/>
    </sound>
    <audio id="1" type="spice"/>
    <video>
      <model type="virtio" heads="1" primary="yes"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0"/>
    </video>
    <watchdog model="itco" action="reset"/>
    <memballoon model="virtio">
      <address type="pci" domain="0x0000" bus="0x04" slot="0x00" function="0x0"/>
    </memballoon>
  </devices>
</domain>
```

### Brave Browser

Brave is not in the official Ubuntu repos and must be installed via its own apt repository.

```bash
sudo apt install curl -y

sudo curl -fsSLo /usr/share/keyrings/brave-browser-archive-keyring.gpg \
  https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/brave-browser-archive-keyring.gpg arch=amd64] \
  https://brave-browser-apt-release.s3.brave.com/ stable main" \
  | sudo tee /etc/apt/sources.list.d/brave-browser.list

sudo apt update && sudo apt install brave-browser -y
```

This adds Brave's official signed repository to your system, so future updates arrive automatically via `apt upgrade`.

### Make Zsh your default shell

After installing `zsh`, switch your user account's default shell from bash:

```bash
chsh -s $(which zsh)
```

You will be prompted for your password. **Log out and log back in** for the change to take effect. When you next open a terminal, Zsh will greet you with a first-time setup wizard — follow the prompts to generate a default `~/.zshrc`.

To confirm the change worked:

```bash
echo $SHELL
```

Should output `/usr/bin/zsh` or `/bin/zsh`.

> **Want more from Zsh?** Consider installing [Oh My Zsh](https://ohmyz.sh/) afterwards for themes and plugins:
> ```bash
> sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
> ```

---

## 10. Cinnamon Desktop Tweaks

> **Skip this section if you are not using the Cinnamon desktop** (e.g. Linux Mint).

These are recommended tweaks for a cleaner, more usable Cinnamon experience.

### Grouped Window List — Taskbar Behaviour

Right-click the taskbar → **Applets** → find **Grouped Window List** → click the settings icon.

- Set **Left click action** to **Cycle windows** — clicking a taskbar group cycles through its windows instead of toggling the whole group
- Disable **Enable Super + number shortcut** — frees up the Super key combinations for other uses

### Window Placement

**System Settings → Windows → Behaviour**

- Set **Location of newly opened windows** to **Center** — new windows open in the middle of the screen instead of a random position

### Effects

**System Settings → Effects**

- Disable all effects — removes animations and transitions for a faster, snappier desktop feel

### Notifications

**System Settings → Notifications**

- Enable **Show notifications at the bottom of the screen** — moves notification popups to the bottom where they are less intrusive

### Language & Region

**System Settings → Languages**

- Install your preferred language if not already present
- Set your interface language (English recommended for consistency with guides and error messages)
- Set your region and time format separately to match your locale

### Startup Applications

**System Settings → Startup Applications**

- Review the list and disable anything you do not need starting automatically at login — common candidates are cloud sync clients, update managers, and chat apps you prefer to launch manually

### Power Management

**System Settings → Power Management**

- Set **Turn off the screen when inactive** to **Never** for both **Battery** and **AC power** — prevents the screen from blanking unexpectedly during long tasks or reading

### Keyboard Shortcuts

**System Settings → Keyboard → Shortcuts**

- Review and configure your preferred keyboard shortcuts — set up workspace switching, window snapping, screenshot, terminal launch, or whatever suits your workflow

### Clock — Custom Date Format

Right-click the clock in the taskbar → **Configure**

- Disable **Show calendar events** — keeps the calendar popup clean
- Enable **Show week numbers in calendar**
- Enable **Use custom date format** and set your preferred format, for example `%A %e %B  %H:%M` for `Monday 3 March  14:30`

---

## 11. Applying Everything & Verifying

Here is a single block you can run after creating all the files above:

```bash
# Apply sysctl settings (tuning + swappiness)
sudo sysctl --system

# Restart DNS resolver
sudo systemctl restart systemd-resolved

# Restart NetworkManager (random MAC)
sudo systemctl restart NetworkManager

# Check UFW status
sudo ufw status verbose

# Verify DNS
resolvectl status

# Verify swappiness
cat /proc/sys/vm/swappiness

# Verify key kernel params
sysctl kernel.kptr_restrict kernel.dmesg_restrict net.ipv4.tcp_syncookies vm.mmap_rnd_bits

# Verify disabled services
systemctl is-enabled cups.service cups-browsed.service bluetooth.service
```

All settings (except the active MAC address) will also **persist across reboots** automatically — no additional steps needed.

---

## 12. Quick Reference Cheatsheet

| File to create/edit | Location | Command to apply |
|---|---|---|
| `99-hardening.conf` | `/etc/sysctl.d/99-hardening.conf` | `sudo sysctl --system` |
| `99-swappiness.conf` | `/etc/sysctl.d/99-swappiness.conf` | `sudo sysctl --system` |
| `resolved.conf` | `/etc/systemd/resolved.conf` | `sudo systemctl restart systemd-resolved` |
| `NetworkManager.conf` | `/etc/NetworkManager/NetworkManager.conf` | `sudo systemctl restart NetworkManager` |
| UFW setup | — (CLI only) | `sudo ufw enable` |
| UFW forward policy | `/etc/default/ufw` | `sudo ufw reload` |
| Disable CUPS | — (CLI only) | `sudo systemctl disable cups.service cups-browsed.service` |
| Disable Bluetooth | — (CLI only) | `sudo systemctl disable bluetooth.service` |
| Disable OpenVPN | — (CLI only) | `sudo systemctl disable openvpn.service` |
| Disable kerneloops | — (CLI only) | `sudo systemctl disable kerneloops.service` |
| Disable ModemManager | — (CLI only) | `sudo systemctl disable ModemManager.service` |
| Disable casper-md5check | — (CLI only) | `sudo systemctl disable casper-md5check.service` |
| Remove unwanted packages | — (CLI only) | `sudo apt purge warpinator sticky samba-common ...` |
| Install software | — (CLI only) | `sudo apt install ...` |
| Set default shell to Zsh | — (CLI only) | `chsh -s $(which zsh)` |
| Cinnamon tweaks | — (GUI only) | System Settings / right-click taskbar |

---

*Guide tested on Ubuntu 22.04 LTS, Ubuntu 24.04 LTS, Linux Mint 21.x, and Pop!_OS 22.04.*
