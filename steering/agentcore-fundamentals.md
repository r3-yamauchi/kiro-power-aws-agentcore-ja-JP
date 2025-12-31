---
inclusion: manual
---

# Bedrock AgentCore とは何か？ - 基礎知識

このステアリングファイルは、Amazon Bedrock AgentCore の基本概念、アーキテクチャ、および主要機能について包括的に説明します。

## Amazon Bedrock AgentCore の概要

### AgentCore とは

Amazon Bedrock AgentCore は、**生成AI エージェントを構築、デプロイ、運用するための包括的なプラットフォーム**です。開発者が複雑なエージェントアプリケーションを効率的に作成できるよう設計されています。

### 主要な特徴

#### 1. **統合開発環境**
- **ローカル開発**: ホットリロード機能付きの開発サーバー
- **テスト環境**: 本番デプロイ前のローカルテスト機能
- **デバッグ支援**: 詳細なログとエラー追跡

#### 2. **マルチフレームワーク対応**
- **Strands**: AWS独自のエージェントフレームワーク
- **Claude**: Anthropic のエージェント SDK
- **OpenAI**: OpenAI のエージェント機能
- **カスタムフレームワーク**: 独自実装のサポート

#### 3. **マルチモデルプロバイダー**
- **Amazon Bedrock**: Claude、Llama、Titan などの基盤モデル
- **OpenAI**: GPT-4、GPT-3.5 などのモデル
- **その他**: 互換性のあるモデルプロバイダー

#### 4. **エンタープライズ機能**
- **スケーラビリティ**: AWS インフラストラクチャによる自動スケーリング
- **セキュリティ**: IAM、VPC、暗号化による包括的なセキュリティ
- **監視**: CloudWatch による詳細な監視とアラート

## AgentCore のアーキテクチャ

### コアコンポーネント

#### 1. **エージェントランタイム (AgentCore Runtime)**

AgentCore Runtime は、エージェントアプリケーションを AWS クラウド上で実行するための包括的なランタイム環境です。

```
┌─────────────────────────────────────────────────────────────┐
│                    AgentCore Runtime                        │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  Request Router │  │ Session Manager │  │ Model Interface │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ Error Handler   │  │ Logger System   │  │ Security Layer  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ Memory Manager  │  │ Gateway Bridge  │  │ Metrics Monitor │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

##### **Runtime の主要コンポーネント**

**1. Request Router (リクエストルーター)**
- 受信リクエストの解析と振り分け
- プロトコル変換（HTTP/WebSocket/gRPC）
- 負荷分散とルーティング
- リクエスト検証とサニタイゼーション

**2. Session Manager (セッションマネージャー)**
- セッション ID の生成と管理
- セッション状態の永続化
- セッションタイムアウト処理
- 並行セッションの制御

**3. Model Interface (モデルインターフェース)**
- 複数モデルプロバイダーへの統一インターフェース
- モデル選択とフォールバック
- レスポンス形式の標準化
- モデル固有の最適化

**4. Error Handler (エラーハンドラー)**
- 例外の捕捉と分類
- エラーレスポンスの生成
- 再試行ロジック
- 障害時の graceful degradation

**5. Logger System (ログシステム)**
- 構造化ログの出力
- ログレベルの動的制御
- CloudWatch との統合
- セキュリティログの管理

**6. Security Layer (セキュリティレイヤー)**
- 認証・認可の処理
- 入力値の検証
- レート制限の実装
- セキュリティヘッダーの付与

##### **Runtime の動作フロー**

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │───▶│   Gateway   │───▶│   Runtime   │───▶│   Agent     │
│   Request   │    │   (API GW)  │    │   Router    │    │   Function  │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                           │                   │                   │
                           ▼                   ▼                   ▼
                   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
                   │    Auth     │    │   Session   │    │   Memory    │
                   │  Validation │    │   Manager   │    │   System    │
                   └─────────────┘    └─────────────┘    └─────────────┘
                                              │                   │
                                              ▼                   ▼
                                      ┌─────────────┐    ┌─────────────┐
                                      │   Model     │    │   Storage   │
                                      │ Interface   │    │   Backend   │
                                      └─────────────┘    └─────────────┘
```

