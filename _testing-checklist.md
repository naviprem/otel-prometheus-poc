# Testing Checklist & Success Criteria

This document provides comprehensive testing procedures and success criteria for the OTEL metrics demonstration project.

---

## Pre-Deployment Checklist

### Code Completeness

- [ ] All endpoint handlers implemented (`health`, `users`, `orders`, `slow`)
- [ ] OTEL metrics initialization code complete
- [ ] Middleware for automatic instrumentation implemented
- [ ] Custom business metrics added to handlers
- [ ] Graceful shutdown implemented
- [ ] Error handling in all handlers

### Configuration Files

- [ ] `go.mod` with all dependencies
- [ ] `prometheus.yml` with scrape configuration
- [ ] `docker-compose.yml` with both services
- [ ] `Dockerfile` with multi-stage build
- [ ] `.dockerignore` to exclude unnecessary files

### Scripts

- [ ] `load-test.sh` executable and functional
- [ ] Optional `load-test.py` if implemented
- [ ] Proper shebang and permissions set

### Documentation

- [ ] `README.md` with quick start guide
- [ ] Architecture documentation
- [ ] Step-by-step implementation guide
- [ ] Metrics query reference
- [ ] Demo walkthrough instructions
- [ ] Testing checklist (this file)
- [ ] Resources and learning materials

---

## Functional Testing

### API Endpoints

#### Health Endpoint
```bash
curl -s http://localhost:8080/health | jq
```

**Expected**:
- [ ] Returns HTTP 200
- [ ] Response time < 50ms
- [ ] JSON contains `status: "ok"`
- [ ] JSON contains timestamp

#### Users Endpoint
```bash
# Test 10 times to verify variability
for i in {1..10}; do
  curl -s http://localhost:8080/api/users -w "\nHTTP: %{http_code}, Time: %{time_total}s\n"
done
```

**Expected**:
- [ ] Returns HTTP 200 most of the time (~95%)
- [ ] Occasionally returns HTTP 500 (~5%)
- [ ] Response time between 50-200ms
- [ ] Returns array of user objects on success

#### Orders Endpoint
```bash
# Valid request
curl -s -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"user_id":"test123","items":[{"id":"item1","qty":2}],"total":99.99}' \
  | jq

# Invalid request (missing fields)
curl -s -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"user_id":"test123"}' \
  -w "\nHTTP: %{http_code}\n"
```

**Expected**:
- [ ] Valid request returns HTTP 201 (~90%)
- [ ] Valid request occasionally returns HTTP 500 (~10%)
- [ ] Invalid request returns HTTP 400
- [ ] Response time between 100-300ms
- [ ] Returns order object with `order_id`, `status`, `created_at`

#### Slow Endpoint
```bash
time curl -s http://localhost:8080/api/slow | jq
```

**Expected**:
- [ ] Returns HTTP 200
- [ ] Response time between 3-5 seconds
- [ ] Returns JSON with message

---

## Metrics Testing

### Metrics Endpoint Availability

```bash
curl -s http://localhost:8080/metrics | head -20
```

**Expected**:
- [ ] Returns HTTP 200
- [ ] Response is in Prometheus exposition format
- [ ] Contains `# HELP` and `# TYPE` comments
- [ ] No authentication required

### HTTP Metrics Presence

```bash
curl -s http://localhost:8080/metrics | grep http_requests_total
```

**Expected**:
- [ ] `http_requests_total` metric present
- [ ] Has labels: `method`, `endpoint`, `status`
- [ ] Counter type

```bash
curl -s http://localhost:8080/metrics | grep http_request_duration_seconds
```

**Expected**:
- [ ] `http_request_duration_seconds_bucket` present
- [ ] `http_request_duration_seconds_sum` present
- [ ] `http_request_duration_seconds_count` present
- [ ] Has labels: `method`, `endpoint`, `le`
- [ ] Histogram type

```bash
curl -s http://localhost:8080/metrics | grep http_requests_in_flight
```

**Expected**:
- [ ] `http_requests_in_flight` metric present
- [ ] Has label: `method`
- [ ] Gauge type (UpDownCounter)

### Business Metrics Presence

Generate some orders first:
```bash
for i in {1..5}; do
  curl -s -X POST http://localhost:8080/api/orders \
    -H "Content-Type: application/json" \
    -d "{\"user_id\":\"user$i\",\"items\":[{\"id\":\"item1\",\"qty\":2}],\"total\":99.99}" \
    > /dev/null
done
```

Then check metrics:
```bash
curl -s http://localhost:8080/metrics | grep -E "(orders_total|orders_value_total|cache_hits_total)"
```

**Expected**:
- [ ] `orders_total` metric present with `status` label
- [ ] `orders_value_total` metric present
- [ ] `cache_hits_total` metric present with `result` label
- [ ] Values > 0 after generating traffic

---

