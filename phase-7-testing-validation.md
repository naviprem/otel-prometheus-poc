# Phase 7: Testing & Validation

## Overview
Systematically test and validate all components of the OTEL Prometheus PoC to ensure proper functionality and demo-readiness.

---

## Step 1: Pre-Test Setup

### 1.1 Ensure Clean State
```bash
# Stop and clean existing containers
docker-compose down -v

# Remove old images (optional)
docker rmi otel-api-service:1.0.0 || true
docker rmi otel-dashboard:1.0.0 || true

# Clean docker system
docker system prune -f
```

### 1.2 Rebuild All Images
```bash
# Build all services
docker-compose build --no-cache

# Verify images
docker images | grep otel
```

---

## Step 2: Component Testing

### 2.1 Test Data Plane API Services

**Start API Services:**
```bash
docker-compose up -d api-prod api-dev api-qa
```

**Test Health Endpoints:**
```bash
# Prod
curl -s http://localhost:8081/health | jq .

# Dev
curl -s http://localhost:8082/health | jq .

# QA
curl -s http://localhost:8083/health | jq .
```

Expected output:
```json
{
  "success": true,
  "message": "Service is healthy",
  "data": {
    "status": "UP"
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**Test All Endpoints:**
```bash
# Products
curl -s http://localhost:8081/api/products | jq .

# Orders
curl -s -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -d '{"product_id":"1","quantity":2}' | jq .

# Reports
curl -s http://localhost:8081/api/reports | jq .

# Search
curl -s "http://localhost:8081/api/search?q=laptop" | jq .

# Payments
curl -s -X POST http://localhost:8081/api/payments \
  -H "Content-Type: application/json" \
  -d '{"order_id":"123","amount":199.99}' | jq .
```

**Validation Checklist:**
- [ ] All endpoints return 200 OK
- [ ] Response format is valid JSON
- [ ] Response includes timestamp
- [ ] POST endpoints accept and process payloads

### 2.2 Test OTEL Collectors

**Start Collectors:**
```bash
docker-compose up -d otel-collector-prod otel-collector-dev otel-collector-qa
```

**Verify Collectors Are Running:**
```bash
# Check logs
docker logs otel-collector-prod | tail -20

# Expected: "Everything is ready. Begin running and processing data."
```

**Test OTLP Receivers:**
```bash
# Check OTLP HTTP endpoint
curl -s http://localhost:4318/

# Check health (via metrics endpoint)
curl -s http://localhost:8889/metrics | head -20
```

**Generate Traffic and Check Metrics:**
```bash
# Generate some requests
for i in {1..20}; do
  curl -s http://localhost:8081/api/products > /dev/null
  sleep 0.5
done

# Check Prometheus exporter
curl -s http://localhost:8889/metrics | grep otel_http_server_duration

# Expected: histogram metrics with environment labels
```

**Validation Checklist:**
- [ ] Collectors start without errors
- [ ] OTLP receivers accept connections (4317, 4318)
- [ ] Prometheus exporter serves metrics (8889)
- [ ] Metrics include environment labels
- [ ] Collector self-metrics available (8888)

### 2.3 Test Prometheus

**Start Prometheus:**
```bash
docker-compose up -d prometheus
```

**Verify Prometheus UI:**
```bash
# Open in browser
open http://localhost:9090

# Or test via curl
curl -s http://localhost:9090/-/healthy
# Expected: Prometheus is Healthy
```

**Check Targets:**
```bash
# Via UI
open http://localhost:9090/targets

# Via API
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

Expected output:
```json
{
  "job": "dataplane-prod",
  "health": "up"
}
{
  "job": "dataplane-dev",
  "health": "up"
}
{
  "job": "dataplane-qa",
  "health": "up"
}
```

**Test PromQL Queries:**
```bash
# Query request rate
curl -s 'http://localhost:9090/api/v1/query?query=rate(otel_http_server_duration_count[1m])' | jq .

# Query by environment
curl -s 'http://localhost:9090/api/v1/query?query=rate(otel_http_server_duration_count{environment="prod"}[1m])' | jq .

# Query latency percentiles
curl -s 'http://localhost:9090/api/v1/query?query=histogram_quantile(0.95,rate(otel_http_server_duration_bucket[5m]))' | jq .
```

