# 先行事例調査

Claude Code を含むローカル AI エージェントのメトリクス収集・可視化に関する既存プロダクト／記事の調査メモ。Gengar のポジショニング決定、再利用可能な実装やデータモデルの洗い出し、差別化ポイントの明確化に用いる。

> 調査日: 2026-04-22

## 1. 公式 (Anthropic)

| 名称 | 種別 | 概要 | 備考 |
|---|---|---|---|
| Analytics Dashboard | SaaS UI | claude.ai/analytics/claude-code。受け入れ率・DAU・PR 紐付けなど | Team / Enterprise 限定、個人利用では使えない |
| Claude Code Analytics Admin API | Web API | 日次集計を取れる組織向け API | 組織利用が前提 |
| OpenTelemetry 内蔵エクスポート | SDK/CLI 機能 | `CLAUDE_CODE_ENABLE_TELEMETRY=1` + OTLP で 8 種類のメトリクス (session 数、コスト、トークン種別、edit 行数、active time 等) + logs/events を吐ける | exporter は `console` / `otlp` / `prometheus` 等を選択可 |

参考:
- <https://code.claude.com/docs/en/analytics>
- <https://platform.claude.com/docs/en/build-with-claude/claude-code-analytics-api>
- <https://code.claude.com/docs/en/monitoring-usage>

### Gengar への示唆

- OTEL 経由で公式にコア時系列メトリクスが取れるので「OTEL を前提として補完に徹する」か「OTEL を使わず独自完結する」かの方針選択が必要。
- 管理 API は個人ユーザーでは使えない。ローカル情報源 (hook + JSONL) の統合が実質的な差別化軸。

---

## 2. JSONL ベースのコスト／使用量 CLI

| 名称 | URL | 特徴 | 参考度 |
|---|---|---|---|
| **ccusage** | <https://ccusage.com/> / <https://github.com/ryoppippi/ccusage> | `~/.claude/projects/*.jsonl` を解析し日次/月次/セッション別コストを表示。`--breakdown`, `--since/--until`, `--json` 等。Codex CLI にも対応。4,800+ ★ | ★★★ (実質デファクト) |
| cccost | <https://github.com/badlogic/cccost> | よりシンプルなコスト集計 | ★★ |
| Claude-Code-Usage-Monitor | <https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor> | リアルタイム表示 + 予測/警告 | ★★ |
| phuryn/claude-usage | <https://github.com/phuryn/claude-usage> | localhost:8080 の SPA ダッシュボード、Chart.js。Pro/Max 向けの進捗バーも | ★★ |

### Gengar への示唆

- コスト計算ロジック・モデル単価テーブル・キャッシュトークンの扱いは ccusage を参照（ライセンス要確認）。
- CLI インターフェース `gengar costs` 系の UX は ccusage 相当を最低ラインにする。

---

## 3. トランスクリプトビューア

| 名称 | URL | 特徴 |
|---|---|---|
| simonw/claude-code-transcripts | <https://github.com/simonw/claude-code-transcripts> | JSONL → HTML、Gist 公開 |
| daaain/claude-code-log | <https://github.com/daaain/claude-code-log> | Python CLI、HTML/Markdown 変換 |
| withLinda/claude-JSONL-browser | <https://github.com/withLinda/claude-JSONL-browser> | Web UI + ファイルエクスプローラ |
| raine/claude-history | <https://github.com/raine/claude-history> | ファジー検索 + vim 風ビューア |

### Gengar への示唆

- 「会話の中身を人が読める形で見る」機能は競合多数のため、Gengar 本体ではセッションタイムライン（ツール呼び出し中心）に絞り、本文閲覧は deep link で既存ツールに譲る選択肢もある。

---

## 4. Hook ベースのリアルタイム観測

| 名称 | URL | 特徴 |
|---|---|---|
| disler/claude-code-hooks-multi-agent-observability | <https://github.com/disler/claude-code-hooks-multi-agent-observability> | hook イベントを収集・保存・可視化。複数エージェントの同時監視に強い |

### Gengar への示唆

- Gengar の F1.1 (Hook イベントハンドラ) と発想が重複する。イベントスキーマ・hook 登録方法を参照する価値が高い。
- 差別化は「hook + JSONL 統合」「MCP / CLI からのクエリ」「自己改善ヒント生成」で出す。

---

## 5. Statusline 系

