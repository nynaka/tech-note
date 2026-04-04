---
title: openssl コマンドの署名
description: openssl コマンドを使用したデジタル署名の実行例です。
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

openssl コマンドの署名
===

## 動作環境

| 項目    | 内容                       |
| :------ | :------------------------- |
| OS      | Fedora Linux 43            |
| OpenSSL | 3.5.4 (dnf でインストール) |

> **注意:** OpenSSL 3.5.x 系の具体的なパッチバージョンおよびリリース日は、ディストリビューションのパッケージングによって異なる場合があります。**openssl version** コマンドで実際の環境を確認してください。

---

## 概要

ここでは、openssl コマンドを使用して以下の6種類のデジタル署名アルゴリズムについて、キーペア生成・署名・署名検証 を行います。  
各アルゴリズムについて、直接署名（メッセージをそのまま渡す）と ダイジェスト署名（ハッシュを計算してから渡す）の両方を記載しています。

| アルゴリズム    | 種別       | 標準規格    | 特徴                          |
| --------------- | ---------- | ----------- | ----------------------------- |
| RSA PKCS#1 v1.5 | 従来型     | PKCS#1      | 広く普及。決定論的署名        |
| RSA-PSS         | 従来型     | PKCS#1 v2.1 | ランダム化署名。RSAの推奨方式 |
| ECDSA           | 従来型     | ANSI X9.62  | 楕円曲線。RSAより短い鍵長     |
| EdDSA (Ed25519) | 従来型     | RFC 8032    | 高速・安全。決定論的署名      |
| ML-DSA          | **耐量子** | FIPS 204    | NIST標準の格子ベース署名      |
| SLH-DSA         | **耐量子** | FIPS 205    | NIST標準のハッシュベース署名  |

> **openssl dgst コマンドについて:** OpenSSL 3.x では、**openssl dgst -sign / -verify** ではなく、**openssl pkeyutl** または **openssl sign / openssl verify** コマンドが推奨されています。

### 直接署名とダイジェスト署名の違い

| 方式             | 処理の流れ                                      | ユースケース                         |
| ---------------- | ----------------------------------------------- | ------------------------------------ |
| 直接署名         | メッセージ → （内部でハッシュ） → 署名          | 通常の用途。シンプルで一般的         |
| ダイジェスト署名 | メッセージ → ハッシュ計算 → ダイジェスト → 署名 | ハッシュを別目的でも利用する場合など |

### 署名アルゴリズムと署名方式

| アルゴリズム    | 直接署名 | ダイジェスト署名                                          |
| --------------- | -------- | --------------------------------------------------------- |
| RSA PKCS#1 v1.5 | ✅       | ✅                                                        |
| RSA-PSS         | ✅       | ✅                                                        |
| ECDSA           | ✅       | ✅                                                        |
| EdDSA (Ed25519) | ✅       | ❌ 仕様上不可（PureEdDSA の設計上、メッセージ全体が必要） |
| ML-DSA          | ✅       | ❌ 仕様上不可（HashML-DSA バリアントはCLI非対応）         |
| SLH-DSA         | ✅       | ❌ 仕様上不可（HashSLH-DSA バリアントはCLI非対応）        |

---

## 事前準備

### OpenSSLバージョン確認

```bash
$ openssl version
OpenSSL 3.5.4 30 Sep 2025 (Library: OpenSSL 3.5.4 30 Sep 2025)
```

ML-DSA、SLH-DSA を使用するには OpenSSL 3.5 以降 が推奨です（OpenSSL 3.3 から使用できるそうです）。

### 作業ディレクトリとメッセージファイルの準備

```bash
mkdir -p crypto-demo
cd crypto-demo

# 署名対象のメッセージファイルを作成
echo "Hello, OpenSSL Signature Demo!" > message.txt
```

---

## 1. RSA PKCS#1 v1.5

RSA PKCS#1 v1.5 は最も広く使われているRSA署名方式です。  
署名時にハッシュ値へ DER 形式の AlgorithmIdentifier プレフィックスを付与した後、PKCS#1 v1.5 パディング（0x00 0x01 0xFF...0xFF 0x00 || DigestInfo）を適用します。  
パディングが決定論的であるため、同じメッセージ・同じ鍵に対して常に同じ署名値を生成します。

