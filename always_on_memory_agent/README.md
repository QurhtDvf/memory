# 🧠 Always-On Memory Agent — llama.cpp 版

> [GoogleCloudPlatform/generative-ai の always-on-memory-agent](https://github.com/GoogleCloudPlatform/generative-ai/tree/main/gemini/agents/always-on-memory-agent) を、**llama-cpp-python + Qwen2.5 GGUF** で動かせるよう改変したノートブックです。  
> **Ollama 不要・pip のみ**でローカル LLM を使った3層メモリエージェントを構築できます。

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/YOUR_USERNAME/YOUR_REPO/blob/main/always_on_memory_agent_llamacpp.ipynb)

---

## ✨ 特徴

- **Ollama 不要** — `pip install` だけで完結
- **HuggingFace から直接 GGUF を取得** — モデル管理がシンプル
- **OpenAI 互換 API** — llama-cpp-python が port 8000 でサーブ
- **3層メモリアーキテクチャ** — 生記憶・統合記憶・高次洞察を階層的に管理
- **CPU でも動作** — Colab 無料枠（CPU ランタイム）でそのまま実行可能
- **GPU オプション** — T4 / Apple Silicon Metal / NVIDIA CUDA に対応

---

## 🆚 Ollama 版との違い

| 項目 | Ollama 版 | **この llama.cpp 版** |
|------|-----------|-----------------------|
| バックエンド | Ollama (Go + llama.cpp) | **llama-cpp-python (純Python バインディング)** |
| インストール | curl スクリプト | **pip のみ** |
| モデル取得 | `ollama pull` | **HuggingFace から直接 GGUF** |
| API 互換 | Ollama API | **OpenAI 互換 API (port 8000)** |
| CPU 最適化 | Ollama 任せ | **スレッド数・量子化を細かく制御** |
| 依存 | Ollama バイナリ必須 | **pip install のみ・Ollama 不要** |

---

## 🏗️ スタック構成

```
┌──────────────────────────────────────────────────────────┐
│              Memory Agent HTTP API (port 8888)            │
│   IngestAgent │ QueryAgent │ ConsolidateAgent (ADK)       │
│                     ↕ LiteLLM                            │
│   llama-cpp-python サーバー  (OpenAI互換 port 8000)       │
│          Qwen2.5-1.5B-Instruct-GGUF (Q4_K_M)             │
│                  ↑ HuggingFace Hub からダウンロード        │
└──────────────────────────────────────────────────────────┘
```

---

## 🧱 3層メモリアーキテクチャ

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 3 : Insight（高次洞察）                               │
│    複数の統合記憶をさらに抽象化した高次の知識                  │
│    例）「全体として○○という傾向がある」                       │
│                        ↑ InsightAgent が生成                │
│  Layer 2 : Consolidation（統合記憶）                         │
│    複数の生記憶をまとめ、関係性を見つけた中間知識              │
│    例）「記憶A・B・Cは○○で繋がっている」                     │
│                        ↑ ConsolidateAgent が生成            │
│  Layer 1 : Raw Memory（生記憶）                              │
│    IngestAgent が取り込んだ元テキスト＋構造化メタデータ        │
└─────────────────────────────────────────────────────────────┘
              ↕  QueryAgent が3層すべてを参照
```

QueryAgent は **L3 → L2 → L1 の順に優先度をつけて**回答を合成します。

---

## 📦 使用モデル

| モデル | サイズ | 推奨環境 | HF リポジトリ |
|--------|--------|----------|--------------|
| `Qwen2.5-1.5B Q4_K_M` | ~1GB | **CPU（推奨・デフォルト）** | `Qwen/Qwen2.5-1.5B-Instruct-GGUF` |
| `Qwen2.5-3B Q4_K_M` | ~2GB | CPU (RAM 8GB以上) | `Qwen/Qwen2.5-3B-Instruct-GGUF` |
| `Qwen2.5-7B Q4_K_M` | ~5GB | T4 GPU / RAM 16GB | `Qwen/Qwen2.5-7B-Instruct-GGUF` |

---

## 🚀 クイックスタート（ローカル環境）

```bash
# 1. 依存パッケージのインストール
pip install "llama-cpp-python[server]" huggingface_hub google-adk litellm aiohttp

