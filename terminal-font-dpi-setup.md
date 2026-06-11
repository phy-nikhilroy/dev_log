# Unifying Terminal Font Rendering Across Alacritty and Kitty on Arch Linux (bspwm)

**Date:** 2026-06-11  
**System:** ThinkPad T14s — Arch Linux, bspwm, X11  
**Display:** 1920×1080 @ 309mm×174mm (eDP-1)

---

## Background

I run a bspwm-based Arch Linux setup where all my config files live in a `~/dotfiles` directory and are symlinked into `~/.config/`. This gives me a single source of truth that I can version-control and reproduce on any new machine.

This log covers:
1. Persisting Alacritty's font size into dotfiles instead of relying on runtime key bindings
2. Bootstrapping a Kitty config in the same dotfiles structure with a proper symlink
3. Diagnosing and fixing a confusing font size mismatch between the two terminals caused by DPI handling differences

---

## Part 1 — Persisting the Alacritty Font Size

### The Problem

Every time I launched Alacritty, the default font size was too large. My workaround was pressing `Ctrl+-` four times on startup to bring it to a comfortable size. This was obviously not sustainable — it wasn't persisted anywhere, and it had to be done manually for every new terminal window.

### Finding the Config

My Alacritty config lives in the dotfiles repo:

```
~/dotfiles/alacritty/alacritty.toml   ← source of truth
~/.config/alacritty/                  ← symlink → ~/dotfiles/alacritty/
```

At the time, the config looked like this:

```toml
[window]
opacity = 0.6

[[keyboard.bindings]]
key = "Return"
mods = "Shift"
chars = "\r"
```

No font section at all — Alacritty was using its built-in default of `11.25pt`.

### Calculating the Right Size

In Alacritty, each `Ctrl+-` keypress reduces the font size by **1 point**. Since I was pressing it 4 times, the comfortable size was:

```
11.25 (Alacritty default) - 4 = 7.25pt
```

I then bumped it up by 1 after testing:

```
7.25 + 1 = 8.25pt  ← final comfortable size
```

### The Fix

Added a `[font]` section to `~/dotfiles/alacritty/alacritty.toml`:

```toml
[font]
size = 8.25

[window]
opacity = 0.6

[[keyboard.bindings]]
key = "Return"
mods = "Shift"
chars = "\r"
```

Because `~/.config/alacritty` is already a symlink to `~/dotfiles/alacritty/`, the change takes effect immediately in any new Alacritty window. No restart of any service needed.

---

## Part 2 — Setting Up Kitty in Dotfiles

### Goal

Mirror the same dotfiles-first pattern for Kitty:
- Config lives in `~/dotfiles/kitty/kitty.conf`
- `~/.config/kitty` is a symlink to `~/dotfiles/kitty/`

### Step 1 — Check the Current State

```bash
ls /home/nikhil/.config/kitty     # check if kitty config dir exists
cat /home/nikhil/.config/kitty/kitty.conf   # check for existing config
```

There was no existing Kitty config, but `~/.config/kitty` existed as an **empty directory** (likely created by Kitty on first launch).

### Step 2 — Detect the Active Font in Alacritty

Since the Alacritty config didn't specify a font family, it was using the system `monospace` alias resolved by **fontconfig**. To find out what that resolves to:

```bash
fc-match monospace
# → NotoSansMono-Regular.ttf: "Noto Sans Mono" "Regular"

fc-match "monospace:size=8.25"
# → NotoSansMono-Regular.ttf: "Noto Sans Mono" "Regular"
```

**Noto Sans Mono** is the system monospace font. I also checked available variants:

```bash
fc-match "Noto Sans Mono:style=Regular"
# → NotoSansMono-Regular.ttf: "Noto Sans Mono" "Regular"

fc-match "Noto Sans Mono:style=Bold"
# → NotoSansMono-Bold.ttf: "Noto Sans Mono" "Bold"

fc-match "Noto Sans Mono:style=Italic"
# → NotoSansMono-Regular.ttf: "Noto Sans Mono" "Regular"   ← falls back, no italic

fc-match "Noto Sans Mono:style=Bold Italic"
# → NotoSansMono-Regular.ttf: "Noto Sans Mono" "Regular"   ← falls back, no bold italic
```

