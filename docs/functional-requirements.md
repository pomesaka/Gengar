# 機能要件

`docs/requirements.md` の要求を実現するための具体機能を定義する。**Phase 1 MVP は実装コードを書かない**。成果物は設定ファイル・ダッシュボード JSON・docs のみ。

## 全体構成

```
┌─────────────────────────────────────────┐
│ Claude Code                             │
│ (CLAUDE_CODE_ENABLE_TELEMETRY=1)        │
└──────────────────┬──────────────────────┘
                   │ OTLP (metrics + logs/events)
                   ▼
          ┌───────────────────┐
          │ Prometheus        │ ← metrics 受信 (OTLP native, 2.55+)
          │  (retention 90d)  │
          └─────────┬─────────┘
                   │
          ┌────────┴────────┐
          │ Loki            │ ← events 受信 (OTLP logs)
          │  (retention 30d)│
          └─────────┬───────┘
                    │
                    ▼
             ┌─────────────┐
             │ Grafana     │ ← Gengar ダッシュボード群
             └─────────────┘
```

※ 任意で **OTel Collector** を挟める (フィルタ・再ラベル等が必要な場合)。MVP のデフォルトでは挟まない。

---

## F1. 同梱する設定ファイル

対応要求: R3

### F1.1 Claude Code 側の設定例

- **F1.1.1** `claude-code-settings.example.json` を同梱
- **F1.1.2** 内容は managed settings 形式で、必須環境変数を網羅:
  ```json
  {
    "env": {
      "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
      "OTEL_METRICS_EXPORTER": "otlp",
      "OTEL_LOGS_EXPORTER": "otlp",
      "OTEL_EXPORTER_OTLP_PROTOCOL": "grpc",
      "OTEL_EXPORTER_OTLP_ENDPOINT": "http://127.0.0.1:4317"
    }
  }
  ```
- **F1.1.3** プライバシーに関わる opt-in 設定 (`OTEL_LOG_USER_PROMPTS`, `OTEL_LOG_TOOL_DETAILS`) は既定で無効のまま。有効化する際のコメントを記述
- **F1.1.4** ユーザーは `~/.claude/settings.json` の `env` キー配下にマージして使う

### F1.2 Prometheus 設定

- **F1.2.1** `prometheus/prometheus.yml` を同梱
- **F1.2.2** OTLP receiver を有効化 (Prometheus 2.55+ で native 対応)
- **F1.2.3** `--enable-feature=otlp-write-receiver` 起動フラグを docker-compose で指定
- **F1.2.4** `--storage.tsdb.retention.time=90d` をデフォルト設定
- **F1.2.5** カーディナリティ対策として公式の `OTEL_METRICS_INCLUDE_SESSION_ID=true` をそのまま受ける (個人利用想定)

### F1.3 Loki 設定

- **F1.3.1** `loki/loki-config.yaml` を同梱
- **F1.3.2** OTLP logs receiver を有効化 (Loki 3.x で対応)
- **F1.3.3** retention 30 日 (events は metrics より容量を食うため短め)

### F1.4 (Optional) OTel Collector 設定

- **F1.4.1** `otel-collector/config.yaml` を参考として同梱 (MVP のデフォルト構成では使わない)
- **F1.4.2** 用途: `OTEL_LOG_USER_PROMPTS` を有効化した場合のマスク処理、カスタム属性付与、バックエンド分離など
- **F1.4.3** Quickstart docker-compose では **使用しない**。必要なユーザーだけ有効化できるようプロファイル分岐で同梱

### F1.5 Grafana プロビジョニング

- **F1.5.1** `grafana/provisioning/datasources/prometheus.yml` — Prometheus データソース自動登録
- **F1.5.2** `grafana/provisioning/datasources/loki.yml` — Loki データソース自動登録
- **F1.5.3** `grafana/provisioning/dashboards/loader.yml` — ダッシュボードを `/grafana/dashboards/` からロード

### F1.6 docker-compose

- **F1.6.1** `docker-compose.yml` に以下のサービスを定義
  - `prometheus` — OTLP receiver 有効化
  - `loki` — OTLP logs receiver 有効化
  - `grafana` — 自動プロビジョニング有効
  - (任意プロファイル) `otel-collector`
- **F1.6.2** すべてのサービスは `127.0.0.1` バインド (N1 ローカルファースト)
- **F1.6.3** `.env.example` を同梱し、Grafana admin パスワードなどのカスタマイズ箇所を記述

---

## F2. Grafana ダッシュボード

対応要求: R2

Phase 1 の**核心**。以下のダッシュボードを `grafana/dashboards/` に JSON で同梱する。

### F2.1 Overview (ホーム画面)

- **F2.1.1** 今日の総コスト、トークン、セッション数、active_time (user vs cli 比)
- **F2.1.2** 直近 7 日間のコストトレンド line chart
- **F2.1.3** 最近のセッション TOP10 (コスト降順)
- **F2.1.4** 主要 KPI の数値パネル

**使用メトリクス/イベント**:
- `claude_code.cost.usage` (Counter)
- `claude_code.token.usage` (Counter, type 別)
- `claude_code.session.count` (Counter)
- `claude_code.active_time.total` (Counter, type=user/cli 別)

### F2.2 Cost

- **F2.2.1** 日次 / 週次 / 月次コスト line chart
- **F2.2.2** モデル別 stacked area (model 属性)
- **F2.2.3** query_source 別 (main / subagent / auxiliary)
- **F2.2.4** 高コストセッション TOP10 (session.id 別)

### F2.3 Tools

