# Phase 3: Control Plane - Prometheus & Thanos

## Overview
Set up Prometheus to scrape metrics from multiple OTEL Collector instances (data planes) and configure Thanos for long-term storage and high availability.

---

## Step 1: Create Control Plane Directory Structure

### 1.1 Create Directories
```bash
mkdir -p controlplane/prometheus
mkdir -p controlplane/thanos
mkdir -p controlplane/storage  # For local MinIO (S3-compatible)
cd controlplane
```

---

## Step 2: Configure Prometheus

### 2.1 Create `prometheus/prometheus.yml`
```yaml
# Prometheus Configuration for OTEL PoC
# Scrapes metrics from multiple data plane OTEL Collectors

global:
  scrape_interval: 10s      # How often to scrape targets
  evaluation_interval: 10s  # How often to evaluate rules
  external_labels:
    cluster: 'otel-poc'
    replica: 'prometheus-01'

# Alertmanager configuration (optional for PoC)
# alerting:
#   alertmanagers:
#     - static_configs:
#         - targets: ['alertmanager:9093']

# Rule files (optional for PoC)
# rule_files:
#   - /etc/prometheus/rules/*.yml

# Scrape configurations
scrape_configs:
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
        labels:
          component: 'prometheus'

  # Production Data Plane
  - job_name: 'dataplane-prod'
    scrape_interval: 10s
    scrape_timeout: 5s
    static_configs:
      - targets: ['otel-collector-prod:8889']
        labels:
          environment: 'prod'
          data_plane: 'production'
    metric_relabel_configs:
      # Preserve existing environment label from metrics
      - source_labels: [environment]
        target_label: __tmp_environment
      - target_label: environment
        replacement: 'prod'
        action: replace
        regex: '^$'
      - source_labels: [__tmp_environment]
        target_label: environment
        action: replace
        regex: '(.+)'

  # Development Data Plane
  - job_name: 'dataplane-dev'
    scrape_interval: 10s
    scrape_timeout: 5s
    static_configs:
      - targets: ['otel-collector-dev:8889']
        labels:
          environment: 'dev'
          data_plane: 'development'

  # QA Data Plane
  - job_name: 'dataplane-qa'
    scrape_interval: 10s
    scrape_timeout: 5s
    static_configs:
      - targets: ['otel-collector-qa:8889']
        labels:
          environment: 'qa'
          data_plane: 'quality-assurance'

  # OTEL Collectors self-monitoring
  - job_name: 'otel-collectors'
    scrape_interval: 15s
    static_configs:
      - targets:
          - 'otel-collector-prod:8888'
          - 'otel-collector-dev:8888'
          - 'otel-collector-qa:8888'
        labels:
          component: 'otel-collector'

# Storage configuration
storage:
  tsdb:
    path: /prometheus/data
    retention.time: 15d      # Keep data for 15 days
    retention.size: 10GB     # Or until 10GB is reached
```

### 2.2 Create Alert Rules (Optional)
Create `prometheus/rules/api-alerts.yml`:
```yaml
groups:
  - name: api_alerts
    interval: 30s
    rules:
      # High error rate alert
      - alert: HighErrorRate
        expr: |
          (
            rate(otel_http_server_duration_count{http_status_code=~"5.."}[5m])
            /
            rate(otel_http_server_duration_count[5m])
          ) > 0.05
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "{{ $labels.environment }} environment has {{ $value | humanizePercentage }} error rate"

      # High latency alert
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(otel_http_server_duration_bucket[5m])
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High API latency detected"
          description: "{{ $labels.environment }} p95 latency is {{ $value }}s"

      # Data plane down alert
      - alert: DataPlaneDown
        expr: up{job=~"dataplane-.*"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Data plane is down"
          description: "{{ $labels.job }} has been down for more than 1 minute"
```

---

## Step 3: Setup MinIO (S3-Compatible Storage)

### 3.1 Create `docker-compose.minio.yml`
```yaml
version: '3.8'

services:
  # MinIO - S3-compatible object storage for Thanos
  minio:
    image: minio/minio:latest
    container_name: minio
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"   # S3 API
      - "9001:9001"   # Web UI
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    volumes:
      - minio-data:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
    restart: unless-stopped

  # MinIO client to create buckets automatically
  minio-init:
    image: minio/mc:latest
    container_name: minio-init
    depends_on:
      - minio
    entrypoint: >
      /bin/sh -c "
      sleep 5;
      /usr/bin/mc alias set myminio http://minio:9000 minioadmin minioadmin;
      /usr/bin/mc mb myminio/thanos --ignore-existing;
      /usr/bin/mc policy set download myminio/thanos;
      echo 'MinIO bucket created successfully';
      "

volumes:
  minio-data:
    driver: local

networks:
  default:
    name: otel-network
```

### 3.2 Start MinIO
```bash
docker-compose -f docker-compose.minio.yml up -d
```

### 3.3 Verify MinIO
```bash
# Access MinIO web UI
open http://localhost:9001

# Login with:
# Username: minioadmin
# Password: minioadmin

# Verify bucket 'thanos' exists
```

