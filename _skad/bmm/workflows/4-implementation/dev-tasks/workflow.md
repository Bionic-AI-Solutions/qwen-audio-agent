---
name: dev-tasks
description: 'Orchestrate task-by-task story implementation with automated implement → review → test pipeline. Use when the user says "dev tasks", "run dev tasks", or "implement tasks for [story]"'
---

# Dev Tasks Workflow

**Goal:** Orchestrate atomic task-level implementation across tasks, stories, and epics using a 3-phase pipeline per task: Implement → Review → Test. Advance automatically through tasks and stories; halt at configurable checkpoints for human sign-off.

**Your Role:** Orchestrator. You do not write production code directly. You spawn and monitor sub-agents, read state from task files and sprint-status, and advance the pipeline.

- Communicate all responses in {communication_language} tailored to {user_skill_level}
- Execute ALL steps in exact order; do NOT skip steps
- NEVER advance to the next task until the current task has `passed` status
- NEVER modify test assertions to make a test pass — only production code is fixed
- Read state from files at the start of every iteration — never assume state from memory

## MANDATORY PROCESS RULES (override defaults)

- **R1 — No integration test with mocks.** Integration/E2E tests MUST hit the project's REAL infrastructure (its actual databases, auth, message/streaming, object store, external/AI services). The Test phase REJECTS any integration test that uses in-memory fakes, an in-process server with stubbed downstreams, monkeypatched services, or fake databases — that is a defect, not a pass. (Unit tests may mock; integration tests may not.)
- **R2 — Infra gap → Infrastructure Epic.** If a real-infra test cannot run because infrastructure isn't wired, do NOT mock and do NOT mark the task/story `passed` — HALT, record the gap against an **Infrastructure Epic**, and leave the story blocked.
- **R3 — Traceability.** Every task links to its story, story to epic, epic to capability, capability to the product GOAL. Flag any orphan or unwired connection in the run output.
- **R4/R5 — QA adversarial real-app gate.** A story/epic is NOT complete until the **Phase 4 QA adversarial verification** (below) passes: the adversarial QA role drives the REAL application (browser / agent-browser / Playwright) on real infrastructure, audits the story's integration tests for mocks, and tries to break the user journey. Spawn the QA agent (`bmm/agents/qa`).
- **R6 — Self-initiated QA, before commit/merge/done.** Trigger the Phase 4 QA pass YOURSELF (never wait to be asked) whenever a task/story adds or changes tests/verification and before opening a PR or marking done. QA must FETCH/drive the real artifact (not trust a 200 / truthy field) — a check that does not exercise the artifact is a false-green defect, fixed in the same cycle.

---

## INITIALIZATION

### Configuration Loading

Load config from `{project-root}/_skad/bmm/config.yaml` and resolve:

- `project_name`, `user_name`
- `communication_language`, `document_output_language`
- `user_skill_level`
- `implementation_artifacts`
- `date` as system-generated current datetime

### Paths

- `installed_path` = `{project-root}/_skad/bmm/workflows/4-implementation/dev-tasks`
- `sprint_status` = `{implementation_artifacts}/sprint-status.yaml`
- `story_path` = `` (explicit story path; auto-discovered if empty)

### OpenProject Integration

Read `openproject_id` from config (may be null if OP not configured).

- `op_sync_workflow` = `{project-root}/_skad/bmm/workflows/4-implementation/openproject-sync/workflow.md`
- `op_map_file` = `{project-root}/_skad/bmm/openproject-map.yaml`
- `op_enabled` = true if `openproject_id` is present AND `op_map_file` exists, else false

If `op_enabled` is false: all OP sync steps in this workflow are silently skipped.

### Autonomy Mode

Read `autonomy_mode` from config or user invocation argument. Valid values:

| Mode               | Behavior                                                                                            |
| ------------------ | --------------------------------------------------------------------------------------------------- |
| `implement-only`   | Run Phase 1 (implement) only per task; no auto review/test                                          |
| `halt-after-story` | **(Default)** Pause after all tasks in a story are passed, awaiting human approval before advancing |
| `halt-on-high`     | Pause only when Phase 2 review finds a High-severity issue                                          |
| `full-hands-off`   | Run to completion (or failure) without pausing                                                      |

If `autonomy_mode` is not set, default to `halt-after-story`.

### Stall Detection Settings

Base thresholds (adjusted per task `Stall Profile`):

| Stall Profile          | `stall_warn` | `stall_kill` | Use Case                                                                      |
| ---------------------- | ------------ | ------------ | ----------------------------------------------------------------------------- |
| `file-heavy` (default) | 10 min       | 20 min       | Code writing, unit tests — frequent file writes expected                      |
| `api-heavy`            | 20 min       | 40 min       | MCP calls, API integrations, infra validation — long periods without file I/O |
| `mixed`                | 15 min       | 30 min       | Both file writes and API calls                                                |

Config overrides (`stall_warn_minutes`, `stall_kill_minutes`) take precedence over profile defaults.

### Multi-Signal Activity Detection

A sub-agent is considered **alive** if ANY of these signals are true:

