# Phase 10: ClickHouse Deployment

## Overview
Deploy ClickHouse as the analytics query engine for long-term metrics stored in S3. Configure ClickHouse to query Iceberg tables and optimize for analytical workloads.

---

## Architecture

```
┌────────────────────────────────────────────┐
│  ClickHouse Server                         │
│  ┌──────────────────────────────────────┐ │
│  │  S3 Table Engine                     │ │
│  │  ↓                                   │ │
│  │  Iceberg Integration                 │ │
│  │  ↓                                   │ │
│  │  Materialized Views (Aggregations)  │ │
│  └──────────────────────────────────────┘ │
└───────────────┬────────────────────────────┘
                │
                ↓
        AWS S3 (Iceberg Tables)
```

---

## Step 1: ClickHouse Server Setup

### 1.1 Create ClickHouse Directory Structure
```bash
mkdir -p analytics/clickhouse
cd analytics/clickhouse
mkdir -p config
mkdir -p data
mkdir -p logs
```

### 1.2 Create ClickHouse Configuration

**Create `config/config.xml`:**
```xml
<?xml version="1.0"?>
<clickhouse>
    <logger>
        <level>information</level>
        <log>/var/log/clickhouse-server/clickhouse-server.log</log>
        <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
        <size>1000M</size>
        <count>10</count>
    </logger>

    <http_port>8123</http_port>
    <tcp_port>9000</tcp_port>
    <mysql_port>9004</mysql_port>

    <listen_host>::</listen_host>

    <max_connections>4096</max_connections>
    <max_concurrent_queries>100</max_concurrent_queries>

    <!-- Memory limits -->
    <max_server_memory_usage>0</max_server_memory_usage>
    <max_memory_usage>10000000000</max_memory_usage>

    <!-- Paths -->
    <path>/var/lib/clickhouse/</path>
    <tmp_path>/var/lib/clickhouse/tmp/</tmp_path>
    <user_files_path>/var/lib/clickhouse/user_files/</user_files_path>
    <format_schema_path>/var/lib/clickhouse/format_schemas/</format_schema_path>

    <!-- Users configuration -->
    <users_config>users.xml</users_config>

    <!-- Default profile settings -->
    <profiles>
        <default>
            <max_memory_usage>10000000000</max_memory_usage>
            <use_uncompressed_cache>0</use_uncompressed_cache>
            <load_balancing>random</load_balancing>
        </default>

        <readonly>
            <readonly>1</readonly>
        </readonly>
    </profiles>

    <!-- Quotas -->
    <quotas>
        <default>
            <interval>
                <duration>3600</duration>
                <queries>0</queries>
                <errors>0</errors>
                <result_rows>0</result_rows>
                <read_rows>0</read_rows>
                <execution_time>0</execution_time>
            </interval>
        </default>
    </quotas>

    <!-- S3 Configuration -->
    <s3>
        <endpoint>https://s3.us-east-1.amazonaws.com</endpoint>
        <access_key_id from_env="AWS_ACCESS_KEY_ID"/>
        <secret_access_key from_env="AWS_SECRET_ACCESS_KEY"/>
        <region>us-east-1</region>
        <use_environment_credentials>true</use_environment_credentials>
        <max_connections>100</max_connections>
    </s3>

    <!-- Query cache -->
    <query_cache>
        <max_size_in_bytes>1073741824</max_size_in_bytes>
        <max_entries>1024</max_entries>
        <max_entry_size_in_bytes>1048576</max_entry_size_in_bytes>
    </query_cache>
</clickhouse>
```

**Create `config/users.xml`:**
```xml
<?xml version="1.0"?>
<clickhouse>
    <users>
        <!-- Default user for maintenance -->
        <default>
            <password></password>
            <networks>
                <ip>::/0</ip>
            </networks>
            <profile>default</profile>
            <quota>default</quota>
        </default>

        <!-- Admin user -->
        <admin>
            <password_sha256_hex><!-- SHA256 hash of password --></password_sha256_hex>
            <networks>
                <ip>::1</ip>
                <ip>127.0.0.1</ip>
            </networks>
            <profile>default</profile>
            <quota>default</quota>
        </admin>

        <!-- Read-only user for dashboard -->
        <dashboard>
            <password from_env="CLICKHOUSE_DASHBOARD_PASSWORD"/>
            <networks>
                <ip>::/0</ip>
            </networks>
            <profile>readonly</profile>
            <quota>default</quota>
        </dashboard>
    </users>
</clickhouse>
```

