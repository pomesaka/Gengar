# 要求定義

Claude Code のメトリクス可視化ツール「Gengar」の要求を整理する。

## 1. 背景と目的

Claude Code は `CLAUDE_CODE_ENABLE_TELEMETRY=1` により、メトリクス・イベント・トレース (beta) を OpenTelemetry (OTLP) で公式出力する。トークン使用量、コスト、ツール実行結果、コンパクション、MCP 接続状態などがほぼすべて公式 OTel 経由で取得可能 (`docs/prior-art.md` §1 参照)。

**Gengar は、この公式 OTel データを受け止めて可視化・分析するための設定パッケージとダッシュボード**を提供する。

- **自作の collector / daemon / ストレージ / クエリエンジンは作らない**
- 公式 OTel が提供するデータを前提とし、受け側の OSS (Prometheus / Loki / Grafana) を束ねる
- Gengar 固有の価値は **Grafana ダッシュボードテンプレート** と **ワンショットの docker-compose セットアップ** にある

## 2. ステークホルダー

| ステークホルダー | 関心事 |
|-----------------|--------|
| 開発者（Claude Code ユーザー） | 自分の使い方を把握したい／コストを抑えたい／生産性を上げたい |
| Claude Code 自身 | 過去の実行傾向をクエリし、自己改善ループを回したい (post-MVP) |
| チーム / 組織 | チーム横断の利用傾向・コスト (初期スコープ外) |

## 3. 要求一覧

### R1. 公式 OTel の活用

- **R1.1** Claude Code の公式 OTel (`CLAUDE_CODE_ENABLE_TELEMETRY=1`) を前提とし、**Gengar は受け側に徹する**
- **R1.2** 取り込み対象は公式 OTel の **metrics** (`claude_code.*`) と **events** (`claude_code.user_prompt`, `tool_result`, `api_request`, `api_error`, `compaction` 他)。traces (beta) は Phase 1 では任意
- **R1.3** 公式 OTel を**独自に補完しない** (hook 受信や JSONL 差分読みなどの独自 collector は作らない)
- **R1.4** プライバシー配慮のデフォルトは公式の既定値に従う (`OTEL_LOG_USER_PROMPTS=0`, `OTEL_LOG_TOOL_DETAILS=0`)。ツール詳細 (Bash コマンド、MCP tool 名等) を見たい場合はユーザーが明示的に有効化する

### R2. ダッシュボードでの可視化 (Phase 1 の主役)

- **R2.1** Grafana dashboard JSON を同梱し、インポートするだけで主要ビュー (コスト、トークン、ツール、セッション、エラー、コンパクション) が見られる
- **R2.2** ダッシュボードは公式 OTel の標準メトリクス・イベントのみを参照し、Gengar 独自メトリクスには依存しない
- **R2.3** セッション単位のドリルダウンができる (`session.id` でフィルタ)
- **R2.4** コスト計算は公式 OTel の `claude_code.cost.usage` をそのまま使う (Gengar 側で単価掛け算しない)

### R3. 配布形態 (設定パッケージ)

- **R3.1** Gengar は以下の成果物で構成される
  - `docker-compose.yml` — Prometheus + Loki + Grafana をワンショット起動
  - `prometheus/` 設定 — OTLP receiver 有効化 + retention
  - `loki/` 設定 — OTLP logs receiver
  - `otel-collector/` 設定 — 任意 (Collector を挟みたい場合のテンプレ)
  - `grafana/provisioning/` — データソース・ダッシュボードの自動プロビジョン
  - `grafana/dashboards/*.json` — Gengar オリジナルダッシュボード群
  - `claude-code-settings.example.json` — `~/.claude/settings.json` へ追記する環境変数サンプル
  - `README.md` — Quickstart
- **R3.2** 実装コードは書かない (Phase 1)。すべて設定ファイル・ダッシュボード JSON・docs
- **R3.3** 依存は Docker / docker-compose のみ (ローカルで完結)

### R4. Claude Code からの集計アクセス (post-MVP)

- **R4.1** CLI (PromQL ラッパー) — post-MVP v0.2
- **R4.2** MCP サーバー (PromQL ラッパー) — post-MVP v0.3
- **R4.3** いずれも backend の query API の**薄いラッパー**として実装。自作クエリエンジンは持たない

### R5. データライフサイクル (外部委譲)

- **R5.1** 保持期間は backend (Prometheus / Loki) の retention 設定に委ねる。デフォルトの同梱設定は 90 日
- **R5.2** 個別削除は backend の delete API を使う (CLI ラッパーは post-MVP)
- **R5.3** Gengar 自身はデータを保持しない

## 4. 非機能要求

- **N1. ローカルファースト**: 同梱の docker-compose は `127.0.0.1` バインドのみ。外部送信しない既定構成
- **N2. プライバシー**: 本文・ツール引数の収集有無は公式 OTel の環境変数でユーザーが制御する (`OTEL_LOG_USER_PROMPTS`, `OTEL_LOG_TOOL_DETAILS`, `OTEL_LOG_TOOL_CONTENT`, `OTEL_LOG_RAW_API_BODIES`)。Gengar は既定でどれも有効化しない
- **N3. 軽量性**: Claude Code 側の負荷は公式 OTel の責務。Gengar の docker stack は個人マシンで常時稼働できる規模
- **N4. 拡張性**: ダッシュボードは Grafana で自由に追加・編集できる。Loki / Prometheus のクエリは PromQL / LogQL で直接書ける
- **N5. 可搬性**: Docker が動く環境 (macOS / Linux / WSL) で動作

## 5. スコープ

### 5.1 Phase 1 MVP (v0.1) のスコープ

**含める:**
- 公式 OTel 前提の受け側スタック (Prometheus + Loki + Grafana) を docker-compose で提供
- Gengar オリジナル Grafana ダッシュボード (Overview, Cost, Tools, Sessions, Subagent, Errors, Compaction)
- Claude Code 側の managed-settings 設定例
- Quickstart ドキュメント

**含めない (post-MVP / 永久スコープ外):**
- CLI / MCP (post-MVP v0.2 / v0.3)
- 自作 collector / daemon / JSONL reader (**永久に作らない**)
- 自作ストレージ / クエリエンジン (**永久に作らない**)
- 独自 OTel メトリクスの emit (**永久に作らない**。公式が提供するもので十分)
- Claude Code plugin 経由の hook 補完 (必要になったら検討、現時点では不要)

### 5.2 Phase 1 で明示的に扱わない

- 他エージェント (Codex / Cursor / Aider 等) 対応
- 複数ユーザー／チームでの集計・共有
- クラウド同期・SaaS 化
- 認証・権限管理
- UI の日本語化 (Grafana の日本語化に依存)

### 5.3 Phase 2 以降で検討

- 他エージェント対応
- チーム共有・クラウド同期
- CLI / MCP ラッパー

## 6. 配布・ライセンス

- **Phase 1 は private 開発**。ライセンスは未確定。将来 OSS 公開時は MIT を第一候補
- 同梱する OSS (Prometheus / Loki / Grafana / OTel Collector) はそれぞれのライセンスに従う

## 7. 成功の定義

- `docker compose up` + `~/.claude/settings.json` の追記、この 2 ステップで Grafana に主要ビューが表示される
- ユーザーが Claude Code を 1 日使った後、ダッシュボードでコスト・ツール使用傾向を一目で確認できる
- 体感で Claude Code の動作が遅くなっていない (公式 OTel 既定値内)

数値 SLO は Phase 2 以降で実測後に設定する。
