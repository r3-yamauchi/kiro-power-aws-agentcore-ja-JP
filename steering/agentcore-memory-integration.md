# AgentCore メモリ統合ガイド

## 概要

このガイドでは、AgentCore メモリを Strands エージェントに追加するためのステップバイステップの手順を説明します。
メモリにより、セッション間での会話の永続性と長期的な知識保持が可能になります。

## 前提条件

- デプロイ済み、またはデプロイ準備が整っている既存の Strands エージェント
- `bedrock-agentcore-starter-toolkit` がインストールされていること
- AWS 認証情報が設定されていること

## ステップ 1: 必要なパッケージのインストール

```bash
pip install 'bedrock-agentcore[strands-agents]'
```

`pyproject.toml` または `requirements.txt` に追加します。
```
bedrock-agentcore[strands-agents]
```

## ステップ 2: メモリリソースの作成

**重要**: メモリの作成は、エージェントコードとは別に、**1 回限りのセットアップ**です。

### オプション A: CLI の使用 (推奨)

```bash
# 基本メモリの作成 (STM のみ)
agentcore memory create my_agent_memory \
    --description "My agent のメモリ" \
    --region us-west-2 \
    --wait

# LTM 戦略を使用した作成 (高度な設定)
agentcore memory create my_agent_memory \
    --description "長期戦略を持つメモリ" \
    --strategies '[
        {
            "summaryMemoryStrategy": {
                "name": "SessionSummarizer",
                "namespaces": ["/summaries/{actorId}/{sessionId}"]
            }
        },
        {
            "userPreferenceMemoryStrategy": {
                "name": "PreferenceLearner",
                "namespaces": ["/preferences/{actorId}"]
            }
        },
        {
            "semanticMemoryStrategy": {
                "name": "FactExtractor",
                "namespaces": ["/facts/{actorId}"]
            }
        }
    ]' \
    --region us-west-2 \
    --wait
```

### オプション B: Python の使用 (スクリプト用)

```python
from bedrock_agentcore.memory import MemoryClient

client = MemoryClient(region_name="us-west-2")

# 基本メモリ
basic_memory = client.create_memory(
    name="MyAgentMemory",
    description="My agent のメモリ"
)

memory_id = basic_memory.get('id')
print(f"作成されたメモリの ID: {memory_id}")
```

**メモリ ID を保存してください** - エージェントの設定で必要になります。

## ステップ 3: エージェントコードの更新

エージェントにメモリのインポートと設定を追加します。

```python
import os
from datetime import datetime
from strands import Agent
from bedrock_agentcore import BedrockAgentCoreApp
from bedrock_agentcore.memory.integrations.strands.config import AgentCoreMemoryConfig
from bedrock_agentcore.memory.integrations.strands.session_manager import AgentCoreMemorySessionManager

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload):
    # メモリ設定
    memory_id = "your-memory-id-here"  # ステップ 2 で取得した ID
    
    # ペイロードからセッション ID とアクター ID を取得するか、デフォルトを生成
    session_id = payload.get("session_id", f"session_{datetime.now().strftime('%Y%m%d%H%M%S')}")
    actor_id = payload.get("actor_id", "default_user")
    
    # AgentCore メモリセッションマネージャーの作成
    agentcore_memory_config = AgentCoreMemoryConfig(
        memory_id=memory_id,
        session_id=session_id,
        actor_id=actor_id
    )
    
    session_manager = AgentCoreMemorySessionManager(
        agentcore_memory_config=agentcore_memory_config,
        region_name="us-west-2"
    )
    
    # メモリを持つエージェントの作成
    agent = Agent(
        system_prompt="あなたは役立つアシスタントです。",
        session_manager=session_manager
    )
    
    # ユーザープロンプトの処理
    prompt = payload.get("prompt", "Hello")
    response = agent(prompt)
    
    return {
        "response": response,
        "session_id": session_id,
        "actor_id": actor_id
    }

if __name__ == "__main__":
    app.run()
```