##### **Runtime の設定と管理**

**基本設定ファイル (.bedrock_agentcore.yaml)**
```yaml
# AgentCore Runtime 設定
runtime:
  # エントリポイント設定
  entrypoint: "src/main.py"
  handler: "app"
  
  # ランタイム環境
  python_version: "3.11"
  timeout: 300
  memory_size: 1024
  
  # 環境変数
  environment:
    LOG_LEVEL: "INFO"
    REGION: "us-east-1"
    
  # セッション設定
  session:
    timeout: 1800  # 30分
    max_concurrent: 100
    
  # セキュリティ設定
  security:
    cors_enabled: true
    rate_limit: 1000
    
  # モデル設定
  models:
    default_provider: "bedrock"
    fallback_enabled: true
```

**Runtime 管理コマンド**
```bash
# Runtime 状態確認
uv run agentcore status --region us-east-1

# Runtime ログ確認
uv run agentcore logs --follow --region us-east-1

# Runtime メトリクス確認
uv run agentcore metrics --region us-east-1

# Runtime 設定更新
uv run agentcore configure --timeout 600 --memory 2048 --region us-east-1

# Runtime 再起動
uv run agentcore restart --region us-east-1
```

##### **Runtime のスケーリング**

**自動スケーリング設定**
```yaml
scaling:
  # 最小・最大インスタンス数
  min_instances: 1
  max_instances: 100
  
  # スケーリングトリガー
  cpu_threshold: 70
  memory_threshold: 80
  request_threshold: 1000
  
  # スケーリング速度
  scale_up_cooldown: 60
  scale_down_cooldown: 300
```

**負荷分散設定**
```yaml
load_balancing:
  algorithm: "round_robin"  # round_robin, least_connections, weighted
  health_check:
    enabled: true
    interval: 30
    timeout: 5
    unhealthy_threshold: 3
```

##### **Runtime の監視とメトリクス**

**主要メトリクス**
- **リクエストメトリクス**: RPS、レイテンシ、エラー率
- **リソースメトリクス**: CPU、メモリ、ネットワーク使用量
- **セッションメトリクス**: アクティブセッション数、セッション継続時間
- **モデルメトリクス**: モデル呼び出し回数、レスポンス時間

**CloudWatch 統合**
```python
# カスタムメトリクスの送信
from bedrock_agentcore.monitoring import MetricsCollector

metrics = MetricsCollector()
metrics.put_metric("CustomMetric", 1.0, unit="Count")
metrics.put_metric("ProcessingTime", duration, unit="Seconds")
```

**アラート設定例**
```yaml
alerts:
  - name: "HighErrorRate"
    metric: "ErrorRate"
    threshold: 5.0
    comparison: "GreaterThanThreshold"
    
  - name: "HighLatency"
    metric: "ResponseTime"
    threshold: 5000
    comparison: "GreaterThanThreshold"
```

##### **Runtime のセキュリティ**

**認証・認可**
```python
# JWT トークン検証
from bedrock_agentcore.security import JWTValidator

validator = JWTValidator(
    issuer="https://your-auth-provider.com",
    audience="your-agent-api"
)

def secure_agent(request, context):
    # トークン検証
    token = request.headers.get("Authorization")
    claims = validator.validate(token)
    
    # 認可チェック
    if not claims.get("agent_access"):
        raise UnauthorizedError("Insufficient permissions")
    
    return your_agent_logic(request, context)
```

**入力検証**
```python
from bedrock_agentcore.validation import RequestValidator

validator = RequestValidator({
    "prompt": {"type": "string", "max_length": 10000},
    "session_id": {"type": "string", "pattern": "^[a-zA-Z0-9-]+$"}
})

def validated_agent(request, context):
    # 入力検証
    validator.validate(request.json)
    return your_agent_logic(request, context)
```

##### **Runtime のパフォーマンス最適化**

