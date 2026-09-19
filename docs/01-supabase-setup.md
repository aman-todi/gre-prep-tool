# 1. Create a free Supabase account and project

Supabase is this tool's memory: a Postgres database that will hold your vocabulary bank, your config, and a log of every day's session. You're not writing any SQL by hand — the [`gre-prep-setup` skill](../skills/gre-prep-setup/SKILL.md) does that in step 3. This step is just getting an empty project to point it at.

## Create the account and project

1. Go to [supabase.com](https://supabase.com) and sign up (GitHub sign-in is fastest). No card required for the free tier.
2. Click **New project**.
3. Fill in:
   - **Name** — anything, e.g. `gre-prep-tracker`
   - **Database password** — generate one and save it somewhere safe. You won't need to type it day-to-day (Claude talks to the project through the MCP connection, not this password), but keep it in case you ever want direct `psql`/connection-string access.
   - **Region** — whatever's closest to you.
   - **Plan** — Free.
4. Click **Create new project** and wait ~1-2 minutes for provisioning. Leave every table empty — nothing to configure here.

That's it. Next: [connect this project to Claude](./02-connect-mcp.md).
