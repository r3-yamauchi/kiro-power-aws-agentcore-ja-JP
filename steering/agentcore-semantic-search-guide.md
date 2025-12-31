# AgentCore Gateway - セマンティック検索ガイド

## 概要

このガイドでは、AgentCore Gateway のセマンティック検索機能について詳細に説明します。セマンティック検索により、AI エージェントが大量のツールの中から最適なツールを自動的に発見し、選択できるようになります。

## セマンティック検索の利点

1. **インテリジェントなツール発見**: 自然言語による意図から適切なツールを自動選択
2. **スケーラビリティ**: 数百から数千のツールがある環境でも効率的に動作
3. **コンテキスト理解**: ユーザーの要求とツールの機能の意味的な関連性を評価
4. **動的フィルタリング**: 実行時にツールセットを動的に絞り込み
5. **パフォーマンス向上**: 不要なツールの除外により実行効率を改善

## セマンティック検索の仕組み

### 1. ツール埋め込みベクトルの生成

Gateway は各ツールの説明、パラメータ、使用例から埋め込みベクトルを生成します：

```python
# ツール情報の例
tool_info = {
    "name": "get_weather",
    "description": "指定した都市の現在の天気情報を取得します",
    "parameters": {
        "city": "都市名（日本語または英語）",
        "units": "温度の単位（metric または imperial）"
    },
    "examples": [
        "東京の天気を教えて",
        "ニューヨークの気温は？",
        "今日の大阪の天候"
    ]
}

# Gateway が自動生成する埋め込みベクトル
embedding_vector = generate_embedding(
    f"{tool_info['name']} {tool_info['description']} {' '.join(tool_info['examples'])}"
)
```

### 2. クエリベースの検索

ユーザーのクエリから埋め込みベクトルを生成し、ツールベクトルとの類似度を計算：

```python
# ユーザークエリの例
user_query = "今日の東京の天気はどうですか？"

# クエリの埋め込みベクトル生成
query_embedding = generate_embedding(user_query)

# 類似度計算とランキング
tool_similarities = []
for tool in available_tools:
    similarity = cosine_similarity(query_embedding, tool.embedding)
    tool_similarities.append((tool, similarity))

# 類似度でソート
ranked_tools = sorted(tool_similarities, key=lambda x: x[1], reverse=True)
```

## セマンティック検索の有効化

### Gateway 作成時の設定

```bash
# セマンティック検索を有効にした Gateway の作成
agentcore gateway create-mcp-gateway \
    --name SemanticGateway \
    --enable_semantic_search \
    --semantic_search_config '{
        "embedding_model": "amazon.titan-embed-text-v1",
        "similarity_threshold": 0.7,
        "max_results": 10,
        "rerank_enabled": true
    }' \
    --region us-east-1
```

### 既存 Gateway でのセマンティック検索有効化

```bash
# 既存 Gateway にセマンティック検索を追加
agentcore gateway update-mcp-gateway \
    --name ExistingGateway \
    --enable_semantic_search \
    --semantic_search_config '{
        "embedding_model": "amazon.titan-embed-text-v1",
        "similarity_threshold": 0.6,
        "max_results": 15,
        "cache_embeddings": true,
        "update_frequency": "daily"
    }' \
    --region us-east-1
```

## セマンティック検索の設定オプション

### 基本設定

```json
{
    "semantic_search_config": {
        "embedding_model": "amazon.titan-embed-text-v1",
        "similarity_threshold": 0.7,
        "max_results": 10,
        "rerank_enabled": true,
        "cache_embeddings": true,
        "update_frequency": "daily"
    }
}
```

**設定パラメータの説明:**
- `embedding_model`: 使用する埋め込みモデル（Titan、Cohere など）
- `similarity_threshold`: ツール選択の最小類似度閾値（0.0-1.0）
- `max_results`: 返される最大ツール数
- `rerank_enabled`: 再ランキング機能の有効化
- `cache_embeddings`: 埋め込みベクトルのキャッシュ
- `update_frequency`: 埋め込みベクトルの更新頻度

