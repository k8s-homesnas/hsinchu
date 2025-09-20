## K8s Cert-manager 概述

Cert-manager 是一款功能強大且廣受歡迎的 Kubernetes 憑證管理工具，可自動化在 Kubernetes 叢集中獲取、續訂和使用 TLS 憑證的整個生命週期。本文件將詳細介紹其運作原理、使用方式以及節點需要開啟的連接埠。

### 原理

Cert-manager 作為 Kubernetes 的一個附加元件，透過自訂資源定義 (Custom Resource Definitions, CRDs) 來擴充 Kubernetes API，以實現憑證的自動化管理。其核心元件和運作流程如下：

*   **核心元件**：
    *   **控制器 (Controller)**：是 cert-manager 的核心，負責監控其自訂資源的狀態，並協調憑證的簽發、續訂等操作。
    *   **Webhook**：提供驗證 (Validation) 和變異 (Mutation) 的准入控制器 (Admission Controller)。驗證功能可確保 cert-manager 資源的設定正確無誤；變異功能則會為資源設定預設值。
    *   **CA 注入器 (CAInjector)**：負責將 CA 憑證捆綁注入到相關的資源中。

*   **自訂資源 (CRDs)**：
    *   **`Issuer` / `ClusterIssuer`**：代表憑證簽發機構 (Certificate Authority, CA)。`Issuer` 是命名空間級別的資源，只能在其所在的命名空間中簽發憑證；而 `ClusterIssuer` 則是叢集級別的，可以為任何命名空間簽發憑證。Cert-manager 支援多種簽發機構，包括：
        *   ACME (例如 Let's Encrypt)
        *   HashiCorp Vault
        *   Venafi
        *   自簽署 (SelfSigned)
        *   內部 CA
    *   **`Certificate`**：定義了期望的憑證，包括要保護的域名、憑證的儲存位置 (Kubernetes Secret) 以及要使用的 `Issuer` 或 `ClusterIssuer`。

*   **運作流程**：
    1.  使用者建立一個 `Certificate` 資源，指定域名和簽發機構。
    2.  Cert-manager 控制器偵測到新的 `Certificate` 資源後，會建立一個 `CertificateRequest` 資源。
    3.  控制器根據 `CertificateRequest` 中指定的簽發機構，與其進行通訊以簽發憑證。
    4.  在與 ACME 簽發機構 (如 Let's Encrypt) 互動時，cert-manager 會自動完成域名所有權的驗證挑戰。這通常有兩種方式：
        *   **HTTP-01 挑戰**：Cert-manager 會暫時建立一個 Pod 和一個 Ingress 規則，讓 ACME 伺服器可以透過 HTTP (80 連接埠) 存取一個特定的路徑來驗證域名所有權。
        *   **DNS-01 挑戰**：Cert-manager 會透過與您的 DNS 提供商的 API 互動，新增一個特定的 DNS TXT 記錄，ACME 伺服器會查詢此記錄以驗證域名所有權。
    5.  驗證成功後，簽發機構會回傳簽發的憑證。
    6.  Cert-manager 將收到的憑證和私鑰儲存在 `Certificate` 資源中指定的 Kubernetes Secret 中。
    7.  應用程式 (例如 Ingress 控制器) 可以掛載此 Secret 來使用 TLS 憑證。
    8.  Cert-manager 會持續監控憑證的到期時間，並在到期前自動進行續訂，確保服務不中斷。

### 使用方式

使用 cert-manager 的基本步驟如下：

1.  **安裝 Cert-manager**：
    最常見的安裝方式是使用 Helm。
    ```bash
    helm repo add jetstack https://charts.jetstack.io
    helm repo update
    helm install \
      cert-manager jetstack/cert-manager \
      --namespace cert-manager \
      --create-namespace \
      --version v1.13.0 \
      --set installCRDs=true
    ```

2.  **設定 `Issuer` 或 `ClusterIssuer`**：
    建立一個 YAML 檔案來定義您的憑證簽發機構。以下是一個使用 Let's Encrypt 正式環境的 `ClusterIssuer` 範例，採用 HTTP-01 挑戰：
    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: your-email@example.com
        privateKeySecretRef:
          name: letsencrypt-prod-account-key
        solvers:
        - http01:
            ingress:
              class: nginx
    ```
    然後使用 `kubectl apply -f <your-issuer-file>.yaml` 來建立它。

3.  **申請憑證**：
    為您的應用程式建立一個 `Certificate` 資源。以下範例為 `example.com` 申請憑證：
    ```yaml
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: example-com-tls
      namespace: default
    spec:
      secretName: example-com-tls-secret
      issuerRef:
        name: letsencrypt-prod
        kind: ClusterIssuer
      dnsNames:
      - example.com
    ```    同樣地，使用 `kubectl apply -f <your-certificate-file>.yaml` 來建立它。

4.  **在 Ingress 中使用憑證**：
    在您的 Ingress 資源中，引用上面建立的 Secret：
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: my-ingress
      annotations:
        kubernetes.io/ingress.class: "nginx"
    spec:
      tls:
      - hosts:
        - example.com
        secretName: example-com-tls-secret
      rules:
      - host: example.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
    ```
    Cert-manager 會自動為此 Ingress 獲取並續訂憑證。

### 需要 Node 開啟的 Port

為了讓 cert-manager 正常運作，您的 Kubernetes 節點 (或防火牆) 需要允許特定的流量。以下是主要的連接埠需求：

| 元件/情境 | 方向 | 連接埠 | 協定 | 目的 |
| :--- | :--- | :--- | :--- | :--- |
| **Webhook** | Ingress | 10250 (預設) | TCP | Kubernetes API 伺服器需要存取 cert-manager 的 webhook 服務來進行資源的驗證和變異。 **注意**：預設的 10250 連接埠常與 Kubelet 的唯讀連接埠衝突，因此在安裝時通常會透過 Helm 的 `webhook.securePort` 參數將其更改為其他連接埠，例如 **10260**。 |
| **Metrics** | Ingress | 9402 | TCP | Prometheus 或其他監控系統需要存取此連接埠來抓取 cert-manager 的指標數據。 |
| **ACME HTTP-01 挑戰** | Ingress | 80 | TCP | 為了完成 HTTP-01 域名驗證，您的叢集必須能夠從網際網路接收 Let's Encrypt 等 ACME 伺服器在 80 連接埠上的傳入流量。 這些流量會被導向到由 cert-manager 臨時建立的 `acmesolver` Pod。 |
| **ACME DNS-01 挑戰** | Egress | 443 | TCP | Cert-manager 的控制器需要向您的 DNS 提供商的 API 發起 HTTPS 請求，以建立驗證所需的 DNS 記錄。 |
| **內部通訊** | Egress | 443 | TCP | Cert-manager 的所有元件都需要能夠與 Kubernetes API 伺服器進行通訊。 |
| **DNS 查詢** | Egress | 53 | UDP/TCP | 所有 cert-manager 元件都需要能夠執行叢集內部和外部的 DNS 查詢。 |

總結來說，對於一個標準的使用情境，您需要確保：

*   叢集節點的 **80** 連接埠對外開放，以支援 HTTP-01 挑戰。
*   Kubernetes API 伺服器可以存取到 cert-manager webhook 的連接埠 (通常是 **10260**)。
*   若要使用 DNS-01 挑戰，節點需要有對外的 **443** 連接埠存取權限。
*   若要監控 cert-manager，需要允許存取 **9402** 連接埠。
