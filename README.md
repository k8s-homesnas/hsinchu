## K8s Flux CD 概述：GitOps 持續交付的核心工具

### 1. 簡介

Flux CD 是一個由 CNCF (Cloud Native Computing Foundation) 託管的專案，它實現了基於 GitOps 原則的持續交付 (Continuous Delivery)。簡單來說，Flux 讓您可以使用 Git 儲存庫作為 Kubernetes 叢集的「單一真實來源 (Single Source of Truth)」。它會持續監控您的 Git 儲存庫，並自動將其中定義的 Kubernetes 配置（例如應用程式、配置、Ingress 等）同步到您的叢集。

### 2. 核心原理 (Principle)

Flux CD 的運作基於以下幾個核心原理：

*   **GitOps：**
    *   **Git 作為單一真實來源 (Single Source of Truth)：** 叢集的所有狀態都以聲明式配置（例如 YAML 檔案）的形式儲存在 Git 儲存庫中。這包括應用程式部署、配置、服務、網路策略等等。
    *   **拉取式部署 (Pull-based Deployment)：** 與傳統的 CI/CD 工具從外部推送到叢集不同，Flux 在叢集內部運行，它會主動從 Git 儲存庫中「拉取」變更。這提升了安全性，因為叢集不再需要暴露憑證給外部系統。
    *   **自動化同步：** Flux 會持續監控 Git 儲存庫中是否有新的提交或分支變更。一旦檢測到變更，它會自動將這些變更應用到 Kubernetes 叢集，確保叢集的實際狀態與 Git 中的期望狀態一致。
    *   **宣告式配置 (Declarative Configuration)：** 您只需宣告期望的最終狀態，而不是定義如何達到該狀態的步驟。Kubernetes 和 Flux 會負責將系統引導到該狀態。
    *   **可觀察性 (Observability)：** 由於所有變更都通過 Git 提交，並且 Flux 會生成事件和日誌，因此可以輕鬆追蹤和審計叢集的任何變更，實現完整的可追溯性。

*   **協調循環 (Reconciliation Loop)：**
    Flux 內部由一系列控制器 (Controllers) 組成，它們共同構成一個協調循環。這些控制器持續執行以下步驟：
    1.  **觀察 Git 儲存庫：** `Source Controller` 負責監控 Git 儲存庫（以及 Helm 儲存庫、S3 儲存桶等）的變更。
    2.  **檢測差異：** 當 Git 儲存庫中發生變更時，控制器會檢測到這些變更。
    3.  **計算期望狀態：** 根據 Git 中的配置，計算出叢集的期望狀態。
    4.  **與實際狀態比較：** 將期望狀態與叢集目前的實際狀態進行比較。
    5.  **應用變更：** 如果存在差異，相關的控制器（例如 `Kustomize Controller` 或 `Helm Controller`）會自動應用必要的 Kubernetes 資源，以使叢集的實際狀態與 Git 中的期望狀態保持一致。

### 3. Flux CD 的主要元件

Flux CD 由多個專用控制器組成，每個控制器負責特定的 GitOps 功能：

*   **Source Controller：** 管理 Git 儲存庫、Helm 儲存庫等外部來源的資料。它負責從這些來源獲取最新的配置。
*   **Kustomize Controller：** 處理 Kustomize 資源，將 Kustomize 配置應用到叢集。
*   **Helm Controller：** 管理 Helm Chart 的部署和升級。
*   **Notification Controller：** 處理通知，例如將部署狀態發送到 Slack、Microsoft Teams 或其他通知服務。
*   **Image Reflector Controller / Image Automation Controller：** 用於自動更新 Git 儲存庫中應用程式的容器鏡像版本（可選功能，但對 CI/CD 非常有用）。

### 4. Node 節點需要開啟的網路 Port

Flux CD 服務本身作為 Kubernetes 叢集內部的 Pod 運行，並**不會在 Node 節點上直接開啟專用的外部進入 Port** 來提供服務。它的運作仰賴於標準的 Kubernetes 網路和 API Server。

然而，為了讓 Kubernetes 叢集正常運作，以及 Flux 能夠完成其任務，**Node 節點需要開啟以下標準的 Kubernetes 網路 Port**：

1.  **Kubelet Ports (節點上 Kubelet 服務使用):**
    *   **`10250/TCP` (Kubelet API):** 用於 Kubernetes 控制平面 (kube-apiserver) 與節點上的 Kubelet 代理進行通信。Kubelet 接收來自 API Server 的指令，管理 Pod 的生命週期、資源監控等。
    *   **`10255/TCP` (Kubelet Read-Only API):** 較舊版本，提供只讀的 Kubelet API，現在通常會被 `10250` 取代或禁用。
    *   **`4194/TCP` (cAdvisor):** Kubelet 內建的容器監控工具，用於收集容器的資源使用數據。
    *   **`10248/TCP` (Kubelet Healthz):** Kubelet 的健康檢查端口。

2.  **API Server Ports (Node 到 Control Plane 的通訊):**
    *   **`6443/TCP` (Kubernetes API Server):** 這是 Kubernetes 控制平面的主要 API 服務端口。所有 Node 上的 Kubelet 都需要能夠通過此端口連接到 API Server，以報告節點狀態、Pod 狀態並接收指令。您的 CI/CD 工具、kubectl CLI 以及 Flux 內部控制器也需要與 API Server 通信。

3.  **Pod Network (Pod 間的通訊):**
    *   **`任意 Port (取決於 CNI)`:** Pod 之間的通信通常透過 Overlay 網路實現 (例如 Calico, Flannel, Cilium 等 CNI 插件)。這些通信通常是內部的，但 CNI 插件可能需要在節點間開啟特定的 UDP 或 TCP Port（例如 Flannel 使用 `8472/UDP`，Calico 使用 `BGP 的 179/TCP` 等）。這取決於您選擇的 CNI 插件配置。

4.  **NodePort Services (如果使用):**
    *   **`30000-32767/TCP & UDP` (預設範圍):** 如果您部署的應用程式透過 `Service` 類型為 `NodePort` 的方式暴露服務，那麼這些服務將會在 Node 節點上打開一個指定範圍內的端口（預設是 `30000-32767`）。**但這與 Flux 本身無關，而是與您通過 Flux 部署的應用程式有關。**

5.  **Flux 對外連接 (Outbound Connections):**
    *   **`443/TCP` (HTTPS):** Flux 控制器（特別是 `Source Controller`）需要能夠從集群內部**出站 (outbound)** 連接到外部的 Git 儲存庫（例如 GitHub, GitLab, Bitbucket 等）。這通常是透過 HTTPS 協定 (`443/TCP`) 完成的。這不是 Node 需要開啟的**入站 (inbound)** 端口，而是 Node 允許進行的**出站**連接。
    *   **其他 Git 協定端口 (極少見):** 如果您的 Git 儲存庫使用 SSH (22/TCP) 或其他非標準端口，則也需要允許出站連接。

**總結來說：**

Flux CD 本身在 Node 節點上**不需要額外開啟任何專門的入站 Port**。它像其他 Kubernetes Pod 一樣，依賴於 Kubernetes 叢集的標準網路基礎設施。您需要確保您的 Kubernetes 節點具備運作良好、能夠與控制平面通信、以及 Pod 間可以相互通信的網路環境，並允許集群內的 Pod (包括 Flux) **出站**連接到您的 Git 儲存庫（主要是 HTTPS `443/TCP`）。