- **F2.3.1** ツール別呼び出し回数 bar chart (tool_result events から集計)
- **F2.3.2** ツール別失敗率 (`success="false"` / total)
- **F2.3.3** ツール別平均所要時間 (`duration_ms` の avg / p95)
- **F2.3.4** `tool_decision` イベントによる accept / reject 率 (Edit/Write/NotebookEdit 対象)
- **F2.3.5** MCP ツール使用状況 (tool_parameters の mcp_server_name が有効な場合のみ)

**データソース**: Loki の `{service_name="claude-code"} | json | event_name="tool_result"` を LogQL で集計

### F2.4 Sessions

- **F2.4.1** 最近 24 時間のセッション一覧テーブル (session.id, 開始時刻, コスト, トークン, ターン数)
- **F2.4.2** 特定 session.id でフィルタして、そのセッションのイベントタイムラインを表示 (Loki ログビュー)
- **F2.4.3** セッション継続時間分布 (Histogram)

### F2.5 Subagent

- **F2.5.1** `query_source="subagent"` のコスト・トークン集計
- **F2.5.2** `subagent_type` 別 (Task tool の tool_parameters.subagent_type、`OTEL_LOG_TOOL_DETAILS=1` 時のみ)
- **F2.5.3** 親セッション vs subagent のコスト比率

### F2.6 Errors & Retries

- **F2.6.1** `claude_code.api_error` の発生頻度 (time series)
- **F2.6.2** `claude_code.api_retries_exhausted` の発生回数
- **F2.6.3** status_code 別集計
- **F2.6.4** `claude_code.internal_error` の発生状況

### F2.7 Compaction

- **F2.7.1** コンパクション発生頻度 (auto vs manual)
- **F2.7.2** `pre_tokens` / `post_tokens` の比較 (削減量)
- **F2.7.3** コンパクション所要時間分布

### F2.8 Lines of Code / Git Activity

- **F2.8.1** `claude_code.lines_of_code.count` を added / removed 別に表示
- **F2.8.2** `claude_code.commit.count` / `claude_code.pull_request.count` の推移

---

## F3. Quickstart / セットアップ導線

対応要求: R3, N5

### F3.1 README.md (Quickstart)

- **F3.1.1** 5 ステップ以内でダッシュボードが見える
  1. リポジトリを clone
  2. `docker compose up -d`
  3. `claude-code-settings.example.json` の内容を `~/.claude/settings.json` の `env` にマージ
  4. `claude` を起動、何か作業する
  5. `http://localhost:3000` で Grafana にアクセス (既定 admin/admin)
- **F3.1.2** トラブルシューティング節 (ポート競合、OTLP 受信確認の `curl` 例、Grafana datasource テスト)

### F3.2 設定カスタマイズ・ドキュメント

- **F3.2.1** `docs/setup.md` で以下を詳述
  - プライバシー設定: `OTEL_LOG_USER_PROMPTS` / `OTEL_LOG_TOOL_DETAILS` の有効化影響
  - retention 変更方法
  - OTel Collector を有効化する手順 (フィルタリングが必要な場合)
  - Claude Code 側を別マシンで動かす場合のエンドポイント設定
- **F3.2.2** `docs/dashboards.md` で各ダッシュボードの使い方・カスタマイズ方法

---

## F4. post-MVP (Should / v0.2〜)

### F4.1 CLI (v0.2)

- **F4.1.1** `gengar summary --window 7d` 等のサブコマンドを提供
- **F4.1.2** 中身は Prometheus / Loki の HTTP API を叩く薄いラッパー
- **F4.1.3** 出力は人間可読テキスト + `--json`

### F4.2 MCP サーバー (v0.3)

- **F4.2.1** `gengar` MCP サーバー名で提供
- **F4.2.2** ツール: `metrics_summary`, `recent_sessions`, `top_tools`, `top_failing_tools`, `expensive_sessions`, `self_improvement_hints`
- **F4.2.3** CLI と同じラッパーロジックを共有

### F4.3 自己改善ヒント (v0.3 以降)

- **F4.3.1** PromQL / LogQL ベースの閾値ルールを同梱 (失敗率、所要時間異常、コンパクション頻度等)
- **F4.3.2** Claude Code からクエリしたときに即座にアラート形式で返せる

### F4.4 Claude Code plugin (必要時のみ)

- **F4.4.1** 公式 OTel で取れないメトリクスが必要になった場合、Claude Code plugin (`.claude-plugin/` + `hooks/hooks.json`) として配布する選択肢
- **F4.4.2** Phase 1 では**不要**。公式 OTel で十分

---

## 優先度 (Phase 1 MVP)

| 優先度 | 機能要件 |
|--------|---------|
| **Must (v0.1)** | F1.1, F1.2, F1.3, F1.5, F1.6, F2.1, F2.2, F2.3, F2.4, F2.6, F3.1 |
| **Should** | F2.5 (Subagent), F2.7 (Compaction), F2.8 (LoC/Git), F3.2 (setup.md / dashboards.md), F1.4 (Collector 例) |
| **Could (post-MVP)** | F4.1 (CLI), F4.2 (MCP), F4.3 (self-improvement), F4.4 (plugin) |
| **Won't (永久に作らない)** | 自作 collector / daemon / JSONL reader, 自作ストレージ, 自作クエリエンジン, 独自 OTel メトリクス emit |

---

## 開いている論点 (実装前に決める)

- ダッシュボード JSON の作成方法: 手書き vs Jsonnet vs Grafonnet
- Grafana バージョン固定 (11.x で固定推奨)
- Prometheus / Loki のバージョン (OTLP 受信が安定している最新 stable)
- プライバシー有効化時の OTel Collector 推奨設定 (プロンプトのマスク処理ルール)

これらは設定ファイル作成時に実作業で決める。
