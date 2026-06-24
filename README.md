# FitFindr 🛍️

FitFindr is a secondhand-shopping agent. You describe a piece you want — with an
optional size and price cap — and the agent finds the best matching listing, styles
it against your wardrobe, and writes a shareable Instagram-style "fit card" caption
for the finished look. It runs as a Gradio web app with three output panels: the
listing it found, an outfit idea, and the fit card.

```
pip install -r requirements.txt
# add GROQ_API_KEY=your_key_here to a .env file (free key at console.groq.com)
python app.py
```

Then open the URL printed in your terminal. It is usually
`http://localhost:7860`, but **check the terminal output** — the port may differ if
7860 is taken.

---

## Tool Inventory

The agent uses three tools, each a standalone function in [tools.py](tools.py). No tool
calls another tool directly — they communicate only through the session dict (see
[State Management](#state-management)).

### 1. `search_listings`

- **Inputs:**
  - `description` (`str`) — keywords describing the wanted item, e.g. `"vintage graphic tee"`. Matched against each listing's `style_tags`, `title`, `description`, and `category`.
  - `size` (`str | None`) — size to filter by, e.g. `"M"`. Case-insensitive substring match (`"M"` matches `"S/M"`). `None` skips the size filter.
  - `max_price` (`float | None`) — inclusive price ceiling. `None` skips the price filter.
- **Output:** `list[dict]` — matching listing dicts sorted by relevance (most keyword overlap first). Each dict has `id`, `title`, `description`, `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`, `platform`. Returns `[]` when nothing matches — never raises.
- **Purpose:** Turn a free-text query into a ranked shortlist of real listings. This is the only tool that touches the dataset; it is the gatekeeper that decides whether the rest of the loop runs at all.
- **Example call:**
  ```python
  >>> results = search_listings("vintage graphic tee", None, 30.0)
  >>> len(results)
  20
  >>> results[0]["title"], results[0]["price"], results[0]["platform"], results[0]["size"]
  ('Graphic Tee — 2003 Tour Bootleg Style', 24.0, 'depop', 'L')
  ```

### 2. `suggest_outfit`

- **Inputs:**
  - `new_item` (`dict`) — a single listing dict from `search_listings` (uses `title`, `category`, `style_tags`, `colors`, `description`).
  - `wardrobe` (`dict`) — the user's wardrobe, a dict with an `items` list. May be empty.
- **Output:** `str` — a non-empty styling suggestion. If the wardrobe has items, it names specific wardrobe pieces and explains why they pair well. If the wardrobe is empty, it gives general styling advice anchored to the new item only.
- **Purpose:** Make the listing actionable. Instead of "here's a tee," it tells the user *how to wear it* using clothes they already own. Calls the Groq LLM (`llama-3.3-70b-versatile`).
- **Example call:**
  ```python
  >>> suggest_outfit(results[0], get_example_wardrobe())
  'You should totally pair the Graphic Tee with your baggy straight-leg jeans and
   black combat boots for a sick grunge-inspired look. The faded graphic on the tee
   will match perfectly with the vintage vibes of the jeans and boots. Alternatively,
   you could also wear it with your wide-leg khaki trousers and chunky white sneakers
   for a more laid-back, streetwear look...'
  ```
  (It pulls the named pieces — *baggy straight-leg jeans*, *black combat boots*, *wide-leg khaki trousers*, *chunky white sneakers* — straight from the example wardrobe.)

### 3. `create_fit_card`

- **Inputs:**
  - `outfit` (`str`) — the full string returned by `suggest_outfit`.
  - `new_item` (`dict`) — the listing dict, used to pull `title`, `price`, and `platform` into the caption.
- **Output:** `str` — a short (2–4 sentence) Instagram-style caption that mentions the item, price, and platform and sounds like a real OOTD post, not a product description. Varies per input.
- **Purpose:** Give the user something they'd actually post. It's the payoff step that turns a search result + styling note into shareable content. Calls the Groq LLM (`llama-3.3-70b-versatile`).
- **Example call:**
  ```python
  >>> create_fit_card(outfit, results[0])
  'Thrifted this sick Graphic Tee from depop for $24, and it's giving me total grunge
   vibes when paired with baggy jeans and combat boots. The faded graphic adds to the
   vintage feel, perfect for a laid-back day out. This tee is so versatile, but for
   now, it's all about the grunge 🖤'
  ```
  (Item title, price, and platform are woven in naturally; the caption varies on each call thanks to `temperature=0.9`.)

---

## Planning Loop

The loop lives in `run_agent()` in [agent.py](agent.py) and runs **once per query**. It is
not an open-ended ReAct loop — it is a fixed sequence with one decision point that can
cut the run short. The interesting part is not *which* tools exist but *what the agent
decides between each step*.

**Step 1 — Initialize state.** `_new_session()` builds a session dict that is the single
source of truth for the run.

**Step 2 — Parse the query.** Regex pulls a price cap (a number after `under` / `$`) and a
size (after `size`) out of the raw text, then strips those phrases so what's left is a
clean `description`. Each parsed field is written into `session["parsed"]`. *Decision:* if
no price or size is found, those filters stay `None` and are simply skipped downstream —
the agent does not ask the user to clarify.

**Step 3 — `search_listings(description, size, max_price)`.** This is the loop's one branch
point. **If the result is empty,** the agent writes a specific recovery message into
`session["error"]` and **returns immediately** — it does *not* call `suggest_outfit` or
`create_fit_card`, because there is no item to style or caption. This is a deliberate
decision: better to tell the user how to broaden their search than to invent an outfit
for a product that doesn't exist. **If results exist,** the agent picks `results[0]` (the
highest-scoring match) as `selected_item` and continues.

**Step 4 — `suggest_outfit(selected_item, wardrobe)`.** This step *never* stops the loop.
The decision here is about *what kind* of advice to give, made inside the tool: an empty
wardrobe yields generic styling advice; a populated wardrobe yields advice that names
specific owned pieces. Either way a non-empty string comes back and the loop proceeds.

**Step 5 — `create_fit_card(outfit, selected_item)`.** Also never stops the loop. *Decision:*
if the outfit string is missing or blank, the tool skips the LLM entirely and returns a
safe fallback caption built from the item's `title`/`price`/`platform`, so the user never
sees an empty panel.

**Step 6 — Return the session.** [app.py](app.py) reads `selected_item`, the runner-up
(`search_results[1]` if present), `outfit_suggestion`, and `fit_card` out of the session
and maps them to the three output panels.

In short: the loop is linear, **search is the only gate**, and the two LLM tools are
designed to always produce *something* usable rather than ever failing the run.

---

## State Management

All information moves through one **session dict** created at the start of `run_agent()`
([agent.py](agent.py)). Tools never call each other and the user never re-enters anything
mid-run; every value is written to the session by the loop and read back by the loop.

```python
{
    "query": str,              # original user query
    "parsed": dict,            # {description, size, max_price} from Step 2
    "search_results": list,    # full ranked list from search_listings
    "selected_item": dict,     # search_results[0], fed into suggest_outfit
    "wardrobe": dict,          # loaded once, reused all run
    "outfit_suggestion": str,  # output of suggest_outfit, fed into create_fit_card
    "fit_card": str,           # output of create_fit_card
    "error": str | None,       # set only when the run stops early
}
```

How data flows:

- **query → `search_listings`:** the loop parses the raw query into `session["parsed"]` and passes those three values as arguments. The raw message is never handed to a tool.
- **`search_listings` → `suggest_outfit`:** `results[0]` is stored as `session["selected_item"]` and passed in as `new_item`. The user never re-describes the item — it flows automatically from the search hit.
- **wardrobe → `suggest_outfit`:** the wardrobe is selected once in [app.py](app.py) (`get_example_wardrobe()` or `get_empty_wardrobe()`), stored in `session["wardrobe"]`, and reused for the whole run.
- **`suggest_outfit` → `create_fit_card`:** the returned string goes into `session["outfit_suggestion"]` and is passed as `outfit`, alongside the `selected_item` already sitting in the session.
- **session → UI:** no tool returns to the user. The loop returns the session; [app.py](app.py) reads its fields into the three panels and shows `session["error"]` on its own when set.

`session["error"]` doubles as the control signal: it is `None` on the happy path, and a
message string when search came up empty — which is exactly what tells [app.py](app.py)
to show the error and blank the other two panels.

---

## Error Handling

Each tool has one explicitly handled failure mode. The principle is the same throughout:
**a failure should never produce a crash or an empty panel** — it produces either a helpful
message or a graceful fallback.

| Tool | Failure mode | What the agent does |
|------|--------------|---------------------|
| `search_listings` | No listing matches the query | Writes a specific recovery message to `session["error"]` and returns early. `suggest_outfit` and `create_fit_card` are skipped entirely — no item means nothing to style. |
| `suggest_outfit` | Wardrobe is empty | Does not crash. Branches to a generic-styling prompt anchored to the new item only and returns a non-empty string. The loop continues normally. |
| `create_fit_card` | `outfit` string missing or blank | Skips the LLM call and returns a fallback caption built from the item's `title`/`price`/`platform`, so the user always sees a complete fit card. |

### Concrete examples from testing

Each failure mode below was triggered deliberately and the real output captured.

**`search_listings` — no match (early stop).** Running `python agent.py` includes the
no-results query `"designer ballgown size XXS under $5"`, which parses to
`description="designer ballgown"`, `size="XXS"`, `max_price=5.0`. Nothing in the 40-listing
dataset matches, so `search_listings` returns `[]` and the loop produced:

```
Error message: I couldn't find any listings matching 'designer ballgown' under $5
in size XXS. Try broadening your description or removing the size or price filter.
```

`suggest_outfit` and `create_fit_card` never ran — confirming the early-stop branch fires
and the LLM tools are not called with empty input. This query is wired into the Gradio
UI's example list (the last entry in `EXAMPLE_QUERIES` in [app.py](app.py)), so the error
path is one click away in the demo.

**`suggest_outfit` — empty wardrobe.** Calling `suggest_outfit(item, get_empty_wardrobe())`
on the graphic tee did not crash. It branched to the generic-styling prompt and returned a
non-empty string anchored to the new item only (no wardrobe pieces named):

```
This graphic tee is perfect for a laid-back, grunge-inspired look, so try pairing it
with some distressed denim jeans and chunky boots for a cool, casual vibe. You can also
throw on a leather jacket or a denim jacket to add some edge to the outfit...
```

The loop then continued to `create_fit_card` as normal. In the Gradio UI this is the
**"Empty wardrobe (new user)"** radio option.

**`create_fit_card` — missing/blank outfit (fallback).** Calling `create_fit_card("", item)`
(and `create_fit_card("   ", item)`) skipped the LLM entirely and returned the safe
fallback built from the item's `title`/`price`/`platform`:

```
just copped this Graphic Tee — 2003 Tour Bootleg Style for $24.0 off depop 🖤
```

No exception, no empty panel — the user always sees a complete fit card.

On the happy path (`"vintage graphic tee under $30"`) all three panels populated correctly:
the bootleg tee listing, an outfit pairing it with wardrobe jeans and boots, and a matching
fit card caption.

---

## End-to-End Run

Running `python agent.py` drives the full planning loop with both the happy path and the
no-results path. Actual output:

```
=== Happy path: graphic tee ===

looking for a vintage graphic tee
Found: Graphic Tee — 2003 Tour Bootleg Style

Outfit: You should totally pair the Graphic Tee with your baggy straight-leg jeans
and black combat boots - it's a grunge-inspired dream come true. The faded graphic
on the tee will complement the dark wash of the jeans, and the boots will add a cool,
edgy touch. Alternatively, you could also wear the tee with your wide-leg khaki
trousers and chunky white sneakers for a more laid-back, streetwear vibe.

Fit card: Thrifted this sick Graphic Tee, a 2003 Tour Bootleg Style, for $24.0 on
depop and it's a total grunge dream come true. Paired it with baggy straight-leg jeans
and black combat boots for a cool, edgy vibe. The faded graphic on the tee looks
perfect with the dark wash of the jeans, and the boots add a nice tough touch 🙌.


=== No-results path ===

designer ballgown
Error message: I couldn't find any listings matching 'designer ballgown' under $5
in size XXS. Try broadening your description or removing the size or price filter.
```

This single run demonstrates every part of the loop: the query parses into
`description`/`size`/`max_price`, `selected_item` flows from search into `suggest_outfit`
without the user re-entering it, the outfit string flows into `create_fit_card`, and the
no-results query early-stops before either LLM tool runs. (LLM output is non-deterministic,
so the exact wording will differ run to run — the structure stays the same.)

---

## Spec Reflection

Writing the spec in [planning.md](planning.md) before any code made the implementation
mostly mechanical — the four-field tool definitions became function signatures and
docstrings almost verbatim, the five-step planning loop translated directly into
`run_agent()`, and the documented session dict matches the fields in `_new_session()`
field-for-field.

The spec and code line up closely because a couple of early design decisions were
folded back into the spec before this README was written:

- **`suggest_outfit` returns a plain string, not a slots dict.** An earlier idea was to score wardrobe items into outfit "slots" (bottom/shoes/outerwear) and return structured data, with the loop tracking `missing_slots`. It turned out simpler and more robust to let the LLM do the styling reasoning and return prose, so any wardrobe gap is communicated naturally inside the outfit string. The Planning Loop and State Management sections of planning.md now describe this string-based design directly, and there is no `missing_slots` field. (The "A Complete Interaction" worked example still walks through the older slot-scoring idea as an illustration — the binding spec is the Planning Loop / State Management sections.)
- **Two wardrobe branches, not three.** Because the tool returns a string, there is no slot bookkeeping to flag a specifically missing bottom or shoe — the LLM works with whatever the wardrobe contains. The real branch is just empty vs. populated wardrobe, both handled inside `suggest_outfit`.
- **Query parsing is regex, as the spec now specifies.** Step 1 of the planning loop calls for extracting `description`, `size`, and `max_price` with regex (`under $X`, `size X`). This is reliable and free, so no model call is spent on parsing.

What held up well: making the session dict the single source of truth meant wiring
[app.py](app.py) to the agent was trivial, and keeping `search_listings` as the only
hard stop kept the loop easy to reason about.

---

## AI Usage

I used Claude to generate the implementation from the spec, then reviewed and corrected
its output. Two specific instances:

**1. Generating `search_listings` (Milestone 3).**
*Input I gave the AI:* the Tool 1 section of [planning.md](planning.md) (what it does, the
three input parameters with types, the return value with every listing field, and the
empty-result failure mode), plus the listings field list and an instruction to use
`load_listings()` from `utils/data_loader.py`.
*What it produced:* a `search_listings(description, size, max_price)` function that filtered
by price and size, then scored each listing by keyword overlap.
*What I changed:* its first version only scored against `style_tags`. Queries like
`"track jacket"` matched poorly because the relevant words lived in the title and
description, not the tags. I overrode it to also score keyword hits in `title`,
`description`, and `category` (see [tools.py:96-103](tools.py#L96-L103)), which made the
ranking match what the spec's worked example expected. I also confirmed it returned `[]`
rather than `None` on no match, per the spec.

**2. Generating the planning loop (Milestone 4).**
*Input I gave the AI:* the Architecture diagram, the State Management section (the full
session dict), and the five-step Planning Loop section from [planning.md](planning.md),
plus the three tool signatures.
*What it produced:* a `run_agent()` that initialized the session, called the three tools in
order, and assembled the response.
*What I changed:* the generated version followed my (then incorrect) spec and tried to read
`session["outfit"]["description"]` and track `missing_slots`, treating the outfit as a dict.
Since I'd decided `suggest_outfit` should return a string, I overrode this to store and pass
the string directly ([agent.py:97-106](agent.py#L97-L106)) and removed the slot tracking.
I also tightened the early-return so the error message interpolates the actual parsed price
and size, then verified the no-results path skips both LLM tools by running `python agent.py`.
