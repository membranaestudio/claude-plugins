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

*Quick reference auto-contenido para makerkit-planner plugin v1.3.0*
