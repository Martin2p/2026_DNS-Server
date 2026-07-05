# 2026 — DNS Server

A self-hosted, recursive DNS resolver built with Unbound, secured behind a WireGuard VPN, and fully monitored with Prometheus and Grafana.

## Overview

This project deploys a private DNS resolver on an existing Contabo VPS (Ubuntu 24.04). It performs full recursive DNS resolution using root hints, validates DNSSEC signatures end-to-end, and is accessible exclusively through a WireGuard tunnel. All DNS query metrics are exported to Prometheus and visualized in Grafana.

**Use case:** Private DNS resolution for WireGuard-connected clients with caching, DNSSEC validation, and observability.

## Architecture

```
                        ┌──────────────────────────────────┐
                        │        Contabo VPS               │
                        │        Ubuntu 24.04              │
                        │                                  │
  WireGuard Clients ────┤  wg0 (10.100.0.1)                │
  (10.100.0.0/24)       │    │                             │
                        │    ├──► Unbound (Docker)         │
                        │    │      - Recursive resolver   │
                        │    │      - DNSSEC validation    │
                        │    │      - Query logging        │
                        │    │      - Root hints           │
                        │    │                             │
                        │    ├──► Unbound Prometheus       │
                        │    │      Exporter (Docker)      │
                        │    │                             │
                        │    ├──► Prometheus (existing)    │
                        │    └──► Grafana (existing)       │
                        │                                  │
                        │  nftables (inet meinefirewall)   │
                        │  Docker ("iptables": false)      │
                        └──────────────────────────────────┘
                                    │
                                    ▼
                        ┌──────────────────────────┐
                        │   Public Internet        │
                        │   Root DNS Servers       │
                        │   (Recursive Resolution) │
                        └──────────────────────────┘
```
## Components

| Component | Purpose | Notes |
|---|---|---|
| **Unbound** (`mvance/unbound`) | Recursive DNS resolver with caching | Docker container, bound to `10.100.0.1` |
| **DNSSEC** | End-to-end signature validation | Enabled in `unbound.conf` |
| **WireGuard** | VPN access control | Resolver only reachable via `wg0` (`10.100.0.0/24`) |
| **nftables** | Host-level firewall | Table `inet meinefirewall`, manual Docker rules (`"iptables": false`) |
| **Prometheus** | Metrics collection | Scrapes Unbound exporter + existing Node Exporter |
| **Grafana** | Visualization | DNS-specific dashboards (cache hit rate, query latency, etc.) |

## Security Model

- **No public exposure.** Unbound listens only on the WireGuard interface (`10.100.0.1`). There is no public DNS port.
- **Dual-layer enforcement.** Docker port binding to `10.100.0.1` + explicit nftables rules with `iifname "wg0"` as the allowed interface.
- **DNSSEC.** All responses are validated against the root trust anchor. Bogus responses are discarded.
- **Query logging.** Enabled for debugging and audit purposes.

## DNS Configuration

- **Resolution mode:** Recursive (root hints). Unbound resolves from the root zone down. No forwarding to third-party resolvers.
- **Zone type:** Caching-only. No local zones or internal name mappings.
- **Access control:** Restricted to `10.100.0.0/24` via `access-control` in `unbound.conf`.

## Monitoring

The Unbound Prometheus exporter exposes metrics including:

- Total queries (by type)
- Cache hits / misses
- Average and median query resolution time
- Queries forwarded vs. resolved locally
- DNSSEC validation outcomes (secure, insecure, bogus)

Metrics are scraped by the existing Prometheus instance and visualized in Grafana alongside system-level metrics from Node Exporter.

## Infrastructure Context

This project runs on the same Contabo VPS that hosts the ECN2026 infrastructure. The existing nftables configuration and Docker daemon settings (`"iptables": false`) apply — all container networking rules are managed manually.

## Documentation

Technical decisions are documented in ADR format (Context / Decision / Reasoning / Trade-offs):

```
docs/
├── README.md                  ← this file
├── architecture/
│   └── overview.md            ← detailed architecture description
├── decisions/
│   ├── recursive-vs-forwarding.md
│   ├── caching-only-no-local-zones.md
│   └── exporter-selection.md
├── configuration/
│   ├── unbound.conf           ← annotated config (placeholders for IPs)
│   ├── docker-compose.yml     ← service definitions
│   └── nftables-rules.md      ← DNS-specific firewall rules
└── lessons-learned/
    └── learned.md                    ← issues encountered and resolved
```
## Status

In development.

## Related Projects

- **ECN2026 — European Connected Network**: [github.com/Martin2p/2026_ECN2026-European-Connected-Network](https://github.com/Martin2p/2026_ECN2026-European-Connected-Network)

## License

MIT
