---
title: FreeRDP (xfreerdp)
description: Linux 環境における FreeRDP (xfreerdp) のインストール手順、日本語キーボードや高画質設定に対応した接続スクリプト、トラブルシューティングに関するメモです。
---

xfreerdp
===

FreeRDP (xfreerdp) の各種 Linux ディストリビューション向けインストール手順と、接続スクリプトの解説です。

## xfreerdp のインストール方法

- RHEL / AlmaLinux / Rocky Linux / Fedora

    ```bash
    sudo dnf install -y freerdp
    ```

- Debian / Ubuntu

    ディストリビューションのバージョンによって提供されているパッケージが異なります。
    - **Debian 13 (Trixie) / Ubuntu 24.04 LTS 以降 (FreeRDP 3.x)**:
      ```bash
      sudo apt install -y freerdp3-x11
      ```
      ※ コマンド名は `xfreerdp3` となります。
    - **Debian 12 (Bookworm) / Ubuntu 22.04 LTS 以前 (FreeRDP 2.x)**:
      ```bash
      sudo apt install -y freerdp2-x11
      ```
      ※ コマンド名は `xfreerdp` となります。

### 動作確認

インストールした環境に合わせてバージョン確認コマンドを実行します。

```bash
# FreeRDP 2.x、または RHEL / Fedora 等
xfreerdp --version

# FreeRDP 3.x (Debian / Ubuntu の freerdp3-x11 等)
xfreerdp3 --version
```

---

## 接続スクリプト例

接続先やユーザー名を柔軟に指定できるシェルスクリプトの例です。  
FreeRDP 2.x と 3.x の両方に対応し、コマンド名（`xfreerdp` / `xfreerdp3`）やオプションの書式差分を自動判別します。

**connect-rdp.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail

###############################################################################
# 設定項目（環境に合わせて変更してください）
###############################################################################
RDP_HOST="${1:-"192.168.1.100"}"        # 接続先ホスト名またはIPアドレス
RDP_USER="${2:-"Administrator"}"        # 接続ユーザー名
RDP_DOMAIN="${RDP_DOMAIN:-""}"          # Active Directory ドメイン名 (任意)
RDP_PORT="${RDP_PORT:-3389}"            # RDP ポート番号
INIT_WIDTH="1600"                       # 起動時のウィンドウ幅
INIT_HEIGHT="900"                       # 起動時のウィンドウ高さ

###############################################################################
# FreeRDP コマンドおよびバージョンの自動判別 (xfreerdp / xfreerdp3, v2 / v3)
###############################################################################
if command -v xfreerdp3 &>/dev/null; then
    RDP_BIN="xfreerdp3"
elif command -v xfreerdp &>/dev/null; then
    RDP_BIN="xfreerdp"
else
    echo "エラー: xfreerdp または xfreerdp3 コマンドが見つかりません。FreeRDP をインストールしてください。" >&2
    exit 1
fi

FREERDP_VER=$("${RDP_BIN}" --version 2>&1 | grep -oE '[0-9]+' | head -n 1 || echo "3")
if [[ "${FREERDP_VER}" -ge 3 ]]; then
    KBD_OPT="/kbd:layout:0x00000411"    # FreeRDP 3.x 形式
    CERT_OPT="/cert:ignore"             # FreeRDP 3.x 形式
else
    KBD_OPT="/kbd:0x00000411"           # FreeRDP 2.x 形式
    CERT_OPT="/cert-ignore"             # FreeRDP 2.x 形式
fi

###############################################################################
# オプションの組み立て
###############################################################################
OPTS=(
    # 接続先設定
    "/v:${RDP_HOST}:${RDP_PORT}"
    "/u:${RDP_USER}"

    # 1. 通信品質最高 (LAN プロファイル & グラフィック強化)
    "/network:lan"           # LAN向け高品質設定
    "/bpp:32"                # 32bit カラー
    "+fonts"                 # ClearType フォントスムージング
    "+aero"                  # デスクトップコンポジション (Aero)
    "+window-drag"           # ドラッグ中のウィンドウ内容表示
    "+menu-anims"            # メニューアニメーション
    "/gfx"                   # GFXパイプライン (自動最高画質ネゴシエーション)
    "/sound:sys:pulse"       # 音声リダイレクト (PulseAudio / PipeWire)

    # 2. 日本語キーボード
    "${KBD_OPT}"

    # 3. クリップボード共有
    "+clipboard"             # 双方向クリップボードの共有

    # 4. 画面サイズ追随 (動的解像度変更)
    "/dynamic-resolution"    # ウィンドウリサイズ時にリモート解像度を自動追随
    "/size:${INIT_WIDTH}x${INIT_HEIGHT}"  # 起動時の初期ウィンドウサイズ

    # その他おすすめ設定
    "${CERT_OPT}"            # 自己署名証明書の警告を無視して接続
)

