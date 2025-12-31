---
inclusion: manual
---

# MCP サーバーを AgentCore Runtime にデプロイする完全ガイド

このステアリングファイルは、Model Context Protocol (MCP) サーバーを Amazon Bedrock AgentCore Runtime にデプロイして、MCPツールをホストする詳細な手順を説明します。

## 概要

Model Context Protocol (MCP) は、AIアプリケーションとコンテキストプロバイダー間の標準化されたプロトコルです。AgentCore Runtime では、MCPサーバーをデプロイすることで、エージェントが外部システムやデータソースと統合できます。

### MCP サーバーの特徴

- **標準化されたプロトコル**: 一貫したインターフェース
- **ツール提供**: エージェントが使用できる機能の拡張
- **リソース管理**: 外部データソースへのアクセス
- **プロンプト管理**: 動的なプロンプトテンプレート
- **AgentCore 統合**: ネイティブなランタイムサポート

## 前提条件

### 1. 開発環境の準備

```bash
# Python 環境（推奨：uv を使用）
uv init mcp-server-project
cd mcp-server-project

# 必要なパッケージをインストール
uv add bedrock-agentcore-starter-toolkit
uv add mcp-server-sdk
uv add pydantic
uv add fastapi
uv add uvicorn
```

### 2. AWS 認証情報の設定

```bash
# AWS CLI の設定
aws configure

# または環境変数で設定
export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_DEFAULT_REGION=us-east-1
```

### 3. 必要な IAM 権限

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "lambda:CreateFunction",
        "lambda:UpdateFunctionCode",
        "lambda:InvokeFunction",
        "lambda:GetFunction",
        "apigateway:*",
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "dynamodb:CreateTable",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ],
      "Resource": "*"
    }
  ]
}
```

## MCP サーバーの作成

### 1. プロジェクト構造の作成

```bash
# プロジェクト構造
mcp-server-project/
├── src/
│   ├── mcp_server.py        # メイン MCP サーバー
│   ├── tools/               # MCP ツール実装
│   │   ├── __init__.py
│   │   ├── database_tools.py
│   │   ├── file_tools.py
│   │   └── api_tools.py
│   ├── resources/           # MCP リソース
│   │   ├── __init__.py
│   │   └── data_resources.py
│   └── prompts/             # MCP プロンプト
│       ├── __init__.py
│       └── prompt_templates.py
├── tests/
│   └── test_mcp_server.py   # テストファイル
├── requirements.txt         # 依存関係
├── mcp_config.json         # MCP サーバー設定
└── .bedrock_agentcore.yaml # AgentCore 設定
```

### 2. MCP サーバーの実装

**src/mcp_server.py**
```python
from mcp import Server, Tool, Resource, Prompt
from mcp.types import TextContent, ImageContent, EmbeddedResource
from typing import Any, Dict, List, Optional, Union
import json
import asyncio
import logging
from datetime import datetime

# ツールとリソースのインポート
from tools.database_tools import DatabaseTools
from tools.file_tools import FileTools
from tools.api_tools import APITools
from resources.data_resources import DataResources
from prompts.prompt_templates import PromptTemplates

