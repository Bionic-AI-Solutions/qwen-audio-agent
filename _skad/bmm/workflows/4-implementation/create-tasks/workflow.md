---
name: create-tasks
description: 'Break a ready-for-dev story into self-contained atomic task files for sub-agent implementation. Use when the user says "create tasks for [story]" or "break down the story into tasks"'
---

# Create Tasks Workflow

**Goal:** Atomize a story's Tasks/Subtasks into self-contained task files that sub-agents can implement in a single short session without loading any external documents.

**Your Role:** Task decomposition engine that prevents context overload in sub-agents.

## MANDATORY PROCESS RULES

- **R3 Traceability:** each task file names its parent story (→ epic → capability → GOAL).
- **R1 No mock integration tests:** every integration/E2E test task states the REAL infrastructure it hits + an infra-precheck, and explicitly forbids mocks (in-memory fakes, in-process stubs, monkeypatched downstreams). Unit tests may mock; integration tests may not.
- **R2 Infra gap → Infrastructure Epic:** if a task needs unwired infrastructure, do not write a mock task — emit an infra-dependency task pointing at the Infrastructure Epic and mark the test task blocked.
- **R4/R5 QA task:** the final task of a story is a QA-adversarial-real-app task: drive the REAL application on real infrastructure, audit integration tests for mocks (flag as defects), and try to break the user journey.

- Communicate all responses in {communication_language} and generate all documents in {document_output_language}
- Each task file must be 100% self-contained: no "see architecture doc" — inline the relevant excerpt verbatim
- Tasks must be tiny: 1-3 files to touch maximum, one function or feature increment per task
- Tasks must be independently testable and ordered by dependency
- Story file remains the source of truth; task files reference back to it
- ZERO USER INTERVENTION except for the optional checkpoint in Step 3

---

## INITIALIZATION

### Configuration Loading

Load config from `{project-root}/_skad/bmm/config.yaml` and resolve:

- `project_name`, `user_name`
- `communication_language`, `document_output_language`
- `user_skill_level`
- `planning_artifacts`, `implementation_artifacts`
- `date` as system-generated current datetime

### Paths

- `installed_path` = `{project-root}/_skad/bmm/workflows/4-implementation/create-tasks`
- `task_template` = `{installed_path}/task-template.md`
- `sprint_status` = `{implementation_artifacts}/sprint-status.yaml`
- `story_file` = `` (explicit story path; auto-discovered if empty)
- `architecture_file` = `{planning_artifacts}/architecture.md`

---

## EXECUTION

<workflow>

<step n="1" goal="Locate and load the target story file">
  <check if="{{story_path}} is provided by user">
    <action>Use {{story_path}} directly</action>
    <action>Read COMPLETE story file</action>
    <action>Extract story_key from filename (e.g., "1-2-user-authentication")</action>
    <action>Extract epic_num and story_num from story_key</action>
    <action>Set {{tasks_dir}} = {implementation_artifacts}/tasks/{{story_key}}</action>
    <action>GOTO step 2 analysis</action>
  </check>

  <!-- Auto-discover from sprint status -->
  <check if="{{sprint_status}} file exists AND no story_path provided">
    <critical>MUST read COMPLETE sprint-status.yaml from start to end to preserve order</critical>
    <action>Load the FULL file: {{sprint_status}}</action>
    <action>Read ALL lines from beginning to end — do not skip any content</action>
    <action>Parse the development_status section completely</action>

    <action>Find the FIRST story (reading top to bottom) where:
      - Key matches pattern: number-number-name (e.g., "1-2-user-auth")
      - NOT an epic key (epic-X) or retrospective (epic-X-retrospective)
      - Status value equals "ready-for-dev"
    </action>

    <check if="no ready-for-dev story found">
      <output>No ready-for-dev story found in sprint-status.yaml.

        Run create-story first to prepare a story file, then run create-tasks.
        Or provide a specific story path: "create tasks for path/to/story.md"
      </output>
      <action>HALT</action>
    </check>

    <action>Extract story_key from found entry</action>
    <action>Extract epic_num and story_num from story_key</action>
    <action>Construct story path: {implementation_artifacts}/{{story_key}}.md</action>
    <action>Read COMPLETE story file</action>
    <action>Set {{tasks_dir}} = {implementation_artifacts}/tasks/{{story_key}}</action>

  </check>

  <check if="no sprint_status file AND no story_path provided">
    <output>No sprint-status.yaml found and no story path provided.

      Provide a story path: "create tasks for path/to/story.md"
    </output>
    <action>HALT</action>

  </check>

  <!-- Parse story content -->

