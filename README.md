## Repository Structure
```bash
dso202-practical-02/
├── README.md                          
├── cluster/
│   └── kind-cluster.yaml              # Listing 1
├── manifests/
│   ├── 00-namespace.yaml              # Listing 2
│   ├── 01-quota-and-limits.yaml       # Listing 3
│   ├── 02-storageclass-retain.yaml    # Listing 4
│   ├── 03-pv-static.yaml              # Listing 5
│   ├── 04-pvc-static.yaml             # Listing 6
│   ├── 05-pod-static-writer.yaml      # Listing 7
│   ├── 06-pvc-dynamic.yaml            # Listing 8
│   ├── 07-pod-dynamic-writer.yaml     # Listing 9
│   ├── 08-deployment-shared-pvc.yaml  # Listing 10
│   ├── 09-service-webnote.yaml        # Listing 11
│   ├── 10-statefulset-webnote.yaml    # Listing 12
│   ├── 11-pod-client.yaml             # Listing 13
│   ├── 12-secret-postgres.yaml        # Listing 14
│   ├── 13-service-postgres.yaml       # Listing 15
│   └── 14-statefulset-postgres.yaml   # Listing 16
├── evidence/                          
└── report/
    └── practical-02-report.md        
```
![1](evidence/1.png)

## Stage 0: Prerequisites and Verification
### Required software
```bash
docker --version
kind --version
kubectl version --client
```
![2](evidence/2.png)

### Creating the host directory that Stage 2 uses for statically provisioned storage. The directory must exist before the cluster is created, because kind mounts it into a node at creation time.

```bash
mkdir -p /tmp/dso202-p2-storage
ls -ld /tmp/dso202-p2-storage
```
![3](evidence/3.png)

### checking the desk space 
```bash
df -h /var/lib/docker 2>/dev/null || df -h /
```
![4](evidence/4.png)


## Stage 1: Cluster, Namespace, and the Storage Landscape

### Create the clauster

Step 1. Copy Listing 1 into cluster/kind-cluster.yaml, then create the cluster.
```bash
kind create cluster --config cluster/kind-cluster.yaml
```
![5](evidence/5.png)

Step 2. Confirm the nodes and record the mapping between Node object names and Docker container names. Later stages inspect the node filesystem with docker exec, which needs the container name.

```bash
kubectl get nodes -o wide
docker ps --format 'table {{.Names}}\t{{.Status}}'
```
![6](evidence/6.png)

Step 3. Confirm that the host directory reached the intended node.
```bash 
docker exec dso202-p2-worker ls -ld /mnt/dso202-static
```
![7](evidence/7.png)


### Create the namespace, the quota, and the retaining StorageClass
Step 4. Apply Listings 2, 3 and 4, then make the namespace the default for the session so that later commands stay short.
```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl config set-context --current --namespace=dso202-practical-02
kubectl apply -f manifests/01-quota-and-limits.yaml
kubectl apply -f manifests/02-storageclass-retain.yaml
```
![8](evidence/8.png)

Step 5. Inspect the quota. Practical 1 constrained CPU, memory and object counts. Listing 3 adds three storage constraints, which is the part to read closely: a cap on the number of claims, a cap on total requested storage, and a per-StorageClass cap.
```bash
kubectl describe resourcequota dso202-p2-quota
```
![9](evidence/9.png)

### Locate the provisioner
Step 6. Find the StorageClasses and the provisioner behind them.
```bash
kubectl get storageclass
```
![10](evidence/10.png)

Step 7. Find the provisioner Pod. It runs in its own namespace, not in the practical namespace.
```bash
kubectl -n local-path-storage get pods
```
![11](evidence/11.png)

Step 8. Ask the provisioner where it intends to write. The answer is held in a ConfigMap, so this command is the authoritative source rather than any document, including this one.
```bash
kubectl -n local-path-storage get configmap local-path-config -o jsonpath='{.data.config\.json}'
```
![12](evidence/12.png)

check
```bash
kubectl get storageclass
```
![13](evidence/13.png)


## Stage 2 — Static Provisioning, and the Meaning of Retain