| 名称 | URL | 特徴 |
|---|---|---|
| Claude HUD (Jarrod Watts) | <https://pub.towardsai.net/claude-hud-building-real-time-observability-for-claude-code-via-the-statusline-api-b114b825d3ef> | context / tools / agents / todos をステータスラインに |
| CCometixLine | awesome-claude-code 掲載 | Rust 製、Git 連携 + TUI 設定 |
| claude-pace | awesome-claude-code 掲載 | レート制限の消費ペース表示 |
| ccusage statusline | <https://ccusage.com/guide/statusline> | ccusage をステータスラインに組み込む |

### Gengar への示唆

- 常駐ダッシュボードが別にあるなら、Statusline は既存 OSS を推奨するだけで十分。Gengar では「Statusline 向けの軽量 API (`/api/statusline`)」を用意する程度に留めるのが現実的。

---

## 6. OTEL + 外部スタック

| 名称 | URL | 特徴 |
|---|---|---|
| ColeMurray/claude-code-otel | <https://github.com/ColeMurray/claude-code-otel> | OTel Collector + Prometheus + Grafana の一式 |
| SigNoz | <https://signoz.io/blog/claude-code-monitoring-with-opentelemetry/> / <https://signoz.io/docs/dashboards/dashboard-templates/claude-code-dashboard/> | SigNoz に取り込むテンプレ |
| Sealos | <https://sealos.io/blog/claude-code-metrics/> | Grafana セットアップ手順 (2026) |
| Quesma | <https://quesma.com/blog/track-claude-code-usage-and-limits-with-grafana-cloud/> | Grafana Cloud 連携 |
| Honeycomb | <https://www.honeycomb.io/blog/can-claude-code-observe-its-own-code> | Honeycomb での分析事例 |
| VictoriaMetrics + Grafana | <https://tcude.net/how-i-monitor-my-claude-code-usage-with-grafana-opentelemetry-and-victoriametrics/> | 個人セルフホスト事例 |
| Grafana Dashboard #24993 | <https://grafana.com/grafana/dashboards/24993-claude-code-metrics/> | 公式マーケットのダッシュボード定義 |

### Gengar への示唆

- 「本格的な時系列解析が欲しい人」向けの選択肢は既に成熟。Gengar は **時系列よりもセッション単位のドリルダウン + 自己改善クエリ** に寄せるのが合理的。
- ただし OTEL 出力を受け取って取り込むモードがあると上位互換になる（F1 の入力源として OTLP receiver をオプション追加）。

---

## 7. マルチツール横断

| 名称 | URL | 特徴 |
|---|---|---|
| **OpenUsage** | <https://openusage.sh/> | Claude Code / Codex / Cursor / Copilot / Gemini CLI / OpenRouter / OpenAI / Anthropic を横断。ローカル SQLite、MCP 使用量・コード統計・daemon 履歴まで拾う |

### Gengar への示唆

- **最も競合し得る既存プロダクト**。横断対応は今回のスコープ外 (docs/requirements.md §5) としているので、Gengar は「Claude Code に特化して hook イベント粒度まで深掘りする」方向で棲み分ける。
- ストレージに SQLite を選ぶアーキ判断は OpenUsage と一致しており、方向性として妥当である裏付けにもなる。

---

## 8. アドホック分析／書き物

| 名称 | URL | 特徴 |
|---|---|---|
| Analyzing Claude Code Interaction Logs with DuckDB (Liam ERD) | <https://liambx.com/blog/claude-code-log-analysis-with-duckdb> | JSONL を DuckDB で直接 SQL する手法 |
| Claude Code Cost Tracking: Monitor and Cut Your Spending | <https://dev.to/aavisangle/claude-code-cost-tracking-monitor-and-cut-your-spending-4cge> | 個人のコスト最適化記事 |
| How to Monitor Claude Code Usage Across Your Engineering Team (Jellyfish) | <https://jellyfish.co/library/claude-code-monitoring/> | チーム向け運用論 |
| awesome-claude-code | <https://github.com/hesreallyhim/awesome-claude-code> | skills / hooks / slash-commands / 拡張の総合リンク集 |
| A new way to extract detailed transcripts from Claude Code (Simon Willison) | <https://simonw.substack.com/p/a-new-way-to-extract-detailed-transcripts> | JSONL 抽出の背景 |

### Gengar への示唆

