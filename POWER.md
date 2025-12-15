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

## はじめに

**新規ユーザーまたは新しいエージェントを構築する場合:** 前提条件、プロジェクト作成、開発ワークフロー、デプロイに関する完全なステップバイステップのガイダンスについては、getting-started ステアリングファイルを使用してください。`readPowerSteering("agentcore", "getting-started.md")` でアクセスできます。

**既存のエージェントをデプロイする場合:** `manage_agentcore_runtime` MCP ツールを使用して、既存のエージェントを AgentCore ランタイムにラップしてデプロイするための完全なデプロイ要件と手順を取得します。

## 統合ガイド

**AgentCore Gateway:**
- Gateway リソースを作成および管理するには、フレームワークに依存しない CLI コマンドについては `manage_agentcore_gateway` MCP ツールを使用してください。
- Gateway を Strands エージェントと完全に統合するには、`readPowerSteering("agentcore", "agentcore-gateway-integration.md")` を使用してください。

**AgentCore Memory:**
- Memory リソースを作成および管理するには、フレームワークに依存しない CLI コマンドについては `manage_agentcore_memory` MCP ツールを使用してください。
- Memory を Strands エージェントと完全に統合するには、`readPowerSteering("agentcore", "agentcore-memory-integration.md")` を使用してください。

## MCP ツールの使用

### ドキュメントの検索

AgentCore ドキュメントを検索するには:
```
usePower("agentcore", "agentcore-mcp-server", "search_agentcore_docs", {
  "query": "deployment configuration"
})
```

### 特定のドキュメントの取得

```
usePower("agentcore", "agentcore-mcp-server", "fetch_agentcore_doc", {
  "doc_id": "getting-started"
})
```

### AgentCore ランタイムの管理

デプロイ要件とランタイム構成を取得します。
```
usePower("agentcore", "agentcore-mcp-server", "manage_agentcore_runtime", {})
```

### AgentCore メモリの管理

メモリリソースの作成と CLI コマンドを取得します。
```
usePower("agentcore", "agentcore-mcp-server", "manage_agentcore_memory", {})
```

### AgentCore Gateway の管理

ゲートウェイ構成とデプロイ手順を取得します。
```
usePower("agentcore", "agentcore-mcp-server", "manage_agentcore_gateway", {})
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

## その他のリソース

詳細な開発ワークフローガイダンスについては、以下の内容をカバーするステアリングファイルを参照してください。
- 完全なプロジェクト作成オプション
- 開発サーバー構成
- テスト戦略
- デプロイのベストプラクティス

完全なガイドについては、`readPowerSteering("agentcore", "getting-started")` を使用してください。
