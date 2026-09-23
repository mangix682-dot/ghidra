---
description: Evidence-first review of binary-codegraph, run entirely by the main agent (no subagents), archived under docs/reviews/<date>/
argument-hint: [target] [--fix] [--quick]
---

# bcg-audit — binary-codegraph 定域审查(主代理直审)

User arguments (may be empty): `$ARGUMENTS`

## Constants

| Name | Value |
| --- | --- |
| `REPO` | `G:/codingnet/ghidra/external-repo/binary-codegraph` |
| `REVIEWS` | `$REPO/docs/reviews` |
| `OUT_DIR` | `$REVIEWS/<YYYY-MM-DD>` (one directory per calendar day) |

## Hard rules — read before anything else

1. ⛔ **Do this review yourself, in this conversation.** Do **not** call the Agent tool, do **not**
   invoke the `code-review` / `security-review` skills (they fork to a background agent), do **not**
   call Workflow. If you catch yourself wanting to delegate a sweep, write the sweep as a script
   under the scratchpad and run it — a mechanical check you can *run* beats a reader you have to
   trust. This is the whole point of this command versus `/bcg-review`.
2. ⛔ **`REPO` is a separate git repository**, excluded from the cwd repo (`G:/codingnet/ghidra`)
   via `.git/info/exclude`. **Every** git command must carry `-C "$REPO"`. A bare `git diff` sees
   the ghidra repo and gives you an empty or plainly wrong change set.
3. ⛔ **Never `git add` / `git commit` / `git stash`** in `REPO`. Not the archive file, not a fix,
   not "just to see what staging would do" — several gates key off index/HEAD state and staging
   silently changes what they report. Simulate staging instead (see §2.3).

## 判据 · Scope contract

Inlined from `$REPO/docs/workflows/REVIEW_SCOPE.md` (stable — no need to open it).

**Fixed artifacts.** Every conclusion must be grounded in this exact pair; changing it requires an
explicit decision by the project owner:

- `D:\unityinstall\2021.3.45f2\Editor\Data\PlaybackEngines\windowsstandalonesupport\Variations\win64_player_development_il2cpp\UnityPlayer_Win64_player_development_il2cpp_x64.pdb`
  (`8cf5bd2883455edeb472d14bbd02d94d9ef58644b80df0f009173c81a51cecbc`)
- `D:\unityinstall\2021.3.45f2\Editor\Data\PlaybackEngines\windowsstandalonesupport\Variations\win64_player_development_il2cpp\UnityPlayer.dll`
  (`78415ef1844e21a6d6bdc76aa8e0840b26904301c672f92452e753108c2e3de2`)

**Perspective.** Start from the results the current implementation produces. The central question is
whether those results show the present path is sound, or whether continuing on it is unreasonable and
demands a code / model / API / architecture adjustment. When a result is wrong, inconsistent, or
surprising, trace it back through the implemented path — check whether the code read, classified,
converted, counted, joined, or attributed the fixed artifact incorrectly. ⛔ Do **not** assume an
unexpected result proves the *design* wrong before ruling out an implementation or
format-interpretation error. Then: reproduce → compare against anchors / independent oracles /
closure rules / downstream expectations → trace the discrepancy end-to-end → decide whether the cause
is an implementation error, missing functionality, an inadequate data model, or an architectural
boundary that results show must change → recommend the **smallest** adjustment that makes the current
path correct and complete.

Prioritize errors touching **identities, domains, ownership, cardinalities, partitions, joins, or
downstream behavior**.

**Valid findings — a finding needs at least one:**

1. Reproduced with current code on the fixed PDB/DLL.
2. The result disagrees with a frozen anchor, an applicable independent oracle, byte closure, or
   another already-accepted result.
3. Functionality the current stage requires is missing, incomplete, or cannot represent data that
   actually occurs in the fixed artifacts.
4. Measured results show an API, data model, ownership boundary, or architecture must change.
5. The result is internally inconsistent or unreasonable — lost record identity, a failed declared
   partition, incompatible counts for the same measured universe.

Each finding must state: the observed result, the code path causing it, its consequence for this
artifact pair, and the smallest relevant correction.

