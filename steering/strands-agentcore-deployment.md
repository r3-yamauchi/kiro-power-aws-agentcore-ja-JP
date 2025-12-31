---
inclusion: manual
---

# Strands Agent を AgentCore Runtime にデプロイする完全ガイド

このステアリングファイルは、Strands Agent フレームワークで作成したエージェントを Amazon Bedrock AgentCore Runtime にデプロイして実行する詳細な手順を説明します。

## 概要

Strands は AWS が開発したエージェントフレームワークで、AgentCore Runtime との統合により、スケーラブルで本番対応のエージェントアプリケーションを構築できます。

### Strands Agent の特徴

- **宣言的設定**: YAML ベースの設定ファイル
- **モジュラー設計**: 再利用可能なコンポーネント
- **型安全性**: TypeScript/Python での強力な型システム
- **テスト支援**: 組み込みテストフレームワーク
- **AgentCore 統合**: ネイティブな AgentCore サポート

## 前提条件

### 1. 開発環境の準備

```bash
# Python 環境（推奨：uv を使用）
uv init strands-agent-project
cd strands-agent-project

# 必要なパッケージをインストール
uv add bedrock-agentcore-starter-toolkit
uv add strands-agent-framework
uv add boto3
```

### Strands Agents MCP サーバーの活用

Strands Agent 開発時には、利用可能な `strands-agents` MCP サーバーを活用してドキュメントを検索できます。このサーバーは Strands Agents の公式ドキュメントへのアクセスを提供し、開発中の疑問解決に役立ちます。

**利用可能なツール:**
- `search_docs`: Strands 関連ドキュメントの検索
- `fetch_doc`: 特定のドキュメントの取得

**使用例:**
```
# Strands SDK のツール作成に関するドキュメントを検索
kiroPowers({
  "action": "use",
  "powerName": "strands-agents", 
  "serverName": "strands-agents",
  "toolName": "search_docs",
  "arguments": {
    "query": "tool creation strands sdk"
  }
})

# MCP 統合の具体的な実装例を検索
kiroPowers({
  "action": "use",
  "powerName": "strands-agents",
  "serverName": "strands-agents", 
  "toolName": "search_docs",
  "arguments": {
    "query": "mcp integration examples"
  }
})
```

### 2. AWS 認証情報の設定

```bash
# AWS CLI の設定
aws configure

# または環境変数で設定
export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_DEFAULT_REGION=us-east-1
```

### 3. 必要な IAM 権限

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream",
        "lambda:CreateFunction",
        "lambda:UpdateFunctionCode",
        "lambda:InvokeFunction",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

## Strands Agent の作成

### 1. プロジェクト構造の作成

```bash
# プロジェクト構造
strands-agent-project/
├── src/
│   ├── agent.py          # メインエージェントロジック
│   ├── config.yaml       # Strands 設定ファイル
│   └── handlers/         # カスタムハンドラー
├── tests/
│   └── test_agent.py     # テストファイル
├── requirements.txt      # 依存関係
└── .bedrock_agentcore.yaml  # AgentCore 設定
```

### 2. Strands Agent の実装

