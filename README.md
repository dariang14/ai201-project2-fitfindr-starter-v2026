# FitFindr

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first** — setup, every
> command, and what to do when something breaks.
>
> Once `python test.py` passes:
>
> ```bash
> python app.py listings --full -n 6      # read the data (Milestone 1)
> python app.py fields                    # what you can filter on
> python app.py ask 'vintage graphic tee under $30'
> ```
>
> All three tools are stubs, so that last command will do nothing useful yet.
> That's the starting position.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     HOW TO USE THIS FILE

     This is your submission. Fill each section in as you finish the milestone
     it belongs to — don't leave it all to the end.

     Unit 3 asks for the first five sections. Unit 4 adds the five below them.
     Leave the unit 4 sections alone until then; they're here so you know
     what's coming.

     Everything is pasted as TEXT. No screenshots, no images, no video links.
     A typed block of output gets full credit; a picture of the same output
     gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 3 — THE BUILD ═══════════════════════ -->

## What This Does

A user asks for something in plain language — "vintage graphic tee under $30,
size M" — and gets back a thrifted listing that matches, one or two outfit
ideas built from pieces they already own, and a short caption they could
actually post about the find. If nothing in the data matches what they asked
for, they get a message telling them what to loosen (price, size, or
keywords) instead of a crash or a made-up item.

---

## Tool Inventory

### `search_listings`

- **What it does:** Filters the 40 listings by an optional price ceiling and
  an optional size, then scores whatever's left by keyword overlap between
  `description` and each listing's title/description/style_tags, best match
  first.
- **Inputs:** `description` (str, required keywords), `size` (str or `None`
  — a whole-token, case-insensitive match against the listing's size field,
  so `"M"` matches `"S/M"` but not `"XL (oversized)"`), `max_price` (float
  or `None`, inclusive ceiling).
- **Returns:** A list of listing dicts (`id`, `title`, `description`,
  `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`,
  `platform`), sorted best-match-first, capped at
  `config.SEARCH_RESULT_LIMIT` (10).
- **When it has nothing:** Returns `[]` — an empty list, never `None` and
  never an exception. This is exactly what `run_agent` branches on.

### `suggest_outfit`

- **What it does:** Builds a prompt describing the candidate item and the
  user's wardrobe (naming each wardrobe piece by name), and asks the model
  for one or two outfits that pair them.
- **Inputs:** `new_item` (dict — a listing dict from `search_listings`),
  `wardrobe` (dict with an `'items'` key holding a list of wardrobe-item
  dicts; the list may be empty).
- **Returns:** A non-empty string of outfit suggestions from the model.
- **When it has nothing:** If `wardrobe['items']` is empty, it asks for
  general styling advice for the item instead of failing — it still returns
  a non-empty string, just without naming specific owned pieces.

### `create_fit_card`

- **What it does:** Turns an outfit suggestion and the item into a short,
  social-media-style caption that mentions the item, its price, and its
  platform once each.
- **Inputs:** `outfit` (str — the return value of `suggest_outfit`),
  `new_item` (dict — the listing dict for the item).
- **Returns:** A two-to-four sentence caption string.
- **When it has nothing:** If `outfit` is empty or whitespace-only, returns a
  descriptive message (e.g. `"No outfit suggestion to caption for <title>."`)
  instead of calling the model or raising.

---

## Planning Loop

**Branch rule:** If `search_listings` returns an empty list, put a message
in `session["error"]` naming what to change (price ceiling, size, or
keywords) and return the session immediately — do NOT call `suggest_outfit`.
Otherwise, take `search_results[0]` as `session["selected_item"]` and
continue on to `suggest_outfit` and then `create_fit_card`.

**Where it lives:** `agent.py::run_agent`

**How the query is parsed:** Regex, in `agent.py::parse_query`. One pattern
pulls a max price out of `"under $30"` or a bare `"$30"`; another pulls a
size out of `"size M"` / `"size 8"`. Both matched spans are stripped out of
the original query, and whatever text remains (whitespace-collapsed) becomes
the `description` passed to `search_listings`.

**What moves through the session:** `query` → `parsed` (description, size,
max_price) → `search_results` → `selected_item` (first of
`search_results`) → `outfit_suggestion` → `fit_card`. `error` is set only on
the empty-search branch, and when it is, every field after `search_results`
stays at its `new_session()` default (`None`).

