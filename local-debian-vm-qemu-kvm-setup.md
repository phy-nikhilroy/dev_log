# From Zero to a Bootable Sandbox: Setting Up My First Local Debian VM with QEMU/KVM

**Date:** July 7, 2026
**Host:** ThinkPad T14s — Arch Linux, i7 11th gen, 16GB RAM
**Goal:** a disposable, isolated environment for testing packages and different runtimes without polluting my main Arch install

I wanted a place to install and try out random software — packages, different language runtimes, things I don't fully trust or just don't want cluttering my actual system — without any risk to my daily-driver Arch install. I already know venvs/containers solve this for language-level or dependency-level isolation, but I wanted the real thing: a full VM with its own kernel, so I could also use it to poke at other distros, test kernel-level stuff, or run genuinely untrusted code with a hard isolation boundary. This log is the complete, warts-and-included account of standing up my first one: a bare-bones Debian 13 guest under QEMU/KVM, managed entirely from the CLI via `virt-install`/`virsh`, no `virt-manager` GUI involved.

I hit more snags than I expected for something this "basic." All of them are documented below, in the order I actually hit them, because future-me will absolutely forget how any of this works by the next time I touch it.

---

## Phase 1: Confirming Hardware Support & Installing the Stack

Before installing anything, checked whether the hardware actually supports virtualization:

```bash
lscpu | grep -i virtualization      # → VT-x
lsmod | grep kvm                    # → kvm_intel, kvm, irqbypass all already loaded
```

Both came back clean — the 11th-gen i7 supports VT-x and the KVM kernel modules were already loaded, so no BIOS/UEFI changes needed.

**Ran a full system update before installing anything new.** Arch explicitly discourages partial upgrades (installing new packages without syncing the whole system first) since it risks dependency mismatches:

```bash
sudo pacman -Syu
```

Then installed the actual virtualization stack:

```bash
sudo pacman -S --needed qemu-full libvirt virt-install dnsmasq
```

- `qemu-full` — the virtualization engine
- `libvirt` — VM lifecycle/network/storage management
- `virt-install` — CLI tool to define and create VMs
- `dnsmasq` — DHCP/DNS for libvirt's default NAT network (turned out later I didn't actually need this — see Phase 7)

Enabled the daemon and added myself to the relevant groups:

```bash
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt,kvm $USER
```

**Note:** group membership changes don't apply to the current shell — needs a new terminal session or a full logout/login to take effect.

---

## Phase 2: Grabbing the Debian ISO

```bash
mkdir -p ~/vms/debian-basic
cd ~/vms/debian-basic
```

Debian's netinst filenames include a point-release number that changes over time, so rather than guessing, tried to scrape the current filename from the official directory listing:

```bash
curl -s https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/ | grep -o 'debian-[0-9.]*-amd64-netinst.iso' | head -1
```

**The problem:** this returned nothing.

**Diagnosis:** checked the HTTP status first (`curl -sI ...` → clean `200 OK`, so not a connectivity/redirect issue), then dumped the raw page (`curl -s ... | head -80`). Turned out the page opens with a huge block of plain-English explanatory text ("What's in this directory?", "What is a netinst image?", etc.) before the actual file listing even appears — the loose grep pattern wasn't the problem, it just needed to actually anchor onto an `href` attribute rather than free text.

**The fix:**

```bash
curl -s https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/ | grep -o 'href="[^"]*\.iso"'
```

