# Docker Setup
> One-time installation guide for **Ubuntu 24.04 (Noble)** — installed from Docker's official repo, not `apt`'s bundled `docker.io`.

---

## Table of Contents
- [Preparation](#preparation)
- [Installation](#installation)
  - [Install Dependencies](#1-install-dependencies)
  - [Add Docker GPG Key](#2-add-docker-gpg-key)
  - [Add Docker Repository](#3-add-docker-repository)
  - [Install Docker Engine](#4-install-docker-engine)
- [Configuration](#configuration)
- [Verify Installation](#verify-installation)

---

## Preparation

Update the system first:

```bash
sudo apt update && sudo apt upgrade -y
```

Remove any old or conflicting Docker installations:

```bash
sudo apt remove docker docker-engine docker.io containerd runc
```

---

## Installation

### 1. Install Dependencies

```bash
sudo apt install ca-certificates curl gnupg
```

> **Note:** These are usually pre-installed. You can verify with `apt search <package>` before running.

These three packages are needed specifically to securely fetch and verify Docker's GPG signing key in the next step.

---

### 2. Add Docker GPG Key

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

This ensures packages are verified as coming from Docker — prevents tampered or spoofed packages from being installed.

---

### 3. Add Docker Repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  noble stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

> `noble` is the codename for Ubuntu 24.04. Adjust if on a different release.

---

### 4. Install Docker Engine

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

| Package | Role |
| :--- | :--- |
| `docker-ce` | Main engine (daemon) |
| `docker-ce-cli` | CLI — the `docker` command |
| `containerd.io` | Low-level container runtime |
| `docker-buildx-plugin` | Advanced image building |
| `docker-compose-plugin` | Enables `docker compose` (modern version) |

---

## Configuration

Add your user to the `docker` group so you don't need `sudo` for every command:

```bash
sudo usermod -aG docker $USER
```

Apply the group change to the current session without logging out:

```bash
newgrp docker
```

---

## Verify Installation

Run the hello-world container to confirm everything works:

```bash
docker run hello-world
```

Check versions and service status:

```bash
docker --version
docker compose version
docker info
systemctl status docker
```
