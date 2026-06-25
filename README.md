# james-skills

Agent skills for software engineering, per my opinions.

Each skill is a self-contained directory under [`skills/`](skills/) with a
`SKILL.md` and any bundled assets. They're written for [Claude Code](https://claude.com/claude-code)
(and any agent that follows the [Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
format).

## Installation

Skills are discovered from `~/.claude/skills/` (available in every project) or a
repo's `.claude/skills/` (scoped to that repo). Pick one.

### Option A — install for all your projects (personal)

```bash
git clone https://github.com/jreate/james-skills.git ~/src/james-skills

mkdir -p ~/.claude/skills
# symlink each skill so `git pull` keeps them current
ln -s ~/src/james-skills/skills/dotnet-observability ~/.claude/skills/dotnet-observability
```

### Option B — install into a single repo (shared with your team)

```bash
mkdir -p .claude/skills
cp -R /path/to/james-skills/skills/dotnet-observability .claude/skills/
git add .claude/skills && git commit -m "Add dotnet-observability skill"
```

Committing under `.claude/skills/` means anyone who clones the repo — or any
agent run in CI — gets the skill automatically.

### Verify

Start Claude Code and run `/skills` (or ask "what skills do you have?"). The
skill name from each `SKILL.md` should appear. Skills load on demand — the agent
invokes one when your request matches its `description`, so no extra wiring is
needed.

---

## Skills

### dotnet-observability

Wires a **.NET** service (ASP.NET Core **or** a worker/console host) into the
Grafana observability stack — **metrics and/or logs** — entirely through the
service's **Helm chart**. Two independent, opt-in halves:

- **Metrics (golden signals).** OpenTelemetry Prometheus metrics on a dedicated
  metrics-only port (9090), a `ServiceMonitor` for `kube-prometheus-stack`, and
  a per-app **four-golden-signals** dashboard — **latency, traffic, errors,
  saturation** per `http.route`.
- **Logs (Loki).** Serilog emits one single-line JSON object per event to
  stdout; the cluster's Alloy DaemonSet tails it into Loki; a per-app **logs
  dashboard** shows volume-by-level, error/warning counts, and live error +
  all-log streams. Optionally an errors-log panel is appended to the
  golden-signals dashboard.

It also provisions, **once per cluster**, the Grafana **observability hub** home
page — a landing page whose `dashlist` panels auto-populate by tag, so every
instrumented app appears with no edits.

It touches only the **application repo** (the `.csproj`, `Program.cs`/worker,
and `helm/<app>/`), plus the one-time hub in the infra/promstack repo. The
cluster side (`kube-prometheus-stack` for metrics, Loki + Alloy for logs) is
assumed to already exist. The skill **asks first** which pieces to apply — "the
works" or any subset (metrics-only and logs-only are both fine).

**Use it when** a .NET service should report golden signals or appear in
Grafana/Loki, or someone asks for a `ServiceMonitor`, a `/metrics` endpoint,
structured JSON logging, or a logs dashboard on a .NET service.

#### What it adds

| Layer | File(s) |
|---|---|
| .NET — metrics | OpenTelemetry package refs in `.csproj`; OTel + Prometheus exporter in `Program.cs`, Kestrel on `8080` (public) + `9090` (metrics-only) |
| .NET — logs | Serilog package refs; `UseSerilog`/`AddSerilog` with a JSON `ExpressionTemplate` to stdout (ASP.NET or worker) |
| Helm — Service | second named port `metrics` (9090) |
| Helm — discovery | `templates/servicemonitor.yaml` (prometheus-operator CRD) |
| Helm — dashboards | `templates/grafana-dashboard.yaml` (ConfigMap) + `dashboards/golden-signals.json` and/or `dashboards/logs.json` |
| Helm — toggles | `serviceMonitor.enabled`, `dashboards.{enabled,goldenSignals,logs}`, `service.metricsPort` in `values.yaml` |
| Cluster — hub | `observability-home.yaml` (org-home dashlist) + `grafana.ini: default_home_dashboard_path` — once per cluster |

#### Cluster expectations (for automatic CI/CD pickup)

The dashboard is picked up automatically when the chart is checked in and your
CI/CD deploys it with Helm — **no manual Grafana import** — provided the target
cluster meets these expectations:

1. **`kube-prometheus-stack` is installed** (Prometheus, Grafana, and the
   prometheus-operator). The skill only adds CRDs the operator already knows how
   to consume.
2. **The prometheus-operator watches ServiceMonitors in all namespaces.** This
   is the kube-prometheus-stack default. If your install scopes the operator to
   specific namespaces, the app's namespace must be in that set, and the
   `ServiceMonitor`'s labels must match the operator's `serviceMonitorSelector`
   (the chart applies the standard chart labels — align them if your install
   filters by label).
3. **The Grafana sidecar auto-discovers dashboard ConfigMaps** labeled
   `grafana_dashboard: "1"` in any namespace (the kube-prometheus-stack default,
   `kiwigrid/k8s-sidecar`). The skill applies that label, so a `helm upgrade`
   surfaces the dashboard within a sidecar refresh interval. Because the sidecar
   flattens all dashboards into one directory, the embedded JSON filename is
   prefixed with the app name (`<app>-golden-signals.json`) to stay globally
   unique.
4. **Your CD applies the chart with Helm** (`helm upgrade --install`, Argo CD,
   Flux, etc.). The `ServiceMonitor` and dashboard `ConfigMap` are plain chart
   templates, so they ship on the next sync — committing the chart change is the
   only action required.

For logs, the same applies via Loki + Alloy: Grafana needs a **Loki datasource**
named `Loki` with uid `loki` (the assets reference that uid), and Alloy derives
the `app` Loki label from each pod's `app.kubernetes.io/name`.

Environments **without** prometheus-operator can opt out by setting
`serviceMonitor.enabled=false` and `dashboards.enabled=false` in that
environment's values.

#### The dashboards

**Golden signals:** latency p50 + p99 per route, traffic (req/min) per route,
5xx errors per route, CPU/memory saturation as a % of each pod's limit — plus an
optional error-log stream at the bottom. **Logs:** total/error/warning counts,
log-volume-by-level (stacked bars), a "Recent errors" stream, and an "All logs"
stream with a `$search` regex box. Namespace is a Grafana template variable, so
one dashboard serves both `dev` and `prod`.

See [`skills/dotnet-observability/SKILL.md`](skills/dotnet-observability/SKILL.md)
for the full step-by-step, the "ask what to instrument" menu, the placeholder
resolution table, and the verification checklist.

---

## Adding a new skill

1. Create `skills/<skill-name>/SKILL.md` with YAML frontmatter (`name`,
   `description`) followed by the instructions. Put any templates under
   `skills/<skill-name>/assets/`.
2. Write a tight `description` — it's what the agent matches against to decide
   when to load the skill.
3. Add a `### <skill-name>` section to **Skills** above, following the same
   shape (summary → what it adds → expectations → details link).
