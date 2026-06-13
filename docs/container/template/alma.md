---
title: Alma Linux
description: AlmaLinux の Docker コンテナテンプレです
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

AlmaLinux Container
===

## ファイル構成

```text
almalinux
|-- Dockerfile
|-- docker-compose.yaml
```

## 通常のコンテナ

```dockerfile title="Dockerfile"
# Base Image
FROM almalinux:latest

# glibc-langpack-ja: 日本語ロケールデータ
# procps-ng: ps コマンドなどのシステムプロセス管理ツール
RUN dnf install -y \
        glibc-langpack-ja procps-ng \
    && dnf clean all

# ロケールの設定
ENV LANG=ja_JP.UTF-8 \
    LC_ALL=ja_JP.UTF-8

CMD ["bash"]
```

```yaml title="docker-compose.yaml"
services:

  almalinux:
    build: .
    container_name: almalinux
    hostname: almalinux
    restart: always
    #ports:
    #  - "8080:80"
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
FROM almalinux:latest

# systemd と日本語環境のインストール
RUN dnf update -y \
    && dnf install -y \
        systemd \
        glibc-langpack-ja \
        procps-ng \
    && dnf clean all

# 不要なユニットファイルの削除（コンテナ内での安定動作のため）
RUN (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i == \
    systemd-tmpfiles-setup.service ] || rm -f $i; done); \
    rm -f /lib/systemd/system/multi-user.target.wants/*;\
    rm -f /etc/systemd/system/*.wants/*;\
    rm -f /lib/systemd/system/local-fs.target.wants/*; \
    rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
    rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
    rm -f /lib/systemd/system/basic.target.wants/*;\
    rm -f /lib/systemd/system/anaconda.target.wants/*;

# ロケールの設定
ENV LANG=ja_JP.UTF-8 \
    LC_ALL=ja_JP.UTF-8

# systemd を利用する場合は、シグナルを正しく受け取れるように設定
STOPSIGNAL SIGRTMIN+3

CMD ["/sbin/init"]
```

```yaml title="docker-compose.yaml"
services:

  almalinux:
    build: .
    container_name: almalinux
    hostname: almalinux
    restart: always
    privileged: true
    ## cgroup v1 環境のみ: 必要な場合は以下をアンコメント（:rw が必要）
    ## cgroup v2 環境では privileged: true のみで動作するためマウント不要
    #volumes:
    #  - /sys/fs/cgroup:/sys/fs/cgroup:rw
    tty: true
    command: >
      /sbin/init
    environment:
      TZ: Asia/Tokyo
```
