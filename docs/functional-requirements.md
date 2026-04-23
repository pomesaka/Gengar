# 機能要件

`docs/requirements.md` の要求 (R1〜R4, N1〜N7) を実現するための機能を定義する。**Phase 1 MVP は collector のみ**とし、UI / ストレージ / クエリエンジン / API サーバーは自作せず外部ツール (OTel backend、Grafana 等) に委譲する。

## 全体構成

```
┌──────────────────────────────────────────┐
│ Claude Code                              │
│  ├─ hook ───▶ ~/.gengar/queue/*.ndjson   │ (短命 hook は追記のみ)
│  └─ ~/.claude/projects/**.jsonl          │
└─────────────────────┬────────────────────┘
                      ▼
       ┌──────────────────────────────┐
       │ gengar daemon (単一ワーカー) │
       │  - hook queue tail           │
       │  - JSONL 差分読み            │
       │  - 計算 (カウンタ/ヒスト更新)│
       │  - OTel SDK: OTLP export    │
       └──────────────┬───────────────┘
                      │ OTLP (gRPC or HTTP)
                      ▼
          ┌────────────────────────────┐
          │ OTel backend (user choice) │
          │  - Prometheus (default 例) │
          │  - SigNoz / VictoriaMetrics│
          │  - その他 OTLP 受信可      │
          └──────────────┬─────────────┘
                         ▼
                     [Grafana 等]
```

---

## F1. データ収集 (Collector)

対応要求: R1, N3, N4, N7

### F1.1 Hook イベントハンドラ

- **F1.1.1** 以下の hook イベントを受信し、標準化された JSON (NDJSON) で `~/.gengar/queue/` 配下に**追記のみ**を行う
  - `SessionStart`, `SessionEnd`
  - `UserPromptSubmit`
  - `PreToolUse`, `PostToolUse`
  - `Stop`, `SubagentStop`
  - `Notification`
  - `PreCompact`
- **F1.1.2** hook は短命プロセスとして呼び出される前提で、**追記のみ**を行い 50ms 以内に終了する。OTel emit や集計は行わない。
- **F1.1.3** 受信失敗時は hook プロセスを非ゼロ終了させず、Claude Code 本体をブロックしない (N4)
- **F1.1.4** hook ペイロードから**本文相当のフィールド** (prompt, response, tool_input の raw テキスト等) は**追記前に除去**する (N2)

### F1.2 JSONL ングェスタ

- **F1.2.1** `~/.claude/projects/**/*.jsonl` を監視し、追記行を差分取り込みする
- **F1.2.2** 各行から以下の**メタデータのみ**を抽出する (本文は抽出しない):
  - `sessionId`, `uuid`, `timestamp`
  - `type`（`user` / `assistant` / `summary` 等）
  - `message.usage`（`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`）
  - `message.model`
  - ツール呼び出し (`tool_use`) の **ツール名**のみ。引数・結果本文は抽出しない
  - **Bash ツールの場合のみ**、`tool_input.command` から**第一トークン (外部プログラム名)** を抽出。`&&` / `;` / `|` 連結時は各プログラム名を個別記録 (R1.6)
  - `toolUseResult` の `is_error` 真偽値のみ
- **F1.2.3** 取り込み済みオフセットをファイルごとに永続化 (`~/.gengar/state/offsets.json`) し、再起動後も差分取り込みを続行できる
- **F1.2.4** ファイル削除・回転 (logrotate 的挙動) に対して安全に振る舞う (inode 変化で検知)
- **F1.2.5** 壊れた JSONL 行はスキップしてエラーログに記録する

### F1.3 メトリクス計算

- **F1.3.1** 収集イベントから以下のメトリクスを計算・更新する:
  - セッション期間 (SessionStart〜SessionEnd の差)
  - ターン数 (user/assistant メッセージ数)
  - トークン合計 (input / output / cache_read / cache_creation)
  - ツール呼び出し回数（種類別 / `is_error` 別）
  - 使用モデル別の集計
  - Bash 内の外部プログラム呼び出し回数（プログラム名別 / `is_error` 別）
- **F1.3.2** 計算ロジックは**冪等** (R1.4)。同じイベントを再処理しても結果が変わらない
- **F1.3.3** コスト計算は **Phase 1 では行わない**。トークン数のみを emit し、単価換算は backend / dashboard 側 (Grafana の expression など) に委ねる

> Phase 1 は「いつ・何をした・いくらトークン使った」の核に徹する。コストは単価情報が別管理になるため、backend で算出する方がメンテ性が高い。

### F1.4 プライバシー／フィルタ

