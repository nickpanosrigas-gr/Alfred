# MCP-Alfred (`mcp-alfred`)

**MCP-Alfred** is the infrastructure observability Model Context Protocol (MCP) server for **Project Alfred**. It gives the Alfred agent topological awareness and deep diagnostic capabilities across Proxmox VE, LXC containers, Virtual Machines, TrueNAS storage, Docker stacks, Nginx reverse proxy routes, Cloudflare Argo tunnels, InfluxDB v2.8 metrics, Loki logs, and Home Assistant ambient server room telemetry.

Rather than exposing dozens of granular API wrappers that cause LLM "ping-pong latency" or a single monolithic tool that removes agent autonomy, `mcp-alfred` implements **Shallow-Graph Triage + Progressive Disclosure** across four unified tools and one static topology resource.

---

## 1. Architecture & Design Philosophy

### Core Principles
1. **Declarative Topology (`topology.yaml`):** The server requires zero hardcoded IPs, domains, or service names in Python. It reads a flat adjacency-list entity graph from the monorepo root (`Project-Alfred/topology.yaml`) and computes upstream dependencies, reverse dependencies (blast radiuses), and network ingress chains at startup.
2. **Unified Tool Signature:** Every tool accepts the exact same primary target identifier (`<type>.<host>[.<service>]`, e.g., `lxc.jellyfin` or `vm.docker.navidrome`), preventing LLM parameter hallucination.
3. **Multi-Hop Network Path Tracing:** When diagnosing a web service, the network tool tests the entire delivery pipeline concurrently:
   $$\text{Direct Backend IP:Port} \longrightarrow \text{Local LAN Domain (Nginx)} \longrightarrow \text{Public WAN Domain (Cloudflare Argo)}$$
   This immediately pinpoints whether a failure is an application crash, a local proxy misconfiguration, or an external tunnel/SSL issue.
4. **LLM-Optimized Context Formatting:** Raw time-series arrays from InfluxDB and repetitive stack traces from Loki are never dumped into the LLM context window. Python pre-processes metrics into statistical baselines + anomaly spikes, and deduplicates logs into frequency-ranked patterns.
5. **On-Demand Execution (`stdio`):** Managed via `uv`, the server launches on-demand as a subprocess when the Alfred agent needs infrastructure context and terminates cleanly when idle.

### Communication Matrix

| Target Layer | Technology | Protocol / Client Library | Auth Mechanism |
| :--- | :--- | :--- | :--- |
| **Hypervisor & LXCs** | Proxmox VE | REST API (`httpx` / `proxmoxer`) | Read-Only API Token (`PVEAuditor`) |
| **Storage Server** | TrueNAS | TrueNAS REST API v2.0 (`httpx`) | TrueNAS API Key |
| **Docker Host & Mounts** | Linux VM / LXC | SSH (`asyncssh`) | Dedicated SSH Key (`alfred-ro`) |
| **Reverse Proxy** | Nginx (`lxc.nginx`) | SSH / File Reader (`asyncssh`) | Dedicated SSH Key (`alfred-ro`) |
| **Edge Ingress** | Cloudflare Tunnel | REST API (`httpx`) / Local Metrics (`:20241`) | Cloudflare API Token (Tunnel:Read) |
| **Time-Series Metrics** | InfluxDB v2.8.0 | Flux REST API (`influxdb-client`) | Read-Only Bucket Token |
| **Log Aggregation** | Grafana Loki + Promtail | LogQL HTTP API (`httpx`) | Internal HTTP / Basic Auth |
| **Ambient Telemetry** | Home Assistant | REST API (`/api/states/{entity}`) | Long-Lived Access Token |

---

## 2. Monorepo Directory Structure

`mcp-alfred` lives inside the **Project Alfred** monorepo and resolves all configuration and secrets from the monorepo root directory:

```text
Project-Alfred/                        # Monorepo Root
├── .env                               # All API tokens, SSH key paths, and URLs
├── topology.yaml                      # Declarative infrastructure & dependency graph
├── alfred-agent/                      # Local AI Agent host application
└── mcp-alfred/                        # THIS SUBPROJECT
    ├── README.md
    ├── pyproject.toml                 # Managed by uv
    └── src/
        └── mcp_alfred/
            ├── __init__.py
            ├── server.py              # FastMCP server entrypoint, Resource & Tool registration
            ├── config.py              # Root path resolver, .env loader, Pydantic YAML parser
            ├── graph.py               # Adjacency graph builder & 1-hop dependency resolver
            ├── clients/
            │   ├── proxmox.py         # PVE node status, LXC .conf, cluster task history
            │   ├── truenas.py         # ZFS pool health, dataset usage, SMART disk temps
            │   ├── ssh_docker.py      # Docker inspect, compose reader (with secret redaction), stat mounts
            │   ├── nginx.py           # Virtual host parser, proxy_pass target verification, nginx -t check
            │   ├── cloudflare.py      # Tunnel health, connector status, and ingress rule inspector
            │   ├── influx.py          # InfluxDB v2.8 Flux queries & spike/anomaly detector
            │   ├── loki.py            # LogQL queries & regex log deduplicator
            │   └── homeassistant.py   # Ambient room temperature & smart plug wattage reader
            └── tools/
                ├── connectivity.py    # Tool 1: check_connectivity (Multi-hop network tracer)
                ├── sysinfo.py         # Tool 2: get_system_info
                ├── metrics.py         # Tool 3: get_metrics
                └── logs.py            # Tool 4: get_logs
```

