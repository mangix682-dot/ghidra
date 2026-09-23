---
description: Countersign a binary-codegraph closing/ruling slice — independently re-run the evidence, audit gate strength, issue an ACCEPT/REJECT verdict, and archive it as a registration MD
argument-hint: [target] [--quick]
---

# bcg-countersign — binary-codegraph closing countersignature

User arguments (may be empty): `$ARGUMENTS`

## Constants

| Name | Value |
| --- | --- |
| `REPO` | `G:/codingnet/ghidra/external-repo/binary-codegraph` |
| `REVIEWS` | `$REPO/docs/reviews` |
| `OUT` | `$REVIEWS/<YYYY-MM-DD>/<YYYYMMDD>-<NNN>-<topic>-accept.md` (use `-reject.md` on REJECT) |

⚠ **The command is in English; the archive document it produces stays in Chinese.** That is not a
style preference — the registration MD is grepped verbatim by the *next* slice's gate, and every
existing pinned string in this repo is Chinese (see Hard rule 5). Writing the archive in English
would red the downstream gate. Everything below — instructions, checklists, reasoning — is English;
only the Step 7 template body is Chinese.

## Hard rules — read before anything else

1. ⛔ **Countersigning is not reviewing.** `/bcg-audit` hunts defects. This command answers a
   different question: **"can I put my name on what this slice claims?"** The two commands treat
   user-supplied gate output in **exactly opposite** ways — audit may accept attached output to save
   time; a countersignature ⛔ **may not**. Evidence you did not re-run cannot be signed for; the
   re-run *is* the signature.
2. ⛔ **Do this yourself — no subagents.** Do not call the Agent tool or Workflow, do not invoke the
   `code-review` / `security-review` skills. If you want a mechanical sweep, write it as a script in
   the scratchpad and run it — a check you can *run* beats a reader you have to trust.
3. ⛔ **`REPO` is a separate git repository**, excluded from the cwd repo (`G:/codingnet/ghidra`) via
   `.git/info/exclude`. **Every** git command must carry `-C "$REPO"`. A bare `git diff` sees the
   ghidra repo and gives you a plainly wrong change set.
4. ⛔ **Never `git add` / `git commit` / `git stash`.** Several gates key off index/HEAD state and
   staging silently changes what they report. Write the registration MD and stop — ⛔ do not commit
   it unless the user explicitly asks.
5. ★ **The registration MD is a machine-read contract surface for the next slice.** The next slice's
   gate will **grep this document's strings verbatim**. Confirmed instance:
   `l030_l2c_v26_contract_charter_gate.py` P1/P2 assert, against the R7 ACCEPT doc, the literals
   `复审结论：**ACCEPT**`, `可以按 charter §5`, `四码 producer 仍不存在`, `不创建 v2.6 candidate`.
   So the §0 verdict sentence and the §6 boundary sentences **must be verbatim-stable** — reworded
   phrasing falsely reds the downstream gate. After writing, run
   `git -C "$REPO" grep -ln "<the verdict sentence you wrote>" -- tools/` to find who reads it.

## Contract · what a countersignature actually certifies

The signature covers three **mutually independent** axes. Missing one means you cannot sign.

- **A · Claims match reality** — every claim in the commit message, the charter, and the user's
  narration is true in the repo. Especially the **negative** ones: "not created", "not activated",
  "v2.5 unmodified", "frozen evidence untouched".
- **B · Gates match the closing requirements** — every item the charter *itself* declares must be
  mechanically checked (usually in its final "closing / out of scope" section) is in fact covered by
  some gate, and that gate **can go red**.
- **C · Citations match their sources** — every constant and ledger the charter asserts traces back
  to a frozen source, and the arithmetic closes.

**Verdicts you may issue:**

| Verdict | Meaning |
| --- | --- |
| `ACCEPT` | All three axes clear; the next slice may start |
| `ACCEPT (with corrections)` | Design is sound, but axis B has gate-strength gaps that change none of this slice's rulings and can be fixed alongside the next slice |
| `REJECT` | Axis A has a false claim, or axis C has a transcription error, or an axis B gap would carry a wrong design into the next slice |

⛔ **The forbidden fourth outcome: reformatting the attached green output and calling it ACCEPT.**
That is a rubber stamp, not a countersignature.

