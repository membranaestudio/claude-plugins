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

## 14. Verificación Pre-Generación

**SIEMPRE verificar antes de usar:**

| Qué Verificar | Herramienta | Cómo |
|---------------|-------------|------|
| Package exports | Grep | `pattern="exports" path="packages/.../package.json"` |
| Funciones DB | MCP | `search_database_functions({query: "nombre"})` |
| Componentes UI | MCP | `components_search({query: "nombre"})` |
| Campos de tabla | MCP | `get_table_info({tableName: "nombre"})` |
| RLS válido | MCP | `validate_rls_policies({table: "nombre"})` |

**Regla de oro:** Si no puedes verificar que existe, no lo uses.

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

## 18. Junction Tables (Many-to-Many)

> Cuando una entidad tiene relación N:M con otra.

### SQL Template

```sql
-- Tabla principal A
CREATE TABLE public.projects (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES public.accounts(id) ON DELETE CASCADE,
  name text NOT NULL
);

-- Tabla principal B
CREATE TABLE public.tags (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES public.accounts(id) ON DELETE CASCADE,
  name text NOT NULL
);

-- Junction table
CREATE TABLE public.project_tags (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id uuid NOT NULL REFERENCES public.projects(id) ON DELETE CASCADE,
  tag_id uuid NOT NULL REFERENCES public.tags(id) ON DELETE CASCADE,
  created_at timestamptz DEFAULT now(),
  UNIQUE(project_id, tag_id)  -- Evitar duplicados
);

-- RLS para junction: heredar del parent
ALTER TABLE public.project_tags ENABLE ROW LEVEL SECURITY;

CREATE POLICY junction_select ON public.project_tags
  FOR SELECT TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM public.projects p
      WHERE p.id = project_id
      AND public.has_role_on_account(p.account_id)
    )
  );

CREATE POLICY junction_insert ON public.project_tags
  FOR INSERT TO authenticated
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM public.projects p
      WHERE p.id = project_id
      AND public.has_permission(auth.uid(), p.account_id, 'projects.manage'::app_permissions)
    )
  );

CREATE POLICY junction_delete ON public.project_tags
  FOR DELETE TO authenticated
  USING (
    EXISTS (
      SELECT 1 FROM public.projects p
      WHERE p.id = project_id
      AND public.has_permission(auth.uid(), p.account_id, 'projects.manage'::app_permissions)
    )
  );
```

### Zod Schema

```typescript
export const AddTagToProjectSchema = z.object({
  projectId: z.string().uuid(),
  tagId: z.string().uuid(),
});

export const RemoveTagFromProjectSchema = AddTagToProjectSchema;
```

### Loader Pattern

```typescript
// Cargar project con sus tags
export async function loadProjectWithTags(client: SupabaseClient, projectId: string) {
  const { data, error } = await client
    .from('projects')
    .select(`
      *,
      project_tags (
        tag:tags (id, name)
      )
    `)
    .eq('id', projectId)
    .single();

  if (error) throw error;
  return data;
}
```

---

## 19. Singleton per Account

> Una sola instancia por team account (configuración, preferencias).

### SQL Template

```sql
CREATE TABLE public.account_settings (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES public.accounts(id) ON DELETE CASCADE,
  setting_name text,
  setting_value jsonb DEFAULT '{}',
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now(),
  UNIQUE(account_id)  -- SINGLETON: Solo uno por account
);

-- RLS simplificado para singleton
CREATE POLICY settings_select ON public.account_settings
  FOR SELECT TO authenticated
  USING (public.has_role_on_account(account_id));

CREATE POLICY settings_update ON public.account_settings
  FOR UPDATE TO authenticated
  USING (public.has_permission(auth.uid(), account_id, 'settings.manage'::app_permissions));

-- NO permitir INSERT/DELETE manual (usar upsert o auto-crear)
```

### Server Action (Upsert)

```typescript
export const updateAccountSettingsAction = enhanceAction(
  async (data, { user }) => {
    const client = getSupabaseServerClient();

    const { data: result, error } = await client
      .from('account_settings')
      .upsert({
        account_id: data.accountId,
        setting_name: data.settingName,
        setting_value: data.settingValue,
      }, {
        onConflict: 'account_id'  // Singleton upsert
      })
      .select()
      .single();

    if (error) throw error;
    return { success: true, data: result };
  },
  { schema: UpdateSettingsSchema }
);
```

---

## 20. Custom RPC Functions

