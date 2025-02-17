# Nginx Reverse Proxy Setup Guide

## Prerequisites
- **Ubuntu Server** with Nginx installed.
- **Self-hosted Let's Encrypt SSL certificate:**
  - Install Certbot:
    ```bash
    sudo apt install certbot python3-certbot-nginx -y
    ```
  - Issue SSL certificates:
    ```bash
    sudo certbot --nginx -d example.com -d media.example.com
    ```
  - Enable auto-renewal:
    ```bash
    sudo systemctl enable certbot.timer
    ```
- **Docker container running**:
  - Media Server → `192.168.1.100:8096`
- **Domain & DNS Configuration**:
  - CNAME records:
    - `media.example.com → example.com`
- **Port Forwarding**:
  - **80 → Server IP** (For Let's Encrypt validation)
  - **443 → Server IP** (For secure HTTPS traffic)

---

## Step 1: Install & Configure Nginx
1. **Install Nginx**:
    ```bash
    sudo apt update && sudo apt install nginx -y
    ```
2. **Enable Nginx on boot**:
    ```bash
    sudo systemctl enable nginx
    sudo systemctl start nginx
    ```
3. **Allow traffic through firewall**:
    ```bash
    sudo ufw allow "Nginx Full"
    ```
4. **Verify Nginx is running**:
    ```bash
    sudo systemctl status nginx
    ```

---

## Step 2: Global Rate Limiting
1. **Edit the global Nginx config**:
    ```bash
    sudo nano /etc/nginx/nginx.conf
    ```
2. **Add inside `http {}` block**:
    ```nginx
    limit_req_zone $binary_remote_addr zone=rate_limit:10m rate=5r/s;
    ```
3. **Save and exit** (`CTRL+X`, then `Y`, then `ENTER`).

---

## Step 3: Default Site Configuration (Landing Page)
1. **Create configuration file**:
    ```bash
    sudo nano /etc/nginx/sites-available/default_site
    ```
2. **Paste configuration**:
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

        # Add strong SSL/TLS protocols and ciphers
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384';
        ssl_prefer_server_ciphers on;

        # OCSP Stapling for improved SSL/TLS performance
        ssl_stapling on;
        ssl_stapling_verify on;

        root /var/www/html;
        index index.html index.htm;

        # Enable Compression
        gzip on;
        gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        # Caching for static files
        location ~* \.(ico|css|js|gif|jpe?g|png|woff2?|eot|ttf|otf|svg|mp4)$ {
            expires 6M;
            access_log off;
            add_header Cache-Control "public, no-transform";
        }

        # Security Headers
        add_header X-Frame-Options DENY;
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options nosniff;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";

        # Log files for monitoring and troubleshooting
        access_log /var/log/nginx/example_access.log;
        error_log /var/log/nginx/example_error.log;
    }
    ```
3. **Enable the site & restart Nginx**:
    ```bash
    sudo ln -s /etc/nginx/sites-available/default_site /etc/nginx/sites-enabled/
    sudo nginx -t && sudo systemctl restart nginx
    ```

---

## Step 4: Reverse Proxy for Media Server
1. **Create configuration file**:
    ```bash
    sudo nano /etc/nginx/sites-available/media_server
    ```
2. **Paste configuration**:
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

        # Add strong SSL/TLS protocols and ciphers
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384';
        ssl_prefer_server_ciphers on;

        # OCSP Stapling for improved SSL/TLS performance
        ssl_stapling on;
        ssl_stapling_verify on;

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
        # Log files for monitoring and troubleshooting
        access_log /var/log/nginx/media_example_access.log;
        error_log /var/log/nginx/media_example_error.log;
    }
    ```
3. **Enable the site & restart Nginx**:
    ```bash
    sudo ln -s /etc/nginx/sites-available/media_server /etc/nginx/sites-enabled/
    sudo nginx -t && sudo systemctl restart nginx
    ```

---

## Step 5: Set Correct Permissions
1. **Apply correct ownership**:
    ```bash
    sudo chown -R www-data:www-data /var/www/html
    sudo chmod -R 755 /var/www/html
    ```
2. **Restart Nginx**:
    ```bash
    sudo systemctl restart nginx
    ```

---
