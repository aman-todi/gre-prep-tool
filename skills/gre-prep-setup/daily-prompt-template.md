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
  - `config` table: key/value settings — read it each run (items_per_day, word_bank_size, new_words_min_per_day, passage_topic_lookback_days, rc_passages_per_day, questions_per_passage_min/max, target_format, coverage_goal, artifact_url, day_length_minutes).

## Step 1 — Read state
1. `select * from config` to get current parameters — do not hardcode numbers below, read them.
2. `select word, pos, definition, tier from words where times_covered = 0 order by tier asc, word asc` — this is your pool of brand-new words. If this pool is empty, the entire word bank has been exhausted at least once — in that case select from `words order by times_covered asc, tier asc, word asc` instead (fully recycle, prioritizing least-recently-covered and easier tiers first), and note this in `daily_sessions.notes`.
3. `select passage_topics from daily_sessions where session_date >= current_date - interval '7 days'` (or `config.passage_topic_lookback_days` if different) — flatten this into a list of recent topics to avoid repeating.
4. `select max(day_number) as last_day from daily_sessions` to compute today's day_number (= last_day + 1).

## Step 2 — Select vocabulary
- Pick `items_per_day` words total for today's content (read the current value each run).
- At least `new_words_min_per_day` must come from the never-covered pool (times_covered = 0), prioritizing tier 1 first, then tier 2, then tier 3, for approachability.
- Fill the remainder from previously-covered words, prioritizing the ones with the lowest times_covered (least recently reinforced), never repeating a word already used in the last 3 days if avoidable.
- These words should be woven into the text completion, double-blank, and sentence equivalence items (not necessarily the RC passages, though incidental use there is fine).

## Step 3 — Generate today's content
Ground everything in the CURRENT (post-September 2023 "shortened") GRE General Test Verbal Reasoning format — NOT the old antonym/analogy format. Build a clever, well-balanced mix simulating both GRE Verbal section 1 and section 2 style and difficulty. Target structure (confirm exact counts against `config.items_per_day` and `config.rc_passages_per_day`):
- 5 single-blank Text Completion items (5 options each, exactly 1 correct)
- 1 double-blank Text Completion item ((i) and (ii), 3 options per blank)
- 4 Sentence Equivalence items (6 options each, exactly 2 correct — synonymous pair, no partial credit)
- `rc_passages_per_day` Reading Comprehension passages, each with `questions_per_passage_min`-`questions_per_passage_max` associated questions (question types should vary: main idea, inference, word-in-context, function-of-a-sentence, etc.), 5 options each

Passage topics must be genuinely different from anything in the lookback list from Step 1 (varied domains: science, social science, history, arts, current-ish general-interest nonfiction — avoid anything requiring post-cutoff current events). Each passage should be substantive (GRE-length, ~150-250 words), argumentatively dense, not just descriptive.

Every item needs a correct-answer explanation (1-2 sentences, referencing the specific word/textual evidence — this is shown to the user immediately after answering).

**Self-check before publishing:** re-read every item. Confirm: each TC/SE has exactly one unambiguous best answer given standard dictionary definitions; SE pairs are genuine synonyms (not just same part of speech); RC questions are answerable from the passage alone; no factual errors; vocabulary matches the GRE register (no words simpler than the word bank's tier 3 or wildly obscure ones outside it).

## Step 4 — Update the artifact
1. Read the current artifact via the Artifact tool (action "read", url above) to get the live HTML.
2. Replace the `var DAY = { ... }` object in the `<script>` block with today's content, following this exact shape (unchanged from the current version — do not restructure the JS, only replace DAY's contents):
```js
var DAY = { // Day <N> — <YYYY-MM-DD>
  tc: [ {prompt, options:[5 strings], correct: idx, explain}, ... 5 items ],
  tc2: { prompt, blanks:[ {options:[3 strings], correct:idx}, {options:[3 strings], correct:idx} ], explain },
  se: [ {prompt, options:[6 strings], correct:[i,j], explain}, ... 4 items ],
  rc: [
    { topic: "short topic label", passage: "...", questions: [ {question, options:[5 strings], correct:idx, explain}, ... items ] },
    ...
  ]
};
```
3. Update the comment on the `var DAY = {` line to the correct day number and date.
4. Update the footer text and the start-screen `.breakdown` chips if the exact item counts changed from the previous version (keep them accurate to what's actually in DAY).
5. Before publishing: verify the file is valid JS (e.g. extract the `<script>` contents and run them through `new Function(...)` in Node to catch syntax errors) and that `buildItems()`'s flattening logic still matches the `rc` shape above.
6. Publish via the Artifact tool, action "publish", passing the same `url` as above, so it updates in place (same link the user already has).

## Step 5 — Update Supabase (do this atomically / carefully, after the artifact publish succeeds)
1. For each word used today: `update words set times_covered = times_covered + 1, last_covered_date = current_date, first_covered_date = coalesce(first_covered_date, current_date) where word = '...'`.
2. Insert today's log row:
```sql
insert into daily_sessions (session_date, day_number, passage_topics, vocab_words, total_questions, notes)
values (current_date, <N>, array['topic1','topic2'], array['word1','word2',...], <total>, '<any notes, e.g. "word bank fully exhausted, began recycling" if applicable>');
```

## Notes / constraints
- No fallback content: if generation fails partway, it's fine to leave the previous day's artifact live rather than publish something broken — do not publish a half-finished DAY object. Prefer to retry generation once before giving up.
- Do not create a new artifact — always update the existing one at the URL above via publish with `url` set.
- This is a fully autonomous run — do not ask the user any questions; use your best judgment per the rules above.
```
