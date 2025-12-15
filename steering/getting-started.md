# AgentCore の使用開始

## 前提条件
- 開始する前に、`bedrock-agentcore-starter-toolkit` をインストールしてください: `pip install bedrock-agentcore-starter-toolkit`

## 追加のステアリングファイルの取得
AgentCore を使用する際は、メモリとゲートウェイ管理のための専用ステアリングファイルを取得する必要があります。

**メモリステアリングファイルを取得するには:**
```bash
# Kiroにメモリステアリングドキュメントを取得して.kiro/steering/に保存するよう依頼する
"AgentCore Memory steering file を取得し、.kiro/steering/ に保存してください"
```

**ゲートウェイステアリングファイルを取得するには:**
```bash
# Kiroにゲートウェイステアリングドキュメントを取得して.kiro/steering/に保存するよう依頼する
"AgentCore Gateway steering file を取得し、.kiro/steering/ に保存してください"
```

これらのステアリングファイルは、以下の詳細なガイダンスを提供します。
- メモリリソースの作成とCLIコマンド
- ゲートウェイのデプロイと設定
- ベストプラクティスとトラブルシューティング

## メモリ管理
**重要:** ユーザーが「メモリ」について言及したり、AgentCore Memory について尋ねたりした場合は、常に `manage_agentcore_memory` MCP ツールを使用して、完全なドキュメントとCLIコマンドを取得してください。このツールは以下を提供します。
- メモリリソースの作成と設定
- メモリ操作のための完全なCLIコマンドリファレンス
- メモリ統合のベストプラクティス

このツールを参照せずに、手動でメモリを設定しようとしないでください。

**重要 - メモリとデプロイ:**
- ユーザーが AgentCore Memory リソースを作成したが、エージェントをデプロイしたい場合は、常に `.bedrock_agentcore.yaml` 設定で `mode: NO_MEMORY` を使用して最初に起動してください。
- メモリ統合は、最初のデプロイが成功した後に追加する必要があります。
- エージェントコードにはメモリセッションマネージャーロジックを含めることができますが、`NO_MEMORY` が設定されている場合は永続化されません。
- エージェントが正常にデプロイされたら、設定を `STM_ONLY` または `STM_AND_LTM` に更新して再デプロイできます。

## MCPツールのための AgentCore Gateway
**重要:** ユーザーが「ゲートウェイ」について言及したり、AgentCore Gateway、MCPツール、またはAPIをツールとして公開することについて尋ねたりした場合は、常に `manage_agentcore_gateway` MCP ツールを使用して、完全なドキュメントとCLIコマンドを取得してください。このツールは以下を提供します。
- ゲートウェイの作成と設定要件
- ステップバイステップのCLIデプロイワークフロー
- Lambda、OpenAPI、Smithyモデルのターゲット管理
- 認証と認可の設定
- 一般的な問題とトラブルシューティング

このツールを参照せずに、手動でゲートウェイを設定しようとしないでください。

## 2つのパス: 新規プロジェクト vs 既存エージェント

### パス 1: 既存エージェントのデプロイ
**すでにエージェントをお持ちの場合**、`agentcore create` は使用しないでください。代わりに:

1. **MCPツールを使用してデプロイ要件を取得します:**
   - `manage_agentcore_runtime` ツールを呼び出してコード要件を理解します。
   - これにより、既存エージェントの完全なデプロイガイドが提供されます。

2. **既存エージェントの主要な要件:**
   - エージェントを `BedrockAgentCoreApp` でラップします。
   - `@app.entrypoint` デコレータを追加します。
   - `requirements.txt` に `bedrock-agentcore` を含めます。
   - その後、`agentcore configure` と `agentcore launch` を使用します。

**重要:** `agentcore create` コマンドは、新しいプロジェクトをゼロから作成する場合にのみ使用します。既存のエージェントに対して使用すると、コードが上書きされます。

---

### パス 2: ゼロからの新規プロジェクト作成
`agentcore create` を使用して、新しいエージェントプロジェクトのワークスペースを初期化します。

**AIアシスタントにとって重要:** このコマンドをプログラムで実行する場合は、常に `--non-interactive` モードを使用してください。インタラクティブモードは、人間が手動でコマンドを実行する場合にのみ使用します。

**デフォルトの動作:** ユーザーが特定のオプション (テンプレート、エージェントフレームワーク、モデルプロバイダー) を指定しない場合、デフォルトを使用します。
- テンプレート: `basic`
- エージェントフレームワーク: `Strands`
- モデルプロバイダー: `Bedrock`

つまり、`agentcore create --non-interactive --project-name <name>` を実行するだけで、すべてのデフォルトが使用されます。

**コマンド:**
```bash
agentcore create --non-interactive --project-name <name> --template <template> --agent-framework <framework> --model-provider <provider>
```