> Lógica de negocio compleja que debe ejecutarse en DB.

### SQL Template

```sql
-- SECURITY INVOKER: Hereda RLS del caller (preferido)
CREATE FUNCTION public.get_project_stats(p_account_id uuid)
RETURNS TABLE(
  total_projects integer,
  active_projects integer,
  completed_projects integer
)
LANGUAGE plpgsql
SECURITY INVOKER
SET search_path = ''
AS $$
BEGIN
  RETURN QUERY
  SELECT
    COUNT(*)::integer as total_projects,
    COUNT(*) FILTER (WHERE status = 'active')::integer as active_projects,
    COUNT(*) FILTER (WHERE status = 'completed')::integer as completed_projects
  FROM public.projects
  WHERE account_id = p_account_id;
END;
$$;

-- GRANT a authenticated
GRANT EXECUTE ON FUNCTION public.get_project_stats TO authenticated;
```

### Llamar desde TypeScript

```typescript
// En loader
export async function loadProjectStats(client: SupabaseClient, accountId: string) {
  const { data, error } = await client.rpc('get_project_stats', {
    p_account_id: accountId,
  });

  if (error) throw error;
  return data?.[0] ?? { total_projects: 0, active_projects: 0, completed_projects: 0 };
}
```

### Cuándo Usar SECURITY DEFINER

```sql
-- SECURITY DEFINER: Ejecuta con permisos del creador
-- SOLO usar cuando necesitas bypass RLS controlado
CREATE FUNCTION public.admin_get_all_users()
RETURNS SETOF auth.users
LANGUAGE plpgsql
SECURITY DEFINER  -- ⚠️ CUIDADO: Bypass RLS
SET search_path = ''
AS $$
BEGIN
  -- Validar MANUALMENTE que es super admin
  IF NOT public.is_super_admin() THEN
    RAISE EXCEPTION 'Unauthorized';
  END IF;

  RETURN QUERY SELECT * FROM auth.users;
END;
$$;
```

---

## 21. State Machine Pattern

> Entidades con status que transiciona entre estados válidos.

### SQL Template

```sql
-- Enum de estados
CREATE TYPE public.task_status AS ENUM ('draft', 'pending', 'in_progress', 'completed', 'cancelled');

-- Tabla con status
CREATE TABLE public.tasks (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES public.accounts(id) ON DELETE CASCADE,
  name text NOT NULL,
  status public.task_status DEFAULT 'draft',
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);

-- Trigger para validar transiciones
CREATE FUNCTION public.validate_task_transition()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
  -- Definir transiciones válidas
  IF OLD.status = 'draft' AND NEW.status NOT IN ('pending', 'cancelled') THEN
    RAISE EXCEPTION 'Invalid transition from draft to %', NEW.status;
  ELSIF OLD.status = 'pending' AND NEW.status NOT IN ('in_progress', 'cancelled') THEN
    RAISE EXCEPTION 'Invalid transition from pending to %', NEW.status;
  ELSIF OLD.status = 'in_progress' AND NEW.status NOT IN ('completed', 'cancelled') THEN
    RAISE EXCEPTION 'Invalid transition from in_progress to %', NEW.status;
  ELSIF OLD.status IN ('completed', 'cancelled') THEN
    RAISE EXCEPTION 'Cannot change status from final state %', OLD.status;
  END IF;

  RETURN NEW;
END;
$$;

CREATE TRIGGER check_task_transition
  BEFORE UPDATE OF status ON public.tasks
  FOR EACH ROW
  WHEN (OLD.status IS DISTINCT FROM NEW.status)
  EXECUTE FUNCTION public.validate_task_transition();
```

### Zod Schema con Enum

```typescript
export const TaskStatusSchema = z.enum(['draft', 'pending', 'in_progress', 'completed', 'cancelled']);

export const UpdateTaskStatusSchema = z.object({
  id: z.string().uuid(),
  status: TaskStatusSchema,
});

// Validación de transición en cliente (opcional, DB es la fuente de verdad)
const VALID_TRANSITIONS: Record<string, string[]> = {
  draft: ['pending', 'cancelled'],
  pending: ['in_progress', 'cancelled'],
  in_progress: ['completed', 'cancelled'],
  completed: [],
  cancelled: [],
};
```

---

## 22. Partial Unique Indexes

> Unique constraint que aplica solo bajo ciertas condiciones.

### SQL Template

