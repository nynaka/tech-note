---
title: ufw
description: Linux のファイアウォールツール ufw のインストール方法と、設定確認・ポートの開閉・NAPT の設定などの基本操作をまとめたドキュメントです。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

ufw (Uncomplicated Firewall)
===

## インストール

### apt (Debian / Ubuntu)

```bash
sudo apt update
sudo apt install ufw
```

### dnf (Fedora / RHEL / AlmaLinux など)

```bash
sudo dnf install ufw
```

---

## 各種操作

### 設定確認

```bash
# ufwの有効/無効とルール一覧を表示
sudo ufw status

# 詳細なルール一覧（ポート番号や方向も表示）
sudo ufw status verbose

# ルールに番号を付けて表示（削除時に使用）
sudo ufw status numbered
```

### 有効化 / 無効化

```bash
# ufwを有効化（SSHが切れないよう注意）
sudo ufw enable

# ufwを無効化
sudo ufw disable

# ufwをリセット（全ルールを削除し無効化）
sudo ufw reset
```

### ポートを開ける

```bash
# 特定ポートを開ける（TCP）
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# TCP/UDP 両方を開ける
sudo ufw allow 53

# サービス名で指定する
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https

# 特定のIPアドレスからのアクセスを許可
sudo ufw allow from 192.168.1.0/24

# 特定IPアドレスから特定ポートへのアクセスを許可
sudo ufw allow from 192.168.1.100 to any port 22

# ポート範囲を開ける
sudo ufw allow 8000:8080/tcp

# mDNS（マルチキャストDNS）を許可 UDP 5353
sudo ufw allow 5353/udp

# Samba を許可 TCP 139,445 / UDP 137,138
sudo ufw allow samba
```

### ポートを閉める

```bash
# 特定ポートを閉める（拒否）
sudo ufw deny 80/tcp

# 番号を指定してルールを削除
sudo ufw status numbered
sudo ufw delete 3          # 番号3のルールを削除

# ルールの内容を指定して削除
sudo ufw delete allow 80/tcp
sudo ufw delete allow ssh
```

### NAPTの設定

UFW 単体では NAPT（IP マスカレード）の設定はできないため、`/etc/ufw/before.rules` に iptables の `POSTROUTING` ルールを追加します。

#### 1. IPフォワーディングを有効化

```bash
sudo nano /etc/ufw/sysctl.conf
```

以下の行のコメントを外す（または追記）:

```
net/ipv4/ip_forward=1
```

あるいは `/etc/sysctl.conf` に直接書く場合:

```bash
sudo nano /etc/sysctl.conf
```

```
net.ipv4.ip_forward=1
```

設定を反映:

```bash
sudo sysctl -p
```

#### 2. before.rules に MASQUERADE ルールを追加

```bash
sudo nano /etc/ufw/before.rules
```

ファイルの先頭（`*filter` より前）に以下を追記:

```
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]

# Internet 側のインターフェース名（例: eth0）に向けて MASQUERADE
-A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE

COMMIT
```

> `eth0` はインターネット側のNICに合わせて変更してください（`ip a` で確認）。

#### 3. UFW のデフォルトフォワードポリシーを変更

```bash
sudo nano /etc/default/ufw
```

```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

#### 4. UFWを再起動して設定を反映

```bash
sudo ufw disable
sudo ufw enable
```

または:

```bash
sudo ufw reload
```

---

## 参考

- `man ufw`
- `/etc/ufw/before.rules` — フィルタ・NAT ルールの手動記述
- `/etc/default/ufw` — デフォルトポリシーの設定
