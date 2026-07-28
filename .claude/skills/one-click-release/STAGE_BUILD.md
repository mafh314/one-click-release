# Stage 2: Build

Process release PRs, verify Konflux snapshots are current, merge nudge PRs, render OLM catalog, verify FBC builds, and set code freeze. Steps run sequentially — stop at the first step that requires action (unless the user approves execution).

**Pipeline source:** `one-click-release/pipelines/openshift-pipelines-release.yaml` — tasks `process-pull-requests`, `wait-for-core-snapshot`, `process-nudge-prs`, `wait-for-fbc-snapshot`. Also covers OLM catalog rendering and FBC index build verification from `one-click-release/README.md` Build Stage.

**Inputs:** `VERSION`, `MAJOR_MINOR`, `MM_DASHED`, `RELEASE_BRANCH`, `KONFLUX_NS`, `KONFLUX_SERVER`, `KONFLUX_TOKEN`, `TZ_FMT`, `REPORT_BASE`, `REPORT_TIMESTAMP`

**Constraints:**
- Konflux cluster: **READ-ONLY** for verify commands (`oc get`/`kubectl get` only). Release CRs may be created (`oc create -f`) as execute commands after user approval.
- Execute commands require explicit user approval before running
- **Stage release ordering (hard requirement):** core → bundle → index. Never create a bundle stage release before the core stage release succeeds. Never create index stage releases before the bundle stage release succeeds.

**Formatting:**
- PR links: `[#NUM](https://github.com/OWNER/REPO/pull/NUM)`
- SHA links: `[SHORT](https://github.com/OWNER/REPO/commit/FULL)` (12-char short)
- Timestamps: absolute local time with timezone (e.g. `2026-07-08 14:30 IST`)

---

## Step 2.1: Process release PRs

Mirrors pipeline task `osp-github-pr-processor` with label `hack`. The pipeline searches for open PRs with release-related labels across all `openshift-pipelines` repos on the release branch, checks their CI status, and merges them when green.

### Verify

Search for open release PRs on the release branch:
```bash
gh search prs --owner openshift-pipelines \
  --base "${RELEASE_BRANCH}" \
  --state open \
  --json repository,url,title,labels \
  "label:hack,upstream,automated" 2>/dev/null
```

For each open PR, check its mergeable and CI status:
```bash
gh pr view "${PR_URL}" --json mergeable,mergeStateStatus,statusCheckRollup
```

Classify each PR:
- `mergeable: CONFLICTING` or `mergeStateStatus: DIRTY` → **CONFLICT** (cannot auto-merge)
- `mergeStateStatus: BEHIND` → **BEHIND** (needs rebase)
- CI checks with `bucket: fail` → **CI FAILING**
- CI checks with `bucket: pending` → **CI PENDING**
- All checks pass and mergeable → **READY TO MERGE**

**Collect links:** Report each PR as `repo [#NUM](URL)` with its status.

**Expected when DONE:** No open PRs found — all release PRs have been merged.

### If not done — Execute (requires approval)

For each PR that is **READY TO MERGE**:
```bash
gh pr edit "${PR_URL}" --add-label "lgtm,approved,one-click-release"
gh pr review --approve "${PR_URL}"
gh pr merge "${PR_URL}" -d -r --auto
```

For PRs that are **BEHIND**:
```bash
gh pr update-branch "${PR_URL}" --rebase
```

For PRs with **CI FAILING**: report the failing checks and skip. The user must investigate manually.

For PRs with **CI PENDING**: report that checks are still running. Re-check after a few minutes.

After merging, re-verify to confirm no open PRs remain.

---

## Step 2.2: Wait for core snapshot

Mirrors pipeline task `wait-for-snapshot` (APP=core) and script `wait-for-latest-snapshot.sh`. Verifies the latest Konflux core snapshot has the correct commit SHA for each component repo.

**Requires:** `KONFLUX_SERVER` and `KONFLUX_TOKEN`. If missing, SKIP this step.

### Verify

Get the latest core snapshot:
```bash
CORE_APP=$(oc get applications.appstudio.redhat.com -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify -o json 2>/dev/null \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
mm = '${MM_DASHED}'
apps = [i['metadata']['name'] for i in data['items'] if 'core' in i['metadata']['name'] and mm in i['metadata']['name']]
print(apps[0] if apps else '')
")

LATEST_SNAPSHOT=$(oc get snapshots -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -l "pac.test.appstudio.openshift.io/event-type=push,appstudio.openshift.io/application=${CORE_APP}" \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[*].metadata.name}' 2>/dev/null | awk '{print $NF}')

echo "Latest core snapshot: ${LATEST_SNAPSHOT}"
```

Extract components and compare against release branch HEAD:
```bash
oc get snapshot ${LATEST_SNAPSHOT} -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -o jsonpath='{.spec.components}' 2>/dev/null | python3 -c "
import sys, json, subprocess

components = json.load(sys.stdin)
repos = {}
for c in components:
    src = c.get('source', {}).get('git', {})
    url = src.get('url', '').rstrip('.git')
    rev = src.get('revision', '?')
    repo_path = url.replace('https://github.com/', '')
    repo_name = repo_path.split('/')[-1]

    # Skip operator for core snapshot (has nudge commits ahead)
    if repo_name == 'operator':
        continue

    if repo_path not in repos:
        repos[repo_path] = {'revision': rev, 'components': []}
    repos[repo_path]['components'].append(c['name'])

for repo_path, info in sorted(repos.items()):
    print(f'{repo_path} {info[\"revision\"][:12]} ({len(info[\"components\"])} components)')
"
```

For each unique repo in the snapshot, compare against HEAD:
```bash
REMOTE_SHA=$(git ls-remote "https://github.com/${REPO_PATH}.git" "refs/heads/${RELEASE_BRANCH}" | awk '{print $1}')
```

