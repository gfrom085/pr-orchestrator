# 🔄 PR Orchestrator - Workflow Détaillé

## 📋 Vue d'Ensemble du Workflow

Le workflow de PR Orchestrator se décompose en 5 phases principales, chacune avec des responsabilités claires.

---

## Phase 1: Input Processing

### 1.1 Load Input File

```
Input: tasks.yaml ou tasks.json
Process:
  - Detect file format (YAML vs JSON)
  - Read file content
  - Handle file not found errors
  - Handle malformed file errors
Output: Raw content (string)
```

### 1.2 Parse Content

```
Input: Raw content
Process:
  - Parse YAML/JSON structure
  - Validate schema version
  - Extract project metadata
  - Extract tasks list
Output: ParsedInput object
```

### 1.3 Validate Structure

```
Input: ParsedInput
Validations:
  ✓ All required fields present
  ✓ Task IDs unique
  ✓ Dependencies reference valid task IDs
  ✓ Complexity values in valid range
  ✓ Estimated hours > 0
  ✓ No duplicate task names
Errors: Explicit error messages with location
Output: List[Task]
```

### 1.4 Build Task Objects

```
Input: Validated data
Process:
  - Create Task objects
  - Normalize fields
  - Set defaults for optional fields
  - Prepare for dependency resolution
Output: List[Task]
```

---

## Phase 2: Dependency Analysis

### 2.1 Build Dependency Graph

```
Input: List[Task]
Process:
  - Extract all dependencies
  - Create graph nodes (tasks)
  - Create graph edges (depends_on relationships)
  - Validate all dependency IDs exist
Output: DependencyGraph
```

### 2.2 Detect Cycles

```
Input: DependencyGraph
Algorithm: Depth-First Search with coloring
Process:
  - Mark all nodes as unvisited (white)
  - For each unvisited node:
    - Mark as visiting (gray)
    - Visit all children recursively
    - If encounter gray node → CYCLE
    - Mark as visited (black)
Output: No cycles (success) OR Error with cycle path
```

**Example Cycle Detection**:
```
Task 1 → Task 2 → Task 3 → Task 1
                    ↑____________|
                    CYCLE DETECTED!

Error: "Cycle detected: Task 1 → Task 2 → Task 3 → Task 1"
```

### 2.3 Calculate Blocks Relationships

```
Input: DependencyGraph with depends_on
Process:
  - For each task T:
    - For each dependency D in T.depends_on:
      - Add T to D.blocks
Output: DependencyGraph with both depends_on and blocks
```

**Example**:
```yaml
Task 1:
  depends_on: []
  blocks: [2, 3]   # Calculated

Task 2:
  depends_on: [1]
  blocks: [4]      # Calculated

Task 3:
  depends_on: [1]
  blocks: [4]      # Calculated

Task 4:
  depends_on: [2, 3]
  blocks: []       # Calculated
```

### 2.4 Determine Ready Tasks

```
Input: DependencyGraph
Process:
  - For each task T:
    - If T.depends_on is empty:
      - Mark as READY (🟢)
    - Else:
      - Mark as BLOCKED (🔴)
Output: Tasks marked with status
```

### 2.5 Calculate Critical Path

```
Input: DependencyGraph with estimated_hours
Algorithm: Longest Path in DAG
Process:
  - Topological sort of tasks
  - For each task in order:
    - Calculate earliest start time
    - Calculate latest start time
  - Tasks where earliest == latest → Critical Path
Output: List of task IDs on critical path
```

---

## Phase 3: Content Generation

### 3.1 Generate Priming Prompts

```
For each Task:
  Input: Task object

  Process:
    1. Infer Role:
       - If technologies contains ["Python", "FastAPI"] → "Expert Python/FastAPI Developer"
       - If technologies contains ["React", "TypeScript"] → "Expert React/TypeScript Developer"
       - Else → "Expert Software Developer"

    2. Extract Context:
       - Files to create: from task.files_to_create
       - Files to modify: from task.files_to_modify
       - Technologies: from task.technologies
       - Patterns: from task.patterns_to_follow

    3. Build Constraints:
       - Always: "Follow project coding standards"
       - Always: "Test coverage > 90%"
       - If patterns: "Follow pattern: {pattern_path}"
       - Custom: from task.constraints

    4. Build Success Criteria:
       - Always: "All tests pass"
       - Always: "All todo items completed"
       - Custom: from task.success_criteria

    5. Format Prompt:
       Template:
       ```
       ROLE: {role}
       TASK: {task.objective or task.description}

       CONTEXT:
         - Files to create: {files_to_create}
         - Files to modify: {files_to_modify}
         - Technologies: {technologies}
         - Patterns to follow: {patterns}

       CONSTRAINTS:
         - {constraint_1}
         - {constraint_2}
         ...

       SUCCESS CRITERIA:
         - {criterion_1}
         - {criterion_2}
         ...
       ```

  Output: PrimingPrompt (string)
```