Noto Sans Mono only ships Regular and Bold — no italic or bold-italic variants on this system.

### Step 3 — Create the Kitty Dotfiles Directory and Config

```bash
mkdir -p ~/dotfiles/kitty
```

Initial `~/dotfiles/kitty/kitty.conf`:

```
font_family Noto Sans Mono
font_size 8.25

background_opacity 0.6
```

### Step 4 — Create the Symlink (and Fix a Mistake)

The first attempt:

```bash
ln -s /home/nikhil/dotfiles/kitty /home/nikhil/.config/kitty
```

This silently went wrong. Because `~/.config/kitty` already existed as a directory, `ln -s` placed the symlink **inside** the directory rather than replacing it. The result was:

```
~/.config/kitty/          ← still a regular directory
└── kitty → ~/dotfiles/kitty    ← symlink nested inside, wrong
```

Verified with:

```bash
ls -la /home/nikhil/.config/kitty
# lrwxrwxrwx  kitty -> /home/nikhil/dotfiles/kitty   ← inside the dir, not the dir itself
```

And confirmed at the `.config` level:

```bash
ls -la /home/nikhil/.config/ | grep kitty
# drwxr-xr-x  kitty    ← still shows as a plain directory
```

**The fix** — remove the nested symlink and the empty directory, then re-create correctly:

```bash
rm /home/nikhil/.config/kitty/kitty
rmdir /home/nikhil/.config/kitty
ln -s /home/nikhil/dotfiles/kitty /home/nikhil/.config/kitty
```

Verified:

```bash
ls -la /home/nikhil/.config/ | grep kitty
# lrwxrwxrwx  kitty -> /home/nikhil/dotfiles/kitty   ← correct
```

### Step 5 — Full Kitty Config Matching All Alacritty Defaults

Beyond just font and opacity, I also needed to account for defaults that **differ** between Alacritty and Kitty:

| Setting | Alacritty default | Kitty default |
|---|---|---|
| Scrollback lines | 10,000 | 2,000 |
| Cursor shape | Block | Block (same) |
| Cursor blink | Off | Off (same) |
| Window padding | 0 | 0 (same) |

Final `~/dotfiles/kitty/kitty.conf`:

```
# Font
font_family      Noto Sans Mono
bold_font        Noto Sans Mono Bold
italic_font      auto
bold_italic_font auto
font_size        8.25

# Window
background_opacity 0.6
window_padding_width 0

# Scrollback (matches alacritty default of 10000)
scrollback_lines 10000

# Cursor (matches alacritty defaults: block, no blink)
cursor_shape block
cursor_blink_interval 0

# Key bindings
map shift+return send_text all \r
```

The `Shift+Return → \r` binding matches the Alacritty keybinding:

```toml
# Alacritty equivalent:
[[keyboard.bindings]]
key = "Return"
mods = "Shift"
chars = "\r"
```

---

## Part 3 — The DPI Mystery: Why the Same Font Size Looked Completely Different

### The Symptom

After setting up Kitty with `font_size 8.25` to match Alacritty's `size = 8.25`, the fonts looked **drastically different**. Kitty's text was noticeably smaller than Alacritty's despite using the identical number.

### Investigation

#### Step 1 — Check Display Info

```bash
xrandr | grep -E "connected|[0-9]+x[0-9]+"
```

Output:

```
eDP-1 connected primary 1920x1080+0+0 (normal left inverted right x axis y axis) 309mm x 174mm
   1920x1080     60.05*+
```

Key facts extracted:
- Resolution: **1920×1080 pixels**
- Physical size: **309mm × 174mm**

#### Step 2 — Calculate Physical DPI

DPI (dots per inch) = pixels / (millimetres / 25.4)

