---
title: 暗号アルゴリズムの性能測定
description: ここでは ECDSA secp256r1 (P-256) を例に、暗号アルゴリズムの性能測定を行います
#sidebar_position: 0
#id: home
#slug: /my-custom-url
---

暗号アルゴリズムの性能測定
===

ここでは ECDSA secp256r1 (P-256) を例に、暗号アルゴリズムの性能測定実施例を紹介します。

- **測定ツール**
    - **SUPERCOP**: 処理性能を CPU サイクル数で測定する
    - **Valgrind (Massif)**: ヒープメモリのピーク使用量を測定する

- **測定項目**
    - **鍵ペア生成** (`keypair` / `keygen`)
    - **署名生成** (`sign`)
    - **署名検証** (`verify` / `open`)



## 実行環境準備

Ubuntu Linux 24.04、26.04 で下記のパッケージをインストールしておきます。

```bash
sudo apt install -y build-essential git curl python3 \
    libssl-dev pkg-config make autoconf automake \
    gawk valgrind
```

### SUPERCOP の入手と初期化

- [開発元サイト](https://bench.cr.yp.to/supercop.html) からダウンロードする手順

    ```bash
    wget https://bench.cr.yp.to/supercop/supercop-20260627.tar.xz
    tar Jxf supercop-20260627.tar.xz
    cd supercop-20260627

    # 環境の初期化 (時間がかかることがある)
    time nohup ./do
    ```

- Github

    ```bash
    git clone https://github.com/jedisct1/supercop.git
    cd supercop

    # 環境の初期化 (時間がかかることがある)
    time nohup ./do
    ```

### 性能測定の安定化 (オプション)

:::note
測定結果が安定するらしい。  
クラウドサービスや仮想マシンの場合エラーになることがある。
:::

```bash
# Intel のターボブースト OFF
echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo

# AMD の boost OFF
echo 0 | sudo tee /sys/devices/system/cpu/cpufreq/boost

# 全コアを "performance" governor に
for i in /sys/devices/system/cpu/cpu*/cpufreq; do
    echo performance | sudo tee "$i/scaling_governor"
done
```

### コンパイルパラメータの調整

**okcompilers/c** で -O3 以外の行をコメントにする。

```bash
gcc -march=native -mtune=native -O3 -fwrapv -fPIC -fPIE -gdwarf-4 -Wall
#gcc -march=native -mtune=native -Os -fwrapv -fPIC -fPIE -gdwarf-4 -Wall
#gcc -march=native -mtune=native -O2 -fwrapv -fPIC -fPIE -gdwarf-4 -Wall
#gcc -march=native -mtune=native -O -fwrapv -fPIC -fPIE -gdwarf-4 -Wall
#clang -march=native -O3 -fwrapv -Qunused-arguments -fPIC -fPIE -gdwarf-4 -Wall
#clang -march=native -Os -fwrapv -Qunused-arguments -fPIC -fPIE -gdwarf-4 -Wall
#clang -march=native -O2 -fwrapv -Qunused-arguments -fPIC -fPIE -gdwarf-4 -Wall
#clang -march=native -O -fwrapv -Qunused-arguments -fPIC -fPIE -gdwarf-4 -Wall
#clang -mcpu=native -O3 -fwrapv -Qunused-arguments -fPIC -fPIE -gdwarf-4 -Wall
```

### 測定環境の初期化

```bash
./do-part init
./do-part crypto_verify 32
./do-part crypto_hash sha512
./do-part crypto_stream chacha20
./do-part crypto_rng chacha20
./do-part crypto_sign ed25519
```

:::note
エラーが出ると思いますが、あまり重要ではないので無視します。  
ここで重要なのは **./bench/ホスト名ディレクトリ/** が作成され、その中にいろいろなファイルが生成されることになります。
:::

---

## 性能測定の実施 (SUPERCOP)

測定用コード格納ディレクトリは **crypto_sign/ecdsap256/ref/** とします。

### 測定用コードの用意

- ソース格納ディレクトリ作成

    ```bash
    mkdir -p crypto_sign/ecdsap256/ref
    ```

<details>
<summary>crypto_sign/ecdsap256/ref/api.h</summary>

```c
#ifndef ECDSAP256_API_H
#define ECDSAP256_API_H

/* 秘密鍵・公開鍵・署名長のサイズ定義（DER を [長さ2B][本体] 格納） */

#define CRYPTO_SECRETKEYBYTES  2048  /* 2 + DER(secret) を想定して十分大きく */
#define CRYPTO_PUBLICKEYBYTES   512  /* 2 + DER(pub)    を想定して十分大きく */
#define CRYPTO_BYTES             64  /* ECDSA P-256 の r(32B)||s(32B) */

/* 関数プロトタイプ（NISTのAPIノート通り） */
int crypto_sign_keypair(unsigned char *pk, unsigned char *sk);

int crypto_sign(
    unsigned char *sm, unsigned long long *smlen,
    const unsigned char *m, unsigned long long mlen,
    const unsigned char *sk
);

int crypto_sign_open(
    unsigned char *m, unsigned long long *mlen,
    const unsigned char *sm, unsigned long long smlen,
    const unsigned char *pk
);

#endif
```
</details>


<details>
<summary>crypto_sign/ecdsap256/ref/sign.c</summary>

```c
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

#include <openssl/evp.h>
#include <openssl/pem.h>
#include <openssl/err.h>
#include <openssl/ec.h>
#include <openssl/ecdsa.h>

/* ここでは crypto_sign.h を include しないことが重要 */
#include "api.h"

/*
    * timingleaks 用ラッパ関数
    *  SUPERCOP が呼ぶ関数名（_timingleaks_）から、
    *  実装本体の crypto_sign_* を呼び出すだけ。
    */
int crypto_sign_ecdsap256_ref_timingleaks_keypair(
    unsigned char *pk,
    unsigned char *sk
) {
    return crypto_sign_keypair(pk, sk);
}

int crypto_sign_ecdsap256_ref_timingleaks(
    unsigned char *sm, unsigned long long *smlen,
    const unsigned char *m, unsigned long long mlen,
    const unsigned char *sk
) {
    return crypto_sign(sm, smlen, m, mlen, sk);
}

int crypto_sign_ecdsap256_ref_timingleaks_open(
    unsigned char *m, unsigned long long *mlen,
    const unsigned char *sm, unsigned long long smlen,
    const unsigned char *pk
) {
    return crypto_sign_open(m, mlen, sm, smlen, pk);
}

/* 2byte 長の読み書き用ヘルパ */
static void store_u16_be(unsigned char *buf, uint16_t v)
{
    buf[0] = (unsigned char)((v >> 8) & 0xff);
    buf[1] = (unsigned char)(v & 0xff);
}

static uint16_t load_u16_be(const unsigned char *buf)
{
    return (uint16_t)((buf[0] << 8) | buf[1]);
}

/* 公開鍵：EVP_PKEY → pk バッファに [len][DER] 形式で保存 */
static int write_pubkey(unsigned char *pk, EVP_PKEY *pkey)
{
    int len;
    unsigned char *der = NULL;
    unsigned char *p;

    len = i2d_PUBKEY(pkey, NULL);
    if (len <= 0 || len + 2 > CRYPTO_PUBLICKEYBYTES) {
        return -1;
    }

    p = der = (unsigned char *)OPENSSL_malloc(len);
    if (!der) return -1;

    if (i2d_PUBKEY(pkey, &p) != len) {
        OPENSSL_free(der);
        return -1;
    }

    store_u16_be(pk, (uint16_t)len);
    memcpy(pk + 2, der, len);

    OPENSSL_free(der);
    return 0;
}

/* pk バッファから EVP_PKEY を生成 */
static EVP_PKEY *read_pubkey(const unsigned char *pk)
{
    uint16_t len = load_u16_be(pk);
    const unsigned char *p = pk + 2;
    EVP_PKEY *pkey = NULL;

    if (2 + len > CRYPTO_PUBLICKEYBYTES) {
        return NULL;
    }

    pkey = d2i_PUBKEY(NULL, &p, len);
    return pkey; /* NULL の場合はエラー */
}

/* 秘密鍵：EVP_PKEY → sk バッファに [len][DER] 形式で保存 */
static int write_privkey(unsigned char *sk, EVP_PKEY *pkey)
{
    int len;
    unsigned char *der = NULL;
    unsigned char *p;

    /* 汎用秘密鍵 (PKCS#8) として保存 */
    len = i2d_PrivateKey(pkey, NULL);
    if (len <= 0 || len + 2 > CRYPTO_SECRETKEYBYTES) {
        return -1;
    }

    p = der = (unsigned char *)OPENSSL_malloc(len);
    if (!der) return -1;

    if (i2d_PrivateKey(pkey, &p) != len) {
        OPENSSL_free(der);
        return -1;
    }

    store_u16_be(sk, (uint16_t)len);
    memcpy(sk + 2, der, len);

    OPENSSL_free(der);
    return 0;
}

/* sk バッファから EVP_PKEY を生成 */
static EVP_PKEY *read_privkey(const unsigned char *sk)
{
    uint16_t len = load_u16_be(sk);
    const unsigned char *p = sk + 2;
    EVP_PKEY *pkey = NULL;

    if (2 + len > CRYPTO_SECRETKEYBYTES) {
        return NULL;
    }

    /* 中身を見て自動判定してくれる便利関数 */
    pkey = d2i_AutoPrivateKey(NULL, &p, len);
    return pkey; /* NULL の場合はエラー */
}

/*
    * ECDSA P-256 用の鍵ペア生成
    *   - EC(secp256r1) 鍵ペア生成
    *   - 公開鍵/秘密鍵をそれぞれ pk/sk バッファに格納
    */
int crypto_sign_keypair(unsigned char *pk, unsigned char *sk)
{
    int ret = -1;
    EVP_PKEY_CTX *ctx = NULL;
    EVP_PKEY *pkey = NULL;

    /* EC キー生成コンテキスト作成 (secp256r1) */
    ctx = EVP_PKEY_CTX_new_id(EVP_PKEY_EC, NULL);
    if (!ctx) goto cleanup;

    if (EVP_PKEY_keygen_init(ctx) <= 0) goto cleanup;

    /* 曲線 secp256r1 (prime256v1) を指定 */
    if (EVP_PKEY_CTX_set_ec_paramgen_curve_nid(ctx, NID_X9_62_prime256v1) <= 0)
        goto cleanup;

    /* 名前付き曲線として保存 */
    if (EVP_PKEY_CTX_set_ec_param_enc(ctx, OPENSSL_EC_NAMED_CURVE) <= 0)
        goto cleanup;

    if (EVP_PKEY_keygen(ctx, &pkey) <= 0) goto cleanup;

    /* バッファに保存 */
    if (write_pubkey(pk, pkey) != 0) goto cleanup;
    if (write_privkey(sk, pkey) != 0) goto cleanup;

    ret = 0;

cleanup:
    if (pkey) EVP_PKEY_free(pkey);
    if (ctx) EVP_PKEY_CTX_free(ctx);

    return ret;
}

/*
    * crypto_sign
    *   入力:
    *     m, mlen : 元メッセージ
    *     sk      : 秘密鍵バイト列
    *   出力:
    *     sm      : [署名 64B (r||s)] || [元メッセージ m]
    *     *smlen  : CRYPTO_BYTES + mlen
    */
int crypto_sign(
    unsigned char *sm, unsigned long long *smlen,
    const unsigned char *m, unsigned long long mlen,
    const unsigned char *sk
) {
    int ret = -1;
    EVP_PKEY *pkey = NULL;
    EVP_MD_CTX *mdctx = NULL;
    unsigned char *der = NULL;
    size_t derlen = 0;
    ECDSA_SIG *ecsig = NULL;
    const BIGNUM *r = NULL, *s = NULL;
    unsigned char *msgpos;

    if (mlen + CRYPTO_BYTES > (unsigned long long)-1) {
        return -1; /* オーバーフロー防止 */
    }

    pkey = read_privkey(sk);
    if (!pkey) goto cleanup;

    mdctx = EVP_MD_CTX_new();
    if (!mdctx) goto cleanup;

    if (EVP_DigestSignInit(mdctx, NULL, EVP_sha256(), NULL, pkey) <= 0)
        goto cleanup;

    if (EVP_DigestSignUpdate(mdctx, m, (size_t)mlen) <= 0)
        goto cleanup;

    /* まず DER 署名長を取得 */
    if (EVP_DigestSignFinal(mdctx, NULL, &derlen) <= 0)
        goto cleanup;

    der = (unsigned char *)OPENSSL_malloc(derlen);
    if (!der) goto cleanup;

    /* 実際に DER 署名を生成 */
    if (EVP_DigestSignFinal(mdctx, der, &derlen) <= 0)
        goto cleanup;

    /* DER → ECDSA_SIG へパース */
    {
        const unsigned char *p = der;
        ecsig = d2i_ECDSA_SIG(NULL, &p, derlen);
        if (!ecsig) goto cleanup;
    }

    /* r, s を取得 */
    ECDSA_SIG_get0(ecsig, &r, &s);

    if (!r || !s) goto cleanup;

    /* r, s をそれぞれ 32byte にパディングして格納 (big-endian) */
    if (BN_num_bytes(r) > 32 || BN_num_bytes(s) > 32)
        goto cleanup; /* P-256 なので 32byte を超えるのは異常 */

    if (BN_bn2binpad(r, sm, 32) != 32)
        goto cleanup;
    if (BN_bn2binpad(s, sm + 32, 32) != 32)
        goto cleanup;

    /* 続けて元メッセージを連結 */
    msgpos = sm + CRYPTO_BYTES;
    memcpy(msgpos, m, (size_t)mlen);
    *smlen = (unsigned long long)CRYPTO_BYTES + mlen;

    ret = 0;

cleanup:
    if (ecsig) ECDSA_SIG_free(ecsig);
    if (der) OPENSSL_free(der);
    if (mdctx) EVP_MD_CTX_free(mdctx);
    if (pkey) EVP_PKEY_free(pkey);

    return ret;
}

/*
    * crypto_sign_open
    *   入力:
    *     sm, smlen : [署名 64B (r||s)] || [元メッセージ]
    *     pk        : 公開鍵
    *   出力:
    *     m, *mlen  : 復元した元メッセージ
    *
    *   戻り値:
    *     0   : 検証成功
    *    -1等: 検証失敗
    */
int crypto_sign_open(
    unsigned char *m, unsigned long long *mlen,
    const unsigned char *sm, unsigned long long smlen,
    const unsigned char *pk
) {
    int ret = -1;
    EVP_PKEY *pkey = NULL;
    EVP_MD_CTX *mdctx = NULL;
    const unsigned char *sig;
    const unsigned char *msg;
    unsigned long long msglen;
    ECDSA_SIG *ecsig = NULL;
    BIGNUM *r = NULL, *s = NULL;
    unsigned char *der = NULL;
    int derlen = 0;

    if (smlen < CRYPTO_BYTES) {
        return -1; /* 不正な長さ */
    }

    /* 先頭 64B が r||s、残りがメッセージ */
    sig = sm;
    msg = sm + CRYPTO_BYTES;
    msglen = smlen - CRYPTO_BYTES;

    /* r, s を 32byte から復元 */
    r = BN_bin2bn(sig, 32, NULL);
    s = BN_bin2bn(sig + 32, 32, NULL);
    if (!r || !s) goto cleanup;

    ecsig = ECDSA_SIG_new();
    if (!ecsig) goto cleanup;

    if (ECDSA_SIG_set0(ecsig, r, s) != 1) {
        goto cleanup;
    }
    /* ここから先 r,s を直接 free してはいけない */

    /* ECDSA_SIG → DER へ変換 */
    derlen = i2d_ECDSA_SIG(ecsig, NULL);
    if (derlen <= 0) goto cleanup;

    der = (unsigned char *)OPENSSL_malloc((size_t)derlen);
    if (!der) goto cleanup;

    {
        unsigned char *p = der;
        if (i2d_ECDSA_SIG(ecsig, &p) != derlen)
            goto cleanup;
    }

    pkey = read_pubkey(pk);
    if (!pkey) goto cleanup;

    mdctx = EVP_MD_CTX_new();
    if (!mdctx) goto cleanup;

    if (EVP_DigestVerifyInit(mdctx, NULL, EVP_sha256(), NULL, pkey) <= 0)
        goto cleanup;

    if (EVP_DigestVerifyUpdate(mdctx, msg, (size_t)msglen) <= 0)
        goto cleanup;

    /* DER 署名で検証 */
    if (EVP_DigestVerifyFinal(mdctx, der, (size_t)derlen) != 1) {
        ret = -1;
        goto cleanup;
    }

    /* 検証 OK の場合、メッセージを呼び出し側に返す */
    memcpy(m, msg, (size_t)msglen);
    *mlen = msglen;
    ret = 0;

cleanup:
    if (der) OPENSSL_free(der);
    if (ecsig) ECDSA_SIG_free(ecsig);
    if (mdctx) EVP_MD_CTX_free(mdctx);
    if (pkey) EVP_PKEY_free(pkey);

    return ret;
}
```

</details>


### 測定の実行

```bash
./do-part crypto_sign ecdsap256
```

### 測定結果の集計

測定結果は CPU のサイクルで出力されるため、秒に変換します。  
gawk がインストールされていないとエラーになります。

<details>
<summary>測定結果変換スクリプト (ここでは、summary.sh とする)</summary>

```bash
#!/usr/bin/env bash

LOGFILE="bench/$(hostname)/data"

# CPU cycles/sec を取得
CPS=$(grep 'cpucycles_persecond' "$LOGFILE" | awk '{print $NF}' | head -n 1)

if [ -z "$CPS" ]; then
    echo "Error: cpucycles_persecond がログから取得できませんでした。" >&2
    exit 1
fi

echo "Detected CPU cycles/sec = $CPS"
echo

# 指定パターンにマッチする全行の 9〜24列 cycles を集め、
# 全体の中央値を求める関数
compute_median() {
local pattern="$1"

grep "$pattern" "$LOGFILE" | gawk -v cps="$CPS" '
    {
    # 各行の 16 サンプルを a[] に格納
    for (i = 9; i <= 24; i++) {
        samples[n++] = $i
    }
    }
    END {
    if (n == 0) {
        exit 1
    }
    # 全部まとめてソート
    asort(samples)

    # 0-based → 中央値は (n-1)/2
    median = samples[int((n-1)/2)]
    sec = median / cps

    printf "%.0f %.6f\n", median, sec
    }
'
}

# 実際の出力処理
print_stat() {
local label="$1"
local desc="$2"
local pattern="$3"

local out
out=$(compute_median "$pattern") || {
    printf "%-10s %-25s Not found (pattern: %s)\n" "[$label]" "$desc" "$pattern"
    return
}

local median sec
median=$(echo "$out" | awk '{print $1}')
sec=$(echo    "$out" | awk '{print $2}')

printf "%-10s %-25s median cycles = %-12s median time = %s sec\n" \
    "[$label]" "$desc" "$median" "$sec"
}

echo "==== crypto_sign ecdsap256/timingleaks summary (ALL SAMPLES) ===="

# 鍵生成（全 keypair_cycles 行）
print_stat "keypair" "key generation" \
"crypto_sign ecdsap256/timingleaks keypair_cycles"

# 署名（全 cycles <index> 行）
print_stat "sign" "sign (all cycles)" \
"crypto_sign ecdsap256/timingleaks cycles [0-9]"

# 署名検証（全 open_cycles <index> 行）
print_stat "verify" "verify (all open_cycles)" \
"crypto_sign ecdsap256/timingleaks open_cycles [0-9]"

echo "==============================================================="
```

</details>

**実行例**

```bash
$ bash summary.sh
Detected CPU cycles/sec = 2995199000

==== crypto_sign ecdsap256/timingleaks summary (ALL SAMPLES) ====
[cpu]      CPU cycles/sec            2995199000   cycles/sec (2.995 GHz)
[keypair]  key generation            median cycles = 231732       median time = 0.000077 sec
[sign]     sign (all cycles)         median cycles = 263992       median time = 0.000088 sec
[verify]   verify (all open_cycles)  median cycles = 261084       median time = 0.000087 sec
===============================================================
```

:::note
公開されている Supercop のベンチマーク結果は秒数ではなく CPU サイクル数であることが多いです。  
なので、測定結果としては変換した秒数ではなく、CPU サイクル数も記録しておくと、公開データとの比較検討に便利です。
:::

---

## メモリ使用量の測定 (valgrind massif)

- 鍵作成、署名、署名検証の3つの処理を別々に実行してメモリ使用量を測定します。
- 性能測定で作成した **crypto_sign/ecdsap256/ref/** を作業ディレクトリとして使用する。

### 測定コードの用意

<details>
<summary>crypto_sign/ecdsap256/ref/main.c</summary>

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdbool.h>
#include <malloc.h>
#include <openssl/crypto.h>
#include "api.h"

static bool g_tracking = false;
static size_t g_current_heap = 0;
static size_t g_peak_heap = 0;
static size_t g_alloc_count = 0;

static void *mem_track_malloc(size_t num, const char *file, int line)
{
    (void)file; (void)line;
    void *ptr = malloc(num);
    if (g_tracking && ptr) {
        size_t usable = malloc_usable_size(ptr);
        g_current_heap += usable;
        if (g_current_heap > g_peak_heap) {
            g_peak_heap = g_current_heap;
        }
        g_alloc_count++;
    }
    return ptr;
}

static void *mem_track_realloc(void *ptr, size_t num, const char *file, int line)
{
    (void)file; (void)line;
    size_t old_usable = ptr ? malloc_usable_size(ptr) : 0;
    void *new_ptr = realloc(ptr, num);
    if (g_tracking && new_ptr) {
        size_t new_usable = malloc_usable_size(new_ptr);
        if (ptr) {
            g_current_heap = g_current_heap + new_usable - old_usable;
        } else {
            g_current_heap += new_usable;
            g_alloc_count++;
        }
        if (g_current_heap > g_peak_heap) {
            g_peak_heap = g_current_heap;
        }
    }
    return new_ptr;
}

static void mem_track_free(void *ptr, const char *file, int line)
{
    (void)file; (void)line;
    if (ptr && g_tracking) {
        size_t usable = malloc_usable_size(ptr);
        if (g_current_heap >= usable) {
            g_current_heap -= usable;
        } else {
            g_current_heap = 0;
        }
    }
    free(ptr);
}

static void reset_mem_stats(void)
{
    g_current_heap = 0;
    g_peak_heap = 0;
    g_alloc_count = 0;
    g_tracking = true;
}

static void stop_mem_stats(size_t *peak, size_t *net, size_t *allocs)
{
    g_tracking = false;
    if (peak) *peak = g_peak_heap;
    if (net) *net = g_current_heap;
    if (allocs) *allocs = g_alloc_count;
}

static void warmup_openssl(void)
{
    unsigned char dummy_pk[CRYPTO_PUBLICKEYBYTES];
    unsigned char dummy_sk[CRYPTO_SECRETKEYBYTES];
    unsigned char dummy_msg[32] = "warmup";
    unsigned char dummy_sm[CRYPTO_BYTES + sizeof(dummy_msg)];
    unsigned long long dummy_smlen;
    unsigned char dummy_m2[sizeof(dummy_msg)];
    unsigned long long dummy_m2len;

    /* OpenSSLのライブラリ初期化・プロバイダロード・テーブル構築等を完了させる */
    crypto_sign_keypair(dummy_pk, dummy_sk);
    crypto_sign(dummy_sm, &dummy_smlen, dummy_msg, sizeof(dummy_msg), dummy_sk);
    crypto_sign_open(dummy_m2, &dummy_m2len, dummy_sm, dummy_smlen, dummy_pk);
}

static void run_keygen(void)
{
    unsigned char pk[CRYPTO_PUBLICKEYBYTES];
    unsigned char sk[CRYPTO_SECRETKEYBYTES];
    size_t peak, net, allocs;

    reset_mem_stats();
    if (crypto_sign_keypair(pk, sk) != 0) {
        fprintf(stderr, "keygen failed\n");
        exit(1);
    }
    stop_mem_stats(&peak, &net, &allocs);

    printf("keygen: peak=%zu bytes, net=%zu bytes, allocs=%zu\n", peak, net, allocs);
}

static void run_sign(void)
{
    unsigned char pk[CRYPTO_PUBLICKEYBYTES];
    unsigned char sk[CRYPTO_SECRETKEYBYTES];
    unsigned char msg[32] = "hello world";
    unsigned char sm[CRYPTO_BYTES + sizeof(msg)];
    unsigned long long smlen;
    size_t peak, net, allocs;

    /* 署名処理の測定対象外として鍵生成を事前に行う */
    if (crypto_sign_keypair(pk, sk) != 0) {
        fprintf(stderr, "keygen failed\n");
        exit(1);
    }

    reset_mem_stats();
    if (crypto_sign(sm, &smlen, msg, sizeof(msg), sk) != 0) {
        fprintf(stderr, "sign failed\n");
        exit(1);
    }
    stop_mem_stats(&peak, &net, &allocs);

    printf("sign  : peak=%zu bytes, net=%zu bytes, allocs=%zu\n", peak, net, allocs);
}

static void run_verify(void)
{
    unsigned char pk[CRYPTO_PUBLICKEYBYTES];
    unsigned char sk[CRYPTO_SECRETKEYBYTES];
    unsigned char msg[32] = "hello world";
    unsigned char sm[CRYPTO_BYTES + sizeof(msg)];
    unsigned long long smlen;
    unsigned char m2[sizeof(msg)];
    unsigned long long m2len;
    size_t peak, net, allocs;

    /* 検証処理の測定対象外として鍵生成と署名を事前に行う */
    if (crypto_sign_keypair(pk, sk) != 0) {
        fprintf(stderr, "keygen failed\n");
        exit(1);
    }
    if (crypto_sign(sm, &smlen, msg, sizeof(msg), sk) != 0) {
        fprintf(stderr, "sign failed\n");
        exit(1);
    }

    reset_mem_stats();
    if (crypto_sign_open(m2, &m2len, sm, smlen, pk) != 0) {
        fprintf(stderr, "verify failed\n");
        exit(1);
    }
    stop_mem_stats(&peak, &net, &allocs);

    printf("verify: peak=%zu bytes, net=%zu bytes, allocs=%zu\n", peak, net, allocs);
}

int main(int argc, char **argv)
{
    if (argc < 2) {
        fprintf(stderr, "usage: %s [keygen|sign|verify|all]\n", argv[0]);
        return 1;
    }

    CRYPTO_set_mem_functions(mem_track_malloc, mem_track_realloc, mem_track_free);
    warmup_openssl();

    if (strcmp(argv[1], "keygen") == 0) {
        run_keygen();
    } else if (strcmp(argv[1], "sign") == 0) {
        run_sign();
    } else if (strcmp(argv[1], "verify") == 0) {
        run_verify();
    } else if (strcmp(argv[1], "all") == 0) {
        run_keygen();
        run_sign();
        run_verify();
    } else {
        fprintf(stderr, "unknown mode: %s\n", argv[1]);
        return 1;
    }

    return 0;
}
```

</details>


<details>
<summary>crypto_sign/ecdsap256/ref/Makefile</summary>

```makefile
CC      := gcc
CFLAGS  := -Wall -Wextra -O0 -g
LDFLAGS := -lcrypto

TARGET  := test
OBJS    := main.o sign.o

.PHONY: all clean massif-keygen massif-sign massif-verify

all: $(TARGET)

$(TARGET): $(OBJS)
    $(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)

main.o: main.c api.h
    $(CC) $(CFLAGS) -c main.c

sign.o: sign.c api.h
    $(CC) $(CFLAGS) -c sign.c

clean:
    rm -f $(TARGET) $(OBJS) massif.keygen massif.sign massif.verify

# ---- Valgrind Massif measurement ----

massif-keygen: $(TARGET)
    valgrind --tool=massif --massif-out-file=massif.keygen ./test keygen
    ms_print massif.keygen > massif.keygen.txt
    @echo "Massif result saved to massif.keygen.txt"

massif-sign: $(TARGET)
    valgrind --tool=massif --massif-out-file=massif.sign ./test sign
    ms_print massif.sign > massif.sign.txt
    @echo "Massif result saved to massif.sign.txt"

massif-verify: $(TARGET)
    valgrind --tool=massif --massif-out-file=massif.verify ./test verify
    ms_print massif.verify > massif.verify.txt
    @echo "Massif result saved to massif.verify.txt"
```

</details>

### ビルドと測定実行

```bash
pushd crypto_sign/ecdsap256/ref

# 測定対象モジュールのビルド
make
# 鍵作成時のメモリ使用量測定
make massif-keygen
# 署名時のメモリ使用量測定
make massif-sign
# 署名検証時のメモリ使用量測定
make massif-verify
```

### ヒープ使用量ピーク値の一括測定スクリプト (ここでは、memory_usage.sh とする)

<details>
<summary>crypto_sign/ecdsap256/ref/memory_usage.sh</summary>

```bash
#!/usr/bin/env bash
set -euo pipefail

PROG=./test

# 必要ならビルド
if [ ! -x "$PROG" ]; then
    echo "[INFO] $PROG が無いので make します..."
    make
fi

measure_one() {
    local mode="$1"
    local output
    output=$("$PROG" "$mode")

    local peak_bytes allocs net_bytes
    peak_bytes=$(echo "$output" | sed -n 's/.*peak=\([0-9]*\) bytes.*/\1/p')
    net_bytes=$(echo "$output" | sed -n 's/.*net=\([0-9]*\) bytes.*/\1/p')
    allocs=$(echo "$output" | sed -n 's/.*allocs=\([0-9]*\).*/\1/p')

    local human="(numfmt not available)"
    if command -v numfmt >/dev/null 2>&1; then
        human=$(numfmt --to=iec "${peak_bytes}")
    fi

    printf "%-7s: %'12d bytes  (%6s)  [allocations: %5d, net leak: %d B]\n" "${mode}" "${peak_bytes}" "${human}" "${allocs}" "${net_bytes}"
}

echo "==== memory usage (excluding initialization & prerequisites) ===="
measure_one keygen
measure_one sign
measure_one verify
```

</details>

**実行例**

```bash
$ bash memory_usage.sh
==== memory usage (excluding initialization & prerequisites) ====
keygen :         9728 bytes  (  9.5K)  [allocations:  1022, net leak: 0 B]
sign   :        17488 bytes  (   18K)  [allocations:   847, net leak: 0 B]
verify :         7496 bytes  (  7.4K)  [allocations:   245, net leak: 0 B]
```
