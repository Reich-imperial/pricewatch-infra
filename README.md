# pricewatch-infra

Kubernetes manifests for [PriceWatch](https://github.com/Reich-imperial/pricewatch-app), a queue-driven price pipeline that scales its worker with KEDA.

This repo is the deployment side of the project. Application code, Dockerfiles, Docker Compose and the load test live in `pricewatch-app`.

---

## Status

| Piece | Status | Evidence |
|---|---|---|
| kind cluster `pricewatch` (Kubernetes v1.35.8) | Running | pods healthy |
| KEDA 2.21.0 | Installed, spike tested | stand-in worker scaled 1 to 5 pods on 23 messages |
| ElasticMQ in the cluster | Running | used in the spike; queues to be re-checked after a restart |
| Postgres (StatefulSet, 1Gi PVC) | Done, tested | row survived deleting the pod |
| Real worker and API | **Next** | not built |
| KEDA `ScaledObject` for the real worker | **Next** | not built |
| Kustomize, CI, Argo CD | Planned | not started |
| Terraform and AWS | Planned | not started |

---

## Layout

```text
.
├── k8s/
│   ├── postgres.yaml        Service + StatefulSet with a PVC
│   └── elasticmq.yaml       queue deployment (copy in from the spike if missing)
├── .gitignore               ignores *.tar image archives
└── README.md
```

Not stored in git: the Postgres Secret (created by command) and image archives (`*.tar`).

---

## Prerequisites

- Docker, kind and kubectl
- A kind cluster named `pricewatch`

A short alias keeps commands pointed at the right cluster and namespace (this machine has several kind clusters):

```bash
alias kp='kubectl --context kind-pricewatch -n pricewatch'
```

---

## Deploy Postgres

**1. Create the credentials and schema.** The Secret is created by command so no password is committed. Use the same development values as the Compose file in `pricewatch-app`.

```bash
kp create secret generic postgres-credentials \
  --from-literal=POSTGRES_USER=pricewatch \
  --from-literal=POSTGRES_PASSWORD=<dev-password> \
  --from-literal=POSTGRES_DB=pricewatch

kp create configmap postgres-init \
  --from-file=init.sql=<path-to>/pricewatch-app/dev/init.sql
```

**2. Load the image into kind.** `kind load docker-image` failed with Docker 29, so use an archive:

```bash
docker save --platform linux/amd64 postgres:17-alpine -o /tmp/pg.tar
kind load image-archive /tmp/pg.tar --name pricewatch
```

**3. Apply and verify.**

```bash
kp apply -f k8s/postgres.yaml
kp rollout status statefulset/postgres --timeout=120s
kp exec postgres-0 -- psql -U pricewatch -d pricewatch -c '\dt'
```

Expected: a `price_events` table.

---

## Persistence drill

This is the check that proves the data lives on the volume, not in the pod.

```bash
kp exec postgres-0 -- psql -U pricewatch -d pricewatch -c \
  "insert into price_events (event_id, symbol, price_ngn, observed_at, source, received_at, drop_pct, is_drop) \
   values (gen_random_uuid(), 'RICE_50KG', 91234, now(), 'drill', now(), null, false)"
kp delete pod postgres-0
kp wait --for=condition=ready pod/postgres-0 --timeout=120s
kp exec postgres-0 -- psql -U pricewatch -d pricewatch -c \
  "select count(*) from price_events where source='drill'"
kp get pods,pvc
```

Result: the count stayed at 1. The new pod was 40 seconds old with 0 restarts, while the PVC was 29 minutes old and `Bound`.

---

## Design notes

- **Service names, not localhost.** Inside the cluster, pods reach each other as `postgres` and `elasticmq`. Each pod has its own network namespace.
- **StatefulSet plus PVC for Postgres.** `emptyDir` would lose data with the pod. The cost is more YAML and a failure mode worth drilling: a stuck volume can block a restart.
- **`PGDATA` in a subfolder** of the mount, because Postgres refuses to start when the volume root contains unexpected files.
- **Volumes are local.** The `local-path` volumes live on the kind node's disk, so deleting the cluster deletes the data.
- **Idle cost.** A kind cluster uses CPU even when idle (the control plane sat near 55% of one core while nothing ran). Stop it when you are not using it.

---

## Roadmap

1. Load the real worker and API images, add Deployments and Services, and check that a message sent to the API reaches Postgres.
2. Add a KEDA `ScaledObject` for the real worker and compare scaling with the measured speed of about 4.3 messages per second per worker at a 200 ms delay.
3. Add Prometheus, Grafana and k6 in the cluster (watch the RAM cost).
4. Kustomize base and overlays, CI that builds and tests images, Argo CD syncing from this repo.
5. Terraform and AWS (SQS with a DLQ, IAM), replacing ElasticMQ.
6. Incident-style write-ups for each fault drill.
