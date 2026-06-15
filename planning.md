# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

### Tool 1: search_listings

**What it does:**
Searches the mock listings dataset for secondhand items matching the user's description, filtered by size and price ceiling. Returns a ranked list of matches sorted by keyword relevance.

**Input parameters:**
- `description` (str): Keywords describing what the user wants, e.g. "vintage graphic tee"
- `size` (str | None): Size string to filter by, e.g. "M" or "S/M". None skips size filtering.
- `max_price` (float | None): Maximum price inclusive, e.g. 30.0. None skips price filtering.

**What it returns:**
A list of listing dicts sorted by relevance score (highest first). Each dict contains: id, title, description, category, style_tags (list), size, condition, price (float), colors (list), brand, platform. Returns an empty list if nothing matches.

**What happens if it fails or returns nothing:**
The planning loop checks if results is empty. If yes, it sets session["error"] to a message explaining what filters were applied and suggests trying broader keywords, a different size, or a higher budget. The agent returns early and never calls suggest_outfit.

---

### Tool 2: suggest_outfit

**What it does:**
Given a thrifted item and the user's wardrobe, calls the Groq LLM to suggest 1-2 complete outfit combinations using pieces the user already owns.

**Input parameters:**
- `new_item` (dict): A listing dict — the item the user is considering buying.
- `wardrobe` (dict): A wardrobe dict with an 'items' key containing a list of wardrobe item dicts. May be empty.

**What it returns:**
A non-empty string with outfit suggestions. If the wardrobe has items, suggestions reference specific named pieces. If the wardrobe is empty, suggestions are general styling advice.

**What happens if it fails or returns nothing:**
If wardrobe["items"] is empty, the function sends a different prompt to the LLM asking for general styling ideas rather than wardrobe-specific combinations. Always returns a non-empty string — never crashes or returns "".

---

### Tool 3: create_fit_card

**What it does:**
Takes an outfit suggestion and listing details and calls the Groq LLM to generate a casual, shareable Instagram-style caption for the outfit.

**Input parameters:**
- `outfit` (str): The outfit suggestion string returned by suggest_outfit.
- `new_item` (dict): The listing dict for the thrifted item.

**What it returns:**
A 2-3 sentence string written like a real OOTD caption — casual, specific, mentions item name, price, and platform naturally. Uses temperature=1.2 so output varies each time.

**What happens if it fails or returns nothing:**
If outfit is empty or whitespace-only, returns the string "Error: no outfit suggestion provided. Run suggest_outfit first before creating a fit card." Never raises an exception.

---

## Planning Loop

After initializing the session, the agent parses the query with regex to extract description, size, and max_price. It calls search_listings with those parameters and checks the result:

- If results is empty: set session["error"] with a helpful message and return the session immediately. suggest_outfit is never called.
- If results is non-empty: set session["selected_item"] to results[0] and call suggest_outfit with the selected item and wardrobe. Store the result in session["outfit_suggestion"]. Then call create_fit_card with the outfit suggestion and selected item. Store the result in session["fit_card"]. Return the completed session.

The agent stops as soon as an error is detected — it never passes empty or None values into downstream tools.

---

## State Management

All state is stored in a single session dict initialized at the start of each run. Fields are written after each step and read by subsequent steps:

- session["parsed"] — set after query parsing, used by search_listings
- session["search_results"] — set after search_listings, checked by planning loop
- session["selected_item"] — set after results confirmed non-empty, passed into suggest_outfit and create_fit_card
- session["outfit_suggestion"] — set after suggest_outfit, passed into create_fit_card
- session["fit_card"] — set after create_fit_card, returned to UI
- session["error"] — set on early termination, checked by UI to display error

No value is re-entered by the user between steps.

---

## Error Handling

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Sets session["error"] to "No listings found for '[description]' in size [size] under $[max_price]. Try broader keywords, a different size, or a higher budget." Returns session early. |
| suggest_outfit | Wardrobe is empty | Sends a different LLM prompt asking for general styling advice instead of wardrobe-specific combinations. Returns a non-empty string regardless. |
| create_fit_card | Outfit input is missing or empty | Returns "Error: no outfit suggestion provided. Run suggest_outfit first before creating a fit card." without calling the LLM. |

---

## Architecture
User query

│

▼

Planning Loop ────────────────────────────────────────────────┐

│                                                         │

├─► Parse query (regex)                                   │

│       extracts: description, size, max_price            │

│       stored in session["parsed"]                       │

│                                                         │

├─► search_listings(description, size, max_price)         │

│       │                                                 │

│       │ results=[]                                      │

│       ├──► session["error"] = "No listings found..." ──►│

│       │    return session early                         │

│       │                                                 │

│       │ results=[item, ...]                             │

│       ▼                                                 │

│   session["selected_item"] = results[0]                 │

│       │                                                 │

├─► suggest_outfit(selected_item, wardrobe)               │

│       │                                                 │

│   session["outfit_suggestion"] = "..."                  │

│       │                                                 │

└─► create_fit_card(outfit_suggestion, selected_item)     │

│                                                 │

session["fit_card"] = "..."                           │

│                                          ───────┘

▼                                         error path ends here

Return session
---

## AI Tool Plan

**Milestone 3 — Individual tool implementations:**

For search_listings, I gave Claude the Tool 1 spec block (inputs, return value, failure mode) and asked it to implement the function using load_listings() from the data loader. I verified the generated code filtered by all three parameters and handled the empty-results case before running it. Then tested with 3 queries including one designed to return no results.

For suggest_outfit and create_fit_card, I gave Claude the Tool 2 and Tool 3 spec blocks and asked it to write the Groq API calls and prompts. I verified the empty wardrobe branch was present in suggest_outfit and that create_fit_card guarded against empty outfit input before running. I adjusted the create_fit_card prompt to sound more like a real caption and added temperature=1.2 after testing showed identical outputs on repeated calls.

**Milestone 4 — Planning loop and state management:**

I gave Claude the Architecture diagram and the Planning Loop + State Management sections and asked it to implement run_agent() in agent.py. I verified the generated code branched on empty search results, stored values in the session dict, and did not call all three tools unconditionally before running it.

---

## A Complete Interaction (Step by Step)

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
The agent parses the query with regex. Extracts description="vintage graphic tee", size=None (no size mentioned), max_price=30.0. Stores in session["parsed"]. Calls search_listings("vintage graphic tee", None, 30.0).

**Step 2:**
search_listings loads all 40 listings, filters to those under $30, scores each by keyword overlap with "vintage graphic tee", drops zero scores, and returns 3 matches sorted by score. Top result: Y2K Baby Tee — Butterfly Print, $18, Depop. Stored in session["selected_item"].

**Step 3:**
Agent calls suggest_outfit(session["selected_item"], wardrobe). Wardrobe has 10 items so the LLM receives the full wardrobe list and the item details. LLM returns two specific outfit combinations referencing baggy jeans, chunky sneakers, combat boots, and denim jacket by name. Stored in session["outfit_suggestion"].

**Step 4:**
Agent calls create_fit_card(session["outfit_suggestion"], session["selected_item"]). LLM receives the outfit suggestion and item details and returns a 2-3 sentence Instagram-style caption mentioning the item name, price, and platform. Stored in session["fit_card"].

**Final output to user:**
- Left panel: Y2K Baby Tee listing details (title, price, platform, size, condition, colors, description)
- Middle panel: Two outfit combinations referencing specific wardrobe pieces
- Right panel: Instagram-style caption ready to copy and post