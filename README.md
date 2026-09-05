# DSO202 — Practical 02: Kubernetes Storage and Stateful Workloads

## Overview

This practical explores how Kubernetes manages **persistent storage** and
**stateful applications**, moving beyond the stateless workloads covered in
Practical 1. Using a local `kind` (Kubernetes-in-Docker) cluster, the practical
walks through static provisioning, dynamic provisioning, the limitations of
running stateful data on a `Deployment`, and finally StatefulSets as the
correct primitive for stable identity and per-replica storage — culminating in
deploying and testing a real PostgreSQL database.

Guide followed: [DSO202 Practical 02 — HackMD](https://hackmd.io/@sarojsanyasi/dso202-practical-02)

## Learning Objectives

By completing this practical, the following concepts are demonstrated hands-on:

- The relationship between **PersistentVolumes (PV)**, **PersistentVolumeClaims (PVC)**, and **StorageClasses**
- The difference between **static** and **dynamic** provisioning
- How **reclaim policies** (`Retain` vs `Delete`) affect data after a claim or volume is deleted
- Why a `Deployment` with a shared PVC is unsuitable for stateful workloads
- How a **headless Service** combined with a **StatefulSet** provides stable network identity and stable, per-Pod storage
- Ordered Pod creation, scaling, and **partitioned rolling updates** with StatefulSets
- Running a real stateful application (**PostgreSQL**) and verifying that data survives Pod deletion
- The cost and implications of the `Retain` reclaim policy during cleanup

## Repository Structure

```bash
dso202-practical-02/
├── README.md
├── cluster/
│   └── kind-cluster.yaml              # kind cluster definition (multi-node)
├── manifests/
│   ├── 00-namespace.yaml
│   ├── 01-quota-and-limits.yaml
│   ├── 02-storageclass-retain.yaml
│   ├── 03-pv-static.yaml
│   ├── 04-pvc-static.yaml
│   ├── 05-pod-static-writer.yaml
│   ├── 06-pvc-dynamic.yaml
│   ├── 07-pod-dynamic-writer.yaml
│   ├── 08-deployment-shared-pvc.yaml
│   ├── 09-service-webnote.yaml
│   ├── 10-statefulset-webnote.yaml
│   ├── 11-pod-client.yaml
│   ├── 12-secret-postgres.yaml
│   ├── 13-service-postgres.yaml
│   └── 14-statefulset-postgres.yaml
├── evidence/                           # Screenshots captured at each step
└── report/
    └── practical-02-report.md          # Full write-up of the practical
```

## Prerequisites

- Docker
- `kind` (Kubernetes-in-Docker)
- `kubectl`
- A host directory for static provisioning: `/tmp/dso202-p2-storage` (created
  before the cluster starts, since `kind` mounts it into a node at creation time)

## How the Practical Is Organised

| Stage | Focus |
|---|---|
| 0 | Prerequisite checks and host directory setup |
| 1 | Cluster creation, namespace/quota setup, locating the storage provisioner |
| 2 | Static provisioning with a `Retain` StorageClass; proving volumes outlive Pods and claims |
| 3 | Dynamic provisioning; binding behaviour, oversized volumes, and non-expandable claims |
| 4 | Why a `Deployment` cannot safely own shared state |
| 5 | StatefulSets: headless Services, stable DNS names, and private per-Pod volumes |
| 6 | Scaling, PVC retention on scale-down, and partitioned rolling updates |
| 7 | Deploying PostgreSQL as a StatefulSet and verifying data durability |
| 8 | Evidence capture and full cluster/storage cleanup |

## Running the Practical

1. Create the host storage directory and the `kind` cluster (Stage 0–1).
2. Apply manifests in numeric order as instructed in each stage — order matters,
   particularly for the headless Service before the StatefulSet in Stage 5.
3. Capture evidence (screenshots / command output) at each numbered step, as
   referenced in `report/practical-02-report.md`.
4. Follow Stage 8 carefully during cleanup — deleting workloads does **not**
   delete PVCs, and deleting PVCs does **not** delete `Retain`-policy PVs or
   their underlying host data automatically.

## Report

See [`report/practical-02-report.md`](report/practical-02-report.md) for the
full narrative write-up, including command sequences, expected observations,
and the reasoning behind each design decision made in the manifests.