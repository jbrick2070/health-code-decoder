# QA brief: Health Code Decoder (static page on ONC CARTOS)

This is a QA review of a BUILT artifact, not a plan. The code under review is real and in this
repo. Read the files; do not review this brief in the abstract.

## What to read

| File | What it is |
|---|---|
| `index.html` | The whole product: markup, CSS, and one inline `<script>` (vanilla JS, no dependencies). |
| `README.md` | Endpoints used, verified CARTOS quirks, terms and license summary. |
| `LICENSE` | MIT, code only. |
| `docs/codex-review.md` | An earlier design review whose recommendations were adopted. |
| `docs/codex-brief.md` | The verified facts about CARTOS that the design rests on. |

Live copy: https://jbrick2070.github.io/health-code-decoder/ (GitHub Pages, main branch root).

## What the page does

A single search box. If the input looks like a code (ICD-10-CM such as `E11.9`, LOINC such as
`2345-7`, or a bare number tried as CVX, RxNorm, and SNOMED CT), it decodes it with
`CodeSystem/$lookup` and shows an "Exact match" card per vocabulary that answers. Otherwise it
word-searches five groups: four HL7 US Core value sets via `ValueSet/$expand?filter=` (conditions,
medications, lab tests, procedures) plus a locally cached CDC CVX vaccine list. Category chips
narrow the search and resolve ambiguous numbers. Everything is fetched live from
`https://cartos.healthit.gov/TerminologyServer/R4` in the visitor's browser; there is no backend.

## Audience and constraints

- Audience: the general public and people curious about health IT. Not clinicians at work.
- Must stay within the CARTOS terms of use (see README "Terms, licenses, and disclaimers"):
  discovery/education use, no bulk harvesting, honor HTTP 429 Retry-After, at most modest request
  volume per visitor.
- Must carry the vocabulary notices each steward requires (SNOMED CT Affiliate License clause
  8.3.1 text; LOINC copyright notice; NLM RxNorm courtesy statement) and must not imply
  endorsement by ONC, NLM, CDC, SNOMED International, Regenstrief, or HL7.
- Must not read as medical advice or as a production clinical tool.

## Known CARTOS behavior the code works around (verified live 2026-09-02)

1. `CodeSystem/$lookup` for LOINC codes returns HTTP 400 "Term was not found" even though LOINC
   2.82 is loaded. The page falls back to `ValueSet/$validate-code` against the US Core lab value
   set, which answers in well under a second with the display name.
2. A hyphenated code used as an `$expand` `filter` takes about 13 seconds. The page never does it.
3. A lookup miss is HTTP 400 with an OperationOutcome body. The page treats that as a stable
   "not found", not a transport error, and caches it for the visit.
4. CARTOS intermittently stalls a random request for a minute or more while an identical repeat
   answers in under a second. The page sends one duplicate after 8 s of silence, takes the first
   answer, cancels only the loser, and gives up at 45 s with a plain message.
5. CORS is `Access-Control-Allow-Origin: *`, GET only.
6. ConceptMap count is zero; there are no crosswalks.

## Focus for this review

1. **Correctness of the inline JavaScript.** Especially `fhir()` (hedged fetch, AbortController
   use, the 45 s ceiling, what gets cached and when), `runQuery()` (stale-response guards via
   `seq`, the `lookupsDone` gate that prevents duplicate exact cards, the exact-card promotion from
   list results), `classify()` (pattern detection and category narrowing), and the row-click path
   (`runQuery(code, { system })`). Look for unhandled promise rejections, races, cache poisoning,
   and any path that shows the wrong thing or nothing.
2. **UX for a general-public audience.** Wording, empty states, error states, ordering, the
   "not found is inconclusive" message, mobile layout.
3. **Accessibility.** Keyboard focus visibility (note `all: unset` on result-row buttons), color
   contrast of chips and tags in both light and dark themes, aria attributes, live regions.
4. **Legal and terms.** Do the footer notices satisfy the CARTOS terms and the SNOMED CT, LOINC,
   and RxNorm requirements as summarized in README? Is anything missing, misplaced (for example
   collapsed inside `<details>` when a license expects it visible), or overstated?
5. **Anything that can be cut** without losing the goal.

Be specific and checkable: cite the function or CSS selector, describe the failing input or
scenario, and propose the smallest fix.