class CustomMCPServer:
    """カスタム MCP サーバー実装"""
    
    def __init__(self, name: str = "custom-mcp-server"):
        self.server = Server(name)
        self.logger = logging.getLogger(__name__)
        
        # ツールとリソースの初期化
        self.db_tools = DatabaseTools()
        self.file_tools = FileTools()
        self.api_tools = APITools()
        self.data_resources = DataResources()
        self.prompt_templates = PromptTemplates()
        
        # ツールの登録
        self._register_tools()
        
        # リソースの登録
        self._register_resources()
        
        # プロンプトの登録
        self._register_prompts()
    
    def _register_tools(self):
        """MCP ツールの登録"""
        
        @self.server.tool("query_database")
        async def query_database(
            query: str,
            table: str,
            parameters: Optional[Dict[str, Any]] = None
        ) -> List[Dict[str, Any]]:
            """データベースクエリツール"""
            try:
                self.logger.info(f"Executing database query: {query}")
                
                # セキュリティ検証
                if not self.db_tools.validate_query(query, table):
                    raise ValueError("Invalid or unsafe query")
                
                # クエリ実行
                results = await self.db_tools.execute_query(
                    query=query,
                    table=table,
                    parameters=parameters or {}
                )
                
                return {
                    "success": True,
                    "results": results,
                    "count": len(results),
                    "query": query
                }
                
            except Exception as e:
                self.logger.error(f"Database query failed: {e}")
                return {
                    "success": False,
                    "error": str(e),
                    "query": query
                }
        
        @self.server.tool("read_file")
        async def read_file(
            file_path: str,
            encoding: str = "utf-8",
            max_size: int = 1024 * 1024  # 1MB
        ) -> Dict[str, Any]:
            """ファイル読み取りツール"""
            try:
                self.logger.info(f"Reading file: {file_path}")
                
                # セキュリティ検証
                if not self.file_tools.validate_path(file_path):
                    raise ValueError("Invalid or unsafe file path")
                
                # ファイル読み取り
                content = await self.file_tools.read_file(
                    file_path=file_path,
                    encoding=encoding,
                    max_size=max_size
                )
                
                return {
                    "success": True,
                    "content": content,
                    "file_path": file_path,
                    "size": len(content)
                }
                
            except Exception as e:
                self.logger.error(f"File read failed: {e}")
                return {
                    "success": False,
                    "error": str(e),
                    "file_path": file_path
                }
        
        @self.server.tool("write_file")
        async def write_file(
            file_path: str,
            content: str,
            encoding: str = "utf-8",
            create_dirs: bool = True
        ) -> Dict[str, Any]:
            """ファイル書き込みツール"""
            try:
                self.logger.info(f"Writing file: {file_path}")
                
                # セキュリティ検証
                if not self.file_tools.validate_path(file_path):
                    raise ValueError("Invalid or unsafe file path")
                
                # ファイル書き込み
                await self.file_tools.write_file(
                    file_path=file_path,
                    content=content,
                    encoding=encoding,
                    create_dirs=create_dirs
                )
                
                return {
                    "success": True,
                    "file_path": file_path,
                    "size": len(content)
                }
                
            except Exception as e:
                self.logger.error(f"File write failed: {e}")
                return {
                    "success": False,
                    "error": str(e),
                    "file_path": file_path
                }
        
        @self.server.tool("call_api")
        async def call_api(
            url: str,
            method: str = "GET",
            headers: Optional[Dict[str, str]] = None,
            data: Optional[Dict[str, Any]] = None,
            timeout: int = 30
        ) -> Dict[str, Any]:
            """API 呼び出しツール"""
            try:
                self.logger.info(f"Calling API: {method} {url}")
                
                # セキュリティ検証
                if not self.api_tools.validate_url(url):
                    raise ValueError("Invalid or unsafe URL")
                
                # API 呼び出し
                response = await self.api_tools.call_api(
                    url=url,
                    method=method,
                    headers=headers or {},
                    data=data,
                    timeout=timeout
                )
                
                return {
                    "success": True,
                    "status_code": response.status_code,
                    "headers": dict(response.headers),
                    "data": response.json() if response.headers.get("content-type", "").startswith("application/json") else response.text
                }
                
            except Exception as e:
                self.logger.error(f"API call failed: {e}")
                return {
                    "success": False,
                    "error": str(e),
                    "url": url
                }
        
        @self.server.tool("search_documents")
        async def search_documents(
            query: str,
            collection: str = "default",
            limit: int = 10,
            similarity_threshold: float = 0.7
        ) -> Dict[str, Any]:
            """ドキュメント検索ツール（ベクトル検索）"""
            try:
                self.logger.info(f"Searching documents: {query}")
                
                # ベクトル検索実行
                results = await self.data_resources.search_documents(
                    query=query,
                    collection=collection,
                    limit=limit,
                    similarity_threshold=similarity_threshold
                )
                
                return {
                    "success": True,
                    "query": query,
                    "results": results,
                    "count": len(results)
                }
                
            except Exception as e:
                self.logger.error(f"Document search failed: {e}")
                return {
                    "success": False,
                    "error": str(e),
                    "query": query
                }
    
    def _register_resources(self):
        """MCP リソースの登録"""
        
        @self.server.resource("database://tables")
        async def list_database_tables() -> List[Dict[str, Any]]:
            """データベーステーブル一覧"""
            try:
                tables = await self.data_resources.list_tables()
                return [
                    {
                        "uri": f"database://tables/{table['name']}",
                        "name": table["name"],
                        "description": table.get("description", ""),
                        "mimeType": "application/json"
                    }
                    for table in tables
                ]
            except Exception as e:
                self.logger.error(f"Failed to list tables: {e}")
                return []
        
        @self.server.resource("files://")
        async def list_files(path: str = "/") -> List[Dict[str, Any]]:
            """ファイル一覧"""
            try:
                files = await self.data_resources.list_files(path)
                return [
                    {
                        "uri": f"files://{file['path']}",
                        "name": file["name"],
                        "description": f"File: {file['name']}",
                        "mimeType": file.get("mime_type", "text/plain")
                    }
                    for file in files
                ]
            except Exception as e:
                self.logger.error(f"Failed to list files: {e}")
                return []
        
        @self.server.resource("documents://")
        async def list_document_collections() -> List[Dict[str, Any]]:
            """ドキュメントコレクション一覧"""
            try:
                collections = await self.data_resources.list_collections()
                return [
                    {
                        "uri": f"documents://{collection['name']}",
                        "name": collection["name"],
                        "description": collection.get("description", ""),
                        "mimeType": "application/json"
                    }
                    for collection in collections
                ]
            except Exception as e:
                self.logger.error(f"Failed to list collections: {e}")
                return []
    
    def _register_prompts(self):
        """MCP プロンプトの登録"""
        
        @self.server.prompt("analyze_data")
        async def analyze_data_prompt(
            data_type: str,
            context: Optional[str] = None
        ) -> str:
            """データ分析用プロンプト"""
            return await self.prompt_templates.get_analysis_prompt(
                data_type=data_type,
                context=context
            )
        
        @self.server.prompt("summarize_content")
        async def summarize_content_prompt(
            content_type: str,
            length: str = "medium"
        ) -> str:
            """コンテンツ要約用プロンプト"""
            return await self.prompt_templates.get_summary_prompt(
                content_type=content_type,
                length=length
            )
        
        @self.server.prompt("generate_report")
        async def generate_report_prompt(
            report_type: str,
            data_sources: List[str],
            format: str = "markdown"
        ) -> str:
            """レポート生成用プロンプト"""
            return await self.prompt_templates.get_report_prompt(
                report_type=report_type,
                data_sources=data_sources,
                format=format
            )
    
    async def start(self, host: str = "0.0.0.0", port: int = 8000):
        """MCP サーバーの開始"""
        self.logger.info(f"Starting MCP server on {host}:{port}")
        await self.server.start(host=host, port=port)
    
    async def stop(self):
        """MCP サーバーの停止"""
        self.logger.info("Stopping MCP server")
        await self.server.stop()

# AgentCore 統合用のエントリポイント
mcp_server = CustomMCPServer()