# ドメイン指定がある場合
if [[ -n "${RDP_DOMAIN}" ]]; then
    OPTS+=("/d:${RDP_DOMAIN}")
fi

###############################################################################
# 実行
###############################################################################
echo "=================================================="
echo "Connecting to: ${RDP_HOST}:${RDP_PORT} as [${RDP_USER}]"
echo "=================================================="
echo "※ パスワード入力を求められたら入力してください。"

exec "${RDP_BIN}" "${OPTS[@]}"
```

### スクリプトの使い方

- 実行権限の付与

    ```bash
    chmod +x connect-rdp.sh
    ```

- 接続の実行

    ```bash
    # デフォルト設定で接続
    ./connect-rdp.sh

    # 接続先ホストとユーザー名を引数で指定
    ./connect-rdp.sh 192.168.1.200 admin

    # ドメインを指定して接続
    RDP_DOMAIN="MYDOMAIN" ./connect-rdp.sh 192.168.1.200 admin
    ```

---

## 起動コマンド例

コマンドラインから直接実行する場合は、以下のように実行します。

```bash
xfreerdp \
    /v:192.168.1.100 \
    /u:Administrator \
    /network:lan \
    /bpp:32 \
    +fonts \
    +aero \
    +window-drag \
    +menu-anims \
    /gfx \
    /sound:sys:pulse \
    /kbd:layout:0x00000411 \
    +clipboard \
    /dynamic-resolution \
    /size:1600x900 \
    /cert:ignore
```

:::note[バージョンによるコマンド名・オプションの違い]
- **Debian / Ubuntu の FreeRDP 3.x (`freerdp3-x11`)**:
  - 実行コマンド名は `xfreerdp3` となります。
- **FreeRDP 2.x の場合**:
  - キーボード指定: `/kbd:layout:0x00000411` → `/kbd:0x00000411`
  - 証明書警告無視: `/cert:ignore` → `/cert-ignore`
:::

---

## トラブルシューティング・Tips

1. 画面が重い / ネットワーク帯域を節約したい場合

    `/network:lan` ⇒ `/network:auto` に変更すると、回線速度に応じて自動調整されます。

2. Windows に接続するとキーボードが強制的に英語配列（101/104キー）になってしまう

    Linux クライアントから RDP 接続すると、クライアント側で日本語キーボードを指定していても、Windows がクライアントのキーボードレイアウト情報を正しく解釈できず、**強制的に英語キーボード（US 101/104配列）にフォールバックしてしまう**現象が発生することがあります。  
    これを防ぐには、**接続先 Windows 側で「クライアント側のキーボード判定を無視し、Windows 本体の日本語レイアウト設定を強制する」レジストリ**を設定します。

    **設定するレジストリキー**

    | 項目   | 内容                                                                |
    | :----- | :------------------------------------------------------------------ |
    | Key    | HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Keyboard Layout |
    | 名前   | IgnoreRemoteKeyboardLayout                                          |
    | 種類   | REG_DWORD (32ビット)                                                |
    | 設定値 | 1                                                                   |

    - PowerShell で設定する場合（管理者権限）

        ```powershell
        New-ItemProperty `
            -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Keyboard Layout" `
            -Name "IgnoreRemoteKeyboardLayout" `
            -PropertyType DWord `
            -Value 1 -Force
        ```

    - コマンドプロンプトの場合（管理者権限）

        ```cmd
        reg add `
            "HKLM\SYSTEM\CurrentControlSet\Control\Keyboard Layout" `
            /v IgnoreRemoteKeyboardLayout /t REG_DWORD /d 1 /f
        ```

    :::info
    レジストリ変更後、Windows を再起動（または一度サインアウト）することで設定が反映されます。
    :::

    :::note[それでも英語配列のままになる場合の追加確認]
    Windows 本体のハードウェアキーボード認識が日本語（106/109キー）になっているかも確認します：
    - キー: `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\i8042prt\Parameters`
        - LayerDriver JPN: `kbd106.dll` (※英語配列だと `kbd101.dll` になっている)
        - OverrideKeyboardIdentifier: `PCAT_106KEY`
        - OverrideKeyboardType: `7` (DWORD)
        - OverrideKeyboardSubtype: `2` (DWORD)
    :::

3. キーボードの「半角/全角」キーが効かない場合
    - 接続先 Windows 側のタスクバー右下の入力言語が「日本語 (IME)」になっていることを確認してください。
    - FreeRDP 側のオプションとして、日本語キーボードタイプの指定を追加してみてください：
        - FreeRDP 3.x: `/kbd:type:7`
        - FreeRDP 2.x: `/kbd-type:7`

4. Wayland 環境でクリップボード共有がうまく動作しない場合

    XWayland 経由での実行、または環境変数 `GDK_BACKEND=x11` を指定して起動してみてください。  
    ※ FreeRDP 3 では Wayland ネイティブクライアント（`wlfreerdp` / `freerdp3-wayland` パッケージ）も利用可能です。
