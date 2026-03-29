---
title: sshuttle（VPN over SSH）
description: sshuttle を使って SSH サーバ経由でアプリケーションサーバのサブネットに接続する設定例です
sidebar_position: 4
#id: home
#slug: /my-custom-url
---

sshuttle（VPN over SSH）
===

sshuttle は、SSH プロトコルを利用して特定のサブネット宛のトラフィックをトンネリングする、VPN ライクな Python ツールです。

サービスごとに個別のポートフォワーディングを設定する手間がなく、SSH 接続さえ確立できれば、リモートネットワーク上のサブネット全体へ透過的にアクセスできます。  
通常、こうした構成には複雑な IP Forwarding やルーティング設定が必要ですが、sshuttle を使えばそれらを意識することなくシームレスな通信が可能になります。

## インストール

<details>
### Debian / Ubuntu

```bash
sudo apt install sshuttle
```

### Fedora / Alma Linux

```bash
sudo dnf install sshuttle
```

### macOS

```bash
brew install sshuttle
```
</details>


## 実行例

### 構成図

```mermaid
graph LR
    PC["PC\n(sshuttle 実行)"]
    SSH["SSH サーバ\nssh.example.com:22"]
    APP["アプリケーションサーバ\n192.168.10.10:80"]

    PC -->|"SSH トンネル"| SSH
    SSH -->|"アプリ用サブネット\n192.168.10.0/24"| APP
```

### 特定サブネットへの接続

アプリケーションサーバが所属するサブネット（ここでは、192.168.10.0/24）へのトラフィックを SSH サーバ経由でルーティングします。

```bash title="PC 上で実行"
sshuttle -r user@ssh.example.com 192.168.10.0/24
```

| オプション・引数        | 説明                                      |
| ----------------------- | ----------------------------------------- |
| -r user@ssh.example.com | 踏み台となる SSH サーバのユーザ・ホスト名 |
| 192.168.10.0/24         | トンネル経由でルーティングするサブネット  |

接続後、PC から直接アプリケーションサーバの IP・ポートにアクセスできます。

```bash title="PC 上で接続確認"
curl http://192.168.10.10:80
```

### 複数のサブネットを指定する場合

```bash title="PC 上で実行"
sshuttle -r user@ssh.example.com 192.168.10.0/24 192.168.20.0/24
```

### すべてのトラフィックをトンネル経由にする場合

```bash title="PC 上で実行"
sshuttle -r user@ssh.example.com 0.0.0.0/0
```

:::caution
0.0.0.0/0 を指定すると DNS を含むすべてのトラフィックがトンネル経由になります。
:::

### バックグラウンドで実行する場合

```bash title="PC 上で実行（バックグラウンド）"
sshuttle -r user@ssh.example.com 192.168.10.0/24 --daemon --pidfile=/tmp/sshuttle.pid
```

```bash title="切断する場合"
kill $(cat /tmp/sshuttle.pid)
```

### SSH 秘密鍵を指定する場合

```bash title="PC 上で実行"
sshuttle -r user@ssh.example.com 192.168.10.0/24 --ssh-cmd "ssh -i $HOME/.ssh/id_ed25519"
```
