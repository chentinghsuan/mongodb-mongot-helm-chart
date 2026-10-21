# 🍃 MongoDB Community with Native Vector Search (`mongot`) on Kubernetes

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.25+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Helm](https://img.shields.io/badge/Helm-3+-0F1689?logo=helm&logoColor=white)](https://helm.sh/)
[![MongoDB](https://img.shields.io/badge/MongoDB-8.2-47A248?logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> **專為 UAT 測試環境、PoC 驗證與個人開發打造的輕量級 Kubernetes Helm Chart。一鍵整合社群版 MongoDB Replica Set (`mongod`) 與官方搜尋引擎 (`mongot`)，無須依賴 MongoDB Atlas 或繁重的 Operator，即可在自建 K8s 中快速啟用全文檢索 (Full-Text Search) 與向量搜尋 (Vector Search / RAG)。**
>
> *A lightweight, easy-to-use Helm chart tailored for **UAT, staging, and personal development environments**. It bridges the gap in official tooling by cleanly integrating **MongoDB Community Replica Set (`mongod`)** with **MongoDB Search (`mongot`)** on Kubernetes for native Full-Text Search and Vector Search (RAG).*

---

## 🌟 為什麼要做這個專案？ (Why This Project?)

1. **彌補官方工具與文件的空白 (Fill the Gap in Official Operator & Charts)**：
   MongoDB 雖然在 8.0+ 釋出了社群版搜尋引擎映像檔 (`mongodb/mongodb-community-search`)，但**官方的 Community Kubernetes Operator 以及主流的 Bitnami Helm Chart，都沒有提供社群版 `mongod` + `mongot` 的整合部署方案或串接說明**。本專案直接把兩者的 gRPC 通訊、Keyfile 認證與專屬權限配置完整封裝起來。
2. **輕量、啟動快，極度適合 UAT 與個人測試 (Lightweight & Fast for UAT / Dev)**：
   不需要在叢集中額外安裝與維護複雜的 CRD 與 Operator。透過單一 Helm Chart 指令，就能在既有 K8s 測試環境或本機 (Minikube / Kind) 快速拉起一套支援 `$search` 與 `$vectorSearch` 的環境。
3. **省下測試環境的 Atlas 費用 (Zero Atlas Cost for Testing)**：
   在開發 AI/RAG 應用或進行 UAT 驗收時，不必為了驗證向量搜尋而去租用昂貴的雲端 Atlas 叢集，也不必為了配合雲端連線而修改內網測試架構。
4. **全自動初始化，開箱即用 (Out-of-the-Box Initialization)**：
   內建初始化 Job，部署後自動完成 Replica Set 節點選舉 (`rs.initiate`)，並自動建立 `root` 管理員與 `mongot` 專用的 `searchCoordinator` 帳號，省去手動進容器打指令的繁瑣步驟。

---

## 🏗️ 系統架構與實作細節 (Architecture & Implementation Details)

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

### 核心設計重點 (Key Highlights)
* **`mongod` + `mongot` 原生串接**：在 `ConfigMap` 中預先配置好 `mongod` 的 `searchIndexManagementHostAndPort` 與 `useGrpcForSearch` 參數，並讓 `mongot` 透過 Change Streams 自動監聽 Replica Set 同步建立 Lucene 與向量索引。
* **自動化啟動順序控制**：`mongot` 內建 `wait-for-user` InitContainer，會自動等待 `init` Job 完成 Replica Set 初始化與 `searchCoordinator` 帳號建立後才啟動主程式，避免初次部署時因為時序問題而 Crash。
* **權限與儲存處理**：
  * 容器全程以非 Root (`UID/GID 999`) 運行，透過 InitContainer 將掛載的 Keyfile 與密碼檔轉為 MongoDB 強制要求的 `0400` 權限。
  * 設定 `fsGroupChangePolicy: "OnRootMismatch"`，避免每次測試重啟 Pod 時花時間重複掃描 Volume 權限。
  * 密碼統一由 Kubernetes `Secret` (`secretKeyRef`) 注入環境變數，不將明文密碼寫死於啟動參數中。
* **彈性的網路連線方式**：除了叢集內部透過 Headless Service 連線外，也支援在 UAT 環境為各節點綁定獨立的內部/外部 LoadBalancer IP 與 CIDR 白名單，方便開發人員從外部工具（如 MongoDB Compass）直接連入測試。

---

## 🚀 快速開始 (Quick Start)

### 1. 前置需求 (Prerequisites)
* Kubernetes 1.25+
* Helm 3+
* `openssl`（用於產生 Replica Set 內部節點認證用的 Keyfile）

### 2. 產生 Replica Set Keyfile (Generate Keyfile)
MongoDB Replica Set 節點之間需要共享一把 Keyfile 進行內部驗證：

```bash
openssl rand -base64 756 > mongo-keyfile
chmod 400 mongo-keyfile
```

> ⚠️ **注意 (Note)**：請勿將 `mongo-keyfile` 提交至 Git 版本控制（已預設加入 `.gitignore`）。

### 3. 一鍵部署 Helm Chart (Deploy the Chart)

你可以使用預設參數部署，或指定 `examples/` 資料夾中的環境設定檔：

```bash
helm upgrade --install my-mongo ./mongodb-search-chart \
  -f examples/values-minimal.yaml \
  --set-file auth.keyfile=./mongo-keyfile \
  --set auth.rootPassword="YourStrongRootPassword" \
  --set auth.mongotPassword="YourStrongMongotPassword"
```

### 4. 驗證 Vector Search 功能是否正常 (Run E2E Vector Search Test)

本專案內建 `helm test` 驗證腳本，可一鍵測試「寫入測試向量資料 $\rightarrow$ 透過 `mongot` 建立 `vectorSearch` 索引 $\rightarrow$ 執行 `$vectorSearch` 查詢並比對結果」：

```bash
helm test my-mongo --logs
```

---

## 📂 設定檔範例與參數說明 (Configuration Examples)

### 情境範例檔 (`examples/`)
* **[examples/values-minimal.yaml](examples/values-minimal.yaml)**：適合個人測試或本機開發（Minikube / Kind），採單節點與輕量資源配置。
* **[examples/values-replica-set.yaml](examples/values-replica-set.yaml)**：適合團隊共用 UAT 環境的 3 節點 Replica Set 配置，包含完整的資源限制與 PVC 容量設定。
* **[examples/values-cloud-lb.yaml](examples/values-cloud-lb.yaml)**：適合雲端 UAT 環境，展示如何為各節點配置專屬的 LoadBalancer 靜態 IP 與防火牆來源網段白名單 (CIDR)，方便開發人員從外部連線。

### 主要參數一覽 (`values.yaml`)

| 參數名稱 (Parameter) | 說明 (Description) | 預設值 (Default) |
| :--- | :--- | :--- |
| `general.name` | 叢集與資源名稱前綴 | `"mongodb"` |
| `general.replicaSetName` | MongoDB Replica Set 名稱 | `"rs0"` |
| `global.storageClass` | PVC 使用的 StorageClass（留空則使用叢集預設） | `""` |
| `auth.keyfile` | 節點內部驗證 Keyfile 內容 | `""` |
| `auth.rootPassword` | MongoDB `root` 管理員密碼 | `""` |
| `auth.mongotPassword` | `mongot` 專用帳號 (`searchCoordinator`) 密碼 | `""` |
| `initJob.enabled` | 是否自動執行 Replica Set 初始化與帳號建立 Job | `true` |
| `mongod.replicas` | `mongod` 資料庫節點數量（個人測試可設為 `1`） | `3` |
| `mongod.persistence.enabled` | 是否啟用持久化儲存 (PVC) | `true` |
| `mongod.persistence.size` | 每個 `mongod` 節點的磁碟大小 | `10Gi` |
| `mongot.replicas` | `mongot` 搜尋引擎節點數量 | `1` |
| `mongot.persistence.enabled` | 是否啟用 `mongot` 索引持久化儲存 (PVC) | `true` |
| `mongot.persistence.size` | 每個 `mongot` 節點的索引磁碟大小 | `10Gi` |

---

## 🛡️ 授權條款 (License)

本專案採用 [MIT License](LICENSE) 授權。
