# Amazon Bedrock AgentCore 用 Kiro power の 日本語訳

これは [AgentCore 用 Kiro power](https://github.com/kirodotdev/powers/tree/main/aws-agentcore) の日本語訳です。
この Kiro power は Amazon Bedrock AgentCore を用いて AI エージェントをローカルで開発し、テスト・デプロイすることを支援します。

## ライセンス

オリジナルリポジトリ (https://github.com/kirodotdev/powers) で現状ライセンスが明示されていません。
あくまでオリジナル版が著作権を所有しており、本翻訳は参考用としてください。

## そもそも Kiro powers とは

Kiro powers は Kiro に特定の専門知識や技能を瞬時に与える拡張機能です。
MCPツール、ステアリングファイル、フックを一つのパッケージにまとめたもので Kiro にワンクリックでインストールできます。
そして、必要な機能（power）だけが会話の文脈に応じて動的に読み込まれます。
例えば、ユーザーが「ドキュメントを検索して」と指示すると、Knowledge Base Powerが有効化されます。
これにより Kiro は 不要な情報による混乱（コンテキストの過負荷）を防ぎ、高速かつ高品質な応答を維持します。
今後様々な、各分野に特化した power が開発・提供されていくことにより Kiro は各分野の専門家になることを期待できます。

## 内容物

- `POWER.md`: power の定義と利用可能な MCP ツールの概要
- `mcp.json`: 必要に応じて利用する MCP Server の定義

### ステアリングファイル

#### 基本ガイド
- `steering/getting-started.md`: 前提条件、`agentcore create` の使い方、ローカル開発・呼び出し・デプロイの流れ
- `steering/agentcore-fundamentals.md`: Bedrock AgentCore の基本概念、アーキテクチャ、主要機能の包括的説明

#### Gateway 関連ガイド
- `steering/agentcore-gateway-integration.md`: Gateway（MCP エンドポイント）で Lambda / OpenAPI / Smithy / MCP サーバーをツール化する手順
- `steering/agentcore-lambda-mcp-guide.md`: Lambda 関数の MCP 化の詳細ガイド
- `steering/agentcore-openapi-mcp-guide.md`: OpenAPI 仕様の MCP 変換の詳細ガイド
- `steering/agentcore-semantic-search-guide.md`: セマンティック検索機能の活用ガイド

#### デプロイメントガイド
- `steering/strands-agentcore-deployment.md`: Strands Agent フレームワークを AgentCore Runtime にデプロイする完全ガイド
- `steering/mcp-agentcore-deployment.md`: MCP サーバーを AgentCore Runtime にデプロイして MCP ツールをホストする完全ガイド

#### 高度な機能ガイド
- `steering/agentcore-streaming-responses.md`: ストリーミングレスポンスの実装と WebSocket 通信の詳細ガイド
- `steering/agentcore-session-context-management.md`: セッション管理とランタイムコンテキストの活用ガイド
- `steering/agentcore-multimodal-guide.md`: マルチモーダルペイロード（テキスト、画像、音声）処理の完全ガイド

#### 統合ガイド
- `steering/agentcore-memory-integration.md`: Memory リソースの Strands エージェントとの統合ガイド

## クイックスタート

<!-- markdownlint-disable MD033 -->
<img height="500" src="20251214.png" alt="AgentCore 用 Kiro power の使用を開始する" style="margin-bottom: 1.6rem;" />
<!-- markdownlint-enable MD033 -->
