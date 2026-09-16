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

<!--
## Optional observability secrets

Consumers may provision these repository Actions secrets to enable OTLP export:

- `GH_AW_OTEL_SENTRY_ENDPOINT`
- `GH_AW_OTEL_SENTRY_AUTHORIZATION`
- `GH_AW_OTEL_GRAFANA_ENDPOINT`
- `GH_AW_OTEL_GRAFANA_AUTHORIZATION`

`if-missing: ignore` keeps this shared import safe when observability is intentionally disabled. Configured destinations continue to export normally; telemetry configuration must not be treated as a prerequisite for the core workflow/provider path.
-->
