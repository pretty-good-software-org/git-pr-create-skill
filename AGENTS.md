---
last_validated: 2026-11-09T00:09:25Z
project_type: agent-skill
---

# Agent Instructions: git-pr-create-skill

## Repository Overview

`git-pr-create-skill` publishes the `git-pr-create` Agent Skill as a Claude plugin. The skill creates or refreshes a
GitHub pull request from the final net diff and follows its checks to green. The repository is public under the MIT
license.

## Ownership

- `apm.yml` is the source of the plugin manifest: `apm pack --force` generates `.claude-plugin/plugin.json`, and
  `mise run test:plugin-manifest` fails when the committed manifest drifts from that output.
- `skills/git-pr-create/SKILL.md` is the authoritative skill. The plugin name, skill directory, and frontmatter name are
  `git-pr-create`; the repository name differs on purpose and is not part of the plugin identity.
- `.mise.toml` and `mise.lock` pin the tools CI uses; the development profile adds the Git hook tools (Lefthook and
  gitleaks). `tests/test_plugin_contract.py` checks that the manifest and the skill describe one identity and that the
  skill's links resolve.
- `pyproject.toml` and `uv.lock` own Python dependencies; `UV_NO_MANAGED_PYTHON=1` forces uv onto Mise's Python, and
  `UV_LOCKED=1` makes every `uv run` fail on a stale `uv.lock` instead of re-locking it.
- Changie owns `CHANGELOG.md`: add entries under `.changes/unreleased/`, batch them into `.changes/<version>.md`, and
  merge.
- Branch protection and repository settings are managed outside this repository; do not change them in the GitHub UI.

## Repository Structure

```text
.
├── .actionlint.yml
├── .changes
│   ├── 1.0.0.md
│   ├── 1.0.1.md
│   ├── 1.1.0.md
│   ├── 1.2.0.md
│   ├── 1.2.1.md
│   ├── header.tpl.md
│   └── unreleased
│       └── .gitkeep
├── .changie.yaml
├── .claude-plugin
│   └── plugin.json
├── .github
│   ├── PULL_REQUEST_TEMPLATE.md
│   ├── dependabot.yml
│   └── workflows
│       ├── lint.yml
│       └── test.yml
├── .gitignore
├── .mise.ci.toml
├── .mise.development.toml
├── .mise.toml
├── .miserc.toml
├── .rumdl.toml
├── .yamllint.yml
├── AGENTS.md
├── CHANGELOG.md
├── CLAUDE.md
├── LICENSE
├── README.md
├── apm.yml
├── cog.toml
├── lefthook
│   ├── commit-msg.yml
│   ├── files.yml
│   ├── lint.yml
│   ├── python.yml
│   └── secrets.yml
├── lefthook.yml
├── mise-tasks
│   ├── check
│   │   └── markdown-format
│   ├── format
│   │   └── markdown
│   ├── lint
│   │   ├── actionlint
│   │   ├── audit
│   │   ├── default
│   │   ├── python
│   │   ├── rumdl
│   │   └── yamllint
│   ├── setup
│   │   └── default
│   └── test
│       ├── default
│       ├── plugin-manifest
│       └── python-runtime
├── mise.development.lock
├── mise.lock
├── pyproject.toml
├── skills
│   └── git-pr-create
│       └── SKILL.md
├── tests
│   └── test_plugin_contract.py
└── uv.lock
```

## Commands

```bash
mise run setup:default
mise run lint:default
mise run test:default
```

Use the fastest loop that exercises the change: a focused pytest test first, then the complete lint and test tasks.

## Development Guidelines

This repository is public.

- Keep product names, customer data, test accounts, hostnames, session links, and local paths out of files and commit
  messages. Examples use neutral apps and text.
- CI runs on GitHub-hosted runners, so pull requests from forks never run on self-hosted machines.

## Testing

The plugin contract test keeps the manifest, the skill's frontmatter, and its links consistent. The skill itself is
instructions for an agent; verify a behavior change by running an agent with the installed skill and record what you
ran in the pull request.

## Git Workflow

Check status first, work on feature branches, use Conventional Commits, and open a pull request; run
`mise run lint:default` and `mise run test:default` before opening it. To release, bump `version` in `apm.yml`, run
`apm pack --force`, batch the changie entries for that version, and merge to `main` through a pull request. Tag the
merged commit `v<version>` with an annotated tag and publish a GitHub release; consumers pin the tagged commit.