## Prometheus Integration Testing

### Prometheus Startup

```bash
docker-compose up -d prometheus
docker-compose logs prometheus
```

**Expected**:
- [ ] Prometheus container starts successfully
- [ ] No error messages in logs
- [ ] Web UI accessible at http://localhost:9090

### Target Discovery

Navigate to http://localhost:9090/targets

**Expected**:
- [ ] `go-api` job listed
- [ ] Target state is **UP** (green)
- [ ] Target endpoint is `http://api:8080/metrics`
- [ ] Last scrape shows recent timestamp
- [ ] No scrape errors

### Configuration Load

Navigate to http://localhost:9090/config

**Expected**:
- [ ] Configuration matches `config/prometheus.yml`
- [ ] Scrape interval is 10 seconds
- [ ] Target is correctly configured

### Metrics Scraping

In Prometheus UI, run:
```promql
up{job="go-api"}
```

**Expected**:
- [ ] Returns value `1` (target is up)
- [ ] Shows recent timestamp

### Data Ingestion

Generate traffic, then run:
```promql
http_requests_total
```

**Expected**:
- [ ] Metric appears in autocomplete
- [ ] Returns time series data
- [ ] Data is recent (within last scrape interval)
- [ ] Multiple label combinations present

---

## Query Testing

### Basic Queries

```promql
# Total requests
http_requests_total

# Request rate
rate(http_requests_total[1m])

# Requests by endpoint
sum by (endpoint) (http_requests_total)
```

**Expected**:
- [ ] All queries execute without errors
- [ ] Return expected data types
- [ ] Time series have correct labels

### Advanced Queries

```promql
# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[1m])) /
sum(rate(http_requests_total[1m])) * 100

# P95 latency
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket[1m])))

# In-flight requests
http_requests_in_flight
```

**Expected**:
- [ ] Queries execute successfully
- [ ] Results are within expected ranges
- [ ] No division by zero errors
- [ ] Percentiles calculated correctly

---

## Load Testing

### Script Execution

```bash
./scripts/load-test.sh &
LOAD_PID=$!
```

**Expected**:
- [ ] Script starts without errors
- [ ] Generates ~10 requests per second
- [ ] No permission errors
- [ ] Can be stopped with `kill $LOAD_PID`

### Traffic Distribution

After running load test for 1 minute, query:
```promql
sum by (endpoint) (increase(http_requests_total[1m]))
```

**Expected distribution** (approximate):
- [ ] `/health`: ~300 requests (50%)
- [ ] `/api/users`: ~180 requests (30%)
- [ ] `/api/orders`: ~90 requests (15%)
- [ ] `/api/slow`: ~30 requests (5%)

### Error Generation

Query after 1 minute of load test:
```promql
sum by (endpoint) (increase(http_requests_total{status=~"5.."}[1m]))
```

**Expected**:
- [ ] `/api/users`: ~9 errors (5% of 180)
- [ ] `/api/orders`: ~9 errors (10% of 90)
- [ ] `/health` and `/api/slow`: 0 errors

---

## Performance Testing

### Response Time Verification

```promql
histogram_quantile(0.95,
  sum by (endpoint, le) (rate(http_request_duration_seconds_bucket[5m])))
```

**Expected** (while load test is running):
- [ ] `/health`: < 0.05 seconds
- [ ] `/api/users`: 0.1-0.25 seconds
- [ ] `/api/orders`: 0.15-0.35 seconds
- [ ] `/api/slow`: 3-5 seconds

### Throughput Verification

```promql
sum(rate(http_requests_total[1m]))
```

**Expected**:
- [ ] With one load test: ~10 req/s
- [ ] API handles load without crashing
- [ ] No significant error rate increase

### Concurrency Verification

```promql
max_over_time(http_requests_in_flight[5m])
```

**Expected**:
- [ ] Value > 0 (indicating concurrent requests)
- [ ] Higher when slow endpoint is active
- [ ] Returns to 0 when traffic stops

---

## Docker Compose Testing

### Build Process

```bash
docker-compose build
```

**Expected**:
- [ ] Build completes successfully
- [ ] No build errors
- [ ] Image size is reasonable (< 50MB for Go API)

### Service Dependencies

```bash
docker-compose up -d
docker-compose ps
```

**Expected**:
- [ ] Both services start
- [ ] Both show status as "Up"
- [ ] Network is created
- [ ] Volume is created

### Network Connectivity

```bash
# Test from Prometheus to API
docker exec prometheus wget -qO- http://api:8080/health
```

**Expected**:
- [ ] Returns health check JSON
- [ ] No network errors
- [ ] Services can communicate

### Volume Persistence

```bash
# Create data
./scripts/load-test.sh &
sleep 30
kill %1

# Restart Prometheus
docker-compose restart prometheus

# Check Prometheus UI - historical data should persist
```

