---
name: "aws-agentcore"
displayName: "Amazon Bedrock AgentCore でエージェントを構築"
description: "ローカル開発ワークフローで AWS Bedrock AgentCore を使用して AI エージェントを構築、テスト、デプロイします。Amazon Bedrock AgentCore は、効果的なエージェントを構築、デプロイ、運用するためのエージェントプラットフォームです。"
keywords: ["agentcore", "bedrock", "aws", "agents", "ai", "development", "agent"]
author: "AWS"
---

# AWS Bedrock AgentCore

## 概要

完全なローカル開発ワークフローで AWS Bedrock AgentCore を使用して AI エージェントを構築およびデプロイします。
この機能は、MCP ツールを介した AgentCore ドキュメント、ランタイム管理、メモリ操作、ゲートウェイ構成へのアクセス、および作成-開発-テスト-デプロイサイクルに関する包括的なガイダンスを提供します。

AgentCore は、複数のエージェント SDK (Strands、Claude、OpenAI) とモデルプロバイダー (Bedrock、OpenAI) をサポートし、CDK または Terraform を介したインフラストラクチャデプロイメントをサポートします。

## この機能を使用するタイミング

- `agentcore create` を使用して新しいエージェントをゼロから構築する場合
- エージェント開発を開始し、ワークフローに関するガイダンスが必要な場合
- 既存のエージェントを AgentCore ランタイムにデプロイする場合
- AgentCore プリミティブ (メモリ、ゲートウェイ) を既存のエージェントに統合する場合
- ホットリロードでローカル開発サーバーを起動する場合
- クラウドデプロイメントの前にエージェントをローカルでテストする場合
- AgentCore ドキュメントを検索する場合
- エージェントのランタイム、メモリ、ゲートウェイ構成を管理する場合
- エージェントを AWS にデプロイする場合
- Strands エージェントフレームワークを使用する場合

## 利用可能な MCP ツール

この機能は agentcore-mcp-server を提供します。
- `search_agentcore_docs` - AgentCore ドキュメントを検索
- `fetch_agentcore_doc` - 特定のドキュメントページを取得
- `manage_agentcore_runtime` - エージェントのランタイム構成を管理
- `manage_agentcore_memory` - エージェントのメモリ操作を処理
- `manage_agentcore_gateway` - エージェントのゲートウェイ設定を構成

## 利用可能なステアリングファイル

このパワーには以下のステアリングファイルが含まれています：

- **getting-started** - 新規ユーザー向けの完全なセットアップガイド、プロジェクト作成、開発ワークフロー
- **agentcore-gateway-integration** - Gateway リソースの Strands エージェントとの統合ガイド
- **agentcore-memory-integration** - Memory リソースの Strands エージェントとの統合ガイド

特定のワークフローについては、以下のようにステアリングファイルにアクセスしてください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "getting-started.md"
})
```

## はじめに

### 🌍 重要：AWS リージョンの設定

**AgentCore を使用する前に、使用する AWS リージョンを決定してください。**

AgentCore は以下のリージョンをサポートしています：
- **us-east-1** (バージニア北部) - 最も多くの AWS サービスが利用可能
- **us-west-2** (オレゴン) - 西海岸での低レイテンシ
- **ap-northeast-1** (東京) - 日本国内での低レイテンシ
- **eu-west-1** (アイルランド) - ヨーロッパでの低レイテンシ
- その他のサポートされているリージョン

**どのリージョンを使用しますか？**

リージョン選択の考慮事項：
- **地理的な近さ**: 低レイテンシのため、最寄りのリージョンを選択
- **Bedrock モデルの可用性**: 使用したいモデルが利用可能なリージョン
- **コンプライアンス要件**: データの保存場所に関する規制要件
- **コスト**: リージョンによって料金が異なる場合があります

**リージョンを決定したら、すべての AgentCore コマンドで `--region` パラメータを使用してください。**

### 📚 ガイダンス

**新規ユーザーまたは新しいエージェントを構築する場合：** 前提条件、プロジェクト作成、開発ワークフロー、デプロイに関する完全なステップバイステップのガイダンスについては、getting-started ステアリングファイルを使用してください：

```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "getting-started.md"
})
```

**既存のエージェントをデプロイする場合：** `manage_agentcore_runtime` MCP ツールを使用して、既存のエージェントを AgentCore ランタイムにラップしてデプロイするための完全なデプロイ要件と手順を取得します。

## 統合ガイド

### AgentCore Gateway
- **基本的な Gateway 管理：** フレームワークに依存しない CLI コマンドについては `manage_agentcore_gateway` MCP ツールを使用してください
- **Strands との完全統合：** Gateway を Strands エージェントと統合するには、以下のステアリングファイルを参照してください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-gateway-integration.md"
})
```

