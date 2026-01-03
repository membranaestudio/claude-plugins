# MakerKit Quick Reference (Consolidado)

> Referencia única para blueprint generation y Ralph execution.
> Contiene todo lo esencial verificado de MakerKit.
> **Auto-contenido**: No requiere archivos externos.

---

## 1. Orden de Comandos (CRÍTICO)

```bash
# Workflow completo de DB changes
1. pnpm supabase:web:start              # Iniciar Supabase
2. [Crear schema SQL]                   # apps/web/supabase/schemas/XX-feature.sql
3. pnpm --filter web supabase:db:diff -f nombre   # Generar migración
4. [REVISAR MANUALMENTE la migración]   # El diff puede fallar con enums
5. pnpm supabase:web:reset              # Aplicar cambios
6. pnpm supabase:web:typegen            # Generar tipos (OBLIGATORIO)
7. pnpm typecheck                       # Verificar tipos
8. pnpm lint:fix                        # Auto-fix lint
9. pnpm format:fix                      # Formatear código
```

---

## 2. Funciones RLS (NO inventar nuevas)

| Función | Uso Correcto | Security |
|---------|--------------|----------|
| `has_role_on_account(account_id)` | Verificar membresía en cuenta | DEFINER |
| `has_role_on_account(account_id, 'role')` | Verificar rol específico | DEFINER |
| `has_permission(auth.uid(), account_id, 'perm'::app_permissions)` | Verificar permiso | INVOKER |
| `is_account_owner(account_id)` | Verificar propietario principal | INVOKER |
| `is_team_member(account_id, user_id)` | Verificar si OTRO usuario es miembro | DEFINER |

**IMPORTANTE:** Para RLS de team resources, usar `has_role_on_account(account_id)`, NO `is_team_member()`.

---

## 3. Orden de RLS en SQL

```sql
-- SIEMPRE EN ESTE ORDEN:

-- 1. ENABLE RLS (primero)
ALTER TABLE public.mi_tabla ENABLE ROW LEVEL SECURITY;

-- 2. REVOKE (quitar defaults)
REVOKE ALL ON public.mi_tabla FROM authenticated, service_role;

-- 3. GRANT (otorgar específicos)
GRANT SELECT, INSERT, UPDATE, DELETE ON public.mi_tabla TO authenticated;

-- 4. CREATE POLICY (por operación)
CREATE POLICY mi_tabla_select ON public.mi_tabla
  FOR SELECT TO authenticated
  USING (public.has_role_on_account(account_id));
```

---

## 4. Next.js 16 Params

```typescript
// CORRECTO - async function usa await
async function Page({ params }: Props) {
  const { account } = await params;  // ✅
}

// INCORRECTO - NO usar use() en async
async function Page({ params }: Props) {
  const { account } = use(params);  // ❌ NUNCA
}
```

---

## 5. Schema SQL Template

```sql
-- XX-feature.sql

-- Permisos (MANUAL - diff no los soporta)
ALTER TYPE public.app_permissions ADD VALUE 'feature.manage';
COMMIT;

-- Tabla
CREATE TABLE IF NOT EXISTS public.features (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES public.accounts(id) ON DELETE CASCADE,
  name text NOT NULL CHECK (char_length(name) >= 2 AND char_length(name) <= 100),
  status public.feature_status DEFAULT 'active',
  notes text,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now(),
  created_by uuid REFERENCES auth.users(id),
  updated_by uuid REFERENCES auth.users(id)
);

-- Índices
CREATE INDEX idx_features_account ON public.features(account_id);
CREATE INDEX idx_features_created ON public.features(created_at DESC);

-- Triggers
CREATE TRIGGER set_features_timestamps
  BEFORE UPDATE ON public.features
  FOR EACH ROW EXECUTE FUNCTION trigger_set_timestamps();

CREATE TRIGGER set_features_user_tracking
  BEFORE INSERT OR UPDATE ON public.features
  FOR EACH ROW EXECUTE FUNCTION trigger_set_user_tracking();

-- RLS (EN ESTE ORDEN)
ALTER TABLE public.features ENABLE ROW LEVEL SECURITY;
REVOKE ALL ON public.features FROM authenticated, service_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON public.features TO authenticated;

CREATE POLICY features_select ON public.features
  FOR SELECT TO authenticated
  USING (public.has_role_on_account(account_id));

CREATE POLICY features_insert ON public.features
  FOR INSERT TO authenticated
  WITH CHECK (public.has_permission(auth.uid(), account_id, 'feature.manage'::app_permissions));

CREATE POLICY features_update ON public.features
  FOR UPDATE TO authenticated
  USING (public.has_permission(auth.uid(), account_id, 'feature.manage'::app_permissions));

CREATE POLICY features_delete ON public.features
  FOR DELETE TO authenticated
  USING (public.has_permission(auth.uid(), account_id, 'feature.manage'::app_permissions));
```

