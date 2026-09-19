# 2. Connect Supabase to Claude (MCP)

Claude talks to your Supabase project through Supabase's own MCP (Model Context Protocol) connector. This is what lets Claude create the schema during setup, and later lets a scheduled run execute SQL against your database with nobody watching. It's a first-party connector Anthropic ships — you're not installing or hosting anything.

## Add the connector

1. In Claude (claude.ai or the desktop app), open **Settings → Connectors** (naming may vary slightly by surface/version — look for "Connectors" or "Integrations").
2. Find **Supabase** and click **Connect**.
3. You'll go through Supabase's OAuth flow — sign in with the account you used in step 1, and authorize the connection.
4. When asked which project(s) to grant access to, select the project you just created (or grant org-wide access — either works, since every prompt this tool uses names the project id explicitly rather than assuming which project is meant).

## Permissions

The Supabase MCP connector exposes a set of tools (execute SQL, list tables, apply migrations, deploy edge functions, manage branches, pause/restore/delete the project, and more). This tool only ever needs:

- **`execute_sql`** — required, for everything: creating the schema during setup, and reading/writing rows every day after.
- `list_tables`, `get_project`, `list_projects`, `get_advisors` — optional, harmless to leave on; the setup skill uses `list_projects`/`get_project` to confirm which project it's working against, and you may find `get_advisors` useful later for a security check.

Branching, edge functions, pausing/restoring/deleting the project, and anything else in the connector's tool list are unnecessary for this tool. If your Claude surface lets you scope tool permissions per-connector or per-scheduled-task, turn those off — least privilege.

## Verify the connection works

Open a normal Claude conversation and ask it to run:

```
Using the Supabase connector, list my projects and confirm you can reach the one I just created.
```

If it comes back naming your project, the connection is live and scoped correctly. Next: [run the setup skill](./03-run-the-setup-skill.md).
