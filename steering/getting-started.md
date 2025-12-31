# AgentCore の使用開始

## 前提条件

### 🌍 重要：AWS リージョンの選択

**AgentCore を使用開始する前に、使用する AWS リージョンを決定してください。**

#### サポートされているリージョン
- **us-east-1** (バージニア北部) - **推奨** - 最も多くの AWS サービスが利用可能で、最新機能が最初に提供される
- **us-west-2** (オレゴン) - 西海岸での低レイテンシ
- **ap-northeast-1** (東京) - 日本国内での低レイテンシ
- **eu-west-1** (アイルランド) - ヨーロッパでの低レイテンシ
- その他のサポートされているリージョン

**推奨：特に理由がない限り `us-east-1` を使用してください。**

#### リージョン選択の考慮事項
1. **サービス可用性**: `us-east-1` は最も多くの AWS サービスと最新機能が利用可能
2. **地理的な近さ**: 低レイテンシのため、最寄りのリージョンを選択
3. **Bedrock モデルの可用性**: 使用したいモデルが利用可能なリージョン
4. **コンプライアンス要件**: データの保存場所に関する規制要件
5. **コスト**: リージョンによって料金が異なる場合があります

#### リージョンの設定方法
すべての AgentCore コマンドで `--region` パラメータを使用してください：
```bash
# 例：東京リージョンを使用
uv run agentcore create --project-name MyAgent --region ap-northeast-1

# 例：バージニア北部リージョンを使用（推奨）
uv run agentcore create --project-name MyAgent --region us-east-1
```

**このガイドでは推奨の `us-east-1` を使用しますが、必要に応じて他のリージョンに置き換えてください。**

### 環境構築（uv を使用）

#### uv のインストール

まず、`uv` がインストールされていることを確認してください：

```bash
uv --version
```

`uv` がインストールされていない場合は、以下の方法でインストールしてください：

**macOS/Linux:**
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Windows:**
```powershell
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

**Homebrew (macOS):**
```bash
brew install uv
```

詳細なインストール手順については、[uv 公式ドキュメント](https://docs.astral.sh/uv/getting-started/installation/) を参照してください。

#### プロジェクト環境の構築

```bash
# 新しいプロジェクトディレクトリを作成
mkdir my-agentcore-project
cd my-agentcore-project

# uv プロジェクトを初期化
uv init

# bedrock-agentcore-starter-toolkit を追加
uv add bedrock-agentcore-starter-toolkit

# 以降のコマンドは uv run を使用して実行
uv run agentcore --help
```

### uv の利点
- **高速なパッケージ管理**: pip より大幅に高速
- **自動仮想環境管理**: 仮想環境の作成と管理を自動化
- **依存関係の解決**: より効率的な依存関係管理
- **プロジェクト管理**: `pyproject.toml` ベースの現代的なプロジェクト管理

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

## Strands Agent 開発での MCP サーバー活用

Strands Agent を開発する際は、利用可能な `strands-agents` MCP サーバーを活用して Strands Agents SDK の包括的なドキュメントを検索できます：

### Strands SDK ドキュメントの検索
```
# ツール作成に関するドキュメントを検索
kiroPowers({
  "action": "use",
  "powerName": "strands-agents",
  "serverName": "strands-agents",
  "toolName": "search_docs", 
  "arguments": {
    "query": "tool creation framework"
  }
})

# MCP 統合に関するドキュメントを検索
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

### 特定ドキュメントの取得
```
kiroPowers({
  "action": "use",
  "powerName": "strands-agents", 
  "serverName": "strands-agents",
  "toolName": "fetch_doc",
  "arguments": {
    "doc_id": "getting-started"
  }
})
```

この MCP サーバーは、Strands Agents SDK の包括的なドキュメントへのアクセスを提供し、AI コーディングアシスタントと組み合わせて効率的な開発を支援します。開発中に疑問が生じた際に、公式ドキュメントから適切な情報を素早く取得できます。

## 2つのパス: 新規プロジェクト vs 既存エージェント
### パス 1: 既存エージェントのデプロイ

**すでにエージェントをお持ちの場合**、`uv run agentcore create` は使用しないでください。代わりに:
   ```
   kiroPowers({
     "action": "use",
     "powerName": "aws-agentcore",
     "serverName": "agentcore-mcp-server",
     "toolName": "manage_agentcore_runtime",
     "arguments": {}
   })
   ```
   これにより、既存エージェントの完全なデプロイガイドが提供されます。

2. **既存エージェントの主要な要件:**
   - エージェントを `BedrockAgentCoreApp` でラップします。
   - `@app.entrypoint` デコレータを追加します。
   - `requirements.txt` に `bedrock-agentcore` を含めます。
   - その後、`uv run agentcore configure` と `uv run agentcore launch` を使用します。

