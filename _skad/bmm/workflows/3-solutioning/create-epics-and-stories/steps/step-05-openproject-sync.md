---
name: 'step-05-openproject-sync'
description: 'Bootstrap OpenProject project and push epics/stories as work packages with artifact attachments'

# Path Definitions
workflow_path: '{project-root}/_skad/bmm/workflows/3-solutioning/create-epics-and-stories'
op_endpoint: 'https://mcp.baisoln.com/openproject/mcp'

# File References
thisStepFile: './step-05-openproject-sync.md'
configFile: '{project-root}/_skad/bmm/config.yaml'
opMapFile: '{project-root}/_skad/bmm/openproject-map.yaml'
outputFile: '{planning_artifacts}/epics.md'
skillDir: '{project-root}/.claude/skills/openproject'
---

# Step 5: OpenProject Sync

## STEP GOAL

Bootstrap the OpenProject project, create Epic and Story work packages in the correct hierarchy, upload `epics.md` as an artifact, and write `openproject-map.yaml` as the permanent ID registry. Self-install the OpenProject Claude skill if not present.

## MANDATORY RULES

- 🟡 If OpenProject is unreachable: warn clearly and HALT gracefully — do NOT fail silently
- 🔁 If `openproject-map.yaml` already contains IDs for an epic or story, UPDATE (do not create duplicates)
- 💾 Write all created WP IDs to `openproject-map.yaml` before finishing
- 🔑 All calls use the MCP JSON-RPC endpoint via `curl` through the Bash tool
- 📋 Communicate in {communication_language} tailored to {user_skill_level}

---

## MCP CALL PATTERN

Use this pattern for ALL OpenProject tool calls:

```bash
curl -s -X POST https://mcp.baisoln.com/openproject/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"jsonrpc":"2.0","id":<ID>,"method":"tools/call","params":{"name":"<TOOL>","arguments":<ARGS_JSON>}}'
```

Parse the response:
```bash
# Get the result text (a JSON string)
RESULT=$(echo "$RESPONSE" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['result']['content'][0]['text'])")
# Pretty-print it
echo "$RESULT" | python3 -m json.tool
```

---

## EXECUTION

### Phase 0 — Self-Install Claude Skill

Ensure the OpenProject Claude skill is present in the target project so future conversations have OP awareness.

```bash
# Check if the skill already exists
ls {project-root}/.claude/skills/openproject/SKILL.md 2>/dev/null && echo "EXISTS" || echo "MISSING"
```

If EXISTS: skip this phase.

If MISSING, try in order:

**Option A — OpenProject module was installed via `skad install`:**
```bash
# Check if module skill files are present in the installed module
ls {project-root}/_skad/openproject/claude-skills/openproject/SKILL.md 2>/dev/null && echo "MODULE_PRESENT" || echo "MODULE_ABSENT"
```
If MODULE_PRESENT:
```bash
mkdir -p {project-root}/.claude/skills/openproject/prompts
cp -r {project-root}/_skad/openproject/claude-skills/openproject/. {project-root}/.claude/skills/openproject/
```

**Option B — Write inline (module not installed):**
Create `{project-root}/.claude/skills/openproject/prompts/` directory, then write two files:

`{project-root}/.claude/skills/openproject/SKILL.md`:
```markdown
---
name: openproject
description: >
  OpenProject MCP bridge. Interact with the OpenProject project management
  instance at mcp.baisoln.com/openproject/mcp. Use when the user asks about
  projects, work packages, tasks, stories, epics, bugs, sprints, time logging,
  assignees, relations, watchers, attachments, or any OpenProject / PM
  operations. Also activate proactively when you detect a need to create,
  update, or query project management data. Do NOT load full context unless
  this skill is explicitly invoked or a clear PM action is required.
---

Read `prompts/instructions.md` and execute.
```