```sql
-- Ejemplo: Solo un item "active" por usuario
CREATE UNIQUE INDEX unique_active_subscription
ON public.subscriptions (user_id)
WHERE status = 'active';

-- Ejemplo: Unique por periodo excepto cancelled
CREATE UNIQUE INDEX unique_payment_period
ON public.payments (student_id, period_month, period_year)
WHERE status != 'cancelled';

-- Ejemplo: Unique slug solo para publicados
CREATE UNIQUE INDEX unique_published_slug
ON public.posts (account_id, slug)
WHERE published = true;
```

### Cuándo Usar

| Caso | Index |
|------|-------|
| Solo uno activo por usuario | `WHERE status = 'active'` |
| Único excepto soft-deleted | `WHERE deleted_at IS NULL` |
| Único por periodo vigente | `WHERE NOT expired` |

---

## 23. Array Fields

> Campos que almacenan múltiples valores.

### SQL Template

```sql
CREATE TABLE public.notifications_settings (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES public.accounts(id) ON DELETE CASCADE,
  reminder_days integer[] DEFAULT '{3,5,7}'::integer[],  -- Array de enteros
  notification_channels text[] DEFAULT '{email}'::text[],  -- Array de texto
  UNIQUE(account_id)
);

-- Consultar si contiene valor
SELECT * FROM notifications_settings
WHERE 3 = ANY(reminder_days);

-- Consultar si contiene todos
SELECT * FROM notifications_settings
WHERE reminder_days @> ARRAY[3, 5];
```

### Zod Schema

```typescript
export const NotificationSettingsSchema = z.object({
  reminderDays: z.array(z.number().int().positive()).default([3, 5, 7]),
  notificationChannels: z.array(z.enum(['email', 'sms', 'push'])).default(['email']),
});
```

### TypeScript Type

```typescript
// Los arrays se mapean a arrays TypeScript
type NotificationSettings = {
  reminder_days: number[] | null;
  notification_channels: string[] | null;
}
```

---

## 24. Direct User Access (Sin Team Membership)

> Usuario accede a recursos sin ser miembro del team account.
> Útil para: clientes, invitados, usuarios externos.

### SQL Template

```sql
CREATE TABLE public.guest_resources (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES public.accounts(id) ON DELETE CASCADE,
  user_id uuid REFERENCES auth.users(id),  -- OPCIONAL: puede ser NULL
  access_token text UNIQUE,  -- Para acceso sin login
  data jsonb,
  created_at timestamptz DEFAULT now()
);

-- RLS: Team member O usuario directo O token válido
CREATE POLICY guest_select ON public.guest_resources
  FOR SELECT TO authenticated
  USING (
    -- Team members ven todo del account
    public.has_role_on_account(account_id)
    OR
    -- Usuario ve solo sus propios recursos
    user_id = auth.uid()
  );

-- Para acceso anónimo via token (usar en API route, no RLS)
CREATE POLICY guest_anon_select ON public.guest_resources
  FOR SELECT TO anon
  USING (
    access_token IS NOT NULL  -- Solo recursos con token público
  );
```

### Pattern de Magic Link

```typescript
// Generar link de acceso
export async function generateAccessLink(resourceId: string) {
  const token = crypto.randomUUID();

  await client
    .from('guest_resources')
    .update({ access_token: token })
    .eq('id', resourceId);

  return `${baseUrl}/access/${token}`;
}

// Verificar acceso via token
export async function verifyAccess(token: string) {
  const { data } = await client
    .from('guest_resources')
    .select('*')
    .eq('access_token', token)
    .single();

  return data;
}
```

---

## 25. Soft Delete Pattern

> Marcar como eliminado en lugar de DELETE real.

### SQL Template

```sql
CREATE TABLE public.documents (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id uuid NOT NULL REFERENCES public.accounts(id) ON DELETE CASCADE,
  name text NOT NULL,
  deleted_at timestamptz,  -- NULL = activo, timestamp = eliminado
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);

-- Índice para queries de activos
CREATE INDEX idx_documents_active
ON public.documents (account_id)
WHERE deleted_at IS NULL;

-- RLS: Solo ver activos por defecto
CREATE POLICY documents_select ON public.documents
  FOR SELECT TO authenticated
  USING (
    public.has_role_on_account(account_id)
    AND deleted_at IS NULL  -- Solo activos
  );

-- Vista para incluir eliminados (admin)
CREATE VIEW public.documents_all AS
SELECT * FROM public.documents
WHERE public.has_role_on_account(account_id);
```

