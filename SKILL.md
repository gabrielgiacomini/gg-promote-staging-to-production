---
name: promote-staging-to-production
description: Use when opening or reusing staging-to-production promotion PRs across a project-defined repository set, inferring PR titles/descriptions, inspecting merge readiness, validating metadata, and reporting production handoff without merging or deployment verification.
---

# GG → Promote Staging To Production → PR Handoff

> **Snapshot age:** reusable operational guidance. Verify live `gh` and `git` CLI flags before running commands in a target repository.

## Overview

Use this skill for a project-agnostic `staging` → `production` release handoff. Branch names default to `staging` → `production` and must be verified remotely before mutation. Monorepos are the default topology unless project docs say otherwise. It opens or reuses promotion PRs with automatically inferred titles and descriptions, validates metadata, inspects readiness, and reports blockers before human approval and CI-owned production deployment. The source and target release branches are permanent branches and must not be deleted unless a project policy explicitly requires it and the user authorizes the exact repository and branch.

This workflow does **not** merge PRs, direct-push production branches, force deploy, or run production smoke tests. Those actions require explicit follow-up workflows and project-specific production authority.

## When to Use This Skill

**TRIGGER when:**
- The user asks to promote a frozen staging candidate into production.
- The user wants staging-to-production PRs opened, reused, inspected, or metadata-validated.
- A production handoff report is needed before human approval or CI deployment.

**SKIP when:**
- The source branch is `main`, `development`, or a feature branch.
- The user wants direct target-branch pushes or force deploys.
- The user asks to merge PRs or verify live production deployment in the same workflow.

## Required Project Inputs

| Input | Purpose | Example placeholder |
|-------|---------|---------------------|
| Repository topology | Monorepo by default, multi-repo, or submodule-aware | monorepo |
| Repository set | Repos included in production release | `<owner>/<repo>` list; one repo for monorepos |
| Release surfaces | Apps/packages/workspaces/deployments affected inside each repo | `<apps/web>`, `<packages/api>` |
| Source branch default | Use `staging` unless project docs explicitly override it; verify remotely | `staging` |
| Target branch default | Use `production` unless project docs explicitly override it; verify remotely | `production` |
| Branch preservation policy | Source and target release branches are permanent; never delete them unless explicitly required and authorized | preserve source and target |
| PR metadata policy | Optional title/body convention; otherwise infer from lane and evidence | inferred title and description |
| Approval policy | Human/branch protection requirements | project-defined |
| Root/meta repository policy | Whether a root or meta repo also needs a tracking PR | optional |
| Post-merge verification owner | Who verifies deployment after merge | CI, release operator, or follow-up skill |

## Quick Commands

```bash
# Branch defaults are built in; override only for non-standard projects after verification.
SOURCE_BRANCH="${SOURCE_BRANCH:-staging}"
TARGET_BRANCH="${TARGET_BRANCH:-production}"
PR_TITLE="${PR_TITLE:-chore(release): promote ${SOURCE_BRANCH} into ${TARGET_BRANCH}}"
EXPECTED_TITLE="${EXPECTED_TITLE:-$PR_TITLE}"
PR_BODY_FILE="${PR_BODY_FILE:-$(mktemp)}"
# Monorepo default: use a one-entry REPOS array. Add more repos only when the project is multi-repo or submodule-aware.
REPOS=("<owner>/<monorepo-or-repo>")

# Branch discovery.
for REPO in "${REPOS[@]}"; do
  git ls-remote --heads "git@github.com:${REPO}.git" "$SOURCE_BRANCH" "$TARGET_BRANCH"
done

# Existing PR inspection before creation.
gh pr list -B "$TARGET_BRANCH" -H "$SOURCE_BRANCH" -R "<owner>/<repo>" \
  --state open --json number,url,title,body,headRefOid

# Infer the PR body from verified promotion evidence before creation.
cat > "$PR_BODY_FILE" <<'EOF'
## Summary
Promotes `<source-branch>` into `<target-branch>` for `<owner>/<repo>`.

## Scope
- Repository topology: `<monorepo|multi-repo|submodule-aware>`
- Release surfaces: `<apps/packages/deployments inferred from changed paths>`

## Diff evidence
- Source SHA: `<source-sha>`
- Target SHA: `<target-sha>`
- Changed files / risk notes: `<summary>`

## Handoff status
- GitHub reviews/checks to inspect: `<required checks and reviewers>`
- Downstream owner: `<CI/release operator/follow-up skill>`
- Boundary: Production merge, deployment, and smoke verification are downstream steps outside this skill.

## Unknowns / follow-up
- `<none or explicit missing project inputs>`
EOF

# Create only when no matching PR exists and the branches differ.
gh pr create -B "$TARGET_BRANCH" -H "$SOURCE_BRANCH" -R "<owner>/<repo>" \
  --title "$PR_TITLE" --body-file "$PR_BODY_FILE"

# Inspect readiness and checks; do not merge in this workflow.
gh pr view <number> -R "<owner>/<repo>" \
  --json state,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,comments,reviews,url,headRefOid
gh pr checks <number> -R "<owner>/<repo>" --watch=false
```