**1. コールドスタート最適化**
```python
# グローバル初期化（コンテナ再利用時に実行されない）
import boto3
from bedrock_agentcore import BedrockAgentCoreApp

# 事前初期化
bedrock_client = boto3.client('bedrock-runtime')
model_cache = {}

def optimized_agent(request, context):
    # 高速な処理
    return process_with_cache(request, model_cache)
```

**2. メモリ効率化**
```python
# ストリーミング処理
def streaming_agent(request, context):
    for chunk in process_large_data_stream(request):
        yield {"chunk": chunk}
```

**3. 並行処理**
```python
import asyncio
from bedrock_agentcore.async_support import AsyncAgentCoreApp

async def async_agent(request, context):
    # 並行処理
    tasks = [
        process_task_1(request),
        process_task_2(request),
        process_task_3(request)
    ]
    results = await asyncio.gather(*tasks)
    return combine_results(results)
```

#### 2. **メモリシステム**
```
┌─────────────────────────────────────┐
│           Memory System             │
├─────────────────────────────────────┤
│ • 短期メモリ (セッション)              │
│ • 長期メモリ (永続化)                 │
│ • ベクトル検索                       │
│ • メモリ管理                         │
└─────────────────────────────────────┘
```

**メモリタイプ:**
- **セッションメモリ**: 会話中の一時的な情報保存
- **永続メモリ**: ユーザー情報や学習データの長期保存
- **ベクトルメモリ**: 埋め込みベースの類似性検索
- **構造化メモリ**: データベース形式での情報管理

#### 3. **ゲートウェイシステム**
```
┌─────────────────────────────────────┐
│          Gateway System             │
├─────────────────────────────────────┤
│ • API ゲートウェイ                    │
│ • 認証・認可                         │
│ • レート制限                         │
│ • ルーティング                       │
└─────────────────────────────────────┘
```

**機能:**
- RESTful API の提供
- JWT/OAuth による認証
- API レート制限とスロットリング
- 複数エージェントへのルーティング

### データフロー

```
┌─────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Client  │───▶│  Gateway    │───▶│   Runtime   │───▶│   Memory    │
│         │    │             │    │             │    │             │
└─────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                       │                   │                   │
                       ▼                   ▼                   ▼
               ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
               │    Auth     │    │   Models    │    │  Storage    │
               │             │    │             │    │             │
               └─────────────┘    └─────────────┘    └─────────────┘
```

## AgentCore の主要機能

### 1. **エージェント開発ライフサイクル**

#### 開発フェーズ
```bash
# プロジェクト作成
uv run agentcore create --project-name MyAgent --region us-east-1

# ローカル開発
uv run agentcore dev

# ローカルテスト
uv run agentcore invoke --dev '{"prompt": "Hello"}'
```

#### デプロイフェーズ
```bash
# 設定
uv run agentcore configure --entrypoint src/main.py --region us-east-1

# デプロイ
uv run agentcore launch --region us-east-1

# 本番テスト
uv run agentcore invoke '{"prompt": "Hello"}' --region us-east-1
```

### 2. **セッション管理**

#### セッションの概念
- **セッション ID**: 会話の継続性を保つ一意識別子
- **セッション状態**: 会話履歴とコンテキストの保持
- **セッション終了**: リソースの適切な解放

#### セッション操作
```bash
# セッション付きで呼び出し
uv run agentcore invoke '{"prompt": "Hello"}' --session-id my-session

# セッション状態確認
uv run agentcore get-session --session-id my-session

# セッション終了
uv run agentcore stop-session --session-id my-session
```

### 3. **メモリ統合**

#### メモリの種類と用途

**短期メモリ (Session Memory)**
- 会話履歴の保持
- 一時的なコンテキスト情報
- セッション終了時に削除

**長期メモリ (Persistent Memory)**
- ユーザープロファイル
- 学習データ
- 永続的な知識ベース

#### メモリ設定例
```python
from bedrock_agentcore import BedrockAgentCoreApp
from bedrock_agentcore.memory import MemoryConfig

memory_config = MemoryConfig(
    mode="PERSISTENT",
    storage_type="DYNAMODB",
    ttl_seconds=86400  # 24時間
)

app = BedrockAgentCoreApp(
    agent=my_agent,
    memory_config=memory_config
)
```