This returned the real listing: `debian-13.5.0-amd64-netinst.iso` (plus `debian-edu-*` and `debian-mac-*` variants I didn't need). Downloaded and verified:

```bash
wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-13.5.0-amd64-netinst.iso
curl -s https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/SHA256SUMS | grep netinst
sha256sum debian-13.5.0-amd64-netinst.iso
```

Checksum matched.

---

## Phase 3: The First `virt-install` Attempt and a Naming Collision

Went pure CLI (no `virt-manager`), and since the goal was a headless serial-console-only install, used `--graphics none` with a serial console instead of a graphical viewer:

```bash
virt-install \
  --name debian-basic \
  --memory 2048 \
  --vcpus 2 \
  --disk size=15 \
  --location ~/vms/debian-basic/debian-13.5.0-amd64-netinst.iso \
  --os-variant debian13 \
  --graphics none \
  --extra-args "console=ttyS0,115200n8 serial"
```

(`--os-variant debian13` confirmed available via `osinfo-query os | grep -i debian` — libvirt's osinfo database already knew about Debian 13.)

**The problem:** re-running the command (after the console-attach step appeared to fail with a scary "Domain installation does not appear to have been successful" message) errored with:

```
ERROR    Guest name 'debian-basic' is already in use.
```

**The discovery:** the VM had actually installed and started successfully the first time — the "not successful" message was just `virt-install`'s own console-attach step failing/exiting, not the VM itself. Checking:

```bash
virsh --connect qemu:///session list --all     # → debian-basic, running
virsh --connect qemu:///system list --all      # → empty
virsh uri                                       # → qemu:///session
```

revealed the real gotcha: despite adding myself to the `libvirt`/`kvm` groups (which matter for the *system-wide* `qemu:///system` connection), `virt-install`'s default connection on this box resolved to `qemu:///session` — a per-user, unprivileged libvirt connection that doesn't need those groups or `sudo` at all. All subsequent commands in this whole project explicitly target `qemu:///session` for this reason.

Reconnected to the already-running installer instead of fighting it:

```bash
virsh --connect qemu:///session console debian-basic
```

---

## Phase 4: Installing Debian Over a Serial Console

Picked **"Install"** from the boot menu, not "Graphical install" — a graphical UI can't render over a plain serial line.

**Cosmetic hiccup:** the installer's text UI (built on `newt`) rendered as a tiny box in the top-left corner of the terminal instead of filling the window, and the top of the screen showed tabs like `1 Main / 2 Shell / 3 Shell / 4 Log`. Both are normal: `newt`-based UIs default to a fixed size (traditionally 80x24) over serial rather than detecting real terminal dimensions, and the numbered tabs are the serial-console equivalent of the classic `Alt+F1`–`Alt+F4` virtual console switching you'd use on physical hardware (Main = the installer itself, Shell x2 = manual debugging shells, Log = live installer log). None of it affects functionality — just stay on tab 1 and proceed normally.

Went through the standard installer flow, with two spots that mattered for keeping this minimal:

- **Partitioning:** "Guided - use entire disk" → "All files in one partition" (simplest possible layout for a disposable test VM).
- **Software selection (tasksel):** unchecked everything except **"SSH server"** and **"standard system utilities"** — no desktop environment at all, headless from first boot, SSH working out of the box (in theory — see Phase 7 for the actual networking story).

GRUB installed to `/dev/vda`. Hostname set to `debian` (independent of the VM's libvirt name `debian-basic` — no conflict).

---

## Phase 5: A Clean Restart (Wiping and Redoing)

Mid-install, fumbled a few steps and wanted a completely clean slate rather than trying to fix it in place. The wipe-and-restart procedure, which turned out to be useful more than once during this whole project:

```bash
# Ctrl + ] to detach from the console first
virsh --connect qemu:///session destroy debian-basic                        # force power-off (not delete)
virsh --connect qemu:///session undefine debian-basic --remove-all-storage  # delete config + disk image
virsh --connect qemu:///session list --all                                  # confirm it's gone
```

Then just re-run the exact same `virt-install` command from Phase 3 to start over from the boot menu.

---

## Phase 6: The "Stuck at Loading Initial Ramdisk" Mystery

After the install finished and the VM rebooted, the console showed:

```
Loading Linux ...
Loading initial ramdisk ...
```

...and then nothing. No further output, no login prompt, indefinitely.

**Diagnosis attempt 1 — is it actually hung?** Detached from the console and checked CPU activity twice, a few seconds apart:

```bash
virsh --connect qemu:///session domstats debian-basic --cpu-total
```

`cpu.time` barely moved and `cpu.haltpoll.success.time` didn't move at all between the two checks — consistent with an idling system, but genuinely ambiguous: a fully-booted system sitting at an invisible login prompt looks identical to a wedged one by this metric alone.

**Diagnosis attempt 2 — reconnected to console, pressed Enter several times.** Nothing appeared either way.

**Root cause:** the `console=ttyS0,115200n8 serial` parameter from Phase 3 was passed via `--extra-args`, which only wires into the **installer's own ephemeral kernel boot line** — it never becomes part of the freshly-installed system's permanent GRUB configuration. The installed OS had almost certainly been booting completely fine the entire time; it simply had no instruction to print anything to the serial line, so there was nothing to see. "Stuck" and "silently working" look identical over a console with no output configured.

**Confirming the theory (temporary fix):** did a hard reset (`virsh reset`, which works even on an apparently-wedged domain since it doesn't depend on guest cooperation) and immediately reconnected to console to catch the boot sequence before GRUB's countdown timer expired:

```bash
virsh --connect qemu:///session reset debian-basic
virsh --connect qemu:///session console debian-basic   # reconnect fast, then mash a key to stop the GRUB countdown
```

Caught the GRUB menu, pressed `e` to edit the highlighted entry, found the `linux /boot/vmlinuz ...` line, appended `console=ttyS0,115200n8` to the end, booted with `Ctrl+X`. Full kernel boot log and a `debian login:` prompt appeared immediately — confirming the system had been fine the whole time, just invisible.

**Making it permanent:** logged in (see the `sudo` gotcha below first) and edited `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet console=ttyS0,115200n8"
```

(kept `quiet` — it just suppresses verbose kernel spam, doesn't block the serial console itself), then:

```bash
sudo update-grub
sudo reboot
```

From then on, every boot streams output over serial automatically, ending in a persistent `debian login:` prompt — no more manual GRUB editing needed.

**A second gotcha discovered right here:** trying `sudo update-grub` produced `sudo: command not found`. Debian's minimal install does **not** install `sudo` or add the first user to any admin group by default (unlike Ubuntu, which does both). Had to log out (`exit`) and back in as **`root`** (using the root password set during install) to run admin commands at all for this session. Optionally, once logged in as root:

```bash
apt install sudo
usermod -aG sudo <username>
```

...to get `sudo` working for the regular user going forward (requires logging out/in again for the group to apply).

---

## Phase 7: Getting SSH Working from the Host

Once logged in, checked networking:

```bash
ip a
```

Guest showed `enp1s0: inet 10.0.2.15/24`. Recognized this immediately as QEMU's classic built-in SLIRP usermode-networking subnet — the default for any `qemu:///session` guest, since session-mode has no access to the system-wide bridged `virbr0`/`dnsmasq` NAT network that `qemu:///system` guests get automatically (the network I'd installed `dnsmasq` for in Phase 1, which turned out to be entirely unused here).

SLIRP gives the guest outbound internet access for free, but nothing on the host can reach *into* the guest by IP by default — the private `10.0.2.x` network is invisible to the host's own network stack.

**Live/temporary fix** — added a port-forward rule to the already-running QEMU process via its monitor interface:

```bash
virsh --connect qemu:///session qemu-monitor-command debian-basic --hmp "hostfwd_add tcp::2222-:22"
```

Confirmed `systemctl status ssh` was `active (running)` inside the guest (courtesy of selecting "SSH server" in tasksel back in Phase 4), then from the host:

```bash
ssh -p 2222 nikhil@localhost
```

Worked immediately.

**Subtlety discovered by testing:** ran a guest-level `reset`, expecting the port-forward to disappear — it didn't; SSH still worked afterward. Reasoning: a guest reset/reboot only power-cycles the OS *inside* the VM; the actual QEMU **process** on the host (which is what holds the SLIRP state and the `hostfwd_add` rule) never terminates, so anything injected via the monitor survives guest reboots. It's only lost when the QEMU process itself dies — i.e. a full `virsh destroy` or a guest-initiated shutdown/poweroff followed by a fresh `virsh start`, which spins up a brand-new process with no memory of the earlier monitor command.

**Getting back in after a real shutdown:**

```bash
virsh --connect qemu:///session list --all       # confirm "shut off"
virsh --connect qemu:///session start debian-basic
virsh --connect qemu:///session console debian-basic                                    # always works
# — or, for SSH —
virsh --connect qemu:///session qemu-monitor-command debian-basic --hmp "hostfwd_add tcp::2222-:22"
ssh -p 2222 nikhil@localhost
```

**A third PATH gotcha**, hit while trying to shut the guest down cleanly: `shutdown`/`poweroff`/`shutdown now` all returned `command not found` for the regular (non-root) user. Same root cause as the earlier `sudo` issue — `/usr/sbin` isn't on a plain Debian user's `$PATH`. Fixed with the full path:

```bash
/usr/sbin/poweroff
```

(or `sudo poweroff`, once `sudo` was actually installed).

---

## Phase 8: Making the Port-Forward Permanent

Wanted `ssh -p 2222 ...` to just work after any `virsh start`, without re-running `hostfwd_add` by hand every time.

**Attempt 1:** shut the VM off, then `virsh --connect qemu:///session edit debian-basic`, and added a `<portForward>` block inside the existing `<interface type='user'>` element:

```xml
<interface type='user'>
  <mac address='52:54:00:0d:33:77'/>
  <model type='virtio'/>
  <portForward proto='tcp'>
    <range start='2222' to='22'/>
  </portForward>
  <alias name='net0'/>
</interface>
```

**The problem:** libvirt rejected it on save:

```
error: unsupported configuration: The <portForward> element can only be used with the 'passt' backend of interface type='user' or type='vhostuser'
```

**The fix:** added `<backend type='passt'/>` as the first child of the interface (before `<mac>`), and installed the actual `passt` package on the **host** (it's what implements this backend, not just a schema requirement):

```bash
sudo pacman -S --needed passt
```

```xml
<interface type='user'>
  <backend type='passt'/>
  <mac address='52:54:00:0d:33:77'/>
  <model type='virtio'/>
  <portForward proto='tcp'>
    <range start='2222' to='22'/>
  </portForward>
  <alias name='net0'/>
</interface>
```

Confirmed the host's libvirt (12.5.0, running QEMU 11.0.2 per `virsh version`) fully supports this — worked on the second attempt. `virsh start` followed by a plain `ssh -p 2222 ...` now works immediately with zero manual monitor commands.

---

## Phase 9: A Host-Security Sanity Check

Since this project just opened a forwarded port on my actual laptop, took the opportunity to audit what else is exposed on the host itself:

```bash
systemctl status nftables iptables ufw firewalld
```

Only `firewalld` came back active + enabled. Checked its actual policy:

```bash
sudo firewall-cmd --get-default-zone      # → public
sudo firewall-cmd --list-all
```

`public` zone, active on `wlp0s20f3` (the WiFi interface), allowing only the `dhcpv6-client` and `ssh` services — nothing else, no custom ports, no rich rules.

Then checked what's actually listening:

```bash
sudo ss -tulnp
```

Notable line:

```
tcp   LISTEN   0   128   *:2222   *:*   users:(("passt.avx2",pid=19205,fd=72))
```

`passt` is bound to `*:2222` — **all interfaces**, not just loopback, despite the XML config from Phase 8 not saying anything about the bind address.

**Reasoning:** since port 2222 isn't in firewalld's explicit allow-list for the `public` zone, its default-deny policy should still block inbound connections to it from the LAN or internet, even though the raw socket itself is bound broadly. This is real protection, but it's a single layer, not defense-in-depth — if firewalld were ever disabled or misconfigured, that broad bind would immediately become reachable from the whole LAN.

**Follow-up, not yet done:** restrict the `<portForward>` explicitly with `address='127.0.0.1'` so the port is unreachable from the network at the source, independent of firewalld:

```xml
<portForward proto='tcp'>
  <range start='2222' to='22' address='127.0.0.1'/>
</portForward>
```

And verify empirically from another device on the same WiFi (`nmap -p 2222 <laptop-LAN-IP>`) rather than trusting the theory alone.

Also noted: `cupsd` is listening on `127.0.0.1:631` / `[::1]:631` only — loopback-restricted by default, no concern there.

---

## Phase 10: A Faulty RAM Region — a Non-Issue for VMs

Sidebar that came up mid-project: the host has a `memmap=` kernel parameter excluding a known-bad physical RAM region (see the original Arch install log for the backstory). Question: does this need to be replicated inside every VM guest?

**Answer: no.** A guest never sees real host physical addresses. QEMU/KVM allocates guest RAM only from the host's pool of already-known-good memory — the host kernel already excluded the bad region at its own boot, so nothing downstream (VMs included) can ever be handed it. The guest is then presented with its own private, virtual "physical" address space starting at 0, entirely disconnected from real hardware addressing. Adding `memmap=` inside a guest would just be excluding part of a fake address space that has no relationship to the real faulty hardware — pointless at best, a wasted chunk of usable guest RAM at worst.

Fixing it once at the host level protects everything that runs on that host, VMs included, automatically.

---

## Phase 11: Basic Dev Environment Inside the VM

Neovim as default editor:

```bash
apt install neovim
update-alternatives --config editor      # select /usr/bin/nvim from the list
```

```bash
echo 'export EDITOR=nvim' >> ~/.bashrc
echo 'export VISUAL=nvim' >> ~/.bashrc
```

(`update-alternatives` covers `/usr/bin/editor`, which some tools respect; `$EDITOR`/`$VISUAL` covers everything else that checks the env var directly.)

Git:

```bash
apt install git
git config --global user.name "phy-nikhilroy"
git config --global user.email "..."
git config --global core.editor nvim
git config --global init.defaultBranch main
git config --global pull.rebase false
```

---

## Phase 12: The `user.name` vs. GitHub Username Non-Issue

Noticed a mismatch: the base Arch system has git `user.name` set to `phy.nikhilroy` (period), while the actual GitHub username is `phy-nikhilroy` (dash) — and the VM above got set up with the dash version, creating an inconsistency across machines.

**The realization:** `git config user.name` is purely cosmetic commit metadata — free text baked into each commit, with zero functional connection to GitHub auth, URLs, or attribution. What actually links a commit to a GitHub profile is `user.email` matching a verified email on the account; the GitHub *username* itself only governs repo URLs and `@mentions`, and is read entirely from GitHub's own account settings, never from local git config. So the period/dash mismatch is cosmetic only — it just means `git log` shows a slightly different author string per machine, nothing breaks.

Decided to update the base Arch system to match the dash form anyway, for tidiness:

```bash
git config --global user.name "phy-nikhilroy"
```

Confirmed this only affects the author field of **future** commits — it does not rewrite existing history (old commits permanently keep whatever name was set when they were made), and has zero effect on any current uncommitted work or repo state.

---

## Lessons Learned

- **Always run `pacman -Syu` before installing new packages on Arch** — partial upgrades are explicitly unsupported and can cause dependency mismatches.
- **`virt-install`'s default connection can silently be `qemu:///session`, not `qemu:///system`**, regardless of `libvirt`/`kvm` group membership — check with `virsh uri` before assuming which one you're targeting.
- **A failed-looking `virt-install` console attach doesn't mean the VM install failed** — check `virsh list --all` on both connections before assuming you need to start over.
- **`--extra-args "console=ttyS0..."` only affects the installer's ephemeral boot, never the installed system.** Make the serial console permanent via `/etc/default/grub` + `update-grub` if you want it on every future boot.
- **Debian's minimal install has no `sudo` and no admin group for the first user by default** — log in as `root` first if you need to bootstrap either.
- **SLIRP/usermode networking (the `qemu:///session` default) is NAT-outbound only** — nothing reaches in without an explicit `hostfwd_add` (temporary, tied to the QEMU process) or a `passt`-backed `<portForward>` in the XML (persistent).
- **A guest reboot/reset ≠ a guest shutdown**, when it comes to what survives: the former keeps the same QEMU process (and any monitor-injected state) alive; the latter kills it.
- **`<portForward>` in libvirt requires the `passt` backend explicitly** — the default `user`-type interface backend doesn't support it.
- **A broad socket bind (`*:port`) doesn't necessarily mean network exposure** if a host firewall with a default-deny policy sits in front of it — but relying on that alone isn't defense-in-depth.
- **`memmap=` for a bad RAM region belongs on the host only** — VMs are automatically protected since guest RAM is allocated purely from the host's already-good memory pool.
- **`git config user.name` is cosmetic** — only `user.email` actually ties commits to a GitHub identity, and changing either only affects future commits, never existing history.

---

## The Stack

| Layer | Choice |
|---|---|
| Hypervisor | QEMU/KVM (via `qemu-full`) |
| Management | `libvirt` + `virt-install`/`virsh` (no `virt-manager` GUI) |
| Connection | `qemu:///session` (unprivileged, per-user) |
| Guest OS | Debian 13.5.0, minimal (no desktop, SSH server + standard utilities only) |
| Console | Serial (`ttyS0`), fixed permanently via `/etc/default/grub` |
| Networking | QEMU SLIRP/usermode, `passt` backend for persistent port-forwarding |
| Host access | `ssh -p 2222 nikhil@localhost` |
| Host firewall | `firewalld`, `public` zone, default-deny beyond `ssh`/`dhcpv6-client` |
| VM specs | 2GB RAM, 2 vCPU, 15GB disk, all-in-one partition |

Not a production sandbox by any means, but a genuinely useful disposable box for trying things I don't want touching my main system — and a much better understanding of what's actually happening under the hood of every cloud VPS I've ever clicked "Deploy" on.
