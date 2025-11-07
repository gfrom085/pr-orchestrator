# 🧠 PR Orchestrator - Business Logic

## 📋 Vue d'Ensemble

Ce document détaille la logique métier spécifique de PR Orchestrator, les règles de gestion, les algorithmes, et les décisions d'implémentation.

---

## 1. Task Status Management

### 1.1 Status States

```
READY (🟢)
  - Task has no dependencies
  - OR all dependencies are COMPLETED
  - Can be worked on immediately

BLOCKED (🔴)
  - Task has at least one dependency
  - At least one dependency is not COMPLETED
  - Cannot be worked on yet

IN_PROGRESS (🟡)
  - Agent has claimed the task
  - Work is ongoing
  - PR is open and active

COMPLETED (✅)
  - PR has been merged
  - Task is done
  - Can unblock dependent tasks

FAILED (❌)
  - PR was closed without merge
  - OR tests failed repeatedly
  - Requires intervention
```

### 1.2 Status Transitions

```
READY → IN_PROGRESS
  Trigger: Agent claims task (checks out PR)
  Action: Add label "in-progress"

IN_PROGRESS → COMPLETED
  Trigger: PR merged
  Action:
    - Mark task as completed
    - Update all dependent tasks
    - Remove "blocked-by:task-X" from dependents
    - Check if dependents become READY

IN_PROGRESS → FAILED
  Trigger: PR closed OR tests failed 3+ times
  Action:
    - Mark task as failed
    - Notify stakeholders
    - Block all dependent tasks

BLOCKED → READY
  Trigger: All dependencies completed
  Action:
    - Remove "blocked" label
    - Add "ready-to-work" label
    - Notify available agents
```

---

## 2. Dependency Resolution Logic

### 2.1 Dependency Types

#### Hard Dependencies
```
Task B depends on Task A (hard)
  - Task B CANNOT start until A is COMPLETED
  - Default type
  - Enforced via labels

Example:
  Task 2 (API) depends on Task 1 (Database)
  → Cannot create API without database schema
```

#### Soft Dependencies (Future)
```
Task B prefers Task A completed (soft)
  - Task B CAN start without A
  - But may need rework if A changes
  - Warning label

Example:
  Task 3 (UI) soft-depends on Task 2 (API)
  → Can mock API, but better to have real API
```

#### Mutual Exclusions (Future)
```
Task A and Task B cannot run simultaneously
  - Conflict on same files
  - Need sequential execution
  - Lock mechanism

Example:
  Task 2 and Task 3 both modify config.py
  → Must be sequential
```

### 2.2 Dependency Graph Algorithms

#### Cycle Detection (Depth-First Search)

```python
# Pseudocode
def detect_cycles(graph):
    colors = {}  # white, gray, black
    path = []

    for node in graph.nodes:
        if colors[node] == white:
            if dfs_visit(node, colors, path):
                return True, path  # Cycle found

    return False, []  # No cycle

def dfs_visit(node, colors, path):
    colors[node] = gray
    path.append(node)

    for child in node.dependencies:
        if colors[child] == gray:
            # Back edge = cycle
            cycle_path = path[path.index(child):] + [child]
            return True

        if colors[child] == white:
            if dfs_visit(child, colors, path):
                return True

    colors[node] = black
    path.pop()
    return False
```

**Example**:
```
Graph:
  1 → 2 → 3
      ↑   |
      |___| ← Cycle!

Detection:
  Visit 1: white → gray
  Visit 2: white → gray
  Visit 3: white → gray
  Visit 2 (from 3): gray! → CYCLE DETECTED
  Path: [2, 3, 2]
```

#### Topological Sort

```python
# Pseudocode
def topological_sort(graph):
    in_degree = {}
    queue = []
    result = []

    # Calculate in-degrees
    for node in graph.nodes:
        in_degree[node] = len(node.dependencies)
        if in_degree[node] == 0:
            queue.append(node)

    # Process nodes
    while queue:
        node = queue.pop(0)
        result.append(node)

        for dependent in node.blocks:
            in_degree[dependent] -= 1
            if in_degree[dependent] == 0:
                queue.append(dependent)

    if len(result) != len(graph.nodes):
        raise CycleError("Graph has cycles")

    return result
```

