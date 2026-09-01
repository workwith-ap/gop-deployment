# GOP Deployment — Telegraf → Prometheus → Thanos → Grafana

## Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │  TYO7 — HUB (central, deployed once)         │
                    │                                               │
                    │  Thanos Receive (HA, ingests remote_write)   │
                    │       │              ns: gop-thanos           │
                    │       ▼                                       │
                    │  SeaweedFS bucket "thanos" (already live)    │
                    │       ▲                                       │
                    │       │                                       │
                    │  Thanos Store Gateway ──┐                     │
                    │  Thanos Compactor        │                    │
                    │       │                  │                    │
                    │       ▼                  ▼                    │
                    │  Thanos Query ◄──────────┘                    │
                    │       ▲                                       │
                    │       │                                       │
                    │  Grafana (ns: gop-grafana)                    │
                    └───────┬───────────────────────────────────────┘
                            │ remote_write (push, over WAN)
        ┌───────────────────┼───────────────────┐
        │                    │                    │
┌───────▼────────┐  ┌────────▼────────┐  ┌────────▼────────┐
│ DC: TYO7        │  │ DC: JKT1        │  │ DC: ...47 more  │
│ Prometheus      │  │ Prometheus      │  │ Prometheus      │
│ ns: gop-prometheus│ │ ns: gop-prometheus│ ns: gop-prometheus│
│      ▲          │  │      ▲          │  │      ▲          │
│  scrapes /metrics│  │  scrapes /metrics│ │  scrapes /metrics│
│      │          │  │      │          │  │      │          │
│  Telegraf        │  │  Telegraf        │ │  Telegraf        │
│  ns: gop-telegraf│  │  ns: gop-telegraf│  │  ns: gop-telegraf│
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

Namespace-per-component (`gop-thanos`, `gop-grafana`, `gop-prometheus`,
`gop-telegraf`) is deliberate — when something breaks, `kubectl -n <ns>`
narrows the search immediately instead of grepping a shared namespace.

**Why Thanos Receive, not Sidecar+Query federation:** federation needs Tokyo
to hold a live gRPC connection to every remote site's Prometheus. At 47
DCs over WAN links of varying quality, that's 47 things that can silently
go stale. Receive flips it: each site *pushes* to Tokyo. If a link drops,
Prometheus's WAL buffers and retries — no query-side blast radius.

## Two separate timelines: network prerequisites vs. software deploy order

These are easy to conflate but should run on different clocks.

**Network/firewall request — start this immediately, per DC, in parallel with everything else.**
BMS devices need to reach Telegraf's sensor-facing listener (UDP/30094,
NodePort — see `sensors/telegraf-bms-udp-service.yaml`). That's a firewall
rule request to a network/security team, with its own approval lead time
that has nothing to do with whether any pod is running yet. File it the
same day you start onboarding a DC — see the checklist in
`values/sites/_template-site.yaml`. This is very likely your actual
long pole across 47 sites, not the Helm deploys below.

**Software deploy order — run once for the hub, then per-DC for the rest.**

| Order | Component | Depends on | Why this order |
|---|---|---|---|
| 1 | Thanos Receive + Store + Compactor + Query (hub, TYO7 only) | SeaweedFS `thanos` bucket (already live) | Nothing else needs to exist yet; gives every future site a stable ingest endpoint from day one |
| 2 | Prometheus (per DC) | Thanos Receive endpoint reachable | Pre-wired with `remote_write` so it never needs reconfiguring later |
| 3 | Telegraf (per DC) | Nothing (Prometheus scrapes it, pull model) | Can deploy in any order relative to step 2 — it just sits exposing `/metrics` until scraped |
| 4 | Grafana (hub, TYO7 only) | Thanos Query | Deployed last so there's real data to point it at immediately |

Note that Telegraf can go up (pod running, service exposed) at any point
in this table without breaking anything — but it's functionally useless
until the firewall rule above is actually approved and live, since no
sensor packet can reach it until then. Don't let "Telegraf's pod is
running" be mistaken for "Telegraf is receiving data" — check the
firewall ticket status, not just `kubectl get pods`.

## Adding DC #4 through #47

Copy `values/sites/_template-site.yaml` → `values/sites/<dc-code>.yaml`,
fill in the ~6 site-specific fields, add one line to `helmfile.yaml`'s
environments block, run `scripts/deploy-site.sh <dc-code>`. No chart
changes, no hub changes.

## Adding a new sensor type (e.g. water leak)

Copy `sensors/_template-new-sensor.conf` → `sensors/water-leak.conf`,
define the input plugin + tags, reference it in the site's Telegraf
values under `extraConfigFiles`. Prometheus and Thanos need zero changes
— they're schema-agnostic; new metric names just show up.
