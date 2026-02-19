# MBB GHES → GHEC Migration — Implementation Plan

## 1. Overview

This document describes the implementation plan for the **GitHub Enterprise Server (GHES) to GitHub Enterprise Cloud (GHEC)** repository migration solution. The system is driven by a GitHub Issues form and an automated GitHub Actions workflow that uses the **GitHub Enterprise Importer (GEI) CLI** to perform the migration end-to-end.

### Components

| Component | Path | Purpose |
|-----------|------|---------|
| Issue Template | `.github/ISSUE_TEMPLATE/ghec-migration-request.yml` | Self-service form for users to request a repository migration |
| Workflow | `.github/workflows/issue-ghec-migration.yml` | 7-job pipeline that validates, migrates, configures, verifies, and closes the request |

---

## 2. Prerequisites

Before the solution can be used, the following must be in place:

| # | Prerequisite | Details |
|---|-------------|---------|
| 1 | **GH_SOURCE_PAT** secret | A Personal Access Token with **read** access to the source GHES instance (repo, org scopes) |
| 2 | **GH_TARGET_PAT** secret | A Personal Access Token with **write/admin** access to the target GHEC organization (repo, admin:org, workflow scopes) |
| 3 | **GHES version ≥ 3.8** | Required for direct migration without intermediate blob storage |
| 4 | **Target org exists on GHEC** | The target GitHub.com organization must already be created |
| 5 | **Network connectivity** | GitHub Actions runners must be able to reach the GHES instance URL |
| 6 | **`migration-approval` environment** | A GitHub environment with required reviewers configured for the approval gate |

---

## 3. Issue Template — `ghec-migration-request.yml`

### 3.1 Form Structure

The issue template collects all information needed to execute a migration:

| Section | Field | Type | Required | Validation |
|---------|-------|------|----------|------------|
| **Source (GHES)** | Source GHES URL | `input` | Yes | Must be a valid HTTP(S) URL |
| | Source Organization | `input` | Yes | — |
| | Source Repository Name | `input` | Yes | — |
| **Target (GHEC)** | Target Organization | `input` | Yes | Must exist on GitHub.com |
| | Target Repository Name | `input` | Yes | Regex: `^[a-z0-9]+(-[a-z0-9]+)+$` |
| | Target Visibility | `dropdown` | Yes | `private` (default), `internal`, `public` |
| **Migration Options** | Checkboxes | `checkboxes` | No | 5 options (commit history, PRs, issues, releases, archive source) |
| **Access** | Admins | `input` | Yes | Comma-separated GitHub usernames |
| | Team Access Mappings | `textarea` | No | Format: `source-team → target-team : permission` |
| **Other** | Justification | `textarea` | Yes | Free-text reason for migration |
| | Confirmation | `checkboxes` | Yes | 3 mandatory confirmations |

### 3.2 Naming Convention

Target repository names must follow the pattern:

```
<org><region>-<dept>-<type>[tiering-optional]-<name>
```

Example: `mbbmy-dbdo-app-mae-mobile`

### 3.3 Implementation Steps

| # | Task | Status |
|---|------|--------|
| 1 | Define issue template YAML with all fields | Done |
| 2 | Add regex validation for target repo naming | Done |
| 3 | Add mandatory confirmation checkboxes | Done |
| 4 | Add prerequisite documentation in the template header | Done |

---

## 4. Workflow — `issue-ghec-migration.yml`

### 4.1 Trigger

```yaml
on:
  issues:
    types: [opened]
```

The workflow triggers when an issue is opened and filters by title prefix `[GHEC Migration]` (set automatically by the issue template).

### 4.2 Job Pipeline

The workflow consists of **7 sequential jobs** with dependency gates:

```
┌──────────┐    ┌──────────┐    ┌───────────────┐    ┌─────────┐
│ validate │───►│ approval │───►│ pre-migration │───►│ migrate │
└──────────┘    └──────────┘    └───────────────┘    └────┬────┘
                                                          │
                                                          ▼
                                ┌─────────────┐    ┌──────────────────┐
                                │   verify     │◄───│ post-migration   │
                                └──────┬──────┘    └──────────────────┘
                                       │
                                       ▼
                                ┌──────────┐    ┌─────────────┐
                                │ cutover  │───►│ close-issue │
                                └──────────┘    └─────────────┘
```

---

### 4.3 Job 1: Validate (`validate`)

**Purpose:** Parse the issue body, extract all fields, and validate them against the source GHES and target GHEC.