**Use Case**: Order tasks for sequential execution if needed

#### Critical Path Calculation

```python
# Pseudocode
def calculate_critical_path(graph):
    # Longest path in DAG
    topo_order = topological_sort(graph)

    earliest_start = {}
    latest_start = {}

    # Forward pass (earliest start)
    for node in topo_order:
        if not node.dependencies:
            earliest_start[node] = 0
        else:
            earliest_start[node] = max(
                earliest_start[dep] + dep.estimated_hours
                for dep in node.dependencies
            )

    # Backward pass (latest start)
    max_time = max(earliest_start.values())
    for node in reversed(topo_order):
        if not node.blocks:
            latest_start[node] = max_time - node.estimated_hours
        else:
            latest_start[node] = min(
                latest_start[dep] - node.estimated_hours
                for dep in node.blocks
            )

    # Critical path: nodes where earliest == latest
    critical_path = [
        node for node in graph.nodes
        if earliest_start[node] == latest_start[node]
    ]

    return critical_path, max_time
```

**Use Case**: Identify bottleneck tasks, estimate minimum project duration

---

## 3. Priming Prompt Generation Logic

### 3.1 Role Inference Rules

```python
def infer_role(technologies: List[str]) -> str:
    # Priority order matters

    # Backend frameworks
    if "FastAPI" in technologies and "Python" in technologies:
        return "Expert Python/FastAPI Backend Developer"
    if "Django" in technologies:
        return "Expert Python/Django Developer"
    if "Flask" in technologies:
        return "Expert Python/Flask Developer"
    if "Node.js" in technologies and "Express" in technologies:
        return "Expert Node.js/Express Developer"

    # Frontend frameworks
    if "React" in technologies and "TypeScript" in technologies:
        return "Expert React/TypeScript Frontend Developer"
    if "React" in technologies:
        return "Expert React Developer"
    if "Vue" in technologies:
        return "Expert Vue.js Developer"
    if "Angular" in technologies:
        return "Expert Angular Developer"

    # Full-stack
    if "React" in technologies and "FastAPI" in technologies:
        return "Expert Full-Stack Developer (React/FastAPI)"

    # Database
    if "PostgreSQL" in technologies or "MySQL" in technologies:
        return "Expert Database Developer"

    # DevOps
    if "Docker" in technologies or "Kubernetes" in technologies:
        return "Expert DevOps Engineer"

    # Mobile
    if "React Native" in technologies:
        return "Expert React Native Developer"

    # Data/ML
    if "PyTorch" in technologies or "TensorFlow" in technologies:
        return "Expert ML Engineer"

    # Generic
    if "Python" in technologies:
        return "Expert Python Developer"
    if "TypeScript" in technologies or "JavaScript" in technologies:
        return "Expert TypeScript/JavaScript Developer"

    # Default
    return "Expert Software Developer"
```

### 3.2 Constraint Generation Rules

```python
def generate_constraints(task: Task) -> List[str]:
    constraints = []

    # Always include
    constraints.append("Follow project coding standards")
    constraints.append("Test coverage > 90%")
    constraints.append("All code must be type-safe")

    # Pattern constraints
    if task.patterns_to_follow:
        for pattern in task.patterns_to_follow:
            constraints.append(f"Follow pattern: {pattern}")

    # Technology-specific
    if "Python" in task.technologies:
        constraints.append("Follow PEP 8 style guide")
        constraints.append("Use type hints (mypy --strict)")

    if "TypeScript" in task.technologies:
        constraints.append("Enable strict mode")
        constraints.append("No 'any' types without justification")

    if "React" in task.technologies:
        constraints.append("Use functional components and hooks")
        constraints.append("Ensure accessibility (WCAG 2.1 AA)")

    # File operation constraints
    if task.files_to_modify:
        constraints.append("Preserve existing functionality when modifying files")
        constraints.append("Add tests for modified behavior")

    # Complexity constraints
    if task.complexity > 70:
        constraints.append("Break down into smaller functions (max 50 lines each)")
        constraints.append("Document complex logic with comments")

    # Custom constraints from task
    if task.constraints:
        constraints.extend(task.constraints)

    return constraints
```

