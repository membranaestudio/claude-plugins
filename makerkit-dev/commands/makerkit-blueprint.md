---
description: Generate architecture blueprint for MakerKit features using MCP analysis
argument-hint: Feature description (e.g., "Add student management")
---

# MakerKit Blueprint Generator

Generate architecture blueprints for MakerKit features. Uses MCP tools to analyze codebase and produces Ralph-ready implementation plans.

## Core Principles

- **MCP First**: Always use MCP tools before making architecture decisions
- **No Assumptions**: Ask clarifying questions for all ambiguities
- **Ralph-Ready Output**: Blueprints must be executable without human intervention
- **Follow Patterns**: Match existing MakerKit patterns exactly

---

## Phase 0: Prerequisites Verification

**Goal**: Verify MCP connection and context before proceeding

> **WARNING**: Do NOT skip this phase. ANY assumption = blueprint failure.

**Actions**:

1. **Verify MCP Connection** (MANDATORY):
   ```
   get_database_summary()    → Must return tables/enums
   find_complete_features()  → Must return feature list
   ```
   If either fails → STOP and inform: "MCP not connected. Cannot proceed."

2. **Verify Context Files Exist**:
   | File | Required | If Missing |
   |------|----------|------------|
   | `apps/web/CLAUDE.md` | Yes | Do not continue |
   | `apps/web/supabase/CLAUDE.md` | Yes | Do not continue |
   | `CLAUDE.md` (root) | Recommended | Warn user |

3. **Check Input Requirements**:
   - Feature description provided (via $ARGUMENTS)
   - If unclear, ask user before proceeding

**Output**: Confirmation that MCP works and context is available.

---

## Phase 1: Discovery

**Goal**: Understand what needs to be built

Initial request: $ARGUMENTS

**Actions**:

1. Create todo list with all phases
2. If feature unclear, ask user for:
   - What entity/data does this feature manage?
   - What problem are they solving?
   - Any constraints or requirements?
3. Identify:
   - **Entity name** (singular, e.g., "student", "payment")
   - **Account type** (personal or team)
   - **Operations needed** (create, read, update, delete, list)
4. Summarize understanding and confirm with user

**Output**: Clear feature definition with entity, account type, and operations.

---

## Phase 2: Codebase Exploration

**Goal**: Understand MakerKit patterns using MCP tools

**Actions**:

1. Launch 2 `makerkit-explorer` agents in parallel:

   **Agent 1 - Database & Patterns**:
   ```
   Use MCP tools to analyze:
   - get_database_summary() → existing tables, enums, functions
   - find_complete_features() → reference features to study
   - analyze_feature_pattern("<similar_feature>") → patterns to follow
   - get_all_enums() → existing enums to reuse

   Return: Database patterns, helper functions available, reference feature recommendation
   ```

   **Agent 2 - Routes & Components**:
   ```
   Use MCP tools to analyze:
   - get_app_routes({filter: "home"}) → route structure
   - get_server_actions() → action patterns
   - components_search("<keyword>") → reusable UI components

   Also read CLAUDE.md files:
   - apps/web/CLAUDE.md
   - apps/web/supabase/CLAUDE.md
   - packages/features/CLAUDE.md

   Return: Route patterns, component recommendations, CLAUDE.md key insights
   ```

2. Read all files identified by agents
3. Check if inventory exists: `docs/arquitectura/00-INVENTARIO-MAKERKIT.md`
   - If NO: Generate inventory first using makerkit-patterns skill
   - If YES: Read it for project decisions
4. Present comprehensive summary of findings

**Output**: MCP analysis results, reference feature, patterns to follow.

---

## Phase 3: Clarifying Questions

**Goal**: Resolve ALL ambiguities before designing

**CRITICAL**: DO NOT SKIP THIS PHASE.

**Actions**:

