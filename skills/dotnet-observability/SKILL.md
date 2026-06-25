---
name: dotnet-observability
description: >-
  Instrument a .NET service (ASP.NET Core or worker/console host) for the
  Grafana observability stack: OpenTelemetry Prometheus metrics with a
  four-golden-signals dashboard, and/or Serilog JSON logs queryable in Loki
  with a per-app logs dashboard — all provisioned through the service's Helm
  chart, targeting a cluster that already runs kube-prometheus-stack (+ Loki/
  Alloy for logs). Also wires the per-cluster Grafana observability hub home
  page. Use when adding observability/monitoring/metrics/logging, a
  golden-signals dashboard, a ServiceMonitor, a /metrics endpoint, structured
  JSON logging, or a Loki logs dashboard to a .NET service.
---

# .NET Observability (metrics + logs)

Wire a .NET service into the Grafana observability stack. Two independent
halves, each opt-in:

- **Metrics (golden signals).** OpenTelemetry auto-instrumentation exposes
  Prometheus metrics on a dedicated port; a ServiceMonitor makes the cluster's
  kube-prometheus-stack scrape them; a ConfigMap ships a per-app golden-signals
  Grafana dashboard (**latency, traffic, errors, saturation** per `http.route`).
- **Logs (Loki).** Serilog emits one single-line JSON object per event to
  stdout; the cluster's Alloy DaemonSet tails it into Loki; a per-app logs
  dashboard surfaces volume-by-level, error/warning counts, and live error +
  all-log streams. The `app` Loki label comes from the pod's
  `app.kubernetes.io/name` label, and level filtering uses Loki's
  `detected_level` (which normalizes Serilog's `Information`/`Warning`/`Error`
  casing).

Both halves assume the cluster side already exists — `kube-prometheus-stack`
for metrics, **Loki + Alloy** for logs (both standard Grafana-stack installs).
This skill only touches the **application repo** (its .NET project + Helm
chart), plus a **one-time, per-cluster** hub dashboard that lives in the infra/
promstack repo. It is cluster-agnostic.

## Step 0 — ask what to instrument

The pieces are independent and not everyone wants all of them. **Before
touching anything, ask the user which to apply** (offer "the works" =
everything). Use AskUserQuestion with a multi-select:

| Piece | What it adds | Depends on |
|---|---|---|
| **Metrics / golden signals** | OTel packages, `/metrics` on 9090, ServiceMonitor, golden-signals dashboard | kube-prometheus-stack |
| **JSON logging (Loki)** | Serilog packages + JSON-to-stdout, `app` pod label | Loki + Alloy |
| **Per-app logs dashboard** | `logs.json` dashboard (volume, errors, log streams) | JSON logging above |
| **Errors panel on golden-signals** | live error-log stream appended to the bottom of the metrics dashboard | both halves |
| **Observability hub home page** | per-cluster Grafana landing page listing every app's dashboards by tag | one-time, infra repo |

"The works" = all five. A metrics-only or logs-only subset is fine — skip the
sections below that the user didn't pick. The **hub** is a once-per-cluster
setup; if it already exists, picking it is a no-op (new apps appear in it
automatically via tags — no edit needed).

## Prerequisites (verify first)

- The service is **.NET** with a **Helm chart** in the repo (typically under
  `helm/<app>/`). If there's no chart, stop and say so — this skill provisions
  observability *through* the chart. Metrics additionally require **ASP.NET
  Core** (Kestrel); logging works for ASP.NET Core **and** worker/console hosts
  (`Host.CreateApplicationBuilder`).
- **For metrics:** the cluster runs `kube-prometheus-stack` with default
  scoping — the operator watches ServiceMonitors in **all** namespaces, and
  Grafana auto-discovers dashboard ConfigMaps labeled `grafana_dashboard: "1"`
  in any namespace. If unsure, run the introspection below.
