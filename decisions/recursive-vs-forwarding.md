# Recursive Resolution vs. Forwarding

## Status

Accepted

## Context

The DNS resolver needs to obtain answers for queries not served from cache. Two modes are available in Unbound:

- **Recursive resolution (root hints):** Unbound queries the root servers directly and walks the DNS tree (root → TLD → authoritative) to resolve each query.
- **Forwarding:** Unbound delegates uncached queries to a third-party upstream resolver (e.g. Quad9, Cloudflare) and acts as a caching layer in front.

The choice affects privacy, independence, performance characteristics, DNSSEC behavior, and the technical complexity demonstrated by the project.

## Decision

Use **recursive resolution with root hints**. No forwarding to third-party resolvers.

## Reasoning

- **Independence.** The resolver operates without dependency on any external DNS provider. No upstream can log, alter, or deny queries.
- **DNSSEC integrity.** While Unbound validates DNSSEC in both modes, recursive resolution ensures the full signature chain from root to authoritative is traversed and verified locally. In forwarding mode, the upstream could theoretically serve manipulated responses that pass through if the local validator trusts the transport.
- **Privacy.** No single third party sees all DNS queries originating from the WireGuard network. Queries go directly to the respective authoritative servers.
- **Portfolio value.** Demonstrating understanding of the full DNS resolution path (root hints, iterative queries, caching behavior, DNSSEC chain) is more technically substantive than configuring a forwarding address. This aligns with the project's goal of showcasing real-world DNS infrastructure skills.

## Trade-offs

| Aspect | Recursive (chosen) | Forwarding (rejected) |
|---|---|---|
| Cache miss latency | Higher — multiple round-trips to root/TLD/auth servers | Lower — single hop to upstream |
| Privacy | Queries reach authoritative servers directly | All queries visible to upstream provider |
| Provider dependency | None | Full dependency on upstream availability |
| DNSSEC | Full chain validated locally | Validated locally but response sourced from upstream |
| Complexity | Requires root hints maintenance | Simpler config, single forward address |
| First-query speed | Slower for uncached domains | Faster — upstream cache likely hit |

### Acceptable downsides

- **Cache miss latency** is negligible for a private resolver with few users. Once cached, subsequent queries are served at local network speed.
- **Root hints maintenance** is minimal — root hints change rarely and the `mvance/unbound` image ships with current hints that auto-update.
