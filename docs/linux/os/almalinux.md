---
title: Alma Linux
description: Alma Linux の操作に関するメモです
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

Alma Linux
===

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
    # mDNS (avahi)
    sudo firewall-cmd --add-service=mdns --zone=internal --permanent
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

### 画像

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
