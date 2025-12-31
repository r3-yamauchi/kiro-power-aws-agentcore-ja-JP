# AgentCore Gateway - Lambda MCP 化ガイド

## 概要

このガイドでは、既存の Lambda 関数を AgentCore Gateway 経由で MCP ツールとして利用できるように変換する方法を詳細に説明します。Lambda の MCP 化により、既存のビジネスロジックを AI エージェントが直接呼び出せるツールとして統合できます。

## Lambda MCP 化の利点

1. **既存資産の活用**: 既存の Lambda 関数をそのまま AI エージェントのツールとして利用
2. **統一されたインターフェース**: MCP プロトコルによる標準化されたツールアクセス
3. **スケーラビリティ**: Lambda の自動スケーリング機能をそのまま活用
4. **セキュリティ**: IAM ロールによる細かいアクセス制御
5. **コスト効率**: 使用量ベースの課金モデル

## MCP 化のプロセス

### ステップ 1: Lambda 関数の MCP 対応

既存の Lambda 関数を MCP ツールとして動作するように修正します。

#### 従来の Lambda 関数の例

```python
import json

def lambda_handler(event, context):
    # 従来の処理
    name = event.get('name', 'World')
    message = f"Hello, {name}!"
    
    return {
        'statusCode': 200,
        'body': json.dumps({'message': message})
    }
```

#### MCP 対応 Lambda 関数への変換

```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """
    MCP ツール呼び出し用の Lambda 関数
    
    期待されるイベント構造:
    {
        "tool_name": "greeting_tool",
        "arguments": {
            "name": "Alice",
            "language": "ja"
        }
    }
    """
    
    try:
        # MCP ツール呼び出しの解析
        tool_name = event.get('tool_name')
        arguments = event.get('arguments', {})
        
        logger.info(f"ツール呼び出し: {tool_name}, 引数: {arguments}")
        
        # ツール別の処理分岐
        if tool_name == 'greeting_tool':
            return handle_greeting_tool(arguments)
        elif tool_name == 'calculate_tool':
            return handle_calculate_tool(arguments)
        elif tool_name == 'data_query_tool':
            return handle_data_query_tool(arguments)
        else:
            return {
                "error": f"不明なツール: {tool_name}",
                "available_tools": ["greeting_tool", "calculate_tool", "data_query_tool"]
            }
            
    except Exception as e:
        logger.error(f"Lambda 実行エラー: {str(e)}")
        return {
            "error": f"内部エラー: {str(e)}",
            "tool_name": event.get('tool_name', 'unknown')
        }

def handle_greeting_tool(arguments):
    """挨拶ツールの処理"""
    name = arguments.get('name', 'World')
    language = arguments.get('language', 'en')
    
    greetings = {
        'en': f"Hello, {name}!",
        'ja': f"こんにちは、{name}さん！",
        'es': f"¡Hola, {name}!",
        'fr': f"Bonjour, {name}!"
    }
    
    message = greetings.get(language, greetings['en'])
    
    return {
        "greeting": message,
        "name": name,
        "language": language,
        "timestamp": context.aws_request_id
    }

def handle_calculate_tool(arguments):
    """計算ツールの処理"""
    operation = arguments.get('operation')
    a = arguments.get('a')
    b = arguments.get('b')
    
    if not all([operation, a is not None, b is not None]):
        return {
            "error": "必要なパラメータが不足しています",
            "required": ["operation", "a", "b"]
        }
    
    try:
        a, b = float(a), float(b)
        
        if operation == 'add':
            result = a + b
        elif operation == 'subtract':
            result = a - b
        elif operation == 'multiply':
            result = a * b
        elif operation == 'divide':
            if b == 0:
                return {"error": "ゼロ除算エラー"}
            result = a / b
        else:
            return {
                "error": f"サポートされていない演算: {operation}",
                "supported_operations": ["add", "subtract", "multiply", "divide"]
            }
        
        return {
            "result": result,
            "operation": operation,
            "operands": {"a": a, "b": b}
        }
        
    except ValueError:
        return {"error": "数値の変換に失敗しました"}

def handle_data_query_tool(arguments):
    """データクエリツールの処理"""
    query_type = arguments.get('query_type')
    filters = arguments.get('filters', {})
    
    # 模擬データベースクエリ
    mock_data = [
        {"id": 1, "name": "Alice", "department": "Engineering", "salary": 75000},
        {"id": 2, "name": "Bob", "department": "Marketing", "salary": 65000},
        {"id": 3, "name": "Charlie", "department": "Engineering", "salary": 80000},
        {"id": 4, "name": "Diana", "department": "Sales", "salary": 70000}
    ]
    
    if query_type == 'all':
        return {"data": mock_data, "count": len(mock_data)}
    elif query_type == 'filter':
        filtered_data = mock_data
        
        if 'department' in filters:
            filtered_data = [
                item for item in filtered_data 
                if item['department'].lower() == filters['department'].lower()
            ]
        
        if 'min_salary' in filters:
            filtered_data = [
                item for item in filtered_data 
                if item['salary'] >= int(filters['min_salary'])
            ]
        
        return {"data": filtered_data, "count": len(filtered_data)}
    else:
        return {
            "error": f"サポートされていないクエリタイプ: {query_type}",
            "supported_types": ["all", "filter"]
        }
```

