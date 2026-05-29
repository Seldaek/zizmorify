# zizmorify

A [Claude Code](https://claude.com/claude-code) **skill** that adds a
[zizmor](https://docs.zizmor.sh) GitHub Actions security-analysis workflow to a repository
and then fixes **every** finding zizmor surfaces in the existing workflows.

zizmor runs in `pedantic` mode, so the bar is high. The skill drives an agent to:

- add a `zizmor` workflow (`.github/workflows/zizmor.yml`) and a `dependabot.yml` — the latter on
  the default branch only, since dependabot ignores it on other branches; a non-default branch is
  covered by a `target-branch` entry added to the default branch's `dependabot.yml`,
- run zizmor and iterate until it reports **no findings** — pinning actions to commit SHAs at
  their latest release, tightening and documenting `permissions`, adding `concurrency`,
  setting `persist-credentials`, removing template injection, replacing `$GITHUB_ENV` writes
  with step outputs, switching PR-creating steps to the `gh` CLI, etc.,
- open a PR (or hand off if `gh` isn't available).

It fixes issues with **explicit, real values — never with zizmor `ignore`/suppression comments** —
and surfaces anything that genuinely can't be fixed (e.g. an action with no release tags) instead
of hiding it.

## What's here

```
SKILL.md                  # the skill instructions (frontmatter + playbook)
assets/zizmor.yml         # the workflow that gets added to target repos
assets/dependabot.yml     # the dependabot config added on the default branch
```

## Install

Drop the skill into your Claude Code skills directory — either user-wide (`~/.claude/skills/`)
or per-project (`<repo>/.claude/skills/`):

```bash
git clone https://github.com/Seldaek/zizmorify ~/.claude/skills/zizmorify
```

The skill registers under the name in its frontmatter, so it's invoked as **`/add-zizmor`**
regardless of the directory name.

## Use

In Claude Code, run `/add-zizmor` (or just ask it to "add zizmor to this repo"). The skill
adapts to your environment: it runs `zizmor` locally if it's installed, otherwise iterates via
the workflow run using the `gh` CLI, and otherwise prepares a local branch for you to push.

## Credits

zizmor is by [zizmorcore](https://github.com/zizmorcore/zizmor). This repo just packages a
workflow + an agent playbook around it.
