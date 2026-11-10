# git-pr-create-skill

An [Agent Skill](https://agentskills.io/specification), `git-pr-create`, that creates a GitHub pull request, or
refreshes the one already open for the branch, from the branch's final net diff. It then follows the pull request's
checks to green when the failures are within reach.

- **Title and body from the net diff:** every summary line maps to a change in the diff, and the body describes the
  final state, not the work log. Refreshing an existing pull request updates only stale claims and keeps reviewers'
  notes.
- **Drafts by default:** pull requests open as drafts unless the user asks for a ready one.
- **Videos:** a bug or before/after recording the user provided or approved is checked and attached with
  `gh --attach`, under a captioned `## Video` section.
- **Checks:** after creating or refreshing, it waits a bounded time for the checks on the current head commit, reads
  failed job logs, and allows at most two fix-and-push cycles. It never merges, bypasses a required check, or marks a
  draft ready on its own.

## Requirements

- `git`, and the [GitHub CLI](https://cli.github.com/) (`gh`), signed in with access to the repository.
- To attach a video: `gh` 2.99.0 or newer, whose `gh pr create --help` lists `--attach`. `ffmpeg` checks the
  recording's frames and size before upload.

## Installation

With [APM](https://github.com/microsoft/apm), pinned to a release:

```bash
apm install pretty-good-software-org/git-pr-create-skill#v1.2.2
```

Without APM, copy the skill directory into your agent's skills directory, for example for Claude Code:

```bash
git clone --depth 1 --branch v1.2.2 https://github.com/pretty-good-software-org/git-pr-create-skill.git
cp -R git-pr-create-skill/skills/git-pr-create ~/.claude/skills/
```

## Usage

On a feature branch, ask the agent to "create a PR" or "update the PR". The skill's
[`SKILL.md`](skills/git-pr-create/SKILL.md) holds the workflow, the body format, and the check-following rules.

## Development

Development uses [mise](https://mise.jdx.dev/), which installs the pinned tools from the lock files:

```bash
mise run setup:default
mise run lint:default
mise run test:default
```

`setup:default` also installs the Git hooks. Changes are recorded with [Changie](https://changie.dev/) in `.changes/`
and merged into [`CHANGELOG.md`](CHANGELOG.md).

## License

[MIT](LICENSE)
