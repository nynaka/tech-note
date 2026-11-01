---
title: Lynis
description: Linux / Unix 系 OS 向けのオープンソースセキュリティ監査・ハードニング支援ツール「Lynis」のインストール方法と基本的な使い方をまとめたドキュメントです。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

Lynis
===

## 概要

**Lynis** は、CISOfy 社によって開発されている Linux / macOS / Unix 系 OS 向けのオープンソースのセキュリティ監査およびハードニング（要塞化）支援ツールです。  
エージェントレスで動作し、システムの設定ミスや脆弱性、セキュリティ上の改善点をスキャンしてレポートを生成します。

主な特徴と用途は以下の通りです。

- **システムセキュリティ監査**: カーネル、認証・アカウント設定、ネットワーク、ファイアウォール、SSH、Web サーバー、ファイルパーミッションなど数百項目を自動チェック
- **ハードニング指標の算出**: システムの堅牢性を **Hardening Index**（0〜100 のスコア）として数値化
- **改善提案の提示**: 重大な問題（**Warning**）と改善の推奨事項（**Suggestion**）をテスト ID 付きで提示し、具体的な対策方法を確認可能
- **コンプライアンス確認支援**: ISO27001、PCI-DSS、HIPAA などのセキュリティ基準に準じたチェック

---

## インストール

Lynis は各ディストリビューションの標準パッケージマネージャー、または GitHub リポジトリ（最新版をすぐ試したい場合）からインストールできます。

:::tip[公式リポジトリから最新版を導入する場合]
OS 標準リポジトリのパッケージはバージョンが古い場合があります。最新版の監査項目を利用したい場合は、CISOfy 公式 APT リポジトリまたは後述の Git クローン方式を使用してください。
:::

### Debian / Ubuntu

```bash
sudo apt update
sudo apt install -y lynis
```

### RHEL / AlmaLinux / Rocky Linux / Fedora

RHEL 系ディストリビューションでは EPEL リポジトリにパッケージが含まれています（Fedora は標準リポジトリからインストール可能）。

```bash
# RHEL / AlmaLinux / Rocky Linux の場合は事前に EPEL を有効化
sudo dnf install -y epel-release

# Lynis のインストール
sudo dnf install -y lynis
```

### Git リポジトリからの実行（インストール不要・最新版）

システムにパッケージをインストールせず、最新のソースコードを取得してそのまま実行することも可能です。

```bash title="Git クローンと実行"
git clone https://github.com/CISOfy/lynis.git
cd lynis

# ファイル所有者が root でない旨の警告を回避する場合
sudo chown -R 0:0 .

# 監査の実行
sudo ./lynis audit system
```

---

## ファイル構成

パッケージからインストールした場合の主要なファイルとディレクトリの構成は以下の通りです。

| パス | 説明 |
| --- | --- |
| `/etc/lynis/default.prf` | デフォルトの設定プロファイル（直接編集せず `custom.prf` で上書き推奨） |
| `/etc/lynis/custom.prf` | ユーザー独自のカスタマイズ設定プロファイル |
| `/usr/share/lynis/` | 監査スクリプト（`include/`）、データベース（`db/`）、プラグイン（`plugins/`）の配置先 |
| `/var/log/lynis.log` | スキャン時の詳細な実行ログ（各チェック項目の判定理由やデバッグ情報） |
| `/var/log/lynis-report.dat` | スキャン結果の構造化レポートデータ（`key=value` 形式） |

---

## 環境構築

デフォルトのままでも問題なくシステム監査を実行できますが、環境に合わせて一部のテストをスキップしたい場合などはカスタムプロファイルを作成します。

### カスタムプロファイルの作成 (`custom.prf`)

`/etc/lynis/default.prf` はパッケージ更新時に上書きされる可能性があるため、設定変更は `/etc/lynis/custom.prf` に記述します。

```ini
# 特定のテスト ID をスキップする場合（例: SSH-7408 や FILE-6310 を除外）
skip-test=SSH-7408
skip-test=FILE-6310

# 特定のテストのサブ項目のみスキップする場合
# skip-test=SSH-7408:permitrootlogin

# スキャン中に一時停止せず連続実行する（--quick オプション相当）
quick=yes
```

---

## 各種操作

:::note[実行権限について]
一般ユーザーでも実行可能ですが、すべての設定ファイルやカーネルパラメータ、プロセス情報を正確に監査するためには **`root` 権限（`sudo`）での実行** が推奨されます。
:::

### 基本情報の確認

```bash
# バージョン確認
sudo lynis show version

# 最新バージョンが公開されているかチェック
sudo lynis update info

# 利用可能なコマンド一覧の表示
sudo lynis show commands
```

### システム全体のセキュリティ監査

もっとも基本的なローカルシステム全体の監査コマンドです。

```bash
sudo lynis audit system
```

### 主な実行オプション

用途に応じた主なオプションは以下の通りです。

| オプション | 説明 |
| --- | --- |
| `--quick` / `-Q` | セクションごとのキー入力待ちを省略して最後まで連続実行（デフォルトで連続実行される環境もあり） |
| `--quiet` / `-q` | 通常のチェック結果出力を省略し、**Warning（警告）** や **Suggestion（提案）** のみを表示 |
| `--pentest` | 非特権ユーザー想定のペネトレーションテストモードで監査を実行 |
| `--forensics` | 稼働中またはマウント済みのシステムに対してフォレンジック調査用のチェックを実行 |
| `--tests "TEST-ID..."` | 指定したテスト ID のみをピンポイントで実行（設定変更後の再確認に便利） |
| `--tests-from-group "GROUP"` | 特定のカテゴリ（例: `ssh`, `firewall`, `kernel`, `authentication`）のみを実行 |
| `--cronjob` | 定期実行向けモード（制御文字や色付けを無効化して出力） |
| `--auditor "名前"` | レポートに監査実施者名を記録 |

