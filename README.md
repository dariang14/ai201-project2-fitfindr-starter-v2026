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

  Outfit:   **Outfit 1: Casual Y2K Streetwear**
  *   **Top:** Y2K Butterfly Baby Tee
  *   **Bottoms:** Baggy straight-leg jeans (dark wash)
  *   **Outerwear:** Vintage black denim jacket
  *   **Shoes:** Chunky white sneakers
  *   **Accessory:** Black crossbody bag

  **Outfit 2: Contrast Neutral Mix**
  *   **Top:** Y2K Butterfly Baby Tee
  *   **Bottoms:** Wide-leg khaki trousers
  *   **Accessory:** Brown leather belt
  *   **Outerwear:** Black cropped zip hoodie (worn open or carried)
  *   **Shoes:** Black combat boots

  Fit card: Found this butterly print baby tee on Depop for $18 and my inner 2000s pop star is screaming. Paired it with baggy denim and chunky sneakers for the ultimate Y2K street style moment. Who knew $18 could buy time travel?

2 model calls this session, 506 prompt + 199 output tokens
```

**The three tools, tested one at a time**

```
$ python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"
[{'id': 'lst_002', 'title': 'Y2K Baby Tee — Butterfly Print', 'description': 'Super cute early 2000s baby tee with butterfly graphic. Fitted crop length. Tag says medium but fits like a small.', 'category': 'tops', 'style_tags': ['y2k', 'vintage', 'graphic tee', 'cottagecore'], 'size': 'S/M', 'condition': 'excellent', 'price': 18.0, 'colors': ['white', 'pink', 'purple'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_006', 'title': 'Graphic Tee — 2003 Tour Bootleg Style', 'description': 'Vintage-style bootleg tee with faded graphic. Slightly boxy fit. 100% cotton, soft and worn-in.', 'category': 'tops', 'style_tags': ['graphic tee', 'vintage', 'grunge', 'streetwear', 'band tee'], 'size': 'L', 'condition': 'good', 'price': 24.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_017', 'title': 'Mesh Long-Sleeve Top — Black', 'description': 'Sheer black mesh long-sleeve. Great for layering under a graphic tee or over a bralette. Stretchy material, fits true to size.', 'category': 'tops', 'style_tags': ['y2k', 'grunge', 'goth', 'layering'], 'size': 'S/M', 'condition': 'excellent', 'price': 15.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_033', 'title': 'Vintage Band Tee — Faded Grey', 'description': 'Faded grey band-style tee with distressed graphic. Crew neck. Fits boxy. Well-loved but no holes or major damage.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'band tee', 'graphic tee', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 19.0, 'colors': ['grey', 'charcoal'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_011', 'title': 'Low-Rise Cargo Pants — Khaki', 'description': 'Y2K era low-rise cargo pants. Lots of pockets. Khaki color, slightly distressed at the hems. Great for layering with a long tee.', 'category': 'bottoms', 'style_tags': ['y2k', 'cargo', '2000s', 'streetwear'], 'size': 'W29', 'condition': 'fair', 'price': 27.0, 'colors': ['khaki', 'tan'], 'brand': None, 'platform': 'poshmark'}, {'id': 'lst_015', 'title': 'Vintage Graphic Hoodie — Faded Black', 'description': 'Faded black pullover hoodie with barely-visible vintage graphic on the chest. Cozy interior. Some pilling but adds to the worn-in look.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'graphic', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 26.0, 'colors': ['black', 'charcoal'], 'brand': None, 'platform': 'depop'}]
```

```
$ python -c "from tools import suggest_outfit; from utils.data_loader import get_example_wardrobe, load_listings; print(suggest_outfit(load_listings()[0], get_example_wardrobe()))"
**Outfit 1: Casual Streetwear**
*   **Top:** White ribbed tank top
*   **Outerwear:** Oversized grey crewneck sweatshirt (worn draped over the shoulders or layered)
*   **Shoes:** Chunky white sneakers
*   **Accessory:** Black crossbody bag

**Outfit 2: Edgy Contrast**
*   **Top:** Black cropped zip hoodie
*   **Shoes:** Black combat boots
*   **Accessory:** Brown leather belt (to break up the waist and anchor the vintage blue wash)
```

```
$ python -c "from tools import create_fit_card; from utils.data_loader import load_listings; print(create_fit_card('jeans and white sneakers', load_listings()[0]))"
Found these vintage Levi's 501 jeans on Depop for $38 and my casual off-duty model era is officially unlocked. Paired them with crisp white sneakers for that effortlessly cool 90s running-errands aesthetic. Honestly, nothing beats broken-in denim that actually fits like a glove.
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

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1. matching query completes all three tools | 4/5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 2. impossible query stops before second tool | 5/5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 3. selected_item identity into suggest_outfit | 5/5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 4. fit card mentions the item's exact price | 4/5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 5. empty wardrobe returns non-empty advice | 5/5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |

Full table and output: `results/run_2026-10-18_2110_before.md`.

**Real output from one try**, from `run_eval.py::run_once` calling
`agent.py::run_agent` (scenario "matching query completes", try 1):

```
- stopped early: no
- selected_item: Y2K Baby Tee — Butterfly Print ($18.0, depop)
- search_results: 10

Outfit suggestion:
**Outfit 1: Y2K Streetwear**
*   **Top:** Y2K Butterfly Baby Tee
*   **Bottoms:** Baggy straight-leg jeans
*   **Outerwear:** Vintage black denim jacket
*   **Shoes:** Chunky white sneakers
*   **Accessory:** Black crossbody bag

**Outfit 2: Casual Contrast**
*   **Top:** Y2K Butterfly Baby Tee
*   **Bottoms:** Wide-leg khaki trousers
*   **Outerwear:** Black cropped zip hoodie (worn open or tied around the waist)
*   **Shoes:** Chunky white sneakers
*   **Accessory:** Brown leather belt

Fit card:
Found the ultimate Y2K streetwear piece on Depop for just $18. Pair this butterfly baby tee with baggy denim and chunky sneakers to channel peak 2000s mall-goth energy. Nostalgia looks good on a budget.
```

---

## Verdicts and Diagnoses

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 | matching query completes all three tools | 4/5 | MET (5/5) | All 5 tries had `error: None`, a non-null `selected_item`, a non-empty `outfit_suggestion`, and a non-empty `fit_card` — read directly off `results/run_..._before.md`. |
| 2 | impossible query stops before second tool | 5/5 | MET (5/5) | All 5 tries show `stopped early: yes`, `search_results: 0`, and the trace ends at step `[3] branch` — `suggest_outfit` never appears in the trace for any try. |
| 3 | selected_item identity into suggest_outfit | 5/5 | MET (5/5) | For each try, the trace's `[3] suggest_outfit  in:` line (e.g. `Y2K Baby Tee — Butterfly Print ($18.0, depop)`) is character-for-character the same as that try's `selected_item` line. |
| 4 | fit card mentions the item's exact price | 4/5 | MET (5/5) | The item's price is `$18.0`. All 5 fit cards contain the substring `$18` (e.g. "...Depop for $18 and my inner 2000s pop star..."). |
| 5 | empty wardrobe returns non-empty advice | 5/5 | MET (5/5) | All 5 tries with the empty wardrobe produced a non-empty `outfit_suggestion` of general advice (no wardrobe piece named, since there were none to name) rather than an empty string or an error. |

**Diagnoses**

No misses on the five formal criteria this run — all five targets were met,
several at a higher rate than the target called for (criteria 1 and 4 both
hit 5/5 against a 4/5 target).

That's worth being suspicious of rather than just taking a win, so I went
looking for where a target might be too easy rather than assuming the build
is flawless. **Criterion 1's target (4/5) was never actually tested, because
the scenario reruns the identical query five times.** `search_listings` has
no randomness in it — the same `description`/`size`/`max_price` always
produces the same ranked list — so five tries of one query only ever measure
whether it matches *once*, not whether the keyword search holds up across
different phrasings, which is exactly the risk criterion 1's reasoning names.

To actually probe that, I ran `search_listings` directly (no model needed)
against phrasings further from the listing data's own wording:

```
$ python -c "from tools import search_listings; r = search_listings('sneakers for running'); [print(x['title']) for x in r[:5]]"
90s Track Jacket — Navy/White Stripe
Knit Cardigan — Chunky Brown
Low-Rise Cargo Pants — Khaki
Mesh Long-Sleeve Top — Black
Platform Sneakers — White Chunky Sole
```

**The mechanism:** `_keywords()` tokenized the whole query, including filler
words like "for". The word "for" appears inside many unrelated listings'
descriptions ("great **for** layering," "**for** layering with a long tee"),
so it counted as a real keyword match and inflated those listings' scores —
enough to outrank the one listing that's actually a pair of sneakers
(`Platform Sneakers — White Chunky Sole`, buried at position 5 instead of 1).
This is a tool-level bug in `search_listings`'s scoring, not a loop or
session problem — the loop and state handling around it are fine, as
criterion 3 already confirms. It's outside all five formal criteria because
none of their scenario queries happen to contain a common filler word, which
is itself the finding: **the formal test suite under-tests criterion 1's own
stated risk.**

---

## Loop Trace

**Happy path**

```
$ python app.py ask 'vintage graphic tee under $30' --trace

[1] parse_query
      in:  vintage graphic tee under $30
      out: dict with keys: description, size, max_price
[2] search_listings (via MCP)
      in:  dict with keys: description, size, max_price
      out: 10 items: Y2K Baby Tee — Butterfly Print, Graphic Tee — 2003 Tour Bootleg Style, Vintage Band Tee — Faded Grey … +7 more
[3] suggest_outfit
      in:  Y2K Baby Tee — Butterfly Print ($18.0, depop)
      out: **Outfit 1: Y2K Streetwear** *   **Top:** Y2K Butterfly Baby Tee *   **Bottoms:** Baggy straight-leg jeans *  …
[4] create_fit_card
      in:  **Outfit 1: Y2K Streetwear** *   **Top:** Y2K Butterfly Baby Tee *   **Bottoms:** Baggy straight-leg jeans *  …
      out: Found the ultimate Y2K streetwear piece on Depop for just $18. Pair this butterfly baby tee with baggy denim a…
```

**Empty search**

```
$ python app.py ask 'designer ballgown size XXS under $5' --trace

[1] parse_query
      in:  designer ballgown size XXS under $5
      out: dict with keys: description, size, max_price
[2] search_listings (via MCP)
      in:  dict with keys: description, size, max_price
      out: [] (empty)
[3] branch
      →    no search results — stopping before suggest_outfit

  No listings matched that query. Try loosening the price ceiling, dropping the size filter, or using fewer/different keywords.

0 model calls this session
```

The empty-search trace is 3 steps and makes 0 model calls; the happy path is
4 steps and makes 2. That gap is the branch actually doing its job.

**On the MCP move:** `search_listings` moved from a direct function call to
`mcp_client.call_tool("search_listings", {...})` in `agent.py::run_agent` —
step `[2]` above is labeled `search_listings (via MCP)` for exactly that
reason. The return shape didn't change at all: same list of listing dicts,
same empty-list-on-no-match behavior. I verified the rewire two ways —
`python mcp_client.py` lists the registered tool with its full schema
(`description: string`, `size: string (optional)`, `max_price: number
(optional)`), and both traces above show the MCP step returning normal data.

I also deliberately called a tool name that isn't registered, to see the
failure mode before trusting the happy path:

```
$ python -c "from mcp_client import call_tool, MCPError; call_tool('nonexistent_tool', {})"
MCPError: Couldn't call 'nonexistent_tool' over MCP: unhandled errors in a TaskGroup (1 sub-exception)
```

Not the specific "the server doesn't offer a tool called..." message
`mcp_client.py`'s own code tries to raise for this case — the error happens
inside the `stdio_client`/`ClientSession` async context manager, and asyncio
wraps whatever's raised there in a `TaskGroup` exception before it reaches my
code. Either way, it surfaces as a catchable `MCPError`, which is what
matters for `run_agent`: `call_tool` is already wrapped in a `try/except
MCPError` there, so a broken MCP call becomes a `session["error"]` message
instead of a stack trace — the same pattern used for `ModelUnavailable`
below.

---

## The Improvement

**What I changed:** Added a `_STOPWORDS` set (`a`, `an`, `the`, `for`, `in`,
`on`, `of`, `with`, `and`, `or`, `to`, `is`, `it`, `my`, `me`, `i`, `under`,
`size` — about 18 common filler words) and filtered it out of
`tools.py::_keywords()`, the function that tokenizes both the query and each
listing's text before scoring keyword overlap.

**Which failure it was meant to fix:** Not one of the five formal criteria —
all five were already MET on the before run. It fixes the search-quality bug
found in the Diagnoses section above: a query like `"sneakers for running"`
matched on the word `"for"` against any listing whose description happened
to contain it (e.g. `"great for layering"`), which could push an irrelevant
item above the one listing that actually matched. Before the fix, that exact
query ranked a track jacket and a cardigan above both real sneaker listings;
after, it returns exactly the two sneaker listings and nothing else:

```
$ python -c "from tools import search_listings; r = search_listings('sneakers for running'); [print(x['title']) for x in r]"
Platform Sneakers — White Chunky Sole
Low-Top Canvas Sneakers — Off-White
```

### Run Log — After

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1. matching query completes all three tools | 4/5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 2. impossible query stops before second tool | 5/5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 3. selected_item identity into suggest_outfit | 5/5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 4. fit card mentions the item's exact price | 4/5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 5. empty wardrobe returns non-empty advice | 5/5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |

Full table and output: `results/run_2026-10-18_2123_after.md`.

**Did it help, and how do I know:** Yes, but the formal run log alone
doesn't show it — and I want to be upfront about that rather than imply it
does. All five criteria read identically before and after (same
`selected_item` for every scenario, same pass counts), because none of the
five scenario queries contain a stopword that was actually causing trouble.
That's expected, not a failed measurement: this fix targets a bug the
official scenarios never exercised in the first place, which is the whole
reason I had to go looking for it with a separate, targeted test rather than
reading it off the table. The real evidence is the standalone before/after
on `"sneakers for running"` above — 2 correct results out of what was 9
noisy ones, with no change to any of the five tracked scenarios' top
results. If I were to redo Milestone 3, I'd add a sixth scenario built
specifically to query with natural, stopword-containing phrasing, so this
class of regression would show up in the formal table instead of needing a
side investigation to catch.

---

## What's Still Broken

None of the five formal criteria are currently missed, so there's nothing in
that category to diagnose further. But "nothing failed" isn't the same as
"nothing left," and a few real gaps remain:

- **Criterion 1's scenario still doesn't test what its own reasoning worries
  about.** I fixed the stopword bug the current scenario couldn't catch, but
  I didn't go back and add a scenario that actually varies phrasing the way
  criterion 1's target was written to anticipate. If I had more time, I'd add
  2-3 more scenario entries under criterion 1 with deliberately different
  wording for the same intent (e.g. "comfy old band shirt" instead of
  "vintage graphic tee") and see if the 4/5 target still holds — right now
  it's an aspirational number, not a measured one.
- **The stopword list is hand-picked and small (~18 words), not a real
  stopword corpus.** It fixes the specific case I found, but a word like
  "with" or "great" appearing in enough descriptions could cause the same
  kind of noise and isn't necessarily covered. A proper fix would use a
  standard stopword list (e.g. from `nltk` or `sklearn`) instead of one I
  typed by hand while testing one query.
- **Only `search_listings` is on MCP, by design** (the assignment's
  Milestone 1 explicitly says move one tool, not three), but that means
  `suggest_outfit` and `create_fit_card` still aren't reachable by any other
  agent — they're the two tools that actually call the model, which is
  arguably where a published, reusable tool contract would matter most.
- **`ModelUnavailable` was only tested against one failure mode** — a
  syntactically-present-but-rejected API key. I didn't test what happens
  under a real network outage or a model name that stops resolving
  mid-session; `generate.py`'s `_explain()` has branches for those cases, but
  I only exercised the "bad key" branch, deliberately, since it's the one I
  could trigger without needing a flaky network to fail on cue.

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
