# AgentCore Gateway - OpenAPI MCP 変換ガイド

## 概要

このガイドでは、既存の OpenAPI 仕様を持つ REST API を AgentCore Gateway 経由で MCP ツールとして利用できるように変換する方法を詳細に説明します。OpenAPI の MCP 変換により、外部の REST API を AI エージェントが直接呼び出せるツールとして統合できます。

## OpenAPI MCP 変換の利点

1. **既存 API の活用**: 既存の REST API をそのまま AI エージェントのツールとして利用
2. **標準化されたインターフェース**: OpenAPI 仕様による明確な API 定義
3. **自動ツール生成**: OpenAPI 仕様から MCP ツールスキーマを自動生成
4. **認証の統合**: API キーや OAuth2 認証の統一管理
5. **エラーハンドリング**: 標準化されたエラーレスポンス処理

## MCP 変換のプロセス

### ステップ 1: OpenAPI 仕様の準備

既存の OpenAPI 仕様を MCP 対応用に最適化します。

#### 元の OpenAPI 仕様例（天気 API）

```yaml
openapi: 3.0.0
info:
  title: Weather API
  version: 1.0.0
  description: 天気情報を提供する API
servers:
  - url: https://api.weather-service.com/v1
paths:
  /weather/current:
    get:
      summary: 現在の天気を取得
      parameters:
        - name: city
          in: query
          required: true
          schema:
            type: string
          description: 都市名
        - name: units
          in: query
          required: false
          schema:
            type: string
            enum: [metric, imperial]
            default: metric
          description: 単位系
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  city:
                    type: string
                  temperature:
                    type: number
                  humidity:
                    type: number
                  description:
                    type: string
  /weather/forecast:
    get:
      summary: 天気予報を取得
      parameters:
        - name: city
          in: query
          required: true
          schema:
            type: string
        - name: days
          in: query
          required: false
          schema:
            type: integer
            minimum: 1
            maximum: 7
            default: 3
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  city:
                    type: string
                  forecasts:
                    type: array
                    items:
                      type: object
                      properties:
                        date:
                          type: string
                        temperature_high:
                          type: number
                        temperature_low:
                          type: number
                        description:
                          type: string
```

#### MCP 最適化版 OpenAPI 仕様

```yaml
openapi: 3.0.0
info:
  title: Weather API for MCP
  version: 1.0.0
  description: AI エージェント用の天気情報 API
  x-mcp-info:
    tools:
      - name: get_current_weather
        description: 指定した都市の現在の天気情報を取得します
        operation: getCurrentWeather
      - name: get_weather_forecast
        description: 指定した都市の天気予報を取得します
        operation: getWeatherForecast
servers:
  - url: https://api.weather-service.com/v1
paths:
  /weather/current:
    get:
      operationId: getCurrentWeather
      summary: 現在の天気を取得
      description: 指定された都市の現在の天気情報を詳細に取得します
      parameters:
        - name: city
          in: query
          required: true
          schema:
            type: string
            minLength: 1
            maxLength: 100
          description: 天気情報を取得したい都市名（日本語または英語）
          example: "東京"
        - name: units
          in: query
          required: false
          schema:
            type: string
            enum: [metric, imperial]
            default: metric
          description: 温度の単位系（metric: 摂氏, imperial: 華氏）
      responses:
        '200':
          description: 天気情報の取得に成功
          content:
            application/json:
              schema:
                type: object
                required: [city, temperature, humidity, description]
                properties:
                  city:
                    type: string
                    description: 都市名
                  temperature:
                    type: number
                    description: 現在の気温
                  humidity:
                    type: number
                    minimum: 0
                    maximum: 100
                    description: 湿度（%）
                  description:
                    type: string
                    description: 天気の説明
                  wind_speed:
                    type: number
                    description: 風速
                  pressure:
                    type: number
                    description: 気圧
        '400':
          description: 不正なリクエスト
          content:
            application/json:
              schema:
                type: object
                properties:
                  error:
                    type: string
                  message:
                    type: string
        '404':
          description: 都市が見つからない
          content:
            application/json:
              schema:
                type: object
                properties:
                  error:
                    type: string
                  message:
                    type: string
  /weather/forecast:
    get:
      operationId: getWeatherForecast
      summary: 天気予報を取得
      description: 指定された都市の複数日の天気予報を取得します
      parameters:
        - name: city
          in: query
          required: true
          schema:
            type: string
            minLength: 1
            maxLength: 100
          description: 天気予報を取得したい都市名
          example: "大阪"
        - name: days
          in: query
          required: false
          schema:
            type: integer
            minimum: 1
            maximum: 7
            default: 3
          description: 予報日数（1-7日）
      responses:
        '200':
          description: 天気予報の取得に成功
          content:
            application/json:
              schema:
                type: object
                required: [city, forecasts]
                properties:
                  city:
                    type: string
                    description: 都市名
                  forecasts:
                    type: array
                    description: 天気予報のリスト
                    items:
                      type: object
                      required: [date, temperature_high, temperature_low, description]
                      properties:
                        date:
                          type: string
                          format: date
                          description: 予報日（YYYY-MM-DD形式）
                        temperature_high:
                          type: number
                          description: 最高気温
                        temperature_low:
                          type: number
                          description: 最低気温
                        description:
                          type: string
                          description: 天気の説明
                        precipitation_chance:
                          type: number
                          minimum: 0
                          maximum: 100
                          description: 降水確率（%）
components:
  securitySchemes:
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
security:
  - ApiKeyAuth: []
```