**重要:** `uv run agentcore create` コマンドは、新しいプロジェクトをゼロから作成する場合にのみ使用します。既存のエージェントに対して使用すると、コードが上書きされます。

---

### パス 2: ゼロからの新規プロジェクト作成
`uv run agentcore create` を使用して、新しいエージェントプロジェクトのワークスペースを初期化します。

**AI アシスタントにとって重要：** このコマンドをプログラムで実行する場合は、常に `--non-interactive` モードを使用してください。インタラクティブモードは、人間が手動でコマンドを実行する場合にのみ使用します。

**デフォルトの動作：** ユーザーが特定のオプション（テンプレート、エージェントフレームワーク、モデルプロバイダー）を指定しない場合、デフォルトを使用します：
- テンプレート：`basic`
- エージェントフレームワーク：`Strands`
- モデルプロバイダー：`Bedrock`

つまり、`uv run agentcore create --non-interactive --project-name <name>` を実行するだけで、すべてのデフォルトが使用されます。

**コマンド:**
```bash
uv run agentcore create --non-interactive --project-name <name> --template <template> --agent-framework <framework> --model-provider <provider>
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

**例：**
```bash
# AI アシスタントの使用（非対話型、デフォルトの basic テンプレート）
uv run agentcore create --non-interactive --project-name MyAgent --template basic --agent-framework Strands --model-provider Bedrock --region us-east-1

# カスタムエージェントフレームワークを使用
uv run agentcore create --non-interactive --project-name MyAgent --template basic --agent-framework ClaudeAgents --model-provider Bedrock --region us-east-1

# IaC を含む production テンプレート
uv run agentcore create --non-interactive --project-name MyAgent --template production --agent-framework Strands --model-provider Bedrock --iac CDK --region us-east-1

# 最小限のコマンド（すべてのデフォルトを使用）
uv run agentcore create --non-interactive --project-name MyAgent --region us-east-1

# 人間ユーザーのインタラクティブモード（手動使用のみ）
uv run agentcore create
```

**注意：** `us-east-1` の部分は、必要に応じて他のリージョンに置き換えてください。

**作成されるもの：**
- 選択した名前のプロジェクトディレクトリ
- エージェント SDK のボイラープレートコード
- 設定ファイル（.bedrock_agentcore.yaml）
- 依存関係の設定（pyproject.toml または requirements.txt）
- 仮想環境（--no-venv が指定されていない場合）

**uv 環境での注意点：**
- `uv run agentcore create` で作成されたプロジェクトでも、`uv` を使用して依存関係を管理できます
- 既存の `requirements.txt` がある場合は `uv add --requirements requirements.txt` で移行可能
- `uv run` を使用してコマンドを実行することで、自動的に適切な環境で実行されます

---

### 2. 開発サーバーの起動
プロジェクトに移動し、`uv run agentcore dev` を実行してホットリロード付きのローカル開発サーバーを起動します。

**コマンド:**
```bash
cd <project-name>

# 開発サーバーを起動
uv run agentcore dev
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
# 基本的な使用法
uv run agentcore dev

# カスタムポートを使用
uv run agentcore dev --port 3000

# 環境変数を使用
uv run agentcore dev --env API_KEY=abc123 --env DEBUG=true

# 複数の環境変数
uv run agentcore dev --env KEY1=value1 --env KEY2=value2
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
開発サーバーが実行中に、`uv run agentcore invoke --dev` を使用してエージェントをテストします。

**コマンド:**
```bash
uv run agentcore invoke --dev '{"prompt": "Hello"}'
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
# JSON ペイロード
uv run agentcore invoke --dev '{"prompt": "AWS とは何ですか？"}'

# プレーンテキスト（自動ラップ）
uv run agentcore invoke --dev 'こんにちは、お元気ですか？'

# カスタムポート
uv run agentcore invoke --dev --port 3000 '{"prompt": "Hello"}'

# 複雑な JSON
uv run agentcore invoke --dev '{"prompt": "これを分析してください", "context": "追加データ"}'

# 会話の継続性のためのセッション ID
uv run agentcore invoke --dev --session-id abc123 '{"prompt": "会話を続けてください"}'
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
   uv run agentcore create myproject
   cd myproject
   uv run agentcore dev
   uv run agentcore invoke --dev "Hello"
```

---

## 開発のベストプラクティス

### 変更のテスト
**重要:** コードを変更するたびに、`uv run agentcore invoke --dev` を使用してローカルでテストし、以下を確認する必要があります。
- 変更が意図どおりに機能すること
- エラーが導入されていないこと
- エージェントが正しく応答すること

