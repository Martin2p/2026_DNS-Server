# ADR-002: Caching-Only Resolver Without Local Zones

## Context

A recursive DNS resolver can serve two purposes:

1. **Caching-only**: Forward all queries to upstream servers, cache responses locally. No authority over any zones.
2. **Authoritative/Local Zones**: Serve DNS records for internal networks (e.g., `.home`, `internal.local`, custom hostnames).

The decision affects how local names resolve, whether split-horizon DNS is possible, and the operational complexity of managing zone files.

## Decision

Configure Unbound as a **caching-only forwarder** with no local zones. All DNS queries (root domain `"."`) are forwarded to upstream resolvers (Mullvad DNS, Quad9).

Clients in the WireGuard network (`10.100.0.0/24`) can only resolve public DNS names. Internal hostnames are not resolved by Unbound.

## Reasoning

- **Simplicity**: No zone files to maintain, no risk of DNS misconfigurations affecting local services
- **Privacy**: No internal hostname leakage — Unbound doesn't know about local devices
- **Consistency**: Same DNS view for all clients regardless of location (VPN vs. outside)
- **Security**: Attack surface reduced — no local zone attacks, no split-horizon confusion
- **Decoupling**: Local device discovery happens at the network layer (DHCP, mDNS) not DNS

### Configuration in `unbound.conf`

forward-zone: name: "." forward-tls-upstream: yes forward-addr: 194.242.2.9@853 # Mullvad DNS (Sweden) forward-addr: 194.242.2.3@853 forward-addr: 9.9.9.9@853 # Quad9 (Switzerland) forward-addr: 149.112.112.112@853


No `local-zone:` or `local-data:` directives are present.

## Trade-offs

| Pro | Con |
|-----|-----|
| Simple configuration, low maintenance | Cannot resolve local hostnames (e.g. `raspberrypi.local`) |
| Privacy-preserving (no internal name exposure) | Split-horizon DNS not supported |
| Consistent DNS view everywhere | Services must be discoverable via other means (mDNS, DHCP, static IPs) |
| Reduced attack surface | Users need external DNS names for internal services |

## Alternatives Considered

1. **Split-Horizon DNS** — Configure Unbound to resolve internal names locally (e.g., `*.internal`) while forwarding public queries. Provides convenience for accessing internal services but increases configuration complexity and potential security risks if internal names leak.

2. **Authoritative Zone for Home Network** — Set up Unbound as authoritative for a private domain (e.g., `home.lan`). Useful for organizing internal services but creates DNS silos that require manual management and potential client-side DNS overrides.

3. **Dedicated DNS for Internal Names (Pi-hole/AdGuard)** — Run a separate DNS server solely for internal names. Keeps concerns separated but adds another service to maintain.

4. **Hosts File per Client** — Static `/etc/hosts` entries on each machine. Works for small setups but becomes unmanageable as the fleet grows.

## When to Revisit This Decision

Consider switching to local zones if:
- Internal services become too numerous to track via static IPs
- Split-horizon DNS is required for production environments
- Team members need easy access to internal hostnames without manual setup

Until then, I keep it simple: no local zones, pure caching/forwarding.
