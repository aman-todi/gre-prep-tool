---
name: gre-prep-setup
description: One-time bootstrap for a free, self-updating daily GRE Verbal practice tool built on Supabase and a Claude Artifact. Creates the database schema, generates an initial vocabulary bank, publishes the practice page with a real first day of content, and creates the daily scheduled task that regenerates it every morning. Use when asked to set up, install, or bootstrap the GRE prep tool, or to replicate someone else's copy of it.
---

# GRE Verbal Reps — one-time setup

You are bootstrapping a free, fully automated daily GRE Verbal practice tool for the person you're talking to. Run this once, end to end, in this session. After today, a scheduled task takes over and nobody needs to touch this again.

The finished system: a Supabase Postgres database holding a vocabulary bank and a coverage/session log, and a single self-contained HTML page (a Claude Artifact) that a scheduled Claude run rewrites in place every morning with a fresh 16-item GRE Verbal set. You are building all of it in this conversation, then handing off to that scheduled run.

## Before you start: check prerequisites

Confirm, don't assume:
1. The Supabase MCP connector is connected and reachable (e.g. call its `list_projects` or `get_project` tool).
2. You can publish a Claude Artifact in this session.
3. You can create a scheduled task in this session (a tool that schedules a recurring prompt — name varies by surface).

If any of these is missing, say so plainly and stop rather than building part of the system silently. Don't guess at credentials or ask the person for API keys or passwords — the MCP connection is the only access you need.

## Step 1 — Identify the Supabase project

List the Supabase projects reachable through your connection. If there's exactly one, use it and just confirm its name back to the person. If there's more than one, ask which to use. Either way, note its **project ref** (a short id like `owjpkhlxooiutwzmabpz`) — you'll need it repeatedly below. Call it `PROJECT_ID` for the rest of this skill.

## Step 2 — Create the schema

