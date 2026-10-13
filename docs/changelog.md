# Changelog

- **2026-09-19: Launch.** Supabase, the Artifact page and a daily routine, all bootstrapped by one setup skill. This replaced an earlier static CSV word list and a hand-copied page.
- **Late Sept: Glossary.** Text completion and sentence equivalence items show short definitions of the hard answer choices.
- **Late Sept: Date fix.** `session_date` is always `current_date`, and a second run on the same day stops. An earlier workaround had pushed logged dates a day ahead.
- **2026-10-03: Topic de-duplication.** Topics are checked against the full history, rephrasings included. The old 7-day window had let topics repeat after about a week. The `passage_topic_lookback_days` config key is retired, and live databases that still have it can delete it.
- **2026-10-13: Answer positions.** Correct answers are placed by random draw, because they had been clustering in habitual slots.
- **2026-10-13: Free navigation.** The page gained Back/Next, clickable progress dots and a review screen with a Finish button. Answers you pick are graded even if you never press Check, and blank answers count as wrong.
- **2026-10-13: Condensed.** The routine prompt, skill and docs were trimmed to describe only how the tool works now.
