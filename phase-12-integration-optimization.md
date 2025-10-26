# Phase 12: Integration and Optimization

## Overview
Integrate all analytics components, optimize the data pipeline, add advanced features, and prepare for production deployment. This phase brings together the S3/Iceberg storage, ClickHouse analytics, and dashboard into a cohesive system.

---

## Step 1: Complete Data Pipeline Integration

### 1.1 Update OTEL Collector with Dual Pipeline

**Create `dataplane/otel-collector/config-production.yaml`:**
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  # Batch for efficiency
  batch:
    timeout: 10s
    send_batch_size: 1024
    send_batch_max_size: 2048

  # Add resource attributes
  resource:
    attributes:
      - key: environment
        value: ${env:ENVIRONMENT}
        action: upsert
      - key: data_plane_id
        value: ${env:DATA_PLANE_ID}
        action: upsert
      - key: cluster_id
        value: ${env:CLUSTER_ID:default-cluster}
        action: upsert
      - key: region
        value: ${env:AWS_REGION:us-east-1}
        action: upsert

  # Transform for storage
  transform/storage:
    metric_statements:
      - context: datapoint
        statements:
          # Extract timestamp components for partitioning
          - set(attributes["year"], Year(start_time_unix_nano))
          - set(attributes["month"], Month(start_time_unix_nano))
          - set(attributes["day"], Day(start_time_unix_nano))
          - set(attributes["hour"], Hour(start_time_unix_nano))

  # Memory limiter
  memory_limiter:
    check_interval: 1s
    limit_mib: 1024
    spike_limit_mib: 256

  # Filter out internal metrics
  filter/drop_internal:
    metrics:
      exclude:
        match_type: regexp
        metric_names:
          - ^otelcol_.*

exporters:
  # Prometheus for real-time monitoring
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

  # File exporter for long-term storage
  file/metrics:
    path: /buffer/metrics.jsonl
    rotation:
      max_megabytes: 50
      max_days: 1
      max_backups: 5
    format: json
    flush_interval: 10s

  # Logging for debugging
  logging:
    loglevel: info
    sampling_initial: 5
    sampling_thereafter: 200

service:
  telemetry:
    logs:
      level: info
    metrics:
      address: 0.0.0.0:8888
      level: detailed

  pipelines:
    # Real-time pipeline (hot path)
    metrics/realtime:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]
      exporters: [prometheus]

    # Long-term storage pipeline (cold path)
    metrics/storage:
      receivers: [otlp]
      processors: [memory_limiter, resource, transform/storage, filter/drop_internal, batch]
      exporters: [file/metrics]

    # Debug pipeline (optional)
    metrics/debug:
      receivers: [otlp]
      processors: [memory_limiter, resource]
      exporters: [logging]
```

### 1.2 Create Enhanced S3 Uploader with Parquet

**Update `analytics/s3-uploader/parquet_writer.go`:**
```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
	"time"

	"github.com/xitongsys/parquet-go-source/local"
	"github.com/xitongsys/parquet-go/parquet"
	"github.com/xitongsys/parquet-go/writer"
)

// MetricParquet represents the Parquet schema
type MetricParquet struct {
	Timestamp    int64             `parquet:"name=timestamp, type=INT64, convertedtype=TIMESTAMP_MILLIS"`
	MetricName   string            `parquet:"name=metric_name, type=BYTE_ARRAY, convertedtype=UTF8"`
	MetricValue  float64           `parquet:"name=metric_value, type=DOUBLE"`
	MetricType   string            `parquet:"name=metric_type, type=BYTE_ARRAY, convertedtype=UTF8"`

	// Labels
	HttpMethod       string `parquet:"name=http_method, type=BYTE_ARRAY, convertedtype=UTF8"`
	HttpRoute        string `parquet:"name=http_route, type=BYTE_ARRAY, convertedtype=UTF8"`
	HttpStatusCode   string `parquet:"name=http_status_code, type=BYTE_ARRAY, convertedtype=UTF8"`

	// Resource attributes
	Environment  string `parquet:"name=environment, type=BYTE_ARRAY, convertedtype=UTF8"`
	DataPlaneID  string `parquet:"name=data_plane_id, type=BYTE_ARRAY, convertedtype=UTF8"`
	ServiceName  string `parquet:"name=service_name, type=BYTE_ARRAY, convertedtype=UTF8"`
	ClusterID    string `parquet:"name=cluster_id, type=BYTE_ARRAY, convertedtype=UTF8"`
	Region       string `parquet:"name=region, type=BYTE_ARRAY, convertedtype=UTF8"`

	// Partition columns
	Year  int32 `parquet:"name=year, type=INT32"`
	Month int32 `parquet:"name=month, type=INT32"`
	Day   int32 `parquet:"name=day, type=INT32"`
	Hour  int32 `parquet:"name=hour, type=INT32"`
}

