# Handoff — Matt & Louise Prep Book

## Session Summary
Two things happened. First, a reported bug in the Walmart list turned out not to be a bug — the list already contains exactly the ingredients of the selected recipes, verified live. Second, the real issue surfaced: closing Safari (or just closing a recipe) reset the session. Session persistence is now implemented and verified across reloads.

## What Changed
- **File:** `index.html`
  - **What changed:** All session state moved to one versioned localStorage blob (`prep-state-v1`) that now also covers checked recipe steps, the active filter chip, and which recipe was open; storage failures surface a visible banner instead of being swallowed; added a manifest link and iOS web-app meta tags.
  - **Why:** Checked steps and the filter were never persisted at all, and the old `try{}catch(e){}` hid Safari storage failures silently.
- **File:** `manifest.webmanifest` (new)
  - **What changed:** Web app manifest — standalone display, scoped to `./`.
  - **Why:** Installed (Add to Home Screen) apps are exempt from Safari's ~7-day purge of script-written storage. This is the only durable fix for a weekly-use site.
- **File:** `icon.svg` (new)
  - **What changed:** 512×512 app icon, masking-tape motif matching the site.
  - **Why:** Home-screen icon for the installed app.
- **File:** `.claude/launch.json` (new)
  - **What changed:** Static server config on port **8811** (`python3 -m http.server`).
  - **Why:** localStorage needs a real HTTP origin; `file://` and the preview pane's `data:` snapshot don't provide one. **Port 8765 was already taken by another Python process** — don't reuse it.
- **File:** `HANDOFF.md`
  - **What changed:** This document.

Nothing is committed. Last commit is still `136697d`.

## Decisions Made
- **Persist step checkmarks keyed `recipeId:stepIndex`** (e.g. `m6:3`). Chose index-based keys over hashing step text so editing a step's wording doesn't silently orphan progress; the trade-off is that **reordering or inserting a step shifts saved checkmarks** on that recipe.
- **Only top-level steps are checkable**, matching the pre-existing behavior. Sub-steps inside grouped items stay non-interactive — clicking one checks its parent step, because `closest("ol.steps>li")` walks up to the top-level `li`. Left as-is; making sub-steps individually checkable is a separate feature.
- **Reopen the in-progress recipe on load, but only when at least one of its steps is checked.** Reason: mid-cook you want to land back on the recipe; on a casual visit an auto-opening modal would be obnoxious. The guard gives both.
- **Added a "Start over" control in the dialog footer.** Once steps persist, stale checkmarks would otherwise greet you next week with everything already done.
- **Clearing the week (or loading a preset) also clears step progress.** A new week's plan means a fresh cook.
- **Surface storage failure instead of swallowing it.** Silent failure was the worst property of the old code — the user had no way to know why things reset.
- **Rejected: a service worker / Cache API.** Overkill; the problem was state durability, not offline loading. The manifest alone buys the storage exemption.
- **Rejected: cookies as a localStorage fallback.** Safari's 7-day cap applies to script-set cookies too, so it buys nothing.
- **Rejected: changing the Walmart list.** It was already correct (see below).

## Current State
**Working — verified on `http://localhost:8811` with real reloads:**
- Full round-trip survives reload: week picks, shopping checkmarks, step checkmarks, filter chip, and the open recipe. Test run restored `week=[b5,m8,m11]`, `filter=breakfast`, `steps=[m8:0,m8:1]`, and reopened "Turkey Meatballs Marinara" at "2 of 8 steps done".
- Pre-v1 `prep-week` / `prep-got` keys migrate on first load, so existing state isn't lost.
- Seeded default week (Week 1) is now written to storage on first boot.
- Blocked-storage path verified by stubbing `Storage.prototype.setItem` to throw: flips to the red banner, no crash.
- Install hint shows once, dismissible via "Got it" (`prep-installed-hint`), auto-hidden when running standalone.
- `manifest.webmanifest` and `icon.svg` both serve 200 with correct content types. No console errors.
- Regression checks pass: Walmart list still correctly scoped (`extra:[]`, `missing:[]`); `b1` macros unchanged at `[627,42,44,31]`.

**The original Walmart-list report — closed, no code change needed.** With Week 1 picked, `buildList()` returns 31 items for the 31 distinct ingredients of those 3 recipes: zero extras, zero missing. Quantities combine correctly (`turkey` 3 lb + 3 lb → 6 lb). Also checked and clean: no recipe carries ingredients its steps don't use (the one apparent hit, `m6`/`soy`, was a false positive from a >3-character word filter), and no duplicate lines from unit mismatches. The likely perceptual cause was the 16-line **"Pantry (check before you buy)"** group at the bottom of the list. **That question was never answered** — see Open Questions.

