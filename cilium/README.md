## K8s Cilium 概述：原理與節點所需開放的 Port

Cilium 是一款為 Kubernetes 等雲原生環境設計的高效能網路、可觀測性和安全性解決GCP專案。它利用 Linux 核心中強大的 eBPF (Extended Berkeley Packet Filter) 技術，在核心層級進行網路數據包的處理、監控和過濾，從而提供比傳統基於 iptables 的方案更優越的效能和更細緻的控制。

### Cilium 核心原理

Cilium 的核心在於其利用 eBPF 來實現網路功能，這使其能夠在不修改應用程式程式碼或容器映像檔的情況下，提供深入的網路控制和可見性。以下是其主要原理：

*   **eBPF 基礎：** Cilium 將網路策略、負載平衡等規則轉換為 eBPF 程式，並將這些程式附加到網路介面或核心中的特定掛鉤點上。 當網路封包在核心中流動時，這些 eBPF 程式會被觸發執行，進行快速的封包處理，例如轉發、丟棄或修改。

*   **基於身份的安全性：** 傳統的網路防火牆主要依賴 IP 位址和埠號進行封包過濾。Cilium 則採用了基於身份的安全模型。 它為每個 Pod 或容器分配一個唯一的身份標識，並將這個身份資訊嵌入到網路封包中。 當封包在節點間傳輸時，接收端的 Cilium 可以驗證此身份，並根據預先定義的網路策略來決定是否允許該連線。 這種方式與 Pod 的生命週期緊密結合，不受 Pod IP 位址變化的影響。

*   **取代 Kube-proxy：** Cilium 能夠完全取代 Kubernetes 中的 kube-proxy 元件。 它使用 eBPF 實現了分散式的負載平衡，將服務的流量直接導向到後端的 Pod，避免了傳統 iptables 規則的複雜性和效能瓶頸。

*   **L7 政策與可觀測性：** Cilium 不僅能理解 L3/L4 的網路資訊，還能解析 L7 的應用層協定，例如 HTTP、gRPC 和 Kafka。這使得它可以實作更精細的網路策略，例如限制特定 API 的存取。此外，Cilium 也提供了強大的可觀測性功能，能夠監控和收集網路流量數據，並與 Hubble 等工具整合進行分析。

### Cilium 元件

Cilium 主要由以下幾個元件組成：

*   **Cilium Agent:** 在每個 Kubernetes 節點上以 DaemonSet 的形式運作，負責監聽來自 Kubernetes API Server 的事件，管理該節點上的 eBPF 程式，並執行網路策略、負載平衡等功能。
*   **Cilium Operator:** 負責管理整個叢集範圍內的任務，例如 IP 位址管理 (IPAM) 和維護 Cilium Agent 與 Kubernetes 的整合。
*   **Cilium CLI:** 一個命令列工具，讓使用者可以與節點上的 Cilium Agent 進行互動，檢查其狀態和設定。
*   **CNI Plugin:** 負責在 Pod 啟動或終止時，與節點上的 Cilium Agent 互動，為 Pod 設定網路。

### 節點需要開啟的 Port

為了讓 Cilium 正常運作，需要在防火牆或安全群組中為節點開啟特定的 Port。所需的 Port 會根據您的 Cilium 部署模式而有所不同：

| 模式 | 協定 | Port | 說明 |
| :--- | :--- | :--- | :--- |
| **VXLAN (預設)** | UDP | 8472 | 用於節點之間的 VXLAN 覆蓋網路通訊。 |
| **Geneve** | UDP | 6081 | 用於節點之間的 Geneve 覆蓋網路通訊。 |
| **WireGuard (加密)** | UDP | 51871 | 用於 WireGuard 加密通道。 |
| **IPsec (加密)** | ESP | - | 需要允許 ESP 協定流量通過。 |
| **Cilium Health Check** | TCP | 4240 | 用於 cilium-health 在節點間進行健康狀態監控。 |
| **ICMP** | ICMP | Type 8, Code 0 (Echo Request) / Type 0, Code 0 (Echo Reply) | 用於節點間的連線健康檢查。 |
| **Hubble** | TCP | 4245 | (可選) Hubble Relay 的預設 Port。 |
| **Core Kubernetes Traffic** | TCP | 6443 | Kubernetes API Server。 |
| | TCP | 2379, 2380 | etcd。 |
| | TCP | 10250 | Kubelet。 |

**注意：**

*   在直接路由模式 (Direct Routing) 下，您需要確保您的網路基礎設施允許 Pod 的 IP 位址在節點之間路由。
*   如果您使用 Ingress Controller，可能需要根據設定開啟額外的 Port (例如 80, 443)。
*   上述 Core Kubernetes Traffic 的 Port 是 Kubernetes 核心元件之間通訊所需，確保這些 Port 的連通性對於叢集的穩定運作至關重要。

總結來說，Cilium 透過其創新的 eBPF 技術，為 Kubernetes 提供了高效能、高安全性和高可觀測性的網路解決方案。了解其運作原理並正確設定節點所需的 Port，是成功部署和維運 Cilium 的關鍵。
