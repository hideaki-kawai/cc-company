# PM（プロジェクト管理）

## 役割
プロジェクトの立ち上げから完了まで進捗を管理する。

## ルール
- プロジェクトは `projects/<project-name>/` ディレクトリで管理
- **各プロジェクトディレクトリには必ず `CLAUDE.md` を置く**
- プロジェクト概要は `projects/<project-name>/<project-name>.md`（`CLAUDE.md` から `@<project-name>.md` でインポート）
- チケットは `tickets/YYYY-MM-DD-title.md`
- 議事録は `meetings/<プロジェクト名>/<yyyymmddhhmm_ミーティング名>/議事録.md`
- プロジェクトのステータス: planning → in-progress → review → completed → archived
- チケットのステータス: open → in-progress → done
- チケット優先度: high / normal / low
- 新規プロジェクト作成時は必ずゴールとマイルストーンを定義
- マイルストーン完了時は秘書のTODOに報告を追記

## フォルダ構成
- `projects/` - プロジェクト管理（1プロジェクト1ディレクトリ）
  ```
  projects/<project-name>/
  ├── CLAUDE.md           # 必須。@<project-name>.md でプロジェクト概要をインポート
  ├── <project-name>.md   # プロジェクト概要・ゴール・マイルストーン
  └── （資料・用語集など）
  ```
- `tickets/` - タスクチケット（1チケット1ファイル）
- `meetings/` - 議事録（プロジェクト別）
