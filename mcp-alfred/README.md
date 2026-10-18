# MCP-Alfred (`mcp-alfred`)

**MCP-Alfred** is the infrastructure observability Model Context Protocol (MCP) server for **Alfred**. It gives the Alfred agent topological awareness and deep diagnostic capabilities across Proxmox VE, LXC containers, Virtual Machines, TrueNAS storage, Docker stacks, InfluxDB v2.8 metrics, Loki logs, and Home Assistant ambient server room telemetry.

Rather than exposing dozens of granular API wrappers that cause LLM "ping-pong latency" or a single monolithic tool that removes agent autonomy, `mcp-alfred` implements **Shallow-Graph Triage + Progressive Disclosure** across four unified tools and one static topology resource.

---

## 1. Architecture & Design Philosophy

### Core Principles
1. **Declarative Topology (`topology.yaml`):** The server requires zero hardcoded IPs or service names in Python. It reads a flat adjacency-list entity graph from the monorepo root (`Alfred/topology.yaml`) and computes upstream dependencies and downstream blast radiuses at startup.
2. **Unified Tool Signature:** Every tool accepts the exact same primary target identifier (`<type>.<host>[.<service>]`, e.g., `lxc.jellyfin` or `vm.docker.navidrome`), preventing LLM parameter hallucination.
3. **LLM-Optimized Context Formatting:** Raw time-series arrays from InfluxDB and repetitive stack traces from Loki are never dumped into the LLM context window. Python pre-processes metrics into statistical baselines + anomaly spikes, and deduplicates logs into frequency-ranked patterns.
4. **On-Demand Execution (`stdio`):** Managed via `uv`, the server launches on-demand as a subprocess when the Alfred agent needs infrastructure context and terminates cleanly when idle.

### Communication Matrix

| Target Layer | Technology | Protocol / Client Library | Auth Mechanism |
| :--- | :--- | :--- | :--- |
| **Hypervisor & LXCs** | Proxmox VE | REST API (`httpx` / `proxmoxer`) | Read-Only API Token (`PVEAuditor`) |
| **Storage Server** | TrueNAS | TrueNAS REST API v2.0 (`httpx`) | TrueNAS API Key |
| **Docker Host & Mounts** | Linux VM / LXC | SSH (`asyncssh`) | Dedicated SSH Key (`alfred-ro`) |
| **Time-Series Metrics** | InfluxDB v2.8.0 | Flux REST API (`influxdb-client`) | Read-Only Bucket Token |
| **Log Aggregation** | Grafana Loki + Promtail | LogQL HTTP API (`httpx`) | Internal HTTP / Basic Auth |
| **Ambient Telemetry** | Home Assistant | REST API (`/api/states/{entity}`) | Long-Lived Access Token |

---

## 2. Monorepo Directory Structure

`mcp-alfred` lives inside the **Alfred** monorepo and resolves all configuration and secrets from the monorepo root directory:

```text
Alfred/                        # Monorepo Root
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
            │   ├── influx.py          # InfluxDB v2.8 Flux queries & spike/anomaly detector
            │   ├── loki.py            # LogQL queries & regex log deduplicator
            │   └── homeassistant.py   # Ambient room temperature & smart plug wattage reader
            └── tools/
                ├── connectivity.py    # Tool 1: check_connectivity
                ├── sysinfo.py         # Tool 2: get_system_info
                ├── metrics.py         # Tool 3: get_metrics
                └── logs.py            # Tool 4: get_logs
```

---

## 3. MCP Capabilities (Resources & Tools)

### MCP Resource: `topology://homelab`
Exposes the parsed `topology.yaml` graph (including computed reverse dependencies / blast radiuses) as a read-only JSON document. The agent reads this to understand what services exist and how they connect before invoking tools.

---

### Tool 1: `check_connectivity` (Fast Triage & Dependency Rollup)
Performs a fast (< 2s) multi-layer reachability check on a target entity and optionally performs a 1-hop shallow health check on all upstream dependencies defined in `topology.yaml`.

