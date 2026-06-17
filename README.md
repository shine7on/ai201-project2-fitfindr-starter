# FitFindr — Starter Kit

A multi-tool AI agent that helps users find secondhand pieces and figure out how to style them. 
Built for CodePath AI201 Project 2.

## What's Included

```
ai201-project2-fitfindr-starter/
├── data/
│   ├── listings.json          # 40 mock secondhand listings
│   └── wardrobe_schema.json   # Wardrobe format + example wardrobe
├── utils/
│   └── data_loader.py         # Helper functions for loading the data
├── tools.md                   # Three agent tools: search, outfit, fit card
├── agent.py                   # Planning loop and session state management
├── app.py                     # Gradio UI
├── tests/
│   └── test_tools.py          # Pytest tests for each tool
├── planning.md                # Planning doc filled out before implementation
└── requirements.txt           # Python dependencies
```

## Setup

**macOS / Linux:**
```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

**Windows:**
```bash
python -m venv .venv
source .venv/Scripts/activate
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

`search_listings(description, size, max_price)`
- `description` (str): keywords describing the item (e.g. "vintage graphic tee")
- `size` (str | None): size to filter by, case-insensitive substring match — None skips size filtering
- `max_price` (float | None): price ceiling inclusive — None skips price filtering

Returns `list[dict]`: matching listing dicts sorted by keyword relevance, highest first. Empty list if nothing matches.

`suggest_outfit(new_item, wardrobe)`
- `new_item` (dict): a listing dict for the item the user is considering
- `wardrobe` (dict): wardrobe dict with an items key containing a list of wardrobe item dicts — may be empty

Returns `str`: 1–2 outfit suggestions. If wardrobe is empty, returns general styling advice for the item instead.

`create_fit_card(outfit, new_item)`
- `outfit` (str): the outfit suggestion string from suggest_outfit()
- `new_item` (dict): the listing dict for the thrifted item

Returns `str`: a 2–4 sentence casual Instagram-style caption mentioning the item name, price, and platform. If outfit is empty, returns an error message string instead of raising an exception.


## How the Planning Loop Works


The planning loop in agent.py runs these steps in order and branches based on what each step returns:

1. Parse the query — the LLM extracts description, size, and max_price from the user's natural language input and stores them in session["parsed"]
2. Search — calls search_listings() with the parsed parameters. If results are empty, sets session["error"] and returns early — suggest_outfit is never called with empty input
3. Select item — picks results[0] (top relevance match) and stores it in session["selected_item"]
4. Suggest outfit — calls suggest_outfit() with the selected item and the user's wardrobe. Result stored in session["outfit_suggestion"]
5. Create fit card — calls create_fit_card() with the outfit suggestion and selected item. Result stored in session["fit_card"]
6. Return session — the full session dict is returned to handle_query() in app.py
The key branch is at step 2: if search_listings returns [], the agent stops and never proceeds to the outfit or fit card tools.


## State Management


All state lives in a single session dict initialized by _new_session() at the start of each interaction. Key fields:

| Field | Set when | Used by |
|------|-------------|----------------|
| session["parsed"] | After LLM query parsing | search_listings call |
| session["search_results"] | After search_listings | Selecting selected_item |
| session["selected_item"] | After picking results[0] | suggest_outfit, create_fit_card, UI |
| session["outfit_suggestion"] | After suggest_outfit | create_fit_card, UI |
| session["fit_card"] | After create_fit_card | UI |
| session["error"] | On any early exit | UI (shown in first panel) |

No tool re-prompts the user or uses hardcoded values — every tool receives its input directly from the session.


## Interaction Walkthrough

<!-- Walk through a complete interaction step by step: natural language query → each tool call (and why) → final fit card.
     Walk through this carefully — it's how graders follow your agent's reasoning without a live demo.
     Use a specific example — do not leave this as a template. -->

**User query:**  "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers."

**Step 1 — Tool called:**
- Tool: `search_listings`
- Input: `description="vintage graphic tee", size=None, max_price=30.0`
- Why this tool: Need to find a real listing before anything else can run.
- Output: list of matching items sorted by relevance; top result is "Y2K Baby Tee — Butterfly Print, $18, Depop"

**Step 2 — Tool called:**
- Tool: `suggest_outfit`
- Input: `new_item=<Y2K Baby Tee dict>, wardrobe=<example wardrobe>`
- Why this tool: Have a selected item, now generate outfit ideas using the user's existing wardrobe pieces
- Output: "Pair the Y2K Baby Tee with your baggy straight-leg jeans and chunky white sneakers for a relaxed vintage look..."

**Step 3 — Tool called:**
- Tool: `create_fit_card`
- Input: `outfit=<suggestion from step 2>, new_item=<Y2K Baby Tee dict>`
- Why this tool: Turn the outfit into a shareable caption
- Output: "just thrifted this Y2K butterfly tee off depop for $18 and it goes with literally everything in my closet 🦋"

**Final output to user:** All three panels populate: listing details, outfit suggestion, and fit card caption.



## Error Handling and Fail Points

<!-- For each tool, describe the specific failure mode and what your agent does in response.
     This maps to the error handling section of the rubric (F5-C1). -->

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| `search_listings` | No listings match the query filters | Returns []; agent sets session["error"] with a message telling the user to try a broader description, different size, or higher budget. Early return — outfit tools never called. |
| `suggest_outfit` | Wardrobe is empty (items list is []) | Skips wardrobe-specific prompt; calls LLM with a general styling prompt instead. Returns useful advice rather than crashing. |
| `create_fit_card` | outfit string is empty or whitespace | Returns "Error: no outfit suggestion available to generate a fit card." — no exception raised. |


## Spec Reflection

<!-- Answer both questions with at least 2–3 sentences each. -->

**One way planning.md helped during implementation:**
The agent diagram made the early-return branch obvious before writing any code. Because the diagram showed search_listings returning to an error path before suggest_outfit, it was clear the planning loop needed a conditional check on the results — not just three sequential tool calls. Without drawing it out first, it would have been easy to wire all three tools unconditionally.

**One divergence from your spec, and why:**
The original spec said suggest_outfit should return an error message if the wardrobe is empty. During implementation, returning an error there felt unhelpful — the tool can still give useful general styling advice without a wardrobe. So the empty wardrobe case was changed from an error path to a fallback path that calls the LLM with a different prompt. The agent stays useful instead of stopping early.


## AI Usage

**Instance 1 — search_listings implementation:**
Gave Claude the Tool 1 spec from planning.md (inputs, return value, failure mode) and asked it to implement keyword scoring using load_listings(). Reviewed the generated code and caught a typo (saerchable instead of searchable) before running it. Also verified the size filtering used substring matching so "M" correctly matched "S/M".

**Instance 2 — Planning loop in agent.py:**
Gave Claude the agent diagram and the Planning Loop + State Management sections from planning.md. Reviewed the generated run_agent() and confirmed it branched on the search_listings result before calling suggest_outfit, and that state was stored in the session dict rather than passed as direct arguments between tools. Fixed a malformed messages dict (f-string used as a set literal instead of a proper {"role": ..., "content": ...} dict) before running.

---

## Where to Start

1. **Read `planning.md` and fill it out before writing any code.**
2. Verify the data loads correctly by running `python utils/data_loader.py`.
3. Build and test each tool individually before connecting them through your planning loop.
