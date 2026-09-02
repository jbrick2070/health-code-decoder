(claude lane: Claude Sonnet, run as an in-session subagent against a frozen snapshot of the repo, because `claude -p` cannot start inside a Claude Code session. Snapshot index.html SHA-256 13bbf38efe7e12b07c54dad6...)

VERDICT: yes-with-fixes. The architecture correctly implements the documented CARTOS workarounds (hedged fetch, 45s ceiling, LOINC $validate-code fallback, stable-cache of "not found" OperationOutcomes) in the common path, but three concrete JavaScript defects break or regress that architecture on specific inputs, plus one accessibility contrast failure and one legal-notice-visibility risk that the brief explicitly asks about. None require a redesign; all have small, localized fixes.

MUST-FIX BEFORE BUILD:

1. [runQuery(), row-click path] Clicking a result row for any LOINC-system item always shows a false "not found." renderRow()'s onclick calls runQuery(item.code, { system: item.system }); the opts.system block sets plan.lookups to that system only and plan.groups = [], and the guard `if (plan.vsCheck && !opts.system)` skips the $validate-code fallback because opts.system is truthy. Since direct CodeSystem/$lookup for LOINC always returns HTTP 400, lookup() resolves null, exactHits stays 0, and the status says "CARTOS did not return a match" for a code the user just clicked out of a visible result.
   Fix: when opts.system === SYSTEMS.loinc.url, keep the vsCheck fallback active.

2. [classify()] Selecting any category chip re-introduces the documented 13-second $expand-with-hyphen slow path. `plan.groups = [g];` unconditionally overwrites the LOINC branch's `plan.groups = []`. Repro: click "Labs", paste 2345-7; expand(labs-vs, "2345-7") runs. The same override also fires $expand calls for ICD-shaped and long-numeric codes whenever a category is active.
   Fix: do not let the category override widen what the classifier decided is unsearchable.

3. [clear(); Escape handler] Escape does not cancel a pending debounced search. clear() bumps seq and clears the DOM but never calls clearTimeout(timer). Type "asthma", press Escape within 400 ms: the stale timer fires runQuery("asthma") and, because it does ++seq itself, becomes the current query.
   Fix: clearTimeout(timer) as the first line of clear().

4. [:root; .tag; .code; .chip[aria-pressed="true"]] Light-theme contrast failure on three of five system colors. --sys-loinc:#0d9488, --sys-rxnorm:#16a34a, --sys-cvx:#ea580c are used as white text on a solid fill (pressed chip) and as colored text on a ~14% tint near-white background (.tag, .code). Contrast against white is about loinc 3.8:1, rxnorm 3.3:1, cvx 3.6:1, all below WCAG AA 4.5:1 for normal text. icd10cm (5.2), sct (5.7), other (4.8) pass. Dark-theme equivalents pass.
   Fix: darken the three failing light-theme values and verify with a contrast tool.

5. [ul.results li button.row] Row-button keyboard focus is effectively invisible: all: unset strips the default outline and the only focus-visible rule is the same faint background as hover.
   Fix: outline: 2px solid var(--accent); outline-offset: -2px on :focus-visible.

6. [footer <details>] The three steward-required notices (SNOMED CT 8.3.1 text, LOINC copyright notice, RxNorm courtesy statement) are collapsed until clicked. [ASSUMPTION] whether the licenses require visibility without interaction is not confirmable from the snapshot, but the brief names this risk, so treat as must-fix pending verification.
   Fix: move the three blockquotes out of <details> into always-visible footer elements.

SHOULD-FIX:

1. [getCvx()] No in-flight de-duplication: concurrent calls before the first resolves each issue a live CodeSystem request. Fix: memoize the in-flight promise.
2. [fillGroup()/groupTasks] CVX results cap at 50 with no "Show more" because loadMore is null for the local group. Fix: local loadMore that slices.
3. [classify()] ICD regex excludes U (U07.1, COVID-19). Fix: [A-Z].
4. [README "Decodes codes"] Doc/code mismatch: README says a bare number is tried against all three systems; code gates by length (CVX <= 3 digits, RxNorm <= 8, SNOMED >= 6). Fix: qualify the README sentence.
5. [#examples] Missing role="group" aria-label for consistency with #cats.
6. [copyButton()] "Copied" confirmation not routed through the live region. Fix: setStatus('Copied to clipboard') or aria-live on the button.

OPTIONAL / NICE-TO-HAVE:
- Custom :focus-visible for .chip, .copy, .more button, and the submit button.
- .search input { outline: none } relies on the parent .search:focus-within glow; acceptable.
- mem cache has no cap; harmless for one visit.
- The pulsing live dot respects prefers-reduced-motion; fine as is.

CUT THESE:
1. og:image / twitter:image meta tags: social-card.png is not present in this snapshot; if absent from the deployed repo, the tags produce a broken preview. Safe to cut or add the asset.
2. Nothing else. Would not cut the visit-stats line, the "about" disclosure, or the raw-FHIR details.

VERIFY-AT-BUILD checklist:
- Confirm social-card.png exists on the deployed site.
- Confirm the SNOMED 8.3.1, LOINC, and RxNorm notice texts verbatim against each steward's current document.
- Confirm whether those terms require the notice visible without interaction.
- Re-run the contrast numbers through a contrast tool before finalizing hex values.
- Confirm live timing behavior (8 s hedge, 45 s ceiling, ~13 s hyphenated filter) against the actual service.
- Confirm mobile rendering on a small viewport.
- Confirm screen-reader behavior for the #status live region and the copy-button text under NVDA/JAWS/VoiceOver.
- Confirm the live page matches this index.html.
