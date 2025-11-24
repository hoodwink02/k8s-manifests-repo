# Kubernetes Manifests for Python App (GitOps)

此儲存庫包含 Python Flask 應用程式的 Kubernetes 部署清單 (Manifests)。它是 GitOps 工作流程中的**「狀態來源 (Source of Truth)」**，專供 ArgoCD 監控並同步至 K3s 叢集。

當應用程式原始碼庫 (`python-app-repo`) 完成建置並推送新的 Docker Image 後，CI 流程會自動更新此儲存庫中的 `deployment.yaml` 映像檔版本，觸發 ArgoCD 的自動部署。

## 📂 檔案結構與說明 (Manifests Description)

本專案包含以下標準的 Kubernetes 資源定義：

| 檔案名稱 | 資源類型 | 命名空間 | 說明 |
| :--- | :--- | :--- | :--- |
| **`namespace.yaml`** | `Namespace` | - | 定義專用的命名空間 `python-app-ns`，用於隔離應用程式資源。 |
| **`deployment.yaml`** | `Deployment` | `python-app-ns` | 定義應用程式的部署策略：<br>• **Replicas**: 2 個副本以確保高可用性。<br>• **Image**: 指向 GHCR (`ghcr.io/hoodwink02/python-app`)。<br>• **ImagePullSecrets**: 使用 `ghcr-creds` 以拉取私有映像檔。 |
| **`service.yaml`** | `Service` | `python-app-ns` | 定義網路服務：<br>• **Type**: `NodePort` (允許從節點外部存取)。<br>• **Port Mapping**: 將容器的 `5000` 埠對應至 Service 的 `80` 埠。 |

## ⚙️ GitOps 工作流程 (Architecture)

1.  **開發與建置**: 開發者推送程式碼至 App Repo，GitHub Actions 建置 Docker Image 並推送至 GHCR。
2.  **自動更新**: GitHub Actions 自動修改此儲存庫 `deployment.yaml` 中的 `image` 標籤 (Tag)。
3.  **同步部署**: ArgoCD 偵測到此儲存庫的 Git Commit 變更，自動將新版本同步 (Sync) 至 Kubernetes 叢集。

## 🚀 部署需求 (Prerequisites)

若要將此配置部署到您的叢集（如 K3s），請確保已滿足以下條件：

1.  **Kubernetes Cluster**: 已安裝 K3s 或其他 K8s 發行版。
2.  **ArgoCD**: (選用) 已在叢集中安裝並配置好 Application 指向此 Repo。
3.  **GHCR 認證 Secret**:
    由於映像檔儲存於 GitHub Container Registry (GHCR)，您必須在 `python-app-ns` 命名空間中建立名為 `ghcr-creds` 的 Secret，Deployment 才能成功拉取映像檔。

    ```bash
    # 建立 Namespace
    kubectl apply -f namespace.yaml

    # 建立 Secret (範例)
    kubectl create secret docker-registry ghcr-creds \
      --docker-server=ghcr.io \
      --docker-username=<YOUR_GITHUB_USERNAME> \
      --docker-password=<YOUR_GITHUB_PAT> \
      -n python-app-ns
    ```

## 🛠️ 手動部署 (Manual Deployment)

如果不使用 ArgoCD，您也可以透過 `kubectl` 手動部署：

```bash
# 1. 建立命名空間
kubectl apply -f namespace.yaml

# 2. (重要) 確保已建立 ghcr-creds secret (參考上方說明)

# 3. 部署應用程式與服務
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# 部署完成後，您可以使用以下指令檢查狀態：
kubectl get all -n python-app-ns
```
