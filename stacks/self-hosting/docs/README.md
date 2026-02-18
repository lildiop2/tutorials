# **Self-Hosting Sekirize ak Docker Swarm + Traefik + Cloudflare Tunnel + UFW | Gid pou Debitan**

---

# 1️⃣ Ki sa nou pral konstwi?

Yon sèvè ki:

✅ Gen Docker Swarm
✅ Jere stack ak Portainer
✅ Gen reverse proxy ak Traefik
✅ Gen HTTPS otomatik
✅ Sèvi ak Cloudflare Tunnel
✅ Gen firewall konfigire kòrèkteman
✅ Pa ekspoze container entèn

---

# 2️⃣ Prereki Materyèl

Pou swiv gid sa a, ou bezwen:

- 2 CPU
- 4GB RAM (8GB rekòmande)
- SSD 40GB+
- Ubuntu/ Debian

---

# 3️⃣ Verifye Docker

```bash
docker --version
docker compose version
```

Si docker pa enstale:

```bash
sudo apt update

sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings


curl -fsSL https://get.docker.com | sudo sh

sudo systemctl enable docker.service
sudo systemctl enable containerd.service

sudo usermod -aG docker $USER
newgrp docker
```

---

# 4️⃣ Firewall — Konfigirasyon UFW (ETAP KRITIK)

Docker gen tandans kontoune firewall.
Nou pral fè li respekte UFW.

---

## 🔹 Enstalasyon ufw + fail2ban

```bash
sudo apt install ufw fail2ban -y
```

---

## 🔹 Default Policy

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed
```

---

## 🔹 Ouvri Port Obligatwa

Swarm (si multi-node):

```bash
sudo ufw allow 2377/tcp
sudo ufw allow 7946/tcp
sudo ufw allow 7946/udp
sudo ufw allow 4789/udp
```

Public access:

```bash
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
```

---

## 🔹 Aktive UFW

```bash
sudo ufw enable
sudo ufw status verbose
```

## 🔹 Configire fail2ban

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Modifye:

```bash
sudo nano /etc/fail2ban/jail.local
```

Aktive sshd:

```bash
[sshd]
enabled = true
mode = systemd
logpath = %(sshd_log)s
backend = %(sshd_backend)s
port = 22
maxretry = 5
findtime = 10m
bantime = 1h
banaction = ufw
```

Redemare fail2ban:

```bash
sudo systemctl restart fail2ban
```

Verifye:

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

---

# 5️⃣ Fè Docker Respekte Firewall (OBLIGATWA)

Enstale script:

```bash
sudo wget -O /usr/local/bin/ufw-docker \
https://github.com/chaifeng/ufw-docker/raw/master/ufw-docker

sudo chmod +x /usr/local/bin/ufw-docker
sudo ufw-docker install
```

Initialize:

```bash
sudo ufw enable
sudo systemctl restart docker
```

Sa ap itilize chain `DOCKER-USER` pou bloke trafik ki pa otorize.

---

# 6️⃣ Inisyalize Docker Swarm

```bash
docker swarm init --advertise-addr <SERVER_IP>
```

---

# 7️⃣ Kreye Overlay Networks (MANDATWA)

Public:

```bash
docker network create \
  --driver overlay \
  --attachable \
  public
```

Internal:

```bash
docker network create \
  --driver overlay \
  --attachable \
  internal
```

📌 Règ:

- Web apps → public
- Database / backend → internal sèlman

---

# 8️⃣ Estrikti Dosye Rekòmande

```bash
/repository/stacks
├── portainer/
├── infra/
├── service1/
├── service2/
```

Sa ede òganizasyon pwofesyonèl.

---

# 9️⃣ Deploy Portainer (Swarm Mode)

`portainer/docker-compose.yml`

```yaml
services:
  # =====================================================
  # PORTAINER
  # =====================================================
  portainer:
    image: portainer/portainer-ce:latest
    networks:
      - public
      - internal
    ports:
      - "9000:9000"
    volumes:
      - portainer_data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      TRUSTED_ORIGINS: portainer.domain.com
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      restart_policy:
        condition: any
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.portainer.rule=Host(`portainer.domain.com`)"
        - "traefik.http.routers.portainer.entrypoints=web"
        - "traefik.http.services.portainer.loadbalancer.server.port=9000"

  portainer_agent:
    image: portainer/agent:latest
    networks:
      - internal
    volumes:
      - /:/host
      - /var/lib/docker/volumes:/var/lib/docker/volumes
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      AGENT_CLUSTER_ADDR: tasks.portainer_agent
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      mode: global
      placement:
        constraints: [node.platform.os == linux]
      restart_policy:
        condition: any

volumes:
  portainer_data:

networks:
  public:
    name: public
    external: true

  internal:
    name: internal
    external: true
