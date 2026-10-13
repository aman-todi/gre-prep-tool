---
name: gre-prep-setup
description: One-time bootstrap for a self-updating daily GRE Verbal practice tool built on Supabase and a Claude Artifact. Creates the schema, generates a vocabulary bank, publishes the practice page with a real Day 1 and creates the daily routine that regenerates it. Use when asked to set up, install or replicate the GRE prep tool.
---

# GRE Verbal Reps: one-time setup

Build the whole system in this session, then hand off to a daily routine. The system is a Supabase database (vocabulary bank, session log, config) plus one HTML page (a Claude Artifact) that the routine rewrites in place every morning.

## Prerequisites

Before you start, confirm all three:
- The Supabase connector is reachable. Call `list_projects`.
- You can publish an Artifact.
- You can create a scheduled routine.

If one is missing, say so and stop. Other connectors don't matter. Never ask for keys or passwords.

## 1. Pick the project

List the Supabase projects. If there's only one, use it and confirm its name. If there are several, ask which one. Call its ref `PROJECT_ID` from here on.

## 2. Schema

```sql
create table if not exists words (
  id                 bigint generated always as identity primary key,
  word               text not null unique,
  pos                text not null,                                -- n / v / adj / adv
  definition         text not null,
  tier               smallint not null check (tier in (1, 2, 3)),  -- 1 core, 2 common, 3 advanced
  times_covered      integer not null default 0,
  first_covered_date date,
  last_covered_date  date
);

create table if not exists daily_sessions (
  id              bigint generated always as identity primary key,
  session_date    date not null unique,
  day_number      integer not null,
  vocab_words     text[] not null default '{}',
  passage_topics  text[] not null default '{}',
  total_questions integer,
  notes           text,
  created_at      timestamptz not null default now()
);

create table if not exists config (
  key   text primary key,
  value text not null
);
```

RLS stays off on purpose. Only Claude touches the database, through MCP, and the page has no Supabase client. Mention this once and don't enable it.

## 3. Config

```sql
insert into config (key, value) values
  ('artifact_url', 'PENDING'),
  ('coverage_goal', 'Work toward ~95% coverage of the words table over time, prioritizing tier 1 (essential) words first, then tier 2, then tier 3'),
  ('day_length_minutes', '15-25'),
  ('items_per_day', '16'),
  ('new_words_min_per_day', '10'),
  ('questions_per_passage_max', '3'),
  ('questions_per_passage_min', '1'),
  ('rc_passages_per_day', '2'),
  ('target_format', 'Current (post-Sept 2023 shortened) GRE General Test, Verbal Reasoning section'),
  ('word_bank_size', 'PENDING')
on conflict (key) do nothing;
```

These are defaults, so don't ask the person about them.

## 4. Vocabulary bank

Generate at least 500-600 GRE-register words from your own knowledge, more if you can. Split them roughly 55% tier 1, 35% tier 2 and 10% tier 3. Leave out any word whose definition or part of speech you're unsure of. Insert a few hundred rows per call:

```sql
insert into words (word, pos, definition, tier) values
  ('abate', 'v', 'to lessen in intensity', 1), ...
on conflict (word) do nothing;

update config set value = (select count(*)::text from words) where key = 'word_bank_size';
```

## 5. Day 1

Write a real first set, not a placeholder. Follow step 3 of [`daily-prompt-template.md`](./daily-prompt-template.md): the item mix, glossaries, explanations, randomized answer positions and the self-check. Lean on tier 1 words for Day 1.

Publish [`artifact-template.html`](./artifact-template.html) as a new Artifact. Change only its `DAY` object, using the shape in step 4 of the prompt template, and check that the script parses before publishing. Call the resulting URL `ARTIFACT_URL`.

## 6. Record and log Day 1

```sql
update config set value = '<ARTIFACT_URL>' where key = 'artifact_url';

update words set times_covered = times_covered + 1, last_covered_date = current_date,
  first_covered_date = coalesce(first_covered_date, current_date)
where word in (<the words used>);

insert into daily_sessions (session_date, day_number, passage_topics, vocab_words, total_questions, notes)
values (current_date, 1, array[<topics>], array[<words>], <question count>, 'initial setup run');
```

## 7. Ask for a time

Ask what time the set should be ready each day. Default to 6:00 AM in the person's timezone.

## 8. Create the routine

- Name: "GRE Verbal Daily Set"
- Schedule: daily at the chosen time
- Approval: automatic. Nobody is around to approve it, and the prompt never asks questions.
- Connectors: Supabase must be attached. Any others are irrelevant.
- Prompt: [`daily-prompt-template.md`](./daily-prompt-template.md), with `<NAME>`, `ARTIFACT_URL` and `PROJECT_ID` filled in

## 9. Report back

Tell the person:
- the artifact link to bookmark
- the word count by tier
- the routine's name and time
- that `config` holds every setting and can be edited any time
