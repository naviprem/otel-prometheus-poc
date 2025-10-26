# Step-by-Step Implementation Guide

This guide provides detailed instructions for implementing the OTEL metrics demonstration project.

---

## Step 1: Initialize Go Project

**Objective**: Set up the Go module and project structure

**Tasks**:

1. Initialize Go module
   ```bash
   go mod init github.com/yourusername/prometheus-poc
   ```

2. Create directory structure
   ```bash
   mkdir -p cmd/api internal/{handlers,metrics,middleware,models} scripts config docs
   ```

3. Install core dependencies
   ```bash
   go get github.com/gorilla/mux
   ```

**Deliverables**:
- `go.mod` file
- Directory structure created

**Verification**:
```bash
tree -L 3 prometheus-poc/
```

---

## Step 2: Build Basic API Endpoints

**Objective**: Create HTTP server with four demonstration endpoints

### 2.1 Health Endpoint (`GET /health`)
- **Purpose**: Baseline endpoint, always fast and successful
- **Response Time**: < 10ms
- **Success Rate**: 100%
- **Response**: `{"status": "ok", "timestamp": "2025-10-25T..."}`

**Implementation**:
```go
// internal/handlers/health.go
package handlers

import (
    "encoding/json"
    "net/http"
    "time"
)

type HealthResponse struct {
    Status    string    `json:"status"`
    Timestamp time.Time `json:"timestamp"`
}

func HealthHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(HealthResponse{
        Status:    "ok",
        Timestamp: time.Now(),
    })
}
```

### 2.2 Users Endpoint (`GET /api/users`)
- **Purpose**: Simulate database query with variable latency
- **Response Time**: 50-200ms (randomized)
- **Success Rate**: 95% (5% random failures)
- **Response**: List of user objects
- **Behavior**:
  - Simulates DB latency with `time.Sleep()`
  - Occasionally returns 500 errors

**Implementation**:
```go
// internal/handlers/users.go
package handlers

import (
    "encoding/json"
    "math/rand"
    "net/http"
    "time"
)

type User struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func UsersHandler(w http.ResponseWriter, r *http.Request) {
    // Simulate database latency (50-200ms)
    latency := time.Duration(50+rand.Intn(150)) * time.Millisecond
    time.Sleep(latency)

    // Simulate 5% error rate
    if rand.Float32() < 0.05 {
        http.Error(w, "Internal server error", http.StatusInternalServerError)
        return
    }

    users := []User{
        {ID: "1", Name: "Alice", Email: "alice@example.com"},
        {ID: "2", Name: "Bob", Email: "bob@example.com"},
        {ID: "3", Name: "Charlie", Email: "charlie@example.com"},
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}
```

### 2.3 Orders Endpoint (`POST /api/orders`)
- **Purpose**: Create operation with validation and processing
- **Response Time**: 100-300ms
- **Success Rate**: 90% (10% validation/processing failures)
- **Request Body**: `{"user_id": "...", "items": [...], "total": 99.99}`
- **Response**: Created order object
- **Behavior**:
  - Validates request body
  - Returns 400 for invalid data
  - Returns 500 for processing errors
  - Returns 201 on success

**Implementation**:
```go
// internal/handlers/orders.go
package handlers

import (
    "encoding/json"
    "math/rand"
    "net/http"
    "time"
)

type OrderRequest struct {
    UserID string        `json:"user_id"`
    Items  []OrderItem   `json:"items"`
    Total  float64       `json:"total"`
}

type OrderItem struct {
    ID       string `json:"id"`
    Quantity int    `json:"qty"`
}

type OrderResponse struct {
    OrderID   string    `json:"order_id"`
    Status    string    `json:"status"`
    CreatedAt time.Time `json:"created_at"`
}

func OrdersHandler(w http.ResponseWriter, r *http.Request) {
    var req OrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    // Validate request
    if req.UserID == "" || len(req.Items) == 0 || req.Total <= 0 {
        http.Error(w, "Invalid order data", http.StatusBadRequest)
        return
    }

    // Simulate processing time (100-300ms)
    latency := time.Duration(100+rand.Intn(200)) * time.Millisecond
    time.Sleep(latency)

    // Simulate 10% processing error rate
    if rand.Float32() < 0.10 {
        http.Error(w, "Processing error", http.StatusInternalServerError)
        return
    }

    // Success response
    response := OrderResponse{
        OrderID:   generateOrderID(),
        Status:    "created",
        CreatedAt: time.Now(),
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(response)
}

func generateOrderID() string {
    return fmt.Sprintf("ORD-%d", rand.Intn(1000000))
}
```