### 4. **ゲートウェイ統合**

#### API ゲートウェイの設定
```yaml
# gateway-config.yaml
gateway:
  name: "my-agent-gateway"
  authentication:
    type: "JWT"
    issuer: "https://my-auth-provider.com"
  rate_limiting:
    requests_per_minute: 100
  cors:
    allowed_origins: ["https://my-app.com"]
```

#### エンドポイント設定
```python
from bedrock_agentcore.gateway import GatewayConfig

gateway_config = GatewayConfig(
    name="my-agent-api",
    authentication_required=True,
    rate_limit=100,
    cors_enabled=True
)
```

## AgentCore の利点

### 1. **開発効率の向上**
- **ボイラープレートコードの削減**: 共通機能の自動化
- **ホットリロード**: コード変更の即座反映
- **統合テスト環境**: ローカルでの完全テスト

### 2. **運用の簡素化**
- **自動スケーリング**: トラフィックに応じた自動調整
- **監視とアラート**: CloudWatch による包括的監視
- **ログ管理**: 構造化されたログ出力

### 3. **セキュリティの強化**
- **IAM 統合**: AWS の権限管理システム
- **VPC サポート**: ネットワークレベルの分離
- **暗号化**: 保存時・転送時の暗号化

### 4. **コスト最適化**
- **サーバーレス**: 使用量ベースの課金
- **リソース効率**: 自動的なリソース管理
- **スケールゼロ**: 未使用時のコスト削減

## 実装パターン

### 1. **シンプルなチャットボット**
```python
from bedrock_agentcore import BedrockAgentCoreApp

def simple_agent(request):
    prompt = request.get("prompt", "")
    # エージェントロジック
    response = f"Echo: {prompt}"
    return {"response": response}

app = BedrockAgentCoreApp(agent=simple_agent)
```

### 2. **メモリ付きエージェント**
```python
from bedrock_agentcore import BedrockAgentCoreApp
from bedrock_agentcore.memory import MemoryConfig

def memory_agent(request, memory):
    prompt = request.get("prompt", "")
    
    # 過去の会話を取得
    history = memory.get("conversation_history", [])
    
    # 新しい会話を追加
    history.append({"user": prompt})
    
    # エージェントロジック
    response = generate_response(prompt, history)
    
    # メモリに保存
    history.append({"assistant": response})
    memory.set("conversation_history", history)
    
    return {"response": response}

memory_config = MemoryConfig(mode="PERSISTENT")
app = BedrockAgentCoreApp(
    agent=memory_agent,
    memory_config=memory_config
)
```

### 3. **マルチモーダルエージェント**
```python
def multimodal_agent(request):
    content_type = request.get("content_type", "text")
    
    if content_type == "text":
        return handle_text(request)
    elif content_type == "image":
        return handle_image(request)
    elif content_type == "audio":
        return handle_audio(request)
    else:
        return {"error": "Unsupported content type"}
```

