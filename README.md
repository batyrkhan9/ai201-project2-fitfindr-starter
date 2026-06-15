# FitFindr — Starter Kit

This starter kit contains everything you need to begin Project 2.

## What's Included

```
ai201-project2-fitfindr-starter/
├── data/
│   ├── listings.json          # 40 mock secondhand listings
│   └── wardrobe_schema.json   # Wardrobe format + example wardrobe
├── utils/
│   └── data_loader.py         # Helper functions for loading the data
├── planning.md                # Your planning template — fill this out first
└── requirements.txt           # Python dependencies
```

## Setup

```bash
pip install -r requirements.txt
```

Set your Groq API key in a `.env` file (get a free key at [console.groq.com](https://console.groq.com)):
```
GROQ_API_KEY=your_key_here
```

## The Mock Listings Dataset

`data/listings.json` contains 40 mock secondhand listings across categories (tops, bottoms, outerwear, shoes, accessories) and styles (vintage, y2k, grunge, cottagecore, streetwear, and more).

Each listing has: `id`, `title`, `description`, `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`, and `platform`.

Load it with:
```python
from utils.data_loader import load_listings
listings = load_listings()
```

## The Wardrobe Schema

`data/wardrobe_schema.json` defines the format your agent uses to represent a user's existing wardrobe. It includes:

- `schema`: field definitions for a wardrobe item
- `example_wardrobe`: a sample wardrobe with 10 items you can use for testing
- `empty_wardrobe`: a starting template for a new user

Load an example wardrobe with:
```python
from utils.data_loader import get_example_wardrobe
wardrobe = get_example_wardrobe()
```

## Tool Inventory

### `search_listings(description: str, size: str | None, max_price: float | None) → list[dict]`
Searches the mock listings dataset for items matching the query. Filters by price and size if provided, then scores each listing by keyword overlap with the description across title, description, and style_tags fields. Returns a list of matching listing dicts sorted by relevance score, highest first. Returns an empty list if nothing matches — never raises an exception.

### `suggest_outfit(new_item: dict, wardrobe: dict) → str`
Takes the selected listing and the user's wardrobe and calls the Groq LLM to suggest 1-2 complete outfit combinations. If the wardrobe is empty, returns general styling advice for the item instead of crashing. Always returns a non-empty string.

### `create_fit_card(outfit: str, new_item: dict) → str`
Takes the outfit suggestion and the listing dict and calls the Groq LLM to generate a casual, shareable caption in the style of an Instagram OOTD post. Guards against empty outfit input — returns a descriptive error string instead of crashing. Uses temperature=1.2 so output varies each time.

---

## How the Planning Loop Works

The planning loop in `agent.py` runs these steps in order and branches on results:

1. Parse the query with regex to extract a clean description, size, and max_price
2. Call `search_listings()` with the parsed parameters
3. **Branch:** if results are empty, set `session["error"]` with a helpful message and return early — `suggest_outfit` is never called with empty input
4. If results exist, set `session["selected_item"]` to the top result
5. Call `suggest_outfit()` with the selected item and wardrobe
6. Call `create_fit_card()` with the outfit suggestion and selected item
7. Return the completed session dict

The key decision point is step 3 — the agent behaves differently depending on whether listings were found.

---

## State Management

All state lives in a single session dict initialized by `_new_session()`:

| Key | Set when | Used by |
|-----|----------|---------|
| `parsed` | After query parsing | `search_listings` |
| `search_results` | After `search_listings` | Planning loop branch |
| `selected_item` | After results confirmed non-empty | `suggest_outfit`, `create_fit_card` |
| `outfit_suggestion` | After `suggest_outfit` | `create_fit_card` |
| `fit_card` | After `create_fit_card` | Returned to UI |
| `error` | On any early termination | UI error display |

No value is re-entered by the user between steps — each tool receives its inputs directly from the session.

---

## Error Handling

**`search_listings`** — returns `[]` on no matches. The planning loop catches this and sets `session["error"]` to a message that tells the user what filters were applied and suggests trying broader keywords, a different size, or a higher budget.

**`suggest_outfit`** — checks `wardrobe["items"]` before building the prompt. If empty, sends a different prompt asking for general styling advice rather than wardrobe-specific combinations. Never returns an empty string.

**`create_fit_card`** — guards against an empty or whitespace-only outfit string at the top of the function. Returns `"Error: no outfit suggestion provided. Run suggest_outfit first before creating a fit card."` instead of passing empty input to the LLM.

### Concrete example from testing
Running `create_fit_card("", results[0])` returns the error string above instead of crashing. Running `suggest_outfit(results[0], get_empty_wardrobe())` returns general styling advice like "pair with high-waisted jeans or a flowy skirt" instead of an empty string.

---

## Spec Reflection

**One way the spec helped:** Writing the planning loop conditional logic in plain English before coding made the branch point (empty results → early return) obvious. Without that, it would have been easy to accidentally call `suggest_outfit` with an empty list.

**One way implementation diverged:** The spec said to document how the query gets parsed but left the method open. I used regex instead of asking the LLM to parse the query — regex is faster, cheaper (no API call), and reliable enough for the structured patterns in these queries (size X, under $Y).

---

## AI Usage

**Instance 1 — `search_listings` implementation:** I gave Claude the tool spec from planning.md (inputs, return value, failure mode) and asked it to implement the function using `load_listings()`. The generated code had an indentation bug on the first line — `listings = load_listings()` was missing its leading spaces, causing a syntax error. I caught and fixed this before running.

**Instance 2 — `suggest_outfit` and `create_fit_card`:** I gave Claude the tool specs for both LLM-calling tools and asked it to write the prompts and Groq API calls. The prompts it generated were functional but I adjusted the `create_fit_card` prompt to be more specific about caption style — the original read too much like a product description rather than an authentic OOTD post. I also added `temperature=1.2` explicitly after testing showed identical outputs on repeated calls.
