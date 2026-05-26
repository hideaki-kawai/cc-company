# 環境セットアップ手順

新しい Mac やトークン期限切れ時に実行する。

## 前提

- このリポジトリ（cc-company）がクローン済み
- `gog` CLI がインストール済み（`brew install steipete/tap/gogcli` などで導入）
- `npx` が使える（Node.js 環境）
- `claude` CLI がインストール済み

---

## ステップ 1: gog CLI（Gmail 問い合わせ）

```bash
# OAuth クライアントをリポジトリから登録
gog auth credentials .claude/credentials/gog-oauth-botchan.json

# ブラウザが開くので h.kawai@botchan-system.com でログイン
gog login h.kawai@botchan-system.com

# エイリアスを設定
gog auth alias set zeims h.kawai@botchan-system.com
```

動作確認：
```bash
gog auth list
# → h.kawai@botchan-system.com が表示されれば OK
```

---

## ステップ 2: Cloudflare（D1 クエリ）

```bash
cd /Users/hkawai/workspace/zeims/apps/hono-api
npx wrangler login
# ブラウザが開くので Cloudflare アカウントでログイン
```

動作確認：
```bash
npx wrangler whoami
# → You are logged in ... が表示されれば OK
```

---

## ステップ 3: GA4 MCP（Stape）

```bash
# Claude Code に GA4 MCP を追加
claude mcp add-json "ga4" '{"type":"http","url":"https://mcp-ga.stape.ai/mcp"}'
```

Claude Code を**再起動**すると認証 URL が表示されるので、ブラウザで開いて `info@unson.jp` でログイン。

動作確認：スキル実行時にアクティブユーザー数が取得できれば OK。

---

## 認証の有効期限

| ツール | 有効期限の目安 | 再認証コマンド |
|--------|-------------|--------------|
| gog（リフレッシュトークン） | 数ヶ月〜 | `gog login h.kawai@botchan-system.com` |
| wrangler | 数ヶ月〜 | `npx wrangler login` |
| Stape GA4 MCP | セッション単位（Claude Code 再起動ごと） | Claude Code 起動時に自動プロンプト |

スキル実行時に認証切れを検出した場合、上記の再認証コマンドを案内する。

---

## gcloud 設定（参考）

GA4 の追加設定として gcloud に `info@unson.jp` のコンフィグを作成済み。  
通常のスキル実行には不要だが、ADC 経由で GA4 を使いたい場合に参照する。

```bash
gcloud config configurations list
# NAME             IS_ACTIVE  ACCOUNT
# zeims            True       h.kawai.tech@gmail.com  ← メイン
# zeims-analytics  False      info@unson.jp           ← GA4 用
```
