# Minecraft Server Setup (Docker)

> Fabric server running inside Docker on **Linux Mint 22.3 (Ubuntu 24.04 Noble, x86_64)**.
> Containerized to avoid scattered dependency installs on the host machine.
>
> Requires Docker — see [Docker Setup](./Docker-Setup.md) if not yet installed.

---

## Table of Contents
- [Preparation](#preparation)
- [Install](#install)
- [Configure](#configure)
- [Test](#test)
- [Serving Publicly](#serving-publicly)
  - [Firewall & Router](#firewall--router)
  - [DDNS](#ddns)
  - [Server Properties](#server-properties)
  - [Assigning Operators](#assigning-operators)
- [Voice Chat (Server Side)](#voice-chat-server-side)
- [Backing Up the Server](#backing-up-the-server)
  - [SCP Method](#scp-method)
  - [RSYNC Method](#rsync-method)

---

## Preparation

Verify Docker is installed and running:

```bash
docker --version
docker compose version
```

Check available system resources:

```bash
free -h
df -h
```

Create a directory for the server files:

```bash
mkdir -p ~/minecraft-server
cd ~/minecraft-server
```

---

## Install

This setup uses the [`itzg/minecraft-server`](https://github.com/itzg/docker-minecraft-server) image — the most popular and well-maintained Minecraft Docker image available.

Create the compose file:

```bash
nano ~/minecraft-server/docker-compose.yml
```

```yaml
services:
  minecraft:
    image: itzg/minecraft-server
    container_name: minecraft-server
    ports:
      - "25565:25565"
      - "24454:24454/udp"   # Voice chat
    environment:
      EULA: "TRUE"
      MEMORY: "6G"
      TYPE: "FABRIC"
      CREATE_CONSOLE_IN_PIPE: "true"
    volumes:
      - ./data:/data
    restart: unless-stopped
```

**`TYPE` options:** `VANILLA`, `FABRIC`, `FORGE`, `PAPER`, `SPIGOT`, `BUKKIT`, `PURPUR`, `FOLIA`, `QUILT`, `SPONGEVANILLA`

> **Port mapping format:** `"<host>:<container>"` — left side is your machine, right side is inside the container.

---

## Configure

Key environment variables to adjust in the compose file:

| Variable | Purpose |
| :--- | :--- |
| `MEMORY` | RAM allocated to the server (e.g. `6G`) |
| `TYPE` | Server type — use `PAPER` or `SPIGOT` for plugin support |
| `EULA` | Must be `TRUE` to accept Minecraft's EULA |
| `CREATE_CONSOLE_IN_PIPE` | Enables sending in-game commands via `docker exec` |

**Persistent data:** Everything — world files, logs, configs, plugins — lives in `./data`. Back up this folder to save the server state.

---

## Test

Start the server:

```bash
docker compose up -d
```

Follow the logs to watch it initialize:

```bash
docker logs -f minecraft-server
```

Connect from Minecraft using your local IP. Once it loads, the server is working.

> **Tips:**
> - Use `PAPER` or `SPIGOT` as the type if you plan to add plugins
> - Allocate more RAM via `MEMORY` if the server stutters under load
> - `JAVA_OPTS` can be added to the environment block for advanced JVM tuning

---

## Serving Publicly

### Firewall & Router

Find your public IP:

```bash
curl ifconfig.me
```

Open the Minecraft port on the firewall:

```bash
sudo ufw allow 25565/tcp
sudo ufw status numbered
```

On your router, set up **port forwarding** — forward external port `25565` to the internal IP of the host machine, port `25565`.

> **Optional — security tools:**
> - `ufw` — firewall (already used above)
> - `Fail2ban` — useful if the machine runs other services with login pages

---

### DDNS

If your ISP assigns a **dynamic public IP** (changes periodically), use a Dynamic DNS service to attach a fixed hostname to it:

- [DuckDNS](https://www.duckdns.org) — free
- [No-IP](https://www.noip.com) — free tier available

This way players connect via a hostname instead of an IP that can change.

---

### Server Properties

After first launch, `./data/server.properties` is generated. Key values to review:

```properties
online-mode=true
server-port=25565
max-players=20
```

---

### Assigning Operators

Edit `./data/ops.json` to assign operator permissions:

```json
[
  {
    "uuid": "<player-uuid>",
    "name": "<player-name>",
    "level": 4,
    "bypassPlayerLimit": true
  }
]
```

| Field | Notes |
| :--- | :--- |
| `level` | `1` (lowest) to `4` (full operator access) |
| `bypassPlayerLimit` | If `true`, this player can join even when the server is full |

---

## Voice Chat (Server Side)

This uses [Simple Voice Chat](https://modrinth.com/plugin/simple-voice-chat) — requires a Fabric server.

1. Download both mods from Modrinth, matching your server's Minecraft version and **Fabric** platform:
   - [Simple Voice Chat](https://modrinth.com/plugin/simple-voice-chat)
   - [Fabric API](https://modrinth.com/mod/fabric-api)

2. Move both `.jar` files into `./data/mods/`

3. Restart the server to generate the voice chat config:

   ```bash
   docker restart minecraft-server
   ```

4. Navigate to `./data/config/voicechat/` and adjust settings (e.g. port number) as needed.

5. Forward the voice chat port on your router — same process as `25565`, but for port `24454` (UDP).

> For the client-side setup, see [Minecraft Client Setup](./Minecraft-Client-Setup.md).

---

## Backing Up the Server

Both methods below back up from a **Linux server** to a **Windows local machine**.

### SCP Method

**1. Flush and save world progress:**

```bash
docker exec -u 1000 -it minecraft-server rcon-cli "save-all flush"
```

**2. Stop the server gracefully:**

```bash
docker exec -u 1000 -it minecraft-server rcon-cli "stop"
docker stop minecraft-server
```

**3. From the Windows client machine, copy the data folder:**

```bash
scp -P 2200 -r <username>@<server-ip>:~/minecraft-server/data C:\Users\<you>\destination
```

**4. Start the container again:**

```bash
docker start minecraft-server
```

**5. Re-enable auto-saving:**

```bash
docker exec -u 1000 -it minecraft-server rcon-cli "save-on"
```

---

### RSYNC Method

*Requires WSL on the Windows machine.*

**1. Copy your SSH private key into the WSL filesystem:**

```bash
# Run from inside WSL
cp /mnt/c/Users/<username>/.ssh/id_ed25519 ~/.ssh/
chmod 600 ~/.ssh/id_ed25519
```

> By default, the private key on Windows lives at `C:\Users\<username>\.ssh\id_ed25519`.

**2. Run the rsync command from WSL:**

```bash
rsync -avz --progress -e "ssh -p <port> -i ~/.ssh/id_ed25519" \
  <username>@<server-ip>:~/minecraft-server/data \
  /mnt/c/Users/<you>/destination
```

**Troubleshooting connection issues:**

If the connection fails despite the key being accepted, run with verbose output:

```bash
rsync -avz --progress -e "ssh -p <port> -i ~/.ssh/id_ed25519 -vvv" ...
```

Also try SSHing directly from the host machine's terminal first, then switch to the WSL session afterward to confirm connectivity.
