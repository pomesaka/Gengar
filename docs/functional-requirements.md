# 機能要件

本ドキュメントは `docs/requirements.md` に定義された要求 (R1〜R3, N1〜N6) を実現するための機能要件を洗い出すものである。各機能要件には対応する要求 ID を併記する。

## 全体構成（概念）

```
┌───────────────────────────────┐
│ Claude Code (CLI)             │
│  ├─ hooks  ──────────┐        │
│  └─ ~/.claude/...jsonl        │
└───────────────┬──────┴────────┘
                │ events / files
                ▼
  ┌──────────────────────────────┐
  │ F1. Collector (hook handler /│
  │     JSONL ingester)          │
  └──────────────┬───────────────┘
                 ▼
  ┌──────────────────────────────┐
  │ F2. Storage (events + aggr.) │
  └──────┬───────────────┬───────┘
         │               │
         ▼               ▼
  ┌───────────┐   ┌────────────────┐
  │ F3. API   │   │ F5. MCP / CLI  │
  │   Server  │   │   (Claude 連携) │
  └─────┬─────┘   └────────────────┘
        ▼
  ┌───────────┐
  │ F4. Dash  │
  │  board UI │
  └───────────┘
```

---

## F1. データ収集（Collector）

対応要求: R1.1, R1.2, R1.3, R1.4, N3, N4

### F1.1 Hook イベントハンドラ

- **F1.1.1** 以下の hook イベントを受信し、標準化された JSON スキーマで記録する
  - `SessionStart`, `SessionEnd`
  - `UserPromptSubmit`
  - `PreToolUse`, `PostToolUse`
  - `Stop`, `SubagentStop`
  - `Notification`
  - `PreCompact`
- **F1.1.2** hook は短命なプロセスとして呼び出される前提で、受信イベントをローカルキュー（ファイル追記／Unix ソケット経由の常駐プロセス）へ即座に引き渡し、50ms 以内にプロセスを終了させる
- **F1.1.3** 重い処理（集計、DB 書き込み）は非同期ワーカが担当する
- **F1.1.4** 受信失敗時は hook プロセスを非ゼロ終了させず、Claude Code 本体をブロックしない

### F1.2 JSONL ングェスタ

- **F1.2.1** `~/.claude/projects/**/*.jsonl` を監視し、追記行を差分取り込みする
- **F1.2.2** 各行から以下のフィールドを抽出する
  - `sessionId`, `parentUuid`, `uuid`, `timestamp`
  - `type`（`user` / `assistant` / `summary` など）
  - `message.usage`（`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`）
  - `message.model`
  - `message.content` 内のテキスト／ツール使用／ツール結果
  - `toolUseResult` のエラー有無
- **F1.2.3** 再処理 (R1.4) のために、取り込み済みオフセットをファイルごとに永続化する
- **F1.2.4** ファイル削除・回転（logrotate 的挙動）に対して安全に振る舞う

### F1.3 メトリクス計算

- **F1.3.1** 収集イベントからセッション単位のサマリを計算する
  - 期間（開始〜終了、アクティブ時間）
  - ターン数（ユーザー発話数 / アシスタント応答数）
  - トークン合計（input / output / cache_read / cache_creation）
  - コスト概算（モデル別単価テーブルで積算）
  - ツール呼び出し回数（種類別）
  - ツール失敗率（PostToolUse の `is_error` をカウント）
  - 使用モデル一覧
  - 編集ファイル数、追加／削除行数（Edit / Write の差分サイズから算出。ファイルパス自体の保存可否は論点 5 で別途決定）
  - Bash 実行回数・失敗数（コマンド文字列の保存可否は論点 5 で別途決定）
- **F1.3.2** 時間窓（日次／週次／月次）ごとの集計ビューを生成する
- **F1.3.3** 計算ロジックはイベントストア → 集計テーブルへの純関数として実装し、再実行で冪等に結果が揃うこと (R1.4)

### F1.4 プライバシー／フィルタ

- **F1.4.1** 収集対象は**メタデータのみ**とする。プロンプト／アシスタント応答／ツール結果の**生テキスト本文は一切取り込まない**。本文相当のフィールドは hook ペイロード／JSONL 行から明示的に除外する（R1 / N2 に対応）
- **F1.4.2** 全レコードに `session_id` を必須カラムとして保持する。ユーザーがダッシュボード／CLI から当該 JSONL ファイルのパスを参照できるようにする（R1.5）
- **F1.4.3** 除外ルール: プロジェクトパスの正規表現で、そもそも取り込まないプロジェクトを指定できる
- **F1.4.4** ツール引数・コマンド文字列の扱いは別途定義する（論点 5 参照）

