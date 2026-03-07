# Linux Mint Cinnamon Post-Install Guide

> Written for Linux Mint Cinnamon, but most sections apply equally to Ubuntu 22.04/24.04 and other derivatives (Pop!_OS, Zorin OS, etc.).

This guide covers everything to do right after a fresh Linux Mint install — system tuning, security configuration, disabling unnecessary services, setting up your preferred software, and configuring the Cinnamon desktop. Each section explains **what the setting does**, **where the file goes**, and **which service to restart**.

---

## Table of Contents

1. [Kernel & System Tuning (`99-hardening.conf`)](#1-kernel--system-tuning)
2. [GRUB Boot Parameters](#1a-grub-boot-parameters)
3. [Tune Swappiness (`99-swappiness.conf`)](#2-tune-swappiness)
4. [Cloudflare DNS over TLS (`resolved.conf`)](#3-cloudflare-dns-over-tls)
5. [Hosts File Ad & Malware Blocking](#4-hosts-file-ad--malware-blocking)
6. [Random MAC Address at Every Boot (`NetworkManager.conf`)](#5-random-mac-address-at-every-boot)
7. [UFW Firewall](#6-ufw-firewall)
8. [UFW + QEMU/Virt-Manager (Optional)](#7-ufw--qemuvirt-manager-optional)
9. [Disable Unnecessary Services](#8-disable-unnecessary-services)
10. [Remove Unwanted Packages](#9-remove-unwanted-packages)
11. [Software to Install](#10-software-to-install)
12. [Brave Browser](#11-brave-browser)
13. [Zsh Configuration](#12-zsh-configuration)
14. [Cinnamon Desktop Tweaks](#13-cinnamon-desktop-tweaks)
15. [Unified Qt & GTK Theming](#14-unified-qt--gtk-theming)
16. [Font Configuration](#15-font-configuration)
17. [Applying Everything & Verifying](#16-applying-everything--verifying)
18. [Quick Reference Cheatsheet](#17-quick-reference-cheatsheet)

---

## 1. Kernel & System Tuning

### What this does

This file tunes kernel parameters at boot via `sysctl`. It restricts information leaks, hardens the kernel against common exploit techniques, enables network attack mitigations, and hardens the filesystem.

Linux Mint already ships several sysctl files in `/etc/sysctl.d/` with some baseline settings. Our `99-` prefixed file loads last (files are applied in alphabetical order) and overrides any weaker defaults. Here is what Mint sets out of the box and what we change:

| Parameter | Mint default | Our value | What changes |
|---|---|---|---|
| `kernel.kptr_restrict` | `1` | `2` | Hides kernel pointers from *all* users, not just unprivileged ones |
| `kernel.printk` | `4 4 1 7` | `3 3 3 3` | Stricter suppression of kernel messages on the boot console |
| `kernel.sysrq` | `176` | `4` | Locks SysRq down to secure attention key only — Mint's default allows many functions |
| `kernel.yama.ptrace_scope` | `1` | `2` | Restricts ptrace to `CAP_SYS_PTRACE` — Mint only restricts to parent processes |
| `net.ipv4.conf.all.rp_filter` | `2` | `1` | Loosened slightly for VM/VPN compatibility — see note below |

Everything else in this file — BPF restrictions, kexec, userfaultfd, ICMP redirects, source routing, TCP SACK, protected fifos/regular, ASLR bits — is not set by Mint at all and is genuinely new.

> **Note on `rp_filter`:** Mint ships `rp_filter=2` (strict mode), which is technically stronger, but it can break asymmetric routing used by KVM virtual machines and VPNs. We set `1` (loose mode) which still blocks IP spoofing and is the right choice if you use Virt-Manager or a VPN. If you don't use either, you can safely set it to `2`.

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
# ============================================================
# Kernel self-protection
# ============================================================

# Hide kernel pointer addresses from all users (prevents info leaks)
kernel.kptr_restrict = 2

# Restrict dmesg output to root only
kernel.dmesg_restrict = 1

# Suppress kernel log messages on console during boot
kernel.printk = 3 3 3 3

# Restrict eBPF to CAP_BPF — large kernel attack surface
kernel.unprivileged_bpf_disabled = 1

# Harden the eBPF JIT compiler (constant blinding etc.)
net.core.bpf_jit_harden = 2

# Disable loading another kernel at runtime (kexec)
kernel.kexec_load_disabled = 1

# Restrict SysRq to secure attention key only
kernel.sysrq = 4

# Restrict userfaultfd() to CAP_SYS_PTRACE — commonly abused in kernel exploits
vm.unprivileged_userfaultfd = 0

# Restrict TTY line discipline loading to CAP_SYS_MODULE
dev.tty.ldisc_autoload = 0

# ============================================================
# Network protections
# ============================================================

# Protect against SYN flood DoS attacks
net.ipv4.tcp_syncookies = 1

# Protect against TIME_WAIT assassination attacks
net.ipv4.tcp_rfc1337 = 1

# Enable reverse path filtering — blocks IP spoofing
# Using loose mode (1) instead of Mint's strict mode (2) for VM/VPN compatibility
# Change to 2 if you don't use KVM/Virt-Manager or a VPN
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Disable ICMP redirect acceptance — prevents man-in-the-middle attacks
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Disable sending ICMP redirects
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Disable source routing — prevents traffic redirection attacks
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# Disable IPv6 router advertisement acceptance — prevents MitM via rogue RA
net.ipv6.conf.all.accept_ra = 0
net.ipv6.conf.default.accept_ra = 0

# Disable TCP SACK — commonly exploited and unnecessary on most desktops
net.ipv4.tcp_sack = 0
net.ipv4.tcp_dsack = 0
net.ipv4.tcp_fack = 0

# ============================================================
# Userspace protections
# ============================================================

# Restrict ptrace to CAP_SYS_PTRACE — prevents memory inspection of
# other running processes.
# Note: breaks gdb and other debuggers for non-root users.
# Set to 3 to disable ptrace entirely.
kernel.yama.ptrace_scope = 2

# Maximise ASLR entropy for 64-bit processes
vm.mmap_rnd_bits = 32

# Maximise ASLR entropy for 32-bit (compat) processes
vm.mmap_rnd_compat_bits = 16

# ============================================================
# Filesystem protections
# ============================================================

# Prevent symlink-based privilege escalation attacks
fs.protected_symlinks = 1

# Prevent hardlink-based privilege escalation attacks
fs.protected_hardlinks = 1

# Prevent creating files in attacker-controlled world-writable directories
fs.protected_fifos = 2
fs.protected_regular = 2
```

Save and close (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

### Parameter breakdown

**Kernel self-protection**

| Parameter | Value | Effect |
|---|---|---|
| `kernel.kptr_restrict` | `2` | Hides kernel pointer addresses from all users — prevents info leaks useful for exploits |
| `kernel.dmesg_restrict` | `1` | Restricts `dmesg` (kernel log) to root only — limits info available to attackers |
| `kernel.printk` | `3 3 3 3` | Suppresses kernel messages on the console during boot — prevents screen-visible info leaks |
| `kernel.unprivileged_bpf_disabled` | `1` | Restricts eBPF to privileged processes — eBPF is a large kernel attack surface |
| `net.core.bpf_jit_harden` | `2` | Hardens the eBPF JIT compiler against attacks (constant blinding). Read-only after boot — verified with sudo |
| `kernel.kexec_load_disabled` | `1` | Prevents loading a new kernel at runtime — blocks a method of gaining arbitrary kernel code execution |
| `kernel.sysrq` | `4` | Restricts SysRq to the secure attention key only — prevents remote/physical SysRq abuse |
| `vm.unprivileged_userfaultfd` | `0` | Restricts `userfaultfd()` syscall — frequently abused in use-after-free kernel exploits |
| `dev.tty.ldisc_autoload` | `0` | Prevents unprivileged loading of TTY line disciplines — abused in past privilege escalation exploits |

**Network**

| Parameter | Value | Effect |
|---|---|---|
| `net.ipv4.tcp_syncookies` | `1` | Protects against SYN flood DoS attacks |
| `net.ipv4.tcp_rfc1337` | `1` | Protects against TIME_WAIT assassination attacks |
| `net.ipv4.conf.all.rp_filter` | `1` | Reverse path filtering — blocks IP spoofing. Set to loose mode (`1`) instead of Mint's strict (`2`) for VM/VPN compatibility |
| `net.ipv4.conf.default.rp_filter` | `1` | Same as above, applied to new interfaces |
| `accept_redirects` (IPv4 + IPv6) | `0` | Disables ICMP redirect acceptance — prevents man-in-the-middle attacks |
| `send_redirects` | `0` | Stops the machine from sending ICMP redirects (it is not a router) |
| `accept_source_route` (IPv4 + IPv6) | `0` | Disables source routing — prevents traffic redirection attacks |
| `net.ipv6.conf.all.accept_ra` | `0` | Ignores IPv6 router advertisements — malicious RAs can cause MitM attacks |
| `net.ipv4.tcp_sack` / `tcp_dsack` / `tcp_fack` | `0` | Disables TCP selective acknowledgement — commonly exploited, rarely needed on desktops |

**Userspace**

| Parameter | Value | Effect |
|---|---|---|
| `kernel.yama.ptrace_scope` | `2` | Restricts ptrace to `CAP_SYS_PTRACE` — prevents processes from inspecting each other's memory. Note: breaks `gdb` for non-root users |
| `vm.mmap_rnd_bits` | `32` | Maximises ASLR entropy for 64-bit processes. Read-only after boot — verified with sudo |
| `vm.mmap_rnd_compat_bits` | `16` | Maximises ASLR entropy for 32-bit (compat) processes. Read-only after boot — verified with sudo |

**Filesystem**

| Parameter | Value | Effect |
|---|---|---|
| `fs.protected_symlinks` | `1` | Prevents symlink-based privilege escalation |
| `fs.protected_hardlinks` | `1` | Prevents hardlink-based privilege escalation |
| `fs.protected_fifos` | `2` | Prevents creating FIFOs in world-writable directories — hardens against data spoofing. Some kernels silently cap this at `1`; setting `2` is harmless and future-proof |
| `fs.protected_regular` | `2` | Same protection for regular files in world-writable directories |

### Apply without rebooting

```bash
sudo sysctl --system
```

This reloads all files in `/etc/sysctl.d/` and `/etc/sysctl.conf`. You will see each parameter echoed as it is applied. Some values (`net.core.bpf_jit_harden`, `vm.mmap_rnd_bits`, `vm.mmap_rnd_compat_bits`) are write-only and will show "permission denied" when read as a normal user — this is expected behaviour, not an error.

### Verify

```bash
sudo sysctl \
  kernel.kptr_restrict \
  kernel.dmesg_restrict \
  kernel.unprivileged_bpf_disabled \
  net.core.bpf_jit_harden \
  kernel.kexec_load_disabled \
  kernel.yama.ptrace_scope \
  vm.mmap_rnd_bits \
  vm.mmap_rnd_compat_bits \
  fs.protected_fifos \
  net.ipv4.tcp_sack \
  net.ipv6.conf.all.accept_ra
```

All should return the values set above.

---

## 1a. GRUB Boot Parameters

### What this does

Some kernel hardening options can only be set at boot time via kernel command-line parameters passed through GRUB — they cannot be changed via sysctl once the system is running. This section adds a small set of safe, well-tested parameters that complement the sysctl hardening above.

### Edit the GRUB config

```bash
sudo nano /etc/default/grub
```

Find the `GRUB_CMDLINE_LINUX_DEFAULT` line. It will look something like:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```

Append the new parameters after your existing ones:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash slab_nomerge init_on_alloc=1 init_on_free=1 page_alloc.shuffle=1 pti=on vsyscall=none"
```

Save and close, then regenerate the GRUB config:

```bash
sudo update-grub
```

Reboot to apply:

```bash
sudo reboot
```

### Parameter breakdown

| Parameter | Effect |
|---|---|
| `slab_nomerge` | Disables merging of kernel slab caches — makes heap exploitation significantly harder by preventing cache layout manipulation |
| `init_on_alloc=1 init_on_free=1` | Zeroes memory on allocation and free — mitigates use-after-free vulnerabilities and clears sensitive data from memory. Small performance cost (~1–3%) on heavy I/O workloads |
| `page_alloc.shuffle=1` | Randomises page allocator freelists — hardens against heap attacks and slightly *improves* performance due to better cache behaviour |
| `pti=on` | Enforces Kernel Page Table Isolation — mitigates Meltdown and prevents some KASLR bypasses. Already on by default on most kernels; this makes it explicit |
| `vsyscall=none` | Disables vsyscalls (obsolete, replaced by vDSO) — vsyscalls sit at fixed memory addresses making them useful for ROP attacks. Safe on all modern software |

### Verify

After rebooting, confirm all parameters are active:

```bash
cat /proc/cmdline
```

You should see all five parameters in the output alongside your existing ones.

### Reverting if something goes wrong

If the system misbehaves after a reboot, you can edit the boot entry temporarily at the GRUB menu without touching any files:

1. At the GRUB menu, press `e` to edit the current boot entry
2. Find the `linux` line and remove the added parameters
3. Press `Ctrl+X` to boot with the edited entry (one time only — nothing is saved)
4. Once booted, edit `/etc/default/grub` to remove the parameters permanently and run `sudo update-grub`

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

## 4. Hosts File Ad & Malware Blocking

### What this does

The `/etc/hosts` file maps hostnames to IP addresses before any DNS query is made. By redirecting known ad and malware domains to `0.0.0.0`, every app and process on the system — browsers, terminals, background services — is blocked from reaching those domains at the OS level, with no browser extension or proxy required.

This section uses [StevenBlack's hosts](https://github.com/StevenBlack/hosts) — a well-maintained, widely used merged blocklist covering adware and malware domains (~76k entries). A systemd timer keeps it updated weekly automatically.

> **Note:** Hosts file blocking works at the system level and is completely independent of your browser. It complements browser-level blocking (uBlock Origin etc.) rather than replacing it.

### Step 1 — Back up the original hosts file

```bash
sudo cp /etc/hosts /etc/hosts.bak
```

To restore the original at any time:

```bash
sudo cp /etc/hosts.bak /etc/hosts
```

### Step 2 — Download and install the blocklist

```bash
sudo curl -fsSL https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts -o /etc/hosts
```

This replaces `/etc/hosts` entirely. The downloaded file includes the standard localhost entries (`127.0.0.1 localhost`, `::1 localhost` etc.) at the top, so nothing is lost.

### Step 3 — Verify it worked

```bash
# Check the file starts with the expected localhost entries
head -10 /etc/hosts

# Check the total number of blocked entries
grep -c "^0.0.0.0" /etc/hosts
```

You should see ~76,000+ blocked entries. Test that a known ad domain is blocked:

```bash
ping -c 1 doubleclick.net
```

It should resolve to `0.0.0.0` and fail immediately rather than reaching a real server.

### Step 4 — Create the update service

Create a systemd service that downloads the latest blocklist:

```bash
sudo nano /etc/systemd/system/hosts-update.service
```

```ini
[Unit]
Description=Update StevenBlack hosts blocklist
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/curl -fsSL https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts -o /etc/hosts
```

### Step 5 — Create the timer

```bash
sudo nano /etc/systemd/system/hosts-update.timer
```

```ini
[Unit]
Description=Weekly update of StevenBlack hosts blocklist

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target
```

`Persistent=true` means if the system was off at the scheduled time (e.g. the machine was shut down on the weekend), the update will run on the next boot instead of being skipped entirely.

### Step 6 — Enable and start the timer

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now hosts-update.timer
```

### Verify the timer is active

```bash
systemctl status hosts-update.timer
```

You should see `Active: active (waiting)` and the next trigger time listed. To see all active timers:

```bash
systemctl list-timers hosts-update.timer
```

### Run an update manually at any time

```bash
sudo systemctl start hosts-update.service
```

### Troubleshooting

| Symptom | Fix |
|---|---|
| Legitimate site is blocked | Add `0.0.0.0 example.com` temporarily or find and report the false positive to the StevenBlack repo |
| `ping doubleclick.net` still resolves | Run `sudo systemctl restart systemd-resolved` to flush the DNS cache |
| Timer not firing | Check `journalctl -u hosts-update.service` for errors |

---

## 5. Random MAC Address at Every Boot

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
| `gnome-terminal` `gnome-terminal-data` | Default GNOME terminal emulator | Replaced by Kitty (see Section 9) |

```bash
sudo apt purge \
  warpinator \
  sticky \
  samba-common samba-common-bin \
  webapp-manager \
  rhythmbox rhythmbox-data rhythmbox-plugins rhythmbox-plugin-tray-icon \
  hypnotix \
  gnome-terminal gnome-terminal-data
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
  goodvibes \
  grsync \
  keepassxc \
  liferea \
  gimp \
  bleachbit \
  zsh \
  kitty \
  ffmpegthumbnailer \
  ffmpeg \
  synaptic \
  grml-rescueboot \
  bridge-utils \
  virt-manager \
  torbrowser-launcher
```

**What each package does:**

| Package | Description |
|---|---|
| `geary` | Clean, modern GNOME email client |
| `goodvibes` | Lightweight internet radio player for GTK desktops |
| `grsync` | GUI frontend for rsync — great for backups |
| `keepassxc` | Offline, open-source password manager |
| `liferea` | RSS/Atom feed reader — desktop client with offline reading |
| `gimp` | Full-featured image editor |
| `bleachbit` | System cleaner — frees disk space, clears caches and logs |
| `zsh` | Z shell — a powerful, highly customisable shell |
| `kitty` | Fast, GPU-accelerated terminal emulator with tabs and splits |
| `ffmpegthumbnailer` | Generates video thumbnails in file managers |
| `ffmpeg` | CLI tool for video/audio conversion and processing |
| `synaptic` | Graphical package manager — useful for browsing and pinning packages |
| `grml-rescueboot` | Adds a GRML rescue system entry to your GRUB boot menu |
| `bridge-utils` | Tools for managing network bridges — required for VM networking |
| `virt-manager` | GUI for managing KVM/QEMU virtual machines (also pulls in libvirt) |
| `torbrowser-launcher` | Downloads, verifies, and launches the official Tor Browser |

> **Note:** `libreoffice`, `celluloid`, and `transmission-gtk` are pre-installed on Linux Mint and do not need to be reinstalled.

### Kitty terminal

Kitty is a fast, GPU-accelerated terminal emulator with tab and split-pane support. Linux Mint ships `gnome-terminal` by default — replace it with Kitty.

**Install Kitty and remove gnome-terminal:**

```bash
sudo apt install kitty
sudo apt purge gnome-terminal gnome-terminal-data
sudo apt autoremove --purge
```

**Set Kitty as the default terminal** in Cinnamon:

```
System Settings → Preferred Applications → Terminal → kitty
```

**Apply the config:**

```bash
mkdir -p ~/.config/kitty
nano ~/.config/kitty/kitty.conf
```

Paste the following:

```
# Place this file at: ~/.config/kitty/kitty.conf
# Reload config without restart: ctrl+shift+f5

# =============================================================================
# FONTS
# =============================================================================

font_family      JetBrains Mono Regular
bold_font        JetBrains Mono Bold
italic_font      JetBrains Mono Italic
bold_italic_font JetBrains Mono Bold Italic
font_size        12.0

# =============================================================================
# CURSOR
# =============================================================================

cursor_shape               beam
cursor_shape_unfocused     hollow
cursor_blink_interval      0.5
cursor_stop_blinking_after 15.0

# =============================================================================
# SCROLLBACK
# =============================================================================

scrollback_lines        10000
scrollback_pager_history_size 0

# =============================================================================
# MOUSE
# =============================================================================

mouse_hide_wait    3.0
copy_on_select     no
strip_trailing_spaces smart

# =============================================================================
# PERFORMANCE
# =============================================================================

repaint_delay   10
input_delay     3
sync_to_monitor yes

# =============================================================================
# BELL
# =============================================================================

enable_audio_bell no
visual_bell_duration 0.0
window_alert_on_bell yes

# =============================================================================
# WINDOW
# =============================================================================

remember_window_size   yes
initial_window_width   1000
initial_window_height  650

window_padding_width   8

hide_window_decorations no
draw_minimal_borders yes

# =============================================================================
# TABS
# =============================================================================

tab_bar_style          powerline
tab_powerline_style    slanted
tab_title_template     "{index}: {title}"
tab_bar_min_tabs       2

# =============================================================================
# COLORS — Tokyo Night
# =============================================================================

background            #1a1b26
foreground            #c0caf5

selection_background  #364a82
selection_foreground  #c0caf5

cursor                #c0caf5
cursor_text_color     #1a1b26

url_color             #73daca

# Black
color0  #15161e
color8  #414868

# Red
color1  #f7768e
color9  #f7768e

# Green
color2  #9ece6a
color10 #9ece6a

# Yellow
color3  #e0af68
color11 #e0af68

# Blue
color4  #7aa2f7
color12 #7aa2f7

# Magenta
color5  #bb9af7
color13 #bb9af7

# Cyan
color6  #7dcfff
color14 #7dcfff

# White
color7  #a9b1d6
color15 #c0caf5

# =============================================================================
# KEYBOARD SHORTCUTS
# =============================================================================

# -- Tabs --
map ctrl+t        new_tab_with_cwd
map ctrl+w        close_tab
map ctrl+shift+l  next_tab
map ctrl+shift+h  previous_tab

# -- Windows / Splits --
map ctrl+shift+enter  new_window_with_cwd
map ctrl+shift+w      close_window

# Navigate splits
map ctrl+shift+left   neighboring_window left
map ctrl+shift+right  neighboring_window right
map ctrl+shift+up     neighboring_window up
map ctrl+shift+down   neighboring_window down

# Layouts
map ctrl+shift+z  toggle_layout stack

# -- Font size --
map ctrl+equal   change_font_size all +1.0
map ctrl+minus   change_font_size all -1.0
map ctrl+0       change_font_size all 0

# -- Clipboard --
map ctrl+shift+c  copy_to_clipboard
map ctrl+shift+v  paste_from_clipboard

# -- Scrollback --
map ctrl+shift+k  scroll_line_up
map ctrl+shift+j  scroll_line_down
map ctrl+shift+u  scroll_page_up
map ctrl+shift+d  scroll_page_down

# -- Miscellaneous --
map ctrl+shift+f5  load_config_file
map ctrl+shift+F2  edit_config_file

# =============================================================================
# SHELL INTEGRATION
# =============================================================================

shell_integration enabled

# =============================================================================
# MISC
# =============================================================================

open_url_with default
url_style curly
close_on_child_death no
allow_remote_control  no
update_check_interval 0
```

**Config highlights:**

| Setting | Value | Effect |
|---|---|---|
| `font_family` | JetBrains Mono | Uses the monospace font installed in Section 14 |
| `cursor_shape` | `beam` | Thin beam cursor when focused, hollow when unfocused |
| `scrollback_lines` | `10000` | Keeps 10,000 lines of scrollback history |
| `enable_audio_bell` | `no` | Disables the terminal bell |
| `tab_bar_style` | `powerline` | Powerline-style slanted tab bar — only appears when 2+ tabs are open |
| `shell_integration` | `enabled` | Enables jump-to-prompt, cwd tracking, and other Kitty shell features |
| `update_check_interval` | `0` | Disables Kitty's built-in update checker |
| Colors | Tokyo Night | Dark theme matching the overall desktop palette |

**Key shortcuts:**

| Shortcut | Action |
|---|---|
| `Ctrl+T` | New tab (opens in current directory) |
| `Ctrl+W` | Close tab |
| `Ctrl+Shift+L / H` | Next / previous tab |
| `Ctrl+Shift+Enter` | New split window (opens in current directory) |
| `Ctrl+Shift+Arrow` | Navigate between splits |
| `Ctrl+Shift+Z` | Toggle zoom (stack layout — maximise current split) |
| `Ctrl+Equal / Minus` | Increase / decrease font size |
| `Ctrl+Shift+F5` | Reload config without restarting |

> **Note:** JetBrains Mono must be installed for the font config to work — it is covered in Section 14. If Kitty launches before fonts are installed it will fall back to a system monospace font automatically.

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

---

## 10. Brave Browser

### What this does

Installs Brave Browser from its official apt repository, then applies a managed policy file to disable telemetry, AI features, crypto/rewards bloat, and other non-essential extras — giving you a cleaner, more privacy-focused browser out of the box.

Policies are enforced via Brave's built-in Group Policy system (the same mechanism used by IT admins). Settings locked by policy show a building/grid icon in Brave's UI and appear greyed out — they cannot be overridden by the user through the browser UI. You can verify all active policies at any time by visiting `brave://policy/`.

### Install Brave

```bash
sudo apt install curl
sudo curl -fsSLo /usr/share/keyrings/brave-browser-archive-keyring.gpg \
  https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg
sudo curl -fsSLo /etc/apt/sources.list.d/brave-browser-release.sources \
  https://brave-browser-apt-release.s3.brave.com/brave-browser.sources
sudo apt update
sudo apt install brave-browser
```

### Apply debloat policies

Create the managed policies directory and drop in the policy file:

```bash
sudo mkdir -p /etc/brave/policies/managed/
```

Create `/etc/brave/policies/managed/policies.json` with the following content:

```json
{
  "BraveAIChatEnabled": false,
  "BraveRewardsDisabled": true,
  "BraveWalletDisabled": true,
  "BraveVPNDisabled": true,
  "TorDisabled": true,
  "BraveP3AEnabled": false,
  "BraveStatsPingEnabled": false,
  "BraveWebDiscoveryEnabled": false,
  "BraveNewsDisabled": true,
  "BraveTalkDisabled": true,
  "BraveSpeedreaderEnabled": false,
  "BraveWaybackMachineEnabled": false,
  "BravePlaylistEnabled": false,
  "SyncDisabled": false,
  "PasswordManagerEnabled": false,
  "AutofillAddressEnabled": false,
  "AutofillCreditCardEnabled": false,
  "DnsOverHttpsMode": "automatic"
}
```

**Policy breakdown:**

| Policy | Effect |
|---|---|
| `BraveAIChatEnabled: false` | Disables Leo AI assistant |
| `BraveRewardsDisabled / WalletDisabled / VPNDisabled / TorDisabled: true` | Turns off Brave's crypto and monetisation suite |
| `BraveP3AEnabled / BraveStatsPingEnabled / BraveWebDiscoveryEnabled: false` | Disables all telemetry and usage reporting |
| `BraveNewsDisabled / BraveTalkDisabled / BraveSpeedreaderEnabled / BraveWaybackMachineEnabled / BravePlaylistEnabled` | Removes sidebar clutter features |
| `SyncDisabled: false` | Leaves Sync **enabled** — disable manually if you don't want it |
| `PasswordManagerEnabled / AutofillAddressEnabled / AutofillCreditCardEnabled: false` | Disables the built-in password manager and autofill (use KeePassXC instead) |
| `DnsOverHttpsMode: automatic` | Uses secure DNS when available, falls back to your system/OS DNS resolver — no hardcoded provider |

### DNS provider options

By default the policy uses `"DnsOverHttpsMode": "automatic"` with no hardcoded provider — Brave will use secure DNS when available and fall back to your OS resolver. If you want to force a specific provider instead, set `"DnsOverHttpsMode": "secure"` and add a `"DnsOverHttpsTemplates"` line with your chosen URL.

**Available modes:**

| Mode | Behaviour |
|---|---|
| `"automatic"` | Use secure DNS when available, fall back to OS/system resolver. No `DnsOverHttpsTemplates` needed. |
| `"secure"` | Always use the provider in `DnsOverHttpsTemplates`. Fails if the provider is unreachable (no fallback). |
| `"off"` | Disable DoH entirely — use OS resolver as-is. Remove `DnsOverHttpsTemplates` too. |

**Popular DoH providers:**

| Provider | Focus | `DnsOverHttpsTemplates` value |
|---|---|---|
| Cloudflare | Speed, privacy | `https://cloudflare-dns.com/dns-query` |
| Cloudflare (malware blocking) | Speed + blocks malware | `https://security.cloudflare-dns.com/dns-query` |
| Cloudflare (family) | Blocks malware + adult content | `https://family.cloudflare-dns.com/dns-query` |
| Quad9 | Privacy + malware blocking | `https://dns.quad9.net/dns-query` |
| AdGuard | Privacy + ad/tracker blocking | `https://dns.adguard-dns.com/dns-query` |
| NextDNS | Fully customisable (account required) | `https://dns.nextdns.io/your-id` |
| Google | Speed, reliability | `https://dns.google/dns-query` |

**Example — force Cloudflare, no fallback:**

```json
{
  ...
  "DnsOverHttpsMode": "secure",
  "DnsOverHttpsTemplates": "https://cloudflare-dns.com/dns-query"
}
```

**Example — force Quad9, no fallback:**

```json
{
  ...
  "DnsOverHttpsMode": "secure",
  "DnsOverHttpsTemplates": "https://dns.quad9.net/dns-query"
}
```

**Example — disable DoH, use OS resolver:**

```json
{
  ...
  "DnsOverHttpsMode": "off"
}
```

> **Note:** When `DnsOverHttpsMode` is set to `"secure"` with a hardcoded provider, the setting will appear locked (greyed out with a policy icon) in Brave's Security settings — you won't be able to change it from the UI. Use `"automatic"` if you want to keep the setting adjustable.

### Restart Brave and verify

Restart Brave, then visit `brave://policy/` to confirm all policies are loaded and showing no errors.

> **Note:** Brave Shields and Sync are left untouched by these policies. Configure them to your preference in Brave Settings.

---

## 11. Zsh Configuration

### What this does

Zsh (Z Shell) is a more powerful and customisable shell than the default bash. This section covers making it your default shell, setting up a plugin directory structure, installing fast-syntax-highlighting, and applying a personal `.zshrc` configuration with a coloured prompt, sensible history settings, smart tab completion, useful aliases, and rsync backup commands.

### Step 1 — Install Zsh

Zsh should already be installed if you followed Section 9. If not:

```bash
sudo apt install zsh -y
```

### Step 2 — Make Zsh your default shell

```bash
chsh -s $(which zsh)
```

You will be prompted for your password. **Log out and log back in** for the change to take effect. On first launch, Zsh will offer a setup wizard — you can press `q` to skip it since the `.zshrc` below replaces it entirely.

Confirm after relogging:

```bash
echo $SHELL
```

Should output `/usr/bin/zsh` or `/bin/zsh`.

### Step 3 — Create the plugin directory structure

Plugins are kept under `~/.config/zsh/plugins/` so your home directory stays clean. Create the folder:

```bash
mkdir -p ~/.config/zsh/plugins
```

### Step 4 — Install fast-syntax-highlighting

[fast-syntax-highlighting](https://github.com/zdharma-continuum/fast-syntax-highlighting) highlights commands as you type them in the terminal — valid commands appear in one colour, invalid ones in another, strings, paths, and flags are all distinguished. It is significantly faster than the older `zsh-syntax-highlighting`.

Clone it directly into the plugin directory:

```bash
git clone https://github.com/zdharma-continuum/fast-syntax-highlighting.git \
  ~/.config/zsh/plugins/fast-syntax-highlighting
```

Your plugin directory should now look like this:

```
~/.config/zsh/
└── plugins/
    └── fast-syntax-highlighting/
```

The `.zshrc` below sources it automatically from this path.

### Step 5 — Create `~/.zshrc`

Place the following file at `~/.zshrc`:

```bash
nano ~/.zshrc
```

Paste the full configuration:

```zsh
##### ZSH CONFIG #####
autoload -U colors && colors
PS1="%B%{$fg[red]%}[%{$fg[yellow]%}%n%{$fg[green]%}@%{$fg[blue]%}%M %{$fg[magenta]%}%~%{$fg[red]%}]%{$reset_color%}$%b "
setopt autocd
setopt interactive_comments
stty stop undef

##### History #####
HISTSIZE=10000000
SAVEHIST=10000000
HISTFILE="$HOME/.zsh_history"
setopt inc_append_history
setopt share_history
setopt hist_ignore_dups
setopt hist_reduce_blanks

##### Completion #####
autoload -U compinit
zmodload zsh/complist
zstyle ':completion:*' menu select
zstyle ':completion:*' matcher-list \
    'm:{a-zA-Z}={A-Za-z}' \
    'r:|[._-]=* r:|=*' \
    'l:|=* r:|=*'
compinit
_comp_options+=(globdots)

##### Functions #####

# backup-usb: Backs up home directory to a USB drive mounted under /media/robin/
# Usage: just run 'backup-usb' and follow the prompt
#
# At the prompt you can:
#   - Press ENTER to use the default: /media/robin/Backup-Linux/Backup/
#   - Type a custom path relative to /media/robin/, for example:
#       MyOtherUSB/Backup   -> backs up to /media/robin/MyOtherUSB/Backup/
#       MyOtherUSB          -> backs up to /media/robin/MyOtherUSB/
#
# The function will list available drives before prompting so you can
# see what is currently mounted under /media/robin/
#
# Excludes: .cache, Music, Downloads, .var, Trash, BraveSoftware,
#           Chromium, torbrowser, flatpak, snap, Desktop
backup-usb() {
  echo "Available drives in /media/robin/:"
  ls /media/robin/
  echo ""
  read "drive?Enter USB drive name [Backup-Linux/Backup]: "
  drive=${drive:-Backup-Linux/Backup}
  rsync -rltvP --delete --exclude=".cache" --exclude="Music" --exclude="Downloads" --exclude=".var" --exclude=".local/share/Trash" --exclude=".config/BraveSoftware" --exclude=".config/chromium" --exclude=".config/torbrowser" --exclude=".local/share/flatpak" --exclude="snap" --exclude="Desktop" ~/ "/media/robin/$drive/"
}

##### Aliases #####
alias \
	cp="cp -iv" \
	mv="mv -iv" \
	rm="rm -vI" \
	bc="bc -ql" \
	mkd="mkdir -pv" \
	yt="yt-dlp --embed-metadata -i" \
	yta="yt -x -f bestaudio/best" \
	ytt="yt --skip-download --write-thumbnail" \
	ffmpeg="ffmpeg -hide_banner"

alias \
	ls="ls -hN --color=auto --group-directories-first" \
	grep="grep --color=auto" \
	diff="diff --color=auto" \
	ccat="highlight --out-format=ansi" \
	ip="ip -color=auto"

##### Backup Aliases #####
# backup:     Backs up home directory to second SSD at /mnt/data/Backup/
# backup-dry: Dry run of the SSD backup — simulates without copying anything.
#             Always run this first to verify before a real backup.
#
# Both commands use rsync with:
#   -r  recursive
#   -l  preserve symlinks as symlinks
#   -t  preserve timestamps (needed for rsync to detect changes efficiently)
#   -v  verbose (show files being transferred)
#   -P  show progress + keep partially transferred files if interrupted
#   --delete  mirror deletions from source to destination
#
# Excludes: .cache, Music, Downloads, .var, Trash, BraveSoftware,
#           Chromium, torbrowser, flatpak, snap, Desktop
#
# Note: owner, group and permissions are intentionally NOT preserved
# so backups restore cleanly on different machines/users
alias backup='rsync -rltvP --delete --exclude=".cache" --exclude="Music" --exclude="Downloads" --exclude=".var" --exclude=".local/share/Trash" --exclude=".config/BraveSoftware" --exclude=".config/chromium" --exclude=".config/torbrowser" --exclude=".local/share/flatpak" --exclude="snap" --exclude="Desktop" ~/ /mnt/data/Backup/'

alias backup-dry='rsync -rltvP --delete --dry-run --exclude=".cache" --exclude="Music" --exclude="Downloads" --exclude=".var" --exclude=".local/share/Trash" --exclude=".config/BraveSoftware" --exclude=".config/chromium" --exclude=".config/torbrowser" --exclude=".local/share/flatpak" --exclude="snap" --exclude="Desktop" ~/ /mnt/data/Backup/'

##### Keybindings #####
# Ctrl + Left/Right to jump words
bindkey "^[[1;5C" forward-word
bindkey "^[[1;5D" backward-word
bindkey "^[[5C" forward-word
bindkey "^[[5D" backward-word

##### Syntax Highlighting #####
source ~/.config/zsh/plugins/fast-syntax-highlighting/fast-syntax-highlighting.plugin.zsh
```

### Configuration breakdown

**Prompt (`PS1`)**

The prompt `[user@hostname ~/current/path]$` is drawn entirely in colour using Zsh's `%F{colour}` and `$fg[]` variables. `autoload -U colors && colors` must be called first to enable these. `%B` and `%b` make the whole prompt bold.

| Colour | Part |
|---|---|
| Red | Brackets `[` `]` |
| Yellow | Username (`%n`) |
| Green | `@` and hostname (`%M`) |
| Blue | Hostname |
| Magenta | Current directory (`%~`) |

**Shell options**

| Option | Effect |
|---|---|
| `setopt autocd` | Type a directory name to `cd` into it without typing `cd` |
| `setopt interactive_comments` | Allow `#` comments in interactive shell (useful for documenting commands) |
| `stty stop undef` | Unbinds `Ctrl+S` which would otherwise freeze the terminal — frees it for other use |

**History**

| Setting | Effect |
|---|---|
| `HISTSIZE=10000000` | Keep up to 10 million entries in memory during the session |
| `SAVEHIST=10000000` | Save up to 10 million entries to disk in `~/.zsh_history` |
| `inc_append_history` | Write each command to history immediately, not only when the shell exits |
| `share_history` | Share history across all open terminal sessions in real time |
| `hist_ignore_dups` | Don't record a command if it's the same as the previous one |
| `hist_reduce_blanks` | Strip extra whitespace from history entries before saving |

**Completion**

| Setting | Effect |
|---|---|
| `zstyle menu select` | Shows a navigable menu when there are multiple completions — use arrow keys to select |
| `matcher-list` | Makes completion case-insensitive and handles partial matches on word separators |
| `_comp_options+=(globdots)` | Makes completion show hidden files (dotfiles) without needing to type the `.` first |

**Aliases**

| Alias | Expands to | Effect |
|---|---|---|
| `cp` | `cp -iv` | Prompts before overwriting (`-i`), shows what was copied (`-v`) |
| `mv` | `mv -iv` | Prompts before overwriting, shows what was moved |
| `rm` | `rm -vI` | Shows what is deleted; prompts once when removing 3+ files (safer than `-i` per file) |
| `bc` | `bc -ql` | Opens the calculator in quiet mode (`-q`) without the copyright banner |
| `mkd` | `mkdir -pv` | Creates nested directories and shows each one created |
| `yt` | `yt-dlp --embed-metadata -i` | Downloads video with metadata embedded; `-i` ignores errors and continues |
| `yta` | `yt -x -f bestaudio/best` | Downloads audio only in the best available format |
| `ytt` | `yt --skip-download --write-thumbnail` | Downloads only the thumbnail image without downloading the video |
| `ffmpeg` | `ffmpeg -hide_banner` | Suppresses the long version/copyright header that ffmpeg prints by default |
| `ls` | `ls -hN --color=auto --group-directories-first` | Human-readable sizes, literal special characters, colour output, directories listed first |
| `grep` | `grep --color=auto` | Highlights matches in colour |
| `diff` | `diff --color=auto` | Colours additions and deletions |
| `ccat` | `highlight --out-format=ansi` | Syntax-highlighted cat — requires the `highlight` package |
| `ip` | `ip -color=auto` | Coloured output for the `ip` command |

**Backup function — `backup-usb`**

An interactive function for backing up the home directory to a USB drive. It lists all mounted drives under `/media/robin/` first so you can see what is available, then prompts for the destination path. Pressing Enter uses `Backup-Linux/Backup` as the default. The actual copy is done with rsync using the same flags as the `backup` alias below.

**Backup aliases — `backup` and `backup-dry`**

Two rsync aliases for backing up the home directory to a second internal SSD mounted at `/mnt/data/Backup/`.

`backup-dry` performs a dry run — it shows exactly what *would* be copied or deleted without touching anything. Always run this first to verify the operation before running `backup` for real.

**rsync flags used:**

| Flag | Effect |
|---|---|
| `-r` | Recursive — copy all subdirectories |
| `-l` | Preserve symlinks as symlinks rather than following them |
| `-t` | Preserve timestamps — allows rsync to detect unchanged files efficiently on subsequent runs |
| `-v` | Verbose — print each file as it is transferred |
| `-P` | Show transfer progress and keep partially transferred files if interrupted |
| `--delete` | Mirror deletions — removes files from the destination that no longer exist in the source |
| `--dry-run` | Simulate the operation without making any changes (backup-dry only) |

Files and directories excluded from both backups: `.cache`, `Music`, `Downloads`, `.var`, `Trash`, `BraveSoftware`, `chromium`, `torbrowser`, `flatpak`, `snap`, `Desktop`.

> Owner, group, and permissions are intentionally not preserved (no `-a` / `--archive` flag). This means backups restore cleanly even on a different machine or under a different username.

**Keybindings**

Binds `Ctrl + Left` and `Ctrl + Right` to jump backward and forward by word. The two entries per direction handle different terminal emulators that send different escape codes for the same key combination.

**Syntax highlighting**

Sources `fast-syntax-highlighting` from the plugin path set up in Step 3. This must be the **last line** in `.zshrc` — sourcing it earlier can interfere with completion and other settings.

### Verify everything works

Reload your config without restarting:

```bash
source ~/.zshrc
```

Check syntax highlighting is active by typing a command — valid commands should appear in green, invalid ones in red. Check word-jump with `Ctrl + Left/Right`. Run `backup-dry` to confirm the rsync dry run works.

---

## 12. Cinnamon Desktop Tweaks

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

## 13. Unified Qt & GTK Theming

### What this does

By default on Linux Mint, Qt applications (such as KeePassXC, qBittorrent, and others) use their own built-in widget style and ignore your GTK theme entirely — appearing light and unstyled while the rest of your desktop is dark. This section makes Qt apps read your GTK2 theme so they match the rest of the desktop. No `qt5ct` or Kvantum required — `qt5-gtk-platformtheme` is already installed on Linux Mint.

### Step 1 — Set environment variables in `/etc/environment`

These two variables tell Qt which theme bridge plugin to load and which widget style to draw with. Both must use `gtk2` — mixing `gtk2` and `gtk3` here causes a segfault.

```bash
sudo nano /etc/environment
```

Add these two lines (no `export` keyword — this file uses plain `KEY=value` syntax):

```
QT_QPA_PLATFORMTHEME=gtk2
QT_STYLE_OVERRIDE=gtk2
```

Save and close. These apply system-wide to all users at every login.

### Step 2 — Create `~/.config/gtk-3.0/settings.ini`

Qt's bridge reads font, cursor, and icon settings from the standard GTK3 config file. Cinnamon creates this file automatically but does not always write the theme name into it — add it if missing.

```bash
mkdir -p ~/.config/gtk-3.0
nano ~/.config/gtk-3.0/settings.ini
```

```ini
[Settings]
gtk-font-name=Inter Medium 11
gtk-cursor-theme-name=DMZ-Black
gtk-icon-theme-name=Mint-Y-Sand
gtk-theme-name=Flat-Remix-GTK-Blue-Darkest-Solid-Cinnamon-Patch
```

> Adjust the font, cursor, icon, and theme values to match whatever you have set in Cinnamon System Settings.

### Step 3 — Add the theme name to `~/.gtkrc-2.0`

The GTK2 widget style (`QT_STYLE_OVERRIDE=gtk2`) reads from `~/.gtkrc-2.0` rather than the GTK3 settings file. Cinnamon creates this file but does not write the theme name into it. Append it:

```bash
echo 'gtk-theme-name="Flat-Remix-GTK-Blue-Darkest-Solid-Cinnamon-Patch"' >> ~/.gtkrc-2.0
```

The file should now look similar to this:

```
###############################################
# Created by cinnamon-settings - please do not edit or reformat.
#
style "cs-scrollbar-style" {
}
class "GtkScrollbar" style "cs-scrollbar-style"
###############################################
gtk-theme-name="Flat-Remix-GTK-Blue-Darkest-Solid-Cinnamon-Patch"
```

### Step 4 — Log out and back in

The `/etc/environment` variables are only picked up at login. After logging back in, Qt apps will follow your dark GTK theme automatically without any extra flags.

### Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Qt apps still look light after relogin | `/etc/environment` not read | Confirm with `echo $QT_QPA_PLATFORMTHEME` |
| Qt app segfaults on launch | gtk2/gtk3 conflict | Ensure both variables use `gtk2`, not a mix |
| Theme applied but wrong colour | `~/.gtkrc-2.0` missing theme name | Re-run the `echo` command in Step 3 |
| App ignores system theme entirely | App has internal theme override | Set theme to "System" in the app's own preferences |

---

## 14. Font Configuration

Good font configuration on Linux Mint covers three things: rendering quality (how fonts are drawn on screen), metric-compatible substitutions (so documents and web pages that request Windows fonts render with correct spacing), and choosing readable system fonts for the desktop UI.

Linux Mint Cinnamon ships with sensible rendering defaults, but the substitution rules and font choices below improve on them meaningfully.

### 14.1 Install fonts

Most of the fonts below are available directly from the Ubuntu repositories. A few must be downloaded manually as noted.

**From apt:**

```bash
sudo apt install -y \
  fonts-liberation \
  fonts-liberation2 \
  fonts-liberation-sans-narrow \
  fonts-croscore \
  fonts-crosextra-carlito \
  fonts-crosextra-caladea \
  fonts-dejavu \
  fonts-noto \
  fonts-noto-color-emoji \
  fonts-inter \
  fonts-linuxlibertine \
  fonts-jetbrains-mono \
  fonts-texgyre
```

**Downloaded manually** — these are not in the Ubuntu repos:

| Font | Purpose | Download |
|---|---|---|
| Gelasio | Metric-compatible replacement for Georgia | [fonts.google.com/specimen/Gelasio](https://fonts.google.com/specimen/Gelasio) |
| Comic Neue | Replacement for Comic Sans MS | [fonts.google.com/specimen/Comic+Neue](https://fonts.google.com/specimen/Comic+Neue) |
| Selawik | Replacement for Segoe UI | [github.com/microsoft/Selawik/releases/tag/1.01](https://github.com/microsoft/Selawik/releases/tag/1.01) |

To install downloaded fonts, copy the `.ttf` files to your user font directory and rebuild the font cache:

```bash
mkdir -p ~/.local/share/fonts
cp ~/Downloads/*.ttf ~/.local/share/fonts/
fc-cache -fv
```

Verify a font installed correctly:

```bash
fc-match "Gelasio"
fc-match "Comic Neue"
fc-match "Selawik"
```

### 14.2 fontconfig — `~/.config/fontconfig/fonts.conf`

This file controls rendering quality and sets up metric-compatible substitutions so that documents and web pages requesting Windows fonts get correctly-sized free alternatives instead of random fallbacks.

Place it at `~/.config/fontconfig/fonts.conf` (per-user) or `/etc/fonts/local.conf` (system-wide). After saving, run `fc-cache -fv`.

```xml
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "urn:fontconfig:fonts.dtd">
<!--
╔══════════════════════════════════════════════════════════════════╗
║         fonts.conf — Optimized for Readability on Linux          ║
║                                                                  ║
║  Place at:  ~/.config/fontconfig/fonts.conf  (per-user)          ║
║         or  /etc/fonts/local.conf            (system-wide)       ║
║                                                                  ║
║  After installing, run:  fc-cache -fv                            ║
║                                                                  ║
║  Recommended packages (Mint/Ubuntu):                             ║
║    fonts-liberation  fonts-carlito  fonts-caladea                ║
║    fonts-dejavu  fonts-noto  fonts-noto-color-emoji              ║
║    + Comic Neue from fonts.google.com → ~/.local/share/fonts/    ║
║                                                                  ║
║  For MS fonts (if legally obtained):                             ║
║    ttf-ms-fonts (AUR) or copy from a Windows install             ║
╚══════════════════════════════════════════════════════════════════╝
-->
<fontconfig>

  <!-- ============================================================
       SECTION 1: RENDERING QUALITY
       Best settings for modern LCD monitors (RGB subpixel layout).
       Change rgba to "bgr" if your screen uses that layout.
       Test your subpixel layout at: https://www.lagom.nl/lcd-test/subpixel.php
       ============================================================ -->

  <match target="font">
    <edit name="antialias" mode="assign"><bool>true</bool></edit>
  </match>

  <match target="font">
    <edit name="hinting" mode="assign"><bool>true</bool></edit>
  </match>

  <match target="font">
    <edit name="hintstyle" mode="assign"><const>hintslight</const></edit>
  </match>

  <match target="font">
    <edit name="rgba" mode="assign"><const>rgb</const></edit>
  </match>

  <match target="font">
    <edit name="lcdfilter" mode="assign"><const>lcddefault</const></edit>
  </match>

  <match target="font">
    <edit name="autohint" mode="assign"><bool>false</bool></edit>
  </match>

  <match target="font">
    <edit name="embeddedbitmap" mode="assign"><bool>false</bool></edit>
  </match>

  <match target="font">
    <test name="family" qual="any"><string>Noto Color Emoji</string></test>
    <edit name="embeddedbitmap" mode="assign"><bool>true</bool></edit>
  </match>


  <!-- ============================================================
       SECTION 2: DEFAULT FONT FAMILIES
       ============================================================ -->

  <alias>
    <family>sans-serif</family>
    <prefer>
      <family>Liberation Sans</family>
      <family>Arimo</family>
      <family>DejaVu Sans</family>
      <family>Noto Sans</family>
    </prefer>
  </alias>

  <alias>
    <family>serif</family>
    <prefer>
      <family>Liberation Serif</family>
      <family>Tinos</family>
      <family>DejaVu Serif</family>
      <family>Noto Serif</family>
    </prefer>
  </alias>

  <alias>
    <family>monospace</family>
    <prefer>
      <family>Liberation Mono</family>
      <family>DejaVu Sans Mono</family>
      <family>Cousine</family>
      <family>Noto Sans Mono</family>
    </prefer>
  </alias>

  <alias>
    <family>sans</family>
    <prefer>
      <family>Liberation Sans</family>
      <family>Arimo</family>
    </prefer>
  </alias>

  <alias>
    <family>emoji</family>
    <prefer><family>Noto Color Emoji</family></prefer>
  </alias>


  <!-- ============================================================
       SECTION 3: METRIC-COMPATIBLE SUBSTITUTIONS
       ============================================================ -->

  <match target="pattern">
    <test name="family" qual="any"><string>Arial</string></test>
    <edit name="family" mode="assign" binding="same"><string>Liberation Sans</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Helvetica</string></test>
    <edit name="family" mode="assign" binding="same"><string>Liberation Sans</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Arial Narrow</string></test>
    <edit name="family" mode="assign" binding="same"><string>Liberation Sans Narrow</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Times New Roman</string></test>
    <edit name="family" mode="assign" binding="same"><string>Liberation Serif</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Times</string></test>
    <edit name="family" mode="assign" binding="same"><string>Liberation Serif</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Courier New</string></test>
    <edit name="family" mode="assign" binding="same"><string>Liberation Mono</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Courier</string></test>
    <edit name="family" mode="assign" binding="same"><string>Liberation Mono</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Calibri</string></test>
    <edit name="family" mode="assign" binding="same"><string>Carlito</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Cambria</string></test>
    <edit name="family" mode="assign" binding="same"><string>Caladea</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Georgia</string></test>
    <edit name="family" mode="assign" binding="same"><string>Gelasio</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Segoe UI</string></test>
    <edit name="family" mode="assign" binding="same"><string>Selawik</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>MS Sans Serif</string></test>
    <edit name="family" mode="assign" binding="same"><string>Liberation Sans</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Microsoft Sans Serif</string></test>
    <edit name="family" mode="assign" binding="same"><string>Liberation Sans</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>MS Serif</string></test>
    <edit name="family" mode="assign" binding="same"><string>Liberation Serif</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Century Gothic</string></test>
    <edit name="family" mode="assign" binding="same"><string>TeX Gyre Adventor</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Comic Sans MS</string></test>
    <edit name="family" mode="assign" binding="same"><string>Comic Neue</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Comic Sans</string></test>
    <edit name="family" mode="assign" binding="same"><string>Comic Neue</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Palatino Linotype</string></test>
    <edit name="family" mode="assign" binding="same"><string>TeX Gyre Pagella</string></edit>
  </match>

  <match target="pattern">
    <test name="family" qual="any"><string>Book Antiqua</string></test>
    <edit name="family" mode="assign" binding="same"><string>TeX Gyre Pagella</string></edit>
  </match>


  <!-- ============================================================
       SECTION 4: PER-FONT RENDERING OVERRIDES
       ============================================================ -->

  <match target="font">
    <test name="family" qual="any"><string>Noto Sans</string></test>
    <edit name="hintstyle" mode="assign"><const>hintfull</const></edit>
  </match>

  <match target="font">
    <test name="family" qual="any"><string>Noto Serif</string></test>
    <edit name="hintstyle" mode="assign"><const>hintfull</const></edit>
  </match>

  <match target="font">
    <test name="family" qual="any"><string>Liberation Sans</string></test>
    <edit name="hintstyle" mode="assign"><const>hintfull</const></edit>
  </match>

  <match target="font">
    <test name="family" qual="any"><string>Liberation Mono</string></test>
    <edit name="hintstyle" mode="assign"><const>hintfull</const></edit>
  </match>


  <!-- ============================================================
       SECTION 5: HiDPI / 4K DISPLAY OVERRIDE
       On displays above ~192 DPI, hinting is counterproductive.
       Uncomment this block if you use a HiDPI screen.

  <match target="font">
    <edit name="hinting" mode="assign"><bool>false</bool></edit>
  </match>
  <match target="font">
    <edit name="hintstyle" mode="assign"><const>hintnone</const></edit>
  </match>
       ============================================================ -->

</fontconfig>
```

**Verify rendering settings are active:**

```bash
fc-match --verbose sans | grep -E 'hintstyle|rgba|lcdfilter|antialias|embeddedbitmap'
```

**Verify substitutions are working:**

```bash
for family in \
  serif sans-serif monospace \
  Arial Helvetica "Arial Narrow" \
  Verdana Tahoma \
  "Times New Roman" Times \
  "Courier New" Courier \
  Calibri Cambria Georgia \
  "Comic Sans MS" "Comic Sans" \
  "Segoe UI" \
  "MS Sans Serif" "Microsoft Sans Serif" "MS Serif" \
  "Century Gothic" "Palatino Linotype" "Book Antiqua"; do
  echo -n "$family: "
  fc-match "$family"
done
```

### 14.3 Cinnamon desktop font settings

Open **System Settings → Fonts** and set the following:

| Setting | Font | Size |
|---|---|---|
| Default font | Inter Medium | 11 |
| Desktop font | Inter Medium | 11 |
| Document font | Linux Libertine O Regular | 12 |
| Monospace font | JetBrains Mono Regular | 12 |
| Window title font | Inter Medium | 11 |
| Hinting | Slight | — |
| Antialiasing | Rgba | — |
| RGBA order | RGB | — |

### 14.4 LibreOffice font settings

Open **Tools → Options → LibreOffice Writer → Basic Fonts (Western)** and set:

| Setting | Font | Size |
|---|---|---|
| Default | Carlito | 11pt |
| Heading | Caladea | 14pt |
| List | Carlito | 11pt |
| Caption | Carlito | 10pt |
| Index | Carlito | 11pt |

---

## 15. Applying Everything & Verifying

Here is a single block you can run after creating all the files above:

```bash
# Apply sysctl settings (tuning + swappiness)
sudo sysctl --system

# Verify key hardening values (some require sudo to read)
sudo sysctl \
  kernel.kptr_restrict \
  kernel.dmesg_restrict \
  kernel.unprivileged_bpf_disabled \
  net.core.bpf_jit_harden \
  kernel.kexec_load_disabled \
  kernel.yama.ptrace_scope \
  vm.mmap_rnd_bits \
  vm.mmap_rnd_compat_bits \
  fs.protected_fifos \
  net.ipv4.tcp_sack \
  net.ipv6.conf.all.accept_ra

# Verify GRUB boot parameters are active (after reboot)
cat /proc/cmdline

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

# Verify hosts blocklist is active
grep -c "^0.0.0.0" /etc/hosts

# Verify hosts update timer is running
systemctl list-timers hosts-update.timer

# Verify fast-syntax-highlighting is loaded
echo $ZSH_HIGHLIGHT_VERSION 2>/dev/null || echo "Check: source ~/.zshrc and type a command"

# Verify Qt theming env vars are active (after relogin)
echo $QT_QPA_PLATFORMTHEME
echo $QT_STYLE_OVERRIDE

# Verify font rendering
fc-match --verbose sans | grep -E 'hintstyle|rgba|lcdfilter|antialias|embeddedbitmap'

# Verify font substitutions
for family in \
  serif sans-serif monospace \
  Arial Helvetica "Arial Narrow" \
  Verdana Tahoma \
  "Times New Roman" Times \
  "Courier New" Courier \
  Calibri Cambria Georgia \
  "Comic Sans MS" "Comic Sans" \
  "Segoe UI" \
  "MS Sans Serif" "Microsoft Sans Serif" "MS Serif" \
  "Century Gothic" "Palatino Linotype" "Book Antiqua"; do
  echo -n "$family: "; fc-match "$family"
done

# Verify Brave policies
# Open Brave and visit brave://policy/
```

All settings (except the active MAC address) will also **persist across reboots** automatically — no additional steps needed.

---

## 16. Quick Reference Cheatsheet

| File to create/edit | Location | Command to apply |
|---|---|---|
| Hosts blocklist (initial install) | `/etc/hosts` | `sudo curl -fsSL https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts -o /etc/hosts` |
| `hosts-update.service` | `/etc/systemd/system/hosts-update.service` | `sudo systemctl start hosts-update.service` |
| `hosts-update.timer` | `/etc/systemd/system/hosts-update.timer` | `sudo systemctl enable --now hosts-update.timer` |
| GRUB boot parameters | `/etc/default/grub` | `sudo update-grub` then reboot |
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
| Install Kitty + remove gnome-terminal | — (CLI only) | `sudo apt install kitty && sudo apt purge gnome-terminal gnome-terminal-data` |
| Kitty config | `~/.config/kitty/kitty.conf` | Reload with `Ctrl+Shift+F5` inside Kitty |
| Install Brave | — (CLI only) | See Section 10 |
| Brave policies | `/etc/brave/policies/managed/policies.json` | Restart Brave, verify at `brave://policy/` |
| Set default shell to Zsh | — (CLI only) | `chsh -s $(which zsh)` |
| Zsh plugin directory | `~/.config/zsh/plugins/` | `mkdir -p ~/.config/zsh/plugins` |
| fast-syntax-highlighting | `~/.config/zsh/plugins/fast-syntax-highlighting/` | `git clone ...` (see Section 11) |
| `.zshrc` | `~/.zshrc` | `source ~/.zshrc` |
| Qt theming env vars | `/etc/environment` | Log out and back in |
| GTK3 settings | `~/.config/gtk-3.0/settings.ini` | Immediate (per-app relaunch) |
| GTK2 theme name | `~/.gtkrc-2.0` | Immediate (per-app relaunch) |
| fontconfig | `~/.config/fontconfig/fonts.conf` | `fc-cache -fv` |
| Cinnamon font settings | — (GUI only) | System Settings → Fonts |
| LibreOffice fonts | — (GUI only) | Tools → Options → LibreOffice Writer → Basic Fonts |
| Cinnamon tweaks | — (GUI only) | System Settings / right-click taskbar |

---

*Guide written for Linux Mint Cinnamon. Most sections also work on Ubuntu 22.04/24.04 LTS, Pop!_OS 22.04, and other Ubuntu-based derivatives.*
