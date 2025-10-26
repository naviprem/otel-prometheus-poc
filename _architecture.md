# Architecture & Project Structure

## System Architecture

```
┌─────────────┐         ┌──────────────┐         ┌──────────────┐         ┌────────────┐
│ Load Script │────────>│   Go API     │────────>│    OTEL      │────────>│ Prometheus │
│  (Traffic)  │         │(OTLP Export) │  OTLP   │  Collector   │ scrape  │  (TSDB)    │
└─────────────┘         └──────────────┘  :4318  └──────────────┘  :8889  └──────┬─────┘
                                                        │                          │
                                                   Receivers                       │
                                                   Processors                      ▼
                                                   Exporters              ┌─────────────────┐
                                                                          │ Docker Volume   │
                                                                          │ prometheus-data │
                                                                          │ (/prometheus)   │
                                                                          └─────────────────┘
```

## Components

### 1. Go API
- HTTP server exposing multiple endpoints
- Instrumented with OpenTelemetry SDK
- Exports metrics via OTLP (OpenTelemetry Protocol)
- Sends metrics to OTEL Collector on port 4318 (HTTP) or 4317 (gRPC)
- Port: 8080

### 2. OTEL Collector
- Vendor-neutral telemetry data pipeline
- **Receivers**: Accepts OTLP metrics from Go API
- **Processors**: Batches and transforms metrics
- **Exporters**: Exposes metrics in Prometheus format
- Decouples application from monitoring backend
- Ports: 4317 (OTLP gRPC), 4318 (OTLP HTTP), 8889 (Prometheus exporter)

### 3. Prometheus
- Scrapes metrics from OTEL Collector every 10 seconds
- Stores time-series data
- Provides PromQL query interface
- Web UI at http://localhost:9090

### 4. Load Generation Script
- Bash/Python script generating realistic traffic
- Weighted distribution across endpoints
- Configurable request rate and duration
- Simulates various usage patterns

### 5. Persistent Storage
- **Docker Volume**: `prometheus-data` for Prometheus TSDB
- **Location**: `/prometheus` inside container
- **Host Location**: `/var/lib/docker/volumes/prometheus-poc_prometheus-data/_data`
- **Retention**: 15 days (default, configurable)
- **Format**: Custom time-series database (TSDB) with blocks

---

## Storage & Persistence

### Prometheus Time-Series Database (TSDB)

Prometheus stores all scraped metrics in a **local, custom-built time-series database** on disk.

#### Storage Architecture

```
/prometheus/
├── chunks_head/              # Current, in-memory data being written
├── wal/                      # Write-Ahead Log for crash recovery
├── 01HXXX.../               # Time-based blocks (2h chunks)
│   ├── index                # Inverted index for queries
│   ├── chunks/              # Compressed metric data
│   │   └── 000001
│   ├── tombstones           # Deleted series markers
│   └── meta.json            # Block metadata
├── 01HYYY.../               # Another time block
└── lock                     # Lock file
```

#### Data Organization

1. **Blocks**: Data organized into 2-hour time windows
2. **Compression**: Extremely efficient (1.3-3 bytes per sample)
   - XOR delta encoding for values
   - Variable-bit encoding for timestamps
3. **Compaction**: Older blocks merged into larger blocks
4. **Write-Ahead Log**: Ensures no data loss on crash

#### Storage Lifecycle

```
Scraped Metric
    ↓
Write-Ahead Log (WAL)
    ↓
In-Memory Chunks (chunks_head)
    ↓
Persisted Blocks (every 2 hours)
    ↓
Compacted Blocks (background process)
    ↓
Deleted After Retention Period
```

### Docker Volume Persistence

Our setup uses a **named volume** for persistence:

```yaml
volumes:
  prometheus-data:           # Named volume
    driver: local

services:
  prometheus:
    volumes:
      - prometheus-data:/prometheus
```

#### Persistence Behavior

| Command | Data Persists? |
|---------|---------------|
| `docker-compose restart` | ✅ Yes |
| `docker-compose stop` + `start` | ✅ Yes |
| `docker-compose down` | ✅ Yes |
| `docker-compose down -v` | ❌ No (deletes volumes) |

#### Storage Configuration

```yaml
prometheus:
  command:
    - '--storage.tsdb.path=/prometheus'
    - '--storage.tsdb.retention.time=15d'     # Keep 15 days
    - '--storage.tsdb.retention.size=10GB'    # Or max 10GB
    - '--storage.tsdb.wal-compression'        # Compress WAL
```

### Viewing Storage Stats

#### From Prometheus UI
Navigate to **Status → TSDB Status** to see:
- Number of series
- Number of chunks
- Data size on disk
- Head series count
- WAL size

