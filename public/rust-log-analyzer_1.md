---
title: 【個人開発】Rust×Python×Redisで挑む、イベント駆動型ログ管理アプリ　〜設計と環境構築編〜
tags:
  - Python
  - Rust
  - 初心者
  - 個人開発
private: false
updated_at: '2025-11-26T00:46:35+09:00'
id: 36bc7fe31084aae48eea
organization_url_name: null
slide: false
ignorePublish: false
---

## 1. はじめに

### プロジェクト概要

今回が初めてのアプリ開発ということもあり、Geminiにたくさん相談に乗ってもらいました。
そこでこれから開発を進めようと思っているのは、Rustとpythonを用いたログ管理アプリです。
Rustを用いた高パフォーマンスなCUIログ解析ツールの開発を目標に進めていきます。
ログの対象は、複数のコンテナからのログ出力を集計、リアルタイム監視することを目的としています。
このアプリ開発を通して、Rustの非同期処理、安全性、そしてイベント駆動のアーキテクチャに関する理解を深めたいと考えています。

このアプリの開発は収益化や誰か特定のユーザーを考えているのではなく、自身の学習とポートフォリオのために作成しているものです。
（途中で頓挫する可能性が大いにあります...）

### 今回のゴール

開発環境のコード化、いわゆる、**Environment as Code**と、Rust-Python間のメッセージング疎通までを目標にしていきます。

## 2. アーキテクチャ設計
今回のアーキテクチャの全体像です（geminiに作ってもらいました）。
![rust-log-analyzer_architecture.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371529/9c902a21-c743-46ab-98f2-c66adbd8abc2.jpeg)
▲図: 今回設計したアーキテクチャの全体像。Rust（Collector）がログを収集・表示し、Python（Analyzer）が裏側で集計処理を行うイベント駆動型の構成です。
RustによるCUIによるログ管理のサービスと、Pythonによるログ分析のサービスの間にメッセージ基盤を導入することでそれぞれのサービスを疎結合にすることを目的としています。

### 技術選定の理由

**Rust**:
メモリ安全性と非同期I/O性能のため選択しました。特に今回は分散サービスを目指しているため非同期処理に対して一定以上の性能があるという理由もあります。
コンテナとの相性も含めてGoでも良かったのですが、自分の技術スタック的にGoは1から学習を進めていく必要があったので今回はRustを選択しました。

**Python**:
Pandasなど豊富なデータ分析エコシステムがあり、かつ筆者がPythonの経験があるという理由からバックエンドにおける集計用に選択しました。
ログ分析にはPolarsを選択しました。PolarsはRustで開発されており、動作が高速でありリアルタイムに処理するという今回の要件に適しているため、PandasではなくPolarsを選択しました。

**Redis**:
RustとPythonを疎結合に保つためのメッセージブローカーとして使用します。それぞれのコンポーネントに障害が発生しても全体が止まらない「耐障害性」を意識してこちらを採用しました。
Redis以外にもいくつか選択肢がありましたが、最も手軽に試せるという理由から選択しました。
また、Rust、Pythonともにライブラリが成熟しており、実装が簡単だという理由でも選択しています。

## 3. 開発環境へのこだわり
今回のアプリ開発では、RustやPython、Redisを使用することで環境構築が複雑になりがちです。
開発にローカル環境を使用して環境を汚したくないと考えていました。

**解決策1： VS Code　Dev Containers**:
`docker compose`コマンドを用いて、リポジトリを開くだけで全環境が立ち上がる、「Environment as Code」を実現しています。
ただ現在は、ビルドやコンテナイメージの最適化などについてまでは行えていません。

**解決策2: Rust製のPythonツール `uv`**:
pythonのパッケージ管理に、話題のRust製ツール`uv`を採用しました。
コンテナビルドと依存解決の高速化と、RustプロジェクトなのでツールチェーンもRust製で統一し、エコシステムへの感度を示すために採用しました。

## 4. ディレクトリ構成
本プロジェクトのディレクトリ構成です。
```bash
rust-log-analyzer
├── .devcontainer
│   ├── compose.yaml
│   └── devcontainer.json
├── analyzer
│   ├── main.py
│   ├── pyproject.toml
│   ├── README.md
│   └── uv.lock
└── collector
    ├── Cargo.lock
    ├── Cargo.toml
    ├── src
    │   └── main.rs
    └── target
```
今回はモノレポで作成を進めていきます。

- 個人開発における開発効率を優先
- JSONスキーマなどのインターフェース定義の共有しやすさ
- 将来的にCIパイプラインでパスフィルタリングを行うことで、デプロイの独立性を確保
以上の理由から、モノレポによる開発で十分であると判断しました。

## 5. 実装
今回実際に実装したコードを紹介します。
```rust:collector/src/main.rs
use anyhow::Result;
use futures_util::StreamExt;

#[tokio::main]
async fn main() -> Result<()> {
    println!("🦀 Rust Log Collector started...");

    // Docker内の 'redis' ホストへ接続
    let client = redis::Client::open("redis://redis/")?;
    let mut con = client.get_async_pubsub().await?;

    // 'logs.ingest' チャンネルを購読
    con.subscribe("logs.ingest").await?;
    println!("Listening on channel: 'logs.ingest'...");

    // ストリームとしてメッセージを処理
    let mut stream = con.on_message();
    
    while let Some(msg) = stream.next().await {
        let payload: String = msg.get_payload()?;
        println!("Received: {}", payload);
        
        // ここに将来、TUIへの描画処理が入る
    }

    Ok(())
}
```
Rust側では非同期ストリームとしてメッセージを受け取っています。
StreamExtがあることで`Stream`型の`next`が使用できるようになります。これによってストリーム内の次のメッセージを取り出すような処理が実現できます。
`tokio`を用いることで、非同期処理を実現していることがわかると思います。
現在はAnalyzerから送られてきた情報をそのまま表示しているだけですが、本来はRust側からJSONを送信し、その結果をAnalyzerで処理し、処理結果を受け取るという手順になります。
受信と送信を非同期で行う必要があるので少し注意したいところです。

```python:analyzer/main.py
import redis
import json
import time
import datetime

def main():
    client = redis.Redis(host='redis', port=6379, db=0)
    
    print("🚀 Python Log Publisher started...")
    
    while True:
        # ダミーログデータ
        log_entry = {
            "timestamp": datetime.datetime.now().isoformat(),
            "level": "INFO",
            "service": "auth-service",
            "message": "User login successful"
        }
        
        # JSONに変換して'logs.ingest'チャンネルに送信
        message = json.dumps(log_entry)
        client.publish('logs.ingest', message)
        
        print(f"Send: {message}")
        time.sleep(1)

if __name__ == "__main__":
    main()
```
python側では、ダミーログをRedisに投げ続ける、シンプルなスクリプトを作成しました。
こちらも実際には受信と送信を非同期に行う必要があるのでもうすこしコードが複雑になります。

実際に動作している様子がこちらです。
左側がRustのログ受信画面、右側がPythonのログ送信画面です。
![Adobe Express GIF 2025-11-26.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371529/9aaa6f77-0f7f-45f4-b70f-cffad7ea6e26.gif)
それぞれのコンポーネントが通信していることがわかります。
これで、今回の目標は完了しました。

## 6. まとめと今後の展望

今回はアプリ開発のための「土台」と、基礎的な機能の確認を行いました。
次回はRust側での「JSONパースと構造化」および「TUI（CUI）の構築」に進む予定です。
今後も適宜開発記録として残していく予定ですので、応援よろしくお願いします🙇