- **For logs:** the cluster runs **Loki + Alloy** (Alloy DaemonSet tailing
  `/var/log/pods`), and Grafana has a **Loki datasource** named `Loki` with uid
  **`loki`** (the assets reference that uid). Confirm with the introspection
  below.

## (Optional) Verify the cluster

Skip if you already know the cluster matches the standard install. Run when you
have `kubectl` access. Replace `<ns>` with the monitoring namespace (often
`monitoring`).

1. **Metrics — operator + CRD + selection scope.** The ServiceMonitor is
   ignored without the operator/CRD, and a non-empty selector means the chart's
   labels must match:
   ```bash
   kubectl get crd servicemonitors.monitoring.coreos.com
   kubectl get prometheus -A \
     -o custom-columns=NAME:.metadata.name,SM_SELECTOR:.spec.serviceMonitorSelector,SM_NS_SELECTOR:.spec.serviceMonitorNamespaceSelector
   ```
   Empty/null selectors = "match all" (the default this skill expects).

2. **Dashboard sidecar.** The Grafana sidecar turns a labeled ConfigMap into a
   live dashboard. Confirm it runs and learn its watched label (default
   `grafana_dashboard: "1"`, all namespaces):
   ```bash
   kubectl get pods -A -l app.kubernetes.io/name=grafana \
     -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{" "}{end}{"\n"}{end}'
   # then inspect the grafana-sc-dashboard sidecar env:
   kubectl -n <ns> set env deploy/<grafana-deploy> --list | grep -i 'LABEL\|NAMESPACE'
   ```

