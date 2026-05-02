OpenCTI + TAXII + NGINX Deployment Guide
========================================


Overview
--------

This document describes how to deploy a test environment for:

* OpenCTI
* TAXII 2.1 endpoint
* NGINX reverse proxy with HTTPS (Let’s Encrypt)

Architecture
------------
                       Client
                         │
                         │  HTTPS (443)
                         ▼
             ┌─────────────────────────┐
             │   NGINX Reverse Proxy   │
             │                         │
             │ - TLS Termination       │
             │ - Let's Encrypt         │
             └──────────┬──────────────┘
                        │
                        │ HTTP (8080)
                        ▼
             ┌─────────────────────────┐
             │    OpenCTI Server       │
             │   (Docker Host)         │
             │                         │
             │ - OpenCTI Platform      │
             │ - TAXII 2.1 Endpoint    │
             │ - Connectors            │
             └──────────┬──────────────┘
                        │
                        │ Internal Services
                        ▼
             ┌─────────────────────────┐
             │   External Feeds        │
             │                         │
             │ - CrowdStrike           │
             │ - Mandiant              │
             │ - Other Intel Sources   │
             └─────────────────────────┘

🖥️ System Requirements
-----------------------

### NGINX Server (Internet-facing)

| Resource | Recommended |
|----------|-------------|
| CPU      | 2–4 vCPU    |
| RAM      | 4–8 GB      |
| Disk     | 50–100 GB   |
| Network  | Public IP   |


### OpenCTI Server (Docker Host)

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU      | 4 vCPU  | 8 vCPU      |
| RAM      | 8 GB    | 16–32 GB    |
| Disk     | 100 GB  | 200+ GB     |

Step 1 - OpenCTI setup using Docker
-----------------------------------

OpenCTI can be deployed using the docker-compose command.

### Pre-requisites -  Linux 🐧

```
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo apt update
```

### Clone repository

```
mkdir -p /path/to/your/app && cd /path/to/your/app
git clone https://github.com/OpenCTI-Platform/docker.git
cd docker
```

### Configure environment

Before running the `docker-compose` command, the `docker-compose.yml` file should be configured.  Create a `.env` file with the following:

```
sudo apt install -y jq
cd ~/docker
(cat << EOF
###########################
# DEPENDENCIES            #
###########################

MINIO_ROOT_USER=$(cat /proc/sys/kernel/random/uuid)
MINIO_ROOT_PASSWORD=$(cat /proc/sys/kernel/random/uuid)
RABBITMQ_DEFAULT_USER=opencti
RABBITMQ_DEFAULT_PASS=$(openssl rand -base64 32)
SMTP_HOSTNAME=localhost
OPENSEARCH_ADMIN_PASSWORD=changeme
ELASTIC_MEMORY_SIZE=4G

###########################
# COMMON                  #
###########################

XTM_COMPOSER_ID=8215614c-7139-422e-b825-b20fd2a13a23
COMPOSE_PROJECT_NAME=xtm

###########################
# OPENCTI                 #
###########################

OPENCTI_HOST=localhost
OPENCTI_PORT=8080
OPENCTI_EXTERNAL_SCHEME=http
OPENCTI_ADMIN_EMAIL=admin@example.com
OPENCTI_ADMIN_PASSWORD=StrongPassword123!
OPENCTI_ADMIN_TOKEN=$(cat /proc/sys/kernel/random/uuid)
OPENCTI_HEALTHCHECK_ACCESS_KEY=$(cat /proc/sys/kernel/random/uuid)
OPENCTI_ENCRYPTION_KEY=$(openssl rand -base64 32)

###########################
# OPENCTI CONNECTORS      #
###########################

CONNECTOR_EXPORT_FILE_STIX_ID=dd817c8b-abae-460a-9ebc-97b1551e70e6
CONNECTOR_EXPORT_FILE_CSV_ID=7ba187fb-fde8-4063-92b5-c3da34060dd7
CONNECTOR_EXPORT_FILE_TXT_ID=ca715d9c-bd64-4351-91db-33a8d728a58b
CONNECTOR_IMPORT_FILE_STIX_ID=72327164-0b35-482b-b5d6-a5a3f76b845f
CONNECTOR_IMPORT_DOCUMENT_ID=c3970f8a-ce4b-4497-a381-20b7256f56f0
CONNECTOR_IMPORT_FILE_YARA_ID=7eb45b60-069b-4f7f-83a2-df4d6891d5ec
CONNECTOR_IMPORT_EXTERNAL_REFERENCE_ID=d52dcbc8-fa06-42c7-bbc2-044948c87024
CONNECTOR_ANALYSIS_ID=4dffd77c-ec11-4abe-bca7-fd997f79fa36

###########################
# OPENCTI DEFAULT DATA    #
###########################

CONNECTOR_OPENCTI_ID=dd010812-9027-4726-bf7b-4936979955ae
CONNECTOR_MITRE_ID=8307ea1e-9356-408c-a510-2d7f8b28a0e2
EOF
) > .env
```

