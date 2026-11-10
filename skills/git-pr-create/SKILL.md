---
name: git-pr-create
description: "Create or refresh GitHub PRs from the live net diff, attach bug or before/after videos when provided, and drive checks green when possible. Use when: opening PRs, updating stale PRs after new commits, or adding a recording to a fix PR. Triggers: /git-pr-create, create PR, open pull request, update PR, attach video to PR."
---

# PR Create

Create GitHub pull requests with `gh pr create`, or refresh an existing branch PR with `gh pr edit`.

## Format

**Title:** Commit subject or change summary

**Body:**

```markdown
## Summary
- Change description (bullets OK)

## Video
(Only with a recording; see "Video and Screenshots")

## Test plan
- [ ] Verification steps
```

## Workflow

1. Verify not on main/master
2. Find an existing open PR for the branch; reuse it, or create one only when creation was requested
3. Use the existing PR's base branch, or detect the default base for a new PR
4. Inspect the live PR diff or the final net diff against the base branch
5. Generate the title from the final change and its commits
6. Build or refresh the body with a summary and test plan
7. When a recording of the bug or the fix is available, add it as described in "Video and Screenshots"
8. Create the PR or edit stale metadata on the existing PR
9. Verify the live PR, then drive its checks green or report a concrete blocker

## Commands

```bash
# Check current branch
git branch --show-current

# Look for an existing open PR before creating another one
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
BRANCH=$(git branch --show-current)
gh pr list -R "$REPO" --head "$BRANCH" --state open --json number,headRefName,headRefOid,baseRefName

# Create PR only when no existing PR was found (draft by default)
gh pr create --draft --title "$TITLE" --body "$(cat <<'EOF'
## Summary
- Change 1
- Change 2

## Test plan
- [ ] Test item
EOF
)"
```

## Title Generation

Use a single commit subject only when it accurately summarizes the final net diff. Otherwise, summarize the final
change. Commit history is supporting context, not the title authority:

```bash
git log "origin/$BASE"..HEAD --format='%s'
```

## Body Style

Bullets allowed in PR body (unlike commit messages):

- Summary section lists changes
- Test plan uses checkboxes

Keep concise. One line per logical change.

Before writing the title or body, inspect the final net diff:

```bash
BASE=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
git fetch origin "$BASE"
git diff --stat "origin/$BASE"...HEAD
git diff --name-status "origin/$BASE"...HEAD
git diff "origin/$BASE"...HEAD
```

Every summary bullet must map to a file or behavior present in that diff. Describe the final state, not the work log:

- Omit discarded approaches, removed flags or helpers, and superseded intermediate designs.
- Do not present a pre-existing tool, file, hook, or behavior as introduced by the PR unless the net diff changes it.
- Avoid brittle process details such as the number of commits.
- Validation may name pre-existing tools and temporary probes when they were actually used, but phrase them as evidence,
  not as artifacts shipped by the PR.

After creating or refreshing the PR, compare its live title and body with `gh pr diff` and correct stale or unsupported
claims. Repeat this comparison after any fix changes the PR's net diff.

