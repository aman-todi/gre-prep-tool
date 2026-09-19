# GRE Verbal Reps

A free, self-updating daily GRE Verbal practice tool. Every morning, a scheduled Claude run writes a fresh 16-item set (text completion, sentence equivalence, reading comprehension — current GRE format) to a page you bookmark once, and tracks which of a growing vocabulary bank you've been drilled on. No app, no server, no bill.

**Setup is one prompt.** You don't hand-build a database or a webpage here — you connect a free Supabase project to Claude, then run the [`gre-prep-setup`](./skills/gre-prep-setup/SKILL.md) skill once. Claude designs the schema, generates the initial vocabulary bank, publishes the practice page, and creates the daily scheduled task, end to end, in that one run.

## How it works

```
   one-time setup                          every morning after
        │                                          │
        ▼                                          ▼
┌────────────────────┐                   ┌────────────────────┐
│  gre-prep-setup     │   creates         │  Scheduled Claude   │
│  skill runs once,   │──schema, seed────►│  task fires         │
│  in your Claude     │   words, task     │  (no memory of      │
│  session             │                   │   any prior run)    │
└─────────┬───────────┘                   └─────────┬───────────┘
          │                                          │ reads config + word
          │ publishes                                │ history, writes new
          ▼                                          ▼ content, logs session
┌────────────────────┐                   ┌────────────────────┐
│  Claude Artifact     │◄──── republished ─┤  Supabase (free)    │
│  (the page you open  │      in place      │  words / config /   │
│   each morning)      │      each morning  │  daily_sessions     │
└────────────────────┘                   └────────────────────┘
```

- **Supabase** holds three tables: `words` (the practice bank, with per-word coverage tracking), `daily_sessions` (a log used to avoid repeating passage topics and words), and `config` (the knobs — item counts, coverage goals, the artifact's URL — that every run reads instead of hardcoding numbers). The setup skill creates and seeds all three; nothing to write by hand.
- **The artifact** is one self-contained HTML page with a `var DAY = {...}` object holding that day's content. It has no backend of its own — the scheduled run edits `DAY` and republishes to the same URL every morning, so the link you bookmark never changes. Your streak and completion state live in the browser's `localStorage`, per device.
- **The scheduled task** is a Claude prompt that fires daily with no memory of any previous run — it reads everything it needs from Supabase and the live artifact each time, generates and self-checks new content, publishes, then logs what it did back to Supabase.

## Set it up (free, ~10 minutes, mostly waiting on Supabase to provision)

1. [Create a free Supabase account and project](./docs/01-supabase-setup.md) — just the account and an empty project. No schema to write.
2. [Connect that project to Claude via the Supabase MCP connector](./docs/02-connect-mcp.md)
3. [Run the `gre-prep-setup` skill once](./docs/03-run-the-setup-skill.md) — this is the step that builds everything

After that, it runs itself. The only thing you keep is the artifact link.

## Repo layout

```
docs/
  01-supabase-setup.md        free account + empty project — nothing else
  02-connect-mcp.md           connecting the Supabase MCP connector, scoped permissions
  03-run-the-setup-skill.md   how to actually invoke the skill, what it asks, what it produces
skills/
  gre-prep-setup/SKILL.md     the one-time bootstrap: schema, vocabulary bank, artifact, scheduled task
```

## Why no seed word list or starter page in this repo

Earlier drafts of this repo shipped a static ~800-word CSV and a pre-built artifact HTML file to copy in by hand. That's the wrong shape for something meant to be cloned and reused: a fixed word list goes stale and locks everyone into one person's choices, and a hand-copied artifact is one more manual step that can drift from what the scheduled prompt actually expects. Now Claude generates the word bank and builds the artifact itself, from the same skill that sets up the schema and the scheduled task — so there's one source of truth (`skills/gre-prep-setup/SKILL.md`) instead of three files that can go out of sync.

## Making it your own

Everything that shapes daily output lives in Supabase, not the scheduled prompt, on purpose:

- Tune item counts, coverage targets, or the review lookback window → edit `config`
- Grow, trim, or correct the vocabulary → edit `words` directly, or just ask Claude to add words
- Restyle the page → edit the artifact's HTML/CSS directly (the scheduled run only ever touches the `DAY` object, so a restyle survives future runs)

## A note on Row Level Security

Supabase's own advisors will flag that RLS is off on all three tables — the setup skill leaves it off deliberately. That's safe *for this architecture*: the artifact is a static page with no direct Supabase client in the browser, so the anon/publishable key is never exposed or used — the only thing that ever touches the database is a Claude session, authenticated through your own MCP connection. If you extend this to query Supabase directly from the browser, enable RLS and add policies first.