---

## Sample Run

**One full query**

```
$ python app.py ask 'vintage graphic tee under $30'

  Found:    Y2K Baby Tee — Butterfly Print — $18.0 on depop

  Outfit:   <!-- PASTE HERE — needs a real GEMINI_API_KEY, see note below -->

  Fit card: <!-- PASTE HERE — needs a real GEMINI_API_KEY, see note below -->

2 model calls this session
```

> **Note on this run:** this session (a cloud container) had no GEMINI_API_KEY
> configured, so `suggest_outfit` and `create_fit_card` couldn't be exercised
> here. Everything that doesn't touch the model — `search_listings`, the
> query parser, and the empty-search branch below — was run and verified
> directly. The command above was run locally with a real key and the output
> pasted in.

**The three tools, tested one at a time**

```
$ python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"
[{'id': 'lst_002', 'title': 'Y2K Baby Tee — Butterfly Print', 'description': 'Super cute early 2000s baby tee with butterfly graphic. Fitted crop length. Tag says medium but fits like a small.', 'category': 'tops', 'style_tags': ['y2k', 'vintage', 'graphic tee', 'cottagecore'], 'size': 'S/M', 'condition': 'excellent', 'price': 18.0, 'colors': ['white', 'pink', 'purple'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_006', 'title': 'Graphic Tee — 2003 Tour Bootleg Style', 'description': 'Vintage-style bootleg tee with faded graphic. Slightly boxy fit. 100% cotton, soft and worn-in.', 'category': 'tops', 'style_tags': ['graphic tee', 'vintage', 'grunge', 'streetwear', 'band tee'], 'size': 'L', 'condition': 'good', 'price': 24.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_017', 'title': 'Mesh Long-Sleeve Top — Black', 'description': 'Sheer black mesh long-sleeve. Great for layering under a graphic tee or over a bralette. Stretchy material, fits true to size.', 'category': 'tops', 'style_tags': ['y2k', 'grunge', 'goth', 'layering'], 'size': 'S/M', 'condition': 'excellent', 'price': 15.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_033', 'title': 'Vintage Band Tee — Faded Grey', 'description': 'Faded grey band-style tee with distressed graphic. Crew neck. Fits boxy. Well-loved but no holes or major damage.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'band tee', 'graphic tee', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 19.0, 'colors': ['grey', 'charcoal'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_011', 'title': 'Low-Rise Cargo Pants — Khaki', 'description': 'Y2K era low-rise cargo pants. Lots of pockets. Khaki color, slightly distressed at the hems. Great for layering with a long tee.', 'category': 'bottoms', 'style_tags': ['y2k', 'cargo', '2000s', 'streetwear'], 'size': 'W29', 'condition': 'fair', 'price': 27.0, 'colors': ['khaki', 'tan'], 'brand': None, 'platform': 'poshmark'}, {'id': 'lst_015', 'title': 'Vintage Graphic Hoodie — Faded Black', 'description': 'Faded black pullover hoodie with barely-visible vintage graphic on the chest. Cozy interior. Some pilling but adds to the worn-in look.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'graphic', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 26.0, 'colors': ['black', 'charcoal'], 'brand': None, 'platform': 'depop'}]
```

```
$ python -c "from tools import suggest_outfit; from utils.data_loader import get_example_wardrobe, load_listings; print(suggest_outfit(load_listings()[0], get_example_wardrobe()))"
<!-- PASTE HERE — needs a real GEMINI_API_KEY -->
```

```
$ python -c "from tools import create_fit_card; from utils.data_loader import load_listings; print(create_fit_card('jeans and white sneakers', load_listings()[0]))"
<!-- PASTE HERE — needs a real GEMINI_API_KEY -->
```

---

## How I Used AI

**Moment 1**

- *What I asked for:* I gave Claude the `search_listings` spec from the
  docstring, including the size-matching warning about `"s" in "us 9"` being
  `True`.
- *What came back:* A tokenized match — splitting both the query size and
  the listing's size string into alphanumeric tokens with regex, and
  requiring the query to equal one whole token (so `"M"` matches `"S/M"` but
  not `"XL (oversized)"`), rather than any substring check.
