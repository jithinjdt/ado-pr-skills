# MCP_TOOLS.md — Azure DevOps MCP tool reference

Quick reference for the ADO MCP tool names and parameter schemas the prompt files in `.github/prompts/` call. **Reference only** — Copilot does not load this file at runtime; each prompt file is self-contained.

Tool names follow the [open-source server source](https://github.com/microsoft/azure-devops-mcp/blob/main/src/tools/repositories.ts). The published Microsoft docs table is occasionally out of date.

Toolset header (set on the MCP server registration): `X-MCP-Toolsets: repos` — add `wit` only if `ado-pr-test-gaps` should fetch linked work-item titles.

---

## Per-skill tool requirements

All five skills now sample target-repo context (tree skeleton + README + language manifest) by default, gated behind `--repo-context off|skeleton|deep` (default `skeleton`). That promotes `repo_list_directory` and `repo_get_file_content` to **required** on every skill — `--repo-context off` is the only mode that bypasses them.

| Skill | Required tools | Optional |
|---|---|---|
| `ado-pr-review` | `repo_get_pull_request_by_id`, `repo_get_pull_request_changes`, `repo_list_pull_request_threads`, `repo_create_pull_request_thread`, `repo_vote_pull_request`, `repo_list_directory`, `repo_get_file_content` | — |
| `ado-pr-describe` | `repo_get_pull_request_by_id`, `repo_get_pull_request_changes`, `repo_update_pull_request`, `repo_list_directory`, `repo_get_file_content` | — |
| `ado-pr-improve` | `repo_get_pull_request_by_id`, `repo_get_pull_request_changes`, `repo_create_pull_request_thread`, `repo_list_pull_request_threads`, `repo_list_directory`, `repo_get_file_content` | — |
| `ado-pr-ask` | `repo_get_pull_request_by_id`, `repo_get_pull_request_changes`, `repo_list_directory`, `repo_get_file_content` | `repo_create_pull_request_thread`, `repo_reply_to_comment` |
| `ado-pr-test-gaps` | `repo_get_pull_request_by_id`, `repo_get_pull_request_changes`, `repo_list_directory`, `repo_get_file_content` | `repo_create_pull_request_thread`, `wit_get_work_items_batch_by_ids` |

---

## Gotchas (encoded in the prompt files — listed here for reference)

1. **`repo_get_pull_request_changes` is the diff tool**, even though the public Microsoft docs table omits it. Returns line-by-line diffs with `includeDiffs: true, includeLineContent: true`.
2. **No "list iterations" tool exists.** `iterationId` and `compareTo` are parameters on `repo_get_pull_request_changes`; omit them for the latest iteration.
3. **Inline comments use flat anchor fields**, not the REST API's `threadContext` shape: `filePath`, `rightFileStartLine`, `rightFileStartOffset`, `rightFileEndLine`, `rightFileEndOffset`. Offsets are 1-based; the server attaches the latest iteration. **No left-side fields exist** — anchor deletion-related comments to surrounding right-side context.
4. **`repo_update_pull_request.labels` REPLACES the existing set.** Fetch existing labels via `repo_get_pull_request_by_id` with `includeLabels: true` and union before sending — otherwise human-added labels disappear.
5. **No native "approve" verb.** Use `repo_vote_pull_request` with the vote enum: `Approved`(10), `Suggestions`(5), `None`(0), `Waiting`(-5), `Rejected`(-10). The review skill always asks a separate second confirmation before voting.
6. **`repo_update_pull_request.description` has a 4000-char hard cap.** The describe skill truncates with a `(truncated)` marker and offers to post the full version as a thread.

---

## Schemas — `repos` toolset

### Read tools

#### `repo_get_pull_request_by_id`

| Param | Type | Required | Notes |
|---|---|---|---|
| `repositoryId` | string | yes | GUID or name; if name, `project` is required |
| `pullRequestId` | number | yes | |
| `project` | string | no | required when `repositoryId` is a name |
| `includeWorkItemRefs` | boolean | no | default `false` |
| `includeLabels` | boolean | no | default `false` — set in `describe` |
| `includeChangedFiles` | boolean | no | default `false` |

#### `repo_get_pull_request_changes`

| Param | Type | Required | Notes |
|---|---|---|---|
| `repositoryId` | string | yes | |
| `pullRequestId` | number | yes | |
| `project` | string | no | |
| `iterationId` | number | no | omit for latest |
| `compareTo` | number | no | iteration to compare against |
| `top` | number | no | default 100 — max files per page |
| `skip` | number | no | pagination |
| `includeDiffs` | boolean | no | default `true` |
| `includeLineContent` | boolean | no | default `true` |

For PRs with > 100 changed files, page via `top` / `skip`.

#### `repo_get_file_content`

| Param | Type | Required | Notes |
|---|---|---|---|
| `repositoryId` | string | yes | |
| `path` | string | yes | full repo path, e.g. `/src/main.ts` |
| `project` | string | no | |
| `version` | string | no | branch/tag/commit |
| `versionType` | enum | no | default `Commit` — pass `Branch` or `Tag` as needed |

#### `repo_list_directory`

| Param | Type | Required | Notes |
|---|---|---|---|
| `repositoryId` | string | yes | |
| `path` | string | no | default `/` |
| `project` | string | no | |
| `version` | string | no | |
| `versionType` | enum | no | default `Branch` |
| `recursive` | boolean | no | default `false` |
| `recursionDepth` | number | no | 1–10, default 1 |

#### `repo_list_pull_request_threads`

| Param | Type | Required | Notes |
|---|---|---|---|
| `repositoryId` | string | yes | |
| `pullRequestId` | number | yes | |
| `project` | string | no | |
| `iteration` | number | no | latest if omitted |
| `baseIteration` | number | no | |
| `top` | number | no | default 100 |
| `skip` | number | no | default 0 |
| `fullResponse` | boolean | no | default `false` (trimmed) |
| `status` | enum | no | filter by thread status |
| `authorEmail` | string | no | filter |
| `authorDisplayName` | string | no | partial match, case-insensitive |

#### `repo_get_repo_by_name_or_id`

Resolves a repo name → GUID. Used during input parsing when only a name is known.

| Param | Type | Required |
|---|---|---|
| `project` | string | yes |
| `repositoryNameOrId` | string | yes |

### Write tools

#### `repo_create_pull_request_thread` *(inline-comment tool)*

| Param | Type | Required | Notes |
|---|---|---|---|
| `repositoryId` | string | yes | |
| `pullRequestId` | number | yes | |
| `content` | string | yes | markdown supported |
| `project` | string | no | |
| `filePath` | string | no | required for inline (anchored) comments |
| `status` | enum | no | default `Active` |
| `rightFileStartLine` | number | no | 1-based; required for line-anchored |
| `rightFileStartOffset` | number | no | 1-based char offset; required if start line set |
| `rightFileEndLine` | number | no | required if start line set |
| `rightFileEndOffset` | number | no | required if end line set |

**Anchor a single full line**: `rightFileStartLine = N`, `rightFileStartOffset = 1`, `rightFileEndLine = N`, `rightFileEndOffset = 9999` (ADO clamps to end-of-line).
**Anchor a range**: start of first line, end of last line.
**Non-anchored summary comment**: omit `filePath` and the four line/offset fields.

#### `repo_update_pull_request` *(used by `describe`)*

| Param | Type | Required | Notes |
|---|---|---|---|
| `repositoryId` | string | yes | |
| `pullRequestId` | number | yes | |
| `project` | string | no | |
| `title` | string | no | |
| `description` | string | no | **max 4000 chars** |
| `isDraft` | boolean | no | |
| `targetRefName` | string | no | |
| `status` | enum | no | `Active` or `Abandoned` |
| `autoComplete` | boolean | no | |
| `mergeStrategy` | enum | no | default `NoFastForward` |
| `deleteSourceBranch` | boolean | no | |
| `transitionWorkItems` | boolean | no | default `true` |
| `bypassReason` | string | no | |
| `labels` | string[] | no | **REPLACES** existing label set — fetch & merge first |

#### `repo_vote_pull_request` *(used by `review`, second confirmation only)*

| Param | Type | Required | Notes |
|---|---|---|---|
| `repositoryId` | string | yes | |
| `pullRequestId` | number | yes | |
| `vote` | enum | yes | `Approved`(10), `Suggestions`(5), `None`(0), `Waiting`(-5), `Rejected`(-10) |
| `project` | string | no | |

#### `repo_reply_to_comment` *(used by `ask` when attaching to an existing thread)*

| Param | Type | Required |
|---|---|---|
| `repositoryId` | string | yes |
| `pullRequestId` | number | yes |
| `threadId` | number | yes |
| `content` | string | yes |
| `project` | string | no |

#### `repo_update_pull_request_thread`

Used to mark a thread `Fixed` / `Closed` after a re-review. Out of scope for v1, listed for completeness.

| Param | Type | Required |
|---|---|---|
| `repositoryId` | string | yes |
| `pullRequestId` | number | yes |
| `threadId` | number | yes |
| `project` | string | no |
| `status` | enum | no |

---

## Optional: `wit` toolset

Used only by `ado-pr-test-gaps`:

- `wit_get_work_items_batch_by_ids` — fetch titles + descriptions for the work-item refs returned by `repo_get_pull_request_by_id` with `includeWorkItemRefs: true`. Informs what test scenarios should exist.

If `wit` is not in the toolset header, `ado-pr-test-gaps` falls back to using only the work-item IDs and notes the limitation in its output.
