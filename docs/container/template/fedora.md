---
title: Fedora
description: Fedora の Docker コンテナテンプレです
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

Fedora Container
===

## ファイル構成

```text
fedora
|-- Dockerfile
|-- docker-compose.yaml
```

## 通常のコンテナ

```dockerfile title="Dockerfile"
# Base Image
FROM fedora:latest

# glibc-langpack-ja: libcの日本語ロケールデータ
# procps-ng: ps コマンド
RUN dnf install -y \
    glibc-langpack-ja \
    procps-ng

# ロケールの設定
ENV LANG=ja_JP.UTF-8 \
    LC_ALL=ja_JP.UTF-8

CMD ["bash"]
```

```yaml title="docker-compose.yaml"
services:

  fedora:
    build: .
    container_name: fedora
    hostname: fedora
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
FROM fedora:latest

# systemd のインストール
RUN dnf update -y \
    && dnf install -y systemd

# glibc-langpack-ja: libcの日本語ロケールデータ
# procps-ng: ps コマンド
RUN dnf install -y \
    glibc-langpack-ja \
    procps-ng

# ロケールの設定
ENV LANG=ja_JP.UTF-8 \
    LC_ALL=ja_JP.UTF-8

CMD ["bash"]
```

```yaml title="docker-compose.yaml"
services:

  fedora:
    build: .
    container_name: fedora
    hostname: fedora
    restart: always
    privileged: true
    #ports:
    #  - "5432:5432"
    #volumes:
    #  - ./volume/home:/home
    tty: true
    command: >
      /sbin/init
    environment:
      TZ: Asia/Tokyo
```