## ステップ 4: 環境変数の設定 (オプション)

`.env` ファイルを作成します。
```bash
AGENTCORE_MEMORY_ID=your-memory-id-here
AWS_REGION=us-west-2
```

コードで次のように使用します。
```python
memory_id = os.environ.get("AGENTCORE_MEMORY_ID", "fallback-memory-id")
```

## ステップ 5: AgentCore 設定の更新

`.bedrock_agentcore.yaml` を編集します。

```yaml
agents:
  YourAgent:
    memory:
      mode: STM_ONLY  # または長期メモリの場合は STM_AND_LTM
      memory_id: your-memory-id-here
      memory_name: MyAgentMemory
      event_expiry_days: 30
```

**重要なデプロイに関する注意点:**
- メモリを持つ新しいエージェントをデプロイする場合、まず `mode: NO_MEMORY` で開始します。
- エージェントをデプロイし、正常にテストします。
- その後、`mode: STM_ONLY` または `mode: STM_AND_LTM` に更新します。
- エージェントを再デプロイします。

これにより、メモリの複雑さを追加する前にエージェントが機能することを確認できます。

## ステップ 6: エージェントのデプロイ

```bash
agentcore launch
```

## ステップ 7: メモリ機能のテスト

会話の永続性をテストします。

```bash
# 最初のメッセージ
agentcore invoke '{
    "prompt": "My name is Alice and I like pizza",
    "session_id": "test_session_001",
    "actor_id": "alice"
}'

# 2 番目のメッセージ - エージェントは記憶しているはずです
agentcore invoke '{
    "prompt": "What is my name and what do I like?",
    "session_id": "test_session_001",
    "actor_id": "alice"
}'
```

## メモリ管理コマンド

### すべてのメモリを一覧表示
```bash
agentcore memory list --region us-west-2
```

### メモリの詳細を取得
```bash
agentcore memory get your-memory-id --region us-west-2
```

### メモリのステータスを確認
```bash
agentcore memory status your-memory-id --region us-west-2
```

### メモリの削除 (警告: 永続的です)
```bash
agentcore memory delete your-memory-id --region us-west-2 --wait
```

## メモリ設定オプション

### AgentCoreMemoryConfig パラメータ

| パラメータ | タイプ | 必須 | 説明 |
|-----------|------|----------|-------------|
| `memory_id` | str | はい | メモリリソースの ID |
| `session_id` | str | はい | 一意のセッション識別子 |
| `actor_id` | str | はい | 一意のユーザー/アクター識別子 |
| `retrieval_config` | Dict | いいえ | LTM (長期記憶) 検索の設定 |

### RetrievalConfig (LTM 用)

```python
from bedrock_agentcore.memory.integrations.strands.config import RetrievalConfig

config = AgentCoreMemoryConfig(
    memory_id=memory_id,
    session_id=session_id,
    actor_id=actor_id,
    retrieval_config={
        "/preferences/{actorId}": RetrievalConfig(
            top_k=5,
            relevance_score=0.7
        ),
        "/facts/{actorId}": RetrievalConfig(
            top_k=10,
            relevance_score=0.3
        )
    }
)
```

## メモリ戦略の説明

### 1. 短期記憶 (STM)
- セッション内の会話イベントを保存します。
- 保持ポリシーに基づいて自動的に期限切れになります (デフォルト: 90 日)。
- 最適な用途: 会話の継続性。

### 2. 要約メモリ戦略
- 会話セッションを自動的に要約します。
- 名前空間: `/summaries/{actorId}/{sessionId}`
- 最適な用途: セッションの要約、コンテキストの圧縮。

### 3. ユーザー設定メモリ戦略
- セッション間でユーザー設定を学習し、保存します。
- 名前空間: `/preferences/{actorId}`
- 最適な用途: パーソナライゼーション、ユーザー設定。

