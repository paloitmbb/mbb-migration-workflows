# MBB GHES → GHEC Migration — Implementation Plan

## 1. Overview

This document describes the implementation plan for the **GitHub Enterprise Server (GHES) to GitHub Enterprise Cloud (GHEC)** repository migration solution. The system is driven by a GitHub Issues form and an automated GitHub Actions workflow that uses the **GitHub Enterprise Importer (GEI) CLI** to perform the migration end-to-end.

### Components

| Component | Path | Purpose |
|-----------|------|---------|
| Issue Template | `.github/ISSUE_TEMPLATE/ghec-migration-request.yml` | Self-service form for users to request a repository migration |
| Workflow | `.github/workflows/issue-ghec-migration.yml` | 7-job pipeline that validates, migrates, configures, and closes the request |
| Parse Issue Form Action | `.github/actions/parse-issue-form/action.yml` | Composite action to parse GitHub issue form bodies and extract field values |
| Validate Repos Action | `.github/actions/validate-repos/action.yml` | Composite action to validate source/target repository existence and availability |
| GitHub Stats Action | `.github/actions/gh-stats/action.yml` | Composite action to gather comprehensive repository statistics (19 metrics) |
| Migrate Branch Protections Action | `.github/actions/migrate-branch-protections/action.yml` | Composite action to migrate branch protection rules between repositories |

---

## 2. Prerequisites

Before the solution can be used, the following must be in place:

| # | Prerequisite | Details |
|---|-------------|---------|
| 1 | **GH_SOURCE_PAT** secret | A Personal Access Token with **read** access to the source GitHub.com organization (repo, admin:org scopes) |
| 2 | **GH_TARGET_PAT** secret | A Personal Access Token with **write/admin** access to the target GitHub.com organization (repo, admin:org, workflow scopes) |
| 3 | **Source org exists on GitHub.com** | The source GitHub.com organization must be accessible |
| 4 | **Target org exists on GitHub.com** | The target GitHub.com organization must already be created |
| 5 | **`migration-approval` environment** | A GitHub environment with required reviewers configured for the approval gate |

---

## 3. Issue Template — `ghec-migration-request.yml`

### 3.1 Form Structure

The issue template collects all information needed to execute a migration:

| Section | Field | Type | Required | Validation |
|---------|-------|------|----------|------------|
| **Source** | Source GHES URL | `input` | Yes | Placeholder URL (currently unused by workflow) |
| | Source Organization | `input` | Yes | — |
| | Source Repository Name | `input` | Yes | — |
| **Target (GHEC)** | Target Organization | `input` | Yes | Must exist on GitHub.com |
| | Target Repository Name | `input` | Yes | Must follow naming convention (lowercase, hyphens) |
| | Target Visibility | `dropdown` | Yes | `private` (default), `internal`, `public` |
| **Migration Options** | Checkboxes | `checkboxes` | No | 1 option: Archive source repository after successful migration |
| **Access** | Admins | `input` | Yes | Comma-separated GitHub usernames |
| | Team Access Mappings | `textarea` | No | Format: `source-team → target-team : permission` |
| **Other** | Justification | `textarea` | Yes | Free-text reason for migration |
| | Confirmation | `checkboxes` | Yes | 4 mandatory confirmations |

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
┌──────────┐    ┌───────────────┐    ┌──────────┐    ┌─────────┐
│ validate │───►│ pre-migration │───►│ approval │───►│ migrate │
└──────────┘    └───────────────┘    └──────────┘    └────┬────┘
                                                          │
                                                          ▼
                ┌─────────────┐    ┌──────────────────┐   │
                │ close-issue │◄───│    cutover        │◄──┘
                └─────────────┘    └──────┬───────────┘
                                          │
                                   ┌──────┴───────────┐
                                   │ post-migration    │
                                   └──────────────────┘
