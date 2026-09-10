# Sturgeon Technologies

Managed infrastructure, ISP, and software for small and mid‑size businesses —
run the way a platform team runs production: **everything as code, Git is the
source of truth, machines do the applying.**

This repo is the org's internal index: what we work on, where it lives, and how
the pieces fit together.

---

## What we work on

| Area | What it is |
| ---- | ---------- |
| **ISP / connectivity** | Business internet delivered over a Fatbeam fiber uplink, with a UniFi‑managed edge (gateway, VLANs, Wi‑Fi). Configuration, drift detection, backups, and billing are automated out of the cluster — see [Managing the ISP in GitOps](#managing-the-isp-in-gitops). |
| **Managed infrastructure & consulting** | Standing up and operating GitOps platforms, Kubernetes, secrets management, backup/DR, and monitoring for SMB clients. |
| **Homelab / production cluster** | A 6‑node bare‑metal Kubernetes cluster (kubeadm) that runs our own workloads and is the reference implementation for client work. |
| **Application development** | Serverless full‑stack apps on AWS (SAM + React), plus multi‑agent tooling. |

---

## Repositories

| Repo | Purpose |
| ---- | ------- |
| [`kubernetes_flux`](https://github.com/SturgeonTechnologies/kubernetes_flux) | GitOps manifests for the production cluster. Push a commit, Flux applies it — nobody runs `kubectl` by hand. Includes the ISP/network, billing, backup, and monitoring workloads. |
| [`kubernetes_deployment`](https://github.com/SturgeonTechnologies/kubernetes_deployment) | One‑time cluster bootstrap: kubeadm, Calico, HAProxy + keepalived, Flux install. Hands off to `kubernetes_flux` for day‑2. |
| [`OPS_ansible`](https://github.com/SturgeonTechnologies/OPS_ansible) | Operational Ansible — cluster health sweeps, workload/service remediation, failed‑job cleanup, app deploys, and the vault‑brief job. Run from AWX. |
| [`AWX_CaC`](https://github.com/SturgeonTechnologies/AWX_CaC) | AWX configuration as code for `awx.sturgeon.tech`. Every org, project, inventory, credential, job template, workflow, and schedule is declared in YAML and reconciled against the AWX API. |
| [`schuit-sharing`](https://github.com/SturgeonTechnologies/schuit-sharing) / [`BackFriend_FullStack`](https://github.com/SturgeonTechnologies/BackFriend_FullStack) | Invite‑only file‑sharing app, live at **schuit.io**. AWS SAM + React/Vite SPA; multi‑"space" deployable with public deploy tooling. Deployed per space from the `OPS_ansible` `deploy_backfriend` playbook. |
| [`sturgeon_dot_tech`](https://github.com/SturgeonTechnologies/sturgeon_dot_tech) | Serverless CMS + store for **sturgeon.tech**, on AWS SAM. Runs the full stack offline with no AWS deploy; Stripe/PayPal stubbed. |
| [`AWS_IOT_MGMT`](https://github.com/SturgeonTechnologies/AWS_IOT_MGMT) | Serverless management app for **AWS IoT Core** — device ACLs, cloud sketch compile (`arduino-cli` in Lambda), certificate provisioning, and Device‑Shadow OTA. Python SAM backend + React SPA. |
| [`openclaude`](https://github.com/SturgeonTechnologies/openclaude) | Agent runtime — "runs anywhere, uses anything." Basis for the multi‑agent work on the cluster. |

---

## The platform

```
GitHub (main)  ──git poll 1m──▶  Flux  ──▶  infrastructure/  ──▶  application namespaces
                                   │         cert-manager               media · monitoring · home
                                   │         Traefik ingress             billing · network · awx …
                                   │         External Secrets            backup/ CronJobs
                                   ▼
                             Alerts → Discord / Slack
```

- **Reconciliation:** Flux polls `kubernetes_flux@main` every minute. Durable
  change goes through Git and CI (yamllint + prettier); hand‑applied change
  drifts and is reverted.
- **Secrets:** never committed. External Secrets Operator pulls them from **AWS
  Secrets Manager** at runtime; workloads authenticate to AWS via IRSA.
- **Storage:** RWX **NFS** backed by an Unraid box; bulk backups go to
  **`s3://schuit-backups/`**.
- **Automation orchestration:** **AWX** runs the Ansible playbooks. Direction of
  travel for operations is `monitor → trigger → Ansible playbook → log the
  effort → alert on failure`.
- **Observability:** Prometheus + Grafana, with Discord/Slack notifications.

---

## Managing the ISP in GitOps

The edge network is UniFi‑managed (a UDM Pro gateway fronting the LAN, VLANs, and
Wi‑Fi) on a Fatbeam fiber uplink. The UniFi controller is still the place changes
are *made*, but everything around it — the record of intended state, drift
detection, backups, monitoring, and the vendor bill — is **code in
`kubernetes_flux`** and runs as scheduled jobs in the cluster.

### 1. Declarative record — `clusters/production/network/`

- `unifi-config/` holds a JSON dump of the controller's state: LAN + VLAN
  definitions (`networkconf`), Wi‑Fi SSIDs (`wlanconf`), and site settings
  (`settings`). The human‑readable VLAN map is documented alongside it.
- Before anything is committed, a fixed set of sensitive fields
  (`x_passphrase`, `pre_shared_key`, `shared_secret`, `password`, `api_key`,
  `wpa_psk`, `snmp_community`, …) is scrubbed to `***REDACTED***`. The full
  unredacted state lives only on the UDM Pro and in the encrypted S3 backup.
- A `network` namespace is carved out here for any controller‑facing tooling
  (drift detection, provisioning, exporters).

### 2. Drift detection — `network/unifi-config-export` CronJob

Daily job that:

1. authenticates to the controller with a scoped local admin account (creds via
   External Secrets → AWS Secrets Manager),
2. pulls every relevant endpoint — `networkconf`, `wlanconf`, `firewallrule`,
   `firewallgroup`, `routing`, `portconf`, `user-group`, `settings` — into one
   snapshot,
3. redacts secrets,
4. diffs it against the last snapshot in `s3://schuit-backups/unifi-config-snapshots/`,
5. on change: uploads a timestamped snapshot + updates `latest.json`, and posts
   a **per‑section diff summary to Discord** (`networkconf: 10 → 11 entries`, …).
   No change means no upload and no noise.

> Status: **suspended since 2026‑06‑02** pending an S3 cost review. Set
> `suspend: false` to resume.

### 3. Backups — `backup/unifi-backup` CronJob

Weekly full controller backup (Sunday 04:00) to
`s3://schuit-backups/unifi-backups/`, scheduled before the config export so the
two never collide on the controller's auth lockout.

### 4. Monitoring — `monitoring/`

`unifi-poller` scrapes the controller's API read‑only into Prometheus; Grafana
dashboards (`unifi-dashboards`) render client counts, throughput, and AP health.
UniFi Protect is exposed through an MCP server for camera/event queries.

### 5. Vendor billing — `billing/fatbeam-bill-watcher` CronJob

Runs at noon PT on the 1st–5th of each month. Authenticates as
`billing@sturgeon.tech` via a Google Cloud service account with domain‑wide
delegation (no OAuth refresh‑token dance), finds the month's Fatbeam AR invoice
in the mailbox, labels it so it only fires once, and pings Discord — or pings a
**"bill missing"** alert if day 5 arrives with no invoice. v2 will create the
QuickBooks Bill and schedule the payment.

### Change workflow

```
edit manifest in kubernetes_flux ──▶ PR + CI (yamllint / prettier)
        │
        └─▶ merge to main ──▶ Flux reconciles (≤1m) ──▶ CronJobs / secrets updated
```

Controller‑side changes (a new VLAN, a firewall rule) are made in the UniFi UI
today, then captured on the next export. The roadmap is to close that loop:
automate the export as a first‑class job again, add drift *correction* (not just
detection), and drive VLAN/firewall provisioning from committed config.

---

## Serverless applications

Three apps share one architecture: **AWS SAM** — a React (Vite) SPA on a private
S3 bucket behind CloudFront (Origin Access Control), an API Gateway HTTP API in
front of Lambda, a single‑table DynamoDB backend, Cognito for auth (email +
optional Google/Facebook, role groups, first confirmed user → admin), and SES
for verification / reset / transactional mail. Custom domain, cert, and DNS are
an opt‑in ACM + Route 53 layer. Every deployment‑specific value is a blank‑default
template parameter, so nothing identifying lives in the repos; real values sit in
a git‑ignored `samconfig.local.toml`.

### schuit‑sharing — file sharing (`schuit-sharing` / `BackFriend_FullStack`)

Invite‑only web app for browsing and downloading shared files out of S3
(per‑mount access control, admin‑managed invites). **Live in production at
[schuit.io](https://schuit.io).** Migrated off the Serverless Framework to SAM
(cutover Aug 2026). Deployable by a stranger in ~1 hour: `quickstart.mjs` /
`teardown.mjs` handle frontend hosting + SES setup, region‑mismatch guards, and a
type‑the‑phrase confirmation before any prod‑shaped teardown. Multiple isolated
"spaces" (`schuit-sharing`, `beeks-sharing`, `samtest`) run from the same code,
each stood up from an `OPS_ansible` per‑space vars file. Runtime `GET /config`
drives which login options the SPA shows, so a stack's real Cognito state — not
build‑time assumptions — decides the UI. In‑browser image/video preview with
1‑hour presigned URLs. An Expo / React Native client targets the same backend
with federated spaces.

### sturgeon.tech — marketing site + store (`sturgeon_dot_tech`)

Serverless CMS and storefront for the company site: public homepage, admin‑only
article CRUD with a WYSIWYG editor, hero images and attachments, `homepage` /
`published` flags. Whole stack runs **offline** — `make local-up` gives you
DynamoDB Local, `make api` / `make web` the rest, with a dev role switcher
(guest → customer → store/site/super‑admin) so every screen is reachable without
Cognito. Baseline scaffold, not yet deployed; Stripe and PayPal are stubbed in
the UI.

### AWS IoT management (`AWS_IOT_MGMT`)

Serverless console for **AWS IoT Core**: manage Things, per‑device email ACLs
(authoritative Set attribute + reverse index, dual‑written in one transaction),
and users. Deliberately split into **three stacks** — `cert` (us‑east‑1 ACM),
`stateful` (Cognito / DynamoDB / S3 / SES, all `DeletionPolicy: Retain`), and a
disposable `app` stack that finds the stateful resources through SSM Parameter
Store — so tearing down or rolling back the app can never destroy accounts,
device ACLs, or firmware. The device never compiles anything: an Arduino sketch
lives on the device's DynamoDB row and is built by a container‑image Lambda
running `arduino-cli` + the ESP32 toolchain; the binary lands in a private S3
bucket keyed by an immutable `thingId`. `POST /provision` mints the device's
X.509 certificate (verified end‑to‑end against AWS IoT over mutual TLS); OTA is
driven by `desired` / `reported` `firmwareEpoch` on the Device Shadow. Auth is
Cognito Managed Login v2 (OAuth2 + PKCE) with native/Google account linking via
`PreSignUp` / `PostConfirmation` / `PostAuthentication` triggers; unknown callers
get **404, not 403**, so device names can't be probed. GitHub Actions deploys the
app stack + frontend via OIDC (no stored AWS keys); the Retain‑protected stateful
stack is deployed by hand.

---

## Conventions

- **Git is the source of truth.** UIs (AWX, UniFi, the cluster) are read‑models
  of what's in a repo.
- **Document how services fit together** in the repo that owns them — not just
  the bare manifest.
- **Secrets never land in Git.** If a workload needs one, it goes in AWS Secrets
  Manager and is pulled by External Secrets.
- **Restarts are pod deletes, not rollouts** — Flux owns the spec.

---

*Internal overview — the org's public GitHub profile is rendered separately from
`SturgeonTechnologies/.github`.*
