# Arch Linux on ThinkPad T14s Gen 1 — Complete Installation Record
### A Detailed Account of Every Step, Problem, and Fix

**Machine:** Lenovo ThinkPad T14s Gen 1
**CPU:** Intel Core i7 10th Generation
**RAM:** 16GB (with a 32MB failing region — see critical section below)
**Storage:** NVMe SSD
**User:** nikhil
**Hostname:** t14s
**Philosophy:** Suckless Unix — TTY login → startx → bspwm, no display manager, no DE
**Purpose:** Software development (web, Python, HPC, quant finance)

---

## Table of Contents

1. [Critical Hardware Issue — Faulty RAM](#1-critical-hardware-issue--faulty-ram)
2. [Pre-Installation Decisions](#2-pre-installation-decisions)
3. [Live ISO Boot](#3-live-iso-boot)
4. [Disk Partitioning](#4-disk-partitioning)
5. [Running archinstall](#5-running-archinstall)
6. [Chroot Configuration](#6-chroot-configuration)
7. [First Boot Verification](#7-first-boot-verification)
8. [Post-Install System Setup](#8-post-install-system-setup)
9. [Xorg and Display](#9-xorg-and-display)
10. [Window Manager Stack](#10-window-manager-stack)
11. [The startx Failure and xinitrc Fix](#11-the-startx-failure-and-xinitrc-fix)
12. [Dotfiles Setup and Symlinks](#12-dotfiles-setup-and-symlinks)
13. [The Root vs User Problem](#13-the-root-vs-user-problem)
14. [Node.js via NVM](#14-nodejs-via-nvm)
15. [Python Toolchain](#15-python-toolchain)
16. [Rust via Rustup](#16-rust-via-rustup)
17. [Claude Code](#17-claude-code)
18. [TLP and Power Management](#18-tlp-and-power-management)
19. [The UKI Discovery](#19-the-uki-discovery)
20. [Pacman Hooks for Update Safety](#20-pacman-hooks-for-update-safety)
21. [GRUB Boot Screen](#21-grub-boot-screen)
22. [Verification Script](#22-verification-script)
23. [Lessons Learned](#23-lessons-learned)
24. [Quick Reference Commands](#24-quick-reference-commands)

---

## 1. Critical Hardware Issue — Faulty RAM

### The Problem

The ThinkPad T14s has a failing RAM region of **32MB** starting at approximately **45554 MB** in the physical address space. This is a hardware defect where reads and writes to that memory region return incorrect values. If the Linux kernel allocates any process, file cache, swap page, or kernel buffer to this physical region, it causes silent data corruption or system crashes.

### Why This Shapes Every Installation Decision

This single hardware fact drove a cascade of decisions throughout the entire installation:

**Filesystem choice — ext4, not btrfs:**
btrfs is a copy-on-write filesystem that continuously checksums data blocks and stores metadata trees across the disk. The checksumming process reads from RAM to verify data integrity. If btrfs metadata ever gets loaded into the bad RAM region, the checksums will mismatch, btrfs will detect what it thinks is corruption, and it will either refuse to mount or start making destructive "repairs." btrfs can also panic the kernel when it encounters checksum failures. ext4 is a much simpler filesystem — it reads and writes data without the continuous integrity checking layer. Combined with the kernel's `memmap=` exclusion, ext4 is the safe choice.

**Swap — swapfile, not partition:**
A swap partition is carved at disk partitioning time as a fixed block of sectors on the NVMe drive. The kernel can swap any physical memory page to any sector in that partition. Crucially, a swap partition is set up before the kernel has a chance to read its `memmap=` parameter and exclude the bad RAM region. The risk is that during a swap-in operation, a page gets placed into the bad physical address range. A swapfile created inside the ext4 root filesystem, after the first boot with `memmap=` confirmed active, is safe because the kernel's memory allocator with the bad region excluded handles all page placement, and the swapfile's disk blocks are managed by ext4 rather than raw block addressing.

**Kernel — LTS, not mainline:**
The LTS (Long Term Support) kernel receives security patches but not new features. This matters because the `memmap=` kernel parameter and the way it interacts with the IOMMU and memory allocator is stable and well-understood in the LTS branch. Mainline kernels occasionally refactor memory management in ways that can interact unexpectedly with custom memory maps.

### The memmap Parameter

The kernel parameter that excludes the bad region is:

```
memmap=36M$0x11C800000
```

Note: the actual value used in this installation is `36M$0x11C800000` (slightly larger than 32MB, covering the full bad region). This was determined during installation by checking `dmesg` output.

The `$` character separates the size from the physical address. The backslash in `/etc/kernel/cmdline` (`36M\$0x11C800000`) is intentional — it escapes the `$` so the shell doesn't interpret it as a variable when the file is read by scripts.

**To verify it is active at any time:**
```bash
cat /proc/cmdline | grep memmap
```

---

## 2. Pre-Installation Decisions

### What archinstall Was Configured to Do

The `archinstall` script was used for speed. The key selections made:

- **Bootloader:** grub
- **Swap:** False (explicitly disabled — see RAM section above)
- **Profile:** minimal (no DE, no display manager — pure TTY to startx)
- **Audio:** pipewire
- **Kernels:** linux-lts (NOT linux mainline)
- **Network:** NetworkManager
- **Filesystem:** ext4
- **Timezone:** Asia/Kolkata

### What Was Done Manually Before archinstall

The disk was partitioned manually with `gdisk` before running `archinstall`, for two reasons: full control over partition types and sizes, and certainty that no swap partition would be silently created.

**GPT partition table** (required for UEFI):
- Partition 1: 512MB, type `ef00` (EFI System Partition), formatted FAT32
- Partition 2: Remaining space, type `8300` (Linux filesystem), formatted ext4

```bash
gdisk /dev/nvme0n1
# o → new GPT
# n → partition 1 → +512M → ef00
# n → partition 2 → rest of disk → 8300
# w → write

mkfs.fat -F32 /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p2

mount /dev/nvme0n1p2 /mnt
mkdir -p /mnt/boot/efi
mount /dev/nvme0n1p1 /mnt/boot/efi
```

---

## 3. Live ISO Boot

### Connecting to Internet

The ThinkPad T14s has an Intel ethernet controller that works out of the box in the live ISO. Ethernet was used for the installation. WiFi was not needed during install.

```bash
ip link
ping -c 3 archlinux.org
timedatectl set-ntp true
```

### Finding the Disk

```bash
lsblk
# disk showed as /dev/nvme0n1
```

---

## 4. Disk Partitioning

### Why Only Two Partitions

No separate `/home`, no separate `/boot`, no `/var`. A single root partition holds everything. Reasons:

1. Separate `/home` partitions cause problems when `/` fills up from package cache, logs, or docker images while home is empty, and vice versa
2. Separate `/var` partitions have caused real Arch installs to break when pacman fills `/var/cache` during a large update and `/var` runs out of space mid-transaction
3. Two partitions is the minimum needed for UEFI boot — EFI for the bootloader, root for everything else
4. With LVM or btrfs subvolumes you can split things up later if needed; you cannot easily merge a separate partition back

### No Swap Partition

As explained in section 1. The swap was created as a swapfile after the first boot.

---

## 5. Running archinstall

### The Chroot Question at the End

After `archinstall` completes installation, it asks:

```
Do you want to chroot into the newly installed system?
```

**Answer: YES.** This is essential — you need to be inside the new system to configure GRUB with the `memmap=` parameter before the first reboot. If you say no, you have to manually `arch-chroot /mnt` yourself anyway.

### How to Know You Are in the Chroot

After saying yes, the prompt changes to:

```
[root@archiso /]#
```

The hostname stays `archiso` even though you are inside the new system — this is normal. The hostname only shows correctly after a full reboot. To confirm you are genuinely inside the new system and not the live ISO:

```bash
df -h /                    # shows your nvme partition, not tmpfs
cat /etc/fstab             # shows your nvme partition UUIDs
pacman -Q | wc -l          # shows 100+ packages from the install
pacman -Q linux-lts        # shows your LTS kernel
```

When you type `exit`, you drop back to the live ISO. Do not exit until GRUB is fully configured.

---

## 6. Chroot Configuration

### GRUB and memmap

This was the most critical step. The memmap parameter must be in GRUB's cmdline before the first boot.

```bash
nano /etc/default/grub
```

The line to modify:
```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet memmap=36M$0x11C800000"
```

**Important — the `$` character problem:**
In shell, `$` is used for variable expansion. If you write this line using double-quoted `echo` or heredoc with double quotes, the shell eats the `$0x11C800000` and it becomes empty. Always use single quotes when writing this line via sed or echo:

```bash
# CORRECT — single quotes protect the $
sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet memmap=36M$0x11C800000"/' /etc/default/grub

# verify $ is literally there
grep memmap /etc/default/grub
```

Install GRUB and generate config:

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ARCH
grub-mkconfig -o /boot/grub/grub.cfg
grep memmap /boot/grub/grub.cfg    # must appear in the output
```

### nano Not Found in Chroot

`archinstall` with the minimal profile does not install `nano`. First time editing files in chroot, `nano` was not available. Options:

1. Install it immediately: `pacman -S nano`
2. Use `vi` which is always present
3. Use `sed` for surgical one-line edits (safest for the GRUB cmdline)

The `sed` approach was used for the critical GRUB line to avoid any editor-related mistakes.

### Enabling NetworkManager

```bash
systemctl enable NetworkManager
```

This must be done in chroot. If forgotten, you boot into a system with no internet and have to either boot the live ISO again or manually start NetworkManager from TTY.

### Enabling fstrim

```bash
systemctl enable fstrim.timer
```

For NVMe drive health. The timer runs weekly and sends TRIM commands to the SSD, reclaiming deleted blocks.

---

## 7. First Boot Verification

### Confirming memmap is Active

Immediately after first boot, before doing anything else:

```bash
cat /proc/cmdline
```

Must show `memmap=36M$0x11C800000` in the output. If it is missing, the bad RAM region is not excluded and the system is running in an unsafe state.

Also check dmesg:

```bash
dmesg | grep -i "user-defined\|memmap\|reserved"
```

The kernel prints a line confirming the reserved region when `memmap=` is active.

### Checking Usable RAM

```bash
free -h
```

Will show approximately 15.97GB rather than 16GB — the 36MB excluded region is visible as slightly reduced total RAM. This is correct and expected.

---

## 8. Post-Install System Setup

### The Root vs User Problem (CRITICAL — Read This)

**This was one of the biggest mistakes made during this installation.**

The initial setup after booting was done logged in as `root`. Everything installed at this stage — nvm, node, npm globals, pipx, black, ruff, ipython, uv, rustup, Claude Code — was installed into `/root/` (root's home directory). When logging in as the actual user `nikhil`, none of these tools were available because they live in `/root/.nvm`, `/root/.local/bin/`, `/root/.cargo/` etc., which are not in nikhil's PATH.

**The rule:** `pacman -S` installs system-wide to `/usr/bin/` and is available to all users. Everything else (nvm, rustup, pipx tools, npm globals) installs to the current user's home directory and is ONLY available to that user.

**What had to be cleaned up from root:**

```bash
su -    # switch to root to clean up

rm -rf /root/.nvm
rm -rf /root/.npm
rm -rf /root/.rustup
rm -rf /root/.cargo
rm -rf /root/.local/share/nvim
rm -rf /root/.local/share/pipx
rm -rf /root/.local/bin/          # contained: black, blackd, ipython, ipython3, ruff, uv, uvx
rm -rf /root/.config/claude
rm /root/.xinitrc

# clean nvm lines from root's .bashrc
nano /root/.bashrc   # remove NVM_DIR export lines

exit    # back to nikhil
```

**The correct workflow going forward:** Always work as `nikhil`. Use `sudo` only when a command genuinely needs root (pacman, systemctl enable, editing /etc files). Never `su -` to root and install user tools.

### Swapfile Creation (Done After Confirming memmap Active)

```bash
# as root via sudo
sudo dd if=/dev/zero of=/swapfile bs=1M count=4096 status=progress
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# make permanent
echo '/swapfile none swap defaults 0 0' | sudo tee -a /etc/fstab

# tune swappiness
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system
```

### System Update

```bash
sudo pacman -Syu
```

---

## 9. Xorg and Display

### Packages Installed

```bash
sudo pacman -S xorg-server xorg-xinit xorg-xrandr xorg-xsetroot xorg-xprop \
               xorg-xwininfo xf86-input-libinput \
               mesa intel-media-driver libva-intel-driver vulkan-intel
```

### ThinkPad T14s Input Configuration

Created `/etc/X11/xorg.conf.d/40-libinput.conf` for trackpad natural scrolling and TrackPoint configuration. The TrackPoint middle-button scroll (hold middle button, move nub to scroll) is enabled via:

```bash
xinput set-prop "TPPS/2 Elan TrackPoint" "libinput Scroll Method Enabled" 0 0 1
```

This is placed in `bspwmrc` with `2>/dev/null` so it silently skips if the device name differs slightly on a different hardware revision.

---

## 10. Window Manager Stack

### Philosophy

Pure suckless approach: no display manager (no GDM, SDDM, lightdm), no desktop environment. Login at TTY, run `startx`, enter bspwm. Nothing running that isn't explicitly started.

### Packages

```bash
sudo pacman -S bspwm sxhkd polybar picom dunst libnotify xclip \
               thunar feh alacritty rofi brightnessctl playerctl \
               xss-lock i3lock scrot
```

### bspwmrc — Key Adaptations for ThinkPad

The original dotfiles from the MacBook Pro (a1278, 2015) had:

```bash
bspc monitor -d 1 2 3 4 5 6 7 8 9 10
```

This was changed to explicitly name the ThinkPad's internal display:

```bash
bspc monitor eDP-1 -d 1 2 3 4 5 6 7 8 9 10
```

Without naming the monitor explicitly, plugging in an external display causes bspwm to split the 10 desktops randomly between the two monitors in unpredictable ways.

Other additions to bspwmrc for ThinkPad:

```bash
# Fix blank Java GUI windows (affects some HPC/dev tools)
export _JAVA_AWT_WM_NONREPARENTING=1

# Keyboard repeat rate (default Linux rate is too slow for coding)
xset r rate 300 50 &

# TrackPoint middle-button scroll
xinput set-prop "TPPS/2 Elan TrackPoint" "libinput Scroll Method Enabled" 0 0 1 2>/dev/null &
```

### sxhkdrc — Key Adaptations for ThinkPad

The MacBook dotfiles used `XF86LaunchA` and `XF86LaunchB` for Flameshot screenshot bindings. **These keys do not exist on any ThinkPad keyboard.** They are MacBook-specific eject/launch keys. On the ThinkPad, these were replaced with:

```bash
Print
    flameshot full

super + Print
    flameshot gui
```

ThinkPad-specific function key bindings added:

```bash
XF86MonBrightnessUp     → brightnessctl set +5%
XF86MonBrightnessDown   → brightnessctl set 5%-
XF86AudioRaiseVolume    → wpctl set-volume -l 1.0 @DEFAULT_AUDIO_SINK@ 5%+
XF86AudioLowerVolume    → wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%-
XF86AudioMute           → wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle
XF86AudioMicMute        → wpctl set-mute @DEFAULT_SOURCE@ toggle
XF86AudioPlay           → playerctl play-pause
XF86AudioPrev           → playerctl previous
XF86AudioNext           → playerctl next
```

Note: `wpctl` (wireplumber control) was used for volume rather than `pactl` because the original dotfiles used it and it is the native pipewire tool.

---

## 11. The startx Failure and xinitrc Fix

### What Happened

Running `startx` for the first time produced this error:

```
/etc/X11/xinit/xinitrc: line 53: xclock: command not found
/etc/X11/xinit/xinitrc: line 54: xterm: command not found
xinit: connection to X server lost
```

X started, tried to launch things, crashed immediately, and dropped back to TTY.

### Root Cause

There was no `~/.xinitrc` in the user's home directory. When `startx` finds no `~/.xinitrc`, it falls back to the system default at `/etc/X11/xinit/xinitrc`. The system default tries to launch `xclock` and `xterm` — both of which are not installed in our minimal setup. `xclock` not found → X has nothing to manage → X exits → back to TTY.

### The Fix

Create `~/.xinitrc`:

```bash
nano ~/.xinitrc
```

```bash
#!/bin/sh
[ -f ~/.Xresources ] && xrdb -merge ~/.Xresources
picom --backend glx --vsync --daemon &
dunst &
exec bspwm
```

```bash
chmod +x ~/.xinitrc
```

`exec bspwm` is the critical line — `exec` replaces the shell process with bspwm, which means when bspwm exits (logout), X also exits cleanly. Without `exec`, bspwm runs as a child of the shell, and when it exits the shell keeps running and X stays open indefinitely.

### Auto-startx on TTY1 Login

To avoid typing `startx` every login, added to `~/.bash_profile`:

```bash
if [[ -z "$DISPLAY" ]] && [[ "$XDG_VTNR" -eq 1 ]]; then
    exec startx
fi
```

This only triggers on TTY1 (the first virtual terminal) and only when no display is already running (`$DISPLAY` is empty). TTY2-6 remain as pure TTYs for debugging.

### Getting Into X After startx Works

Once in bspwm with no windows open, the screen is blank. Nothing happens until you use keybindings. The critical ones to know:

```
super + Return    → open alacritty terminal
super + b         → open firefox
super + d         → open rofi app launcher
```

`super` = Windows key on ThinkPad.

If keybindings do nothing: sxhkd may not have started. Switch to TTY2 with `Ctrl+Alt+F2`, check `pgrep -x sxhkd`, and if nothing shows, run `sxhkd &` then switch back with `Ctrl+Alt+F1`.

---

## 12. Dotfiles Setup and Symlinks

### Repository Structure

The dotfiles repository cloned from GitHub (`https://github.com/phy-nikhilroy/dotfiles-arch-bspwm`) contains:

```
dotfiles/
├── bspwm/
│   └── bspwmrc
├── sxhkd/
│   └── sxhkdrc
├── polybar/
│   ├── config.ini
│   └── launch.sh
└── nvim/
    ├── init.lua
    └── lua/
        ├── config/
        │   ├── autocmds.lua
        │   ├── globals.lua
        │   ├── keymaps.lua
        │   ├── lazy.lua
        │   ├── options.lua
        │   └── terminal.lua
        └── plugins/
            ├── cmp.lua
            ├── conform.lua
            ├── fzf.lua
            ├── gemini.lua
            ├── git.lua
            ├── lsp.lua
            ├── lua_line.lua
            ├── mini.lua
            ├── snacks.lua
            ├── tree.lua
            └── treesitter.lua
```

### Creating Symlinks

The `.config` directory did not exist and had to be created:

```bash
mkdir ~/.config
```

Then all symlinks:

```bash
mkdir -p ~/.config/bspwm
mkdir -p ~/.config/sxhkd
mkdir -p ~/.config/polybar
mkdir -p ~/.config/nvim

ln -sf ~/dotfiles/bspwm/bspwmrc         ~/.config/bspwm/bspwmrc
ln -sf ~/dotfiles/sxhkd/sxhkdrc         ~/.config/sxhkd/sxhkdrc
ln -sf ~/dotfiles/polybar/config.ini     ~/.config/polybar/config.ini
ln -sf ~/dotfiles/polybar/launch.sh      ~/.config/polybar/launch.sh
ln -sf ~/dotfiles/nvim                   ~/.config/nvim

chmod +x ~/.config/bspwm/bspwmrc
chmod +x ~/.config/polybar/launch.sh
```

### Symlink Safety Rules

- Use `ln -sf` (force) to overwrite an existing symlink without needing to delete first
- To delete a symlink: `rm ~/.config/bspwm/bspwmrc` — no flags needed, deletes only the link
- **Never** `rm -rf ~/.config/nvim` — the `-rf` flag treats it as a real directory and deletes the contents of your dotfiles repo
- Always `rm` a symlink without trailing slash and without `-r`
- Verify health with `ls -la ~/.config/bspwm/` — look for the `->` arrow pointing to an existing file

---

## 13. The Root vs User Problem

This is documented in section 8 but deserves its own section because it caused significant rework.

### What Went Wrong

All post-install setup was done as `root` because the initial login after archinstall dropped into the root shell. The following were installed as root:

- nvm and Node.js (in `/root/.nvm/`)
- Claude Code npm global (in `/root/.nvm/.../bin/`)
- pipx (in `/root/.local/bin/`)
- black, ruff, ipython, uv via pipx (in `/root/.local/bin/`)
- rustup and rust toolchain (in `/root/.rustup/` and `/root/.cargo/`)

When switching to user `nikhil`, none of these were available. `which node` returned nothing. `which claude` returned nothing.

### What Was Cleaned

```bash
su -
rm -rf /root/.nvm
rm -rf /root/.npm
rm -rf /root/.rustup /root/.cargo
rm -rf /root/.local/share/nvim
rm -rf /root/.local/share/pipx
rm -rf /root/.local/share/uv
rm -rf /root/.local/bin/
rm -rf /root/.config/claude
exit
```

### Why pacman Packages Are Fine

`pacman -S` installs to `/usr/bin/`, `/usr/lib/` etc. which are on every user's PATH. So `git`, `gcc`, `python`, `nvim`, `firefox`, `bspwm` etc. installed as root via pacman are perfectly accessible to nikhil. Only home-directory-scoped installers (nvm, rustup, pipx) need to be run as the target user.

---

## 14. Node.js via NVM

### Why NVM, Not pacman

The pacman `nodejs` package is a single fixed version. NVM allows multiple Node versions to coexist and switching between them per-project. For web development and Claude Code, NVM is the correct tool.

### Installation (as nikhil)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install --lts
nvm use --lts
nvm alias default lts/*
```

### What NVM Adds to ~/.bashrc

The installer automatically appends:

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

### Critical Check

If `nodejs` is installed via pacman alongside nvm, `which node` may point to `/usr/bin/node` (pacman version) instead of `~/.nvm/versions/node/.../bin/node`. Check:

```bash
which node    # must show /home/nikhil/.nvm/versions/...
```

If it shows `/usr/bin/node`, remove the pacman version:

```bash
sudo pacman -R nodejs
```

---

## 15. Python Toolchain

### System Python

Python itself is installed via pacman (system-wide is fine):

```bash
sudo pacman -S python python-pip python-pipx python-virtualenv
```

### User Tools via pipx (as nikhil)

pipx installs Python applications in isolated virtual environments in `~/.local/share/pipx/` and puts their executables in `~/.local/bin/`. This keeps tools like `black` and `ruff` from polluting the system Python or each other.

```bash
pipx install black
pipx install ruff
pipx install ipython
pipx install uv
```

Ensure `~/.local/bin` is in PATH:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Quant/HPC Python Environment

Project-specific packages go in a virtualenv, never in system Python:

```bash
python -m venv ~/envs/quant
source ~/envs/quant/bin/activate
pip install numpy scipy pandas matplotlib jupyterlab scikit-learn numba
deactivate
```

---

## 16. Rust via Rustup

### The pacman Problem

During initial setup, `rust` was installed via `pacman -S rust`. This installs Rust to `/usr/bin/rustc` as a system package. The problem with this approach:

1. It's a single fixed version — you can't switch Rust channels (stable/beta/nightly)
2. `rust-analyzer`, `clippy`, `rustfmt` components aren't easily managed
3. It's separate from rustup's toolchain management, causing confusion about which `rustc` is active

The verify script detected this with:

```
! rustc found but not via rustup: /usr/bin/rustc
```

### The Fix

```bash
# remove pacman rust
sudo pacman -Rns rust

# install rustup as nikhil
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# press Enter to accept defaults (option 1)

source ~/.bashrc
# or: source "$HOME/.cargo/env"

# install components
rustup component add rust-analyzer clippy rustfmt

# verify
which rustc    # must show /home/nikhil/.cargo/bin/rustc
```

---

## 17. Claude Code

### Installation

After nvm and Node are correctly set up as nikhil:

```bash
npm install -g @anthropic/claude-code
```

### Authentication — Pure TTY Problem

Claude Code's default authentication opens a browser for OAuth. On a fresh install with no X running, there is no browser. Two solutions:

**Option A — API Key (works on TTY):**
Get the API key from `console.anthropic.com` on another device, then:

```bash
echo 'export ANTHROPIC_API_KEY="sk-ant-XXXXXXXXX"' >> ~/.bashrc
source ~/.bashrc
claude
```

**Option B — startx first:**
Get X running, open Firefox via `super + b`, authenticate via browser OAuth, then use Claude Code in the terminal.

### Running Claude Code Before X

Claude Code runs perfectly in a bare TTY — it is a terminal application. The only requirements are Node.js (via nvm) and internet (via NetworkManager). This makes it possible to use Claude Code to help with the remaining setup tasks directly from TTY before the window manager is configured.

---

## 18. TLP and Power Management

### The Conflict

After installing and enabling TLP, running `sudo tlp start` produced:

```
Warning: PLATFORM_PROFILE_ON_AC/BAT is not set because power-profiles-daemon is running.
Warning: CPU_ENERGY_PERF_POLICY_ON_AC/BAT/SAV is not set because power-profiles-daemon is running.
```

`power-profiles-daemon` is a systemd service that also manages CPU power profiles. It is installed by some desktop environment dependencies or firmware packages. TLP and power-profiles-daemon both try to set the CPU energy/performance policy and they conflict — TLP's settings get overridden or silently ignored.

### The Fix

```bash
sudo systemctl stop power-profiles-daemon
sudo systemctl disable power-profiles-daemon
sudo systemctl mask power-profiles-daemon
sudo reboot
```

`mask` creates a symlink from the service file to `/dev/null`, making it impossible to start by any means — manual, automatic, or as a dependency — until explicitly unmasked. This is stronger than `disable` and ensures no future package installation accidentally re-enables it.

### Why TLP Over power-profiles-daemon

TLP has extensive ThinkPad-specific features unavailable in power-profiles-daemon:

- Battery charge thresholds (start charging at 40%, stop at 80%) — dramatically extends battery lifespan
- Per-device power management (WiFi, bluetooth, USB autosuspend)
- ThinkPad-specific ACPI calls via `acpi_call`
- Fine-grained CPU frequency and turbo boost control

### Battery Charge Thresholds

In `/etc/tlp.conf`:

```
START_CHARGE_THRESH_BAT0=40
STOP_CHARGE_THRESH_BAT0=80
```

This means the battery only charges when below 40% and stops at 80%. Lithium batteries last significantly longer when not kept at 100%. For a laptop used primarily on AC power this extends battery life from ~3 years to potentially 5-7 years.

### The tlp-sleep.service Error

When trying to enable `tlp-sleep.service`:

```
Failed to enable unit: Unit tlp-sleep.service does not exist
```

This service was removed in modern TLP versions. Sleep/wake power management is now handled internally by `tlp.service` itself. Only `tlp.service` needs to be enabled.

---

## 19. The UKI Discovery

### What Was Expected vs What Was Found

The installation guide assumed a standard GRUB setup where `memmap=` is in `/etc/default/grub` and `grub-mkconfig` propagates it to `/boot/grub/grub.cfg`. During troubleshooting, it was discovered the actual setup is different.

### What archinstall Actually Created

Running `find /boot/efi -type f | sort` revealed:

```
/boot/efi/BOOT/BOOTX64.EFI
/boot/efi/EFI/ARCH/grubx64.efi
/boot/efi/Linux/arch-linux-lts.efi
```

The file `/boot/efi/Linux/arch-linux-lts.efi` is a **Unified Kernel Image (UKI)** — a single EFI binary that contains the kernel, initramfs, and kernel command line all baked together into one file. GRUB detects this UKI and chainloads it instead of directly loading a kernel.

### Implication for memmap

In a UKI setup, the kernel command line is **embedded into the `.efi` file at build time**. `/etc/default/grub` has no effect on the kernel cmdline — GRUB just hands off execution to the UKI which already has its cmdline baked in.

The source of truth for the kernel cmdline is:

```
/etc/kernel/cmdline
```

Contents:
```
root=PARTUUID=9cc0d00e-2fd8-403e-a582-8d5d7a534e80 rw rootfstype=ext4 memmap=36M\$0x11C800000
```

The `memmap=` was already correctly present here, which is why it appeared in `/proc/cmdline` even though `/etc/default/grub` wasn't the source.

### GRUB Still Controls Menu Display

Even in a UKI setup, GRUB's `GRUB_TIMEOUT` and `GRUB_TIMEOUT_STYLE` settings in `/etc/default/grub` still work — they control GRUB's own menu display before it chainloads the UKI. So hiding the GRUB menu with `GRUB_TIMEOUT=0` and `GRUB_TIMEOUT_STYLE=hidden` still works as expected.

---

## 20. Pacman Hooks for Update Safety

Two hooks ensure the system never breaks on updates.

### grub-update.hook (present from initial setup)

```ini
# /etc/pacman.d/hooks/grub-update.hook
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = linux-lts
Target = linux-lts-headers
Target = grub

[Action]
Description = Regenerating GRUB config after kernel/grub update...
When = PostTransaction
Exec = /usr/bin/grub-mkconfig -o /boot/grub/grub.cfg
```

Regenerates the GRUB menu after kernel or GRUB updates.

### uki-update.hook (added after UKI discovery)

```ini
# /etc/pacman.d/hooks/uki-update.hook
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = linux-lts
Target = linux-lts-headers
Target = mkinitcpio

[Action]
Description = Rebuilding UKI after kernel update, memmap will be preserved...
When = PostTransaction
Exec = /usr/bin/mkinitcpio -p linux-lts
```

Rebuilds the UKI (including re-reading `/etc/kernel/cmdline` with the `memmap=` parameter) after every kernel update. Without this hook, a kernel update would install a new kernel but the UKI would remain stale until manually rebuilt, potentially causing a boot failure.

### Why Both Are Needed

- `grub-update.hook` → keeps GRUB's menu entries current
- `uki-update.hook` → keeps the UKI (and its embedded cmdline with `memmap=`) current

Together they mean `pacman -Syu` is fully automatic with no manual intervention needed after kernel updates.

---

## 21. GRUB Boot Screen

### The Lenovo Interrupt Screen

The "press Enter to interrupt" screen during boot is a Lenovo BIOS feature, not a GRUB or Linux feature. It is disabled in BIOS:

```
F1 → BIOS → Startup → Quick Boot = Enabled, Splash Screen = Disabled → F10 save
```

### Hiding the GRUB Menu

```bash
sudo nano /etc/default/grub
```

```
GRUB_TIMEOUT=0
GRUB_TIMEOUT_STYLE=hidden
```

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

With `TIMEOUT=0` and `hidden`, boot goes directly from POST to kernel without any visible menu.

**Emergency fallback:** Hold **Shift** during boot to force the GRUB menu to appear even with these settings. This is the recovery path if a kernel update breaks something.

---

## 22. Verification Script

A bash script (`verify-setup.sh`) was created to check all 15 aspects of the setup:

1. memmap active in `/proc/cmdline`
2. `/etc/kernel/cmdline` source of truth
3. Both pacman hooks present
4. GRUB EFI and UKI files present
5. LTS kernel running
6. ext4 filesystem (not btrfs)
7. Swapfile active (not partition)
8. Swapfile in fstab
9. `~/.xinitrc` exists and launches bspwm
10. All WM stack packages installed
11. All dotfiles symlinks healthy (not broken)
12. Node from NVM (not pacman)
13. Rust from rustup (not pacman)
14. Claude Code installed and API key set
15. TLP running, power-profiles-daemon masked

Usage:
```bash
chmod +x verify-setup.sh
bash verify-setup.sh
```

---

## 23. Lessons Learned

### 1. Always log in as your regular user immediately after install
Do not stay in root. Do `su - nikhil` or log out and log back in as nikhil before installing any user-scoped tools. This prevents the entire nvm/rustup/pipx cleanup problem.

### 2. Verify memmap before anything else on first boot
`cat /proc/cmdline` is the first command to run after first boot. If `memmap=` is not there, reboot the live ISO and fix GRUB before continuing. Everything else can be fixed later. The bad RAM cannot be fixed — it must be excluded.

### 3. Check the actual bootloader setup before assuming
This installation used a UKI despite archinstall being configured for standard GRUB. The `find /boot/efi -type f` command should be run early to understand exactly what was created, rather than assuming the expected layout.

### 4. The $ character must be escaped or single-quoted everywhere
In shell, `$` in double-quoted strings gets interpreted as variable expansion. `memmap=36M$0x11C800000` in double quotes becomes `memmap=36M` with everything after `$` silently eaten. Always use single quotes or backslash-escape the `$` when writing this parameter.

### 5. MacBook dotfiles need specific ThinkPad changes
- `XF86LaunchA`/`XF86LaunchB` → don't exist on ThinkPad, replace with `Print`/`super + Print`
- `bspc monitor -d` → must become `bspc monitor eDP-1 -d` with explicit name
- macOS `cmd`/`option` keys → Linux `super`/`alt`
- Polybar network interface names → check with `ip link show` (likely `enp2s0`, not `en0`)
- Polybar battery → `BAT0` (ThinkPad), not macOS battery source

### 6. nano not in minimal chroot
Install it immediately or use `sed` for critical one-line edits. The sed approach is actually more reliable for the GRUB cmdline edit because it's surgical and immediately verifiable.

### 7. tlp-sleep.service no longer exists
Don't try to enable it. Modern TLP handles sleep internally. Only `tlp.service` needed.

### 8. power-profiles-daemon must be masked, not just disabled
`disable` can be undone by dependency resolution. `mask` is permanent until explicitly reversed.

---

## 24. Quick Reference Commands

### System Health Checks

```bash
# Is the bad RAM region excluded?
cat /proc/cmdline | grep memmap

# Is swap a file (not partition)?
swapon --show

# What kernel am I running?
uname -r

# Is TLP running without conflicts?
sudo tlp-stat -s

# Are both pacman hooks present?
ls /etc/pacman.d/hooks/

# Is node from NVM (not pacman)?
which node    # must show /home/nikhil/.nvm/...
```

### Emergency Recovery

```bash
# Force GRUB menu to appear (hold during boot)
[Hold Shift key during POST]

# Boot live ISO and get back into installed system
mount /dev/nvme0n1p2 /mnt
mount /dev/nvme0n1p1 /mnt/boot/efi
arch-chroot /mnt

# Rebuild UKI manually (if kernel update broke boot)
sudo mkinitcpio -p linux-lts

# Regenerate GRUB config
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### Daily Driver Commands

```bash
# Update system (both hooks run automatically)
sudo pacman -Syu

# Start X
startx

# In bspwm
super + Return    # terminal
super + b         # firefox
super + d         # rofi launcher
super + q         # close window
super + {1-9}     # switch desktop
super + f         # fullscreen
super + m         # monocle layout

# Switch Node versions
nvm install 20
nvm use 20
nvm alias default 20

# Activate quant Python environment
source ~/envs/quant/bin/activate
```

### File Locations Reference

```
/etc/kernel/cmdline                    ← memmap= source of truth (UKI)
/etc/default/grub                      ← GRUB menu settings (timeout=0)
/etc/pacman.d/hooks/grub-update.hook   ← auto grub-mkconfig on kernel update
/etc/pacman.d/hooks/uki-update.hook    ← auto UKI rebuild on kernel update
/etc/X11/xorg.conf.d/40-libinput.conf  ← trackpad/TrackPoint config
/etc/tlp.conf                          ← battery thresholds (40/80)
/etc/sysctl.d/99-swappiness.conf       ← vm.swappiness=10
/swapfile                              ← swap (not a partition)
~/.xinitrc                             ← startx entry point → bspwm
~/.bash_profile                        ← auto-startx on TTY1
~/.bashrc                              ← NVM, EDITOR, PATH exports
~/.config/bspwm/bspwmrc               ← symlink → ~/dotfiles/bspwm/bspwmrc
~/.config/sxhkd/sxhkdrc               ← symlink → ~/dotfiles/sxhkd/sxhkdrc
~/.config/polybar/                     ← symlinks → ~/dotfiles/polybar/
~/.config/nvim/                        ← symlink → ~/dotfiles/nvim/
~/.nvm/                                ← Node Version Manager
~/.cargo/                              ← Rust toolchain (rustup)
~/.local/bin/                          ← pipx tools (black, ruff, ipython, uv)
~/envs/quant/                          ← Python quant/HPC virtualenv
~/dotfiles/                            ← dotfiles git repo
```

---

*Installation completed: June 2026*
*Machine: ThinkPad T14s Gen 1 · Intel i7 10th Gen · 16GB RAM · Arch Linux LTS*
*User: nikhil · Hostname: t14s*