1. Review discovery + exploration findings
2. Identify underspecified aspects:

   **Data Model**:
   - What fields does the entity need?
   - Any relationships to existing tables?
   - What validations are required?
   - Any computed/derived fields?

   **Business Logic**:
   - What states can the entity have?
   - Any workflows or state transitions?
   - Permission requirements beyond basic CRUD?
   - Any triggers or side effects?

   **UI/UX**:
   - List view: table, cards, or custom?
   - Create/edit: modal, page, or inline?
   - Any bulk operations needed?
   - Search/filter requirements?

   **Integration**:
   - Connect to other features?
   - External APIs involved?
   - Notifications needed?

3. **Present all questions in organized list**
4. **Wait for answers before proceeding**

If user says "whatever you think is best":
- Provide recommendation based on MakerKit patterns
- Get explicit confirmation

**Output**: All questions answered, no ambiguities remain.

---

## Phase 4: Architecture Design

**Goal**: Generate complete Ralph-ready blueprint

**Actions**:

1. Launch 1-2 `makerkit-architect` agents:

   **Primary Architect**:
   ```
   Design complete feature architecture:
   - Database schema with RLS (use generate_rls_policy MCP tool)
   - Zod validation schemas
   - Server actions with enhanceAction
   - Page loaders with 'server-only'
   - Route structure following MakerKit patterns
   - Component props/interfaces

   Output format must be Ralph-ready with:
   - Copy-paste ready code (no placeholders)
   - Implementation checklist with verify commands
   - Completion criteria for Ralph
   ```

   **Optional - If complex feature, add second architect**:
   ```
   Review primary design for:
   - Security (RLS completeness)
   - Performance (indexes, query patterns)
   - MakerKit convention compliance

   Suggest improvements if needed
   ```

2. Review architecture output
3. Validate with MCP:
   ```
   validate_rls_policies("<table>") → check RLS completeness
   ```

4. Generate blueprint file: `docs/arquitectura/XX-<feature>.md`

**Blueprint must include**:

```markdown
# [Feature] Architecture Blueprint

## Context
- Inventory reference
- MCP tools used
- Reference features analyzed
- CLAUDE.md files consulted

## Requirements Summary
| Aspect | Decision |
|--------|----------|
| Entity | [name] |
| Account Type | personal / team |
| Operations | create, read, update, delete |

## Database Layer
- Complete SQL (CREATE TABLE, indexes, triggers)
- RLS policies (SELECT, INSERT, UPDATE, DELETE)
- Helper functions used

## Server Layer
- Zod schemas (Create, Update, Delete)
- Server actions with enhanceAction
- Page loaders with 'server-only'

## UI Layer
- Route structure
- Component props/interfaces
- @kit/ui components to use

## Implementation Checklist (Ralph-Ready)
| # | Task | File | Blocked By | Verify Command |
|---|------|------|------------|----------------|
| 1 | Create schema | path | - | test -f path |
...

## Completion Criteria
All verify commands must pass for FEATURE_COMPLETE.
```

5. Present blueprint summary to user
6. Confirm blueprint is complete and ready for Ralph

**Output**: Complete blueprint at `docs/arquitectura/XX-<feature>.md`

---

## After Blueprint Complete

Provide user with Ralph command:

```bash
/ralph-loop "Lee docs/arquitectura/XX-feature.md y CLAUDE.md. Implementa el blueprint completo siguiendo el Implementation Checklist. Ejecuta Verify Command después de cada paso. Output <promise>FEATURE_COMPLETE</promise> cuando todos pasen." --max-iterations 15 --completion-promise "FEATURE_COMPLETE"
```

---

## Important Notes

- **Always use MCP tools** - Never guess database structure
- **Follow CLAUDE.md** - These contain MakerKit-specific patterns
- **Personal vs Team accounts** - Critical decision, verify with user
- **RLS is non-negotiable** - Every table needs complete policies
- **Ralph-ready means no ambiguity** - Code must be copy-paste ready
