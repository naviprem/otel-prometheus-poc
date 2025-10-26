# Demo Walkthrough

This guide walks you through running a complete demonstration of OTEL metrics with Prometheus.

---

## Prerequisites

Before starting, ensure you have:

- **Docker** & **Docker Compose** installed
- **curl** (for manual testing)
- **bash** (for load test script)
- Port **8080** and **9090** available

**Verify prerequisites**:
```bash
docker --version
docker-compose --version
curl --version
```

---

## Step 1: Start the Stack

### Clone and Navigate
```bash
cd /path/to/prometheus-poc
```

### Build and Start Services
```bash
docker-compose up -d
```

**Expected output**:
```
Creating network "prometheus-poc_monitoring" with driver "bridge"
Creating volume "prometheus-poc_prometheus-data" with default driver
Creating go-api ... done
Creating prometheus ... done
```

### Verify Services are Running
```bash
docker-compose ps
```

**Expected output**:
```
   Name                 Command               State           Ports
-------------------------------------------------------------------------
go-api        ./api                            Up      0.0.0.0:8080->8080/tcp
prometheus    /bin/prometheus --config.f...   Up      0.0.0.0:9090->9090/tcp
```

### View Logs (Optional)
```bash
# All services
docker-compose logs -f

# API only
docker-compose logs -f api

# Prometheus only
docker-compose logs -f prometheus
```

---

## Step 2: Verify API is Working

### Test Health Endpoint
```bash
curl http://localhost:8080/health
```

**Expected response**:
```json
{
  "status": "ok",
  "timestamp": "2025-10-25T..."
}
```

### Test Users Endpoint
```bash
curl http://localhost:8080/api/users
```

**Expected response**:
```json
[
  {"id":"1","name":"Alice","email":"alice@example.com"},
  {"id":"2","name":"Bob","email":"bob@example.com"},
  {"id":"3","name":"Charlie","email":"charlie@example.com"}
]
```

### Test Orders Endpoint
```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"user_id":"user123","items":[{"id":"item1","qty":2}],"total":99.99}'
```

**Expected response**:
```json
{
  "order_id": "ORD-123456",
  "status": "created",
  "created_at": "2025-10-25T..."
}
```

### Test Slow Endpoint
```bash
curl http://localhost:8080/api/slow
```

**Expected response** (after 3-5 seconds):
```json
{
  "message": "This was intentionally slow"
}
```

---

## Step 3: Verify OTEL Collector

### Check OTEL Collector is Running
```bash
docker-compose logs otel-collector
```

**Expected output**: Should see logs indicating the collector started successfully

### Check OTEL Collector Metrics Endpoint
```bash
curl http://localhost:8889/metrics
```

**Expected output** (sample - after generating some traffic):
```
# HELP otel_http_requests_total Total number of HTTP requests
# TYPE otel_http_requests_total counter
otel_http_requests_total{endpoint="/health",environment="demo",method="GET",status="200"} 1

# HELP otel_http_request_duration_seconds HTTP request duration in seconds
# TYPE otel_http_request_duration_seconds histogram
otel_http_request_duration_seconds_bucket{endpoint="/health",environment="demo",method="GET",le="0.005"} 1
otel_http_request_duration_seconds_bucket{endpoint="/health",environment="demo",method="GET",le="+Inf"} 1
otel_http_request_duration_seconds_sum{endpoint="/health",environment="demo",method="GET"} 0.001234
otel_http_request_duration_seconds_count{endpoint="/health",environment="demo",method="GET"} 1
```

**Note**: The Go API no longer exposes a `/metrics` endpoint. Metrics are pushed via OTLP to the OTEL Collector, which then exposes them in Prometheus format.

---

## Step 4: Verify Prometheus is Scraping

### Access Prometheus UI
Open your browser to:
```
http://localhost:9090
```

### Check Targets
1. Click **Status** → **Targets** in the top menu
2. Verify the `otel-collector` target is **UP**

**Expected**:
- Endpoint: `http://otel-collector:8889/metrics`
- State: **UP** (green)
- Last Scrape: Recent timestamp

### Check Configuration
1. Click **Status** → **Configuration**
2. Verify your `prometheus.yml` is loaded correctly

---

## Step 5: Run Sample Queries

Navigate to **Graph** tab in Prometheus UI.

### Query 1: Total Request Rate
```promql
rate(http_requests_total[1m])
```

**What to expect**: Should show small rate from your manual tests

