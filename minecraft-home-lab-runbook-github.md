# Minecraft Lab Server Build and Operations Guide - GitHub Version

> **Status:** GitHub-friendly sanitized reference based on a verified working configuration  
> **Host:** Dell OptiPlex 5050  
> **Operating System:** Ubuntu 26.04 LTS  
> **Minecraft Editions:** Java and Bedrock  
> **Remote Access:** Playit.gg  
> **Container Runtime:** Docker  
> **Compose:** Standalone Docker Compose v5.3.1

---

## Table of Contents

1. [Purpose](#purpose)
2. [Architecture](#architecture)
3. [Reference Host Configuration](#reference-host-configuration)
4. [Verified Software Versions](#verified-software-versions)
5. [Network Design](#network-design)
6. [Software and Documentation URLs](#software-and-documentation-urls)
7. [Fresh Ubuntu Build](#fresh-ubuntu-build)
8. [Install Docker](#install-docker)
9. [Install Standalone Docker Compose](#install-standalone-docker-compose)
10. [Create the Minecraft Project](#create-the-minecraft-project)
11. [Docker Compose Configuration](#docker-compose-configuration)
12. [Start and Validate Minecraft](#start-and-validate-minecraft)
13. [Java Server Configuration](#java-server-configuration)
14. [Bedrock Server Configuration](#bedrock-server-configuration)
15. [Persistent Storage](#persistent-storage)
16. [Bedrock File Ownership and Permissions](#bedrock-file-ownership-and-permissions)
17. [Playit.gg Installation](#playitgg-installation)
18. [Playit.gg Tunnel Configuration](#playitgg-tunnel-configuration)
19. [Client Connection Information](#client-connection-information)
20. [Day-to-Day Operations](#day-to-day-operations)
21. [Managing Bedrock Allow-List Users](#managing-bedrock-allow-list-users)
22. [Managing Java Whitelist Users](#managing-java-whitelist-users)
23. [Day 2 Backup Design](#day-2-backup-design)
24. [Backup Script](#backup-script)
25. [Schedule Nightly Backups](#schedule-nightly-backups)
26. [Restore Procedure](#restore-procedure)
27. [Updates](#updates)
28. [Security Notes](#security-notes)
29. [Optional UFW Configuration](#optional-ufw-configuration)
30. [Router Verification](#router-verification)
31. [Troubleshooting](#troubleshooting)
32. [Reference Commands](#reference-commands)
33. [Known Follow-Up Actions](#known-follow-up-actions)

---

# Purpose

This GitHub-friendly document describes a reproducible Minecraft lab environment based on a verified working deployment running on a Dell OptiPlex 5050 with Ubuntu 26.04 LTS.

The environment provides two independent Minecraft servers:

- Minecraft Java Edition
- Minecraft Bedrock Edition

The servers run as Docker containers and intentionally use separate persistent worlds. Local players connect directly across the lab network. Remote players connect through Playit.gg tunnels.

No router port forwarding was intentionally configured. Remote Minecraft traffic is expected to traverse Playit.gg.

> **GitHub sanitization note:** The hostname, lab IP addresses, router/gateway name, local administrative account, player names, and Playit tunnel hostnames in this document are public-safe examples. Replace them with values appropriate for your own environment.

This guide includes:

- Verified current-state configuration
- Fresh installation steps
- Docker and Docker Compose setup
- Minecraft Java and Bedrock deployment
- Playit.gg installation and tunnels
- Local and remote connection details
- Daily operating procedures
- Backup and restore procedures
- Update and rollback guidance
- Security notes
- Troubleshooting procedures

---

# Architecture

## Java Edition Traffic Flow

```text
Remote Java Player
        |
        | Minecraft Java / TCP
        v
java-example.tun.ply.gg
        |
        v
Playit.gg
        |
        v
Playit Agent on Ubuntu
        |
        | 127.0.0.1:25565/TCP
        v
Docker Host Port 25565
        |
        v
mc-java Container
        |
        v
minecraft_java-data Volume
```

## Bedrock Edition Traffic Flow

```text
Remote Bedrock Player
        |
        | Minecraft Bedrock / UDP
        v
bedrock-example.tun.ply.gg:40041
        |
        v
Playit.gg
        |
        v
Playit Agent on Ubuntu
        |
        | 127.0.0.1:19132/UDP
        v
Docker Host Port 19132
        |
        v
mc-bedrock Container
        |
        v
minecraft_bedrock-data Volume
```

## Local Access

Local players bypass Playit.gg and connect directly to the Ubuntu server:

```text
Java:    10.1.1.116:25565
Bedrock: 10.1.1.116:19132
```

---

# Reference Host Configuration

| Item | Verified Value |
|---|---|
| Hardware | Dell OptiPlex 5050 |
| Installation type | Physical system |
| Hostname | `minecraft-OptiPlex-5050` |
| Operating system | Ubuntu 26.04 LTS |
| Ubuntu codename | `resolute` |
| Kernel | `7.0.0-28-generic` |
| Architecture | `x86-64` |
| Processor | Intel Core i5-6600T @ 2.70 GHz |
| CPU | 4 cores, 4 threads |
| Memory | 38 GiB |
| Swap | 8 GiB |
| Storage | 465.8 GB |
| Root filesystem | ext4 |
| Primary interface | `enp0s31f6` |
| Private IP | `10.1.1.116/24` |
| Default gateway | `10.1.1.1` |
| Address assignment | DHCP reservation on lab router/gateway |
| Wi-Fi | Disabled |

The DHCP reservation is important because this host provides stable local endpoints for Minecraft and Playit.gg.

---

# Verified Software Versions

| Software | Verified Version / Source |
|---|---|
| Docker Engine | `29.1.3-0ubuntu4.1` |
| Docker package | Ubuntu `docker.io` |
| containerd | `2.2.2-0ubuntu1.1` |
| Docker Compose | Standalone `v5.3.1` |
| Compose executable | `/usr/local/bin/docker-compose` |
| Playit package | `1.0.9-1` |
| Playit CLI | `/usr/bin/playit` |
| Playit daemon | `/opt/playit/playitd` |
| Java image | `itzg/minecraft-server` |
| Bedrock image | `itzg/minecraft-bedrock-server` |
| Java runtime | OpenJDK `25.0.3+9` |
| Java Minecraft version | `26.2` |
| Bedrock version | `1.26.40.8` |

## Important Docker Compose Command Difference

This system uses the standalone executable:

```bash
sudo docker-compose
```

It does **not** currently use:

```bash
sudo docker compose
```

Use `docker-compose` consistently unless the host is deliberately migrated to the Docker Compose plugin.

---

# Network Design

## Home Network

```text
Network: 10.1.1.0/24
Gateway: 10.1.1.1
Server:  10.1.1.116
```

## Docker Network

```text
Network name: minecraft_default
Driver:       bridge
Subnet:       172.20.0.0/16
Gateway:      172.20.0.1
IPv6:         disabled
```

## Container Addresses

| Container | Docker Address | Published Host Port |
|---|---|---|
| `mc-java` | `172.20.0.2` | TCP `25565` |
| `mc-bedrock` | `172.20.0.3` | UDP `19132` |

Docker container addresses are dynamic implementation details. Do not configure players or Playit.gg to use the `172.20.0.x` container addresses.

## Published Ports

```text
0.0.0.0:25565/TCP -> mc-java:25565
[::]:25565/TCP    -> mc-java:25565

0.0.0.0:19132/UDP -> mc-bedrock:19132
[::]:19132/UDP    -> mc-bedrock:19132
```

## Firewall State

UFW is currently inactive:

```text
Status: inactive
```

This document preserves that as the current verified state. Optional firewall hardening is documented later.

---

# Software and Documentation URLs

## Ubuntu

- https://ubuntu.com/download
- https://packages.ubuntu.com/

## Docker

- https://docs.docker.com/engine/
- https://docs.docker.com/compose/
- https://docs.docker.com/compose/install/standalone/
- https://github.com/docker/compose/releases

## Minecraft Container Images

- https://docker-minecraft-server.readthedocs.io/
- https://github.com/itzg/docker-minecraft-server
- https://github.com/itzg/docker-minecraft-bedrock-server
- https://hub.docker.com/r/itzg/minecraft-server
- https://hub.docker.com/r/itzg/minecraft-bedrock-server

## Playit.gg

- https://packages.playit.gg/
- https://playit.gg/download
- https://playit.gg/support/
- https://playit.gg/account/tunnels

## Minecraft EULA

- https://www.minecraft.net/eula

---

# Fresh Ubuntu Build

The verified host uses Ubuntu 26.04 LTS.

After installing Ubuntu, update the system:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

After rebooting, verify the host:

```bash
hostnamectl
cat /etc/os-release
ip -br address
ip route
free -h
lsblk
```

Reserve the server IP address in the router so the host remains at:

```text
10.1.1.116
```

Set a generic server hostname if desired:

```bash
sudo hostnamectl set-hostname minecraft-OptiPlex-5050
```

This guide uses a dedicated host administration account named `minecraftadmin` for the GitHub example. Create it before creating `/srv/minecraft`:

```bash
sudo adduser minecraftadmin
sudo usermod -aG docker minecraftadmin
```

Log out and back in before relying on Docker group membership. If all Docker commands are run with `sudo`, Docker group membership is optional.

> **Security note:** Membership in the Docker group grants root-equivalent control of the host. Only trusted administrators should be members.

---

# Install Docker

Install required packages:

```bash
sudo apt update
sudo apt install -y \
  ca-certificates \
  curl \
  wget \
  jq \
  nano \
  vim \
  tar \
  gzip \
  rsync \
  docker.io
```

Enable Docker:

```bash
sudo systemctl enable --now docker
```

Verify:

```bash
sudo docker version
sudo systemctl status docker --no-pager
```

The verified system uses Ubuntu's `docker.io` package, not Docker CE.

---

# Install Standalone Docker Compose

The verified host uses:

```text
/usr/local/bin/docker-compose
Docker Compose v5.3.1
```

Check an existing installation:

```bash
command -v docker-compose
sudo docker-compose version
ls -l /usr/local/bin/docker-compose
file /usr/local/bin/docker-compose
```

For a new installation, follow:

https://docs.docker.com/compose/install/standalone/

Select the required release and architecture from:

https://github.com/docker/compose/releases

After downloading the appropriate x86-64 binary, install it:

```bash
sudo install -o root -g root -m 0755 docker-compose-linux-x86_64 \
  /usr/local/bin/docker-compose
```

Verify:

```bash
sudo docker-compose version
```

---

# Create the Minecraft Project

Create the project directory:

```bash
sudo mkdir -p /srv/minecraft
sudo chown minecraftadmin:minecraftadmin /srv/minecraft
cd /srv/minecraft
```

The verified Compose file is:

```text
/srv/minecraft/compose.yaml
```

---

# Docker Compose Configuration

Create the file:

```bash
nano /srv/minecraft/compose.yaml
```

Current working configuration:

```yaml
services:
  # Java Edition Server
  minecraft-java:
    image: itzg/minecraft-server
    container_name: mc-java
    dns:
      - 8.8.8.8
      - 8.8.4.4
    ports:
      - "25565:25565"
    environment:
      EULA: "TRUE"
      MEMORY: "3G"
      MODE: "creative"
    volumes:
      - java-data:/data
    restart: unless-stopped

  # Bedrock Edition Server
  minecraft-bedrock:
    image: itzg/minecraft-bedrock-server
    container_name: mc-bedrock
    ports:
      - "19132:19132/udp"
    environment:
      EULA: "TRUE"
      GAMEMODE: "creative"
      ALLOW_LIST: "TRUE"
      ALLOW_LIST_USERS: "PlayerOne,PlayerTwo,PlayerThree"
    volumes:
      - bedrock-data:/data
    restart: unless-stopped

volumes:
  java-data:
  bedrock-data:
```

## Player Name Customization

Replace the Bedrock player list with the exact Microsoft/Xbox account names that should be permitted to connect.

Example:

```yaml
ALLOW_LIST_USERS: "PlayerOne,PlayerTwo,PlayerThree"
```

Edit:

```bash
cd /srv/minecraft
nano compose.yaml
```

Validate:

```bash
sudo docker-compose config
```

Apply only the Bedrock service change:

```bash
sudo docker-compose up -d minecraft-bedrock
```

Verify the generated allow list:

```bash
sudo docker exec mc-bedrock cat /data/allowlist.json
```

Do not store passwords, Playit secrets, RCON passwords, or other credentials in the Compose file.

---

# Start and Validate Minecraft

Validate the Compose configuration:

```bash
cd /srv/minecraft
sudo docker-compose config
```

Start the project:

```bash
sudo docker-compose up -d
```

Check status:

```bash
sudo docker-compose ps
```

Expected containers:

```text
mc-java
mc-bedrock
```

View logs:

```bash
sudo docker logs --tail=100 mc-java
sudo docker logs --tail=100 mc-bedrock
```

Follow logs live:

```bash
sudo docker logs -f mc-java
```

```bash
sudo docker logs -f mc-bedrock
```

Press `Ctrl+C` to stop following logs. The containers continue running.

---

# Java Server Configuration

| Setting | Verified Value |
|---|---|
| Compose service | `minecraft-java` |
| Container | `mc-java` |
| Image | `itzg/minecraft-server` |
| Server type | Vanilla |
| Version policy | `LATEST` |
| Current Minecraft version | `26.2` |
| Java version | OpenJDK `25.0.3+9` |
| Initial heap | 3 GB |
| Maximum heap | 3 GB |
| Game mode | Creative |
| Difficulty | Easy |
| World | `world` |
| Maximum players | 20 |
| Online mode | Enabled |
| Whitelist | Disabled |
| View distance | 10 |
| Simulation distance | 10 |
| Pause when empty | 60 seconds |
| RCON | Enabled internally |
| Published RCON port | None |

The Java server processes run under UID/GID 1000.

Verify:

```bash
sudo docker exec mc-java ps -eo user,uid,gid,pid,ppid,cmd
```

Expected server process pattern:

```text
UID 1000 / GID 1000
java -Xmx3G -Xms3G -jar minecraft_server.26.2.jar
```

## Java Data

```text
Docker volume:  minecraft_java-data
Host path:      /var/lib/docker/volumes/minecraft_java-data/_data
Container path: /data
World path:     /data/world
```

Important files:

```text
/data/server.properties
/data/eula.txt
/data/ops.json
/data/whitelist.json
/data/banned-ips.json
/data/banned-players.json
/data/logs/
/data/world/
```

Do not publish:

```text
/data/.rcon-cli.env
/data/.rcon-cli.yaml
```

Those files may contain credentials.

---

# Bedrock Server Configuration

| Setting | Verified Value |
|---|---|
| Compose service | `minecraft-bedrock` |
| Container | `mc-bedrock` |
| Image | `itzg/minecraft-bedrock-server` |
| Version policy | `LATEST` |
| Current version | `1.26.40.8` |
| Game mode | Creative |
| Difficulty | Easy |
| World | `Bedrock level` |
| Maximum players | 10 |
| Online mode | Enabled |
| Allow list | Enabled |
| Cheats | Disabled |
| IPv4 port | UDP 19132 |
| IPv6 server port | UDP 19133, not published |
| LAN visibility | Enabled |
| View distance | 32 |
| Tick distance | 4 |
| Player idle timeout | 30 minutes |
| Maximum threads | 8 |
| Default player permission | Member |

## Verify Bedrock Runtime User

A normal `docker exec` command currently starts as root:

```bash
sudo docker exec mc-bedrock id
```

Previously observed result:

```text
uid=0(root) gid=0(root) groups=0(root)
```

This does not automatically prove that the Bedrock server process itself runs as root.

Verify the actual server process:

```bash
sudo docker top mc-bedrock -eo user,uid,gid,pid,ppid,cmd
```

If available inside the container:

```bash
sudo docker exec mc-bedrock ps -eo user,uid,gid,pid,ppid,cmd
```

Document the actual Bedrock server process UID/GID after verification.

---

# Persistent Storage

## Java

```text
Volume:         minecraft_java-data
Host path:      /var/lib/docker/volumes/minecraft_java-data/_data
Container path: /data
World:          /data/world
```

## Bedrock

```text
Volume:         minecraft_bedrock-data
Host path:      /var/lib/docker/volumes/minecraft_bedrock-data/_data
Container path: /data
World:          /data/worlds/Bedrock level
```

Important Bedrock files:

```text
/data/server.properties
/data/allowlist.json
/data/permissions.json
/data/worlds/
/data/behavior_packs/
/data/resource_packs/
```

---

# Bedrock File Ownership and Permissions

The Bedrock persistent volume was originally owned by `root:root`.

Ownership was intentionally changed to `minecraftadmin:minecraftadmin` with:

```bash
sudo chown -R minecraftadmin:minecraftadmin \
  /var/lib/docker/volumes/minecraft_bedrock-data/_data
```

The expected ownership is now:

```text
Owner: minecraftadmin
Group: minecraftadmin
```

Verify the top-level directory:

```bash
sudo ls -ld \
  /var/lib/docker/volumes/minecraft_bedrock-data/_data
```

Verify contents:

```bash
sudo ls -lah \
  /var/lib/docker/volumes/minecraft_bedrock-data/_data
```

Detailed verification:

```bash
sudo find \
  /var/lib/docker/volumes/minecraft_bedrock-data/_data \
  -maxdepth 2 \
  -printf '%M %u:%g %p\n' \
  | sort
```

Verify the mount inside the container:

```bash
sudo docker inspect mc-bedrock \
  --format '{{range .Mounts}}{{println "Type:" .Type}}{{println "Name:" .Name}}{{println "Source:" .Source}}{{println "Destination:" .Destination}}{{println "RW:" .RW}}{{end}}'
```

Expected mount:

```text
Type: volume
Name: minecraft_bedrock-data
Source: /var/lib/docker/volumes/minecraft_bedrock-data/_data
Destination: /data
RW: true
```

Check the container view:

```bash
sudo docker exec mc-bedrock ls -ld /data
sudo docker exec mc-bedrock ls -lah /data
```

After changing ownership, verify Bedrock remains healthy and can write to its world:

```bash
sudo docker ps --filter name=mc-bedrock
sudo docker logs --tail=100 mc-bedrock
```

Verify the world directory:

```bash
sudo ls -ld \
  "/var/lib/docker/volumes/minecraft_bedrock-data/_data/worlds/Bedrock level"
```

Do not perform another recursive ownership change unless a specific container or host-side permission requirement has been identified.

---

# Playit.gg Installation

The verified host uses the Playit APT package.

Official package page:

https://packages.playit.gg/

Install using the official installer:

```bash
curl -fsSL https://packages.playit.gg/install.sh | bash
```

Review remote scripts before execution.

Verify:

```bash
playit version
playit status
```

Verified installation paths:

```text
CLI:          /usr/bin/playit
CLI target:   /opt/playit/playit
Daemon:       /opt/playit/playitd
Secret:       /etc/playit/playit.toml
IPC socket:   /run/playit/playitd.sock
Log:          /var/log/playit/playit.log
Systemd unit: /usr/lib/systemd/system/playit.service
```

Never display or commit the contents of:

```text
/etc/playit/playit.toml
```

## Claim a New Agent

Use:

```bash
sudo playit setup
```

or:

```bash
playit claim
```

Follow the browser URL supplied by Playit.

Verify:

```bash
playit status
```

Expected state:

```text
Secret configured: true
Phase: running
```

Do not store claim codes or agent secrets in GitHub.

---

# Playit.gg Tunnel Configuration

Open:

https://playit.gg/account/tunnels

## Java Tunnel

```text
Public hostname: java-example.tun.ply.gg
Public port:     Not required
Tunnel type:     Minecraft Java / TCP
Local address:   127.0.0.1
Local port:      25565
```

## Bedrock Tunnel

```text
Public hostname: bedrock-example.tun.ply.gg
Public port:     40041
Tunnel type:     Minecraft Bedrock / UDP
Local address:   127.0.0.1
Local port:      19132
```

The loopback address is correct because Playit and Docker run on the same Ubuntu host.

## Playit Service

Enable and start:

```bash
sudo systemctl enable --now playit
```

Check status:

```bash
playit status
sudo systemctl status playit --no-pager
```

If systemd reports that the service file changed on disk:

```bash
sudo systemctl daemon-reload
sudo systemctl restart playit
playit status
```

Do not run:

```bash
playit reset
```

unless the intention is to remove the current agent secret and claim the agent again.

---

# Client Connection Information

## Java Local

```text
Server: 10.1.1.116
Port:   25565
```

Players can normally enter:

```text
10.1.1.116
```

## Java Remote

```text
java-example.tun.ply.gg
```

No public port is required.

## Bedrock Local

```text
Server: 10.1.1.116
Port:   19132
```

## Bedrock Remote

```text
Server: bedrock-example.tun.ply.gg
Port:   40041
```

## Separate Worlds

Java and Bedrock intentionally use separate worlds.

```text
Java:    /data/world
Bedrock: /data/worlds/Bedrock level
```

No Geyser, Floodgate, or shared cross-platform world configuration is currently in use.

---

# Day-to-Day Operations

## Check Overall Health

```bash
cd /srv/minecraft
sudo docker-compose ps
playit status
sudo ss -lntup | grep -E '25565|19132'
```

## Start Minecraft

```bash
sudo systemctl start docker
sudo systemctl start playit

cd /srv/minecraft
sudo docker-compose up -d
```

## Stop Minecraft

```bash
cd /srv/minecraft
sudo docker-compose stop
```

Stop Java only:

```bash
sudo docker-compose stop minecraft-java
```

Stop Bedrock only:

```bash
sudo docker-compose stop minecraft-bedrock
```

## Restart Minecraft

```bash
cd /srv/minecraft
sudo docker-compose restart
```

Restart Playit:

```bash
sudo systemctl restart playit
playit status
```

## Logs

Java:

```bash
sudo docker logs --tail=100 mc-java
```

Bedrock:

```bash
sudo docker logs --tail=100 mc-bedrock
```

Playit:

```bash
sudo journalctl -u playit -n 100 --no-pager
```

File-based Playit log:

```bash
sudo tail -n 100 /var/log/playit/playit.log
```

Live logs:

```bash
sudo docker logs -f mc-java
sudo docker logs -f mc-bedrock
sudo journalctl -u playit -f
```

---

# Managing Bedrock Allow-List Users

The Bedrock allow list is managed through the Compose environment variable:

```yaml
ALLOW_LIST: "TRUE"
ALLOW_LIST_USERS: "PlayerOne,PlayerTwo,PlayerThree"
```

Edit:

```bash
cd /srv/minecraft
nano compose.yaml
```

Validate:

```bash
sudo docker-compose config
```

Apply:

```bash
sudo docker-compose up -d minecraft-bedrock
```

Verify:

```bash
sudo docker exec mc-bedrock cat /data/allowlist.json
```

Do not manually maintain `/data/allowlist.json` while simultaneously managing the list through `ALLOW_LIST_USERS`, because the container may regenerate the file.

---

# Managing Java Whitelist Users

The Java whitelist is currently disabled.

To enable it through Compose, add:

```yaml
environment:
  WHITELIST: "PlayerOne,PlayerTwo,PlayerThree"
  ENFORCE_WHITELIST: "TRUE"
```

Apply:

```bash
cd /srv/minecraft
sudo docker-compose config
sudo docker-compose up -d minecraft-java
```

Verify:

```bash
sudo grep -E '^(white-list|enforce-whitelist)=' \
  /var/lib/docker/volumes/minecraft_java-data/_data/server.properties
```

Verify generated whitelist:

```bash
sudo docker exec mc-java cat /data/whitelist.json
```

A Java whitelist is recommended if the public Playit hostname is shared beyond trusted players.

---

# Day 2 Backup Design

No backup process was originally implemented.

The backup solution should protect:

```text
minecraft_java-data
minecraft_bedrock-data
/srv/minecraft/compose.yaml
```

The procedure below deliberately stops both Minecraft containers before archiving the volumes. This creates a short outage but provides a consistent backup of the world files.

Backups should eventually be copied to storage outside the Minecraft host.

---

# Backup Script

Create the backup directory:

```bash
sudo mkdir -p /var/backups/minecraft
sudo chown root:root /var/backups/minecraft
sudo chmod 750 /var/backups/minecraft
```

Create the script:

```bash
sudo nano /usr/local/sbin/backup-minecraft.sh
```

Script:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

PROJECT_DIR="/srv/minecraft"
BACKUP_DIR="/var/backups/minecraft"
RETENTION_DAYS=14
TIMESTAMP="$(date '+%Y%m%d-%H%M%S')"
BACKUP_SET="${BACKUP_DIR}/${TIMESTAMP}"

JAVA_VOLUME="minecraft_java-data"
BEDROCK_VOLUME="minecraft_bedrock-data"

log() {
  printf '%s %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*"
}

restart_minecraft() {
  log "Starting Minecraft containers."
  cd "${PROJECT_DIR}"
  /usr/local/bin/docker-compose up -d
}

mkdir -p "${BACKUP_SET}"

log "Validating Docker Compose configuration."
cd "${PROJECT_DIR}"
/usr/local/bin/docker-compose config >/dev/null

log "Stopping Minecraft containers cleanly."
/usr/local/bin/docker-compose stop

trap restart_minecraft EXIT

log "Backing up Java volume."
docker run --rm \
  -v "${JAVA_VOLUME}:/source:ro" \
  -v "${BACKUP_SET}:/backup" \
  alpine:latest \
  tar -czf /backup/minecraft-java.tar.gz -C /source .

log "Backing up Bedrock volume."
docker run --rm \
  -v "${BEDROCK_VOLUME}:/source:ro" \
  -v "${BACKUP_SET}:/backup" \
  alpine:latest \
  tar -czf /backup/minecraft-bedrock.tar.gz -C /source .

log "Backing up Compose configuration."
install -o root -g root -m 0640 \
  "${PROJECT_DIR}/compose.yaml" \
  "${BACKUP_SET}/compose.yaml"

log "Writing metadata."
{
  echo "created=$(date --iso-8601=seconds)"
  echo "host=$(hostname)"
  echo "java_volume=${JAVA_VOLUME}"
  echo "bedrock_volume=${BEDROCK_VOLUME}"
  docker version --format 'docker_server={{.Server.Version}}'
  /usr/local/bin/docker-compose version
  docker inspect mc-java --format 'java_image={{.Config.Image}} java_image_id={{.Image}}'
  docker inspect mc-bedrock --format 'bedrock_image={{.Config.Image}} bedrock_image_id={{.Image}}'
} > "${BACKUP_SET}/metadata.txt"

log "Creating checksums."
(
  cd "${BACKUP_SET}"
  sha256sum \
    minecraft-java.tar.gz \
    minecraft-bedrock.tar.gz \
    compose.yaml \
    metadata.txt \
    > SHA256SUMS
)

log "Removing local backup sets older than ${RETENTION_DAYS} days."
find "${BACKUP_DIR}" \
  -mindepth 1 \
  -maxdepth 1 \
  -type d \
  -mtime "+${RETENTION_DAYS}" \
  -exec rm -rf -- {} +

log "Backup completed: ${BACKUP_SET}"
```

Secure the script:

```bash
sudo chown root:root /usr/local/sbin/backup-minecraft.sh
sudo chmod 750 /usr/local/sbin/backup-minecraft.sh
```

Test:

```bash
sudo /usr/local/sbin/backup-minecraft.sh
```

Verify:

```bash
sudo find /var/backups/minecraft -maxdepth 2 -type f -ls
sudo docker-compose -f /srv/minecraft/compose.yaml ps
```

---

# Schedule Nightly Backups

Create the service:

```bash
sudo nano /etc/systemd/system/minecraft-backup.service
```

```ini
[Unit]
Description=Back Up Minecraft Docker Volumes
Requires=docker.service
After=docker.service
ConditionPathExists=/srv/minecraft/compose.yaml

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/backup-minecraft.sh
```

Create the timer:

```bash
sudo nano /etc/systemd/system/minecraft-backup.timer
```

```ini
[Unit]
Description=Nightly Minecraft Backup

[Timer]
OnCalendar=*-*-* 04:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now minecraft-backup.timer
```

Verify:

```bash
systemctl list-timers minecraft-backup.timer
sudo systemctl status minecraft-backup.timer --no-pager
```

Test the backup service:

```bash
sudo systemctl start minecraft-backup.service
sudo journalctl -u minecraft-backup.service -n 100 --no-pager
```

---

# Restore Procedure

> Restoring replaces current world data. Create an additional backup first whenever the current volumes remain readable.

Select a backup:

```bash
BACKUP_SET="/var/backups/minecraft/YYYYMMDD-HHMMSS"
```

Validate checksums:

```bash
cd "${BACKUP_SET}"
sudo sha256sum -c SHA256SUMS
```

Stop the project:

```bash
cd /srv/minecraft
sudo docker-compose down
```

## Restore Java

```bash
sudo docker volume rm minecraft_java-data
sudo docker volume create minecraft_java-data
```

Extract:

```bash
sudo docker run --rm \
  -v minecraft_java-data:/restore \
  -v "${BACKUP_SET}:/backup:ro" \
  alpine:latest \
  sh -c 'cd /restore && tar -xzf /backup/minecraft-java.tar.gz'
```

## Restore Bedrock

```bash
sudo docker volume rm minecraft_bedrock-data
sudo docker volume create minecraft_bedrock-data
```

Extract:

```bash
sudo docker run --rm \
  -v minecraft_bedrock-data:/restore \
  -v "${BACKUP_SET}:/backup:ro" \
  alpine:latest \
  sh -c 'cd /restore && tar -xzf /backup/minecraft-bedrock.tar.gz'
```

Because Bedrock data is expected to be host-owned by `minecraftadmin:minecraftadmin`, reapply ownership after restoring:

```bash
sudo chown -R minecraftadmin:minecraftadmin \
  /var/lib/docker/volumes/minecraft_bedrock-data/_data
```

Restore the Compose file when required:

```bash
sudo install -o minecraftadmin -g minecraftadmin -m 0664 \
  "${BACKUP_SET}/compose.yaml" \
  /srv/minecraft/compose.yaml
```

Validate and start:

```bash
cd /srv/minecraft
sudo docker-compose config
sudo docker-compose up -d
```

Verify:

```bash
sudo docker-compose ps
sudo docker logs --tail=100 mc-java
sudo docker logs --tail=100 mc-bedrock
playit status
```

Test local and remote connections for both editions.

## Destructive Command Warning

Do not run this casually:

```bash
sudo docker-compose down --volumes
```

The `--volumes` option deletes the named volumes and can permanently remove both worlds.

Normal project shutdown:

```bash
sudo docker-compose down
```

---

# Updates

## Ubuntu

```bash
sudo apt update
sudo apt full-upgrade -y
```

Before rebooting:

```bash
sudo /usr/local/sbin/backup-minecraft.sh
sudo reboot
```

After reboot:

```bash
sudo systemctl status docker --no-pager
sudo systemctl status playit --no-pager
cd /srv/minecraft
sudo docker-compose ps
```

## Playit

```bash
sudo apt update
sudo apt install --only-upgrade playit
playit version
playit status
```

## Minecraft Images

Back up first:

```bash
sudo /usr/local/sbin/backup-minecraft.sh
```

Update:

```bash
cd /srv/minecraft
sudo docker-compose pull
sudo docker-compose up -d
```

Verify:

```bash
sudo docker-compose ps
sudo docker logs --tail=100 mc-java
sudo docker logs --tail=100 mc-bedrock
```

## Version Policy

Both services currently resolve:

```text
VERSION=LATEST
```

Current results:

- Java: `26.2`
- Bedrock: `1.26.40.8`

For controlled upgrades, explicitly pin versions in `compose.yaml`.

Java example:

```yaml
VERSION: "26.2"
```

Bedrock example:

```yaml
VERSION: "1.26.40.8"
```

Always create a verified world backup before changing Minecraft versions.

---

# Security Notes

## Secrets

Never commit or publish:

```text
/etc/playit/playit.toml contents
RCON password
Minecraft management-server secret
Playit claim codes
Playit agent secret
Microsoft account credentials
```

The RCON password and management server secret that were previously exposed during troubleshooting should be rotated.

Do not include the replacement values in GitHub.

## Rotate Java Generated Secrets

Stop Java:

```bash
cd /srv/minecraft
sudo docker-compose stop minecraft-java
```

Remove existing generated entries:

```bash
sudo sed -i '/^rcon.password=/d' \
  /var/lib/docker/volumes/minecraft_java-data/_data/server.properties

sudo sed -i '/^management-server-secret=/d' \
  /var/lib/docker/volumes/minecraft_java-data/_data/server.properties
```

Restart Java:

```bash
sudo docker-compose up -d minecraft-java
```

Confirm new values exist without displaying them:

```bash
sudo grep -E '^(rcon.password|management-server-secret)=' \
  /var/lib/docker/volumes/minecraft_java-data/_data/server.properties \
  | sed 's/=.*/=[REDACTED]/'
```

## RCON

RCON is enabled inside the Java container on TCP 25575, but the port is not published to the Ubuntu host.

Do not add this mapping without a documented security requirement:

```yaml
- "25575:25575"
```

---

# Optional UFW Configuration

Current state:

```text
UFW inactive
```

Optional LAN rules:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow from 10.1.1.0/24 to any port 22 proto tcp
sudo ufw allow from 10.1.1.0/24 to any port 25565 proto tcp
sudo ufw allow from 10.1.1.0/24 to any port 19132 proto udp

sudo ufw enable
sudo ufw status verbose
```

Test Playit and local clients immediately after enabling UFW.

Docker manages its own packet-filtering rules, so UFW behavior must be validated rather than assumed.

Do not enable UFW during a remote-only maintenance session unless SSH access has already been explicitly permitted.

---

# Router Verification

No Minecraft router port forwarding was intentionally configured.

Verify on the lab router/gateway:

1. Open the gateway administration interface.
2. Review **Firewall** and **NAT / Port Forwarding**.
3. Confirm that TCP `25565` is not forwarded to `10.1.1.116`.
4. Confirm that UDP `19132` is not forwarded to `10.1.1.116`.
5. Confirm that `10.1.1.116` remains reserved for the Dell OptiPlex.

Playit.gg should remain the only intended public ingress path.

---

# Troubleshooting

## Containers Are Not Running

```bash
sudo systemctl status docker --no-pager
cd /srv/minecraft
sudo docker-compose ps
sudo docker-compose up -d
```

Review logs:

```bash
sudo docker logs --tail=200 mc-java
sudo docker logs --tail=200 mc-bedrock
```

## Java Local Connection Fails

```bash
sudo ss -lntp | grep 25565
sudo docker logs --tail=100 mc-java
```

From Windows PowerShell:

```powershell
Test-NetConnection 10.1.1.116 -Port 25565
```

## Bedrock Local Connection Fails

```bash
sudo ss -lnup | grep 19132
sudo docker logs --tail=100 mc-bedrock
```

Confirm:

```text
Address: 10.1.1.116
Port:    19132
```

## Local Works but Remote Fails

Check Playit:

```bash
playit status
sudo systemctl status playit --no-pager
sudo journalctl -u playit -n 200 --no-pager
```

Confirm local tunnel destinations:

```text
Java:    127.0.0.1:25565/TCP
Bedrock: 127.0.0.1:19132/UDP
```

Confirm listeners:

```bash
sudo ss -lntup | grep -E '25565|19132'
```

Restart Playit:

```bash
sudo systemctl restart playit
playit status
```

## Java Server Pauses When Empty

The Java server currently uses:

```properties
pause-when-empty-seconds=60
```

This is expected behavior.

## Verify Persistent Volumes

```bash
sudo docker volume inspect minecraft_java-data
sudo docker volume inspect minecraft_bedrock-data
```

Expected paths:

```text
/var/lib/docker/volumes/minecraft_java-data/_data
/var/lib/docker/volumes/minecraft_bedrock-data/_data
```

## Resource Usage

```bash
free -h
df -h
sudo docker stats --no-stream
```

## Automatic Startup

```bash
sudo systemctl is-enabled docker
sudo systemctl is-enabled playit
```

Check container restart policies:

```bash
sudo docker inspect mc-java \
  --format '{{.HostConfig.RestartPolicy.Name}}'

sudo docker inspect mc-bedrock \
  --format '{{.HostConfig.RestartPolicy.Name}}'
```

Expected:

```text
unless-stopped
```

---

# Reference Commands

## Environment

```bash
hostnamectl
cat /etc/os-release
sudo docker version
sudo docker-compose version
playit version
playit status
```

## Minecraft Status

```bash
cd /srv/minecraft
sudo docker-compose ps
sudo docker ps --format \
  'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
```

## Network

```bash
ip -br address
ip route
sudo docker network inspect minecraft_default
sudo ss -lntup | grep -E '25565|19132'
```

## Volumes

```bash
sudo docker volume inspect minecraft_java-data
sudo docker volume inspect minecraft_bedrock-data
```

## Bedrock Permissions

```bash
sudo ls -ld \
  /var/lib/docker/volumes/minecraft_bedrock-data/_data

sudo find \
  /var/lib/docker/volumes/minecraft_bedrock-data/_data \
  -maxdepth 2 \
  -printf '%M %u:%g %p\n' \
  | sort
```

## Bedrock Process User

```bash
sudo docker exec mc-bedrock id
sudo docker top mc-bedrock -eo user,uid,gid,pid,ppid,cmd
```

## Safe Stop

```bash
cd /srv/minecraft
sudo docker-compose stop
```

## Safe Start

```bash
cd /srv/minecraft
sudo docker-compose up -d
```

---

# Known Follow-Up Actions

- [ ] Verify the actual Bedrock server process UID/GID with `docker top`.
- [ ] Confirm Bedrock remains healthy after changing `/data` ownership to `minecraftadmin:minecraftadmin`.
- [ ] Rotate the Java RCON password.
- [ ] Rotate the Java management server secret.
- [ ] Verify that the lab router/gateway has no Minecraft NAT / Port Forwarding rules.
- [ ] Implement and test the Day 2 backup timer.
- [ ] Copy backups to another server, NAS, or external storage device.
- [ ] Perform a full test restore.
- [ ] Decide whether to enable the Java whitelist.
- [ ] Decide whether to pin Java and Bedrock server versions.
- [ ] Decide whether to enable UFW after testing Docker and Playit behavior.

---

# GitHub Publishing Notes

Before committing this file to GitHub:

1. Replace or review any player names that you do not want published.
2. Confirm no passwords, agent secrets, claim codes, tokens, or account credentials are present.
3. Never add `/etc/playit/playit.toml` to the repository.
4. Never add `.rcon-cli.env` or `.rcon-cli.yaml` to the repository.
5. Consider adding the following patterns to `.gitignore` if configuration files are later copied into the repository:

```gitignore
*.env
*.secret
*.key
playit.toml
.rcon-cli.env
.rcon-cli.yaml
```