func (u *S3Uploader) convertToParquet(metrics []MetricRecord) (string, error) {
	timestamp := time.Now().Unix()
	filename := fmt.Sprintf("/tmp/metrics_%d.parquet", timestamp)

	// Create Parquet file
	fw, err := local.NewLocalFileWriter(filename)
	if err != nil {
		return "", fmt.Errorf("failed to create parquet file: %w", err)
	}

	// Create Parquet writer
	pw, err := writer.NewParquetWriter(fw, new(MetricParquet), 4)
	if err != nil {
		fw.Close()
		return "", fmt.Errorf("failed to create parquet writer: %w", err)
	}

	// Set compression
	pw.CompressionType = parquet.CompressionCodec_SNAPPY

	// Write metrics
	for _, metric := range metrics {
		parquetMetric := MetricParquet{
			Timestamp:      metric.Timestamp.UnixMilli(),
			MetricName:     metric.MetricName,
			MetricValue:    metric.MetricValue,
			MetricType:     metric.MetricType,
			HttpMethod:     metric.Labels["http_method"],
			HttpRoute:      metric.Labels["http_route"],
			HttpStatusCode: metric.Labels["http_status_code"],
			Environment:    metric.Labels["environment"],
			DataPlaneID:    metric.DataPlaneID,
			ServiceName:    metric.Labels["service_name"],
			ClusterID:      metric.Labels["cluster_id"],
			Region:         metric.Labels["region"],
			Year:           int32(metric.Year),
			Month:          int32(metric.Month),
			Day:            int32(metric.Day),
			Hour:           int32(metric.Hour),
		}

		if err := pw.Write(parquetMetric); err != nil {
			pw.WriteStop()
			fw.Close()
			return "", fmt.Errorf("failed to write parquet record: %w", err)
		}
	}

	// Close writer
	if err := pw.WriteStop(); err != nil {
		fw.Close()
		return "", fmt.Errorf("failed to finalize parquet file: %w", err)
	}

	fw.Close()

	log.Printf("Created Parquet file: %s with %d records", filename, len(metrics))
	return filename, nil
}
```

---

## Step 2: Optimize ClickHouse Performance

### 2.1 Create Optimized Table Schema

**Create `analytics/clickhouse/optimized_schema.sql`:**
```sql
USE otel_metrics;

-- Drop existing tables if reoptimizing
-- DROP TABLE IF EXISTS metrics_local;
-- DROP TABLE IF EXISTS metrics_hourly;
-- DROP TABLE IF EXISTS metrics_daily;

