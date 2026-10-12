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

FitFindr takes a plain-language shopping query — what someone's looking for,
optionally a size and a price ceiling — and searches a secondhand listings
catalog for matches. When it finds one, it pulls in the user's existing
wardrobe and asks the model to suggest an outfit pairing the new item with
pieces they already own, then writes a short, postable caption for the find.
If nothing in the catalog matches the query, it stops and tells the user
what to change — a looser price ceiling, a different size, broader keywords
— instead of guessing or crashing.

---

## Tool Inventory

<!-- Four lines per tool. This is worth 2 points and it's the single most
     common place students lose them.

     "Returns a list" earns NOTHING. The description has to say what is IN
     the list.

     The empty case isn't optional either — it's the thing your loop branches
     on, and if you don't decide it here you'll discover it as a crash in
     Milestone 5. -->

### `search_listings`

- **What it does:** Searches the 40 listings in `data/listings.json` for items matching a free-text description, optionally filtered by size and a price ceiling, and returns the best matches ranked by keyword overlap.
- **Inputs:** `description` (str, required) — keywords like `"vintage graphic tee"`. `size` (str, optional, default `None`) — matched by splitting the listing's `size` field on non-alphanumeric characters (`"S/M"` → tokens `{"s", "m"}`) and checking the query size is one of the tokens, case-insensitive — so `"M"` matches `"S/M"` but not `"US 9"` or `"XL"`. `max_price` (float, optional, default `None`) — inclusive ceiling.
- **Returns:** A list of matching listing dicts, best match first, capped at `config.SEARCH_RESULT_LIMIT`. Each dict has `id, title, description, category, style_tags (list), size, condition, price (float), colors (list), brand (str or None), platform`.
- **When it has nothing:** Returns `[]` — an empty list, never `None` and never an exception. This is what the loop branches on.

### `suggest_outfit`

- **What it does:** Calls the model to suggest one or two outfit pairings between a thrifted item and the pieces already in the user's wardrobe.
- **Inputs:** `new_item` (dict, required) — a listing dict, typically `session["selected_item"]`. `wardrobe` (dict, required) — a wardrobe dict with an `items` key holding a list of wardrobe-item dicts (`id, name, category, colors, style_tags, notes`); `items` may be `[]`.
- **Returns:** A non-empty string describing one or two outfit combinations, naming specific wardrobe pieces by name when the wardrobe is non-empty.
- **When it has nothing:** When `wardrobe["items"]` is empty, returns general styling advice for the new item alone (not an empty string, not an exception) — this is the empty-wardrobe path unit 4 triggers on purpose.

### `create_fit_card`

- **What it does:** Calls the model to write a short, postable caption for the thrifted find, using the outfit suggestion as context.
- **Inputs:** `outfit` (str, required) — the string returned by `suggest_outfit`. `new_item` (dict, required) — the listing dict for the item.
- **Returns:** A two-to-four sentence caption that mentions the item, its price, and its platform exactly once each, and varies wording across different inputs (not word-for-word identical runs — see `CACHE_ENABLED`/`TEMPERATURE` in `config.py`).
- **When it has nothing:** If `outfit` is empty or whitespace-only, returns a descriptive message ("No outfit to post yet" style) instead of raising or calling the model.

---

## Planning Loop

<!-- Your branch rule, stated as a rule — the condition AND both paths — plus
     the file and function that holds it.

     Like this:
       "If search_listings returns an empty list, put a message in the session
        and stop. Otherwise take the first result and go to suggest_outfit."
        — agent.py::run_agent

     The grader checks your code against what you claim here, so the file and
     function have to be real. -->

**Branch rule:** If `search_listings` returns an empty list, put a message in `session["error"]` naming what to change (e.g. loosen the price ceiling or size) and stop — do not call `suggest_outfit`. Otherwise, take the first result as `session["selected_item"]` and continue to `suggest_outfit`, then `create_fit_card`.

