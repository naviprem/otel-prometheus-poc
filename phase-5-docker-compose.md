# Phase 5: Docker Compose Orchestration

## Overview
Create a unified Docker Compose configuration that orchestrates all components (data planes, control plane, and supporting services) into a single deployable stack.

---

## Step 1: Project Root Structure

### 1.1 Verify Project Structure
```
otel-prometheus-poc/
├── dataplane/
│   ├── api/
│   │   ├── Dockerfile
│   │   └── ...
│   └── otel-collector/
│       ├── config-prod.yaml
│       ├── config-dev.yaml
│       └── config-qa.yaml
├── controlplane/
│   ├── prometheus/
│   │   └── prometheus.yml
│   ├── thanos/
│   │   └── bucket-config.yaml
│   └── dashboard/
│       ├── Dockerfile
│       └── ...
├── load-generator/
│   └── (to be created in Phase 6)
├── docker-compose.yml          # Main file (this phase)
├── .env                        # Environment variables
└── README.md
```

---

## Step 2: Create Environment Configuration

### 2.1 Create `.env` File
```bash
# .env - Environment variables for Docker Compose

# Version tags
API_VERSION=1.0.0
OTEL_COLLECTOR_VERSION=0.91.0
PROMETHEUS_VERSION=latest
THANOS_VERSION=v0.32.5
MINIO_VERSION=latest

# Network
NETWORK_NAME=otel-network

# Prometheus
PROMETHEUS_PORT=9090
PROMETHEUS_RETENTION=15d

# MinIO (S3)
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
MINIO_PORT=9000
MINIO_CONSOLE_PORT=9001

# Dashboard
DASHBOARD_PORT=3000

# Data Plane - Production
API_PROD_PORT=8081
OTEL_PROD_GRPC_PORT=4317
OTEL_PROD_HTTP_PORT=4318
OTEL_PROD_PROMETHEUS_PORT=8889
OTEL_PROD_METRICS_PORT=8888

# Data Plane - Development
API_DEV_PORT=8082
OTEL_DEV_GRPC_PORT=4327
OTEL_DEV_HTTP_PORT=4328
OTEL_DEV_PROMETHEUS_PORT=8890
OTEL_DEV_METRICS_PORT=8891

# Data Plane - QA
API_QA_PORT=8083
OTEL_QA_GRPC_PORT=4337
OTEL_QA_HTTP_PORT=4338
OTEL_QA_PROMETHEUS_PORT=8892
OTEL_QA_METRICS_PORT=8893

# Application Settings
ERROR_RATE_PROD=0.01
ERROR_RATE_DEV=0.05
ERROR_RATE_QA=0.10
```

---

## Step 3: Create Main Docker Compose File

