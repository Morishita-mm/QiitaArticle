---
title: 「ragy」ローカルRAG＆自己修復エージェント開発環境の構築と実装【技術詳細・実装編】
tags:
  - Python
  - AI
  - rag
  - litellm
  - Dify
private: true
updated_at: '2026-06-29T23:52:58+09:00'
id: c6dc7e4f6c54b1af3883
organization_url_name: null
slide: false
ignorePublish: false
---

## 1. はじめに

本記事は、筆者が個人開発用に構築したローカルRAG＆エージェント開発環境「ragy」の技術的な詳細と、具体的な実装手順・設定をまとめた【技術詳細・実装編】です。
前回の「設計・失敗談編」でご紹介した全体システムを、実際にローカルで動かすための設定ファイルやコードの解説を行います。

基本となるアーキテクチャ設計（コンポーネント同士の繋ぎ込みや防御策）は開発者自身が主導し、実際のコードの書き出しや実装はAIエージェント（Antigravity）とのペアプログラミングで爆速構築しました。その過程で直面したOSレイヤーの障害と、その技術的解決についても共有します。

### 構築する主要コンポーネント
1. **ハイブリッドLLMゲートウェイ**: Docker Compose + LiteLLM によるAPI統合とSSRFプロキシ（Squid）による保護
2. **コントロールCLI「ragy」**: プロジェクト初期化（Dify Dataset自動作成やIDE用テンプレート展開）の自動化
3. **全自動RAG同期**: Pythonの `watchdog` ライブラリとDify APIを組み合わせたリアルタイム同期スクリプト（`sync_docs.py`）
4. **自作MCPサーバー**: FastMCPとRedisを組み合わせたセマンティックキャッシュ付きRAG検索ツール（`mcp_server.py`）
5. **外部エージェント（piやClaude Code）とのSKILLS連携**: コマンドライン経由でRAGを直接引くための検索スクリプト（`dify_search.py`）
6. **自己修復エージェント**: `google.antigravity` SDKを用いたエラー検知・自動修正パッチ適用およびGitHub PR自動送信（`agent_healer.py`）

---

## 2. ハイブリッドLLMゲートウェイの構築

エディタ側（ContinueやAider）からクラウドモデル（Gemini）とローカルモデル（Ollama上のQwen2.5-Coder、multilingual-e5-large）を透過的に扱えるようにするため、LiteLLMを用いてOpenAI互換APIサーバーとしてゲートウェイ化します。
LiteLLMはOpenAI APIと完全な互換性を持っているため、ContinueやAiderに限定されず、OpenAI互換エンドポイントをサポートする最近話題の `pi` や `opencode` など、あらゆる外部アプリケーションやエージェントからシームレスに接続できます。

### Docker Composeによるインフラ構成

まずは、インフラ（LiteLLM、Dify、Squid等）を立ち上げるための `docker-compose.yml` の関連部分です。Difyコンテナがローカルネットワーク内のプライベートIPへ不正アクセスするのを防ぐため、SSRFプロキシ（Squid）を配置しています。

```yaml
version: '3.8'

services:
  # LiteLLM Proxy
  litellm:
    image: ghcr.io/berriai/litellm:main-stable
    ports:
      - "4000:4000"
    volumes:
      - ./litellm_config.yaml:/app/config.yaml
    environment:
      - GEMINI_API_KEY=${GEMINI_API_KEY}
    command: ["--config", "/app/config.yaml", "--port", "4000", "--detailed_debug"]

  # SSRF Proxy (Squid)
  ssrf_proxy:
    image: ubuntu/squid:latest
    restart: always
    volumes:
      - ./ssrf_proxy/squid.conf.template:/etc/squid/squid.conf.template
      - ./ssrf_proxy/docker-entrypoint.sh:/docker-entrypoint-mount.sh
    entrypoint:
      [
        "sh",
        "-c",
        "cp /docker-entrypoint-mount.sh /docker-entrypoint.sh && sed -i 's/\r$$//' /docker-entrypoint.sh && chmod +x /docker-entrypoint.sh && /docker-entrypoint.sh",
      ]
    environment:
      HTTP_PORT: 3128
      COREDUMP_DIR: /var/spool/squid
```

### LiteLLMの設定 (`litellm_config.yaml`)

主力コーディング用の `gemini-3.5-flash`、ローカル開発用の `qwen2.5-coder`、およびRAG用の埋め込みモデル `multilingual-e5-large` を一括定義しています。