### ステップ 2: ツールスキーマの定義

Lambda 関数用の詳細なツールスキーマを作成します。

```json
{
    "lambdaArn": "arn:aws:lambda:us-east-1:123456789:function:mcp-multi-tool",
    "toolSchema": {
        "inlinePayload": [
            {
                "name": "greeting_tool",
                "description": "多言語対応の挨拶メッセージを生成します",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "name": {
                            "type": "string",
                            "description": "挨拶する相手の名前"
                        },
                        "language": {
                            "type": "string",
                            "enum": ["en", "ja", "es", "fr"],
                            "description": "挨拶の言語 (en: 英語, ja: 日本語, es: スペイン語, fr: フランス語)",
                            "default": "en"
                        }
                    },
                    "required": ["name"]
                }
            },
            {
                "name": "calculate_tool",
                "description": "基本的な数学演算を実行します",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "operation": {
                            "type": "string",
                            "enum": ["add", "subtract", "multiply", "divide"],
                            "description": "実行する演算の種類"
                        },
                        "a": {
                            "type": "number",
                            "description": "最初の数値"
                        },
                        "b": {
                            "type": "number",
                            "description": "2番目の数値"
                        }
                    },
                    "required": ["operation", "a", "b"]
                }
            },
            {
                "name": "data_query_tool",
                "description": "従業員データベースからデータを検索します",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "query_type": {
                            "type": "string",
                            "enum": ["all", "filter"],
                            "description": "クエリの種類 (all: 全データ, filter: フィルタ検索)"
                        },
                        "filters": {
                            "type": "object",
                            "properties": {
                                "department": {
                                    "type": "string",
                                    "description": "部署名でフィルタ"
                                },
                                "min_salary": {
                                    "type": "number",
                                    "description": "最小給与でフィルタ"
                                }
                            },
                            "description": "フィルタ条件 (query_type が filter の場合に使用)"
                        }
                    },
                    "required": ["query_type"]
                }
            }
        ]
    }
}
```

### ステップ 3: Gateway ターゲットの作成

MCP 対応 Lambda 関数を Gateway ターゲットとして登録します。

```bash
# 環境変数の設定
export GATEWAY_ARN="arn:aws:bedrock-agentcore:us-east-1:123456789:gateway/MyGateway"
export GATEWAY_URL="https://gateway-id.gateway.bedrock-agentcore.us-east-1.amazonaws.com"
export GATEWAY_ROLE_ARN="arn:aws:iam::123456789:role/AgentCoreGatewayRole"
export LAMBDA_ARN="arn:aws:lambda:us-east-1:123456789:function:mcp-multi-tool"

# ターゲットの作成
agentcore gateway create-mcp-gateway-target \
    --gateway-arn $GATEWAY_ARN \
    --gateway-url $GATEWAY_URL \
    --role-arn $GATEWAY_ROLE_ARN \
    --name MultiToolLambdaTarget \
    --target-type lambda \
    --target-payload "$(cat multi-tool-schema.json)" \
    --region us-east-1
```

### ステップ 4: Lambda 関数のデプロイ

```bash
# Lambda 関数のパッケージ化
zip -r mcp-multi-tool.zip lambda_function.py

# Lambda 関数の作成
aws lambda create-function \
    --function-name mcp-multi-tool \
    --runtime python3.9 \
    --role arn:aws:iam::123456789:role/LambdaExecutionRole \
    --handler lambda_function.lambda_handler \
    --zip-file fileb://mcp-multi-tool.zip \
    --timeout 30 \
    --memory-size 256 \
    --region us-east-1

# 関数の更新（既存の場合）
aws lambda update-function-code \
    --function-name mcp-multi-tool \
    --zip-file fileb://mcp-multi-tool.zip \
    --region us-east-1
```