### 3.1 Create `docker-compose.yml`
```yaml
version: '3.8'

# ============================================================================
# OTEL Prometheus PoC - Complete Stack
# ============================================================================
#
# This compose file deploys:
# - 3 Data Planes (prod, dev, qa)
# - Control Plane (Prometheus, Dashboard)
# - Optional: Thanos + MinIO for long-term storage
#
# Usage:
#   docker-compose up -d                 # Start all services
#   docker-compose up -d --scale qa=0    # Start without QA environment
#   docker-compose down                  # Stop all services
#
# ============================================================================

services:
  # ==========================================================================
  # DATA PLANE - PRODUCTION
  # ==========================================================================

  otel-collector-prod:
    image: otel/opentelemetry-collector-contrib:${OTEL_COLLECTOR_VERSION}
    container_name: otel-collector-prod
    command: ["--config=/etc/otel-collector/config.yaml"]
    volumes:
      - ./dataplane/otel-collector/config-prod.yaml:/etc/otel-collector/config.yaml:ro
    ports:
      - "${OTEL_PROD_GRPC_PORT}:4317"
      - "${OTEL_PROD_HTTP_PORT}:4318"
      - "${OTEL_PROD_PROMETHEUS_PORT}:8889"
      - "${OTEL_PROD_METRICS_PORT}:8888"
    environment:
      - ENVIRONMENT=prod
      - DATA_PLANE_ID=dp-prod-01
    networks:
      - otel-network
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8888/"]
      interval: 10s
      timeout: 5s
      retries: 3
    restart: unless-stopped

  api-prod:
    build:
      context: ./dataplane/api
      dockerfile: Dockerfile
    image: otel-api-service:${API_VERSION}
    container_name: api-prod
    ports:
      - "${API_PROD_PORT}:8080"
    environment:
      - PORT=8080
      - ENVIRONMENT=prod
      - DATA_PLANE_ID=dp-prod-01
      - OTEL_ENDPOINT=http://otel-collector-prod:4318
      - SERVICE_NAME=api-service
      - SERVICE_VERSION=${API_VERSION}
      - ERROR_RATE=${ERROR_RATE_PROD}
      - ENABLE_SLOW_REQUESTS=true
    depends_on:
      otel-collector-prod:
        condition: service_healthy
    networks:
      - otel-network
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # ==========================================================================
  # DATA PLANE - DEVELOPMENT
  # ==========================================================================

  otel-collector-dev:
    image: otel/opentelemetry-collector-contrib:${OTEL_COLLECTOR_VERSION}
    container_name: otel-collector-dev
    command: ["--config=/etc/otel-collector/config.yaml"]
    volumes:
      - ./dataplane/otel-collector/config-dev.yaml:/etc/otel-collector/config.yaml:ro
    ports:
      - "${OTEL_DEV_GRPC_PORT}:4317"
      - "${OTEL_DEV_HTTP_PORT}:4318"
      - "${OTEL_DEV_PROMETHEUS_PORT}:8889"
      - "${OTEL_DEV_METRICS_PORT}:8888"
    environment:
      - ENVIRONMENT=dev
      - DATA_PLANE_ID=dp-dev-01
    networks:
      - otel-network
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8888/"]
      interval: 10s
      timeout: 5s
      retries: 3
    restart: unless-stopped

  api-dev:
    image: otel-api-service:${API_VERSION}
    container_name: api-dev
    ports:
      - "${API_DEV_PORT}:8080"
    environment:
      - PORT=8080
      - ENVIRONMENT=dev
      - DATA_PLANE_ID=dp-dev-01
      - OTEL_ENDPOINT=http://otel-collector-dev:4318
      - SERVICE_NAME=api-service
      - SERVICE_VERSION=${API_VERSION}
      - ERROR_RATE=${ERROR_RATE_DEV}
      - ENABLE_SLOW_REQUESTS=true
    depends_on:
      otel-collector-dev:
        condition: service_healthy
    networks:
      - otel-network
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # ==========================================================================
  # DATA PLANE - QA
  # ==========================================================================

  otel-collector-qa:
    image: otel/opentelemetry-collector-contrib:${OTEL_COLLECTOR_VERSION}
    container_name: otel-collector-qa
    command: ["--config=/etc/otel-collector/config.yaml"]
    volumes:
      - ./dataplane/otel-collector/config-qa.yaml:/etc/otel-collector/config.yaml:ro
    ports:
      - "${OTEL_QA_GRPC_PORT}:4317"
      - "${OTEL_QA_HTTP_PORT}:4318"
      - "${OTEL_QA_PROMETHEUS_PORT}:8889"
      - "${OTEL_QA_METRICS_PORT}:8888"
    environment:
      - ENVIRONMENT=qa
      - DATA_PLANE_ID=dp-qa-01
    networks:
      - otel-network
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8888/"]
      interval: 10s
      timeout: 5s
      retries: 3
    restart: unless-stopped

  api-qa:
    image: otel-api-service:${API_VERSION}
    container_name: api-qa
    ports:
      - "${API_QA_PORT}:8080"
    environment:
      - PORT=8080
      - ENVIRONMENT=qa
      - DATA_PLANE_ID=dp-qa-01
      - OTEL_ENDPOINT=http://otel-collector-qa:4318
      - SERVICE_NAME=api-service
      - SERVICE_VERSION=${API_VERSION}
      - ERROR_RATE=${ERROR_RATE_QA}
      - ENABLE_SLOW_REQUESTS=true
    depends_on:
      otel-collector-qa:
        condition: service_healthy
    networks:
      - otel-network
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # ==========================================================================
  # CONTROL PLANE - PROMETHEUS
  # ==========================================================================

  prometheus:
    image: prom/prometheus:${PROMETHEUS_VERSION}
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=${PROMETHEUS_RETENTION}'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    ports:
      - "${PROMETHEUS_PORT}:9090"
    volumes:
      - ./controlplane/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./controlplane/prometheus/rules:/etc/prometheus/rules:ro
      - prometheus-data:/prometheus
    networks:
      - otel-network
    depends_on:
      - otel-collector-prod
      - otel-collector-dev
      - otel-collector-qa
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # ==========================================================================
  # CONTROL PLANE - DASHBOARD
  # ==========================================================================

  dashboard:
    build:
      context: ./controlplane/dashboard
      dockerfile: Dockerfile
    image: otel-dashboard:${API_VERSION}
    container_name: dashboard
    ports:
      - "${DASHBOARD_PORT}:80"
    networks:
      - otel-network
    depends_on:
      - prometheus
    environment:
      - PROMETHEUS_URL=http://prometheus:9090
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:80/"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

# ============================================================================
# NETWORKS
# ============================================================================

networks:
  otel-network:
    name: ${NETWORK_NAME}
    driver: bridge

# ============================================================================
# VOLUMES
# ============================================================================

volumes:
  prometheus-data:
    driver: local
```

