---
description: Run /code-review on binary-codegraph (default effort xhigh) and archive the findings under docs/reviews/<date>/
argument-hint: [target] [effort] [--fix|--comment]
---

# bcg-review — binary-codegraph code review + archive

User arguments (may be empty): `$ARGUMENTS`

## Constants

| Name | Value |
| --- | --- |
| `REPO` | `G:/codingnet/ghidra/external-repo/binary-codegraph` |
| `REVIEWS` | `G:/codingnet/ghidra/external-repo/binary-codegraph/docs/reviews` |
| `OUT_DIR` | `$REVIEWS/<YYYY-MM-DD>` (one directory per calendar day) |
| default effort | `xhigh` |

> ⛔ **`REPO` is a separate git repository.** It is excluded from the cwd repo
> (`G:/codingnet/ghidra`) via `.git/info/exclude`. The repository under review is `REPO`,
> **not** the ghidra repo at cwd. Therefore **every** git command in this review must carry
> `-C "$REPO"` (`git -C "$REPO" diff …`). A bare `git diff` only sees the ghidra main repo and
> yields an empty or plain wrong change set.

## Step 1 · Parse arguments

Split `$ARGUMENTS` on whitespace into three buckets:

- **effort**: one of `low` / `medium` / `high` / `xhigh` / `max` / `ultra`.
  If the user gave none, use **`xhigh`** (this command's own default — do *not* fall back to
  "whatever level was used last time").
- **flags**: `--fix` / `--comment` / `--post` / `--no-post` etc., passed through verbatim.
- **target**: whatever remains (PR number / branch name / path). If absent, determine it in step 2.

## Step 2 · Determine the review scope

If the user gave an explicit target, use it (resolve path-like targets relative to `REPO`).
Otherwise resolve in this order:

1. `git -C "$REPO" status --porcelain` is non-empty → review the **uncommitted changes**
   (working tree + index).
2. Otherwise → review **the current branch's commits relative to `origin/master`**:
   `git -C "$REPO" log --oneline $(git -C "$REPO" merge-base HEAD origin/master)..HEAD`,
   diffing against that same merge-base. (The branch usually has no upstream, so don't use
   `@{u}`; if `origin/master` is unavailable, fall back to local `master`.)
3. Both empty → **do not** go hunting for something to review. Tell the user "`REPO` has nothing
   pending review", print HEAD and the branch name, and stop.

Before starting the review, run `git -C "$REPO" diff --stat <scope>` once and record the file /
line counts — the archive document needs them.

## Step 3 · Run the review

Invoke the `code-review` skill via the Skill tool with args `<target> <effort> <flags>`, then
**follow its instructions in full** — including its required `ReportFindings` reporting step.
The "every git command needs `-C "$REPO"`" constraint above stays in force for the whole review.

With `--fix`: report first, then fix, and record each finding's actual outcome
(fixed / skipped / no_change_needed). The archive document must state the outcome per finding.

## Step 4 · Archive into `OUT_DIR`

**Write this file whether or not there are findings** — a zero-finding review is still a record
worth keeping.

Path: `$REVIEWS/<YYYY-MM-DD>/<YYYY-MM-DD>-<slug>-review-<shortsha>.md`

- Date: `date +%Y-%m-%d` (local machine date, not the commit date). The **same** date is used for
  both the directory name and the filename prefix.
- Create the day directory if it does not exist yet (`mkdir -p`); older days already have one.
- `<slug>`: kebab-case, summarizing **what was reviewed** (taken from the branch name or commit
  subject, e.g. `tpi-identity-gate2`); for a PR target use `pr<N>`. Max 5 words.
- `<shortsha>`: `git -C "$REPO" rev-parse --short HEAD`.
- Layout rules for `docs/reviews/`:
  - review documents live **only** inside a `<YYYY-MM-DD>/` day directory — never loose at the
    top level of `docs/reviews/`;
  - `docs/reviews/evidence/` is for run artifacts (logs, JSON, SHA256SUMS) and is **off limits**
    for this document;
  - day directories also contain charter / ruling / plan documents named
    `<YYYYMMDD>-<NNN>-<topic>.md`, written by other workflows. This command always uses the
    `<YYYY-MM-DD>-<slug>-review-<shortsha>.md` form — do not mix the two styles.
- If that filename already exists (re-review of the same sha on the same day) → append `-r2`,
  `-r3` after the slug. **Never overwrite** an existing document.

Write the body in **Chinese** (keep code, identifiers and paths verbatim — this matches every
existing document under `docs/reviews/`), following this skeleton:

```markdown
# <slug> 代码审查(<YYYY-MM-DD>,effort=<effort>)

| | |
| --- | --- |
| 仓库 | `binary-codegraph` |
| 分支 / HEAD | `<branch>` / `<shortsha>` |
| 审查范围 | <未提交改动 / <base>..HEAD / PR #N / 路径>,共 N 个文件、+X/−Y 行 |
| effort | `<effort>` |
| 结论 | <N 条 finding(P0 a / P1 b / P2 c) 或 未发现问题> |

## 摘要
<3–6 行:改动在做什么、审出来的主线问题是什么。没问题就写清楚"覆盖了什么、为什么判定干净"。>

## Findings

### F1 · <一句话结论> · `<category>`
- **位置**:`<file>:<line>`
- **问题**:<缺陷本身,一两句>
- **触发场景**:<具体输入/状态 → 错误输出或崩溃>
- **建议**:<改法;短的话贴 diff 片段>
- **处置**:<--fix 时写 已修复/已跳过/无需改动;否则写 未处理>

<One section per finding, ordered most severe first.>

## 覆盖范围与未覆盖项
- 已审:<文件/模块清单>
- 未审 / 存疑:<没看的部分、需要跑测试或人工确认的点>
```

## Step 5 · Reply

Reply in one or two sentences: the number of findings plus the full path of the archive file.
The finding details were already rendered to the user by `ReportFindings` — **do not** restate
them in the reply.

⛔ Do not `git add` / `git commit` the archive file unless the user explicitly asks.
