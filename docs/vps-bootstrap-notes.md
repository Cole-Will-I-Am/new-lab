# VPS Bootstrap Notes

Last updated: 2026-06-06 UTC

This repository records the current state and setup decisions for the experimental AI VPS. It is intentionally documentation-only at the moment.

## Host Summary

- Hostname: `srv1736822`
- OS: Ubuntu 24.04.4 LTS Noble
- Kernel observed initially: `6.8.0-111-generic`
- Kernel observed after reboot/update: `6.8.0-124-generic`
- Virtualization: KVM/QEMU
- CPU: 1 vCPU on AMD EPYC 7543P host
- RAM: 3.8 GiB
- Swap initially: none
- Swap after hardening/update: 2 GiB `/swapfile`
- Root disk: 48 GiB, roughly 7.5 GiB used at the 2026-06-06 follow-up review
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

Custom model prompt/config update workflow:

- Prompt source files live in `Cole-Will-I-Am/MiniMadMax`:
  - `models/minimadmax/system.md`
  - `models/codex-nemotron/system.md`
- Generated Ollama Modelfiles:
  - `Modelfile`
  - `models/codex-nemotron/Modelfile`
- Rebuild command installed on the VPS:
  - `rebuild-ollama-models [all|minimadmax|codex-nemotron|render|profiles] [--profile NAME]`
- Useful MiniMaxine-derived pieces adapted into `MiniMadMax`:
  - shared tuning profiles in `models/profiles/`
  - external outcome memory in `data/outcomes.jsonl` and `data/learned.json`
  - `model-outcome` for recording task outcomes
  - `model-json-extract` for cleaning JSON out of noisy Ollama model output

Plain Ollama model sessions cannot persist their own changes. A tool-enabled Codex session can edit the source-controlled prompt files, run `rebuild-ollama-models`, smoke-test the model, and commit/push the result.

Git execution boundary verified on 2026-06-06:

- Test repo: `https://github.com/Cole-Will-I-Am/testy`
- Plain `ollama run --experimental --experimental-yolo` did not let `minimadmax:latest` or `codex-nemotron:latest` actually edit/commit/push.
- MiniMadMax produced a simulated command transcript for the wrong repo during the direct execution test.
- CodexNemotron produced commands but did not execute them.
- Both models produced structured commit requests when asked.
- This tool-enabled Codex session executed those requests and pushed commits `6c01cb2` and `ec50ce9` to `testy`.

## Codex Workspace

The AI lab workspace was created at:

```text
/root/ai-lab
```

An operating charter was added at:

```text
/root/ai-lab/AGENTS.md
```

A source-controlled copy lives in this repository:

```text
/root/ai-lab/new-lab/AGENTS.md
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

A source-controlled copy lives in this repository:

```text
/root/ai-lab/new-lab/bin/codex-github
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

Initial SSH posture was permissive during bootstrap:

- `PermitRootLogin yes`
- `PasswordAuthentication yes`
- root password is set
- `/root/.ssh/authorized_keys` was empty
- UFW was installed but inactive

Follow-up review on 2026-06-06 showed the critical SSH/firewall items had been corrected:

- `PermitRootLogin no`
- `PasswordAuthentication no`
- `PubkeyAuthentication yes`
- `KbdInteractiveAuthentication no`
- UFW active with default deny incoming and only public SSH allowed
- `fail2ban.service` active with the `sshd` jail enabled
- Non-root sudo user `admin` exists
- `/home/admin/.ssh/authorized_keys` has one `ssh-ed25519` key
- `/root/.ssh/authorized_keys` has one `ssh-rsa` key, but root SSH login is disabled

Remaining security posture decisions:

- Decide whether Codex should remain globally `danger-full-access` or move back to a scoped profile for day-to-day work.
- Consider whether public SSH should remain open to the world or be restricted by source IP/VPN if a stable admin access path exists.
- Consider replacing any remaining RSA-only root key material with ed25519 key material, even though root SSH login is currently disabled.

## Package Update State

APT showed pending updates during initial review. Installing `gh` also produced a kernel notice:

- Running kernel: `6.8.0-111-generic`
- Expected newer kernel: `6.8.0-124-generic`

Follow-up review on 2026-06-06 showed:

- Running kernel: `6.8.0-124-generic`
- `apt-get -s upgrade`: `0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded`
- `/var/run/reboot-required`: absent
- `unattended-upgrades.service`: enabled

## Follow-Up Review Notes

Live review on 2026-06-06 found:

- No failed systemd services.
- Public listeners: SSH on port 22 only.
- Local listeners: Ollama on `127.0.0.1:11434`, systemd resolver on loopback.
- `ssh.socket` is enabled and socket-activates `ssh.service`; `ssh.service` itself may show disabled, which is expected for this setup.
- `ollama.service`, `fail2ban.service`, `ufw`, and `unattended-upgrades.service` are enabled.
- `Cole-Will-I-Am/MiniMadMax` and `Cole-Will-I-Am/new-lab` were clean and pushed to `origin/main`.
- `codex-nemotron "prompt"` returned clean output through the Ollama API with `think:false`.
- Source-controlled copies now exist for `/root/ai-lab/AGENTS.md` and `/root/ai-lab/bin/codex-github`.

Observed low-priority warnings:

- `/etc/pam.d/login` references missing `pam_lastlog.so`, causing a console-login PAM warning.
- Cloud-init generated netplan config includes `localhost` as a DNS search domain, which systemd-networkd ignores with a warning.
- Custom Ollama wrapper models may show `unknown capabilities` warnings in Ollama logs; base cloud models still expose capabilities normally.

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