| Signal                      | What It Detects                                    | How to Check                                                   |
| --------------------------- | -------------------------------------------------- | -------------------------------------------------------------- |
| **TaskOutput growth**       | Agent produced new output (tool results, text)     | `TaskOutput(agent_id)` length increased since last poll        |
| **Active child processes**  | Agent is running tools (curl, python3, node, etc.) | `pgrep -P <agent_pid> -la` returns active children             |
| **Network socket activity** | Agent is making API/MCP calls                      | `ss -tnp \| grep -c <agent_pid>` shows established connections |

The orchestrator only declares a stall when **ALL signals are negative** for the full `stall_kill` duration.

Override base thresholds via config or invocation args if needed.

---

## EXECUTION

<workflow>
  <critical>NEVER advance to the next task until the current task has Status = passed in its task file</critical>
  <critical>NEVER modify test assertions or test goals to make a test pass — only fix production code</critical>
  <critical>Re-read task file Status at the start of every phase — never assume state from memory</critical>
  <critical>If a sub-agent stalls: git stash uncommitted changes, kill sub-agent, restart with Recovery Context</critical>
  <critical>Tests are the source of truth — failing tests identify what production code must be fixed</critical>
  <critical>Execute ALL steps in exact order; do NOT skip steps</critical>

  <step n="1" goal="Find current position in sprint" tag="sprint-status">
    <check if="{{story_path}} is provided">
      <action>Read COMPLETE story file at {{story_path}}</action>
      <action>Extract story_key from filename or metadata</action>
      <action>Verify story Dev Notes contains a '### Task Files' subsection — HALT if not (run create-tasks first)</action>
      <goto anchor="task_discovery" />
    </check>

    <check if="{{sprint_status}} file exists">
      <critical>Read COMPLETE sprint-status.yaml from start to end to preserve order</critical>
      <action>Load FULL file: {{sprint_status}}</action>
      <action>Parse development_status section completely</action>

      <!-- Look for in-progress story first (resume case) -->
      <action>Find FIRST entry matching pattern number-number-name where status = "in-progress"</action>
      <check if="in-progress story found">
        <action>Use that story as {{story_key}} — this is a resume</action>
        <action>Read story file to check for current_task comment (format: # current_task: task-N-slug)</action>
        <output>⏯️ Resuming story {{story_key}}</output>
        <goto anchor="task_discovery" />
      </check>

      <!-- Else find next ready-for-dev story -->
      <action>Find FIRST entry matching pattern number-number-name where status = "ready-for-dev"</action>
      <check if="ready-for-dev story found">
        <action>Use that story as {{story_key}}</action>
        <goto anchor="task_discovery" />
      </check>

      <check if="no in-progress or ready-for-dev story found">
        <!-- Check if there's a backlog story we can auto-create -->
        <action>Find FIRST entry matching pattern number-number-name where status = "backlog"</action>
        <check if="backlog story found">
          <output>📋 No ready-for-dev stories, but found backlog story {{backlog_story_key}}. Auto-creating story file and task files...</output>
          <action>Read fully and follow: `{project-root}/_skad/bmm/workflows/4-implementation/create-story/workflow.md`</action>
          <note>create-story will auto-discover the first backlog story from sprint-status.yaml, generate the comprehensive story file, and auto-chain into create-tasks to produce self-contained atomic task files. After completion, dev-tasks resumes from step 1.</note>
          <goto step="1">Re-discover now that story and tasks exist</goto>
        </check>
        <check if="no backlog story found either">
          <output>📋 No actionable stories found in sprint-status.yaml — all stories are in-progress, review, or done.

            **Options:**
            1. Specify a story file path directly
            2. Check sprint-status for current state
          </output>
          <ask>Provide a story file path or check sprint-status:</ask>
          <action>Handle user response and set {{story_path}} or HALT as appropriate</action>
        </check>
      </check>
    </check>

    <check if="{{sprint_status}} file does NOT exist">
      <ask>No sprint-status.yaml found. Please provide the full path to the story file to implement:</ask>
      <action>Store user-provided path as {{story_path}}</action>
      <goto anchor="task_discovery" />
    </check>

    <anchor id="task_discovery" />
    <action>Find story file in {implementation_artifacts} using pattern: {{story_key}}.md</action>
    <action>Read COMPLETE story file</action>
    <action>Verify Dev Notes contains '### Task Files' subsection</action>
    <check if="'### Task Files' subsection is missing">
      <output>⚠️ Story {{story_key}} has no task files. Auto-generating self-contained atomic task files...</output>
      <action>Set {{story_path}} = path to current story file</action>
      <action>Read fully and follow: `{project-root}/_skad/bmm/workflows/4-implementation/create-tasks/workflow.md`</action>
      <note>create-tasks will atomize the story into self-contained task files with all architecture context, code patterns, and verification commands inlined. Sub-agents can then execute each task with zero starting context.</note>
      <action>After create-tasks completes, re-read the story file to pick up the newly added Task Files subsection</action>
    </check>

    <action>Extract all task file paths from markdown links in the '### Task Files' subsection</action>
    <action>For each task file: read its Status field</action>
    <action>Find FIRST task file where Status is NOT 'passed'</action>
    <action>Set {{current_task_file}} = path to that task file</action>
    <action>Set {{total_tasks}} = total count of task files listed</action>
    <action>Set {{passed_tasks}} = count of task files with Status = 'passed'</action>

    <check if="ALL task files have Status = 'passed'">
      <goto step="7">Story complete — run story boundary sequence</goto>
    </check>

    <output>📋 **Dev Tasks: {{story_key}}**
      Progress: {{passed_tasks}} / {{total_tasks}} tasks passed
      Next task: {{current_task_file}}
      Autonomy mode: {{autonomy_mode}}
    </output>

  </step>

  <step n="2" goal="Update sprint status to in-progress" tag="sprint-status">
    <check if="{{sprint_status}} file exists">
      <action>Load FULL file: {{sprint_status}}</action>
      <action>Find development_status[{{story_key}}]</action>
      <check if="current status == 'ready-for-dev'">
        <action>Update status to "in-progress"</action>
        <action>Add comment on same line: # current_task: {{current_task_file_basename}}</action>
        <action>Update last_updated to current date</action>
        <output>🚀 Story {{story_key}}: ready-for-dev → in-progress</output>
      </check>
      <check if="current status == 'in-progress'">
        <action>Update the current_task comment to reflect {{current_task_file_basename}}</action>
        <output>⏯️ Resuming story {{story_key}} at task {{current_task_file_basename}}</output>
      </check>
    </check>
  </step>

  <step n="2b" goal="OpenProject: Bootstrap Task WPs for current story" tag="op-sync">
    <check if="{{op_enabled}} == true">
      <action>Read COMPLETE {{op_sync_workflow}}</action>
      <action>Execute ACTION: bootstrap-tasks with:
        - story_key = {{story_key}}
        - project_root = {project-root}
      </action>
      <critical>OP sync failures must NEVER halt the dev-tasks pipeline. If the sync workflow returns a warning, log it and continue.</critical>
    </check>
  </step>

  <step n="3" goal="Dependency check for current task">
    <action>Read COMPLETE task file: {{current_task_file}}</action>
    <action>Extract 'Requires:' field from Dependency Order section</action>
    <check if="Requires field lists one or more task names">
      <action>For each required task: read its task file and check Status field</action>
      <check if="any required task does NOT have Status = 'passed'">
        <output>🚫 Dependency violation: {{current_task_file}} requires {{unmet_dependency}} which is not yet passed.

          This indicates a task ordering issue in the task files. Tasks must be ordered so that no task depends on a future (uncompleted) task.

          **Action required:** Review the Task Files list in the story Dev Notes and ensure tasks are ordered correctly. Run `create-tasks` again if the ordering is wrong.
        </output>
        <action>HALT</action>
      </check>
    </check>
    <output>✅ Dependencies satisfied for {{current_task_file_basename}}</output>

  </step>

  <step n="4" goal="Phase 1 — Implement" tag="implement">
    <critical>Fix code to meet tests — NEVER modify test assertions to force a pass</critical>

    <action>Read current task file Status field</action>
    <check if="task Status == 'passed'">
      <action>Skip to step 6 (task already complete)</action>
    </check>

    <action>Update task file Status field: → `in-dev`</action>
    <output>🔨 **Phase 1: Implement** — {{current_task_file_basename}}
      Spawning implementation sub-agent...
    </output>

    <action>Spawn background sub-agent with the following prompt:
      ```
      You are implementing a single atomic task. Load and follow the task file EXACTLY:
      Task file: {{current_task_file}}

      Instructions:
      - This task file is your ONLY source of truth. Follow it completely.
      - Use ONLY the Embedded Architecture Context in the task file — do not load external architecture docs.
      - Modify ONLY the files listed in 'Exact Files to Touch'.
      - Follow all 'DO NOT' prohibitions without exception.
      - Run the Verification Commands at the end. All must exit 0.
      - If a verification command fails: fix PRODUCTION CODE ONLY. NEVER alter test assertions or test goals.
      - When ALL Completion Checklist items are satisfied AND all Verification Commands pass: update the task file Status → 'in-dev-complete' and fill in Task Agent Record.
      - HALT if: new dependencies are needed, 3 consecutive implementation failures occur, or required config is missing.
      ```
    </action>
    <action>Store sub-agent ID as {{implement_agent_id}}</action>

    <!-- Resolve stall thresholds from task Stall Profile -->
    <action>Read "Stall Profile:" field from current task file (default: "file-heavy" if absent)</action>
    <action>Set {{effective_stall_warn}} and {{effective_stall_kill}} based on profile:
      - file-heavy: stall_warn=10, stall_kill=20
      - api-heavy:  stall_warn=20, stall_kill=40
      - mixed:      stall_warn=15, stall_kill=30
      If config overrides (stall_warn_minutes / stall_kill_minutes) are set, use those instead.
    </action>

    <!-- Multi-signal stall monitoring loop -->
    <action>Set {{last_output_len}} = 0, {{stall_count}} = 0</action>
    <loop until="sub-agent completes OR stall confirmed">
      <action>Wait {{effective_stall_warn}} minutes</action>

      <!-- Signal 1: TaskOutput growth -->
      <action>Read TaskOutput({{implement_agent_id}})</action>
      <action>Set {{current_output_len}} = length of output so far</action>
      <action>Set {{output_grew}} = ({{current_output_len}} > {{last_output_len}})</action>

      <!-- Signal 2: Active child processes (curl, python3, node, etc.) -->
      <action>Run: pgrep -la "curl|python3|node|npm|npx|pytest|jest" 2>/dev/null | wc -l</action>
      <action>Set {{has_active_children}} = (result > 0)</action>

      <!-- Signal 3: Network socket activity -->
      <action>Run: ss -tnp 2>/dev/null | grep -c "ESTAB" || echo "0"</action>
      <action>Set {{has_network_activity}} = (result > 0)</action>

      <!-- Evaluate: agent is alive if ANY signal is positive -->
      <check if="{{output_grew}} OR {{has_active_children}} OR {{has_network_activity}}">
        <action>Set {{last_output_len}} = {{current_output_len}}, {{stall_count}} = 0</action>
        <check if="NOT {{output_grew}} AND ({{has_active_children}} OR {{has_network_activity}})">
          <output>⏳ No output growth but agent is active (child processes: {{has_active_children}}, network: {{has_network_activity}}). Stall timer reset.</output>
        </check>
      </check>

      <!-- All signals negative — possible stall -->
      <check if="NOT {{output_grew}} AND NOT {{has_active_children}} AND NOT {{has_network_activity}}">
        <action>Increment {{stall_count}}</action>
        <output>⏳ No activity detected on any signal ({{stall_count * effective_stall_warn}} min). Monitoring for stall...</output>
        <check if="{{stall_count}} * {{effective_stall_warn}} >= {{effective_stall_kill}}">
          <action>TaskStop({{implement_agent_id}})</action>
          <output>🔴 Implementation sub-agent stalled (all signals negative for {{effective_stall_kill}} min). Initiating recovery...</output>
          <goto anchor="implement_recovery" />
        </check>
      </check>
    </loop>

    <!-- Recovery on stall -->
    <anchor id="implement_recovery" />
    <action>Run: git status (check for uncommitted partial changes from failed agent)</action>
    <action>If uncommitted changes exist: run git stash with message "dev-tasks recovery: stalled on {{current_task_file_basename}}"</action>
    <action>Set {{retry_count}} = ({{retry_count}} or 0) + 1</action>
    <check if="{{retry_count}} > 2">
      <output>🚫 Implementation of {{current_task_file_basename}} failed after 2 retries. Manual intervention required.
        Update task file Status → 'failed' and investigate before re-running dev-tasks.
      </output>
      <action>Update task file Status → 'failed'</action>
      <action>HALT</action>
    </check>

    <action>Spawn new sub-agent with the same prompt PLUS a Recovery Context section:
      ```
      [Recovery Context]
      Previous agent stalled or failed (attempt {{retry_count}} of 2).
      Stashed changes message: "dev-tasks recovery: stalled on {{current_task_file_basename}}"
      Check git stash list. If stash exists, review stashed diff before deciding whether to pop or discard.
      Start fresh from the beginning of the task file unless the stashed changes are correct and complete.
      ```
    </action>
    <action>Store new sub-agent ID as {{implement_agent_id}}</action>
    <goto anchor="implement_recovery" />
    <!-- loop back into monitoring -->

    <!-- Success path -->
    <check if="sub-agent completed successfully AND task file Status == 'in-dev-complete'">
      <action>Update task file Status → `in-review`</action>
      <output>✅ Phase 1 complete: {{current_task_file_basename}} → in-review</output>
    </check>

    <check if="autonomy_mode == 'implement-only'">
      <output>ℹ️ Autonomy mode is 'implement-only'. Stopping after implementation phase.
        Run dev-tasks again to continue with review and test phases.
      </output>
      <action>HALT</action>
    </check>

  </step>

  <step n="5" goal="Phase 2 — Lightweight Self-Review" tag="review">
    <action>Read current task file Status field</action>
    <check if="task Status != 'in-review'">
      <output>⚠️ Expected task Status = 'in-review', found '{{task_status}}'. Skipping review phase.</output>
      <goto step="6" />
    </check>

    <output>🔍 **Phase 2: Review** — {{current_task_file_basename}}
      Spawning self-review sub-agent...
    </output>

    <action>Spawn background sub-agent with the following prompt:
      ```
      You are performing a focused self-review of a just-implemented task. Your job is to verify correctness and quality.

      Task file: {{current_task_file}}

      Review against the task file's Completion Checklist:
      - Are ALL checklist items satisfied? (verify each one independently)
      - Do the modified files match ONLY the 'Exact Files to Touch' list?
      - Does the implementation satisfy the stated acceptance criteria references?
      - Are there any DO NOT violations?
      - Is there any dead code, debug output, or commented-out test logic?

      For each finding, classify severity:
        - High: correctness bug, AC not met, test integrity violation (test modified to pass instead of fixing code)
        - Medium: code quality, missing edge case, minor deviation from task spec
        - Low: style, naming, comment quality

      Output format:
        REVIEW RESULT: PASS | PASS-WITH-FIXES | FAIL
        Findings:
          [High] <description> — <file:line>
          [Medium] <description>
          [Low] <description>

      If PASS-WITH-FIXES: fix all Medium/Low issues in production code now, then output REVIEW RESULT: PASS.
      If FAIL (any High finding): output REVIEW RESULT: FAIL with full details. Do NOT attempt to fix.
      NEVER alter test assertions to resolve a finding — only fix production code.
      ```
    </action>
    <action>Store sub-agent ID as {{review_agent_id}}</action>

    <!-- Multi-signal stall monitoring (same pattern as Phase 1) -->
    <action>Set {{last_output_len}} = 0, {{stall_count}} = 0</action>
    <loop until="sub-agent completes OR stall confirmed">
      <action>Wait {{effective_stall_warn}} minutes</action>
      <action>Read TaskOutput({{review_agent_id}})</action>
      <action>Set {{current_output_len}} = length of output</action>
      <action>Set {{output_grew}} = ({{current_output_len}} > {{last_output_len}})</action>
      <action>Check child processes: pgrep -la "curl|python3|node" 2>/dev/null | wc -l</action>
      <action>Check network: ss -tnp 2>/dev/null | grep -c "ESTAB" || echo "0"</action>

      <check if="any signal positive (output grew OR active children OR network)">
        <action>Set {{last_output_len}} = {{current_output_len}}, {{stall_count}} = 0</action>
      </check>
      <check if="all signals negative">
        <action>Increment {{stall_count}}</action>
        <check if="{{stall_count}} * {{effective_stall_warn}} >= {{effective_stall_kill}}">
          <action>TaskStop({{review_agent_id}})</action>
          <output>⚠️ Review sub-agent stalled (all signals negative). Re-spawning...</output>
          <action>Spawn fresh review sub-agent with same prompt</action>
          <action>Update {{review_agent_id}}, reset {{stall_count}} = 0</action>
        </check>
      </check>
    </loop>

    <action>Read review output for REVIEW RESULT</action>

    <check if="REVIEW RESULT == 'PASS'">
      <action>Update task file Status → `in-test`</action>
      <output>✅ Phase 2 complete: {{current_task_file_basename}} → in-test</output>
    </check>

    <check if="REVIEW RESULT == 'FAIL' (High findings present)">
      <check if="autonomy_mode == 'halt-after-story' OR autonomy_mode == 'halt-on-high'">
        <output>🛑 **Review FAILED — High-severity findings require human review**

          Task: {{current_task_file_basename}}
          {{review_findings}}

          Fix the production code issues listed above, then re-run dev-tasks to continue.
          (Remember: NEVER modify test assertions — fix only production code.)
        </output>
        <action>Update task file Status → 'failed'</action>
        <action>HALT</action>
      </check>
      <check if="autonomy_mode == 'full-hands-off'">
        <output>⚠️ High-severity findings detected. Attempting auto-fix in full-hands-off mode...</output>
        <action>Spawn fix sub-agent scoped to High findings — production code only, never tests</action>
        <action>Monitor fix sub-agent with stall detection</action>
        <action>After fix: re-run review sub-agent (one retry only)</action>
        <check if="second review still fails">
          <action>Update task file Status → 'failed'</action>
          <output>🚫 Auto-fix failed. Manual intervention required for {{current_task_file_basename}}.</output>
          <action>HALT</action>
        </check>
      </check>
    </check>

  </step>

  <step n="6" goal="Phase 3 — Test" tag="test">
    <critical>NEVER modify test assertions or test goals. If tests fail, fix production code only.</critical>

    <action>Read current task file Status field</action>
    <check if="task Status != 'in-test'">
      <output>⚠️ Expected task Status = 'in-test', found '{{task_status}}'. Skipping test phase.</output>
      <goto step="8" />
    </check>

    <output>🧪 **Phase 3: Test** — {{current_task_file_basename}}
      Spawning test sub-agent...
    </output>

    <action>Set {{test_retry_count}} = 0</action>

    <anchor id="test_run" />
    <action>Spawn background sub-agent with the following prompt:
      ```
      You are the test verification agent for a completed task.

      Task file: {{current_task_file}}

      Your job:
      1. Read the 'Verification Commands' section from the task file.
      2. Run EVERY verification command exactly as written.
      3. For each command that exits non-zero:
         - Diagnose the failure.
         - Fix ONLY production code (source files). NEVER alter test files, test assertions, or test goals.
         - Re-run the command after fixing.
      4. If you cannot fix a failure in production code without also changing the test, output:
         TEST-INTEGRITY-HALT: <description of the conflict>
         Then STOP — do not modify any test.
      5. When ALL commands exit 0: output VERIFICATION: PASS
      6. Run the full regression suite to confirm no regressions. If regressions found: fix production code. Output REGRESSION: PASS when clean.

      Output FINAL STATUS: PASSED only when both VERIFICATION: PASS and REGRESSION: PASS.
      ```
    </action>
    <action>Store sub-agent ID as {{test_agent_id}}</action>

    <!-- Multi-signal stall monitoring -->
    <action>Set {{last_output_len}} = 0, {{stall_count}} = 0</action>
    <loop until="sub-agent completes OR stall confirmed">
      <action>Wait {{effective_stall_warn}} minutes</action>
      <action>Read TaskOutput({{test_agent_id}})</action>
      <action>Set {{current_output_len}} = length of output</action>
      <action>Set {{output_grew}} = ({{current_output_len}} > {{last_output_len}})</action>
      <action>Check child processes: pgrep -la "curl|python3|node|pytest|jest" 2>/dev/null | wc -l</action>
      <action>Check network: ss -tnp 2>/dev/null | grep -c "ESTAB" || echo "0"</action>

      <check if="any signal positive (output grew OR active children OR network)">
        <action>Set {{last_output_len}} = {{current_output_len}}, {{stall_count}} = 0</action>
      </check>
      <check if="all signals negative">
        <action>Increment {{stall_count}}</action>
        <check if="{{stall_count}} * {{effective_stall_warn}} >= {{effective_stall_kill}}">
          <action>TaskStop({{test_agent_id}})</action>
          <action>git stash if uncommitted changes exist</action>
          <action>Set {{test_retry_count}} = {{test_retry_count}} + 1</action>
          <check if="{{test_retry_count}} > 2">
            <action>Update task file Status → 'failed'</action>
            <output>🚫 Test phase stalled after 2 retries (all signals negative). Manual intervention required.</output>
            <action>HALT</action>
          </check>
          <action>Spawn fresh test sub-agent</action>
          <goto anchor="test_run" />
        </check>
      </check>
    </loop>

    <check if="sub-agent output contains 'TEST-INTEGRITY-HALT'">
      <output>🛑 **Test integrity conflict detected in {{current_task_file_basename}}**

        The test sub-agent flagged a case where fixing the failure would require modifying a test assertion.
        This indicates either:
        a) The implementation fundamentally misunderstands the acceptance criterion, OR
        b) The test was written incorrectly in create-tasks (rare)

        {{test_integrity_details}}

        **Human review required.** Do not modify tests — reassess the implementation approach.
      </output>
      <action>Update task file Status → 'failed'</action>
      <action>HALT</action>
    </check>

    <check if="sub-agent output contains 'FINAL STATUS: PASSED'">
      <action>Update task file Status → `passed`</action>
      <action>Mark corresponding checkbox [x] in story file Tasks/Subtasks section</action>
      <action>Update story File List with any new/modified files from this task</action>
      <output>✅ **Task PASSED: {{current_task_file_basename}}** ({{passed_tasks + 1}} / {{total_tasks}})</output>

      <!-- OpenProject: sync task status -->
      <check if="{{op_enabled}} == true">
        <action>Read COMPLETE {{op_sync_workflow}}</action>
        <action>Execute ACTION: task-passed with:
          - story_key = {{story_key}}
          - task_file_basename = {{current_task_file_basename}}
          - project_root = {project-root}
        </action>
        <critical>OP sync failure must NOT block the pipeline. Log warning and continue.</critical>
      </check>
    </check>

    <check if="sub-agent output contains failure and no TEST-INTEGRITY-HALT">
      <action>Set {{test_retry_count}} = {{test_retry_count}} + 1</action>
      <check if="{{test_retry_count}} > 2">
        <action>Update task file Status → 'failed'</action>
        <output>🚫 Tests failed after 2 retries. Manual intervention required for {{current_task_file_basename}}.</output>
        <action>HALT</action>
      </check>
      <output>⚠️ Test phase failed (attempt {{test_retry_count}}). Retrying...</output>
      <goto anchor="test_run" />
    </check>

  </step>

  <step n="7" goal="Advance to next task or story boundary">
    <action>Re-read ALL task files for {{story_key}} and count Status = 'passed'</action>

    <check if="remaining tasks with Status != 'passed' exist">
      <action>Set {{current_task_file}} = next task file where Status != 'passed'</action>
      <action>Update current_task comment in sprint-status.yaml</action>
      <output>➡️ Advancing to next task: {{current_task_file_basename}}</output>
      <goto step="3">Next task — dependency check</goto>
    </check>

    <check if="ALL tasks have Status = 'passed'">
      <output>🎉 All {{total_tasks}} tasks passed for story {{story_key}}!

        Running story-level adversarial code review...
      </output>

      <!-- Full story code review at boundary -->
      <action>Spawn code-review sub-agent using the installed code-review workflow:
        `{project-root}/_skad/bmm/workflows/4-implementation/code-review/workflow.md`
        Scope: story {{story_key}} — full adversarial review of all changes since story began
      </action>
      <action>Monitor code-review sub-agent with stall detection</action>
      <action>Read code-review output for outcome (Approve / Changes Requested / Blocked)</action>

      <check if="code-review outcome == 'Changes Requested' with High findings">
        <output>🛑 Story-level code review found High-severity issues. Resolving before marking story complete...</output>
        <action>Spawn fix sub-agent to address High findings (production code only)</action>
        <action>After fixes: re-run test phase for affected tasks</action>
      </check>

      <!-- Phase 4 — MANDATORY QA adversarial real-app verification + mock audit (rules R1/R4/R5) -->
      <output>🧪 **Phase 4: QA adversarial verification** — story {{story_key}}</output>
      <action critical="true">Spawn the ADVERSARIAL QA agent (`bmm/agents/qa`, command [AV]). QA MUST:
        1. MOCK AUDIT — inspect every test the story labels integration/E2E and FLAG any mock (in-memory fakes, in-process servers with stubbed downstreams, monkeypatched services/clients, fake databases). Any such mock is a DEFECT (rule R1).
        2. REAL-APP E2E — drive the REAL application (browser / agent-browser / Playwright / real HTTP) on REAL infrastructure and adversarially attempt to break the story's user journey end-to-end (do NOT merely re-run the dev's tests).
        3. INFRA GAP — if a real-infra check cannot run because infrastructure is missing, HALT and route the gap to an Infrastructure Epic (rule R2); do not accept a mock.
        4. TRACEABILITY — confirm the story traces to its epic/capability/GOAL and flag any orphan/unwired flow (rule R3).
      </action>
      <action>Monitor the QA sub-agent with stall detection; read its verdict (Pass / Defects-Found / Blocked-on-Infra).</action>
      <check if="QA verdict != 'Pass'">
        <output>🛑 QA adversarial verification FAILED for {{story_key}} — {{qa_summary}}. Story is NOT complete.</output>
        <action>If mocks were found in integration tests: spawn a fix sub-agent to re-point them at real infrastructure, then re-run the Test phase + Phase 4.</action>
        <action>If blocked on infra: record the gap against the Infrastructure Epic, set the story status to blocked, and HALT — do NOT mark complete.</action>
        <action>Do not proceed to mark the story complete until QA verdict == 'Pass'.</action>
      </check>

      <action>Update story file Status → "review"</action>
      <action>Update sprint-status.yaml: development_status[{{story_key}}] → "review"</action>
      <action>Remove current_task comment from sprint-status entry</action>
      <action>Update last_updated to current date</action>

      <!-- OpenProject: sync story-complete status -->
      <check if="{{op_enabled}} == true">
        <action>Read COMPLETE {{op_sync_workflow}}</action>
        <action>Execute ACTION: story-complete with:
          - story_key = {{story_key}}
          - project_root = {project-root}
        </action>
        <critical>OP sync failure must NOT block the pipeline. Log warning and continue.</critical>
      </check>

      <check if="autonomy_mode == 'halt-after-story' OR autonomy_mode == 'halt-on-high'">
        <output>⏸️ **Story {{story_key}} complete — awaiting human approval**

          **Summary:**
          - Tasks completed: {{total_tasks}} / {{total_tasks}}
          - Story status: review
          - Code review: {{code_review_outcome}}
          - Files changed: {{file_list_summary}}

          **Next story would be:** {{next_story_key}} (if applicable)

          Review the story, run manual tests if needed, then respond:
          - [A] Approve — continue to next story
          - [H] HALT — stop here

          Or provide specific feedback to address before continuing.
        </output>
        <ask>Approve and continue, or halt?</ask>

        <check if="user approves">
          <goto anchor="next_story" />
        </check>
        <check if="user halts or provides feedback">
          <action>Address feedback if provided, then re-ask for approval</action>
        </check>
      </check>

      <check if="autonomy_mode == 'full-hands-off'">
        <output>✅ Story {{story_key}} complete. Advancing to next story...</output>
        <goto anchor="next_story" />
      </check>

      <anchor id="next_story" />
      <action>Load FULL sprint-status.yaml</action>
      <action>Find the next story after {{story_key}} (by order in file) where status is 'ready-for-dev' or 'backlog'</action>

      <check if="next story is 'backlog' (story file not yet created)">
        <output>📋 Next story {{next_story_key}} is in backlog. Auto-creating story file and task files...</output>
        <action>Read fully and follow: `{project-root}/_skad/bmm/workflows/4-implementation/create-story/workflow.md`</action>
        <note>create-story will generate the comprehensive story file from epics.md context and auto-chain into create-tasks to produce self-contained atomic task files. After completion, dev-tasks resumes with the new story.</note>
        <action>Set {{story_key}} = {{next_story_key}}</action>
        <goto step="1">Re-discover now that story and tasks exist</goto>
      </check>

      <check if="next story is 'ready-for-dev'">
        <action>Check if next story has Task Files (### Task Files in Dev Notes)</action>
        <check if="Task Files missing">
          <output>📋 Story {{next_story_key}} is ready-for-dev but has no task files. Auto-generating...</output>
          <action>Set {{story_path}} = path to {{next_story_key}} story file</action>
          <action>Read fully and follow: `{project-root}/_skad/bmm/workflows/4-implementation/create-tasks/workflow.md`</action>
          <note>create-tasks will atomize the story into self-contained task files with all context inlined for zero-context sub-agent execution.</note>
          <action>Set {{story_key}} = {{next_story_key}}</action>
          <goto step="1">Re-discover now that tasks exist</goto>
        </check>
        <action>Set {{story_key}} = {{next_story_key}}</action>
        <output>🚀 Starting next story: {{story_key}}</output>
        <goto step="1">Find position in new story</goto>
      </check>

      <check if="no further stories are actionable in current epic">
        <action>Load sprint-status and check across ALL epics for next backlog story</action>
        <action>Find FIRST story across all epics where status = "backlog"</action>
        <check if="backlog story found in another epic">
          <output>🏁 **Current epic complete!** Moving to next epic...

            Epic progress summary:
            {{epic_summary}}

            Next story: {{next_backlog_story_key}} — auto-creating story and tasks...
          </output>
          <action>Read fully and follow: `{project-root}/_skad/bmm/workflows/4-implementation/create-story/workflow.md`</action>
          <note>create-story will auto-discover the next backlog story, generate the story file, and chain into create-tasks. After completion, dev-tasks resumes.</note>
          <goto step="1">Re-discover now that story and tasks exist</goto>
        </check>
        <check if="no backlog stories remain anywhere">
          <output>🏁 **All epics and stories are complete!**

            Epic progress summary:
            {{epic_summary}}

            No remaining backlog stories. The project implementation is complete.
          </output>
          <action>HALT</action>
        </check>
      </check>
    </check>

  </step>

</workflow>

---

## Task Status Reference

Task files use the `Status:` field to track pipeline phase. Valid values:

| Status            | Meaning                                                          |
| ----------------- | ---------------------------------------------------------------- |
| `ready-for-task`  | Task generated, not yet started                                  |
| `in-dev`          | Phase 1 (implement) sub-agent running                            |
| `in-dev-complete` | Implementation done, awaiting review phase                       |
| `in-review`       | Phase 2 (review) sub-agent running                               |
| `in-test`         | Phase 3 (test) sub-agent running                                 |
| `passed`          | All 3 phases complete — task is done                             |
| `failed`          | Halted due to unresolvable failure — requires human intervention |

---

## Stall Detection Reference

### Multi-Signal Activity Detection

The orchestrator polls sub-agent health every `effective_stall_warn` minutes using three independent signals:

| Signal                      | Command                                         | What it catches                                       |
| --------------------------- | ----------------------------------------------- | ----------------------------------------------------- |
| **TaskOutput growth**       | `TaskOutput(agent_id)` length delta             | Agent producing any output (tool calls, text, errors) |
| **Active child processes**  | `pgrep -la "curl\|python3\|node\|pytest\|jest"` | Agent running tools, test suites, API calls           |
| **Network socket activity** | `ss -tnp \| grep -c "ESTAB"`                    | Active TCP connections (MCP calls, HTTP APIs)         |

A stall is declared **only when ALL THREE signals are negative** for the full `effective_stall_kill` duration. Any single positive signal resets the stall timer.

### Stall Profile Thresholds

Each task file declares a `Stall Profile:` field that adjusts detection sensitivity:

| Profile      | `stall_warn` | `stall_kill` | When to use                                                                   |
| ------------ | ------------ | ------------ | ----------------------------------------------------------------------------- |
| `file-heavy` | 10 min       | 20 min       | Code writing, unit tests — frequent file writes expected                      |
| `api-heavy`  | 20 min       | 40 min       | MCP calls, API integrations, infra validation — long periods without file I/O |
| `mixed`      | 15 min       | 30 min       | Both file writes and API calls                                                |

Config overrides (`stall_warn_minutes`, `stall_kill_minutes`) always take precedence over profile defaults.

### Recovery Sequence

When a stall is confirmed (all signals negative for `stall_kill` duration):

1. `TaskStop(agent_id)` — kill stalled agent
2. `git status` — identify uncommitted partial changes
3. `git stash "dev-tasks recovery: stalled on <task>"` — stash partial state
4. Spawn fresh sub-agent with Recovery Context describing the stash and last known point
5. Track retry count — HALT after 2 failed retries

### Why Not CPU/Memory?

**CPU** is unreliable as a stall signal: the sub-agent's process spends most of its time waiting on Anthropic's inference API (near-zero CPU) even while actively reasoning. CPU spikes only during tool execution — the same pattern as a sleeping process that wakes briefly. False negatives make it unsuitable as a primary signal.

**Memory** is not useful: agent RSS stays roughly constant once loaded. A stalled agent and an active agent are indistinguishable by memory footprint.

---

## Autonomy Mode Configuration

Set `autonomy_mode` in `{project-root}/_skad/bmm/config.yaml`:

```yaml
autonomy_mode: halt-after-story # implement-only | halt-after-story | halt-on-high | full-hands-off
stall_warn_minutes: 10
stall_kill_minutes: 20
```

Or override at invocation time: `dev-tasks autonomy_mode=full-hands-off`

---

## Test Integrity Principle

> **Tests are the source of truth. They define what the code must do.**

If a verification command fails, the orchestrator and all sub-agents MUST:

- Diagnose the root cause in production code
- Fix production code only
- Re-run the command to confirm the fix

If fixing production code appears to require changing a test assertion, this is a signal that either:

1. The implementation fundamentally misunderstands the acceptance criterion (fix the implementation)
2. The test was authored incorrectly in `create-tasks` (escalate to human — do not auto-fix)

The orchestrator will HALT and surface a `TEST-INTEGRITY-HALT` when this situation is detected.
