# Zeims プロジェクト

## 概要

税理士事務所向け SaaS「Zeims」の開発・運営プロジェクト。  
Cloudflare スタック（Workers / D1 / R2）で構築。

## ステータス

`in-progress`

## 作成日

2026-05-26

## 技術スタック

| レイヤー | 技術 |
|---------|------|
| API | Cloudflare Workers (Hono) |
| DB | Cloudflare D1 (SQLite) |
| ストレージ | Cloudflare R2 |
| フロントエンド | zeims-web, zeims-lp, zeims-admin |
| メール送信 | Resend (`zeims@unson.jp`) |
| 認証 | Better Auth |
| ORM | Drizzle |

## リポジトリ

`/Users/hkawai/workspace/zeims`

## 関連アカウント

| 用途 | アカウント |
|------|-----------|
| GA4・Google Cloud | `info@unson.jp` |
| 問い合わせ Gmail | `h.kawai@botchan-system.com` |
| Cloudflare | Cloudflare ダッシュボード（account_id: `788e556343893a7135c29b782c22fb24`） |
| NocoDB | `noco.unson.jp`（Base ID: `pr8u5q4qnb8op11`） |

## フォルダ構成

```
.company/pm/projects/zeims/
├── zeims.md              このファイル（プロジェクト概要）
└── weekly-report/
    ├── overview.md       週次レポートの目的・NocoDB フィールド定義
    ├── data-sources.md   各データソースの接続情報・クエリ詳細
    └── setup.md          新しい Mac でのセットアップ手順
```

## 関連ミーティング

`meetings/zeims/` 配下に議事録を管理