### Query 2: Request Count by Endpoint
```promql
sum by (endpoint) (http_requests_total)
```

**What to expect**: Breakdown showing /health, /api/users, /api/orders, /api/slow

### Query 3: Check In-Flight Requests
```promql
http_requests_in_flight
```

**What to expect**: Should be 0 (or small number if actively testing)

---

## Step 6: Generate Load

### Start Load Test Script
```bash
./scripts/load-test.sh
```

**Expected output**:
```
===================================
Load Test Configuration
===================================
API URL: http://localhost:8080
Duration: 300 seconds
===================================
Starting load test...
```

The script will run for 5 minutes (300 seconds) by default.

### Monitor in Real-Time

While the load test is running, switch to Prometheus UI and run these queries:

#### Live Request Rate
```promql
sum(rate(http_requests_total[1m]))
```

**What to expect**: ~10 requests/second

#### Request Distribution
```promql
sum by (endpoint) (rate(http_requests_total[1m]))
```

**What to expect**:
- `/health`: ~5 req/s (50%)
- `/api/users`: ~3 req/s (30%)
- `/api/orders`: ~1.5 req/s (15%)
- `/api/slow`: ~0.5 req/s (5%)

---

## Step 7: Explore Key Metrics

### Error Rate Monitoring

#### Total Error Rate
```promql
sum(rate(http_requests_total{status=~"5.."}[1m]))
```

**What to expect**: Small number of errors (~0.5-1 req/s from users endpoint 5% error rate and orders 10% error rate)

#### Error Percentage
```promql
sum(rate(http_requests_total{status=~"5.."}[1m])) /
sum(rate(http_requests_total[1m])) * 100
```

**What to expect**: ~6-8% (weighted average based on traffic distribution)

#### Errors by Endpoint
```promql
sum by (endpoint) (rate(http_requests_total{status=~"5.."}[1m]))
```

**What to expect**: Errors mostly from `/api/users` and `/api/orders`

### Latency Analysis

#### P95 Latency
```promql
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket[1m])))
```

**What to expect**: 3-5 seconds (influenced by slow endpoint)

#### P95 Latency Excluding Slow Endpoint
```promql
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket{endpoint!="/api/slow"}[1m])))
```

**What to expect**: 0.2-0.3 seconds

#### Latency by Endpoint
```promql
histogram_quantile(0.95,
  sum by (endpoint, le) (rate(http_request_duration_seconds_bucket[1m])))
```

**What to expect**:
- `/health`: <0.01s
- `/api/users`: ~0.2s
- `/api/orders`: ~0.3s
- `/api/slow`: ~5s

### Concurrent Requests

#### Current In-Flight
```promql
http_requests_in_flight
```

**What to expect**: 0-5 (varies, higher when slow endpoint is being hit)

#### Max In-Flight (Last 5 Minutes)
```promql
max_over_time(http_requests_in_flight[5m])
```

**What to expect**: 5-10 (peaks when multiple slow requests overlap)

### Business Metrics

#### Orders Per Minute
```promql
rate(orders_total[1m]) * 60
```

**What to expect**: ~1.5 orders/minute (15% of ~10 req/s = ~1.5 req/s, but 90% success = ~1.35 orders/min)

#### Revenue Per Minute
```promql
rate(orders_value_total[1m]) * 60
```

**What to expect**: ~$135/min (1.35 orders × $99.99)

#### Average Order Value
```promql
rate(orders_value_total[1m]) / rate(orders_total[1m])
```

**What to expect**: ~$99.99

#### Cache Hit Rate
```promql
rate(cache_hits_total{result="hit"}[1m]) /
rate(cache_hits_total[1m]) * 100
```

**What to expect**: ~70% (as configured in users handler)

---

## Step 8: Create Visualizations

### Graph View

1. Enter a query (e.g., `sum(rate(http_requests_total[1m]))`)
2. Click **Graph** tab
3. Adjust time range (e.g., last 15 minutes)

**What to see**: Line graph showing metric over time

### Multiple Queries on One Graph

1. Enter first query
2. Click **Add Query**
3. Enter second query
4. Click **Graph**

**Example**:
- Query 1: `sum(rate(http_requests_total{status=~"2.."}[1m]))`
- Query 2: `sum(rate(http_requests_total{status=~"5.."}[1m]))`

**What to see**: Success vs error rates overlaid

---

## Step 9: Common Scenarios