For the full reusable command template, see `references/staging-to-production-reference.md`.


## Repository Topology Modes

Default to **monorepo mode** unless project docs or the operator identify multiple repos or submodules.

| Mode | How to Model Scope | PR Shape | Reporting Unit |
|------|--------------------|----------|----------------|
| **Monorepo default** | `REPOS=("<owner>/<monorepo>")` plus a release-surface list | One source-target PR | Per app/package/workspace/deployment surface |
| **Multi-repo** | Multiple independent repositories in `REPOS` | One PR per repository | Per repository plus shared release surface |
| **Submodule repo** | Root repo plus submodule repos when project policy requires both | Root PR and/or submodule PRs | Per root/submodule repo and per deployed surface |

For monorepos, the handoff must identify impacted release surfaces, not just the single repository:

- app/workspace/package paths changed by the PR
- shared libraries that can affect multiple apps
- lockfiles and dependency manifests
- database migrations or generated clients
- infrastructure, CI/CD, routing, and environment files
- deployment-provider projects mapped to each app path

For repos with submodules, do not assume the root PR is sufficient. Confirm whether the project requires submodule repo PRs, a root pointer-update PR, both, or root-only tracking. Report the topology choice explicitly.

## Branch Defaults and Verification

Default branch names for this skill are:

| Role | Default branch |
|------|----------------|
| Source | `staging` |
| Target | `production` |

Do not ask the user to supply these branch names for standard workflows. Instead, verify that both branches exist in every in-scope repository before creating PRs or reporting handoff readiness. If either branch is missing, ambiguous, protected in an unexpected way, or project docs explicitly use a different branch ladder, stop and report the mismatch before mutating anything.

Override `SOURCE_BRANCH` or `TARGET_BRANCH` only when project docs or the operator explicitly identify non-standard branch names.

### Branch Preservation Requirement

The source and target branches for this workflow are long-lived release branches, not disposable feature branches. Never delete either branch, pass merge flags that delete either branch, close a PR with branch cleanup, or tell the user to click GitHub's **Delete branch** button for `SOURCE_BRANCH` or `TARGET_BRANCH` unless a documented project policy explicitly requires that cleanup and the user authorizes the exact repository and branch.

Whenever reporting a production PR handoff, remind the user that the source and target branches should remain in place after the promotion. If GitHub or another provider offers branch deletion after merge, tell the user not to delete these branches because they are supposed to be permanent promotion lanes.

## PR Metadata Inference

When this skill creates a PR, infer the title and description automatically. Do not ask the user to write routine PR text.

Default inferred title pattern:

```text
chore(release): promote <source-branch> into <target-branch>
```

Use a project-provided exact title only when project docs, branch protection, or release automation explicitly require one. Otherwise derive the title from the active source and target branch names.

Infer the PR description from verified evidence before calling `gh pr create`:

1. **Promotion lane:** source branch, target branch, repository, topology mode, and whether this is monorepo, multi-repo, or submodule-aware.
2. **Diff evidence:** source SHA, target SHA, commit count or compare URL, changed-file summary, and risky paths such as migrations, lockfiles, generated files, infrastructure, CI/CD, secrets, or environment files.
3. **Release surfaces:** impacted apps, packages, workspaces, deployments, or submodules discovered from changed paths and project docs.
4. **Handoff context:** review/check policy, required approvers, downstream verification owner, and the explicit boundary that merge/deploy/smoke checks happen outside this skill.
5. **Known gaps:** missing project inputs, blocked checks, unresolved reviewers, unknown release surfaces, or provider details still needed.

Never create promotion PRs with an empty body unless the project explicitly requires blank descriptions. If any metadata input is unknown, include an `Unknowns / follow-up` section rather than inventing project facts.

