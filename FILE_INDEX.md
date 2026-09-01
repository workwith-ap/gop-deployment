# File Index — everything we've built, by component

Total: 36 files. This is the map when RUNBOOK.md's steps stop making
sense and you need to know "where does X actually live."

---

## UI access / ingress — one subdomain per app

Real browser UIs exist for **Grafana, Prometheus, and Thanos Query**,
each on its own subdomain. Loki and Telegraf have no UI — their
subdomains route to raw API/metrics endpoints, useful for debugging with
curl but not something you browse day to day.

| File | What it is |
|---|---|
| `k8s/ingress/00-namespace.yaml` | Creates `gop-ingress` |
| `k8s/ingress/01-basicauth-secret.yaml` | Shared htpasswd secret, reused across all five apps — **placeholder value, must be replaced** |
| `k8s/ingress/02-middleware.yaml` | Traefik `Middleware`: shared basic auth |
| `k8s/ingress/03-ingressroute-grafana.yaml` | `grafana.tyo7.digitaledgedc.net` — real UI |
| `k8s/ingress/04-ingressroute-prometheus.yaml` | `prometheus.tyo7.digitaledgedc.net` — real UI, TYO7 only (other DCs route through Thanos Query instead) |
| `k8s/ingress/05-ingressroute-thanos.yaml` | `thanos.tyo7.digitaledgedc.net` — real UI, the global cross-DC view |
| `k8s/ingress/06-ingressroute-loki.yaml` | `loki.tyo7.digitaledgedc.net` — API only, no UI; use Grafana's Explore tab instead for actually reading logs |
| `k8s/ingress/07-ingressroute-telegraf.yaml` | `telegraf.tyo7.digitaledgedc.net` — raw `/metrics` text, optional, TYO7 only |

Grafana and Prometheus's charts have their own native `ingress:` blocks
turned **off** on purpose (`values/base/grafana-values.yaml`,
`values/base/prometheus-values.yaml`) — routing goes through these
hand-rolled IngressRoutes instead of a second, competing ingress per app.
Each of the five hostnames needs its own DNS record pointed at your
existing Traefik entrypoint — see RUNBOOK.md Step 2b.

---


## Thanos (namespace: gop-thanos) — hub only, hand-rolled manifests

No Helm chart — applied directly with `kubectl apply -f`.

| File | What it is |
|---|---|
| `k8s/thanos-hub/00-namespace.yaml` | Creates `gop-thanos` |
| `k8s/thanos-hub/01-objstore-secret.yaml` | S3 credentials pointing at SeaweedFS — **you must fill in real keys before applying** |
| `k8s/thanos-hub/02-thanos-receive.yaml` | StatefulSet + 2 Services — the ingest endpoint every site's Prometheus remote_writes to |
| `k8s/thanos-hub/03-thanos-store.yaml` | StatefulSet + headless Service — serves historical blocks from SeaweedFS to Query |
| `k8s/thanos-hub/04-thanos-compactor.yaml` | Deployment (replicas: 1, hard rule) + PVC — compaction/downsampling/retention |
| `k8s/thanos-hub/05-thanos-query.yaml` | Deployment + Service — the single query surface, this is what Grafana talks to |

## Prometheus (namespace: gop-prometheus) — per DC, official chart

| File | What it is |
|---|---|
| `values/base/prometheus-values.yaml` | Shared config: local 6h retention, remote_write target, scrapes Telegraf on port 9273 |
| `values/sites/<dc>.yaml` | Per-DC overlay: `external_labels.dc`, remote_write URL if a separate cluster |
| `prometheus-rules/th-sensor-staleness.yaml` | Alert rules — not yet wired into Prometheus config, that's the open item in RUNBOOK.md Step 6 |

Deployed via `helm install prometheus-<dc> prometheus-community/prometheus -f values/base/prometheus-values.yaml -f values/sites/<dc>.yaml`

## Telegraf (namespace: gop-telegraf) — per DC, official chart