<action>Parse all story sections: Story, Acceptance Criteria, Tasks/Subtasks, Dev Notes, References, Dev Agent Record</action>
<action>Extract complete Tasks/Subtasks list including all subtask bullets</action>
<action>Count total top-level tasks as {{total_tasks}}</action>
<action>Warn (but continue) if story Status is not "ready-for-dev"</action>

  <!-- Handle existing task files -->
  <check if="{{tasks_dir}} directory already exists with task files">
    <output>Task files already exist for {{story_key}} in {{tasks_dir}}.

      [R] Regenerate all — overwrite all existing task files
      [U] Update incomplete — regenerate only tasks not yet completed
      [X] Cancel — keep existing task files
    </output>
    <ask>Choose [R], [U], or [X]:</ask>

    <check if="user chooses X">
      <action>HALT — existing task files preserved</action>
    </check>

    <check if="user chooses U">
      <action>Scan existing task files for completed Completion Checklists (all items checked)</action>
      <action>Mark completed task indices to skip in Step 4 generation</action>
      <action>Continue with Step 2 for incomplete tasks only</action>
    </check>

    <check if="user chooses R">
      <action>Continue with Step 2 — all task files will be overwritten</action>
    </check>

  </check>
</step>

<step n="2" goal="Deep context extraction — exhaustive, zero shortcuts">
  <critical>EVERY task file must be self-contained. Extract all context that sub-agents will need so they NEVER have to load external documents.</critical>

  <!-- Parse Dev Notes for all references -->

<action>From story Dev Notes, extract: - All file paths mentioned (files to create, modify, or reference) - All library/framework names and version requirements - All architecture pattern names referenced (e.g., "repository pattern", "service layer") - All testing requirements (framework, coverage expectations, test locations) - All "do not" constraints already listed - All references from the References subsection
</action>

  <!-- Load architecture — whole or sharded, mirroring create-story step 3 -->
  <check if="whole architecture file exists at {planning_artifacts}/*architecture*.md">
    <action>Load complete architecture file</action>
  </check>
  <check if="architecture is sharded (folder with index)">
    <action>Load architecture index file</action>
    <action>Load all architecture shard files referenced in the index</action>
  </check>

<action>For each architecture pattern referenced in Dev Notes: - Find the relevant section in architecture document - Copy that section VERBATIM (not a summary — the actual text the dev needs) - Tag it with which story task(s) it applies to
</action>

  <!-- Previous story intelligence -->
  <check if="story_num > 1">
    <action>Find the immediately preceding story file in {implementation_artifacts} for epic {{epic_num}}</action>
    <action>From its Dev Agent Record, extract: File List, Completion Notes, patterns established</action>
    <action>Store as {{prior_patterns}} — will be inlined into task files where relevant</action>
  </check>

  <!-- Current file state from git -->
  <check if="git repository detected">
    <action>For each file path mentioned in Dev Notes, check if it currently exists in the repo</action>
    <action>For files that exist: capture the relevant current lines (imports, exports, existing functions, class structure)</action>
    <action>Store as {{existing_file_context}} — actual current state as baseline for sub-agents</action>
  </check>

  <!-- Dependency graph across tasks -->

<action>For each top-level task in the story's Tasks/Subtasks: 1. Identify what files/functions/data this task creates or modifies 2. Identify what the NEXT task needs from this task's output 3. Determine if any task has a hard dependency on a previous task's output 4. Build strict ordering: Task N must complete before Task N+1 starts (or note if parallel is safe)
</action>
<action>Store dependency chain as {{task_dependency_map}}</action>

  <!-- File scope enforcement: detect oversized tasks early -->