```

**Job dependency chain:**
- `validate` → independent (first job)
- `pre-migration` → needs `validate`
- `approval` → needs `validate`, `pre-migration`
- `migrate` → needs `validate`, `approval`
- `post-migration` → needs `validate`, `pre-migration`, `migrate`
- `cutover` → needs `validate`, `pre-migration`, `post-migration`
- `close-issue` → needs `validate`, `pre-migration`, `cutover`, `post-migration`

---

### 4.3 Job 1: Validate (`validate`)

**Purpose:** Parse the issue body, extract all fields, and validate the source and target repositories.

| # | Step | Description | Implementation Detail |
|---|------|-------------|----------------------|
| 1 | Checkout | Clone the repository | `actions/checkout@v4` |
| 2 | Parse Issue Form | Extract fields from issue body using regex-based composite action | `./.github/actions/parse-issue-form` — parses single-line fields, textarea fields (`team_mappings`, `justification`), and checkbox fields (`migration_options`) via configurable `field-mapping` JSON |
| 3 | Validate source and target repositories | Check source repo exists/not archived, target org exists, target repo name available | `./.github/actions/validate-repos` — posts error comments on issue if validation fails |
| 4 | Label as in-progress | Create (if needed) and apply `in-progress` label to issue | `gh label create` + `gh issue edit --add-label` |

**Outputs propagated to downstream jobs:**

- `source_organization`, `source_repo`, `target_organization`, `target_repo`
- `target_visibility`, `migration_options`, `admins`, `team_mappings`, `justification`
- `issue_number`

---

### 4.4 Job 2: Pre-Migration Setup (`pre-migration`)

**Purpose:** Record the source repository's current state for post-migration reporting and verification.

| # | Step | Description | Implementation Detail |
|---|------|-------------|----------------------|
| 1 | Checkout | Clone the repository | `actions/checkout@v4` |
| 2 | Parse migration options | Extract `archive_source` flag from checkboxes | String match on `"Archive source repository"` |
| 3 | Gather source repository stats | Capture 19 comprehensive repository metrics | `./.github/actions/gh-stats` composite action with paginated API calls |
| 4 | Post pre-migration summary | Post detailed metrics table and migration details to issue; add labels | `actions/github-script@v7` — builds issue comment + workflow job summary |

**Outputs (19 metrics + options):**

| Output | Description |
|--------|-------------|
| `source_branches` | Total branch count |
| `source_tags` | Total tag count |
| `source_head_sha` | HEAD commit SHA of default branch |
| `source_default_branch` | Default branch name |
| `source_protected_branches` | Number of protected branches |
| `source_pr_count` | Total PR count (all states) |
| `source_issue_count` | Total issue count (all states) |
| `source_release_count` | Total releases |
| `source_repo_size` | Human-readable repo size |
| `source_is_archived` | Whether repo is archived |
| `source_last_push` | Last push timestamp |
| `source_last_update` | Last update timestamp |
| `source_pr_review_count` | PRs with at least one review |
| `source_created` | Creation timestamp |
| `source_has_codeowners` | Whether CODEOWNERS file exists |
| `archive_source` | Whether to archive source post-migration |

**Pre-migration summary comment includes:**
- Migration details (source/target repo, visibility, admins)
- Full source repository stats table (19 metrics)
- CODEOWNERS alert if not present
- Migration options, team mappings, justification
- Labels added: `validation-passed`, `ghec-migration-request`

---

### 4.5 Job 3: Approval Gate (`approval`)

**Purpose:** Pause the workflow and require manual approval before proceeding with the migration.

| # | Step | Description |
|---|------|-------------|
| 1 | Comment approval requested | Post comment with migration summary table asking for review |
| 2 | Wait for approval | GitHub environment protection rule (`migration-approval`) blocks until a reviewer approves |
| 3 | Approval granted | Log confirmation |
| 4 | Comment approval granted | Post confirmation comment to issue |

**Dependencies:** Needs both `validate` and `pre-migration` to succeed.

**Implementation Requirements:**

- Create a GitHub environment named `migration-approval`
- Add required reviewers (team leads, admins) to the environment
- Optionally configure a wait timer

---

### 4.6 Job 4: Execute GEI Migration (`migrate`)

**Purpose:** Run the GitHub Enterprise Importer CLI to migrate the repository from source to target on GitHub.com.

| # | Step | Description |
|---|------|-------------|
| 1 | Install GEI CLI | `gh extension install github/gh-gei` |
| 2 | Run GEI migration | Execute `gh gei migrate-repo` with all parameters |

**GEI Command (GitHub.com → GitHub.com):**

```bash
gh gei migrate-repo \
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

