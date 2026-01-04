---
description: Generate Context Maps for MakerKit features using MCP analysis (consumes /product-spec:ideacion)
argument-hint: "[feature_name] or leave empty to start workflow"
---

# MakerKit Blueprint Generator

Generate **Context Maps** for MakerKit features. Not copy-paste code, but guides for autonomous development.

## Core Principles

- **Context, Not Code**: Generate maps of what to read, not code to copy
- **AskUserQuestion**: Always clarify before assuming
- **MCP Analysis**: Use MCP tools to verify what exists (no hallucinations)
- **Opus Autonomy**: Let Opus plan, develop, and iterate with freedom
- **Adaptive**: Works with existing structure or creates new one

---

## Phase 0: Project Context Detection

**Goal**: Understand project structure and configuration

**Actions**:

1. **Check for existing configuration**:
   ```
   Look for: .claude/makerkit-planner.local.md
   ```

   If exists → Read saved configuration (specs path, blueprints path, version)
   If not exists → Proceed to ask user

2. **Detect existing structure** (search for patterns):
   ```
   Specs patterns:
   - docs/producto/*/03-FEATURE-SPECS.md
   - docs/producto-*/03-FEATURE-SPECS.md
   - docs/specs/*.md
   - docs/*.md with "Feature:" headers

   Blueprint patterns:
   - docs/build/*/
   - docs/arquitectura/
   - docs/blueprints/
   ```

3. **AskUserQuestion based on findings**:

   **If specs found:**
   ```yaml
   question: "Encontre specs en [path]. ¿Usar este archivo?"
   header: "Specs"
   options:
     - label: "Si, usar ese archivo"
       description: "Continuar con los specs detectados"
     - label: "No, buscar en otro lugar"
       description: "Especificar path diferente"
     - label: "No tengo specs aun"
       description: "Ejecuta /product-spec:ideacion primero para generarlos"
   ```

   **If no specs found:**
   ```yaml
   question: "No encontre specs de producto. ¿Que hacemos?"
   header: "Specs"
   options:
     - label: "Ejecutar /product-spec:ideacion primero"
       description: "Generar specs de producto con el plugin product-spec"
     - label: "Tengo specs en otro lugar"
       description: "Especificar path manualmente"
     - label: "Crear feature sin specs"
       description: "Definir feature manualmente ahora"
   ```

4. **If no blueprints structure exists:**
   ```yaml
   question: "¿Donde guardar los blueprints?"
   header: "Output"
   options:
     - label: "docs/build/"
       description: "Carpeta 'build' orientada a accion"
     - label: "docs/blueprints/"
       description: "Carpeta explicita 'blueprints'"
     - label: "docs/arquitectura/"
       description: "Carpeta 'arquitectura' (espanol)"
     - label: "Otro"
       description: "Especificar path personalizado"
   ```

5. **Version structure:**
   ```yaml
   question: "¿Usar versionado en la estructura?"
   header: "Version"
   options:
     - label: "Si, con versiones"
       description: "Ej: docs/build/v1.0/feature/"
     - label: "No, carpeta unica"
       description: "Ej: docs/build/feature/"
   ```

6. **Save configuration** to `.claude/makerkit-planner.local.md`:
   ```yaml
   ---
   specs_path: "docs/producto"
   blueprints_path: "docs/build"
   use_versions: true
   current_version: "v1.0"
   ---

   # MakerKit Planner Configuration

   Saved configuration for this project.
   Edit this file to change paths.
   ```

**Output**: Configuration determined (specs path, blueprints path, version settings)

---

## Phase 1: Specs Loading & Feature Selection

**Goal**: Load specs and let user choose features

**Actions**:

1. **Read specs file** at configured path

2. **Parse features** from specs:
   - Look for `## Feature:` headers
   - Extract feature names, fields, operations
   - Note any `[DEFAULT]` or `[PENDIENTE]` markers