---

## F2. データストレージ

対応要求: R1.4, N1, N3, N5

### F2.1 イベントストア

- **F2.1.1** 受信イベントを追記専用で保存する（ファイル／SQLite いずれか）
- **F2.1.2** スキーマは hook 種別ごとに拡張可能な JSON カラム + 共通メタ（`event_id`, `session_id`, `project_path`, `timestamp`, `source`）
- **F2.1.3** 保存場所は `~/.gengar/events/` をデフォルトとし、設定で変更可能

### F2.2 集計ストア

- **F2.2.1** SQLite（推奨）または DuckDB 等の埋め込み DB を用いる
- **F2.2.2** テーブル: `sessions`, `tool_uses`, `messages`, `daily_aggregates`, `project_aggregates`
- **F2.2.3** マイグレーションスクリプトをリポジトリに同梱する

### F2.3 取り込みオフセット管理

- **F2.3.1** JSONL ごとに `path` / `inode` / `read_offset` / `last_seen_at` を保持
- **F2.3.2** 壊れた行はスキップしてエラーログに記録

---

## F3. API サーバー

対応要求: R2.1, R3.1, R3.2

### F3.1 ローカル HTTP API

- **F3.1.1** `127.0.0.1` 限定バインドで起動する
- **F3.1.2** エンドポイント（最小）
  - `GET /api/sessions?from=&to=&project=` セッション一覧
  - `GET /api/sessions/:id` セッション詳細
  - `GET /api/sessions/:id/events` 当該セッションのイベント列
  - `GET /api/metrics/summary?window=24h|7d|30d` サマリ
  - `GET /api/metrics/tools?window=...` ツール別集計
  - `GET /api/metrics/costs?window=...` コスト推移
  - `POST /api/query` 汎用クエリ（プリセット + 自由 SQL はオプション）
- **F3.1.3** レスポンスは JSON、`ETag` / `Last-Modified` を可能な限り付与

### F3.2 認証

- **F3.2.1** 初期はローカルのみのためトークン無し。将来拡張として API トークン方式を検討する

---

## F4. ダッシュボード UI

対応要求: R2.1, R2.2, R2.3, R2.4

### F4.1 画面

- **F4.1.1** ホーム: 直近 24 時間のサマリ（セッション数、総コスト、総トークン、よく使われたツール Top5）
- **F4.1.2** セッション一覧: フィルタ（期間／プロジェクト／モデル）と並び替え
- **F4.1.3** セッション詳細: タイムライン表示（ユーザー発話・アシスタント応答・ツール呼び出しを時系列で）、使用モデル、トークン内訳、コスト
- **F4.1.4** ツール別ビュー: 使用回数・失敗率・平均所要時間、時系列チャート
- **F4.1.5** コストビュー: 日次／週次／月次チャート、プロジェクト別 stacked
- **F4.1.6** プロジェクトビュー: プロジェクトごとのアクティビティヒートマップ

### F4.2 共通機能

- **F4.2.1** 時刻はローカルタイムゾーンで表示
- **F4.2.2** ダークモード対応
- **F4.2.3** 全画面でキーボードショートカット（`/` で検索、`g s` でセッション一覧など）

### F4.3 非機能

- **F4.3.1** 1,000 セッションまでは主要ビューが 2 秒以内に表示される
- **F4.3.2** ダッシュボードはローカル HTTP API のみに依存する（外部 CDN / アナリティクス禁止）

---

## F5. Claude Code からの集計アクセス

対応要求: R3.1, R3.2, R3.3, R3.4

### F5.1 MCP サーバー

- **F5.1.1** `gengar` という MCP サーバー名で提供する
- **F5.1.2** 提供ツール（最小）
  - `metrics_summary(window, project?)` : サマリ取得
  - `top_tools(window, limit=10)` : よく使われたツール
  - `top_failing_tools(window, limit=10)` : 失敗の多いツール
  - `session_report(session_id)` : 特定セッションのレポート
  - `recent_sessions(limit=20, project?)` : 最近のセッション
  - `expensive_sessions(window, limit=10)` : コスト上位
  - `self_improvement_hints(window)` : 推奨事項（後述 F5.3）
- **F5.1.3** 返り値は Claude Code のコンテキストに乗せやすいよう、JSON と整形済みテキストの両方を返せる

### F5.2 CLI

