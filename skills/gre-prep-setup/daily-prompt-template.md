# Daily routine prompt

The prompt the scheduled routine runs each morning (setup step 7). Replace `<NAME>`, `ARTIFACT_URL` and `PROJECT_ID` before installing it. Use "the user" for `<NAME>` if the person didn't give a name. Keep this file and the live routine identical apart from those values.

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
- rc: `rc_passages_per_day` passages (150-250 words, argumentative rather than descriptive), each with EXACTLY 3 questions — never 2, and this applies to the second passage exactly as much as the first (config `questions_per_passage_min` and `questions_per_passage_max` are both 3) — of varied type, with 5 options per question. Each passage's 3 questions must include one global question (main idea, primary purpose, or overall structure/organization) and two drawn from: inference, author's attitude/tone, function of a specific sentence or phrase, word in context, and strengthen/weaken or assumption (when the passage makes an argument). Don't use the same three-type combination for both passages.

Rules:
- Topics: each passage topic must differ in substance from every topic in the full history. A rephrasing of the same subject counts as a repeat. Vary the domains, and avoid anything that depends on recent news.
- Distractor difficulty (match the real exam): ETS distractors are rarely eliminable by vibe alone — they reuse the passage's or sentence's own vocabulary and fail for a specific, checkable reason, not because they sound off-topic or extreme. For rc main-idea/primary-purpose and inference/function questions, build the 4 wrong options using real GRE trap types — mix at least three of: too narrow (true of only part of the passage), too broad (extends past what the passage actually supports), distorted (reverses or misstates the passage's point, emphasis, or tone), and unsupported/out-of-scope (adds specifics, causation, or comparisons the passage never makes). At most one wrong option per question may rely on extreme/absolute language ("conclusively," "entirely," "all," "never," "unanimously") as its main flaw — the rest must fail for scope, support, or distortion reasons that require rereading the passage, not surface tone. For tc and tc2, make at least two of the four wrong options per blank the same part of speech and emotional register as the correct word — plausible in isolation, wrong only given the sentence's specific clue — rather than flatly opposite-valence or unrelated words. A real GRE text completion deliberately mixes easier wrong options (ruled out by grammar, register, or basic sense) with one or two genuine near-miss options that require the clue to decide — but the correct answer must remain the single most eligible fit: no wrong option may be a true synonym of the correct word (that manufactures a second correct answer), and no near-miss distractor may be left standing on a different semantic axis than the one the clue actually tests. Concretely: before finalizing each tc/tc2 blank, name in one phrase the specific axis the clue disambiguates (e.g. harshness of tone vs. intensity of conviction vs. sarcasm of content; degree vs. kind; cause vs. effect), and confirm every near-miss distractor fails specifically on that named axis — not merely on some other axis the sentence never addresses. If a near-miss distractor would also satisfy the sentence under a plausible alternate reading the clue doesn't foreclose, either sharpen the clue so it explicitly forecloses that reading, or swap the distractor out. For se, beyond the correct synonym pair, include one additional pair of options that are synonyms of each other but do not actually complete the sentence correctly (the classic SE trap pair), so the pair can't be spotted by synonym-matching alone.
- RC difficulty (apply all of the following on top of the trap types above):
  - Passage build: hard questions need a passage with layers. Every passage must contain at least two distinguishable positions (e.g. a received view and the author's response, or two scholars' competing readings), at least one concession or qualification (the author grants part of an opposing point, or limits their own claim), and an author's stance that is nuanced rather than simply for/against (e.g. "right about X but overstates Y"). Use contrast and pivot words (however, yet, to be sure, while, nonetheless) so that who-believes-what has to be tracked. A one-position passage cannot support hard questions.
  - Near-miss distractor: every rc question must have at least one "almost right" wrong option that a careful reader would seriously consider and that is killed only by one specific word or phrase in the passage. Good near-misses: a view the passage mentions but attributes to someone other than the author, presented as the author's view; a statement true of the passage that does not answer the question actually asked (right answer to a different question); the right idea with the wrong degree or wrong emphasis (e.g. "rejects" where the author only "qualifies"); and, for global questions, an accurate summary of only the first or last paragraph's point.
  - Wording: wrong options should reuse the passage's distinctive terms and phrases; the correct answer should usually paraphrase the passage in different words, as ETS does. Never make the correct answer the only option that echoes the passage, the only moderately hedged option, or the only one about the passage's main subject. Keep all 5 options within roughly the same length and grammatical shape, and make sure the correct answer is not consistently the longest or most qualified option across the set.
  - At most one wrong option per question may be eliminable on a quick read (off-topic, plainly contradicted, or extreme); the other three must require going back to a specific part of the passage.
- Explanations: give every item a 1-2 sentence explanation that cites the deciding word or text. For rc questions only, add one more sentence naming the strongest wrong option and the specific passage text that rules it out (so 2-3 sentences total for rc).
- Glossary: give every tc, tc2 and se item `glossary: [{word, meaning, synonym}]` covering each uncommon word among its options, correct answers and distractors alike. Skip everyday words. `word` matches the option text exactly. `meaning` is 3-8 plain words. `synonym` is one simple word. List entries in the final option order. tc2 has one glossary that covers blank (i) and then blank (ii).
- Answer positions and the `correct` field — do these as the LAST step, after every item's wording is final: for each item, first shuffle its options into their final published order by a real random draw, not by habit; only THEN set `correct` (the index for tc/tc2/rc, the pair of indices for se) by locating the right word(s) in that final shuffled array. Never carry over an index/indices computed earlier (e.g. from an unshuffled draft or an earlier reordering pass) — recompute it fresh against the final array every time the options are reordered. Vary the resulting positions within the day and across days, including the se pair and the first question of each passage.

Before publishing, check that:
- every tc and se item has exactly one defensible answer
- for every tc/tc2 near-miss distractor, reread the sentence as if arguing that distractor is the correct answer, and confirm the clue's named disambiguating axis (from the Distractor difficulty rule above) actually forecloses it in the exact wording given — not just that it sounds like a weaker fit. If it doesn't, sharpen the sentence's clue or swap the distractor before moving on.
- each se pair is a true synonym pair
- every rc question can be answered from its passage alone
- every rc passage has exactly 3 questions (count them for each passage separately, including the last one)
- for each rc question, cover the passage and look only at the question and its 5 options: if the correct answer can be picked out from the options alone (it is the only moderate one, the only on-topic one, the only one that sounds like the passage's main subject, the longest, etc.), rewrite the distractors until it cannot. Then argue for the strongest wrong option as if it were correct: it must take a specific line of the passage to rule it out, and that line must actually rule it out — if the strongest wrong option is still defensible, fix the option or the passage so there is exactly one defensible answer.
- there are no factual errors
- every glossary is accurate and complete
- both topics are new
- no wrong option in any item can be eliminated on tone or vibe alone without checking it against the passage or sentence — each one fails for a specific, stated reason (too narrow/broad, distorted, unsupported, extreme language, wrong-fit synonym pair, or wrong semantic axis per the near-miss check above)
- every item's `correct` index (or pair of indices) is re-derived directly from its FINAL options array and its explanation — for each item, read the explanation, find which option word(s) it names as correct, locate that word's position in the final `options` array, and confirm `correct` points there; fix any mismatch before publishing

Then list every correct-answer position and re-shuffle any items that cluster — and after any such re-shuffle, redo the `correct` re-derivation check above for the affected items before moving on.

## 4. Publish
1. Read the artifact with the Artifact tool (action "read", the url above).
2. Replace only the `var DAY = { ... }` object. Keep this shape, and put today's day number and current_date in the comment:
   var DAY = { // Day <N> — <YYYY-MM-DD>
     tc:  [ {prompt, options:[5], correct:i, explain, glossary:[...]} ×5 ],
     tc2: { prompt, blanks:[ {options:[3], correct:i}, {options:[3], correct:i} ], explain, glossary:[...] },
     se:  [ {prompt, options:[6], correct:[i,j], explain, glossary:[...]} ×4 ],
     rc:  [ {topic, passage, questions:[ {question, options:[5], correct:i, explain} ×3 ]} ×rc_passages_per_day ]
   };
   Mark blanks in tc and se prompts with ___. Mark them in the tc2 prompt with (i)___ and (ii)___.
3. If the item counts change, update the footer and the start-screen `.breakdown` chips to match. Change nothing else on the page. Navigation, the review screen and rendering live outside `DAY`.
4. Check that the `<script>` parses (run it through `new Function(...)` in Node). Then, separately, run a small Node script that evals the `DAY` object and asserts, for every tc/tc2-blank/rc item, that `correct` is a valid index into that item's `options` array, for every se item, that both of `correct`'s two indices are valid and distinct, and for every rc passage, that `questions.length === 3`. If any assertion fails, fix `DAY` and rerun before publishing. This only catches structural mistakes, not a correct-but-wrong-word index, so it does not replace the explanation cross-check above. Then publish with action "publish" and the same `url`. Never create a new artifact.
If generation fails, retry once. If it fails again, leave yesterday's page live rather than publish a partial `DAY`.

## 5. Log (only after the publish succeeds)
update words set times_covered = times_covered + 1, last_covered_date = current_date,
  first_covered_date = coalesce(first_covered_date, current_date)
where word in (<today's words>);
insert into daily_sessions (session_date, day_number, passage_topics, vocab_words, total_questions, notes)
values (current_date, <N>, array[<topics>], array[<words>], <question count>, <notes or null>);

## 6. Report coverage stats
After the log in step 5 succeeds, run:
select
  count(*) as total_words,
  count(*) filter (where times_covered >= 1) as covered_1plus,
  count(*) filter (where times_covered >= 2) as covered_2plus,
  count(*) filter (where times_covered >= 3) as covered_3plus
from words;

Then, in your final chat reply for this run (plain text, not inside the artifact), state plainly:
1. Words covered at least once: <covered_1plus> / <total_words>
2. Words covered at least twice: <covered_2plus> / <total_words>
3. Words covered at least three times: <covered_3plus> / <total_words>
```
