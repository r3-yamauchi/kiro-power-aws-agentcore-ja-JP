---
inclusion: manual
---

# AgentCore Runtime セッション・コンテキスト管理 - 完全ガイド

このステアリングファイルは、Amazon Bedrock AgentCore Runtime でのセッション管理とランタイムコンテキスト管理について詳細に説明します。状態の永続化、会話の継続性、動的コンテキスト管理などの実装方法を包括的にカバーします。

## 概要

セッション・コンテキスト管理は、エンタープライズグレードのAIエージェントにおいて、ユーザーとの継続的な対話を実現するための重要な機能です。適切な状態管理により、パーソナライズされた体験と効率的なリソース利用を両立できます。

### セッション管理の重要性

- **継続性**: 会話の文脈を保持
- **パーソナライゼーション**: ユーザー固有の体験
- **効率性**: 状態の再利用による高速化
- **スケーラビリティ**: 大量セッションの効率的管理
- **信頼性**: 障害時の状態復旧

## セッション管理の実装

### 1. 高度なセッションマネージャー

```python
from bedrock_agentcore.session import SessionManager
from bedrock_agentcore.context import RuntimeContext
import boto3
import json
from datetime import datetime, timedelta
import uuid
import hashlib

class AdvancedSessionManager:
    """高度なセッション管理システム"""
    
    def __init__(self):
        self.dynamodb = boto3.resource('dynamodb')
        self.session_table = self.dynamodb.Table('agentcore-sessions')
        self.context_table = self.dynamodb.Table('agentcore-contexts')
        self.history_table = self.dynamodb.Table('conversation-history')
        
        # セッション設定
        self.default_ttl_hours = 24
        self.max_sessions_per_user = 10
        self.session_cleanup_interval = 3600  # 1時間
    
    async def create_session(
        self, 
        user_id: str, 
        session_config: dict = None,
        initial_context: dict = None
    ) -> str:
        """セッションの作成"""
        
        session_id = self._generate_session_id(user_id)
        current_time = datetime.utcnow()
        
        # セッション設定のデフォルト値
        config = {
            "max_history_length": 50,
            "auto_summarize": True,
            "summarize_threshold": 20,
            "context_window": 4000,
            "preferred_model": "claude-3-sonnet",
            "temperature": 0.7,
            **(session_config or {})
        }
        
        session_data = {
            'session_id': session_id,
            'user_id': user_id,
            'created_at': current_time.isoformat(),
            'last_activity': current_time.isoformat(),
            'status': 'active',
            'config': config,
            'metadata': {
                'message_count': 0,
                'total_tokens': 0,
                'total_cost': 0.0,
                'last_model_used': None,
                'conversation_topics': [],
                'user_preferences': {},
                'session_quality_score': 1.0
            },
            'ttl': int((current_time + timedelta(hours=self.default_ttl_hours)).timestamp()),
            'version': 1
        }
        
        # 初期コンテキストの設定
        if initial_context:
            session_data['initial_context'] = initial_context
        
        try:
            # ユーザーの既存セッション数をチェック
            await self._enforce_session_limits(user_id)
            
            # セッションの保存
            self.session_table.put_item(
                Item=session_data,
                ConditionExpression='attribute_not_exists(session_id)'
            )
            
            # 初期コンテキストの作成
            if initial_context:
                await self._create_initial_context(session_id, initial_context)
            
            return session_id
        
        except Exception as e:
            raise ValueError(f"Failed to create session: {e}")
    
    async def get_session(self, session_id: str, update_activity: bool = True) -> dict:
        """セッション情報の取得"""
        
        try:
            response = self.session_table.get_item(
                Key={'session_id': session_id}
            )
            
            if 'Item' not in response:
                raise ValueError(f"Session not found: {session_id}")
            
            session = response['Item']
            
            # セッションの有効性チェック
            if session.get('status') != 'active':
                raise ValueError(f"Session is not active: {session_id}")
            
            # TTL チェック
            if session.get('ttl', 0) < int(datetime.utcnow().timestamp()):
                await self._expire_session(session_id)
                raise ValueError(f"Session has expired: {session_id}")
            
            # 最終アクティビティの更新
            if update_activity:
                await self._update_last_activity(session_id)
                session['last_activity'] = datetime.utcnow().isoformat()
            
            return session
        
        except Exception as e:
            raise ValueError(f"Failed to get session: {e}")
    
    async def update_session(self, session_id: str, updates: dict, increment_version: bool = True):
        """セッション情報の更新"""
        
        update_expression_parts = []
        expression_values = {}
        expression_names = {}
        
        # 更新式の構築
        for key, value in updates.items():
            if '.' in key:
                # ネストされたキーの処理
                parts = key.split('.')
                attr_name = f"#{parts[0]}"
                expression_names[attr_name] = parts[0]
                
                if len(parts) == 2:
                    update_key = f"{attr_name}.{parts[1]}"
                else:
                    update_key = key
            else:
                attr_name = f"#{key}"
                expression_names[attr_name] = key
                update_key = attr_name
            
            if isinstance(value, str) and value.startswith('ADD '):
                # 数値の加算
                add_value = value.replace('ADD ', '')
                update_expression_parts.append(f"{update_key} = if_not_exists({update_key}, :zero) + :{key}")
                expression_values[f":{key}"] = float(add_value)
                expression_values[":zero"] = 0
            else:
                update_expression_parts.append(f"{update_key} = :{key}")
                expression_values[f":{key}"] = value
        
        # バージョン管理
        if increment_version:
            update_expression_parts.append("#version = if_not_exists(#version, :zero) + :one")
            expression_names["#version"] = "version"
            expression_values[":one"] = 1
            if ":zero" not in expression_values:
                expression_values[":zero"] = 0
        
        update_expression = "SET " + ", ".join(update_expression_parts)
        
        try:
            self.session_table.update_item(
                Key={'session_id': session_id},
                UpdateExpression=update_expression,
                ExpressionAttributeNames=expression_names,
                ExpressionAttributeValues=expression_values
            )
        except Exception as e:
            raise ValueError(f"Failed to update session: {e}")
    
    async def _generate_session_id(self, user_id: str) -> str:
        """セッションIDの生成"""
        
        timestamp = str(int(datetime.utcnow().timestamp() * 1000))
        random_part = str(uuid.uuid4())[:8]
        user_hash = hashlib.md5(user_id.encode()).hexdigest()[:8]
        
        return f"session_{timestamp}_{user_hash}_{random_part}"
    
    async def _enforce_session_limits(self, user_id: str):
        """セッション数制限の実施"""
        
        # ユーザーのアクティブセッション数を取得
        response = self.session_table.query(
            IndexName='user-id-status-index',
            KeyConditionExpression='user_id = :user_id AND #status = :status',
            ExpressionAttributeNames={'#status': 'status'},
            ExpressionAttributeValues={
                ':user_id': user_id,
                ':status': 'active'
            }
        )
        
        active_sessions = response.get('Items', [])
        
        if len(active_sessions) >= self.max_sessions_per_user:
            # 最も古いセッションを終了
            oldest_session = min(active_sessions, key=lambda x: x['last_activity'])
            await self.end_session(oldest_session['session_id'])
    
    async def _update_last_activity(self, session_id: str):
        """最終アクティビティの更新"""
        
        current_time = datetime.utcnow()
        
        self.session_table.update_item(
            Key={'session_id': session_id},
            UpdateExpression="SET last_activity = :timestamp, #ttl = :new_ttl",
            ExpressionAttributeNames={'#ttl': 'ttl'},
            ExpressionAttributeValues={
                ':timestamp': current_time.isoformat(),
                ':new_ttl': int((current_time + timedelta(hours=self.default_ttl_hours)).timestamp())
            }
        )
```

