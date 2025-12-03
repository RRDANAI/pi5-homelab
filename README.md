# Pi5 Homelab

A complete Raspberry Pi 5 homelab setup for running n8n workflow automation with secure remote access via Tailscale and public webhook endpoints.

## Overview

This project provides a production-ready n8n automation platform running on a Raspberry Pi 5 with:

- **n8n**: Self-hosted workflow automation tool with queue-based execution
- **PostgreSQL 16**: Persistent database storage
- **Redis**: Queue management for distributed workers
- **Caddy**: Automatic HTTPS reverse proxy
- **Tailscale**: Secure VPN access for private UI

## Hardware

- Raspberry Pi 5 (8GB RAM)
- Pironman 5 MAX case
- WD Black SN750SE 1TB NVMe Gen 4

## Architecture

```
Internet
  │
  ├─► n8n.yourdomain.com (Public - Webhooks only)
  │        │
  │        └─► Caddy (HTTPS via Let's Encrypt)
  │                 │
  └─► tailnetdomain.ts.net (Private - Full UI/API)
           │
           └─► Caddy (HTTPS via Tailscale)
                    │
                    └─► n8n (localhost:5678)
                          │
                          ├─► PostgreSQL (Database)
                          └─► Redis (Queue)
```

## Features

- **Dual Access Model**: Private UI access via Tailscale, public webhooks only via DNS
- **Scalable Architecture**: Queue-based execution with dedicated worker process
- **Automatic HTTPS**: Let's Encrypt for public domain, Tailscale certs for private
- **Persistent Storage**: PostgreSQL for workflows and execution data
- **High Availability**: Health checks for all services with automatic restarts

## Prerequisites

- Raspberry Pi 5 with Raspberry Pi OS
- Docker and Docker Compose installed
- Tailscale account and client installed
- Public domain name with DNS configured (for webhooks)
- Minimum 4GB RAM recommended

## Installation

### 1. Clone the Repository

```bash
git clone <your-repo-url>
cd pi5-homelab
```

### 2. Configure Environment Variables

Create a `.env` file in the project root:

```bash
# PostgreSQL Configuration
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your_secure_password
POSTGRES_DB=n8n
POSTGRES_NON_ROOT_USER=n8n
POSTGRES_NON_ROOT_PASSWORD=your_n8n_password

# n8n Configuration
ENCRYPTION_KEY=your_encryption_key_here
```

Generate a secure encryption key:
```bash
openssl rand -base64 32
```

### 3. Set Up Template Files

Copy and configure the template files:

```bash
# Docker Compose
cp docker-compose.yml.template docker-compose.yml

# Caddy configuration
sudo cp Caddyfile.template /etc/caddy/Caddyfile

# Tailscale configuration
sudo cp tailscaled.template /etc/default/tailscaled
```

### 4. Update Configuration

Edit the following in your configuration files:

- `Caddyfile`: Replace domains with your actual Tailscale and public domains
- `docker-compose.yml`: Update `WEBHOOK_URL` and `N8N_HOST` with your public domain

### 5. Start Services

```bash
# Start n8n stack
docker compose up -d

# Check service status
docker compose ps

# View logs
docker compose logs -f
```

### 6. Configure Tailscale

```bash
# Allow Caddy to access Tailscale certificates
sudo tailscale cert <your-tailscale-domain>

# Restart Caddy
sudo systemctl restart caddy
```

## Configuration Details

### Caddy Reverse Proxy

The Caddy configuration provides:

- **Private Access** (`tailnetdomain.ts.net`): Full n8n UI and API access via Tailscale
- **Public Access** (`n8n.yourdomain.com`): Webhook endpoints only, UI blocked

### n8n Workers

The setup includes:
- Main n8n instance handling UI and workflow triggers
- Dedicated worker process for executing workflow steps
- Redis-based queue for distributed execution

### Database Initialization

The `init-data.sh` script automatically creates a non-root PostgreSQL user for n8n during first startup.

## Usage

### Access n8n UI

Connect to Tailscale, then navigate to:
```
https://tailnetdomain.ts.net
```

### Webhook Endpoints

Public webhooks are accessible at:
```
https://n8n.yourdomain.com/webhook/*
```

### Managing Services

```bash
# Stop all services
docker compose down

# Restart specific service
docker compose restart n8n

# View logs for specific service
docker compose logs -f n8n-worker

# Update images
docker compose pull
docker compose up -d
```

## Security Considerations

- Private UI access restricted to Tailscale network only
- Public domain blocks all non-webhook endpoints (403 Forbidden)
- All traffic encrypted via HTTPS (automatic certificate management)
- Database credentials stored in environment variables (not committed to git)
- Non-root database user with minimal required privileges

## File Structure

```
.
├── Caddyfile.template         # Caddy reverse proxy configuration
├── docker-compose.yml.template # Docker services definition
├── init-data.sh               # PostgreSQL initialization script
├── tailscaled.template        # Tailscale daemon configuration
└── README.md                  # This file
```

## Troubleshooting

### n8n won't start
Check database connection:
```bash
docker compose logs postgres
docker compose logs n8n
```

### Tailscale domain not accessible
Verify Caddy has certificate access:
```bash
sudo ls -la /var/lib/tailscale/certs/
sudo systemctl status caddy
```

### Webhooks not working
Check public domain DNS and Caddy configuration:
```bash
sudo caddy validate --config /etc/caddy/Caddyfile
curl -I https://n8n.yourdomain.com/webhook-test
```

### Worker not processing executions
Check Redis connection and worker logs:
```bash
docker compose logs redis
docker compose logs n8n-worker
```

## Resources

- [n8n Documentation](https://docs.n8n.io/)
- [Caddy Documentation](https://caddyserver.com/docs/)
- [Tailscale Documentation](https://tailscale.com/kb/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

## License

[Specify your license]

## Author

[Your name/contact]
