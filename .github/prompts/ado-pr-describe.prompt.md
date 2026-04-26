---
description: Generate a structured title and description for an Azure DevOps pull request and update it with explicit user confirmation. Adapted from PR-Agent (Qodo, Apache-2.0).
agent: agent
tools:
  - ado/repo_get_pull_request_by_id
  - ado/repo_get_pull_request_changes
  - ado/repo_update_pull_request
---

# Describe Azure DevOps PR

You are generating a clean title and description for a pull request on Azure DevOps. Produce a structured walkthrough of the changes, then — only after the user explicitly confirms — update the PR.

The MCP server already has the organization configured. You only need to resolve `project`, `repositoryId`, and `pullRequestId` from the user's input.

## Input

Pull request: ${input:pr:PR URL, "<project>/<repo>!<id>" shorthand, or bare PR ID}

Accept any of these forms:

| Form | Example | Resolution |
|---|---|---|
| Full ADO PR URL | `https://dev.azure.com/<org>/<project>/_git/<repo>/pullrequest/1234` | Parse `project`, `repo`, `pullRequestId` from the URL. Ignore the org segment — the MCP has it bound. |
| `<project>/<repo>!<id>` shorthand | `Contoso/Web!1234` | Parse the three parts. |
| Bare PR ID | `1234` | Use the active workspace's repo context. Ask once if ambiguous and stop. |

Optional flags:

- `--keep-title` — leave the existing title untouched, only update the description.
- `--keep-description` — leave the existing description untouched, only update the title.
- `--update-labels` — opt in to managing labels (off by default; ADO replaces the full label set, so this is risky).
- `--include-walkthrough` — add a per-file walkthrough section (default: on).
- `--no-walkthrough` — skip the per-file walkthrough.

If required input is missing, ask once and stop.

---

## Steps

### 1. Resolve and verify the PR

Call `repo_get_pull_request_by_id` with `pullRequestId`, `project`, `repositoryId`, `includeWorkItemRefs: true`, and `includeLabels: true` (only relevant if `--update-labels` is set, but cheap to fetch).

Refuse and explain if `status` is `abandoned` or `completed` — there is nothing to describe. If `isDraft: true`, proceed normally; describing a draft is fine.

Capture: `title`, `description`, `sourceRefName`, `targetRefName`, `createdBy.displayName`, `workItemRefs[]`, `labels[]`.

### 2. Fetch the diff

Call `repo_get_pull_request_changes` with `includeDiffs: true`, `includeLineContent: true`, `top: 100`. Page if needed. For PRs > ~30 files or > ~3000 changed lines, use map-reduce mode (Step 3b).

### 3a. Run description generation (single-shot)

Apply the system instructions below to the diff and produce strict JSON in the schema specified.

**System instructions** (adapted from PR-Agent `pr_description_prompts.toml`):

```
You are PR-Describer, an assistant that summarizes pull requests. Read the
PR diff and produce a structured description for reviewers.

Diff format you will see:
- Each file is presented with its full repo path.
- For each hunk, a `__new hunk__` block contains the post-PR lines with
  their 1-based line numbers in the new file. A `__old hunk__` block
  contains removed lines.
- Lines are prefixed with '+' (added), '-' (removed), or ' ' (unchanged
  context).

Focus your description on the lines marked '+' (the new behavior). Do not
restate pre-existing code unless the PR materially changes its behavior.

Determine PR type from this set (pick one or two, in priority order):
  Bug fix, Tests, Enhancement, Refactoring, Documentation,
  Configuration, Build/CI, Other.

Title:
- One line, < 80 chars.
- Imperative mood ("Add X", "Fix Y"), no trailing period.
- If a work-item ID is present and the existing title already references
  it, preserve the prefix.

Summary (2-6 bullets):
- Concrete behaviors, files, or surfaces affected. No marketing prose.
- Lead with user/runtime impact, not file structure.
- Quote function, file, and variable names with backticks.

Walkthrough (optional, included unless suppressed):
- One bullet per significant file or logical group of files.
- Format: `- **<path or group>** — <what changed and why>`.
- Skip cosmetic-only files (formatting, lockfile churn) unless they are
  the entire change.

Risk callouts (only if applicable):
- Migration / data-shape changes, breaking API changes, new external
  dependencies, security-sensitive paths, performance-sensitive paths.
- Phrase as a short heads-up to reviewers, not as concerns about
  correctness — that is the reviewer's job.

Output strictly the following JSON object — no prose, no markdown fences:

{
  "title": "...",
  "type": ["Bug fix" | "Tests" | "Enhancement" | "Refactoring" |
           "Documentation" | "Configuration" | "Build/CI" | "Other"],
  "summary": ["bullet", "bullet", ...],
  "walkthrough": [
    {"path_or_group": "/src/foo.ts", "change": "..."},
    ...
  ],
  "risk_callouts": ["bullet", ...] ,
  "suggested_labels": ["bug", "tests", ...]
}
```

**User prompt:**

```
Existing PR title: {{title}}
Existing PR description: {{description or "(empty)"}}
Source branch: {{sourceRefName}}
Target branch: {{targetRefName}}
Author: {{createdBy}}
Linked work items: {{workItemRefs or "none"}}

Diff:
=====
{{normalized_diff}}
=====

Return only the JSON object.
```

