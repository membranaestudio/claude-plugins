# CLAUDE.md Files Index

Quick reference to MakerKit's CLAUDE.md documentation files.

## File Locations

```
/CLAUDE.md                      → Root project config
/apps/web/CLAUDE.md             → Web app patterns
/apps/web/supabase/CLAUDE.md    → Database patterns
/apps/web/app/admin/CLAUDE.md   → Admin patterns
/apps/e2e/CLAUDE.md             → E2E test patterns
/packages/features/CLAUDE.md    → Feature patterns
/packages/supabase/CLAUDE.md    → Supabase helpers
/packages/ui/CLAUDE.md          → UI components
/packages/next/CLAUDE.md        → Next.js utilities
/packages/mailers/CLAUDE.md     → Email patterns
/packages/analytics/CLAUDE.md   → Analytics patterns
/packages/policies/CLAUDE.md    → Policy patterns
/packages/mcp-server/CLAUDE.md  → MCP server config
```

## Key Content by File

### Root CLAUDE.md
- Project status and setup
- Core technologies (Next.js 16, Supabase, React 19)
- Essential commands
- Multi-tenant architecture overview
- TypeScript and React patterns
- Data fetching architecture

### apps/web/CLAUDE.md
- Route structure patterns
- Async params handling (Next.js 16)
- Page component patterns
- Data fetching with loaders

**Critical Pattern - Async Params:**
```typescript
// CORRECT (Next.js 16)
async function Page({ params }: Props) {
  const { account } = await params;
}

// WRONG
async function Page({ params }: Props) {
  const { account } = use(params);
}
```

### apps/web/supabase/CLAUDE.md
- Schema file organization
- Migration workflow
- RLS policy patterns
- Declarative schema approach

**Schema Workflow:**
1. Create schema in `schemas/XX-name.sql`
2. Create migration with `supabase migrations new`
3. Copy schema to migration
4. Apply with `supabase:web:reset`
5. Generate types with `supabase:web:typegen`

### packages/features/CLAUDE.md
- Personal vs Team accounts
- Account workspace patterns
- Feature package structure

**Account Types:**
- Personal: `auth.users.id = accounts.id`
- Team: Shared workspace with members

### packages/supabase/CLAUDE.md
- Database helper functions
- RLS utility functions
- Trigger functions

**Key Functions:**
- `is_team_member(account_id, user_id)` - RLS for team resources
- `has_role_on_account(user_id, account_id)` - Check membership
- `has_permission(user_id, account_id, permission)` - Permission check
- `trigger_set_timestamps()` - Auto-update dates
- `trigger_set_user_tracking()` - Auto-set created_by/updated_by

### packages/ui/CLAUDE.md
- Available UI components
- Form patterns with react-hook-form
- Component import patterns

**Form Pattern:**
```typescript
import { Form, FormField, FormItem, FormLabel, FormControl, FormMessage } from '@kit/ui/form';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
```

### packages/next/CLAUDE.md
- `enhanceAction` for server actions
- `enhanceRouteHandler` for API routes
- `withI18n` for page exports

**Server Action Pattern:**
```typescript
export const myAction = enhanceAction(
  async (data, { user }) => {
    // implementation
    return { success: true };
  },
  { schema: MySchema }
);
```

## When to Read Each File

| Task | Read |
|------|------|
| Creating new feature | apps/web/CLAUDE.md, packages/features/CLAUDE.md |
| Database changes | apps/web/supabase/CLAUDE.md |
| RLS policies | packages/supabase/CLAUDE.md |
| Form components | packages/ui/CLAUDE.md |
| Server actions | packages/next/CLAUDE.md |
| Understanding project | CLAUDE.md (root) |

## Quick Commands Reference

```bash
# Development
pnpm dev                        # Start all apps

# Database
pnpm supabase:web:start         # Start Supabase
pnpm supabase:web:reset         # Reset with schema
pnpm supabase:web:typegen       # Generate types
pnpm --filter web supabase migrations new NAME  # New migration

# Verification
pnpm typecheck                  # Type check
pnpm lint:fix                   # Fix lint issues
pnpm format:fix                 # Format code
```
