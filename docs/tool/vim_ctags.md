---
title: vim ctags
description: vim での ctags の使い方メモ
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

Vim タグジャンプ
===

## インストール

```bash title="Debian / Ubuntu 他"
sudo apt install -y universal-ctags
```

```bash title="Fedora / RHEL 系 (ソースビルド)"
# 依存パッケージのインストール
sudo dnf install -y autoconf automake gcc make pkg-config

# ソースの取得とビルド
git clone https://github.com/universal-ctags/ctags.git
cd ctags
./autogen.sh
./configure
make
sudo make install
```

## タグファイルの生成

タグジャンプを使うには、まず `ctags` でタグファイル (`tags`) を生成する必要がある。

```bash title="基本 (カレントディレクトリ以下を再帰スキャン)"
ctags -R .
```

```bash title=".git ディレクトリを除外"
ctags -R --exclude=.git .
```

```bash title="特定のファイルだけを対象に生成"
ctags src/main.py src/utils.py
```

:::tip `.gitignore` への追加
生成された `tags` ファイルはリポジトリに含めないことが多い。

```
echo "tags" >> .gitignore
```
:::

## 言語

- ctags が対応している言語・拡張子の確認

    ```bash
    ctags --list-maps
    ```

- Python コードだけを対象としたタグ生成

    ```bash
    ctags --languages=Python -R .
    ```


## Vim の設定

`tags` ファイルの検索パスを設定しておくと、サブディレクトリ内からでも自動的に上位の `tags` を見つけてくれる。

```vim title="~/.vimrc"
" カレントファイルのディレクトリから上位へ向かって tags を自動探索
set tags=./tags;,tags;
```


## タグジャンプ

| コマンド              | 内容                                                             |
| :-------------------- | :--------------------------------------------------------------- |
| Ctrl-]                | カーソルのある単語に対してタグジャンプ                           |
| [N]Ctrl-t             | N 個前のタグ位置に戻る (タグスタックは保持)                      |
| :[N]pop(:po)          | N 個前のタグ位置に戻る (タグスタックを削除)                      |
| :tags                 | タグスタックを表示                                               |
| :tag(:ta) keyword      | キーワードの定義位置にタグジャンプする                           |
| :tselect(:ts) keyword | キーワードのジャンプ先候補を表示                                 |
| :tjump(:tj) keyword   | キーワードのジャンプ先候補を表示 (1個しかない場合は直接ジャンプ) |
| :tnext(:tn)           | 次の候補へジャンプ                                               |
| :tNext(:tN)           | 前の候補へジャンプ                                               |
| :tprevious(:tp)       | 前の候補へジャンプ                                               |
| :trewind(:tr)         | 最初に一致した候補へジャンプ                                     |
| :tfirst(:tf)          | 最初に一致した候補へジャンプ                                     |
| :tlast                | 最後に一致した候補へジャンプ                                     |
| g]                    | カーソル位置の単語に対して :tselect を実行                       |
| g Ctrl-]              | カーソル位置の単語に対して :tjump を実行                         |
| Ctrl-W ]              | タグを水平分割した新しいウィンドウで開く                         |
| Ctrl-W Ctrl-]         | 同上                                                             |
| Ctrl-W }              | タグをプレビューウィンドウで開く                                 |
| :pclose (:pc)         | プレビューウィンドウを閉じる                                     |


## vim 起動時のタグジャンプ

Vim 起動時に `-t` オプションを使うと、直接タグ位置を開ける。

```bash
vim -t <タグ名>
```


## 参考サイト

* [とほほのVim入門(タグジャンプ)](https://www.tohoho-web.com/vim/tag.html)
* [タグジャンプを実現するためにctagsの環境を整える](https://ryamashina.com/itml/20210429/)
