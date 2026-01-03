# MakerKit Planner Plugin

Blueprint generator for MakerKit projects. **Adapts to any project structure.**

## Overview

This plugin provides a structured workflow for designing features in MakerKit projects. It:

1. **Detects or asks** for project structure (specs path, blueprints path)
2. **Saves configuration** for future runs
3. **Consumes specs** from `/ideacion` or other sources
4. **Analyzes codebase** using MCP tools
5. **Resolves ambiguities** with AskUserQuestion
6. **Generates blueprints** ready for Ralph Wiggum execution

## Command: `/makerkit-blueprint`

### First Run (New Project)

```bash
/makerkit-blueprint
```

The plugin will:
1. Search for existing specs files
2. Ask where to save blueprints
3. Ask about versioning preferences
4. Save configuration to `.claude/makerkit-planner.local.md`

### Subsequent Runs

```bash
/makerkit-blueprint
```

Uses saved configuration. Detects new specs and generates blueprints.

### Workflow Phases

```
Phase 0: Project Context Detection
├── Check for saved config (.claude/makerkit-planner.local.md)
├── Detect existing specs and blueprints
├── Ask user for missing configuration
└── Save configuration for next time

Phase 1: Specs Loading & Feature Selection
├── Read specs from configured path
├── Parse features
├── Ask which features to generate
└── Confirm order

Phase 2: MCP Analysis
├── Validate MCP connection
├── Analyze database structure
├── Find reference features
└── Read CLAUDE.md patterns

Phase 3: Clarifying Questions (Per Feature)
├── Account type?
├── Estados?
├── Borrado?
├── Permisos?
└── UI Layout?

Phase 4: Blueprint Generation
├── Generate inventory (if needed)
├── Generate BLUEPRINT.md per feature
└── Generate estado.md per feature

Phase 5: Summary & Next Steps
├── Show generated files
├── Provide Ralph commands
└── Offer to start implementation
```

## Configuration

Saved at `.claude/makerkit-planner.local.md`:

```yaml
---
specs_path: "docs/producto"
blueprints_path: "docs/build"
use_versions: true
current_version: "v1.0"
---
```

Edit this file to change paths. Delete it to reconfigure.

## Adaptive Structure

The plugin works with **any folder structure**:

```
# Example 1: With versioning
docs/producto/v1.0/03-FEATURE-SPECS.md
docs/build/v1.0/01-feature/BLUEPRINT.md

# Example 2: Without versioning
docs/specs/features.md
docs/blueprints/feature/BLUEPRINT.md

# Example 3: Custom paths
my-docs/product/specs.md
my-docs/architecture/feature/BLUEPRINT.md
```

## Agents

### makerkit-explorer (Yellow)

Analyzes MakerKit codebase using MCP tools.

### makerkit-architect (Green)

Designs feature architecture. Generates BLUEPRINT.md and estado.md.

## Skills

### makerkit-patterns

Quick reference for MakerKit patterns.

### makerkit-docs

Dynamic access to official MakerKit documentation.

## Integration with Ralph

After generating blueprints, the plugin provides the exact Ralph command:

```bash
/ralph-loop "Implementa [Feature] siguiendo [path]/BLUEPRINT.md.
Actualiza [path]/estado.md con tu progreso.
<promise>FEATURE_COMPLETE</promise>"
--max-iterations 20
--completion-promise "FEATURE_COMPLETE"
```

## Requirements

- MakerKit MCP server active
- Project with MakerKit structure
- Specs from `/ideacion` or manual input

## Version

1.1.0
- Adaptive to any project structure
- Saves configuration for future runs
- AskUserQuestion for all ambiguities
- estado.md generation for Ralph tracking
