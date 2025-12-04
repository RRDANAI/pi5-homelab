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
├── Caddyfile.template              # Caddy reverse proxy configuration
├── docker-compose.yml.template      # Main n8n stack definition
├── docker-compose.logging.yml       # Logging stack (Loki/Promtail/Grafana)
├── init-data.sh                    # PostgreSQL initialization script
├── tailscaled.template             # Tailscale daemon configuration
├── config/
│   ├── loki/
│   │   └── loki-config.yml        # Loki configuration with retention
│   ├── promtail/
│   │   └── promtail-config.yml    # Log collection and filtering
│   └── grafana/
│       └── provisioning/
│           ├── datasources/       # Auto-configure Loki datasource
│           └── dashboards/        # Pre-built n8n logs dashboard
└── README.md                       # This file
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

## Centralized Logging (Optional)

This project includes an optional Loki + Promtail + Grafana stack for centralized logging and monitoring of your n8n homelab.

### Logging Features

- **Loki**: Efficient log aggregation optimized for Raspberry Pi
- **Promtail**: Automatic Docker container log collection with intelligent filtering
- **Grafana**: Beautiful dashboards for log visualization and search
- **Smart Filtering**: Automatically filters out noisy Redis/PostgreSQL health checks
- **Retention Policies**: 14 days for all logs, 30+ days for errors/warnings
- **Tailscale Access**: Secure access to Grafana UI via your Tailscale domain

### Setup Logging Stack

#### 1. Update Environment Variables

Add Grafana configuration to your `.env` file:

```bash
# Grafana Configuration
GF_SERVER_ROOT_URL=https://logs.your-hostname.your-tailnet.ts.net
GF_SERVER_DOMAIN=logs.your-hostname.your-tailnet.ts.net
GF_ADMIN_USER=admin
GF_ADMIN_PASSWORD=your_secure_grafana_password
```

#### 2. Update Caddyfile

Add the Grafana subdomain to your Caddyfile (replace with your actual Tailscale domain):

```
logs.your-hostname.your-tailnet.ts.net {
    reverse_proxy 127.0.0.1:3000
}
```

Then reload Caddy:
```bash
sudo systemctl reload caddy
```

#### 3. Start Logging Stack

Launch both the main n8n stack and logging stack together:

```bash
# Start everything
docker compose -f docker-compose.yml -f docker-compose.logging.yml up -d

# Check status
docker compose -f docker-compose.yml -f docker-compose.logging.yml ps

# View logging stack logs
docker compose -f docker-compose.logging.yml logs -f
```

#### 4. Access Grafana

Navigate to your Grafana instance via Tailscale:
```
https://logs.your-hostname.your-tailnet.ts.net
```

Login with the credentials from your `.env` file (default: admin/admin).

### Using the Logging Dashboard

The included "n8n Homelab Logs" dashboard provides:

1. **Errors & Warnings Panel**: Shows only ERROR and WARN level logs from n8n
2. **PostgreSQL Logs**: Database logs (health checks filtered out)
3. **Redis Logs**: Queue logs (connection chatter filtered out)
4. **All n8n Logs**: Complete n8n container logs for detailed debugging

### Log Queries

Some useful LogQL queries for Grafana Explore:

```logql
# All errors across all containers
{container=~".+"} |~ "ERROR"

# n8n workflow execution errors
{container=~"n8n.*"} |~ "execution.*failed"

# PostgreSQL connection issues
{container="postgres"} |~ "connection|error"

# Redis errors only (not INFO)
{container="redis"} |~ "ERROR|ERR"

# Logs from last hour with "webhook" keyword
{container=~"n8n.*"} |~ "webhook" | json | line_format "{{.message}}"
```

### Log Retention

The logging stack is configured with retention policies optimized for your 1TB SSD:

- **Standard logs**: 14 days (debugging recent workflows)
- **Errors/warnings**: Kept longer via label-based retention
- **Automatic cleanup**: Compactor runs every 10 minutes
- **Write optimization**: Chunking configured to minimize SSD wear

### Managing the Logging Stack

```bash
# Stop only logging services (keeps n8n running)
docker compose -f docker-compose.logging.yml down

# Restart Promtail after config changes
docker compose -f docker-compose.logging.yml restart promtail

# Check Loki storage usage
docker exec loki du -sh /loki

# View real-time logs being collected
docker compose -f docker-compose.logging.yml logs -f promtail

# Remove all logging data and start fresh
docker compose -f docker-compose.logging.yml down -v
docker compose -f docker-compose.logging.yml up -d
```

### Filtering Out Noisy Logs

The Promtail configuration (`config/promtail/promtail-config.yml`) includes filters for:

- Redis PING commands and connection chatter
- PostgreSQL pg_isready health checks
- PostgreSQL connection lifecycle logs
- Database checkpoint messages

You can add more filters by editing the configuration and restarting Promtail.

### Troubleshooting Logging

#### Grafana shows "No data"
```bash
# Check if Loki is receiving logs
curl http://localhost:3100/loki/api/v1/labels

# Check Promtail is running and connected
docker compose -f docker-compose.logging.yml logs promtail
```

#### Logs not appearing
```bash
# Verify Promtail can access Docker socket
docker compose -f docker-compose.logging.yml exec promtail ls -la /var/run/docker.sock

# Check Promtail positions file
docker compose -f docker-compose.logging.yml exec promtail cat /tmp/positions.yaml
```

#### High disk usage
```bash
# Check Loki storage
docker exec loki du -sh /loki/*

# Manually trigger compaction (if needed)
curl -X POST http://localhost:3100/loki/api/v1/delete?match={container=~".+"}
```

## Resources

- [n8n Documentation](https://docs.n8n.io/)
- [Caddy Documentation](https://caddyserver.com/docs/)
- [Tailscale Documentation](https://tailscale.com/kb/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Grafana Loki Documentation](https://grafana.com/docs/loki/latest/)
- [LogQL Query Language](https://grafana.com/docs/loki/latest/logql/)

## License

[Specify your license]

## Author

[Your name/contact]
