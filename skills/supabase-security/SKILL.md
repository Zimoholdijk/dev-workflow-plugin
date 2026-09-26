---
name: supabase-security
description: Research Supabase best practices for a specific security issue, verify the current state (live through the Supabase MCP when connected, otherwise from the migration files plus read-only queries the user runs), and apply the fix. Use for RLS policies, SECURITY DEFINER functions, storage policies, and other Supabase security hardening.
disable-model-invocation: false
argument-hint: "<issue description>"
---

# Supabase Security Fix Workflow

You are fixing a specific Supabase security issue: $ARGUMENTS

## Step 1: Research

Research best practices for this specific issue in Supabase's official documentation: through the Supabase MCP `search_docs` tool if it is connected, otherwise on supabase.com/docs. Search for:
- The specific Supabase feature involved (RLS, storage policies, SECURITY DEFINER, etc.)
- Security hardening recommendations
- Common pitfalls and how to avoid them

Read the relevant documentation carefully. Do not rely on training data: the official docs are the source of truth.

## Step 2: Verify current state

Read the migration files to learn what the schema, policies, functions, and grants should be. Then check what is actually deployed, because the live database is the source of truth and can drift from the files:

- **With the Supabase MCP connected:** run read-only checks yourself. Use `execute_sql` (SELECT only) to inspect current policies, functions, grants, and triggers, `list_tables` for table structure, and `list_extensions` if the fix involves extensions. Show each query before it runs.
- **Without it:** give the user the exact read-only queries to run in the SQL editor or with the CLI, say what each result would change about the fix, and wait for the results.

Compare the live state against the migration files. If the user cannot run the queries, say plainly that the fix is based on the migration files alone and that live state is unverified.

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
- If the fix is application code, run the project's type checker or linter if it has one
- Run the existing tests that cover the changed code, using the project's test command (see `.claude/CLAUDE.md`)

## Output

After completing the fix, report:
1. What was changed and why
2. What the user needs to do to apply it (e.g., `supabase db push` for migrations)
3. Any remaining risks or follow-up items
