---
name: zeims-weekly-report
description: >
  Zeims の週次定例用実績レポートを自動集計して NocoDB に記録するスキル。
  GA4 のアクティブユーザー数、Gmail のメール問い合わせ数、Cloudflare D1 のトライアル/本契約データを収集し、NocoDB の「実績」テーブルを更新する。
  「週次レポート」「実績を埋めて」「zeims の週次」「定例の数字を集めて」「先週の実績」など、Zeims の週次集計・実績更新を求める発言で必ず使うこと。
---

# Zeims 週次実績レポート

週次定例で使う実績データを自動収集して NocoDB に記録するスキル。

---

## ステップ0: 認証チェック（毎回最初に実行）

データ収集の前に全ツールの認証状態を確認する。未認証のものがあれば、データ収集の前にユーザーへ案内して止まること。

### gog CLI（Gmail）チェック

```bash
gog auth list 2>&1
```

`h.kawai@botchan-system.com` が一覧に**ない**場合 → 以下を案内する：

```
⚠️ gog CLI の認証が必要です。以下を順番に実行してください：

1. gog auth credentials <リポジトリルート>/.claude/credentials/gog-oauth-botchan.json
2. gog login h.kawai@botchan-system.com
   （ブラウザが開くので h.kawai@botchan-system.com でログイン）
```

### wrangler（Cloudflare D1）チェック

```bash
cd /Users/hkawai/workspace/zeims/apps/hono-api && npx wrangler whoami 2>&1
```

`You are logged in` が含まれていない場合 → 以下を案内する：

```
⚠️ Cloudflare の認証が必要です。以下を実行してください：

cd /Users/hkawai/workspace/zeims/apps/hono-api && npx wrangler login
（ブラウザが開くので Cloudflare アカウントでログイン）
```

### GA4 MCP チェック

`mcp__ga4__run_report` ツールが利用可能かどうかを確認する。  
利用できない（ツールが見当たらない）場合 → 以下を案内する：

```
⚠️ GA4 MCP の認証が必要です。以下を実行してください：

claude mcp add-json "ga4" '{"type":"http","url":"https://mcp-ga.stape.ai/mcp"}'

その後 Claude Code を再起動し、表示される認証 URL をブラウザで開いて
info@unson.jp でログインしてください。
```

全て認証済みであれば次のステップへ進む。

---

## ステップ1: 週の範囲を決定する

引数で週が指定されていれば（例: `5/19~5/25`）それを使う。  
なければ**直前の月曜〜日曜**を自動計算する（JST 基準）。

以下の変数を準備する：

| 変数 | 例 | 用途 |
|------|---|------|
| `weekLabel` | `"5/19~5/25"` | NocoDB の week フィールド |
| `weekStartMs` | `1747573200000` | D1 の created_at/updated_at 比較（Unix ms） |
| `weekEndMs` | `1748217599000` | 同上 |
| `gaStartDate` | `"2026-05-19"` | GA4 の date_ranges |
| `gaEndDate` | `"2026-05-25"` | GA4 の date_ranges |
| `gmailAfter` | `"2026/05/19"` | gog の after: フィルタ |
| `gmailBefore` | `"2026/05/26"` | gog の before: フィルタ（翌日） |

**JST→UTC 変換**: `weekStartMs = 月曜 JST 00:00 - 9時間`  
例: 2026-05-19 00:00 JST = 2026-05-18 15:00 UTC = Unix ms `1747573200000`

---

## ステップ2: データを並行収集する

以下を**同時に**実行して効率よく集める。

### 2-A. GA4 アクティブユーザー数

`mcp__ga4__run_report` ツールを使う：

```
property_id: 534919278
date_ranges: [{"start_date": "<gaStartDate>", "end_date": "<gaEndDate>"}]
dimensions: []
metrics: ["activeUsers"]
```

取得値 → `アクティブユーザー`（整数）

### 2-B. メール問い合わせ数

LP フォームからの通知は `from:zeims@unson.jp` で届く。

```bash
gog gmail list "from:zeims@unson.jp after:<gmailAfter> before:<gmailBefore>" -a zeims -j --results-only 2>&1
```

返ってきた JSON 配列の長さ = `メール問い合わせ` の件数。  
（DMARC レポートや営業メールは `zeims@unson.jp` からではないので自動的に除外される）

### 2-C. Cloudflare D1 データ

作業ディレクトリ: `/Users/hkawai/workspace/zeims/apps/hono-api`  
コマンド: `npx wrangler d1 execute prod-zeims-db --env production --remote --command "<SQL>"`

