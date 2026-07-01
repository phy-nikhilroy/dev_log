# Reconstructing My Arch Install Timeline From pacman.log

**Date generated:** 2026-07-01
**System:** ThinkPad T14s — Arch Linux
**Source:** `/var/log/pacman.log`, cross-checked against [`arch-bspwm-thinkpad-t14s.md`](arch-bspwm-thinkpad-t14s.md)

---

## Background

When I wrote my original install log, I reconstructed the timeline from memory and from directory `mtime` timestamps. I got some of it wrong — in particular, I was convinced I'd installed Neovim on a separate day, and I had a whole story in my head about installing the standard `rust` package before switching to `rustup`.

It turns out none of that guesswork was necessary. `/var/log/pacman.log` has **never rotated** on this machine, so it holds a complete, second-by-second record of every package transaction since the very first `pacstrap` call on install day — 817 real events: 659 installs, 158 upgrades, 0 removals.

This document is built entirely from that log, cross-checked against my original install log for the *why* behind each step. Where the two disagree, the log wins — it's a mechanical record; my original log is a person's account written from memory. Both kinds of disagreement turned up, and I've flagged them explicitly below.

**A caution for future me:** don't trust `mtime` alone. A routine `pacman -Syu` bumps a package's files and resets its directory's mtime, which makes an old package look like it was freshly installed on upgrade day. That's exactly the trap that made me misdate Neovim originally — see Day 3 below.

---

## The One-Line Version

657 of 659 packages arrived in the **first two days** (2026-06-10 and 2026-06-11). Only **2 packages** have been newly installed since: `hugo` (2026-06-21) and `libhwasan` (2026-06-29, a side effect of a `gcc` version bump splitting a sanitizer runtime into its own package). Every other event on 2026-06-12, 2026-06-21, and 2026-06-29 was a version **upgrade** of something already installed — mostly two LTS kernel point-releases and a couple of large `pacman -Syu` runs.

| Date | New installs | Upgrades | What it was |
|---|---|---|---|
| 2026-06-10 | 652 | 3 | Install day: `archinstall` + the entire manual setup session |
| 2026-06-11 | 5 | 0 | Added `kitty` terminal + `terminus-font` |
| 2026-06-12 | 0 | 7 | Routine `pacman -Syu` (neovim, bash, github-cli, etc. bumped) |
| 2026-06-21 | 1 | 63 | `pacman -Syu` incl. an LTS kernel point-release, + installed `hugo` |
| 2026-06-29 | 1 | 85 | `pacman -Syu` incl. another LTS kernel point-release, + `libhwasan` appeared |

---

## Day 1 — 2026-06-10 — The Full Build, Minute by Minute

### 13:17:43–13:20:35 IST — archinstall's own `pacstrap` calls

Archinstall doesn't run one giant install command — it fires a separate `pacman -Sy --needed -r /mnt ...` per configuration screen. In order, 9 transactions covering 652 packages:

| Time (IST) | Command target | New packages | Notes |
|---|---|---|---|
| 13:17:43 | `base sudo linux-firmware linux-lts intel-ucode` | 154 | The core system + LTS kernel choice |
| 13:19:49 | `grub` | 1 | Bootloader choice |
| 13:19:51 | `efibootmgr` | 2 (+`efivar`) | UEFI boot management |
| 13:19:53 | `networkmanager wpa_supplicant` | 20 | Network choice |
| 13:19:57 | `bluez bluez-utils` | 5 | **Not in my original log at all** — bluetooth was part of the archinstall package selection |
| 13:19:59 | `sof-firmware` | 1 | Sound Open Firmware for Intel audio |
| 13:20:02 | `pipewire pipewire-alsa pipewire-jack pipewire-pulse gst-plugin-pipewire libpulse wireplumber` | 76 | Audio = pipewire, as documented |
| 13:20:19 | `power-profiles-daemon` | 7 (incl. `upower`) | This is *why* it was already present for the later TLP conflict — archinstall put it there, I never installed it by hand |
| 13:20:21 | `cups system-config-printer cups-pk-helper` | 75 | **Also not in my original log** — printing support, pulling in a large GTK3 stack as a side effect |
| 13:20:35 | `firewalld` | 3 | **Also not in my original log** |

