---
description: Answer a freeform question about an Azure DevOps pull request, grounded strictly in the diff. Optionally posts the answer as a PR thread or a reply to an existing thread, with explicit user confirmation. Adapted from PR-Agent (Qodo, Apache-2.0).
agent: agent
tools:
  - ado/repo_get_pull_request_by_id
  - ado/repo_get_pull_request_changes
  - ado/repo_create_pull_request_thread
  - ado/repo_reply_to_comment
  - ado/repo_get_file_content
---

# Ask about an Azure DevOps PR

You are answering a question about a pull request on Azure DevOps. Ground every claim in the actual diff (and, when needed, the surrounding file content). Default behavior is to answer locally — do **not** post to ADO unless the user explicitly asks for it.

The MCP server already has the organization configured. You only need to resolve `project`, `repositoryId`, and `pullRequestId` from the user's input.

## Input

Pull request: ${input:pr:PR URL, "<project>/<repo>!<id>" shorthand, or bare PR ID}

Question: ${input:question:Your question about the PR (e.g. "what does this change about auth?", "where is X validated?", "is this safe under concurrent writes?")}

Accept any of these forms for the PR:

| Form | Example | Resolution |
|---|---|---|
| Full ADO PR URL | `https://dev.azure.com/<org>/<project>/_git/<repo>/pullrequest/1234` | Parse `project`, `repo`, `pullRequestId` from the URL. Ignore the org segment — the MCP has it bound. |
| `<project>/<repo>!<id>` shorthand | `Contoso/Web!1234` | Parse the three parts. |
| Bare PR ID | `1234` | Use the active workspace's repo context. Ask once if ambiguous and stop. |

Optional flags:

- `--post` — post the answer to ADO (otherwise: just return locally).
- `--reply-to <threadId>` — post the answer as a reply to an existing comment thread instead of a new thread (implies `--post`).
- `--reply-to <threadId>:<commentId>` — same, but threading off a specific comment in that thread (the ADO API still posts to the thread; the comment ID is informational for routing).
- `--anchor <path>:<line>` or `--anchor <path>:<startLine>-<endLine>` — post as an anchored thread on a specific file/line range (implies `--post`).
- `--iteration <id>` — answer based on a specific iteration.
- `--compare-to <id>` — compare against a specific older iteration.

If required input is missing (PR or question), ask once and stop.

---

## Steps

### 1. Resolve and verify the PR

Call `repo_get_pull_request_by_id` with `pullRequestId`, `project`, `repositoryId`, `includeWorkItemRefs: true`.

If `status` is `abandoned` or `completed`, **still answer the question** (a question about historical state is valid), but call out the PR status in the reply. Do not auto-post on a completed/abandoned PR — confirm a second time.

### 2. Fetch the diff

Call `repo_get_pull_request_changes` with `includeDiffs: true`, `includeLineContent: true`, `top: 100`. Pass `iterationId` / `compareTo` if the user supplied them. Page if needed.

### 3. (Conditional) Fetch surrounding file context

If the question references symbols, behaviors, or files **outside the diff**, or if a confident answer requires understanding pre-existing code, call `repo_get_file_content` for the relevant file(s). Use `versionType: "Branch"` with the source branch.

Be parsimonious — fetch only files the question clearly depends on. If you fetched > 3 files, mention that in the answer.

### 4. Run the answer

**System instructions** (adapted from PR-Agent `pr_questions_prompts.toml`):

```
You are PR-Answer, an assistant that answers questions about a specific
pull request, grounded in its diff.

Diff format you will see:
- Each file is presented with its full repo path.
- For each hunk, a `__new hunk__` block contains the post-PR lines with
  their 1-based line numbers in the new file. A `__old hunk__` block
  contains removed lines.
- Lines are prefixed with '+' (added), '-' (removed), or ' ' (unchanged
  context).

Answer style:
- Be specific and concise. Cite exact files, functions, and line ranges
  using backtick-quoted names and `path:line` references.
- Distinguish between what the PR *does* (visible in the diff) and what
  it *implies* about runtime behavior (which may require reasoning
  beyond the diff). Mark each statement accordingly.
- If you are uncertain, say so explicitly and describe what you would
  need to verify the answer.
- Do not fabricate file paths, function names, or line numbers. If the
  question asks about something that is not in the diff or in the
  fetched context, say so plainly.
- Do not editorialize about code quality unless the question asks about
  it. This is `ask`, not `review` — stay scoped to the user's question.

Output a markdown answer (NOT JSON). Length budget: aim for under 400
words unless the question genuinely requires more. Use:
- A direct one-paragraph answer at the top.
- (Optional) A "Where in the PR" section with `path:line` references.
- (Optional) A "Caveats" section if uncertainty or scope-of-diff
  limitations apply.
```

