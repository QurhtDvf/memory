2025年公開のMemMachine（AIエージェント向けオープンソースメモリレイヤー）を、完全ローカルのLLMバックエンド（llama.cpp）と組み合わせて動作させるデモノートブックです。外部APIキーやDockerは一切不要で、すべてpip/aptでインストールして実行できます。

ユーザー（ノートブック）
       ↓  memmachine-client SDK
MemMachine サーバー（ポート 8765）
  ├─ PostgreSQL + pgvector  → セマンティックメモリ
  └─ Neo4j                  → エピソードメモリ（グラフDB）
       ↓
llama-cpp-python サーバー × 2
  ├─ LLM サーバー（ポート 9081）  Llama-3.2-1B-Instruct Q4_K_M
  └─ Embeddingサーバー（ポート 9082）  nomic-embed-text-v1.5 Q4_K_M

**動作環境
GPU（CUDA）・CPU どちらでも動作（自動判定）
Python 3.12
Debian/Ubuntu ベースの Linux 環境

**実装上の注意点

ポート8080は使用しない — 実行環境によってシステムが予約している場合があるため、MemMachine(8765)・LLM(9081)・Embedding(9082) をすべて別ポートに割り当て
MemMachineのポート指定は環境変数PORTで行う — cfg.ymlのserver.portより環境変数が優先される仕様のため、起動時にPORT=8765を明示的に渡す
llama-cpp-pythonの起動チェックは/v1/modelsで行う — /healthエンドポイントは存在しない
2サーバーは逐次起動 — LLMとEmbeddingを同時起動するとメモリ競合が起きるため、LLM完全起動後にEmbeddingを起動
CPU モードはタイムアウトを延長 — クライアント600秒、推論300秒に設定
