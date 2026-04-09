---
title: OpenSSL + oqs-provider による FN-DSA 署名・署名検証
description: FN-DSA による署名・署名検証を例とした OpenSSL + oqs-provider 使用手順です。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

OpenSSL + oqs-provider による FN-DSA 署名・署名検証
===

ここでは、耐量子暗号アルゴリズム **FN-DSA (IPS 206 / Falcon ベース)** を用いたデジタル署名を例に、oqs-provider を使用した openssl コマンドの署名・署名検証する手順を紹介します。 

> **アルゴリズム名について**  
> oqs-provider 内での FN-DSA の識別子は falcon512 (Security Level 1、AES-128 相当) および falcon1024 (Security Level 5、AES-256 相当) です。NIST の標準名 "FN-DSA" と oqs-provider 上の名前が異なる点に注意してください。  
> また、パディング付きバリアントとして falconpadded512 / falconpadded1024 も利用可能です。

---

## 前提条件

|              |                                                       |                         |
| :----------- | :---------------------------------------------------- | :---------------------- |
| OS           | Fedora Linux 43                                       | Docker Container        |
| OpenSSL      | 3.5.4                                                 | dnf でインストール      |
| oqs-provider | commit hash: 334f9fc57dc17335f10df561d531eadd8b2ed4d6 | Github のメインブランチ |


## システムアップデートと依存関係のインストール

ビルドに必要なツールとライブラリをインストールします。

```bash
sudo dnf update -y
sudo dnf install -y \
    git cmake gcc gcc-c++ ninja-build \
    openssl-devel python3-pytest \
    pkgconf-pkg-config
```

---

## liboqs のビルドとインストール

oqs-provider は、耐量子暗号の実装ライブラリ liboqs に依存しています。

```bash title="liboqs のインストール"
cd $HOME
git clone -b main https://github.com/open-quantum-safe/liboqs.git
cd liboqs
mkdir build && cd build
cmake -GNinja \
    -DCMAKE_INSTALL_PREFIX=/usr/local \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_SHARED_LIBS=ON \
    ..
ninja
sudo ninja install
```

```bash title="ダイナミックリンクキャッシュ更新"
echo "/usr/local/lib64" | sudo tee /etc/ld.so.conf.d/liboqs.conf
sudo ldconfig
```

---

## oqs-provider のビルドとインストール

OpenSSL 3.x で PQC アルゴリズムを使用可能にするプロバイダーをビルドします。

```bash title="oqs-provider のインストール"
cd $HOME
git clone -b main https://github.com/open-quantum-safe/oqs-provider.git
cd oqs-provider
mkdir build && cd build
cmake -GNinja \
    -DCMAKE_BUILD_TYPE=Release \
    -Dliboqs_DIR=/usr/local/lib64/cmake/liboqs/ \
    -DOPENSSL_ROOT_DIR=/usr \
    -DCMAKE_INSTALL_PREFIX=/usr/local \
    ..
ninja
sudo ninja install
```

> **注意**: liboqs_DIR のパスは環境によって異なります。見つからない場合は以下で確認してください。
>
> ```bash
> find /usr/local -name "liboqsConfig.cmake" 2>/dev/null
> # 例: /usr/local/lib64/cmake/liboqs/liboqsConfig.cmake
> ```

```bash title="oqsprovider.so の場所の確認"
find /usr -name "oqsprovider.so" 2>/dev/null
# 一般的なパス候補:
#   /usr/lib64/ossl-modules/oqsprovider.so
```

---

## OpenSSL プロバイダーの設定

### 環境変数による一時設定（動作確認・テスト向け）

現在のシェルセッションのみで有効にする場合：

```bash
# oqsprovider.so のあるディレクトリを指定（前節で確認したパス）
export OPENSSL_MODULES=/usr/lib64/ossl-modules
```

### openssl.cnf への永続設定（推奨）

セッションをまたいで常時利用するには、OpenSSL の設定ファイルを編集します。

```bash
# 設定ファイルの場所を確認
openssl version -d
# 例: OPENSSLDIR: "/etc/pki/tls"
# → 設定ファイルは /etc/pki/tls/openssl.cnf
```

設定ファイルの先頭付近にある openssl_conf セクションを以下のように編集・追記します。

```ini
# openssl.cnf の先頭に追記または修正
openssl_conf = openssl_init

[openssl_init]
providers = provider_sect

[provider_sect]
default = default_sect
oqsprovider = oqsprovider_sect

[default_sect]
activate = 1

[oqsprovider_sect]
activate = 1
# oqsprovider.so のフルパスを指定（OPENSSL_MODULES が設定されていない環境向け）
# module = /usr/lib64/ossl-modules/oqsprovider.so
```

> **重要**: default プロバイダーは PEM エンコード / デコードなど基本機能を担います。  
> oqsprovider だけを有効化すると鍵の読み書きに失敗するため、必ず両方を有効にしてください。

### プロバイダーの読み込み確認

```bash
# /etc 配下の openssl.cnf を設定した場合
openssl list -providers
# 上記の設定を /tmp/openssl.cnf に設定場合
OPENSSL_CONF=/tmp/openssl.cnf openssl list -providers
```

