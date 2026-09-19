---
title: Alma Linux
description: Alma Linux の操作に関するメモです
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

Alma Linux
===

## 設定

### swappiness

- swappiness の設定

    ```bash
    echo "vm.swappiness = 1" | sudo tee /etc/sysctl.d/99-swappiness.conf
    sudo sysctl --system
    ```

- swappiness の設定値確認

    ```bash
    sysctl vm.swappiness
    # または
    cat /proc/sys/vm/swappiness
    ```

- 設定値の意味

    - `vm.swappiness` は、カーネルがメモリページをどの程度積極的にスワップ領域へ退避（スワップアウト）させるかを制御するパラメータ（0〜100）です。
    - **`60`（デフォルト値）**:
        - メモリとスワップの利用バランスを取る標準的な設定です。
        - カーネルがメモリ解放（ページ回収）を行う際、スキャン比率は内部的に `ファイルキャッシュ : スワップ = (200 - swappiness) : swappiness` で計算されます。`swappiness = 60` の場合は `140 : 60`（約 7:3）の比率となり、ファイルキャッシュの破棄を優先しつつ適度にスワップアウトも行います（※「メモリ残量が60%になったら発動」という閾値ではありません）。
    - **`0` より `1` を設定すべき理由**:
        - **`0`**: カーネル 3.5 以降では、空きメモリとキャッシュが完全に枯渇するまでスワップをほぼ完全に停止します。これによりキャッシュが過剰に破棄されてディスク I/O 性能が低下したり、メモリ不足時に OOM Killer が突発的に作動してプロセスが強制終了されるリスクが高まります。
        - **`1`**: 通常時はスワップアウトを極力回避しながらも、完全なメモリ枯渇時にはスワップを許可して OOM（Out of Memory）を回避する安全弁を残すことができます。そのため、スワップを最小限に抑えたい環境では `0` ではなく `1` を設定することが推奨されます。


## ネットワーク

### mDNS

- インストール

    ```bash
    sudo dnf install avahi
    ```

- 起動設定

    ```bash
    sudo systemctl enable avahi-daemon
    sudo systemctl start avahi-daemon
    ```

- ファイアウォールの設定

    ```bash
    # 恒久的な設定として追加
    sudo firewall-cmd --permanent --add-service=mdns

    # 設定を反映
    sudo firewall-cmd --reload
    ```

---

## Firewall

### firewalld の設定確認

```bash
sudo firewall-cmd --list-all
```

### Zone 関連コマンド

- Zone 一覧

    ```bash
    sudo firewall-cmd --get-zones
    ```

- Active Zone の確認

    ```bash
    sudo firewall-cmd --get-active-zones
    ```

- Default Zone の確認

    ```bash
    sudo firewall-cmd --get-default-zones
    ```

- Default Zone の変更

    ```bash
    sudo ZONE_NAME="internal"
    sudo firewall-cmd --set-default-zone=${ZONE_NAME}
    ```

- 特定の NIC に設定する Zone の変更

    ```bash
    NIC_NAME="eno1"
    ZONE_NAME="internal"
    sudo firewall-cmd --change-interface=${NIC_NAME} --zone=${ZONE_NAME}
    ```

- ゾーンに許可されているサービス一覧だけの確認

    ```bash
    sudo firewall-cmd --zone=internal --list-services
    ```

- ゾーンのすべての設定確認 (サービス一覧を含む)

    ```bash
    sudo firewall-cmd --zone=internal --list-all
    ```

### 通信を許可するサービスの追加・確認・削除

- サービスの追加

    ```bash
    # SSH
    sudo firewall-cmd --add-service=ssh --zone=internal --permanent
    # HTTPS
    sudo firewall-cmd --add-service=https --zone=internal --permanent
    # mDNS (avahi)
    sudo firewall-cmd --add-service=mdns --zone=internal --permanent

    # Docusaurus
    sudo firewall-cmd --add-port=3000/tcp --permanent
    # MkDocs
    sudo firewall-cmd --add-port=8000/tcp --permanent

    # 設定反映
    sudo firewall-cmd --reload
    ```

---

## スリープ (サスペンドやハイバーネート) の切り替え

