# Prometheus Metrics & Queries Guide

This guide provides comprehensive Prometheus (PromQL) queries for analyzing the metrics exposed by the Go API.

---

## Available Metrics

### HTTP Metrics (Auto-instrumented)

| Metric Name | Type | Labels | Description |
|-------------|------|--------|-------------|
| `http_requests_total` | Counter | `method`, `endpoint`, `status` | Total number of HTTP requests |
| `http_request_duration_seconds` | Histogram | `method`, `endpoint` | Request latency distribution with buckets |
| `http_request_duration_seconds_sum` | Counter | `method`, `endpoint` | Sum of all request durations |
| `http_request_duration_seconds_count` | Counter | `method`, `endpoint` | Count of requests (for average calculation) |
| `http_requests_in_flight` | Gauge | `method` | Number of currently in-flight requests |

### Business Metrics (Custom)

| Metric Name | Type | Labels | Description |
|-------------|------|--------|-------------|
| `orders_total` | Counter | `status` | Total number of orders |
| `orders_value_total` | Counter | - | Sum of all order values |
| `cache_hits_total` | Counter | `result` (hit/miss) | Cache hit/miss count |

---

## Request Rate Queries

### Total Request Rate
```promql
# Requests per second (instant rate over 1 minute)
rate(http_requests_total[1m])

# Requests per second (5 minute average)
rate(http_requests_total[5m])
```

**Use case**: Monitor overall API traffic

### Request Rate by Endpoint
```promql
# Rate per endpoint
sum by (endpoint) (rate(http_requests_total[1m]))

# Rate for specific endpoint
rate(http_requests_total{endpoint="/api/users"}[1m])
```

**Use case**: Identify most-used endpoints

### Request Rate by Status Code
```promql
# Rate by status code
sum by (status) (rate(http_requests_total[1m]))

# 2xx responses
sum(rate(http_requests_total{status=~"2.."}[1m]))

# 4xx responses
sum(rate(http_requests_total{status=~"4.."}[1m]))

# 5xx responses
sum(rate(http_requests_total{status=~"5.."}[1m]))
```

**Use case**: Monitor success vs error response rates

### Request Rate by Method
```promql
# Rate by HTTP method
sum by (method) (rate(http_requests_total[1m]))

# GET requests only
sum(rate(http_requests_total{method="GET"}[1m]))
```

---

## Error Rate Queries

### Error Count and Rate
```promql
# Server errors (5xx) per second
sum(rate(http_requests_total{status=~"5.."}[1m]))

# Client errors (4xx) per second
sum(rate(http_requests_total{status=~"4.."}[1m]))

# All errors (4xx + 5xx)
sum(rate(http_requests_total{status=~"[45].."}[1m]))
```

**Use case**: Monitor error occurrences

### Error Percentage
```promql
# Percentage of 5xx errors
sum(rate(http_requests_total{status=~"5.."}[1m])) /
sum(rate(http_requests_total[1m])) * 100

# Percentage of all errors (4xx + 5xx)
sum(rate(http_requests_total{status=~"[45].."}[1m])) /
sum(rate(http_requests_total[1m])) * 100

# Success rate (2xx and 3xx)
sum(rate(http_requests_total{status=~"[23].."}[1m])) /
sum(rate(http_requests_total[1m])) * 100
```

**Use case**: Calculate SLA compliance, error budgets

### Error Rate by Endpoint
```promql
# 5xx errors per endpoint
sum by (endpoint) (rate(http_requests_total{status=~"5.."}[1m]))

# Error rate for specific endpoint
rate(http_requests_total{endpoint="/api/orders",status=~"5.."}[1m])
```

**Use case**: Identify problematic endpoints

---

## Latency Queries

### Percentile Latencies (P50, P95, P99)
```promql
# P50 (median) latency
histogram_quantile(0.50,
  sum by (le) (rate(http_request_duration_seconds_bucket[1m])))

# P95 latency
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket[1m])))

# P99 latency
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[1m])))

# P99.9 latency
histogram_quantile(0.999,
  sum by (le) (rate(http_request_duration_seconds_bucket[1m])))
```

**Use case**: SLO tracking, performance monitoring

### Latency by Endpoint
```promql
# P95 latency per endpoint
histogram_quantile(0.95,
  sum by (endpoint, le) (rate(http_request_duration_seconds_bucket[1m])))

# P95 for specific endpoint
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket{endpoint="/api/slow"}[1m])))
```

**Use case**: Identify slow endpoints