### 2.4 Slow Endpoint (`GET /api/slow`)
- **Purpose**: Demonstrate high latency metrics
- **Response Time**: 3-5 seconds
- **Success Rate**: 100%
- **Response**: `{"message": "This was intentionally slow"}`

**Implementation**:
```go
// internal/handlers/slow.go
package handlers

import (
    "encoding/json"
    "math/rand"
    "net/http"
    "time"
)

type SlowResponse struct {
    Message string `json:"message"`
}

func SlowHandler(w http.ResponseWriter, r *http.Request) {
    // Intentional delay: 3-5 seconds
    delay := time.Duration(3+rand.Intn(2)) * time.Second
    time.Sleep(delay)

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(SlowResponse{
        Message: "This was intentionally slow",
    })
}
```

### Main Server Setup

**Implementation**:
```go
// cmd/api/main.go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "time"

    "github.com/gorilla/mux"
    "github.com/yourusername/prometheus-poc/internal/handlers"
)

func main() {
    router := mux.NewRouter()

    // Register routes
    router.HandleFunc("/health", handlers.HealthHandler).Methods("GET")
    router.HandleFunc("/api/users", handlers.UsersHandler).Methods("GET")
    router.HandleFunc("/api/orders", handlers.OrdersHandler).Methods("POST")
    router.HandleFunc("/api/slow", handlers.SlowHandler).Methods("GET")

    // Create server
    srv := &http.Server{
        Addr:         ":8080",
        Handler:      router,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Start server in goroutine
    go func() {
        log.Println("Server starting on :8080")
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server failed: %v", err)
        }
    }()

    // Graceful shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt)
    <-quit

    log.Println("Shutting down server...")
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("Server shutdown failed: %v", err)
    }
    log.Println("Server stopped")
}
```

**Deliverables**:
- Working API server on port 8080
- All endpoints responding correctly
- Basic error handling and logging

**Verification**:
```bash
# Start the server
go run cmd/api/main.go

# Test endpoints
curl http://localhost:8080/health
curl http://localhost:8080/api/users
curl -X POST http://localhost:8080/api/orders -H "Content-Type: application/json" -d '{"user_id":"123","items":[{"id":"item1","qty":2}],"total":99.99}'
curl http://localhost:8080/api/slow
```

---

## Step 3: Add OTEL Dependencies

**Objective**: Install OpenTelemetry SDK and OTLP exporter

**Dependencies**:
```bash
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/otel/sdk/metric
go get go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp
go get go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc  # Alternative: gRPC
go get go.opentelemetry.io/otel/sdk/resource
go get go.opentelemetry.io/otel/semconv/v1.24.0
```

**Note**: We're using the OTLP exporter instead of Prometheus exporter to push metrics to the OTEL Collector. You can use either HTTP (`otlpmetrichttp`) or gRPC (`otlpmetricgrpc`) transport.

**Deliverables**:
- Updated `go.mod` with OTEL dependencies

**Verification**:
```bash
go mod tidy
cat go.mod | grep opentelemetry
```

---

## Step 4: Configure OTEL Metrics

**Objective**: Set up OTEL metrics provider and OTLP exporter

**Implementation** (`internal/metrics/otel.go`):

```go
package metrics

import (
    "context"
    "log"
    "time"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp"
    "go.opentelemetry.io/otel/metric"
    sdkmetric "go.opentelemetry.io/otel/sdk/metric"
    "go.opentelemetry.io/otel/sdk/resource"
    semconv "go.opentelemetry.io/otel/semconv/v1.24.0"
)

var (
    Meter metric.Meter
    meterProvider *sdkmetric.MeterProvider
)

// InitMetrics initializes the OTEL metrics provider with OTLP exporter
func InitMetrics(ctx context.Context) error {
    // Create resource with service information
    res, err := resource.New(
        ctx,
        resource.WithAttributes(
            semconv.ServiceName("go-api"),
            semconv.ServiceVersion("1.0.0"),
        ),
    )
    if err != nil {
        return err
    }

    // Create OTLP HTTP exporter
    // By default, connects to http://localhost:4318
    // Override with OTEL_EXPORTER_OTLP_ENDPOINT environment variable
    exporter, err := otlpmetrichttp.New(
        ctx,
        otlpmetrichttp.WithEndpoint("otel-collector:4318"),
        otlpmetrichttp.WithInsecure(), // Use HTTP instead of HTTPS for local dev
    )
    if err != nil {
        return err
    }

    // Create meter provider with periodic reader
    meterProvider = sdkmetric.NewMeterProvider(
        sdkmetric.WithResource(res),
        sdkmetric.WithReader(
            sdkmetric.NewPeriodicReader(
                exporter,
                sdkmetric.WithInterval(10*time.Second), // Export every 10 seconds
            ),
        ),
    )

    // Set global meter provider
    otel.SetMeterProvider(meterProvider)

    // Get meter for this service
    Meter = meterProvider.Meter("go-api")

    log.Println("OTEL metrics initialized with OTLP exporter")
    return nil
}

// Shutdown gracefully shuts down the meter provider
func Shutdown(ctx context.Context) error {
    if meterProvider != nil {
        return meterProvider.Shutdown(ctx)
    }
    return nil
}
```

