# AgentCore Gateway 統合ガイド

## 概要

このガイドでは、AgentCore Gateway を Strands エージェントに追加するためのステップバイステップの手順を説明します。
Gateway を使用すると、エージェントは MCP (Model Context Protocol) を介して、Lambda 関数、OpenAPI サービスなどの外部ツールにアクセスできます。

## 前提条件

- 展開済みまたは展開準備が整っている既存の Strands エージェント
- `bedrock-agentcore-starter-toolkit` がインストールされていること
- AWS 認証情報が設定されていること
- ツールとして公開する Lambda 関数または API (オプション)

## AgentCore Gateway とは？

AgentCore Gateway は、以下の機能を提供するマネージド MCP エンドポイントです。
- **一元化されたツールアクセス**: 複数のツールソースに対する単一のエンドポイント
- **組み込み認証**: Cognito を使用した自動 OAuth2 セットアップ
- **複数のターゲットタイプ**: Lambda、OpenAPI、Smithy モデル、MCP サーバー
- **セマンティック検索**: オプションのツールに対するセマンティック検索
- **自動スケーリング**: AWS によって管理

## アーキテクチャ

```
あなたのエージェント (Strands)
    ↓ (OAuth2 ベアラートークン)
AgentCore Gateway (MCP エンドポイント)
    ↓ (IAM ロール / API キー)
├── Lambda 関数
├── OpenAPI サービス
├── Smithy モデル (AWS サービス)
└── MCP サーバー
```

## ステップ 1: Gateway の作成

```bash
agentcore gateway create-mcp-gateway \
    --name MyAgentGateway \
    --region us-west-2 \
    --enable_semantic_search
```

これにより、以下が自動的に作成されます。
- Gateway エンドポイント URL
- OAuth2 用の Cognito ユーザープール
- IAM 実行ロール
- クライアント認証情報

**出力を保存してください** - 以下が必要になります。
- Gateway ARN
- Gateway URL
- Gateway ID
- クライアント ID (Cognito から)
- クライアントシークレット (Cognito から)

## ステップ 2: Gateway ターゲットの追加

### オプション A: Lambda 関数ターゲット

Lambda ターゲット設定ファイル `lambda_target.json` を作成します。

```json
{
    "lambdaArn": "arn:aws:lambda:us-west-2:123456789:function:MyFunction",
    "toolSchema": {
        "inlinePayload": [
            {
                "name": "my_tool",
                "description": "このツールが何をするかの説明",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "param1": {
                            "type": "string",
                            "description": "最初のパラメータ"
                        },
                        "param2": {
                            "type": "number",
                            "description": "2番目のパラメータ"
                        }
                    },
                    "required": ["param1"]
                }
            }
        ]
    }
}
```

ターゲットを追加します。

```bash
agentcore gateway create-mcp-gateway-target \
    --gateway-arn <gateway-arn-from-step-1> \
    --gateway-url <gateway-url-from-step-1> \
    --role-arn <role-arn-from-step-1> \
    --name MyLambdaTarget \
    --target-type lambda \
    --target-payload "$(cat lambda_target.json)" \
    --region us-west-2
```

### オプション B: OpenAPI サービスターゲット

```bash
agentcore gateway create-mcp-gateway-target \
    --gateway-arn <gateway-arn> \
    --gateway-url <gateway-url> \
    --role-arn <role-arn> \
    --name MyAPITarget \
    --target-type openApiSchema \
    --target-payload "{\"openApiSchema\": {\"uri\": \"https://api.example.com/openapi.json\"}}" \
    --credentials "{\"api_key\": \"your-api-key\", \"credential_location\": \"header\", \"credential_parameter_name\": \"X-API-Key\"}" \
    --region us-west-2
```

### オプション C: Smithy モデルターゲット (AWS サービス)

```bash
agentcore gateway create-mcp-gateway-target \
    --gateway-arn <gateway-arn> \
    --gateway-url <gateway-url> \
    --role-arn <role-arn> \
    --name DynamoDBTarget \
    --target-type smithyModel \
    --region us-west-2
```

## ステップ 3: Gateway 認証情報の取得

Cognito クライアント認証情報を取得します。