**Where it lives:** `agent.py::run_agent`

**How the query is parsed:** Regex. A price ceiling is pulled with a pattern matching `under $N` / `$N` (e.g. `r"\$(\d+(?:\.\d+)?)"`), a size token is pulled by checking the query against a fixed set of known size strings from the data, and whatever text remains after stripping both becomes `description`. No model call for parsing — it's cheap and deterministic, and the model budget is spent on `suggest_outfit` and `create_fit_card` instead.

**What moves through the session:** `query` → parsed into `parsed` (`description`, `size`, `max_price`) → fed to `search_listings`, whose output fills `search_results` → the first item becomes `selected_item` → `selected_item` + `wardrobe` go to `suggest_outfit`, filling `outfit_suggestion` → `outfit_suggestion` + `selected_item` go to `create_fit_card`, filling `fit_card`. `error` is set only on the empty-search branch, and when it's set every field after `search_results` stays `None`.

---

## Sample Run

<!-- Two things go here.

     1. One FULL query and its output, pasted as text.
     2. Your three per-tool terminal tests — the command and what it printed. -->

**One full query**

```
$ python app.py ask 'vintage graphic tee under $30'

  Found:    Y2K Baby Tee — Butterfly Print — $18.0 on depop

  Outfit:   Pair the Y2K baby tee with your baggy straight-leg jeans and chunky white sneakers
  to nail that nostalgic 2000s streetwear silhouette. Throw on your vintage black denim jacket
  over top and sling your black crossbody bag across your chest to add an effortless edge that
  grounds the pastel butterfly print. Alternatively, tuck the baby tee into your wide-leg khaki
  trousers, secure it with the brown leather belt, and wear your chunky white sneakers for a
  softer, earth-toned look.

  Fit card: Manifesting ultimate 2000s pop star energy in this pastel butterfly Y2K baby tee!
  Snagged it on Depop for just $18.00 and I am already living in it. Pair it with baggy jeans
  and chunky sneakers for the absolute easiest nostalgic fit.

  0 model calls this session, 2 served from cache
```

**The empty-search branch, proven by the call count**

```
$ python app.py ask 'designer ballgown size XXS under $5'

  No listings matched. Try raising the price ceiling, dropping the size filter, or using
  broader keywords in your description.

  0 model calls this session
```

Zero model calls on the empty path is the branch working — `suggest_outfit` and
`create_fit_card` were never reached.

**The three tools, tested one at a time**

```
$ python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"
[{'id': 'lst_002', 'title': 'Y2K Baby Tee — Butterfly Print', 'price': 18.0, ...},
 {'id': 'lst_006', 'title': 'Graphic Tee — 2003 Tour Bootleg Style', 'price': 24.0, ...},
 {'id': 'lst_017', 'title': 'Mesh Long-Sleeve Top — Black', 'price': 15.0, ...},
 {'id': 'lst_033', 'title': 'Vintage Band Tee — Faded Grey', 'price': 19.0, ...},
 {'id': 'lst_011', 'title': 'Low-Rise Cargo Pants — Khaki', 'price': 27.0, ...},
 {'id': 'lst_015', 'title': 'Vintage Graphic Hoodie — Faded Black', 'price': 26.0, ...}]
(six matches, ranked by keyword overlap — full dicts truncated here for space)
```

```
$ python -c "from tools import suggest_outfit; from utils.data_loader import get_example_wardrobe, load_listings; print(suggest_outfit(load_listings()[0], get_example_wardrobe()))"
Pair the medium wash Levi's 501s with your fitted white ribbed tank top and cinch the waist
using your brown leather belt for a classic 90s silhouette. Finish this effortless daytime
look by stepping into your chunky white sneakers and slinging your black crossbody bag over
your shoulder. For a cooler day, layer your oversized grey crewneck sweatshirt over the top
and swap the sneakers for your black combat boots to add a touch of grunge.
```