### 1-1. 秘密鍵の生成

```bash
openssl genpkey -algorithm RSA \
    -pkeyopt rsa_keygen_bits:4096 \
    -out rsa_pkcs_private.pem
```

| オプション                    | 説明                                                       |
| ----------------------------- | ---------------------------------------------------------- |
| -algorithm RSA                | RSAアルゴリズムを指定                                      |
| -pkeyopt rsa_keygen_bits:4096 | 鍵長（最小推奨: 2048bit、長期利用には 3072bit 以上を推奨） |
| -out rsa_pkcs_private.pem     | 出力ファイル名                                             |

**生成した秘密鍵の内容確認:**

```bash
openssl pkey -in rsa_pkcs_private.pem -text -noout
```

### 1-2. 公開鍵の抽出

```bash
openssl pkey -in rsa_pkcs_private.pem -pubout -out rsa_pkcs_public.pem
```

### 1-3. 署名の生成・検証

RSA PKCS#1 v1.5 は **直接署名** と **ダイジェスト署名** の両方をサポートします。

#### 方法1: 直接署名（メッセージをそのまま渡す）

署名コマンドがメッセージのハッシュ計算と署名を一括して行います。

```bash
# 署名生成
openssl dgst -sha256 \
    -sign rsa_pkcs_private.pem \
    -out rsa_pkcs_sig_direct.bin \
    message.txt

# 署名検証
openssl dgst -sha256 \
    -verify rsa_pkcs_public.pem \
    -signature rsa_pkcs_sig_direct.bin \
    message.txt
```

| オプション | 説明                                                                  |
| ---------- | --------------------------------------------------------------------- |
| -sha256    | ハッシュアルゴリズム（SHA-256）。他に -sha384, -sha512 なども指定可能 |
| -sign      | 秘密鍵を指定して署名生成                                              |
| -verify    | 公開鍵を指定して署名検証                                              |

**出力（検証成功時）:** Verified OK


#### 方法2: ダイジェスト署名（事前にハッシュを計算してから署名）

メッセージのハッシュを先に計算しておき、そのダイジェスト値だけを署名します。  
送信するデータが大きい場合や、ハッシュを別用途でも使いたい場合に有効です。

```bash
# (1) メッセージのダイジェスト（ハッシュ値）を計算
openssl dgst -sha256 -binary message.txt > rsa_pkcs_digest.bin

# (2) ダイジェストに対して署名を生成
openssl pkeyutl -sign \
    -inkey rsa_pkcs_private.pem \
    -pkeyopt digest:sha256 \
    -in rsa_pkcs_digest.bin \
    -out rsa_pkcs_sig_digest.bin

# (3) ダイジェストに対して署名を検証
openssl pkeyutl -verify \
    -pubin \
    -inkey rsa_pkcs_public.pem \
    -pkeyopt digest:sha256 \
    -in rsa_pkcs_digest.bin \
    -sigfile rsa_pkcs_sig_digest.bin
```

| オプション             | 説明                                                                                      |
| ---------------------- | ----------------------------------------------------------------------------------------- |
| -binary                | ダイジェストをバイナリ形式で出力（hex なし）                                              |
| -pkeyopt digest:sha256 | 入力がSHA-256ダイジェストであることを指定。① のハッシュアルゴリズムと一致させる必要がある |
| -pubin                 | -inkey で指定するファイルが公開鍵であることを示す                                         |

**出力（検証成功時）:** Signature Verified Successfully

**署名サイズ:** 512バイト（4096 bit ÷ 8）※ 両方法で同一


---

## 2. RSA-PSS

RSA-PSS（Probabilistic Signature Scheme）は、ランダムなソルトを使用するため同じメッセージでも毎回異なる署名値を生成します。  
RSA PKCS#1 v1.5 より安全性の証明が強固であり、現在推奨される RSA 署名方式です。

### 2-1. 秘密鍵の生成

```bash
openssl genpkey -algorithm RSA-PSS \
    -pkeyopt rsa_keygen_bits:4096 \
    -pkeyopt rsa_pss_keygen_md:sha256 \
    -pkeyopt rsa_pss_keygen_mgf1_md:sha256 \
    -pkeyopt rsa_pss_keygen_saltlen:32 \
    -out rsa_pss_private.pem
```

