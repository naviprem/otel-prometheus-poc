# Phase 2: Data Plane - OTEL Collector

## Overview
Configure OpenTelemetry Collector to receive metrics from the Go API, process them with environment labels, and export them in Prometheus format.

---

## Step 1: Understand OTEL Collector Architecture

The OTEL Collector has three main components:
- **Receivers**: Accept telemetry data (OTLP, Jaeger, Prometheus, etc.)
- **Processors**: Transform, filter, batch telemetry data
- **Exporters**: Send data to backends (Prometheus, Jaeger, Logging, etc.)

```
[Go API] --OTLP--> [Receiver] --> [Processor] --> [Exporter] --> [Prometheus]
```

---

## Step 2: Create Collector Configuration Directory

### 2.1 Create Directory Structure
```bash
mkdir -p dataplane/otel-collector
cd dataplane/otel-collector
```

---

## Step 3: Create OTEL Collector Configuration

### 3.1 Create `config.yaml`
```yaml
# OpenTelemetry Collector Configuration
# This configuration receives OTLP metrics and exports them in Prometheus format

receivers:
  # OTLP receiver accepts metrics via HTTP and gRPC
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
        cors:
          allowed_origins:
            - "*"

processors:
  # Batch processor groups metrics for efficient processing
  batch:
    timeout: 10s
    send_batch_size: 1024
    send_batch_max_size: 2048

  # Resource processor adds/modifies resource attributes
  # These attributes will be added as labels in Prometheus
  resource:
    attributes:
      - key: environment
        value: ${env:ENVIRONMENT}
        action: upsert
      - key: data_plane_id
        value: ${env:DATA_PLANE_ID}
        action: upsert
      - key: collector.version
        value: "0.91.0"
        action: insert

  # Memory limiter prevents OOM by refusing data when limits are reached
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128

  # Attributes processor can modify metric attributes
  attributes:
    actions:
      - key: deployment.type
        value: "docker"
        action: insert

exporters:
  # Prometheus exporter exposes metrics in Prometheus format
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: "otel"
    const_labels:
      environment: ${env:ENVIRONMENT}
      data_plane_id: ${env:DATA_PLANE_ID}
    send_timestamps: true
    metric_expiration: 5m
    enable_open_metrics: true
    resource_to_telemetry_conversion:
      enabled: true

  # Logging exporter for debugging (optional, can be removed in production)
  logging:
    loglevel: info
    sampling_initial: 5
    sampling_thereafter: 200

# Service section defines the telemetry pipeline
service:
  # Telemetry for the collector itself
  telemetry:
    logs:
      level: info
    metrics:
      address: 0.0.0.0:8888
      level: detailed

  # Define pipelines: receivers -> processors -> exporters
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, resource, attributes, batch]
      exporters: [prometheus, logging]

  # Extensions provide additional capabilities (health checks, etc.)
  extensions: []
```

### 3.2 Create Environment-Specific Configs (Optional)

For production data plane:
```bash
# config-prod.yaml
cp config.yaml config-prod.yaml
```

Edit `config-prod.yaml` to set defaults:
```yaml
resource:
  attributes:
    - key: environment
      value: ${env:ENVIRONMENT:prod}  # Default to 'prod'
      action: upsert
```

---

## Step 4: Create Docker Compose for Testing

### 4.1 Create `docker-compose.test.yml`
```yaml
version: '3.8'

services:
  # OTEL Collector
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.91.0
    container_name: otel-collector-test
    command: ["--config=/etc/otel-collector/config.yaml"]
    volumes:
      - ./config.yaml:/etc/otel-collector/config.yaml:ro
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8889:8889"   # Prometheus exporter
      - "8888:8888"   # Collector metrics
    environment:
      - ENVIRONMENT=dev
      - DATA_PLANE_ID=dp-dev-01
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8888/"]
      interval: 10s
      timeout: 5s
      retries: 3
    restart: unless-stopped

  # Test API (from Phase 1)
  api:
    image: otel-api-service:latest
    container_name: api-test
    ports:
      - "8080:8080"
    environment:
      - PORT=8080
      - ENVIRONMENT=dev
      - DATA_PLANE_ID=dp-dev-01
      - OTEL_ENDPOINT=otel-collector:4318
      - SERVICE_NAME=api-service
      - SERVICE_VERSION=1.0.0
      - ERROR_RATE=0.05
    depends_on:
      - otel-collector
    restart: unless-stopped

networks:
  default:
    name: otel-test-network
```