### 1.3 Create Docker Compose Service

**Add to `docker-compose.yml`:**
```yaml
  # ClickHouse for long-term analytics
  clickhouse:
    image: clickhouse/clickhouse-server:23.12
    container_name: clickhouse
    ports:
      - "8123:8123"  # HTTP interface
      - "9000:9000"  # Native client
      - "9004:9004"  # MySQL protocol
    volumes:
      - ./analytics/clickhouse/config/config.xml:/etc/clickhouse-server/config.xml:ro
      - ./analytics/clickhouse/config/users.xml:/etc/clickhouse-server/users.xml:ro
      - clickhouse-data:/var/lib/clickhouse
      - clickhouse-logs:/var/log/clickhouse-server
    environment:
      - CLICKHOUSE_DB=otel_metrics
      - CLICKHOUSE_USER=default
      - CLICKHOUSE_PASSWORD=
      - CLICKHOUSE_DASHBOARD_PASSWORD=${CLICKHOUSE_DASHBOARD_PASSWORD:-dashboard123}
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - AWS_REGION=${AWS_REGION:-us-east-1}
    networks:
      - otel-network
    healthcheck:
      test: ["CMD", "clickhouse-client", "--query", "SELECT 1"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    ulimits:
      nofile:
        soft: 262144
        hard: 262144

volumes:
  clickhouse-data:
    driver: local
  clickhouse-logs:
    driver: local
```

---

## Step 2: Create Database Schema

### 2.1 Initialize Database

**Create `analytics/clickhouse/init.sql`:**
```sql
-- Create database
CREATE DATABASE IF NOT EXISTS otel_metrics;

USE otel_metrics;

-- Main metrics table reading from S3 Iceberg
CREATE TABLE IF NOT EXISTS metrics_s3
(
    timestamp DateTime64(3),
    metric_name String,
    metric_value Float64,
    metric_type String,

    -- Labels (using Map type)
    labels Map(String, String),

    -- Resource attributes
    environment LowCardinality(String),
    data_plane_id LowCardinality(String),
    service_name LowCardinality(String),
    cluster_id LowCardinality(String),
    region LowCardinality(String),

    -- Partition columns
    year UInt16,
    month UInt8,
    day UInt8,
    hour UInt8
)
ENGINE = S3(
    'https://s3.us-east-1.amazonaws.com/otel-metrics-lake/metrics/*/*.parquet',
    'Parquet'
)
PARTITION BY (year, month, day, data_plane_id)
SETTINGS
    s3_max_redirects = 10,
    input_format_parquet_import_nested = 1;

-- Optimized table with MergeTree for faster queries
CREATE TABLE IF NOT EXISTS metrics_local
(
    timestamp DateTime64(3),
    metric_name LowCardinality(String),
    metric_value Float64,
    metric_type LowCardinality(String),

    -- Flattened common labels
    http_method LowCardinality(String),
    http_route String,
    http_status_code LowCardinality(String),

    -- Resource attributes
    environment LowCardinality(String),
    data_plane_id LowCardinality(String),
    service_name LowCardinality(String),

    -- Partition columns
    date Date,
    hour UInt8
)
ENGINE = MergeTree()
PARTITION BY (toYYYYMM(date), data_plane_id)
ORDER BY (environment, metric_name, timestamp)
TTL date + INTERVAL 90 DAY
SETTINGS index_granularity = 8192;

-- Materialized view to populate local table from S3
CREATE MATERIALIZED VIEW IF NOT EXISTS metrics_local_mv TO metrics_local
AS SELECT
    timestamp,
    metric_name,
    metric_value,
    metric_type,

    -- Extract common labels
    labels['http_method'] AS http_method,
    labels['http_route'] AS http_route,
    labels['http_status_code'] AS http_status_code,

    environment,
    data_plane_id,
    service_name,

    toDate(timestamp) AS date,
    toHour(timestamp) AS hour
FROM metrics_s3;
```

