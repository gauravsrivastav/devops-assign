# Project overview: Deployment of Real-Time WebSocket Application (Docker + Nginx + CI/CD): 
A multi-user real-time chat application built with **FastAPI** (WebSocket backend), **NGINX** (reverse proxy + static file server), and **Docker Compose** for orchestration. Fully automated deployment via **GitHub Actions CI/CD**.

---
## Architecture diagram

```
┌────────────────────────────────────────────────────┐
│                  User's Browser                    │
│                                                    │
│   HTTP GET /          WebSocket ws://IP/ws         │
└────────────┬──────────────────┬────────────────────-
             │                  │
             ▼                  ▼
┌────────────────────────────────────────────────────-┐
│              NGINX Container  (port 80)             │
│                                                     │
│  location /      → serve /usr/share/nginx/html      │
│  location /ws    → proxy_pass http://backend:8000   │
│                    + WebSocket upgrade headers      │
└────────────────────────┬───────────────────────────-┘
                         │  Docker bridge network
                         │  (chat-nw)
                         ▼
┌────────────────────────────────────────────────────-┐
│           Backend Container  (port 8000)            │
│                                                     │
│   FastAPI + Uvicorn                                 │
│   Handles WebSocket connections                     │
│   Broadcasts messages to all connected clients      │
└────────────────────────────────────────────────────-┘
```
 
---

## Docker Container Setup
### Backend container
Built from `./Dockerfile` using `python:3.11-slim`. It:
- Installs Python dependencies from `app/requirements.txt`
- Copies `app/main.py` into the container
- Starts Uvicorn bound to `0.0.0.0:8000` (all interfaces)
- Is not exposed to the public — only reachable inside the Docker network

### Nginx Worker
Uses the official `nginx:alpine` image. It:
- Binds to **host port 80** (the only public-facing port)
- Serves frontend static files from `/usr/share/nginx/html` bind 
- Proxies all `/ws` WebSocket traffic to the backend container
- Mounts `./frontend` and `./nginx.conf` as read-only volumes

### Restart Policy
Both containers use `restart: always` — Docker will automatically restart them if they crash or if the server reboots.

---

## Docker Networking
 
Both containers are attached to a custom bridge network named `chat-nw`:
 
```yaml
networks:
  chat-nw:
    driver: bridge
```
 
**How it works:**
- This ensures reliable DNS resolution between containers.
- Docker's embedded DNS assigns each service a hostname matching its service name
- NGINX resolves `backend`  the backend container's internal IP automatically
- No ports need to be published for the backend; `expose: 8000` documents the port internally only
- The only publicly exposed port is `80` on the NGINX container
 
---

## 🔁 How NGINX Reverse Proxy Works
 
```nginx
# Static frontend
location / {
    root /usr/share/nginx/html;
    index index.html;
    try_files $uri $uri/ /index.html;
}
 
# WebSocket backend
location /ws {
    proxy_pass http://backend:8000/ws;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    ...
}
```
- **`location /`** — serves `index.html` and any static assets from the mounted frontend directory
- **`location /ws`** — forwards requests to `http://backend:8000/ws` using the Docker DNS name `backend`
 
---

##  How WebSocket Works Through NGINX
 
WebSocket connections start as a standard HTTP/1.1 request with an **Upgrade** header. NGINX must:
 
1. Forward the `Upgrade: websocket` header to the backend
2. Forward the `Connection: upgrade` header
3. Keep the connection alive with long timeouts
 
```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;    # tells backend to upgrade
proxy_set_header Connection "upgrade";     # keeps connection open
proxy_read_timeout 86400s;                 # 24h — keep WS alive
proxy_send_timeout 86400s;
```
Without the `Upgrade` and `Connection` headers, NGINX downgrades the connection to plain HTTP and the WebSocket handshake fails with a `101 Switching Protocols` error never being returned.

## CI/CD Pipeline (GitHub Actions)
 
### Trigger
Every push to the `main` branch automatically triggers the pipeline.
 
### Pipeline Steps
 
```
Push to main
     │
     ▼
GitHub Actions Runner
     │
     ├─ 1. Checkout repository
     │
     └─ 2. SSH into server
           │
           ├─ cd devops-assign
           ├─ git pull origin main
           ├─ docker compose down --remove-orphans
           ├─ docker compose up -d --build
           └─ docker image prune -f
```