### Scenario 1: Identify Slow Endpoints

**Query**:
```promql
topk(3, sum by (endpoint) (rate(http_request_duration_seconds_sum[5m])) /
           sum by (endpoint) (rate(http_request_duration_seconds_count[5m])))
```

**Result**: `/api/slow` will be #1, followed by `/api/orders` and `/api/users`

**Action**: Focus optimization efforts on slowest endpoints

### Scenario 2: Detect Error Rate Increase

**Query**:
```promql
sum(rate(http_requests_total{status=~"5.."}[1m])) >
  1.5 * avg_over_time(sum(rate(http_requests_total{status=~"5.."}[1m]))[1h:1m])
```

**Result**: Returns values when error rate exceeds 1.5× the hourly average

**Action**: Set up alert based on this query

### Scenario 3: Monitor API Availability

**Query**:
```promql
sum(rate(http_requests_total{status=~"2.."}[5m])) /
sum(rate(http_requests_total[5m])) * 100
```

**Result**: Success rate percentage (should be >90%)

**Action**: Track against SLA targets

### Scenario 4: Capacity Planning

**Query**:
```promql
max_over_time(http_requests_in_flight[5m])
```

**Result**: Peak concurrent requests

**Action**: Size infrastructure based on peak load

---

## Step 10: Experiment with Traffic Patterns

### Run Short Burst Test
```bash
DURATION=60 ./scripts/load-test.sh
```

Runs for 1 minute instead of 5.

### Run Multiple Load Tests Simultaneously
```bash
# Terminal 1
./scripts/load-test.sh &

# Terminal 2
./scripts/load-test.sh &
```

**What to expect**: 2× traffic rate (~20 req/s)

### Custom Traffic Distribution

Edit `scripts/load-test.sh` and change the probabilities:

```bash
# 80% users requests (change from 30%)
if [ $((RANDOM % 10)) -lt 8 ]; then
    curl -s "$API_URL/api/users" > /dev/null &
fi
```

---

## Step 10A: Explore Prometheus Storage

While load testing is running, explore how Prometheus stores data.

### Check Storage Size

```bash
# Check total storage size
docker exec prometheus du -sh /prometheus
```

**What to expect**: 5-20 MB depending on how long load test has been running

### List Storage Contents

```bash
# List storage directory
docker exec prometheus ls -lh /prometheus
```

**What to see**:
```
drwxr-xr-x    2 nobody   nobody      64 Oct 25 18:30 chunks_head
drwxr-xr-x    3 nobody   nobody      96 Oct 25 18:30 01HJBC6XYZ123
drwxr-xr-x    2 nobody   nobody     128 Oct 25 18:30 wal
-rw-r--r--    1 nobody   nobody       0 Oct 25 18:25 lock
```

- **chunks_head**: Current in-memory data being written
- **01HJBC...**: Time-based blocks (2-hour windows)
- **wal**: Write-Ahead Log for durability
- **lock**: Lock file to prevent concurrent access

### View TSDB Statistics

**In Prometheus UI**:
1. Navigate to **Status** → **TSDB Status**
2. Observe:
   - Number of series
   - Number of chunks
   - Data size on disk
   - Head series count

**Expected values** (after 5 minutes of load testing):
- Series: 50-100
- Chunks: 200-500
- Size: 5-15 MB

### Check via API

```bash
# Get detailed TSDB stats
curl -s http://localhost:9090/api/v1/status/tsdb | python3 -m json.tool
```

### Examine a Time Block

```bash
# Find a block
BLOCK=$(docker exec prometheus ls -d /prometheus/01* | head -1 | xargs basename)

# View block contents
docker exec prometheus ls -lh /prometheus/$BLOCK/

# View block metadata
docker exec prometheus cat /prometheus/$BLOCK/meta.json | python3 -m json.tool
```

**What to see in metadata**:
- `minTime` / `maxTime`: Time range of block
- `stats.numSamples`: Number of data points
- `stats.numSeries`: Number of unique time series
- `compaction.level`: How many times block has been compacted

### Monitor Storage Growth

```bash
# Check size every 30 seconds
watch -n 30 'docker exec prometheus du -sh /prometheus'
```

**What to observe**: Storage grows as metrics accumulate

### Check Volume on Host

```bash
# Find volume location
docker volume inspect prometheus-poc_prometheus-data | grep Mountpoint

# Check size from host (may need sudo)
sudo du -sh /var/lib/docker/volumes/prometheus-poc_prometheus-data/_data
```