async def handler(event, context):
    """AgentCore Runtime 用のハンドラー"""
    
    try:
        # リクエストの解析
        request_data = json.loads(event.get('body', '{}'))
        
        # MCP プロトコルメッセージの処理
        if request_data.get("method") == "tools/list":
            # ツール一覧の取得
            tools = await mcp_server.server.list_tools()
            return {
                "statusCode": 200,
                "body": json.dumps({
                    "tools": tools
                })
            }
        
        elif request_data.get("method") == "tools/call":
            # ツール呼び出し
            tool_name = request_data.get("params", {}).get("name")
            tool_args = request_data.get("params", {}).get("arguments", {})
            
            result = await mcp_server.server.call_tool(tool_name, **tool_args)
            
            return {
                "statusCode": 200,
                "body": json.dumps({
                    "result": result
                })
            }
        
        elif request_data.get("method") == "resources/list":
            # リソース一覧の取得
            resources = await mcp_server.server.list_resources()
            return {
                "statusCode": 200,
                "body": json.dumps({
                    "resources": resources
                })
            }
        
        elif request_data.get("method") == "prompts/list":
            # プロンプト一覧の取得
            prompts = await mcp_server.server.list_prompts()
            return {
                "statusCode": 200,
                "body": json.dumps({
                    "prompts": prompts
                })
            }
        
        else:
            return {
                "statusCode": 400,
                "body": json.dumps({
                    "error": "Unsupported method",
                    "method": request_data.get("method")
                })
            }
    
    except Exception as e:
        return {
            "statusCode": 500,
            "body": json.dumps({
                "error": str(e),
                "error_type": type(e).__name__
            })
        }

# 開発用サーバー起動
if __name__ == "__main__":
    import uvicorn
    
    async def main():
        await mcp_server.start()
    
    # 非同期実行
    asyncio.run(main())
```

### 3. ツール実装

**src/tools/database_tools.py**
```python
import asyncio
import boto3
from typing import Dict, List, Any, Optional
import re
import logging

class DatabaseTools:
    """データベース操作ツール"""
    
    def __init__(self):
        self.dynamodb = boto3.resource('dynamodb')
        self.logger = logging.getLogger(__name__)
        
        # 許可されたテーブル
        self.allowed_tables = [
            "users", "products", "orders", "sessions"
        ]
        
        # 危険なキーワード
        self.dangerous_keywords = [
            "DROP", "DELETE", "TRUNCATE", "ALTER", "CREATE"
        ]
    
    def validate_query(self, query: str, table: str) -> bool:
        """クエリの安全性検証"""
        
        # テーブル名の検証
        if table not in self.allowed_tables:
            return False
        
        # 危険なキーワードの検証
        query_upper = query.upper()
        for keyword in self.dangerous_keywords:
            if keyword in query_upper:
                return False
        
        return True
    
    async def execute_query(
        self, 
        query: str, 
        table: str, 
        parameters: Dict[str, Any]
    ) -> List[Dict[str, Any]]:
        """DynamoDB クエリの実行"""
        
        try:
            table_resource = self.dynamodb.Table(table)
            
            # クエリタイプの判定
            if query.upper().startswith("SELECT"):
                # Scan 操作（簡易実装）
                response = table_resource.scan()
                return response.get('Items', [])
            
            elif query.upper().startswith("GET"):
                # Get 操作
                key = parameters.get('key', {})
                response = table_resource.get_item(Key=key)
                return [response.get('Item', {})] if 'Item' in response else []
            
            else:
                raise ValueError(f"Unsupported query type: {query}")
        
        except Exception as e:
            self.logger.error(f"Database query failed: {e}")
            raise

class FileTools:
    """ファイル操作ツール"""
    
    def __init__(self):
        self.logger = logging.getLogger(__name__)
        
        # 許可されたディレクトリ
        self.allowed_paths = [
            "/tmp", "/data", "/workspace"
        ]
        
        # 危険なパターン
        self.dangerous_patterns = [
            r"\.\./", r"~/", r"/etc/", r"/root/", r"/home/"
        ]
    
    def validate_path(self, file_path: str) -> bool:
        """ファイルパスの安全性検証"""
        
        # 危険なパターンの検証
        for pattern in self.dangerous_patterns:
            if re.search(pattern, file_path):
                return False
        
        # 許可されたパスの検証
        for allowed_path in self.allowed_paths:
            if file_path.startswith(allowed_path):
                return True
        
        return False
    
    async def read_file(
        self, 
        file_path: str, 
        encoding: str = "utf-8",
        max_size: int = 1024 * 1024
    ) -> str:
        """ファイル読み取り"""
        
        try:
            # ファイルサイズチェック
            import os
            if os.path.getsize(file_path) > max_size:
                raise ValueError(f"File too large: {file_path}")
            
            # ファイル読み取り
            with open(file_path, 'r', encoding=encoding) as f:
                return f.read()
        
        except Exception as e:
            self.logger.error(f"File read failed: {e}")
            raise
    
    async def write_file(
        self, 
        file_path: str, 
        content: str,
        encoding: str = "utf-8",
        create_dirs: bool = True
    ) -> None:
        """ファイル書き込み"""
        
        try:
            # ディレクトリ作成
            if create_dirs:
                import os
                os.makedirs(os.path.dirname(file_path), exist_ok=True)
            
            # ファイル書き込み
            with open(file_path, 'w', encoding=encoding) as f:
                f.write(content)
        
        except Exception as e:
            self.logger.error(f"File write failed: {e}")
            raise

