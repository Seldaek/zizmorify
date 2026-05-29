---
name: add-zizmor
description: >-
  Add zizmor (GitHub Actions security analysis) CI to a repository and fix every
  finding it surfaces. Use when asked to "add zizmor", harden a repo's GitHub
  Actions workflows, pin actions to SHAs, or make workflows pass a security
  audit. Adds a zizmor workflow + a dependabot config, then iterates locally
  until zizmor reports no findings, and opens a PR.
---

# Add zizmor CI and fix all findings

Goal: add a `zizmor` GitHub Actions security-analysis workflow to a repo, then fix
**every** issue zizmor reports in the repo's existing workflows, verifying locally
until the run is clean, and open a PR. zizmor runs in `pedantic` mode, so the bar
is high — actions pinned to SHAs, minimal documented permissions, no template
injection, etc.

## Bundled templates (do not fetch from anywhere else)

This skill ships the two files to add — use them, they have no external dependency:

- `assets/zizmor.yml`  → goes to `.github/workflows/zizmor.yml`
- `assets/dependabot.yml` → goes to `.github/dependabot.yml` (only if the repo has none)

Both live in this skill's directory; copy them from there.

## Tooling & how you get zizmor's feedback

Detect what's available and pick the best feedback loop, in this order:

1. **zizmor available locally** (`zizmor --version`) — the fast path. Run it directly and iterate
   in place. Install if easy (`pipx install zizmor`, `uv tool install zizmor`,
   `cargo install zizmor`, or the `ghcr.io/zizmorcore/zizmor` container). Run **with online audits**
   for the authoritative check (matches the CI action):
   `GH_TOKEN=$(gh auth token) zizmor --persona pedantic .github`. Online audits catch
   `ref-version-mismatch`, `known-vulnerable-actions`, and `stale-action-refs` that the offline run
   (`--no-online-audits`) misses; use offline only for quick iteration.

2. **No local zizmor, but `gh` is available & authenticated** (`gh auth status`) — rely on the
   **zizmor workflow itself** for findings. Push the branch / open the PR, let `zizmor.yml` run, and
   read results with `gh run view <run-id> --log-failed` and `gh pr checks <n>`. Fix, push, re-read,
   repeat until the check is green. Slower (a CI round-trip per iteration) but fully functional.
   `gh` also resolves action SHAs via `gh api repos/OWNER/REPO/...`.

3. **No `gh` either** — work on a local checkout only. Add the zizmor workflow + dependabot and apply
   every fix from the playbook proactively (resolve action SHAs without `gh`, e.g.
   `git ls-remote --tags https://github.com/OWNER/REPO`). Commit to a branch, then **stop and hand off**:
   tell the user to push and open the PR, and that the zizmor check on the PR is the source of truth
   for any remaining findings. Do not push.

## Procedure

1. **Create a branch off the latest default branch:**
   `git fetch origin && git switch -c add-zizmor-dependabot origin/<default-branch>`.

2. **Add the zizmor workflow.** Copy `assets/zizmor.yml` to `.github/workflows/zizmor.yml`.
   - **Match the repo's indentation.** Inspect an existing workflow file and detect its
     indent unit (commonly 2 spaces; the bundled template is 4). If it differs, re-indent
     the copied `zizmor.yml` to the same unit so it reads like the surrounding files.
   - **Scope the `push` trigger to the branch you're targeting** (the template uses `main`).
     For a maintenance branch, set the pushed branch accordingly.
   - If the repo has no other workflows, the template is fine as-is.

3. **Add / update dependabot.**
   - No `.github/dependabot.yml` → copy `assets/dependabot.yml`.
   - Existing one that lacks a `cooldown` → add `cooldown:\n  default-days: 7` under the
     `github-actions` update block. Keep the repo's existing `interval` (don't increase
     update frequency on rarely-touched repos).

4. **Get zizmor's findings** using the best available loop (see *Tooling* above): run it locally if
   you can; otherwise push and read the workflow run via `gh`; otherwise apply the playbook fixes
   proactively without a feedback loop.