Classify:
- Snapshot revision == HEAD → **CURRENT**
- Snapshot revision != HEAD → **STALE** (needs rebuild)
- Same repo has multiple revisions → **SPLIT** (partial rebuild)

Report a table:
```
| Repo | Snapshot SHA | HEAD SHA | Status |
|------|-------------|----------|--------|
```

**Collect links:** Report the snapshot name and creation time. For stale repos, note which need rebuilds.

**Expected when DONE:** All non-operator repos are CURRENT.

### If not done — Execute (requires approval)

For each stale repo, trigger a rebuild by pushing a placeholder commit (mirrors `rebuild_repo()` from the pipeline):
```bash
TMP_DIR=$(mktemp -d)
gh repo clone "openshift-pipelines/${STALE_REPO}" "${TMP_DIR}" -- -b "${RELEASE_BRANCH}" --depth 1 --quiet
cd "${TMP_DIR}"
mkdir -p .konflux/patches/
echo "Forced multi-component rebuild at $(date)" > .konflux/patches/.placeholder
git config user.name "One Click Release Bot"
git config user.email "one-click-release-bot@redhat.com"
git add .
git commit -m "One Click Release: build all konflux components" --quiet
git push origin "${RELEASE_BRANCH}" --quiet
cd -
rm -rf "${TMP_DIR}"
```

After pushing, wait for Konflux pipelines to complete. Check pipeline status:
```bash
gh api "repos/openshift-pipelines/${STALE_REPO}/commits/${NEW_SHA}/check-runs" \
  --jq '[.check_runs[] | select(.status == "queued" or .status == "in_progress")] | length'
```

When pipelines finish, re-verify the snapshot to confirm all repos are CURRENT.

---

## Step 2.3: Core stage release

Create a Konflux Release CR targeting the core stage release plan. This must happen as soon as the core snapshot is verified — before nudge PRs are merged and before the bundle stage release.

**Important:** The core snapshot operator revision should be compared against the last operator SOURCE commit, not the current branch HEAD. Nudge PR merges update `project.yaml` only — they do not trigger a new operator image build and will make the HEAD appear ahead. Verify by checking Konflux check runs on the snapshot's operator revision: if only `operator-1-23-bundle` runs are present (no `operator-1-23-operator` build), the snapshot is valid.

**Requires:** `KONFLUX_SERVER` and `KONFLUX_TOKEN`. If missing, SKIP this step.

### Verify

Check for an existing core stage release using the latest snapshot:
```bash
oc get releases -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify -o json 2>/dev/null \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
mm = '${MM_DASHED}'
for item in data.get('items', []):
    rp = item.get('spec', {}).get('releasePlan', '')
    snapshot = item.get('spec', {}).get('snapshot', '')
    conditions = item.get('status', {}).get('conditions', [])
    released = next((c for c in conditions if c.get('type') == 'Released'), {})
    if mm in rp and 'core' in rp and 'stage' in rp and 'cdn' not in rp:
        print(f\"Release: {item['metadata']['name']}\")
        print(f\"ReleasePlan: {rp}\")
        print(f\"Snapshot: {snapshot}\")
        print(f\"Status: {released.get('status', 'Unknown')} ({released.get('reason', '')})\")
"
```

**Expected when DONE:** A core stage release exists for the latest core snapshot with status `Succeeded`.

### If not done — Execute (requires approval)

Find the core stage release plan:
```bash
CORE_APP="openshift-pipelines-core-${MM_DASHED}"
CORE_STAGE_RP=$(oc get releaseplans -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify -o json 2>/dev/null \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for item in data['items']:
    name = item['metadata']['name']
    app = item['spec'].get('application','')
    if app == '${CORE_APP}' and 'stage' in name and 'cdn' not in name:
        # Prefer the openshift-pipelines-core-{mm}-stage-rp naming
        if 'core-${MM_DASHED}-stage' in name:
            print(name)
            break
        print(name)
")

LATEST_CORE=$(oc get snapshots -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -l "pac.test.appstudio.openshift.io/event-type=push,appstudio.openshift.io/application=${CORE_APP}" \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[*].metadata.name}' 2>/dev/null | awk '{print $NF}')

echo "Core app:      ${CORE_APP}"
echo "Release plan:  ${CORE_STAGE_RP}"
echo "Snapshot:      ${LATEST_CORE}"
```

Generate the Release YAML and save it:
```bash
mkdir -p "${REPORT_BASE}/manifest/stage"

cat > "${REPORT_BASE}/manifest/stage/release-${VERSION}-core-stage.yaml" <<EOF
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  labels:
    appstudio.openshift.io/application: ${CORE_APP}
  generateName: ${CORE_STAGE_RP}-
  namespace: ${KONFLUX_NS}
spec:
  releasePlan: ${CORE_STAGE_RP}
  snapshot: ${LATEST_CORE}
EOF
```

Apply (`oc create`, not `oc apply` — uses `generateName`):
```bash
RELEASE_NAME=$(oc create -f "${REPORT_BASE}/manifest/stage/release-${VERSION}-core-stage.yaml" \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -o jsonpath='{.metadata.name}')
echo "Created release: ${RELEASE_NAME}"
```

Check status:
```bash
oc get release ${RELEASE_NAME} -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -o jsonpath='Released={.status.conditions[?(@.type=="Released")].status} Reason={.status.conditions[?(@.type=="Released")].reason}'
```

If `Released=True` → DONE. If still in progress, report the release name and continue — the re-verify will catch it.

**Record the snapshot used:** The core snapshot chosen here MUST be reused for the production core release (step 4.2). Save it as `STAGE_CORE_SNAPSHOT`.

---

## Step 2.4: Process nudge PRs