### 3.3 Success Criteria Generation

```python
def generate_success_criteria(task: Task) -> List[str]:
    criteria = []

    # Always include
    criteria.append("All tests pass (pytest/jest)")
    criteria.append("All todo items completed")
    criteria.append("Code follows patterns and standards")

    # Type checking
    if "Python" in task.technologies:
        criteria.append("mypy --strict passes with no errors")
    if "TypeScript" in task.technologies:
        criteria.append("tsc compiles with no errors")

    # Linting
    criteria.append("Linter passes (flake8/eslint)")

    # Coverage
    criteria.append("Test coverage > 90% for new code")

    # Documentation
    if task.files_to_create:
        criteria.append("All new functions have docstrings/JSDoc")

    # Integration
    if task.dependencies:
        criteria.append("Integration with dependencies verified")

    # Performance
    if "database" in task.name.lower() or "Database" in task.name:
        criteria.append("Database queries optimized (indexes, N+1 queries avoided)")

    if "api" in task.name.lower() or "API" in task.name:
        criteria.append("API response time < 200ms for typical requests")

    # Custom criteria
    if task.success_criteria:
        criteria.extend(task.success_criteria)

    return criteria
```

---

## 4. Complexity & Estimation Logic

### 4.1 Complexity Interpretation

```
Complexity Score (from TaskMaster):
  0-20:   Trivial (simple changes, no logic)
  21-40:  Easy (basic logic, few dependencies)
  41-60:  Medium (moderate logic, some complexity)
  61-80:  Hard (complex logic, many dependencies)
  81-100: Very Hard (architectural changes, high risk)
```

### 4.2 Estimation Adjustments

```python
def adjust_estimation(base_hours: float, task: Task) -> float:
    multiplier = 1.0

    # Complexity adjustment
    if task.complexity > 80:
        multiplier *= 1.5  # Very hard tasks take longer
    elif task.complexity < 30:
        multiplier *= 0.8  # Easy tasks faster than estimated

    # Dependency count adjustment
    dep_count = len(task.dependencies)
    if dep_count > 3:
        multiplier *= 1.2  # Many dependencies = coordination overhead

    # File count adjustment
    file_count = len(task.files_to_create) + len(task.files_to_modify)
    if file_count > 10:
        multiplier *= 1.3  # Many files = more work

    # Technology unfamiliarity (if available)
    # (Would require agent skill tracking)

    return base_hours * multiplier
```

### 4.3 Parallelism Calculation

```python
def calculate_parallelism_potential(graph: DependencyGraph) -> dict:
    levels = build_dependency_levels(graph)

    potential = {
        "total_tasks": len(graph.nodes),
        "dependency_levels": len(levels),
        "max_parallel_tasks": max(len(level) for level in levels),
        "avg_parallel_tasks": sum(len(level) for level in levels) / len(levels),
        "sequential_hours": sum(task.estimated_hours for task in graph.nodes),
        "parallel_hours": sum(
            max(task.estimated_hours for task in level)
            for level in levels
        ),
        "efficiency_gain": 0.0
    }

    if potential["sequential_hours"] > 0:
        potential["efficiency_gain"] = (
            1 - potential["parallel_hours"] / potential["sequential_hours"]
        ) * 100

    return potential

def build_dependency_levels(graph: DependencyGraph) -> List[List[Task]]:
    levels = []
    remaining = set(graph.nodes)

    while remaining:
        # Find tasks with no dependencies in remaining set
        current_level = [
            task for task in remaining
            if all(dep not in remaining for dep in task.dependencies)
        ]

        if not current_level:
            raise CycleError("Cycle detected in dependency graph")

        levels.append(current_level)
        remaining -= set(current_level)

    return levels
```

