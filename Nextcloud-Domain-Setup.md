# Nextcloud AIO — Domain Setup

> Three ways to assign a domain to the AIO instance, from simplest to most flexible.
> Pick **one** — these are alternative architectures, not steps to combine.
>
> Requires the mastercontainer already running — see [Nextcloud AIO Setup](./Nextcloud-AIO-Setup.md).

---

## Table of Contents
- [Option 1 — DuckDNS](#option-1--duckdns)
- [Option 2 — Cloudflare + Apache (built-in HTTPS)](#option-2--cloudflare--apache-built-in-https)
- [Option 3 — Cloudflare + Nginx Proxy Manager](#option-3--cloudflare--nginx-proxy-manager)
- [Changing the Domain Later](#changing-the-domain-later)

> **Note on the examples below:** `cloud.example.com` and `203.0.113.10` are placeholders. Replace them with your own domain and public IP.

---

## Option 1 — DuckDNS

The simplest path — free, no domain purchase required. Best for personal/homelab use without extra features like proxying or DDoS protection.

1. Go to [duckdns.org](https://www.duckdns.org) and create an account.
2. Map your public IP to a DuckDNS subdomain (e.g. `yourname.duckdns.org`).
3. On your router, port-forward **443** to the internal IP of the server.
4. In the Nextcloud AIO interface, enter the DuckDNS hostname when prompted for a domain.
5. Continue following the AIO interface — it links out to relevant GitHub docs as needed.

Once validated, AIO deploys the rest of the stack (Apache, PostgreSQL, Nextcloud, Redis, etc.) and displays initial admin credentials.

---

## Option 2 — Cloudflare + Apache (built-in HTTPS)

Uses AIO's bundled Apache container for automatic HTTPS via Let's Encrypt, with Cloudflare only handling DNS. No reverse proxy software needed on the host.

### 1. Buy a domain and add it to Cloudflare

In the Cloudflare dashboard: **Domains → Overview → (your domain) → Records**

Add an A record:

| Field | Value |
| :--- | :--- |
| Type | `A` |
| Name | Your desired subdomain, e.g. `cloud` → becomes `cloud.example.com` |
| IPv4 address | Your public IP |
| Proxy status | **Grayed (DNS only)** — temporarily, see note below |

### 2. Disable Cloudflare proxying during setup

Before entering the domain in the AIO interface:
- Set **Proxy status** to grayed/DNS-only on the A record
- Under **SSL/TLS → Overview**, disable **Custom SSL/TLS**

If this isn't disabled, AIO's domain validation fails with:

```
Domain does not point to this server or the reverse proxy is not configured correctly.
See the mastercontainer logs for more details. ('sudo docker logs -f nextcloud-aio-mastercontainer')
If you should be using Cloudflare, make sure to disable the Cloudflare Proxy feature
as it might block the domain validation.
```

You can re-enable the Cloudflare proxy **after** the domain is validated and containers are installed.

### 3. Complete setup in the AIO interface

Enter the domain, validate, and let AIO install the remaining containers.

---

## Option 3 — Cloudflare + Nginx Proxy Manager

The most flexible setup — useful if you're running other services behind the same reverse proxy. Cloudflare handles DNS + proxying, Nginx Proxy Manager (NPM) routes traffic to AIO.

> **Recommended:** start from a clean state. See [removing images/containers](./Docker-Reference.md#images) if a previous AIO instance needs to be fully reset first.

### 1. Adjust the mastercontainer's ports

NPM will own ports 80 and 443 instead of Apache, so comment those out and expose Apache internally instead:

**Before:**
```yaml
ports:
  - "80:80"
  - "8080:8080"
  - "8443:8443"
# environment:
  # APACHE_PORT: 11000
```

**After:**
```yaml
ports:
  #- "80:80"
  - "8080:8080"
  #- "8443:8443"
environment:
  APACHE_PORT: 11000
```

> **Why comment out those ports?** NPM now receives traffic from Cloudflare and forwards it internally — only the AIO interface port (`8080`) stays exposed directly.

### 2. Set up Nginx Proxy Manager

```yaml
services:
  nginx-proxy-manager:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    container_name: nginx-proxy-manager
    ports:
      - '80:80'
      - '443:443'
      - '81:81'   # NPM admin UI
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

> Uncomment `DISABLE_IPV6=true` under `environment` if IPv6 is disabled on the host.

Start both stacks:

```bash
docker compose -f ~/<path>/nginx-compose.yaml up -d
docker compose -f ~/<path>/nextcloud-aio-mastercontainer.yaml up -d
```

### 3. Add the subdomain in Cloudflare

Same as Option 2 — **Domains → Overview → (your domain) → Records → Add A Record**:

| Field | Value |
| :--- | :--- |
| Type | `A` |
| Name | e.g. `cloud` → `cloud.example.com` |
| IPv4 address | Your public IP |
| Proxy status | Grayed (temporarily) |

### 4. Generate an SSL certificate in Cloudflare

> This method uses Cloudflare's own origin certificate — no Let's Encrypt API token needed.

1. Go to **SSL/TLS → Origin Server** and generate a certificate
2. Copy both the **Origin Certificate** and **Private Key**
3. Save each as a separate `.pem` file

### 5. Add the certificate to NPM

Log in to NPM at `http://<server-local-ip>:81`.

- **SSL Certificates → Add SSL Certificate → Custom**
- Upload the private key to **Cert Key**
- Upload the origin certificate to **Cert**

### 6. Add a proxy host in NPM

| Field | Value |
| :--- | :--- |
| Domain name | `cloud.example.com` |
| Scheme | `http` |
| Forward hostname/IP | Server's local IP |
| Forward port | `11000` (matches `APACHE_PORT` set earlier) |
| Options | Check **Block Common Exploits** and **Websocket Support** |

Under the **SSL** tab: select the saved Cloudflare certificate, enable **Force SSL** and **HTTP/2 Support**.

Save.

### 7. Finish setup in the AIO interface

1. Access `http://<server-local-ip>:8080`
2. Log in with the saved passphrase
3. Enter the domain name configured above

From here, domain validation and remaining container installation should proceed normally.

---

## Changing the Domain Later

AIO currently has **no built-in way to change the domain** once set, from the interface itself. Two options:

**1. Edit the configuration JSON directly:**

```bash
sudo docker run -it --rm --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config:rw alpine \
  sh -c "apk add --no-cache nano && nano /mnt/docker-aio-config/data/configuration.json"
```

**2. Full reset (recommended):** Remove all containers and the associated volume, then start over. This removes all user files and data — see [removing images/containers](./Docker-Reference.md#images) for the cleanup commands.

---

**Back to:** [Nextcloud AIO Setup](./Nextcloud-AIO-Setup.md) · [Troubleshooting](./Nextcloud-Troubleshooting.md)
