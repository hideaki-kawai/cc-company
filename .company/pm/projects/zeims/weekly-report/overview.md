# 週次実績レポート 概要

## 目的

Zeims の週次定例で報告する実績指標を自動集計し、NocoDB の「実績」テーブルに記録する。

## スキル

`.claude/skills/zeims-weekly-report/` に実装済み。  
「先週の実績集めて」「zeims の週次レポート」などで自動起動する。

## NocoDB 実績テーブル

- **URL**: `https://noco.unson.jp/dashboard/#/nc/pr8u5q4qnb8op11/mg8zdhldel0iic6/`
- **Base ID**: `pr8u5q4qnb8op11`
- **Table ID**: `mg8zdhldel0iic6`
- **week フォーマット**: `M/D~M/D`（例: `5/19~5/25`、月〜日）

## フィールド定義

| フィールド | 型 | 取得元 | 備考 |
|-----------|---|-------|------|
| `week` | Text | 自動計算 | 月〜日の範囲 |
| `メール問い合わせ` | Text | Gmail（gog CLI） | LP フォームからの件数のみ |
| `アポ` | Text | **手動** | — |
| `アクティブユーザー` | Number | Google Analytics 4 | zeims プロパティ（ID: 534919278） |
| `PRTimesビュー` | Number | **スキップ** | API なし |
| `紹介` | Number | **手動** | — |
| `イベント` | Number | **手動** | — |
| `本契約獲得事務所数` | Number | Cloudflare D1 | 当週に active になった事務所数 |
| `獲得ユーザー数` | Number | **手動** | — |
| `トライアル事務所数` | Number | Cloudflare D1 | 当週に新規トライアル開始した事務所数 |
| `トライアル事務所合計` | Formula | NocoDB 自動 | `ADD({トライアル事務所数})` の累計 |
| `現状の事務所数` | Number | Cloudflare D1 | trial + active の合計（現時点） |
| `現状のユーザー数` | Number | Cloudflare D1 | user テーブルの総数（現時点） |
| `解約数` | Number | Cloudflare D1 | 当週に cancelled になった事務所数 |
| `トライアル解約` | Number | Cloudflare D1 | 当週に trial のまま cancelled になった事務所数 |
