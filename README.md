# kind + ArgoCD で nginx を GitOps デプロイ（管理クラスタ/配備クラスタ分離）

このリポジトリは、ArgoCD を動かすクラスタと nginx を配備するクラスタを分離した GitOps 構成です。

- 管理クラスタ: `kind-argocd-mgmt`（ArgoCD）
- 配備先クラスタ: `kind-nginx-prod`（nginx ワークロード）

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

## 前提

- Docker Desktop (or Rancher Desktop) 起動済み
- `kind`, `kubectl`, `gh`, `argocd` CLI が利用可能

## 1. GitHub リポジトリ作成と push

```bash
git init -b main
git add .
git commit -m "Add production-style GitOps manifests for nginx via ArgoCD"
gh repo create k8s-argocd-sample --private --source=. --remote=origin --push
```

`clusters/kind/argocd/nginx-app.yaml` の `repoURL` を実URLにして push:

```bash
git add clusters/kind/argocd/nginx-app.yaml
git commit -m "Set ArgoCD repoURL to created GitHub repository"
git push origin main
```

## 2. kind クラスタを2つ作成

```bash
kind create cluster --name argocd-mgmt
kind create cluster --name nginx-prod
```

確認:

```bash
kind get clusters
kubectl config get-contexts -o name
```

## 3. 管理クラスタに ArgoCD をインストール

```bash
kubectl --context kind-argocd-mgmt create namespace argocd
kubectl --context kind-argocd-mgmt apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

CRD annotation サイズエラー時:

```bash
kubectl --context kind-argocd-mgmt apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

起動確認:

```bash
kubectl --context kind-argocd-mgmt -n argocd get pods
```

## 4. 配備先クラスタを ArgoCD に登録

ArgoCD API にログイン:

```bash
kubectl --context kind-argocd-mgmt -n argocd port-forward svc/argocd-server 8081:443
# 別ターミナル
argocd login localhost:8081 \
  --username admin \
  --password "$(kubectl --context kind-argocd-mgmt -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 --decode)" \
  --insecure
```

配備先クラスタ `kind-nginx-prod` を登録（ArgoCD 内のクラスタ名を `nginx-prod-target` に統一）:

```bash
argocd cluster add kind-nginx-prod --name nginx-prod-target --cluster-endpoint kube-public --yes --insecure
```

`--cluster-endpoint` の意味:

- `kubeconfig`: kubeconfig の `server` を使う
- `kube-public`: `kube-public/cluster-info` を使う
- `internal`: `kubernetes.default.svc` を使う

この手順で `kube-public` を使う理由（重要）:

- kind の kubeconfig は `https://127.0.0.1:<port>` になりやすく、ArgoCD Pod からはその `127.0.0.1` が Pod 自身を指すため配備先 API に届かない。
- その結果、`connection refused` でクラスタ登録・同期が失敗する（今回も `127.0.0.1:60790` で実際に発生）。
- `argocd cluster add ... --cluster-endpoint kube-public` を使うと、Pod から到達可能な API endpoint を使えて解消できる。

登録確認:

```bash
argocd cluster list
```

## 5. Application を管理クラスタへ適用

```bash
kubectl --context kind-argocd-mgmt apply -f clusters/kind/argocd/nginx-app.yaml
kubectl --context kind-argocd-mgmt -n argocd get application nginx-prod -o wide
```

> `nginx-app.yaml` は `destination.name: nginx-prod-target` を参照します。

## 6. private repo を使う場合の注意

private のままだと ArgoCD が取得できず `authentication required` になることがあります。  
その場合はどちらかを実施:

- ArgoCD に GitHub 認証情報（PAT/SSH）を登録
- 一時的に public 化

public 化例:

```bash
gh repo edit <your-user>/k8s-argocd-sample --visibility public --accept-visibility-change-consequences
kubectl --context kind-argocd-mgmt -n argocd annotate application nginx-prod argocd.argoproj.io/refresh=hard --overwrite
```

## 7. 動作確認コマンド

ArgoCD 側（管理クラスタ）:

```bash
kubectl --context kind-argocd-mgmt -n argocd get application nginx-prod -o wide
kubectl --context kind-argocd-mgmt -n argocd describe application nginx-prod
```

nginx 側（配備先クラスタ）:

```bash
kubectl --context kind-nginx-prod get ns web-prod
kubectl --context kind-nginx-prod -n web-prod get deploy,pods,svc
kubectl --context kind-nginx-prod -n web-prod rollout status deploy/nginx
```

疎通確認:

```bash
kubectl --context kind-nginx-prod -n web-prod port-forward svc/nginx 8080:80
curl -i http://127.0.0.1:8080
```

## 8. Auto Deploy 検証（replicas 変更）

```bash
git add apps/nginx/base/deployment.yaml
git commit -m "Scale nginx to 4 replicas"
git push origin main
```

検証後に `replicas` を `3` へ戻す場合:

```bash
git add apps/nginx/base/deployment.yaml
git commit -m "Scale nginx back to 3 replicas"
git push origin main
```

確認:

```bash
kubectl --context kind-argocd-mgmt -n argocd get application nginx-prod -o wide
kubectl --context kind-nginx-prod -n web-prod get deploy nginx
kubectl --context kind-nginx-prod -n web-prod get pods
```

期待値:

- `nginx-prod` が `Synced` / `Healthy`
- `REVISION` が最新 commit SHA
- `deployment/nginx` が Git 定義 replicas で Ready

## 9. ArgoCD UI へのアクセス

1) ArgoCD API Server をローカル公開:

```bash
kubectl --context kind-argocd-mgmt -n argocd port-forward svc/argocd-server 8081:443
```

2) 別ターミナルで初期パスワード取得:

```bash
kubectl --context kind-argocd-mgmt -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 --decode; echo
```

3) ブラウザでアクセス:

- URL: `https://localhost:8081`
- username: `admin`
- password: 2) の出力値

## 運用ポイント

- Auto Sync: Git 変更を自動反映
- Self Heal: 手動変更ドリフトを自動修復
- Prune: Git から削除したリソースをクリーンアップ
- 本番では管理クラスタと配備クラスタを分離して blast radius を抑える

## お掃除コマンド

1) 現在の kind クラスタを確認:

```bash
kind get clusters
```

2) 分離構成をまとめて削除する:

```bash
kind delete cluster --name argocd-mgmt
kind delete cluster --name nginx-prod
```
