# OTEL Prometheus PoC - Implementation Plan

## Overview
Build a proof-of-concept demonstrating COTS observability solution where customers deploy the entire stack (control plane + data planes) in their environment. Data planes represent different customer environments (dev, qa, prod) or lines of business.

## Project Structure
```
otel-prometheus-poc/
├── dataplane/
│   ├── api/                    # Go API application
│   │   ├── main.go
│   │   ├── handlers/
│   │   ├── middleware/
│   │   └── Dockerfile
│   └── otel-collector/
│       └── config.yaml         # OTEL Collector configuration
├── controlplane/
│   ├── prometheus/
│   │   └── prometheus.yml      # Prometheus configuration with static targets
│   ├── thanos/
│   │   └── thanos-sidecar.yml  # Thanos sidecar configuration
│   └── dashboard/
│       ├── src/                # React application
│       ├── package.json
│       └── Dockerfile
├── load-generator/
│   └── generate_load.py        # Traffic generation script
├── docker-compose.yml          # Main orchestration file
├── architecture.md
└── implementation-plan.md
```

---

## Phase 1: Data Plane - Go API Service

### 1.1 Setup Go Project
- [x] Initialize Go module
- [ ] Create project structure (handlers, middleware, models)
- [ ] Setup basic HTTP server (port 8080)

### 1.2 Implement API Endpoints
Create sample endpoints to generate interesting metrics:
- [ ] `GET /health` - Health check endpoint
- [ ] `GET /api/products` - List products (fast, low latency)
- [ ] `POST /api/orders` - Create order (medium latency)
- [ ] `GET /api/reports` - Generate report (slow, high latency)
- [ ] `GET /api/search` - Search functionality (variable latency)
- [ ] `POST /api/payments` - Process payment (can simulate errors)

### 1.3 OpenTelemetry Instrumentation
- [ ] Add OTEL SDK dependencies
- [ ] Configure OTEL exporter (OTLP HTTP to collector on port 4318)
- [ ] Instrument HTTP server with automatic metrics:
  - Request count
  - Request duration (histogram for percentiles)
  - Active requests (gauge)
  - Request size / Response size
- [ ] Add custom business metrics:
  - Orders created counter
  - Payment success/failure counter
  - Cache hit/miss ratio
- [ ] Add resource attributes (service.name, service.version)

### 1.4 Error Simulation
- [ ] Add configurable error rate (via environment variable)
- [ ] Implement random latency injection
- [ ] Add circuit breaker simulation for dependencies

### 1.5 Dockerfile
- [ ] Create multi-stage Dockerfile for Go API
- [ ] Optimize for small image size
- [ ] Configure health check

**Deliverable**: Instrumented Go API exposing OTEL metrics via OTLP

---

## Phase 2: Data Plane - OTEL Collector

### 2.1 OTEL Collector Configuration
- [ ] Configure receivers:
  - OTLP HTTP (port 4318)
  - OTLP gRPC (port 4317)
- [ ] Configure processors:
  - Batch processor (efficiency)
  - Resource processor (add environment labels)
  - Attributes processor (add data_plane_id, environment)
- [ ] Configure exporters:
  - Prometheus exporter (port 8889)
  - Logging exporter (debugging)

### 2.2 Resource Labeling Strategy
Add consistent labels for multi-environment setup:
```yaml
environment: "prod" | "dev" | "qa"
data_plane_id: "dp-prod-01"
service.name: "api-service"
```

### 2.3 Docker Setup
- [ ] Use official OTEL Collector image
- [ ] Mount configuration file
- [ ] Expose required ports (4317, 4318, 8889)

**Deliverable**: OTEL Collector accepting OTLP and exposing Prometheus metrics

---

## Phase 3: Control Plane - Prometheus & Thanos

### 3.1 Prometheus Configuration
- [ ] Create prometheus.yml with:
  - Global scrape interval (10s)
  - Static scrape configs for multiple data planes:
    - `job_name: dataplane-prod`
    - `job_name: dataplane-dev`
    - `job_name: dataplane-qa`
  - Relabeling rules to preserve environment labels
- [ ] Configure retention period (15d for PoC)
- [ ] Enable remote write for Thanos (optional for PoC)

### 3.2 Thanos Setup (Simplified for PoC)
Choose one approach:
- [ ] **Option A**: Thanos Sidecar + Local MinIO (recommended for demo)
  - Sidecar uploads blocks to S3-compatible storage
  - Query component for unified view
- [ ] **Option B**: Skip Thanos for initial PoC (add later)
  - Focus on core metrics pipeline first