### Create the PersistentVolume
Step 1. Apply Listing 5 and inspect the result.
```bash
kubectl apply -f manifests/03-pv-static.yaml
kubectl get pv
```
![14](evidence/14.png)


Step 2. Read the two fields that constrain scheduling.
```bash
kubectl get pv pv-web-static -o jsonpath='{.spec.hostPath.path}{"\n"}'
kubectl get pv pv-web-static -o jsonpath='{.spec.nodeAffinity}{"\n"}'
```
![15](evidence/15.png)

### Claim it and write to it
Step 3. Apply Listing 6 and observe immediate binding.
```bash
kubectl apply -f manifests/04-pvc-static.yaml
kubectl get pvc
```
![16](evidence/16.png)

Step 4. Confirm the match rules by asking what the claim received.
```bash 
kubectl get storageclass manual
```
there was an error the 02-storageclass-retain.yaml defines the name as `dso202-retain`, not `manual` so i change the name to manual in the command to dso202-retain
```bash
kubectl apply -f manifests/02-storageclass-retain.yaml
kubectl get storageclass manual
```
![17](evidence/17.png)


Step 5. Apply Listing 7. The Pod appends one line to a file on the volume each time it starts.
```bash
kubectl apply -f manifests/05-pod-static-writer.yaml
kubectl wait --for=condition=Ready pod/static-writer --timeout=90s
kubectl get pod static-writer -o wide
```
![18](evidence/18.png)

Step 6. Read the file from inside the container, then from the host.
```bash
kubectl exec static-writer -- cat /data/ledger.txt
cat /tmp/dso202-p2-storage/pv-web-static/ledger.txt
```
![19](evidence/19.png)

### Prove the volume outlives the Pod
Step 7. Delete the Pod, recreate it, and read the file again.
```bash 
kubectl delete pod static-writer
kubectl apply -f manifests/05-pod-static-writer.yaml
kubectl wait --for=condition=Ready pod/static-writer --timeout=90s
kubectl exec static-writer -- cat /data/ledger.txt
```
![20](evidence/20.png)

### Delete the claim and observe Released
Step 8. Delete the Pod and then the claim, and watch the phase of the PV.
```bash
kubectl delete pod static-writer
kubectl delete pvc pvc-web-static
kubectl get pv
```
![21](evidence/21.png)

Step 9. Confirm that the data is untouched, then remove the PV object.
```bash
cat /tmp/dso202-p2-storage/pv-web-static/ledger.txt
kubectl delete pv pv-web-static
ls -l /tmp/dso202-p2-storage/pv-web-static/
```
![22](evidence/22.png)

Step 10. Recreate all three objects and read the file one final time.
```bash
kubectl apply -f manifests/03-pv-static.yaml
kubectl apply -f manifests/04-pvc-static.yaml
kubectl apply -f manifests/05-pod-static-writer.yaml
kubectl wait --for=condition=Ready pod/static-writer --timeout=90s
kubectl exec static-writer -- cat /data/ledger.txt
```
![23](evidence/23.png)

## Stage 3 — Dynamic Provisioning, StorageClasses, and Two Uncomfortable Truths

### A claim that will not bind
Step 1. Apply Listing 8, which is a claim and nothing else.
```bash
kubectl apply -f manifests/06-pvc-dynamic.yaml
kubectl get pvc dynamic-data
```
![24](evidence/24.png)
Step 2. Ask why, rather than guessing. The events at the bottom of the description are the answer.
```bash
kubectl describe pvc dynamic-data | tail -n 6
```
![25](evidence/25.png)

### Introduce a consumer
Step 3. Apply Listing 9 and watch both objects.
```bash
kubectl apply -f manifests/07-pod-dynamic-writer.yaml
kubectl wait --for=condition=Ready pod/dynamic-writer --timeout=120s
kubectl get pvc dynamic-data
kubectl get pv
```
![26](evidence/26.png)

Step 4. Find the volume on the node. Read the Pod's node first, then translate it to a container name using the table in section 6.1.
```bash
kubectl get pod dynamic-writer -o wide
docker exec dso202-p2-worker2 ls /var/local-path-provisioner
```
![27](evidence/27.png)

