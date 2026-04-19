# Microsoft Entra External ID と Google アカウントの連携

Microsoft Entra External ID (旧 CIAM: Customer Identity and Access Management) を使用して、Google アカウントでログインし、Python アプリケーションで認証・認可を行う構成の手順です。

## 前提条件

- Microsoft Entra の**外部テナント (CIAM)** が作成済みであること
- Google Cloud のプロジェクトが利用可能であること
- Python パッケージのインストール:

```bash
pip install msal PyJWT requests cryptography
```

> **注意**: `cryptography` パッケージは `PyJWT` が RS256 署名を検証するために必要です。インストールし忘れると実行時エラーになります。

## 1. Google アカウントの設定 (Google Cloud Console)

Microsoft Entra が Google と通信するための OAuth 2.0 認証情報を作成します。

1.  **Google Cloud Console** ([console.cloud.google.com](https://console.cloud.google.com/)) にログインします。
2.  **プロジェクトの作成**: 新しいプロジェクトを作成するか、既存のプロジェクトを選択します。
3.  **OAuth 同意画面の設定**:
    - 「API とサービス」 > 「OAuth 同意画面」に移動します。
    - User Type は **外部 (External)** を選択し、「作成」をクリックします。
    - アプリ名、ユーザーサポートメール、連絡先情報を入力します。
    - **承認済みドメイン** に `microsoftonline.com` と `ciamlogin.com` を追加します。

    > **注意 (本番環境)**: 同意画面の公開ステータスが「テスト中」の場合、ログインできるのは事前登録したテストユーザー (最大 100 名) のみです。本番公開には Google によるアプリ審査が必要です。

4.  **認証情報の作成**:
    - 「認証情報」 > 「認証情報を作成」 > 「OAuth クライアント ID」をクリックします。
    - アプリケーションの種類として **ウェブ アプリケーション** を選択します。
    - **承認済みのリダイレクト URI** に以下を追加します (テナント情報に合わせて置換):
        - `https://login.microsoftonline.com/te/<Tenant-ID>/oauth2/authresp`
        - `https://<Tenant-Name>.ciamlogin.com/<Tenant-ID>/federation/oidc/accounts.google.com`

    > **補足**: 1 つ目の URI (`/te/` を含む形式) は後方互換性のために追加します。Entra External ID では 2 つ目の `ciamlogin.com` の URI が主に使用されます。

5.  作成後に表示される **クライアント ID** と **クライアント シークレット** を控えておきます。

---

## 2. Microsoft Entra External ID の設定

### 2.1. Google を ID プロバイダーとして追加
1.  **Microsoft Entra 管理センター** ([entra.microsoft.com](https://entra.microsoft.com/)) にログインします。
2.  外部テナント (CIAM) を選択していることを確認します。
3.  「External Identities」 > 「すべての ID プロバイダー」に移動します。
4.  「+ Google」を選択します。
5.  Google Cloud Console で取得した **クライアント ID** と **クライアント シークレット** を入力し、保存します。

### 2.2. ユーザーフローの作成
1.  「External Identities」 > 「ユーザー フロー」に移動します。
2.  「+ 新しいユーザー フロー」をクリックし、サインアップとサインインフローを作成します。
3.  「ID プロバイダー」の設定で **Google** にチェックを入れます。
4.  フロー作成後、**「アプリケーション」タブでクライアントアプリを関連付けます**。

    > **注意**: この関連付けを忘れると、アプリからユーザーフローが呼び出されず認証が開始されません。

### 2.3. アプリケーションの登録 (クライアント & サーバー)
- **サーバー (Web API)**: 
    - 「アプリの登録」から API 用のアプリを登録します。
    - 「API の公開」で **Application ID URI** を設定します (既定値: `api://<サーバーのクライアントID>`)。
    - 「スコープの追加」を行い、`access_as_user` などのカスタムスコープを定義します。
    - (オプション) 「トークン設定」 > 「オプションの要求の追加」でアクセストークンに `email`、`given_name`、`family_name` などの要求を追加します。

    > **注意**: デフォルトではアクセストークンに `email` や `name` クレームは含まれません。これらが必要な場合は「オプションの要求」で明示的に追加してください。
- **クライアント (Python App)**:
    - 「アプリの登録」からクライアント用のアプリを登録します。
    - 「API のアクセス許可」から、サーバー API で定義したスコープを追加し、管理者の同意を与えます。

---

## 3. Python 実装例

### 3.1. クライアントアプリ: 認証トークンの取得
`msal` ライブラリを使用して、ブラウザ経由で Google ログインを実行し、アクセストークンを取得します。

```python
import msal

# 設定情報
TENANT_SUBDOMAIN = "your-tenant-name"  # 例: 'contoso'
CLIENT_ID = "your-client-id"           # クライアントアプリのクライアントID
AUTHORITY = f"https://{TENANT_SUBDOMAIN}.ciamlogin.com"
SCOPES = ["api://your-server-app-id/access_as_user"] # サーバーAPIのスコープ

def get_access_token():
    app = msal.PublicClientApplication(
        CLIENT_ID,
        authority=AUTHORITY
    )

    # 1. キャッシュからトークンを取得試行
    accounts = app.get_accounts()
    result = None
    if accounts:
        result = app.acquire_token_silent(SCOPES, account=accounts[0])

    # 2. キャッシュにない場合は対話型ログイン
    if not result:
        print("ブラウザを開いてログインしてください...")
        result = app.acquire_token_interactive(scopes=SCOPES)

    if "access_token" in result:
        return result["access_token"]
    else:
        print(f"エラー: {result.get('error_description')}")
        return None

# 使用例
# token = get_access_token()
# print(f"Token: {token}")
```

> **注意**: `acquire_token_interactive()` はブラウザを起動するため、ローカル実行・CLI ツール専用です。Web アプリ・サーバーサイドアプリには `acquire_token_by_auth_code_flow()` を使用してください。また、対話型ログインにはアプリ登録の「リダイレクト URI」に `http://localhost` を追加しておく必要があります。

> **補足 (本番環境)**: 上記の例ではトークンキャッシュはメモリ内のみに保持されます。アプリ再起動後もキャッシュを維持するには `SerializableTokenCache` を使用してファイルや DB に永続化してください。

### 3.2. サーバーアプリ: トークンの検証とユーザー情報の取得
`PyJWT` を使用して、Microsoft が公開している公開鍵で署名を検証し、ユーザー情報を抽出します。

```python
import jwt
import requests
from jwt import PyJWKClient

# 設定情報
TENANT_SUBDOMAIN = "your-tenant-name"
TENANT_ID = "your-tenant-id-guid"      # ディレクトリ(テナント)ID
# Application ID URI (「API の公開」で設定した値。既定値は api://<サーバーのクライアントID>)
API_ID_URI = "api://your-server-app-id"

# OIDC Discovery エンドポイント
DISCOVERY_URL = f"https://{TENANT_SUBDOMAIN}.ciamlogin.com/{TENANT_ID}/v2.0/.well-known/openid-configuration"

def validate_and_get_user(token):
    try:
        # 1. OIDC設定から公開鍵(JWKS)と発行者(Issuer)の情報を取得
        oidc_config = requests.get(DISCOVERY_URL).json()
        jwks_uri = oidc_config["jwks_uri"]
        valid_issuer = oidc_config["issuer"]

        # 2. 公開鍵を取得して署名を検証
        jwks_client = PyJWKClient(jwks_uri)
        signing_key = jwks_client.get_signing_key_from_jwt(token)

        # 3. デコードと検証 (Audience, Issuer, Expiry の確認)
        claims = jwt.decode(
            token,
            signing_key.key,
            algorithms=["RS256"],
            audience=API_ID_URI,  # Application ID URI を指定 (クライアントIDのGUIDではない)
            issuer=valid_issuer
        )

        # 4. ユーザー情報の抽出
        user_info = {
            "user_id": claims.get("sub"),
            "email": claims.get("email"),   # オプションの要求を設定した場合のみ含まれる
            "name": claims.get("name"),     # オプションの要求を設定した場合のみ含まれる
            "oid": claims.get("oid")        # Object ID (Entra External ID テナント内での一意ID)
        }
        return user_info

    except jwt.ExpiredSignatureError:
        print("Error: トークンの有効期限が切れています。")
    except jwt.InvalidTokenError as e:
        print(f"Error: トークンが無効です: {e}")
    except Exception as e:
        print(f"Error: 検証中にエラーが発生しました: {e}")
    
    return None

# 使用例
# bearer_token = "..." # クライアントから送られてきたトークン
# user = validate_and_get_user(bearer_token)
# if user:
#     print(f"Authenticated as: {user['name']} ({user['email']})")
```

> **注意 (audience の誤り)**: `audience` には Application ID URI (`api://your-server-app-id` の形式) を指定します。クライアントIDのGUID (`xxxxxxxx-xxxx-...`) を直接指定するとトークン検証が失敗します。正しい値はアプリ登録の「API の公開」画面で確認できます。

> **補足**: `PyJWKClient` は内部でキーをキャッシュしますが、`DISCOVERY_URL` からの OIDC 設定の取得はリクエストのたびに行っています。本番環境では `oidc_config` と `jwks_client` をモジュールレベルで初期化し、定期的に更新する設計を検討してください。