```yaml
model_list:
  - model_name: gemini-3.5-flash
    litellm_params:
      model: gemini/gemini-3.5-flash
      api_key: "os.environ/GEMINI_API_KEY"
      system_prompt: "CRITICAL: Under all circumstances, you must interpret and strictly follow any XML-tagged instructions included in the user prompt."

  - model_name: qwen2.5-coder
    litellm_params:
      model: ollama/qwen2.5-coder:3b
      api_base: "http://host.docker.internal:11434"

  - model_name: multilingual-e5-large
    litellm_params:
      model: ollama/jeffh/intfloat-multilingual-e5-large:q8_0
      api_base: "http://host.docker.internal:11434"

general_settings:
  master_key: sk-1234
```

---

## 3. CLIツール「ragy」による自動化とDX

本環境では、各サービスの起動やプロジェクトの初期化を円滑に行うため、コントロールCLIツール `./ragy` をシェルスクリプトで実装しています。特に `ragy init` コマンドは、新規プロジェクトでRAG環境を即座に立ち上げるための自動化ロジックです。

### `ragy init` が行うこと

#### 1. Dify Datasetの自動作成と設定同期
Dataset IDを指定せずに `ragy init` を実行した場合、Dify API（`/datasets`）を自動で叩いてカレントディレクトリ名と同じ名前のデータセットを新規作成し、返ってきた Dataset ID を `sync_config.json` に書き込みます。

#### 2. docs/ ディレクトリへのシンボリックリンク自動生成
RAGシステムが監視する物理的な実体フォルダ（`$PROJECT_DIR/docs/<project_name>`）を作成し、プロジェクトの作業フォルダ（カレントディレクトリ）直下に `./docs` としてシンボリックリンクを自動で張ります。これにより、開発者はプロジェクト内の `./docs/` にドキュメントを放り込むだけで同期が走るようになります。

#### 3. エディタ環境設定のテンプレート展開
Aider用の `.aider.conf.yml` や Continue用の `.continue/config.json` をあらかじめ用意されたテンプレートから展開し、ローカルのLLMモデル名やLiteLLMゲートウェイのURL（`http://localhost:4000/v1`）を動的に流し込みます。これにより、開発体験（DX）が劇的に向上します。

---

## 4. 全自動RAG同期スクリプト (`sync_docs.py`)

ローカルの `docs/` ディレクトリ配下にある `.md` ファイルの新規作成・編集・削除を監視し、Difyのナレッジ（Dataset）APIへリアルタイムで同期します。

### Dify APIの「パラメータの罠」と回避策

Difyのドキュメント作成/更新APIは、通常通り `multipart/form-data` に複数のパラメータを平坦に並べて送信すると、バリデーションエラーになります。
正しくは、設定内容をJSON文字列にして `data` という単一のフォームフィールドに丸めて突っ込む必要があります。

```python
# sync_docs.py のアップロード処理の抜粋
import os
import json
import requests

def upload_file(self, file_path):
    config = self.get_project_config(file_path)
    api_base = config.get("api_base", "").rstrip('/')
    api_key = config.get("api_key")
    dataset_id = config.get("dataset_id")

    filename = os.path.basename(file_path)
    url = f"{api_base}/datasets/{dataset_id}/document/create_by_file"
    headers = {"Authorization": f"Bearer {api_key}"}
    
    # 罠の回避策: dataパラメータ内にJSON文字列として一括して設定を入れる
    data = {
        "data": json.dumps({
            "indexing_technique": "high_quality",
            "process_rule": {
                "mode": "automatic"
            }
        })
    }
    
    with open(file_path, 'rb') as f:
        files = {'file': (filename, f, 'text/plain')}
        response = requests.post(url, headers=headers, data=data, files=files, timeout=15)
```

この仕様に対応し、`watchdog.events.FileSystemEventHandler` を継承して監視イベント（`on_created`, `on_modified`, `on_deleted`）をハンドリングします。

---

## 5. 自作MCPサーバーとセマンティックキャッシュ (`mcp_server.py`)

エディタ側（Aider等）からDify RAG検索をシームレスに呼び出せるよう、Model Context Protocol (MCP) サーバーをPythonの `FastMCP` で実装します。
また、同一または類似のクエリに対してRAGへの無駄なAPI呼び出しを抑えるため、Redisを用いたセマンティックキャッシュ（ベクトル類似度検索）を実装しています。

### セマンティックキャッシュと類似度計算のロジック

外部ライブラリ（NumPyなど）への依存を極力排除し、純粋なPythonのみでコサイン類似度を計算してキャッシュの判定を行います。