**src/agent.py**
```python
from strands import Agent, Message, Context
from strands.handlers import BedrockModelHandler
from strands.memory import ConversationMemory
from strands.tools import Tool
import json

class MyStrandsAgent(Agent):
    """Strands フレームワークを使用したエージェント実装"""
    
    def __init__(self):
        super().__init__()
        
        # モデルハンドラーの設定
        self.model_handler = BedrockModelHandler(
            model_id="anthropic.claude-3-sonnet-20240229-v1:0",
            region="us-east-1"
        )
        
        # メモリの設定
        self.memory = ConversationMemory(
            max_history=10,
            summarize_threshold=20
        )
        
        # ツールの登録
        self.register_tools([
            WeatherTool(),
            CalculatorTool(),
            SearchTool()
        ])
    
    async def process_message(self, message: Message, context: Context) -> Message:
        """メッセージ処理のメインロジック"""
        
        # セッション管理
        session_id = context.session_id
        conversation_history = await self.memory.get_history(session_id)
        
        # ユーザーメッセージをメモリに追加
        await self.memory.add_message(session_id, message)
        
        # コンテキスト構築
        system_prompt = self._build_system_prompt()
        messages = self._build_message_history(conversation_history, message)
        
        # モデル呼び出し
        response = await self.model_handler.invoke(
            messages=messages,
            system_prompt=system_prompt,
            max_tokens=1000,
            temperature=0.7
        )
        
        # ツール呼び出しの処理
        if response.tool_calls:
            tool_results = await self._execute_tools(response.tool_calls)
            
            # ツール結果を含めて再度モデル呼び出し
            messages.append(response.to_message())
            messages.extend(tool_results)
            
            response = await self.model_handler.invoke(
                messages=messages,
                system_prompt=system_prompt,
                max_tokens=1000
            )
        
        # レスポンスをメモリに保存
        await self.memory.add_message(session_id, response.to_message())
        
        return response.to_message()
    
    def _build_system_prompt(self) -> str:
        """システムプロンプトの構築"""
        return """
        あなたは親切で知識豊富なAIアシスタントです。
        ユーザーの質問に正確で有用な回答を提供してください。
        
        利用可能なツール:
        - weather: 天気情報の取得
        - calculator: 数学計算の実行
        - search: インターネット検索
        
        ツールが必要な場合は適切に使用してください。
        """
    
    def _build_message_history(self, history: list, current_message: Message) -> list:
        """メッセージ履歴の構築"""
        messages = []
        
        # 履歴メッセージを追加
        for msg in history[-10:]:  # 最新10件のみ
            messages.append({
                "role": msg.role,
                "content": msg.content
            })
        
        # 現在のメッセージを追加
        messages.append({
            "role": current_message.role,
            "content": current_message.content
        })
        
        return messages
    
    async def _execute_tools(self, tool_calls: list) -> list:
        """ツール実行の処理"""
        results = []
        
        for tool_call in tool_calls:
            tool_name = tool_call.function.name
            tool_args = json.loads(tool_call.function.arguments)
            
            # ツール実行
            tool = self.get_tool(tool_name)
            if tool:
                result = await tool.execute(**tool_args)
                results.append(Message(
                    role="tool",
                    content=json.dumps(result),
                    tool_call_id=tool_call.id
                ))
        
        return results

# カスタムツールの実装
class WeatherTool(Tool):
    """天気情報取得ツール"""
    
    name = "weather"
    description = "指定された場所の現在の天気情報を取得します"
    
    parameters = {
        "type": "object",
        "properties": {
            "location": {
                "type": "string",
                "description": "天気を調べたい場所（例：東京、大阪）"
            }
        },
        "required": ["location"]
    }
    
    async def execute(self, location: str) -> dict:
        """天気情報の取得（実際の実装では外部APIを呼び出し）"""
        # 実際の実装では OpenWeatherMap API などを使用
        return {
            "location": location,
            "temperature": "22°C",
            "condition": "晴れ",
            "humidity": "60%"
        }

class CalculatorTool(Tool):
    """計算ツール"""
    
    name = "calculator"
    description = "数学的な計算を実行します"
    
    parameters = {
        "type": "object",
        "properties": {
            "expression": {
                "type": "string",
                "description": "計算式（例：2 + 3 * 4）"
            }
        },
        "required": ["expression"]
    }
    
    async def execute(self, expression: str) -> dict:
        """安全な計算の実行"""
        try:
            # 安全な評価（実際の実装ではより厳密な検証が必要）
            result = eval(expression)
            return {
                "expression": expression,
                "result": result
            }
        except Exception as e:
            return {
                "expression": expression,
                "error": str(e)
            }

class SearchTool(Tool):
    """検索ツール"""
    
    name = "search"
    description = "インターネット検索を実行します"
    
    parameters = {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "検索クエリ"
            }
        },
        "required": ["query"]
    }
    
    async def execute(self, query: str) -> dict:
        """検索の実行（実際の実装では検索APIを使用）"""
        # 実際の実装では Google Search API や Bing Search API を使用
        return {
            "query": query,
            "results": [
                {
                    "title": f"{query} に関する情報",
                    "url": "https://example.com",
                    "snippet": f"{query} についての詳細な情報です。"
                }
            ]
        }

# AgentCore 統合用のエントリポイント
agent_instance = MyStrandsAgent()

async def handler(event, context):
    """AgentCore Runtime 用のハンドラー"""
    
    # リクエストの解析
    request_data = json.loads(event.get('body', '{}'))
    
    # メッセージオブジェクトの作成
    message = Message(
        role="user",
        content=request_data.get("prompt", "")
    )
    
    # コンテキストの作成
    agent_context = Context(
        session_id=request_data.get("session_id", "default"),
        user_id=request_data.get("user_id", "anonymous"),
        metadata=request_data.get("metadata", {})
    )
    
    try:
        # エージェント処理
        response_message = await agent_instance.process_message(message, agent_context)
        
        return {
            "statusCode": 200,
            "body": json.dumps({
                "response": response_message.content,
                "session_id": agent_context.session_id
            })
        }
        
    except Exception as e:
        return {
            "statusCode": 500,
            "body": json.dumps({
                "error": str(e),
                "error_type": type(e).__name__
            })
        }
```

