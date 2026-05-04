---
name: screenshots
description: Use when capturing screenshots of a web app from any browser tool (chrome-devtools, playwright, gstack /browse, /qa, /design-review, manual screenshot dumps). Triggers on words "screenshot", "screen capture", "take_screenshot", "browser_take_screenshot", "save the screenshot", "show me the page", "snap the UI". Triggers when about to call any *take_screenshot tool, when saving a PNG/JPG capture of a page, when comparing before/after UI states, when collecting QA evidence, when debugging layout/visual issues. Use when deciding whether to open a screenshot for the user to review.
---

# screenshots

## Overview

All UI screenshots taken by an agent in this repo go into `.screenshots/` at the repo root, with a timestamped, slugified filename, split into two buckets: **qa** (final captures meant for the user to look at — opened automatically) and **debug** (working captures the agent uses for itself — never opened). The folder is gitignored. No screenshots anywhere else, no untimestamped names, no opening debug shots.

## Repo preconditions (verify once per session)

Before the first capture in a session, confirm both are in place. If either is missing, fix it before taking the screenshot.

1. **`.gitignore` contains `/.screenshots/`** — captures must never be committed. Check with `grep -n '^/\.screenshots/' .gitignore`. If absent, append the line under an "agent screenshot scratch" comment.
2. **`AGENTS.md` references this skill** — there must be a "Browser screenshots" section pointing at `.claude/skills/screenshots/` so every agent (not just the one that loaded this skill) follows the same convention. Check with `grep -n 'screenshots' AGENTS.md`. If absent, add a short section before "Build commands" linking to this skill and summarising the qa/debug split and filename format.

These two are part of the skill's contract. A repo where `.screenshots/` is tracked, or where AGENTS.md doesn't mention the skill, is a misconfigured repo — fix it, don't work around it.

## Folder layout

```
.screenshots/
  qa/      # final captures the user should review — OPEN these
  debug/   # agent's own working shots — do NOT open
```

Both subfolders are created on demand. `.screenshots/` is in `.gitignore`.

## Filename format

`YYYYMMDD-HHMMSS-<slug>.<ext>`

- Local time, zero-padded, no separators in date/time.
- `<slug>` is kebab-case, lowercase, ASCII only, ≤ 60 chars, describes what is shown — page + state, e.g. `pratiche-list-empty-state`, `crea-pratica-dialog-validation-errors`, `mail-folder-inbox-after-archive`.
- `<ext>` matches the tool's output (usually `png`).

Examples:
- `.screenshots/qa/20260504-143012-pratiche-list-after-search.png`
- `.screenshots/debug/20260504-143055-sidebar-overflow-bug.png`

## Decision: qa or debug?

Default to **debug**. Promote to **qa** only when the screenshot is evidence the user will want to look at.

| Situation | Bucket | Open? |
|---|---|---|
| Final capture for `/qa`, `/qa-only`, `/design-review`, `/canary` | qa | yes |
| "Here's what the new screen looks like" — handing work back to user | qa | yes |
| Before/after pair on a UI change the user requested | qa | yes (both) |
| Reproducing a bug the user reported, to confirm fix | qa | yes (the fixed one) |
| Agent checking its own work mid-task ("did the dialog open?") | debug | no |
| Iterating on a layout, taking 10 shots in a loop | debug | no |
| Verifying a selector exists before clicking | debug | no |
| `/investigate` working captures while hunting a bug | debug | no |

If unsure: **debug**. Better to under-open than spam the user with Preview windows.

## Workflow

1. **Before calling any screenshot tool**, decide the bucket and build the filename:
   - timestamp: `date +%Y%m%d-%H%M%S`
   - slug: derive from what is on screen, not from the URL or the task ID
2. Pass the absolute path under `.screenshots/{qa|debug}/` to the tool's `filePath` / `path` argument. If the tool only writes to a default location, move the file with `mv` immediately after.
3. Ensure the parent directory exists (`mkdir -p .screenshots/qa` etc.) — do this once per session, not per shot.
4. After the capture:
   - **qa** → run `open <path>` (macOS) so the user sees it. For a before/after pair, open both.
   - **debug** → do not open. Mention the path in your reply only if relevant.
5. Reference the file in your reply with a clickable path: `.screenshots/qa/20260504-143012-pratiche-list.png`.

## Quick recipe (bash)

```bash
TS=$(date +%Y%m%d-%H%M%S)
SLUG="pratiche-list-after-search"
BUCKET=qa   # or: debug
DIR=".screenshots/$BUCKET"
mkdir -p "$DIR"
OUT="$DIR/$TS-$SLUG.png"
# … pass "$OUT" as filePath to the screenshot tool …
[ "$BUCKET" = "qa" ] && open "$OUT"
```

## Tool-specific notes

- **chrome-devtools MCP** (`take_screenshot`): pass the absolute path as `filePath`. Default is `fullPage: false`; use `true` for layout-of-the-whole-page QA shots.
- **playwright MCP** (`browser_take_screenshot`): pass `filename` (relative is fine, it lands in the test output dir) and then `mv` it into `.screenshots/<bucket>/`.
- **gstack `/browse`, `/qa`, `/design-review`**: these skills already produce screenshots — relocate their outputs into `.screenshots/qa/` with the right name before reporting back, and `open` them.

## Common mistakes

| Mistake | Fix |
|---|---|
| Saving to `/tmp`, `~/Downloads`, or repo root | Always under `.screenshots/{qa,debug}/` |
| Filenames like `screenshot.png`, `1.png`, `Screen Shot …` | Always `YYYYMMDD-HHMMSS-<slug>.png` |
| Slug is the URL path or a task ID | Slug describes what's on screen |
| Opening every debug shot | Only `qa/` shots get `open` |
| Forgetting to open qa shots | Run `open <path>` before reporting back |
| Committing the folder | `.screenshots/` must be in `.gitignore` |
| Stale qa shots piling up | Old captures are local-only, gitignored — fine to leave; user can `rm -rf .screenshots` any time |

## Red flags — STOP

- About to write `take_screenshot` / `browser_take_screenshot` without a prepared `.screenshots/{qa,debug}/<timestamp>-<slug>.png` path → stop, build the path first.
- About to `open` a debug capture → don't.
- About to skip `open` on a qa capture → don't; the user asked for review-ready shots to surface automatically.
