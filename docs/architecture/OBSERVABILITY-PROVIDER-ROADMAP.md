# Observability & Provider Architecture Roadmap

> Production roadmap for the `gh-aw_fork` template. This document is intentionally implementation-oriented: each priority is independently actionable so collaborators and automation can work in parallel without coupling unrelated changes.

## Priority lanes

| Priority | Lane | Definition of done |
| --- | --- | --- |
| P0 | 🔥 Workflow execution failures | Identify, classify, repair, and regression-test failing Actions paths; do not treat downstream empty/safe outputs as agent success. |
| P0 | 🔥 Grafana / OTel observability path | Keep telemetry optional by default; validate the OTLP configuration, document secret ownership, and support direct backend or collector-mediated export. |
| P0 | 🔥 Provider / WebWrapper architecture | Establish provider-neutral interfaces and isolate backend-specific WebWrapper adapters from core workflow/runtime behavior. |
| P1 | 🧠 Copilot integration isolation | Keep Copilot-specific authentication, model selection, prompts, and failure handling behind the provider boundary. |
| P1 | 🧠 Sentry integration | Add Sentry as an optional observability destination without making Sentry credentials a template prerequisite. |
| P1 | 🧠 Telemetry / cost dashboards | Define a backend-neutral event/metric contract for workflow duration, outcome, model/token/cost signals, and provider health; implement dashboards using free-tier/trial resources where available. |
| P2 | Dependabot tuning | Reduce actionable dependency noise while retaining security coverage; document intentional update groups/exclusions. |
| P2 | Generalized dependency hygiene | Consolidate lock/update policy, stale dependency detection, and repeatable maintenance checks. |

## Parallel collaborator workstreams

### Lane A — Workflow reliability (P0)

- Inventory recent failed workflow runs and group failures by root cause.
- Reproduce compiler/runtime failures before changing generated locks.
- Verify every fix with the originating workflow and a relevant compile/check workflow.
- Preserve the `wait → inspect logs → patch → rerun → validate` loop until the affected path is green.

### Lane B — Grafana / OpenTelemetry (P0)

- Treat OTLP as an optional capability for this template.
- Keep provider/backend credentials in GitHub Actions secrets; never commit credential material.
- Maintain the shared OTLP import as safe when no telemetry credentials are configured.
- Support two deployment modes:
  1. direct `gh-aw → Grafana Cloud` export;
  2. `gh-aw → Alloy/OTel Collector → Grafana/Sentry/other backends`.
- Define failure semantics explicitly: telemetry failure must not silently masquerade as workflow/agent success or become an unconditional workflow prerequisite.
- **Notify path:** GitHub Actions webhooks / `repository_dispatch` (not invented Grafana email contact points). See `shared/otlp.md`.

### Lane C — Provider / WebWrapper architecture (P0)

- Define a provider-neutral contract before adding backend-specific behavior.
- Keep WebWrapper adapters replaceable and independently testable.
- Separate provider concerns from observability concerns; an AI provider must not own the telemetry backend contract.
- Prefer capability discovery/configuration over hard-coded provider branches in core workflows.
- **EXTEND Lane B Tempo stream** — free routes emit CLIENT spans with `peer.service` / `gen_ai.*` into the same OTLP path; do not replace B.

### Lane D — Copilot isolation (P1)

- Audit all Copilot-specific paths for leakage into generic workflow logic.
- Centralize Copilot authentication/configuration and document its failure modes.
- Add focused smoke coverage without making other providers depend on Copilot availability.

### Lane E — Sentry (P1)

- Keep Sentry disabled unless explicitly configured.
- Reuse the shared observability contract rather than introducing a second telemetry configuration language.
- Verify Sentry transport/authentication separately from Grafana transport/authentication.

### Lane F — Telemetry / cost dashboards (P1)

- Start with a minimal schema: workflow, run/job identifiers, provider, model, outcome, duration, token/cost fields when available, and error class.
- Avoid storing prompts, secrets, or unnecessary repository content in telemetry.
- Make dashboards useful at both template-maintainer and consumer-repository scope.

### Lane G — Dependency hygiene (P2)

- Tune Dependabot after P0/P1 paths are stable.
- Separate security updates from routine update noise.
- Record policy decisions so future automated maintenance is deterministic.

## Collaboration protocol

1. **Claim a lane before modifying it.** Keep commits narrow and named for the lane/root cause.
2. **Do not block on unrelated lanes.** Provider, observability, and reliability changes should remain independently reviewable.
3. **Cross-lane contract changes require documentation.** Update the relevant schema/reference before or with the implementation.
4. **Generated artifacts follow source changes.** Compile and validate locks; never hand-edit generated output when the compiler can regenerate it.
5. **Evidence over assumptions.** Attach the failing run/job/log or test result to the change rationale.
6. **Wait and re-check.** After each production-facing change, inspect the resulting Actions run(s), then continue the cycle if anything remains red.

## Current implementation note

The shared OTLP import uses `if-missing: ignore`, making observability opt-in for repositories that do not configure the four documented endpoint/authorization secrets. Keep this invariant while expanding the backend suite.

Credentials inventory (names + last-used only): monorepo issue **#184**. Values only in Actions secrets.

### BIUDL densify evidence (2026-09-23)

Lane B multi-peer CLIENT path is **operational**:

| Signal | Evidence |
| --- | --- |
| Write path | Hourly smoke schedule; run **#46 success** |
| Tempo | Continuous multi-span `gh-aw.smoke-otel-agentless` CLIENT peers |
| Service graph servers (7d) | `github.api` ~41 · `openrouter` ~41 · `grafana-otlp-gateway` ~43 |
| Alert | `afyhsc0c23ocgb` state **normal/ok** (evaluates; notify via Actions webhooks when wired) |
| Dashboard | `gh-aw-ops` v7 in folder `gh-aw` |
| Docs | `shared/otlp.md` — Tempo Explore path + webhook/Actions notify + #184 plane |

**Still open (Lane B/C):**

1. Optional Grafana **webhook** contact point → Actions `repository_dispatch` (operator secret wiring; not email invention).
2. **Native CLIENT** in gh-aw engine binary.
3. Lane C free routes emit the same CLIENT attrs into this Tempo stream.

Explore false “No data”: Tempo `grafanacloud-traces`, not Prometheus.

## Exit criteria

The roadmap is considered operationally complete when:

- P0 workflow failures have a verified root cause and regression coverage;
- OTLP can be enabled without making telemetry credentials mandatory for template consumers;
- provider/WebWrapper adapters can evolve without modifying unrelated core runtime paths;
- Copilot and Sentry failures are isolated and diagnosable;
- telemetry/cost dashboards expose useful workflow/provider signals without secret or prompt leakage; and
- P2 dependency maintenance is repeatable and low-noise.