**Validation Checklist:**
- [ ] Prometheus UI accessible
- [ ] All scrape targets UP
- [ ] Metrics visible in Prometheus
- [ ] PromQL queries return data
- [ ] Environment labels present

### 2.4 Test Dashboard

**Start Dashboard:**
```bash
docker-compose up -d dashboard
```

**Verify Dashboard:**
```bash
# Check container
docker ps | grep dashboard

# Check logs
docker logs dashboard

# Access UI
open http://localhost:3000
```

**Manual UI Testing:**
1. Navigate to landing page
2. Check data plane cards are visible
3. Click "View Metrics" for each environment
4. Verify charts load with data
5. Test environment selector
6. Test time range selector
7. Test auto-refresh toggle

**API Integration Test:**
```bash
# Test Prometheus proxy (if configured)
curl -s http://localhost:3000/api/v1/query?query=up | jq .
```

**Validation Checklist:**
- [ ] Landing page loads
- [ ] Data planes detected
- [ ] Dashboard displays metrics
- [ ] Charts render correctly
- [ ] Environment filtering works
- [ ] Time range selection works
- [ ] Auto-refresh updates charts

---

## Step 3: Integration Testing

### 3.1 End-to-End Metric Flow

**Test Full Pipeline:**
```bash
# 1. Generate request to API
curl http://localhost:8081/api/products

# 2. Verify metric in OTEL Collector
sleep 2
curl -s http://localhost:8889/metrics | grep "otel_http_server_duration_count.*products"

# 3. Wait for Prometheus scrape (10s interval)
sleep 15

# 4. Query from Prometheus
curl -s 'http://localhost:9090/api/v1/query?query=otel_http_server_duration_count{http_route="/api/products"}' | jq .

# 5. Verify in Dashboard
open http://localhost:3000/dashboard
```

**Validation Checklist:**
- [ ] Request reaches API successfully
- [ ] Metric appears in OTEL Collector within 2 seconds
- [ ] Prometheus scrapes metric within 15 seconds
- [ ] Dashboard displays metric within 30 seconds
- [ ] Metric includes correct labels (environment, endpoint)

### 3.2 Multi-Environment Testing

**Generate Traffic to All Environments:**
```bash
# Parallel requests
curl http://localhost:8081/api/products & # Prod
curl http://localhost:8082/api/products & # Dev
curl http://localhost:8083/api/products & # QA
wait

# Wait for metrics to propagate
sleep 20

# Query aggregated metrics
curl -s 'http://localhost:9090/api/v1/query?query=sum by (environment) (rate(otel_http_server_duration_count[1m]))' | jq .
```

Expected: Separate series for prod, dev, qa

**Validation Checklist:**
- [ ] Metrics from all environments visible
- [ ] Can filter by environment in Prometheus
- [ ] Dashboard environment selector shows all
- [ ] Metrics properly labeled by environment

### 3.3 Load Testing Integration

**Run Load Generator:**
```bash
cd load-generator
python load_generator.py -t http://localhost:8081 -e prod -r 50 -d 120
```

**Monitor During Load:**
```bash
# Watch Prometheus query
watch -n 5 'curl -s "http://localhost:9090/api/v1/query?query=rate(otel_http_server_duration_count[1m])" | jq ".data.result[0].value[1]"'

# Check dashboard updates
# Refresh dashboard and observe real-time charts
```

**Validation Checklist:**
- [ ] High request rate visible in metrics
- [ ] Dashboard charts update during load
- [ ] No errors in collector logs
- [ ] Prometheus scrapes don't timeout
- [ ] System remains stable under load

---

## Step 4: Performance Testing

### 4.1 Query Performance

**Test Query Latency:**
```bash
#!/bin/bash
# query_perf_test.sh

queries=(
  "rate(otel_http_server_duration_count[5m])"
  "histogram_quantile(0.95,rate(otel_http_server_duration_bucket[5m]))"
  "sum by (environment) (rate(otel_http_server_duration_count[1m]))"
)

for query in "${queries[@]}"; do
  echo "Testing: $query"
  time curl -s "http://localhost:9090/api/v1/query?query=$(echo $query | jq -sRr @uri)" > /dev/null
  echo ""
done
```

