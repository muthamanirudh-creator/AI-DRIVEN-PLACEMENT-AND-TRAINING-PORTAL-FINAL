# Adaptive Difficulty Practice Engine — "AI response wasn't valid JSON" fix

**Symptom:** Skill Hub → Aptitude → Adaptive Difficulty Practice Engine → *Start Practice* showed
"The AI response wasn't valid JSON (it may have been cut off for being too long)…".

**Root cause:** the panel requested only `maxTokens: 500`. Hard / Very Hard and Coding questions (long stem,
4 options, explanation) exceeded it, so the model's JSON was cut off mid-string; `parseAiJson` could only
recover truncated *arrays*, not a single truncated object, and threw the error above.

**Fix (`campus/mainapp.js` + `campus/frontend/app-runtime.js`, identical in both):**
- `AdaptivePracticePanel.fetchQuestion`: token budget 500 -> 1200 / 1800 / 2400 across up to 3 automatic
  attempts (3rd attempt prefers Gemini); retries only for bad/truncated output, never for quota/network errors;
  compact-output instructions; response shape validated/normalised (question, 2-4 options, valid correctIndex);
  `noCache` + variation seed so each click yields a genuinely new question.
- `parseAiJson(rawText, parseOpts)`: now also escapes raw newlines/tabs inside JSON strings (common with code
  snippets) for every caller, and offers opt-in `{ repairTruncated: true }` to close a cut-off string/dangling
  key and balance brackets. Existing callers behave exactly as before.
- Streak / level-up (3 correct) / level-down (1 miss), topic list, badge, explanation panel: unchanged.
- `frontend/index.html`: cache token bumped to `...-adaptivefix1` so browsers load the new script.