### First uncomfortable truth: the requested size is not a limit here
Step 5. Ask the container how large its 1Gi volume is.
```bash
kubectl exec dynamic-writer -- df -h /data
```
![28](evidence/28.png)

### Second uncomfortable truth: this volume cannot grow
Step 6. Attempt to expand the claim.
```bash 
kubectl patch pvc dynamic-data --type merge -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
```
![29](evidence/29.png)
The rejection comes from the API server, not from the provisioner, and it is caused by allowVolumeExpansion: false on the class. The valuable part of this experiment is that the request failed loudly. A claim whose class permits expansion is grown by editing the same field, after which the volume and then the filesystem are resized. The choice of class is therefore a decision about the future of a workload, taken before any data exists, which is why 1.4.3 belongs in this practical rather than in a later one.

### Compare the two classes, then delete the claim
Step 7. Write to the volume, then delete Pod and claim, and observe what Delete means.
```bash 
kubectl exec dynamic-writer -- cat /data/ledger.txt
kubectl delete pod dynamic-writer
kubectl delete pvc dynamic-data
kubectl get pv
docker exec dso202-p2-worker2 ls /var/local-path-provisioner
```
![30](evidence/30.png)

## Stage 4 — Why a Deployment Cannot Own State

Step 1. Apply Listing 10. The file contains one claim and one Deployment with three replicas, all mounting that single claim.
```bash 
kubectl apply -f manifests/08-deployment-shared-pvc.yaml
kubectl rollout status deployment/shared-writer --timeout=180s
kubectl get pods -l app=shared-writer -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
```
![31](evidence/31.png)

Step 2. Read the file that all three replicas mounted.
```bash 
kubectl exec deploy/shared-writer -- cat /data/visitors.log
```
![32](evidence/32.png)

Step 3. Delete one replica and observe the identity problem.
```bash 
kubectl delete pod -l app=shared-writer --field-selector status.phase=Running --wait=false
sleep 15
kubectl get pods -l app=shared-writer -o custom-columns=NAME:.metadata.name
```
![33](evidence/33.png)
Step 4. Note the failure that this cluster hides, then remove the experiment.
```bash
kubectl delete -f manifests/08-deployment-shared-pvc.yaml
kubectl get pvc
```
![34](evidence/34.png)

### Stage 5 — StatefulSets and Stable Identity

### The headless Service first
Step 1. Apply Listing 11 before the StatefulSet. The Service is what makes per-Pod DNS names exist, and creating it first avoids a window in which Pods are running but unaddressable.
```bash 
kubectl apply -f manifests/09-service-webnote.yaml
kubectl get service webnote
```
![35](evidence/35.png)

### Create the StatefulSet and watch the order
Step 2. Apply Listing 12 and watch. The -w flag keeps the command running; interrupt it with Ctrl-C once three Pods are Running.
```bash 
kubectl apply -f manifests/10-statefulset-webnote.yaml
kubectl get pods -l app=webnote -w
```
![36](evidence/36.png)

Step 3. Inspect the claims the controller created.
```bash 
kubectl get pvc -l app=webnote
```
![37](evidence/37.png)

Step 4. Confirm that placement is now free, because each Pod has its own volume.
```bash 
kubectl get pods -l app=webnote -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP
```
![38](evidence/38.png)

## Address individual Pods by name
Step 5. Start the client Pod from Listing 13 and resolve both DNS forms.
```bash 
kubectl apply -f manifests/11-pod-client.yaml
kubectl wait --for=condition=Ready pod/client --timeout=90s
kubectl exec client -- nslookup webnote.dso202-practical-02.svc.cluster.local
```
![39](evidence/39.png)

Step 6. Fetch the page from one specific Pod.
```bash
kubectl exec client -- wget -qO- http://webnote-1.webnote.dso202-practical-02.svc.cluster.local
```
![40](evidence/40.png)

Step 7. Confirm the addressing mechanism, which is the reverse of a normal Service.
```bash 
kubectl get endpointslice -l kubernetes.io/service-name=webnote
```
![41](evidence/41.png)