class APITools:
    """API 呼び出しツール"""
    
    def __init__(self):
        self.logger = logging.getLogger(__name__)
        
        # 許可されたドメイン
        self.allowed_domains = [
            "api.example.com",
            "httpbin.org",
            "jsonplaceholder.typicode.com"
        ]
    
    def validate_url(self, url: str) -> bool:
        """URL の安全性検証"""
        
        from urllib.parse import urlparse
        
        try:
            parsed = urlparse(url)
            
            # HTTPS のみ許可
            if parsed.scheme != "https":
                return False
            
            # 許可されたドメインの検証
            for domain in self.allowed_domains:
                if parsed.netloc == domain or parsed.netloc.endswith(f".{domain}"):
                    return True
            
            return False
        
        except Exception:
            return False
    
    async def call_api(
        self,
        url: str,
        method: str = "GET",
        headers: Dict[str, str] = None,
        data: Any = None,
        timeout: int = 30
    ):
        """API 呼び出し"""
        
        import aiohttp
        
        try:
            async with aiohttp.ClientSession(timeout=aiohttp.ClientTimeout(total=timeout)) as session:
                async with session.request(
                    method=method,
                    url=url,
                    headers=headers,
                    json=data if method in ["POST", "PUT", "PATCH"] else None
                ) as response:
                    return response
        
        except Exception as e:
            self.logger.error(f"API call failed: {e}")
            raise
```

### 4. リソース実装

**src/resources/data_resources.py**
```python
import boto3
from typing import List, Dict, Any
import logging

class DataResources:
    """データリソース管理"""
    
    def __init__(self):
        self.dynamodb = boto3.resource('dynamodb')
        self.s3 = boto3.client('s3')
        self.logger = logging.getLogger(__name__)
    
    async def list_tables(self) -> List[Dict[str, Any]]:
        """DynamoDB テーブル一覧"""
        
        try:
            client = boto3.client('dynamodb')
            response = client.list_tables()
            
            tables = []
            for table_name in response.get('TableNames', []):
                # テーブル詳細を取得
                table_info = client.describe_table(TableName=table_name)
                
                tables.append({
                    "name": table_name,
                    "description": f"DynamoDB table: {table_name}",
                    "item_count": table_info['Table'].get('ItemCount', 0),
                    "size_bytes": table_info['Table'].get('TableSizeBytes', 0)
                })
            
            return tables
        
        except Exception as e:
            self.logger.error(f"Failed to list tables: {e}")
            return []
    
    async def list_files(self, path: str = "/") -> List[Dict[str, Any]]:
        """S3 ファイル一覧"""
        
        try:
            # S3 バケット設定（環境変数から取得）
            import os
            bucket_name = os.getenv('S3_BUCKET_NAME', 'default-bucket')
            
            response = self.s3.list_objects_v2(
                Bucket=bucket_name,
                Prefix=path.lstrip('/')
            )
            
            files = []
            for obj in response.get('Contents', []):
                files.append({
                    "name": obj['Key'].split('/')[-1],
                    "path": obj['Key'],
                    "size": obj['Size'],
                    "last_modified": obj['LastModified'].isoformat(),
                    "mime_type": self._get_mime_type(obj['Key'])
                })
            
            return files
        
        except Exception as e:
            self.logger.error(f"Failed to list files: {e}")
            return []
    
    async def list_collections(self) -> List[Dict[str, Any]]:
        """ドキュメントコレクション一覧"""
        
        try:
            # OpenSearch や Pinecone などのベクトルDBとの統合
            # ここでは簡易実装
            collections = [
                {
                    "name": "documents",
                    "description": "General document collection",
                    "document_count": 1000,
                    "embedding_model": "text-embedding-ada-002"
                },
                {
                    "name": "knowledge_base",
                    "description": "Knowledge base articles",
                    "document_count": 500,
                    "embedding_model": "text-embedding-ada-002"
                }
            ]
            
            return collections
        
        except Exception as e:
            self.logger.error(f"Failed to list collections: {e}")
            return []
    
    async def search_documents(
        self,
        query: str,
        collection: str = "default",
        limit: int = 10,
        similarity_threshold: float = 0.7
    ) -> List[Dict[str, Any]]:
        """ドキュメント検索（ベクトル検索）"""
        
        try:
            # ベクトル検索の実装（簡易版）
            # 実際の実装では OpenSearch、Pinecone、Chroma などを使用
            
            results = [
                {
                    "id": f"doc_{i}",
                    "title": f"Document {i} matching '{query}'",
                    "content": f"This is document {i} that matches the query '{query}'",
                    "similarity_score": 0.9 - (i * 0.1),
                    "metadata": {
                        "source": f"source_{i}.pdf",
                        "page": i + 1,
                        "collection": collection
                    }
                }
                for i in range(min(limit, 5))
            ]
            
            # 類似度でフィルタリング
            filtered_results = [
                result for result in results 
                if result["similarity_score"] >= similarity_threshold
            ]
            
            return filtered_results
        
        except Exception as e:
            self.logger.error(f"Document search failed: {e}")
            return []
    
    def _get_mime_type(self, file_path: str) -> str:
        """ファイル拡張子から MIME タイプを推定"""
        
        import mimetypes
        mime_type, _ = mimetypes.guess_type(file_path)
        return mime_type or "application/octet-stream"
```

### 5. プロンプトテンプレート

**src/prompts/prompt_templates.py**
```python
from typing import List, Optional