#### From Command Line
```bash
# Check volume size
docker volume inspect prometheus-poc_prometheus-data

# Check storage usage inside container
docker exec prometheus du -sh /prometheus

# List blocks
docker exec prometheus ls -lh /prometheus
```

### Storage Capacity Planning

**Typical storage requirements**:
- **Low traffic** (100 series, 1 sample/s): ~2-3 MB/day
- **Medium traffic** (1,000 series, 1 sample/s): ~20-30 MB/day
- **High traffic** (10,000 series, 1 sample/s): ~200-300 MB/day

**Our demo** (during active load testing):
- ~50-100 series
- 10-second scrape interval
- ~1-2 MB per hour
- ~15-30 MB for 15 days retention

### Backup and Recovery

#### Backup Volume
```bash
# Create backup
docker run --rm \
  -v prometheus-poc_prometheus-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/prometheus-backup.tar.gz /data

# Restore backup
docker run --rm \
  -v prometheus-poc_prometheus-data:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/prometheus-backup.tar.gz -C /
```

#### Prometheus Snapshots (Alternative)
```bash
# Create snapshot via API
curl -X POST http://localhost:9090/api/v1/admin/tsdb/snapshot

# Copy snapshot from container
docker cp prometheus:/prometheus/snapshots/20251025... ./backup/
```

### Long-Term Storage (Optional)

For production deployments requiring longer retention:

#### Remote Write Targets
- **Thanos**: S3/GCS-backed, horizontally scalable
- **Cortex/Mimir**: Multi-tenant, horizontally scalable
- **VictoriaMetrics**: Fast, cost-effective
- **Grafana Cloud**: Managed service

#### Configuration Example
```yaml
# In prometheus.yml
remote_write:
  - url: "http://thanos:19291/api/v1/receive"
    queue_config:
      capacity: 10000
      max_samples_per_send: 5000
```

---

## Project Structure

```
prometheus-poc/
├── cmd/
│   └── api/
│       └── main.go                 # Application entry point
│
├── internal/
│   ├── handlers/
│   │   ├── health.go              # Health check handler
│   │   ├── users.go               # Users API handler
│   │   ├── orders.go              # Orders API handler
│   │   └── slow.go                # Slow endpoint handler
│   ├── metrics/
│   │   └── otel.go                # OTEL metrics configuration
│   ├── middleware/
│   │   └── instrumentation.go    # HTTP instrumentation middleware
│   └── models/
│       └── types.go               # Request/Response types
│
├── scripts/
│   ├── load-test.sh               # Bash-based load generator
│   └── load-test.py               # Python-based load generator (optional)
│
├── config/
│   ├── otel-collector-config.yaml # OTEL Collector configuration
│   └── prometheus.yml             # Prometheus scrape configuration
│
├── docs/
│   ├── implementation-plan.md     # High-level overview and plan
│   ├── architecture.md            # This file
│   ├── step-by-step-guide.md      # Detailed implementation steps
│   ├── metrics-guide.md           # Prometheus queries reference
│   ├── demo-walkthrough.md        # Demo instructions
│   ├── testing-checklist.md       # Testing and success criteria
│   └── resources.md               # Learning resources
│
├── docker-compose.yml             # Container orchestration
├── Dockerfile                     # Go API container
├── .dockerignore                  # Docker ignore patterns
├── go.mod                         # Go module definition
├── go.sum                         # Go dependencies checksum
└── README.md                      # Project documentation
```

---

## Directory Breakdown

### `/cmd/api`
Application entry point following Go project layout conventions. Contains the main package and server initialization.

**Files**:
- `main.go` - HTTP server setup, routing, graceful shutdown

### `/internal`
Private application code that cannot be imported by other projects.

**Subdirectories**:
- `handlers/` - HTTP request handlers for each endpoint
- `metrics/` - OTEL configuration and metrics initialization
- `middleware/` - HTTP middleware for instrumentation and logging
- `models/` - Data structures for requests and responses

### `/scripts`
Utility scripts for load testing and automation.

**Files**:
- `load-test.sh` - Bash script for traffic generation
- `load-test.py` - Python alternative with advanced features

### `/config`
Configuration files for external services.

**Files**:
- `prometheus.yml` - Prometheus scrape configuration

### `/docs`
Project documentation organized by topic.

---

## API Endpoints

### `GET /health`
**Purpose**: Health check endpoint
- Response Time: < 10ms
- Success Rate: 100%
- Use Case: Baseline for fast, reliable requests

### `GET /api/users`
**Purpose**: Simulate database queries
- Response Time: 50-200ms (variable)
- Success Rate: 95%
- Use Case: Demonstrate variable latency and occasional errors

### `POST /api/orders`
**Purpose**: Create operation with validation
- Response Time: 100-300ms
- Success Rate: 90%
- Use Case: Show request validation, business logic, error handling

