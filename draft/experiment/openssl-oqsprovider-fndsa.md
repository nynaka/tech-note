---
title: OpenSSL + oqs-provider による FN-DSA 署名・署名検証
description: FN-DSA による署名・署名検証を例とした OpenSSL + oqs-provider 使用手順です。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

OpenSSL + oqs-provider による FN-DSA 署名・署名検証
===

ここでは、耐量子暗号アルゴリズム **FN-DSA (IPS 206 / Falcon ベース)**を用いたデジタル署名を例に、oqs-provider を使用して openssl コマンドで署名・署名検証する手順を紹介します。 

> **アルゴリズム名について**  
> oqs-provider 内での FN-DSA の識別子は falcon512 (Security Level 1、AES-128 相当) および falcon1024 (Security Level 5、AES-256 相当) です。NIST の標準名 "FN-DSA" と oqs-provider 上の名前が異なる点に注意してください。  
> また、パディング付きバリアントとして falconpadded512 / falconpadded1024 も利用可能です。

---

## 1. 前提条件と環境確認

|              |                                                       |                         |
| :----------- | :---------------------------------------------------- | :---------------------- |
| OS           | Fedora Linux 43                                       | Docker Container        |
| OpenSSL      | 3.5.4                                                 | dnf でインストール      |
| oqs-provider | commit hash: 334f9fc57dc17335f10df561d531eadd8b2ed4d6 | Github のメインブランチ |


### システムアップデートと依存関係のインストール

ビルドに必要なツールとライブラリをインストールします。

```bash
sudo dnf update -y
sudo dnf install -y \
    git cmake gcc gcc-c++ ninja-build \
    openssl-devel python3-pytest \
    pkgconf-pkg-config
```

---

## 2. liboqs のビルドとインストール

oqs-provider は、耐量子暗号の実装ライブラリ liboqs に依存しています。

```bash
cd ~
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

### インストール後の共有ライブラリ設定

```bash
# ダイナミックリンカのキャッシュを更新
sudo ldconfig
```

---

## 3. oqs-provider のビルドとインストール

OpenSSL 3.x で PQC アルゴリズムを使用可能にするプロバイダーをビルドします。

```bash
cd ~
git clone -b main https://github.com/open-quantum-safe/oqs-provider.git
cd oqs-provider
mkdir build && cd build
cmake -GNinja \
    -DCMAKE_BUILD_TYPE=Release \
    -Dliboqs_DIR=/usr/local/lib/cmake/liboqs \
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
>
> lib64 が見つかった場合は -Dliboqs_DIR=/usr/local/lib64/cmake/liboqs に変更してください。

### インストール先の確認

```bash
# oqsprovider.so の場所を確認する
find /usr/local -name "oqsprovider.so" 2>/dev/null
# 一般的なパス候補:
#   /usr/local/lib64/ossl-modules/oqsprovider.so
#   /usr/local/lib/ossl-modules/oqsprovider.so
```

---

## 4. OpenSSL プロバイダーの設定

### 4-1. 環境変数による一時設定（動作確認・テスト向け）

現在のシェルセッションのみで有効にする場合：

```bash
# oqsprovider.so のあるディレクトリを指定（前節で確認したパス）
export OPENSSL_MODULES=/usr/local/lib64/ossl-modules
```

### 4-2. openssl.cnf への永続設定（推奨）

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
# module = /usr/local/lib64/ossl-modules/oqsprovider.so
```

> **重要**: default プロバイダーは PEM エンコード / デコードなど基本機能を担います。  
> oqsprovider だけを有効化すると鍵の読み書きに失敗するため、必ず両方を有効にしてください。

### 4-3. プロバイダーの読み込み確認

```bash
openssl list -providers
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
    version: 0.x.x
    status: active