-- Optimized local table with better compression
CREATE TABLE IF NOT EXISTS metrics_local_v2
(
    timestamp DateTime64(3),
    metric_name LowCardinality(String),
    metric_value Float64 CODEC(Gorilla, ZSTD(3)),
    metric_type LowCardinality(String),

    -- Common labels
    http_method LowCardinality(String),
    http_route String CODEC(ZSTD(3)),
    http_status_code LowCardinality(String),

    -- Resource attributes
    environment LowCardinality(String),
    data_plane_id LowCardinality(String),
    service_name LowCardinality(String),
    cluster_id LowCardinality(String),
    region LowCardinality(String),

    -- Partition columns
    date Date,
    hour UInt8,

    -- Indexes
    INDEX idx_metric_name metric_name TYPE set(100) GRANULARITY 4,
    INDEX idx_http_route http_route TYPE bloom_filter(0.01) GRANULARITY 4,
    INDEX idx_timestamp timestamp TYPE minmax GRANULARITY 1
)
ENGINE = MergeTree()
PARTITION BY (toYYYYMM(date), data_plane_id)
ORDER BY (environment, metric_name, date, hour, timestamp)
TTL date + INTERVAL 90 DAY
SETTINGS
    index_granularity = 8192,
    min_bytes_for_wide_part = 10485760,
    compress_primary_key = 1;

-- Projection for endpoint analysis
ALTER TABLE metrics_local_v2
ADD PROJECTION endpoint_analysis (
    SELECT
        date,
        environment,
        http_route,
        count() AS request_count,
        quantile(0.5)(metric_value) AS p50,
        quantile(0.95)(metric_value) AS p95,
        quantile(0.99)(metric_value) AS p99
    GROUP BY date, environment, http_route
);

-- Projection for time-series analysis
ALTER TABLE metrics_local_v2
ADD PROJECTION time_series (
    SELECT
        toStartOfHour(timestamp) AS hour,
        environment,
        metric_name,
        count() AS count,
        avg(metric_value) AS avg_value,
        quantile(0.95)(metric_value) AS p95
    GROUP BY hour, environment, metric_name
);

-- Optimize hourly aggregations with better partitioning
CREATE TABLE IF NOT EXISTS metrics_hourly_v2
(
    date Date,
    hour UInt8,
    environment LowCardinality(String),
    data_plane_id LowCardinality(String),
    metric_name LowCardinality(String),
    http_route String,
    http_status_code LowCardinality(String),

    -- Aggregated metrics
    request_count SimpleAggregateFunction(sum, UInt64),
    error_count SimpleAggregateFunction(sum, UInt64),
    total_duration SimpleAggregateFunction(sum, Float64),
    min_duration SimpleAggregateFunction(min, Float64),
    max_duration SimpleAggregateFunction(max, Float64),

    -- Quantiles stored as state
    duration_quantiles AggregateFunction(quantiles(0.5, 0.95, 0.99), Float64)
)
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, hour, environment, data_plane_id, metric_name, http_route)
TTL date + INTERVAL 90 DAY
SETTINGS index_granularity = 8192;

-- Materialized view with better aggregation
CREATE MATERIALIZED VIEW IF NOT EXISTS metrics_hourly_v2_mv TO metrics_hourly_v2
AS SELECT
    toDate(timestamp) AS date,
    toHour(timestamp) AS hour,
    environment,
    data_plane_id,
    metric_name,
    http_route,
    http_status_code,

    count() AS request_count,
    countIf(http_status_code >= '500') AS error_count,
    sum(metric_value) AS total_duration,
    min(metric_value) AS min_duration,
    max(metric_value) AS max_duration,
    quantilesState(0.5, 0.95, 0.99)(metric_value) AS duration_quantiles
FROM metrics_local_v2
WHERE metric_name = 'http_server_duration'
GROUP BY date, hour, environment, data_plane_id, metric_name, http_route, http_status_code;
```

### 2.2 Create Query Optimization Views

**Create `analytics/clickhouse/query_helpers.sql`:**
```sql
USE otel_metrics;

-- Pre-aggregated view for dashboard queries
CREATE VIEW IF NOT EXISTS v_dashboard_hourly AS
SELECT
    toDateTime(toStartOfHour(date) + INTERVAL hour HOUR) AS timestamp,
    environment,
    data_plane_id,
    metric_name,
    http_route,

    request_count,
    error_count,
    (error_count / request_count) * 100 AS error_rate,
    total_duration / request_count AS avg_duration,

    -- Extract quantiles from state
    quantilesMerge(0.5, 0.95, 0.99)(duration_quantiles)[1] AS p50_duration,
    quantilesMerge(0.5, 0.95, 0.99)(duration_quantiles)[2] AS p95_duration,
    quantilesMerge(0.5, 0.95, 0.99)(duration_quantiles)[3] AS p99_duration
