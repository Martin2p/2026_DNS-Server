# ADR-003: Exporter Selection for Unbound Metrics

## Context

The existing monitoring stack (Prometheus + Grafana + Node Exporter) on the VPS only covers system-level metrics (CPU, RAM, disk, network). There is no visibility into Unbound's DNS-specific behavior — cache performance, query volume, recursion latency, or DNSSEC validation results. Without this, DNS issues would only become apparent when clients complain.

## Decision

Use [`rsprta/unbound_exporter`](https://hub.docker.com/r/rsprta/unbound_exporter) as a Docker container, deployed within the existing Prometheus stack (`~/prometheus-docker/docker-compose.yml`).

The exporter connects to Unbound's `remote-control` interface (TLS-secured on port 8953) and translates Unbound's native statistics into Prometheus-compatible metrics.

## Reasoning

- **Same-stack deployment**: Runs alongside Prometheus, Grafana, and Node Exporter in the existing `docker-compose.yml` — no separate stack to manage
- **Minimal footprint**: ~8.8 MB Go binary, no additional runtime dependencies
- **Native Unbound stats**: Uses Unbound's built-in `remote-control` interface — no custom scripts or cron jobs needed
- **Prometheus-native format**: Metrics are exposed on port 9167 in Prometheus format — no converter required
- **Consistent networking**: All monitoring containers use `network_mode: host`, so the exporter reaches Unbound directly on the WireGuard interface IP

## Trade-offs

| Pro | Con |
|-----|-----|
| Simple Docker deployment, no extra stack | Remote-control interface requires TLS certificates |
| Uses Unbound's native stats interface | `mvance/unbound` image lacks `unbound-control-setup` — certs must be generated on the host |
| Lightweight Go binary | Image not actively maintained (last update 3+ years ago) |
| Good metrics coverage for Grafana dashboards | Metrics names are exporter-specific — not standardized |

## Alternatives Considered

1. **Node Exporter + textfile collector** — Write custom scripts that parse `unbound-control stats` output into Prometheus textfiles. More flexible but requires custom scripting, cron jobs, and ongoing maintenance.

2. **Built-in Unbound statistics port** — Unbound can expose JSON stats directly, but not in Prometheus format. Would still require a converter, so no advantage over a dedicated exporter.

3. **Switch to CoreDNS** — Has native Prometheus plugin built in. Rejected because the decision for Unbound was already made (see [recursive-vs-forwarding.md](./recursive-vs-forwarding.md)).

4. **`egguy/unbound-prometheus-exporter`** — Alternative Docker image. Not as widely used as `rsprta/unbound_exporter` and less documentation available.

## Implementation Notes

- Exporter is added to `~/prometheus-docker/docker-compose.yml`, not to the Unbound stack
- Unbound's `remote-control` interface must be enabled in `unbound.conf` with TLS certificates
- Prometheus scrape job configured on `localhost:9167` (since both run with `network_mode: host`)
