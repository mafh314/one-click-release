# Setup

Shared setup for the one-click-release skill.

**Input:** A version string argument matching `X.Y.Z` (e.g. `1.21.3`).

## Parse Version

Parse the version argument. It must match `X.Y.Z` (patch) or `X.Y.0` (minor).

Derive:
- `VERSION` — e.g. `1.21.3`
- `MAJOR_MINOR` — e.g. `1.21`
- `MM_DASHED` — e.g. `1-21`
- `RELEASE_BRANCH` — `release-v${MAJOR_MINOR}.x`
- `RELEASE_TAG` — `v${VERSION}`
- `IS_PATCH` — true if Z > 0
- `KONFLUX_NS` — always `tekton-ecosystem-tenant`

## Source Credentials

```bash
source .env
```

Required variables and what they gate:

| Variable | Required For |
|----------|-------------|
| `GITHUB_TOKEN` or `gh auth` | All steps — GitHub API |
| `GITHUB_USER` + `GITHUB_EMAIL` | Git commit authorship for automated PRs (Step 1.8: OPC version.json) |
| `KONFLUX_SERVER` + `KONFLUX_TOKEN` | Steps that query Konflux cluster |
| `GITLAB_URL` + `GITLAB_TOKEN` | Steps that query GitLab (verify: any scope; execute: requires `api` scope) |
| `JIRA_URL` + `JIRA_EMAIL` + `JIRA_TOKEN` | Audit and Jira steps |

If a credential is missing, note which steps will be skipped.

## Check GitLab Token Write Access

After sourcing credentials, check whether `GITLAB_TOKEN` has `api` scope (write access). Steps 1.5, 1.6, 1.11, and 1.12 use `GITLAB_TOKEN_WRITEABLE` to decide between the automated execute path and the MANUAL fallback.

```bash
GITLAB_TOKEN_WRITEABLE=false
if [ -n "$GITLAB_TOKEN" ] && [ -n "$GITLAB_URL" ]; then
  TOKEN_SCOPES=$(curl -s \
    --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
    "$GITLAB_URL/api/v4/personal_access_tokens/self" \
    2>/dev/null | python3 -c "
import sys, json
try:
    d = json.load(sys.stdin)
    print(','.join(d.get('scopes', [])))
except Exception:
    print('')
")
  if echo "$TOKEN_SCOPES" | grep -q "\bapi\b"; then
    GITLAB_TOKEN_WRITEABLE=true
    echo "GitLab token: write access confirmed (api scope)"
  else
    echo "GitLab token: missing 'api' scope (current scopes: ${TOKEN_SCOPES:-unknown})"
    echo "  Steps 1.5, 1.6, 1.11, 1.12 execute paths will fall back to MANUAL."
    echo "  To enable auto-creation, provision a token with 'api' scope."
  fi
else
  echo "GitLab token: not set — steps 1.5, 1.6, 1.11, 1.12 will be skipped."
fi
```

`GITLAB_TOKEN_WRITEABLE` is available to all stage files for the remainder of the session.

## Time Formatting

```bash
LOCAL_TZ=$(date +%Z)
TZ_FMT='%Y-%m-%d %H:%M %Z'
```

All timestamps in output MUST use absolute local time. Convert API timestamps:
```bash
date -d "${TIMESTAMP}" +"${TZ_FMT}"
```

Never use relative times ("3 min ago", "6d5h").

## Fetch Release Config

```bash
RELEASE_CFG=$(gh api repos/openshift-pipelines/hack/contents/config/downstream/releases/${MAJOR_MINOR}.yaml --jq '.content' | base64 -d)
```

Parse:
```bash
CURRENT_RELEASE_TAG=$(echo "$RELEASE_CFG" | grep 'release-tag:' | awk '{print $2}')
CODE_FREEZE=$(echo "$RELEASE_CFG" | grep 'code-freeze:' | awk '{print $2}')
```

If the release config doesn't exist, stop: "Release config for ${MAJOR_MINOR} not found in hack repo."

## Report Paths

Each stage writes its own report file under a version-scoped directory:

```bash
REPORT_BASE="reports/${MAJOR_MINOR}/${VERSION}"
REPORT_TIMESTAMP=$(date +'%Y-%m-%d_%H-%M-%S_%Z')
```

Stage report paths (created by each stage file):
- Config: `${REPORT_BASE}/config/report_${REPORT_TIMESTAMP}.md`
- Build: `${REPORT_BASE}/build/report_${REPORT_TIMESTAMP}.md`
- Image Copy: `${REPORT_BASE}/image-copy/report_${REPORT_TIMESTAMP}.md`
- Release: `${REPORT_BASE}/release/report_${REPORT_TIMESTAMP}.md`
- Manifests: `${REPORT_BASE}/manifest/stage/` and `${REPORT_BASE}/manifest/prod/`

Create the directories at setup time:
```bash
mkdir -p "${REPORT_BASE}/config" "${REPORT_BASE}/build" "${REPORT_BASE}/image-copy" "${REPORT_BASE}/release" "${REPORT_BASE}/manifest/stage" "${REPORT_BASE}/manifest/prod"
```

## Return

All variables above are available for use by the calling stage files.
