# MemMachine × llama.cpp デモ（ローカルLLM・Docker不使用）

## 概要

MemMachine（AIエージェント向けオープンソースメモリレイヤー）を、完全ローカルのLLMバックエンド（llama.cpp）と組み合わせて動作させるデモノートブックです。外部APIキーやDockerは一切不要で、すべてpip/aptでインストールして実行できます。

## システム構成

```
ユーザー（ノートブック）
       ↓  memmachine-client SDK
MemMachine サーバー（ポート 8765）
  ├─ PostgreSQL + pgvector  → セマンティックメモリ
  └─ Neo4j                  → エピソードメモリ（グラフDB）
       ↓
llama-cpp-python サーバー × 2
  ├─ LLM サーバー（ポート 9081）  Llama-3.2-1B-Instruct Q4_K_M
  └─ Embedding サーバー（ポート 9082）  nomic-embed-text-v1.5 Q4_K_M
```

## 動作環境

- GPU（CUDA）・CPU どちらでも動作（自動判定）
- Python 3.12
- Debian/Ubuntu ベースの Linux 環境

## デモ内容

| Step | 内容 |
|------|------|
| 1 | GPU/CUDA バージョン自動判定・ポート設定 |
| 2 | llama-cpp-python（CUDA プリビルドホイール）・MemMachine 等のインストール |
| 3 | Neo4j インストール・起動 |
| 4 | PostgreSQL + pgvector ビルド・インストール |
| 5 | GGUF モデルを Hugging Face からダウンロード |
| 6 | llama-cpp-python サーバーを2ポートで逐次起動 |
| 7 | MemMachine 設定ファイル（cfg.yml）生成 |
| 8 | MemMachine サーバー起動 |
| 9 | llama.cpp サーバー単体テスト（推論・埋め込み） |
| 10 | メモリの保存（ユーザー属性・旅行履歴など） |
| 11 | メモリ検索テスト |
| 12 | メモリをプロンプトに注入した会話エージェントのデモ |
| 13 | 複数ユーザー間のメモリ分離デモ |
| 14 | クリーンアップ |

## 使用モデル

| 用途 | モデル | サイズ |
|------|--------|--------|
| LLM | Llama-3.2-1B-Instruct-Q4_K_M | 約 0.8 GB |
| Embedding | nomic-embed-text-v1.5-Q4_K_M | 約 80 MB |

## メモリの共有・分離の仕組み

MemMachine はデフォルトでユーザー間のメモリを**共有しません**。メモリは以下の4つのキーの組み合わせで完全に分離されます。

```python
project.memory(
    group_id="default",      # グループ（部署・チームなど）
    agent_id="travel_agent", # エージェントの種類
    user_id="alice",         # ユーザー識別子（異なれば別メモリ）
    session_id="session_001" # セッション識別子
)
```

| キー | 役割 |
|------|------|
| `user_id` | ユーザーごとの分離（最も重要） |
| `agent_id` | 同じユーザーでも用途別に分離 |
| `group_id` | 組織・チーム単位の分離 |
| `session_id` | セッション単位の分離 |

意図的に複数ユーザーでメモリを共有したい場合は、`user_id`を同じ値にすれば共有できます。例えばチームの共有ナレッジベースとして使う場合は`user_id="team_shared"`のように固定します。

## 実装上の注意点

### ポート 8080 は使用しない
実行環境によってシステムが 8080 を予約している場合があります。本デモでは MemMachine(8765)・LLM(9081)・Embedding(9082) をすべて別ポートに割り当てています。

### MemMachine のポート指定は環境変数 `PORT` で行う
`cfg.yml` の `server.port` より環境変数が優先される仕様のため、起動時に `PORT=8765` を明示的に渡しています。

### llama-cpp-python の起動チェックは `/v1/models` で行う
`/health` エンドポイントは存在しないため、`/v1/models` が 200 を返した時点を起動完了とみなします。

### 2サーバーは逐次起動
LLM と Embedding を同時起動するとメモリ競合が起きるため、LLM の完全起動後に Embedding を起動します。

### CPU モードはタイムアウトを延長
GPU 非搭載環境では推論に時間がかかるため、クライアントタイムアウトを 600 秒、推論タイムアウトを 300 秒に設定しています。
