---
inclusion: manual
---

# AgentCore Runtime マルチモーダルペイロード処理 - 完全ガイド

このステアリングファイルは、Amazon Bedrock AgentCore Runtime でのマルチモーダルペイロード処理について詳細に説明します。テキスト、画像、音声の統合処理、大容量データの効率的な処理などの実装方法を包括的にカバーします。

## 概要

マルチモーダルペイロード処理は、現代のAIエージェントにおいて、テキスト以外のデータ形式（画像、音声、動画など）を統合的に処理する重要な機能です。AgentCore Runtime では、これらの多様なデータ形式を効率的に処理し、統合的な分析と応答を提供します。

### 主要な機能

- **マルチモーダル処理**: テキスト、画像、音声の統合処理
- **大容量データ処理**: 効率的な大データ処理とストリーミング
- **S3統合**: 大容量ファイルの自動アップロード・ダウンロード
- **形式変換**: 各種データ形式の相互変換
- **メタデータ抽出**: ファイル情報とメタデータの自動抽出

## マルチモーダルデータ処理

### 1. 基本的なマルチモーダルエージェント

```python
import boto3
import base64
import io
import json
import time
import asyncio
from PIL import Image
from typing import Dict, List, Any, Union, Optional
import mimetypes
import hashlib

class MultimodalAgent:
    """マルチモーダル対応エージェント"""
    
    def __init__(self):
        self.bedrock_client = boto3.client('bedrock-runtime')
        self.s3_client = boto3.client('s3')
        self.transcribe_client = boto3.client('transcribe')
        self.rekognition_client = boto3.client('rekognition')
        
        # 設定
        self.max_memory_size = 10 * 1024 * 1024  # 10MB
        self.s3_bucket = 'agentcore-multimodal-data'
        self.supported_image_formats = ['jpeg', 'jpg', 'png', 'gif', 'webp', 'bmp']
        self.supported_audio_formats = ['wav', 'mp3', 'm4a', 'flac', 'ogg']
        self.supported_video_formats = ['mp4', 'avi', 'mov', 'wmv']
    
    async def process_multimodal_request(self, request: dict) -> dict:
        """マルチモーダルリクエストの処理"""
        
        content_type = request.get('content_type', 'text')
        
        if content_type == 'text':
            return await self._process_text(request)
        elif content_type == 'image':
            return await self._process_image(request)
        elif content_type == 'audio':
            return await self._process_audio(request)
        elif content_type == 'video':
            return await self._process_video(request)
        elif content_type == 'mixed':
            return await self._process_mixed_content(request)
        elif content_type == 'document':
            return await self._process_document(request)
        else:
            raise ValueError(f"Unsupported content type: {content_type}")
```

### 2. 大容量ペイロード処理

```python
class LargePayloadHandler:
    """大容量ペイロード処理システム"""
    
    def __init__(self):
        self.s3_client = boto3.client('s3')
        self.chunk_size = 1024 * 1024  # 1MB チャンク
        self.max_memory_size = 50 * 1024 * 1024  # 50MB メモリ制限
        self.temp_bucket = 'agentcore-temp-processing'
    
    async def process_large_payload(self, request: dict) -> dict:
        """大容量ペイロードの処理"""
        
        payload_size = request.get('payload_size', 0)
        processing_mode = request.get('processing_mode', 'auto')
        
        # 処理モードの決定
        if processing_mode == 'auto':
            if payload_size > self.max_memory_size:
                processing_mode = 'streaming'
            else:
                processing_mode = 'memory'
        
        if processing_mode == 'streaming':
            return await self._process_streaming_payload(request)
        else:
            return await self._process_memory_payload(request)
```

## AgentCore 統合

### マルチモーダル対応ハンドラー

```python
from bedrock_agentcore import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

@app.entrypoint
async def multimodal_handler(payload):
    """マルチモーダル対応 AgentCore ハンドラー"""
    
    # リクエストサイズの確認
    request_size = len(str(payload))
    
    if request_size > 50 * 1024 * 1024:  # 50MB 以上
        # 大容量ペイロード処理
        handler = LargePayloadHandler()
        result = await handler.process_large_payload(payload)
    else:
        # 通常のマルチモーダル処理
        agent = MultimodalAgent()
        result = await agent.process_multimodal_request(payload)
    
    return result
```

## 設定とデプロイ

### AgentCore 設定ファイル