以下の SQL を実行する：

```sql
-- トライアル事務所数（その週に新規作成）
SELECT COUNT(*) as value FROM office_billing_settings
WHERE billing_status = 'trial'
AND created_at >= <weekStartMs> AND created_at < <weekEndMs>;

-- 本契約獲得事務所数（その週に active になった）
SELECT COUNT(*) as value FROM office_billing_settings
WHERE billing_status = 'active'
AND updated_at >= <weekStartMs> AND updated_at < <weekEndMs>;

-- 現状の事務所数（trial + active の合計）
SELECT COUNT(*) as value FROM office_billing_settings
WHERE billing_status IN ('trial', 'active');

-- 現状のユーザー数
SELECT COUNT(*) as value FROM user;

-- 解約数（その週に cancelled になった）
SELECT COUNT(*) as value FROM office_billing_settings
WHERE billing_status = 'cancelled'
AND updated_at >= <weekStartMs> AND updated_at < <weekEndMs>;

-- トライアル解約（trial のまま cancelled になった）
SELECT COUNT(*) as value FROM office_billing_settings
WHERE billing_status = 'cancelled'
AND trial_ends_at IS NOT NULL
AND updated_at >= <weekStartMs> AND updated_at < <weekEndMs>;
```

wrangler の出力は JSON 配列。`results[0].value` が各クエリの結果。  
WARNING（LOG_LEVEL / INITIAL_PASSWORD の vars 継承警告）が出ても無視してよい。

---

## ステップ3: NocoDB を更新する

`mcp__nocodb__queryRecords` で `week` フィールドが一致する行を検索する：

```
tableId: mg8zdhldel0iic6
where: (week,eq,<weekLabel>)
pageSize: 1
```

- **行が存在する** → `mcp__nocodb__updateRecords` で更新（`id` を使う）
- **行が存在しない** → `mcp__nocodb__createRecords` で新規作成

更新するフィールド：

| フィールド | 値 |
|-----------|---|
| `week` | weekLabel |
| `アクティブユーザー` | GA4 の activeUsers（数値） |
| `メール問い合わせ` | gog の件数（文字列） |
| `トライアル事務所数` | D1 クエリ結果（数値） |
| `本契約獲得事務所数` | D1 クエリ結果（数値） |
| `現状の事務所数` | D1 クエリ結果（数値） |
| `現状のユーザー数` | D1 クエリ結果（数値） |
| `解約数` | D1 クエリ結果（数値） |
| `トライアル解約` | D1 クエリ結果（数値） |

**スキップするフィールド**（手動入力 or NocoDB の Formula）:
- `トライアル事務所合計`（Formula で自動計算）
- `PRTimesビュー`、`紹介`、`イベント`、`アポ`、`獲得ユーザー数`

---

## ステップ4: 結果を報告する

```
## 週次実績: <weekLabel>

| 指標 | 値 |
|------|---|
| アクティブユーザー | XX |
| メール問い合わせ | XX |
| トライアル事務所数（週間） | XX |
| 本契約獲得事務所数 | XX |
| 現状の事務所数 | XX |
| 現状のユーザー数 | XX |
| 解約数 | XX |
| トライアル解約 | XX |

NocoDB に記録しました（新規作成 or 更新）。
```

---

## 新しい Mac でのセットアップ（参考）

リポジトリをクローンした後、以下を一度だけ実行する：

### 1. gog CLI（Gmail）
```bash
# OAuth クライアントをリポジトリから登録
gog auth credentials .claude/credentials/gog-oauth-botchan.json
# ブラウザ認証
gog login h.kawai@botchan-system.com
```

### 2. Cloudflare（D1）
```bash
cd /Users/hkawai/workspace/zeims/apps/hono-api
npx wrangler login
```

### 3. GA4 MCP（Stape）
```bash
claude mcp add-json "ga4" '{"type":"http","url":"https://mcp-ga.stape.ai/mcp"}'
```
Claude Code 再起動後、表示される URL をブラウザで開いて `info@unson.jp` でログイン。

---

## 認証の有効期限について

| ツール | 有効期限 | 切れたら |
|--------|---------|---------|
| gog（リフレッシュトークン） | 長期（数ヶ月〜） | `gog login h.kawai@botchan-system.com` を再実行 |
| wrangler | 長期（数ヶ月〜） | `npx wrangler login` を再実行 |
| Stape GA4 MCP | セッション単位 | Claude Code 再起動時に自動で再認証プロンプト |