### Server Action

```typescript
export const softDeleteDocumentAction = enhanceAction(
  async ({ id }) => {
    const client = getSupabaseServerClient();

    const { error } = await client
      .from('documents')
      .update({ deleted_at: new Date().toISOString() })
      .eq('id', id);

    if (error) throw error;
    return { success: true };
  },
  { schema: DeleteDocumentSchema }
);

export const restoreDocumentAction = enhanceAction(
  async ({ id }) => {
    const client = getSupabaseServerClient();

    const { error } = await client
      .from('documents')
      .update({ deleted_at: null })
      .eq('id', id);

    if (error) throw error;
    return { success: true };
  },
  { schema: RestoreDocumentSchema }
);
```

---

## 26. Cuándo Usar Cada Patrón

| Patrón | Usar Cuando | Ejemplo |
|--------|-------------|---------|
| Junction Table | Relación N:M | tags, categories, permissions |
| Singleton | Una config por account | settings, preferences |
| Custom RPC | Lógica compleja en DB | stats, aggregations, validations |
| State Machine | Status con transiciones | orders, tasks, workflows |
| Partial Unique | Unique condicional | "solo uno activo" |
| Array Fields | Lista de valores simples | días, tags inline, opciones |
| Direct User Access | Usuario sin membership | clientes, invitados, público |
| Soft Delete | Preservar historial | documentos, registros auditables |

---

## 27. Checklist de Patrones Avanzados

### Pre-Blueprint
- [ ] ¿La feature tiene relaciones N:M? → Sección 18
- [ ] ¿Es configuración única por account? → Sección 19
- [ ] ¿Necesita lógica DB compleja? → Sección 20
- [ ] ¿Tiene estados con transiciones? → Sección 21
- [ ] ¿Unique solo bajo condiciones? → Sección 22
- [ ] ¿Campos con múltiples valores? → Sección 23
- [ ] ¿Usuarios acceden sin ser members? → Sección 24
- [ ] ¿Necesita soft delete? → Sección 25

### Durante Blueprint
- [ ] Incluir template de patrón identificado
- [ ] Adaptar RLS según patrón
- [ ] Documentar en estado.md qué patrón se usa

---

---

## 28. Sidebar Navigation Configuration

> Cada feature debe agregar su link al sidebar del team account.

### Archivo de Configuración

```typescript
// File: apps/web/config/team-account-navigation.config.tsx

import {
  LayoutDashboard,
  Settings,
  Users,
  CreditCard,
  Calendar,
  // Agregar iconos para features nuevas
} from 'lucide-react';

export const teamAccountNavigationConfig = {
  // Application Section
  routes: [
    {
      label: 'Dashboard',
      path: '/home/[account]',
      Icon: LayoutDashboard,
    },
    // AGREGAR FEATURES AQUÍ:
    // {
    //   label: 'Mi Feature',
    //   path: '/home/[account]/mi-feature',
    //   Icon: IconName,
    // },
  ],

  // Settings Section
  settings: [
    {
      label: 'Settings',
      path: '/home/[account]/settings',
      Icon: Settings,
    },
    {
      label: 'Members',
      path: '/home/[account]/members',
      Icon: Users,
    },
    {
      label: 'Billing',
      path: '/home/[account]/billing',
      Icon: CreditCard,
    },
  ],
};
```

### Checklist para Agregar Feature

```markdown
- [ ] Importar icono de `lucide-react`
- [ ] Agregar entrada en `routes` o `settings` según tipo
- [ ] Verificar que el path coincida con la carpeta de la feature
- [ ] Reiniciar dev server para ver cambios
```

---

## 29. Breadcrumb Implementation

> Navegación jerárquica consistente en todas las páginas.

### Pattern Estándar

```typescript
// apps/web/app/home/[account]/features/page.tsx

import {
  Breadcrumb,
  BreadcrumbItem,
  BreadcrumbLink,
  BreadcrumbList,
  BreadcrumbPage,
  BreadcrumbSeparator,
} from '@kit/ui/breadcrumb';

interface Props {
  params: Promise<{ account: string }>;
}

export default async function FeaturesPage({ params }: Props) {
  const { account } = await params;

  return (
    <div className="space-y-6">
      {/* Breadcrumb */}
      <Breadcrumb>
        <BreadcrumbList>
          <BreadcrumbItem>
            <BreadcrumbLink href={`/home/${account}`}>
              Home
            </BreadcrumbLink>
          </BreadcrumbItem>
          <BreadcrumbSeparator />
          <BreadcrumbItem>
            <BreadcrumbPage>Features</BreadcrumbPage>
          </BreadcrumbItem>
        </BreadcrumbList>
      </Breadcrumb>

      {/* Page Content */}
      <PageHeader title="Features" description="Manage your features" />
      {/* ... */}
    </div>
  );
}
```

