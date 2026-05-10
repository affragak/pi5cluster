---
id: agent-cisco-monitoring
aliases: []
tags:
  - agent-cisco-monitoring
  - kubernetes
  - grafana
  - cisco
  - monitoring
---

# agent-cisco-monitoring

Rule: Commits
Always use conventional commits (e.g. feat:, fix:, docs:, chore:)
Never include any reference to AI agents or Copilot in commit messages, co-authorship, documentation, or any other files in this repository
Committing directly to main is okay in this repo

---

# AGENT.md — Cisco Network Observability (athena cluster)

This is a working maintainer's runbook for the Cisco router + ASA monitoring stack actually deployed on this cluster. It records the specific decisions, file paths, gotchas, and operational commands learned while building it.

## 1. Mission and scope

Centralise health, interface counters, security events, and ACL/threat logs from Cisco routers and ASAs (management network `10.3.19.0/24`) into the existing k3s observability stack — **Prometheus + Grafana + Loki**, all under FluxCD GitOps. NetFlow/IPFIX is intentionally **out of scope** for v1.

Non-goals:
- Don't modify Cisco device running configs from inside the cluster.
- Don't expose monitoring ports to the internet.
- Don't put device credentials in ConfigMaps, manifests, or git plaintext.

## 2. Stack actually in use

```
Cisco Devices (10.3.19.0/24)
  ├── SNMPv3 polling (interfaces, CPU, memory, ASA conn count)
  └── Syslog UDP/514 (ASA security events, link state, etc.)
        │
        ▼
  snmp_exporter ──► Prometheus (Probe CRDs)         (metrics)
  syslog-ng    ──► promtail ──► Loki                 (logs)
        │
        ▼
              Grafana (dashboard + alerting)
                    │
                    ▼  (Botkube polls Prometheus alert API)
                  Slack
```

Versions / images (pinned):
- `quay.io/prometheus/snmp-exporter:v0.26.0`
- `alpine:3.20` (init container for envsubst)
- kube-prometheus-stack chart (managed by Flux HelmRelease in `monitoring/`)
- Loki via `loki-stack` Helm chart (already deployed)
- Promtail via raw DaemonSet (`monitoring/controllers/base/loki-stack/promtail.yaml`)
- syslog-ng via raw Deployment (`monitoring/controllers/base/syslog-ng/`)

## 3. Reference values

| Thing | Value |
|---|---|
| Cluster | `athena` (single cluster) |
| Cluster nodes | `10.10.10.244` (athena, control-plane), `10.10.10.242` (nuc242), `10.10.10.243` (nuc243) |
| Cilium LB pool | `10.10.10.110–120` |
| syslog-ng LB IP | `10.10.10.119` (UDP/514, TCP/601) |
| Mgmt network | `10.3.19.0/24` |
| Vault server | `https://vault.uclab8.net` (mount `apps/`, KVv2) |
| ClusterSecretStore | `vault-backend-global` |
| Vault key for SNMP creds | `apps/snmp-cisco` (`username`, `auth_password`, `priv_password`) |
| SNMPv3 user | `prom_snmp` (SHA + AES128, authPriv) |
| snmp_exporter Service | `snmp-exporter.monitoring.svc.cluster.local:9116` |
| Loki datasource UID | `P8E80F9AEF21F6940` |
| Prometheus datasource UID | `prometheus` |
| Grafana admin password | `grafana123` (set in HelmRelease values; sensitive) |

## 4. File layout

```
monitoring/
├── controllers/
│   ├── base/
│   │   ├── kube-prometheus-stack/
│   │   │   ├── release.yaml                    # HelmRelease (incl. grafana values)
│   │   │   ├── kustomization.yaml              # configMapGenerator + kustomizeconfig
│   │   │   ├── kustomizeconfig.yaml            # rewrites valuesFrom name on hash
│   │   │   ├── kube-state-metrics-config.yaml  # existing
│   │   │   └── cisco-asa-grafana-alerts.yaml   # Grafana provisioned alert rules (this stack)
│   │   ├── loki-stack/
│   │   │   └── promtail.yaml                   # DaemonSet + ConfigMap (vendor=cisco match)
│   │   └── snmp-exporter/
│   │       ├── configmap.yaml                  # snmp.yml template (auths + 3 modules)
│   │       ├── deployment.yaml                 # alpine init -> envsubst -> snmp_exporter
│   │       ├── external-secret.yaml            # Vault apps/snmp-cisco -> Secret
│   │       ├── service.yaml                    # ClusterIP :9116
│   │       └── kustomization.yaml
│   └── athena/snmp-exporter/
│       └── kustomization.yaml                  # sets namespace: monitoring
└── configs/athena/snmp-exporter/
    ├── kustomization.yaml                      # sets namespace + dashboard configMap
    ├── probe-cisco-routers.yaml                # Probe CRD (module: cisco)
    ├── probe-cisco-asa.yaml                    # Probe CRD (module: cisco_asa)
    ├── prometheusrule.yaml                     # SNMP metric-based alerts
    └── dashboards/cisco-snmp.json              # Grafana dashboard (uid: cisco-snmp)
```

