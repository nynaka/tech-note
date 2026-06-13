
---
title: containerd
description: containerdの基本的な使い方の解説です。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

containerd
===

containerd は CNCF（Cloud Native Computing Foundation）が管理する、業界標準のコンテナランタイムです。  
Docker の内部でも使われており、Kubernetes のデフォルトランタイムとしても採用されています。

## 概要

| ツール       | 用途                            |
| ------------ | ------------------------------- |
| `containerd` | コンテナランタイムデーモン      |
| `ctr`        | containerd 付属のデバッグ用 CLI |
| `nerdctl`    | Docker 互換の汎用 CLI（推奨）   |
| `crictl`     | Kubernetes CRI デバッグ用 CLI   |

> 日常的な操作には `nerdctl` を使用してください。  
> `ctr` はデバッグ専用です。

---

## インストール

### 公式バイナリからインストール

#### Step 1: containerd のインストール

```bash
# 最新バージョンを https://github.com/containerd/containerd/releases で確認
VERSION="2.1.0"
ARCH="amd64"  # arm64 も利用可能

wget https://github.com/containerd/containerd/releases/download/v${VERSION}/containerd-${VERSION}-linux-${ARCH}.tar.gz
tar Cxzvf /usr/local containerd-${VERSION}-linux-${ARCH}.tar.gz
```

インストールされるバイナリ:
- `/usr/local/bin/containerd`
- `/usr/local/bin/ctr`
- `/usr/local/bin/containerd-shim-runc-v2`

#### Step 2: runc のインストール

```bash
# https://github.com/opencontainers/runc/releases で最新版を確認
wget https://github.com/opencontainers/runc/releases/download/v1.2.6/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc
```

#### Step 3: CNI プラグインのインストール

```bash
mkdir -p /opt/cni/bin
wget https://github.com/containernetworking/plugins/releases/download/v1.6.2/cni-plugins-linux-amd64-v1.6.2.tgz
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.6.2.tgz
```

#### Step 4: systemd サービスの設定

```bash
wget -O /usr/local/lib/systemd/system/containerd.service \
    https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

systemctl daemon-reload
systemctl enable --now containerd
```

---

### apt-get / dnf からインストール（Docker リポジトリ経由）

`containerd.io` パッケージは Docker が配布しています。runc を含みますが CNI プラグインは含まれません。

#### Ubuntu / Debian

```bash
# 必要なパッケージのインストール
apt-get update
apt-get install -y ca-certificates curl gnupg

# Docker の GPG キーを追加
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

# リポジトリの追加
echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
    https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    tee /etc/apt/sources.list.d/docker.list > /dev/null

# containerd.io のインストール
apt-get update
apt-get install -y containerd.io
```

#### RHEL / AlmaLinux / Rocky Linux

```bash
# Docker リポジトリの追加
dnf -y install dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# containerd.io のインストール
dnf install -y containerd.io

# サービスの有効化
systemctl enable --now containerd
```

---

### 設定

デフォルト設定ファイルの生成:

```bash
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

Kubernetes で使用する場合は `systemd` cgroup ドライバーを有効化:

```bash
# /etc/containerd/config.toml を編集
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
    /etc/containerd/config.toml

systemctl restart containerd
```

---

## コンテナのライフサイクル

### nerdctl（推奨）

Docker と同じ操作感で使用できます。

#### イメージ操作

```bash
# イメージの取得
nerdctl pull nginx:alpine
nerdctl pull docker.io/library/redis:alpine

# イメージ一覧
nerdctl images

# イメージの削除
nerdctl rmi nginx:alpine

# イメージのビルド（BuildKit が必要）
nerdctl build -t myapp:latest .
```

#### コンテナの作成・起動

```bash
# 実行（フォアグラウンド、終了後に削除）
nerdctl run --rm -it alpine sh

# バックグラウンドで実行
nerdctl run -d --name nginx -p 8080:80 nginx:alpine

# コンテナの作成のみ（起動しない）
nerdctl create --name mycontainer nginx:alpine

# 停止中コンテナの起動
nerdctl start mycontainer
```

#### コンテナの確認・操作

```bash
# 実行中コンテナの一覧
nerdctl ps

# 全コンテナの一覧（停止中を含む）
nerdctl ps -a

# コンテナの詳細情報
nerdctl inspect mycontainer

# ログの表示
nerdctl logs mycontainer
nerdctl logs -f mycontainer     # フォローモード
nerdctl logs --tail 100 mycontainer

# コンテナ内でコマンド実行
nerdctl exec -it mycontainer sh
nerdctl exec mycontainer ls /app

# リソース使用状況
nerdctl stats
nerdctl top mycontainer
```

#### コンテナの停止・削除

```bash
# コンテナの停止
nerdctl stop mycontainer
nerdctl stop -t 30 mycontainer  # タイムアウト指定（秒）

# コンテナの再起動
nerdctl restart mycontainer

# コンテナの一時停止・再開
nerdctl pause mycontainer
nerdctl unpause mycontainer

# コンテナの強制停止
nerdctl kill mycontainer

# コンテナの削除（停止後）
nerdctl rm mycontainer

# 実行中コンテナを強制削除
nerdctl rm -f mycontainer

# 停止中の全コンテナを削除
nerdctl container prune
```

#### ファイル操作・その他

```bash
# ホストとコンテナ間のファイルコピー
nerdctl cp mycontainer:/app/config.json ./config.json
nerdctl cp ./config.json mycontainer:/app/

