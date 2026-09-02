VERDICT: yes-with-fixes — the artifact matches the original #2+#3 static CARTOS demo, but four code defects still produce wrong or stalled answers (ICD-10-CM `U*` codes, hyphenated `$expand` when a category chip is on, `fhir()` hedge race/unhandledrejection, empty CVX persisted 7 days), and the brief’s own a11y/legal focus items are still open.

MUST-FIX BEFORE BUILD:
1. [index.html `classify()` ~L393] ICD-10-CM regex is `/^[A-TV-Z]\d[0-9A-Z](\.?[0-9A-Z]{0,4})$/i`, which excludes `U`. Pasting `U07.1` / `U09.9` (COVID-19 / post-COVID, the most likely bill/record paste of the last several years) takes the word-search branch, not `$lookup`. Should have been caught in R1. Fix: use `[A-Z]` (ICD-10-CM uses `U`).
2. [index.html `classify()` L403–428 vs brief “Known CARTOS behavior” #2] LOINC path correctly sets `plan.groups = []` so a hyphenated code is never used as `$expand` `filter`. Then `if (cat !== 'all') { ... plan.groups = [g]; }` undoes that. Repro: set chip to “Lab tests” (or any chip), submit `2345-7` → `expand()` is called with `filter=2345-7` (~13 s stall the brief says the page never does), plus `vsCheck`. Category narrowing (Codex item folded in) introduced this regression. Fix: if `loinc.test(q)` (or `filter` matches `/^\d{1,5}-\d$/`), do not assign `plan.groups` to an `$expand` group; keep `groups = []` and rely on `$lookup` + `vsCheck` (and category-filtered lookups only).
3. [index.html `fhir()` L347–364] Hedge is a fix-introduced regression. `Promise.race([tag(p1), tag(p2)])` fails the whole call if the first `fetch` rejects while the hedge is in flight; `winner` is set only on fulfill, so a rejecting loser is not aborted. After the race settles, `first.abort()` / `hedge.abort()` rejects the other `fetch` with `AbortError` and nobody catches it → unhandledrejection. Any `AbortError` (including that loser abort, if it wins the race) is mapped to the 45 s message (L356). Fix: swallow `AbortError` on the loser; fail only if both attempts fail or the 45 s ceiling fires; do not treat a non-ceiling abort as a timeout.
4. [index.html `getCvx()` L475–485] If `CodeSystem?url=…&_count=1` returns 200 without `entry[0].resource.concept` (or fhir cached an OperationOutcome and `cs` is missing), `cvxList` becomes `[]` and is written to `localStorage` under `cartos:cvx:v1` for `CVX_TTL_MS` (7 days). Vaccines stay empty until TTL. Fix: persist/accept only `Array.isArray(v) && v.length > 0`; on empty, throw and leave cache unset.
5. [index.html CSS L134 + L50–73] `.chip[aria-pressed="true"] { color: #fff; background: var(--cat) }`. Dark theme sets `--cat` to light swatches (`--accent: #78b8f5`, `--sys-loinc: #5fd3c6`, `--sys-rxnorm: #6ee7a0`, `--sys-cvx: #fb9a5c`, etc.). White-on-light selected chips fail WCAG against the brief’s own Focus #3. Fix: selected-chip `color: var(--accent-ink)` (already `#061120` in dark) or a dedicated `--chip-on` that stays dark on those backgrounds.
6. [index.html CSS L173–175, Focus #3] `button.row { all: unset }` with only a 7% accent background on `:focus-visible`. Chips, `.search button`, `.copy`, `.more button` have no `:focus`/`:focus-visible` ring. Keyboard focus is not visible. Fix: after `all: unset`, set `outline` / a 2px `box-shadow` on `:focus-visible` for `.row`, `.chip`, `.copy`, `.search button`, `.more button` (do not rely on background alone).

SHOULD-FIX:
1. [index.html footer L235–252, Focus #4] SNOMED 8.3.1, LOINC copyright, and NLM RxNorm courtesy text are verbatim but inside collapsed `<details>`. [ASSUMPTION] those licenses expect the notice visible without an extra click. Smallest: hoist the three `<blockquote>`s to always-visible footer paragraphs; keep the how-to-license prose in `<details>`.
2. [index.html `fhir()` L374–377] Any non-OK body with `resourceType === 'OperationOutcome'` is cached for the visit, including 5xx. A transient CARTOS error becomes a sticky miss. Fix: cache OO only when `res.status === 400`.
3. [index.html `expand()` L466–471] Non-ValueSet body (cached 400 OO) returns `{ total: 0, items: [] }`, which `fillGroup` treats as “no matches” and `runQuery` can present as an inconclusive empty, not a transport/server failure. Fix: if `body.resourceType !== 'ValueSet'`, throw (same as a failed group).
4. [index.html `validateInVs()` L449–454] `p.result === true` misses `valueString: "true"`. Live CARTOS likely sends `valueBoolean` (brief: verified 2026-09-02); still a single-point LOINC fallback break. Fix: treat `result === true || result === 'true'`.
5. [index.html `runQuery` L606–611 + `syncHash` L593–596] Row click `runQuery(item.code, { system })` is correct for the session but hash is only `q` + `in`, so refresh/share of a clicked SNOMED/RxNorm/CVX row re-runs the ambiguous numeric plan. Fix: `&sys=` (key or URI) and honor it like `opts.system`.
6. [index.html `EXAMPLES` L297 + `classify` ICD branch] Chip `B12` matches the ICD pattern (`B12` is an ICD-10-CM category), so it does not word-search labs/meds. Misleading for a general-public “Try” list. Fix: replace with a non-ICD token (`a1c`, `covid`) or, after an ICD category-code lookup, still run word search.
7. [index.html `runQuery` L703] Success path `setStatus('')` clears the only `aria-live` region, so SR users get no “results arrived” status (WCAG 4.1.3). Fix: set a one-line count (“Exact match plus 3 groups”) instead of emptying.
8. [index.html `fhir()` vs `seq` L590–600] Stale `runQuery` discards DOM updates but does not abort in-flight fetches; debounce (400 ms) + 4 `$expand`s + hedge can still stack modest-volume lookups. Fix: pass an AbortController tied to `seq` into `fhir()`, or coalesce in-flight by URL.
9. [index.html `getCvx()` L483] `inactive: false` is hardcoded, so retired CVX codes are never tagged. Fix: map `c.property` / `c.inactive` when present.

OPTIONAL / NICE-TO-HAVE:
- Persist `sys` in the hash as in SHOULD-FIX 5; announce live-dot state; `@media (forced-colors: active)` fallback for gradient `h1` (`color: transparent`).
- Coalesce identical in-flight `fhir()` URLs.
- `autofocus` on `#q` is rough for SR/keyboard; remove or gate.

CUT THESE:
1. Footer visit counter (`renderStats` / `#stats` L314–317, L254) — not required for decode/search, terms, or notices; drops live DOM writes on every request/hedge.
2. `.badge .dot.live` pulse (`@keyframes pulse` L101, L98) — connection state is already color + title; reduced-motion already disables it; zero product loss.
3. Example chip `B12` (not the whole Try row) — it fights `classify()`; `glucose` / `diabetes` already cover word search.

VERIFY-AT-BUILD checklist:
(No R1–R3 Kibitz artifacts exist; `kibitz-runs/2026-09-02-decoder-qa/r4/scope_receipt.md` states r1–r3 were not run. These are the brief’s live claims plus static-untraceable bits.)
- verify: `U07.1` `$lookup` on CARTOS after regex fix returns Parameters (not word-search noise).
- verify: with chip “Everything”, `2345-7` never hits `$expand?filter=2345-7`; with chip “Lab tests”, same after MUST-FIX 2.
- verify: LOINC `$lookup` still HTTP 400 + OO; `$validate-code` on `us-core-laboratory-test-codes` still returns `result` + display; confirm parameter is `valueBoolean`.
- verify: hyphenated `$expand` filter still ~13 s if accidentally sent (so the guard is still required).
- verify: `GET CodeSystem?url=http://hl7.org/fhir/sid/cvx&_count=1` still inlines a non-empty `concept[]` (brief: 290). Do not ship if empty.
- verify: `$expand` `contains[].system` URIs exactly match `SYSTEMS.*.url` in index.html L269–279 (row-click `systemByUrl` miss takes the full classify path).
- verify: CORS still `Access-Control-Allow-Origin: *`, GET only; ConceptMap count still 0.
- verify: hedge: first stall >8 s, duplicate wins, loser abort does not surface unhandledrejection or the 45 s copy; 45 s ceiling still shows the slow-moment message.
- verify: HTTP 400 + OperationOutcome is cached and shown as miss, not “Could not reach CARTOS”; 429 honors `Retry-After`.
- verify: selected-chip and tag contrast in light and dark (and `:focus-visible` rings) with a contrast checker; do not trust eyeballing.
- verify: live GitHub Pages `https://jbrick2070.github.io/health-code-decoder/` serves this `index.html` (og:image `social-card.png` is in the repo; confirm it 200s on Pages).
- verify: SNOMED Affiliate 8.3.1 / LOINC license / RxNorm TOS actually allow collapsed-vs-visible placement (code cannot settle this).
- [ASSUMPTION] period filters like `E11.9` are not in the 13 s hyphen class; confirm `$expand?filter=E11.9` stays fast.