---

## Step 4: Create Optional Thanos Compose File

### 4.1 Create `docker-compose.thanos.yml`
```yaml
version: '3.8'

# Optional: Add Thanos for long-term storage
# Usage: docker-compose -f docker-compose.yml -f docker-compose.thanos.yml up -d

services:
  # MinIO for S3-compatible storage
  minio:
    image: minio/minio:${MINIO_VERSION}
    container_name: minio
    command: server /data --console-address ":9001"
    ports:
      - "${MINIO_PORT}:9000"
      - "${MINIO_CONSOLE_PORT}:9001"
    environment:
      - MINIO_ROOT_USER=${MINIO_ROOT_USER}
      - MINIO_ROOT_PASSWORD=${MINIO_ROOT_PASSWORD}
    volumes:
      - minio-data:/data
    networks:
      - otel-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
    restart: unless-stopped

  # Create MinIO bucket
  minio-init:
    image: minio/mc:latest
    container_name: minio-init
    depends_on:
      - minio
    networks:
      - otel-network
    entrypoint: >
      /bin/sh -c "
      sleep 5;
      /usr/bin/mc alias set myminio http://minio:9000 ${MINIO_ROOT_USER} ${MINIO_ROOT_PASSWORD};
      /usr/bin/mc mb myminio/thanos --ignore-existing;
      /usr/bin/mc policy set download myminio/thanos;
      echo 'MinIO bucket initialized';
      "

  # Override Prometheus with Thanos-compatible settings
  prometheus:
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.max-block-duration=2h'
      - '--storage.tsdb.min-block-duration=2h'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'

  # Thanos Sidecar
  thanos-sidecar:
    image: quay.io/thanos/thanos:${THANOS_VERSION}
    container_name: thanos-sidecar
    command:
      - 'sidecar'
      - '--tsdb.path=/prometheus'
      - '--prometheus.url=http://prometheus:9090'
      - '--grpc-address=0.0.0.0:10901'
      - '--http-address=0.0.0.0:10902'
      - '--objstore.config-file=/etc/thanos/bucket-config.yaml'
    ports:
      - "10902:10902"
      - "10901:10901"
    volumes:
      - prometheus-data:/prometheus:ro
      - ./controlplane/thanos/bucket-config.yaml:/etc/thanos/bucket-config.yaml:ro
    networks:
      - otel-network
    depends_on:
      - prometheus
      - minio
    restart: unless-stopped

  # Thanos Query
  thanos-query:
    image: quay.io/thanos/thanos:${THANOS_VERSION}
    container_name: thanos-query
    command:
      - 'query'
      - '--http-address=0.0.0.0:10903'
      - '--grpc-address=0.0.0.0:10904'
      - '--store=thanos-sidecar:10901'
      - '--store=thanos-store:10905'
    ports:
      - "10903:10903"
    networks:
      - otel-network
    depends_on:
      - thanos-sidecar
    restart: unless-stopped

  # Thanos Store
  thanos-store:
    image: quay.io/thanos/thanos:${THANOS_VERSION}
    container_name: thanos-store
    command:
      - 'store'
      - '--data-dir=/var/thanos/store'
      - '--grpc-address=0.0.0.0:10905'
      - '--http-address=0.0.0.0:10906'
      - '--objstore.config-file=/etc/thanos/bucket-config.yaml'
    ports:
      - "10906:10906"
    volumes:
      - thanos-store-data:/var/thanos/store
      - ./controlplane/thanos/bucket-config.yaml:/etc/thanos/bucket-config.yaml:ro
    networks:
      - otel-network
    depends_on:
      - minio
    restart: unless-stopped

  # Thanos Compactor
  thanos-compactor:
    image: quay.io/thanos/thanos:${THANOS_VERSION}
    container_name: thanos-compactor
    command:
      - 'compact'
      - '--data-dir=/var/thanos/compactor'
      - '--http-address=0.0.0.0:10907'
      - '--objstore.config-file=/etc/thanos/bucket-config.yaml'
      - '--wait'
    ports:
      - "10907:10907"
    volumes:
      - thanos-compactor-data:/var/thanos/compactor
      - ./controlplane/thanos/bucket-config.yaml:/etc/thanos/bucket-config.yaml:ro
    networks:
      - otel-network
    depends_on:
      - minio
    restart: unless-stopped

volumes:
  minio-data:
  thanos-store-data:
  thanos-compactor-data:
```