### 3.3 Storage Backend
- [ ] Setup MinIO container (S3-compatible) for demo
- [ ] Configure bucket and credentials
- [ ] Test Thanos upload/query

### 3.4 Docker Configuration
- [ ] Prometheus container with volume mounts
- [ ] Thanos sidecar container
- [ ] MinIO container (if using Thanos)
- [ ] Expose Prometheus UI (port 9090)

**Deliverable**: Prometheus scraping multiple data planes with optional long-term storage

---

## Phase 4: Control Plane - React Dashboard

### 4.1 React Project Setup
- [ ] Initialize React app (Vite or Create React App)
- [ ] Install dependencies:
  - Apache ECharts (visualization)
  - Axios or Fetch (Prometheus API client)
  - React Router (navigation)
  - UI library (optional: Material-UI, Ant Design)

### 4.2 Prometheus Query Client
- [ ] Create Prometheus API client service
- [ ] Implement query functions:
  - Range queries (time series data)
  - Instant queries (current values)
- [ ] Add error handling and retries
- [ ] Configure Prometheus endpoint (http://prometheus:9090)

### 4.3 Dashboard Components
- [ ] **Landing Page**:
  - Overview of all data planes
  - Health status indicators
  - Key metrics summary
- [ ] **Metrics Dashboard**:
  - Environment selector dropdown
  - Time range picker (last 5m, 15m, 1h, 6h, 24h)
  - Charts using ECharts:
    - Request rate (requests/sec) - Line chart
    - Latency percentiles (p50, p95, p99) - Multi-line chart
    - Error rate (%) - Line chart with threshold
    - Active requests - Area chart
  - Multi-environment comparison view
- [ ] **Data Plane Status**:
  - List of configured data planes
  - Last scrape time
  - Metrics availability indicator

### 4.4 PromQL Queries
Define queries for key metrics:
```promql
# Request rate
rate(http_server_duration_count{environment="prod"}[5m])

# p95 latency
histogram_quantile(0.95, rate(http_server_duration_bucket[5m]))

# Error rate
rate(http_server_duration_count{http_status_code=~"5.."}[5m]) /
rate(http_server_duration_count[5m])
```

### 4.5 Real-time Updates
- [ ] Implement auto-refresh (configurable: 10s, 30s, 1m)
- [ ] Add manual refresh button
- [ ] Show last updated timestamp

### 4.6 Dockerfile & Build
- [ ] Create production Dockerfile (nginx serving static files)
- [ ] Optimize build for size
- [ ] Configure nginx to proxy Prometheus API (avoid CORS)

**Deliverable**: React dashboard displaying multi-environment metrics

---

## Phase 5: Docker Compose Orchestration

### 5.1 Network Configuration
- [ ] Define custom bridge network for all services
- [ ] Plan service DNS names for inter-service communication

### 5.2 Service Definitions
- [ ] **Data Plane - Prod**:
  - api-prod (Go API)
  - otel-collector-prod
- [ ] **Data Plane - Dev**:
  - api-dev (Go API)
  - otel-collector-dev
- [ ] **Data Plane - QA** (optional, add if needed):
  - api-qa (Go API)
  - otel-collector-qa
- [ ] **Control Plane**:
  - prometheus
  - thanos-sidecar (optional)
  - minio (optional, for Thanos)
  - dashboard (React app)

### 5.3 Environment Variables
Define environment-specific variables:
```yaml
api-prod:
  environment:
    - ENV=prod
    - DATA_PLANE_ID=dp-prod-01
    - OTEL_ENDPOINT=http://otel-collector-prod:4318
```

### 5.4 Volume Mounts
- [ ] Prometheus data volume (persistence)
- [ ] OTEL Collector configs (read-only)
- [ ] MinIO data volume (if using Thanos)

### 5.5 Port Mapping
Expose to host:
- [ ] Prometheus UI: 9090
- [ ] Dashboard: 3000 or 80
- [ ] Data Plane APIs: 8081 (prod), 8082 (dev), 8083 (qa)
- [ ] MinIO UI: 9001 (optional)

### 5.6 Healthchecks & Dependencies
- [ ] Add healthcheck configurations
- [ ] Define service dependencies (depends_on)
- [ ] Startup order: Collectors → Prometheus → Dashboard

**Deliverable**: Single `docker-compose up` command deploys entire stack

---

## Phase 6: Load Generation

### 6.1 Python Script Setup
- [ ] Create `generate_load.py` script
- [ ] Add dependencies: requests, argparse, concurrent.futures

### 6.2 Traffic Patterns
- [ ] Configure weighted endpoint distribution:
  - /health: 40% (fast)
  - /api/products: 30% (fast)
  - /api/orders: 15% (medium)
  - /api/reports: 10% (slow)
  - /api/payments: 5% (can fail)
- [ ] Implement concurrent request generation
- [ ] Add random latency between requests

### 6.3 Environment-Specific Patterns
- [ ] **Prod**: Steady baseline (50 req/sec)
- [ ] **Dev**: Sporadic bursts (5-20 req/sec)
- [ ] **QA**: Include error injection (10% failure rate)

### 6.4 Command-Line Interface
```bash
python generate_load.py \
  --target http://localhost:8081 \
  --environment prod \
  --rate 50 \
  --duration 300
```

### 6.5 Multi-Environment Mode
- [ ] Add option to load test multiple environments simultaneously
- [ ] Create wrapper script to launch parallel load generators

**Deliverable**: Realistic traffic generation for demo scenarios

---

## Phase 7: Testing & Validation

### 7.1 Component Testing
- [ ] Verify Go API endpoints return expected responses
- [ ] Verify OTEL metrics exported from Go API
- [ ] Verify OTEL Collector receives and transforms metrics
- [ ] Verify Prometheus scrapes all data planes
- [ ] Verify dashboard displays metrics correctly

### 7.2 Integration Testing
- [ ] End-to-end metric flow (API → Collector → Prometheus → Dashboard)
- [ ] Multi-environment filtering works
- [ ] Time range selection works
- [ ] Auto-refresh updates charts

### 7.3 Demo Scenarios
- [ ] **Scenario 1**: Normal operations across all environments
- [ ] **Scenario 2**: High latency in prod, normal in dev
- [ ] **Scenario 3**: Error spike in qa environment
- [ ] **Scenario 4**: Compare metrics across environments

### 7.4 Performance Validation
- [ ] Verify Prometheus query performance (<2s)
- [ ] Check resource usage (CPU, memory)
- [ ] Validate metric cardinality is reasonable

**Deliverable**: Validated, demo-ready PoC

---

## Phase 8: Documentation & Presentation

### 8.1 README.md
- [ ] Quick start guide (prerequisites, how to run)
- [ ] Architecture diagram
- [ ] Component descriptions
- [ ] Troubleshooting section

### 8.2 Demo Script
- [ ] Step-by-step demo walkthrough
- [ ] Key talking points
- [ ] Screenshot/video of expected output

### 8.3 Production Roadmap
- [ ] Differences between PoC and production (Docker → Helm)
- [ ] Additional features needed for production
- [ ] Security considerations
- [ ] Operational requirements

**Deliverable**: Complete documentation for handoff or presentation

---

## Success Criteria

✅ **Functional Requirements**:
- Multiple data planes reporting metrics independently
- Control plane aggregates and displays metrics from all data planes
- Dashboard allows filtering by environment
- Metrics update in near real-time (10-30s delay)

✅ **Demo Requirements**:
- Single command deployment (`docker-compose up -d`)
- Visible metrics within 60 seconds of startup
- Load generator creates realistic traffic patterns
- Can demonstrate incident scenario (high latency/errors)

✅ **Technical Requirements**:
- All components use industry standards (OTEL, Prometheus, PromQL)
- Proper metric labeling (environment, data_plane_id)
- No licensing issues (no Grafana redistribution)
- Clean, maintainable code

---

## Implementation Order (Recommended)

1. **Week 1**: Phases 1-2 (Data Plane API + OTEL Collector)
2. **Week 2**: Phase 3 (Prometheus without Thanos)
3. **Week 3**: Phase 4 (React Dashboard - basic version)
4. **Week 4**: Phases 5-6 (Docker Compose + Load Generator)
5. **Week 5**: Phase 7-8 (Testing, Thanos integration, Documentation)

---

## Optional Enhancements (Post-PoC)

- [ ] Alerting rules in Prometheus
- [ ] Alert visualization in dashboard
- [ ] Trace collection (OTEL supports distributed tracing)
- [ ] Log aggregation (OTEL Collector can handle logs)
- [ ] Dashboard template system (customers can customize)
- [ ] Authentication for dashboard
- [ ] API for dashboard (backend service layer)
- [ ] Export metrics to CSV/PDF

---

## Notes

- Start simple: Get basic pipeline working before adding Thanos
- Focus on demo value: Visual, easy to understand metrics
- Keep configurations parameterized: Easy to add more data planes
- Document as you go: Easier than retrofitting documentation
