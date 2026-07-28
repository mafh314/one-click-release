# One-Click Release — Skill Operator Guide

How to run the `/one-click-release` Claude Code skill. This document covers invocation, prerequisites, the interaction model, outputs, and troubleshooting.

For what each step does and why it matters, see [release-process.md](release-process.md).

---

## Table of Contents

- [Quick Start](#quick-start)
- [Prerequisites](#prerequisites)
- [How the Skill Works](#how-the-skill-works)
- [Stage Reference](#stage-reference)
- [Reports and Manifests](#reports-and-manifests)
- [Running Across Sessions](#running-across-sessions)
- [Troubleshooting](#troubleshooting)

---

## Quick Start

```
/one-click-release 1.21.3
```

The version argument must be `X.Y.Z` format. The skill derives everything else:

| Derived variable | Example |
|-----------------|---------|
| `MAJOR_MINOR` | `1.21` |
| `RELEASE_BRANCH` | `release-v1.21.x` |
| `KONFLUX_NS` | `tekton-ecosystem-tenant` |

**What happens:**

1. **Setup** — parses the version, sources `.env`, fetches the hack repo release config, creates the report directory tree under `reports/{MAJOR_MINOR}/{VERSION}/`
2. Runs Stage 1 (Config), checking each step in order
3. At each step, runs a verify command to check current state
4. If done — reports status, moves to next step
5. If not done — shows the execute command and asks for approval
6. If you approve — runs the command, then re-verifies
7. If you decline — stops and reports the blocking step
8. After each stage completes (or is blocked), writes a report file

The skill asks for explicit confirmation before starting Stage 4 (Production Release): "All builds verified and images copied. Ready to start production release?"

---

## Prerequisites

### Credentials

All credentials live in `.env` at the repo root. The skill sources this file at startup.

| Credential | What it gates | How to get it |
|-----------|---------------|---------------|
| `gh` CLI auth | All GitHub operations | `gh auth login` |
| `KONFLUX_SERVER` + `KONFLUX_TOKEN` | Cluster queries + Release CR creation | Konflux cluster credentials. Read access to `tekton-ecosystem-tenant`. Write access for Release CRs (stages 2 and 4) |
| `GITLAB_URL` + `GITLAB_TOKEN` | Steps 1.5, 1.6, 1.12, 1.13 | GitLab PAT at `gitlab.cee.redhat.com` — read-only sufficient |
| `JIRA_URL` + `JIRA_EMAIL` + `JIRA_TOKEN` | Audit steps (not core release flow) | Jira API token |

If a credential is missing, the skill notes which steps are skipped rather than failing.

### Tooling

- **`gh`** — GitHub CLI, authenticated
- **`oc` / `kubectl`** — for Konflux cluster operations
- **`skopeo`** — for Stage 3 image copy. If missing, the skill generates a copy script instead
- **`python3`** — for JSON parsing

### Permission Model

| Operation type | When it runs | Approval needed |
|---------------|-------------|-----------------|
| **Verify** (read-only) | Automatically | No |
| **Execute** (write) | After showing you the command | Yes — you must approve |
| **Release CR creation** (`oc create -f`) | After showing you the manifest | Yes — explicit approval |

Verify-only operations: GitHub API reads, `oc get` / `kubectl get`, GitLab `GET` requests.

You can always run the skill just to check status — it won't change anything without your approval.

---

## How the Skill Works

### The Verify-Execute Loop

Every step follows the same pattern:

```
Verify -> DONE? -> yes -> report, next step
                -> no  -> show execute command
                          -> user approves -> execute -> re-verify
                          -> user declines -> stop, report blocker
```

You never have to remember what to check or what command to run. Your job is to review the execute commands and approve or decline.

### Sequential Stopping

Steps within each stage are sequential. The skill stops at the **first** step that isn't done. This is by design — later steps depend on earlier ones. When you unblock a step, re-run the skill to continue.

### Skipping Steps

When a step isn't done and the skill asks for approval, you can tell it to **skip the step and continue** instead of approving or declining. Just say something like "skip this step" or "move on to the next one."

This is not recommended — steps are sequential for a reason, and skipping one usually means later steps will fail or produce wrong results. But there are legitimate cases:

- **You know the step will resolve itself** — e.g. a Konflux pipeline is running and will finish soon, but you want to check the status of later steps now
- **The step is genuinely not needed** — e.g. Pyxis config (step 1.6) for a patch release that ships no new components
- **You're debugging** — you want to see which later steps are also blocking, not just the first one
- **You already handled it outside the skill** — the change hasn't propagated yet, but you know it will

The skill will mark the skipped step in the report and continue to the next one. If a later step fails because of the skip, you'll need to go back and complete the skipped step, then re-run.

### Release CR Execution Pattern

Steps that create Konflux Release CRs (2.6, 4.2, 4.6, 4.8) follow a specific pattern:

1. **Generate** — writes the Release YAML to `manifest/prod/`
2. **Apply** — `oc create -f` (not `oc apply` — Release CRs use `generateName`)
3. **Wait** — `oc wait --for=condition=Released --timeout=300s` with `|| true` (non-blocking)
4. **Check** — reads the release status

If the release is still in progress after 300 seconds, the skill reports the release name and moves on. The next time you run the skill, the verify step catches the result.

### Automated vs Manual Steps

Most steps have automated execute commands. A few are marked MANUAL — the skill reports what needs to be done, but you must perform the action yourself:

| Manual steps | What to do |
|-------------|-----------|
| 1.5 (RPA) | Copy RPAs from hack repo to GitLab via MR |
| 1.6 (Pyxis) | Copy Pyxis config to GitLab (usually no-op for patches) |
| 1.9 (OPC version.json) | Update versions, go.mod, vendor, create PR |
| 1.10 (p12n-opc sync) | Sync upstream/ directory, create PR |
| 1.11 (serve-tkn-cli submodules) | Update submodules on release branch |
| 1.12 (CLI product version) | Create GitLab MR with version YAML |
| 1.13 (CLI CDN RP/RPA) | Copy from previous version via GitLab MR |

---

## Stage Reference

Quick reference for what each stage covers. For detailed step descriptions, see [release-process.md](release-process.md).

### Stage 1: Config (steps 1.1–1.13)

Sets up all configuration before builds start. 13 steps across the hack repo, Konflux cluster, GitLab, operator repo, OPC, and CLI pipeline.

**Typical duration:** 30 minutes to a few hours, depending on how many manual steps need attention.

### Stage 2: Build (steps 2.1–2.7)

Processes release PRs, waits for Konflux snapshots, handles the nudge/OLM/FBC pipeline, releases CLI to CDN, and sets code freeze.

**Typical duration:** 1–3 hours. Most time is spent waiting for Konflux builds (5–15 min each) and workflow runs.

### Stage 3: Image Copy (steps 3.1–3.2)

Extracts index image digests and copies them to quay.io for QE.

**Typical duration:** 5–10 minutes.

### Stage 4: Production Release (steps 4.1–4.8)

Releases core, bundle, and index to production. Requires explicit gate confirmation before starting.

**Typical duration:** 1–2 hours. Release pipelines take 10–30 minutes each. The non-blocking 300s wait means you don't have to watch — re-run to check status.

---

## Reports and Manifests

### Report directory

```
reports/{MAJOR_MINOR}/{VERSION}/
  config/report_{timestamp}.md
  build/report_{timestamp}.md
  image-copy/report_{timestamp}.md
  release/report_{timestamp}.md
```

Each run generates a new timestamped report. Reports include:
- Summary table with status, details, and links for every step
- Detailed per-step sections (SHAs, PR links, comparison tables)
- Blocking step section (if the stage didn't complete)

### Manifest directory

```
reports/{MAJOR_MINOR}/{VERSION}/manifest/
  stage/     (reserved for future use)
  prod/
    release-{VERSION}-cdn-prod.yaml
    release-{VERSION}-core-prod.yaml
    release-{VERSION}-bundle-prod.yaml
    release-{VERSION}-index-{OCP_VERSION}-prod.yaml  (one per OCP version)
```

All generated Release CR YAML files are saved here before being applied. This provides an audit trail of exactly which Release CRs were created.

### Link formats in reports

| Type | Format |
|------|--------|
| GitHub PR | `repo [#NUM](URL)` |
| Workflow run | `[workflow-name](URL)` |
| GitLab MR | `[MR](URL)` |
| Commit SHA | `[12-char-short](URL)` |
| No links | `---` |

---

## Running Across Sessions

### Resumability

The skill is fully resumable. Stop mid-release (close terminal, end session) and pick up by running `/one-click-release {VERSION}` again. The skill re-verifies all steps from the beginning, rapidly skipping completed ones, until it reaches the first incomplete step.

Key properties:
- **No state stored between sessions** — the skill queries external systems each time
- **Manual fixes are detected** — the verify commands see updated state regardless of how changes were made
- **Progress tracked in reports** — timestamped reports provide a history of what was done when
- **Multi-user safe** — the skill reads system state, not per-user state

### Typical multi-session workflow

**Session 1 (morning):** Config stage completes. Build stage starts, processes release PRs. Stops at step 2.2 (waiting for builds). Close terminal.

**Session 2 (30 min later):** Steps 1.1–2.1 verify as DONE instantly. Proceed through nudge PRs, OLM render, FBC builds. CDN release started. Close terminal.

**Session 3 (next day):** Everything through Stage 3 is DONE. Hand off to QE.

**Session 4 (after QE approval):** Confirm production gate. Proceed through core, bundle, index releases.

---

## Troubleshooting

### Missing credentials

**Symptom:** Steps reported as `SKIPPED`.

**Fix:** Check `.env` has all required variables. Most common: `KONFLUX_TOKEN` (expires periodically), `GITLAB_TOKEN` (steps 1.5, 1.6, 1.12, 1.13 skipped without it).

### Release CR stuck in progress

**Symptom:** `oc wait` times out after 300s. Release status shows `Unknown`.

**Fix:** Re-run the skill. The verify step checks the status and will report `Succeeded` or `Failed` once the pipeline finishes. Manual check:
```bash
oc get release {NAME} -n tekton-ecosystem-tenant \
  --server="$KONFLUX_SERVER" --token="$KONFLUX_TOKEN" \
  --insecure-skip-tls-verify \
  -o jsonpath='Released={.status.conditions[?(@.type=="Released")].status}'
```

### "oc create" vs "oc apply" error

**Symptom:** `oc apply -f release-*.yaml` fails with `generateName` error.

**Fix:** Use `oc create -f`. Release YAMLs use `generateName` which is incompatible with `apply`. The skill handles this automatically.

### Skill stops at a step I already fixed

**Symptom:** Re-running still stops at the same step after a manual fix.

**Cause:** Propagation delay. GitHub API, Konflux cluster, and GitLab all have sync delays.

**Fix:** Wait a few minutes and re-run. The verify commands check live system state, not cached results.

### Stale snapshots

**Symptom:** Steps 2.2, 2.5, 4.5, or 4.7 show `STALE` status.

**Fix:** Approve the execute command — it pushes a placeholder to trigger a rebuild. Wait 5–15 minutes for Konflux pipelines, then re-run.

### CI failures on PRs

**Symptom:** Steps 2.1 or 2.3 report PRs with `CI FAILING`.

**Common causes:**
- Flaky tests — `/retest` usually resolves
- Missing config — step 1.4 not done before builds started
- Dependency issues — needs manual investigation

### Wrong registry in CSV PR

**Symptom:** Step 4.4 finds staging references in a production CSV PR.

**Fix:** Close the PR. Re-dispatch `operator-update-images` with `environment: production` (step 4.3).

### Code freeze blocks needed updates

**Symptom:** Need to land a fix on the release branch, but `update-sources` is disabled.

**Fix:** Temporarily unset code freeze in the hack config, land the fix, wait for the build, re-set freeze (step 2.7).
