# Phase 9: S3 and Iceberg Integration

## Overview
Extend the OTEL Collector to write metrics data to AWS S3 in Apache Iceberg table format for long-term analytics. This phase establishes the foundation for cold-path analytics alongside the existing hot-path monitoring.

---

## Architecture Overview

```
OTEL Collector → File Exporter → Local Parquet Files
                                        ↓
                                  S3 Uploader
                                        ↓
                                   AWS S3 (Iceberg)
                                        ↓
                                Iceberg Catalog (AWS Glue)
```

---

## Step 1: Setup AWS Infrastructure

### 1.1 Create S3 Bucket

**Via AWS CLI:**
```bash
# Set variables
export AWS_REGION=us-east-1
export S3_BUCKET_NAME=otel-metrics-lake
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create S3 bucket
aws s3api create-bucket \
  --bucket ${S3_BUCKET_NAME} \
  --region ${AWS_REGION}

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket ${S3_BUCKET_NAME} \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket ${S3_BUCKET_NAME} \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# Add lifecycle policy for intelligent tiering
aws s3api put-bucket-lifecycle-configuration \
  --bucket ${S3_BUCKET_NAME} \
  --lifecycle-configuration file://s3-lifecycle-policy.json
```

### 1.2 Create `s3-lifecycle-policy.json`
```json
{
  "Rules": [
    {
      "Id": "MoveToIntelligentTiering",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "metrics/"
      },
      "Transitions": [
        {
          "Days": 0,
          "StorageClass": "INTELLIGENT_TIERING"
        }
      ]
    },
    {
      "Id": "DeleteOldData",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "metrics/"
      },
      "Expiration": {
        "Days": 730
      }
    }
  ]
}
```

### 1.3 Setup AWS Glue Catalog

**Create Glue Database:**
```bash
aws glue create-database \
  --database-input '{
    "Name": "otel_metrics_db",
    "Description": "OTEL metrics data lake"
  }'
```

**Verify Database:**
```bash
aws glue get-database --name otel_metrics_db
```

### 1.4 Create IAM Role for S3 Access

