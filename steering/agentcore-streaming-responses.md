---
inclusion: manual
---

# AgentCore Runtime ストリーミングレスポンス - 完全ガイド

このステアリングファイルは、Amazon Bedrock AgentCore Runtime でのストリーミングレスポンス機能について詳細に説明します。リアルタイムでの応答配信、WebSocket統合、進捗表示などの実装方法を包括的にカバーします。

## 概要

ストリーミングレスポンスは、長時間の処理や大量のデータを段階的に配信する技術です。ユーザーエクスペリエンスの向上と応答性の改善を実現し、エンタープライズグレードのAIエージェントには不可欠な機能です。

### ストリーミングの利点

- **リアルタイム性**: 処理完了を待たずに結果を配信
- **ユーザーエクスペリエンス**: 進捗表示による安心感
- **メモリ効率**: 大量データの段階的処理
- **応答性**: 長時間処理でのタイムアウト回避
- **インタラクティブ性**: リアルタイムな対話体験

## ストリーミングレスポンスの実装

### 1. 基本的なストリーミングエージェント

```python
from bedrock_agentcore import BedrockAgentCoreApp
from bedrock_agentcore.streaming import StreamingResponse
import asyncio
import json
import time
import boto3

class StreamingAgent:
    """ストリーミング対応エージェント"""
    
    def __init__(self):
        self.bedrock_client = boto3.client('bedrock-runtime')
    
    async def process_streaming_request(self, request: dict) -> StreamingResponse:
        """ストリーミングリクエストの処理"""
        
        prompt = request.get("prompt", "")
        stream_type = request.get("stream_type", "text")
        
        if stream_type == "text":
            return await self._stream_text_response(prompt)
        elif stream_type == "analysis":
            return await self._stream_analysis_response(prompt)
        elif stream_type == "generation":
            return await self._stream_generation_response(prompt)
        elif stream_type == "conversation":
            return await self._stream_conversation_response(prompt, request)
        else:
            raise ValueError(f"Unsupported stream type: {stream_type}")
    
    async def _stream_text_response(self, prompt: str) -> StreamingResponse:
        """テキストストリーミング"""
        
        async def text_generator():
            """テキスト生成ジェネレーター"""
            
            try:
                # Bedrock でストリーミング呼び出し
                response = self.bedrock_client.invoke_model_with_response_stream(
                    modelId="anthropic.claude-3-sonnet-20240229-v1:0",
                    body=json.dumps({
                        "anthropic_version": "bedrock-2023-05-31",
                        "max_tokens": 1000,
                        "messages": [{"role": "user", "content": prompt}],
                        "stream": True
                    })
                )
                
                # ストリーミング開始の通知
                yield {
                    "type": "stream_start",
                    "timestamp": time.time(),
                    "model": "claude-3-sonnet"
                }
                
                accumulated_text = ""
                
                # ストリーミングレスポンスの処理
                for event in response['body']:
                    if 'chunk' in event:
                        chunk = json.loads(event['chunk']['bytes'])
                        
                        if chunk['type'] == 'content_block_start':
                            yield {
                                "type": "content_start",
                                "timestamp": time.time()
                            }
                        
                        elif chunk['type'] == 'content_block_delta':
                            text_chunk = chunk['delta'].get('text', '')
                            if text_chunk:
                                accumulated_text += text_chunk
                                yield {
                                    "type": "text_chunk",
                                    "content": text_chunk,
                                    "accumulated_content": accumulated_text,
                                    "chunk_length": len(text_chunk),
                                    "total_length": len(accumulated_text),
                                    "timestamp": time.time()
                                }
                        
                        elif chunk['type'] == 'content_block_stop':
                            yield {
                                "type": "content_end",
                                "final_content": accumulated_text,
                                "total_length": len(accumulated_text),
                                "timestamp": time.time()
                            }
                        
                        elif chunk['type'] == 'message_stop':
                            yield {
                                "type": "stream_end",
                                "final_content": accumulated_text,
                                "total_tokens": chunk.get('usage', {}).get('output_tokens', 0),
                                "timestamp": time.time()
                            }
                            break
            
            except Exception as e:
                yield {
                    "type": "error",
                    "error": str(e),
                    "error_type": type(e).__name__,
                    "timestamp": time.time()
                }
        
        return StreamingResponse(text_generator(), media_type="application/json")
    
    async def _stream_analysis_response(self, prompt: str) -> StreamingResponse:
        """分析結果のストリーミング"""
        
        async def analysis_generator():
            """分析ストリーミングジェネレーター"""
            
            # 分析ステップの定義
            analysis_steps = [
                {
                    "step": "data_collection", 
                    "description": "データ収集中...",
                    "estimated_time": 3
                },
                {
                    "step": "preprocessing", 
                    "description": "前処理実行中...",
                    "estimated_time": 2
                },
                {
                    "step": "analysis", 
                    "description": "分析実行中...",
                    "estimated_time": 5
                },
                {
                    "step": "visualization", 
                    "description": "可視化生成中...",
                    "estimated_time": 2
                },
                {
                    "step": "summary", 
                    "description": "サマリー作成中...",
                    "estimated_time": 2
                }
            ]
            
            total_steps = len(analysis_steps)
            total_estimated_time = sum(step["estimated_time"] for step in analysis_steps)
            
            # 分析開始の通知
            yield {
                "type": "analysis_start",
                "total_steps": total_steps,
                "estimated_total_time": total_estimated_time,
                "steps": [{"name": step["step"], "description": step["description"]} for step in analysis_steps],
                "timestamp": time.time()
            }
            
            # 各ステップの実行
            for i, step in enumerate(analysis_steps):
                step_start_time = time.time()
                
                # ステップ開始の通知
                yield {
                    "type": "step_start",
                    "step": step["step"],
                    "step_number": i + 1,
                    "total_steps": total_steps,
                    "description": step["description"],
                    "estimated_time": step["estimated_time"],
                    "progress": i / total_steps * 100,
                    "timestamp": step_start_time
                }
                
                # ステップ内の進捗更新
                for progress in range(0, 101, 20):  # 20%刻みで進捗更新
                    await asyncio.sleep(step["estimated_time"] / 5)  # 実際の処理をシミュレート
                    
                    yield {
                        "type": "step_progress",
                        "step": step["step"],
                        "step_progress": progress,
                        "overall_progress": ((i + progress / 100) / total_steps) * 100,
                        "timestamp": time.time()
                    }
                
                # 実際の分析処理
                step_result = await self._execute_analysis_step(step["step"], prompt)
                step_end_time = time.time()
                
                # ステップ完了の通知
                yield {
                    "type": "step_complete",
                    "step": step["step"],
                    "step_number": i + 1,
                    "result": step_result,
                    "execution_time": step_end_time - step_start_time,
                    "progress": (i + 1) / total_steps * 100,
                    "timestamp": step_end_time
                }
            
            # 最終結果の生成
            final_result = await self._generate_final_analysis(prompt, analysis_steps)
            
            # 分析完了の通知
            yield {
                "type": "analysis_complete",
                "final_result": final_result,
                "total_execution_time": time.time() - analysis_steps[0].get("start_time", time.time()),
                "completed_steps": total_steps,
                "timestamp": time.time()
            }
        
        return StreamingResponse(analysis_generator(), media_type="application/json")
    
    async def _stream_generation_response(self, prompt: str) -> StreamingResponse:
        """コンテンツ生成のストリーミング"""
        
        async def generation_generator():
            """生成ストリーミングジェネレーター"""
            
            # 生成タスクの分割
            generation_tasks = [
                {"task": "outline", "name": "アウトライン作成", "weight": 0.1},
                {"task": "introduction", "name": "導入部分の生成", "weight": 0.2},
                {"task": "body", "name": "本文の生成", "weight": 0.5},
                {"task": "conclusion", "name": "結論の生成", "weight": 0.15},
                {"task": "refinement", "name": "最終調整", "weight": 0.05}
            ]
            
            generated_content = {}
            total_weight = sum(task["weight"] for task in generation_tasks)
            
            # 生成開始の通知
            yield {
                "type": "generation_start",
                "total_tasks": len(generation_tasks),
                "tasks": [{"name": task["name"], "weight": task["weight"]} for task in generation_tasks],
                "timestamp": time.time()
            }
            
            cumulative_progress = 0
            
            for i, task in enumerate(generation_tasks):
                task_start_time = time.time()
                
                # タスク開始の通知
                yield {
                    "type": "task_start",
                    "task": task["task"],
                    "task_name": task["name"],
                    "task_number": i + 1,
                    "total_tasks": len(generation_tasks),
                    "progress": cumulative_progress,
                    "timestamp": task_start_time
                }
                
                # 各タスクの実行
                content_chunk = await self._generate_content_chunk(task["task"], prompt, generated_content)
                generated_content[task["task"]] = content_chunk
                
                # タスク完了の通知
                cumulative_progress += (task["weight"] / total_weight) * 100
                task_end_time = time.time()
                
                yield {
                    "type": "task_complete",
                    "task": task["task"],
                    "task_name": task["name"],
                    "content": content_chunk,
                    "progress": cumulative_progress,
                    "execution_time": task_end_time - task_start_time,
                    "timestamp": task_end_time
                }
                
                # 中間結果の提供
                if task["task"] in ["outline", "introduction", "body"]:
                    partial_content = self._combine_content(generated_content)
                    yield {
                        "type": "partial_result",
                        "partial_content": partial_content,
                        "completed_tasks": list(generated_content.keys()),
                        "progress": cumulative_progress,
                        "timestamp": time.time()
                    }
            
            # 最終コンテンツの組み立て
            final_content = self._combine_content(generated_content)
            
            # 生成完了の通知
            yield {
                "type": "generation_complete",
                "final_content": final_content,
                "word_count": len(final_content.split()),
                "character_count": len(final_content),
                "completed_tasks": len(generation_tasks),
                "timestamp": time.time()
            }
        
        return StreamingResponse(generation_generator(), media_type="application/json")
    
    async def _stream_conversation_response(self, prompt: str, request: dict) -> StreamingResponse:
        """会話型ストリーミング"""
        
        async def conversation_generator():
            """会話ストリーミングジェネレーター"""
            
            conversation_history = request.get("conversation_history", [])
            user_id = request.get("user_id", "anonymous")
            
            # 会話開始の通知
            yield {
                "type": "conversation_start",
                "user_id": user_id,
                "history_length": len(conversation_history),
                "timestamp": time.time()
            }
            
            # コンテキスト分析
            yield {
                "type": "context_analysis",
                "status": "analyzing_context",
                "timestamp": time.time()
            }
            
            context_analysis = await self._analyze_conversation_context(conversation_history, prompt)
            
            yield {
                "type": "context_complete",
                "analysis": context_analysis,
                "timestamp": time.time()
            }
            
            # 応答生成の準備
            enhanced_prompt = await self._build_conversation_prompt(prompt, conversation_history, context_analysis)
            
            yield {
                "type": "response_preparation",
                "enhanced_prompt_length": len(enhanced_prompt),
                "timestamp": time.time()
            }
            
            # ストリーミング応答生成
            async for chunk in self._stream_bedrock_response(enhanced_prompt):
                yield chunk
            
            # 会話終了の通知
            yield {
                "type": "conversation_end",
                "timestamp": time.time()
            }
        
        return StreamingResponse(conversation_generator(), media_type="application/json")
    
    async def _execute_analysis_step(self, step: str, prompt: str) -> dict:
        """分析ステップの実行"""
        
        # 実際の分析ロジック（簡易実装）
        results = {
            "data_collection": {
                "records_found": 1000,
                "data_sources": ["database", "api", "files"],
                "quality_score": 0.95
            },
            "preprocessing": {
                "cleaned_records": 950,
                "features_extracted": 15,
                "normalization_applied": True
            },
            "analysis": {
                "patterns_identified": 5,
                "correlations_found": 12,
                "statistical_significance": 0.01
            },
            "visualization": {
                "charts_generated": 3,
                "tables_created": 2,
                "interactive_elements": 1
            },
            "summary": {
                "key_insights": 8,
                "recommendations": 4,
                "confidence_level": 0.92
            }
        }
        
        return results.get(step, {"status": "completed"})
    
    async def _generate_final_analysis(self, prompt: str, steps: list) -> dict:
        """最終分析結果の生成"""
        
        return {
            "summary": f"{prompt}の分析が完了しました。",
            "total_steps": len(steps),
            "key_findings": [
                "データ品質は良好です",
                "明確なパターンが識別されました",
                "統計的に有意な結果が得られました"
            ],
            "recommendations": [
                "定期的なデータ更新を推奨",
                "追加の分析軸を検討",
                "結果の継続的な監視"
            ]
        }
    
    async def _generate_content_chunk(self, task: str, prompt: str, existing_content: dict) -> str:
        """コンテンツチャンクの生成"""
        
        # 実際のコンテンツ生成（Bedrockを使用）
        task_prompts = {
            "outline": f"{prompt}について、詳細なアウトラインを作成してください。",
            "introduction": f"{prompt}について、魅力的な導入部分を書いてください。",
            "body": f"{prompt}について、詳細な本文を書いてください。アウトライン: {existing_content.get('outline', '')}",
            "conclusion": f"以下の内容を基に、説得力のある結論を書いてください。\n{existing_content.get('body', '')}",
            "refinement": "全体の文章を見直し、読みやすさと一貫性を向上させてください。"
        }
        
        task_prompt = task_prompts.get(task, f"{task}の内容を生成してください。")
        
        try:
            response = self.bedrock_client.invoke_model(
                modelId="anthropic.claude-3-sonnet-20240229-v1:0",
                body=json.dumps({
                    "anthropic_version": "bedrock-2023-05-31",
                    "max_tokens": 500,
                    "messages": [{"role": "user", "content": task_prompt}]
                })
            )
            
            result = json.loads(response['body'].read())
            return result['content'][0]['text']
        
        except Exception as e:
            return f"{task}の生成中にエラーが発生しました: {str(e)}"
    
    def _combine_content(self, content_dict: dict) -> str:
        """コンテンツの結合"""
        
        order = ["outline", "introduction", "body", "conclusion", "refinement"]
        combined = []
        
        for section in order:
            if section in content_dict:
                combined.append(content_dict[section])
        
        return "\n\n".join(combined)
    
    async def _analyze_conversation_context(self, history: list, current_prompt: str) -> dict:
        """会話コンテキストの分析"""
        
        return {
            "conversation_length": len(history),
            "topics_discussed": ["AI", "技術", "開発"],  # 簡易実装
            "user_intent": "情報収集",
            "sentiment": "neutral",
            "context_continuity": True
        }
    
    async def _build_conversation_prompt(self, prompt: str, history: list, context: dict) -> str:
        """会話用プロンプトの構築"""
        
        enhanced_prompt = "会話履歴:\n"
        
        # 最新の会話履歴を追加
        for msg in history[-5:]:  # 最新5件
            role = msg.get("role", "user")
            content = msg.get("content", "")
            enhanced_prompt += f"{role}: {content}\n"
        
        enhanced_prompt += f"\n現在のユーザー入力: {prompt}\n"
        enhanced_prompt += f"コンテキスト: {json.dumps(context, ensure_ascii=False)}\n"
        
        return enhanced_prompt
    
    async def _stream_bedrock_response(self, prompt: str):
        """Bedrock レスポンスのストリーミング"""
        
        try:
            response = self.bedrock_client.invoke_model_with_response_stream(
                modelId="anthropic.claude-3-sonnet-20240229-v1:0",
                body=json.dumps({
                    "anthropic_version": "bedrock-2023-05-31",
                    "max_tokens": 1000,
                    "messages": [{"role": "user", "content": prompt}],
                    "stream": True
                })
            )
            
            for event in response['body']:
                if 'chunk' in event:
                    chunk = json.loads(event['chunk']['bytes'])
                    
                    if chunk['type'] == 'content_block_delta':
                        text_chunk = chunk['delta'].get('text', '')
                        if text_chunk:
                            yield {
                                "type": "response_chunk",
                                "content": text_chunk,
                                "timestamp": time.time()
                            }
        
        except Exception as e:
            yield {
                "type": "response_error",
                "error": str(e),
                "timestamp": time.time()
            }
```

