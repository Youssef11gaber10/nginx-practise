# 🔧 NGINX Practice Lab

A hands-on NGINX learning project covering virtual hosting, URL rewriting, reverse proxying, load balancing, and HTTPS with self-signed certificates — all tested locally using `/etc/hosts` for DNS simulation.

---

## 📁 Repository Structure

```
nginx-practice/
├── etc/
│   ├── nginx/
│   │   ├── nginx.conf                  # Main NGINX config
│   │   └── conf.d/
│   │       ├── example.com.conf        # Basic virtual host with rewrite rules
│   │       ├── vois.com.conf           # Virtual host with autoindex & custom 404
│   │       ├── oldstie.com.conf        # 301 redirect to example.com
│   │       ├── proxy_server.com.conf   # Reverse proxy + load balancer
│   │       ├── backend1.com.conf       # Backend server 1 (port 8080)
│   │       ├── backend2.com.conf       # Backend server 2 (port 8081)
│   │       └── secure-example.com.conf # HTTPS with self-signed certificate
│   └── hosts                           # Local DNS simulation file
└── usr/
    └── share/
        └── nginx/
            ├── example.com/            # Root documents for example.com
            ├── vois.com/               # Root documents for vois.com (+ /files/ dir)
            ├── backend1.com/           # Root documents for backend1
            ├── backend2.com/           # Root documents for backend2
            └── secure-example.com/     # Root documents for secure-example.com
```

---

## 🌐 Local DNS Setup (`/etc/hosts`)

Since this is a local lab, we simulate real domain resolution by mapping domain names to our machine's IP inside `/etc/hosts`:

```
192.168.0.4  example.com vois.com oldsite.com backend1.com backend2.com proxy_server.com
```

This means any request to `example.com` (or any of those names) from your machine gets routed to `192.168.0.4` — your local NGINX server — instead of hitting real DNS.

---

## 📄 Virtual Hosts — Server Blocks on Port 80

NGINX uses `server_name` to differentiate between multiple websites all listening on the same port 80. Each domain gets its own `.conf` file inside `conf.d/`.

### 1. `example.com`

**Config:** `etc/nginx/conf.d/example.com.conf`

```nginx
server {
    listen 80;
    server_name example.com;
    root /usr/share/nginx/example.com;
    index index.html;

    error_page 404 /404.html;
    location = /404.html {
        root /usr/share/nginx/example.com;
        internal;
    }

    location /oldpage {
        rewrite ^/oldpage$ /newpage permanent;
    }
}
```