Mirrors pipeline task `osp-github-pr-processor` with label `konflux-nudge`. After core components build, Konflux creates nudge PRs in the operator repo to update image SHAs in `project.yaml`.

### Verify

Search for open nudge PRs:
```bash
gh search prs --owner openshift-pipelines \
  --base "${RELEASE_BRANCH}" \
  --state open \
  --json repository,url,title,labels \
  "label:konflux-nudge" 2>/dev/null
```

For each open nudge PR, check its CI status:
```bash
gh pr view "${PR_URL}" --json mergeable,mergeStateStatus,statusCheckRollup
```

Also check recently merged nudge PRs to report progress:
```bash
gh pr list --repo openshift-pipelines/operator \
  --base ${RELEASE_BRANCH} \
  --label konflux-nudge \
  --state merged --limit 20 \
  --json number,title,mergedAt
```

**Collect links:** Report each open nudge PR as `operator [#NUM](URL)` with its status. Include count of recently merged nudge PRs.

**Expected when DONE:** No open nudge PRs remain.

### If not done — Execute (requires approval)

For each nudge PR that is ready to merge (all CI passing, mergeable):
```bash
gh pr edit "${PR_URL}" --add-label "lgtm,approved,one-click-release"
gh pr review --approve "${PR_URL}"
gh pr merge "${PR_URL}" -d -r --auto
```

For nudge PRs with failing CI:
```bash
gh pr checks "${PR_URL}" --json name,bucket,link
```

Report the failing checks. If the failure is a known transient issue, suggest `/retest`:
```bash
gh pr comment "${PR_URL}" --body "/retest"
```

After merging, re-verify to confirm no open nudge PRs remain.

**Registry prefix preservation:** Nudge PRs only change image SHA digests — they do NOT change the registry prefix. For example, a nudge PR changes `registry.stage.redhat.io/osp-stage/image@sha256:OLD` to `registry.stage.redhat.io/osp-stage/image@sha256:NEW` — the `registry.stage.redhat.io` prefix must be preserved. If consolidating multiple nudge PRs into one, use sed to substitute only the SHA portion (the part after `@sha256:`), not the full image reference. Never replace the registry prefix.

**Auto-trigger hazard after nudge PRs merge:** Merging nudge PRs updates `project.yaml` with new image SHAs. On release branches, this push auto-triggers `operator-update-images` (with empty environment → `devel`), which updates the CSV with devel registry references and creates an auto-merge PR. When that PR merges, the push to `olm/` auto-triggers `render-olm-catalog` (also `devel`), writing devel registry references into the catalog JSONs. **These devel auto-triggers produce wrong catalogs for the build stage — staging environment is required.** Step 2.5 handles dispatching the correct staging runs.

---

## Step 2.5: OLM catalog render

After nudge PRs merge, `project.yaml` push auto-triggers `operator-update-images` and subsequently `render-olm-catalog` — but both auto-triggered runs use devel environment (empty `github.event.inputs.environment`), which writes devel registry references into the CSV and catalogs. **The build stage targets staging, so explicit `workflow_dispatch` with `environment: staging` is required for both workflows.**

**Auto-trigger hazard:**
- `operator-update-images` auto-triggers on `project.yaml` push (release branches only). The devel auto-trigger creates a CSV PR with devel registry references — this PR must NOT be merged. Instead, dispatch with `environment: staging`.
- `render-olm-catalog` auto-triggers when `bundle.yaml` or `olm/**` files are pushed. Also runs daily at 1 AM UTC from `main` (dispatches to all release branches). The devel auto-trigger writes wrong catalog JSONs and can trigger bad FBC index builds.
- **Always wait for any in-progress devel auto-triggers to finish before dispatching staging runs.** The staging runs overwrite the devel catalogs.

**Pipeline source:** `one-click-release/README.md` Build Stage — "Update OLM config with latest bundle image, Render OLM, Build Index Images."

### Verify

Check for the auto-triggered `operator-update-images` CSV PR (should be auto-merged):
```bash
gh pr list --repo openshift-pipelines/operator \
  --head "actions/update/operator-update-images-${RELEASE_BRANCH}" \
  --state all --limit 5 \
  --json number,title,state,mergedAt,url
```

Check recent `render-olm-catalog` workflow runs on the operator repo:
```bash
gh run list --repo openshift-pipelines/operator \
  --workflow=render-olm-catalog.yaml \
  --limit 10 \
  --json databaseId,headBranch,status,conclusion,createdAt,displayTitle,url,event \
  | python3 -c "
import sys, json
runs = json.load(sys.stdin)
branch = '${RELEASE_BRANCH}'
for r in runs:
    if branch in r.get('displayTitle', '') or branch in r.get('headBranch', ''):
        event = r.get('event', 'unknown')
        trigger = 'auto-trigger' if event == 'push' else ('dispatch' if event == 'workflow_dispatch' else event)
        print(f\"Run: {r['url']}\")
        print(f\"Trigger: {trigger}  Status: {r['status']}/{r.get('conclusion', 'pending')}\")
        print(f\"Created: {r['createdAt']}\")
        print()
"
```

Filter for runs on `${RELEASE_BRANCH}` or dispatched runs (which target all release branches). Show `createdAt` timestamps as absolute local time.

Also check for auto-generated catalog JSON commits on the release branch:
```bash
gh api repos/openshift-pipelines/operator/commits?sha=${RELEASE_BRANCH}\&per_page=10 \
  --jq '.[] | select(.commit.message | test("catalog|render|OCP catalog")) | "\(.sha[:12]) \(.commit.message | split("\n")[0])"'
```

**Collect links:** Report the most recent successful `workflow_dispatch` run URL as `[render-olm-catalog](URL)`. Also report the CSV PR as `operator [#NUM](URL)` if found.