**Correction to my original log:** I described the archinstall profile as "minimal (no DE, no display manager — pure TTY to startx)" and only listed bootloader/swap/audio/kernel/network/filesystem/timezone as selections. The real transaction log shows archinstall also installed **bluetooth, CUPS printing, and firewalld** in this same phase — these must have been picked in archinstall's package-selection step but I never wrote them down.

### 13:36:48 — `pacman -S nano`

The very first command run after first boot, logged in as root. Confirms my original account: nano really was missing and really was the first thing I fixed.

*(A ~66-minute gap follows — almost certainly the time spent on GRUB `memmap=` chroot config, rebooting, and verifying `/proc/cmdline`, none of which touches pacman.)*

### 14:46:13 — `pacman -S linux-lts-headers`

Pulls in `pahole` + `linux-lts-headers`. Not part of archinstall's original selection — added by hand, presumably once something needed kernel headers.

*(Two more `pacman -S linux-lts` / `pacman -S sudo` calls around this time produced zero new installs — both packages were already present from the archinstall phase; these look like verification commands, consistent with my instinct to run `pacman -Q linux-lts` after first boot.)*

### 14:54:40 — Xorg & display

```bash
pacman -S xorg-server xorg-xinit xorg-xrandr xorg-xsetroot xorg-xprop xorg-xwininfo xf86-input-libinput
```

**A real typo exists in the log:** at 14:54:26, the exact same command was run with `xorg-xwinfo` (missing an `n`) instead of `xorg-xwininfo`, which would have failed to resolve — I corrected it 14 seconds later. 26 new packages landed, mostly font/input plumbing (`libxfont2`, `libinput`, `libwacom`).

### 14:55:36 — GPU/media stack

```bash
pacman -S mesa intel-media-driver libva-intel-driver vulkan-intel
```

9 new packages: `intel-gmmlib`, `libva`, Vulkan loader/layers.

### 15:02:27 — Window manager stack

```bash
pacman -S bspwm sxhkd polybar picom dunst libnotify xclip thunar feh alacritty rofi xss-lock i3lock brightnessctl playerctl
```

**Another real typo:** 11 seconds earlier, I ran the same command with `brighnessctl` (missing a `t`) instead of `brightnessctl` — it failed, and I corrected it. 35 new packages landed, dominated by Thunar's GTK/Xfce dependency chain (`libxfce4ui`, `exo`, `xfconf`).

> **Open question: `scrot` was never installed.** My original log's command list includes `scrot`, but it does not appear anywhere in `pacman.log` — not installed, not upgraded, not removed. Same for `flameshot`, which my `sxhkdrc` is wired to for the `Print` / `super + Print` bindings. Neither tool has ever touched this system via pacman. Either those keybindings have never actually worked, or I set up a screenshot tool some other way (AUR/Flatpak) that leaves no pacman trace — I can't tell which from the log alone. **To check next time I touch bspwm config.**

### 15:09:56 — Neovim

```bash
pacman -S neovim python-pynvim
```

18 new packages — the Lua/tree-sitter runtime (`luajit`, `tree-sitter-c/lua/markdown/query/vim/vimdoc`) matching the `lazy.nvim` + Mason + tree-sitter setup in my dotfiles.

**Correction to my earlier assumption:** I had guessed, from directory mtimes alone, that Neovim was installed on 2026-06-12. The real log shows it was installed here, on Day 1 — 2026-06-12 was just a routine system upgrade that happened to bump Neovim's version, resetting its directory's mtime and making it *look* like a fresh install. Exactly the trap I warned about above.

### 15:13:55–15:14:45 — The big one: dev toolchain + Firefox

First attempt at 15:13:55 included `g++` as a target — **not a real Arch package name** (C++ support ships inside the `gcc` package itself), so it failed. I retried 50 seconds later without `g++`:

```bash
pacman -S git base-devel gcc make cmake python python-pip python-virtualenv \
  rustup go jdk-openjdk openssh curl wget htop btop ripgrep fd bat eza \
  man-db man-pages unzip zip p7zip tmux firefox tree strace ltrace gdb \
  valgrind perf clang llvm meson ninja
```

146 new packages in one shot — by far the largest single manual command of the day. This is where my HPC/quant-finance toolchain (`git`, `gcc`, `clang`, `llvm`, `go`, `jdk-openjdk`, `cmake`, `meson`, `ninja`, `gdb`, `valgrind`, `perf`, `strace`, `ltrace`) and my CLI quality-of-life tools (`ripgrep`, `fd`, `bat`, `eza`, `htop`, `btop`, `tree`) were all installed together, in a single command — not built up gradually like I originally speculated.