Wired into:
- `monitoring/controllers/athena/kustomization.yaml` (resources include `snmp-exporter`)
- `monitoring/configs/athena/kustomization.yaml` (resources include `snmp-exporter`)

## 5. Secret management — Vault + ESO

Single ExternalSecret (`monitoring/controllers/base/snmp-exporter/external-secret.yaml`) maps `apps/snmp-cisco` → in-cluster Secret `snmp-cisco-credentials` in the `monitoring` namespace.

Populate before first deploy:
```sh
vault kv put apps/snmp-cisco \
  username=prom_snmp \
  auth_password='<auth-pass>' \
  priv_password='<priv-pass>'
```

ESO refresh interval is 15 s. Verify:
```sh
kubectl -n monitoring get externalsecret snmp-cisco-credentials   # READY=True / SecretSynced
kubectl -n monitoring get secret snmp-cisco-credentials -o json | jq '.data | keys'
```

## 6. snmp_exporter — config rendering

snmp_exporter does **not** support env-var substitution in its config file. We handle this with an init container:

```
ConfigMap (snmp.yml template with ${SNMP_USERNAME} / ${SNMP_AUTH_PASSWORD} / ${SNMP_PRIV_PASSWORD})
        │
init container: alpine + apk add gettext, envsubst < /template/snmp.yml > /config/snmp.yml
        │
main container: --config.file=/config/snmp.yml
```

`snmp.yml` defines one auth (`cisco_v3`, SNMPv3 authPriv with SHA + AES128) and three modules:
- `if_mib` — interfaces only (sysUpTime, ifTable + ifXTable, errors, HC counters, ifName/ifAlias lookups)
- `cisco` — `if_mib` + `cpmCPUTotal5minRev` + `ciscoMemoryPool*` from **CISCO-MEMORY-POOL-MIB** (`1.3.6.1.4.1.9.9.48`) — for IOS / IOS-XE routers and switches
- `cisco_asa` — `if_mib` + `cpmCPUTotal5minRev` + memory from **CISCO-ENHANCED-MEMPOOL-MIB** (`1.3.6.1.4.1.9.9.221`) + `cfwConnectionStatValue` (firewall active connections)

**ASA needs the enhanced memory MIB** (`.221`), not the legacy one (`.48`) — the ASA returns `No Such Object` for the legacy OIDs. The `cisco_asa` module walks `1.3.6.1.4.1.9.9.221.1.1.1.1` and uses `cempMemPoolHCUsed` (.18) / `cempMemPoolHCFree` (.20) with `cempMemPoolName` (.3) as the lookup. Metric *names* are kept as `ciscoMemoryPoolUsed` / `ciscoMemoryPoolFree` with a `ciscoMemoryPoolName` label so the dashboard query and PrometheusRule are MIB-agnostic.

**Lookups format (snmp_exporter v0.20+ change)**: `lookups` are **per-metric**, not per-module. Each interface metric has nested `lookups` adding `ifName` and `ifAlias` labels via OIDs `1.3.6.1.2.1.31.1.1.1.1` and `1.3.6.1.2.1.31.1.1.1.18`. Wrong format = `field lookups not found in type config.plain` parse error.

## 7. Probe CRDs — passing module + auth to snmp_exporter

`probeSelectorNilUsesHelmValues: false` in the kube-prometheus-stack values means every Probe CRD cluster-wide is scraped — no label needed.

For SNMPv3 we have to pass the `auth` query parameter (because `snmp.yml` has more than one auth defined: `public_v2` and `cisco_v3`). The Probe CRD doesn't expose `auth` directly, so we relabel:

```yaml
spec:
  module: cisco              # becomes ?module=cisco
  prober:
    url: snmp-exporter.monitoring.svc.cluster.local:9116
    path: /snmp
  targets:
    staticConfig:
      static: [10.3.19.11, 10.3.19.12, 10.3.19.13, 10.3.19.14]
      labels:
        device_class: cisco_router
      relabelingConfigs:
        - targetLabel: __param_auth     # becomes ?auth=cisco_v3
          replacement: cisco_v3
```