### 4. **Runtime 最適化エージェント**
```python
from bedrock_agentcore import BedrockAgentCoreApp
from bedrock_agentcore.runtime import RuntimeConfig
import boto3

# グローバル初期化（コールドスタート最適化）
bedrock_client = boto3.client('bedrock-runtime', region_name='us-east-1')
model_cache = {}

def optimized_agent(request, context):
    """Runtime 最適化されたエージェント実装"""
    
    # リクエスト検証
    if not request.get("prompt"):
        return {"error": "Prompt is required"}
    
    prompt = request["prompt"]
    session_id = request.get("session_id")
    
    # セッション管理
    if session_id:
        context.session_manager.update_activity(session_id)
    
    # モデル呼び出し（キャッシュ利用）
    model_id = request.get("model_id", "anthropic.claude-3-sonnet-20240229-v1:0")
    
    try:
        # Bedrock モデル呼び出し
        response = bedrock_client.invoke_model(
            modelId=model_id,
            body=json.dumps({
                "anthropic_version": "bedrock-2023-05-31",
                "max_tokens": 1000,
                "messages": [{"role": "user", "content": prompt}]
            })
        )
        
        # レスポンス処理
        result = json.loads(response['body'].read())
        answer = result['content'][0]['text']
        
        # メトリクス記録
        context.metrics.put_metric("ModelInvocation", 1.0, unit="Count")
        
        return {
            "response": answer,
            "model_id": model_id,
            "session_id": session_id
        }
        
    except Exception as e:
        # エラーハンドリング
        context.logger.error(f"Model invocation failed: {str(e)}")
        context.metrics.put_metric("ModelError", 1.0, unit="Count")
        
        return {
            "error": "Internal server error",
            "error_code": "MODEL_INVOCATION_FAILED"
        }

# Runtime 設定
runtime_config = RuntimeConfig(
    timeout=300,
    memory_size=1024,
    environment={
        "LOG_LEVEL": "INFO",
        "MODEL_REGION": "us-east-1"
    }
)

app = BedrockAgentCoreApp(
    agent=optimized_agent,
    runtime_config=runtime_config
)
```

### 5. **非同期処理エージェント**
```python
import asyncio
from bedrock_agentcore.async_support import AsyncAgentCoreApp

async def async_agent(request, context):
    """非同期処理を活用したエージェント"""
    
    prompt = request.get("prompt", "")
    
    # 並行処理タスク
    tasks = [
        fetch_knowledge_base(prompt),
        analyze_sentiment(prompt),
        generate_response(prompt)
    ]
    
    # 並行実行
    knowledge, sentiment, response = await asyncio.gather(*tasks)
    
    # 結果統合
    enhanced_response = enhance_with_context(response, knowledge, sentiment)
    
    return {
        "response": enhanced_response,
        "metadata": {
            "sentiment": sentiment,
            "knowledge_sources": len(knowledge)
        }
    }

async def fetch_knowledge_base(query):
    """知識ベースから関連情報を取得"""
    # 非同期データベースクエリ
    pass

async def analyze_sentiment(text):
    """感情分析を実行"""
    # 非同期 AI サービス呼び出し
    pass

async def generate_response(prompt):
    """レスポンス生成"""
    # 非同期モデル呼び出し
    pass

app = AsyncAgentCoreApp(agent=async_agent)
```

### 6. **エンタープライズエージェント**
```python
from bedrock_agentcore import BedrockAgentCoreApp
from bedrock_agentcore.security import JWTValidator, RateLimiter
from bedrock_agentcore.monitoring import MetricsCollector
from bedrock_agentcore.validation import RequestValidator

# セキュリティ設定
jwt_validator = JWTValidator(
    issuer="https://your-company.auth0.com/",
    audience="agent-api"
)

rate_limiter = RateLimiter(
    requests_per_minute=100,
    burst_limit=10
)

# 入力検証
request_validator = RequestValidator({
    "prompt": {
        "type": "string",
        "min_length": 1,
        "max_length": 10000,
        "pattern": "^[^<>]*$"  # XSS 防止
    },
    "user_id": {
        "type": "string",
        "pattern": "^[a-zA-Z0-9-]+$"
    }
})

def enterprise_agent(request, context):
    """エンタープライズグレードのエージェント"""
    
    # 認証チェック
    token = request.headers.get("Authorization")
    claims = jwt_validator.validate(token)
    user_id = claims.get("sub")
    
    # レート制限チェック
    rate_limiter.check_limit(user_id)
    
    # 入力検証
    request_validator.validate(request.json)
    
    # 監査ログ
    context.audit_logger.info({
        "event": "agent_invocation",
        "user_id": user_id,
        "timestamp": context.timestamp,
        "request_id": context.request_id
    })
    
    try:
        # ビジネスロジック
        result = process_business_logic(request, user_id)
        
        # 成功メトリクス
        context.metrics.put_metric("SuccessfulInvocation", 1.0, unit="Count")
        
        return result
        
    except Exception as e:
        # エラーログ
        context.logger.error(f"Business logic error: {str(e)}", extra={
            "user_id": user_id,
            "request_id": context.request_id
        })
        
        # エラーメトリクス
        context.metrics.put_metric("FailedInvocation", 1.0, unit="Count")
        
        raise

app = BedrockAgentCoreApp(
    agent=enterprise_agent,
    security_config={
        "jwt_validator": jwt_validator,
        "rate_limiter": rate_limiter
    }
)
```

