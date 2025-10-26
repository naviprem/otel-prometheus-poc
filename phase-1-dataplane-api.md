# Phase 1: Data Plane - Go API Service

## Overview
Build a Go-based API service instrumented with OpenTelemetry to generate realistic metrics for the PoC.

---

## Step 1: Initialize Go Project

### 1.1 Create Project Directory
```bash
mkdir -p dataplane/api
cd dataplane/api
```

### 1.2 Initialize Go Module
```bash
go mod init github.com/yourusername/otel-prometheus-poc/dataplane/api
```

### 1.3 Create Project Structure
```bash
mkdir -p handlers
mkdir -p middleware
mkdir -p config
mkdir -p models
```

Your structure should look like:
```
dataplane/api/
├── main.go
├── handlers/
│   ├── health.go
│   ├── products.go
│   ├── orders.go
│   ├── reports.go
│   ├── search.go
│   └── payments.go
├── middleware/
│   └── logging.go
├── config/
│   └── config.go
└── models/
    └── response.go
```

---

## Step 2: Install Dependencies

### 2.1 Install Core Dependencies
```bash
# HTTP router
go get github.com/gorilla/mux

# OpenTelemetry SDK
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/otel/sdk/metric
go get go.opentelemetry.io/otel/sdk/resource
go get go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp

# OpenTelemetry instrumentation
go get go.opentelemetry.io/contrib/instrumentation/github.com/gorilla/mux/otelmux

# Semantic conventions
go get go.opentelemetry.io/otel/semconv/v1.24.0
```

### 2.2 Verify Dependencies
```bash
go mod tidy
```

---

## Step 3: Create Configuration Module

### 3.1 Create `config/config.go`
```go
package config

import (
	"os"
	"strconv"
)

type Config struct {
	Port             string
	Environment      string
	DataPlaneID      string
	OTELEndpoint     string
	ServiceName      string
	ServiceVersion   string
	ErrorRate        float64  // Simulated error rate (0.0 to 1.0)
	EnableSlowRequests bool
}

func LoadConfig() *Config {
	return &Config{
		Port:             getEnv("PORT", "8080"),
		Environment:      getEnv("ENVIRONMENT", "dev"),
		DataPlaneID:      getEnv("DATA_PLANE_ID", "dp-dev-01"),
		OTELEndpoint:     getEnv("OTEL_ENDPOINT", "http://localhost:4318"),
		ServiceName:      getEnv("SERVICE_NAME", "api-service"),
		ServiceVersion:   getEnv("SERVICE_VERSION", "1.0.0"),
		ErrorRate:        getEnvFloat("ERROR_RATE", 0.05),
		EnableSlowRequests: getEnvBool("ENABLE_SLOW_REQUESTS", true),
	}
}

func getEnv(key, defaultValue string) string {
	if value := os.Getenv(key); value != "" {
		return value
	}
	return defaultValue
}

func getEnvFloat(key string, defaultValue float64) float64 {
	if value := os.Getenv(key); value != "" {
		if f, err := strconv.ParseFloat(value, 64); err == nil {
			return f
		}
	}
	return defaultValue
}

func getEnvBool(key string, defaultValue bool) bool {
	if value := os.Getenv(key); value != "" {
		if b, err := strconv.ParseBool(value); err == nil {
			return b
		}
	}
	return defaultValue
}
```

---

## Step 4: Create Response Models

### 4.1 Create `models/response.go`
```go
package models

import (
	"encoding/json"
	"net/http"
	"time"
)

type Response struct {
	Success   bool        `json:"success"`
	Message   string      `json:"message,omitempty"`
	Data      interface{} `json:"data,omitempty"`
	Timestamp time.Time   `json:"timestamp"`
}

type ErrorResponse struct {
	Success   bool      `json:"success"`
	Error     string    `json:"error"`
	Timestamp time.Time `json:"timestamp"`
}

func SendJSON(w http.ResponseWriter, statusCode int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(statusCode)
	json.NewEncoder(w).Encode(data)
}

func SendSuccess(w http.ResponseWriter, message string, data interface{}) {
	SendJSON(w, http.StatusOK, Response{
		Success:   true,
		Message:   message,
		Data:      data,
		Timestamp: time.Now(),
	})
}

func SendError(w http.ResponseWriter, statusCode int, message string) {
	SendJSON(w, statusCode, ErrorResponse{
		Success:   false,
		Error:     message,
		Timestamp: time.Now(),
	})
}
```

---

## Step 5: Implement API Handlers