### 2.2 Create Aggregation Tables

**Create `analytics/clickhouse/aggregations.sql`:**
```sql
USE otel_metrics;

-- Hourly aggregations
CREATE TABLE IF NOT EXISTS metrics_hourly
(
    date Date,
    hour UInt8,
    environment LowCardinality(String),
    data_plane_id LowCardinality(String),
    metric_name LowCardinality(String),
    http_route String,

    -- Aggregated metrics
    request_count UInt64,
    error_count UInt64,
    total_duration Float64,
    min_duration Float64,
    max_duration Float64,
    p50_duration Float64,
    p95_duration Float64,
    p99_duration Float64
)
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, hour, environment, data_plane_id, metric_name, http_route)
TTL date + INTERVAL 90 DAY;

-- Materialized view for hourly aggregations
CREATE MATERIALIZED VIEW IF NOT EXISTS metrics_hourly_mv TO metrics_hourly
AS SELECT
    toDate(timestamp) AS date,
    toHour(timestamp) AS hour,
    environment,
    data_plane_id,
    metric_name,
    http_route,

    count() AS request_count,
    countIf(http_status_code >= '500') AS error_count,
    sum(metric_value) AS total_duration,
    min(metric_value) AS min_duration,
    max(metric_value) AS max_duration,
    quantile(0.50)(metric_value) AS p50_duration,
    quantile(0.95)(metric_value) AS p95_duration,
    quantile(0.99)(metric_value) AS p99_duration
FROM metrics_local
WHERE metric_name = 'http_server_duration'
GROUP BY date, hour, environment, data_plane_id, metric_name, http_route;

-- Daily aggregations for long-term trends
CREATE TABLE IF NOT EXISTS metrics_daily
(
    date Date,
    environment LowCardinality(String),
    data_plane_id LowCardinality(String),

    total_requests UInt64,
    total_errors UInt64,
    avg_p95_latency Float64,
    max_p95_latency Float64
)
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, environment, data_plane_id)
TTL date + INTERVAL 730 DAY;

-- Materialized view for daily aggregations
CREATE MATERIALIZED VIEW IF NOT EXISTS metrics_daily_mv TO metrics_daily
AS SELECT
    date,
    environment,
    data_plane_id,

    sum(request_count) AS total_requests,
    sum(error_count) AS total_errors,
    avg(p95_duration) AS avg_p95_latency,
    max(p95_duration) AS max_p95_latency
FROM metrics_hourly
GROUP BY date, environment, data_plane_id;
```

### 2.3 Create Helper Functions

**Create `analytics/clickhouse/functions.sql`:**
```sql
USE otel_metrics;

-- Function to calculate error rate
CREATE FUNCTION IF NOT EXISTS errorRate AS (errors, total) ->
    if(total > 0, (errors / total) * 100, 0);

-- Function to format duration
CREATE FUNCTION IF NOT EXISTS formatDuration AS (seconds) ->
    multiIf(
        seconds < 0.001, concat(toString(round(seconds * 1000000)), 'µs'),
        seconds < 1, concat(toString(round(seconds * 1000)), 'ms'),
        seconds < 60, concat(toString(round(seconds, 2)), 's'),
        concat(toString(floor(seconds / 60)), 'm ', toString(round(seconds % 60)), 's')
    );
```

---

## Step 3: Data Loading and Verification

### 3.1 Start ClickHouse

```bash
docker-compose up -d clickhouse

# Wait for healthy status
docker-compose ps clickhouse
```

### 3.2 Initialize Schema

```bash
# Copy SQL files to container
docker cp analytics/clickhouse/init.sql clickhouse:/tmp/
docker cp analytics/clickhouse/aggregations.sql clickhouse:/tmp/
docker cp analytics/clickhouse/functions.sql clickhouse:/tmp/

# Execute initialization
docker exec -it clickhouse clickhouse-client --multiquery < /tmp/init.sql
docker exec -it clickhouse clickhouse-client --multiquery < /tmp/aggregations.sql
docker exec -it clickhouse clickhouse-client --multiquery < /tmp/functions.sql
```

