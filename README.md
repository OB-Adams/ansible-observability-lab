# OpenClaw Infrastructure Assistant

An AI-assisted Linux infrastructure operations project built on a self-hosted Ansible observability stack.

OpenClaw acts as a conversational agent and tool orchestrator. It connects to a self-hosted observability MCP server to query Prometheus metrics and Loki logs, allowing it to investigate infrastructure health through real monitoring data without modifying infrastructure.

This project combines Linux administration, infrastructure automation, observability, Model Context Protocol (MCP), and AI agent security.

**Project status:** Learning and portfolio project. Not production-ready.

## What This Demonstrates

- Deploying and automating a multi-host Linux observability stack with Ansible.
- Operating Prometheus, Loki, Grafana, Alertmanager, Node Exporter, and Grafana Alloy.
- Running OpenClaw on a dedicated Linux VM with Qwen3 30B served by Ollama on a Mac.
- Connecting an AI agent to infrastructure monitoring tools through MCP.
- Restricting agent capabilities through a hardened OpenClaw configuration.
- Investigating infrastructure health using actual metrics and log sources.
- Validating service configurations before replacing live configurations.
- Verifying release archive checksums.
- Managing privileged automation through a dedicated Ansible account.
- Documenting security limitations and distinguishing verified findings from unavailable capabilities.

## Architecture

The project consists of three main components:

1. **Ansible observability infrastructure:** Collects host metrics and system logs.
2. **Observability MCP server:** Exposes monitoring data through tools that OpenClaw can call.
3. **OpenClaw assistant:** Uses the available tools to investigate infrastructure and summarise findings.

### Lab topology

| Host | IP address | Role |
| --- | --- | --- |
| Ubuntu observability server | `192.168.0.30` | Prometheus, Loki, Grafana, Alertmanager, Node Exporter, Grafana Alloy |
| Rocky Linux node | `192.168.0.10` | Node Exporter, Grafana Alloy |
| CentOS node | `192.168.0.20` | Node Exporter, Grafana Alloy |
| OpenClaw VM | `192.168.0.40` | OpenClaw gateway, observability MCP server, monitoring SSH tunnels |
| MacBook Pro M3 Pro | Local machine | Ansible control node, Ollama host running Qwen3 30B |

All Linux VMs run in UTM on an Apple Silicon Mac.

### Architecture diagram

```text
                       MacBook Pro M3 Pro
                  ┌──────────────────────────┐
                  │ Ansible control node     │
                  │ Ollama: Qwen3 30B        │
                  └────────────┬─────────────┘
                               │
                  SSH / Ansible│
                               ▼
                  ┌──────────────────────────┐
                  │ Linux infrastructure     │
                  │ ansible-admin account    │
                  └────────────┬─────────────┘
                               │
                               ▼
        ┌─────────────────────────────────────────────┐
        │ Ubuntu observability server                 │
        │ 192.168.0.30                                │
        │                                             │
        │ Prometheus :9090                            │
        │ Loki :3100                                  │
        │ Grafana :3000                               │
        │ Alertmanager :9093                          │
        │ Node Exporter :9100                         │
        │ Grafana Alloy                               │
        └───────────────┬─────────────────────────────┘
                        │
              Scrapes   │   Log collection
                        │
             ┌──────────┴───────────┐
             │                      │
             ▼                      ▼
    ┌──────────────────┐   ┌──────────────────┐
    │ Rocky Linux      │   │ CentOS           │
    │ 192.168.0.10     │   │ 192.168.0.20     │
    │                  │   │                  │
    │ Node Exporter    │   │ Node Exporter    │
    │ Grafana Alloy    │   │ Grafana Alloy    │
    └──────────────────┘   └──────────────────┘


                 Ollama on the Mac
                         │
                  SSH reverse tunnel
                         │
                         ▼
    ┌─────────────────────────────────────────────┐
    │ OpenClaw VM: 192.168.0.40                   │
    │                                             │
    │ OpenClaw gateway: 127.0.0.1:18789           │
    │ Observability MCP server: :3000/mcp         │
    │                                             │
    │ SSH tunnels to Prometheus and Loki          │
    │                                             │
    │ OpenClaw                                   │
    │    │                                        │
    │    │ MCP: streamable HTTP                   │
    │    ▼                                        │
    │ Observability MCP server                    │
    │    │                                        │
    │    ├── Prometheus queries                   │
    │    └── Loki queries                         │
    └─────────────────────────────────────────────┘
```

### Data flow

1. Ansible provisions and configures the Linux infrastructure.
2. Node Exporter exposes host metrics.
3. Prometheus scrapes the monitored hosts.
4. Grafana Alloy collects journald logs and pushes them to Loki.
5. The observability MCP server connects to Prometheus and Loki.
6. OpenClaw calls the MCP tools to retrieve monitoring information.
7. The model interprets the returned data and produces an investigation summary.

