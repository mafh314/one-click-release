---
description: Search for Jira tickets on unlinked commits and generate missing release notes
---

# Audit Release

<objective>
Given a release audit result JSON (produced by `go run cmd/audit/main.go`), enrich commits with Jira links and release notes by processing each component via an isolated subagent, writing results back to the JSON incrementally.

Skip all `dependabot[bot]` commits for both Jira search and release note generation.
</objective>

<process>
<step name="setup">
**Source environment and assess work.**

```bash
source .env
```

Verify `JIRA_URL`, `JIRA_EMAIL`, `JIRA_TOKEN` are set.

Read the audit result JSON. The user provides the path as an argument, or default to the most recent `reports/release_*.json` in the current directory. Store the absolute path as `$JSON_FILE`.

Check `skill_progress` in the JSON:
- If `skill_progress.report_generated == true`: announce audit is already complete and exit.
- Otherwise, determine which components still need processing (not in `skill_progress.processed_components`).

Filter out components with `error` set or no `commits`. Mark those as processed immediately (append to `skill_progress.processed_components`, write JSON).

Sort remaining unprocessed components by commit count (smallest first for quick progress). Also skip components where all commits are from `dependabot[bot]` — mark those as processed too.

For each remaining component, report name and commit count. Announce total work remaining.
</step>

<step name="process_components">
**Process each component via an isolated subagent.**

For each unprocessed component (sequentially, NOT in parallel -- they share the JSON file):

1. **Launch a subagent** using the Agent tool:

   ```
   Process component "{name}" (array index {i} in the components array) for the release audit.

   Read the instructions at {cwd}/.claude/skills/audit-release/PROCESS_COMPONENT.md first, then execute all steps.

   JSON file: {absolute_json_path}
   Component name: {name}
   Component array index: {i}
   
   IMPORTANT: The index is the position in the full components array, NOT a counter of unprocessed components. Verify the component name matches before modifying.
   ```

2. **After the subagent completes**, briefly report what it found (commits linked, release notes generated).

3. Proceed to the next component.

After all components are processed, continue to the next step.
</step>

<step name="match_unmatched_jiras">
**Cross-component: match unmatched_jiras to commits.**

Read the JSON. If `unmatched_jiras` is empty, skip this step.

For each unmatched Jira ticket, use the `summary` field already in the JSON (do NOT fetch from Jira). Scan all components' commits for keyword matches in commit messages. If a plausible match exists, update the commit's `jira` and set `jira_source` to `"unmatched-jira"`.

Write the updated JSON back using a Python script.
</step>

<step name="generate_report">
**Generate the markdown report from the enriched JSON.**

Read the complete JSON. Generate the report organized per component.

**Index and collapsible sections:** Add a table of contents after the summary stats linking to each component. Wrap each component in an HTML `<details>` tag so large sections are collapsible. Use a one-line summary showing commit count and Jira coverage.

```markdown
# Release Audit: {version}

**Source:** `{json_file_path}`<br>
**Total commits:** N across M components<br>
**Already linked (by Go tool):** N · **Newly linked:** N · **Unlinked:** N · **Dependabot (skipped):** N<br>

## Index

| Component | Commits | Jira | Unsynced | Unmatched Jiras |
|-----------|---------|------|----------|-----------------|
| [component-name](#component-name) | 25 | 20/25 | 3 | 5 |
| [another-comp](#another-comp) | 10 | 8/10 | 0 | 0 |

---

<details>
<summary><h2>component-name</h2> — 25 commits, 20 linked, 5 dependabot</summary>

**Upstream:** {upstream} · **Commits:** N · **Unsynced:** N · **Dependabot:** N

| SHA | Message | Author | PR | Jira | Release Note |
|-----|---------|--------|----|------|--------------|
| abc123 | fix: something | Name | [#100](orig-url) → [#123](pr-url) | [SRVKP-1234](url) | **Type:** Bug Fix<br>**Text:** Fixes panic on nil context. *(Jira)* |
| def456 | feat: new thing | Name | [#123](pr-url) | [SRVKP-5678](url) *(PR body)* | **Type:** Feature<br>**Text:** Adds custom CA bundles. *(extracted)* |

</details>
```

**Important:** Place a blank line after `<summary>` and before `</details>` so markdown inside renders correctly. The `<h2>` inside `<summary>` creates a clickable heading. Use the component name as the anchor (lowercase, hyphens).

**Skip dependabot commits in the table entirely.** Do not list them as rows. Only count them in the summary metrics (header stats, index, and cross-component summary table).

**Determine annotations from source fields:**
- **Jira column:**
  - `jira_source` empty + `jira` non-empty -> Go tool found it, no annotation
  - `jira_source == "pr-body"` -> *(PR body)*
  - `jira_source == "keyword"` -> *(keyword)*
  - `jira_source == "unmatched-jira"` -> *(unmatched)*
  - `jira` empty -> `—`
- **Release Note column:**
  - `rn_source` empty + `release_note` non-null -> *(Jira)*
  - `rn_source == "extracted"` -> *(extracted)*
  - `rn_source == "generated"` -> *(generated)*
  - No release note -> `—`
- **PR column:** `[#orig](url) → [#cherry](url)` when both exist and differ; just `[#N](url)` otherwise; `—` when missing.

If the component has `unsynced_commits`, show them in a separate "### Unsynced Commits" subsection with the same table format (same columns, same annotation rules, same dependabot filtering). These are upstream commits not yet cherry-picked to downstream.

**Unmatched Jiras with component labels:**
If `unmatched_jiras` contains tickets with a `components` field, map them to release components using this table:

| Jira Component | Release Component |
|----------------|-------------------|
| Tekton Pipelines | tektoncd-pipeline |
| Tekton Triggers | tektoncd-triggers |
| Tekton Chains | tektoncd-chains |
| Tekton CLI | tektoncd-cli |
| Tekton Results | tektoncd-results |
| Tekton Hub | tektoncd-hub |
| Pipelines as Code / pac | pipelines-as-code |
| Operator | operator |
| Pruner | tektoncd-pruner |
| Manual Approval | manual-approval-gate |
| Tekton Cache / cache | tekton-caches |
| kueue | tekton-kueue |
| OpenShift OPC | opc |
| UI | console-plugin |

For tickets that map to a component, show them in a "### Unmatched Jiras" subsection under that component's section:

```markdown
### Unmatched Jiras
| Key | Summary | Status |
|-----|---------|--------|
| [SRVKP-1234](url) | Fix something | Closed |
```

Tickets with no component label or no matching component go in a top-level "## Unmatched Jiras" section at the end (before Summary).

End with a cross-component summary table:

| Component | Commits | Linked (tool) | Linked (audit) | Unlinked | Has RN | Generated RN | Dependabot |
|-----------|---------|---------------|----------------|----------|--------|--------------|------------|

Write the report to `reports/audit-release-{version}.md`.

Update JSON: set `skill_progress.report_generated = true`, write back.
</step>
</process>

<success_criteria>
- [ ] Each component processed in an isolated subagent (context does not accumulate across components)
- [ ] JSON augmented after each step with `jira_source` and `rn_source` provenance fields
- [ ] No duplicate API calls for same PR or Jira ticket within a component
- [ ] Report written to `reports/audit-release-{version}.md`
</success_criteria>
