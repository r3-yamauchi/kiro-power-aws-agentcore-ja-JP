# AgentCore Gateway 統合ガイド

## 概要

このガイドでは、AgentCore Gateway の基本概念から実践的な統合手順まで、包括的に説明します。
Gateway を使用して AI エージェントと外部ツールやサービスを統合する方法を学習できます。

## AgentCore Gateway とは？

### 基本概念

AgentCore Gateway は、AI エージェントと外部ツールやサービスの間の橋渡しを行うマネージドサービスです。エージェントが様々な外部リソースにアクセスできるようにする統一されたインターフェースを提供します。

### 主要な特徴

1. **統一されたアクセスポイント**: 複数の異なるサービスやツールに対して単一のエンドポイントを提供
2. **セキュアな認証**: OAuth2 と Cognito を使用した自動認証システム
3. **スケーラブルなアーキテクチャ**: AWS のマネージドサービスによる自動スケーリング
4. **多様なターゲットサポート**: Lambda、OpenAPI、AWS サービス、MCP サーバーなど
5. **セマンティック検索**: ツールの発見と選択を支援する検索機能

### Gateway が解決する課題

**従来の課題:**
- 各外部サービスごとに異なる認証方式
- 複数のエンドポイント管理の複雑さ
- セキュリティ設定の一貫性の欠如
- スケーリングとパフォーマンスの問題

**Gateway による解決:**
- 統一された OAuth2 認証
- 単一のエンドポイントによる簡素化
- AWS マネージドサービスによる高可用性
- 自動的なロードバランシングとスケーリング

## Gateway のアーキテクチャ

### 全体構成

```
┌─────────────────┐    OAuth2     ┌──────────────────┐
│   AI エージェント   │ ──────────→ │ AgentCore Gateway │
│   (Strands)     │   Bearer Token │    (MCP Server)   │
└─────────────────┘              └──────────────────┘
                                          │
                                          ├── Lambda Functions
                                          ├── OpenAPI Services  
                                          ├── AWS Services (Smithy)
                                          └── MCP Servers
```

### 詳細なアーキテクチャフロー

1. **認証フェーズ**: エージェントが Cognito から OAuth2 トークンを取得
2. **リクエストフェーズ**: エージェントが Gateway に MCP リクエストを送信
3. **ルーティングフェーズ**: Gateway が適切なターゲットにリクエストをルーティング
4. **実行フェーズ**: ターゲットサービスがリクエストを処理
5. **レスポンスフェーズ**: 結果が Gateway 経由でエージェントに返される

### セキュリティレイヤー

- **認証**: OAuth2 + Cognito による身元確認
- **認可**: IAM ロールによる細かいアクセス制御
- **暗号化**: HTTPS による通信の暗号化
- **監査**: CloudTrail による全アクセスの記録

### コンポーネント詳細

1. **Gateway エンドポイント**: MCP プロトコルを実装したメインエンドポイント
2. **Cognito ユーザープール**: OAuth2 認証を管理
3. **IAM 実行ロール**: AWS リソースへのアクセス権限を管理
4. **ターゲット管理**: 各種外部サービスへの接続を管理

## Gateway の利用シナリオ

### 1. エンタープライズ統合
- 既存の社内システムとの連携
- レガシーシステムの API 統合
- データベースアクセスの統一化

### 2. サードパーティサービス統合
- 外部 API サービスの利用
- SaaS アプリケーションとの連携
- パートナーシステムとの接続

### 3. AWS サービス統合
- DynamoDB、S3 などの AWS サービス利用
- Lambda 関数による カスタムロジック実行
- AWS AI/ML サービスとの連携

### 4. マイクロサービスアーキテクチャ
- 複数のマイクロサービスへの統一アクセス
- サービス間通信の簡素化
- 認証・認可の一元管理

## Gateway を使用する利点

### 開発者の利点
- **簡素化された統合**: 単一の MCP インターフェース
- **認証の自動化**: OAuth2 トークン管理の自動化
- **エラーハンドリング**: 統一されたエラー処理
- **デバッグの容易さ**: 一元化されたログとモニタリング