```bash
# ゲートウェイの詳細を取得
agentcore gateway get-mcp-gateway --name MyAgentGateway --region us-west-2

# 出力からクライアント ID を抽出し、シークレットを取得
aws cognito-idp describe-user-pool-client \
    --user-pool-id <user-pool-id> \
    --client-id <client-id> \
    --region us-west-2 \
    --query 'UserPoolClient.[ClientId,ClientSecret]' \
    --output json
```

## ステップ 4: エージェントコードの更新

### 必要な依存関係のインストール

`pyproject.toml` に追加します。
```toml
dependencies = [
    "bedrock-agentcore >= 1.0.3",
    "httpx >= 0.27.0",
    "strands-agents >= 1.13.0"
]
```

または `requirements.txt` に追加します。
```
bedrock-agentcore>=1.0.3
httpx>=0.27.0
strands-agents>=1.13.0
```

### OAuth2 を使用した MCP クライアントの作成

`src/mcp_client/client.py` を作成または更新します。

```python
import os
import httpx
from datetime import datetime, timedelta
from mcp.client.streamable_http import streamablehttp_client
from strands.tools.mcp.mcp_client import MCPClient

# Gateway の設定
GATEWAY_MCP_URL = os.environ.get("GATEWAY_MCP_URL")
GATEWAY_CLIENT_ID = os.environ.get("GATEWAY_CLIENT_ID")
GATEWAY_CLIENT_SECRET = os.environ.get("GATEWAY_CLIENT_SECRET")
GATEWAY_TOKEN_ENDPOINT = os.environ.get("GATEWAY_TOKEN_ENDPOINT")
GATEWAY_SCOPE = os.environ.get("GATEWAY_SCOPE")

# トークンキャッシュ
_token_cache = {"token": None, "expires_at": None}

def get_oauth_token():
    """キャッシュを使用してゲートウェイ認証用の OAuth2 トークンを取得します"""
    # キャッシュされたトークンがまだ有効か確認
    if _token_cache["token"] and _token_cache["expires_at"] and datetime.now() < _token_cache["expires_at"]:
        return _token_cache["token"]
    
    # 新しいトークンを取得
    response = httpx.post(
        GATEWAY_TOKEN_ENDPOINT,
        data={
            "grant_type": "client_credentials",
            "client_id": GATEWAY_CLIENT_ID,
            "client_secret": GATEWAY_CLIENT_SECRET,
            "scope": GATEWAY_SCOPE
        },
        headers={"Content-Type": "application/x-www-form-urlencoded"}
    )
    
    if response.status_code == 200:
        data = response.json()
        token = data["access_token"]
        # 有効期限の 5 分前にキャッシュを更新
        expires_in = data.get("expires_in", 3600) - 300
        _token_cache["token"] = token
        _token_cache["expires_at"] = datetime.now() + timedelta(seconds=expires_in)
        return token
    else:
        raise Exception(f"OAuth トークンの取得に失敗しました: {response.status_code} - {response.text}")

def get_streamable_http_mcp_client() -> MCPClient:
    """AgentCore Gateway 用に設定された MCP クライアントを返します"""
    access_token = get_oauth_token()
    
    return MCPClient(
        lambda: streamablehttp_client(
            GATEWAY_MCP_URL,
            headers={"Authorization": f"Bearer {access_token}"}
        )
    )
```

### メインエージェントコードの更新

**推奨パターン** (よりクリーンで保守しやすい):

```python
from strands import Agent, tool
from bedrock_agentcore import BedrockAgentCoreApp
from mcp_client.client import get_streamable_http_mcp_client
from model.load import load_model

# ローカルツール
@tool
def local_tool(param: str) -> str:
    """エージェント内で実行されるローカルツール"""
    return f"処理済み: {param}"

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload):
    # MCP クライアントを作成
    mcp_client = get_streamable_http_mcp_client()
    
    # コンテキストマネージャー内でゲートウェイツールを取得
    with mcp_client:
        gateway_tools = mcp_client.list_tools_sync()
        
        # すべてのツールでエージェントを作成
        agent = Agent(
            model=load_model(),
            tools=[local_tool] + gateway_tools,
            system_prompt="あなたは様々なツールにアクセスできる役立つアシスタントです。"
        )
        
        prompt = payload.get("prompt", "Hello")
        response = agent(prompt)
        
        return {"response": response}

if __name__ == "__main__":
    app.run()
```

**代替パターン** (クライアントを再利用する必要がある場合):

