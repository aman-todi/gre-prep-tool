# Daily scheduled-task prompt (template)

Used by [`SKILL.md`](./SKILL.md) step 8. This is what the scheduled task actually runs every morning, forever, once setup is done. It fires in a brand-new session each time with no memory of any prior run — that's why it names the artifact URL and project id explicitly rather than assuming context, restates the schema, and reads `config` instead of trusting hardcoded numbers.

Before installing it as the scheduled task's prompt, replace `ARTIFACT_URL` and `PROJECT_ID` with the real values from setup, and `<NAME>` with the person's name (or drop the personalization if they didn't give one).

---

```
You are generating today's GRE Verbal practice set for <NAME>, as a fully autonomous daily job. You have no memory of any prior conversation — everything you need is below or in the Supabase project.

## System overview
- Published artifact (the page the user opens each morning): ARTIFACT_URL
- Supabase project, project_id `PROJECT_ID`, holds all state:
  - `words` table: word, pos, definition, tier (1=core/most frequent, 2=common, 3=advanced/rarer), times_covered, first_covered_date, last_covered_date.
  - `daily_sessions` table: session_date (unique), day_number, passage_topics (text[]), vocab_words (text[]), total_questions, notes, created_at.
  - `config` table: key/value settings — read it each run (items_per_day, word_bank_size, new_words_min_per_day, rc_passages_per_day, questions_per_passage_min/max, target_format, coverage_goal, artifact_url, day_length_minutes).

## IMPORTANT — date handling
`session_date` must always be `current_date` (today's actual date per the database's clock) — never "tomorrow," never an offset, never adjusted because some other date looks "taken." An earlier version of this job, on finding today's date already logged, logged content under tomorrow's date instead, and every later run inherited the shift. Do not recreate that:
- Run `select current_date` and use that exact value for `session_date`.
- Before inserting, run `select 1 from daily_sessions where session_date = current_date`. If a row exists (the job already ran today, e.g. a duplicate or manual fire), do NOT insert under a different date. Either skip the run entirely (safest) or, with a clear reason to regenerate, `UPDATE` that row in place.
- `day_number` for a new row is `coalesce((select max(day_number) from daily_sessions), 0) + 1` — never derived from the date.

## IMPORTANT — passage-topic repetition
Never de-duplicate passage topics with a rolling date window; a 7-day window let topics resurface 8-10 days later under new wording (octopus cognition, the Antikythera mechanism, "junk" DNA). Always:
- Pull the **entire** passage-topic history (every session ever logged, no date filter).
- Check candidates against it for **near-duplicates and rephrasings of the same underlying subject**, not just exact strings. Two topics are the same if they would make essentially the same argument or center on the same subject, whatever the title. If a candidate overlaps substantially with anything used before, pick a different one.
- The history is short enough to read in full every run.

## Step 1 — Read state
1. `select * from config` to get current parameters — do not hardcode numbers below, read them.
2. `select word, pos, definition, tier from words where times_covered = 0 order by tier asc, word asc` — your pool of brand-new words. If empty, the whole bank has been used at least once — select from `words order by times_covered asc, tier asc, word asc` instead (recycle, least-covered and easier tiers first) and note this in `daily_sessions.notes`.
3. `select session_date, passage_topics from daily_sessions order by session_date asc` — the full history; flatten it into a permanent no-repeat list of topics.
4. `select max(day_number) as last_day from daily_sessions` — today's day_number is last_day + 1.
5. `select 1 from daily_sessions where session_date = current_date` — see the date-handling section.

## Step 2 — Select vocabulary
- Pick `items_per_day` words total for today's content.
- At least `new_words_min_per_day` must come from the never-covered pool, prioritizing tier 1, then tier 2, then tier 3.
- Fill the remainder from previously-covered words with the lowest times_covered, avoiding any word used in the last 3 days where possible.
- Weave these into the text completion, double-blank, and sentence equivalence items (incidental use in RC passages is fine).

## Step 3 — Generate today's content
Ground everything in the CURRENT (post-September 2023 "shortened") GRE General Test Verbal Reasoning format — NOT the old antonym/analogy format. Target structure (confirm against `config.items_per_day` and `config.rc_passages_per_day`):
- 5 single-blank Text Completion items (5 options each, exactly 1 correct)
- 1 double-blank Text Completion item ((i) and (ii), 3 options per blank)
- 4 Sentence Equivalence items (6 options each, exactly 2 correct — a synonymous pair, no partial credit)
- `rc_passages_per_day` Reading Comprehension passages, each with `questions_per_passage_min`-`questions_per_passage_max` questions of varied type (main idea, inference, word-in-context, function-of-a-sentence, etc.), 5 options each

Passage topics must differ in substance, not just phrasing, from the full history (varied domains: science, social science, history, arts, general-interest nonfiction — nothing needing post-cutoff current events). Each passage ~150-250 words, argumentatively dense, not just descriptive.

Every item needs a 1-2 sentence correct-answer explanation referencing the specific word or textual evidence.

**Word glossary:** every single-blank TC, double-blank TC, and SE item (not RC) must carry a `glossary` array covering each *uncommon* word among its options — correct answers and advanced distractors alike; skip only everyday words. Each entry: `word` (exactly as in the options), `meaning` (3-8 word plain-English gloss), `synonym` (one simple everyday synonym), in the order the words appear in the options. The double-blank item has one `glossary` covering both blanks' options, blank (i) first.

**Self-check before publishing:** re-read every item. Confirm each TC/SE has exactly one unambiguous best answer; SE pairs are genuine synonyms; RC questions are answerable from the passage alone; no factual errors; vocabulary matches the word bank's GRE register; every glossary entry is correct, concise, and covers every advanced option; and the RC topics are not near-duplicates of anything in the full history.

## Step 4 — Update the artifact
1. Read the current artifact via the Artifact tool (action "read", url above) to get the live HTML.
2. Replace the `var DAY = { ... }` object in the `<script>` block with today's content, in exactly this shape — do not restructure the JS, only replace DAY's contents:
```js
var DAY = { // Day <N> — <YYYY-MM-DD, the same current_date used for session_date>
  tc: [ {prompt, options:[5 strings], correct: idx, explain, glossary:[{word,meaning,synonym}, ...]}, ... 5 items ],
  tc2: { prompt, blanks:[ {options:[3 strings], correct:idx}, {options:[3 strings], correct:idx} ], explain, glossary:[{word,meaning,synonym}, ...] },
  se: [ {prompt, options:[6 strings], correct:[i,j], explain, glossary:[{word,meaning,synonym}, ...]}, ... 4 items ],
  rc: [
    { topic: "short topic label", passage: "...", questions: [ {question, options:[5 strings], correct:idx, explain}, ... ] },
    ...
  ]
};
```
3. Update the comment on the `var DAY = {` line to the correct day number and date — the date MUST match the `current_date` used for `session_date` in Step 5.
4. Update the footer text and the start-screen `.breakdown` chips if the item counts changed (keep them accurate to what's in DAY).
5. Before publishing: verify the script is valid JS (extract the `<script>` contents and run them through `new Function(...)` in Node), that `buildItems()`'s flattening still matches the `rc` shape, and that `glossaryHTML()` and the `.glossary`/`.gloss-row` CSS are still present and unmodified — this task only ever edits `DAY`.
6. Publish via the Artifact tool, action "publish", passing the same `url`, so it updates in place.

## Step 5 — Update Supabase (after the artifact publish succeeds)
1. For each word used today: `update words set times_covered = times_covered + 1, last_covered_date = current_date, first_covered_date = coalesce(first_covered_date, current_date) where word = '...'`.
2. Insert today's log row — `session_date` is the literal `current_date`:
```sql
insert into daily_sessions (session_date, day_number, passage_topics, vocab_words, total_questions, notes)
values (current_date, <N>, array['topic1','topic2'], array['word1','word2',...], <total>, '<any notes, e.g. "word bank fully exhausted, began recycling" if applicable>');
```

## Notes / constraints
- No fallback content: if generation fails partway, leave the previous day's artifact live rather than publish something broken. Retry generation once before giving up.
- Do not create a new artifact — always update the existing one at the URL above.
- This is a fully autonomous run — do not ask the user any questions; use your best judgment per the rules above.
```