### Breadcrumb con Subpágina

```typescript
// apps/web/app/home/[account]/features/[id]/page.tsx

<Breadcrumb>
  <BreadcrumbList>
    <BreadcrumbItem>
      <BreadcrumbLink href={`/home/${account}`}>Home</BreadcrumbLink>
    </BreadcrumbItem>
    <BreadcrumbSeparator />
    <BreadcrumbItem>
      <BreadcrumbLink href={`/home/${account}/features`}>Features</BreadcrumbLink>
    </BreadcrumbItem>
    <BreadcrumbSeparator />
    <BreadcrumbItem>
      <BreadcrumbPage>{feature.name}</BreadcrumbPage>
    </BreadcrumbItem>
  </BreadcrumbList>
</Breadcrumb>
```

---

## 30. Account Context Pattern

> Resolver account_id de slug de forma centralizada.

### Helper Centralizado

```typescript
// File: apps/web/lib/server/account.ts

import 'server-only';

import { cache } from 'react';
import { getSupabaseServerClient } from '@kit/supabase/server-client';

export const getAccountBySlug = cache(async (slug: string) => {
  const client = getSupabaseServerClient();

  const { data, error } = await client
    .from('accounts')
    .select('id, name, slug, picture_url')
    .eq('slug', slug)
    .single();

  if (error || !data) {
    throw new Error(`Account not found: ${slug}`);
  }

  return data;
});

// Atajo para solo obtener ID
export const getAccountIdBySlug = cache(async (slug: string) => {
  const account = await getAccountBySlug(slug);
  return account.id;
});
```

### Uso en Loaders

```typescript
// _lib/server/features.loader.ts

import 'server-only';
import { getAccountBySlug } from '~/lib/server/account';

export async function loadFeaturesPageData(slug: string) {
  const client = getSupabaseServerClient();
  const account = await getAccountBySlug(slug);  // Centralizado

  const { data, error } = await client
    .from('features')
    .select('*')
    .eq('account_id', account.id);

  if (error) throw error;
  return { account, features: data ?? [] };
}
```

### Uso en Server Actions

```typescript
// _lib/server/features.actions.ts

'use server';

import { getAccountIdBySlug } from '~/lib/server/account';

export const createFeature = enhanceAction(
  async (data, { account }) => {
    const client = getSupabaseServerClient();
    const accountId = await getAccountIdBySlug(account);  // Centralizado

    // ... rest of action
  },
  { schema: CreateFeatureSchema }
);
```

---

## 31. Permission-Based UI Rendering

> Mostrar/ocultar UI según permisos del usuario.

### Pattern en Server Components (RECOMENDADO)

```typescript
// page.tsx - verificar permisos server-side

// IMPORTANTE: loadTeamWorkspace es un archivo LOCAL, no un export de package
import { loadTeamWorkspace } from '~/home/[account]/_lib/server/team-account-workspace.loader';

export default async function SettingsPage({ params }: Props) {
  const { account: accountSlug } = await params;

  // loadTeamWorkspace retorna: { account, accounts, user }
  // account viene del RPC team_account_workspace e incluye:
  //   - role: string
  //   - permissions: app_permissions[]
  const { account, user } = await loadTeamWorkspace(accountSlug);

  // CORRECTO: permissions está en account, no en workspace
  const canManage = account.permissions?.includes('settings.manage') ?? false;
  const isOwner = account.role === 'owner';

  if (!canManage) {
    return (
      <Alert variant="warning">
        <AlertTriangle className="h-4 w-4" />
        <AlertTitle>Sin permisos</AlertTitle>
        <AlertDescription>
          No tienes permisos para gestionar esta sección.
          Contacta al owner del equipo.
        </AlertDescription>
      </Alert>
    );
  }

  // Pasar canManage como prop a componentes client
  return <SettingsForm canManage={canManage} isOwner={isOwner} />;
}
```

### Alternativa: Usar API Directamente

