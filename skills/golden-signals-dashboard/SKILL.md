---
name: golden-signals-dashboard
description: >-
  Instrument an ASP.NET Core backend service with OpenTelemetry Prometheus
  metrics and provision a four-golden-signals Grafana dashboard (latency,
  traffic, errors, saturation per http.route) through the service's Helm
  chart, targeting a standard kube-prometheus-stack cluster. Use when
  adding observability/monitoring, a golden-signals dashboard, a ServiceMonitor,
  or a /metrics endpoint to an ASP.NET Core backend repo.
---

# Golden Signals Dashboard

Add the four golden signals — **latency, traffic, errors, saturation** — to an
ASP.NET Core backend service, and surface them in Grafana. The pattern:
OpenTelemetry auto-instrumentation exposes Prometheus metrics on a dedicated
port; a ServiceMonitor makes the cluster's kube-prometheus-stack scrape them; a
ConfigMap ships a per-app Grafana dashboard.

The cluster side (Prometheus + Grafana + operator) is assumed to already be
installed via `kube-prometheus-stack` — a standard, widely-used Helm chart, so
this skill is cluster-agnostic. It only touches the **application repo** — its
.NET project and its Helm chart.

## When to use

- A backend service should report golden signals / appear in Grafana.
- Someone asks for a ServiceMonitor, a `/metrics` endpoint, or "monitoring" on
  an ASP.NET Core .NET service.

## Prerequisites (verify first)

- The service is **ASP.NET Core** (Kestrel) and has a **Helm chart** in the repo
  (typically under `helm/<app>/`). If there's no chart, stop and say so — this
  skill provisions monitoring *through* the chart.
- The cluster already runs `kube-prometheus-stack`. With its default values, the
  operator watches ServiceMonitors in **all** namespaces, and Grafana
  auto-discovers dashboard ConfigMaps labeled `grafana_dashboard: "1"` in any
  namespace — so no cluster-side change is needed. If you're unsure the install
  matches these defaults, run the optional introspection below.

## (Optional) Verify the cluster's kube-prometheus-stack

Skip this if you already know the cluster runs a default `kube-prometheus-stack`.
Run it when you have `kubectl` access and want to confirm the install actually
matches the four assumptions this skill depends on — different installs scope the
operator by namespace or by label, which silently breaks auto-discovery. Replace
`<ns>` with the monitoring namespace (often `monitoring` or `kube-prometheus-stack`).

1. **Operator + CRDs present** — the prometheus-operator must be running and the
   `ServiceMonitor` CRD installed, or the chart's ServiceMonitor is ignored:
   ```bash
   kubectl get crd servicemonitors.monitoring.coreos.com
   kubectl get pods -A -l app.kubernetes.io/name=prometheus-operator
   ```

2. **ServiceMonitor selection scope** — confirm the Prometheus CR doesn't filter
   ServiceMonitors by namespace or label. Empty/null selectors mean "match all"
   (the default this skill expects):
   ```bash
   kubectl get prometheus -A \
     -o custom-columns=NAME:.metadata.name,\
SM_SELECTOR:.spec.serviceMonitorSelector,\
SM_NS_SELECTOR:.spec.serviceMonitorNamespaceSelector
   ```
   If `serviceMonitorSelector` is set, the chart's ServiceMonitor labels must
   match it. If `serviceMonitorNamespaceSelector` is set, the app's namespace
   must satisfy it.

3. **Grafana dashboard sidecar enabled** — the sidecar is what turns a labeled
   ConfigMap into a live dashboard. Confirm it's running and learn which label it
   watches (default `grafana_dashboard: "1"`, all namespaces):
   ```bash
   kubectl get pods -A -l app.kubernetes.io/name=grafana \
     -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{" "}{end}{"\n"}{end}'
   # look for a "grafana-sc-dashboard" sidecar container, then inspect its env:
   kubectl -n <ns> set env deploy/<grafana-deploy> --list | grep -i 'LABEL\|NAMESPACE'
   ```
   `LABEL=grafana_dashboard`, `LABEL_VALUE=1`, and `NAMESPACE=ALL` are the
   defaults the assets rely on. If the label differs, adjust the
   `grafana_dashboard: "1"` label in `assets/grafana-dashboard.yaml`.

If any check disagrees with the defaults, note the difference and adapt the
assets (labels/namespace) before checking them in — otherwise the dashboard or
scrape target silently won't appear.

## Placeholders