```

Deploy:

```bash
docker stack deploy -c docker-compose.yml portainer
```

---

# 🔟 Deploy Infra (Traefik + Cloudflare Tunnel)

`infra/docker-compose.yml`

Traefik v3 ak DNS Challenge (Cloudflare):

```yaml
services:
  # =====================================================
  # TRAEFIK
  # =====================================================
  traefik:
    image: traefik:v3.6.7
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_letsencrypt:/letsencrypt
    networks:
      - public
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host
      - target: 443
        published: 443
        protocol: tcp
        mode: host
      - target: 8080
        published: 8080
        protocol: tcp
        mode: host
    environment:
      TZ: America/Sao_Paulo
      CF_API_EMAIL: your_email@email.com
      CF_DNS_API_TOKEN: cloudfare_token
      TRAEFIK_DASHBOARD_CREDENTIALS: admin:hashed_password
    command:
      # Providers
      - "--providers.swarm.endpoint=unix:///var/run/docker.sock"
      - "--providers.swarm.watch=true"
      - "--providers.swarm.exposedbydefault=false"
      - "--providers.swarm.network=public"

      # API & Dashboard
      - "--api.dashboard=true"
      - "--api.insecure=true"

      # Observability
      - "--log.level=INFO"
      - "--accesslog=true"
      - "--metrics.prometheus=true"

      # EntryPoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.websecure.http.tls=true"
      - "--entrypoints.websecure.forwardedHeaders.insecure=true"

      # LetsEncrypt (DNS Challenge - Cloudflare)
      - "--certificatesresolvers.letsencrypt.acme.dnschallenge=true"
      - "--certificatesResolvers.letsencrypt.acme.dnschallenge.resolvers=1.1.1.1:53,1.0.0.1:53"
      - "--certificatesResolvers.letsencrypt.acme.dnschallenge.propagation.delayBeforeChecks=20"
      - "--certificatesresolvers.letsencrypt.acme.email=your_email@email.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"

      # TLS
      - "--entrypoints.websecure.http.tls=true"
      - "--entrypoints.websecure.http.tls.certresolver=letsencrypt"
      - "--entrypoints.websecure.http.tls.domains[0].main=domain.com"
      - "--entrypoints.websecure.http.tls.domains[0].sans=*.domain.com"
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 0
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      labels:
        - "traefik.enable=true"

        # Dashboard
        - "traefik.http.routers.dashboard.rule=Host(`traefik.domain.com`)"
        - "traefik.http.routers.dashboard.entrypoints=web"
        - "traefik.http.routers.dashboard.service=api@internal"
        - "traefik.http.services.dashboard.loadbalancer.server.port=8080"

        # Auth + Global Errors
        - "traefik.http.routers.dashboard.middlewares=dashboard-auth"
        - "traefik.http.middlewares.dashboard-auth.basicauth.users=admin:hashed_password"
        # echo $(htpasswd -nb user password) | sed -e s/\\$/\\$\\$/g

  # =====================================================
  # CLOUDFLARE TUNNEL
  # =====================================================
  cloudflare-tunnel:
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run
    networks:
      - public
    volumes:
      - /etc/localtime:/etc/localtime:ro
    environment:
      TUNNEL_TOKEN: ClOUDFLARE_TOKEN
    healthcheck:
      test: ["CMD", "cloudflared", "--version"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 0
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first

volumes:
  traefik_letsencrypt:

networks:
  public:
    name: public
    external: true
```

Deploy:

```bash
docker stack deploy -c docker-compose.yml infra
```

---

# 1️⃣1️⃣ Deploy Service Egzanp

```yaml
services:
  app:
    image: nginx:alpine
    networks:
      - public
    volumes:
      - ~/Documentos/tutorials/stacks/self-hosting/docs:/usr/share/nginx/html:ro
    deploy:
      replicas: 1
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.app.rule=Host(`app.domain.com`)"
        - "traefik.http.routers.app.entrypoints=websecure"
        - "traefik.http.services.app.loadbalancer.server.port=80"

networks:
  public:
    external: true
```

---

# 1️⃣2️⃣ Validasyon Sekirite

Firewall:

```bash
sudo ufw status verbose
```

Docker chain:

```bash
sudo iptables -L DOCKER-USER -n -v
```

Scan ekstèn:

```bash
nmap -Pn YOUR_IP
```

Rezilta dwe montre sèlman:

- 22
- 80
- 443

Pa dwe gen 9000 ouvè piblik si w itilize Traefik.

---

# 1️⃣3️⃣ Erè Komen

❌ Pa itilize network internal
❌ Pa mete exposedbydefault=false
❌ Pa itilize DNS Challenge
❌ Pa entegre Docker ak UFW
❌ Mete database sou network public

---

# 🔁 Rezime Senp

- UFW = premye baryè sekirite
- Docker Swarm = orchestration
- Overlay networks = izolasyon
- Traefik = sèl pòt piblik
- Cloudflare Tunnel = pwoteksyon IP
- DNS Challenge = SSL otomatik

Sa se yon enfrastrikti production-ready.

---