Update the following variables in the `.env` file:

```
SMTP_HOSTNAME=localhost
OPENSEARCH_ADMIN_PASSWORD=changeme

OPENCTI_HOST=localhost
OPENCTI_EXTERNAL_SCHEME=http
OPENCTI_ADMIN_EMAIL=admin@example.com
OPENCTI_ADMIN_PASSWORD=StrongPassword123!
```

### Edit the `docker-compose.yml` file

#### ElasticSearch configuration:

If you are installing from scratch, Filigran strongly recommends that you add the following ElasticSearch / OpenSearch parameter in `docker-compose.yml`:

```
  elasticsearch:
    environment:
      - thread_pool.search.queue_size=5000
```

#### OpenCTI configuration:

To create local users within the OpenCTI UI, you will need to add the following parameter in `docker-compose.yml`:

```
  opencti:
    environment:
      - PROVIDERS__LOCAL__CONFIG__DISABLED=false
```

The `APPAPP__BASE_URL` should match the URL on the external side of Nginx. If it is a URL listening on a standard port, just remove `:${OPENCTI_PORT}` from the end of the variable.

```
  opencti:
    environment:
      - APP__BASE_URL=${OPENCTI_EXTERNAL_SCHEME}://${OPENCTI_HOST}:${OPENCTI_PORT}
```

As OpenCTI has a dependency on ElasticSearch, you have to set `vm.max_map_count` before running the containers, as mentioned in the ElasticSearch documentation.

`sudo sysctl -w vm.max_map_count=1048575`

To make this parameter persistent, add the following to the end of your /etc/sysctl.conf:

`vm.max_map_count=1048575`

#### CrowdStrike OpenCTI connector:

To pull Threat Intel from CrowdStrike:

Add the following in the `.env`

```
###########################
# CrowdStrike             #
###########################

CONNECTOR_CROWDSTRIKE_ID=e9d58dbd-698c-4c22-b05b-bd1ec710ea43
CRWD_URL=https://api.us-2.crowdstrike.com
CRWD_CLIENT_ID=<client_id>
CRWD_CLIENT_SECRET=<client_secret>
```

Make sure you use the real `<client_id>` and `<client_secret>`.

Add the following in the `# OPENCTI CONNECTORS #` section in `docker-compose.yml`:

```
  connector-crowdstrike:
    image: opencti/connector-crowdstrike:7.260417.0
    restart: always
    environment:
      - OPENCTI_URL=http://opencti:8080
      - OPENCTI_TOKEN=${OPENCTI_ADMIN_TOKEN}
      - CONNECTOR_ID=${CONNECTOR_CROWDSTRIKE_ID}
      - CONNECTOR_NAME=CrowdStrike
      - CONNECTOR_SCOPE=crowdstrike
      - CONNECTOR_LOG_LEVEL=info
      - CONNECTOR_DURATION_PERIOD=PT30M

      - CROWDSTRIKE_BASE_URL=${CRWD_URL}
      - CROWDSTRIKE_CLIENT_ID=${CRWD_CLIENT_ID}
      - CROWDSTRIKE_CLIENT_SECRET=${CRWD_CLIENT_SECRET}
      - CROWDSTRIKE_TLP=amber+strict

      - CROWDSTRIKE_CREATE_OBSERVABLES=true
      - CROWDSTRIKE_CREATE_INDICATORS=true
      - CROWDSTRIKE_SCOPES=indicator
      - CROWDSTRIKE_INDICATOR_START_TIMESTAMP=0
      - CROWDSTRIKE_INDICATOR_EXCLUDE_TYPES=hash_ion,password
      - CROWDSTRIKE_DEFAULT_X_OPENCTI_SCORE=50
      # Score mapping
      - CROWDSTRIKE_INDICATOR_LOW_SCORE=40
      - CROWDSTRIKE_INDICATOR_LOW_SCORE_LABELS=MaliciousConfidence/Low
      - CROWDSTRIKE_INDICATOR_MEDIUM_SCORE=60
      - CROWDSTRIKE_INDICATOR_MEDIUM_SCORE_LABELS=MaliciousConfidence/Medium
      - CROWDSTRIKE_INDICATOR_HIGH_SCORE=100
      - CROWDSTRIKE_INDICATOR_HIGH_SCORE_LABELS=MaliciousConfidence/High
      # Score Filtering
      # - CROWDSTRIKE_INDICATOR_UNWANTED_LABELS=MaliciousConfidence/Low
```