---

## Step 5: Create Helper Scripts

### 5.1 Create `scripts/start.sh`
```bash
#!/bin/bash
# Start the OTEL PoC stack

set -e

echo "🚀 Starting OTEL Prometheus PoC..."

# Build images
echo "📦 Building Docker images..."
docker-compose build

# Start services
echo "🔧 Starting services..."
docker-compose up -d

# Wait for services to be healthy
echo "⏳ Waiting for services to become healthy..."
sleep 10

# Check service health
echo "✅ Checking service health..."
docker-compose ps

echo ""
echo "🎉 Stack is running!"
echo ""
echo "📊 Access points:"
echo "  - Dashboard:     http://localhost:3000"
echo "  - Prometheus:    http://localhost:9090"
echo "  - API (Prod):    http://localhost:8081"
echo "  - API (Dev):     http://localhost:8082"
echo "  - API (QA):      http://localhost:8083"
echo ""
echo "📝 View logs: docker-compose logs -f"
echo "🛑 Stop stack: docker-compose down"
```

### 5.2 Create `scripts/stop.sh`
```bash
#!/bin/bash
# Stop the OTEL PoC stack

set -e

echo "🛑 Stopping OTEL Prometheus PoC..."

docker-compose down

echo "✅ Stack stopped"
```

### 5.3 Create `scripts/clean.sh`
```bash
#!/bin/bash
# Clean up all data and images

set -e

echo "🧹 Cleaning up OTEL Prometheus PoC..."

# Stop and remove containers, networks, volumes
docker-compose down -v

# Remove images
echo "🗑️  Removing images..."
docker rmi otel-api-service:1.0.0 || true
docker rmi otel-dashboard:1.0.0 || true

echo "✅ Cleanup complete"
```

### 5.4 Create `scripts/logs.sh`
```bash
#!/bin/bash
# View logs for specific service or all services

SERVICE=${1:-}

if [ -z "$SERVICE" ]; then
  echo "📜 Showing logs for all services..."
  docker-compose logs -f
else
  echo "📜 Showing logs for $SERVICE..."
  docker-compose logs -f $SERVICE
fi
```

### 5.5 Make Scripts Executable
```bash
chmod +x scripts/*.sh
```

---

## Step 6: Testing the Complete Stack

### 6.1 Build and Start
```bash
# Option 1: Using script
./scripts/start.sh

# Option 2: Manual
docker-compose build
docker-compose up -d
```

