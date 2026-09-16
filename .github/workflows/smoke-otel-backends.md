---
private: true
emoji: "🧪"
description: RETIRED — use smoke-otel-write-agentless.yml (no Copilot)
on:
  workflow_dispatch:
permissions:
  contents: read
name: Smoke OTEL (retired agentic)
# RETIRED 2026-09-16
# Reason: engine:codex + model:copilot/* always hard-fails Validate COPILOT_GITHUB_TOKEN
# even with features.samples:true. Paid path. Not for free-template use cases.
#
# Replacement SSOT:
#   .github/workflows/smoke-otel-write-agentless.yml
# Proven: run 35161680890 → http_status=200 WRITE_OK grafana
engine:
  id: codex
model: copilot/mai-code-1-flash-picker
strict: true
features:
  gh-aw-detection: false
  samples: true
safe-outputs:
  threat-detection: false
timeout-minutes: 5
imports:
  - shared/otlp.md
---

# RETIRED — Agentic Smoke OTEL

**Do not use.** Copilot secret validation fails activation before any OTEL work.

Use **`smoke-otel-write-agentless.yml`** instead (pure GHA, free, proven).