`{project-root}/.claude/skills/openproject/prompts/instructions.md`:
Write a minimal stub:
```markdown
# OpenProject MCP Bridge

**Endpoint:** https://mcp.baisoln.com/openproject/mcp
**Protocol:** JSON-RPC 2.0 over HTTPS POST (44 tools available)

## Call Pattern
Use Bash tool with curl:
```bash
curl -s -X POST https://mcp.baisoln.com/openproject/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"<TOOL>","arguments":<ARGS>}}'
```

Key tools: test_connection, list_projects, get_project, create_project,
list_work_packages, create_work_package, update_work_package, delete_work_package,
update_work_package_status, set_work_package_parent, bulk_create_work_packages,
add_work_package_attachment, list_users, log_time, list_statuses, list_types.

For the full 44-tool catalog, run `skad install openproject` to install the complete skill.
```
```

Output: `✅ OpenProject Claude skill installed at .claude/skills/openproject/`

---

### Phase 1 — Test Connectivity

```bash
RESPONSE=$(curl -s -X POST https://mcp.baisoln.com/openproject/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"test_connection","arguments":{}}}')
echo "$RESPONSE" | python3 -c "import json,sys; r=json.loads(json.load(sys.stdin)['result']['content'][0]['text']); print('OK' if r.get('success') else 'FAIL: '+r.get('message','unknown'))"
```

- If result is `FAIL`: output a clear warning with the failure reason, then offer:
  - [R] Retry — try connecting again
  - [S] Skip — complete the workflow without OpenProject sync
  If user selects S: output "⚠️ OpenProject sync skipped. You can run it later with `sync openproject status`."
  Then: Read fully and follow: `{project-root}/_skad/core/tasks/help.md`
- If result is `OK`: continue to Phase 2.

---

### Phase 2 — Ensure OpenProject Project

Read `{project-root}/_skad/bmm/config.yaml`. Extract `openproject_id`.

**If `openproject_id` is present (not null, not empty):**
```bash
# Verify project exists
RESPONSE=$(curl -s -X POST https://mcp.baisoln.com/openproject/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":2,\"method\":\"tools/call\",\"params\":{\"name\":\"get_project\",\"arguments\":{\"project_id\":$OPENPROJECT_ID}}}")
```
- If project found: use it. Set `{{op_project_id}}` = the confirmed ID.
- If project not found (error): warn user "Project ID `{{openproject_id}}` not found — creating a new project."

**If `openproject_id` is absent OR project not found:**
```bash
# Create the project
IDENTIFIER=$(echo "{project_name}" | tr '[:upper:]' '[:lower:]' | tr ' ' '-' | tr -cd '[:alnum:]-' | cut -c1-25)
RESPONSE=$(curl -s -X POST https://mcp.baisoln.com/openproject/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":3,\"method\":\"tools/call\",\"params\":{\"name\":\"create_project\",\"arguments\":{\"name\":\"{project_name}\",\"identifier\":\"$IDENTIFIER\",\"description\":\"Generated by SKAD-Method BMM workflow\",\"public\":false}}}")
```
- Extract `id` from result.
- Set `{{op_project_id}}` = newly created project ID.
- **Write `openproject_id: {{op_project_id}}` into `_skad/bmm/config.yaml`** (append or update the field).

---

### Phase 3 — Resolve Type IDs

```bash
RESPONSE=$(curl -s -X POST https://mcp.baisoln.com/openproject/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":4,\"method\":\"tools/call\",\"params\":{\"name\":\"list_types\",\"arguments\":{\"project_id\":$OP_PROJECT_ID}}}")
```

Parse the types list. Match (case-insensitive):
- **Epic type**: first match of "Epic"
- **Story type**: first match of "User Story", "Story", or "Feature"
- **Task type**: first match of "Task"

Store as `{{type_epic_id}}`, `{{type_story_id}}`, `{{type_task_id}}`.

If any type is NOT found, warn: "Type '{{name}}' not found in OpenProject — using default type (null). Epics/Stories/Tasks will be created as the default work package type."

---

### Phase 4 — Resolve Status IDs

```bash
RESPONSE=$(curl -s -X POST https://mcp.baisoln.com/openproject/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"list_statuses","arguments":{}}}')
```

Build a status map (local → OpenProject status ID):

| Local Status | Match (case-insensitive) |
|---|---|
| `backlog` / `ready_for_dev` / `ready_for_task` | "New", "Backlog", "Open", or first available |
| `in_progress` / `in_dev` | "In Progress", "In Development" |
| `review` / `in_review` / `in_test` | "In Review", "Testing", "Under Review" |
| `done` / `passed` | "Closed", "Done", "Resolved" |
| `failed` | "Rejected", "Failed", "Cancelled" |

Store as `{{status_map}}` (a dict of local_name → op_status_id).

---

### Phase 5 — Load or Initialize Map

Check if `{project-root}/_skad/bmm/openproject-map.yaml` exists:
- If YES: load it into `{{op_map}}`.
- If NO: initialize `{{op_map}}` as an empty map structure:

```yaml
# Auto-generated by SKAD-BMM OpenProject Sync — do not edit manually
openproject_id: <op_project_id>
type_ids:
  epic: <type_epic_id>
  story: <type_story_id>
  task: <type_task_id>
status_map:
  backlog: <id>
  ready_for_dev: <id>
  in_progress: <id>
  review: <id>
  done: <id>
  failed: <id>
  ready_for_task: <id>
  in_dev: <id>
  in_dev_complete: <id>
  in_review: <id>
  in_test: <id>
  passed: <id>
work_packages: {}
```

---

### Phase 6 — Parse Epics & Stories from epics.md

Read `{planning_artifacts}/epics.md`.

Extract the epic/story hierarchy by scanning for:
- Epic headers: `## Epic N: <title>` — sets `{{epic_key}}` = `epic-N`, `{{epic_title}}`
- Story headers: `### Story N.M: <title>` — sets `{{story_key}}` = `N-M-<slug>`, `{{story_title}}`
  - `<slug>` = kebab-case of the story title (max 40 chars, lowercase, hyphens only)

Build `{{epics_stories}}` list:
```
[
  { key: "epic-1", title: "Epic 1: ...", stories: [
      { key: "1-1-story-slug", title: "Story 1.1: ...", epic_key: "epic-1" },
      ...
  ]},
  ...
]
```

---

### Phase 7 — Create Epic Work Packages

For each epic in `{{epics_stories}}`:

1. Check `{{op_map}}.work_packages[epic.key]` — if `wp_id` exists, skip creation (already synced).
2. If NOT in map:
```bash
RESPONSE=$(curl -s -X POST https://mcp.baisoln.com/openproject/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":<ID>,\"method\":\"tools/call\",\"params\":{\"name\":\"create_work_package\",\"arguments\":{\"project_id\":$OP_PROJECT_ID,\"subject\":\"<epic.title>\",\"type_id\":$TYPE_EPIC_ID}}}")
```
3. Extract `id` from result. Store in map: `{{op_map}}.work_packages[epic.key].wp_id = <id>`.
4. Output: `✅ Epic WP created: {{epic.title}} → WP #{{id}}`

---

### Phase 8 — Upload epics.md to Each Epic WP

For each epic that was just created OR already existed in the map:

```bash
# Base64-encode the epics.md file
FILE_DATA=$(base64 -w 0 "{planning_artifacts}/epics.md")
EPIC_WP_ID={{op_map.work_packages[epic.key].wp_id}}

RESPONSE=$(curl -s -X POST https://mcp.baisoln.com/openproject/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":<ID>,\"method\":\"tools/call\",\"params\":{\"name\":\"add_work_package_attachment\",\"arguments\":{\"work_package_id\":$EPIC_WP_ID,\"file_data\":\"$FILE_DATA\",\"filename\":\"epics.md\",\"content_type\":\"text/markdown\",\"description\":\"SKAD epics and stories breakdown\"}}}")
```

Extract `attachment_id` from result. Store: `{{op_map}}.work_packages[epic.key].epics_attachment_id = <attachment_id>`.

---

### Phase 9 — Create Story Work Packages

For each story in `{{epics_stories}}`:

1. Check `{{op_map}}.work_packages[story.key]` — if `wp_id` exists, skip creation.
2. Get `{{parent_wp_id}}` = `{{op_map}}.work_packages[story.epic_key].wp_id`
3. If NOT in map:
```bash
RESPONSE=$(curl -s -X POST https://mcp.baisoln.com/openproject/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":<ID>,\"method\":\"tools/call\",\"params\":{\"name\":\"create_work_package\",\"arguments\":{\"project_id\":$OP_PROJECT_ID,\"subject\":\"<story.title>\",\"type_id\":$TYPE_STORY_ID}}}")
```
4. Extract `id`. Then set parent:
```bash
curl -s -X POST https://mcp.baisoln.com/openproject/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":<ID>,\"method\":\"tools/call\",\"params\":{\"name\":\"set_work_package_parent\",\"arguments\":{\"work_package_id\":$STORY_WP_ID,\"parent_id\":$PARENT_WP_ID}}}"
```
5. Store in map:
```yaml
<story.key>:
  wp_id: <story_wp_id>
  parent_wp_id: <epic_wp_id>
  title: <story.title>
```
6. Output: `  ✅ Story WP created: {{story.title}} → WP #{{id}} (child of Epic #{{parent_wp_id}})`

---

### Phase 10 — Save openproject-map.yaml

Write the complete `{{op_map}}` to `{project-root}/_skad/bmm/openproject-map.yaml`.

Include a header comment:
```yaml
# Auto-generated by SKAD-BMM OpenProject Sync
# Managed by: create-epics-and-stories step-05 and openproject-sync workflow
# Do not edit manually — values are updated automatically during dev workflow
```

---

### Phase 11 — Completion Report

Output a summary:

```
🔗 OpenProject Sync Complete

Project:    {project_name} → OP Project #{{op_project_id}}
Epics:      {{epic_count}} work packages created/verified
Stories:    {{story_count}} work packages created/verified
Artifacts:  epics.md uploaded to each epic WP
Map saved:  _skad/bmm/openproject-map.yaml

Next: Run create-story → create-tasks → dev-tasks
      Status will sync to OpenProject automatically as tasks progress.
```

Workflow complete. Read fully and follow: `{project-root}/_skad/core/tasks/help.md`