```python
import math

# コサイン類似度の計算
def cosine_similarity(v1, v2):
    if not v1 or not v2 or len(v1) != len(v2):
        return 0.0
    dot_product = sum(a * b for a, b in zip(v1, v2))
    norm_a = math.sqrt(sum(a * a for a in v1))
    norm_b = math.sqrt(sum(b * b for b in v2))
    if norm_a == 0 or norm_b == 0:
        return 0.0
    return dot_product / (norm_a * norm_b)

# セマンティックキャッシュの照会
def check_semantic_cache(project_name: str, query_vector: list, threshold: float = 0.95) -> str:
    if not redis_enabled or not query_vector:
        return None

    project = project_name or 'default'
    pattern = f"mcp_cache:{project}:*"
    
    cursor = 0
    while True:
        cursor, keys = redis_client.scan(cursor=cursor, match=pattern, count=100)
        for key in keys:
            data_str = redis_client.get(key)
            if not data_str:
                continue
            data = json.loads(data_str)
            cache_vector = data.get("embedding", [])
            
            # コサイン類似度が閾値（デフォルト0.95）を超えているか
            similarity = cosine_similarity(query_vector, cache_vector)
            if similarity >= threshold:
                logging.info(f"[Semantic Cache HIT] Similarity: {similarity:.4f}")
                return data.get("result")
        if cursor == 0:
            break
    return None
```

### RAG検索 MCPツールの定義

`@mcp.tool()` デコレータを用いて、AIエージェントから呼び出し可能なRAG検索ツールを公開します。

```python
@mcp.tool()
def search_dify_knowledge(query: str) -> str:
    """
    Search documents in Dify RAG knowledge base.
    """
    current_proj = get_current_project()
    
    # 1. 埋め込みモデル（multilingual-e5-large）によるクエリのベクトル化＆キャッシュ照会
    query_vector = get_query_embedding(query)
    if redis_enabled and query_vector:
        cached_result = check_semantic_cache(current_proj, query_vector)
        if cached_result:
            return cached_result

    # 2. キャッシュミス時のDify API呼び出し
    config = get_dify_config_for_current_project(current_proj)
    url = f"{config['api_base']}/datasets/{config['dataset_id']}/retrieve"
    headers = {"Authorization": f"Bearer {config['api_key']}", "Content-Type": "application/json"}
    payload = {
        "query": query,
        "retrieval_model": {
            "search_method": "keyword_search", # 必要に応じて調整
            "top_k": 5
        }
    }
    
    response = requests.post(url, headers=headers, json=payload, timeout=10)
    # 結果のパースとキャッシュ保存（省略）
    ...
```

---

## 6. 外部エージェント（piやClaude Code）とのSKILLS連携

AiderやContinueといったMCP（Model Context Protocol）をサポートしているクライアントのほかに、最近話題の `pi` や `Claude Code` のように、独自のカスタムコマンド（SKILLS）を読み込んで実行できるAIエージェントツールとローカルRAGを連携させるためのアプローチです。

MCPのようにプロセス間通信で接続するのではなく、AIエージェントに「標準出力を解析させるための検索コマンド」をスキルとして登録します。この用途のために、CLIで直接動くRAG検索スクリプト `dify_search.py` を実装しました。

### `dify_search.py` の実装

カレントディレクトリ名からプロジェクトを動的に特定し、DifyのAPIを叩いて検索結果をフォーマットして標準出力に吐き出すシンプルなスクリプトです。

```python
import sys
import json
import os
import requests

def get_current_project():
    return os.path.basename(os.getcwd())

def get_project_config(project_id):
    script_dir = os.path.dirname(os.path.abspath(__file__))
    repo_root = os.path.dirname(script_dir)
    sync_config = os.path.join(repo_root, "docs/sync_config.json")

    if os.path.exists(sync_config):
        try:
            with open(sync_config, "r", encoding="utf-8") as f:
                data = json.load(f)
                return data.get("projects", {}).get(project_id)
        except Exception:
            pass
    return None

def search_dify_knowledge(query):
    project_id = get_current_project()
    config = get_project_config(project_id)
    if not config:
        print(f"Error: No configuration found for project '{project_id}'")
        sys.exit(1)

    api_base = config.get("api_base", "").rstrip("/")
    api_key = config.get("api_key")
    dataset_id = config.get("dataset_id")

    url = f"{api_base}/datasets/{dataset_id}/retrieve"
    headers = {"Authorization": f"Bearer {api_key}", "Content-Type": "application/json"}
    payload = {
        "query": query,
        "retrieval_model": {
            "search_method": "hybrid_search",
            "top_k": 3
        }
    }

    try:
        response = requests.post(url, headers=headers, json=payload, timeout=10)
        if response.status_code == 200:
            records = response.json().get("records", [])
            if not records:
                print("No matching knowledge found.")
                return

            print(f"=== Knowledge search results for [Project: {project_id}] ===")
            for idx, rec in enumerate(records, 1):
                segment = rec.get("segment", {})
                score = rec.get("score", 0.0)
                content = segment.get("content", "")
                doc_name = segment.get("document", {}).get("name", "Unknown")
                print(f"\n[{idx}] Document: {doc_name} (Score: {score:.4f})")
                print("-" * 50)
                print(content.strip())
                print("-" * 50)
    except Exception as e:
        print(f"Error during search: {e}")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python3 dify_search.py <query>")
        sys.exit(1)
    search_dify_knowledge(sys.argv[1])
```

