# observability-with-grafana-stack

A ready-to-run, single-host observability stack built around **Grafana Alloy** as the
collector and the Grafana **LGTM-ish** backends:

| Signal  | Backend     | Port (host)            |
|---------|-------------|------------------------|
| Metrics | Prometheus  | `9090`                 |
| Logs    | Loki        | `3100`                 |
| Traces  | Tempo       | `3200`, `4317`, `4318` |
| UI      | Grafana     | `3000`                 |
| Collector | Alloy     | `12345`, `4319`, `4320` |

Everything is wired together via Docker Compose and Grafana datasource provisioning,
so once it starts you have metrics ↔ logs ↔ traces correlation out of the box.

---

## Architecture

```
                ┌───────────────────────────────┐
                │            Grafana            │  :3000
                │ (anonymous Admin, provisioned │
                │  Prometheus / Loki / Tempo)   │
                └───────▲───────▲───────▲───────┘
                        │       │       │
              ┌─────────┘       │       └──────────┐
              │                 │                  │
        ┌─────┴──────┐    ┌─────┴─────┐      ┌─────┴─────┐
        │ Prometheus │    │   Loki    │      │   Tempo   │
        │  :9090     │    │  :3100    │      │  :3200    │
        │ remote_write│   │ push API  │      │ OTLP 4317 │
        └─────▲──────┘    └─────▲─────┘      └─────▲─────┘
              │                 │                  │
              │   metrics       │ logs             │ traces
              │                 │                  │
              │           ┌─────┴───────┐          │
              └───────────┤    Alloy    ├──────────┘
                          │   :12345    │
                          │ OTLP in:    │
                          │  4319/4320  │
                          └──────▲──────┘
                                 │
                       Docker socket / OTLP clients
```

- **Alloy** discovers running Docker containers, scrapes opt-in metrics, ships all
  container logs to Loki, and accepts OTLP traces (gRPC/HTTP) which it forwards to
  Tempo.
- **Prometheus** has `--web.enable-remote-write-receiver` and exemplar storage
  enabled so it can ingest Alloy’s `prometheus.remote_write` and link metrics →
  traces.
- **Tempo** runs with local file-system storage and exposes OTLP receivers.
- **Grafana** is provisioned with all three datasources plus correlations
  (`tracesToLogsV2`, derived fields, service map, exemplars).

---

## Repository layout

| File | Purpose |
|------|---------|
| [docker-compose.yaml](docker-compose.yaml) | Service definitions and port mappings |
| [config.alloy](config.alloy) | Alloy pipelines: Docker discovery, metrics scrape, Loki logs, OTLP→Tempo traces |
| [prometheus.yml](prometheus.yml) | Prometheus self-scrape + Alloy self-metrics |
| [loki-config.yaml](loki-config.yaml) | Loki single-binary config (TSDB v13, structured metadata, volume API) |
| [tempo.yaml](tempo.yaml) | Tempo config with OTLP gRPC/HTTP receivers and local storage |
| [provisioning/datasources/datasources.yaml](provisioning/datasources/datasources.yaml) | Grafana datasource provisioning + cross-signal correlation |

---

## Quick start

Requirements: Docker and Docker Compose v2.

```bash
docker compose up -d
docker compose ps
```

Then open Grafana: **http://localhost:3000** (anonymous Admin is enabled).

Verify the three datasources from **Connections → Data sources** — they should all
report *“working”*.

To stop and wipe state:

```bash
docker compose down -v
```

---

## Sending data into the stack

### Metrics (Prometheus scrape via Alloy)

Alloy only scrapes containers that **opt in** with a Docker label. Add this to any
service in your own compose file:

```yaml
services:
  my-app:
    image: my-app:latest
    labels:
      metrics: "true"   # tells Alloy to scrape this container
```

The container must expose a Prometheus-compatible `/metrics` endpoint on its
container port.

### Logs

Nothing to configure — Alloy tails **every** Docker container’s stdout/stderr via
the Docker socket and ships them to Loki with a `container=<name>` label.

### Traces (OTLP)

Point your application’s OTLP exporter at **Alloy** (preferred, gives you batching
and a single ingestion point):

| Protocol | Endpoint                       |
|----------|--------------------------------|
| OTLP gRPC | `http://localhost:4319`       |
| OTLP HTTP | `http://localhost:4320`       |

Or send directly to **Tempo** on `4317` / `4318` if you don’t need Alloy in the
middle.

Example with the OpenTelemetry SDK environment variables:

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4319
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_SERVICE_NAME=my-app
```

---

## Cross-signal correlation in Grafana

Provisioned out of the box:

- **Logs → Traces** — Loki derived field matches `traceID=<id>` in log lines and
  links to Tempo.
- **Traces → Logs** — Tempo `tracesToLogsV2` jumps to Loki filtered by
  `service.name` / `job` / `instance` / `container`.
- **Metrics → Traces** — Prometheus exemplars (`exemplarTraceIdDestinations`) link
  to Tempo.
- **Service map / node graph** — Tempo uses Prometheus for RED metrics.

---

## Useful URLs

| Service     | URL                              |
|-------------|----------------------------------|
| Grafana     | http://localhost:3000            |
| Prometheus  | http://localhost:9090            |
| Loki ready  | http://localhost:3100/ready      |
| Tempo ready | http://localhost:3200/ready      |
| Alloy UI    | http://localhost:12345           |

---

## Troubleshooting

- **No metrics from my container** — Did you add the `metrics: "true"` label and
  expose a `/metrics` endpoint on the container port?
- **`Prometheus` remote_write 404** — Make sure Prometheus is started with
  `--web.enable-remote-write-receiver` (already set in this repo).
- **No traces in Tempo** — Send to `localhost:4319` (Alloy gRPC) or `localhost:4317`
  (Tempo gRPC). Check Alloy’s UI at http://localhost:12345 → *Components* → the
  `otelcol.*` nodes should be healthy.
- **Loki “too old sample” / schema errors** — `docker compose down -v` to drop the
  Loki volume and start fresh (the schema starts at `2024-01-01`).
