---
title: Docker インストール
description: Debian/Ubuntu 系および Fedora/RHEL 系 Linux への Docker Engine インストール手順です。
---

:::note ファイアウォールに関する注意事項

- ufw や firewalld を使用している場合、Docker はコンテナのポートを公開する際にファイアウォールのルールをバイパスします。  
  詳細は [Docker and ufw](https://docs.docker.com/engine/network/packet-filtering-firewalls/#docker-and-ufw) を参照してください。
- Docker は iptables-nft および iptables-legacy のみに対応しています。nft で作成されたファイアウォールルールはサポートされません。
:::

Docker インストール
===

## Debian / Ubuntu 系

| ディストリビューション | 対応バージョン                                              |
| ---------------------- | ----------------------------------------------------------- |
| Debian                 | Trixie 13 / Bookworm 12 / Bullseye 11                       |
| Ubuntu                 | Resolute 26.04 / Questing 25.10 / Noble 24.04 / Jammy 22.04 |

アーキテクチャ: x86_64 (amd64)、armhf (arm/v7)、arm64、ppc64le、s390x

## Fedora / RHEL 系

| ディストリビューション | 対応バージョン |
| ---------------------- | -------------- |
| Fedora                 | 42 / 43 / 44   |
| RHEL                   | 8 / 9 / 10     |

アーキテクチャ: x86_64 (amd64)、aarch64 (arm64)、s390x

---

## Debian / Ubuntu 系 Linux へのインストール

### 古いバージョン・競合パッケージの削除

ディストリビューションが提供する非公式パッケージが残っている場合、公式パッケージと競合することがあります。以下のコマンドで事前に削除してください。

```bash
sudo apt remove \
    $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
```

> `apt` が「パッケージがインストールされていない」と報告する場合は問題ありません。

### apt リポジトリを使ったインストール（推奨）

1. Docker の GPG 鍵とリポジトリを追加

    - Debian Linux

        ```bash
        # Docker の公式 GPG 鍵を追加
        sudo apt update
        sudo apt install -y ca-certificates curl
        sudo install -m 0755 -d /etc/apt/keyrings
        sudo curl \
            -fsSL https://download.docker.com/linux/debian/gpg \
            -o /etc/apt/keyrings/docker.asc
        sudo chmod a+r /etc/apt/keyrings/docker.asc

        # apt リポジトリを追加
        sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
        Types: deb
        URIs: https://download.docker.com/linux/debian
        Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
        Components: stable
        Architectures: $(dpkg --print-architecture)
        Signed-By: /etc/apt/keyrings/docker.asc
        EOF

        sudo apt update
        ```

    - Ubuntu Linux

        ```bash
        # Docker の公式 GPG 鍵を追加
        sudo apt update
        sudo apt install ca-certificates curl
        sudo install -m 0755 -d /etc/apt/keyrings
        sudo curl \
            -fsSL https://download.docker.com/linux/ubuntu/gpg \
            -o /etc/apt/keyrings/docker.asc
        sudo chmod a+r /etc/apt/keyrings/docker.asc

        # apt リポジトリを追加
        sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
        Types: deb
        URIs: https://download.docker.com/linux/ubuntu
        Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
        Components: stable
        Architectures: $(dpkg --print-architecture)
        Signed-By: /etc/apt/keyrings/docker.asc
        EOF

        sudo apt update
        ```

2. Docker パッケージのインストール

    ```bash
    sudo apt install -y \
        docker-ce docker-ce-cli \
        containerd.io docker-buildx-plugin \
        docker-compose-plugin
    ```

3. 自動起動設定

    Debian / Ubuntu 系ではデフォルトで自動起動が有効になっています。Fedora / RHEL 系では `systemctl enable --now docker` で設定済みです。手動で変更する場合は以下を使用してください。

    ```bash
    # 自動起動を有効化
    sudo systemctl enable docker.service
    sudo systemctl enable containerd.service

    # 自動起動を無効化
    sudo systemctl disable docker.service
    sudo systemctl disable containerd.service
    ```

### アンインストール

```bash
# パッケージの削除
sudo apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras

# イメージ・コンテナ・ボリュームの削除
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd

# リポジトリ設定と GPG 鍵の削除
sudo rm /etc/apt/sources.list.d/docker.sources
sudo rm /etc/apt/keyrings/docker.asc
```

---

## Fedora / RHEL 系 Linux へのインストール

### 古いバージョン・競合パッケージの削除

- Fedora

    ```bash
    sudo dnf remove -y \
        docker \
        docker-client \
        docker-client-latest \
        docker-common \
        docker-latest \
        docker-latest-logrotate \
        docker-logrotate \
        docker-selinux \
        docker-engine-selinux \
        docker-engine
    ```

- RHEL

    ```bash
    sudo dnf remove -y \
        docker \
        docker-client \
        docker-client-latest \
        docker-common \
        docker-latest \
        docker-latest-logrotate \
        docker-logrotate \
        docker-engine \
        podman \
        runc
    ```

> `dnf` が「パッケージがインストールされていない」と報告する場合は問題ありません。

### dnf リポジトリを使ったインストール（推奨）

1. Docker リポジトリの追加

    - Fedora

        ```bash
        sudo dnf config-manager addrepo \
            --from-repofile https://download.docker.com/linux/fedora/docker-ce.repo
        ```

    - RHEL

        ```bash
        sudo dnf -y install dnf-plugins-core
        sudo dnf config-manager \
            --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
        ```

2. Docker パッケージのインストール

    ```bash
    sudo dnf install -y \
        docker-ce docker-ce-cli \
        containerd.io docker-buildx-plugin \
        docker-compose-plugin
    ```

    GPG 鍵の受け入れを求められた場合、フィンガープリントが `060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35` であることを確認してから承認してください。

3. Docker の起動と自動起動の有効化

    ```bash
    sudo systemctl start docker
    sudo systemctl enable --now docker
    ```

### アンインストール

```bash
# パッケージの削除
sudo dnf remove -y \
    docker-ce docker-ce-cli \
    containerd.io docker-buildx-plugin \
    docker-compose-plugin docker-ce-rootless-extras

# イメージ・コンテナ・ボリュームの削除
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

---

## インストール後の設定（共通）

### sudo なしで docker コマンドを実行する

:::warning
docker グループのメンバーは root 相当の権限を持ちます。  
セキュリティへの影響を理解した上で設定してください。
:::

デフォルトでは docker コマンドの実行に sudo が必要です。  
一般ユーザーで実行できるようにするには、docker グループにユーザーを追加します。

```bash
# docker グループを作成（既に存在する場合はスキップ）
sudo groupadd docker

# 現在のユーザーを docker グループに追加
sudo usermod -aG docker $USER

# グループの変更を即時反映（または一度ログアウト・ログイン）
newgrp docker
```

---

## 参考リンク

- [Install Docker Engine on Debian](https://docs.docker.com/engine/install/debian/)
- [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Install Docker Engine on Fedora](https://docs.docker.com/engine/install/fedora/)
- [Install Docker Engine on RHEL](https://docs.docker.com/engine/install/rhel/)
- [Linux post-installation steps for Docker Engine](https://docs.docker.com/engine/install/linux-postinstall/)