### ステップ 2: Gateway ターゲットの作成

OpenAPI 仕様を使用して Gateway ターゲットを作成します。

#### 外部 URL からの OpenAPI 仕様取得

```bash
# OpenAPI ターゲット設定ファイルの作成
cat > weather-api-target.json << 'EOF'
{
    "openApiSchema": {
        "uri": "https://api.weather-service.com/v1/openapi.json"
    }
}
EOF

# Gateway ターゲットの作成
agentcore gateway create-mcp-gateway-target \
    --gateway-arn $GATEWAY_ARN \
    --gateway-url $GATEWAY_URL \
    --role-arn $GATEWAY_ROLE_ARN \
    --name WeatherAPITarget \
    --target-type openApiSchema \
    --target-payload "$(cat weather-api-target.json)" \
    --credentials '{
        "api_key": "your-weather-api-key",
        "credential_location": "header",
        "credential_parameter_name": "X-API-Key"
    }' \
    --region us-east-1
```

#### ローカル OpenAPI 仕様の使用

```bash
# ローカル OpenAPI 仕様をインライン化
cat > weather-api-inline.json << 'EOF'
{
    "openApiSchema": {
        "inlinePayload": {
            "openapi": "3.0.0",
            "info": {
                "title": "Weather API for MCP",
                "version": "1.0.0"
            },
            "servers": [
                {
                    "url": "https://api.weather-service.com/v1"
                }
            ],
            "paths": {
                "/weather/current": {
                    "get": {
                        "operationId": "getCurrentWeather",
                        "summary": "現在の天気を取得",
                        "parameters": [
                            {
                                "name": "city",
                                "in": "query",
                                "required": true,
                                "schema": {
                                    "type": "string"
                                }
                            }
                        ],
                        "responses": {
                            "200": {
                                "description": "成功",
                                "content": {
                                    "application/json": {
                                        "schema": {
                                            "type": "object",
                                            "properties": {
                                                "city": {"type": "string"},
                                                "temperature": {"type": "number"},
                                                "description": {"type": "string"}
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
EOF

# インライン仕様でターゲット作成
agentcore gateway create-mcp-gateway-target \
    --gateway-arn $GATEWAY_ARN \
    --gateway-url $GATEWAY_URL \
    --role-arn $GATEWAY_ROLE_ARN \
    --name WeatherAPIInlineTarget \
    --target-type openApiSchema \
    --target-payload "$(cat weather-api-inline.json)" \
    --credentials '{
        "api_key": "your-api-key",
        "credential_location": "header",
        "credential_parameter_name": "X-API-Key"
    }' \
    --region us-east-1
```