<function_calls>
<invoke name="fsAppend">
<parameter name="path">steering/agentcore-session-context-management.md    async def end_session(self, session_id: str, reason: str = "user_request"):
        """セッションの終了"""
        
        try:
            # セッション統計の計算
            session = await self.get_session(session_id, update_activity=False)
            end_time = datetime.utcnow()
            start_time = datetime.fromisoformat(session['created_at'])
            duration = (end_time - start_time).total_seconds()
            
            # 終了情報の更新
            await self.update_session(session_id, {
                'status': 'ended',
                'ended_at': end_time.isoformat(),
                'end_reason': reason,
                'metadata.session_duration': duration
            })
            
            # 会話履歴のアーカイブ
            await self._archive_conversation_history(session_id)
            
        except Exception as e:
            raise ValueError(f"Failed to end session: {e}")
    
    async def _archive_conversation_history(self, session_id: str):
        """会話履歴のアーカイブ"""
        
        # 会話履歴を取得
        history = await self.get_conversation_history(session_id)
        
        if history:
            # アーカイブテーブルに移動
            archive_table = self.dynamodb.Table('conversation-history-archive')
            
            with archive_table.batch_writer() as batch:
                for message in history:
                    message['archived_at'] = datetime.utcnow().isoformat()
                    batch.put_item(Item=message)
            
            # 元の履歴を削除
            with self.history_table.batch_writer() as batch:
                for message in history:
                    batch.delete_item(Key={
                        'session_id': message['session_id'],
                        'timestamp': message['timestamp']
                    })
    
    async def cleanup_expired_sessions(self):
        """期限切れセッションのクリーンアップ"""
        
        current_timestamp = int(datetime.utcnow().timestamp())
        
        # 期限切れセッションの検索
        response = self.session_table.scan(
            FilterExpression='#ttl < :current_time',
            ExpressionAttributeNames={'#ttl': 'ttl'},
            ExpressionAttributeValues={':current_time': current_timestamp}
        )
        
        expired_sessions = response.get('Items', [])
        
        # 期限切れセッションの処理
        for session in expired_sessions:
            try:
                await self._expire_session(session['session_id'])
            except Exception as e:
                print(f"Failed to expire session {session['session_id']}: {e}")
    
    async def _expire_session(self, session_id: str):
        """セッションの期限切れ処理"""
        
        await self.end_session(session_id, reason="expired")

class RuntimeContextManager:
    """ランタイムコンテキスト管理システム"""
    
    def __init__(self):
        self.dynamodb = boto3.resource('dynamodb')
        self.context_table = self.dynamodb.Table('agentcore-contexts')
        self.variable_table = self.dynamodb.Table('context-variables')
    
    async def create_context(
        self, 
        session_id: str, 
        request_id: str,
        initial_context: dict = None,
        context_type: str = "standard"
    ) -> 'RuntimeContext':
        """ランタイムコンテキストの作成"""
        
        context_id = f"{session_id}_{request_id}"
        current_time = datetime.utcnow()
        
        context_data = {
            'context_id': context_id,
            'session_id': session_id,
            'request_id': request_id,
            'context_type': context_type,
            'created_at': current_time.isoformat(),
            'updated_at': current_time.isoformat(),
            'context_data': initial_context or {},
            'execution_state': 'initialized',
            'variables': {},
            'function_calls': [],
            'memory_usage': 0,
            'execution_time': 0,
            'performance_metrics': {
                'cpu_time': 0,
                'memory_peak': 0,
                'io_operations': 0,
                'network_calls': 0
            },
            'error_log': [],
            'debug_info': {},
            'ttl': int((current_time + timedelta(hours=1)).timestamp())
        }
        
        # コンテキストの保存
        self.context_table.put_item(Item=context_data)
        
        return RuntimeContext(context_data, self)
    
    async def get_context(self, context_id: str) -> 'RuntimeContext':
        """コンテキストの取得"""
        
        response = self.context_table.get_item(
            Key={'context_id': context_id}
        )
        
        if 'Item' not in response:
            raise ValueError(f"Context not found: {context_id}")
        
        return RuntimeContext(response['Item'], self)
    
    async def update_context(self, context_id: str, updates: dict):
        """コンテキストの更新"""
        
        update_expression = "SET updated_at = :timestamp"
        expression_values = {':timestamp': datetime.utcnow().isoformat()}
        
        for key, value in updates.items():
            update_expression += f", {key} = :{key}"
            expression_values[f":{key}"] = value
        
        self.context_table.update_item(
            Key={'context_id': context_id},
            UpdateExpression=update_expression,
            ExpressionAttributeValues=expression_values
        )
    
    async def set_variable(self, context_id: str, key: str, value: any, variable_type: str = "runtime"):
        """コンテキスト変数の設定"""
        
        variable_data = {
            'context_id': context_id,
            'variable_key': key,
            'variable_value': value,
            'variable_type': variable_type,
            'created_at': datetime.utcnow().isoformat(),
            'ttl': int((datetime.utcnow() + timedelta(hours=1)).timestamp())
        }
        
        self.variable_table.put_item(Item=variable_data)
        
        # コンテキストの変数リストも更新
        await self.update_context(context_id, {
            f'variables.{key}': {
                'value': value,
                'type': variable_type,
                'updated_at': datetime.utcnow().isoformat()
            }
        })
    
    async def get_variable(self, context_id: str, key: str, default=None):
        """コンテキスト変数の取得"""
        
        try:
            response = self.variable_table.get_item(
                Key={
                    'context_id': context_id,
                    'variable_key': key
                }
            )
            
            if 'Item' in response:
                return response['Item']['variable_value']
            else:
                return default
        
        except Exception:
            return default