---

## 3. MCP Capabilities (Resources & Tools)

### MCP Resource: `topology://homelab`
Exposes the parsed `topology.yaml` graph (including computed reverse dependencies, blast radiuses, and ingress routes) as a read-only JSON document. The agent reads this to understand what services exist and how they connect before invoking tools.

---

### Tool 1: `check_connectivity` (Multi-Hop Network Tracer & Health Check)
Performs a fast (< 2s) multi-layer reachability check on a target entity. For services exposing web interfaces, it tests the entire path from direct container backend to public domain, checks mount liveness, and performs a 1-hop health check on all upstream dependencies.

* **Arguments:**
  * `target` (`str`, required): Entity ID (e.g., `lxc.jellyfin`, `vm.docker.navidrome`, `vm.truenas`).
  * `include_deps` (`bool`, default: `True`): When `True`, pings all immediate dependencies in parallel.
* **What It Checks:**
  1. **Compute State:** Is the Proxmox VM/LXC or Docker container running?
  2. **Hop 1 - Direct Backend:** TCP/HTTP ping against the internal IP/port (e.g., `http://192.168.1.210:8096/health`). Differentiates between `Connection Refused` (service crashed/not listening) and `Timeout` (host down/firewall drop).
  3. **Hop 2 - Local LAN Ingress (Nginx):** Tests `https://<local_domain>` via local DNS. Checks local SSL certificate validity, days until expiration, and SNI matching.
  4. **Hop 3 - Public WAN Ingress (Cloudflare Argo Tunnel):** Tests `https://<public_domain>` via public DNS. Translates Cloudflare-specific error codes into plain English:
     * `502 Bad Gateway`: Cloudflared reached Nginx, but Nginx could not reach the backend.
     * `521 Web Server Is Down`: Nginx or local host dropped the connection.
     * `522 Connection Timed Out`: Routing failure between Cloudflared and Nginx origin.
     * `526 Invalid SSL Certificate`: SSL mismatch between Cloudflared and Nginx origin (e.g., missing `noTLSVerify` or untrusted cert).
     * `530 / Error 1033`: `lxc.cloudflared` daemon is completely disconnected from Cloudflare Edge.
  5. **Mount Liveness:** Runs a non-blocking `stat` (2s timeout) on configured NFS/SMB mount paths to catch frozen shares ("Stale file handle").
  6. **1-Hop Dependencies:** Summarizes each upstream dependency as `healthy`, `degraded`, or `unreachable`.

---

### Tool 2: `get_system_info` (Configuration, Runtime State & Ingress Routing)
Retrieves the static configuration specification alongside the live runtime state and proxy routing rules of the target.

* **Arguments:**
  * `target` (`str`, required): Entity ID.
  * `include_tasks` (`bool`, default: `True`): Includes recent Proxmox node task history (`vzdump` backups, snapshots, migrations).
* **What It Returns:**
  1. **Sanitized Configuration (`config_spec`):**
     * For `lxc.*` or `vm.*`: Fetches the Proxmox `.conf` via the PVE API (cores, memory, mount points, hardware passthrough like `/dev/dri`).
     * For `vm.docker.*`: Reads the `docker-compose.yml` service block via SSH and **automatically redacts sensitive keys** matching `(?i)(password|secret|token|key|api_key)`.
  2. **Nginx & Ingress Routing Verification (`clients/nginx.py` & `clients/cloudflare.py`):**
     * Parses the Nginx virtual host block matching the service domain.
     * **Config Drift Detection:** Verifies that Nginx's `proxy_pass` directive points to the current active IP and port from `topology.yaml`.
     * Validates that required headers (`Upgrade`, `Connection "upgrade"`, `client_max_body_size`) are present for WebSocket/media services.
     * Runs `nginx -t` to flag any broken syntax in the proxy container.
     * Inspects Cloudflare tunnel ingress rules to confirm external routing targets.
  3. **Live Runtime State (`runtime_state`):**
     * Uptime, `RestartCount`, `OOMKilled` boolean flag, `ExitCode`, and filesystem capacity (`df -h` on root and mounted volumes).
     * For `vm.truenas`: Queries TrueNAS API for ZFS Pool state (`ONLINE`, `DEGRADED`), scrub status, and pool fragmentation/capacity.
  4. **Hardware Passthrough Verification:** Checks that passthrough devices (e.g., `/dev/dri/renderD128`) are actually present inside the container.
  5. **Proxmox Tasks:** Last 5 hypervisor tasks affecting the VMID to immediately spot backup locks.