### 3.3 Verify Tables Created

```bash
docker exec -it clickhouse clickhouse-client --query "SHOW TABLES FROM otel_metrics"

# Expected output:
# metrics_s3
# metrics_local
# metrics_hourly
# metrics_daily
```

### 3.4 Query S3 Data

```sql
-- Connect to ClickHouse
docker exec -it clickhouse clickhouse-client

-- Test S3 connectivity
SELECT count()
FROM otel_metrics.metrics_s3
LIMIT 10;

-- View sample data
SELECT
    timestamp,
    environment,
    metric_name,
    metric_value,
    data_plane_id
FROM otel_metrics.metrics_s3
LIMIT 10;

-- Check partitions
SELECT
    partition,
    count() as records
FROM otel_metrics.metrics_s3
GROUP BY partition
ORDER BY partition DESC;
```

---

## Step 4: Optimization and Tuning

### 4.1 Create Indexes

```sql
USE otel_metrics;

-- Add skipping indexes for common queries
ALTER TABLE metrics_local
ADD INDEX idx_metric_name metric_name TYPE set(100) GRANULARITY 4;

ALTER TABLE metrics_local
ADD INDEX idx_environment environment TYPE set(10) GRANULARITY 4;

ALTER TABLE metrics_local
ADD INDEX idx_http_route http_route TYPE bloom_filter() GRANULARITY 4;
```

### 4.2 Configure Query Cache

**Update `config/config.xml` to enable query cache:**
```xml
<query_cache>
    <max_size_in_bytes>5368709120</max_size_in_bytes> <!-- 5GB -->
    <max_entries>10000</max_entries>
    <max_entry_size_in_bytes>10485760</max_entry_size_in_bytes> <!-- 10MB -->
</query_cache>
```

### 4.3 Setup Compression

```sql
-- Optimize compression for metrics_local
ALTER TABLE metrics_local
MODIFY SETTING
    compress_primary_key = 1,
    min_compress_block_size = 65536,
    max_compress_block_size = 1048576;
```

---

## Step 5: Sample Analytical Queries

### 5.1 Create Views for Common Queries

**Create `analytics/clickhouse/views.sql`:**
```sql
USE otel_metrics;

-- Request rate over time
CREATE VIEW IF NOT EXISTS v_request_rate AS
SELECT
    toStartOfHour(timestamp) AS hour,
    environment,
    data_plane_id,
    count() / 3600 AS requests_per_second
FROM metrics_local
WHERE metric_name = 'http_server_duration'
GROUP BY hour, environment, data_plane_id
ORDER BY hour DESC;

-- Latency percentiles by endpoint
CREATE VIEW IF NOT EXISTS v_latency_by_endpoint AS
SELECT
    toDate(timestamp) AS date,
    environment,
    http_route,
    count() AS request_count,
    quantile(0.50)(metric_value) AS p50_latency,
    quantile(0.95)(metric_value) AS p95_latency,
    quantile(0.99)(metric_value) AS p99_latency
FROM metrics_local
WHERE metric_name = 'http_server_duration'
GROUP BY date, environment, http_route
ORDER BY date DESC, request_count DESC;

-- Error rate by environment
CREATE VIEW IF NOT EXISTS v_error_rate AS
SELECT
    toDate(timestamp) AS date,
    environment,
    data_plane_id,
    count() AS total_requests,
    countIf(http_status_code >= '500') AS error_count,
    errorRate(countIf(http_status_code >= '500'), count()) AS error_rate_percent
FROM metrics_local
WHERE metric_name = 'http_server_duration'
GROUP BY date, environment, data_plane_id
ORDER BY date DESC;

-- Top slowest endpoints
CREATE VIEW IF NOT EXISTS v_slowest_endpoints AS
SELECT
    environment,
    http_route,
    count() AS request_count,
    quantile(0.95)(metric_value) AS p95_latency,
    max(metric_value) AS max_latency
FROM metrics_local
WHERE metric_name = 'http_server_duration'
  AND timestamp >= now() - INTERVAL 7 DAY
GROUP BY environment, http_route
ORDER BY p95_latency DESC
LIMIT 20;

-- Daily trends
CREATE VIEW IF NOT EXISTS v_daily_trends AS
SELECT
    date,
    environment,
    total_requests,
    total_errors,
    errorRate(total_errors, total_requests) AS error_rate,
    avg_p95_latency,
    max_p95_latency
FROM metrics_daily
ORDER BY date DESC;
```