**.bedrock_agentcore.yaml**
```yaml
# マルチモーダル対応 AgentCore 設定
runtime:
  entrypoint: "src/multimodal_agent.py"
  handler: "multimodal_handler"
  
  # ランタイム環境
  python_version: "3.11"
  timeout: 900  # 15分（大容量処理対応）
  memory_size: 3008  # 最大メモリ
  
  # 環境変数
  environment:
    LOG_LEVEL: "INFO"
    S3_BUCKET: "agentcore-multimodal-data"
    TEMP_BUCKET: "agentcore-temp-processing"
    MAX_MEMORY_SIZE: "52428800"  # 50MB

# マルチモーダル設定
multimodal:
  enabled: true
  
  # サポートする形式
  supported_formats:
    image: ["jpeg", "jpg", "png", "gif", "webp", "bmp", "tiff"]
    audio: ["wav", "mp3", "m4a", "flac", "ogg", "aac"]
    video: ["mp4", "avi", "mov", "wmv", "mkv", "webm"]
    document: ["pdf", "docx", "txt", "rtf", "odt"]
  
  # サイズ制限
  size_limits:
    max_image_size: 10485760    # 10MB
    max_audio_size: 52428800    # 50MB
    max_video_size: 104857600   # 100MB
    max_document_size: 52428800 # 50MB
  
  # 処理オプション
  processing:
    enable_rekognition: true
    enable_transcribe: true
    enable_textract: true
    parallel_processing: true
    chunk_processing: true

# S3 設定
s3:
  multimodal_bucket: "agentcore-multimodal-data"
  temp_bucket: "agentcore-temp-processing"
  encryption: "AES256"
  lifecycle_days: 30

# 監視設定
monitoring:
  enable_metrics: true
  enable_tracing: true
  log_level: "INFO"
```

### デプロイ手順

```bash
# 1. 依存関係のインストール
uv add bedrock-agentcore-starter-toolkit
uv add pillow boto3 aiohttp

# 2. 設定の確認
uv run agentcore configure --entrypoint src/multimodal_agent.py --region us-east-1

# 3. デプロイ
uv run agentcore launch --region us-east-1

# 4. テスト
uv run agentcore invoke '{
  "content_type": "image",
  "image_data": "base64_encoded_image_data",
  "prompt": "この画像について説明してください"
}' --region us-east-1
```

## 使用例

### 1. 画像処理の例

```python
# 画像処理リクエスト
image_request = {
    "content_type": "image",
    "image_data": "base64_encoded_image_data",
    "image_format": "jpeg",
    "prompt": "この画像に写っているものを詳しく説明してください",
    "options": {
        "advanced_analysis": True,
        "use_rekognition": True,
        "content_moderation": True
    }
}

result = await agent.process_multimodal_request(image_request)
```

### 2. 音声処理の例

```python
# 音声処理リクエスト
audio_request = {
    "content_type": "audio",
    "audio_data": "base64_encoded_audio_data",
    "audio_format": "wav",
    "options": {
        "language_code": "ja-JP",
        "speaker_identification": True,
        "max_speakers": 2
    }
}

result = await agent.process_multimodal_request(audio_request)
```

### 3. 混合コンテンツ処理の例

```python
# 混合コンテンツ処理リクエスト
mixed_request = {
    "content_type": "mixed",
    "contents": [
        {
            "type": "text",
            "text": "プレゼンテーションの概要"
        },
        {
            "type": "image",
            "image_data": "base64_encoded_slide_image",
            "image_format": "png"
        },
        {
            "type": "audio",
            "audio_data": "base64_encoded_narration",
            "audio_format": "mp3"
        }
    ],
    "options": {
        "advanced_integration": True
    }
}

result = await agent.process_multimodal_request(mixed_request)
```

## ベストプラクティス

### 1. パフォーマンス最適化

- **並列処理**: 複数のコンテンツを並行処理
- **チャンク処理**: 大容量データのストリーミング処理
- **キャッシュ活用**: 処理結果のキャッシュ
- **メモリ管理**: 適切なメモリ使用量の制御

### 2. セキュリティ

- **データ暗号化**: S3 での暗号化保存
- **アクセス制御**: IAM による細かい権限設定
- **コンテンツモデレーション**: 不適切なコンテンツの検出
- **監査ログ**: 全処理の記録

### 3. エラーハンドリング

- **段階的フォールバック**: 処理失敗時の代替手段
- **リトライ機能**: 一時的な障害への対応
- **詳細なエラー情報**: デバッグ支援
- **ユーザーフレンドリーなメッセージ**: 分かりやすいエラー通知

## トラブルシューティング

### よくある問題と解決策

1. **メモリ不足エラー**
   - チャンク処理の有効化
   - メモリサイズの増加
   - ストリーミング処理への切り替え

2. **タイムアウトエラー**
   - タイムアウト時間の延長
   - 並列処理の活用
   - 処理の分割

3. **形式サポートエラー**
   - サポート形式の確認
   - 形式変換の実装
   - エラーメッセージの改善

4. **S3 アクセスエラー**
   - IAM 権限の確認
   - バケット設定の確認
   - ネットワーク接続の確認

## まとめ

AgentCore Runtime でのマルチモーダルペイロード処理は、現代のAIエージェントに必要不可欠な機能です。適切な実装により、テキスト、画像、音声、動画などの多様なデータ形式を統合的に処理し、ユーザーに価値のある洞察を提供できます。

大容量データの処理、パフォーマンスの最適化、セキュリティの確保など、実装時に考慮すべき点は多岐にわたりますが、このガイドで示したベストプラクティスに従うことで、堅牢で効率的なマルチモーダルエージェントを構築できます。