### 2. WebSocket ストリーミング

```python
from bedrock_agentcore.websocket import WebSocketManager
import asyncio
import json

class WebSocketStreamingAgent:
    """WebSocket ストリーミングエージェント"""
    
    def __init__(self):
        self.websocket_manager = WebSocketManager()
        self.active_connections = {}
        self.streaming_sessions = {}
    
    async def handle_websocket_connection(self, connection_id: str, event: dict):
        """WebSocket 接続の処理"""
        
        route_key = event.get('requestContext', {}).get('routeKey')
        
        if route_key == '$connect':
            await self._handle_connect(connection_id, event)
        elif route_key == '$disconnect':
            await self._handle_disconnect(connection_id)
        elif route_key == 'stream':
            await self._handle_stream_request(connection_id, event)
        elif route_key == 'control':
            await self._handle_control_request(connection_id, event)
    
    async def _handle_connect(self, connection_id: str, event: dict):
        """接続処理"""
        
        query_params = event.get('queryStringParameters', {}) or {}
        
        self.active_connections[connection_id] = {
            "connected_at": time.time(),
            "last_activity": time.time(),
            "user_id": query_params.get("user_id", "anonymous"),
            "client_info": {
                "user_agent": event.get('headers', {}).get('User-Agent', ''),
                "source_ip": event.get('requestContext', {}).get('identity', {}).get('sourceIp', '')
            }
        }
        
        await self.websocket_manager.send_message(
            connection_id,
            {
                "type": "connection_established",
                "connection_id": connection_id,
                "server_time": time.time(),
                "supported_features": [
                    "text_streaming",
                    "analysis_streaming", 
                    "generation_streaming",
                    "conversation_streaming",
                    "progress_tracking",
                    "stream_control"
                ]
            }
        )
    
    async def _handle_disconnect(self, connection_id: str):
        """切断処理"""
        
        # アクティブなストリーミングセッションを停止
        if connection_id in self.streaming_sessions:
            session = self.streaming_sessions[connection_id]
            session["cancelled"] = True
            del self.streaming_sessions[connection_id]
        
        # 接続情報を削除
        if connection_id in self.active_connections:
            del self.active_connections[connection_id]
    
    async def _handle_stream_request(self, connection_id: str, event: dict):
        """ストリーミングリクエストの処理"""
        
        try:
            body = json.loads(event.get('body', '{}'))
            request_id = body.get('request_id', f"req_{int(time.time())}")
            
            # ストリーミングセッションの作成
            self.streaming_sessions[connection_id] = {
                "request_id": request_id,
                "started_at": time.time(),
                "cancelled": False
            }
            
            # ストリーミング開始の通知
            await self.websocket_manager.send_message(
                connection_id,
                {
                    "type": "stream_start",
                    "request_id": request_id,
                    "timestamp": time.time()
                }
            )
            
            # ストリーミング処理
            streaming_agent = StreamingAgent()
            streaming_response = await streaming_agent.process_streaming_request(body)
            
            # ストリーミングデータの送信
            async for chunk in streaming_response.body_iterator:
                # 接続状態とキャンセル状態の確認
                if (connection_id not in self.active_connections or 
                    self.streaming_sessions.get(connection_id, {}).get("cancelled", False)):
                    break
                
                # チャンクデータの送信
                chunk_data = json.loads(chunk) if isinstance(chunk, str) else chunk
                chunk_data["request_id"] = request_id
                
                await self.websocket_manager.send_message(connection_id, chunk_data)
                
                # 送信間隔の調整（バックプレッシャー制御）
                await asyncio.sleep(0.01)
            
            # ストリーミング終了の通知
            if connection_id in self.active_connections:
                await self.websocket_manager.send_message(
                    connection_id,
                    {
                        "type": "stream_end",
                        "request_id": request_id,
                        "timestamp": time.time()
                    }
                )
            
            # セッション情報のクリーンアップ
            if connection_id in self.streaming_sessions:
                del self.streaming_sessions[connection_id]
        
        except Exception as e:
            await self.websocket_manager.send_message(
                connection_id,
                {
                    "type": "error",
                    "error": str(e),
                    "error_type": type(e).__name__,
                    "timestamp": time.time()
                }
            )
    
    async def _handle_control_request(self, connection_id: str, event: dict):
        """制御リクエストの処理"""
        
        try:
            body = json.loads(event.get('body', '{}'))
            control_type = body.get('control_type')
            
            if control_type == 'pause':
                await self._pause_stream(connection_id, body)
            elif control_type == 'resume':
                await self._resume_stream(connection_id, body)
            elif control_type == 'cancel':
                await self._cancel_stream(connection_id, body)
            elif control_type == 'status':
                await self._get_stream_status(connection_id, body)
        
        except Exception as e:
            await self.websocket_manager.send_message(
                connection_id,
                {
                    "type": "control_error",
                    "error": str(e),
                    "timestamp": time.time()
                }
            )
    
    async def _pause_stream(self, connection_id: str, body: dict):
        """ストリーミングの一時停止"""
        
        if connection_id in self.streaming_sessions:
            self.streaming_sessions[connection_id]["paused"] = True
            
            await self.websocket_manager.send_message(
                connection_id,
                {
                    "type": "stream_paused",
                    "request_id": body.get("request_id"),
                    "timestamp": time.time()
                }
            )
    
    async def _resume_stream(self, connection_id: str, body: dict):
        """ストリーミングの再開"""
        
        if connection_id in self.streaming_sessions:
            self.streaming_sessions[connection_id]["paused"] = False
            
            await self.websocket_manager.send_message(
                connection_id,
                {
                    "type": "stream_resumed",
                    "request_id": body.get("request_id"),
                    "timestamp": time.time()
                }
            )
    
    async def _cancel_stream(self, connection_id: str, body: dict):
        """ストリーミングのキャンセル"""
        
        if connection_id in self.streaming_sessions:
            self.streaming_sessions[connection_id]["cancelled"] = True
            
            await self.websocket_manager.send_message(
                connection_id,
                {
                    "type": "stream_cancelled",
                    "request_id": body.get("request_id"),
                    "timestamp": time.time()
                }
            )
    
    async def _get_stream_status(self, connection_id: str, body: dict):
        """ストリーミング状態の取得"""
        
        session = self.streaming_sessions.get(connection_id, {})
        
        await self.websocket_manager.send_message(
            connection_id,
            {
                "type": "stream_status",
                "request_id": body.get("request_id"),
                "status": {
                    "active": connection_id in self.streaming_sessions,
                    "paused": session.get("paused", False),
                    "cancelled": session.get("cancelled", False),
                    "duration": time.time() - session.get("started_at", time.time())
                },
                "timestamp": time.time()
            }
        )
```

