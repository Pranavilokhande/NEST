# NEST – NANDA Sandbox and Testbed

NEST (NANDA Sandbox and Testbed) is the production-ready reference platform for building, validating, and deploying multi-agent systems within Project NANDA. This repository now includes the end-to-end assets for the **Group Medical Triage** workflow, a 10-agent care-coordination mesh that spans intake, diagnostics, escalation, and public-health notifications.

---

## Summary

- Adds `scripts/agent_configs/group-medical-triage-all.json`, a Claude-powered, 10-agent workflow for emergency medical triage.
- Documents the full build, validation, and deployment path followed by the project team, including local macOS verification and remote Linode testing.
- Running individual agents (`examples/nanda_agent.py`) and orchestrating the full group using the Linode/Akamai multi-agent deployment script.

---

## What We Delivered

- **Workflow design**: Authored role definitions, prompts, capabilities, and port mappings for ten tightly scoped agents that collaborate through A2A messaging and MCP bindings.
- **Tooling integration**: Validated communications across `@agent-id` handoffs, Smithery MCP connectors, and registry discovery flows.
- **Operational validation**: Ran the stack locally on macOS, executed deployment scripts, spun up a Linode verifier node, and confirmed Claude-backed reasoning with live API keys.
- **Documentation**: This README collates prerequisites, setup, validation commands, troubleshooting notes, and contributor credits.

---

## 10-Agent Medical Triage Workflow

| Agent | Port | Mission Focus |
| --- | --- | --- |
| `triage-nurse` | 6000 | Initial intake, acuity scoring, escalation triggers |
| `diagnostics-analyzer` | 7000 | Differential diagnosis, test selection |
| `radiology-assistant` | 8000 | Imaging prioritization, findings summaries |
| `lab-results-interpreter` | 9000 | Critical lab detection, trend interpretation |
| `medication-safety` | 10000 | Drug interactions, dosing guardrails |
| `care-plan-coordinator` | 11000 | Orders, consults, discharge coordination |
| `emergency-dispatcher-medical` | 12000 | EMS routing, facility selection |
| `public-health-communicator` | 13000 | Stakeholder briefings, advisory messaging |
| `medical-orchestrator` | 14000 | Workflow oversight, SLA enforcement |
| `resource-mapper` | 15000 | Capacity tracking, ETA projections |

Full configuration lives in `scripts/agent_configs/group-medical-triage-all.json`.

---

## Repository Map (Key Paths)

- `examples/nanda_agent.py` – reference single-agent runner (Claude-powered or fallback mode).
- `scripts/agent_configs/group-medical-triage-all.json` – triage workflow definition delivered in this project.
- `scripts/akamai-multi-agent-deployment.sh` – turnkey Linode (Akamai Connected Cloud) multi-agent deployment.
- `nanda_core/core/adapter.py` – main adapter used by every agent instance.

---

## Environment Setup

**Prerequisites**
- Python 3.9 or newer on macOS (validated on macOS Sequoia 15 beta).
- Anthropic Claude API key (`ANTHROPIC_API_KEY`) with access to `claude-3-haiku-20240307`.
- Linode CLI configured with a personal access token (`linode-cli configure` or `LINODE_CLI_TOKEN`).
- Optional: Smithery API key (`SMITHERY_API_KEY`) if telemetry and MCP registries are required.
- `pip install -r requirements.txt` (ensures `anthropic`, `fastapi`, `uvicorn`, etc.).

**Baseline Environment Variables**
```bash
export ANTHROPIC_API_KEY="sk-ant-xxxxxxxx"
export SMITHERY_API_KEY="smithery-xxxxxxxx"        # optional
export REGISTRY_URL="http://registry.chat39.com:6900"
export MCP_REGISTRY_URL="https://<your-mcp-registry>.ngrok-free.app"
```

---

## Local Validation Checklist (Completed)

1. **Schema sanity check** – Confirmed JSON structure with `jq` and internal validators.
   ```bash
   jq empty scripts/agent_configs/group-medical-triage-all.json
   ```
2. **Single agent smoke test** – Ran `python3 examples/nanda_agent.py` to verify Anthropic integration, prompt loading, telemetry hooks.
3. **Multi-agent dry run** – Leveraged deployment scripts in local mode to ensure ports (6000–15000) bind cleanly and logs stream correctly.
4. **A2A messaging** – Posted sample tasks using `curl` against `http://localhost:PORT/a2a` targeting `@agent-id` graph edges defined in the config.
5. **MCP bindings** – Exercised Smithery lookups and tool execution via prompt directives (e.g., `#smithery:@labs fetch latest chemistry panel`).
6. **Akamai script rehearsal** – Ran `bash scripts/akamai-multi-agent-deployment.sh --help` and a dry run with a staging API key to validate CLI authentication, firewall creation, and supervisor bootstrapping.

---

## Running Agents Locally

### Single Agent (Reference)
```bash
python3 examples/nanda_agent.py
```

The script auto-generates an `agent_id`, prints the active specialization, and exposes an A2A endpoint on `http://localhost:6000/a2a` by default. Override defaults with environment variables (`AGENT_ID`, `PORT`, etc.).