**Purpose:** Configure the target repository with proper access, protection rules, variables, secrets listing, and security settings.

| # | Step | Description | API Used |
|---|------|-------------|----------|
| 1 | Wait 15s | Let GitHub.com finalize the import | `sleep 15` |
| 2 | Add admin collaborators | Iterate comma-separated admin list, add each as admin | `PUT /repos/{org}/{repo}/collaborators/{user}` |
| 3 | Inject CODEOWNERS file | If source has no CODEOWNERS, auto-generate one with admin owners | `PUT /repos/{org}/{repo}/contents/.github/CODEOWNERS` (conditional) |
| 4 | Apply team access mappings | Parse `source → target : permission` format, apply each | `PUT /orgs/{org}/teams/{team}/repos/{org}/{repo}` |
| 5 | Migrate branch protection rules | Full migration of all branch protection settings (see §4.7.1) | Multiple `GET/PUT` calls per protected branch |
| 6 | Migrate variables | Copy repository-level Actions variables from source to target | `GET /repos/{org}/{repo}/actions/variables` → `POST` to target |
| 7 | List Secret Names | List repo-level, Dependabot, and environment secret names (values not readable) | `GET /repos/{org}/{repo}/actions/secrets`, `/dependabot/secrets`, `/environments/{env}/secrets` |
| 8 | Migrate GHAS permissions | Multi-step security migration (see §4.7.2) | Multiple GitHub.com API calls |
| 9 | Comment post-migration setup | Post summary with all migration results and manual action items | `gh issue comment` with env vars |

**Outputs:**

| Output | Description |
|--------|-------------|
| `secret_names` | Comma-separated secret names by scope (repo-level, Dependabot) |
| `secret_count` | Total number of secrets found |
| `secret_error` | Error message if secrets could not be listed |

#### 4.7.1 Branch Protection Migration

The `Migrate branch protection rules` step performs a comprehensive migration:

**Strategy:**
1. First attempts `?protected=true` filter to list protected branches
2. Falls back to listing ALL branches and probing each for protection rules (handles free-plan orgs where rules exist but are "Not enforced")

**Settings migrated automatically:**

| Setting | Details |
|---------|---------|
| Required status checks | Strict mode + check names (app_id reset to -1 for cross-org compatibility) |
| Enforce admins | Boolean |
| Required PR reviews | Dismiss stale reviews, require code owner reviews, approving review count, require last push approval |
| Required linear history | Boolean |
| Allow force pushes | Boolean (simple mode) |
| Allow deletions | Boolean |
| Block creations | Boolean |
| Required conversation resolution | Boolean |
| Allow fork syncing | Boolean |
| Required signatures | Separate POST endpoint |

**Settings skipped (logged for manual re-configuration):**

| Setting | Reason |
|---------|--------|
| Push restrictions (users/teams/apps) | Actor-specific — IDs differ between orgs |
| PR review dismissal restrictions | Actor-specific |
| PR bypass allowances | Actor-specific |
| Lock branch | Not migratable via API |
| Required deployments | Environment-specific |

#### 4.7.2 Advanced Security (GHAS) Migration Sub-steps

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

### 4.8 Job 6: Cutover & Cleanup (`cutover`)

**Purpose:** Archive the source repo (if requested) after successful post-migration setup.

| # | Step | Condition | Description |
|---|------|-----------|-------------|
| 1 | Checkout | Always | Clone repo for potential manifest updates |
| 2 | Archive source on GitHub.com | `archive_source == 'true'` | `PATCH https://api.github.com/repos/{org}/{repo}` with `{"archived": true, "description": "[MIGRATED]..."}` |