## 高度な OpenAPI MCP パターン

### パターン 1: 複数認証方式の対応

#### OAuth2 認証

```bash
# OAuth2 認証の設定
agentcore gateway create-mcp-gateway-target \
    --gateway-arn $GATEWAY_ARN \
    --gateway-url $GATEWAY_URL \
    --role-arn $GATEWAY_ROLE_ARN \
    --name SalesforceAPITarget \
    --target-type openApiSchema \
    --target-payload '{
        "openApiSchema": {
            "uri": "https://api.salesforce.com/openapi.json"
        }
    }' \
    --credentials '{
        "oauth2_provider_config": {
            "customOauth2ProviderConfig": {
                "oauthDiscovery": {
                    "discoveryUrl": "https://login.salesforce.com/.well-known/openid_configuration"
                },
                "clientId": "your-salesforce-client-id",
                "clientSecret": "your-salesforce-client-secret"
            }
        },
        "scopes": ["api", "refresh_token"]
    }' \
    --region us-east-1
```

#### カスタムヘッダー認証

```bash
# カスタムヘッダー認証の設定
agentcore gateway create-mcp-gateway-target \
    --gateway-arn $GATEWAY_ARN \
    --gateway-url $GATEWAY_URL \
    --role-arn $GATEWAY_ROLE_ARN \
    --name CustomAPITarget \
    --target-type openApiSchema \
    --target-payload '{
        "openApiSchema": {
            "uri": "https://api.custom-service.com/openapi.json"
        }
    }' \
    --credentials '{
        "custom_headers": {
            "Authorization": "Bearer your-token",
            "X-Client-ID": "your-client-id",
            "X-API-Version": "v2"
        }
    }' \
    --region us-east-1
```

### パターン 2: 複合 API の統合

複数の関連する API を一つの Gateway ターゲットとして統合：

```yaml
# 統合 OpenAPI 仕様例
openapi: 3.0.0
info:
  title: Business Services API
  version: 1.0.0
  description: 統合ビジネスサービス API
servers:
  - url: https://api.business-services.com/v1
paths:
  /customers:
    get:
      operationId: listCustomers
      summary: 顧客一覧を取得
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
            default: 10
        - name: offset
          in: query
          schema:
            type: integer
            default: 0
      responses:
        '200':
          description: 顧客一覧
          content:
            application/json:
              schema:
                type: object
                properties:
                  customers:
                    type: array
                    items:
                      $ref: '#/components/schemas/Customer'
    post:
      operationId: createCustomer
      summary: 新規顧客を作成
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CustomerInput'
      responses:
        '201':
          description: 顧客作成成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Customer'
  /orders:
    get:
      operationId: listOrders
      summary: 注文一覧を取得
      parameters:
        - name: customer_id
          in: query
          schema:
            type: string
        - name: status
          in: query
          schema:
            type: string
            enum: [pending, processing, completed, cancelled]
      responses:
        '200':
          description: 注文一覧
          content:
            application/json:
              schema:
                type: object
                properties:
                  orders:
                    type: array
                    items:
                      $ref: '#/components/schemas/Order'
components:
  schemas:
    Customer:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
        email:
          type: string
        created_at:
          type: string
          format: date-time
    CustomerInput:
      type: object
      required: [name, email]
      properties:
        name:
          type: string
          minLength: 1
        email:
          type: string
          format: email
    Order:
      type: object
      properties:
        id:
          type: string
        customer_id:
          type: string
        status:
          type: string
        total_amount:
          type: number
        created_at:
          type: string
          format: date-time
```

## OpenAPI MCP 変換のベストプラクティス

### 1. 適切な operationId の設定

```yaml
paths:
  /users/{id}:
    get:
      operationId: getUserById  # 明確で一意な ID
      summary: ユーザー情報を取得
      # ...
    put:
      operationId: updateUser   # 動詞 + 名詞の形式
      summary: ユーザー情報を更新
      # ...
```

### 2. 詳細な説明とサンプル