The model does not receive Prometheus or Loki credentials directly. It interacts with monitoring data through the MCP integration.

The integration is designed for read-only infrastructure investigation. It does not provide the agent with Ansible remediation tools.

## Ansible Observability Stack

The observability infrastructure is managed through Ansible roles and a central `site.yml` playbook.

### Roles

| Role | Responsibility |
| --- | --- |
| `bootstrap` | Base packages, hostname, `/etc/hosts`, and the host firewall (`ufw` or `firewalld`) with SSH allowed |
| `journald` | Persistent system journal and storage limits |
| `node_exporter` | Node Exporter 1.10.2 systemd service |
| `loki` | Loki 3.7.7 systemd service |
| `prometheus` | Prometheus 3.15.0, scrape configuration, and alert rules |
| `alertmanager` | Alertmanager 0.28.1 and Discord webhook configuration |
| `grafana` | Grafana installation from the vendor repository |
| `alloy` | Grafana Alloy installation and journald log forwarding |
| `admin_user` | Non-root `ansible-admin` account, key-based SSH access, and validated sudoers configuration |

### Running the stack

From the repository root:

```bash
cd ansible
ansible-playbook site.yml
```

The playbook is designed to be idempotent. Re-running it should converge the managed hosts toward the declared configuration without unnecessary changes.

Additional safeguards include:

- Checksum verification for downloaded release archives.
- Validation of candidate service configurations before replacing live configurations.
- Service handlers that apply configuration changes.
- A dedicated Ansible administration account.
- Ansible Vault for sensitive configuration values.

### Inventory

The inventory contains three observability hosts divided into two groups.

```text
observability
└── ubuntu    192.168.0.30

monitored
├── rocky     192.168.0.10
└── centos    192.168.0.20
```

The `observability_host` inventory variable drives the Loki push URL, and Prometheus scrape targets are generated from the inventory groups. Host addresses are not hardcoded in the role templates.

### Secrets management

The Ansible Vault file is located at:

```text
ansible/inventory/group_vars/observability.yml
```

It stores sensitive configuration values, including:

- Grafana administrator password.
- Discord webhook URL.

The Vault password is not committed to the repository.

## OpenClaw Setup

OpenClaw runs on a dedicated VM at `192.168.0.40` under a non-root `openclaw` user.

The language model is Qwen3 30B, served by Ollama on the MacBook Pro M3 Pro.

An SSH reverse tunnel exposes the Mac-hosted Ollama service to the OpenClaw VM without exposing the Ollama listener directly to the LAN.

The full setup procedure is documented in [`docs/openclaw-setup.md`](docs/openclaw-setup.md).

### Hardened configuration

The configuration restricts OpenClaw to the intended infrastructure investigation workflow.

| Setting | Configuration |
| --- | --- |
| Gateway address | `127.0.0.1:18789` |
| Authentication | Token authentication required |
| Tool profile | Messaging profile with general-purpose tools denied |
| Filesystem access | Denied through the configured tool policy |
| Command execution | Denied through the configured tool policy |
| Runtime and automation tools | Denied |
| Elevated mode | Disabled |
| Web search | Disabled |
| Communication channels | Not configured |

The observability MCP server is registered as the approved monitoring integration.

These restrictions reduce the agent's available capabilities. They do not, by themselves, prove that every possible attack path is eliminated.

## Observability MCP Integration

