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

- kind クラスタ作成済み
- ArgoCD インストール済み（namespace: `argocd`）
- このリポジトリを GitHub で管理していること

## 1. GitHub リポジトリ URL を設定

`clusters/kind/argocd/nginx-app.yaml` の `spec.source.repoURL` を実リポジトリに変更してください。

```yaml
repoURL: https://github.com/<your-org>/<your-repo>.git
```

## 2. マニフェストを GitHub に push

```bash
git add .
git commit -m "Add production-style GitOps manifests for nginx via ArgoCD"
git push origin main
```

## 3. ArgoCD Application を適用

```bash
kubectl apply -f clusters/kind/argocd/nginx-app.yaml
```

## 4. 同期状態を確認

```bash
kubectl -n argocd get applications.argoproj.io
kubectl -n argocd describe application nginx-prod
```

期待状態:
- `SYNC STATUS: Synced`
- `HEALTH STATUS: Healthy`

## 5. nginx のデプロイ確認

```bash
kubectl get ns web-prod
kubectl -n web-prod get deploy,rs,pods,svc
kubectl -n web-prod rollout status deploy/nginx
```

## 6. 動作確認（ローカル疎通）

Port-forward で確認:

```bash
kubectl -n web-prod port-forward svc/nginx 8080:80
curl -i http://127.0.0.1:8080
```

## 運用ポイント

- Auto Sync: Git 変更を自動反映
- Self Heal: 手動変更ドリフトを自動修復
- Prune: Git から削除したリソースをクリーンアップ
- `resources-finalizer.argocd.argoproj.io`: Application 削除時の子リソース管理を明示
- `revisionHistoryLimit` と `retry` を設定し、安定運用を補助

## 補足（ArgoCD CLI がある場合）

```bash
argocd app get nginx-prod
argocd app sync nginx-prod
argocd app wait nginx-prod --health --sync
```
