# Health Code Decoder

A single-page, no-backend web app for the general public: paste a code from a medical bill, lab
report, or immunization record and see what it means, or type a word and see how it appears across
the vocabularies used inside US health records.

All data comes live from [CARTOS](https://healthit.gov/standards-and-technology/cartos/), the free
public FHIR R4 terminology service run by the Office of the National Coordinator for Health IT.
No account, key, or server is needed.

**Live page:** https://jbrick2070.github.io/health-code-decoder/

## Run it

Open `index.html` directly in a browser, or serve the folder:

```bash
python -m http.server 8765
```

then visit http://localhost:8765/.

## What it does

- **Searches words.** Runs the query through four US Core value sets on CARTOS (conditions and
  diagnoses, medications, lab tests, procedures) plus a locally cached list of CVX vaccines, and
  shows the groups in a fixed order with totals and "Show more" paging.
- **Decodes codes.** Recognizes ICD-10-CM (`E11.9`, `J45`), LOINC (`2345-7`), and bare numbers.
  A bare number is tried as a CVX vaccine code, an RxNorm drug code, and a SNOMED CT concept, and
  every hit is shown, because the pattern alone cannot tell which vocabulary it belongs to.
- **Narrows by category.** Chips for conditions, medications, lab tests, procedures, and vaccines
  both filter word searches and resolve ambiguous numbers.
- **Explains the vocabulary.** Each hit says which code set it belongs to and what that code set is
  for, in one plain sentence. Inactive codes are labeled.
- **Shows raw FHIR** for the curious, behind a disclosure, and offers a copy button.
- **Behaves.** Debounced input, stale responses discarded, in-memory cache for the visit, the small
  CVX list kept in localStorage for a week, HTTP 429 `Retry-After` honored, every query linkable
  via the URL hash (`#q=diabetes&in=meds`).

## CARTOS endpoints used

| Purpose | Request |
|---|---|
| Decode a code | `GET /R4/CodeSystem/$lookup?system=…&code=…` |
| Decode a LOINC code | `GET /R4/ValueSet/$validate-code?url=<lab value set>&system=http://loinc.org&code=…` |
| Word search | `GET /R4/ValueSet/$expand?url=…&filter=…&count=20&offset=…` |
| Vaccine list | `GET /R4/CodeSystem?url=http://hl7.org/fhir/sid/cvx&_count=1` |

Base URL: `https://cartos.healthit.gov/TerminologyServer/R4`.

## Known limits (verified 2026-09-02)

- CARTOS has no ConceptMaps, so there is no SNOMED to ICD-10-CM crosswalk.
- Direct `$lookup` for LOINC codes returns HTTP 400 "Term was not found" even though LOINC 2.82 is
  loaded. The page falls back to `$validate-code` against the US Core lab value set, which answers
  in under a second and returns the display name.
- A hyphenated code used as an `$expand` filter takes CARTOS about 13 seconds. Avoid it.
- CARTOS intermittently stalls on a random request for a minute or more while an identical repeat
  answers in under a second (seen 2026-09-02 afternoon, not tied to URL form, origin, or cache).
  The page sends one duplicate after 8 seconds of silence, takes the first answer, and gives up
  at 45 seconds with a plain message.
- A lookup miss is an HTTP 400 with an OperationOutcome body. The browser console logs each one as
  a failed resource load; that is expected and harmless.
- Whole-vocabulary text search is not exposed; searching goes through value sets, so a valid code
  outside those value sets will not be found by word search.
- CPT, HCPCS, NDC, and other proprietary or unlisted code sets are not in CARTOS.
- Rate limit: 2,000 requests per 5 minutes per IP. Do not bulk-harvest.

## Second opinion

The design was reviewed by OpenAI Codex (GPT) before the final pass. Its main points, all adopted:
make word search the primary experience, show every plausible system for a bare number instead of
guessing, add category narrowing, treat a missing code as inconclusive rather than invalid, keep
persistent caching minimal, and surface inactive status and steward attributions.

## Terms, licenses, and disclaimers

**Code:** MIT License, see [LICENSE](LICENSE). The license covers the code only, not the terminology.

**CARTOS terms of use** (from the [Cartos User Guide](https://healthit.gov/resources/cartos-oncs-terminology-service/),
checked 2026-09-02): the API is "intended for terminology discovery, design-time queries and
validation, research, and periodic retrieval of terminology content"; "production clinical
applications must not rely on the Cartos API for direct runtime terminology queries"; do not
bulk-harvest or work around rate limits; and "users are responsible for following the licensing,
attribution, redistribution, and use requirements for each code system." This page is an
educational discovery tool that makes small interactive lookups within the published limits. It is
not a clinical or production system and says so on the page.

**Vocabulary notices** shown verbatim in the page footer:

- **SNOMED CT** requires the SNOMED CT Affiliate License even though use in the US is free. US
  users get it through the free [UMLS Metathesaurus License](https://www.nlm.nih.gov/healthit/snomedct/snomed_licensing.html)
  (sign up at UTS). If you run your own copy of this page, register your own license.
- **LOINC** is free under the [LOINC license](https://loinc.org/license/), which requires the LOINC
  copyright notice on any product or online service that includes LOINC content.
- **RxNorm** is public data from NLM under the [RxNorm terms of service](https://www.nlm.nih.gov/research/umls/rxnorm/docs/termsofservice.html),
  which require the NLM courtesy statement and forbid implying NLM endorsement.
- **ICD-10-CM** and **CVX** are US government works in the public domain. The US Core value sets
  are published by HL7 International; HL7® and FHIR® are registered trademarks of HL7.

**Disclaimers:** not medical advice, not affiliated with or endorsed by ONC, HHS, NLM, CDC,
SNOMED International, Regenstrief, or HL7; provided as is without warranty.
