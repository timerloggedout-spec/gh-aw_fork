# Grafana MCP Collaboration Contract

## Purpose

Treat Grafana Cloud as the observability **control plane for agentic development collaborators** (Grok, ChatGPT, other MCP operators). Human UI is secondary; a unified app UX comes later.

Telemetry transport stays independent of any particular AI client. Grafana MCP is an **operator/agent productivity surface**, not a GH-AW runtime dependency.

## Lane map (do not collapse)

| Lane | Role |
| --- | --- |
| **B — Observability** | OTLP → Grafana → Tempo/Prom; agentless smoke; MCP operator |
| **C — Provider / WebWrapper** | Free/provider routes; **EXTENDS** Lane B (same Tempo stream, CLIENT spans for model/API calls) — **does not replace B** |
| **A — Workflow reliability** | Separate P0; keep out of transport green path |

Provider hub emitting OTEL is **Lane C extending B**, never a substitute for the agentless write path.

## Capability model

| Layer | Capability | Primary interface | Production role |
| --- | --- | --- | --- |
| Transport | OTLP traces/metrics/logs | GH-AW → Grafana Cloud OTLP | Runtime telemetry |
| Trace analysis | TraceQL/search/retrieve | Tempo / Grafana MCP | Agentic investigation |
| Metrics | PromQL / spanmetrics / service graph | Grafana MCP | Capacity, latency, edges |
| Dashboards | Search, patch, `gh-aw-ops` | Grafana MCP | Collaborator operating surface |
| Alerting / Incidents / Sift | Rules + RCA investigations | Grafana MCP | Reliability automation |
| Service graph | CLIENT peer edges | Tempo metrics-generator | Dependency visibility |
| GitHub automation | Agentless multi-peer smoke | Actions | Continuous verification |

## Instrumentation contract (CLIENT-first)

Engine workflow spans are largely `SPAN_KIND_INTERNAL` today. Service graphs need **CLIENT** (and ideally SERVER) pairs.

**Agentless multi-peer smoke (v3)** emits under `gh-aw.smoke-otel-agentless`:

1. INTERNAL root — `gh-aw.smoke-otel-agentless.run`
2. CLIENT → `peer.service=github.api`
3. CLIENT → `peer.service=openrouter` (`gh-aw.lane=C-extend`, `gen_ai.system=openrouter`)
4. CLIENT → `peer.service=grafana-otlp-gateway`

When the gh-aw binary gains native CLIENT instrumentation, mirror these peers for real GitHub API and provider calls. Until then, the smoke is the topology SSOT for metrics-generator + collaborators.

## Grok / MCP operator lane

Grok's `GRAFANA_MCP` is an **external parallel operator**. Collaboration is via artifacts and handoff data, not hidden coupling from Actions → Grok.

### Operator owns

1. MCP tool inventory and RBAC scopes
2. Datasource UIDs (Tempo/Prom/Loki)
3. Dashboard `gh-aw-ops` and folder state
4. Alert rules / investigation profiles
5. Service-graph series and CLIENT edge materialization
6. Evidence-backed mutations only

### Handoff format

```text
GRAFANA_MCP_HANDOFF
stack=<hostname>
checked_at=<UTC>

DATASOURCES
- type=tempo uid=grafanacloud-traces
- type=prometheus uid=grafanacloud-prom

DASHBOARDS
- uid=gh-aw-ops title=gh-aw Operations

ALERTS
- (none | uid=...)

SERVICE_GRAPH
- enabled=true|false
- series=traces_service_graph_* present|absent
- peers_observed=github.api,openrouter,grafana-otlp-gateway

RBAC
- minimum scopes for read tools

ACTIONS
- proposed=...
- evidence=run/trace/query
- mutation_required=true|false
```

No credentials in handoff.

## Grafana AI (Assistant / Investigations)

Stack-local AI over **your** Tempo/Prom — not a Lane C provider replacement and not Copilot AIC.

| Surface | Agentic use |
| --- | --- |
| Assistant | NL → TraceQL/PromQL from collaborator sessions |
| Investigations | MCP `create_investigation` on failed runs / OTLP gaps |
| agento11y | Later: score gen_ai attrs if emitted |

Investigations fire **after** CLIENT instrumentation is seeding edges (this contract), not before.

## Guardrails

1. MCP optional; GH-AW runs without it
2. Telemetry optional (`if-missing: ignore`)
3. Read before write
4. Least privilege SA
5. No secrets in source, traces, or handoffs
6. **Lane C extends B** — provider adapters own free routes; observability owns export/correlation
7. Evidence required for mutations

## Wait → inspect → patch → rerun → validate

1. Wait for Actions / metrics-gen lag
2. Inspect logs + GRAFANA_MCP (Tempo + Prom series)
3. Patch smallest defect
4. Rerun narrowest job
5. Validate write **and** discoverability
6. Repeat

### Exit criteria

- OTLP write green
- Trace in Tempo with multi-peer CLIENT attrs
- `traces_spanmetrics_*` / `traces_service_graph_*` present or absence documented
- Core workflows independent of telemetry/MCP
- Provider (Lane C) can attach CLIENT spans to the **same** stream without forking transport
