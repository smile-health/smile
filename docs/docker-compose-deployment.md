# Smile Health Docker Compose Deployment Guide

This guide explains how to deploy the Smile Health application using Docker Compose.

## Prerequisites

Before starting the deployment, ensure you have the following installed:

- **Docker Engine** (v20.10+) - [Installation Guide](https://docs.docker.com/engine/install/)
- **Docker Compose** (v2.0+) - [Installation Guide](https://docs.docker.com/compose/install/)
- **Git** - For cloning the repository
- **Bash** - For running setup scripts (Linux/macOS/WSL)

### Clone the Repository with Submodules

This project uses Git submodules. Clone with all submodules:

```bash
git clone --recursive https://github.com/smile-health/smile.git
```

If you already cloned without submodules, initialize them:

```bash
git submodule update --init --recursive
```

To update submodules later:

```bash
git submodule update --recursive --remote
```

### System Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 50 GB SSD | 100 GB SSD |
| Network | Stable internet | High-speed connection |

## Technology Stack

| Layer | Technology / Framework | Description |
|------|------------------------|-------------|
| **Frontend Web** | React.js, Next.js | Single Page Application (SPA) with responsive user interface |
| **Backend** | Bun, Hono | Bun as backend runtime, Hono as lightweight framework |
| **Language Translation** | Tolgee | Dynamic multi-language support based on user preference |
| **Mobile Frontend** | React Native | Cross-platform mobile application development |
| **Database (OLTP)** | MySQL 8 | Primary transactional database for application data |
| **Database (OLAP)** | ClickHouse | Analytical data processing for reporting |
| **Authentication** | Keycloak, OAuth2 | Identity and access management |
| **Dashboard** | Metabase | Data visualisation and analytics |
| **Object Storage** | MinIO | File storage, caching, and backup |
| **Message Queue** | RabbitMQ | Asynchronous messaging between services |
| **Cache** | Redis | In-memory caching for performance optimisation |
| **Notification** | Firebase | Email and WhatsApp notifications |
| **Infrastructure** | Docker | Container orchestration and infrastructure management |
| **Reverse Proxy** | Traefik | Edge router and load balancer |

### Backend Services (`@backend/apps/`)

| Service | Framework | Language | Port | Description |
|---------|-----------|----------|------|-------------|
| `core` | Hono | TypeScript/Node.js | 4000 | Core API service |
| `auth-service` | Hono | TypeScript/Node.js | 4003 | Authentication & authorization |
| `main` | Hono | TypeScript/Node.js | 4004 | Main business logic API |
| `warehouse-service` | Hono | TypeScript/Node.js | 4008 | Data warehouse & reporting |
| `sync-service` | Hono | TypeScript/Node.js | - | Data synchronization |
| `notification` | Hono | TypeScript/Node.js | - | Notification service |

**Backend Tech Stack:**
- **Runtime:** Bun (primary) / Node.js 18+
- **Framework:** Hono (lightweight)
- **Package Manager:** pnpm
- **Build Tool:** Turbo
- **Database ORM:** Kysely

### Frontend Application (`@frontend/apps/web`)

| Application | Framework | Language | Port | Description |
|-------------|-----------|----------|------|-------------|
| `web` | Next.js 14 | React/TypeScript | 4001 | Main web application |

**Frontend Tech Stack:**
- **Framework:** Next.js 14.2.30
- **UI Library:** React 18.3.1
- **Styling:** TailwindCSS 1.9.6
- **Package Manager:** npm 9.8.1

### Infrastructure Services

| Service | Image | External Port | Internal Port | Purpose |
|---------|-------|---------------|---------------|---------|
| MySQL | mysql:8 | 4306 | 3306 | Primary database |
| Redis | redis:7 | 4379 | 6379 | Caching & sessions |
| RabbitMQ | rabbitmq:4-management | 4672, 4673 | 5672, 15672 | Message queue |
| Keycloak | quay.io/keycloak/keycloak:23.0 | 4080 | 8080 | Identity management |
| MinIO | quay.io/minio/minio | 4900, 4901 | 9000, 9001 | Object storage |
| Traefik | traefik:v3.0 | 80, 443, 8080 | - | Reverse proxy & load balancer |

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Traefik (80/443)                           │
│                         Reverse Proxy / Load Balancer                   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
           ┌────────────────────────┼────────────────────────┐
           │                        │                        │
           ▼                        ▼                        ▼
┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│   Frontend (4001)   │  │   Backend APIs      │  │   Keycloak (4080)   │
│   Next.js Web App   │  │   (4000/4003/4004)  │  │   Identity Provider │
└─────────────────────┘  └─────────────────────┘  └─────────────────────┘
                                │
           ┌────────────────────┼────────────────────┐
           │                    │                    │
           ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   MySQL (4306)  │  │  Redis (4379)   │  │ RabbitMQ (4672) │
│  Primary DB     │  │    Cache        │  │  Message Queue  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
           │
           ▼
┌─────────────────┐  ┌─────────────────┐
│  MinIO (4900)   │  │  Docker Network │
│ Object Storage  │  │  smile-network  │
└─────────────────┘  └─────────────────┘
```

## Deployment Steps

### 1. Environment Configuration

Navigate to the Docker environment directory and configure the environment files:

```bash
cd deployment/docker/env
```

#### 1.1 Main Environment Variables (`.env`)

Create `.env` from the example file:

```bash
cp .env.example .env
```

Edit `.env` with your values:

```env
# Database Configuration
MYSQL_ROOT_PASSWORD=your_secure_root_password
MYSQL_DATABASE=smile_health
MYSQL_USER=smileuser
MYSQL_PASSWORD=your_secure_password

# RabbitMQ Configuration
RABBITMQ_USER=admin
RABBITMQ_PASS=your_secure_rabbit_password

# Keycloak Configuration
KEYCLOAK_DB_PASSWORD=your_keycloak_db_password
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=your_keycloak_admin_password

# MinIO Configuration
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=your_secure_minio_password

# ClickHouse (Optional)
CLICKHOUSE_DB=default
CLICKHOUSE_USER=default
CLICKHOUSE_PASSWORD=
```

#### 1.2 Service-Specific Environment Files

Each backend service has its own environment file:

```bash
# Copy all service environment examples
for file in *.env.example; do
    cp "$file" "${file%.example}"
done
```

Configure each service environment file as needed:
- `core.env` - Core API service
- `auth-service.env` - Authentication service
- `main.env` - Main API service
- `sync-service.env` - Sync service
- `warehouse-service.env` - Warehouse service
- `notification.env` - Notification service

#### 1.3 Frontend Environment Variables (`frontend-web.env`)

```bash
cp frontend-web.env.example frontend-web.env
```

Edit `frontend-web.env` with your API endpoints:

```env
NODE_ENV="production"
SITE_TITLE="SMILE Health"

# API Endpoints - Update for production
API_URL="https://your-domain.com"
API_AUTH_URL="https://your-domain.com"
API_ASSET_URL="https://your-domain.com"
API_REPORT_URL="https://your-domain.com"
API_BIG_DATA_URL="https://your-domain.com/warehouse-report"

# Firebase Configuration (if using)
NEXT_PUBLIC_FIREBASE_API_KEY=""
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=""
# ... other firebase config
```

### 2. Run Setup Script

Navigate to the compose directory and run the setup script:

```bash
cd deployment/docker/compose
bash setup.sh
```

#### Setup Script Options

```bash
# Standard setup (builds both backend and frontend)
bash setup.sh

# Rebuild all images
bash setup.sh --rebuild

# Use GitHub Container Registry images (no local build)
bash setup.sh --ghcr

# Backend only
bash setup.sh --backend-only

# Frontend only
bash setup.sh --frontend-only
```

The setup script performs the following steps:
1. Creates environment files from examples (if not exist)
2. Copies `.npmrc` to service directories
3. Creates volume directories for persistent data
4. Creates Docker network `smile-network`
5. Builds and starts all services
6. Creates additional databases (smile_health_mapping, smile_health_notification)
7. Validates health checks

After successful setup, you should see:

```
==========================================
✓ Backend Setup Complete!
==========================================

Service Endpoints:
  Core API:       http://localhost:4000
  Auth Service:   http://localhost:4003
  Main Service:   http://localhost:4004
  Warehouse:      http://localhost:4008

Infrastructure:
  MySQL:          localhost:4306
  Redis:          localhost:4379
  RabbitMQ:       localhost:4672
  RabbitMQ UI:    http://localhost:4673
  Keycloak:       http://localhost:4080
  MinIO Console:  http://localhost:4901
```

### 3. Database Migration

Once setup completes successfully, run database migrations:

```bash
bash migrate.sh
```

This will:
- Migrate `core` service database
- Seed and migrate `main` service database
- Migrate `sync-service` database

To seed initial data:

```bash
bash migrate.sh seed
```

### 4. Access the Application

#### Local Development

After deployment, access the services at:

| Service | Local URL |
|---------|-----------|
| Web Application | http://localhost:4001 |
| Core API | http://localhost:4000 |
| Auth Service | http://localhost:4003 |
| Main Service | http://localhost:4004 |
| Warehouse | http://localhost:4008 |
| Keycloak Admin | http://localhost:4080/admin |
| RabbitMQ Management | http://localhost:4673 |
| MinIO Console | http://localhost:4901 |
| Traefik Dashboard | http://localhost:8080 |

Default credentials:
- **Keycloak:** admin / admin (or your configured password)
- **RabbitMQ:** admin / admin (or your configured password)
- **MinIO:** minioadmin / minioadmin (or your configured password)

### 5. Server Deployment (Production)

For production server deployment, additional configuration is required:

#### 5.1 Update API URLs

In `frontend-web.env`, update all API URLs to point to your server:

```env
API_URL="https://api.your-domain.com"
API_AUTH_URL="https://api.your-domain.com"
API_ASSET_URL="https://api.your-domain.com"
```

#### 5.2 Configure Traefik for SSL/TLS

Update `traefik-dynamic.yml` with your domain and SSL certificates:

```yaml
http:
  routers:
    web-secure:
      rule: "Host(`your-domain.com`)"
      tls:
        certResolver: letsencrypt
      service: frontend-web
```

#### 5.3 DNS Configuration

Point your domain DNS records to the server:

```
A Record: your-domain.com -> SERVER_IP
A Record: api.your-domain.com -> SERVER_IP
```

#### 5.4 Reverse Proxy Setup

Traefik acts as the reverse proxy. Configure labels in compose files for routing:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.myapp.rule=Host(`your-domain.com`)"
  - "traefik.http.routers.myapp.entrypoints=websecure"
  - "traefik.http.routers.myapp.tls.certresolver=letsencrypt"
```

#### 5.5 SSL Certificates

For production SSL, configure Let's Encrypt in `traefik.yml`:

```yaml
certificatesResolvers:
  letsencrypt:
    acme:
      email: your-email@domain.com
      storage: /acme.json
      tlsChallenge: {}
```

## Docker Compose Files Reference

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Main compose entry point |
| `compose-tools.yml` | Infrastructure services (MySQL, Redis, RabbitMQ, etc.) |
| `compose-services.yml` | Backend microservices (local build) |
| `compose-services-ghcr.yml` | Backend services (GHCR images) |
| `compose-frontend.yml` | Frontend web application |
| `compose-data.yml` | Data streaming services (Kafka, ClickHouse, RisingWave) |

## Useful Commands

```bash
# View all running containers
docker compose ps

# View logs
docker compose logs -f

# View specific service logs
docker compose logs -f core

# Stop all services
docker compose down

# Stop and remove volumes (WARNING: data loss)
docker compose down -v

# Restart a service
docker compose restart core

# Shell into a container
docker exec -it core sh

# Check service health
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Health}}"
```

## Troubleshooting

### Issue: Services fail to start

**Solution:** Check environment files are properly configured:
```bash
cd deployment/docker/env
ls -la *.env  # Ensure all .env files exist
```

### Issue: Database connection errors

**Solution:** Verify MySQL is healthy and credentials are correct:
```bash
docker compose ps mysql
docker logs mysql
```

### Issue: Frontend cannot connect to API

**Solution:** Check `API_URL` in `frontend-web.env` matches your server address.

### Issue: Permission denied on scripts

**Solution:** Make scripts executable:
```bash
chmod +x setup.sh migrate.sh
```

## Data Persistence

All data is persisted in Docker volumes mapped to:

```
deployment/docker/volumes/
├── mysql/              # MySQL data
├── redis/              # Redis data
├── rabbitmq/           # RabbitMQ data
├── rabbitmq-logs/      # RabbitMQ logs
├── keycloak-postgres/  # Keycloak database
├── minio/              # MinIO object storage
└── ...
```

## Security Considerations

1. **Change all default passwords** in environment files
2. **Use strong passwords** for database and service accounts
3. **Enable SSL/TLS** for production deployments
4. **Restrict port access** using firewall rules
5. **Regular backups** of volume data
6. **Keep Docker images updated** for security patches

## Support

For issues or questions:
- Check service logs: `docker compose logs -f [service-name]`
- Review the [QUICKSTART.md](QUICKSTART.md) for quick reference
- Consult [deployment-guide.md](../../smile/docs/deployment-guide.md) for additional guidance