### 3. ストリーミング最適化とベストプラクティス

```python
class OptimizedStreamingAgent:
    """最適化されたストリーミングエージェント"""
    
    def __init__(self):
        self.bedrock_client = boto3.client('bedrock-runtime')
        self.chunk_buffer = []
        self.buffer_size = 10
        self.min_chunk_interval = 0.1  # 100ms
        self.last_chunk_time = 0
    
    async def optimized_text_streaming(self, prompt: str, options: dict = None):
        """最適化されたテキストストリーミング"""
        
        options = options or {}
        chunk_size = options.get("chunk_size", 50)  # 文字数
        buffer_enabled = options.get("buffer_enabled", True)
        adaptive_timing = options.get("adaptive_timing", True)
        
        async def optimized_generator():
            try:
                response = self.bedrock_client.invoke_model_with_response_stream(
                    modelId="anthropic.claude-3-sonnet-20240229-v1:0",
                    body=json.dumps({
                        "anthropic_version": "bedrock-2023-05-31",
                        "max_tokens": 1000,
                        "messages": [{"role": "user", "content": prompt}],
                        "stream": True
                    })
                )
                
                accumulated_text = ""
                buffer = ""
                
                for event in response['body']:
                    if 'chunk' in event:
                        chunk = json.loads(event['chunk']['bytes'])
                        
                        if chunk['type'] == 'content_block_delta':
                            text_chunk = chunk['delta'].get('text', '')
                            if text_chunk:
                                buffer += text_chunk
                                accumulated_text += text_chunk
                                
                                # チャンクサイズまたは時間間隔での送信
                                current_time = time.time()
                                
                                should_send = (
                                    len(buffer) >= chunk_size or
                                    (adaptive_timing and current_time - self.last_chunk_time >= self.min_chunk_interval) or
                                    not buffer_enabled
                                )
                                
                                if should_send and buffer:
                                    yield {
                                        "type": "optimized_chunk",
                                        "content": buffer,
                                        "accumulated_content": accumulated_text,
                                        "chunk_length": len(buffer),
                                        "total_length": len(accumulated_text),
                                        "timestamp": current_time,
                                        "optimization": {
                                            "buffer_size": len(buffer),
                                            "time_since_last": current_time - self.last_chunk_time
                                        }
                                    }
                                    
                                    buffer = ""
                                    self.last_chunk_time = current_time
                        
                        elif chunk['type'] == 'message_stop':
                            # 残りのバッファを送信
                            if buffer:
                                yield {
                                    "type": "final_chunk",
                                    "content": buffer,
                                    "accumulated_content": accumulated_text,
                                    "timestamp": time.time()
                                }
                            
                            yield {
                                "type": "stream_complete",
                                "final_content": accumulated_text,
                                "total_length": len(accumulated_text),
                                "total_tokens": chunk.get('usage', {}).get('output_tokens', 0),
                                "timestamp": time.time()
                            }
                            break
            
            except Exception as e:
                yield {
                    "type": "stream_error",
                    "error": str(e),
                    "timestamp": time.time()
                }
        
        return optimized_generator()
    
    async def adaptive_streaming(self, prompt: str, client_info: dict = None):
        """クライアント適応型ストリーミング"""
        
        client_info = client_info or {}
        connection_speed = client_info.get("connection_speed", "medium")
        device_type = client_info.get("device_type", "desktop")
        
        # 接続速度とデバイスに基づく最適化
        if connection_speed == "slow" or device_type == "mobile":
            chunk_size = 100  # 大きなチャンク
            buffer_time = 0.5  # 長いバッファ時間
        elif connection_speed == "fast":
            chunk_size = 20   # 小さなチャンク
            buffer_time = 0.05  # 短いバッファ時間
        else:
            chunk_size = 50   # 標準チャンク
            buffer_time = 0.1  # 標準バッファ時間
        
        async def adaptive_generator():
            yield {
                "type": "adaptation_info",
                "settings": {
                    "chunk_size": chunk_size,
                    "buffer_time": buffer_time,
                    "connection_speed": connection_speed,
                    "device_type": device_type
                },
                "timestamp": time.time()
            }
            
            async for chunk in self.optimized_text_streaming(
                prompt, 
                {
                    "chunk_size": chunk_size,
                    "min_chunk_interval": buffer_time
                }
            ):
                yield chunk
        
        return adaptive_generator()
```