**⛔ Out of scope — never a finding on its own:** unobserved theoretical hash collisions; constructed
counterexamples that don't occur in the fixed artifacts; speculative malformed PDB/PE input; extreme
exception paths outside the supported workflow; exhaustive CodeView/MSVC/PDB-version coverage;
generalized hardening or future-proofing with no current requirement; "could fail" claims with no
reproduced failure, missing required function, or unreasonable current result. **These must not block
a stage from closing.**

**Mutations and unexecuted paths.** A mutation may prove an already-required check is effective; it
does **not** by itself expand the supported input space or justify a finding. ⛔ Don't invent
mutations to manufacture out-of-scope failures. A branch with zero executions on the fixed artifacts
is a **coverage fact**, not a defect — unless the current stage explicitly requires it be supported
and tested.

**Outcome.** If nothing clears the bar, report that and let the work proceed. ⛔ Do not reframe
planned, already-registered work as a newly discovered defect, and do not hold completion on
unobserved theoretical risk.

## Step 1 · Parse arguments and determine scope

**Flags**: `--fix` (apply fixes after reporting), `--quick` (skip the full `tools/verify.py` chain;
run only the gates the diff touches). **Target**: whatever remains — path, branch, or commit range,
resolved relative to `REPO`.

If no target was given, resolve in order:

1. `git -C "$REPO" status --porcelain` non-empty → review the **uncommitted changes**.
2. Otherwise → review the branch against its merge-base:
   `git -C "$REPO" merge-base HEAD origin/master` (fall back to local `master`; the branch usually
   has no upstream, so don't use `@{u}`).
3. Both empty → say "`REPO` has nothing pending review", print branch + HEAD, and stop. ⛔ Do not go
   hunting for something to review.

Record `git -C "$REPO" diff --stat <scope>` file/line counts — the archive needs them.

## Step 2 · Ground yourself in results **before** reading the diff

The contract requires findings be reproduced, not reasoned into existence. So **results first, diff
second** — whether the results come from your own run (§2.2) or from output the user attached (§2.1).
This ordering is the point of the command — inverting it produces exactly the speculative findings
the contract bans.

### 2.1 If the user attached gate output, use it — don't re-run

The user often pastes gate output another AI (or an earlier session) already produced. **When such
output is attached, treat it as the empirical grounding and do not re-run those gates.** Re-running
burns minutes and usually reproduces what you were handed. Say in the archive which results were
supplied rather than run by you.

⚠ But attached output is **evidence to be judged, not testimony to be trusted**. Before building on
it, sanity-check it — and note that the contract's condition 5 (internally inconsistent or
unreasonable results) makes an implausible *gate report* a legitimate finding in its own right.
Signals that something is off:

- a gate reporting **0 items checked** yet passing (the repo's own stated recurring failure);
- counts that contradict each other, or that contradict the diff you can see;
- **all-green on a change that provably touches something a gate is supposed to protect** — e.g. a
  frozen `SHA256SUMS`, a `frozenAt.commit` coordinate, a load-bearing path constant;
- checks/failed totals that don't add up, or a summary line that disagrees with the per-check lines;
- output referring to files, lanes, or shas that don't exist in the current tree;
- results that look like they came from a **different scope** than the one under review (wrong
  branch, wrong sha, stale paths).

When something looks unreasonable, ⛔ don't just discard it and move on. Decide which it is:

1. **The run was bad or stale** → re-run that specific gate yourself and use your own output.
2. **The gate or framework is wrong** — it measures the wrong thing, passes on an empty set, or
   cannot express the condition it claims to enforce → that is a **finding** (contract conditions 4
   and 5), often more valuable than whatever the diff was doing. Confirm it by running the gate's
   `--selftest`, which is designed to expose exactly this.

⛔ Never silently "fix up" numbers that look wrong, and never report an attached result as CONFIRMED
unless you ran it yourself — mark supplied-but-unverified findings `PLAUSIBLE` and say who ran them.

### 2.2 Otherwise, run the gates yourself

This repo's review surface is ~56 gate scripts under `tools/` (`*_gate.py`, `verify*.py`), of which
**54 carry a `--selftest`** that ablates the gate to prove it can go red.

- Full chain: `python tools/verify.py` (the real commit gate; `--only <lane>` is a dev shortcut —
  lanes are listed in `python tools/verify.py --help`).
- With `--quick`, or when the full chain is too slow: run only the gates whose constants, inputs, or
  evidence pages the diff touches, plus `python tools/verify_evidence_freeze.py` (both the freeze
  check and `--links`).
- **When the diff edits a gate script, run that gate's `--selftest` too.** A gate can stay green on
  live data while its own positive case has become unsatisfiable — invisible unless you run it.
- **Empty-set false green is this repo's stated recurring failure.** A gate reporting 0 items checked
  is red, not a pass.

### 2.3 Triage every red before you call it a finding

A red does **not** imply a defect here. Classify each by *what git state the gate reads* — getting
this wrong is the easiest way to file a bogus finding, or to miss the real one:

| Gate reads | Typical call | Goes green when | Is it a finding? |
| --- | --- | --- | --- |
| worktree | `open(path)` / `Path.read_bytes()` | immediately | **yes**, if still red |
| **index** | `git show :<path>`, `git ls-files` | after `git add -A` | **no** — staging artifact |
| **HEAD** | `git show HEAD:<path>` | after an actual commit | **no** — staging artifact |
| **ancestor commit** | `git show <frozenAt.commit>:<path>` | ⛔ **never** | **yes — the severe class** |

That last row is the one to hunt for. Evidence pages carrying a `frozenAt.commit` resolve their
`sources[].path` against **that past commit**, so such a path is a *historical coordinate*, not a
live pointer. Rewriting it — by a rename, a path migration, or a search-and-replace — is unfixable
by committing, because a rename never rewrites history. Same idea for `headBlobSha256` and
`SHA256SUMS`: those freeze **bytes**, and the toolchain cannot tell a cosmetic edit from a
substantive one, so a mass edit can silently rotate a signed-off freeze.

To distinguish "staging artifact" from "real", **simulate staging instead of doing it**: recompute
what the gate would see (e.g. hash the LF-normalized worktree bytes, since blobs are LF-normalized)
in a scratchpad script. ⛔ Never run `git add` to find out.

## Step 3 · Read the diff, residual-first

For a large or mechanical change (mass rename, path migration, constant bump, codemod), do **not**
read hundreds of near-identical hunks. Instead:

1. Write a scratchpad script that **normalizes away the intended mechanical transformation**.
2. Report only the **residual** — lines that changed for some *other* reason.
3. Review the residual by hand, and state the reduction in the archive
   ("N files reduce to K non-mechanical residuals").

This turns "187 files changed" into a handful of real decisions, and reliably surfaces collateral the
author did not intend (an encoding/BOM flip, a dropped line, a re-blessed pre-existing bug).
Sanity-check the normalizer itself: if it reports suspiciously many residuals, the normalizer is
usually wrong before the diff is.

For a small or hand-written change, just read it — with the fixed artifact pair in mind, tracing
whether identities, domains, ownership, cardinalities, partitions, or joins could be affected.

## Step 4 · Filter against the contract, honestly

For each candidate, name which of the five valid-finding conditions it satisfies. Can't name one →
**drop it**. Concretely: a "could break" with no reproduction → drop; a malformed-PDB /
hash-collision / unsupported-toolchain hypothetical → drop; a zero-execution branch → register as a
**coverage fact** in the archive, not as a finding; work already registered in a charter / ruling /
plan under `docs/reviews/**` → not a defect, say it's already registered and where.

Separate what survives into two buckets, kept visibly apart:

- **缺陷 (defects)** — reproduced, or a demonstrated inconsistency against a frozen anchor / oracle /
  closure rule;
- **判断 (judgment)** — altitude, mechanism, and cleanup observations. Legitimate to raise, marked as
  such, and never outranking a reproduced defect.

Distinguish **pre-existing** from **introduced**. If the diff merely touched a line that was already
wrong, say so plainly — it changes the fix's urgency and whose lane it belongs to.

## Step 5 · Report

Call `ReportFindings` once, most severe first, with `verdict` set: `CONFIRMED` only when **you**
actually ran something that reproduces it; `PLAUSIBLE` for reasoned/judgment items **and for anything
resting on run output you were handed but did not re-run**. Each `failure_scenario` must carry the
**command and the output** — and say whose run it was — because in this repo an assertion without a
reproduction is not worth much.

With `--fix`: report first, then apply fixes, then re-report with each finding's `outcome`
(`fixed` / `skipped` / `no_change_needed`). Prefer the **smallest** correction that makes the current
path right. After fixing, re-run the affected gates **and their selftests**, and put the new output
in the archive.

## Step 6 · Archive

Write the file **whether or not there are findings** — a clean review is a record worth keeping.

Path: `$REVIEWS/<YYYY-MM-DD>/<YYYY-MM-DD>-<slug>-audit-<shortsha>.md`

- Date from `date +%Y-%m-%d` (local machine date), same date in directory and filename.
- `mkdir -p` the day directory.
- `<slug>`: kebab-case, ≤5 words, describing **what was reviewed**.
- `<shortsha>`: `git -C "$REPO" rev-parse --short HEAD`.
- Suffix `-r2` / `-r3` if the name exists. ⛔ **Never overwrite.**
- ⛔ Review documents live **only** inside a day directory. `docs/reviews/evidence/` is for run
  artifacts and is off limits for this document. The `<YYYYMMDD>-<NNN>-<topic>.md` form belongs to
  charter/ruling workflows — don't use it here.

Body in **Chinese**, identifiers/paths/commands verbatim, matching house style (`⛔` prohibition,
`★` key point, `⚠` caveat, `📌` cost note):

```markdown
# <slug> 定域审查(<YYYY-MM-DD>)

| | |
| --- | --- |
| 仓库 | `binary-codegraph` |
| 分支 / HEAD | `<branch>` / `<shortsha>` |
| 审查范围 | <未提交改动 / <base>..HEAD / 路径>,共 N 个文件、+X/−Y 行 |
| 判据 | `docs/workflows/REVIEW_SCOPE.md` |
| 结论 | <N 条 finding(缺陷 a / 判断 b) 或 未发现问题> |

## 摘要
<3–6 行:改动在做什么、主线问题是什么。没问题就写清楚「跑了哪些门、为什么判定干净」。>

## 门禁结果
| 门 | 结果 | 来源 | 归类 |
| --- | --- | --- | --- |
| `<gate>` | 绿 / 红(`checks=N failed=M`) | 本次实跑 / 用户提供 | 缺陷 / 暂存态 / 待提交 / ⛔ 提交也修不好 |
<① 把 §2.3 的四类分清楚——「红但只是没 git add」必须写明,否则读者会当成缺陷去修。
 ② 「来源」列必须如实——用户提供而未复跑的,⛔ 不得写成本次实跑。
 ③ 若判定某条提供的结果不可信,写明为什么、以及是重跑了还是把它变成了 finding。>

## Findings
### F1 · <一句话结论> · `<category>`
- **位置**:`<file>:<line>`
- **问题**:<缺陷本身>
- **复现**:<跑的命令 + 实际输出;⛔ 没跑过就写「未复现,依据推断」并说明依据>
- **判据**:<命中五条 valid-finding 中的哪一条>
- **引入性**:<本次引入 / 既存(本次仅触碰)>
- **建议**:<最小修法;短的话贴 diff 片段>
- **处置**:<--fix 时写 已修复/已跳过/无需改动;否则写 未处理>

## 覆盖范围与未覆盖项
- 已审:<文件/模块;大改动写明残差归约「N 个文件 → K 处非机械残差」>
- 已跑:<门禁清单及结果>
- 覆盖事实:<固定 artifact 上零执行的分支等——登记,⛔ 不算缺陷>
- 未审 / 存疑:<没跑的门、需要 Ghidra/MSVC 环境的项、推断而非实测的结论>
```

## Step 7 · Reply

One or two sentences: finding count (缺陷 vs 判断) and the archive's full path. `ReportFindings`
already rendered the details — ⛔ do not restate them. If any gate is red **only** because the work is
unstaged or uncommitted, say so in one clause so the user doesn't go fix a non-problem.

⛔ Do not `git add` / `git commit` the archive unless the user explicitly asks.