**Known-incomplete:**
- `got` (checked shopping items) is still only cleared by "Clear week", "Uncheck all", and preset load — not by `toggle()`. Remove one recipe and add another and stale checkmarks persist on items that are still in the list.
- Not persisted, by choice: scroll position, open `<details>` in Basics.

## Next Steps
1. **Test on the actual device.** Open the GitHub Pages URL in Safari, Share → Add to Home Screen, check some steps, force-quit Safari, reopen. This is the only test that proves the ITP fix; localhost can't.
2. **Deploy.** Nothing is committed or pushed yet, and the site is GitHub Pages off `hewy-nova/prep-book`, so none of this is live. Commit the five paths and push to `main`.
3. **Answer the pantry question** (Open Questions) and act on it, or close it out.
4. **Prune stale `got` keys** in `buildList()` or `toggle()` so removing a recipe drops its orphaned shopping checkmarks.

## Open Questions
- **The pantry group.** Still unanswered: collapse "Pantry (check before you buy)" behind a `<details>` toggle, drop it from the shopping list entirely, or leave it? If dropping it, filter group index 6 in `renderWeek()` and `listText()` **only** — `buildList()` must keep collecting pantry items, because `C[k][3]` also drives the recipe dialog's `· pantry` tag and `share()`'s portion math.
- **Latent bug, not yet fixed:** a non-pantry ingredient used in two different units across two recipes would render as two separate lines, since non-pantry keys are `k+"|"+u`. No current recipe pair triggers it. Fix would normalize through the existing `UCONV` map (`{tsp:1/3, tbsp:1, cup:16, cups:16}`) before keying. Harden now or wait for a new recipe to hit it?
- **Step-key fragility:** if recipes get edited often, consider whether index-based step keys are the right trade-off.

## Context the Next Session Needs
- **One file, no build step, no dependencies.** `index.html` is ~780 lines. GitHub Pages serves it directly from `hewy-nova/prep-book`, so local == deployed once pushed.
- **Data model:** `R` = recipes (`b1`–`b5` breakfast, `m1`–`m16` mains); `C` = ingredient key → `[label, walmartQuery, aisleIndex, isPantry?]`; `NUT` = per-unit nutrition; `AISLES` = 7 names, **index 6 is pantry**; `PRESETS` = 8-week rotation. `r.ings` entries are `[key, qty, unit, displayText]`.
- **Storage keys:** `prep-state-v1` (the whole session blob: `week`, `got`, `steps`, `filter`, `open`, `savedAt`), `prep-installed-hint` (banner dismissal). Legacy `prep-week` / `prep-got` are read once for migration and never written again.
- **Gotcha — you cannot test persistence in the preview pane.** A `file://` URL outside the project renders as a `data:` snapshot with no usable storage origin. Use the `prep-book` launch config (port 8811) and drive `tab-2`; `javascript_tool` against the live page is how all of this was verified.
- **Gotcha — `<dialog>`'s `close` event is queued, not synchronous.** Right after `dlg.close()`, `open_` still holds the old id; it clears on the next tick. Cost 5 minutes of false alarm. Assert after `await new Promise(r=>setTimeout(r,50))`.
- **How to re-verify list scoping in one call** (via `javascript_tool`):
  ```js
  const sel=[...week].map(byId), need=new Set();
  sel.forEach(r=>r.ings.forEach(([k])=>need.add(k)));
  const shown=[]; buildList().forEach(g=>g.forEach(it=>shown.push(it.k)));
  ({extra:shown.filter(k=>!need.has(k)), missing:[...need].filter(k=>!shown.includes(k))})
  ```
  Both empty == correct.
- **Non-obvious coupling:** `C[k][3]` (pantry flag) is read in three unrelated places — `buildList()` aisle routing, the dialog's `· pantry` tag, and `share()`'s you/Louise portion split (pantry items and meats use 0.12/0.08). Flipping it on an ingredient silently changes displayed macros.
- **Safari storage rules that drove the design:** script-written storage in a normal tab is purged after ~7 days without user interaction with the site; storage in an installed Add-to-Home-Screen app is exempt. Private browsing allows reads but throws on write — which is exactly why `save()` now detects the flip and shows the banner.
