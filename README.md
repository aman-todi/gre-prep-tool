# GRE Verbal Reps

A free daily GRE Verbal practice page that updates itself. Every morning a scheduled Claude routine writes a new 16-question set to a page you bookmark once. The set has 5 single-blank text completions, 1 double-blank, 4 sentence equivalence items and 2 reading passages, in the current GRE format. The routine also tracks which words from a vocabulary bank you've covered.

```
setup skill (once) ──► Supabase: words · daily_sessions · config
        │                        ▲ reads state / logs the day
        ▼                        │
  Artifact page ◄── rewritten ── daily routine
```

- **Supabase** holds three tables:
  - `words`: the bank, with tiers and how often each word has been covered
  - `daily_sessions`: one row per day, used so passage topics and words don't repeat
  - `config`: item counts, coverage goal and the page URL
- **The page** is one HTML file whose content lives in a `var DAY = {...}` object. Each item gets an explanation and a glossary of the hard answer-choice words. You can move freely between questions with Back/Next and the progress dots, and a review screen shows each item's status before you finish. Streaks are stored in the browser.
- **The routine** ([prompt](./skills/gre-prep-setup/daily-prompt-template.md)) reads Supabase and the page, writes and checks a new set, republishes the page to the same URL, then logs the day.

## Setup

See [docs/setup.md](./docs/setup.md). You create a free Supabase project, connect it to Claude and run the [`gre-prep-setup`](./skills/gre-prep-setup/SKILL.md) skill once.

## Customizing

- **Counts and coverage:** edit `config`.
- **Vocabulary:** edit `words`, or ask Claude to add words.
- **Look and feel:** edit the page's HTML and CSS. The routine only replaces `DAY`, so your changes survive.

RLS is off on purpose. Only Claude, connected through your Supabase connector, ever touches the database. Turn RLS on and add policies before you query the database from a browser.

[Changelog](./docs/changelog.md)
