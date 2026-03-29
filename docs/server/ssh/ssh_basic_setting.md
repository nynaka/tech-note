---
title: SSH基本設定
description: SSH の基本的な設定例です
sidebar_position: 0
#id: home
#slug: /my-custom-url
---

SSH 基本設定
===


## インストール

<details>
### Debian / Ubuntu

```bash title="SSH サーバ・クライアントのインストール"
sudo apt install ssh
```

```bash title="SSH サーバの起動"
# SSH サーバの起動
sudo systemctl start ssh
# SSH サーバの自動起動
sudo systemctl enable ssh
```

**ssh** ではなく **openssh-server**、**openssh-client** のように、サーバ・クライアントを明示してインストールすると、サーバアプリだけとかクライアントアプリだけをインストールすることができます。

### Fedora / Alma Linux

```bash title="SSH サーバ・クライアントのインストール"
sudo dnf install openssh-server openssh-clients
```

```bash title="SSH サーバの起動"
# SSH サーバの起動
sudo systemctl start sshd
# SSH サーバの自動起動
sudo systemctl enable sshd
```
</details>

## 設定

### パスワード認証

デフォルトで Linux アカウントの ID／パスワードを使用した認証が有効になっています。  
**/etc/ssh/sshd_config** で **PasswordAuthentication** に **no** を設定すれば、パスワード認証を無効にできます。

### 公開鍵認証

- キーペア作成

    RSA より EdDSA の方が安全で、鍵長も短いのでお勧めです。

    - RSA 鍵

        ```bash
        ssh-keygen -t rsa -b 4096
        ```

        上記コマンドを実行すると `$HOME/.ssh` 配下に、秘密鍵 (id_rsa) と公開鍵 (id_rsa.pub) が生成されます。

    - ED 25519 鍵

        ```bash
        ssh-keygen -t ed25519
        ```

        上記コマンドを実行すると `$HOME/.ssh` 配下に、秘密鍵 (id_ed25519) と公開鍵 (id_ed25519.pub) が生成されます。

- 公開鍵の設定

    $HOME/.ssh/authorized_keys に登録します。  
    このファイル名は /etc/ssh/sshd_config で変更できます。

    ```bash
    # RSA 鍵
    cat $HOME/.ssh/id_rsa.pub >> $HOME/.ssh/authorized_keys
    # ED 25519 鍵
    cat $HOME/.ssh/id_ed25519.pub >> $HOME/.ssh/authorized_keys
    ```

- 秘密鍵の取り扱い

    安全な方法で SSH クライアントにコピーします。秘密鍵の安全なコピー手段を検討するより、SSH クライアントで鍵を生成して公開鍵を SSH サーバにコピーした方が安全性は高いと思います。

- SSH クライアントコンフィグ

    下記のように設定ファイルを用意すると、`ssh ssh_server` のように `Host` で設定した名前指定で SSH ログインできるようになります。

    ```text title="$HOME/.ssh/config"
    Host ssh_server
        HostName localhost
        User debian
        Port 22
        IdentityFile /home/debian/.ssh/id_rsa
    ```


## 稀によくある問題

- SSH 接続が一定時間で切断される

    SSH では一定時間、クライアントから応答がないと自動的に切断する機能があり、デフォルト値 (ClientAliveInterval=0) の場合、応答確認は行わずに切断する、という設定なのですが、たいていはかなりの時間、接続が維持されます。  
    ただ、よくわからないタイミングで切断されることを回避したい場合は、下記のように ClientAliveInterval と ClientAliveCountMax を設定すると `ClientAliveInterval - ClientAliveCountMax` 秒間、無応答でも接続が維持されるようになります。

    ```text title="/etc/ssh/sshd_config"
    # ClientAliveInterval 0
    # ClientAliveCountMax 3
    ↓
    ClientAliveInterval 60
    ClientAliveCountMax 20
    ```
