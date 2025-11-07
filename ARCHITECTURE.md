# 🏗️ PR Orchestrator - Architecture

## 📋 Vue d'Ensemble

**PR Orchestrator** est une couche de coordination qui transforme les tâches analysées (par TaskMaster ou autre) en Pull Requests GitHub structurées, permettant à plusieurs agents de travailler en parallèle.

## 🎯 Position dans le Workflow Global

```
┌─────────────────────────────────────────────────────────────┐
│ 1. IDÉE → TRD → TDD                                         │
│    "Ajouter système de notifications"                       │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. TASKMASTER (externe)                                     │
│    - Analyse codebase (AST, imports, complexité)            │
│    - Calcul de complexité (cyclomatic, lignes, deps)        │
│    - Découpage intelligent en tâches/sous-tâches            │
│    - Estimation de durée                                    │
│    - Détection de dépendances                               │
│    OUTPUT: tasks.yaml / tasks.json                          │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. PR ORCHESTRATOR (ce système)                            │
│    INPUT: tasks.yaml                                        │
│                                                             │
│    Processus:                                               │
│    ├─ Parse tasks.yaml                                      │
│    ├─ Génère priming prompts contextuels                   │
│    ├─ Crée branches GitHub                                  │
│    ├─ Crée PRs avec contexte complet                       │
│    ├─ Configure labels de dépendances                      │
│    └─ Génère contexte JSON par tâche                       │
│                                                             │
│    OUTPUT: PRs GitHub prêtes                                │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. AGENTS (Claude, humains, etc.)                          │
│    - Query PRs disponibles (label: ready-to-work)          │
│    - Checkout branche                                       │
│    - Lis contexte + priming prompt                         │
│    - Code                                                   │
│    - Merge                                                  │
│    - Coordination automatique via GitHub                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🧩 Composants Principaux

### 1. Input Parser
**Rôle**: Parse le fichier tasks.yaml/json généré par TaskMaster

**Responsabilités**:
- Validation du format
- Extraction des tâches
- Extraction des métadonnées (complexité, dépendances, etc.)
- Transformation en structure interne

**Input**: `tasks.yaml` ou `tasks.json`
**Output**: `List[Task]` (structure interne)

---

### 2. Dependency Resolver
**Rôle**: Calcule le graph de dépendances et détermine les tâches ready

**Responsabilités**:
- Construire le graph de dépendances
- Détecter les cycles (erreur si présent)
- Calculer l'inverse: "blocks" depuis "depends_on"
- Marquer les tâches ready vs blocked
- Calculer le chemin critique

**Input**: `List[Task]`
**Output**: `DependencyGraph`

---

### 3. Priming Prompt Generator
**Rôle**: Génère des priming prompts contextuels pour chaque tâche

**Responsabilités**:
- Analyser les métadonnées de la tâche
- Inférer le rôle (Expert Python, Expert React, etc.)
- Construire un prompt structuré
- Inclure les contraintes, success criteria, contexte

**Input**: `Task`
**Output**: `PrimingPrompt` (string)

**Template**:
```
ROLE: [Inféré depuis technologies]
TASK: [Objectif de la tâche]

CONTEXT:
  - Files to create: [...]
  - Files to modify: [...]
  - Technologies: [...]
  - Patterns to follow: [...]

CONSTRAINTS:
  - [Contrainte 1]
  - [Contrainte 2]

SUCCESS CRITERIA:
  - [Critère 1]
  - [Critère 2]
```

---

### 4. Branch Manager
**Rôle**: Gère la création et configuration des branches Git

**Responsabilités**:
- Créer branches avec naming convention
- Checkout sur chaque branche
- Créer fichiers de contexte
- Commit initial avec metadata
- Push vers remote

**Naming Convention**: `pr-orchestrator/task-{id}-{slug}`

**Exemple**: `pr-orchestrator/task-1-database-schema`

---

### 5. PR Creator
**Rôle**: Crée les Pull Requests sur GitHub

**Responsabilités**:
- Générer titre et description PR
- Inclure contexte complet dans description
- Configurer labels (ready-to-work, blocked, etc.)
- Créer PR draft si configuré
- Gérer les annotations de dépendances

**PR Description Format**:
```markdown
# 🎯 Task {id}: {name}