**Expected Results:**
- All queries should complete in <2 seconds
- Complex aggregations in <5 seconds

**Validation Checklist:**
- [ ] Instant queries < 2s
- [ ] Range queries < 5s
- [ ] Dashboard loads < 3s
- [ ] No timeout errors

### 4.2 Resource Usage

**Monitor Resource Consumption:**
```bash
# Container stats
docker stats --no-stream

# Specific containers
docker stats --no-stream prometheus otel-collector-prod api-prod
```

**Expected Resource Usage:**
```
CONTAINER            CPU %     MEM USAGE / LIMIT     MEM %
api-prod             5%        50MB / 512MB          10%
otel-collector-prod  3%        100MB / 512MB         20%
prometheus           10%       500MB / 2GB           25%
dashboard            1%        50MB / 256MB          20%
```

**Validation Checklist:**
- [ ] No container using >80% memory
- [ ] CPU usage reasonable (<50% under load)
- [ ] No memory leaks (stable over time)
- [ ] Disk usage acceptable

### 4.3 Data Retention

**Test TSDB Storage:**
```bash
# Check Prometheus data directory size
docker exec prometheus du -sh /prometheus

# Check number of samples
curl -s http://localhost:9090/api/v1/status/tsdb | jq .

# Generate sustained load for retention test
cd load-generator
python load_generator.py -t http://localhost:8081 -r 10 -d 3600 &
```

**Validation Checklist:**
- [ ] Data persists across restarts
- [ ] Old data cleaned per retention policy
- [ ] TSDB size grows predictably
- [ ] No "out of disk space" errors

---

## Step 5: Error Handling & Resilience

### 5.1 Component Failure Testing

**Test API Failure:**
```bash
# Stop one API
docker-compose stop api-prod

# Check Prometheus target status
open http://localhost:9090/targets
# Expected: dataplane-prod should be DOWN

# Check dashboard
# Expected: Prod metrics stop updating, dev/qa continue

# Restart API
docker-compose start api-prod

# Verify recovery
sleep 30
curl http://localhost:8081/health
```

**Test OTEL Collector Failure:**
```bash
# Stop collector
docker-compose stop otel-collector-prod

# Generate API requests
curl http://localhost:8081/api/products

# Check API logs for errors
docker logs api-prod --tail 20
# Expected: Connection errors to OTEL endpoint

# Restart collector
docker-compose start otel-collector-prod

# Verify metrics resume
sleep 20
curl http://localhost:8889/metrics | grep otel_http_server_duration
```

**Test Prometheus Failure:**
```bash
# Stop Prometheus
docker-compose stop prometheus

# Dashboard should show connection errors
open http://localhost:3000

# Restart Prometheus
docker-compose start prometheus

# Verify dashboard recovers
```

**Validation Checklist:**
- [ ] Partial failures don't crash system
- [ ] Components reconnect automatically
- [ ] Data loss is minimal (<1 minute)
- [ ] Error messages are clear

### 5.2 Network Partition Testing

**Simulate Network Issues:**
```bash
# Create network latency
docker exec api-prod tc qdisc add dev eth0 root netem delay 100ms

# Test with latency
curl -w "@curl-format.txt" http://localhost:8081/api/products

# Remove latency
docker exec api-prod tc qdisc del dev eth0 root netem
```

**Validation Checklist:**
- [ ] System handles network delays
- [ ] Timeouts configured appropriately
- [ ] Retries work correctly

---

## Step 6: Demo Scenario Validation

### 6.1 Scenario 1: Normal Operations
```bash
# Run steady load pattern
cd load-generator
python load_patterns.py steady

# Expected Results:
# - All environments showing metrics
# - Request rates: prod ~50/s, dev ~20/s, qa ~15/s
# - Low error rates (<1%)
# - Dashboard updates smoothly
```

**Validation:**
- [ ] All data planes reporting
- [ ] Metrics match expected rates
- [ ] Charts show clear trends
- [ ] No errors in logs

