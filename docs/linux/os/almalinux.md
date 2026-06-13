---
title: Alma Linux メモ
description: Alma Linux の操作に関するメモです
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

Alma Linux メモ
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

## アプリ

### [LibreOffice](https://ja.libreoffice.org/)

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
    tar zxvf LibreOffice_26.2.3_Linux_x86-64_rpm.tar.gz
    tar zxvf LibreOffice_26.2.3_Linux_x86-64_rpm_langpack_ja.tar.gz
    tar zxvf LibreOffice_26.2.3_Linux_x86-64_rpm_helppack_ja.tar.gz
    # インストール
    find ./ -name "*.rpm" -exec sudo dnf install -y {} +
    ```

### テキストエディタ

- VSCode

    ```bash
    wget "https://code.visualstudio.com/sha/download?build=stable&os=linux-rpm-x64" \
        -O code.rpm
    sudo dnf install -y code.rpm
    ```

### ブラウザ

- Chromium

    ```bash
    sudo dnf install -y chromium
    ```

- thunderbird

    ```bash
    sudo dnf install -y thunderbird
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