**Example**:
```
Tasks:
  1: 3h, deps=[]
  2: 4h, deps=[1]
  3: 2h, deps=[1]
  4: 5h, deps=[2,3]

Levels:
  Level 0: [Task 1] → 3h
  Level 1: [Task 2, Task 3] → max(4h, 2h) = 4h
  Level 2: [Task 4] → 5h

Sequential: 3+4+2+5 = 14h
Parallel: 3+4+5 = 12h
Efficiency Gain: (14-12)/14 = 14.3%
```

---

## 5. Label Management Logic

### 5.1 Label Lifecycle

```
Creation:
  - "pr-orchestrator" → Added to all PRs
  - "ready-to-work" → Added if task.status == READY
  - "blocked" → Added if task.status == BLOCKED
  - "blocked-by:task-X" → Added for each dependency X
  - "priority:N" → Added based on task.priority

Updates:
  - When dependency merged:
    - Remove "blocked-by:task-X" for that dependency
    - If no more "blocked-by" labels:
      - Remove "blocked"
      - Add "ready-to-work"

  - When agent claims task:
    - Add "in-progress"
    - Optional: assign agent to PR

  - When PR merged:
    - Add "completed"
    - Remove all status labels

Cleanup:
  - After PR merged, labels can be archived
  - State tracked in state.json
```

### 5.2 Label Naming Convention

```
Format: {category}:{value}

Categories:
  - status: ready-to-work, blocked, in-progress, completed
  - dependency: blocked-by:task-{id}
  - priority: priority:{level}
  - type: type:{backend|frontend|devops|docs}
  - complexity: complexity:{trivial|easy|medium|hard|very-hard}

Examples:
  - pr-orchestrator
  - ready-to-work
  - blocked-by:task-1
  - priority:high
  - type:backend
  - complexity:medium
```

---

## 6. Branch Naming Logic

### 6.1 Branch Name Generation

```python
def generate_branch_name(task: Task, config: Config) -> str:
    prefix = config.branch_prefix  # "pr-orchestrator"
    task_id = task.id
    slug = generate_slug(task.name)

    return f"{prefix}/task-{task_id}-{slug}"

def generate_slug(name: str) -> str:
    # Convert to lowercase
    slug = name.lower()

    # Replace spaces with hyphens
    slug = slug.replace(" ", "-")

    # Remove special characters
    slug = re.sub(r'[^a-z0-9-]', '', slug)

    # Remove consecutive hyphens
    slug = re.sub(r'-+', '-', slug)

    # Trim hyphens from ends
    slug = slug.strip('-')

    # Limit length
    if len(slug) > 50:
        slug = slug[:50].rstrip('-')

    return slug
```

**Examples**:
```
Task name: "Database Schema & Models"
→ pr-orchestrator/task-1-database-schema-models

Task name: "API Endpoints (User Management)"
→ pr-orchestrator/task-2-api-endpoints-user-management

Task name: "Frontend: Dashboard UI Component"
→ pr-orchestrator/task-3-frontend-dashboard-ui-component
```

---

## 7. Error Recovery Logic

### 7.1 Idempotency Strategies

```python
def create_branch_idempotent(task: Task) -> str:
    branch_name = generate_branch_name(task)

    # Check if branch exists
    if branch_exists(branch_name):
        # Check if it's from this orchestration run
        if is_our_branch(branch_name):
            # Safe to reuse
            checkout(branch_name)
            return branch_name
        else:
            # Conflict: branch exists from elsewhere
            raise BranchConflictError(
                f"Branch {branch_name} already exists. "
                f"Use --force to override or rename task."
            )

    # Create new branch
    create_branch(branch_name)
    return branch_name

def create_pr_idempotent(task: Task, branch: str) -> int:
    # Check if PR already exists for this branch
    existing_pr = find_pr_by_branch(branch)

    if existing_pr:
        # Check if it's from this orchestration run
        if has_label(existing_pr, "pr-orchestrator"):
            # Safe to reuse
            task.pr_number = existing_pr.number
            return existing_pr.number
        else:
            # Conflict: PR exists but not ours
            raise PRConflictError(
                f"PR already exists for branch {branch} (#{existing_pr.number}). "
                f"Use --force to override."
            )

    # Create new PR
    pr = create_pr(task, branch)
    task.pr_number = pr.number
    return pr.number
```

### 7.2 State Recovery