**Status**: [🟢 Ready | 🔴 Blocked]
**Complexity**: {complexity}
**Estimated Hours**: {estimated_hours}h
**Priority**: {priority}

---

## 📋 Description

{description}

---

## 🤖 Priming Prompt

```
{priming_prompt}
```

---

## ✅ Todo List

{todo_items}

---

## 🔗 Dependencies

**Blocks**: Task {ids}
**Blocked by**: Task {ids}

---

## 📁 Files

### To Create
- `file1.py`
- `file2.py`

### To Modify
- `file3.py`

---

## 🎨 Patterns to Follow
- `/patterns/xxx.py`

---

## 🎯 Success Criteria
- [ ] All tests pass
- [ ] All todos completed
- [ ] Code follows patterns
```

---

### 6. Label Manager
**Rôle**: Gère les labels GitHub pour coordination

**Responsabilités**:
- Créer labels si n'existent pas
- Appliquer labels aux PRs
- Gérer ready-to-work / blocked
- Gérer dependencies (blocked-by:task-X)

**Labels Types**:
- `pr-orchestrator` - Toutes les PRs du système
- `ready-to-work` - Tâches disponibles
- `blocked` - Tâches bloquées
- `blocked-by:task-{id}` - Dépendance spécifique
- `priority:{level}` - Niveau de priorité

---

### 7. Context Generator
**Rôle**: Génère les fichiers de contexte JSON pour les agents

**Responsabilités**:
- Créer context.json par tâche
- Inclure toutes les métadonnées
- Format facile à parser pour agents

**Context JSON Format**:
```json
{
  "task_id": 1,
  "name": "Database Schema",
  "description": "...",
  "priming_prompt": "...",
  "complexity": 45,
  "estimated_hours": 3.0,
  "files_to_create": [...],
  "files_to_modify": [...],
  "patterns_to_follow": [...],
  "dependencies": [...],
  "blocks": [...],
  "technologies": [...],
  "success_criteria": [...]
}
```

---

### 8. State Manager
**Rôle**: Gère l'état du projet orchestré

**Responsabilités**:
- Sauvegarder l'état (state.json)
- Tracker les PRs créées
- Tracker le statut de chaque tâche
- Permettre la reprise après interruption

**State JSON Format**:
```json
{
  "project": {
    "name": "...",
    "created_at": "...",
    "source": "tasks.yaml"
  },
  "tasks": [
    {
      "id": 1,
      "name": "...",
      "branch": "pr-orchestrator/task-1-...",
      "pr_number": 42,
      "status": "ready",
      "dependencies": [...]
    }
  ]
}
```

---

## 🔄 Workflow Interne

### Phase 1: Initialization
```
1. Load tasks.yaml
2. Parse → List[Task]
3. Validate format
4. Build DependencyGraph
5. Detect cycles (error if found)
6. Mark tasks as ready/blocked
```

### Phase 2: Preparation
```
For each Task:
  1. Generate priming prompt
  2. Prepare context JSON
  3. Prepare PR description
```

### Phase 3: Git Operations
```
For each Task:
  1. Create branch
  2. Checkout branch
  3. Create context files
  4. Commit with metadata
  5. Push to remote
```

### Phase 4: PR Creation
```
For each Task:
  1. Create PR via GitHub API
  2. Apply labels
  3. Link dependencies
  4. Save PR number to state
```

### Phase 5: Finalization
```
1. Save state.json
2. Generate summary report
3. Output ready tasks
4. Output blocked tasks
```

---

## 🎨 Architecture en Couches

```
┌─────────────────────────────────────────┐
│         CLI / Interface                  │
│  (pr-orchestrator init tasks.yaml)      │
└───────────────┬─────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│      Orchestration Layer                │
│  - Coordinator                          │
│  - State Manager                        │
└───────────────┬─────────────────────────┘
                │
        ┌───────┴───────┐
        ▼               ▼