Run this against `PROJECT_ID` (safe to re-run; it won't clobber an existing setup):

```sql
create table if not exists words (
  id                  bigint generated always as identity primary key,
  word                text not null unique,
  pos                 text not null,               -- part of speech: n / v / adj / adv
  definition          text not null,
  tier                smallint not null,            -- 1 = core/most frequent, 2 = common, 3 = advanced/rarer
  times_covered       integer not null default 0,
  first_covered_date  date,
  last_covered_date   date
);

create table if not exists daily_sessions (
  id                bigint generated always as identity primary key,
  session_date      date not null unique,
  day_number        integer not null,
  passage_topics    text[] not null default '{}',
  vocab_words       text[] not null default '{}',
  total_questions   integer,
  notes             text,
  created_at        timestamptz not null default now()
);

create table if not exists config (
  key   text primary key,
  value text not null
);
```

Row Level Security is intentionally left off on all three tables. That's safe here because nothing ever queries Supabase from a browser with the anon/publishable key — the artifact is a static page with no Supabase client in it, and the only thing that ever touches this database is a Claude session authenticated through the MCP connection. Mention this to the person once, in passing, and move on — don't enable RLS yourself (it would need policies you can't design blind, and would break the scheduled run if added without them).

## Step 3 — Seed config

```sql
insert into config (key, value) values
  ('artifact_url', 'PENDING'),
  ('coverage_goal', 'Work toward ~95% coverage of the words table over time, prioritizing tier 1 (essential) words first, then tier 2, then tier 3'),
  ('day_length_minutes', '15-25'),
  ('items_per_day', '16'),
  ('new_words_min_per_day', '10'),
  ('passage_topic_lookback_days', '7'),
  ('questions_per_passage_max', '3'),
  ('questions_per_passage_min', '1'),
  ('rc_passages_per_day', '2'),
  ('target_format', 'Current (post-Sept 2023 shortened) GRE General Test, Verbal Reasoning section'),
  ('word_bank_size', 'PENDING')
on conflict (key) do nothing;
```

These are reasonable defaults, not something to interrogate the person about — they can be changed later just by editing rows in `config`. You'll fill in `artifact_url` and `word_bank_size` for real in a later step.

## Step 4 — Generate the initial vocabulary bank

Generate a substantial GRE-register vocabulary bank from your own knowledge — **do not** invent a small placeholder list. Aim for at least 500-600 words, ideally more, so the daily job has months of non-repeating material before it has to recycle. For each word, be confident the definition and part of speech are correct; if you're not sure a word or its definition is right, leave it out rather than guess — accuracy matters more than hitting a round number.

- **Tier 1** (core/most frequent): the vocabulary that shows up constantly in GRE-level text completion and RC — words most strong test-takers should already half-know.
- **Tier 2** (common): solid GRE-register words, less ubiquitous than tier 1 but still common in official material.
- **Tier 3** (advanced/rarer): the harder, lower-frequency words that show up occasionally and separate very strong verbal scores from good ones.

Aim for roughly the same tier balance as real GRE frequency: tier 1 should be the largest group, tier 3 the smallest (as a rough guide, something like a 55/35/10 split works well). Dedupe against yourself before inserting.

Insert in batches (a few hundred rows per `execute_sql` call is fine) with statements shaped like:

```sql
insert into words (word, pos, definition, tier) values
  ('abate', 'v', 'to lessen in intensity', 1),
  ('accolade', 'n', 'an expression of praise', 1),
  ...
on conflict (word) do nothing;
```

When done, update config with the real count:

```sql
update config set value = (select count(*)::text from words) where key = 'word_bank_size';
```

## Step 5 — Generate and publish Day 1

Now produce a real, complete first day of content — not a placeholder — and publish it as a new Claude Artifact.

**Select vocabulary for today:** pick ~16-17 words from the bank you just built, weighted toward tier 1 for approachability on day one, and weave them into the text completion, double-blank, and sentence equivalence items below (incidental use in the RC passages is fine but not required).

**Build the set**, grounded in the current (post-September 2023 "shortened") GRE General Test Verbal Reasoning format — not the old antonym/analogy format:
- 5 single-blank Text Completion items (5 options each, exactly 1 correct)
- 1 double-blank Text Completion item ((i) and (ii), 3 options per blank)
- 4 Sentence Equivalence items (6 options each, exactly 2 correct — a genuine synonym pair, no partial credit)
- 2 Reading Comprehension passages (~150-250 words, substantive and argumentatively dense, not just descriptive), each with 2-3 questions of varied type (main idea, inference, word-in-context, function-of-a-sentence, etc.), 5 options each

Every item needs a 1-2 sentence explanation of the correct answer, referencing the specific word or textual evidence. Before publishing, re-read every item: confirm each TC/SE has exactly one unambiguous best answer under standard dictionary definitions, SE pairs are genuine synonyms (not just the same part of speech), RC questions are answerable from the passage alone, and vocabulary register matches your tier 3 ceiling (nothing wildly more obscure).

**Publish the page.** Use the HTML in [`artifact-template.html`](./artifact-template.html) (next to this file) as the page's full source, verbatim except for the `DAY` object inside the `<script>` block near the top of the script, which you replace with today's real content in this exact shape:

```js
var DAY = { // Day 1 — <today's date, YYYY-MM-DD>
  tc: [ {prompt, options:[5 strings], correct: idx, explain}, ... 5 items ],
  tc2: { prompt, blanks:[ {options:[3 strings], correct:idx}, {options:[3 strings], correct:idx} ], explain },
  se: [ {prompt, options:[6 strings], correct:[i,j], explain}, ... 4 items ],
  rc: [
    { topic: "short topic label", passage: "...", questions: [ {question, options:[5 strings], correct:idx, explain}, ... 2-3 items ] },
    { topic: "...", passage: "...", questions: [ ... ] }
  ]
};
```

Don't restructure anything else in the page — the rest of the HTML/CSS/JS is a working, tested quiz app (start screen, stepper, immediate feedback, summary screen, streak tracking in `localStorage`) that the daily scheduled run will keep reusing by editing only this `DAY` object. Verify the script is syntactically valid before publishing.

Publish this and get back the artifact's URL. Call it `ARTIFACT_URL` for the rest of this skill.

## Step 6 — Write the artifact URL into config

```sql
update config set value = 'ARTIFACT_URL' where key = 'artifact_url';
```

(substitute the real URL — not the literal text `ARTIFACT_URL`)

## Step 7 — Log Day 1 like a normal run would

So the recycling and lookback logic in the daily job works correctly starting from day 2, close out today's "session" the same way the daily job will every day after:

```sql
update words set times_covered = times_covered + 1, last_covered_date = current_date, first_covered_date = coalesce(first_covered_date, current_date) where word in (/* the ~16 words you used above */);

insert into daily_sessions (session_date, day_number, passage_topics, vocab_words, total_questions, notes)
values (current_date, 1, array[/* your two passage topics */], array[/* the ~16 words you used */], 16, 'initial setup run');
```

## Step 8 — Create the daily scheduled task

Ask the person what time they want their set ready each morning (default: 6:00 AM in their local timezone if they don't have a preference). Create a daily scheduled task using whatever scheduling tool this session has, with:

- **Name**: something like "GRE Verbal Daily Set"
- **Schedule**: daily, at the time they gave you
- **Approval mode**: automatic / no approval required — nobody is there at 6 AM to click "approve," and the prompt below is written to never ask a question
- **Tools available to the task**: only the Supabase connector, scoped as in step 2 of the setup docs (`execute_sql` at minimum). It doesn't need any other connector.
- **Prompt**: the text in [`daily-prompt-template.md`](./daily-prompt-template.md) (next to this file), with `ARTIFACT_URL` and `PROJECT_ID` replaced with the real values from this session (not left as literal placeholder text), and the person's name filled in if they gave you one (otherwise just say "the user" and drop the name).

## Step 9 — Report back

Tell the person, concisely:
- The artifact link (the one thing they need to bookmark)
- How many words are in the bank and the tier breakdown
- The scheduled task's name and the time it'll run
- That everything about item counts and coverage strategy lives in `config`, editable anytime without touching the scheduled task
