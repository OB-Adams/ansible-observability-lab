# OpenClaw Infrastructure Assistant

An AI-assisted Linux infrastructure operations project built on top of a self-hosted Ansible observability stack. OpenClaw acts as a conversational agent and tool orchestrator. It connects to a self-hosted observability MCP server to query real Prometheus metrics and Loki logs, then investigates infrastructure health using those tools — without modifying anything.

## What this demonstrates

- Deploying and automating a multi-host Linux observability stack with Ansible (Prometheus, Loki, Grafana, Alertmanager, Node Exporter, Grafana Alloy)
- Running a self-hosted AI agent (OpenClaw) against a local model (Qwen3 30B via Ollama)
- Connecting an AI agent to real infrastructure data through MCP tools
- Enforcing a hardened agent configuration: loopback-only gateway, token auth, tools denied except the approved MCP server, no web access
- Structured infrastructure investigation: the agent lists services, queries metrics and logs, and summarises findings — and explicitly states when a capability is unavailable rather than inventing results

This is a learning and portfolio project. It is not production-ready.

---

## Lab topology

| Host | IP | Role |
| --- | --- | --- |
| Ubuntu observability server | 192.168.0.30 | Prometheus, Loki, Grafana, Alertmanager (systemd services) |
| Rocky Linux node | 192.168.0.10 | Node Exporter, Grafana Alloy |
| CentOS node | 192.168.0.20 | Node Exporter, Grafana Alloy |
| OpenClaw VM | 192.168.0.40 | OpenClaw gateway, Ollama (model served from Mac via SSH tunnel) |
| Mac (M3 Pro) | — | Ansible control node, Ollama host (Qwen3 30B) |

All VMs run in UTM on an Apple Silicon Mac.

---

## Architecture

```
Mac (Ansible control)
  │  SSH
  ▼
ansible-admin account on all hosts
  │  deploys via site.yml
  ▼
┌──────────────────────────────────────────────────────┐
│  Ubuntu 192.168.0.30                                  │
│  Prometheus :9090  Loki :3100  Grafana :3000          │
│  Alertmanager :9093  Node Exporter :9100              │
│  Grafana Alloy (collects journald → Loki)             │
└──────────────────────────────────────────────────────┘
         ▲ scrapes Node Exporter             ▲ scrapes
         │                                   │
┌────────┴────────┐               ┌──────────┴──────────┐
│  Rocky :9100    │               │  CentOS :9100        │
│  Node Exporter  │               │  Node Exporter       │
│  Alloy          │               │  Alloy               │
└─────────────────┘               └──────────────────────┘

Mac (Ollama host, Qwen3:30b)
  │  SSH reverse tunnel :11434
  ▼
┌─────────────────────────────────────────────────────┐
│  OpenClaw VM 192.168.0.40                            │
│  OpenClaw gateway :18789 (loopback only)             │
│  SSH tunnels to 192.168.0.30: :9090, :3100           │
│                                                      │
│  You ──SSH tunnel──▶ Control UI :18789               │
│                           │  MCP streamable-HTTP     │
│                           ▼                          │
│              (observability-mcp) :3000/mcp                       │
│         (data sources: Prometheus :9090, Loki :3100)             │
└─────────────────────────────────────────────────────┘
```

### Trust boundaries

OpenClaw calls the observability MCP server (`@thotischner/observability-mcp`) running on the OpenClaw VM. The MCP server holds the Prometheus and Loki connections. The model never receives Prometheus or Loki credentials directly and never modifies infrastructure. The MCP server is read-only by design.

---

## Ansible stack

### Roles

| Role | What it deploys |
| --- | --- |
| `bootstrap` | ufw, base packages, SSH hardening |
| `journald` | persistent journal, size limits |
| `node_exporter` | Node Exporter 1.10.2 as a systemd service |
| `loki` | Loki 3.7.7 as a systemd service |
| `prometheus` | Prometheus 3.15.0, scrape config, alert rules |
| `alertmanager` | Alertmanager 0.28.1, Discord webhook |
| `grafana` | Grafana latest from vendor repo |
| `alloy` | Grafana Alloy, collects journald logs and pushes to Loki |
| `admin_user` | Non-root `ansible-admin` account with key-only SSH and validated sudoers |

### Running the stack

```bash
cd ansible
ansible-playbook site.yml
```

Re-running is idempotent. Release archives are checksum-verified. Service configs are validated in a candidate file before replacing the live one.

### Inventory

Three hosts in two groups:

```
observability → ubuntu (192.168.0.30)
monitored     → rocky  (192.168.0.10)
              → centos (192.168.0.20)
```

`observability_host` in `inventory/hosts.yml` drives Alloy's Loki push URL and Prometheus's remote scrape targets. No address is hardcoded in role templates.

### Secrets

A single Ansible Vault file at `ansible/inventory/group_vars/observability.yml` holds the Grafana admin password and Discord webhook URL. The vault password is never committed.

