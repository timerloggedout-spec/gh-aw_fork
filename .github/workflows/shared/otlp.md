---
network:
  allowed:
    - "*.sentry.io"
    - "*.grafana.net"
observability:
  otlp:
    if-missing: ignore
    endpoint:
      - url: ${{ secrets.GH_AW_OTEL_SENTRY_ENDPOINT }}
        headers:
          Authorization: ${{ secrets.GH_AW_OTEL_SENTRY_AUTHORIZATION }}
      - url: ${{ secrets.GH_AW_OTEL_GRAFANA_ENDPOINT }}
        headers:
          Authorization: ${{ secrets.GH_AW_OTEL_GRAFANA_AUTHORIZATION }}
---

# OTLP observability

This shared import is intentionally **optional**. `if-missing: ignore` allows the core workflow/provider path to run when telemetry is not configured.

## Grafana Cloud

Grafana Cloud provides the OTLP destination and the access-policy token; the **Secrets Management → Secure Values** page is a separate Grafana secret store and is not where the GitHub Actions OTLP credential is created.

For direct OTLP export, use the values shown by the Grafana Cloud **OpenTelemetry** card:

1. Copy the **OTLP endpoint URL**. Store it in the GitHub Actions repository secret `GH_AW_OTEL_GRAFANA_ENDPOINT`.
2. Create a Grafana Cloud **Access Policy** with the write scopes required by the telemetry being exported (`metrics:write`, `traces:write`, and `logs:write` as applicable), then create its token. Grafana shows the token once and it normally starts with `glc_`.
3. Copy the **OTLP Instance ID** from the OpenTelemetry card. It is the Basic-auth username for OTLP and is distinct from merely using the stack hostname.
4. Build the Basic-auth value from `OTLP_INSTANCE_ID:TOKEN`, base64-encode that exact string, and store the resulting header value as `Basic <base64-value>` in `GH_AW_OTEL_GRAFANA_AUTHORIZATION`.

Do **not** use the stack URL (for example `https://<stack>.grafana.net`) as `GH_AW_OTEL_GRAFANA_ENDPOINT` unless Grafana's OpenTelemetry card explicitly gives that URL as the OTLP endpoint. The OTLP endpoint normally looks like `https://otlp-gateway-<region>.grafana.net/otlp`.

### GitHub secret mapping

| GitHub Actions secret | Value source |
| --- | --- |
| `GH_AW_OTEL_GRAFANA_ENDPOINT` | Grafana Cloud OpenTelemetry card → **OTLP endpoint URL** |
| `GH_AW_OTEL_GRAFANA_AUTHORIZATION` | `Basic <base64(OTLP instance ID:Grafana Cloud access-policy token)>` |
| `GH_AW_OTEL_SENTRY_ENDPOINT` | Sentry OTLP endpoint |
| `GH_AW_OTEL_SENTRY_AUTHORIZATION` | Sentry OTLP authorization header |

Keep the token only in GitHub Actions Secrets (or an approved external secret-management path); never commit it to the repository or paste it into workflow source.

## Reading traces (why Explore says "No data")

Traces live in **Tempo**, not Prometheus. Default Explore often opens Prometheus → TraceQL returns empty.

1. Open Explore → select datasource **`grafanacloud-pinkkinkajou2544-traces`** (uid `grafanacloud-traces`, type Tempo).
2. TraceQL:

```
{ resource.service.name =~ ".*gh-aw.*" }
```

3. Time range: **Last 24 hours** (or wider).
4. Dashboard: https://pinkkinkajou2544.grafana.net/d/gh-aw-ops/gh-aw-operations — Service variable = `.*gh-aw.*`.

Metrics derived from traces (`traces_spanmetrics_*`, `traces_service_graph_*`) are in Prometheus (`grafanacloud-prom`). Service Graph UI needs CLIENT spans + metrics generation ON (already enabled for this stack).

## Multi-peer CLIENT contract (Lane B + Lane C EXTEND)

Agentless smoke SSOT: `.github/workflows/smoke-otel-write-agentless.yml` emits:

| Span | kind | peer.service |
| --- | --- | --- |
| root run | INTERNAL | — |
| GET api.github.com | CLIENT | github.api |
| POST openrouter…/chat/completions | CLIENT | openrouter (`gen_ai.system=openrouter`, `gh-aw.lane=C-extend`) |
| POST /v1/traces | CLIENT | grafana-otlp-gateway |

**Lane C (provider hub) EXTENDS this stream** — free routes should emit the same CLIENT attrs into Tempo, not a parallel observability stack.

Native engine CLIENT for real GitHub/provider I/O remains the densify path beyond hourly smoke.

## Collector mode

A Grafana Alloy/OpenTelemetry Collector can sit between GH-AW and Grafana Cloud. In that model the workflow sends OTLP to the collector, while the collector owns the Grafana Cloud credentials. This is useful when credentials, retries, routing, or additional observability backends need to be centralized.
