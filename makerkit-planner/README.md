# MakerKit Dev Plugin

Blueprint generator for MakerKit projects with MCP integration.

## Overview

This plugin provides a structured workflow for designing features in MakerKit projects. It generates architecture blueprints ready for autonomous execution with Ralph Wiggum.

## Command: `/makerkit-blueprint`

Launches a 4-phase blueprint generation workflow:

1. **Discovery** - Understand what needs to be built
2. **Exploration** - Analyze codebase with MCP tools (parallel agents)
3. **Questions** - Clarify all ambiguities before designing
4. **Architecture** - Generate Ralph-ready blueprint

### Usage

```bash
/makerkit-blueprint Add student management feature
```

### Output

Generates `docs/arquitectura/XX-feature.md` with:
- Complete SQL schema (tables, RLS, triggers)
- Zod validation schemas
- Server actions
- Page loaders
- Component props
- Implementation checklist with verify commands

## Agents

### makerkit-explorer

Analyzes MakerKit codebase using MCP tools:
- `get_database_summary()` - Existing tables/enums
- `find_complete_features()` - Reference features
- `analyze_feature_pattern()` - Implementation patterns
- `get_app_routes()` - Route structure

### makerkit-architect

Designs feature architecture following MakerKit patterns:
- Personal vs Team account decisions
- RLS policy generation
- Component structure
- Ralph-ready output format

## Skills

### makerkit-patterns

Reference documentation for MakerKit patterns:
- CLAUDE.md file index
- MCP tools reference
- RLS patterns

### makerkit-docs

Acceso dinamico a la documentacion oficial de MakerKit (150 paginas indexadas):
- Patrones conceptuales y guias
- Configuracion de billing, auth, deploy
- Troubleshooting

Invocable desde cualquier contexto: "Usa el skill makerkit-docs para [consulta]"

## Integration with Ralph Wiggum

After generating a blueprint:

```bash
/ralph-loop "Implement blueprint from docs/arquitectura/XX-feature.md" --max-iterations 15
```

## Requirements

- MakerKit MCP server active
- Project with MakerKit structure

## Version

1.0.0
