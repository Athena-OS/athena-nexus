# Athena Nexus

**Container Orchestrator for Security Professionals** built for [AthenaOS](https://athenaos.org) to deploy, manage, and monitor security tools running inside Docker or Podman containers.

![Athena Nexus](extra/screenshot.png)

## Overview

Athena Nexus is a Tauri desktop application (Rust backend + React frontend) that provides a unified interface for deploying and managing a curated registry of cybersecurity tools as containers. It supports both single-image tools and complex multi-service Docker Compose stacks, streaming real-time output and health status directly to the UI.

---

## Features

### Dashboard
- Live container cards with real-time CPU/RAM metrics, uptime ticker, and health badges
- Per-tool action buttons: Deploy, Start, Stop, Restart, Update, Delete
- All actions show the exact runtime command being executed (`docker stop abc123…`) and stream live output to a log drawer
- Compose stack containers shown with per-container status indicators
- Buttons automatically grey out during any in-progress operation

### Tool Registry
- Curated library of security tools deployable with a single click
- Pre-flight checks before every deploy (port conflicts, socket reachability, compose source availability)
- Live progress output with per-layer progress bars during image pulls (`docker pull` style)
- Full compose stack support: URL-based, file-based, and Git repo-based compose files
- Environment variable configuration UI for tools that require secrets or settings before deploy
- Update (pull latest + restart) and Delete (compose down + volume removal) from the registry view

### Pre-flight Checks
- 17-check system health scan grouped by category: System, Runtime, Network, Storage, Security, Deployed Tools
- Instantly reuses already-connected runtime info - no redundant socket pings
- Checks include: Docker/Podman socket, API version, rootless mode, systemd session, compose binary, disk space, DNS, network interfaces, and more

### Secrets Vault
- Encrypted key-value store backed by the system keyring
- Store API keys, tokens, and credentials used by deployed tools

### Snapshot & Backup
- Create, restore, export, and delete snapshots of container images
- Snapshots stored in `~/.local/share/athena-nexus/snapshots/`

### Audit Log
- Tamper-evident timestamped event log for all container actions
- Filterable by category, outcome, and date range
- Exportable to JSON

### Network Topology
- Visual graph of running containers and their network connections (D3-powered)

### User-Defined Tools
- Add custom tools (image-based or compose-based) alongside built-in registry tools
- Stored in `~/.config/athena-nexus/tools.json`

### Settings
- Switch between Docker and Podman runtimes without restarting
- Custom registry file path support
- Config export/import for portable setups

---

## Built-in Tool Registry

| Tool | Category | Type |
|---|---|---|
| Greenbone OpenVAS | Vulnerability | Compose |
| Nuclei | Vulnerability | Image |
| Wazuh SIEM | SIEM | Compose |
| MISP | Threat Intel | Compose |
| Zeek IDS | Network | Image |
| Suricata IDS/IPS | Network | Image |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Desktop shell | Tauri v2 |
| Backend | Rust (Tokio async runtime) |
| Frontend | React 18 + Vite |
| Container API | Direct Unix socket HTTP (no Docker SDK dependency) |
| Charts | Recharts, D3 |
| Icons | Lucide React |

---

## Requirements

- **Runtime**: Docker Engine ≥ v24 **or** Podman ≥ v4 with socket enabled
- **Compose**: `docker compose` plugin **or** `podman-compose`
- **webkit2gtk-4.1**
- **cmake** (for development builds)
- **npm** (for development builds)
- **pkg-config** (for development builds)
- **Rust** (for development builds)

### Enable Docker socket

```bash
sudo systemctl enable --now docker
sudo usermod -aG docker $USER   # then log out and back in
```

### Enable Podman socket

```bash
systemctl --user enable --now podman.socket
```

---

## Development

### Clone and install

```bash
git clone https://github.com/Athena-OS/athena-nexus
cd athena-nexus
npm install
```

### Run in development mode

```bash
npm run tauri dev
```

### Build for production

```bash
npm run tauri build
```

Bundles (`.deb`, `.rpm`, `AppImage`) are output to `src-tauri/target/release/bundle/`.

For getting only the binary, run:
```
npm run tauri build -- --no-bundle
```

---

## Configuration

All persistent data lives under standard XDG directories:

| Path | Contents |
|---|---|
| `~/.config/athena-nexus/config.json` | Runtime settings (Docker vs Podman, registry path) |
| `~/.config/athena-nexus/tools.json` | Tool registry (built-in + user-defined) |
| `~/.config/athena-nexus/vault.json` | Encrypted secrets |
| `~/.config/athena-nexus/audit.json` | Audit event log |
| `~/.config/athena-nexus/kv.json` | Internal key-value store |
| `~/.config/athena-nexus/snapshots.json` | Snapshot metadata |
| `~/.local/share/athena-nexus/snapshots/` | Snapshot tarballs |
| `/tmp/athena-nexus/{tool_id}/` | Compose file working directories (ephemeral) |

---

## Tool Registry Schema

Tools are defined in `tools.json` using a `source` / `access` structure.

**Single-image tool:**

```json
{
  "id": "nuclei",
  "name": "Nuclei",
  "category": "vulnerability",
  "description": "Fast and customizable vulnerability scanner.",
  "source": {
    "registry": "docker.io",
    "image": "projectdiscovery/nuclei",
    "version": "latest"
  },
  "tags": ["scanner", "cli"],
  "cli_tool": true
}
```

**Compose stack (Git repo):**

```json
{
  "id": "wazuh",
  "name": "Wazuh SIEM",
  "category": "siem",
  "description": "...",
  "source": {
    "compose_repo": "https://github.com/wazuh/wazuh-docker",
    "compose_repo_tag": "v4.14.3",
    "compose_subdir": "single-node",
    "pre_deploy": [
      { "type": "shell", "cmd": "mkdir -p config/wazuh_indexer_ssl_certs" },
      { "type": "compose", "file": "generate-indexer-certs.yml", "args": ["run", "--rm", "generator"] }
    ]
  },
  "access": {
    "entrypoint": "https://localhost:443",
    "health_check": "https://localhost:443",
    "ports": ["443:443", "1514:1514", "1515:1515", "55000:55000"]
  }
}
```

### `source` fields

| Field | Description |
|---|---|
| `image` | Image name, e.g. `projectdiscovery/nuclei` |
| `registry` | Registry prefix, e.g. `docker.io`, `ghcr.io` (default: `docker.io`) |
| `version` | Image tag (default: `latest`) |
| `compose_url` | Remote compose file URL |
| `compose_file` | Local compose file path |
| `compose_repo` | Git repo URL (downloaded as tarball - no git required) |
| `compose_repo_tag` | Branch or tag to download (default: `main`) |
| `compose_subdir` | Subdirectory inside the repo containing `docker-compose.yml` |
| `port_overrides` | Rewrite host-side ports in the compose file before deploying |
| `pre_deploy` | Steps to run before `compose up` (`shell` commands or `compose` runs) |

### `access` fields

| Field | Description |
|---|---|
| `entrypoint` | Web UI URL shown in the card, e.g. `https://localhost:{port}` |
| `health_check` | URL polled periodically to determine health status |
| `ports` | Host:container port mappings, e.g. `["443:443", "9392:9392"]` |

### `env_vars` (optional)

For tools that require configuration before deploying, define interactive fields shown in the Deploy Modal:

```json
"env_vars": [
  {
    "key": "ADMIN_PASSWORD",
    "label": "Admin Password",
    "description": "Password for the web console",
    "default": "",
    "required": true,
    "secret": true,
    "auto_uuid": false
  }
]
```
