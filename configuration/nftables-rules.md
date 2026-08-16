# nftables Rules for DNS Server

## Overview

All firewall rules are managed manually via nftables (`inet meinefirewall`). 
Docker's iptables integration is disabled (`"iptables": false`), so all container traffic rules must be defined explicitly.

____

## Input Chain — DNS Access

#### Rule: DNS only via WireGuard VPN
nft iifname "wg0" udp dport 53 accept comment "Unbound DNS (VPN only)" iifname "wg0" tcp dport 53 accept comment "Unbound DNS TCP (VPN only)"
Unbound is not reachable from the public internet. Only clients connected via WireGuard (`10.100.0.0/24`) can send DNS queries.

____

## Forward Chain — Docker & VPN Routing

#### Rule: Docker internal DNS
```
ip saddr 172.16.0.0/12 udp dport 53 accept comment "Docker DNS (all networks)"
ip saddr 172.16.0.0/12 tcp dport 53 accept comment "Docker DNS (all networks)"
```

#### Rule: WireGuard → Unbound container
```
iifname "wg0" oifname "docker0" ip daddr 10.100.0.1 udp dport 53 accept comment "DNS request to container"
iifname "wg0" oifname "docker0" ip daddr 10.100.0.1 tcp dport 53 accept comment "DNS TCP request to Unbound container"
```
#### Rule: Unbound → response back to client
```
iifname "docker0" oifname "wg0" ip saddr 10.100.0.1 ct state established,related accept comment "DNS response from container"
```
#### Rule: Unbound outgoing recursive resolution
```
ip saddr 172.16.0.0/12 oifname "eth0" udp dport 53 accept comment "Container outgoing DNS"
oifname "eth0" tcp dport 53 accept comment "Container outgoing DNS TCP"
```
#### Rule: Docker containers → Internet (general)
```
ip saddr 172.16.0.0/12 oifname "eth0" accept comment "Docker container to internet"
oifname "eth0" ip daddr 172.16.0.0/12 ct state established,related accept comment "Internet response to Docker"
```
#### Rule: HTTP/HTTPS for Docker (Certbot etc.)
```
ip saddr 172.16.0.0/12 tcp dport { 80, 443 } accept comment "Docker HTTP/HTTPS outbound (incl. Certbot)"
```
____

## Notes

- `"iptables": false` in Docker daemon config means all container traffic filtering is manual
- The `established,related` state matching ensures only response traffic flows back — no unsolicited inbound
  
-> Port 53 is never exposed on `eth0` (public interface)