## Plain-Language Progress Updates and Handoffs

This skill must be useful while it runs, not just after it finishes. Assume the user may have little coding experience and may not know what a branch, pull request, check, deployment, or Vercel preview means.

Before every meaningful step, explain:

- **What I am about to do:** one plain-English sentence.
- **Why it matters:** the release risk or decision the step protects.
- **Whether it changes anything:** say `read-only`, `creates a PR`, `updates a PR`, `merges`, or `deployment verification only`.
- **What evidence or links I expect to return:** PR URL, compare URL, check links, deployment links, logs, or runtime URL.

After every meaningful step, explain:

- **What happened:** success, no-op, blocker, waiting state, or failure.
- **What it means:** ready for the next step, no PR needed, needs review, checks failed, conflict, missing input, or safe to stop.
- **Useful links:** include every applicable URL discovered during the step.
- **What the user should do next:** review a PR, wait for checks, approve, ask a release owner, provide missing project input, or take no action.

Use short, consistent blocks such as:

```text
Next step: I am going to check whether GitHub has both release branches.
Why: This prevents creating a PR in the wrong direction.
Changes anything?: No, this is read-only.
Expected evidence: branch names and GitHub compare/PR links if available.
```

```text
Result: The promotion PR is ready for review.
What it means: GitHub now has a review page where people can inspect the change before it is merged.
Links: PR, Files changed, Checks, compare, deployment preview if available.
What you should do next: Open the PR link, read the Summary, review Files changed, and check whether required checks are green before approving or asking the release owner to continue.
```

This workflow is a production PR handoff, so explain production caution and that it stops before merge, deployment, and live smoke testing.

### Pull Request Handoff Instructions

Whenever you provide a PR URL, also explain what the user is supposed to do with it:

1. Open the PR URL.
2. Read the PR description sections: Summary, Scope, Diff evidence, Verification/Handoff status, and Unknowns.
3. Open **Files changed** to see what code or configuration changed.
4. Open **Checks** to see whether CI/build/deployment checks passed, failed, or are still running.
5. Read comments and reviewer threads for unresolved questions.
6. Do not delete the source or target branch after merge; ignore any GitHub branch-deletion prompt for these permanent promotion branches.
7. Follow the project policy for approval, merge, deployment, or escalation.

When handing off a production PR URL, tell the user or release owner to open the PR, read the summary, inspect Files changed, confirm Checks and approvals, and merge/deploy only through the project-approved production release process.

### Useful Link Inventory

Collect and report links whenever they are applicable and discoverable:

| Link Type | What to Provide | Why It Helps |
|-----------|-----------------|--------------|
| GitHub PR | PR URL plus PR number and title | Main review and handoff page |
| GitHub Files changed | PR files URL when known | Shows exactly what changed |
| GitHub Checks | Required/failing/pending check URLs | Shows build/test/review status |
| GitHub compare | Source-target compare URL | Shows branch delta before PR creation |
| GitHub Actions/CI | Workflow run URLs from check output | Lets users inspect failing jobs |
| Vercel deployment | Deployment URL and deployment detail/dashboard URL when available | Shows preview/build status and runtime entrypoint |
| Vercel project dashboard | Project dashboard URL when scope/project are known and verified | Lets users inspect deployments, logs, and settings |
| Vercel logs | Build/runtime log URL or exact command used when URL is unavailable | Explains failed or suspicious deployments |
| Runtime/health URL | Project-supplied smoke or health URL | Shows user-facing status |
| Provider dashboard | Render, Netlify, cloud, or internal dashboard links when relevant | Lets release owners continue investigation |

Do not invent links. If a useful link cannot be discovered, say exactly which link is missing and what input is needed to get it, such as Vercel scope, project name, deployment URL, or provider dashboard access.


## Common Misconceptions

| # | Misconception | Correction | Key concept |
|---|---------------|------------|-------------|
| 1 | Production promotion can use any recent branch. | Use the frozen staging candidate unless explicitly overridden by the project. | Candidate discipline |
| 2 | This skill merges and deploys production. | It only prepares and inspects PRs; merge/deploy are separate authority boundaries. | Handoff boundary |
| 3 | Direct production pushes are acceptable for conflicted PRs. | Conflict resolution requires explicit separate authorization. | No direct push |
| 4 | A missing PR is always an error. | Equal staging and production branches mean no PR is needed. | No-diff outcome |
| 5 | Metadata validation is optional. | Production handoff requires audit-friendly titles and body evidence when a metadata contract exists. | Metadata contract |
| 6 | A reusable skill can define production endpoints. | Production URLs, service IDs, and smoke tests belong to project docs. | Agnostic guidance |


