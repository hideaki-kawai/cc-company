# データソース詳細

## 1. Google Analytics 4（アクティブユーザー）

| 項目 | 値 |
|------|---|
| プロパティ名 | zeims |
| プロパティ ID | `534919278` |
| 認証アカウント | `info@unson.jp` |
| MCP サーバー | `ga4`（Stape: `https://mcp-ga.stape.ai/mcp`） |
| 取得指標 | `activeUsers`（週間） |

### クエリ例

```
tool: mcp__ga4__run_report
property_id: 534919278
date_ranges: [{"start_date": "2026-05-19", "end_date": "2026-05-25"}]
dimensions: []
metrics: ["activeUsers"]
```

---

## 2. Gmail（メール問い合わせ）

| 項目 | 値 |
|------|---|
| Gmail アカウント | `h.kawai@botchan-system.com` |
| gog エイリアス | `zeims` |
| LP 問い合わせ通知の送信元 | `zeims@unson.jp` |
| OAuth クライアント JSON | `.claude/credentials/gog-oauth-botchan.json` |
| Google Cloud プロジェクト | `zeims-490710-497507` |

### クエリ例

```bash
# 問い合わせ件数カウント（from:zeims@unson.jp で絞り込み）
gog gmail list "from:zeims@unson.jp after:2026/05/19 before:2026/05/26" -a zeims -j --results-only
```

> LP フォームからの通知は全て `from:zeims@unson.jp` で届く。DMARC レポートや営業メールは除外される。

---

## 3. Cloudflare D1（トライアル・契約データ）

| 項目 | 値 |
|------|---|
| 本番 DB 名 | `prod-zeims-db` |
| DB ID | `15987081-75be-4c2b-b4c9-dafcbcf5f32c` |
| Account ID | `788e556343893a7135c29b782c22fb24` |
| wrangler 設定 | `/Users/hkawai/workspace/zeims/apps/hono-api/wrangler.jsonc` |
| タイムスタンプ形式 | Unix ミリ秒（timestamp_ms） |

### コマンド形式

```bash
cd /Users/hkawai/workspace/zeims/apps/hono-api
npx wrangler d1 execute prod-zeims-db --env production --remote --command "<SQL>"
```

### SQL 一覧

```sql
-- トライアル事務所数（当週新規）
SELECT COUNT(*) as value FROM office_billing_settings
WHERE billing_status = 'trial'
AND created_at >= {weekStartMs} AND created_at < {weekEndMs};

-- 本契約獲得（当週に active になった）
SELECT COUNT(*) as value FROM office_billing_settings
WHERE billing_status = 'active'
AND updated_at >= {weekStartMs} AND updated_at < {weekEndMs};

-- 現状の事務所数（trial + active）
SELECT COUNT(*) as value FROM office_billing_settings
WHERE billing_status IN ('trial', 'active');

-- 現状のユーザー数
SELECT COUNT(*) as value FROM user;

-- 解約数（当週に cancelled になった）
SELECT COUNT(*) as value FROM office_billing_settings
WHERE billing_status = 'cancelled'
AND updated_at >= {weekStartMs} AND updated_at < {weekEndMs};

-- トライアル解約（trial のまま cancelled）
SELECT COUNT(*) as value FROM office_billing_settings
WHERE billing_status = 'cancelled'
AND trial_ends_at IS NOT NULL
AND updated_at >= {weekStartMs} AND updated_at < {weekEndMs};
```

> `{weekStartMs}` = 月曜 00:00:00 JST の Unix ms（JST = UTC+9 なので -9時間）  
> 例: 2026-05-19 00:00 JST = `1747573200000`

---

## 4. NocoDB（書き込み先）

| 項目 | 値 |
|------|---|
| Base ID | `pr8u5q4qnb8op11` |
| Table ID | `mg8zdhldel0iic6`（実績テーブル） |
| MCP | `nocodb`（`noco.unson.jp`） |

### 行の特定

`week` フィールドで一致する行を検索 → 存在すれば更新、なければ新規作成。