```yaml
parameters:
  - name: city
    in: query
    required: true
    schema:
      type: string
      minLength: 1
      maxLength: 100
    description: |
      天気情報を取得したい都市名。
      日本語または英語で指定可能。
      例: "東京", "Tokyo", "大阪", "Osaka"
    example: "東京"
```

### 3. エラーレスポンスの標準化

```yaml
responses:
  '400':
    description: 不正なリクエスト
    content:
      application/json:
        schema:
          type: object
          properties:
            error:
              type: string
              description: エラーコード
            message:
              type: string
              description: エラーメッセージ
            details:
              type: object
              description: 詳細情報
        example:
          error: "INVALID_PARAMETER"
          message: "都市名が無効です"
          details:
            parameter: "city"
            value: ""
```

### 4. セキュリティ設定の明確化

```yaml
components:
  securitySchemes:
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
      description: API キー認証
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT トークン認証
security:
  - ApiKeyAuth: []
  - BearerAuth: []
```

## 実践的な活用例

### 例: CRM システムの MCP 化

#### 元の REST API

```bash
# 従来の API 呼び出し
curl -X GET "https://api.crm-system.com/v1/customers" \
     -H "Authorization: Bearer token" \
     -H "Content-Type: application/json"

curl -X POST "https://api.crm-system.com/v1/customers" \
     -H "Authorization: Bearer token" \
     -H "Content-Type: application/json" \
     -d '{"name": "田中太郎", "email": "tanaka@example.com"}'
```

#### MCP 対応 OpenAPI 仕様

```yaml
openapi: 3.0.0
info:
  title: CRM System API
  version: 1.0.0
  description: 顧客管理システム API for MCP
servers:
  - url: https://api.crm-system.com/v1
paths:
  /customers:
    get:
      operationId: listCustomers
      summary: 顧客一覧を取得
      description: 登録されている顧客の一覧を取得します
      parameters:
        - name: limit
          in: query
          required: false
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 10
          description: 取得する顧客数の上限
        - name: search
          in: query
          required: false
          schema:
            type: string
            maxLength: 100
          description: 顧客名での検索キーワード
      responses:
        '200':
          description: 顧客一覧の取得に成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  customers:
                    type: array
                    items:
                      $ref: '#/components/schemas/Customer'
                  total:
                    type: integer
                    description: 総顧客数
    post:
      operationId: createCustomer
      summary: 新規顧客を作成
      description: 新しい顧客を登録します
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [name, email]
              properties:
                name:
                  type: string
                  minLength: 1
                  maxLength: 100
                  description: 顧客名
                email:
                  type: string
                  format: email
                  description: メールアドレス
                phone:
                  type: string
                  pattern: '^[0-9-+()\\s]+$'
                  description: 電話番号
                company:
                  type: string
                  maxLength: 100
                  description: 会社名
      responses:
        '201':
          description: 顧客の作成に成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Customer'
  /customers/{id}:
    get:
      operationId: getCustomerById
      summary: 顧客詳細を取得
      description: 指定された ID の顧客詳細情報を取得します
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
          description: 顧客ID
      responses:
        '200':
          description: 顧客詳細の取得に成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Customer'
        '404':
          description: 顧客が見つからない
    put:
      operationId: updateCustomer
      summary: 顧客情報を更新
      description: 指定された ID の顧客情報を更新します
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
          description: 顧客ID
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                name:
                  type: string
                  minLength: 1
                  maxLength: 100
                email:
                  type: string
                  format: email
                phone:
                  type: string
                company:
                  type: string
      responses:
        '200':
          description: 顧客情報の更新に成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Customer'
components:
  schemas:
    Customer:
      type: object
      properties:
        id:
          type: string
          description: 顧客ID
        name:
          type: string
          description: 顧客名
        email:
          type: string
          description: メールアドレス
        phone:
          type: string
          description: 電話番号
        company:
          type: string
          description: 会社名
        created_at:
          type: string
          format: date-time
          description: 作成日時
        updated_at:
          type: string
          format: date-time
          description: 更新日時
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
security:
  - BearerAuth: []
```

#### Gateway ターゲットの作成