---

## Step 4: Configure Thanos

### 4.1 Create S3 Configuration
Create `thanos/bucket-config.yaml`:
```yaml
type: S3
config:
  bucket: "thanos"
  endpoint: "minio:9000"
  access_key: "minioadmin"
  secret_key: "minioadmin"
  insecure: true
  signature_version2: false
  http_config:
    idle_conn_timeout: 90s
    response_header_timeout: 2m
    insecure_skip_verify: true
```

### 4.2 Create Thanos Sidecar Configuration
Create `docker-compose.thanos.yml`:
```yaml
version: '3.8'

services:
  # Prometheus with Thanos sidecar
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.max-block-duration=2h'
      - '--storage.tsdb.min-block-duration=2h'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus-data:/prometheus
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Thanos Sidecar - uploads Prometheus blocks to S3
  thanos-sidecar:
    image: quay.io/thanos/thanos:v0.32.5
    container_name: thanos-sidecar
    command:
      - 'sidecar'
      - '--tsdb.path=/prometheus'
      - '--prometheus.url=http://prometheus:9090'
      - '--grpc-address=0.0.0.0:10901'
      - '--http-address=0.0.0.0:10902'
      - '--objstore.config-file=/etc/thanos/bucket-config.yaml'
    ports:
      - "10902:10902"  # HTTP
      - "10901:10901"  # gRPC
    volumes:
      - prometheus-data:/prometheus:ro
      - ./thanos/bucket-config.yaml:/etc/thanos/bucket-config.yaml:ro
    depends_on:
      - prometheus
      - minio
    restart: unless-stopped

  # Thanos Query - provides unified query interface
  thanos-query:
    image: quay.io/thanos/thanos:v0.32.5
    container_name: thanos-query
    command:
      - 'query'
      - '--http-address=0.0.0.0:10903'
      - '--grpc-address=0.0.0.0:10904'
      - '--store=thanos-sidecar:10901'
      - '--store=thanos-store:10905'
    ports:
      - "10903:10903"  # Thanos Query UI
    depends_on:
      - thanos-sidecar
    restart: unless-stopped

  # Thanos Store - queries historical data from S3
  thanos-store:
    image: quay.io/thanos/thanos:v0.32.5
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
      - ./thanos/bucket-config.yaml:/etc/thanos/bucket-config.yaml:ro
    depends_on:
      - minio
    restart: unless-stopped

  # Thanos Compactor - downsamples and compacts data in S3
  thanos-compactor:
    image: quay.io/thanos/thanos:v0.32.5
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
      - ./thanos/bucket-config.yaml:/etc/thanos/bucket-config.yaml:ro
    depends_on:
      - minio
    restart: unless-stopped

volumes:
  prometheus-data:
  thanos-store-data:
  thanos-compactor-data:

networks:
  default:
    name: otel-network
    external: true
```

---

## Step 5: Simplified Setup (Without Thanos) - Recommended for Initial PoC

### 5.1 Create `docker-compose.prometheus.yml` (Simple Version)
```yaml
version: '3.8'

services:
  # Prometheus only (no Thanos for initial testing)
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus-data:/prometheus
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  prometheus-data:
    driver: local

networks:
  default:
    name: otel-network
    external: true
```

---

## Step 6: Testing Prometheus

### 6.1 Start Prometheus (Simple Version First)
```bash
# Create network if not exists
docker network create otel-network

# Start Prometheus
cd controlplane
docker-compose -f docker-compose.prometheus.yml up -d
```

### 6.2 Verify Prometheus is Running
```bash
# Check logs
docker logs prometheus

# Expected output:
# "Server is ready to receive web requests."
```

### 6.3 Access Prometheus UI
```bash
# Open in browser
open http://localhost:9090

# Or use curl
curl http://localhost:9090/-/healthy
```

### 6.4 Check Targets
Navigate to: http://localhost:9090/targets

You should see:
- prometheus (self-monitoring) - UP
- dataplane-prod - DOWN (not yet started)
- dataplane-dev - DOWN (not yet started)
- dataplane-qa - DOWN (not yet started)

### 6.5 Test PromQL Queries
In Prometheus UI, try these queries:

```promql
# Check what metrics are available
{__name__=~".+"}

# Check scrape success
up

# Check Prometheus self-metrics
prometheus_tsdb_head_samples_appended_total
```

---

## Step 7: Testing with Data Planes

### 7.1 Start One Data Plane for Testing
```bash
# In a separate terminal, start a data plane
cd dataplane/otel-collector
docker-compose -f docker-compose.test.yml up -d
```

### 7.2 Generate Traffic
```bash
# Generate some requests
for i in {1..50}; do
  curl http://localhost:8080/api/products
  sleep 0.5
done
```

### 7.3 Check Prometheus Targets
Refresh http://localhost:9090/targets

One target should now be UP (if you named it correctly in docker-compose)

### 7.4 Query Data Plane Metrics
In Prometheus UI:

```promql
# All OTEL metrics
{__name__=~"otel_.*"}

# Request rate per endpoint
rate(otel_http_server_duration_count[1m])

# P95 latency
histogram_quantile(0.95,
  rate(otel_http_server_duration_bucket[5m])
)

# Error rate
rate(otel_http_server_duration_count{http_status_code=~"5.."}[1m])
/
rate(otel_http_server_duration_count[1m])

# Filter by environment
rate(otel_http_server_duration_count{environment="dev"}[1m])
```

---

## Step 8: Adding Thanos (Optional - For Complete PoC)

### 8.1 Start MinIO
```bash
docker-compose -f docker-compose.minio.yml up -d
```

### 8.2 Verify MinIO Setup
```bash
# Check bucket exists
docker exec -it minio mc ls myminio/
# Should show: thanos/
```

### 8.3 Start Prometheus with Thanos
```bash
# Stop simple Prometheus first
docker-compose -f docker-compose.prometheus.yml down

# Start with Thanos
docker-compose -f docker-compose.thanos.yml up -d
```

### 8.4 Verify Thanos Components
```bash
# Check all containers
docker ps | grep thanos

# Should see:
# thanos-sidecar
# thanos-query
# thanos-store
# thanos-compactor
```

### 8.5 Access Thanos Query UI
```bash
open http://localhost:10903

# This provides same interface as Prometheus
# But can query across multiple Prometheus instances and S3 data
```

### 8.6 Verify Data Upload to S3
```bash
# After 2 hours (Prometheus block time), check MinIO
open http://localhost:9001

# Navigate to thanos bucket
# Should see uploaded TSDB blocks
```

---

## Step 9: Sample PromQL Queries for Dashboard

Document these queries for Phase 4 (Dashboard):

### 9.1 Create `prometheus/sample-queries.md`
```markdown
# Sample PromQL Queries for Dashboard

## Request Rate
```promql
# Total requests per second
sum(rate(otel_http_server_duration_count[5m]))

# Requests per second by environment
sum by (environment) (rate(otel_http_server_duration_count[5m]))

# Requests per second by endpoint
sum by (http_route) (rate(otel_http_server_duration_count[5m]))
```

## Latency Percentiles
```promql
# P50 latency (all environments)
histogram_quantile(0.50,
  sum(rate(otel_http_server_duration_bucket[5m])) by (le)
)

# P95 latency by environment
histogram_quantile(0.95,
  sum by (environment, le) (rate(otel_http_server_duration_bucket[5m]))
)

# P99 latency by environment
histogram_quantile(0.99,
  sum by (environment, le) (rate(otel_http_server_duration_bucket[5m]))
)
```

## Error Rate
```promql
# Error rate percentage
(
  sum(rate(otel_http_server_duration_count{http_status_code=~"5.."}[5m]))
  /
  sum(rate(otel_http_server_duration_count[5m]))
) * 100

# Error rate by environment
(
  sum by (environment) (rate(otel_http_server_duration_count{http_status_code=~"5.."}[5m]))
  /
  sum by (environment) (rate(otel_http_server_duration_count[5m]))
) * 100
```

## Active Requests
```promql
# Currently active requests
sum(otel_http_server_active_requests)

# Active requests by environment
sum by (environment) (otel_http_server_active_requests)
```

## Data Plane Health
```promql
# Up/down status
up{job=~"dataplane-.*"}

# Time since last successful scrape
time() - prometheus_tsdb_head_max_time / 1000
```
```

---

## Expected Outcomes

✅ **Prometheus Running**:
- Scraping metrics from data planes every 10 seconds
- Storing metrics with 15-day retention
- UI accessible at port 9090
- All configured targets visible

✅ **Thanos (Optional)**:
- Sidecar uploading blocks to MinIO/S3
- Query providing unified interface
- Store component querying historical data
- Compactor processing old data

✅ **Queries Working**:
- Can filter metrics by environment
- Can calculate latency percentiles
- Can compute error rates
- Can compare across data planes

---

## Troubleshooting

**Issue: Prometheus shows targets as DOWN**
```
Check:
1. Network connectivity: docker network inspect otel-network
2. OTEL Collector is running and exposing :8889
3. Correct service names in docker-compose
4. Check Prometheus logs: docker logs prometheus
```

**Issue: No metrics appearing**
```
Check:
1. Data planes are running and receiving traffic
2. OTEL Collectors are exporting metrics (:8889/metrics)
3. Prometheus scrape configs match container names
4. Check for scrape errors in Prometheus UI -> Targets
```

**Issue: Thanos can't upload to MinIO**
```
Check:
1. MinIO is running: docker ps | grep minio
2. Bucket exists: docker exec minio mc ls myminio/
3. Credentials correct in bucket-config.yaml
4. Check Thanos logs: docker logs thanos-sidecar
```

**Issue: Queries are slow**
```
Solution:
1. Reduce time range
2. Add more specific label filters
3. Increase Prometheus memory limit
4. Check TSDB size: du -sh /prometheus/data
```

---

## Next Steps

Proceed to **Phase 4: Control Plane - React Dashboard** to build the web interface that visualizes these metrics.
