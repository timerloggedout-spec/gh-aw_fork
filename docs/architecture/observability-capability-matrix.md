# Observability Capability Matrix

Status: active

This repository is a reusable GitHub Agentic Workflows (GH-AW) template. Observability is an optional capability layer and must not become a workflow-execution dependency.

## Capability layers

| Layer | Capability | Contract | Verification |
| --- | --- | --- | --- |
| Transport | OTLP export | `GH_AW_OTEL_*` endpoint + authorization secrets | HTTP write smoke |
| Traces | Tempo ingestion | OTLP traces accepted by Grafana Cloud | TraceQL/read-back when configured |
| Topology | Service graph | INTERNAL root + CLIENT peer spans | Tempo service graph |
| Metrics | OTLP metrics | Metrics emitted/exported when enabled | Grafana query |
| Logs | OTLP logs | Logs emitted/exported when enabled | Loki query |
| Operator | Grafana MCP | Read/query resources with least privilege | MCP inventory |
| Investigation | Trace/log correlation | GitHub run → telemetry identifiers | TraceQL + log query |
| Automation | Alerting | Rules derived from stable signals | Alert evaluation |
| Diagnosis | Sift / incident workflows | Evidence-first investigation | Linked investigation |

## Grafana MCP collaboration contract

An external MCP operator (for example, Grok with `GRAFANA_MCP`) owns Grafana-side discovery and analysis. The GitHub-side implementation owns repository contracts, telemetry semantics, workflow instrumentation, and safe CI validation.

### MCP operator may

1. Inventory Grafana data sources, dashboards, folders, alerts, incidents, and Sift capabilities.
2. Query Tempo, Loki, and metrics to validate telemetry emitted by GH-AW.
3. Produce dashboard, alert, and investigation proposals with exact resource UIDs.
4. Create or patch Grafana resources only when explicitly authorized by the operator and when the proposed mutation is reversible.
5. Return evidence using the handoff format below.

### MCP operator must not

- Put Grafana tokens, cookies, DSNs, or other credentials in repository files, commits, issues, or handoff text.
- Treat dashboard presence as proof of telemetry ingestion.
- Treat a successful OTLP HTTP response as proof that a trace is queryable.
- Change GH-AW workflow execution semantics without a GitHub-side review/commit.

## Evidence ladder

```text
WRITE_OK
   ↓
INGESTED
   ↓
QUERYABLE
   ↓
CORRELATED
   ↓
ACTIONABLE
```

A higher level requires evidence from the preceding level. `WRITE_OK` alone is not a claim that Grafana can retrieve the resulting telemetry.

## Standard handoff

```text
GRAFANA_MCP_HANDOFF
DATASOURCES: <inventory>
DASHBOARDS: <relevant UIDs>
ALERTS: <relevant UIDs>
SERVICE_GRAPH: <observed edges/status>
TRACEQL: <validated query patterns>
LOGQL: <validated query patterns>
RBAC: <minimum scopes required>
ACTIONS: <proposed or completed mutations>
EVIDENCE: <links/identifiers without secrets>
```

## GitHub-side invariants

- OTLP imports remain optional through `if-missing: ignore`.
- Provider adapters and observability adapters remain separate interfaces.
- Copilot/provider credentials are never required by an agentless observability smoke test.
- Read-back tests are optional and must fail closed without converting missing read credentials into workflow execution failures.
- Telemetry identifiers should make GitHub workflow/run correlation possible without recording credentials.

## Capacity expansion roadmap

### P0

- Maintain OTLP write-path smoke coverage.
- Maintain Tempo read-back verification.
- Validate service-graph edge generation.
- Keep provider/WebWrapper failures isolated from observability transport failures.

### P1

- Standardize workflow/run/provider/span correlation attributes.
- Add Grafana dashboards for execution health, latency, errors, retries, and telemetry delivery.
- Add alert rules for sustained workflow/provider failures and telemetry degradation.
- Add GitHub ↔ Grafana deep links where stable identifiers permit them.

### P2

- Add provider/model usage and cost signals where available.
- Add reusable dashboard/alert provisioning for consumers of this template.
- Add richer incident/Sift automation behind explicit authorization.

## Consumer-template rule

All capabilities in this document are designed as opt-in template features. A consumer must be able to use GH-AW without Grafana, Tempo, Sift, or any external observability provider configured.
