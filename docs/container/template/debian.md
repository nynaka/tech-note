---
title: Debian
description: Debian の Docker コンテナテンプレです
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

Debian Container
===

## ファイル構成

```text
debian
|-- Dockerfile
|-- docker-compose.yaml
```

## 通常のコンテナ

```dockerfile title="Dockerfile"
# Base Image
FROM debian:latest

# パッケージリストの更新とインストール
# locales: ロケール生成用
# procps: ps コマンド
RUN apt-get update \
    && apt-get install -y \
        locales procps \
    && rm -rf /var/lib/apt/lists/*

# 日本語ロケールの生成
RUN sed -i -e 's/# ja_JP.UTF-8 UTF-8/ja_JP.UTF-8 UTF-8/' /etc/locale.gen && \
    locale-gen

# ロケールの設定
ENV LANG=ja_JP.UTF-8 \
    LANGUAGE=ja_JP:ja \
    LC_ALL=ja_JP.UTF-8

CMD ["bash"]
```

```yaml title="docker-compose.yaml"
services:

  debian:
    build: .
    container_name: debian
    hostname: debian
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
FROM debian:latest

# パッケージの更新と systemd, locales, procps のインストール
RUN apt-get update \
    && apt-get install -y \
        systemd systemd-sysv locales procps \
    && rm -rf /var/lib/apt/lists/*

# 日本語ロケールの生成
RUN sed -i -e 's/# ja_JP.UTF-8 UTF-8/ja_JP.UTF-8 UTF-8/' /etc/locale.gen && \
    locale-gen

# ロケールの設定
ENV LANG=ja_JP.UTF-8 \
    LANGUAGE=ja_JP:ja \
    LC_ALL=ja_JP.UTF-8

# systemd 用の環境変数
ENV container docker

CMD ["/lib/systemd/systemd"]
```

```yaml title="docker-compose.yaml"
services:

  debian:
    build: .
    container_name: debian
    hostname: debian
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
