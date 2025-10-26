# Long term metrics analytics

## Goals

Extend the PoC outlined in the implementaion plan for long term metrics analytics

## Plan

- Write OTEL metrics data to AWS S3 storage in iceberg table format
- partition scheme = year/month/date/dataplaneid
- Add a single node clickhouse in the control plane to query s3 data
- Add an analytics dashboard to the control plane web application to show long term metrics