```
$ python -c "from tools import create_fit_card; from utils.data_loader import load_listings; print(create_fit_card('jeans and white sneakers', load_listings()[0]))"
Finally scored the holy grail of denim: vintage Levi's 501s with that perfectly broken-in
medium wash. At just $38, these are about to become my daily uniform paired with crisp white
sneakers. Snag them on my Depop before I change my mind and keep them forever!
```

Ran `create_fit_card` three times on the same item with `AI201_CACHE=0` to confirm it isn't
just handing back a cached answer — three different captions came back, all mentioning the
$38 price and Depop:

```
Nothing beats broken-in vintage Levi's 501s, especially when they already have that perfect
medium wash. Just dropped these in my Depop shop for $38! Throw them on with crisp white
sneakers and you're good to go.

Nothing beats worn-in vintage Levi's 501s that fit like a glove right out of the gate. Tossed
these on with crisp white sneakers and the casual look was instantly set. Grabbed them on
Depop for just $38 and I'm never taking them off.

Nothing beats real broken-in denim, especially when you score true vintage Levi's 501s for
just $38. I'm living in these with crisp white sneakers all season long. Snagged this medium
wash pair on Depop before someone else could!
```

---

## How I Used AI

<!-- Two specific moments. What you asked, what came back, what you changed.

     "I used Claude to help me code" is not enough.

     "I gave Claude my search_listings spec. It returned None on no match
     instead of an empty list, so I changed it" is the level we want. -->

**Moment 1**

- *What I asked for:* I drafted acceptance criterion 4 as "The model should
  always show the price of the outfit after picking one" and asked Claude to
  check it.
- *What came back:* Claude pointed out "the outfit" doesn't have a price
  field anywhere in the data — wardrobe items don't carry a price, only
  listings do — so the sentence didn't say which number it meant, and it
  couldn't test it without asking me first.
- *What I changed:* Reworded it to "the agent lists the price for the new
  item," and specified the check is against `create_fit_card`'s output
  specifically, since that's the one tool whose job is to produce the thing
  a person actually reads.

**Moment 2**

- *What I asked for:* My draft of criterion 3 was "never repeats a category
  for each clothes in 5 out of 5 tries."
- *What came back:* Claude asked whether layering two tops — a tank under a
  cropped hoodie, say — should count as a violation, since a strict
  same-category rule would mark normal, reasonable styling advice as a fail.
- *What I changed:* Decided tops are the one exception; only bottoms, shoes,
  outerwear, and accessories have to stay unique within one outfit. I hadn't
  considered that my own rule would penalize a legitimate suggestion until it
  was phrased back to me as a concrete case.

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

Produced by `python run_eval.py --label before` — full output in
`results/run_2026-10-11_2213_before.md`. Scenarios: `scenarios.py`; loop:
`agent.py::run_agent`; timing for criterion 5 measured separately (see below)
since `run_eval.py` doesn't record per-try duration.

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1. Matching query completes all three tools | 4 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 2. Impossible query stops before tool 2 | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 3. Outfit never repeats a non-top category | 5 of 5 | FAIL | FAIL | FAIL | FAIL | FAIL | MISSED (0/5) |
| 4. Fit card always shows the price | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |
| 5. Full run stays under 5 seconds | 5 of 5 | PASS | PASS | PASS | PASS | PASS | MET (5/5) |

**Criterion 1** — `agent.py::run_agent`, scenario `matching query completes`,
query `vintage graphic tee under $30`, try 1:

```
selected_item: Y2K Baby Tee — Butterfly Print ($18.0, depop)

Outfit suggestion:
Pair the butterfly baby tee with your baggy straight-leg dark wash jeans and chunky white
sneakers to nail that Y2K street style proportion play. Throw your black cropped zip hoodie
loosely over your shoulders and finish the look with your black crossbody bag.

Fit card:
Finally scored the ultimate Y2K butterfly baby tee for just $18 on Depop! I'm living for this
print and can't wait to style it with baggy dark-wash denim and chunky white sneakers. Grab it
on my page before I change my mind and keep it for myself!
```

