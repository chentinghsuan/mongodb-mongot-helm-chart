<p align="right">
  <b>English</b> | <a href="README.zh-TW.md">繁體中文</a>
</p>

# 🍃 MongoDB Community with Native Vector Search (`mongot`) on Kubernetes

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.25+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Helm](https://img.shields.io/badge/Helm-3+-0F1689?logo=helm&logoColor=white)](https://helm.sh/)
[![MongoDB](https://img.shields.io/badge/MongoDB-8.2-47A248?logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> **A lightweight, easy-to-use Kubernetes Helm Chart tailored for UAT, PoC, and local development. It bridges the gap in official tooling by cleanly integrating MongoDB Community Replica Set (`mongod`) with MongoDB Search (`mongot`) on Kubernetes for native Full-Text Search and Vector Search (RAG) — zero Atlas cost and zero heavy Operator overhead.**

---

## 🌟 Why This Project?

1. **Fill the Gap in Official Tooling & Documentation**:
   Although MongoDB released the community edition search engine image (`mongodb/mongodb-community-search`) in 8.0+, neither the official MongoDB Community Kubernetes Operator nor popular charts (such as Bitnami) offer a turnkey deployment or connection guide for community `mongod` + `mongot`. This chart completely encapsulates gRPC communication, mutual keyfile authentication, and dedicated role permissions into a single package.
2. **Lightweight & Fast for UAT / Dev**:
   No need to install, maintain, or troubleshoot complex CRDs and heavy Operators. With a single `helm install` command, spin up a full-featured cluster supporting `$search` and `$vectorSearch` on local clusters (Minikube, Kind, Docker Desktop) or remote staging clusters.
3. **Zero Atlas Cost for Testing**:
   When developing GenAI / RAG applications or performing acceptance testing, there is no need to subscribe to costly MongoDB Atlas tiers just to test vector search, nor do you need to modify private network architectures to reach cloud endpoints.
4. **Out-of-the-Box Automated Initialization**:
   A built-in initialization Job automatically coordinates Replica Set election (`rs.initiate`), creates the `root` administrative user, and provisions the `searchCoordinator` credentials required by `mongot`, removing tedious manual in-container setup steps.

---

## 🏗️ Architecture & Implementation Details

```text
                       +---------------------------------------+
                       |      Kubernetes (UAT / Dev Cluster)   |
                       |                                       |
                       |   +-------------------------------+   |
                       |   |      mongot StatefulSet       |   |
                       |   |    (Lucene / Vector Engine)   |   |
                       |   |  gRPC: 27028 | Metrics: 9946  |   |
                       |   +---------------^---------------+   |
                       |                   | gRPC (27028)      |
                       |                   v                   |
+------------------+   |   +-------------------------------+   |
|   Client / App   +-----> |      mongod StatefulSet       |   |
| (Read/Write/RAG) |   |   |   (1 or 3-Node Replica Set)   |   |
+------------------+   |   +-------------------------------+   |
                       +---------------------------------------+
```

### Key Highlights
* **Native `mongod` + `mongot` Wiring**: Pre-configures `mongod` parameters (`searchIndexManagementHostAndPort` and `useGrpcForSearch`) via `ConfigMap`. `mongot` leverages MongoDB Change Streams to monitor the Replica Set and automatically build Lucene and vector indices.
* **Automated Startup Sequencing**: `mongot` includes a `wait-for-user` InitContainer that waits until the `init` Job completes Replica Set initiation and user setup before launching, preventing startup race conditions and `CrashLoopBackOff`.
* **Permissions & Storage Security**:
  * Containers run non-root (`UID/GID 999`). InitContainers adjust mounted keyfiles and secrets to the strict `0400` permissions required by MongoDB.
  * Configured with `fsGroupChangePolicy: "OnRootMismatch"` to avoid long volume permission scan times on pod restarts.
  * Passwords are securely passed via Kubernetes `Secret` (`secretKeyRef`) environment variables, never exposed in plaintext command-line flags.
* **Flexible Networking**:
  * Internal communication between pods uses Headless Services.
  * Supports assigning independent internal/external LoadBalancer IPs and CIDR source ranges to each node, allowing developers to connect directly from external tools (e.g., MongoDB Compass).

---

## 🚀 Quick Start

### 1. Prerequisites
* Kubernetes 1.25+
* Helm 3+
* `openssl` (to generate the internal authentication Keyfile)

### 2. Generate Replica Set Keyfile
MongoDB Replica Set nodes require a shared Keyfile for internal cluster authentication:

```bash
openssl rand -base64 756 > mongo-keyfile
chmod 400 mongo-keyfile
```

> ⚠️ **Note**: Do not commit `mongo-keyfile` to version control (it is included in `.gitignore` by default).

### 3. Deploy the Helm Chart

Deploy using default parameters, or pass one of the preset files in `examples/`:

```bash
helm upgrade --install my-mongo ./mongodb-search-chart \
  -f examples/values-minimal.yaml \
  --set-file auth.keyfile=./mongo-keyfile \
  --set auth.rootPassword="YourStrongRootPassword" \
  --set auth.mongotPassword="YourStrongMongotPassword"
```

### 4. Verify Vector Search (Run E2E Test)

This project includes a built-in `helm test` script that automatically validates the complete workflow: **Inserts test vector data $\rightarrow$ Creates a `vectorSearch` index via `mongot` $\rightarrow$ Executes a `$vectorSearch` query and verifies results**:

```bash
helm test my-mongo --logs
```

---

## 📂 Configuration Presets & Parameters

### Preset Files (`examples/`)
* **[examples/values-minimal.yaml](examples/values-minimal.yaml)**: Designed for personal testing and local development (Minikube / Kind / Docker Desktop), using single-node and lightweight resource allocations.
* **[examples/values-replica-set.yaml](examples/values-replica-set.yaml)**: Designed for team UAT/staging environments featuring a 3-node Replica Set with comprehensive resource limits and PVC definitions.
* **[examples/values-cloud-lb.yaml](examples/values-cloud-lb.yaml)**: Designed for cloud environments, demonstrating how to assign dedicated LoadBalancer static IPs and CIDR firewall whitelists for external access.

### Key Parameters (`values.yaml`)

| Parameter | Description | Default |
| :--- | :--- | :--- |
| `general.name` | Prefix for cluster and resource names | `"mongodb"` |
| `general.replicaSetName` | MongoDB Replica Set name | `"rs0"` |
| `global.storageClass` | StorageClass for PVCs (uses cluster default if empty) | `""` |
| `auth.keyfile` | Keyfile content for internal node authentication | `""` |
| `auth.rootPassword` | MongoDB `root` administrator password | `""` |
| `auth.mongotPassword` | `mongot` service account (`searchCoordinator`) password | `""` |
| `initJob.enabled` | Automatically run Replica Set initialization and account setup Job | `true` |
| `mongod.replicas` | Number of `mongod` database nodes (set to `1` for local dev) | `3` |
| `mongod.persistence.enabled` | Enable persistent storage (PVC) for `mongod` | `true` |
| `mongod.persistence.size` | Disk size for each `mongod` node | `10Gi` |
| `mongot.replicas` | Number of `mongot` search engine replicas | `1` |
| `mongot.persistence.enabled` | Enable persistent storage (PVC) for `mongot` indices | `true` |
| `mongot.persistence.size` | Index storage size for each `mongot` node | `10Gi` |

---

## 🛡️ License

This project is licensed under the [MIT License](LICENSE).
