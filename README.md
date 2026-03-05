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

`nginx-prod` は `localhost:80/443` で Ingress/Gateway の確認ができるよう、port mapping 付きで作成します。

```bash
cat > kind-nginx-prod.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: nginx-prod
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
EOF

kind create cluster --name argocd-mgmt
kind create cluster --config kind-nginx-prod.yaml
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

## 10. Ingress 用の別 Application 追加（今回実行したコマンド）

1) Application を管理クラスタへ適用:

```bash
kubectl --context kind-argocd-mgmt apply -f clusters/kind/argocd/nginx-ingress-app.yaml
kubectl --context kind-argocd-mgmt -n argocd get application nginx-ingress-prod -o wide
```

2) 新規マニフェストを GitHub へ反映（`app path does not exist` 対策）:

```bash
git add apps/nginx-ingress/base clusters/kind/argocd/nginx-ingress-app.yaml
git commit -m "Add separate ArgoCD app for nginx ingress namespace"
git push origin main
```

3) ArgoCD に再評価をトリガー:

```bash
kubectl --context kind-argocd-mgmt -n argocd annotate application nginx-ingress-prod argocd.argoproj.io/refresh=hard --overwrite
kubectl --context kind-argocd-mgmt -n argocd get application nginx-ingress-prod -o wide
```

4) 配備先クラスタで作成確認:

```bash
kubectl --context kind-nginx-prod get ns web-ingress
kubectl --context kind-nginx-prod -n web-ingress get deploy,pods,svc,ingress
kubectl --context kind-nginx-prod -n web-ingress rollout status deploy/nginx-ingress
```

5) Ingress 入口の診断（未確立時の確認）:

```bash
# Ingress経由（入口未確立なら失敗）
curl -i --max-time 3 -H "Host: nginx-ingress.local" http://127.0.0.1/

# Service直通（成功するか比較）
kubectl --context kind-nginx-prod -n web-ingress port-forward svc/nginx-ingress 8080:80
curl -i --max-time 5 http://127.0.0.1:8080
```

期待値:

- Ingress経由:
  - `nginx-prod` を port mapping なしで作成した場合は `connection refused` などで失敗
  - 本 README の port mapping 付き構成では成功する（`HTTP/1.1 200 OK`）
- Service直通: `HTTP/1.1 200 OK`

6) Ingress Controller 経由での到達確認（今回成功した手順）:

```bash
# 8081 が使用中だったため 8082 を使用
kubectl --context kind-nginx-prod -n ingress-nginx port-forward svc/ingress-nginx-controller 8082:80

# 別ターミナル
curl -i --max-time 5 -H "Host: nginx-ingress.local" http://127.0.0.1:8082
```

期待値:

- `HTTP/1.1 200 OK` が返る
- nginx welcome page が表示される

## 11. Gateway API 用の別 Application 追加（今回実行したコマンド）

1) Gateway 用 Application を管理クラスタへ適用:

```bash
kubectl --context kind-argocd-mgmt apply -f clusters/kind/argocd/nginx-gateway-app.yaml
kubectl --context kind-argocd-mgmt -n argocd get application nginx-gateway-prod -o wide
```

2) 新規マニフェストを GitHub へ反映:

```bash
git add apps/nginx-gateway clusters/kind/argocd/nginx-gateway-app.yaml
git commit -m "Add separate ArgoCD Gateway API example on web-gateway namespace"
git push origin main
```

3) 配備先クラスタに Gateway API CRD を導入:

```bash
kubectl --context kind-nginx-prod apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

4) NGINX Gateway Fabric を導入（Gateway controller）:

```bash
# CRD
kubectl --context kind-nginx-prod apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.2.0/deploy/crds.yaml

# annotation サイズエラー時
kubectl --context kind-nginx-prod apply --server-side -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.2.0/deploy/crds.yaml

# controller 本体
kubectl --context kind-nginx-prod apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.2.0/deploy/default/deploy.yaml
kubectl --context kind-nginx-prod -n nginx-gateway rollout status deploy/nginx-gateway --timeout=240s
```

5) 過去の SyncError が残る場合は Application を再作成:

```bash
kubectl --context kind-argocd-mgmt -n argocd delete application nginx-gateway-prod --wait=true
kubectl --context kind-argocd-mgmt apply -f clusters/kind/argocd/nginx-gateway-app.yaml
```

6) 状態確認:

```bash
kubectl --context kind-argocd-mgmt -n argocd get application nginx-gateway-prod -o wide
kubectl --context kind-nginx-prod -n web-gateway get deploy,pods,svc,gateway,httproute
```

期待値:

- `nginx-gateway-prod` が `Synced` / `Healthy`
- `Gateway` が `PROGRAMMED=True`

## 12. kind 再作成後の復旧手順（今回実行したコマンド）

この節は「再作成時の差分」だけを記載します。  
通常の手順は以下を再実行してください。

- クラスタ作成: `## 2`
- ArgoCD 導入: `## 3`
- クラスタ登録: `## 4`
- 各 Application 適用: `## 5`, `## 10`, `## 11`

1) 既存クラスタ削除:

```bash
kind delete cluster --name argocd-mgmt
kind delete cluster --name nginx-prod
```

2) クラスタ再作成（`## 2` をそのまま再実行）:

```bash
kind create cluster --name argocd-mgmt
kind create cluster --config kind-nginx-prod.yaml
```

3) 必須コントローラ再導入:

```bash
# Ingress
kubectl --context kind-nginx-prod apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Gateway API + NGINX Gateway Fabric
kubectl --context kind-nginx-prod apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
kubectl --context kind-nginx-prod apply --server-side -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.2.0/deploy/crds.yaml
kubectl --context kind-nginx-prod apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v2.2.0/deploy/default/deploy.yaml
```

4) 3 Application を再適用:

```bash
kubectl --context kind-argocd-mgmt apply -f clusters/kind/argocd/nginx-app.yaml
kubectl --context kind-argocd-mgmt apply -f clusters/kind/argocd/nginx-ingress-app.yaml
kubectl --context kind-argocd-mgmt apply -f clusters/kind/argocd/nginx-gateway-app.yaml
kubectl --context kind-argocd-mgmt -n argocd get applications.argoproj.io -o wide
```

補足（到達確認の違い）:

- Ingress は `localhost:80` + `Host: nginx-ingress.local` で `200 OK` を確認可能
- Gateway は同じ `localhost:80` では `404` の場合がある（ingress-nginx が受けるため）
- Gateway の確認は controller/service の公開経路に合わせて別ポートで確認する

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
