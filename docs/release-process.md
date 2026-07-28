# OpenShift Pipelines Release Process

A guide to the OpenShift Pipelines release workflow — what happens at each step, why it matters, and what can go wrong. This document is for people onboarding to the release team or anyone who wants to understand the end-to-end process.

For how to operate the `/one-click-release` skill that automates this process, see [one-click-release.md](one-click-release.md).

---

## Table of Contents

- [Overview](#overview)
- [Systems and Concepts](#systems-and-concepts)
- [Stage 1: Config](#stage-1-config)
- [Stage 2: Build](#stage-2-build)
- [Stage 3: Image Copy](#stage-3-image-copy)
- [Stage 4: Production Release](#stage-4-production-release)
- [Dependency Chains](#dependency-chains)
- [Common Failure Modes](#common-failure-modes)

---

## Overview

An OpenShift Pipelines release ships a new version of the operator and its component images (pipelines, triggers, chains, results, etc.) to production, along with CLI binaries (tkn, opc, pac) to the Red Hat developer download portal. A single release touches ~20 component repos, multiple registries, and several external systems.

The process has four stages:

| Stage | Steps | Purpose |
|-------|-------|---------|
| **Config** | 1.1–1.13 | Set up all configuration across repos and systems before any builds start |
| **Build** | 2.1–2.7 | Build all component images, render OLM catalogs, release CLI to CDN, freeze code |
| **Image Copy** | 3.1–3.2 | Copy index images to quay.io for QE testing |
| **Production Release** | 4.1–4.8 | Release core, bundle, and index images to production registries |

Steps within each stage are sequential — later steps depend on earlier ones. You cannot skip ahead.

Between Stage 3 and Stage 4, QE tests the staged images. Production release only proceeds after QE approval.

---

## Systems and Concepts

These are the systems and concepts you'll encounter throughout the process. You don't need to memorize all of this upfront — refer back here when a step mentions something unfamiliar.

### The hack repo

[`openshift-pipelines/hack`](https://github.com/openshift-pipelines/hack) is the central configuration repo. It holds:

- **Release config** (`config/downstream/releases/{MAJOR_MINOR}.yaml`) — the `release-tag` field (current version) and `code-freeze` flag. Every downstream workflow reads this config.
- **Konflux config** (`.konflux/openshift-pipelines/{MM_DASHED}/`) — Application and Component YAML that defines what Konflux builds. Generated automatically by the `generate-konflux` workflow.

Changes to the hack repo cascade to all other repos via automated PRs and workflows.

### Konflux

Konflux is the build and release system. Key concepts:

- **Application** — a logical grouping (e.g. `openshift-pipelines-core-1-21`, `openshift-pipelines-bundle-1-21`, `openshift-pipelines-index-4-16-1-21`). Each maps to a set of components.
- **Component** — a single buildable artifact within an application. Maps to a repo + Dockerfile.
- **Snapshot** — a point-in-time manifest listing every component image in an application and the source commit it was built from. Snapshots are created automatically after successful builds.
- **Release CR** — a Kubernetes custom resource that tells Konflux to release a specific snapshot through a specific release plan. Uses `generateName` (must be applied with `oc create -f`, never `oc apply -f`).
- **ReleasePlan / ReleasePlanAdmission (RP/RPA)** — configuration that defines how a release works — which registry, signing keys, advisory config. RPAs live in the `konflux-release-data` GitLab repo.
- **Namespace** — all operations target `tekton-ecosystem-tenant`.

### The operator repo

[`openshift-pipelines/operator`](https://github.com/openshift-pipelines/operator) is where the operator, bundle, and index images are built. Key files:

- **`project.yaml`** — contains `versions.current` (the release version) and image SHA references for all components. Nudge PRs update the image SHAs.
- **`olm/config.yaml`** — lists bundle versions that `render-olm-catalog` processes. The target version must be the last entry.
- **`bundle.yaml`** / CSV — the ClusterServiceVersion that describes the operator to OLM. Updated by `operator-update-images` workflow.

### Nudge PRs

After Konflux builds a component image, it creates a "nudge" PR in the operator repo. The PR updates the image SHA in `project.yaml` to point to the newly built image. Multiple nudge PRs arrive as components finish building — they need to be merged in sequence.

### OLM catalogs and FBC index images

OLM (Operator Lifecycle Manager) uses catalogs to discover operators in OperatorHub. The release process:

1. `operator-update-images` — updates the CSV with the correct registry references (staging or production)
2. `render-olm-catalog` — generates `catalog.json` files for each supported OCP version
3. Konflux builds **index images** (one per OCP version) containing these catalogs
4. Index images are what OCP clusters pull to populate OperatorHub

### Environment-specific workflows

Two operator repo workflows have an `environment` input that controls which registry references they use:

| Workflow | devel | staging | production |
|----------|-------|---------|------------|
| `operator-update-images` | quay.io (dev) | staging registry | production registry |
| `render-olm-catalog` | devel refs | staging refs | production refs |

On release branches, both workflows auto-trigger on push events with **empty environment** (defaults to devel). This is a common source of problems — see [Auto-Trigger Hazards](#auto-trigger-hazards) under Common Failure Modes.

### CDN releases

CLI binaries (tkn, opc, pac) are shipped to the Red Hat developer download portal via CDN, separate from container images. CDN releases require:

- Product version config in `konflux-release-data` (defines the version on the portal)
- CDN-specific RP/RPA resources (separate from image RP/RPAs)
- A Release CR targeting the CDN release plan

---

## Stage 1: Config

Everything needed before builds can start. This is the longest stage because the release touches many systems.

---

### Step 1.1: Create new patch version

**What:** Bumps `release-tag` in the hack repo's release config to the target version. Resets `code-freeze: false`.

**Why:** The hack repo release config is the source of truth. Every downstream workflow reads this config to know which version to build. Nothing else can proceed until it's bumped.

**Done when:** `release-tag` equals target version (e.g. `1.21.3`).

**Risks:** Workflow run failure (check the run URL). Config for this major.minor doesn't exist (need to create a new minor release config first).

---

### Step 1.2: Merge release-manager PR

**What:** Merges the PR created by the `release-new-patch` workflow. Titled `[bot: {MAJOR_MINOR}] Release Action: new-patch`.

**Why:** Until this merges, the hack repo still has the old version. Merging triggers downstream automation (step 1.3).

**Done when:** PR has state `MERGED`.

**Risks:** No PR exists (step 1.1 still running or failed). Merge conflicts (rare).

---

### Step 1.3: Konflux config PR merged

**What:** The `generate-konflux` workflow auto-triggers on push to main (from step 1.2) and creates a PR with Konflux Application/Component YAML.

**Why:** Defines what Konflux will build — which repos, branches, Dockerfiles. Without it, Konflux doesn't know about this release.

**Done when:** Generated PR (branch `actions/update/hack-update-konflux-main-{MAJOR_MINOR}`) has state `MERGED`.

**Risks:** Workflow hasn't run yet (wait a few minutes after step 1.2). Unexpected config changes (review diff).

---

### Step 1.4: Konflux config applied on cluster

**What:** Verifies the Konflux cluster state matches hack repo config. Checks Applications (1.4a) and Components per Application (1.4b).

**Why:** The cluster must have the right resources before builds can run. Missing Components cause builds to fail or land in wrong snapshots.

**Done when:** Every hack repo directory has a matching cluster resource. All `OK`.

**Naming:** Dots in OCP versions become dashes on cluster (e.g. hack directory `index-4.17` maps to cluster Application `index-4-17`).

**Risks:**
- `DRIFT` — Component assigned to wrong Application. Re-applying config fixes it.
- `MISSING` — Component doesn't exist on cluster. Creating it fixes it.
- RBAC errors for `role.yaml` / `service-account.yaml` — these are one-time admin resources excluded from automated apply.

**Critical:** Do NOT merge component PRs in downstream repos (step 2.1) before this config is applied — pipelines triggered by those PRs will fail.

---

### Step 1.5: RPA in konflux-release-data

**What:** Checks for ReleasePlanAdmission files in the `konflux-release-data` GitLab repo. Expected RPAs: core, bundle, fbc, cdn (stage + prod = 8 files).

**Why:** Without RPAs, Konflux can build images but can't release them. Releases fail with "no matching ReleasePlanAdmission."

**Done when:** RPA files exist for all four categories with stage and prod variants.

**How to fix:** Copy RPAs from hack repo `.konflux/` to GitLab via merge request. Reference a previous release's MR for the format.

---

### Step 1.6: Pyxis config

**What:** Checks Pyxis repo configurations for `openshift-pipelines` container images.

**Why:** Defines how images appear in the Red Hat container catalog.

**Done when:** Config entries exist under `products/openshift-pipelines/`. Usually a no-op for patch releases.

**How to fix:** Copy config from hack repo to `pyxis-repo-configs` GitLab repo. Only needed for minor releases with new components.

---

### Step 1.7: OLM bundle version in olm/config.yaml

**What:** Checks that `olm/config.yaml` on the release branch lists the target version as the last bundle entry.

**Why:** If the version is missing, `render-olm-catalog` produces **empty `catalog.json` files**. This silently breaks all index builds — they build but ship empty catalogs, and the operator won't appear in OperatorHub.

**Done when:** Last `version:` entry equals target version.

**Risks:**
- `render-catalog.sh` CI check may re-add the old version (remove it again)
- Initial CI failure on a stale image SHA is expected
- **Must be done BEFORE code freeze** — `update-sources` is disabled during freeze (`if: false`), so missing this requires manually re-enabling the workflow

---

### Step 1.8: Operator project.yaml version

**What:** Checks that `project.yaml` on the release branch has `versions.current` set to the target version.

**Why:** The operator binary reads its version from `project.yaml` at build time. Wrong version here means a wrong version string in production.

**Done when:** `versions.current` equals target version.

**Risks:** If merged after the operator image is already built, the image ships the wrong version. Requires a rebuild. Must complete before build validation (Stage 2).

---

### Step 1.9: OPC version.json

**What:** Checks OPC's `pkg/version.json` on the release branch against latest upstream releases for each component (pac, tkn, results, manualapprovalgate, assist).

**Why:** OPC bundles multiple CLI tools. Stale versions ship outdated binaries.

**Done when:** All components `CURRENT` and `opc` field equals target version.

**How to fix:** Update `pkg/version.json`, update `go.mod`, run `go mod tidy && go mod vendor`, create PR.

**Note:** A component with a different major/minor series than upstream (e.g. `0.42.x` vs `0.43.x`) is flagged as `REVIEW`, not `OUTDATED` — it may be intentional for this release branch.

---

### Step 1.10: p12n-opc sync

**What:** Checks that `p12n-opc` repo's `upstream/pkg/version.json` matches OPC's on the release branch.

**Why:** `p12n-opc` is a productization wrapper around OPC. Divergence means the productized binary doesn't match what was tested upstream.

**Done when:** `version.json` files in both repos are identical.

**How to fix:** Sync `upstream/` directory and create PR.

---

### Step 1.11: serve-tkn-cli submodules

**What:** Checks git submodule SHAs under `sources/` match the HEAD of their tracking branches.

**Why:** `serve-tkn-cli` builds actual CLI binaries for CDN. Stale submodules mean binaries built from old code.

**Done when:** Every submodule's recorded SHA matches the HEAD of its tracking branch.

**How to fix:** `git submodule update --init --remote --force --checkout` on the release branch. Plain `git submodule update` (without `--remote --force --checkout`) only checks out the recorded SHA and reports "already up to date."

---

### Step 1.12: CLI product version config

**What:** Checks for product version YAML in `konflux-release-data` at `data/external/developer-portal/openshift-pipelines/{VERSION}.yaml`.

**Why:** Defines how the version appears on the Red Hat Developer Portal. CDN RP/RPA (step 1.13) references this version — you can't create a release plan for a version that doesn't exist.

**Done when:** File exists with `versionName` matching target version.

**Key fields:** `ga` (false for patches, true for minor releases), `releaseDate`, `hidden`/`invisible` (both false).

**Critical:** Must be merged BEFORE step 1.13.

---

### Step 1.13: CLI CDN RP/RPA

**What:** Checks for CDN-specific ReleasePlanAdmission and ReleasePlan resources. These are separate from image RPAs in step 1.5.

**Why:** Without CDN RP/RPAs, CLI binaries can't reach the download portal even if they're built.

**Done when:** All required CDN RP/RPA files exist.

**How to fix:** Copy from a previous version and update version numbers.

**Critical:** Step 1.12 must be merged first.

---

## Stage 2: Build

Build all component images, render OLM catalogs for staging, release CLI to CDN, and freeze the release branch.

---

### Step 2.1: Process release PRs

**What:** Searches for open PRs across all `openshift-pipelines` repos on the release branch with labels `hack`, `upstream`, `automated`. This is the "fan-out" moment where config changes from Stage 1 propagate to ~20 component repos.

**Why:** Component repos need updated Dockerfiles, dependencies, and build config before Konflux can build the release images.

**Done when:** No open PRs remain with release labels.

**Risks:**
- **CI FAILING** — most common blocker. Flaky tests, missing config (step 1.4 not done), or dependency issues.
- **CONFLICT** — someone merged to the release branch between PR creation and now. Needs manual rebase.
- **CI PENDING** — normal shortly after PR creation. Wait a few minutes.

---

### Step 2.2: Wait for core snapshot

**What:** Verifies the latest Konflux core snapshot has the correct commit SHA for each component. The core snapshot is a manifest of every component image and its source commit.

**Why:** The core snapshot is the input to everything downstream — nudge PRs, releases, CDN. Stale commits mean stale images.

**Done when:** All non-operator repos are `CURRENT`. The operator is excluded because nudge commits (step 2.3) push it ahead.

**Risks:**
- **STALE** — a PR merged after the last build. Triggering a rebuild fixes it (5–15 minutes).
- **SPLIT** — same repo has components built from different revisions. Partial rebuild — a full rebuild fixes it.

---

### Step 2.3: Process nudge PRs

**What:** After core components build, Konflux creates nudge PRs in the operator repo updating image SHAs in `project.yaml`.

**Why:** Nudge PRs are how the operator learns about new component images. Without merging, the operator deploys stale components.

**Done when:** No open nudge PRs remain.

**Risks:**
- Nudge PRs arrive in waves as each component finishes. May need multiple passes.
- CI failures are often transient — retesting usually resolves them.
- Conflicts between concurrent nudge PRs — merge one at a time.

**Auto-trigger warning:** Merging nudge PRs pushes `project.yaml`, which auto-triggers `operator-update-images` with devel environment (wrong registry). This is expected — step 2.4 dispatches the correct staging run.

---

### Step 2.4: OLM catalog render (staging)

**What:** Dispatches `operator-update-images` and `render-olm-catalog` with `environment: staging`. This updates the CSV and catalog JSONs with staging registry references.

**Why:** Auto-triggered runs from nudge PR merges use devel environment, producing wrong registry references. Explicit staging dispatch is required.

**Done when:** A staging `operator-update-images` run completed, the staging CSV PR merged (all images point to staging registry), and a staging `render-olm-catalog` run completed after the CSV merge.

**Sequence:**
1. Wait for in-progress devel auto-triggers to finish. Close any devel CSV PR.
2. Dispatch `operator-update-images` with `environment: staging`. Wait, verify CSV PR, merge.
3. Wait for devel `render-olm-catalog` auto-trigger (from CSV merge) to finish. Dispatch staging render.

**Risks:** Merging the staging CSV PR auto-triggers a devel `render-olm-catalog` — this is expected. The staging dispatch overwrites it.

---

### Step 2.5: Wait for FBC build

**What:** Verifies bundle and FBC index snapshots are current.

**Why:** Index images are what OCP clusters pull to populate OperatorHub. They're the final built artifact before release.

**Done when:** Bundle snapshot exists (automated commit gap is acceptable). All index snapshots with non-empty config are current.

**Note on bundle staleness:** The bundle revision may not exactly match operator HEAD. If the gap is only automated commits (catalog renders, CSV updates), the bundle is considered acceptable.

**Risks:** Index apps with empty config dirs (OCP versions that don't ship this operator) have no snapshot — expected and OK.

---

### Step 2.6: CDN production release

**What:** Creates a Konflux Release CR releasing CLI binaries to the Red Hat developer download portal.

**Why:** CLI binaries ship separately from container images. Users download them from CDN.

**Done when:** CDN production release exists with status `Succeeded`.

**Why production directly (not stage first):** Stage CDN requires manual product version configuration in the stage CDN environment. Going directly to production with `invisible: true` skips this. After the release succeeds, update the product version YAML to set `invisible: false`.

**Risks:** Core snapshot must be verified first (step 2.2). Pipeline failures are typically RPA/RP config issues (steps 1.12–1.13).

---

### Step 2.7: Code freeze

**What:** Sets `code-freeze: true` in the hack release config.

**Why:** Disables `update-sources` on the release branch, preventing upstream changes from landing during QE testing. Without freeze, a new upstream commit could break the verified build.

**Done when:** `code-freeze: true` in hack config.

**Risks:** Setting too early blocks needed updates. `create-new-patch` (step 1.1) resets freeze to `false`, so it must be re-set for each patch.

---

## Stage 3: Image Copy

Copy FBC-built index images from the Konflux build registry to quay.io for QE testing. This is a temporary stage standing in for a full test stage.

---

### Step 3.1: Extract index image digests

**What:** For each index application, fetches the latest push snapshot and extracts the `containerImage` reference (a `quay.io/...@sha256:...` digest).

**Why:** The Konflux build registry is not directly accessible to QE. Images need to be copied to `quay.io/openshift-pipeline/` with a version-tagged reference.

**Done when:** All index apps with snapshots have `containerImage` references extracted.

**Risks:** A snapshot with no `containerImage` means the FBC build hasn't completed — go back to step 2.5.

---

### Step 3.2: Copy index images to quay.io

**What:** Copies each index image to `quay.io/openshift-pipeline/pipelines-index-{OCP_VERSION}:v{VERSION}-stage`.

**Done when:** All images exist on quay.io with correct digests.

**Risks:** Quay.io auth needed. Digest mismatch if source was rebuilt between extract and copy (re-run from step 3.1).

---

## Stage 4: Production Release

Release core, bundle, and index applications to production via Konflux Release CRs. This stage only begins after QE approves the staged images.

**Critical invariant:** The core snapshot used for stage MUST be reused for production. Do NOT pick a different snapshot.

---

### Step 4.1: Verify stage releases / identify core snapshot

**What:** Checks that stage releases (core, bundle, index) completed successfully. Extracts the core snapshot used for stage.

**Why:** Production release must use the same snapshot that was stage-tested. This ensures production gets exactly what QE verified.

**Done when:** Stage core release exists with status `Succeeded`. Core snapshot extracted.

**If not done:** This is a BLOCKER. Stage releases must succeed before production can begin.

---

### Step 4.2: Core production release

**What:** Creates a Release CR targeting the production release plan, reusing the core snapshot from stage.

**Done when:** Production core release exists with status `Succeeded`.

---

### Step 4.3: Production CSV update (operator-update-images)

**What:** Dispatches `operator-update-images` with `environment: production` to update the CSV with production registry references.

**Why:** Auto-triggered runs use devel environment. Explicit dispatch with `environment: production` is required.

**Done when:** A production `operator-update-images` run completed and the CSV PR is created.

---

### Step 4.4: Merge production CSV PR

**What:** Verifies and merges the CSV PR from step 4.3. ALL image references must point to the production registry.

**Why:** Any staging registry references in a production CSV would cause the operator to pull from the wrong registry in production.

**Done when:** CSV PR merged with no staging registry references.

**Risks:**
- Staging registry references in diff = BLOCKER (re-trigger step 4.3)
- Merging auto-triggers a devel `render-olm-catalog` — expected, step 4.7 handles it

---

### Step 4.5: Wait for bundle snapshot

**What:** After the production CSV PR merges, the bundle image rebuilds with production image references. Verifies the bundle snapshot includes the production CSV merge commit.

**Done when:** Bundle snapshot revision includes the production CSV merge (or only automated commits follow it).

---

### Step 4.6: Bundle production release

**What:** Creates a Release CR for the bundle application using the latest bundle snapshot (which now has production CSV).

**Done when:** Production bundle release exists with status `Succeeded`.

---

### Step 4.7: Production OLM catalog render + wait for index snapshots

**What:** Dispatches `render-olm-catalog` with `environment: production` to update catalog JSONs. Then waits for all index images to rebuild from production catalogs.

**Why:** The devel auto-trigger from step 4.4 wrote wrong catalog JSONs. The production dispatch overwrites them, triggering fresh index builds.

**Sequence:**
1. Wait for in-progress devel auto-triggers to finish
2. Dispatch production `render-olm-catalog`
3. Wait for production run to complete
4. Wait for index snapshots to rebuild

**Done when:** A production `render-olm-catalog` dispatch completed. All index snapshots with non-empty config are `CURRENT` and built AFTER the production catalog commit.

---

### Step 4.8: Index production releases

**What:** For each index application with a valid snapshot, creates a Release CR targeting the production FBC release plan.

**Done when:** All index applications with valid snapshots have production releases with status `Succeeded`.

---

## Dependency Chains

### Within Stage 1 (Config)

```
1.1 (version bump) -> 1.2 (merge PR) -> 1.3 (Konflux config PR) -> 1.4 (apply config on cluster)
```

Steps 1.5–1.13 are independent of 1.1–1.4 and could theoretically run in parallel. The important ordering constraints are:

```
1.12 (CLI product version) -- must complete BEFORE -- 1.13 (CLI CDN RP/RPA)
```

### Within Stage 2 (Build)

```
2.1 (release PRs) -> 2.2 (core snapshot) -> 2.3 (nudge PRs) -> 2.4 (OLM render) -> 2.5 (FBC builds) -> 2.7 (code freeze)
                     |
                     2.6 (CDN release -- uses core snapshot)
```

### Within Stage 4 (Production Release)

```
4.1 (verify stage) -> 4.2 (core prod) -> 4.3 (CSV update) -> 4.4 (merge CSV PR) -> 4.5 (bundle snapshot) -> 4.6 (bundle prod) -> 4.7 (OLM render + index) -> 4.8 (index prod)
```

### Cross-stage dependencies

These constraints cross stage boundaries. Violating them leads to broken builds or failed releases.

| Must complete | Before | Why |
|---------------|--------|-----|
| 1.4 (config on cluster) | 2.1 (release PRs) | Pipelines triggered by release PRs fail without cluster config |
| 1.7 (OLM bundle version) | 2.7 (code freeze) | `update-sources` is disabled during freeze; can't update olm/config.yaml |
| 1.8 (operator project.yaml) | 2.2 (core snapshot) | Images built without version bump ship wrong version string |
| 1.12 (CLI product version) | 1.13 (CLI CDN RP/RPA) | RPA references a version that must already exist |
| Stage 2 complete | Stage 3 | All snapshots must be current before copying images |
| Stage 3 + QE approval | Stage 4 | Production release uses the same snapshot QE tested |
| 4.1 (stage releases verified) | 4.2 (core prod) | Must reuse the same core snapshot from stage |

---

## Common Failure Modes

### Auto-trigger hazards

On release branches, `operator-update-images` and `render-olm-catalog` auto-trigger on push events. These auto-triggered runs receive an empty `environment` input, defaulting to `devel`. This produces wrong registry references for staging or production.

The fix is always the same pattern:
1. Wait for the devel auto-trigger to finish
2. Dispatch the correct environment run (staging in Stage 2, production in Stage 4)
3. The dispatch overwrites the devel output

**Never dispatch before the devel run finishes** — otherwise the devel run may commit on top of the correct output.

### Empty OLM catalogs

If step 1.7 (OLM bundle version) is missed, `render-olm-catalog` produces empty `catalog.json` files. Index builds succeed but ship empty catalogs, and the operator silently disappears from OperatorHub. Fix: update `olm/config.yaml`, re-render catalogs, rebuild index images.

### Nudge PR conflicts

Multiple nudge PRs targeting the same `project.yaml` lines can conflict. Merge one at a time and rebase the rest.

### Stale snapshots after manual fixes

If you fix something outside the normal flow (manual commit, direct cluster edit), Konflux may not detect the change and rebuild automatically. Push a placeholder file to the appropriate `.konflux/` directory to trigger a rebuild.

### Wrong version in operator image

If `project.yaml` (step 1.8) is updated after the operator image builds, the image ships the wrong version string. Requires a rebuild — push a placeholder to trigger it.

### CDN release plan missing

CDN Release CR creation fails with "no matching ReleasePlanAdmission" if steps 1.12 (product version) or 1.13 (CDN RP/RPA) are incomplete.

### Release CR applied with `oc apply` instead of `oc create`

Release YAMLs use `generateName`, which is incompatible with `oc apply`. Always use `oc create -f`.

### Propagation delays

GitHub API, Konflux cluster syncs, and GitLab all have propagation delays. If a step appears incomplete right after fixing it, wait a few minutes and check again.
