# Sturgeon Technologies

Managed infrastructure, ISP, and software for small and mid‑size businesses —
run the way a platform team runs production: **everything as code, Git is the
source of truth, machines do the applying.**

This repo is the org's internal index: what we've built, and how the pieces fit
together.

---

## What we do

| Area | What it is |
| ---- | ---------- |
| **ISP / connectivity** | Business internet over a Fatbeam fiber uplink with a UniFi‑managed edge. Configuration, drift detection, backups, and the vendor bill are automated out of the cluster — see [Managing the ISP in GitOps](#managing-the-isp-in-gitops). |
| **Managed infrastructure & consulting** | Standing up and operating GitOps platforms, Kubernetes, secrets management, backup/DR, and monitoring for SMB clients. |
| **Production cluster** | A 6‑node bare‑metal Kubernetes cluster (kubeadm) that runs our own workloads and is the reference implementation for client work. |
| **Application development** | Serverless full‑stack apps on AWS (SAM + React), an on‑prem IoT platform, and multi‑agent tooling. |

---

## Services we've designed and implemented

### Platform & operations

- **GitOps Kubernetes platform** — a 6‑node bare‑metal cluster (kubeadm, HA
  control plane via HAProxy + keepalived) reconciled entirely by Flux from Git.
  A commit is the only way to change what runs; hand‑applied changes drift and
  are reverted.
- **Cluster bootstrap pipeline** — a one‑shot install (kubeadm, CNI, load
  balancer, Flux) that takes bare Ubuntu hosts to a GitOps‑ready cluster, then
  hands off to the manifests repo for day‑2.
- **Ansible automation control plane** — an AWX instance whose every object
  (organizations, projects, inventories, credentials, job templates, workflows,
  schedules) is declared as YAML and reconciled against the AWX API. The UI is a
  read‑only view.
- **Operational runbooks as code** — Ansible playbooks for cluster health
  sweeps, workload and external‑service remediation, and failed‑job cleanup,
  moving operations toward `monitor → trigger → playbook → log → alert`.
- **Secrets management** — External Secrets Operator syncing from AWS Secrets
  Manager, with workloads authenticating to AWS via IRSA. No secret is ever
  committed.
- **Backup & disaster recovery** — scheduled jobs pushing configs, photo
  libraries, game saves, and network state to versioned S3, with Discord
  reporting on every run.
- **Observability** — Prometheus + Grafana dashboards with Discord / Slack
  alerting across infrastructure, storage, network, and application workloads.
- **Ingress & TLS** — cert‑manager issuing Let's Encrypt certificates for every
  public hostname, currently mid‑migration from ingress‑nginx to Traefik.
- **MCP servers** — Model Context Protocol endpoints (Grafana, UniFi Protect,
  Kubernetes) exposing operational data to assistants.

### Networking & ISP

- **Edge network** — a UniFi gateway fronting the LAN, VLAN segmentation, and
  Wi‑Fi on a Fatbeam fiber uplink.
- **Declarative network record** — the controller's full configuration
  (networks, VLANs, SSIDs, firewall, routing, site settings) exported, secret‑
  redacted, and version‑controlled.
- **Config drift detection** — a daily job that snapshots the controller,
  compares it to the last known state, and posts a per‑section diff to Discord
  only when something changed.
- **Controller backups** — a weekly full backup of the gateway to S3, scheduled
  around the export so the two never collide on the controller's auth lockout.
- **Network monitoring** — a poller feeding client counts, throughput, and AP
  health into Prometheus / Grafana; UniFi Protect surfaced through MCP.
- **Vendor billing automation** — a monthly job that watches the billing mailbox
  for the ISP invoice, fires once, and raises a "bill missing" alert if it never
  arrives.

### Applications

- **File‑sharing platform (`schuit.io`)** — an invite‑only web app for browsing
  and downloading shared files out of S3, with per‑mount access control and
  admin‑managed invites. **Live in production.** Serverless AWS SAM backend +
  React SPA, multi‑tenant: many isolated "spaces" run from one codebase, each
  stood up from an Ansible vars file. Stranger‑deployable in about an hour via
  guided setup scripts with region‑mismatch guards and a type‑the‑phrase
  teardown confirmation. Runtime `/config` decides which sign‑in options the UI
  shows. Includes in‑browser image/video preview and a React Native mobile
  client against the same backend.
- **Marketing site + storefront (`sturgeon.tech`)** — a serverless CMS and store:
  public homepage, admin‑only article CRUD with a WYSIWYG editor, hero images
  and attachments, homepage / published flags. The entire stack runs offline
  with a dev role switcher, so every screen is reachable without deploying.
  Payments (Stripe, PayPal) stubbed.
- **AWS IoT management console** — a serverless app for AWS IoT Core: manage
  Things, per‑device email ACLs (authoritative Set attribute plus a reverse
  index, dual‑written in one transaction), and users. Split into three stacks —
  a us‑east‑1 certificate stack, a Retain‑protected stateful stack (Cognito,
  DynamoDB, S3, SES), and a disposable app stack that finds the stateful
  resources through SSM — so a rollback can never destroy accounts, ACLs, or
  firmware. Firmware is compiled **in the cloud** (`arduino-cli` in a
  container‑image Lambda) and keyed to an immutable device id; device identity
  is a minted X.509 certificate verified end‑to‑end over mutual TLS; updates are
  driven by `desired` / `reported` firmware epochs on the Device Shadow. Cognito
  Managed Login v2 (OAuth2 + PKCE) with native/Google account linking.

### IoT & agents

- **On‑prem IoT platform** — a self‑hostable device backend (`iot-onprem.sturgeon.tech`):
  EMQX for MQTT, MongoDB for state, `step-ca` for device certificates, and an API
  service, all running in the cluster.
- **Agent runtime** — a portable agent that "runs anywhere, uses anything,"
  deployed on the cluster as department‑shaped workers (dev, management,
  marketing/sales, research) with persistent per‑agent state.

---

## Platform architecture

```
GitHub (main)  ──git poll 1m──▶  Flux  ──▶  infrastructure/  ──▶  application namespaces
                                   │         cert-manager               media · monitoring · home
                                   │         Traefik ingress             billing · network · awx …
                                   │         External Secrets            backup CronJobs
                                   ▼
                             Alerts → Discord / Slack
```

- **Reconciliation:** Flux polls the manifests repo every minute. Durable change
  goes through Git and CI (yamllint + prettier); hand‑applied change drifts and
  is reverted.
- **Secrets:** never committed. External Secrets Operator pulls them from **AWS
  Secrets Manager** at runtime; workloads authenticate to AWS via IRSA.
- **Storage:** RWX **NFS** backed by an Unraid box; bulk backups go to
  versioned **S3**.
- **Automation orchestration:** **AWX** runs the Ansible playbooks. Direction of
  travel for operations is `monitor → trigger → Ansible playbook → log the
  effort → alert on failure`.
- **Observability:** Prometheus + Grafana, with Discord / Slack notifications.

---

## Managing the ISP in GitOps

The edge network is UniFi‑managed (a gateway fronting the LAN, VLANs, and Wi‑Fi)
on a Fatbeam fiber uplink. The controller is still where changes are *made*, but
everything around it — the record of intended state, drift detection, backups,
monitoring, and the vendor bill — is **code in the GitOps repo**, running as
scheduled jobs in the cluster.

### 1. Declarative record

A committed JSON dump of the controller's state: LAN + VLAN definitions, Wi‑Fi
SSIDs, and site settings, with a human‑readable VLAN map alongside it. Before
anything is committed, a fixed set of sensitive fields (`x_passphrase`,
`pre_shared_key`, `shared_secret`, `password`, `api_key`, `wpa_psk`,
`snmp_community`, …) is scrubbed to `***REDACTED***`. The full unredacted state
lives only on the gateway and in the encrypted S3 backup.

### 2. Drift detection — daily job

1. Authenticates to the controller with a scoped local admin account (creds via
   External Secrets → AWS Secrets Manager).
2. Pulls every relevant endpoint — networks, WLANs, firewall rules and groups,
   routing, port config, user groups, site settings — into one snapshot.
3. Redacts secrets.
4. Diffs it against the last snapshot in S3.
5. On change: uploads a timestamped snapshot, updates `latest.json`, and posts a
   **per‑section diff summary to Discord** (`networks: 10 → 11 entries`, …). No
   change means no upload and no noise.

> Status: **suspended since 2026‑06‑02** pending an S3 cost review. Re‑enable to
> resume.

### 3. Backups — weekly job

A full controller backup every Sunday 04:00 to versioned S3, scheduled before the
config export so the two never collide on the controller's auth lockout.

### 4. Monitoring

A poller scrapes the controller's API read‑only into Prometheus; Grafana
dashboards render client counts, throughput, and AP health. UniFi Protect is
exposed through an MCP server for camera / event queries.

### 5. Vendor billing — monthly job

Runs at noon PT on the 1st–5th of each month. Authenticates as the billing
mailbox via a Google Cloud service account with domain‑wide delegation (no OAuth
refresh‑token dance), finds the month's ISP invoice, labels it so it only fires
once, and pings Discord — or raises a **"bill missing"** alert if day 5 arrives
with no invoice. A later revision will create the accounting‑system bill and
schedule the payment.

### Change workflow

```
edit manifest ──▶ PR + CI (yamllint / prettier)
        │
        └─▶ merge to main ──▶ Flux reconciles (≤1m) ──▶ CronJobs / secrets updated
```

Controller‑side changes (a new VLAN, a firewall rule) are made in the UniFi UI
today, then captured on the next export. The roadmap is to close that loop:
automate the export as a first‑class job again, add drift *correction* (not just
detection), and drive VLAN / firewall provisioning from committed config.

---

## Conventions

- **Git is the source of truth.** UIs (AWX, UniFi, the cluster) are read‑models
  of what's in a repo.
- **Document how services fit together** where they're defined — not just the
  bare manifest.
- **Secrets never land in Git.** If a workload needs one, it goes in AWS Secrets
  Manager and is pulled by External Secrets.
- **Restarts are pod deletes, not rollouts** — Flux owns the spec.

---

*Internal overview — the org's public GitHub profile is rendered separately from
the `.github` repo.*