The bundled assets use placeholders. Resolve them from the target repo before
writing files:

| Placeholder | Meaning | How to determine |
|---|---|---|
| `__APP__` | Service display name | e.g. `core-api`, `common-api` |
| `__CHART__` | Helm chart name / `_helpers.tpl` prefix | the `name:` in `helm/<app>/Chart.yaml`; used in `include "<chart>.fullname"` |
| `__CONTAINER__` | Container name in the Deployment | from the chart's `deployment.yaml` |
| `__POD_PREFIX__` | Pod name prefix for saturation queries | usually `<release>-<chart>` or the helm fullname, e.g. `core-api`; confirm via `kubectl get pods` |
| `__JOB__` | Prometheus `job` label for the scraped service | defaults to the **Service name** the ServiceMonitor selects (the chart fullname). Confirm after deploy with `label_values(http_server_request_duration_seconds_count, job)` in Grafana Explore |

`__NAMESPACE__` is **not** a placeholder — the dashboard takes namespace as a
Grafana template variable so one dashboard serves both `dev` and `prod`.

## Steps

### 1. Instrument the .NET service (web project)

1. Add the OpenTelemetry package references from `assets/csproj.snippet.xml`
   to the web project's `.csproj` (keep the versions unless the repo pins a
   newer OTel line).
2. In `Program.cs`, apply `assets/Program.cs.snippet`:
   - In non-Development, make Kestrel listen on **8080** (public) **and 9090**
     (metrics-only).
   - Register `AddOpenTelemetry().WithMetrics(...).AddPrometheusExporter()`
     with AspNetCore + HttpClient + Runtime instrumentation.
   - Map the scrape endpoint scoped to the metrics port:
     `app.MapPrometheusScrapingEndpoint().RequireHost("*:9090");`
   The 8080/9090 split keeps `/metrics` off the public ingress/tunnel — only
   the in-cluster ServiceMonitor reaches port 9090.

   Adapt to the repo: match its `serviceName`, its existing Kestrel config, and
   whether it already calls `AddOpenTelemetry()`. Don't duplicate registrations.

### 2. Wire monitoring into the Helm chart (`helm/<app>/`)

1. **Service** — add the metrics port. Merge `assets/service-metrics-port.snippet.yaml`
   into the chart's `templates/service.yaml` (a second named port `metrics`,
   9090). Don't route it from the ingress/tunnel.
2. **ServiceMonitor** — copy `assets/servicemonitor.yaml` to
   `templates/servicemonitor.yaml`, replacing `__CHART__` with the chart's
   helper prefix.
3. **Dashboard ConfigMap** — copy `assets/grafana-dashboard.yaml` to
   `templates/grafana-dashboard.yaml` (same `__CHART__` substitution). The
   embedded filename **must be globally unique** (`__APP__-golden-signals.json`)
   — the Grafana sidecar flattens all dashboard ConfigMaps into one directory,
   so a generic name collides across apps and silently loses.
4. **Dashboard JSON** — copy `assets/golden-signals.json` to
   `helm/<app>/dashboards/golden-signals.json`, substituting `__APP__`,
   `__JOB__`, `__POD_PREFIX__`, `__CONTAINER__`.
5. **values.yaml** — add the toggles from `assets/values.snippet.yaml`
   (`serviceMonitor.enabled`, `dashboards.enabled`, and `service.metricsPort`).
   Keep them on by default; they let environments without prometheus-operator
   opt out.

### 3. Verify

- `helm template helm/<app>` renders without error and includes the
  ServiceMonitor + dashboard ConfigMap.
- After deploy: in Grafana, the dashboard **"`__APP__` — Golden Signals"**
  appears (auto-provisioned), and its panels populate once traffic flows.
- If panels read "No data": the `__JOB__` value is almost always the culprit —
  check `label_values(http_server_request_duration_seconds_count, job)` and
  fix the dashboard JSON. Confirm the target is `UP` in Prometheus
  (`kubectl -n monitoring port-forward svc/prom-stack-kube-prometheus-prometheus 9090` → /targets).

## What the dashboard shows

Latency p50 + p99 per route, traffic (req/min) per route, 5xx errors per route,
and CPU/memory saturation as a % of each pod's limit. `http.route` comes free
from ASP.NET Core route templates, so `api/patients/{id}` collapses to one
series instead of one-per-id. App-specific panels (cache hit ratio, custom
business counters) can be appended later by adding a `Meter` and a panel — the
OTel setup already reserves a custom meter namespace.
