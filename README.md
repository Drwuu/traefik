# Shared Traefik Setup

This directory contains a standalone Traefik instance that can be shared across all your Docker projects.

## 🚀 Quick Start

### 1. Initial Setup

```bash
cd traefik
cp .env.example .env
# Edit .env with your settings
```

**Required variables:**

- `ACME_EMAIL` - Email for Let's Encrypt certificates
- `TRAEFIK_HOST` - Domain for Traefik dashboard (e.g., traefik.yourdomain.com)
- `TRAEFIK_AUTH` - Basic auth credentials (generate below)

**Generate auth hash:**

```bash
# Install htpasswd (if needed)
sudo apt-get install apache2-utils

# Generate password hash
htpasswd -nb admin yourSecurePassword
# Copy output to TRAEFIK_AUTH in .env
```

### 2. Start Traefik

```bash
docker compose up -d
```

This creates:

- **Network:** `proxy` (for all projects to join)
- **Volume:** `traefik_letsencrypt` (for SSL certificates)
- **Container:** `traefik` (reverse proxy)

### 3. Verify

```bash
docker ps | grep traefik
docker logs traefik

# Visit your dashboard
# https://traefik.yourdomain.com
```

---

## 🔌 Connecting Other Projects

Any Docker Compose project can now use Traefik by:

### Step 1: Join the `proxy` network

```yaml
networks:
  proxy:
    external: true
    name: proxy
```

### Step 2: Add Traefik labels to your service

```yaml
services:
  myapp:
    image: myapp:latest
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`myapp.yourdomain.com`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=letsencrypt"
      - "traefik.http.services.myapp.loadbalancer.server.port=8080"
```

**Key labels:**

- `traefik.enable=true` - Enable routing for this container
- `rule=Host(...)` - Domain to route
- `entrypoints=websecure` - Use HTTPS (443)
- `tls.certresolver=letsencrypt` - Auto SSL certificate
- `loadbalancer.server.port` - Internal container port

---

## 📋 Complete Example

### Example: Adding a new service

`my-project/docker-compose.yml`:

```yaml
version: '3.8'

networks:
  proxy:
    external: true
    name: proxy

services:
  webapp:
    image: nginx:alpine
    container_name: my_webapp
    restart: unless-stopped
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.webapp.rule=Host(`webapp.example.com`)"
      - "traefik.http.routers.webapp.entrypoints=websecure"
      - "traefik.http.routers.webapp.tls.certresolver=letsencrypt"
      - "traefik.http.services.webapp.loadbalancer.server.port=80"
```

Then:

```bash
cd my-project
docker compose up -d
# Visit https://webapp.example.com
```

---

## 🛠️ Management

### View all routes

```bash
docker logs traefik | grep -i router
```

### Check certificates

```bash
docker exec traefik cat /letsencrypt/acme.json | jq
```

### Restart Traefik

```bash
cd traefik
docker compose restart
```

### Stop Traefik

```bash
cd traefik
docker compose down
# Note: This will make ALL routed services unreachable
```

---

## 🔒 Security Notes

1. **Dashboard access:** Always use strong passwords and consider IP whitelisting
2. **Network isolation:** Only expose necessary services to the `proxy` network
3. **Certificate storage:** The `acme.json` file contains private keys - keep secure
4. **Docker socket:** Mounted read-only, but still sensitive

---

## 🐛 Troubleshooting

**Service not accessible:**

```bash
# Check Traefik sees the container
docker logs traefik | grep -i "myapp"

# Verify container is on proxy network
docker inspect myapp | grep -A 10 Networks
```

**SSL certificate issues:**

```bash
# Check ACME logs
docker logs traefik | grep -i acme

# Verify DNS points to your server
dig +short yourdomain.com

# Let's Encrypt rate limits: https://letsencrypt.org/docs/rate-limits/
```

**Connection refused:**

- Ensure `loadbalancer.server.port` matches the container's internal port
- Check container is actually running and healthy

---

## 📚 Advanced Configuration

### HTTP to HTTPS redirect

Already configured globally in Traefik command

### Add custom middleware (e.g., rate limiting)

```yaml
labels:
  - "traefik.http.middlewares.ratelimit.ratelimit.average=100"
  - "traefik.http.routers.myapp.middlewares=ratelimit"
```

### Multiple domains for one service

```yaml
- "traefik.http.routers.myapp.rule=Host(`app.com`) || Host(`www.app.com`)"
```

### TCP/UDP routing

See [Traefik TCP docs](https://doc.traefik.io/traefik/routing/routers/#configuring-tcp-routers)

---

## 🔗 Resources

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Let's Encrypt Docs](https://letsencrypt.org/docs/)
- [Docker Networks Guide](https://docs.docker.com/network/)
