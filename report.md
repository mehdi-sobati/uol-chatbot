# Iran Quiz Chatbot — Report

## 1. Use-case and Domain

I built a rule-based Iran trivia quiz chatbot — no machine learning. Rather than a simple Q&A bot, I wanted something with a proper flow: the bot greets the user, captures their name, then works through 9 questions covering Iran's capital, language, currency, geography and culture. Each answer gets right/wrong feedback plus a factual detail. Score is tracked throughout and summarised at the end. At any point the user can ask for a hint, check their score, or say goodbye.

## 2. Test Cases

**Test Case 1 — Name capture and quiz start**  
`"my name is Student"` → `"Nice to meet you Student! Let's begin with 9 questions."` + Question 1 appended. Demonstrates named-group regex `(?P<name>[A-Z][a-z]+)`, storage in `user_state["name"]`, and template substitution `{name}/{total}` from `intents.json`. Input `"my name is max"` does not match (lowercase first letter fails `[A-Z]`), so the chatbot reprompts — the pattern deliberately enforces proper capitalisation.

**Test Case 2 — Flexible answer phrasing**  
Inputs like `"Tehran"`, `"I think it's Tehran"`, `"afaik it's Tehran"`, and `"maybe Tehran"` all extract the correct guess and score Correct. The optional-prefix named-group pattern handles phrasing variations. Apostrophe pre-stripping (`re.sub(r"'", "", ...)`) is applied first because `'` is not in `[A-Za-z\s\-]` and would stop the match early. `check_answer` uses substring matching against `answer` and `aliases`, so `"simply Damavand"` also scores correctly.

**Test Case 3 — NLTK preprocessing improves matching (Part 4)**

| User input | Preprocessed tokens | Intent matched |
|-----------|---------------------|----------------|
| `scoring so far` | `['score']` | `score` |
| `many thanks` | `['many', 'thank']` | `thanks` |
| `give me a clue` | `['give', 'clue']` | `hint` |
| `I don't know!` | `['know']` | `hint` |

Lemmatization (`scoring→score`, `thanks→thank`) lets variant phrasings match patterns built from canonical forms, without hardcoding every variant in the JSON.

## 3. Advanced Techniques

**Named-group regex** — I used named groups `(?P<name>[A-Z][a-z]+)` and `(?P<guess>[A-Za-z][A-Za-z\s\-]{1,30})` with optional non-capturing prefixes to handle phrasings like `"I think it's X"` or `"maybe X"`. Named groups let you extract matched text by label (`result.group("name")`) rather than positional index. Both patterns run on raw input before preprocessing — otherwise lemmatization lowercases everything and the `[A-Z]` check fails.

**NLTK lemmatization pipeline** — I ran user input through `word_tokenize`, an `isalpha()` filter, stopword removal, `pos_tag`, and `WordNetLemmatizer` in that order. The key extra piece was `get_wordnet_pos()`, which converts Penn Treebank tags to the four categories WordNet understands (verb, adjective, adverb, noun) — without that, every word gets treated as a noun. I applied the same pipeline to the intent patterns at load time, so both sides are always in the same normalised form.

**Template substitution from JSON** — responses in `intents.json` use placeholders like `{name}`, `{score}`, and `{hint}` filled at runtime with `.format(...)`. This keeps all response text out of the code so wording can be changed without touching any logic.

**Session memory** — a single `user_state` dictionary tracks name, score, current question, answered count, and last guess across all turns, making state easy to reset between games.

## 4. Code Structure

`intents.json` has two top-level keys: `intents` (11 entries with `tag`, `patterns`, `responses`) and `quiz` (9 entries with `question`, `answer`, `aliases`, `fact`, `hint`). `json.load()` is used instead of `pd.read_json()` because the two arrays have different lengths that pandas cannot parse cleanly. At runtime, `load_files()` builds `rgx2int` (pattern→tag) and `int2res` (tag→responses), excluding named-group intents handled separately. The three core functions — `load_files`, `detect_pattern`, `chatbot_response` — are redefined across all four parts to show deliberate iteration.

## 5. Process Reflection

I built the chatbot incrementally across four parts, and each one revealed something I hadn't anticipated.

**Part 1 → Part 2.** Moving to regex in Part 2 showed me why keeping `detect_pattern` separate from `chatbot_response` matters. When I added named-group logic later, I only had to change one function. That separation paid off more and more as the bot grew.

**The apostrophe problem.** I typed `"afaik it's Tehran"` and the bot didn't recognise it as a quiz answer. The character class `[A-Za-z\s\-]` doesn't include `'`, so `re.search` stopped at the apostrophe and captured `"afaik it"` instead of `"Tehran"`. Stripping apostrophes with `re.sub(r"'", "", ...)` before the guess pattern fixed it — one line, but it took a while to understand what was actually going wrong.

**Switching from `pd.read_json()` to `json.load()`.** When I first set up Part 3 I used pandas. It threw a shape mismatch because `intents.json` has two top-level keys with arrays of different lengths. `json.load()` gave me direct dict access and was a much better fit.

**Part 4 — preprocessing order matters.** My first attempt preprocessed all input before doing anything else. That broke immediately: lemmatization lowercases everything, so `[A-Z][a-z]+` never matched and no name was stored. Named-group patterns now run on raw input first, then the rest gets preprocessed.

**Iterative improvements from testing.** Two things stood out. First, the bot felt repetitive — same response every time. I added multiple templates per intent in `intents.json`, selected at random. Second, answer matching was too strict — `"Farsi"` was being marked wrong for `"Persian"`. An `aliases` list and substring matching in `check_answer` fixed this without adding any logic to the code.