### 3. Strands 設定ファイル

**src/config.yaml**
```yaml
# Strands Agent 設定
agent:
  name: "MyStrandsAgent"
  version: "1.0.0"
  description: "Strands フレームワークを使用したサンプルエージェント"

# モデル設定
models:
  primary:
    provider: "bedrock"
    model_id: "anthropic.claude-3-sonnet-20240229-v1:0"
    region: "us-east-1"
    parameters:
      max_tokens: 1000
      temperature: 0.7
      top_p: 0.9

# メモリ設定
memory:
  type: "conversation"
  max_history: 10
  summarize_threshold: 20
  storage:
    type: "dynamodb"
    table_name: "strands-agent-memory"

# ツール設定
tools:
  - name: "weather"
    enabled: true
    config:
      api_key: "${WEATHER_API_KEY}"
  
  - name: "calculator"
    enabled: true
    
  - name: "search"
    enabled: true
    config:
      api_key: "${SEARCH_API_KEY}"

# セキュリティ設定
security:
  input_validation: true
  output_filtering: true
  rate_limiting:
    requests_per_minute: 100
    burst_limit: 10

# ログ設定
logging:
  level: "INFO"
  structured: true
  include_request_id: true
```

## AgentCore Runtime への統合

### 1. AgentCore 設定ファイル

**.bedrock_agentcore.yaml**
```yaml
# AgentCore Runtime 設定
runtime:
  entrypoint: "src/agent.py"
  handler: "handler"
  
  # ランタイム環境
  python_version: "3.11"
  timeout: 300
  memory_size: 1024
  
  # 環境変数
  environment:
    LOG_LEVEL: "INFO"
    STRANDS_CONFIG_PATH: "src/config.yaml"
    WEATHER_API_KEY: "${WEATHER_API_KEY}"
    SEARCH_API_KEY: "${SEARCH_API_KEY}"

# メモリ統合
memory:
  mode: "PERSISTENT"
  storage_type: "DYNAMODB"
  table_name: "strands-agent-memory"
  ttl_seconds: 86400

# ゲートウェイ統合
gateway:
  enabled: true
  authentication:
    type: "JWT"
    issuer: "https://your-auth-provider.com"
  cors:
    enabled: true
    allowed_origins: ["*"]
  rate_limiting:
    requests_per_minute: 100

# 監視設定
monitoring:
  metrics_enabled: true
  tracing_enabled: true
  custom_metrics:
    - name: "StrandsAgentInvocations"
      unit: "Count"
    - name: "ToolExecutions"
      unit: "Count"
```

### 2. 依存関係の定義