**Alternative: Using gRPC Instead of HTTP**:
```go
import (
    "go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc"
)

// In InitMetrics:
exporter, err := otlpmetricgrpc.New(
    ctx,
    otlpmetricgrpc.WithEndpoint("otel-collector:4317"),
    otlpmetricgrpc.WithInsecure(),
)
```

**Key Metrics to Initialize**:

| Metric Name | Type | Description | Labels |
|-------------|------|-------------|--------|
| `http_requests_total` | Counter | Total HTTP requests | `method`, `endpoint`, `status` |
| `http_request_duration_seconds` | Histogram | Request latency distribution | `method`, `endpoint` |
| `http_requests_in_flight` | Gauge | Active concurrent requests | `method` |
| `orders_total` | Counter | Total orders created | `status` |
| `orders_value_total` | Counter | Sum of order values | - |

**Update main.go**:
```go
// cmd/api/main.go
import (
    "context"
    "github.com/yourusername/prometheus-poc/internal/metrics"
)

func main() {
    ctx := context.Background()

    // Initialize metrics
    if err := metrics.InitMetrics(ctx); err != nil {
        log.Fatalf("Failed to initialize metrics: %v", err)
    }

    // Ensure graceful shutdown
    defer func() {
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()
        if err := metrics.Shutdown(shutdownCtx); err != nil {
            log.Printf("Error shutting down metrics: %v", err)
        }
    }()

    router := mux.NewRouter()

    // No need to expose /metrics endpoint - OTLP pushes to collector
    // Register your API routes here...

    // ... rest of the setup
}
```

**Deliverables**:
- `internal/metrics/otel.go` with OTLP exporter setup
- Metrics pushed to OTEL Collector (no /metrics endpoint needed)
- Proper shutdown handling

**Verification**:
```bash
# The API no longer exposes /metrics endpoint
# Metrics are pushed to the OTEL Collector
# Verify OTEL Collector is receiving metrics (covered in Step 7)
```

**Verification**:
```bash
curl http://localhost:8080/metrics
```

---

## Step 5: Implement Instrumentation Middleware

**Objective**: Automatically capture HTTP metrics for all requests

**Implementation** (`internal/middleware/instrumentation.go`):

```go
package middleware

import (
    "context"
    "net/http"
    "strconv"
    "time"

    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/metric"
    "github.com/yourusername/prometheus-poc/internal/metrics"
)

var (
    requestCounter  metric.Int64Counter
    requestDuration metric.Float64Histogram
    inFlightGauge   metric.Int64UpDownCounter
)

func init() {
    var err error

    // Initialize counter
    requestCounter, err = metrics.Meter.Int64Counter(
        "http_requests_total",
        metric.WithDescription("Total number of HTTP requests"),
    )
    if err != nil {
        panic(err)
    }

    // Initialize histogram
    requestDuration, err = metrics.Meter.Float64Histogram(
        "http_request_duration_seconds",
        metric.WithDescription("HTTP request duration in seconds"),
    )
    if err != nil {
        panic(err)
    }

    // Initialize gauge (using UpDownCounter)
    inFlightGauge, err = metrics.Meter.Int64UpDownCounter(
        "http_requests_in_flight",
        metric.WithDescription("Current number of in-flight requests"),
    )
    if err != nil {
        panic(err)
    }
}

// Metrics middleware captures request metrics
func Metrics(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Increment in-flight gauge
        ctx := r.Context()
        inFlightGauge.Add(ctx, 1, metric.WithAttributes(
            attribute.String("method", r.Method),
        ))

        // Wrap response writer to capture status code
        wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

        // Call next handler
        next.ServeHTTP(wrapped, r)

        // Calculate duration
        duration := time.Since(start).Seconds()

        // Common attributes
        attrs := metric.WithAttributes(
            attribute.String("method", r.Method),
            attribute.String("endpoint", r.URL.Path),
            attribute.String("status", strconv.Itoa(wrapped.statusCode)),
        )

        // Record metrics
        requestCounter.Add(ctx, 1, attrs)
        requestDuration.Record(ctx, duration, metric.WithAttributes(
            attribute.String("method", r.Method),
            attribute.String("endpoint", r.URL.Path),
        ))

        // Decrement in-flight gauge
        inFlightGauge.Add(ctx, -1, metric.WithAttributes(
            attribute.String("method", r.Method),
        ))
    })
}

// responseWriter wraps http.ResponseWriter to capture status code
type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

**Update main.go to use middleware**:
```go
router.Use(middleware.Metrics)
```

**Deliverables**:
- Middleware functions in `internal/middleware/instrumentation.go`
- Integration in `cmd/api/main.go`
- All endpoints automatically instrumented

**Verification**:
```bash
# Make some requests
curl http://localhost:8080/health
curl http://localhost:8080/api/users