### Understanding Retention

Prometheus keeps data for **15 days by default**.

**Calculate current retention**:
```bash
# Check oldest block
docker exec prometheus sh -c 'ls -lt /prometheus/01* | tail -1'

# Check newest block
docker exec prometheus sh -c 'ls -lt /prometheus/01* | head -1'
```

After 15 days, oldest blocks are automatically deleted.

**For more storage operations**, see [storage-guide.md](storage-guide.md).

---

## Step 11: Troubleshooting

### Issue: Prometheus Shows Target as DOWN

**Check**:
```bash
# Verify OTEL Collector is accessible from Prometheus container
docker exec prometheus wget -qO- http://otel-collector:8889/metrics
```

**Solution**: Ensure all containers are on the same network and OTEL Collector is running

### Issue: OTEL Collector Not Receiving Metrics

**Check**:
```bash
# View OTEL Collector logs
docker-compose logs otel-collector

# Verify API can reach collector
docker exec go-api ping otel-collector
```

**Solution**:
- Ensure OTEL Collector is running (`docker-compose ps`)
- Check collector config is valid
- Verify API is pushing to correct endpoint (`otel-collector:4318`)

### Issue: No Metrics Appearing in Prometheus

**Check**:
```bash
# Check OTEL Collector metrics endpoint
curl http://localhost:8889/metrics | grep http_requests_total
```

**Solution**:
1. Generate traffic to the API first
2. Wait for OTEL SDK export interval (10 seconds)
3. Verify OTEL Collector is processing metrics (check logs)
4. Verify Prometheus is scraping Collector

### Issue: Load Test Script Not Working

**Check**:
```bash
ls -la scripts/load-test.sh
```

**Solution**: Make executable with `chmod +x scripts/load-test.sh`

### Issue: Prometheus Queries Return No Data

**Check**: Time range in Prometheus UI

**Solution**: Adjust to "Last 5 minutes" or ensure load test has been running

---

## Step 12: Cleanup

### Stop Load Test
```bash
# Press Ctrl+C or kill the process
pkill -f load-test.sh
```

### Stop Services
```bash
docker-compose down
```

### Remove Volumes (Optional)
```bash
docker-compose down -v
```

This deletes all Prometheus data.

---

## Demo Script (15 Minutes)

### Minutes 0-2: Setup
```bash
docker-compose up -d
docker-compose ps  # Verify all 3 services are running
curl http://localhost:8080/health
```

### Minutes 2-4: Verify Metrics Pipeline
```bash
# Generate some traffic first
curl http://localhost:8080/api/users

# Wait a few seconds for OTLP export, then check collector
curl http://localhost:8889/metrics | grep http_requests

# Open Prometheus
open http://localhost:9090
```

### Minutes 4-6: Start Load Test
```bash
./scripts/load-test.sh &
```

### Minutes 6-10: Show Key Queries

1. **Request rate**: `sum(rate(http_requests_total[1m]))`
2. **Error rate**: `sum(rate(http_requests_total{status=~"5.."}[1m]))`
3. **P95 latency**: `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[1m])))`
4. **Success rate**: `sum(rate(http_requests_total{status=~"2.."}[1m])) / sum(rate(http_requests_total[1m])) * 100`

### Minutes 10-13: Business Metrics

1. **Orders/min**: `rate(orders_total[1m]) * 60`
2. **Revenue/min**: `rate(orders_value_total[1m]) * 60`
3. **AOV**: `rate(orders_value_total[1m]) / rate(orders_total[1m])`

### Minutes 13-15: Q&A and Cleanup

---

## Next Steps

After completing the demo:

1. Review [metrics-guide.md](metrics-guide.md) for more queries
2. Try creating custom queries
3. Experiment with different traffic patterns
4. Consider adding Grafana for visualization (see extensions)
5. Explore alerting rules in Prometheus

---

## Learning Exercises

### Exercise 1: Find the Slowest Endpoint
Use Prometheus to identify which endpoint has the highest P99 latency.

### Exercise 2: Calculate SLA Compliance
Determine what percentage of requests complete under 1 second.

### Exercise 3: Analyze Error Patterns
Investigate which endpoints have the highest error rates and why.

### Exercise 4: Monitor Cache Effectiveness
Track cache hit rate and determine if it's meeting performance goals.

### Exercise 5: Capacity Planning
Based on the metrics, estimate how much traffic the API could handle.
