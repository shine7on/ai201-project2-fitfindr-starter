# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
Searches the mock listings dataset based on the input and returns matching items.
If it dont not find any matches, return 'couldn't find' to the user and ask more info (or fallback strategy: different size?)

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `description` (str): describe about this item; name
- `size` (str): size the user is looking for
- `max_price` (float): maximum price the user would pay

**What it returns:**
<!-- Describe the return value — what fields does a result contain? -->
If it finds something that matches user's request, it will returns three matching listings by relevalnce.
The return result should follow the 'items' schema.

**What happens if it fails or returns nothing:**
<!-- What should the agent do if no listings match? -->
Return 'couldn't find' to the user and ask more info.

---

### Tool 2: suggest_outfit

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
It suggests the outfit that fits well with the new item based on user's wardrobe.


**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `new_item` (dict): new item; use item schema
- `wardrobe` (dict): user's wardrobe that contains multiple items.

**What it returns:**
<!-- Describe the return value -->
Return string that tells outfit such as "Pair this with your wide-leg jeans and platform Docs for a classic 90s grunge look. Roll the sleeves once and tuck the front corner slightly for shape."

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the wardrobe is empty or no outfit can be suggested? -->
Return that 'Could not find' message to a user.

---

### Tool 3: create_fit_card

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `outfit` (str): explain about the outfit
- `new_item` (dict): new item user bought which used in the outfit coordinate

**What it returns:**
<!-- Describe the return value -->
Return the string talking about this new item and outfit such as "thrifted this faded band tee off depop for $22 and honestly it was made for my wide-legs 🖤 full look in my stories."

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the outfit data is incomplete? -->
Return the message 'Incomplete outfit.'

---

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->

---

## Planning Loop

**How does your agent decide which tool to call next?**
<!-- Describe the logic your planning loop uses. What does it look at? What conditions change its behavior? How does it know when it's done? -->
After search_listings(desc, size, max_price) runs, check if results is empty. If yes, set an error message in the session and return early. 
If no, set selected_item = results[0] and proceed to suggest_outfit(new_item, wardrobe).
Once you run suggest_outfit, check if the input wardrobe is empty. If so, return an error message.
Also, if the items in wardrobe can not make any outfit with the new item, it should return an error message as well.

---

## State Management

**How does information from one tool get passed to the next?**
<!-- Describe how your agent stores and accesses state within a session. What data is tracked? How is it passed between tool calls? -->
After search_listings(desc, size, max_price), if it finds the items, it should store the list of suggested items.
Then, in the session select one of them from the list; for example, selected_item = results[0].
As keeping the selected item, it inserts in suggest_outfit(selected_item, wardrobe).
Then, as it returns the valid output (not error message), it stores as the outfit_suggestion.
Then, put in the create_fit_card(outfit_suggestion, selected_item). Then, it returns the output.
---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Send an error message "Couldn't find" and return. |
| suggest_outfit | Wardrobe is empty | Send an error message "Wardrobe is empty" and return. |
| create_fit_card | Outfit input is missing or incomplete | Send error messages "Invalid input" or "Incomplete outfit" and return. |

---

## Architecture

<!-- Draw a diagram of your agent showing how the components connect:
     User input → Planning Loop → Tools (search_listings, suggest_outfit, create_fit_card)
                                                                          ↕
                                                                   State / Session
     Show what triggers each tool, how state flows between them, and where error paths branch off.
     Use ASCII art or a Mermaid diagram (https://mermaid.js.org/syntax/flowchart.html).
     Do NOT embed an image — graders need to read your diagram directly in the file;
     an embedded image or screenshot cannot be evaluated.
     You'll share this diagram with an AI tool when asking it to implement
     the planning loop and each individual tool. -->
User query
    │
    ▼
Planning Loop ───────────────────────────────────────────┐
    │                                                    │
    ├─► search_listings(description, size, max_price)    │
    │       │ results=[]                                 │
    │       ├──► [ERROR] "No listings found..." → return │
    │       │                                            │
    │       │ results=[item, ...]                        │
    │       ▼                                            │
    │   Session: selected_item = results[0]              │
    │       │                                            │
    ├─► suggest_outfit(selected_item, wardrobe)          │
    │       │                                            │
    │       |──► [ERROR] "No outfit found..." → return   │
    │       │                                            │
    │   Session: outfit_suggestion = "..."               │
    │       │                                            │
    └─► create_fit_card(outfit_suggestion, selected_item)│
            │                                            │
            |──► [ERROR] "Invalid outfit..." → return    │
            │                                            │
            │                                            │
        Session: fit_card = "..."                        │
            │                                            └─ error path returns here
            ▼
        Return session

---

## AI Tool Plan

<!-- For each part of the implementation below, describe:
     - Which AI tool you plan to use (Claude, Copilot, ChatGPT, etc.)
     - What you'll give it as input (which sections of this planning.md, your agent diagram)
     - What you expect it to produce
     - How you'll verify the output matches your spec before moving on

     "I'll use AI to help me code" is not a plan.
     "I'll give Claude my Tool 1 spec (inputs, return value, failure mode) and ask it to implement
     search_listings() using load_listings() from the data loader — then test it against 3 queries
     before trusting it" is a plan. -->

**Milestone 3 — Individual tool implementations:**
- For search_listings, I'll give Claude the Tool 1 block from planning.md (inputs, return value, failure mode) and ask it to implement the function using load_listings() from the data loader. Before running it, I'll check that the generated code filters by all three parameters and handles the empty-results case. Then I'll test it with 3 queries.

- For suggest_outfit, I'll give Claude the Tool 2 block from planning.md (inputs, return value, failure mode) and ask it to implement the function using load_listings() from the data loader. Before running it, I'll check that the generated code and handles the empty-results case. Then I'll test it with 3 queries.

- For fit_card, I'll give Claude the Tool 3 block from planning.md (inputs, return value, failure mode) and ask it to implement the function using load_listings() from the data loader. Before running it, I'll check that the generated code and handles the empty-results case. Then I'll test it with 3 queries.

**Milestone 4 — Planning loop and state management:**
- For planning loop, I'll give Claude the diagram and Planning Loop + State Management sections from planning.md. When I receive the generated code, review if it handles error, store the valid result into sesion memory.

---

## A Complete Interaction (Step by Step)

Write out what a full user interaction looks like from start to finish — tool call by tool call. Use a specific example query.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
<!-- What does the agent do first? Which tool is called? With what input? -->
Input: description, size, max_price
Use search_listings(description, size, max_price). Inside this, call load_listing() to check if it has the item.
If yes, return the list of at most 3 items; if no, send the error message and return.

**Step 2:**
<!-- What happens next? What was returned from step 1? What tool is called now? -->
Now, select the item from the list from search_listings. selected_item = results[0].
Then, insert the selected_item into suggest_outfit with wardrobe that user owns.
Then, it will store the outfit_suggestion if it finds the outfit.
Otherwise, it will send the error message and return.

**Step 3:**
<!-- Continue until the full interaction is complete -->
Now, use the outfit_suggestion and selected_item to run create_fit_card.
If the outfit is invalid (empty, incomplete), it sends the error message and return.
If it's valid, store the result into fit_card. Then, return session.

**Final output to user:**
<!-- What does the user actually see at the end? -->
User will get the string saying the outfit with the new item.