**Gate:** Runs after `validate`, `pre-migration`, and `post-migration` succeed.

---

### 4.9 Job 7: Close Issue (`close-issue`)

**Purpose:** Post a final success comment, generate a workflow summary, and close the issue.

| # | Step | Description |
|---|------|-------------|
| 1 | Generate apply summary | Write detailed GitHub Actions job summary with migration details, verification results, secrets info, and next steps |
| 2 | Post success comment | Post comprehensive completion comment to issue with migration details table, verification results, admin access, secrets list, and next steps |
| 3 | Close issue | Close issue as `completed`, remove `in-progress` label, add `completed` label |

**Gate:** Runs when `cutover` succeeded or was skipped (via `always()` condition). Depends on `validate`, `pre-migration`, `cutover`, and `post-migration`.

**Success comment includes:**
- Migration details (source/target, visibility, default branch, run link)
- Verification results table (branches, tags, HEAD SHA)
- Migration options applied
- Admin access grants
- Post-migration actions (archive status, branch protection, manifest)
- Secrets requiring manual re-creation (with table of names and scopes)
- Next steps checklist

---

## 5. Secrets Configuration

| Secret Name | Scope | Required Permissions |
|-------------|-------|---------------------|
| `GITHUB_TOKEN` | Auto-provided | `issues: write`, `contents: read` |
| `GH_SOURCE_PAT` | GitHub.com (source org) | `repo` (read), `admin:org` (read) on the source GitHub.com org |
| `GH_TARGET_PAT` | GitHub.com (target org) | `repo`, `admin:org`, `workflow`, `delete_repo` on the target GitHub.com org |

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
| Missing required fields | Validation composite action posts error to issue; workflow exits |
| Source repo not found / archived | Validate-repos action posts error to issue; workflow exits |
| Target repo name already taken | Validate-repos action posts error to issue; workflow exits |
| Target org not found | Validate-repos action posts error to issue; workflow exits |
| GEI migration failure | Migration job fails; GEI output captured via `tee` |
| Branch protection partial failure | Actor-specific settings logged as manual items; job continues |
| GHAS migration partial failure | Issue continues; manual steps documented in comment |
| CODEOWNERS injection failure | Warning logged; job continues |
| Team access mapping failure | Warning logged per team; job continues |
| Secrets listing failure | Error reported in post-migration comment; job continues |
| Archive source failure | Warning logged; issue still closes successfully |

---

## 9. Implementation Checklist

### Phase 1: Foundation

- [x] Create issue template (`ghec-migration-request.yml`) with all form fields
- [x] Add input validation (naming convention, required fields)
- [x] Add confirmation checkboxes for user acknowledgment (4 confirmations)
- [x] Document prerequisites in the template header

### Phase 2: Composite Actions

- [x] Implement `parse-issue-form` composite action (regex-based field extraction)
- [x] Implement `validate-repos` composite action (source exists, not archived, target org exists, name available)
- [x] Implement `gh-stats` composite action (19 repository metrics with pagination)
- [x] Implement `migrate-branch-protections` composite action (full protection rule migration with fallback probe)

### Phase 3: Workflow — Validation & Pre-Migration

- [x] Implement issue body parsing via `parse-issue-form` action (Job 1)
- [x] Implement repository validation via `validate-repos` action (Job 1)
- [x] Implement pre-migration state recording with comprehensive stats (Job 2)
- [x] Post detailed pre-migration summary with stats table to issue (Job 2)
- [x] Add `migration-approval` environment gate (Job 3)

### Phase 4: Workflow — Migration Execution

- [x] Implement GEI CLI installation and migration execution (Job 4)
- [x] Capture GEI output via `tee`

### Phase 5: Workflow — Post-Migration Setup