期待される出力例：

```
Providers:
  default
    name: OpenSSL Default Provider
    version: 3.5.4
    status: active
  oqsprovider
    name: OpenSSL OQS Provider
    version: 0.12.0-dev
    status: active
```

### FN-DSA アルゴリズムの確認

```bash
# openssl.cnf を設定していない場合
openssl list -signature-algorithms -provider oqsprovider | grep -i falcon
# /tmp/openssl.cnf を設定した場合
OPENSSL_CONF=/tmp/openssl.cnf openssl list -signature-algorithms | grep -i falcon
```

以下のようなエントリが表示されれば正常です：

```
falcon512 @ oqsprovider
p256_falcon512 @ oqsprovider
rsa3072_falcon512 @ oqsprovider
falconpadded512 @ oqsprovider
p256_falconpadded512 @ oqsprovider
rsa3072_falconpadded512 @ oqsprovider
falcon1024 @ oqsprovider
p521_falcon1024 @ oqsprovider
falconpadded1024 @ oqsprovider
p521_falconpadded1024 @ oqsprovider
```

---

## 直接署名・署名検証

ここでは falcon512（NIST Security Level 1、AES-128 相当）を使用します。  
より高いセキュリティが必要な場合は falcon1024（Security Level 5、AES-256 相当）に置き換えてください。

### -provider オプションについて

openssl.cnf で永続設定している場合は、-provider オプションは省略できます。  

```
-provider default -provider oqsprovider
```

default は PEM のエンコード / デコードのために必要です。


### 1. 秘密鍵の生成

```bash
openssl genpkey \
    -algorithm falcon512 \
    -provider default -provider oqsprovider \
    -out fndsa_priv.pem
```

生成確認：

```bash
openssl pkey -in fndsa_priv.pem -text -noout \
    -provider default -provider oqsprovider
```

### 2. 公開鍵の抽出

```bash
openssl pkey \
    -in fndsa_priv.pem \
    -pubout -out fndsa_pub.pem \
    -provider default -provider oqsprovider
```

### 3. 署名対象ファイルの作成

```bash
echo "This is a post-quantum test message." > message.txt
```

### 4. 直接署名

```bash title="直接署名"
openssl pkeyutl \
    -sign \
    -in message.txt \
    -inkey fndsa_priv.pem \
    -out message.sig \
    -rawin \
    -provider default -provider oqsprovider
```

> **`-rawin` オプションについて**  
> OpenSSL 3.x の pkeyutl では、ファイルを直接（ハッシュ化せずに）署名するために -rawin フラグが必要です。  
> これを省略すると、入力がすでにダイジェスト済みのデータとして扱われ、長いメッセージでエラーが発生することがあります。

```bash title="直接署名の検証"
openssl pkeyutl \
    -verify \
    -in message.txt \
    -sigfile message.sig \
    -pubin -inkey fndsa_pub.pem \
    -rawin \
    -provider default -provider oqsprovider
```

成功時の出力：

```
Signature Verified Successfully
```

失敗時の出力例（ファイル改ざんや鍵の不一致など）：

```
Signature Verification Failure
```

---

## ダイジェスト署名・署名検証

直接署名 (-rawin) はメッセージ全体をそのまま署名アルゴリズムに渡します。  
これに対し ダイジェスト署名 は、**署名者が事前にメッセージのハッシュ値（ダイジェスト）を計算し、そのハッシュ値に対して署名する** 方式です。

### 直接署名とダイジェスト署名の比較

| 方式                      | OpenSSL オプション    | 入力                             | 用途・特徴                                   |
| ------------------------- | --------------------- | -------------------------------- | -------------------------------------------- |
| 直接署名                  | -rawin                | 元メッセージそのもの             | シンプル。ファイル全体をライブラリが内部処理 |
| ダイジェスト署名（方法A） | -digest sha256        | 元メッセージ（内部でハッシュ化） | OpenSSL 3.5 で推奨。コマンド1つで完結        |
| ダイジェスト署名（方法B） | なし（-rawin も不要） | 事前計算済みのハッシュ値バイナリ | ハッシュ計算を外部で行いたい場合             |

> **FN-DSA (Falcon) のハッシュ処理について**  
> Falcon の仕様（FIPS 206）では、内部的に SHAKE-256 によるハッシュ処理が含まれます。  
> -rawin または -digest を使う場合、OpenSSL がその前処理を担います。  
> 方法B (事前計算済みハッシュを渡す) では、**OpenSSL は追加のハッシュを行わず、渡されたバイト列をそのまま Falcon の内部入力として使用** します。

### 方法A: -digest オプションによるダイジェスト署名 (推奨)

OpenSSL 3.5 では、-digest アルゴリズム名 を指定すると内部で自動的にハッシュ化してから署名します。  
-digest は OpenSSL 3.5 以降 -rawin を暗黙的に含むため、両者を重複指定する必要はありません。

```bash title="署名"
openssl pkeyutl \
    -sign \
    -in message.txt \
    -inkey fndsa_priv.pem \
    -out message_digest.sig \
    -provider default -provider oqsprovider
```