---

## 6. Zod Schema Template

```typescript
// _lib/schemas/feature.schema.ts
import { z } from 'zod';

export const CreateFeatureSchema = z.object({
  name: z.string().min(2, 'Mínimo 2 caracteres').max(100),
  status: z.enum(['active', 'paused', 'archived']).default('active'),
  notes: z.string().max(500).optional(),
});

export const UpdateFeatureSchema = CreateFeatureSchema.partial().extend({
  id: z.string().uuid(),
});

export const DeleteFeatureSchema = z.object({
  id: z.string().uuid(),
});

export type CreateFeatureInput = z.infer<typeof CreateFeatureSchema>;
export type UpdateFeatureInput = z.infer<typeof UpdateFeatureSchema>;
```

---

## 7. Server Action Template

```typescript
// _lib/server/feature-server-actions.ts
'use server';

import { enhanceAction } from '@kit/next/actions';
import { getSupabaseServerClient } from '@kit/supabase/server-client';
import { z } from 'zod';

import {
  CreateFeatureSchema,
  UpdateFeatureSchema,
  DeleteFeatureSchema,
} from '../schemas/feature.schema';

export const createFeatureAction = enhanceAction(
  async (data, { user }) => {
    const client = getSupabaseServerClient();

    const { data: result, error } = await client
      .from('features')
      .insert({
        name: data.name,
        status: data.status,
        notes: data.notes,
        account_id: data.accountId,
      })
      .select()
      .single();

    if (error) throw error;
    return { success: true, data: result };
  },
  { schema: CreateFeatureSchema.extend({ accountId: z.string().uuid() }) }
);

export const updateFeatureAction = enhanceAction(
  async ({ id, ...data }, { user }) => {
    const client = getSupabaseServerClient();

    const { data: result, error } = await client
      .from('features')
      .update(data)
      .eq('id', id)
      .select()
      .single();

    if (error) throw error;
    return { success: true, data: result };
  },
  { schema: UpdateFeatureSchema }
);

export const deleteFeatureAction = enhanceAction(
  async ({ id }, { user }) => {
    const client = getSupabaseServerClient();

    const { error } = await client
      .from('features')
      .delete()
      .eq('id', id);

    if (error) throw error;
    return { success: true };
  },
  { schema: DeleteFeatureSchema }
);
```

---

## 8. Page Loader Template

```typescript
// _lib/server/features-page.loader.ts
import 'server-only';

import { SupabaseClient } from '@supabase/supabase-js';
import { Database } from '@kit/supabase/database';

export async function loadFeaturesPageData(
  client: SupabaseClient<Database>,
  accountId: string,
) {
  const { data, error } = await client
    .from('features')
    .select('*')
    .eq('account_id', accountId)
    .order('created_at', { ascending: false });

  if (error) throw error;
  return data ?? [];
}

export async function loadFeatureById(
  client: SupabaseClient<Database>,
  id: string,
) {
  const { data, error } = await client
    .from('features')
    .select('*')
    .eq('id', id)
    .single();

  if (error) throw error;
  return data;
}
```