FROM metrics_hourly_v2
GROUP BY timestamp, environment, data_plane_id, metric_name, http_route;

-- Fast daily summary
CREATE VIEW IF NOT EXISTS v_daily_summary AS
SELECT
    date,
    environment,
    data_plane_id,

    sum(request_count) AS total_requests,
    sum(error_count) AS total_errors,
    (sum(error_count) / sum(request_count)) * 100 AS error_rate,
    avg(total_duration / request_count) AS avg_duration,
    max(max_duration) AS max_duration
FROM metrics_hourly_v2
GROUP BY date, environment, data_plane_id;

-- Top endpoints by request count
CREATE VIEW IF NOT EXISTS v_top_endpoints AS
SELECT
    environment,
    http_route,
    sum(request_count) AS total_requests,
    avg(total_duration / request_count) AS avg_duration,
    quantilesMerge(0.95)(duration_quantiles)[1] AS p95_duration
FROM metrics_hourly_v2
WHERE date >= today() - INTERVAL 7 DAY
GROUP BY environment, http_route
ORDER BY total_requests DESC
LIMIT 20;
```

### 2.3 Setup Query Result Cache

**Create `analytics/clickhouse/cache_config.sql`:**
```sql
-- Enable query cache for common queries
SYSTEM DROP QUERY CACHE;

-- Configure query cache settings
SET query_cache_max_size_in_bytes = 5368709120; -- 5GB
SET query_cache_max_entries = 10000;
SET query_cache_ttl = 300; -- 5 minutes

-- Example cached query (ClickHouse will automatically cache)
SELECT
    date,
    environment,
    sum(total_requests) AS requests
FROM v_daily_summary
WHERE date >= today() - INTERVAL 30 DAY
GROUP BY date, environment
SETTINGS use_query_cache = 1;
```

---

## Step 3: Add Data Quality Monitoring

### 3.1 Create Data Quality Checks

**Create `analytics/data-quality/checks.sql`:**
```sql
USE otel_metrics;

-- Check for data gaps (missing hours)
CREATE VIEW IF NOT EXISTS v_data_quality_gaps AS
WITH expected_hours AS (
    SELECT
        toDateTime(toStartOfHour(now()) - INTERVAL number HOUR) AS expected_hour
    FROM numbers(168) -- Last 7 days
)
SELECT
    eh.expected_hour,
    count(DISTINCT m.data_plane_id) AS data_planes_reporting,
    arraySort(groupArray(DISTINCT m.data_plane_id)) AS reporting_planes
FROM expected_hours eh
LEFT JOIN metrics_hourly_v2 m
    ON toStartOfHour(m.date + INTERVAL m.hour HOUR) = eh.expected_hour
GROUP BY eh.expected_hour
HAVING data_planes_reporting < 3  -- Assuming we expect 3 data planes
ORDER BY eh.expected_hour DESC;

-- Check for duplicate records
CREATE VIEW IF NOT EXISTS v_data_quality_duplicates AS
SELECT
    date,
    hour,
    environment,
    data_plane_id,
    metric_name,
    http_route,
    count() AS duplicate_count
FROM metrics_hourly_v2
GROUP BY date, hour, environment, data_plane_id, metric_name, http_route
HAVING count() > 1;

-- Check for anomalous values
CREATE VIEW IF NOT EXISTS v_data_quality_anomalies AS
SELECT
    date,
    hour,
    environment,
    http_route,
    max_duration,
    avg(max_duration) OVER (
        PARTITION BY environment, http_route
        ORDER BY date, hour
        ROWS BETWEEN 24 PRECEDING AND 1 PRECEDING
    ) AS avg_24h,
    stddevPop(max_duration) OVER (
        PARTITION BY environment, http_route
        ORDER BY date, hour
        ROWS BETWEEN 24 PRECEDING AND 1 PRECEDING
    ) AS stddev_24h
