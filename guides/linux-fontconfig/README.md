# How to Set Default Fonts and Font Aliases on Linux

> A complete reference for configuring system and web fonts using **fontconfig** on Linux systems.

---

## 📌 Overview

On Linux, setting default fonts *in applications* is easy — but ensuring preferred fonts are used consistently (e.g., in browsers for all websites) often requires deeper configuration.

Many popular websites like Google, Yahoo, Facebook, and GitHub specify font families such as Arial, Helvetica, or Times New Roman — which don't exist by default on most Linux systems.

This guide shows how to:

* Install the required fonts
* Inspect current default fonts
* Alias common Microsoft font names to your own fonts
* Ensure font priorities apply everywhere, including websites

---

## 📦 Installing the Fonts (Ubuntu / Debian)

This configuration uses three font packages:

**`fonts-croscore`** — Arimo, Tinos, Cousine — metric-compatible with Arial, Times New Roman, and Courier New.

**`fonts-crosextra-caladea`** — metric-compatible with Cambria.

**`fonts-crosextra-carlito`** — metric-compatible with Calibri.
```bash
sudo apt install fonts-croscore fonts-crosextra-caladea fonts-crosextra-carlito
```

For Noto fonts (Unicode fallback and emoji support):
```bash
sudo apt install fonts-noto fonts-noto-color-emoji
```
---

## ⚙️ Deploying `fonts.conf`

Download the `fonts.conf` from this repo and place it at:
```
~/.config/fontconfig/fonts.conf
```

Then rebuild the cache and re-login:
```bash
fc-cache -fv
sudo dpkg-reconfigure fontconfig
sudo dpkg-reconfigure fontconfig-config
```

---

## 🔍 Inspect Current Default Fonts

Use `fc-match` to verify the configuration is applied:
```bash
for family in serif sans-serif monospace Arial Helvetica Verdana Tahoma "Comic Sans MS" "Times New Roman" "Times" "Courier New" "Cambria" "Calibri"; do
  echo -n "$family: "
  fc-match "$family"
done
```

Good output should show Arimo, Tinos, Cousine, Caladea, or Carlito for each entry.

---

## 🔁 Font Substitution Table

| Requested Font | Replaced With | Package |
|---|---|---|
| Arial | Arimo | `fonts-croscore` |
| Helvetica | Arimo | `fonts-croscore` |
| Verdana | Arimo | `fonts-croscore` |
| Tahoma | Arimo | `fonts-croscore` |
| Comic Sans MS | Arimo | `fonts-croscore` |
| Times New Roman | Tinos | `fonts-croscore` |
| Times | Tinos | `fonts-croscore` |
| Courier New | Cousine | `fonts-croscore` |
| Cambria | Caladea | `fonts-crosextra-caladea` |
| Calibri | Carlito | `fonts-crosextra-carlito` |

---

## 💡 Notes & Troubleshooting

### Per-User vs System-Wide

* Per-user: `~/.config/fontconfig/fonts.conf` (recommended)
* System-wide: `/etc/fonts/local.conf` or a file in `/etc/fonts/conf.d/`

### Why Noto Fallbacks?

Noto fonts cover virtually all Unicode scripts. Including them as fallbacks ensures characters not present in Arimo, Tinos, or Cousine (e.g., CJK, Arabic, Devanagari) still render correctly, and emoji display works system-wide.

### Deprecated Syntax

Older examples online use `<match target="pattern" name="family">`, which causes:
```
Fontconfig error: invalid attribute 'name'
```

Use `<match>` without `target` and `name` attributes to avoid this.