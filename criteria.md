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
<!-- Why 4 of 5 and not 5 of 5? Something about your search, probably —
     "my search is a plain keyword match and some phrasings will miss" is a
     real answer. -->

---

## 2. An impossible query stops before the second tool

Given a query that matches no listings, the agent stops before calling
`suggest_outfit` and returns a message naming what to change — 5 of 5 tries.

**Why this target:**
<!-- Why is 5 of 5 reasonable here when criterion 1 isn't? What's different
     about this path? -->

---

## 3. Outfits don't repeat a category

Given a query that matches at least one listing, the outfit `suggest_outfit`
returns never pairs two items from the same category — except tops, which may
layer — in 5 of 5 tries.

**Why this target:** This is close to common sense: pairing two bottoms, two
pairs of shoes, or two outerwear pieces in one outfit doesn't make sense, so
it should hold every single time, not most of the time. Tops are the one
exception, since layering two tops (a tank under a cropped hoodie, say) is
normal styling advice, not a mistake — so the rule only applies to bottoms,
shoes, outerwear, and accessories.

---

## 4. The fit card always shows the price

Given a query that matches at least one listing, the price of the new item
appears in the `create_fit_card` output, in 5 of 5 tries.

**Why this target:** The price is a requirement of the software, not an
optional nicety — a fit card that leaves it out failed at its job, so this
should hold every time rather than most of the time.

---

## 5. The full run stays under 5 seconds

A full run of all three tools (`search_listings`, `suggest_outfit`,
`create_fit_card`) completes in under 5 seconds, in 5 of 5 tries.

**Why this target:** Five seconds is roughly the max a user should have to
wait for an outfit suggestion before it stops feeling responsive. I'm keeping
5 of 5 for now, though I know this target could end up too strict —
`generate.py` deliberately pauses to respect rate limits, and that pacing
could push a run past 5 seconds for a reason that has nothing to do with the
agent itself being slow.



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
