# Storage & Persistence Guide

This guide provides detailed information about Prometheus storage, data persistence, and operational tasks.

---

## Table of Contents

- [Storage Overview](#storage-overview)
- [Exploring Storage](#exploring-storage)
- [Storage Configuration](#storage-configuration)
- [Backup & Recovery](#backup--recovery)
- [Troubleshooting](#troubleshooting)
- [Advanced Topics](#advanced-topics)

---

## Storage Overview

### Where is Data Stored?

Prometheus stores all metrics in a **local time-series database (TSDB)** on disk.

**Container path**: `/prometheus`
**Host path**: `/var/lib/docker/volumes/prometheus-poc_prometheus-data/_data`
**Volume name**: `prometheus-data`

### Storage Architecture

```
┌─────────────────────────────────────────────────┐
│              Prometheus Container               │
│                                                 │
│  ┌───────────────────────────────────────────┐ │
│  │        /prometheus (TSDB)                 │ │
│  │                                           │ │
│  │  ┌─────────────┐    ┌─────────────┐     │ │
│  │  │ chunks_head │    │     WAL     │     │ │
│  │  │ (in-memory) │    │ (durability)│     │ │
│  │  └─────────────┘    └─────────────┘     │ │
│  │                                           │ │
│  │  ┌──────────────────────────────────┐   │ │
│  │  │  Time-based Blocks (2h each)     │   │ │
│  │  │  ┌─────┐  ┌─────┐  ┌─────┐       │   │ │
│  │  │  │Block│  │Block│  │Block│  ...  │   │ │
│  │  │  └─────┘  └─────┘  └─────┘       │   │ │
│  │  └──────────────────────────────────┘   │ │
│  └───────────────────────────────────────────┘ │
│                      │                          │
└──────────────────────┼──────────────────────────┘
                       │
                       ▼
           ┌───────────────────────┐
           │   Docker Volume       │
           │   prometheus-data     │
           │   (persistent)        │
           └───────────────────────┘
```

### Data Lifecycle

```
1. Scrape → 2. WAL → 3. Memory → 4. Disk Blocks → 5. Compaction → 6. Retention Cleanup
   ▼           ▼        ▼            ▼              ▼                ▼
 10s        Immediate  2h buffer   2h blocks     Background        After 15d
interval    durability              created       merge older       deleted
```

---

## Exploring Storage

### Check Storage Size

```bash
# Quick size check
docker exec prometheus du -sh /prometheus

# Expected output: 10M-50M (depending on traffic)
```

### List Storage Contents

```bash
# List all directories and files
docker exec prometheus ls -lh /prometheus

# Expected output:
# drwxr-xr-x    2 nobody   nobody      64 Oct 25 18:30 chunks_head
# drwxr-xr-x    3 nobody   nobody      96 Oct 25 18:30 01HJBC6XYZ123
# drwxr-xr-x    2 nobody   nobody     128 Oct 25 18:30 wal
# -rw-r--r--    1 nobody   nobody       0 Oct 25 18:25 lock
```

### Examine a Block

```bash
# List blocks
docker exec prometheus ls -d /prometheus/01*

# Examine block contents
docker exec prometheus ls -lh /prometheus/01HJBC6XYZ123/

# Expected contents:
# -rw-r--r--    1 nobody   nobody    1.2M index
# drwxr-xr-x    2 nobody   nobody      64 chunks/
# -rw-r--r--    1 nobody   nobody     512 meta.json
# -rw-r--r--    1 nobody   nobody       0 tombstones
```

### View Block Metadata

```bash
# Read block metadata
docker exec prometheus cat /prometheus/01HJBC6XYZ123/meta.json | python3 -m json.tool

# Output shows:
# - Time range (minTime, maxTime)
# - Number of samples
# - Number of series
# - Compaction level
```

### Check TSDB Stats via Prometheus UI

1. Open http://localhost:9090
2. Navigate to **Status** → **TSDB Status**

**You'll see**:
```
Head Stats:
  Number of Series: 87
  Number of Chunks: 234
  Number of Samples: 15,432
  Chunks Size: 1.8 MB

On-Disk Stats:
  Number of Series: 450
  Number of Chunks: 1,289
  Total Size: 12.5 MB
```

### Query Storage via API

```bash
# Get TSDB statistics
curl -s http://localhost:9090/api/v1/status/tsdb | python3 -m json.tool

# Get runtime information
curl -s http://localhost:9090/api/v1/status/runtimeinfo | python3 -m json.tool
```

---

## Storage Configuration

### Retention Settings

Configure how long Prometheus keeps data:

```yaml
# In docker-compose.yml
prometheus:
  command:
    # Option 1: Retention by time
    - '--storage.tsdb.retention.time=30d'

    # Option 2: Retention by size
    - '--storage.tsdb.retention.size=10GB'

    # Can use both - whichever limit hits first
    - '--storage.tsdb.retention.time=30d'
    - '--storage.tsdb.retention.size=10GB'
```

### Storage Path

```yaml
prometheus:
  command:
    - '--storage.tsdb.path=/prometheus'  # Default location
```

### Block Duration

Advanced: Control block size (default: 2h)

```yaml
prometheus:
  command:
    - '--storage.tsdb.min-block-duration=2h'   # Minimum block duration
    - '--storage.tsdb.max-block-duration=36h'  # Maximum after compaction
```

### WAL Compression

Enable Write-Ahead Log compression to save disk space:

```yaml
prometheus:
  command:
    - '--storage.tsdb.wal-compression'
```

### Complete Example

```yaml
# docker-compose.yml
services:
  prometheus:
    image: prom/prometheus:latest
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--storage.tsdb.retention.size=10GB'
      - '--storage.tsdb.wal-compression'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
```

---

## Backup & Recovery

### Method 1: Volume Backup (Recommended)

#### Create Backup

```bash
# Stop Prometheus to ensure consistency
docker-compose stop prometheus

# Create tarball of volume
docker run --rm \
  -v prometheus-poc_prometheus-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/prometheus-backup-$(date +%Y%m%d-%H%M%S).tar.gz /data

# Restart Prometheus
docker-compose start prometheus

# Verify backup
ls -lh prometheus-backup-*.tar.gz
```

#### Restore Backup

```bash
# Stop Prometheus
docker-compose stop prometheus

# Restore from tarball
docker run --rm \
  -v prometheus-poc_prometheus-data:/data \
  -v $(pwd):/backup \
  alpine sh -c "rm -rf /data/* && tar xzf /backup/prometheus-backup-YYYYMMDD-HHMMSS.tar.gz -C /"

# Start Prometheus
docker-compose start prometheus
```

### Method 2: Prometheus Snapshots

Snapshots create a consistent point-in-time copy without stopping Prometheus.

#### Create Snapshot

```bash
# Enable admin API (add to prometheus command in docker-compose.yml)
# - '--web.enable-admin-api'

# Create snapshot
curl -X POST http://localhost:9090/api/v1/admin/tsdb/snapshot

# Response includes snapshot directory:
# {"status":"success","data":{"name":"20251025T183045Z-1a2b3c4d"}}
```

#### Copy Snapshot

```bash
# Copy from container to host
docker cp prometheus:/prometheus/snapshots/20251025T183045Z-1a2b3c4d ./backups/

# Create tarball
tar czf prometheus-snapshot-20251025.tar.gz -C ./backups/ 20251025T183045Z-1a2b3c4d
```

#### Restore from Snapshot

```bash
# Stop Prometheus
docker-compose stop prometheus

# Copy snapshot to volume
docker run --rm \
  -v prometheus-poc_prometheus-data:/data \
  -v $(pwd)/backups:/source \
  alpine sh -c "rm -rf /data/* && cp -r /source/20251025T183045Z-1a2b3c4d/* /data/"

# Start Prometheus
docker-compose start prometheus
```

### Method 3: Volume Clone

Create a duplicate volume for testing:

```bash
# Create new volume
docker volume create prometheus-poc_prometheus-data-backup

# Copy data between volumes
docker run --rm \
  -v prometheus-poc_prometheus-data:/from \
  -v prometheus-poc_prometheus-data-backup:/to \
  alpine sh -c "cp -av /from/. /to/"
```

### Automated Backup Script

```bash
#!/bin/bash
# save as: scripts/backup-prometheus.sh

BACKUP_DIR="./backups"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="prometheus-backup-$TIMESTAMP.tar.gz"

mkdir -p $BACKUP_DIR

echo "Creating Prometheus backup..."

# Option 1: Hot backup (while running)
docker run --rm \
  -v prometheus-poc_prometheus-data:/data \
  -v $(pwd)/$BACKUP_DIR:/backup \
  alpine tar czf /backup/$BACKUP_FILE /data

# Option 2: Cold backup (safer, requires downtime)
# docker-compose stop prometheus
# docker run --rm ...
# docker-compose start prometheus

echo "Backup created: $BACKUP_DIR/$BACKUP_FILE"
ls -lh $BACKUP_DIR/$BACKUP_FILE

# Keep only last 7 backups
cd $BACKUP_DIR && ls -t prometheus-backup-*.tar.gz | tail -n +8 | xargs -r rm
```

---

## Troubleshooting

### Issue: Storage Growing Too Fast

**Check current size:**
```bash
docker exec prometheus du -sh /prometheus
```

**Solutions:**

1. **Reduce retention**:
   ```yaml
   command:
     - '--storage.tsdb.retention.time=7d'  # Instead of 15d
   ```

2. **Set size limit**:
   ```yaml
   command:
     - '--storage.tsdb.retention.size=5GB'
   ```

3. **Enable WAL compression**:
   ```yaml
   command:
     - '--storage.tsdb.wal-compression'
   ```

4. **Reduce scrape frequency**:
   ```yaml
   # In prometheus.yml
   global:
     scrape_interval: 30s  # Instead of 10s
   ```

5. **Reduce metric cardinality**:
   - Remove unnecessary labels
   - Drop unused metrics
   - Use relabel_configs

### Issue: "Out of Space" Error

**Check disk space:**
```bash
# Host machine
df -h /var/lib/docker/volumes

# Inside container
docker exec prometheus df -h /prometheus
```

**Solutions:**

1. **Clean old data**:
   ```bash
   # Delete all data (nuclear option)
   docker-compose down
   docker volume rm prometheus-poc_prometheus-data
   docker-compose up -d
   ```

2. **Manually delete old blocks**:
   ```bash
   # Find old blocks
   docker exec prometheus ls -lt /prometheus/01*

   # Delete specific block (BE CAREFUL!)
   docker exec prometheus rm -rf /prometheus/01HJBC6XYZ123
   ```

3. **Increase disk space** (host machine):
   - Add more disk to VM/host
   - Move Docker data directory
   - Use remote storage

### Issue: Corrupted Storage

**Symptoms:**
- Prometheus won't start
- Errors in logs: "corruption", "invalid block"

**Check logs:**
```bash
docker-compose logs prometheus | grep -i error
```

**Solutions:**

1. **Verify TSDB**:
   ```bash
   docker run --rm \
     -v prometheus-poc_prometheus-data:/prometheus \
     prom/prometheus:latest \
     promtool tsdb analyze /prometheus
   ```

2. **Restore from backup**:
   See [Backup & Recovery](#backup--recovery) section

3. **Start fresh** (last resort):
   ```bash
   docker-compose down -v
   docker-compose up -d
   ```

### Issue: Queries are Slow

**Check TSDB stats:**
- Go to Status → TSDB Status
- Look for high cardinality series

**Solutions:**

1. **Reduce query range**:
   ```promql
   # Instead of [7d]
   rate(http_requests_total[1h])
   ```

2. **Use recording rules**:
   ```yaml
   # In prometheus.yml
   rule_files:
     - "recording_rules.yml"
   ```

3. **Limit series**:
   ```yaml
   # Drop high-cardinality metrics
   metric_relabel_configs:
     - source_labels: [__name__]
       regex: 'high_cardinality_metric.*'
       action: drop
   ```

---

## Advanced Topics

### Storage Internals

#### Chunk Format

Prometheus uses **XOR compression** for float64 values:

```
First value:  64 bits (full float64)
Second value: Delta XOR with first (variable bits)
Third value:  Delta XOR with second (variable bits)
...

Result: ~1.37 bytes per sample on average
```

#### Index Format

The `index` file uses:
- **Posting lists**: Map labels to series IDs
- **Inverted index**: Fast label matcher lookups
- **Symbol table**: Deduplicated label names/values

### Remote Storage Integration

For long-term retention beyond local TSDB:

#### Configure Remote Write

```yaml
# In prometheus.yml
remote_write:
  - url: "http://thanos-receive:19291/api/v1/receive"
    queue_config:
      capacity: 10000
      max_shards: 10
      min_shards: 1
      max_samples_per_send: 5000
      batch_send_deadline: 5s
      min_backoff: 30ms
      max_backoff: 100ms
```

#### Configure Remote Read

```yaml
# In prometheus.yml
remote_read:
  - url: "http://thanos-query:9090/api/v1/read"
    read_recent: true
```

### Prometheus Operator (Kubernetes)

For Kubernetes deployments:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  storage:
    volumeClaimTemplate:
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 50Gi
        storageClassName: fast-ssd
  retention: 30d
  retentionSize: 45GB
```

### Monitoring Storage Health

Create alerts for storage issues:

```yaml
# In alert_rules.yml
groups:
  - name: prometheus_storage
    rules:
      - alert: PrometheusStorageAlmostFull
        expr: prometheus_tsdb_storage_blocks_bytes / prometheus_tsdb_retention_limit_bytes > 0.9
        for: 5m
        annotations:
          summary: "Prometheus storage is almost full"

      - alert: PrometheusWALCorruptions
        expr: rate(prometheus_tsdb_wal_corruptions_total[5m]) > 0
        annotations:
          summary: "Prometheus WAL corruptions detected"
```

---

## Best Practices

### Storage Planning

1. **Estimate storage needs**:
   ```
   Samples per second × Retention seconds × Bytes per sample

   Example:
   100 series × 1 sample/10s × 15 days × 1.5 bytes
   = 100 × 6 × 86400 × 15 × 1.5
   = ~116 MB
   ```

2. **Monitor growth**:
   - Check TSDB Status weekly
   - Alert on rapid growth
   - Review high-cardinality series

3. **Regular backups**:
   - Daily snapshots
   - Weekly full backups
   - Test restore procedures

### Performance Optimization

1. **Use recording rules** for frequently-used queries
2. **Enable WAL compression** to save ~30% space
3. **Tune scrape intervals** based on use case
4. **Limit label cardinality** to avoid explosion

### Disaster Recovery

1. **Document backup procedures**
2. **Test restoration regularly**
3. **Have rollback plan**
4. **Monitor backup success**

---

## Related Documentation

- [architecture.md](architecture.md) - Storage architecture overview
- [step-by-step-guide.md](step-by-step-guide.md) - Implementation steps
- [demo-walkthrough.md](demo-walkthrough.md) - Operational demos
- [testing-checklist.md](testing-checklist.md) - Storage verification tests

---

## References

- [Prometheus Storage Documentation](https://prometheus.io/docs/prometheus/latest/storage/)
- [TSDB Format](https://github.com/prometheus/prometheus/tree/main/tsdb/docs/format)
- [Operational Aspects](https://prometheus.io/docs/prometheus/latest/storage/#operational-aspects)