# Check metrics
curl http://localhost:8080/metrics | grep http_requests_total
```

---

## Step 6: Add Custom Business Metrics

**Objective**: Demonstrate domain-specific metrics in handlers

**Implementation**:

**1. Orders Handler Enhancements** (`internal/handlers/orders.go`):
```go
import (
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/metric"
    "github.com/yourusername/prometheus-poc/internal/metrics"
)

var (
    ordersCounter      metric.Int64Counter
    orderValueCounter  metric.Float64Counter
)

func init() {
    var err error

    ordersCounter, err = metrics.Meter.Int64Counter(
        "orders_total",
        metric.WithDescription("Total number of orders"),
    )
    if err != nil {
        panic(err)
    }

    orderValueCounter, err = metrics.Meter.Float64Counter(
        "orders_value_total",
        metric.WithDescription("Total value of all orders"),
    )
    if err != nil {
        panic(err)
    }
}

func OrdersHandler(w http.ResponseWriter, r *http.Request) {
    // ... existing code ...

    // Success - record business metrics
    ctx := r.Context()
    ordersCounter.Add(ctx, 1, metric.WithAttributes(
        attribute.String("status", "success"),
    ))
    orderValueCounter.Add(ctx, req.Total)

    // ... send response ...
}
```

**2. Users Handler Enhancements** (`internal/handlers/users.go`):
```go
var (
    cacheCounter metric.Int64Counter
)

func init() {
    var err error
    cacheCounter, err = metrics.Meter.Int64Counter(
        "cache_hits_total",
        metric.WithDescription("Cache hit/miss count"),
    )
    if err != nil {
        panic(err)
    }
}

func UsersHandler(w http.ResponseWriter, r *http.Request) {
    // Simulate cache hit/miss
    ctx := r.Context()
    if rand.Float32() < 0.7 {
        cacheCounter.Add(ctx, 1, metric.WithAttributes(
            attribute.String("result", "hit"),
        ))
    } else {
        cacheCounter.Add(ctx, 1, metric.WithAttributes(
            attribute.String("result", "miss"),
        ))
    }

    // ... rest of handler ...
}
```

**Deliverables**:
- Business metrics integrated into handlers
- Proper error handling with metric labels

---

## Step 7: OTEL Collector and Prometheus Configuration

**Objective**: Configure OTEL Collector to receive metrics and expose them to Prometheus

### 7.1: Create OTEL Collector Configuration

**Implementation** (`config/otel-collector-config.yaml`):

```yaml
# OTEL Collector configuration
# Receives metrics via OTLP and exports to Prometheus

receivers:
  # OTLP receiver for metrics
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317  # OTLP gRPC receiver
      http:
        endpoint: 0.0.0.0:4318  # OTLP HTTP receiver

processors:
  # Batch processor to batch metrics before export
  batch:
    timeout: 10s
    send_batch_size: 100

