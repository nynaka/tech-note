---
title: Amazon Linux 2023
description: Amazon Linux 2023 を VMware にインストール・セットアップする手順です。
---

Amazon Linux 2023
===

## VMware へのインストール

:::note
Amazon Linux 2023 (AL2023) は、Amazon EC2 以外の環境（オンプレミスや VMware などの仮想化基盤）向けに公式の仮想マシンイメージ（OVA）を提供しています。

ただし、オンプレミス環境には EC2 のインスタンスメタデータサービス（IMDS）が存在しないため、初回起動時のユーザー情報（パスワードや SSH 鍵など）を設定するために **`cloud-init` 用の初期設定ディスク（`seed.iso`）** を作成して接続する必要があります。
:::

---

### 1. OS イメージの取得

AWS 公式の CDN から VMware 用の OVA イメージをダウンロードします。

- ダウンロード先: [Amazon Linux 2023 VMware Images](https://cdn.amazonlinux.com/al2023/os-images/latest/vmware/)

ブラウザまたはコマンドライン（`wget` / `curl`）から最新の `.ova` ファイルを取得します。

```bash
# 最新版のダウンロード例 (ファイル名は最新バージョンに合わせて変更してください)
wget https://cdn.amazonlinux.com/al2023/os-images/2023.12.20260909.0/vmware/al2023-vmware_esx-2023.12.20260909.0-kernel-6.1-x86_64.xfs.gpt.ova
```

### 2. 初期設定用ディスク (seed.iso) の作成

`cloud-init` の NoCloud データソース機能を利用し、初期設定（ホスト名、初期ユーザー名 `ec2-user` のパスワード、SSH 鍵など）を行うための ISO ファイルを作成します。

#### 設定ファイルの準備

作業用のディレクトリを作成し、その中に `meta-data` と `user-data` の 2 つのファイルを作成します。

1. 作業ディレクトリ作成

    ```bash
    mkdir seedconfig
    cd seedconfig
    ```

1. **`meta-data` の作成**

    ホスト名などを定義します。

    ```yaml
    echo "local-hostname: al2023-vm" > meta-data
    ```

2. **`user-data` の作成**

    初期ユーザー（`ec2-user`）のパスワード設定や SSH 接続設定を記述します。

    ```bash
    cat << EOF > user-data
    #cloud-config
    #vim:syntax=yaml
    users:
      - name: ec2-user
        sudo: ALL=(ALL) NOPASSWD:ALL
        lock_passwd: false
        plain_text_passwd: 'password'
        ssh_authorized_keys:
          - ssh-ed25519 AAAA***********************************
    
    ssh_pwauth: true
    
    # package_update: true
    # packages:
    #   - open-vm-tools
    EOF
    ```

#### seed.iso の生成

作成した設定ファイルから ISO ファイルを作成します。  
**ボリュームラベルは必ず `cidata` に設定する必要があります。**（`cloud-init` がこのラベルを検出して読み込みます）

- **Linux の場合 (`genisoimage` / `mkisofs`)**

    ```bash
    ## ツールのインストール
    # Debian/Ubuntu の場合
    sudo apt install -y genisoimage
    # RHEL/AlmaLinux の場合
    sudo dnf install -y genisoimage

    ## seed.iso の生成
    genisoimage -output seed.iso \
        -volid cidata -joliet \
        -rock user-data meta-data
    ```

- **macOS の場合 (`hdiutil`)**

    ```bash
    cp user-data meta-data seedconfig/
    hdiutil makehybrid -o seed.iso \
        -hfs -joliet -iso \
        -default-volume-name cidata seedconfig/
    ```

- **Windows の場合**

    WSL (Windows Subsystem for Linux) を使用して Linux 手順を実行するか、ISO 作成ツールを使用してボリュームラベルを `cidata` に設定した ISO を生成してください。

---

### 3. VMware へのインポートと仮想マシンの構成

1. **OVA のインポート**
    - VMware Workstation / Player を起動し、**「ファイル」 > 「開く」** をクリックします。
    - ダウンロードした `.ova` ファイルを選択します。
    - 仮想マシン名（例: `Amazon Linux 2023`）と格納先パスを指定し、**「インポート」** を実行します。
    - ※「OVF 仕様に準拠していない」旨の警告が表示された場合は、**「再試行」** をクリックして続行します。

    :::note
    インポートできない場合は下記のコマンドを実行してみてください。

    ```bash
    ovftool --lax ./al2023-vmware_esx-2023.12.20260909.0-kernel-6.1-x86_64.xfs.gpt.ova $HOME/vmware/al2023/al2023.vmx
    ```

    AlmaLinux 10 の場合、下記のライブラリがインストールされていないことがあるようです。

    ```bash
    sudo dnf install -y libnsl libxcrypt-compat
    ```
    :::

2. **seed.iso のマウント**
    - インポートした仮想マシンの **「仮想マシン設定の編集」** を開きます。
    - **「CD/DVD」** ドライブを選択します。
    - 「接続」で **「ISO イメージファイルを使用する」** を選び、手順 2 で作成した **`seed.iso`** を指定します。
    - **「起動時に接続」** に必ずチェックを入れます。

3. **ハードウェア構成の調整**
    - メモリ、CPU コア数、ネットワークアダプタ（NAT または ブリッジ）を用途に合わせて調整します。
    - 「OK」をクリックして設定を保存します。

---

### 4. 初回起動とログイン

1. **仮想マシンの起動**
    - 仮想マシンをパワーオンします。
    - 起動時に `cloud-init` が `seed.iso` 内の設定を読み込み、ユーザーやネットワークの初期化が行われます。

2. **ログイン**
    - コンソールまたは SSH からログインします。
        - **ユーザー名**: `ec2-user`
        - **パスワード**: `user-data` で指定したパスワード
        - **SSH 接続コマンド例**:

            ```bash
            ssh ec2-user@<仮想マシンのIPアドレス>
            ```

3. **seed.iso の切断**

    - 初回起動・ログインが完了した後は、仮想マシンの設定から CD/DVD の接続を解除（または `seed.iso` の割り当てを解除）して問題ありません。

---

### 5. 起動後の推奨初期設定

#### パッケージの更新

```bash
sudo dnf update -y
```

#### VMware Tools (open-vm-tools) の確認

VMware 向けの AL2023 公式イメージには、標準で `open-vm-tools` が組み込まれています。サービスが稼働しているか確認します。

```bash
systemctl status vmtoolsd
```

※もし起動していない、または未導入の場合は以下で導入・有効化できます。

```bash
sudo dnf install -y open-vm-tools
sudo systemctl enable --now vmtoolsd
```

#### タイムゾーンの設定

デフォルトでは UTC に設定されています。必要に応じて JST（日本標準時）に変更します。

```bash
# タイムゾーンを東京に変更
sudo timedatectl set-timezone Asia/Tokyo

# 設定確認
timedatectl
```

#### root パスワードの設定 (任意)

デフォルトでは `root` パスワードは未設定であり、管理者操作は `ec2-user` から `sudo` コマンドで実行します。必要に応じて root パスワードを設定します。

```bash
sudo passwd root
```