class PromptTemplates:
    """MCP プロンプトテンプレート管理"""
    
    async def get_analysis_prompt(
        self, 
        data_type: str, 
        context: Optional[str] = None
    ) -> str:
        """データ分析用プロンプト"""
        
        base_prompt = f"""
あなたは{data_type}データの分析専門家です。
以下のデータを詳細に分析し、洞察を提供してください。

分析観点:
1. データの概要と特徴
2. 傾向とパターンの識別
3. 異常値や外れ値の検出
4. 改善提案や推奨事項

"""
        
        if context:
            base_prompt += f"\n追加コンテキスト: {context}\n"
        
        base_prompt += """
分析結果は以下の形式で提供してください:
- 要約
- 詳細分析
- 推奨アクション
"""
        
        return base_prompt
    
    async def get_summary_prompt(
        self, 
        content_type: str, 
        length: str = "medium"
    ) -> str:
        """コンテンツ要約用プロンプト"""
        
        length_instructions = {
            "short": "1-2文で簡潔に",
            "medium": "3-5文で適度に詳しく",
            "long": "段落形式で詳細に"
        }
        
        instruction = length_instructions.get(length, length_instructions["medium"])
        
        return f"""
以下の{content_type}を{instruction}要約してください。

要約の要件:
- 主要なポイントを漏らさない
- 正確性を保つ
- 読みやすい形式で提供
- 重要度順に整理

要約対象のコンテンツ:
"""
    
    async def get_report_prompt(
        self, 
        report_type: str, 
        data_sources: List[str], 
        format: str = "markdown"
    ) -> str:
        """レポート生成用プロンプト"""
        
        sources_text = "、".join(data_sources)
        
        format_instructions = {
            "markdown": "Markdown形式で構造化",
            "html": "HTML形式で整理",
            "text": "プレーンテキストで簡潔に"
        }
        
        format_instruction = format_instructions.get(format, format_instructions["markdown"])
        
        return f"""
{report_type}レポートを作成してください。

データソース: {sources_text}
出力形式: {format_instruction}

レポート構成:
1. エグゼクティブサマリー
2. データ概要
3. 詳細分析
4. 結論と推奨事項
5. 付録（必要に応じて）

レポート作成の指針:
- データに基づいた客観的な分析
- 明確で理解しやすい表現
- 実行可能な推奨事項
- 適切な視覚化の提案
"""
```

## AgentCore Runtime への統合

### 1. AgentCore 設定ファイル

**.bedrock_agentcore.yaml**
```yaml
# AgentCore Runtime 設定
runtime:
  entrypoint: "src/mcp_server.py"
  handler: "handler"
  
  # ランタイム環境
  python_version: "3.11"
  timeout: 300
  memory_size: 1024
  
  # 環境変数
  environment:
    LOG_LEVEL: "INFO"
    MCP_SERVER_NAME: "custom-mcp-server"
    S3_BUCKET_NAME: "${S3_BUCKET_NAME}"
    DYNAMODB_REGION: "us-east-1"

# MCP サーバー設定
mcp:
  enabled: true
  server_name: "custom-mcp-server"
  protocol_version: "1.0"
  
  # ツール設定
  tools:
    enabled: true
    timeout: 30
    max_concurrent: 10
  
  # リソース設定
  resources:
    enabled: true
    cache_ttl: 300
  
  # プロンプト設定
  prompts:
    enabled: true
    template_cache: true

# セキュリティ設定
security:
  input_validation: true
  output_filtering: true
  rate_limiting:
    requests_per_minute: 100
    burst_limit: 10
  
  # MCP 固有のセキュリティ
  mcp_security:
    tool_execution_timeout: 30
    resource_access_control: true
    prompt_injection_protection: true

# 監視設定
monitoring:
  metrics_enabled: true
  tracing_enabled: true
  custom_metrics:
    - name: "MCPToolInvocations"
      unit: "Count"
    - name: "MCPResourceAccess"
      unit: "Count"
    - name: "MCPPromptGeneration"
      unit: "Count"
```

### 2. MCP 設定ファイル

**mcp_config.json**
```json
{
  "server": {
    "name": "custom-mcp-server",
    "version": "1.0.0",
    "description": "Custom MCP server for AgentCore Runtime"
  },
  "capabilities": {
    "tools": {
      "listChanged": true,
      "supportsProgress": true
    },
    "resources": {
      "subscribe": true,
      "listChanged": true
    },
    "prompts": {
      "listChanged": true
    }
  },
  "tools": [
    {
      "name": "query_database",
      "description": "Execute database queries",
      "inputSchema": {
        "type": "object",
        "properties": {
          "query": {"type": "string"},
          "table": {"type": "string"},
          "parameters": {"type": "object"}
        },
        "required": ["query", "table"]
      }
    },
    {
      "name": "read_file",
      "description": "Read file contents",
      "inputSchema": {
        "type": "object",
        "properties": {
          "file_path": {"type": "string"},
          "encoding": {"type": "string", "default": "utf-8"},
          "max_size": {"type": "integer", "default": 1048576}
        },
        "required": ["file_path"]
      }
    },
    {
      "name": "write_file",
      "description": "Write content to file",
      "inputSchema": {
        "type": "object",
        "properties": {
          "file_path": {"type": "string"},
          "content": {"type": "string"},
          "encoding": {"type": "string", "default": "utf-8"},
          "create_dirs": {"type": "boolean", "default": true}
        },
        "required": ["file_path", "content"]
      }
    },
    {
      "name": "call_api",
      "description": "Make HTTP API calls",
      "inputSchema": {
        "type": "object",
        "properties": {
          "url": {"type": "string"},
          "method": {"type": "string", "default": "GET"},
          "headers": {"type": "object"},
          "data": {"type": "object"},
          "timeout": {"type": "integer", "default": 30}
        },
        "required": ["url"]
      }
    },
    {
      "name": "search_documents",
      "description": "Search documents using vector similarity",
      "inputSchema": {
        "type": "object",
        "properties": {
          "query": {"type": "string"},
          "collection": {"type": "string", "default": "default"},
          "limit": {"type": "integer", "default": 10},
          "similarity_threshold": {"type": "number", "default": 0.7}
        },
        "required": ["query"]
      }
    }
  ],
  "resources": [
    {
      "uri": "database://tables",
      "name": "Database Tables",
      "description": "List of available database tables"
    },
    {
      "uri": "files://",
      "name": "File System",
      "description": "File system access"
    },
    {
      "uri": "documents://",
      "name": "Document Collections",
      "description": "Available document collections for search"
    }
  ],
  "prompts": [
    {
      "name": "analyze_data",
      "description": "Generate data analysis prompt",
      "arguments": [
        {
          "name": "data_type",
          "description": "Type of data to analyze",
          "required": true
        },
        {
          "name": "context",
          "description": "Additional context for analysis",
          "required": false
        }
      ]
    },
    {
      "name": "summarize_content",
      "description": "Generate content summarization prompt",
      "arguments": [
        {
          "name": "content_type",
          "description": "Type of content to summarize",
          "required": true
        },
        {
          "name": "length",
          "description": "Summary length (short/medium/long)",
          "required": false
        }
      ]
    },
    {
      "name": "generate_report",
      "description": "Generate report creation prompt",
      "arguments": [
        {
          "name": "report_type",
          "description": "Type of report to generate",
          "required": true
        },
        {
          "name": "data_sources",
          "description": "List of data sources",
          "required": true
        },
        {
          "name": "format",
          "description": "Output format (markdown/html/text)",
          "required": false
        }
      ]
    }
  ]
}
```

## デプロイ手順

### 1. ローカル開発とテスト

```bash
# 開発環境の起動
uv run agentcore dev --config .bedrock_agentcore.yaml