---

## 9. Page Component Template

```typescript
// page.tsx
import { getSupabaseServerClient } from '@kit/supabase/server-client';
import { loadFeaturesPageData } from './_lib/server/features-page.loader';

interface Props {
  params: Promise<{ account: string }>;
}

export default async function FeaturesPage({ params }: Props) {
  const { account } = await params;  // AWAIT - Next.js 16
  const client = getSupabaseServerClient();
  const features = await loadFeaturesPageData(client, account);

  return <FeaturesList items={features} accountSlug={account} />;
}
```

---

## 10. Imports Esenciales

```typescript
// Server Actions
import { enhanceAction } from '@kit/next/actions';

// Route Handlers
import { enhanceRouteHandler } from '@kit/next/routes';

// Supabase
import { getSupabaseServerClient } from '@kit/supabase/server-client';
import { useSupabase } from '@kit/supabase/hooks/use-supabase';

// Types
import { Database } from '@kit/supabase/database';

// Validation
import { z } from 'zod';

// Account Hooks
import { useTeamAccountWorkspace } from '@kit/team-accounts/hooks/use-team-account-workspace';

// UI
import { Button } from '@kit/ui/button';
import { Card, CardContent, CardHeader } from '@kit/ui/card';
import { Form, FormField, FormItem, FormLabel, FormControl, FormMessage } from '@kit/ui/form';
import { Input } from '@kit/ui/input';
import { toast } from '@kit/ui/sonner';
```

---

## 11. Estructura de Archivos

```
apps/web/app/home/[account]/features/
├── page.tsx                         # List page (Server Component)
├── [id]/
│   └── page.tsx                     # Detail page
├── _components/
│   ├── features-list.tsx            # List component
│   ├── feature-card.tsx             # Card component
│   └── feature-form.tsx             # Form component (Client)
└── _lib/
    ├── server/
    │   ├── features-page.loader.ts  # Data loaders
    │   └── feature-server-actions.ts # Mutations
    └── schemas/
        └── feature.schema.ts        # Zod schemas

apps/web/supabase/
├── schemas/
│   └── XX-features.sql              # Schema declarativo
├── migrations/
│   └── YYYYMMDD_features.sql        # Migración generada
└── seed.sql                         # Datos de prueba (UUIDs reales)
```

---

## 12. Seed Data Pattern

```sql
-- EN seed.sql (NO en migrations)
-- UUIDs REALES de test data existente:
-- Account: '5deaa894-2094-4da3-b4fd-1fada0809d1c'
-- User: '31a03e74-1639-45b6-bfa7-77447f1a4762'

INSERT INTO public.features (account_id, name, status)
VALUES
  ('5deaa894-2094-4da3-b4fd-1fada0809d1c', 'Test Feature 1', 'active'),
  ('5deaa894-2094-4da3-b4fd-1fada0809d1c', 'Test Feature 2', 'paused');
```

---

## 13. Checklist de Verificación

### Database Layer
- [ ] Schema en `apps/web/supabase/schemas/XX-feature.sql`
- [ ] Migración generada con `db:diff`
- [ ] Migración REVISADA manualmente
- [ ] RLS en orden: ENABLE → REVOKE → GRANT → POLICY
- [ ] Políticas para SELECT, INSERT, UPDATE, DELETE
- [ ] Triggers: timestamps y user_tracking
- [ ] Índices para account_id y created_at
- [ ] `pnpm supabase:web:reset` ejecutado
- [ ] `pnpm supabase:web:typegen` ejecutado

### Server Layer
- [ ] Zod schemas en `_lib/schemas/`
- [ ] Loader en `_lib/server/`
- [ ] Server actions con `enhanceAction`
- [ ] `'use server'` en archivo de actions
- [ ] `import 'server-only'` en loaders

### UI Layer
- [ ] Page component con `await params`
- [ ] Components en `_components/`
- [ ] Forms con `react-hook-form` + `@kit/ui/form`
- [ ] `'use client'` en components interactivos

