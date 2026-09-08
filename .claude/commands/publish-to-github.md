---
description: Secret-scan, then publish this project to GitHub with Pages, README, and repo About
argument-hint: [github repo URL or owner/name — omit to use existing origin]
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, WebFetch
---

# Publish this project to GitHub

Target repository: **$ARGUMENTS**
(If that is empty, use the existing `origin` remote. If there is no `origin` either, stop and ask
the user for the repo URL before doing anything else.)

Work through the phases in order. **Phase 1 is a gate: nothing is pushed until it passes.**
Report what you did at the end as a short checklist with the live Pages URL.

---

## Phase 0 — Establish the target and the tooling

1. `git rev-parse --is-inside-work-tree` — if this is not a repo, `git init` and set `main` as the branch.
2. Resolve the target repo:
   - An argument that looks like `https://github.com/<owner>/<name>[.git]` or `<owner>/<name>` wins.
   - Otherwise read `git remote get-url origin`.
   - If the argument names a *different* repo than an existing `origin`, **stop and ask** which to use.
     Never silently repoint a remote.
3. Check for the GitHub CLI: `command -v gh && gh auth status`.
   - **If `gh` is available and authenticated**, use it for the repo-settings work in Phases 4 and 5.
   - **If it is not**, do not try to install it and do not invent an API token. Do everything you can
     with plain `git`, then hand the user exact click-by-click instructions for the parts that need
     the GitHub UI. Say clearly which steps they must finish by hand.
4. Confirm the remote repo exists (`gh repo view <owner>/<name>` or `git ls-remote`). If it does not
   exist and `gh` is available, ask the user whether to create it and whether it should be **public**
   or **private** — never guess, and note that GitHub Pages on a private repo needs a paid plan.

---

## Phase 1 — Sensitive-data scan (BLOCKING — run before any push)

A push is effectively irreversible: rewriting history does not remove data from forks, caches, or
anyone who already cloned. So scan first, and treat any real finding as a hard stop.

Scan **tracked files, untracked files, and existing git history** — a secret already committed on an
earlier commit is still exposed even if the working tree is clean.

1. If `gitleaks` or `trufflehog` is installed, run it (`gitleaks detect --no-banner --redact`) and
   include the output. If neither is installed, do not install anything — fall back to the greps below.