3. **Present features found**:
   ```
   Specs cargados de: [path]

   Features detectadas:
   1. [Feature A] - [brief description]
   2. [Feature B] - [brief description]
   ...
   ```

4. **AskUserQuestion - Feature selection**:
   ```yaml
   question: "¿Que features generar?"
   header: "Features"
   multiSelect: true
   options:
     - label: "Todas las features"
       description: "Generar blueprints para todas"
     - label: "Solo la primera (dependencia base)"
       description: "Empezar con la feature base"
     - label: "Seleccionar manualmente"
       description: "Elegir features especificas"
   ```

5. **Confirm order**:
   ```
   Orden de generacion: Feature A → Feature B → Feature C
   ¿Confirmar?
   ```

**Output**: List of features to generate, in order

---

## Phase 2: MCP Analysis

**Goal**: Understand MakerKit patterns in this codebase

**Actions**:

1. **Validate MCP connection**:
   ```
   get_database_summary()    → Must return tables/enums
   find_complete_features()  → Must return feature list
   ```
   If fails → STOP: "MCP no conectado. Verifica que el servidor MCP este activo."

2. **Launch exploration agents**:

   **Database & Patterns Agent**:
   ```
   - get_database_summary() → existing tables, enums, functions
   - find_complete_features() → reference features
   - get_all_enums() → available enums
   - search_database_functions() → helper functions
   ```

   **Routes & Components Agent**:
   ```
   - get_app_routes() → route structure
   - get_server_actions() → action patterns
   - components_search() → reusable UI
   - Read CLAUDE.md files
   ```

3. **Check if inventory exists** at blueprints path:
   - If NO: Flag for generation in Phase 4
   - If YES: Read for project decisions

**Output**: MCP analysis results, reference patterns

---

## Phase 3: Clarifying Questions (Per Feature)

**Goal**: Resolve ALL ambiguities using AskUserQuestion

**CRITICAL**: Every ambiguity now = Ralph failure later.

**For each feature, check specs and ask if unclear:**

### 3.1 Account Type
```yaml
question: "¿[Feature] es de cuenta personal o de equipo?"
header: "Account"
options:
  - label: "Team Account"
    description: "Datos compartidos entre miembros del equipo"
  - label: "Personal Account"
    description: "Solo el usuario ve sus propios datos"
```

### 3.2 Estados
```yaml
question: "¿Que estados puede tener [entidad]?"
header: "Estados"
options:
  - label: "active, archived"
    description: "Patron MakerKit default (soft delete)"
  - label: "Estados especificos"
    description: "Definir lista personalizada"
  - label: "Sin estados"
    description: "No aplica"
```

### 3.3 Borrado
```yaml
question: "¿Como manejar eliminacion de [entidad]?"
header: "Delete"
options:
  - label: "Soft delete"
    description: "Campo archived_at, mantiene historial"
  - label: "Hard delete"
    description: "Elimina permanentemente"
  - label: "No permitir"
    description: "Solo cambiar estado"
```

### 3.4 Permisos (if team account)
```yaml
question: "¿Que permisos por rol?"
header: "Permisos"
options:
  - label: "Owner/Admin: CRUD, Member: Read"
    description: "Patron tipico MakerKit"
  - label: "Solo Owner/Admin"
    description: "Members no ven"
  - label: "Todos CRUD"
    description: "Sin restriccion"
```

### 3.5 UI Layout
```yaml
question: "¿Vista principal de [entidad]?"
header: "Vista"
options:
  - label: "Tabla (DataTable)"
    description: "Lista con columnas, sorting"
  - label: "Cards"
    description: "Grid de tarjetas"
  - label: "Lista simple"
    description: "Items verticales"
```

**Handling "No se"**:
- Propose MakerKit default
- Mark as `[DEFAULT]` in blueprint
- Document reasoning

**Output**: All questions answered, decisions documented

---

## Phase 4: Blueprint Generation

**Goal**: Generate blueprints at configured paths