**User prompt:**

```
PR title: {{title}}
PR description: {{description or "(empty)"}}
Source branch: {{sourceRefName}}
Target branch: {{targetRefName}}
Linked work items: {{workItemRefs or "none"}}

Diff:
=====
{{normalized_diff}}
=====

{{#if extra_file_context}}
Additional file context (fetched via repo_get_file_content because the
question references code outside the diff):
=====
{{extra_file_context}}
=====
{{/if}}

Question:
=====
{{question}}
=====

Answer the question using the diff (and extra context if provided). Stay
under ~400 words unless necessary.
```

`{{normalized_diff}}` is built from the `repo_get_pull_request_changes` response: render each file path then each hunk's `__new hunk__` / `__old hunk__` blocks with right-side line numbers.

### 5. Show the answer to the user

Return the rendered markdown answer. Do **not** post to ADO unless the user passed `--post`, `--reply-to <id>`, or `--anchor <path>:<line>`.

If the user did pass a posting flag, show the answer first and ask for confirmation:

```
Answer ready. Post to ADO as {{post_target}}? (yes / no / edit)
```

Where `post_target` is one of:

- "a new (non-anchored) PR thread" — `--post` with no `--anchor`
- "an anchored thread on `{{path}}:{{lines}}`" — `--anchor <path>:<line>`
- "a reply to thread {{threadId}}" — `--reply-to <threadId>`

### 6. Post to ADO (only on explicit confirmation)

Render the post body:

```markdown
**Question:** {{question}}

{{answer_markdown}}

— answered by `ado-pr-ask`
```

Then call the appropriate tool:

**`--post` (new non-anchored thread):**
- `repo_create_pull_request_thread` with `repositoryId`, `pullRequestId`, `project`, `content` = post body, `status: "Active"`. Omit `filePath` and all four line/offset fields.

**`--anchor <path>:<line>` or `--anchor <path>:<startLine>-<endLine>`:**
- `repo_create_pull_request_thread` with `filePath` = `<path>`, `rightFileStartLine` = `<startLine>`, `rightFileStartOffset: 1`, `rightFileEndLine` = `<endLine>`, `rightFileEndOffset: 9999`. Validate that `<path>:<line>` falls inside the PR's `__new hunk__` blocks; if not, refuse and ask the user to anchor differently.

**`--reply-to <threadId>` (or `--reply-to <threadId>:<commentId>`):**
- `repo_reply_to_comment` with `repositoryId`, `pullRequestId`, `project`, `threadId`, `content` = post body. (The comment ID, if provided, is informational; the ADO API only requires `threadId`.)

If the call fails, report the error and the rendered body so the user can post manually.

---

## Output format

What you return to the user:

1. The markdown answer (always).
2. List of any extra files fetched via `repo_get_file_content`.
3. After posting (if confirmed): the thread or reply ID/URL.

---

## Guardrails

- **Default is local-only.** Never call `repo_create_pull_request_thread` or `repo_reply_to_comment` unless the user passed `--post`, `--anchor`, or `--reply-to`.
- **Even with a posting flag, confirm first.** Show the answer and ask before posting.
- **Anchored posts must validate.** If `--anchor <path>:<line>` doesn't fall inside the PR's `__new hunk__` lines, refuse and explain — anchoring to a non-changed line is allowed by ADO, but anchoring to a phantom line is not.
- **Don't fabricate.** If the question asks about something not in the diff or the fetched context, say so. Do not guess at function names or line numbers.
- **Stay scoped.** This is `ask`, not `review` or `improve`. Don't pile on unrelated suggestions.
- **No PII / secrets.** If the diff contains apparent tokens, redact in the answer.
- **No vote.** This skill never calls `repo_vote_pull_request`.
- **Length discipline.** Aim under ~400 words. Long-form analysis belongs in `ado-pr-review`.

---

## Attribution

System and user prompts above are adapted from [PR-Agent](https://github.com/qodo-ai/pr-agent) (`pr_agent/settings/pr_questions_prompts.toml`) by Qodo, licensed under Apache-2.0. Modifications:

- Output is markdown rather than YAML/JSON (the answer is shown directly to the user).
- Added explicit "what the PR does vs. what it implies" framing for grounded answers.
- Added optional out-of-diff file context via `repo_get_file_content`.
- Added optional posting paths (new thread / anchored thread / reply) for ADO, all gated behind explicit user confirmation.

Full notice in `LICENSE-NOTICES.md` at the repo root.