```python
from strands import Agent, tool
from bedrock_agentcore import BedrockAgentCoreApp
from mcp_client.client import get_streamable_http_mcp_client
from model.load import load_model

# ローカルツール
@tool
def local_tool(param: str) -> str:
    """エージェント内で実行されるローカルツール"""
    return f"処理済み: {param}"

# MCP クライアントを一度作成 (再利用可能)
mcp_client = get_streamable_http_mcp_client()

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload):
    # ゲートウェイツールを取得
    with mcp_client:
        gateway_tools = mcp_client.list_tools_sync()
        
        # すべてのツールでエージェNT を作成
        agent = Agent(
            model=load_model(),
            tools=[local_tool] + gateway_tools,
            system_prompt="あなたは様々なツールにアクセスできる役立つアシスタントです。"
        )
        
        prompt = payload.get("prompt", "Hello")
        response = agent(prompt)
        
        return {"response": response}

if __name__ == "__main__":
    app.run()
```

## ステップ 5: 環境変数の設定

`.env` ファイルを作成します。

```bash
# Gateway の設定
GATEWAY_MCP_URL=https://your-gateway-id.gateway.bedrock-agentcore.us-west-2.amazonaws.com/mcp
GATEWAY_CLIENT_ID=your-client-id
GATEWAY_CLIENT_SECRET=your-client-secret
GATEWAY_TOKEN_ENDPOINT=https://your-cognito-domain.auth.us-west-2.amazoncognito.com/oauth2/token
GATEWAY_SCOPE=YourGatewayName/invoke

# AWS の設定
AWS_REGION=us-west-2
```

**重要**: シークレットを含む `.env` ファイルをバージョン管理にコミットしないでください！

## ステップ 6: エージェントのデプロイ

```bash
agentcore launch
```

## ステップ 7: Gateway 統合のテスト

```bash
# ゲートウェイツールを使用するはずのプロンプトでテスト
agentcore invoke \
    "{ 
    "prompt": "my_tool 関数を param1 を test に設定して使用してください"
}"
```

## Gateway 管理コマンド

### Gateway の一覧表示
```bash
agentcore gateway list-mcp-gateways --region us-west-2
```

### Gateway の詳細取得
```bash
agentcore gateway get-mcp-gateway --name MyAgentGateway --region us-west-2
```

### Gateway ターゲットの一覧表示
```bash
agentcore gateway list-mcp-gateway-targets --name MyAgentGateway --region us-west-2
```

### ターゲットの詳細取得
```bash
agentcore gateway get-mcp-gateway-target \
    --name MyAgentGateway \
    --target-name MyLambdaTarget \
    --region us-west-2
```

### Gateway ターゲットの削除
```bash
agentcore gateway delete-mcp-gateway-target \
    --name MyAgentGateway \
    --target-name MyLambdaTarget \
    --region us-west-2
```

### Gateway の削除 (すべてのターゲットを含む)
```bash
agentcore gateway delete-mcp-gateway \
    --name MyAgentGateway \
    --force \
    --region us-west-2
```

## ターゲットタイプの説明

### 1. Lambda ターゲット
- **用途**: カスタムビジネスロジック、AWS サービス統合
- **認証**: IAM ロール (自動)
- **設定**: Lambda ARN とツールスキーマが必要
- **例**: データ処理、データベースクエリ、カスタム API

### 2. OpenAPI ターゲット
- **用途**: OpenAPI 仕様を持つ外部 REST API
- **認証**: API キーまたは OAuth2
- **設定**: OpenAPI 仕様 URI が必要
- **例**: 天気 API、決済ゲートウェイ、サードパーティサービス

### 3. Smithy モデルターゲット
- **用途**: AWS サービス (DynamoDB、S3 など)
- **認証**: IAM ロール (自動)
- **設定**: 最小限で、事前設定されたモデルを使用
- **例**: DynamoDB 操作、S3 ファイル管理

### 4. MCP サーバーターゲット
- **用途**: 既存の MCP 互換サービス
- **認証**: サーバーによって異なる
- **設定**: サーバーエンドポイントと認証情報
- **例**: カスタム MCP サーバー、サードパーティ MCP サービス

## Lambda 関数の要件

Lambda 関数はツール呼び出しを処理する必要があります。

