# Judgment -- r4 QA of the Health Code Decoder (driver: Claude)

Every claim below was checked against D:\cartos\index.html as it stood at fan-out (SHA-256 of the
reviewed snapshot: 13bbf38efe7e12b07c54dad6...). Labels: CONFIRMED (real, fixed unless noted),
MISREAD (does not match the code or the verified facts), UNVERIFIABLE (license interpretation or
live-server behavior; recorded as verify-at-build), REJECTED (confirmed but not adopted, with why).

## Codex (gpt-5.6-sol, reasoning high) -- VERDICT "no"

MUST-FIX
1. CVX contradictory status (In use: Yes + Status: Inactive), rows never flag retired CVX. CONFIRMED.
   Fixed: when the source publishes its own Status property the generic inactive row is dropped and
   the Status value is explained. Rows: the CVX CodeSystem concept list carries no properties (verified
   earlier today: `props []`), so rows cannot flag retired codes; the group text now says so.
2. Category narrowing leaks LOINC vsCheck into other categories. CONFIRMED. Fixed: vsCheck cleared
   unless the category is Lab tests. Verified: `2345-7` under Medications renders no LOINC card.
3. Hedge races headers, not complete responses; stalled body can be abandoned and null cached.
   CONFIRMED (also in the driver anchor). Fixed: each attempt is fetch AND parse; the first complete
   response wins; null bodies throw and are never cached; the 45 s ceiling aborts all attempts.
4. Every non-429 OperationOutcome cached, including 5xx. CONFIRMED. Fixed: only HTTP 400/404
   OperationOutcomes are cached as a stable miss; other non-2xx throw.
5. Escape / example / category actions do not cancel a pending debounce. CONFIRMED. Fixed:
   runQuery() and clear() both cancel the timer.
6. ICD regex excludes U (U07.1). CONFIRMED (Cursor found it too). Fixed: `[A-Z]`. Verified: U07.1
   decodes to "COVID-19".
7. Pressed-chip contrast and missing focus ring on result rows. CONFIRMED (also in the anchor).
   Fixed: pressed chips use var(--accent-ink); explicit :focus-visible outlines on rows, chips,
   copy, more, search button, and summaries.
8. Failed group reported under "No matches in ...". CONFIRMED. Fixed: tri-state group result
   (true / false / 'error'); only genuinely empty groups are listed.