**Where the giant multimedia stack actually comes from:** `firefox` in this same command is what pulls in the ~70-package video/audio codec chain visible in the log (`ffmpeg`, `x264`, `x265`, `libvpx`, `libass`, `aom`, `svt-av1`, `sdl2-compat`, etc.) — needed for in-browser video playback. I'd originally (and wrongly) blamed `kitty` for this stack, which wasn't even installed yet at this point.

### 15:18:29–15:27:51 — Small additions

- `fastfetch` (+`yyjson`)
- `python-pipx` (+`python-click`, `python-distro`, `python-userpath`, `python-argcomplete`) — the pipx tool for `black`/`ruff`/`ipython`/`uv`
- `openmpi` (+`hwloc`, `libfabric`, `openpmix`, `openucx`, `prrte`) and `blas lapack cblas` — the HPC/quant math stack

### 16:36:12 — `pavucontrol`, second pipewire pass

```bash
pacman -S pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber pavucontrol
```

Everything except `pavucontrol` was already installed at 13:20; pacman correctly no-ops those and only pulls `pavucontrol`'s own new dependencies (20 packages — `gtk4`, `libsigc++`, `gssdp`/`gupnp` for UPnP, etc.).

### 16:37:47 — Fonts, 16:38:42 — TLP

```bash
pacman -S ttf-jetbrains-mono-nerd ttf-fira-code noto-fonts noto-fonts-emoji noto-fonts-cjk ttf-font-awesome otf-font-awesome
pacman -S tlp tlp-rdw acpi_call-lts
```

The TLP command pulls in `hdparm`, `iw`, `usbutils` as dependencies. Note that `acpi_call-lts` was installed **here, on Day 1**, alongside `tlp` — matching my own framing of it as part of the TLP setup, not a later addition like my first draft guessed.

### 18:14:08–18:14:40 — `fzf`, `tree-sitter-cli`

### 18:45:24–18:45:34 — The Rust episode (my original log contradicted)

```bash
pacman -R rust
pacman -Rns rust
```

**Neither command has a corresponding `[ALPM] removed` entry anywhere in the log — because `rust` was never installed in the first place.** There is no `installed rust` line at any point in this system's history.