# ポート情報の確認
nerdctl port mycontainer

# コンテナの名前変更
nerdctl rename mycontainer newname

# コンテナをイメージとして保存
nerdctl commit mycontainer myimage:v1
```

---

### ctr（デバッグ用）

`ctr` は containerd に付属するデバッグ専用 CLI です。ポートフォワーディングやログ取得など一部機能が不足しているため、通常操作には `nerdctl` を推奨します。

#### 名前空間

containerd はリソースを名前空間で分離しています。

```bash
# 名前空間の一覧
ctr namespaces ls

# 名前空間を指定してコマンド実行
ctr -n k8s.io images ls   # Kubernetes の名前空間
```

#### イメージ操作

```bash
# イメージの取得（完全なレジストリパスが必要）
ctr images pull docker.io/library/nginx:alpine

# イメージ一覧
ctr images ls

# イメージの削除
ctr images rm docker.io/library/nginx:alpine
```

#### コンテナの操作

```bash
# コンテナの作成と実行（1コマンド）
ctr run docker.io/library/redis:alpine redis-server

# バックグラウンド実行
ctr run -d docker.io/library/redis:alpine redis-server

# コンテナ一覧
ctr containers ls

# タスク（実行中プロセス）の一覧
ctr tasks ls

# コンテナへの接続
ctr tasks exec --exec-id myexec redis-server sh

# タスクの停止
ctr tasks kill redis-server

# コンテナの削除
ctr containers rm redis-server
```

---

## nerdctl compose（オーケストレーション）

`nerdctl compose` は Docker Compose 互換のオーケストレーションツールです。  
[Compose 仕様](https://github.com/compose-spec/compose-spec)（Docker Compose v3 から派生）を実装しています。

### 前提条件

- nerdctl のインストール
- CNI プラグインのインストール
- BuildKit（`nerdctl compose build` を使用する場合）

### nerdctl のインストール

フルパッケージ（CNI プラグイン・BuildKit・RootlessKit を含む）:

```bash
VERSION="2.0.4"
wget https://github.com/containerd/nerdctl/releases/download/v${VERSION}/nerdctl-full-${VERSION}-linux-amd64.tar.gz
tar Cxzvvf /usr/local nerdctl-full-${VERSION}-linux-amd64.tar.gz
```

nerdctl 単体のインストール:

```bash
VERSION="2.0.4"
wget https://github.com/containerd/nerdctl/releases/download/v${VERSION}/nerdctl-${VERSION}-linux-amd64.tar.gz
tar Cxzvvf /usr/local/bin nerdctl-${VERSION}-linux-amd64.tar.gz
```

### 基本的な使い方

```bash
# サービスの起動（バックグラウンド）
nerdctl compose up -d

# サービスの停止・削除
nerdctl compose down

# ログの確認
nerdctl compose logs
nerdctl compose logs -f web

# サービスの一覧
nerdctl compose ps

# 特定のサービスのスケール
nerdctl compose up -d --scale web=3

# イメージのビルド
nerdctl compose build

# 設定ファイルを指定
nerdctl compose -f docker-compose.yml up -d
```

### docker-compose.yml の例

`nerdctl compose` は `docker-compose.yml` をそのまま使用できます。

```yaml
version: "3"

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    volumes:
      - db_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:alpine
    restart: unless-stopped

volumes:
  db_data:
```

### compose コマンド一覧

| コマンド                  | 説明                             |
| ------------------------- | -------------------------------- |
| `nerdctl compose up`      | サービスの作成・起動             |
| `nerdctl compose down`    | サービスの停止・削除             |
| `nerdctl compose start`   | 停止中サービスの起動             |
| `nerdctl compose stop`    | サービスの停止                   |
| `nerdctl compose restart` | サービスの再起動                 |
| `nerdctl compose ps`      | サービス一覧                     |
| `nerdctl compose logs`    | ログの表示                       |
| `nerdctl compose exec`    | サービスコンテナ内でコマンド実行 |
| `nerdctl compose build`   | イメージのビルド                 |
| `nerdctl compose pull`    | イメージの取得                   |
| `nerdctl compose push`    | イメージのプッシュ               |
| `nerdctl compose pause`   | サービスの一時停止               |
| `nerdctl compose unpause` | サービスの再開                   |
| `nerdctl compose kill`    | サービスの強制停止               |
| `nerdctl compose rm`      | 停止中サービスの削除             |
| `nerdctl compose run`     | 1 回限りのコマンド実行           |
| `nerdctl compose config`  | 設定の検証・表示                 |
| `nerdctl compose top`     | 実行中プロセスの表示             |

---

## 参考リンク

- [containerd 公式サイト](https://containerd.io/)
- [containerd Getting Started](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)
- [containerd リリース](https://github.com/containerd/containerd/releases)
- [nerdctl GitHub](https://github.com/containerd/nerdctl)
- [nerdctl コマンドリファレンス](https://github.com/containerd/nerdctl/blob/main/docs/command-reference.md)
- [nerdctl compose ドキュメント](https://github.com/containerd/nerdctl/blob/main/docs/compose.md)
- [CNI プラグイン リリース](https://github.com/containernetworking/plugins/releases)
- [runc リリース](https://github.com/opencontainers/runc/releases)