# 別ターミナルで MCP ツールのテスト
uv run agentcore invoke --dev '{
  "method": "tools/list"
}'

# データベースクエリツールのテスト
uv run agentcore invoke --dev '{
  "method": "tools/call",
  "params": {
    "name": "query_database",
    "arguments": {
      "query": "SELECT * FROM users",
      "table": "users"
    }
  }
}'

# ファイル操作ツールのテスト
uv run agentcore invoke --dev '{
  "method": "tools/call",
  "params": {
    "name": "read_file",
    "arguments": {
      "file_path": "/tmp/test.txt"
    }
  }
}'

# ドキュメント検索ツールのテスト
uv run agentcore invoke --dev '{
  "method": "tools/call",
  "params": {
    "name": "search_documents",
    "arguments": {
      "query": "machine learning",
      "limit": 5
    }
  }
}'
```

### 2. 単体テストの実行

**tests/test_mcp_server.py**
```python
import pytest
import asyncio
import json
from src.mcp_server import CustomMCPServer

@pytest.fixture
async def mcp_server():
    server = CustomMCPServer()
    yield server
    await server.stop()

@pytest.mark.asyncio
async def test_list_tools(mcp_server):
    """ツール一覧のテスト"""
    tools = await mcp_server.server.list_tools()
    
    assert len(tools) > 0
    tool_names = [tool["name"] for tool in tools]
    assert "query_database" in tool_names
    assert "read_file" in tool_names
    assert "call_api" in tool_names

@pytest.mark.asyncio
async def test_database_tool(mcp_server):
    """データベースツールのテスト"""
    result = await mcp_server.server.call_tool(
        "query_database",
        query="SELECT * FROM users",
        table="users"
    )
    
    assert result["success"] is True
    assert "results" in result

@pytest.mark.asyncio
async def test_file_tools(mcp_server):
    """ファイルツールのテスト"""
    # ファイル書き込みテスト
    write_result = await mcp_server.server.call_tool(
        "write_file",
        file_path="/tmp/test_mcp.txt",
        content="Test content for MCP"
    )
    
    assert write_result["success"] is True
    
    # ファイル読み取りテスト
    read_result = await mcp_server.server.call_tool(
        "read_file",
        file_path="/tmp/test_mcp.txt"
    )
    
    assert read_result["success"] is True
    assert read_result["content"] == "Test content for MCP"

@pytest.mark.asyncio
async def test_document_search(mcp_server):
    """ドキュメント検索のテスト"""
    result = await mcp_server.server.call_tool(
        "search_documents",
        query="test query",
        limit=3
    )
    
    assert result["success"] is True
    assert "results" in result
    assert len(result["results"]) <= 3

@pytest.mark.asyncio
async def test_resources(mcp_server):
    """リソースのテスト"""
    resources = await mcp_server.server.list_resources()
    
    assert len(resources) > 0
    resource_uris = [resource["uri"] for resource in resources]
    assert any("database://" in uri for uri in resource_uris)
    assert any("files://" in uri for uri in resource_uris)

@pytest.mark.asyncio
async def test_prompts(mcp_server):
    """プロンプトのテスト"""
    prompts = await mcp_server.server.list_prompts()
    
    assert len(prompts) > 0
    prompt_names = [prompt["name"] for prompt in prompts]
    assert "analyze_data" in prompt_names
    assert "summarize_content" in prompt_names

# テスト実行
if __name__ == "__main__":
    pytest.main([__file__, "-v"])
```

```bash
# テストの実行
uv run pytest tests/ -v

# カバレッジ付きテスト
uv run pytest tests/ --cov=src --cov-report=html
```

### 3. 本番デプロイ

```bash
# 1. 設定の確認
uv run agentcore configure \
  --entrypoint src/mcp_server.py \
  --handler handler \
  --region us-east-1 \
  --timeout 300 \
  --memory 1024

# 2. 環境変数の設定
export S3_BUCKET_NAME="my-mcp-data-bucket"
export DYNAMODB_REGION="us-east-1"