---

## Step 5: Testing the Collector

### 5.1 Start the Test Environment
```bash
cd dataplane/otel-collector
docker-compose -f docker-compose.test.yml up -d
```

### 5.2 Verify Collector is Running
```bash
# Check container logs
docker logs otel-collector-test

# Expected output should include:
# "Everything is ready. Begin running and processing data."
```

### 5.3 Check Collector Health
```bash
# Check collector's own metrics endpoint
curl http://localhost:8888/metrics

# This should return Prometheus-format metrics about the collector itself
```

### 5.4 Generate Test Traffic
```bash
# Generate some API requests
for i in {1..20}; do
  curl http://localhost:8080/health
  curl http://localhost:8080/api/products
  sleep 1
done
```

### 5.5 Verify Metrics Export
```bash
# Check Prometheus exporter endpoint
curl http://localhost:8889/metrics

# Look for metrics like:
# otel_http_server_duration_bucket{...}
# otel_http_server_duration_count{...}
# otel_http_server_duration_sum{...}
```

Expected output (sample):
```
# HELP otel_http_server_duration HTTP server request duration
# TYPE otel_http_server_duration histogram
otel_http_server_duration_bucket{data_plane_id="dp-dev-01",environment="dev",http_method="GET",http_route="/health",le="0.005"} 15
otel_http_server_duration_bucket{data_plane_id="dp-dev-01",environment="dev",http_method="GET",http_route="/health",le="0.01"} 20
...
```

### 5.6 Verify Labels
Check that environment labels are present:
```bash
curl http://localhost:8889/metrics | grep environment

# Should see lines containing:
# environment="dev"
# data_plane_id="dp-dev-01"
```

---

## Step 6: Advanced Configuration Options

### 6.1 Adding Metric Filtering
To filter out specific metrics (optional):

```yaml
processors:
  filter:
    metrics:
      exclude:
        match_type: regexp
        metric_names:
          - ^otel_.*internal.*$  # Exclude internal OTEL metrics
```

### 6.2 Adding Metric Transformation
To rename or modify metrics:

```yaml
processors:
  metricstransform:
    transforms:
      - include: ^http_server_duration$
        match_type: regexp
        action: update
        new_name: api_request_duration
```

### 6.3 Adding Resource Detection
To automatically detect cloud environment:

```yaml
processors:
  resourcedetection:
    detectors: [env, system, docker]
    timeout: 5s
    override: false
```

---

## Step 7: Create Configuration for Multiple Data Planes

### 7.1 Production Data Plane Config
Create `config-prod.yaml`:
```yaml
# Same as config.yaml but with environment=prod default
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
  resource:
    attributes:
      - key: environment
        value: ${env:ENVIRONMENT:prod}
        action: upsert
      - key: data_plane_id
        value: ${env:DATA_PLANE_ID:dp-prod-01}
        action: upsert
  memory_limiter:
    check_interval: 1s
    limit_mib: 1024

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: "otel"
    const_labels:
      environment: ${env:ENVIRONMENT:prod}
      data_plane_id: ${env:DATA_PLANE_ID:dp-prod-01}

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]
      exporters: [prometheus]
```

### 7.2 Dev Data Plane Config
Create `config-dev.yaml` with environment=dev defaults.

### 7.3 QA Data Plane Config
Create `config-qa.yaml` with environment=qa defaults.

---

## Step 8: Performance Tuning