---

## OpenClaw setup

OpenClaw runs on the dedicated VM at 192.168.0.40 under a non-root `openclaw` user. The model is Qwen3 30B served by Ollama on the Mac and exposed to the VM through an SSH reverse tunnel.

Full setup steps are in [docs/openclaw-setup.md](docs/openclaw-setup.md).

### Hardened configuration

- Gateway binds to `127.0.0.1:18789` only
- Token authentication required
- Tool profile: `messaging` with web, filesystem, exec, runtime, and automation groups denied
- Elevated mode disabled
- No channels, no web search

### MCP integration

`@thotischner/observability-mcp` is a standalone MCP server that reads from Prometheus and Loki. It exposes its own UI on port 3000 and the MCP endpoint at `/mcp`. Register it in OpenClaw:

```json
"mcp": {
  "servers": {
    "observability": {
      "url": "http://127.0.0.1:3000/mcp",
      "transport": "streamable-http",
      "enabled": true,
      "connectionTimeoutMs": 5000,
      "requestTimeoutMs": 20000
    }
  }
}
```

The observability MCP's data sources (Prometheus at `http://localhost:9090`, Loki at `http://localhost:3100`) are reachable on the OpenClaw VM through SSH port forwards from the Mac:

```bash
ssh -N \
  -L 127.0.0.1:9090:127.0.0.1:9090 \
  -L 127.0.0.1:3100:127.0.0.1:3100 \
  openclaw@192.168.0.30
```

---

## Sample investigation

**Prompt sent to OpenClaw:**

> Use the observability MCP tools to investigate the current state of my infrastructure.
> 1. Check service health.
> 2. List the monitored services and available data sources.
> 3. Retrieve relevant metrics and logs for any unhealthy services.
> 4. Check for anomalies.
> 5. Summarise the findings and identify any potential incidents.
>
> Use the available MCP tools directly. Do not invent results if data is unavailable. Do not modify infrastructure or execute remediation.

**Tools called by the agent:**

1. `observability__list_services` — listed node, prometheus, journald
2. `observability__list_sources` — confirmed prometheus-source (up, 11ms) and Loki-source (up, 9ms)
3. `observability__query_metrics` — retrieved CPU metrics for the node service

**Result summary produced by the agent:**

- All monitored services and data sources up
- Node CPU stable (avg 20.2%, max 28.9% over last 15 minutes)
- No anomalies detected
- No anomaly detection tool available — explicitly noted, not fabricated
- No remediation executed

See `docs/screenshots/M2-openclaw-infrastructure-health-report.png`.

---

## Security notes and limitations

- **Prometheus and Loki have no authentication.** Access is controlled by ufw: port 9090 is not open to the LAN; 3100 is open but only reachable via the SSH tunnel in this setup.
- **The observability MCP exposes Prometheus and Loki query capability.** Unlike a purpose-built integration with a fixed allowlist, it can run arbitrary queries. This is a trade-off between simplicity and control.
- **Ollama has no authentication** on the Mac. The reverse tunnel keeps it off the LAN.
- **Ansible connects as `ansible-admin`** with passwordless sudo. Root SSH remains enabled as a fallback during the lab lifetime.
- **Host key checking is disabled** in `ansible.cfg`. Acceptable for a closed local lab; not for any shared or internet-facing environment.
- **OpenClaw runs with a hardened profile** but the model is 30B parameters. Smaller models are more susceptible to prompt injection through log content. The MCP endpoint does not enforce input sanitisation.
- **The OpenClaw VM and the model tunnel must both be running** for the assistant to work. Neither restarts automatically on reboot without further configuration.

---

## Tooling versions

| Tool | Version |
| --- | --- |
| Prometheus | 3.15.0 |
| Loki | 3.7.7 |
| Alertmanager | 0.28.1 |
| Node Exporter | 1.10.2 |
| OpenClaw | 2026.10.4 |
| Qwen3 (Ollama) | 30B |
| ansible-core | verified clean with ansible-lint production profile, 0 failures |

---

## Repository layout

```
ansible-observability-lab/
├── ansible/
│   ├── ansible.cfg
│   ├── site.yml
│   ├── inventory/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   ├── playbooks/
│   │   ├── admin_user.yml
│   │   └── ...
│   └── roles/
│       ├── admin_user/
│       ├── alertmanager/
│       ├── alloy/
│       ├── bootstrap/
│       ├── grafana/
│       ├── journald/
│       ├── loki/
│       ├── node_exporter/
│       └── prometheus/
├── docs/
│   ├── openclaw-setup.md
│   └── screenshots/
├── .gitignore
└── README.md
```

---

## Branches

| Branch | Contents |
| --- | --- |
| `main` | Original Ansible observability stack |
| `openclaw` | improvements (admin_user role, .gitignore, ansible-lint clean) + this documentation |
