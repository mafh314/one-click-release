# Process Single Component for Release Audit

You are processing ONE component from a release audit JSON. Your job:
1. Find Jira tickets for unlinked commits (Tier 2: PR body, Tier 3: keyword JQL)
2. Generate release notes for commits missing them
3. Write all results back to the JSON file after each step

The component name, component index, and JSON file path are provided in your task prompt.

## Setup

```bash
source .env
```

Read the JSON file. Extract the component at the given index. **Verify the component name matches** — if it doesn't, abort with an error (wrong index).

If the component has an `error` field set or no `commits` (and no `unsynced_commits`), skip to **Finalize**.

Process both `commits` and `unsynced_commits` arrays through all steps below. They go through the same Jira search and release note pipeline — the only difference is how they are displayed in the report (separate tables).

Count (across both arrays):
- Commits where `author != "dependabot[bot]"` AND `jira` is empty AND `jira_source` is empty -> need Jira search
- Commits where `author != "dependabot[bot]"` AND `jira` is non-empty but `release_note` is null/missing AND `rn_source` is empty -> need release notes

If no commits need work, skip to **Finalize**.

## Tier 2: PR Body Jira Search

For each commit needing Jira search (empty `jira`, empty `jira_source`, author != `dependabot[bot]`):

1. Prefer `original_pr` URL when available (upstream PRs more likely to contain Jira keys). Only fall back to `pr` if `original_pr` yields no keys.
2. Fetch PR data (cache results by URL within this component -- never fetch the same PR twice):
   ```bash
   gh pr view {number} --repo {org/repo} --json title,body,labels
   ```
3. Extract Jira keys from title + body:
   ```bash
   echo "$body $title" | grep -oE '[A-Z]{2,}-[0-9]+' | sort -u
   ```
4. If a key is found, set the commit's `jira` field to `https://redhat.atlassian.net/browse/{KEY}` and `jira_source` to `"pr-body"`.

After processing all commits in this step, **write the updated JSON** (see Write Pattern below).

## Tier 3: Keyword JQL Search

For commits still without Jira after Tier 2 (empty `jira`, empty `jira_source`, `author != "dependabot[bot]"`):

1. Extract search keywords from the commit message:
   - Strip conventional commit prefixes (`fix:`, `feat:`, `build(deps):`, etc.)
   - Strip `Merge pull request #NNN from ...` -- use cached PR title instead
   - For CVE/GHSA IDs, search by that ID directly
   - Take the first meaningful phrase (up to ~5 words)

2. Search Jira via JQL. Batch where possible using `OR` clauses:
   ```bash
   curl -s -u "$JIRA_EMAIL:$JIRA_TOKEN" \
     -H "Content-Type: application/json" -H "Accept: application/json" \
     -X POST "$JIRA_URL/rest/api/3/search/jql" \
     -d '{"jql":"project = SRVKP AND summary ~ \"keywords\"","fields":["key","summary","status"],"maxResults":5}'
   ```

3. Score results:
   - **medium**: keyword match + ticket is open/in-progress + summary clearly related
   - **low**: weak signal (generic terms, old/closed ticket)

4. For CVE commits with no keyword match, also try: `"project = SRVKP AND text ~ \"CVE-2026-XXXX\""`

5. If found, set `jira` to the browse URL and `jira_source` to `"keyword"`.

After processing, **write the updated JSON**.

## Generate Release Notes

Process commits where `author != "dependabot[bot]"` AND `jira` is non-empty AND (`release_note` is null/missing OR `rn_source` is empty).

**Step A -- Extract from PR description:**
Look for a "Release Notes" section in the PR body (normalize `\r\n` to `\n`):
- Code blocks: ` ```release-note ``` `
- Markdown headings: `## Release Notes`
- Labeled text: `Release Notes:`

Extract content, strip fences. If text is "NONE", treat as absent.
If found, set `release_note` to `{"text": "...", "type": "...", "status": ""}` and `rn_source` to `"extracted"`.

**Step B -- Generate using AI:**
If not extractable, gather context:
- Commit message + cached PR title/description
- Fetch `gh pr diff {number} --repo {org/repo} | head -200` only if commit message + PR body are too terse

Generate a 1-3 sentence release note: present tense, user-facing perspective, no PR numbers or internal references. Set `rn_source` to `"generated"`.

**Step C -- Classify type:**
Skip if `release_note.type` is already set and non-empty. Otherwise classify:
- PR labels: `kind/bug` -> Bug Fix, `kind/feature` -> Feature, `kind/enhancement` -> Enhancement
- CVE/GHSA IDs -> CVE
- `release-note-none` label or "NONE" text -> Release Notes Not Required
- Dependency bumps for CVE fixes -> CVE
- Internal chores, CI, image SHA updates -> Release Notes Not Required
- Default when uncertain -> Unspecified

**Type enum:** Bug Fix, CVE, Deprecated Functionality, Developer Preview, Enhancement, Feature, Known Issue, Removed Functionality, Tech Preview, Release Notes Not Required, Unspecified

After processing, **write the updated JSON**.

## Finalize

Add the component name to `skill_progress.processed_components`. Initialize `skill_progress` if missing:
```json
{"processed_components": [], "report_generated": false}
```

Write the final JSON.

## JSON Write Pattern

Use Python for all JSON updates (robust for nested structures):

```bash
python3 << 'PYEOF'
import json, os, shutil

JSON_FILE = "REPLACE_WITH_PATH"
COMP_IDX = REPLACE_WITH_INDEX
COMP_NAME = "REPLACE_WITH_NAME"

with open(JSON_FILE) as f:
    data = json.load(f)

comp = data['components'][COMP_IDX]
assert comp['name'] == COMP_NAME, f"Index mismatch: expected {COMP_NAME}, got {comp['name']}"

# Example: update a commit's jira field
# comp['commits'][j]['jira'] = 'https://redhat.atlassian.net/browse/SRVKP-1234'
# comp['commits'][j]['jira_source'] = 'pr-body'

# Example: update release_note
# comp['commits'][j]['release_note'] = {'text': '...', 'type': 'Bug Fix', 'status': ''}
# comp['commits'][j]['rn_source'] = 'generated'

# Initialize skill_progress if missing
if 'skill_progress' not in data or data['skill_progress'] is None:
    data['skill_progress'] = {'processed_components': [], 'report_generated': False}

# Mark component as processed
if COMP_NAME not in data['skill_progress']['processed_components']:
    data['skill_progress']['processed_components'].append(COMP_NAME)

# Atomic write: temp file + rename
tmp = JSON_FILE + '.tmp'
with open(tmp, 'w') as f:
    json.dump(data, f, indent=2)
    f.write('\n')
shutil.move(tmp, JSON_FILE)
PYEOF
```

Adapt the template above for each step. You can combine multiple commit updates in a single Python script run.

## Return

Report a summary:
- Commits linked to Jira: N (N via PR body, N via keyword)
- Release notes: N extracted, N generated
- Commits still unlinked: N
- Any errors encountered
