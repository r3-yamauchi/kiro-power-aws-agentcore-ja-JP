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

- `uv run agentcore create` を使用して新しいエージェントをゼロから構築する場合
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

この機能は以下の MCP サーバーを提供します：

### agentcore-mcp-server
- `search_agentcore_docs` - AgentCore ドキュメントを検索
- `fetch_agentcore_doc` - 特定のドキュメントページを取得
- `manage_agentcore_runtime` - エージェントのランタイム構成を管理
- `manage_agentcore_memory` - エージェントのメモリ操作を処理
- `manage_agentcore_gateway` - エージェントのゲートウェイ設定を構成

### strands-agents MCP Server
- `search_docs` - Strands Agents SDK ドキュメントを検索
- `fetch_doc` - 特定の Strands SDK ドキュメントを取得

**Strands Agent 開発時の活用例:**
```
# Strands SDK のツール作成に関するドキュメントを検索
kiroPowers({
  "action": "use",
  "powerName": "strands-agents",
  "serverName": "strands-agents", 
  "toolName": "search_docs",
  "arguments": {
    "query": "tool creation agent framework"
  }
})

# MCP 統合に関する具体的なドキュメントを取得
kiroPowers({
  "action": "use",
  "powerName": "strands-agents",
  "serverName": "strands-agents",
  "toolName": "fetch_doc", 
  "arguments": {
    "doc_id": "mcp-integration"
  }
})
```

このMCPサーバーは、Strands Agents SDK の包括的なドキュメントへのアクセスを提供し、AI コーディングアシスタントと組み合わせて効率的な Strands Agent 開発を支援します。

## 利用可能なステアリングファイル

このパワーには以下のステアリングファイルが含まれています：

### 基本ガイド
- **getting-started** - 新規ユーザー向けの完全なセットアップガイド、プロジェクト作成、開発ワークフロー
- **agentcore-fundamentals** - Bedrock AgentCore の基本概念、アーキテクチャ、主要機能の包括的説明

### Gateway 関連ガイド
- **agentcore-gateway-integration** - Gateway の基本概念から実践的な統合手順まで包括的なガイド
- **agentcore-lambda-mcp-guide** - Lambda 関数の MCP 化の詳細ガイド
- **agentcore-openapi-mcp-guide** - OpenAPI 仕様の MCP 変換の詳細ガイド
- **agentcore-semantic-search-guide** - セマンティック検索機能の活用ガイド

### デプロイメントガイド
- **strands-agentcore-deployment** - Strands Agent フレームワークを AgentCore Runtime にデプロイする完全ガイド
- **mcp-agentcore-deployment** - MCP サーバーを AgentCore Runtime にデプロイして MCP ツールをホストする完全ガイド

### 高度な機能ガイド
- **agentcore-streaming-responses** - ストリーミングレスポンスの実装と WebSocket 通信の詳細ガイド
- **agentcore-session-context-management** - セッション管理とランタイムコンテキストの活用ガイド
- **agentcore-multimodal-guide** - マルチモーダルペイロード（テキスト、画像、音声）処理の完全ガイド

### 統合ガイド
- **agentcore-memory-integration** - Memory リソースの Strands エージェントとの統合ガイド

特定のワークフローについては、以下のようにステアリングファイルにアクセスしてください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "getting-started.md"
})
```

**「Bedrock AgentCore とは何か？」について詳しく知りたい場合：**
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-fundamentals.md"
})
```

**AgentCore Gateway の基本を理解したい場合：**
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-gateway-integration.md"
})
```

**Lambda 関数を MCP ツールにしたい場合：**
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-lambda-mcp-guide.md"
})
```

**OpenAPI を MCP ツールにしたい場合：**
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-openapi-mcp-guide.md"
})
```

**セマンティック検索を使いたい場合：**
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-semantic-search-guide.md"
})
```

**Strands Agent SDK を使用して Agent を実装し、 AgentCore にデプロイしたい場合：**
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "strands-agentcore-deployment.md"
})
```

**MCP サーバーを AgentCore にデプロイしたい場合：**
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "mcp-agentcore-deployment.md"
})
```

**AgentCore のストリーミング機能を使いたい場合：**
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-streaming-responses.md"
})
```

**AgentCore のセッション管理を使いたい場合：**
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-session-context-management.md"
})
```