**Expected when DONE:** A staging `operator-update-images` `workflow_dispatch` run completed. The staging CSV PR is merged with all images pointing to the staging registry. A staging `render-olm-catalog` `workflow_dispatch` run completed successfully after the CSV merge. Catalog JSON commits are present on the release branch.

### If not done — Execute (requires approval)

**Step 1:** Wait for any in-progress devel auto-triggered runs to finish before dispatching staging:
```bash
DEVEL_RUN_ID=$(gh run list --repo openshift-pipelines/operator \
  --workflow=operator-update-images.yaml \
  --limit 5 \
  --json databaseId,status,event \
  --jq '[.[] | select(.status == "in_progress" or .status == "queued")] | .[0].databaseId // empty')

if [ -n "${DEVEL_RUN_ID}" ]; then
  echo "Waiting for in-progress operator-update-images run ${DEVEL_RUN_ID}..."
  gh run watch --repo openshift-pipelines/operator ${DEVEL_RUN_ID}
fi
```

If a devel CSV PR was auto-created, close it — do NOT merge it:
```bash
DEVEL_PR=$(gh pr list --repo openshift-pipelines/operator \
  --head "actions/update/operator-update-images-${RELEASE_BRANCH}" \
  --state open --limit 1 \
  --json number --jq '.[0].number // empty')

if [ -n "${DEVEL_PR}" ]; then
  echo "Closing devel CSV PR #${DEVEL_PR}..."
  gh pr close --repo openshift-pipelines/operator ${DEVEL_PR}
fi
```

**Step 2:** Dispatch `operator-update-images` with staging environment:
```bash
gh workflow run operator-update-images.yaml \
  --repo openshift-pipelines/operator \
  --ref ${RELEASE_BRANCH} \
  -f environment=staging
```

Wait for the workflow to complete and the staging CSV PR to appear:
```bash
gh run list --repo openshift-pipelines/operator \
  --workflow=operator-update-images.yaml \
  --limit 3 \
  --json databaseId,status,conclusion,createdAt,displayTitle
```

Verify the staging CSV PR — ALL images must point to the staging registry (no quay.io or devel references):
```bash
PR_NUMBER=$(gh pr list --repo openshift-pipelines/operator \
  --head "actions/update/operator-update-images-${RELEASE_BRANCH}" \
  --state open --limit 1 \
  --json number --jq '.[0].number')

gh pr diff --repo openshift-pipelines/operator ${PR_NUMBER} 2>/dev/null \
  | grep -E '^\+.*image:' \
  | head -20
```

Merge the staging CSV PR:
```bash
gh pr edit --repo openshift-pipelines/operator ${PR_NUMBER} --add-label "lgtm,approved,one-click-release"
gh pr review --approve --repo openshift-pipelines/operator ${PR_NUMBER}
gh pr merge --repo openshift-pipelines/operator ${PR_NUMBER} -d -r --auto
```

**Step 3:** Wait for any devel auto-triggered `render-olm-catalog` run to finish (merging the staging CSV PR will auto-trigger one):
```bash
DEVEL_RENDER_ID=$(gh run list --repo openshift-pipelines/operator \
  --workflow=render-olm-catalog.yaml \
  --limit 5 \
  --json databaseId,status,event \
  --jq '[.[] | select(.status == "in_progress" or .status == "queued")] | .[0].databaseId // empty')

if [ -n "${DEVEL_RENDER_ID}" ]; then
  echo "Waiting for in-progress render-olm-catalog run ${DEVEL_RENDER_ID}..."
  gh run watch --repo openshift-pipelines/operator ${DEVEL_RENDER_ID}
fi
```

Then dispatch the staging render:
```bash
gh workflow run render-olm-catalog.yaml \
  --repo openshift-pipelines/operator \
  -f branch=${RELEASE_BRANCH} \
  -f environment=staging
```

After triggering, wait for the workflow to complete:
```bash
gh run list --repo openshift-pipelines/operator \
  --workflow=render-olm-catalog.yaml \
  --limit 3 \
  --json databaseId,status,conclusion,createdAt
```

Re-verify once complete.

---

## Step 2.6: Wait for FBC build

After the OLM catalog is rendered, catalog JSON changes trigger index image builds via Konflux push pipelines. Verify that bundle and index snapshots exist and are current.

**Pipeline source:** `one-click-release/scripts/wait-for-fbc-build.sh` — verifies index app snapshots the same way `wait-for-latest-snapshot.sh` verifies core.

**Requires:** `KONFLUX_SERVER` and `KONFLUX_TOKEN`. If missing, SKIP this step.

### Verify

#### 2.5a: Check bundle snapshot

```bash
BUNDLE_APP=$(oc get applications.appstudio.redhat.com -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify -o json 2>/dev/null \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
mm = '${MM_DASHED}'
apps = [i['metadata']['name'] for i in data['items'] if 'bundle' in i['metadata']['name'] and mm in i['metadata']['name']]
print(apps[0] if apps else '')
")

LATEST_BUNDLE=$(oc get snapshots -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -l "pac.test.appstudio.openshift.io/event-type=push,appstudio.openshift.io/application=${BUNDLE_APP}" \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[*].metadata.name}' 2>/dev/null | awk '{print $NF}')

echo "Latest bundle snapshot: ${LATEST_BUNDLE}"
```

Extract bundle snapshot revision and compare against operator HEAD:
```bash
BUNDLE_REV=$(oc get snapshot ${LATEST_BUNDLE} -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -o jsonpath='{.spec.components[0].source.git.revision}' 2>/dev/null)

OPERATOR_HEAD=$(git ls-remote "https://github.com/openshift-pipelines/operator.git" \
  "refs/heads/${RELEASE_BRANCH}" | awk '{print $1}')

echo "Bundle revision: ${BUNDLE_REV:0:12}"
echo "Operator HEAD:   ${OPERATOR_HEAD:0:12}"
```