```bash title="署名検証"
openssl pkeyutl \
    -verify \
    -in message.txt \
    -sigfile message_digest.sig \
    -pubin -inkey fndsa_pub.pem \
    -provider default -provider oqsprovider
```

成功時の出力：

```
Signature Verified Successfully
```

---

### 方法B: 事前にハッシュ値を計算して署名 (事前ハッシュ)

ハッシュ計算と FN-DSA 署名を分離したい場合 (例：HSM との連携や、ハッシュ計算済みデータを受け取って署名するワークフロー) に使用します。


```bash title="SHA-256 ハッシュ値をバイナリ形式で出力"
openssl dgst -sha256 -binary message.txt > message.sha256
```

バイナリ内容の確認（16進数表示）：

```bash title="ハッシュ値の確認"
xxd message.sha256
```

#### ステップ 2: ハッシュ値への署名

-rawin も -digest も付けない場合、pkeyutl は入力を「すでにダイジェスト済みのデータ」として扱います。

```bash title="ハッシュ値への署名"
openssl pkeyutl \
    -sign \
    -in message.sha256 \
    -inkey fndsa_priv.pem \
    -out message_prehash.sig \
    -provider default -provider oqsprovider
```

#### ステップ 3: ハッシュ値への署名検証

検証側も同様に、同じアルゴリズムで計算したハッシュ値を渡します。

```bash title="ハッシュ値署名の検証"
# 検証用にも同じハッシュ値を使用（または検証側で独自に再計算）
openssl dgst -sha256 -binary message.txt > message_verify.sha256

openssl pkeyutl \
    -verify \
    -in message_verify.sha256 \
    -sigfile message_prehash.sig \
    -pubin -inkey fndsa_pub.pem \
    -provider default -provider oqsprovider
```

成功時の出力：

```
Signature Verified Successfully
```

> **方法Bの注意点**  
> - 署名時と署名検証時で **必ず同じハッシュアルゴリズム・同じ入力データ** からハッシュ値を生成してください。
> - 方法Aと方法Bで生成した署名ファイルは相互に検証できません (内部の処理フローが異なるため)。
> - 一般的なユースケースでは方法Aの使用を推奨します。方法Bはハッシュ計算の外部管理が必要な場面向けです。

---

## X.509 証明書への応用（参考）

FN-DSA 鍵を使った自己署名証明書の生成例です。TLS やコード署名などへの応用が可能です。

```bash title="自己署名証明書の生成"
openssl req -x509 \
    -new \
    -key fndsa_priv.pem \
    -out fndsa_cert.pem \
    -days 365 \
    -subj "/CN=FN-DSA Test/O=Example Org/C=JP" \
    -provider default -provider oqsprovider
```

```bash title="証明書の内容確認"
openssl x509 -in fndsa_cert.pem -text -noout \
    -provider default -provider oqsprovider
```

---

## 署名方式の選択指針

```
元メッセージを直接渡せる場合
├─ シンプルに署名したい      → 直接署名 (-rawin)
├─ ハッシュ計算込みの署名    → ダイジェスト署名 方法A
└─ ハッシュ計算を外部で行う  → ダイジェスト署名 方法B（事前計算ハッシュを入力）
```

## アルゴリズム選択の指針

| アルゴリズム名   | NIST 名称   | Security Level    | 公開鍵サイズ | 署名サイズ  | 用途・特徴                   |
| ---------------- | ----------- | ----------------- | ------------ | ----------- | ---------------------------- |
| falcon512        | FN-DSA-512  | 1（AES-128 相当） | 897 B        | ～690 B     | 汎用。署名サイズが小さく高速 |
| falcon1024       | FN-DSA-1024 | 5（AES-256 相当） | 1793 B       | ～1330 B    | 高セキュリティ要件向け       |
| falconpadded512  | —           | 1                 | 897 B        | 固定 666 B  | 固定長署名が必要な場面向け   |
| falconpadded1024 | —           | 5                 | 1793 B       | 固定 1280 B | 固定長署名が必要な場面向け   |

> **FN-DSA (Falcon) の実装上の注意**  
> Falcon の署名生成には倍精度浮動小数点演算（IEEE 754）を使用します。  
> 組み込みデバイスや電力・電磁波解析が可能な環境では、サイドチャネル攻撃への対策が必要です。  
> サーバー用途などの一般的な x86/x86_64 環境では liboqs の参照実装で問題ありません。


---

## 参考資料

- [Open Quantum Safe: oqs-provider](https://github.com/open-quantum-safe/oqs-provider)
- [Open Quantum Safe: liboqs](https://github.com/open-quantum-safe/liboqs)
- [oqs-provider USAGE.md](https://github.com/open-quantum-safe/oqs-provider/blob/main/USAGE.md)
- [oqs-provider ALGORITHMS.md](https://github.com/open-quantum-safe/oqs-provider/blob/main/ALGORITHMS.md)
- [OpenSSL 3.5 ドキュメント](https://docs.openssl.org/3.5/)