Resulting scrape URL:
```
http://snmp-exporter.monitoring.svc:9116/snmp?target=10.3.19.X&module=cisco&auth=cisco_v3
```

ASA Probe is identical with `module: cisco_asa` and `static: [10.3.19.10]`.

## 8. PrometheusRule — SNMP metric-based alerts

`monitoring/configs/athena/snmp-exporter/prometheusrule.yaml` covers:

| Alert | Expression | severity |
|---|---|---|
| `CiscoDeviceUnreachable` | `up{job=~"cisco-.*"} == 0 for 3m` | critical |
| `CiscoInterfaceDown` | `ifOperStatus != 1 and ifAdminStatus == 1 for 2m` | warning |
| `CiscoHighCPU` | `cpmCPUTotal5minRev > 80 for 5m` | warning |
| `CiscoHighMemory` | `Used / (Used + Free) > 0.9 for 5m` | warning |
| `CiscoInterfaceErrors` | `rate(ifInErrors[5m]) + rate(ifOutErrors[5m]) > 1 for 10m` | warning |

These flow Prometheus → Botkube's prometheus source (NOT Alertmanager — see §13). Alertmanager has only the default `null` receiver; configure Botkube channel bindings if Slack delivery is wanted.

## 9. Grafana dashboard

UID: `cisco-snmp`. Bundled into ConfigMap `flux-grafana-dashboards-cisco` (with `disableNameSuffixHash: true` — see §12 for why).

Panel inventory:
- Top row: 4 stats (Devices Up/Down, Interfaces Up, ASA Active Conns).
- Middle: CPU 5-min avg, Memory utilisation (per device, per pool).
- Lower: Bandwidth in/out (`rate(ifHCInOctets[5m]) * 8`), Interface errors, Interface status table with value-mapped `ifOperStatus`.
- Collapsible **Cisco Device Logs** row: 3 stats (Total log lines, Errors severity ≤ 3, ASA ACL denies), log rate timeseries, 3 log panels (All / Errors / free-text Search).

Variables: `DS_PROMETHEUS`, `instance` (multi, from `up{job=~"cisco-.*"}`), `interface` (multi, from `ifOperStatus`), `DS_LOKI`, `device` (multi, from `label_values({vendor="cisco"}, routerboard)`), `search` (textbox).

All Loki panels filter by `{vendor="cisco", routerboard=~"$device"}` — see §11 for how that label is set.

## 10. Grafana alert rules (provisioned)

`monitoring/controllers/base/kube-prometheus-stack/cisco-asa-grafana-alerts.yaml` provides them via `grafana.alerting.<file>.yaml` Helm value. The kustomize `configMapGenerator` + `kustomizeconfig.yaml` `nameReference` rewrite means the hash-suffixed ConfigMap name is auto-substituted into the HelmRelease's `valuesFrom`.

Folder: `cisco-asa`. All evaluate every 1 min. Each rule's query passes the line through `regexp` to extract an `asa_code` capture group, then groups `by (routerboard, asa_code)` — every unique error code per device fires its own alert instance.

| UID | Title | Line filter | Extracted `asa_code` | Threshold | For |
|---|---|---|---|---|---|
| `cisco-asa-critical` | ASA critical / error events | `\|~ "%ASA-[0-3]-"` | any digits after `%ASA-[0-3]-` | `>0` | 1m |
| `cisco-asa-acl-deny-spike` | ASA ACL deny spike | `\|~ "%ASA-[0-7]-(106\|710003)"` | `106\d{3}\|710003` | `>100` | 5m |
| `cisco-asa-threat-detection` | ASA threat detection event | `\|~ "%ASA-4-7331(00\|02)"` | `7331(00\|02)` | `>0` | 1m |
| `cisco-asa-auth-failures` | ASA repeated auth failures | `\|~ "%ASA-[46]-(113015\|605004\|113006)"` | `113015\|605004\|113006` | `>5` | 5m |
| `cisco-asa-link-state-change` | ASA interface state change | `\|~ "%ASA-[0-5]-411[0-9]{3}"` | `411\d{3}` | `>0` | 0m |

All `noDataState: OK`. Labels `severity=…`, `team=network`. Annotation summary/description use `{{ $labels.asa_code }}`, `{{ $labels.routerboard }}`, and `{{ $values.A.Value }}` (the actual count — **not** `$values.B` which is the threshold-pass result `0`/`1`). Each description includes a copy-paste LogQL line for Grafana → Explore.

