# Final -- r4 QA synthesis, Health Code Decoder (2026-09-02)

Scope: a single QA round on built code (see scope_receipt.md). Panel: Codex (gpt-5.6-sol), Cursor
(grok-4.6), Claude Sonnet (in-session subagent; see judgment.md for why not `claude -p`). Driver and
sole judge: Claude. Every adopted change below was verified in a browser against live CARTOS before
and after, on localhost and on the public GitHub Pages origin.

## State of the build after this round

`index.html` (commit 702839a and later) now holds:

- **Classification.** ICD-10-CM accepts every letter (`U07.1` decodes). LOINC-shaped input never
  becomes an `$expand` filter under any category chip; category narrowing clears the LOINC
  validate-code fallback unless Lab tests is selected. Row clicks carry their system into the URL
  (`&sys=`) and LOINC row clicks use the lab-list fallback.
- **Transport.** `fhir()` races complete responses (headers plus parsed body). A duplicate is sent
  after 8 s of silence; the call fails only when every started attempt fails or the 45 s ceiling
  fires; the loser is cancelled after the winner is fully read. Only complete successful bodies and
  HTTP 400/404 OperationOutcomes are cached; 5xx, unreadable bodies, and 429s are not. Retry-After
  is parsed as seconds or an HTTP date, clamped to 1..600 s, default 30.
- **Orchestration.** Any explicit action cancels a pending debounce. Group results are tri-state
  (rows / empty / error) so a failed search is never reported as "no matches". Stale "Show more"
  completions are ignored. `expand()` treats a non-ValueSet body as a failed search. The empty
  vaccine list is never cached; cached entries are validated.
- **UX and accessibility.** A spoken completion tally in the `role="status"` live region. Vaccine
  results page locally in 20s. CVX status shown once, from the source's own Status property, with
  the group text explaining that rows cannot flag retired codes. Focus rings on every control.
  Pressed chips use the theme's contrasting ink. Gradient title has `@supports` and forced-colors
  fallbacks. "covid" replaces the "B12" example.
- **Legal.** SNOMED CT (Affiliate License clause 8.3.1 wording), LOINC, and RxNorm notices are
  visible paragraphs in the footer. Privacy text discloses the URL fragment. The CARTOS terms
  quotation is dated. MIT LICENSE covers code only.

## Rejected with reason

- Per-query abort of in-flight requests (Codex S2, Cursor S8): answers are cached and reused by the
  next keystroke; the debounce bounds volume; aborting would discard useful responses.
- Cutting the request counter and the connection pulse (Codex, Cursor): the counter makes the
  "be gentle" promise auditable; the pulse conveys connection state and honors reduced motion.
- Removing `autofocus` (Cursor optional): a single-purpose search page is the canonical case for it.
- Codex M9 (README "misquotes" the Cartos User Guide): MISREAD; the quotations were taken verbatim
  from the live guide on 2026-09-02, and ONC publishes the rate limit there.

## Verify-at-build checklist (open items that only live behavior can settle)

1. Watch request counts under fast typing; if stacking is observed, revisit per-query aborts.
2. Confirm CARTOS still returns `valueBoolean` for `$validate-code` `result`; the page also accepts
   the string "true".
3. Re-check contrast of `.tag` and `.code` text in both themes with a contrast checker (pressed chips
   and focus rings were fixed; tags were not measured).
4. Whether the SNOMED Affiliate License is satisfied for a globally reachable hobby page hosted in
   the US: register the free UMLS license (uts.nlm.nih.gov) so the 8.3.1 notice is true for the
   site owner, and keep the notice visible.
5. Whether the US Core lab value set includes LOINC terms carrying an EXTERNAL_COPYRIGHT_NOTICE; if
   so, per-code notices would be required for those rows. Not observed in testing.
6. CARTOS stall behavior: re-time the hedge threshold if the service's p95 changes.

## Sonnet lane (added after grounding; details in judgment.md)

Sonnet converged with Codex and Cursor on the row-click, category, debounce, focus, and notice
findings, and contributed two fixes the others missed, both adopted:

- **Light-theme contrast.** The vocabulary colors used for pressed chips and for `.tag` / `.code`
  text failed WCAG AA in the light theme. All six light-theme values are now darkened and measured:
  5.4-7.6:1 as white-on-fill, 5.4-6.1:1 as text on their 14% tint.
- **Concurrent duplicate requests.** Identical URLs requested before the first completes (rapid
  chip clicks, the vaccine list) were fetched twice. `fhir()` now coalesces in-flight requests by
  URL.

Also adopted from Sonnet: README now states the digit-length gating for bare numbers, and the
example row carries `role="group"` with a label. Its proposal to cut the social-preview tags was
based on the PNG being absent from the text-only snapshot; the image is deployed and returns 200.

## Actual agent calls this round

3 reviewer reads (codex via `codex exec`, cursor via `agent -p`, claude via an in-session Sonnet
subagent) plus the driver anchor. Rounds r1-r3 were not run (scope_receipt.md).