## 高度な Lambda MCP パターン

### パターン 1: データベース統合 Lambda

```python
import json
import boto3
import pymysql
import os
from typing import Dict, Any

# DynamoDB クライアント
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ.get('DYNAMODB_TABLE', 'employees'))

def lambda_handler(event, context):
    tool_name = event.get('tool_name')
    arguments = event.get('arguments', {})
    
    if tool_name == 'query_employee':
        return query_employee(arguments)
    elif tool_name == 'update_employee':
        return update_employee(arguments)
    elif tool_name == 'create_employee':
        return create_employee(arguments)
    else:
        return {"error": f"不明なツール: {tool_name}"}

def query_employee(arguments: Dict[str, Any]) -> Dict[str, Any]:
    """従業員情報を検索"""
    employee_id = arguments.get('employee_id')
    
    if not employee_id:
        return {"error": "employee_id が必要です"}
    
    try:
        response = table.get_item(Key={'employee_id': employee_id})
        
        if 'Item' in response:
            return {
                "employee": dict(response['Item']),
                "found": True
            }
        else:
            return {
                "error": f"従業員 ID {employee_id} が見つかりません",
                "found": False
            }
    except Exception as e:
        return {"error": f"データベースエラー: {str(e)}"}

def update_employee(arguments: Dict[str, Any]) -> Dict[str, Any]:
    """従業員情報を更新"""
    employee_id = arguments.get('employee_id')
    updates = arguments.get('updates', {})
    
    if not employee_id or not updates:
        return {"error": "employee_id と updates が必要です"}
    
    try:
        # 更新式の構築
        update_expression = "SET "
        expression_values = {}
        
        for key, value in updates.items():
            update_expression += f"{key} = :{key}, "
            expression_values[f":{key}"] = value
        
        update_expression = update_expression.rstrip(", ")
        
        response = table.update_item(
            Key={'employee_id': employee_id},
            UpdateExpression=update_expression,
            ExpressionAttributeValues=expression_values,
            ReturnValues="ALL_NEW"
        )
        
        return {
            "updated_employee": dict(response['Attributes']),
            "success": True
        }
    except Exception as e:
        return {"error": f"更新エラー: {str(e)}"}
```

### パターン 2: 外部 API 統合 Lambda

```python
import json
import requests
import os
from typing import Dict, Any

def lambda_handler(event, context):
    tool_name = event.get('tool_name')
    arguments = event.get('arguments', {})
    
    if tool_name == 'get_weather':
        return get_weather(arguments)
    elif tool_name == 'send_notification':
        return send_notification(arguments)
    else:
        return {"error": f"不明なツール: {tool_name}"}

def get_weather(arguments: Dict[str, Any]) -> Dict[str, Any]:
    """天気情報を取得"""
    city = arguments.get('city')
    api_key = os.environ.get('WEATHER_API_KEY')
    
    if not city:
        return {"error": "city パラメータが必要です"}
    
    if not api_key:
        return {"error": "Weather API キーが設定されていません"}
    
    try:
        url = f"http://api.openweathermap.org/data/2.5/weather"
        params = {
            'q': city,
            'appid': api_key,
            'units': 'metric',
            'lang': 'ja'
        }
        
        response = requests.get(url, params=params, timeout=10)
        response.raise_for_status()
        
        data = response.json()
        
        return {
            "city": data['name'],
            "country": data['sys']['country'],
            "temperature": data['main']['temp'],
            "feels_like": data['main']['feels_like'],
            "humidity": data['main']['humidity'],
            "description": data['weather'][0]['description'],
            "wind_speed": data['wind']['speed']
        }
        
    except requests.RequestException as e:
        return {"error": f"天気 API エラー: {str(e)}"}
    except KeyError as e:
        return {"error": f"レスポンス解析エラー: {str(e)}"}

def send_notification(arguments: Dict[str, Any]) -> Dict[str, Any]:
    """Slack 通知を送信"""
    message = arguments.get('message')
    channel = arguments.get('channel', '#general')
    webhook_url = os.environ.get('SLACK_WEBHOOK_URL')
    
    if not message:
        return {"error": "message パラメータが必要です"}
    
    if not webhook_url:
        return {"error": "Slack Webhook URL が設定されていません"}
    
    try:
        payload = {
            'channel': channel,
            'text': message,
            'username': 'AgentCore Bot'
        }
        
        response = requests.post(
            webhook_url,
            json=payload,
            timeout=10
        )
        response.raise_for_status()
        
        return {
            "success": True,
            "message": "通知を送信しました",
            "channel": channel
        }
        
    except requests.RequestException as e:
        return {"error": f"Slack 通知エラー: {str(e)}"}
```

