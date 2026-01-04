# MakerKit Planner Plugin

**Context Map generator** for MakerKit projects. Not copy-paste code, but guides for autonomous development.

## Philosophy (v1.6.0)

**OLD**: Blueprint = Code to copy-paste → Hallucinations, rigid execution
**NEW**: Blueprint = Context Map → Opus works autonomously with freedom

The plugin generates:
1. **Qué hacer** - Feature specs
2. **Qué leer** - Specific files to study
3. **Qué existe** - Verified with MCP (no hallucinations)
4. **Patrones de referencia** - Features to learn from
5. **Criterios de éxito** - Clear completion criteria

## Command: `/makerkit-blueprint`

### First Run (New Project)

```bash
/makerkit-blueprint
```

The plugin will:
1. Search for existing specs files (from `/product-spec:ideacion`)
2. Ask where to save blueprints
3. Ask about versioning preferences
4. Save configuration to `.claude/makerkit-planner.local.md`

### Subsequent Runs

```bash
/makerkit-blueprint
```

Uses saved configuration. Detects new specs and generates Context Maps.

### Workflow Phases

```
Phase 0: Project Context Detection
├── Check for saved config
├── Detect existing specs and blueprints
├── Ask user for missing configuration
└── Save configuration

Phase 1: Specs Loading & Feature Selection
├── Read specs from configured path
├── Parse features
├── Ask which features to generate
└── Confirm order

Phase 2: MCP Analysis
├── Validate MCP connection
├── Analyze database structure (get_database_summary)
├── Find reference features (find_complete_features)
└── Verify what exists (no assumptions)

Phase 3: Clarifying Questions (Per Feature)
├── Account type? (Team/Personal)
├── Estados?
├── Borrado? (Soft/Hard delete)
├── Permisos?
└── UI Layout?

Phase 4: Context Map Generation
├── Generate inventory (if needed)
└── Generate BLUEPRINT.md per feature (Context Map, NOT code)

Phase 5: Summary & Next Steps
├── Show generated files
├── Provide Ralph commands (with autonomy)
└── Offer to start implementation
```

## Output: Context Map (NOT code)

```markdown
# [Feature] Context Map

## 1. Tu Misión
[What to implement]

## 2. Qué Leer Antes de Codificar
- CLAUDE.md files to study
- Feature de referencia (ESTUDIA ESTO)

## 3. Qué Existe (Verificado con MCP)
- Funciones RLS disponibles
- Imports verificados
- Componentes UI

## 4. Estructura de Archivos a Crear
[Structure, NOT code]

## 5. Patrones a Seguir
[Guidelines, NOT code to copy]

## 6. Criterios de Éxito
- pnpm typecheck pasa
- pnpm lint pasa
- Feature funciona

## 7. Prompt para Ralph
[With full autonomy]
```

## Integration with Ralph

After generating Context Maps, the plugin provides the Ralph command:

```bash
/ralph-loop "Implementa la feature '[Feature]' para MakerKit.

LEE PRIMERO:
1. El blueprint en [path]/BLUEPRINT.md - tu mapa de contexto
2. Los archivos CLAUDE.md que indica
3. La feature de referencia

TU PROCESO (libertad total):
1. Estudia el código existente
2. Planifica tu approach
3. Implementa por capas: DB → Server → UI
4. Verifica con pnpm typecheck
5. Itera hasta que funcione

El blueprint es CONTEXTO, no código para copiar.
<promise>FEATURE_COMPLETE</promise>
" --max-iterations 30 --completion-promise "FEATURE_COMPLETE"
```

## Agents

### makerkit-explorer (Yellow)
Analyzes MakerKit codebase using MCP tools.

### makerkit-architect (Green)
Designs feature architecture. Generates Context Maps (NOT copy-paste code).

## Skills

### makerkit-patterns
Quick reference for MakerKit patterns.

## Requirements

- MakerKit MCP server active
- Project with MakerKit structure
- Specs from `/product-spec:ideacion` or manual input

## Version History

- **1.6.0** - Context Maps philosophy (BREAKING CHANGE)
- **1.5.0** - Pre-generation verification
- **1.4.0** - UI pattern sections
- **1.3.0** - Advanced patterns
- **1.2.0** - Ralph flow integration