## Step 1 · Determine what is being countersigned

**Flags**: `--quick` (run only the lanes this slice touches; skip the full `tools/verify.py` chain).
**Target**: whatever remains — a commit, a slice id (e.g. `#2-LC`), or a charter path.

With no target, resolve in order:

1. `git -C "$REPO" log --oneline -5` — if the newest commit's message looks like
   `<NNN> #<lane>: <topic>`, countersign it;
2. else if `git -C "$REPO" status --porcelain` is non-empty → say "uncommitted changes present, the
   countersign target is ambiguous", list them, and stop;
3. both empty → print branch + HEAD, say "nothing pending countersignature", stop. ⛔ Do not go
   hunting for something to sign.

Record: `git -C "$REPO" rev-parse --abbrev-ref HEAD`, `--short HEAD`,
`git -C "$REPO" show --stat --name-status <commit>`, and `git -C "$REPO" status --porcelain`
(is the worktree clean?).

Then locate the slice's three artifacts: the **charter** (the design being signed), the **gate**
(its enforcement body), and the **prerequisite ACCEPT** (the authorization it claims to derive from).

## Step 2 · Re-run everything — ⛔ do not accept attached output

The user will almost always attach gate output. **Read it, use it to decide what to run, then run it
yourself.**

- The slice's dedicated gate: run both `python tools/<gate>.py` and `python tools/<gate>.py --selftest`;
- The slice's lane: `python tools/verify.py --only <lane>`;
- The freeze surface: `python tools/verify.py --only evidencefreeze`;
- Without `--quick`, and when the environment allows, the full `python tools/verify.py`.

📌 A single lane often takes minutes; the full chain needs a Ghidra/MSVC environment. Anything you
could not run goes in §7 "not covered" — ⛔ never default it to green.

★ **Recount the denominator independently.** The user says "52/52" — count the actual `g.ok(` calls
in the gate source, grouped by section (e.g. `4+15+5+4+11+13 = 52`), and confirm the denominator is
not just what the gate reports about itself. Same for the selftest: ablation cases + positive
control + repo-fact ablations + coverage meta-check should equal the reported total. A mismatch is a
finding.

⚠ **Empty-set false green is this repo's stated recurring failure.** A gate reporting 0 items
checked is red, not a pass.

## Step 3 · Verify the negative claims

Positive claims are what gates cover. **Negative claims ("X was not touched") usually are not — check
git yourself:**

```bash
git -C "$REPO" diff --name-only <commit>~1 <commit> -- <paths claimed untouched...>
git -C "$REPO" log --oneline -1 -- <that path>      # who last actually modified it
```

Typical negative claims worth checking: no new schema directory created, active version constants
not switched, v2.5 / frozen evidence bytes unmoved, new record codes still unreachable in
producer/validator, frozen `SHA256SUMS` not rotated.

## Step 4 · Gate-strength audit (★ the substance of a countersignature)

Green is not the same as strong. Do all five; axis-B findings essentially all come from here.

### 4.1 Map each closing requirement onto a gate

Open the charter's closing-requirements section and **transcribe every "must be mechanically checked"
sentence into a list**. For each, name the gate and the specific check enforcing it. Anything you
cannot name is a gap — "the charter's own closing requirement is not mechanically checked" is the
most common and the most worth saying out loud before signing.

### 4.2 Classify the checks: self-referential text assertions vs repo-fact nails

Read each check's condition and sort it into two buckets:

| Class | Shape | Discriminating power |
| --- | --- | --- |
| **Self-referential text assertion** | `"some sentence" in charter` / `in s5` | ⚠ Low. The ablation "delete that sentence" reddens **by construction** — it proves the sentence is load-bearing, ⛔ not that the design is right |
| **Repo-fact nail** | reads a source enum, v2.5 bytes, another doc, a version constant | ★ High. Can actually catch repo-vs-charter divergence |

**Report the ratio** (e.g. "of 52 checks, ~40 self-referential, 12 repo-fact nails"). ⛔ Never let a
`56/56 selftest` be read as "the design is self-proven" — this repo already recorded the same class
at `2026-08-13-r7-bridge-axis-ruling-review-e4983d5.md:271`: *"they read like independent closure
criteria, but in fact cannot independently fail."*