### `GET /api/slow`
**Purpose**: Intentionally slow endpoint
- Response Time: 3-5 seconds
- Success Rate: 100%
- Use Case: Demonstrate high latency in metrics

---

## Metrics Flow

```
HTTP Request
    │
    ├─> Middleware: Increment in-flight gauge
    │
    ├─> Middleware: Record start time
    │
    ├─> Handler: Process request
    │   │
    │   └─> Handler: Record business metrics (orders, cache hits, etc.)
    │
    ├─> Middleware: Calculate duration
    │
    ├─> Middleware: Record histogram (duration)
    │
    ├─> Middleware: Increment counter (total requests)
    │
    ├─> Middleware: Decrement in-flight gauge
    │
    └─> OTEL SDK: Batch metrics
        │
        └─> Export via OTLP (HTTP/gRPC)
            │
            └─> OTEL Collector
                │
                ├─> Receiver: Ingest OTLP data
                │
                ├─> Processor: Batch & transform
                │
                └─> Exporter: Prometheus format
                    │
                    └─> Prometheus: Scrape & store
```

---

## Metrics Exposed

### HTTP Metrics (Auto-instrumented)
| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `http_requests_total` | Counter | method, endpoint, status | Total HTTP requests |
| `http_request_duration_seconds` | Histogram | method, endpoint | Request latency distribution |
| `http_requests_in_flight` | Gauge | method | Active concurrent requests |

### Business Metrics (Custom)
| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `orders_total` | Counter | status | Total orders created |
| `orders_value_total` | Counter | - | Sum of all order values |
| `cache_hits_total` | Counter | result (hit/miss) | Cache hit/miss count |

---

## Data Flow

1. **Traffic Generation**: Load script sends HTTP requests to API
2. **Request Processing**: Middleware captures timing and request metadata
3. **Metrics Recording**: OTEL SDK records counters, histograms, gauges in-memory
4. **Metrics Push**: OTEL SDK exports metrics via OTLP to Collector (push model)
5. **Collector Processing**: OTEL Collector receives, batches, and transforms metrics
6. **Metrics Exposure**: Collector exposes metrics in Prometheus format on port 8889
7. **Metrics Scraping**: Prometheus scrapes Collector every 10 seconds (pull model)
8. **Data Storage**: Prometheus stores time-series data
9. **Querying**: PromQL queries analyze stored metrics

---

## Deployment Architecture

### Docker Compose
```yaml
Networks:
  - monitoring (bridge network)

Services:
  - api (custom built from Dockerfile)
  - otel-collector (official otel/opentelemetry-collector image)
  - prometheus (official prom/prometheus image)

Volumes:
  - prometheus-data (persistent storage)
```

### Container Communication
- All services on same Docker network (`monitoring`)
- **API → Collector**: API pushes metrics to `otel-collector:4318` (OTLP HTTP)
- **Collector → Prometheus**: Prometheus scrapes `otel-collector:8889`
- **Host Access**:
  - API: `localhost:8080`
  - OTEL Collector: `localhost:4317` (gRPC), `localhost:4318` (HTTP), `localhost:8889` (Prometheus)
  - Prometheus: `localhost:9090`

---

## Technology Stack

### Backend
- **Language**: Go 1.21+
- **HTTP Router**: gorilla/mux
- **Metrics SDK**: OpenTelemetry Go SDK
- **Exporter**: OTLP Exporter (HTTP/gRPC)

### Telemetry Pipeline
- **Collector**: OpenTelemetry Collector
- **Protocol**: OTLP (OpenTelemetry Protocol)
- **Processors**: Batch processor
- **Exporters**: Prometheus exporter

### Monitoring
- **Metrics Storage**: Prometheus
- **Query Language**: PromQL
- **Scrape Protocol**: Prometheus exposition format

### Infrastructure
- **Containerization**: Docker
- **Orchestration**: Docker Compose
- **OS**: Alpine Linux (container base)

---

## Design Decisions

### Why OpenTelemetry?
- Vendor-neutral instrumentation
- Future extensibility (tracing, logging)
- Industry standard
- Active community and support

### Why OTEL Collector?
- **Decoupling**: Separates application code from monitoring backends
- **Flexibility**: Switch backends without changing application code
- **Real-world pattern**: Mirrors production deployments
- **Processing power**: Centralized filtering, batching, and transformation
- **Multi-backend**: Can export to multiple destinations simultaneously
- **Educational**: Demonstrates complete OTEL telemetry pipeline

### Why Multiple Endpoints?
- Demonstrate different latency profiles
- Show error handling in metrics
- Realistic traffic distribution
- Educational value

### Why Docker Compose?
- Easy local development
- Reproducible environment
- Simple setup for demos
- Minimal infrastructure requirements