**requirements.txt**
```txt
bedrock-agentcore-starter-toolkit>=1.0.0
strands-agent-framework>=2.0.0
boto3>=1.34.0
pydantic>=2.0.0
asyncio-mqtt>=0.13.0
structlog>=23.0.0
```

## デプロイ手順

### 1. ローカル開発とテスト

```bash
# 開発環境の起動
uv run agentcore dev --config .bedrock_agentcore.yaml

# 別ターミナルでテスト
uv run agentcore invoke --dev '{
  "prompt": "東京の天気を教えて",
  "session_id": "test-session-001"
}'

# ツール機能のテスト
uv run agentcore invoke --dev '{
  "prompt": "2 + 3 * 4 を計算して",
  "session_id": "test-session-002"
}'

# 会話継続のテスト
uv run agentcore invoke --dev '{
  "prompt": "前の計算結果に10を足して",
  "session_id": "test-session-002"
}'
```

### 2. 単体テストの実行

**tests/test_agent.py**
```python
import pytest
import asyncio
from src.agent import MyStrandsAgent, Message, Context

@pytest.fixture
def agent():
    return MyStrandsAgent()

@pytest.fixture
def sample_context():
    return Context(
        session_id="test-session",
        user_id="test-user",
        metadata={}
    )

@pytest.mark.asyncio
async def test_simple_conversation(agent, sample_context):
    """基本的な会話のテスト"""
    message = Message(role="user", content="こんにちは")
    
    response = await agent.process_message(message, sample_context)
    
    assert response.role == "assistant"
    assert len(response.content) > 0

@pytest.mark.asyncio
async def test_weather_tool(agent, sample_context):
    """天気ツールのテスト"""
    message = Message(role="user", content="東京の天気を教えて")
    
    response = await agent.process_message(message, sample_context)
    
    assert "天気" in response.content or "気温" in response.content

@pytest.mark.asyncio
async def test_calculator_tool(agent, sample_context):
    """計算ツールのテスト"""
    message = Message(role="user", content="2 + 3 を計算して")
    
    response = await agent.process_message(message, sample_context)
    
    assert "5" in response.content

@pytest.mark.asyncio
async def test_conversation_memory(agent, sample_context):
    """会話メモリのテスト"""
    # 最初のメッセージ
    message1 = Message(role="user", content="私の名前は田中です")
    await agent.process_message(message1, sample_context)
    
    # 名前を覚えているかテスト
    message2 = Message(role="user", content="私の名前は何ですか？")
    response = await agent.process_message(message2, sample_context)
    
    assert "田中" in response.content

# テスト実行
if __name__ == "__main__":
    pytest.main([__file__, "-v"])
```

```bash
# テストの実行
uv run pytest tests/ -v

# カバレッジ付きテスト
uv run pytest tests/ --cov=src --cov-report=html
```

### 3. 本番デプロイ

```bash
# 1. 設定の確認
uv run agentcore configure \
  --entrypoint src/agent.py \
  --handler handler \
  --region us-east-1 \
  --memory-mode PERSISTENT \
  --timeout 300

# 2. デプロイの実行
uv run agentcore launch --region us-east-1

# 3. デプロイ状況の確認
uv run agentcore status --region us-east-1

# 4. 本番環境でのテスト
uv run agentcore invoke '{
  "prompt": "こんにちは、Strands エージェントです",
  "session_id": "prod-test-001"
}' --region us-east-1
```

### 4. 本番環境での動作確認

```bash
# 基本機能のテスト
uv run agentcore invoke '{
  "prompt": "あなたの機能を教えて",
  "session_id": "feature-test"
}' --region us-east-1

# ツール機能のテスト
uv run agentcore invoke '{
  "prompt": "大阪の天気と、10 + 20 の計算結果を教えて",
  "session_id": "tool-test"
}' --region us-east-1

# セッション継続のテスト
uv run agentcore invoke '{
  "prompt": "私の名前は山田です",
  "session_id": "memory-test"
}' --region us-east-1

uv run agentcore invoke '{
  "prompt": "私の名前を覚えていますか？",
  "session_id": "memory-test"
}' --region us-east-1
```