### 5.2 Example Analytical Queries

```sql
-- Month-over-month comparison
SELECT
    toStartOfMonth(date) AS month,
    environment,
    sum(total_requests) AS monthly_requests,
    avg(avg_p95_latency) AS avg_latency,
    sum(total_errors) / sum(total_requests) * 100 AS error_rate
FROM metrics_daily
WHERE date >= today() - INTERVAL 6 MONTH
GROUP BY month, environment
ORDER BY month DESC, environment;

-- Capacity planning - growth rate
WITH monthly_stats AS (
    SELECT
        toStartOfMonth(date) AS month,
        environment,
        sum(total_requests) AS requests
    FROM metrics_daily
    GROUP BY month, environment
)
SELECT
    month,
    environment,
    requests,
    requests - lagInFrame(requests, 1) OVER (PARTITION BY environment ORDER BY month) AS growth,
    round((requests - lagInFrame(requests, 1) OVER (PARTITION BY environment ORDER BY month)) /
          lagInFrame(requests, 1) OVER (PARTITION BY environment ORDER BY month) * 100, 2) AS growth_percent
FROM monthly_stats
ORDER BY month DESC, environment;

-- Anomaly detection - find unusual latency spikes
SELECT
    date,
    hour,
    environment,
    http_route,
    p95_duration,
    avg(p95_duration) OVER (
        PARTITION BY environment, http_route
        ORDER BY date, hour
        ROWS BETWEEN 24 PRECEDING AND 1 PRECEDING
    ) AS avg_24h,
    stddevPop(p95_duration) OVER (
        PARTITION BY environment, http_route
        ORDER BY date, hour
        ROWS BETWEEN 24 PRECEDING AND 1 PRECEDING
    ) AS stddev_24h
FROM metrics_hourly
WHERE date >= today() - INTERVAL 7 DAY
HAVING p95_duration > avg_24h + (3 * stddev_24h)
ORDER BY date DESC, hour DESC;
```

---

## Step 6: Performance Testing

### 6.1 Benchmark Queries

**Create `analytics/clickhouse/benchmark.sql`:**
```sql
-- Benchmark simple aggregation
SELECT count() FROM metrics_local;

-- Benchmark time-range query
SELECT
    toStartOfHour(timestamp) AS hour,
    count() AS requests
FROM metrics_local
WHERE timestamp >= now() - INTERVAL 7 DAY
GROUP BY hour
ORDER BY hour;

-- Benchmark percentile calculation
SELECT
    environment,
    quantile(0.95)(metric_value) AS p95
FROM metrics_local
WHERE metric_name = 'http_server_duration'
  AND timestamp >= now() - INTERVAL 24 HOUR
GROUP BY environment;

-- Benchmark complex join
SELECT
    d.date,
    d.environment,
    d.total_requests,
    h.p95_duration
FROM metrics_daily d
INNER JOIN metrics_hourly h
    ON d.date = h.date
    AND d.environment = h.environment
WHERE d.date >= today() - INTERVAL 30 DAY
ORDER BY d.date DESC;
```

### 6.2 Monitor Query Performance

```sql
-- Enable query logging
SET log_queries = 1;
SET log_query_threads = 1;

-- View slow queries
SELECT
    query_id,
    query,
    query_duration_ms,
    read_rows,
    read_bytes,
    memory_usage
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_duration_ms > 1000
ORDER BY query_duration_ms DESC
LIMIT 20;

-- Check table sizes
SELECT
    database,
    table,
    formatReadableSize(sum(bytes)) AS size,
    sum(rows) AS rows,
    max(modification_time) AS latest_modification
FROM system.parts
WHERE database = 'otel_metrics'
GROUP BY database, table
ORDER BY sum(bytes) DESC;
```

---

## Step 7: Backup and Maintenance

### 7.1 Setup Automated Backups

