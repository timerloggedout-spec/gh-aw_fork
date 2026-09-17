# Grafana MCP Collaboration Contract

## Purpose

Treat Grafana Cloud as the observability control plane for this TEMPLATE while keeping telemetry transport independent from any particular AI client. Grafana MCP is an **operator/agent productivity surface**, not a runtime dependency of GH-AW.

The repository already has a data-first OTLP lane: the agentless smoke path has been used to prove a successful Grafana OTLP write, and the current smoke implementation emits an INTERNAL root span plus a CLIENT child span to seed Tempo service-graph edges. The shared OTLP import remains optional so provider/workflow execution is not coupled to telemetry availability.

## Capability model

| Layer | Capability | Primary interface | Production role |
| --- | --- | --- | --- |
| Transport | OTLP traces/metrics/logs | GH-AW → Grafana Cloud OTLP | Runtime telemetry |
| Trace analysis | TraceQL/search/retrieve | Tempo / Grafana MCP | Incident/debug investigation |
| Metrics | PromQL / dashboard panel queries | Grafana MCP | Capacity, latency, error analysis |
| Logs | LogQL / log exploration | Grafana MCP | Failure correlation |
| Dashboards | Search, summaries, targeted patches | Grafana MCP | Human + agent operating surface |
| Alerting | Rule inspection and controlled changes | Grafana MCP | Reliability automation |
| Incidents | Incident notes/state and investigation context | Grafana MCP | Response workflow |
| Sift | Error-pattern / slow-request investigations | Grafana MCP | Higher-level diagnosis |
| Navigation | Deep links into Explore/resources | Grafana MCP | Fast handoff from automation to operator |
| Service graph | CLIENT/SERVER span edges | Tempo metrics-generator | Dependency/capacity visibility |
| GitHub automation | Smoke, probes, correlation metadata | Actions | Continuous verification |

Grafana documents these MCP areas as available capabilities, with RBAC/scopes controlling which tools can be used. Keep the MCP service account least-privileged; broad Editor access is a convenience option, not the default contract.

## Grok collaboration lane

Grok's `GRAFANA_MCP` is an **external parallel operator**. This repository cannot directly invoke or delegate to Grok from GitHub Actions, so collaboration is established through artifacts and explicit handoff data rather than hidden coupling.

### Grok should own / inspect

1. Grafana Cloud MCP tool availability and enabled-tool categories.
2. Current datasource inventory and UIDs for Prometheus/Mimir, Loki, and Tempo.
3. Existing dashboards/folders relevant to GH-AW and MCP observability.
4. Existing alert rules, notification policies, and SLOs that can cover GH-AW.
5. Tempo service-graph configuration and whether the CLIENT edge emitted by the smoke workflow is materializing as an edge.
6. The minimum service-account/RBAC scopes needed for the above read paths.
7. Candidate dashboard/alert changes, returning exact resource identifiers and evidence before mutation.

### GitHub/GH-AW should own

1. Workflow source, locks, provider adapters, smoke tests, and CI.
2. OTLP secret names and transport contracts.
3. Trace/resource semantic conventions and correlation fields.
4. Safe, repeatable telemetry probes.
5. Documentation of Grafana resource UIDs returned by the operator lane.
6. Any repository-side automation that consumes the agreed Grafana contract.

### Handoff format

Grok should return findings in this compact form so they can be committed or implemented without rediscovery:

```text
GRAFANA_MCP_HANDOFF
stack=<redacted hostname if sensitive>
checked_at=<UTC>

DATASOURCES
- type=<type> uid=<uid> role=<metrics|logs|traces>

DASHBOARDS
- uid=<uid> title=<title> folder=<folder>

ALERTS
- uid=<uid> title=<title> state=<state>

SERVICE_GRAPH
- enabled=<true|false>
- source=<tempo metrics-generator|other>
- gh_aw_edge_observed=<true|false|unknown>

RBAC
- tool/category=<minimum permission + scope>

ACTIONS
- proposed=<specific change>
- evidence=<query/result/run/deeplink>
- mutation_required=<true|false>
```

