# Changelog - Plugin /makerkit-blueprint

Registro de mejoras basadas en testing real y consolidación de patrones MakerKit.

---

## [1.3.0] - 2026-01-03

### Agregado

**10 Advanced Patterns en makerkit-consolidado.md (Secciones 18-27):**

| Sección | Pattern | Uso |
|---------|---------|-----|
| 18 | Junction Tables | Relaciones N:M con RLS heredado |
| 19 | Singleton per Account | Config única por cuenta |
| 20 | Custom RPC Functions | Lógica DB compleja con security invoker |
| 21 | State Machine | Estados con transiciones validadas |
| 22 | Partial Unique Indexes | Constraints condicionales |
| 23 | Array Fields | Arrays PostgreSQL + Zod |
| 24 | Direct User Access | RLS dual (team OR user) |
| 25 | Soft Delete | deleted_at con políticas automáticas |
| 26 | Pattern Selection Guide | Cuándo usar cada pattern |
| 27 | Advanced Patterns Checklist | Pre-blueprint verification |

**Templates SQL completos para cada pattern:**
- CREATE TABLE con constraints
- RLS policies correctas
- Triggers de validación
- Zod schemas correspondientes

### Razón del cambio

Análisis de specs vs arquitectura MakerKit reveló gaps en patterns avanzados.
Objetivo: Blueprints más completos para que Ralph/Opus tenga todo lo necesario.

---

## [1.2.0] - 2026-01-03

### Agregado

**Secciones nuevas en makerkit-consolidado.md:**
- Sección 16: Flujo de Implementación (Ralph) - Orden de fases, parallel groups, checkpoints
- Sección 17: Dependencias Entre Capas - Diagrama DATABASE → SERVER → UI

**Quality Gates (hookify rules):**
- `post-write-typecheck`: Recordatorio de typecheck después de .ts/.tsx
- `migration-reminder`: Workflow completo después de schemas SQL
- `block-env-direct`: Bloqueo de edición directa de .env

**Documentación de flujo:**
- `FLUJO-DESARROLLO.md` - Orquestador de 5 fases
- `session-state.template.yaml` - Template para tracking de features

### Corregido

**Funciones RLS:**
- `is_team_member()` → `has_role_on_account(account_id)` en makerkit-explorer.md
- Documentación clara en sección 2 de consolidado

### Razón del cambio

Investigación profunda de 6 agentes paralelos reveló:
- Ralph es genérico, el blueprint controla todo
- Sistema de 3 niveles: Patterns (plugin) + Flows (proyecto) + Quality Gates (hookify)
- Dependencias críticas entre capas que deben respetarse

---

## [1.1.0] - 2026-01-02

### Agregado

- Agentes: makerkit-explorer, makerkit-architect
- Skill: makerkit-patterns con referencia consolidada
- Templates: estado.md, inventario.md
- Integración con MCP tools de MakerKit

### Versión inicial funcional

- Comando `/makerkit-blueprint` para generar arquitecturas
- Consume specs de `/ideacion`
- Genera blueprints compatibles con `/ralph-loop`

---

## [1.0.0] - 2026-01-01

### Versión inicial

- Estructura básica del plugin
- Comando placeholder

---

## Roadmap de mejoras futuras

- [ ] Auto-detección de features existentes en codebase
- [ ] Validación de blueprints contra MCP antes de Ralph
- [ ] Soporte para features con dependencias entre sí
- [ ] Generación de tests E2E en blueprint