| # | Step | Description | Implementation Detail |
|---|------|-------------|----------------------|
| 1 | Checkout | Clone the repository | `actions/checkout@v4` |
| 2 | Parse Issue Form | Extract fields from issue body using regex | `actions/github-script@v7` with custom JS parsing functions (`extractField`, `extractTextarea`, `extractCheckboxes`) |
| 3 | Display parsed values | Log all parsed values for debugging | Shell `echo` statements |
| 4 | Validate required fields | Check all mandatory fields are non-empty; validate GHES URL format (`^https?://`); validate target repo naming convention | Shell script with error accumulation |
| 5 | Validate GHES reachable | `GET {ghes_url}/api/v3/meta` with `GH_SOURCE_PAT` | `curl` with 15s timeout; HTTP 200 = pass, 000 = fail, other = warn |
| 6 | Validate source repo exists | `GET {ghes_api}/repos/{org}/{repo}` | Also checks if repo is archived (blocks migration) |
| 7 | Validate target org exists | `GET https://api.github.com/orgs/{org}` (falls back to `/users/{org}`) | Uses `GH_TARGET_PAT` |
| 8 | Check target repo name available | `GET https://api.github.com/repos/{org}/{repo}` | HTTP 200 means name is taken → fail |
| 9 | Post validation results | Post detailed comment to issue | `actions/github-script@v7` — adds labels (`validation-passed` or `validation-failed`) |
| 10 | Comment validation passed | Summary table posted to issue | `peter-evans/create-or-update-comment@v4` |
| 11 | Generate validation summary | Write GitHub Actions job summary | `core.summary.addRaw().write()` |
| 12 | Label as in-progress | Add `in-progress` label to issue | `gh issue edit --add-label` |

**Outputs propagated to downstream jobs:**

- `source_organization`, `source_repo`, `target_organization`, `target_repo`
- `target_visibility`, `migration_options`, `admins`, `team_mappings`, `justification`
- `ghes_url`, `source_api_url`, `issue_number`

---

### 4.4 Job 2: Approval Gate (`approval`)

**Purpose:** Pause the workflow and require manual approval before proceeding with the migration.

| # | Step | Description |
|---|------|-------------|
| 1 | Comment approval requested | Post comment with migration summary asking for review |
| 2 | Wait for approval | GitHub environment protection rule (`migration-approval`) blocks until a reviewer approves |
| 3 | Comment approval granted | Post confirmation comment |

**Implementation Requirements:**

- Create a GitHub environment named `migration-approval`
- Add required reviewers (team leads, admins) to the environment
- Optionally configure a wait timer

---

### 4.5 Job 3: Pre-Migration Setup (`pre-migration`)

**Purpose:** Record the source repository's current state for post-migration verification.

| # | Step | Description | API Endpoint (GHES) |
|---|------|-------------|---------------------|
| 1 | Parse migration options | Extract `archive_source` flag from checkboxes | String match on `"Archive source repository"` |
| 2 | Record source state | Capture branch count, tag count, HEAD SHA, default branch | `GET /repos/{org}/{repo}`, `/branches`, `/tags`, `/branches/{default}` |
| 3 | Comment pre-migration state | Post metrics table to issue | `peter-evans/create-or-update-comment@v4` |

**Outputs:**

- `source_branches`, `source_tags`, `source_head_sha`, `source_default_branch`, `archive_source`

---

### 4.6 Job 4: Execute GEI Migration (`migrate`)

**Purpose:** Run the GitHub Enterprise Importer CLI to migrate the repository.

| # | Step | Description |
|---|------|-------------|
| 1 | Install GEI CLI | `gh extension install github/gh-gei` |
| 2 | Run GEI migration | Execute `gh gei migrate-repo` with all parameters |
| 3 | Comment migration result | Post success/failure with GEI output |

**GEI Command:**

```bash
gh gei migrate-repo \
  --ghes-api-url "${GHES_API_URL}" \
  --github-source-org "$SRC_ORG" \
  --source-repo "$SRC_REPO" \
  --github-target-org "$TGT_ORG" \
  --target-repo "$TGT_REPO" \
  --target-repo-visibility "$VISIBILITY" \
  --verbose
```

**Environment Variables:**

| Variable | Source |
|----------|--------|
| `GH_SOURCE_PAT` | `secrets.GH_SOURCE_PAT` |
| `GH_PAT` | `secrets.GH_TARGET_PAT` |

---

### 4.7 Job 5: Post-Migration Setup (`post-migration`)

**Purpose:** Configure the target repository with proper access, protection rules, and security settings.

| # | Step | Description | API Used |
|---|------|-------------|----------|
| 1 | Wait 15s | Let GitHub.com finalize the import | `sleep 15` |
| 2 | Add admin collaborators | Iterate comma-separated admin list, add each as admin | `PUT /repos/{org}/{repo}/collaborators/{user}` |
| 3 | Apply team access mappings | Parse `source → target : permission` format, apply each | `PUT /orgs/{org}/teams/{team}/repos/{org}/{repo}` |
| 4 | Enable branch protection | Set branch protection on default branch (require PR reviews, dismiss stale reviews, enforce on admins) | `PUT /repos/{org}/{repo}/branches/{branch}/protection` |
| 5 | Migrate GHAS permissions | Multi-step security migration (see §4.7.1) | Multiple GHES & GHEC API calls |
| 6 | Comment results | Post summary with GHAS status | `peter-evans/create-or-update-comment@v4` |

