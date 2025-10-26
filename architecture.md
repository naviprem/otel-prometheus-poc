# OTEL Prometheus POC

## Goals

I want to build a PoC to demo a COTS (Commercial off the shelf) application with a control plane and multiple data planes all deployed in a customers cloud environment, with OTEL metrics dashboards available on the control plane based on the OTEL data generated in the data plane instances.

## Notes

- This is not a multi-tenant application. It is a COTS deployment which means both the control plane and data planes are all deployed directly in a customer's environment.
- Grafana's licensing does not allow for redistribution in commercial applications. Hence custom dashboards will be built as part of the control plan web application.
- In production, helm charts will be used for packaging and deployment. But for the PoC docker compose will be used for packaging and deployment
- In a single customer deployment, "multiple data planes" means, the customer will deploy one data plane per enviroment (dev, qa, uat and prod) or one data plane per line of business.
- For the PoC, Prometheus discover and scrape multiple data plane OTEL Collectors through static configuration. 

## Deployment Model

- **Type**: COTS - Full stack deployed in customer's cloud environment
- **Topology**: 1 Control Plane + N Data Planes per customer
- **Isolation**: Each customer gets dedicated infrastructure

## Data Plane Discovery

- Prometheus configured with static_configs
- Data plane collectors expose metrics on :8889/metrics
- Control plane aggregates metrics from all data planes

## Implementation plan

- The data planes have sample API endpoint implemented in golang for demo purposes
  - HTTP server exposing multiple endpoints in Port: 8080
  - Instrumented with OpenTelemetry SDK
  - Exports metrics via OTLP (OpenTelemetry Protocol)
  - Sends metrics to OTEL Collector on port 4318 (HTTP)
- The data planes have an OTEL Collector
  - Vendor-neutral telemetry data pipeline
  - **Receivers**: Accepts OTLP metrics from Go API
  - **Processors**: Batches and transforms metrics
  - **Exporters**: Exposes metrics in Prometheus format
  - Decouples application from monitoring backend
  - Ports: 4317 (OTLP gRPC), 4318 (OTLP HTTP), 8889 (Prometheus exporter)
- The control plane has a prometheus instance with thanos extension
  - Scrapes metrics from OTEL Collector every 10 seconds
  - Stores time-series data
  - Provides PromQL query interface
  - Web UI at http://localhost:9090
  - Prometheus is configured with Thanos extension for highly available, and long-term storage using AWS S3.
- Control Plane Web application 
  - Has a landing page and a metrics Dashboard
  - Built using React frontend
  - Simple timeseries charts using ECharts
- The PoC has a Load Generation Script 
  - Bash/Python script generating realistic traffic to the data plane API endpoints
  - Weighted distribution across endpoints
  - Configurable request rate and duration
  - Simulates various usage patterns
  - Error injection scenarios (simulate failures)

## Future Enhancements

- Service discovery for dynamic data plane registration
- Alertmanager for notification routing
- Example SLIs: p99 latency <500ms, error rate <1%
- Provide dashboard templates/presets in the React app
- Backup/restore procedures for metrics data?

## Non-Functional Requirements

  - Metrics retention: 30 days raw, 90 days downsampled
  - Query latency: <2s for dashboard queries
  - Data plane isolation: Complete metric isolation between data planes
  - Network: Control plane must reach data planes on port 8889