## ベストプラクティス

### 1. **Runtime 開発のベストプラクティス**

#### **コールドスタート最適化**
```python
# ❌ 悪い例：毎回初期化
def slow_agent(request, context):
    client = boto3.client('bedrock-runtime')  # 毎回作成
    return process_request(request, client)

# ✅ 良い例：グローバル初期化
import boto3
client = boto3.client('bedrock-runtime')  # 一度だけ作成

def fast_agent(request, context):
    return process_request(request, client)
```

#### **メモリ効率化**
```python
# ✅ ストリーミング処理
def memory_efficient_agent(request, context):
    for chunk in process_large_dataset_stream(request):
        yield {"chunk": chunk}

# ✅ 適切なガベージコレクション
import gc

def cleanup_agent(request, context):
    try:
        result = heavy_processing(request)
        return result
    finally:
        gc.collect()  # 明示的なメモリ解放
```

#### **エラーハンドリング**
```python
from bedrock_agentcore.exceptions import AgentCoreException

def robust_agent(request, context):
    try:
        return process_request(request)
    except ValidationError as e:
        context.logger.warning(f"Validation error: {e}")
        return {"error": "Invalid input", "code": "VALIDATION_ERROR"}
    except ModelError as e:
        context.logger.error(f"Model error: {e}")
        return {"error": "Service unavailable", "code": "MODEL_ERROR"}
    except Exception as e:
        context.logger.error(f"Unexpected error: {e}")
        return {"error": "Internal error", "code": "INTERNAL_ERROR"}
```

### 2. **Runtime 本番運用のベストプラクティス**

#### **監視とアラート設定**
```yaml
# CloudWatch アラーム設定
monitoring:
  alarms:
    - name: "HighErrorRate"
      metric: "ErrorRate"
      threshold: 5.0
      period: 300
      evaluation_periods: 2
      
    - name: "HighLatency"
      metric: "Duration"
      threshold: 30000  # 30秒
      period: 300
      
    - name: "LowSuccessRate"
      metric: "SuccessRate"
      threshold: 95.0
      comparison: "LessThanThreshold"
```

#### **ログ管理**
```python
import structlog

# 構造化ログの設定
logger = structlog.get_logger()

def well_logged_agent(request, context):
    logger.info("Agent invocation started", 
                request_id=context.request_id,
                user_id=request.get("user_id"))
    
    start_time = time.time()
    
    try:
        result = process_request(request)
        
        duration = time.time() - start_time
        logger.info("Agent invocation completed",
                   request_id=context.request_id,
                   duration=duration,
                   success=True)
        
        return result
        
    except Exception as e:
        duration = time.time() - start_time
        logger.error("Agent invocation failed",
                    request_id=context.request_id,
                    duration=duration,
                    error=str(e),
                    success=False)
        raise
```

#### **セキュリティ強化**
```python
# 入力サニタイゼーション
import html
import re

def secure_agent(request, context):
    # HTML エスケープ
    prompt = html.escape(request.get("prompt", ""))
    
    # 危険なパターンの除去
    prompt = re.sub(r'<script.*?</script>', '', prompt, flags=re.IGNORECASE)
    
    # 長さ制限
    if len(prompt) > 10000:
        raise ValidationError("Prompt too long")
    
    return process_secure_request(prompt)
```

### 3. **Runtime パフォーマンス最適化**

