# 🚀 nginx-quic

[![Build](https://github.com/K1LLSCRIPT/nginx-quic/actions/workflows/build.yml/badge.svg)](https://github.com/K1LLSCRIPT/nginx-quic/actions/workflows/build.yml)
[![APT Repo](https://img.shields.io/badge/apt-repo-blue?logo=debian)](https://k1llscript.github.io/nginx-quic)
[![Pages](https://img.shields.io/github/deployments/K1LLSCRIPT/nginx-quic/github-pages?label=gh-pages&logo=github)](https://k1llscript.github.io/nginx-quic)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

**NGINX with QUIC/HTTP3 + extra modules** packaged as a **Debian** `.deb` and distributed via GitHub Pages APT repo.  
This repo builds NGINX from latest sources with extra modules and OpenSSL, and publishes an installable package.  

---

## 📑 Contents
  ✨ [Features](#-features)  
  📦 [Installation](#-installation)  
  🌐 [Quick Start with HTTP/3](#-quick-start-with-http3)  
  🔑 [GPG Keys & GitHub Secrets](#-gpg-keys--github-secrets)  
  ✅ [TL;DR for forkers](#-tldr-for-forkers)  
  ⚙️ [Requirements](#-requirements)  
  🔒 [Notes for production](#-notes-for-production)  
  📜 [Disclaimer](#-disclaimer)  
  📝 [License](#-license)  

---

## ✨ Features
- ✅ QUIC + HTTP/3 support (nginx-quic)  
- ✅ [ngx_brotli](https://github.com/google/ngx_brotli) – Brotli compression  
- ✅ [headers-more](https://github.com/openresty/headers-more-nginx-module) – response header manipulation  
- ✅ [njs](https://nginx.org/en/docs/njs/) – NGINX JavaScript  
- ✅ [zstd-nginx-module](https://github.com/tokers/zstd-nginx-module) – Zstandard compression  
- ✅ [QuickJS](https://bellard.org/quickjs/) runtime  
- ✅ Packaged as a `.deb` with systemd service, default config, and APT repo support  

---

## 📦 Installation

### 1. Add the repo GPG key
```bash
curl -fsSL https://k1llscript.github.io/nginx-quic/public.key | \
  sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/nginx-quic.gpg
```

### 2. Add APT source
```bash
echo "deb [arch=amd64] https://k1llscript.github.io/nginx-quic trixie main" \
  | sudo tee /etc/apt/sources.list.d/nginx-quic.list
```

### 3. Update and install
```bash
sudo apt update
sudo apt install nginx-quic
```

### 4. Manage service
```bash
sudo systemctl enable --now nginx
sudo systemctl status nginx
```
---


## 🌐 Quick Start with HTTP/3
After installing `nginx-quic` from the custom APT repo, you can quickly spin up a server with **QUIC/HTTP3** support.

### 1. Create certificate (self-signed or from Let’s Encrypt)
```bash
mkdir -p /etc/nginx/ssl
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/example.key \
  -out /etc/nginx/ssl/example.crt \
  -subj "/CN=example.com" -days 365
```
Or use a real certificate (e.g. **Let’s Encrypt**).

### 2. Minimal QUIC-enabled server block
`/etc/nginx/conf.d/quic.conf`:
```nginx
# Redirect HTTP → HTTPS
server {
    server_name example.com;
    listen 80;
    return 308 https://$host$request_uri;
}

# QUIC / HTTP3 server
server {
    server_name example.com;
    listen 443  ssl;
    listen 443  quic reuseport;
    http2  on;
    http3  on;

    ssl_certificate     /etc/nginx/ssl/example.crt;
    ssl_certificate_key /etc/nginx/ssl/example.key;

    # Advertise HTTP/3
    add_header alt-svc 'h3=":443"; ma=86400' always;
    add_header X-HTTP3 $http3 always;

    # Security: clean unwanted headers
    more_clear_headers 'X-Powered-By';

    root  /var/www/quic;
    index index.html;

    location / {
        gzip_static   on;
        brotli_static on;
        expires 1d;
        try_files $uri $uri/ =404;
    }

    location /test {
        default_type  text/plain;
        add_header    alt-svc 'h3=":443"; ma=86400' always;
        add_header    X-QUIC  $http3 always;
        return 200;
    }
}
```

### 3. Create test page
```bash
mkdir -p /var/www/quic
echo "<h1>Welcome to nginx-quic over HTTP/3 🚀</h1>" > /var/www/quic/index.html
```

### 4. Reload nginx
```bash
sudo systemctl enable --now nginx
sudo systemctl reload nginx
```

### 5. Test QUIC / HTTP3
With `curl`:
```bash
curl -vk --http3 https://example.com/test
```
Expected output:
```nginx
ok
```
Or open **Chrome/Firefox/Edge** and check that **HTTP/3** is negotiated (**DevTools** → **Network** → **Protocol** = **h3**).

### 🔥 Boom! You now have a fully working HTTP/3 nginx server from your custom Debian package.

---

## 🔑 GPG Keys & GitHub Secrets
This repository uses **GPG** signing to publish `.deb` packages into the APT repository.
If you fork this project and want to build your own repo, you must **generate your own GPG key** and configure **GitHub Actions** secrets.

### 1. Generate a GPG key
```bash
gpg --full-generate-key
```

- **Type:** `RSA` and `RSA`
- **Size:** `4096`
- **Expiration:** your choice (or never)
- **Name/Email:** use your GitHub identity

### 2. Export keys
Get YOUR_KEYID:
```bash
gpg --list-keys --keyid-format LONG
```

Public (for APT users):
```bash
gpg --armor --export YOUR_KEYID > public.key
```

Private (for **GitHub Actions**):
```bash
gpg --armor --export-secret-keys YOUR_KEYID > private.key
```

⚠️ **Never commit `private.key` to git.** ⚠️

### 3. Add GitHub secrets
Go to **Settings** → **Secrets and variables** → **Actions** in your repo:
- `GPG_PRIVATE_KEY` → contents of `private.key`
- `GPG_PASSPHRASE` → the passphrase you set
- `GH_PAT` → **GitHub Personal Access Token** (with `repo` scope)

### 4. Update workflow
The workflow imports the key:
```yaml
- name: Import GPG key
  run: |
    echo "${{ secrets.GPG_PRIVATE_KEY }}" | gpg --batch --import
```

And in `repo/conf/distributions`:
```makefile
SignWith: YOUR_KEYID
```

Replace `YOUR_KEYID` with your actual GPG key ID (`gpg --list-keys`).

### 5. Distribute the public key
After build, `public.key` is published in your repo.
Users install it with:
```bash
curl -fsSL https://<your-username>.github.io/nginx-quic/public.key | \
  sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/nginx-quic.gpg
```

---

## ✅ TL;DR for forkers
**1)** Generate GPG key  
**2)** Export public/private  
**3)** Add private key + passphrase to secrets  
**4)** Add GitHub PAT  
**5)** Replace `SignWith` in workflow  
**6)** Done → you’ve got your own signed APT repo 🎉  

---

## ⚙️ Requirements
- **Debian 13 (trixie)**  
- **amd64** architecture  
- Tested on `systemd` environments  

---

## 🔒 Notes for production
- Self-signed certs are **only for testing**. Use **Let’s Encrypt** or another **trusted CA** in production.  
- Keep your **GPG private key** secure - leaks compromise your repo trust.  
- This is a **custom build** of NGINX. Validate compatibility with your workloads before deploying to production.  

---

## 📜 Disclaimer
This is **not an official NGINX build**.  
It’s a community-driven project packaged for convenience.  
Use at your own risk - no warranties provided.  

---

## 📝 License
**[MIT](LICENSE)**
