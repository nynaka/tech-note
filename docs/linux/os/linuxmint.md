Linux Mint
===

## 設定

### 日本語入力

```bash
sudo apt install -y \
    fcitx5 fcitx5-mozc fcitx5-frontend-gtk2 \
    fcitx5-frontend-gtk3 fcitx5-frontend-gtk4 \
    fcitx5-frontend-qt5 kde-config-fcitx5
```

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
