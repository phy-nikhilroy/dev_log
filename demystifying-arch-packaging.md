# AUR, pacman, and Cross-Distro Packaging: How They Actually Work

**Date:** 2026-07-01
**Context:** Started from a simple question — what's a good markdown viewer for Arch? — and turned into a full pass through package-manager internals.

---

## Background

I wanted a terminal-first markdown viewer that would render headers and code blocks directly in Alacritty. Research turned up two options: `glow` and `mdcat`. When `pacman -S mdcat` failed, it sent me down a rabbit hole into what the AUR actually is, why other distros don't have one, and why `.pkg.tar.zst`/`.deb`/`.rpm` all exist separately in the first place. This log is that whole pass, in the order the questions came up.

---

## 1. Picking a Markdown Viewer

**`glow`** — confirmed available in the official `extra` repo (not AUR):

```bash
sudo pacman -S glow
glow installed_packages.md      # render once
glow -p installed_packages.md   # paginated, less-style
glow                            # TUI browser for all .md files in a dir
```

Renders headers, tables, code blocks, and links with color directly in the terminal — fits a workflow that's already using `bat`/`fzf`/`ripgrep`.

Other options I considered:
- **`mdcat`** — turned out to be **AUR-only**, not in official repos (see §2). Installable via `yay -S mdcat` if I wanted it instead of `glow`.
- **`markdown-preview.nvim`** / **`glow.nvim`** — Neovim plugins, worth it if I'm *editing* markdown often rather than just reading it, since they preview from inside the editor instead of switching to a separate command.

I went with `glow`. Official-repo binary install, no build step required.

## 2. Why `mdcat` Wouldn't Install via pacman

```bash
pacman -Si mdcat     # package not found in any official repo (core/extra/multilib)
yay -Si mdcat         # Repository: aur
yay -S mdcat          # how to actually install it
```

It simply isn't in the official repos — it only exists in the AUR.

## 3. How Many Packages Have I Actually Pulled From the AUR?

```bash
pacman -Qm    # lists "foreign" packages — not found in any sync (official) db
```

Result on this system: **2** — `yay` and `yay-debug` (12.6.0-1), i.e. `yay` itself. Nothing else has actually been pulled from the AUR yet — I've only ever used the AUR helper to install packages that resolved to the official `extra` repo anyway (`timeshift`, `github-cli`), which `yay` handles transparently by just calling `pacman` under the hood.

## 4. How Does the AUR Actually Work? Does It Pull From GitHub?

### Official repos (`core`/`extra`/`multilib`) — plain `pacman -S`

These host actual prebuilt binaries:

1. `pacman -Sy` downloads each repo's package index into `/var/lib/pacman/sync/`.
2. `pacman -S foo` resolves dependencies against that index, downloads the matching `.pkg.tar.zst` binary from a mirror in `/etc/pacman.d/mirrorlist`.
3. Checks the GPG signature against the trusted keyring (`archlinux-keyring`).
4. Extracts the files, and records the install in `/var/lib/pacman/local/<pkg>-<version>/`.

No compiling happens locally — straight binary install.

### The AUR — what `yay`/`makepkg` actually uses

The AUR is **not a package repository** — it's git repos on `aur.archlinux.org`, one per package, each containing only a **`PKGBUILD`** (a bash script: name, version, dependencies, and a `source=()` array pointing to wherever the *actual* source code lives — GitHub, GitLab, a project's own tarball server, anywhere). No binaries, no vetting, anyone can upload one.

When `yay -S somepkg` runs:

1. Clones/pulls that package's `PKGBUILD` repo from `aur.archlinux.org` — the only part that's actually "the AUR."
2. Shows the `PKGBUILD` for review (should be read before trusting it, especially for low-vote/obscure packages).
3. `makepkg` downloads the real source from the script's `source=()` targets, verifies checksums/signatures, runs `build()` (compiles) and `package()` (stages files).
4. Produces a normal `.pkg.tar.zst` — same format as an official package.
5. Installs it via `pacman -U`.

I have a real example of this from my own `pacman.log` — `yay` had to be bootstrapped this exact way, since no AUR helper existed yet to install it:

```
2026-06-10 18:50:15  pacman -U /home/nikhil/yay/yay-12.6.0-1-x86_64.pkg.tar.zst \
                               /home/nikhil/yay/yay-debug-12.6.0-1-x86_64.pkg.tar.zst
```

**So no — it doesn't pull "everything from GitHub."** The AUR itself only ever hosts the *recipe*; the recipe's `source=()` line can point anywhere. GitHub is just where a lot of upstream projects happen to host releases.

**What "the AUR" really means:** a crowdsourced index of build recipes — a naming/hosting layer for `PKGBUILD`s, not a source-code or binary host. This is also why `yay` is written in **Go** (confirmed via `pacman -Qi yay` → "Pacman wrapper and AUR helper written in go").

## 5. Why Doesn't Something Like the AUR Exist on Fedora or Debian?

It does, in a different shape:

