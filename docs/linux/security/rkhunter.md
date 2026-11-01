---
title: rkhunter (Rootkit Hunter)
description: Linux / Unix 系 OS 向けのルートキット・バックドア・ローカルエクスプロイト検知ツール「rkhunter (Rootkit Hunter)」のインストール方法と使い方をまとめたドキュメントです。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

rkhunter (Rootkit Hunter)
===

## 概要

**rkhunter (Rootkit Hunter)** は、Linux / Unix 系 OS 向けのオープンソースのルートキット・バックドア・ローカルエクスプロイト検知およびシステム診断ツールです。  
シェルスクリプトベースで動作し、既知のルートキットのシグネチャ検査だけでなく、システムバイナリのハッシュ値（ファイル整合性）比較や不審な設定の監査を包括的に行います。

主な特徴と検査項目は以下の通りです。

- **ルートキット・マルウェア検査**: 既知のルートキット、トロイの木馬、バックドアのファイルやシグネチャを検出
- **システムコマンドの整合性チェック**: `/bin` や `/usr/bin` などの重要バイナリについて、SHA-256 等のハッシュ値・パーミッション・属性（UID/GID 等）を基準データベース（`rkhunter.dat`）およびパッケージマネージャー（`dpkg` / `rpm`）の情報と照合
- **不審なファイル・ディレクトリの検出**: `/dev` や `/tmp` 等に隠されたファイル（ドットファイル）や不審な共有ライブラリ設定（`LD_PRELOAD` 等）を検査
- **ネットワーク・プロセスの検査**: バックドアがよく使用する不審な待受ポート、プロミスキャスモードのネットワークインターフェース、削除済みファイルを実行中のプロセスなどを検出
- **OS・アプリケーション設定の診断**: `sshd_config` の `PermitRootLogin` 設定や、アカウント・パスワード関連の設定、スタートアップファイルを検査

---

## インストール

### Debian / Ubuntu

```bash title="apt でのインストール"
sudo apt update
sudo apt install -y rkhunter
```

:::note[Postfix などの MTA 設定画面が表示された場合]
Debian / Ubuntu では推奨パッケージとしてメール転送エージェント（Postfix 等）が同時にインストールされることがあります。メール通知を使用しない場合は「設定なし (No configuration)」または「ローカルのみ (Local only)」を選択するか、`--no-install-recommends` を付与してインストールしてください。
:::

### RHEL / AlmaLinux / Rocky Linux / Fedora

RHEL 系ディストリビューションでは EPEL リポジトリにパッケージが含まれています（Fedora は標準リポジトリからインストール可能）。

```bash title="dnf でのインストール"
# RHEL / AlmaLinux / Rocky Linux の場合は事前に EPEL を有効化
sudo dnf install -y epel-release

# rkhunter のインストール
sudo dnf install -y rkhunter
```

---

## ファイル構成

パッケージからインストールした場合の主要なファイルとディレクトリの構成は以下の通りです。

| パス | 説明 |
| --- | --- |
| `/etc/rkhunter.conf` | メイン設定ファイル（パッケージ更新時の競合を避けるため直接編集せず `.local` の利用を推奨） |
| `/etc/rkhunter.conf.local` | ユーザー独自のカスタマイズ設定ファイル（`/etc/rkhunter.conf` の設定を上書き） |
| `/etc/default/rkhunter` | Debian / Ubuntu における Cron 定期実行・更新動作の設定ファイル |
| `/etc/sysconfig/rkhunter` | RHEL / Fedora 系における Cron 定期実行・メール通知の設定ファイル |
| `/var/lib/rkhunter/db/` | ルートキット定義データおよびファイルプロパティデータベース（`rkhunter.dat`）の格納先 |
| `/var/log/rkhunter.log` | スキャン時の詳細な実行ログ（警告の原因調査に利用） |

---

## 環境構築

rkhunter を導入した直後は、誤検知（False Positive）を防ぎ、現在の正常なシステム状態をベースラインとして記録するための初期設定を行います。

### 1. Debian / Ubuntu での `WEB_CMD` 設定修正

Debian / Ubuntu のデフォルト設定では `/etc/rkhunter.conf` 内に `WEB_CMD="/bin/false"` が指定されている場合があり、`rkhunter --update` 実行時に `Invalid WEB_CMD configuration option` エラーが発生することがあります。

`/etc/rkhunter.conf.local` を作成（または編集）し、以下のように上書き設定します。