9. README "misquotes" the Cartos User Guide and the production-use prohibition "is absent from the
   current terms". MISREAD. Both sentences were extracted verbatim from the live user guide on
   2026-09-02 ("Production clinical applications must not rely on the Cartos API for direct runtime
   terminology queries"; "intended for terminology discovery, design-time queries and validation,
   research, and periodic retrieval of terminology content"), and ONC publishes the 2,000 per
   5-minute limit in the same guide. Codex's read-only sandbox has no network, so it could not check.
   Kept; the footer now dates the quotation ("as of September 2, 2026").

SHOULD-FIX
1. Stale "Show more" lacks a seq guard. CONFIRMED. Fixed: fillGroup captures the query sequence.
2. Stale queries do not abort in-flight requests. CONFIRMED, REJECTED for v1: results are cached and
   reused, volume is bounded by the 400 ms debounce, and per-query aborts would discard answers the
   next keystroke often needs. Verify-at-build: watch request counts under fast typing.
3. Retry-After HTTP-date -> NaN. CONFIRMED. Fixed: parseRetryAfter() handles seconds and dates,
   falls back to 30 s, clamps to 1..600.
4. Footer overstates that a version is shown "on each result". CONFIRMED. Fixed: wording now says
   "on each decoded result"; expand() also keeps contains[].version when CARTOS sends it.
5. CVX local search capped at 50 with no paging. CONFIRMED. Fixed: local paging in pages of 20
   with CVX-specific wording.
6. Live region cleared silently on success. CONFIRMED. Fixed: a one-line tally is announced
   ("Found 13 conditions and diagnoses, 107 medications, ...").
7. Privacy wording ignores the URL fragment. CONFIRMED. Fixed: footer discloses it and advises
   against typing identifying information.
8. CVX localStorage entry not validated. CONFIRMED. Fixed: entries must be non-empty arrays of
   {code, display} strings.

CUT
- Request/cache counter: REJECTED. It makes the "be gentle" promise auditable by the visitor.
- Pulse animation: REJECTED. It carries connection state, is disabled under reduced motion, and
  costs nothing.

## Cursor (cursor-grok-4.6-high, ask mode) -- VERDICT "yes-with-fixes"

MUST-FIX
1. ICD regex excludes U. CONFIRMED, fixed (see Codex 6).
2. Category chip turns a LOINC code into a hyphenated $expand filter (13 s stall). CONFIRMED, a real
   regression from category narrowing. Fixed: LOINC-shaped input never gets an $expand group.
   Verified: `2345-7` under Lab tests makes exactly 2 requests (lookup + validate-code).
3. Hedge: (a) first fetch rejecting while the hedge is in flight fails the whole call. CONFIRMED,
   fixed: the call now fails only when every started attempt has failed. (b) aborting the loser
   causes an unhandled rejection. MISREAD: a Node replica of the exact pattern (scratch
   hedge_test.mjs) ran three win/lose scenarios with zero unhandledRejection events, because
   Promise.race attaches handlers to both branches. Moot after the rewrite anyway. (c) a loser's
   AbortError could be mapped to the 45 s message. MISREAD: the loser is aborted only after the race
   has settled. The rewrite maps to the timeout message only when the ceiling actually fired.
4. Empty CVX list persisted 7 days. PARTIAL: the read path already rejected empty arrays on reload,
   so "7 days" is a MISREAD, but an in-visit empty list was CONFIRMED. Fixed: empty lists throw and
   are never cached.
5. Pressed-chip contrast. CONFIRMED, fixed (see Codex 7).
6. No focus rings on chips, copy, more, search button. PARTIAL: those controls kept the browser's
   default outline (only .row used all: unset), but explicit rings are cheap and consistent. Fixed.

SHOULD-FIX
1. Notices collapsed in <details>. UNVERIFIABLE as license interpretation, but the conservative fix
   is cheap. Fixed: the SNOMED, LOINC, and RxNorm notices are now visible paragraphs; only the
   how-to-license prose stays collapsed.
2. 5xx OperationOutcome cached. CONFIRMED, fixed (see Codex 4).
3. expand() turns a cached miss into "no matches". CONFIRMED. Fixed: non-ValueSet bodies throw and
   the group shows "could not search".
4. validateInVs ignores valueString "true". CONFIRMED (CARTOS sends valueBoolean, verified live);
   accepted as a cheap guard.
5. Hash lacks the system after a row click. CONFIRMED. Fixed: `&sys=<key>` written and honored on
   load. Verified: `#q=2345-7&sys=loinc` renders the card directly.
6. Example chip "B12" fights the classifier. CONFIRMED. Fixed: replaced with "covid", which shows
   conditions, medications, procedures, and vaccines at once.
7. Success clears the live region. CONFIRMED, fixed (see Codex S6).
8. In-flight requests not aborted on new queries. CONFIRMED, REJECTED (see Codex S2).
9. CVX rows hard-code inactive: false. CONFIRMED; rows cannot know (no properties in the concept
   list). Documented in the group text.

OPTIONAL: forced-colors fallback for the gradient h1. Accepted (also @supports fallback). Removing
autofocus: REJECTED, a single-purpose search page is the classic case for it.

CUT: counter and pulse REJECTED (see Codex). "B12" chip: ACCEPTED.

## Claude lane (Sonnet)

`claude -p` cannot run inside a Claude Code session, so per the user's instruction the Sonnet
opinion came from an in-session Sonnet subagent reading a frozen snapshot of the same files. Its
review is saved as claude.md. VERDICT "yes-with-fixes".

MUST-FIX
1. LOINC row click shows a false "not found". CONFIRMED (anchor M1), fixed.
2. Category chip turns a LOINC code into a hyphenated $expand. CONFIRMED (Cursor M2), fixed. The
   extra point that a category also fires $expand for ICD-shaped and long-numeric codes: CONFIRMED
   but REJECTED as a defect. Those filters are not hyphenated, answer fast, and act as a fallback
   that finds the code in the list when the direct lookup misses; the list is hidden when it only
   repeats the exact card.
3. Escape does not cancel the pending debounce. CONFIRMED (Codex M5), fixed.
4. Light-theme contrast: LOINC, RxNorm, and CVX colors fail 4.5:1 as white-on-fill and as text on
   the 14% tint. CONFIRMED and the only finding unique to this lane. Measured with a WCAG
   luminance script: the old values scored 3.3-3.8 on white and 2.8-3.2 on tint; ICD-10-CM blue
   also fell to 4.24 on tint, which Sonnet did not catch. Fixed: all six light-theme vocabulary
   colors darkened (now 5.4-7.6 on white, 5.4-6.1 on tint). Dark theme was already passing.
5. Invisible row focus. CONFIRMED (anchor M4), fixed.
6. Notices collapsed. Fixed as visible paragraphs (see Cursor S1).

SHOULD-FIX
1. getCvx() issues duplicate requests when called concurrently. CONFIRMED and generalizable: any
   identical URL requested twice before the first completes was fetched twice. Fixed: fhir() now
   coalesces identical in-flight URLs, which covers CVX and every other request.
2. CVX 50-item cap without paging. CONFIRMED (Codex S5), fixed.
3. ICD regex excludes U. CONFIRMED, fixed.
4. README overstates bare-number decoding (code gates by length). CONFIRMED. Fixed: README now
   states the length ranges.
5. #examples lacks role="group" and a label. CONFIRMED, fixed.
6. Copy confirmation not announced. CONFIRMED, already fixed in the rewrite (setStatus on copy).

CUT
- og:image tags because social-card.png "is not present in this snapshot". MISREAD in context: the
  snapshot given to the reviewer held text files only; the PNG is in the repo and returned HTTP 200
  from the live site earlier today. Kept.
- Nothing else; Sonnet explicitly declined to cut the counter, the about disclosure, or raw FHIR,
  agreeing with the driver against Codex and Cursor on the counter and pulse.
