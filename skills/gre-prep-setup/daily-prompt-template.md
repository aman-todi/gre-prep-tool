# Daily scheduled-task prompt (template)

Used by [`SKILL.md`](./SKILL.md) step 8. This is what the scheduled task actually runs every morning, forever, once setup is done. It fires in a brand-new session each time with no memory of any prior run — that's why it names the artifact URL and project id explicitly rather than assuming context, restates the schema, and reads `config` instead of trusting hardcoded numbers.

Before installing it as the scheduled task's prompt, replace `ARTIFACT_URL`, `PROJECT_NAME`, `PROJECT_ID` and `WORD_BANK_SIZE` with the real values from setup, and `<NAME>` with the person's name (or "the user" if they didn't give one).

The body below is kept in sync with the live scheduled task word for word, apart from those placeholders. The two bug notes are deliberate: they record real failures of earlier versions, so a fresh run doesn't reinvent them. When the live prompt changes, copy it back here.

---

```
You are generating today's GRE Verbal practice set for <NAME>, as a fully autonomous daily job. You have no memory of any prior conversation — everything you need is below or in the Supabase project.

## System overview
- Published artifact (the page <NAME> opens each morning): ARTIFACT_URL
- Supabase project "PROJECT_NAME", project_id `PROJECT_ID`, holds all state:
  - `words` table: word, pos, definition, tier (1=core/most frequent, 2=common, 3=advanced/rarer), times_covered, first_covered_date, last_covered_date. Currently ~WORD_BANK_SIZE words.
  - `daily_sessions` table: session_date (unique), day_number, passage_topics (text[]), vocab_words (text[]), total_questions, notes, created_at.
  - `config` table: key/value settings — read it each run (items_per_day, word_bank_size, new_words_min_per_day, passage_topic_lookback_days, rc_passages_per_day, questions_per_passage_min/max, target_format, coverage_goal, artifact_url, day_length_minutes).

## IMPORTANT — date handling (a past bug here, now fixed; do not reintroduce it)
`session_date` must always be set to `current_date` (today's actual date, per the database's clock) — never "tomorrow," never an offset, never adjusted because some other date looks "taken." A previous version of this job once invented a workaround where, on finding today's date already logged, it started logging content under tomorrow's date instead — and because every subsequent run inherited that shift, the logged date drifted one full day ahead of reality forever after. That bug has been fixed and the historical data corrected. Do not recreate it:
- Always compute `select current_date` and use that exact value for `session_date` in your insert.
- Before inserting, run `select 1 from daily_sessions where session_date = current_date`. If a row already exists for today (meaning this job already ran today — e.g. a duplicate/manual fire), do NOT create a second row under a different date. Instead, either skip the run entirely (safest — leave the existing artifact and log alone) or, if you have a clear reason to regenerate today's content, `UPDATE` that existing row in place rather than inserting a new one under a different date.
- `day_number` for a new row is `coalesce((select max(day_number) from daily_sessions), 0) + 1` — never derived from the date.

## IMPORTANT — passage-topic repetition bug (found 2026-10-03, now fixed; do not reintroduce it)
An earlier version of this job de-duplicated RC passage topics by checking only a 7-day rolling window (`passage_topic_lookback_days` was 7, and Step 1 used a literal `current_date - interval '7 days'` filter). That window was too short relative to how fast a 2-topics/day job cycles through "good debate-style nonfiction topics," so several topics quietly resurfaced 8-10 days later under slightly different wording: "Distributed cognition in octopuses" (Day 1) resurfaced as "Octopus cognition and the question of distributed intelligence" (Day 10); the Antikythera mechanism repeated Day 6 → Day 14; "the debate over 'junk' DNA" repeated Day 5 → Day 15. The `passage_topic_lookback_days` config value has been bumped to a large placeholder (36500) precisely so it can no longer act as a short window, but the real fix is behavioral, not numeric — follow Step 1 item 3 below exactly:
- Always pull the **entire** passage-topic history (every session ever logged, no date filter) — do not filter by `passage_topic_lookback_days` or any other day count when deciding whether a topic is fresh.
- Check new candidate topics against that full list for **near-duplicates and rephrasings of the same underlying subject**, not just exact string matches. Two topics count as the same if they'd make essentially the same argument or center on the same subject, even with a different title (e.g. "octopus cognition" vs. "distributed intelligence in octopuses" vs. "what octopus brains tell us about intelligence" are all one topic). If a candidate overlaps substantially with anything ever used before, pick a different one.
- The history is still short (well under 100 rows as of this writing), so reading and eyeballing all of it every run is cheap — there's no excuse to window it.

## Step 1 — Read state
1. `select * from config` to get current parameters (do not hardcode numbers below — read them, but they are currently: 16 items/day, ~10+ of those must use never-before-covered words, 2 RC passages/day with 1-3 questions each). `passage_topic_lookback_days` is a legacy key now set to a large placeholder number and should NOT be used as a rolling-window day count for topic de-duplication — see the bug note above and item 3 below, which always check full history regardless of this value.
2. `select word, pos, definition, tier from words where times_covered = 0 order by tier asc, word asc` — this is your pool of brand-new words. If this pool is empty, the entire word bank has been exhausted at least once — in that case select from `words order by times_covered asc, tier asc, word asc` instead (fully recycle, prioritizing least-recently-covered and easier tiers first), and note this in `daily_sessions.notes`.
3. `select session_date, passage_topics from daily_sessions order by session_date asc` — pull the full history (every session ever logged, no date filter). Flatten all of it into one list of past topics and treat it as a permanent no-repeat list; see the bug note above on checking for near-duplicates/rephrasings, not just exact string matches.
4. `select max(day_number) as last_day from daily_sessions` to compute today's day_number (= last_day + 1).
5. `select 1 from daily_sessions where session_date = current_date` — see the date-handling section above; if this returns a row, do not blindly insert a duplicate for a different date.

## Step 2 — Select vocabulary
- Pick 16-17 words total for today's content (match `items_per_day` from config).
- At least `new_words_min_per_day` (currently 10) must come from the never-covered pool (times_covered = 0), prioritizing tier 1 first, then tier 2, then tier 3, for approachability.
- Fill the remainder from previously-covered words, prioritizing the ones with the lowest times_covered (least recently reinforced), never repeating a word already used in the last 3 days if avoidable.
- These words should be woven into the text completion, double-blank, and sentence equivalence items (not necessarily the RC passages, though incidental use there is fine).

## Step 3 — Generate today's content
Ground everything in the CURRENT (post-September 2023 "shortened") GRE General Test Verbal Reasoning format — NOT the old antonym/analogy format. Build a clever, well-balanced mix simulating both GRE Verbal section 1 and section 2 style and difficulty. Target structure (confirm exact counts against `config.items_per_day`, currently 16 total):
- 5 single-blank Text Completion items (5 options each, exactly 1 correct)
- 1 double-blank Text Completion item ((i) and (ii), 3 options per blank)
- 4 Sentence Equivalence items (6 options each, exactly 2 correct — synonymous pair, no partial credit)
- 2 Reading Comprehension passages, each with 2-3 associated questions (question types should vary: main idea, inference, word-in-context, function-of-a-sentence, etc.), 5 options each

Passage topics must be genuinely different — in substance, not just phrasing — from anything in the full passage-topic history gathered in Step 1 (varied domains: science, social science, history, arts, current-ish general-interest nonfiction — avoid anything requiring post-cutoff current events). Each passage should be substantive (GRE-length, ~150-250 words), argumentatively dense, not just descriptive.

Every item needs a correct-answer explanation (1-2 sentences, referencing the specific word/textual evidence — this is shown to <NAME> immediately after answering).

**Word glossary:** every Text Completion (single-blank), double-blank Text Completion, and Sentence Equivalence item — NOT the Reading Comprehension items, whose options are full phrases rather than single words — must also carry a `glossary` array covering each *uncommon* word among that item's options (this includes the correct answer(s) as well as any advanced-vocabulary distractors; skip only genuinely everyday words like "friendly," "happy," or "indifferent" that don't need defining). For each such word give:
- `word`: the option text exactly as it appears in `options`/blank `options`.
- `meaning`: a brief (3-8 word) plain-English gloss.
- `synonym`: one simple, everyday synonym.
Order entries to match the order the words appear in the options list. A single-blank TC item typically glosses 3-5 of its 5 options; an SE item typically glosses 4-6 of its 6 options; the double-blank TC2 item has one `glossary` array (not one per blank) covering uncommon words from both blanks' option lists combined, in blank-then-option order.

**Self-check before publishing:** re-read every item. Confirm: each TC/SE has exactly one unambiguous best answer given standard dictionary definitions; SE pairs are genuine synonyms (not just same part of speech); RC questions are answerable from the passage alone; no factual errors; vocabulary matches the GRE register (no words simpler than the word bank's tier 3 or wildly obscure ones outside it); every TC/TC2/SE item's `glossary` entries have correct, concise meanings and genuinely simple synonyms, and every advanced word among that item's options is covered; and the two RC topics are not near-duplicates of anything in the full history pulled in Step 1 item 3.

## Step 4 — Update the artifact
1. Read the current artifact via the Artifact tool (action "read", url above) to get the live HTML.
2. Replace the `var DAY = { ... }` object in the `<script>` block with today's content, following this exact shape (unchanged from the current version — do not restructure the JS, only replace DAY's contents). Note the `glossary` field on tc/tc2/se items — the artifact's rendering code already knows how to display it in the explanation box, so just supply the data in this shape:
```js
var DAY = { // Day <N> — <YYYY-MM-DD, the same current_date used for session_date>
  tc: [ {prompt, options:[5 strings], correct: idx, explain, glossary:[{word,meaning,synonym}, ...]}, ... 5 items ],
  tc2: { prompt, blanks:[ {options:[3 strings], correct:idx}, {options:[3 strings], correct:idx} ], explain, glossary:[{word,meaning,synonym}, ...] },
  se: [ {prompt, options:[6 strings], correct:[i,j], explain, glossary:[{word,meaning,synonym}, ...]}, ... 4 items ],
  rc: [
    { topic: "short topic label", passage: "...", questions: [ {question, options:[5 strings], correct:idx, explain}, ... 2-3 items ] },
    { topic: "...", passage: "...", questions: [ ... ] }
  ]
};
```
3. Update the comment on the `var DAY = {` line to the correct day number and date — the date here MUST match the `current_date` value you use for `session_date` in Step 5 (this is exactly the invariant that broke before; keep them in lockstep).
4. Update the footer text and the start-screen `.breakdown` chips if the exact item counts changed from the previous version (keep them accurate to what's actually in DAY).
5. Before publishing: verify the file is valid JS (e.g. extract the `<script>` contents and run them through `new Function(...)` in Node to catch syntax errors) and that `buildItems()`'s flattening logic still matches the `rc` shape above. Also spot-check that `glossaryHTML()` (or equivalent rendering helper already in the page) is still present and unmodified — if a prior edit ever changed the DOM/CSS around `.glossary`/`.gloss-row`, leave it as-is; this task only ever edits `DAY`.
6. Publish via the Artifact tool, action "publish", passing the same `url` as above, so it updates in place (same link <NAME> already has).

## Step 5 — Update Supabase (do this atomically / carefully, after the artifact publish succeeds)
1. For each of the ~16-17 words used today: `update words set times_covered = times_covered + 1, last_covered_date = current_date, first_covered_date = coalesce(first_covered_date, current_date) where word = '...'`.
2. Insert today's log row — `session_date` here MUST be the literal `current_date` (see the date-handling section above — no shifting, no offsets):
```sql
insert into daily_sessions (session_date, day_number, passage_topics, vocab_words, total_questions, notes)
values (current_date, <N>, array['topic1','topic2'], array['word1','word2',...], 16, '<any notes, e.g. "word bank fully exhausted, began recycling" if applicable>');
```

## Notes / constraints
- No fallback content: if generation fails partway, it's fine to leave the previous day's artifact live rather than publish something broken — do not publish a half-finished DAY object. Prefer to retry generation once before giving up.
- Do not create a new artifact — always update the existing one at the URL above via publish with `url` set.
- This is a fully autonomous run — do not ask the user any questions; use your best judgment per the rules above.
```