* **Arguments:**
  * `target` (`str`, required): Entity ID (e.g., `lxc.jellyfin`, `vm.docker.navidrome`, `vm.truenas`).
  * `include_deps` (`bool`, default: `True`): When `True`, pings all immediate dependencies in parallel.
* **What It Checks:**
  1. **Compute State:** Is the Proxmox VM/LXC or Docker container running?
  2. **Local Endpoint:** TCP/HTTP check against the internal IP/port (e.g., `http://192.168.1.210:8096/health`).
  3. **External Ingress:** DNS resolution, SSL certificate validity, and HTTP status code of the public Cloudflare/Nginx URL.
  4. **Mount Liveness:** Runs a non-blocking `stat` (2s timeout) on configured NFS/SMB mount paths to catch frozen shares ("Stale file handle").
  5. **1-Hop Dependencies:** Summarizes each upstream dependency as `healthy`, `degraded`, or `unreachable`.

---

### Tool 2: `get_system_info` (Configuration, Runtime State & Tasks)
Retrieves the static configuration specification alongside the live runtime state of the target.

* **Arguments:**
  * `target` (`str`, required): Entity ID.
  * `include_tasks` (`bool`, default: `True`): Includes recent Proxmox node task history (`vzdump` backups, snapshots, migrations).
* **What It Returns:**
  1. **Sanitized Configuration (`config_spec`):**
     * For `lxc.*` or `vm.*`: Fetches the Proxmox `.conf` via the PVE API (cores, memory, mount points, hardware passthrough like `/dev/dri`).
     * For `vm.docker.*`: Reads the `docker-compose.yml` service block via SSH and **automatically redacts sensitive keys** matching `(?i)(password|secret|token|key|api_key)`.
  2. **Live Runtime State (`runtime_state`):**
     * Uptime, `RestartCount`, `OOMKilled` boolean flag, `ExitCode`, and filesystem capacity (`df -h` on root and mounted volumes).
     * For `vm.truenas`: Queries TrueNAS API for ZFS Pool state (`ONLINE`, `DEGRADED`), scrub status, and pool fragmentation/capacity.
  3. **Hardware Passthrough Verification:** Checks that passthrough devices (e.g., `/dev/dri/renderD128`) are actually present inside the container.
  4. **Proxmox Tasks:** Last 5 hypervisor tasks affecting the VMID to immediately spot backup locks.

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

### 4.1 Root `.env` File (`Alfred/.env`)
```ini
# Optional override if mcp-alfred is moved outside the monorepo
# ALFRED_ROOT=/abs/path/to/Alfred

# Proxmox API (Create a token with PVEAuditor role)
PROXMOX_HOST=[https://192.168.1.10:8006](https://192.168.1.10:8006)
PROXMOX_USER=alfred@pve
PROXMOX_TOKEN_NAME=mcp
PROXMOX_TOKEN_VALUE=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
PROXMOX_VERIFY_SSL=false

# TrueNAS API
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

# SSH Key for Docker VM & Mount inspection
SSH_USERNAME=alfred
SSH_KEY_PATH=/home/user/.ssh/alfred_ed25519
```

### 4.2 Root `topology.yaml` (`Alfred/topology.yaml`)
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
    health_check:
      local_url: "[https://192.168.1.201:443](https://192.168.1.201:443)"

  lxc.cloudflared:
    type: proxmox_lxc
    hosted_on: node.pve1
    vmid: 202
    ip: 192.168.1.202
    role: "Cloudflare Argo Tunnel"

  lxc.jellyfin:
    type: proxmox_lxc
    hosted_on: node.pve1
    vmid: 210
    ip: 192.168.1.210
    role: "Jellyfin Media Server"
    hardware_passthrough:
      - "/dev/dri/renderD128"
    health_check:
      local_url: "[http://192.168.1.210:8096/health](http://192.168.1.210:8096/health)"
      external_url: "[https://jellyfin.example.com/health](https://jellyfin.example.com/health)"
      expected_status: 200
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
    health_check:
      local_url: "[http://192.168.1.110:4533/ping](http://192.168.1.110:4533/ping)"
      external_url: "[https://music.example.com/ping](https://music.example.com/ping)"
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
```

---

## 5. Implementation Guide (Step-by-Step)

### Step 1: Initialize Project with `uv`
From the `Alfred/mcp-alfred` directory:
```bash
uv init --package --name mcp-alfred
uv add "mcp[cli]" pydantic pyyaml python-dotenv httpx asyncssh influxdb-client
```

### Step 2: Root Path Resolution & Smart Defaults (`src/mcp_alfred/config.py`)
Always resolve `.env` and `topology.yaml` relative to the file tree (4 levels up from `config.py`) or via `ALFRED_ROOT`:
```python
import os
from pathlib import Path
import yaml
from dotenv import load_dotenv

