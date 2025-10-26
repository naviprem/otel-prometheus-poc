# Learning Resources

This document provides curated resources for learning about OpenTelemetry, Prometheus, and observability concepts.

---

## OpenTelemetry (OTEL)

### Official Documentation

- **[OpenTelemetry Official Site](https://opentelemetry.io/)** - Main project website
- **[OpenTelemetry Go Documentation](https://opentelemetry.io/docs/languages/go/)** - Go-specific documentation
- **[OpenTelemetry Specification](https://opentelemetry.io/docs/specs/otel/)** - Technical specification
- **[Getting Started Guide](https://opentelemetry.io/docs/languages/go/getting-started/)** - Quick start for Go

### Go SDK

- **[OpenTelemetry Go GitHub](https://github.com/open-telemetry/opentelemetry-go)** - Source code repository
- **[Go SDK Reference](https://pkg.go.dev/go.opentelemetry.io/otel)** - Package documentation
- **[Go Examples](https://github.com/open-telemetry/opentelemetry-go/tree/main/example)** - Official examples
- **[Metrics API](https://pkg.go.dev/go.opentelemetry.io/otel/metric)** - Metrics API documentation

### Concepts

- **[Signals Overview](https://opentelemetry.io/docs/concepts/signals/)** - Traces, metrics, logs
- **[Metrics Concepts](https://opentelemetry.io/docs/concepts/signals/metrics/)** - Understanding metrics
- **[Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)** - Standard naming conventions
- **[Context Propagation](https://opentelemetry.io/docs/concepts/context-propagation/)** - Distributed tracing

### Exporters

- **[Prometheus Exporter](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/prometheus)** - Prometheus exporter docs
- **[OTLP Exporter](https://pkg.go.dev/go.opentelemetry.io/otel/exporters/otlp/otlpmetric)** - OTLP protocol exporter
- **[Available Exporters](https://opentelemetry.io/ecosystem/registry/?language=go&component=exporter)** - Registry of exporters

---

## Prometheus

### Official Documentation

- **[Prometheus Official Site](https://prometheus.io/)** - Main project website
- **[Getting Started](https://prometheus.io/docs/prometheus/latest/getting_started/)** - Prometheus quickstart
- **[Installation Guide](https://prometheus.io/docs/prometheus/latest/installation/)** - Installation instructions
- **[Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)** - Configuration reference

### Query Language (PromQL)

- **[Querying Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)** - PromQL fundamentals
- **[Operators](https://prometheus.io/docs/prometheus/latest/querying/operators/)** - PromQL operators
- **[Functions](https://prometheus.io/docs/prometheus/latest/querying/functions/)** - Built-in functions
- **[PromQL Cheat Sheet](https://promlabs.com/promql-cheat-sheet/)** - Quick reference guide
- **[PromQL for Humans](https://timber.io/blog/promql-for-humans/)** - Beginner-friendly guide

### Best Practices

- **[Naming Conventions](https://prometheus.io/docs/practices/naming/)** - Metric naming standards
- **[Instrumentation](https://prometheus.io/docs/practices/instrumentation/)** - How to instrument applications
- **[Histograms vs Summaries](https://prometheus.io/docs/practices/histograms/)** - When to use each
- **[Recording Rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)** - Precomputing queries
- **[Alerting Rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)** - Defining alerts

### Exposition Format

- **[Metric Types](https://prometheus.io/docs/concepts/metric_types/)** - Counter, gauge, histogram, summary
- **[Exposition Formats](https://prometheus.io/docs/instrumenting/exposition_formats/)** - Text and protobuf formats
- **[Writing Exporters](https://prometheus.io/docs/instrumenting/writing_exporters/)** - Creating custom exporters

---

## Go Programming

### HTTP Servers

- **[net/http Package](https://pkg.go.dev/net/http)** - Standard library HTTP
- **[Gorilla Mux](https://github.com/gorilla/mux)** - HTTP router used in this project
- **[HTTP Server Guide](https://golang.org/doc/articles/wiki/)** - Building web apps in Go

### Concurrency

- **[Goroutines](https://go.dev/tour/concurrency/1)** - Lightweight threads
- **[Channels](https://go.dev/tour/concurrency/2)** - Communication between goroutines
- **[Context Package](https://pkg.go.dev/context)** - Request-scoped values and cancellation

### Project Structure

- **[Standard Go Project Layout](https://github.com/golang-standards/project-layout)** - Community conventions
- **[Organizing Go Code](https://go.dev/blog/organizing-go-code)** - Official guidance
- **[Package Organization](https://rakyll.org/style-packages/)** - Best practices

---

## Docker & Containerization

### Docker

- **[Docker Get Started](https://docs.docker.com/get-started/)** - Docker basics
- **[Dockerfile Best Practices](https://docs.docker.com/develop/dev-best-practices/)** - Writing good Dockerfiles
- **[Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)** - Optimizing image size
- **[Docker Compose](https://docs.docker.com/compose/)** - Multi-container applications

### Go-Specific

- **[Containerizing Go Apps](https://docs.docker.com/language/golang/)** - Official Go guide
- **[Distroless Images](https://github.com/GoogleContainerTools/distroless)** - Minimal base images
- **[Go Alpine Images](https://hub.docker.com/_/golang)** - Official Go Docker images

---

## Observability Concepts

### General

- **[Observability 101](https://www.honeycomb.io/what-is-observability)** - Introduction to observability
- **[Three Pillars of Observability](https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/ch04.html)** - Logs, metrics, traces
- **[SLIs, SLOs, SLAs](https://sre.google/sre-book/service-level-objectives/)** - Service level concepts
- **[The Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/)** - Key metrics to monitor

### Metrics

- **[Understanding Metrics](https://www.datadoghq.com/knowledge-center/metrics/)** - Metrics fundamentals
- **[RED Method](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)** - Rate, Errors, Duration
- **[USE Method](http://www.brendangregg.com/usemethod.html)** - Utilization, Saturation, Errors
- **[Cardinality](https://www.robustperception.io/cardinality-is-key/)** - Managing label combinations

### SRE & Monitoring

- **[Google SRE Book](https://sre.google/sre-book/table-of-contents/)** - Free online book
- **[Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)** - Chapter from SRE book
- **[Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)** - Practical SLO alerting

---

## Tutorials & Guides

### OTEL + Prometheus

- **[OTEL Metrics Tutorial](https://opentelemetry.io/docs/languages/go/instrumentation/#metrics)** - Official tutorial
- **[Instrumenting a Go App](https://grafana.com/blog/2022/05/04/how-to-capture-go-metrics-with-the-opentelemetry-api/)** - Step-by-step guide
- **[OTEL Prometheus Example](https://github.com/open-telemetry/opentelemetry-go/tree/main/example/prometheus)** - Official example

### Prometheus

- **[Prometheus Up & Running](https://www.oreilly.com/library/view/prometheus-up/9781492034131/)** - O'Reilly book
- **[Robust Perception Blog](https://www.robustperception.io/blog)** - Prometheus expert blog
- **[Prometheus Workshop](https://training.promlabs.com/)** - Interactive training

### Practical Projects

- **[Building Microservices Observability](https://www.nginx.com/blog/microservices-reference-architecture-nginx-building-microservices/)** - Real-world example
- **[Go Microservices Monitoring](https://developer.ibm.com/tutorials/monitoring-go-microservices-with-prometheus/)** - IBM tutorial
- **[Production-Ready Metrics](https://blog.container-solutions.com/production-ready-metrics-with-prometheus-and-go)** - Best practices

---

## Tools & Utilities

### Visualization

- **[Grafana](https://grafana.com/docs/grafana/latest/)** - Dashboarding and visualization
- **[Grafana Dashboards](https://grafana.com/grafana/dashboards/)** - Pre-built dashboards
- **[PromLens](https://promlens.com/)** - PromQL query builder

### Testing & Load Generation

- **[k6](https://k6.io/)** - Modern load testing tool
- **[Locust](https://locust.io/)** - Python load testing
- **[hey](https://github.com/rakyll/hey)** - HTTP load generator
- **[wrk](https://github.com/wg/wrk)** - HTTP benchmarking

### Development

- **[PromQL VSCode Extension](https://marketplace.visualstudio.com/items?itemName=RoboticMind.prometheus-snippets)** - Syntax highlighting
- **[Prometheus CLI](https://github.com/prometheus/prometheus/tree/main/cmd/promtool)** - Validation and testing
- **[OTEL Collector](https://opentelemetry.io/docs/collector/)** - Telemetry data pipeline

---

## Video Resources

### Conferences

- **[PromCon](https://www.youtube.com/c/PrometheusIo)** - Official Prometheus conference
- **[KubeCon + CloudNativeCon](https://www.youtube.com/c/cloudnativefdn)** - CNCF conferences
- **[OpenTelemetry Community Meetings](https://www.youtube.com/channel/UCHZDBZTIfdy94xMjMKz-_MA)** - Project meetings

### Tutorials

- **[Prometheus Crash Course](https://www.youtube.com/watch?v=h4Sl21AKiDg)** - TechWorld with Nana
- **[OpenTelemetry 101](https://www.youtube.com/watch?v=r8UvWSX3KA8)** - CNCF webinar
- **[Go + Prometheus](https://www.youtube.com/watch?v=A8cW1LPcJqk)** - Practical tutorial

---

## Community & Support

### OTEL Community

- **[CNCF Slack #opentelemetry](https://cloud-native.slack.com/)** - Community chat
- **[GitHub Discussions](https://github.com/open-telemetry/opentelemetry-go/discussions)** - Q&A forum
- **[Community Meetings](https://github.com/open-telemetry/community#calendar)** - Join discussions

### Prometheus Community

- **[Prometheus Users Mailing List](https://groups.google.com/forum/#!forum/prometheus-users)** - Email list
- **[CNCF Slack #prometheus](https://slack.cncf.io/)** - Community chat
- **[GitHub Discussions](https://github.com/prometheus/prometheus/discussions)** - Q&A forum

### Go Community

- **[Go Forum](https://forum.golangbridge.org/)** - Community forum
- **[Gophers Slack](https://invite.slack.golangbridge.org/)** - Go community Slack
- **[r/golang](https://www.reddit.com/r/golang/)** - Reddit community

---

## Sample Projects & Examples

### Reference Implementations

- **[OTEL Demo](https://github.com/open-telemetry/opentelemetry-demo)** - Multi-language demo app
- **[Prometheus Examples](https://github.com/prometheus/client_golang/tree/main/examples)** - Official examples
- **[Go Microservices](https://github.com/GoogleCloudPlatform/microservices-demo)** - GCP microservices demo

### Real-World Projects

- **[Kubernetes](https://github.com/kubernetes/kubernetes)** - Uses Prometheus extensively
- **[Jaeger](https://github.com/jaegertracing/jaeger)** - Distributed tracing (OTEL compatible)
- **[Cortex](https://github.com/cortexproject/cortex)** - Scalable Prometheus

---

## Courses & Training

### Free

- **[Prometheus Training](https://training.promlabs.com/)** - PromLabs free course
- **[OpenTelemetry Bootcamp](https://opentelemetry.io/docs/demo/)** - Official demo and exercises
- **[Go by Example](https://gobyexample.com/)** - Go programming fundamentals

### Paid

- **[Prometheus & Grafana on Udemy](https://www.udemy.com/topic/prometheus/)** - Various courses
- **[Linux Foundation Training](https://training.linuxfoundation.org/training/observability-with-prometheus-and-grafana/)** - Official CNCF training
- **[Pluralsight Go Path](https://www.pluralsight.com/paths/go)** - Comprehensive Go learning

---

## Books

### Observability & Monitoring

- **"Prometheus: Up & Running"** by Brian Brazil (O'Reilly)
- **"Distributed Systems Observability"** by Cindy Sridharan (O'Reilly)
- **"Site Reliability Engineering"** by Google (Free online)
- **"Observability Engineering"** by Charity Majors et al. (O'Reilly)

### Go Programming

- **"The Go Programming Language"** by Donovan & Kernighan
- **"Concurrency in Go"** by Katherine Cox-Buday (O'Reilly)
- **"Cloud Native Go"** by Matthew Titmus (O'Reilly)

---

## Blogs & Articles

### Essential Reads

- **[Monitoring is a Pain](https://www.robustperception.io/monitoring-is-a-pain/)** - Why good monitoring matters
- **[Logs vs Metrics](https://blog.getambassador.io/logs-vs-metrics-a-false-dichotomy-22e337adb1c6)** - Understanding the difference
- **[Choosing Metric Types](https://prometheus.io/docs/tutorials/understanding_metric_types/)** - When to use each
- **[Cardinality Best Practices](https://grafana.com/blog/2022/02/15/what-are-cardinality-spikes-and-why-do-they-matter/)** - Managing label explosion

### Advanced Topics

- **[Prometheus Long-term Storage](https://prometheus.io/docs/prometheus/latest/storage/)** - Retention and remote storage
- **[High Availability Prometheus](https://prometheus.io/docs/introduction/faq/#can-prometheus-be-made-highly-available)** - HA deployment
- **[OTEL vs Prometheus](https://grafana.com/blog/2022/05/10/opentelemetry-vs.-prometheus-whats-the-difference/)** - Comparison

---

## Cheat Sheets

- **[PromQL Cheat Sheet](https://promlabs.com/promql-cheat-sheet/)** - Quick reference
- **[Go Cheat Sheet](https://github.com/a8m/golang-cheat-sheet)** - Go syntax reference
- **[Docker Cheat Sheet](https://docs.docker.com/get-started/docker_cheatsheet.pdf)** - Docker commands
- **[OTEL Go Cheat Sheet](https://github.com/open-telemetry/opentelemetry-go/blob/main/CHANGELOG.md)** - API changes

---

## Standards & Specifications

- **[OpenMetrics](https://openmetrics.io/)** - Metrics exposition standard
- **[OTLP Protocol](https://github.com/open-telemetry/opentelemetry-proto)** - OTEL wire protocol
- **[Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)** - Naming standards
- **[W3C Trace Context](https://www.w3.org/TR/trace-context/)** - Distributed tracing standard

---

## Extensions & Related Projects

### Complementary Tools

- **[Grafana](https://grafana.com/)** - Visualization and dashboarding
- **[Jaeger](https://www.jaegertracing.io/)** - Distributed tracing
- **[Loki](https://grafana.com/oss/loki/)** - Log aggregation
- **[Thanos](https://thanos.io/)** - Long-term Prometheus storage
- **[Cortex](https://cortexmetrics.io/)** - Scalable Prometheus

### Alert Management

- **[Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)** - Alert handling
- **[PagerDuty](https://www.pagerduty.com/)** - Incident management
- **[Opsgenie](https://www.atlassian.com/software/opsgenie)** - Alert aggregation

---

## Keeping Up-to-Date

### Release Notes

- **[OTEL Go Releases](https://github.com/open-telemetry/opentelemetry-go/releases)** - Latest changes
- **[Prometheus Releases](https://github.com/prometheus/prometheus/releases)** - Version updates
- **[Go Releases](https://go.dev/doc/devel/release)** - Go version history

### Newsletters

- **[CNCF Newsletter](https://www.cncf.io/newsroom/newsletter/)** - Cloud native updates
- **[Golang Weekly](https://golangweekly.com/)** - Go news and articles
- **[SRE Weekly](https://sreweekly.com/)** - SRE and operations

### Podcasts

- **[Cloud Native Computing Podcast](https://www.cncf.io/newsroom/podcast/)** - CNCF topics
- **[Go Time](https://changelog.com/gotime)** - Go programming
- **[Software Engineering Daily](https://softwareengineeringdaily.com/)** - General engineering

---

## Contributing Back

Once you've learned from this project, consider:

- Contributing to [OpenTelemetry](https://github.com/open-telemetry/community/blob/main/CONTRIBUTING.md)
- Contributing to [Prometheus](https://prometheus.io/community/)
- Sharing your learnings via blog posts
- Creating your own tutorials and examples
- Helping others in community forums

---

## Related Documentation

- [architecture.md](architecture.md) - Project architecture
- [step-by-step-guide.md](step-by-step-guide.md) - Implementation guide
- [metrics-guide.md](metrics-guide.md) - Prometheus queries
- [demo-walkthrough.md](demo-walkthrough.md) - Running the demo