### Required GitHub Secrets  
1. Generate SSH Key on your EC2 server
ssh-keygen -t rsa -b 4096 -C "github-actions-deploy" -f ~/.ssh/github_actions -N ""
2. Add public key to authorized_keys on server
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
3. Copy the private key (you'll need this in next step when configure github action)
cat ~/.ssh/github_actions

4. Add Secrets to GitHub 
| Secret | Description |
|---|---|
| `SERVER_HOST` | Public IP or hostname of your server |
| `SERVER_USER` | SSH username (e.g.  `<your-user-name>`) |
| `SERVER_SSH_KEY` | Private SSH key (contents of `~/.ssh/id_rsa`) |
| `SERVER_PORT` | SSH port (default: `22`) |
| `PROJECT_PATH` | Absolute path to project on server (e.g., `/home/ubuntu/devops-assign`) |
 
### Setting Up Secrets
Go to your GitHub repository → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**.
 
---

## Bugs Found and Fixed

### Bug 1 — Dockerfile: App bound to loopback
**Problem:** `--host 127.0.0.1` made the backend listen only on its own loopback interface. NGINX (a different container) could not reach it at all.
 
**Fix:**
```diff
- CMD ["uvicorn", "main:app", "--host", "127.0.0.1", "--port", "8000"]
+ CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
 
---
 
### Bug 2 — nginx.conf: WebSocket upgrade headers commented out
**Problem:** The two headers required for WebSocket protocol upgrade were commented out, causing all WebSocket connections to silently fail.
 
**Fix:**
```diff
- # proxy_set_header Upgrade $http_upgrade;
- # proxy_set_header Connection "upgrade";
+ proxy_set_header Upgrade $http_upgrade;
+ proxy_set_header Connection "upgrade";
```
 
---
 
### Bug 3 — nginx.conf: `localhost` used instead of Docker service name
**Problem:** `proxy_pass http://localhost:8000/ws` points to NGINX's own loopback — the backend container doesn't exist there.
 
**Fix:**
```diff
- proxy_pass http://localhost:8000/ws;
+ proxy_pass http://backend:8000/ws;
```
 
---
 
### Bug 4 — docker-compose.yml: Broken YAML indentation
**Problem:** All service properties were at the root level (no indentation), making the file invalid YAML that Docker Compose would reject.
 
**Fix:** Corrected to proper 2-space indentation throughout.
 
---
 
### Bug 5 — docker-compose.yml: Frontend volume commented out
**Problem:** The volume mapping for the frontend directory was commented out, so NGINX had no HTML files to serve (all requests returned 404).
 
**Fix:**
```diff
- # - ./frontend:/usr/share/nginx/html:ro
+ - ./frontend:/usr/share/nginx/html:ro
```
 
---
 
##  Deployment Steps
 
### Prerequisites
- A Linux server with Docker and Docker Compose installed
- Git installed on the server
- SSH access to the server
 
### First-Time Setup on Server
 
```bash
# 1. SSH into your server
ssh user@<YOUR_SERVER_IP>
 
# 2. Install Docker if not installed -- here steps for ubuntu
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
 
# 3. Install Docker Compose plugin
sudo apt-get install -y docker-compose-plugin
 
# 4. Clone the repository
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>
 
# 5. Start the application
docker compose up -d --build
 
# 6. Verify containers are running
docker compose ps
```
 
### After First-Time Setup
Every subsequent push to `main` on GitHub will automatically:
1. Pull latest code
2. Rebuild containers
3. Restart the application
 
No manual intervention needed.
 
### Verify It Works
```bash
# Check containers are up
docker compose ps
 
# Tail logs
docker compose logs -f
 
# Test NGINX is serving
curl http://localhost
 
 Tested in multiple browser #  Ensure multi-user chat works in multiple browser tabs, please check screenshot

Open `http://<YOUR_SERVER_IP>` in two browser tabs to test multi-user chat.

## 📸 Screenshots

### Chat Application
![Chat Application](screenshots/single-user.png)

### Multi-user Chat (Two-Three Browser Tabs)
![Multi-user Chat](screenshots/multi-user.png)
 
---