| File | What it is |
|---|---|
| `values/base/telegraf-values.yaml` | Shared config: exposes `/metrics` on 9273 for Prometheus, references sensor input files |
| `sensors/bms-temp-humidity.conf` | Generic UDP/JSON input — use this for any site NOT using RLE WIFI-TH |
| `sensors/rle-wifi-th.conf` | RLE-specific input — **stubbed, blocked on payload format confirmation**, TYO6 only |
| `sensors/telegraf-bms-udp-service.yaml` | Fallback raw NodePort Service (UDP/30094 generic sites, or UDP/3184 for TYO6 — see below) if the chart's `extraServices` isn't supported by your pinned version |
| `sensors/_template-new-sensor.conf` | Copy this when onboarding a new sensor type (e.g. water leak) |
| `values/sites/<dc>.yaml` | Per-DC overlay: `global_tags.dc`, `site_name` |

Deployed via `helm install telegraf-<dc> influxdata/telegraf -f values/base/telegraf-values.yaml -f values/sites/<dc>.yaml`

## Grafana (namespace: gop-grafana) — hub only, official chart

| File | What it is |
|---|---|
| `values/base/grafana-values.yaml` | Datasource = Thanos Query, Traefik ingress + basic auth |

Deployed via `helm install grafana grafana/grafana -f values/base/grafana-values.yaml`

## Loki (namespace: gop-loki) — hub only, official chart

| File | What it is |
|---|---|
| `values/base/loki-values.yaml` | Single-binary mode, backed by the SeaweedFS `loki` bucket via the `loki-rw` scoped identity. No Promtail/Alloy log-shipping agents yet — this gets Loki itself up and queryable, log collection from the 47 DCs is a separate future step. |

Needs a manually-created `loki-s3-credentials` Secret before deploying —
see RUNBOOK.md Step 2. Deployed via `helm install loki grafana/loki -f values/base/loki-values.yaml`.
Queried through Grafana's Explore tab (datasource already wired in
`grafana-values.yaml`), not a standalone UI.

---

## Per-DC site files (used by Prometheus AND Telegraf simultaneously)

| File | DC | Status |
|---|---|---|
| `values/sites/tyo7.yaml` | TYO7 (hub site) | ready |
| `values/sites/jkt1.yaml` | JKT1 | ready, needs real basic-auth password filled in |
| `values/sites/tyo6.yaml` | TYO6 | blocked — see below |
| `values/sites/_template-site.yaml` | template | copy for DC #4 onward |

## Orchestration / glue (not component-specific)

| File | What it is |
|---|---|
| `helmfile.yaml` | Ties Prometheus + Telegraf + Grafana releases together per environment (per DC + hub) |
| `RUNBOOK.md` | The actual step-by-step commands, in order |
| `README.md` | Architecture overview, diagram, why-Receive-not-Sidecar reasoning |
| `scripts/deploy-hub.sh` | Runs Thanos manifests + Grafana chart, in order, once |
| `scripts/deploy-site.sh <dc>` | Runs Prometheus + Telegraf chart for one DC |
| `scripts/onboard-new-dc.sh <code> <name> <region>` | Scaffolds a new `values/sites/<dc>.yaml` from the template |

## TYO6-specific (not a component — site investigation artifacts)

| File | What it is |
|---|---|
| `tyo6-th-sensor-ips.txt` | The 75 real sensor IPs, extracted from your uploaded mapping file |
| `tyo6-firewall-request.md` | Ready-to-submit firewall ticket using those real IPs |

---

## What's actually blocking, right now

1. **Thanos objstore secret** — `k8s/thanos-hub/01-objstore-secret.yaml` has placeholder keys, must be filled before Step 1 of the runbook works at all.
2. **TYO6 firewall + RLE payload format** — blocks Telegraf deployment for TYO6 only. Every other DC is unaffected.
3. **jkt1 basic-auth password** — placeholder in `values/sites/jkt1.yaml`.
4. **Ingress basic-auth secret** — `k8s/ingress/01-basicauth-secret.yaml` has a placeholder htpasswd value; all five app subdomains are reachable with a fake credential until it's replaced.
5. **Loki's S3 credentials secret** — must be created manually (`loki-s3-credentials` in `gop-loki`) before `helmfile -e hub -l name=loki apply` will come up healthy; see RUNBOOK.md Step 2 for the exact command. Loki itself IS built now (`values/base/loki-values.yaml`) — this is just the one manual credential step charts can't do for you.