- 「とりあえず DuckDB で SQL を書いて済ませる」層が存在する。自由 SQL クエリ (F3.1 の `POST /api/query` あるいは `gengar query` サブコマンド) を初期から用意しておくと、この層にも受け入れられやすい。

---

## 9. 機能マトリクス（ざっくり比較）

凡例: ● あり / ◐ 部分対応 / ○ なし / — 該当せず

| 観点 | Anthropic 公式 (OTel) | ccusage | disler/multi-agent-observability | OpenUsage | **Gengar (Phase 1)** |
|---|:---:|:---:|:---:|:---:|:---:|
| ローカル完結 | ◐ (受け側要) | ● | ● | ● | ● (docker-compose で完結) |
| 公式 OTel を前提 | ● (source) | ○ | ○ | ○ | ● (sink 側を束ねる) |
| 即使えるダッシュボード | ○ | ○ | ◐ | ◐ | ● (Grafana JSON 同梱) |
| メトリクス収集 | ● (自前) | ● (JSONL) | ● (hook) | ● | ○ (公式 OTel に委譲) |
| コスト可視化 | ● (metric) | ● (CLI) | ○ | ● | ● (ダッシュボード) |
| セッションドリルダウン | ◐ | ◐ | ● | ◐ | ● (session.id フィルタ) |
| エラー・リトライ追跡 | ● (events) | ○ | ● | ○ | ● |
| subagent 分析 | ● (query_source) | ○ | ◐ | ○ | ● |
| マルチツール (Codex/Cursor 等) | ○ | ◐ (Codex のみ) | ○ | ● | ○ (スコープ外) |
| Claude Code からの MCP クエリ | ○ | ○ | ○ | ◐ | ◐ (post-MVP) |
| 実装コード量 | — | 多 | 多 | 多 | **Phase 1 ゼロ** |

---

## 10. Gengar のポジショニング

公式 OTel の網羅性が想定以上だったため、Gengar の立ち位置は「**公式 OTel を最速で可視化するための設定パッケージ + Grafana ダッシュボード**」に固定する。

1. **Claude Code 専用**として開始する。初期フェーズでは他エージェント対応は明確にスコープ外 (`docs/requirements.md` §5)。
2. **公式 OTel (`CLAUDE_CODE_ENABLE_TELEMETRY=1`) を前提とし、独自の collector / daemon は作らない**。公式が metrics・events・traces を OTLP で出せるため、Gengar は受け側を束ねて可視化する役割に徹する。
3. **Gengar の核心価値は Grafana ダッシュボード**。Overview / Cost / Tools / Sessions / Subagent / Errors / Compaction の主要ビューを JSON として同梱する。
4. **docker-compose 一発**で Prometheus + Loki + Grafana が立ち上がり、Claude Code 側は managed-settings を追記するだけで動く。
5. **実装コードを Phase 1 では書かない**。設定ファイル・ダッシュボード JSON・docs のみ。
6. CLI / MCP ラッパーは post-MVP として追加する (いずれも backend の query API の薄いラッパー)。

**Phase 1 で意図的に切り捨てる選択肢**

- 独自 collector / daemon / JSONL reader
- 独自 OTel メトリクスの emit (公式が提供する分で十分)
- 自作ダッシュボード UI / API サーバー / ストレージ / クエリエンジン
- Claude Code plugin 経由の hook 補完 (必要になったら検討)
- 他エージェント横断対応
- チーム共有・クラウド連携

## 11. 再利用／参照候補

| 参照先 | 参照する理由 |
|---|---|
| ccusage のモデル単価テーブル | コスト計算の再発明回避 |
| disler/multi-agent-observability の hook イベントスキーマ | F1.1 のスキーマ設計の参考 |
| 公式 OTEL メトリクス定義 (8 種) | F1.3 計算項目の漏れチェック |
| OpenUsage の SQLite スキーマ | F2.2 集計テーブルの参考 |
| claude-code-log / claude-JSONL-browser | JSONL パースのエッジケース洗い出し |

## 12. 未調査／追加で見ておくべきもの

- Aider / Cline / Cursor Agent / Codex CLI の JSONL 形式の差分（将来の横断対応を保留するかの判断材料）
- Anthropic OTEL スキーマと hook ペイロードの重複度合い（二重計上の回避設計）
- ライセンス (ccusage / disler / OpenUsage) — 再利用の範囲確認
