# model-harbor

Multi-model AI routing gateway with OPA-based policy engine (Go).
Routes LLM requests to the optimal provider (Anthropic Claude / OpenAI-compatible) based on task type, token count, and risk level — enforced by Open Policy Agent rules.

OPA ベースのポリシーエンジンを持つ、マルチモデル AI ルーティングゲートウェイです（Go）。
タスク種別・トークン数・リスクレベルに応じて LLM リクエストを最適なプロバイダーへルーティングします。ルールは Open Policy Agent で管理されます。

## Architecture / アーキテクチャ

目標構成です。現状の実装範囲は [Status](#status--ステータス) を参照してください。
Target design. See [Status](#status--ステータス) for what is implemented today.

```
Client → POST /v1/chat/completions
              │
       ┌──────▼──────┐
       │   Router     │  task_type + token_count + risk → model selection
       │   (OPA)      │
       └──────┬──────┘
              │
     ┌────────┴────────┐
     ▼                 ▼
 Anthropic          OpenAI-compatible
 (Claude)           (any endpoint)
```

## Usage / 使い方

```bash
go build -o modelharbor ./cmd/modelharbor

# Comma-separated caller keys. If unset, every request is rejected (fail closed).
# 呼び出し側キー(カンマ区切り)。未設定なら全リクエストを拒否します。
MODELHARBOR_API_KEYS=<key> ./modelharbor serve --addr :8080

curl -s http://localhost:8080/healthz          # 200

curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer <key>" \
  -H "Content-Type: application/json" \
  -d '{"model":"x","messages":[{"role":"user","content":"Write a binary search in Go"}]}'
# → 501 {"error":{"code":"not_implemented", ...}}  (routing is not wired yet)
```

## Tech Stack / 技術スタック

- **Language / 言語:** Go 1.22
- **Policy engine / ポリシーエンジン:** Open Policy Agent (OPA)
- **Providers / プロバイダー:** Anthropic Claude, OpenAI-compatible APIs
- **Auth / 認証:** Bearer token (constant-time via `crypto/subtle`)

## API

| Method | Path | Description / 説明 |
|---|---|---|
| POST | `/v1/chat/completions` | Bearer auth required. **Currently returns 501 `not_implemented`** / Bearer 認証必須。現状は 501 を返すスタブ |
| GET | `/healthz` | Health check / ヘルスチェック |

## Status / ステータス

Early skeleton. 初期の骨組みの段階です。

| Area / 項目 | State / 状態 |
|---|---|
| Bearer auth (`MODELHARBOR_API_KEYS`, constant-time, fail closed) | Implemented / 実装済み |
| `/healthz` | Implemented / 実装済み |
| `internal/router`, `internal/provider` (Anthropic / OpenAI-compatible) | Implemented as libraries, **not yet wired into the HTTP handler** / ライブラリとして実装済み、HTTP ハンドラには未接続 |
| `/v1/chat/completions` routing | Stub (501) / スタブ |
| OPA policy engine | Not started (no OPA dependency yet) / 未着手(依存も未導入) |

## CI

- `go test` (coverage threshold 70%) runs via the reusable workflow `flipslidersand-labs/qa-workflows` `go-test.yml` (`.github/workflows/ci.yml`), plus gitleaks.
  CI は qa-workflows の reusable workflow `go-test.yml` で test(カバレッジ閾値 70%)を実行し、gitleaks も走ります。
- **Required setting / 必須設定:** the runner is selected by the Actions variable `GATE_RUNNER` (set at org level). Forks must define `GATE_RUNNER` themselves.
  runner は Actions 変数 `GATE_RUNNER`(org で設定)で選びます。fork では各自 `GATE_RUNNER` を設定してください。

## License

MIT