- 有効化

    ```bash
    sudo systemctl unmask \
        sleep.target \
        suspend.target \
        hibernate.target \
        hybrid-sleep.target
    ```

    - GDM の自動サスペンド無効化

        1. 設定ディレクトリを作成

            ```bash
            sudo mkdir -p /etc/dconf/db/gdm.d
            ```

        2. GDM の電源管理設定ファイルを作成

            ```bash
            sudo tee /etc/dconf/db/gdm.d/01-power <<EOF
            [org/gnome/settings-daemon/plugins/power]
            sleep-inactive-ac-timeout=0
            sleep-inactive-ac-type='nothing'
            sleep-inactive-battery-timeout=0
            sleep-inactive-battery-type='nothing'
            EOF
            ```

        3. dconf データベースを更新して設定を反映

            ```bash
            sudo dconf update
            ```


        4. GDM を再起動

            :::warning
            現在ログイン中のデスクトップ環境は強制ログアウトされる。
            :::

            ```bash
            sudo systemctl restart gdm
            ```

- 無効化

    ```bash
    sudo systemctl mask \
        sleep.target \
        suspend.target \
        hibernate.target \
        hybrid-sleep.target
    ```

- 状態確認

    ```bash
    sudo systemctl status \
        sleep.target \
        suspend.target \
        hibernate.target \
        hybrid-sleep.target
    ```

---

## CUI アプリ

### Guake

:::warning
下記の手順ではインストールできるが動作しない。AlmaLinux ではもう一工夫必要らしい。
:::

```bash
# 1. pipx のインストール
sudo dnf install -y pipx python3-gobject

# 2. pipx で guake をインストール
pipx install --system-site-packages guake

# 3. 実行パスを通す（初回のみ）
pipx ensurepath
```

---

## GUI アプリ

### [LibreOffice](https://ja.libreoffice.org/)

#### RPM ファイルを使ったインストール

- ダウンロード

    ```bash
    # 本体
    wget https://download.documentfoundation.org/libreoffice/stable/26.2.3/rpm/x86_64/LibreOffice_26.2.3_Linux_x86-64_rpm.tar.gz
    # 日本語 UI
    wget https://download.documentfoundation.org/libreoffice/stable/26.2.3/rpm/x86_64/LibreOffice_26.2.3_Linux_x86-64_rpm_langpack_ja.tar.gz
    # 日本語ヘルプ
    wget https://download.documentfoundation.org/libreoffice/stable/26.2.3/rpm/x86_64/LibreOffice_26.2.3_Linux_x86-64_rpm_helppack_ja.tar.gz
    ```

- インストール

    ```bash
    # アーカイブファイルを展開
    tar zxvf LibreOffice_26.2.4_Linux_x86-64_rpm.tar.gz
    tar zxvf LibreOffice_26.2.4_Linux_x86-64_rpm_langpack_ja.tar.gz
    tar zxvf LibreOffice_26.2.4_Linux_x86-64_rpm_helppack_ja.tar.gz
    # インストール
    find ./ -name "*.rpm" -exec sudo dnf install -y {} +
    ```

#### Flatpak を使ったインストール

Flathub URL: https://flathub.org/ja/apps/org.libreoffice.LibreOffice

```bash
# LibreOffice 本体
sudo flatpak install flathub org.libreoffice.LibreOffice
# スペルチェッカー (アドオン)
sudo flatpak install flathub org.libreoffice.LibreOffice.BundledExtension.Voikko
```


### テキストエディタ

- VSCode

    **RPM ファイルを使ったインストール**

    ```bash
    wget "https://code.visualstudio.com/sha/download?build=stable&os=linux-rpm-x64" \
        -O code.rpm
    sudo dnf install -y code.rpm
    ```

    **Flatpak を使ったインストール**

    ```bash
    sudo flatpak install flathub com.visualstudio.code
    ```  

