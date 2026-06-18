# session-resume

> Claude Code の **セッション中断 → 次回再開** を滑らかにする Skill。
> A Claude Code skill that makes **session interruption → next-session resume** seamless.

**[日本語](#日本語) ・ [English](#english)** — 📖 人間向けの使い方ページ / human-facing guide: [`docs/index.html`](docs/index.html)（GitHub Pages 有効化時: <https://eumeda.github.io/session-resume/>）

---

## 日本語

作業コンテキスト・完了/未完了タスク（status 分類）・ファイル構成・設計決定・次の一手 recommendation を、
**LLM プロンプティング best practice**（XML 構造化 / context-first / MUST·MUST NOT）に沿った再開プロンプト
として生成し、対象ディレクトリの `SESSION-RESUME.md` に書き出します。

次のセッションは **「`SESSION-RESUME.md` を読んで再開して」と言うだけ** で文脈が戻る、というのが狙いです。

### 特徴
- `--dir <path>` で対象を明示でき、**プロジェクトを移動/改名した直後でも**正しく動く（cwd 非依存）。
- 未完了 TODO を `[actionable]` / `[deferred]` / `[blocked]` に分類し、actionable があれば
  最有力 1 件 + alternative 1-2 件を理由付きで recommend。
- user が明示依頼したときのみ起動（Claude の自己判断では自発起動しない）。

### インストール
```sh
git clone https://github.com/eUmeda/session-resume.git
mkdir -p ~/.claude/skills
ln -s "$PWD/session-resume" ~/.claude/skills/session-resume
```
シンボリックリンクが使えない環境（developer mode 無効の Windows 等）では、`ln -s` の代わりに
ディレクトリごとコピー（`cp -R session-resume ~/.claude/skills/session-resume`）でも動作します。

Claude Code で `/session-resume [--dir <path>] [補足メモ]` で起動。

### 出力
- 対象 dir の `SESSION-RESUME.md`（ディレクトリ移動後も残るので場所非依存で復帰可）+ チャットに要点サマリー。
- 次セッションは「`SESSION-RESUME.md` を読んで再開」だけで復帰できる。

### 実際の使い方（実運用でのユースケース）
作者が 2026 年春から日々の Claude Code セッションで使ってきた、定着した使い方です。共通しているのは
**「Claude にその場で生成させ、次セッションの冒頭でそれを読ませて続きから入る」** という運用です。

- **夜に閉じる → 翌朝そのまま再開（最頻）** — セッション末尾に `/session-resume`、翌朝は
  「`SESSION-RESUME.md` を読んで再開して」だけ。前夜に保存した状態を翌朝の別セッションが即復元し、
  そのまま作業継続できた、というのが一番効いた場面。
- **長時間ジョブ / バッチの結果待ちで中断** — シミュレーションやバッチを投下して結果待ちになったとき、
  閉じる前に「次に見るべき output」と「次ステップ TODO」を確定させておく。
- **複数プロジェクトを並行** — セッション末尾に各プロジェクト dir で `--dir` 指定して実行し、
  それぞれの続きを独立に保存（継続性が安定する）。
- **複数日・複数再起動を跨ぐ長丁場** — ストレージ整理や大規模 migration など、再起動を挟む作業で
  随時 `SESSION-RESUME.md` / TODO を更新して途切れず継続する。
- **方針転換・意思決定の記録** — アプローチを巻き戻したときに、到達点と「なぜそう判断したか」を残し、
  次セッションの自分（と Claude）へ引き継ぐ。
- **コンテキスト圧縮対策 / token 節約** — 「現セッションでは検証コマンドだけ実行、重い解釈は次セッションで」
  のように分担を宣言し、解釈を再開後に回す。

### 記憶の扱い
公式 auto-memory（`~/.claude/projects/<dir slug>/memory/`）を正本として参照する。`<dir slug>` は対象
絶対パスの「英数字以外を `-` に置換」したもの。プロジェクト移動直後は旧 slug 下の memory も読み取りで参照する
（新規作成はしない）。

### Roadmap（将来構想 / 未実装）
現状は手動運用（`/session-resume` で生成 → 次セッションで読む）で完結します。以下は構想段階のアイデアで、
**まだ実装されていません**。

- **コンテキスト圧迫時の半自動代行** — 任意で Gemini / Groq などの API キーを設定しておくと、
  Claude Code 本体のコンテキストが圧迫されている状況で、terminal を全選択して貼り付ける → 外部 LLM が
  再開プロンプト生成を半自動で代行する、というモード。
- 上記は **opt-in**（キー未設定なら従来どおり手動で完結）を前提に検討中。

### 利用上の注意（Terms / Disclaimer）
本 Skill は現状有姿（**AS IS**）で提供され、いかなる保証もありません。**利用は自己責任**で。
`SESSION-RESUME.md` は実行のたび最新内容で**上書き**されます。作者は損害について責任を負いません。

### 引用（Citation）
役立った場合、引用やリンクでの言及を歓迎します（必須ではありません）。
> Eisaku Umeda (2026). *session-resume*. https://github.com/eUmeda/session-resume

### License
MIT — see [LICENSE](LICENSE).

---

## English

`session-resume` generates a **resume prompt** — work context, done / not-done tasks (status-classified),
file layout, design decisions, and a next-step recommendation — following **LLM prompting best practices**
(XML structure / context-first / MUST·MUST NOT), and writes it to `SESSION-RESUME.md` in the target directory.

The goal: in your next session you just say **"read `SESSION-RESUME.md` and resume"** and your context is back.

### Features
- `--dir <path>` makes the target explicit, so it keeps working **even right after you move or rename a project**
  (no dependency on the current working directory).
- Open TODOs are classified into `[actionable]` / `[deferred]` / `[blocked]`; if anything is actionable, it
  recommends one top pick + 1–2 alternatives with reasons.
- Runs **only when you explicitly ask** (Claude never auto-invokes it on its own judgment).

### Install
```sh
git clone https://github.com/eUmeda/session-resume.git
mkdir -p ~/.claude/skills
ln -s "$PWD/session-resume" ~/.claude/skills/session-resume
```
On environments without symlinks (e.g. Windows without developer mode), copy the directory instead:
`cp -R session-resume ~/.claude/skills/session-resume`.

Then run `/session-resume [--dir <path>] [optional notes]` in Claude Code.

### Output
- A `SESSION-RESUME.md` in the target dir (it survives directory moves, so you can resume location-independently)
  plus a short summary in the chat.
- Your next session resumes from just: "read `SESSION-RESUME.md` and continue."

### Real-world use cases
These are the patterns the author has actually used in daily Claude Code sessions since spring 2026. The common
thread: **have Claude generate it on the spot, then have the next session read it and pick up where you left off.**

- **Close at night → resume next morning (most common)** — run `/session-resume` at the end of a session;
  next morning just say "read `SESSION-RESUME.md` and resume." A fresh session restores last night's state
  immediately and continues — this was the single most useful case.
- **Pause while a long job / batch is running** — when you've launched a simulation or batch and are waiting
  on results, lock in "which output to check next" and the "next-step TODO" before closing.
- **Juggling multiple projects** — at session end, run it per project with `--dir`, saving each thread
  independently for stable continuity.
- **Multi-day work across several restarts** — for storage cleanups or large migrations that span reboots,
  keep `SESSION-RESUME.md` / a TODO file updated so the work never loses its thread.
- **Recording pivots and decisions** — when you roll an approach back, record where you ended up and *why* you
  decided that, handing it off to your future self (and Claude).
- **Context-compaction insurance / token savings** — declare a split like "this session runs only the
  verification commands; the heavy interpretation happens next session," and defer interpretation to the resume.

### How memory is handled
It reads the official auto-memory (`~/.claude/projects/<dir slug>/memory/`) as the source of truth. `<dir slug>`
is the absolute target path with every non-alphanumeric character replaced by `-`. Right after a project move it
will also *read* memory under the old slug (it never creates new files there).

### Roadmap (ideas / not yet implemented)
Today everything works via manual operation (generate with `/session-resume`, read it next session). The items
below are **ideas only — not implemented yet**.

- **Semi-automatic handoff under context pressure** — optionally set an API key (e.g. Gemini / Groq); when
  Claude Code's own context is getting tight, select-all in the terminal and paste it, and an external LLM
  drafts the resume prompt semi-automatically.
- This is being considered as **opt-in** (with no key set, it stays fully manual as today).

### Terms / Disclaimer
Provided **AS IS** with no warranty; **use at your own risk**. `SESSION-RESUME.md` is **overwritten** with the
latest content on every run. The author is not liable for any damages.

### Citation
If it helps, a citation or a link is appreciated (not required).
> Eisaku Umeda (2026). *session-resume*. https://github.com/eUmeda/session-resume

### License
MIT — see [LICENSE](LICENSE).

---
🤖 Built and maintained with [Claude Code](https://claude.com/claude-code).

> 注 / Note: `references/examples.md` は合成サンプル（架空プロジェクト）です。`docs/index.html` は人間向けの
> 使い方ページ。/ `references/examples.md` holds synthetic samples (fictional projects); `docs/index.html` is the
> human-facing usage page.
