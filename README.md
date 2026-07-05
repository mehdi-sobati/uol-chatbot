# Iran Quiz Chatbot

A rule-based trivia chatbot about Iran, built as a mid-term assignment for the UOL Data Programming module.

## Files

| File | Description |
|------|-------------|
| `chat-bot.ipynb` | Main notebook — 4-part chatbot implementation |
| `intents.json` | All patterns, responses, and quiz questions |
| `report.md` | Project report and process reflection |
| `reference.ipynb` | Original scaffold provided for the assignment |

## How to run

1. Open `chat-bot.ipynb` in Jupyter
2. Run all cells in order (Parts 1 → 4)
3. Interact with the chatbot at the bottom of each part

> Part 4 requires `nltk`. The notebook installs it automatically via `!pip install nltk`.

## Quick overview

The chatbot evolves across four parts:

- **Part 1** — Basic conversation loop with keyword lists
- **Part 2** — Regex pattern matching, named-group capture, `user_state` memory
- **Part 3** — All data loaded from `intents.json` at runtime
- **Part 4** — NLTK tokenisation, stopword removal, and lemmatization for flexible matching

## Dependencies

- Python 3.x
- `nltk` (auto-installed in Part 4)
- `re`, `json`, `random` — standard library
