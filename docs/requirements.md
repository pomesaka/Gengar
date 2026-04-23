# 要求定義

Claude Code のメトリクス収集ツール「Gengar」に対する要求を整理する。

## 1. 背景と目的

Claude Code の利用が進む中で「どのツールがどれだけ使われているか」「セッションあたりのコスト／トークン消費量はどの程度か」「どこで時間がかかっているか」といった情報を体系的に把握する手段が乏しい。本ツールはこれを解決する。

ただし、可視化・クエリ・ストレージを**自作しない**方針を採る。代わりに **OpenTelemetry (OTel)** で標準的にメトリクスを emit し、**Prometheus / Grafana など成熟した observability スタック**に処理を任せる。Gengar は「Claude Code 特有のメトリクス収集 (collector)」に徹する。

## 2. ステークホルダー

| ステークホルダー | 関心事 |
|-----------------|--------|
| 開発者（Claude Code ユーザー） | 自分の使い方を把握したい／コストを抑えたい／生産性を上げたい |
| Claude Code 自身 | 過去の実行傾向をクエリし、より良い作業方針を立てたい（post-MVP） |
| チーム / 組織（将来拡張） | チーム横断の利用傾向・コストを把握したい（初期スコープ外） |

## 3. 要求一覧

### R1. メトリクスの記録 (核)

> **核**: 各 Claude Code イベントについて「**いつ** (timestamp)・**何をしたか** (イベント種別、ツール名)・**いくらトークンを消費したか** (input/output/cache)」を `session_id` ラベル付きの追記レコードとして収集し、**OpenTelemetry (OTel) SDK 経由で OTLP 出力**する。Gengar 自身は保存・集計・クエリを行わず、backend (Prometheus / SigNoz / 他 OTel 互換バックエンド) に委譲する。

- **R1.1** Claude Code の hook 機構（`PreToolUse` / `PostToolUse` / `UserPromptSubmit` / `Stop` / `SubagentStop` / `SessionStart` / `SessionEnd` / `Notification` / `PreCompact` など）を通じてイベントをリアルタイムに取得できること
- **R1.2** セッションごとに `~/.claude/projects/<project>/*.jsonl` に保存されるトランスクリプトを差分取り込みし、hook で取り逃した**メタデータ**（トークン使用量、モデル、ツール呼び出しの種類・回数など）を補完できること。**本文（プロンプト／応答／ツール結果の生テキスト）は取り込まない**
- **R1.3** プロジェクトを問わず、ローカルで発生した全 Claude Code セッションを対象にできること
- **R1.4** 収集処理は **冪等** であり、同じイベントを複数回処理しても結果が変わらないこと
- **R1.5** すべてのメトリクスに **`session_id` ラベル**を付与し、ユーザーが必要に応じて `~/.claude/projects` 配下の JSONL を直接参照して本文を確認できること。Gengar 本体は本文を保持しない
- **R1.6 メトリクスの粒度**: Phase 1 の計測粒度は以下に留める
  - **ツール種別ごとの呼び出し回数・失敗回数**
  - **セッション単位のトークン消費量・コスト・所要時間**
  - **モデル別の内訳**
  - **Bash ツール内で呼ばれた外部プログラム名** (第一トークン。例: `rg`, `find`, `git`, `npm`)。引数・パス・オプションは一切保存しない
  - 上記以外のツール引数 (Edit の `file_path`、Grep の `pattern` 等) は保存しない
- **R1.7 セッションの単位**: 「セッション」の境界は **Claude Code が付与する `session_id` に完全一致**させる。Gengar 側で時間ギャップ分割・論理セッション合成・`/compact` / `/clear` をマーカーとした再定義は行わない

### R2. 可視化 (外部委譲)

- **R2.1** メトリクスは **OTel backend** (Prometheus / SigNoz / VictoriaMetrics / Honeycomb / その他 OTLP 受信可能なバックエンド) を介して可視化する
- **R2.2** Grafana dashboard JSON や `prometheus.yml` などの**設定例をリポジトリに同梱**し、ユーザーがすぐに主要ビュー (セッション数、コスト、ツール使用頻度、エラー率等) を得られるようにする
- **R2.3** **自作ダッシュボード UI は作らない**。R2.1 の backend とその可視化フロントエンドに完全に委譲する
- **R2.4** バックエンドはユーザー選択。Gengar が特定のバックエンドに依存しない

### R3. Claude Code からの集計アクセス (post-MVP / 自己改善)

- **R3.1** Claude Code が MCP サーバー経由でメトリクスをクエリできること (post-MVP)
- **R3.2** CLI からも同等の集計を取得できること (post-MVP)
- **R3.3** クエリ結果は Claude Code が文脈として取り込みやすい構造（JSON または簡潔なテキストサマリ）で返せること
- **R3.4** CLI / MCP はいずれも **backend の query API (PromQL など) の薄いラッパー**として実装する。Gengar 自身はクエリエンジンを持たない
- **R3.5** 自己改善に有用な典型クエリ (最も失敗したツール、平均ターン数、高コストセッション等) をプリセットの PromQL として同梱する

### R4. データライフサイクル (外部委譲)

- **R4.1 保持期間・ロールアップ**: backend (Prometheus 等) の retention / recording rules に完全に委譲する。Gengar 自身は保持期間を管理しない
- **R4.2 個別削除**: backend の delete API (Prometheus なら `/api/v1/admin/tsdb/delete_series`) の薄いラッパーを CLI として提供する (post-MVP)。機密情報が意図せず収集された場合の救済手段
- **R4.3 遡及取り込み (バックフィル)**: 過去 JSONL の取り込みは `gengar backfill --since <date>` で **OpenMetrics text 形式でファイル出力**し、`promtool tsdb create-blocks-from openmetrics` などユーザー側のツールに渡して取り込んでもらう形とする (Phase 1 では OpenMetrics 出力のみ提供、backend への投入はユーザー操作)
- **R4.4 エクスポート**: backend 側で完結するため Gengar としては提供しない

