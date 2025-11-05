# Shared Traefik Setup

This directory contains a standalone Traefik instance that can be shared across all your Docker projects.

## 🚀 Quick Start

### 1. Initial Setup

```bash
cd traefik
cp .env.example .env
# Edit .env with your settings
```

**Main variables to set in `.env`:**
- `TRAEFIK_DOMAIN` - Local domain for Traefik (optional)
- `TRAEFIK_HOST` - Dashboard domain (e.g., traefik.yourdomain.com)
- `ACME_EMAIL` - Email for Let's Encrypt certificates
- `TRAEFIK_AUTH` - Basic auth credentials (see below)
- `TRAEFIK_SSL` - Enable SSL (true/false)
- `TRAEFIK_DASHBOARD` - Enable dashboard (true/false)
- `TRAEFIK_LETSENCRYPT_STORAGE` - Path for SSL cert storage

**Generate auth hash:**

```bash
# Install htpasswd (if needed)
sudo apt-get install apache2-utils

# Generate password hash
htpasswd -nb admin mySecurePassword
# Output: admin:$apr1$example$hash
# Copy output to TRAEFIK_AUTH in .env (replace $ with $$)
```

### 2. Start Traefik

```bash
docker compose up -d
```

Traefik will use all settings from your `.env` file.

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

## 📝 Environment Variables Reference

See `.env.example` for all available variables. Edit `.env` to customize your setup.

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