Also look for **checks that should have crossed sources but didn't**: if the same gate already
derives ground truth from another file for some checks, but pins a ledger with a bare
`literal in charter`, that one should be rewritten to regex the number out of its source and compare.

### 4.3 Which git state does each freeze nail read (⛔ easiest to miss)

For every check claiming "bytes unchanged / no drift", determine what it actually reads:

| Reads | Typical call | Proves | Enough? |
| --- | --- | --- | --- |
| worktree | `Path.read_bytes()` | current worktree | depends on intent |
| index | `git show :<path>` | staged state | ⚠ changes the moment you stage |
| **HEAD** | `git show HEAD:<path>` | **only "worktree == HEAD"** | ⛔ **HEAD *is* the commit being signed — if that commit changed the file, this still goes green** |
| **fixed ancestor** | `git show <freeze commit>:<path>` | genuinely "unchanged since the freeze" | ★ this is a real freeze anchor |

On finding a HEAD anchor, ask one more question: **is that same set of bytes frozen anywhere else in
the repo?** (`git -C "$REPO" grep -ln "<path>" -- "*SHA256SUMS*"`). If not, that check is the **only**
mechanical guard on those bytes and the gap is real; if so, downgrade it to redundant. Smallest fix:
re-anchor to a fixed ancestor commit constant.

### 4.4 Every mirror of a frozen closed set

When a charter freezes a **closed key set** (record codes, required counts keys, a reason enum), that
set usually has **several mirrors** in the repo — one in the schema, one in a producer constant, one
in a validator, one in a golden generator. Enumerate them with
`git -C "$REPO" grep -ln '"<one key>"' -- "*.java" "*.cs" "*.py"` and check whether the gate watches
each. **Nailing the schema side while missing a producer-side constant table** is this repo's typical
gap — add the keys to the missed mirror and the gate stays fully green.

### 4.5 Discriminating power: the digit-perturbation scan

For self-referential ledger checks, run the mechanical power test this repo already uses (see R7
ACCEPT §2): write a scratchpad script that **decrements every numeric token in the charter by one,
one at a time**, re-running the gate's `audit()` for each, and count how many go red.

- The ones that stay green are mostly dates, section numbers, and prose copies already enforced
  elsewhere — ⚠ staying green ≠ a vacuous check, but look at each one;
- If a **load-bearing ledger** does not redden under perturbation, that is a real gap; fix it per the
  last paragraph of 4.2.

## Step 5 · Constant provenance and arithmetic closure

For every number and constant the charter cites, do two things:

1. **Trace it**: `git -C "$REPO" grep -n "<number>" -- docs/ tools/` — confirm it comes from a frozen
   evidence page or the prerequisite ACCEPT, and is not an orphan constant appearing only in this
   charter;
2. **Close it**: hand-compute every conservation identity (`A = B + C + D`). ⛔ Gates typically assert
   only that the literal string is present; they do not verify the arithmetic.

A mismatch is an axis-C finding → **REJECT** (a transcription error gets consumed as ground truth by
the next slice).

## Step 6 · Verdict

Issue one of the three from the Contract table. Grading rules:

- False claim on axis A, or transcription error on axis C → `REJECT`;
- An axis-B gap that **would carry a wrong design into the next slice** → `REJECT`;
- An axis-B gap that only makes the gate weak and changes none of this slice's rulings →
  `ACCEPT (with corrections)`, stating that the corrections **ship alongside the next slice rather
  than blocking it** (blocking converts gate debt into schedule debt);
- All three axes clear → `ACCEPT`.

⛔ Do not reframe planned work already registered in a charter / ruling / plan as a newly discovered
defect. ⛔ Do not block a closing on unreproduced theoretical risk (hash collisions, malformed input,
exhaustive toolchain-version coverage).

## Step 7 · Write the registration MD

Write it **whatever the verdict** — a clean countersignature is a record worth keeping.

- Path: `$REVIEWS/<YYYY-MM-DD>/<YYYYMMDD>-<NNN>-<topic>-accept.md`; `-reject.md` on REJECT.
  Take `<NNN>` and `<topic>` from the charter being signed (e.g. `030` / `l2c-v26-candidate-contract`).
