# Asana PR Review Task

A reusable GitHub Actions workflow that automatically creates an Asana review subtask whenever a reviewer is requested on a pull request, assigns it, sets a due date, fills a custom "Workload Estimate" field, and drops a link to it as a PR comment.

Designed to be called from any repository across any number of GitHub organizations via `workflow_call`, with per-org configuration stored as organization variables and secrets.

---

## Features

- Reads the linked Asana task from the PR description (supports legacy `/0/...` and current `/1/...` URL formats).
- Resolves the linked task's parent so review tasks live alongside the original work item instead of nesting inside an existing subtask.
- Assigns the review subtask to a GitHub reviewer via a GitHub username → Asana user GID map.
- Sets a due date for today (configurable timezone).
- Fills an Asana custom field (e.g. "Workload Estimate") with a configurable numeric value.
- Repositions the new subtask at the bottom of the parent's subtask list (Asana defaults to top).
- Posts a PR comment linking to the new Asana task.
- All calls use bounded timeouts and conservative retries; transient failures fall back gracefully without blocking the workflow.

---

## Installation

### 1. Create an Asana Personal Access Token (PAT)

1. Sign in to Asana as the user who will own the automation. Anything this user can see in Asana, the workflow can read and write, so a dedicated service user is ideal if your plan supports it.
2. Open the [Personal Access Tokens page](https://app.asana.com/0/my-apps).
3. Click **"+ New access token"**, name it (e.g. `github-actions-review-task`), and accept the API terms.
4. Copy the token — Asana only shows it once.

### 2. Add the PAT as a GitHub secret

In the consuming organization (the org where your project repos live):

1. Go to **Organization Settings → Secrets and variables → Actions → Secrets** tab.
2. Click **New organization secret**.
3. Name: `ASANA_PAT`
4. Value: the token you copied.
5. Scope it to the repos that will use this workflow (all repos, or a curated list).

If you manage multiple consuming orgs, repeat this in each.

### 3. Add the user map as an organization variable

The workflow expects a JSON object mapping GitHub usernames to Asana user GIDs.

1. Go to **Organization Settings → Secrets and variables → Actions → Variables** tab.
2. Click **New organization variable**.
3. Name: `ASANA_USER_MAP`
4. Value (example):
   ```json
   {"octocat":"1234567890123456","hubot":"6543210987654321","kelseymcauley":"15944996466906"}
   ```
5. Scope it to the repos that will use this workflow.

GitHub usernames are case-sensitive — match the exact casing on each user's profile.

#### How to find Asana user GIDs

```bash
export ASANA_PAT="<your token>"

curl -s -H "Authorization: Bearer $ASANA_PAT" \
  "https://app.asana.com/api/1.0/workspaces/YOUR_WORKSPACE_ID/users?opt_fields=gid,name,email" \
  | jq '.data[] | {gid, name, email}'
```

Replace `YOUR_WORKSPACE_ID` with your workspace GID (visible in any Asana URL).

### 4. Add the custom field GID as an organization variable (optional)

If you want the workflow to populate the "Workload Estimate" (or any other number custom field):

1. Variables tab → **New organization variable**.
2. Name: `ASANA_WORKLOAD_FIELD_GID`
3. Value: the numeric GID of the custom field (no quotes needed in the GitHub UI).

#### How to find the custom field GID

```bash
curl -s -H "Authorization: Bearer $ASANA_PAT" \
  "https://app.asana.com/api/1.0/tasks/<any_task_id>?opt_fields=custom_fields.gid,custom_fields.name,custom_fields.resource_subtype" \
  | jq '.data.custom_fields[] | select(.name == "Workload Estimate")'
```

Expected output (if it's a number field):

```json
{
  "gid": "1234567890123456",
  "name": "Workload Estimate",
  "resource_subtype": "number"
}
```

If `resource_subtype` is not `"number"`, see [Limitations](#limitations) below.

### 5. Add the caller workflow to each consuming repo

Create `.github/workflows/asana-review-task.yml` in each repo that needs the automation:

```yaml
name: Asana Review Task

on:
  pull_request:
    types: [review_requested]

jobs:
  create-task:
    uses: Raborn-Digital/Asana-PR-Review-Task/.github/workflows/asana-review-task.yml@main
    with:
      user-map: ${{ vars.ASANA_USER_MAP }}
      workload-field-gid: ${{ vars.ASANA_WORKLOAD_FIELD_GID }}
    secrets:
      ASANA_PAT: ${{ secrets.ASANA_PAT }}
```

That's it. Open a PR, request a review, and the workflow handles the rest.

---

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `user-map` | Yes | — | JSON object mapping GitHub usernames to Asana user GIDs. Typically `${{ vars.ASANA_USER_MAP }}`. |
| `workload-field-gid` | No | `''` | Asana custom field GID for "Workload Estimate". Leave empty to skip. |
| `workload-value` | No | `1` | Numeric value to set for the Workload Estimate custom field. |
| `due-date-timezone` | No | `America/New_York` | IANA timezone name used to compute "today" for the due date. |

## Secrets

| Secret | Required | Description |
|---|---|---|
| `ASANA_PAT` | Yes | Asana Personal Access Token with access to the target workspace. |

---

## PR body requirements

The workflow extracts an Asana task ID from the PR description. Add a PR template to enforce this. Example:

```markdown
## Asana Task

<!-- Paste the Asana task URL below. Any of the following formats works: -->
<!-- https://app.asana.com/0/{project}/{task} -->
<!-- https://app.asana.com/1/{workspace}/project/{project}/task/{task} -->

[View Task](https://app.asana.com/1/…/task/…)
```

If no Asana URL is found in the PR body, the workflow exits cleanly (status 0) — no failure is reported.

---

## Permissions

The reusable workflow declares the permissions it needs on the job it runs in. However, because this is called via `workflow_call`, the **calling workflow's** token permissions still apply as the ceiling. Verify this setting in every consuming repo (or at the organization level):

**Settings → Actions → General → Workflow permissions** set to either:
- **Read and write permissions**, or
- **Read repository contents and packages permissions** (the restrictive default — works because the reusable workflow declares `pull-requests: write` on its job).

If your org enforces a read-only policy, `gh pr comment` will 403 and the comment step will log a warning (but the Asana subtask is still created).

---

## Cross-organization usage

This workflow lives in a public repository (`Raborn-Digital/Asana-PR-Review-Task`), which means **any organization or user on GitHub can call it** from their own workflows, regardless of plan level. No cross-org sharing configuration is needed.

Each consuming org sets up its own:
- `ASANA_PAT` secret
- `ASANA_USER_MAP` variable
- `ASANA_WORKLOAD_FIELD_GID` variable

Consuming repos just add the tiny caller workflow shown above.

---

## Versioning

Consumers pin to a specific reference in the `uses:` line:

```yaml
uses: Raborn-Digital/Asana-PR-Review-Task/.github/workflows/asana-review-task.yml@main
```

Options:

- `@main` — always tracks the latest. Simplest, but consumers get updates automatically (including breaking changes).
- `@v1`, `@v1.2.0` — release tags. Recommended once the workflow is stable. Consumers manually opt in to upgrades.
- `@<sha>` — a specific commit SHA. Most secure (immune to tag reassignment), least convenient.

For internal org use, `@main` is fine if you own both the host and consumer repos. For anything shared more widely, tag releases.

---

## Troubleshooting

### The action runs but the task isn't created

Check the Actions log for:

- `❌ No Asana task link found in PR body.` — the PR description doesn't contain an Asana URL. Workflow exits cleanly.
- `❌ Failed to create subtask. HTTP status: 401` — the `ASANA_PAT` secret is invalid, expired, or missing.
- `❌ Failed to create subtask. HTTP status: 403` — the PAT owner doesn't have access to the target project.
- `❌ Failed to create subtask. HTTP status: 404` — the task ID extracted from the PR body doesn't exist.

### The task is created but unassigned

- The reviewer isn't in `ASANA_USER_MAP`. Add them.
- The review was requested from a **team**, not an individual. This is expected — GitHub doesn't expose a specific user in that case.

### The task appears at the top of the subtask list

Check for `⚠️  Failed to reposition subtask`. The PAT may lack permission to modify the parent task's sibling order. Reposition failures are non-fatal — the task itself is still created.

### No comment on the PR

Check for `⚠️  Failed to post PR comment`. Most likely a workflow permissions issue — see [Permissions](#permissions).

### "user-map input is empty"

The `ASANA_USER_MAP` organization variable isn't set, or its access scope doesn't include the consuming repo. Check **Organization Settings → Secrets and variables → Actions → Variables** and confirm the variable exists and the consuming repo has access.

---

## Limitations

- **Only resolves one parent level.** If a PR links to a sub-subtask, the review lands one step up (under the middle subtask), not at the top-level work item. Extending to full upward traversal is possible but hasn't been needed.
- **Siblings query is capped at 100.** For parents with more than 100 direct subtasks, we grab the 100th (in Asana's order) as the insert-after target and miss anything beyond. Very rare edge case.
- **Custom field must be a number.** If your Workload Estimate field is an enum (single-select dropdown) or text, the current implementation will reject the value. To support enums, swap `--argjson` for `--arg` in the payload step and supply the enum option's GID instead of a raw number.
- **Fork PRs cannot use this workflow.** GitHub redacts secrets (including `ASANA_PAT`) when a PR comes from a fork, and the workflow's `GITHUB_TOKEN` is read-only in that context. This is a first-party-repos-only automation by default.

---

## How it works

On a `pull_request: review_requested` event, the reusable workflow runs a single bash step that:

1. Looks up the reviewer in the `user-map` JSON input.
2. Extracts the Asana task ID from the PR body via PCRE regex (handles legacy and current URL formats).
3. Calls `GET /tasks/{id}?opt_fields=parent.gid,parent.name` to find the parent.
4. Calls `GET /tasks/{target}/subtasks?opt_fields=gid&limit=100` to find the last existing sibling.
5. Builds the task payload with `jq`, conditionally including assignee and custom field only if present.
6. Calls `POST /tasks/{target}/subtasks` to create the subtask.
7. Calls `POST /tasks/{new_subtask}/setParent` with `insert_after: <last_sibling>` to move it to the bottom.
8. Calls `gh pr comment` to drop the Asana link on the PR.

All network calls use `--connect-timeout 10 --max-time 30 --retry N --retry-delay 2 --retry-connrefused`. GET calls fall back gracefully on failure; the task-creation POST surfaces failures as a non-zero exit.

---

## Contributing

Open an issue or PR. Keep in mind that any consumer pinned to `@main` picks up changes immediately, so breaking changes should be released behind a new major version tag (`v2`) and announced.

---

## License

MIT.
