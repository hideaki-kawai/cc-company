---
name: zeims-sync-cancellations
description: >
  Cloudflare D1 の trial_cancellation_feedbacks テーブルにある解約フィードバックを NocoDB の「推進案件」テーブルに同期するスキル。
  D1 にあって NocoDB にない事務所は自動的に新規追加する。
  「解約理由を同期して」「D1の解約フィードバックをNocoDBに反映して」「解約データを取り込んで」など、
  Zeims の解約フィードバック同期を求める発言で必ず使うこと。
---

# Zeims 解約フィードバック同期

D1 の `trial_cancellation_feedbacks` を NocoDB の「推進案件」テーブルに同期するスキル。

---

## 前提知識

### D1 テーブル構造

**`trial_cancellation_feedbacks`**
- `id` TEXT (UUID) — レコード識別子
- `office_id` TEXT — `tax_accounting_offices.id`（整数）を文字列で保持
- `user_id` TEXT — 解約したユーザーのID
- `reason` TEXT — 解約理由（例: "機能が期待と違った", "料金が高い"）
- `detail` TEXT — 自由記述
- `would_reconsider` TEXT — 再契約意向 ("yes" / "no" / "unknown")
- `allow_contact` INTEGER — 連絡許可フラグ (0/1)
- `created_at` INTEGER — Unix ms

**`tax_accounting_offices`**
- `id` INTEGER — 事務所ID（`office_id` の CAST 元）
- `company_name` TEXT — 事務所名
- `contact_person` TEXT — 担当者名
- `email` TEXT — メールアドレス

### JOIN 方法

```sql
CAST(tcf.office_id AS INTEGER) = tao.id
```

（`office_id` は整数IDの文字列表現のため）

### NocoDB

- テーブルID: `mwlle8v92sge2c8`（推進案件）
- マッチング優先順位: ① `メールアドレス` → ② `案件名`（部分一致）

---

## ステップ1: wrangler 認証チェック

```bash
cd /Users/hkawai/workspace/zeims/apps/hono-api && npx wrangler whoami 2>&1
```

`You are logged in` が含まれていない場合:
```
⚠️ Cloudflare の認証が必要です:
cd /Users/hkawai/workspace/zeims/apps/hono-api && npx wrangler login
```

---

## ステップ2: D1 からフィードバックを取得

```bash
cd /Users/hkawai/workspace/zeims/apps/hono-api && npx wrangler d1 execute prod-zeims-db --env production --remote --command "SELECT tcf.id, tcf.office_id, tcf.reason, tcf.detail, tcf.would_reconsider, tcf.allow_contact, datetime(tcf.created_at/1000, 'unixepoch', '+9 hours') as created_jst, tao.company_name, tao.contact_person, tao.email FROM trial_cancellation_feedbacks tcf LEFT JOIN tax_accounting_offices tao ON CAST(tcf.office_id AS INTEGER) = tao.id ORDER BY tcf.created_at DESC;" 2>&1
```

結果が `"results": []` の場合:
```
D1 に解約フィードバックのデータがありません。同期するデータがないため終了します。
```
と報告して終了。

---

## ステップ3: NocoDB の推進案件を全件取得

`mcp__nocodb__queryRecords` を使って推進案件テーブルの全レコードを取得する。

```
tableId: mwlle8v92sge2c8
pageSize: 200
fields: [Id, 案件名, 顧客名, メールアドレス, 解約理由]
```

---

## ステップ4: D1 フィードバックと NocoDB レコードを突合

各 D1 フィードバックに対して以下の順でマッチングを試みる:

### マッチング順序
1. **メールアドレスで完全一致** — `tao.email` と NocoDB の `メールアドレス` を比較
2. **事務所名で部分一致** — `tao.company_name` が NocoDB の `案件名` に含まれる、または逆

### 解約理由フォーマット（D1データを整形）

```
【D1フィードバック (created_jst)】
理由: {reason}
詳細: {detail}
再検討の可能性: {would_reconsider}
```

`detail` が null の場合は `詳細:` 行を省略。
`would_reconsider` が null の場合は `再検討の可能性:` 行を省略。

### 重複チェック
既存の `解約理由` に `【D1フィードバック (created_jst)】` の文字列がすでに含まれている場合はスキップ（2重登録防止）。

---

## ステップ5: NocoDB を更新（マッチした場合）

`mcp__nocodb__updateRecords` で `解約理由` フィールドを更新する。

- 既存の `解約理由` が空の場合: D1フォーマットをそのまま書き込む
- 既存の `解約理由` に内容がある場合: **既存内容を先頭に残し、末尾に追記する**

```
{既存の解約理由}

---
{D1フォーマットのフィードバック}
```

---

## ステップ6: NocoDB に新規追加（マッチしなかった場合）

D1 の事務所が NocoDB に存在しない場合、`mcp__nocodb__createRecords` で新規作成する。

```json
{
  "案件名": "{company_name}",
  "顧客名": "{contact_person}",
  "ステータス": "失注",
  "角度": "☆☆☆☆☆ (0%) 失注",
  "メールアドレス": "{email}",
  "解約理由": "{D1フォーマットのフィードバック}"
}
```

`company_name` が null（JOINできなかった場合）は:
- `案件名`: `office_id: {office_id}` として仮登録
- 備考に `※ tax_accounting_offices に対応する事務所が見つかりませんでした` と記載

---

## ステップ7: 結果を報告

```
## 解約フィードバック同期結果

| 事務所名 | 処理 | 備考 |
|---------|------|------|
| ○○事務所 | 更新 | メールアドレスでマッチ |
| △△税理士 | 新規追加 | NocoDB に存在しなかった |
| □□会計 | スキップ | 既に同期済み |

合計: X件更新 / Y件新規追加 / Z件スキップ
```