- Date from `date +%Y-%m-%d` (local machine), same date in directory and filename.
- Suffix `-r2` / `-r3` if the name exists. ⛔ **Never overwrite.**
- ⚠ This command's output **is** a registration document and uses the `<YYYYMMDD>-<NNN>-…` form —
  a **different document type** from `/bcg-audit`'s `<YYYY-MM-DD>-<slug>-audit-<sha>.md`. ⛔ Do not
  mix the two forms.

Body in **Chinese** (see the note at the top of this file), identifiers/paths/commands verbatim,
house style (`⛔` prohibition, `★` key point, `⚠` caveat, `📌` cost note):

```markdown
# <NNN> `#<lane>` <topic> 核签登记（<YYYY-MM-DD>）

## 0. 裁定

复审结论：**<ACCEPT / ACCEPT(附补正) / REJECT>**。`<shortsha>` 作为 <topic> 的收片 checkpoint；
本登记只记录核签裁定及其边界，<list what this registration does not do>。
获得 ACCEPT 只表示可以 <the specific next authorization>，⛔ 不代表 <explicitly list what is unfinished>。

<★ These two sentences are the downstream gate's grep targets — the verdict string and the
 "⛔ 不代表" string must be precise and stable.>

承重材料：
- [<charter>](<relative link>)；
- [<gate>](<relative link>)；
- [<prerequisite ACCEPT>](<relative link>)。

本次核签在 `<shortsha>` 的工作树上执行，工作树 <干净 / 有 N 处未提交改动>。

## 1. 声明核对（A 轴）

| 声明 | 出处 | 核验方式 | 结果 |
| --- | --- | --- | --- |
| <one row per positive/negative claim> | 提交信息 / charter §N / 用户口述 | <the command actually run> | 属实 / ⛔ 不符 |

<Negative claims must state they were checked with git diff — ⛔ "the gate is green" is not evidence.>

## 2. 独立复跑（本次实跑，⛔ 非采信附带输出）

| 门 | 本次读数 | 与用户所述 |
| --- | --- | --- |
| `<gate>.py` | `checks=N failed=0` | 一致 / 不一致 |
| `<gate>.py --selftest` | `checks=M failed=0` | 一致 / 不一致 |
| `verify.py --only <lane>` | 全绿（N 档） | 一致 |

分母独立复算：门源码 `g.ok(` 实际 <per-section arithmetic> = N 条，与报出分母相符。

## 3. 门强度审查（B 轴）

**收片要求落点**：charter §<N> 列 <K> 条"必须机检"，逐条落点为 <…>；⛔ 未落点的 <…>。

**判据构成**：<N> 条中约 <a> 条自指文本断言、<b> 条仓内事实钉子。
⚠ 自指判据的消融构造性必红，`--selftest <M>/<M>` ⛔ 不等于"设计已自证"。

**冻结锚点**：<each check that reads HEAD vs a fixed ancestor, and the consequence>。

**封闭集镜像**：<every mirror coordinate of the closed set, and whether the gate watches it>。

## 4. 常量溯源与算术（C 轴）

| 常量/账目 | charter 位置 | 溯源坐标 | 算术 |
| --- | --- | --- | --- |
| `<identity>` | `<file>:<line>` | `<frozen evidence / prerequisite ACCEPT>:<line>` | 闭合 / ⛔ 不闭合 |

## 5. 补正项（不改变本片裁定）

### 补正 1 · <one-line conclusion>
- **位置**：`<file>:<line>`
- **问题**：<the gate-strength gap itself>
- **复现**：<command + output; if not run, write "未复现，依据推断" and state the basis>
- **实际影响**：<does it already produce a wrong result today — usually "fact is true, gate is weak">
- **最小修法**：<smallest change>
- **建议时机**：与 <next slice> 同批 / 签字前必须修

## 6. 核签不解除的边界

- <HOLDs this ACCEPT ⛔ does not thaw; coverage facts it does not re-rule; implementation gaps it must not be read as completing>

## 7. 本次未覆盖项

- <gates not run, items needing a full Ghidra/MSVC environment, conclusions inferred rather than measured, ruling-layer judgments that are ⛔ not mechanically checkable>
```

## Step 8 · Reply

Three to five sentences: the verdict, one sentence per axis, the number of corrections, and the full
path of the registration MD. If there are corrections, close by asking whether to patch the gate now.
⛔ Do not restate the registration body. ⛔ Do not `git add` / `git commit` the registration file
unless the user explicitly asks.