## AgentCore 統合

### ストリーミング対応ハンドラー

```python
async def streaming_handler(event, context):
    """ストリーミング対応 AgentCore ハンドラー"""
    
    request_data = json.loads(event.get('body', '{}'))
    
    # リクエストタイプの判定
    if event.get('requestContext', {}).get('eventType') == 'MESSAGE':
        # WebSocket リクエスト
        connection_id = event.get('requestContext', {}).get('connectionId')
        websocket_agent = WebSocketStreamingAgent()
        await websocket_agent.handle_websocket_connection(connection_id, event)
        
        return {"statusCode": 200}
    
    else:
        # HTTP ストリーミングリクエスト
        agent = StreamingAgent()
        
        try:
            streaming_response = await agent.process_streaming_request(request_data)
            
            return {
                "statusCode": 200,
                "headers": {
                    "Content-Type": "application/json",
                    "Cache-Control": "no-cache",
                    "Connection": "keep-alive",
                    "Access-Control-Allow-Origin": "*"
                },
                "body": streaming_response
            }
        
        except Exception as e:
            return {
                "statusCode": 500,
                "body": json.dumps({
                    "error": str(e),
                    "error_type": type(e).__name__
                })
            }
```

## 設定とデプロイ

### AgentCore 設定ファイル

**.bedrock_agentcore.yaml**
```yaml
# ストリーミング対応 AgentCore 設定
runtime:
  entrypoint: "src/streaming_agent.py"
  handler: "streaming_handler"
  
  # ランタイム環境
  python_version: "3.11"
  timeout: 900  # 15分（長時間ストリーミング対応）
  memory_size: 2048
  
  # 環境変数
  environment:
    LOG_LEVEL: "INFO"
    STREAMING_ENABLED: "true"
    WEBSOCKET_ENABLED: "true"

# ストリーミング設定
streaming:
  enabled: true
  
  # HTTP ストリーミング
  http:
    chunk_size: 1024
    timeout: 300
    keep_alive: true
  
  # WebSocket ストリーミング
  websocket:
    enabled: true
    idle_timeout: 600
    max_connections: 1000
    message_size_limit: 32768
  
  # 最適化設定
  optimization:
    buffer_enabled: true
    adaptive_timing: true
    compression_enabled: true

# 監視設定
monitoring:
  metrics_enabled: true
  custom_metrics:
    - name: "StreamingRequests"
      unit: "Count"
    - name: "StreamingDuration"
      unit: "Seconds"
    - name: "WebSocketConnections"
      unit: "Count"
    - name: "ChunksSent"
      unit: "Count"
```

## ベストプラクティス

### 1. パフォーマンス最適化
- 適切なチャンクサイズの設定
- バッファリングによる効率化
- 接続状態の適切な管理

### 2. エラーハンドリング
- 接続切断の検出
- 再接続ロジックの実装
- グレースフルな停止処理

### 3. リソース管理
- メモリ使用量の監視
- 接続数の制限
- タイムアウト設定

### 4. ユーザーエクスペリエンス
- 進捗表示の実装
- キャンセル機能の提供
- 適応的な配信速度

## トラブルシューティング

### よくある問題と解決策

#### 1. ストリーミング接続エラー
**問題**: `Connection lost during streaming`
**解決策**: 
- 接続タイムアウトの調整
- 再接続ロジックの実装
- ハートビート機能の追加

#### 2. パフォーマンス問題
**問題**: `Streaming lag or delays`
**解決策**:
- チャンクサイズの最適化
- バッファリング設定の調整
- ネットワーク最適化

#### 3. メモリ使用量問題
**問題**: `Memory usage too high`
**解決策**:
- ストリーミングバッファの制限
- ガベージコレクションの最適化
- メモリ効率的な実装

これらの実装により、AgentCore Runtime で高品質なストリーミングレスポンス機能を提供できます。