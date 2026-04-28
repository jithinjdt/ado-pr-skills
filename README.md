# ado-pr-skills

GitHub Copilot prompt files for working on Azure DevOps pull requests from VS Code. Each skill calls the [Azure DevOps MCP server](https://github.com/microsoft/azure-devops-mcp) under the hood.

Adapted from [PR-Agent](https://github.com/qodo-ai/pr-agent) (Qodo, Apache-2.0).

## Skills

| Skill | What it does |
|---|---|
| [`/ado-pr-review`](.github/prompts/ado-pr-review.prompt.md) | Reviews a PR. Flags bugs, security, perf, and missing-test concerns; posts inline + summary threads and (optionally) votes — all behind explicit confirmation. |
| [`/ado-pr-describe`](.github/prompts/ado-pr-describe.prompt.md) | Generates a structured PR title + description (type, summary, walkthrough, risk callouts) and updates the PR after confirmation. |
| [`/ado-pr-improve`](.github/prompts/ado-pr-improve.prompt.md) | Proposes concrete code-improvement patches a reviewer/author can apply, posted as inline threads after confirmation. Complements `review`. |
| [`/ado-pr-ask`](.github/prompts/ado-pr-ask.prompt.md) | Answers a freeform question about a PR, grounded strictly in the diff. Local-only by default; can post or reply to a thread on request. |
| [`/ado-pr-test-gaps`](.github/prompts/ado-pr-test-gaps.prompt.md) | Identifies missing test coverage and lists concrete scenarios that should exist. Optional summary thread after confirmation. |

## Prerequisites

1. **VS Code with GitHub Copilot Chat** (agent mode).
2. **Azure DevOps MCP server registered** (local or remote) with your organization bound. Toolset header:
   - `X-MCP-Toolsets: repos` — required by all skills.
   - Add `wit` if you want `ado-pr-test-gaps` to fetch linked work-item titles/descriptions. Optional.

Authentication is handled by the MCP server itself — see the [azure-devops-mcp](https://github.com/microsoft/azure-devops-mcp) docs for the auth methods it supports (e.g. `az login` / Entra, or a PAT). The skills don't take credentials directly.

The skills assume the MCP server already has `organization` bound — you only supply `project`, `repository`, and PR ID.

## Install

Clone or copy this repo into a workspace where Copilot picks up `.github/prompts/`. The five `.prompt.md` files are self-contained — Copilot loads each one when the matching slash command is invoked.

## Usage

Invoke any skill from Copilot Chat with a PR reference:

```
/ado-pr-review https://dev.azure.com/<org>/<project>/_git/<repo>/pullrequest/1234
/ado-pr-describe Contoso/Web!1234
/ado-pr-ask 1234   What does this change about auth?
```

### Accepted PR-input forms (all skills)

| Form | Example |
|---|---|
| Full ADO PR URL | `https://dev.azure.com/<org>/<project>/_git/<repo>/pullrequest/1234` |
| `<project>/<repo>!<id>` shorthand | `Contoso/Web!1234` |
| Bare PR ID | `1234` (uses the active workspace's repo context; asks if ambiguous) |

### Common flags

These appear on most skills (see each prompt file for the full list):

- `--iteration <id>` — target a specific iteration instead of the latest.
- `--compare-to <id>` — diff against a specific older iteration.
- `--focus <areas>` — comma-separated subset of focus areas (skills define their own sets).
- `--max-findings <n>` / `--max-suggestions <n>` / `--max-scenarios <n>` — cap output.

Skill-specific flags worth knowing:

- `review`: `--include-nitpicks` (off by default).
- `describe`: `--keep-title`, `--keep-description`, `--update-labels` (off by default — ADO replaces the full label set on update), `--no-walkthrough`.
- `improve`: `--no-code-blocks`.
- `ask`: `--post`, `--reply-to <threadId>`, `--anchor <path>:<line>`.
- `test-gaps`: `--test-dirs <paths>`, `--include-existing-tests`, `--no-work-items`.

## Safety model

The skills are designed to be safe to invoke on real PRs:

- **Nothing posts to ADO without explicit confirmation.** Each skill computes its output, shows it to you, and waits for `yes` before calling any write tool.
- **Voting requires a separate, second confirmation.** `review` will not cast a vote in the same turn as posting comments.
- **Labels are not touched by default** in `describe` — ADO replaces the full label set on update, which silently drops human-curated labels. Opt in with `--update-labels` (the skill then merges with the existing set).
- **`ask` is local-only by default** — it answers in chat. Pass `--post`, `--reply-to`, or `--anchor` to send anything to ADO.
- **Abandoned/completed PRs are refused.** Drafts are allowed (with a confirmation prompt for `improve`).

## Further reading

- [`MCP_TOOLS.md`](MCP_TOOLS.md) — schemas, gotchas, and per-skill tool requirements for the underlying Azure DevOps MCP tools. Maintainer reference.
- Each `.prompt.md` file contains the full system prompt, input parsing, steps, output format, guardrails, and attribution for that skill.