### 8.1 Optimize Batch Processing
```yaml
processors:
  batch:
    timeout: 5s              # Send every 5 seconds
    send_batch_size: 512     # Or when 512 metrics accumulated
    send_batch_max_size: 1024
```

### 8.2 Optimize Memory Usage
```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512          # Adjust based on available memory
    spike_limit_mib: 128    # 25% of limit_mib
```

### 8.3 Optimize Prometheus Exporter
```yaml
exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
    metric_expiration: 5m   # Remove stale metrics after 5 min
    enable_open_metrics: true
```

---

## Step 9: Monitoring the Collector

### 9.1 Collector Self-Monitoring
The collector exposes its own metrics on port 8888:
```bash
curl http://localhost:8888/metrics | grep otelcol_

# Key metrics to watch:
# otelcol_receiver_accepted_metric_points - Metrics received
# otelcol_exporter_sent_metric_points - Metrics exported
# otelcol_processor_batch_batch_send_size - Batch sizes
```

### 9.2 Health Check Endpoint
```bash
# Check if collector is healthy
curl http://localhost:8888/

# Returns 200 if healthy
```

---

## Step 10: Cleanup and Documentation

### 10.1 Stop Test Environment
```bash
docker-compose -f docker-compose.test.yml down
```

### 10.2 Create README
Create `dataplane/otel-collector/README.md`:
```markdown
# OTEL Collector Configuration

## Overview
OpenTelemetry Collector configuration for receiving metrics from API services and exposing them in Prometheus format.

## Configuration Files
- `config.yaml` - Base configuration (environment-agnostic)
- `config-prod.yaml` - Production data plane
- `config-dev.yaml` - Development data plane
- `config-qa.yaml` - QA data plane

## Environment Variables
- `ENVIRONMENT` - Data plane environment (prod/dev/qa)
- `DATA_PLANE_ID` - Unique data plane identifier

## Ports
- 4317 - OTLP gRPC receiver
- 4318 - OTLP HTTP receiver
- 8889 - Prometheus exporter (scraped by Prometheus)
- 8888 - Collector metrics (self-monitoring)

## Testing
See docker-compose.test.yml for local testing setup.
```

---

## Expected Outcomes

✅ **Collector Running**:
- Receives OTLP metrics on ports 4317/4318
- Processes and labels metrics correctly
- Exposes Prometheus metrics on port 8889

✅ **Proper Labeling**:
- Environment label attached to all metrics
- Data plane ID label attached to all metrics
- Resource attributes converted to metric labels

✅ **Performance**:
- Low latency (<100ms processing time)
- Efficient batching
- Memory usage under limits

---

## Troubleshooting

**Issue: Collector not receiving metrics**
```
Check:
1. Verify API service is sending to correct endpoint
2. Check network connectivity (docker network ls)
3. Review collector logs: docker logs otel-collector-test
4. Verify receiver is enabled in pipeline
```

**Issue: Metrics missing labels**
```
Check:
1. Environment variables set correctly (docker exec otel-collector-test env)
2. Resource processor configured correctly
3. resource_to_telemetry_conversion enabled in exporter
```

**Issue: High memory usage**
```
Solution:
1. Reduce batch size
2. Lower memory_limiter limits
3. Increase metric_expiration in Prometheus exporter
```

**Issue: Prometheus exporter endpoint returns 404**
```
Check:
1. Exporter configured in service pipeline
2. Port 8889 exposed in docker-compose
3. Collector logs for exporter errors
```

---

## Validation Checklist

- [ ] Collector starts without errors
- [ ] Health check endpoint responds (port 8888)
- [ ] OTLP receivers accept metrics (ports 4317/4318)
- [ ] Prometheus exporter endpoint works (port 8889)
- [ ] Metrics include environment and data_plane_id labels
- [ ] Collector self-metrics available
- [ ] Multiple API requests generate visible metrics
- [ ] No memory leaks over extended runtime

---

## Next Steps

Proceed to **Phase 3: Control Plane - Prometheus & Thanos** to set up the control plane that will scrape these metrics from multiple data planes.