## 高度な統合パターン

### 1. カスタムメモリプロバイダー

```python
from strands.memory import MemoryProvider
import boto3

class DynamoDBMemoryProvider(MemoryProvider):
    """DynamoDB を使用したカスタムメモリプロバイダー"""
    
    def __init__(self, table_name: str, region: str = "us-east-1"):
        self.dynamodb = boto3.resource('dynamodb', region_name=region)
        self.table = self.dynamodb.Table(table_name)
    
    async def get_history(self, session_id: str) -> list:
        """セッション履歴の取得"""
        response = self.table.get_item(
            Key={'session_id': session_id}
        )
        
        if 'Item' in response:
            return response['Item'].get('messages', [])
        return []
    
    async def add_message(self, session_id: str, message: Message):
        """メッセージの追加"""
        # 既存履歴を取得
        history = await self.get_history(session_id)
        
        # 新しいメッセージを追加
        history.append({
            'role': message.role,
            'content': message.content,
            'timestamp': message.timestamp
        })
        
        # 履歴の制限（最新50件のみ保持）
        if len(history) > 50:
            history = history[-50:]
        
        # DynamoDB に保存
        self.table.put_item(
            Item={
                'session_id': session_id,
                'messages': history,
                'updated_at': int(time.time())
            }
        )
```

### 2. カスタム認証プロバイダー

```python
from strands.security import AuthProvider
import jwt

class JWTAuthProvider(AuthProvider):
    """JWT 認証プロバイダー"""
    
    def __init__(self, secret_key: str, algorithm: str = "HS256"):
        self.secret_key = secret_key
        self.algorithm = algorithm
    
    async def authenticate(self, request_context: dict) -> dict:
        """JWT トークンの検証"""
        auth_header = request_context.get('headers', {}).get('Authorization')
        
        if not auth_header or not auth_header.startswith('Bearer '):
            raise AuthenticationError("Missing or invalid authorization header")
        
        token = auth_header[7:]  # "Bearer " を除去
        
        try:
            payload = jwt.decode(
                token, 
                self.secret_key, 
                algorithms=[self.algorithm]
            )
            
            return {
                'user_id': payload.get('sub'),
                'permissions': payload.get('permissions', []),
                'expires_at': payload.get('exp')
            }
            
        except jwt.ExpiredSignatureError:
            raise AuthenticationError("Token has expired")
        except jwt.InvalidTokenError:
            raise AuthenticationError("Invalid token")
```

### 3. カスタムツールの実装