### 高度な設定

```json
{
    "semantic_search_config": {
        "embedding_model": "amazon.titan-embed-text-v1",
        "similarity_threshold": 0.6,
        "max_results": 20,
        "rerank_enabled": true,
        "rerank_model": "amazon.titan-text-lite-v1",
        "boost_factors": {
            "name_match": 1.5,
            "description_match": 1.2,
            "parameter_match": 1.1
        },
        "category_weights": {
            "weather": 1.0,
            "database": 1.2,
            "api": 1.1,
            "calculation": 0.9
        },
        "language_preferences": ["ja", "en"],
        "context_window": 512
    }
}
```

## エージェントでのセマンティック検索活用

### 基本的な使用方法

```python
from strands import Agent
from bedrock_agentcore import BedrockAgentCoreApp
from mcp_client.client import get_streamable_http_mcp_client
from model.load import load_model

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload):
    """セマンティック検索対応エージェント"""
    
    mcp_client = get_streamable_http_mcp_client()
    
    with mcp_client:
        # セマンティック検索でツールを取得
        user_query = payload.get("prompt", "")
        
        # Gateway のセマンティック検索を使用
        relevant_tools = mcp_client.search_tools_semantic(
            query=user_query,
            max_results=5,
            threshold=0.7
        )
        
        # 関連性の高いツールのみでエージェントを作成
        agent = Agent(
            model=load_model(),
            tools=relevant_tools,
            system_prompt=f"""
            あなたは以下のクエリに最適化されたアシスタントです: "{user_query}"
            
            利用可能なツール: {[tool.name for tool in relevant_tools]}
            
            ユーザーの意図を理解し、最も適切なツールを使用してください。
            """
        )
        
        response = agent(user_query)
        
        return {
            "response": response,
            "selected_tools": [tool.name for tool in relevant_tools],
            "search_query": user_query
        }

if __name__ == "__main__":
    app.run()
```

### 動的ツール選択パターン

```python
def get_contextual_tools(mcp_client, user_query: str, context: dict = None):
    """コンテキストに基づく動的ツール選択"""
    
    # 基本的なセマンティック検索
    semantic_tools = mcp_client.search_tools_semantic(
        query=user_query,
        max_results=10,
        threshold=0.6
    )
    
    # コンテキストによる追加フィルタリング
    if context:
        domain = context.get('domain')
        user_role = context.get('user_role')
        
        # ドメイン固有のツールを優先
        if domain == 'finance':
            finance_tools = mcp_client.search_tools_semantic(
                query=f"{user_query} finance financial money",
                max_results=5,
                threshold=0.5
            )
            semantic_tools.extend(finance_tools)
        
        # ユーザーロールに基づくフィルタリング
        if user_role == 'admin':
            admin_tools = mcp_client.list_tools_by_category('administration')
            semantic_tools.extend(admin_tools)
    
    # 重複除去と優先度ソート
    unique_tools = {}
    for tool in semantic_tools:
        if tool.name not in unique_tools:
            unique_tools[tool.name] = tool
    
    return list(unique_tools.values())

@app.entrypoint
def invoke(payload):
    """コンテキスト対応セマンティック検索エージェント"""
    
    user_query = payload.get("prompt", "")
    context = payload.get("context", {})
    
    mcp_client = get_streamable_http_mcp_client()
    
    with mcp_client:
        # コンテキスト対応ツール選択
        contextual_tools = get_contextual_tools(mcp_client, user_query, context)
        
        agent = Agent(
            model=load_model(),
            tools=contextual_tools,
            system_prompt=f"""
            コンテキスト: {context}
            クエリ: {user_query}
            
            選択されたツール: {[tool.name for tool in contextual_tools]}
            
            コンテキストとクエリに基づいて最適なツールを使用してください。
            """
        )
        
        response = agent(user_query)
        
        return {
            "response": response,
            "context": context,
            "selected_tools": [tool.name for tool in contextual_tools]
        }
```

