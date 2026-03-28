---
title: SSH ProxyJump / ProxyCommand
description: 踏み台サーバを経由した SSH 接続の設定例です
sidebar_position: 1
#id: home
#slug: /my-custom-url
---

SSH ProxyJump / ProxyCommand
===

## 構成図

```mermaid
graph LR
    PC["PC (クライアント)"]
    Bastion["踏み台サーバ\n(bastion.example.com)"]
    Target["接続先サーバ\n(target.example.com)"]

    PC -->|"SSH (22)"| Bastion
    Bastion -->|"SSH (22)"| Target
```

## ProxyCommand を使う方法

踏み台サーバを経由して接続先サーバへ SSH 接続します。

```bash title="ProxyCommand でのSSH接続"
ssh -o ProxyCommand="ssh -W %h:%p user@bastion.example.com" user@target.example.com
```

## ProxyJump を使う方法

OpenSSH 7.3 以降で利用できる、よりシンプルな記法です。

```bash title="ProxyJump でのSSH接続"
ssh -J user@bastion.example.com user@target.example.com
```

複数の踏み台を経由する場合はカンマ区切りで指定できます。

```bash title="複数の踏み台を経由する場合"
ssh -J user@bastion1.example.com,user@bastion2.example.com user@target.example.com
```

## ~/.ssh/config を使う方法

`~/.ssh/config` に設定を記述することでコマンドを簡略化できます。

```config title="~/.ssh/config"
# 踏み台サーバ
Host bastion
    HostName bastion.example.com
    User user
    IdentityFile ~/.ssh/id_ed25519

# 接続先サーバ（ProxyJump を使用）
Host target
    HostName target.example.com
    User user
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump bastion
```

設定後は以下のコマンドで接続できます。

```bash title="config を使った接続"
ssh target
```
