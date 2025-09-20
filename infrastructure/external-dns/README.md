## K8s External-DNS: 自動化 Kubernetes 服務的 DNS 管理

在 Kubernetes (K8s) 環境中，`External-DNS` 是一個至關重要的工具，它能自動化地將叢集內的服務對外暴露的 DNS 記錄同步到外部的 DNS 服務商，從而簡化服務發現的流程並提高效率。本文件將深入探討其運作原理、使用方式，並釐清在節點上所需的連接埠。

### 核心原理

`External-DNS` 的核心原理是作為一個 Kubernetes 控制器，持續監控叢集中的特定資源，並根據這些資源的變化來建立、更新或刪除外部 DNS 服務商中的 DNS 記錄。

其主要運作流程如下：

1.  **監控 Kubernetes 資源**：`External-DNS` 會監控 Kubernetes API 伺服器，以偵測特定資源的變化。它主要關注以下兩種類型的資源：
    *   **服務 (Services)**：特別是 `LoadBalancer` 類型的服務，`External-DNS` 會讀取其外部 IP 位址。對於 `NodePort` 類型的服務，它可以設定為將節點的 IP 作為 DNS 記錄。
    *   **Ingress**：`External-DNS` 會從 Ingress 資源中取得主機名稱 (hostname) 和負載平衡器的 IP 位址。

2.  **讀取註解 (Annotations)**：開發者可以透過在 Service 或 Ingress 資源上添加特定的註解，來告知 `External-DNS` 應該為該資源建立什麼樣的 DNS 記錄。最核心的註解是 `external-dns.alpha.kubernetes.io/hostname`，用以指定所需的域名。

3.  **與 DNS 服務商同步**：`External-DNS` 支援多家主流的 DNS 服務商，例如 AWS Route 53、Google Cloud DNS、Azure DNS、Cloudflare 以及其他自架的 DNS 服務 (如 CoreDNS、BIND)。它會使用預先配置好的憑證，透過各家服務商提供的 API 來進行 DNS 記錄的同步操作。

4.  **記錄所有權 (Ownership)**：為了避免意外刪除不由其管理的 DNS 記錄，`External-DNS` 會在它所建立的每筆 DNS 記錄中，額外添加一筆 TXT 記錄，用以標示該記錄的所有權。當 `External-DNS` 刪除 A 記錄時，也會一併移除對應的 TXT 記錄。

### 使用方式

要在 Kubernetes 叢集中使用 `External-DNS`，通常需要完成以下幾個步驟：

1.  **部署 External-DNS**：`External-DNS` 本身是以一個 Pod 的形式運行在 Kubernetes 叢集中。部署時需要定義相關的 RBAC 規則，包括 `ServiceAccount`、`ClusterRole` 和 `ClusterRoleBinding`，以確保 `External-DNS` 有足夠的權限來讀取叢集中的 Service 和 Ingress 資源。

2.  **配置 DNS 服務商憑證**：為了讓 `External-DNS` 能夠存取您的 DNS 服務商，您需要提供必要的憑證。這通常是透過 Kubernetes Secret 來安全地儲存，並在 `External-DNS` 的部署配置中掛載使用。

3.  **設定啟動參數**：在 `External-DNS` 的部署配置中，需要透過啟動參數來指定要使用的 DNS 服務商 (`--provider`)、要監控的資源類型 (`--source`)，以及其他相關的配置。

4.  **為資源添加註解**：部署並配置好 `External-DNS` 後，您就可以開始在需要對外暴露的 Service 或 Ingress 資源中添加註解，以觸發 DNS 記錄的自動建立。

**範例：為一個 LoadBalancer 類型的 Service 建立 DNS 記錄**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    external-dns.alpha.kubernetes.io/hostname: my-app.example.com.
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: my-app
```

當這個 Service 成功建立並取得一個外部 IP 後，`External-DNS` 會自動在 `example.com` 這個 DNS zone 中，建立一筆 A 記錄，將 `my-app.example.com` 指向該外部 IP。

### Node 需要開啟的 Port

一個常見的誤解是 `External-DNS` 需要在 Kubernetes 節點上開啟特定的**傳入 (inbound)** 連接埠才能運作。事實上，**`External-DNS` 本身並不需要在節點上開啟任何特定的傳入連接埠**。

`External-DNS` 是以一個 Pod 的形式在叢集中運行，其主要的網路通訊是**傳出 (outbound)** 的：

*   **與 Kubernetes API 伺服器通訊**：`External-DNS` Pod 需要能夠存取 Kubernetes API 伺服器，以監控資源的變化。這個通訊通常是叢集內部的，並且會使用 API 伺服器指定的連接埠（通常是 443 或 6443）。
*   **與外部 DNS 服務商通訊**：`External-DNS` Pod 需要能夠向外部 DNS 服務商的 API 端點發送請求。這通常是透過 HTTPS 協定，也就是 TCP 的 443 連接埠。

因此，您需要確保的是：

*   **節點的防火牆或安全群組規則，允許 `External-DNS` Pod 的傳出流量** 到達 Kubernetes API 伺服器以及您所使用的 DNS 服務商的 API 端點。
*   如果您在叢集中使用了**網路策略 (Network Policies)**，您需要設定一個 `Egress` 規則，明確允許 `External-DNS` Pod 對外的通訊。

總結來說，`External-DNS` 是一個強大且靈活的工具，能夠顯著簡化 Kubernetes 中服務的 DNS 管理。了解其原理和正確的使用方式，並釐清其網路通訊模型，將有助於您更安全、更有效地將應用程式暴露給外部使用者。
