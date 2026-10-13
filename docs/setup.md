# Setup

It's free and takes about 10 minutes, most of it waiting for Supabase to provision.

## 1. Create a Supabase project

1. Sign up at [supabase.com](https://supabase.com). The free tier needs no card.
2. Click **New project**. Choose any name and the nearest region, and keep the generated password.
3. Wait for it to provision and leave it empty. The skill creates everything.

## 2. Connect Supabase to Claude

1. In Claude, open **Settings → Connectors → Supabase → Connect**.
2. Authorize your project, or your whole org.
3. To check it works, ask Claude: "List my Supabase projects."

The tool only needs `execute_sql`. Setup also uses `list_projects` to confirm which project to use.

## 3. Run the setup skill

You need a Claude surface that can use the Supabase connector, publish an Artifact and create a scheduled routine, all in one session.

- If your surface supports skills, enable `skills/gre-prep-setup/` and run it.
- Otherwise, paste the body of [`SKILL.md`](../skills/gre-prep-setup/SKILL.md) into a new conversation.

The skill asks at most two things: which project to use (only if you have more than one) and what time the set should be ready (default 6:00 AM). It finishes with the page link, which is the only thing you need to keep.