### ▶️ Run OpenCTI


#### Using single node Docker

After changing your `.env` file run `docker-compose` in detached (-d) mode:

```
sudo systemctl start docker.service
# Run docker-compose in detached
sudo docker compose up -d
```

⏳ Allow 5–10 minutes for full startup

#### Verify

`curl http://localhost:8080`

Step 2 - Create a TAXII Collection & API Key
--------------------------------------------

Before you can use the API to pull indicators from OpenCTI, you must create an API key and TAXII Collection.

### a) Create an API Token

In the upper right corner of the screen, click on the user icon and click `Profile`

Then:

#### a) Create a Feed

Go to:

`Data → Data sharing → Feeds`

Then:

Click “__+ Create feed__”

Fill in:

Name: `<collection name>`

Description: (optional)

Type: TAXII 2.1 (default)

#### b) Test with curl

```
curl -s "http://<opencti>:8080/taxii2/<api-root>/collections/" \
  -H "Authorization: Bearer <your-api-token>" \
  -H "Accept: application/taxii+json;version=2.1" | jq
```

#### c) Then pull indicators:

```
curl -s "http://<opencti>:8080/taxii2/<api-root>/collections/<collection-id>/objects/?match[type]=indicator" \
  -H "Authorization: Bearer <your-api-token>" \
  -H "Accept: application/taxii+json;version=2.1" | jq
```

Step 3 – Verify TAXII Endpoint
------------------------------

Open in browser:

`http://<opencti-ip>:8080`

Then confirm:

`/taxii2/`

Step 4 – Prepare NGINX Server
-----------------------------

```
sudo apt update
sudo apt upgrade -y

sudo apt install -y \
  curl wget vim git jq \
  ca-certificates gnupg lsb-release \
  net-tools iproute2 tcpdump htop \
  ufw fail2ban nftables

#Install Docker
curl -fsSL https://get.docker.com | sudo sh

sudo usermod -aG docker $USER

# Create project structure
mkdir -p /opt/nginx/{conf,letsencrypt,www,cache}
sudo chown -R $USER:$USER /opt/nginx
cd /opt/nginx
```

Step 5 – NGINX Configuration
----------------------------

NOTE:  If the default gateway of the OpenCTI host does not point back to this Nginx host, proxy_bind will also need to be added to the location sections in the HTTPS server below.

`vi conf/default.conf`

```
gzip on;
gzip_types application/javascript text/css application/json;

# HTTP → HTTPS
server {
    listen 80;
    server_name taxii.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS
server {
    listen 443 ssl;
    server_name taxii.example.com;

    ssl_certificate     /etc/letsencrypt/live/taxii.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/taxii.example.com/privkey.pem;

    location / {
        proxy_pass http://<OPENCTI_IP>:8080;

        proxy_cache                 off;
        proxy_buffering             off;

        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        proxy_read_timeout 300;
        proxy_send_timeout 300;

        chunked_transfer_encoding   off;
    }
}
```

Step 6 – Docker Compose (NGINX)
-------------------------------

`vi docker-compose.yml`