For a compact summary of the live PR's changed files, additions, and deletions, use
[`gh pr view`](https://cli.github.com/manual/gh_pr_view):

```bash
gh pr view "$PR" -R "$REPO" --json changedFiles,additions,deletions,files
```

Set `REPO` to the target `OWNER/REPO`. Use `gh pr diff "$PR" -R "$REPO"` to review patch content;
statistics do not replace that review. Unlike `git diff`, `gh pr diff` does not support `--stat` or `--check`.
Check `gh pr diff --help` before using unfamiliar flags rather than assuming Git flags apply.

## Refreshing an Existing PR

When an open PR already targets this branch, inspect its live `headRefOid`, `headRepositoryOwner`, `baseRefName`, title,
and body. If several PRs match or the head owner/branch differs from the intended source, stop and select the exact PR
rather than updating an arbitrary one. For an update-only request with no matching PR, report that it is missing; do
not create a replacement.

Use the PR's own base branch and `gh pr diff "$PR" -R "$REPO"` as the source for the *published* net change. Do not
include uncommitted or unpushed local changes in its description. If the local branch differs from the live head, first
identify whether the change needs an authorized push, then re-read the PR at its new head. Compare every title and
summary claim with the current diff; update only stale claims and checkboxes. Preserve reviewer-authored context,
manual notes, and unrelated body sections. If their ownership or intended meaning is unclear, ask before replacing
them. Do not edit accurate metadata simply because a new commit exists.

```bash
gh pr view "$PR" -R "$REPO" --json headRefOid,headRepositoryOwner,headRefName,baseRefName,title,body
gh pr diff "$PR" -R "$REPO"
# Only after reviewing a stale title/body and preserving unrelated text:
gh pr edit "$PR" -R "$REPO" --title "$TITLE" --body "$BODY"
```

Re-read the live PR after editing and verify the metadata against the latest diff. Then follow the same CI feedback
workflow as for a new PR. Never reset an existing draft/review state merely to refresh its description.

## Video and Screenshots

A fix for a user-visible bug is reviewed faster when the reviewer can watch it: one video of the bug, or the bug
beside the fix. Attach a recording only when the user provided or approved it for this PR. Uploading publishes it to
everyone who can read the repository, and a screen recording can show personal or test data. This skill attaches
existing files; it does not record them. For mobile apps with Maestro flows, `maestro-bug-video` produces both kinds.

Before attaching, check the file:

- It shows what the PR claims: extract and look at its first and last frame
  (`ffmpeg -ss 1 -i video.mp4 -frames:v 1 first.png`, `ffmpeg -sseof -1 -i video.mp4 -frames:v 1 last.png`).
- It is MP4, MOV, or WebM, and at most 10 MB on GitHub Free or 100 MB on paid plans. Shrink a larger file before
  uploading, for example `ffmpeg -i in.mp4 -vf scale=1280:-2 -c:v libx264 -crf 28 -an out.mp4`.
- `gh` is 2.99.0 or newer: `gh pr create --help` must list `--attach`.

Under `## Video` in the body, write one caption line that states what the viewer sees, such as "Before: the keyboard
stays over the note. After: it closes on scroll.", then reference the file with Markdown image syntax using the same
relative path you pass to `--attach`, such as `./before-after.mp4`. `gh` uploads the file and rewrites that reference
to the attachment URL, the only form GitHub renders as an inline video player; an attachment the body does not
reference is appended at the end instead. A video plays without alt text, so the caption carries its description; the
`path#alt` form only labels images.

```bash
gh pr create --draft --title "$TITLE" --body "$BODY" --attach ./before-after.mp4
gh pr edit "$PR" -R "$REPO" --body "$BODY" --attach ./before-after.mp4
```

If an upload fails, `gh pr create` still creates the PR with the attachments that succeeded, exits nonzero, and prints
the new PR's URL. Read that URL and continue with `gh pr edit` on that PR; never rerun `gh pr create`, which would open
a duplicate.

In the test plan, say what the video shows and which build or commit each side ran. When refreshing a PR, keep an
existing attachment URL instead of uploading the same recording again; attach a new file only when the recording
changed.

If `gh` lacks `--attach` or the file stays over the limit, create or refresh the PR without the video, say so, and
give the user the file path to drag into the description on github.com. Do not call GitHub's private upload endpoints.

## Draft PRs (Default)

PRs are created as **drafts by default**. This prevents premature CI reviews (e.g., CodeRabbit) on work-in-progress
branches.

To create a **ready** (non-draft) PR when the user explicitly requests it:

```bash
gh pr create --title "$TITLE" --body "$BODY"
```

To mark an existing draft PR as ready for review:

```bash
gh pr ready "$PR" -R "$REPO"
```

## Base Branch

Detect automatically:

```bash
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'
```

Override with `--base`:

```bash
gh pr create --draft --base develop --title "$TITLE" --body "$BODY"
```

## After Creation or Refresh: CI Feedback

Do not stop at the PR URL. Read the live PR and its checks for the current head commit, then follow the checks to a
verified green state when the failures are within scope. Use an explicit `OWNER/REPO` on GitHub commands so a fork or
renamed remote cannot silently change the target.

```bash
gh pr view "$PR" -R "$REPO" --json headRefOid,state,statusCheckRollup,url
gh pr checks "$PR" -R "$REPO"
# For a failed Actions job, take its run ID from the reported details URL:
gh run view "$RUN_ID" -R "$REPO" --log-failed
```

1. Identify the expected checks from the repository's workflow and branch rules. A missing, skipped, or inaccessible
   check is not evidence of success. Distinguish queued/in-progress work from completed success and completed failure;
   treat unrecognized or empty results as unknown rather than green. `gh pr checks` can exit nonzero for pending work,
   so read its output instead of inferring failure from the exit code alone.
2. Wait for pending checks with a bounded budget of 15 minutes, checking the PR's **current head SHA** each time. Do
   not watch indefinitely. If a new push changes the SHA, discard earlier check results and inspect the new run.
3. For a failed check, read the actual job log and reproduce the failing layer with the narrowest local loop. Fix the
   root cause in its owning repository when authorized; run the focused test and required final checks, commit, push,
   and inspect the new head. Allow at most two fix-and-push cycles in this invocation. Never skip a required check,
   weaken a gate, grant broader credentials, or use a CI-bypass flag merely to make the PR green.
4. Stop and report the specific failure, owner, and needed authorization when credentials, external services,
   permissions, or unrelated repositories block a durable fix. Do not claim green based on local tests alone. When
   all expected checks on the current head have completed successfully, report their names and the verified head SHA.
   PR creation does not authorize merge or changing a draft to ready.

## Safety

- Verify branch is not main/master before creating
- Check for existing PR to avoid duplicates
- Never create PR from main to main
- Never merge or bypass required checks as part of this skill