If the bundle revision doesn't match operator HEAD, check the commits between them:
```bash
gh api repos/openshift-pipelines/operator/compare/${BUNDLE_REV}...${OPERATOR_HEAD} \
  --jq '.commits[] | "\(.sha[:12]) \(.commit.message | split("\n")[0])"'
```

If all intervening commits are automated (catalog/CSV/nudge), the bundle is acceptable — the gap is from expected automated activity after the bundle was built.

#### 2.5b: Check index snapshots

```bash
INDEX_APPS=$(oc get applications.appstudio.redhat.com -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify -o json 2>/dev/null \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
mm = '${MM_DASHED}'
apps = sorted([i['metadata']['name'] for i in data['items'] if 'index' in i['metadata']['name'] and mm in i['metadata']['name']])
for a in apps:
    print(a)
")
```

For each index app, get its latest snapshot and verify:
```bash
LATEST_INDEX=$(oc get snapshots -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -l "pac.test.appstudio.openshift.io/event-type=push,appstudio.openshift.io/application=${INDEX_APP}" \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[*].metadata.name}' 2>/dev/null | awk '{print $NF}')
```

Compare each index snapshot's operator revision against HEAD. Index apps with empty config dirs (no components for this release) will have no snapshot — this is expected.

**Collect links:** Report snapshot names and creation times. Note any stale snapshots.

**Expected when DONE:** Bundle snapshot exists (automated commit gap is acceptable). All index snapshots with non-empty config are current.

### If not done — Execute (requires approval)

For stale bundle, push placeholder to `.konflux/olm-catalog/bundle/` to trigger a bundle rebuild:
```bash
TMP_DIR=$(mktemp -d)
gh repo clone "openshift-pipelines/operator" "${TMP_DIR}" -- -b "${RELEASE_BRANCH}" --depth 1 --quiet
cd "${TMP_DIR}"
mkdir -p .konflux/olm-catalog/bundle/
echo "Forced bundle rebuild at $(date)" > .konflux/olm-catalog/bundle/.placeholder
git config user.name "One Click Release Bot"
git config user.email "one-click-release-bot@redhat.com"
git add .
git commit -m "One Click Release: rebuild bundle" --quiet
git push origin "${RELEASE_BRANCH}" --quiet
cd -
rm -rf "${TMP_DIR}"
```

For stale index images, push placeholder to `.konflux/olm-catalog/index/` to trigger index rebuilds:
```bash
TMP_DIR=$(mktemp -d)
gh repo clone "openshift-pipelines/operator" "${TMP_DIR}" -- -b "${RELEASE_BRANCH}" --depth 1 --quiet
cd "${TMP_DIR}"
mkdir -p .konflux/olm-catalog/index/
echo "Forced index rebuild at $(date)" > .konflux/olm-catalog/index/.placeholder
git config user.name "One Click Release Bot"
git config user.email "one-click-release-bot@redhat.com"
git add .
git commit -m "One Click Release: rebuild index images" --quiet
git push origin "${RELEASE_BRANCH}" --quiet
cd -
rm -rf "${TMP_DIR}"
```

After pushing, wait for Konflux pipelines to complete, then re-verify snapshots.

---

## Step 2.7: Bundle stage release

Create a Konflux Release CR targeting the bundle stage release plan. This must happen **after** the core stage release (step 2.3) succeeds — the core release publishes component images to the staging registry, which the bundle enterprise contract validates.

**Ordering constraint:** Core stage release (step 2.3) must have status `Succeeded` before creating this release. If core release is still in progress or failed, do not proceed.

**Requires:** `KONFLUX_SERVER` and `KONFLUX_TOKEN`. If missing, SKIP this step.

### Verify

Check for an existing bundle stage release using the latest bundle snapshot:
```bash
oc get releases -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify -o json 2>/dev/null \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
mm = '${MM_DASHED}'
for item in data.get('items', []):
    rp = item.get('spec', {}).get('releasePlan', '')
    snapshot = item.get('spec', {}).get('snapshot', '')
    conditions = item.get('status', {}).get('conditions', [])
    released = next((c for c in conditions if c.get('type') == 'Released'), {})
    if mm in rp and 'bundle' in rp and 'stage' in rp:
        print(f\"Release: {item['metadata']['name']}\")
        print(f\"ReleasePlan: {rp}\")
        print(f\"Snapshot: {snapshot}\")
        print(f\"Status: {released.get('status', 'Unknown')} ({released.get('reason', '')})\")
"
```

**Expected when DONE:** A bundle stage release exists for the latest bundle snapshot with status `Succeeded`.

### If not done — Execute (requires approval)

First verify the core stage release (step 2.3) has Succeeded — do not proceed if it hasn't.

Find the bundle application and stage release plan:
```bash
BUNDLE_APP="openshift-pipelines-bundle-${MM_DASHED}"
BUNDLE_STAGE_RP=$(oc get releaseplans -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify -o json 2>/dev/null \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for item in data['items']:
    name = item['metadata']['name']
    app = item['spec'].get('application','')
    if app == '${BUNDLE_APP}' and 'stage' in name:
        print(name)
        break
")

LATEST_BUNDLE=$(oc get snapshots -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -l "pac.test.appstudio.openshift.io/event-type=push,appstudio.openshift.io/application=${BUNDLE_APP}" \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[*].metadata.name}' 2>/dev/null | awk '{print $NF}')

echo "Bundle app:    ${BUNDLE_APP}"
echo "Release plan:  ${BUNDLE_STAGE_RP}"
echo "Snapshot:      ${LATEST_BUNDLE}"
```