**Expected**:
- [ ] Data persists across restart
- [ ] No data loss
- [ ] Prometheus reads from volume

---

## Documentation Testing

### README Accuracy

Follow `README.md` quick start:

```bash
docker-compose up -d
./scripts/load-test.sh
open http://localhost:9090
```

**Expected**:
- [ ] Instructions are accurate
- [ ] Commands work as written
- [ ] Links are valid
- [ ] Examples produce expected results

### Code Examples

Test code snippets from documentation:

**Expected**:
- [ ] All curl commands work
- [ ] All PromQL queries work
- [ ] All bash commands execute
- [ ] Results match documentation

---

## Success Criteria

The project is considered successful when ALL of the following are true:

### Functional Success

1. [ ] **API Operational**: All endpoints respond correctly
2. [ ] **Metrics Exposed**: `/metrics` endpoint returns valid Prometheus format
3. [ ] **Scraping Works**: Prometheus successfully scrapes API
4. [ ] **Queries Work**: PromQL queries return expected data
5. [ ] **Load Generation**: Script generates realistic traffic

### Metric Quality

1. [ ] **HTTP Metrics**: Request counter, duration histogram, in-flight gauge working
2. [ ] **Business Metrics**: Orders, revenue, cache metrics working
3. [ ] **Labels**: All metrics have appropriate labels
4. [ ] **Cardinality**: Label cardinality is reasonable (not exploding)
5. [ ] **Accuracy**: Metrics accurately reflect application behavior

### Performance

1. [ ] **API Performance**: Endpoints respond within expected latency ranges
2. [ ] **Scrape Performance**: Prometheus scrapes complete in < 1 second
3. [ ] **Query Performance**: Queries return in < 5 seconds
4. [ ] **Stability**: System runs for 5+ minutes without crashes

### Documentation

1. [ ] **Completeness**: All components documented
2. [ ] **Accuracy**: Documentation matches implementation
3. [ ] **Clarity**: Instructions are clear and actionable
4. [ ] **Examples**: Code examples work as written

### Educational Value

1. [ ] **Demonstrates OTEL**: Shows key OTEL metrics concepts
2. [ ] **Demonstrates Prometheus**: Shows Prometheus scraping and querying
3. [ ] **Realistic**: Simulates realistic monitoring scenarios
4. [ ] **Accessible**: Easy for others to run and understand

---

## Regression Testing

After making changes, run this quick regression test:

```bash
#!/bin/bash

echo "1. Build and start services..."
docker-compose up -d --build

echo "2. Wait for services to be ready..."
sleep 5

echo "3. Test API endpoints..."
curl -f http://localhost:8080/health || exit 1
curl -f http://localhost:8080/api/users || exit 1

echo "4. Test metrics endpoint..."
curl -f http://localhost:8080/metrics | grep -q http_requests_total || exit 1

echo "5. Verify Prometheus scraping..."
docker exec prometheus wget -qO- http://api:8080/metrics | grep -q http_requests_total || exit 1

echo "6. Start load test..."
timeout 60 ./scripts/load-test.sh

echo "7. Verify metrics in Prometheus..."
# Add Prometheus API query test here

echo "✅ All tests passed!"
docker-compose down
```

---

## Troubleshooting Tests

### Test: Verify Port Availability

```bash
# Check if ports are free before starting
lsof -i :8080
lsof -i :9090
```

**Expected**: No output (ports are free)

### Test: Verify Docker Resources

```bash
docker system df
docker stats --no-stream
```

**Expected**:
- [ ] Sufficient disk space
- [ ] Containers using reasonable resources

### Test: Verify Network

```bash
docker network ls | grep prometheus-poc
docker network inspect prometheus-poc_monitoring
```

**Expected**:
- [ ] Network exists
- [ ] Both containers connected

---

## Continuous Integration Tests

If setting up CI/CD, include these tests:

1. [ ] **Build Test**: `docker-compose build` succeeds
2. [ ] **Unit Tests**: Go unit tests pass (if implemented)
3. [ ] **Integration Test**: Services start and communicate
4. [ ] **Smoke Test**: Basic API calls succeed
5. [ ] **Metrics Test**: Metrics endpoint returns valid data

---

## Sign-Off Checklist

Before considering the project complete:

- [ ] All functional tests pass
- [ ] All metrics tests pass
- [ ] All Prometheus integration tests pass
- [ ] All query tests pass
- [ ] Load testing works correctly
- [ ] Docker Compose setup works
- [ ] Documentation is accurate
- [ ] Demo can be run successfully
- [ ] No critical bugs or issues
- [ ] Code is commented appropriately

---

## Next Steps After Testing

Once all tests pass:

1. Tag a release version
2. Consider setting up CI/CD pipeline
3. Explore optional extensions (Grafana, tracing, etc.)
4. Use as template for real projects
5. Share with team for learning