3. **Logs — Loki datasource + Alloy.** Confirm the Loki datasource uid is
   `loki` and Alloy is shipping:
   ```bash
   kubectl -n <ns> get pods | grep -E 'loki|alloy'   # expect 1 loki + 1 alloy PER node
   # apps currently logging (read the datasource uid from prom-stack-values.yaml
   # additionalDataSources, or via the Grafana datasources API):
   kubectl -n <ns> port-forward svc/loki 3100:3100 &
   curl -s 'http://localhost:3100/loki/api/v1/label/app/values' | jq
   ```
   If the datasource uid is **not** `loki`, change it in `assets/logs.json`
   (the `datasource` template var's `value`) and in
   `assets/golden-signals-errors-panel.json` (hardwired `"uid": "loki"`).

If any check disagrees with the defaults, adapt the assets (labels / namespace
/ datasource uid) before checking them in — otherwise a dashboard or scrape
target silently won't appear.

## Placeholders

The bundled assets use placeholders. Resolve them from the target repo before
writing files:

| Placeholder | Meaning | How to determine |
|---|---|---|
| `__APP__` | Service identity — the Prometheus `service.name` **and** the Loki `app` label | e.g. `core-api`, `searchsync-worker`. Must equal the pod's `app.kubernetes.io/name` label so the `app` Loki label matches. |
| `__CHART__` | Helm chart name / `_helpers.tpl` prefix | the `name:` in `helm/<app>/Chart.yaml`; used in `include "<chart>.fullname"` |
| `__CONTAINER__` | Container name in the Deployment | from the chart's `deployment.yaml` (metrics dashboard saturation panels) |
| `__POD_PREFIX__` | Pod name prefix for saturation queries | usually `<release>-<chart>`; confirm via `kubectl get pods` |
| `__JOB__` | Prometheus `job` label for the scraped service | defaults to the **Service name** the ServiceMonitor selects (the chart fullname). Confirm after deploy with `label_values(http_server_request_duration_seconds_count, job)` in Grafana Explore |

`__NAMESPACE__` is **not** a placeholder — dashboards take namespace as a
Grafana template variable so one dashboard serves both `dev` and `prod`.

---

## A. Metrics — golden signals

### A1. Instrument the .NET service (web project)

1. Add the OpenTelemetry packages from `assets/csproj.snippet.xml` to the web
   project's `.csproj` (keep the versions unless the repo pins a newer OTel line).
2. In `Program.cs`, apply `assets/Program.cs.snippet`:
   - In non-Development, make Kestrel listen on **8080** (public) **and 9090**
     (metrics-only).
   - Register `AddOpenTelemetry().WithMetrics(...).AddPrometheusExporter()` with
     AspNetCore + HttpClient + Runtime instrumentation.
   - Map the scrape endpoint scoped to the metrics port:
     `app.MapPrometheusScrapingEndpoint().RequireHost("*:9090");`
   The 8080/9090 split keeps `/metrics` off the public ingress/tunnel — only the
   in-cluster ServiceMonitor reaches 9090. Adapt to the repo: match its
   `serviceName`, its existing Kestrel config, and don't duplicate an existing
   `AddOpenTelemetry()`.

### A2. Wire metrics into the Helm chart

1. **Service** — merge `assets/service-metrics-port.snippet.yaml` into
   `templates/service.yaml` (a second named port `metrics`, 9090). Don't route
   it from the ingress/tunnel.
2. **ServiceMonitor** — copy `assets/servicemonitor.yaml` to
   `templates/servicemonitor.yaml`, replacing `__CHART__`.
3. **Dashboard JSON** — copy `assets/golden-signals.json` to
   `helm/<app>/dashboards/golden-signals.json`, substituting `__APP__`,
   `__JOB__`, `__POD_PREFIX__`, `__CONTAINER__`. (It's already tagged
   `golden-signals`, so the hub lists it automatically.)
4. **Dashboard ConfigMap** — copy `assets/grafana-dashboard.yaml` to
   `templates/grafana-dashboard.yaml` (`__CHART__` + `__APP__` substitution). It
   embeds both the golden-signals and logs dashboards, each gated by a toggle.
5. **values.yaml** — add the toggles from `assets/values.snippet.yaml`. Keep
   `serviceMonitor.enabled` and `dashboards.goldenSignals` on by default.

---

## B. Logs — Serilog JSON → Loki

The hard requirement is small: **emit single-line JSON to stdout with a `level`
field.** Service identity is the `app` Loki label (not the body), and level
filtering uses `detected_level`, so apps don't need byte-identical level
strings — just valid JSON per line.

### B1. Instrument the .NET service

1. Add the Serilog packages from `assets/serilog.csproj.snippet.xml` — the
   **ASP.NET** set (`Serilog.AspNetCore` + `Serilog.Expressions`) or, for a
   worker/console host, the **worker** set in the same file's comment.
2. Wire Serilog:
   - **ASP.NET Core** → apply `assets/Program.cs.serilog.snippet`
     (`builder.Host.UseSerilog(...)` + `app.UseSerilogRequestLogging()`).
   - **Worker / console** (`Host.CreateApplicationBuilder`) → apply
     `assets/Worker.serilog.snippet` (`builder.Services.AddSerilog(...)`).
   Both write to Console with the same `ExpressionTemplate`:
   `{ {timestamp: @t, level: @l, message: @m, exception: @x, ..@p} }`. `..@p`
   spreads structured properties as top-level JSON keys (so
   `LogInformation("done for {TenantId}", id)` yields a queryable `TenantId`);
   `@x` folds multi-line exceptions onto ONE line (a raw `Console.WriteLine`
   stack trace would become N separate Loki lines — avoid those).
   - If the app already logs JSON via another path (some Go/Ory services do),
     it may already be compliant — don't double-instrument. The .NET default
     console logger is multi-line text and is **not** compliant.

### B2. Ensure the `app` pod label

Alloy derives the `app` Loki label from the pod's `app.kubernetes.io/name` (or
`app`) label. Confirm the chart's Deployment **pod template** carries it —
standard `_helpers.tpl` `selectorLabels` already include
`app.kubernetes.io/name`, so most charts comply. Verify it equals `__APP__`:
```bash
helm template helm/<app> | yq 'select(.kind=="Deployment").spec.template.metadata.labels'
```
If it's missing or differs from `__APP__`, fix the chart's pod labels — else
the logs land under the wrong (or no) `app` label and the dashboard's
`{app="__APP__"}` queries return nothing.

