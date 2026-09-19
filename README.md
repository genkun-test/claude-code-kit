# claude-code-kit

The Claude Code configuration I actually run every day: two subagent definitions,
one `settings.json` key that stops a specific silent failure, and a `CLAUDE.md`
template. Small on purpose — copying it takes about a minute.

**But the config is the boring half. The list of silent failures below is the point.**

*日本語の説明は [下のセクション](#日本語) にあります。*

---

## Why shipping a config repo is not enough

Configuration is the copyable half. You can paste it, or hand it to the agent and
let it apply it, and as models get stronger "good defaults" drift toward the actual
defaults. That half does not stay a difference for long.

**What stays is what you stopped believing.**

Every item in this kit exists because something broke *without producing an error*.
Type check passed. Build passed. The agent's report said "Done." The content was wrong.

Those are the expensive ones, because the normal feedback loop — run it, see it fail,
fix it — never fires. So for each one the question that matters is not "what broke"
but **"why did it take so long to notice"**.

## 13 silent failures

### The four you can check against your own setup right now

These four are the free part of the long-form write-up, so they are reproduced here.

**A-1. The worktree's base ref is not the branch you are on.**
You are on a feature branch, hand the continuation to a subagent, and get back code
written as if your changes never existed. The default `baseRef` is `"fresh"` — branch
from the *remote* default branch (usually `main`), not from where you are.
*Why it takes so long to notice:* in a local repo with no remote configured, `"fresh"`
falls back to local HEAD, so it behaves correctly the whole time you are testing on a
personal repo. The moment you `git remote add origin`, the base ref changes without you
touching any setting. No error. Build still passes.
*Fix:* set `baseRef: "head"` (in `settings.json` here), and commit a checkpoint before
launching — `"head"` still means the HEAD *commit*, so uncommitted work is not handed over.

**A-2. Sharing `node_modules` by symlink kills the build.**
Symlink `node_modules` into each worktree to save install time and the build panics with
`Symlink [project]/node_modules is invalid, it points out of the filesystem root`.
Some bundlers (Turbopack among them) reject a `node_modules` symlink pointing outside the
project root.
*Why it takes so long to notice:* **`tsc --noEmit` passes with the symlink in place.** If
your check is a type check, nothing tells you until something actually builds.
*Fix:* run a real `npm install` per worktree — and put that fixed cost (tens of seconds to
minutes per agent) into the parallelize-or-not estimate up front. Splitting short work
across worktrees loses on install time alone.

**A-3. Judging "these tasks don't overlap" from the files they write.**
Three cleanly split tasks that don't mesh at integration time. "(1) define shared tokens
(2) restyle every page (3) rebuild the nav" write disjoint files — but (2) and (3) are
written *against the output of* (1).
*Why it takes so long to notice:* the overlap check looked at files each task **edits** and
missed the files each task **reads**. At launch the split looks correct.
*Fix:* include read dependencies in the overlap check. Don't split work that touches a
shared dependency layer — settle that layer with one agent first.

**A-4. Splitting while a design decision is still open.**
Each agent's output is internally coherent and mutually inconsistent. Subagents start with
zero context, so any "how should this be designed?" left inside a task gets answered
independently by each of them.
*Why it takes so long to notice:* every individual report comes back "Done." The
contradiction is only visible after integration — which makes this the most expensive one,
because the fix is redoing the work.
*Fix:* treat parallelism as a tool for the *apply-what's-decided* phase. Don't split design,
root-cause investigation, or spec work.

### The other nine

Same shape — symptom / actual cause / why it takes so long to notice / fix — in the
long-form version:

| # | Symptom |
|---|---|
| B-1 | Large amounts of code get written and never verified |
| B-2 | "This needs a human, so I couldn't verify it" — and the part that *was* verifiable got skipped too |
| B-3 | You integrate because the report said it passed |
| B-4 | The completion criterion you wrote cannot actually be checked |
| C-1 | The senior model's reasoning does not carry over to the implementing model |
| C-2 | A withdrawn plan still in the conversation becomes a decision the implementer has to make |
| C-3 | An agent definition is not usable by the session that just wrote it |
| C-4 | worktree isolation never even starts for cross-repository work |
| D-1 | The execution environment itself is non-deterministically broken |

## What's in here

| File | Where it goes | What it does |
|---|---|---|
| `settings.json` | `~/.claude/settings.json` | Sets the worktree base ref to the branch you are on. See A-1 — the default branches from the remote's default branch, and looks correct until you add a remote. |
| `CLAUDE.md.template` | `~/.claude/CLAUDE.md`, or per project | Standing instructions for "senior model designs, junior model implements": delegation thresholds, when to parallelize, what a handoff must contain. |
| `.claude/agents/spec-implementer.md` | your project's `.claude/agents/` | Implements a settled design and nothing else. Stops and reports the moment a design decision turns out to be open (see A-4). |
| `.claude/agents/perspective-auditor.md` | same | Audits with **exactly one perspective per agent**. Ask one agent to "review everything" and it goes shallow on everything and returns satisfied with the first thing it found. |
| `sample-spec.md` | anywhere | The shape of a spec worth handing to `spec-implementer`: decisions made / files in scope / completion criteria / areas not to touch. |

## Install

```bash
# 1. global
cp settings.json ~/.claude/settings.json          # merge if you already have one
cp CLAUDE.md.template ~/.claude/CLAUDE.md         # tune the thresholds for your setup

# 2. per project
mkdir -p your-project/.claude/agents
cp .claude/agents/*.md your-project/.claude/agents/
```

**The thresholds in `CLAUDE.md.template` will not fit you as-is** (how large a task has to
be before delegating, and so on). Measure once in your own setup before trusting them.

## Assumptions and limits

- **Some of it assumes Windows + PowerShell.** Translate those parts for macOS / Linux.
- The `model` / `effort` values in `.claude/agents/*.md` were chosen against the pricing
  and speed at the time of writing. Different plan, different model, different optimum —
  do not treat them as constants.
- **This is one person's operating record on one setup (n=1), not a benchmark.**

## The long version

The full 13 failures plus five measurement write-ups (what context actually costs, where
delegation stops paying for itself, a measurement that fooled me, agents stalling on empty
responses, one-perspective-per-agent auditing) are a book on Zenn:

<https://zenn.dev/genkunjc/books/claude-code-parallel-kit>

**It is written in Japanese.** All chapters are free to read; chapter 2 is the A-1…A-4
material reproduced above. The price tag (¥200) is a tip jar, not a paywall.

If you would read an English edition, **open an issue** saying so (or star the repo). That
is the only signal I have to decide whether to write one.

## License

MIT. Copy it as-is, commercially or otherwise.

---

<a id="日本語"></a>

## 日本語

AIコーディングエージェント（Claude Code）を毎日回すために実際に使っている設定一式です。
**中身は設定・エージェント定義・テンプレートだけ**で、どれもそのまま置けば動きます。

### なぜ設定だけ配るのか

設定は**コピーすれば済む側**だからです。AIに渡せば向こうが読んで適用しますし、
モデルが強くなるほど「良い設定」は既定値に近づいていきます。ここは差になりません。

**差になるのは「何を信じるのをやめたか」のほうです。**
このキットの各項目には、そうなった理由——実際に踏んだ、**エラーを出さずに壊れた**事故——が
1つずつ紐づいています。型チェックは通り、ビルドも通り、エージェントの報告には
「完了しました」と書いてあるのに中身が違う、という壊れ方です。

その13件を「症状／本当の原因／**なぜ発見が遅れるか**／対策」の形でまとめたものが下です。
**3番目が本体**で、そこだけは設定ファイルに落とせませんでした。

https://zenn.dev/genkunjc/books/claude-code-parallel-kit

（全章無料で読めます。並列実行まわりの4件は第2章に入っています。値札の ¥200 は、役に立ったときの乾杯用です）

### 入っているもの

| ファイル | 置き場所 | 何をするか |
|---|---|---|
| `settings.json` | `~/.claude/settings.json` | worktree の分岐元を「今いるブランチ」にする。既定は**リモートの main から分岐**で、リモート未設定だと一見動くので気づけない |
| `CLAUDE.md.template` | `~/.claude/CLAUDE.md` または各プロジェクト直下 | 「上位モデルが設計・下位モデルが実装」を常時適用させる指示書。委譲の閾値や並列化の条件を書いてある |
| `.claude/agents/spec-implementer.md` | プロジェクトの `.claude/agents/` | 確定した設計だけを実装する担当。設計判断が要るとわかった時点で手を止めて報告する |
| `.claude/agents/perspective-auditor.md` | 同上 | **1体につき視点を1つ**だけ渡して監査させる担当。1体に「全部見て」と頼むと全部浅くなり、最初に見つけた1件で満足して返ってくる |
| `sample-spec.md` | どこでも | `spec-implementer` に渡す設計書の見本。「決定事項／対象ファイル／完了条件／触ってはいけない領域」の4節 |

### 使い方

```bash
# 1. グローバル設定
cp settings.json ~/.claude/settings.json          # 既にあるならマージする
cp CLAUDE.md.template ~/.claude/CLAUDE.md         # 閾値は自分の環境に合わせて調整する

# 2. プロジェクトごと
mkdir -p your-project/.claude/agents
cp .claude/agents/*.md your-project/.claude/agents/
```

`CLAUDE.md.template` の**閾値（何ファイル以上で委譲するか、など）はそのままでは合いません。**
自分の環境で1度測ってから調整してください。数字の根拠は本のほうに書いてあります。

### 前提と限界

- **Windows + PowerShell を前提にした記述が一部あります。** macOS / Linux では
  そこだけ読み替えてください
- `.claude/agents/*.md` の `model` / `effort` は、**執筆時点の料金と速度で選んだ値**です。
  プランやモデルが変われば最適値も変わります。固定値として扱わないでください
- **ここにある設定は、この構成で回している1人ぶんの実測に基づくもの**です。
  n=1 の運用記録であって、ベンチマークではありません

### ライセンス

MIT。**そのままコピーして商用でも使ってかまいません。**
