# Story {{epic_num}}.{{story_num}}: {{story_title}}

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Traceability (R3)

- Epic: {{epic_num}} → Capability: {{capability}} → GOAL. Flag any divergence/orphan.

## Definition of Done — process rules (R1/R2/R4/R5)

- Integration/E2E tests hit REAL infrastructure (named) + infra-precheck; **no mocks** (in-memory fakes / in-process stubs / monkeypatched downstreams forbidden in integration tests).
- If required infrastructure is unwired: blocked on the **Infrastructure Epic**, not mocked.
- **QA adversarial real-app run** (real app, real infra) passes: mock audit of integration tests + break-the-journey attempt. Story is not done until QA passes.

## Story

As a {{role}},
I want {{action}},
so that {{benefit}}.

## Acceptance Criteria

1. [Add acceptance criteria from epics/PRD]

## Tasks / Subtasks

- [ ] Task 1 (AC: #)
  - [ ] Subtask 1.1
- [ ] Task 2 (AC: #)
  - [ ] Subtask 2.1

## Dev Notes

- Relevant architecture patterns and constraints
- Source tree components to touch
- Testing standards summary

### Project Structure Notes

- Alignment with unified project structure (paths, modules, naming)
- Detected conflicts or variances (with rationale)

### References

- Cite all technical details with source paths and sections, e.g. [Source: docs/<file>.md#Section]

## Dev Agent Record

### Agent Model Used

{{agent_model_name_version}}

### Debug Log References

### Completion Notes List

### File List