**Create `iam-policy.json`:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::otel-metrics-lake/*",
        "arn:aws:s3:::otel-metrics-lake"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetTable",
        "glue:GetPartitions",
        "glue:CreateTable",
        "glue:UpdateTable",
        "glue:BatchCreatePartition",
        "glue:BatchUpdatePartition"
      ],
      "Resource": [
        "arn:aws:glue:*:*:catalog",
        "arn:aws:glue:*:*:database/otel_metrics_db",
        "arn:aws:glue:*:*:table/otel_metrics_db/*"
      ]
    }
  ]
}
```

**Create IAM Policy:**
```bash
aws iam create-policy \
  --policy-name OTELMetricsS3Access \
  --policy-document file://iam-policy.json
```

---

## Step 2: Enhanced OTEL Collector Configuration

### 2.1 Create Directory Structure
```bash
mkdir -p dataplane/otel-collector/exporters
mkdir -p dataplane/otel-collector/buffer
```

### 2.2 Update OTEL Collector Configuration

**Create `dataplane/otel-collector/config-with-s3.yaml`:**
```yaml
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
    send_batch_size: 1024

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

  # Transform metrics for long-term storage
  metricstransform:
    transforms:
      - include: .*
        match_type: regexp
        action: update
        operations:
          # Add timestamp partitioning attributes
          - action: add_label
            new_label: year
            new_value: EXPR(Year(timestamp))
          - action: add_label
            new_label: month
            new_value: EXPR(Month(timestamp))
          - action: add_label
            new_label: day
            new_value: EXPR(Day(timestamp))
          - action: add_label
            new_label: hour
            new_value: EXPR(Hour(timestamp))

  memory_limiter:
    check_interval: 1s
    limit_mib: 512

exporters:
  # Existing Prometheus exporter for real-time monitoring
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: "otel"
    const_labels:
      environment: ${env:ENVIRONMENT}
      data_plane_id: ${env:DATA_PLANE_ID}

  # File exporter for S3 upload
  file:
    path: /buffer/metrics.json
    rotation:
      max_megabytes: 100
      max_days: 1
      max_backups: 10
    format: json

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

  pipelines:
    # Real-time pipeline (existing)
    metrics/realtime:
      receivers: [otlp]
      processors: [memory_limiter, resource, batch]
      exporters: [prometheus]

    # Long-term storage pipeline (new)
    metrics/longterm:
      receivers: [otlp]
      processors: [memory_limiter, resource, metricstransform, batch]
      exporters: [file, logging]
```

---

## Step 3: Build S3 Uploader Service

### 3.1 Create Directory Structure
```bash
mkdir -p analytics/s3-uploader
cd analytics/s3-uploader
```

### 3.2 Create Go-based S3 Uploader

**Initialize Go Module:**
```bash
go mod init github.com/yourusername/otel-prometheus-poc/analytics/s3-uploader
```

**Install Dependencies:**
```bash
go get github.com/aws/aws-sdk-go-v2/config
go get github.com/aws/aws-sdk-go-v2/service/s3
go get github.com/apache/iceberg-go
go get github.com/xitongsys/parquet-go/parquet
go get github.com/xitongsys/parquet-go/writer
go get github.com/fsnotify/fsnotify
```

### 3.3 Create `main.go`

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"path/filepath"
	"time"

	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/s3"
	"github.com/fsnotify/fsnotify"
)

type Config struct {
	BufferDir    string
	S3Bucket     string
	S3Prefix     string
	AWSRegion    string
	UploadInterval time.Duration
}

type MetricRecord struct {
	Timestamp    time.Time         `json:"timestamp"`
	MetricName   string            `json:"metric_name"`
	MetricValue  float64           `json:"metric_value"`
	MetricType   string            `json:"metric_type"`
	Labels       map[string]string `json:"labels"`

	// Partition columns
	Year         int    `json:"year"`
	Month        int    `json:"month"`
	Day          int    `json:"day"`
	Hour         int    `json:"hour"`
	DataPlaneID  string `json:"data_plane_id"`
}

type S3Uploader struct {
	config    *Config
	s3Client  *s3.Client
	watcher   *fsnotify.Watcher
}

func NewS3Uploader(cfg *Config) (*S3Uploader, error) {
	ctx := context.Background()

	// Load AWS config
	awsCfg, err := config.LoadDefaultConfig(ctx, config.WithRegion(cfg.AWSRegion))
	if err != nil {
		return nil, fmt.Errorf("failed to load AWS config: %w", err)
	}

	// Create S3 client
	s3Client := s3.NewFromConfig(awsCfg)

	// Create file watcher
	watcher, err := fsnotify.NewWatcher()
	if err != nil {
		return nil, fmt.Errorf("failed to create watcher: %w", err)
	}

	return &S3Uploader{
		config:   cfg,
		s3Client: s3Client,
		watcher:  watcher,
	}, nil
}

func (u *S3Uploader) Start(ctx context.Context) error {
	log.Println("Starting S3 Uploader...")

	// Watch buffer directory
	err := u.watcher.Add(u.config.BufferDir)
	if err != nil {
		return fmt.Errorf("failed to watch directory: %w", err)
	}

	// Start periodic upload
	ticker := time.NewTicker(u.config.UploadInterval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return ctx.Err()

		case event := <-u.watcher.Events:
			if event.Op&fsnotify.Write == fsnotify.Write {
				log.Printf("File modified: %s", event.Name)
				// Process rotated files only (not the active file)
				if filepath.Ext(event.Name) == ".json" && !isActiveFile(event.Name) {
					if err := u.processFile(ctx, event.Name); err != nil {
						log.Printf("Error processing file %s: %v", event.Name, err)
					}
				}
			}

		case err := <-u.watcher.Errors:
			log.Printf("Watcher error: %v", err)

		case <-ticker.C:
			// Periodic scan for any missed files
			if err := u.scanAndUpload(ctx); err != nil {
				log.Printf("Error in periodic scan: %v", err)
			}
		}
	}
}

func (u *S3Uploader) processFile(ctx context.Context, filePath string) error {
	log.Printf("Processing file: %s", filePath)

	// Read metrics from JSON file
	metrics, err := u.readMetricsFromFile(filePath)
	if err != nil {
		return fmt.Errorf("failed to read metrics: %w", err)
	}

	if len(metrics) == 0 {
		log.Printf("No metrics found in file: %s", filePath)
		return nil
	}

	// Group metrics by partition
	partitionedMetrics := u.groupByPartition(metrics)

	// Convert to Parquet and upload each partition
	for partition, partitionMetrics := range partitionedMetrics {
		parquetFile, err := u.convertToParquet(partitionMetrics)
		if err != nil {
			return fmt.Errorf("failed to convert to Parquet: %w", err)
		}

		// Upload to S3
		s3Key := u.buildS3Key(partition, parquetFile)
		if err := u.uploadToS3(ctx, parquetFile, s3Key); err != nil {
			return fmt.Errorf("failed to upload to S3: %w", err)
		}

		// Clean up local parquet file
		os.Remove(parquetFile)
	}

	// Remove processed JSON file
	log.Printf("Removing processed file: %s", filePath)
	return os.Remove(filePath)
}

func (u *S3Uploader) readMetricsFromFile(filePath string) ([]MetricRecord, error) {
	file, err := os.Open(filePath)
	if err != nil {
		return nil, err
	}
	defer file.Close()

	var metrics []MetricRecord
	decoder := json.NewDecoder(file)

	for decoder.More() {
		var record MetricRecord
		if err := decoder.Decode(&record); err != nil {
			log.Printf("Error decoding record: %v", err)
			continue
		}

		// Extract partition values from timestamp
		record.Year = record.Timestamp.Year()
		record.Month = int(record.Timestamp.Month())
		record.Day = record.Timestamp.Day()
		record.Hour = record.Timestamp.Hour()

		// Extract data plane ID from labels
		if dpID, ok := record.Labels["data_plane_id"]; ok {
			record.DataPlaneID = dpID
		}

		metrics = append(metrics, record)
	}

	return metrics, nil
}

type Partition struct {
	Year        int
	Month       int
	Day         int
	Hour        int
	DataPlaneID string
}

func (u *S3Uploader) groupByPartition(metrics []MetricRecord) map[Partition][]MetricRecord {
	partitioned := make(map[Partition][]MetricRecord)

	for _, metric := range metrics {
		partition := Partition{
			Year:        metric.Year,
			Month:       metric.Month,
			Day:         metric.Day,
			Hour:        metric.Hour,
			DataPlaneID: metric.DataPlaneID,
		}
		partitioned[partition] = append(partitioned[partition], metric)
	}

	return partitioned
}

func (u *S3Uploader) convertToParquet(metrics []MetricRecord) (string, error) {
	// Simplified - in production, use parquet-go library
	// For now, we'll keep as JSON for the PoC

	timestamp := time.Now().Unix()
	filename := fmt.Sprintf("/tmp/metrics_%d.parquet", timestamp)

	// TODO: Implement actual Parquet conversion
	// For PoC, we'll write JSON temporarily
	file, err := os.Create(filename)
	if err != nil {
		return "", err
	}
	defer file.Close()

	encoder := json.NewEncoder(file)
	for _, metric := range metrics {
		if err := encoder.Encode(metric); err != nil {
			return "", err
		}
	}

	return filename, nil
}

func (u *S3Uploader) buildS3Key(partition Partition, localFile string) string {
	// S3 path: metrics/year=2024/month=01/day=15/hour=14/dataplane_id=dp-prod-01/metrics_timestamp.parquet
	return fmt.Sprintf("%s/year=%d/month=%02d/day=%02d/hour=%02d/dataplane_id=%s/%s",
		u.config.S3Prefix,
		partition.Year,
		partition.Month,
		partition.Day,
		partition.Hour,
		partition.DataPlaneID,
		filepath.Base(localFile),
	)
}

func (u *S3Uploader) uploadToS3(ctx context.Context, localFile, s3Key string) error {
	file, err := os.Open(localFile)
	if err != nil {
		return err
	}
	defer file.Close()

	log.Printf("Uploading to s3://%s/%s", u.config.S3Bucket, s3Key)

	_, err = u.s3Client.PutObject(ctx, &s3.PutObjectInput{
		Bucket: &u.config.S3Bucket,
		Key:    &s3Key,
		Body:   file,
	})

	if err != nil {
		return fmt.Errorf("failed to upload to S3: %w", err)
	}

	log.Printf("Successfully uploaded: %s", s3Key)
	return nil
}

func (u *S3Uploader) scanAndUpload(ctx context.Context) error {
	// Scan buffer directory for any unprocessed files
	files, err := filepath.Glob(filepath.Join(u.config.BufferDir, "*.json"))
	if err != nil {
		return err
	}

	for _, file := range files {
		if !isActiveFile(file) {
			if err := u.processFile(ctx, file); err != nil {
				log.Printf("Error processing file %s: %v", file, err)
			}
		}
	}

	return nil
}

func isActiveFile(filename string) bool {
	// Active file is "metrics.json", rotated files have timestamps
	return filepath.Base(filename) == "metrics.json"
}

func loadConfig() *Config {
	return &Config{
		BufferDir:      getEnv("BUFFER_DIR", "/buffer"),
		S3Bucket:       getEnv("S3_BUCKET", "otel-metrics-lake"),
		S3Prefix:       getEnv("S3_PREFIX", "metrics"),
		AWSRegion:      getEnv("AWS_REGION", "us-east-1"),
		UploadInterval: time.Duration(getEnvInt("UPLOAD_INTERVAL_SEC", 300)) * time.Second,
	}
}

func getEnv(key, defaultValue string) string {
	if value := os.Getenv(key); value != "" {
		return value
	}
	return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
	if value := os.Getenv(key); value != "" {
		var intVal int
		fmt.Sscanf(value, "%d", &intVal)
		return intVal
	}
	return defaultValue
}

func main() {
	cfg := loadConfig()

	uploader, err := NewS3Uploader(cfg)
	if err != nil {
		log.Fatalf("Failed to create uploader: %v", err)
	}

	ctx := context.Background()
	if err := uploader.Start(ctx); err != nil {
		log.Fatalf("Uploader error: %v", err)
	}
}
```

### 3.4 Create Dockerfile

**Create `analytics/s3-uploader/Dockerfile`:**
```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o s3-uploader .

FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

COPY --from=builder /app/s3-uploader .

CMD ["./s3-uploader"]
```

---

## Step 4: Create Iceberg Table Schema

### 4.1 Create Python Script for Iceberg Table

**Create `analytics/iceberg-setup/create_table.py`:**
```python
#!/usr/bin/env python3
"""
Create Iceberg table in AWS Glue catalog
"""

import boto3
from pyiceberg.catalog import load_catalog
from pyiceberg.schema import Schema
from pyiceberg.types import (
    NestedField,
    TimestampType,
    StringType,
    DoubleType,
    IntegerType,
    MapType,
)
from pyiceberg.partitioning import PartitionSpec, PartitionField
from pyiceberg.transforms import YearTransform, MonthTransform, DayTransform, HourTransform, IdentityTransform

def create_metrics_table():
    # Define schema
    schema = Schema(
        NestedField(1, "timestamp", TimestampType(), required=True),
        NestedField(2, "metric_name", StringType(), required=True),
        NestedField(3, "metric_value", DoubleType(), required=True),
        NestedField(4, "metric_type", StringType(), required=True),

        # Labels as map
        NestedField(5, "labels", MapType(
            key_id=6,
            key_type=StringType(),
            value_id=7,
            value_type=StringType(),
            value_required=False
        ), required=False),

        # Resource attributes
        NestedField(8, "environment", StringType(), required=True),
        NestedField(9, "data_plane_id", StringType(), required=True),
        NestedField(10, "service_name", StringType(), required=False),
        NestedField(11, "cluster_id", StringType(), required=False),
        NestedField(12, "region", StringType(), required=False),

        # Partition columns
        NestedField(13, "year", IntegerType(), required=True),
        NestedField(14, "month", IntegerType(), required=True),
        NestedField(15, "day", IntegerType(), required=True),
        NestedField(16, "hour", IntegerType(), required=True),
    )

    # Define partitioning
    partition_spec = PartitionSpec(
        PartitionField(source_id=13, field_id=1000, transform=IdentityTransform(), name="year"),
        PartitionField(source_id=14, field_id=1001, transform=IdentityTransform(), name="month"),
        PartitionField(source_id=15, field_id=1002, transform=IdentityTransform(), name="day"),
        PartitionField(source_id=16, field_id=1003, transform=IdentityTransform(), name="hour"),
        PartitionField(source_id=9, field_id=1004, transform=IdentityTransform(), name="data_plane_id"),
    )

    # Load catalog (AWS Glue)
    catalog = load_catalog(
        "glue",
        **{
            "type": "glue",
            "s3.endpoint": "https://s3.us-east-1.amazonaws.com",
            "glue.skip-archive": "true",
        }
    )

    # Create table
    table = catalog.create_table(
        identifier="otel_metrics_db.metrics",
        schema=schema,
        partition_spec=partition_spec,
        location="s3://otel-metrics-lake/metrics/",
        properties={
            "write.format.default": "parquet",
            "write.parquet.compression-codec": "snappy",
            "write.metadata.compression-codec": "gzip",
        }
    )

    print(f"Created Iceberg table: {table}")

if __name__ == "__main__":
    create_metrics_table()
```

**Create `requirements.txt`:**
```txt
pyiceberg[glue]==0.5.0
boto3==1.29.0
```

---

## Step 5: Update Docker Compose

### 5.1 Add S3 Uploader Service

**Update `docker-compose.yml` to include S3 uploader:**
```yaml
  # S3 Uploader for long-term storage
  s3-uploader-prod:
    build:
      context: ./analytics/s3-uploader
      dockerfile: Dockerfile
    image: otel-s3-uploader:latest
    container_name: s3-uploader-prod
    environment:
      - BUFFER_DIR=/buffer
      - S3_BUCKET=${S3_BUCKET_NAME}
      - S3_PREFIX=metrics
      - AWS_REGION=${AWS_REGION}
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - UPLOAD_INTERVAL_SEC=300
    volumes:
      - otel-buffer-prod:/buffer:ro
    networks:
      - otel-network
    restart: unless-stopped

volumes:
  otel-buffer-prod:
    driver: local
```

### 5.2 Update OTEL Collector

**Mount buffer volume in collector:**
```yaml
  otel-collector-prod:
    volumes:
      - ./dataplane/otel-collector/config-with-s3.yaml:/etc/otel-collector/config.yaml:ro
      - otel-buffer-prod:/buffer
```

---

## Step 6: Testing

### 6.1 Local Testing with LocalStack

**Install LocalStack:**
```bash
pip install localstack
```

**Start LocalStack:**
```bash
localstack start -d
```

**Create local S3 bucket:**
```bash
aws --endpoint-url=http://localhost:4566 s3 mb s3://otel-metrics-lake
```

**Test S3 uploader:**
```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export S3_BUCKET=otel-metrics-lake
export AWS_REGION=us-east-1

go run main.go
```

### 6.2 Generate Test Data

```bash
# Generate traffic to API
cd load-generator
python load_generator.py -t http://localhost:8081 -r 100 -d 600

# Wait for file rotation (100MB or 1 day)
# Check buffer directory
ls -lh dataplane/otel-collector/buffer/
```

### 6.3 Verify S3 Upload

```bash
# List S3 objects
aws s3 ls s3://otel-metrics-lake/metrics/ --recursive

# Expected structure:
# metrics/year=2024/month=01/day=15/hour=14/dataplane_id=dp-prod-01/metrics_*.parquet
```

### 6.4 Verify Iceberg Table

```python
from pyiceberg.catalog import load_catalog

catalog = load_catalog("glue")
table = catalog.load_table("otel_metrics_db.metrics")

# Show schema
print(table.schema())

# Show partitions
print(table.inspect.partitions())

# Count rows
df = table.scan().to_pandas()
print(f"Total rows: {len(df)}")
```

---

## Step 7: Monitoring and Observability

### 7.1 Add S3 Uploader Metrics

**Create `/metrics` endpoint in uploader:**
```go
import (
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	filesProcessed = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name: "s3_uploader_files_processed_total",
			Help: "Total number of files processed",
		},
		[]string{"status"},
	)

	bytesUploaded = prometheus.NewCounter(
		prometheus.CounterOpts{
			Name: "s3_uploader_bytes_uploaded_total",
			Help: "Total bytes uploaded to S3",
		},
	)
)

func init() {
	prometheus.MustRegister(filesProcessed)
	prometheus.MustRegister(bytesUploaded)
}

// In main():
http.Handle("/metrics", promhttp.Handler())
go http.ListenAndServe(":8080", nil)
```

### 7.2 Create Alerts

**Add to Prometheus rules:**
```yaml
- alert: S3UploaderDown
  expr: up{job="s3-uploader"} == 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "S3 uploader is down"

- alert: S3UploadFailing
  expr: rate(s3_uploader_files_processed_total{status="error"}[5m]) > 0
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "S3 uploads are failing"
```

---

## Expected Outcomes

✅ **S3 Integration Working**:
- OTEL Collector writing to local buffer
- S3 Uploader processing rotated files
- Data uploaded to S3 in partitioned structure

✅ **Iceberg Table Created**:
- Glue catalog contains metrics table
- Schema properly defined
- Partitioning configured

✅ **Data Queryable**:
- Can query Iceberg table via Python
- Partition pruning works
- Data properly structured

---

## Troubleshooting

**Issue: Files not uploading**
```bash
# Check S3 uploader logs
docker logs s3-uploader-prod

# Verify AWS credentials
docker exec s3-uploader-prod env | grep AWS

# Check buffer directory permissions
docker exec otel-collector-prod ls -la /buffer
```

**Issue: Large files accumulating**
```bash
# Reduce rotation size in OTEL config
# Change max_megabytes from 100 to 10

# Increase upload frequency
# Change UPLOAD_INTERVAL_SEC from 300 to 60
```

**Issue: Iceberg table not found**
```bash
# Verify Glue database exists
aws glue get-database --name otel_metrics_db

# Check table creation
aws glue get-table --database-name otel_metrics_db --name metrics
```

---

## Next Steps

Proceed to **Phase 10: ClickHouse Deployment** to set up the analytics query engine.