| オプション                    | 説明                                                                               |
| ----------------------------- | ---------------------------------------------------------------------------------- |
| -algorithm RSA-PSS            | RSA-PSSアルゴリズムを指定。鍵の用途を PSS に限定するメタ情報が鍵ファイルに含まれる |
| rsa_pss_keygen_md:sha256      | 使用ハッシュ関数を鍵に埋め込む（この鍵は指定外のハッシュでの署名を拒否する）       |
| rsa_pss_keygen_mgf1_md:sha256 | MGF1（マスク生成関数）に使用するハッシュ関数。通常はメインハッシュと同じにする     |
| rsa_pss_keygen_saltlen:32     | ソルト長（バイト）。SHA-256 のダイジェスト長 (32) に合わせるのが一般的             |

> **注意:** -algorithm RSA-PSS で生成した鍵は RSA-PSS 専用の制約付き鍵です。プレーンな RSA 鍵（-algorithm RSA）に -sigopt rsa_padding_mode:pss を組み合わせる方法とは鍵の種類が異なります。

### 2-2. 公開鍵の抽出

```bash
openssl pkey -in rsa_pss_private.pem -pubout -out rsa_pss_public.pem
```

### 2-3. 署名の生成・検証

RSA-PSS も **直接署名** と **ダイジェスト署名** の両方をサポートします。

#### 方法1: 直接署名

```bash
# 署名生成
openssl dgst -sha256 \
    -sigopt rsa_padding_mode:pss \
    -sigopt rsa_pss_saltlen:32 \
    -sign rsa_pss_private.pem \
    -out rsa_pss_sig_direct.bin \
    message.txt

# 署名検証
openssl dgst -sha256 \
    -sigopt rsa_padding_mode:pss \
    -sigopt rsa_pss_saltlen:32 \
    -verify rsa_pss_public.pem \
    -signature rsa_pss_sig_direct.bin \
    message.txt
```

| オプション                   | 説明                                                                                |
| ---------------------------- | ----------------------------------------------------------------------------------- |
| -sigopt rsa_padding_mode:pss | PSS パディングを指定                                                                |
| -sigopt rsa_pss_saltlen:32   | ソルト長（バイト）。-1 は hashlen と同じ長さ、-2 は最大ソルト長（検証時は自動検出） |

**出力（検証成功時）:** Verified OK

#### 方法2: ダイジェスト署名（事前にハッシュを計算してから署名）

```bash
# (1) メッセージのダイジェストを計算
openssl dgst -sha256 -binary message.txt > rsa_pss_digest.bin

# (2) ダイジェストに対して署名を生成
openssl pkeyutl -sign \
    -inkey rsa_pss_private.pem \
    -pkeyopt digest:sha256 \
    -pkeyopt rsa_padding_mode:pss \
    -pkeyopt rsa_pss_saltlen:32 \
    -in rsa_pss_digest.bin \
    -out rsa_pss_sig_digest.bin

# (3) ダイジェストに対して署名を検証
openssl pkeyutl -verify \
    -pubin \
    -inkey rsa_pss_public.pem \
    -pkeyopt digest:sha256 \
    -pkeyopt rsa_padding_mode:pss \
    -pkeyopt rsa_pss_saltlen:32 \
    -in rsa_pss_digest.bin \
    -sigfile rsa_pss_sig_digest.bin
```

**出力（検証成功時）:** Signature Verified Successfully

**署名サイズ:** 512バイト（4096 bit ÷ 8）※両方法で同一。ソルトはランダムだが最終的な署名長は固定


---

## 3. ECDSA

ECDSA（Elliptic Curve Digital Signature Algorithm）は楕円曲線暗号を用いた署名方式です。  
RSAより短い鍵長で同等の安全性を実現します。署名値は内部でランダムな一時鍵（ノンス）を使用するため、同じメッセージでも毎回異なる署名値を生成します。

### 3-1. 秘密鍵の生成

```bash
openssl genpkey -algorithm EC \
    -pkeyopt ec_paramgen_curve:P-256 \
    -out ecdsa_private.pem
```