### Average Latency
```promql
# Overall average latency
sum(rate(http_request_duration_seconds_sum[1m])) /
sum(rate(http_request_duration_seconds_count[1m]))

# Average latency by endpoint
sum by (endpoint) (rate(http_request_duration_seconds_sum[1m])) /
sum by (endpoint) (rate(http_request_duration_seconds_count[1m]))

# Average latency for specific endpoint
rate(http_request_duration_seconds_sum{endpoint="/api/users"}[1m]) /
rate(http_request_duration_seconds_count{endpoint="/api/users"}[1m])
```

**Use case**: General performance trends

### Latency Distribution
```promql
# Requests under 100ms
sum(rate(http_request_duration_seconds_bucket{le="0.1"}[1m]))

# Requests under 500ms
sum(rate(http_request_duration_seconds_bucket{le="0.5"}[1m]))

# Requests over 1 second
sum(rate(http_request_duration_seconds_bucket{le="+Inf"}[1m])) -
sum(rate(http_request_duration_seconds_bucket{le="1"}[1m]))
```

**Use case**: Analyze latency distribution

---

## Throughput Queries

### Total Requests
```promql
# Total requests in last 1 minute
sum(increase(http_requests_total[1m]))

# Total requests in last 5 minutes
sum(increase(http_requests_total[5m]))

# Total requests in last hour
sum(increase(http_requests_total[1h]))
```

**Use case**: Capacity planning, billing

### Successful Requests
```promql
# Successful requests (2xx) per minute
sum(increase(http_requests_total{status=~"2.."}[1m]))

# Success count by endpoint
sum by (endpoint) (increase(http_requests_total{status=~"2.."}[1m]))
```

**Use case**: Track successful operations

### Throughput by Endpoint
```promql
# Requests per minute by endpoint
sum by (endpoint) (increase(http_requests_total[1m]))

# Percentage distribution across endpoints
sum by (endpoint) (rate(http_requests_total[1m])) /
scalar(sum(rate(http_requests_total[1m]))) * 100
```

**Use case**: Understand traffic distribution

---

## Concurrent Requests (In-Flight)

### Current In-Flight Requests
```promql
# Current in-flight requests
http_requests_in_flight

# In-flight requests by method
sum by (method) (http_requests_in_flight)
```

**Use case**: Real-time concurrency monitoring

### In-Flight Statistics
```promql
# Maximum in-flight requests over 5 minutes
max_over_time(http_requests_in_flight[5m])

# Average in-flight requests over 5 minutes
avg_over_time(http_requests_in_flight[5m])

# Minimum in-flight requests (should be 0 or close to 0)
min_over_time(http_requests_in_flight[5m])
```

**Use case**: Capacity planning, load understanding

---

## Business Metrics Queries

### Order Metrics
```promql
# Orders created per minute
rate(orders_total[1m]) * 60

# Total orders in last hour
sum(increase(orders_total[1h]))

# Successful orders per second
rate(orders_total{status="success"}[1m])

# Failed orders per second
rate(orders_total{status="error"}[1m])

# Order success rate
rate(orders_total{status="success"}[1m]) /
rate(orders_total[1m]) * 100
```

**Use case**: Business KPI tracking

### Order Value Metrics
```promql
# Total order value per minute
rate(orders_value_total[1m]) * 60

# Total revenue in last hour
sum(increase(orders_value_total[1h]))

# Average order value
rate(orders_value_total[1m]) / rate(orders_total[1m])
```

**Use case**: Revenue tracking, AOV analysis

### Cache Metrics
```promql
# Cache hits per second
rate(cache_hits_total{result="hit"}[1m])

# Cache misses per second
rate(cache_hits_total{result="miss"}[1m])

# Cache hit rate (percentage)
rate(cache_hits_total{result="hit"}[1m]) /
rate(cache_hits_total[1m]) * 100

# Total cache operations
sum(rate(cache_hits_total[1m]))
```

**Use case**: Cache performance optimization

---

## Advanced Queries

### SLO Tracking
```promql
# Availability (uptime percentage)
sum(rate(http_requests_total{status=~"[23].."}[5m])) /
sum(rate(http_requests_total[5m])) * 100

# Latency SLO: % of requests under 200ms
sum(rate(http_request_duration_seconds_bucket{le="0.2"}[5m])) /
sum(rate(http_request_duration_seconds_count[5m])) * 100

# Combined SLI: fast AND successful
sum(rate(http_request_duration_seconds_bucket{le="0.2"}[5m]) *
    on() group_left rate(http_requests_total{status=~"2.."}[5m])) /
sum(rate(http_requests_total[5m])) * 100
```

**Use case**: Service Level Objectives monitoring

