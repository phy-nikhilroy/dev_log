# Building My Fortress: How I Setup My First Hardened VPS on Rocky Linux

Setting up a server for the first time is a rite of passage. I didn't just want a website; I wanted a secure, modern infrastructure where I could deploy anything using Docker. This is the chronicle of how I turned a raw Rocky Linux VPS into a hardened production environment.

---

## Phase 1: The Foundation - Initial Access & Hardening

The moment a VPS is born, it is under attack by bots. My first priority was closing the doors.

### 1. SSH Keys: Moving Beyond Passwords
Passwords can be guessed or brute-forced. SSH keys cannot.
- **Concept:** Asymmetric Cryptography. I generated a public/private key pair on my local machine.
- **Specific Action:** To avoid messing up my existing GitHub keys, I generated a key with a specific filename:
  ```bash
  ssh-keygen -t ed25519 -f ~/.ssh/hostinger_vps -C "my_email"
  ssh-copy-id -i ~/.ssh/hostinger_vps nikhil@<VPS_IP>
  ```

### 2. Local SSH Config
To simplify access and handle terminal compatibility, I added an entry to `~/.ssh/config`:
```text
Host hostinger
  HostName <VPS_IP>
  User nikhil
  IdentityFile ~/.ssh/hostinger_vps
  SetEnv TERM=xterm-256color
  LocalForward 8181 127.0.0.1:81
```
The `SetEnv TERM=xterm-256color` was crucial. My local terminal is kitty, and without this, the remote server didn't recognize my terminal type, causing issues with CLI tools and formatting.

### 3. Disabling the "Easy Way In"
I edited `/etc/ssh/sshd_config` to ensure no one could ever log in with a password again.
- **The Pedantic Detail:** I set `PasswordAuthentication no` and `PermitRootLogin prohibit-password`. This forces the server to only talk to someone who has my specific private key.

### 4. The OS Firewall (`firewalld`)
Rocky Linux uses `firewalld`. Think of this as a bouncer at a club. If your name (port) isn't on the list, you aren't getting in.
- **Action:** I only allowed `ssh`, `http`, and `https`.
- **Lesson:** Always use `--permanent` or your rules will vanish after a reboot.

---

## Phase 2: DNS - The Map to My Server

I bought `nikhilroy.com`. But the internet doesn't know that name; it only knows IP addresses.

### 1. The A Record
- **Concept:** DNS (Domain Name System). I went to Hostinger and created an **A Record**.
- **The "Why":** An A Record maps a domain name to an IPv4 address. I pointed `@` (the root domain) to my VPS IP.
- **The Problem:** `ping nikhilroy.com` failed initially.
- **The Fix:** DNS propagation. It takes time for the "phonebook of the internet" to update globally. I learned to use `nslookup` to check Google's DNS servers (`8.8.8.8`) directly.

---

## Phase 3: Docker - The Container Revolution

I didn't want to clutter my OS with random packages. I wanted everything "containerized."

### 1. The Repository Setup
- **Action:** Added the official Docker repo and installed `docker-ce` and `docker-compose-plugin`.
- **The "Why":** Docker allows me to run applications in isolated environments. If one app breaks, it doesn't take down the whole server.

---

## Phase 4: The Reverse Proxy - Nginx Proxy Manager (NPM)

This was the most complex part of the networking logic.

### 1. The Entry Point
- **Concept:** A **Reverse Proxy** is a server that sits in front of other servers. When someone visits `nikhilroy.com`, they talk to NPM, and NPM decides which Docker container should answer.
- **The SSL Magic:** NPM handles **Let's Encrypt**. It automatically proves to the world that I own the domain and encrypts the traffic (HTTPS).

### 2. Docker Networking: The Private Tunnel
- **Concept:** `docker network create web-network`.
- **The Lesson:** By putting NPM and my website on the same internal Docker network, they can talk to each other by name (e.g., `http://web`). I don't need to expose my website's internal port to the public internet.

---

## Phase 5: The "Aha!" Moment - The Security Leak

I hit a major learning point here. I had closed port 81 (the NPM dashboard) in my firewall, but I could still access it from another laptop. Why?

### 1. The Docker-Iptables "Gotcha"
- **The Discovery:** Docker is aggressive. When you map a port like `- 81:81`, Docker writes its own rules into the Linux kernel (`iptables`) that **bypass** `firewalld`. 
- **The Fix:** I changed the mapping to `- 127.0.0.1:81:81`.
- **The Logic:** This tells Docker: "Only listen for requests coming from *inside* this machine (the loopback address)." Now, the public internet can't see port 81 at all.

### 2. SSH Tunneling: The Secret Passage
- **Action:** `ssh -L 8181:localhost:81 hostinger`
- **Concept:** Since port 81 is only listening *inside* the server, I created a "wormhole" through SSH. My local computer's port 8181 now maps directly to the VPS's internal port 81. It is encrypted, private, and invisible to everyone else.

### 3. The "Stuck" Logout: A Lesson in Persistent Connections
- **The Problem:** When I tried to `exit` the SSH session, the terminal would just hang and not return to my local prompt.
- **The Discovery:** SSH is smart. If a tunnel is active and a browser tab is still "holding" that connection open (even in the background), SSH won't close because it thinks the tunnel is still in use.
- **The Fix:** Close the browser tab pointing to `localhost:8181`. Alternatively, I learned the "Secret Handshake": `Enter` then `~.` (tilde and period) forces an immediate disconnect.

---

## Summary of My Stack
- **OS:** Rocky Linux (Hardened)
- **Engine:** Docker (Containerized Apps)
- **Proxy:** Nginx Proxy Manager (SSL & Routing)
- **Security:** SSH Tunneling, Private Docker Bindings, Firewall.

I didn't just follow steps. I built a fortress.