**Criterion 2** — `agent.py::run_agent`, scenario `impossible query stops early`,
query `designer ballgown size XXS under $5`, try 1:

```
stopped early: yes — No listings matched. Try raising the price ceiling, dropping the size
filter, or using broader keywords in your description.
selected_item: (none)
search_results: 0
```

**Criterion 3** — `tools.py::suggest_outfit`, scenario `outfit has no repeated category`,
query `silk slip dress in midi length under $40`, try 1 — **FAIL**:

```
selected_item: 90s Silk Slip Dress — Floral, Midi Length ($30.0, depop)  [category: bottoms]

Outfit suggestion:
Layer the floral slip dress right over your baggy straight-leg jeans and white ribbed tank top
for an effortless 90s indie sleaze look. Ground the outfit by pulling on your black combat
boots and throwing the vintage black denim jacket over your shoulders. ...
```

The new item itself — the slip dress — is tagged `category: bottoms` in the data (checked via
`load_listings()`), the same category as "baggy straight-leg jeans" in the example wardrobe
(also `bottoms`). The model consistently (5 of 5 tries) suggests layering the dress over the
jeans — a real, intentional styling move — but under my criterion's literal
same-non-top-category rule, dress + jeans is a repeat every time. All 5 tries failed for this
same reason.

**Criterion 4** — `tools.py::create_fit_card`, scenario `fit card shows the price`,
query `90s track jacket in size M`, try 1:

```
selected_item: 90s Track Jacket — Navy/White Stripe ($45.0, poshmark)

Fit card:
Pulled this crispy 90s navy and white stripe track jacket and I am obsessed! Just scored it
for $45 over on Poshmark and it's the ultimate throw-on-and-go piece. Throw it on with a white
ribbed tank, baggy denim, and chunky sneakers for the easiest off-duty streetwear fit.
```

**Criterion 5** — timed separately with `time.monotonic()` around
`agent.py::run_agent`, query `denim jacket under $50`, `AI201_CACHE=0`:

```
try 1: 3.68s, error=None
try 2: 2.36s, error=None
try 3: 2.11s, error=None
try 4: 2.11s, error=None
try 5: 2.21s, error=None
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
$ python app.py ask 'vintage graphic tee under $30' --trace

[1] parse_query
      in:  dict with keys: query
      out: dict with keys: description, size, max_price
[2] search_listings (via MCP)
      in:  dict with keys: description, size, max_price
      out: 10 items: Y2K Baby Tee — Butterfly Print, Graphic Tee — 2003 Tour Bootleg Style, Vintage Band Tee — Faded Grey … +7 more
[3] suggest_outfit
      in:  dict with keys: new_item
      out: Pair the Y2K baby tee with your baggy straight-leg jeans and chunky white sneakers to nail that nostalgic 2000…
[4] create_fit_card
      in:  dict with keys: outfit
      out: Manifesting ultimate 2000s pop star energy in this pastel butterfly Y2K baby tee! Snagged it on Depop for just…

  Found:    Y2K Baby Tee — Butterfly Print — $18.0 on depop
  Outfit:   Pair the Y2K baby tee with your baggy straight-leg jeans and chunky white sneakers...
  Fit card: Manifesting ultimate 2000s pop star energy in this pastel butterfly Y2K baby tee!...

0 model calls this session, 2 served from cache
```

**Empty search**

```
$ python app.py ask 'designer ballgown size XXS under $5' --trace

[1] parse_query
      in:  dict with keys: query
      out: dict with keys: description, size, max_price
[2] search_listings (via MCP)
      in:  dict with keys: description, size, max_price
      out: [] (empty)
[3] branch
      →    empty search_results — stopping before suggest_outfit

  No listings matched. Try raising the price ceiling, dropping the size filter, or using
  broader keywords in your description.

0 model calls this session
```