class RuntimeContext:
    """ランタイムコンテキストクラス"""
    
    def __init__(self, context_data: dict, manager: RuntimeContextManager):
        self.manager = manager
        self.context_id = context_data['context_id']
        self.session_id = context_data['session_id']
        self.request_id = context_data['request_id']
        self.context_type = context_data.get('context_type', 'standard')
        self.created_at = context_data['created_at']
        self.updated_at = context_data.get('updated_at', self.created_at)
        self.context_data = context_data.get('context_data', {})
        self.execution_state = context_data.get('execution_state', 'initialized')
        self.variables = context_data.get('variables', {})
        self.function_calls = context_data.get('function_calls', [])
        self.memory_usage = context_data.get('memory_usage', 0)
        self.execution_time = context_data.get('execution_time', 0)
        self.performance_metrics = context_data.get('performance_metrics', {})
        self.error_log = context_data.get('error_log', [])
        self.debug_info = context_data.get('debug_info', {})
    
    async def set_variable(self, key: str, value: any, variable_type: str = "runtime"):
        """変数の設定"""
        
        await self.manager.set_variable(self.context_id, key, value, variable_type)
        self.variables[key] = {
            'value': value,
            'type': variable_type,
            'updated_at': datetime.utcnow().isoformat()
        }
    
    async def get_variable(self, key: str, default=None):
        """変数の取得"""
        
        if key in self.variables:
            return self.variables[key]['value']
        
        return await self.manager.get_variable(self.context_id, key, default)
    
    async def add_function_call(self, function_name: str, args: dict, result: any, execution_time: float = 0):
        """関数呼び出しの記録"""
        
        call_record = {
            'function_name': function_name,
            'args': args,
            'result': result,
            'execution_time': execution_time,
            'timestamp': datetime.utcnow().isoformat(),
            'call_id': str(uuid.uuid4())
        }
        
        self.function_calls.append(call_record)
        
        # データベースに保存
        await self.manager.update_context(self.context_id, {
            'function_calls': self.function_calls
        })
    
    async def update_execution_state(self, state: str, additional_info: dict = None):
        """実行状態の更新"""
        
        self.execution_state = state
        
        update_data = {'execution_state': state}
        if additional_info:
            update_data.update(additional_info)
        
        await self.manager.update_context(self.context_id, update_data)
    
    async def log_error(self, error: Exception, context_info: dict = None):
        """エラーの記録"""
        
        error_record = {
            'error_type': type(error).__name__,
            'error_message': str(error),
            'context_info': context_info or {},
            'timestamp': datetime.utcnow().isoformat(),
            'error_id': str(uuid.uuid4())
        }
        
        self.error_log.append(error_record)
        
        await self.manager.update_context(self.context_id, {
            'error_log': self.error_log
        })
    
    async def update_performance_metrics(self, metrics: dict):
        """パフォーマンスメトリクスの更新"""
        
        self.performance_metrics.update(metrics)
        
        await self.manager.update_context(self.context_id, {
            'performance_metrics': self.performance_metrics
        })
    
    async def add_debug_info(self, key: str, info: any):
        """デバッグ情報の追加"""
        
        self.debug_info[key] = {
            'value': info,
            'timestamp': datetime.utcnow().isoformat()
        }
        
        await self.manager.update_context(self.context_id, {
            f'debug_info.{key}': self.debug_info[key]
        })
    
    def to_dict(self) -> dict:
        """辞書形式への変換"""
        
        return {
            'context_id': self.context_id,
            'session_id': self.session_id,
            'request_id': self.request_id,
            'context_type': self.context_type,
            'created_at': self.created_at,
            'updated_at': self.updated_at,
            'context_data': self.context_data,
            'execution_state': self.execution_state,
            'variables': self.variables,
            'function_calls': self.function_calls,
            'memory_usage': self.memory_usage,
            'execution_time': self.execution_time,
            'performance_metrics': self.performance_metrics,
            'error_log': self.error_log,
            'debug_info': self.debug_info
        }
---
inclusion: manual
---

# AgentCore Runtime セッション・コンテキスト管理 - 完全ガイド

このステアリングファイルは、Amazon Bedrock AgentCore Runtime でのセッション管理とランタイムコンテキスト管理について詳細に説明します。状態の永続化、継続性、動的情報管理などの実装方法を包括的にカバーします。

## 概要