- **Fedora → COPR** (`copr.fedorainfracloud.org`): users submit `.spec` files, Fedora's own build servers compile real RPMs server-side, hosted under each maintainer's personal repo namespace (opt-in individually via `copr enable user/repo` — not one global helper like `yay`).
- **Ubuntu/Debian → PPAs** (Launchpad): same idea — maintainers upload source, Launchpad's builders produce real `.deb`s, each PPA is a separate apt source added by hand. Debian proper leans toward getting things properly sponsored into the *real* archive instead (`mentors.debian.net`).

### Why the AUR's specific model doesn't map onto either

The AUR's defining trait: **just git repos of build recipes, zero build infrastructure, no vetting, one global namespace.** That specific combination doesn't fit Fedora/Debian because:

1. **Rolling vs. frozen releases** — Arch is rolling, so a `PKGBUILD` just builds against whatever's currently installed. Fedora/Debian freeze library versions per release, so old unvetted community scripts are far more likely to break against a "supported" base.
2. **The official archive is already huge** — Debian's `main` alone has ~60,000 vetted packages under the DFSG (Debian Free Software Guidelines). Much of what the AUR covers on Arch is simply already packaged and reviewed on Debian.
3. **Trust/liability posture** — running arbitrary, unreviewed build scripts as part of the ecosystem clashes with Debian's Social Contract and Fedora's Red-Hat-backed QA process. Arch is upfront that the AUR is "unsupported, use at your own risk."
4. **Culture** — Arch expects DIY (manual partitioning, hand-edited GRUB cmdlines, reading `PKGBUILD`s before trusting them). Debian/Fedora's cultures are stability-and-process-first, not vet-it-yourself-first.

### What Fedora/Debian reach for instead