```python
w_px, h_px = 1920, 1080
w_mm, h_mm = 309, 174

dpi_x = 1920 / (309 / 25.4)  # = 157.8 DPI
dpi_y = 1080 / (174 / 25.4)  # = 157.7 DPI
```

**Physical DPI ≈ 158 DPI.**

#### Step 3 — Understand How Font Size in "Points" Works

A typographic **point** is defined as **1/72 of an inch**. To render text at a given point size on a screen, the terminal must convert points → pixels using the display's DPI:

```
pixels = (points / 72) × DPI
```

So `8.25pt` at different DPIs:

| DPI | Formula | Pixels |
|---|---|---|
| 96 DPI (old X11 standard) | (8.25 / 72) × 96 | **11.0 px** |
| 158 DPI (physical) | (8.25 / 72) × 158 | **18.1 px** |

That's a **64% difference** in rendered pixel size — from the same `8.25` value.

#### Step 4 — How Each Terminal Handles DPI

**Alacritty** on X11 reads the DPI from the X server, which computes it from the screen's physical dimensions reported by `xrandr`. Since xrandr knows this display is 309mm×174mm at 1920×1080, the X server reports **~158 DPI**. Alacritty uses this to convert points to pixels.

```
Alacritty: 8.25pt × (158/72) = 18.1px per character height
```

**Kitty**, when no `Xft.dpi` resource is explicitly set, falls back to the legacy X11 assumption of **96 DPI** (the old "standard" DPI that predates modern HiDPI displays).

```
Kitty: 8.25pt × (96/72) = 11.0px per character height
```

This is the root cause. Same number. Completely different pixel output. Kitty renders characters at 11px, Alacritty at 18px — Kitty looks much smaller.

#### Step 5 — Verify the Maths

```bash
python3 -c "
w_px, h_px = 1920, 1080
w_mm, h_mm = 309, 174
physical_dpi = w_px / (w_mm / 25.4)
print(f'Physical DPI: {physical_dpi:.1f}')

alacritty_px = 8.25 * physical_dpi / 72
kitty_px = 8.25 * 96 / 72
print(f'Alacritty at {physical_dpi:.0f} DPI: {alacritty_px:.1f}px')
print(f'Kitty at 96 DPI: {kitty_px:.1f}px')
print(f'Kitty font_size to visually match Alacritty: {8.25 * physical_dpi / 96:.2f}pt')
"
```

Output:

```
Physical DPI: 157.8
Alacritty at 158 DPI: 18.1px
Kitty at 96 DPI: 11.0px
Kitty font_size to visually match Alacritty: 13.56pt
```

---

## Part 4 — The Fix: Setting `Xft.dpi` in `~/.Xresources`

### Why `Xft.dpi`?

`Xft` (X FreeType) is the font rendering layer that sits between X11 applications and FreeType. The `Xft.dpi` X resource tells all Xft-aware applications what DPI to use when converting typographic units (points) to screen pixels. Both Alacritty and Kitty respect this setting, making it the correct single place to define the display's DPI.

Setting `Xft.dpi: 158` means:
- Kitty now computes: `8.25 × (158/72) = 18.1px` — matching Alacritty
- Both terminals now agree on what `8.25pt` means in pixels

### Check for Existing Xresources

```bash
cat ~/.Xresources
# → FILE_NOT_FOUND (didn't exist)

xrdb -query | grep -i dpi
# → (no output — no DPI was set)
```

### Create `~/.Xresources`

```bash
# Created ~/.Xresources with:
Xft.dpi: 158
```

### Apply to the Current Session

```bash
xrdb -merge ~/.Xresources
```

`xrdb -merge` loads the file into the X server's resource database without wiping existing entries. Changes take effect for any newly opened application — running terminals are unaffected until reopened.

---

## Part 5 — Making It Permanent (Wiring into `~/.xinitrc`)

`xrdb -merge` only applies to the live X session. On next login, the setting would be gone. The standard place to load `~/.Xresources` on X11 startup is `~/.xinitrc` (or `~/.xprofile` for display-manager-based logins).