**AgentCore のマルチモーダル処理を使いたい場合：**
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-multimodal-guide.md"
})
```

## はじめに

### 🌍 重要：AWS リージョンの設定

**推奨：特に理由がない限り `us-east-1` を使用してください。**

AgentCore は複数のリージョンをサポートしています。詳細なリージョン選択については `getting-started.md` を参照してください。

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

## 統合ガイド

### AgentCore Gateway
- **基本的な Gateway 管理：** フレームワークに依存しない CLI コマンドについては `manage_agentcore_gateway` MCP ツールを使用してください
- **Gateway の基本概念：** Gateway の基本的な理解については、以下のステアリングファイルを参照してください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-gateway-basics.md"
})
```
- **Gateway 統合の開始：** Gateway 統合の概要と専門ガイドへの導入については、以下のステアリングファイルを参照してください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-gateway-integration.md"
})
```
- **Lambda MCP 化：** Lambda 関数を MCP ツールに変換するには、以下のステアリングファイルを参照してください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-lambda-mcp-guide.md"
})
```
- **OpenAPI MCP 変換：** OpenAPI 仕様を MCP ツールに変換するには、以下のステアリングファイルを参照してください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-openapi-mcp-guide.md"
})
```
- **セマンティック検索：** インテリジェントなツール選択を実装するには、以下のステアリングファイルを参照してください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-semantic-search-guide.md"
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

### Strands Agent Framework
- **基本的な Strands 開発：** Strands フレームワークの基本的な使用方法については公式ドキュメントを参照
- **AgentCore への完全デプロイ：** Strands Agent を AgentCore Runtime にデプロイするには、以下のステアリングファイルを参照してください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "strands-agentcore-deployment.md"
})
```

### MCP Server Integration
- **基本的な MCP 開発：** Model Context Protocol の基本的な使用方法については公式ドキュメントを参照
- **AgentCore への完全デプロイ：** MCP サーバーを AgentCore Runtime にデプロイするには、以下のステアリングファイルを参照してください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "mcp-agentcore-deployment.md"
})
```

### Advanced AgentCore Features
- **基本的な AgentCore 機能：** 標準的な AgentCore 機能については基礎知識ガイドを参照
- **ストリーミングレスポンス：** リアルタイムストリーミング機能を実装するには、以下のステアリングファイルを参照してください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-streaming-responses.md"
})
```
- **セッション管理：** セッションとランタイムコンテキスト管理を実装するには、以下のステアリングファイルを参照してください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-session-context-management.md"
})
```
- **マルチモーダル処理：** テキスト、画像、音声などのマルチモーダルデータ処理を実装するには、以下のステアリングファイルを参照してください：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-multimodal-guide.md"
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

## クイックスタート

詳細な手順については `getting-started.md` を参照してください。

### 新規プロジェクト
```bash
# プロジェクト作成
uv run agentcore create --non-interactive --project-name MyAgent --region us-east-1
cd MyAgent

# 開発・テスト
uv run agentcore dev
uv run agentcore invoke --dev '{"prompt": "Hello"}'

# デプロイ
uv run agentcore configure --entrypoint src/main.py --region us-east-1
uv run agentcore launch --region us-east-1
```

### 既存エージェント
```bash
# デプロイ要件を確認（MCP ツール使用推奨）
uv run agentcore configure --entrypoint your_agent.py --region us-east-1
uv run agentcore launch --region us-east-1
```

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
- `uv` がインストールされていること（インストール方法は `getting-started.md` を参照）

### 環境構築

詳細な環境構築手順については `getting-started.md` を参照してください。

**基本的な手順:**
```bash
# uv プロジェクトを初期化
uv init
uv add bedrock-agentcore-starter-toolkit
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

### 基本ガイド
- **完全なセットアップガイド：** `getting-started.md`
- **AgentCore 基礎知識：** `agentcore-fundamentals.md`

### Gateway 関連
- **Gateway 統合概要：** `agentcore-gateway-integration.md`
- **Gateway 基本：** `agentcore-gateway-basics.md`
- **Lambda MCP 化：** `agentcore-lambda-mcp-guide.md`
- **OpenAPI MCP 変換：** `agentcore-openapi-mcp-guide.md`
- **セマンティック検索：** `agentcore-semantic-search-guide.md`

