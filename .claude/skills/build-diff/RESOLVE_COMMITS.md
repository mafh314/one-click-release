# Resolve Upstream Commits and Jiras

For each changed downstream repo, find the upstream repo, resolve the upstream commit range, list commits, and find linked Jiras.

**Inputs:** List of `{downstream_repo, old_sha, new_sha}` from DIFF_SNAPSHOTS, `TZ_FMT`, `MAJOR_MINOR`

**Jira linking rule:** STRICT. Query Jira's "Git Pull Request" custom field (`cf[10875]`) for the PR URL. Only link a Jira if it explicitly has the PR URL in that field. No keyword search, no body parsing, no inference.

## For each changed downstream repo

### 1. Identify upstream repo and branch

Look up the component name and upstream repo from the hack release config:
```bash
# Component name is typically the downstream repo name without org prefix
# e.g. openshift-pipelines/tektoncd-pipeline -> tektoncd-pipeline
COMPONENT=$(echo "${DOWNSTREAM_REPO}" | sed 's|openshift-pipelines/||')
REPO_CFG=$(gh api repos/openshift-pipelines/hack/contents/config/downstream/repos/${COMPONENT}.yaml \
  --jq '.content' | base64 -d)
UPSTREAM_REPO=$(echo "$REPO_CFG" | grep '^upstream:' | awk '{print $2}')
```

Get the upstream branch from the release config:
```bash
RELEASE_CFG=$(gh api repos/openshift-pipelines/hack/contents/config/downstream/releases/${MAJOR_MINOR}.yaml \
  --jq '.content' | base64 -d)
UPSTREAM_BRANCH=$(echo "$RELEASE_CFG" | python3 -c "
import sys, yaml
cfg = yaml.safe_load(sys.stdin)
print(cfg['branches'].get('${COMPONENT}', {}).get('upstream', ''))
")
```

Cache the release config — fetch it once, reuse for all repos.

### 2. Read upstream head SHAs from the `head` file

Each downstream repo has a `head` file at its root containing the upstream commit SHA it tracks:

```bash
OLD_UPSTREAM_SHA=$(gh api repos/${DOWNSTREAM_REPO}/contents/head?ref=${OLD_SHA} \
  --jq '.content' | base64 -d | tr -d '[:space:]')
NEW_UPSTREAM_SHA=$(gh api repos/${DOWNSTREAM_REPO}/contents/head?ref=${NEW_SHA} \
  --jq '.content' | base64 -d | tr -d '[:space:]')
```

If both SHAs are identical, there are no upstream changes — note "downstream-only changes" and skip to step 5.

### 3. Get upstream commits between SHAs

```bash
gh api repos/${UPSTREAM_REPO}/compare/${OLD_UPSTREAM_SHA}...${NEW_UPSTREAM_SHA} \
  --jq '.commits[] | "\(.sha)\t\(.commit.message | split("\n") | .[0])\t\(.commit.author.name)\t\(.commit.author.date)"'
```

If the compare returns an error, note it and skip to the next repo.

Filter out `dependabot[bot]` commits — count them but don't list them individually. Only show human-authored commits in the table.

### 4. Resolve PRs and Jiras for upstream commits

For each non-dependabot upstream commit:

**Resolve PR:**
```bash
gh api repos/${UPSTREAM_REPO}/commits/${SHA}/pulls --jq '.[0] | {number: .number, url: .html_url}' 2>/dev/null
```

**Find Jira via the "Git Pull Request" custom field (`cf[10875]`):**

Jira tickets have a "Git Pull Request" custom field that stores the PR URL. Search it:

```bash
PR_URL="https://github.com/${UPSTREAM_REPO}/pull/${PR_NUMBER}"
curl -s -u "$JIRA_EMAIL:$JIRA_TOKEN" \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -X POST "$JIRA_URL/rest/api/3/search/jql" \
  -d "{\"jql\":\"cf[10875] ~ \\\"${PR_URL}\\\"\",\"fields\":[\"key\",\"summary\",\"status\"],\"maxResults\":5}" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for issue in data.get('issues', []):
    print(f\"{issue['key']}\t{issue['fields']['summary']}\")
" 2>/dev/null
```

**Cherry-pick resolution for Jira:** On release branches, commits are often cherry-picks of main-branch PRs. The Jira ticket typically links to the **original PR on main**, not the cherry-pick PR on the release branch. If no Jira found for the release-branch PR:

1. Get the full commit message:
   ```bash
   gh api repos/${UPSTREAM_REPO}/commits/${SHA} --jq '.commit.message'
   ```
2. Check for cherry-pick trailer: `(cherry picked from commit <SHA>)`
3. If found, look up the original PR for that SHA:
   ```bash
   gh api repos/${UPSTREAM_REPO}/commits/${ORIGINAL_SHA}/pulls --jq '.[0] | {number: .number, url: .html_url}' 2>/dev/null
   ```
4. Search Jira `cf[10875]` for the **original PR URL**

Cache PR and Jira lookups — never fetch the same PR or query the same PR URL twice.

If no Jira found via either the release-branch PR or the original PR, leave Jira empty (`—` in the report).

### 5. Identify downstream-only changes

Get the downstream commits between the old and new SHAs:
```bash
gh api repos/${DOWNSTREAM_REPO}/compare/${OLD_SHA}...${NEW_SHA} \
  --jq '.commits[] | "\(.sha)\t\(.commit.message | split("\n") | .[0])\t\(.commit.author.name)\t\(.commit.author.date)"'
```

Filter to only **non-bot, non-nudge** commits:
- Exclude author `red-hat-konflux-kflux-prd-rh02[bot]`
- Exclude author `openshift-pipelines-bot` (these are upstream syncs)
- Exclude author `dependabot[bot]`
- Exclude messages matching `chore(deps): update *`, `chore: auto-generate OCP catalog JSONs`, `[bot:*] Update generate CSV`

The remaining commits are downstream-only changes (e.g. OCP-specific fixes, config changes). Resolve PRs and Jiras for these using the same approach (PR lookup on the downstream repo, then Jira search via `cf[10875]`).

### 6. Format timestamps

Convert all commit dates to absolute local time:
```bash
date -d "${COMMIT_DATE}" +"${TZ_FMT}"
```

For the report table, use date-only format: `YYYY-MM-DD`.

## Output

For each repo, return:
- Upstream repo and branch
- Old and new upstream head SHAs
- Upstream commits (non-dependabot): `{sha, message, author, date, pr_url, jiras[]}`
- Dependabot commit count (summary only)
- Downstream-only commits (if any): same format
- Total Jira count

**Return:** Per-repo upstream commit data for the report generator.
