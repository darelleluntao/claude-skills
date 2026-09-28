# claude-skills

Personal collection of custom Claude Code skills, shared across all local projects via
symlinks into `~/.claude/skills/`.

## Skills

- **project-pulse** — audits every repo in a multi-repo project via `gh` (no local
  checkout needed), one parallel agent per repo, then assembles a single `lavish-axi`
  dashboard: stack & integrations, shipped features, next up, tech recommendations, and
  urgent/needs-review items per repo, plus a cross-repo release-readiness rollup. Every
  claim is cited to a file path or PR/issue number.

## Installing a skill locally

Each skill folder here gets symlinked into `~/.claude/skills/<skill-name>` so Claude Code
picks it up globally, in any project, without a per-project install step:

```sh
ln -s "$(pwd)/<skill-name>" "$HOME/.claude/skills/<skill-name>"
```
