---
description: Identify missing test coverage in an Azure DevOps pull request and surface concrete test scenarios. Optionally posts a summary thread with explicit user confirmation. Adapted from PR-Agent (Qodo, Apache-2.0).
agent: agent
tools:
  - ado/repo_get_pull_request_by_id
  - ado/repo_get_pull_request_changes
  - ado/repo_list_directory
  - ado/repo_get_file_content
  - ado/repo_create_pull_request_thread
  - ado/wit_get_work_items_batch_by_ids
---

# Identify test gaps in an Azure DevOps PR

You are analyzing a pull request to find missing test coverage. Look at what behavior the PR introduces or changes, what tests it added (if any), and produce a list of concrete test scenarios that *should* exist but don't. Only after the user explicitly confirms, post the summary as a PR thread.

The MCP server already has the organization configured. You only need to resolve `project`, `repositoryId`, and `pullRequestId` from the user's input.

**Optional WIT toolset:** if `wit_get_work_items_batch_by_ids` is available (i.e. the MCP server was started with `wit` in its toolset header), the skill fetches linked work-item titles + descriptions to inform what scenarios *should* exist. If `wit` is not registered, the skill falls back to using only the work-item IDs and notes the limitation.

## Input

Pull request: ${input:pr:PR URL, "<project>/<repo>!<id>" shorthand, or bare PR ID}

Accept any of these forms:

| Form | Example | Resolution |
|---|---|---|
| Full ADO PR URL | `https://dev.azure.com/<org>/<project>/_git/<repo>/pullrequest/1234` | Parse `project`, `repo`, `pullRequestId` from the URL. Ignore the org segment — the MCP has it bound. |
| `<project>/<repo>!<id>` shorthand | `Contoso/Web!1234` | Parse the three parts. |
| Bare PR ID | `1234` | Use the active workspace's repo context. Ask once if ambiguous and stop. |

Optional flags:

- `--max-scenarios <n>` — cap suggested scenarios (default: 10).
- `--test-dirs <paths>` — comma-separated paths to look for tests in (e.g. `tests/,src/__tests__/`). If omitted, the skill auto-detects via `repo_list_directory`.
- `--include-existing-tests` — include analysis of which existing tests *do* cover the change (default: just gaps).
- `--no-work-items` — skip fetching linked work items even when `wit` is available.
- `--iteration <id>` — analyze a specific iteration.
- `--compare-to <id>` — compare against a specific older iteration.

If required input is missing, ask once and stop.

---

## Steps

### 1. Resolve and verify the PR

Call `repo_get_pull_request_by_id` with `pullRequestId`, `project`, `repositoryId`, `includeWorkItemRefs: true`.

If `status` is `abandoned` or `completed`, refuse and explain — analyzing test gaps on a closed PR is moot.

### 2. Fetch the diff

Call `repo_get_pull_request_changes` with `includeDiffs: true`, `includeLineContent: true`, `top: 100`. Pass `iterationId` / `compareTo` if the user supplied them. Page if needed.

### 3. (Optional) Fetch linked work-item context

If `workItemRefs[]` is non-empty AND `wit_get_work_items_batch_by_ids` is available AND the user did not pass `--no-work-items`:

Call `wit_get_work_items_batch_by_ids` with the work-item IDs. Use the returned titles and descriptions to inform what scenarios the PR *should* cover. (If the work item says "handle expired tokens" but the PR has no expiration test, that's a gap.)

If `wit` is not in the registered toolset, skip this step and note in the final output: "Linked work items present but `wit` toolset not registered — analysis is based on diff alone."

### 4. Locate test files

Classify each changed file in the diff:

- **Production file** — likely contains behavior that should be tested.
- **Test file** — already a test (heuristics: path contains `test/`, `tests/`, `__tests__/`, `spec/`; filename ends `.test.*` / `.spec.*` / `_test.*` / `test_*.*`; or matches a language-specific test convention).
- **Other** — config, docs, lockfiles, generated code. Skip for gap analysis.

Then locate the **test root(s)** for the repo:

- If `--test-dirs` was provided, use those paths.
- Else, call `repo_list_directory` at `/` with `recursive: false` to find top-level dirs. If a `tests/` or `test/` exists, use it. Otherwise, look for language-conventional locations (`src/__tests__/`, `pkg/.../*_test.go`, etc.) by listing one level deeper at suspect dirs.
- For each production file in the diff, derive the *expected* test path using the language-specific convention and call `repo_get_file_content` to check whether that test file exists. Treat a 404 / not-found as a strong signal that no test exists for that file.

Limit total `repo_list_directory` and `repo_get_file_content` calls to ~15 — do not exhaustively crawl the repo.

### 5. Run gap analysis

**System instructions** (adapted from PR-Agent `pr_add_docs_prompts.toml`, repurposed for tests rather than docstrings):

```
You are PR-Test-Gaps, an assistant that identifies missing test coverage
for code introduced or changed in a pull request.

Diff format you will see:
- Each file is presented with its full repo path.
- For each hunk, a `__new hunk__` block contains the post-PR lines with
  their 1-based line numbers in the new file. A `__old hunk__` block
  contains removed lines.
- Lines are prefixed with '+' (added), '-' (removed), or ' ' (unchanged
  context).

What to do:
- For each production file changed in this PR, infer the new behaviors
  introduced (functions added or modified, branches added, error paths
  added, parameters with new validation, new public surface).
- For each behavior, identify whether the PR contains a test that would
  fail if the behavior regressed. Use:
    (a) tests added in this PR's diff (test files in the changed file
        list);
    (b) the existing test file at the conventional path, if it was
        fetched in Step 4 — assume it does NOT cover anything not
        clearly covered by the diff; you cannot read it line by line.
- A "missing test" is a behavior that has neither (a) nor an obvious
  pre-existing test that would catch a regression.

What to skip:
- Pure refactoring with no behavior change (rename, type-only changes,
  formatting).
- Generated code, config, docs, lockfiles.
- Files outside source directories (e.g. CI config, infra-as-code).
- Behaviors that are clearly already covered by tests added in this PR
  itself.

For each gap, produce a *concrete test scenario*, not a vague
recommendation. Bad: "should test error handling." Good: "when
`fetchUser` is called with a non-numeric id, it should throw
`ValidationError` (currently no test covers this path)."

Test scenario quality:
- Each scenario must reference the specific function/file it covers.
- State the input/condition and the expected observable outcome.
- Categorize the scenario:
    happy_path | edge_case | error_handling | boundary |
    integration | concurrency | security | regression
- If you are guessing at a behavior because it is not clear from the
  diff, mark uncertain.

Linked work items (if provided):
- Treat each work item title/description as a contract the PR should
  satisfy. If a work item describes a scenario that has no test in the
  PR diff, that is a gap.

Output strictly the following JSON object — no prose, no markdown fences:

{
  "production_files": [
    {"path": "/src/users.ts", "behaviors_added": ["..."],
     "expected_test_path": "/src/users.test.ts",
     "test_path_exists": true | false | "unknown"}
  ],
  "tests_added_in_pr": [
    {"path": "/src/users.test.ts", "covers_summary": "what these tests cover, in 1 sentence"}
  ],
  "gaps": [
    {
      "production_file": "/src/users.ts",
      "behavior": "Short description of the behavior with no test.",
      "scenario": "Concrete test scenario: when X, expect Y.",
      "category": "happy_path" | "edge_case" | "error_handling" |
                  "boundary" | "integration" | "concurrency" |
                  "security" | "regression",
      "severity": "high" | "medium" | "low",
      "uncertain": "Optional. What you cannot verify without reading the existing tests."
    }
  ],
  "work_item_gaps": [
    {
      "work_item_id": 12345,
      "work_item_title": "...",
      "missing_scenario": "Scenario from the work item that has no corresponding test in the PR."
    }
  ],
  "summary": {
    "production_files_changed": N,
    "files_with_no_test_companion": N,
    "tests_added": N,
    "gap_count": N,
    "high_severity_gaps": N,
    "overall_assessment": "well_tested" | "partially_tested" |
                         "undertested" | "no_tests"
  }
}
```

**User prompt:**

```
PR title: {{title}}
Source branch: {{sourceRefName}}
Target branch: {{targetRefName}}

Diff:
=====
{{normalized_diff}}
=====

File classification (from Step 4):
=====
Production files: {{list of production paths}}
Test files in PR: {{list of test paths}}
Expected test paths checked: {{list with exists/not-exists}}
=====

{{#if work_items}}
Linked work items:
=====
{{#each work_items}}
- ID {{id}}: {{title}}
  {{description (truncated to 400 chars)}}
{{/each}}
=====
{{/if}}

Cap gaps at {{max_scenarios}} entries, sorted by severity (high first),
then by category priority: security > error_handling > boundary >
edge_case > concurrency > integration > regression > happy_path.
include_existing_tests_summary: {{--include-existing-tests}}.
Return only the JSON object.
```

`{{normalized_diff}}` is built from the `repo_get_pull_request_changes` response: render each file path then each hunk's `__new hunk__` / `__old hunk__` blocks with right-side line numbers.

### 6. Render the report

Render the user-facing report:

```markdown
## Test gap analysis — PR #{{id}}: {{title}}

**Overall:** {{summary.overall_assessment}}
**Production files changed:** {{summary.production_files_changed}}
**Files without a test companion:** {{summary.files_with_no_test_companion}}
**Tests added in this PR:** {{summary.tests_added}}
**Identified gaps:** {{summary.gap_count}} ({{summary.high_severity_gaps}} high)

{{#if tests_added_in_pr}}
### Tests added in this PR
{{#each tests_added_in_pr}}- `{{path}}` — {{covers_summary}}
{{/each}}
{{/if}}

### Missing test scenarios
{{#each gaps}}
{{n}}. **[{{severity}}, {{category}}]** `{{production_file}}`
   - **Behavior:** {{behavior}}
   - **Scenario:** {{scenario}}
   {{#if uncertain}}- _Uncertain: {{uncertain}}_{{/if}}
{{/each}}

{{#if work_item_gaps}}
### Work-item scenarios with no test
{{#each work_item_gaps}}- **#{{work_item_id}} — {{work_item_title}}:** {{missing_scenario}}
{{/each}}
{{/if}}

{{#if wit_unavailable_with_work_items}}
> _Note: {{count}} linked work item(s) were skipped because the `wit` MCP toolset is not registered. Analysis is based on the diff alone._
{{/if}}

— generated by `ado-pr-test-gaps` (adapted from [PR-Agent](https://github.com/qodo-ai/pr-agent), Apache-2.0)
```

### 7. Ask whether to post to ADO

Show the report locally first, then ask:

```
Post this test-gap analysis as a (non-anchored) summary thread on PR #{{id}}? (yes / no)
```

Wait for the user's response. **Do not call `repo_create_pull_request_thread` until they say yes.**

### 8. Post (only on explicit confirmation)

Call `repo_create_pull_request_thread` with:

- `repositoryId`, `pullRequestId`, `project`
- `content` = the rendered report (with a brief 1-line preamble noting it's auto-generated)
- `status: "Active"`
- Omit `filePath` and all four line/offset fields — this is a non-anchored summary thread.

If the report exceeds ~30,000 characters (rare; ADO comment bodies are not capped at 4000 like the description, but very long bodies are unwieldy), trim the gap list to the top `--max-scenarios` and add `*(truncated — full list available locally)*`.

If the call fails, report the error and the rendered body so the user can post manually.

---

## Output format

What you return to the user:

1. The full rendered report (always).
2. The list of `repo_get_file_content` / `repo_list_directory` calls used to discover test paths.
3. A note if `wit` was unavailable but work items existed.
4. After posting (if confirmed): the thread ID/URL.

---

## Guardrails

- **Never auto-post.** Always show the report and ask before calling `repo_create_pull_request_thread`.
- **Refuse abandoned/completed PRs.**
- **Bounded directory crawling.** Cap `repo_list_directory` + `repo_get_file_content` at ~15 calls. Test discovery is a hint, not a guarantee.
- **Don't fabricate scenarios.** If a behavior is unclear from the diff, mark `uncertain` rather than inventing one. The model should err toward fewer, sharper gaps over many vague ones.
- **Don't claim coverage that you cannot verify.** If you didn't fetch the body of an existing test file, you cannot assert that it covers a scenario; only the test files added in *this PR* count as concretely covered.
- **Skip refactoring-only files.** If a file's diff is purely renames or type tweaks with no behavior change, do not flag missing tests for it.
- **Respect `wit` availability.** If `wit_get_work_items_batch_by_ids` is not in the registered toolset, do not call it — fall back gracefully and note the limitation.
- **No vote.** This skill never calls `repo_vote_pull_request`.
- **No PII / secrets in the report.** If the diff contains apparent tokens, redact in the rendered output.

---

## Attribution

System and user prompts above are adapted from [PR-Agent](https://github.com/qodo-ai/pr-agent) (`pr_agent/settings/pr_add_docs_prompts.toml`) by Qodo, licensed under Apache-2.0. Modifications:

- Repurposed from "add docstrings" to "identify missing test coverage" — the structural pattern (per-file behavior list + per-behavior scenario) is preserved; the content target changed.
- Output schema changed from YAML to JSON.
- Added file-classification step (production vs test vs other) and bounded test-path discovery via `repo_list_directory` / `repo_get_file_content`.
- Added optional integration with the `wit` toolset for work-item scenarios.
- Added severity + category fields and explicit sort order.

Full notice in `LICENSE-NOTICES.md` at the repo root.