## セマンティック検索の最適化

### ツール説明の最適化

```python
# 良い例: 詳細で検索しやすい説明
good_tool_description = {
    "name": "calculate_mortgage_payment",
    "description": """
    住宅ローンの月々の支払い額を計算します。
    元金、金利、返済期間から正確な月額支払い額を算出し、
    総支払額や利息総額も提供します。
    
    用途: 住宅購入、ローン計算、返済計画、金融計算
    キーワード: 住宅ローン、月額支払い、金利計算、返済額
    """,
    "examples": [
        "3000万円のローンで金利1.5%、35年返済の月額は？",
        "住宅ローンの支払い額を計算して",
        "ローン返済額をシミュレーション"
    ]
}

# 悪い例: 簡潔すぎる説明
bad_tool_description = {
    "name": "calc_payment",
    "description": "支払い計算",
    "examples": []
}
```

### カテゴリとタグの活用

```python
# ツールのメタデータ拡張
enhanced_tool_metadata = {
    "name": "get_stock_price",
    "description": "指定した銘柄の現在の株価と関連情報を取得します",
    "category": "finance",
    "tags": ["株価", "投資", "市場", "金融", "証券"],
    "domain": "financial_services",
    "complexity": "simple",
    "response_time": "fast",
    "data_source": "real_time"
}
```

## セマンティック検索のモニタリング

### 検索パフォーマンスの監視

```bash
# セマンティック検索のメトリクス確認
aws cloudwatch get-metric-statistics \
    --namespace AWS/BedrockAgentCore/Gateway/SemanticSearch \
    --metric-name SearchLatency \
    --dimensions Name=GatewayName,Value=SemanticGateway \
    --start-time $(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 300 \
    --statistics Average,Maximum

# 検索精度のメトリクス
aws cloudwatch get-metric-statistics \
    --namespace AWS/BedrockAgentCore/Gateway/SemanticSearch \
    --metric-name SearchAccuracy \
    --dimensions Name=GatewayName,Value=SemanticGateway \
    --start-time $(date -d '24 hours ago' -u +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 3600 \
    --statistics Average
```

### 検索ログの分析

```python
import boto3
import json
from datetime import datetime, timedelta

def analyze_semantic_search_logs():
    """セマンティック検索ログの分析"""
    
    logs_client = boto3.client('logs')
    
    # 過去24時間のログを取得
    end_time = datetime.now()
    start_time = end_time - timedelta(hours=24)
    
    response = logs_client.filter_log_events(
        logGroupName='/aws/bedrock-agentcore/gateway/semantic-search',
        startTime=int(start_time.timestamp() * 1000),
        endTime=int(end_time.timestamp() * 1000),
        filterPattern='[timestamp, request_id, query, results]'
    )
    
    search_analytics = {
        'total_searches': 0,
        'avg_results_count': 0,
        'popular_queries': {},
        'low_confidence_searches': []
    }
    
    for event in response['events']:
        try:
            log_data = json.loads(event['message'])
            search_analytics['total_searches'] += 1
            
            query = log_data.get('query', '')
            results_count = len(log_data.get('results', []))
            confidence = log_data.get('avg_confidence', 0)
            
            # 人気クエリの追跡
            if query in search_analytics['popular_queries']:
                search_analytics['popular_queries'][query] += 1
            else:
                search_analytics['popular_queries'][query] = 1
            
            # 低信頼度検索の記録
            if confidence < 0.6:
                search_analytics['low_confidence_searches'].append({
                    'query': query,
                    'confidence': confidence,
                    'results_count': results_count
                })
                
        except json.JSONDecodeError:
            continue
    
    return search_analytics

# 分析実行
analytics = analyze_semantic_search_logs()
print(f"総検索数: {analytics['total_searches']}")
print(f"人気クエリ: {analytics['popular_queries']}")
print(f"低信頼度検索: {len(analytics['low_confidence_searches'])}")
```