```typescript
// Si no quieres usar el loader local
import { createTeamAccountsApi } from '@kit/team-accounts/api';
import { getSupabaseServerClient } from '@kit/supabase/server-client';

export default async function Page({ params }: Props) {
  const { account: slug } = await params;
  const client = getSupabaseServerClient();
  const api = createTeamAccountsApi(client);

  // Obtener workspace
  const { data: workspace } = await api.getAccountWorkspace(slug);
  const account = workspace?.account;

  // Verificar permisos con RPC (más preciso)
  const { data: user } = await client.auth.getUser();
  const canManage = user?.user ? await api.hasPermission({
    accountId: account.id,
    userId: user.user.id,
    permission: 'settings.manage'
  }) : false;

  return <SettingsForm canManage={canManage} />;
}
```

### Hook de Workspace (Solo Client Components)

```typescript
'use client';

// useTeamAccountWorkspace es para COMPONENTES CLIENT
// Retorna el contexto provisto por TeamAccountWorkspaceContextProvider
import { useTeamAccountWorkspace } from '@kit/team-accounts/hooks/use-team-account-workspace';

function FeatureSettings({ canManage }: { canManage: boolean }) {
  // El hook da acceso al workspace context
  const { account } = useTeamAccountWorkspace();

  // NOTA: Para permisos, preferir pasarlos como props desde el server
  // El account del hook tiene: role, permissions, etc.
  const isOwner = account.role === 'owner';

  return (
    <div>
      {/* Visible para todos */}
      <FeaturesList />

      {/* Solo visible si tiene permiso (pasado desde server) */}
      {canManage && (
        <CreateFeatureButton />
      )}

      {/* Solo visible para owner */}
      {isOwner && (
        <DangerZone>
          <DeleteFeatureButton />
        </DangerZone>
      )}
    </div>
  );
}
```

### Permisos Disponibles

```typescript
type AppPermission =
  | 'roles.manage'
  | 'billing.manage'
  | 'settings.manage'
  | 'members.manage'
  | 'invites.manage';

// Agregar permisos custom al enum:
// ALTER TYPE app_permissions ADD VALUE 'features.manage';
```

### Verificación Pre-Uso

**Antes de usar imports de workspace, verificar con Grep:**
```bash
Grep: pattern="exports" path="packages/features/team-accounts/package.json"
# Exports disponibles: "./api", "./components", "./hooks/*"
```

**Antes de acceder a campos, verificar con MCP:**
```
search_database_functions({query: "team_account_workspace"})
# Campos disponibles: id, name, slug, role, permissions[]
```

---

## 32. Table Patterns

> Search, filter, pagination, y empty states.

### Table con Search y Filter

```typescript
'use client';

import { useState, useMemo } from 'react';
import { Input } from '@kit/ui/input';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@kit/ui/select';
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@kit/ui/table';

interface DataTableProps<T> {
  data: T[];
  columns: { key: keyof T; label: string }[];
  searchKey: keyof T;
  filterKey?: keyof T;
  filterOptions?: { value: string; label: string }[];
  emptyMessage?: string;
}

export function DataTable<T extends Record<string, any>>({
  data,
  columns,
  searchKey,
  filterKey,
  filterOptions,
  emptyMessage = 'No hay datos disponibles',
}: DataTableProps<T>) {
  const [search, setSearch] = useState('');
  const [filter, setFilter] = useState('all');

  const filteredData = useMemo(() => {
    return data.filter((item) => {
      // Search
      const matchesSearch = String(item[searchKey])
        .toLowerCase()
        .includes(search.toLowerCase());

      // Filter
      const matchesFilter = filter === 'all' ||
        (filterKey && item[filterKey] === filter);

      return matchesSearch && matchesFilter;
    });
  }, [data, search, filter, searchKey, filterKey]);

  return (
    <div className="space-y-4">
      {/* Controls */}
      <div className="flex items-center gap-4">
        <Input
          placeholder="Buscar..."
          value={search}
          onChange={(e) => setSearch(e.target.value)}
          className="max-w-sm"
        />
        {filterKey && filterOptions && (
          <Select value={filter} onValueChange={setFilter}>
            <SelectTrigger className="w-40">
              <SelectValue placeholder="Filtrar" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="all">Todos</SelectItem>
              {filterOptions.map((opt) => (
                <SelectItem key={opt.value} value={opt.value}>
                  {opt.label}
                </SelectItem>
              ))}
            </SelectContent>
          </Select>
        )}
      </div>

      {/* Table */}
      <Table>
        <TableHeader>
          <TableRow>
            {columns.map((col) => (
              <TableHead key={String(col.key)}>{col.label}</TableHead>
            ))}
          </TableRow>
        </TableHeader>
        <TableBody>
          {filteredData.length > 0 ? (
            filteredData.map((item, i) => (
              <TableRow key={i}>
                {columns.map((col) => (
                  <TableCell key={String(col.key)}>
                    {String(item[col.key])}
                  </TableCell>
                ))}
              </TableRow>
            ))
          ) : (
            <TableRow>
              <TableCell
                colSpan={columns.length}
                className="text-center py-8 text-muted-foreground"
              >
                {emptyMessage}
              </TableCell>
            </TableRow>
          )}
        </TableBody>
      </Table>
    </div>
  );
}
```

