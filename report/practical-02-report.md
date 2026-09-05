# DSO202 — Practical 02 Report
## Kubernetes Storage and Stateful Workloads

**Guide reference:** [DSO202 Practical 02 (HackMD)](https://hackmd.io/@sarojsanyasi/dso202-practical-02)

---

## 1. Objective

Practical 1 dealt with stateless workloads. This practical addresses the
opposite problem: how Kubernetes handles data that must **survive** a Pod's
lifecycle. It builds up the storage stack layer by layer — PersistentVolumes,
PersistentVolumeClaims, StorageClasses, static vs dynamic provisioning,
reclaim policies — and then shows why none of that is sufficient on its own
without a controller that understands *identity*, which is the role of the
`StatefulSet`. The practical closes with a realistic workload, PostgreSQL,
to prove the concepts hold under an actual database engine rather than a
toy writer Pod.

![Repository structure](../evidence/1.png)

---

## 2. Environment Setup

### 2.1 Prerequisite Verification

Docker, `kind`, and `kubectl` versions were verified before starting:

```bash
docker --version
kind --version
kubectl version --client
```

![Prerequisite verification](../evidence/2.png)

### 2.2 Host Storage Directory

Static provisioning in Stage 2 requires a host path that exists **before**
the cluster is created, since `kind` mounts host directories into a node at
node-creation time, not afterwards:

```bash
mkdir -p /tmp/dso202-p2-storage
ls -ld /tmp/dso202-p2-storage
```

![Host storage directory created](../evidence/3.png)

```bash
df -h /var/lib/docker 2>/dev/null || df -h /
```

![Disk space check](../evidence/4.png)

---

## 3. Stage 1 — Cluster, Namespace, and the Storage Landscape

### 3.1 Cluster Creation

The cluster was created from `cluster/kind-cluster.yaml` (Listing 1):

```bash
kind create cluster --config cluster/kind-cluster.yaml
```

![Cluster created](../evidence/5.png)

```bash
kubectl get nodes -o wide
docker ps --format 'table {{.Names}}\t{{.Status}}'
```

![Node-to-container mapping](../evidence/6.png)

Recording the mapping between Kubernetes **Node objects** and their
underlying **Docker container names** was necessary, since later stages
inspect node-local filesystems directly via `docker exec` rather than through
the Kubernetes API.

The host directory mount was verified on the worker node:

```bash
docker exec dso202-p2-worker ls -ld /mnt/dso202-static
```

![Host directory mounted into node](../evidence/7.png)

### 3.2 Namespace, Quota, and StorageClass

The namespace, resource quota, and a `Retain`-policy StorageClass were
applied and the namespace set as the default context:

```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl config set-context --current --namespace=dso202-practical-02
kubectl apply -f manifests/01-quota-and-limits.yaml
kubectl apply -f manifests/02-storageclass-retain.yaml
```

![Namespace, quota, and StorageClass applied](../evidence/8.png)

```bash
kubectl describe resourcequota dso202-p2-quota
```

![Resource quota including storage constraints](../evidence/9.png)

Unlike Practical 1's quota (CPU, memory, object counts), this quota adds
**three storage-specific constraints**: a cap on the number of PVCs, a cap on
total requested storage, and a per-StorageClass cap — all of which matter
directly for the provisioning experiments that follow.

### 3.3 Locating the Provisioner

```bash
kubectl get storageclass
```

![StorageClasses and provisioner](../evidence/10.png)

```bash
kubectl -n local-path-storage get pods
```

![Provisioner Pod in its own namespace](../evidence/11.png)

```bash
kubectl -n local-path-storage get configmap local-path-config -o jsonpath='{.data.config\.json}'
```

![Provisioner ConfigMap showing write path](../evidence/12.png)

The default dynamic provisioner in this cluster is **local-path-provisioner**,
running in its own `local-path-storage` namespace rather than in the
practical's namespace. Its `ConfigMap` is the authoritative source for where
it writes data on disk — not any external documentation. After applying the
custom `manual` StorageClass, both classes appear side-by-side in
`kubectl get storageclass`.

![Both StorageClasses visible after applying manual class](../evidence/13.png)

---

## 4. Stage 2 — Static Provisioning and the Meaning of `Retain`

### 4.1 Creating and Inspecting the PV

```bash
kubectl apply -f manifests/03-pv-static.yaml
kubectl get pv
```

![Static PersistentVolume created](../evidence/14.png)

```bash
kubectl get pv pv-web-static -o jsonpath='{.spec.hostPath.path}{"\n"}'
kubectl get pv pv-web-static -o jsonpath='{.spec.nodeAffinity}{"\n"}'
```

![hostPath and nodeAffinity fields](../evidence/15.png)

The two fields inspected here — `hostPath.path` and `nodeAffinity` — are what
constrain scheduling: a statically provisioned, host-path-backed volume ties
any Pod using it to a specific node.

### 4.2 Claiming and Writing

```bash
kubectl apply -f manifests/04-pvc-static.yaml
kubectl get pvc
```

![PVC bound immediately](../evidence/16.png)

```bash
kubectl get storageclass manual
```

![The manual StorageClass](../evidence/17.png)

```bash
kubectl apply -f manifests/05-pod-static-writer.yaml
kubectl wait --for=condition=Ready pod/static-writer --timeout=90s
kubectl get pod static-writer -o wide
```

![static-writer Pod ready](../evidence/18.png)

```bash
kubectl exec static-writer -- cat /data/ledger.txt
cat /tmp/dso202-p2-storage/pv-web-static/ledger.txt
```

![Ledger file read from container and host](../evidence/19.png)

Because both the PV and PVC specify `storageClassName: manual`, the claim
binds directly to the pre-created `pv-web-static` volume instead of
triggering a provisioner — this is the defining behaviour of static
provisioning.

### 4.3 Volume Outlives the Pod

```bash
kubectl delete pod static-writer
kubectl apply -f manifests/05-pod-static-writer.yaml
kubectl wait --for=condition=Ready pod/static-writer --timeout=90s
kubectl exec static-writer -- cat /data/ledger.txt
```

![Ledger file after Pod recreation](../evidence/20.png)

The ledger file accumulates an additional line on each Pod restart,
confirming the volume's data persists independently of the Pod's lifecycle.

### 4.4 Deleting the Claim: `Released`, Not `Available`

```bash
kubectl delete pod static-writer
kubectl delete pvc pvc-web-static
kubectl get pv
```

![PV phase becomes Released](../evidence/21.png)

With a `Retain` reclaim policy, deleting the PVC does **not** free the PV for
reuse — it transitions to the `Released` phase and must be manually
reclaimed. Confirming the underlying data was untouched, then removing the
PV object itself:

```bash
cat /tmp/dso202-p2-storage/pv-web-static/ledger.txt
kubectl delete pv pv-web-static
ls -l /tmp/dso202-p2-storage/pv-web-static/
```

![Data intact on host after PV object deletion](../evidence/22.png)

Even after the PV object is deleted, the host directory and its contents
remain — deleting the Kubernetes object never deletes host-path data under
`Retain`. All three objects were then recreated to confirm the ledger
persisted across the full teardown/rebuild cycle.

```bash
kubectl apply -f manifests/03-pv-static.yaml
kubectl apply -f manifests/04-pvc-static.yaml
kubectl apply -f manifests/05-pod-static-writer.yaml
kubectl wait --for=condition=Ready pod/static-writer --timeout=90s
kubectl exec static-writer -- cat /data/ledger.txt
```

![Ledger file after full recreation](../evidence/23.png)

---

## 5. Stage 3 — Dynamic Provisioning: Two Uncomfortable Truths

### 5.1 A Claim That Will Not Bind

```bash
kubectl apply -f manifests/06-pvc-dynamic.yaml
kubectl get pvc dynamic-data
```

![Dynamic PVC stuck Pending](../evidence/24.png)

```bash
kubectl describe pvc dynamic-data | tail -n 6
```

![Events explaining WaitForFirstConsumer](../evidence/25.png)

With `local-path-provisioner`, a PVC alone does **not** trigger volume
creation — this class uses `WaitForFirstConsumer` binding, so the claim stays
`Pending` until a Pod actually mounts it. The Events section of `describe`
is the authoritative explanation, not assumption.

### 5.2 Introducing a Consumer

```bash
kubectl apply -f manifests/07-pod-dynamic-writer.yaml
kubectl wait --for=condition=Ready pod/dynamic-writer --timeout=120s
kubectl get pvc dynamic-data
kubectl get pv
```

![PVC now bound and PV created](../evidence/26.png)

```bash
kubectl get pod dynamic-writer -o wide
docker exec dso202-p2-worker2 ls /var/local-path-provisioner
```

![Dynamic volume located on the node](../evidence/27.png)

Once a Pod references the claim, the provisioner creates the volume on
whichever node the Pod is scheduled to — the node lookup from Stage 1
translates the Pod's node name into the correct `docker exec` target.

### 5.3 Truth #1 — Requested Size Is Not Enforced

```bash
kubectl exec dynamic-writer -- df -h /data
```

![Mounted filesystem size vs requested 1Gi](../evidence/28.png)

Despite requesting `1Gi`, the mounted filesystem reports the full size of
the underlying host disk. `local-path-provisioner` does not enforce
capacity limits the way a true block-storage backend would.

### 5.4 Truth #2 — The Volume Cannot Grow

```bash
kubectl patch pvc dynamic-data --type merge -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
```

![Expansion request rejected](../evidence/29.png)

This patch is rejected by the API server itself (not the provisioner),
because the StorageClass sets `allowVolumeExpansion: false`. The rejection
being loud and immediate is the useful outcome here: **StorageClass choice
is effectively a decision about a workload's future**, made before any data
exists — a claim on an expandable class is grown simply by editing the same
`resources.requests.storage` field, after which both the volume and its
filesystem are resized.

### 5.5 Comparing Reclaim Behaviour, Then Deleting

```bash
kubectl exec dynamic-writer -- cat /data/ledger.txt
kubectl delete pod dynamic-writer
kubectl delete pvc dynamic-data
kubectl get pv
docker exec dso202-p2-worker2 ls /var/local-path-provisioner
```

![PV and host data removed under Delete policy](../evidence/30.png)

Unlike the `manual`/`Retain` class in Stage 2, the dynamic class's default
`Delete` reclaim policy removes both the PV object and the underlying host
data as soon as the claim is deleted.

---

## 6. Stage 4 — Why a Deployment Cannot Own State

```bash
kubectl apply -f manifests/08-deployment-shared-pvc.yaml
kubectl rollout status deployment/shared-writer --timeout=180s
kubectl get pods -l app=shared-writer -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```

![Three Deployment replicas mounting the same PVC](../evidence/31.png)

```bash
kubectl exec deploy/shared-writer -- cat /data/visitors.log
```

![Shared visitors log written by all replicas](../evidence/32.png)

Three replicas of a `Deployment` all mounting the **same** PVC can write to
a shared log file — but that only works because the underlying access mode
and the test cluster happen to permit it.

```bash
kubectl delete pod -l app=shared-writer --field-selector status.phase=Running --wait=false
sleep 15
kubectl get pods -l app=shared-writer -o custom-columns=NAME:.metadata.name
```

![New Pod with a randomly suffixed name](../evidence/33.png)

Deleting a replica reveals the **identity problem**: `Deployment` Pods get
new, randomly suffixed names on replacement, with no durable relationship
between a replica and "its" data. This is masked in the toy example, but is
exactly the failure mode that makes `Deployment` unsuitable for real
multi-replica stateful workloads such as databases, where each replica must
own distinct, addressable storage.

```bash
kubectl delete -f manifests/08-deployment-shared-pvc.yaml
kubectl get pvc
```

![Deployment removed, PVC still present](../evidence/34.png)

---

## 7. Stage 5 — StatefulSets and Stable Identity

### 7.1 The Headless Service First

```bash
kubectl apply -f manifests/09-service-webnote.yaml
kubectl get service webnote
```

![Headless Service created](../evidence/35.png)

The headless Service (`clusterIP: None`) is what enables per-Pod DNS
records. Applying it **before** the StatefulSet avoids a window where Pods
exist but are not yet individually addressable.

### 7.2 Ordered Pod Creation

```bash
kubectl apply -f manifests/10-statefulset-webnote.yaml
kubectl get pods -l app=webnote -w
```

![Pods created in order 0, 1, 2](../evidence/36.png)

```bash
kubectl get pvc -l app=webnote
```

![One PVC auto-created per Pod](../evidence/37.png)

```bash
kubectl get pods -l app=webnote -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP
```

![Pods freely scheduled across nodes](../evidence/38.png)

Pods are created in strict order (`webnote-0`, `webnote-1`, `webnote-2`),
each with its own automatically provisioned PVC. Because storage is now
per-Pod rather than shared, scheduling is no longer constrained the way the
static Stage 2 volume was.

### 7.3 Addressing Individual Pods

```bash
kubectl apply -f manifests/11-pod-client.yaml
kubectl wait --for=condition=Ready pod/client --timeout=90s
kubectl exec client -- nslookup webnote.dso202-practical-02.svc.cluster.local
```

![DNS resolution of the headless Service](../evidence/39.png)

```bash
kubectl exec client -- wget -qO- http://webnote-1.webnote.dso202-practical-02.svc.cluster.local
```

![Fetching content from webnote-1 by name](../evidence/40.png)

```bash
kubectl get endpointslice -l kubernetes.io/service-name=webnote
```

![EndpointSlice showing one address per Pod](../evidence/41.png)

Each Pod resolves under a stable, predictable DNS name
(`<pod-name>.<service>.<namespace>.svc.cluster.local`) — the reverse of a
normal Service, which load-balances across replicas rather than exposing
them individually.

### 7.4 Private, Per-Pod Volumes

```bash
kubectl exec webnote-0 -- sh -c 'echo "note added by hand in Stage 5" >> /usr/share/nginx/html/index.html'
kubectl exec client -- wget -qO- http://webnote-0.webnote.dso202-practical-02.svc.cluster.local
kubectl exec client -- wget -qO- http://webnote-1.webnote.dso202-practical-02.svc.cluster.local
```

![Note visible only via webnote-0](../evidence/42.png)

The edit made to `webnote-0` is visible only through `webnote-0` — proving
each replica's storage is isolated, unlike the shared-PVC `Deployment` in
Stage 4.

### 7.5 Identity and Storage Survive Deletion

```bash
kubectl delete pod webnote-1
kubectl wait --for=condition=Ready pod/webnote-1 --timeout=120s
kubectl get pod webnote-1 -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP
kubectl get pvc content-webnote-1
kubectl exec client -- wget -qO- http://webnote-1.webnote.dso202-practical-02.svc.cluster.local
```

![Same name, same PVC, note still present after Pod deletion](../evidence/43.png)

The replacement Pod keeps the exact same name and reattaches the same PVC —
this is the core guarantee a `StatefulSet` provides that a `Deployment`
cannot.

---

## 8. Stage 6 — Scaling, Retention, and Ordered Updates

### 8.1 Scale Up

```bash
kubectl scale statefulset webnote --replicas=4
kubectl wait --for=condition=Ready pod/webnote-3 --timeout=120s
kubectl get pvc -l app=webnote --no-headers | wc -l
```

![New PVC created alongside webnote-3](../evidence/44.png)

A new PVC is created automatically alongside the new ordinal Pod.

### 8.2 Scale Down — Claims Are Kept

```bash
kubectl scale statefulset webnote --replicas=2
kubectl get pods -l app=webnote -w
```

![Reverse-order termination](../evidence/45.png)

```bash
kubectl get pvc -l app=webnote
```

![PVCs for scaled-down replicas still present](../evidence/46.png)

Termination happens in strict reverse order (highest ordinal first). Scaling
down does **not** delete the PVCs belonging to removed replicas — this is a
deliberate safety default, so that scaling back up restores the same data:

```bash
kubectl scale statefulset webnote --replicas=3
kubectl wait --for=condition=Ready pod/webnote-2 --timeout=120s
kubectl exec client -- wget -qO- http://webnote-2.webnote.dso202-practical-02.svc.cluster.local
```

![webnote-2 returns with its original data](../evidence/47.png)

### 8.3 Partitioned Rolling Update

The manifest was edited to set `spec.updateStrategy.rollingUpdate.partition`
to `2` and the image tag changed from `nginx:1.30-alpine` to
`nginx:1.31-alpine`:

```bash
kubectl apply -f manifests/10-statefulset-webnote.yaml
sleep 30
kubectl get pods -l app=webnote -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```

![Only webnote-2 updated to the new image](../evidence/48.png)

With `partition: 2`, only Pods with an ordinal **greater than or equal to**
the partition value are updated — meaning only `webnote-2` picks up the new
image, while `webnote-0` and `webnote-1` remain on the old version. This is
how a canary-style, ordinal-controlled rollout is achieved with
StatefulSets.

Completing the rollout:

```bash
kubectl apply -f manifests/10-statefulset-webnote.yaml   # partition reset to 0
kubectl rollout status statefulset/webnote --timeout=300s
kubectl get pods -l app=webnote -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```

![Rollout completed, all Pods on new image](../evidence/49.png)

### 8.4 Deleting the Controller, Keeping the Data

```bash
kubectl delete statefulset webnote
kubectl get pods -l app=webnote
kubectl get pvc -l app=webnote --no-headers | wc -l
```

![StatefulSet deleted, PVCs remain](../evidence/50.png)

Deleting the `StatefulSet` object removes the Pods but leaves the PVCs
intact. Recreating it reattaches the existing volumes with no data loss:

```bash
kubectl apply -f manifests/10-statefulset-webnote.yaml
kubectl rollout status statefulset/webnote --timeout=300s
kubectl exec client -- wget -qO- http://webnote-1.webnote.dso202-practical-02.svc.cluster.local
```

![Workload recreated, data preserved](../evidence/51.png)

---

## 9. Stage 7 — A Real Stateful Application: PostgreSQL

### 9.1 Credentials and Services

```bash
kubectl apply -f manifests/12-secret-postgres.yaml
kubectl get secret postgres-credentials
kubectl get secret postgres-credentials -o jsonpath='{.data.POSTGRES_USER}' | base64 -d
```

![Secret created and decoded](../evidence/52.png)

A `Secret` base64-encodes values for storage and transit convenience — it is
**not** encryption at rest by default, which is worth noting explicitly.

```bash
kubectl apply -f manifests/13-service-postgres.yaml
kubectl get services
```

![Both PostgreSQL Services created](../evidence/53.png)

Two Services are defined together: a normal ClusterIP Service for
application connections, and a headless Service for direct per-Pod
addressing — mirroring the pattern established in Stage 5.

### 9.2 Deploying the Database

```bash
kubectl apply -f manifests/14-statefulset-postgres.yaml
kubectl get pods -l app=postgres -w
```

![PostgreSQL Pod starting up](../evidence/54.png)

```bash
kubectl logs postgres-0 | tail -n 4
kubectl get pvc data-postgres-0
kubectl get pv
```

![Postgres logs and provisioned storage](../evidence/55.png)

```bash
kubectl exec postgres-0 -- sh -c 'echo "$PGDATA"; ls /var/lib/postgresql'
```

![PGDATA location inside the volume](../evidence/56.png)

The first startup takes noticeably longer than subsequent ones, since
PostgreSQL must initialise its data directory (`$PGDATA`) on a fresh volume.

### 9.3 Writing Data and Testing Durability

```bash
kubectl exec postgres-0 -- psql -U taskuser -d tasktracker -c \
  "CREATE TABLE tasks (id serial PRIMARY KEY, title text NOT NULL, done boolean NOT NULL DEFAULT false, created_at timestamptz NOT NULL DEFAULT now());"

kubectl exec postgres-0 -- psql -U taskuser -d tasktracker -c \
  "INSERT INTO tasks (title) VALUES ('Complete Practical 2'), ('Read Unit II notes'), ('Draft the report');"

kubectl exec postgres-0 -- psql -U taskuser -d tasktracker -c \
  "SELECT id, title, done FROM tasks ORDER BY id;"
```

![Table created and rows inserted](../evidence/57.png)

Connections here use the container's local Unix socket, which the official
PostgreSQL image trusts without a password inside the Pod.

**The core test of the practical:**

```bash
kubectl delete pod postgres-0
kubectl wait --for=condition=Ready pod/postgres-0 --timeout=180s
kubectl exec postgres-0 -- psql -U taskuser -d tasktracker -c "SELECT count(*) FROM tasks;"
```

![Row count confirms data survived Pod deletion](../evidence/58.png)

All three inserted rows survive Pod deletion and recreation, confirming the
StatefulSet + PVC combination correctly preserves database state
independent of the Pod's lifecycle.

```bash
kubectl logs postgres-0 | head -n 8
```

![Recovery log recognising the existing data directory](../evidence/59.png)

The recovery log lines show PostgreSQL recognising the existing data
directory on the reattached volume rather than reinitialising it.

```bash
kubectl get pvc data-postgres-0
kubectl exec client -- nslookup postgres.dso202-practical-02.svc.cluster.local
kubectl exec client -- nslookup postgres-0.postgres-headless.dso202-practical-02.svc.cluster.local
```

![Claim reused; both DNS forms resolve](../evidence/60.png)

The claim `data-postgres-0` was reused (not recreated), and both DNS forms
resolve correctly — the load-balancing Service name for application use, and
the direct Pod name for operational access.

---

## 10. Stage 8 — Cleanup and the Cost of Retain

### 10.1 Evidence Capture

Before any deletion, the full cluster state was captured for the record:

```bash
mkdir -p evidence
kubectl get all -o wide > evidence/final-state-all.txt
kubectl get pv,pvc,storageclass -o wide > evidence/final-state-storage.txt
kubectl get statefulset webnote -o yaml > evidence/final-statefulset-webnote.yaml
kubectl get events --sort-by=.lastTimestamp > evidence/final-state-events.txt
kubectl exec postgres-0 -- pg_dump -U taskuser -d tasktracker > evidence/tasktracker-dump.sql
```

![Final-state evidence captured](../evidence/61.png)

### 10.2 Deleting Workloads

```bash
kubectl delete -f manifests/14-statefulset-postgres.yaml
kubectl delete -f manifests/10-statefulset-webnote.yaml
kubectl delete -f manifests/11-pod-client.yaml
kubectl delete -f manifests/05-pod-static-writer.yaml
kubectl get pods
```

![All workload Pods removed](../evidence/62.png)

### 10.3 PVCs Are Not Removed Automatically

```bash
kubectl get pvc
```

![PVCs still present after workload deletion](../evidence/63.png)

Deleting workloads (StatefulSets, Deployments, Pods) never deletes the PVCs
they used — this is a deliberate safeguard against accidental data loss and
must be done as an explicit, separate step:

```bash
kubectl delete pvc --all
kubectl get pv
```

![Delete-policy PVs gone, Retain-policy PV Released](../evidence/64.png)

At this point, the two reclaim policies diverge visibly: PVs backed by
`Delete`-policy claims disappear entirely, while PVs backed by
`Retain`-policy claims (the `manual` class) move to `Released` and must be
cleaned up by hand.

### 10.4 Manual Reclamation

```bash
kubectl delete pv pv-web-static
kubectl get pv
docker exec dso202-p2-worker ls /var/local-path-provisioner
```

![PV object removed; provisioner directory checked](../evidence/65.png)

Removing the PV **API object** still does not remove the underlying host
data — that is a separate, manual filesystem operation.

### 10.5 Cluster Teardown

```bash
kubectl config set-context --current --namespace=default
kind delete cluster --name dso202-p2
kind get clusters
docker ps
```

![Cluster deleted, no dso202-p2 containers remain](../evidence/66.png)

### 10.6 The Final Asymmetry

```bash
ls -l /tmp/dso202-p2-storage/pv-web-static/
```

![Host data survives even after cluster deletion](../evidence/67.png)

![Final host cleanup](../evidence/68.png)

Even after the entire `kind` cluster is deleted, the `ledger.txt` file
created under the `Retain`-policy static PV remains on the host filesystem.
This is the practical's closing demonstration: **`Retain` means exactly what
it says, all the way down to the host** — cluster deletion is not, by
itself, a substitute for deliberate storage cleanup.

---

## 11. Key Takeaways

1. **PV/PVC/StorageClass** together decouple storage requests from storage
   implementation, but the binding mode (`Immediate` vs `WaitForFirstConsumer`)
   and reclaim policy (`Retain` vs `Delete`) must be chosen deliberately —
   they materially change operational behaviour.
2. **Static provisioning** ties a Pod to a specific node via `nodeAffinity`;
   **dynamic provisioning** is more flexible but, with `local-path-provisioner`,
   does not enforce requested capacity and cannot expand a class marked
   `allowVolumeExpansion: false`.
3. A **`Deployment`** is the wrong tool for workloads that need durable,
   per-replica identity — the shared-PVC experiment in Stage 4 exposes the
   identity problem that StatefulSets exist to solve.
4. A **`StatefulSet`**, paired with a headless Service, provides stable
   ordinals, stable per-Pod DNS names, and stable per-Pod storage that
   survives Pod deletion, scaling, and even deletion of the StatefulSet
   controller itself.
5. **Partitioned rolling updates** give ordinal-level control over which
   replicas receive an update, enabling canary-style rollouts of stateful
   workloads.
6. Running **PostgreSQL** as a StatefulSet demonstrates these guarantees
   under a real database engine: inserted data survives Pod deletion and the
   claim is reused rather than recreated on restart.
7. **Cleanup requires explicit action at every layer** — deleting workloads
   does not delete claims, deleting claims under `Retain` does not delete
   volumes, deleting volumes does not delete host data, and deleting the
   entire cluster does not delete host data either.

---

## 12. Evidence

Screenshots corresponding to each numbered step (1–68) are stored in the
`evidence/` directory, as referenced throughout the original walkthrough.
Final-state exports (`final-state-all.txt`, `final-state-storage.txt`,
`final-statefulset-webnote.yaml`, `final-state-events.txt`,
`tasktracker-dump.sql`) were captured in Stage 8 prior to teardown and are
also retained in `evidence/`.