### Anomaly Detection
```promql
# Requests higher than 2x average
rate(http_requests_total[1m]) >
  2 * avg_over_time(rate(http_requests_total[1m])[1h:1m])

# Error rate higher than normal
sum(rate(http_requests_total{status=~"5.."}[1m])) >
  1.5 * avg_over_time(sum(rate(http_requests_total{status=~"5.."}[1m]))[1h:1m])

# Latency spike detection
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[1m]))) >
  1.5 * histogram_quantile(0.95, sum by (le) (avg_over_time(rate(http_request_duration_seconds_bucket[1m])[1h:1m])))
```

**Use case**: Alerting, incident detection

### Comparison Queries
```promql
# Compare current hour to previous hour
sum(rate(http_requests_total[1h])) /
sum(rate(http_requests_total[1h] offset 1h))

# Week-over-week comparison
sum(increase(http_requests_total[1d])) /
sum(increase(http_requests_total[1d] offset 1w))
```

**Use case**: Trend analysis, capacity planning

### Top-N Queries
```promql
# Top 5 endpoints by request count
topk(5, sum by (endpoint) (rate(http_requests_total[5m])))

# Bottom 5 endpoints by request count
bottomk(5, sum by (endpoint) (rate(http_requests_total[5m])))

# Top 3 slowest endpoints
topk(3, sum by (endpoint) (rate(http_request_duration_seconds_sum[5m])) /
           sum by (endpoint) (rate(http_request_duration_seconds_count[5m])))
```

**Use case**: Prioritization, optimization targeting

---

## Alerting Query Examples

### High Error Rate Alert
```promql
# Alert if 5xx error rate > 5% for 5 minutes
sum(rate(http_requests_total{status=~"5.."}[1m])) /
sum(rate(http_requests_total[1m])) > 0.05
```

### High Latency Alert
```promql
# Alert if P95 latency > 1 second for 5 minutes
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))) > 1
```

### Low Availability Alert
```promql
# Alert if success rate < 99% for 10 minutes
sum(rate(http_requests_total{status=~"2.."}[10m])) /
sum(rate(http_requests_total[10m])) < 0.99
```

### Traffic Spike Alert
```promql
# Alert if traffic doubles in 5 minutes
sum(rate(http_requests_total[5m])) >
  2 * avg_over_time(sum(rate(http_requests_total[5m]))[1h:5m])
```

---

## Query Tips

### Time Ranges
- `[1m]` - 1 minute window
- `[5m]` - 5 minute window
- `[1h]` - 1 hour window
- `[1d]` - 1 day window

### Functions
- `rate()` - Per-second rate over time range
- `increase()` - Total increase over time range
- `sum()` - Sum values
- `avg()` - Average values
- `max()` - Maximum value
- `min()` - Minimum value
- `histogram_quantile()` - Calculate percentiles
- `topk(N, ...)` - Top N values
- `bottomk(N, ...)` - Bottom N values

### Aggregations
- `sum by (label)` - Sum grouped by label
- `avg by (label)` - Average grouped by label
- `max by (label)` - Max grouped by label
- `count by (label)` - Count grouped by label

### Label Matching
- `{label="value"}` - Exact match
- `{label=~"regex"}` - Regex match
- `{label!="value"}` - Not equal
- `{label!~"regex"}` - Negative regex match

---

## Useful Dashboards Queries

### Overview Dashboard
```promql
# Request rate
sum(rate(http_requests_total[5m]))

# Error rate
sum(rate(http_requests_total{status=~"5.."}[5m]))

# P95 latency
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# Success rate
sum(rate(http_requests_total{status=~"2.."}[5m])) / sum(rate(http_requests_total[5m])) * 100
```

### Endpoint Performance Dashboard
```promql
# Request rate per endpoint
sum by (endpoint) (rate(http_requests_total[5m]))

# P95 latency per endpoint
histogram_quantile(0.95, sum by (endpoint, le) (rate(http_request_duration_seconds_bucket[5m])))

# Error rate per endpoint
sum by (endpoint) (rate(http_requests_total{status=~"5.."}[5m]))
```

### Business Metrics Dashboard
```promql
# Orders per hour
sum(increase(orders_total[1h]))

# Revenue per hour
sum(increase(orders_value_total[1h]))

# Average order value
rate(orders_value_total[5m]) / rate(orders_total[5m])

# Cache hit rate
rate(cache_hits_total{result="hit"}[5m]) / rate(cache_hits_total[5m]) * 100
```

---

## Further Reading

- [Prometheus Query Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [PromQL for Humans](https://timber.io/blog/promql-for-humans/)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/naming/)
- [Histogram vs Summary](https://prometheus.io/docs/practices/histograms/)