### 運用の利点
- **セキュリティ**: AWS のセキュリティベストプラクティス
- **スケーラビリティ**: 自動スケーリングとロードバランシング
- **可用性**: AWS のマネージドサービスによる高可用性
- **コスト効率**: 使用量ベースの課金

### ビジネスの利点
- **迅速な統合**: 既存システムとの素早い連携
- **柔軟性**: 新しいサービスの容易な追加
- **標準化**: 統一されたアクセスパターン
- **保守性**: 一元化された管理とメンテナンス

## ターゲットタイプの詳細

### 1. Lambda ターゲット
- **用途**: カスタムビジネスロジック、AWS サービス統合
- **認証**: IAM ロール（自動）
- **設定**: Lambda ARN とツールスキーマが必要
- **例**: データ処理、データベースクエリ、カスタム API

### 2. OpenAPI ターゲット
- **用途**: OpenAPI 仕様を持つ外部 REST API
- **認証**: API キーまたは OAuth2
- **設定**: OpenAPI 仕様 URI が必要
- **例**: 天気 API、決済ゲートウェイ、サードパーティサービス

### 3. Smithy モデルターゲット
- **用途**: AWS サービス（DynamoDB、S3 など）
- **認証**: IAM ロール（自動）
- **設定**: 最小限で、事前設定されたモデルを使用
- **例**: DynamoDB 操作、S3 ファイル管理

### 4. MCP サーバーターゲット
- **用途**: 既存の MCP 互換サービス
- **認証**: サーバーによって異なる
- **設定**: サーバーエンドポイントと認証情報
- **例**: カスタム MCP サーバー、サードパーティ MCP サービス

## セマンティック検索機能

### 概要
セマンティック検索は、AI エージェントが大量のツールの中から最適なツールを自動的に発見し、選択できるようにする機能です。

### 仕組み
1. **埋め込みベクトル生成**: 各ツールの説明から意味的なベクトルを生成
2. **クエリマッチング**: ユーザーのクエリとツールの類似度を計算
3. **ランキング**: 関連性の高いツールを優先順位付け
4. **動的選択**: 実行時に最適なツールセットを選択

### 利点
- **インテリジェントな選択**: 自然言語による意図理解
- **スケーラビリティ**: 数百から数千のツールに対応
- **パフォーマンス向上**: 不要なツールの除外

## 前提条件

- AWS CLI が設定されていること
- `bedrock-agentcore-starter-toolkit` がインストールされていること
- 基本的な AWS サービスの知識

## クイックスタート

### 1. Gateway の作成

```bash
# 基本的な Gateway の作成
agentcore gateway create-mcp-gateway \
    --name MyGateway \
    --enable_semantic_search \
    --region us-east-1
```

### 2. ターゲットの追加

#### Lambda ターゲット（詳細は agentcore-lambda-mcp-guide.md を参照）
```bash
agentcore gateway create-mcp-gateway-target \
    --gateway-arn $GATEWAY_ARN \
    --gateway-url $GATEWAY_URL \
    --role-arn $GATEWAY_ROLE_ARN \
    --name MyLambdaTarget \
    --target-type lambda \
    --target-payload "$(cat lambda-schema.json)" \
    --region us-east-1
```

#### OpenAPI ターゲット（詳細は agentcore-openapi-mcp-guide.md を参照）
```bash
agentcore gateway create-mcp-gateway-target \
    --gateway-arn $GATEWAY_ARN \
    --gateway-url $GATEWAY_URL \
    --role-arn $GATEWAY_ROLE_ARN \
    --name MyAPITarget \
    --target-type openApiSchema \
    --target-payload '{
        "openApiSchema": {
            "uri": "https://api.example.com/openapi.json"
        }
    }' \
    --credentials '{
        "api_key": "your-api-key",
        "credential_location": "header",
        "credential_parameter_name": "X-API-Key"
    }' \
    --region us-east-1
```