- **F1.4.1** 収集対象は**メタデータのみ**。プロンプト／応答／ツール結果の**生テキストは一切扱わない** (R1, N2)
- **F1.4.2** 全メトリクスに `session_id` ラベルを付与する
- **F1.4.3** 除外ルール: プロジェクトパスの正規表現で、そもそも取り込まないプロジェクトを指定できる
- **F1.4.4** ツール引数は保存・emit しない。**例外: Bash の第一トークン (プログラム名)** のみラベル化して emit する

---

## F2. OTel メトリクスエクスポート

対応要求: R1 (OTel 経由), R2.1

### F2.1 OTel SDK 採用

- **F2.1.1** daemon は OpenTelemetry SDK を用いてメトリクスを emit する
- **F2.1.2** エクスポーターは **OTLP (gRPC または HTTP)** を基本とする
- **F2.1.3** 任意で **Prometheus exposition (`/metrics`)** も利用可能にする (ユーザーが Prometheus scrape を選んだ場合向け)

### F2.2 メトリクス命名規則

- **F2.2.1** すべてのメトリクスは `gengar.claude_code.` プレフィックスで namespace する
- **F2.2.2** 公式 OTel (`claude_code.*`) と**衝突しない**。並立利用が可能
- **F2.2.3** ラベル (attribute) は `session_id`, `project_path`, `model`, `tool_name`, `bash_program`, `is_error` など固定セットから選ぶ

### F2.3 提供メトリクス一覧 (MVP)

| メトリクス名 | 型 | 説明 | 主なラベル |
|---|---|---|---|
| `gengar.claude_code.hook.events` | Counter | hook イベント受信数 | `hook_type`, `session_id`, `project_path` |
| `gengar.claude_code.tool_use.total` | Counter | ツール呼び出し数 | `tool_name`, `is_error`, `session_id` |
| `gengar.claude_code.tool_use.duration_ms` | Histogram | ツール所要時間 | `tool_name`, `session_id` |
| `gengar.claude_code.bash.program_calls` | Counter | Bash 内プログラム呼び出し数 | `program`, `is_error`, `session_id` |
| `gengar.claude_code.session.tokens` | Counter | トークン消費量 | `session_id`, `model`, `token_type` |
| `gengar.claude_code.session.turns` | Counter | ターン数 | `session_id`, `role` |
| `gengar.claude_code.session.duration_seconds` | Gauge / Histogram | セッション継続時間 | `session_id`, `project_path` |
| `gengar.claude_code.session.active` | Gauge | アクティブセッション数 | `project_path` |

### F2.4 エクスポート先の設定

- **F2.4.1** OTLP endpoint は `config.toml` で指定。既定は `http://127.0.0.1:4317` (gRPC) または `http://127.0.0.1:4318` (HTTP)
- **F2.4.2** OTel Collector 経由 / backend 直接送信の**どちらも動作**する (Prometheus 2.55+ / SigNoz / 各種バックエンドは OTLP を直接受けられる)
- **F2.4.3** backend は**ユーザーが用意する**前提。Gengar がバックエンドを起動・管理することはない

### F2.5 同梱する設定例

- **F2.5.1** `examples/prometheus.yml` (Prometheus 単体、OTLP 受信設定を含む)
- **F2.5.2** `examples/otel-collector-config.yaml` (任意で Collector を挟む場合)
- **F2.5.3** `examples/grafana-dashboard.json` (Gengar メトリクス向けの Grafana dashboard テンプレ、主要ビュー入り)
- **F2.5.4** `examples/docker-compose.yml` (お試し用の Prometheus + Grafana 一式、あくまで参考)

---

## F3. セットアップと設定管理

対応要求: N1, N4, N6

### F3.1 インストール／アンインストール

- **F3.1.1** `gengar install` で以下を自動実行する
  - `~/.claude/settings.json`（またはプロジェクト `.claude/settings.json`）への hook 登録（冪等）
  - `~/.gengar/` 配下のディレクトリ (queue, state, logs) 作成
  - 既定 `config.toml` の生成
- **F3.1.2** `gengar uninstall` で hook 登録を解除する（データは残す）
- **F3.1.3** インストール対象スコープ（user / project / local）を `--scope` で選択可能

### F3.2 daemon 起動

- **F3.2.1** `gengar daemon` で hook queue tail + JSONL 差分読み + OTel emit を行う単一ワーカーを起動
- **F3.2.2** `launchd` / `systemd --user` 用のサンプルユニットファイルを同梱
- **F3.2.3** daemon は inbound ポートを持たない (OTLP は outbound push)。Prometheus exposition を選んだ場合のみ `127.0.0.1:<port>` で `/metrics` を出す

