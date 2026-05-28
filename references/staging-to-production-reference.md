# Staging to Production Promotion Reference

This reference contains project-agnostic command templates for opening, reusing, and inspecting `staging` → `production` promotion PRs. Replace placeholders with project-supplied values.

## Shell Variables

```bash
# Defaults for this promotion lane. Override only when project docs explicitly use different branch names.
SOURCE_BRANCH="${SOURCE_BRANCH:-staging}"
TARGET_BRANCH="${TARGET_BRANCH:-production}"
PR_TITLE="${PR_TITLE:-chore(release): promote ${SOURCE_BRANCH} into ${TARGET_BRANCH}}"
EXPECTED_TITLE="${EXPECTED_TITLE:-$PR_TITLE}"
PR_BODY_FILE="${PR_BODY_FILE:-$(mktemp)}"
REPOS=("<owner>/<repo-one>" "<owner>/<repo-two>")
# Include the root/meta repository here when the project uses one for release tracking.
```


## Repository Topology Setup

Default to monorepo mode. For a monorepo, use one repository and track release surfaces separately:

```bash
REPOS=("<owner>/<monorepo>")
RELEASE_SURFACES=("<apps/web>" "<packages/shared>")
```

For multi-repo projects, list each independent repository:

```bash
REPOS=("<owner>/<repo-one>" "<owner>/<repo-two>")
```

For submodule-aware projects, project policy determines whether to include root repo, submodule repos, or both:

```bash
ROOT_REPO="<owner>/<root-repo>"
SUBMODULE_REPOS=("<owner>/<submodule-one>" "<owner>/<submodule-two>")
REPOS=("$ROOT_REPO" "${SUBMODULE_REPOS[@]}")
```

Do not assume a root repository PR updates submodule repository branches. Confirm whether the release requires submodule source-target PRs, root pointer updates, both, or neither.

## Branch Discovery

```bash
for REPO in "${REPOS[@]}"; do
  git ls-remote --heads "git@github.com:${REPO}.git" "$SOURCE_BRANCH" "$TARGET_BRANCH"
done
```

Use the default `staging` and `production` branch names unless the project explicitly documents a different ladder. Stop before mutations if either required branch is missing, ambiguous, or unexpectedly protected.

## Branch Preservation

Treat both `SOURCE_BRANCH` and `TARGET_BRANCH` as permanent release branches. Do not delete either branch, use merge cleanup flags that delete either branch, or instruct the user to use GitHub's **Delete branch** button unless a documented project policy explicitly requires it and the user authorizes the exact repository and branch.

When reporting a production PR handoff, tell the user that any provider branch-deletion prompt should be ignored for these source and target branches because they are supposed to remain available for future promotions.

## Plain-Language Progress and Link Reporting Template

Use this reference with user-facing narration. Before each command or mutation, explain in simple language what is about to happen, why it matters, whether it changes anything, and what evidence or links it should produce. After each command, explain the result, what it means, the next action, and all useful links.

Minimum step-update format:

```text
Next step: <plain-English action>
Why: <risk or decision this protects>
Changes anything?: <read-only | creates PR | updates PR | merges | verification only>
Expected evidence: <PR URL | compare URL | checks | deployment URL | logs | health URL>
```

```text
Result: <success | no-op | blocker | pending | failure>
What it means: <beginner-friendly interpretation>
Links: <GitHub PR, Files changed, Checks, compare, Vercel deployment/dashboard, logs, runtime URL>
What you should do next: <review PR | wait for checks | approve | ask release owner | provide missing input | no action>
```

PR handoff explanation:

```text
Here is the pull request: <pr-url>
What this is: a GitHub review page for the `staging → production` promotion.
What to do: open it, read the description, inspect Files changed, check the Checks tab, and read comments.
Boundary: Does not merge or deploy production; it prepares an understandable handoff.
Branch precaution: do not delete the `staging` or `production` branches if GitHub offers branch cleanup; these are permanent promotion branches.
If something is red or pending: do not approve/merge yet; use the blocker list below.
```

Useful links to collect when applicable:

```bash
COMPARE_URL="https://github.com/${REPO}/compare/${TARGET_BRANCH}...${SOURCE_BRANCH}"
gh pr view <number> -R "<owner>/<repo>" --json url,number,title,body,headRefOid,baseRefName,headRefName,statusCheckRollup
# Check objects often include direct CI/deployment links.
gh pr checks <number> -R "<owner>/<repo>" --watch=false --json name,state,bucket,link,workflow,startedAt,completedAt
```

For Vercel, report exact links exposed by GitHub deployment cards, Vercel comments, or Vercel CLI output. If scope/project/deployment are known and verified, include the deployment URL and dashboard/project URL; otherwise say which Vercel input is missing instead of guessing.

## List Existing PRs

```bash
for REPO in "${REPOS[@]}"; do
  gh pr list -B "$TARGET_BRANCH" -H "$SOURCE_BRANCH" -R "$REPO" \
    --state open --json number,url,title,body,headRefOid
done
```

## Create Missing PRs

Infer the PR title and body before creating anything. Use the default lane-derived title unless project docs define an exact metadata contract. Build the body from verified branch, diff, topology, release-surface, review/check, and production-handoff evidence:

```bash
cat > "$PR_BODY_FILE" <<'EOF'
## Summary
Promotes `<source-branch>` into `<target-branch>` for `<owner>/<repo>`.

## Scope
- Repository topology: `<monorepo|multi-repo|submodule-aware>`
- Release surfaces: `<apps/packages/deployments inferred from changed paths>`

## Diff evidence
- Source SHA: `<source-sha>`
- Target SHA: `<target-sha>`
- Compare or commit summary: `<compare-url-or-commit-count>`
- Risk notes: `<migrations/lockfiles/infra/generated/env/secrets/none>`

## Handoff status
- Review/check policy: `<required checks and approvers>`
- Downstream owner: `<CI/release operator/follow-up skill>`
- Boundary: Production merge, deployment, and smoke verification are downstream steps outside this skill.

## Unknowns / follow-up
- `<none or explicit missing project inputs>`
EOF
```

Create only when no matching open PR exists and branches are not equal:

```bash
gh pr create -B "$TARGET_BRANCH" -H "$SOURCE_BRANCH" -R "<owner>/<repo>" \
  --title "$PR_TITLE" --body-file "$PR_BODY_FILE"
```

Do not use an empty body unless project policy explicitly requires it.

If GitHub reports no diff, treat that as no PR needed for the repo.

## Inspect PR State

```bash
gh pr view <number> -R "<owner>/<repo>" \
  --json state,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,comments,reviews,url,headRefOid
gh pr checks <number> -R "<owner>/<repo>" --watch=false
```


## GitHub PR Review Commands

Use these commands to build the PR review packet. Verify command flags with `gh --help` when local versions differ.

```bash
gh pr view <number> -R "<owner>/<repo>" --comments \
  --json state,isDraft,author,title,body,baseRefName,headRefName,headRefOid,mergeable,mergeStateStatus,reviewDecision,reviewRequests,latestReviews,comments,reviews,statusCheckRollup,files,commits,url
gh pr diff <number> -R "<owner>/<repo>" --name-only
gh pr diff <number> -R "<owner>/<repo>" --patch
gh pr checks <number> -R "<owner>/<repo>" --watch=false --json name,state,bucket,link,workflow,startedAt,completedAt
gh pr checks <number> -R "<owner>/<repo>" --required --watch=false
gh pr view <number> -R "<owner>/<repo>" --web
```

Review packet fields to report:

1. PR URL, number, title, draft state, source branch, target branch, and `headRefOid`.
2. Author, latest reviews, requested reviewers, review decision, and unresolved discussion risk.
3. Changed file count and risky paths: infrastructure, dependencies, lockfiles, generated files, migrations, secrets, environment files, release scripts, CI/CD config, and monorepo app/package ownership.
4. Required and optional check state, including failing or pending check links.
5. Deployment cards, preview links, or provider comments shown on GitHub that are not captured by CLI JSON, mapped to monorepo release surfaces when applicable.
6. Human-readable recommendation: ready, blocked, no diff, waiting for checks, waiting for review, metadata mismatch, or conflict.

