# kind + ArgoCD で nginx を GitOps デプロイ（本番想定）

このリポジトリは、`kind` 上の Kubernetes クラスタへ ArgoCD を使って `nginx` を GitOps デプロイするための最小かつ本番運用を意識した構成です。

## ディレクトリ構成

```text
.
├── apps/
│   └── nginx/
│       └── base/
│           ├── kustomization.yaml
│           ├── namespace.yaml
│           ├── deployment.yaml
│           └── service.yaml
└── clusters/
    └── kind/
        └── argocd/
            └── nginx-app.yaml
```

- `apps/nginx/base`: アプリ本体（Namespace / Deployment / Service）
- `clusters/kind/argocd/nginx-app.yaml`: ArgoCD `Application` 定義（Auto Sync, Self Heal, Prune 有効）

## 前提

- macOS + Homebrew
- Docker Desktop (or Rancher Desktop) が起動済み
- `kubectl`, `gh` が利用可能

## 1. GitHub リポジトリ作成〜初回 push（実行手順）

```bash
# このディレクトリで実行
git init -b main
git add .
git commit -m "Add production-style GitOps manifests for nginx via ArgoCD"

# GitHubに作成してそのままpush（privateで作成）
gh repo create k8s-argocd-sample --private --source=. --remote=origin --push
```

この時点で作成される URL 例:

```text
https://github.com/<your-user>/k8s-argocd-sample
```

## 2. ArgoCD Application の repoURL を実 URL に更新して push

`clusters/kind/argocd/nginx-app.yaml` の `spec.source.repoURL` を更新:

```yaml
repoURL: https://github.com/<your-user>/k8s-argocd-sample.git
```

反映:

```bash
git add clusters/kind/argocd/nginx-app.yaml
git commit -m "Set ArgoCD repoURL to created GitHub repository"
git push origin main
```

## 3. kind クラスタ作成

```bash
kind create cluster --name argocd-sample
kubectl config use-context kind-argocd-sample
kubectl config current-context
```

期待値:

```text
kind-argocd-sample
```

## 4. ArgoCD インストール

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

もし CRD annotation サイズエラーが出た場合は再実行:

```bash
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

起動確認:

```bash
kubectl -n argocd get pods
```

## 5. ArgoCD Application 適用（GitOps 開始）

```bash
kubectl apply -f clusters/kind/argocd/nginx-app.yaml
kubectl -n argocd get application nginx-prod -o wide
```

## 6. private repo を使う場合の注意

今回の実行では、GitHub リポジトリが private のままだと ArgoCD が取得できず、以下エラーになりました。

- `authentication required: Repository not found`

手早く進める場合は一時的に public 化:

```bash
gh repo edit <your-user>/k8s-argocd-sample --visibility public --accept-visibility-change-consequences
```

その後、再評価をトリガー:

```bash
kubectl -n argocd annotate application nginx-prod argocd.argoproj.io/refresh=hard --overwrite
```

## 7. 動作確認コマンド

```bash
# ArgoCD 側
kubectl -n argocd get application nginx-prod -o wide
kubectl -n argocd describe application nginx-prod

# ワークロード側
kubectl get ns web-prod
kubectl -n web-prod get deploy,pods,svc
kubectl -n web-prod rollout status deploy/nginx

# ローカル疎通
kubectl -n web-prod port-forward svc/nginx 8080:80
curl -i http://127.0.0.1:8080
```

期待値:

- `nginx-prod` が `Synced` / `Healthy`
- `web-prod` namespace が存在
- `deployment/nginx` が `3/3` Ready

## 運用ポイント

- Auto Sync: Git 変更を自動反映
- Self Heal: 手動変更ドリフトを自動修復
- Prune: Git から削除したリソースをクリーンアップ
- `resources-finalizer.argocd.argoproj.io`: Application 削除時の子リソース管理
- `revisionHistoryLimit` + `retry` で安定運用を補助