## Prove the volumes are private
Step 8. Write a note into one Pod only, then read both Pods.
```bash
kubectl exec webnote-0 -- sh -c 'echo "note added by hand in Stage 5" >> /usr/share/nginx/html/index.html'
kubectl exec client -- wget -qO- http://webnote-0.webnote.dso202-practical-02.svc.cluster.local
kubectl exec client -- wget -qO- http://webnote-1.webnote.dso202-practical-02.svc.cluster.local
```
![42](evidence/42.png)

### Prove that identity and storage survive deletion
Step 9. Delete the middle Pod and watch the replacement.
```bash 
kubectl delete pod webnote-1
kubectl wait --for=condition=Ready pod/webnote-1 --timeout=120s
kubectl get pod webnote-1 -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP
kubectl get pvc content-webnote-1
kubectl exec client -- wget -qO- http://webnote-1.webnote.dso202-practical-02.svc.cluster.local
```
![43](evidence/43.png)

## Stage 6 — Scaling, Retention, and Ordered Updates

### Scale up
Step 1. Add a replica and watch the claim appear with it.
```bash 
kubectl scale statefulset webnote --replicas=4
kubectl wait --for=condition=Ready pod/webnote-3 --timeout=120s
kubectl get pvc -l app=webnote --no-headers | wc -l
```
![44](evidence/44.png)

### Scale down and inspect what remains
Step 2. Remove two replicas and watch the order of termination.
```bash 
kubectl scale statefulset webnote --replicas=2
kubectl get pods -l app=webnote -w
```
![45](evidence/45.png)

Step 3. Count the claims again.
```bash 
kubectl get pvc -l app=webnote
```
![46](evidence/46.png)

Step 4. Scale back to three and read the returning replica.
```bash
kubectl scale statefulset webnote --replicas=3
kubectl wait --for=condition=Ready pod/webnote-2 --timeout=120s
kubectl exec client -- wget -qO- http://webnote-2.webnote.dso202-practical-02.svc.cluster.local
```
![47](evidence/47.png)

### A partitioned rolling update
Step 5. Edit manifests/10-statefulset-webnote.yaml and make two changes:

* set spec.updateStrategy.rollingUpdate.partition to 2;
* change the nginx image tag from nginx:1.30-alpine to nginx:1.31-alpine.
```bash
kubectl apply -f manifests/10-statefulset-webnote.yaml
sleep 30
kubectl get pods -l app=webnote -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```
![48](evidence/48.png)

Step 6. Complete the rollout by setting partition back to 0 in the same file, then apply and watch.
```bash 
kubectl apply -f manifests/10-statefulset-webnote.yaml
kubectl rollout status statefulset/webnote --timeout=300s
kubectl get pods -l app=webnote -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```
![49](evidence/49.png)

### Delete the controller, keep the data
Step 7. Delete the StatefulSet itself and inspect what is left.
```bash 
kubectl delete statefulset webnote
kubectl get pods -l app=webnote
kubectl get pvc -l app=webnote --no-headers | wc -l
```
![50](evidence/50.png)

Step 8. Recreate it and confirm that the workload returns with its data.
```bash 
kubectl apply -f manifests/10-statefulset-webnote.yaml
kubectl rollout status statefulset/webnote --timeout=300s
kubectl exec client -- wget -qO- http://webnote-1.webnote.dso202-practical-02.svc.cluster.local
```
![51](evidence/51.png)


## Stage 7 — A Real Stateful Application: PostgreSQL
### Credentials and Services
Step 1. Apply Listing 14 and confirm what a Secret does and does not do.
```bash
kubectl apply -f manifests/12-secret-postgres.yaml
kubectl get secret postgres-credentials
kubectl get secret postgres-credentials -o jsonpath='{.data.POSTGRES_USER}' | base64 -d
```
![52](evidence/52.png)

Step 2. Apply Listing 15, which contains two Services in one file.
```bash 
kubectl apply -f manifests/13-service-postgres.yaml
kubectl get services
```
![53](evidence/53.png)

### Deploy the database
Step 3. Apply Listing 16 and wait. First start takes longer than later ones, because the database initialises its data directory.
```bash 
kubectl apply -f manifests/14-statefulset-postgres.yaml
kubectl get pods -l app=postgres -w
```
![54](evidence/54.png)

