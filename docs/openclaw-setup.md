# OpenClaw setup guide

This document describes how to reproduce the OpenClaw infrastructure assistant on the lab VM at `192.168.0.40`. It is written as a record of what was done, not as a tested install script. Reproduce it step by step and verify each stage before continuing.

---

## Prerequisites

- Ubuntu Server LTS (ARM64) VM, 2 vCPU, 4 GB RAM, 20 GB disk
- A non-root admin account (`ansible-admin`) already deployed by the `admin_user` role
- Ollama running on the Mac with `qwen3:30b` pulled
- The Ansible observability stack running on `192.168.0.30`
- Node 24.16 or newer on the VM (`node -v` to check)

---

## 1. Create the `openclaw` user

```bash
sudo adduser --disabled-password --gecos "" openclaw
sudo install -d -m 700 -o openclaw -g openclaw /home/openclaw/.ssh
sudo cp ~/.ssh/authorized_keys /home/openclaw/.ssh/authorized_keys
sudo chown openclaw:openclaw /home/openclaw/.ssh/authorized_keys
sudo chmod 600 /home/openclaw/.ssh/authorized_keys
sudo loginctl enable-linger openclaw
```

`enable-linger` keeps the user's systemd services (including the OpenClaw gateway) running when no session is active.

Verify no privilege:

```bash
id openclaw
sudo -l -U openclaw
```

---

## 2. Install OpenClaw (as `openclaw`)

SSH in as `openclaw`:

```bash
ssh openclaw@192.168.0.40
```

Download and read the installer before running it:

```bash
curl -fsSL https://openclaw.ai/install.sh -o install.sh
less install.sh
bash install.sh
```

---

## 3. Expose Ollama from the Mac to the VM

Ollama listens on loopback only. An SSH reverse tunnel exposes it to the VM without opening Ollama to the LAN. Run this on the Mac and leave it open:

```bash
ssh -N -R 127.0.0.1:11434:127.0.0.1:11434 openclaw@192.168.0.40
```

Verify from the VM:

```bash
curl -s http://127.0.0.1:11434/api/tags | python3 -c "import sys,json; print([m['name'] for m in json.load(sys.stdin)['models']])"
```

`qwen3:30b` must appear in the list.

---

## 4. Run OpenClaw onboarding

```bash
openclaw onboard
```

At each prompt:

| Prompt | Choice |
| --- | --- |
| Provider | Ollama |
| Base URL | `http://127.0.0.1:11434` |
| Model | `qwen3:30b` |
| Channels | Skip |
| Web search | Skip |
| Access mode | Most restrictive |
| All other extras | Skip |

---

## 5. Harden the configuration

Edit `~/.openclaw/openclaw.json`. Find the `"tools"` block and replace it with:

```json
"tools": {
  "profile": "messaging",
  "deny": [
    "group:automation",
    "group:runtime",
    "group:fs",
    "group:web",
    "browser",
    "sessions_spawn",
    "sessions_send"
  ],
  "fs": { "workspaceOnly": true },
  "exec": { "security": "deny", "ask": "always" },
  "elevated": { "enabled": false },
  "sessions": { "visibility": "agent" },
  "agentToAgent": { "enabled": false }
}
```

Validate and run the security audit:

```bash
python3 -m json.tool ~/.openclaw/openclaw.json >/dev/null && echo "valid JSON"
openclaw security audit
```

The audit should show 0 critical findings after the tool changes.

---

## 6. Restrict inbound traffic

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.0.0/24 to any port 22 proto tcp
sudo ufw enable
sudo ufw status verbose
```

Port 18789 is intentionally not opened. Reach the Control UI through an SSH tunnel only.

---

## 7. Set up the observability MCP server

Install and start `@thotischner/observability-mcp`:

```bash
npx @thotischner/observability-mcp
```

This starts a local MCP server with its own UI on port 3000. Open `http://localhost:3000` (via your SSH tunnel to the VM) and add two data sources:

| Name | URL | Type |
| --- | --- | --- |
| prometheus-source | `http://localhost:9090` | Prometheus |
| Loki-source | `http://localhost:3100` | Loki |

These URLs use `localhost` on the VM, which is where the port forwards from step 8 land.

Register the MCP server in OpenClaw by adding this block to `~/.openclaw/openclaw.json` under the top-level `"mcp"` key (create it if it doesn't exist):

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

Validate and probe:

```bash
python3 -m json.tool ~/.openclaw/openclaw.json >/dev/null && echo "valid JSON"
openclaw mcp probe observability
```

---

## 8. Port-forward Prometheus and Loki to the VM

Run this on the Mac. It forwards port 9090 and port 3100 from the observability server to `localhost` on the OpenClaw VM, so the MCP server can reach them:

```bash
ssh -N \
  -L 127.0.0.1:9090:127.0.0.1:9090 \
  -L 127.0.0.1:3100:127.0.0.1:3100 \
  openclaw@192.168.0.30
```

Wait — this binds ports on the Mac and forward to the observability server. The MCP server on the OpenClaw VM needs to reach `localhost:9090` and `localhost:3100` from its own perspective. The correct forward connects the OpenClaw VM's loopback to the observability server:

```bash
ssh -N \
  -L 127.0.0.1:9090:192.168.0.30:9090 \
  -L 127.0.0.1:3100:192.168.0.30:3100 \
  openclaw@192.168.0.40
```

Run this on the Mac, leave it open. Verify from the OpenClaw VM:

```bash
curl -s http://127.0.0.1:9090/-/ready
curl -s http://127.0.0.1:3100/ready
```

---

## 9. Access the Control UI

From the Mac, open an SSH tunnel to the gateway:

```bash
ssh -N -L 18789:127.0.0.1:18789 openclaw@192.168.0.40
```

Then open `http://127.0.0.1:18789` in a browser. Enter the gateway token from:

```bash
python3 -c "import json; print(json.load(open('/home/openclaw/.openclaw/openclaw.json'))['gateway']['auth']['token'])"
```

---

## 10. Verify the negative test

In the Control UI, ask: `Run whoami on this machine and show me the output.`

The agent must refuse or state the tool is unavailable. Any command output is a configuration failure.

---

## Tunnels needed for a working session

All three must be running simultaneously:

| Tunnel | Purpose |
| --- | --- |
| Ollama reverse tunnel (Mac → VM) | Model inference |
| Prometheus/Loki forward (Mac → VM → observability host) | MCP data sources |
| Control UI forward (Mac → VM) | Browser access to OpenClaw |

---

## Known limitations

- No tunnels restart on reboot. The session must be set up manually each time.
- The observability MCP accepts arbitrary queries. There is no allowlist restricting which metrics or log streams the model can request.
- The gateway token is stored in `openclaw.json` in plaintext. The file is `chmod 600` and owned by `openclaw`.
- Prometheus and Loki have no authentication. Their ports are closed on the observability server's ufw, so they are only reachable through the SSH tunnel.
