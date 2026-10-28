Linux Mint
===

## 日本語入力

```bash
sudo apt install -y \
    fcitx5 fcitx5-mozc fcitx5-frontend-gtk2 \
    fcitx5-frontend-gtk3 fcitx5-frontend-gtk4 \
    fcitx5-frontend-qt5 kde-config-fcitx5
```

---

## ブラウザのインストール

### Brave

```bash
# Braveの公式GPGキーをダウンロードと配置
sudo curl -fsSLo \
    /usr/share/keyrings/brave-browser-archive-keyring.gpg \
    https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg
# BraveのAPTリポジトリの追加
echo \
    "deb [signed-by=/usr/share/keyrings/brave-browser-archive-keyring.gpg] https://brave-browser-apt-release.s3.brave.com/ stable main" \
    | sudo tee /etc/apt/sources.list.d/brave-browser-release.list
# Brave ブラウザのインストール
sudo apt update
sudo apt install -y brave-browser
```

### Google Chrome

```bash
# Googleの署名キー（GPGキー）の登録
wget -qO- https://dl.google.com/linux/linux_signing_key.pub \
    | gpg --dearmor \
    | sudo tee /etc/apt/keyrings/google-chrome.gpg > /dev/null

# ChromeのAPTリポジトリの追加
echo \
    "deb [arch=amd64 signed-by=/etc/apt/keyrings/google-chrome.gpg] http://dl.google.com/linux/chrome/deb/ stable main" \
    | sudo tee /etc/apt/sources.list.d/google-chrome.list > /dev/null
# Google Chrome のインストール
sudo apt update
sudo apt install -y google-chrome-stable
```

### Microsoft Edge

```bash
# 公開鍵（GPGキー）の登録
curl -fSsL https://packages.microsoft.com/keys/microsoft.asc \
    | gpg --dearmor \
    | sudo tee /usr/share/keyrings/microsoft-edge.gpg > /dev/null

# Edge用リポジトリの追加
echo \
    "deb [arch=amd64 signed-by=/usr/share/keyrings/microsoft-edge.gpg] https://packages.microsoft.com/repos/edge stable main" \
    | sudo tee /etc/apt/sources.list.d/microsoft-edge.list > /dev/null

# Microsoft Edge のインストール
sudo apt update
sudo apt install -y microsoft-edge-stable
```

---

## テキストエディタのインストール

### Sublime Text

```bash
# 公式のGPGキーのダウンロードと登録
wget -qO - \
    https://download.sublimetext.com/sublimehq-pub.gpg \
    | gpg --dearmor \
    | sudo tee /etc/apt/keyrings/sublimehq-archive.gpg > /dev/null

# Sublime TextのAPTリポジトリの追加
echo \
    "deb [signed-by=/etc/apt/keyrings/sublimehq-archive.gpg] https://download.sublimetext.com/ apt/stable/" \
    | sudo tee /etc/apt/sources.list.d/sublime-text.list > /dev/null

# Sublime Text のインストール
sudo apt update
sudo apt install -y sublime-text
```

### VSCode

```bash
# 必要な依存パッケージのインストール
sudo apt update
sudo apt install -y wget gpg apt-transport-https

# MicrosoftのGPG公開鍵の登録
wget -qO- https://packages.microsoft.com/keys/microsoft.asc \
    | gpg --dearmor > packages.microsoft.gpg
sudo install -D -o root -g root -m 644 packages.microsoft.gpg \
    /etc/apt/keyrings/packages.microsoft.gpg
rm -f packages.microsoft.gpg

# VS Code の APT リポジトリの追加
echo \
    "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" \
    | sudo tee /etc/apt/sources.list.d/vscode.list > /dev/null

# VSCode のインストール
sudo apt update
sudo apt install -y code
```


---

## その他

### VMware tools

- インストール

    ```bash
    sudo apt install -y open-vm-tools open-vm-tools-desktop
    ```

- 起動設定

    ```bash
    sudo systemctl start open-vm-tools
    sudo systemctl enable open-vm-tools
    ```

- 共有フォルダのマウント

    **/etc/fstab 設定例**

    ```text
    .host:/ /mnt/hgfs fuse.vmhgfs-fuse allow_other,auto_unmount,defaults 0 0
    ```

    マウントポイントは **/mnt/hgfs** を想定しています。事前に、

    ```bash
    sudo mkdir -p /mnt/hgfs
    ```

    を実行するなどして、マウントポイントを作成しておいてください。

    :::note
    VMware Workstation の共有フォルダを使用するためには、事前に仮想マシンの設定で共有フォルダを設定しておく必要があります。
    :::

    ```bash
    # systemctl の再読み込み
    sudo systemctl daemon-reload
    # マウント
    mount -a
    ```
