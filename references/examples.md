# Session Resume Prompt — 実例集（合成サンプル）

以下は再開プロンプトの**書きぶり・情報量・具体性の参考**となる合成例（架空プロジェクト）。
実在のプロジェクトではなく、フォーマットの基準を示すためのもの。

> **記憶参照の方針**: context-file は**公式 auto-memory**
> (`~/.claude/projects/<dir slug>/memory/MEMORY.md`) を第一に挙げる。
> （`<dir slug>` は対象絶対パスの「英数字以外を `-` に置換」したもの。）

---

## 例1: 静的サイトのファイル整理（シンプルなタスク）

```
⏺ セッション再開情報

  Current Directory
  /Users/you/projects/portfolio-site

  Context維持のための Markdown
  /Users/you/.claude/projects/-Users-you-projects-portfolio-site/memory/MEMORY.md
  /Users/you/projects/portfolio-site/README.md

  再開プロンプト

  前セッションで assets の再配置を完了した。

  完了済み:
  - images/ を assets/img/ に移動（37 ファイル）
  - 相対リンクを一括置換し、リンク切れ 0 を確認
  - バックアップ取得済み

  ファイル構成:
    /Users/you/projects/portfolio-site/
    ├── index.html
    ├── assets/img/          # 画像 37 枚
    ├── css/
    └── README.md

  次のタスクを指示してください。
```

---

## 例2: REST API リファクタ（複雑なプロジェクト）

複数セクション・設計決定・複数コンテキストファイルを含む例。

```
⏺ Current directory:
  /Users/you/projects/orders-api

  Context維持のための markdown:
  /Users/you/.claude/projects/-Users-you-projects-orders-api/memory/MEMORY.md
  /Users/you/projects/orders-api/docs/ARCHITECTURE.md

以下のコンテキストで作業を再開してください。

  ## 作業ディレクトリ
  /Users/you/projects/orders-api

  ## プロジェクト概要
  注文管理 REST API のサービス層リファクタ。コントローラから業務ロジックを分離する。

  ## 現在の状態
  - OrderService / PaymentService を抽出済み（コントローラは薄くなった）
  - 単体テスト 84% カバレッジ
  - 未完了: 在庫サービスの抽出、トランザクション境界の見直し

  ## セクション構成
  - controllers/ → services/ → repositories/ の 3 層に整理

  ## 重要な設計決定
  - 業務ロジックは service 層のみ（controller/repository に書かない）
  - トランザクションは service 層で開始・コミット
  - 外部 API 呼び出しは必ずリトライ + タイムアウトを付ける

  まず docs/ARCHITECTURE.md とメモリを読み込んで状況を把握してください。
```

---

## 例3: ログ解析パイプライン（大規模データ・フェーズ管理）

Phase 管理・生成物・未完了タスクの status 分類を含む例。

```
⏺ セッション再開情報

  Current Directory
  /Users/you/projects/log-analytics

  Context 維持のための Markdown
  /Users/you/.claude/projects/-Users-you-projects-log-analytics/memory/MEMORY.md
  /Users/you/projects/log-analytics/HISTORY.md
  /Users/you/projects/log-analytics/TODO.md

  再開プロンプト

  以下のコンテキストで作業を再開してください。

  ## プロジェクト概要
  アクセスログ（数百万行）からエラー傾向を可視化する解析パイプライン。

  ## 現在の状態
  完了済み:
  - Phase A: ログのパースとスキーマ正規化
  - Phase B: 日次集計（エラー率・レイテンシ分位点）
  - Phase C: ダッシュボード HTML 生成

  未完了:
  - [ ] Phase D: 異常検知（移動平均からの逸脱フラグ） `[actionable]`
  - [ ] 旧データセット（生ログ）のアーカイブ `[blocked]` — ストレージ増設待ち
  - [ ] 命名規則の最終決定 `[deferred]` — チームの合意待ち

  ## ファイル構成
    /Users/you/projects/log-analytics/
    ├── parse/                # パーサ
    ├── aggregate/            # 集計ジョブ
    ├── report.html           # 生成ダッシュボード
    ├── HISTORY.md
    └── TODO.md

  ## 重要な設計決定
  - タイムゾーンは UTC に統一して集計
  - 大きな中間ファイルは Git 管理しない（.gitignore）

  まず MEMORY.md と HISTORY.md を読み込んで状況を把握してください。
```

---

## 例4: 通知自動化（システム構築）

cron/launchd・設定変更を含むシステム系プロジェクトの例。

```
⏺ セッション再開情報

  Current Directory
  /Users/you

  Context維持のための Markdown
  /Users/you/.claude/projects/-Users-you/memory/MEMORY.md

  再開プロンプト

  前セッションで日次サマリーの通知自動化を構築した。

  完了済み:
  - 日次サマリー生成スクリプトを作成（~/scripts/daily-summary.sh）
  - cron で毎朝 8:00 実行に登録
  - 結果をメール送信（SMTP 設定済み、テスト送信 OK）

  未完了/今後:
  - [ ] 失敗時のリトライ・アラート `[actionable]`
  - [ ] 長期運用での安定性確認 `[blocked]` — 数日の経過観察待ち

  ファイル構成:
    /Users/you/
    ├── scripts/daily-summary.sh
    └── (crontab に 1 エントリ追加済み)

  次のタスクを指示してください。
```

---

## 例から読み取れるパターン
1. **情報量の調整**: シンプルなタスクは簡潔に、複雑なプロジェクトは詳細に。
2. **ファイル構成**: tree 形式で主要ファイルとコメントを含める。
3. **設計決定**: 次セッションで違反してはいけない制約を明記。
4. **未完了 TODO**: `[actionable]` / `[deferred]` / `[blocked]` を付与。
5. **末尾の指示**: 必ず「まず … を読み込んで」で終える。
6. **絶対パス**: コンテキストファイルは絶対パスで記述。
