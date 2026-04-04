---
title: WSL
description: Windows Subsystem for Linux (WSL) は、Windows 上で Linux 環境をネイティブに実行できる機能です。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

Windows Subsystem for Linux (WSL)
===

## 前提条件

|     |                                      |
| :-- | :----------------------------------- |
| CPU | x86_64                               |
| OS  | Windows 11 Pro（バージョン21H2以降） |

## WSL のインストール手順

1. 管理者権限で Powershell 起動
2. WSL のインストール

    ```powershell
    wsl --install
    ```

    このコマンドで WSL2 と既定の Linux ディストリビューション（Ubuntu）がインストールされます。  
    Ubuntu 24.04 をインストールしたい場合は、下記のように Linux ディストリビューションを指定してインストールコマンドを実行します。

    ```powershell
    wsl --install -d Ubuntu-24.04
    ```

3. Windows の再起動
4. WSL ゲスト OS の初期設定

    OS 再起動後、WSL 起動時にユーザー名とパスワードを設定します。

    ```bash
    Enter new UNIX username: [ユーザー名を入力]
    Enter new UNIX password: [パスワードを入力]
    Retype new UNIX password: [パスワードを再入力]
    ```

---

## よく使用する wsl コマンド

- wsl コマンドでインストールできる Linux 一覧表示

    ```powershell
    wsl --list --online
    # または
    wsl -l -o
    ```

- WSL 環境にインストールした Linux 一覧表示

    ```powershell
    wsl --list --verbose
    # または
    wsl -l -v
    ```

- WSL 環境で起動している Linux の停止

    ```powershell
    wsl --terminate Ubuntu-24.04
    # または
    wsl -t Ubuntu-24.04
    ```

- WSL 環境で起動しているすべての Linux の停止

    ```powershell
    wsl --shutdown
    ```

- WSL にインストールした Linux の削除

    ```powershell
    wsl --unregister <Distro>
    ```

- WSL 自身の更新

    ```powershell
    wsl --update
    ```

- WSL コマンドのヘルプ

    ```powershell
    wsl --help
    ```

---

## WSL のメンテナンス

### 肥大化した VHDX の未使用領域の開放

WSL の VHDX ファイルはファイルの削除後も、仮想ディスクファイルのサイズは自動的に縮小されません。  
Hyper-V をインストールすると別の方法もあるのですが、ここでは　**diskpart コマンド** を使用して未使用領域を開放し、ファイルサイズを縮小します。

1. WSL を停止する

    ```powershell
    wsl --shutdown
    ```

2. VHDX ファイルのパスを確認する

    VHDX ファイルは通常、以下のパスに格納されています。

    ```
    %LOCALAPPDATA%\wsl\{UUID}\ext4.vhdx
    ```

    Debian 13 の場合）:
    ```
    C:\Users\<ユーザ名>\AppData\Local\wsl\{8b2a7f73-ae66-4f09-b25a-d9694be2558b}\ext4.vhdx
    ```

3. 管理者権限で PowerShell を起動し、`diskpart` を実行する

    ```powershell
    diskpart
    ```

    プロンプトが **DISKPART>** になる、もしくは、別のコマンドプロンプトが起動します。

4. **diskpart** 内で以下のコマンドを順に実行する

    ```
    select vdisk file="C:\Users\<ユーザ名>\AppData\Local\wsl\{8b2a7f73-ae66-4f09-b25a-d9694be2558b}\ext4.vhdx"
    attach vdisk readonly
    compact vdisk
    detach vdisk
    exit
    ```

    - **select vdisk file=** には手順 2 で確認した VHDX ファイルの絶対パスを指定します。
    - **compact vdisk** の完了までに時間がかかる場合があります。