## セマンティック検索のベストプラクティス

### 1. 適切な閾値設定

```python
# 用途別の推奨閾値
thresholds = {
    'strict_matching': 0.8,      # 厳密なマッチングが必要
    'general_purpose': 0.7,      # 一般的な用途
    'exploratory': 0.6,          # 探索的な検索
    'broad_search': 0.5          # 幅広い検索
}

def get_optimal_threshold(query_type: str, domain: str) -> float:
    """最適な閾値を動的に決定"""
    
    base_threshold = thresholds.get(query_type, 0.7)
    
    # ドメイン固有の調整
    domain_adjustments = {
        'medical': +0.1,    # 医療分野は厳密に
        'finance': +0.05,   # 金融分野も厳密に
        'general': 0.0,     # 一般分野は標準
        'creative': -0.1    # 創造的分野は柔軟に
    }
    
    adjustment = domain_adjustments.get(domain, 0.0)
    return min(max(base_threshold + adjustment, 0.3), 0.9)
```

### 2. ツール説明の標準化

```python
# ツール説明のテンプレート
tool_description_template = """
{primary_function}

詳細: {detailed_description}

用途: {use_cases}
対象データ: {data_types}
出力形式: {output_format}

キーワード: {keywords}
関連概念: {related_concepts}

例:
{examples}
"""

def create_optimized_description(tool_info: dict) -> str:
    """最適化されたツール説明を生成"""
    
    return tool_description_template.format(
        primary_function=tool_info['primary_function'],
        detailed_description=tool_info['detailed_description'],
        use_cases=', '.join(tool_info['use_cases']),
        data_types=', '.join(tool_info['data_types']),
        output_format=tool_info['output_format'],
        keywords=', '.join(tool_info['keywords']),
        related_concepts=', '.join(tool_info['related_concepts']),
        examples='\n'.join(f"- {ex}" for ex in tool_info['examples'])
    )
```

### 3. パフォーマンス最適化

```python
from functools import lru_cache
import hashlib

@lru_cache(maxsize=1000)
def cached_semantic_search(query_hash: str, mcp_client, max_results: int, threshold: float):
    """セマンティック検索結果のキャッシュ"""
    # 実際のクエリは別途管理
    return mcp_client.search_tools_semantic(
        query=get_query_from_hash(query_hash),
        max_results=max_results,
        threshold=threshold
    )

def optimized_semantic_search(mcp_client, query: str, max_results: int = 10, threshold: float = 0.7):
    """最適化されたセマンティック検索"""
    
    # クエリのハッシュ化（キャッシュキー用）
    query_hash = hashlib.md5(query.encode()).hexdigest()
    
    # キャッシュされた結果を使用
    return cached_semantic_search(query_hash, mcp_client, max_results, threshold)
```

## 実践的な活用例

### 例1: インテリジェントエージェント

大量のツールを持つ環境でセマンティック検索を活用：