**Flatpak** (Fedora's default now) or **Snap** (Ubuntu) — universal formats that bundle their own runtime/dependencies, sidestepping the need for a distro-specific build recipe entirely. More on these in §7.

## 6. Why Do Packages Need Different Formats for Arch, Fedora, and Debian?

Not a technical necessity — three independent projects solved the same problem (bundle files + describe dependencies + run install hooks) decades apart, with different priorities, and the formats are now too load-bearing to unify.

### Side-by-side

| | Arch (`.pkg.tar.zst`) | Debian (`.deb`) | Fedora (`.rpm`) |
|---|---|---|---|
| Container format | plain `tar` + zstd | old Unix `ar` archive | own binary format |
| Metadata | `.PKGINFO` (flat key-value) | `control` file in `control.tar.xz` | header fields in the RPM binary |
| Install hooks | `.INSTALL` bash functions | `postinst`/`prerm`/etc. scripts | `%pre`/`%post`/etc. scriptlets |
| Build recipe | `PKGBUILD` (plain bash) | `debian/rules` (a Makefile) + `control` + `changelog` | `.spec` (macro-driven DSL) |
| Build tool | `makepkg` | `dpkg-buildpackage` | `rpmbuild` |
| Created | ~2002 | ~1993–94 | 1997 |

Real example — my own installed metadata for `yay` (`/var/lib/pacman/local/yay-12.6.0-1/desc`), showing how minimal Arch's format is:

```
%NAME%       %VERSION%      %DESC%
yay          12.6.0-1       Yet another yogurt. Pacman wrapper and AUR helper written in go.
```

### Why they never converged

1. **Built independently, decades apart** — `dpkg` predates `rpm` by ~3–4 years, both predate `pacman` by roughly a decade. No shared standard existed to build on, and by the time any matured, too many packages already depended on the existing format to change it.
2. **Dependency semantics differ** — Debian has `Depends`/`Recommends`/`Suggests`/`Breaks`/`Conflicts`/`Provides`/`Replaces`; RPM has an overlapping-but-different set; Arch's `PKGBUILD` arrays (`depends=()`/`optdepends=()`/`conflicts=()`) are deliberately the simplest — "the user decides, don't guess for them."
3. **Install-hook execution models are structurally incompatible** — `.deb` scripts run through `dpkg`'s trigger system, RPM scriptlets run through `rpm`'s transaction engine, Arch's `.INSTALL` functions are called directly by `pacman`, which *also* layers a separate global hooks system on top (`/etc/pacman.d/hooks/`) — exactly the mechanism behind the `grub-update.hook`/`uki-update.hook` pair that auto-rebuilds my kernel image on every `linux-lts` upgrade (see `arch-bspwm-thinkpad-t14s.md` §20).
4. **Build tooling reflects philosophy, not just taste** — Debian's `debian/` directory + Policy Manual + `lintian` reflects its obsession with correctness and FHS/licensing compliance; RPM's macro-heavy `.spec` reflects its enterprise/RHEL roots; Arch's `PKGBUILD` is just bash with two functions (`build()`, `package()`), matching Arch's KISS principle — and is also why the AUR can exist as "just push a git repo," with no macro system or policy manual to learn first.
5. **Even a unified format wouldn't guarantee cross-distro binaries anyway** — different `glibc` versions/patches, different compiler hardening defaults, different library sonames, different filesystem-layout conventions (Debian's multiarch `/usr/lib/x86_64-linux-gnu/` vs. Arch's flat `/usr/lib` + separate multilib repo) mean a `.deb`'s contents often wouldn't run correctly on Fedora even if the archive format matched. This is exactly why Flatpak/Snap/AppImage exist — bundle the whole runtime so the binary stops caring what distro it's on.

## 7. Flatpak vs. Snap vs. AppImage

The three "universal package" formats mentioned above as the answer to "how do you get one binary to run on any distro." None of these is installed on this system (`flatpak`/`snapd` both absent), but they're worth understanding as the direct alternative to distro-native packages covered in §6.

### Side-by-side

| | Flatpak | Snap | AppImage |
|---|---|---|---|
| Backer | Community/freedesktop.org (Red Hat-driven) | Canonical (Ubuntu) | Community, no single backer |
| Package internals | OSTree-managed file tree + shared "runtimes" (e.g. GNOME/KDE Platform) | SquashFS image | SquashFS image, bundled inside one executable file |
| Requires a daemon? | No background daemon; uses `bubblewrap` per-launch | Yes — `snapd` must be running | No daemon at all |
| Distribution model | Flathub (or any custom "remote") — decentralized, anyone can host a remote | Snap Store — centralized, run by Canonical, closed-source backend | None — download and run any file from anywhere, no store required |
| Sandboxing | Yes, via `bubblewrap` + portals (filesystem/camera/mic access mediated, similar spirit to Android permissions) | Yes, via `AppArmor` confinement (`strict`/`classic`/`devmode` modes) | No sandboxing by default (can be added manually, e.g. with `firejail`) |
| Needs root to install? | No (per-user installs supported) | Yes, typically (`snapd` install is system-wide) | No — just `chmod +x` and run |
| Disk usage | Better than it sounds — shared runtimes are deduplicated across apps that use the same one | Each snap ships its own full dependency set — more duplication | Each AppImage is fully self-contained — most duplication of the three |
| Auto-updates | Yes, via `flatpak update` (user- or admin-triggered, not silent) | Yes, and **on by default in the background** — a common complaint | No built-in updater; needs a separate tool like `AppImageUpdate`, or the app checks itself |
| Startup overhead | Small (namespace setup per launch) | Noticeably higher — mounting the squashfs + AppArmor + snapd overhead, especially first launch | Minimal — mounts itself and runs, no daemon round-trip |
| Fits my setup | Default on Fedora now; philosophically the "sanctioned" answer to third-party GUI apps on RPM distros | Ubuntu-only by default elsewhere; often actively avoided on other distros due to the closed Snap Store backend | Most Arch-adjacent in spirit — no daemon, no central authority, just a file, matching the "minimal moving parts" preference in the rest of my setup (suckless WM, no DE, no display manager) |

### Why these exist at all

All three solve the exact problem §6 ended on: a `.deb` built on Debian often won't run correctly on Fedora even with a format converter, because the actual library versions, sonames, and compiler defaults differ underneath. Each format sidesteps that by shipping the dependencies *with* the app instead of relying on the host distro to provide matching versions:

- **Flatpak** ships shared "runtimes" (e.g., all GNOME apps built against the same `org.gnome.Platform` version) so multiple apps reuse one dependency set instead of each bundling everything themselves — a middle ground between AppImage's "bundle everything" and a distro's "share everything."
- **Snap** bundles the full dependency tree per-snap (closer to AppImage on this axis), but adds `snapd`-managed confinement and forced background updates — a tradeoff aimed at unattended/server and IoT use as much as desktop apps.
- **AppImage** goes furthest toward "just a file" — no daemon, no central store, no install step at all. That also means no sandboxing and no update mechanism unless the app author wires one in themselves — the tradeoff for zero infrastructure.

### The honest tradeoff underneath all three

None of them eliminates the "different distros expect different things" problem — they just move where the duplication happens. Instead of the *distro* maintaining N slightly different builds of an app (one per distro's library versions, per §6), the *app* now ships its own copy of everything it needs, once, and that copy works everywhere. The cost is disk space and, for Flatpak/Snap, an extra permissions/sandboxing layer to reason about.

---

## Key Takeaways

- The AUR is a crowdsourced index of `PKGBUILD` recipes, not a binary or source host — always read the `PKGBUILD` before trusting an obscure package, since it's an unvetted script running on my machine.
- `pacman -Qm` tells me exactly how many "foreign" (AUR/local) packages are actually installed — useful for auditing how much of my system is genuinely off the official repos.
- Fedora (COPR) and Debian (PPAs) solve the same "community package" problem the AUR does, but server-side and per-maintainer rather than as one global, unvetted namespace — a direct consequence of their frozen-release culture vs. Arch's rolling-release DIY culture.
- `.pkg.tar.zst` / `.deb` / `.rpm` differ because of history, not necessity — three build tools decades apart, with dependency/hook models too different to unify even if the container format did.
- For my minimal, suckless-style setup: stick to native `pacman`/AUR first, reach for Flatpak/Snap/AppImage only when nothing else packages the app at all — and if I ever do, AppImage fits the philosophy best (no daemon, no central authority).