---

### Tool 3: `get_metrics` (InfluxDB v2.8 Telemetry & Ambient Thermals)
Queries InfluxDB v2.8 using Flux and formats resource utilization into LLM-digestible summaries and anomaly events.

* **Arguments:**
  * `target` (`str`, required): Entity ID.
  * `mode` (`Literal["standard", "historical"]`, default: `"standard"`):
    * `"standard"`: Analyzes the last **2 hours** (current values, 2h average, 2h peak).
    * `"historical"`: Analyzes the last **7 days** (7d baseline, P95, and discrete timestamped spike events where values exceeded $2\sigma$ or hard thresholds like >85% CPU, >90% RAM, >15% Disk I/O Wait).
* **Context-Aware Thermal & Power Logic (Home Assistant + Hardware):**
  * **On `node.*`:** Always includes CPU package temp, GPU temp, **Home Assistant Room Thermostat temp**, **$\Delta T$ (CPU minus Ambient)**, and **Home Assistant Smart Plug Wattage**.
  * **On `vm.truenas`:** Always includes Disk SMART temps (Min/Max/Avg via TrueNAS API) + Home Assistant Room Thermostat temp.
  * **On `lxc.*` and `vm.docker.*`:** Omits temperature data during normal operation to save context tokens. **Exception:** If the service uses hardware passthrough (e.g., GPU transcoding on `lxc.jellyfin`) and the host CPU/GPU exceeds thermal thresholds (>80°C), automatically injects a `THERMAL_THROTTLE_WARNING` with host and room temperatures.

---

### Tool 4: `get_logs` (Loki + Promtail Deduplicated Log Analyzer)
Queries Grafana Loki via LogQL and collapses repetitive log storms into concise patterns.

* **Arguments:**
  * `target` (`str`, required): Entity ID.
  * `mode` (`Literal["standard", "historical"]`, default: `"standard"`):
    * `"standard"`: Last **2 hours** of `error`, `warn`, and `critical` logs, grouped by normalized message signature.
    * `"historical"`: Last **7 days** summary of top recurring error signatures, fatal crash timestamps, and systemd/container restart events.
* **Deduplication Pipeline:**
  1. Queries Loki using the target's `loki_query` selector (defaulting to `{host="<service_name>"}` collected by Promtail).
  2. Normalizes log lines in Python by stripping variable timestamps, ephemeral ports, IP addresses, and UUIDs.
  3. Groups identical messages and outputs: `[Occurrences | First Seen | Last Seen] Representative Log Line`.

---

## 4. Configuration Specification

### 4.1 Root `.env` File (`Project-Alfred/.env`)
```ini
# Optional override if mcp-alfred is moved outside the monorepo
# ALFRED_ROOT=/abs/path/to/Project-Alfred

# Proxmox API (Token with PVEAuditor role)
PROXMOX_HOST=[https://192.168.1.10:8006](https://192.168.1.10:8006)
PROXMOX_USER=alfred@pve
PROXMOX_TOKEN_NAME=mcp
PROXMOX_TOKEN_VALUE=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
PROXMOX_VERIFY_SSL=false

# TrueNAS API v2.0
TRUENAS_URL=[http://192.168.1.100](http://192.168.1.100)
TRUENAS_API_KEY=1-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# InfluxDB v2.8.0
INFLUX_URL=[http://192.168.1.50:8086](http://192.168.1.50:8086)
INFLUX_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx==
INFLUX_ORG=homelab
INFLUX_BUCKET=proxmox

# Grafana Loki
LOKI_URL=[http://192.168.1.50:3100](http://192.168.1.50:3100)

# Home Assistant (Read-only ambient telemetry)
HA_URL=[http://192.168.1.60:8123](http://192.168.1.60:8123)
HA_TOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Cloudflare API (Zero Trust Tunnel Read permissions)
CLOUDFLARE_API_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
CLOUDFLARE_ACCOUNT_ID=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# SSH Key for Docker Host VM & Nginx LXC
SSH_USERNAME=alfred
SSH_KEY_PATH=/home/user/.ssh/alfred_ed25519
```

