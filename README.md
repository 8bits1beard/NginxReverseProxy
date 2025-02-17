# **Nginx Reverse Proxy Setup Guide**

This guide outlines the step-by-step process for setting up an Nginx reverse proxy with SSL for multiple sites, including a default landing page, a media server, and a request management system.

## **Prerequisites**

- **Ubuntu Server** with Nginx installed
- **Self-hosted Let's Encrypt SSL certificate installed with Nginx**
  - Certbot installed: `sudo apt install certbot python3-certbot-nginx -y`
  - SSL certificate issued for your domain:
    ```bash
    sudo certbot --nginx -d example.com -d media.example.com -d request.example.com
    ```
  - Automatic renewal configured:
    ```bash
    sudo systemctl enable certbot.timer
    ```
- **Docker containers running**:
  - **Media Server**: Example: `192.168.1.100:8096`
  - **Request Management**: Example: `192.168.1.100:3579`
- **Domain & DNS Configuration**:
  - Each subdomain should have a **CNAME record** pointing to the main domain:
    - `media.example.com → example.com`
    - `request.example.com → example.com`
- **Port Forwarding**:
  - **80 → Server IP (HTTP)** (Required for Let's Encrypt verification)
  - **443 → Server IP (HTTPS)**

---

## **Step 1: Install & Configure Nginx**

Ensure Nginx is installed and configured correctly:

```bash
sudo apt update && sudo apt install nginx -y
```

Enable Nginx to start on boot:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

Allow Nginx traffic through the firewall:

```bash
sudo ufw allow "Nginx Full"
```

Test Nginx:

```bash
sudo systemctl status nginx
```

If running, you should see `active (running)`.

---

## **Step 2: Define Global Rate Limiting**

Edit the main Nginx configuration file:

```bash
sudo nano /etc/nginx/nginx.conf
```

Add the following inside the `http {}` block:

```nginx
limit_req_zone $binary_remote_addr zone=rate_limit:10m rate=5r/s;
```

Save and exit (`CTRL+X`, then `Y`, then `ENTER`).

---

## **Step 3: Set Up the Default Site**

Create the configuration file:

```bash
sudo nano /etc/nginx/sites-available/default_site
```

Paste the following:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    root /var/www/html;
    index index.html index.htm;

    # Enable Gzip Compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    # Enable Browser Caching
    location ~* \.(?:ico|css|js|gif|jpe?g|png|woff2?|eot|ttf|otf|svg|mp4)$ {
        expires 6M;
        access_log off;
        add_header Cache-Control "public, no-transform";
    }

    # Add Security Headers
    add_header X-Frame-Options DENY;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options nosniff;
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/default_site /etc/nginx/sites-enabled/
```

Test and restart Nginx:

```bash
sudo nginx -t && sudo systemctl restart nginx
```

---

## **Step 4: Set Up Reverse Proxy for Media Server**

Create the configuration file:

```bash
sudo nano /etc/nginx/sites-available/media_server
```

Paste the following:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name media.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name media.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    location / {
        proxy_pass http://192.168.1.100:8096;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_cache_bypass $http_upgrade;
        proxy_connect_timeout 600;
        proxy_send_timeout 600;
        proxy_read_timeout 600;
        send_timeout 600;
        client_max_body_size 10G;
        tcp_nodelay on;
        tcp_nopush on;
    }
}
```

Enable and restart:

```bash
sudo ln -s /etc/nginx/sites-available/media_server /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx
```

---

## **Step 5: Set Up Reverse Proxy for Request Management**

Create the configuration file:

```bash
sudo nano /etc/nginx/sites-available/request_manager
```

Paste the following:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name request.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name request.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    location / {
        proxy_pass http://192.168.xxx.xxx:3579;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket Support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";

        # Increase Timeouts
        proxy_connect_timeout 120;
        proxy_send_timeout 120;
        proxy_read_timeout 120;

        # Rate Limiting
        limit_req zone=rate_limit burst=10;
    }
}
```

Enable and restart:

```bash
sudo ln -s /etc/nginx/sites-available/request_manager /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx
```

---

## **Step 6: Set Correct Permissions**

Ensure that the necessary permissions are applied to the web root directory:

```bash
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html
```

Restart Nginx to apply changes:

```bash
sudo systemctl restart nginx
```
---