<action>For each top-level task, estimate the number of distinct files it will touch</action>
<action>Flag any task that appears to touch more than 3 files as needing a split</action>
<action>Plan splits for oversized tasks: task-Na and task-Nb with clear handoff between them</action>
</step>

<step n="3" goal="Optional checkpoint before generation">
  <output>**Context Analysis Complete for {{story_key}}**

    Tasks identified: {{total_tasks}} (plus any planned splits)
    Architecture sections extracted: for inlining per task
    File paths resolved: from Dev Notes and git scan
    Dependency chain: mapped across all tasks

    [A] Advanced Elicitation — push for deeper context (missing architecture, ambiguous file paths, unclear dependencies)
    [C] Continue — generate task files now

  </output>
  <ask>Choose [A] or [C]:</ask>

  <check if="user chooses A">
    <action>Load and follow: {project-root}/_skad/core/workflows/advanced-elicitation/workflow.md</action>
    <action>Focus elicitation on: missing architecture context, ambiguous file paths, unclear dependency order</action>
    <action>After elicitation returns, proceed to Step 4</action>
  </check>

  <check if="user chooses C">
    <action>Proceed to Step 4</action>
  </check>
</step>

<step n="4" goal="Generate all task files from template">
  <critical>Each task file MUST be self-contained. NEVER write "see architecture doc" or "refer to story". INLINE the relevant excerpt verbatim.</critical>
  <critical>3-file hard limit per task. If a task touches more than 3 files, SPLIT it into task-Na and task-Nb.</critical>

<action>Create directory {{tasks_dir}} if it does not exist</action>
<action>Load task-template.md from: {{task_template}}</action>

  <!-- Iterate over each top-level task -->

<action>For each Task N (iterating 1 through {{total_tasks}}, skipping completed tasks if [U] mode):

    **Resolve template variables for this task:**

    1. {{task_num}} = N (or Na/Nb if split)
    2. {{total_tasks}} = total count (including splits)
    3. {{task_title}} = kebab-cased task title (max 40 chars for slug)
    4. {{story_key}} = the story key
    5. {{date}} = current date
    6. {{task_description}} = full task text from story Tasks/Subtasks plus context sentence explaining WHY this task exists
    7. {{ac_refs}} = list of AC numbers this task satisfies (from "(AC: #)" annotations in story)
    8. {{dependency_on}} = "None — this is Task 1" OR "Task {{N-1}}: [title] must be complete — it creates [specific output this task needs]"
    9. {{produces_for_next}} = "Creates [specific file/function/data] that Task {{N+1}} needs" OR "None — this is the final task"
    10. {{files_table}} = markdown table rows, one per file (max 3); Action = Create|Modify|Delete; What to do = specific 1-line instruction
        - If more than 3 files detected: SPLIT into Na/Nb and generate two task files instead
    11. {{implementation_instructions}} = expanded, unambiguous instructions derived from the story task + architecture context; include:
        - What function/method/component to create or modify
        - Exact function signatures if determinable
        - Data shapes/types if relevant
        - Error handling approach
        - No ambiguity — the sub-agent must not make design decisions
    12. {{subtasks_checklist}} = all subtask bullets from story converted to "- [ ] subtask text"
    13. {{architecture_excerpt}} = verbatim copy of the architecture section(s) relevant to this task's files/patterns; include section header so context is clear
    14. {{code_patterns}} = code snippets from {{prior_patterns}} or {{existing_file_context}} that this task must follow; include the actual snippet, not a description
    15. {{existing_file_excerpts}} = current content of files this task modifies (imports block, class/function signatures, key lines); label each with file path
    16. {{test_requirements}} = specific tests to write: unit test targets (function names), integration test scenarios if applicable, edge cases from story ACs; be specific, not generic
    17. {{verification_commands}} = actual runnable shell commands, e.g.:
        - npm test -- --testPathPattern=auth
        - npx jest src/services/auth.test.ts
        - npm run lint src/services/auth.ts
    18. {{do_not_list}} = task-specific prohibitions derived from architecture constraints + story Dev Notes; e.g., "Do NOT call the external API directly — use the service layer abstraction"
    19. {{stall_profile}} = classify the task's expected activity pattern:
        - `file-heavy` — (default) task primarily creates/modifies source files and tests
        - `api-heavy` — task involves MCP tool calls, API integrations, external service round-trips, or infrastructure validation where long periods pass without file writes
        - `mixed` — task does both file I/O and API/MCP calls
        Determine by inspecting: does the task call external APIs, MCP servers, or run long integration test suites? If yes → `api-heavy` or `mixed`. If purely writing code and unit tests → `file-heavy`.

    **Write task file:**
    - Filename: task-{{N}}-{{slug}}.md where slug = kebab-case of task_title
    - Path: {{tasks_dir}}/task-{{N}}-{{slug}}.md
    - Substitute ALL template variables — the output file contains NO {{variables}}, only resolved content

  </action>