#### 4.7.1 Advanced Security (GHAS) Migration Sub-steps

| Sub-step | Action | Source API | Target API |
|----------|--------|------------|------------|
| 1 | Read source GHAS settings | `GET /repos/{org}/{repo}` → `security_and_analysis` | — |
| 2 | Enable Advanced Security on target | — | `PATCH /repos/{org}/{repo}` with `advanced_security.status: "enabled"` |
| 3 | Apply Secret Scanning settings | — | `PATCH /repos/{org}/{repo}` with `secret_scanning`, `push_protection`, `validity_checks`, `non_provider_patterns` |
| 4 | Enable Dependabot | — | `PUT /repos/{org}/{repo}/vulnerability-alerts` + `PATCH` with `dependabot_security_updates` |
| 5 | Migrate Code Scanning setup | `GET /repos/{org}/{repo}/code-scanning/default-setup` | `PATCH /repos/{org}/{repo}/code-scanning/default-setup` |
| 6 | Check Security Manager teams | `GET /orgs/{org}/security-managers` | Informational only (org-level, manual re-config required) |

**Graceful degradation:** If any GHAS step fails, it is logged to `GHAS_MANUAL_FILE` and reported in the issue comment as requiring manual configuration. The job continues (does not fail hard).

---

### 4.8 Job 6: Verify Migration Integrity (`verify`)

**Purpose:** Compare source and target repository states to confirm migration fidelity.

| Check | Source | Target | Pass Criteria |
|-------|--------|--------|---------------|
| Branch count | `pre-migration.source_branches` | `GET /repos/{org}/{repo}/branches` on GHEC | Exact match |
| Tag count | `pre-migration.source_tags` | `GET /repos/{org}/{repo}/tags` on GHEC | Exact match |
| HEAD SHA | `pre-migration.source_head_sha` (first 7 chars) | `GET /repos/{org}/{repo}/branches/{default}` on GHEC | First 7 chars match |

**Output:** `verification_passed` = `true` / `false`

---

### 4.9 Job 7: Cutover & Cleanup (`cutover`)

**Purpose:** Archive the source repo (if requested) after successful verification.

| # | Step | Condition | Description |
|---|------|-----------|-------------|
| 1 | Checkout | Always | Clone repo for potential manifest updates |
| 2 | Archive source on GHES | `archive_source == 'true'` | `PATCH {ghes_api}/repos/{org}/{repo}` with `{"archived": true, "description": "[MIGRATED]..."}` |

**Gate:** Only runs if `verify.verification_passed == 'true'`

---

### 4.10 Job 8: Close Issue (`close-issue`)

**Purpose:** Post a final success comment, generate a workflow summary, and close the issue.

| # | Step | Description |
|---|------|-------------|
| 1 | Generate apply summary | Write a detailed job summary with migration details, verification results, and next steps |
| 2 | Post success comment | Post comprehensive completion comment to issue with migration details table |
| 3 | Close issue | Close issue as `completed`, remove `in-progress` label, add `completed` label |

**Gate:** Runs only when `verify.verification_passed == 'true'` AND `cutover` succeeded or was skipped.

---

## 5. Secrets Configuration

| Secret Name | Scope | Required Permissions |
|-------------|-------|---------------------|
| `GITHUB_TOKEN` | Auto-provided | `issues: write`, `contents: read` |
| `GH_SOURCE_PAT` | GHES instance | `repo` (read), `admin:org` (read) on the source GHES |
| `GH_TARGET_PAT` | GitHub.com (GHEC) | `repo`, `admin:org`, `workflow`, `delete_repo` on the target GHEC org |

---

## 6. Environment Setup

| Environment Name | Purpose | Configuration |
|-----------------|---------|---------------|
| `migration-approval` | Gate between validation and execution | Add required reviewers (e.g., platform team leads). Optionally add a wait timer. |

---

## 7. Label Management

The workflow automatically manages the following labels:

| Label | Color | When Applied | When Removed |
|-------|-------|-------------|--------------|
| `ghec-migration-request` | — | On issue creation (via template) | Never |
| `validation-passed` | — | After successful validation | Never |
| `validation-failed` | — | After failed validation | Never |
| `in-progress` | `#FBCA04` | After validation passes | On issue close |
| `completed` | — | On successful migration | Never |

---

## 8. Error Handling Strategy