5. **Fix every finding** in the existing workflows (and any local composite actions under
   `.github/actions/**`) using the playbook below. Re-check and repeat until zizmor prints
   **“No findings to report.”** (or, in the gh-only loop, the PR's zizmor check is green). Validate
   YAML after edits
   (`python3 -c "import yaml,glob; [yaml.safe_load(open(f)) for f in glob.glob('.github/**/*.y*ml', recursive=True)]"`).

6. **Commit** on the branch:
   - Commit 1: add `zizmor.yml` + `dependabot.yml`.
   - Commit 2: "Harden existing workflows to pass zizmor".

7. **Push & open the PR** — *if `gh` is available*: `git push -u origin add-zizmor-dependabot` then
   `gh pr create`, summarize the changes, and **verify the zizmor check is green**
   (`gh pr checks <n>` / `gh run list --workflow zizmor.yml`). *If `gh` is not available*: stop after
   committing and tell the user to push and open the PR themselves.

8. **Repeat per maintained branch** if requested — one PR per branch, each with the zizmor
   `push` trigger scoped to that branch.

## Hard rules

- **Avoid** using zizmor `ignore`/suppression comments. Fix each finding with a real, explicit
  value.
- **Only comment the non-obvious.** Add an explanatory inline comment when a value is not the
  obvious safe default (e.g. `contents: write`, `persist-credentials: true`). Do **not** comment
  defaults like `contents: read` or `persist-credentials: false`.
- **Pin to the latest current release**, not whatever is referenced today. Resolve it with:
  `tag=$(gh api repos/OWNER/REPO/releases/latest --jq .tag_name)` then
  `gh api repos/OWNER/REPO/commits/$tag --jq .sha`. Write `uses: OWNER/REPO@<full-sha> # <tag>`,
  and the comment **must match the tag name exactly** (e.g. `# v6.0.2`, or `# 2.37.1` if the
  tag has no `v`).
- **For workflows that create PRs, comments, or push**, use the `gh` CLI (`gh pr create`, `git
  push`) instead of third-party actions, and set `persist-credentials: true` (with a comment) on
  checkout so the token is available — don't re-init git auth later.
- **Preserve behavior.** Don't change what a workflow does beyond what a finding requires; don't
  bump pins that are already correct.

## Finding-by-finding playbook

- **unpinned-uses** — pin the action to the SHA of its latest release (see Hard rules).
- **ref-version-mismatch** — the `# comment` must name a tag that actually points to the pinned
  SHA. If a proper release exists, re-pin to it and match the comment.
- **stale-action-refs** — the SHA doesn't point to any tag. Re-pin to the latest release SHA. If
  the action has **no tags at all**, it can't be made clean — surface this to the user (a branch
  pin with a `# branchname` comment is the best available); don't suppress.
- **known-vulnerable-actions** — bump the pin to a fixed (latest) release.
- **artipacked / credential persistence** — add `persist-credentials: false` to read-only
  checkouts. For a checkout in a job that pushes/opens PRs, use `persist-credentials: true` with a
  comment explaining why.
- **excessive-permissions** — add explicit `permissions`. Default to workflow-level
  `permissions:\n  contents: read`. Grant write scopes (`contents: write`, `packages: write`,
  `pull-requests: write`, …) **only on the specific job that needs them**, each with a comment.
- **undocumented-permissions** — add an explanatory comment to each non-default permission entry.
- **concurrency-limits** — add a workflow-level (or job-level, for matrix branches) block:
  `concurrency:\n  group: ${{ github.workflow }}-${{ github.ref }}\n  cancel-in-progress: true` or
  ask the user what concurrency rule makes sense as this is very project specific.
- **template-injection** — never expand `${{ ... }}` (matrix / env / steps / github / inputs
  contexts) directly inside a `run:` block. Move the value into the step's `env:` and reference the
  shell variable instead:
  - bash/sh: `env: { FOO: ${{ matrix.foo }} }` then use `$FOO`.
  - inside a single-quoted program (e.g. `jq`): pass it in, e.g. `jq --arg foo "$FOO" '… $foo …'`.
  - `shell: cmd`: reference it as `%FOO%`.
  Env vars already defined in a workflow-level `env:` block are available directly as `$NAME` in
  run blocks — just drop the `${{ env.NAME }}` wrapper.
- **github-env** (dangerous use of `$GITHUB_ENV` / `$GITHUB_PATH`) — don't write dynamic values to
  the env file. Prefer **step outputs**: write to `$GITHUB_OUTPUT` and have consumers read
  `steps.<id>.outputs.<name>` (for a composite action, declare `outputs:` and update callers to use
  the action's outputs). Always quote `>> "$GITHUB_OUTPUT"`.
- **anonymous-definition** — give every job a `name:` (and the workflow a top-level `name:`).
- **deprecated `::set-output`** — replace with `echo "name=value" >> "$GITHUB_OUTPUT"`.

## Iterating efficiently

Make the edits, then **re-run zizmor after every batch** — fixing one finding sometimes clears or
reveals others (e.g. removing a `${{ }}` from a cmd step drops its `misfeature` noise). Keep going
until the only things left are genuinely unfixable (no-tag actions, required `cmd` shells), which
you call out explicitly in the PR description instead of hiding.
