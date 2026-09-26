# 🪿 Goose — Talk to Me, Goose

<p align="center">
  <strong>Lightweight Kafka-to-HTTP Firehose</strong><br>
  <em>"Talk to me, Goose." — Maverick, Top Gun (1986)</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Go-1.22+-00ADD8?logo=go" alt="Go">
  <img src="https://img.shields.io/badge/Docker-~26MB-2496ED?logo=docker" alt="Docker">
  <img src="https://img.shields.io/badge/License-Apache%202.0-blue.svg" alt="License">
  <img src="https://img.shields.io/badge/Tests-14%20E2E-brightgreen" alt="Tests">
</p>

---

## 🎯 What is Goose?

Goose is a **cloud-native Kafka consumer** that delivers streaming data to HTTP endpoints — fast, reliably, and with full observability.

Inspired by **Goose** from _Top Gun_ — the trusted RIO (Radar Intercept Officer) who feeds critical data to Maverick in real-time, never misses a callout, and always has his pilot's back.

Goose does the same for your services: it sits behind them, feeds data from Kafka, calls out threats (circuit breaker), and never lets a message drop.

## ⚡ Key Features

| Feature | Description |
|---------|-------------|
| 🚀 **Lightweight** | ~26MB binary, ~30MB RAM, <1s startup (vs 786MB Java) |
| 📊 **Observable** | 15 Prometheus metrics + OpenTelemetry traces + Loki logs |
| 🛡️ **Reliable** | At-least-once delivery with offset commits, DLQ, circuit breaker |
| 🔀 **Schema Support** | JSON passthrough + Protobuf→JSON via Stencil/schema registry |
| 🔍 **Filtering** | JSONPath (simple) + CEL expressions (JEXL-equivalent) |
| ✅ **Validation** | CEL-based schema validation with DLQ on failure |
| 🔄 **Error Handling** | Config-driven retry → DLQ → circuit breaker → never crash on HTTP errors |
| 🌐 **Connection TTL** | Fixes connection pinning — redirects traffic across backend pods |
| 📦 **K8s Ready** | Helm chart, HPA-ready, multi-stage Dockerfile (distroless) |

## 🏗️ Architecture

```
Kafka Topic
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Consumer (1 goroutine)                                     │
│  Poll → Schema Parse → Validate → Filter → Batch → Channel  │
└─────────────────────┬───────────────────────────────────────┘
                      │  chan *Batch (buffered, natural backpressure)
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  Worker Pool (N goroutines)                                 │
│  Circuit Breaker → HTTP POST → Retry → DLQ                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  Offset Manager (contiguity-gated commit to Kafka)          │
└─────────────────────────────────────────────────────────────┘

Parallel: Prometheus Metrics + OTel Traces + Loki Logs
```

## 📈 Performance

| Metric | Raystack (Java) | Goose (Go) | Improvement |
|--------|-----------------|------------|-------------|
| Image size | 786 MB | ~26 MB | 30x smaller |
| RAM at idle | ~200-300 MB | ~30 MB | 10x less |
| CPU at 2,000/s | ~2-3 vCPU | ~0.4 vCPU | 5x less |
| Startup | 5-10s | <1s | 10x faster |
| Dependencies | 50+ | 6 | 8x fewer |

**Load tested:** 2,500+ msg/s sustained on single pod with 100ms endpoint latency.

## 📦 Repositories