DEFAULT_ROOT = Path(__file__).resolve().parents[3]
ALFRED_ROOT = Path(os.getenv("ALFRED_ROOT", DEFAULT_ROOT))

load_dotenv(ALFRED_ROOT / ".env")

def load_raw_topology() -> dict:
    topo_path = ALFRED_ROOT / "topology.yaml"
    with open(topo_path, "r", encoding="utf-8") as f:
        data = yaml.safe_load(f)
    
    # Apply Convention-over-Configuration Smart Defaults
    for entity_id, cfg in data.get("entities", {}).items():
        short_name = entity_id.split(".")[-1]
        telemetry = cfg.setdefault("telemetry", {})
        telemetry.setdefault("service_name", short_name)
        telemetry.setdefault("loki_query", f'{{host="{short_name}"}}')
    return data
```

### Step 3: InfluxDB v2.8 Flux Query Pattern (`src/mcp_alfred/clients/influx.py`)
Use parameterized Flux queries that compute aggregations server-side in InfluxDB v2.8 before returning to Python, keeping memory and latency low:
```flux
from(bucket: "proxmox")
  |> range(start: -2h)
  |> filter(fn: (r) => r["host"] == "jellyfin")
  |> filter(fn: (r) => r["_field"] == "cpu" or r["_field"] == "mem" or r["_field"] == "disk_iowait")
```
In Python, compute `mean`, `max`, `last`, and extract time windows where values exceed `mean + (2 * std_dev)`.

### Step 4: FastMCP Server Entrypoint (`src/mcp_alfred/server.py`)
```python
from typing import Literal
from mcp.server.fastmcp import FastMCP
from mcp_alfred.config import load_raw_topology

mcp = FastMCP("mcp-alfred")
TOPOLOGY = load_raw_topology()

@mcp.resource("topology://homelab")
def get_topology() -> dict:
    """Returns the complete homelab entity topology and dependency graph."""
    return TOPOLOGY

@mcp.tool()
async def check_connectivity(target: str, include_deps: bool = True) -> dict:
    """Fast reachability, mount liveness, and 1-hop dependency health check."""
    ...

@mcp.tool()
async def get_system_info(target: str, include_tasks: bool = True) -> dict:
    """Returns sanitized Proxmox/Docker config, runtime state, ZFS health, and recent tasks."""
    ...

@mcp.tool()
async def get_metrics(
    target: str,
    mode: Literal["standard", "historical"] = "standard"
) -> dict:
    """Queries InfluxDB v2.8 and Home Assistant sensors for resource metrics and anomaly spikes."""
    ...

@mcp.tool()
async def get_logs(
    target: str,
    mode: Literal["standard", "historical"] = "standard"
) -> dict:
    """Queries Loki (Promtail) and returns frequency-ranked, deduplicated error/warn patterns."""
    ...

def main():
    mcp.run(transport="stdio")

if __name__ == "__main__":
    main()
```

---

## 6. Running & Connecting to Alfred

### Local Development & Interactive Testing
Use the MCP Inspector with `uv` to test tool outputs and verify your `.env` credentials before connecting your agent:
```bash
cd Alfred/mcp-alfred
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
        "/absolute/path/to/Alfred/mcp-alfred",
        "mcp-alfred"
      ]
    }
  }
}
```