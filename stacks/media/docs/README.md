# 📚 ARR Stack ak Docker Swarm

## 📌 Kisa ki ARR Stack?

- **[Prowlarr](https://github.com/Prowlarr/Prowlarr)** → gere ak kontwole indexer
- **[Sonarr](https://github.com/Sonarr/Sonarr)** → seri
- **[Radarr](https://github.com/Radarr/Radarr)** → fim
- **[qBittorrent](https://github.com/qbittorrent/qBittorrent)** → downloads via torrent
- **[Jellyfin](https://github.com/jellyfin/jellyfin)** → streaming local

👉 Tout sa ap fonksyone otomatikman anndan containers.

---

## 🖥️ Prereki

### Hardware minimòm

- 4 GB RAM
- 2 CPU
- 20 GB free

### Sistèm

- Linux (Ubuntu/Debian rekòmande)
- Windows 10/11 **com WSL2**

---

# 🐳 PARTI 1 – Enstale Docker

## 🐧 Linux (Ubuntu / Debian)

```bash
sudo apt update

sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings


curl -fsSL https://get.docker.com | sudo

sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

Pèmet Docker fonsyone san pèmisyon administratè:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verifye:

```bash
docker version
```

---

## 🪟 Windows (WSL2)

### Pa 1 – Enstale Docker Desktop

- Download nan lyen sa: [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
- Pandan enstalasyon an:
- ✅ Enable WSL2
- ✅ Linux Containers

### Pa 2 – Verifye

No terminal WSL ou PowerShell:

```bash
docker version
```

---

# 🐳 PARTI 2 – Aktive Docker Swarm

```bash
docker swarm init
```

Si kòmand anvan an, baw erè IP:

```bash
docker swarm init --advertise-addr 127.0.0.1
```

Verifye:

```bash
docker node ls
```

---

# 📁 PARTI 3 – Estriti dosye

```bash
mkdir -p media/{config,downloads/{series,movies}}
cd media
```

---

# 📦 PARTI 4 – Docker Compose (yon sèl fichye)

📄 Kreye fichye a:

```bash
nano docker-compose.yml
```

---

## 🧠 docker-compose.yml (ARR Stack konplèt)

```yaml
networks:
  media_net:
    driver: overlay

services:
  prowlarr:
    image: "lscr.io/linuxserver/prowlarr:latest"
    deploy:
      replicas: 1
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Sao_Paulo
    volumes:
      - "./config/prowlarr:/config"
    ports:
      - "9696:9696"
    networks:
      - media_net

  sonarr:
    image: "lscr.io/linuxserver/sonarr:latest"
    deploy:
      replicas: 1
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - "./config/sonarr:/config"
      - "./downloads:/downloads"
    ports:
      - "8989:8989"
    networks:
      - media_net

  radarr:
    image: "lscr.io/linuxserver/radarr:latest"
    deploy:
      replicas: 1
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Sao_Paulo
    volumes:
      - "./config/radarr:/config"
      - "./downloads:/downloads"
    ports:
      - "7878:7878"
    networks:
      - media_net
  qbittorrent:
    image: "lscr.io/linuxserver/qbittorrent:latest"
    deploy:
      replicas: 1
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Sao_Paulo
      - WEBUI_PORT=8080
    volumes:
      - "./config/qbittorrent:/config"
      - "./downloads:/downloads"
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
    networks:
      - media_net

  jellyfin:
    image: "lscr.io/linuxserver/jellyfin:latest"
    deploy:
      replicas: 1
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Sao_Paulo
    volumes:
      - "./config/jellyfin:/config"
      - "./downloads:/data"
    ports:
      - "8096:8096"
    networks:
      - media_net
```

---

# ▶️ PARTI 5 – Deploye stack la

```bash
docker stack deploy -d --prune -c docker-compose.yml media
```

Verifye:

```bash
docker service ls
```

---

# 🌐 PARTI 6 – Aksede ak sèvis yo

| Sèvis       | Lyen                                           | Enfo                                            |
| ----------- | ---------------------------------------------- | ----------------------------------------------- |
| qBittorrent | [http://localhost:8080](http://localhost:8080) | _pran password inisyal la nan log container a._ |
| Radarr      | [http://localhost:7878](http://localhost:7878) |                                                 |
| Sonarr      | [http://localhost:8989](http://localhost:8989) |                                                 |
| Prowlarr    | [http://localhost:9696](http://localhost:9696) |                                                 |
| Jellyfin    | [http://localhost:8096](http://localhost:8096) |                                                 |

---

# 🧪 Troubleshooting Rapid

Gade lòg yo:

```bash
docker service logs media_sonarr
```

Efase stack:

```bash
docker stack rm media
```

---
