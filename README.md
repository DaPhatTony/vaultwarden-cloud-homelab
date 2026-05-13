# Vaultwarden Password Manager Deployment

This repository documents the infrastructure-as-code and deployment steps for a self-hosted Vaultwarden (Bitwarden-compatible) password manager. It is designed to run securely on a Raspberry Pi 4 utilizing containerization, zero-trust networking, and strict secret management principles.

## Architecture & Security Overview
As part of a broader Cybersecurity and Information Assurance homelab portfolio, this deployment emphasizes security best practices:
* **Secret Decoupling:** Administrative keys are isolated in a local `.env` file, preventing credential exposure in version control.
* **Network Isolation:** Bound to port `8082` to cleanly integrate alongside existing DNS sinkhole and media server deployments without conflicts.
* **Zero-Trust Access:** Remote access is strictly handled via a Tailscale VPN overlay network, keeping the vault completely off the public internet.

## System Prerequisites
* **Hardware:** Raspberry Pi 4
* **OS:** Raspberry Pi OS (64-bit)
* **Dependencies:** Docker Engine, Docker Compose V2, OpenSSL
* **Network:** Tailscale installed and authenticated

---

## Step-by-Step Installation Guide

Follow these instructions to deploy the Vaultwarden container from scratch on the host machine.

### Step 1: Initialize the Environment
Create an isolated directory for the application and initialize version control to track infrastructure changes.
```bash
mkdir ~/vaultwarden-homelab
cd ~/vaultwarden-homelab
git init
```

### Step 2: Generate the Administrative Token
Vaultwarden requires a highly secure cryptographic token to access the backend administration panel. Generate this locally and store it as an environment variable.
```bash
openssl rand -base64 48 > .env
```
Open the `.env` file (`nano .env`) and format the generated string exactly like this:
```text
ADMIN_TOKEN=your_generated_random_string_here
```

### Step 3: Configure the Docker Infrastructure
Create the deployment configuration. This tells the host how to map the network ports and mount the persistent storage volume.
```bash
nano docker-compose.yml
```
Paste the following service definition:
```yaml
version: '3.8'

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      - WEBSOCKET_ENABLED=true
      - ADMIN_TOKEN=${ADMIN_TOKEN}
    volumes:
      - ./vw-data:/data
    ports:
      - 8082:80
```

### Step 4: Execute the Deployment
Pull the latest image and launch the Vaultwarden container in detached mode.
```bash
docker compose up -d
```

---

## Accessing the Application
Once the container status is healthy, the application can be accessed securely across the Tailscale network.

* **Primary Web Vault:** `http://(YOUR_TAILSCALE_IP_HERE):8082`
* **Admin Dashboard:** `http://(YOUR_TAILSCALE_IP_HERE):8082/admin` *(Requires the token generated in Step 3)*

## Maintenance & Backups
All encrypted passwords and user data are persistently stored in the local `./vw-data` directory. For disaster recovery, ensure this specific directory is securely backed up.
