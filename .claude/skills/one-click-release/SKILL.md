---
description: One-click release — config, build, test, and release stages with execution support
---

# One-Click Release

<objective>
Given a release version (e.g. `1.21.3`), walk through the OpenShift Pipelines release workflow in stages, verify each step by querying external systems, and offer to execute fixes when steps are not done.

Steps within each stage are **sequential** — stop at the first step that requires action. If the user approves execution, run the execute command and re-verify.

The version is passed as the skill argument. Example: `/one-click-release 1.21.3`

**Permission model:**
- **Verify** commands run automatically (read-only)
- **Execute** commands are shown to the user and run ONLY after explicit approval

**External systems** (credentials in `.env`):
- GitHub: `gh` CLI (read/write via existing auth)
- GitLab (gitlab.cee.redhat.com): `GITLAB_URL` + `GITLAB_TOKEN` — **READ-ONLY** for verify
- Konflux cluster: `KONFLUX_SERVER` + `KONFLUX_TOKEN` — **READ-ONLY** for verify
- Jira: `JIRA_URL` + `JIRA_EMAIL` + `JIRA_TOKEN`

**Konflux namespace:** All Konflux operations target `tekton-ecosystem-tenant`.

**Formatting rules:**
- PR links: `[#NUM](https://github.com/OWNER/REPO/pull/NUM)`
- SHA links: `[SHORT](https://github.com/OWNER/REPO/commit/FULL)` (12-char short)
- Timestamps: absolute local time with timezone (e.g. `2026-07-08 14:30 IST`). Never relative.
</objective>

<process>
<step name="setup">
**Parse arguments, initialize, and source credentials.**

Read and follow the instructions in `SETUP.md`.
</step>

<step name="stage_config">
**Stage 1: Config — everything needed before builds can start.**

Read and follow the instructions in `STAGE_CONFIG.md`.

For each step:
1. Run the **Verify** command
2. If DONE → report status, move to next step
3. If not done → show the **Execute** command and ask: "Step X.Y: [description]. Execute?"
4. If user approves → run the execute command, then re-verify
5. If user declines → stop and report the blocking step

Pass `VERSION`, `MAJOR_MINOR`, `MM_DASHED`, `RELEASE_BRANCH`, `CURRENT_RELEASE_TAG`, `KONFLUX_NS`, `KONFLUX_SERVER`, `KONFLUX_TOKEN`, `GITLAB_URL`, `GITLAB_TOKEN`, `TZ_FMT`, `REPORT_BASE`, `REPORT_TIMESTAMP`.
</step>

<step name="stage_build">
**Stage 2: Build — process PRs, verify snapshots, and code freeze.**

Read and follow the instructions in `STAGE_BUILD.md`.

For each step:
1. Run the **Verify** command
2. If DONE → report status, move to next step
3. If not done → show the **Execute** command and ask: "Step X.Y: [description]. Execute?"
4. If user approves → run the execute command, then re-verify
5. If user declines → stop and report the blocking step

Pass `VERSION`, `MAJOR_MINOR`, `MM_DASHED`, `RELEASE_BRANCH`, `KONFLUX_NS`, `KONFLUX_SERVER`, `KONFLUX_TOKEN`, `TZ_FMT`, `REPORT_BASE`, `REPORT_TIMESTAMP`.
</step>

<step name="stage_image_copy">
**Stage 3: Image Copy — copy FBC index images to quay.io for QE.**

Read and follow the instructions in `STAGE_IMAGE_COPY.md`.

For each step:
1. Run the **Verify** command
2. If DONE → report status, move to next step
3. If not done → show the **Execute** command and ask: "Step X.Y: [description]. Execute?"
4. If user approves → run the execute command, then re-verify
5. If user declines → stop and report the blocking step

Pass `VERSION`, `MAJOR_MINOR`, `MM_DASHED`, `RELEASE_BRANCH`, `KONFLUX_NS`, `KONFLUX_SERVER`, `KONFLUX_TOKEN`, `TZ_FMT`, `REPORT_BASE`, `REPORT_TIMESTAMP`.
</step>

<step name="stage_prod_release">
**Stage 4: Production Release — release core, bundle, and index to production.**

**This stage requires explicit user approval before starting.** After Image Copy completes, ask:
"All builds verified and images copied. Ready to start production release? (yes/no)"
Do NOT proceed unless the user confirms.

Read and follow the instructions in `STAGE_PROD_RELEASE.md`.

For each step:
1. Run the **Verify** command
2. If DONE → report status, move to next step
3. If not done → show the **Execute** command and ask: "Step X.Y: [description]. Execute?"
4. If user approves → run the execute command, then re-verify
5. If user declines → stop and report the blocking step

Pass `VERSION`, `MAJOR_MINOR`, `MM_DASHED`, `RELEASE_BRANCH`, `KONFLUX_NS`, `KONFLUX_SERVER`, `KONFLUX_TOKEN`, `TZ_FMT`, `REPORT_BASE`, `REPORT_TIMESTAMP`.
</step>

<step name="summary">
**Write the stage report and print the summary.**

Each stage writes its report using the format defined in its **Report Output** section. The report is written to `${REPORT_BASE}/<stage>/report_${REPORT_TIMESTAMP}.md`.

After writing the report file, print the path and the summary table to the user.
</step>
</process>

<success_criteria>
- [ ] All verify commands execute successfully
- [ ] Execute commands only run after explicit user approval
- [ ] Production release stage only starts after explicit user confirmation
- [ ] Steps checked sequentially — stopped at first blocking step (unless user executed fix)
- [ ] All Konflux verify commands target `tekton-ecosystem-tenant` namespace
- [ ] GitLab and Konflux verify commands are READ-ONLY
- [ ] Konflux Release CRs applied via `oc create -f` only after explicit user approval — YAML files also saved to reports dir
- [ ] All PR numbers rendered as markdown links
- [ ] All timestamps use absolute local time with timezone
- [ ] Stage report written to `reports/{MAJOR_MINOR}/{VERSION}/{stage}/report_{timestamp}.md`
- [ ] Release manifests written to `reports/{MAJOR_MINOR}/{VERSION}/manifest/{stage|prod}/`
- [ ] Report includes workflow run URLs and PR/MR links for every step
</success_criteria>