| オプション              | 説明                                                |
| ----------------------- | --------------------------------------------------- |
| -algorithm EC           | 楕円曲線暗号を指定                                  |
| ec_paramgen_curve:P-256 | NIST P-256曲線（secp256r1）。他に P-384, P-521 など |

**主な曲線の比較:**

| 曲線              | 鍵長   | セキュリティ強度 |
| ----------------- | ------ | ---------------- |
| P-256 (secp256r1) | 256bit | 128bit相当       |
| P-384 (secp384r1) | 384bit | 192bit相当       |
| P-521 (secp521r1) | 521bit | 260bit相当       |

### 3-2. 公開鍵の抽出

```bash
openssl pkey -in ecdsa_private.pem -pubout -out ecdsa_public.pem
```

### 3-3. 署名の生成・検証

ECDSA も **直接署名** と **ダイジェスト署名** の両方をサポートします。

#### 方法1: 直接署名

```bash
# 署名生成
openssl dgst -sha256 \
    -sign ecdsa_private.pem \
    -out ecdsa_sig_direct.bin \
    message.txt

# 署名検証
openssl dgst -sha256 \
    -verify ecdsa_public.pem \
    -signature ecdsa_sig_direct.bin \
    message.txt
```

**出力（検証成功時）:** Verified OK

**署名サイズ:** 約71〜72バイト（P-256 + SHA-256 の場合）  
ECDSA の署名値 (r, s) は各 32バイトの整数2つを DER 形式の ASN.1 SEQUENCE に格納します。r や s の最上位ビットが 1 の場合は負数と解釈されないよう 0x00 パディングが付くため、バイト数が1バイト前後します（最小 70 バイト〜最大 72 バイト程度）。

#### 方法2: ダイジェスト署名（事前にハッシュを計算してから署名）

```bash
# (1) メッセージのダイジェストを計算
openssl dgst -sha256 -binary message.txt > ecdsa_digest.bin

# (2) ダイジェストに対して署名を生成
openssl pkeyutl -sign \
    -inkey ecdsa_private.pem \
    -pkeyopt digest:sha256 \
    -in ecdsa_digest.bin \
    -out ecdsa_sig_digest.bin

# (3) ダイジェストに対して署名を検証
openssl pkeyutl -verify \
    -pubin \
    -inkey ecdsa_public.pem \
    -pkeyopt digest:sha256 \
    -in ecdsa_digest.bin \
    -sigfile ecdsa_sig_digest.bin
```

**出力（検証成功時）:** Signature Verified Successfully

#### ASN.1 構造の確認（参考）

ECDSA 署名のバイナリ構造を確認したい場合:

```bash
openssl asn1parse -in ecdsa_sig_direct.bin -inform DER
```


---

## 4. EdDSA (Ed25519)

EdDSA（Edwards-curve Digital Signature Algorithm）はツイステッドエドワーズ曲線 Ed25519 を使用する署名方式です。  
ハッシュ関数として SHA-512 を内部に組み込んでいるため、署名時にハッシュアルゴリズムを個別に指定する必要はありません。  
乱数ではなく秘密鍵とメッセージから決定論的にノンスを生成する設計のため、同じ入力に対して常に同じ署名値を生成します（サイドチャネル攻撃への耐性向上にも寄与）。

### 4-1. 秘密鍵の生成

```bash
openssl genpkey -algorithm Ed25519 -out eddsa_private.pem
```

> Ed448 を使用する場合は `-algorithm Ed448` を指定します（SHAKE-256 ベース。セキュリティ強度: 224bit相当）。Ed448 も同じ `pkeyutl` コマンドで署名・検証できます。

### 4-2. 公開鍵の抽出

```bash
openssl pkey -in eddsa_private.pem -pubout -out eddsa_public.pem
```

### 4-3. 署名の生成・検証

EdDSA では `pkeyutl` を使用します（`dgst` コマンドは使用不可）。