- *What I changed:* Nothing — I checked it against a few tricky listings by
  hand (`"US 8"` vs size `"8"`, `"One Size / Oversized"` vs size `"L"`) and
  it held up, so I kept it as given.

**Moment 2**

- *What I asked for:* I asked Claude to implement `create_fit_card` and
  specifically to handle the documented empty-outfit case without calling
  the model.
- *What came back:* A guard that checks `if not outfit or not outfit.strip()`
  before building any prompt, returning a message naming the item instead.
- *What I changed:* The first draft of `suggest_outfit`'s prompt didn't
  explicitly tell the model to name wardrobe pieces by name when the
  wardrobe wasn't empty — I asked for that to be added so criterion 3
  (state actually reaching the tool) would be checkable against something
  concrete in the output, not just a vague "outfit idea."

<!-- ═══════════════════════ UNIT 4 — THE TEST ═══════════════════════

     Don't fill these in during unit 3.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Run Log — Before

<!-- Five criteria, five tries each, in this exact format.

     Five, because your criteria are written out of five. Mark each try PASS
     or FAIL, count the passes, and read that count against your target — a
     row targeting 4 of 5 with three PASS cells is MISSED (3/5).

     `python run_eval.py --label before` runs everything and writes the table
     into results/. Paste it here and fill in the verdicts. -->

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Real output from one try**, pasted as text, naming the file and function
that produced it:

```

```

---

## Verdicts and Diagnoses

<!-- MET or MISSED per criterion against LAST UNIT's target, plus a sentence on
     how you decided.

     Then, for every miss: which of the four places it happened — a tool, the
     loop's branch, the session, or the model's output — AND the mechanism.

     Not a diagnosis:  "The fit card was bad."
     A diagnosis:      "The fit card criterion missed on 2 of 5 items. Both had
                        an empty brand field. My prompt puts the brand in the
                        first sentence, so the card opened with a blank and read
                        like a fragment. The tool worked; the prompt assumed a
                        field that isn't always there."

     Look for a pattern. Three misses on the same tool is one problem, not
     three. -->

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 |  |  |  |  |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |

**Diagnoses**



---

## Loop Trace

<!-- One full run, printed step by step, with the MCP call visible in it.

     `python app.py ask '...' --trace` once you've added the trace.step()
     calls in Milestone 2.

     Worth pasting BOTH the happy path and the empty-search path. The empty
     one should be visibly shorter, because it stops. If your two traces are
     the same length, your branch isn't working — and this is the fastest way
     anyone will ever find that out. -->

**Happy path**

```

```

**Empty search**

```

```

**On the MCP move:** <!-- what changed in your code, and whether anything
behaved differently afterwards. If the rewire didn't work, say exactly where it
broke — the error text and the last thing that worked. That earns the point in
full. -->



---

## The Improvement

<!-- What you changed, why your diagnosis pointed at it, and the after-run in
     the same table format. One change, measured properly.

     `python run_eval.py --label after` -->

**What I changed:**

**Which failure it was meant to fix:**

### Run Log — After

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it made things worse, say that. Honestly reported, that earns full
     credit and is more interesting than one that worked. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped where
     you did. "I ran out of time" is fine if it's true. Pretending nothing is
     left is not. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 3

       [ ] criteria.md has five numbered criteria, each with a target
       [ ] Each criterion has a reason underneath it
       [ ] All five unit 3 sections above have real content
       [ ] Tool Inventory: all three tools, inputs WITH TYPES, a specific
           return value, and the empty case
       [ ] Planning Loop names the branch rule and agent.py::run_agent
       [ ] Sample Run: one full query plus the three per-tool tests, as text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN, you submit the same one
           next unit

     SUBMISSION CHECKLIST — unit 4

       [ ] mcp_server.py exists with one tool registered
           (or a written record of exactly where the rewire broke)
       [ ] Run Log — Before, five criteria, five tries each
       [ ] Real output pasted underneath, naming file and function
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, naming a place AND a mechanism
       [ ] Loop Trace, with the MCP call visible in it
       [ ] All three failure modes triggered and handled
       [ ] One improvement, with Run Log — After in the same format
       [ ] What's Still Broken
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository. Your commit history is what
     shows your criteria existed before your results did.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
