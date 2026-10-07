# Acceptance criteria — FitFindr

Five criteria that say what "working" means for this agent, written in unit 3
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"The agent handles errors"* is an opinion.
*"When search returns nothing, the agent stops before calling the second tool,
in 5 of 5 tries"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter one. A reason that says something about your tools, your loop, or the
data earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

**Two are written for you. You write three.**

---

## 1. A matching query completes all three tools

Given a query that matches at least one listing, the agent completes all three
tool calls and returns a fit card — in at least 4 of 5 tries.

**Why this target:**
My search is a plain keyword-overlap match against title, description, and
style_tags — there's no synonym handling and no fuzzy matching. A phrasing
that uses a word the listing data doesn't ("jumper" instead of "sweatshirt")
can legitimately score zero even though a human would call it a match. That's
a property of the search, not a loop bug, so I don't want a criterion that
punishes the loop for it. 4 of 5 leaves room for a genuinely hard phrasing
without hiding a real regression.

---

## 2. An impossible query stops before the second tool

Given a query that matches no listings, the agent stops before calling
`suggest_outfit` and returns a message naming what to change — 5 of 5 tries.

**Why this target:**
This path never touches the model at all — `search_listings` is pure keyword
matching against static JSON, so the same impossible query returns the same
empty list every single time. There's no randomness anywhere in this branch
for 1 of 5 tries to differ on, so anything less than 5 of 5 would mean the
branch itself is broken, not that the test is strict.

---

## 3. Something about state

`session["selected_item"]` is the exact listing dict passed into `suggest_outfit()` —
same `id`, same `title`, same `price` — in 5 of 5 tries on a matching query.

**Why this target:**
`run_agent` assigns `session["selected_item"] = results[0]` and then passes
that same session field straight into `suggest_outfit()` — there's no copy,
no re-fetch, no second lookup in between. Nothing in that path depends on the
model or on randomness, so if this ever drifted it would mean the loop
itself is broken, not that an edge case slipped through. That's worth 5 of 5,
not 4 of 5.

---

## 4. Something about the fit card

At least 4 of 5 fit cards generated for the same item mention that item's
exact price (e.g. `$18.00` or `$18`) somewhere in the text.

**Why this target:**
`create_fit_card`'s prompt explicitly instructs the model to mention the
price once, but the model is still free to phrase it in words ("eighteen
dollars") instead of digits on an off try, and temperature is 0.9 on purpose
so the cards aren't identical. 4 of 5 catches a prompt that's actually broken
(never mentioning price) without failing the criterion over one stylistic
phrasing choice.

---

## 5. Your choice

With an empty wardrobe, `suggest_outfit` returns a non-empty string of
general styling advice (never an empty string, `None`, or an exception) — in
5 of 5 tries.

**Why this target:**
The empty-wardrobe branch in `suggest_outfit` is a plain `if not items:` check
that picks a different prompt — it doesn't depend on what the model says back,
only on whether `generate()` returns something. Since the branch itself is
deterministic and doesn't touch model randomness for its *shape*, only its
wording, there's no reason to accept failures here. A new user with nothing
saved is the very first thing a real user of this tool would be, so this is
also the path I'd be most embarrassed to see break.



---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 4 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 4. Something about the fit card

         The fit card is different every time.

         **Why this target:** ...

         > **Revised in unit 4:** For 5 different items, the 5 fit cards share
         > no opening sentence.
         >
         > **Why revised:** "different" wasn't checkable — two cards that
         > differed by one word still counted. The new version is something I
         > can actually score.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said the empty search stops it 5 of 5 times, but I got 3 of 5,
            so 3 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