**Example**:
```
Input Task:
  name: "Database Schema"
  technologies: ["Python", "SQLAlchemy", "PostgreSQL"]
  files_to_create: ["models/user.py"]
  patterns_to_follow: ["/patterns/database-model.py"]

Output Prompt:
ROLE: Expert Python/SQLAlchemy Developer
TASK: Create database schema for user management

CONTEXT:
  - Files to create: models/user.py
  - Files to modify: None
  - Technologies: Python, SQLAlchemy, PostgreSQL
  - Patterns to follow: /patterns/database-model.py

CONSTRAINTS:
  - Follow pattern: /patterns/database-model.py
  - Test coverage > 90%
  - Use type hints
  - Follow PEP 8

SUCCESS CRITERIA:
  - All tests pass
  - Migration reversible
  - Indexes properly defined
```

### 3.2 Generate Context JSON

```
For each Task:
  Input: Task + PrimingPrompt

  Process:
    Build JSON with all metadata

  Output: context_{task_id}.json
```

**Format**: See ARCHITECTURE.md Context JSON Format

### 3.3 Generate PR Description

```
For each Task:
  Input: Task + PrimingPrompt + DependencyGraph

  Process:
    1. Build header:
       - Emoji status (🟢 Ready / 🔴 Blocked)
       - Task ID and name
       - Complexity, estimated hours, priority

    2. Build description section

    3. Include priming prompt in code block

    4. Format todo list as checklist

    5. Add dependency information:
       - Blocks: from graph
       - Blocked by: from graph

    6. List files to create/modify

    7. List patterns to follow

    8. Add success criteria as checklist

  Output: PR description (Markdown)
```

**Format**: See ARCHITECTURE.md PR Description Format

---

## Phase 4: Git & GitHub Operations

### 4.1 Prepare Git Environment

```
Process:
  1. Verify git repo initialized
  2. Verify remote configured
  3. Fetch latest from remote
  4. Identify base branch (main, master, develop)
  5. Checkout base branch
  6. Pull latest changes
```

### 4.2 For Each Task: Create Branch

```
Input: Task
Process:
  1. Generate branch name:
     Format: pr-orchestrator/task-{id}-{slug}
     Slug: lowercase, hyphenated, max 50 chars
     Example: pr-orchestrator/task-1-database-schema

  2. Create branch from base:
     git checkout -b {branch_name}

  3. Verify branch created:
     git branch --show-current
```

### 4.3 For Each Task: Create Context Files

```
Input: Task + Context JSON + Priming Prompt
Process:
  1. Create directory:
     .pr-orchestrator/tasks/task_{id}/

  2. Write context.json:
     .pr-orchestrator/tasks/task_{id}/context.json

  3. Write README.md:
     .pr-orchestrator/tasks/task_{id}/README.md
     Contains:
     - Task overview
     - Quick start instructions
     - Priming prompt
     - Todo list

  4. Write DEPENDENCIES.md (if has dependencies):
     .pr-orchestrator/tasks/task_{id}/DEPENDENCIES.md
     Lists all dependency tasks and their status
```

### 4.4 For Each Task: Commit & Push

```
Process:
  1. Stage files:
     git add .pr-orchestrator/tasks/task_{id}/

  2. Commit:
     Message: "pr-orchestrator: Initialize Task {id} - {name}"
     git commit -m "{message}"

  3. Push with upstream:
     git push -u origin {branch_name}

  4. Capture push output:
     - Verify success
     - Extract remote URL
```

### 4.5 For Each Task: Create PR

```
Input: Task + Branch + PR Description
Process:
  1. Prepare PR data:
     - Title: "Task {id}: {name}"
     - Body: {pr_description}
     - Head: {branch_name}
     - Base: {base_branch}
     - Draft: {config.draft_by_default}

  2. Call GitHub API:
     POST /repos/{owner}/{repo}/pulls
     With PR data

  3. Capture response:
     - PR number
     - PR URL
     - PR state

  4. Save PR number to task:
     task.pr_number = {pr_number}
```

### 4.6 For Each Task: Apply Labels

```
Input: Task + PR number
Process:
  1. Prepare labels:
     Always: ["pr-orchestrator"]

     If task.status == READY:
       Add: ["ready-to-work"]

     If task.status == BLOCKED:
       Add: ["blocked"]
       For each dependency:
         Add: ["blocked-by:task-{dep_id}"]

     If task.priority:
       Add: ["priority:{task.priority}"]

  2. Apply labels via GitHub API:
     POST /repos/{owner}/{repo}/issues/{pr_number}/labels

  3. Verify labels applied
```

---

## Phase 5: Finalization

### 5.1 Save State

```
Input: List[Task with PR numbers] + Metadata
Process:
  1. Build state object:
     - Project info
     - Timestamp
     - Source file
     - Base branch
     - Tasks with statuses

  2. Write state.json:
     .pr-orchestrator/state.json

  3. Commit state file:
     git checkout {base_branch}
     git add .pr-orchestrator/state.json
     git commit -m "pr-orchestrator: Save project state"
     git push
```

### 5.2 Generate Summary Report

