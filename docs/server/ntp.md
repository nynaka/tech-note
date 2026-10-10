---
title: NTP サーバの設定
description: Linux 環境における NTP (ntpd) サーバのインストール、時刻同期、および設定手順についての解説です。
#sidebar_position: 2
---

NTP サーバの設定
===

## インストール

- **Debian / Ubuntu**

    ```bash
    sudo apt-get install ntp ntpdate
    ```

    :::note
    Debian 12 (Bookworm) や Ubuntu 22.04 以降では、従来の `ntp` パッケージに代わり `ntpsec` や `chrony` の利用が推奨・標準となっています。
    :::

- **CentOS / RHEL**

    ```bash
    sudo yum install ntp ntpdate
    ```

    :::note
    CentOS 8 / RHEL 8 以降では `chrony` が標準の NTP 実装となっています。
    :::

## 設定

### 事前の時刻同期

NTP では同期元 NTP サーバと大幅に時間がズレている場合（デフォルトで 1000 秒以上）、急激な時刻修正によるシステムへの影響を避けるため同期が行われません。そのため、NTP サーバの運用を始める前にシステム時刻を手動で合わせておきます。

```bash
# インストール後に ntpd が起動している場合は一時停止（Debian系: ntp, RHEL系: ntpd）
sudo systemctl stop ntp

# 上位 NTP サーバと手動で時刻同期
sudo ntpdate ntp.nict.jp

# システムクロックをハードウェアクロック (BIOS/RTC) に反映
# ※ Raspberry Pi など RTC モジュールを持たない環境ではエラーになります
sudo hwclock -w
```

:::tip
現在 `ntpdate` は非推奨となっているディストリビューションが多くあります。その場合は `sudo ntpd -gq` を実行することでも同様に一度だけ即時同期を行えます。
:::

### NTP サーバの設定 (/etc/ntp.conf)

Debian / CentOS ともに `/etc/ntp.conf` が主な設定ファイルです。主な設定項目は「参照先 NTP サーバの指定」と「アクセス制御（restrict）」です。

#### 1. 参照先 NTP サーバの指定

```conf
server ntp.nict.jp iburst         # NICT (日本標準時)
server ntp.jst.mfeed.ad.jp iburst # MFEED (インターネットマルチフィード)
```

上記のような書式で `/etc/ntp.conf` に追加します。デフォルトで記述されている NTP サーバ（ディストリビューション提供の pool サーバなど）を使用したくない場合は、各行の先頭に `#` を付けてコメントアウトします。

#### 2. 上位 NTP サーバへのアクセス許可

デフォルトで厳しいアクセス制限（`restrict default ignore` など）を設定している場合、`server` を指定しただけでは上位サーバからの応答パケットも拒否されてしまいます。そのため、指定した上位サーバへのアクセスを許可する設定が必要です。

**方法 A: `restrict source` を使用する（推奨）**

NTP 4.2.7 以降では、`restrict source` を記述することで、`server` や `pool` 行で指定されたホストからの通信を動的に許可できます。

```conf
restrict source nomodify notrap noquery
```

**方法 B: 個別の IP アドレスを指定する（従来の方法）**

NICT や MFEED は DNS ラウンドロビンで複数の IP アドレスが返されるため、対象となるサーバの IP アドレスを個別に許可します。

```conf
# NICT (133.243.238.*)
restrict 133.243.238.163 mask 255.255.255.255 nomodify notrap noquery
restrict 133.243.238.164 mask 255.255.255.255 nomodify notrap noquery
restrict 133.243.238.243 mask 255.255.255.255 nomodify notrap noquery
restrict 133.243.238.244 mask 255.255.255.255 nomodify notrap noquery

# MFEED (210.173.160.*)
restrict 210.173.160.27  mask 255.255.255.255 nomodify notrap noquery
restrict 210.173.160.57  mask 255.255.255.255 nomodify notrap noquery
restrict 210.173.160.87  mask 255.255.255.255 nomodify notrap noquery
```

#### 3. LAN からのアクセス制御

```conf
# デフォルトですべての NTP アクセスを拒否
restrict default ignore
restrict -6 default ignore

# ローカルホスト (自身) からの NTP アクセスは許可
restrict 127.0.0.1
restrict ::1

# LAN 内のクライアントからの時刻問い合わせを許可 (設定変更等は拒否)
restrict 192.168.230.0 mask 255.255.255.0 nomodify notrap
```

NTP クライアントによって任意に時計をズラされると認証関係の有効期限が無効になったり、ログの時系列が追えなくなったりするため、必要以上の権限は制限（`nomodify notrap`）しておきます。

### サービスの起動・有効化

設定が完了したら、NTP サービスを起動し自動起動を有効化します。

```bash
# Debian / Ubuntu の場合
sudo systemctl restart ntp
sudo systemctl enable ntp

# CentOS / RHEL の場合
sudo systemctl restart ntpd
sudo systemctl enable ntpd
```

### 同期の確認

`ntpq -p` コマンドで同期状態を確認できます。

```bash
$ ntpq -p
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*ntp-a2.nict.go. .NICT.           1 u   32   64  377   77.679    8.541  18.654
+ntp2.jst.mfeed. 210.173.176.4    2 u   38   64  377   95.518   16.027   2.467
 LOCAL(0)        .LOCL.          10 l  174   64  374    0.000    0.000   0.000
```

- **先頭記号の意味**:
  - `*`: 現在同期中のプライマリ NTP サーバ
  - `+`: 同期候補（バックアップ）の NTP サーバ
  - ` `（空白）や `-`: 参照対象外または破棄されたサーバ
- `ntpd` 起動から同期が完了してマークが付くまで、通常数分〜10分程度かかります。

## その他

### DHCP サーバで NTP サーバを広告する

ISC DHCP サーバ (`dhcpd.conf`) を使用している場合、下記のような設定を追加することで DHCP クライアントに NTP サーバの IP アドレスを通知できます。

```conf
option ntp-servers 192.168.230.1;
```

### 通信ポートとファイアウォール設定

NTP は **UDP ポート 123** を使用します（`/etc/services` には TCP 123 の記述もありますが、通常の NTP 通信は UDP のみです）。

NTP サーバとして外部や LAN からの同期リクエストを受け付ける場合、ファイアウォール（iptables 等）で通信を許可します。

```bash
# NTP サーバとしてクライアントからの問い合わせを受け付ける場合 (INPUT)
sudo iptables -A INPUT -p udp --dport 123 -j ACCEPT

# 自身が上位 NTP サーバへ同期するための送信 (OUTPUT)
sudo iptables -A OUTPUT -p udp --dport 123 -j ACCEPT
```

:::note
すでにステートフルパケットフィルタ（`ESTABLISHED,RELATED` の許可）が設定されている環境であれば、基本的には要求を受ける側のルール（NTP サーバなら `INPUT --dport 123`）のみを追加すれば応答パケットは自動的に許可されます。
:::
