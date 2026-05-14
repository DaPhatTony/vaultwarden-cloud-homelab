# Vaultwarden Password Manager: Secure Deployment Guide

This repository contains the infrastructure-as-code and deployment steps for a self-hosted Vaultwarden (Bitwarden-compatible) password manager. It is designed to run securely on a Raspberry Pi 4 utilizing containerization, zero-trust networking, and strict secret management principles.

## Architecture & Security Overview
As part of a broader Cybersecurity and Information Assurance homelab portfolio, this deployment emphasizes security best practices:
* **Encryption at Rest:** All vault data is encrypted using AES-256.
* **Encryption in Transit:** Full SSL/TLS termination using Tailscale MagicDNS and automated certificates to satisfy the Subtle Crypto API requirements.
* **Administrative Security:** The `ADMIN_TOKEN` is protected using an Argon2id PHC string, ensuring that even if the configuration file is compromised, the plain-text password remains secure.
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

### Step 2: Establish the Security Perimeter
To ensure encrypted databases, administrative tokens, and SSL keys are never pushed to a public repository, create a `.gitignore` file immediately.
```bash
nano .gitignore
```
Add the following exclusion rules:
```text
.env
vw-data/
certs/
```

### Step 3: SSL/TLS Certificate Generation
Utilize Tailscale to provision valid certificates for the internal MagicDNS domain:
```bash
mkdir certs
sudo tailscale cert --cert-file ./certs/tls.crt --key-file ./certs/tls.key <your-tailscale-domain>
```

### Step 4: Configure the Docker Infrastructure
Create the deployment configuration. This tells the host how to map the network ports, mount the persistent storage volume, and inject the `ROCKET_TLS` structure.
```bash
nano docker-compose.yml
```
Paste the following service definition:
```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      - ADMIN_TOKEN=${ADMIN_TOKEN}
      - WEBSOCKET_ENABLED=true
      - ROCKET_TLS={certs="/ssl/tls.crt",key="/ssl/tls.key"}
    volumes:
      - ./vw-data:/data
      - ./certs:/ssl:ro
    ports:
      - 8082:80
```

### Step 5: Execute the Deployment
Pull the latest image and launch the Vaultwarden container in detached mode.
```bash
docker compose up -d
```

### Step 6: Security Initialization (Argon2 Hash)
Generate an Argon2 hash for the admin dashboard to replace insecure plain-text tokens.
```bash
docker exec -it vaultwarden /vaultwarden hash
```
Escape all `$` characters with `$$` and add it to your `.env` file:
```bash
nano .env
# Format: ADMIN_TOKEN=$$argon2id$$v=19$$...
```
Restart the container to apply the secure token:
```bash
docker compose down
docker compose up -d
```

---

## Accessing the Application
Once the container status is healthy, the application can be accessed securely across the Tailscale network.

* **Primary Web Vault:** `https://<tailscale-machine-name>.<tailnet-id>.ts.net:8082`
* **Admin Dashboard:** `https://<tailscale-machine-name>.<tailnet-id>.ts.net:8082/admin` *(Requires the Argon2-hashed token)*

## Maintenance & Backups
All encrypted passwords and user data are persistently stored in the local `./vw-data` directory. For disaster recovery, ensure this specific directory is backed up securely.