Step 4. Read the log and the storage.
```bash 
kubectl logs postgres-0 | tail -n 4
kubectl get pvc data-postgres-0
kubectl get pv
```
![55](evidence/55.png)

Step 5. Confirm where the data directory actually sits inside the volume.
```bash 
kubectl exec postgres-0 -- sh -c 'echo "$PGDATA"; ls /var/lib/postgresql'
```
![56](evidence/56.png)

## 12.3 Write data and destroy the Pod
Step 6. Create a table and insert rows. The commands below connect over the container's local socket, which the official image trusts, so no password is needed inside the Pod.
```bash 
kubectl exec postgres-0 -- psql -U taskuser -d tasktracker -c \
  "CREATE TABLE tasks (id serial PRIMARY KEY, title text NOT NULL, done boolean NOT NULL DEFAULT false, created_at timestamptz NOT NULL DEFAULT now());"

kubectl exec postgres-0 -- psql -U taskuser -d tasktracker -c \
  "INSERT INTO tasks (title) VALUES ('Complete Practical 2'), ('Read Unit II notes'), ('Draft the report');"

kubectl exec postgres-0 -- psql -U taskuser -d tasktracker -c \
  "SELECT id, title, done FROM tasks ORDER BY id;"
```
![57](evidence/57.png)

Step 7. Delete the database Pod. This is the test the whole practical has been building towards.
```bash 
kubectl delete pod postgres-0
kubectl wait --for=condition=Ready pod/postgres-0 --timeout=180s
kubectl exec postgres-0 -- psql -U taskuser -d tasktracker -c "SELECT count(*) FROM tasks;"
```
![58](evidence/58.png)

Step 8. Read the recovery messages, which show what the database did with the volume on start.
```bash 
kubectl logs postgres-0 | head -n 8
```
![59](evidence/59.png)

Step 9. Confirm the claim was reused rather than recreated, and confirm what the application would connect to.
```bash 
kubectl get pvc data-postgres-0
kubectl exec client -- nslookup postgres.dso202-practical-02.svc.cluster.local
kubectl exec client -- nslookup postgres-0.postgres-headless.dso202-practical-02.svc.cluster.local
```
![60](evidence/60.png)

## Stage 8 — Cleanup, and the Cost of Retain
Step 1. Capture evidence before deleting anything.
```bash 
mkdir -p evidence
kubectl get all -o wide > evidence/final-state-all.txt
kubectl get pv,pvc,storageclass -o wide > evidence/final-state-storage.txt
kubectl get statefulset webnote -o yaml > evidence/final-statefulset-webnote.yaml
kubectl get events --sort-by=.lastTimestamp > evidence/final-state-events.txt
kubectl exec postgres-0 -- pg_dump -U taskuser -d tasktracker > evidence/tasktracker-dump.sql
```
![61](evidence/61.png)

Step 2. Delete the workloads.
```bash 
kubectl delete -f manifests/14-statefulset-postgres.yaml
kubectl delete -f manifests/10-statefulset-webnote.yaml
kubectl delete -f manifests/11-pod-client.yaml
kubectl delete -f manifests/05-pod-static-writer.yaml
kubectl get pods
```
![62](evidence/62.png)

Step 3. Inspect what survived, and note that no command so far has removed a single claim.
```bash 
kubectl get pvc
```
![63](evidence/63.png)

Step 4. Delete the claims explicitly and watch the two reclaim policies diverge.
```bash 
kubectl delete pvc --all
kubectl get pv
```
![64](evidence/64.png)

Step 5. Reclaim the released volumes deliberately, and confirm what removing the API object did not remove.
```bash 
kubectl delete pv pv-web-static
kubectl get pv
docker exec dso202-p2-worker ls /var/local-path-provisioner
```
![65](evidence/65.png)

Step 6. Reset the context, then delete the cluster.
```bash 
kubectl config set-context --current --namespace=default
kind delete cluster --name dso202-p2
kind get clusters
docker ps
```
![66](evidence/66.png)

Step 7. Demonstrate the final asymmetry, then clean the host.
```bash 
ls -l /tmp/dso202-p2-storage/pv-web-static/
```
![67](evidence/67.png)

![68](evidence/68.png)