FROM metrics_hourly_v2
WHERE date >= today() - INTERVAL 7 DAY
HAVING max_duration > avg_24h + (4 * stddev_24h);

-- Data freshness check
CREATE VIEW IF NOT EXISTS v_data_freshness AS
SELECT
    data_plane_id,
    environment,
    max(date + INTERVAL hour HOUR) AS last_data_timestamp,
    dateDiff('minute', max(date + INTERVAL hour HOUR), now()) AS minutes_since_last_data
FROM metrics_hourly_v2
GROUP BY data_plane_id, environment
HAVING minutes_since_last_data > 60; -- Alert if data older than 1 hour
```

### 3.2 Create Automated Quality Reports

**Create `analytics/data-quality/report.sh`:**
```bash
#!/bin/bash
# Data Quality Report Generator

CLICKHOUSE_HOST="${CLICKHOUSE_HOST:-localhost}"
CLICKHOUSE_PORT="${CLICKHOUSE_PORT:-8123}"

echo "==================================================================="
echo "Data Quality Report - $(date)"
echo "==================================================================="

echo ""
echo "1. Data Gaps (Missing Hours):"
echo "-------------------------------------------------------------------"
docker exec clickhouse clickhouse-client --query "
SELECT
    expected_hour,
    data_planes_reporting,
    reporting_planes
FROM otel_metrics.v_data_quality_gaps
LIMIT 10
FORMAT PrettyCompact
"

echo ""
echo "2. Duplicate Records:"
echo "-------------------------------------------------------------------"
docker exec clickhouse clickhouse-client --query "
SELECT count() AS total_duplicates
FROM otel_metrics.v_data_quality_duplicates
FORMAT PrettyCompact
"

echo ""
echo "3. Data Freshness:"
echo "-------------------------------------------------------------------"
docker exec clickhouse clickhouse-client --query "
SELECT
    data_plane_id,
    environment,
    last_data_timestamp,
    minutes_since_last_data
FROM otel_metrics.v_data_freshness
FORMAT PrettyCompact
"

echo ""
echo "4. Storage Statistics:"
echo "-------------------------------------------------------------------"
docker exec clickhouse clickhouse-client --query "
SELECT
    table,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    sum(rows) AS rows,
    max(modification_time) AS last_modified
FROM system.parts
WHERE database = 'otel_metrics' AND active
GROUP BY table
FORMAT PrettyCompact
"

echo ""
echo "==================================================================="
```

---

## Step 4: Advanced Analytics Features

### 4.1 Add Anomaly Detection

**Create `analytics/clickhouse/anomaly_detection.sql`:**
```sql
USE otel_metrics;

-- Anomaly detection using Z-score
CREATE VIEW IF NOT EXISTS v_latency_anomalies AS
WITH stats AS (
    SELECT
        environment,
        http_route,
        avg(total_duration / request_count) AS mean_latency,
        stddevPop(total_duration / request_count) AS stddev_latency
    FROM metrics_hourly_v2
    WHERE date >= today() - INTERVAL 30 DAY
      AND metric_name = 'http_server_duration'
    GROUP BY environment, http_route
),
recent_data AS (
    SELECT
        date,
        hour,
        environment,
        http_route,
        total_duration / request_count AS current_latency
    FROM metrics_hourly_v2
    WHERE date >= today() - INTERVAL 7 DAY
      AND metric_name = 'http_server_duration'
)
SELECT
    r.date,
    r.hour,
    r.environment,
    r.http_route,
    r.current_latency,
    s.mean_latency,
    s.stddev_latency,
    (r.current_latency - s.mean_latency) / s.stddev_latency AS z_score,
    multiIf(
        abs((r.current_latency - s.mean_latency) / s.stddev_latency) > 3, 'critical',
        abs((r.current_latency - s.mean_latency) / s.stddev_latency) > 2, 'warning',
        'normal'
    ) AS severity