exporters:
  # Prometheus exporter exposes metrics in Prometheus format
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: "otel"
    const_labels:
      environment: "demo"

  # Logging exporter for debugging (optional)
  logging:
    loglevel: info
    sampling_initial: 5
    sampling_thereafter: 200

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus, logging]
```

**Configuration Breakdown**:
- **Receivers**: Accept OTLP metrics on ports 4317 (gRPC) and 4318 (HTTP)
- **Processors**: Batch metrics to optimize export
- **Exporters**:
  - Prometheus exporter on port 8889
  - Logging exporter for debugging
- **Pipeline**: Routes metrics from receiver → processor → exporters

### 7.2: Create Prometheus Configuration

**Implementation** (`config/prometheus.yml`):

```yaml
global:
  scrape_interval: 15s       # How often to scrape targets by default
  evaluation_interval: 15s    # How often to evaluate rules

scrape_configs:
  # Scrape metrics from OTEL Collector (not directly from Go API)
  - job_name: 'otel-collector'
    static_configs:
      - targets: ['otel-collector:8889']  # OTEL Collector Prometheus exporter
    metrics_path: '/metrics'
    scrape_interval: 10s       # Scrape every 10 seconds
```

**Configuration Details**:
- Scrape interval: 10 seconds (frequent for demo purposes)
- Target: OTEL Collector service (via Docker network name `otel-collector:8889`)
- **Key change**: Prometheus scrapes from Collector, NOT from Go API
- No authentication required

**Deliverables**:
- `config/otel-collector-config.yaml` file
- `config/prometheus.yml` file
- Comments explaining configuration options

**Verification** (after Docker Compose is running):
```bash
# Check OTEL Collector is exposing metrics
curl http://localhost:8889/metrics

# Should see OTEL metrics in Prometheus format
```

---

## Step 8: Docker Compose Setup

**Objective**: Containerize application, OTEL Collector, and Prometheus

**Implementation** (`docker-compose.yml`):

```yaml
version: '3.8'

services:
  # Go API - sends metrics via OTLP
  api:
    build: .
    container_name: go-api
    ports:
      - "8080:8080"
    environment:
      - ENV=development
    networks:
      - monitoring
    depends_on:
      - otel-collector
    restart: unless-stopped

  # OTEL Collector - receives OTLP metrics and exposes to Prometheus
  otel-collector:
    image: otel/opentelemetry-collector:latest
    container_name: otel-collector
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./config/otel-collector-config.yaml:/etc/otel-collector-config.yaml:ro
    ports:
      - "4317:4317"   # OTLP gRPC receiver
      - "4318:4318"   # OTLP HTTP receiver
      - "8889:8889"   # Prometheus exporter
    networks:
      - monitoring
    restart: unless-stopped

  # Prometheus - scrapes metrics from OTEL Collector
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
    networks:
      - monitoring
    depends_on:
      - otel-collector
    restart: unless-stopped

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus-data:
```

**Dockerfile**:
```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build binary
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o /api ./cmd/api

# Final stage
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

COPY --from=builder /api .

EXPOSE 8080

CMD ["./api"]
```

**.dockerignore**:
```
.git
.gitignore
README.md
docs/
scripts/
*.md
.env
```

**Deliverables**:
- `docker-compose.yml`
- `Dockerfile`
- `.dockerignore` file

**Verification**:
```bash
# Build and start services
docker-compose up -d

# Check running containers
docker-compose ps

# View logs
docker-compose logs -f api

# Access Prometheus UI
open http://localhost:9090
```

---

## Step 9: Load Generation Script

**Objective**: Create script to generate realistic traffic

**Implementation** (`scripts/load-test.sh`):

```bash
#!/bin/bash

API_URL="${API_URL:-http://localhost:8080}"
DURATION="${DURATION:-300}"  # 5 minutes default

echo "==================================="
echo "Load Test Configuration"
echo "==================================="
echo "API URL: $API_URL"
echo "Duration: $DURATION seconds"
echo "==================================="

# Function to generate traffic
generate_traffic() {
    local counter=0
    while true; do
        counter=$((counter + 1))

        # 50% health checks (fast)
        curl -s "$API_URL/health" > /dev/null &

        # 30% users requests
        if [ $((RANDOM % 10)) -lt 3 ]; then
            curl -s "$API_URL/api/users" > /dev/null &
        fi

        # 15% orders requests
        if [ $((RANDOM % 10)) -lt 2 ]; then
            curl -s -X POST "$API_URL/api/orders" \
                -H "Content-Type: application/json" \
                -d "{\"user_id\":\"user$counter\",\"items\":[{\"id\":\"item1\",\"qty\":2}],\"total\":99.99}" \
                > /dev/null &
        fi

        # 5% slow requests
        if [ $((RANDOM % 20)) -eq 0 ]; then
            curl -s "$API_URL/api/slow" > /dev/null &
        fi

        # ~10 requests per second
        sleep 0.1
    done
}

