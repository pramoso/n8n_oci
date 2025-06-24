# 🌐 Configure NGINX with SSL for n8n

These optional steps let you serve n8n through HTTPS using a custom domain name.
You should already have n8n running via Docker on port **5678**.

---

## 1. Point your domain

1. In your DNS provider, create an **A record** pointing your domain (e.g. `n8n.example.com`) to your VM's public IP.
2. Wait for the DNS change to propagate before continuing.

---

## 2. Install NGINX and Certbot

Connect to your instance via SSH and install the packages:

```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx -y
```

Enable NGINX at boot:

```bash
sudo systemctl enable nginx --now
```

---

## 3. Create a reverse proxy config

Create `/etc/nginx/sites-available/n8n` with the following content
(replace `_` => `n8n.example.com` with your domain or leave it with `_` so it catches all domains):

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;  # Catch-all

    location / {
        proxy_pass http://127.0.0.1:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
    }
}
```

Activate the configuration and test:

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

You should now reach n8n via `http://n8n.example.com`.

---

## 4. Obtain a Let's Encrypt certificate

Run Certbot to request an SSL certificate and let it update the NGINX config:

```bash
sudo certbot --nginx -d n8n.example.com
```

Certbot will configure HTTPS and set up automatic renewal.
You can test renewal with:

```bash
sudo certbot renew --dry-run
```


---

### Update n8n settings (optional)

To ensure n8n generates correct webhook URLs, add these environment variables
to the `n8n` service in your `docker-compose.yml`:

```yaml
    environment:
      - N8N_HOST=n8n.example.com
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.example.com/
```


You also neer to remove `N8N_SECURE_COOKIE=false` which was set to allow logins over plain HTTP.
You can remove that line after obtaining the certificate so that cookies are marked
secure.

Recreate the container after editing:

```bash
docker compose up -d
```

Your instance is now secured with HTTPS via NGINX. You can now connect to `https://n8n.example.com`.