- [x] Implement admin collaborator assignment (Job 5)
- [x] Implement CODEOWNERS auto-injection when not present in source
- [x] Implement team access mapping parser and applicator
- [x] Implement comprehensive branch protection migration (with fallback probe and manual item tracking)
- [x] Implement Actions variables migration (repository-level)
- [x] Implement secret names listing (repo-level, Dependabot, environment)
- [x] Implement GHAS permissions migration (6 sub-steps with graceful degradation)
- [x] Implement post-migration summary comment with all results

### Phase 6: Workflow — Cutover & Closure

- [x] Implement source repo archival on GitHub.com (Job 6)
- [x] Implement workflow job summary generation (Job 7)
- [x] Implement issue closure with comprehensive completion comment (Job 7)
- [x] Implement label lifecycle management (`in-progress` → `completed`)

### Phase 7: Operational Readiness

- [ ] Create `migration-approval` environment with required reviewers
- [ ] Generate and store `GH_SOURCE_PAT` with correct source org scopes
- [ ] Generate and store `GH_TARGET_PAT` with correct target org scopes
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
| Source repo archived | Archived source repo | Validation fails with "cannot migrate archived repo" |
| Target repo exists | Pre-existing target repo name | Validation fails with "already exists" |
| Target org not found | Non-existent target org | Validation fails with org not found error |
| Approval rejected | Reviewer denies approval | Workflow stops at approval gate |
| GEI failure | Network issue during migration | Migration job fails, error captured via tee |
| No CODEOWNERS in source | Source repo without CODEOWNERS | CODEOWNERS auto-injected in target with admin owners |
| Branch protection with actor settings | Source has push restrictions | Migratable settings applied, actor-specific logged as manual |
| Partial GHAS migration | GHAS not licensed on target | GHAS steps report partial, list manual actions |
| Variables migration | Source has Actions variables | Variables copied to target repo |
| Secrets listing | Source has repo + Dependabot secrets | Secret names listed in post-migration comment |
| Archive source disabled | Checkbox unchecked | Cutover skips archival, issue still closes |
| Archive source enabled | Checkbox checked | Source repo archived on GitHub.com |

---

## 11. Security Considerations

1. **PAT rotation**: Both `GH_SOURCE_PAT` and `GH_TARGET_PAT` should be rotated periodically
2. **Least privilege**: PATs should have the minimum scopes required
3. **Approval gate**: Prevents unauthorized migrations via the `migration-approval` environment
4. **No secrets in logs**: All `curl` responses are written to temp files, not echoed directly
5. **Branch protection**: Automatically migrated from source to target post-migration
6. **GHAS parity**: Security settings from source are replicated to target where possible
7. **CODEOWNERS enforcement**: Auto-injected when missing from source to ensure code review gates
8. **Secret values not exposed**: Only secret names are listed — values cannot be read via API

---

## 12. Known Limitations

| Limitation | Impact | Mitigation |
|------------|--------|------------|
| Branch/tag pagination | Repos with >100 branches/tags are handled by `gh-stats` pagination | `gh-stats` action uses paginated API calls |
| GEI does not migrate GitHub Actions secrets | Secrets must be re-added manually | Secret names listed in post-migration comment for reference |
| GEI does not migrate Actions variables | Variables must be re-created manually | Variables are migrated separately via REST API in post-migration |
| Actor-specific branch protection settings | Push restrictions, dismissal restrictions, bypass allowances skipped | Logged in issue comment for manual re-configuration |
| Lock branch setting | Not migratable via branch protection API | Logged in issue comment for manual action |
| Required deployments | Environment-specific, skipped during migration | Logged in issue comment for manual action |
| Org-level security managers | Cannot be migrated programmatically | Reported in issue comment for manual action |
| Webhooks not migrated | Webhooks must be reconfigured manually | Document in runbook |
| GitHub Pages configuration | Not migrated by GEI | Manual reconfiguration needed |
| Deploy keys | Not migrated by GEI | Manual reconfiguration needed |
| Rulesets | Migration attempted but depends on org plan support | Logged with status in post-migration comment |
