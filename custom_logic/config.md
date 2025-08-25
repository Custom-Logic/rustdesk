### CUSTOM LOGIC RUSTDESK CONFIG

* **RustDesk servers** (`hbbs` + `hbbr`)
* **Nginx** (handles both HTTPS website + RustDesk stream proxy)
* **Certbot** (for Let’s Encrypt auto-renewal)

All in one **`docker-compose.yml`**.

---

## 🚀 Docker Compose Setup

Create a folder:

```bash
mkdir ~/rustdesk && cd ~/rustdesk
```

Create `docker-compose.yml`:

```yaml
version: '3.9'

services:
  rustdesk-hbbs:
    image: rustdesk/rustdesk-server:latest
    container_name: rustdesk-hbbs
    command: hbbs -r hbbr:21117
    restart: unless-stopped
    networks:
      - rustdesk-net
    expose:
      - "21115"
      - "21117"

  rustdesk-hbbr:
    image: rustdesk/rustdesk-server:latest
    container_name: rustdesk-hbbr
    command: hbbr
    restart: unless-stopped
    networks:
      - rustdesk-net
    ports:
      - "21116:21116/tcp"
      - "21116:21116/udp"
      - "21118:21118/tcp"
      - "21118:21118/udp"

  nginx:
    image: nginx:latest
    container_name: rustdesk-nginx
    restart: unless-stopped
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/html:/usr/share/nginx/html
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - rustdesk-hbbs
    networks:
      - rustdesk-net

  certbot:
    image: certbot/certbot
    container_name: rustdesk-certbot
    entrypoint: sh -c "trap exit TERM; while :; do certbot renew --webroot -w /var/www/certbot; sleep 12h & wait $${!}; done"
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    networks:
      - rustdesk-net

networks:
  rustdesk-net:
    driver: bridge
```

---

## ⚙️ Nginx Config

Create `nginx/conf.d/rustdesk.conf`:

```nginx
# HTTP server for Let's Encrypt challenge
server {
    listen 80;
    server_name rustdesk.yourdomain.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# Website (optional, just to keep certbot happy)
server {
    listen 443 ssl;
    server_name rustdesk.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/rustdesk.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/rustdesk.yourdomain.com/privkey.pem;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}

# TCP Stream for RustDesk hbbs
stream {
    upstream rustdesk_hbbs {
        server rustdesk-hbbs:21115;
    }

    server {
        listen 443;
        proxy_pass rustdesk_hbbs;
        proxy_protocol on;
    }
}
```

---

## 🔐 Initial Certificate

Run certbot once manually to fetch certs:

```bash
docker run --rm \
  -v $(pwd)/certbot/conf:/etc/letsencrypt \
  -v $(pwd)/certbot/www:/var/www/certbot \
  certbot/certbot certonly --webroot -w /var/www/certbot -d rustdesk.yourdomain.com --email you@example.com --agree-tos --non-interactive
```

---

## 🔥 Deploy

```bash
docker-compose up -d
```

---

## 🔗 Client Config

In RustDesk client:

* **ID Server** → `rustdesk.yourdomain.com:443` (proxied, Cloudflare orange-cloud OK)
* **Relay Server** → `relay.yourdomain.com:21118` (direct, Cloudflare DNS-only / gray cloud)

---

✅ With this, your VPS:

* Uses Cloudflare proxy for **hbbs** (hidden behind HTTPS 443)
* Exposes **hbbr** relay directly for UDP/TCP fallback
* Auto-renews SSL via Certbot

---