```python
# src/intelligent_agent.py
import logging
from typing import List, Dict, Any
from strands import Agent, tool
from bedrock_agentcore import BedrockAgentCoreApp
from mcp_client.client import get_streamable_http_mcp_client
from model.load import load_model

logger = logging.getLogger(__name__)

@tool
def analyze_user_intent(query: str) -> Dict[str, Any]:
    """ユーザーの意図を分析してカテゴリと優先度を決定"""
    
    # 簡単な意図分析（実際にはより高度な NLP を使用）
    intent_keywords = {
        'weather': ['天気', '気温', '降水', '風', '湿度', 'weather', 'temperature'],
        'finance': ['株価', '為替', '投資', '金融', 'stock', 'finance', 'money'],
        'calculation': ['計算', '足し算', '引き算', '掛け算', '割り算', 'calculate', 'math'],
        'database': ['検索', 'データ', '顧客', '注文', 'search', 'data', 'query'],
        'general': []
    }
    
    detected_categories = []
    for category, keywords in intent_keywords.items():
        if any(keyword in query.lower() for keyword in keywords):
            detected_categories.append(category)
    
    if not detected_categories:
        detected_categories = ['general']
    
    return {
        'categories': detected_categories,
        'primary_category': detected_categories[0],
        'confidence': 0.8 if len(detected_categories) == 1 else 0.6,
        'query_complexity': 'simple' if len(query.split()) < 10 else 'complex'
    }

class IntelligentToolSelector:
    """セマンティック検索を活用したインテリジェントツール選択"""
    
    def __init__(self, mcp_client):
        self.mcp_client = mcp_client
        self.tool_cache = {}
        
    def select_tools(self, query: str, context: Dict[str, Any] = None) -> List[Any]:
        """クエリとコンテキストに基づいて最適なツールを選択"""
        
        # 意図分析
        intent = analyze_user_intent(query)
        logger.info(f"検出された意図: {intent}")
        
        # セマンティック検索の実行
        semantic_tools = self._semantic_search(query, intent)
        
        # コンテキストベースのフィルタリング
        if context:
            semantic_tools = self._apply_context_filter(semantic_tools, context)
        
        # 重複除去と優先度ソート
        final_tools = self._deduplicate_and_sort(semantic_tools, intent)
        
        logger.info(f"選択されたツール: {[tool.name for tool in final_tools]}")
        return final_tools
    
    def _semantic_search(self, query: str, intent: Dict[str, Any]) -> List[Any]:
        """セマンティック検索の実行"""
        
        # 基本的なセマンティック検索
        primary_tools = self.mcp_client.search_tools_semantic(
            query=query,
            max_results=8,
            threshold=0.7
        )
        
        # カテゴリ特化検索
        category_tools = []
        for category in intent['categories']:
            category_query = f"{query} {category}"
            tools = self.mcp_client.search_tools_semantic(
                query=category_query,
                max_results=3,
                threshold=0.6
            )
            category_tools.extend(tools)
        
        return primary_tools + category_tools
    
    def _apply_context_filter(self, tools: List[Any], context: Dict[str, Any]) -> List[Any]:
        """コンテキストに基づくツールフィルタリング"""
        
        user_role = context.get('user_role', 'user')
        domain = context.get('domain', 'general')
        security_level = context.get('security_level', 'standard')
        
        filtered_tools = []
        
        for tool in tools:
            # ロールベースのフィルタリング
            if user_role == 'admin' or not hasattr(tool, 'admin_only'):
                # セキュリティレベルのチェック
                tool_security = getattr(tool, 'security_level', 'standard')
                if self._check_security_clearance(security_level, tool_security):
                    filtered_tools.append(tool)
        
        return filtered_tools
    
    def _check_security_clearance(self, user_level: str, tool_level: str) -> bool:
        """セキュリティクリアランスのチェック"""
        levels = {'low': 1, 'standard': 2, 'high': 3, 'restricted': 4}
        return levels.get(user_level, 2) >= levels.get(tool_level, 2)
    
    def _deduplicate_and_sort(self, tools: List[Any], intent: Dict[str, Any]) -> List[Any]:
        """重複除去と優先度ソート"""
        
        # 重複除去
        unique_tools = {}
        for tool in tools:
            if tool.name not in unique_tools:
                unique_tools[tool.name] = tool
        
        # 優先度計算とソート
        prioritized_tools = []
        for tool in unique_tools.values():
            priority = self._calculate_priority(tool, intent)
            prioritized_tools.append((tool, priority))
        
        # 優先度でソート（降順）
        prioritized_tools.sort(key=lambda x: x[1], reverse=True)
        
        # 上位5つのツールを返す
        return [tool for tool, _ in prioritized_tools[:5]]
    
    def _calculate_priority(self, tool: Any, intent: Dict[str, Any]) -> float:
        """ツールの優先度を計算"""
        
        base_priority = 1.0
        
        # カテゴリマッチボーナス
        tool_category = getattr(tool, 'category', 'general')
        if tool_category in intent['categories']:
            base_priority += 0.5
        
        # 名前マッチボーナス
        if intent['primary_category'] in tool.name.lower():
            base_priority += 0.3
        
        # 複雑さマッチ
        tool_complexity = getattr(tool, 'complexity', 'simple')
        if tool_complexity == intent.get('query_complexity', 'simple'):
            base_priority += 0.2
        
        return base_priority

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload):
    """インテリジェントセマンティック検索エージェント"""
    
    user_query = payload.get("prompt", "")
    context = payload.get("context", {
        "user_role": "user",
        "domain": "general",
        "security_level": "standard"
    })
    
    logger.info(f"クエリ: {user_query}")
    logger.info(f"コンテキスト: {context}")
    
    mcp_client = get_streamable_http_mcp_client()
    
    with mcp_client:
        # インテリジェントツール選択
        tool_selector = IntelligentToolSelector(mcp_client)
        selected_tools = tool_selector.select_tools(user_query, context)
        
        # 選択されたツールでエージェントを作成
        agent = Agent(
            model=load_model(),
            tools=[analyze_user_intent] + selected_tools,
            system_prompt=f"""
            あなたはインテリジェントなアシスタントです。
            
            ユーザークエリ: {user_query}
            コンテキスト: {context}
            
            利用可能なツール: {[tool.name for tool in selected_tools]}
            
            セマンティック検索により、このクエリに最も関連性の高いツールが
            自動選択されました。ユーザーの意図を理解し、最適なツールを
            使用して正確で有用な回答を提供してください。
            
            ツールの選択理由も簡潔に説明してください。
            """
        )
        
        response = agent(user_query)
        
        return {
            "response": response,
            "selected_tools": [tool.name for tool in selected_tools],
            "context": context,
            "search_metadata": {
                "query": user_query,
                "tool_count": len(selected_tools),
                "search_method": "semantic_with_context"
            }
        }

if __name__ == "__main__":
    app.run()
```