```python
def resume_from_interruption(state_file: str):
    # Load previous state
    state = load_state(state_file)

    # Identify completed tasks
    completed = [
        task for task in state.tasks
        if task.pr_number is not None
    ]

    # Identify remaining tasks
    remaining = [
        task for task in state.tasks
        if task.pr_number is None
    ]

    # Resume from first remaining task
    for task in remaining:
        try:
            process_task(task)
            save_state(state)  # Save after each task
        except Exception as e:
            log.error(f"Failed to process task {task.id}: {e}")
            # Continue with next task (or stop based on config)
```

---

## 8. Validation Rules

### 8.1 Input Validation

```python
def validate_task(task: dict) -> List[str]:
    errors = []

    # Required fields
    required = ["id", "name", "description"]
    for field in required:
        if field not in task:
            errors.append(f"Missing required field: {field}")

    # Type validations
    if "id" in task and not isinstance(task["id"], int):
        errors.append(f"Task ID must be integer, got {type(task['id'])}")

    if "estimated_hours" in task:
        hours = task["estimated_hours"]
        if not isinstance(hours, (int, float)) or hours <= 0:
            errors.append(f"estimated_hours must be positive number")

    if "complexity" in task:
        complexity = task["complexity"]
        if not isinstance(complexity, int) or not (0 <= complexity <= 100):
            errors.append(f"complexity must be 0-100")

    # Reference validations (done after all tasks loaded)
    if "dependencies" in task:
        if not isinstance(task["dependencies"], list):
            errors.append(f"dependencies must be a list")

    return errors
```

### 8.2 Cross-Task Validation

```python
def validate_all_tasks(tasks: List[Task]) -> List[str]:
    errors = []
    task_ids = {task.id for task in tasks}

    # Check unique IDs
    id_counts = Counter(task.id for task in tasks)
    duplicates = [id for id, count in id_counts.items() if count > 1]
    if duplicates:
        errors.append(f"Duplicate task IDs: {duplicates}")

    # Check dependency references
    for task in tasks:
        for dep_id in task.dependencies:
            if dep_id not in task_ids:
                errors.append(
                    f"Task {task.id} depends on non-existent task {dep_id}"
                )

    # Check for cycles
    graph = build_graph(tasks)
    has_cycle, cycle_path = detect_cycles(graph)
    if has_cycle:
        path_str = " → ".join(str(id) for id in cycle_path)
        errors.append(f"Dependency cycle detected: {path_str}")

    return errors
```

---

## 9. Future Extensions

### 9.1 Agent Assignment Logic

```python
# Future: Auto-assign tasks to available agents
def assign_tasks_to_agents(ready_tasks: List[Task], agents: List[Agent]):
    assignments = []

    for task in ready_tasks:
        # Find best agent
        best_agent = find_best_match(task, agents)

        if best_agent:
            # Assign via GitHub API
            assign_pr(task.pr_number, best_agent.github_username)
            assignments.append((task, best_agent))

    return assignments

def find_best_match(task: Task, agents: List[Agent]) -> Agent:
    # Score each agent
    scores = []
    for agent in agents:
        if not agent.is_available():
            continue

        score = 0

        # Technology match
        matching_techs = set(task.technologies) & set(agent.skills)
        score += len(matching_techs) * 10

        # Complexity match
        if agent.max_complexity >= task.complexity:
            score += 5

        # Workload balance
        score -= agent.current_tasks * 3

        scores.append((agent, score))

    # Return agent with highest score
    scores.sort(key=lambda x: x[1], reverse=True)
    return scores[0][0] if scores else None
```

### 9.2 Dynamic Re-planning

```python
# Future: Adjust plan based on actual completion times
def replan_based_on_actuals(state: State):
    # Compare estimated vs actual hours
    for task in state.completed_tasks:
        if task.actual_hours > task.estimated_hours * 1.5:
            # Task took much longer
            # Adjust estimates for similar tasks
            adjust_similar_tasks(task, state.remaining_tasks)
```

---

**Version**: 1.0.0
**Status**: Business Logic Design
**Next**: Contract definitions (YAML/JSON schemas)