# 3. デプロイの実行
uv run agentcore launch --region us-east-1

# 4. デプロイ状況の確認
uv run agentcore status --region us-east-1

# 5. 本番環境でのテスト
uv run agentcore invoke '{
  "method": "tools/list"
}' --region us-east-1
```

### 4. 本番環境での動作確認

```bash
# MCP ツール一覧の確認
uv run agentcore invoke '{
  "method": "tools/list"
}' --region us-east-1

# データベースツールのテスト
uv run agentcore invoke '{
  "method": "tools/call",
  "params": {
    "name": "query_database",
    "arguments": {
      "query": "SELECT * FROM products LIMIT 5",
      "table": "products"
    }
  }
}' --region us-east-1

# リソース一覧の確認
uv run agentcore invoke '{
  "method": "resources/list"
}' --region us-east-1

# プロンプト生成のテスト
uv run agentcore invoke '{
  "method": "prompts/get",
  "params": {
    "name": "analyze_data",
    "arguments": {
      "data_type": "sales",
      "context": "Q4 performance review"
    }
  }
}' --region us-east-1
```

## エージェントとの統合

### 1. MCP ツールを使用するエージェント

```python
from bedrock_agentcore import BedrockAgentCoreApp
import json
import boto3

class MCPIntegratedAgent:
    """MCP ツールを統合したエージェント"""
    
    def __init__(self, mcp_endpoint: str):
        self.mcp_endpoint = mcp_endpoint
        self.bedrock_client = boto3.client('bedrock-runtime')
    
    async def process_request(self, request: dict) -> dict:
        """リクエスト処理"""
        
        prompt = request.get("prompt", "")
        
        # プロンプトの分析
        intent = await self._analyze_intent(prompt)
        
        # 必要に応じて MCP ツールを呼び出し
        if intent.get("needs_data"):
            data = await self._fetch_data_via_mcp(intent)
            context = f"取得したデータ: {json.dumps(data, ensure_ascii=False)}"
        else:
            context = ""
        
        # 最終レスポンスの生成
        response = await self._generate_response(prompt, context)
        
        return {
            "response": response,
            "tools_used": intent.get("tools", []),
            "data_sources": intent.get("data_sources", [])
        }
    
    async def _analyze_intent(self, prompt: str) -> dict:
        """プロンプトの意図分析"""
        
        analysis_prompt = f"""
以下のユーザープロンプトを分析し、必要なツールやデータソースを特定してください。

ユーザープロンプト: {prompt}

分析結果をJSON形式で返してください:
{{
  "needs_data": boolean,
  "tools": ["tool_name1", "tool_name2"],
  "data_sources": ["source1", "source2"],
  "query_type": "database|file|api|search"
}}
"""
        
        # Bedrock でプロンプト分析
        response = self.bedrock_client.invoke_model(
            modelId="anthropic.claude-3-sonnet-20240229-v1:0",
            body=json.dumps({
                "anthropic_version": "bedrock-2023-05-31",
                "max_tokens": 500,
                "messages": [{"role": "user", "content": analysis_prompt}]
            })
        )
        
        result = json.loads(response['body'].read())
        content = result['content'][0]['text']
        
        try:
            return json.loads(content)
        except:
            return {"needs_data": False, "tools": [], "data_sources": []}
    
    async def _fetch_data_via_mcp(self, intent: dict) -> dict:
        """MCP ツール経由でのデータ取得"""
        
        import aiohttp
        
        data = {}
        
        for tool in intent.get("tools", []):
            try:
                if tool == "query_database":
                    # データベースクエリ
                    payload = {
                        "method": "tools/call",
                        "params": {
                            "name": "query_database",
                            "arguments": {
                                "query": "SELECT * FROM products LIMIT 10",
                                "table": "products"
                            }
                        }
                    }
                
                elif tool == "search_documents":
                    # ドキュメント検索
                    payload = {
                        "method": "tools/call",
                        "params": {
                            "name": "search_documents",
                            "arguments": {
                                "query": intent.get("search_query", ""),
                                "limit": 5
                            }
                        }
                    }
                
                else:
                    continue
                
                # MCP サーバーへのリクエスト
                async with aiohttp.ClientSession() as session:
                    async with session.post(
                        self.mcp_endpoint,
                        json=payload
                    ) as response:
                        if response.status == 200:
                            result = await response.json()
                            data[tool] = result.get("result", {})
            
            except Exception as e:
                data[tool] = {"error": str(e)}
        
        return data
    
    async def _generate_response(self, prompt: str, context: str) -> str:
        """最終レスポンスの生成"""
        
        enhanced_prompt = f"""
ユーザーの質問: {prompt}

{context}

上記の情報を基に、ユーザーの質問に対して適切で有用な回答を提供してください。
"""
        
        response = self.bedrock_client.invoke_model(
            modelId="anthropic.claude-3-sonnet-20240229-v1:0",
            body=json.dumps({
                "anthropic_version": "bedrock-2023-05-31",
                "max_tokens": 1000,
                "messages": [{"role": "user", "content": enhanced_prompt}]
            })
        )
        
        result = json.loads(response['body'].read())
        return result['content'][0]['text']