FROM recent_data r
INNER JOIN stats s
    ON r.environment = s.environment
    AND r.http_route = s.http_route
WHERE s.stddev_latency > 0
  AND abs((r.current_latency - s.mean_latency) / s.stddev_latency) > 2
ORDER BY abs(z_score) DESC;
```

### 4.2 Add Forecasting (Simple Linear Regression)

**Create `analytics/clickhouse/forecasting.sql`:**
```sql
USE otel_metrics;

-- Simple trend forecasting using linear regression
CREATE VIEW IF NOT EXISTS v_request_forecast AS
WITH daily_totals AS (
    SELECT
        date,
        environment,
        sum(total_requests) AS daily_requests,
        toDayOfYear(date) AS day_num
    FROM v_daily_summary
    WHERE date >= today() - INTERVAL 90 DAY
    GROUP BY date, environment
),
regression AS (
    SELECT
        environment,
        simpleLinearRegression(day_num, daily_requests) AS lr_model
    FROM daily_totals
    GROUP BY environment
)
SELECT
    environment,
    lr_model.1 AS slope,
    lr_model.2 AS intercept,

    -- Forecast next 7 days
    arrayMap(
        x -> lr_model.1 * (toDayOfYear(today()) + x) + lr_model.2,
        range(1, 8)
    ) AS forecasted_requests,

    -- Forecast dates
    arrayMap(
        x -> toString(today() + INTERVAL x DAY),
        range(1, 8)
    ) AS forecast_dates
FROM regression;
```

### 4.3 Add SLO Tracking

**Create `analytics/clickhouse/slo_tracking.sql`:**
```sql
USE otel_metrics;

-- Define SLOs (Service Level Objectives)
CREATE TABLE IF NOT EXISTS slo_definitions
(
    environment String,
    metric_name String,
    slo_type Enum('latency_p95', 'latency_p99', 'error_rate', 'availability'),
    threshold Float64,
    description String
)
ENGINE = MergeTree()
ORDER BY (environment, metric_name, slo_type);

-- Insert sample SLOs
INSERT INTO slo_definitions VALUES
    ('prod', 'http_server_duration', 'latency_p95', 0.5, 'P95 latency < 500ms'),
    ('prod', 'http_server_duration', 'error_rate', 1.0, 'Error rate < 1%'),
    ('dev', 'http_server_duration', 'latency_p95', 1.0, 'P95 latency < 1s'),
    ('dev', 'http_server_duration', 'error_rate', 5.0, 'Error rate < 5%');

-- SLO compliance tracking
CREATE VIEW IF NOT EXISTS v_slo_compliance AS
WITH hourly_metrics AS (
    SELECT
        date,
        hour,
        environment,
        (error_count / request_count) * 100 AS error_rate,
        quantilesMerge(0.95)(duration_quantiles)[1] AS p95_latency
    FROM metrics_hourly_v2
    WHERE metric_name = 'http_server_duration'
      AND date >= today() - INTERVAL 7 DAY
    GROUP BY date, hour, environment
)
SELECT
    s.environment,
    s.metric_name,
    s.slo_type,
    s.threshold,
    s.description,

    countIf(
        (s.slo_type = 'error_rate' AND m.error_rate <= s.threshold) OR
        (s.slo_type = 'latency_p95' AND m.p95_latency <= s.threshold)
    ) AS compliant_hours,

    count() AS total_hours,

    (compliant_hours / total_hours) * 100 AS slo_compliance_percent,

    multiIf(
        (compliant_hours / total_hours) >= 0.999, '99.9%',
        (compliant_hours / total_hours) >= 0.99, '99%',
        (compliant_hours / total_hours) >= 0.95, '95%',
        'Below 95%'
    ) AS slo_tier
FROM slo_definitions s
LEFT JOIN hourly_metrics m
    ON s.environment = m.environment