## Lambda MCP 化のベストプラクティス

### 1. エラーハンドリング

```python
import logging
import traceback

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    try:
        # メイン処理
        return process_tool_request(event, context)
    except Exception as e:
        # 詳細なエラーログ
        logger.error(f"Lambda エラー: {str(e)}")
        logger.error(f"スタックトレース: {traceback.format_exc()}")
        
        return {
            "error": f"内部エラー: {str(e)}",
            "request_id": context.aws_request_id,
            "tool_name": event.get('tool_name', 'unknown')
        }
```

### 2. 入力検証

```python
from typing import Dict, Any, List

def validate_arguments(arguments: Dict[str, Any], required_fields: List[str]) -> Dict[str, Any]:
    """引数の検証"""
    missing_fields = [field for field in required_fields if field not in arguments]
    
    if missing_fields:
        return {
            "valid": False,
            "error": f"必要なフィールドが不足しています: {missing_fields}",
            "required_fields": required_fields
        }
    
    return {"valid": True}

def handle_greeting_tool(arguments):
    # 入力検証
    validation = validate_arguments(arguments, ['name'])
    if not validation['valid']:
        return validation
    
    # 処理続行
    name = arguments['name']
    # ...
```

### 3. パフォーマンス最適化

```python
import boto3
from functools import lru_cache

# 接続の再利用
@lru_cache(maxsize=1)
def get_dynamodb_table():
    """DynamoDB テーブル接続をキャッシュ"""
    dynamodb = boto3.resource('dynamodb')
    return dynamodb.Table(os.environ['DYNAMODB_TABLE'])

# 設定のキャッシュ
@lru_cache(maxsize=10)
def get_config(key: str) -> str:
    """設定値をキャッシュ"""
    return os.environ.get(key)
```

### 4. セキュリティ

```python
import boto3
import json

def get_secret(secret_name: str) -> Dict[str, Any]:
    """AWS Secrets Manager からシークレットを取得"""
    client = boto3.client('secretsmanager')
    
    try:
        response = client.get_secret_value(SecretId=secret_name)
        return json.loads(response['SecretString'])
    except Exception as e:
        logger.error(f"シークレット取得エラー: {str(e)}")
        raise

def handle_secure_operation(arguments):
    # シークレットの取得
    secrets = get_secret('my-app-secrets')
    api_key = secrets['api_key']
    
    # セキュアな処理
    # ...
```

## 実践的な活用例

### 例: 既存ビジネスシステムの MCP 化

**元の注文処理 Lambda:**
```python
# 従来の注文処理 Lambda
import json
import boto3

def lambda_handler(event, context):
    order_data = json.loads(event['body'])
    
    # 注文処理ロジック
    order_id = process_order(order_data)
    
    return {
        'statusCode': 200,
        'body': json.dumps({'order_id': order_id})
    }
```

**MCP 対応版:**
```python
# MCP 対応注文処理 Lambda
import json
import boto3
import logging
from datetime import datetime
from typing import Dict, Any

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """注文管理システムの MCP ツール"""
    
    try:
        tool_name = event.get('tool_name')
        arguments = event.get('arguments', {})
        
        logger.info(f"注文システムツール呼び出し: {tool_name}")
        
        if tool_name == 'create_order':
            return create_order(arguments)
        elif tool_name == 'get_order_status':
            return get_order_status(arguments)
        elif tool_name == 'cancel_order':
            return cancel_order(arguments)
        elif tool_name == 'list_orders':
            return list_orders(arguments)
        else:
            return {
                "error": f"不明なツール: {tool_name}",
                "available_tools": ["create_order", "get_order_status", "cancel_order", "list_orders"]
            }
            
    except Exception as e:
        logger.error(f"注文システムエラー: {str(e)}")
        return {"error": f"システムエラー: {str(e)}"}

def create_order(arguments: Dict[str, Any]) -> Dict[str, Any]:
    """新規注文の作成"""
    customer_id = arguments.get('customer_id')
    items = arguments.get('items', [])
    
    if not customer_id or not items:
        return {
            "error": "customer_id と items が必要です",
            "required_fields": ["customer_id", "items"]
        }
    
    # 注文処理ロジック
    order_id = f"ORD-{datetime.now().strftime('%Y%m%d%H%M%S')}"
    total_amount = sum(item.get('price', 0) * item.get('quantity', 1) for item in items)
    
    # DynamoDB への保存（実装例）
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('orders')
    
    order_data = {
        'order_id': order_id,
        'customer_id': customer_id,
        'items': items,
        'total_amount': total_amount,
        'status': 'pending',
        'created_at': datetime.now().isoformat()
    }
    
    table.put_item(Item=order_data)
    
    return {
        "order_id": order_id,
        "total_amount": total_amount,
        "status": "pending",
        "message": "注文が正常に作成されました"
    }

def get_order_status(arguments: Dict[str, Any]) -> Dict[str, Any]:
    """注文状況の確認"""
    order_id = arguments.get('order_id')
    
    if not order_id:
        return {"error": "order_id が必要です"}
    
    # DynamoDB から注文情報を取得
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('orders')
    
    try:
        response = table.get_item(Key={'order_id': order_id})
        
        if 'Item' in response:
            order = response['Item']
            return {
                "order_id": order['order_id'],
                "status": order['status'],
                "total_amount": order['total_amount'],
                "created_at": order['created_at'],
                "items_count": len(order['items'])
            }
        else:
            return {"error": f"注文 {order_id} が見つかりません"}
            
    except Exception as e:
        return {"error": f"データベースエラー: {str(e)}"}
```

