---
name: iterate
description: Do the next logical piece of work — one issue, from pick to merged PR to closed
argument-hint: "[#issue] [--auto]"
allowed-tools: Bash(gh:*), Bash(git:*), Bash(bash ${CLAUDE_PLUGIN_ROOT}/skills/iterate/scripts/*), Bash(make:*), Bash(go:*), Bash(helm:*), Bash(docker:*), Bash(uv:*), Bash(pytest:*), Bash(npm:*), Bash(node:*), Bash(shellcheck:*), Read, Edit, Write, Glob, Grep
---
<!-- Canonical /iterate (language-operator#932), shared by every repo through the langop
     plugin. Nothing here is per-repo: how to test a change comes from the `## Testing`
     section of the consuming repo's CLAUDE.md. -->

# Iterate: do the next logical piece of work

One run handles **one issue**, from selection to a merged PR and a closed issue, then stops.
For continuous work, use `/loop /iterate` or a scheduled agent.

## Context

Read:
- `CLAUDE.md`
- `README.md`
- `.claude/MEMORY.md`, if it exists

## Arguments

`$ARGUMENTS` may contain:
- *(nothing)*: pick the next issue (see below).
- `#N` or `N`: work issue N.
- `--auto`: unattended mode (see step 4).

## Picking the next issue

```bash
gh issue list --state open --limit 500 --json number,title,labels,createdAt --jq '
  map(select([.labels[].name] | (index("in-progress") or index("question")) | not))
  | map(. + {rank: ([.labels[].name] as $l |
      if $l | index("ready") then 0
      elif $l | index("bug") then 1
      elif $l | index("enhancement") then 2
      elif ($l | index("tech-debt")) or ($l | index("documentation")) then 3
      else 4 end)})
  | sort_by(.rank, .createdAt) | first // empty'
```

This skips issues labelled `in-progress` or `question`, then takes the first match in order: `ready` (set by `/prioritize`), `bug`, `enhancement`, `tech-debt`/`documentation`, everything else; oldest first within a group. If the output is empty, report idle and stop.

## Steps

1. **Select** the issue, either as above or from `#N`. If it's closed or not found, report and stop. Read the body and comments: `gh issue view <N> --comments`.
2. **Validate.** If the issue is invalid, a duplicate or out of date, comment why, close it, and go back to step 1. (With `#N`, stop instead.)
3. **Claim** the issue. Pick a short slug (2–4 words) from the title, then:
   ```bash
   bash "${CLAUDE_PLUGIN_ROOT}/skills/iterate/scripts/start-issue.sh" <N> <short-slug>
   ```
   - The script adds `in-progress` (creating the label if the repo lacks it) before creating the worktree.
   - If it exits non-zero because the issue is already `in-progress`, go back to step 1 (with `#N`, stop).
   - It prints `worktree:<path>`. `cd` into that path and stay there for the rest of the run.
4. **Plan.** The run is unattended only if `$ARGUMENTS` contains `--auto`. A scheduled or task-mode agent should pass it; `AGENT_NAME` must not be used for this, because the operator sets it in *every* agent pod and those agents are interactive by design — the terminal is the whole point — so it is also set when someone is watching.
   - Interactive: enter plan mode, propose the plan, and wait for approval.
   - Unattended: post the plan as a comment (`gh issue comment <N> --body "<plan>"`) and continue.
5. **Implement** the plan inside the worktree.
6. **Test**, following `## Testing` below. Add tests as needed.
7. **Commit** with a one-line conventional message (e.g. `fix: set GatewayReady false on error`), then push. Run these as separate commands, without inline variable assignments:
   ```bash
   bash "${CLAUDE_PLUGIN_ROOT}/skills/iterate/scripts/push-branch.sh" <branch-name>
   ```
8. **Open a PR**: `gh pr create --title "<commit message>" --body "Closes #<N>"`.
9. **Watch CI**: `gh pr checks <PR> --watch`. Fix failures until all checks are green.
10. **Merge**, then delete the remote branch. Not `--delete-branch`: that also deletes the
    local branch and switches the checkout to the default, which cannot work from a
    worktree whose parent has the default branch checked out — which is where this step
    always runs. It fails with `fatal: 'main' is already checked out` *after* merging, so
    the error reads like a failed merge when the merge succeeded.
    ```bash
    gh pr merge <PR> --squash
    git push origin --delete <branch-name>
    ```
11. **Clean up.** This is the one step that leaves the worktree. Go back to the main
    checkout first, because the script deletes the worktree directory, and pass the
    worktree's absolute path. Then delete the local branch, which `git worktree remove`
    leaves behind.
    ```bash
    cd <main checkout>
    bash "${CLAUDE_PLUGIN_ROOT}/skills/iterate/scripts/remove-worktree.sh" <worktree-path>
    git branch -D <branch-name>
    ```
12. **Close the issue**:
    ```bash
    gh issue comment <N> --body "<resolution details>"
    gh issue edit <N> --remove-label "in-progress"
    gh issue close <N>
    ```
13. **Update `.claude/MEMORY.md`** if it exists and something is worth remembering for the next run (it's not a changelog). Then **stop**.

## Testing

How to test a change is per-repo, so it lives in the repo and not here.

- Follow the `## Testing` section of the repo's `CLAUDE.md`.
- If `CLAUDE.md` is missing or has no `## Testing` section, fall back to the repo's PR CI
  workflow: find the workflows in `.github/workflows/` that run on `pull_request` and run
  locally what their jobs run. Say that you fell back, in the plan and in the PR body, so
  the gap in `CLAUDE.md` gets noticed.