### Full Triage Stack (Manual Launch)

For development scenarios where every triage agent runs on the same workstation:
```bash
while read -r agent; do
  agent_id=$(echo "$agent" | jq -r '.agent_id')
  port=$(echo "$agent" | jq -r '.port')
  AGENT_ID="$agent_id" PORT="$port" python3 examples/nanda_agent.py &
done < <(jq -c '.[]' scripts/agent_configs/group-medical-triage-all.json)
wait
```

> Tip: run inside a `tmux` session or process supervisor to manage logs per agent.

---

## Akamai (Linode) Deployment Guide

This repository ships with `scripts/akamai-multi-agent-deployment.sh`, a configurable automation that stands up the triage workflow on Akamai Connected Cloud (Linode). The script performs the following:

- Validates the JSON configuration, checks for port collisions, and warns if the selected Linode plan lacks memory.
- Creates (or updates) a firewall that opens SSH and each agent port defined in `group-medical-triage-all.json`.
- Generates an SSH keypair (if absent) and crafts a user-data bootstrap script that installs dependencies, clones this repo, and installs it in a virtual environment.
- Provisions supervisors that launch every agent with the correct environment variables, including `REGISTRY_URL` and public A2A URLs.
- Executes post-deployment health checks, prints agent endpoints, and surfaces the generated agent IDs for A2A messaging.

### Required Inputs

- Anthropic Claude API key (passed as the first argument).
- Path to the agent configuration JSON (second argument).
- Optional overrides: registry URL, Linode region, instance type, and root password.

### Launch Command

```bash
bash scripts/akamai-multi-agent-deployment.sh \
  "$ANTHROPIC_API_KEY" \
  "scripts/agent_configs/group-medical-triage-all.json" \
  "http://registry.chat39.com:6900" \
  "us-east" \
  "g6-standard-2" \
  "StrongRootPassw0rd!"
```

> Omit the final parameters to accept defaults. If no root password is supplied the script generates a 25-character one and echoes it for record keeping.

### What to Expect

- Provisioning can take several minutes while the instance installs dependencies and supervisor starts every agent.
- Agent logs are stored under `/var/log/agent_<id>.out.log` (stdout) and `.err.log` (stderr).
- Use `ssh -i nanda-multi-agent-key ubuntu@<public-ip>` to inspect the environment; supervisor is the process manager (`sudo supervisorctl status`).
- Health endpoints live at `http://<public-ip>:<port>/health`; A2A messaging at `/a2a`.

### Completed Linode / Akamai Runbook

- Provisioned a g6-standard-2 instance with Ubuntu 25.04.
- Installed system dependencies (`python3-venv`, `git`, `jq`, `supervisor`).
- Deployed the triage agents via supervisor; each agent announced itself with suffixed IDs.
- Validated inbound A2A requests, MCP tool execution, and confirmed all agents appear on the NANDA NEST dashboard.

---

## Verification & QA

- **A2A Conversations** – Confirmed `@triage-nurse` successfully escalates to `@diagnostics-analyzer`, `@care-plan-coordinator`, and `@medical-orchestrator`.
- **Vital extraction** – Smoke-tested vitals parsing prompts through `lab-results-interpreter`.
- **Decision support** – Ensured `medical-orchestrator` resolves conflicting recommendations and emits escalation notices.
- **NEST dashboard visibility** – Verified all ten agents appear in the NANDA NEST dashboard catalog with expected identifiers and offline status until activated.
- **Telemetry** – Observed logging via Smithery integration and native NANDA telemetry module.
- **Port collisions** – None observed between 6000–15000; scripts auto-handle bind failures.

---

## Troubleshooting

- **Missing Claude credentials** – Agents fall back to deterministic responses; ensure `ANTHROPIC_API_KEY` is exported on every host.
- **Linode CLI authentication** – Confirm `linode-cli configure` was completed or `LINODE_CLI_TOKEN` is exported before invoking the script.
- **Registry lookups failing** – Double-check `REGISTRY_URL` reachability and firewall rules; local runs can omit this variable.
- **MCP command errors** – Verify server names (`#smithery:@<server>`) match registry entries and that the Smithery key is active.
- **Agent startup order** – Launch `medical-orchestrator` after dependencies or use supervisor scripts to guarantee availability.

---

## Contributors

- Pranavi Lokhande
- Jeet Patel
- Yi Zhang
- Nabila Nabila
- Shu-Ying Li
- Yingying Su

---

## Additional Resources

- Project NANDA: [https://github.com/projnanda](https://github.com/projnanda)
- Original NEST repository reference: [https://github.com/projnanda/NEST](https://github.com/projnanda/NEST)
- Anthropic Claude API docs: [https://docs.anthropic.com](https://docs.anthropic.com)
- MCP (Model Context Protocol): [https://modelcontextprotocol.io](https://modelcontextprotocol.io)

This README reflects the full scope of work delivered for the Group 7 medical triage submission, capturing design intent, technical steps, and validation artifacts so future contributors can reproduce and extend the workflow.