セッション・コンテキスト管理は、エンタープライズグレードのAIエージェントにおいて、ユーザーとの継続的な対話、状態の保持、実行時情報の追跡を可能にする重要な機能です。

### 主要な機能

- **セッション管理**: ユーザーとの対話セッションの作成、維持、終了
- **ランタイムコンテキスト**: 実行時の動的情報管理
- **状態永続化**: DynamoDB を使用した状態の保存
- **会話履歴**: 対話履歴の管理とアーカイブ
- **パフォーマンス追跡**: 実行メトリクスの収集

## セッション管理システム

### 1. 高度なセッション管理

```python
import boto3
import json
import time
import uuid
from datetime import datetime, timedelta
from typing import Dict, List, Any, Optional

class AdvancedSessionManager:
    """高度なセッション管理システム"""
    
    def __init__(self):
        self.dynamodb = boto3.resource('dynamodb')
        self.session_table = self.dynamodb.Table('agentcore-sessions')
        self.history_table = self.dynamodb.Table('conversation-history')
        self.session_cache = {}  # ローカルキャッシュ
    
    async def create_session(
        self, 
        user_id: str, 
        session_config: dict = None,
        session_type: str = "conversation"
    ) -> str:
        """セッションの作成"""
        
        session_id = f"session_{int(time.time())}_{user_id}_{str(uuid.uuid4())[:8]}"
        current_time = datetime.utcnow()
        
        # デフォルト設定
        default_config = {
            "max_messages": 100,
            "timeout_minutes": 30,
            "memory_enabled": True,
            "context_window": 10,
            "auto_save": True
        }
        
        # 設定のマージ
        config = {**default_config, **(session_config or {})}
        
        session_data = {
            'session_id': session_id,
            'user_id': user_id,
            'session_type': session_type,
            'created_at': current_time.isoformat(),
            'last_activity': current_time.isoformat(),
            'status': 'active',
            'config': config,
            'metadata': {
                'message_count': 0,
                'total_tokens': 0,
                'last_model_used': None,
                'conversation_topics': [],
                'user_preferences': {},
                'session_quality_score': 0.0
            },
            'performance_stats': {
                'avg_response_time': 0.0,
                'total_processing_time': 0.0,
                'error_count': 0,
                'successful_interactions': 0
            },
            'ttl': int((current_time + timedelta(hours=24)).timestamp())
        }
        
        # DynamoDB に保存
        self.session_table.put_item(Item=session_data)
        
        # ローカルキャッシュに追加
        self.session_cache[session_id] = session_data
        
        return session_id
    
    async def get_session(self, session_id: str, update_activity: bool = True) -> dict:
        """セッション情報の取得"""
        
        # ローカルキャッシュから確認
        if session_id in self.session_cache:
            session = self.session_cache[session_id]
        else:
            # DynamoDB から取得
            response = self.session_table.get_item(
                Key={'session_id': session_id}
            )
            
            if 'Item' not in response:
                raise ValueError(f"Session not found: {session_id}")
            
            session = response['Item']
            self.session_cache[session_id] = session
        
        # セッションの有効性チェック
        if session.get('status') != 'active':
            raise ValueError(f"Session is not active: {session_id}")
        
        # タイムアウトチェック
        last_activity = datetime.fromisoformat(session['last_activity'])
        timeout_minutes = session.get('config', {}).get('timeout_minutes', 30)
        
        if datetime.utcnow() - last_activity > timedelta(minutes=timeout_minutes):
            await self.end_session(session_id, reason="timeout")
            raise ValueError(f"Session has timed out: {session_id}")
        
        # 最終アクティビティの更新
        if update_activity:
            await self._update_last_activity(session_id)
        
        return session
    
    async def update_session(self, session_id: str, updates: dict):
        """セッション情報の更新"""
        
        # 更新式の構築
        update_expression = "SET "
        expression_values = {}
        expression_names = {}
        
        for key, value in updates.items():
            if '.' in key:
                # ネストされたキーの処理
                parts = key.split('.')
                attr_name = f"#{parts[0]}"
                expression_names[attr_name] = parts[0]
                
                if len(parts) == 2:
                    update_expression += f"{attr_name}.{parts[1]} = :{key.replace('.', '_')}, "
                else:
                    # より深いネストの場合
                    nested_path = '.'.join(parts[1:])
                    update_expression += f"{attr_name}.{nested_path} = :{key.replace('.', '_')}, "
            else:
                update_expression += f"{key} = :{key}, "
            
            expression_values[f":{key.replace('.', '_')}"] = value
        
        update_expression = update_expression.rstrip(", ")
        
        # DynamoDB 更新
        update_params = {
            'Key': {'session_id': session_id},
            'UpdateExpression': update_expression,
            'ExpressionAttributeValues': expression_values
        }
        
        if expression_names:
            update_params['ExpressionAttributeNames'] = expression_names
        
        self.session_table.update_item(**update_params)
        
        # ローカルキャッシュの更新
        if session_id in self.session_cache:
            for key, value in updates.items():
                if '.' in key:
                    # ネストされた更新
                    parts = key.split('.')
                    current = self.session_cache[session_id]
                    for part in parts[:-1]:
                        if part not in current:
                            current[part] = {}
                        current = current[part]
                    current[parts[-1]] = value
                else:
                    self.session_cache[session_id][key] = value
    
    async def _update_last_activity(self, session_id: str):
        """最終アクティビティの更新"""
        
        current_time = datetime.utcnow().isoformat()
        
        await self.update_session(session_id, {
            'last_activity': current_time
        })
    
    async def get_conversation_history(
        self, 
        session_id: str, 
        limit: int = 20,
        include_metadata: bool = False
    ) -> List[dict]:
        """会話履歴の取得"""
        
        response = self.history_table.query(
            KeyConditionExpression='session_id = :session_id',
            ExpressionAttributeValues={':session_id': session_id},
            ScanIndexForward=True,  # 時系列順
            Limit=limit
        )
        
        messages = response.get('Items', [])
        
        if not include_metadata:
            # メタデータを除去してクリーンな履歴を返す
            cleaned_messages = []
            for msg in messages:
                cleaned_messages.append({
                    'role': msg.get('role'),
                    'content': msg.get('content'),
                    'timestamp': msg.get('timestamp')
                })
            return cleaned_messages
        
        return messages
    
    async def add_conversation_message(
        self, 
        session_id: str, 
        role: str, 
        content: str,
        metadata: dict = None
    ):
        """会話メッセージの追加"""
        
        timestamp = datetime.utcnow().isoformat()
        message_id = f"{session_id}_{timestamp}_{role}_{str(uuid.uuid4())[:8]}"
        
        message_data = {
            'session_id': session_id,
            'timestamp': timestamp,
            'message_id': message_id,
            'role': role,
            'content': content,
            'metadata': metadata or {},
            'ttl': int((datetime.utcnow() + timedelta(days=30)).timestamp())
        }
        
        # 会話履歴テーブルに保存
        self.history_table.put_item(Item=message_data)
        
        # セッション統計の更新
        await self.update_session(session_id, {
            'metadata.message_count': 'metadata.message_count + :inc'
        })
        
        # メッセージ数制限のチェック
        session = await self.get_session(session_id, update_activity=False)
        max_messages = session.get('config', {}).get('max_messages', 100)
        current_count = session.get('metadata', {}).get('message_count', 0)
        
        if current_count > max_messages:
            await self._cleanup_old_messages(session_id, max_messages)
    
    async def _cleanup_old_messages(self, session_id: str, keep_count: int):
        """古いメッセージのクリーンアップ"""
        
        # 全メッセージを取得
        response = self.history_table.query(
            KeyConditionExpression='session_id = :session_id',
            ExpressionAttributeValues={':session_id': session_id},
            ScanIndexForward=True
        )
        
        messages = response.get('Items', [])
        
        # 古いメッセージを削除
        if len(messages) > keep_count:
            messages_to_delete = messages[:-keep_count]
            
            with self.history_table.batch_writer() as batch:
                for message in messages_to_delete:
                    batch.delete_item(Key={
                        'session_id': message['session_id'],
                        'timestamp': message['timestamp']
                    })
### 2. セッション統合エージェント

```python
class SessionAwareAgent:
    """セッション対応エージェント"""
    
    def __init__(self):
        self.session_manager = AdvancedSessionManager()
        self.context_manager = RuntimeContextManager()
        self.bedrock_client = boto3.client('bedrock-runtime')
    
    async def process_request(self, request: dict) -> dict:
        """セッション対応リクエスト処理"""
        
        # セッション情報の取得または作成
        session_id = request.get('session_id')
        user_id = request.get('user_id', 'anonymous')
        
        if not session_id:
            session_config = request.get('session_config', {})
            session_id = await self.session_manager.create_session(
                user_id, 
                session_config,
                session_type=request.get('session_type', 'conversation')
            )
        
        session = await self.session_manager.get_session(session_id)
        
        # ランタイムコンテキストの作成
        request_id = f"req_{int(time.time())}_{str(uuid.uuid4())[:8]}"
        context = await self.context_manager.create_context(
            session_id, 
            request_id,
            {
                'user_id': user_id,
                'request_data': request,
                'session_metadata': session.get('metadata', {})
            },
            context_type=request.get('context_type', 'standard')
        )
        
        try:
            # リクエスト処理の開始
            await context.update_execution_state('processing')
            processing_start_time = time.time()
            
            # 会話履歴の取得
            conversation_history = await self.session_manager.get_conversation_history(
                session_id,
                limit=session.get('config', {}).get('context_window', 10)
            )
            
            # ユーザーメッセージの記録
            user_message = request.get('prompt', '')
            await self.session_manager.add_conversation_message(
                session_id, 
                'user', 
                user_message,
                {
                    'request_id': request_id,
                    'timestamp': datetime.utcnow().isoformat()
                }
            )
            
            # コンテキスト変数の設定
            await context.set_variable('conversation_history', conversation_history)
            await context.set_variable('user_message', user_message)
            await context.set_variable('session_config', session.get('config', {}))
            
            # プロンプトの構築
            enhanced_prompt = await self._build_enhanced_prompt(
                user_message,
                conversation_history,
                session,
                context
            )
            
            # モデル呼び出し
            model_response = await self._call_model_with_context(enhanced_prompt, context)
            
            # アシスタントレスポンスの記録
            await self.session_manager.add_conversation_message(
                session_id,
                'assistant',
                model_response,
                {
                    'request_id': request_id,
                    'model_used': 'claude-3-sonnet',
                    'processing_time': time.time() - processing_start_time
                }
            )
            
            # セッション統計の更新
            await self._update_session_stats(session_id, context, model_response)
            
            # 実行完了
            await context.update_execution_state('completed')
            
            return {
                'response': model_response,
                'session_id': session_id,
                'request_id': request_id,
                'context_summary': await self._get_context_summary(context),
                'session_stats': session.get('metadata', {})
            }
        
        except Exception as e:
            # エラー処理
            await context.log_error(e, {
                'session_id': session_id,
                'request_id': request_id,
                'user_message': request.get('prompt', '')
            })
            
            await context.update_execution_state('failed')
            
            # セッションエラー統計の更新
            await self.session_manager.update_session(session_id, {
                'performance_stats.error_count': 'performance_stats.error_count + :inc'
            })
            
            raise
    
    async def _build_enhanced_prompt(
        self, 
        prompt: str, 
        history: list, 
        session: dict, 
        context: RuntimeContext
    ) -> str:
        """拡張プロンプトの構築"""
        
        # セッション情報を考慮したプロンプト構築
        user_id = session.get('user_id', 'anonymous')
        session_type = session.get('session_type', 'conversation')
        user_preferences = session.get('metadata', {}).get('user_preferences', {})
        
        enhanced_prompt = f"""
セッション情報:
- ユーザーID: {user_id}
- セッションタイプ: {session_type}
- セッション開始: {session.get('created_at')}
- メッセージ数: {session.get('metadata', {}).get('message_count', 0)}
- ユーザー設定: {json.dumps(user_preferences, ensure_ascii=False)}

会話履歴:
"""
        
        # 会話履歴の追加（コンテキストウィンドウ内）
        context_window = session.get('config', {}).get('context_window', 10)
        recent_history = history[-context_window:] if history else []
        
        for msg in recent_history:
            role = msg.get('role', 'user')
            content = msg.get('content', '')
            timestamp = msg.get('timestamp', '')
            enhanced_prompt += f"[{timestamp}] {role}: {content}\n"
        
        enhanced_prompt += f"\n現在のユーザー入力: {prompt}\n"
        
        # コンテキスト変数の追加
        context_vars = context.variables
        if context_vars:
            enhanced_prompt += f"\nコンテキスト変数:\n"
            for key, var_info in context_vars.items():
                if key not in ['conversation_history', 'user_message']:  # 重複を避ける
                    enhanced_prompt += f"- {key}: {var_info.get('value')}\n"
        
        return enhanced_prompt
    
    async def _call_model_with_context(self, prompt: str, context: RuntimeContext) -> str:
        """コンテキスト付きモデル呼び出し"""
        
        start_time = time.time()
        
        try:
            # 関数呼び出しの記録
            await context.add_function_call(
                'bedrock_invoke_model',
                {'prompt_length': len(prompt)},
                None,  # 結果は後で更新
                0  # 実行時間は後で更新
            )
            
            response = self.bedrock_client.invoke_model(
                modelId="anthropic.claude-3-sonnet-20240229-v1:0",
                body=json.dumps({
                    "anthropic_version": "bedrock-2023-05-31",
                    "max_tokens": 1000,
                    "messages": [{"role": "user", "content": prompt}]
                })
            )
            
            execution_time = time.time() - start_time
            
            result = json.loads(response['body'].read())
            response_text = result['content'][0]['text']
            
            # パフォーマンスメトリクスの更新
            await context.update_performance_metrics({
                'model_call_time': execution_time,
                'input_tokens': result.get('usage', {}).get('input_tokens', 0),
                'output_tokens': result.get('usage', {}).get('output_tokens', 0)
            })
            
            # 関数呼び出し記録の更新
            context.function_calls[-1]['result'] = {
                'response_length': len(response_text),
                'tokens_used': result.get('usage', {}).get('output_tokens', 0)
            }
            context.function_calls[-1]['execution_time'] = execution_time
            
            return response_text
        
        except Exception as e:
            execution_time = time.time() - start_time
            
            # エラー情報の記録
            await context.log_error(e, {
                'function': 'bedrock_invoke_model',
                'prompt_length': len(prompt),
                'execution_time': execution_time
            })
            
            raise
    
    async def _update_session_stats(self, session_id: str, context: RuntimeContext, response: str):
        """セッション統計の更新"""
        
        execution_time = context.execution_time
        performance_metrics = context.performance_metrics
        
        # 統計の計算
        updates = {
            'metadata.total_tokens': f'metadata.total_tokens + :tokens',
            'metadata.last_model_used': 'claude-3-sonnet',
            'performance_stats.total_processing_time': f'performance_stats.total_processing_time + :time',
            'performance_stats.successful_interactions': 'performance_stats.successful_interactions + :inc'
        }
        
        # 平均応答時間の更新
        session = await self.session_manager.get_session(session_id, update_activity=False)
        current_avg = session.get('performance_stats', {}).get('avg_response_time', 0.0)
        successful_count = session.get('performance_stats', {}).get('successful_interactions', 0)
        
        new_avg = ((current_avg * successful_count) + execution_time) / (successful_count + 1)
        updates['performance_stats.avg_response_time'] = new_avg
        
        await self.session_manager.update_session(session_id, updates)
    
    async def _get_context_summary(self, context: RuntimeContext) -> dict:
        """コンテキストサマリーの生成"""
        
        return {
            'execution_time': context.execution_time,
            'function_calls_count': len(context.function_calls),
            'variables_count': len(context.variables),
            'errors_count': len(context.error_log),
            'performance_metrics': context.performance_metrics,
            'execution_state': context.execution_state
        }