**Why the `cisco-asa-acl-deny-spike` matches `710003` and the 106 family**: ASA emits `%ASA-3-710003` (control-plane access denied) by default. The often-cited `%ASA-4-106023` only fires if the ACL entry has the `log` keyword — most ASAs don't have it set, so a rule matching only 106023 will see no data.

**`noDataState: OK` matters**: an earlier mistake had it as `NoData`, which makes Grafana fire a synthetic `DatasourceNoData` alert with empty labels (you'll see `summary = ... on [no value]` in Slack). Always `OK` unless missing data is itself meaningful.

**Helm `tpl` gotcha**: the Grafana subchart pipes alerting values through `tpl`. Any literal Grafana template syntax (`{{ $labels.x }}`, `{{ $values.A.Value }}`) must be escaped as `{{`{{ $labels.x }}`}}` so Helm emits them unchanged.

## 11. Promtail — vendor classification

Pipeline in `monitoring/controllers/base/loki-stack/promtail.yaml` syslog scrape job:

```yaml
relabel_configs:
  - source_labels: ['__syslog_message_hostname']
    target_label: 'routerboard'                # device hostname (or fallback IP)
  - source_labels: ['__syslog_connection_hostname']
    target_label: 'syslog_host'                # always the syslog-ng pod
pipeline_stages:
  - match:
      selector: '{job="syslog"} |~ "%[A-Z]+-[0-7]-"'
      stages:
        - static_labels:
            vendor: cisco
```

Lines matching the Cisco severity prefix get `vendor=cisco`. Mikrotik (`zeus`) doesn't match, so it stays unlabelled and gets filtered out of the Cisco panels.

After editing the ConfigMap, **restart the DaemonSet** — promtail does not auto-reload by default:
```sh
kubectl -n monitoring rollout restart daemonset/promtail-daemonset
```

Note: `routerboard` is from `__syslog_message_hostname`. If the device omits the HOSTNAME field in syslog, promtail falls back to the connection source IP — which is the syslog-ng pod IP (e.g. `10.42.2.58`). Fix on the device side (§14).

## 12. Dashboard ConfigMap pin

Grafana sidecar deletes the dashboard file if its source ConfigMap disappears (delete event), even if a replacement ConfigMap with the same dashboard arrives at the same time. Kustomize's hash-suffix on `configMapGenerator` triggers this exact race on every content change.

Fix: `disableNameSuffixHash: true` in `monitoring/configs/athena/snmp-exporter/kustomization.yaml`. ConfigMap name stays `flux-grafana-dashboards-cisco` across content changes; sidecar sees in-place update, not delete+create. Don't remove this flag.

## 13. Alert delivery — Botkube cloud config

`CONFIG_PROVIDER_ENDPOINT=https://api.botkube.io/graphql`. The running Botkube pod fetches `comm_config.yaml` (channel bindings, real Slack `xoxb-…`/`xapp-…` tokens) from the SaaS at startup, **overriding** anything in `infrastructure/controllers/base/botkube/values.yaml`. To add a Prometheus source binding to the Slack channel, edit it in **app.botkube.io**, not values.yaml. Plugin definitions (under top-level `sources:`) DO come from values.yaml — only the `communications.*.channels.*.bindings` block is overridden by cloud config.

Botkube polls the Prometheus alert API directly (`http://kube-prometheus-stack-prometheus.monitoring.svc:9090`), so PrometheusRule alerts reach Slack via Botkube — **not** through Alertmanager. Alertmanager keeps its `null` receiver. No webhook URL needed.

Grafana alerts go through a different path: Grafana contact point (manually set up in UI) → Botkube webhook → Slack. The Slack webhook URL is a UI-managed secret; not in git.

## 14. Cisco device commands

### IOS / IOS-XE routers and switches

```
! Permit cluster nodes for SNMP and syslog
ip access-list standard ACL-MONITORING
 permit 10.10.10.0 0.0.0.255

snmp-server view MONITORING-VIEW iso included
snmp-server group MONITORING v3 priv read MONITORING-VIEW access ACL-MONITORING
snmp-server user prom_snmp MONITORING v3 auth sha <auth-pass> priv aes 128 <priv-pass> access ACL-MONITORING

logging trap informational
logging facility local6
logging host 10.10.10.119 transport udp port 514
service timestamps log datetime msec    ! NB: do NOT add show-timezone (see §15)
```

### ASA

```
! ─── SNMPv3 ───
snmp-server group MONITORING v3 priv
snmp-server user prom_snmp MONITORING v3 auth sha <auth-pass> priv aes 128 <priv-pass>
! On ASA, snmp-server host doubles as the access list — list every cluster node IP:
snmp-server host management 10.10.10.242 version 3 prom_snmp
snmp-server host management 10.10.10.243 version 3 prom_snmp
snmp-server host management 10.10.10.244 version 3 prom_snmp

! ─── Syslog with proper hostname & timestamp ───
logging enable
logging timestamp                            ! or: logging timestamp rfc5424 (newer ASA)
logging device-id hostname                   ! → routerboard=ASA-1 (pick this OR ipaddress, not both)
! logging device-id ipaddress management     ! → routerboard=10.3.19.10 (mgmt IP instead of name)
logging trap informational
logging host management 10.10.10.119 udp/514
```

## 15. Field-tested gotchas

1. **ASA `snmp-server host` is the ACL.** ASA silently drops SNMP polls from any source IP not in `snmp-server host` lines. Cluster traffic egresses as the **node IP** (Cilium SNAT), not as a Service IP — list all 3 cluster nodes (`.242/.243/.244`). Cisco routers use the regular `access-list` ACL; `10.10.10.0/24` covers all nodes.
2. **`logging device-id hostname` is mandatory on ASA.** Without it the syslog HOSTNAME field is empty, promtail falls back to the connection source IP, and the device shows up in Loki as `routerboard=<syslog-ng pod IP>` instead of `routerboard=ASA-1`.
3. **Don't add `show-timezone` to the ASA timestamp service.** `service timestamps log datetime msec localtime show-timezone` produces `<pri>Mmm dd hh:mm:ss EET hostname …` — RFC-3164 parser eats `EET` as the HOSTNAME and you get a phantom `routerboard=EET` stream. Use plain `service timestamps log datetime msec` or `logging timestamp rfc5424`.
4. **Grafana PVC is RWO + Recreate strategy.** Default RollingUpdate hits Multi-Attach errors on every rollout. `grafana.deploymentStrategy.type: Recreate` (in HelmRelease values) accepts brief downtime but actually completes.
5. **Loki ingestion errors observed.** kube-prometheus-stack-operator pod logs have 16 labels (Loki limit 15) → 400s. Plus periodic 429 ingestion-rate-limit. Both are pre-existing, not caused by Cisco syslog volume, but worth knowing if logs go missing.
6. **Promtail does not hot-reload its ConfigMap.** Always `rollout restart daemonset/promtail-daemonset` after editing.
7. **Botkube does not hot-reload either.** Same restart treatment.
8. **The Helm `tpl` escape pattern** for Grafana templates: `{{`{{ $labels.x }}`}}` — don't forget when adding new alert rule annotations.
9. **ASA does not expose CISCO-MEMORY-POOL-MIB.** Querying `1.3.6.1.4.1.9.9.48.*` returns `No Such Object`. Use CISCO-ENHANCED-MEMPOOL-MIB (`1.3.6.1.4.1.9.9.221.*`) — `cempMemPoolHCUsed` (.18), `cempMemPoolHCFree` (.20), `cempMemPoolName` (.3). The `cisco_asa` snmp.yml module already does this; if you ever copy memory metrics from `cisco` to `cisco_asa`, swap the OIDs.
10. **ASA ACLs don't log denies by default.** `%ASA-4-106023` only fires when an ACL entry has the `log` keyword. The default ASA emits `%ASA-3-710003` (control-plane access denied) instead. Match both: `%ASA-[0-7]-(106\|710003)`.
11. **`noDataState: NoData` on a Loki rule = synthetic alert spam.** When the query returns no data, Grafana fires a `DatasourceNoData` alert with empty labels (`summary = … on [no value]`). Set `noDataState: OK` unless missing data is itself an alert condition.
12. **Use `$values.A.Value` in alert annotations, not `$values.B`.** `$values.B` is the threshold-pass result (`0` or `1`), not the count. Easy to confuse — the message reads "1 ACL denies in 5m" no matter the actual number.

## 16. Verification checklist

After any change:

```sh
# Manifests are syntactically valid
kubectl kustomize monitoring/controllers/athena/snmp-exporter/   >/dev/null
kubectl kustomize monitoring/configs/athena/snmp-exporter/       >/dev/null
kubectl kustomize monitoring/controllers/athena/                 >/dev/null
kubectl kustomize monitoring/configs/athena/                     >/dev/null

# Flux state
flux get kustomizations -A
flux get hr -n monitoring kube-prometheus-stack

# Cluster state
kubectl -n monitoring get externalsecret snmp-cisco-credentials   # READY=True
kubectl -n monitoring get pod -l app=snmp-exporter                 # 1/1 Running
kubectl -n monitoring get probe                                    # cisco-routers-snmp, cisco-asa-snmp
kubectl -n monitoring get prometheusrule cisco-snmp-alerts
kubectl -n monitoring get cm -l grafana_dashboard=1 | grep cisco

# Targets actually scraping
kubectl -n monitoring exec prometheus-kube-prometheus-stack-prometheus-0 -c prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/targets?state=active' | python3 -c "import json,sys; [print(t['labels']['job'], t['labels']['instance'], t['health']) for t in json.load(sys.stdin)['data']['activeTargets'] if 'cisco' in t['labels']['job']]"

# Loki has Cisco data
kubectl -n monitoring port-forward svc/loki 3100:3100 &
curl -sG localhost:3100/loki/api/v1/label/routerboard/values --data-urlencode 'query={vendor="cisco"}'

# Grafana sees provisioned alert rules
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80 &
curl -s -u "admin:grafana123" 'http://localhost:3000/api/v1/provisioning/alert-rules' \
  | python3 -c "import json,sys; [print(r['uid'],r['title']) for r in json.load(sys.stdin) if r.get('ruleGroup')=='cisco-asa']"
```

## 17. Common diagnostics

| Symptom | First check |
|---|---|
| Probe target `down`, `request timeout` | ASA: source IPs in `snmp-server host`. Routers: ACL. Try `snmpget -v3 -u prom_snmp -l authPriv ...` from inside cluster (`kubectl run --rm -it ... alpine ...`) |
| Probe target `down`, HTTP 500 from snmp_exporter | Look at snmp_exporter logs — `error walking target ...`, often missing OIDs (module mismatch) |
| `cisco-asa-critical` fires as `DatasourceNoData` | `noDataState: NoData` on the rule — should be `OK` (§15.11) |
| Grafana dashboard "No data" but Loki has Cisco entries | `Cisco Device` variable selected on a stale `routerboard` (e.g. old `ASA-1` after device-id hostname was unset). Set to `All`. |
| Multiple `routerboard` values for one device (`ASA-1`, pod IP, `EET`) | Apply `logging device-id hostname` and remove `show-timezone` — see §15.2/3 |
| Memory utilisation panel empty for ASA but fine for routers | ASA needs CISCO-ENHANCED-MEMPOOL-MIB (`.221`), not legacy `.48`. cisco_asa module must walk `1.3.6.1.4.1.9.9.221.1.1.1.1` — see §6 / §15.9 |
| ASA ACL denies stat = 0 even though there are denies | Default ASA emits `%ASA-3-710003`, not `%ASA-4-106023`. Use `\|~ "%ASA-[0-7]-(106\|710003)"` — §15.10 |
| Alert text shows `summary = ... on [no value]` | `noDataState=NoData` synthetic alert; set to `OK` — §15.11 |
| Alert description always says "1 ... in 5m" regardless of actual count | Annotation uses `$values.B` (threshold result 0/1) instead of `$values.A.Value` — §15.12 |
| Pod logs show `field lookups not found in type config.plain` | snmp.yml uses old (module-level) `lookups` syntax — must be per-metric (§6) |
| Helm upgrade fails with `undefined variable "$labels"` | Missing `tpl` escape on Grafana templates — see §10 |

## 18. Out of scope (deferred)

- **NetFlow / IPFIX** ("Top Talkers"). Prometheus is a poor fit for high-cardinality flow data; would need a separate stack (goflow2 + ClickHouse or pmacct).
- **Alertmanager → Slack receiver.** Botkube's Prometheus polling already covers this. If it's later wanted: add a Slack receiver via Helm values, with the webhook URL synced from Vault (`apps/grafana-alerting`).
- **Cisco MIB downloads.** snmp_exporter ships with built-in MIB knowledge for standard MIBs; we use raw OIDs for Cisco-specific entries — no MIB ConfigMap needed.
- **NetworkPolicies on the monitoring namespace.** None used in cluster today; would be a separate, broader exercise.
- **Loki Ruler-as-code for log alerts.** Currently Grafana-managed alerting (provisioned via Helm values). Migrating to Loki Ruler would push alert evaluation into Loki and route via Alertmanager — bigger change, defer until needed.