## GitHub PR Review Guidance

This workflow is a PR handoff, so review quality is the main deliverable. For each PR:

1. **Conversation:** read the description, comments, bot/deployment assistant notes, and unresolved reviewer threads.
2. **Commit identity:** capture `headRefOid`, base branch, target branch, author, and draft state.
3. **Files changed:** inspect file names and risky patches; flag infrastructure, dependency, generated, lockfile, migration, secret, or environment changes.
4. **Checks and deployments:** inspect required checks, optional checks, and deployment statuses from the PR page or CLI.
5. **Reviews:** identify approvals, changes-requested reviews, stale approvals, and requested reviewers.
6. **Handoff boundary:** do not merge or deploy; report the review and readiness state.

Useful read-only commands:

```bash
gh pr view <number> -R "<owner>/<repo>" --comments \
  --json state,isDraft,author,title,body,baseRefName,headRefName,headRefOid,mergeable,mergeStateStatus,reviewDecision,reviewRequests,latestReviews,comments,reviews,statusCheckRollup,files,commits,url
gh pr diff <number> -R "<owner>/<repo>" --name-only
gh pr diff <number> -R "<owner>/<repo>" --patch
gh pr checks <number> -R "<owner>/<repo>" --watch=false --json name,state,bucket,link,workflow,startedAt,completedAt
gh pr view <number> -R "<owner>/<repo>" --web
```

For large diffs, inspect file names first, then targeted patches for risky files. If the PR page shows deployments or provider comments that the CLI summary does not expose, use the browser view and include those observations in the handoff report.

## Non-Negotiable Policy

1. Do not hardcode project repositories, domains, services, credentials, or production endpoints.
2. Promote production candidates from `staging` unless the user explicitly overrides the source branch.
3. Open or reuse PRs; do not push the production branch directly in this workflow.
4. Check existing PRs before creating new ones.
5. Infer PR titles and descriptions from verified production-handoff evidence before creation; use a project-provided metadata convention only when documented.
6. Explain each step before and after it runs, and include useful links plus user next actions in every handoff.
7. Do not merge PRs, run production deploy commands, or perform smoke tests in this workflow.
8. Report every repo whose PR is missing, blocked, conflicted, pending review, failing checks, or no-diff.

## Active PR Metadata Policy

Default title is inferred from the active lane:

```text
chore(release): promote <source-branch> into <target-branch>
```

For the standard branch defaults this becomes `chore(release): promote staging into production`. Before creating a PR, also infer a body that summarizes the lane, topology, release surfaces, source/target SHAs, changed-file risk, review/check expectations, downstream verification owner, and known gaps. Use a project-provided exact title/body only when project docs define one. After creating or reusing each PR, validate the actual title and description against the active metadata policy and report mismatches as blockers.

## Workflow

1. **Classify the request.** Confirm this is a production handoff from staging.
2. **Confirm project inputs.** Identify repository topology, repository set, release surfaces, branch names, metadata policy if documented, approval policy, root/meta repository policy, and downstream verification owner. Default to monorepo topology when only one repo is identified.
3. **Verify branch existence.** Ensure source and target branches exist remotely for every repo, including any root/meta repository that tracks release state. For monorepos, this is usually one repo; for submodules, verify root and required submodule repos according to policy.
4. **List existing PRs.** Reuse matching open PRs.
5. **Infer metadata and create missing PRs.** Generate the active title and description from lane, repo, source/target SHAs, changed paths, impacted release surfaces, handoff owner, and known gaps; create only when no matching PR exists and source has a diff.
6. **Validate metadata.** Compare every PR title and body to the active metadata policy.
7. **Inspect readiness.** Capture mergeability, reviews, comments, checks, branch-protection notes, head SHA, changed paths, and impacted release surfaces.
8. **Report handoff.** Provide per-repo and per-release-surface PR URLs, blockers, metadata results, no-diff outcomes, topology choice, and note that production merge/deploy are outside scope.

## Reference Loading by Task Type

