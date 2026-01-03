---
type: inventory
version: "{VERSION}"
generated: "{DATE}"
makerkit_version: "{MAKERKIT_VERSION}"
project: "{PROJECT_NAME}"
---

# Inventario MakerKit - {VERSION}

> Generado: {DATE}
> MakerKit Version: {MAKERKIT_VERSION}
> Proyecto: {PROJECT_NAME}

---

## Decisiones Arquitecturales

| Decision | Valor | Razon |
|----------|-------|-------|
| Account Type | Team / Personal | {DECISION_REASON} |
| Billing | Activo / No usado | {DECISION_REASON} |
| Multi-language | Si / No | {DECISION_REASON} |
| Auth Providers | Email, Google, ... | {DECISION_REASON} |

---

## Features en este Proyecto

| Feature | Tabla Base | Account Type | RLS Pattern |
|---------|-----------|--------------|------------|
| [Rellenar despues de generar blueprints] | - | - | - |

---

## Tablas MakerKit (Usamos)

| Tabla | Proposito | Relacion con nuestras features |
|-------|-----------|-------------------------------|
| `accounts` | Team accounts | FK para todas las entidades de negocio |
| `accounts_memberships` | Miembros de equipos | Gestion de acceso |
| `roles` | Definicion de roles | owner, admin, member |
| `role_permissions` | Permisos por rol | RLS policies |

---

## Tablas MakerKit (No Usamos)

| Tabla | Proposito | Por que no |
|-------|-----------|-----------|
| `subscriptions` | Billing/suscripciones | MVP sin billing |
| `subscription_items` | Items de suscripcion | MVP sin billing |
| `orders` | Ordenes de pago | MVP sin billing |
| `order_items` | Items de orden | MVP sin billing |
| `billing_customers` | Clientes Stripe/Paddle | MVP sin billing |
| `notifications` | Sistema notificaciones | Simplificar MVP |
| `nonces` | Tokens one-time | No requerido |

---

## Funciones Helper Disponibles

### Autorizacion y RLS

| Funcion | Uso | Ejemplo |
|---------|-----|---------|
| `has_role_on_account(account_id)` | Verificar si usuario tiene rol en cuenta | RLS policies (team) |
| `has_permission(user_id, account_id, permission)` | Verificar permiso especifico | RLS granular |
| `is_account_owner(account_id)` | Verificar si es owner | Operaciones criticas |
| `is_super_admin()` | Verificar super admin | Panel admin |

### Utilidades

| Funcion | Uso | Ejemplo |
|---------|-----|---------|
| `slugify(text)` | Convertir texto a slug | URLs amigables |
| `trigger_set_timestamps()` | Auto-actualizar created/updated | Triggers |
| `trigger_set_user_tracking()` | Trackear usuario que modifica | Auditoria |

---

## Enums Disponibles

| Enum | Valores | Para reutilizar |
|------|---------|-----------------|
| `app_permissions` | roles.manage, members.manage, billing.manage, settings.manage, invites.manage | RLS con permisos |
| `subscription_status` | active, trialing, past_due, canceled, unpaid, incomplete, paused | [No usado] |
| `payment_status` | pending, succeeded, failed | [No usado] |
| `notification_channel` | in_app, email | [No usado] |
| `notification_type` | info, warning, error | [No usado] |

---

## Componentes UI Reutilizables

### Forms y Data

| Componente | Import | Uso |
|------------|--------|-----|
| DataTable | `@kit/ui/enhanced-data-table` | Listas con sorting/filtering |
| Form | `@kit/ui/form` | Forms con react-hook-form |
| Sheet | `@kit/ui/sheet` | Modales laterales create/edit |
| AlertDialog | `@kit/ui/alert-dialog` | Confirmacion de acciones |

### Feedback

| Componente | Import | Uso |
|------------|--------|-----|
| Button | `@kit/ui/button` | Acciones |
| Badge | `@kit/ui/badge` | Estados |
| Spinner | `@kit/ui/spinner` | Loading states |
| EmptyState | `@kit/ui/empty-state` | Listas vacias |

---

## Patrones de Codigo

### Server Action

```typescript
'use server';

import { enhanceAction } from '@kit/next/actions';
import { z } from 'zod';

const Schema = z.object({...});

export const createEntityAction = enhanceAction(
  async (data) => {
    const client = getSupabaseServerClient();
    // ... implementation
    return { success: true };
  },
  { schema: Schema }
);
```

### Page Loader

```typescript
import 'server-only';

export async function loadEntityPageData(
  client: SupabaseClient<Database>,
  accountId: string
) {
  const { data, error } = await client
    .from('entities')
    .select('*')
    .eq('account_id', accountId);

  if (error) throw error;
  return data ?? [];
}
```

### RLS Policy (Team Account)

```sql
CREATE POLICY "Team members can view"
ON public.entities FOR SELECT
TO authenticated
USING (
  public.has_role_on_account(account_id)
);
```

---

## Rutas Base

```
apps/web/app/
├── (marketing)/          # Landing pages
├── auth/                 # Autenticacion
├── home/
│   ├── (user)/          # Dashboard personal
│   └── [account]/       # Dashboard de equipo
│       ├── settings/    # Configuracion
│       ├── members/     # Miembros
│       ├── billing/     # Billing [no usado]
│       └── {features}/  # Nuestras features
└── admin/               # Panel admin
```

---

## Archivos CLAUDE.md Relevantes

| Archivo | Contiene |
|---------|----------|
| `CLAUDE.md` (root) | Comandos, stack, estructura general |
| `apps/web/CLAUDE.md` | Patrones Next.js, rutas, async params |
| `apps/web/supabase/CLAUDE.md` | Schemas, migrations, RLS, naming |
| `packages/features/CLAUDE.md` | Personal vs Team accounts |
| `packages/ui/CLAUDE.md` | Componentes disponibles |
| `packages/next/CLAUDE.md` | enhanceAction, enhanceRouteHandler |

---

## Notas del Proyecto

{PROJECT_NOTES}

---

*Generado por /makerkit-blueprint*
