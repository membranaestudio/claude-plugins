# Changelog - Plugin /makerkit-blueprint

Registro de mejoras basadas en testing real y consolidación de patrones MakerKit.

---

## [1.6.0] - 2026-01-04

### Cambio de Filosofía (BREAKING CHANGE)

**ANTES:** Blueprint = Código copy-paste ready
**AHORA:** Blueprint = Mapa de Contexto para trabajo autónomo

El blueprint ya NO genera código completo para copiar. En su lugar, genera:
1. **Qué hacer** - Specs de la feature
2. **Qué leer** - Archivos específicos a estudiar
3. **Qué existe** - Funciones/componentes verificados con MCP
4. **Patrones de referencia** - Features existentes para aprender
5. **Criterios de éxito** - Cuándo está completo

### Cambios en makerkit-architect.md

**Eliminado:**
- Código SQL completo para copiar
- Código TypeScript completo para copiar
- Checklist rígido paso a paso
- Verify commands por cada paso
- estado.md (Opus trackea su propio progreso)

**Agregado:**
- Sección "Qué Leer Antes de Codificar"
- Sección "Qué Existe (Verificado con MCP)"
- Feature de referencia para estudiar
- Nuevo prompt para Ralph con libertad total
- Ejemplos de Good vs Bad output

### Nuevo Prompt para Ralph

```
/ralph-loop "Implementa la feature 'X' para MakerKit.

LEE PRIMERO:
1. El blueprint - tu mapa de contexto
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
" --max-iterations 30
```

### Razón del cambio

Conversación profunda sobre el flujo de desarrollo reveló:
- El blueprint intentaba ser "copy-paste ready" pero tenía alucinaciones
- Opus es capaz de planificar y desarrollar autónomamente
- El contexto es más valioso que el código
- La libertad produce mejores resultados que la rigidez

**Filosofía nueva:** Dame contexto, no código. Dame referencias, no instrucciones. Dame libertad, no rigidez.

---

## [1.5.0] - 2026-01-04

### Agregado

**Pre-Generation Verification Workflow en makerkit-architect.md:**

Nuevo workflow obligatorio ANTES de generar código que previene alucinaciones:

| Paso | Verificación | Herramienta |
|------|--------------|-------------|
| 1 | Verificar exports de packages | Grep en package.json |
| 2 | Confirmar funciones RLS | search_database_functions() |
| 3 | Validar componentes UI | components_search() |
| 4 | Verificar imports locales vs packages | Glob + Read |

**Workspace Pattern Correcto (Sección 31 reescrita):**

```typescript
// CORRECTO - Import LOCAL, no de package
import { loadTeamWorkspace } from '~/home/[account]/_lib/server/team-account-workspace.loader';

const { account, user } = await loadTeamWorkspace(accountSlug);
const canManage = account.permissions?.includes('settings.manage') ?? false;
```

**Cambios en makerkit-consolidado.md:**
- Sección 14: Renombrada de "Errores Comunes" → "Verificación Pre-Generación"
- Sección 31: Completamente reescrita con patrones correctos de workspace/permisos

### Razón del cambio

Validación de blueprints v1.4.0 reveló alucinaciones:
- `getTeamAccountWorkspace` de `@kit/team-accounts/workspace` (NO EXISTE)
- `loadTeamWorkspace` de `@kit/team-accounts/server` (NO EXISTE - es archivo LOCAL)
- `workspace.permissions.includes()` (INCORRECTO - es `account.permissions`)

**Enfoque nuevo:** Verificar ANTES de generar, no documentar errores después.

### Roadmap actualizado
- [x] Validación de blueprints contra MCP antes de Ralph (IMPLEMENTADO)

---

## [1.4.0] - 2026-01-04

### Agregado

**9 UI Pattern Sections en makerkit-consolidado.md (Secciones 28-36):**

| Sección | Pattern | Uso |
|---------|---------|-----|
| 28 | Sidebar Navigation | Agregar items al sidebar de navegación |
| 29 | Breadcrumb Implementation | Breadcrumbs con componentes MakerKit |
| 30 | Account Context Pattern | Resolver account_id centralizado con cache() |
| 31 | Permission-Based UI Rendering | Renderizado condicional por permisos/rol |
| 32 | Table Patterns | Search, filter, empty states, pagination |
| 33 | Loading States | Skeletons con loading.tsx |
| 34 | Danger Zone Pattern | Acciones destructivas con confirmación |
| 35 | Page Layout Patterns | PageHeader consistente |
| 36 | Blueprint Checklist (Updated) | Verificación completa pre-Ralph |

**Patterns específicos agregados:**
- `getAccountBySlug()` - Función centralizada con React cache()
- `useTeamAccountWorkspace()` - Hook para permisos en cliente
- Empty states informativos con CTA
- Badge variant maps por tipo de status
- Loading skeletons para tablas y forms

### Razón del cambio

Análisis de UI real de MakerKit (10-MAKERKIT-UI-EXPLORATION.md) reveló gaps entre
blueprints generados y patterns de UI establecidos. Estos patterns son genéricos
de MakerKit, no específicos de ningún proyecto.

Objetivo: Blueprints que incluyan TODA la UI necesaria (navegación, permisos, estados).

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