Generate the Release YAML and apply:
```bash
mkdir -p "${REPORT_BASE}/manifest/stage"

cat > "${REPORT_BASE}/manifest/stage/release-${VERSION}-bundle-stage.yaml" <<EOF
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  labels:
    appstudio.openshift.io/application: ${BUNDLE_APP}
  generateName: ${BUNDLE_STAGE_RP}-
  namespace: ${KONFLUX_NS}
spec:
  releasePlan: ${BUNDLE_STAGE_RP}
  snapshot: ${LATEST_BUNDLE}
EOF

RELEASE_NAME=$(oc create -f "${REPORT_BASE}/manifest/stage/release-${VERSION}-bundle-stage.yaml" \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -o jsonpath='{.metadata.name}')
echo "Created release: ${RELEASE_NAME}"
```

Check status:
```bash
oc get release ${RELEASE_NAME} -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -o jsonpath='Released={.status.conditions[?(@.type=="Released")].status} Reason={.status.conditions[?(@.type=="Released")].reason}'
```

If `Released=True` → DONE. If still in progress, report the release name and continue.

**`olm.unmapped_references` failure:** If the bundle stage release fails with `olm.unmapped_references`, the bundle CSV references component images not yet in the staging registry. Fix: create a new core stage release using a snapshot that includes all images referenced by the bundle's staging CSV, then retry the bundle release after the new core release succeeds.

---

## Step 2.8: Index stage releases

Create Konflux Release CRs targeting the stage FBC release plans for each index application. These releases trigger IIB builds that produce the index images used in Step 3 (Image Copy).

**Ordering constraint:** Bundle stage release (step 2.7) must have status `Succeeded` before creating index releases. The full ordering is: core → bundle → index.

**Requires:** `KONFLUX_SERVER` and `KONFLUX_TOKEN`. If missing, SKIP this step.

### Verify

Check for existing index stage releases:
```bash
oc get releases -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify -o json 2>/dev/null \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
mm = '${MM_DASHED}'
releases = []
for item in data.get('items', []):
    rp = item.get('spec', {}).get('releasePlan', '')
    if mm in rp and ('fbc' in rp or 'index' in rp) and 'stage' in rp:
        conditions = item.get('status', {}).get('conditions', [])
        released = next((c for c in conditions if c.get('type') == 'Released'), {})
        releases.append({
            'name': item['metadata']['name'],
            'releasePlan': rp,
            'snapshot': item.get('spec', {}).get('snapshot', ''),
            'status': released.get('status', 'Unknown'),
            'reason': released.get('reason', '')
        })
for r in sorted(releases, key=lambda x: x['releasePlan']):
    print(f\"{r['releasePlan']}: snapshot={r['snapshot']} status={r['status']} reason={r['reason']}\")
"
```

**Expected when DONE:** Index stage releases exist for all index applications with valid snapshots, all with status `Succeeded`.

### If not done — Execute (requires approval)

First verify the bundle stage release (step 2.7) has Succeeded — do not proceed if it hasn't.

For each index app with a snapshot, create a stage Release CR:
```bash
INDEX_APPS=$(oc get applications.appstudio.redhat.com -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify -o json 2>/dev/null \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
mm = '${MM_DASHED}'
apps = sorted([i['metadata']['name'] for i in data['items'] if 'index' in i['metadata']['name'] and mm in i['metadata']['name']])
for a in apps:
    print(a)
")

mkdir -p "${REPORT_BASE}/manifest/stage"

for INDEX_APP in ${INDEX_APPS}; do
  LATEST_SNAPSHOT=$(oc get snapshots -n ${KONFLUX_NS} \
    --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
    --insecure-skip-tls-verify \
    -l "pac.test.appstudio.openshift.io/event-type=push,appstudio.openshift.io/application=${INDEX_APP}" \
    --sort-by=.metadata.creationTimestamp \
    -o jsonpath='{.items[*].metadata.name}' 2>/dev/null | awk '{print $NF}')

  if [ -z "${LATEST_SNAPSHOT}" ]; then
    echo "Skipping ${INDEX_APP}: no snapshot"
    continue
  fi

  INDEX_STAGE_RP=$(oc get releaseplans -n ${KONFLUX_NS} \
    --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
    --insecure-skip-tls-verify -o json 2>/dev/null \
    | python3 -c "
import sys, json
data = json.load(sys.stdin)
app = '${INDEX_APP}'
for item in data['items']:
    if item['spec'].get('application') == app and 'stage' in item['metadata']['name']:
        print(item['metadata']['name'])
        break
")

  if [ -z "${INDEX_STAGE_RP}" ]; then
    echo "Skipping ${INDEX_APP}: no stage release plan found"
    continue
  fi

  OCP_VERSION=$(echo "${INDEX_APP}" | sed "s/openshift-pipelines-index-//" | sed "s/-${MM_DASHED}//")

  cat > "${REPORT_BASE}/manifest/stage/release-${VERSION}-index-${OCP_VERSION}-stage.yaml" <<EOF
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  labels:
    appstudio.openshift.io/application: ${INDEX_APP}
  generateName: ${INDEX_STAGE_RP}-
  namespace: ${KONFLUX_NS}
spec:
  releasePlan: ${INDEX_STAGE_RP}
  snapshot: ${LATEST_SNAPSHOT}
EOF

  RELEASE_NAME=$(oc create -f "${REPORT_BASE}/manifest/stage/release-${VERSION}-index-${OCP_VERSION}-stage.yaml" \
    --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
    --insecure-skip-tls-verify \
    -o jsonpath='{.metadata.name}')
  echo "Created release: ${RELEASE_NAME} (${INDEX_APP})"
done
```

