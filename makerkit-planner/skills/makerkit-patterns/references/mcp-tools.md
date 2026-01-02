# MakerKit MCP Tools Reference

Complete reference for all MakerKit MCP tools available for codebase analysis.

## Database Tools

### get_database_summary()
Returns comprehensive overview of all tables, enums, and functions.

**Use first** - This gives you the complete picture.

```
Output:
- tables: [{name, schema, topic, sourceFile}]
- enums: [{name, values}]
- functions: [{name, schema, purpose}]
- tablesByTopic: {topic: [table names]}
```

### get_table_info({tableName, schema?})
Returns detailed schema for a specific table.

```
Parameters:
- tableName: string (required)
- schema: string (default: "public")

Output:
- columns with types, nullability, defaults
- foreign keys with references
- indexes
- RLS policies
```

### get_all_enums()
Returns all enum types with their values.

```
Output:
- [{name, values: []}]
```

### get_schemas_by_topic({topic})
Returns schema files related to a topic.

```
Topics: accounts, auth, billing, permissions, teams, notifications
```

### get_schema_content({fileName})
Returns raw content of a schema file.

### search_database_functions({query})
Search functions by name or purpose.

```
Example: search_database_functions({query: "team"})
Returns: is_team_member, create_team_account, etc.
```

### get_function_details({functionName})
Returns complete function implementation.

## Feature Analysis Tools

### find_complete_features()
Lists all complete features in the codebase, grouped by domain.

```
Output:
- features by domain
- completeness indicators (has DB, server, UI)
```

### analyze_feature_pattern({feature_name})
Analyzes a complete feature end-to-end.

```
Output:
- Database layer (tables, RLS)
- Server layer (actions, loaders)
- UI layer (routes, components)
- Patterns used
```

### compare_feature_patterns({feature1, feature2})
Compares two features to find differences and best practices.

## Route Tools

### get_app_routes({filter?})
Maps complete Next.js App Router structure.

```
Parameters:
- filter: string (optional, e.g., "dashboard", "auth")

Output:
- routes with types (page, layout, route)
- dynamic segments
- route groups
```

### find_route_by_path({path})
Finds the file for an HTTP path.

```
Example: find_route_by_path({path: "/dashboard/posts/123"})
Returns: apps/web/app/dashboard/posts/[id]/page.tsx
```

### analyze_route_structure({route})
Analyzes a route's exports, metadata, and data fetching.

## Server Action Tools

### get_server_actions({includeAll?})
Lists all server actions grouped by category.

```
Output:
- actions by feature/category
- file paths
```

### search_server_actions({query})
Search actions by keyword.

### analyze_action_pattern({filePath})
Analyzes security patterns in a server action.

```
Output:
- Auth patterns used
- Validation approach
- RLS reliance
```

## Component Tools

### get_components()
Lists all UI components from the monorepo.

```
Output:
- design-system components (@kit/ui)
- route-specific components
```

### components_search({query})
Search components by keyword.

```
Example: components_search({query: "form"})
Returns: Form, FormField, FormItem, etc.
```

### get_component_props({componentName})
Returns props, interfaces, and variants.

### get_component_content({componentName})
Returns full source code.

## RLS Tools

### generate_rls_policy({table, access_type, pattern?, roleHierarchy?})
Generates RLS policy SQL following MakerKit patterns.

```
Parameters:
- table: string (required)
- access_type: "select" | "insert" | "update" | "delete" | "all"
- pattern: "personal_account" | "user_specific" | "permission_based" (auto-detected if omitted)
- roleHierarchy: boolean (for permission_based pattern)

Output: Complete SQL for the policy
```

### validate_rls_policies({table})
Validates existing RLS policies.

```
Output:
- Missing policies
- Overly permissive access
- Performance issues
```

### generate_rls_tests({table, scenarios?})
Generates test SQL for RLS policies.

## Script Tools

### get_scripts()
Lists all npm/pnpm scripts with descriptions.

### get_script_details({scriptName})
Detailed info about a specific script.

### get_healthcheck_scripts()
Returns scripts to run after writing code.

```
Output: typecheck, lint:fix, format:fix, test
```

## Usage Patterns

### Starting a New Feature

```
1. get_database_summary()           → Understand existing structure
2. find_complete_features()         → Find reference feature
3. analyze_feature_pattern(ref)     → Study the pattern
4. generate_rls_policy(new_table)   → Generate security
```

### Understanding an Existing Feature

```
1. find_route_by_path("/path")      → Find the files
2. analyze_route_structure(route)   → Understand structure
3. get_table_info(table)            → See the data model
4. analyze_action_pattern(action)   → Check security
```

### Validating Before Implementation

```
1. validate_rls_policies(table)     → Check security complete
2. get_healthcheck_scripts()        → Know what to run
```