### 6.2 Scenario 2: Production Incident
```bash
# Run spike pattern on prod
python load_patterns.py spike prod

# Expected Results:
# - Prod shows sudden spike to 200 req/s
# - Latency increases under load
# - Error rate may increase slightly
# - Dashboard clearly shows the spike
```

**Validation:**
- [ ] Spike visible in dashboard
- [ ] Can identify affected environment
- [ ] Latency percentiles show impact
- [ ] System recovers after spike

### 6.3 Scenario 3: Environment Comparison
```bash
# Generate different loads
python load_generator.py -t http://localhost:8081 -r 100 -d 300 &  # Heavy load prod
python load_generator.py -t http://localhost:8082 -r 10 -d 300 &   # Light load dev

# In dashboard:
# - Select "All Environments"
# - Compare metrics side-by-side
```

**Validation:**
- [ ] Can compare environments visually
- [ ] Metrics properly separated by labels
- [ ] Dashboard handles multiple series
- [ ] Legend clearly identifies environments

---

## Step 7: Documentation Validation

### 7.1 Verify README Accuracy
```bash
# Follow steps in main README.md
# Verify all commands work
# Check for outdated information
```

### 7.2 Verify Phase Guides
```bash
# Test steps from each phase guide
# Ensure code examples are correct
# Verify configuration files match
```

---

## Step 8: Create Test Report

### 8.1 Create `test-report.md`
```markdown
# OTEL Prometheus PoC - Test Report

**Date:** [Date]
**Tester:** [Name]
**Version:** 1.0.0

## Summary
- **Total Tests:** 50
- **Passed:** 48
- **Failed:** 2
- **Blocked:** 0

## Component Tests

### Data Plane API
- ✅ Health endpoints
- ✅ All API endpoints
- ✅ Error handling
- ✅ OTEL instrumentation

### OTEL Collector
- ✅ Receives OTLP metrics
- ✅ Exports Prometheus metrics
- ✅ Resource labeling
- ⚠️  High memory under extreme load (>2GB)

### Prometheus
- ✅ Scrapes all targets
- ✅ PromQL queries
- ✅ Data retention
- ✅ Performance

### Dashboard
- ✅ Landing page
- ✅ Metrics display
- ✅ Environment filtering
- ❌ Auto-refresh occasionally stalls (bug filed)

## Integration Tests
- ✅ End-to-end metric flow
- ✅ Multi-environment
- ✅ Load testing

## Performance
- ✅ Query latency <2s
- ✅ Resource usage acceptable
- ✅ Handles 100 req/sec

## Issues Found
1. Dashboard auto-refresh stops after 30 min (needs investigation)
2. OTEL Collector memory usage high under sustained load (tune config)

## Recommendations
- Increase OTEL Collector memory limit to 1GB
- Fix dashboard auto-refresh bug
- Add alert for collector memory usage

## Demo Readiness
**Status:** ✅ READY

The PoC is ready for demonstration with minor known issues that don't impact core functionality.
```

---

## Expected Outcomes

✅ **All Components Working**:
- APIs serving requests correctly
- Collectors processing metrics
- Prometheus scraping successfully
- Dashboard displaying data

✅ **Integration Verified**:
- End-to-end metric flow confirmed
- Multi-environment setup validated
- Load testing successful

✅ **Performance Acceptable**:
- Query latency <2s
- Resource usage reasonable
- System stable under load

✅ **Demo-Ready**:
- All scenarios work
- Clear visualization
- No critical bugs

---

## Troubleshooting Common Issues

**Metrics not appearing:**
```bash
# Check full pipeline
docker logs api-prod | grep -i error
docker logs otel-collector-prod | grep -i error
curl http://localhost:8889/metrics | grep otel_http
curl 'http://localhost:9090/api/v1/query?query=up'
```

**Dashboard not loading:**
```bash
# Check nginx logs
docker logs dashboard

# Verify Prometheus connectivity
docker exec dashboard ping prometheus

# Check browser console for errors
```

**High memory usage:**
```bash
# Restart collectors with lower batch sizes
# Edit otel-collector config
# Reduce Prometheus retention
```

---

## Next Steps

Proceed to **Phase 8: Documentation & Presentation** to finalize documentation and prepare for handoff.