| Task type | Load first | Skip |
|-----------|------------|------|
| PR creation/reuse | `references/staging-to-production-reference.md` | Deployment-provider docs |
| PR inspection | `references/staging-to-production-reference.md` plus GitHub PR page | Smoke-test docs |
| GitHub PR review | PR page plus `references/staging-to-production-reference.md` PR review section | Merge commands |
| Vercel preview analysis | `references/staging-to-production-reference.md` Vercel section plus project Vercel inputs | Vercel mutation/alias commands |
| Metadata validation | `references/staging-to-production-reference.md` metadata section | Merge commands |
| Production blocker report | `references/staging-to-production-reference.md` reporting section | Direct push guidance |


## Vercel Deployment Analysis When Relevant

Use this only when the project uses Vercel and the PR exposes Vercel checks, preview deployments, or deployment assistant comments. In this PR-handoff workflow, Vercel analysis is read-only evidence; it is **not** post-merge deployment verification.

1. **Identify linked deployments:** look for GitHub check links, deployment cards, PR comments, or project docs that name the Vercel deployment URL or ID.
2. **Inspect preview metadata:** confirm the Vercel deployment belongs to the expected project, branch, PR, and commit SHA when the CLI output exposes that data.
3. **Read build logs if checks fail:** use deployment logs to explain failed or pending checks.
4. **Read runtime logs only for handoff evidence:** runtime logs can help explain preview failures, but do not turn this into a smoke-test workflow unless the user asks for a separate verification step.
5. **Report Vercel links:** include the GitHub deployment/check link, Vercel deployment URL, Vercel deployment detail/dashboard URL when available, Vercel project dashboard URL when verified, and any log URL or exact log command used.
6. **Do not mutate aliases or protection:** route, alias, domain, and preview-protection changes are outside this PR-only skill unless the user explicitly asks for a separate deployment-provider workflow.

Reusable read-only command templates, verified against local Vercel CLI help during authoring:

```bash
vercel inspect <deployment-url-or-id> --format=json --scope <scope>
vercel inspect <deployment-url-or-id> --logs --scope <scope>
vercel logs --deployment <deployment-url-or-id> --json --since 30m --scope <scope>
vercel logs --project <project-name> --environment preview --branch <branch-name> --level error --since 1h --json --scope <scope>
```

Replace `<scope>`, `<project-name>`, deployment IDs, branch names, and environments with project inputs. If Vercel details are missing, report that GitHub indicates a Vercel surface but the project input is insufficient for deeper analysis.

### Monorepo Vercel Mapping

In monorepos, one GitHub PR can produce multiple Vercel deployments from the same commit. Build a mapping before declaring Vercel evidence complete:

| App/Package Path | Vercel Project | Deployment URL/ID | Environment | Branch | Commit/SHA Evidence | Status |
|------------------|----------------|-------------------|-------------|--------|---------------------|--------|
| `<apps/web>` | `<vercel-project>` | `<deployment>` | preview/production | `<branch>` | `<sha/source>` | ready/error/pending |

Use the PR changed-file list to decide which Vercel projects matter. If path filters skip a relevant project, report that as a verification gap rather than assuming it is safe.


## Quality Checklist

| # | Checklist Item | Why It Matters | Gate |
|---|---------------|---------------|------|
| 1 | Repository set identified | Full production release coverage | Pre-op |
| 2 | Default staging source verified remotely | Prevents wrong source branch | Pre-op |
| 3 | Default production target verified remotely | Prevents wrong destination | Pre-op |
| 4 | Existing PRs checked | Avoids duplicates | Draft |
| 5 | Plain-language step explanations and useful links provided | Keeps non-technical users oriented | Every step |
| 6 | Inferred PR metadata, diff risk, and PR review state validated | Audit, automation, and review safety | Draft |
| 7 | Merge/deploy not attempted | Authority boundary | Draft |
| 8 | Blockers documented | Actionable handoff | Closeout |
| 9 | No-diff outcomes reported | Completeness | Closeout |

### Quality Tiers

| Tier | Criteria | Use When |
|------|----------|----------|
| **Minimal** | Items 1-5 and 7 | Quick PR discovery or handoff planning |
| **Standard** | Items 1-8 | Standard promotion PR handoff |
| **Full** | All 9 items plus root/meta repo policy and post-merge owner captured | Complete release-candidate handoff |

### Pre-Op Verification