```python
import json

def lambda_handler(event, context):
    """
    イベント構造:
    {
        "tool_name": "my_tool",
        "arguments": {
            "param1": "value1",
            "param2": 123
        }
    }
    """
    tool_name = event.get('tool_name')
    arguments = event.get('arguments', {})
    
    if tool_name == 'my_tool':
        param1 = arguments.get('param1')
        # リクエストを処理
        result = {"status": "success", "data": f"Processed {param1}"}
        return result
    
    return {"error": f"不明なツール: {tool_name}"}
```

## OpenAPI 認証情報の種類

### API キー認証
```json
{
    "api_key": "your-api-key",
    "credential_location": "header",
    "credential_parameter_name": "X-API-Key"
}
```

### OAuth2 (カスタムプロバイダー)
```json
{
    "oauth2_provider_config": {
        "customOauth2ProviderConfig": {
            "oauthDiscovery": {
                "discoveryUrl": "https://auth.example.com/.well-known/openid-configuration"
            },
            "clientId": "your-client-id",
            "clientSecret": "your-client-secret"
        }
    },
    "scopes": ["read", "write"]
}
```

### OAuth2 (Google)
```json
{
    "oauth2_provider_config": {
        "googleOauth2ProviderConfig": {
            "clientId": "your-google-client-id",
            "clientSecret": "your-google-client-secret"
        }
    },
    "scopes": ["https://www.googleapis.com/auth/userinfo.email"]
}
```

## ベストプラクティス

1. **トークンキャッシュ**: レート制限を避けるために、常に OAuth トークンをキャッシュする
2. **エラー処理**: ゲートウェイ呼び出しに再試行ロジックを実装する
3. **ツール命名**: 説明的で一意なツール名を使用する
4. **スキーマ検証**: ツールスキーマが Lambda の期待値と一致することを確認する
5. **セキュリティ**: 認証情報をバージョン管理にコミットしない
6. **モニタリング**: ゲートウェイエラーについては CloudWatch ログを確認する
7. **テスト**: デプロイする前にゲートウェイツールをローカルでテストする

## トラブルシューティング

### Gateway ツールが利用できない
- ゲートウェイ URL が正しいことを確認する
- OAuth トークンが生成されていることを確認する
- ゲートウェイのステータスが READY であることを確認する
- ターゲットのステータスが READY であることを確認する

### 認証エラー
- クライアント ID とシークレットが正しいことを確認する
- トークンエンドポイント URL を確認する
- スコープがゲートウェイ名と一致することを確認する
- トークンの有効期限と更新ロジックを確認する

### ツール実行の失敗
- Lambda 関数のアクセス許可を確認する
- CloudWatch で Lambda 関数のログを確認する
- ツールスキーマが Lambda の期待値と一致することを確認する
- IAM ロールに Lambda 呼び出しのアクセス許可があることを確認する

### Gateway 作成の失敗
- AgentCore Gateway の AWS アクセス許可を確認する
- リージョンがサポートされていることを確認する
- ゲートウェイ名が一意であることを確認する

## 例: Gateway を使用した完全なエージェント

**完全な動作例** (src/main.py):

```python
import os
from strands import Agent, tool
from bedrock_agentcore import BedrockAgentCoreApp
from mcp_client.client import get_streamable_http_mcp_client
from model.load import load_model

@tool
def local_calculator(a: int, b: int) -> int:
    """2つの数値をローカルで加算します"""
    return a + b

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload):
    # ゲートウェイ用の MCP クライアントを作成
    mcp_client = get_streamable_http_mcp_client()
    
    # コンテキストマネージャー内でゲートウェイツールを取得
    with mcp_client:
        gateway_tools = mcp_client.list_tools_sync()
        
        # ローカルツールとゲートウェイツールの両方でエージェントを作成
        agent = Agent(
            model=load_model(),
            tools=[local_calculator] + gateway_tools,
            system_prompt="あなたはローカルツールとリモートツールの両方にアクセスできる役立つアシスタントです。"
        )
        
        prompt = payload.get("prompt", "Hello")
        response = agent(prompt)
        
        return {"response": response}

if __name__ == "__main__":
    app.run()
```

**MCP クライアントモジュール** (src/mcp_client/client.py):