### Verification
- [ ] `pnpm typecheck` ✓
- [ ] `pnpm lint:fix` ✓
- [ ] `pnpm format:fix` ✓
- [ ] `grep -q "features" packages/supabase/src/database.types.ts` ✓

---

## 14. Errores Comunes

| Error | Corrección |
|-------|------------|
| `is_team_member()` en RLS | Usar `has_role_on_account(account_id)` |
| UUID placeholder en seeds | Usar UUIDs reales de seed.sql existente |
| `use(params)` en async | Usar `await params` |
| Olvidar typegen | Siempre ejecutar después de reset |
| diff para enums | Agregar permisos manualmente |
| GRANT sin REVOKE | Siempre REVOKE ALL primero |

---

## 15. Ralph Completion Criteria

```bash
# Todos estos comandos DEBEN pasar:

# Archivos existen
test -f apps/web/supabase/schemas/XX-feature.sql
test -f apps/web/app/home/[account]/features/page.tsx
test -f apps/web/app/home/[account]/features/_lib/schemas/feature.schema.ts
test -f apps/web/app/home/[account]/features/_lib/server/features-page.loader.ts
test -f apps/web/app/home/[account]/features/_lib/server/feature-server-actions.ts

# Tipos generados
grep -q "features" packages/supabase/src/database.types.ts

# No errores
pnpm typecheck && pnpm lint

# Success
echo "<promise>FEATURE_COMPLETE</promise>"
```

---

## 16. Flujo de Implementacion (Ralph)

### Orden de Fases (CRITICO - No saltar)

```
FASE 1: DATABASE (sin tipos no hay nada)
├── 1. Crear schema SQL
├── 2. Generar migracion (db:diff)
├── 3. REVISAR migracion manualmente
├── 4. Aplicar (reset)
├── 5. typegen ← SIN ESTO TODO FALLA
└── 6. Verificar: grep "tabla" database.types.ts

FASE 2: SERVER (depende de tipos)
├── 7. Crear Zod schemas
├── 8. Crear page loader
├── 9. Crear server actions
└── 10. Verificar: pnpm typecheck

FASE 3: UI (depende de server)
├── 11. Crear directorios
├── 12. Crear page.tsx
├── 13. Crear components
└── 14. Verificar: pnpm typecheck

FASE 4: FINAL
├── 15. pnpm lint:fix
├── 16. pnpm format:fix
└── 17. <promise>FEATURE_COMPLETE</promise>
```

### Parallel Groups

| Grupo | Operaciones | Tipo |
|-------|-------------|------|
| A | Exploracion MCP (get_database_summary, find_complete_features) | Paralelo |
| B | Verificacion archivos (test -f ...) | Paralelo |
| C | Quality checks (typecheck → lint → format) | Secuencial |

### Checkpoints

| CP | Cuando | Verifica | Si Falla |
|----|--------|----------|----------|
| CP1 | Pre-Blueprint | MCP responde | STOP |
| CP2 | Post-Migration | typegen exitoso | Revisar SQL |
| CP3 | Post-Archivo | typecheck exit 0 | Corregir |
| CP4 | Pre-Completion | Todos criterios | Documentar |

### Quality Gates (Hookify)

Reglas automaticas activas:
- `post-write-typecheck`: Recordar typecheck despues de .ts/.tsx
- `migration-reminder`: Recordar db:diff/reset/typegen despues de schemas/
- `block-env-direct`: Bloquear edicion de .env

---

## 17. Dependencias Entre Capas

```
DATABASE LAYER
  Schema → Migration → Reset → Typegen → Types
  │
  ▼
SERVER LAYER
  Zod Schemas → Loaders → Actions
  │
  ▼
UI LAYER
  Routes → Pages → Components → Forms
```

**Regla de oro:** No crear codigo de capa N+1 sin completar capa N.

---

*Quick reference auto-contenido para makerkit-planner plugin*