2. Filename sweep — anything matching these should not be committed:
   `.env*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `*.keystore`, `id_rsa*`, `*.ppk`,
   `credentials`, `*credentials*.json`, `service-account*.json`, `.npmrc`, `.pypirc`,
   `.aws/`, `.ssh/`, `*.sqlite`, `*.db`, `*.bak`, `*.log`, `.DS_Store`, editor/IDE dirs.
3. Content sweep across tracked + untracked files, and across history with
   `git log -p --all` piped through the same patterns:
   - Provider keys: `AKIA[0-9A-Z]{16}`, `ASIA[0-9A-Z]{16}`, `sk-[A-Za-z0-9]{20,}`,
     `sk-ant-`, `ghp_`, `gho_`, `ghu_`, `ghs_`, `github_pat_`, `glpat-`, `xox[baprs]-`,
     `AIza[0-9A-Za-z_-]{35}`, `SG\.[A-Za-z0-9_-]{22}`, `-----BEGIN [A-Z ]*PRIVATE KEY-----`
   - Generic assignments: `(api[_-]?key|apikey|secret|token|passwd|password|pwd|access[_-]?key|
     private[_-]?key|client[_-]?secret|bearer|authorization)\s*[:=]\s*['"][^'"]{8,}`
   - Connection strings with inline credentials:
     `(postgres|postgresql|mysql|mongodb(\+srv)?|redis|amqp|ftp|ssh)://[^/\s:]+:[^@\s]+@`
   - Internal/PII leakage: hardcoded internal hostnames and private IPs
     (`10\.`, `192\.168\.`, `172\.(1[6-9]|2[0-9]|3[01])\.`), real employee emails, staff names,
     customer records, ticket/system identifiers from internal tools.
4. **Project-specific check for this repo** (see @CLAUDE.md): the only permitted network call in
   `index.html` is the FormSubmit endpoint, and `FORMSUBMIT_ENDPOINT` must still be the literal
   `YOUR_EMAIL@example.com` placeholder. If it holds a real address, that address becomes public on
   push — flag it and ask before continuing. Also re-run the constraint greps from `CLAUDE.md` so a
   broken hard constraint is not what gets published.
5. Confirm a sane `.gitignore` exists covering the filename patterns above; create or extend it if not.

**Triage and report.** Print a table of every hit: file, line, what it looks like, and your judgement
— *real secret* / *placeholder or example* / *false positive*. Then:

- **Real secret in the working tree** → STOP. Do not push. Tell the user to remove it and rotate the
  credential (rotation matters even pre-push, because it may already be in a backup or another clone).
  Offer to move it to an ignored `.env` or an environment variable.
- **Real secret in git history** → STOP. Do not push. Explain that a plain deletion commit does not
  help, that the credential must be rotated regardless, and offer to rewrite history with
  `git filter-repo` (destructive — get explicit confirmation first).
- **Only placeholders / false positives** → say so explicitly and continue.

Never continue past a real finding on your own judgement. Getting the user's explicit go-ahead is the
only way past this gate.

---

## Phase 2 — README

Read the existing `README.md` (and @CLAUDE.md) before writing. **Preserve anything the user wrote by
hand** — improve and fill in, do not flatten. `CLAUDE.md` is instructions for Claude; the README is
for humans, so translate rather than copy.

It should cover, sized to the project — for a small demo keep it short, do not pad it:

- Project name, one-line description, and what it is *for* (state plainly that this one is an
  internal demo/training tool, not a production system).
- A **live demo** link to the Pages URL from Phase 3.
- A **screenshot** of the running board, captured fresh in Phase 2a below and embedded as
  `![alt](docs/screenshot.png)` under the demo link.
- How to run it locally (for this repo: `open index.html` — no build, no server, no dependencies).
- Features, and the notable constraints that explain the design (single file, vanilla JS,
  no persistence — a refresh resets the board, and that is intended).
- Any setup a user must do themselves, e.g. swapping `FORMSUBMIT_ENDPOINT` and completing
  FormSubmit's one-time email activation.
- Licence, if the user has one in mind — ask rather than assuming.

Do not invent features, benchmarks, badges, or a licence that does not exist.

---

## Phase 2a — Screenshot via Playwright MCP

Regenerate `docs/screenshot.png` on every publish, so the README never shows a stale board.

1. **Check the Playwright MCP server is connected.** It is configured in `.mcp.json` and provides
   `mcp__playwright__*` tools. If those tools are not available, the server failed to connect —
   MCP servers attach once at startup, so it cannot be revived mid-session. Tell the user to restart
   Claude Code rather than trying to reconnect it. Common causes:
   - `node`/`npx` missing. Node is installed via Homebrew at `/opt/homebrew/bin`; `.mcp.json` pins
     the absolute `npx` path and an explicit `PATH`, because `npx` is a `#!/usr/bin/env node` script
     and fails if `node` is not on the spawned process's PATH.
   - Chromium not downloaded: `npx playwright install chromium`.
2. Capture the board from the **local file**, not the Pages URL — Pages may not have deployed yet,
   and the local file is the copy you are about to publish:
   - `mcp__playwright__browser_resize` to **1440 × 900** — wide enough that all four columns are
     visible side by side, which is the point of the shot.
   - `mcp__playwright__browser_navigate` to the `file://` absolute path of `index.html`.
   - Let the seeded cards render before shooting.
   - `mcp__playwright__browser_take_screenshot` with `fullPage: true`, saved to `docs/screenshot.png`.
3. **Look at the resulting image** before committing it. It must show the four columns with seeded
   cards and at least one **Overdue** badge. A blank or half-rendered board is a failure — re-shoot,
   do not commit it.
4. This is the **one** permitted image in the repo. It lives in `docs/` and is referenced only from
   `README.md`. The "zero external resources / no image files" rule in @CLAUDE.md is a constraint on
   `index.html` — the app itself must still embed no images and load nothing over the network.

---

## Phase 3 — GitHub Pages via Actions

1. If `.github/workflows/deploy-pages.yml` already exists, read it and only fix what is actually
   wrong (outdated action versions, wrong branch, wrong upload path). Do not rewrite a working file.
2. If it does not exist, create it: checkout → `actions/configure-pages` → `actions/upload-pages-artifact`
   → `actions/deploy-pages`, triggered on push to the default branch plus `workflow_dispatch`, with
   `permissions: contents read / pages write / id-token write` and a `pages` concurrency group.
   For a static site with no build step, upload the repo root.
3. Pages source must be set to **GitHub Actions**. With `gh`:
   `gh api -X POST repos/<owner>/<name>/pages -f build_type=workflow` (use `-X PUT` if it already
   exists). Without `gh`, tell the user: *Settings → Pages → Build and deployment → Source: GitHub Actions*.
4. The resulting URL is `https://<owner>.github.io/<name>/` — you need it for Phases 2 and 5.

---

## Phase 4 — Commit and push

1. `git status` and `git diff` — show the user exactly what is about to be uploaded, including any
   file they may not realise is tracked.
2. Stage deliberately. Never `git add -A` without having reviewed the untracked list in Phase 1.
3. Commit with a real message describing the change.
4. Push to the target branch. If the remote has commits you do not, **stop and ask** — never
   `--force` without explicit permission.
5. Watch the deploy: `gh run watch` if available, otherwise point the user at the Actions tab. If it
   fails, read the logs and fix the cause; do not report success on a red run.

---

## Phase 5 — Repo About + homepage link

Set the repo description and attach the Pages URL as the homepage, so the link shows in the About box:

```sh
gh repo edit <owner>/<name> \
  --description "<one-line description matching the README>" \
  --homepage "https://<owner>.github.io/<name>/" \
  --add-topic <relevant topics>
```

Without `gh`: *repo page → About (gear icon) → Description + Website → tick "Use your GitHub Pages
website"*. Give them the exact text to paste.

---

## Finish

Verify the Pages URL actually serves the app (fetch it, or ask the user to open it — a 404 usually
means Pages source is still set to a branch instead of GitHub Actions). Then report:

- ✅/❌ per phase, with the secret-scan verdict stated explicitly
- The repo URL and the live Pages URL
- Anything the user must still do by hand (Pages source, About box, FormSubmit activation)

Do not claim a step succeeded that you did not actually verify.
