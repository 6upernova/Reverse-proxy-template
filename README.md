# NGINX Reverse Proxy (Docker) – Setup Minimal y Escalable

Este repositorio contiene una configuración simple y eficiente para levantar un **reverse proxy con NGINX en Docker**, similar a Nginx Proxy Manager pero sin interfaz gráfica. Está pensado para entornos de bajo consumo.

---

## Estructura del proyecto

```
.
├── docker-compose.yml
├── nginx/
│   ├── nginx.conf
│   ├── conf.d/
│   │   └── app.conf
│   ├── snippets/
│   │   ├── proxy.conf
│   │   └── ssl.conf
│   └── ssl/ (opcional)
├── certbot/
│   └── www/
├── frontend/
├── backend/
```

---

##  Servicios

* **nginx** → Reverse proxy (entrada principal HTTP/HTTPS)
* **frontend** → App estática (ej: React/Vite build)
* **backend** → API (ej: Go en puerto 8080)
* **certbot** → Generación y renovación de SSL

---

##  Requisitos

* Docker + Docker Compose
* Dominio apuntando a tu IP pública
* Puertos abiertos:

  * 80 (HTTP)
  * 443 (HTTPS)

---

##  Configuración básica

### docker-compose.yml (ejemplo simplificado)

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:latest
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/snippets:/etc/nginx/snippets
      - ./certbot/www:/var/www/certbot
      - ./certbot/conf:/etc/letsencrypt
    depends_on:
      - frontend
      - backend
    networks:
      - proxy_net

  frontend:
    build: ./frontend
    networks:
      - proxy_net

  backend:
    build: ./backend
    networks:
      - proxy_net

  certbot:
    image: certbot/certbot
    volumes:
      - ./certbot/www:/var/www/certbot
      - ./certbot/conf:/etc/letsencrypt

networks:
  proxy_net:
    driver: bridge
```

---

##  Configuración NGINX

### nginx.conf (mínimo requerido)

```nginx
worker_processes 1;

events {
    worker_connections 512;
}

http {
    include       /etc/nginx/mime.types;
    sendfile      on;

    include /etc/nginx/conf.d/*.conf;
}
```

---

### conf.d/app.conf

```nginx
server {
    listen 80;
    server_name tudominio.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        proxy_pass http://frontend:80;
        include /etc/nginx/snippets/proxy.conf;
    }

    location /api/ {
        proxy_pass http://backend:8080/;
        include /etc/nginx/snippets/proxy.conf;
    }
}
```

---

##  Proxy snippet

### snippets/proxy.conf

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

---

##  SSL con Certbot

### Generar certificado

```bash
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  -d tudominio.com \
  --email tu@email.com \
  --agree-tos
```

---

### Activar HTTPS

Agregar en `app.conf`:

```nginx
server {
    listen 80;
    server_name tudominio.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name tudominio.com;

    ssl_certificate /etc/letsencrypt/live/tudominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tudominio.com/privkey.pem;

    location / {
        proxy_pass http://frontend:80;
    }

    location /api/ {
        proxy_pass http://backend:8080/;
    }
}
```

---

##  Renovación automática

Agregar cron:

```bash
0 3 * * * docker compose run --rm certbot renew && docker exec nginx-proxy nginx -s reload
```

---

##  Testing

### Verificar servicios

```bash
docker ps
```

### Probar backend interno

```bash
docker exec -it nginx-proxy curl http://backend:8080
```

### Probar desde host

```bash
curl http://localhost
```

---