```ini title="/etc/rkhunter.conf.local"
# デフォルトの WEB_CMD="/bin/false" を無効化し、wget や curl の自動検出を有効化
WEB_CMD=""

# データベースのミラー設定をローカル・リモート両対応にする
MIRRORS_MODE=0
UPDATE_MIRRORS=1
```

### 2. カスタマイズ設定 (`rkhunter.conf.local`)

システムの構成に合わせて、パッケージマネージャー連携や誤検知のホワイトリスト登録を `/etc/rkhunter.conf.local` に記述します。

```ini title="/etc/rkhunter.conf.local"
# パッケージマネージャーによるファイルハッシュ検証を有効化
# Debian / Ubuntu の場合: DPKG
# RHEL / AlmaLinux / Rocky / Fedora の場合: RPM
PKGMGR=DPKG

# SSH の PermitRootLogin 設定を合わせることで警告を防止（sshd_config 側と同じ値を指定）
ALLOW_SSH_ROOT_USER=no

# シェルスクリプト化されている標準コマンドの誤検知を除外（環境に応じて追加）
SCRIPTWHITELIST=/usr/bin/egrep
SCRIPTWHITELIST=/usr/bin/fgrep
SCRIPTWHITELIST=/usr/bin/ldd
SCRIPTWHITELIST=/usr/bin/lwp-request
SCRIPTWHITELIST=/usr/sbin/adduser

# 正規の隠しディレクトリ・隠しファイルをホワイトリストに登録する例
# ALLOWHIDDENDIR=/etc/.java
# ALLOWHIDDENFILE=/etc/.etckeeper
```

:::tip[`PKGMGR` の活用]
`PKGMGR=DPKG`（または `RPM`）を設定しておくと、`apt upgrade` や `dnf update` でシステムバイナリが更新された際にもパッケージ管理データベースのハッシュ値を参照するため、パッケージ更新直後の誤検知（Warning）を大幅に減らすことができます。
:::

### 3. 設定ファイルの構文チェックと初期データベース作成

設定完了後、設定値のチェック、定義ファイルの更新、および現在のシステム状態のベースライン（ファイルプロパティデータベース `rkhunter.dat`）の作成を実行します。

```bash title="初期データベースの更新とベースライン作成"
# 設定ファイルの構文チェック
sudo rkhunter -C

# 定義データベースのアップデート
sudo rkhunter --update

# 現在のシステムバイナリの状態（ハッシュ値等）をベースラインとして記録
sudo rkhunter --propupd
```

:::caution[`--propupd` を実行するタイミング]
`rkhunter --propupd` は「現在のシステム上のファイルが改ざんされていない正常な状態である」ことを前提として基準データベースを作成します。OS インストール直後や、管理者自身が `apt` / `dnf` でパッケージを更新した直後に実行してください。
:::

---

## 各種操作

### データベースの更新

```bash title="定義ファイルとファイルプロパティの更新"
# ルートキット定義ファイルのオンライン更新チェック
sudo rkhunter --update

# システム全体のファイルプロパティデータベース（rkhunter.dat）を更新
sudo rkhunter --propupd

# 特定のファイルのみプロパティデータベースを更新（例: /usr/bin/curl 更新後）
sudo rkhunter --propupd /usr/bin/curl
```

### システムスキャンの実行

```bash title="システムスキャンの実行"
# 対話形式でシステム全体をスキャン（セクションごとに Enter キー待ちあり）
sudo rkhunter --check

# キー入力待ちを省略して最後まで連続スキャン
sudo rkhunter --check --sk

# 警告（Warning）のみを画面に表示して連続スキャン
sudo rkhunter --check --sk --rwo
```

### 主な実行オプション

| オプション | 短縮形 | 説明 |
| --- | --- | --- |
| `--check` | `-c` | システム全体のスキャンを実行 |
| `--skip-keypress` | `--sk` | 各テストセクション終了時の `[ Press <ENTER> to continue ]` をスキップして連続実行 |
| `--report-warnings-only` | `--rwo` | 通常のチェック結果を省略し、**Warning（警告）** のみを標準出力に表示 |
| `--cronjob` | - | Cron 定期実行向けオプション（`--check` `--sk` `--nocolors` 相当） |
| `--update` | - | ルートキット定義データベースをオンラインで更新 |
| `--propupd [file]` | - | ファイル整合性チェック用の基準データベース（`rkhunter.dat`）を作成・更新 |
| `--config-check` | `-C` | 設定ファイル（`rkhunter.conf` / `rkhunter.conf.local`）の構文と設定値を検証 |
| `--list [項目]` | - | 利用可能なテスト名（`tests`）、ルートキット一覧（`rootkits`）、言語（`languages`）などを表示 |
| `--enable <test>` | - | 指定したテストのみを実行（例: `--enable rootkits`） |
| `--disable <test>` | - | 指定したテストを除外して実行 |
| `--versioncheck` | - | rkhunter 本体の新しいバージョンが公開されているか確認 |