**オプション:**
- `--project-name, -p`: プロジェクト名 (英数字のみ、最大36文字、文字で始まり、ダッシュやアンダースコアなし)
- `--template, -t`: テンプレートタイプ - ランタイムのみの `basic` または IaCを含むモノレポの `production` (デフォルト: basic)
- `--agent-framework`: エージェントSDKプロバイダー (Strands, ClaudeAgents, OpenAIなど) (デフォルト: Strands)
- `--model-provider, -mp`: モデルプロバイダー (Bedrock, OpenAIなど) (デフォルト: Bedrock)
- `--provider-api-key, -key`: Bedrock以外のプロバイダーのAPIキー
- `--iac`: Infrastructure as Codeプロバイダー (CDKまたはTerraform) - productionテンプレートに必須
- `--non-interactive`: 非対話モードで実行 (フラグが提供されると自動的に有効になります)
- `--venv` / `--no-venv`: 仮想環境を自動的に作成し、依存関係をインストールします (デフォルト: true)

**例:**
```bash
# AIアシスタントの使用 (非対話型、デフォルトのbasicテンプレート)
agentcore create --non-interactive --project-name MyAgent --template basic --agent-framework Strands --model-provider Bedrock

# カスタムエージェントフレームワークを使用
agentcore create --non-interactive --project-name MyAgent --template basic --agent-framework ClaudeAgents --model-provider Bedrock

# IaCを含むproductionテンプレート
agentcore create --non-interactive --project-name MyAgent --template production --agent-framework Strands --model-provider Bedrock --iac CDK

# 最小限のコマンド (すべてのデフォルトを使用: basicテンプレート、Strands、Bedrock)
agentcore create --non-interactive --project-name MyAgent

# 人間ユーザーのインタラクティブモード (手動使用のみ)
agentcore create
```

**作成されるもの:**
- 選択した名前のプロジェクトディレクトリ
- エージェントSDKのボイラープレートコード
- 設定ファイル (.bedrock_agentcore.yaml)
- 依存関係の設定 (pyproject.toml または requirements.txt)
- 仮想環境 (--no-venv が指定されていない場合)

---

### 2. 開発サーバーの起動
プロジェクトに移動し、`agentcore dev` を実行してホットリロード付きのローカル開発サーバーを起動します。

**コマンド:**
```bash
cd <project-name>
agentcore dev
```

**オプション:**
- `--port, -p`: 開発サーバーのポート (デフォルト: 8080、使用中の場合は次に利用可能なポートを自動検出)
- `--env, -env`: 環境変数を設定 (形式: KEY=VALUE、複数回使用可能)

**実行されること:**
- `.bedrock_agentcore.yaml` からエントリポイントを自動検出するか、デフォルトの `src.main:app` を使用します。
- `uv run uvicorn` を使用して、ホットリロード付きのuvicornサーバーを起動します。
- ファイルの変更を監視し、自動的にリロードします。
- エージェントを `http://localhost:<port>/invocations` で利用可能にします。
- 0.0.0.0 にバインドします (すべてのネットワークインターフェースからアクセス可能)。

**例:**
```bash
# 基本的な使用法 (ポート8080または次に利用可能なポートを使用)
agentcore dev

# カスタムポートを使用
agentcore dev --port 3000

# 環境変数を使用
agentcore dev --env API_KEY=abc123 --env DEBUG=true

# 複数の環境変数
agentcore dev --env KEY1=value1 --env KEY2=value2
```

**サーバーの詳細:**
- デフォルトURL: `http://localhost:8080/invocations`
- `.bedrock_agentcore.yaml` からエントリポイントを自動検出します。
- 設定が見つからない場合やエラーが発生した場合は、`src.main:app` にフォールバックします。
- `uv run uvicorn` を使用します (個別のuvicornインストールは不要です)。
- `LOCAL_DEV=1` 環境変数を自動的に設定します。

**サーバーの停止:**
`Ctrl+C` を押して開発サーバーを停止します。

---

### 3. エージェントをローカルでテストする
開発サーバーが実行中に、`agentcore invoke --dev` を使用してエージェントをテストします。

**コマンド:**
```bash
agentcore invoke --dev '{"prompt": "Hello"}'
```

**オプション:**
- `--dev, -d`: ローカル開発サーバーにリクエストを送信します (ローカルテストに必須)。
- `--port`: ローカル開発サーバーのポート (デフォルト: 8080)。
- `--local, -l`: 実行中のローカルコンテナにリクエストを送信します。
- `--session-id, -s`: 会話の継続性のためのセッションID。
- `--bearer-token, -bt`: OAuth認証のためのベアラートークン。
- `--user-id, -u`: 認証フローのためのユーザーID。
- `--headers`: カスタムヘッダー (形式: 'Header1:value,Header2:value2')。
- `--agent, -a`: エージェント名 (複数のエージェントが設定されている場合)。

**ペイロード形式:**
- JSON文字列: `'{"prompt": "あなたのメッセージ"}'`
- プレーンテキスト: `'あなたのメッセージ'` (自動的に `{"prompt": "あなたのメッセージ"}` でラップされます)

**実行されること:**
- `http://localhost:<port>/invocations` にHTTP POSTリクエストを送信します。
- エージェントの応答をJSON形式で返します。
- 開発サーバーが実行されていない場合は、役立つエラーパネルを表示します。