GROUP BY s.environment, s.metric_name, s.slo_type, s.threshold, s.description;
```

---

## Step 5: Create Comprehensive Monitoring

### 5.1 Add System Health Dashboard

**Create `analytics/monitoring/health_check.sh`:**
```bash
#!/bin/bash
# System Health Check Script

echo "==================================================================="
echo "OTEL Analytics System Health Check"
echo "==================================================================="

# Check OTEL Collectors
echo ""
echo "1. OTEL Collector Status:"
echo "-------------------------------------------------------------------"
for env in prod dev qa; do
    status=$(docker inspect -f '{{.State.Health.Status}}' otel-collector-${env} 2>/dev/null || echo "not running")
    echo "  otel-collector-${env}: ${status}"
done

# Check S3 Uploaders
echo ""
echo "2. S3 Uploader Status:"
echo "-------------------------------------------------------------------"
for env in prod dev qa; do
    status=$(docker inspect -f '{{.State.Status}}' s3-uploader-${env} 2>/dev/null || echo "not running")
    echo "  s3-uploader-${env}: ${status}"
done

# Check ClickHouse
echo ""
echo "3. ClickHouse Status:"
echo "-------------------------------------------------------------------"
clickhouse_status=$(docker exec clickhouse clickhouse-client --query "SELECT 1" 2>/dev/null || echo "ERROR")
if [ "$clickhouse_status" = "1" ]; then
    echo "  ClickHouse: healthy"
else
    echo "  ClickHouse: unhealthy"
fi

# Check data freshness
echo ""
echo "4. Data Freshness:"
echo "-------------------------------------------------------------------"
docker exec clickhouse clickhouse-client --query "
SELECT
    data_plane_id,
    minutes_since_last_data
FROM otel_metrics.v_data_freshness
FORMAT PrettyCompact
" 2>/dev/null || echo "  Unable to check data freshness"

# Check disk usage
echo ""
echo "5. Disk Usage:"
echo "-------------------------------------------------------------------"
docker exec clickhouse clickhouse-client --query "
SELECT
    formatReadableSize(sum(bytes_on_disk)) AS total_size
FROM system.parts
WHERE database = 'otel_metrics' AND active
FORMAT PrettyCompact
" 2>/dev/null || echo "  Unable to check disk usage"

# Check S3 upload rate
echo ""
echo "6. S3 Upload Metrics (last hour):"
echo "-------------------------------------------------------------------"
docker exec s3-uploader-prod curl -s http://localhost:8080/metrics 2>/dev/null | grep s3_uploader || echo "  Metrics not available"

