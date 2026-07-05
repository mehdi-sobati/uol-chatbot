# Iran Quiz Chatbot — Report

## 1. Use-case and Domain

I built a rule-based Iran trivia quiz chatbot — no machine learning. Rather than a simple Q&A bot, I wanted something with a proper flow: the bot greets the user, captures their name, then works through 9 questions covering Iran's capital, language, currency, geography and culture. Each answer gets right/wrong feedback plus a factual detail about the topic. Score is tracked throughout and summarised at the end. At any point the user can ask for a hint, check their score, or say goodbye.

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

**Named-group regex** — I used named groups `(?P<name>[A-Z][a-z]+)` and `(?P<guess>[A-Za-z][A-Za-z\s\-]{1,30})` with optional non-capturing prefixes to handle phrasings like `"I think it's X"` or `"maybe X"`. Named groups let you extract matched text by label (`result.group("name")`) rather than positional index, which made the code much easier to follow. Both patterns need to run on raw input before preprocessing — otherwise lemmatization lowercases everything and the `[A-Z]` capitalisation check fails.

**NLTK lemmatization pipeline** — I ran user input through `word_tokenize`, an `isalpha()` filter, stopword removal, `pos_tag`, and `WordNetLemmatizer` in that order. The key extra piece was `get_wordnet_pos()`, which converts Penn Treebank tags to the four categories WordNet understands (verb, adjective, adverb, noun) — without that, every word gets treated as a noun and lemmatization is much less accurate. I applied the same pipeline to the intent patterns at load time, so both sides are always in the same normalised form.

**Template substitution from JSON** — I stored responses as template strings in `intents.json` (e.g. `"Nice to meet you {name}! Let's begin with {total} questions."`) and filled them at runtime using `.format(...)`. This keeps all the response text out of the code, so the wording can be adjusted without touching any logic.

**Session memory** — I used a single `user_state` dictionary to track name, score, current question, answered count, and last guess across all turns. This made it easy to reset state between games and kept the state management in one place.

---

## 4. Code Structure

`intents.json` has two top-level keys: `intents` (11 entries: `tag`, `patterns`, `responses`) and `quiz` (9 entries: `question`, `answer`, `aliases`, `fact`, `hint`). `json.load()` is used instead of `pd.read_json()` because the two arrays have different lengths and mixed nested types that pandas cannot cleanly parse into a uniform structure.

At runtime, `load_files()` builds `rgx2int` (pattern→tag) and `int2res` (tag→responses), excluding named-group intents (`name`, `quiz_guess`) which are handled separately. The three core functions — `load_files`, `detect_pattern`, `chatbot_response` — are redefined and extended across all four parts to show deliberate iteration rather than rewriting from scratch.

---

## 5. Process Reflection

I built the chatbot incrementally across four parts, and each one revealed something I hadn't anticipated from the previous.

**Part 1 → Part 2.** Part 1 was straightforward — lists and a loop — but once I moved to regex in Part 2, I quickly realised why keeping `detect_pattern` separate from `chatbot_response` matters. When I later needed to add named-group logic for name and quiz guess capture, I only had to change one function rather than untangling everything. That separation paid off more and more as the bot grew.

**The apostrophe problem.** The most frustrating bug came from testing. I typed `"afaik it's Tehran"` and the bot didn't recognise it as a quiz answer. After some investigation I worked out that the character class `[A-Za-z\s\-]` doesn't include apostrophes, so `re.search` was stopping at the `'` in `it's` and capturing `"afaik it"` rather than `"Tehran"`. Stripping apostrophes with `re.sub(r"'", "", ...)` before the guess pattern fixed it — one line, but it took a while to understand what was actually going wrong.

**Switching from `pd.read_json()` to `json.load()`.** When I first set up Part 3 I followed the reference and used pandas. It threw a shape mismatch because my `intents.json` has two top-level keys (`intents` and `quiz`) with arrays of different lengths — pandas couldn't handle that structure. `json.load()` gave me direct dict access and was a better fit for how I was already working with the data.

**Part 4 — preprocessing order matters.** My first attempt was to preprocess all input before doing anything else, including the name and quiz guess capture. That broke immediately: lemmatization lowercases everything, so `[A-Z][a-z]+` never matched and no name was ever stored. I had to restructure so that named-group patterns run on raw input first, then the rest gets preprocessed. Once I understood why the order mattered it was an easy fix, but it wasn't obvious upfront.

**Iterative improvements from testing.** Two things stood out from my own testing. First, the bot felt repetitive — giving the same response every time for the same intent. I fixed this by adding multiple response templates per intent in `intents.json`, selected at random. Second, the answer matching was too strict — typing `"Farsi"` instead of `"Persian"` was being marked wrong. I added an `aliases` list per quiz question and used substring matching in `check_answer`, so common alternatives score correctly without any extra logic in the code.