### Empty State Component

```typescript
import { LucideIcon } from 'lucide-react';
import { Button } from '@kit/ui/button';

interface EmptyStateProps {
  icon: LucideIcon;
  title: string;
  description: string;
  action?: {
    label: string;
    onClick: () => void;
  };
}

export function EmptyState({ icon: Icon, title, description, action }: EmptyStateProps) {
  return (
    <div className="flex flex-col items-center justify-center py-12 text-center">
      <Icon className="h-12 w-12 text-muted-foreground/50 mb-4" />
      <h3 className="text-lg font-medium">{title}</h3>
      <p className="text-sm text-muted-foreground mt-1 max-w-sm">{description}</p>
      {action && (
        <Button onClick={action.onClick} className="mt-4">
          {action.label}
        </Button>
      )}
    </div>
  );
}
```

---

## 33. Loading States

> Skeleton loaders mientras se carga contenido.

### loading.tsx Pattern

```typescript
// apps/web/app/home/[account]/features/loading.tsx

import { Skeleton } from '@kit/ui/skeleton';

export default function FeaturesLoading() {
  return (
    <div className="space-y-6">
      {/* Breadcrumb skeleton */}
      <Skeleton className="h-4 w-48" />

      {/* Header skeleton */}
      <div className="space-y-2">
        <Skeleton className="h-8 w-64" />
        <Skeleton className="h-4 w-96" />
      </div>

      {/* Controls skeleton */}
      <div className="flex gap-4">
        <Skeleton className="h-10 w-64" />
        <Skeleton className="h-10 w-32" />
      </div>

      {/* Table skeleton */}
      <div className="space-y-2">
        <Skeleton className="h-12 w-full" />
        {[1, 2, 3, 4, 5].map((i) => (
          <Skeleton key={i} className="h-16 w-full" />
        ))}
      </div>
    </div>
  );
}
```

### Card Grid Loading

```typescript
// Para vistas de cards
export default function CardsLoading() {
  return (
    <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
      {[1, 2, 3, 4, 5, 6].map((i) => (
        <div key={i} className="rounded-lg border p-4 space-y-3">
          <Skeleton className="h-4 w-3/4" />
          <Skeleton className="h-3 w-1/2" />
          <Skeleton className="h-20 w-full" />
        </div>
      ))}
    </div>
  );
}
```

---

## 34. Danger Zone Pattern

> Sección visual para acciones destructivas.

### Component

```typescript
import { Card, CardHeader, CardTitle, CardDescription, CardContent } from '@kit/ui/card';
import { Button } from '@kit/ui/button';
import { AlertTriangle } from 'lucide-react';

interface DangerZoneProps {
  title?: string;
  description?: string;
  children: React.ReactNode;
}

export function DangerZone({
  title = 'Zona de Peligro',
  description = 'Las siguientes acciones son irreversibles.',
  children,
}: DangerZoneProps) {
  return (
    <Card className="border-destructive">
      <CardHeader>
        <CardTitle className="flex items-center gap-2 text-destructive">
          <AlertTriangle className="h-5 w-5" />
          {title}
        </CardTitle>
        <CardDescription>{description}</CardDescription>
      </CardHeader>
      <CardContent>{children}</CardContent>
    </Card>
  );
}

// Uso:
<DangerZone>
  <div className="space-y-4">
    <p className="text-sm text-muted-foreground">
      Esta acción eliminará permanentemente todos los datos asociados.
    </p>
    <Button variant="destructive" onClick={handleDelete}>
      Eliminar permanentemente
    </Button>
  </div>
</DangerZone>
```

### Delete Confirmation Pattern

