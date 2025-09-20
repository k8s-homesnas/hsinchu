## K8s Ingress Controller 綜合指南：原理、使用方式與 Port 設定

在 Kubernetes (K8s) 叢集中，服務 (Service) 提供了叢集內部的服務發現與負載平衡，但如何將外部流量安全且高效地導入到叢集內的服務，便是 Ingress Controller 的核心任務。本文件旨在提供一份關於 K8s Ingress Controller 的綜合概述，涵蓋其運作原理、具體使用方式，以及節點上需要開啟的 Port。

### 一、 K8s Ingress Controller 的運作原理

要了解 Ingress Controller，首先需要區分 **Ingress** 與 **Ingress Controller** 這兩個概念：

*   **Ingress**: 是 Kubernetes 中的一個 API 物件，它定義了一組規則，用來管理外部對叢集中服務的存取。 這些規則可以基於主機名稱 (Host) 或 URL 路徑 (Path) 來設定，決定如何將 HTTP 和 HTTPS 的流量轉送到後端的服務 (Service)。 Ingress 物件本身並不能處理流量，它僅僅是一組規則的集合。

*   **Ingress Controller**: 則是實現這些 Ingress 規則的關鍵元件。 它是一個在叢集中運行的 Pod，通常內含一個反向代理和負載平衡器 (例如 Nginx、HAProxy 或 Traefik)。 Ingress Controller 會持續監控 Kubernetes API Server 中 Ingress 物件的變化，並根據這些規則動態地生成並更新反向代理的設定檔，然後重新載入設定，使得流量可以依照定義的規則進行轉送。

**工作流程概述如下：**

1.  **使用者定義 Ingress 規則**: 開發者或管理員透過 YAML 檔案建立一個 Ingress 物件，其中描述了例如 `example.com/api` 的請求應該被轉送到名為 `api-service` 的服務。
2.  **Ingress Controller 監控變化**: 部署在叢集中的 Ingress Controller 會偵測到這個新的 Ingress 物件。
3.  **動態生成設定**: Ingress Controller 根據 Ingress 物件中定義的規則，轉換成其內部反向代理 (如 Nginx) 的設定檔。
4.  **流量路由**: 當外部請求到達 Ingress Controller 時，它會根據已經生效的設定，將請求代理到對應的後端服務及其 Pod。

這種方式提供了極大的靈活性，讓開發者可以透過標準的 Kubernetes API 來管理七層 (L7) 的網路流量，實現諸如負載平衡、SSL/TLS 終止以及基於名稱的虛擬主機等功能。

### 二、 K8s Ingress Controller 的使用方式

使用 Ingress Controller 通常包含以下幾個步驟：

#### 1. 部署 Ingress Controller

Kubernetes 本身並不預設安裝 Ingress Controller，您需要根據需求選擇並部署一個。市面上有許多選擇，其中以社群維護的 **NGINX Ingress Controller** 最為普遍。

您可以使用 `kubectl apply` 指令來部署 NGINX Ingress Controller：

```yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```

這個指令會建立運行 NGINX Ingress Controller 所需的相關資源，包括 Deployment、Service、ConfigMap 等。

#### 2. 部署應用程式與服務

在定義 Ingress 規則之前，您需要先有在叢集中運行的應用程式，並透過 Service 將它們暴露出來。假設我們有兩個服務：`webapp-service` 和 `api-service`。

#### 3. 建立 Ingress 物件

接下來，您可以建立一個 Ingress 物件來定義流量的路由規則。以下是一個範例 YAML 檔案 (`example-ingress.yaml`)：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /webapp
        pathType: Prefix
        backend:
          service:
            name: webapp-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

在這個範例中：

*   我們定義了一個基於主機名稱 `myapp.example.com` 的規則。
*   當請求的路徑是 `/webapp` 開頭時，流量會被送到 `webapp-service` 的 80 Port。
*   當請求的路徑是 `/api` 開頭時，流量會被送到 `api-service` 的 8080 Port。

使用以下指令來建立這個 Ingress 物件：

```bash
kubectl apply -f example-ingress.yaml```

建立完成後，Ingress Controller 就會自動根據這些規則來路由發送到 `myapp.example.com` 的流量。

### 三、 Node 需要開啟的 Port

要讓外部流量能夠到達 Ingress Controller，您需要在叢集的節點 (Node) 上開啟特定的 Port。暴露 Ingress Controller 的方式主要有以下幾種，每種方式對應需要開啟的 Port 也不同：

#### 1. 使用 `NodePort` Service

這是最常見且直接的方式，特別是在裸機 (Bare-metal) 或本地環境中。Ingress Controller 的服務會被設定為 `NodePort` 類型，這會在叢集的**每一個 Node** 上開啟一個隨機但固定的高編號 Port (預設範圍為 **30000-32767**)。 外部流量可以透過 `http://<任何節點的 IP>:<NodePort>` 來存取 Ingress Controller。

在這種設定下，您需要確保防火牆或安全群組允許外部流量存取這個範圍內的 Port。通常，`NodePort` 服務會將外部的 HTTP (80) 和 HTTPS (443) 流量對應到 Node 上的兩個不同 `NodePort`。

#### 2. 使用 `LoadBalancer` Service

在公有雲環境 (如 GCP, AWS, Azure) 中，最推薦的方式是將 Ingress Controller 的服務設定為 `LoadBalancer` 類型。雲端供應商會自動配置一個外部的負載平衡器，並將流量導向到 Ingress Controller。

在這種情況下，外部流量通常是透過標準的 **HTTP (80)** 和 **HTTPS (443)** Port 進入負載平衡器，然後再由負載平衡器轉送到節點上的 `NodePort`。因此，您需要確保雲端平台的防火牆規則允許外部流量進入負載平衡器的 80 和 443 Port。

#### 3. 使用 `hostNetwork: true`

另一種方式是在 Ingress Controller 的 Pod 設定中啟用 `hostNetwork: true`。這會讓 Ingress Controller 的 Pod 直接使用 Node 的網路介面，並監聽 Node 上的 **80** 和 **443** Port。

這種方法的優點是效能較好，因為它減少了網路封包的轉送層級。但缺點是會在 Pod 和 Node 之間產生 Port 衝突的風險，且一個 Node 上只能運行一個使用相同 `hostPort` 的 Pod。

**總結來說，您需要在節點上開啟的 Port 取決於您暴露 Ingress Controller 的方式：**

| 暴露方式 | 建議開啟的 Node Port | 說明 |
| --- | --- | --- |
| **NodePort** | 30000 - 32767 (TCP) | 允許外部流量直接存取每個節點上的高編號 Port。 |
| **LoadBalancer** | 80 (TCP), 443 (TCP) | 雲端負載平衡器會將標準的 HTTP/HTTPS 流量轉送到節點。 |
| **hostNetwork** | 80 (TCP), 443 (TCP) | Ingress Controller Pod 直接監聽節點上的標準 Port。 |