- Sublime Text

    ```bash
    # Stable for Fedora 41/dnf5 or newer
    sudo dnf config-manager --add-repo https://download.sublimetext.com/rpm/stable/x86_64/sublime-text.repo
    # Update dnf and install Sublime Text
    sudo dnf install sublime-text
    ```

    [dnf インストール手順](https://www.sublimetext.com/docs/linux_repositories.html#dnf) には GPG key のインストール手順がありますが、インストールエラーのまま省略しても大丈夫らしい。


### ブラウザ

- Google Chrome

    **RPM ファイルを使ったインストール**

    ```bash
    wget https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm
    sudo dnf install -y google-chrome-stable_current_x86_64.rpm
    ```

    **Flatpak を使ったインストール**

    ```bash
    sudo flatpak install flathub com.google.Chrome
    ```

- Brave

    **RPM ファイルを使ったインストール**

    ```bash
    sudo dnf install dnf-plugins-core
    sudo dnf config-manager \
        --add-repo https://brave-browser-rpm-release.s3.brave.com/brave-browser.repo
    sudo dnf install brave-browser
    ```

    **Flatpak を使ったインストール**

    ```bash
    sudo flatpak install flathub com.brave.Browser
    ```

- Microsoft Edge

    ```bash 
    sudo flatpak install flathub com.microsoft.Edge
    ```

- Chromium

    ```bash
    sudo dnf install -y chromium
    ```

- thunderbird

    **RPM ファイルを使ったインストール**

    ```bash
    sudo dnf install -y thunderbird
    ```

    **Flatpak を使ったインストール**

    ```bash
    sudo flatpak install flathub org.mozilla.thunderbird
    ```

### 画像・音楽

EPEL リポジトリや CRB（Code Ready Builder）に gimp や shotwell は登録されなくなったらしい？？

- Flatpakのインストール

    ```bash
    sudo dnf install flatpak
    ```

- Flathubリポジトリの追加

    ```bash
    flatpak remote-add \
        --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
    ```

- GIMPのインストール

    ```bash
    sudo flatpak install flathub org.gimp.GIMP
    ```

- Shotwellのインストール

    ```bash
    sudo flatpak install flathub org.gnome.Shotwell
    ```

- Rhythmbox

    ```bash
    sudo flatpak install flathub org.gnome.Rhythmbox3
    ```

### 動画

- VLC

    ```bash
    sudo flatpak install flathub org.videolan.VLC
    ```

## プログラミング言語

### C/C++

```bash
sudo dnf groupinstall "Development Tools" -y
sudo dnf install -y g++ cmake gdb
```

---

## その他

### UDEVGothic

```bash
wget https://github.com/yuru7/udev-gothic/releases/download/v2.2.0/UDEVGothic_HS_v2.2.0.zip -O /tmp/UDEVGothic_HS.zip
wget https://github.com/yuru7/udev-gothic/releases/download/v2.2.0/UDEVGothic_NF_v2.2.0.zip -O /tmp/UDEVGothic_NF.zip
wget https://github.com/yuru7/udev-gothic/releases/download/v2.2.0/UDEVGothic_v2.2.0.zip -O /tmp/UDEVGothic.zip
mkdir -p .fonts/udev_gothic
cd .fonts/udev_gothic
unzip /tmp/UDEVGothic_HS.zip
unzip /tmp/UDEVGothic_NF.zip
unzip /tmp/UDEVGothic.zip
```

### VMware Workstation Pro 共有フォルダ

- open-vm-tools のインストール

    ```bash
    # パッケージの更新（推奨）
    sudo dnf update -y

    # open-vm-tools と関連ツールのインストール
    sudo dnf install open-vm-tools open-vm-tools-desktop -y

    # サービスが有効か確認し、起動
    sudo systemctl enable --now vmtoolsd
    ```

- VMware の共有フォルダが見えているか確認

    ```bash
    vmware-hgfsclient
    ```

    ここでは **_share** が表示されたものとします。

- マウントポイント作成

    ```bash
    sudo mkdir -p /mnt/hgfs
    ```

- 手動マウント

    ```bash
    sudo vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other -o auto_unmount
    ```

- 自動マウント

    - /etc/fstab に追記

        ```bash
        .host:/    /mnt/hgfs    fuse.vmhgfs-fuse    allow_other,defaults    0    0
        ```

    - マウント

        ```bash
        sudo systemctl daemon-reload
        mount -a
        ```

        /mnt/hgfs/_share/ から共有フォルダに接続できます。