<action>Track all generated task file paths as {{generated_task_files}}</action>
<action>Store count of implementation task files as {{impl_task_count}} = number of files generated so far</action>

  <!-- MANDATORY: Always append two standard test tasks after all implementation tasks -->

<action>Set {{unit_test_task_num}} = {{impl_task_count}} + 1</action>
<action>Set {{acceptance_task_num}} = {{impl_task_count}} + 2</action>
<action>Set {{final_total}} = {{acceptance_task_num}}</action>

  <!-- Update "Task N of M" denominator in all previously written task files to use {{final_total}} -->

<action>In each already-written task file in {{tasks_dir}}, replace the total count in the heading line ("Task N of M") so M = {{final_total}}</action>

  <!-- Standard Task U: Unit Tests -->

<action>Generate task file: {{tasks_dir}}/task-{{unit_test_task_num}}-unit-tests.md using task-template.md with:

    {{task_num}}          = {{unit_test_task_num}}
    {{total_tasks}}       = {{final_total}}
    {{task_title}}        = "Write and Run Unit Tests"
    {{story_key}}         = {{story_key}}
    {{date}}              = current date

    {{task_description}}  = "Write comprehensive unit tests for every function, method, and component introduced across Tasks 1 through {{impl_task_count}}. Tests must be isolated — each unit tested in isolation using mocks/stubs for all external dependencies. This task does NOT implement any feature code."

    {{ac_refs}}           = "All ACs — unit tests verify the underlying logic that each AC depends on"

    {{dependency_on}}     = "All implementation tasks (1 through {{impl_task_count}}) must be complete. Unit tests target functions that those tasks created — test files cannot be written until the implementation exists."

    {{produces_for_next}} = "Passing isolated unit test suite. Task {{acceptance_task_num}} (Story Acceptance Tests) requires this to be complete and green before running."

    {{files_table}}       = One table row per implementation file introduced in tasks 1-{{impl_task_count}}:
                            | Create or Modify | [impl-file-path].test.[ext] | Write unit tests for [function names from that task] |
                            Derive file paths from the implementation files identified during Step 2 context extraction.
                            Use the project's test file naming convention (e.g., .test.ts, .spec.ts, _test.go).

    {{implementation_instructions}} = "
      For each function, method, or component created in Tasks 1–{{impl_task_count}}:

      1. Create a describe/test block named after the function or module
      2. Test coverage required per unit:
         - Happy path: expected input → expected output
         - Edge cases: empty/null inputs, boundary values, maximum values
         - Error paths: invalid input, dependency failure, timeout
      3. Mock ALL external dependencies (database, HTTP APIs, filesystem, timers)
         Tests must run offline with zero network or I/O calls
      4. Tests must be deterministic — no random values, no timing assumptions
      5. Use the test framework and import patterns shown in Embedded Code Patterns

      NEVER modify implementation files — this task touches test files only."

    {{subtasks_checklist}} = One checkbox per implementation task:
                             "- [ ] Unit tests for Task [N]: [task title]"

    {{architecture_excerpt}} = Verbatim testing standards section from architecture:
                               test framework name, test file locations, naming conventions, coverage expectations

    {{code_patterns}}     = One or more actual existing test files from the codebase showing:
                            describe/it/test block structure, mock patterns, assertion style
                            (Copy the real file content, not a description of it)

    {{existing_file_excerpts}} = Current content of any test files that will be modified (if they already exist)

    {{test_requirements}} = "N/A — this task IS the test-writing task. Verification commands below confirm tests pass."

    {{verification_commands}} = Runnable commands targeting only the new test files, e.g.:
                                npx jest src/services/auth.test.ts src/models/user.test.ts --coverage
                                (Derive actual paths from files_table above)

    {{do_not_list}}       = "Do NOT modify any implementation files — test files only
                             Do NOT write integration or e2e tests here — those belong in Task {{acceptance_task_num}}
                             Do NOT use real databases, network calls, or filesystem in unit tests — mock everything
                             Do NOT skip any function introduced in Tasks 1–{{impl_task_count}}"

  </action>

  <!-- Standard Task S: Story Acceptance Tests -->