- 特定カテゴリ・特定テストのみを実行する例

    ```bash
    # 利用可能なテストグループ（カテゴリ）の一覧を表示
    sudo lynis show groups

    # SSH とファイアウォールに関する監査のみを実行
    sudo lynis audit system --tests-from-group "ssh firewall"

    # 特定のテスト ID（例: SSH-7408）のみを再テスト
    sudo lynis audit system --tests SSH-7408
    ```

- Dockerfile の監査

    コンテナイメージ作成用の `Dockerfile` に対してベストプラクティスやセキュリティリスクの静的解析を行うこともできます。

    ```bash title="Dockerfile の監査"
    sudo lynis audit dockerfile /path/to/Dockerfile
    ```

---

## 使用例

### 1. スキャン実行から対策・再確認までの流れ

1. **システム監査の実行**

    ```bash
    sudo lynis audit system --quick
    ```

    スキャンが完了すると、末尾に以下のようなサマリーと **Warning（警告）**、**Suggestion（改善提案）** の一覧、および **Hardening index（ハードニングスコア）** が表示されます。

    <details>
    <summary>スキャン結果サマリーの出力例を表示</summary>

    ```text
    ================================================================================

      -[ Lynis 3.1.1 Results ]-

      Warnings (1):
      ----------------------------
      ! Found one or more vulnerable packages. [PKGS-7392]
          https://cisofy.com/lynis/controls/PKGS-7392/

      Suggestions (28):
      ----------------------------
      * This release is more than 4 months old. Check the website or GitHub to see if there is an update available. [LYNIS]
          https://cisofy.com/lynis/controls/LYNIS/

      * Install libpam-tmpdir to set $TMP and $TMPDIR for PAM sessions [DEB-0280]
          https://cisofy.com/lynis/controls/DEB-0280/

      * Consider hardening SSH configuration [SSH-7408]
        - Details  : AllowTcpForwarding (set YES to NO)
          https://cisofy.com/lynis/controls/SSH-7408/
        - Details  : ClientAliveCountMax (set 3 to 2)
          https://cisofy.com/lynis/controls/SSH-7408/

    ================================================================================

      Lynis security scan details:

      Hardening index : 68 [#############       ]
      Tests performed : 256
      Plugins enabled : 1

      Files:
      - Test and debug information      : /var/log/lynis.log
      - Report data                     : /var/log/lynis-report.dat
    ================================================================================
    ```

    </details>

2. **指摘内容（テスト ID）の詳細確認**

    レポート内の `[SSH-7408]` のようなテスト ID を指定して `show details` を実行すると、`/var/log/lynis.log` から該当箇所の詳細な判定ログを抽出して確認できます。

    ```bash
    sudo lynis show details SSH-7408
    ```

    ```text title="出力例"
    2026-10-31 20:00:00 Performing test ID SSH-7408 (Check SSH daemon configuration)
    2026-10-31 20:00:00 Test: Checking specific defined options in /tmp/lynis.XXXXXX
    2026-10-31 20:00:00 Result: Option AllowTcpForwarding found
    2026-10-31 20:00:00 Result: Option AllowTcpForwarding value is YES
    2026-10-31 20:00:00 Suggestion: Consider hardening SSH configuration [test:SSH-7408] [details:AllowTcpForwarding (set YES to NO)] [solution:text:Edit /etc/ssh/sshd_config]
    ```

3. **設定の修正と単体再テスト**

    対象の設定（例: `/etc/ssh/sshd_config` の変更と `sshd` の再読み込み）を実施した後、`--tests` オプションで該当のテスト ID のみを素早く再検証できます。

    ```bash
    sudo lynis audit system --tests SSH-7408
    ```

### 2. レポートファイルからの情報抽出

`/var/log/lynis-report.dat` には監査結果がテキスト形式で保存されているため、`grep` 等で警告や提案だけを後から一覧抽出できます。

```bash
# Warning（警告）の一覧を抽出
sudo grep "^warning\[\]=" /var/log/lynis-report.dat

# Suggestion（改善提案）の一覧を抽出
sudo grep "^suggestion\[\]=" /var/log/lynis-report.dat

# Hardening Index（スコア）の確認
sudo grep "^hardening_index=" /var/log/lynis-report.dat
```

### 3. Cron / systemd タイマーによる定期監査

`--cronjob` オプションを使用して定期スキャンを実行し、ログやレポートを定点観測する例です。

- /etc/cron.weekly/lynis-audit

    ```bash
    #!/bin/sh
    set -eu

    AUDITOR="automated-cron"
    DATE=$(date +%Y%m%d)
    REPORT_DIR="/var/log/lynis/archive"

    mkdir -p "${REPORT_DIR}"

    # 監査の実行
    /usr/sbin/lynis audit system --cronjob --auditor "${AUDITOR}" > "${REPORT_DIR}/lynis-${DATE}.txt"

    # レポートデータの退避
    cp /var/log/lynis-report.dat "${REPORT_DIR}/lynis-report-${DATE}.dat"
    ```

- 実行権限の付与

    ```bash
    sudo chmod 750 /etc/cron.weekly/lynis-audit
    ```

---

## 参考

- [Lynis 公式サイト (CISOfy)](https://cisofy.com/lynis/)
- [Lynis - GitHub リポジトリ](https://github.com/CISOfy/lynis)
- [Lynis 公式ドキュメント](https://cisofy.com/documentation/lynis/)
- [Lynis Controls (テスト ID 一覧と解説)](https://cisofy.com/lynis/controls/)