```python
import os
import httpx
from datetime import datetime, timedelta
from mcp.client.streamable_http import streamablehttp_client
from strands.tools.mcp.mcp_client import MCPClient

# 環境からの Gateway 設定
GATEWAY_MCP_URL = os.environ.get("GATEWAY_MCP_URL")
GATEWAY_CLIENT_ID = os.environ.get("GATEWAY_CLIENT_ID")
GATEWAY_CLIENT_SECRET = os.environ.get("GATEWAY_CLIENT_SECRET")
GATEWAY_TOKEN_ENDPOINT = os.environ.get("GATEWAY_TOKEN_ENDPOINT")
GATEWAY_SCOPE = os.environ.get("GATEWAY_SCOPE")

# パフォーマンスのためのトークンキャッシュ
_token_cache = {"token": None, "expires_at": None}

def get_oauth_token():
    """キャッシュを使用して OAuth2 トークンを取得します"""
    if _token_cache["token"] and _token_cache["expires_at"] and datetime.now() < _token_cache["expires_at"]:
        return _token_cache["token"]
    
    response = httpx.post(
        GATEWAY_TOKEN_ENDPOINT,
        data={
            "grant_type": "client_credentials",
            "client_id": GATEWAY_CLIENT_ID,
            "client_secret": GATEWAY_CLIENT_SECRET,
            "scope": GATEWAY_SCOPE
        },
        headers={"Content-Type": "application/x-www-form-urlencoded"}
    )
    
    if response.status_code == 200:
        data = response.json()
        token = data["access_token"]
        expires_in = data.get("expires_in", 3600) - 300
        _token_cache["token"] = token
        _token_cache["expires_at"] = datetime.now() + timedelta(seconds=expires_in)
        return token
    else:
        raise Exception(f"OAuth トークンの取得に失敗しました: {response.status_code}")

def get_streamable_http_mcp_client() -> MCPClient:
    """AgentCore Gateway 用に設定された MCP クライアントを返します"""
    access_token = get_oauth_token()
    
    return MCPClient(
        lambda: streamablehttp_client(
            GATEWAY_MCP_URL,
            headers={"Authorization": f"Bearer {access_token}"}
        )
    )
```

## Gateway とメモリの組み合わせ

Gateway と Memory を一緒に使用できます。

```python
from bedrock_agentcore.memory.integrations.strands.config import AgentCoreMemoryConfig
from bedrock_agentcore.memory.integrations.strands.session_manager import AgentCoreMemorySessionManager

@app.entrypoint
def invoke(payload):
    # メモリの設定
    memory_config = AgentCoreMemoryConfig(
        memory_id=os.environ.get("AGENTCORE_MEMORY_ID"),
        session_id=payload.get("session_id", "default"),
        actor_id=payload.get("actor_id", "user")
    )
    session_manager = AgentCoreMemorySessionManager(
        agentcore_memory_config=memory_config,
        region_name="us-west-2"
    )
    
    # Gateway の設定
    with gateway_client as client:
        gateway_tools = client.list_tools_sync()
        
        # メモリとゲートウェイツールの両方を持つエージェント
        agent = Agent(
            model=load_model(),
            tools=gateway_tools,
            session_manager=session_manager
        )
        
        prompt = payload.get("prompt")
        response = agent(prompt)
        
        return {"response": response}
```

## リソース

- [AgentCore Gateway ドキュメント](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway.html)
- [MCP プロトコル仕様](https://modelcontextprotocol.io/)
- [Strands MCP ツール](https://strandsagents.com/latest/documentation/docs/user-guide/concepts/tools/mcp-tools/)

## クイックリファレンス

```bash
# ゲートウェイを作成
agentcore gateway create-mcp-gateway --name MyGateway --region us-west-2

# Lambda ターゲットを追加
agentcore gateway create-mcp-gateway-target \
    --gateway-arn <arn> \
    --gateway-url <url> \
    --role-arn <role> \
    --name MyTarget \
    --target-type lambda \
    --target-payload "$(cat config.json)"

# ゲートウェイを一覧表示
agentcore gateway list-mcp-gateways

# ゲートウェイを削除
agentcore gateway delete-mcp-gateway --name MyGateway --force
```

---

**注意**: Gateway は外部ツールへの集中アクセスを提供します。Gateway を一度作成し、必要に応じてターゲットを追加し、OAuth2 認証を介してエージェントコードで参照します。