### 5.1 Create `handlers/health.go`
```go
package handlers

import (
	"net/http"
	"github.com/yourusername/otel-prometheus-poc/dataplane/api/models"
)

func HealthHandler(w http.ResponseWriter, r *http.Request) {
	models.SendSuccess(w, "Service is healthy", map[string]string{
		"status": "UP",
	})
}
```

### 5.2 Create `handlers/products.go`
```go
package handlers

import (
	"net/http"
	"time"
	"github.com/yourusername/otel-prometheus-poc/dataplane/api/models"
)

type Product struct {
	ID          string  `json:"id"`
	Name        string  `json:"name"`
	Price       float64 `json:"price"`
	Category    string  `json:"category"`
}

func ProductsHandler(w http.ResponseWriter, r *http.Request) {
	// Simulate fast response (10-50ms)
	time.Sleep(time.Duration(10+randomInt(40)) * time.Millisecond)

	products := []Product{
		{ID: "1", Name: "Laptop", Price: 999.99, Category: "Electronics"},
		{ID: "2", Name: "Mouse", Price: 29.99, Category: "Electronics"},
		{ID: "3", Name: "Keyboard", Price: 79.99, Category: "Electronics"},
		{ID: "4", Name: "Monitor", Price: 299.99, Category: "Electronics"},
		{ID: "5", Name: "Headphones", Price: 149.99, Category: "Electronics"},
	}

	models.SendSuccess(w, "Products retrieved successfully", products)
}
```

### 5.3 Create `handlers/orders.go`
```go
package handlers

import (
	"encoding/json"
	"math/rand"
	"net/http"
	"time"
	"github.com/yourusername/otel-prometheus-poc/dataplane/api/models"
)

type Order struct {
	OrderID     string    `json:"order_id"`
	ProductID   string    `json:"product_id"`
	Quantity    int       `json:"quantity"`
	TotalAmount float64   `json:"total_amount"`
	CreatedAt   time.Time `json:"created_at"`
}

func CreateOrderHandler(w http.ResponseWriter, r *http.Request) {
	// Simulate medium latency (50-200ms)
	time.Sleep(time.Duration(50+randomInt(150)) * time.Millisecond)

	var orderRequest struct {
		ProductID string `json:"product_id"`
		Quantity  int    `json:"quantity"`
	}

	if err := json.NewDecoder(r.Body).Decode(&orderRequest); err != nil {
		models.SendError(w, http.StatusBadRequest, "Invalid request body")
		return
	}

	order := Order{
		OrderID:     generateID(),
		ProductID:   orderRequest.ProductID,
		Quantity:    orderRequest.Quantity,
		TotalAmount: float64(orderRequest.Quantity) * 99.99,
		CreatedAt:   time.Now(),
	}

	models.SendSuccess(w, "Order created successfully", order)
}
```

### 5.4 Create `handlers/reports.go`
```go
package handlers

import (
	"net/http"
	"time"
	"github.com/yourusername/otel-prometheus-poc/dataplane/api/models"
)

type Report struct {
	ReportID     string    `json:"report_id"`
	Type         string    `json:"type"`
	GeneratedAt  time.Time `json:"generated_at"`
	RecordCount  int       `json:"record_count"`
}

func ReportsHandler(w http.ResponseWriter, r *http.Request) {
	// Simulate slow response (500-2000ms)
	time.Sleep(time.Duration(500+randomInt(1500)) * time.Millisecond)

	report := Report{
		ReportID:    generateID(),
		Type:        "Sales Report",
		GeneratedAt: time.Now(),
		RecordCount: 1000 + randomInt(5000),
	}

	models.SendSuccess(w, "Report generated successfully", report)
}
```

### 5.5 Create `handlers/search.go`
```go
package handlers

import (
	"net/http"
	"time"
	"github.com/yourusername/otel-prometheus-poc/dataplane/api/models"
)

func SearchHandler(w http.ResponseWriter, r *http.Request) {
	query := r.URL.Query().Get("q")
	if query == "" {
		models.SendError(w, http.StatusBadRequest, "Query parameter 'q' is required")
		return
	}

	// Variable latency based on query length
	latency := 20 + len(query)*5 + randomInt(100)
	time.Sleep(time.Duration(latency) * time.Millisecond)

	results := []map[string]interface{}{
		{"id": "1", "title": "Result 1 for: " + query, "relevance": 0.95},
		{"id": "2", "title": "Result 2 for: " + query, "relevance": 0.87},
		{"id": "3", "title": "Result 3 for: " + query, "relevance": 0.72},
	}

	models.SendSuccess(w, "Search completed", results)
}
```