```typescript
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
  AlertDialogTrigger,
} from '@kit/ui/alert-dialog';

<AlertDialog>
  <AlertDialogTrigger asChild>
    <Button variant="destructive">Eliminar</Button>
  </AlertDialogTrigger>
  <AlertDialogContent>
    <AlertDialogHeader>
      <AlertDialogTitle>¿Estás seguro?</AlertDialogTitle>
      <AlertDialogDescription>
        Esta acción no se puede deshacer. Se eliminarán permanentemente
        todos los datos asociados.
      </AlertDialogDescription>
    </AlertDialogHeader>
    <AlertDialogFooter>
      <AlertDialogCancel>Cancelar</AlertDialogCancel>
      <AlertDialogAction onClick={handleConfirmDelete}>
        Sí, eliminar
      </AlertDialogAction>
    </AlertDialogFooter>
  </AlertDialogContent>
</AlertDialog>
```

---

## 35. Page Layout Patterns

> Estructura consistente de páginas.

### PageHeader Component

```typescript
// Usar el componente existente de @kit/ui
import { PageHeader } from '@kit/ui/page-header';

<PageHeader
  title="Features"
  description="Manage your features and configurations"
>
  {/* Action button (top right) */}
  <Button>
    <Plus className="h-4 w-4 mr-2" />
    Create Feature
  </Button>
</PageHeader>
```

### Standard Page Layout

```typescript
// Template de página estándar
export default async function FeaturePage({ params }: Props) {
  const { account } = await params;
  const { account: accountData, features } = await loadPageData(account);

  return (
    <div className="space-y-6">
      {/* 1. Breadcrumb */}
      <Breadcrumb>
        <BreadcrumbList>
          <BreadcrumbItem>
            <BreadcrumbLink href={`/home/${account}`}>Home</BreadcrumbLink>
          </BreadcrumbItem>
          <BreadcrumbSeparator />
          <BreadcrumbItem>
            <BreadcrumbPage>Features</BreadcrumbPage>
          </BreadcrumbItem>
        </BreadcrumbList>
      </Breadcrumb>

      {/* 2. Page Header */}
      <PageHeader
        title="Features"
        description="Manage your features"
      >
        <CreateFeatureDialog accountSlug={account} />
      </PageHeader>

      {/* 3. Filters/Controls (optional) */}
      <div className="flex items-center gap-4">
        <Input placeholder="Search..." className="max-w-sm" />
        <StatusFilter />
      </div>

      {/* 4. Main Content */}
      <FeaturesTable data={features} accountSlug={account} />

      {/* 5. Danger Zone (if applicable) */}
      {isOwner && (
        <DangerZone>
          <DeleteAllButton />
        </DangerZone>
      )}
    </div>
  );
}
```

### Status Badge Variants

```typescript
import { Badge } from '@kit/ui/badge';

// Mapa de variantes por status
const STATUS_VARIANTS = {
  // Estados generales
  active: 'default',
  inactive: 'secondary',
  pending: 'outline',
  error: 'destructive',

  // Estados de pago
  confirmed: 'default',
  detected: 'secondary',
  overdue: 'destructive',

  // Estados de tarea
  draft: 'outline',
  in_progress: 'secondary',
  completed: 'default',
  cancelled: 'destructive',
} as const;

// Uso
<Badge variant={STATUS_VARIANTS[status]}>
  {status}
</Badge>
```

---

## 36. Blueprint Generation Checklist (Updated)

### Pre-Generation

- [ ] ¿Qué pattern de DB? (18-25)
- [ ] ¿Requiere permisos custom?
- [ ] ¿Dónde va en sidebar? (routes vs settings)

### Blueprint Must Include

- [ ] **Database:** Schema + RLS + Indexes
- [ ] **Server:** Schemas + Loaders + Actions
- [ ] **UI:** Breadcrumb + PageHeader + Table/Cards
- [ ] **Loading:** loading.tsx skeleton
- [ ] **Permissions:** Conditional rendering
- [ ] **Navigation:** Sidebar entry
- [ ] **Empty States:** Para tablas vacías
- [ ] **Danger Zone:** Si hay acciones destructivas

### Post-Generation Verify

- [ ] Sidebar navigation added
- [ ] Breadcrumb implemented
- [ ] Loading state created
- [ ] Empty states included
- [ ] Permission checks in UI
- [ ] Danger zone if needed

---

*Quick reference auto-contenido para makerkit-planner plugin v1.4.0*