┌──────────────┐  ┌──────────────┐
│ Input Layer  │  │ Output Layer │
│ - Parser     │  │ - Branch Mgr │
│ - Validator  │  │ - PR Creator │
│ - Dep Graph  │  │ - Label Mgr  │
└──────────────┘  └──────────────┘
        │               │
        └───────┬───────┘
                ▼
┌─────────────────────────────────────────┐
│         Core Logic                      │
│  - Priming Prompt Generator             │
│  - Context Generator                    │
│  - Dependency Resolver                  │
└─────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│       External Systems                  │
│  - GitHub API                           │
│  - Git CLI                              │
│  - File System                          │
└─────────────────────────────────────────┘
```

---

## 🗂️ Structure de Fichiers

```
pr-orchestrator/
├── docs/
│   ├── ARCHITECTURE.md         (ce fichier)
│   ├── WORKFLOW.md             (workflow détaillé)
│   ├── BUSINESS_LOGIC.md       (logique métier)
│   └── API.md                  (API externe si nécessaire)
│
├── contracts/
│   ├── task.yaml               (format input TaskMaster)
│   ├── config.yaml             (configuration)
│   ├── state.json              (format de state)
│   └── context.json            (format de context)
│
├── examples/
│   ├── simple-project/
│   │   └── tasks.yaml          (exemple simple)
│   ├── complex-project/
│   │   └── tasks.yaml          (exemple complexe avec deps)
│   └── reddit-notifications/
│       └── tasks.yaml          (exemple réel)
│
├── src/                        (à créer - implémentation)
│   ├── parsers/
│   ├── generators/
│   ├── managers/
│   └── orchestrator.py
│
├── tests/                      (à créer - tests)
│
└── README.md                   (guide utilisateur)
```

---

## 🔌 Points d'Extension

### 1. Input Formats
- **Actuel**: tasks.yaml, tasks.json
- **Extension**: Support pour d'autres formats (TOML, CSV, etc.)

### 2. Platforms
- **Actuel**: GitHub
- **Extension**: GitLab, Bitbucket, Azure DevOps

### 3. Priming Prompt Templates
- **Actuel**: Template générique
- **Extension**: Templates par type de tâche, par langage, par framework

### 4. Monitoring
- **Extension**: Dashboard web, webhooks, notifications

---

## 🎯 Design Principles

### 1. Single Responsibility
Chaque composant a une responsabilité claire et unique.

### 2. Separation of Concerns
Input parsing ≠ PR creation ≠ Dependency resolution

### 3. Fail Fast
Validation stricte à l'entrée. Erreurs explicites.

### 4. Idempotence
Relancer la commande ne crée pas de doublons.

### 5. Observable
State tracking, logs, summary reports.

### 6. Extensible
Architecture en plugins pour support multi-plateformes.

---

## 📊 Flux de Données

```
tasks.yaml
    ↓ [Parser]
List[Task]
    ↓ [Dependency Resolver]
DependencyGraph
    ↓ [Priming Prompt Generator]
List[Task + Prompt]
    ↓ [Branch Manager]
Git Branches
    ↓ [Context Generator]
Context JSON Files
    ↓ [PR Creator]
GitHub PRs
    ↓ [Label Manager]
Labeled PRs
    ↓ [State Manager]
state.json
```

---

## 🚀 Future: Claude Skill

Ce système sera packagé comme un **Claude Skill** permettant à Claude de:

1. Recevoir un `tasks.yaml`
2. Orchestrer automatiquement via PRs
3. Coordonner plusieurs instances Claude en parallèle
4. Monitorer la progression
5. Rapporter le status

**Usage Futur**:
```bash
claude orchestrate tasks.yaml
# → Crée toutes les PRs
# → Lance agents en parallèle si configuré
# → Monitore et rapporte
```

---

**Version**: 1.0.0
**Status**: Architecture Design
**Next**: Business Logic détaillée
