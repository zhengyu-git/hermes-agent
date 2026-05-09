---
name: systemd-env-injection
description: Inject .env environment variables into systemd user services
triggers:
  - systemd service environment variables not loaded
  - bot works in CLI but not when deployed
  - EnvironmentFile directive in systemd
---

# Systemd Service Environment Injection

## Problem
Systemd services do NOT automatically source `.env` files. An interactive shell may load `.env` via `.bashrc` or profile scripts, but a systemd service only reads variables explicitly declared in its unit file or via `EnvironmentFile`.

**Key insight:** CLI shell environment ≠ systemd service environment. They are separate processes with separate environment blocks.

## Solution
Add `EnvironmentFile=/path/to/.env` to the `[Service]` section of the systemd unit file.

### Example
```ini
[Service]
Environment="HERMES_HOME=/home/user/.hermes"
EnvironmentFile=/home/user/.hermes/.env
ExecStart=/home/user/.hermes/venv/bin/python -m hermes_cli.main gateway run
WorkingDirectory=/home/user/.hermes/hermes-agent
Restart=on-failure
```

### Steps to apply
1. Find the service file: `~/.config/systemd/user/<service-name>.service`
2. Add `EnvironmentFile=/full/path/to/.env` under `[Service]`
3. Reload: `systemctl --user daemon-reload`
4. Restart: `systemctl --user restart <service-name>`
5. Verify: `cat /proc/$(pgrep <service>)/environ | tr '\0' '\n' | grep KEY_NAME`

## Hermes Gateway Specifics
The Hermes gateway (`hermes-gateway.service`) runs as a systemd user service. All environment variables needed by the bot (TAVILY_API_KEY, API keys, etc.) must be declared via `EnvironmentFile` — they will NOT be inherited from the CLI session even if the CLI has them set.

## Verification Commands
```bash
# Check what env vars a running service has
cat /proc/$(pgrep -f hermes-gateway)/environ | tr '\0' '\n' | grep -i tavily

# Check service status
systemctl --user status hermes-gateway

# View service log
journalctl --user -u hermes-gateway -f
