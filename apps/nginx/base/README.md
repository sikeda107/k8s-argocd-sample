# apps/nginx/base

このディレクトリは nginx アプリの base マニフェストです。
ArgoCD は `clusters/kind/argocd/nginx-app.yaml` の `spec.source.path: apps/nginx/base` を参照し、
`kustomization.yaml` を検出して Kustomize として展開・適用します。

## ファイルの役割

- `kustomization.yaml`
  - このディレクトリのエントリポイント。
  - `namespace.yaml` / `deployment.yaml` / `service.yaml` を resources として束ねる。

- `namespace.yaml`
  - 配備先 namespace `web-prod` を定義する。

- `deployment.yaml`
  - nginx Pod（`app.kubernetes.io/name: nginx`）を定義する。
  - replicas、イメージ、probe、resources、securityContext を管理する。

- `service.yaml`
  - `selector: app.kubernetes.io/name: nginx` で Pod を選択する。
  - `port: 80` から Pod 側 `targetPort: http(=8080)` へルーティングする。

## 関係性

1. `namespace.yaml` が `web-prod` を用意する。
2. `deployment.yaml` が `web-prod` に nginx Pod 群を作る。
3. `service.yaml` がその Pod 群への安定したアクセス入口になる。
4. `kustomization.yaml` が 1-3 をまとめて 1 回で適用可能にする。