### 6.2 Verify All Services Are Running
```bash
docker-compose ps

# Expected output:
# NAME                    STATUS
# api-prod                Up (healthy)
# api-dev                 Up (healthy)
# api-qa                  Up (healthy)
# otel-collector-prod     Up (healthy)
# otel-collector-dev      Up (healthy)
# otel-collector-qa       Up (healthy)
# prometheus              Up (healthy)
# dashboard               Up (healthy)
```

### 6.3 Test Individual Components

**Test API Services:**
```bash
curl http://localhost:8081/health  # Prod
curl http://localhost:8082/health  # Dev
curl http://localhost:8083/health  # QA
```

**Test OTEL Collectors:**
```bash
curl http://localhost:8889/metrics  # Prod collector
curl http://localhost:8890/metrics  # Dev collector
curl http://localhost:8892/metrics  # QA collector
```

**Test Prometheus:**
```bash
# Open in browser
open http://localhost:9090

# Check targets are UP
open http://localhost:9090/targets
```

**Test Dashboard:**
```bash
open http://localhost:3000
```

### 6.4 Generate Test Traffic
```bash
# Simple load test
for i in {1..100}; do
  curl http://localhost:8081/api/products &
  curl http://localhost:8082/api/orders -X POST -d '{"product_id":"1","quantity":2}' -H "Content-Type: application/json" &
  curl http://localhost:8083/api/search?q=test &
done

wait
```

---

## Step 7: Monitoring and Debugging

### 7.1 View Logs
```bash
# All logs
./scripts/logs.sh

# Specific service
./scripts/logs.sh api-prod
./scripts/logs.sh prometheus
./scripts/logs.sh dashboard
```

### 7.2 Inspect Service
```bash
# Check service details
docker-compose exec api-prod env

# Access shell
docker-compose exec api-prod sh
```

### 7.3 Resource Usage
```bash
docker stats
```

---

## Step 8: Scaling (Optional)

### 8.1 Add More Data Planes
Edit `docker-compose.yml` to add another environment (e.g., staging):

```yaml
  otel-collector-staging:
    # ... similar to other collectors
    environment:
      - ENVIRONMENT=staging
      - DATA_PLANE_ID=dp-staging-01

  api-staging:
    # ... similar to other APIs
    environment:
      - ENVIRONMENT=staging
```

### 8.2 Scale Existing Services
```bash
# Not applicable for this PoC since each data plane is unique
# But you could scale API replicas within an environment
```

---

## Step 9: Cleanup

### 9.1 Stop Services (Keep Data)
```bash
docker-compose stop
```

### 9.2 Stop and Remove (Keep Data)
```bash
docker-compose down
```

### 9.3 Complete Cleanup (Remove Data)
```bash
./scripts/clean.sh

# Or manually:
docker-compose down -v
docker system prune -a
```

---

## Expected Outcomes

✅ **Single Command Deployment**:
- `docker-compose up -d` starts entire stack
- All services start in correct order
- Health checks ensure readiness

✅ **Integrated Stack**:
- Data planes send metrics to collectors
- Collectors export to Prometheus
- Prometheus scrapes all collectors
- Dashboard queries Prometheus
- All components on same network

✅ **Production-Ready Configuration**:
- Environment variables externalized
- Persistent volumes for data
- Health checks configured
- Restart policies set
- Logging configured

---

## Troubleshooting

**Issue: Services fail to start**
```bash
# Check logs
docker-compose logs

# Check specific service
docker-compose logs api-prod

# Verify network
docker network inspect otel-network
```

**Issue: Can't reach services**
```bash
# Check ports are not already in use
netstat -an | grep 9090

# Verify container is running
docker ps | grep prometheus

# Check service health
docker-compose ps
```

**Issue: Out of memory**
```bash
# Check resource usage
docker stats

# Increase Docker memory limit in Docker Desktop settings
# Or adjust memory_limiter in OTEL Collector configs
```

---

## Next Steps

Proceed to **Phase 6: Load Generation** to create traffic generation scripts for realistic demo scenarios.