## テストとデバッグ

### ローカルテスト

```python
# test_lambda.py
import json
from lambda_function import lambda_handler

def test_greeting_tool():
    event = {
        "tool_name": "greeting_tool",
        "arguments": {
            "name": "Alice",
            "language": "ja"
        }
    }
    
    class MockContext:
        aws_request_id = "test-request-id"
    
    result = lambda_handler(event, MockContext())
    print(json.dumps(result, indent=2, ensure_ascii=False))

if __name__ == "__main__":
    test_greeting_tool()
```

### Gateway 経由でのテスト

```bash
# Gateway ツールのテスト
agentcore invoke --dev '{
    "prompt": "greeting_tool を使って Alice さんに日本語で挨拶してください"
}'

# 計算ツールのテスト
agentcore invoke --dev '{
    "prompt": "calculate_tool を使って 15 と 25 を足し算してください"
}'
```

## Lambda 関数の要件

Lambda 関数は MCP ツール呼び出しを処理する必要があります。以下は基本的な要件です：

### 基本的な MCP 対応 Lambda 関数

```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """
    MCP ツール呼び出し用の基本的な Lambda 関数
    
    イベント構造:
    {
        "tool_name": "my_tool",
        "arguments": {
            "param1": "value1",
            "param2": 123
        }
    }
    """
    
    try:
        tool_name = event.get('tool_name')
        arguments = event.get('arguments', {})
        
        logger.info(f"ツール呼び出し: {tool_name}")
        
        if tool_name == 'my_tool':
            param1 = arguments.get('param1')
            # リクエストを処理
            result = {"status": "success", "data": f"Processed {param1}"}
            return result
        else:
            return {"error": f"不明なツール: {tool_name}"}
            
    except Exception as e:
        logger.error(f"Lambda エラー: {str(e)}")
        return {"error": f"内部エラー: {str(e)}"}
```

### 必須要件

1. **イベント構造の対応**: `tool_name` と `arguments` フィールドを処理
2. **エラーハンドリング**: 適切なエラーレスポンスの返却
3. **ログ出力**: CloudWatch Logs への適切なログ記録
4. **タイムアウト設定**: 適切なタイムアウト値の設定（推奨: 30秒以下）
5. **IAM 権限**: 必要な AWS リソースへのアクセス権限

### 推奨事項

1. **入力検証**: 引数の妥当性チェック
2. **構造化レスポンス**: 一貫したレスポンス形式
3. **パフォーマンス最適化**: 接続の再利用とキャッシュ
4. **セキュリティ**: 機密情報の適切な管理

## 次のステップ

Lambda MCP 化が完了したら、以下のガイドも参照してください：

- **OpenAPI MCP 変換**: `agentcore-openapi-mcp-guide.md` - REST API の MCP 変換
- **セマンティック検索**: `agentcore-semantic-search-guide.md` - インテリジェントなツール選択
- **運用・トラブルシューティング**: `agentcore-gateway-operations.md` - 運用と問題解決

---

**注意**: Lambda 関数を MCP ツールとして使用する場合、適切なエラーハンドリングとログ出力を実装し、セキュリティベストプラクティスに従ってください。