```
Input: List[Task] + DependencyGraph
Process:
  1. Count tasks by status:
     - Ready: X tasks
     - Blocked: Y tasks

  2. List ready tasks with PRs

  3. List blocked tasks with dependencies

  4. Show critical path

  5. Estimate total time:
     - Sequential: sum(all estimated_hours)
     - Parallel: critical_path_hours
     - Parallelism gain: (sequential - parallel) / sequential

  Output: Formatted report (stdout)
```

**Example Report**:
```
============================================================
🚀 PR ORCHESTRATOR - Project Initialized
============================================================

📊 Summary:
   Total Tasks: 8
   Ready Tasks: 2 (can start now)
   Blocked Tasks: 6 (waiting for dependencies)

🟢 Ready Tasks:
   - Task 1: Database Schema (PR #42) - 3.0h
     https://github.com/user/repo/pull/42

   - Task 5: Documentation (PR #46) - 2.0h
     https://github.com/user/repo/pull/46

🔴 Blocked Tasks:
   - Task 2: API Endpoints (PR #43) - 4.0h
     Blocked by: Task 1

   - Task 3: Frontend UI (PR #44) - 6.0h
     Blocked by: Task 2

   ... (4 more)

🎯 Critical Path:
   Task 1 → Task 2 → Task 3 → Task 4
   Total: 16.0h

⏱️ Time Estimates:
   Sequential: 26.0h
   Parallel: 16.0h (critical path)
   Parallelism Gain: 38.5%

📍 Next Steps:
   1. Agents can start on Task 1 and Task 5 immediately
   2. Monitor PR progress on GitHub
   3. When Task 1 merges, Task 2 will auto-unblock
   4. Track progress: cat .pr-orchestrator/state.json

🔗 View all PRs:
   https://github.com/user/repo/pulls?q=label:pr-orchestrator

============================================================
```

### 5.3 Output Next Steps

```
Output:
  1. Commands to view PRs
  2. Instructions for agents
  3. How to monitor progress
  4. Link to state.json
```

---

## 🔄 Error Handling

### Input Errors
```
- Invalid YAML/JSON → Clear parse error with line number
- Missing required field → List all missing fields
- Invalid dependency ID → "Task X depends on non-existent Task Y"
- Cycle detected → Show cycle path
```

### Git Errors
```
- Not a git repo → "Run 'git init' first"
- No remote → "Configure remote: git remote add origin <url>"
- Dirty working tree → "Commit or stash changes first"
- Branch already exists → "Branch {name} already exists, use --force to override"
```

### GitHub Errors
```
- Authentication failed → "Run 'gh auth login' first"
- Rate limit → "GitHub API rate limit reached, wait {time}"
- PR already exists → "PR for branch {name} already exists (#123)"
- Permission denied → "Insufficient permissions to create PRs"
```

### Recovery
```
All operations are designed to be:
- Idempotent: Can re-run safely
- Atomic where possible
- State-tracked: Can resume from interruption

If interrupted:
  1. Check state.json
  2. Re-run command
  3. System skips completed tasks
  4. Continues from where it stopped
```

---

## 🎛️ Configuration Options

```yaml
# .pr-orchestrator.yaml

git:
  base_branch: main         # or master, develop
  branch_prefix: pr-orchestrator
  auto_push: true

github:
  platform: github          # or gitlab, bitbucket
  draft_by_default: true
  auto_assign: false

labels:
  enable: true
  ready: "ready-to-work"
  blocked: "blocked"
  prefix_blocked_by: "blocked-by:"
  prefix_priority: "priority:"

prompts:
  template: default         # or custom path
  role_inference: auto      # or manual

monitoring:
  enable_dashboard: false
  webhook_url: null
  slack_notifications: false
```

---

## 🔍 Monitoring & Observability

### State Tracking
```
state.json contains:
  - Current status of each task
  - PR numbers and URLs
  - Timestamps
  - Dependency resolution status
```

### Logs
```
All operations logged to:
  .pr-orchestrator/logs/orchestrator.log

Log levels:
  - DEBUG: Detailed operation info
  - INFO: Major milestones
  - WARNING: Recoverable issues
  - ERROR: Failed operations
```

### Metrics
```
Tracked:
  - Time per phase
  - Tasks created
  - PRs created
  - Errors encountered
  - Parallelism achieved
```

---

## 🚀 Advanced Workflows

### Resume After Interruption
```bash
# If interrupted during initialization
pr-orchestrator resume
# → Reads state.json
# → Continues from last completed task
```

### Update After Task Merge
```bash
# When a task is merged
pr-orchestrator update
# → Reads PR statuses from GitHub
# → Updates state.json
# → Auto-removes "blocked-by" labels from unblocked tasks
# → Adds "ready-to-work" to newly unblocked tasks
```

### Monitor Progress
```bash
pr-orchestrator status
# → Shows current state
# → Lists ready vs blocked
# → Shows completion percentage
```

---

**Version**: 1.0.0
**Status**: Workflow Design
**Next**: Business Logic implementation details
