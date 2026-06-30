# Nextcloud All-in-One (AIO) Setup

> Self-hosted cloud storage using Nextcloud's official AIO Docker image — trading a monthly storage subscription for physical storage and a bit of setup time.
>
> **Machine:** Linux Mint 22.3 (Ubuntu 24.04 Noble, x86_64)
> **Reference:** [nextcloud/all-in-one](https://github.com/nextcloud/all-in-one)

---

## Table of Contents
- [Overview & Trade-offs](#overview--trade-offs)
- [Install Docker](#install-docker)
- [Get the Compose File](#get-the-compose-file)
- [First Run](#first-run)
- [Initial Login & Domain](#initial-login--domain)
- [How It Works](#how-it-works)
- [Allowing Nextcloud to Access Host Directories](#allowing-nextcloud-to-access-host-directories)

---

## Overview & Trade-offs

AIO bundles everything — the app, database, caching, office suite, video calls — into one managed stack instead of installing each piece manually.

| Gain | Lose |
| :--- | :--- |
| Easy deployment | Full control |
| Automatic updates | Transparency into each component |
| Working stack out of the box | Deep understanding of the underlying setup (for now) |

---

## Install Docker

See [Docker Setup](./Docker-Setup.md) if Docker isn't installed yet.

> **Warning:** Snap-based Docker installations are **not supported** by AIO. Check with:
> ```bash
> sudo docker info | grep "Docker Root Dir" | grep "/var/snap/docker/"
> ```
> If this returns a match, Docker was installed via Snap and needs to be reinstalled using the official method instead.

---

## Get the Compose File

Official source: [`nextcloud/all-in-one/compose.yaml`](https://github.com/nextcloud/all-in-one/blob/main/compose.yaml)

The minimal working configuration used in this setup:

```yaml
name: nextcloud-aio
services:
  nextcloud-aio-mastercontainer:
    image: ghcr.io/nextcloud-releases/all-in-one:latest
    init: true
    restart: always
    container_name: nextcloud-aio-mastercontainer  # Required — do not rename
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config  # Required — backup solution depends on this name
      - /var/run/docker.sock:/var/run/docker.sock:ro
    network_mode: bridge
    ports:
      - "80:80"
      - "8080:8080"   # AIO interface (self-signed HTTPS)
      - "8443:8443"

volumes:
  nextcloud_aio_mastercontainer:
    name: nextcloud_aio_mastercontainer  # Required — do not rename
```

> The `mastercontainer_name` and the `nextcloud_aio_mastercontainer` volume name are hardcoded requirements — renaming either breaks AIO's backup system.

<details>
<summary>Full compose file with all optional settings (reverse proxy, GPU passthrough, custom MTU, etc.)</summary>

```yaml
name: nextcloud-aio # Add the container to the same compose project like all the sibling containers are added to automatically.
services:
  nextcloud-aio-mastercontainer:
    image: ghcr.io/nextcloud-releases/all-in-one:latest # Switch to :beta to help test new releases.
    init: true
    restart: always
    container_name: nextcloud-aio-mastercontainer # Not allowed to be changed.
    volumes:
      - nextcloud_aio_mastercontainer:/mnt/docker-aio-config # Not allowed to be changed.
      - /var/run/docker.sock:/var/run/docker.sock:ro
    # devices: ["/dev/dri"] # Uncomment to enable hardware acceleration (only if /dev/dri exists on host).
    network_mode: bridge
    # networks: ["nextcloud-aio"]
    ports:
      - "80:80"       # Removable if behind a reverse proxy
      - "8080:8080"   # AIO interface
      - "8443:8443"   # Removable if behind a reverse proxy
    # security_opt: ["label:disable"] # Needed for SELinux
    # environment:
      # AIO_DISABLE_BACKUP_SECTION: false
      # APACHE_PORT: 11000              # Needed when behind a reverse proxy
      # APACHE_IP_BINDING: 127.0.0.1
      # APACHE_ADDITIONAL_NETWORK: frontend_net
      # BORG_RETENTION_POLICY: --keep-within=7d --keep-weekly=4 --keep-monthly=6
      # AIO_LOG_LEVEL: warn
      # COLLABORA_SECCOMP_DISABLED: false
      # DOCKER_API_VERSION: 1.44
      # FULLTEXTSEARCH_JAVA_OPTIONS: "-Xms1024M -Xmx1024M"
      # NEXTCLOUD_DATADIR: /mnt/ncdata          # ⚠️ Do not change after initial install
      # NEXTCLOUD_MOUNT: /mnt/
      # NEXTCLOUD_UPLOAD_LIMIT: 16G
      # NEXTCLOUD_MAX_TIME: 3600
      # NEXTCLOUD_MEMORY_LIMIT: 512M
      # NEXTCLOUD_TRUSTED_CACERTS_DIR: /path/to/my/cacerts
      # NEXTCLOUD_STARTUP_APPS: deck twofactor_totp tasks calendar contacts notes
      # NEXTCLOUD_ADDITIONAL_APKS: imagemagick
      # NEXTCLOUD_ADDITIONAL_PHP_EXTENSIONS: imagick
      # NEXTCLOUD_ENABLE_NVIDIA_GPU: true       # Requires an NVIDIA GPU on the host
      # NEXTCLOUD_KEEP_DISABLED_APPS: false
      # SKIP_DOMAIN_VALIDATION: false
      # TALK_PORT: 3478
      # WATCHTOWER_DOCKER_SOCKET_PATH: /var/run/docker.sock

#   # Optional: Caddy reverse proxy. See https://github.com/nextcloud/all-in-one/discussions/575
#   caddy:
#     image: caddy:alpine
#     restart: always
#     container_name: caddy
#     volumes:
#       - caddy_certs:/certs
#       - caddy_config:/config
#       - caddy_data:/data
#       - caddy_sites:/srv
#     network_mode: "host"
#     configs:
#       - source: Caddyfile
#         target: /etc/caddy/Caddyfile
# configs:
#   Caddyfile:
#     content: |
#       https://cloud.example.com:443 {
#         reverse_proxy localhost:11000
#       }

volumes:
  nextcloud_aio_mastercontainer:
    name: nextcloud_aio_mastercontainer # Not allowed to be changed.
  # caddy_certs:
  # caddy_config:
  # caddy_data:
  # caddy_sites:

# networks:
#   nextcloud-aio:
#     name: nextcloud-aio
#     driver_opts:
#       com.docker.network.driver.mtu: 1440
```

Full reference with explanations for every option: [compose.yaml on GitHub](https://github.com/nextcloud/all-in-one/blob/main/compose.yaml)

</details>

---

## First Run

Create a directory and place the compose file inside:

```bash
mkdir ~/containers/nextcloud-aio-mastercontainer
cd ~/containers/nextcloud-aio-mastercontainer
# place compose.yaml here

docker compose -f compose.yaml up -d
```

Check that it started correctly:

```bash
docker logs -f nextcloud-aio-mastercontainer
docker ps
```

A successful startup log looks roughly like:

```
Trying to fix docker.sock permissions internally...
Creating docker group internally with id 984
Initial startup of Nextcloud All-in-One complete!
You should be able to open the Nextcloud AIO Interface now on port 8080 of this server!
E.g. https://internal.ip.of.this.server:8080
```

> **Important:** Access port `8080` using an **IP address**, not a domain — HSTS can block domain access to this port later once a real domain is configured.

---

## Initial Login & Domain

1. Open `https://<server-internal-ip>:8080` and save the passphrase shown.
2. Choose how to assign a domain — see [Domain Setup](./Nextcloud-Domain-Setup.md) for the available options (DuckDNS, Cloudflare + Apache, or Cloudflare + Nginx Proxy Manager).
3. Once a domain is entered and validated, AIO deploys the rest of the stack automatically: Apache, PostgreSQL, Nextcloud, Redis, Talk, Office, Whiteboard, and more.
4. Initial admin credentials are displayed after the stack finishes deploying — use these to log in.

> **Forgot the admin password and have no SMTP configured, but have root access to the server?**
> ```bash
> docker ps
> docker exec -it nextcloud-aio-nextcloud bash
> sudo -E -u www-data php occ user:resetpassword <username>
> ```

> **Where are the actual files stored?**
> `/var/lib/docker/volumes/nextcloud_aio_nextcloud_data/_data/` on the host machine.

---

## How It Works

```
            MASTER CONTAINER
                   │
   ┌───────┬───────┼───────┬────────┐
APACHE  NEXTCLOUD  POSTGRESQL  REDIS  ...
```

The **mastercontainer** is a controller, not the cloud server itself. It uses `/var/run/docker.sock` to spawn and manage the rest of the stack.

| Container | Role |
| :--- | :--- |
| **Apache** | The front door — every request hits this first. Handles HTTPS via Let's Encrypt and forwards traffic internally to Nextcloud. Relevant when placing a reverse proxy in front of it. |
| **PostgreSQL** | The database — stores accounts, sharing permissions, metadata, app settings, folder structure. Not the files themselves. |
| **Nextcloud** | The actual application — the PHP backend running file management, auth, and the app framework. |
| **Client Push** | Enables real-time notifications, so clients don't have to repeatedly poll the server for changes. |
| **Redis** | Caching and session storage, plus file locking to prevent two users corrupting the same file simultaneously. |
| **Nextcloud Office** | Collabora-based engine for editing `.docx`/`.xlsx`/`.odt` directly in-browser. |
| **Nextcloud Talk** | Built-in video calls, chat, and messaging. |
| **Nextcloud Whiteboard** | Collaborative whiteboard — self-hosted Miro equivalent. |

Normally each of these would be set up individually. AIO deploys all of them as a bundle.

### Access Points

**Before installation (AIO setup only):**
- Reachable at `https://<local-ip>:8080`
- This is just the AIO control panel — not the final app
- A domain is required to proceed past this stage

**After installation:**
- All containers listed above are running
- Apache handles automatic SSL/TLS (Let's Encrypt), reverse proxying, and routing to internal containers

### DDNS vs. a Real Domain

| | DuckDNS (DDNS) | Real Domain (e.g. via Cloudflare) |
| :--- | :--- | :--- |
| Cost | Free | Paid (domain registration) |
| Setup | Basic | Basic, plus extras |
| Extras | None | DDoS protection, SSL automation, proxying, DNS hosting |

---

## Allowing Nextcloud to Access Host Directories

By default, the Nextcloud container is sandboxed and cannot see directories on the host OS. To use **local external storage** (mounting a host folder into Nextcloud), add the `NEXTCLOUD_MOUNT` environment variable to the mastercontainer **before** the image line in the compose file.

> If the mastercontainer is already running, it needs to be stopped, removed, and recreated with this variable added — no data is lost in the process.

Allowed values must start with `/` and not be `/` itself:

| OS | Example |
| :--- | :--- |
| Linux | `NEXTCLOUD_MOUNT: "/mnt/"` or `"/media/"` |
| macOS | `NEXTCLOUD_MOUNT: "/Volumes/your_drive/"` |
| Synology | `NEXTCLOUD_MOUNT: "/volume1/"` |
| Windows | `NEXTCLOUD_MOUNT: "/run/desktop/mnt/host/d/your-folder/"` (maps to `D:\your-folder`) |

> ⚠️ This does not work with external USB or network drives — only internal SATA/NVMe drives.

After setting this, fix permissions on the target directory:

```bash
sudo chown -R 33:0 /mnt/your-drive-mountpoint
sudo chmod -R 750 /mnt/your-drive-mountpoint
```

On Windows, run the equivalent inside the container:

```bash
docker exec -it nextcloud-aio-nextcloud chown -R 33:0 /run/desktop/mnt/host/d/your-folder/
docker exec -it nextcloud-aio-nextcloud chmod -R 750 /run/desktop/mnt/host/d/your-folder/
```

Then in the Nextcloud web UI: enable the **External Storage** app at `/settings/apps/disabled`, then add a local storage entry at `/settings/admin/externalstorages` pointing to the same path used above.

> ⚠️ Locations added this way are **not** covered by AIO's built-in backup. Add the corresponding host paths to your own backup routine separately.

---

**Next:** [Domain Setup](./Nextcloud-Domain-Setup.md) · [Troubleshooting](./Nextcloud-Troubleshooting.md)