### 3. エージェントでの使用

```python
from strands import Agent
from bedrock_agentcore import BedrockAgentCoreApp
from mcp_client.client import get_streamable_http_mcp_client
from model.load import load_model

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload):
    mcp_client = get_streamable_http_mcp_client()
    
    with mcp_client:
        gateway_tools = mcp_client.list_tools_sync()
        
        agent = Agent(
            model=load_model(),
            tools=gateway_tools,
            system_prompt="あなたは様々なツールにアクセスできるアシスタントです。"
        )
        
        response = agent(payload.get("prompt", "Hello"))
        return {"response": response}
```

## 統合パターンの選択

### パターン 1: Lambda 関数の活用
- **用途**: 既存のビジネスロジックの活用
- **参照ガイド**: `agentcore-lambda-mcp-guide.md`
- **適用場面**: カスタムデータ処理、既存システム統合

### パターン 2: REST API の統合
- **用途**: 外部サービスとの連携
- **参照ガイド**: `agentcore-openapi-mcp-guide.md`
- **適用場面**: SaaS サービス、パートナー API

### パターン 3: セマンティック検索の活用
- **用途**: 大量のツールからの自動選択
- **参照ガイド**: `agentcore-semantic-search-guide.md`
- **適用場面**: 多機能エージェント、動的ツール選択

## 基本的な管理コマンド

```bash
# Gateway の一覧表示
agentcore gateway list-mcp-gateways --region us-east-1

# Gateway の詳細取得
agentcore gateway get-mcp-gateway --name MyGateway --region us-east-1

# ターゲットの一覧表示
agentcore gateway list-mcp-gateway-targets --name MyGateway --region us-east-1

# Gateway の削除
agentcore gateway delete-mcp-gateway --name MyGateway --force --region us-east-1
```

## 実装パターン

### パターン 1: シンプルな API 統合
```python
# 単一の外部 API との統合
gateway_tools = mcp_client.list_tools_sync()
agent = Agent(model=model, tools=gateway_tools)
```

### パターン 2: ハイブリッド統合
```python
# ローカルツールと Gateway ツールの組み合わせ
local_tools = [calculator, formatter]
gateway_tools = mcp_client.list_tools_sync()
agent = Agent(model=model, tools=local_tools + gateway_tools)
```

### パターン 3: セマンティック検索
```python
# 条件に基づくツールの動的選択
relevant_tools = mcp_client.search_tools_semantic(
    query=user_query, max_results=5, threshold=0.7
)
agent = Agent(model=model, tools=relevant_tools)
```

## 基本的なトラブルシューティング

### Gateway ツールが利用できない
1. Gateway の状態確認: `agentcore gateway get-mcp-gateway --name MyGateway`
2. ターゲットの状態確認: `agentcore gateway list-mcp-gateway-targets --name MyGateway`
3. 認証設定の確認: OAuth2 トークンの生成状況

### 認証エラー
1. クライアント ID とシークレットの確認
2. トークンエンドポイント URL の確認
3. スコープ設定の確認

## 次のステップ

### 1. 専門ガイドの選択
用途に応じて以下のガイドを参照：

- **Lambda MCP 化**: `agentcore-lambda-mcp-guide.md`
- **OpenAPI MCP 変換**: `agentcore-openapi-mcp-guide.md`
- **セマンティック検索**: `agentcore-semantic-search-guide.md`

### 2. 高度な機能の活用
- 複数ターゲットの組み合わせ
- カスタム認証の設定
- パフォーマンス最適化

## 関連ガイド

- **Lambda 統合**: `agentcore-lambda-mcp-guide.md` - Lambda 関数の MCP 化
- **API 統合**: `agentcore-openapi-mcp-guide.md` - REST API の統合
- **検索機能**: `agentcore-semantic-search-guide.md` - セマンティック検索

---

**注意**: このガイドは Gateway の基本概念から実践的な統合手順まで包括的に説明します。専門的な統合方法については、関連ガイドを参照してください。