### 例2: 効果測定システム

```python
# src/search_analytics.py
import json
import time
from typing import Dict, List, Any

class SemanticSearchAnalytics:
    """セマンティック検索の効果を測定"""
    
    def __init__(self):
        self.search_logs = []
    
    def log_search(self, query: str, selected_tools: List[str], 
                   user_feedback: str = None, execution_time: float = None):
        """検索結果をログに記録"""
        
        log_entry = {
            'timestamp': time.time(),
            'query': query,
            'selected_tools': selected_tools,
            'tool_count': len(selected_tools),
            'user_feedback': user_feedback,
            'execution_time': execution_time
        }
        
        self.search_logs.append(log_entry)
    
    def analyze_effectiveness(self) -> Dict[str, Any]:
        """検索効果の分析"""
        
        if not self.search_logs:
            return {"error": "ログデータがありません"}
        
        total_searches = len(self.search_logs)
        avg_tools_selected = sum(log['tool_count'] for log in self.search_logs) / total_searches
        
        # フィードバック分析
        positive_feedback = sum(1 for log in self.search_logs 
                              if log.get('user_feedback') == 'positive')
        feedback_rate = positive_feedback / total_searches if total_searches > 0 else 0
        
        # 実行時間分析
        execution_times = [log['execution_time'] for log in self.search_logs 
                          if log.get('execution_time')]
        avg_execution_time = sum(execution_times) / len(execution_times) if execution_times else 0
        
        return {
            'total_searches': total_searches,
            'avg_tools_selected': avg_tools_selected,
            'positive_feedback_rate': feedback_rate,
            'avg_execution_time': avg_execution_time,
            'search_efficiency': feedback_rate * (1 / max(avg_execution_time, 0.1))
        }

# 使用例
analytics = SemanticSearchAnalytics()

# 検索結果のログ記録
analytics.log_search(
    query="東京の天気",
    selected_tools=["getCurrentWeather", "getWeatherForecast"],
    user_feedback="positive",
    execution_time=1.2
)

# 効果分析
effectiveness = analytics.analyze_effectiveness()
print(f"検索効果分析: {effectiveness}")
```