> **⚠️ ダイジェスト署名は仕様上サポートされていません**
>
> Ed25519（および Ed448）は **PureEdDSA** と呼ばれる設計であり、内部でメッセージ全体に対して2段階のハッシュ処理（`SHA-512(nonce_seed || message)` によるノンス生成と `SHA-512(R || public_key || message)` による検証値計算）を行います。外部で事前計算したハッシュ値を入力として受け付けることができません。
>
> 実際に試すと OpenSSL は次のエラーを返します:
> ```
> error: invalid digest: Explicit digest not allowed with EdDSA operations
> ```
>
> なお、RFC 8032 には事前ハッシュを利用する「Ed25519ph」（pre-hash variant）も定義されていますが、OpenSSL の CLI はこのバリアントを公開していません。

#### 直接署名のみ（メッセージ全体を渡す）

```bash
# 署名生成
openssl pkeyutl -sign \
    -inkey eddsa_private.pem \
    -in message.txt \
    -out eddsa_sig.bin

# 署名検証
openssl pkeyutl -verify \
    -pubin \
    -inkey eddsa_public.pem \
    -in message.txt \
    -sigfile eddsa_sig.bin
```

**署名サイズ:** 64バイト (固定長)

Ed25519 の署名値は (R, S) の 2 つの 32 バイト値を連結したもので、DER エンコードなしのコンパクトな固定長フォーマットです。

**出力（検証成功時）:** Signature Verified Successfully


---

## 5. ML-DSA（FIPS 204）

ML-DSA (Module-Lattice-Based Digital Signature Algorithm) は NIST FIPS 204 として標準化された**耐量子計算機署名**方式です。  
CRYSTALS-Dilithium を原型とする格子暗号 (Module-LWE / Module-SIS 問題の困難性) に基づきます。内部では SHAKE-128 / SHAKE-256 を使用します。

### パラメータセットの比較

| パラメータ | セキュリティレベル | 公開鍵サイズ | 秘密鍵サイズ | 署名サイズ  |
| ---------- | ------------------ | ------------ | ------------ | ----------- |
| ML-DSA-44  | NIST Level 2       | 1312 バイト  | 2528 バイト  | 2420 バイト |
| ML-DSA-65  | NIST Level 3       | 1952 バイト  | 4032 バイト  | 3309 バイト |
| ML-DSA-87  | NIST Level 5       | 2592 バイト  | 4896 バイト  | 4627 バイト |

> NIST Level 2 は AES-128 相当、Level 3 は AES-192 相当、Level 5 は AES-256 相当のセキュリティ強度です。

### 5-1. 秘密鍵の生成

```bash
openssl genpkey -algorithm ML-DSA-65 -out mldsa_private.pem
```

### 5-2. 公開鍵の抽出

```bash
openssl pkey -in mldsa_private.pem -pubout -out mldsa_public.pem
```

### 5-3. 署名の生成・検証

> **⚠️ ダイジェスト署名は仕様上サポートされていません**
>
> ML-DSA（FIPS 204）は、メッセージ全体を入力として受け取り、内部で SHAKE-256 を用いたハッシュ処理を行います。外部で事前計算したダイジェストを入力として署名することはできません。
>
> 実際に試すと OpenSSL は次のエラーを返します:
> ```
> error: invalid digest: Explicit digest not supported for ML-DSA operations
> ```
>
> なお、FIPS 204 の Appendix には「HashML-DSA」バリアントも定義されていますが、OpenSSL の CLI はこのバリアントを公開していません。

#### 直接署名のみ（メッセージ全体を渡す）

```bash
# 署名生成
openssl pkeyutl -sign \
    -inkey mldsa_private.pem \
    -in message.txt \
    -out mldsa_sig.bin

# 署名検証
openssl pkeyutl -verify \
    -pubin \
    -inkey mldsa_public.pem \
    -in message.txt \
    -sigfile mldsa_sig.bin
```

**署名サイズ:** 3309バイト（ML-DSA-65の場合）

**出力（検証成功時）:** Signature Verified Successfully


---

## 6. SLH-DSA（FIPS 205）

SLH-DSA (Stateless Hash-Based Digital Signature Algorithm) は NIST FIPS 205 として標準化された **耐量子計算機署名** 方式です。  
SPHINCS+ を原型とする、ハッシュ関数のみを組み合わせたマークルツリー構造であり、格子暗号とは異なる数学的基盤を持ちます。  
長期的な安全性の根拠がシンプルである一方、署名サイズが大きいのが特徴です。

