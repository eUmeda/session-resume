# session-resume

Claude Code の**セッション中断 → 次回再開**を滑らかにする Skill。作業コンテキスト・完了/未完了タスク
（status 分類）・ファイル構成・設計決定・次の一手 recommendation を、**LLM プロンプティング best practice**
（XML 構造化 / context-first / MUST·MUST NOT）に沿った再開プロンプトとして生成し、対象ディレクトリの
`SESSION-RESUME.md` に書き出します。

## 特徴
- `--dir <path>` で対象を明示でき、**プロジェクトを移動/改名した直後でも**正しく動く（cwd 非依存）。
- 未完了 TODO を `[actionable]` / `[deferred]` / `[blocked]` に分類し、actionable があれば
  最有力 1 件 + alternative 1-2 件を理由付きで recommend。
- user が明示依頼したときのみ起動（Claude の自己判断では自発起動しない）。

## インストール
```sh
git clone https://github.com/eUmeda/session-resume.git
mkdir -p ~/.claude/skills
ln -s "$PWD/session-resume" ~/.claude/skills/session-resume
```
シンボリックリンクが使えない環境（developer mode 無効の Windows 等）では、`ln -s` の代わりに
ディレクトリごとコピー（`cp -R session-resume ~/.claude/skills/session-resume`）でも動作します。

Claude Code で `/session-resume [--dir <path>] [補足メモ]` で起動。

## 出力
- 対象 dir の `SESSION-RESUME.md`（ディレクトリ移動後も残るので場所非依存で復帰可）+ チャットに要点サマリー。
- 次セッションは「`SESSION-RESUME.md` を読んで再開」だけで復帰できる。

## 記憶の扱い
公式 auto-memory（`~/.claude/projects/<dir slug>/memory/`）を正本として参照する。`<dir slug>` は対象
絶対パスの「英数字以外を `-` に置換」したもの。プロジェクト移動直後は旧 slug 下の memory も読み取りで参照する
（新規作成はしない）。

## 利用上の注意（Terms / Disclaimer）
本 Skill は現状有姿（**AS IS**）で提供され、いかなる保証もありません。**利用は自己責任**で。
`SESSION-RESUME.md` は実行のたび最新内容で**上書き**されます。作者は損害について責任を負いません。

## 引用（Citation）
役立った場合、引用やリンクでの言及を歓迎します（必須ではありません）。
> Eisaku Umeda (2026). *session-resume*. https://github.com/eUmeda/session-resume

## License
MIT — see [LICENSE](LICENSE).

---
🤖 Built and maintained with [Claude Code](https://claude.com/claude-code).

> 注: `references/examples.md` は合成サンプル（架空プロジェクト）で、再開プロンプトの書式・情報量の参考用です。
