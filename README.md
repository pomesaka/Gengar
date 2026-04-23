# Gengar

Claude Code 公式 OpenTelemetry を最速で可視化するための **設定パッケージ + Grafana ダッシュボード**。

Phase 1 は実装コードを書かず、受け側 (Prometheus / Loki / Grafana) の docker-compose と Gengar オリジナルの Grafana ダッシュボード JSON を束ねて提供する。詳細な設計意図は [`docs/requirements.md`](./docs/requirements.md) と [`docs/decisions.md`](./docs/decisions.md) を参照。

---

## Quickstart

### 前提

- Docker / docker-compose
- Claude Code (現行最新版)

### 1. リポジトリを clone

```sh
git clone <this-repo> gengar
cd gengar
```

### 2. スタックを起動

```sh
cp .env.example .env   # 必要なら GF_ADMIN_PASSWORD を編集
docker compose up -d
```

以下が立ち上がる:

| サービス | ポート | 役割 |
|---|---|---|
| Prometheus | `127.0.0.1:9090` | Claude Code の OTel メトリクスを OTLP で受信・保存 (90 日) |
| Loki | `127.0.0.1:3100` | Claude Code の OTel イベント (logs) を OTLP で受信・保存 (30 日) |
| Grafana | `127.0.0.1:3000` | Gengar ダッシュボードを表示 |

### 3. Claude Code に OTel 送信設定を追記

[`claude-code-settings.example.json`](./claude-code-settings.example.json) の `env` ブロックの中身を、自分の `~/.claude/settings.json` の `env` キー配下にマージする。

最小構成の例:

```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_METRICS_PROTOCOL": "http/protobuf",
    "OTEL_EXPORTER_OTLP_METRICS_ENDPOINT": "http://127.0.0.1:9090/api/v1/otlp/v1/metrics",
    "OTEL_EXPORTER_OTLP_LOGS_PROTOCOL": "http/protobuf",
    "OTEL_EXPORTER_OTLP_LOGS_ENDPOINT": "http://127.0.0.1:3100/otlp/v1/logs"
  }
}
```

### 4. Claude Code を使う

普段通り `claude` を起動して何か作業する。OTel は 5〜10 秒ごとにバッチ送信されるので、少し経つとデータが入る。

### 5. Grafana でダッシュボードを開く

ブラウザで <http://127.0.0.1:3000> にアクセス。初回ログインは `admin` / `.env` で指定したパスワード。

左メニュー **Dashboards → Gengar → Gengar — Overview** を開く。以下が見える:

- 直近 24h の **総コスト / 総トークン / セッション数 / Active Time**
- モデル別コスト推移、トークン種別推移
- ツール呼び出し頻度 (Loki events から集計)
- 最近のイベント列 (ライブログビュー)

---

## プライバシー設定

Claude Code 公式 OTel は既定で以下を**送信しない**:

- ユーザープロンプトの本文 (`OTEL_LOG_USER_PROMPTS=0`)
- ツール引数の詳細 (`OTEL_LOG_TOOL_DETAILS=0`) — Bash コマンド、MCP ツール名、ファイルパスなど
- ツール結果の内容 (`OTEL_LOG_TOOL_CONTENT=0`)
- API リクエスト/レスポンスの生 body (`OTEL_LOG_RAW_API_BODIES=未設定`)

ダッシュボードでツール名以上の粒度 (例: Bash コマンド、MCP ツール名) を見たい場合、`OTEL_LOG_TOOL_DETAILS=1` を `claude-code-settings.example.json` の env に追加する。**有効化すると Bash コマンドやファイルパスなどの引数が Loki に保存される**ため、docker stack が `127.0.0.1` でバインドされていることを確認すること。

---

## トラブルシューティング

### スタックが起動しない

```sh
docker compose ps           # サービス状態
docker compose logs prometheus
docker compose logs loki
docker compose logs grafana
```

### Claude Code からメトリクスが届かない

Prometheus OTLP receiver の疎通確認:

```sh
# 空の POST は 400 を返すが、200/400 のいずれかが返れば受信は生きている
curl -I http://127.0.0.1:9090/api/v1/otlp/v1/metrics
```

Loki OTLP receiver:

```sh
curl -I http://127.0.0.1:3100/otlp/v1/logs
```

Claude Code 側で OTel エクスポーターが有効化されているかの確認 (デバッグ用):

```sh
# 一時的に console exporter を有効化して stderr にメトリクスを吐かせる
OTEL_METRICS_EXPORTER=console,otlp OTEL_METRIC_EXPORT_INTERVAL=5000 claude
```

### Grafana にダッシュボードが出ない

Grafana コンテナ内のプロビジョニングログを確認:

```sh
docker compose logs grafana | grep -i provision
```

---

## 設定のカスタマイズ

| 項目 | 場所 |
|---|---|
| Prometheus retention | `docker-compose.yml` の `--storage.tsdb.retention.time` |
| Loki retention | `loki/loki-config.yaml` の `limits_config.retention_period` |
| Grafana 管理者パスワード | `.env` の `GF_ADMIN_PASSWORD` |
| OTel endpoint / プロトコル | `claude-code-settings.example.json` (`~/.claude/settings.json` にマージ) |
| 追加ダッシュボード | `grafana/dashboards/*.json` にファイルを足すだけで自動読み込み (30 秒間隔) |

---

## ライセンス

Phase 1 は private 開発のためライセンスは未確定。将来 OSS 公開する場合は MIT を想定 ([`docs/decisions.md`](./docs/decisions.md) §決定 16 参照)。

同梱する OSS (Prometheus / Loki / Grafana) はそれぞれのライセンスに従う。

---

## 関連ドキュメント

- [`docs/requirements.md`](./docs/requirements.md) — 要求定義
- [`docs/functional-requirements.md`](./docs/functional-requirements.md) — 機能要件
- [`docs/decisions.md`](./docs/decisions.md) — 要件決定ログ (全 19 決定)
- [`docs/prior-art.md`](./docs/prior-art.md) — 先行事例調査 (ccusage, disler, ZOZO, OpenUsage など)

## 参考

- [Monitoring — Claude Code Docs](https://code.claude.com/docs/en/monitoring-usage)
- [社員に何もさせずにClaude Code利用ログを集める (ZOZO TECH BLOG)](https://techblog.zozo.com/entry/claudecode-otel)
