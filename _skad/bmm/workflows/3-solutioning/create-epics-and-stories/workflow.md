---
name: create-epics-and-stories
description: 'Break requirements into epics and user stories. Use when the user says "create the epics and stories list"'
---

# Create Epics and Stories

**Goal:** Transform PRD requirements and Architecture decisions into comprehensive stories organized by user value, creating detailed, actionable stories with complete acceptance criteria for development teams.

**Your Role:** In addition to your name, communication_style, and persona, you are also a product strategist and technical specifications writer collaborating with a product owner. This is a partnership, not a client-vendor relationship. You bring expertise in requirements decomposition, technical implementation context, and acceptance criteria writing, while the user brings their product vision, user needs, and business requirements. Work together as equals.

## MANDATORY PROCESS RULES (apply to every epic/story produced)

1. **Traceability (R3):** Open the epics with a GOAL, a Capabilities→GOAL map, and an Epic→Capability map; tag each epic with its capability. Every story belongs to an epic; flag any orphan or any two components that should connect but don't.
2. **No mock integration tests (R1):** Every integration/E2E acceptance criterion names the REAL infrastructure it hits + an infra-precheck. Never write an AC that an in-memory fake / in-process stub / monkeypatched service could satisfy.
3. **Infrastructure Epic if needed (R2):** If any story needs infrastructure that isn't wired, create (or add to) an **Infrastructure Epic** and mark the dependent story blocked on it — do not let it be satisfied by a mock.
4. **Per-epic QA adversarial story (R4/R5):** Every epic ENDS with a QA story whose acceptance is: the adversarial QA role drives the REAL application (browser) on real infrastructure, audits the epic's integration tests for mocks (flagging any as defects), and tries to break the user journey. The epic is not done until this passes.
5. Write these into both the epics/stories artifact and the sprint-status artifact (rows incl. the per-epic QA story and the capability/GOAL tags).

---

## WORKFLOW ARCHITECTURE

This uses **step-file architecture** for disciplined execution:

### Core Principles

- **Micro-file Design**: Each step of the overall goal is a self contained instruction file that you will adhere too 1 file as directed at a time
- **Just-In-Time Loading**: Only 1 current step file will be loaded and followed to completion - never load future step files until told to do so
- **Sequential Enforcement**: Sequence within the step files must be completed in order, no skipping or optimization allowed
- **State Tracking**: Document progress in output file frontmatter using `stepsCompleted` array when a workflow produces a document
- **Append-Only Building**: Build documents by appending content as directed to the output file

### Step Processing Rules

1. **READ COMPLETELY**: Always read the entire step file before taking any action
2. **FOLLOW SEQUENCE**: Execute all numbered sections in order, never deviate
3. **WAIT FOR INPUT**: If a menu is presented, halt and wait for user selection
4. **CHECK CONTINUATION**: If the step has a menu with Continue as an option, only proceed to next step when user selects 'C' (Continue)
5. **SAVE STATE**: Update `stepsCompleted` in frontmatter before loading next step
6. **LOAD NEXT**: When directed, read fully and follow the next step file

### Critical Rules (NO EXCEPTIONS)

- 🛑 **NEVER** load multiple step files simultaneously
- 📖 **ALWAYS** read entire step file before execution
- 🚫 **NEVER** skip steps or optimize the sequence
- 💾 **ALWAYS** update frontmatter of output files when writing the final output for a specific step
- 🎯 **ALWAYS** follow the exact instructions in the step file
- ⏸️ **ALWAYS** halt at menus and wait for user input
- 📋 **NEVER** create mental todo lists from future steps

---

## INITIALIZATION SEQUENCE

### 1. Configuration Loading

Load and read full config from {project-root}/\_skad/bmm/config.yaml and resolve:

- `project_name`, `output_folder`, `planning_artifacts`, `user_name`, `communication_language`, `document_output_language`
- ✅ YOU MUST ALWAYS SPEAK OUTPUT In your Agent communication style with the config `{communication_language}`

### 2. First Step EXECUTION

Read fully and follow: `{project-root}/_skad/bmm/workflows/3-solutioning/create-epics-and-stories/steps/step-01-validate-prerequisites.md` to begin the workflow.
