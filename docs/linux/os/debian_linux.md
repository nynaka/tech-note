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