```

### 4-4. FN-DSA アルゴリズムの確認

```bash
openssl list -signature-algorithms -provider oqsprovider | grep -i falcon
```

以下のようなエントリが表示されれば正常です：

```
{ 1.3.9999.3.11, falcon512 } @ oqsprovider
{ 1.3.9999.3.13, falcon1024 } @ oqsprovider
{ 1.3.9999.3.16, falconpadded512 } @ oqsprovider
{ 1.3.9999.3.18, falconpadded1024 } @ oqsprovider
```

> **備考**: openssl.cnf に永続設定済みの場合は -provider oqsprovider フラグは省略できます。

---

## 5. 直接署名・署名検証

以下では falcon512（NIST Security Level 1、AES-128 相当）を使用した例を示します。  
より高いセキュリティが必要な場合は falcon1024（Security Level 5、AES-256 相当）に置き換えてください。

### コマンドの -provider オプションについて

openssl.cnf で永続設定している場合は、各コマンドの -provider オプションは省略できます。  
**設定していない場合**は、すべてのコマンドに以下のオプションを付加してください：

```
-provider default -provider oqsprovider
```

default を同時に指定するのは、PEM のエンコード / デコードに default プロバイダーが必要なためです。

---

### ステップ 1: 秘密鍵の生成

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

### ステップ 2: 公開鍵の抽出

```bash
openssl pkey \
    -in fndsa_priv.pem \
    -pubout -out fndsa_pub.pem \
    -provider default -provider oqsprovider
```

### ステップ 3: 署名対象ファイルの作成

```bash
echo "This is a post-quantum test message." > message.txt
```

### ステップ 4: ファイルへの署名

```bash
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

### ステップ 5: 署名の検証

```bash
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

## 6. ダイジェスト署名・署名検証

前章の「直接署名 (-rawin)」はメッセージ全体をそのまま署名アルゴリズムに渡します。これに対し「ダイジェスト署名」は、**署名者が事前にメッセージのハッシュ値（ダイジェスト）を計算し、そのハッシュ値に対して署名する** 方式です。

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

---

### 方法A: -digest オプションによるダイジェスト署名 (推奨)

OpenSSL 3.5 では、-digest アルゴリズム名 を指定すると内部で自動的にハッシュ化してから署名します。  
-digest は OpenSSL 3.5 以降 -rawin を暗黙的に含むため、両者を重複指定する必要はありません。

#### 署名

```bash
openssl pkeyutl \
    -sign \
    -in message.txt \
    -inkey fndsa_priv.pem \
    -out message_digest.sig \
    -digest sha256 \
    -provider default -provider oqsprovider
```

#### 検証

```bash
openssl pkeyutl \
    -verify \
    -in message.txt \
    -sigfile message_digest.sig \
    -pubin -inkey fndsa_pub.pem \
    -digest sha256 \
    -provider default -provider oqsprovider
```

成功時の出力：

```
Signature Verified Successfully
```

> **注意**: 署名時と検証時に**同じダイジェストアルゴリズム** (ここでは sha256) を指定する必要があります。  
> 異なるアルゴリズムを指定すると検証に失敗します。

---

### 方法B: 事前にハッシュ値を計算して署名 (外部ハッシュ)

ハッシュ計算とFN-DSA署名を分離したい場合 (例：HSM との連携や、ハッシュ計算済みデータを受け取って署名するワークフロー) に使用します。

#### ステップ 1: ダイジェスト (ハッシュ値) の計算

```bash
# SHA-256 ハッシュ値をバイナリ形式で出力
openssl dgst -sha256 -binary message.txt > message.sha256
```

バイナリ内容の確認（16進数表示）：

```bash
xxd message.sha256
# 32バイト（256ビット）のハッシュ値が表示される
```

#### ステップ 2: ハッシュ値への署名

-rawin も -digest も付けない場合、pkeyutl は入力を「すでにダイジェスト済みのデータ」として扱います。

```bash
openssl pkeyutl \
    -sign \
    -in message.sha256 \
    -inkey fndsa_priv.pem \
    -out message_prehash.sig \
    -provider default -provider oqsprovider
