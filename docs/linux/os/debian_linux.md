---
title: Debian Linux
description: Debian Linux の設定メモです。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

Debian Linux
===

## 設定

### Wifi

```bash title="ツールのインストール"
sudo apt install -y wpasupplicant
```

```bash title="インタフェース名の確認"
ip link show
```

```bash title="SSID とパスワードを使用して設定ファイルの生成"
wpa_passphrase "SSID名" "パスワード" \
    | sudo tee /etc/wpa_supplicant/wpa_supplicant.conf
```

`/etc/network/interfaces` にインタフェースの設定を追記します。  
`wlo1` は無線 LAN インタフェース名です。環境に合わせて変更してください。

```text title="/etc/network/interfaces"
auto wlo1
iface wlo1 inet dhcp
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```

設定を反映します。

```bash title="インタフェースの再起動"
sudo ifdown wlo1 && sudo ifup wlo1
```

### sudo

sudo の設定は、直接設定ファイルを編集する手もありますが、通常 `visudo` コマンドを使用します。

1. 設定の編集

    ```bash
    sudo visudo
    ```

2. 設定をの編集

    下記の書式で設定を追加します。  
    下記の例では ubuntu アカウントは、パスワードなしで sudo コマンドを実行できるようにします。  
    セキュリティ的によろしくないので非推奨設定です。

    ```
    ubuntu ALL=(ALL) NOPASSWD: ALL
    ```

    **各項目の意味**
    | 項目          | 意味                                       |
    | :------------ | ------------------------------------------ |
    | ubuntu        | 対象のユーザー名                           |
    | ALL=          | すべてのホストで有効                       |
    | (ALL)         | すべてのユーザー（root含む）として実行可能 |
    | NOPASSWD: ALL | すべてのコマンドをパスワードなしで実行可能 |

3. 保存
    - nano の場合（標準）: Ctrl + O を押したあと Enter で保存し、Ctrl + X で閉じます。
    - vi / vim の場合: :wq と入力して Enter を押します。
