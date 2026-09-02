# Second-opinion request: a public prototype on ONC CARTOS

You are advising a hobbyist tinkerer (not a clinician, not a health IT vendor) who wants to build
something useful for the general public or for people curious about health IT, using ONC's free
public FHIR terminology service, CARTOS. Answer in plain English. Be concrete and opinionated.
Do not write code. Keep the answer under 700 words.

## Verified facts about CARTOS (all checked live today, 2026-09-02)

- FHIR R4 base URL: https://cartos.healthit.gov/TerminologyServer/R4
- No account, API key, or login. CORS is wide open (Access-Control-Allow-Origin: *), GET only.
  So a single static HTML page can call it directly from the browser with no backend.
- Software: Clinical Architecture FHIR Terminology Services 1.0.
- Resources: CodeSystem ($lookup, $validate-code, $subsumes, $find-matches), ValueSet ($expand,
  $validate-code), ConceptMap ($translate, $closure). 303 code systems, 2,588 value sets.
- ConceptMap count is ZERO. No crosswalks (e.g. SNOMED -> ICD-10-CM) are possible.
- Loaded vocabularies include LOINC 2.82, SNOMED CT US Edition 2026-03-01, RxNorm 2026-08-05,
  ICD-10-CM 2026, CVX (CDC vaccine codes, 290 concepts, returned in full in one call), and the
  HL7 v2/v3 tables. Proprietary sets (CPT, CDT, NUBC, X12) are NOT present.
- CodeSystem/$lookup works for ICD-10-CM (E11.9 -> "Type 2 diabetes mellitus without
  complications", with a Billable property), SNOMED (73211009 -> "Diabetes mellitus"), CVX (208 ->
  COVID-19 mRNA vaccine), RxNorm (860975 -> metformin ER 500 MG tablet).
- LOINC $lookup for 2345-7 (glucose) returned "Term was not found" and $validate-code said no
  concept found, even though LOINC 2.82 is listed. So direct LOINC code lookup looks unreliable.
- $find-matches returned "Code System '$find-matches' was not found" (not actually usable).
- Implicit whole-code-system value sets (url=http://snomed.info/sct?fhir_vs=..., http://loinc.org/vs)
  all returned "not found". Reading CodeSystem/{id} returns at most 2,000 inline concepts with no
  paging, so you cannot text-search a whole big vocabulary directly.
- Text search DOES work via ValueSet/$expand?url=<value set>&filter=<words>&count=N on real value
  sets. Filter is word-prefix, AND across tokens, case-insensitive, also matches synonyms. Verified:
    - us-core-condition-code (SNOMED + ICD-10-CM, 193,682 codes): filter=diabetes -> 850 hits;
      filter=E11 -> 118 ICD-10-CM hits
    - us-core-medication-codes (RxNorm): filter=metformin -> 354 hits
    - us-core-laboratory-test-codes (LOINC): filter=glucose -> 997 hits
    - us-core-procedure-code (SNOMED): filter=appendectomy -> 11 hits
- Rate limit: 2,000 requests per 5 minutes per IP, HTTP 429 with Retry-After when exceeded.
  Max 2,000 concepts per expansion page. Terms of use prohibit bulk harvesting and rate-limit
  workarounds, and require following each vocabulary steward's license (SNOMED, LOINC, RxNorm).
- Bulk downloads exist separately at cartos.healthit.gov/ONCCollections/ per implementation guide.

## Candidate prototypes (Claude's list, easiest first)

1. Vaccine explorer: browse/search all 290 CVX codes. One call, then local.
2. Health code decoder: paste a code from an after-visit summary, lab report, shot record, or
   insurance statement; auto-detect the code system from the pattern; show the official meaning.
3. "One word, many vocabularies": type "asthma", see matching conditions, drugs, lab tests, and
   procedures side by side, showing how health IT splits one idea across SNOMED/RxNorm/LOINC.
4. "Explain my record": drop in a FHIR bundle exported from a patient portal / Apple Health and
   label every code. Bigger; builds on #2.
5. Code quiz: guess the meaning of a random vaccine or condition code.

Claude's current plan: build #2 and #3 together as ONE static HTML page with a single search box.
If the input looks like a code, decode it (try every plausible system in parallel). Otherwise search
by word across the four US Core value sets plus the locally cached CVX list. Debounce typing, cache
results in localStorage, honor 429 Retry-After, show steward attributions, say "not medical advice".

## Questions for you

1. Which prototype (or combination) is most useful for the general public or health-IT-curious
   people, and why? Disagree with Claude's pick if you think another is better.
2. What would make the chosen prototype genuinely useful rather than a toy? Name the 3 to 5 features
   that matter most, in priority order.
3. What are the pitfalls specific to this API and these vocabularies that the design must handle
   (ambiguous numeric codes across CVX/RxNorm/SNOMED, LOINC lookup gaps, rate limits, licensing
   text, misleading displays, inactive codes, etc.)?
4. Anything in the plan that is wrong, risky, or over-engineered for a tinkerer's first version?

Reply with numbered sections matching the questions.
