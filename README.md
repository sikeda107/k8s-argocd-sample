# kind + ArgoCD で nginx を GitOps デプロイ（管理クラスタ/配備クラスタ分離）

このリポジトリは、ArgoCD を動かすクラスタと nginx を配備するクラスタを分離した GitOps 構成です。

- 管理クラスタ: `kind-argocd-mgmt`（ArgoCD）
- 配備先クラスタ: `kind-nginx-prod`（nginx ワークロード）

## ディレクトリ構成

```text
.
├── apps/
│   ├── nginx/base/
│   ├── nginx-ingress/base/
│   └── nginx-gateway/base/
└── clusters/
    └── kind/
        └── argocd/
            ├── nginx-app.yaml
            ├── nginx-ingress-app.yaml
            └── nginx-gateway-app.yaml
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

`nginx-prod` は Ingress/Gateway 同時検証のため、port mapping 分離版で作成します。

```bash
cat > kind-nginx-prod-multi.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: nginx-prod
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 80
        protocol: TCP
      - containerPort: 30443
        hostPort: 443
        protocol: TCP
      - containerPort: 30088
        hostPort: 8088
        protocol: TCP
EOF

kind create cluster --name argocd-mgmt
kind create cluster --config kind-nginx-prod-multi.yaml
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

5) Ingress 入口の確認:

```bash
curl -i --max-time 5 -H "Host: nginx-ingress.local" http://127.0.0.1/
```

期待値:

- `HTTP/1.1 200 OK` が返る

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
kubectl --context kind-nginx-prod apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml
```

4) NGINX Gateway Fabric を導入（Gateway controller）:

```bash
# CRD
kubectl --context kind-nginx-prod apply --server-side -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/main/deploy/crds.yaml

# controller 本体
kubectl --context kind-nginx-prod apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/main/deploy/default/deploy.yaml
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

差分だけを記載します（詳細は `##2` 〜 `##11`）。

1) 既存クラスタ削除:

```bash
kind delete cluster --name argocd-mgmt
kind delete cluster --name nginx-prod
```

2) クラスタ再作成:

```bash
kind create cluster --name argocd-mgmt
kind create cluster --config kind-nginx-prod-multi.yaml
```

3) 必須コントローラ再導入:

```bash
# Ingress
kubectl --context kind-nginx-prod apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Gateway API + NGINX Gateway Fabric
kubectl --context kind-nginx-prod apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml
kubectl --context kind-nginx-prod apply --server-side -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/main/deploy/crds.yaml
kubectl --context kind-nginx-prod apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/main/deploy/default/deploy.yaml
```

4) ArgoCD と Application を再適用:

```bash
kubectl --context kind-argocd-mgmt create namespace argocd
kubectl --context kind-argocd-mgmt apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl --context kind-argocd-mgmt apply -f clusters/kind/argocd/nginx-app.yaml
kubectl --context kind-argocd-mgmt apply -f clusters/kind/argocd/nginx-ingress-app.yaml
kubectl --context kind-argocd-mgmt apply -f clusters/kind/argocd/nginx-gateway-app.yaml
kubectl --context kind-argocd-mgmt -n argocd get applications.argoproj.io -o wide
```

## 13. Ingress / Gateway を同時に検証する

`kind-nginx-prod-multi.yaml` で port mapping を分離しているため、同時に検証できます。

- Ingress: `localhost:80`（nodePort `30080`）
- App Service (Gateway backend): `localhost:8088`（nodePort `30088`）
- Gateway listener: `kubectl port-forward svc/nginx-gateway-nginx 18080:80` 経由の `localhost:18080`

注記:

- `localhost:8088` は `Service/nginx-gateway`（アプリ Service）への入口です。
- `RateLimitPolicy` など Gateway policy の実検証は、Gateway listener Service (`nginx-gateway-nginx`) を使ってください。

確認コマンド:

```bash
kubectl --context kind-nginx-prod -n ingress-nginx get svc ingress-nginx-controller -o wide
kubectl --context kind-nginx-prod -n web-gateway get svc nginx-gateway -o wide
kubectl --context kind-nginx-prod -n web-gateway get svc nginx-gateway-nginx -o wide

curl -i -H "Host: nginx-ingress.local" http://127.0.0.1/
curl -i -H "Host: nginx-gateway.local" http://127.0.0.1:8088/
```

## 14. NGINX Gateway Fabric Policy（ClientSettings / RateLimit）検証

`apps/nginx-gateway/base` には以下の policy を含めています。

- `ClientSettingsPolicy/nginx-gateway-client-settings`
- `RateLimitPolicy/nginx-gateway-rate-limit`（`rejectCode: 429`, `2r/s`）

前提:

- Gateway API CRD は `v1.5.0` 以上
- NGINX Gateway Fabric は `main/edge`（`RateLimitPolicy` CRD を含む）を利用

適用:

```bash
kubectl --context kind-nginx-prod apply -k apps/nginx-gateway/base
kubectl --context kind-nginx-prod -n web-gateway get clientsettingspolicy,ratelimitpolicy
```

RateLimit の実測（Gateway listener 経由）:

```bash
# Terminal A
kubectl --context kind-nginx-prod -n web-gateway port-forward svc/nginx-gateway-nginx 18080:80

# Terminal B
(for i in $(seq 1 100); do (curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: nginx-gateway.local' http://127.0.0.1:18080/ &) ; done; wait) | sort | uniq -c
```

期待値:

- `200` と `429` が混在（例: `1 x 200`, `99 x 429`）
- すべて `200` の場合は `Gateway` ではなくアプリ `Service` 側に当たっていないかを確認

必要に応じて片方だけ止める場合:

```bash
kubectl --context kind-argocd-mgmt -n argocd delete application nginx-ingress-prod
# または
kubectl --context kind-argocd-mgmt -n argocd delete application nginx-gateway-prod
```

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