### 3. セッション分析とインサイト

```python
class SessionAnalytics:
    """セッション分析システム"""
    
    def __init__(self):
        self.session_manager = AdvancedSessionManager()
        self.dynamodb = boto3.resource('dynamodb')
    
    async def analyze_session(self, session_id: str) -> dict:
        """セッション分析"""
        
        session = await self.session_manager.get_session(session_id, update_activity=False)
        conversation_history = await self.session_manager.get_conversation_history(
            session_id, 
            limit=1000,  # 全履歴
            include_metadata=True
        )
        
        analysis = {
            'session_overview': await self._analyze_session_overview(session),
            'conversation_analysis': await self._analyze_conversation(conversation_history),
            'performance_analysis': await self._analyze_performance(session),
            'user_behavior': await self._analyze_user_behavior(conversation_history),
            'recommendations': await self._generate_recommendations(session, conversation_history)
        }
        
        return analysis
    
    async def _analyze_session_overview(self, session: dict) -> dict:
        """セッション概要分析"""
        
        created_at = datetime.fromisoformat(session['created_at'])
        last_activity = datetime.fromisoformat(session['last_activity'])
        duration = (last_activity - created_at).total_seconds()
        
        return {
            'session_duration': duration,
            'message_count': session.get('metadata', {}).get('message_count', 0),
            'status': session.get('status'),
            'session_type': session.get('session_type'),
            'user_id': session.get('user_id'),
            'avg_response_time': session.get('performance_stats', {}).get('avg_response_time', 0),
            'error_rate': self._calculate_error_rate(session)
        }
    
    async def _analyze_conversation(self, history: list) -> dict:
        """会話分析"""
        
        if not history:
            return {'message_count': 0}
        
        user_messages = [msg for msg in history if msg.get('role') == 'user']
        assistant_messages = [msg for msg in history if msg.get('role') == 'assistant']
        
        # メッセージ長の分析
        user_msg_lengths = [len(msg.get('content', '')) for msg in user_messages]
        assistant_msg_lengths = [len(msg.get('content', '')) for msg in assistant_messages]
        
        # 時間間隔の分析
        timestamps = [datetime.fromisoformat(msg['timestamp']) for msg in history]
        intervals = []
        for i in range(1, len(timestamps)):
            interval = (timestamps[i] - timestamps[i-1]).total_seconds()
            intervals.append(interval)
        
        return {
            'total_messages': len(history),
            'user_messages': len(user_messages),
            'assistant_messages': len(assistant_messages),
            'avg_user_message_length': sum(user_msg_lengths) / len(user_msg_lengths) if user_msg_lengths else 0,
            'avg_assistant_message_length': sum(assistant_msg_lengths) / len(assistant_msg_lengths) if assistant_msg_lengths else 0,
            'avg_message_interval': sum(intervals) / len(intervals) if intervals else 0,
            'conversation_topics': self._extract_topics(history)
        }
    
    async def _analyze_performance(self, session: dict) -> dict:
        """パフォーマンス分析"""
        
        perf_stats = session.get('performance_stats', {})
        
        return {
            'avg_response_time': perf_stats.get('avg_response_time', 0),
            'total_processing_time': perf_stats.get('total_processing_time', 0),
            'successful_interactions': perf_stats.get('successful_interactions', 0),
            'error_count': perf_stats.get('error_count', 0),
            'success_rate': self._calculate_success_rate(perf_stats),
            'performance_grade': self._calculate_performance_grade(perf_stats)
        }
    
    async def _analyze_user_behavior(self, history: list) -> dict:
        """ユーザー行動分析"""
        
        user_messages = [msg for msg in history if msg.get('role') == 'user']
        
        if not user_messages:
            return {}
        
        # メッセージパターンの分析
        message_patterns = self._analyze_message_patterns(user_messages)
        
        # 活動時間の分析
        activity_patterns = self._analyze_activity_patterns(user_messages)
        
        return {
            'message_patterns': message_patterns,
            'activity_patterns': activity_patterns,
            'engagement_level': self._calculate_engagement_level(user_messages),
            'communication_style': self._analyze_communication_style(user_messages)
        }
    
    async def _generate_recommendations(self, session: dict, history: list) -> list:
        """推奨事項の生成"""
        
        recommendations = []
        
        # パフォーマンス改善の推奨
        avg_response_time = session.get('performance_stats', {}).get('avg_response_time', 0)
        if avg_response_time > 5.0:
            recommendations.append({
                'type': 'performance',
                'priority': 'high',
                'message': 'レスポンス時間が長すぎます。モデルの最適化を検討してください。'
            })
        
        # セッション設定の推奨
        message_count = session.get('metadata', {}).get('message_count', 0)
        if message_count > 50:
            recommendations.append({
                'type': 'configuration',
                'priority': 'medium',
                'message': '長時間のセッションです。定期的な要約機能の有効化を推奨します。'
            })
        
        # ユーザーエクスペリエンス改善の推奨
        error_rate = self._calculate_error_rate(session)
        if error_rate > 0.1:
            recommendations.append({
                'type': 'user_experience',
                'priority': 'high',
                'message': 'エラー率が高いです。入力検証の強化を検討してください。'
            })
        
        return recommendations
    
    def _calculate_error_rate(self, session: dict) -> float:
        """エラー率の計算"""
        
        perf_stats = session.get('performance_stats', {})
        successful = perf_stats.get('successful_interactions', 0)
        errors = perf_stats.get('error_count', 0)
        total = successful + errors
        
        return errors / total if total > 0 else 0.0
    
    def _calculate_success_rate(self, perf_stats: dict) -> float:
        """成功率の計算"""
        
        successful = perf_stats.get('successful_interactions', 0)
        errors = perf_stats.get('error_count', 0)
        total = successful + errors
        
        return successful / total if total > 0 else 0.0
    
    def _calculate_performance_grade(self, perf_stats: dict) -> str:
        """パフォーマンスグレードの計算"""
        
        success_rate = self._calculate_success_rate(perf_stats)
        avg_response_time = perf_stats.get('avg_response_time', 0)
        
        if success_rate >= 0.95 and avg_response_time <= 2.0:
            return 'A'
        elif success_rate >= 0.90 and avg_response_time <= 5.0:
            return 'B'
        elif success_rate >= 0.80 and avg_response_time <= 10.0:
            return 'C'
        else:
            return 'D'
    
    def _extract_topics(self, history: list) -> list:
        """会話トピックの抽出（簡易実装）"""
        
        # 実際の実装では自然言語処理を使用
        topics = []
        
        for msg in history:
            content = msg.get('content', '').lower()
            if 'ai' in content or '人工知能' in content:
                topics.append('AI')
            if 'プログラミング' in content or 'コード' in content:
                topics.append('プログラミング')
            if 'データ' in content or '分析' in content:
                topics.append('データ分析')
        
        return list(set(topics))
    
    def _analyze_message_patterns(self, user_messages: list) -> dict:
        """メッセージパターンの分析"""
        
        lengths = [len(msg.get('content', '')) for msg in user_messages]
        
        return {
            'avg_length': sum(lengths) / len(lengths) if lengths else 0,
            'min_length': min(lengths) if lengths else 0,
            'max_length': max(lengths) if lengths else 0,
            'message_frequency': len(user_messages)
        }
    
    def _analyze_activity_patterns(self, user_messages: list) -> dict:
        """活動パターンの分析"""
        
        if not user_messages:
            return {}
        
        timestamps = [datetime.fromisoformat(msg['timestamp']) for msg in user_messages]
        hours = [ts.hour for ts in timestamps]
        
        # 時間帯別の活動
        hour_counts = {}
        for hour in hours:
            hour_counts[hour] = hour_counts.get(hour, 0) + 1
        
        most_active_hour = max(hour_counts, key=hour_counts.get) if hour_counts else 0
        
        return {
            'most_active_hour': most_active_hour,
            'activity_distribution': hour_counts,
            'session_span_hours': (max(timestamps) - min(timestamps)).total_seconds() / 3600
        }
    
    def _calculate_engagement_level(self, user_messages: list) -> str:
        """エンゲージメントレベルの計算"""
        
        if not user_messages:
            return 'none'
        
        message_count = len(user_messages)
        avg_length = sum(len(msg.get('content', '')) for msg in user_messages) / message_count
        
        if message_count >= 20 and avg_length >= 50:
            return 'high'
        elif message_count >= 10 and avg_length >= 25:
            return 'medium'
        else:
            return 'low'
    
    def _analyze_communication_style(self, user_messages: list) -> dict:
        """コミュニケーションスタイルの分析"""
        
        if not user_messages:
            return {}
        
        total_content = ' '.join(msg.get('content', '') for msg in user_messages)
        
        # 簡易的なスタイル分析
        question_count = total_content.count('?') + total_content.count('？')
        exclamation_count = total_content.count('!') + total_content.count('！')
        
        return {
            'question_ratio': question_count / len(user_messages),
            'exclamation_ratio': exclamation_count / len(user_messages),
            'avg_message_length': len(total_content) / len(user_messages),
            'style': 'inquisitive' if question_count > len(user_messages) * 0.3 else 'conversational'
        }
```
## AgentCore 統合