### デプロイメント
- **Strands Agent デプロイ：** `strands-agentcore-deployment.md`
- **MCP サーバーデプロイ：** `mcp-agentcore-deployment.md`

### 高度な機能
- **ストリーミング機能：** `agentcore-streaming-responses.md`
- **セッション管理：** `agentcore-session-context-management.md`
- **マルチモーダル処理：** `agentcore-multimodal-guide.md`

### 統合
- **Memory 統合：** `agentcore-memory-integration.md`

各ステアリングファイルには以下の内容が含まれています：
- ステップバイステップの手順
- 完全なコード例
- トラブルシューティングガイド
- ベストプラクティス

## よくある質問への対応

### 「Bedrock AgentCore とは何か？」
この質問には `agentcore-fundamentals.md` ステアリングファイルで包括的に回答します：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-fundamentals.md"
})
```

### 「Gateway とは何か？どう使うのか？」
この質問には `agentcore-gateway-basics.md` ステアリングファイルで詳細に回答します：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-gateway-basics.md"
})
```

### 「Lambda 関数を AI エージェントのツールにするには？」
この質問には `agentcore-lambda-mcp-guide.md` ステアリングファイルで詳細に回答します：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-lambda-mcp-guide.md"
})
```

### 「既存の REST API を AI エージェントで使うには？」
この質問には `agentcore-openapi-mcp-guide.md` ステアリングファイルで詳細に回答します：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-openapi-mcp-guide.md"
})
```

### 「大量のツールから最適なものを自動選択するには？」
この質問には `agentcore-semantic-search-guide.md` ステアリングファイルで詳細に回答します：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-semantic-search-guide.md"
})
```

このファイルには以下の内容が含まれています：
- AgentCore の概要と主要特徴
- アーキテクチャとコアコンポーネント
- メモリシステムとゲートウェイシステム
- 開発ライフサイクルと実装パターン
- ベストプラクティスとトラブルシューティング

### 「Strands Agent を AgentCore にデプロイするには？」
この質問には `strands-agentcore-deployment.md` ステアリングファイルで詳細に回答します：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "strands-agentcore-deployment.md"
})
```

このファイルには以下の内容が含まれています：
- Strands Agent フレームワークの概要
- AgentCore Runtime との統合方法
- 完全な実装例とコード
- カスタムツールとメモリプロバイダーの作成
- 監視、ログ、トラブルシューティング

### 「MCP サーバーを AgentCore にデプロイするには？」
この質問には `mcp-agentcore-deployment.md` ステアリングファイルで詳細に回答します：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "mcp-agentcore-deployment.md"
})
```

このファイルには以下の内容が含まれています：
- Model Context Protocol (MCP) の概要
- MCP サーバーの実装方法
- AgentCore Runtime との統合
- カスタムツール、リソース、プロンプトの作成
- エージェントとの統合パターン
- 監視、ログ、トラブルシューティング

### 「AgentCore のストリーミング機能を使うには？」
この質問には `agentcore-streaming-responses.md` ステアリングファイルで詳細に回答します：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-streaming-responses.md"
})
```

このファイルには以下の内容が含まれています：
- ストリーミングレスポンスの実装方法
- WebSocket 通信の設定と管理
- リアルタイムデータ配信
- ストリーミング API の活用
- パフォーマンス最適化とベストプラクティス

### 「AgentCore のセッション管理を使うには？」
この質問には `agentcore-session-context-management.md` ステアリングファイルで詳細に回答します：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-session-context-management.md"
})
```

このファイルには以下の内容が含まれています：
- セッション管理の実装方法
- ランタイムコンテキストの活用
- 会話の継続性とメモリ管理
- セッション状態の永続化
- 高度なコンテキスト管理パターン

### 「AgentCore のマルチモーダル処理を使うには？」
この質問には `agentcore-multimodal-guide.md` ステアリングファイルで詳細に回答します：
```
kiroPowers({
  "action": "readSteering",
  "powerName": "aws-agentcore",
  "steeringFile": "agentcore-multimodal-guide.md"
})
```

このファイルには以下の内容が含まれています：
- マルチモーダルペイロード処理の実装
- テキスト、画像、音声データの統合処理
- 大容量データの効率的な処理方法
- メディアファイルの変換と最適化
- 高度なマルチモーダル統合パターン
