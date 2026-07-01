# Choosing an Install Method for opencode, and the Node vs Deno vs Bun Rabbit Hole

**Date:** 2026-07-01
**Context:** Installing `opencode` (an AI coding assistant) surfaced five different install paths on Arch, which turned into a detour through JS runtime internals.

---

## Background

Installing software on Arch is usually a one-liner: `pacman -S <package>`, done. `opencode` offered five different paths for Linux instead: `pacman`, a `curl | bash` script, `npm`, `bun`, and `brew`. Working through which one actually made sense forced me to think about trust models across package managers, and seeing `npm`/`bun` listed side by side sent me down a tangent into how Node, Deno, and Bun are actually built.

---

## 1. Available Install Methods on Arch

All five work. Ranked by fit for this system:

### pacman (official repo) — what I went with

```bash
sudo pacman -S opencode
```

- Confirmed present: `extra/opencode 1.17.10-1`.
- **Pros:** fully native, tracked by pacman, updates via normal `pacman -Syu`, clean uninstall, no extra runtime/package manager needed.
- **Cons:** version lags slightly behind upstream latest until the Arch package maintainer syncs it (3 patch versions behind at the time — see the version table below).

### curl (official install script)

```bash
curl -fsSL https://opencode.ai/install | bash
```

- Installs to `$HOME/.opencode/bin`.
- **Version logic**, from reading the install script itself:
  - No flag → queries `https://api.github.com/repos/anomalyco/opencode/releases/latest`, parses `tag_name`, installs that (resolved to **v1.17.13** at the time).
  - Can pin a version: `curl -fsSL https://opencode.ai/install | bash -s -- --version 1.0.180`.
  - Also supports `--binary <path>` (install from a local binary) and `--no-modify-path` (skip shell rc file edits).
  - Detects OS/arch (linux/darwin/windows, x64/arm64), musl vs glibc, and CPU baseline (checks `/proc/cpuinfo` for avx2 on Linux) to pick the right binary variant.
- **Pros:** always latest release, official, no dependency on Node/Bun/Homebrew being installed.
- **Cons:** curl-to-bash trust model; installs outside pacman's tracking; non-standard install location.

### npm

```bash
npm install -g opencode-ai
```

- Requires Node.js.
- **Pros:** familiar if already in JS tooling; version-pinnable.
- **Cons:** slower than bun; global npm install permission/PATH quirks on Linux; untracked by pacman.

### bun

```bash
bun install -g opencode-ai
```

- Requires bun installed.
- **Pros:** much faster installs than npm.
- **Cons:** extra dependency if bun isn't already used; still untracked by pacman.
- **npm and bun use the same package repository** — bun's installer is a faster reimplementation of the npm client, but it pulls from `registry.npmjs.org`, same as npm. Identical package/tarball/integrity either way; only difference is install speed and which global bin dir it lands in. Respects `.npmrc` too.

### brew (Homebrew/Linuxbrew) — not recommended on Arch

```bash
brew install anomalyco/tap/opencode
```

- **Pros:** clean if already a Homebrew user.
- **Cons:** not native to Arch — installs an entire second package manager under `/home/linuxbrew` just for one tool. Redundant given pacman/AUR coverage. Skip unless already using Linuxbrew for other reasons.

### AUR (`paru`/`yay` — equivalent, interchangeable syntax)

```bash
yay -S opencode-bin
```

- Confirmed via the AUR RPC API: `opencode-bin` version **1.17.13-1** (matches the latest GitHub release / curl installer default).
- **Important:** the AUR package metadata shows `Conflicts: opencode` — `opencode-bin` cannot be installed alongside the official `extra/opencode` pacman package. Installing one requires removing the other.
- Since `opencode-bin` just repackages the same upstream binary release with no build step or AUR-specific value-add, and conflicts with the official repo package, sticking with `pacman -S opencode` was the better call despite the version lag — no reason to pull in the AUR for a now-redundant package.

## 2. Version Comparison Snapshot (as of 2026-07-01)

| Source | Version |
|---|---|
| `extra/opencode` (pacman) | 1.17.10-1 |
| GitHub latest release / curl installer default | v1.17.13 |
| AUR `opencode-bin` | 1.17.13-1 |

## 3. The Decision

I chose `pacman -S opencode`. The curl method and the AUR both offered a slightly newer patch version, but the peace of mind of having the package tracked, cleanly uninstalled, and updated alongside the rest of the system was worth more than three patch versions. I didn't want a stray binary living outside pacman's jurisdiction, and pulling from the AUR for something already in `extra` felt redundant.

*(Windows-specific methods, for reference only — not relevant to this Arch laptop: Chocolatey `choco install opencode`, Scoop `scoop install opencode`, npm, Mise `mise use -g github:anomalyco/opencode`, Docker `docker run -it --rm ghcr.io/anomalyco/opencode`. The opencode docs recommend WSL for the best Windows experience.)*

