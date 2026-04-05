---
title: Ubuntu
description: Ubuntu の Docker コンテナテンプレです
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

Ubuntu Container
===

## ファイル構成

```text
ubuntu
|-- Dockerfile
|-- docker-compose.yaml
```

## 通常のコンテナ

```dockerfile title="Dockerfile"
# Base Image
FROM ubuntu:latest

# パッケージリストの更新とインストール
# locales: ロケール生成用
# procps: ps コマンド
# tzdata: タイムゾーン設定用
RUN apt-get update \
    && apt-get install -y \
        locales procps tzdata \
    && rm -rf /var/lib/apt/lists/*

# 日本語ロケールの生成
RUN locale-gen ja_JP.UTF-8

# ロケールの設定
ENV LANG=ja_JP.UTF-8 \
    LANGUAGE=ja_JP:ja \
    LC_ALL=ja_JP.UTF-8

CMD ["bash"]
```

```yaml title="docker-compose.yaml"
services:

  ubuntu:
    build: .
    container_name: ubuntu
    hostname: ubuntu
    restart: always
    #ports:
    #  - "5432:5432"
    #volumes:
    #  - ./volume/home:/home
    tty: true
    command: >
      bash
    environment:
      TZ: Asia/Tokyo
```


## privileged なコンテナ

```dockerfile title="Dockerfile"
# Base Image
FROM ubuntu:latest

# パッケージの更新と systemd, locales, procps のインストール
# ubuntu では DEBIAN_FRONTEND=noninteractive を指定して
# インストール時の対話プロンプトを回避します
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y \
        systemd systemd-sysv locales procps tzdata \
    && rm -rf /var/lib/apt/lists/*

# 日本語ロケールの生成
RUN locale-gen ja_JP.UTF-8

# ロケールの設定
ENV LANG=ja_JP.UTF-8 \
    LANGUAGE=ja_JP:ja \
    LC_ALL=ja_JP.UTF-8

# systemd 用の環境変数
ENV container docker

# systemd をフォアグラウンドで実行
CMD ["/lib/systemd/systemd"]
```

```yaml title="docker-compose.yaml"
services:

  ubuntu:
    build: .
    container_name: ubuntu
    hostname: ubuntu
    restart: always
    privileged: true
    ## cgroup v1 環境のみ: 必要な場合は以下をアンコメント（:rw が必要）
    ## cgroup v2 環境では privileged: true のみで動作するためマウント不要
    #volumes:
    #  - /sys/fs/cgroup:/sys/fs/cgroup:rw
    #ports:
    #  - "5432:5432"
    tty: true
    stop_signal: SIGRTMIN+3
    command: >
      /lib/systemd/systemd
    environment:
      TZ: Asia/Tokyo
```
