# Codex review of the prototype plan (2026-09-02)

Second opinion from OpenAI Codex (codex-cli 0.150.0-alpha.8, reasoning effort high) on the brief in `codex-brief.md`.
Everything it recommended was folded into `index.html`; see the README for the summary.

1. Best prototype

Claude’s combination of #2 and #3 is the strongest idea, but I would make the “one word, many vocabularies” search the main experience and code decoding the secondary path.

Word search works with the API’s proven strengths and teaches something genuinely interesting: “diabetes” can mean a diagnosis, medicine, laboratory test, or procedure depending on context. Code decoding is useful when someone has a real document in hand, but automatic system detection will often be uncertain.

The vaccine explorer is the easiest dependable first milestone, but too narrow as the final product. “Explain my record” is premature because FHIR bundles introduce privacy, parsing, and context problems. The quiz is educational garnish, not the core product.

2. Features that matter most

1. **Organized, understandable results.** Separate conditions, medications, laboratory tests, procedures, and vaccines into clearly labeled sections. Always show the code, display name, code system, and vocabulary version.

2. **Honest code-system identification.** For pasted codes, show all credible matches rather than silently choosing one. A numeric code could belong to CVX, RxNorm, or SNOMED. Ask for context such as “shot record,” “medication list,” or “diagnosis” when that would resolve ambiguity.

3. **Plain-English orientation.** Explain what each category represents and why similar-looking terms appear in multiple systems. Do not pretend the official display name is a patient-friendly medical explanation.

4. **Useful narrowing controls.** Let users select Conditions, Medications, Labs, Procedures, or Vaccines; cap initial results; show the reported total; and support “load more.” Hundreds of unranked diabetes or glucose matches are not useful.

5. **Shareable, inspectable results.** Put the query and selected category in the URL and provide copy buttons. Include direct source/version information and a concise “terminology lookup, not medical interpretation” notice.

3. API and vocabulary pitfalls

- Never infer a code system from format alone. Even distinctive patterns are hints, not proof. Display ambiguity explicitly.
- LOINC `$lookup` is demonstrably unreliable here. Search the real US Core laboratory value set instead, and say that failure to find a code is inconclusive—not evidence that the code is invalid.
- Value-set search is not universal vocabulary search. Results reflect each US Core value set’s scope and may omit legitimate codes outside it.
- Results may be enormous and relevance ordering may be poor. Limit, group, and progressively reveal them; do not imply the first result is the best clinical match.
- Preserve system URI and version. The same concept can change status or wording between releases.
- Surface inactive/deprecated status when CARTOS returns it. Do not hide obsolete codes merely because they still resolve.
- Handle 429 responses using `Retry-After`, debounce searches, cancel stale requests, and avoid automatic retries across every system.
- Include required steward attribution and license links. Do not offer bulk export or build a local vocabulary mirror.
- “Not medical advice” is necessary but insufficient. Avoid diagnostic language such as “you have” or “this result means.”

4. What to simplify or change

Trying every plausible system in parallel for every code is wasteful and can produce misleading certainty. First use context and pattern to narrow candidates; query several systems only when ambiguity remains.

Do not make persistent `localStorage` caching central to version one. It creates stale-result, storage, and vocabulary-license complications. An in-memory session cache is enough initially; caching the small CVX set locally is reasonable.

Start with word search across the four verified value sets plus CVX, then add code decoding for ICD-10-CM, SNOMED, RxNorm, and CVX. Treat LOINC decoding as an explicitly labeled fallback. Leave FHIR-bundle upload, quizzes, synonym enhancement, and sophisticated auto-detection for later.