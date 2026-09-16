---
private: true
emoji: "🧪"
description: Agentless smoke — validate OTEL write/read paths without LLM inference cost
on:
  schedule: every 2 days
  slash_command:
    name: smoke-otel-backends
    strategy: centralized
    events: [issues, issue_comment, pull_request, pull_request_comment]
  workflow_dispatch:
  label_command:
    name: smoke
    events: [pull_request]
  reaction: "eyes"
  status-comment: true
permissions:
  contents: read
  issues: read
  pull-requests: read
  actions: read
name: Smoke OTEL
# Agentless data-first: samples replay never invokes an engine.
# Do NOT add copilot-requests:write (bills org AIC — not free).
# Do NOT require COPILOT_GITHUB_TOKEN / ANTHROPIC_API_KEY / OPENAI_API_KEY.
engine:
  id: codex
model: copilot/mai-code-1-flash-picker
strict: true
features:
  gh-aw-detection: false
  samples: true
tools:
  bash: true
  github:
    mode: gh-proxy
    toolsets: [default, issues]
safe-outputs:
  create-issue:
    expires: 2h
    close-older-issues: true
    close-older-key: "smoke-otel-backends"
    labels: [automation, testing, observability]
    max: 1
    samples:
      - temporary_id: "#aw_smoke_otel"
        title: "Smoke OTEL agentless - ${{ github.run_id }}"
        body: |
          Agentless OTEL smoke for run ${{ github.run_id }}.

          This issue is produced by `safe-outputs.create-issue.samples`
          (deterministic — no LLM). Inspect the workflow job logs for:

          - OTEL env injection (GH_AW_OTLP_ENDPOINTS hosts)
          - conclusion-span export status (401/404/ok per backend)
          - optional Grafana Tempo query result (SA token path)

          Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
  threat-detection: false
timeout-minutes: 10
imports:
  - shared/grafana.md
  - shared/otlp.md
sandbox:
  agent:
    id: awf
---

# Smoke OTEL — Agentless Data First

**No LLM. No Copilot AIC. No third-party inference bill.**

This workflow is intentionally **agentless**:

- `features.samples: true` → safe-outputs emit from samples; prompt is **never** sent to an engine
- Job lifecycle still emits OTLP conclusion spans via `actions/setup` (already instrumented)
- Validation is **data**: env presence, export error files, HTTP status attribution

Provider hub / LikeClaw / codex-termux / OpenRouter free catalog belong to **Lane C**
(provider architecture) — not this observability smoke.

## What is free vs paid

| Path | Cost |
| --- | --- |
| OTLP conclusion spans (this smoke) | **Free** (Actions minutes only) |
| `permissions.copilot-requests: write` | **Paid** — org Copilot AI Credits |
| `COPILOT_GITHUB_TOKEN` | **Paid** — seat/entitlement quota |
| `engine: claude` + `ANTHROPIC_API_KEY` | **Paid** — Anthropic |
| `engine: codex` + `OPENAI_API_KEY` | **Paid** — OpenAI |
| Provider hub free-tier routes | Separate Lane C config |

## Optional observability secrets

Telemetry remains opt-in (`if-missing: ignore` on shared OTLP import).

| Secret | Role |
| --- | --- |
| `GH_AW_OTEL_GRAFANA_ENDPOINT` | OTLP write URL |
| `GH_AW_OTEL_GRAFANA_AUTHORIZATION` | `Basic base64(instance:glc_token)` |
| `GRAFANA_URL` | MCP/query base (`https://pinkkinkajou2544.grafana.net`) |
| `GRAFANA_SERVICE_ACCOUNT_TOKEN` | Read path SA token |
| Sentry / Datadog siblings | Optional; empty → export 404 expected |

## Agentless validation contract

Inspect **job logs** (activation/conclusion post steps), not agent output:

1. **Write config present** — `GH_AW_OTLP_ENDPOINTS` lists `otlp-gateway-*.grafana.net`
2. **Write export succeeded** — no `HTTP 401` attributed to Grafana host in
   `/tmp/gh-aw/agent/otlp-export-errors.jsonl` (or post-step export log)
3. **Read config present** — `GRAFANA_URL` + `GRAFANA_SERVICE_ACCOUNT_TOKEN` set
4. **Read query** (optional operator step) — Tempo TraceQL via Grafana API with SA token

Agentic matrix (Sentry MCP / Grafana MCP / Datadog MCP via LLM) is **out of scope**
until Lane C wires a free provider path. Do not reintroduce paid Copilot auth to
force the old agent matrix.

## Rules

- Telemetry failure must not be treated as task-quality failure (roadmap Lane B).
- Never commit credentials.
- Prefer evidence from export status codes over dashboard screenshots.
- Keep provider concerns off this workflow.