### AgentCore Memory
- **基本的な Memory 管理：** フレームワークに依存しない CLI コマンドについては `manage_agentcore_memory` MCP ツールを使用してください
- **Strands との完全統合：** Memory を Strands エージェントと統合するには、以下のステアリングファイルを参照してください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-memory-integration.md"
})
```

## MCP ツールの使用

### ドキュメントの検索

AgentCore ドキュメントを検索するには：
```
kiroPowers({
  "action": "use",
  "powerName": "aws-agentcore",
  "serverName": "agentcore-mcp-server",
  "toolName": "search_agentcore_docs",
  "arguments": {
    "query": "deployment configuration"
  }
})
```

### 特定のドキュメントの取得

```
kiroPowers({
  "action": "use",
  "powerName": "aws-agentcore",
  "serverName": "agentcore-mcp-server",
  "toolName": "fetch_agentcore_doc",
  "arguments": {
    "doc_id": "getting-started"
  }
})
```

### AgentCore ランタイムの管理

デプロイ要件とランタイム構成を取得します：
```
kiroPowers({
  "action": "use",
  "powerName": "aws-agentcore",
  "serverName": "agentcore-mcp-server",
  "toolName": "manage_agentcore_runtime",
  "arguments": {}
})
```

### AgentCore メモリの管理

メモリリソースの作成と CLI コマンドを取得します：
```
kiroPowers({
  "action": "use",
  "powerName": "aws-agentcore",
  "serverName": "agentcore-mcp-server",
  "toolName": "manage_agentcore_memory",
  "arguments": {}
})
```

### AgentCore Gateway の管理

ゲートウェイ構成とデプロイ手順を取得します：
```
kiroPowers({
  "action": "use",
  "powerName": "aws-agentcore",
  "serverName": "agentcore-mcp-server",
  "toolName": "manage_agentcore_gateway",
  "arguments": {}
})
```

## クイックスタートワークフロー

### 新規エージェントプロジェクト（uv 使用・推奨）
```bash
# 1. uv 環境でプロジェクトを作成
mkdir MyAgent && cd MyAgent
uv init
uv add bedrock-agentcore-starter-toolkit

# 2. AgentCore プロジェクトを作成（リージョンを指定）
uv run agentcore create --non-interactive --project-name MyAgent --region ap-northeast-1

# 3. 作成されたプロジェクトに移動
cd MyAgent

# 4. 開発サーバーを起動
uv run agentcore dev

# 5. 別のターミナルでローカルテスト
uv run agentcore invoke --dev '{"prompt": "Hello"}'

# 6. デプロイ用に設定（リージョンを指定）
uv run agentcore configure --entrypoint src/main.py --region ap-northeast-1

# 7. AWS にデプロイ（リージョンを指定）
uv run agentcore launch --region ap-northeast-1

# 8. デプロイされたエージェントをテスト（リージョンを指定）
uv run agentcore invoke '{"prompt": "Hello"}' --region ap-northeast-1
```

**注意：** `ap-northeast-1` の部分は、選択したリージョンに置き換えてください（例：`us-east-1`、`us-west-2`、`eu-west-1` など）

### 新規エージェントプロジェクト（従来方式）
```bash
# 1. 新しいプロジェクトを作成（リージョンを指定）
agentcore create --non-interactive --project-name MyAgent --region ap-northeast-1

# 2. プロジェクトディレクトリに移動
cd MyAgent

# 3. 開発サーバーを起動
agentcore dev

# 4. 別のターミナルでローカルテスト
agentcore invoke --dev '{"prompt": "Hello"}'

# 5. デプロイ用に設定（リージョンを指定）
agentcore configure --entrypoint src/main.py --region ap-northeast-1

# 6. AWS にデプロイ（リージョンを指定）
agentcore launch --region ap-northeast-1

# 7. デプロイされたエージェントをテスト（リージョンを指定）
agentcore invoke '{"prompt": "Hello"}' --region ap-northeast-1
```

**注意：** `ap-northeast-1` の部分は、選択したリージョンに置き換えてください（例：`us-east-1`、`us-west-2`、`eu-west-1` など）

### 既存エージェントのデプロイ
```bash
# 1. uv 環境を構築（推奨）
uv init
uv add bedrock-agentcore-starter-toolkit

# 2. デプロイ要件を確認（MCP ツールを使用）
# manage_agentcore_runtime ツールを呼び出してガイダンスを取得