# 2. GGUF モデルのダウンロード
huggingface-cli download Qwen/Qwen2.5-1.5B-Instruct-GGUF \
  qwen2.5-1.5b-instruct-q4_k_m.gguf --local-dir ./models

# 3. llama-cpp-python サーバーの起動
MODEL=./models/qwen2.5-1.5b-instruct-q4_k_m.gguf \
CHAT_FORMAT=chatml N_THREADS=8 N_CTX=2048 \
  python -m llama_cpp.server

# 4. Memory Agent の起動（別ターミナル）
LLAMA_API_BASE=http://localhost:8000/v1 \
CONSOLIDATE_INTERVAL=600 INSIGHT_INTERVAL=1800 \
  python agent_llamacpp.py

# 5. テスト: 取り込み → 統合 → 洞察 → 質問
curl -X POST http://localhost:8888/ingest \
  -H 'Content-Type: application/json' \
  -d '{"text": "覚えさせたい情報", "source": "test"}'

curl -X POST 'http://localhost:8888/consolidate?layer=2'  # L1→L2
curl -X POST 'http://localhost:8888/consolidate?layer=3'  # L2→L3
curl 'http://localhost:8888/query?q=何を知っていますか'   # 3層参照
curl 'http://localhost:8888/memories?layer=3'             # L3洞察一覧
```

---

## 📡 API リファレンス

| エンドポイント | メソッド | 説明 |
|---|---|---|
| `/status` | GET | 3層別の件数統計を返す |
| `/memories` | GET | 全3層の記憶を返す。`?layer=1/2/3` で層を絞れる |
| `/ingest` | POST | L1に生記憶を追加 `{"text": "...", "source": "..."}` |
| `/query?q=...` | GET | L3→L2→L1 優先で3層を参照して回答 |
| `/consolidate` | POST | `?layer=2` でL1→L2のみ、`?layer=3` でL2→L3のみ、省略で両方 |
| `/delete` | POST | L1記憶を削除 `{"memory_id": 1}` |
| `/clear` | POST | 全層リセット |

---

## ⚡ GPU アクセラレーション

### Apple Silicon (M1/M2/M3) — Metal
```bash
CMAKE_ARGS="-DGGML_METAL=on" pip install --upgrade --force-reinstall llama-cpp-python[server]
# → N_GPU_LAYERS=99 でほぼ全レイヤーをGPUで処理
```

### NVIDIA GPU — CUDA
```bash
CMAKE_ARGS="-DGGML_CUDA=on" pip install --upgrade --force-reinstall llama-cpp-python[server]
# → N_GPU_LAYERS=32 (1.5B) ～ 99 (全レイヤー) を設定
```

### Google Colab T4 GPU
ノートブック内の「⑪ GPU アクセラレーション」セルのコメントを外して実行してください。

---

## 🔧 環境変数

| 変数 | デフォルト | 説明 |
|------|-----------|------|
| `LLAMA_API_BASE` | `http://localhost:8000/v1` | llama-cpp-python サーバーのエンドポイント |
| `MEMORY_DB_PATH` | `memory.db` | SQLite データベースのパス |
| `CONSOLIDATE_INTERVAL` | `600` | L1→L2 自動統合の間隔（秒） |
| `INSIGHT_INTERVAL` | `1800` | L2→L3 自動洞察抽出の間隔（秒） |
| `API_PORT` | `8888` | Memory Agent HTTP API のポート |

---

## 📚 依存ライブラリ

- [llama-cpp-python](https://github.com/abetlen/llama-cpp-python) — llama.cpp の Python バインディング
- [Google ADK](https://github.com/google/adk-python) — エージェント構築フレームワーク
- [LiteLLM](https://github.com/BerriAI/litellm) — LLM API の統一インターフェース
- [HuggingFace Hub](https://huggingface.co/docs/huggingface_hub) — モデルのダウンロード
- [aiohttp](https://docs.aiohttp.org/) — 非同期 HTTP サーバー

---

## 📝 ライセンス

元の [GoogleCloudPlatform/generative-ai](https://github.com/GoogleCloudPlatform/generative-ai) のライセンスに準拠します。