| Repo | Description |
|------|-------------|
| [goose](https://github.com/Goose-Kafka/goose) | The firehose service — Go binary, Helm chart, Dockerfile |
| [goose-integration-test](https://github.com/Goose-Kafka/goose-integration-test) | Integration test infra — Docker Compose, 14 E2E tests, K8s manifests, Grafana dashboards |

## 🧩 What's Inside

```
goose/
├── config/          ← Env-var config + validation + auto-match connections
├── consumer/        ← Kafka consumer goroutine + OTel spans
├── worker/          ← Worker pool (N goroutines, retry, DLQ, circuit breaker)
├── offset/          ← Contiguity-gated offset commit (at-least-once)
├── filter/          ← JSONPath + CEL filtering (JEXL-equivalent)
├── sink/            ← HTTP sink (batch/individual, connection TTL)
├── error/           ← Error routing, exponential backoff, circuit breaker, DLQ
├── schema/          ← JSON passthrough + Protobuf→JSON (DynamicMessage, long-polling refresh)
├── validation/      ← CEL-based schema validation
├── metrics/         ← 15 Prometheus metrics
├── tracing/         ← OpenTelemetry traces (OTLP → Jaeger/Tempo)
└── helm/            ← Helm chart for Kubernetes deployment
```

## 🚀 Quick Start

```bash
docker run -e SOURCE_KAFKA_BROKERS=kafka:9092 \
           -e SOURCE_KAFKA_TOPIC=events \
           -e SINK_HTTP_SERVICE_URL=http://service:8080/api \
           goose:latest
```

```bash
helm install goose ./helm \
  --set env.SOURCE_KAFKA_BROKERS=kafka:9092 \
  --set env.SINK_HTTP_SERVICE_URL=http://service:8080/api
```

## 🛣️ Roadmap

- ✅ Proto→JSON conversion (Stencil schema registry)
- ✅ Schema registry long-polling refresh
- ✅ CEL filtering (JEXL-equivalent)
- ✅ Schema validation (CEL-based)
- ✅ OTel spans in consumer and worker
- ✅ Consumer lag metrics
- ✅ Network error retry (zero message drops)
- 🔲 gRPC sink support
- 🔲 Batch-poll consumer (500x throughput boost)
- 🔲 HPA in Helm chart
- 🔲 Avro support

## ⚖️ Competitive Analysis

### Goose vs Other Kafka Delivery Tools

| Tool | Language | Type | Best For |
|------|----------|------|----------|
| **Goose** | Go | Standalone consumer | Kafka → HTTP, lightweight, fast |
| **Raystack Firehose** | Java | Standalone consumer | Kafka → 12+ sink types |
| **Kafka Connect** | Java | Framework (distributed) | Enterprise, 100+ connectors |
| **AWS Kinesis Firehose** | Managed | Cloud service | S3/Redshift/ES on AWS |
| **Apache Flink** | Java/Scala | Stream processor | Complex transforms, exactly-once |
| **Logstash** | JRuby | Pipeline tool | Kafka → Elasticsearch |

### Feature Comparison

| Feature | Goose | Raystack | Kafka Connect | AWS Firehose | Flink |
|---------|-------|----------|---------------|--------------|-------|
| Binary size | ~26MB | ~164MB | ~500MB+ | N/A | ~300MB+ |
| RAM at idle | ~30MB | ~200-300MB | ~500MB+ | N/A | ~1GB+ |
| Startup | <1s | 5-10s | 30-60s | N/A | 10-30s |
| Sinks | HTTP (extensible) | 12+ | 100+ | 6 (S3/Redshift/ES/etc) | Anything |
| Protobuf → JSON | ✅ | ✅ | ✅ | ❌ | ✅ |
| Schema registry | ✅ Stencil | ✅ Stencil | ✅ Confluent | ❌ | ✅ |
| Filtering | ✅ CEL + JSONPath | ✅ JEXL | ✅ SMTs | ✅ Lambda | ✅ Custom |
| Validation | ✅ CEL | ❌ | ❌ | ✅ Lambda | ✅ |
| Circuit breaker | ✅ | ❌ | ❌ | ❌ | ❌ |
| DLQ | ✅ Kafka | ✅ Kafka + Blob | ✅ Kafka | ✅ S3 | ✅ |
| Connection TTL | ✅ | ❌ (pinning bug) | ✅ | N/A | ✅ |
| Backpressure | ✅ Channels | ⚠️ Spin loop | ✅ | ✅ | ✅ |
| At-least-once | ✅ | ✅ | ✅ | ✅ | ✅ |
| Exactly-once | ❌ | ❌ | ❌ | ❌ | ✅ |
| OTel tracing | ✅ | ❌ | ❌ | ❌ | ❌ |
| Vendor lock-in | None | None | None | **AWS** | None |
| Dependencies | 6 | 50+ | 50+ | N/A | 50+ |

### Goose's Unique Advantages

- 🔌 **Connection TTL** — prevents pod pinning (Raystack has this bug)
- 🛡️ **Circuit breaker** — only tool with built-in circuit breaker
- 🔍 **CEL filtering** — Google-standard, more modern than JEXL
- ✅ **Schema validation** — validates messages before sending
- 🔄 **Network error retry** — zero message drops on pod restarts
- 📦 **6 dependencies** — vs 50+ in Java tools
- 📊 **OTel tracing** — only tool with built-in distributed tracing

### When to Choose Goose

| Scenario | Goose? | Alternative |
|----------|--------|-------------|
| Kafka → HTTP REST | ✅ **Best choice** | — |
| Kafka → HTTP with protobuf | ✅ **Best choice** | Raystack |
| Kafka → HTTP with complex filtering | ✅ **Best choice** | — |
| Minimal resource usage | ✅ **Best choice** | — |
| Kafka → multiple DBs/storage | ❌ | Raystack or Kafka Connect |
| Exactly-once semantics | ❌ | Apache Flink |
| Fully managed cloud | ❌ | AWS Kinesis Firehose |
| 100+ pre-built connectors | ❌ | Kafka Connect |
| Complex stream processing | ❌ | Apache Flink |

## 📜 License

Apache 2.0 — Built as a replacement for [raystack/firehose](https://github.com/raystack/firehose)