## Validate PR Metadata

```bash
for REPO in "${REPOS[@]}"; do
  PR=$(gh pr list -B "$TARGET_BRANCH" -H "$SOURCE_BRANCH" -R "$REPO"     --state open --json number,title,body --jq '.[0]')
  if [ -z "$PR" ]; then echo "OK: $REPO -- no open PR"; continue; fi
  NUM=$(echo "$PR" | jq -r '.number')
  TITLE=$(echo "$PR" | jq -r '.title')
  BODY=$(echo "$PR" | jq -r '.body // ""')
  if [ "$TITLE" = "$EXPECTED_TITLE" ]; then
    echo "OK: $REPO #$NUM -- title matches"
  else
    echo "FAIL: $REPO #$NUM -- expected '$EXPECTED_TITLE', got '$TITLE'"
  fi
  if [ -n "$BODY" ] && printf '%s' "$BODY" | grep -q "## Summary"; then
    echo "OK: $REPO #$NUM -- inferred body is present"
  else
    echo "FAIL: $REPO #$NUM -- PR body is missing inferred metadata sections"
  fi
done
```

Do not edit, close, recreate, merge, or deploy production PRs unless the user explicitly authorizes a separate workflow. If authorized, use the project-approved close/recreate process, for example:

```bash
gh pr close <number> -R "<owner>/<repo>" --comment "Closing: incorrect metadata for this promotion workflow."
gh pr create -B "$TARGET_BRANCH" -H "$SOURCE_BRANCH" -R "<owner>/<repo>" \
  --title "$PR_TITLE" --body-file "$PR_BODY_FILE"
```


## Vercel Preview Analysis Template

Use this only when GitHub PR checks, deployment cards, provider comments, project docs, or operator input identify Vercel as relevant. This is read-only PR evidence, not post-merge verification.

```bash
vercel inspect <deployment-url-or-id> --format=json --scope <scope>
vercel inspect <deployment-url-or-id> --logs --scope <scope>
vercel logs --deployment <deployment-url-or-id> --json --since 30m --scope <scope>
vercel logs --project <project-name> --environment preview --branch <branch-name> --level error --since 1h --json --scope <scope>
```

Analysis checklist:

1. Identify preview deployment URL/ID from GitHub checks, PR deployment cards, provider comments, project docs, or operator input.
2. Use `vercel inspect --format=json` and inspect the actual JSON keys before scripting assumptions.
3. Confirm project, branch, PR/source context, status, age, aliases, and Git metadata when exposed.
4. Compare Vercel commit/branch metadata to `headRefOid` when available.
5. Use build logs to explain failed or pending checks.
6. Use runtime logs only as handoff evidence unless the user asks for a separate smoke-test workflow.
7. Include Vercel dashboard/project links, deployment URLs, GitHub deployment/check links, and log links or exact log commands when available.
8. Do not mutate aliases, domains, environment variables, or protection settings in this PR-only workflow.

## Error Handling

| Scenario | Expected behavior | Action |
|----------|-------------------|--------|
| No diff | Branches are equal | Report no PR needed |
| Conflicting PR | Candidate and production diverged | Report blocker; do not push production directly |
| Failing checks | CI or release gate failure | List check names and URLs |
| Changes requested | Approval blocker | Report review state |
| Missing branch | Remote branch absent | Stop and ask for production branch policy |

## Reporting Checklist

Start every report with:

- Plain-language result.
- What it means.
- What the user should do next.
- Useful links for the user.

Per repository, report:

1. Existing PR reused, new PR created, no diff, or blocked.
2. PR number and URL.
3. Title/body metadata validation pass/fail.
4. Mergeability and review decision.
5. Failing or pending status checks.
6. Conflicts, branch-protection blockers, or missing branches.
7. Handoff note: production merge, deployment, and smoke verification happen outside this skill.
8. Whether any root/meta repository tracking PR was included or explicitly out of scope.
9. Monorepo release-surface mapping or submodule/root policy result.
10. Useful links for the user: PR URL, Files changed URL, Checks/CI links, compare URL, deployment/dashboard links, log links, and runtime/health URLs when applicable.