### パラメータセットの比較

| パラメータ         | ハッシュ | 種別  | セキュリティ | 公開鍵    | 秘密鍵     | 署名サイズ   |
| ------------------ | -------- | ----- | ------------ | --------- | ---------- | ------------ |
| SLH-DSA-SHA2-128s  | SHA-2    | small | NIST Level 1 | 32 バイト | 64 バイト  | 7856 バイト  |
| SLH-DSA-SHA2-128f  | SHA-2    | fast  | NIST Level 1 | 32 バイト | 64 バイト  | 17088 バイト |
| SLH-DSA-SHA2-192s  | SHA-2    | small | NIST Level 3 | 48 バイト | 96 バイト  | 16224 バイト |
| SLH-DSA-SHA2-192f  | SHA-2    | fast  | NIST Level 3 | 48 バイト | 96 バイト  | 35664 バイト |
| SLH-DSA-SHA2-256s  | SHA-2    | small | NIST Level 5 | 64 バイト | 128 バイト | 29792 バイト |
| SLH-DSA-SHA2-256f  | SHA-2    | fast  | NIST Level 5 | 64 バイト | 128 バイト | 49856 バイト |
| SLH-DSA-SHAKE-128s | SHAKE    | small | NIST Level 1 | 32 バイト | 64 バイト  | 7856 バイト  |
| SLH-DSA-SHAKE-128f | SHAKE    | fast  | NIST Level 1 | 32 バイト | 64 バイト  | 17088 バイト |

> s (small) は署名サイズが小さいが処理が遅く (特に署名生成が低速)、f (fast) は処理が速いが署名が大きくなります。用途に応じて選択してください。

### 6-1. 秘密鍵の生成

```bash
openssl genpkey -algorithm SLH-DSA-SHA2-128s -out slhdsa_private.pem
```

### 6-2. 公開鍵の抽出

```bash
openssl pkey -in slhdsa_private.pem -pubout -out slhdsa_public.pem
```

### 6-3. 署名の生成・検証

> **⚠️ ダイジェスト署名は仕様上サポートされていません**
>
> SLH-DSA (FIPS 205) はハッシュ関数 (SHA-2 または SHAKE) のみを用いた構造ですが、署名処理はメッセージ全体を内部で処理する設計になっており、外部で事前計算したダイジェストを署名の入力として渡すことはできません。
>
> 実際に試すと OpenSSL は次のエラーを返します:
> ```
> error: command not supported (digest:sha256 parameter)
> ```
>
> FIPS 205 には「HashSLH-DSA」バリアントも定義されていますが、OpenSSL の CLI はこのバリアントを公開していません。

#### 直接署名のみ (メッセージ全体を渡す)

```bash
# 署名生成
openssl pkeyutl -sign \
    -inkey slhdsa_private.pem \
    -in message.txt \
    -out slhdsa_sig.bin

# 署名検証
openssl pkeyutl -verify \
    -pubin \
    -inkey slhdsa_public.pem \
    -in message.txt \
    -sigfile slhdsa_sig.bin
```

**署名サイズ:** 7856 バイト（SLH-DSA-SHA2-128s の場合）

**出力（検証成功時）:** Signature Verified Successfully


---

## アルゴリズム比較

### 動作確認結果サマリー

| アルゴリズム      | 直接署名コマンド                          | ダイジェスト署名                                                | 署名サイズ    | 検証結果                                      |
| ----------------- | ----------------------------------------- | --------------------------------------------------------------- | ------------- | --------------------------------------------- |
| RSA PKCS#1 v1.5   | dgst -sha256 -sign                        | ✅ pkeyutl -pkeyopt digest:sha256                               | 256 バイト    | Verified OK                                   |
| RSA-PSS           | dgst -sha256 -sigopt rsa_padding_mode:pss | ✅ pkeyutl -pkeyopt digest:sha256 -pkeyopt rsa_padding_mode:pss | 256 バイト    | Verified OK                                   |
| ECDSA             | dgst -sha256 -sign                        | ✅ pkeyutl -pkeyopt digest:sha256                               | 70〜72 バイト | Verified OK / Signature Verified Successfully |
| EdDSA (Ed25519)   | pkeyutl -sign                             | ❌ 不可（PureEdDSA）                                            | 64 バイト     | Signature Verified Successfully               |
| ML-DSA-65         | pkeyutl -sign                             | ❌ 不可（HashML-DSA はCLI非対応）                               | 3309 バイト   | Signature Verified Successfully               |
| SLH-DSA-SHA2-128s | pkeyutl -sign                             | ❌ 不可（HashSLH-DSA はCLI非対応）                              | 7856 バイト   | Signature Verified Successfully               |