### F3.3 設定ファイル

- **F3.3.1** `~/.gengar/config.toml` で以下を設定
  - OTLP endpoint / protocol (grpc / http)
  - 除外ルール (プロジェクトパス正規表現)
  - `/metrics` exposition を有効にするか (既定 off)
- **F3.3.2** 保持期間・ロールアップ・単価テーブルなどは Gengar の設定にない (backend 側の責務)

### F3.4 ヘルスチェック・診断

- **F3.4.1** `gengar doctor` で以下を確認
  - hook が `settings.json` に登録されているか
  - queue に溜まっているイベント数
  - JSONL 差分取り込みの進捗
  - OTLP endpoint への疎通

---

## F4. 観測性 (Self-observability)

対応要求: N4

- **F4.1** daemon / hook ラッパー自身のログを `~/.gengar/logs/` に構造化 JSON で出力
- **F4.2** daemon 自身の内部メトリクス (処理イベント数、遅延、emit 失敗数) を `gengar.self.*` 名前空間で emit する
- **F4.3** エラーは standard OTel logs/events で backend にも送る (任意)

---

## F5. post-MVP (Should / v0.2〜)

要求 R3 / R4.2 / R4.3 に対応。Phase 1 MVP (v0.1) には含めない。

### F5.1 CLI (v0.2 予定)

- **F5.1.1** `gengar summary --window 7d` 等のサブコマンドを提供
- **F5.1.2** 中身は backend の query API (PromQL など) を事前定義クエリで叩く**薄いラッパー**。自作の集計ロジックは持たない
- **F5.1.3** 出力は人間可読テキスト（既定）と `--json`

### F5.2 MCP サーバー (v0.3 予定)

- **F5.2.1** `gengar` という MCP サーバー名で提供
- **F5.2.2** ツール: `metrics_summary`, `recent_sessions`, `top_tools`, `top_failing_tools`, `expensive_sessions`, `self_improvement_hints` など
- **F5.2.3** CLI と同じ PromQL ラッパーを共有

### F5.3 個別削除・バックフィル (post-MVP)

- **F5.3.1** `gengar delete session <id>` / `gengar delete project <path>` で、backend の delete API (例: Prometheus `/api/v1/admin/tsdb/delete_series`) をラップ
- **F5.3.2** `gengar backfill --since <date>` で過去 JSONL を OpenMetrics text に変換。ユーザーが `promtool tsdb create-blocks-from openmetrics` 等で取り込む

### F5.4 自己改善ヒント (post-MVP)

- **F5.4.1** PromQL ベースのルールを複数同梱
  - 特定ツールの失敗率が高い
  - 特定ツールの平均所要時間が極端に長い
  - トークン消費がセッション平均の N 倍
  - ターン数が極端に多い / compact 頻度が高い
- **F5.4.2** 引数依存のルール (「同一ファイル繰り返し Read」等) は Phase 1 ではサポートしない (Phase 2 以降)

---

## 優先度 (Phase 1 MVP スコープ)

| 優先度 | 機能要件 |
|--------|---------|
| **Must (v0.1 MVP)** | F1.1, F1.2, F1.3 (コスト除く), F1.4, F2.1, F2.2, F2.3, F2.4, F2.5 (prometheus.yml + grafana-dashboard.json), F3.1, F3.2, F3.3, F4.1 |
| **Should (v0.2)** | F5.1 (CLI) |
| **Should (v0.3)** | F5.2 (MCP) |
| **Could (post-MVP)** | F5.3, F5.4, F4.2, F4.3, F2.5 の残り (otel-collector 例, docker-compose 例), `gengar doctor` の拡張 |
| **Won't (Phase 1 永久に作らない)** | 自作ダッシュボード UI, 自作 API サーバー, 自作ストレージ, 自作クエリエンジン, 保持期間・ロールアップ機能 |

---

## 開いている論点 (実装フェーズで決める)

- **実装言語**: Python / TypeScript / Rust のいずれか。OTel SDK の成熟度と配布容易性で選ぶ
- **OTel SDK**: 選んだ言語の公式 SDK
- **hook queue の形式**: NDJSON ファイル / Unix socket / SQLite WAL のいずれか (N3 / N4 の観点で選定)
- **JSONL 差分読みの実装**: inotify / FSEvents / ポーリング (可搬性で選ぶ)
- **hook 登録の冪等性**: 既存 `settings.json` のマージ戦略 (disler/multi-agent-observability 参照 — `docs/prior-art.md` §4)

これらは要件レベルでは未決のまま進め、実装計画フェーズで確定する。
