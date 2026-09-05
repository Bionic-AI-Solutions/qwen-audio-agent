# Task {{task_num}} of {{total_tasks}}: {{task_title}}

**Story:** [{{story_key}}](../../{{story_key}}.md)
**Status:** ready-for-task
**Stall Profile:** {{stall_profile}}
**Generated:** {{date}}

<!-- Status values (managed by dev-tasks orchestrator):
  ready-for-task  — generated, not yet started
  in-dev          — Phase 1 (implement) sub-agent running
  in-dev-complete — implementation done, awaiting review
  in-review       — Phase 2 (self-review) sub-agent running
  in-test         — Phase 3 (test) sub-agent running
  passed          — all 3 phases complete
  failed          — halted, requires human intervention
-->
<!-- Stall Profile values (controls stall detection sensitivity):
  file-heavy  — (default) task primarily writes/modifies files; standard thresholds
  api-heavy   — task makes MCP/API calls with long round-trips; extended thresholds
  mixed       — both file I/O and API calls; moderate thresholds
-->

---

## What This Task Does

{{task_description}}

Satisfies **Acceptance Criteria: {{ac_refs}}**

---

## Dependency Order

**Requires:** {{dependency_on}}

**Produces for next task:** {{produces_for_next}}

> If this is Task 1, there are no dependencies. Start here.

---

## Exact Files to Touch

| Action | File Path | What to do |
| ------ | --------- | ---------- |

{{files_table}}

**HARD LIMIT: 3 files maximum. If you discover you need to touch more files, HALT and ask the user.**

---

## Implementation Instructions

{{implementation_instructions}}

### Subtasks (complete in order)

{{subtasks_checklist}}

---

## Embedded Architecture Context

The following is the relevant excerpt from the project architecture document. This is the ONLY architecture context you need for this task. **Do not load the full architecture document.**

```
{{architecture_excerpt}}
```

---

## Embedded Code Patterns

The following patterns are already established in this codebase. Follow them exactly — do not invent new patterns.

{{code_patterns}}

### Existing File State (Current)

The following shows the current state of files you will modify. Use these as your baseline — do not assume file contents.

{{existing_file_excerpts}}

---

## Test Requirements

{{test_requirements}}

> **Rule R1:** if this task includes an integration/E2E test, it MUST hit the project's REAL infrastructure (name it) with an infra-precheck. **No mocks** in integration tests (in-memory fakes, in-process servers with stubbed downstreams, monkeypatched services). Unit tests may mock. **R2:** if the real infrastructure isn't wired, do NOT mock — mark blocked on the Infrastructure Epic.

### Verification Commands

Run these exact commands to verify this task is complete. All must exit 0.

```bash
{{verification_commands}}
```

---

## DO NOT

{{do_not_list}}

**Always applies:**

- Do NOT modify files not listed in "Exact Files to Touch"
- Do NOT install new dependencies without halting and asking the user
- Do NOT refactor code outside the scope of this task
- Do NOT mark this task complete if any verification command fails
- Do NOT implement anything beyond what this task describes

---

## Completion Checklist

Before marking this task done, verify ALL of the following:

- [ ] All subtasks above are checked
- [ ] All verification commands pass with exit 0
- [ ] Only the listed files were modified
- [ ] No new dependencies were added outside story specifications
- [ ] Implementation matches the task description exactly (no extra features)

---

## Task Agent Record

### Completion Notes

_(Fill in after implementation — summarize what was built and tested)_

### Files Modified

_(List actual files touched — should match "Exact Files to Touch" above)_

### Deviations from Plan

_(Document any necessary deviations and why — keep empty if you followed the plan exactly)_