# 3. エージェントを BedrockAgentCoreApp でラップ
# 4. 依存関係に bedrock-agentcore を追加
# 5. 設定とデプロイ（リージョンを指定）
uv run agentcore configure --entrypoint your_agent.py --non-interactive --region ap-northeast-1
uv run agentcore launch --region ap-northeast-1
uv run agentcore invoke '{"prompt": "Hello"}' --region ap-northeast-1
```

**注意：** `ap-northeast-1` の部分は、選択したリージョンに置き換えてください（例：`us-east-1`、`us-west-2`、`eu-west-1` など）

## トラブルシューティング

### 開発サーバーが起動しない

**エラー:** `Could not find entrypoint module`
**解決策:** `.bedrock_agentcore.yaml` が存在することを確認するか、エントリポイントを手動で指定してください。

**エラー:** `Port 8080 already in use`
**解決策:** CLI は自動的に次の利用可能なポートを試行します。

### ローカル呼び出しが失敗する

**エラー:** `Connection refused`
**解決策:** `agentcore dev` で開発サーバーが実行されていることを確認してください。

**エラー:** `Invalid JSON payload`
**解決策:** 適切な JSON 形式を使用するか、プレーンテキスト (自動的にラップされます) を使用してください。

### デプロイの問題

**エラー:** `AWS authentication failed`
**解決策:** `aws login` を実行して認証してください。

**エラー:** `Model access denied`
**解決策:** AWS コンソールで Bedrock モデルの権限があることを確認してください。

### MCP サーバー接続の問題

**問題:** MCP サーバーが起動しない、または接続できない
**症状:**
- エラー：「Connection refused」
- サーバーが応答しない

**解決策:**
1. インストールを確認：`uvx awslabs.amazon-bedrock-agentcore-mcp-server@latest`
2. 環境変数が正しく設定されていることを確認
3. 特定のエラーについてはログを確認
4. Kiro を再起動して再試行

## ベストプラクティス

### 開発ワークフロー
- コードを変更するたびに `agentcore invoke --dev` でローカルテストを実行
- デプロイ前に十分なローカルテストを実施
- セッション ID を使用して会話の継続性をテスト
- 本番デプロイ後は `agentcore stop-session` でリソースを適切に管理

### メモリ統合
- 初回デプロイ時は `mode: NO_MEMORY` を使用
- エージェントが正常にデプロイされた後にメモリ機能を有効化
- メモリ設定変更後は必ず再デプロイを実行

### Gateway 設定
- Gateway 作成前に認証要件を明確化
- OpenAPI 仕様を事前に準備
- Lambda ターゲットの権限設定を確認

### リソース管理
- 開発完了後は `agentcore destroy --dry-run` で削除対象を確認
- 不要なリソースは `agentcore destroy` で適切に削除
- 定期的に `agentcore status` でリソース状況を確認

## 設定

### 前提条件
- AWS CLI がインストールされ、適切に設定されていること
- Python 3.8+ がインストールされていること
- `uv` がインストールされていること（推奨）または `pip`

### 環境構築（推奨：uv を使用）

**uv を使用した仮想環境の作成：**
```bash
# 新しいプロジェクトディレクトリを作成
mkdir my-agentcore-project
cd my-agentcore-project

# uv プロジェクトを初期化
uv init

# bedrock-agentcore-starter-toolkit を追加
uv add bedrock-agentcore-starter-toolkit

# 仮想環境をアクティベート（自動的に作成されます）
# uv は自動的に仮想環境を管理します
```

**従来の pip を使用する場合：**
```bash
# 仮想環境を作成
python -m venv venv

# 仮想環境をアクティベート
# macOS/Linux:
source venv/bin/activate
# Windows:
# venv\Scripts\activate

# パッケージをインストール
pip install bedrock-agentcore-starter-toolkit
```

### 環境変数
- AWS 認証情報が適切に設定されていること（AWS CLI、環境変数、または IAM ロール）
- Bedrock モデルへのアクセス権限があること

### uv の利点
- **高速なパッケージ管理**: pip より大幅に高速
- **自動仮想環境管理**: 仮想環境の作成と管理を自動化
- **依存関係の解決**: より効率的な依存関係管理
- **プロジェクト管理**: `pyproject.toml` ベースの現代的なプロジェクト管理

**追加設定は不要** - MCP サーバーが Kiro にインストールされた後、すぐに動作します。

## その他のリソース

詳細な開発ワークフローガイダンスについては、ステアリングファイルを参照してください：

- **完全なセットアップガイド：** `getting-started.md`
- **Gateway 統合：** `agentcore-gateway-integration.md`  
- **Memory 統合：** `agentcore-memory-integration.md`

各ステアリングファイルには以下の内容が含まれています：
- ステップバイステップの手順
- 完全なコード例
- トラブルシューティングガイド
- ベストプラクティス