<action>Generate task file: {{tasks_dir}}/task-{{acceptance_task_num}}-story-acceptance-tests.md using task-template.md with:

    {{task_num}}          = {{acceptance_task_num}}
    {{total_tasks}}       = {{final_total}}
    {{task_title}}        = "Story Acceptance Tests"
    {{story_key}}         = {{story_key}}
    {{date}}              = current date

    {{task_description}}  = "Verify the complete story end-to-end against every Acceptance Criterion. Validates the story as a user-facing feature — the full flow from user action to expected outcome, using real dependencies (not unit mocks). Run after all implementation tasks and unit tests are complete."

    {{ac_refs}}           = Every AC number from the story (e.g., "AC 1, AC 2, AC 3, AC 4")

    {{dependency_on}}     = "Task {{unit_test_task_num}}: Unit Tests must be complete and all passing before running acceptance tests."

    {{produces_for_next}} = "None — this is the final task. Story is complete when this task passes and the full regression suite is green."

    {{files_table}}       = One row for the acceptance test file:
                            | Create | [test-dir]/[story_key].acceptance.test.[ext] | Write acceptance tests covering every AC |
                            Use the project's integration/e2e test location convention.

    {{implementation_instructions}} = "
      For each Acceptance Criterion in the story:

      1. Write one test case per AC (minimum), named after the AC
      2. Structure: Given [precondition] → When [user action] → Then [expected outcome]
         Mirror the AC wording exactly so reviewers can match test to requirement
      3. Use real dependencies where possible:
         - Real database in test/seed mode
         - Real service layer (no mocking business logic)
         - Mock only at external system boundaries (third-party APIs, email, payment)
      4. Cover both the success path AND the primary failure path for each AC
      5. If the project has an e2e or integration test framework, prefer it over the unit test framework

      NEVER mark this task complete if any AC lacks at least one covering test."

    {{subtasks_checklist}} = One checkbox per AC, with description:
                             "- [ ] AC [N]: [full AC text from story]"

    {{architecture_excerpt}} = Verbatim integration/e2e testing section from architecture:
                               test environment setup, seed data approach, test database config, e2e framework details

    {{code_patterns}}     = Existing integration or e2e test file(s) from the codebase showing:
                            test setup/teardown, database seeding, real HTTP or service calls
                            (Copy the real file content, not a description)

    {{existing_file_excerpts}} = Current content of the acceptance test file if it already exists (when regenerating)

    {{test_requirements}} = "Every AC must have at least one passing test case. Tests must exercise real integrations — not unit-level mocks."

    {{verification_commands}} = Two commands:
                                1. Run the acceptance test file:
                                   npx jest [story_key].acceptance.test --runInBand
                                   (or project e2e command)
                                2. Full regression suite to confirm no regressions:
                                   npm test

    {{do_not_list}}       = "Do NOT use unit-level mocks for the full AC flow — acceptance tests must exercise real integrations
                             Do NOT skip any AC — every criterion needs at least one test
                             Do NOT mark complete if the full regression suite (npm test) has any failures
                             Do NOT implement feature code here — tests only"

  </action>

