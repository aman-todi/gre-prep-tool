# Daily routine prompt

The prompt the scheduled routine runs each morning (setup step 8). Replace `<NAME>`, `ARTIFACT_URL` and `PROJECT_ID` before installing it. Use "the user" for `<NAME>` if the person didn't give a name. Keep this file and the live routine identical apart from those values.

```
Generate today's GRE Verbal practice set for <NAME>. This is an autonomous daily run with no memory of earlier runs. Never ask questions: everything you need is below, in Supabase or in the artifact.

## System
- Artifact (the page <NAME> opens each morning): ARTIFACT_URL
- Supabase project_id `PROJECT_ID`:
  - `words`: word, pos, definition, tier (1 core, 2 common, 3 advanced), times_covered, first_covered_date, last_covered_date
  - `daily_sessions`: session_date (unique), day_number, passage_topics text[], vocab_words text[], total_questions, notes, created_at
  - `config`: key/value settings. Read them every run instead of assuming numbers.

## 1. Read state
select * from config;
select 1 from daily_sessions where session_date = current_date;
select coalesce(max(day_number), 0) + 1 as day_number from daily_sessions;
select session_date, passage_topics from daily_sessions order by session_date;
select word, pos, definition, tier from words where times_covered = 0 order by tier, word;

- If today already has a row, the job has already run today: stop. Only update that row in place if you're deliberately regenerating today's set. Never log a set under any date other than current_date.
- If no word has times_covered = 0, recycle from `words` ordered by times_covered, tier, word, and say so in `notes`.

## 2. Pick vocabulary
Pick `items_per_day` words. At least `new_words_min_per_day` come from the never-covered pool (tier 1 first, then 2, then 3). Fill the rest with the least-covered words already used, avoiding any word from the last 3 days. Use them in the tc, tc2 and se items.

## 3. Write the set
Use the current (post-September 2023) GRE Verbal format, mixing section 1 and section 2 difficulty:
- tc: 5 single-blank text completions with 5 options and 1 correct answer each
- tc2: 1 double-blank text completion with 3 options per blank
- se: 4 sentence equivalence items with 6 options each, exactly 2 correct, and the 2 correct options a true synonym pair
- rc: `rc_passages_per_day` passages (150-250 words, argumentative rather than descriptive), each with 2-3 questions of varied type (main idea, inference, word in context, function of a sentence) and 5 options per question

Rules:
- Topics: each passage topic must differ in substance from every topic in the full history. A rephrasing of the same subject counts as a repeat. Vary the domains, and avoid anything that depends on recent news.
- Explanations: give every item a 1-2 sentence explanation that cites the deciding word or text.
- Glossary: give every tc, tc2 and se item `glossary: [{word, meaning, synonym}]` covering each uncommon word among its options, correct answers and distractors alike. Skip everyday words. `word` matches the option text exactly. `meaning` is 3-8 plain words. `synonym` is one simple word. List entries in the final option order. tc2 has one glossary that covers blank (i) and then blank (ii).
- Answer positions: place every correct answer by a real random draw, not by habit. Vary the positions within the day and across days, including the se pair and the first question of each passage.

Before publishing, check that:
- every tc and se item has exactly one defensible answer
- each se pair is a true synonym pair
- every rc question can be answered from its passage alone
- there are no factual errors
- every glossary is accurate and complete
- both topics are new

Then list every correct-answer position and re-shuffle any items that cluster.

## 4. Publish
1. Read the artifact with the Artifact tool (action "read", the url above).
2. Replace only the `var DAY = { ... }` object. Keep this shape, and put today's day number and current_date in the comment:
   var DAY = { // Day <N> — <YYYY-MM-DD>
     tc:  [ {prompt, options:[5], correct:i, explain, glossary:[...]} ×5 ],
     tc2: { prompt, blanks:[ {options:[3], correct:i}, {options:[3], correct:i} ], explain, glossary:[...] },
     se:  [ {prompt, options:[6], correct:[i,j], explain, glossary:[...]} ×4 ],
     rc:  [ {topic, passage, questions:[ {question, options:[5], correct:i, explain} ... ]} ... ]
   };
   Mark blanks in tc and se prompts with ___. Mark them in the tc2 prompt with (i)___ and (ii)___.
3. If the item counts change, update the footer and the start-screen `.breakdown` chips to match. Change nothing else on the page. Navigation, the review screen and rendering live outside `DAY`.
4. Check that the `<script>` parses (run it through `new Function(...)` in Node). Then publish with action "publish" and the same `url`. Never create a new artifact.
If generation fails, retry once. If it fails again, leave yesterday's page live rather than publish a partial `DAY`.

## 5. Log (only after the publish succeeds)
update words set times_covered = times_covered + 1, last_covered_date = current_date,
  first_covered_date = coalesce(first_covered_date, current_date)
where word in (<today's words>);
insert into daily_sessions (session_date, day_number, passage_topics, vocab_words, total_questions, notes)
values (current_date, <N>, array[<topics>], array[<words>], <question count>, <notes or null>);
```