---

## 使用例

### 1. スキャン実行から警告（Warning）の原因調査・対処までの流れ

1. **システムスキャンの実行**

    ```bash title="連続スキャンの実行"
    sudo rkhunter --check --sk
    ```

    スキャンが完了すると、末尾に検査結果のサマリーが表示されます。

    <details>
    <summary>スキャン結果サマリーの出力例を表示</summary>

    ```text title="出力例"
    System checks summary
    =====================

    File properties checks...
        Files checked: 148
        Suspect files: 1

    Rootkit checks...
        Rootkits checked : 498
        Possible rootkits: 0

    Applications checks...
        All checks skipped

    The system checks took: 1 minute and 15 seconds

    All results have been written to the log file: /var/log/rkhunter.log

    One or more warnings have been found while checking the system.
    Please check the log file (/var/log/rkhunter.log)
    ```

    </details>

2. **ログファイルから Warning の詳細を確認**

    `Suspect files` や `Warning` が検出された場合は、`/var/log/rkhunter.log` から `Warning:` を抽出して原因を確認します。

    ```bash title="ログから警告箇所を抽出"
    sudo grep -n -C 3 "Warning:" /var/log/rkhunter.log
    ```

    ```text title="出力例（バイナリがスクリプトに置き換わっている警告と、パッケージ更新による変更の警告）"
    [20:10:15]   /usr/bin/egrep                                  [ Warning ]
    [20:10:15] Warning: The command '/usr/bin/egrep' has been replaced by a script: /usr/bin/egrep: POSIX shell script, ASCII text executable
    --
    [20:10:22]   /usr/bin/curl                                   [ Warning ]
    [20:10:22] Warning: The file properties have changed:
    [20:10:22]          File: /usr/bin/curl
    [20:10:22]          Current hash: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
    [20:10:22]          Stored hash : 8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4
    ```

3. **原因に応じた対処**

    - **パッケージアップデートによる正規のファイル変更の場合**:  
      `apt` や `dnf` の更新履歴（`/var/log/apt/history.log` や `dnf history`）および `dpkg -V` / `rpm -Vf` で正規の更新であることを確認した上で、プロパティデータベースを更新します。
      ```bash title="プロパティデータベースの更新"
      sudo rkhunter --propupd
      ```
    - **OS 標準でシェルスクリプト化されているコマンドや正規の隠しファイルの場合**:  
      `/etc/rkhunter.conf.local` に `SCRIPTWHITELIST` や `ALLOWHIDDENFILE` を追記し、再度 `sudo rkhunter --propupd` を実行します。

### 2. Cron による定期自動スキャンの設定

パッケージでインストールすると `/etc/cron.daily/rkhunter` 等が配置されます。OS ごとの設定ファイルで日次スキャンと自動更新を有効化できます。

- **Debian / Ubuntu の場合 (`/etc/default/rkhunter`)**

    ```ini title="/etc/default/rkhunter"
    # 日次での自動スキャンを有効化
    CRON_DAILY_RUN="true"

    # 週次でのデータベース自動更新を有効化
    CRON_DB_UPDATE="true"

    # apt 実行時に自動で --propupd を実行するかどうか（セキュリティ重視なら false のまま手動実行を推奨）
    APT_AUTOGEN="false"

    # 警告発生時の通知先メールアドレス（ローカルの root または管理者のアドレス）
    REPORT_EMAIL="root"
    ```

- **RHEL / AlmaLinux / Rocky Linux / Fedora の場合 (`/etc/sysconfig/rkhunter`)**

    ```ini title="/etc/sysconfig/rkhunter"
    MAILTO=root@localhost
    DIAG_SCAN=no
    ```

---

## 参考

- [Rootkit Hunter 公式サイト (SourceForge)](https://rkhunter.sourceforge.net/)
- [Debian Wiki - rkhunter](https://wiki.debian.org/rkhunter)
- [Ubuntu Manpage - rkhunter(8)](https://manpages.ubuntu.com/manpages/noble/en/man8/rkhunter.8.html)