Three steps instead of four — it stops one step earlier than the happy path, which
is the branch doing its job rather than just producing different-looking text.

**The three failure modes, triggered on purpose**

1. **Empty search** — `python app.py ask 'designer ballgown size XXS under $5'`.
   Message: *"No listings matched. Try raising the price ceiling, dropping the
   size filter, or using broader keywords in your description."* Already handled
   correctly from Milestone 5 — no changes needed.

2. **Empty wardrobe** — `python app.py ask 'vintage graphic tee under $30' --empty-wardrobe`.
   Returns general styling advice (e.g. pairing the tee with a denim mini and
   platform shoes) instead of crashing or returning an empty string. Already
   handled correctly from Milestone 4 — no changes needed.

3. **Model unavailable** — changed one character of `GEMINI_API_KEY` in `.env`,
   then ran a fresh, never-asked query (`'90s track jacket in size M'`), confirmed
   by `1 model calls this session` so it wasn't just a cache hit:

   ```
   $ python app.py ask '90s track jacket in size M' --trace

   [1] parse_query
         in:  dict with keys: query
         out: dict with keys: description, size, max_price
   [2] search_listings (via MCP)
         in:  dict with keys: description, size, max_price
         out: 4 items: 90s Track Jacket — Navy/White Stripe, 90s Leather Bomber — Black, ...
   [3] suggest_outfit
         in:  dict with keys: new_item
         →    ModelUnavailable — stopping

     The model rejected your API key. Check GEMINI_API_KEY in your .env file, or create a
     fresh key at aistudio.google.com.

   1 model calls this session
   ```

   Before a fix, this propagated as an uncaught `ModelUnavailable` all the way to
   `app.py`'s top-level handler — not a raw stack trace (that handler already
   catches `Exception` and prints `type(exc).__name__: exc`), but it exited the
   whole process with `sys.exit(1)`, which would have killed an interactive
   `ask` session after one bad query. Added a `try/except ModelUnavailable`
   around both `suggest_outfit` and `create_fit_card` in `run_agent()` so the
   failure now sets `session["error"]` and returns cleanly, the same pattern as
   the empty-search branch, instead of raising past the loop. Put the real key
   back in `.env` immediately after confirming this.

**A bug the trace work surfaced:** adding `trace.step()` calls made the trace
print on *every* run, not just ones passed `--trace` — `trace.py`'s `step()`
prints unconditionally, so calling it inside `run_agent()` with no guard broke
`app.py`'s documented "add `--trace` to print the loop" behavior. Fixed by
adding a `trace_enabled: bool = False` parameter to `run_agent()`, threaded
through from `app.py`'s `--trace` flag, and wrapping every `trace.step()` call
in a local `_step()` helper that checks it first.

**On the MCP move:** Moved `search_listings` onto MCP — registered it in
`mcp_server.py` with a description and typed inputs written for a reader who
can't see `tools.py`, then in `agent.py::run_agent` swapped the direct call

```python
session["search_results"] = search_listings(
    session["parsed"]["description"], size=..., max_price=...,
)
```

for

```python
session["search_results"] = call_tool("search_listings", {
    "description": session["parsed"]["description"],
    "size": session["parsed"]["size"],
    "max_price": session["parsed"]["max_price"],
})
```

Nothing behaved differently afterward. `python app.py ask 'vintage graphic tee
under $30'` produced the identical item, outfit, and fit card as the pre-MCP
run; `python app.py ask 'designer ballgown size XXS under $5'` still stopped
with the same message and 0 model calls. Checked the types explicitly too —
`session["search_results"]` came back as `list`, and `session["search_results"][0]`
and `session["selected_item"]` as `dict`, not strings — so `mcp_client.call_tool`'s
unwrapping is doing what it claims: the call changes shape, the return value
doesn't.

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
