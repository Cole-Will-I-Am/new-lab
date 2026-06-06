# VPS Bootstrap Notes

Last updated: 2026-06-06 UTC

This repository records the current state and setup decisions for the experimental AI VPS. It is intentionally documentation-only at the moment.

## Host Summary

- Hostname: `srv1736822`
- OS: Ubuntu 24.04.4 LTS Noble
- Kernel observed: `6.8.0-111-generic`
- Virtualization: KVM/QEMU
- CPU: 1 vCPU on AMD EPYC 7543P host
- RAM: 3.8 GiB
- Swap: none
- Root disk: 48 GiB, roughly 5 GiB used at initial review
- Public IPv4: `153.92.222.46`
- Public IPv6: `2a02:4780:d:bdd6::1/48`
- Timezone: UTC
- NTP: active

## Initial Running Services

Running services observed during review:

- `ssh.service`
- `ollama.service`
- `cron.service`
- `rsyslog.service`
- `unattended-upgrades.service`
- `qemu-guest-agent.service`
- `systemd-networkd.service`
- `systemd-resolved.service`
- `systemd-timesyncd.service`
- `ModemManager.service`
- `multipathd.service`
- `udisks2.service`
- `polkit.service`
- `dbus.service`

No failed systemd services were reported.

## Network Exposure

Listening ports observed:

- Public SSH on `0.0.0.0:22` and `[::]:22`
- Local Ollama API on `127.0.0.1:11434`
- Local systemd resolver on `127.0.0.53/54:53`

No public web server, database, Docker daemon, or app service was listening at the time of review.

## Installed Tooling

Important installed tools:

- Git `2.43.0`
- GitHub CLI `gh` `2.45.0`
- Node.js `v22.22.3` from NodeSource
- npm `10.9.8`
- Python `3.12.3`
- Ollama `0.30.6`
- Codex CLI `@openai/codex@0.137.0`
- Common admin/dev tools: `curl`, `wget`, `jq`, `vim`, `tmux`, `htop`, `tcpdump`, `strace`, `sysstat`

Not present during review:

- Docker / Podman
- nginx / Apache / Caddy
- MySQL / PostgreSQL / Redis
- Go
- Rust / Cargo
- `pip3`

## Ollama

Ollama is installed as a system service:

- Service: `ollama.service`
- Command: `/usr/local/bin/ollama serve`
- User: `ollama`
- Listener: `127.0.0.1:11434`

Registered model initially observed:

- `minimax-m3:cloud`

Registered models observed after the later Ollama cloud wrapper update:

- `minimax-m3:cloud`
- `minimadmax:latest`
- `qwen3.5:397b-cloud`
- `nemotron-3-ultra:cloud`
- `codex-nemotron:latest`

`codex-nemotron:latest` is a custom wrapper around `nemotron-3-ultra:cloud` for slower, deeper VPS engineering work. Its spec is tracked in `Cole-Will-I-Am/MiniMadMax` under `models/codex-nemotron/`.

Operational note: Nemotron may emit visible `Thinking...` output in the plain `ollama run` CLI. For clean automation, call Ollama's API with `think:false`, or use the `codex-nemotron` wrapper.

## Codex Workspace

The AI lab workspace was created at:

```text
/root/ai-lab
```

An operating charter was added at:

```text
/root/ai-lab/AGENTS.md
```

The charter tells Codex to:

- Prefer autonomous investigation, implementation, and verification.
- Keep project work inside `/root/ai-lab` unless doing explicit system administration.
- Use Ollama for local model experiments and OpenAI models for higher-reliability work.
- Run tests and health checks before declaring work complete.
- Avoid printing or storing secrets.
- Prefer reproducible scripts over one-off shell state.

## Codex Configuration

Global Codex config lives at:

```text
/root/.codex/config.toml
```

Current global posture was changed to high autonomy at the user's request:

```toml
model = "gpt-5.5"
model_reasoning_effort = "high"
approval_policy = "never"
sandbox_mode = "danger-full-access"
default_permissions = ":danger-full-access"
web_search = "cached"
```

Enabled Codex features:

```toml
[features]
memories = true
multi_agent = true
hooks = true
shell_snapshot = true
```

An `ai-lab` permission profile still exists in config as a safer scoped profile, but the current global default is `:danger-full-access`.

The autonomous profile file also exists:

```text
/root/.codex/autonomous.config.toml
```

It currently mirrors full-access behavior:

```toml
approval_policy = "never"
sandbox_mode = "danger-full-access"
default_permissions = ":danger-full-access"
model_reasoning_effort = "high"
```

Validation performed:

```bash
codex exec --skip-git-repo-check --output-last-message /tmp/codex-full-access-check.txt "Reply only: full access config ok"
```

The run reported:

- `approval: never`
- `sandbox: danger-full-access`
- `model: gpt-5.5`
- `reasoning effort: high`

## Codex MCP Servers

Configured MCP servers in `/root/.codex/config.toml`:

```toml
[mcp_servers.openaiDeveloperDocs]
url = "https://developers.openai.com/mcp"

[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp"]

[mcp_servers.github]
url = "https://api.githubcopilot.com/mcp/"
bearer_token_env_var = "GITHUB_PAT_TOKEN"
```

Notes:

- `context7` uses `npx`; first use may download the package.
- GitHub MCP expects `GITHUB_PAT_TOKEN` in the Codex process environment.
- The GitHub token is not hardcoded in the Codex config.

## GitHub Access

GitHub CLI was installed with APT:

```bash
apt-get update
apt-get install -y gh
```

The VPS was authenticated to GitHub using device login:

```bash
gh auth login --hostname github.com --git-protocol https --web
```

Authenticated account:

```text
Cole-Will-I-Am
```

Token scopes reported by `gh auth status`:

- `gist`
- `read:org`
- `repo`

Git operations protocol:

```text
https
```

Important: the GitHub token is stored by `gh` under root's GitHub CLI config. Do not commit or print token values.

## GitHub MCP Launcher

A launcher script was created:

```text
/root/ai-lab/bin/codex-github
```

It starts Codex with `GITHUB_PAT_TOKEN` populated from `gh auth token`:

```bash
#!/usr/bin/env bash
set -euo pipefail

export GITHUB_PAT_TOKEN="$(gh auth token)"
exec codex "$@"
```

Permissions:

```text
-rwx------ root root
```

Use it for future Codex sessions that should have GitHub MCP access:

```bash
/root/ai-lab/bin/codex-github
```

## Security-Relevant Findings

Initial SSH posture was permissive:

- `PermitRootLogin yes`
- `PasswordAuthentication yes`
- root password is set
- `/root/.ssh/authorized_keys` was empty
- UFW was installed but inactive

This may be intentional for a lab, but it is the highest-priority hardening item if the server will remain reachable on the public internet.

Recommended hardening when ready:

- Add SSH public key login.
- Disable root password SSH login.
- Create a non-root sudo user.
- Enable UFW with only required ports.
- Consider fail2ban or equivalent SSH brute-force protection.
- Decide whether Codex should remain globally `danger-full-access` or move back to a scoped profile for day-to-day work.

## Package Update State

APT showed pending updates during review. Installing `gh` also produced a kernel notice:

- Running kernel: `6.8.0-111-generic`
- Expected newer kernel: `6.8.0-124-generic`

A reboot may be needed when convenient to load the newer kernel after applying updates.

## Repository Setup

This repository was cloned into:

```text
/root/ai-lab/new-lab
```

Remote:

```text
https://github.com/Cole-Will-I-Am/new-lab.git
```

The first documentation commit records the VPS state and bootstrap decisions made so far.
