# 專案簡介
這是一個CI/CD的練習專案，目的是在Kubernetes上架設CI/CD自動化管線，部屬應用程式。
## 系統架構
![GitOps 架構圖](https://github.com/jeeset/gitops_cicd_pratice/blob/main/GitOps%20%E6%9E%B6%E6%A7%8B%E5%9C%96.png)

## 開發環境與工具
- OS: Ubuntu 24.04
- Kubernetes 叢集: kubeadm 1.37.0
- CI 工具: GitHub Actions
- CD 工具: ArgoCD 
- 組態管理工具: kustomize
- Container Registry: [quay.io](https://quay.io/repository/jeeset/flask_3.1.3?tab=tags)
## Repo 結構
[flask_cicd_pratice](https://github.com/jeeset/flask_cicd_pratice)
```
App Repo - flask_cicd_pratice
├── .github/
│   └── workflows/
│       └── flask_ci.yml      # CI 管線:build → push image → 更新 GitOps repo tag
├── Dockerfile                # python:3.12-slim，安裝 flask 後啟動 flask_app.py
├── flask_app.py              # Flask web app，提供 / 、/hello、/healthz 三個路由
├── requirements.txt          # flask == 3.1.3
├── .gitignore
└── README.md

Gitops Repo - gitops_cicd_pratice/
├── argocd/
│   └── hello-app-application.yml   # ArgoCD Application(第一次需手動 apply 一次)
├── kustomization.yml               # kustomize 管理，CI 在此更新 image tag
├── flask_web_app_namespace.yml     # Namespace: hello-app-ns
├── flask_web_app_deployment.yml    # Deployment: replicas 3 + 三種 probe + resources
├── flask_web_app_service.yml       # Service: ClusterIP， port 86 -> targetPort 5000
├── flask_web_app_ingress.yml       # Ingress: nginx ,host hello.local
└── README.md
```
## 建置步驟

```
cd ~
git clone https://github.com/jeeset/gitops_cicd_pratice.git
kubectl apply -f ~/gitops_cicd_pratice/argocd/hello-app-application.yml
```
## 問題與解決方法
### 1. kubeadm在建置時，自建的pod會一直pending
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- https://ithelp.ithome.com.tw/articles/10331124

因為叢集預設不會在 control plane 節點上排程 Pod，所以只有單 node (只有control plane) 的 kubeadm 叢集在建置時，需要移除 taint ，否則自建的 pod 會一直 pending。
解決方法執行以下指令:
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

### CI 自動改 tag 後,部署卻停在舊版

CI 全綠、commit 有進 GitOps Repo、ArgoCD 也 Synced，但叢集裡的 Pod 始終是舊版本，從 ArgoCD UI 的部署詳情發現 image 欄位還是用舊的，回頭檢查 kustomization.yaml，發現 `images:` 底下多出了一筆多餘的 image entry，原因是 flask_ci.yml 中 kustomize edit set image hello-app=...中的 hello-app 這個名稱與 kustomization.yaml 中既有的 `name: flask` 不一致，kustomize 的行為是 `找不到就新增` 而不是報錯，於是每次 CI 都在更新一欄沒有任何資源引用的無效設定。

## 參考資料
- [安装 kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [container runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [使用 kubeadm 創建 cluster](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Argo CD - Kustomize](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/)
- [Argo CD - Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