**What it does:**
- Serves files from `/usr/share/nginx/example.com/`
- Has a custom 404 page (`internal` means users can't access `/404.html` directly)
- Redirects `example.com/oldpage` → `example.com/newpage` permanently (301)

**Test it:**
```bash
# Normal page
curl http://example.com

# Trigger custom 404
curl http://example.com/doesnotexist

# Test the rewrite (follow redirects with -L)
curl -L http://example.com/oldpage
```

---

### 2. `vois.com`

**Config:** `etc/nginx/conf.d/vois.com.conf`

```nginx
server {
    listen 80;
    server_name vois.com;
    root /usr/share/nginx/vois.com;
    index index.html;

    location /files/ {
        autoindex on;
    }

    error_page 404 /404.html;
    location = /404.html {
        root /usr/share/nginx/vois.com;
    }
}
```

**What it does:**
- Serves files from `/usr/share/nginx/vois.com/`
- `autoindex on` for `/files/`: if there's no `index.html` in that directory, NGINX generates a clickable file listing (like an FTP directory view)
- Custom 404 page

**Test it:**
```bash
# Normal homepage
curl http://vois.com

# Browse directory listing for /files/
curl http://vois.com/files/
# Or open in browser: http://vois.com/files/
# You'll see f1.txt and f2.txt listed

# Trigger 404
curl http://vois.com/missing
```

---

### 3. `oldsite.com` → 301 Redirect to `example.com`

**Config:** `etc/nginx/conf.d/oldstie.com.conf`

```nginx
server {
    listen 80;
    server_name oldsite.com;
    return 301 http://example.com$request_uri;
}
```

**What it does:**
Simulates migrating a website. Anyone who visits `oldsite.com` gets permanently redirected to `example.com`, keeping the original path intact (`$request_uri`).

**Test it:**
```bash
# See the 301 redirect header
curl -v http://oldsite.com

# Follow the redirect automatically
curl -L http://oldsite.com/somepage
# → ends up at http://example.com/somepage
```

---

## 🔀 Reverse Proxy & Load Balancer

### 4. `proxy_server.com` (Reverse Proxy + Load Balancer)

**Config:** `etc/nginx/conf.d/proxy_server.com.conf`

```nginx
upstream backend_pool {
    server backend1.com:8080;
    server backend2.com:8081;
}

server {
    listen 80;
    server_name proxy_server.com;

    location / {
        proxy_pass http://backend_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache my_cache;
    }
}
```

**What it does:**

- `upstream backend_pool` defines a group of backend servers
- NGINX acts as a **reverse proxy**: the client only talks to `proxy_server.com`, never directly to the backends
- By default, NGINX uses **Round Robin** load balancing — requests are distributed evenly, alternating between `backend1` and `backend2`
- `X-Real-IP` and `X-Forwarded-For` headers pass the real client IP to the backends
- `Host $host` tells the backend which domain the client was trying to reach

**Round Robin in action:**

```bash
# Run this 4 times and watch which backend responds
curl http://proxy_server.com   # → backend1 responds
curl http://proxy_server.com   # → backend2 responds
curl http://proxy_server.com   # → backend1 responds
curl http://proxy_server.com   # → backend2 responds
```

The pattern will always be: **1 → 2 → 1 → 2 → ...** ♻️

---

### 5. `backend1.com` (Backend Server on port 8080)

**Config:** `etc/nginx/conf.d/backend1.com.conf`

```nginx
server {
    listen 8080;
    server_name backend1.com;
    root /usr/share/nginx/backend1.com;
    index index.html;

    error_page 404 /404.html;
    location = /404.html {
        root /usr/share/nginx/backend1.com;
        internal;
    }
}
```

**Test directly:**
```bash
curl http://backend1.com:8080
```

---

### 6. `backend2.com` (Backend Server on port 8081)

**Config:** `etc/nginx/conf.d/backend2.com.conf`

```nginx
server {
    listen 8081;
    server_name backend2.com;
    root /usr/share/nginx/backend2.com;
    index index.html;

    error_page 404 /404.html;
    location = /404.html {
        root /usr/share/nginx/backend2.com;
        internal;
    }
}
```

**Test directly:**
```bash
curl http://backend2.com:8081
```

---

## 🔒 HTTPS with Self-Signed Certificate

### 7. `secure-example.com`

**Config:** `etc/nginx/conf.d/secure-example.com.conf`

```nginx
# HTTP → HTTPS redirect
server {
    listen 80;
    server_name secure-example.com;
    return 301 https://$host$request_uri;
}

# HTTPS server
server {
    listen 443 ssl;
    server_name secure-example.com;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'HIGH:!aNULL:!MD5';

    root /usr/share/nginx/secure-example.com;
    index index.html;
}
```

**What it does:**

- Two `server` blocks share the same `server_name` but NGINX differentiates them by port (80 vs 443)
- **Port 80 block**: immediately redirects any HTTP request to HTTPS using a 301
- **Port 443 block**: serves the actual content over SSL/TLS
- `HSTS` header (`Strict-Transport-Security`) tells browsers to **always use HTTPS** for this domain for the next year — even if the user types `http://`
- Only allows secure TLS versions (1.2 and 1.3) and strong ciphers

### 🔑 Generating the Self-Signed Certificate

Run this command to generate your certificate and private key:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/nginx-selfsigned.key \
  -out /etc/ssl/certs/nginx-selfsigned.crt
```

| Flag | Meaning |
|------|---------|
| `-x509` | Create a self-signed certificate (not a CSR) |
| `-nodes` | No passphrase on the private key |
| `-days 365` | Certificate valid for 1 year |
| `-newkey rsa:2048` | Generate a new 2048-bit RSA key |
| `-keyout` | Where to save the private key |
| `-out` | Where to save the certificate |

> ⚠️ **Note:** Self-signed certificates will trigger a browser warning ("Your connection is not private"). This is expected in a local lab — real production servers use certificates from a trusted CA (like Let's Encrypt).

**Test it:**
```bash
# HTTP → should redirect to HTTPS
curl -v http://secure-example.com

# HTTPS (ignore cert warning with -k since it's self-signed)
curl -k https://secure-example.com

# Follow the redirect from HTTP to HTTPS
curl -Lk http://secure-example.com
```

---

## 🔄 How NGINX Differentiates Requests

| Request | Matched By | Handled By |
|---|---|---|
| `http://example.com` | `server_name example.com` on port 80 | `example.com.conf` |
| `http://vois.com/files/` | `server_name vois.com` + `location /files/` | `vois.com.conf` |
| `http://oldsite.com` | `server_name oldsite.com` | 301 → `example.com` |
| `http://proxy_server.com` | `server_name proxy_server.com` | forwards to backend pool |
| `http://secure-example.com` | port 80 block | 301 → HTTPS |
| `https://secure-example.com` | port 443 + `server_name` | `secure-example.com.conf` |

---

## ⚙️ Key NGINX Concepts Covered

| Concept | Where Used |
|---|---|
| Virtual Hosts (`server_name`) | All `.conf` files |
| Custom error pages (`error_page`) | example.com, vois.com, backends |
| URL Rewrite (`rewrite`) | example.com `/oldpage` → `/newpage` |
| 301 Permanent Redirect (`return 301`) | oldsite.com, secure-example.com HTTP block |
| Directory Listing (`autoindex on`) | vois.com `/files/` |
| Reverse Proxy (`proxy_pass`) | proxy_server.com |
| Load Balancing (`upstream`, Round Robin) | proxy_server.com |
| Forwarding Client Headers | proxy_server.com (`X-Real-IP`, `X-Forwarded-For`) |
| HTTPS / SSL (`listen 443 ssl`) | secure-example.com |
| HSTS Header | secure-example.com |
| TLS version & cipher control | secure-example.com |
| Self-signed certificate (OpenSSL) | secure-example.com |

---

## 🚀 Getting Started

1. Install NGINX:
   ```bash
   sudo apt install nginx   # Ubuntu/Debian
   sudo dnf install nginx   # Fedora
   ```

2. Copy config files to their locations:
   ```bash
   sudo cp etc/nginx/conf.d/*.conf /etc/nginx/conf.d/
   sudo cp -r usr/share/nginx/* /usr/share/nginx/
   ```

3. Add domain entries to `/etc/hosts` (replace IP with your machine's IP):
   ```bash
   sudo nano /etc/hosts
   # Add: 192.168.0.4 example.com vois.com oldsite.com backend1.com backend2.com proxy_server.com
   ```

4. Generate the SSL certificate (for secure-example.com):
   ```bash
   sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout /etc/ssl/private/nginx-selfsigned.key \
     -out /etc/ssl/certs/nginx-selfsigned.crt
   ```

5. Test the NGINX config and reload:
   ```bash
   sudo nginx -t
   sudo systemctl reload nginx
   ```

---

*Built for learning NGINX fundamentals — virtual hosting, proxying, load balancing, and TLS.*