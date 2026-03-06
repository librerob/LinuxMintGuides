# Flat-Remix-GTK-Blue-Darkest-Solid-Cinnamon-Patch

A patched version of the [Flat Remix GTK theme](https://www.gnome-look.org/p/1214931/) by Daniel Ruiz de Alegría, customised for **Linux Mint Cinnamon** with full compatibility for modern Cinnamon dialogs and UI elements.

> **Variant:** Blue highlights — Darkest Solid (AMOLED pitch black background)

---

## What this patch adds

The upstream Flat Remix GTK theme was designed primarily for GNOME. Newer versions of Cinnamon introduced several dialogs and UI components that either render unstyled or look broken when using the original theme. This patched version fixes all of these.

### Patches applied

| Component | Issue fixed |
|---|---|
| Cinnamon colour picker dialog | Unstyled / defaulted to system colours |
| Cinnamon date/time dialog | White background breaking the dark theme |
| Cinnamon user accounts dialog | Mixed styling with light elements |
| Cinnamon network dialogs | Inconsistent entry field and button colours |
| GTK4 adwaita overrides | Modern GTK4 apps ignoring the theme entirely |
| `gtk-4.0/gtk.css` | Linked in via `~/.config/gtk-4.0` for GTK4 app coverage |
| Selection and accent colours | Unified blue highlight colour across GTK3 and GTK4 |
| Scrollbar styling | Cinnamon's newer scrollbar implementation |
| Header bar buttons | Inconsistent close/minimise/maximise button rendering |

---

## Installation

### 1. Clone or download the theme

Clone from GitHub:

```bash
git clone https://github.com/librerob/LinuxMintGuides.git /tmp/LinuxMintGuides
```

Or from Codeberg:

```bash
git clone https://codeberg.org/librerob/LinuxMintGuides.git /tmp/LinuxMintGuides
```

The theme files are located under `guides/gtk-theme/` in the repository. You can also browse them directly at [codeberg.org/librerob/LinuxMintGuides/src/branch/main/guides/gtk-theme](https://codeberg.org/librerob/LinuxMintGuides/src/branch/main/guides/gtk-theme).

### 2. Copy the theme to your themes directory

```bash
mkdir -p ~/.themes
cp -r /tmp/LinuxMintGuides/guides/gtk-theme/Flat-Remix-GTK-Blue-Darkest-Solid-Cinnamon-Patch ~/.themes/
```

### 3. Apply GTK4 overrides

GTK4 applications (like GNOME Text Editor, Files in newer versions, etc.) do not read `~/.themes` — they use `~/.config/gtk-4.0` instead. Link the theme's GTK4 stylesheet there:

```bash
mkdir -p ~/.config/gtk-4.0
ln -sf ~/.themes/Flat-Remix-GTK-Blue-Darkest-Solid-Cinnamon-Patch/gtk-4.0/gtk.css ~/.config/gtk-4.0/gtk.css
ln -sf ~/.themes/Flat-Remix-GTK-Blue-Darkest-Solid-Cinnamon-Patch/gtk-4.0/gtk-dark.css ~/.config/gtk-4.0/gtk-dark.css
```

### 4. Apply the theme

Via terminal:

```bash
gsettings set org.gnome.desktop.interface gtk-theme "Flat-Remix-GTK-Blue-Darkest-Solid-Cinnamon-Patch"
```

Or via **System Settings → Themes → Applications** — select `Flat-Remix-GTK-Blue-Darkest-Solid-Cinnamon-Patch` from the dropdown.

### 5. Apply for root applications (optional)

If you run applications as root (e.g. GParted, Synaptic) and want them to use the same theme:

```bash
sudo ln -s ~/.themes/Flat-Remix-GTK-Blue-Darkest-Solid-Cinnamon-Patch /usr/share/themes/
```

---

## Flatpak support (optional)

Flatpak apps run in a sandbox and cannot access your `~/.themes` folder by default. Allow them to read your installed themes:

```bash
flatpak override --user --filesystem=xdg-config/gtk-4.0 --filesystem=home/.themes/
```

---

## Optional: matching icon theme

Two icon themes pair well with this GTK theme:

- **[Flat Remix icon theme](https://www.opendesktop.org/p/1012430)** — by the same author, consistent visual language
- **Papirus Dark** — already pre-installed on Linux Mint, works well and blends naturally

To set via terminal (replace with your preferred icon theme name):

```bash
gsettings set org.gnome.desktop.interface icon-theme "Papirus-Dark"
```

---

## Compatibility

| Environment | Status |
|---|---|
| Linux Mint 21.x (Cinnamon) | ✅ Fully tested |
| Linux Mint 22.x (Cinnamon) | ✅ Fully tested |
| Ubuntu 22.04 / 24.04 (GNOME) | ⚠️ Works but Cinnamon patches are not relevant |
| GTK3 applications | ✅ |
| GTK4 applications | ✅ (via `~/.config/gtk-4.0` symlink) |
| Flatpak applications | ✅ (with `flatpak override` command above) |

---

## Credits

- Original theme: **[Flat Remix GTK](https://www.gnome-look.org/p/1214931/)** by [Daniel Ruiz de Alegría](https://github.com/daniruiz)
- Cinnamon patches: applied manually to fix dialog and UI rendering in Linux Mint Cinnamon
- License: GPL-3.0 (same as the upstream theme)