After all releases are created, check their status:
```bash
oc get releases -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify -o json 2>/dev/null \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
mm = '${MM_DASHED}'
for item in data.get('items', []):
    rp = item.get('spec', {}).get('releasePlan', '')
    if mm in rp and ('fbc' in rp or 'index' in rp) and 'stage' in rp:
        conditions = item.get('status', {}).get('conditions', [])
        released = next((c for c in conditions if c.get('type') == 'Released'), {})
        print(f\"{item['metadata']['name']}: {released.get('status','Unknown')} ({released.get('reason','')})\")
"
```

---

## Step 2.9: CDN production release

After the core snapshot is verified (step 2.2) and core stage release succeeds (step 2.3), CLI binaries can be released to CDN. This step creates a Konflux Release CR targeting the CDN release plan.

Stage release of the binaries requires manual product version configuration in stage CDN, so go directly to production release while keeping the `invisible` flag set to `true` in the product version YAML.

**Requires:** `KONFLUX_SERVER` and `KONFLUX_TOKEN`. If missing, SKIP this step.

### Verify

Check for existing CDN releases:
```bash
oc get releases -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify 2>&1 | grep -E "${MM_DASHED}.*(cdn-prod)"
```

If a CDN release exists, check its status (look for `Succeeded` vs `Failed`).

Get the latest core snapshot:
```bash
oc get snapshot -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -o jsonpath="{range .items[*]} {.metadata.creationTimestamp}{'\t'} {.metadata.labels.pac\.test\.appstudio\.openshift\.io\/event-type}{'\t'} {.metadata.name} {'\n'}{end}" \
  --sort-by=.metadata.creationTimestamp \
  -l "pac.test.appstudio.openshift.io/event-type=push,appstudio.openshift.io/application=openshift-pipelines-core-${MM_DASHED}" \
  2>/dev/null | tail -5
```

**Collect links:** Report the latest snapshot name and any existing CDN release status.

**Expected when DONE:** A CDN production release exists with status `Succeeded`.

### If not done — Execute (requires approval)

Generate and apply a Release CR.

Get the latest core snapshot:
```bash
LATEST_SNAPSHOT=$(oc get snapshots -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -l "pac.test.appstudio.openshift.io/event-type=push,appstudio.openshift.io/application=openshift-pipelines-core-${MM_DASHED}" \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[*].metadata.name}' 2>/dev/null | awk '{print $NF}')
```

Write the Release YAML to `${REPORT_BASE}/manifest/prod/release-${VERSION}-cdn-prod.yaml`:
```yaml
apiVersion: appstudio.redhat.com/v1alpha1
kind: Release
metadata:
  labels:
    appstudio.openshift.io/application: openshift-pipelines-core-${MM_DASHED}
  generateName: openshift-pipelines-${MM_DASHED}-core-cdn-prod-release-
  namespace: tekton-ecosystem-tenant
spec:
  data:
  gracePeriodDays: 10
  releasePlan: openshift-pipelines-${MM_DASHED}-core-cdn-prod
  snapshot: ${LATEST_SNAPSHOT}
```

Apply the Release CR (`oc create`, not `oc apply` — uses `generateName`):
```bash
RELEASE_NAME=$(oc create -f ${REPORT_BASE}/manifest/prod/release-${VERSION}-cdn-prod.yaml \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -o jsonpath='{.metadata.name}')
echo "Created release: ${RELEASE_NAME}"
```

Wait for the release to complete:
```bash
oc wait release/${RELEASE_NAME} -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  --for=condition=Released --timeout=300s 2>&1 || true
```

Check result:
```bash
oc get release ${RELEASE_NAME} -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -o jsonpath='Released={.status.conditions[?(@.type=="Released")].status} Reason={.status.conditions[?(@.type=="Released")].reason}'
```

If `Released=True` → DONE. If still in progress (condition not yet set or `Unknown`), report the release name and move on — the re-verify will catch it on the next run.

After the release succeeds, update the product version YAML to set `invisible: false`.

---

## Step 2.10: Code freeze

The `code-freeze` field in the hack release config should be set to `true` when builds are complete and index stage releases have succeeded (step 2.8) — this is when index images are ready for QE testing.

**Prerequisite:** OLM catalog render (step 2.5) must be DONE before setting code freeze. The `update-sources.yaml` workflow in the operator repo is disabled when `code-freeze: true`, so any olm/config.yaml changes and upstream source syncs must happen before this step. If olm/config.yaml is missing the version entry and code freeze is already on, the update-sources workflow is blocked and requires manual re-enabling. The `create-new-patch` workflow resets `code-freeze: false` when bumping the version, so it must be manually set back to `true`.

### Verify

```bash
CODE_FREEZE=$(gh api repos/openshift-pipelines/hack/contents/config/downstream/releases/${MAJOR_MINOR}.yaml \
  --jq '.content' | base64 -d | grep 'code-freeze:' | awk '{print $2}')
echo "code-freeze: ${CODE_FREEZE}"
```

**Collect links:** If a code freeze PR exists, report it:
```bash
gh pr list --repo openshift-pipelines/hack \
  --state all --limit 5 \
  --search "code-freeze ${MAJOR_MINOR} in:title" \
  --json number,title,state,mergedAt,url
```

**Expected when DONE:** `code-freeze: true`.

### If not done — Execute (requires approval)

Create a PR to set code freeze:
```bash
TMP_DIR=$(mktemp -d)
gh repo clone "openshift-pipelines/hack" "${TMP_DIR}" -- --depth 1 --quiet
cd "${TMP_DIR}"

sed -i 's/code-freeze: false/code-freeze: true/' config/downstream/releases/${MAJOR_MINOR}.yaml

git checkout -b "release/${VERSION}/code-freeze"
git config user.name "One Click Release Bot"
git config user.email "one-click-release-bot@redhat.com"
git add config/downstream/releases/${MAJOR_MINOR}.yaml
git commit -m "[bot:${MAJOR_MINOR}] Set code freeze for ${VERSION}"
git push origin "release/${VERSION}/code-freeze" --quiet

gh pr create --repo openshift-pipelines/hack \
  --base main \
  --head "release/${VERSION}/code-freeze" \
  --title "[bot:${MAJOR_MINOR}] Set code freeze for ${VERSION}" \
  --body "Sets code-freeze: true for ${MAJOR_MINOR} after builds are complete.

This prevents the update-sources workflow from running on the release branch." \
  --label automated

cd -
rm -rf "${TMP_DIR}"
```