echo ""
echo "==================================================================="
```

### 5.2 Setup Prometheus Monitoring for Analytics Components

**Create `analytics/monitoring/prometheus-rules.yml`:**
```yaml
groups:
  - name: analytics_pipeline
    interval: 30s
    rules:
      # S3 Uploader Alerts
      - alert: S3UploaderDown
        expr: up{job="s3-uploader"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "S3 uploader is down"
          description: "S3 uploader for {{ $labels.data_plane }} has been down for 5 minutes"

      - alert: S3UploadsFailing
        expr: rate(s3_uploader_files_processed_total{status="error"}[5m]) > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "S3 uploads are failing"
          description: "S3 uploader is experiencing upload failures"

      - alert: HighBufferUsage
        expr: otel_buffer_size_bytes / otel_buffer_max_bytes > 0.8
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "OTEL collector buffer usage high"
          description: "Buffer usage is at {{ $value | humanizePercentage }}"

      # ClickHouse Alerts
      - alert: ClickHouseDown
        expr: up{job="clickhouse"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "ClickHouse is down"
          description: "ClickHouse has been unreachable for 2 minutes"

      - alert: SlowQueries
        expr: histogram_quantile(0.95, rate(clickhouse_query_duration_seconds_bucket[5m])) > 10
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "ClickHouse queries are slow"
          description: "P95 query latency is {{ $value }}s"

      - alert: DataFreshnessIssue
        expr: time() - clickhouse_last_data_timestamp > 3600
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Data is stale"
          description: "No new data received in the last hour"
```

---

## Step 6: Performance Benchmarking

### 6.1 Create Benchmark Suite

**Create `analytics/benchmarks/run_benchmarks.sh`:**
```bash
#!/bin/bash
# Performance Benchmark Suite

echo "==================================================================="
echo "Analytics Pipeline Performance Benchmarks"
echo "==================================================================="

# Benchmark 1: Data ingestion rate
echo ""
echo "Benchmark 1: Data Ingestion Rate"
echo "-------------------------------------------------------------------"
echo "Generating load for 60 seconds..."
cd ../../load-generator
python load_generator.py -t http://localhost:8081 -r 100 -d 60 &
LOAD_PID=$!

sleep 65
kill $LOAD_PID 2>/dev/null

echo "Checking ingestion metrics..."
docker exec clickhouse clickhouse-client --query "
SELECT
    count() AS records_ingested,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    formatReadableSize(sum(data_compressed_bytes)) AS compressed_size,
    round(sum(data_uncompressed_bytes) / sum(data_compressed_bytes), 2) AS compression_ratio
FROM system.parts
WHERE database = 'otel_metrics'
  AND table = 'metrics_local_v2'
  AND modification_time >= now() - INTERVAL 2 MINUTE
FORMAT PrettyCompact
"

# Benchmark 2: Query performance
echo ""
echo "Benchmark 2: Query Performance"
echo "-------------------------------------------------------------------"

queries=(
    "SELECT count() FROM metrics_local_v2"
    "SELECT environment, count() FROM metrics_local_v2 WHERE date >= today() - 7 GROUP BY environment"
    "SELECT environment, http_route, quantile(0.95)(metric_value) FROM metrics_local_v2 WHERE metric_name = 'http_server_duration' AND date >= today() - 7 GROUP BY environment, http_route"
    "SELECT * FROM v_daily_summary WHERE date >= today() - 30"
)

for query in "${queries[@]}"; do
    echo "Query: $query"
    docker exec clickhouse clickhouse-client --time --query "$query FORMAT Null"
    echo ""
done

# Benchmark 3: Storage efficiency
echo ""
echo "Benchmark 3: Storage Efficiency"
echo "-------------------------------------------------------------------"
docker exec clickhouse clickhouse-client --query "
SELECT
    table,
    formatReadableSize(sum(bytes_on_disk)) AS disk_size,
    formatReadableSize(sum(data_uncompressed_bytes)) AS uncompressed_size,
    round(sum(data_compressed_bytes) / sum(data_uncompressed_bytes), 3) AS compression_ratio,
    sum(rows) AS total_rows,
    round(sum(bytes_on_disk) / sum(rows), 2) AS bytes_per_row
FROM system.parts
WHERE database = 'otel_metrics' AND active
GROUP BY table
FORMAT PrettyCompact
"

echo ""
echo "==================================================================="
```

---

## Expected Outcomes

✅ **Fully Integrated Pipeline**:
- OTEL Collector sending to both Prometheus and S3
- S3 uploader processing files efficiently
- ClickHouse querying S3 data successfully
- Dashboard displaying real-time and historical data

✅ **Optimized Performance**:
- Query latency <5 seconds for complex aggregations
- 10x+ compression ratio on stored data
- Efficient partitioning and indexing
- Query cache reducing repeated query load

✅ **Production Ready**:
- Data quality monitoring in place
- Anomaly detection working
- SLO tracking functional
- Health checks automated
- Alerts configured

✅ **Advanced Features**:
- Forecasting capabilities
- Anomaly detection
- SLO compliance tracking
- Cost attribution reporting

---

## Next Steps

You now have a complete, production-ready analytics extension! The system includes:
- ✅ Long-term metrics storage in S3/Iceberg
- ✅ Fast analytics with ClickHouse
- ✅ Rich analytics dashboard
- ✅ Data quality monitoring
- ✅ Performance optimization
- ✅ Advanced analytics features

The implementation is complete and ready for deployment and demonstration!