```bash
# CRM API ターゲットの作成
agentcore gateway create-mcp-gateway-target \
    --gateway-arn $GATEWAY_ARN \
    --gateway-url $GATEWAY_URL \
    --role-arn $GATEWAY_ROLE_ARN \
    --name CRMAPITarget \
    --target-type openApiSchema \
    --target-payload '{
        "openApiSchema": {
            "uri": "https://api.crm-system.com/v1/openapi.json"
        }
    }' \
    --credentials '{
        "api_key": "your-crm-api-token",
        "credential_location": "header",
        "credential_parameter_name": "Authorization",
        "credential_prefix": "Bearer "
    }' \
    --region us-east-1
```

#### エージェントでの使用例

```python
# src/crm_agent.py
from strands import Agent, tool
from bedrock_agentcore import BedrockAgentCoreApp
from mcp_client.client import get_streamable_http_mcp_client
from model.load import load_model

@tool
def format_customer_info(customer_data: dict) -> str:
    """顧客情報を読みやすい形式でフォーマット"""
    name = customer_data.get('name', 'N/A')
    email = customer_data.get('email', 'N/A')
    company = customer_data.get('company', 'N/A')
    
    return f"""
顧客情報:
- 名前: {name}
- メール: {email}
- 会社: {company}
- 登録日: {customer_data.get('created_at', 'N/A')}
"""

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload):
    """CRM システム統合エージェント"""
    
    mcp_client = get_streamable_http_mcp_client()
    
    with mcp_client:
        gateway_tools = mcp_client.list_tools_sync()
        
        # CRM ツールをフィルタ
        crm_tools = [
            tool for tool in gateway_tools 
            if tool.name in ['listCustomers', 'createCustomer', 'getCustomerById', 'updateCustomer']
        ]
        
        agent = Agent(
            model=load_model(),
            tools=[format_customer_info] + crm_tools,
            system_prompt="""
            あなたは CRM システムの管理アシスタントです。以下の機能を提供できます:
            
            1. 顧客一覧の表示 (listCustomers)
            2. 新規顧客の登録 (createCustomer)
            3. 顧客詳細の取得 (getCustomerById)
            4. 顧客情報の更新 (updateCustomer)
            5. 顧客情報の整形表示 (format_customer_info)
            
            ユーザーのリクエストに応じて適切なツールを使用してください。
            顧客情報を表示する際は、format_customer_info ツールを使用して
            読みやすい形式で表示してください。
            """
        )
        
        prompt = payload.get("prompt", "CRM システムの機能を教えてください")
        response = agent(prompt)
        
        return {"response": response}

if __name__ == "__main__":
    app.run()
```

## OpenAPI 仕様の検証とテスト

### 仕様の検証

```bash
# OpenAPI 仕様の妥当性チェック
npx @apidevtools/swagger-parser validate openapi.yaml

# または Python を使用
uv add openapi-spec-validator
python -c "
from openapi_spec_validator import validate_spec
import yaml

with open('openapi.yaml', 'r') as f:
    spec = yaml.safe_load(f)

try:
    validate_spec(spec)
    print('OpenAPI 仕様は有効です')
except Exception as e:
    print(f'仕様エラー: {e}')
"
```

### Gateway 経由でのテスト

```bash
# Gateway ツールのテスト
agentcore invoke --dev '{
    "prompt": "get_current_weather ツールを使って東京の天気を教えてください"
}'

# 特定のパラメータでのテスト
agentcore invoke --dev '{
    "prompt": "get_weather_forecast を使って大阪の5日間の天気予報を取得してください"
}'
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

## 次のステップ

OpenAPI MCP 変換が完了したら、以下のガイドも参照してください：

- **セマンティック検索**: `agentcore-semantic-search-guide.md` - インテリジェントなツール選択
- **Lambda MCP 化**: `agentcore-lambda-mcp-guide.md` - Lambda 関数の MCP 変換
- **運用・トラブルシューティング**: `agentcore-gateway-operations.md` - 運用と問題解決

---

**注意**: OpenAPI 仕様を MCP ツールとして使用する場合、適切な認証設定とエラーハンドリングを実装し、API の利用制限に注意してください。