### 5.6 Create `handlers/payments.go`
```go
package handlers

import (
	"encoding/json"
	"net/http"
	"time"
	"github.com/yourusername/otel-prometheus-poc/dataplane/api/config"
	"github.com/yourusername/otel-prometheus-poc/dataplane/api/models"
)

type Payment struct {
	PaymentID     string    `json:"payment_id"`
	OrderID       string    `json:"order_id"`
	Amount        float64   `json:"amount"`
	Status        string    `json:"status"`
	ProcessedAt   time.Time `json:"processed_at"`
}

var cfg *config.Config

func SetConfig(c *config.Config) {
	cfg = c
}

func PaymentsHandler(w http.ResponseWriter, r *http.Request) {
	// Simulate processing time
	time.Sleep(time.Duration(100+randomInt(200)) * time.Millisecond)

	var paymentRequest struct {
		OrderID string  `json:"order_id"`
		Amount  float64 `json:"amount"`
	}

	if err := json.NewDecoder(r.Body).Decode(&paymentRequest); err != nil {
		models.SendError(w, http.StatusBadRequest, "Invalid request body")
		return
	}

	// Simulate random errors based on configured error rate
	if cfg != nil && rand.Float64() < cfg.ErrorRate {
		models.SendError(w, http.StatusInternalServerError, "Payment processing failed")
		return
	}

	payment := Payment{
		PaymentID:   generateID(),
		OrderID:     paymentRequest.OrderID,
		Amount:      paymentRequest.Amount,
		Status:      "SUCCESS",
		ProcessedAt: time.Now(),
	}

	models.SendSuccess(w, "Payment processed successfully", payment)
}
```

### 5.7 Create `handlers/utils.go`
```go
package handlers

import (
	"math/rand"
	"time"
	"fmt"
)

func init() {
	rand.Seed(time.Now().UnixNano())
}

func randomInt(max int) int {
	return rand.Intn(max)
}

func generateID() string {
	return fmt.Sprintf("%d-%d", time.Now().UnixNano(), rand.Intn(1000))
}
```

---

## Step 6: Setup OpenTelemetry Instrumentation

### 6.1 Create `main.go` - Part 1: OTEL Setup
```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"time"

	"github.com/gorilla/mux"
	"go.opentelemetry.io/contrib/instrumentation/github.com/gorilla/mux/otelmux"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp"
	"go.opentelemetry.io/otel/sdk/metric"
	"go.opentelemetry.io/otel/sdk/resource"
	semconv "go.opentelemetry.io/otel/semconv/v1.24.0"

	"github.com/yourusername/otel-prometheus-poc/dataplane/api/config"
	"github.com/yourusername/otel-prometheus-poc/dataplane/api/handlers"
)

func initMeterProvider(ctx context.Context, cfg *config.Config) (*metric.MeterProvider, error) {
	// Create resource with service information
	res, err := resource.New(ctx,
		resource.WithAttributes(
			semconv.ServiceName(cfg.ServiceName),
			semconv.ServiceVersion(cfg.ServiceVersion),
			semconv.DeploymentEnvironment(cfg.Environment),
		),
		resource.WithAttributes(
			// Custom attributes for data plane identification
			attribute.String("data_plane_id", cfg.DataPlaneID),
			attribute.String("environment", cfg.Environment),
		),
	)
	if err != nil {
		return nil, fmt.Errorf("failed to create resource: %w", err)
	}

	// Create OTLP HTTP exporter
	exporter, err := otlpmetrichttp.New(ctx,
		otlpmetrichttp.WithEndpoint(cfg.OTELEndpoint),
		otlpmetrichttp.WithInsecure(), // Use insecure for local development
	)
	if err != nil {
		return nil, fmt.Errorf("failed to create exporter: %w", err)
	}

	// Create meter provider with periodic reader
	meterProvider := metric.NewMeterProvider(
		metric.WithResource(res),
		metric.WithReader(metric.NewPeriodicReader(exporter,
			metric.WithInterval(10*time.Second), // Export every 10 seconds
		)),
	)

	otel.SetMeterProvider(meterProvider)

	return meterProvider, nil
}
```

