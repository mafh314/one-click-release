---
description: Identify Jiras missing fixVersion or release note fields for a release
---

# Jira Gaps

<objective>
Given a release audit result JSON (produced by `go run cmd/snapshot-diff/main.go` or `cmd/image-diff/main.go`), identify Jira tickets linked to commits that are missing `fixVersion` and/or release note fields in Jira.

Invoked as: `/jira-gaps @reports/release_X.Y.Z.json`

The skill extracts all unique Jira keys from the JSON, queries each via the Jira REST API, and produces a gap report showing which tickets need fixVersion or release note updates.

**External systems** (credentials in `.env`):
- Jira: `JIRA_URL` + `JIRA_EMAIL` + `JIRA_TOKEN` — **READ-ONLY**
- Release note custom fields: `JIRA_RN_TEXT_FIELD`, `JIRA_RN_TYPE_FIELD`, `JIRA_RN_STATUS_FIELD`

**Filtering rules:**
- Skip all `dependabot[bot]` commits when extracting Jira keys.
- Exclude Jiras with "CVE" (case-insensitive) anywhere in their summary.
- Exclude Jiras where both fixVersion and release note fields are already set.
- Exclude inaccessible Jiras (HTTP 404).
</objective>

<process>
<step name="setup">
**Source environment and read the JSON.**

```bash
source .env
```

Verify `JIRA_URL`, `JIRA_EMAIL`, `JIRA_TOKEN`, `JIRA_RN_TEXT_FIELD`, `JIRA_RN_TYPE_FIELD` are set.

Read the release JSON. The user provides the path as an argument, or default to the most recent `reports/release_*.json`. Extract the `version` field (e.g. `1.21.3`). The expected fixVersion string is `Pipelines {version}`.
</step>

<step name="extract_and_query">
**Extract Jira keys and query Jira for gaps.**

Run a single Python script that:

1. **Extracts** all unique Jira keys from the JSON:
   - Iterate all components and their commits.
   - Skip commits where `author == "dependabot[bot]"`.
   - For each commit with a `jira` field, extract the Jira key (last path segment of the URL or the raw key).
   - Track per-key: component name, upstream repo, `jira_source`, and the list of commits (sha, message, author, pr, original_pr).

2. **Queries** each unique Jira key via REST API:
   - Endpoint: `{JIRA_URL}/rest/api/2/issue/{key}`
   - Fields: `key,summary,status,fixVersions,{JIRA_RN_TEXT_FIELD},{JIRA_RN_TYPE_FIELD},{JIRA_RN_STATUS_FIELD}`
   - Auth: HTTP Basic with `JIRA_EMAIL:JIRA_TOKEN`
   - For each ticket, determine:
     - `has_fix_version`: whether any fixVersion contains the expected version string (substring match)
     - `has_rn`: whether both RN type and RN text fields are non-empty
   - Handle 404 responses by marking the ticket as inaccessible.

3. **Filters** the results:
   - Remove tickets with "CVE" (case-insensitive) in the summary.
   - Remove tickets where both `has_fix_version` and `has_rn` are true.
   - Remove inaccessible tickets (404).

4. **Determines "Linked via"** for each ticket based on `jira_source`:
   - Empty or `"pre-existing"` → `Go tool (Jira custom field)`
   - `"pr-body"` → `PR body`
   - `"keyword"` → `Keyword search`
   - `"unmatched-jira"` → `Unmatched Jira cross-match`

5. **Outputs** a JSON file at `reports/jira-gaps-{version}-data.json` with the query results for the report step.

Print a summary to the conversation: how many Jiras extracted, how many filtered out (CVE, fully resolved, inaccessible), how many remain with gaps.
</step>

<step name="generate_report">
**Generate the markdown report.**

Read the data JSON from the previous step. Generate the report:

```markdown
# Jira Gaps: Release {version}

Jiras linked to commits in {version} that are missing `fixVersion: Pipelines {version}` and/or release note fields.
Excludes Jiras with "CVE" in their summary and Jiras with all fields already set.

## Overview

| Jira | Component | Status | fixVersion {version} | RN Fields | Linked via | Commits | PRs |
|------|-----------|--------|----------------------|-----------|------------|---------|-----|
| [KEY](url) | component | Status | YES/**MISSING** | YES/**MISSING** | source | N | N |

---

## [KEY](url)

**Summary:** ...<br>
**Component:** component (upstream)<br>
**Status:** ...<br>
**Current fixVersions:** ...<br>
**RN Type:** Type · **RN Status:** Status<br>  ← only if RN fields are set
**Linked via:** description of how the Jira was discovered<br>
**Missing:** list of missing fields

| SHA | Message | Author | PR |
|-----|---------|--------|----|
| short_sha | message | author | PR link(s) |
```

**Column rules:**
- **SHA**: first 7 characters
- **PR**: `[#orig](url) → [#cherry](url)` when both exist and differ; just `[#N](url)` otherwise; `—` when missing.
- **fixVersion**: `YES` if the expected version is present, `**MISSING**` otherwise.
- **RN Fields**: `YES` if both type and text are set, `**MISSING**` otherwise.
- **Linked via**: human-readable description of the `jira_source`.
- **Missing**: comma-separated list of what's missing (fixVersion, Release Note fields).

**RN detail line**: If RN fields are set (type and/or text exist), show `**RN Type:** {type} · **RN Status:** {status}` in the detail section. Omit this line if both are empty.

Write the report to `reports/jira-gaps-{version}.md`.

Clean up the intermediate data JSON (`reports/jira-gaps-{version}-data.json`).
</step>
</process>

<success_criteria>
- [ ] All unique Jira keys extracted from the release JSON (excluding dependabot commits)
- [ ] Each Jira queried via REST API using the correct custom field IDs from `.env`
- [ ] CVE-named Jiras excluded from the report
- [ ] Fully-resolved Jiras (both fixVersion and RN set) excluded
- [ ] "Linked via" column accurately reflects `jira_source` provenance
- [ ] Report written to `reports/jira-gaps-{version}.md`
</success_criteria>
