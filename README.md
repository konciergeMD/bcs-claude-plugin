# messaging-team-claude

Claude Code plugin for the Accolade messaging team.

## What's included

### Hook: ruff auto-fix on Python edits
Automatically runs `ruff check --fix` and `ruff format` on any `.py` file Claude writes or edits. Requires `uvx` (comes with `uv`).

### Skill: `/commit-and-pr`
Stages changes, generates a conventional commit message, and opens a PR via `gh`. Works in any project.

### Skill: `/release`
Automates the `messaging-model` release process:
- Bumps `ts/package.json` and `java/pom.xml` versions
- Commits the version bump
- Creates the GitHub release + tag via `gh release create`
- Prints Jenkins pipeline links for each language

## Prerequisites

- [`uv`](https://docs.astral.sh/uv/) installed (`brew install uv`)
- [`gh`](https://cli.github.com/) CLI authenticated
- `jq` installed (`brew install jq`)

## Install

### Option A: Local install (for development)
```bash
claude plugin install ./messaging-team-claude-plugin
```

### Option B: From Artifactory npm
```bash
# Coming soon — once published to internal registry
npm install -g @accolade/messaging-team-claude
claude plugin install @accolade/messaging-team-claude
```

## Usage

In any project:
```
/commit-and-pr
```

In `messaging-model`:
```
/release 1.30.0
```

## Publishing (maintainers)

```bash
cd messaging-team-claude-plugin
npm publish --registry https://<artifactory-url>/api/npm/<repo>/
```
