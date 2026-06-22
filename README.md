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
ln -s ~/src/james-skills/skills/golden-signals-dashboard ~/.claude/skills/golden-signals-dashboard
```

### Option B — install into a single repo (shared with your team)

```bash
mkdir -p .claude/skills
cp -R /path/to/james-skills/skills/golden-signals-dashboard .claude/skills/
git add .claude/skills && git commit -m "Add golden-signals-dashboard skill"
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

### golden-signals-dashboard

Instruments an **ASP.NET Core** backend with OpenTelemetry Prometheus metrics
and provisions a **four-golden-signals** Grafana dashboard — **latency,
traffic, errors, saturation** (per `http.route`) — entirely through the
service's **Helm chart**.

It touches only the **application repo** (the `.csproj`, `Program.cs`, and
`helm/<app>/`). The cluster side (Prometheus, Grafana, the operator) is assumed
to already exist. On deploy:

- OpenTelemetry auto-instrumentation exposes Prometheus metrics on a dedicated
  metrics-only port (9090), kept off the public ingress.
- A `ServiceMonitor` makes the cluster's `kube-prometheus-stack` scrape it.
- A dashboard `ConfigMap` ships a per-app Grafana dashboard that travels with
  the app.

**Use it when** a backend service should report golden signals / appear in
Grafana, or someone asks for a `ServiceMonitor`, a `/metrics` endpoint, or
"monitoring" on a .NET service.

#### What it adds

| Layer | File(s) |
|---|---|
| .NET instrumentation | OpenTelemetry package refs in `.csproj`; OTel + Prometheus exporter wired in `Program.cs`, with Kestrel listening on `8080` (public) and `9090` (metrics-only) |
| Helm — Service | second named port `metrics` (9090) |
| Helm — discovery | `templates/servicemonitor.yaml` (prometheus-operator CRD) |
| Helm — dashboard | `templates/grafana-dashboard.yaml` (ConfigMap) + `dashboards/golden-signals.json` |
| Helm — toggles | `serviceMonitor.enabled`, `dashboards.enabled`, `service.metricsPort` in `values.yaml` |

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

Environments **without** prometheus-operator can opt out by setting
`serviceMonitor.enabled=false` and `dashboards.enabled=false` in that
environment's values.

#### The dashboard

Latency p50 + p99 per route, traffic (req/min) per route, 5xx errors per route,
and CPU/memory saturation as a % of each pod's limit. Namespace is a Grafana
template variable, so one dashboard serves both `dev` and `prod`.

See [`skills/golden-signals-dashboard/SKILL.md`](skills/golden-signals-dashboard/SKILL.md)
for the full step-by-step, placeholder resolution table, and verification
checklist.

---

## Adding a new skill

1. Create `skills/<skill-name>/SKILL.md` with YAML frontmatter (`name`,
   `description`) followed by the instructions. Put any templates under
   `skills/<skill-name>/assets/`.
2. Write a tight `description` — it's what the agent matches against to decide
   when to load the skill.
3. Add a `### <skill-name>` section to **Skills** above, following the same
   shape (summary → what it adds → expectations → details link).
