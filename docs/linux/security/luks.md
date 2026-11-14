---
title: LUKS
description: Linux 標準のブロックデバイス暗号化機能です。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

LUKS (Linux Unified Key Setup)
===

## 概要

LUKS (Linux Unified Key Setup) は、Linux 標準のブロックデバイス暗号化機能です。  
**cryptsetup** ツールにより管理され、ブロックデバイス全体を AES などのアルゴリズムで暗号化します。  
複数のキースロットをサポートするため、パスフレーズや鍵ファイルを複数登録でき、柔軟な鍵管理が可能です。

## インストール

- Debian Linux / Ubuntu Linux

    ```bash
    sudo apt install -y cryptsetup
    ```

## 暗号ブロックデバイスの作成

1. 鍵ファイル作成

    LUKS で、鍵ファイルで暗号化を行う場合、鍵ファイルの形式に制約は（たぶん）ありません。ただ、数 MB の、鍵としては巨大なファイルを設定すると正常な動作ができなくなるようです。

    ここでは、試した OS（Debian Linux13）に、OpenSSL 3.5 がインストールされていたので、ML-DSA-65 秘密鍵を生成し、鍵ファイルとして使用することにしました。  

    ```bash
    sudo openssl genpkey \
        -algorithm ML-DSA-65 \
        -outform DER \
        -out /root/key/luks_keyfile.der
    sudo chmod 400 /root/key/luks_keyfile.der
    ```

2. 鍵ファイルでブロックデバイスを暗号化する

    **luksFormat** を使用してブロックデバイスに LUKS ヘッダーを書き込み、暗号化を設定します。  

    ```bash
    # LUKS フォーマット（/dev/nvme0n1p3 は実際のブロックデバイス）
    sudo cryptsetup luksFormat \
        --key-file /root/key/luks_keyfile.der \
        /dev/nvme0n1p3
    ```

    ```bash
    # ブロックデバイスを復号してマッピング
    sudo cryptsetup luksOpen \
        --key-file /root/key/luks_keyfile.der \
        /dev/nvme0n1p3 encrypted_disk
    ```

    コマンドの最後の **encrypted_disk** は復号したブロックデバイスの ID のようなもので、**/dev/mapper/encrypted_disk** の形でマッピングされます。

    ```bash
    # マッピングの解除
    sudo cryptsetup luksClose encrypted_disk
    ```

3. 復号パスワードを登録する

    **luksAddKey** を使用すると、復号に使用する鍵ファイルやパスフレーズを追加登録できます。  
    鍵ファイルが使えなくなった場合の復旧手段や、鍵のローテーション等に使用できます。  
    キースロットは、LUKS1 (古いバージョン) では 8 個、LUKS 2 (最近の Linux) では 32 個だと思います。

    ```bash
    # 新しいパスフレーズの追加
    sudo cryptsetup luksAddKey \
        --key-file /root/key/luks_keyfile.der \
        /dev/nvme0n1p3
    ```

4. **/etc/crypttab** への登録

    システム起動時に自動で復号・マッピングするために `/etc/crypttab` へ登録します。  
    デバイスは UUID で指定するとデバイス名が変わっても影響を受けません。

    ```bash
    # デバイスの UUID の確認
    sudo cryptsetup luksUUID /dev/nvme0n1p3
    ```

    **/etc/crypttab** に以下を追記します（`UUID` は上記コマンドで取得した値）。  
    luks_nvme0n1p3 は管理しやすい名称を設定してください。

    ```text
    luks_nvme0n1p3  UUID=<UUID>  /root/key/luks_keyfile.der  luks,timeout=5
    ```

    **timeout=5** は、鍵ファイルでの復号に失敗した場合のパスフレーズ入力待ち時間で、設定した時間待ってもパスフレーズが入力されなかった場合、諦めて次のプロセスに進ませる、無限待ち回避設定です。


## よくある問題

- Debian Linux

    - /etc/crypttab を設定して再起動したが、/dev/mapper/ 配下に、設定したデバイス名が作られない

        ```bash
        sudo systemd-cryptsetup attach luks_nvme0n1p3 /dev/nvme0n1p3 /root/key/luks_keyfile.der luks
        ```

        を実行して、**sudo: systemd-cryptsetup: コマンドが見つかりません** となる場合、systemd-cryptsetup がインストールされていないため、OS 起動時の自動マッピングが実行されません。

        ```bash
        sudo apt install -y systemd-cryptsetup
        ```

        で解決すると思います。

- パーティション

    - パーティション全未作成領域を一つのパーティションに設定する

        - パーティションテーブルの確認

            ```bash
            sudo parted /dev/nvme0n1 print free
            ```

            ```bash title="パーティションテーブル"
            ディスク /dev/nvme0n1: 512GB
            セクタサイズ (論理/物理): 512B/512B
            パーティションテーブル: gpt
            ディスクフラグ:
            
            番号  開始    終了    サイズ  ファイルシステム  名前                  フラグ
                  17.4kB  1049kB  1031kB  空き容量
             1    1049kB  106MB   105MB   fat16             EFI System Partition  boot, esp
             2    106MB   41.1GB  41.0GB  xfs
                  41.1GB  512GB   471GB   空き容量
            ```

        - ディスクの残り全部を使ってパーティションを作製する

            ```bash
            sudo parted -a optimal /dev/nvme0n1 mkpart data 41.1GB 100%
            ```

        :::note[四捨五入による誤差について]
        print free の 41.1GB は小数点以下が丸められた表示ですが、-a optimal を付けているため、システムが最適な境界位置へ自動調整して作成します。数 MB の隙間すらなくピッタリ隣接させたい場合は unit s でセクタ番号を確認して指定する方法があるので、ギリギリを攻めたい場合はセクタ番号指定を使用する（らしい）。
        :::