### 6.2 Create `main.go` - Part 2: HTTP Server
```go
func main() {
	// Load configuration
	cfg := config.LoadConfig()
	handlers.SetConfig(cfg)

	log.Printf("Starting API service: %s (version: %s)", cfg.ServiceName, cfg.ServiceVersion)
	log.Printf("Environment: %s, Data Plane ID: %s", cfg.Environment, cfg.DataPlaneID)
	log.Printf("OTEL Endpoint: %s", cfg.OTELEndpoint)

	ctx := context.Background()

	// Initialize OpenTelemetry
	meterProvider, err := initMeterProvider(ctx, cfg)
	if err != nil {
		log.Fatalf("Failed to initialize meter provider: %v", err)
	}
	defer func() {
		if err := meterProvider.Shutdown(ctx); err != nil {
			log.Printf("Error shutting down meter provider: %v", err)
		}
	}()

	// Create router with OTEL middleware
	router := mux.NewRouter()
	router.Use(otelmux.Middleware(cfg.ServiceName))

	// Register routes
	router.HandleFunc("/health", handlers.HealthHandler).Methods("GET")
	router.HandleFunc("/api/products", handlers.ProductsHandler).Methods("GET")
	router.HandleFunc("/api/orders", handlers.CreateOrderHandler).Methods("POST")
	router.HandleFunc("/api/reports", handlers.ReportsHandler).Methods("GET")
	router.HandleFunc("/api/search", handlers.SearchHandler).Methods("GET")
	router.HandleFunc("/api/payments", handlers.PaymentsHandler).Methods("POST")

	// Create HTTP server
	srv := &http.Server{
		Addr:         ":" + cfg.Port,
		Handler:      router,
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 15 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Start server in goroutine
	go func() {
		log.Printf("Server listening on port %s", cfg.Port)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("Server error: %v", err)
		}
	}()

	// Wait for interrupt signal
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, os.Interrupt)
	<-quit

	log.Println("Shutting down server...")

	// Graceful shutdown
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	if err := srv.Shutdown(ctx); err != nil {
		log.Fatalf("Server forced to shutdown: %v", err)
	}

	log.Println("Server stopped")
}
```

---

## Step 7: Create Dockerfile

### 7.1 Create `Dockerfile`
```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build binary
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o api-service .

# Runtime stage
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

# Copy binary from builder
COPY --from=builder /app/api-service .

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

# Run the binary
CMD ["./api-service"]
```

### 7.2 Create `.dockerignore`
```
.git
.gitignore
README.md
*.md
.env
```

---

## Step 8: Local Testing

### 8.1 Run Locally Without Docker
```bash
# Set environment variables
export PORT=8080
export ENVIRONMENT=dev
export DATA_PLANE_ID=dp-dev-01
export OTEL_ENDPOINT=localhost:4318
export SERVICE_NAME=api-service
export SERVICE_VERSION=1.0.0
export ERROR_RATE=0.05

# Run the application
go run main.go
```

### 8.2 Test Endpoints
```bash
# Health check
curl http://localhost:8080/health

# Get products
curl http://localhost:8080/api/products

# Create order
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"product_id": "1", "quantity": 2}'

# Generate report
curl http://localhost:8080/api/reports

# Search
curl "http://localhost:8080/api/search?q=laptop"

# Process payment
curl -X POST http://localhost:8080/api/payments \
  -H "Content-Type: application/json" \
  -d '{"order_id": "123", "amount": 199.98}'
```

### 8.3 Build Docker Image
```bash
cd dataplane/api
docker build -t otel-api-service:latest .
```

### 8.4 Run Docker Container
```bash
docker run -d \
  -p 8080:8080 \
  -e ENVIRONMENT=dev \
  -e DATA_PLANE_ID=dp-dev-01 \
  -e OTEL_ENDPOINT=host.docker.internal:4318 \
  --name api-dev \
  otel-api-service:latest
```

---

## Expected Outcomes

✅ **Functional API**:
- All endpoints respond correctly
- Proper error handling
- Simulated latency patterns

✅ **OTEL Integration**:
- Metrics exported to OTEL Collector endpoint
- Request counts, durations, and sizes tracked
- Proper resource attributes attached

✅ **Dockerized**:
- Clean multi-stage build
- Small image size (~20MB)
- Health check configured

---

## Troubleshooting

**Issue: Cannot connect to OTEL Collector**
```
Solution: Verify OTEL_ENDPOINT is correct
- For local: localhost:4318
- For Docker: host.docker.internal:4318 (Mac/Windows) or 172.17.0.1:4318 (Linux)
- For Docker Compose: otel-collector:4318
```

**Issue: Metrics not appearing**
```
Solution: Check logs for OTEL exporter errors
- Ensure OTEL Collector is running
- Verify network connectivity
- Check firewall rules
```

**Issue: Build fails with module errors**
```
Solution: Update module path in all files
- Replace github.com/yourusername/otel-prometheus-poc with your actual module path
- Run: go mod tidy
```

---

## Next Steps

Proceed to **Phase 2: OTEL Collector Configuration** to set up the collector that will receive these metrics.
