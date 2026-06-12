---
name: session-resume
description: >-
  セッション中断時に、次回再開用の構造化プロンプトを LLM プロンプティング best practice に沿って生成し、
  対象プロジェクトディレクトリの SESSION-RESUME.md に書き出す skill。作業コンテキスト・完了/未完了タスク
  (status 分類)・ファイル構成・設計決定・次の一手 recommendation を出力する。対象 dir は `--dir <path>` で
  指定でき cwd 依存をやめたため、プロジェクトを移動/改名した直後でも正しく動く。user が明示的に依頼した
  ときにのみ起動し、Claude の自己判断では自発起動しない。
  Use when: セッション終了, 一旦閉じる, 再開プロンプト作成, session resume, context save, セッションを閉じる, /session-resume
argument-hint: "[--dir <path>] [補足メモ（任意）]"
allowed-tools: Bash, Read, Glob, Grep, Write, AskUserQuestion
---

# Session Resume Prompt Generator

セッション中断時に、次回セッションへコンテキストを引き継ぐ構造化プロンプトを生成し、**対象プロジェクト
ディレクトリの `SESSION-RESUME.md` に書き出す** skill。「次回再開用の prompt を LLM best practice に沿って
作って」を毎回手で指示する代わりに、**対象 dir の確定 → 状況収集 → best-practice 準拠プロンプト生成 →
ファイル書き出し + チャット要点表示** までを一括で行う。

## 設計の前提（重要）

- **記憶の正本は公式 auto-memory** = `~/.claude/projects/<対象 dir を slug 化>/memory/MEMORY.md`
  とその個別メモリファイル。旧 `_memory/`（長期記憶/短期記憶/working-memory）3-Layer プロトコルは廃止済
  → **新規作成しない**。既存の legacy `_memory/` が見つかった場合のみ **読み取り専用**で補助参照してよい
  (作成・更新はしない)。
- **cwd に依存しない**。収集対象は Step 0 で確定する `$DIR` であり、`pwd` を直接の正本にしない。
  プロジェクトを移動/改名した直後は live cwd が旧パスのまま残り、空の旧 dir を収集してしまうため
  (この skill が以前抱えていた最大の問題)。
- 状況収集の bash は **frontmatter で先読みしない**。frontmatter 内でコマンドを動的展開する記法（skill ロード時に評価される）は旧 cwd で
  走ってしまうため、収集はすべて Instructions の明示ステップ（`$DIR` 確定後）で実行する。
  （この説明自体が当該記法を含むとローダに実行されてしまうため、記法そのものは本文に書かない。）