## トラブルシューティング

### セマンティック検索が期待通りの結果を返さない

**症状**: 検索結果の関連性が低い

**診断手順**:
```bash
# セマンティック検索の設定確認
agentcore gateway get-mcp-gateway \
    --name SemanticGateway \
    --region us-east-1 \
    --include-semantic-config

# 検索メトリクスの確認
aws cloudwatch get-metric-statistics \
    --namespace AWS/BedrockAgentCore/Gateway/SemanticSearch \
    --metric-name SearchAccuracy \
    --dimensions Name=GatewayName,Value=SemanticGateway \
    --start-time $(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 300 \
    --statistics Average
```

**解決策**:
- 類似度閾値の調整（0.5-0.8の範囲で最適化）
- ツール説明の改善（キーワードと例の追加）
- 埋め込みベクトルの再生成
- 検索クエリの前処理改善

### 埋め込みベクトルの生成に失敗

**症状**: ベクトル生成や更新エラー

**診断手順**:
```bash
# 埋め込みモデルの状態確認
aws bedrock get-model-invocation-logging-configuration \
    --region us-east-1

# ツール説明の妥当性チェック
python -c "
import json
with open('tool-descriptions.json', 'r') as f:
    tools = json.load(f)

for tool in tools:
    desc = tool.get('description', '')
    if len(desc) < 10:
        print(f'短すぎる説明: {tool[\"name\"]} - {desc}')
    if len(desc) > 1000:
        print(f'長すぎる説明: {tool[\"name\"]} - {len(desc)} 文字')
"
```

**解決策**:
- ツール説明の長さを適切な範囲（50-500文字）に調整
- 特殊文字や不正な文字の除去
- 埋め込みモデルの権限確認
- バッチサイズの調整

### 検索パフォーマンスが遅い

**症状**: セマンティック検索の応答時間が長い

**診断手順**:
```bash
# 検索レイテンシの確認
aws cloudwatch get-metric-statistics \
    --namespace AWS/BedrockAgentCore/Gateway/SemanticSearch \
    --metric-name SearchLatency \
    --dimensions Name=GatewayName,Value=SemanticGateway \
    --start-time $(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 300 \
    --statistics Average,Maximum

# キャッシュヒット率の確認
aws cloudwatch get-metric-statistics \
    --namespace AWS/BedrockAgentCore/Gateway/SemanticSearch \
    --metric-name CacheHitRate \
    --dimensions Name=GatewayName,Value=SemanticGateway \
    --start-time $(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 300 \
    --statistics Average
```

**解決策**:
- 埋め込みキャッシュの有効化
- 検索結果数の制限（max_results を 5-10 に設定）
- 類似度閾値の最適化
- 検索クエリの前処理最適化

## 次のステップ

セマンティック検索機能を効果的に活用するために：

1. **基本設定の確認**: Gateway でセマンティック検索が有効になっていることを確認
2. **ツール説明の最適化**: 既存ツールの説明を検索しやすい形式に改善
3. **閾値の調整**: 用途に応じた適切な類似度閾値の設定
4. **モニタリングの実装**: 検索効果の継続的な測定と改善
5. **高度なパターンの実装**: コンテキスト対応やインテリジェント選択の導入

## 関連ガイド

- **Gateway 基本**: `agentcore-gateway-basics.md` - Gateway の基本概念
- **Lambda MCP 化**: `agentcore-lambda-mcp-guide.md` - Lambda ツールの最適化
- **OpenAPI MCP 変換**: `agentcore-openapi-mcp-guide.md` - API ツールの最適化
- **Gateway 統合**: `agentcore-gateway-integration.md` - 統合の概要

---

**注意**: セマンティック検索は強力な機能ですが、適切なツール説明と設定が重要です。継続的な最適化により、最大の効果を得ることができます。