#### **並行処理の活用**
```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

async def parallel_agent(request, context):
    # CPU集約的タスクは ThreadPoolExecutor
    with ThreadPoolExecutor(max_workers=4) as executor:
        cpu_task = executor.submit(cpu_intensive_task, request)
        
        # I/O集約的タスクは async/await
        io_task = asyncio.create_task(io_intensive_task(request))
        
        # 並行実行
        cpu_result = await asyncio.wrap_future(cpu_task)
        io_result = await io_task
        
        return combine_results(cpu_result, io_result)
```

#### **キャッシュ戦略**
```python
from functools import lru_cache
import redis

# メモリキャッシュ
@lru_cache(maxsize=1000)
def cached_model_call(prompt_hash):
    return expensive_model_call(prompt_hash)

# Redis キャッシュ
redis_client = redis.Redis(host='elasticache-endpoint')

def redis_cached_agent(request, context):
    prompt = request.get("prompt")
    cache_key = f"response:{hash(prompt)}"
    
    # キャッシュ確認
    cached_response = redis_client.get(cache_key)
    if cached_response:
        return json.loads(cached_response)
    
    # 新規処理
    response = process_request(request)
    
    # キャッシュ保存（1時間）
    redis_client.setex(cache_key, 3600, json.dumps(response))
    
    return response
```

### 4. **Runtime コスト最適化**

#### **リソース効率化**
```python
# 適切なメモリサイズ設定
runtime_config = RuntimeConfig(
    memory_size=512,  # 必要最小限
    timeout=30,       # 適切なタイムアウト
    reserved_concurrency=10  # 同時実行数制限
)
```

#### **使用量監視**
```python
def cost_aware_agent(request, context):
    # 処理開始時刻記録
    start_time = time.time()
    
    try:
        result = process_request(request)
        
        # コストメトリクス記録
        duration = time.time() - start_time
        context.metrics.put_metric("ProcessingDuration", duration, unit="Seconds")
        context.metrics.put_metric("RequestCost", calculate_cost(duration), unit="None")
        
        return result
        
    except Exception as e:
        # 失敗時もコスト記録
        duration = time.time() - start_time
        context.metrics.put_metric("FailedRequestCost", calculate_cost(duration), unit="None")
        raise
```

## トラブルシューティング

### Runtime 関連の問題と解決策

#### 1. **Runtime デプロイエラー**

**問題**: `Model access denied`
**解決策**: 
```bash
# Bedrock モデルアクセス権限を確認
aws bedrock list-foundation-models --region us-east-1

# IAM ポリシーを確認
aws iam get-role-policy --role-name AgentCoreExecutionRole --policy-name BedrockAccess
```

**問題**: `Runtime configuration invalid`
**解決策**:
```yaml
# 正しい .bedrock_agentcore.yaml 設定
runtime:
  entrypoint: "src/main.py"  # 正しいパス
  handler: "app"             # 正しいハンドラー名
  timeout: 300               # 適切なタイムアウト値
  memory_size: 1024          # 適切なメモリサイズ
```

#### 2. **Runtime パフォーマンス問題**

**問題**: `High cold start latency`
**解決策**:
```python
# グローバル初期化でコールドスタート最適化
import boto3

# コンテナレベルで初期化
bedrock_client = boto3.client('bedrock-runtime')
model_cache = {}

def optimized_agent(request, context):
    # 高速処理
    return process_with_preloaded_resources(request)
```

**問題**: `Memory limit exceeded`
**解決策**:
```python
# メモリ効率的な処理
def memory_efficient_agent(request, context):
    # ストリーミング処理
    for chunk in process_stream(request):
        yield chunk
        
    # 明示的なメモリ解放
    import gc
    gc.collect()
```

#### 3. **Runtime セッション問題**

**問題**: `Session timeout errors`
**解決策**:
```yaml
# セッション設定の調整
session:
  timeout: 3600      # 1時間に延長
  max_concurrent: 50 # 同時セッション数制限
  cleanup_interval: 300  # 5分間隔でクリーンアップ
```

**問題**: `Session state corruption`
**解決策**:
```python
def robust_session_agent(request, context):
    session_id = request.get("session_id")
    
    try:
        # セッション状態の検証
        session_data = context.session_manager.get(session_id)
        if not validate_session_data(session_data):
            # セッション再初期化
            context.session_manager.reset(session_id)
            
    except SessionError:
        # 新しいセッション作成
        session_id = context.session_manager.create_new()
        
    return process_with_session(request, session_id)
```