- 生成プロンプトは **LLM プロンプティング best practice** に従う(下記)。Step 2 の MUST 一覧が必要な
  best practice をすべて含む。補足が必要な場合のみ、別 skill `LLM-prompting-bestpractice`
  (https://github.com/eUmeda/LLM-prompting-bestpractice) がインストールされていれば参照してよい（無ければ skip）。

## When to invoke / When NOT to invoke

- **invoke する**: user が「セッションを閉じる」「一旦終わる」「再開プロンプトを作って」「session resume」
  「context save」等と**明示的に依頼**したとき、または `/session-resume` を直接呼んだとき。
  この場合 Claude は Skill ツールからこの skill を起動してよい。
- **invoke しない（重要）**: 上記のような user の明示依頼が**無い**のに、Claude の自己判断で自発起動しては
  ならない。セッション中の通常作業の区切りごとに勝手に走らせない。
  （`disable-model-invocation` フラグは廃止し model 起動を許可したが、その代わりこの「明示依頼時のみ・
  自発起動禁止」を運用ルールとして MUST 守ること。両者は矛盾しない: 起動経路は開けつつ、起動の引き金は
  user の明示依頼に限定する。）

## Instructions

以下の手順で再開用プロンプトを生成すること。

### Step 0: 対象ディレクトリ `$DIR` の確定（cwd 非依存・最重要）

収集対象 `$DIR` を以下の優先順で確定する。**`pwd` を既定にしない**。

1. **明示引数 `--dir <path>`**: $ARGUMENTS に `--dir` があれば、その**直後から次のフラグ/補足メモ手前まで**を
   1 つのパスとして取り出し `$DIR` とする（パスに空白が含まれ得るので、空白で途中切りしないこと）。これが最優先。
2. 引数が無ければ **会話文脈から同定**する:
   - この会話で user が言及した作業ディレクトリ・移動先パスを **すべて列挙**する
     （「〜へ移動した」「Desktop に移した」「iCloud から出した」「改名した」等のフレーズを手掛かりに）。
   - 候補が **ちょうど 1 つで明確**ならそれを `$DIR` とする。移動が話題なら **移動先（新パス）**を選ぶ
     （live cwd の旧パスを選ばない）。
   - 候補が **0 個 / 複数 / 曖昧**な場合は、**推測せず即 AskUserQuestion** で対象 dir を 1 問だけ確認する
     （誤った場所を黙って収集するのを未然に防ぐ）。
3. 上記でも決まらない場合のみ、最後の手段として `pwd`。
   - **CAUTION**: `pwd` はプロジェクトを移動/改名していない場合にのみ安全。移動直後は live cwd が旧パスのまま
     残るため危険。少しでも疑わしければ `pwd` を使わず AskUserQuestion で確認すること。

確定後、**`$DIR` を実パスに置換して** sanity check を MUST 実行する
（**ダブルクォートを外さないこと**。空白を含むパスを保護するため。パスはクォートの内側に貼る）:

```bash
DIR="/絶対/パス/を/ここに"   # 例: DIR="/Users/you/projects/my-project"
DIR="${DIR%/}"
if [ ! -d "$DIR" ]; then echo "MISSING_DIR: $DIR"; else
  echo "DIR: $DIR"
  echo "entries: $(ls -A "$DIR" 2>/dev/null | wc -l | tr -d ' ')"
fi
```

- `MISSING_DIR` が出た / `entries: 0`（空）/ `$DIR` が「user が直前に『移動元』と述べた旧パス」に一致する、の
  いずれかなら、**AskUserQuestion で正しい対象 dir を 1 問だけ確認**してから次へ進む。
- sanity check を通った `$DIR` を以降のすべてのステップで使う。

### Step 1: コンテキスト収集（確定した `$DIR` に対して実行）

`$DIR` 基準で収集する（`$DIR` を実パスに置換。**クォートを外さない**。cwd 相対では実行しない）:

```bash
DIR="/絶対/パス/を/ここに"
proj="$HOME/.claude/projects"
# auto-memory の slug は「英数字以外をすべて - に置換」(harness の変換規則。空白・~・/ もすべて - になる)
slug=$(printf '%s' "$DIR" | sed 's#[^A-Za-z0-9]#-#g')
mem="$proj/$slug/memory"
echo "== auto-memory =="
if [ -d "$mem" ]; then echo "($mem)"; ls "$mem"/*.md 2>/dev/null
else
  # robust fallback: slug 変換の取りこぼし/プロジェクト移動に備え、basename で照合
  base=$(printf '%s' "$(basename "$DIR")" | sed 's#[^A-Za-z0-9]#-#g')
  echo "(derived-slug memory なし → basename '$base' で照合)"
  # glob はパス境界 (- は / 由来) でアンカーし、部分文字列の誤マッチ (例: my-base が *base に当たる) を防ぐ。
  # それでも複数ヒットした場合は推測で 1 つを選ばず、候補を列挙して AskUserQuestion で 1 問確認する (Step 0 と同じルール)
  hits=$(ls -d "$proj"/"$base"/memory "$proj"/*-"$base"/memory 2>/dev/null)
  [ -n "$hits" ] && printf '%s\n' "$hits" \
    || echo "  (該当なし → グローバル $proj/$(printf '%s' "$HOME" | sed 's#[^A-Za-z0-9]#-#g')/memory/MEMORY.md を確認)"
fi
# 状態ファイル
echo "== state files =="
find "$DIR" -maxdepth 2 \( -name 'CLAUDE.md' -o -name 'TODO.md' -o -name 'README.md' -o -name 'HISTORY.md' -o -name 'progress.md' -o -name 'SESSION-RESUME.md' \) -type f 2>/dev/null | head -12
# git
echo "== git =="
git -C "$DIR" log --oneline -3 2>/dev/null && echo "---" && git -C "$DIR" status -s 2>/dev/null | head -10 || echo "Not a git repo"
# legacy _memory（あれば読み取りのみ）
echo "== legacy _memory (read-only) =="
out=$(find "$DIR" -path '*/_memory/*' -name '*.md' -type f 2>/dev/null | head -8)
[ -n "$out" ] && printf '%s\n' "$out" || echo "none"
```

**移動/改名直後の auto-memory（任意・該当時のみ）**: 移動先 dir 由来の memory がまだ無い場合、記憶は移動元の
slug 下に残っていることがある。会話文脈で旧パスが分かるなら、それも**読み取りのみ**で確認する:

```bash
OLDDIR=""   # 旧パスが会話文脈で分かる場合のみ設定。不明なら空のまま
if [ -n "$OLDDIR" ]; then
  oldmem="$HOME/.claude/projects/$(printf '%s' "$OLDDIR" | sed 's#[^A-Za-z0-9]#-#g')/memory"
  [ -d "$oldmem" ] && { echo "== pre-move auto-memory (read-only: $oldmem) =="; ls "$oldmem"/*.md 2>/dev/null; }
fi
```

採否ルール: **新パスの memory を優先**。新パスに無く旧パスにあれば旧パスを読み取りで採用（新規作成はしない）。
両方ある場合は新パスを正本としつつ、旧パス側の未完了 TODO は次回 recommendation に**漏らさず引き継ぐ**。

収集後、現在のセッションの作業内容を分析する:

1. **作業ディレクトリ**: 確定した `$DIR`（絶対パス）
2. **コンテキスト維持ファイル**: 次セッションで MUST 読み込むべきファイルを特定（絶対パス）
   - **公式 auto-memory**: 上で見つけた `MEMORY.md` + 関連する個別メモリ。無ければグローバル
     `~/.claude/projects/<$HOME の slug>/memory/MEMORY.md`。移動直後は旧パス slug 下の memory も読み取りで参照。
   - 対象 dir に書き出した `SESSION-RESUME.md`（前回分があれば）
   - プロジェクト固有の CLAUDE.md
   - README.md / TODO.md / HISTORY.md / progress.md など状態管理ファイル
   - (legacy `_memory/` が在る場合のみ、読み取り補助として任意で挙げる。MUST にはしない)
3. **完了タスク**: このセッションで完了した作業をリストアップ
4. **未完了タスク**: 残りを `- [ ]` 形式でリストアップし、各タスクに **status を分類** する:
   - `[actionable]`: 即着手可能。外部依存なし
   - `[deferred]`: user 主導再開のもの (例: 改名議論、心理的に重い判断)。Claude 側から推さない
   - `[blocked]`: 外部入力・データ収集・user の手動 step 待ち
5. **ファイル構成**: 主要なファイル・ディレクトリのツリー構造
6. **設計決定**: 次セッションで MUST 維持すべき重要な判断事項
7. **次回 recommendation**: actionable TODO が 1 件以上あれば、**最有力 1 件 + alternative 1-2 件** を理由付きで選定する。優先度判定の軸:
   - **完了までの時間** が短い (5-30 分) ものは即完了感 + 着手しやすさで上位
   - **既存実装との対称性 / 整合性** を回復するもの (例: 片側にしかない機能を反対側にも揃える) は上位
   - **別セッション専用 spec が既に書かれている** ものは spec 引き継ぎが効くので上位
   - infra・機能拡張・データ蓄積前提 のものは下位

### Step 2: 構造化プロンプト生成（LLM best practice 準拠）

以下のテンプレートに沿って再開プロンプトを生成する。下記 MUST は LLM プロンプティング best practice
(XML 構造化 / context-first・query-last / outcome-first 指示 / MUST・MUST NOT 明示) を反映している。

**MUST**:
- XML セクションタグを使用して Claude がセクションを正確に識別できるようにする
- `<context_files>` を先頭に配置する（long context 最適化: ~30%精度改善）
- `<next_action>` を末尾に配置する（query 末尾配置で精度向上）
- コンテキスト維持ファイルは絶対パスで記述する（`$DIR` 由来の絶対パス）
- 完了/未完了タスクは具体的に記述する（曖昧な要約にしない）
- 未完了タスクは `[actionable]` / `[deferred]` / `[blocked]` の status を末尾に付与する
- actionable TODO が 1 件以上ある場合は `<recommendation>` セクションを `<next_action>` 内に必ず含め、最有力 1 件 + alternative 1-2 件を理由付きで提示する
- `<next_action>` は分岐させる: actionable TODO がある場合は "user に recommend 提示 → 1 質問だけ確認 → 着手" の flow を明記、無い場合は "user 指示待ち" のシンプルな flow
- 記憶参照は公式 auto-memory を第一に挙げる（legacy `_memory/` は在れば補助のみ）

**MUST NOT**:
- セッション固有の一時的な情報（エラーメッセージ、デバッグログ等）を含めない
- コードの内容をプロンプトに埋め込まない（ファイルパスで参照する）
- 推測や仮説を事実として記述しない
- `[deferred]` タスクを recommendation に含めない (user 主導再開モード尊重)
- 一律に「user 入力を待つ」とだけ書かない (actionable TODO がある場合は能動的に recommend する)
- 新規 `_memory/` ディレクトリ作成を再開プロンプトに指示しない（廃止済プロトコル）

#### Output Template

> 注: 下の ``` フェンス内は **そのまま SESSION-RESUME.md に出力する再開プロンプト本体**。`[Case A]` `[Case B]`
> の角括弧マーカーは「現在の Claude が分岐を選ぶための指示」であり、出力には含めない。`<next_action>` 内の
> 番号付き手順は **次セッションの Claude 向けの指示**として出力に含める。

```
以下のコンテキストで作業を再開してください。

<context_files>
MUST: 以下のファイルを最初に読み込んでコンテキストを復元すること:
1. [絶対パス] — [このファイルの役割の1行説明]
2. [絶対パス] — [役割]
</context_files>

<working_directory>[絶対パス]</working_directory>

<project>
## プロジェクト概要
[1-2文のプロジェクト説明]

## 現在の状態
完了済み:
- [具体的な完了項目]

未完了:
- [ ] [具体的な未完了項目] `[actionable]`
- [ ] [具体的な未完了項目] `[deferred]` — [defer 理由]
- [ ] [具体的な未完了項目] `[blocked]` — [何待ちか]
</project>

<file_structure>
[主要ファイル/ディレクトリのtree構造]
</file_structure>

<constraints>
## 重要な設計決定
- [MUST維持すべき判断とその理由]
</constraints>

<next_action>
まず context_files のファイルを読み込んで状況を把握すること。

[Case A: actionable TODO がある場合は以下の <recommendation> ブロックを含める]

このセッションには未完了の actionable TODO が残っている。**user の入力を待たず**、下記 recommendation を能動的に提示すること:

1. 下記 <recommendation> の「最有力」を 1 文で提示 (タスク名 + 所要時間 + 完了成果物)
2. AskUserQuestion で「これで始める? / alternative? / 別の指示?」を 1 質問だけ確認
3. user 同意なら即着手。alternative を選んだらその spec を提示して着手。「別の指示」なら user の input を待つ
4. `[deferred]` タスクは Claude 側から提案しない (user 主導再開を尊重)

<recommendation>
## 最有力 (推奨)
**[タスク名]** — [所要時間目安] / [完了成果物]
理由: [なぜこれを最初にやるべきか、1-2 行]

## alternative
- **[タスク名]** — [所要時間目安] / [理由 1 行]
- **[タスク名]** — [所要時間目安] / [理由 1 行]
</recommendation>

[Case B: actionable TODO が無い、または全て deferred/blocked の場合]

未完了の actionable TODO は無い。context_files を読み込んだ後、user の指示を待つこと。
</next_action>
```

### Step 3: 出力（SESSION-RESUME.md 書き出し + チャット要点表示）

1. 生成したプロンプトを **`$DIR/SESSION-RESUME.md`** に Write で書き出す（対象 dir に同伴させることで、
   `/clear`・端末クローズ・ディレクトリ移動後も残り、cwd に依存せず再開できる）。
   既存の `SESSION-RESUME.md` がある場合は上書きでよい（最新の再開状態が正本）。
2. **チャットにも要点サマリーと生成プロンプト全文を表示**する。サマリーには
   「対象 dir / 完了 n 件・未完了 m 件 (actionable k 件) / 最有力 recommendation 1 行 / 書き出し先パス」を含める。
3. **次セッションの再開方法を 1 行で添える**: 「次回は `$DIR/SESSION-RESUME.md` を読ませて再開（または内容を
   冒頭プロンプトに貼る）。`/session-resume --dir <$DIR>` で再生成も可」。ファイルは移動後も残るので場所非依存。
4. $ARGUMENTS に `--dir` 以外の補足メモが含まれる場合は、その内容を `<next_action>` の補足情報として反映する。

## Style Reference

生成スタイルの参考として [references/examples.md](references/examples.md) を参照。
これらは合成サンプル（架空プロジェクト）で、情報量と具体性のバランスの基準となる。
