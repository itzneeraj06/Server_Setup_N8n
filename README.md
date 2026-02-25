# n8n Queue Mode — Server Setup Guide

> Full n8n setup with Docker, Traefik (SSL), PostgreSQL, Redis, and Worker node.

---

## Step 1: SSH into Server & Install Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu   # replace 'ubuntu' with 'root' if needed
newgrp docker

# Install Docker Compose
sudo apt install docker-compose-plugin -y

# Verify
docker --version
docker compose version
```

---

## Step 2: Create Project Directory

```bash
sudo mkdir n8n-queue
cd n8n-queue
```

---

## Step 3: Create `.env` File

```bash
sudo nano .env
```

Paste the following variables:

```env
DOMAIN_NAME=your_domain_name
SUBDOMAIN=your_subdomain_name
GENERIC_TIMEZONE=Europe/Berlin
SSL_EMAIL=samyotech@gmail.com
```

---

## Step 4: Create `docker-compose.yml`

```bash
sudo nano docker-compose.yml
```

Paste the following content:

```yaml
services:
  traefik:
    image: "traefik"
    restart: always
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro

  n8n:
    image: docker.n8n.io/n8nio/n8n:2.7.5
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    labels:
      - traefik.enable=true
      - traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)
      - traefik.http.routers.n8n.tls=true
      - traefik.http.routers.n8n.entrypoints=web,websecure
      - traefik.http.routers.n8n.tls.certresolver=mytlschallenge
      - traefik.http.middlewares.n8n.headers.SSLRedirect=true
      - traefik.http.middlewares.n8n.headers.STSSeconds=315360000
      - traefik.http.middlewares.n8n.headers.browserXSSFilter=true
      - traefik.http.middlewares.n8n.headers.contentTypeNosniff=true
      - traefik.http.middlewares.n8n.headers.forceSTSHeader=true
      - traefik.http.middlewares.n8n.headers.SSLHost=${DOMAIN_NAME}
      - traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true
      - traefik.http.middlewares.n8n.headers.STSPreload=true
      - traefik.http.routers.n8n.middlewares=n8n@docker
    environment:
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8ndb
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=AplXOr2S7NY80ZRnBzLd
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_PROXY_HOPS=1
      - N8N_LOG_LEVEL=debug
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - OFFLOAD_MANUAL_EXECUTIONS_TO_WORKERS=true
    depends_on:
      - redis
    volumes:
      - n8n_data:/home/node/.n8n
      - /local-files:/files

  redis:
    image: redis:6
    container_name: redis
    restart: always
    volumes:
      - redis_data:/data

  postgres:
    image: postgres:15
    container_name: n8n-postgres
    restart: always
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: AplXOr2S7NY80ZRnBzLd
      POSTGRES_DB: n8ndb
    volumes:
      - pgdata:/var/lib/postgresql/data

  n8n-worker:
    image: n8nio/n8n
    restart: always
    command: worker
    depends_on:
      - redis
      - postgres
    environment:
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - EXECUTIONS_MODE=queue
      - N8N_CONCURRENCY=10
      - QUEUE_BULL_REDIS_HOST=redis
      - QUEUE_BULL_REDIS_PORT=6379
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8ndb
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=AplXOr2S7NY80ZRnBzLd
      - N8N_ENCRYPTION_KEY=dhfe7OhXMnMY4HKNeoDeSXc/USYb1uoF

volumes:
  traefik_data:
    external: true
  n8n_data:
    external: true
  pgdata:
  redis_data:
```

---

## Step 5: Create Docker Volumes

```bash
docker volume create n8n_data
docker volume create traefik_data
```

---

## Step 6: Start All Services

```bash
# Start in detached mode
docker compose up -d

# Watch logs
docker compose logs -f

# Check status
docker compose ps
```

---

> ✅ Once running, n8n will be accessible at `https://your_subdomain.your_domain_name`
