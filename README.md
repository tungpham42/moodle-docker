# Moodle Docker Deployment with Nginx and Cloudflare SSL (Complete Guide)

This guide provides step-by-step instructions for deploying Moodle using Docker Compose, backed by a MariaDB database and an Nginx reverse proxy configured to work securely with Cloudflare SSL (Full Strict mode)[cite: 3]. It includes all necessary configurations and troubleshooting steps to resolve common setup errors.

## Prerequisites

- **Docker** and **Docker Compose** installed on your server[cite: 3].
- A registered domain name (e.g., `bizeng.vn`) managed via **Cloudflare**[cite: 3].
- Ports `80` and `443` open on your server's firewall[cite: 3].

## Directory Structure

Ensure your project folder matches the following structure before starting the containers[cite: 3]:

```text
moodle-docker/
├── docker-compose.yml
└── nginx/
    ├── default.conf
    └── ssl/
        ├── bizeng.vn.pem
        └── bizeng.vn.key
```

## Step 1: Cloudflare Configuration

1. Log in to your Cloudflare dashboard and select your domain.

2. Navigate to **DNS** -> **Records**. Create an `A` record pointing to your server's public IP address. Ensure the "Proxy status" is enabled (the cloud icon should be orange).

3. Navigate to **SSL/TLS** -> **Overview** and set the encryption mode to **Full (Strict)**.

4. Navigate to **SSL/TLS** -> **Origin Server** and click **Create Certificate**. Keep the default settings (RSA, 15 years) and click **Create**.

5. Cloudflare will display two blocks of text. You will need the **Origin Certificate** and the **Private Key** for the next step.

## Step 2: Prepare SSL Certificates on the Server

1. Create the `nginx/ssl` directory in your project folder.

2. Create a file named `bizeng.vn.pem` inside `nginx/ssl/` and paste the **Origin Certificate** provided by Cloudflare into it.

3. Create a file named `bizeng.vn.key` inside `nginx/ssl/` and paste the **Private Key** provided by Cloudflare into it.

## Step 3: Create Configuration Files

### 1. `docker-compose.yml`

Create this file in the root of your `moodle-docker` directory. This version includes the necessary environment variables for Elestio Moodle, including the administrator credentials and the explicit HTTPS URL.

```yaml
services:
  mariadb:
    image: mariadb:10.11
    container_name: moodle_db
    environment:
      - MYSQL_ROOT_PASSWORD=root_secure_password
      - MYSQL_DATABASE=moodle
      - MYSQL_USER=moodle
      - MYSQL_PASSWORD=moodle_secure_password
    volumes:
      - mariadb_data:/var/lib/mysql
    restart: always

  moodle:
    image: elestio/moodle:latest
    container_name: moodle_app
    environment:
      - MOODLE_DATABASE_TYPE=mariadb
      - MOODLE_DATABASE_HOST=mariadb
      - MOODLE_DATABASE_PORT_NUMBER=3306
      - MOODLE_DATABASE_NAME=moodle
      - MOODLE_DATABASE_USER=moodle
      - MOODLE_DATABASE_PASSWORD=moodle_secure_password
      - MOODLE_REVERSEPROXY=yes

      # URL Configuration (Required for Reverse Proxy)
      - MOODLE_URL=[https://bizeng.vn](https://bizeng.vn)

      # Admin Credentials (Required to prevent installation crash)
      - MOODLE_USERNAME=admin
      - MOODLE_PASSWORD=StrongAdminPassword123!
      - MOODLE_EMAIL=admin@bizeng.vn
    volumes:
      - moodle_data:/bitnami/moodle
      - moodledata_data:/bitnami/moodledata
    depends_on:
      - mariadb
    restart: always

  nginx:
    image: nginx:latest
    container_name: moodle_nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - moodle
    restart: always

volumes:
  mariadb_data:
  moodle_data:
  moodledata_data:
```

### 2. `nginx/default.conf`

Create this file in the `nginx/` directory. Ensure the `proxy_pass` points to port `80` (Moodle's internal Apache port).

```nginx
server {
    listen 80;
    server_name bizeng.vn www.bizeng.vn;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name bizeng.vn www.bizeng.vn;

    ssl_certificate /etc/nginx/ssl/bizeng.vn.pem;
    ssl_certificate_key /etc/nginx/ssl/bizeng.vn.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    client_max_body_size 100M;

    location / {
        # Pass traffic to Moodle's internal Apache server on port 80
        proxy_pass http://moodle:80;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $http_cf_connecting_ip;
        proxy_set_header X-Forwarded-For $http_cf_connecting_ip;
        proxy_set_header X-Forwarded-Proto https;

        proxy_connect_timeout 300;
        proxy_send_timeout 300;
        proxy_read_timeout 300;
        send_timeout 300;
    }
}

```

## Step 4: Start the Application

Run the following command from the root of your project directory (`moodle-docker`) to download the images and start the services in the background:

```bash
docker-compose up -d

```

> **Note:** Wait approximately 2–5 minutes for the database to initialize and Moodle to complete its first-time setup process. You can monitor the installation by running `docker logs -f moodle_app`.

## Step 5: Post-Installation Fix (If Required)

If you encounter the error: `Coding error detected... Must use https address in wwwroot when ssl proxy enabled!`, it means Moodle generated its `config.php` file using HTTP before the reverse proxy settings fully applied.

Run this command to force the configuration to HTTPS:

```bash
docker exec -it moodle_app sed -i "s|http://|https://|g" /var/www/html/config.php
docker-compose restart moodle

```

## Step 6: Access Moodle

1. Open your web browser and navigate to your domain: `https://bizeng.vn`.
2. Log in using the credentials defined in your `docker-compose.yml`:

- **Username**: `admin`
- **Password**: `StrongAdminPassword123!`

---

## Troubleshooting Guide

### 1. ERR_CONNECTION_REFUSED

- **Cause:** Nginx failed to start, usually because the SSL files (`.pem` and `.key`) are missing or incorrectly named in the `./nginx/ssl/` directory.
- **Fix:** Verify your SSL certificates exist in the correct folder. Run `docker logs moodle_nginx` to see the exact missing file error.

### 2. 502 Bad Gateway

- **Cause A (During initial startup):** Moodle is still installing its database tables and hasn't started the web server yet.
- **Fix:** Wait 3-5 minutes and refresh the page.

- **Cause B (Persistent):** Nginx is pointing to the wrong internal port.
- **Fix:** Ensure `proxy_pass http://moodle:80;` is set in `default.conf`, not `8080`.

- **Cause C (Container crashed):** Moodle failed to install due to missing admin credentials.
- **Fix:** Ensure `MOODLE_USERNAME` and `MOODLE_PASSWORD` are set in the `docker-compose.yml`. Clean up failed volumes with `docker-compose down -v` and run `docker-compose up -d` again.

### 3. Missing `config.php` during `sed` replace

- **Cause:** Elestio Moodle builds place the web root in `/var/www/html/`, not `/bitnami/moodle/`.
- **Fix:** Always target `/var/www/html/config.php` when modifying core files via the command line for this specific image.