```

#### ステップ 3: ハッシュ値への署名検証

検証側も同様に、同じアルゴリズムで計算したハッシュ値を渡します。

```bash
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

### 署名方式の選択指針

```
元メッセージを直接渡せる場合
├─ シンプルに署名したい         → 直接署名（-rawin）        ← セクション5
├─ ハッシュアルゴリズムを明示   → ダイジェスト署名 方法A（-digest sha256）
└─ ハッシュ計算を外部で行う     → ダイジェスト署名 方法B（事前計算ハッシュを入力）
```

---

## 7. 動作確認：改ざん検知テスト

署名後にファイルを改ざんして検証が失敗することを確認します。

```bash
# 改ざんされたファイルを作成
echo "This message has been tampered." > message_tampered.txt

# 改ざんファイルに対して元の署名で検証（失敗するはず）
openssl pkeyutl \
    -verify \
    -in message_tampered.txt \
    -sigfile message.sig \
    -pubin -inkey fndsa_pub.pem \
    -rawin \
    -provider default -provider oqsprovider
# → "Signature Verification Failure" が表示されること
```

---

## 8. X.509 証明書への応用（参考）

FN-DSA 鍵を使った自己署名証明書の生成例です。TLS やコード署名などへの応用が可能です。

```bash
# 自己署名証明書の生成
openssl req -x509 \
    -new \
    -key fndsa_priv.pem \
    -out fndsa_cert.pem \
    -days 365 \
    -subj "/CN=FN-DSA Test/O=Example Org/C=JP" \
    -provider default -provider oqsprovider

# 証明書の内容確認
openssl x509 -in fndsa_cert.pem -text -noout \
    -provider default -provider oqsprovider
```

---

## 9. アルゴリズム選択の指針

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

## トラブルシューティング

### No such algorithm エラー

```
Error: No such algorithm
```

**原因**: OPENSSL_MODULES が未設定、または oqsprovider.so のパスが誤っている。

**対処**:

```bash
# oqsprovider.so の実際のパスを確認
find /usr -name "oqsprovider.so" 2>/dev/null
find /usr/local -name "oqsprovider.so" 2>/dev/null

# 確認したパスのディレクトリを設定
export OPENSSL_MODULES=<上記で確認したディレクトリ>
```

### No encoders were found エラー

```
Error writing key(s): No encoders were found.
```

**原因**: default プロバイダーが読み込まれていない。

**対処**: コマンドに -provider default を追加する (oqsprovider と同時指定)。

### cmake 時に liboqs が見つからない

**原因**: liboqs_DIR のパスが誤っている。

**対処**:

```bash
find /usr/local -name "liboqsConfig.cmake" 2>/dev/null
# → 見つかったパスを -Dliboqs_DIR に指定
```

### oqsprovider のバージョンと OpenSSL のバージョン不整合

OpenSSL 3.5.0 以降では ML-KEM / ML-DSA / SLH-DSA がデフォルトプロバイダーにネイティブ実装されており、oqs-provider 0.9.0 以降ではこれらは oqs-provider 側で無効化されます。  
**FN-DSA (Falcon) は引き続き oqs-provider 経由でのみ提供** されます。oqs-provider は **0.9.0 以降** の使用を推奨します。

---

## 参考資料

- [Open Quantum Safe: oqs-provider](https://github.com/open-quantum-safe/oqs-provider)
- [Open Quantum Safe: liboqs](https://github.com/open-quantum-safe/liboqs)
- [oqs-provider USAGE.md](https://github.com/open-quantum-safe/oqs-provider/blob/main/USAGE.md)
- [oqs-provider ALGORITHMS.md](https://github.com/open-quantum-safe/oqs-provider/blob/main/ALGORITHMS.md)
- [OpenSSL 3.5 ドキュメント](https://docs.openssl.org/3.5/)
