---
name: supabase-security
description: Research Supabase best practices for a specific security issue, verify current state via MCP, and apply the fix. Use for RLS policies, SECURITY DEFINER functions, storage policies, and other Supabase security hardening.
disable-model-invocation: false
argument-hint: "<issue description>"
---

# Supabase Security Fix Workflow

You are fixing a specific Supabase security issue: $ARGUMENTS

## Step 1: Research

Use the Supabase MCP `search_docs` tool to research best practices for this specific issue. Search for:
- The specific Supabase feature involved (RLS, storage policies, SECURITY DEFINER, etc.)
- Security hardening recommendations
- Common pitfalls and how to avoid them

Read the relevant documentation carefully. Do not rely on training data: the MCP docs are the source of truth.

## Step 2: Verify current state

Use the Supabase MCP tools to check the current live database state:
- `execute_sql` to inspect current policies, functions, grants, triggers
- `list_tables` to check table structure if needed
- `list_extensions` if the fix involves extensions

Compare the live state against what the migration files say. The live DB is the source of truth for what's currently deployed.

## Step 3: Determine the fix

Based on the research and current state:
1. Identify exactly what needs to change
2. Check if there's an existing migration that partially addresses this
3. Determine whether this needs a new migration file or an application-code change

**Important rules:**
- Migrations are created as local files in `supabase/migrations/`: never use the MCP `apply_migration` tool
- Migration files are schema-only: no data manipulation (UPDATE, DELETE, INSERT on user data)
- If the fix is application code (e.g., adding validation in a route handler), edit the source file directly
- Surface any trade-offs to the user before applying

## Step 4: Apply the fix

- For database changes: Create a new migration file with a descriptive name in `supabase/migrations/`
- For application code: Edit the relevant source files
- Include clear comments explaining WHY the change is needed (security context)

## Step 5: Verify

- Re-read the changed files to confirm correctness
- If the fix is a migration, verify SQL syntax is valid
- If the fix is application code, check for TypeScript errors
- Run existing tests if they cover the changed code (`npx vitest run` for the specific test file)

## Output

After completing the fix, report:
1. What was changed and why
2. What the user needs to do to apply it (e.g., `supabase db push` for migrations)
3. Any remaining risks or follow-up items