echo "Starting load test..."
generate_traffic &
PID=$!

# Run for specified duration
sleep $DURATION

# Stop traffic generation
kill $PID 2>/dev/null
wait $PID 2>/dev/null

echo ""
echo "Load test complete!"
echo "Check Prometheus at http://localhost:9090"
```

**Make executable**:
```bash
chmod +x scripts/load-test.sh
```

**Traffic Patterns**:
- **Steady state**: ~10 req/s baseline
- **Endpoint distribution**: Weighted (50% health, 30% users, 15% orders, 5% slow)
- **Error injection**: Built into API handlers
- **Configurable**: Environment variables for URL and duration

**Optional Python Script** (`scripts/load-test.py`):
```python
#!/usr/bin/env python3

import asyncio
import aiohttp
import random
import time
from datetime import datetime

API_URL = "http://localhost:8080"
DURATION = 300  # 5 minutes
RPS = 10  # Requests per second

async def make_request(session, url, method="GET", json=None):
    try:
        async with session.request(method, url, json=json) as response:
            await response.read()
    except Exception as e:
        pass

async def generate_traffic():
    async with aiohttp.ClientSession() as session:
        start_time = time.time()

        while time.time() - start_time < DURATION:
            tasks = []

            # Generate weighted requests
            for _ in range(RPS):
                rand = random.random()

                if rand < 0.50:  # 50% health
                    tasks.append(make_request(session, f"{API_URL}/health"))
                elif rand < 0.80:  # 30% users
                    tasks.append(make_request(session, f"{API_URL}/api/users"))
                elif rand < 0.95:  # 15% orders
                    order_data = {
                        "user_id": f"user{random.randint(1, 1000)}",
                        "items": [{"id": "item1", "qty": 2}],
                        "total": 99.99
                    }
                    tasks.append(make_request(session, f"{API_URL}/api/orders", "POST", order_data))
                else:  # 5% slow
                    tasks.append(make_request(session, f"{API_URL}/api/slow"))

            await asyncio.gather(*tasks)
            await asyncio.sleep(1)

        print("Load test complete!")

if __name__ == "__main__":
    print(f"Starting load test at {datetime.now()}")
    print(f"Target: {API_URL}")
    print(f"Duration: {DURATION}s at {RPS} req/s")
    asyncio.run(generate_traffic())
```

**Deliverables**:
- `scripts/load-test.sh` (executable)
- Optional `scripts/load-test.py`
- Usage documentation in comments

**Verification**:
```bash
# Run load test
./scripts/load-test.sh

# Custom duration
DURATION=60 ./scripts/load-test.sh

# Different API URL
API_URL=http://api:8080 ./scripts/load-test.sh
```

---

## Step 10: Testing

See [testing-checklist.md](testing-checklist.md) for complete testing procedures.

---

## Step 11: Documentation

See [demo-walkthrough.md](demo-walkthrough.md) for demo instructions.

---

## Step 12: Verification

After completing all steps, verify the entire system:

1. **Start the stack**:
   ```bash
   docker-compose up -d
   ```

2. **Generate traffic**:
   ```bash
   ./scripts/load-test.sh
   ```

3. **Access Prometheus**:
   ```bash
   open http://localhost:9090
   ```

4. **Run sample queries** (see [metrics-guide.md](metrics-guide.md))

---

## Timeline Estimate

| Step | Task | Estimated Time |
|------|------|----------------|
| 1 | Initialize Go project | 15 min |
| 2 | Build basic API endpoints | 1-2 hours |
| 3 | Add OTEL dependencies | 10 min |
| 4 | Configure OTEL metrics | 1 hour |
| 5 | Implement middleware | 1 hour |
| 6 | Add business metrics | 30 min |
| 7 | Prometheus configuration | 15 min |
| 8 | Docker Compose setup | 30 min |
| 9 | Load generation script | 45 min |

**Total**: ~6-7 hours (for experienced developer)

---

## Next Steps

After completing the implementation:
- Review [metrics-guide.md](metrics-guide.md) for Prometheus query examples
- Follow [demo-walkthrough.md](demo-walkthrough.md) for running the demo
- Check [testing-checklist.md](testing-checklist.md) for verification
- Explore optional [extensions](implementation-plan.md#extensions-optional)