**例:**
```bash
# JSONペイロード (デフォルトポート8080)
agentcore invoke --dev '{"prompt": "AWSとは何ですか？"}'

# プレーンテキスト (自動ラップ)
agentcore invoke --dev 'こんにちは、お元気ですか？'

# カスタムポート
agentcore invoke --dev --port 3000 '{"prompt": "Hello"}'

# 複雑なJSON
agentcore invoke --dev '{"prompt": "これを分析してください", "context": "追加データ"}'

# 会話の継続性のためのセッションID
agentcore invoke --dev --session-id abc123 '{"prompt": "会話を続けてください"}'
```

**期待される応答:**
```
✓ 開発サーバーからの応答:
{
  "response": "ここにエージェントの応答..."
}
```

**エラー処理:**
開発サーバーが実行されていない場合、以下が表示されます。
```
⚠️ 開発サーバーが見つかりません

http://localhost:8080 で開発サーバーが見つかりません

開始するには:
   agentcore create myproject
   cd myproject
   agentcore dev
   agentcore invoke --dev "Hello"
```

---

## 開発のベストプラクティス

### 変更のテスト
**重要:** コードを変更するたびに、`agentcore invoke --dev` を使用してローカルでテストし、以下を確認する必要があります。
- 変更が意図どおりに機能すること
- エラーが導入されていないこと
- エージェントが正しく応答すること

### 典型的な開発ループ
1. エージェントのコードを変更します。
2. ファイルを保存します (開発サーバーが自動的にリロードします)。
3. `agentcore invoke --dev '{"prompt": "テストメッセージ"}'` を実行します。
4. 応答を確認します。
5. 満足するまで繰り返します。

### トラブルシューティング

**開発サーバーが起動しない:**
- エントリポイントファイルが存在することを確認します (デフォルト: `src/main.py`)。
- コードの構文エラーを確認します。
- 依存関係がインストールされていることを確認します: `uv pip install -e .`
- `.bedrock_agentcore.yaml` が存在し、有効なエントリポイントを持っていることを確認します。
- 注: uvicornは `uv run uvicorn` を介して実行され、個別のインストールは不要です。

**Invokeが失敗する:**
- 開発サーバーが実行中であることを確認します。
- サーバーURLを確認します (デフォルト: `http://localhost:8080`)。
- ペイロード形式が有効なJSONであることを確認します。

**ポートの競合:**
- 開発サーバーは、8080が使用中の場合、次に利用可能なポートを自動的に見つけます。
- 使用されている実際のポートについては、コンソール出力を確認してください。

---

## AgentCore ランタイムへのデプロイ

### 既存のエージェントの場合 (agentcore create で作成されていないもの)
**重要:** `manage_agentcore_runtime` MCP ツールを使用して、完全なデプロイ要件を取得してください。

簡単な要約:
1. **エージェントコードを** BedrockAgentCoreApp パターンで**ラップします**。
2. `requirements.txt` を更新して `bedrock-agentcore` を含めます。
3. **設定:** `agentcore configure --entrypoint your_agent.py --non-interactive`
4. **デプロイ:** `agentcore launch`
5. **テスト:** `agentcore invoke '{"prompt": "Hello"}'`

### 新規プロジェクトの場合 (agentcore create で作成されたもの)
ローカル開発が完了したら:
1. **デプロイ用に設定:** `agentcore configure --entrypoint src/main.py`
2. **クラウドにデプロイ:** `agentcore launch`
3. **ステータスを確認:** `agentcore status`
4. **クラウドで呼び出し:** `agentcore invoke '{"prompt": "Hello"}'`
5. **セッションを停止:** `agentcore stop-session` (リソースを解放するため)
6. **リソースを破棄:** `agentcore destroy` (完了したら、最初に `--dry-run` を使用してください)

---

## クイックリファレンス

### 既存のエージェントの場合
```bash
# デプロイ要件を取得 (MCPツールを使用: manage_agentcore_runtime)
# その後、設定してデプロイ:
agentcore configure --entrypoint agent.py --non-interactive
agentcore launch
agentcore invoke '{"prompt": "Hello"}'
```

### 新規プロジェクトの場合
```bash
# 新規プロジェクトを作成 (AIアシスタントの場合は非対話型)
agentcore create --non-interactive --project-name MyAgent --template basic --agent-framework Strands --model-provider Bedrock

# 新規プロジェクトを作成 (人間ユーザーの場合は対話型)
agentcore create

# プロジェクトに移動
cd <project-name>

# 開発サーバーを起動 (1つのターミナルで)
agentcore dev

# ローカルでテスト (別のターミナルで)
agentcore invoke --dev '{"prompt": "test"}'

# デプロイ用に設定
agentcore configure --entrypoint src/main.py

# AWSにデプロイ
agentcore launch

# デプロイステータスを確認
agentcore status

# デプロイされたエージェントを呼び出し
agentcore invoke '{"prompt": "Hello"}'

# アクティブなセッションを停止
agentcore stop-session

# 破棄されるものをプレビュー
agentcore destroy --dry-run

# すべてのリソースを破棄
agentcore destroy

# 開発サーバーを停止
# 開発サーバーのターミナルで Ctrl+C を押します
```