```python
from strands.tools import Tool
import aiohttp
import json

class WebSearchTool(Tool):
    """Web 検索ツール（実際の検索API使用）"""
    
    name = "web_search"
    description = "インターネット上の情報を検索します"
    
    parameters = {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "検索したいキーワード"
            },
            "num_results": {
                "type": "integer",
                "description": "取得する結果の数（デフォルト：5）",
                "default": 5
            }
        },
        "required": ["query"]
    }
    
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.bing.microsoft.com/v7.0/search"
    
    async def execute(self, query: str, num_results: int = 5) -> dict:
        """Bing Search API を使用した検索"""
        
        headers = {
            "Ocp-Apim-Subscription-Key": self.api_key
        }
        
        params = {
            "q": query,
            "count": num_results,
            "mkt": "ja-JP"
        }
        
        async with aiohttp.ClientSession() as session:
            async with session.get(
                self.base_url, 
                headers=headers, 
                params=params
            ) as response:
                
                if response.status == 200:
                    data = await response.json()
                    
                    results = []
                    for item in data.get("webPages", {}).get("value", []):
                        results.append({
                            "title": item.get("name"),
                            "url": item.get("url"),
                            "snippet": item.get("snippet")
                        })
                    
                    return {
                        "query": query,
                        "results": results,
                        "total_results": len(results)
                    }
                else:
                    return {
                        "query": query,
                        "error": f"Search API error: {response.status}",
                        "results": []
                    }

class DatabaseQueryTool(Tool):
    """データベースクエリツール"""
    
    name = "database_query"
    description = "データベースから情報を検索します"
    
    parameters = {
        "type": "object",
        "properties": {
            "table": {
                "type": "string",
                "description": "検索対象のテーブル名"
            },
            "conditions": {
                "type": "object",
                "description": "検索条件"
            }
        },
        "required": ["table"]
    }
    
    def __init__(self, connection_string: str):
        # 実際の実装では適切なデータベースクライアントを使用
        self.connection_string = connection_string
    
    async def execute(self, table: str, conditions: dict = None) -> dict:
        """データベースクエリの実行"""
        
        # セキュリティ：許可されたテーブルのみアクセス
        allowed_tables = ["products", "customers", "orders"]
        if table not in allowed_tables:
            return {
                "error": f"Access to table '{table}' is not allowed",
                "allowed_tables": allowed_tables
            }
        
        # 実際の実装では SQL インジェクション対策が必要
        try:
            # データベースクエリの実行（疑似コード）
            results = await self._execute_query(table, conditions)
            
            return {
                "table": table,
                "conditions": conditions,
                "results": results,
                "count": len(results)
            }
            
        except Exception as e:
            return {
                "table": table,
                "error": str(e),
                "results": []
            }
    
    async def _execute_query(self, table: str, conditions: dict) -> list:
        """実際のクエリ実行（疑似実装）"""
        # 実際の実装では適切なORMやクエリビルダーを使用
        return [
            {"id": 1, "name": "Sample Product", "price": 1000},
            {"id": 2, "name": "Another Product", "price": 2000}
        ]
```

## 監視とメンテナンス

### 1. カスタムメトリクスの実装

```python
from strands.monitoring import MetricsCollector
import time

class StrandsMetricsCollector(MetricsCollector):
    """Strands エージェント用のカスタムメトリクス"""
    
    def __init__(self):
        super().__init__()
        self.start_times = {}
    
    def start_operation(self, operation_name: str, session_id: str):
        """操作開始時刻の記録"""
        key = f"{operation_name}:{session_id}"
        self.start_times[key] = time.time()
    
    def end_operation(self, operation_name: str, session_id: str, success: bool = True):
        """操作終了とメトリクス送信"""
        key = f"{operation_name}:{session_id}"
        
        if key in self.start_times:
            duration = time.time() - self.start_times[key]
            
            # 処理時間メトリクス
            self.put_metric(f"{operation_name}Duration", duration, unit="Seconds")
            
            # 成功/失敗メトリクス
            if success:
                self.put_metric(f"{operation_name}Success", 1, unit="Count")
            else:
                self.put_metric(f"{operation_name}Failure", 1, unit="Count")
            
            del self.start_times[key]
    
    def record_tool_usage(self, tool_name: str, execution_time: float, success: bool):
        """ツール使用メトリクス"""
        self.put_metric("ToolExecution", 1, unit="Count", 
                       dimensions={"ToolName": tool_name})
        
        self.put_metric("ToolExecutionTime", execution_time, unit="Seconds",
                       dimensions={"ToolName": tool_name})
        
        if success:
            self.put_metric("ToolSuccess", 1, unit="Count",
                           dimensions={"ToolName": tool_name})
        else:
            self.put_metric("ToolFailure", 1, unit="Count",
                           dimensions={"ToolName": tool_name})

# エージェントでの使用例
class MonitoredStrandsAgent(MyStrandsAgent):
    """監視機能付きの Strands エージェント"""
    
    def __init__(self):
        super().__init__()
        self.metrics = StrandsMetricsCollector()
    
    async def process_message(self, message: Message, context: Context) -> Message:
        """監視機能付きメッセージ処理"""
        
        session_id = context.session_id
        self.metrics.start_operation("MessageProcessing", session_id)
        
        try:
            response = await super().process_message(message, context)
            self.metrics.end_operation("MessageProcessing", session_id, success=True)
            return response
            
        except Exception as e:
            self.metrics.end_operation("MessageProcessing", session_id, success=False)
            raise
    
    async def _execute_tools(self, tool_calls: list) -> list:
        """監視機能付きツール実行"""
        results = []
        
        for tool_call in tool_calls:
            tool_name = tool_call.function.name
            start_time = time.time()
            
            try:
                # ツール実行
                result = await super()._execute_single_tool(tool_call)
                
                # 成功メトリクス
                execution_time = time.time() - start_time
                self.metrics.record_tool_usage(tool_name, execution_time, success=True)
                
                results.append(result)
                
            except Exception as e:
                # 失敗メトリクス
                execution_time = time.time() - start_time
                self.metrics.record_tool_usage(tool_name, execution_time, success=False)
                
                # エラー結果を追加
                results.append(Message(
                    role="tool",
                    content=json.dumps({"error": str(e)}),
                    tool_call_id=tool_call.id
                ))
        
        return results
```