### 典型的な開発ループ
1. エージェントのコードを変更します。
2. ファイルを保存します (開発サーバーが自動的にリロードします)。
3. `uv run agentcore invoke --dev '{"prompt": "テストメッセージ"}'` を実行します。
4. 応答を確認します。
5. 満足するまで繰り返します。

### トラブルシューティング

**開発サーバーが起動しない:**
- エントリポイントファイルが存在することを確認します（デフォルト：`src/main.py`）
- コードの構文エラーを確認します
- `uv` がインストールされていることを確認します：`uv --version`
- 依存関係がインストールされていることを確認します：`uv sync` または `uv add <package-name>`
- `.bedrock_agentcore.yaml` が存在し、有効なエントリポイントを持っていることを確認します
- 注：uvicorn は `uv run uvicorn` を介して実行され、個別のインストールは不要です

**Invokeが失敗する:**
- 開発サーバーが実行中であることを確認します。
- サーバーURLを確認します (デフォルト: `http://localhost:8080`)。
- ペイロード形式が有効なJSONであることを確認します。

**ポートの競合:**
- 開発サーバーは、8080が使用中の場合、次に利用可能なポートを自動的に見つけます。
- 使用されている実際のポートについては、コンソール出力を確認してください。

---

## AgentCore ランタイムへのデプロイ

### 既存のエージェントの場合（agentcore create で作成されていないもの）
**重要：** `manage_agentcore_runtime` MCP ツールを使用して、完全なデプロイ要件を取得してください：

```
kiroPowers({
  "action": "use",
  "powerName": "aws-agentcore",
  "serverName": "agentcore-mcp-server",
  "toolName": "manage_agentcore_runtime",
  "arguments": {}
})
```

簡単な要約：
1. **エージェントコードを** BedrockAgentCoreApp パターンで**ラップします**
2. `requirements.txt` を更新して `bedrock-agentcore` を含めます
3. **設定：** `uv run agentcore configure --entrypoint your_agent.py --non-interactive --region us-east-1`
4. **デプロイ：** `uv run agentcore launch --region us-east-1`
5. **テスト：** `uv run agentcore invoke '{"prompt": "Hello"}' --region us-east-1`

**注意：** `us-east-1` の部分は、必要に応じて他のリージョンに置き換えてください。

### 新規プロジェクトの場合（agentcore create で作成されたもの）
ローカル開発が完了したら：
1. **デプロイ用に設定：** `uv run agentcore configure --entrypoint src/main.py --region us-east-1`
2. **クラウドにデプロイ：** `uv run agentcore launch --region us-east-1`
3. **ステータスを確認：** `uv run agentcore status --region us-east-1`
4. **クラウドで呼び出し：** `uv run agentcore invoke '{"prompt": "Hello"}' --region us-east-1`
5. **セッションを停止：** `uv run agentcore stop-session --region us-east-1`（リソースを解放するため）
6. **リソースを破棄：** `uv run agentcore destroy --region us-east-1`（完了したら、最初に `--dry-run` を使用してください）

**注意：** `us-east-1` の部分は、必要に応じて他のリージョンに置き換えてください。

---

## クイックリファレンス

### 既存のエージェントの場合
```bash
# デプロイ要件を取得（MCP ツールを使用：manage_agentcore_runtime）
# その後、設定してデプロイ：

uv run agentcore configure --entrypoint agent.py --non-interactive --region us-east-1
uv run agentcore launch --region us-east-1
uv run agentcore invoke '{"prompt": "Hello"}' --region us-east-1
```

### 新規プロジェクトの場合
```bash
# 新規プロジェクトを作成（AI アシスタントの場合は非対話型）
uv run agentcore create --non-interactive --project-name MyAgent --template basic --agent-framework Strands --model-provider Bedrock

# 新規プロジェクトを作成（人間ユーザーの場合は対話型）
uv run agentcore create

# プロジェクトに移動
cd <project-name>

# uv 環境に移行する場合（オプション）
uv init --no-readme
uv add --requirements requirements.txt

# 開発サーバーを起動（1つのターミナルで）
uv run agentcore dev

# ローカルでテスト（別のターミナルで）
uv run agentcore invoke --dev '{"prompt": "test"}'

# デプロイ用に設定
uv run agentcore configure --entrypoint src/main.py --region us-east-1

# AWS にデプロイ
uv run agentcore launch --region us-east-1

# デプロイステータスを確認
uv run agentcore status --region us-east-1

# デプロイされたエージェントを呼び出し
uv run agentcore invoke '{"prompt": "Hello"}' --region us-east-1

# アクティブなセッションを停止
uv run agentcore stop-session --region us-east-1

# 破棄されるものをプレビュー
uv run agentcore destroy --dry-run --region us-east-1

# すべてのリソースを破棄
uv run agentcore destroy --region us-east-1

# 開発サーバーを停止
# 開発サーバーのターミナルで Ctrl+C を押します
```