<action>Append both new task file paths to {{generated_task_files}}</action>
<action>Set {{total_tasks_generated}} = {{final_total}}</action>
</step>

<step n="5" goal="Update story file to link to generated task files">
  <action>Load the COMPLETE story file from {{story_file}}</action>

  <!-- Add Task Files subsection to Dev Notes — before the References subsection -->

<action>In the Dev Notes section, insert a new "### Task Files" subsection BEFORE "### References":

    ```markdown
    ### Task Files

    Generated by create-tasks on {{date}}. Story is the source of truth; task files are implementation guides.

    {{generated_task_file_links}}
    ```

    Where {{generated_task_file_links}} = one line per file:
    `- [task-N-slug](tasks/{{story_key}}/task-N-slug.md) — [one-line task description]`

  </action>

<critical>Do NOT modify: Tasks/Subtasks checkboxes, Acceptance Criteria, Story statement, Dev Agent Record, Status — those are dev-story's domain</critical>
<action>Save story file preserving ALL existing content and structure</action>
</step>

<step n="6" goal="QUALITY GATE: Validate task file self-containment">
  <critical>This step is MANDATORY. Sub-agents spawned by dev-tasks receive ZERO pre-loaded context. Every task file must stand alone.</critical>

<action>For each generated task file in {{tasks_dir}}, verify:

    **Self-Containment Checks (FAIL = regenerate the task):**
    1. NO external references — file must NOT contain phrases like:
       - "see architecture doc", "refer to story", "check the PRD"
       - "as described in", "per the architecture", "see above"
       - Any {project-root}/_skad/ path references to planning docs
    2. Architecture excerpt is VERBATIM — not a summary, not a paraphrase, but the actual text the dev needs
    3. Existing file excerpts are present — for files being modified, current imports/signatures/structure must be inlined
    4. Verification commands are RUNNABLE — actual shell commands with real file paths, not placeholders like "run the tests"
    5. DO NOT list is specific — derived from architecture constraints, not generic boilerplate
    6. Files to touch are EXACT — real file paths, not patterns or descriptions
    7. Implementation instructions are UNAMBIGUOUS — function signatures, data shapes, error handling approach are specified

    **Structural Checks:**
    8. Task touches at most 3 files (Exact Files to Touch table has ≤ 3 rows)
    9. Dependency Order section specifies what this task Requires and what it Produces
    10. Completion Checklist has specific, verifiable items
    11. Stall Profile is set to file-heavy, api-heavy, or mixed

  </action>

  <check if="any task file fails self-containment checks">
    <output>⚠️ Task file(s) failed self-containment validation. Regenerating with full inlined context...</output>
    <action>For each failing task: re-extract the missing context from architecture/story/git and regenerate the task file with all content inlined</action>
  </check>

<output>✅ All {{total_tasks_generated}} task files passed self-containment validation.</output>
</step>

<step n="7" goal="Update sprint status and report completion">
  <check if="sprint status file exists">
    <action>Load the FULL file: {{sprint_status}}</action>
    <action>Update last_updated field to current date</action>
    <action>Do NOT change the story status (leave as "ready-for-dev") — task files are implementation detail, not a new workflow state</action>
    <action>Save file preserving ALL comments and structure including STATUS DEFINITIONS</action>
  </check>

<output>**Tasks Created for {{story_key}}, {user_name}!**

    **Task Files Generated:** {{total_tasks_generated}} files (all validated for self-containment)
    **Location:** {{tasks_dir}}/

    | # | File | Scope | Depends On |
    |---|------|-------|------------|
    {{task_summary_table}}

    **Story file updated** with links to task files under Dev Notes → Task Files.

    Each task file is 100% self-contained — sub-agents can implement any task without loading external documents.

    **Next Steps:**
    1. Review task files in {{tasks_dir}}/ (each is self-contained, no external docs needed)
    2. Run `dev-tasks` — it will orchestrate task-by-task implementation with automated implement → review → test pipeline
    3. Or run `dev-story` for single-agent story implementation

  </output>
</step>

</workflow>
