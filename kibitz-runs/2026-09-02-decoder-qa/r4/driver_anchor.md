DRIVER ANCHOR REVIEW (Claude, driver) -- round r4 (residual defects), QA of built code
Grounding: read index.html, README.md, LICENSE, docs/*.md in D:\cartos directly. Every claim below
is labeled CONFIRMED (seen in the file), MISREAD (n/a for own review), or UNVERIFIABLE.

VERDICT: yes-with-fixes. The page works end to end on the live origin (verified in a browser
against cartos.healthit.gov: word search, ICD-10-CM decode, ambiguous number, LOINC fallback,
paging, row click). What remains are a handful of real defects in edge paths and accessibility,
plus one placement question on the legal notices.

MUST-FIX BEFORE BUILD:
1. [runQuery, row-click path] CONFIRMED. Clicking a LOINC row from the lab-test list runs only
   `lookup(loinc, code)`, which CARTOS answers with HTTP 400 "Term was not found". The
   `plan.vsCheck` fallback is skipped when `opts.system` is set (`if (plan.vsCheck && !opts.system)`).
   Result: the user clicks a code the page just displayed and is told CARTOS has no match for it.
   Fix: when `opts.system === SYSTEMS.loinc.url`, keep (or set) `plan.vsCheck` for the labs value
   set; or render the card from the row's own display and enrich it if lookup answers.
2. [fhir(), body parse] CONFIRMED. If `res.ok` but `res.json()` fails (HTML error page, truncated
   body, or the 45 s ceiling firing mid-body), `body` is `null`, is stored in `mem`, and every
   later identical query in the visit silently returns "no matches". Fix: if `body === null`,
   throw a transport error and do not cache.
3. [CSS .chip[aria-pressed="true"]] CONFIRMED. Pressed chips use `color: #fff` on `var(--cat)`.
   In the dark theme the category colors are deliberately light (for example `--sys-rxnorm:
   #6ee7a0`), so white-on-light fails contrast. Fix: `color: var(--accent-ink)`, which is dark in
   the dark theme and white in the light theme.
4. [CSS ul.results li button.row] CONFIRMED. `all: unset` removes the focus outline; the only
   focus-visible style is a 7 percent background tint, which keyboard users will not see. Fix:
   add `outline: 2px solid var(--accent); outline-offset: -2px` on `:focus-visible`.

SHOULD-FIX:
1. [fhir(), catch path] CONFIRMED. If the first request rejects with a network error after the
   hedge has started, `Promise.race` rejects immediately and the in-flight hedge is left running;
   its result is discarded. Abort both controllers in the catch path (the 45 s ceiling already
   does this for the timeout case).
2. [footer notices] UNVERIFIABLE (license interpretation). The SNOMED, LOINC, and RxNorm notices
   sit inside collapsed `<details>`. The LOINC license wants the notice "accessible on the same
   Internet page"; a collapsed disclosure is on the page but not visible without a click. The
   safest reading is to render the three steward notices as visible text, compact but not hidden.
3. [SNOMED notice, clause 8.3.2] CONFIRMED partial. The Affiliate License also asks that media
   "specify the version and date of the International Release". The page shows the SNOMED
   version string only on a decoded card. Consider surfacing the SNOMED CT US Edition version in
   the notices block, filled lazily from the first SNOMED result or from the CodeSystem metadata.
4. [h1 gradient text] CONFIRMED. `color: transparent` with `background-clip: text` renders
   invisible text in any engine lacking support. Wrap in `@supports (background-clip: text) or
   (-webkit-background-clip: text)` with a solid-color fallback outside it.
5. [renderExact] CONFIRMED. `hit.sys.about` is appended for every card, including synthesized
   LOINC cards; fine. But the copy button text says "Select and copy manually" on clipboard
   failure and never offers a selectable field. Minor: leave as is or select the display text.

OPTIONAL / NICE-TO-HAVE:
- Show the per-group "Showing N of M" note only once instead of re-adding after each page load
  (currently added once at fill time; fine).
- A keyboard shortcut ("/" focuses the search box).

CUT THESE:
- None. The page is already lean: one file, no dependencies.

VERIFY-AT-BUILD checklist:
- After fix 1, click a LOINC row from a "glucose" search and confirm an Exact match card renders.
- After fix 2, simulate a non-JSON 200 (block the response in devtools) and confirm the error
  message appears and a retry works in the same visit.
- After fix 3, check pressed-chip contrast in both themes (target 4.5:1).
- After fix 4, Tab through result rows and confirm a visible focus ring.
- Confirm no "Uncaught (in promise)" messages appear when a hedge loses the race (the tagged
  promises are handled via Promise.race, so none are expected).
