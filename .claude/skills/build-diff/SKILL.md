---
description: Compare two Konflux snapshots to show what changed between builds
---

# Build Diff

<objective>
Given two Konflux snapshot names (old and new) for the same release version, compare the component SHAs to identify which repos changed, then resolve what changed **upstream** — the actual code changes that went into the build — with linked Jiras.

Downstream repos are mirrors that track upstream repos (e.g. `openshift-pipelines/tektoncd-pipeline` tracks `tektoncd/pipeline`). Each downstream repo has a `head` file at its root containing the upstream commit SHA it was synced to. By reading the `head` file at both the old and new downstream SHAs, we get the upstream commit range and can list the actual upstream changes.

Invoked as: `/build-diff <snapshot-old> <snapshot-new>`

Example: `/build-diff openshift-pipelines-core-1-22-20260630-071148-000 openshift-pipelines-core-1-22-20260701-071148-000-ry`

**External systems** (credentials in `.env`):
- GitHub: `gh` CLI (read-only for PRs and commits)
- Konflux cluster: `KONFLUX_SERVER` + `KONFLUX_TOKEN` — **READ-ONLY** (`oc get` only)

**Konflux namespace:** All operations target `tekton-ecosystem-tenant`.

**Jira linking rule:** Only Jiras explicitly mentioned in the PR title or body count. No keyword search, no JQL fallback, no inference. If no Jira key is found in the PR, the commit has no linked Jira.
</objective>

<process>
<step name="setup">
**Parse arguments and initialize.**

Parse two snapshot names from the skill argument. Both are required.

```bash
source .env
```

Verify `KONFLUX_SERVER` and `KONFLUX_TOKEN` are set.

**Time formatting:**
```bash
TZ_FMT='%Y-%m-%d %H:%M %Z'
```

All timestamps in the report MUST use absolute local time. **Never** use relative time expressions.

Derive the version from the snapshot name. Core snapshots follow the pattern:
`openshift-pipelines-{type}-{MM_DASHED}-{timestamp}-{suffix}`

Extract `MM_DASHED` (e.g. `1-22`) and convert to `MAJOR_MINOR` (e.g. `1.22`).
</step>

<step name="diff_snapshots">
**Compare component SHAs between snapshots.**

Read and follow the instructions in `DIFF_SNAPSHOTS.md`.

Pass `SNAP_OLD`, `SNAP_NEW`, `KONFLUX_NS`, `KONFLUX_SERVER`, `KONFLUX_TOKEN` to the subskill context.

This produces a list of changed repos with `{repo, old_sha, new_sha}` and a list of unchanged repos.
</step>

<step name="resolve_upstream">
**For each changed repo, resolve upstream commits and Jiras.**

Read and follow the instructions in `RESOLVE_COMMITS.md`.

For each changed repo from the previous step, pass `{downstream_repo, old_sha, new_sha}` and `TZ_FMT`.

The subskill will:
1. Look up the upstream repo name from the hack release config (`config/downstream/repos/{component}.yaml`)
2. Read the `head` file at both old and new downstream SHAs to get upstream commit SHAs
3. Compare upstream commits between those SHAs on the upstream repo
4. Resolve PRs and extract Jiras (strict — PR title/body only)

Also capture any downstream-only commits (non-bot, non-nudge) that represent direct downstream changes.
</step>

<step name="report">
**Generate the diff report.**

Compile results and write to `reports/build-diff-{MAJOR_MINOR}-{YYYYMMDD-HHMMSS}.md`.

```markdown
# Build Diff: {MAJOR_MINOR}

**Old snapshot:** {SNAP_OLD}
**New snapshot:** {SNAP_NEW}
**Generated:** {absolute local time with timezone}

## Summary

| Upstream Repo | Upstream Branch | Old Head | New Head | Upstream Commits | Jiras |
|---------------|-----------------|----------|----------|------------------|-------|
| tektoncd/operator | release-v0.80.x | 5c9be32... | daffb82... | 4 | 1 |

---

## tektoncd/operator (N upstream commits)

**Downstream:** openshift-pipelines/operator
**Upstream branch:** release-v0.80.x
**Upstream head:** 5c9be32 → daffb82

| SHA | Message | Author | Date | PR | Jira |
|-----|---------|--------|------|----|------|
| 6fa49fe | fix(manualapprovalgate): inject TLS env vars | Jawed khelil | 2026-06-26 | [#3584](url) | [SRVKP-12613](url) |
| 5025382 | refactor: simplify update-image-sha.sh | Shiv Verma | 2026-06-21 | [#3585](url) | — |

### Downstream-only changes

Non-bot commits on the downstream repo that are not upstream syncs:

| SHA | Message | Author | Date | PR | Jira |
|-----|---------|--------|------|----|------|
| 9c834d2 | Cleanup 4.17 | Khurram Baig | 2026-06-26 | [#24294](url) | — |

---

## Unchanged Components (N)

repo-a, repo-b, repo-c, ...
```

**Column rules:**
- **SHA**: first 7 characters
- **PR**: `[#N](url)` when found, `—` when missing
- **Jira**: `[KEY](browse-url)` when found in PR title/body, `—` otherwise. Never guess.
- **Date**: absolute local time, date only (YYYY-MM-DD)
- **Skip dependabot commits** in the upstream table — count them in a summary line instead

Print the summary to the conversation after writing the report.
</step>
</process>

<success_criteria>
- [ ] Both snapshots fetched from Konflux (READ-ONLY)
- [ ] Changed repos correctly identified by comparing component revisions
- [ ] Upstream repo and branch resolved from hack release config
- [ ] Upstream head SHAs extracted from downstream `head` file at old and new revisions
- [ ] Upstream commits listed via GitHub compare API on the upstream repo
- [ ] PRs resolved for each upstream commit
- [ ] Jiras found via Jira's "Git Pull Request" custom field (`cf[10875]`) — no keyword search, no guessing
- [ ] Downstream-only (non-bot, non-nudge) commits identified separately
- [ ] dependabot commits counted but not listed individually
- [ ] All timestamps use absolute local time with timezone
- [ ] Report written to `reports/build-diff-{version}-{timestamp}.md`
</success_criteria>
