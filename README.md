# SRE-Project: Kubernetes GitOps platform on AWS EKS

A multi-service stack on Kubernetes, delivered with Argo CD from this repository: the Ghost blog
with MySQL, observed with Prometheus, Grafana, Fluent Bit and Splunk Enterprise (through the
Splunk Operator).

I built the EKS cluster and the monitoring stack. The cluster is no longer running, so this
repository is the record of how it was put together.

## What is in this repository

Most folders are Helm charts copied in from upstream (Bitnami charts, the Splunk Operator
charts) with a customised `values.yaml`. The delivery wiring and the storage classes are the
files in the root.

| Path | What it is |
|---|---|
| `argo-cd/` | Argo CD Helm chart and values |
| `ghost/` | Ghost with a MySQL subchart (Helm) |
| `prometheus/`, `grafana/`, `fluent-bit/` | Metrics, dashboards and log shipping (Helm) |
| `splunk-operator/`, `splunk-enterprise/` | The Splunk Operator with its CRDs, and a Splunk Enterprise deployment |
| `deploy-apps-project.yaml` | Argo CD `AppProject` named `deploy` |
| `ghost-app-deploy.yaml` | Six Argo CD `Application` objects: ghost, prometheus, grafana, fluent-bit, splunk-operator, splunk-enterprise |
| `ghost-efs-sc.yaml` | StorageClass `ghost` on EFS (`efs.csi.aws.com`, access points, directory mode 700) |
| `gp2-csi.yaml` | StorageClass `gp2-csi` on EBS (gp2, volume expansion allowed) |

## How delivery works

Each `Application` points at a folder of this repository (`targetRevision: main`) and renders
that folder's chart with its `values.yaml` into the `app` namespace. Sync is automated with
`prune` and `selfHeal`, so Git is the source of truth and manual changes in the cluster are
reverted. Sync waves order the start: Fluent Bit is wave 2, the Splunk Operator 3 (applied with
`ServerSideApply` and `Replace`, because of its large CRDs), Splunk Enterprise 4; the rest
sync first. The `AppProject` allows cluster-scoped `ClusterRole`, `ClusterRoleBinding` and
`CustomResourceDefinition` objects, and every namespace on the in-cluster server.

## Secrets

No passwords are stored in Git. The charts read Kubernetes Secrets that are created by hand in
the `app` namespace before the first sync:

```bash
kubectl -n app create secret generic ghost-credentials \
  --from-literal=ghost-password="$(openssl rand -base64 18)"

kubectl -n app create secret generic ghost-mysql-credentials \
  --from-literal=mysql-root-password="$(openssl rand -base64 18)" \
  --from-literal=mysql-password="$(openssl rand -base64 18)"

kubectl -n app create secret generic grafana-admin \
  --from-literal=password="$(openssl rand -base64 18)"
```

Secrets are created outside Git on purpose. With a chart's own random-password generation,
Argo CD renders the chart without a cluster lookup, so every sync would produce a new password
and `selfHeal` would keep applying it. The Argo CD admin password is left empty in
`argo-cd/values.yaml`, so that chart generates one at install.

## Problems I fixed

From the commit history:

- **Bitnami image paths.** Older Bitnami images moved to the `bitnamilegacy` repository, so
  syncs failed to pull. The image repositories for Ghost, MySQL, the MySQL exporter, the shell
  helper image, Fluent Bit and Prometheus were repointed.
- **Wrong source in the Applications.** They pointed at another repository and at `master`;
  they now point at this repository's `main`.
- **MySQL subchart.** Pinned to one version and the lock file regenerated, with `charts/`
  ignored so downloaded subcharts stay out of Git.
- **Typos that silently did nothing.** The load-balancer annotation was missing `.io`, so the
  internet-facing scheme was not applied; the EFS `fileSystemId` was wrong; the Splunk volumes
  asked for the StorageClass `gpu2-csi` instead of `gp2-csi`; a YAML indentation error in the
  Ghost values.
- **Splunk.** The general-terms licence acceptance was set in the values, and missing operator
  CRDs were added.

## What this is not

- Not production: a learning and portfolio project, with no real users or traffic.
- Not running: there is no live cluster behind it.
- Charts are vendored copies, so they are not kept up to date with upstream.
- Earlier commits contained lab passwords in the values files. They are removed from the
  current files but remain in the Git history, so they must never be reused.