### コマンド体系

| 方式                     | 署名/検証コマンド | ダイジェスト署名                       | 備考                                       |
| ------------------------ | ----------------- | -------------------------------------- | ------------------------------------------ |
| RSA PKCS, RSA-PSS, ECDSA | openssl dgst      | ✅ openssl pkeyutl -pkeyopt digest:XXX | -sha256 などハッシュ指定が必要             |
| EdDSA, ML-DSA, SLH-DSA   | openssl pkeyutl   | ❌ 仕様上不可                          | ハッシュ指定不要（アルゴリズム内部で処理） |

---

## 署名ファイルの検査

生成した署名ファイルやバイナリを確認する際に便利なコマンドを記載します。

### バイナリサイズの確認

```bash
wc -c *.bin
```

### ASN.1 構造のパース（ECDSA / RSA の署名に有効）

```bash
# ECDSA 署名の (r, s) 値を確認
openssl asn1parse -in ecdsa_sig_direct.bin -inform DER

# RSA 署名（生のバイト列。ASN.1 構造なし）
xxd rsa_pkcs_sig_direct.bin | head
```

### ダイジェストファイルのサイズ確認

```bash
wc -c *_digest.bin
# SHA-256: 32バイト、SHA-384: 48バイト、SHA-512: 64バイト
```

---

## セキュリティに関する注意事項

1. **秘密鍵の保護**: 秘密鍵ファイル (*_private.pem) は適切なパーミション（600）で管理し、外部に公開しないでください。

    ```bash
    chmod 600 *_private.pem
    ```

2. **鍵長の選択**: RSA は最小 2048 bit、推奨 3072 bit 以上を使用してください。2048 bit は 2030 年頃まで有効と見込まれています。

3. **ハッシュアルゴリズムの選択**: SHA-1 は署名用途では非推奨です（衝突攻撃が実証済み）。SHA-256 以上を使用してください。

4. **耐量子暗号への移行**: 量子コンピュータの実用化に備え、長期保護が必要なシステムでは ML-DSA または SLH-DSA への移行を検討してください。

5. **アルゴリズム選択指針**:

    | 用途                        | 推奨アルゴリズム                       |
    | --------------------------- | -------------------------------------- |
    | 短期利用 (〜2030年)         | ECDSA P-256 / EdDSA Ed25519            |
    | 中長期利用 (2030年以降)     | ECDSA P-384 / RSA 3072bit以上          |
    | 耐量子要件 (バランス型)     | ML-DSA-65                              |
    | 耐量子要件 (保守的・高保証) | SLH-DSA-SHA2-128s (または 192s / 256s) |

6. **テスト環境と本番環境の分離**: 本ドキュメントのコマンド例はデモ用です。本番環境では HSM (ハードウェアセキュリティモジュール) や適切な鍵管理基盤の利用を検討してください。

---

## 参考情報

- [NIST FIPS 186-5 (DSA/ECDSA/EdDSA)](https://csrc.nist.gov/publications/detail/fips/186/5/final)
- [NIST FIPS 204 (ML-DSA)](https://csrc.nist.gov/publications/detail/fips/204/final)
- [NIST FIPS 205 (SLH-DSA)](https://csrc.nist.gov/publications/detail/fips/205/final)
- [RFC 8032 (EdDSA: Ed25519, Ed448)](https://www.rfc-editor.org/rfc/rfc8032)
- [OpenSSL man pages (3.x)](https://www.openssl.org/docs/man3.5/)
- [OpenSSL genpkey(1)](https://www.openssl.org/docs/man3.5/man1/openssl-genpkey.html)
- [OpenSSL pkeyutl(1)](https://www.openssl.org/docs/man3.5/man1/openssl-pkeyutl.html)