Never include access tokens, authorization headers, cookies, or other credentials in the handoff.

## Capability expansion targets

### P0 — Close the observability verification loop

- Keep the agentless OTLP write smoke as the source-of-truth transport test.
- Verify the write path and read path independently: HTTP success is necessary but not sufficient; a trace must be discoverable in Tempo.
- Keep the optional Tempo read probe non-blocking when a Grafana service-account read token is not configured.
- Correlate every smoke trace with `github.repository`, `github.run_id`, `service.name`, and a stable smoke/version attribute.
- Preserve the CLIENT edge to `grafana-otlp-gateway` so service-graph metrics can be generated when the Grafana/Tempo side is configured for them.

### P1 — Turn Grafana MCP into the operator cockpit

- Add a canonical GH-AW observability dashboard in Grafana, using dashboard-as-code or MCP-managed resources rather than hand-only edits.
- Surface workflow duration, success/failure rate, agent activation failures, provider latency, OTLP delivery status, and cost/usage where data is available.
- Add drill-down links from dashboard panels to GitHub Actions runs, workflow files, and Tempo traces.
- Add alerts only after the corresponding metric/query is proven and has an owner/runbook.
- Use MCP read tools for routine investigation; reserve write tools for explicit, evidence-backed changes.

### P1 — Provider/WebWrapper observability

Standardize provider telemetry around:

- `gh_aw.provider.name`
- `gh_aw.provider.operation`
- `gh_aw.provider.model` when available and non-sensitive
- `gh_aw.workflow`
- `github.repository`
- `github.run_id`
- `github.workflow_ref`
- duration/error/status attributes

Do not emit prompts, tokens, cookies, authorization headers, or other sensitive payloads into telemetry.

### P2 — Capacity and cost intelligence

Build a common view across GitHub Actions, model/provider calls, and Grafana telemetry:

- execution minutes
- workflow concurrency
- queue/wait time
- provider latency
- retry rate
- token/usage data when legitimately available
- telemetry volume and retention pressure

The goal is a single correlation chain:

`GitHub run → GH-AW workflow → provider operation → OTEL trace → Tempo/Loki/Metrics → Grafana dashboard/alert`

## Guardrails

1. **MCP is optional.** GH-AW must continue to function when Grafana MCP is unavailable.
2. **Telemetry is optional.** Missing OTLP configuration must not prevent core workflows from starting.
3. **Read before write.** Inspect current Grafana resources and permissions before mutating them.
4. **Least privilege.** Prefer narrowly scoped service-account permissions for the enabled MCP tools.
5. **No secret propagation.** Grafana credentials remain in GitHub/Grafana secret stores and never enter repository source, telemetry attributes, or collaboration handoffs.
6. **Evidence required.** Dashboard/alert/service-graph changes should cite the query, run, or resource that motivated the change.
7. **Provider/observability separation.** Provider adapters must not contain Grafana-specific control-plane logic; observability adapters own telemetry export/correlation.

## Wait → inspect → patch → rerun → validate

Every observability change follows this loop:

1. **Wait** for the relevant scheduled/manual run to finish.
2. **Inspect** GitHub job logs and Grafana MCP/read evidence.
3. **Patch** the smallest repository-side defect or configuration contract.
4. **Rerun** the narrowest relevant workflow/job.
5. **Validate** both transport and data visibility.
6. Repeat until the exit criteria are met.

### Exit criteria

- OTLP write is green.
- Trace is discoverable in Tempo.
- Service-graph edge is observed or the Grafana-side reason for absence is documented.
- Core workflow execution remains independent of telemetry/MCP availability.
- Grafana MCP can inspect the relevant dashboards/datasources/alerts with the agreed least-privilege identity.
- Provider telemetry follows the shared semantic contract.
- No credentials appear in source, logs, traces, or handoff artifacts.