The project uses [`@thotischner/observability-mcp`](https://github.com/thotischner/observability-mcp), a standalone MCP server that exposes Prometheus and Loki query capabilities.

The server runs on the OpenClaw VM and provides:

- A web interface on port `3000`.
- An MCP endpoint at `/mcp`.
- Access to Prometheus metrics.
- Access to Loki logs.

### OpenClaw MCP configuration

The relevant server configuration is:

```json
{
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
}
```

The MCP server connects to the monitoring services through local endpoints on the OpenClaw VM.

### SSH tunnels to the monitoring services

The Prometheus and Loki endpoints on the observability server are forwarded to the OpenClaw VM through SSH tunnels.

The following command illustrates the local forwarding configuration when run on the OpenClaw VM:

```bash
ssh -N \
  -L 127.0.0.1:9090:127.0.0.1:9090 \
  -L 127.0.0.1:3100:127.0.0.1:3100 \
  openclaw@192.168.0.30
```

The tunnel connects as `openclaw`, a non-sudo account on the observability server used only for port forwarding. The OpenClaw VM does not hold the `ansible-admin` key, so a compromise of the agent's VM does not grant administrative access to the managed hosts.

The tunnel binds the forwarded ports to loopback on the OpenClaw VM, limiting access through those local listeners.

The Ollama reverse tunnel is a separate connection in the opposite direction, from the OpenClaw VM to the Mac-hosted model service.

## Infrastructure Investigation

OpenClaw can investigate infrastructure health by calling tools exposed by the observability MCP server.

The agent retrieves monitoring data and summarises its findings. It does not execute remediation.

### Investigation workflow

The investigation follows this sequence:

1. List monitored services and available data sources.
2. Check the health of the observability services.
3. Query relevant metrics and logs.
4. Look for anomalies when the available tools support the required analysis.
5. Summarise findings and disclose unavailable capabilities.
6. Avoid infrastructure modifications and remediation.

### Example investigation request

The following prompt was submitted to OpenClaw:

> Use the observability MCP tools to investigate the current state of my infrastructure.
>
> 1. Check service health.
> 2. List the monitored services and available data sources.
> 3. Retrieve relevant metrics and logs for any unhealthy services.
> 4. Check for anomalies.
> 5. Summarise the findings and identify any potential incidents.
>
> Use the available MCP tools directly. Do not invent results if data is unavailable. Do not modify infrastructure or execute remediation.

### Investigation request and tool interaction

The first screenshot shows the investigation request and the agent's interaction with the observability MCP integration.

![OpenClaw infrastructure investigation request and tool interaction](docs/screenshots/openclaw-infrastructure-health-report1.png)

### Investigation results

The second screenshot shows the response produced after the agent queried the available monitoring tools.

![OpenClaw infrastructure health report](docs/screenshots/openclaw-infrastructure-health-report2.png)

### Tools called

During the investigation, the agent called the following MCP tools:

| Tool | Purpose |
| --- | --- |
| `observability__list_services` | Listed the monitored services |
| `observability__list_sources` | Checked the available data sources |
| `observability__query_metrics` | Retrieved CPU metrics for the node service |

The agent reported the Prometheus and Loki data sources as available.

### Observed results

The investigation returned the following findings:

| Check | Result |
| --- | --- |
| Monitored services | Node, Prometheus, journald |
| Prometheus data source | Up, approximately 11 ms response time |
| Loki data source | Up, approximately 9 ms response time |
| Node CPU utilisation | 20.2% average, 28.9% maximum over 15 minutes |
| Anomaly detection | No dedicated anomaly-detection tool available |
| Remediation | None performed |

The agent reported no anomalies based on the available evidence.

However, the absence of a dedicated anomaly-detection tool means that this result should not be interpreted as proof that the infrastructure is free from anomalies. It is a summary of the observations returned by the available tools.

### What this demonstrates

This investigation demonstrates that OpenClaw can:

- Discover available monitoring services.
- Query a self-hosted observability integration.
- Retrieve metrics from the monitored infrastructure.
- Summarise findings using the returned data.
- Disclose unavailable capabilities instead of inventing results.
- Operate within a workflow that excludes remediation.

The results describe one investigation at a particular point in time. They are not a guarantee that the infrastructure is always healthy.

## Security Controls and Validation

The assistant is designed for read-only infrastructure investigation.

OpenClaw can query monitoring data through the approved MCP integration, but general-purpose system access and infrastructure modification are outside its intended role.

The security evidence is presented in two parts: restricted tool access and a security audit performed after hardening.

### Restricted tool access

The OpenClaw configuration denies general-purpose tools, including filesystem access, command execution, runtime control, and automation. Elevated mode is disabled.

The following screenshot documents the tested tool restrictions, including attempts to execute privileged commands and access files such as `/etc/passwd`.

![OpenClaw denied tool access](docs/screenshots/chat-tools-denied.png)

The screenshot demonstrates the restrictions observed during these interactions. It is not proof that every possible attack path has been eliminated.

The configured tool policy is one layer of the security model. The permissions and network access of the MCP server and other supporting components must also be considered.

### Security audit after hardening

The following screenshot records the OpenClaw security audit performed after disabling web search and restricting the agent's available tools.

![OpenClaw security audit after hardening](docs/screenshots/openclaw-security-audit.png)

The audit complements the denied-tool interaction by documenting the resulting security configuration.

### Security boundaries

| Control | Implementation |
| --- | --- |
| Gateway exposure | Bound to `127.0.0.1:18789` |
| Gateway authentication | Token authentication required |
| General-purpose tools | Restricted through the OpenClaw tool policy |
| Filesystem access | Denied through the configured tool policy |
| Command execution | Denied through the configured tool policy |
| Elevated mode | Disabled |
| Web search | Disabled |
| Infrastructure remediation | Not performed by the agent |
| Monitoring access | Through the observability MCP server |
| Prometheus and Loki access | Through SSH-forwarded endpoints, as the non-sudo `openclaw` user |

### Trust boundaries

The system has several distinct trust boundaries.

**OpenClaw and the language model**

The model decides which available tools to call and interprets the returned data. It is not given direct Prometheus or Loki credentials.

**Observability MCP server**

The MCP server connects to Prometheus and Loki and exposes their query capabilities to OpenClaw. It is read-only with respect to the infrastructure operations demonstrated in this project.

However, it can run arbitrary Prometheus and Loki queries rather than enforcing a fixed query allowlist. This is a trade-off between integration simplicity and tighter query-level control.

**Linux infrastructure**

Ansible manages the infrastructure separately from the OpenClaw investigation workflow. The agent does not have an Ansible remediation integration.

**Network access**

The gateway and forwarded monitoring endpoints use loopback listeners. SSH tunnels provide connectivity between the OpenClaw VM, the observability server, and the Mac-hosted model service.

### Security limitations

The following limitations remain in the current lab:

- Prometheus and Loki do not have authentication configured.
- Prometheus port `9090` is not opened in UFW, so it is reachable only from the host or through an SSH tunnel. Loki port `3100` is open to the LAN because the Alloy agents on every host push logs to it; the OpenClaw VM reaches it through the tunnel.
- The observability MCP server supports arbitrary monitoring queries rather than a fixed query allowlist.
- Ollama has no authentication configured on the Mac. The reverse tunnel keeps the model service off the LAN.
- Ansible connects as `ansible-admin` with passwordless sudo.
- Root SSH remains enabled as a fallback during the lab's lifetime.
- Host key checking is disabled in `ansible.cfg`. This is a lab convenience, not an appropriate default for shared or internet-facing infrastructure.
- Log content can contain untrusted input. A model may be influenced by malicious instructions embedded in logs, and the MCP integration does not enforce input sanitisation.
- The OpenClaw VM and model tunnel must both be running for the assistant to work.
- Neither the OpenClaw VM nor the model tunnel currently has complete automatic recovery configured for reboots.

These limitations are documented to make the trust boundaries explicit and identify areas for further engineering work.

The project is a learning and portfolio environment, not a production-ready infrastructure operations platform.

## Tooling Versions

| Tool | Version |
| --- | --- |
| Prometheus | 3.15.0 |
| Loki | 3.7.7 |
| Alertmanager | 0.28.1 |
| Node Exporter | 1.10.2 |
| OpenClaw | 2026.10.4 |
| Qwen3 via Ollama | 30B |
| Ansible Core | 2.21.x |

`ansible-lint` passes under the `production` profile with zero failures, and the GitHub Actions workflow enforces that profile on every push to `main`.

## Repository Structure

```text
ansible-observability-lab/
├── .github/workflows/ansible-ci.yml
├── ansible/
│   ├── ansible.cfg
│   ├── .ansible-lint
│   ├── requirements.yml
│   ├── site.yml
│   ├── inventory/
│   │   ├── hosts.yml
│   │   ├── host_vars/
│   │   └── group_vars/
│   │       ├── all.yml
│   │       └── observability.yml
│   ├── playbooks/            # one playbook per role, plus admin_user.yml
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
│       ├── openclaw-infrastructure-health-report1.png
│       ├── openclaw-infrastructure-health-report2.png
│       ├── chat-tools-denied.png
│       └── openclaw-security-audit.png
├── .gitignore
└── README.md
```

The repository also contains the remaining role files, templates, handlers, defaults, and configuration assets required by the playbooks.

## Branches

| Branch | Contents |
| --- | --- |
| `main` | The Ansible observability stack: checksum-verified downloads, validated configuration, the `ansible-admin` account, and CI with `ansible-lint` (production profile) |
| `openclaw` | Everything in `main`, plus the OpenClaw assistant setup, its documentation, and screenshots |

The OpenClaw work builds on the existing observability infrastructure rather than replacing it.

## Future Improvements

Potential improvements include:

- Add authentication or stronger access controls for Prometheus and Loki.
- Replace broad monitoring query access with a fixed query allowlist.
- Add dedicated anomaly detection with clearly defined thresholds and evaluation windows.
- Configure automatic startup and recovery for OpenClaw and its model tunnel.
- Enable SSH host key verification.
- Review the remaining root SSH fallback and passwordless sudo configuration.
- Test resilience against prompt injection through untrusted log content.
- Add human-approved remediation as a separate phase, with explicit approval gates and narrowly scoped Ansible privileges.

Any future remediation capability should remain separate from the current read-only investigation workflow.

## Summary

This project demonstrates an approach to AI-assisted infrastructure operations that combines Ansible automation, Linux observability, MCP tool integration, and restricted agent capabilities.

OpenClaw queries real Prometheus and Loki data through a self-hosted MCP server, summarises the results, and discloses when a requested capability is unavailable.

The project also documents the security boundaries and limitations of the implementation rather than treating tool restrictions as a complete security guarantee.

**The current scope is read-only infrastructure investigation. No automated remediation is performed.**