- **F5.2.1** `gengar` コマンドを提供する
  - `gengar summary --window 7d`
  - `gengar sessions list --project <path>`
  - `gengar session show <id>`
  - `gengar tools top --window 24h`
  - `gengar costs --window 30d --by project`
  - `gengar export --format json|csv --out <file>`
- **F5.2.2** 出力は人間可読テキスト（既定）と `--json` オプション
- **F5.2.3** MCP サーバーと同じ集計ロジックを共有すること

### F5.3 自己改善ヒント

- **F5.3.1** ルールベースで以下のようなヒントを生成する
  - 「同一ファイルを N 回連続で Read していた」→ キャッシュ活用の提案
  - 「Bash のリトライが多い」→ 失敗パターンの確認
  - 「特定ツールの平均所要時間が極端に長い」→ 代替ツールの検討
- **F5.3.2** 検出ロジックはプラガブルに追加可能（ルールは Python／TS ファイル単位で定義）

---

## F6. セットアップと設定管理

対応要求: N1, N3, N4, N6

### F6.1 インストール／アンインストール

- **F6.1.1** `gengar install` で以下を自動実行
  - `~/.claude/settings.json`（またはプロジェクト `.claude/settings.json`）への hook 登録
  - `~/.gengar/` 配下のディレクトリ作成
  - 既定設定ファイルの生成
- **F6.1.2** `gengar uninstall` で hook 登録を解除する（データは残す）
- **F6.1.3** インストール対象スコープ（user / project / local）を選択可能

### F6.2 設定ファイル

- **F6.2.1** `~/.gengar/config.toml`（または JSON）で以下を設定
  - 収集モード（F1.4.1）
  - 除外ルール
  - データ保持期間（既定 90 日、`0` で無期限）
  - 集計ウィンドウ
  - モデル単価テーブル（上書き可能）
  - ダッシュボード／API サーバーの待ち受けポート

### F6.3 常駐プロセス

- **F6.3.1** `gengar daemon` で集計ワーカ + API サーバー + ダッシュボードを同一プロセスで起動
- **F6.3.2** `launchd` / `systemd --user` 用のサンプルユニットファイルを同梱
- **F6.3.3** ヘルスチェックエンドポイント `GET /healthz`

### F6.4 データ管理

- **F6.4.1** `gengar prune --older-than 30d` で古いデータを削除
- **F6.4.2** `gengar reindex` で JSONL を再取り込みし集計を再生成
- **F6.4.3** `gengar doctor` で hook 登録・取り込み状況・DB 整合性をチェック

---

## F7. 観測性（Self-observability）

対応要求: N4

- **F7.1** Collector / Daemon 自身のログを `~/.gengar/logs/` に構造化 JSON で出力
- **F7.2** 取り込み失敗・hook エラーはダッシュボードの「System Health」に表示
- **F7.3** メトリクス収集自体の統計（処理したイベント数、遅延）を内部メトリクスとして計測する

---

## 優先度（MVP スコープ案）

| 優先度 | 機能要件 |
|--------|---------|
| Must (MVP) | F1.1, F1.2, F1.3（主要項目のみ）, F2.1, F2.2, F3.1（`/api/sessions`, `/api/metrics/summary`）, F4.1.1, F4.1.2, F4.1.3, F5.2（最低限のサブコマンド）, F6.1, F6.2 |
| Should | F1.4, F3.1 の残りエンドポイント, F4.1.4〜F4.1.6, F5.1（MCP）, F5.3, F6.3 |
| Could | 自由 SQL クエリ, プラグイン機構, Windows 対応, 自己改善ヒントの高度化 |
| Won't (今回) | チーム共有, クラウド同期, 認証基盤 |

---

## 開いている論点

- ストレージの具体選定: SQLite 単一 DB にするか、イベントは append-only の JSONL / Parquet にして集計側で DuckDB を使うか（先行事例: OpenUsage は SQLite、Liam ERD のブログは DuckDB — `docs/prior-art.md` §7, §8 参照）
- ダッシュボードのフロントエンド技術: Next.js か SvelteKit か、あるいは静的 HTML + サーバーサイドレンダリング
- CLI / MCP サーバーの実装言語: Python（`anthropic` MCP SDK との相性）か TypeScript か
- モデル単価テーブルの更新方法: 同梱 vs オンライン取得（N1 との整合、ccusage のテーブルを参照 — `docs/prior-art.md` §2）
- hook 登録の冪等性確保（既存 `settings.json` への統合方針、disler/multi-agent-observability を参照 — `docs/prior-art.md` §4）
- 公式 OTEL エクスポートとの関係: 補完入力として取り込むか、独自収集のみにするか（`docs/prior-art.md` §1, §6 参照）