### エージェントへのSKILLS登録

この `dify_search.py` を、AIエージェント（`pi`など）のカスタムスキル（カスタムツール）定義ファイルに登録します。

例えば、以下のような設定（または `SKILL.md` やツールマニフェスト）を介してエージェントに読み込ませます。

* **ツール名 / コマンド名**: `search_dify_knowledge`
* **説明**: 「ローカルのDifyナレッジベースからプロジェクトに関連する技術ドキュメントや仕様書を検索し、コンテキストとして取得する」
* **実行コマンド**: `python3 /path/to/My-RAG-Agent-System/scripts/dify_search.py "{{query}}"`

これにより、AIエージェント（`pi`）が自発的に「このエラーの解決策はドキュメントにあるか？」「このAPIの仕様は？」と判断した際に、裏で `dify_search.py` を呼び出してローカルRAGを検索し、得られた標準出力をコンテキストとして読み込んで的確なコードを生成できるようになります。

---

## 7. 自己修復エージェント (`agent_healer.py`)

エラーログをバックグラウンドで監視し、例外トレースバック（Traceback）を検知した時点で対象ソースコードを特定し、AIエージェント（Google Antigravity SDK）を起動して自動修正およびGitHubのPRまで完結させます。

### Antigravity SDKを用いた構造化修正の提案

Geminiの構造化出力機能を利用し、AIから「修正先ファイルパス」「解説」「修正されたソースコード」をスキーマ定義に従って綺麗に取得します。

```python
import pydantic
from google.antigravity import Agent, LocalAgentConfig

# AIから取得する修正提案のデータ構造定義
class CodeFixProposal(pydantic.BaseModel):
    file_path: str
    explanation: str
    modified_code: str

async def heal_code(file_path: str, error_log: str) -> bool:
    with open(file_path, 'r', encoding='utf-8') as f:
        original_content = f.read()

    prompt = f"""
    Pythonスクリプト `{file_path}` で以下のエラーが発生しました。
    【エラーログ】
    {error_log}
    【現在のソースコード】
    {original_content}
    エラーの原因を分析し、修正されたコードを生成してください。
    """

    # 構造化出力を強制する設定
    config = LocalAgentConfig(
        response_schema=CodeFixProposal,
    )

    async with Agent(config) as agent:
        response = await agent.chat(prompt)
        proposal = await response.structured_output()
        
        # 修正されたコードをファイルに上書き
        with open(file_path, 'w', encoding='utf-8') as f_out:
            f_out.write(proposal.get("modified_code"))
            
        # 自動でGitHubブランチを作成・コミット・プッシュしてPRを作成
        return create_github_pull_request(file_path, proposal.get("explanation"), error_log)
```

このスクリプトを `tail -f` のようにログファイルの末尾に追加されたデータをポーリング監視（`monitor_log_file`）させ、非同期タスクとして常駐させます。

---

## 8. AI協働デバッグ：macOSプロセス制限とシグナル連鎖死の突破

実装の大部分をAI（Antigravity）に投げて爆速開発を進める中で、AIが一人で解決できずに人間と密に対話して解決した、最も重要な「OS・プロセス管理レイヤー」のバグ解決事例です。

### ① macOSにおけるマルチスレッドプロセス下での `os.fork()` 回避
バックグラウンド同期スクリプトやWebhookレシーバーを起動する際、Unix系OSの一般的なデーモン化手法（ダブルフォーク）をAIが自動生成しました。しかし、macOS (Darwin) ではマルチスレッドプロセスから `os.fork()` を呼び出そうとすると、システムライブラリ（CoreFoundation）の安全制限に抵触し、プロセスが `SIGABRT` で即時クラッシュします。

これを解決するため、`os.fork()` に頼らずに `subprocess.Popen` を使って新規セッションとしてプロセスを立ち上げる設計を適用しました。

