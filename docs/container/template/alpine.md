---
title: Alpine Linux
description: Alpine Linux の Docker コンテナテンプレです
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

Alpine Container
===

## ファイル構成

```text
alpine
|-- Dockerfile
|-- docker-compose.yaml
```

## 通常のコンテナ

```dockerfile title="Dockerfile"
# Base Image
FROM alpine:latest

# パッケージの更新とインストール
# bash: デフォルトの sh 以外のシェルが必要な場合
# tzdata: タイムゾーン設定用
# musl-locales: ロケール設定用（Alpineは標準でUTF-8をサポートしますが、明示的な設定用）
RUN apk update \
    && apk add --no-cache \
        bash \
        procps \
        tzdata \
        musl-locales \
        musl-locales-lang

# ロケールの設定
ENV LANG=ja_JP.UTF-8 \
    LANGUAGE=ja_JP:ja \
    LC_ALL=ja_JP.UTF-8

CMD ["/bin/bash"]
```

```yaml title="docker-compose.yaml"
services:

  alpine:
    build: .
    container_name: alpine
    hostname: alpine
    restart: always
    #ports:
    #  - "5432:5432"
    #volumes:
    #  - ./volume/home:/home
    tty: true
    command: >
      /bin/bash
    environment:
      TZ: Asia/Tokyo
```


## privileged なコンテナ

```dockerfile title="Dockerfile"
# Base Image
FROM alpine:latest

# OpenRC と必要なパッケージのインストール
RUN apk update \
    && apk add --no-cache \
        openrc \
        bash \
        procps \
        tzdata \
        musl-locales \
        musl-locales-lang \
    && sed -i 's/^\(tty\d\):.:respawn:\/sbin\/getty/\1:.:once:\/sbin\/getty/g' /etc/inittab \
    && mkdir -p /run/openrc \
    && touch /run/openrc/softlevel

# ロケールの設定
ENV LANG=ja_JP.UTF-8 \
    LANGUAGE=ja_JP:ja \
    LC_ALL=ja_JP.UTF-8

# OpenRC を起動
CMD ["/sbin/init"]
```

```yaml title="docker-compose.yaml"
services:

  alpine:
    build: .
    container_name: alpine
    hostname: alpine
    restart: always
    privileged: true
    ## cgroup v1 環境のみ: 必要な場合は以下をアンコメント（:rw が必要）
    ## cgroup v2 環境では privileged: true のみで動作するためマウント不要
    #volumes:
    #  - /sys/fs/cgroup:/sys/fs/cgroup:rw
    tty: true
    # Alpine の init (OpenRC) を利用
    command: >
      /sbin/init
    environment:
      TZ: Asia/Tokyo
```