**Actions**:

### 4.1 Generate Inventory (if not exists)

Use template from `${CLAUDE_PLUGIN_ROOT}/templates/inventario.md`

Output at: `[blueprints_path]/[version]/00-INVENTARIO.md`

Content includes:
- MakerKit tables used vs not used
- Helper functions available
- Enums available
- Architecture decisions

### 4.2 Generate Blueprint per Feature

Use `makerkit-architect` agent for each feature.

Output at: `[blueprints_path]/[version]/XX-[feature]/BLUEPRINT.md`

Structure (Context Map - NOT copy-paste code):
```markdown
# [Feature] Context Map

> Generated: [date]
> Source: [specs_path]

## 1. Tu Misión
[What to implement - specs summary]

## 2. Qué Leer Antes de Codificar
- CLAUDE.md files to study
- Feature de referencia (ESTUDIA ESTO)

## 3. Qué Existe (Verificado con MCP)
- Funciones RLS disponibles
- Imports verificados
- Componentes UI disponibles

## 4. Estructura de Archivos a Crear
[File structure, NOT code]

## 5. Patrones a Seguir
[Guidelines, NOT code to copy]

## 6. Criterios de Éxito
- pnpm typecheck pasa
- pnpm lint pasa
- Feature funciona

## 7. Prompt para Ralph
[Ready-to-execute with correct paths]
```

**IMPORTANT**: The blueprint is a CONTEXT MAP, not copy-paste code.
Opus reads it, studies the references, and implements autonomously.

---

## Phase 5: Summary & Next Steps

**Actions**:

1. **Present generation summary**:
   ```
   Context Maps generados:

   1. [blueprints_path]/[version]/00-INVENTARIO.md
   2. [blueprints_path]/[version]/01-[feature]/BLUEPRINT.md
   ...
   ```

2. **Provide Ralph commands** with correct paths:
   ```bash
   /ralph-loop "Implementa la feature '[Feature]' para MakerKit.

   LEE PRIMERO:
   1. El blueprint en [path]/BLUEPRINT.md - tu mapa de contexto
   2. Los archivos CLAUDE.md que indica el blueprint
   3. La feature de referencia que indica el blueprint

   TU PROCESO (libertad total):
   1. Estudia el código existente
   2. Planifica tu approach
   3. Implementa por capas: DB → Server → UI
   4. Verifica con pnpm typecheck después de cada parte
   5. Itera hasta que funcione

   El blueprint es CONTEXTO, no código para copiar.
   Cuando los criterios de éxito se cumplan:
   <promise>FEATURE_COMPLETE</promise>
   " --max-iterations 30 --completion-promise "FEATURE_COMPLETE"
   ```

3. **Offer next action**:
   ```yaml
   question: "¿Que hacemos ahora?"
   header: "Siguiente"
   options:
     - label: "Ejecutar Ralph para primera feature"
       description: "Iniciar implementacion autonoma"
     - label: "Ver blueprint generado"
       description: "Revisar antes de implementar"
     - label: "Terminar por ahora"
       description: "Implementare despues manualmente"
   ```

---

## Configuration File Format

Saved at `.claude/makerkit-planner.local.md`:

```yaml
---
specs_path: "docs/producto"
blueprints_path: "docs/build"
use_versions: true
current_version: "v1.0"
created: "2026-01-03"
updated: "2026-01-03"
---

# MakerKit Planner - Project Configuration

## Paths
- **Specs**: `docs/producto/` (output de /product-spec:ideacion)
- **Blueprints**: `docs/build/` (output de /makerkit-blueprint)

## Version
- Current: v1.0

## Notes
[Project-specific notes]
```

---

## Important Notes

- **Never assume paths** - Always detect or ask
- **Save configuration** - So next run remembers
- **Adapt to project** - Works with any folder structure
- **AskUserQuestion for ambiguities** - Every unclear item = question
- **MCP before assumptions** - Never guess database structure