#### 4. **Runtime セキュリティ問題**

**問題**: `Authentication failures`
**解決策**:
```python
from bedrock_agentcore.security import JWTValidator

# JWT 設定の確認
jwt_validator = JWTValidator(
    issuer="https://correct-issuer.com",
    audience="correct-audience",
    algorithms=["RS256"],  # 正しいアルゴリズム
    verify_exp=True        # 有効期限チェック
)
```

**問題**: `Rate limit exceeded`
**解決策**:
```python
# レート制限の調整
from bedrock_agentcore.security import RateLimiter

rate_limiter = RateLimiter(
    requests_per_minute=200,  # 制限緩和
    burst_limit=20,           # バースト制限
    key_function=lambda req: req.get("user_id", "anonymous")
)
```

#### 5. **Runtime 監視問題**

**問題**: `Missing metrics data`
**解決策**:
```python
# 適切なメトリクス送信
from bedrock_agentcore.monitoring import MetricsCollector

def monitored_agent(request, context):
    metrics = MetricsCollector()
    
    start_time = time.time()
    
    try:
        result = process_request(request)
        
        # 成功メトリクス
        duration = time.time() - start_time
        metrics.put_metric("ProcessingTime", duration, unit="Seconds")
        metrics.put_metric("SuccessCount", 1, unit="Count")
        
        return result
        
    except Exception as e:
        # エラーメトリクス
        metrics.put_metric("ErrorCount", 1, unit="Count")
        metrics.put_metric("ErrorType", str(type(e).__name__), unit="None")
        raise
```

#### 6. **Runtime ログ問題**

**問題**: `Log data not appearing in CloudWatch`
**解決策**:
```python
import logging
import json

# 構造化ログの設定
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s %(levelname)s %(message)s'
)

def well_logged_agent(request, context):
    logger = logging.getLogger(__name__)
    
    # JSON 形式でログ出力
    log_data = {
        "request_id": context.request_id,
        "timestamp": context.timestamp,
        "event": "agent_invocation"
    }
    
    logger.info(json.dumps(log_data))
    
    return process_request(request)
```

### Runtime デバッグのヒント

#### 1. **ローカルデバッグ**
```bash
# 詳細ログでローカル実行
uv run agentcore dev --log-level DEBUG

# 特定のリクエストでテスト
uv run agentcore invoke --dev '{"prompt": "test", "debug": true}'
```

#### 2. **本番デバッグ**
```bash
# リアルタイムログ確認
uv run agentcore logs --follow --filter ERROR

# メトリクス確認
uv run agentcore metrics --start-time 2024-01-01T00:00:00Z
```

#### 3. **パフォーマンス分析**
```python
import cProfile
import pstats

def profiled_agent(request, context):
    profiler = cProfile.Profile()
    profiler.enable()
    
    try:
        result = process_request(request)
        return result
    finally:
        profiler.disable()
        
        # プロファイル結果をログ出力
        stats = pstats.Stats(profiler)
        stats.sort_stats('cumulative')
        
        # 上位10個の関数を記録
        context.logger.info("Performance profile", extra={
            "top_functions": stats.get_stats_profile()
        })
```

## まとめ

Amazon Bedrock AgentCore は、生成AI エージェントの開発から運用まで、包括的なソリューションを提供するプラットフォームです。

**主要な価値:**
- **開発効率**: 統合開発環境による高速開発
- **運用簡素化**: 自動化された運用機能
- **スケーラビリティ**: AWS インフラによる柔軟なスケーリング
- **セキュリティ**: エンタープライズグレードのセキュリティ

**適用場面:**
- カスタマーサポートボット
- 社内アシスタント
- 専門知識エージェント
- マルチモーダルアプリケーション

AgentCore を使用することで、開発者は複雑なインフラ管理から解放され、エージェントのコアロジックに集中できます。