### 4. 意味メモリ戦略
- 事実情報を抽出し、保存します。
- 名前空間: `/facts/{actorId}`
- 最適な用途: 知識保持、事実の想起。

## ベストプラクティス

1. **セッション ID**: 関連する会話には一貫性のある意味のあるセッション ID を使用します。
2. **アクター ID**: ユーザーごとに一意の識別子 (例: ユーザー ID、メールハッシュ) を使用します。
3. **メモリモード**: `STM_ONLY` から開始し、必要に応じて LTM 戦略を追加します。
4. **テスト**: デプロイする前に必ずローカルでメモリをテストします。
5. **モニタリング**: メモリ関連のエラーについては CloudWatch ログを確認します。
6. **クリーンアップ**: コストを避けるために、未使用のメモリリソースを削除します。

## トラブルシューティング

### メモリが永続化されない
- メモリ ID が正しいことを確認します。
- `session_id` が呼び出し間で一貫していることを確認します。
- メモリモードが `NO_MEMORY` ではないことを確認します。
- エラーについては CloudWatch ログを確認します。

### メモリ作成の失敗
- AgentCore メモリの AWS 権限を確認します。
- メモリ名がすでに存在しないことを確認します。
- リージョンがサポートされていることを確認します。

### メモリ追加後にエージェントが失敗する
- `mode: NO_MEMORY` で開始し、テストします。
- メモリリソースが `ACTIVE` であることを確認します。
- IAM ロールにメモリ権限があることを確認します。

## 例: メモリを持つ完全なエージェント

```python
import os
from datetime import datetime
from strands import Agent, tool
from bedrock_agentcore import BedrockAgentCoreApp
from bedrock_agentcore.memory.integrations.strands.config import AgentCoreMemoryConfig
from bedrock_agentcore.memory.integrations.strands.session_manager import AgentCoreMemorySessionManager

@tool
def get_time() -> str:
    """現在の時刻を取得します"""
    return datetime.now().strftime("%H:%M:%S")

app = BedrockAgentCoreApp()

@app.entrypoint
def invoke(payload):
    # メモリ設定
    memory_id = os.environ.get("AGENTCORE_MEMORY_ID", "your-memory-id")
    session_id = payload.get("session_id", f"session_{datetime.now().strftime('%Y%m%d%H%M%S')}")
    actor_id = payload.get("actor_id", "default_user")
    
    # セッションマネージャーの作成
    config = AgentCoreMemoryConfig(
        memory_id=memory_id,
        session_id=session_id,
        actor_id=actor_id
    )
    
    session_manager = AgentCoreMemorySessionManager(
        agentcore_memory_config=config,
        region_name="us-west-2"
    )
    
    # メモリとツールを持つエージェントの作成
    agent = Agent(
        system_prompt="あなたはメモリを持つ役立つアシスタントです。ユーザーの設定と過去の会話を記憶します。",
        tools=[get_time],
        session_manager=session_manager
    )
    
    # リクエストの処理
    prompt = payload.get("prompt", "Hello")
    response = agent(prompt)
    
    return {
        "response": response,
        "session_id": session_id,
        "actor_id": actor_id
    }

if __name__ == "__main__":
    app.run()
```

## リソース

- [AgentCore メモリ ドキュメント](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory.html)
- [Strands セッション管理](https://strandsagents.com/latest/documentation/docs/user-guide/concepts/agents/session-management/)
- [メモリ名前空間ガイド](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/session-actor-namespace.html)

## クイックリファレンス

```bash
# メモリの作成
agentcore memory create my_memory --wait

# メモリの一覧表示
agentcore memory list

# メモリの詳細を取得
agentcore memory get <memory-id>

# メモリの削除
agentcore memory delete <memory-id> --wait
```

---

**注意**: メモリは 1 回作成され、その後エージェントコードで ID によって参照されます。メモリリソースはエージェントのデプロイとは独立して永続化されます。