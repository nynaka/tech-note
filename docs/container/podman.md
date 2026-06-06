---
title: podman
description: podmanの基本的な使い方の解説です。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

podman
===

## 1. podman のインストール方法

### Debian / Ubuntu (apt)

podman は公式リポジトリに含まれています（Debian 11以降、Ubuntu 20.10以降）。

```bash
sudo apt update
sudo apt install -y podman
```

**rootless（非ルート権限）運用のための推奨ツール:**
```bash
sudo apt install -y slirp4netns fuse-overlayfs uidmap
```

### Fedora / RHEL / AlmaLinux / Rocky Linux (dnf)

Red Hat 系のディストリビューションでは podman は標準的なツールとして提供されています。

```bash
sudo dnf install -y podman
```

### パッケージが提供されていない場合

古いディストリビューションなどでパッケージが提供されていない場合は、[公式のインストールガイド](https://podman.io/getting-started/installation)を参照してください。基本的には各 OS の最新バージョンにアップグレードすることが推奨されます。

---

## 2. コンテナのライフサイクル

podman は Docker とコマンドの互換性が非常に高く、ほとんどの docker コマンドを podman に置き換えるだけで動作します。

### コンテナイメージの取得

```bash
podman pull <イメージ名>
# 例: podman pull alpine
```
※ docker.io/library/alpine のようにフルネームで指定すると確実です。

### Dockerfile 相当のテンプレファイルの書き方

podman では Dockerfile という名前も使えますが、 Containerfile という名前が推奨されることもあります。  
内容は Docker と同じです。

**例: Containerfile**
```dockerfile
# ベースイメージの指定
FROM alpine:latest

# パッケージのインストールなど
RUN apk add --no-cache bash

# ファイルのコピー
COPY myapp.sh /usr/local/bin/

# 実行コマンド
CMD ["bash"]
```

**ビルド方法:**
```bash
podman build -t my-custom-image .
```

### コンテナの起動

```bash
podman run -d --name my-container <イメージ名>
# 例: podman run -d --name web-server nginx

# ポートマッピング (-p ホストポート:コンテナポート)
podman run -d --name web-server -p 8080:80 nginx

# ボリュームマウント (-v ホストパス:コンテナパス)
podman run -d --name web-server -v /host/data:/data:z nginx
```

> `:z` は SELinux 環境でのラベル付け用オプション。不要な場合は省略可。

### コンテナの一覧表示

```bash
# 起動中のコンテナのみ
podman ps

# すべてのコンテナ（停止中を含む）
podman ps -a
```

### コンテナの中のシェルへの入り方/出方

**入る:**
```bash
podman exec -it <コンテナ名またはID> /bin/bash
# bashがない場合は /bin/sh
```

**出る:**
シェルの中で exit を入力するか、 Ctrl + D を押します。

### コンテナのログ確認

```bash
podman logs <コンテナ名またはID>
# リアルタイムで追いかける場合
podman logs -f <コンテナ名またはID>
```

### コンテナの停止・再起動

```bash
podman stop <コンテナ名またはID>
podman restart <コンテナ名またはID>
```

### コンテナの削除

```bash
podman rm <コンテナ名またはID>
# 実行中のコンテナを強制削除する場合
podman rm -f <コンテナ名またはID>
```

### イメージの管理

```bash
# イメージ一覧
podman images

# イメージの削除
podman rmi <イメージ名またはID>

# イメージの保存と読み込み
podman save -o my-image.tar my-custom-image
podman load -i my-image.tar
```

---

## 3. オーケストレーションツール

Docker Compose に相当するツールとして、主に以下の選択肢があります。

### podman-compose (コミュニティツール)

Docker Compose と同様の docker-compose.yml を使用して複数のコンテナを管理できます。  
Python で書かれており、内部で podman コマンドを呼び出します。

**インストール例 (pip):**

```bash
pip install podman-compose
```

**使い方:**

```bash
podman-compose up -d
podman-compose down
```

### podman compose (公式ラッパー)

Podman 4.0 以降では、 podman compose サブコマンドが用意されています。  
これはバックエンドとして podman-compose や本家 docker-compose を利用するラッパーです。

### podman kube play (推奨される「Podman流」)

Podman は Kubernetes との親和性が高く、Kubernetes のマニフェスト（YAML）を直接読み込んでコンテナ群を起動できます。

**使い方:**
```bash
podman kube play my-pod.yaml

# 停止・削除する場合
podman kube down my-pod.yaml
```

これにより、複数のコンテナを 1 つの「Pod」としてまとめて管理でき、将来的に Kubernetes へ移行する際もスムーズです。

> `podman play kube` は Podman 4.x 以降で `podman kube play` に移行しました。

---

## 4. Podman ネイティブな Pod 操作

Podman は Kubernetes を使わずに単体でも Pod を作成・管理できます。

```bash
# Pod の作成
podman pod create --name my-pod -p 8080:80

# Pod 内でコンテナを起動
podman run -d --pod my-pod --name app nginx
podman run -d --pod my-pod --name sidecar alpine sleep infinity

# Pod の一覧
podman pod ls

# Pod の停止・削除
podman pod stop my-pod
podman pod rm my-pod
```

---

## 5. systemd との連携

### Quadlet（Podman 4.4 以降・推奨）

`$HOME/.config/containers/systemd/` にユニットファイル (`.container`) を配置するだけで、systemd サービスとして管理できます。

**例: `$HOME/.config/containers/systemd/myapp.container`**
```ini
[Unit]
Description=My App Container

[Container]
Image=nginx:latest
PublishPort=8080:80
Volume=/host/data:/data:z

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user start myapp
systemctl --user enable myapp
```

### podman generate systemd（従来方法）

```bash
# 既存コンテナからユニットファイルを生成
podman generate systemd --new --name my-container > ~/.config/systemd/user/my-container.service
systemctl --user daemon-reload
systemctl --user enable --now my-container
```

---

## 補足：Docker との共存

podman-docker パッケージをインストールすると、 docker コマンドを入力した際に自動的に podman が実行されるようにエイリアスが設定されます。