| Scenario | Behavior |
|----------|----------|
| Missing required fields | Issue comment with specific errors; workflow exits |
| GHES unreachable | Issue comment with troubleshooting tips; workflow exits |
| Source repo not found / archived | Issue comment explaining the problem; workflow exits |
| Target repo name already taken | Issue comment; workflow exits |
| GEI migration failure | Issue comment with GEI output; workflow exits |
| GHAS migration partial failure | Issue continues; manual steps documented in comment |
| Verification failure (branch/tag/SHA mismatch) | Cutover is skipped; issue remains open for investigation |
| Archive source failure | Warning logged; issue still closes successfully |

---

## 9. Implementation Checklist

### Phase 1: Foundation

- [x] Create issue template (`ghec-migration-request.yml`) with all form fields
- [x] Add input validation (regex for repo naming, required fields)
- [x] Add confirmation checkboxes for user acknowledgment
- [x] Document prerequisites in the template header

### Phase 2: Workflow — Validation & Approval

- [x] Implement issue body parsing with `actions/github-script` (Job 1)
- [x] Implement field-level validation (empty checks, URL format, naming convention)
- [x] Implement GHES connectivity check (`/meta` endpoint)
- [x] Implement source repo existence & archived check
- [x] Implement target org existence check (with user fallback)
- [x] Implement target repo name availability check
- [x] Post detailed validation results as issue comment
- [x] Add `migration-approval` environment gate (Job 2)

### Phase 3: Workflow — Migration Execution

- [x] Implement pre-migration state recording (branches, tags, HEAD SHA) (Job 3)
- [x] Implement GEI CLI installation and migration execution (Job 4)
- [x] Capture and post GEI output to issue

### Phase 4: Workflow — Post-Migration & Verification

- [x] Implement admin collaborator assignment (Job 5)
- [x] Implement team access mapping parser and applicator
- [x] Implement branch protection configuration
- [x] Implement GHAS permissions migration (6 sub-steps)
- [x] Implement migration integrity verification (Job 6)

### Phase 5: Workflow — Cutover & Closure

- [x] Implement source repo archival on GHES (Job 7)
- [x] Implement issue closure with completion summary (Job 8)
- [x] Implement label lifecycle management

### Phase 6: Operational Readiness

- [ ] Create `migration-approval` environment with required reviewers
- [ ] Generate and store `GH_SOURCE_PAT` with correct GHES scopes
- [ ] Generate and store `GH_TARGET_PAT` with correct GHEC scopes
- [ ] Verify network connectivity from GitHub Actions runners to GHES
- [ ] Run end-to-end test with a non-critical repository
- [ ] Document runbook for manual intervention scenarios
- [ ] Establish monitoring/alerting for failed migration workflows

---

## 10. Testing Plan

| Test Case | Input | Expected Outcome |
|-----------|-------|-----------------|
| Happy path | Valid source/target, all fields populated | Full migration completes, issue closed with `completed` label |
| Missing required field | Empty source org | Validation fails, error posted to issue |
| Invalid repo name | `MyRepo_123` | Validation fails with naming convention error |
| Source repo archived | Archived GHES repo | Validation fails with "cannot migrate archived repo" |
| Target repo exists | Pre-existing target repo name | Validation fails with "already exists" |
| GHES unreachable | Invalid/offline GHES URL | Validation fails with connectivity error |
| Approval rejected | Reviewer denies approval | Workflow stops at approval gate |
| GEI failure | Network issue during migration | Migration job fails, error posted to issue |
| Partial GHAS migration | GHAS not licensed on target | GHAS steps report partial, list manual actions |
| Verification mismatch | Tag count differs | Cutover skipped, issue remains open |
| Archive source disabled | Checkbox unchecked | Cutover skips archival, issue still closes |

---

## 11. Security Considerations

1. **PAT rotation**: Both `GH_SOURCE_PAT` and `GH_TARGET_PAT` should be rotated periodically
2. **Least privilege**: PATs should have the minimum scopes required
3. **Approval gate**: Prevents unauthorized migrations via the `migration-approval` environment
4. **No secrets in logs**: All `curl` responses are written to temp files, not echoed directly
5. **Branch protection**: Automatically applied post-migration to prevent accidental force-pushes
6. **GHAS parity**: Security settings from source are replicated to target where possible

---

## 12. Known Limitations

| Limitation | Impact | Mitigation |
|------------|--------|------------|
| Branch/tag pagination | Repos with >100 branches/tags may show incorrect counts in verification | Use paginated API calls or GitHub CLI |
| GEI does not migrate GitHub Actions secrets | Secrets must be re-added manually | Document in "Next Steps" comment |
| Org-level security managers | Cannot be migrated programmatically | Reported in issue comment for manual action |
| Webhooks not migrated | Webhooks must be reconfigured manually | Document in runbook |
| GitHub Pages configuration | Not migrated by GEI | Manual reconfiguration needed |
| Deploy keys | Not migrated by GEI | Manual reconfiguration needed |