### セッション・コンテキスト対応ハンドラー

```python
async def session_context_handler(event, context):
    """セッション・コンテキスト対応 AgentCore ハンドラー"""
    
    request_data = json.loads(event.get('body', '{}'))
    
    # セッション対応エージェントの初期化
    agent = SessionAwareAgent()
    
    try:
        # リクエスト処理
        result = await agent.process_request(request_data)
        
        return {
            "statusCode": 200,
            "headers": {
                "Content-Type": "application/json",
                "Access-Control-Allow-Origin": "*"
            },
            "body": json.dumps(result, ensure_ascii=False)
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
# セッション・コンテキスト管理対応 AgentCore 設定
runtime:
  entrypoint: "src/session_agent.py"
  handler: "session_context_handler"
  
  # ランタイム環境
  python_version: "3.11"
  timeout: 600  # 10分
  memory_size: 2048
  
  # 環境変数
  environment:
    LOG_LEVEL: "INFO"
    SESSION_TABLE: "agentcore-sessions"
    CONTEXT_TABLE: "agentcore-contexts"
    HISTORY_TABLE: "conversation-history"
    VARIABLE_TABLE: "context-variables"

# セッション管理設定
session_management:
  enabled: true
  
  # セッション設定
  session:
    default_timeout_minutes: 30
    max_messages_per_session: 100
    auto_cleanup_enabled: true
    cleanup_interval_hours: 24
  
  # コンテキスト設定
  context:
    variable_ttl_hours: 1
    performance_tracking: true
    debug_mode: false
  
  # 会話履歴設定
  conversation_history:
    retention_days: 30
    archive_enabled: true
    max_messages_per_query: 1000

# DynamoDB テーブル設定
dynamodb_tables:
  sessions:
    table_name: "agentcore-sessions"
    billing_mode: "PAY_PER_REQUEST"
    ttl_enabled: true
    ttl_attribute: "ttl"
  
  contexts:
    table_name: "agentcore-contexts"
    billing_mode: "PAY_PER_REQUEST"
    ttl_enabled: true
    ttl_attribute: "ttl"
  
  conversation_history:
    table_name: "conversation-history"
    billing_mode: "PAY_PER_REQUEST"
    ttl_enabled: true
    ttl_attribute: "ttl"
  
  context_variables:
    table_name: "context-variables"
    billing_mode: "PAY_PER_REQUEST"
    ttl_enabled: true
    ttl_attribute: "ttl"

# 監視設定
monitoring:
  metrics_enabled: true
  custom_metrics:
    - name: "ActiveSessions"
      unit: "Count"
    - name: "SessionDuration"
      unit: "Seconds"
    - name: "ContextOperations"
      unit: "Count"
    - name: "ConversationMessages"
      unit: "Count"
```