```text
□ Repository set is known and explicitly in scope.
□ Default source and target branch names are accepted or a documented override exists.
□ User-facing progress updates will be given before and after each meaningful step.
□ Existing PRs will be checked before creating new PRs.
□ PR title and body will be inferred from lane, diff, release-surface, and handoff evidence unless a documented convention overrides them.
□ Root/meta repository tracking policy is known.
□ Merge/deploy/smoke-test actions are confirmed out of scope.
```

## Consistency Validator

| Check | What to Verify | How to Fix |
|-------|----------------|------------|
| Source | PRs come from staging | Stop and recreate correct PRs |
| Target | PRs target production | Stop and recreate correct PRs |
| Scope | All requested repos inspected | Add missing repo inspections |
| Monorepo release surfaces | Every impacted app/package/deployment was reviewed | Add path/deployment mapping or report unknowns |
| PR metadata | Active title/body policy is satisfied with inferred or documented metadata | Report blocker or update/recreate if authorized |
| Boundary | No merge/deploy/smoke test run | Stop and report boundary issue |
| Root/meta tracking | Required root/meta PR is included or explicitly out of scope | Add inspection or report exclusion |

### Red Flags

- [ ] PR from main, development, or a feature branch.
- [ ] Direct push to production.
- [ ] Production PR merged by this skill.
- [ ] Production smoke test run without separate authorization.
- [ ] Project-specific production identifiers embedded in this skill.
- [ ] PR created with an empty or generic body instead of inferred handoff evidence.
- [ ] Monorepo treated as “done” without app/package/deployment surface mapping.
- [ ] Root repo PR assumed to update submodule repos without policy confirmation.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| PR creation reports no diff | Staging and production are equal | Report no PR needed |
| PR metadata mismatch | Existing/manual PR uses a different title or insufficient body | Report blocker or update/recreate if authorized |
| PR is conflicting | Candidate and production diverged | Report blocker; do not push production directly |
| Checks fail | CI/release gate failure | List failing checks and URLs |
| Review pending or changes requested | Human approval incomplete | Report review blocker |
| Branch missing | Production branch not initialized | Stop and ask for branch policy |
| Wrong-metadata PR needs replacement | Existing PR does not satisfy the active title/body policy | Edit, close, or recreate only when the user authorizes that mutation. |

## Post-Merge Scope Note

Production deployment verification applies **after** these PRs are merged and is intentionally outside this skill. When post-merge verification is requested, hand off to a deployment-provider, browser-validation, smoke-test, or project-specific release skill with the merged SHA and PR list.

## Cross-Skill Coordination

- Use a GitHub PR-review or browser skill when authenticated web UI review is required for conversation threads, deployment cards, or files-changed navigation.
- Use a browser-validation or smoke-test skill only after merge when production runtime evidence is requested.
- Use a deployment-provider-specific skill only after merge or when PR checks require provider log inspection.
- Use a study or decision skill when production release scope, root/meta tracking, or branch policy is unclear.
- Use a monorepo/workspace-aware build or package-manager skill when path ownership, workspace graphs, or package-level tests determine release scope.
- Use a git/submodule skill when the repository set includes nested repositories, submodules, or root/meta tracking branches.

## Guidance Alignment

- Keep this skill project-agnostic; project-specific production surfaces belong in project docs or operator-provided inputs.
- After editing this skill in a skills host repo, run that repo's skill sync and skill validation commands.
- Do not add real repository names, service IDs, domains, credentials, or provider resource IDs to this reusable skill.

## Common Pitfalls

1. Using `development` or `main` as source for production handoff.
2. Creating duplicate PRs instead of reusing existing staging-to-production PRs.
3. Creating PRs with empty descriptions or titles not inferred from the active lane and diff evidence.
4. Merging or deploying production inside a PR-handoff workflow.
5. Treating no-diff repos as errors.
6. Reporting production handoff without per-repo blockers and check state.
7. Baking production endpoints into a reusable skill instead of project docs.
8. Forgetting a required root/meta repository tracking PR.
9. Treating a monorepo as a single undifferentiated deploy surface instead of mapping apps/packages/deployments.

## Temporary Files

If this skill needs temporary files, place them under `.tmp/promote-staging-to-production/YYYY-MM-DD-{subject}` unless the host project gives a stricter temp-file convention. Do not create top-level dotfile temp directories.

## Local Corpus Layout

The `references/` directory contains one hand-authored operational file:

| File | Purpose |
|------|---------|
| `staging-to-production-reference.md` | Generic command templates for branch discovery, PR create/reuse, metadata validation, PR inspection, error handling, and production handoff reporting. |
