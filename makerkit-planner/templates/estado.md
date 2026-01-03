---
feature: "{FEATURE_NAME}"
version: "{VERSION}"
status: pending
account_type: "{ACCOUNT_TYPE}"
rls_pattern: "{RLS_PATTERN}"
specs_source: "../BLUEPRINT.md"
table_name: "{TABLE_NAME}"
created: "{DATE}"
updated: "{DATE}"
iteration: 0
---

# Estado: {FEATURE_DISPLAY_NAME}

## Quick Reference (para Ralph)

| Campo | Valor |
|-------|-------|
| Feature | {FEATURE_NAME} |
| Account Type | {ACCOUNT_TYPE} |
| RLS Pattern | {RLS_PATTERN} |
| Main Table | `public.{TABLE_NAME}` |
| Blueprint | [BLUEPRINT.md](./BLUEPRINT.md) |

---

## Status

| Campo | Valor |
|-------|-------|
| Estado actual | `pending` |
| Iteraciones Ralph | 0 |
| Bloqueadores | Ninguno |

---

## Checklist de Implementacion

### Database Layer

- [ ] Schema SQL creado
  - File: `apps/web/supabase/schemas/XX-{FEATURE_NAME}.sql`
  - Verify: `test -f apps/web/supabase/schemas/XX-{FEATURE_NAME}.sql && echo "OK"`

- [ ] Migracion creada
  - Command: `pnpm --filter web supabase migrations new {FEATURE_NAME}`
  - Verify: `ls apps/web/supabase/migrations/*{FEATURE_NAME}*.sql`

- [ ] Migracion aplicada
  - Command: `pnpm supabase:web:reset`
  - Verify: Exit 0

- [ ] Types generados
  - Command: `pnpm supabase:web:typegen`
  - Verify: `grep -q "{TABLE_NAME}" packages/supabase/src/database.types.ts && echo "OK"`

### Server Layer

- [ ] Zod schemas creados
  - File: `_lib/schemas/{FEATURE_NAME}.schema.ts`
  - Verify: `pnpm typecheck`

- [ ] Server actions implementados
  - File: `_lib/server/{FEATURE_NAME}-server-actions.ts`
  - Verify: `pnpm typecheck`

- [ ] Loader implementado
  - File: `_lib/server/{FEATURE_NAME}-page.loader.ts`
  - Verify: `pnpm typecheck`

### UI Layer

- [ ] Page component creado
  - File: `apps/web/app/home/[account]/{FEATURE_NAME}/page.tsx`
  - Verify: `pnpm typecheck`

- [ ] List component implementado
  - File: `_components/{FEATURE_NAME}-list.tsx`
  - Verify: `pnpm typecheck`

- [ ] Form component implementado
  - File: `_components/{FEATURE_NAME}-form.tsx`
  - Verify: `pnpm typecheck`

- [ ] Detail page implementada (si aplica)
  - File: `[id]/page.tsx`
  - Verify: `pnpm typecheck`

### Final Verification

- [ ] TypeScript sin errores
  - Command: `pnpm typecheck`
  - Verify: Exit 0

- [ ] Lint sin errores
  - Command: `pnpm lint`
  - Verify: Exit 0

- [ ] Formato aplicado
  - Command: `pnpm format:fix`
  - Verify: Exit 0

---

## Verify Commands (Resumen)

```bash
# Database Layer
test -f apps/web/supabase/schemas/XX-{FEATURE_NAME}.sql && echo "Schema OK"
grep -q "{TABLE_NAME}" packages/supabase/src/database.types.ts && echo "Types OK"

# Server + UI Layer
pnpm typecheck

# Final
pnpm typecheck && pnpm lint && echo "ALL CHECKS PASS"
```

---

## Bloqueadores

| Descripcion | Estado | Resolucion |
|-------------|--------|-----------|
| - | - | - |

---

## Notas de Implementacion

> Esta seccion se actualiza durante el loop de Ralph

### Iteracion 1

[Ralph documenta aqui su progreso]

### Problemas Encontrados

[Ralph documenta bloqueadores o decisiones tomadas]

### Decisiones Tomadas

| Decision | Razon |
|----------|-------|
| - | - |

---

## Completion Criteria

Para marcar `FEATURE_COMPLETE`:

1. **Todos los checkboxes marcados** en el checklist arriba
2. **Todos los verify commands pasan** sin errores
3. **No hay errores TypeScript** (`pnpm typecheck` exit 0)
4. **No hay errores de lint** (`pnpm lint` exit 0)

Cuando todo lo anterior se cumple, Ralph debe output EXACTAMENTE:

```
<promise>FEATURE_COMPLETE</promise>
```

---

## Historial

| Fecha | Iteracion | Accion |
|-------|-----------|--------|
| {DATE} | 0 | Estado creado |