## ベストプラクティス

### 1. セッション管理
- 適切なタイムアウト設定
- 定期的なクリーンアップ
- セッション統計の活用
- ユーザー設定の保存

### 2. コンテキスト管理
- 変数の適切なスコープ設定
- パフォーマンスメトリクスの追跡
- エラーログの活用
- デバッグ情報の記録

### 3. 会話履歴管理
- 適切な保持期間の設定
- アーカイブ機能の活用
- プライバシー保護
- 検索性の確保

### 4. パフォーマンス最適化
- ローカルキャッシュの活用
- バッチ処理の実装
- 非同期処理の活用
- リソース使用量の監視

## トラブルシューティング

### よくある問題と解決策

#### 1. セッション管理エラー
**問題**: `Session not found`
**解決策**: 
- セッション有効性の確認
- タイムアウト設定の調整
- 自動復旧機能の実装

#### 2. コンテキスト同期エラー
**問題**: `Context synchronization failed`
**解決策**:
- 排他制御の実装
- 再試行ロジックの追加
- 整合性チェックの強化

#### 3. 会話履歴の問題
**問題**: `History retrieval timeout`
**解決策**:
- インデックスの最適化
- ページネーションの実装
- キャッシュ戦略の見直し

#### 4. パフォーマンス問題
**問題**: `High latency in session operations`
**解決策**:
- DynamoDB の読み書き容量の調整
- ローカルキャッシュの活用
- 非同期処理の実装

これらの実装により、AgentCore Runtime で高度なセッション・コンテキスト管理機能を提供できます。