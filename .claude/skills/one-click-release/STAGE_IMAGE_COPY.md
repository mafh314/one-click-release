# Stage 3: Image Copy (temporary)

Copy FBC-built index images from Konflux snapshots to quay.io for QE testing. This is a temporary stage standing in for the Test stage — it makes index images accessible on quay.io so QE can test them.

**Inputs:** `VERSION`, `MAJOR_MINOR`, `MM_DASHED`, `RELEASE_BRANCH`, `KONFLUX_NS`, `KONFLUX_SERVER`, `KONFLUX_TOKEN`, `TZ_FMT`, `REPORT_BASE`, `REPORT_TIMESTAMP`

**Constraints:**
- Konflux cluster: **READ-ONLY** for verify commands (`oc get`/`kubectl get` only)
- `skopeo` must be available on the local machine
- Execute commands require explicit user approval before running

**Formatting:**
- SHA links: `[SHORT](https://github.com/OWNER/REPO/commit/FULL)` (12-char short)
- Timestamps: absolute local time with timezone (e.g. `2026-07-08 14:30 IST`)

---

## Step 3.1: Extract IIB image digests from index stage releases

Get the resolved IIB image digest from each completed index stage release (step 2.8). The source for skopeo copy is the IIB image built by the IIB service — **not** the raw Konflux snapshot `containerImage` (which is at quay.io/redhat-user-workloads and may be inaccessible externally).

**Source registry:** `registry-proxy.engineering.redhat.com/rh-osbs/iib@sha256:DIGEST`
**Requires VPN:** `registry-proxy.engineering.redhat.com` is a Red Hat internal host — VPN is required for both this verification and the actual skopeo copy in step 3.2.

**Prerequisite:** Index stage releases (step 2.8) must all have status `Succeeded` before running this step.

**Requires:** `KONFLUX_SERVER` and `KONFLUX_TOKEN`. If missing, SKIP this step.

### Verify

Extract IIB image digests from completed index stage release CRs:
```bash
oc get releases -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify -o json 2>/dev/null \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
mm = '${MM_DASHED}'
for item in sorted(data.get('items', []), key=lambda x: x['metadata']['name']):
    rp = item.get('spec', {}).get('releasePlan', '')
    if mm in rp and ('fbc' in rp or 'index' in rp) and 'stage' in rp:
        conditions = item.get('status', {}).get('conditions', [])
        released = next((c for c in conditions if c.get('type') == 'Released'), {})
        artifacts = item.get('status', {}).get('artifacts', {})
        # Try common artifact field names for IIB resolved image
        iib_image = (artifacts.get('indexImageResolved') or
                     artifacts.get('indexImage') or
                     artifacts.get('iibIndexImageResolved') or
                     artifacts.get('index_image_resolved') or '')
        snapshot = item.get('spec', {}).get('snapshot', '')
        print(f\"ReleasePlan: {rp}\")
        print(f\"Release: {item['metadata']['name']}\")
        print(f\"Snapshot: {snapshot}\")
        print(f\"Status: {released.get('status','Unknown')} ({released.get('reason','')})\")
        print(f\"IIB image: {iib_image or '(check .status.artifacts manually)'}\")
        print()
"
```

If the IIB image field is empty, inspect the release CR directly to find the artifact field name:
```bash
oc get release ${RELEASE_NAME} -n ${KONFLUX_NS} \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -o jsonpath='{.status.artifacts}' 2>/dev/null | python3 -m json.tool
```

The IIB image will be a `registry-proxy.engineering.redhat.com/rh-osbs/iib@sha256:DIGEST` reference.

Derive the OCP version from the release plan name to construct the quay.io target tag:
- `openshift-pipelines-index-4-14-1-23-stage-fbc-rp` → OCP version `4.14` → `quay.io/openshift-pipeline/pipelines-index-4.14:v${VERSION}-stage`

Report a table:
```
| Index App | Release | IIB Image | OCP Version | Quay Target |
|-----------|---------|-----------|-------------|-------------|
```

**Expected when DONE:** All index stage releases have Succeeded and IIB image digests are extracted.

### If not done

If a release has failed or IIB digest is missing, check the release status and go back to step 2.8 to create or retry index stage releases.

---

## Step 3.2: Copy IIB index images to quay.io

Copy each IIB index image from the Red Hat internal registry to quay.io for QE access.

**Source:** `registry-proxy.engineering.redhat.com/rh-osbs/iib@sha256:DIGEST` (from step 3.1)
**Destination:** `quay.io/openshift-pipeline/pipelines-index-{OCP}:v${VERSION}-stage`
**Requires VPN:** Must be connected to Red Hat VPN to access `registry-proxy.engineering.redhat.com`.
**Requires:** `skopeo` installed locally. If not available, generate a copy script instead.

