# Observability & Provider Architecture Roadmap

> Production roadmap for the `gh-aw_fork` template. This document is intentionally implementation-oriented: each priority is independently actionable so collaborators and automation can work in parallel without coupling unrelated changes.

## Priority lanes

| Priority | Lane | Definition of done |
| --- | --- | --- |
| P0 | 🔥 Workflow execution failures | Identify, classify, repair, and regression-test failing Actions paths; do not treat downstream empty/safe outputs as agent success. |
| P0 | ✅ Grafana / OTel observability path | **EXIT complete (agentless Lane B):** optional OTLP, densified multi-peer CLIENT, Tempo/Prom/service-graph live, alert evaluates, Actions webhook notify contract. |
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

### Lane B — Grafana / OpenTelemetry (P0) — **EXIT COMPLETE**

Done criteria met:

| Criterion | Evidence |
| --- | --- |
| Optional OTLP (`if-missing: ignore`) | `shared/otlp.md` |
| Direct write path green | Hourly `smoke-otel-write-agentless`; multi-peer CLIENT |
| Tempo + spanmetrics + service graph | Servers github.api / openrouter / grafana-otlp-gateway live |
| Alert evaluation | Rule `afyhsc0c23ocgb` normal/ok |
| Notify contract | `.github/workflows/grafana-alert-dispatch.yml` (`repository_dispatch` type `grafana-alert`) |
| Credentials plane | Monorepo #184 names/last-used only; values in Actions secrets |
| Operator dashboard | `gh-aw-ops` in folder `gh-aw` |

**Notify:** webhooks = **GitHub Actions**, not invented Grafana email. Operator may attach a Grafana Webhook contact point to the repo dispatches API; agents never paste tokens.

**Out of Lane B exit scope (separate lanes):**
- First-party Go SDK OTEL exporter in `pkg/` (OTEL only appears as indirect deps today).
- Lane C free-route CLIENT emit into the same Tempo stream.

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

The shared OTLP import uses `if-missing: ignore`. Credentials inventory: monorepo **#184**. Lane B agentless densify + Actions notify contract is **closed**.

## Exit criteria

- [x] Lane B OTLP optional + densified + alert + Actions notify contract
- [ ] P0 workflow failures have verified root cause and regression coverage (Lane A)
- [ ] provider/WebWrapper adapters evolve without modifying unrelated core runtime (Lane C)
- [ ] Copilot and Sentry failures isolated (Lane D/E)
- [ ] telemetry/cost dashboards without secret/prompt leakage (Lane F)
- [ ] P2 dependency maintenance repeatable and low-noise (Lane G)
