# Diff Snapshots

Extract components from two Konflux snapshots and identify which repos changed.

**Inputs:** `SNAP_OLD`, `SNAP_NEW`, `KONFLUX_NS` (`tekton-ecosystem-tenant`), `KONFLUX_SERVER`, `KONFLUX_TOKEN`

**Constraints:**
- Konflux cluster: **READ-ONLY** (`oc get` only)

## Fetch snapshot components

For each snapshot, extract the full component list:

```bash
oc get snapshot ${SNAPSHOT_NAME} -n tekton-ecosystem-tenant \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify -o jsonpath='{.spec.components}' 2>/dev/null
```

Parse with Python to group by repo:

```bash
echo '$COMPONENTS_JSON' | python3 -c "
import sys, json
components = json.load(sys.stdin)
repos = {}
for c in components:
    src = c.get('source', {}).get('git', {})
    url = src.get('url', '').rstrip('.git').replace('https://github.com/', '')
    rev = src.get('revision', '?')
    name = c['name']
    if url not in repos:
        repos[url] = []
    repos[url].append({'name': name, 'revision': rev})
for repo in sorted(repos):
    for comp in repos[repo]:
        print(f'{repo}\t{comp[\"revision\"]}\t{comp[\"name\"]}')
"
```

Do this for BOTH snapshots.

## Compare repos

For each repo present in either snapshot:

1. **Collect all revisions** from old snapshot and new snapshot
2. **Compare:**
   - If old and new have identical sets of (component_name -> revision) mappings: **UNCHANGED**
   - If any component's revision differs between old and new: **CHANGED**
   - If repo only in new: **ADDED**
   - If repo only in old: **REMOVED**

For CHANGED repos, determine the representative old and new SHAs:
- If all components in a repo share the same revision in a snapshot, use that single SHA
- If components have different revisions (split build), report each unique old->new SHA pair

## Output

Report:
- List of changed repos: `{repo, old_sha, new_sha}` (one entry per unique SHA transition)
- List of unchanged repos (names only)
- Any added or removed repos

**Return:** The changed repos list and unchanged repos list for use by the next step.