### Verify

Check if the images already exist on quay.io (requires quay credentials from `.env`):
```bash
skopeo inspect --no-tags docker://quay.io/openshift-pipeline/pipelines-index-${OCP_VERSION}:v${VERSION}-stage 2>/dev/null
```

If the image exists and the digest matches the source → **DONE** for that index.

Report a table:
```
| OCP Version | IIB Source | Quay Tag | Status |
|-------------|------------|----------|--------|
```

**Expected when DONE:** All index images exist on quay.io with correct digests.

### If not done — Execute (requires approval)

Log in to quay.io first (credentials from `.env`):
```bash
source .env
echo "${QUAY_PASSWORD}" | skopeo login quay.io -u "${QUAY_USER}" --password-stdin
```

For each index image, copy using the IIB digest as source:
```bash
skopeo copy --all \
  docker://registry-proxy.engineering.redhat.com/rh-osbs/iib@sha256:${IIB_DIGEST} \
  docker://quay.io/openshift-pipeline/pipelines-index-${OCP_VERSION}:v${VERSION}-stage \
  --preserve-digests
```

If `skopeo` is not available or VPN is not active now (but will be later), generate a copy script at `scripts/copy-index-images-${VERSION}-stage.sh`:
```bash
#!/bin/bash
# Copy ${VERSION} stage index images to quay.io/openshift-pipeline
# Source: IIB images from index stage releases
#
# Generated: ${REPORT_TIMESTAMP}
# Prerequisites: VPN connected, quay.io login active
#
# Stage releases used:
${RELEASE_COMMENTS}

set -euo pipefail

echo "=== Logging into quay.io ==="
source "$(dirname "$0")/../.env"
echo "${QUAY_PASSWORD}" | skopeo login quay.io -u "${QUAY_USER}" --password-stdin

echo "=== Running image copy script ==="
echo "Copying ${VERSION} stage index images to quay.io..."

${SKOPEO_COMMANDS}

echo "Done — all index images copied."
```

Where `${SKOPEO_COMMANDS}` contains one `skopeo copy` per index:
```bash
# OCP 4.14
skopeo copy --all \
  docker://registry-proxy.engineering.redhat.com/rh-osbs/iib@sha256:${IIB_DIGEST_4_14} \
  docker://quay.io/openshift-pipeline/pipelines-index-4.14:v${VERSION}-stage \
  --preserve-digests
# ... repeat for each OCP version
```

Make the script executable:
```bash
chmod +x scripts/copy-index-images-${VERSION}-stage.sh
```

Show the user the script path and how to run it:
```
Copy script written to: scripts/copy-index-images-${VERSION}-stage.sh

Run it (requires VPN + quay.io login):
  ./scripts/copy-index-images-${VERSION}-stage.sh

Or run individual commands above.
```

After copying, re-verify that images exist on quay.io.

---

## Report Output

After processing all steps, write the stage report to `${REPORT_BASE}/image-copy/report_${REPORT_TIMESTAMP}.md`.

**Report format:**

```markdown
# Image Copy Stage Report — ${VERSION}

**Generated:** ${REPORT_TIMESTAMP}
**Release:** ${VERSION} (${MAJOR_MINOR})
**Branch:** ${RELEASE_BRANCH}

## Summary

| Step | Title | Status | Details | Links |
|------|-------|--------|---------|-------|
| 3.1 | Extract IIB image digests | {status} | {details} | {links} |
| 3.2 | Copy IIB index images to quay.io | {status} | {details} | {links} |

## Step Details

### Step 3.1: Extract IIB image digests
- **Status:** {DONE | ACTION NEEDED | SKIPPED}
- **Index stage releases:**

| Index App | Release | IIB Image | OCP Version |
|-----------|---------|-----------|-------------|
| {app} | {release-name} | registry-proxy.engineering.redhat.com/rh-osbs/iib@sha256:{digest} | {ocp_version} |

### Step 3.2: Copy IIB index images to quay.io
- **Status:** {DONE | ACTION NEEDED | SKIPPED}
- **Images copied:**

| OCP Version | IIB Source Digest | Quay Target | Status |
|-------------|-------------------|-------------|--------|
| {ocp_version} | sha256:{digest} | quay.io/openshift-pipeline/pipelines-index-{ocp_version}:v{VERSION}-stage | {DONE/PENDING} |

- **Copy script:** {path if generated}

## Blocking Step

{If stopped early, show which step blocked and why. Omit if all steps DONE.}
```

Write the report file and print the path to the user:
```
Report written to: reports/${MAJOR_MINOR}/${VERSION}/image-copy/report_${REPORT_TIMESTAMP}.md
```