**Create backup script `analytics/clickhouse/backup.sh`:**
```bash
#!/bin/bash

BACKUP_DIR="/backup/clickhouse"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p ${BACKUP_DIR}/${DATE}

# Backup using clickhouse-backup tool
docker exec clickhouse clickhouse-backup create ${DATE}

# Copy to backup location
docker cp clickhouse:/var/lib/clickhouse/backup/${DATE} ${BACKUP_DIR}/

echo "Backup completed: ${BACKUP_DIR}/${DATE}"
```

### 7.2 Optimize Tables Periodically

```sql
-- Optimize tables to merge parts
OPTIMIZE TABLE metrics_local FINAL;
OPTIMIZE TABLE metrics_hourly FINAL;
OPTIMIZE TABLE metrics_daily FINAL;

-- Clean up old mutations
SYSTEM STOP MERGES otel_metrics.metrics_local;
SYSTEM START MERGES otel_metrics.metrics_local;
```

### 7.3 Monitor Disk Usage

```sql
-- Check disk usage by partition
SELECT
    database,
    table,
    partition,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    sum(rows) AS rows
FROM system.parts
WHERE active AND database = 'otel_metrics'
GROUP BY database, table, partition
ORDER BY sum(bytes_on_disk) DESC;
```

---

## Step 8: Security Hardening

### 8.1 Create User Roles

```sql
-- Create readonly user for dashboard
CREATE USER IF NOT EXISTS dashboard
IDENTIFIED WITH plaintext_password BY 'secure_password_here'
SETTINGS readonly = 1;

-- Grant access to specific database
GRANT SELECT ON otel_metrics.* TO dashboard;

-- Create admin user
CREATE USER IF NOT EXISTS admin
IDENTIFIED WITH sha256_password BY 'admin_password_hash';

GRANT ALL ON *.* TO admin WITH GRANT OPTION;
```

### 8.2 Enable SSL/TLS

**Update `config/config.xml`:**
```xml
<https_port>8443</https_port>
<tcp_port_secure>9440</tcp_port_secure>

<openSSL>
    <server>
        <certificateFile>/etc/clickhouse-server/server.crt</certificateFile>
        <privateKeyFile>/etc/clickhouse-server/server.key</privateKeyFile>
        <verificationMode>none</verificationMode>
        <loadDefaultCAFile>true</loadDefaultCAFile>
        <cacheSessions>true</cacheSessions>
        <disableProtocols>sslv2,sslv3</disableProtocols>
        <preferServerCiphers>true</preferServerCiphers>
    </server>
</openSSL>
```

---

## Expected Outcomes

✅ **ClickHouse Running**:
- Server accessible on ports 8123 (HTTP) and 9000 (native)
- Database and tables created
- S3 integration working

✅ **Data Queryable**:
- Can query S3 Iceberg data
- Aggregations working correctly
- Query performance acceptable (<5s for complex queries)

✅ **Optimizations Active**:
- Materialized views updating
- Indexes created
- Compression enabled
- Query cache working

✅ **Production Ready**:
- Backups configured
- Security hardened
- Monitoring enabled
- Maintenance procedures documented

---

## Troubleshooting

**Issue: Cannot connect to S3**
```bash
# Check AWS credentials
docker exec clickhouse env | grep AWS

# Test S3 access
docker exec clickhouse clickhouse-client --query "
SELECT * FROM s3('https://s3.us-east-1.amazonaws.com/otel-metrics-lake/metrics/*.parquet', 'Parquet') LIMIT 1
"
```

**Issue: Slow queries**
```sql
-- Check if indexes are being used
EXPLAIN indexes = 1
SELECT * FROM metrics_local WHERE environment = 'prod';

-- Check partition pruning
EXPLAIN
SELECT * FROM metrics_local
WHERE date >= today() - INTERVAL 7 DAY;
```

**Issue: High memory usage**
```sql
-- Reduce memory limit for user
ALTER USER dashboard SETTINGS max_memory_usage = 5000000000;

-- Check current memory usage
SELECT
    query,
    memory_usage,
    formatReadableSize(memory_usage) AS memory
FROM system.processes;
```

---

## Next Steps

Proceed to **Phase 11: Analytics Dashboard** to build the UI for long-term metrics analysis.
