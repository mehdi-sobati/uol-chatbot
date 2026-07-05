# Iran Quiz Chatbot — Report

## 1. Use-case and Domain

A rule-based Iran trivia quiz chatbot (no ML). On launch it greets the user, captures their name via named-group regex, then poses 9 questions about Iran covering capital, language, currency, geography, and culture. Each answer gets right/wrong feedback plus an educational fact. Score is tracked throughout; the session closes with a final score summary. Side-intents (hint, score check, thanks, goodbye) are handled at any point in the conversation.

---

## 2. Test Cases

**Test Case 1 — Name capture and quiz start**  
`"my name is Student"` → `"Nice to meet you Student! Let's begin with 9 questions."` + Question 1 appended. Demonstrates named-group regex `(?P<name>[A-Z][a-z]+)`, storage in `user_state["name"]`, and template substitution `{name}/{total}` from `intents.json`.  
`"my name is max"` → no match (lowercase first letter fails `[A-Z]`), chatbot reprompts. Shows that the pattern deliberately enforces proper capitalisation rather than accepting any string.

**Test Case 2 — Flexible answer phrasing**  
All of the following inputs extract `"Tehran"` and score Correct:

| Input | Extracted guess |
|-------|----------------|
| `Tehran` | Tehran |
| `I think it's Tehran` | Tehran |
| `afaik it's Tehran` | Tehran |
| `maybe Tehran` | Tehran |

The optional-prefix named-group pattern `(?:i think it's|maybe|is it|...)?` handles phrasing variations. Apostrophe pre-stripping (`re.sub(r"'", "", ...)`) is applied before the guess pattern because `'` is not in `[A-Za-z\s\-]` and would stop the match early. `check_answer` uses substring matching against `answer` and `aliases`, so `"simply Damavand"` still scores correctly.

**Test Case 3 — NLTK preprocessing improves matching (Part 4)**

| User input | Preprocessed tokens | Intent matched |
|-----------|---------------------|----------------|
| `scoring so far` | `['score']` | `score` |
| `many thanks` | `['many', 'thank']` | `thanks` |
| `give me a clue` | `['give', 'clue']` | `hint` |
| `I don't know!` | `['know']` | `hint` |

Lemmatization (`scoring→score`, `thanks→thank`) allows variant phrasings to match patterns built from canonical forms, without hardcoding every variant in the JSON.

---

## 3. Advanced Techniques

**Named-group regex** — `(?P<name>[A-Z][a-z]+)` and `(?P<guess>[A-Za-z][A-Za-z\s\-]{1,30})` with optional non-capturing prefix groups (`(?:my name is|i am|call me|i'm)` and `(?:i think it's|maybe|is it|...)?`). Both patterns run on raw input before any preprocessing to preserve capitalisation. A three-stage detection order (name → guess → standard patterns) ensures captures take priority.

**NLTK lemmatization pipeline** — `word_tokenize → isalpha() filter → stopword removal → pos_tag → WordNetLemmatizer`. `get_wordnet_pos()` maps Penn Treebank tags (NN, VBZ, JJR, RB) to WordNet's four classes (n/v/a/r) for accurate root forms. Both intent patterns (at load time) and user input (at query time) pass through the same pipeline, ensuring they are always in the same normalised form.

**Template substitution from JSON** — all responses in `intents.json` carry placeholders (`{name}`, `{score}`, `{total}`, `{answer}`, `{hint}`) filled at runtime via `.format(...)`. A `try/except KeyError` guard handles templates that don't use every available variable.

**Session memory** — `user_state` dict (`name`, `score`, `current_question`, `answered`, `last_guess`) persists across all turns, enabling sequential question delivery, score tracking, and personalised responses without any external storage.

---

## 4. Code Structure

`intents.json` has two top-level keys: `intents` (11 entries: `tag`, `patterns`, `responses`) and `quiz` (9 entries: `question`, `answer`, `aliases`, `fact`, `hint`). `json.load()` is used instead of `pd.read_json()` because the two arrays have different lengths and mixed nested types that pandas cannot cleanly parse into a uniform structure.

At runtime, `load_files()` builds `rgx2int` (pattern→tag) and `int2res` (tag→responses), excluding named-group intents (`name`, `quiz_guess`) which are handled separately. The three core functions — `load_files`, `detect_pattern`, `chatbot_response` — are redefined and extended across all four parts to show deliberate iteration rather than rewriting from scratch.

---

## 5. Process Reflection

Development followed the four-part scaffold sequentially, each part introducing one new layer while preserving the prior structure.

**Challenge 1 — Apostrophe breaking guess capture.** Testing with natural inputs like `"afaik it's Tehran"` revealed the regex stopped at `'` (not in `[A-Za-z\s\-]`), capturing `"afaik it"` instead of `"Tehran"`. Fixed by stripping apostrophes before applying the guess pattern. This was not apparent from reading the pattern — only discovered through interactive testing.

**Challenge 2 — `pd.read_json()` incompatibility.** The two-key JSON structure with different-length arrays caused a pandas shape mismatch when loading `intents.json`. Switching to `json.load()` resolved this and gave cleaner dict access with no unexpected coercion.

**Challenge 3 — Named-group patterns must bypass preprocessing.** An early attempt to preprocess all input before named-group matching failed because lemmatization lowercases tokens, breaking the `[A-Z][a-z]+` capitalisation requirement. The fix was to run named-group patterns on raw input first, then preprocess the remainder for standard intent matching.
