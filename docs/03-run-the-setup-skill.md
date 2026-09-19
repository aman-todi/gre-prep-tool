# 3. Run the setup skill once

Everything else in this repo — the schema, the initial vocabulary, the practice page, the daily scheduled task — gets built by a single Claude session running [`skills/gre-prep-setup/SKILL.md`](../skills/gre-prep-setup/SKILL.md). You run this once; after it finishes, the tool maintains itself.

## What you need before starting

- The Supabase connector connected and scoped to your project (steps 1-2).
- A Claude surface that can, in one session: run Supabase MCP tools, publish a Claude Artifact, and create a scheduled task. Claude Code and the Claude desktop/web app with scheduled tasks enabled all qualify. If your surface can't create scheduled tasks, the skill will still build the schema, vocabulary, and artifact — it'll just tell you it can't automate the last step, and you'll need to create the scheduled task by hand using the prompt it hands you.

## Run it

**If your Claude surface supports custom skills** (e.g. Claude Code with this repo checked out, or a surface where you can install a skill from a folder): install/enable the `gre-prep-setup` skill from `skills/gre-prep-setup/`, then invoke it (e.g. `/gre-prep-setup` or by asking Claude to run the skill).

**If it doesn't** (e.g. a plain claude.ai chat): open [`skills/gre-prep-setup/SKILL.md`](../skills/gre-prep-setup/SKILL.md), copy everything below the frontmatter, and paste it into a new Claude conversation as your message, with the Supabase connector attached to that conversation.

Either way, you're handing Claude the same instructions — the only difference is whether your tooling loads them for you or you paste them yourself.

## What it will ask you

Two things, no more:

1. **Which Supabase project**, if more than one is reachable through your connection (it'll otherwise just confirm the one it finds and move on).
2. **What time you want your daily set ready**, e.g. "6:00 AM" — it'll default to 6:00 AM in your local timezone if you don't have a preference.

Everything else — item mix, vocabulary size and tiering, page design, coverage strategy — it decides using the defaults baked into the skill, because those are meant to be reasonable out of the box and easy to tune in `config` afterward rather than interrogate you about upfront.

## What it produces

- A populated Supabase project: `words`, `daily_sessions`, and `config` tables, with an initial vocabulary bank already loaded.
- A published Claude Artifact — the page you'll open every morning — already showing a real, usable Day 1 set (not a placeholder).
- A daily scheduled task, created and enabled, using the prompt spelled out inside the skill with your actual project id and artifact URL filled in.

It'll finish by giving you the artifact link. That's the only thing worth bookmarking — everything behind it updates on its own.