After the PR is created, merge it:
```bash
PR_NUMBER=$(gh pr list --repo openshift-pipelines/hack \
  --head "release/${VERSION}/code-freeze" \
  --state open --limit 1 \
  --json number --jq '.[0].number')
gh pr merge --repo openshift-pipelines/hack ${PR_NUMBER} --rebase
```

Re-verify that `code-freeze: true` after merge.

---

## Report Output

After processing all steps, write the stage report to `${REPORT_BASE}/build/report_${REPORT_TIMESTAMP}.md`.

**Report format:**

```markdown
# Build Stage Report — ${VERSION}

**Generated:** ${REPORT_TIMESTAMP}
**Release:** ${VERSION} (${MAJOR_MINOR})
**Branch:** ${RELEASE_BRANCH}

## Summary

| Step | Title | Status | Details | Links |
|------|-------|--------|---------|-------|
| 2.1 | Process release PRs | {status} | {details} | {links} |
| 2.2 | Wait for core snapshot | {status} | {details} | {links} |
| 2.3 | Core stage release | {status} | {details} | {links} |
| 2.4 | Process nudge PRs | {status} | {details} | {links} |
| 2.5 | OLM catalog render | {status} | {details} | {links} |
| 2.6 | Wait for FBC build | {status} | {details} | {links} |
| 2.7 | Bundle stage release | {status} | {details} | {links} |
| 2.8 | Index stage releases | {status} | {details} | {links} |
| 2.9 | CDN production release | {status} | {details} | {links} |
| 2.10 | Code freeze | {status} | {details} | {links} |

## Step Details

### Step 2.1: Process release PRs
- **Status:** {DONE | ACTION NEEDED | SKIPPED}
- **Open PRs:** {count}
- **Merged PRs:** {count}
- **Details:** {per-PR status if any open}

### Step 2.2: Wait for core snapshot
- **Status:** {DONE | ACTION NEEDED | SKIPPED}
- **Snapshot:** {name} (created {timestamp})
- **SHA comparison:**

| Repo | Snapshot SHA | HEAD SHA | Status |
|------|-------------|----------|--------|
| {repo} | [{short}]({url}) | [{short}]({url}) | {CURRENT/STALE} |

### Step 2.3: Core stage release
- **Status:** {DONE | ACTION NEEDED | SKIPPED}
- **Release:** {name} — {Succeeded/Failed/Progressing/not found}
- **Release Plan:** {rp}
- **Snapshot:** {snapshot}
- **Manifest:** manifest/stage/release-{VERSION}-core-stage.yaml

### Step 2.4: Process nudge PRs
- **Status:** {DONE | ACTION NEEDED | SKIPPED}
- **Open nudge PRs:** {count}
- **Merged nudge PRs:** {count}
- **Details:** {per-PR status if any open}

### Step 2.5: OLM catalog render
- **Status:** {DONE | ACTION NEEDED | SKIPPED}
- **Latest run:** [{workflow-name}]({url}) ({status}/{conclusion}, {timestamp})
- **Catalog commits:** {count} catalog JSON commits on release branch

### Step 2.6: Wait for FBC build
- **Status:** {DONE | ACTION NEEDED | SKIPPED}
- **Bundle snapshot:** {name} ({CURRENT/STALE/automated gap OK})
- **Index snapshots:**

| Index App | Snapshot | Status |
|-----------|----------|--------|
| {app} | {name} | {CURRENT/STALE/EXPECTED (no components)} |

### Step 2.7: Bundle stage release
- **Status:** {DONE | ACTION NEEDED | SKIPPED}
- **Release:** {name} — {Succeeded/Failed/not found}
- **Release Plan:** {rp}
- **Snapshot:** {snapshot}
- **Manifest:** manifest/stage/release-{VERSION}-bundle-stage.yaml

### Step 2.8: Index stage releases
- **Status:** {DONE | ACTION NEEDED | SKIPPED}
- **Releases:**

| Index App | Release Plan | Snapshot | Status |
|-----------|-------------|----------|--------|
| {app} | {rp} | {snapshot} | {Succeeded/Failed/not found} |

- **Manifests:** manifest/stage/release-{VERSION}-index-*-stage.yaml

### Step 2.9: CDN production release
- **Status:** {DONE | ACTION NEEDED | SKIPPED}
- **CDN release:** {name} — {Succeeded/Failed/not found}
- **Snapshot:** {name}

### Step 2.10: Code freeze
- **Status:** {DONE | ACTION NEEDED | SKIPPED}
- **code-freeze:** {true/false}
- **PR:** hack [#{number}]({url}) — {state}

{...include details for each step checked...}

## Blocking Step

{If stopped early, show which step blocked and why. Omit if all steps DONE.}
```

**Column values:**

- **Status:** `DONE`, `ACTION NEEDED`, `SKIPPED`
- **Details:** one-line summary (e.g. `0 open, 14 merged`, `all repos CURRENT`, `code-freeze: true`)
- **Links:** all links collected for that step:
  - GitHub PRs: `repo [#NUM](URL)`
  - Snapshot names (no URL — cluster resource)
  - Multiple links separated by `, `
  - No links: `—`

Write the report file and print the path to the user:
```
Report written to: reports/${MAJOR_MINOR}/${VERSION}/build/report_${REPORT_TIMESTAMP}.md
```