## 4. 非機能要求

- **N1. ローカルファースト**: 収集データは原則ローカルで完結する。ユーザーが選ぶ OTel backend も自己ホスティング前提とし、クラウド送信を既定では行わない。
- **N2. プライバシー**: Gengar は**純粋なメトリクス収集ツール**であり、プロンプト／アシスタント応答／ツール引数・結果の**生テキスト本文は一切保存・送信しない**。emit するのは数値メトリクスと `session_id` / `model` / `project_path` / `tool_name` / `bash_program` などのメタデータラベルのみ。
- **N3. 軽量性**: hook 処理は Claude Code 本体の体感を阻害しない（個別 hook の処理時間は 50ms 以下を目標）。hook はファイル追記するだけで、重い処理 (OTel emit、JSONL 差分読み) は daemon ワーカーが非同期に行う。
- **N4. 堅牢性**: 収集プロセスが落ちても Claude Code の実行を妨げない (hook は非致命的に失敗)。daemon 停止中でも hook によるイベント追記は継続し、daemon 再起動後に追いつける。
- **N5. 拡張性**: メトリクスの追加・ラベルの追加が後から可能な構造であること。
- **N6. 可搬性**: macOS / Linux で動作する（Windows は努力目標）。
- **N7. 追記型 + 単一ワーカー**: 各イベントは hook から**追記のみ**で記録する (append-only)。集計・OTel emit・取り込みオフセット更新など状態を伴う処理は **`gengar daemon` 単一ワーカー**に集約する。これにより複数 `claude` プロセス並走時も書き込み競合が発生しない。

## 5. スコープ

> **基本方針**: 初期フェーズでは **Claude Code のみ** を対象とする。他の AI コーディングエージェント (Codex CLI / Cursor / Aider / Cline / Gemini CLI / Copilot 等) への対応は**初期フェーズではスコープ外**。

### 5.1 Phase 1 MVP (v0.1) のスコープ

**含める:**
- hook 受信 + JSONL 差分読み込み
- OTel SDK による OTLP emit (gRPC / HTTP)
- `gengar install` / `gengar daemon` / `gengar uninstall`
- Prometheus + Grafana 設定例の同梱 (リファレンス構成)
- 対象エージェントは Claude Code のみ

**含めない (post-MVP):**
- CLI でのメトリクスクエリ (PromQL ラッパー) — v0.2 予定
- MCP サーバー — v0.3 予定
- 個別削除コマンド (`gengar delete`) — post-MVP
- 遡及取り込み (`gengar backfill`) — post-MVP

### 5.2 永久にスコープ外 (自作しない)

- **自作ダッシュボード UI** (Grafana 等に委譲)
- **自作 API サーバー** (backend の API で足りる)
- **自作ストレージ** (backend に委譲)
- **自作クエリエンジン** (PromQL 等に委譲)
- **データの保持・削除・ロールアップ機能** (backend の retention に委譲)

### 5.3 Phase 1 で明示的に扱わない

- Codex CLI / Cursor / Aider / Cline / Gemini CLI / GitHub Copilot CLI 等の取り込み
- エージェント横断のコスト統合ビュー
- エージェントを抽象化した共通データモデル
- 複数ユーザー／チームでの集計・共有
- クラウド同期・SaaS 化
- 認証・権限管理
- ファイル単位 / コマンド全文単位の分析 (どのファイルを頻繁に Read したか、どの Bash コマンドがリトライされたか等) — R1.6 参照
- UI の日本語化・多言語化 (英語のみ)
- 公式 OTel (`CLAUDE_CODE_ENABLE_TELEMETRY`) を入力源として取り込む処理 (Gengar は独立して emit する)

### 5.4 Phase 2 以降で検討

- 他エージェント対応
- チーム共有・クラウド同期
- 認証基盤

### 5.5 スコープ固定の含意

- データモデルは Claude Code の形式に素直に合わせてよい（将来の抽象化に備えた過度な汎用化はしない）
- メトリクス名・ラベルも Claude Code 用語 (session, tool, subagent 等) をそのまま用いる
- メトリクスは `gengar.claude_code.*` のプレフィックスで namespace し、公式 OTel (`claude_code.*`) と衝突しないようにする
- 対応 Claude Code バージョンは**現行最新のみ保証**。hook ペイロードや JSONL 形式の変更には都度追従する

## 6. 配布・ライセンス

- **Phase 1 は private 開発**。ライセンスは未確定とし、将来 OSS 公開が決まった段階で MIT 等を採用する
- 依存する OSS (OTel SDK, Prometheus exposition 形式など) はそれぞれのライセンスに従う
- 先行事例 (ccusage / disler など) の参照・実装引用はライセンスを確認してから行う

## 7. 成功の定義

数値目標は Phase 1 では設けない。以下が主観的に満たされていれば成功とする。

- hook 登録後、特別な設定なしで OTel backend にメトリクスが届き、Grafana で主要ビューが見られる
- 体感で Claude Code の動作が遅くなっていない
- 1 日使って困ることがない (daemon が落ちない、ディスクを食い潰さない)

実測値に基づく SLO 設定は Phase 2 以降に実施する。