---

## 4. The JS Runtime Tangent: Node vs Deno vs Bun

Seeing `npm` and `bun` listed side-by-side as install options made me curious how fundamentally different Bun actually is under the hood.

### Implementation languages

| Runtime | Core language | JS engine | Engine's own language |
|---|---|---|---|
| Node | C++ (glue) + JS | V8 (Google) | C++ (+ hand-written/generated assembly for JIT tiers — Turbofan/Maglev) |
| Deno | **Rust** | V8 (Google) | same as above |
| Bun | **Zig** | JavaScriptCore / JSC (Apple, from WebKit) | C++ (+ assembly for baseline/DFG/FTL JIT tiers) |

### Is Bun's engine a fork of WebKit/Safari?

Not a fork of WebKit as a whole — Bun embeds only **JavaScriptCore (JSC)**, the JS-engine component of WebKit, not the rendering/DOM/layout parts. Bun maintains a **patched/vendored fork of JSC specifically**, since JSC wasn't originally designed to be embedded standalone outside WebKit/Safari — Bun carries modifications to make that work.

### What language is Bun actually "implemented in"? A fuller breakdown

Not just Zig:

- **Zig** — Bun's own code: bundler, transpiler, package manager logic, HTTP server, filesystem/IO, glue code. What the Bun team actually authors day-to-day.
- **C++** — the embedded JavaScriptCore engine (vendored/patched from Apple/WebKit), doing the actual JS parsing/execution/JIT. Bun doesn't author this, just links against it.
- **C++/C bindings layer** — glue code at the Zig↔JSC boundary (JSC's API is C++), plus vendored C libs (BoringSSL for TLS, zlib/zstd for compression, etc.).
- **Some JS/TS** — built-in modules/polyfills that ship as JS running on the engine itself.

This is the same general pattern as Node (JS/C++ glue embedding V8) or Deno (Rust embedding V8) — a host language wrapping a C++ JS engine as a dependency. Bun isn't unique in "wrapping someone else's engine," just in which engine and which host language it picked.

### Node vs Deno vs Bun comparison table

| | Node | Deno | Bun |
|---|---|---|---|
| Core implemented in | C++ (JS/C++ glue) | Rust | Zig |
| JS engine | V8 | V8 | JavaScriptCore |
| Created by | Ryan Dahl (2009) | Ryan Dahl (2018, explicitly as a "fix" to Node's design mistakes) | Jarred Sumner / Oven (2021) |
| TypeScript | Needs transpiler (ts-node/tsx); Node 22+ has experimental type-stripping | Built in, zero config | Built in, zero config |
| Module system | CommonJS + ESM (historically messy) | ESM only, URL/JSR imports, `npm:` specifier for compat | CommonJS + ESM, Node-compatible |
| Package manager | npm (separate tool) | Mostly URL-import based; npm compat improved in Deno 2 | Built-in, npm-registry compatible, fast |
| Node API compat | is Node | Improved a lot in Deno 2, still gaps | High — designed as a Node drop-in replacement |
| Security model | None by default — full system access | **Sandboxed by default** — needs explicit `--allow-net`, `--allow-read`, etc. | None by default, like Node |
| Built-in tooling | Minimal — needs separate linter/bundler/test runner | All-in-one: formatter, linter, test runner, bundler, LSP | All-in-one: bundler, test runner, package manager |
| Ecosystem maturity | Largest, most battle-tested | Smaller, growing, own registry (JSR) + npm compat | Newer, growing fast |
| Performance focus | Baseline, not a speed pitch | Faster startup than Node; security adds some overhead | Optimized hard for speed — startup, installs, bundling, tests |

### Why Deno wasn't an install option for opencode

`opencode` ships as a normal npm package (`opencode-ai`), which is why npm/bun work as install methods without modification — Bun's Node-API compatibility makes it a drop-in faster alternative. Deno's stricter security/import model would likely need real adaptation to run a typical npm package like this, which is presumably why opencode doesn't list Deno as an install option at all.

---

## Key Takeaways

- When a tool offers `pacman`/`curl`/`npm`/`bun`/`brew` side by side, the real question is trust and tracking, not speed — a package manager that can cleanly uninstall and upgrade beats a slightly newer patch version living untracked in `$HOME`.
- Check AUR metadata (`yay -Si <pkg>`) for `Conflicts:` before reaching for an AUR variant of something already in the official repos — `opencode-bin` explicitly conflicts with `extra/opencode`.
- npm and bun are two clients for the *same* registry (`registry.npmjs.org`) — switching between them changes install speed, not package provenance.
- Bun isn't a Safari fork; it vendors and patches JavaScriptCore specifically, wrapped in Zig for everything outside the engine itself (bundler, HTTP, filesystem, package manager).
- Deno's default-sandboxed security model is the reason it doesn't show up as an install option for npm-shaped CLI tools like opencode — Bun's Node compatibility is the more frictionless swap.