### 2. ログ管理

```python
import structlog
import json

# 構造化ログの設定
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    wrapper_class=structlog.stdlib.BoundLogger,
    cache_logger_on_first_use=True,
)

class LoggedStrandsAgent(MyStrandsAgent):
    """ログ機能強化版 Strands エージェント"""
    
    def __init__(self):
        super().__init__()
        self.logger = structlog.get_logger(__name__)
    
    async def process_message(self, message: Message, context: Context) -> Message:
        """ログ付きメッセージ処理"""
        
        # リクエストログ
        self.logger.info(
            "Message processing started",
            session_id=context.session_id,
            user_id=context.user_id,
            message_length=len(message.content),
            message_type=message.role
        )
        
        try:
            response = await super().process_message(message, context)
            
            # 成功ログ
            self.logger.info(
                "Message processing completed",
                session_id=context.session_id,
                response_length=len(response.content),
                success=True
            )
            
            return response
            
        except Exception as e:
            # エラーログ
            self.logger.error(
                "Message processing failed",
                session_id=context.session_id,
                error=str(e),
                error_type=type(e).__name__,
                success=False
            )
            raise
```

## トラブルシューティング

### よくある問題と解決策

#### 1. **Strands 設定エラー**
**問題**: `Invalid Strands configuration`
**解決策**:
```yaml
# config.yaml の検証
agent:
  name: "MyAgent"  # 必須フィールド
  version: "1.0.0"  # セマンティックバージョニング

models:
  primary:
    provider: "bedrock"  # 正しいプロバイダー名
    model_id: "anthropic.claude-3-sonnet-20240229-v1:0"  # 正しいモデルID
```

#### 2. **ツール実行エラー**
**問題**: `Tool execution failed`
**解決策**:
```python
# ツールのエラーハンドリング強化
async def execute(self, **kwargs) -> dict:
    try:
        # ツールロジック
        result = await self._execute_tool_logic(**kwargs)
        return {"success": True, "result": result}
        
    except Exception as e:
        self.logger.error(f"Tool execution failed: {e}")
        return {"success": False, "error": str(e)}
```

#### 3. **メモリ同期エラー**
**問題**: `Memory synchronization failed`
**解決策**:
```python
# メモリ操作の排他制御
import asyncio

class ThreadSafeMemory(ConversationMemory):
    def __init__(self):
        super().__init__()
        self._locks = {}
    
    async def add_message(self, session_id: str, message: Message):
        if session_id not in self._locks:
            self._locks[session_id] = asyncio.Lock()
        
        async with self._locks[session_id]:
            await super().add_message(session_id, message)
```

## まとめ

Strands Agent フレームワークと AgentCore Runtime の統合により、以下の利点が得られます：

### **開発効率**
- 宣言的設定による迅速な開発
- 豊富なビルトインコンポーネント
- 強力なテストフレームワーク

### **運用性**
- 自動スケーリング
- 包括的な監視
- 構造化ログ

### **拡張性**
- カスタムツールの簡単な追加
- プラグイン可能なアーキテクチャ
- マルチモデル対応

この統合により、エンタープライズグレードのAIエージェントを効率的に構築・運用できます。