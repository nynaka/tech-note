---
title: Solaar
description: Linux 環境で Logitech（Logicool）製キーボードやマウスの設定（Fnキー反転、バッテリー確認等）を行うツール「Solaar」のインストール・設定手順
#sidebar_position: 2
---

Solaar のインストールと設定手順
===

## 概要

**Solaar** は、Logitech（Logicool）の Unifying レシーバー、Lightspeed レシーバー、Bolt レシーバー、および Bluetooth 接続されたデバイスを Linux 上で管理・設定するためのオープンソースツールです。  
マウスだけでなくキーボードの HID++ 機能にも対応しており、主に以下の用途で使用されます。

- **キーボードの Fn キー反転（`fn-swap`）**: F1〜F12 を標準のファンクションキーとしてデフォルト動作させる（メディアキー動作と反転）
- **バッテリー残量の確認**
- **Unifying / Bolt レシーバーへのデバイスペアリング管理**
- **マウスの詳細設定**（スムーズスクロール、DPI 調整、ジェスチャー等）

---

## インストール手順

### RHEL 系 Linux (RHEL / Fedora /AlmaLinux / RockyLinux 等)

EPEL リポジトリに `solaar` パッケージが提供されていないため、Python の **pip** を使用してインストールします。  
依存ライブラリ（evdev）のビルドに C コンパイラおよび Python 開発用ヘッダー（Python.h）が必要となるため、事前にインストールします。

```bash
# ビルドツールと依存パッケージのインストール
sudo dnf install -y python3-pip python3-gobject python3-devel gcc

# Solaar のインストール（ユーザー環境へ）
pip3 install --user solaar
```

:::tip[PATH の確認]
インストールされたコマンドは `~/.local/bin/solaar` に配置されます。  
`~/.local/bin` が環境変数 `PATH` に含まれていない場合は、`~/.bashrc` などに以下を追記してください。
```bash
export PATH="$HOME/.local/bin:$PATH"
```
:::

### Debian 系 Linux (Debian / Ubuntu 等)

公式リポジトリに `solaar` パッケージが提供されているため、apt でインストール可能です。

```bash
sudo apt update
sudo apt install -y solaar
```

---

## udev ルールの設定 (必須)

Solaar が一般ユーザー権限で USB レシーバーや HID デバイス（/dev/hidraw*）と通信できるようにするため、udev ルールを配置します。

```bash
# 公式の udev ルールファイルをダウンロード
sudo curl -sSL -o /etc/udev/rules.d/42-logitech-unify-permissions.rules \
    https://raw.githubusercontent.com/pwr-Solaar/Solaar/master/rules.d/42-logitech-unify-permissions.rules

# ルールの再読み込みと適用
sudo udevadm control --reload-rules
sudo udevadm trigger
```

:::note[デバイスの再接続]
udev ルール反映後、すでに接続されているレシーバーを一度抜き差しするか、Bluetooth デバイスを再接続してください。
:::

---

## デバイスの設定

### 1. 接続デバイス一覧と設定項目の確認

接続されているデバイス名や対応している設定項目を確認します。

```bash
solaar show
```

**出力例（K780 の場合）**
```text
  1: K780 Multi-Device Wireless Keyboard
     Codename     : K780
     Kind         : keyboard
     ...
     Features:
       ...
       Swap Fx function : [fn-swap]
```

### Fn キーの反転 (fn-swap)

K780 や K380 などのキーボードで、Fn キーを押さずに標準の F1〜F12 キーとして動作させる（Fn ロック）設定を行います。

```bash
# デバイス名を指定して fn-swap を無効化
solaar config "K780 Multi-Device Wireless Keyboard" fn-swap off
```

現在の設定値を確認するには以下を実行します。

```bash
solaar config "K780 Multi-Device Wireless Keyboard" fn-swap
```

:::tip[デバイス名の指定について]
`solaar show` で表示された正確なデバイス名、またはデバイス番号（例: `1`）を指定して設定することも可能です。
```bash
solaar config 1 fn-swap off
```
:::

---

## 各種操作・運用

### GUI モードの起動（デスクトップ環境）

デスクトップ環境（GNOME, KDE 等）がある場合は、GUI 画面からグラフィカルに設定の変更やバッテリー残量の確認ができます。

```bash
solaar
```

システムトレイに常駐させる場合は以下のオプションを使用します。

```bash
solaar --window=hide
```

---

## トラブルシューティング

### `Python.h: そのようなファイルやディレクトリはありません` で pip インストールが失敗する

`evdev` の C 拡張ビルド時に Python の開発ヘッダーが見つからないことが原因です。`python3-devel` をインストールしてから再試行してください。

```bash
sudo dnf install -y python3-devel gcc
```

### `solaar: error: [Errno 13] Permission denied` が発生する

udev ルールが正しく適用されていないか、デバイスのパーミッションが不足しています。[udev ルールの設定](#udev-ルールの設定-必須) を再度実行し、USB レシーバーの再挿入または Bluetooth の再接続を行ってください。

### OS 再起動の度に fn-swap 設定が戻る

キーボード等のデバイス設定（`fn-swap` 等）は、Solaar 実行時に `~/.config/solaar/config.yaml` へ自動的に保存されます。  
しかし、OS の再起動時やスリープ復帰時、キーボード電源の再投入時にはデバイス側の状態がリセットされるため、**Solaar をバックグラウンド常駐（自動起動）させておく**必要があります。Solaar が常駐していれば、デバイスが接続されたタイミングで保存済みの設定が自動的に再適用されます。

1. **事前に一度 `fn-swap` の設定を行っておきます。**（設定ファイルに自動保存されます）

    ```bash
    solaar config "K780 Multi-Device Wireless Keyboard" fn-swap off
    ```

2. **自動起動用のディレクトリを作成します。**

    ```bash
    mkdir -p ~/.config/autostart
    ```

3. **Solaar をバックグラウンド起動（ウィンドウ非表示・トレイ常駐）する自動起動エントリを作成します。**

    :::tip[コマンドパスの確認]
    pip (`--user`) でインストールした場合は `~/.local/bin/solaar`、apt/dnf でインストールした場合は `/usr/bin/solaar` を指定します。`$(which solaar)` で実行パスを自動展開して設定します。
    :::

    ```bash
    cat << EOF > ~/.config/autostart/solaar.desktop
    [Desktop Entry]
    Type=Application
    Name=Solaar
    Comment=Logitech Device Manager
    Exec=$(which solaar) --window=hide
    Icon=solaar
    Terminal=false
    StartupNotify=false
    Categories=Utility;GTK;
    EOF
    ```

4. **動作確認を行います。**

    PC を再起動するか、一旦ログアウト⇒再ログインしてください。  
    システムトレイに Solaar が常駐し、以下のコマンドで設定値が維持されていることを確認します。

    ```bash
    solaar config "K780 Multi-Device Wireless Keyboard" fn-swap
    ```

    値が `False`（または `off`）と表示されれば成功です。

---

## 参考

- [Solaar - GitHub リポジトリ](https://github.com/pwr-Solaar/Solaar)
- [Solaar 公式ドキュメント](https://pwr-solaar.github.io/Solaar/)
- [Solaar RHEL インストールガイド](https://github.com/pwr-Solaar/Solaar/blob/master/docs/RHEL.md)