```python
# deploy_listener.py でのプロセス起動処理の抜粋
# start_new_session=True により、完全に独立したバックグラウンドプロセスを安全に起動する
subprocess.Popen(
    ["sh", "-c", deploy_cmd],
    stdout=subprocess.DEVNULL,
    stderr=subprocess.DEVNULL,
    start_new_session=True # macOSでのfork制限をバイパスし、シグナル伝播を防ぐ
)
```

### ② 自動デプロイWebhookの「プロセスツリー連鎖死（シグナル連鎖死）」の完全回避
`deploy_listener.py` がGitHub Webhookをトリガーに `./ragy restart` を実行しようとした際、`ragy` 内で古いレシーバープロセスを `kill` すると、OSがそのプロセスツリー全体にシグナル（SIGHUP/SIGPIPE）を伝播させ、実行中のデプロイスクリプト自体も巻き添えで死んでしまう不具合が発生しました。

これを回避するため、AIと突き合わせて以下の3重の対策を実装し、自動デプロイパイプラインの完全自律化を実現しました。

1. **Webhookレシーバーの早期自主終了**:
   レシーバー（FastAPI）自身がGitHubに 200 OK レスポンスを返した直後、`os._exit(0)` で自ら正常終了してポートを速やかに解放する。
2. **自動デプロイフラグによるキル処理スキップ**:
   デプロイスクリプトから再起動を実行する際、環境変数 `AUTO_DEPLOY=1` を指定して `ragy` スクリプトを動かし、Webhookレシーバーに対する重複キル命令をバイパスする。
3. **PIDの消滅待機ループ**:
   `ragy` スクリプトの停止処理側に、プロセスがOSレベルで完全に消滅するまで最大10秒間待機するループを導入し、ポートの競合（Address already in use）による再起動失敗を完全に防ぐ。

---

## 9. 今後の展望（v2.0）：RedisキューとDifyワークフローによるSelf-RAGの自律分散化

v1.1.0の密結合（RAGの停止がエージェントのクラッシュに直結する課題）を解決し、さらにRAGの回答精度を極限まで高めるため、以下のロードマップでv2.0へのアップデートを計画しています。

### ① Redis Queueを挟んだ非同期・イベント駆動化
AiderやContinueからMCPサーバーへのリクエストを直接Dify APIへ同期送信するのではなく、一度Redisのキュー（Redis Streams）にタスクとしてスタックします。バックグラウンドのワーカーが順次処理を行うことで、Difyコンテナの再起動時や一時的なネットワークの瞬断が発生しても、エディタ側のエージェントがタイムアウトで道連れクラッシュするのを防ぎます。

### ② Difyワークフローによる「Self-RAG」自己反省ループの実装
RAGの精度向上のため、LangGraphなどの重厚なフレームワークをフルスクラッチで導入するのではなく、Difyの「ワークフロー機能」を活用します。
Difyのワークフロー上で以下のようなループを定義し、MCPサーバーからはそのワークフローAPIを1つ叩くだけにする構成です。

1. **クエリ再生成**: ユーザーの質問から、検索に最適なキーワードや文脈をLLMが抽出・クエリ化する。
2. **ハイブリッド検索**: ナレッジ（Weaviate）から検索を実行する.
3. **Evaluatorによる自己評価**: 取得したドキュメントが質問に対して十分な情報を含んでいるかを別のLLM（Evaluator）が判定する。
4. **自己修正ループ**: 情報が不足していると判定された場合、不足している要素を埋めるための新しい検索クエリを生成し、再検索を実行する（最大N回）。

このように、ドキュメントのパースやベクトルDB管理などのインフラ部分はDify of 恩恵を受けつつ、外側にRedisキュー、内側にDifyのループ処理を組み合わせる「ハイブリッド構成」を採用することで、開発コストを最小限に抑えながら極めて頑健で賢いローカルAI環境を実現していきます。

---

## 10. まとめ

本構築により、以下の開発サイクルがローカル完結で実現できました。

1. `docs/` にドキュメントを置くだけで自動的にベクトルインデックス化（RAG）される。
2. エディタやCLI（Aider/Continue）でコードを書く際、自作MCP経由でドキュメントの文脈を考慮した極めて精度の高い提案が（セマンティックキャッシュの高速レスポンス付きで）得られる。
3. 万が一、バックグラウンド同期やデプロイで例外を吐いても、自己修復エージェントが裏で動き、修正PRを自動で生成してくれる。

それぞれのパーツは、Dify APIのパラメータのクセの回避や、独自の類似度計算によるキャッシュシステム、Google Antigravity SDKを用いたインテリジェントな自動修復など、泥臭い工夫が散りばめられています。

ローカル開発環境のさらなる効率化を目指す方の参考になれば幸いです。