My original log describes a story where I ran `pacman -S rust` first, hit a conflict with `rustup`, and removed it with `pacman -Rns rust`. The removal *commands* are real (they're in the log), but they almost certainly both failed with `error: target not found: rust`, since there's no install event to undo. The official `rustup` pacman package (not the curl installer I described) is what actually ended up on this system — installed back at 15:14:45 as part of the big toolchain command above. My original log recorded an intention or a remembered-but-not-actually-executed step, not what really happened.

### 18:50:15 — Building `yay`

```bash
pacman -U /home/nikhil/yay/yay-12.6.0-1-x86_64.pkg.tar.zst /home/nikhil/yay/yay-debug-12.6.0-1-x86_64.pkg.tar.zst
```

`yay` has to be built locally with `makepkg` and installed via `pacman -U` (installing a local file) — there's no chicken-and-egg way to `pacman -S yay` before an AUR helper exists.

### 19:03:43 — `timeshift`, via the new `yay`

```bash
pacman -S --config /etc/pacman.conf -- extra/timeshift
pacman -D -q --asexplicit --config /etc/pacman.conf -- timeshift
```

This exact two-step shape (`-S` naming the repo explicitly as `extra/timeshift`, immediately followed by a separate `-D --asexplicit` call) is `yay`'s own internal invocation style — this is `yay -S timeshift` being run 13 minutes after `yay` itself was built, even though `timeshift` isn't an AUR package. 14 new dependencies landed, including the `xapp`/`vte3` GTK stack and `cronie` (for scheduled snapshots).

### 19:30:39–20:07:22 — Final stretch of Day 1

- `xdg-utils`
- `pacman-contrib reflector` (for the `maintain` script's `paccache` and mirror-refresh steps), immediately followed by a `pacman -Syu` and several repeated `yay`-style empty sync calls between 19:51 and 20:00 — looks like I ran `yay -Syu` a few times back to back, probably testing the freshly built AUR helper
- `github-cli` — the last new package of the day, at 20:07:22

---

## Day 2 — 2026-06-11 (5 new packages)

- 11:13:14 — `terminus-font`
- 11:21:49 — `kitty` (+`librsync`, `kitty-terminfo`, `kitty-shell-integration`)

A small, deliberate addition of an alternate GPU-accelerated terminal and a bitmap font — not bundled with anything else.

## Day 3 — 2026-06-12 (0 new packages, 7 upgraded)

A plain `pacman -Syu` at 19:49:45 bumped `bash`, `github-cli`, `neovim`, `pcsclite`, `python-cryptography`, `python-python-discovery`, `python-virtualenv` to newer versions. Nothing new was installed — this is the day my mtime-only first draft mistook for "Neovim install day."

## Day 4 — 2026-06-21 (1 new package, 63 upgraded)

`pacman -Syu` (preceded by an `archlinux-keyring` refresh, needed to trust newer package signatures) upgraded 63 packages, including the **first of two observed `linux-lts` kernel point-releases** in this window, plus `firefox`, `mesa`, the `ffmpeg`/`pipewire` stack, `cmake`, `github-cli`, `openmpi`, `ripgrep`, `sudo`, `vulkan-intel`. My pacman hooks (`grub-update.hook`, `uki-update.hook`) would have fired automatically here to rebuild the UKI with `memmap=` preserved. The **only genuinely new package** this day: `hugo` (static site generator — matches `hyperion-deployment-blog.md` elsewhere in this repo).

## Day 5 — 2026-06-29 (1 new package, 85 upgraded)

Another large `pacman -Syu` — **the second `linux-lts` kernel point-release** in this window, plus a `gcc`/`glibc` toolchain bump, another `ffmpeg`/`firefox` refresh, and version bumps to `fzf`, `tmux`, `nano`, `fastfetch`, `firewalld`, `perf`. The only new package: `libhwasan` — not something I typed; it appeared because the newer `gcc` split its HWAddressSanitizer runtime into its own sub-package, which pacman had to install to satisfy the upgraded `gcc`'s dependencies.

---

## Cross-Check: My Original Log, Section by Section

| Original log claim | Verdict |
|---|---|
| §1 Bad RAM → ext4/swapfile/LTS kernel choices | Not pacman-verifiable (hardware/filesystem decision, no package evidence either way) |
| §2 archinstall profile = minimal, no DE | **Partially contradicted** — bluetooth, CUPS, and firewalld packages were installed in the same archinstall transaction batch, undocumented in the log |
| §6 nano missing in chroot, installed early | **Confirmed** — first command after boot |
| §8 root vs. user mistake (nvm/rustup/pipx installed as root) | Not pacman-verifiable — those tools don't touch pacman at all |
| §9–10 Xorg + WM stack package lists | **Confirmed**, plus two real typos (`xorg-xwinfo`, `brighnessctl`) I never wrote down |
| §10 `scrot` installed for screenshots | **Contradicted** — never installed, ever |
| §14 `nodejs` removed in favor of nvm | Consistent — no `nodejs` install/removal event exists either (it was never a pacman package here) |
| §15 Python pacman packages | **Confirmed** |
| §16 `rust` installed then removed for `rustup` | **Contradicted** — `rust` was never installed; the two removal commands almost certainly errored out; `rustup` came from the official pacman package, not the curl installer I described |
| §18 TLP + `acpi_call-lts`, power-profiles-daemon masked | **Confirmed**, and clarifies that `power-profiles-daemon` arrived via archinstall itself, not a separate manual step |
| §19 UKI discovery | Not pacman-verifiable (bootloader internals) |
| §20 pacman hooks for kernel updates | Consistent with two kernel upgrades (06-21, 06-29) landing without any reported boot failure |
| §26 `maintain` script deps (`reflector`, `pacman-contrib`, AUR helper) | **Confirmed** |

---

## Summary

| Metric | Count |
|---|---|
| Total installed packages to date | 659 |
| Real install events in `pacman.log` | 659 |
| Real upgrade events in `pacman.log` | 158 |
| Removal events in `pacman.log` | 0 |
| Distinct calendar dates with any activity | 5 |
| New packages after Day 2 (2026-06-11) | 2 (`hugo`, `libhwasan`) |

**Lesson:** when troubleshooting or documenting a Linux system, don't rely on memory, and don't rely on `mtime` — a routine `pacman -Syu` resets it. If a mechanical log exists (`/var/log/pacman.log` here, but the same applies to shell history, journalctl, or git), read it before writing anything down.