### B3. Per-app logs dashboard (if selected)

1. Copy `assets/logs.json` to `helm/<app>/dashboards/logs.json`, substituting
   `__APP__`. It's tagged `logs`, so the hub lists it automatically.
2. In `values.yaml`, set `dashboards.logs: true` (the `grafana-dashboard.yaml`
   ConfigMap from A2 then embeds it under a unique `__APP__-logs.json` key).
   If you did metrics already the ConfigMap template is in place; if logs-only,
   copy `assets/grafana-dashboard.yaml` now.

### B4. Errors panel on the golden-signals dashboard (if selected)

Append the panel in `assets/golden-signals-errors-panel.json` to the
golden-signals dashboard's `panels` array (substitute `__APP__`; mind the
comma). It hardwires the Loki datasource uid `loki` because that dashboard's
`${datasource}` variable points at Prometheus. Bump its `gridPos.y` if you've
added rows. Now one dashboard answers both "is it slow/erroring?" and "what's
the error?".

---

## C. Observability hub (once per cluster)

A single Grafana home page listing every app's dashboards. **Not** part of an
app chart — it lives in the infra/promstack repo next to `prom-stack-values.yaml`.

1. If `assets/observability-home.yaml`'s ConfigMap (or equivalent) already
   exists in the cluster, **stop** — new apps appear automatically via the
   `logs` / `golden-signals` tags the per-app dashboards already carry. Nothing
   to edit.
2. Otherwise apply `assets/observability-home.yaml` (a ConfigMap labeled
   `grafana_dashboard: "1"` with `dashlist` panels that auto-populate by tag).
3. Make it the org home in `prom-stack-values.yaml` so everyone lands on it:
   ```yaml
   grafana.ini:
     dashboards:
       default_home_dashboard_path: /tmp/dashboards/materia-observability-home.json
   ```
   The path must match the ConfigMap's **data key** exactly — the sidecar
   writes each dashboard to `/tmp/dashboards/<data-key>`. Let the promstack
   pipeline apply the values change.

---

## Verify

- `helm template helm/<app>` renders without error and includes the pieces you
  added (ServiceMonitor, dashboard ConfigMap with the expected `__APP__-*.json`
  keys).
- **Metrics:** in Grafana the **"`__APP__` — Golden Signals"** dashboard appears
  and panels populate once traffic flows. If panels read "No data", the
  `__JOB__` value is almost always the culprit — check
  `label_values(http_server_request_duration_seconds_count, job)` and confirm
  the target is `UP` in Prometheus (`/targets`).
- **Logs:** in **Explore → Loki**, `{app="__APP__"}` over the last 15 min shows
  live JSON lines; add `| json` to confirm structured fields parse. The
  **"`__APP__` — Logs"** dashboard populates. If `{app="__APP__"}` is empty,
  the pod's `app` label doesn't match `__APP__` (B2) or the app isn't emitting
  JSON yet (B1); if `| json` parses nothing, a line isn't valid JSON — usually a
  multi-line stack trace, fix the formatter.
- **Hub:** logging in lands on **"Materia — Observability Home"**, and the new
  app shows under the Logs / Golden Signals lists (it can take a refresh for the
  sidecar to load a newly-applied dashboard).

## What the dashboards show

**Golden signals:** latency p50 + p99 per route, traffic (req/min) per route,
5xx errors per route, CPU/memory saturation as a % of each pod's limit — plus,
optionally, a live error-log stream at the bottom. `http.route` comes free from
ASP.NET Core route templates, so `api/patients/{id}` collapses to one series.

**Logs:** total lines, error and warning counts, log-volume-by-level
(stacked bars, error=red/warn=orange), a live "Recent errors" stream, and an
"All logs" stream filtered by a `$search` regex textbox. App-specific panels
(cache hit ratio, custom counters, business fields via `| json`) can be appended
later — the OTel setup reserves a `__APP__.*` meter namespace and every
structured Serilog property is already queryable in Loki.