### Checking `~/.xinitrc`

```bash
cat ~/.xinitrc
```

```sh
#!/bin/sh

[ -f ~/.Xresources ] && xrdb -merge ~/.Xresources

export GTK_USE_PORTAL=1
systemctl --user import-environment DISPLAY XAUTHORITY

picom --backend glx --vsync --daemon &

dunst &

exec bspwm
```

**Already there.** Line 3 already conditionally merges `~/.Xresources` if it exists:

```sh
[ -f ~/.Xresources ] && xrdb -merge ~/.Xresources
```

This means:
- On every X session start (via `startx` or a display manager that sources `~/.xinitrc`), `~/.Xresources` is loaded
- `Xft.dpi: 158` will be active from the moment the session begins
- No further changes needed

---

## Final State of All Files

### `~/dotfiles/alacritty/alacritty.toml`

```toml
[font]
size = 8.25

[window]
opacity = 0.6

[[keyboard.bindings]]
key = "Return"
mods = "Shift"
chars = "\r"
```

### `~/dotfiles/kitty/kitty.conf`

```
# Font
font_family      Noto Sans Mono
bold_font        Noto Sans Mono Bold
italic_font      auto
bold_italic_font auto
font_size        8.25

# Window
background_opacity 0.6
window_padding_width 0

# Scrollback (matches alacritty default of 10000)
scrollback_lines 10000

# Cursor (matches alacritty defaults: block, no blink)
cursor_shape block
cursor_blink_interval 0

# Key bindings
map shift+return send_text all \r
```

### `~/.Xresources`

```
Xft.dpi: 158
```

### Symlinks in Place

```
~/.config/alacritty  →  ~/dotfiles/alacritty/
~/.config/kitty      →  ~/dotfiles/kitty/
```

---

## Reproducing This From Scratch

If you're setting this up on a fresh machine, here's the complete sequence:

```bash
# 1. Clone your dotfiles
git clone <your-dotfiles-repo> ~/dotfiles

# 2. Create alacritty symlink (if not already done)
ln -s ~/dotfiles/alacritty ~/.config/alacritty

# 3. Create kitty symlink
# If ~/.config/kitty already exists as a directory:
rm -rf ~/.config/kitty       # only if it's empty/generated, not hand-crafted
ln -s ~/dotfiles/kitty ~/.config/kitty

# 4. Get your display physical dimensions
xrandr | grep " connected"
# Note the WxH resolution and WxH mm values

# 5. Calculate your physical DPI
python3 -c "print(1920 / (309 / 25.4))"   # substitute your values
# → 157.8

# 6. Create ~/.Xresources with your DPI
echo "Xft.dpi: 158" > ~/.Xresources       # round to nearest integer

# 7. Apply immediately to the current session
xrdb -merge ~/.Xresources

# 8. Ensure ~/.xinitrc loads it on startup (check first)
grep Xresources ~/.xinitrc
# Should show: [ -f ~/.Xresources ] && xrdb -merge ~/.Xresources
# If not, add it near the top of ~/.xinitrc before exec bspwm
```

---

## Key Takeaways

- **`ln -s <src> <dest>`** — if `<dest>` already exists as a directory, the symlink is created **inside** it rather than replacing it. Always remove the target directory first with `rmdir` (safe if empty) before creating the symlink.
- **Font size in points is meaningless without a DPI context.** `8.25pt` means 11px at 96 DPI and 18px at 158 DPI. Terminals that disagree on DPI will render the same point size at different pixel sizes.
- **Alacritty** reads DPI from X11 screen dimensions (physical). **Kitty** falls back to 96 DPI if `Xft.dpi` is not set. Setting `Xft.dpi` in `~/.Xresources` fixes both to agree.
- **`~/.Xresources`** is the canonical place for X11 application-level settings. Load it via `xrdb -merge ~/.Xresources` in `~/.xinitrc` to have it apply on every session automatically.
