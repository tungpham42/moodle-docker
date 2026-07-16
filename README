# Moodle Docker Deployment with Nginx and Cloudflare SSL

This guide provides step-by-step instructions for deploying Moodle using Docker Compose, backed by a MariaDB database and an Nginx reverse proxy configured to work securely with Cloudflare SSL (Full Strict mode).

## Prerequisites

- **Docker** and **Docker Compose** installed on your server.
- A registered domain name (e.g., `bizeng.vn`) managed via **Cloudflare**.
- Ports `80` and `443` open on your server's firewall.

## Directory Structure

Ensure your project folder matches the following structure before starting the containers:

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
5. Cloudflare will display two blocks of text. Keep this tab open or save them temporarily. You will need the **Origin Certificate** and the **Private Key** for the next step.

## Step 2: Prepare SSL Certificates on the Server

1. Create the `nginx/ssl` directory in your project folder.
2. Create a file named `bizeng.vn.pem` inside `nginx/ssl/` and paste the **Origin Certificate** provided by Cloudflare into it.
3. Create a file named `bizeng.vn.key` inside `nginx/ssl/` and paste the **Private Key** provided by Cloudflare into it.

## Step 3: Start the Application

Run the following command from the root of your project directory (`moodle-docker`) to download the images and start the services in the background:

```bash
docker-compose up -d
```

_Note: Wait a couple of minutes for the database to initialize and Moodle to complete its first-time setup process._

## Step 4: Access Moodle

1. Open your web browser and navigate to your domain: `https://bizeng.vn`.
2. Use the default login credentials (provided by the Bitnami Moodle image):
   - **Username**: `user`
   - **Password**: `bitnami`

_(You will be prompted to change this password upon your first login)._

## Important Notes & Troubleshooting

- **File Upload Limit:** The default max upload size is configured to `100M` in `nginx/default.conf`. You can adjust this value based on your requirements.
- **Client IP Tracking:** The Nginx configuration uses `X-Forwarded-For $http_cf_connecting_ip;` to ensure Moodle logs the real IP address of your users instead of Cloudflare's proxy IPs.
- **Restarting:** If you make any changes to the `docker-compose.yml` or the Nginx configuration, apply them by running:
  ```bash
  docker-compose down
  docker-compose up -d
  ```