```
services:
  nginx:
    image: nginx:stable
    container_name: nginx
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./conf:/etc/nginx/conf.d:ro
      - ./letsencrypt:/etc/letsencrypt:ro
      - ./www:/var/www/certbot:ro

  certbot:
    image: certbot/certbot
    container_name: certbot
    network_mode: host
    volumes:
      - ./letsencrypt:/etc/letsencrypt
      - ./www:/var/www/certbot
```

🔐 Step 7 – Obtain SSL Certificate
----------------------------------

### First certificate issue

Temporarily comment out the HTTPS server block until the cert exists, then start NGINX:

`docker compose up -d nginx`

Request the cert:

```
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path /var/www/certbot \
  -d taxii.example.com \
  --email you@example.com \
  --agree-tos \
  --no-eff-email
```

Then uncomment HTTPS and reload:

▶️ Step 8 – Restart Full Stack
-------------------------------

```
docker compose up -d
docker exec nginx nginx -s reload
```

Step 9 – Test TAXII
-------------------

```
curl -s https://taxii.example.com/taxii2/ \
  -H "Authorization: Bearer <OPENCTI_TOKEN>" \
  -H "Accept: application/taxii+json;version=2.1"
```

🔁 Step 10 – Certificate Auto-Renew (Optional)
-----------------------------------------------

Use systemd timer:

#### 🔹 1. Create the renewal script

`sudo vi /usr/local/sbin/renew-nginx-certs.sh`

Paste:

```
#!/bin/bash
set -e

cd /opt/nginx || exit 1

OUTPUT=$(docker compose run --rm certbot renew --webroot --webroot-path /var/www/certbot)

echo "$OUTPUT"

if echo "$OUTPUT" | grep -q "Congratulations"; then
    echo "Cert renewed, reloading nginx"
    docker exec nginx nginx -s reload
else
    echo "No renewal needed"
fi
```

Make it executable:

`sudo chmod +x /usr/local/sbin/renew-nginx-certs.sh`

#### 🔹 2. Create systemd service

`sudo vi /etc/systemd/system/nginx-cert-renew.service`

```
[Unit]
Description=Renew Let's Encrypt certs and reload NGINX
Wants=docker.service
After=docker.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/renew-nginx-certs.sh
```

#### 🔹 3. Create systemd timer

`sudo vi /etc/systemd/system/nginx-cert-renew.timer`

```
[Unit]
Description=Run cert renewal twice daily

[Timer]
OnCalendar=*-*-* 03,15:17:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

👉 This runs:

* Twice a day (03:17 and 15:17)
* With jitter (avoids hitting Let’s Encrypt at exact times)

#### 🔹 4. Enable the timer

```
sudo systemctl daemon-reexec
sudo systemctl daemon-reload

sudo systemctl enable --now nginx-cert-renew.timer
```

#### 🔹 5. Verify

`systemctl list-timers | grep nginx-cert`

#### 🔹 6. Test manually (important)

`sudo systemctl start nginx-cert-renew.service`

Check logs:

`journalctl -u nginx-cert-renew.service -n 50`

#### 🔹 7. What this gives you

    ✅ Automatic cert renewal
    ✅ NGINX reload after renewal
    ✅ Logging via systemd journal
    ✅ No cron dependency
    ✅ Survives reboots (Persistent=true)

🔐 Security Notes
-----------------

    ❌ No NGINX Basic Auth (OpenCTI handles auth)

    ✅ Only expose:
      - TCP 80
      - TCP 443
    🔒 Keep OpenCTI (8080) private

⚡ Performance Notes
---------------------

    * Ensure OpenCTI has sufficient RAM (≥8GB)

✅ Validation Checklist
------------------------

    * HTTPS loads without warnings
    * /taxii2/ responds
    * OpenCTI UI loads correctly
    * Connectors ingest data

🧠 Final Architecture Summary

    * TLS termination at NGINX
    * Single authentication (OpenCTI)
    * Backend isolated

🚧 Known Considerations

    * Large JS bundle (~50MB uncompressed)
    * First load may be slower without cache

📌 Optional Enhancements

    * Rate limiting on `/taxii2/`
    * IP allowlisting
    * mTLS for TAXII clients
    * AWS ALB in front of NGINX

🎯 Outcome

This deployment provides:

    * Secure TAXII 2.1 endpoint
    * Scalable architecture
    * Clean separation of concerns