### 4.2 Root `topology.yaml` (`Project-Alfred/topology.yaml`)
```yaml
version: "1.0"
defaults:
  domain: "example.com"
  influx_bucket: "proxmox"

entities:
  node.pve1:
    type: proxmox_node
    ip: 192.168.1.10
    role: "Primary Proxmox Hypervisor"
    hardware:
      cpu_sensor: "coretemp-isa-0000"
      gpu_device: "/dev/dri/renderD128"
    environment_sensors:
      provider: home_assistant
      ambient_temp_entity: "sensor.server_room_thermostat_temperature"
      power_wattage_entity: "sensor.server_smart_plug_power"

  vm.truenas:
    type: proxmox_vm
    hosted_on: node.pve1
    vmid: 100
    ip: 192.168.1.100
    role: "TrueNAS ZFS Storage Appliance"
    storage_exports:
      media_nfs:
        protocol: nfs
        export_path: "/mnt/tank/media"
        zfs_pool: "tank"

  vm.docker:
    type: proxmox_vm
    hosted_on: node.pve1
    vmid: 110
    ip: 192.168.1.110
    role: "Linux Docker Host VM"

  lxc.nginx:
    type: proxmox_lxc
    hosted_on: node.pve1
    vmid: 201
    ip: 192.168.1.201
    role: "Nginx Reverse Proxy"
    nginx:
      config_dir: "/etc/nginx/sites-enabled"
      test_command: "sudo nginx -t"
    health_check:
      local_url: "[https://192.168.1.201:443](https://192.168.1.201:443)"

  lxc.cloudflared:
    type: proxmox_lxc
    hosted_on: node.pve1
    vmid: 202
    ip: 192.168.1.202
    role: "Cloudflare Argo Tunnel"
    cloudflare:
      tunnel_id: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
      metrics_url: "[http://192.168.1.202:20241/metrics](http://192.168.1.202:20241/metrics)"

  lxc.jellyfin:
    type: proxmox_lxc
    hosted_on: node.pve1
    vmid: 210
    ip: 192.168.1.210
    role: "Jellyfin Media Server"
    hardware_passthrough:
      - "/dev/dri/renderD128"
    network:
      backend_port: 8096
      backend_protocol: "http"
      health_path: "/health"
      expected_status: 200
      local_domain: "jellyfin.local.example.com"
      public_domain: "jellyfin.example.com"
      ingress_chain:
        - lxc.cloudflared
        - lxc.nginx
      requires_websockets: true
    telemetry:
      service_name: "jellyfin"
      loki_query: '{job="promtail", host="jellyfin"}'
    mounts:
      - provider: vm.truenas
        export: media_nfs
        mount_point: "/mnt/media"
        critical: true
    depends_on:
      - target: vm.truenas
        relationship: nfs_storage
        critical: true
      - target: lxc.nginx
        relationship: reverse_proxy
        critical: false
      - target: lxc.cloudflared
        relationship: argo_tunnel
        critical: false

  vm.docker.navidrome:
    type: docker_container
    hosted_on: vm.docker
    container_name: "navidrome"
    compose_path: "/opt/stacks/navidrome/docker-compose.yml"
    role: "Navidrome Music Streamer"
    network:
      backend_port: 4533
      backend_protocol: "http"
      health_path: "/ping"
      expected_status: 200
      local_domain: "music.local.example.com"
      public_domain: "music.example.com"
      ingress_chain:
        - lxc.cloudflared
        - lxc.nginx
    mounts:
      - provider: vm.truenas
        export: media_nfs
        mount_point: "/music"
        critical: true
    depends_on:
      - target: vm.docker
        relationship: compute_host
        critical: true
      - target: vm.truenas
        relationship: nfs_storage
        critical: true
      - target: lxc.nginx
        relationship: reverse_proxy
        critical: false
      - target: lxc.cloudflared
        relationship: argo_tunnel
        critical: false
```

---

## 5. Running & Testing

### Local Development with MCP Inspector
Use the MCP Inspector to interactively test tool executions, inspect multi-hop connectivity outputs, and verify your `.env` tokens:
```bash
cd Project-Alfred/mcp-alfred
uv run mcp dev src/mcp_alfred/server.py
```

### Agent Client Configuration (`stdio` On-Demand Spawn)
Configure the Alfred agent's MCP client to spawn `mcp-alfred` on demand using `uv`:
```json
{
  "mcpServers": {
    "alfred-infra": {
      "command": "uv",
      "args": [
        "run",
        "--directory",
        "/absolute/path/to/Project-Alfred/mcp-alfred",
        "mcp-alfred"
      ]
    }
  }
}
```