# AgentCore 統合
async def handler(event, context):
    """AgentCore Runtime 用のハンドラー"""
    
    request_data = json.loads(event.get('body', '{}'))
    
    # MCP エンドポイントの設定
    mcp_endpoint = "https://your-mcp-server-endpoint.amazonaws.com"
    
    agent = MCPIntegratedAgent(mcp_endpoint)
    
    try:
        result = await agent.process_request(request_data)
        
        return {
            "statusCode": 200,
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

## 監視とメンテナンス

### 1. MCP サーバーの監視

```python
from bedrock_agentcore.monitoring import MetricsCollector
import time
import logging

class MCPServerMonitor:
    """MCP サーバー監視"""
    
    def __init__(self):
        self.metrics = MetricsCollector()
        self.logger = logging.getLogger(__name__)
    
    async def monitor_tool_execution(self, tool_name: str, execution_func):
        """ツール実行の監視"""
        
        start_time = time.time()
        
        try:
            result = await execution_func()
            
            # 成功メトリクス
            duration = time.time() - start_time
            self.metrics.put_metric("MCPToolSuccess", 1, unit="Count",
                                   dimensions={"ToolName": tool_name})
            self.metrics.put_metric("MCPToolDuration", duration, unit="Seconds",
                                   dimensions={"ToolName": tool_name})
            
            return result
        
        except Exception as e:
            # 失敗メトリクス
            duration = time.time() - start_time
            self.metrics.put_metric("MCPToolFailure", 1, unit="Count",
                                   dimensions={"ToolName": tool_name})
            self.metrics.put_metric("MCPToolErrorDuration", duration, unit="Seconds",
                                   dimensions={"ToolName": tool_name})
            
            self.logger.error(f"MCP tool {tool_name} failed: {e}")
            raise
    
    async def monitor_resource_access(self, resource_uri: str, access_func):
        """リソースアクセスの監視"""
        
        start_time = time.time()
        
        try:
            result = await access_func()
            
            # 成功メトリクス
            duration = time.time() - start_time
            self.metrics.put_metric("MCPResourceAccess", 1, unit="Count",
                                   dimensions={"ResourceURI": resource_uri})
            self.metrics.put_metric("MCPResourceAccessDuration", duration, unit="Seconds",
                                   dimensions={"ResourceURI": resource_uri})
            
            return result
        
        except Exception as e:
            # 失敗メトリクス
            self.metrics.put_metric("MCPResourceAccessFailure", 1, unit="Count",
                                   dimensions={"ResourceURI": resource_uri})
            
            self.logger.error(f"MCP resource access {resource_uri} failed: {e}")
            raise
```

### 2. ログ管理

```python
import structlog
import json

# MCP サーバー用の構造化ログ設定
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    wrapper_class=structlog.stdlib.BoundLogger,
    cache_logger_on_first_use=True,
)

class MCPServerLogger:
    """MCP サーバー専用ログ"""
    
    def __init__(self):
        self.logger = structlog.get_logger(__name__)
    
    def log_tool_call(self, tool_name: str, arguments: dict, result: dict):
        """ツール呼び出しログ"""
        
        self.logger.info(
            "MCP tool executed",
            tool_name=tool_name,
            arguments=arguments,
            success=result.get("success", False),
            execution_time=result.get("execution_time", 0)
        )
    
    def log_resource_access(self, resource_uri: str, access_type: str, result: dict):
        """リソースアクセスログ"""
        
        self.logger.info(
            "MCP resource accessed",
            resource_uri=resource_uri,
            access_type=access_type,
            success=result.get("success", False),
            data_size=len(str(result.get("data", "")))
        )
    
    def log_error(self, operation: str, error: Exception, context: dict = None):
        """エラーログ"""
        
        self.logger.error(
            "MCP operation failed",
            operation=operation,
            error=str(error),
            error_type=type(error).__name__,
            context=context or {}
        )
```

## トラブルシューティング

### よくある問題と解決策

#### 1. **MCP サーバー起動エラー**
**問題**: `MCP server failed to start`
**解決策**:
```bash
# ポート競合の確認
netstat -tulpn | grep :8000

# 設定ファイルの検証
python -c "import json; json.load(open('mcp_config.json'))"

# 依存関係の確認
uv list | grep mcp
```

#### 2. **ツール実行エラー**
**問題**: `Tool execution timeout`
**解決策**:
```yaml
# .bedrock_agentcore.yaml でタイムアウト調整
mcp:
  tools:
    timeout: 60  # 30秒から60秒に延長
```

#### 3. **リソースアクセスエラー**
**問題**: `Resource access denied`
**解決策**:
```python
# IAM 権限の確認
import boto3

def check_permissions():
    try:
        # DynamoDB アクセステスト
        dynamodb = boto3.resource('dynamodb')
        tables = list(dynamodb.tables.all())
        
        # S3 アクセステスト
        s3 = boto3.client('s3')
        buckets = s3.list_buckets()
        
        print("Permissions OK")
    except Exception as e:
        print(f"Permission error: {e}")
```

#### 4. **プロンプト生成エラー**
**問題**: `Prompt template not found`
**解決策**:
```python
# プロンプトテンプレートの検証
async def validate_prompts():
    templates = PromptTemplates()
    
    try:
        # 各プロンプトのテスト
        await templates.get_analysis_prompt("test", "test context")
        await templates.get_summary_prompt("test", "medium")
        await templates.get_report_prompt("test", ["source1"], "markdown")
        
        print("All prompts validated")
    except Exception as e:
        print(f"Prompt validation error: {e}")
```

## まとめ

MCP サーバーを AgentCore Runtime にデプロイすることで、以下の利点が得られます：

### **機能拡張**
- 標準化されたツールインターフェース
- 豊富なリソースアクセス
- 動的なプロンプト生成

### **統合性**
- AgentCore との seamless な統合
- 既存システムとの連携
- スケーラブルなアーキテクチャ

### **運用性**
- 包括的な監視とログ
- セキュリティ機能
- エラーハンドリング

この統合により、エージェントの機能を大幅に拡張し、複雑なタスクを効率的に処理できるようになります。