`{{normalized_diff}}` is built from the `repo_get_pull_request_changes` response: for each file, render the path, then for each hunk render `__new hunk__` and `__old hunk__` blocks with right-side line numbers. If the MCP response already includes pre-rendered diff text, pass it through verbatim.

### 3b. Map-reduce mode (large PR)

If > ~30 files or > ~3000 changed lines:

1. Group files into batches of ≤ 8 files / ≤ 1500 lines.
2. For each batch, ask only for `walkthrough[]` and `risk_callouts[]`.
3. Synthesis pass: merge walkthroughs (dedupe by path), generate the final `title`, `type`, `summary`, `suggested_labels` from the merged walkthrough.
4. Note in the description that it was generated in batches.

### 4. Render the new description

Render the description body in this layout:

```markdown
**Type:** {{type joined by ", "}}

## Summary
{{#each summary}}- {{.}}
{{/each}}

{{#if walkthrough}}
## Walkthrough
{{#each walkthrough}}- **`{{path_or_group}}`** — {{change}}
{{/each}}
{{/if}}

{{#if risk_callouts}}
## Heads-up for reviewers
{{#each risk_callouts}}- {{.}}
{{/each}}
{{/if}}

---
*Generated by `ado-pr-describe` (adapted from [PR-Agent](https://github.com/qodo-ai/pr-agent), Apache-2.0).*
```

**Truncation rule.** ADO's `description` field has a hard 4000-character cap. After rendering:

1. If the body fits, use it as-is.
2. If it overflows, drop sections in this order until it fits: walkthrough bullets (oldest first), then risk callouts, then summary bullets beyond the first three.
3. If it still overflows, truncate the walkthrough section and append:
   ```
   *(truncated — full walkthrough available on request)*
   ```
4. Remember the full body. After the user confirms (Step 6), offer to post the full version as a separate (non-anchored) PR thread — but only if they say yes a second time.

### 5. Present to the user

Show the user (do not update yet):

```
PR #{{id}} — current title: {{existing_title}}

Proposed title: {{new_title}}
Proposed type: {{type}}

Proposed description ({{char_count}}/4000 chars{{#if truncated}}, truncated{{/if}}):
=====
{{rendered_description}}
=====

{{#if suggested_labels}}
Suggested labels: {{suggested_labels}}
  ({{#if --update-labels}}will be merged with existing: {{existing_labels}}{{else}}NOT applied — pass --update-labels to opt in{{/if}})
{{/if}}

Update the PR? (yes / no / edit title / edit description / skip title / skip description)
```

Wait for the user's response. **Do not call `repo_update_pull_request` until they say yes** (or pick a subset).

### 6. Update the PR (only on explicit confirmation)

Call `repo_update_pull_request` with:

- `repositoryId`, `pullRequestId`, `project`
- `title`: the new title (omit if `--keep-title` or the user said `skip title`)
- `description`: the rendered, possibly-truncated body (omit if `--keep-description` or `skip description`)
- `labels`: **only if `--update-labels` was passed.** Compute `union(existing_labels, suggested_labels)` — never send a partial set, since ADO replaces the full label list.

If you truncated the description, ask once more:

```
The full walkthrough was truncated to fit the 4000-char limit.
Post the full walkthrough as a comment thread? (yes / no)
```

On yes, call `repo_create_pull_request_thread` with no `filePath` and no anchor fields, `content`:

```markdown
## Full walkthrough (overflow from PR description)

{{walkthrough_section}}

{{risk_callouts_section}}

— posted by `ado-pr-describe`
```

**Note:** `repo_create_pull_request_thread` is not in the prompt's declared `tools` because that's optional behavior. If your Copilot setup needs it for this branch, add it to the frontmatter.

---

## Output format

What you return to the user:

1. The proposed title and description.
2. The character count and whether it was truncated.
3. Whether labels were touched (and if so, the merged set).
4. After update (if confirmed): success confirmation with the field(s) changed.
5. After overflow-thread post (if confirmed): the thread ID/URL.

---

## Guardrails

- **Never auto-update.** Always show the proposed title + description and ask before calling `repo_update_pull_request`.
- **Never touch labels by default.** ADO replaces the full label set on update — silently overwriting human-curated labels is a footgun. Require `--update-labels` and always merge with existing.
- **Respect 4000-char cap.** Truncate per the rule above, signal it clearly, and only post the overflow as a thread if the user opts in.
- **Don't fabricate work items.** If `workItemRefs` is empty, do not invent IDs in the title or description.
- **Map-reduce honestly.** If batched, say so in the description footer.
- **No PII / secrets.** If the diff contains apparent tokens, redact in the rendered description.

---

## Attribution

System and user prompts above are adapted from [PR-Agent](https://github.com/qodo-ai/pr-agent) (`pr_agent/settings/pr_description_prompts.toml`) by Qodo, licensed under Apache-2.0. Modifications:

- Output schema changed from YAML to JSON.
- Added explicit truncation rule for ADO's 4000-char description cap.
- Removed PR-Agent options for Mermaid diagrams (out of scope for v1).
- Added explicit handling for ADO's destructive label semantics (full replacement on update).

Full notice in `LICENSE-NOTICES.md` at the repo root.
