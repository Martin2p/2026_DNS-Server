# Lessons Learned

## Docker Image Selection

**Problem**: The initial Unbound image (`mvance/unbound`) lacked essential tools like `unbound-control-setup` and shell commands (`ls`, `cat`), making debugging inside the container nearly impossible.

**Resolution**: Switched to the Klutchell image, which provides a more complete environment.

**Takeaway**: When selecting Docker images for infrastructure services, verify that essential diagnostic tools are available or ensure all configuration can be done externally via mounted volumes.

---

## nftables Manual Management

**Problem**: With Docker's iptables integration disabled (`"iptables": false`), all container traffic rules must be defined manually. Missing rules caused connectivity issues between WireGuard clients and the Unbound container.

**Resolution**: Explicitly defined forward chain rules for:
- WireGuard → Docker bridge (DNS requests)
- Docker bridge → WireGuard (DNS responses)
- Docker containers → Internet (recursive resolution)
- Internet → Docker (established/related responses)

**Takeaway**: Disabling Docker's iptables automation gives full control but requires careful rule planning. Document every rule with comments — future debugging depends on understanding why each rule exists.

---

## DNS Test Operation via WireGuard

**Problem**: DNS resolution needed to be verified exclusively through the WireGuard VPN interface to confirm the security model works as intended.

**Resolution**: Connected via WireGuard and tested DNS queries using `dig` and `nslookup` against `10.100.0.1:53`. Queries were correctly resolved and cached.

**Takeaway**: Always test through the intended access path (VPN), not just locally. Local testing bypasses firewall rules and can give false positives.

---

## systemd-resolved Conflict

**Problem**: Port 53 was already occupied by `systemd-resolved` on the host, preventing Unbound from binding to the DNS port.

**Resolution**: Disabled or reconfigured `systemd-resolved` to free up port 53.

**Takeaway**: Before deploying a DNS service, check for existing port conflicts:
bash sudo ss -tlnp | grep :53


---

## Prometheus Scrape Job Duplication

**Problem**: The `prometheus.yml` contained two scrape jobs targeting the same Node Exporter endpoint — `job_name: "node"` and `job_name: "node-exporter"`, both scraping `127.0.0.1:9100`. This caused doubled storage usage and inconsistent Grafana dashboard queries.

**Resolution**: Removed the duplicate `node` job and kept only `node-exporter`. Updated Grafana dashboard queries to use `job="node-exporter"`.

**Takeaway**: Review Prometheus configs for duplicates after iterative changes. Grafana dashboards silently break when job labels change — always verify dashboards after modifying scrape job names.

---

## Exporter Image Availability

**Problem**: The recommended Docker image `koton007/unbound-prometheus-exporter` did not exist on Docker Hub, causing `pull access denied` errors.

**Resolution**: Switched to `rsprta/unbound_exporter` after researching available alternatives.

**Takeaway**: Always verify Docker image existence before adding to `docker-compose.yml`. Cross-reference with Docker Hub and community recommendations.

---

## TLS Certificates for Remote-Control

**Problem**: Unbound's `remote-control` interface requires TLS certificates to operate. The Docker image lacked `unbound-control-setup`, and the interface silently failed to start on port 8953 without any obvious error in the logs.

**Resolution**: Generated TLS certificates on the host using OpenSSL and mounted them into the container.

**Takeaway**: Unbound's `remote-control` section will silently skip startup if certificates are missing — no error in logs. Always verify the port is open after configuration changes:
bash sudo netstat -tlnp | grep 8953


---

## Exporter Placement

**Problem**: Initial confusion about whether the exporter belongs in the Unbound stack or the Prometheus stack.

**Resolution**: The exporter is a monitoring tool and belongs in the Prometheus stack (`~/prometheus-docker/docker-compose.yml`), not the DNS stack. Both stacks use `network_mode: host`, so the exporter can reach Unbound via localhost.

**Takeaway**: Group services by function in separate `docker-compose.yml` files. Monitoring tools go with monitoring, DNS goes with DNS. This keeps concerns separated and reduces blast radius when restarting services.
