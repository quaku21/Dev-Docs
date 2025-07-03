# How to Self-Host n8n on a DigitalOcean Droplet with NGINX and SSL — Without Caddy

> **For developers already running other apps on a production ubuntu server with NGINX.** This guide walks you through running n8n in Docker, setting up NGINX as a reverse proxy, and securing everything with Certbot SSL — without Caddy or touching existing apps.

---

## 🔍 Why This Guide?

The [official n8n documentation](https://docs.n8n.io/hosting/installation/server-setups/digital-ocean/) assumes you're starting fresh with a DigitalOcean Marketplace image and recommends using Caddy as your web server primarily because of its ease of setup and ssl inclusion. That works great for "isolated use" — but does not work when:

- You're using NGINX (or any other web server apart from Caddy) as your reverse proxy
- Ports 80 and 443 are already in use
- You **don't want Caddy** taking over your web server (because you already have an existing work server running very well with other applications)

This guide is tailored for real-world developers with a live server who want to safely run n8n **alongside existing applications**.

It also works for people setting up a new droplet who prefer NGINX to Caddy.

---

## ✅ Prerequisites

- An Ubuntu-based DigitalOcean droplet
- Root/sudo access
- NGINX already installed and working with existing apps (This guide still applies if you're yet to setup NGINX)
- DNS A record for your subdomain (e.g., `ai.yourdomain.com`) pointing to your droplet's IP

---

## Step 1: Install Docker & Docker Compose

If you already have Docker installed on your server, then you need to skip this step

```bash
sudo apt update && sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo usermod -aG docker $USER
```

You may need to log out and back in for group changes to take effect. If you are not comfortable with working as a root user, then create a new user and grant new administrative priveleges. The way you do that it:

```
adduser <username>
```

Follow the prompts on the CLI to finish creating the user.

Then grant the new user administrative privileges with:

```
usermod -aG sudo <username>
```

After this, you will need to login as the new user.

---

## Step 2: Clone Configuration Repository & Create Docker Volumes

Docker Compose and n8n require a series of folders and configuration files. You can clone these from this repository into the home folder of the logged-in user on your Droplet. This will come along with Caddy configuration files/folders are as well. Disregard them as you will not be using Caddy

```bash
git clone https://github.com/n8n-io/n8n-docker-caddy.git
```

And change directory to the root of the repository you cloned:

```bash
cd n8n-docker-caddy
```

Create a Docker volume that Docker reuses between restarts:

```bash
sudo docker volume create n8n_data
```

## Step 3: Set up DNS

n8n typically operates on a subdomain. Create a DNS record with your provider for the subdomain and point it to the public IP address of the server. The exact steps for this depend on your DNS provider, but typically you need to create a new "A" record for the n8n subdomain, for eg ai.yourdomain.com. See https://www.digitalocean.com/community/tutorials/an-introduction-to-dns-terminology-components-and-concepts for support.

## Step 4: Open Ports

For an existing server that already has applications running, the ports must have been opened already. Basically,
n8n runs as a web application, so the server needs to allow incoming access to traffic on port 80 for non-secure traffic, and port 443 for secure traffic. For new servers, open the following ports in the server's firewall by running the following two commands:

```bash
sudo ufw allow 80
sudo ufw allow 443
```

## Step 5: Configure n8n

n8n needs some environment variables set to pass to the application running in the Docker container. The example .env file contains placeholders you need to replace with values of your own.

Open the file with the following command

```bash
nano .env
```

Modify the following:

```env
DOMAIN_NAME=yourdomain.com
SUBDOMAIN=ai
DATA_FOLDER=/home/YOUR_USERNAME/n8n-docker-caddy
GENERIC_TIMEZONE=Europe/London
SSL_EMAIL=you@yourdomain.com
```

For a list of time zones, see https://en.wikipedia.org/wiki/List_of_tz_database_time_zones

## Step 6: The Docker Compose file

The Docker Compose file (docker-compose.yml) defines the services the application needs (Caddy and n8n). Ideally, you should not meddle with this file as it contains exactly what you need. However, because you will not be using Caddy as a web server which is defined as a dependency in the docker-compose.yml file, we will need to get in and make a few changes. The Docker Compose file uses the environment variables set in the .env file. To make this simpler for you, run the below to open the dokcer compose file:

```bash
nano docker-compose.yaml
```

Now, copy paste the below into the docker compose file and save the file. If you are new to using the Nano editor on Linux, see https://www.geeksforgeeks.org/linux-unix/nano-text-editor-in-linux/

As at the time of writing this docs, the version was 3.7, make sure you're using the latest stable version which you will find in the docker-compose.yaml file when you open it. Notice the Caddy dependencies do not feature in the docker-compose.yaml below.

```yaml
version: "3.7"

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
    volumes:
      - n8n_data:/home/node/.n8n
      - ${DATA_FOLDER}/local_files:/files

volumes:
  n8n_data:
    external: true
```

## Step 7: Start Docker Compose

Launch it:

```bash
sudo docker compose up -d
```

Verify it's running on the server:

```bash
curl http://localhost:5678
```

If it's running, you should see an HTML output or a Js warning

---

## Step 8: Configure NGINX Reverse Proxy

Because the pressumption is that NGINX is already being used as a web server, it's already installed and configured. If that's not the case, then https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-plus/ and see https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/ for guidance on how to set it up for reverse proxy

### Create a new site file

```bash
sudo nano /etc/nginx/sites-available/ai.yourdomain.com
```

Paste and replace `ai.yourdomain.com` with the actual subdomain:

```nginx
server {
    listen 80;
    server_name ai.yourdomain.com www.ai.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name ai.yourdomain.com www.ai.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/ai.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ai.yourdomain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/ai.yourdomain.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🔒 Step 5: Set Up SSL with Certbot

```bash
sudo apt install certbot python3-certbot-nginx
```

```bash
sudo certbot --nginx -d ai.yourdomain.com --email you@yourdomain.com --agree-tos --redirect
```

Test renewal:

```bash
sudo certbot renew --dry-run
```

Enable auto-renew timer:

```bash
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer
```

---

## Step 9: Debugging & Gotchas

### Check if n8n is running

```bash
curl http://localhost:5678
```

### Test domain routing:

```bash
curl -I -H "Host: ai.yourdomain.com" http://localhost
```

### Common Mistakes

- ❌ Don’t include full site config in `nginx.conf`. Use `sites-available` + `sites-enabled`
- ❌ Don’t bind ports 80/443 to Caddy or Docker if NGINX is active
- ❌ Don’t forget to reload NGINX after changes

---

## ✅ Final Test

Visit: `https://ai.yourdomain.com`

You should see the full n8n interface loading securely.

---

## 🧩 Bonus Tips

- Use Docker volumes to persist workflows
- Add HTTP basic auth or IP restrictions via NGINX if needed
- Monitor uptime with `docker logs`, `htop`, or external tools like UptimeRobot
